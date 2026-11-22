---
title: Hooks
description: How hooks.py works, how hook values from all apps merge, and the hooks that apps use most.
version: v15
sources:
  - frappe/__init__.py
  - frappe/hooks.py
  - frappe/utils/boilerplate.py
  - frappe/installer.py
  - frappe/migrate.py
  - frappe/model/document.py
  - frappe/desk/form/meta.py
  - frappe/boot.py
  - frappe/sessions.py
  - frappe/app.py
  - frappe/utils/jinja.py
  - frappe/utils/fixtures.py
---

# Hooks

`hooks.py` is the file where an app tells Frappe what to add or change. Each hook is a module-level variable. Frappe reads the hooks of all installed apps and merges them. Use hooks to run code on document events, add assets, add scheduled jobs, override classes and methods, and react to install and migrate. This page explains how hooks work and shows the hooks that apps use most. For every hook key, see the [Hooks reference](../../reference/configuration/hooks.md).

## How Frappe reads hooks

`frappe.get_hooks` loads `<app>.hooks` for each installed app (`frappe/__init__.py`, `_load_app_hooks`):

- Apps are read in the order of `installed_apps`. `frappe` is first.
- Apps in `installed_apps` that are not in the bench are skipped.
- Every module-level name that does not start with `_` is a hook. Imported modules, functions and classes are skipped.

```python
# sample_app/hooks.py
app_name = "sample_app"
app_title = "Sample App"

doc_events = {
	"Sample Order": {
		"on_submit": "sample_app.events.notify_customer",
	},
}
```

## How values merge

`frappe.append_hook` merges the value of each app into one dict:

| Value in `hooks.py` | Merged result |
|---|---|
| A string or number | A list. Each app appends its value. |
| A list | One list. Each app extends it. |
| A dict | A dict. Keys merge across apps, and each leaf becomes a list. |

So `frappe.get_hooks("app_name")` returns a list, even though each app sets a string:

```python
frappe.get_hooks("app_name")
# ['frappe', 'sample_app']

frappe.get_hooks("doc_events")["*"]["on_update"]
# ['frappe.desk.notifications.clear_doctype_notifications', ...]   one list for all apps
```

Most hooks call every value in the list, in app order. Some hooks use only the last value:

| Hook | Which value wins |
|---|---|
| `override_whitelisted_methods` | The last app in `installed_apps` that overrides the method. |
| `override_doctype_class` | The last app that overrides the DocType. |

## Read hooks in your code

```python
# all apps
for method in frappe.get_hooks("after_migrate"):
	frappe.get_attr(method)()

# one app only
frappe.get_hooks("fixtures", app_name="sample_app")

# a default when the hook is not set
frappe.get_hooks("my_hook", default={})
```

| Parameter | Type | Default | Description |
|---|---|---|---|
| `hook` | `str` | `None` | Hook name. If `None`, return all hooks as a dict. |
| `default` | any | `[]` | Value to return when no app sets the hook. |
| `app_name` | `str` | `None` | Read the hooks of this app only. |

Hooks are cached:

- Developer mode off: in the Redis cache (`frappe.cache`), key `app_hooks`.
- Developer mode on: not cached between requests. Frappe reloads them per request.
- In both modes, the result is cached for the length of one request (`request_cache`).

`bench --site <site> clear-cache` clears the Redis cache. Run it after you change `hooks.py` on a site that is not in developer mode. Also restart the bench processes, because workers keep loaded Python modules.

## Most used hooks

### App metadata

`bench make-app` writes these values. They are also read as hooks.

```python
# sample_app/hooks.py
app_name = "sample_app"
app_title = "Sample App"
app_publisher = "Example Corp"
app_description = "Sample app"
app_email = "admin@example.com"
app_license = "mit"
```

All other hooks in the generated `hooks.py` are comments. One of them is `export_python_type_annotations = True`. Uncomment it to make Frappe write field type hints into the controllers of the app when you save a DocType in developer mode.

### Install, uninstall and migrate

```python
# sample_app/hooks.py
required_apps = ["frappe/erpnext"]

before_install = "sample_app.install.before_install"
after_install = "sample_app.install.after_install"
after_sync = "sample_app.install.after_sync"

before_uninstall = "sample_app.uninstall.before_uninstall"
after_uninstall = "sample_app.uninstall.after_uninstall"

# called with the name of the app that is installed or uninstalled
after_app_install = "sample_app.utils.after_app_install"
before_app_uninstall = "sample_app.utils.before_app_uninstall"

before_migrate = "sample_app.migrate.before_migrate"
after_migrate = "sample_app.migrate.after_migrate"
```

| Hook | When it runs |
|---|---|
| `required_apps` | `install-app` installs these apps first. |
| `before_install` | Before the DocTypes of this app are synced. Return `False` to stop the install. |
| `after_install` | After the app is added to `installed_apps` and its patches are marked as done. |
| `after_sync` | At the end of install, after fixtures and customizations are synced. |
| `before_app_install`, `after_app_install` | When any app is installed. The function gets the app name. |
| `before_uninstall`, `after_uninstall` | When this app is uninstalled. |
| `before_app_uninstall`, `after_app_uninstall` | When any app is uninstalled. The function gets the app name. |
| `before_migrate`, `after_migrate` | At the start and end of `bench migrate`. |

See [Apps](apps.md) for the full install order.

### Document events

`doc_events` runs functions on controller events of any DocType. The key is a DocType name, a tuple of names, or `"*"` for all DocTypes.

```python
# sample_app/hooks.py
doc_events = {
	"Sample Order": {
		"validate": "sample_app.events.check_credit_limit",
		"on_submit": [
			"sample_app.events.notify_customer",
			"sample_app.events.update_stock",
		],
	},
	("Sample Item", "Sample Order"): {
		"on_trash": "sample_app.events.log_delete",
	},
	"*": {
		"on_update": "sample_app.events.audit",
	},
}
```

```python
# sample_app/events.py
import frappe


def check_credit_limit(doc, method):
	if doc.total > 100000:
		frappe.throw(frappe._("Total is over the credit limit"))


def audit(doc, method):
	frappe.logger().info(f"{doc.doctype} {doc.name} updated")
```

The handler gets two positional arguments: the document and the event name. Every handler must accept both. Handlers run after the controller method. See [Controllers](../data-model/controllers.md) for the event order.

!!! warning
    Do not call `frappe.db.commit()` in a `doc_events` handler. In v15 the call commits the open transaction. A later error in the same save cannot roll back the work that was committed.

### Override controllers and methods

```python
# sample_app/hooks.py
override_doctype_class = {
	"ToDo": "sample_app.overrides.todo.CustomToDo",
}

override_whitelisted_methods = {
	"frappe.desk.doctype.event.event.get_events": "sample_app.event.get_events",
}
```

| Hook | Rule |
|---|---|
| `override_doctype_class` | The class must be a subclass of `Document`. Frappe raises `ImportError` if it is not. Subclass the original controller to keep its logic. |
| `override_whitelisted_methods` | Requests to the original dotted path call the new function. |

`override_doctype_class` has no effect on custom DocTypes (`custom = 1`). Custom DocTypes always use `Document` or `NestedSet`. See [Extending Apps](extending-apps.md).

### Desk and web assets

```python
# sample_app/hooks.py
app_include_js = "sample_app.bundle.js"
app_include_css = "sample_app.bundle.css"
web_include_js = "sample_app_web.bundle.js"
web_include_css = "sample_app_web.bundle.css"

doctype_js = {"ToDo": "public/js/todo.js"}
doctype_list_js = {"ToDo": "public/js/todo_list.js"}
```

| Hook | Effect |
|---|---|
| `app_include_js`, `app_include_css` | Load in every desk page. |
| `web_include_js`, `web_include_css` | Load in every website page. |
| `doctype_js` | Add a form script to a DocType of another app. Path is relative to the app package. |
| `doctype_list_js`, `doctype_tree_js`, `doctype_calendar_js` | Add view scripts the same way. |

See [Form Scripts](../desk-ui/form-scripts.md) and [List View Settings](../desk-ui/list-view-settings.md).

### Scheduled jobs

```python
# sample_app/hooks.py
scheduler_events = {
	"daily": ["sample_app.tasks.send_reminders"],
	"cron": {
		"0/15 * * * *": ["sample_app.tasks.sync_orders"],
	},
}
```

See [Scheduler](../runtime/scheduler.md).

### Permissions

```python
# sample_app/hooks.py
permission_query_conditions = {
	"Sample Order": "sample_app.permissions.sample_order_query",
}
has_permission = {
	"Sample Order": "sample_app.permissions.sample_order_has_permission",
}
```

See [Permissions](../security/permissions.md).

### Fixtures

```python
# sample_app/hooks.py
fixtures = [
	"Sample Role",
	{"dt": "Custom Field", "filters": [["module", "=", "Sample App"]]},
]
```

An entry is a DocType name or a dict with `dt` (or `doctype`), `filters` and `or_filters`. See [Fixtures and Patches](fixtures-and-patches.md).

### Requests, sessions and templates

| Hook | Shape | Effect |
|---|---|---|
| `before_request`, `after_request` | list of dotted paths | Run before and after each HTTP request. `after_request` functions get `response` and `request`. |
| `boot_session` | list of dotted paths | Each function gets `bootinfo` and can add keys for the desk. |
| `extend_bootinfo` | list of dotted paths | Each function gets `bootinfo=` as a keyword argument. |
| `jinja` | dict with `methods` and `filters` | Add functions and filters to the Jinja environment. |
| `website_route_rules` | list of `{"from_route", "to_route"}` | Map URL patterns to pages. See [Website](../website/website.md). |

## Pitfalls

- A hook value that is a function object, class or module is ignored. Always use a dotted path string.
- Names that start with `_` are not hooks. Use them for private helpers in `hooks.py`.
- A typo in a hook key fails without an error. Frappe reads only the keys it knows.
- Two apps that override the same DocType class do not both apply. Only the last one wins, without a warning.
- A `doc_events` handler with one parameter fails with `TypeError`. Accept `doc` and `method`.
- After you edit `hooks.py`, clear the cache and restart the bench processes.

## Related

- [Hooks reference](../../reference/configuration/hooks.md)
- [Controllers](../data-model/controllers.md)
- [Extending Apps](extending-apps.md)
- [Apps](apps.md)
- [Customizing APIs](../api/customizing-apis.md)
