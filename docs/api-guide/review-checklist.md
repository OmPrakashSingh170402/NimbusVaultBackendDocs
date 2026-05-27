# Code Review Checklist

Use this when reviewing a PR that adds or changes an API. If any box is unchecked, request changes.

## URL layer

- [ ] New route lives in an existing `VaultManagement/urls/<Domain>.py` (or, for a new domain, a new module under `VaultManagement/urls/`).
- [ ] If a new module was added, `NimbusVault/urls.py` includes it under `ModelName.url_vault`.
- [ ] Path follows the local naming convention of the surrounding routes (camelCase or kebab-case — match the file).
- [ ] No business logic in the URL file. Only `path(...)` entries and view imports.

## View layer

- [ ] Class-based view inheriting from `ModelAPIView`.
- [ ] One method per HTTP verb. No verb dispatching by `request.method` inside a method.
- [ ] Method calls a serializer with `data=request.data` (or `data=request.query_params` for GET).
- [ ] Returns 400 + `flatten_serializer_errors(...)` on serializer failure.
- [ ] **Writes** call `OrchestratorRegistry.get("<domain>")` + a staging method + `orch.mainorchestrator.execute()`, then read `result` / `result_status` from `orch.mainorchestrator.get_context_value(...)` to build the response. The view does not rely on a return value from the staging method.
- [ ] **Reads** call `bll/<Domain>/GetApisbll.py` directly.
- [ ] No `Model.objects.…` calls. No direct `service.py` imports.
- [ ] Response is wrapped in DRF `Response(...)` with an explicit status code.

## Serializer layer

- [ ] One serializer per request shape (do not reuse the create serializer for an update unless fields match exactly).
- [ ] No DB lookups inside `validate_*` methods.
- [ ] No permission checks (those belong in `PostApisbll.py`).
- [ ] Field types match the model and downstream BLL expectations.

## Orchestrator layer

- [ ] Class uses `metaclass=BaseOrchestrator`.
- [ ] Class docstring has a `Description:` section. Every method docstring has `Description:`, `Input:`, `Output:` (the metaclass rejects the class otherwise).
- [ ] `__init__(self, orchestrator_obj)` assigns `self.mainorchestrator = orchestrator_obj or OrchestratorRegistry.get("main")`.
- [ ] Registered with one line in `Orchestrators/RegisterOrchestrator.py::register_orchestrator()`. **No `@register_orchestrator` decorator** — that doesn't exist.
- [ ] Domain method **stages** steps via `self.mainorchestrator.add_step(forward_fn, rollback_fn, *args, **kwargs)` and returns nothing meaningful.
- [ ] Forward and rollback are **module-level functions** in `bll/<Domain>/PostApisbll.py`, passed by reference. **No lambdas, no closures, no methods on the class.**
- [ ] Every staged forward step has a real rollback function (or an explicit no-op with a one-line comment justifying it).
- [ ] Every method and every step function carries `@context_manager(get_contexts=..., set_contexts=...)` declaring the keys it reads/writes.
- [ ] Any key declared in a step's `get_contexts` / `set_contexts` is also declared in **every** parent up the call chain (the framework enforces this when `sync_with_parent_context=True`, the default).
- [ ] Conditional buckets (`"is_bulk": [...]`, `"is_admin": [...]`) only used when a key in the orchestrator context legitimately gates the allowed set; `"else"` only used when at least one conditional bucket is present.
- [ ] Callable (`lambda` / named function) form for `get_contexts` / `set_contexts` used **only** when the keys genuinely vary with call arguments; a static dict is preferred.
- [ ] `sync_with_parent_context=False` not used in normal application code (only acceptable for one-off worker / backfill entrypoints, with justification in the PR description).
- [ ] Context schema for any new keys is declared via `self.mainorchestrator.set_context_schema({...})` before any step writes them.
- [ ] Step functions exit the workflow by writing `result_status` to the context — not by returning a value. Successful exit uses a non-`400/500` status; failure uses `400` or `500` (triggers rollback).
- [ ] Cross-domain composition resolves siblings in `__init__` via `self.mainorchestrator.get_other_orchestrator(self, "<name>")` and calls their methods — their steps land on the same `mainorchestrator`.
- [ ] No `Model.objects.…` calls. No direct `service.py` calls from the orchestrator class (only step functions in `bll/<Domain>/PostApisbll.py` call the service).

## BLL layer

- [ ] `HelperFunctions.py` contains only pure helpers — no ORM, no I/O.
- [ ] Read-path logic in `GetApisbll.py`; write-path logic in `PostApisbll.py`.
- [ ] All errors raised through `VaultErrors/BackendErrors.errors.*`. No bare `Exception`, no DRF built-ins.
- [ ] Functions return Python primitives or domain dicts, never DRF `Response`.
- [ ] Permission checks live here, not in serializers or views.
- [ ] Audit-log writes live here, not in `service.py`.
- [ ] Returns are shaped by `format_*` helpers, not by leaking model instances.

## Service layer

- [ ] `service.py` is the **only** file in this domain that imports a Django model.
- [ ] Exposes only `create`, `get`, `filter`, `update`, `delete`, `bulk_create`, `bulk_update`, `bulk_delete`. **No other functions.**
- [ ] No permission checks, no audit-log writes, no conditional business rules, no notifications/emails/webhooks.
- [ ] `select_related` / `prefetch_related` defaults set when the domain has FK-heavy reads.
- [ ] A module-level singleton instance is exported (e.g., `my_thing_service = MyThingService()`).

## Model & migration

- [ ] Migration committed alongside model changes.
- [ ] `unique_together` / `index_together` / explicit `db_index=True` added where the BLL queries by that field.
- [ ] Model registered in `VaultModels/ModelRegistry.py` if dynamic-rules / formulas / bulk-upload need it.
- [ ] No `JSONField` smuggling business state that would be better modelled as separate fields.

## Tenancy & async

- [ ] If the endpoint dispatches a Celery task, the task signature includes the tenant (`org`) and re-routes the DB inside the task.
- [ ] Cross-tenant operations (rare) are called out in the PR description.

## Tests

- [ ] At least one happy-path test per endpoint.
- [ ] At least one validation-failure test (400 response).
- [ ] At least one permission/business-failure test (403 / 404 / 400 per the spec).
- [ ] Test asserts the response shape, not just the status code.

## Docs

- [ ] Public API change: Swagger schema annotations updated (`@SchemaMapping.…`).
- [ ] If you introduced a new convention or relaxed an existing one, the relevant page in `docs/` is updated in the same PR.
- [ ] If you added a feature flag, its expected removal condition is in the PR description.

## Performance

- [ ] No N+1 in read paths — confirmed by querying with `assertNumQueries` or `silk` in dev.
- [ ] List endpoints have `limit` + `offset` (or cursor) validation in the serializer.
- [ ] Bulk operations route through `bulk_create` / `bulk_update` / `bulk_delete`, not loops over `create()`.

## Security

- [ ] User-supplied identifiers (ids, slugs) are validated by the serializer and scoped by `owner=request.user` (or equivalent) in the service filter.
- [ ] No raw SQL except where unavoidable; if present, parameterised.
- [ ] No `eval`, `exec`, `pickle.loads` on request data.
