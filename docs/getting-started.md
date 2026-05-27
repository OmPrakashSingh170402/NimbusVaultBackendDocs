# Getting Started

## Prerequisites

| Tool | Version |
|---|---|
| Python | 3.10 |
| PostgreSQL | 14+ |
| Redis | 6+ |
| RabbitMQ | 3.10+ |
| Elasticsearch | 7.x or 8.x |
| Docker (optional) | 20+ |

External services touched by the backend at runtime (configured via `VaultSettings.json`): PostgreSQL, Redis, RabbitMQ, Elasticsearch, Neo4j, ArangoDB, DuckDB, optionally Keycloak / Azure AD / LDAP / SAML2.

## Configuration

All runtime configuration lives in **`VaultSettings.json`** at the project root. It is **not** committed. Get it from the deployment/secrets pipeline or copy a teammate's local one. See [Configuration](architecture/configuration.md) for the full schema.

The encrypted production version is `VaultSettings.enc.json`; it gets decrypted at container start.

## Local development

```bash
# one-time: install uv (https://docs.astral.sh/uv/)
# Windows (PowerShell)
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
# macOS / Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# create a venv pinned to Python 3.10
uv venv --python 3.10
# activate it — Windows
.venv\Scripts\Activate.ps1
# activate it — macOS / Linux
source .venv/bin/activate

# install deps (uv is a faster, drop-in replacement for pip)
uv pip install -r requirement.txt

# run migrations
python manage.py migrate

# run dev server (port 80)
make local

# alternative — uWSGI
make server

# stop everything
make killme
```

Django admin is at `/vault/admin/`, Swagger UI at `/vault/api/swagger/`, Prometheus metrics at `/metrics` (gated by `RestrictMetricPath`).

## Docker

```bash
make up-vault    # build image
make vault       # docker-compose up + shell into container
make restart
make stop
```

Image is based on `python:3.10`. Entrypoint is `script.sh`. `SERVICE_TYPE` env var picks between `django` (Gunicorn + ASGI + uvicorn workers, memcached, migrations) and `celery` (Beat + priority workers + cron jobs).

## Celery

```bash
# worker for one queue
celery -A NimbusVault worker -l info --queues=high --concurrency=4

# beat
celery -A NimbusVault beat -l info --scheduler django_celery_beat.schedulers:DatabaseScheduler
```

Five priority queues: `high`, `medium`, `low`, `bulk`, `duckdb`. Worker tasks live in `Orchestrators/WorkerTasks/`. Scheduled jobs are in the `crontab` file (run via supercronic in the celery container).

## Testing

```bash
pytest                                              # everything
pytest VaultManagement/tests/test_model_entity_api.py
pytest VaultManagement/tests/test_model_entity_api.py::test_function_name -v
```

Tests live in `VaultManagement/tests/` (~30 files) and `bll/Tests/Auth/`. `conftest.py` at the repo root bootstraps Django before collection.

## Building these docs locally

```bash
uv pip install -r docs/requirements.txt
mkdocs serve     # http://127.0.0.1:8000/NimbusVaultBackend/
mkdocs build     # static site → site/
```

> Conflict with the Django dev server on port 8000? Run `mkdocs serve -a 127.0.0.1:8001`.

GitHub Pages publishes from the `gh-pages` branch via `.github/workflows/docs.yml` on every push to `main`.
