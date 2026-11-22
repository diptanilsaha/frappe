---
title: Field Types
description: Every DocField fieldtype with its database column type on MariaDB, Postgres and SQLite, what Options means, and validation rules.
version: v16
sources:
  - frappe/model/__init__.py
  - frappe/database/database.py
  - frappe/database/schema.py
  - frappe/database/mariadb/database.py
  - frappe/database/postgres/database.py
  - frappe/database/sqlite/database.py
  - frappe/core/doctype/docfield/docfield.json
  - frappe/core/doctype/doctype/doctype.py
  - frappe/model/meta.py
  - frappe/public/js/frappe/form/controls/code.js
  - frappe/public/js/frappe/form/controls/data.js
  - frappe/public/js/frappe/form/controls/date.js
  - frappe/public/js/frappe/form/controls/rating.js
  - frappe/public/js/frappe/form/controls/barcode.js
  - frappe/public/js/frappe/form/controls/icon.js
  - frappe/public/js/frappe/form/controls/image.js
  - frappe/public/js/frappe/form/controls/html.js
  - frappe/public/js/frappe/form/controls/button.js
  - frappe/public/js/frappe/form/controls/attach.js
  - frappe/public/js/frappe/form/controls/vue_component.js
---

# Field Types

The `fieldtype` of a DocField sets the database column, the desk control and the validation of a field. Use this page to choose a fieldtype and to set its `options`. The list of fieldtypes comes from the `fieldtype` options in `frappe/core/doctype/docfield/docfield.json`. Column types come from `setup_type_map()` of each database class.

## Groups of fieldtypes

`frappe/model/__init__.py` defines these groups:

| Constant | Fieldtypes | Meaning |
|---|---|---|
| `data_fieldtypes` | All types with a column, from `Currency` to `JSON` in the table below | Each field of these types gets a column in `tab<DocType>`. |
| `no_value_fields` | `Section Break`, `Column Break`, `Tab Break`, `Attachment Gallery`, `HTML`, `Table`, `Table MultiSelect`, `Button`, `Image`, `Fold`, `Heading`, `Vue Component` | No column in the parent table. |
| `display_fieldtypes` | `no_value_fields` minus `Table` and `Table MultiSelect` | Layout and display only. |
| `table_fields` | `Table`, `Table MultiSelect` | Rows are stored in the child DocType table. |
| `numeric_fieldtypes` | `Currency`, `Int`, `Long Int`, `Float`, `Percent`, `Check` | Numeric values. |
| `float_like_fields` | `Float`, `Currency`, `Percent` | Decimal values with precision. |
| `datetime_fields` | `Datetime`, `Date`, `Time` | Date and time values. |
| `attachment_fieldtypes` | `Attach`, `Attach Image` | Values are file URLs. |
| `data_field_options` | `Email`, `Name`, `Phone`, `URL`, `Barcode`, `IBAN` | Allowed `options` of a `Data` field. |

## Fieldtypes with a column

`VARCHAR_LEN` is 140 (`frappe/database/database.py`).

| Fieldtype | MariaDB | Postgres | SQLite | Options |
|---|---|---|---|---|
| `Data` | `varchar(140)` | `varchar(140)` | `TEXT` | Empty, or one of `Email`, `Name`, `Phone`, `URL`, `Barcode`, `IBAN`. These validate and format the value. |
| `Link` | `varchar(140)` | `varchar(140)` | `TEXT` | Required. Name of the linked DocType. |
| `Dynamic Link` | `varchar(140)` | `varchar(140)` | `TEXT` | Required. Fieldname of a `Link` field with options `DocType`, or of a `Select` field, in the same DocType. That field holds the target DocType. |
| `Select` | `varchar(140)` | `varchar(140)` | `TEXT` | List of values, one per line. A default must be one of them. |
| `Autocomplete` | `varchar(140)` | `varchar(140)` | `TEXT` | List of suggestions, one per line. |
| `Read Only` | `varchar(140)` | `varchar(140)` | `TEXT` | Not used. Use `fetch_from` to fill the value. |
| `Password` | `text` | `text` | `TEXT` | Not used. The value is encrypted in the `__Auth` table. The column holds a mask. |
| `Color` | `varchar(140)` | `varchar(140)` | `TEXT` | Not used. |
| `Icon` | `varchar(140)` | `varchar(140)` | `TEXT` | `Emojis` adds emojis to the picker. |
| `Phone` | `varchar(140)` | `varchar(140)` | `TEXT` | Not used. The control has a country code picker. |
| `Int` | `int(11)` | `int` | `INTEGER` | Not used. |
| `Long Int` | `bigint(20)` | `bigint` | `INTEGER` | Not used. Not in the DocField fieldtype list. An `Int` field with `length` above 11 gets this column. |
| `Check` | `tinyint(4)` | `smallint` | `INTEGER` | Not used. Value is `0` or `1`. |
| `Float` | `decimal(21,9)` | `decimal(21,9)` | `REAL` | Not used. Set `precision` for decimals. |
| `Currency` | `decimal(21,9)` | `decimal(21,9)` | `REAL` | Fieldname that holds the currency, or `DocType:link_field:currency_field` to read the currency from a linked document. |
| `Percent` | `decimal(21,9)` | `decimal(21,9)` | `REAL` | Not used. |
| `Rating` | `decimal(3,2)` | `decimal(3,2)` | `REAL` | Number of stars, 3 to 10. Default 5. The value is stored as a fraction from 0 to 1. |
| `Duration` | `decimal(21,9)` | `decimal(21,9)` | `REAL` | Not used. The value is in seconds. Use the `hide_days` and `hide_seconds` properties. |
| `Date` | `date` | `date` | `DATE` | Optional JSON object of datepicker options. |
| `Datetime` | `datetime(6)` | `timestamp` | `TIMESTAMP` | Optional JSON object of datepicker options. |
| `Time` | `time(6)` | `time(6)` | `TIME` | Not used. |
| `Small Text` | `text` | `text` | `TEXT` | Not used. |
| `Text` | `text` | `text` | `TEXT` | Not used. |
| `Long Text` | `longtext` | `text` | `TEXT` | Not used. |
| `Text Editor` | `longtext` | `text` | `TEXT` | Not used. Rich text (Quill). The value is HTML. |
| `Markdown Editor` | `longtext` | `text` | `TEXT` | Not used. The control sets the language to `Markdown`. |
| `HTML Editor` | `longtext` | `text` | `TEXT` | Not used. The control sets the language to `HTML`. |
| `Code` | `longtext` | `text` | `TEXT` | Editor language: `Javascript` or `JS`, `Python` or `Py`, `PythonExpression`, `HTML`, `CSS`, `SCSS`, `Markdown`, `JSON`, `Golang` or `Go`, `Jinja`, `SQL`. |
| `JSON` | `json` | `json` | `TEXT` | Not used. |
| `Attach` | `text` | `text` | `TEXT` | Not used in DocType JSON. The value is the file URL. |
| `Attach Image` | `text` | `text` | `TEXT` | Not used. Only image files are allowed. Set it as `image_field` of the DocType to show it in the form header. |
| `Signature` | `longtext` | `text` | `TEXT` | Not used. The value is a base64 image. |
| `Barcode` | `longtext` | `text` | `TEXT` | Optional JSON object of JsBarcode options. The value is the SVG of the barcode. |
| `Geolocation` | `longtext` | `text` | `TEXT` | Not used. The value is GeoJSON. |

## Fieldtypes without a column

| Fieldtype | Options | Notes |
|---|---|---|
| `Table` | Required. A child DocType (`istable = 1`). | Rows are stored in the child table with `parent`, `parenttype` and `parentfield`. A virtual DocType needs a virtual child table. |
| `Table MultiSelect` | Required. A child DocType with at least one `Link` field. | Shows the rows as pills of the first Link field. |
| `Section Break` | Not used. | Starts a section. |
| `Column Break` | Not used. | Starts a column in a section. |
| `Tab Break` | Not used. | Starts a tab. |
| `Fold` | Not used. | Hides the rest of the form behind a "Show more" link. Only one per form, and it must be before a `Section Break`. |
| `Heading` | Not used. | Shows the label as a heading. |
| `HTML` | HTML to show. | Form scripts can set the content with `frm.get_field(fieldname).$wrapper.html(...)`. |
| `Image` | Fieldname that holds the image URL. | Shows the image of that field. |
| `Button` | Optional. Name of a controller method. | A click runs the form script event of the field. If there is none and `options` is set, it calls the controller method through `run_doc_method`. The source marks this path as deprecated. |
| `Attachment Gallery` | Not used. | Shows attachments of the document. Its `link_filters` must target `File`. |
| `Vue Component` | Required. Name of a UI island on the site. | Mounts the island in the form. See [Islands](../../developer-guide/desk-ui/islands.md). |

## How a column is built

`get_definition()` in `frappe/database/schema.py` turns a DocField into a column type:

- `Currency`, `Float` and `Percent` get `decimal(length, precision)` on MariaDB and Postgres. The default is `decimal(21,9)`. `precision` must be from 1 to 6. `precision` cannot be greater than `length`.
- A `length` on a `varchar` field changes the size. Values below 64 become 64. SQLite stores these fields as `TEXT`, so `length` has no effect there.
- An `Int` field with `length` above 11 becomes `Long Int` (`bigint`). On MariaDB, a `length` below 11 sets `int(length)`.
- A `Link` to a DocType with `autoname = "UUID"` gets a `uuid` column.
- `Check`, `Int`, `Currency`, `Float` and `Percent` are `NOT NULL`. Their default is `0` if none is set.
- `not_nullable = 1` makes other columns `NOT NULL`.
- `unique = 1` adds a unique constraint, except for `text` and `longtext` columns.

## Validation rules

`validate_fields()` in `frappe/core/doctype/doctype/doctype.py` checks these rules when you save a DocType:

| Rule | Fieldtypes |
|---|---|
| Options must be a valid DocType | `Link`, `Table`, `Table MultiSelect` |
| Options must be a child table | `Table`, `Table MultiSelect` |
| Can be `unique` | `Data`, `Link`, `Read Only`, `Int` only |
| Cannot have `search_index` | `Text`, `Long Text`, `Small Text`, `Code`, `Text Editor` |
| Cannot be mandatory | `no_value_fields` except `Table` and `Table MultiSelect` |
| Not allowed `in_list_view` | `no_value_fields`. In a child table, `Button` and `HTML` are allowed. |
| Not allowed `in_global_search` | `no_value_fields` |
| Default must be `0` or `1` | `Check` |
| Precision from 1 to 6 | `Float`, `Currency`, `Percent` |
| Options from 3 to 10 | `Rating` |
| Hidden and mandatory needs a default | All |

## Desk controls

Each fieldtype has a control class `frappe.ui.form.Control<Fieldtype>` without spaces, for example `ControlDynamicLink`. The files are in `frappe/public/js/frappe/form/controls/`. Dialogs and web forms use the same controls. Some controls exist only for dialogs and filters: `MultiCheck`, `MultiSelect`, `MultiSelectList`, `MultiSelectPills`, `DateRange`, `Switch` and `Comment`.

## Related

- [DocTypes](../../developer-guide/data-model/doctypes.md)
- [Customize Form](../../user-guide/customization/customize-form.md)
- [DocTypes reference](doctypes.md)
- [JavaScript API](../api/js-api.md)
