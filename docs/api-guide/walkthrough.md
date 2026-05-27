# Worked Example — Adding a "Bookmark" Domain

This chapter walks through adding a brand-new domain end-to-end. The goal: a `Bookmark` feature with three endpoints — create, list, delete. Every layer of the contract gets exercised.

## What we're building

| Endpoint | Method | Path |
|---|---|---|
| Create a bookmark | POST | `/vault/bookmark/create` |
| List my bookmarks | GET | `/vault/bookmark/getAll` |
| Delete a bookmark | DELETE | `/vault/bookmark/delete` |

## Step 0 — Model

```python
# VaultModels/models/Bookmark.py
import uuid
from django.db import models
from VaultModels.models import User


class Bookmark(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    owner = models.ForeignKey(User, on_delete=models.CASCADE, related_name="bookmarks")
    target_url = models.URLField(max_length=2048)
    title = models.CharField(max_length=255)
    note = models.TextField(blank=True, default="")
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ("-created_at",)
        unique_together = [("owner", "target_url")]
```

Run:

```bash
python manage.py makemigrations
python manage.py migrate
```

Register the model in `VaultModels/ModelRegistry.py` if dynamic lookup is needed (rules, formulas, bulk-upload). For this example, not required.

## Step 1 — `service.py`

The only file that touches the model.

```python
# bll/Bookmark/service.py
from typing import Optional, Iterable, Any
from django.db.models import QuerySet
from VaultModels.models.Bookmark import Bookmark


class BookmarkService:
    @staticmethod
    def create(**fields) -> Bookmark:
        return Bookmark.objects.create(**fields)

    @staticmethod
    def get(**filters) -> Optional[Bookmark]:
        return Bookmark.objects.filter(**filters).first()

    @staticmethod
    def filter(**filters) -> QuerySet:
        return Bookmark.objects.filter(**filters)

    @staticmethod
    def update(instance: Bookmark, **fields) -> Bookmark:
        for k, v in fields.items():
            setattr(instance, k, v)
        instance.save(update_fields=list(fields.keys()))
        return instance

    @staticmethod
    def delete(instance_or_id: Any) -> bool:
        if isinstance(instance_or_id, Bookmark):
            instance_or_id.delete()
            return True
        deleted, _ = Bookmark.objects.filter(pk=instance_or_id).delete()
        return deleted > 0

    @staticmethod
    def bulk_create(rows: Iterable) -> list[Bookmark]:
        instances = [r if isinstance(r, Bookmark) else Bookmark(**r) for r in rows]
        return Bookmark.objects.bulk_create(instances)

    @staticmethod
    def bulk_update(rows: Iterable[Bookmark], fields: list[str]) -> int:
        return Bookmark.objects.bulk_update(list(rows), fields)

    @staticmethod
    def bulk_delete(filters_or_ids) -> int:
        if isinstance(filters_or_ids, dict):
            deleted, _ = Bookmark.objects.filter(**filters_or_ids).delete()
        else:
            deleted, _ = Bookmark.objects.filter(pk__in=filters_or_ids).delete()
        return deleted


bookmark_service = BookmarkService()
```

## Step 2 — Helpers

```python
# bll/Bookmark/HelperFunctions.py
from urllib.parse import urlparse
from VaultErrors.BackendErrors import errors


def validate_bookmark_url(url: str) -> None:
    parsed = urlparse(url)
    if parsed.scheme not in ("http", "https") or not parsed.netloc:
        raise errors.bad_request.ValidationError("Bookmark URL must be an http(s) URL.")


def format_bookmark(b) -> dict:
    return {
        "id": str(b.id),
        "title": b.title,
        "target_url": b.target_url,
        "note": b.note,
        "created_at": b.created_at.isoformat(),
    }
```

## Step 3 — Verb-split BLL

`GetApisbll.py` is plain top-down read logic. `PostApisbll.py` contains the **step functions** that the orchestrator will register — they take the `MainOrchestrator` as their first argument and publish their outcome into the orchestrator context.

```python
# bll/Bookmark/GetApisbll.py
from bll.Bookmark.service import bookmark_service
from bll.Bookmark.HelperFunctions import format_bookmark


def list_bookmarks(user, search=None, limit=50, offset=0):
    qs = bookmark_service.filter(owner=user)
    if search:
        qs = qs.filter(title__icontains=search)
    qs = qs.order_by("-created_at")
    rows = list(qs[offset : offset + limit])
    return {
        "results": [format_bookmark(b) for b in rows],
        "total": qs.count(),
    }
```

```python
# bll/Bookmark/PostApisbll.py
from Orchestrators.ContextManager import context_manager
from bll.Bookmark.service import bookmark_service
from bll.Bookmark.HelperFunctions import validate_bookmark_url, format_bookmark


@context_manager(
    get_contexts={"common": []},
    set_contexts={"common": ["result", "result_status", "bookmark_obj"]},
)
def create_bookmark_bll(orchestrator, user, target_url, title, note):
    """Forward step for creating a bookmark."""
    validate_bookmark_url(target_url)

    if bookmark_service.get(owner=user, target_url=target_url) is not None:
        orchestrator.set_context_value("result", {"msg": "Bookmark for this URL already exists."})
        orchestrator.set_context_value("result_status", 400)         # → rollback + return
        return

    bookmark = bookmark_service.create(
        owner=user, target_url=target_url, title=title, note=note,
    )
    orchestrator.set_context_value("bookmark_obj", bookmark)
    orchestrator.set_context_value("result", format_bookmark(bookmark))
    orchestrator.set_context_value("result_status", 201)             # → success exit


@context_manager(
    get_contexts={"common": ["bookmark_obj"]},
    set_contexts={},
)
def rollback_create_bookmark_bll(orchestrator):
    """Rollback: drop the bookmark we just created."""
    bookmark = orchestrator.get_context_value("bookmark_obj")
    bookmark_service.delete(bookmark)


@context_manager(
    get_contexts={"common": []},
    set_contexts={"common": ["result", "result_status", "bookmark_snapshot"]},
)
def delete_bookmark_bll(orchestrator, user, bookmark_id):
    """Forward step for deleting a bookmark."""
    bookmark = bookmark_service.get(id=bookmark_id, owner=user)
    if bookmark is None:
        orchestrator.set_context_value("result", {"msg": "Bookmark not found."})
        orchestrator.set_context_value("result_status", 404)
        return

    # Snapshot enough state to recreate the row if a downstream step fails.
    orchestrator.set_context_value("bookmark_snapshot", {
        "owner":      bookmark.owner,
        "target_url": bookmark.target_url,
        "title":      bookmark.title,
        "note":       bookmark.note,
    })
    bookmark_service.delete(bookmark)
    orchestrator.set_context_value("result", {"id": str(bookmark_id), "deleted": True})
    orchestrator.set_context_value("result_status", 200)


@context_manager(
    get_contexts={"common": ["bookmark_snapshot"]},
    set_contexts={},
)
def rollback_delete_bookmark_bll(orchestrator):
    """Rollback: recreate the deleted bookmark from the snapshot."""
    snapshot = orchestrator.get_context_value("bookmark_snapshot")
    bookmark_service.create(**snapshot)
```

## Step 4 — Orchestrator

`create` and `delete` are write paths, so they go through the orchestrator. `list` is read-only and can skip it.

The orchestrator class only **stages steps**. It does not run them, does not return business values, and does not own any state about the work in flight — all of that lives in the `MainOrchestrator` context.

```python
# Orchestrators/BookmarkOrchestrators/__init__.py   ← empty file

# Orchestrators/BookmarkOrchestrators/BookmarkOrchestrator.py
from Orchestrators.OrchestratorRegistry import OrchestratorRegistry
from Orchestrators.AbstractOrchestrators.BaseOrchestrator import BaseOrchestrator
from Orchestrators.ContextManager import context_manager
from VaultModels.ModelRegistry import MODEL_REGISTRY

from bll.Bookmark.PostApisbll import (
    create_bookmark_bll, rollback_create_bookmark_bll,
    delete_bookmark_bll, rollback_delete_bookmark_bll,
)


class BookmarkOrchestrator(metaclass=BaseOrchestrator):
    """
    Description:
    Orchestrates Bookmark write workflows (create + delete).
    """

    def __init__(self, orchestrator_obj):
        """
        Description: Bind to the shared MainOrchestrator.

        Input:
            - orchestrator_obj: The MainOrchestrator instance bound to this run.

        Output:
            - None.
        """
        self.mainorchestrator = orchestrator_obj or OrchestratorRegistry.get("main")

    @context_manager(
        get_contexts={"common": []},
        set_contexts={"common": ["result", "result_status", "bookmark_obj"]},
    )
    def stage_create(self, user, target_url, title, note=""):
        """
        Description: Stage the create-bookmark workflow.

        Input:
            - user: Acting user.
            - target_url, title, note: New bookmark fields.

        Output:
            - None. Outcome is published into the orchestrator context.
        """
        self.mainorchestrator.set_context_schema({
            "bookmark_obj": MODEL_REGISTRY["Bookmark"],
        })
        self.mainorchestrator.add_step(
            create_bookmark_bll,
            rollback_create_bookmark_bll,
            user=user,
            target_url=target_url,
            title=title,
            note=note,
        )

    @context_manager(
        get_contexts={"common": []},
        set_contexts={"common": ["result", "result_status", "bookmark_snapshot"]},
    )
    def stage_delete(self, user, bookmark_id):
        """
        Description: Stage the delete-bookmark workflow.

        Input:
            - user: Acting user.
            - bookmark_id: PK of the bookmark to delete.

        Output:
            - None. Outcome is published into the orchestrator context.
        """
        self.mainorchestrator.set_context_schema({
            "bookmark_snapshot": dict,
        })
        self.mainorchestrator.add_step(
            delete_bookmark_bll,
            rollback_delete_bookmark_bll,
            user=user,
            bookmark_id=bookmark_id,
        )
```

Register it in **one** place — `Orchestrators/RegisterOrchestrator.py`:

```python
# Orchestrators/RegisterOrchestrator.py
from Orchestrators.BookmarkOrchestrators.BookmarkOrchestrator import BookmarkOrchestrator

def register_orchestrator():
    # ... existing entries ...
    OrchestratorRegistry.register("bookmark", BookmarkOrchestrator)
```

## Step 5 — Serializers

```python
# VaultManagement/serializers/Bookmark.py
from rest_framework import serializers


class CreateBookmarkSerializer(serializers.Serializer):
    target_url = serializers.URLField(max_length=2048)
    title = serializers.CharField(max_length=255)
    note = serializers.CharField(required=False, allow_blank=True, default="")


class ListBookmarksQuerySerializer(serializers.Serializer):
    search = serializers.CharField(required=False, allow_blank=True)
    limit = serializers.IntegerField(min_value=1, max_value=200, default=50)
    offset = serializers.IntegerField(min_value=0, default=0)


class DeleteBookmarkSerializer(serializers.Serializer):
    bookmark_id = serializers.UUIDField()
```

## Step 6 — Views

```python
# VaultManagement/views/Bookmark.py
from NimbusVaultConstants.CommonConstants.CommonConstants import (
    ModelAPIView,
    flatten_serializer_errors,
)
from rest_framework.response import Response

from VaultManagement.serializers.Bookmark import (
    CreateBookmarkSerializer,
    ListBookmarksQuerySerializer,
    DeleteBookmarkSerializer,
)
from Orchestrators.OrchestratorRegistry import OrchestratorRegistry
from Orchestrators.BookmarkOrchestrators.BookmarkOrchestrator import BookmarkOrchestrator


def _orch_response(orch, default_status=200):
    """Pull result/result_status from the MainOrchestrator context."""
    result = orch.mainorchestrator.get_context_value("result", check_validation=False)
    status = orch.mainorchestrator.get_context_value("result_status", check_validation=False)
    return Response(data=result, status=status or default_status)


class BookmarkCreate(ModelAPIView):
    def post(self, request):
        serializer = CreateBookmarkSerializer(data=request.data)
        if not serializer.is_valid():
            return Response(
                status=400,
                data={"msg": flatten_serializer_errors(serializer.errors)},
            )
        orch: BookmarkOrchestrator = OrchestratorRegistry.get("bookmark")
        orch.stage_create(user=request.user, **serializer.validated_data)   # stages steps
        orch.mainorchestrator.execute()                                     # runs them
        return _orch_response(orch, default_status=201)


class BookmarkList(ModelAPIView):
    def get(self, request):
        serializer = ListBookmarksQuerySerializer(data=request.query_params)
        if not serializer.is_valid():
            return Response(
                status=400,
                data={"msg": flatten_serializer_errors(serializer.errors)},
            )
        from bll.Bookmark.GetApisbll import list_bookmarks
        return Response(data=list_bookmarks(user=request.user, **serializer.validated_data), status=200)


class BookmarkDelete(ModelAPIView):
    def delete(self, request):
        bookmark_id = request.GET.get("bookmark_id")
        serializer = DeleteBookmarkSerializer(data={"bookmark_id": bookmark_id})
        if not serializer.is_valid():
            return Response(
                status=400,
                data={"msg": flatten_serializer_errors(serializer.errors)},
            )
        orch: BookmarkOrchestrator = OrchestratorRegistry.get("bookmark")
        orch.stage_delete(user=request.user, bookmark_id=serializer.validated_data["bookmark_id"])
        orch.mainorchestrator.execute()
        return _orch_response(orch, default_status=200)
```

## Step 7 — URLs

```python
# VaultManagement/urls/Bookmark.py
from django.urls import path
from VaultManagement.views.Bookmark import BookmarkCreate, BookmarkList, BookmarkDelete

urlpatterns = [
    path("bookmark/create", BookmarkCreate.as_view()),
    path("bookmark/getAll", BookmarkList.as_view()),
    path("bookmark/delete", BookmarkDelete.as_view()),
]
```

And register the new module in the root urls:

```python
# NimbusVault/urls.py — add one line
path(ModelName.url_vault, include("VaultManagement.urls.Bookmark")),
```

## What you just built — the call graph

```
POST /vault/bookmark/create
  → BookmarkCreate.post                       (view)
    → CreateBookmarkSerializer                (serializer)
    → OrchestratorRegistry.get("bookmark")    (registry builds MainOrchestrator + BookmarkOrchestrator)
      → BookmarkOrchestrator.stage_create     (declares context schema + add_step)
                                                  • forward  = create_bookmark_bll
                                                  • rollback = rollback_create_bookmark_bll
    → orch.mainorchestrator.execute()         (now the staged step actually runs)
        ↳ create_bookmark_bll(orch, user, target_url, title, note)
          ↳ validate_bookmark_url             (bll/Bookmark/HelperFunctions.py)
          ↳ bookmark_service.get              (bll/Bookmark/service.py)
          ↳ bookmark_service.create           (bll/Bookmark/service.py)
          ↳ orchestrator.set_context_value("bookmark_obj", ...)
          ↳ orchestrator.set_context_value("result", format_bookmark(...))
          ↳ orchestrator.set_context_value("result_status", 201)  ← raises StepExitException → exit
  → _orch_response(orch, 201)
      reads result + result_status from orchestrator context
  → Response(data, status)
```

Every layer is tiny. No layer skips. The only thing that touches `Bookmark.objects` anywhere in this whole stack is `bll/Bookmark/service.py` — exactly as the contract requires. The orchestrator method has no return value; the view reads the workflow outcome from the orchestrator context.

## Testing the new endpoints

```python
# VaultManagement/tests/test_bookmark_api.py
import pytest
from rest_framework.test import APIClient


@pytest.mark.django_db
def test_create_and_list_bookmark(authed_client: APIClient):
    resp = authed_client.post("/vault/bookmark/create", data={
        "target_url": "https://example.com",
        "title": "Example",
    }, format="json")
    assert resp.status_code == 201

    resp = authed_client.get("/vault/bookmark/getAll")
    assert resp.status_code == 200
    assert resp.json()["total"] == 1
```

(`authed_client` is a fixture conventionally provided in `conftest.py` that sets `Authorization: JWT ...` and `Org: <slug>` headers.)
