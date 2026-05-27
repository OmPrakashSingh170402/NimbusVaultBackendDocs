# Key Files

A quick map of the files you'll touch most.

## Django bootstrap

| File | What's in it |
|---|---|
| `NimbusVault/settings.py` | Django settings, parses `VaultSettings.json`, defines all module-level config constants. |
| `NimbusVault/urls.py` | Root URLconf. Includes all `VaultManagement/urls/<Domain>.py` modules and dynamically mounts plugins. |
| `NimbusVault/CustomMiddlewares.py` | Middleware stack: CORS, CSP, Tenant, Encryption, JWT/OAuth2, Elastic APM. |
| `NimbusVault/celery.py` | Celery app, queue setup, broker config. |
| `NimbusVault/asgi.py` / `NimbusVault/wsgi.py` | ASGI / WSGI entry points. |
| `gunicorn.py` | Gunicorn config (uvicorn workers, ASGI). |
| `script.sh` | Container entrypoint. Branches on `SERVICE_TYPE`. |

## Per-domain layout

For any new domain `MyDomain`:

| File | Purpose |
|---|---|
| `VaultManagement/urls/MyDomain.py` | Route definitions. |
| `VaultManagement/views/MyDomain.py` | DRF class-based views. |
| `VaultManagement/serializers/MyDomain.py` | Request/response serializers. |
| `Orchestrators/MyDomainOrchestrators/MyDomainOrchestrator.py` | Workflow + rollback. |
| `bll/MyDomain/HelperFunctions.py` | Pure helpers. |
| `bll/MyDomain/GetApisbll.py` | Read-path business logic. |
| `bll/MyDomain/PostApisbll.py` | Write-path business logic. |
| `bll/MyDomain/service.py` | The **only** ORM access point. |
| `VaultModels/models/MyDomain.py` | Django models. |
| `Swagger/MyDomain/MyDomainConstants.py` | (optional) Swagger schema mappings. |

## Cross-cutting frameworks

| File | What it does |
|---|---|
| `Orchestrators/MainOrchestrator.py` | Step + rollback engine. |
| `Orchestrators/OrchestratorRegistry.py` | Discovery (`OrchestratorRegistry.get("<key>")`); builds a fresh `MainOrchestrator` + domain orchestrator instance per run. |
| `Orchestrators/RegisterOrchestrator.py` | The single `register_orchestrator()` function that lists every domain orchestrator. Add **one line** here when you create a new domain — there is no decorator. |
| `Orchestrators/AbstractOrchestrators/BaseOrchestrator.py` | The `BaseOrchestrator` **metaclass** (not a base class). Enforces Description/Input/Output docstrings on every orchestrator class and method, and gates instantiation through the registry. |
| `Orchestrators/ContextManager.py` | `@context_manager(get_contexts=..., set_contexts=..., sync_with_parent_context=True)` — declares and enforces which context keys a method/step reads and writes. Supports dict, callable, and conditional/`"else"` buckets. |
| `Orchestrators/OrchestrationExceptions.py` | `StepExitException` — raised when a step writes `result_status` to exit the workflow early. |
| `Orchestrators/WorkerTasks/` | Celery wrappers around orchestrator runs. |
| `VaultModels/ModelRegistry.py` | Central name → model class registry. |
| `Plugins/plugin_manager.py` | Plugin discovery. |
| `VaultErrors/BackendErrors.py` | All typed errors. |
| `NimbusVaultConstants/CommonConstants/CommonConstants.py` | `ModelAPIView`, `flatten_serializer_errors`, and other shared view helpers. |
| `NimbusVaultConstants/Modelentity/Modelentity.py` | `ModelName.url_vault` etc. |

## Config files (not code)

| File | Status |
|---|---|
| `VaultSettings.json` | Plaintext config. **Not committed.** |
| `VaultSettings.enc.json` | Encrypted config. May be committed. |
| `requirement.txt` | Python dependencies. |
| `Makefile` | `make local`, `make server`, `make vault`, `make killme`, `make fake`, `make up-vault`, `make restart`, `make stop`. |
| `crontab` | Supercronic scheduled jobs (celery container only). |
| `docker-compose.yml` | Local Docker stack. |
| `Dockerfile` | Image build. |

## Tests

| File | Purpose |
|---|---|
| `conftest.py` | Bootstraps Django before pytest collection. |
| `VaultManagement/tests/test_*.py` | ~30 API-level test modules. |
| `bll/Tests/Auth/` | Auth BLL unit tests. |
