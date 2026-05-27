# Configuration

All runtime configuration is loaded from a single JSON file: **`VaultSettings.json`** (or its encrypted twin `VaultSettings.enc.json`) at the project root. Parsed once at start in `NimbusVault/settings.py` and exposed as module-level variables.

## File location

| Environment | Source |
|---|---|
| Local dev | `VaultSettings.json` at the project root (gitignored). Copy from a teammate or generate from `VaultSettings.enc.json`. |
| Docker / production | Mounted via volume from the secrets pipeline. Container starts decrypt the `.enc.json` if a key is present. |

## Top-level sections

| Section | Purpose |
|---|---|
| `Base` | Project-wide toggles: `active_plugins`, debug flags, allowed hosts, secret key. |
| `Database` | PostgreSQL primary + read replicas (per-tenant, per-deployment). |
| `Elasticsearch` | Index prefix, hosts, auth. |
| `Redis` | Cache + Celery result backend. |
| `RabbitMQ` | Celery broker URL. |
| `Neo4j` / `ArangoDB` / `DuckDB` | Graph + analytical store config. |
| `Auth` | JWT keys, OAuth2, SAML2, Keycloak, LDAP toggles. |
| `AWS` / `Azure` / `GCP` | Cloud credentials. |
| `SharePoint` / `PowerBI` / `Gitea` | Integration credentials. |
| `Celery` | Queue tuning. |
| `FeatureFlags` | Per-feature toggles. |

## How it's exposed in Python

```python
# NimbusVault/settings.py loads VaultSettings.json into module-level vars
from NimbusVault.settings import (
    vault_settings,        # the whole dict
    VAULT_SETTINGS,        # alias of vault_settings
    SERVER_SETTINGS,       # vault_settings["Server"]
    CELERY_SETTINGS,       # vault_settings["Celery"]
    KEYCLOAK_CONFIG,       # vault_settings["Auth"]["Keycloak"] (or None)
    ACTIVE_PLUGINS,        # vault_settings["Base"]["active_plugins"]
    # ...
)
```

Import these. Don't read `VaultSettings.json` again at runtime.

## Adding a new setting

1. Add the key to `VaultSettings.json` (and the encrypted twin for prod).
2. In `NimbusVault/settings.py`, parse it into a module-level constant.
3. Import that constant where needed.
4. Document the key in this page.
5. **Never** hardcode secrets or environment-specific values in code.

## Encrypted settings

The production deployment ships `VaultSettings.enc.json`. The decryption key + IV come from container env. The decrypted content lands at `VaultSettings.json` before Django starts.

For local development, ask for the plain `VaultSettings.json` — do not commit it.

## Feature flags

The `FeatureFlags` block in `VaultSettings.json` is the canonical place for booleans that gate behaviour at runtime without a deploy. Adding a flag:

```json
{
  "FeatureFlags": {
    "EnableMyNewThing": false
  }
}
```

```python
from NimbusVault.settings import vault_settings
if vault_settings.get("FeatureFlags", {}).get("EnableMyNewThing"):
    ...
```

Plan a removal date when you add a flag. Stale flags rot.
