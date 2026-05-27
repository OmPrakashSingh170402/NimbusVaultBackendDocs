# NimbusVault Backend — Design Docs

NimbusVault is an enterprise content management platform: multi-tenant document management, workflow automation, reporting, and integrations with SharePoint, Power BI, Azure, Keycloak, and more. Built on **Django 5.2 + Django REST Framework**, packaged as a containerised service with Celery workers behind it.

This site documents two things:

1. **How the platform is designed** — the request lifecycle, middleware, multi-tenancy, orchestrators, the data layer, plugin loading, async processing, deployment, and configuration.
2. **How to add new APIs** — a strict, opinionated contract every new endpoint must follow. See [API Creation Guide → The contract](api-guide/contract.md).

## Start here

| If you are… | Read… |
|---|---|
| New to the codebase | [Getting Started](getting-started.md) → [Architecture overview](architecture/overview.md) |
| Adding a new API | [The contract](api-guide/contract.md) → [Worked example](api-guide/walkthrough.md) |
| Writing performant code | [Good practices](api-guide/good-practices.md) |
| Reviewing someone else's PR | [Review checklist](api-guide/review-checklist.md) |
| Debugging a request | [Request lifecycle](architecture/request-lifecycle.md) |
| Building a new plugin | [Plugins overview](plugins/overview.md) → [Development guide](plugins/development-guide.md) |
| Looking up what a plugin does | [Active plugins](plugins/active-plugins.md) |
| Feeding the docs to an LLM | [LLM manifest](llms.txt) |

## The one rule you must internalise

> **No function ever touches a Django Model directly.** All ORM access goes through a domain-scoped `service.py`. That `service.py` exposes only `create`, `get`, `filter`, `update`, `delete`, `bulk_create`, `bulk_update`, `bulk_delete`. No business logic in `service.py`. No `Model.objects.…` calls outside `service.py`.

Everything else in this site is in service of that rule. See [Data layer](architecture/data-layer.md) for the why and [The contract](api-guide/contract.md) for the how.

## Repository layout snapshot

```
NimbusVaultBackend/
├── NimbusVault/             # Django project (settings, root urls, celery, middleware)
├── VaultManagement/         # Core app: views, serializers, urls
│   ├── urls/                # One module per domain — each included in NimbusVault/urls.py
│   ├── views/               # DRF views (class-based, one module per domain)
│   └── serializers/         # DRF serializers, mirrors views/ structure
├── Orchestrators/           # Workflow engine + domain orchestrators
├── bll/                     # Business-logic layer (one folder per domain)
├── VaultModels/             # Django ORM models + central ModelRegistry
├── Plugins/                 # Dynamically loaded plugins (urls auto-mounted)
├── VaultWorkflow/ VaultRules/ VaultErrors/ VaultWebSockets/ VaultJournal/ VaultActions/
├── NimbusVaultConstants/    # Centralised enums, model names, formula engine
├── VaultSettings.json       # Runtime config (NOT committed)
├── docs/                    # ← you are here
└── mkdocs.yml
```
