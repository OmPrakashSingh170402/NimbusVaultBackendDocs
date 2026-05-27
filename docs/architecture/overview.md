# Architecture Overview

NimbusVault is a layered Django service. Every request flows through the same pipeline; every domain has the same internal shape. This page is the map. The next chapters drill into each layer.

## Layers, top to bottom

```
HTTP request
  │
  ▼
┌──────────────────────────────────────────────────────────────────────┐
│ Middleware stack                                                     │
│   CORS → CSP → Tenant → Encryption → JWT/OAuth2 → Elastic APM        │
└──────────────────────────────────────────────────────────────────────┘
  │
  ▼
┌──────────────────────────────────────────────────────────────────────┐
│ URL router (NimbusVault/urls.py — everything under /vault/ prefix)   │
│   include() each VaultManagement/urls/<Domain>.py module             │
│   plugin urls auto-mounted from Plugins/<Name>/urls.py               │
└──────────────────────────────────────────────────────────────────────┘
  │
  ▼
┌──────────────────────────────────────────────────────────────────────┐
│ DRF views (VaultManagement/views/<Domain>.py)                        │
│   class-based, one method per HTTP verb (get/post/put/patch/delete)  │
│   serializer validates request → orchestrator runs business logic    │
└──────────────────────────────────────────────────────────────────────┘
  │
  ▼
┌──────────────────────────────────────────────────────────────────────┐
│ Serializers (VaultManagement/serializers/<Domain>.py)                │
│   request validation + response shaping                              │
└──────────────────────────────────────────────────────────────────────┘
  │
  ▼
┌──────────────────────────────────────────────────────────────────────┐
│ Orchestrators (Orchestrators/<Domain>Orchestrators/)                 │
│   sequential steps, automatic rollback on failure                    │
│   discovered via OrchestratorRegistry                                │
│   composes other orchestrators (cross-domain workflows)              │
└──────────────────────────────────────────────────────────────────────┘
  │
  ▼
┌──────────────────────────────────────────────────────────────────────┐
│ Business logic (bll/<Domain>/)                                       │
│   HelperFunctions.py  ← pure helpers, no ORM                         │
│   GetApisbll.py       ← read paths                                   │
│   PostApisbll.py      ← write paths                                  │
│   service.py          ← THE ONLY PLACE THAT CALLS DJANGO MODELS      │
└──────────────────────────────────────────────────────────────────────┘
  │
  ▼
┌──────────────────────────────────────────────────────────────────────┐
│ Django ORM (VaultModels/models/)                                     │
│   ModelRegistry.py — central model registry                          │
└──────────────────────────────────────────────────────────────────────┘
  │
  ▼
PostgreSQL  +  Elasticsearch  +  Redis  +  Neo4j  +  ArangoDB  +  DuckDB
```

## Why this shape

| Constraint | What enforces it |
|---|---|
| Multi-tenant isolation | `TenantMiddleware` routes by `Org:` header → per-tenant DB; Elasticsearch indices namespaced per tenant. |
| Auth everywhere | JWT/OAuth2 middleware terminates auth before any view sees the request. |
| Rollback on partial failure | Orchestrator framework: each step records a compensating action; failure unwinds in reverse order. |
| ORM in one place | `service.py` per domain is the only file allowed to import a Django model. Everything else calls `service.py`. |
| Hot-swappable integrations | Plugins in `Plugins/<Name>/` are loaded by name from `VaultSettings.json`; each ships its own `urls.py`. |

## Where each thing lives

| Concern | Location |
|---|---|
| Settings + load `VaultSettings.json` | `NimbusVault/settings.py` |
| Root URLs | `NimbusVault/urls.py` |
| Middleware | `NimbusVault/CustomMiddlewares.py` |
| Celery app + queues | `NimbusVault/celery.py` |
| Views | `VaultManagement/views/<Domain>.py` |
| URL modules | `VaultManagement/urls/<Domain>.py` |
| Serializers | `VaultManagement/serializers/<Domain>.py` |
| Orchestrators | `Orchestrators/<Domain>Orchestrators/` |
| Orchestrator framework | `Orchestrators/MainOrchestrator.py`, `Orchestrators/OrchestratorRegistry.py` |
| Business logic | `bll/<Domain>/` |
| Models | `VaultModels/models/`, registered in `VaultModels/ModelRegistry.py` |
| Plugins | `Plugins/<Name>/` (each with `urls.py`, optional `bll/`, etc.) |
| Workflow engine | `VaultWorkflow/` |
| Dynamic rules / permissions | `VaultRules/` |
| Centralised errors | `VaultErrors/BackendErrors.py` |
| WebSockets (ASGI) | `VaultWebSockets/` |
| Audit / activity log | `VaultJournal/` |
| Enums, model-name constants, formula engine | `NimbusVaultConstants/` |

Read the next pages in order if onboarding. Skip to [API Creation Guide → The contract](../api-guide/contract.md) if you already know the layout and just need to ship an endpoint.
