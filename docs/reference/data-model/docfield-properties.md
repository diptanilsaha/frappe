---
title: DocField Properties
description: Every DocField property from docfield.json with its type, default, the fieldtypes it applies to, its effect, and whether Customize Form and Custom Field can set it.
version: v16
sources:
  - frappe/core/doctype/docfield/docfield.json
  - frappe/custom/doctype/customize_form/customize_form.py
  - frappe/custom/doctype/custom_field/custom_field.json
  - frappe/custom/doctype/property_setter/property_setter.py
  - frappe/core/doctype/doctype/doctype.py
  - frappe/model/document.py
  - frappe/model/meta.py
  - frappe/model/__init__.py
  - frappe/database/schema.py
  - frappe/public/js/frappe/model/perm.js
  - frappe/public/js/frappe/form/layout.js
  - frappe/public/js/frappe/form/grid.js
  - frappe/public/js/frappe/form/controls/base_input.js
---

# DocField Properties

A DocField is one row in the `fields` table of a DocType. This page lists every DocField property. Use it to look up a property name, its type and default, and the fieldtypes that use it. For how the main properties behave on the desk and on the server, read [Field Properties](../../developer-guide/fields/field-properties.md).

## How this table is made

A script reads `frappe/core/doctype/docfield/docfield.json` in `field_order` and skips Section, Column and Tab Breaks. **Type**, **Default** and **Label** come from the JSON. **Applies to** comes from the `depends_on` of each property in the JSON, which controls when the DocType form shows it. Some rows add rules from the code. **Customize Form** is "Yes" when the property is a key of `docfield_properties` in `frappe/custom/doctype/customize_form/customize_form.py`. **Custom Field** is "Yes" when `frappe/custom/doctype/custom_field/custom_field.json` has a field with that name. **Effect** was written by hand from the code.

"Fields with a column" means all fieldtypes except the no-value fieldtypes in `frappe.model.no_value_fields`: Section Break, Column Break, Tab Break, Attachment Gallery, HTML, Table, Table MultiSelect, Button, Image, Fold, Heading and Vue Component. The DocType form also shows `search_index`, `unique` and `not_nullable` for some no-value fieldtypes. They have no effect there, because there is no column.

## All properties

| Property | Type | Default | Label | Applies to | Customize Form | Custom Field | Effect |
|---|---|---|---|---|---|---|---|
| `label` | Data |  | Label | All | Yes | Yes | Text next to the input. Frappe makes the fieldname from it when `fieldname` is empty. |
| `fieldtype` | Select | `Data` | Type | All | Yes | Yes | Sets the control, the database column and the validation. See [Field Types](field-types.md). |
| `fieldname` | Data |  | Name | All |  | Yes | Column name and document key. Lowercase, at most 64 characters, no special characters. |
| `length` | Int |  | Length | Data, Link, Dynamic Link, Password, Select, Read Only, Attach, Attach Image, Int, Float, Currency, Percent | Yes | Yes | Size of the varchar column and the maximum length checked on save. Int with length above 11 becomes `bigint`. |
| `precision` | Select |  | Precision | Float, Currency, Percent | Yes | Yes | Decimal places. Also sets the scale of the decimal column. |
| `non_negative` | Check | `0` | Non Negative | Int, Float, Currency, Percent | Yes | Yes | The server rejects a value below 0 with `frappe.NonNegativeError`. |
| `hide_days` | Check | `0` | Hide Days | Duration | Yes | Yes | Duration control hides the days input. |
| `hide_seconds` | Check | `0` | Hide Seconds | Duration | Yes | Yes | Duration control hides the seconds input. |
| `reqd` | Check | `0` | Mandatory | All except Section Break, Column Break, Button, HTML, Attachment Gallery | Yes | Yes | Value is required. The desk blocks save. The server raises `frappe.MandatoryError`. |
| `is_virtual` | Check | `0` | Virtual | All except Link, Attachment Gallery | Yes | Yes | No database column. The value comes from a controller property or the `options` expression. |
| `search_index` | Check | `0` | Index | Fields with a column |  | Yes | Adds a database index. Not allowed for Text, Long Text, Small Text, Code, Text Editor. |
| `not_nullable` | Check | `0` | Not Nullable | Fields with a column, except Check, Currency, Float, Int, Percent, Rating, Select |  |  | The column is `NOT NULL`. Frappe stores the default or an empty value instead of `NULL`. |
| `mask` | Check | `0` | Mask | Select, Read Only, Phone, Percent, Password, Link, Int, Float, Dynamic Link, Duration, Datetime, Currency, Data, Date | Yes | Yes | Users without the `mask` right at this permlevel see a masked value. |
| `options` | Small Text |  | Options | All except Attachment Gallery | Yes | Yes | Meaning depends on the fieldtype. See [Field Types](field-types.md). |
| `sort_options` | Check | `0` | Sort Options | Select | Yes | Yes | The Select dropdown shows options sorted by label. |
| `show_dashboard` | Check | `0` | Show Dashboard | Tab Break | Yes | Yes | The form dashboard shows in this tab. |
| `link_filters` | JSON |  | Filters | Attachment Gallery, Link | Yes | Yes | Filters for the Link search dropdown, as JSON `[[doctype, fieldname, operator, value]]`. Desk only. |
| `default` | Small Text |  | Default | All except Attachment Gallery | Yes | Yes | Value for new documents. See [Field Properties](../../developer-guide/fields/field-properties.md#default). |
| `fetch_from` | Small Text |  | Fetch From | Fields with a column | Yes | Yes | `link_field.source_field`. Copies a value from the linked document. |
| `fetch_if_empty` | Check | `0` | Fetch on Save if Empty | Fields with `fetch_from` | Yes | Yes | Fetch only when the field is empty. Without it, the server fetches again on every save. |
| `button_color` | Select |  | Button Color | Button | Yes | Yes | Style of the button. |
| `hidden` | Check | `0` | Hidden | All | Yes | Yes | The desk does not show the field. The server does not check it. |
| `show_on_timeline` | Check | `0` | Show on Timeline | Hidden fields, all except Attachment Gallery |  |  | Changes to this hidden field show on the form timeline. |
| `bold` | Check | `0` | Bold | All except Attachment Gallery | Yes | Yes | The desk shows the value in bold. Mandatory fields are always bold. |
| `allow_in_quick_entry` | Check | `0` | Allow in Quick Entry | All except Tab Break, Table, Attachment Gallery | Yes | Yes | The field shows in the quick entry dialog. |
| `translatable` | Check | `0` | Translatable | Data, Select, Text, Small Text, Text Editor | Yes | Yes | Shows a translate button that saves `Translation` records for the value. |
| `print_hide` | Check | `0` | Print Hide | All except Attachment Gallery | Yes | Yes | The field does not show in the standard print format. |
| `print_hide_if_no_value` | Check | `0` | Print Hide If No Value | Int, Float, Currency, Percent | Yes | Yes | The field does not show in print when the value is 0. |
| `report_hide` | Check | `0` | Report Hide | All except Attachment Gallery | Yes | Yes | The field is not available in Report view. |
| `in_import_template` | Check | `0` | Include in Import Template | All | Yes | Yes | **Select Mandatory** in the Data Import export dialog also selects this field. |
| `depends_on` | Code |  | Display Depends On (JS) | All | Yes | Yes | JavaScript condition. The field shows only when it is true. Desk only. |
| `collapsible` | Check | `0` | Collapsible | Section Break | Yes | Yes | The section can collapse. It starts collapsed. |
| `collapsible_depends_on` | Code |  | Collapsible Depends On (JS) | Section Break with `collapsible` | Yes | Yes | JavaScript condition. The section opens when it is true. |
| `hide_border` | Check | `0` | Hide Border | Section Break | Yes | Yes | The section has no top border. |
| `in_list_view` | Check | `0` | In List View | All except Attachment Gallery and virtual fields | Yes | Yes | Column in list view. In a child table, column in the grid. |
| `in_standard_filter` | Check | `0` | In Standard Filter | All except Attachment Gallery | Yes | Yes | Filter field in the list view filter bar. |
| `in_preview` | Check | `0` | In Preview | All except Table, Table MultiSelect, Attachment Gallery | Yes | Yes | Shows in the link preview popup and on Kanban cards. |
| `sticky` | Check | `0` | Sticky | All except Table, Table MultiSelect, Attachment Gallery |  |  | The grid column stays visible when the grid scrolls sideways. |
| `in_filter` | Check | `0` | In Filter | All except Attachment Gallery |  |  | Not used by the code. |
| `in_global_search` | Check | `0` | In Global Search | Data, Select, Table, Text, Text Editor, Link, Small Text, Long Text, Read Only, Heading, Dynamic Link | Yes | Yes | Adds the value to the global search index. |
| `read_only` | Check | `0` | Read Only | All | Yes | Yes | The desk shows the value as text. The server does not block API writes. |
| `allow_on_submit` | Check | `0` | Allow on Submit | Any fieldtype, in a submittable DocType or a child table | Yes | Yes | The value can change after submit. |
| `ignore_user_permissions` | Check | `0` | Ignore User Permissions | Link, Dynamic Link, Table MultiSelect | Yes | Yes | Skips User Permission checks for this link. |
| `allow_bulk_edit` | Check | `0` | Allow Bulk Edit | Table | Yes | Yes | The grid shows **Download** and **Upload** buttons for rows. Cleared for non-table fields. |
| `make_attachment_public` | Check | `0` | Make Attachment Public (by default) | Attach, Attach Image, Attachment Gallery |  |  | The upload dialog makes new files public by default. |
| `permlevel` | Int | `0` | Perm Level | All except Section Break, Column Break, Tab Break | Yes | Yes | Permission level. Roles need rights at this level to read or write the field. |
| `ignore_xss_filter` | Check | `0` | Ignore XSS Filter | All except Attachment Gallery | Yes | Yes | The server does not sanitize HTML in the value. |
| `unique` | Check | `0` | Unique | Fields with a column (server allows only Data, Link, Read Only, Int) | Yes | Yes | Adds a unique index. A duplicate raises `frappe.UniqueValidationError`. |
| `no_copy` | Check | `0` | No Copy | All except Attachment Gallery | Yes | Yes | The desk does not copy the value when you duplicate a document. |
| `set_only_once` | Check | `0` | Set only once | All except Attachment Gallery | Yes | Yes | The value can be set only on insert. A later change raises `frappe.CannotChangeConstantError`. |
| `remember_last_selected_value` | Check | `0` | Remember Last Selected Value | Link | Yes | Yes | New documents get the last value that the user selected, until the page reloads. |
| `mandatory_depends_on` | Code |  | Mandatory Depends On (JS) | All except Attachment Gallery | Yes | Yes | JavaScript condition. The field is mandatory when it is true. Desk only. |
| `read_only_depends_on` | Code |  | Read Only Depends On (JS) | All | Yes | Yes | JavaScript condition. The field is read-only when it is true. Desk only. |
| `alignment` | Select |  | Alignment | Data, Int, Float, Currency, Percent | Yes | Yes | Text alignment of the input and the read-only value. |
| `print_width` | Data |  | Print Width | All except Attachment Gallery | Yes | Yes | Column width in print, in `px` or `%`. |
| `width` | Data |  | Width | All except Attachment Gallery | Yes | Yes | Column width in pixels in the grid and in Report view. |
| `max_height` | Data |  | Max Height | All except Attachment Gallery |  |  | Maximum height of the Text or Code control. Must end with `px` or `em`. |
| `columns` | Int |  | Columns | All except Attachment Gallery | Yes | Yes | Grid column width. Values 1 to 12 map to pixel widths. |
| `description` | Small Text |  | Description | All | Yes | Yes | Help text below the input. HTML is allowed. |
| `documentation_url` | Data |  | Documentation URL | All except Tab Break, Section Break, Column Break, Button, HTML, Attachment Gallery |  |  | Shows an info link next to the label. |
| `placeholder` | Data |  | Placeholder | All except Attachment Gallery | Yes | Yes | Placeholder text in the empty input. |
| `oldfieldname` | Data |  |  | All |  |  | Legacy. Hidden. |
| `oldfieldtype` | Data |  |  | All |  |  | Legacy. Hidden. Skips the Data options check when set to a type other than Data. |
| `show_description_on_click` | Check | `0` | Show Description on Click | All |  |  | Shows the description in a popup when the user clicks the label. |

## Customize Form

Customize Form saves each changed property of a standard field as one `Property Setter`. It saves only the keys of `docfield_properties`. These DocField properties are not in that list, so Customize Form cannot change them: `fieldname`, `search_index`, `not_nullable`, `make_attachment_public`, `show_on_timeline`, `sticky`, `in_filter`, `max_height`, `documentation_url`, `show_description_on_click`, `oldfieldname`, `oldfieldtype`.

`docfield_properties` also has `idx` (the row order) and `auto_repeat`, which are not DocField properties in `docfield.json`.

Customize Form also blocks these changes on standard fields (`allow_property_change`):

| Property | Rule |
|---|---|
| `fieldtype` | Only inside one group of `ALLOWED_FIELDTYPE_CHANGE`, for example `Currency`, `Float`, `Percent`. Virtual fields can change freely. |
| `length` | For a smaller length, saved values are checked only against the default length of the fieldtype (for example 140 for `Data`), not against the new length. |
| `reqd` | You cannot clear it when the standard field is mandatory. |
| `read_only` | You cannot clear it when the standard field is read-only. |
| `allow_on_submit` | You cannot set it when the standard field does not have it. |
| `in_list_view` | Not allowed for fieldtypes from `get_fields_not_allowed_in_list_view`. |
| `options` | Only for `Read Only`, `HTML` and `Data` fields (`ALLOWED_OPTIONS_CHANGE`). |
| `translatable` | Only for fieldtypes that support translation. |
| `is_virtual` | You cannot make a standard field virtual. |

A blocked change shows a message and is skipped. A wrong `fieldtype` change raises an error.

## Custom Field and Property Setter

- A `Custom Field` adds a new field. It has its own copy of most DocField properties (see the **Custom Field** column), plus `dt` (the DocType) and `insert_after`.
- A `Property Setter` stores one property value for one field (`doctype_or_field` = `DocField`). It does not check the property name, so code can set any property with `frappe.make_property_setter`. After an update it runs the same field checks as a DocType save (`validate_fields_for_doctype`).

```python
frappe.make_property_setter(
	{
		"doctype": "ToDo",
		"fieldname": "description",
		"property": "documentation_url",
		"value": "https://example.com/help",
		"property_type": "Data",
	}
)
```

## Related

- [Field Properties](../../developer-guide/fields/field-properties.md)
- [Field Types](field-types.md)
- [Customize Form](../../user-guide/customization/customize-form.md)
- [DocTypes](../../developer-guide/data-model/doctypes.md)
