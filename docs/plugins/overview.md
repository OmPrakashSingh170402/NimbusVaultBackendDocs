# Plugins — Overview

Plugins are domains that ship with their own URL surface and BLL but obey the same architectural rules as the core. They can be turned on or off via `VaultSettings.json` without a code change. The system uses **[pluggy](https://pluggy.readthedocs.io/)** for hookable extension points and Django for the URL/view layer.

## How loading works

There are two independent loading mechanisms in play, both gated by `Base.active_plugins` in `VaultSettings.json`.

### 1. URL mounting (every plugin)

The last block of `NimbusVault/urls.py` iterates over `ACTIVE_PLUGINS` and includes each plugin's `urls.py` under the `/vault/` prefix:

```python
for plugin in settings.ACTIVE_PLUGINS:
    urlpatterns.append(path(ModelName.url_vault, include(f"Plugins.{plugin}.urls")))
```

So a plugin appears in the API as soon as it:

1. has a `Plugins/<Name>/urls.py`,
2. is listed in `VaultSettings.json` `Base.active_plugins`,
3. and the server is restarted.

No edits to the core URL config are required.

### 2. Pluggy hook registration (plugins that hook into core events)

`Plugins/plugin_manager.py` builds a [pluggy](https://pluggy.readthedocs.io/) `PluginManager`:

```python
def get_plugin_manager():
    pm = pluggy.PluginManager("vault")

    # Register every HookSpec class defined in Plugins/Hooks/<HookName>.py
    for hookspec_file in Path(settings.PLUGINS_DIR, "Hooks").iterdir():
        if hookspec_file.suffix == ".py" and hookspec_file.name != "__init__.py":
            module = __import__(f"Plugins.Hooks.{hookspec_file.stem}", fromlist=[hookspec_file.stem])
            pm.add_hookspecs(getattr(module, hookspec_file.stem))

    # Register every active plugin's <Name>.py hook implementation
    for plugin in settings.ACTIVE_PLUGINS:
        module = __import__(f"Plugins.{plugin}.{plugin}", fromlist=[plugin])
        pm.register(getattr(module, plugin))

    return pm

plugin_manager = get_plugin_manager()
```

Two contracts here:

- **Hook specifications** live in `Plugins/Hooks/<HookName>.py` and are classes that declare available hooks (with `@hookspec`).
- **Hook implementations** live in `Plugins/<PluginName>/<PluginName>.py` — a file named exactly like the plugin folder — and decorate methods with `@hookimpl`.

Anywhere in the core, you can invoke hooks via:

```python
from Plugins.plugin_manager import plugin_manager
plugin_manager.hook.<hook_name>(<kwargs>)
```

Pluggy fan-outs the call to every registered implementation. The pluggy machinery is optional — a plugin can ship without any hook implementations and still expose URLs.

## Plugin layout

```
Plugins/
├── Hooks/                         # HookSpec classes (one file per hookspec)
│   └── <HookName>.py
└── <PluginName>/
    ├── __init__.py
    ├── urls.py                    # required if the plugin exposes HTTP endpoints
    ├── <PluginName>.py            # required for pluggy registration (hook impls)
    ├── apps.py                    # if the plugin is also a Django app
    ├── views.py / views/          # DRF views
    ├── serializers.py / serializers/
    ├── HelperFunctions.py         # pure helpers
    ├── CommonConstants.py         # local enums/constants
    ├── bll/<SubDomain>/{HelperFunctions, GetApisbll, PostApisbll, service}.py
    ├── models.py                  # plugin-owned Django models (if any)
    └── migrations/                # if the plugin owns models
```

The internal structure mirrors the core: views call orchestrators (or BLL directly for reads), BLL calls `service.py`, only `service.py` touches the ORM. **The [API Creation Contract](../api-guide/contract.md) applies inside plugins exactly as it does in the core** — no exceptions.

## Configuration

| Where | What |
|---|---|
| `VaultSettings.json` → `Base.active_plugins` | The list of plugins to load. Order does not matter. |
| `VaultSettings.json` → plugin-specific blocks (e.g. `SharePoint`, `PowerBI`, `LLM`) | Per-plugin credentials, endpoints, feature flags. |
| `Plugins/<Name>/config.py` (when present) | Plugin-local constants that are not deployment-scoped. |

A plugin should **never** read its own credentials from environment variables directly — they go in `VaultSettings.json`, which is the single source of truth for runtime config.

## When to use a plugin vs the core

| Use a plugin when… | Use the core when… |
|---|---|
| The feature only ships for some customers/deployments (HDFC, Aldermore). | The feature is for every tenant. |
| The integration is a third-party SaaS (Power BI, Teams, SharePoint, LLM provider). | It's a first-party NimbusVault domain. |
| You want the option to disable the feature via config without a deploy. | The feature is foundational. |
| The code base is small and self-contained. | The feature has cross-cutting dependencies on many domains. |

A useful tie-breaker: if turning the feature off would be a **business** decision (per-tenant, per-deal), it's a plugin. If turning it off would be a **bug**, it's core.

## How plugins relate to the rest of the architecture

- **Multi-tenancy:** Plugins inherit tenant routing automatically — `TenantMiddleware` runs before any plugin view sees the request. A plugin should never read a `request.org` or query string for tenant context.
- **Auth:** Same — JWT/OAuth2 middleware terminates auth before the plugin view runs.
- **Orchestrators:** Plugins can register their own orchestrators (the line goes in `Orchestrators/RegisterOrchestrator.py::register_orchestrator()` regardless of whether the orchestrator is owned by core or by a plugin). Plugin orchestrators can compose core orchestrators and vice-versa.
- **Models:** A plugin that owns Django models ships its own migrations. Don't add foreign keys from core models to plugin models — the dependency arrow points from plugin → core, never the other way.
- **Cache / cache invalidation:** Plugin cache keys must be namespaced with the plugin's name (e.g. `sharepoint:list:<tenant>:<...>`) to avoid collisions.

For step-by-step instructions on building a new plugin, see [Plugins → Development guide](development-guide.md). For per-plugin descriptions and endpoint surfaces, see [Active plugins](active-plugins.md).
