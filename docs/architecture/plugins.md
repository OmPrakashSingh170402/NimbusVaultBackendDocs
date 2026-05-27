# Plugins (architecture view)

This page covers the slice of the plugin system that lives at the architecture layer — how plugin URLs end up in the routing table, how plugins relate to tenancy / auth / orchestrators / models, and when something is plugin-shaped vs core-shaped.

For the full development experience (folder layout, pluggy hook system, step-by-step plugin creation, and the catalogue of active plugins), see the dedicated **[Plugins](../plugins/overview.md)** section.

## How loading works

`NimbusVault/urls.py` ends with:

```python
for plugin in settings.ACTIVE_PLUGINS:
    urlpatterns.append(path(ModelName.url_vault, include(f"Plugins.{plugin}.urls")))
```

`ACTIVE_PLUGINS` is read from `VaultSettings.json` `Base.active_plugins`. Drop a plugin out of that list and its routes disappear at the next start. No code change needed.

Independently, `Plugins/plugin_manager.py` builds a [pluggy](https://pluggy.readthedocs.io/) `PluginManager` and registers hook implementations from every active plugin. Hook fan-out is invoked via `plugin_manager.hook.<hook_name>(...)`.

The two mechanisms are independent — a plugin can ship URLs without any hook impls, or hook impls without any URLs.

## How plugins fit the architecture

| Concern | Behaviour for plugins |
|---|---|
| Multi-tenancy | Inherited automatically. `TenantMiddleware` runs before any plugin view. |
| Auth | Inherited automatically. JWT/OAuth2 middleware runs before any plugin view. |
| Orchestrators | A plugin can register its own orchestrator; the registration line goes in `Orchestrators/RegisterOrchestrator.py` regardless of whether the orchestrator is owned by core or by a plugin. Plugin orchestrators can compose core orchestrators and vice-versa. |
| Models | A plugin that owns Django models ships its own migrations. **No FK from a core model to a plugin model** — the dependency arrow points plugin → core. |
| Cache | Plugin cache keys must be namespaced with the plugin's name to avoid collisions across tenants and plugins. |
| Celery | Plugin tasks live under the plugin's own folder but are registered with the same Celery app and use the same five queues. |

## When to use a plugin vs the core

| Use a plugin when… | Use the core when… |
|---|---|
| The feature only ships for some customers/deployments (HDFC, Aldermore). | The feature is for every tenant. |
| The integration is a third-party SaaS (Power BI, Teams, SharePoint, LLM provider). | It's a first-party NimbusVault domain. |
| You want the option to disable the feature via config without a deploy. | The feature is foundational. |
| The codebase is small and self-contained. | The feature has cross-cutting dependencies on many domains. |

Tie-breaker: if turning the feature off would be a **business** decision (per-tenant, per-deal), it's a plugin. If turning it off would be a **bug**, it's core.

## Where to go next

- **[Plugins → Overview](../plugins/overview.md)** — full loading model, pluggy hook system, plugin layout, configuration model.
- **[Plugins → Development guide](../plugins/development-guide.md)** — step-by-step walkthrough for building a new plugin end-to-end.
- **[Plugins → Active plugins](../plugins/active-plugins.md)** — catalogue of every plugin currently in the repo, with purpose, layout, and endpoint surface.
