# Request Lifecycle

Every authenticated HTTP request lands in DRF the same way. Knowing this pipeline cold is what makes debugging tractable.

## Pipeline

```
Client                                                  Server
──────                                                  ──────
HTTP request ──────────────────────────────────►  ┌──────────────────────┐
   (Org, Authorization, body)                     │  Middleware stack    │
                                                  │   CORS               │
                                                  │   CSP                │
                                                  │   Tenant  (Org:)     │  ← swaps DB
                                                  │   Encryption         │
                                                  │   JWT/OAuth2         │  ← request.user
                                                  │   Elastic APM        │
                                                  └──────────┬───────────┘
                                                             │
                                                  ┌──────────▼───────────┐
                                                  │  URL router          │
                                                  │  NimbusVault/urls.py │
                                                  └──────────┬───────────┘
                                                             │
                                                  ┌──────────▼───────────┐
                                                  │  DRF view            │
                                                  │  (method per verb)   │
                                                  └──────────┬───────────┘
                                                             │ data=request.data
                                                  ┌──────────▼───────────┐
                                                  │  Serializer          │
                                                  │  .is_valid()         │  ── 400 on fail
                                                  └──────────┬───────────┘
                                                             │ validated_data
                                                  ┌──────────▼───────────┐
                                                  │  Orchestrator        │
                                                  │  Registry.get(...)   │
                                                  │  .stage_*()          │
                                                  └──────────┬───────────┘
                                                             │ composes steps
                                                  ┌──────────▼───────────┐
                                                  │  bll/<Domain>/       │
                                                  │   HelperFunctions    │ ← pure
                                                  │   GetApisbll  /      │
                                                  │   PostApisbll        │ ← business
                                                  └──────────┬───────────┘
                                                             │
                                                  ┌──────────▼───────────┐
                                                  │  service.py          │ ← ONLY place
                                                  │  CRUD primitives     │   that calls
                                                  └──────────┬───────────┘   the ORM
                                                             │ Model.objects.*
                                                  ┌──────────▼───────────┐
                                                  │  Postgres / ES /     │
                                                  │  Redis / Neo4j / ... │
                                                  └──────────┬───────────┘
                                                             │
                                                  ┌──────────▼───────────┐
                                                  │  mainorchestrator    │
                                                  │  .execute()          │ ← runs steps,
                                                  │                      │   rolls back
                                                  │                      │   on failure
                                                  └──────────┬───────────┘
                                                             │
Response (data, status) ◄────────────────────────────────────┘
```

## Middleware stack

Defined in `NimbusVault/CustomMiddlewares.py` and `NimbusVault/settings.py`. Order matters.

| Middleware | Purpose |
|---|---|
| **CORS** | Allow listed origins (configured in `VaultSettings.json`). |
| **CSP** | Content Security Policy headers. |
| **Tenant** | Reads `Org:` HTTP header → resolves to a tenant DB → swaps the DB connection used by the ORM for this request. Elasticsearch index names are also namespaced by the org slug. |
| **Encryption** | Decrypts encrypted-payload fields when present. |
| **JWT / OAuth2** | Validates the `Authorization: JWT <token>` or `Authorization: Bearer <token>` header; populates `request.user`. Keycloak / SAML2 / LDAP are alternatives — picked by `VaultSettings.json`. |
| **Elastic APM** | WSGI + ASGI tracing middlewares. Spans show up in APM with the URL pattern as transaction name. |

!!! note "Header convention"
    The auth scheme prefix in this codebase is `JWT`, not `Bearer`. Frontend code sends `authorization: JWT eyJ...`. Tenant routing requires `Org: <slug>` as an HTTP header (NOT just `?org=` in the query string).

## URL routing

`NimbusVault/urls.py` is the single root urlconf. Three groups of patterns:

1. **Built-in core URLs**: admin, Prometheus metrics, SAML2, Swagger schema/UI.
2. **VaultManagement domain URLs**: every file in `VaultManagement/urls/` is `include()`'d under the `/vault/` prefix (`ModelName.url_vault`).
3. **Plugin URLs**: every plugin listed in `ACTIVE_PLUGINS` has its `Plugins/<Name>/urls.py` included automatically:

    ```python
    for plugin in settings.ACTIVE_PLUGINS:
        urlpatterns.append(path(ModelName.url_vault, include(f"Plugins.{plugin}.urls")))
    ```

So adding endpoints requires only:

- A new path inside an existing `VaultManagement/urls/<Domain>.py`, OR
- A new domain module + an `include(...)` line in `NimbusVault/urls.py`, OR
- A new plugin under `Plugins/<Name>/` plus listing it in `VaultSettings.json` `Base.active_plugins`.

The full mechanic is spelled out in [API Creation Guide → The contract](../api-guide/contract.md).

## Multi-tenancy

Tenant isolation is **database-level**, not row-level. `TenantMiddleware` reads `Org:` and switches the active DB connection for the request's duration. Every ORM call inside the request lands in that tenant's database. No application code needs to filter by tenant — there is no shared tenant table.

Elasticsearch indices are also tenant-scoped (`{tenant}_{logical_index_name}`).

This has consequences:

- Background tasks (Celery) must explicitly pass the tenant. Look at how `Orchestrators/WorkerTasks/` handles `org` kwargs.
- Cross-tenant operations are a code smell; if you need one, justify it in the PR description.

## Errors

Everything raises typed errors from `VaultErrors/BackendErrors.py`:

```python
from VaultErrors.BackendErrors import errors
raise errors.bad_request.ValidationError("...")
raise errors.bad_request.NotFound("...")
raise errors.bad_request.PermissionDenied("...")
```

DRF exception handlers turn these into JSON responses with the right HTTP status. **Do not** raise plain `Exception` or DRF's built-in exceptions inside the BLL — always go through `errors.*`.

## Auth on views

Views inherit from `NimbusVaultConstants.CommonConstants.CommonConstants.ModelAPIView`, which applies default authentication + permission classes. Anonymous endpoints must explicitly override `authentication_classes = []` and `permission_classes = []`.
