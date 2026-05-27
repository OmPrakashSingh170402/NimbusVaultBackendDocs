# Orchestrators

The orchestrator framework is the spine of every non-trivial write path. It enforces step ordering, captures rollback hooks, and lets domains compose each other's workflows.

## Files

| File | Purpose |
|---|---|
| `Orchestrators/MainOrchestrator.py` | Generic step runner, context manager, and rollback engine. |
| `Orchestrators/OrchestratorRegistry.py` | Discovery: `OrchestratorRegistry.get("<name>")` returns a fully-wired orchestrator instance bound to a fresh `MainOrchestrator`. |
| `Orchestrators/RegisterOrchestrator.py` | **The registration table.** A single `register_orchestrator()` function calls `OrchestratorRegistry.register("<name>", Class)` once per domain. Add your line here when you create a new domain orchestrator. |
| `Orchestrators/AbstractOrchestrators/BaseOrchestrator.py` | **Metaclass** that enforces Description/Input/Output docstrings on every class and method. Also gates instantiation through the registry (you cannot `XxxOrchestrator(...)` directly). |
| `Orchestrators/ContextManager.py` | `@context_manager(get_contexts=..., set_contexts=...)` decorator declaring which context keys a method touches. |
| `Orchestrators/OrchestrationExceptions.py` | `StepExitException` — raised internally when a step sets `result_status` to exit the orchestrator early. |
| `Orchestrators/<Domain>Orchestrators/` | One folder per domain. |
| `Orchestrators/WorkerTasks/` | Celery tasks that wrap orchestrator runs for async execution. |

## Core ideas

1. **Methods stage steps. They don't do work.** A method on `AlertOrchestrator` doesn't create a notification — it appends a step describing *how to* create one (plus its rollback) onto the shared `mainorchestrator`. Nothing runs until the view calls `mainorchestrator.execute()`.
2. **Forward and rollback functions are module-level functions in `bll/<Domain>/PostApisbll.py`** (or equivalent). They are not class methods, not lambdas, not closures. The orchestrator passes them by reference to `add_step`.
3. **Steps communicate through the orchestrator context, not via return values.** A step reads inputs with `orchestrator.get_context_value("key")` and publishes outputs with `orchestrator.set_context_value("key", value)`.
4. **Exit early by setting `result_status`.** That assignment raises `StepExitException` inside the step; `execute()` catches it and either returns `result` (success exit) or triggers rollback (`result_status in [400, 500]`).
5. **`MainOrchestrator` is one-shot.** `OrchestratorRegistry.get("alert")` builds a fresh `MainOrchestrator` + fresh `AlertOrchestrator` bound to it. After `.execute()`, throw it away.

## How a domain orchestrator is wired

### 1. The class

```python
# Orchestrators/AlertOrchestrators/AlertOrchestrator.py
from Orchestrators.OrchestratorRegistry import OrchestratorRegistry
from Orchestrators.AbstractOrchestrators.BaseOrchestrator import BaseOrchestrator
from Orchestrators.ContextManager import context_manager
from VaultModels.ModelRegistry import MODEL_REGISTRY

from bll.Alerts.PostApisbll import (
    create_notification_bll,
    rollback_create_notification_bll,
)


class AlertOrchestrator(metaclass=BaseOrchestrator):
    """
    Description:
    Orchestrates alert/notification write workflows: declares context schema,
    composes EntityOrchestrator to resolve the target entity, then stages the
    create-notification step with its rollback.
    """

    def __init__(self, orchestrator_obj):
        """
        Description: Bind to the shared MainOrchestrator and resolve sibling
        orchestrators this domain depends on.

        Input:
            - orchestrator_obj: The MainOrchestrator instance bound to this run.

        Output:
            - None.
        """
        self.mainorchestrator = orchestrator_obj or OrchestratorRegistry.get("main")
        self.entity_orchestrator = self.mainorchestrator.get_other_orchestrator(self, "entity")

    @context_manager(
        get_contexts={"common": ["entity_obj", "updated_attributes"]},
        set_contexts={"common": ["entity_obj", "result_status", "notification_obj"]},
    )
    def create_notification_details(self, user, msg=None, projectId=None,
                                    entityId=None, entityType=None,
                                    event_name=None, event_details=None):
        """
        Description: Stage the steps required to create a notification for an entity.

        Input:
            - user: Acting user.
            - msg, projectId, entityId, entityType, event_name, event_details: Notification payload.

        Output:
            - None. The result/result_status are published into the orchestrator context.
        """
        # 1. Declare context schema up front so any step that writes these keys is allowed to.
        self.mainorchestrator.set_context_schema({
            "entity_obj":       MODEL_REGISTRY["Entity"],
            "entityId":         int,
            "entityType":       str,
            "notification_obj": MODEL_REGISTRY["Notification"],
        })

        # 2. Compose: ask the sibling orchestrator to stage its lookup step on the same mainorchestrator.
        self.entity_orchestrator.get_entity_details_using_projectId(projectId, entityId, entityType)

        # 3. Stage this domain's step.
        self.mainorchestrator.add_step(
            create_notification_bll,
            rollback_create_notification_bll,
            user=user,
            msg=msg,
            event_name=event_name,
            event_details=event_details,
            entityId=entityId,
            entityType=entityType,
        )
```

**Note on returns:** `create_notification_details` does **not** return the notification, an id, or anything else. It only registers steps. The notification ends up in the orchestrator's context (`set_context_value("notification_obj", ...)` inside the step).

### 2. The step function (in `bll/<Domain>/`)

```python
# bll/Alerts/PostApisbll.py
from Orchestrators.ContextManager import context_manager


@context_manager(
    get_contexts={"common": ["entity_obj", "updated_attributes"]},
    set_contexts={"common": ["result_status", "notification_obj"]},
)
def create_notification_bll(orchestrator, user, msg, event_name,
                            event_details, entityId, entityType):
    """The step's forward operation. Receives the MainOrchestrator as `orchestrator`."""
    entity_obj = orchestrator.get_context_value("entity_obj")
    updated_attributes = orchestrator.get_context_value("updated_attributes")
    # ... business logic + service.py calls ...
    noti = notifications_manager_obj.create_notification(...)
    orchestrator.set_context_value("notification_obj", noti)
    # No return value used by the framework.


@context_manager(
    get_contexts={"common": ["notification_obj"]},
    set_contexts={},
)
def rollback_create_notification_bll(orchestrator):
    """The step's rollback. Called in reverse order if any later step fails."""
    noti = orchestrator.get_context_value("notification_obj")
    notifications_manager_obj.get_notification(notification_id=noti.notification_id).delete()
```

### 3. Registration

Open `Orchestrators/RegisterOrchestrator.py` and add **one line** inside `register_orchestrator()`:

```python
# Orchestrators/RegisterOrchestrator.py
def register_orchestrator():
    OrchestratorRegistry.register("main", MainOrchestrator)
    # ... existing entries ...
    OrchestratorRegistry.register("alert", AlertOrchestrator)   # ← add here
```

There is **no `@register_orchestrator` decorator.** Registration is explicit and listed in one place so the boot sequence is easy to audit.

### 4. The view invokes it

```python
# VaultManagement/views/Alerts.py
class CreateNotification(ModelAPIView):
    def post(self, request):
        serializer = CreateNotificationSerializer(data=request.data)
        if not serializer.is_valid():
            return Response(status=400, data={"msg": flatten_serializer_errors(serializer.errors)})

        alert: AlertOrchestrator = OrchestratorRegistry.get("alert")
        alert.create_notification_details(request.user, **serializer.validated_data)
        alert.mainorchestrator.execute()
        return Response(data={"msg": "Notification created successfully"}, status=200)
```

The view does not capture a return value from `create_notification_details`. To shape the HTTP response from orchestrator-produced data, read the context after `execute()`:

```python
result = alert.mainorchestrator.get_context_value("result", check_validation=False)
status = alert.mainorchestrator.get_context_value("result_status", check_validation=False) or 200
return Response(data=result, status=status)
```

## How exit signalling works

A step communicates "we're done here" by writing `result_status` (and usually `result`) into the context. The `set_context_value` call for `result_status` raises `StepExitException`, which `execute()` catches:

| `result_status` value | Behaviour |
|---|---|
| Falsy / never set | `execute()` continues with the next step. |
| Truthy and **not** in `[400, 500]` | `execute()` stops immediately and returns `context["result"]`. **No rollback.** This is the "early success" path. |
| `400` or `500` | `execute()` triggers rollback of all already-run steps in reverse, then returns `context["result"]`. |
| Step raises an uncaught exception | Rollback chain runs, then the exception is re-raised. |

So step functions exit the workflow like this — not via `return`:

```python
def some_step(orchestrator, ...):
    if precondition_failed:
        orchestrator.set_context_value("result", {"msg": "not allowed"})
        orchestrator.set_context_value("result_status", 400)   # raises StepExitException → rollback
        return  # not reached
    # happy path
    orchestrator.set_context_value("result", payload)
    orchestrator.set_context_value("result_status", 200)       # raises StepExitException → exit, no rollback
```

## The `@context_manager` decorator

Every orchestrator method and every step function declares the context keys it touches. The full signature:

```python
context_manager(
    get_contexts=None,                  # dict, or callable returning a dict
    set_contexts=None,                  # dict, or callable returning a dict
    sync_with_parent_context=True,      # enforce that the parent declared these keys too
)
```

The simplest use:

```python
@context_manager(
    get_contexts={"common": ["entity_obj", "updated_attributes"]},
    set_contexts={"common": ["result_status", "notification_obj"]},
)
def create_notification_bll(orchestrator, ...):
    ...
```

Calling `get_context_value` or `set_context_value` on a key not in your declared lists raises. The declaration is also self-documenting: a reader can scan an orchestrator and see the data flow without reading every step body.

### Dict structure — `common`, `else`, and conditional branches

The dict supports three kinds of keys:

| Key shape | Meaning |
|---|---|
| `"common"` | Always allowed, regardless of context. The default bucket. |
| Any other string (e.g. `"is_bulk"`) | **Conditional bucket** — at runtime the framework reads the orchestrator-context value of this key via `get_context_value("is_bulk", check_validation=False)`. If that value is truthy, the listed keys are appended to the allowed set; if falsy, the bucket is skipped. |
| `"else"` | Allowed **only if no conditional bucket matched.** Useful for "default case" keys when one of several mutually-exclusive code paths fires. |

A worked example:

```python
@context_manager(
    get_contexts={
        "common":   ["user_id", "tenant"],            # always allowed
        "is_bulk":  ["batch_id", "row_index"],        # only when context["is_bulk"] is truthy
        "is_admin": ["admin_session"],                # only when context["is_admin"] is truthy
        "else":     ["request_payload"],              # only if neither is_bulk nor is_admin matched
    },
    set_contexts={
        "common":   ["result", "result_status"],
        "is_bulk":  ["batch_progress"],
    },
)
def my_step(orchestrator, ...):
    ...
```

The conditional buckets are evaluated against **orchestrator context values**, not the step's arguments. The pattern of choice for a function that's reused across multiple workflows where one of several mutually-exclusive flags is set upstream.

### Callable form (lambda or named function) — dynamic per-call dicts

When the allowed keys genuinely depend on the function's arguments at call time, pass a callable instead of a dict. The callable returns the dict.

It can accept the wrapped function's bound arguments in one of two ways:

1. **`**kwargs`-style — receives all bound args:**

   ```python
   @context_manager(
       get_contexts=lambda **kwargs: (
           {"common": ["entity_obj", "audit_payload"]}
           if kwargs.get("entityType") == "Document"
           else {"common": ["entity_obj"]}
       ),
       set_contexts={"common": ["result", "result_status"]},
   )
   def create_for_entity(self, user, entityId, entityType):
       ...
   ```

2. **Named-param-style — receives only the named args it declares** (the framework filters `bound_args` down to the callable's signature):

   ```python
   def _get_for_step(step):
       return {"common": ["common_payload"], step: ["scoped_payload"]}

   @context_manager(
       get_contexts=_get_for_step,           # framework passes step=... by name
       set_contexts={"common": ["result", "result_status"]},
   )
   def stage_step(self, step="Normal", ...):
       ...
   ```

Use the callable form **only** when the keys actually vary by input. A static dict is faster and easier to read; the callable form should be a deliberate choice, not a habit.

### `sync_with_parent_context` — propagation enforcement

Every staged step is called from somewhere (an orchestrator method, which itself is called from a view, a task, or another orchestrator method). When `sync_with_parent_context=True` (the default), the framework checks that **every key declared in this function's `get_contexts` / `set_contexts` is also declared in its caller's** — i.e. context must propagate down the call chain, it can't appear out of nowhere in a leaf function.

If you add a new key to a step's `get_contexts` but forget to add it to the orchestrator method that stages the step, you'll get:

```
KeyError: Context key 'foo' of common of get context of my_step
         not found in parent function context create_thing.
```

That's the framework enforcing that schema decisions are made at the workflow level, not snuck in deep in the stack.

#### Root-entry exemptions

Some functions are roots — they don't have an `@context_manager`-decorated parent. The framework recognises this set of HTTP-verb / lifecycle method names and exempts them from the parent check:

```
get, post, put, delete, execute,
inventory_migration, data_preparation_for_freeze_entity,
update_local_attributes, fix_gov_review, revert_workflow_checker_bll
```

(Defined as `allowed_methods` in `Orchestrators/ContextManager.py`.) If your view's HTTP method is named one of these, you don't have to wrap it in `@context_manager` and the first decorated frame down-stack doesn't trigger a parent-check error.

#### Opting out per-call: `sync_with_parent_context=False`

In rare cases, a function legitimately operates outside the normal call chain (a one-off worker entrypoint, a debugging shim, a backfill script). You can disable the check on that decorator:

```python
@context_manager(
    get_contexts={"common": ["entity_obj"]},
    set_contexts={"common": ["result_status"]},
    sync_with_parent_context=False,           # skip the "is this in parent?" check
)
def special_one_off_step(orchestrator, ...):
    ...
```

This is a **code smell** in normal application code. If you find yourself reaching for it more than rarely, the workflow shape is probably wrong — fix the parent declarations rather than disabling the check.

### Putting it all together

The three rules of thumb:

1. **Default to a static dict** with `"common"` only.
2. **Use conditional / `"else"` buckets** when a single function services multiple workflows where one of several flags is set upstream.
3. **Use a callable** only when the allowed keys are a function of the actual call arguments. Otherwise, write multiple small decorated functions.

Whichever shape you pick, both `get_contexts` and `set_contexts` are propagated up the call stack so the framework can fail loudly the first time someone tries to read or write a key that wasn't declared at every level above.

## Docstring requirement

`BaseOrchestrator` (the metaclass) **rejects the class at import time** if any class or method docstring is missing or malformed. Required sections:

| Item | Required sections |
|---|---|
| Class | `Description:` |
| Method | `Description:`, `Input:`, `Output:` |

Every section must be present and non-empty.

## Composition

Orchestrators can stage steps from other orchestrators. The pattern: resolve the sibling once in `__init__`, then call its method like you'd call your own. Its steps land on the **same** `mainorchestrator`, so a failure anywhere in the composite workflow rolls back the entire chain in reverse.

```python
def __init__(self, orchestrator_obj):
    """..."""
    self.mainorchestrator = orchestrator_obj or OrchestratorRegistry.get("main")
    self.entity_orchestrator = self.mainorchestrator.get_other_orchestrator(self, "entity")
    self.history_orchestrator = self.mainorchestrator.get_other_orchestrator(self, "history")

def some_workflow(self, ...):
    """..."""
    self.entity_orchestrator.load_entity(...)
    self.mainorchestrator.add_step(my_forward, my_rollback, ...)
    self.history_orchestrator.append_history_entry(...)
```

`get_other_orchestrator` defers resolution — if the sibling isn't yet registered at the moment of the call, it's queued and resolved later by `finialize_initialization()`. This avoids import-order headaches.

## When to add a new orchestrator

Create a new `<Domain>Orchestrators/` folder when:

- You're adding a new domain with multi-step write workflows.
- You need cross-domain composition (e.g., creating a document must also create an event + a notification).

A simple read-only endpoint that returns a queryset does **not** need an orchestrator — call `bll/<Domain>/GetApisbll.py` directly from the view.

## Existing domains

```
AlertOrchestrators        AuthOrchestrators       AutomationOrchestrator
BulkOrchestrators         CustomEntityOrchestrators
DocumentOrchestrators     EmailTemplateOrchestrators
EntityOrchestrators       EventOrchestrators
ExternalToolOrchestrators GlobalAttributeOrchestrators
HistoryOrchestrators      ImpactEngineOrchestrators
LockOrchestrators         ReportOrchestrators
RulesOrchestrators        TeamOrchestrators
TemporalOrchestrators     ThemeOrchestrators
VersioningOrchestrators   WorkflowOrchestrators
```

All entry points are listed in `Orchestrators/RegisterOrchestrator.py::register_orchestrator()`.
