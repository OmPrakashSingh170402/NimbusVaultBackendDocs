# Data Layer — Models & `service.py`

This is the most important chapter. The whole architecture stands on one rule:

!!! danger "The rule"
    **No function — anywhere in the codebase — touches a Django Model directly.** ORM access goes through a per-domain `service.py`. That `service.py` exposes **only** the basic data-access primitives. **No business logic in `service.py`.** No queries, joins, or filters built outside `service.py`. No `Model.objects.…` calls outside `service.py`.

## What `service.py` is allowed to contain

A function in `service.py` is permitted if and only if it is one of:

| Primitive | Signature shape | Purpose |
|---|---|---|
| `create` | `create(**fields) -> Model` | Single-row insert. |
| `get` | `get(**filters) -> Optional[Model]` | Single-row read (returns `None` if missing — do not raise here). |
| `filter` | `filter(**filters) -> QuerySet` | Queryset construction. Pass `_select_related`, `_prefetch_related` as kwargs for eager loading hints. |
| `update` | `update(instance, **fields) -> Model` | Single-row update. Save and return. |
| `delete` | `delete(instance_or_id) -> bool` | Single-row delete. Return success bool. |
| `bulk_create` | `bulk_create(list_of_dicts_or_objs) -> list[Model]` | Multi-row insert. |
| `bulk_update` | `bulk_update(list_of_pairs, fields) -> int` | Multi-row update. Return rows affected. |
| `bulk_delete` | `bulk_delete(filter_or_ids) -> int` | Multi-row delete. Return rows affected. |

Nothing else. No `publish_version`, no `compute_impact`, no `migrate_entities`. That logic lives in `bll/<Domain>/` (helpers + GetApisbll/PostApisbll) or in an orchestrator.

## What `service.py` is **not** allowed to contain

- Permission checks
- Audit-log writes
- Cross-model joins beyond a `select_related` / `prefetch_related` hint
- Conditional business rules ("if status is published, also set is_default…")
- Calls to other domain services (those compose at the BLL or orchestrator layer)
- Sending notifications, emails, webhooks
- Cache reads/writes
- Try/except that swallows errors

If you find yourself writing any of those inside `service.py`, the function belongs in `bll/<Domain>/HelperFunctions.py`, `GetApisbll.py`, `PostApisbll.py`, or a step on an orchestrator.

## The shape of a service module

```python
# bll/<Domain>/service.py
from typing import Any, Iterable, Optional
from django.db.models import QuerySet
from VaultModels.models.<your_models_module> import <YourModel>


class <YourModel>Service:
    _DEFAULT_SELECT_RELATED: tuple[str, ...] = ()  # override per-domain

    @staticmethod
    def create(**fields) -> <YourModel>:
        return <YourModel>.objects.create(**fields)

    @staticmethod
    def get(**filters) -> Optional[<YourModel>]:
        select_related = filters.pop("_select_related", <YourModel>Service._DEFAULT_SELECT_RELATED)
        prefetch_related = filters.pop("_prefetch_related", ())
        qs = <YourModel>.objects.filter(**filters)
        if select_related:
            qs = qs.select_related(*select_related)
        if prefetch_related:
            qs = qs.prefetch_related(*prefetch_related)
        return qs.first()

    @staticmethod
    def filter(**filters) -> QuerySet:
        select_related = filters.pop("_select_related", <YourModel>Service._DEFAULT_SELECT_RELATED)
        prefetch_related = filters.pop("_prefetch_related", ())
        qs = <YourModel>.objects.filter(**filters)
        if select_related:
            qs = qs.select_related(*select_related)
        if prefetch_related:
            qs = qs.prefetch_related(*prefetch_related)
        return qs

    @staticmethod
    def update(instance: <YourModel>, **fields) -> <YourModel>:
        for key, value in fields.items():
            setattr(instance, key, value)
        instance.save(update_fields=list(fields.keys()))
        return instance

    @staticmethod
    def delete(instance_or_id: Any) -> bool:
        if isinstance(instance_or_id, <YourModel>):
            instance_or_id.delete()
            return True
        deleted, _ = <YourModel>.objects.filter(pk=instance_or_id).delete()
        return deleted > 0

    @staticmethod
    def bulk_create(rows: Iterable[dict | <YourModel>]) -> list[<YourModel>]:
        instances = [r if isinstance(r, <YourModel>) else <YourModel>(**r) for r in rows]
        return <YourModel>.objects.bulk_create(instances)

    @staticmethod
    def bulk_update(rows: Iterable[<YourModel>], fields: list[str]) -> int:
        return <YourModel>.objects.bulk_update(list(rows), fields)

    @staticmethod
    def bulk_delete(filters_or_ids) -> int:
        if isinstance(filters_or_ids, dict):
            deleted, _ = <YourModel>.objects.filter(**filters_or_ids).delete()
        else:
            deleted, _ = <YourModel>.objects.filter(pk__in=filters_or_ids).delete()
        return deleted


# instantiate once; import this object everywhere else
your_model_service = <YourModel>Service()
```

Then everywhere else, the call sites read like:

```python
# bll/<Domain>/PostApisbll.py
from bll.<Domain>.service import your_model_service

def publish_thing(user, thing_id):
    thing = your_model_service.get(id=thing_id)
    if not thing:
        raise errors.bad_request.NotFound("Thing not found")
    # ... business logic ...
    return your_model_service.update(thing, status="PUBLISHED", updated_by=user)
```

## Why this rule exists

| Without the rule | With the rule |
|---|---|
| `Model.objects.filter(...)` scattered in views, helpers, orchestrators. Adding a `select_related` or fixing an N+1 means hunting across dozens of files. | One place to optimise. Adding `_select_related` defaults in `service.py` cascades to every caller. |
| Permission/audit logic gets duplicated in 4 places because everyone reaches for the model. | Permission + audit live in `bll/<Domain>/` — exactly once per code path. |
| Refactoring a model schema breaks dozens of call sites. | Refactor `service.py`'s public surface; all callers go through it. |
| Hard to mock for tests. | Mock the service object. |
| Tenant routing bugs creep in via raw `Model.objects` calls inside Celery tasks. | Tasks call `service.py`; `service.py` is the single audit point. |

## Models

- Lives under `VaultModels/models/`.
- One module per logical domain (`templateversioning.py`, `CustomModel.py`, `ModelEntity.py`, etc.).
- Registered in `VaultModels/ModelRegistry.py` — a central dictionary that maps model-name strings (from `NimbusVaultConstants/Modelentity/`) to model classes. The registry is what lets dynamic-rules, formula-engine, and bulk-upload code look up models by name without hard imports.
- Migrations are auto-generated (`python manage.py makemigrations`). For shipped production data, prefer `manage.py migrate --fake` (or `make fake`) only when a migration has been applied out-of-band.

## What about `ModelRegistry.py`?

If you need a model class by its registered name (e.g., from a rule definition or a formula reference), use:

```python
from VaultModels.ModelRegistry import get_model
Model = get_model(model_name)   # returns the class
```

…but then you still **must not** call `Model.objects.…` directly. Wrap it behind a `service.py` function or call an existing service.

## Existing reference implementation

`bll/TemplateVersioning/service.py` is the closest existing match to the rule. It exposes `filter_versions`, `get_version`, `delete_version`, `bulk_create_versions`, `bulk_delete_versions`, `bulk_publish_versions`. Some legacy business-logic methods (e.g., `publish_version`, `migrate_entities`) currently live in the same file — these are **inherited debt**, scheduled to be split into `PostApisbll.py`. New domains must not follow that pattern; they follow the clean shape above.

Other partial precedents: `Plugins/HDFCUAMPlugin/bll/UAMParameters/service.py`, `Plugins/HDFCUAMPlugin/bll/ModelId/service.py`, `Plugins/HDFCUAMPlugin/bll/Workflow/service.py`.
