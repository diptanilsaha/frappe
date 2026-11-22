---
title: App State
description: Turn an installed app off on one site without uninstalling it, with bench disable-app and enable-app, and react with the before_disable and after_enable hooks.
version: develop
sources:
  - frappe/app_state.py
  - frappe/apps.py
  - frappe/installer.py
  - frappe/commands/site.py
  - frappe/core/doctype/installed_applications/installed_applications.py
  - frappe/core/doctype/scheduled_job_type/scheduled_job_type.py
  - frappe/__init__.py
  - frappe/permissions.py
  - frappe/hooks.py
  - frappe/migrate.py
  - frappe/utils/boilerplate.py
  - frappe/custom/__init__.py
  - frappe/model/meta.py
---

# App State

A site can disable an installed app. The app stays installed: its tables, data and migrations stay. But the app stops taking effect on that site. Its hooks do not load, its DocTypes are hidden and blocked, its whitelisted methods refuse calls, and its scheduled jobs do not run. Enable the app again to bring everything back. Use this to try an app, to turn off a module for one customer on a shared bench, or to switch an app off before you uninstall it.

## Disable and enable

```bash
bench --site frappe.localhost disable-app sample_app
bench --site frappe.localhost enable-app sample_app
```

A System Manager can do the same in the desk: open **Installed Applications** and use the disable or enable action on the app row. It calls the whitelisted method `frappe.core.doctype.installed_applications.installed_applications.set_app_state(app_name, disabled)`.

From code:

```python
from frappe.installer import disable_app, enable_app

disable_app("sample_app")
enable_app("sample_app")
```

Both functions commit, then clear the cache (`frappe.clear_cache()` and `frappe.client_cache.erase_persistent_caches()`).

Rules:

- `frappe` cannot be disabled.
- The app must be installed on the site.
- You cannot disable an app while an active app lists it in `required_apps`. Disable the dependent app first.
- You cannot enable an app while one of its `required_apps` is disabled. Enable the dependency first.
- `bench install-app` on a disabled app enables it.

## Where the state is stored

The disabled apps are a JSON list in the global default `disabled_apps` (`frappe.db.get_global("disabled_apps")`). `installed_apps` is not changed.

```python
frappe.get_installed_apps()  # all installed apps, disabled ones too
frappe.get_disabled_apps()   # installed apps that are disabled on this site
frappe.get_active_apps()     # installed apps that are not disabled
```

All three are cached per request.

## What a disabled app loses

| Area | Effect |
|---|---|
| Hooks | `frappe.get_hooks()` reads only active apps. The disabled app's `doc_events`, `override_*`, `app_include_js` and all other hooks stop. |
| Whitelisted methods | A call to a method in the app raises `AppDisabledError` ("App sample_app is disabled on this site"). |
| DocTypes | `frappe.has_permission` returns `False` for DocTypes in the app's modules. They are removed from boot data, search and the sidebar. |
| Records with a `module` field | DocType, Page, Report, Dashboard, Dashboard Chart, Number Card, Workspace, Workspace Sidebar, Print Format, Notification, Web Form, Client Script and Server Script rows of the app's modules are hidden from list views. |
| Customizations | Custom Fields and Property Setters whose `module` belongs to the app are ignored. Link and Table fields that point to a disabled DocType are hidden. |
| Desktop Icon | Rows whose `app` is the disabled app are hidden. |
| Notifications | Notifications of the app's modules do not fire. |
| Scheduler | Its `Scheduled Job Type` rows stay, with their `stopped` and `last_execution` values, but are skipped and hidden. The form refuses **Skip Next Execution**. |
| Website | Its `www` pages, templates and routes are not served. |
| Realtime | Its realtime handlers do not run for the site. |
| Default app | It cannot be set as a user's default app. |

Modules that the site made (`Module Def` with `custom = 1`) are never hidden, even if their `app_name` is the disabled app.

Filtering is paused while one of these flags is set: `in_install`, `in_migrate`, `in_patch`, `in_import`, `in_uninstall`, `in_setup_wizard`, `in_app_toggle`. So migrations and patches still see the disabled app's DocTypes.

## Hooks for your app

Add these hooks to your app to hide or restore what it adds to other apps' DocTypes: Custom Fields, Property Setters and Custom DocPerms.

```python
# sample_app/hooks.py
before_disable = "sample_app.uninstall.before_disable"
after_disable = "sample_app.uninstall.after_disable"
before_enable = "sample_app.install.before_enable"
after_enable = "sample_app.install.after_enable"
```

```python
# sample_app/uninstall.py
from frappe.custom import hide_customizations

SAMPLE_CUSTOMIZATIONS = {
	"Custom Field": [{"dt": "ToDo", "fieldname": "sample_order"}],
	"Property Setter": [{"doc_type": "ToDo", "field_name": "description", "property": "reqd"}],
}


def before_disable():
	hide_customizations(SAMPLE_CUSTOMIZATIONS)
```

```python
# sample_app/install.py
from frappe.custom import unhide_customizations

from sample_app.uninstall import SAMPLE_CUSTOMIZATIONS


def after_enable():
	unhide_customizations(SAMPLE_CUSTOMIZATIONS)
```

`hide_customizations(customizations)` sets `is_app_disabled` = 1 on the matching rows. Frappe then ignores them in the meta. The rows and their data stay. `unhide_customizations` sets it back to 0. The keys must be `Custom Field` (filter on `dt`), `Property Setter` (filter on `doc_type`) or `Custom DocPerm` (filter on `parent`). Each filter must name one DocType.

| Hook | When |
|---|---|
| `before_disable` | Before the app is added to `disabled_apps`. Also runs again on each `bench migrate` for every disabled app. |
| `after_disable` | After the app is added to `disabled_apps`, before commit. |
| `before_enable` | Before the app is removed from `disabled_apps`. |
| `after_enable` | After the app is removed from `disabled_apps`, before commit. |

The functions take no arguments. Only the hooks of the app being toggled run. If a hook raises, the change rolls back.

Make `before_disable` idempotent. `bench migrate` calls it again ("Applying state of disabled apps again...") because a migration can make the customizations again.

## Helpers for app code

`frappe/app_state.py` has helpers for code that lists DocTypes or modules:

| Function | Returns |
|---|---|
| `get_disabled_modules()` | Set of `Module Def` names of disabled apps. Empty while a maintenance flag is set. |
| `get_disabled_doctypes()` | Set of DocTypes in those modules. |
| `is_module_disabled(module)` | `True` if the module belongs to a disabled app. |
| `filter_out_disabled_doctypes(doctypes)` | The list without DocTypes of disabled apps. |
| `get_module_permission_query_conditions` | `permission_query_conditions` hook for DocTypes with a `module` field. |
| `get_app_permission_query_conditions` | `permission_query_conditions` hook for DocTypes with an `app` field. |

Use the last two in your `hooks.py` for your own DocTypes that belong to modules or apps:

```python
# sample_app/hooks.py
permission_query_conditions = {
	"Sample Report Config": "frappe.app_state.get_module_permission_query_conditions",
}
```

## Pitfalls

- Disabling does not remove data. Customizations without a `module` of the app stay in effect until you hide them in `before_disable`.
- Jobs that were enqueued with `frappe.enqueue` before the toggle still run. Scheduled jobs check the state when they run and do nothing.
- Data in other apps that links to records of the disabled app stays. Only the Link fields are hidden.

## Related

- [Apps](apps.md)
- [Hooks](hooks.md)
- [Scheduler](../runtime/scheduler.md)
- [Hooks reference](../../reference/configuration/hooks.md)
- [CLI reference](../../reference/configuration/cli.md)
