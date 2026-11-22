---
title: Extending Other Apps
description: Change the behavior of DocTypes and methods from another app with doc_events, controller overrides and mixins, method overrides, doctype_js, custom fields and property setters.
version: v16
sources:
  - frappe/model/document.py
  - frappe/installer.py
  - frappe/migrate.py
  - frappe/__init__.py
  - frappe/handler.py
  - frappe/api/v2.py
  - frappe/desk/form/meta.py
  - frappe/desk/search.py
  - frappe/boot.py
  - frappe/model/meta.py
  - frappe/custom/doctype/custom_field/custom_field.py
  - frappe/custom/doctype/property_setter/property_setter.py
  - frappe/modules/utils.py
  - frappe/utils/boilerplate.py
---

# Extending Other Apps

Use the hooks on this page when your app must change a DocType or a method that another app owns. Do not edit the other app. Your changes stay in your app and survive updates of the other app. Each hook goes in your app's `hooks.py`. Frappe merges the hook values of all installed apps. For how merging works, see [Hooks](hooks.md).

## Choose a mechanism

| Need | Use |
|---|---|
| Run code when a document of another app is saved, submitted, deleted | `doc_events` |
| Add methods or change methods of another app's controller | `extend_doctype_class` (mixin) |
| Replace the controller class of a DocType | `override_doctype_class` |
| Replace a whitelisted method | `override_whitelisted_methods` |
| Add form, list, tree or calendar JavaScript to another app's DocType | `doctype_js`, `doctype_list_js`, `doctype_tree_js`, `doctype_calendar_js`, `doctype_kanban_js` |
| Add fields to another app's DocType | Custom Field (`create_custom_fields` or exported customizations) |
| Change a property of a field or DocType (label, hidden, reqd, options) | Property Setter (`make_property_setter`) |
| Replace the link search query of a DocType | `standard_queries` |
| Change the dashboard (connections) of a DocType | `override_doctype_dashboards` |
| Send extra data to the desk on page load | `boot_session` |

## doc_events

`doc_events` maps a DocType name (or `"*"` for all DocTypes) to controller events. Each event has a dotted path or a list of dotted paths.

```python
# sample_app/hooks.py
doc_events = {
	"ToDo": {
		"validate": "sample_app.events.todo.validate",
		"on_update": ["sample_app.events.todo.notify_owner"],
	},
	"*": {
		"on_trash": "sample_app.events.common.log_delete",
	},
}
```

```python
# sample_app/events/todo.py
import frappe
from frappe import _


def validate(doc, method=None):
	if doc.priority == "High" and not doc.date:
		frappe.throw(_("Set a due date for high priority tasks"))
```

- The handler gets the document and the event name. A handler that has no `method` parameter is also accepted: `def validate(doc)`.
- Handlers run after the controller method of the same name (`Document.hook` in `frappe/model/document.py`). Handlers of the DocType run before the handlers of `"*"`.
- `frappe.db.commit()` and `frappe.db.rollback()` inside a handler do not run. Frappe shows a warning and returns. Use `frappe.db.after_commit.add(fn)` for side effects that must happen after the commit.
- Any event that `run_method` calls works: `before_insert`, `validate`, `on_update`, `on_submit`, `on_cancel`, `on_trash`, `after_insert` and others. See [Controllers](../data-model/controllers.md) for the order.

## extend_doctype_class

`extend_doctype_class` adds mixin classes to the controller of a DocType. Frappe builds a new class with the mixins first and the original controller last. This lets several apps extend one DocType without conflict.

```python
# sample_app/hooks.py
extend_doctype_class = {
	"ToDo": "sample_app.overrides.todo.ToDoMixin",
}
```

```python
# sample_app/overrides/todo.py
import frappe


class ToDoMixin:
	def validate(self):
		super().validate()
		self.description = (self.description or "").strip()

	@frappe.whitelist()
	def mark_urgent(self):
		self.priority = "High"
		self.save()
```

- Call `super()` in a method that also exists on the controller. Otherwise the original method does not run.
- When more than one app extends a DocType, the mixin of the app listed last in the hook list comes first in the MRO.
- The extended class gets the name `Extended<ClassName>`. Code that checks `type(doc).__name__` sees this name.

## override_doctype_class

`override_doctype_class` replaces the controller class. Only the last value is used, so only one app can override a DocType.

```python
# sample_app/hooks.py
override_doctype_class = {
	"ToDo": "sample_app.overrides.todo.CustomToDo",
}
```

```python
# sample_app/overrides/todo.py
from frappe.desk.doctype.todo.todo import ToDo


class CustomToDo(ToDo):
	def on_update(self):
		super().on_update()
		# your code
```

- The class must be a subclass of the original controller. If it is not, `get_controller` raises "must be a subclass of" with the title "Invalid Override".
- `bench install-app` and `bench migrate` print a warning when more than one app overrides the same DocType.
- `extend_doctype_class` mixins are applied on top of the override. Prefer `extend_doctype_class` when you only add or change a few methods.

## override_whitelisted_methods

`override_whitelisted_methods` maps the dotted path of a whitelisted method to your replacement. The last app in the hook list wins.

```python
# sample_app/hooks.py
override_whitelisted_methods = {
	"frappe.desk.doctype.event.event.get_events": "sample_app.overrides.event.get_events",
}
```

- The override applies to `/api/method/<path>` and `/api/v2/method/<path>` (`frappe/handler.py`, `frappe/api/v2.py`).
- The replacement must also have `@frappe.whitelist()`. Frappe checks the replacement, not the original.
- Keep the same parameters as the original. Callers do not know about the override.
- The hook does not change Python calls to the original function. It changes only HTTP calls.

## doctype_js and related hooks

These hooks add JavaScript files from your app to the views of another app's DocType. The path is relative to your app package.

```python
# sample_app/hooks.py
doctype_js = {"ToDo": "public/js/todo.js"}
doctype_list_js = {"ToDo": "public/js/todo_list.js"}
```

```js
// sample_app/public/js/todo.js
frappe.ui.form.on("ToDo", {
	refresh(frm) {
		frm.add_custom_button(__("Mark Urgent"), () => {
			frm.call("mark_urgent").then(() => frm.reload_doc());
		});
	},
});
```

Frappe reads the file on each meta load and appends it to the form script (`frappe/desk/form/meta.py`). You do not need to build assets for these files. The value can be a list of paths. See [Form Scripts](../desk-ui/form-scripts.md) for the form API.

## Custom fields from code

Use `create_custom_fields` to add fields to another app's DocType. Call it from `after_install` and from `after_migrate`, so that the fields also exist on sites that update.

```python
# sample_app/hooks.py
after_install = "sample_app.setup.make_custom_fields"
after_migrate = "sample_app.setup.make_custom_fields"
```

```python
# sample_app/setup.py
from frappe.custom.doctype.custom_field.custom_field import create_custom_fields


def make_custom_fields():
	create_custom_fields(
		{
			"ToDo": [
				{
					"fieldname": "sample_code",
					"label": "Sample Code",
					"fieldtype": "Data",
					"insert_after": "description",
				},
			],
		}
	)
```

`create_custom_fields(custom_fields, ignore_validate=False, update=True)`:

| Name | Type | Default | Description |
|---|---|---|---|
| `custom_fields` | `dict` | | Key: a DocType name or a tuple of names. Value: one field dict or a list of field dicts. |
| `ignore_validate` | `bool` | `False` | Skip field validation. |
| `update` | `bool` | `True` | Update fields that exist. If `False`, only missing fields are inserted. |

- Fields are created with owner `Administrator` and `is_system_generated` set. Only Administrator can delete them from the desk.
- The function clears the DocType cache and adds the columns with `frappe.db.updatedb`.
- `delete_custom_fields({"ToDo": ["sample_code"]})` removes fields. It does not drop the database column.

### Exported customizations

In developer mode, **Customize Form** has an **Export Customizations** action. It writes Custom Fields and Property Setters of a DocType to `<app>/<module>/custom/<doctype>.json` (`export_customizations` in `frappe/modules/utils.py`). If the file has `"sync_on_migrate": 1`, `bench migrate` syncs it on every site. Use this when you build the fields in the desk. Use `create_custom_fields` when you prefer to keep fields in Python.

## Property setters from code

A Property Setter changes one property of a DocType or of a field, without changing the DocType JSON.

```python
from frappe.custom.doctype.property_setter.property_setter import make_property_setter

make_property_setter("ToDo", "priority", "default", "High", "Data")
make_property_setter("ToDo", None, "sort_field", "date", "Data", for_doctype=True)
```

`make_property_setter(doctype, fieldname, property, value, property_type, for_doctype=False, validate_fields_for_doctype=True, is_system_generated=True)`:

- Ignores permissions.
- Deletes the old Property Setter for the same DocType, field and property, then inserts the new one. The name is `<doctype>-<field or "main">-<property>`.
- Clears the DocType cache.

`frappe.make_property_setter(args, ...)` is a second form. It takes a dict with `doctype`, `fieldname`, `property`, `value` and optional `property_type`. If `doctype` is not set, it applies to every DocType that has a field with that `fieldname`.

## standard_queries

`standard_queries` replaces the search query that Link fields use for a DocType. Frappe uses the last value. The method must be whitelisted.

```python
# sample_app/hooks.py
standard_queries = {"Sample Item": "sample_app.queries.sample_item_query"}
```

```python
# sample_app/queries.py
import frappe


@frappe.whitelist()
@frappe.validate_and_sanitize_search_inputs
def sample_item_query(doctype, txt, searchfield, start, page_len, filters):
	return frappe.get_list(
		"Sample Item",
		filters={"disabled": 0, "item_name": ["like", f"%{txt}%"]},
		fields=["name", "item_name"],
		limit_start=start,
		limit_page_length=page_len,
		as_list=True,
	)
```

Frappe calls the method with `doctype, txt, searchfield, start, page_len, filters` (`frappe/desk/search.py`). A `query` set by a form script with `frm.set_query` takes priority over this hook.

## override_doctype_dashboards

Each method gets the dashboard data as `data` and returns the changed data. Frappe calls all methods in order (`frappe/model/meta.py`).

```python
# sample_app/hooks.py
override_doctype_dashboards = {"ToDo": "sample_app.overrides.todo.get_dashboard_data"}
```

```python
# sample_app/overrides/todo.py
def get_dashboard_data(data):
	data.setdefault("transactions", []).append({"label": "Sample", "items": ["Sample Order"]})
	return data
```

The `data` dict can be empty when the DocType has no `<doctype>_dashboard.py` file. Use `setdefault` for keys that can be missing.

## boot_session

Each method gets the `bootinfo` dict before the desk loads (`frappe/boot.py`). Add keys to it. The client reads them from `frappe.boot`.

```python
# sample_app/hooks.py
boot_session = "sample_app.boot.boot_session"
```

```python
# sample_app/boot.py
def boot_session(bootinfo):
	bootinfo.sample_app_settings = {"show_banner": True}
```

The method runs on every desk load. Keep it fast. Do not run queries per DocType or per record.

## Pitfalls

- `override_doctype_class` allows one winner. Two apps that override the same DocType break each other. Use `extend_doctype_class` instead.
- A `doc_events` handler runs for every save, also for imports and API calls. Check `doc.flags` or field changes with `doc.has_value_changed("field")` before you do heavy work.
- Do not call `frappe.db.commit()` in a `doc_events` handler. It does not run, and the transaction stays open.
- `create_custom_fields` only runs where you call it. Call it from `after_migrate` too, or sites that update do not get new fields.
- `doctype_js` files are not bundled. Do not use `import` statements in them.

## Related

- [Hooks](hooks.md)
- [Controllers](../data-model/controllers.md)
- [Fixtures and Patches](fixtures-and-patches.md)
- [Customize Form](../../user-guide/customization/customize-form.md)
- [Whitelisted Methods](../api/whitelisted-methods.md)
- [Hooks reference](../../reference/configuration/hooks.md)
