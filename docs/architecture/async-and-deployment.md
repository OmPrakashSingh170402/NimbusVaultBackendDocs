# Async Processing & Deployment

## Celery

- **Broker**: RabbitMQ.
- **Backend**: configured in `VaultSettings.json` (Redis typically).
- **App**: `NimbusVault.celery.app`.
- **Queues** (priority order): `high`, `medium`, `low`, `bulk`, `duckdb`.
- **Worker tasks**: live in `Orchestrators/WorkerTasks/`. Each task is a thin wrapper that hydrates context (including the tenant) and runs the matching orchestrator.

### Defining a worker task

```python
# Orchestrators/WorkerTasks/MyDomainTasks.py
from NimbusVault.celery import app
from Orchestrators.OrchestratorRegistry import OrchestratorRegistry

@app.task(queue="medium", bind=True)
def do_my_thing(self, org, user_id, payload):
    # 1. switch DB to tenant `org`
    # 2. resolve user
    # 3. fetch orchestrator + run
    orch = OrchestratorRegistry.get("my_domain")
    orch.do_thing(user_id=user_id, **payload)
    orch.mainorchestrator.execute()
```

Dispatch with:

```python
from Orchestrators.WorkerTasks.MyDomainTasks import do_my_thing
do_my_thing.apply_async(args=[org_slug, user.id, payload], queue="high")
```

## Cron jobs & scheduled work

NimbusVault runs **three** parallel scheduling mechanisms. They all live in the `celery` container (never in `django`) and each exists because the others are wrong for some workloads. Knowing which to pick is half the battle.

### The three mechanisms

| Mechanism | Defined in | Started by | What it's for |
|---|---|---|---|
| **supercronic + root `crontab`** | The top-level `crontab` file | `script.sh:131` — `supercronic /code/crontab &` | Static, shell-only periodic jobs that ship with the core. Schedule changes require a deploy. |
| **supercronic + plugin cronjobs** | `Plugins/<Name>/cronjobs` (one per plugin) | `script.sh:132` (and one line per active plugin in the same shape) — `supercronic /code/Plugins/<Name>/cronjobs &` | Plugin-owned static jobs. Same shape as the root `crontab` but namespaced under the plugin folder so the plugin can be disabled cleanly. |
| **Celery Beat with `DatabaseScheduler`** | `PeriodicTask` + `CrontabSchedule` / `IntervalSchedule` rows in the DB (via `django-celery-beat`) | `script.sh:130` — `start_beat &`, which runs `celery -A NimbusVault beat -l info --scheduler django_celery_beat.schedulers:DatabaseScheduler` | Dynamic, tenant-scoped, user-configurable schedules. Schedules can be created / edited / disabled at runtime via the ORM. |

### Mechanism 1: root `crontab` (supercronic)

```
# /code/crontab — checked into the repo at the project root
0 */6 * * * bash /code/send_alert.sh           >> /code/cronjob.log                2>&1
*/30 * * * * bash /code/TimeBasedAutomation.sh >> /code/automationcronjob.log     2>&1
0 */2 * * * bash /code/TATEscalation.sh        >> /code/tat_escalationcronjob.log 2>&1
0 8 * * * bash /code/CleanWorkerQueue.sh       >> /code/cleanup.log              2>&1
```

Each line is a standard 5-field cron expression followed by the command. **The command is a shell script at the project root** — those scripts typically invoke a `manage.py` command or a Python entrypoint that walks every tenant DB and does the work.

To add a new core scheduled job:

1. Write the shell entrypoint (e.g. `nightly_thing.sh`) at the project root. Convention: each script logs to its own `*.log` file so failures are easy to grep.
2. Append one line to `crontab` with the schedule + command + log redirect.
3. Make sure the script handles multi-tenancy itself — supercronic doesn't know about tenants. Typical pattern: iterate `auth_services_obj.list_organisations()` and switch the DB for each.

!!! warning
    `crontab` is a **file**, not the Unix `crontab` command. Editing this file changes what supercronic runs in the celery container at next start. There is no `crontab -e` involved.

### Mechanism 2: plugin-owned `cronjobs` (supercronic)

Plugins that need their own static schedule ship a `cronjobs` file inside the plugin folder. Real example:

```
# /code/Plugins/HDFCUAMPlugin/cronjobs
*/30 * * * * bash /code/Plugins/HDFCUAMPlugin/cron/uam.sh >> /code/hdfcuamplugin.log 2>&1
```

The plugin's shell script lives under `Plugins/<Name>/cron/`. **`script.sh` must launch a `supercronic` process for each active plugin's `cronjobs` file** — currently that's done with explicit lines:

```bash
# script.sh
supercronic /code/crontab &
supercronic /code/Plugins/HDFCUAMPlugin/cronjobs &
```

When you add a new plugin with periodic jobs:

1. Create `Plugins/<Name>/cronjobs` with the schedule entries.
2. Add an explicit `supercronic /code/Plugins/<Name>/cronjobs &` line in `script.sh` under the `SERVICE_TYPE = celery` branch.
3. Disable cleanly: removing the plugin from `Base.active_plugins` does **not** stop supercronic from looking at the file. To fully disable, also remove (or guard) the `supercronic` line in `script.sh`.

This is the part of the system that would most benefit from being driven off `ACTIVE_PLUGINS` automatically — until that refactor lands, the explicit `script.sh` lines are the source of truth.

### Mechanism 3: Celery Beat with `DatabaseScheduler`

For schedules that are **user-configurable at runtime** or **per-tenant**, the supercronic mechanisms are wrong — schedule changes would require a deploy. Use Celery Beat with `django-celery-beat`'s `DatabaseScheduler` instead, which reads the schedule list from the `PeriodicTask` table at startup and re-reads it periodically.

Worked reference: `VaultCustomReports/bll.py::sync_periodic_task` creates a `PeriodicTask` row when a user creates a Report Schedule:

```python
# VaultCustomReports/bll.py
def get_or_create_crontab(cron_expression: str):
    minute, hour, day_of_month, month_of_year, day_of_week = cron_expression.split()
    crontab, _ = CrontabSchedule.objects.get_or_create(
        minute=minute, hour=hour, day_of_month=day_of_month,
        month_of_year=month_of_year, day_of_week=day_of_week,
    )
    return crontab


def sync_periodic_task(schedule):
    """Create or update a Celery Beat task for a ReportSchedule."""
    current_db_config = get_db_config()
    current_org_name = get_current_org_name()
    target_queue = settings.CELERY_SETTINGS["queues"]["high"]["name"]

    task_name = f"report-schedule-{schedule.id}-{current_org_name}"
    set_db_for_router("default")
    crontab = get_or_create_crontab(schedule.cron_expression)

    PeriodicTask.objects.create(
        name=task_name,
        task="VaultCustomReports.bll.generate_and_email_report",
        crontab=crontab,
        kwargs=json.dumps({
            "schedule_id":  schedule.id,
            "db_settings":  current_db_config,    # ← carries tenant into the worker
        }),
        queue=target_queue,
    )
```

Three things to copy from this example for every new dynamic schedule:

1. **Task name encodes the tenant** (`<purpose>-<id>-<org>`) — so multiple tenants scheduling the same logical job don't collide and you can `.filter(name__startswith="report-schedule-")` for housekeeping.
2. **`kwargs` carries the tenant `db_settings`** — the Beat scheduler lives in the default DB; the actual task runs in a worker that has to switch back to the tenant DB before doing anything. Embed enough info in the kwargs for the task to do that switch.
3. **Schedules are persisted under the `default` DB** (`set_db_for_router("default")`) — `PeriodicTask` and `CrontabSchedule` are global, not per-tenant.

To add a new dynamic schedule:

1. Define the worker task in `Orchestrators/WorkerTasks/` (or the relevant plugin) decorated with `@app.task`.
2. Create a `PeriodicTask` row pointing at that task's dotted name, with a `CrontabSchedule` or `IntervalSchedule`.
3. Save the row's `id` against your domain model (in `VaultCustomReports` it's `ReportSchedule.periodic_task`) so subsequent edits can update the same row instead of creating duplicates.
4. **Always `enabled = schedule.is_active`** when toggling — don't delete + recreate.

### When to use which

| Use case | Mechanism |
|---|---|
| Schedule is fixed, the same in every environment, ships with the core | Root `crontab` |
| Schedule is fixed but lives with an optional plugin | Plugin `cronjobs` + a `supercronic` line in `script.sh` |
| Schedule is configured by an end user at runtime (per tenant, per resource) | Celery Beat / `PeriodicTask` |
| Schedule needs to be enabled/disabled without a deploy | Celery Beat / `PeriodicTask` |
| Job is "iterate all tenants and do X" | Either supercronic mechanism — the script itself walks tenants |
| Job is "do X for tenant Y on schedule S" | Celery Beat; tenant baked into the task `kwargs` |

### Multi-tenancy reminders

- **No mechanism is tenant-aware out of the box.** TenantMiddleware only fires on HTTP requests.
- For **supercronic** jobs the shell script must iterate tenants and switch DB per tenant (typically via the same `set_db_for_router(...)` calls used elsewhere).
- For **Celery Beat** tasks, the tenant comes in via the task `kwargs` (see `db_settings` above). The worker task must call `set_db_for_router(...)` before touching the ORM.
- **Cache keys, ES indices, and external-system identifiers** all need the tenant in scope — same rules as a normal request.

### Logging & observability

Each supercronic job redirects to its own log file in `/code/`. The convention is:

```
<script>.sh  →  /code/<script>cronjob.log  (or a job-specific name)
```

Failures show up in `tail -f /code/cronjob.log` inside the celery container — they do **not** surface in APM, since supercronic doesn't run any Python tracing.

Celery Beat tasks **do** show up in APM (as standard Celery transactions) and are subject to the same retry/failure handling as any other worker task — see [Failure handling](#failure-handling) below.

## Deployment

`script.sh` is the Docker entrypoint. It branches on `SERVICE_TYPE`:

| `SERVICE_TYPE` | What runs |
|---|---|
| `django` | Migrations → memcached → Gunicorn with `uvicorn.workers.UvicornWorker` (ASGI, 5 workers). Config in `gunicorn.py`. |
| `celery` | Celery Beat + workers per queue (concurrency tuned per queue) + supercronic for cron jobs. |

Two containers of the same image, different env. Image base: `python:3.10`.

### Runtime config

`VaultSettings.json` (or its encrypted twin `VaultSettings.enc.json`) is mounted at the project root in the container and parsed at start in `NimbusVault/settings.py`. Changes require a container restart.

## Monitoring

| Signal | Where |
|---|---|
| Metrics (Prometheus) | `/metrics` (django-prometheus). Restrict via `VaultSettings.json` `RestrictMetricPath: true`. |
| Traces (APM) | `elasticapm` middleware (WSGI + ASGI). Transactions named by URL pattern. |
| Logs | Stdout (container logs). Use structured logging via `extra={...}`. |
| Dashboards | Grafana on top of Prometheus. See `README.md` for setup. |

## Failure handling

- **Synchronous request failures** bubble up through `VaultErrors/BackendErrors.py` → DRF exception handler → JSON response with the right status.
- **Orchestrator step failures** trigger the compensating chain — every step before the failing one runs its `rollback`. The whole orchestrator returns failure to the caller.
- **Celery task failures** are retried per `@app.task(..., autoretry_for=(...), max_retries=...)` decorations; failed tasks land in the `FailedTasks` model (see `VaultManagement/views/FailedTasks.py`) for visibility.
