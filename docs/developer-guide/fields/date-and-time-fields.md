---
title: Date and Time Fields
description: How Date, Datetime and Time fields store values, use the System Settings formats, convert between user and system time zones, and work with frappe.datetime and frappe.utils.
version: v15
sources:
  - frappe/public/js/frappe/form/controls/date.js
  - frappe/public/js/frappe/form/controls/datetime.js
  - frappe/public/js/frappe/form/controls/time.js
  - frappe/public/js/frappe/form/formatters.js
  - frappe/public/js/frappe/utils/datetime.js
  - frappe/public/js/frappe/model/create_new.js
  - frappe/model/create_new.py
  - frappe/model/document.py
  - frappe/utils/data.py
  - frappe/utils/formatters.py
  - frappe/database/database.py
  - frappe/core/doctype/system_settings/system_settings.json
  - frappe/core/doctype/user/user.json
---

# Date and Time Fields

Use `Date` for a calendar day, `Datetime` for a point in time, and `Time` for a time of day. The database always holds the system format (`yyyy-mm-dd`, `HH:mm:ss`) in the system time zone. The desk shows and reads values in the formats from System Settings. Only `Datetime` converts between the user and system time zones. For column types, see [Field Types](../../reference/data-model/field-types.md).

## Summary

| Fieldtype | Python value after load | REST JSON | Desk control |
|---|---|---|---|
| `Date` | `datetime.date` | `"2026-11-22"` | Date picker with a "Today" button |
| `Datetime` | `datetime.datetime` | `"2026-11-22 10:30:00"` | Date picker with time sliders and a "Now" button |
| `Time` | `datetime.timedelta` on MariaDB | `"10:30:00"` | Time sliders with a "Now" button |

After `insert()` or `save()`, the object in memory keeps the value you set, for example a string. `frappe.get_doc()` and `frappe.db.get_value()` return the types above. On Postgres, `Time` is a `datetime.time`. Use `frappe.utils.get_time()` to get a `datetime.time` on any database.

## Formats and time zones

| Setting | Where | Effect |
|---|---|---|
| `date_format` | System Settings | `yyyy-mm-dd`, `dd-mm-yyyy`, `dd/mm/yyyy`, `dd.mm.yyyy`, `mm/dd/yyyy` or `mm-dd-yyyy`. Used to show and to read dates. |
| `time_format` | System Settings | `HH:mm:ss` (default) or `HH:mm`. With `HH:mm`, the picker has no seconds slider. |
| `first_day_of_the_week` | System Settings | First column of the date picker. Default `Sunday`. |
| `time_zone` | System Settings | The system time zone. All stored datetimes are in this zone. |
| `time_zone` | User | The user time zone. If empty, the system time zone applies. |

The desk gets both zones in `frappe.boot.time_zone` (`system` and `user`).

### What converts

| Fieldtype | Shown in | Stored in |
|---|---|---|
| `Date` | User format | `yyyy-mm-dd`. No time zone conversion. |
| `Datetime` | User format, user time zone | `yyyy-mm-dd HH:mm:ss`, system time zone |
| `Time` | User time format | `HH:mm:ss`. No time zone conversion. |

The `Datetime` control converts to the system time zone in `parse()` and back in `format_for_input()`. For example, with system zone `Asia/Kolkata` and user zone `America/New_York`, the stored value `2026-11-22 10:30:00` shows as `22-11-2026 00:00:00`.

The server never converts time zones. Python code, REST calls and Data Import must send `Datetime` values in the system time zone.

## Desk behavior

- **Picker**: the controls use the jQuery Air Datepicker (`$.fn.datepicker`). Arrow keys do not move in the calendar. The input shows the user format.
- **Shortcut**: press <kbd>t</kbd> in a `Date` input to set today, or in a `Datetime` input to set now. The `Time` control has no shortcut. Use its "Now" button.
- **Typed input**: type the value in the user format. A 2-digit year is accepted: `22-11-26` gives `2026-11-22` with format `dd-mm-yyyy`. Version 15 does not read offsets such as `+5d`. Text after the value is ignored.
- **Validation**: if the value is not a valid date, the desk shows "Date {0} must be in format: {1}" and clears the field. `Time` shows "Time {0} must be in format: {1}".
- **Description**: `Datetime` adds the user time zone name below the field. Set `hide_timezone: 1` on the df to hide it. `Time` adds the system time zone name only when the user zone differs.
- **List view and grid**: `frappe.format()` shows the user format. `Datetime` is converted to the user time zone.

## Datepicker options

For `Date` and `Datetime`, set `options` to a JSON object. The control merges it into the Air Datepicker options, after its own options, so your keys win. Invalid JSON logs a console warning and is ignored.

```json
{"fieldname": "delivery_date", "fieldtype": "Date", "label": "Delivery Date", "options": "{\"firstDay\": 1, \"weekends\": [5, 6]}"}
```

In a dialog or field group, you can also set these df keys:

| Key | Type | Effect |
|---|---|---|
| `min_date` | `Date` | First date the user can pick |
| `max_date` | `Date` | Last date the user can pick |
| `disabled_dates` | list of `"yyyy-mm-dd"` strings | Dates the user cannot pick |
| `hide_timezone` | `0` or `1` | Hide the time zone in the `Datetime` description |

```js
const d = new frappe.ui.Dialog({
	title: __("Pick a date"),
	fields: [
		{
			fieldname: "visit_date",
			fieldtype: "Date",
			label: __("Visit Date"),
			min_date: new Date(),
			disabled_dates: ["2026-12-25"],
		},
	],
	primary_action(values) {
		d.hide();
	},
});
d.show();
```

## Defaults

| `default` | Fieldtype | New document value |
|---|---|---|
| `Today` | `Date` | Current date. Python uses the system time zone. The desk uses the user time zone. |
| `now` or `Now` | `Datetime` | Current datetime. Python uses the system time zone. |
| Any value, or none | `Time` (Python) | Current time in the system time zone |
| `now` or `Now` | `Time` (desk) | Current time in the user time zone |

Python sets them in `frappe/model/create_new.py` (`frappe.new_doc()`, and `insert()` for fields that are empty). The desk sets them in `frappe.model.get_default_value()`. See [Field Properties](field-properties.md) for other defaults.

!!! warning
    In version 15, `frappe.new_doc()` and `insert()` put the current time in every empty `Time` field, also without a default. Set the field before you insert, or clear it after insert if it must stay empty.

The DocType save writes a default to the database column unless it is in `frappe.db.DEFAULT_SHORTCUTS` (`Today`, `__today`, `now`, `Now` and others). Other spellings, such as `today` or `NOW`, make the column change fail. `__today` is accepted by the column, but `frappe.new_doc()` inserts it as text and the insert fails. Use `Today` and `now`.

## Server

- `get_valid_dict()` stores an empty string `""` as `NULL` for all three fieldtypes.
- Frappe does not parse or check the value on save. The database does. On MariaDB, `22-11-2026` in a `Date` fails with "Incorrect date value". `2026-11-22T10:30:00` in a `Datetime` is accepted.
- A `Date` value with a time part, such as `2026-11-22 10:00:00`, is stored as `2026-11-22`.
- `Datetime` and `Time` columns keep microseconds.

## Python helpers

All functions are in `frappe.utils`. Current values use the system time zone.

| Function | Example | Result |
|---|---|---|
| `getdate(string_date=None, parse_day_first=False)` | `getdate("22-11-2026", parse_day_first=True)` | `datetime.date(2026, 11, 22)`. No argument gives today. |
| `get_datetime(datetime_str=None)` | `get_datetime("2026-11-22")` | `datetime.datetime(2026, 11, 22, 0, 0)`. No argument gives now. |
| `get_time(time_str)` | `get_time("10:30")` | `datetime.time(10, 30)` |
| `get_timedelta(time=None)` | `get_timedelta("10:30:00")` | `datetime.timedelta(seconds=37800)` |
| `nowdate()`, `today()` | | `"2026-11-22"` |
| `nowtime()` | | `"16:57:33.745161"`, with microseconds |
| `now()` | | `"2026-11-22 16:57:33.760488"`, with microseconds |
| `now_datetime()` | | `datetime.datetime` without `tzinfo` |
| `add_to_date(date, days=10)` | `add_to_date("2026-11-22", days=10)` | `"2026-12-02"`. A string in gives a string out. |
| `date_diff(end, start)` | `date_diff("2026-12-01", "2026-11-22")` | `9` |
| `format_date(value, format_string=None)` | `format_date("2026-11-22")` | `"22-11-2026"` with format `dd-mm-yyyy` |
| `format_datetime(value)` | `format_datetime("2026-11-22 10:30:00")` | `"22-11-2026 10:30:00"`. No time zone conversion. |
| `format_time(value)` | `format_time("10:30:00")` | `"10:30:00"` |
| `get_system_timezone()` | | The `time_zone` of System Settings, for example `"Asia/Kolkata"` |
| `convert_utc_to_system_timezone(dt)` | | Aware `datetime` in the system time zone |

`frappe.format_value(value, {"fieldtype": "Date"})` calls `format_date()`. Without a df, it guesses the fieldtype from the Python type. For the full list, see [Utilities](../../reference/python/utils.md).

```python
import frappe
from frappe import _
from frappe.utils import add_to_date, getdate

order = frappe.get_doc("Sample Order", "SO-00006")
if getdate(order.order_date) < getdate():
	frappe.throw(_("Order date is in the past"))

next_week = add_to_date(order.order_date, days=7)  # datetime.date in, datetime.date out
```

## JS helpers

All functions are in `frappe.datetime`. Strings use the system formats (`YYYY-MM-DD`, `HH:mm:ss`) unless the name has `user`.

| Function | Result |
|---|---|
| `now_date()`, `get_today()`, `nowdate()` | Today in the user time zone, `"2026-11-22"` |
| `now_time()` | Current time in the user time zone |
| `now_datetime()` | Current datetime in the user time zone |
| `system_datetime()` | Current datetime in the system time zone |
| `str_to_user(value, only_time, only_date)` | System string to user format. A full datetime is converted to the user time zone. |
| `user_to_str(value, only_time)` | User format to system string. No time zone conversion. |
| `str_to_obj(value)`, `obj_to_str(date)` | String to `Date` object and back |
| `convert_to_user_tz(value, format)` | System time zone to user time zone. Pass `false` as `format` to get a moment object. |
| `convert_to_system_tz(value, format)` | User time zone to system time zone |
| `is_system_time_zone()` | `true` if both zones have the same UTC offset now |
| `add_days(date, n)`, `add_months(date, n)` | `"YYYY-MM-DD"`. `add_months("2026-01-31", 1)` is `"2026-02-28"`. |
| `get_diff(a, b)` | Days from `b` to `a` |
| `month_start()`, `month_end()`, `week_start()`, `year_start()` and others | Dates of the current period |
| `validate(value)` | `true` if the value is a valid system date, datetime or time string. A value with microseconds is not valid. |
| `global_date_format(value)` | `"22nd November 2026"` |

```js
frappe.ui.form.on("Sample Order", {
	order_date(frm) {
		if (frm.doc.order_date < frappe.datetime.get_today()) {
			frappe.msgprint(__("Order date is in the past"));
		}
	},
});
```

## Pitfalls

- Do not compare a `Datetime` value with `frappe.datetime.now_datetime()`. That is user time. Use `system_datetime()`.
- Send dates as `yyyy-mm-dd` to the API. The server does not read the user format.
- `Date` default `Today` can differ between the desk and Python near midnight, when the user and system zones differ.
- A `Time` value is a `timedelta` on MariaDB. Use `get_time()` or `format_time()` before you show it.
- An empty `Time` field gets the current time on insert from Python or REST.

## Related

- [Field Types](../../reference/data-model/field-types.md)
- [Field Properties](field-properties.md)
- [Number Fields](number-fields.md)
- [Controls](controls.md)
- [Utilities](../../reference/python/utils.md)
- [System Settings](../../user-guide/settings/system-settings.md)
