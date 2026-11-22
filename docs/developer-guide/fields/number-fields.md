---
title: Number Fields
description: How Int, Float, Currency, Percent, Check, Rating and Duration fields store values, format numbers and currencies, and validate on save.
version: v15
sources:
  - frappe/public/js/frappe/form/controls/int.js
  - frappe/public/js/frappe/form/controls/float.js
  - frappe/public/js/frappe/form/controls/currency.js
  - frappe/public/js/frappe/form/controls/check.js
  - frappe/public/js/frappe/form/controls/rating.js
  - frappe/public/js/frappe/form/controls/duration.js
  - frappe/public/js/frappe/form/formatters.js
  - frappe/public/js/frappe/model/meta.js
  - frappe/public/js/frappe/utils/number_format.js
  - frappe/model/document.py
  - frappe/model/meta.py
  - frappe/utils/data.py
  - frappe/utils/formatters.py
  - frappe/database/schema.py
  - frappe/core/doctype/doctype/doctype.py
  - frappe/core/doctype/system_settings/system_settings.json
---

# Number Fields

Use these fieldtypes for counts, amounts, rates, flags, star ratings and time spans. Each one has a numeric column. `Int`, `Float`, `Currency`, `Percent` and `Check` columns are `NOT NULL` with default `0`. For column types, see [Field Types](../../reference/data-model/field-types.md). For properties that apply to all fields, see [Field Properties](field-properties.md).

## Summary

| Fieldtype | Python value after load | REST JSON | Server on save |
|---|---|---|---|
| `Int` | `int` | `12` | Cast with `cint()`. Range check. |
| `Float` | `float` | `1.5` | Cast with `flt()`. Not rounded to `precision`. |
| `Currency` | `float` | `10.5` | Same as `Float` |
| `Percent` | `float` | `12.5` | Same as `Float`. No range limit. |
| `Check` | `int` (`0` or `1`) | `1` | Any truthy value becomes `1` |
| `Rating` | `float` from `0` to `1` | `0.6` | Clamped to `0`..`1` |
| `Duration` | `float` seconds | `3725.0` | None. A non-number string fails in the database. |

The server casts values only when it writes the row. After `insert()` or `save()`, the object in memory keeps what you set, for example the string `"12"`. `load_from_db()` calls `_fix_numeric_types()`, so `frappe.get_doc()` and `doc.reload()` return the types above.

## System Settings that change numbers

| Field | Effect |
|---|---|
| `number_format` | Digit group and decimal separators, for example `#,###.##` or `#.###,##`. Default `#,###.##` in the desk. |
| `float_precision` | Decimals for `Float` and `Percent` when the field has no `precision`. Python uses `3` if it is empty. |
| `currency_precision` | Decimals for `Currency` when the field has no `precision`. If empty, the decimals of the number format apply. |
| `currency` | Default currency |
| `use_number_format_from_currency` | Use the `number_format` of the `Currency` record instead of the system format |
| `rounding_method` | `Banker's Rounding (legacy)` (default), `Banker's Rounding` or `Commercial Rounding`. Used by `flt()`, `rounded()` and the desk. |

## Number input in the desk

`Int`, `Float`, `Currency` and `Percent` share these rules:

- The input has `inputmode` `numeric` (`Int`) or `decimal` (others). Focus selects all text.
- The value is set on the change event when you leave the input, not while you type.
- You can type an expression with `+ - * / ( )`. The control calculates it: `2+3*4` gives `14`, `100*1.18` gives `118`.
- Digit group separators are removed: `1,234.5` gives `1234.5`. Text after the number is dropped: `12abc` gives `12`.

## `Int`

- **Stores**: an integer. A `length` above 11 makes a `bigint` column.
- **Desk**: no digit groups. In list view and grid, the value is right-aligned. With `options` `File Size`, the desk formatter shows `K` or `M`, for example `1.50K`.
- **Properties**: `non_negative`, `length`, `unique`.
- **Server**: `cint()` on write. `3.7` is stored as `3`. `_validate_length()` throws `frappe.CharacterLengthExceededError` when the value is above `2147483647` for `int`.
- **JS**: `frappe.ui.form.ControlInt`. `ControlLongInt` is the same class.

## `Float`

- **Stores**: a decimal with 9 places in the column by default (`decimal(21,9)`).
- **Desk**: the input shows `precision` decimals with the number format, for example `1,234.500` for precision 3. The list view hides decimals for whole numbers: `1234.0` shows as `1,234`. If `options` names a currency field, the field shows a currency symbol like `Currency`.
- **Properties**: `precision` (1 to 6, checked on DocType save), `length`, `non_negative`.
- **Server**: `flt()` on write, without `precision`. The column rounds to its own scale. Round values yourself in `validate`, for example with `self.round_floats_in(self)`.
- **JS**: `frappe.ui.form.ControlFloat`. `get_precision()` returns `df.precision` or `float_precision`. If both are empty, the control does not round.

## `Currency`

- **Stores**: a decimal, the same as `Float`. The currency is not stored in the field.
- **Desk**: the input shows the number with the currency number format. The read-only view and list view add the currency symbol, for example `₹ 1,234.50`. The `Currency` record sets `symbol`, `symbol_on_right`, `number_format` and `fraction_units`.
- **Properties**: `options`, `precision`, `non_negative`.
- **Server**: the same cast as `Float`.
- **JS**: `frappe.ui.form.ControlCurrency`.

### Currency resolution

`get_field_currency()` (Python, `frappe/model/meta.py`) and `frappe.meta.get_field_currency()` (JS, `frappe/public/js/frappe/model/meta.js`) find the currency from `options`:

| `options` | Currency used |
|---|---|
| Empty | System Settings `currency`. Python `format_value()` uses the `currency` default. |
| A fieldname, for example `currency` | The value of that field in the document. In a child row, the field of the row, then the field of the parent. |
| `DocType:link_field:currency_field`, for example `Company:company:default_currency` | Reads `currency_field` from the record in `link_field`. JS uses the user default of `link_field` if the field is empty. |

```json
{"fieldname": "rate", "fieldtype": "Currency", "label": "Rate", "options": "currency"}
```

### Precision

`get_field_precision()` in `frappe/model/meta.py` uses the first value that is set:

1. `precision` of the field.
2. For `Currency`: System Settings `currency_precision`, then the decimals of the number format for the currency.
3. For other fieldtypes: System Settings `float_precision`, then `3`.

`doc.precision("rate")` returns the same value for a document field.

## `Percent`

- **Stores**: a decimal. `150` is valid.
- **Desk**: the control is the same class as `Float`. The input shows `precision` decimals: `12.5` shows as `12.500` with `float_precision` 3. The desk formatter adds `%`.
- **Server**: the same cast as `Float`. `non_negative` is not checked.
- **Format**: Python `format_value()` always rounds to 2 decimals: `12.345` gives `12.34%` with banker's rounding.
- **JS**: `frappe.ui.form.ControlPercent` is the same class as `ControlFloat`.

## `Check`

- **Stores**: `0` or `1`.
- **Desk**: a checkbox with the label on the right. The list view and grid show a disabled checkbox.
- **Properties**: `default` must be `0` or `1`. The DocType save sets it to `0` if empty, and throws on other values.
- **Server**: `get_valid_dict()` stores `1` for any truthy value, also `True` or `5`. Check fields skip the length check.
- **JS**: `frappe.ui.form.ControlCheck`. `get_input_value()` returns `1` or `0`.

## `Rating`

- **Stores**: a fraction from `0` to `1`. With 5 stars, 3 stars is `0.6`.
- **Desk**: a row of stars. A click on the left half of a star sets a half star. A second click on the same value clears it, unless the field is mandatory. The list view shows the stars.
- **Properties**: `options` is the number of stars, from 3 to 10. The default is 5.
- **Server**: `_fix_rating_value()` sets values below `0` to `0` and above `1` to `1`. It does not change the scale, so `rating = 3` is stored as `1`.
- **Format**: Python `format_value()` returns the raw number.
- **JS**: `frappe.ui.form.ControlRating`. It extends `ControlFloat`.

```python
stars = 3
doc.rating = stars / 5  # 0.6 for a 5-star field
```

## `Duration`

- **Stores**: seconds as a decimal.
- **Desk**: the input shows the value as `1d 2h 3m 4s`. Focus opens a picker with Days, Hours, Minutes and Seconds boxes. Use the picker to set the value. The control does not parse typed text such as `1h 30m`.
- **Properties**: `hide_days` shows days as hours, for example `26h 3m 4s`. `hide_seconds` hides the seconds box and the seconds in the text.
- **Server**: no conversion. Send a number. A string such as `"1h"` fails in the database.
- **Format**: Python `format_duration(seconds, hide_days=False)` and `format_value()` use `hide_days` but not `hide_seconds`. `duration_to_seconds("1d 2h 3m 4s")` returns `93784`.
- **JS**: `frappe.ui.form.ControlDuration`. `get_value()` returns an integer. Helpers: `frappe.utils.get_formatted_duration(value, options)`, `frappe.utils.seconds_to_duration(seconds, options)`, `frappe.utils.duration_to_seconds(days, hours, minutes, seconds)`.

## Format numbers in code

| Python | JS | Result |
|---|---|---|
| `frappe.format_value(1234.5, {"fieldtype": "Currency"})` | `frappe.format(1234.5, {fieldtype: "Currency"}, {only_value: 1})` | `₹ 1,234.50` with currency INR |
| `frappe.utils.fmt_money(1234.5, currency="USD")` | `format_currency(1234.5, "USD")` | `$ 1,234.50` |
| `doc.get_formatted("rate")` | `frappe.format(doc.rate, df, {}, doc)` | Uses the field and document currency |
| `frappe.format_value(93784, {"fieldtype": "Duration"})` | `frappe.format(93784, {fieldtype: "Duration"})` | `1d 2h 3m 4s` |

`frappe.format()` wraps numbers in a right-aligned `<div>`. Pass `{inline: 1}` or `{only_value: 1}` as the third argument to get the text only. For `flt`, `cint` and `rounded`, see [Utilities](../../reference/python/utils.md).

## Pitfalls

- Values are not cast in memory after `insert()`. Use `cint()` or `flt()` before you compare them, or reload the document.
- `Float` and `Currency` values are not rounded to `precision` on save. Two values that look equal in the desk can differ in the database.
- A `Rating` value is a fraction, not a star count.
- `Percent` has no limit of 100, and `non_negative` does not apply to it.
- Version 15 has no `min_value` and `max_value` properties. Check ranges in the controller.
- A `Duration` value is in seconds. Do not send `"1h 30m"` through the API.

## Related

- [Field Types](../../reference/data-model/field-types.md)
- [Field Properties](field-properties.md)
- [Text Fields](text-fields.md)
- [Date and Time Fields](date-and-time-fields.md)
- [Utilities](../../reference/python/utils.md)
- [System Settings](../../user-guide/settings/system-settings.md)
