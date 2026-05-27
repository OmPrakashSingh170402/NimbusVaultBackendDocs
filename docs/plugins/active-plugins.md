# Active Plugins

The plugins currently shipped with the repo. The exact set running in any given environment is whatever `VaultSettings.json` `Base.active_plugins` says — this page describes what's available to enable.

!!! note
    Endpoint paths below are written without the `/vault/` prefix, which is added automatically by `NimbusVault/urls.py`. So `llm/document` resolves to `https://<host>/vault/llm/document`.

---

## LLM_Plugin

**Purpose:** LLM-driven document generation, search, chat, regulatory-guideline assessment, and text-to-* utilities (text-to-SQL, text-to-filter, text-to-workflow).

**Folder:** `Plugins/LLM_Plugin/`

**Key sub-modules:**

| Path | Responsibility |
|---|---|
| `TextToSQL/` | Natural-language → SQL over tenant data. Includes a schema-sync step and a direct-execute path. |
| `TextToFilter/` | Natural-language → DRF filter clauses for document search. |
| `TextToWorkflow/` | Natural-language → workflow definition. |
| `SuggestFields/` | Field auto-suggest for forms. |
| `RegulatoryMapping/` | Map regulatory guidelines to entity attributes; produce an assessment. |

**Endpoint surface (`Plugins/LLM_Plugin/urls.py`):**

```
POST  llm/document
POST  llm/chat
POST  llm/check
POST  llm/modeltrain
POST  llm/modelchat
GET   llm/model-sessions
POST  llm/v<vrsn_id>/doc-assess
POST  llm/sync-attributes-to-docsearch
POST  llm/text-to-filter
POST  llm/regulatory-guideline/file
POST  llm/regulatory-guideline/assessment
```

(Plus TextToSQL endpoints; consult `Plugins/LLM_Plugin/TextToSQL/Views.py` for the full list.)

**Configuration:** LLM provider credentials, model names, and feature toggles live under the `LLM` block in `VaultSettings.json`.

---

## Nimbus_Plugin

**Purpose:** Integration glue for the Nimbus product — model integration, pipeline orchestration, and permission/redirection helpers.

**Folder:** `Plugins/Nimbus_Plugin/`

**Endpoint surface (`Plugins/Nimbus_Plugin/urls.py`):**

```
POST  nimbus/model-entity/enablemodelintegration
GET   nimbus/model-entity/getnimbusprojectlist
GET   nimbus/getallmodels
GET   nimbus/getpipelinelist
POST  nimbus/runpipeline
GET   nimbus/getpipelinestatus
GET   nimbus/get-user-permission
POST  nimbus/redirection
POST  nimbus/unlink_model
```

**Configuration:** Nimbus host, auth, and pipeline routing under the `Nimbus` block in `VaultSettings.json`.

---

## SharePoint

**Purpose:** Import / sync documents from Microsoft SharePoint into NimbusVault.

**Folder:** `Plugins/SharePoint/`

**Endpoint surface (`Plugins/SharePoint/urls.py`):**

```
GET   sharepoint/files
POST  sharepoint/sync_document
POST  sharepoint/search-files
GET   sharepoint/get-processes
POST  sharepoint/pipeline
GET   sharepoint/get-process-files
POST  sharepoint/stop_search
POST  sharepoint/update-migration
```

**Notes:**
- Long-running sync work is delegated to Celery. The HTTP endpoints kick off jobs and surface progress via the `get-processes` / `get-process-files` reads.
- Auth is OAuth2 via the SharePoint app registration; credentials in `VaultSettings.json` under `SharePoint`.

---

## PowerBI

**Purpose:** Power BI report metadata + dataset/model lineage exposure; ships a downloadable Power BI connector.

**Folder:** `Plugins/PowerBI/`

**Endpoint surface (`Plugins/PowerBI/urls.py`):**

```
GET   PowerBI/Templates
GET   PowerBI/Models
GET   PowerBI/Lineages
GET   PowerBI/download-connector
```

**Configuration:** Tenant ID, workspace, and OAuth scope in `VaultSettings.json` under `PowerBI`.

---

## TeamsPlugin

**Purpose:** Propagate model/team membership into NimbusVault artifacts so Teams group changes flow into the entity permission model.

**Folder:** `Plugins/TeamsPlugin/`

**Endpoint surface (`Plugins/TeamsPlugin/urls.py`):**

```
POST  teams/propagate_model_team_to_artifact
```

(The view delegates into `Plugins/Nimbus_Plugin/views.py::GetNimbusProjectList` — keeping the actual implementation co-located with the related Nimbus logic.)

---

## HDFCUAMPlugin

**Purpose:** HDFC-specific User Access Management (UAM) — user lifecycle, segment management, group management requests, parameter change requests, workflow maker/checker, and reporting. The most complex active plugin.

**Folder:** `Plugins/HDFCUAMPlugin/`

**Internal layout (non-exhaustive):**

```
Plugins/HDFCUAMPlugin/
├── bll/
│   ├── Auth/, ModelEntity/, ModelId/, Reports/, Segments/,
│   ├── UAMParameters/, UserProfile/, Workflow/, iSACRequest/
│   └── (each sub-domain has its own HelperFunctions / GetApis / PostApis / service)
├── cron/, cronjobs/      # supercronic jobs scheduled in the celery container
├── config.py
├── constants.py
├── docs/                 # HDFC-specific deployment / runbook notes
└── (views, serializers, urls)
```

**Endpoint surface (selected — versioned via `<str:version>` path segment, e.g., `v1`):**

```
GET   <version>/uam/users
POST  <version>/uam/users
POST  <version>/uam/users/delete
POST  <version>/uam/users/status
GET   <version>/uam/users/details
POST  <version>/uam/users/session
POST  <version>/uam/users/reinstate
GET   <version>/uam/users/report
POST  <version>/uam/request
GET   <version>/uam/request/prefill
GET   <version>/uam/request/list
GET   <version>/uam/report/generate
... (segments, group-management requests, parameter change requests)
```

See `Plugins/HDFCUAMPlugin/urls.py` for the complete list.

**Notes:**
- iSAC (HDFC's internal access-control system) integration is encapsulated under `bll/iSACRequest/`.
- The plugin runs its own cron jobs via supercronic — definitions live under `Plugins/HDFCUAMPlugin/cron/`.

---

## AldermorePlugin

**Purpose:** Aldermore-specific activity-report download endpoint with a customised report shape.

**Folder:** `Plugins/AldermorePlugin/`

**Endpoint surface (`Plugins/AldermorePlugin/urls.py`):**

```
GET   activity/reports/download
```

---

## Adding to this page

When you add a new plugin, append a section here following the template above:

1. One-line purpose.
2. Folder path.
3. Key sub-modules / internal layout (if non-trivial).
4. Endpoint surface — copy from the plugin's `urls.py`.
5. Configuration block name in `VaultSettings.json`.
6. Anything non-obvious (long-running jobs, Celery queues, external auth, cron).

The [development guide](development-guide.md) covers the mechanics; this page is the catalogue.
