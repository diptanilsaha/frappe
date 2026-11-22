---
title: Field Properties
description: How DocField properties behave on the desk and on the server - names, defaults, depends_on rules, fetch_from, constraints, permissions, list and print settings.
version: develop
sources:
  - frappe/core/doctype/docfield/docfield.json
  - frappe/core/doctype/doctype/doctype.py
  - frappe/model/document.py
  - frappe/model/create_new.py
  - frappe/model/meta.py
  - frappe/model/utils/mask.py
  - frappe/model/docfield.py
  - frappe/database/schema.py
  - frappe/permissions.py
  - frappe/desk/form/activity.py
  - frappe/desk/link_preview.py
  - frappe/desk/reportview.py
  - frappe/utils/global_search.py
  - frappe/custom/doctype/customize_form/customize_form.py
  - frappe/public/js/frappe/model/create_new.js
  - frappe/public/js/frappe/model/perm.js
  - frappe/public/js/frappe/form/layout.js
  - frappe/public/js/frappe/form/grid_row.js
  - frappe/public/js/frappe/form/save.js
  - frappe/public/js/frappe/form/script_manager.js
  - frappe/public/js/frappe/form/quick_entry.js
  - frappe/public/js/frappe/form/controls/base_control.js
  - frappe/public/js/frappe/form/controls/base_input.js
  - frappe/public/js/frappe/form/controls/link.js
  - frappe/public/js/frappe/utils/utils.js
---

# Field Properties

Each field of a DocType is a DocField row with properties such as `reqd`, `default` or `depends_on`. This page explains how the main properties behave on the desk and on the server. Use it when you design a DocType or debug why a rule does not apply. For the full list of properties with types and defaults, read [DocField Properties](../../reference/data-model/docfield-properties.md). For fieldtypes, read [Field Types](../../reference/data-model/field-types.md).

## Where the rule runs

Many properties are desk rules only. A REST call, `frappe.get_doc(...).insert()` or a data import does not check them. This table shows where each rule runs. "Server" means `Document.insert()` and `Document.save()`.

| Property | Desk | Server |
|---|---|---|
| `reqd` | Blocks save, marks the label | Raises `frappe.MandatoryError` |
| `mandatory_depends_on` | Blocks save when the condition is true | Not checked |
| `depends_on`, `read_only_depends_on`, `collapsible_depends_on` | Evaluated | Not evaluated |
| `hidden` | Field not shown | Not checked |
| `read_only` | Value shown as text | Not checked. API writes are saved. |
| `set_only_once` | Read-only after insert | Raises `frappe.CannotChangeConstantError` |
| `allow_on_submit` | Editable after submit | Other fields raise `frappe.UpdateAfterSubmitError` |
| `unique` | - | Unique index. Raises `frappe.UniqueValidationError`. |
| `non_negative` | - | Raises `frappe.NonNegativeError` |
| `min_value`, `max_value` | - | Raises `frappe.ValidationError` |
| `length` | - | Raises `frappe.CharacterLengthExceededError` |
| `permlevel` | Read or write per role | Values reset on save, removed on read |
| `fetch_from` | Copies on link change | Copies on save |
| `link_filters` | Filters the dropdown | Not checked |
| `default` | Set on new form | Set on insert |

## Identity

### `fieldname`

- The column name in `tab<DocType>` and the key in the document.
- If `fieldname` is empty, the DocType save makes it from `label`: strip, lowercase, spaces to `_`, `?` removed. Section, Column and Tab Breaks get the suffix `_section`, `_column` or `_tab`.
- The save removes these characters: `' " , . / % @ ( ) < > { }`. Then it lowercases the name.
- A name with other non-word characters raises `frappe.db.InvalidColumnName`.
- The maximum length is 64 characters. On Postgres it is 63 bytes.
- These names are restricted: `name`, `parent`, `creation`, `owner`, `modified`, `modified_by`, `parentfield`, `parenttype`, `file_list`, `flags`, `docstatus`. A typed name raises `InvalidFieldNameError`. A name made from a label gets the suffix `1`, for example `owner1`.
- SQL reserved words (`RESERVED_KEYWORDS` in `frappe/model/__init__.py`) are not allowed. `autoname` is reserved in all DocTypes except `DocType`.
- Fieldnames must be unique in a DocType.

### Label, description and help

| Property | Behavior |
|---|---|
| `label` | Shown next to the input and translated with `__()`. |
| `description` | Shown below the input. HTML is allowed. `<code>` gets syntax colors. |
| `show_description_on_click` | The description shows in a popup when the user clicks the label, not below the input. |
| `documentation_url` | Adds an info popup with the link next to the label. Not for layout fields, Button and HTML. |
| `placeholder` | Placeholder text of the empty input. Color and Icon fields have their own placeholder when empty. |

## Default

`default` sets the value of a new document. The desk sets it when you open a new form (`frappe.model.get_default_value`). The server sets it on insert for each field that has no value (`_set_defaults`), and in `frappe.new_doc`. The server skips defaults during data import (`frappe.flags.in_import`).

| `default` | Desk | Server |
|---|---|---|
| A static value | That value | That value |
| `Today` | `frappe.datetime.get_today()` | `frappe.utils.nowdate()` |
| `now` (Datetime) | Current system date and time | `now_datetime()` |
| `now` (Time) | Current time | `nowtime()` |
| `__user` | `frappe.session.user` | `frappe.session.user` |
| `user` | `frappe.session.user` | The text `user` |
| `user_fullname` | Full name of the user | The text `user_fullname` |
| `:DocType`, for example `:Company` | Value of this field in the record named by the user default of `company` | Same. In a child row, the name comes from the parent field `company`. |

Rules:

- A Select field without a default gets its first option.
- A Check field gets `0` when the default is empty. The DocType save rejects a Check default other than `0` or `1`.
- A Select default must be one of the options.
- The default is ignored for the DocType `title_field`.
- `Today` is case-sensitive. `now` is not.

### User defaults for Link fields

For a Link field (not a Link to `User`), the user's defaults come before `default`:

1. If a User Permission for the linked DocType has **Is Default**, or the user has only one allowed record, use it. `ignore_user_permissions` turns this off.
2. Otherwise use the user default with the key `fieldname`. Set it with `frappe.defaults.set_user_default("company", "Example Corp")`.
3. On the desk, if `remember_last_selected_value` is set, use the last value that the user selected in this Link field. The desk keeps it in memory until the page reloads.

A default that User Permissions do not allow is not used.

### `not_nullable`

The column is `NOT NULL`. When the value is `None`, Frappe stores `default` or an empty value for the fieldtype, for example `""` for Data. Check, numbers, Rating and Select already have a non-null column, so the property does not apply to them.

## Display rules

`depends_on`, `mandatory_depends_on`, `read_only_depends_on` and `collapsible_depends_on` take a condition. The desk evaluates it in `frappe.ui.form.Layout.evaluate_depends_on_value` and in `GridRow` for child rows. The server only checks the syntax on DocType save.

| Condition | Result |
|---|---|
| `eval:<JavaScript>` | The value of the expression. `doc` is the current document or child row. `parent` is the form document. Globals such as `frappe` and `in_list` are available. |
| `fieldname` | True when `doc[fieldname]` has a value. For a table field, true when it has rows. |

```json
{
	"fieldname": "discount_reason",
	"fieldtype": "Small Text",
	"label": "Discount Reason",
	"depends_on": "eval:doc.discount > 0",
	"mandatory_depends_on": "eval:doc.discount > 10",
	"read_only_depends_on": "eval:doc.docstatus === 1"
}
```

- `depends_on`: the field shows only when the condition is true. The desk sets `hidden_due_to_dependency`.
- `mandatory_depends_on`: the desk sets `reqd` when true. The desk checks it before save.
- `read_only_depends_on`: the desk sets `read_only` when true.
- `collapsible_depends_on`: for a Section Break with `collapsible`. The section opens when true. A section with a missing mandatory field always opens.
- The desk evaluates the conditions again after each value change.
- The DocType save rejects a condition with a single `=` assignment, for example `eval:doc.status = "Open"`, with "Invalid ... condition".

!!! warning
    Do not use these conditions for data rules. The server does not evaluate them. Check the same rule in `validate` of the controller.

## `fetch_from` and `fetch_if_empty`

`fetch_from` copies a value from a linked document. The format is `link_fieldname.source_fieldname`.

```json
{
	"fieldname": "rate",
	"fieldtype": "Currency",
	"label": "Rate",
	"fetch_from": "item.rate",
	"fetch_if_empty": 1
}
```

**Desk.** When the user changes the Link field, the desk calls `frappe.client.validate_link_and_fetch` and sets the target fields. The desk sets up the fetch for Data, Read Only, Text, Small Text, Currency, Check, Text Editor, Attach Image, Code, Link, Float, Int, Date, Datetime, Select, Duration, Time, Percent, Phone, Barcode, Autocomplete, Icon, Color and Rating fields, and for any `read_only` or virtual field. Without `fetch_if_empty`, the target is read-only while the Link field has a value.

**Server.** On save, `_validate_links` reads the linked record and sets the target field:

- Without `fetch_if_empty`, the server fetches again on every save and overwrites the value.
- With `fetch_if_empty`, the server fetches only when the target is empty.
- After submit, it fetches only into fields with `allow_on_submit`.
- If the source field does not exist, a Data, Small Text or Text target raises "Wrong Fetch From value".
- A Text Editor source copied to Data, Small Text or Text loses its HTML tags.
- `fetch_from` cannot point to the field itself.

**DocType save.** When you add a `fetch_from` field, the DocType save fills the field in existing records that are empty.

## Constraints

| Property | Rule |
|---|---|
| `reqd` | The value must not be empty. HTML with no text counts as empty, except an `<img>` in Text Editor and HTML Code. Check always passes. A table needs one row. `doc.flags.ignore_mandatory = True` skips the check. A field cannot be `hidden` and `reqd` without `default`. |
| `read_only` | The desk shows the value as text. The `Read Only` fieldtype and virtual fields are always read-only on the desk. |
| `hidden` | The desk does not show the field. The value is still in the document and in the API. |
| `set_only_once` | The value can be set only on insert. A change after insert fails, also from empty to a value. For a table, any row change fails. |
| `allow_on_submit` | The field stays editable after submit. On a submitted document, changing a field without it raises `frappe.UpdateAfterSubmitError`. New rows in a table with `allow_on_submit` are allowed. |
| `unique` | A unique index on the column. Allowed only for Data, Link, Read Only and Int. An empty string is stored as `NULL`, so many empty values are allowed. The DocType save fails when saved values are not unique. |
| `non_negative` | Int, Float, Currency and Percent. A value below 0 fails. |
| `min_value`, `max_value` | Int, Float, Currency and Percent. `0` means not set, so `min_value: 0` does nothing. Use `non_negative` for that. An empty value passes. |
| `length` | The maximum length for varchar fieldtypes, checked on save. The column is at least `varchar(64)`. Int with length above 11 becomes `bigint`. |
| `no_copy` | **Duplicate** on the desk (`frappe.model.copy_doc`) leaves the field empty. `frappe.copy_doc(doc)` in Python copies it. Pass `ignore_no_copy=False` to leave it empty. |

## Permissions and security

| Property | Behavior |
|---|---|
| `permlevel` | Field-level permission. A role needs rights at this level to read or write the field. On save, values at a level without write access are reset to the saved or default values. On read, `frappe.read_doc` and the desk form remove fields without read access. Administrator is not restricted. See [Permissions](../security/permissions.md). |
| `ignore_user_permissions` | Link, Dynamic Link and Table MultiSelect. User Permissions on the linked DocType do not restrict the value, the link search or the default. |
| `mask` | Users without the `mask` right at the field's `permlevel` see a masked value in forms, lists, reports and prints. Phone shows the first 3 characters plus `XXXXXX`. Email shows `XXXXXX@domain`. Date shows `XX-XX-XXXX`. Other types show `XXXXXXXX`. Administrator sees the real value. |
| `ignore_xss_filter` | On save, `_sanitize_content` does not sanitize HTML in this field. Code, JSON, Attach, Attach Image, Barcode and Email fields are never sanitized. |
| `translatable` | Data, Select, Text, Small Text and Text Editor only. The DocType save clears it for other fieldtypes. Users who can write `Translation` see a translate button next to the field. The button saves `Translation` records for the value. |

## `is_virtual`

A virtual field has no database column. Its value comes from a property of the controller class. If there is no property, Frappe evaluates `options` as a Python expression with `doc` in scope (`frappe.safe_eval`).

```python
# sample_app/selling/doctype/sample_order/sample_order.py
from frappe.model.document import Document


class SampleOrder(Document):
	@property
	def item_count(self):
		return len(self.items)
```

- `doc.as_dict()`, the REST API and the desk form include the value. `doc.get("item_count")` returns `None` when only `options` is set.
- `frappe.get_all` and `frappe.db.get_value` cannot select a virtual field. There is no column.
- The desk shows a virtual field as read-only.
- A virtual Table field is a computed child table.

## `link_filters`

`link_filters` limits the rows in the Link search dropdown. It is JSON: a list of `[doctype, fieldname, operator, value]`. A value that starts with `eval:` is evaluated on the desk with `doc`, `parent` and `frappe`.

```json
{
	"fieldname": "item",
	"fieldtype": "Link",
	"options": "Sample Item",
	"link_filters": "[[\"Sample Item\", \"disabled\", \"=\", 0]]"
}
```

The DocType save checks that each filter has four values. For Attachment Gallery, each filter must target `File`. The server does not check `link_filters` when it saves the document.

## List, search and grid

| Property | Effect |
|---|---|
| `in_list_view` | A column in list view. In a child DocType, a column in the grid. Not for virtual fields in list view. |
| `in_standard_filter` | A filter in the list view filter bar. |
| `in_global_search` | Adds the formatted value to global search. |
| `in_preview` | Shows in the link preview popup when the DocType has `show_preview_popup`, and on Kanban cards. Without any `in_preview` field, the popup shows mandatory fields. |
| `bold` | Bold value. Mandatory fields are always bold. |
| `columns` | Grid column width. 1 to 12 map to pixels, for example `2` is 100px and `4` is 200px. |
| `width` | Grid and Report view column width in pixels. |
| `sticky` | The grid column stays visible when the grid scrolls sideways. |
| `allow_in_quick_entry` | Shows in the quick entry dialog. Mandatory fields always show there. Read-only and virtual fields do not. |
| `allow_bulk_edit` | Table fields only. The grid shows **Download** and **Upload** buttons for rows. |
| `report_hide` | Report view drops the field. |
| `search_index` | A database index on the column. `unique` fields do not need it. |
| `in_import_template` | **Select Mandatory** in the Data Import export dialog also selects this field. |

## Print

| Property | Effect |
|---|---|
| `print_hide` | The standard print format does not show the field. |
| `print_hide_if_no_value` | Int, Float, Currency and Percent. Print hides the field when the value is 0. |
| `print_width` | Column width in print, with `px` or `%`. |

## History

| Property | Effect |
|---|---|
| `ignore_versioning` | Changes to the field are not saved in `Version`. |
| `show_on_timeline` | For a `hidden` field. Changes still show on the form timeline. |

## Change properties on one site

Do not edit the DocType JSON of another app. Use these instead:

- **Customize Form** saves a `Property Setter` for each changed property of a standard field. See [Customize Form](../../user-guide/customization/customize-form.md).
- A **Custom Field** adds a new field with its own properties.
- `frappe.make_property_setter` in a patch or a fixture sets one property from code.
- `frm.set_df_property` changes a property in one open form only. See [Form Scripts](../desk-ui/form-scripts.md).

The list of properties that Customize Form can change is in [DocField Properties](../../reference/data-model/docfield-properties.md#customize-form).

## Pitfalls

- `read_only`, `hidden` and all `*_depends_on` rules do not protect data. Check the rule in the controller.
- `min_value: 0` and `max_value: 0` do nothing.
- `set_only_once` blocks a change from empty to a value after insert.
- `fetch_from` without `fetch_if_empty` overwrites manual values on every save.
- The defaults `user` and `user_fullname` work only on the desk. Use `__user` for the user ID.
- `frappe.copy_doc` copies `no_copy` fields unless you pass `ignore_no_copy=False`.
- A `fn:<event>` condition always evaluates as true on the desk. The form event returns a promise. Use `eval:` instead.

## Related

- [DocField Properties](../../reference/data-model/docfield-properties.md)
- [Field Types](../../reference/data-model/field-types.md)
- [DocTypes](../data-model/doctypes.md)
- [Controllers](../data-model/controllers.md)
- [Form Scripts](../desk-ui/form-scripts.md)
- [Permissions](../security/permissions.md)
- [Customize Form](../../user-guide/customization/customize-form.md)
- [Text Fields](text-fields.md), [Number Fields](number-fields.md), [Link and Select Fields](link-and-select-fields.md), [Table Fields](table-fields.md)
