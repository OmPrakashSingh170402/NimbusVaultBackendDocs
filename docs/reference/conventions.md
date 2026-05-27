# Conventions

## Naming

| Concept | Convention | Example |
|---|---|---|
| URL path segment | camelCase or kebab-case (match neighbours) | `getAll`, `markAsRead`, `power-bi-spec` |
| Django model | PascalCase singular | `Bookmark`, `TemplateVersionLink` |
| Service class | `<Model>Service` | `BookmarkService` |
| Service singleton | `<snake_model>_service` | `bookmark_service` |
| Orchestrator class | `<Domain>Orchestrator` | `BookmarkOrchestrator` |
| Orchestrator registry key | `snake_case` | `"bookmark"`, `"alert"` |
| Serializer class | `<Action><Resource>Serializer` | `CreateBookmarkSerializer`, `ListBookmarksQuerySerializer` |
| View class | `<Action><Resource>` or `<Resource><Action>` (match neighbours) | `BookmarkCreate`, `GetAnnouncements` |

## Headers

| Header | Value | Purpose |
|---|---|---|
| `Authorization` | `JWT <token>` | Auth. **Note**: the scheme prefix is `JWT`, not `Bearer`. |
| `Org` | `<tenant slug>` | Routes the request to the tenant DB. Required for every non-public endpoint. |
| `Content-Type` | `application/json` | Default for all JSON endpoints. |

## Errors

```python
from VaultErrors.BackendErrors import errors

raise errors.bad_request.ValidationError("...")     # 400
raise errors.bad_request.NotFound("...")            # 404
raise errors.bad_request.PermissionDenied("...")    # 403
# See VaultErrors/BackendErrors.py for the full taxonomy.
```

DRF turns these into JSON responses with the right HTTP status. Never raise plain `Exception` or DRF built-ins from inside the BLL.

## Imports

- Absolute imports only. No `from .HelperFunctions import …`. Always `from bll.<Domain>.HelperFunctions import …`.
- Group: stdlib → third-party → Django → first-party. Blank line between groups.
- Don't import Django models outside `service.py`.

## Logging

```python
import logging
logger = logging.getLogger(__name__)

logger.info("Created bookmark", extra={"user_id": user.id, "bookmark_id": str(b.id)})
```

- Use `extra={}` for structured fields. Don't f-string everything into the message.
- Log at the BLL layer, not in `service.py`.
- INFO for normal lifecycle events; WARNING for recoverable anomalies; ERROR only for genuine failures.

## Type hints

- Required on all new public functions in `service.py`, `GetApisbll.py`, `PostApisbll.py`, orchestrator methods.
- Use `Optional[X]` not `Union[X, None]`.
- Use `list[X]` / `dict[str, X]` (PEP 585) on Python 3.10.

## Docstrings

- One-line summary on every public function. Multi-line only when the parameter contract is non-obvious.
- Reference the contract: if a function is part of the data-access primitive set, say so.

## Comments

- Default to writing no comments. Only add one when the **why** is non-obvious: a hidden constraint, a subtle invariant, a workaround for a specific bug.
- Don't explain **what** the code does — well-named identifiers do that.
- Don't reference current tasks/PRs/issue numbers — they rot.

## Migrations

- One migration per logical schema change.
- Never edit a merged migration. Add a follow-up.
- For columns with non-trivial defaults on large tables, follow the standard add-nullable → backfill → set NOT NULL pattern across separate migrations.

## Feature flags

Toggleable behaviour goes through `VaultSettings.json` `FeatureFlags`:

```python
from NimbusVault.settings import vault_settings

if vault_settings.get("FeatureFlags", {}).get("EnableMyNewThing"):
    ...
```

When you add a flag, write down the condition that will let us remove it.

## Plugins

- One folder per plugin under `Plugins/<Name>/`.
- Always has `urls.py`.
- Internal layout mirrors the core: `views/`, `serializers/`, `bll/<SubDomain>/{HelperFunctions, GetApisbll, PostApisbll, service}.py`.
- Enabled via `Base.active_plugins` in `VaultSettings.json`.
