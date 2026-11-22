---
title: DocTypes
description: The DocType JSON, standard and custom DocTypes, the main DocType properties, child tables, singles, trees and virtual DocTypes.
version: v15
sources:
  - frappe/core/doctype/doctype/doctype.py
  - frappe/core/doctype/doctype/doctype.json
  - frappe/core/doctype/doctype_action/doctype_action.json
  - frappe/core/doctype/doctype_link/doctype_link.json
  - frappe/core/doctype/doctype_state/doctype_state.json
  - frappe/modules/utils.py
  - frappe/modules/import_file.py
  - frappe/model/__init__.py
  - frappe/model/document.py
  - frappe/model/meta.py
  - frappe/model/virtual_doctype.py
  - frappe/model/db_query.py
  - frappe/desk/reportview.py
  - frappe/database/mariadb/schema.py
  - frappe/utils/nestedset.py
  - frappe/public/js/frappe/form/form.js
---

# DocTypes

A DocType defines a type of record: its fields, permissions, naming and behavior. Frappe makes a database table `tab<DocType name>` for each DocType and builds the form, list view and REST API from it. A record of a DocType is a document. Make a DocType for each kind of business record your app stores. Use [Customize Form](../../user-guide/customization/customize-form.md) to change a DocType of another app.

## Standard and custom DocTypes

| | Standard (`custom = 0`) | Custom (`custom = 1`) |
|---|---|---|
| Where it lives | JSON file in an app, also in the database | Database only |
| Who can make it | Administrator in developer mode | Users with DocType permission, any mode |
| Python controller | `<doctype>.py` class in the app | None. Uses `Document` (or `NestedSet` for trees). |
| Ships with the app | Yes. `bench migrate` syncs it. | No. Export it with fixtures if needed. |
| Can be virtual | Yes | No |

Saving a standard DocType outside developer mode raises `CannotCreateStandardDoctypeError`. Turn on `developer_mode` in `site_config.json` on your development site.

When you save a standard DocType in developer mode, Frappe writes these files into `<app>/<module>/doctype/<doctype>/`:

- `<doctype>.json`, the definition.
- `<doctype>.py`, the controller, if it does not exist.
- `<doctype>.js` and `test_<doctype>.py`, for DocTypes that are not child tables.
- `templates/` with `<doctype>.html` and `<doctype>_row.html` when `has_web_view` is on.

Existing files are not overwritten. Edit DocTypes in the desk form, not by hand in the JSON.

## The DocType JSON

```json
{
 "doctype": "DocType",
 "name": "Sample Item",
 "module": "Selling",
 "autoname": "field:item_name",
 "naming_rule": "By fieldname",
 "title_field": "item_name",
 "sort_field": "creation",
 "sort_order": "DESC",
 "track_changes": 1,
 "field_order": ["item_name", "category", "rate"],
 "fields": [
  {"fieldname": "item_name", "fieldtype": "Data", "label": "Item Name", "reqd": 1, "unique": 1},
  {"fieldname": "category", "fieldtype": "Select", "label": "Category", "options": "Hardware\nSoftware\nService"},
  {"fieldname": "rate", "fieldtype": "Currency", "label": "Rate"}
 ],
 "permissions": [
  {"role": "System Manager", "read": 1, "write": 1, "create": 1, "delete": 1}
 ],
 "actions": [],
 "links": [],
 "states": []
}
```

| Key | Description |
|---|---|
| `fields` | Rows of `DocField`. Each has `fieldname`, `fieldtype`, `label`, `options` and flags such as `reqd`, `unique`, `in_list_view`, `read_only`, `hidden`, `permlevel`. See [Field Types](../../reference/data-model/field-types.md). |
| `field_order` | Order of fields in the form. |
| `permissions` | Rows of `DocPerm`: role, permission level and rights. See [Permissions](../security/permissions.md). |
| `actions`, `links`, `states` | Form buttons, connections and status colors. See below. |
| `modified` | Timestamp. `bench migrate` uses it with `migration_hash` to decide whether to import the file. |

### Standard columns

Every table has these columns in addition to your fields:

| Column | Description |
|---|---|
| `name` | Primary key. See [Naming](naming.md). |
| `owner` | User who made the document. |
| `creation`, `modified` | Timestamps. |
| `modified_by` | User who saved last. |
| `docstatus` | `0` draft, `1` submitted, `2` cancelled. |
| `idx` | Row order, used by child tables. |
| `parent`, `parentfield`, `parenttype` | Child tables only. Link the row to its parent document. |
| `_user_tags`, `_comments`, `_assign`, `_liked_by` | Desk data. Not added to child tables. `_seen` is added when `track_seen` is on. |

Parent tables also get an index on `modified`. Child tables get an index on `parent`.

## Main properties

| Property | Type | Default | Description |
|---|---|---|---|
| `module` | Link `Module Def` | | Module that owns the DocType. Sets the folder of the files. |
| `is_submittable` | Check | 0 | Documents can be submitted and cancelled. Adds `amended_from`. |
| `istable` | Check | 0 | Child table. Used only inside a Table field of another DocType. |
| `issingle` | Check | 0 | One record only. Values are stored in `tabSingles`. |
| `is_tree` | Check | 0 | Tree with nested set fields. See below. |
| `is_virtual` | Check | 0 | No database table. The controller loads and saves data. |
| `editable_grid` | Check | 1 | Child table rows can be edited in the grid. |
| `quick_entry` | Check | 0 | New records open in a dialog with the mandatory fields. |
| `track_changes` | Check | 0 | Save a `Version` with the changes on each save. |
| `track_seen` | Check | 0 | Mark the document as seen when a user opens it. |
| `track_views` | Check | 0 | Log each view in `View Log`. |
| `autoname`, `naming_rule` | Data, Select | | How the `name` is set. See [Naming](naming.md). |
| `allow_rename` | Check | 1 | Users can rename documents. |
| `allow_import` | Check | 0 | Allow Data Import. |
| `title_field` | Data | | Field to show as the title. |
| `search_fields` | Data | | Comma-separated fields shown in link search results. |
| `sort_field`, `sort_order` | Data, Select | `modified`, `DESC` | Default sort of the list view. |
| `image_field` | Data | | An `Attach Image` field to use as the image. |
| `has_web_view` | Check | 0 | Each document gets a web page at `route`. |
| `read_only` | Check | 0 | "User Cannot Search". |
| `in_create` | Check | 0 | "User Cannot Create" from the desk. |
| `max_attachments` | Int | | Max number of attachments. |
| `make_attachments_public` | Check | 0 | New attachments are public by default. |
| `queue_in_background` | Check | 0 | Submit and cancel in a background job (beta). |
| `is_calendar_and_gantt` | Check | 0 | Turn on calendar and Gantt views. |
| `email_append_to` | Check | 0 | "Allow document creation via Email". Needs `subject_field` and `sender_field`. |

For a Single, Frappe sets `allow_import`, `allow_rename`, `is_submittable` and `istable` to 0. For a child table, Frappe clears `permissions` and sets `allow_import` to 0. For `autoname = "autoincrement"`, Frappe sets `allow_rename` to 0.

To read properties in code, use `frappe.get_meta`:

```python
meta = frappe.get_meta("Sample Order")
meta.is_submittable           # 1
meta.has_field("customer_name")
meta.get_field("items").options   # "Sample Order Item"
meta.get_table_fields()
meta.get_link_fields()
```

## Child tables

A child table is a DocType with `istable = 1`. Add it to a parent through a `Table` or `Table MultiSelect` field whose `options` is the child DocType name. Rows are saved and loaded with the parent.

```python
order = frappe.new_doc("Sample Order")
order.customer_name = "Customer A"
order.append("items", {"item": "Support Plan", "qty": 2})
order.insert()

for row in order.items:
	print(row.idx, row.item, row.qty, row.parent)
```

- Child tables have no permissions of their own. Access follows the parent.
- Controller events run on the parent. Child rows get only `before_naming` and `autoname`. Field checks such as mandatory and select values also run on rows.
- `doc.remove(row)` removes a row. Saving the parent deletes it from the database.

## Single DocTypes

A Single has one record. Its `name` is the DocType name. Values are stored as rows in `tabSingles` with `doctype`, `field` and `value`.

```python
settings = frappe.get_single("System Settings")
settings.time_zone

frappe.db.get_single_value("System Settings", "time_zone")
frappe.db.set_single_value("Sample Settings", "enabled", 1)
```

`frappe.db.set_single_value` writes to the database without running the controller. Use `get_single` and `save` when the controller must validate.

## Trees

Set `is_tree = 1` to store a hierarchy with the nested set model. When you save, Frappe adds these fields if `lft` is missing:

| Field | Type | Description |
|---|---|---|
| `lft`, `rgt` | Int, hidden | Nested set bounds. |
| `is_group` | Check | Only group nodes can have children. |
| `old_parent` | Link | Previous parent, used when a node moves. |
| `parent_<doctype>` | Link | The parent node, labeled "Parent <DocType>". Set as `nsm_parent_field`. |

The controller of a standard tree DocType inherits from `NestedSet`:

```python
# sample_app/selling/doctype/sample_category/sample_category.py
from frappe.utils.nestedset import NestedSet


class SampleCategory(NestedSet):
	pass
```

`NestedSet` updates `lft` and `rgt` in `after_insert`, `on_update` and `on_trash`. If you override these methods, call `super()`. It also blocks loops, a non-group parent, and deleting a node that has children.

| Function or method | Returns |
|---|---|
| `doc.get_parent()` | The parent document. |
| `doc.get_children()` | A generator of child documents. |
| `doc.get_ancestors()` | Names of all ancestors. |
| `frappe.utils.nestedset.get_descendants_of(doctype, name)` | Names of all descendants. Uses `get_list`, so permissions apply. |
| `frappe.utils.nestedset.get_ancestors_of(doctype, name)` | Names of all ancestors. |
| `frappe.utils.nestedset.get_root_of(doctype)` | Name of the root node. |
| `frappe.utils.nestedset.rebuild_tree(doctype)` | Recalculates `lft` and `rgt` for all nodes. |

## Virtual DocTypes

A virtual DocType has no table. Its controller reads and writes the data, for example from an external API or a file. Only standard DocTypes can be virtual.

When you save a virtual DocType, Frappe makes a controller with stubs for `db_insert`, `load_from_db`, `db_update`, `get_list`, `get_count` and `get_stats`. Implement them, and add `delete`:

```python
# sample_app/selling/doctype/sample_remote_order/sample_remote_order.py
import frappe
from frappe.model.document import Document


class SampleRemoteOrder(Document):
	def db_insert(self, *args, **kwargs):
		"""Save a new record in the backend."""

	def load_from_db(self):
		"""Load the record for self.name, then call self.init_from_dict(data)."""

	def db_update(self, *args, **kwargs):
		"""Update the record in the backend."""

	def delete(self, *args, **kwargs):
		"""Delete the record from the backend."""

	@staticmethod
	def get_list(args):
		"""Return a list of dicts for the list view and frappe.get_list."""

	@staticmethod
	def get_count(args):
		"""Return the number of records."""

	@staticmethod
	def get_stats(args):
		"""Return sidebar stats."""
```

Each static method gets one argument: a dict with the request arguments (`doctype`, `filters`, `fields` and others). On save, Frappe checks the controller. It shows a message if a static method is missing or an instance method still uses the `Document` version. The required interface is in `frappe/model/virtual_doctype.py`.

## Document actions, links and states

These child tables of DocType change the form.

| Table | Fields | Effect |
|---|---|---|
| `actions` (`DocType Action`) | `label`, `group`, `action_type` (`Server Action` or `Route`), `action`, `hidden` | Adds a button. `Server Action` calls the whitelisted dotted path in `action` with `doc`. `Route` opens the route in `action`. |
| `links` (`DocType Link`) | `link_doctype`, `link_fieldname`, `group`, `parent_doctype`, `table_fieldname`, `is_child_table`, `hidden` | Adds a group in the connections dashboard of the form. |
| `states` (`DocType State`) | `title`, `color` | Sets the indicator color for a value of the `status` field. |

A Server Action returns the document to update the form:

```python
# sample_app/selling/actions.py
import frappe


@frappe.whitelist(methods=["POST"])
def recalculate(doc):
	doc = frappe.get_doc(frappe.parse_json(doc))
	doc.check_permission("write")
	doc.save()
	return doc
```

## Module functions

A controller module can define two module-level functions that run when the DocType is saved:

| Function | When it runs |
|---|---|
| `on_doctype_update()` | After each save of a standard DocType, also when `bench migrate` imports a changed JSON. Use it to add database indexes. |
| `after_doctype_insert()` | After the DocType is made for the first time. |

```python
# sample_app/selling/doctype/sample_order/sample_order.py
import frappe


def on_doctype_update():
	frappe.db.add_index("Sample Order", ["customer_name", "order_date"])
```

## Pitfalls

- A fieldname cannot match a method or property of the controller. Frappe throws "Fieldname ... conflicting with a controller method".
- A DocType name must be unique across all apps on the site.
- Changing the `module` of a DocType does not move its files. Move or delete the old folder yourself.
- Deleting a DocType does not drop its database table.
- `override_doctype_class` does not apply to custom DocTypes.

## Related

- [Naming](naming.md)
- [Controllers](controllers.md)
- [Field Types](../../reference/data-model/field-types.md)
- [DocTypes reference](../../reference/data-model/doctypes.md)
- [Customize Form](../../user-guide/customization/customize-form.md)
- [Field properties](../fields/field-properties.md)
