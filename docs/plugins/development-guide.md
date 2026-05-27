# Plugin Development Guide

A step-by-step recipe for adding a new plugin. The goal: a plugin called `Bookmarks_Plugin` that exposes a single endpoint (`POST /vault/bookmarks/share`) and reacts to a core lifecycle hook (`on_entity_archived`). Every step ties back to the [API Creation Contract](../api-guide/contract.md) — plugins follow the same rules as the core.

## Step 0 — Decide the plugin shape

Ask three questions before writing any code:

1. **Does this need HTTP endpoints?** If yes, you need `urls.py` + `views/` + `serializers/`.
2. **Does this hook into core events?** If yes, you need a `<PluginName>.py` file with `@hookimpl` methods.
3. **Does this own data?** If yes, you need `models.py` + `migrations/` + a `service.py` per model.

A plugin can do any combination — there's no "you must implement X". The minimum viable plugin is just a `urls.py` with one route.

## Step 1 — Scaffold the folder

```
Plugins/Bookmarks_Plugin/
├── __init__.py
├── apps.py                    # if the plugin owns Django models
├── urls.py                    # required for HTTP endpoints
├── Bookmarks_Plugin.py        # required for pluggy hook registration
├── views.py
├── serializers.py
├── HelperFunctions.py
├── CommonConstants.py
├── bll/
│   └── Bookmarks/
│       ├── __init__.py
│       ├── HelperFunctions.py
│       ├── GetApisbll.py
│       ├── PostApisbll.py
│       └── service.py
└── models.py                  # only if the plugin owns models
```

The two **required** files for a plugin to be discoverable by the framework are:

- **`Plugins/Bookmarks_Plugin/urls.py`** — picked up by `NimbusVault/urls.py`'s `for plugin in settings.ACTIVE_PLUGINS:` loop.
- **`Plugins/Bookmarks_Plugin/Bookmarks_Plugin.py`** — picked up by `Plugins/plugin_manager.py`'s `pm.register(...)` call. The file name **must match** the folder name exactly. The class inside must also be named `Bookmarks_Plugin`.

If your plugin doesn't implement any pluggy hooks, `Bookmarks_Plugin.py` can be a near-empty class (see Step 5).

## Step 2 — Model + service

If your plugin owns data, add `Plugins/Bookmarks_Plugin/models.py`, `apps.py`, and a migrations folder, then write a `service.py` that obeys the [data-layer rule](../architecture/data-layer.md): only `create`, `get`, `filter`, `update`, `delete`, `bulk_create`, `bulk_update`, `bulk_delete`.

```python
# Plugins/Bookmarks_Plugin/models.py
import uuid
from django.db import models
from VaultModels.models import User


class SharedBookmark(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    shared_by = models.ForeignKey(User, on_delete=models.CASCADE, related_name="shared_bookmarks")
    target_url = models.URLField(max_length=2048)
    title = models.CharField(max_length=255)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        app_label = "Bookmarks_Plugin"
        indexes = [models.Index(fields=["shared_by", "-created_at"])]
```

```python
# Plugins/Bookmarks_Plugin/apps.py
from django.apps import AppConfig


class BookmarksPluginConfig(AppConfig):
    name = "Plugins.Bookmarks_Plugin"
    label = "Bookmarks_Plugin"
```

```python
# Plugins/Bookmarks_Plugin/bll/Bookmarks/service.py
from typing import Optional, Iterable, Any
from django.db.models import QuerySet
from Plugins.Bookmarks_Plugin.models import SharedBookmark


class SharedBookmarkService:
    @staticmethod
    def create(**fields) -> SharedBookmark:
        return SharedBookmark.objects.create(**fields)

    @staticmethod
    def get(**filters) -> Optional[SharedBookmark]:
        return SharedBookmark.objects.filter(**filters).first()

    @staticmethod
    def filter(**filters) -> QuerySet:
        return SharedBookmark.objects.filter(**filters)

    @staticmethod
    def update(instance, **fields) -> SharedBookmark:
        for k, v in fields.items():
            setattr(instance, k, v)
        instance.save(update_fields=list(fields.keys()))
        return instance

    @staticmethod
    def delete(instance_or_id: Any) -> bool:
        if isinstance(instance_or_id, SharedBookmark):
            instance_or_id.delete()
            return True
        deleted, _ = SharedBookmark.objects.filter(pk=instance_or_id).delete()
        return deleted > 0

    @staticmethod
    def bulk_create(rows) -> list:
        objs = [r if isinstance(r, SharedBookmark) else SharedBookmark(**r) for r in rows]
        return SharedBookmark.objects.bulk_create(objs)

    @staticmethod
    def bulk_update(rows: Iterable, fields: list[str]) -> int:
        return SharedBookmark.objects.bulk_update(list(rows), fields)

    @staticmethod
    def bulk_delete(filters_or_ids) -> int:
        if isinstance(filters_or_ids, dict):
            deleted, _ = SharedBookmark.objects.filter(**filters_or_ids).delete()
        else:
            deleted, _ = SharedBookmark.objects.filter(pk__in=filters_or_ids).delete()
        return deleted


shared_bookmark_service = SharedBookmarkService()
```

If the plugin owns models, run migrations (`makemigrations` + `migrate`) after adding the app to `INSTALLED_APPS` — Django picks plugin apps up automatically when they're listed in `ACTIVE_PLUGINS` because `NimbusVault/settings.py` appends them.

## Step 3 — Helpers + verb-split BLL

```python
# Plugins/Bookmarks_Plugin/bll/Bookmarks/HelperFunctions.py
from urllib.parse import urlparse
from VaultErrors.BackendErrors import errors


def validate_target_url(url: str) -> None:
    parsed = urlparse(url)
    if parsed.scheme not in ("http", "https") or not parsed.netloc:
        raise errors.bad_request.ValidationError("Bookmark URL must be an http(s) URL.")


def format_shared_bookmark(b) -> dict:
    return {
        "id": str(b.id),
        "title": b.title,
        "target_url": b.target_url,
        "shared_by": b.shared_by.username,
        "created_at": b.created_at.isoformat(),
    }
```

```python
# Plugins/Bookmarks_Plugin/bll/Bookmarks/PostApisbll.py
from Orchestrators.ContextManager import context_manager
from Plugins.Bookmarks_Plugin.bll.Bookmarks.service import shared_bookmark_service
from Plugins.Bookmarks_Plugin.bll.Bookmarks.HelperFunctions import (
    validate_target_url, format_shared_bookmark,
)


@context_manager(
    get_contexts={"common": []},
    set_contexts={"common": ["result", "result_status", "shared_bookmark_obj"]},
)
def share_bookmark_bll(orchestrator, user, target_url, title):
    validate_target_url(target_url)
    bookmark = shared_bookmark_service.create(
        shared_by=user, target_url=target_url, title=title,
    )
    orchestrator.set_context_value("shared_bookmark_obj", bookmark)
    orchestrator.set_context_value("result", format_shared_bookmark(bookmark))
    orchestrator.set_context_value("result_status", 201)


@context_manager(get_contexts={"common": ["shared_bookmark_obj"]}, set_contexts={})
def rollback_share_bookmark_bll(orchestrator):
    bookmark = orchestrator.get_context_value("shared_bookmark_obj")
    shared_bookmark_service.delete(bookmark)
```

## Step 4 — Orchestrator (if the plugin has multi-step writes)

If your write workflow is a single forward + rollback, it's still worth wiring an orchestrator so that downstream features can compose. The orchestrator lives **outside** the plugin folder, in `Orchestrators/`:

```python
# Orchestrators/BookmarksOrchestrators/BookmarksOrchestrator.py
from Orchestrators.OrchestratorRegistry import OrchestratorRegistry
from Orchestrators.AbstractOrchestrators.BaseOrchestrator import BaseOrchestrator
from Orchestrators.ContextManager import context_manager
from VaultModels.ModelRegistry import MODEL_REGISTRY

from Plugins.Bookmarks_Plugin.bll.Bookmarks.PostApisbll import (
    share_bookmark_bll, rollback_share_bookmark_bll,
)


class BookmarksOrchestrator(metaclass=BaseOrchestrator):
    """
    Description:
    Orchestrates Bookmarks_Plugin write workflows.
    """

    def __init__(self, orchestrator_obj):
        """
        Description: Bind to the shared MainOrchestrator.

        Input:
            - orchestrator_obj: MainOrchestrator instance for this run.

        Output:
            - None.
        """
        self.mainorchestrator = orchestrator_obj or OrchestratorRegistry.get("main")

    @context_manager(
        get_contexts={"common": []},
        set_contexts={"common": ["result", "result_status", "shared_bookmark_obj"]},
    )
    def stage_share(self, user, target_url, title):
        """
        Description: Stage the share-bookmark workflow.

        Input:
            - user: Acting user.
            - target_url, title: Bookmark fields.

        Output:
            - None. Outcome lands in the orchestrator context.
        """
        self.mainorchestrator.set_context_schema({
            "shared_bookmark_obj": MODEL_REGISTRY.get("SharedBookmark"),
        })
        self.mainorchestrator.add_step(
            share_bookmark_bll, rollback_share_bookmark_bll,
            user=user, target_url=target_url, title=title,
        )
```

Register it with one line in `Orchestrators/RegisterOrchestrator.py`:

```python
from Orchestrators.BookmarksOrchestrators.BookmarksOrchestrator import BookmarksOrchestrator
# inside register_orchestrator():
OrchestratorRegistry.register("bookmarks", BookmarksOrchestrator)
```

## Step 5 — Pluggy hook registration

Even if you don't implement any hooks, `Plugins/Bookmarks_Plugin/Bookmarks_Plugin.py` must exist because `plugin_manager.py` registers it. The minimum is a class named exactly `Bookmarks_Plugin`:

```python
# Plugins/Bookmarks_Plugin/Bookmarks_Plugin.py
import pluggy

hookimpl = pluggy.HookimplMarker("vault")


class Bookmarks_Plugin:
    """Hook implementations for Bookmarks_Plugin."""

    @hookimpl
    def on_entity_archived(self, entity_type, entity_id):
        """Clean up shared bookmarks pointing at an archived entity."""
        from Plugins.Bookmarks_Plugin.bll.Bookmarks.service import shared_bookmark_service
        shared_bookmark_service.bulk_delete({"target_url__contains": f"/entity/{entity_id}"})
```

The hookspec it implements lives in `Plugins/Hooks/EntityHooks.py` (or wherever the spec for `on_entity_archived` lives). If you're inventing a new hook, add the spec to `Plugins/Hooks/` first:

```python
# Plugins/Hooks/EntityHooks.py
import pluggy
hookspec = pluggy.HookspecMarker("vault")


class EntityHooks:
    @hookspec
    def on_entity_archived(self, entity_type, entity_id):
        """Fired after an entity is archived. Implementations receive the entity type and id."""
```

The core then fires the hook via:

```python
from Plugins.plugin_manager import plugin_manager
plugin_manager.hook.on_entity_archived(entity_type="Document", entity_id=doc.id)
```

Pluggy fans the call out to every registered implementation, in plugin-registration order.

## Step 6 — Serializer + view

```python
# Plugins/Bookmarks_Plugin/serializers.py
from rest_framework import serializers


class ShareBookmarkSerializer(serializers.Serializer):
    target_url = serializers.URLField(max_length=2048)
    title = serializers.CharField(max_length=255)
```

```python
# Plugins/Bookmarks_Plugin/views.py
from NimbusVaultConstants.CommonConstants.CommonConstants import (
    ModelAPIView,
    flatten_serializer_errors,
)
from rest_framework.response import Response

from Plugins.Bookmarks_Plugin.serializers import ShareBookmarkSerializer
from Orchestrators.OrchestratorRegistry import OrchestratorRegistry
from Orchestrators.BookmarksOrchestrators.BookmarksOrchestrator import BookmarksOrchestrator


class ShareBookmark(ModelAPIView):
    """Share a bookmark with the rest of the tenant."""

    def post(self, request):
        serializer = ShareBookmarkSerializer(data=request.data)
        if not serializer.is_valid():
            return Response(
                status=400,
                data={"msg": flatten_serializer_errors(serializer.errors)},
            )
        orch: BookmarksOrchestrator = OrchestratorRegistry.get("bookmarks")
        orch.stage_share(user=request.user, **serializer.validated_data)
        orch.mainorchestrator.execute()

        result = orch.mainorchestrator.get_context_value("result", check_validation=False)
        status = orch.mainorchestrator.get_context_value("result_status", check_validation=False) or 201
        return Response(data=result, status=status)
```

## Step 7 — URLs

```python
# Plugins/Bookmarks_Plugin/urls.py
from django.urls import path
from Plugins.Bookmarks_Plugin.views import ShareBookmark

urlpatterns = [
    path("bookmarks/share", ShareBookmark.as_view()),
]
```

## Step 8 — Activate

Edit `VaultSettings.json`:

```json
{
  "Base": {
    "active_plugins": [
      "LLM_Plugin",
      "Nimbus_Plugin",
      "SharePoint",
      "PowerBI",
      "TeamsPlugin",
      "Bookmarks_Plugin"
    ]
  }
}
```

Restart the server. `POST /vault/bookmarks/share` is now live, and `on_entity_archived` hook calls reach `Bookmarks_Plugin.on_entity_archived` automatically.

## Plugin-development checklist

Before merging a new plugin PR:

- [ ] Plugin folder name, file name (`Plugins/<Name>/<Name>.py`), and class name match exactly.
- [ ] `urls.py` exists if HTTP endpoints are exposed.
- [ ] Class lives in `Plugins/<Name>/<Name>.py` and is named `<Name>`.
- [ ] If the plugin owns data: `models.py` + `apps.py` + migrations + a `service.py` that exposes **only** the eight CRUD primitives.
- [ ] BLL split into `HelperFunctions.py`, `GetApisbll.py`, `PostApisbll.py` — same rule as core.
- [ ] Views inherit from `ModelAPIView` and read `result` / `result_status` from orchestrator context.
- [ ] Orchestrator (if any) registered in `Orchestrators/RegisterOrchestrator.py`.
- [ ] Hookspec registered in `Plugins/Hooks/` if introducing a new hook.
- [ ] Tenant-aware: cache keys, ES indices, and Celery tasks all carry the org slug.
- [ ] Listed in `VaultSettings.json` `Base.active_plugins` in **all** environments where it should be on.
- [ ] Plugin-local config in `VaultSettings.json` under a dedicated key (e.g., `"Bookmarks": { ... }`), not hardcoded.
- [ ] No FK from a core model to a plugin model. The dependency arrow points plugin → core only.
- [ ] Listed in [Plugins → Active plugins](active-plugins.md).
