# The API Creation Contract

This is the contract every new endpoint in NimbusVault Backend must follow. It is non-negotiable: a PR that breaks the contract is not mergeable.

## The flow, in one diagram

```
1. VaultManagement/urls/<Domain>.py          (register the route)
        │
        │  ── 1a. NimbusVault/urls.py          (only if new urls module)
        ▼
2. VaultManagement/views/<Domain>.py         (class-based view; one method per verb)
        │
        ▼
3. VaultManagement/serializers/<Domain>.py   (validate the request)
        │
        ▼
4. Orchestrators/<Domain>Orchestrators/      (business workflow + rollback)
        │
        │  ── may invoke other domain orchestrators (cross-domain composition)
        ▼
5. bll/<Domain>/HelperFunctions.py           (pure helpers — no ORM)
        │
        ▼
6. bll/<Domain>/GetApisbll.py     OR         (read-path business logic)
   bll/<Domain>/PostApisbll.py               (write-path business logic)
        │
        ▼
7. bll/<Domain>/service.py                   ← ONLY place that touches the ORM
        │                                       Exposes ONLY:
        │                                         create, get, filter, update,
        │                                         delete, bulk_create,
        │                                         bulk_update, bulk_delete
        ▼
   VaultModels/models/<Domain>.py            (Django models)
```

## The seven steps

To add a new endpoint you must touch — in order — the following files and **only** the following files. (Files marked *new module* are only created the first time a domain is introduced.)

### 1. URL — `VaultManagement/urls/<Domain>.py`

Add one `path()` entry pointing at a class-based view from `VaultManagement/views/<Domain>.py`.

```python
# VaultManagement/urls/MyDomain.py
from django.urls import path
from VaultManagement.views.MyDomain import (
    CreateMyThing,
    GetMyThings,
)

urlpatterns = [
    path("my-domain/create",   CreateMyThing.as_view()),
    path("my-domain/getAll",   GetMyThings.as_view()),
]
```

If this is a brand-new domain (a new `urls/<Domain>.py` file), register it in `NimbusVault/urls.py`:

```python
# NimbusVault/urls.py
urlpatterns += [
    # ... existing entries ...
    path(ModelName.url_vault, include("VaultManagement.urls.MyDomain")),
]
```

URL conventions:

- All paths are mounted under `/vault/` (the `ModelName.url_vault` prefix is applied automatically by the `include`).
- Path segments use **lowerCamelCase** (e.g., `getAll`, `markAsRead`) or **kebab-case** (e.g., `my-domain`) — match the convention already used by the neighbouring URLs in the file.
- One `path()` per HTTP route, even if multiple verbs share a view (the view's class methods handle the verbs).

### 2. View — `VaultManagement/views/<Domain>.py`

Class-based, one method per HTTP verb. Each method:

1. Constructs the serializer with `data=request.data` (or `data=request.query_params` for GET filters).
2. Calls `serializer.is_valid()`; on failure returns a 400 with `flatten_serializer_errors(serializer.errors)`.
3. Resolves the orchestrator from `OrchestratorRegistry`.
4. Calls the orchestrator's domain method with `serializer.validated_data` (and `request.user` when needed). **The method only stages steps — it does not return business values.**
5. Calls `orch.mainorchestrator.execute()`.
6. Reads `result` / `result_status` from the orchestrator context (or uses a hard-coded status if the workflow is simple) and returns a DRF `Response(...)`.

```python
# VaultManagement/views/MyDomain.py
from NimbusVaultConstants.CommonConstants.CommonConstants import (
    ModelAPIView,
    flatten_serializer_errors,
)
from rest_framework.response import Response

from VaultManagement.serializers.MyDomain import (
    CreateMyThingSerializer,
    GetMyThingsQuerySerializer,
)
from Orchestrators.OrchestratorRegistry import OrchestratorRegistry
from Orchestrators.MyDomainOrchestrators.MyDomainOrchestrator import MyDomainOrchestrator


class CreateMyThing(ModelAPIView):
    """Create a new MyThing."""

    def post(self, request):
        serializer = CreateMyThingSerializer(data=request.data)
        if not serializer.is_valid():
            return Response(
                status=400,
                data={"msg": flatten_serializer_errors(serializer.errors)},
            )

        orch: MyDomainOrchestrator = OrchestratorRegistry.get("my_domain")
        orch.create_thing(user=request.user, **serializer.validated_data)   # stages steps
        orch.mainorchestrator.execute()                                     # runs them

        # The step published its outcome into the context; the view reads it back.
        result = orch.mainorchestrator.get_context_value("result", check_validation=False)
        status = orch.mainorchestrator.get_context_value("result_status", check_validation=False) or 201
        return Response(data=result or {"msg": "Created"}, status=status)


class GetMyThings(ModelAPIView):
    """List MyThings for the current user."""

    def get(self, request):
        serializer = GetMyThingsQuerySerializer(data=request.query_params)
        if not serializer.is_valid():
            return Response(
                status=400,
                data={"msg": flatten_serializer_errors(serializer.errors)},
            )

        # READ paths may skip the orchestrator and call BLL directly.
        from bll.MyDomain.GetApisbll import list_things
        results = list_things(user=request.user, **serializer.validated_data)
        return Response(data=results, status=200)
```

View conventions:

- Inherit from `ModelAPIView` (gets auth + perms wired automatically).
- Use type hints on the orchestrator variable so IDEs can navigate.
- **Never** call `Model.objects.…` from a view. **Never** call `service.py` from a view directly — go through the orchestrator (writes) or `GetApisbll.py` (reads).
- Use `@SchemaMapping.<method_name>` decorators where Swagger metadata is configured (see `Swagger/<Domain>/` for the constants module).

### 3. Serializer — `VaultManagement/serializers/<Domain>.py`

One serializer per request shape. Validates required fields, types, ranges, enum values. **No business logic** — no DB lookups, no permission checks. If it can't be expressed declaratively in the serializer, it belongs in the orchestrator or `bll/`.

```python
# VaultManagement/serializers/MyDomain.py
from rest_framework import serializers


class CreateMyThingSerializer(serializers.Serializer):
    name = serializers.CharField(max_length=255)
    description = serializers.CharField(required=False, allow_blank=True)
    category = serializers.ChoiceField(choices=["A", "B", "C"])


class GetMyThingsQuerySerializer(serializers.Serializer):
    category = serializers.ChoiceField(choices=["A", "B", "C"], required=False)
    limit = serializers.IntegerField(min_value=1, max_value=200, default=50)
    offset = serializers.IntegerField(min_value=0, default=0)
```

### 4. Orchestrator — `Orchestrators/<Domain>Orchestrators/<Domain>Orchestrator.py`

The orchestrator owns the **workflow**: which steps run, in what order, with what rollback. It composes calls into `bll/<Domain>/` and (when needed) other domain orchestrators.

**Three rules to internalise before writing one:**

1. **Methods stage steps. They do not return business values.** The view does not get a return back from the staging call.
2. **Forward and rollback are module-level functions in `bll/<Domain>/PostApisbll.py`.** Not lambdas, not closures, not methods on the orchestrator class. The orchestrator passes them by reference to `add_step(forward_fn, rollback_fn, *args, **kwargs)`.
3. **Steps signal "exit the workflow" by writing `result_status` to the orchestrator context** — not by returning a value. Setting `result_status` raises `StepExitException` and bails the rest of the steps (with or without rollback, depending on the value).

```python
# Orchestrators/MyDomainOrchestrators/MyDomainOrchestrator.py
from Orchestrators.OrchestratorRegistry import OrchestratorRegistry
from Orchestrators.AbstractOrchestrators.BaseOrchestrator import BaseOrchestrator
from Orchestrators.ContextManager import context_manager
from VaultModels.ModelRegistry import MODEL_REGISTRY

from bll.MyDomain.PostApisbll import create_thing_bll, rollback_create_thing_bll


class MyDomainOrchestrator(metaclass=BaseOrchestrator):
    """
    Description:
    Orchestrates MyDomain write workflows (create / update / delete).
    """

    def __init__(self, orchestrator_obj):
        """
        Description: Bind to the shared MainOrchestrator and resolve siblings.

        Input:
            - orchestrator_obj: The MainOrchestrator instance bound to this run.

        Output:
            - None.
        """
        self.mainorchestrator = orchestrator_obj or OrchestratorRegistry.get("main")

    @context_manager(
        get_contexts={"common": []},
        set_contexts={"common": ["result", "result_status", "thing_obj"]},
    )
    def create_thing(self, user, name, description="", category="A"):
        """
        Description: Stage the create-MyThing workflow.

        Input:
            - user: Acting user.
            - name, description, category: Field values for the new MyThing.

        Output:
            - None. Result lands in the orchestrator context.
        """
        self.mainorchestrator.set_context_schema({
            "thing_obj": MODEL_REGISTRY["MyThing"],
        })
        self.mainorchestrator.add_step(
            create_thing_bll,
            rollback_create_thing_bll,
            user=user,
            name=name,
            description=description,
            category=category,
        )
```

And the step functions live in `bll/<Domain>/PostApisbll.py`:

```python
# bll/MyDomain/PostApisbll.py
from Orchestrators.ContextManager import context_manager
from VaultErrors.BackendErrors import errors
from bll.MyDomain.HelperFunctions import validate_thing_name, format_thing
from bll.MyDomain.service import my_thing_service


@context_manager(get_contexts={"common": []},
                 set_contexts={"common": ["result", "result_status", "thing_obj"]})
def create_thing_bll(orchestrator, user, name, description, category):
    """Forward step. Receives the MainOrchestrator as the first arg."""
    validate_thing_name(name)

    if my_thing_service.exists(name=name, owner=user):
        orchestrator.set_context_value("result", {"msg": f"'{name}' already exists"})
        orchestrator.set_context_value("result_status", 400)   # raises StepExitException → rollback
        return

    thing = my_thing_service.create(name=name, description=description, category=category, owner=user)
    orchestrator.set_context_value("thing_obj", thing)
    orchestrator.set_context_value("result", format_thing(thing))
    orchestrator.set_context_value("result_status", 201)       # raises StepExitException → exit, no rollback


@context_manager(get_contexts={"common": ["thing_obj"]}, set_contexts={})
def rollback_create_thing_bll(orchestrator):
    """Rollback. Runs only if a later step in the workflow fails."""
    thing = orchestrator.get_context_value("thing_obj")
    my_thing_service.delete(thing)
```

Then **register the orchestrator** by adding one line to `Orchestrators/RegisterOrchestrator.py`:

```python
# Orchestrators/RegisterOrchestrator.py
from Orchestrators.MyDomainOrchestrators.MyDomainOrchestrator import MyDomainOrchestrator

def register_orchestrator():
    # ... existing entries ...
    OrchestratorRegistry.register("my_domain", MyDomainOrchestrator)
```

Orchestrator conventions:

- **No decorator.** Registration is an explicit line in `Orchestrators/RegisterOrchestrator.py::register_orchestrator()`.
- The class uses `metaclass=BaseOrchestrator`, which enforces Description/Input/Output docstrings on every method and the class itself.
- Forward and rollback functions live in `bll/<Domain>/PostApisbll.py` (or wherever the domain's write-side BLL lives) — they are **module-level** so they can be passed by reference.
- Steps communicate via the orchestrator context (`get_context_value` / `set_context_value`). Every method and step declares its access via `@context_manager(get_contexts=..., set_contexts=...)`.
- Compose other orchestrators by resolving them in `__init__` via `self.mainorchestrator.get_other_orchestrator(self, "<name>")` and calling their methods — their steps land on the same `mainorchestrator` and roll back as one.
- A read-only endpoint does **not** need an orchestrator — call `bll/<Domain>/GetApisbll.py` directly from the view.

See [Architecture → Orchestrators](../architecture/orchestrators.md) for the full framework reference, including exit-signalling semantics, `@context_manager` rules, and the docstring metaclass requirements.

### 5. Helpers — `bll/<Domain>/HelperFunctions.py`

Pure functions used across `GetApisbll.py` and `PostApisbll.py`. **No ORM access.** Examples: input normalisation, name validation, formatting helpers, permission predicates that only need data already in the request, computations.

```python
# bll/MyDomain/HelperFunctions.py
import re
from VaultErrors.BackendErrors import errors


def validate_thing_name(name: str) -> None:
    if not re.match(r"^[A-Za-z][A-Za-z0-9 _-]{0,254}$", name):
        raise errors.bad_request.ValidationError(
            "Name must start with a letter and contain only letters, digits, space, _ or -."
        )


def format_thing_payload(thing) -> dict:
    return {
        "id": str(thing.id),
        "name": thing.name,
        "category": thing.category,
    }
```

### 6. Verb-split BLL — `bll/<Domain>/GetApisbll.py`, `bll/<Domain>/PostApisbll.py`

Business logic is split by HTTP verb intent:

| File | What goes here |
|---|---|
| `GetApisbll.py` | Read paths. List, retrieve, search, export, count, exists checks. |
| `PostApisbll.py` | Write paths. Create, update, delete, bulk operations, state transitions, side effects (audit log writes, notifications, webhooks). |

PATCH and DELETE business logic also go in `PostApisbll.py` (it's "non-GET", not literally "POST"). The split is about read vs. write, not the HTTP verb name.

Each function in these files:

- Receives Python primitives + the requesting `user`.
- Calls `service.py` for all ORM access.
- Calls `HelperFunctions.py` for pure transformations.
- Returns dicts / domain objects (not DRF Responses — those are the view's job).
- Raises `errors.*` from `VaultErrors/BackendErrors.py` on business failures.

```python
# bll/MyDomain/PostApisbll.py
from VaultErrors.BackendErrors import errors
from bll.MyDomain.HelperFunctions import format_thing_payload
from bll.MyDomain.service import my_thing_service


def create_thing(user, name, description, category):
    existing = my_thing_service.get(name=name, owner=user)
    if existing is not None:
        raise errors.bad_request.ValidationError(f"Thing '{name}' already exists.")
    thing = my_thing_service.create(
        name=name, description=description, category=category, owner=user,
    )
    return thing


def undo_create_thing(thing):
    my_thing_service.delete(thing)
```

```python
# bll/MyDomain/GetApisbll.py
from bll.MyDomain.HelperFunctions import format_thing_payload
from bll.MyDomain.service import my_thing_service


def list_things(user, category=None, limit=50, offset=0):
    filters = {"owner": user}
    if category is not None:
        filters["category"] = category
    qs = my_thing_service.filter(**filters).order_by("-created_at")
    rows = qs[offset : offset + limit]
    return {
        "results": [format_thing_payload(t) for t in rows],
        "total": qs.count(),
    }
```

### 7. Service — `bll/<Domain>/service.py`

**The only file in the entire codebase allowed to import a Django model from this domain.** It exposes exactly the data-access primitives listed in [Data layer → What `service.py` is allowed to contain](../architecture/data-layer.md#what-servicepy-is-allowed-to-contain) and nothing more.

```python
# bll/MyDomain/service.py — minimal example
from typing import Optional, Iterable, Any
from django.db.models import QuerySet
from VaultModels.models.MyDomain import MyThing


class MyThingService:
    @staticmethod
    def create(**fields) -> MyThing:
        return MyThing.objects.create(**fields)

    @staticmethod
    def get(**filters) -> Optional[MyThing]:
        return MyThing.objects.filter(**filters).first()

    @staticmethod
    def filter(**filters) -> QuerySet:
        return MyThing.objects.filter(**filters)

    @staticmethod
    def update(instance: MyThing, **fields) -> MyThing:
        for k, v in fields.items():
            setattr(instance, k, v)
        instance.save(update_fields=list(fields.keys()))
        return instance

    @staticmethod
    def delete(instance_or_id: Any) -> bool:
        if isinstance(instance_or_id, MyThing):
            instance_or_id.delete()
            return True
        deleted, _ = MyThing.objects.filter(pk=instance_or_id).delete()
        return deleted > 0

    @staticmethod
    def bulk_create(rows: Iterable) -> list[MyThing]:
        instances = [r if isinstance(r, MyThing) else MyThing(**r) for r in rows]
        return MyThing.objects.bulk_create(instances)

    @staticmethod
    def bulk_update(rows: Iterable[MyThing], fields: list[str]) -> int:
        return MyThing.objects.bulk_update(list(rows), fields)

    @staticmethod
    def bulk_delete(filters_or_ids) -> int:
        if isinstance(filters_or_ids, dict):
            deleted, _ = MyThing.objects.filter(**filters_or_ids).delete()
        else:
            deleted, _ = MyThing.objects.filter(pk__in=filters_or_ids).delete()
        return deleted


my_thing_service = MyThingService()
```

## Hard rules

These are not suggestions:

1. **`service.py` is the only file that touches `Model.objects`.** Every other module imports the service and calls its functions.
2. **`service.py` contains only `create`, `get`, `filter`, `update`, `delete`, `bulk_create`, `bulk_update`, `bulk_delete`.** Nothing else.
3. **Views never call `service.py` directly.** Writes go through orchestrators; reads go through `bll/<Domain>/GetApisbll.py`.
4. **Serializers contain no business logic.** Validation only — no DB lookups, no permission checks.
5. **No business logic in URL files.** A URL file imports views and maps paths. That's it.
6. **All errors use `VaultErrors/BackendErrors.errors.*`.** Never raise bare `Exception` or DRF built-ins from the BLL.
7. **Orchestrator steps have rollbacks.** Either a real compensating action or an explicit no-op with a comment explaining why.

## Anti-patterns (don't do this)

| Anti-pattern | Why it's wrong | Where it should live instead |
|---|---|---|
| `Model.objects.filter(...)` inside a view | Bypasses the service layer; N+1 fixes are then everyone's problem. | `service.py.filter(...)`. |
| `from VaultModels.models import …` inside `HelperFunctions.py` | Helpers are pure — no ORM. | Move call into `GetApisbll.py` / `PostApisbll.py`, which calls `service.py`. |
| `service.py` with a `def publish_version(...)` method that does permission check + audit log + status change | Business logic crept into the data-access layer. | Split: status change → `update()` call; permission + audit → `PostApisbll.py`. |
| Serializer's `validate_x` method calls `Model.objects.filter(...).exists()` | Same problem as views — the service layer is bypassed. | Move the existence check into `PostApisbll.py` and raise from there. |
| One giant `Apisbll.py` with everything | Loses the GET/POST split that makes reads easy to optimise independently of writes. | Split into `GetApisbll.py` and `PostApisbll.py`. |
| View directly returns `Response(model_instance)` | Models aren't JSON-serialisable; this leaks ORM shape to the API. | Use a serializer or a `format_*` helper for response shaping. |

See the next chapter, [Worked example](walkthrough.md), for a complete end-to-end build that follows the contract — and [Good practices](good-practices.md) for the performance patterns (N+1 avoidance, bulk ops, caching, indexes) every implementation should use.
