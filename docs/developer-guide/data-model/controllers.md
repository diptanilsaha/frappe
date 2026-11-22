---
title: Controllers
description: The Document class, controller lifecycle methods and their exact run order, doc_events, flags, db_set, change detection and whitelisted controller methods.
version: v16
sources:
  - frappe/model/document.py
  - frappe/model/base_document.py
  - frappe/model/delete_doc.py
  - frappe/model/naming.py
  - frappe/model/rename_doc.py
  - frappe/__init__.py
  - frappe/handler.py
  - frappe/api/v1.py
  - frappe/api/v2.py
  - frappe/desk/form/load.py
  - frappe/www/printview.py
  - frappe/core/doctype/server_script/server_script_utils.py
  - frappe/database/database.py
  - frappe/modules/utils.py
  - frappe/types/DF.py
  - frappe/types/exporter.py
  - frappe/public/js/frappe/form/form.js
---

# Controllers

A controller is the Python class of a DocType. It inherits from `frappe.model.document.Document` and adds business logic with lifecycle methods such as `validate` and `on_submit`. Put validation and side effects for your own DocTypes in the controller. Use [`doc_events`](../building-apps/hooks.md#document-events) to add logic to DocTypes of other apps.

## The controller file

Frappe makes the file when you save a standard DocType in developer mode. The class name is the DocType name without spaces and hyphens.

```python
# sample_app/selling/doctype/sample_order/sample_order.py
import frappe
from frappe import _
from frappe.model.document import Document


class SampleOrder(Document):
	# begin: auto-generated types
	# This code is auto-generated. Do not modify anything in this block.

	from typing import TYPE_CHECKING

	if TYPE_CHECKING:
		from frappe.types import DF
		from sample_app.selling.doctype.sample_order_item.sample_order_item import SampleOrderItem

		amended_from: DF.Link | None
		customer_name: DF.Data
		items: DF.Table[SampleOrderItem]
		order_date: DF.Date
		status: DF.Literal["Draft", "Confirmed", "Delivered"]
		total: DF.Currency
	# end: auto-generated types

	def validate(self):
		self.total = sum(row.qty * row.rate for row in self.items)
		if not self.items:
			frappe.throw(_("Add at least one item"), title=_("Missing Items"))

	def on_submit(self):
		self.db_set("status", "Confirmed")
```

Custom DocTypes have no controller file. They use `Document`, or `NestedSet` for trees.

### Type hints

The block between `# begin: auto-generated types` and `# end: auto-generated types` lists the fields with types from `frappe.types.DF`. Frappe rewrites it when you save the DocType in developer mode, if the app sets `export_python_type_annotations = True` in `hooks.py`. Do not edit the block. The `hooks.py` of a new app has this line as a comment. Remove the `#` to turn it on.

| `DF` type | Python type |
|---|---|
| `DF.Data`, `DF.Link`, `DF.Text`, `DF.SmallText` ... | `str` |
| `DF.Int` | `int` |
| `DF.Float`, `DF.Currency`, `DF.Percent` | `float` |
| `DF.Check` | `bool \| int` |
| `DF.Date`, `DF.Datetime`, `DF.Time` | `str \| date`, `str \| datetime`, `str \| time` |
| `DF.Literal[...]` | A Select field with its options |
| `DF.Table[RowClass]` | `list` of child documents |

A field that is not mandatory gets `| None`.

## Lifecycle methods

Define any of these methods on the controller. Frappe calls each one through `run_method`, so `doc_events`, Notifications, Webhooks and Server Scripts for the same event also run.

| Method | When |
|---|---|
| `before_insert` | Before naming, on insert only. |
| `before_naming` | Before the name is set. |
| `autoname` | Sets `self.name`. See [Naming](naming.md). |
| `before_validate` | Before `validate`, on save and submit. Runs even with `flags.ignore_validate`. |
| `validate` | On save and submit. Throw here to stop the save. |
| `before_save` | After `validate`, on save of a draft. |
| `before_submit` | After `validate`, on submit. |
| `before_cancel` | On cancel. |
| `before_update_after_submit` | On save of a submitted document. |
| `after_insert` | After the row is inserted. Before `on_update`. |
| `on_update` | After the row is written, on insert, save and submit. |
| `on_submit` | After `on_update`, on submit. |
| `on_cancel` | After the row is updated, on cancel. |
| `on_update_after_submit` | After the row is written, on save of a submitted document. |
| `on_change` | At the end of every insert, save, submit, cancel, `db_set` and delete. |
| `before_change` | In `db_set`, before the database write. |
| `on_trash` | On delete, before the row is deleted. |
| `after_delete` | On delete, after the row is deleted. |
| `before_discard`, `on_discard` | Before and after a draft is discarded. |
| `before_rename(old, new, merge)`, `after_rename(old, new, merge)` | Around a rename. `before_rename` can return a new name. |
| `onload` | When the desk loads the document in a form. Use `self.set_onload(key, value)` to send extra data. |
| `before_print(settings)` | Before a print view renders. |

## Run order

These tables come from `frappe/model/document.py` and `frappe/model/delete_doc.py`. They were confirmed by tracing `run_method` while inserting, submitting, cancelling and deleting a `Sample Order` on a v16 site. Rows in bold are controller methods.

### Insert (`doc.insert()`)

1. Set defaults, `owner`, `creation` and `modified`.
2. Check `create` permission.
3. `check_if_latest`: load the stored copy and check the docstatus transition.
4. Validate Link fields.
5. **`before_insert`**
6. Set the name: **`before_naming`**, Document Naming Rule, **`autoname`**, `autoname` property. Child rows are named too.
7. Reset fields that the user cannot write at a higher permission level.
8. **`before_validate`**
9. **`validate`**
10. **`before_save`**, or **`before_submit`** if `docstatus` is 1.
11. Standard checks: mandatory, data formats, select options, length, sanitizing HTML, password fields, workflow.
12. Insert the parent row and all child rows.
13. **`after_insert`**
14. **`on_update`**
15. **`on_submit`**, if `docstatus` is 1.
16. Clear the document cache, publish `doc_update` and `list_update`, update global search, save a `Version`.
17. **`on_change`**

### Save a draft (`doc.save()`)

1. Check the document lock, set defaults.
2. Check `write` permission. Set `modified` and `modified_by`.
3. `check_if_latest`: throw `TimestampMismatchError` if the stored `modified` is newer than the copy you loaded.
4. Reset fields at higher permission levels. Validate Link fields.
5. **`before_validate`**
6. **`validate`**
7. **`before_save`**
8. Standard checks.
9. Update the parent row. Insert, update and delete child rows.
10. **`on_update`**
11. Cache, realtime, global search, `Version`.
12. **`on_change`**

`save()` on a new document calls `insert()`.

### Submit (`doc.submit()`)

Same as save, with `docstatus` set to 1 and `submit` permission checked:

**`before_validate`** > **`validate`** > **`before_submit`** > standard checks > update rows > **`on_update`** > **`on_submit`** > cache, realtime, `Version` > **`on_change`**

### Update after submit (`doc.save()` on a submitted document)

**`before_update_after_submit`** > standard checks > check that only fields with `allow_on_submit` changed > update rows > **`on_update_after_submit`** > cache, realtime, `Version` > **`on_change`**

`before_validate` and `validate` do not run. A change to another field throws `UpdateAfterSubmitError`.

### Cancel (`doc.cancel()`)

**`before_cancel`** > update rows with `docstatus` 2 > **`on_cancel`** > check that no submitted document links to it > cache, realtime, `Version` > **`on_change`**

Cancel checks `cancel` permission. It skips Link validation and the standard field checks.

### Delete (`doc.delete()` or `frappe.delete_doc`)

1. Check `delete` permission. Throw if the document is submitted.
2. **`on_trash`**
3. **`on_change`**, with `flags.in_delete` set.
4. Check that no other document links to it. Skipped with `force=True`.
5. Delete the row and child rows.
6. **`after_delete`**
7. Delete attachments. Enqueue the cleanup of dynamic links.
8. Clear cache, remove global search and tags, add a `Deleted Document` unless `delete_permanently=True`.

`ignore_on_trash=True` skips steps 2 and 3.

### Discard a draft (`doc.discard()`)

**`before_discard`** > `db_set("docstatus", 2)`, which runs **`before_change`** and **`on_change`** > **`on_discard`**

Only drafts can be discarded. It needs `write` permission. No validation runs.

### db_set

**`before_change`** > database update > **`on_change`**

## doc_events and controller order

For each event, `run_method` runs in this order:

1. The controller method.
2. `doc_events` handlers for the DocType, from all apps in app order.
3. `doc_events` handlers for `"*"`, from all apps in app order.
4. Notifications for the event.
5. Webhooks.
6. Server Scripts of type DocType Event.
7. Automations.

Server Script event names map to controller methods in `frappe/core/doctype/server_script/server_script_utils.py`. "Before Save" maps to `validate`, and "After Save" maps to `on_update`.

Handlers get `(doc, method)` or, with one parameter, `(doc)`. Events with arguments, such as `before_rename`, pass them after `method`. A handler runs even when the controller does not define the method.

```python
# sample_app/hooks.py
doc_events = {
	"Sample Order": {"validate": "sample_app.events.check_order"},
	"*": {"on_update": "sample_app.events.log_change"},
}
```

Frappe disables `frappe.db.commit()` and `frappe.db.rollback()` inside `doc_events` handlers. The call shows a warning and does nothing.

## Flags

Set flags on `doc.flags` before `insert`, `save` or `delete`.

| Flag | Effect |
|---|---|
| `ignore_permissions` | Skip permission checks. Also an argument of `insert` and `save`. |
| `ignore_links` | Skip Link validation. Also an argument of `insert`. |
| `ignore_mandatory` | Skip the mandatory field check. Also an argument of `insert`. |
| `ignore_validate` | Skip `validate`, `before_save`, `before_submit`, `before_cancel` and `before_update_after_submit`. `before_validate` and the standard checks still run. |
| `ignore_validate_update_after_submit` | Allow changes to fields without `allow_on_submit`. |
| `ignore_version` | Do not save a `Version`. Also an argument of `save`. Default is `True` in tests. |
| `notify_update` | Set to `False` to skip the `doc_update` and `list_update` realtime events. |
| `ignore_save_passwords` | `True`, or a list of fieldnames, to skip saving Password fields. |
| `in_insert` | Read only. `True` during insert. |
| `in_delete` | Read only. `True` in `on_change` during delete. |

`insert` also takes `ignore_if_duplicate=True` to return without error on a duplicate name, `set_name` to force a name, and `set_child_names=False`.

```python
doc = frappe.get_doc({"doctype": "Sample Order", "customer_name": "Customer A"})
doc.flags.ignore_mandatory = True
doc.insert(ignore_permissions=True)
```

!!! warning
    `ignore_permissions` lets the current user write data they cannot normally write. Use it only for system actions, never to fix a permission problem. See [Security](../security/security.md).

## db_set

`db_set` writes one or more fields to the database without the save cycle.

```python
doc.db_set("status", "Delivered")
doc.db_set({"status": "Delivered", "delivered_on": frappe.utils.today()})
doc.db_set("status", "Delivered", update_modified=False, notify=True)
```

| Parameter | Default | Description |
|---|---|---|
| `fieldname` | | Fieldname, or a dict of fieldnames and values. |
| `value` | `None` | New value. |
| `update_modified` | `True` | Update `modified` and `modified_by`. Inside `on_update` and other post-save methods, `modified` does not change. |
| `notify` | `False` | Publish `doc_update` for open forms. |
| `commit` | `False` | Commit the transaction. |

`db_set` skips validation, permission checks, mandatory checks and `Version`. It runs `before_change` and `on_change`. Use it in `on_submit` or `on_update` to set status fields without a second save.

## Detect changes

| Method | Returns |
|---|---|
| `doc.get_doc_before_save()` | The stored copy that was loaded at the start of `save`. `None` for a new document. |
| `doc.has_value_changed(fieldname)` | `True` if the field differs from the stored copy. Always `True` for a new document. |
| `doc.get_value_before_save(fieldname)` | The stored value, or `None` for a new document. |

These methods work in every lifecycle method of a save, including `on_update` and `on_change`. Child rows have their own `get_doc_before_save()`.

```python
class SampleOrder(Document):
	def on_update(self):
		if self.has_value_changed("status"):
			old = self.get_value_before_save("status")
			self.add_comment("Info", f"Status changed from {old} to {self.status}")
```

## Whitelisted controller methods

Add `@frappe.whitelist()` to a method to call it from the client or the REST API.

```python
class SampleOrder(Document):
	@frappe.whitelist(methods=["POST"])
	def apply_discount(self, percent: float):
		self.check_permission("write")
		for row in self.items:
			row.rate = row.rate * (1 - percent / 100)
		self.save()
		return self.total
```

Call it from a form script:

```js
frm.call("apply_discount", { percent: 10 }).then((r) => {
	frappe.show_alert(__("New total: {0}", [r.message]));
});
```

Or with REST:

```http
POST /api/v2/document/Sample Order/SO-00001/method/apply_discount
Content-Type: application/json

{"percent": 10}
```

| Route | Permission checked |
|---|---|
| `frm.call` (`run_doc_method`) | `read` on the document. The client sends the current form values, which can be unsaved. |
| `GET /api/v2/document/<doctype>/<name>/method/<method>` | `read` |
| `POST /api/v2/document/<doctype>/<name>/method/<method>` | `write` |
| `POST /api/resource/<doctype>/<name>` with `run_method` | `write` |
| `GET /api/resource/<doctype>/<name>?run_method=<method>` | `read` |

The call goes through `run_method`, so `doc_events` for the method name also run. The response includes the document. The form syncs it after `frm.call`. See [Whitelisted Methods](../api/whitelisted-methods.md) for `methods`, `allow_guest` and type checks.

## Other Document methods

| Method | Description |
|---|---|
| `frappe.get_doc(doctype, name)` | Load a document. Use `for_update=True` to lock the row. |
| `frappe.new_doc(doctype)` | A new document with defaults. |
| `doc.reload()` | Load the document again from the database. |
| `doc.append(fieldname, row)` | Add a child row. Returns the row. |
| `doc.check_permission(ptype)` | Throw `frappe.PermissionError` if the user lacks `ptype`. |
| `doc.has_permission(ptype)` | `True` or `False`. Returns `True` if `flags.ignore_permissions` is set. |
| `doc.is_new()` | `True` before insert. |
| `doc.add_comment(comment_type, text)` | Add a comment to the timeline. |
| `doc.queue_action(action, **kwargs)` | Run `submit`, `cancel` or another method in a background job. Locks the document. |
| `doc.get_title()` | Value of the title field, or the name. |

## Pitfalls

- Do not call `self.save()` inside `validate`, `on_update` or other lifecycle methods. Use `db_set` in post-save methods.
- Do not commit inside lifecycle methods or `doc_events`.
- `after_insert` runs before `on_update`. Code in `on_update` runs on both insert and save.
- `validate` does not run on cancel or on update after submit. Put checks for those in `before_cancel` and `before_update_after_submit`.
- A whitelisted method called with `frm.call` needs only `read` permission. Check `write` in the method if it changes data.
- A whitelisted method without `methods` accepts GET. Use `methods=["POST"]` for methods that change data.
- A fieldname cannot match a controller method name.

## Related

- [DocTypes](doctypes.md)
- [Naming](naming.md)
- [Hooks](../building-apps/hooks.md)
- [Extending Apps](../building-apps/extending-apps.md)
- [Form Scripts](../desk-ui/form-scripts.md)
- [Whitelisted Methods](../api/whitelisted-methods.md)
- [Database](../database/database.md)
