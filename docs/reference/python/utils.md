---
title: frappe.utils
description: Reference of every public helper in frappe.utils and its most used submodules, with signatures, defaults, return values and side effects.
version: v15
sources:
  - frappe/utils/__init__.py
  - frappe/utils/data.py
  - frappe/utils/html_utils.py
  - frappe/utils/file_manager.py
  - frappe/utils/password.py
  - frappe/utils/pdf.py
  - frappe/utils/csvutils.py
  - frappe/utils/xlsxutils.py
  - frappe/utils/response.py
  - frappe/utils/nestedset.py
  - frappe/utils/caching.py
  - frappe/utils/image.py
  - frappe/utils/print_format.py
  - frappe/utils/jinja.py
  - frappe/utils/safe_exec.py
  - frappe/utils/deprecations.py
---

# frappe.utils

`frappe.utils` holds helpers for type conversion, dates, numbers, strings, URLs, HTML, paths and files. Use this page to look up the exact signature, default, return value and side effect of a helper. Most helpers live in `frappe/utils/data.py`. `frappe/utils/__init__.py` imports all of `frappe.utils.data` with `from frappe.utils.data import *`, so import them from `frappe.utils`:

```python
from frappe.utils import add_days, cint, flt, getdate, nowdate
```

Helpers in submodules (`frappe.utils.file_manager`, `frappe.utils.password` and others) are not re-exported. Import them from the submodule.

Signatures come from the code. Many v15 helpers have no type hints. `DateTimeLikeObject` means `str | datetime.date | datetime.datetime`. `NumericType` means `int | float`. A function marked "Whitelisted" can be called over HTTP at `/api/method/<dotted.path>`. "All methods" means any HTTP method.

!!! note
    In v15, `scrub`, `unscrub`, `bold`, `as_unicode`, `safe_encode`, `safe_decode`, `get_attr`, `get_module`, `read_file`, `get_file_json`, `get_file_items`, `create_folder`, `mock`, `safe_eval` and `respond_as_web_page` are on `frappe`, not on `frappe.utils`. Import them with `from frappe import scrub`. See [frappe namespace](frappe.md).

!!! note
    The star import also re-exports names that `frappe/utils/data.py` imports, such as `relativedelta`, `parser` and `slug`. Import those from their own modules. This page does not list them.

## Example output

These results come from a v15 site. The site has date format `dd-mm-yyyy`, number format `#,##,###.##`, time zone `Asia/Calcutta` and no currency precision.

```python
from frappe.utils import (
	add_days, add_to_date, cint, comma_and, date_diff, flt, fmt_money, format_duration,
	get_last_day, getdate, rounded,
)

cint("12.7")                            # 12
cint("1,234")                           # 0
cint(None, 5)                           # 5
flt("1,234.567", 2)                     # 1234.57
flt(1.005, 2)                           # 1.0  (banker's rounding, legacy)
rounded(2.5)                            # 2.0
getdate("2026-01-15")                   # datetime.date(2026, 1, 15)
getdate("")                             # today, as datetime.date
add_days("2026-01-31", 1)               # '2026-02-01'  (string in, string out)
add_days(getdate("2026-01-31"), 1)      # datetime.date(2026, 2, 1)
add_to_date("2026-01-31", months=1)     # '2026-02-28'
date_diff("2026-03-01", "2026-02-01")   # 28
get_last_day("2024-02-03")              # datetime.date(2024, 2, 29)
fmt_money(1234567.891)                  # '12,34,567.89'
fmt_money(1234567.891, 2, "USD")        # '$ 12,34,567.89'
format_duration(93784)                  # '1d 2h 3m 4s'
comma_and(["a", "b", "c"])              # "'a', 'b' and 'c'"
```

## Type conversion and JSON

| Function | Description |
|---|---|
| `cint(s: NumericType \| str, default: int = 0) -> int` | Convert to `int`. A float string is truncated (`"12.7"` gives `12`). A string with commas gives `default`. Returns `default` when conversion fails. |
| `flt(s: NumericType \| str, precision: int \| None = None, rounding_method: str \| None = None) -> float` | Convert to `float`. Removes commas from strings. Returns `0.0` when conversion fails. Rounds with `rounded()` when `precision` is set. |
| `cstr(s, encoding="utf-8")` | Convert to `str`. `None` gives `""`. Decodes `bytes` with `encoding`. |
| `sbool(x: str) -> bool \| Any` | `"true"` and `"1"` give `True`. `"false"` and `"0"` give `False`. Case does not matter. Other values come back unchanged. |
| `cast(fieldtype, value=None)` | Cast to the Python type of a Frappe fieldtype. `Float`, `Currency`, `Percent` use `flt`. `Int`, `Check` use `cint(sbool(value))`. Text and link types use `cstr`. An empty `Date` gives `date(1, 1, 1)`. An empty `Datetime` gives `datetime(1, 1, 1)`. `Time` gives a `timedelta`. Other fieldtypes return `value`. |
| `parse_json(val)` | Parse a JSON string with the `json` module. A `dict` result becomes `frappe._dict`. Other values come back unchanged. Defined in `frappe/utils/__init__.py`. Also `frappe.parse_json`. |
| `parse_val(v)` | Convert SQL result values to simple types. `date` and `datetime` become `str`. `timedelta` becomes `"h:mm"`. |
| `safe_json_loads(*args)` | Parse each argument as JSON. Keep an argument as it is if it does not parse. One argument gives one value, more give a list. |
| `dictify(arg)` | Convert dicts, also dicts inside lists, to `frappe._dict`. Changes lists in place. |
| `encode(obj, encoding="utf-8")` | Encode a `str` to `bytes`. Other values come back unchanged. |
| `encode_dict(d, encoding="utf-8")` | Encode every `str` value of `d` to `bytes`. Changes `d` in place. |
| `decode_dict(d, encoding="utf-8")` | Does nothing. Its condition is always false. Returns `d`. |

## Current date and time

All values use the system time zone from System Settings.

| Function | Description |
|---|---|
| `now() -> str` | Current datetime, for example `"2026-11-22 17:37:03.718593"` (with microseconds). |
| `now_datetime()` | Current datetime as a naive `datetime`. |
| `nowdate() -> str` | Current date as `yyyy-mm-dd`. |
| `today() -> str` | Same as `nowdate()`. |
| `nowtime() -> str` | Current time, for example `"17:37:03.718622"` (with microseconds). |
| `get_timestamp(date)` | Unix timestamp of the start of the date in the server local time. `date` is required. |
| `get_eta(from_time, percent_complete)` | Estimated time left as a `timedelta` string, from the start time and the percent done. |

## Parse and convert dates

| Function | Description |
|---|---|
| `getdate(string_date: DateTimeLikeObject \| None = None, parse_day_first: bool = False) -> datetime.date \| None` | Convert to `date`. `None` or `""` gives today. `"0000-00-00"` and `"0001-01-01"` give `None`. A string that does not parse raises `frappe.ValidationError`. |
| `get_datetime(datetime_str: DateTimeLikeObject \| None = None) -> datetime.datetime \| None` | Convert to `datetime`. `None` gives now. A `date` gives midnight. `""` gives `None`. A `timedelta` comes back unchanged. A tuple or list raises `TypeError`. |
| `get_time(time_str: str) -> datetime.time` | Convert to `time`. Also accepts `datetime`, `time` and `timedelta`. |
| `get_timedelta(time: str \| None = None) -> datetime.timedelta \| None` | Convert a time string such as `"10:20:30"` to `timedelta`. |
| `to_timedelta(time_str: str \| datetime.time) -> datetime.timedelta` | Convert a string or `time` to `timedelta`. |
| `get_date_str(date_obj) -> str` | Date as `yyyy-mm-dd`. |
| `get_datetime_str(datetime_obj) -> str` | Datetime as `yyyy-mm-dd hh:mm:ss.ffffff`. `"2026-01-15"` gives `"2026-01-15 00:00:00.000000"`. |
| `get_time_str(timedelta_obj) -> str` | Time as `h:m:s` with no zero padding. `1:02:03` gives `"1:2:3"`. Drops days. |
| `guess_date_format(date_string: str) -> str` | `strftime` format of a date string, for example `"%d-%m-%Y"`. |
| `is_invalid_date_string(date_string: str) -> bool` | `True` if the value is not a string, is empty, or starts with `0001-01-01` or `0000-00-00`. It does not check the format. |

## Date arithmetic

A string input gives a string output. A `date` or `datetime` input gives the same type.

| Function | Description |
|---|---|
| `add_to_date(date: DateTimeLikeObject \| None = None, years=0, months=0, weeks=0, days=0, hours=0, minutes=0, seconds=0, as_string=False, as_datetime=False) -> DateTimeLikeObject` | Add any mix of units with `relativedelta`. `None` uses now. Month ends clamp: `2026-01-31` plus one month is `2026-02-28`. See the pitfalls for strings. |
| `add_days(date, days)` | Add days. |
| `add_months(date, months)` | Add months. |
| `add_years(date, years)` | Add years. `2024-02-29` plus one year is `2025-02-28`. |
| `date_diff(string_ed_date: DateTimeLikeObject, string_st_date: DateTimeLikeObject) -> int` | Days from start to end. Negative if end is before start. |
| `days_diff(string_ed_date: DateTimeLikeObject, string_st_date: DateTimeLikeObject) -> int` | Same as `date_diff`. |
| `month_diff(string_ed_date, string_st_date)` | Months between two dates, counting both partial months. `month_diff("2026-03-01", "2026-01-15")` is `3`. |
| `time_diff(string_ed_date, string_st_date)` | Difference as `timedelta`. |
| `time_diff_in_seconds(string_ed_date, string_st_date)` | Difference in seconds. |
| `time_diff_in_hours(string_ed_date, string_st_date)` | Difference in hours. |

## Week, month, quarter and year

Functions with `as_str` return a `yyyy-mm-dd` string when `as_str=True`, else a `date`. The first day of the week comes from System Settings (`first_day_of_the_week`), Sunday by default.

| Function | Description |
|---|---|
| `get_first_day(dt, d_years: int = 0, d_months: int = 0, as_str: bool = False) -> str \| datetime.date` | First day of the month of `dt`, after moving `d_years` and `d_months`. |
| `get_last_day(dt)` | Last day of the month as `date`. |
| `is_last_day_of_the_month(dt)` | `True` if `dt` is the last day of its month. |
| `get_first_day_of_week(dt, as_str=False)` | First day of the week of `dt`. |
| `get_last_day_of_week(dt)` | Last day of the week of `dt` as `date`. No `as_str`. |
| `get_quarter_start(dt, as_str: bool = False) -> str \| datetime.date` | First day of the quarter. `dt` is required. |
| `get_quarter_ending(date)` | Last day of the quarter as `date`. No `as_str`. |
| `get_year_start(dt, as_str=False)` | 1 January of the year. |
| `get_year_ending(date) -> datetime.date` | 31 December of the year. No `as_str`. |
| `get_timespan_date_range(timespan: str) -> tuple[datetime.date, datetime.date] \| None` | `(start, end)` for a timespan filter value: `"last 7 days"`, `"last 14 days"`, `"last 30 days"`, `"last 90 days"`, `"last week"`, `"last month"`, `"last quarter"`, `"last 6 months"`, `"last year"`, `"yesterday"`, `"today"`, `"tomorrow"`, `"this week"`, `"this month"`, `"this quarter"`, `"this year"`, `"next 7 days"`, `"next 14 days"`, `"next 30 days"`, `"next week"`, `"next month"`, `"next quarter"`, `"next 6 months"`, `"next year"`. |
| `get_weekday(datetime: datetime.datetime \| None = None) -> str` | English weekday name, for example `"Sunday"`. `None` uses now. |
| `get_weekdays()` | Weekday names from `"Monday"` to `"Sunday"`. |
| `get_month(datetime: DateTimeLikeObject \| None = None) -> str` | English month name, for example `"January"`. |
| `get_first_day_of_the_week() -> str` | `first_day_of_the_week` from System Settings, else `"Sunday"`. |
| `get_start_of_week_index() -> int` | Index of the first day of the week, `0` for Sunday. |
| `get_normalized_weekday_index(dt)` | Weekday index with Sunday as `0`. |
| `get_week_start_offset_days(dt)` | Days from the first day of the week to `dt`. |
| `Weekday` | `Enum` from `Sunday = 0` to `Saturday = 6`. |

## Time zones

| Function | Description |
|---|---|
| `get_system_timezone() -> str` | System time zone name from System Settings, for example `"Asia/Calcutta"`. |
| `get_datetime_in_timezone(time_zone)` | Current datetime in a time zone. The result is aware. |
| `convert_utc_to_timezone(utc_timestamp, time_zone)` | Convert a UTC datetime to an aware datetime in `time_zone`. |
| `convert_utc_to_system_timezone(utc_timestamp)` | Convert a UTC datetime to an aware datetime in the system time zone. |

## Format for display

These use the formats from System Settings. To format a value by fieldtype, use `frappe.format_value`.

| Function | Description |
|---|---|
| `format_date(string_date=None, format_string: str \| None = None, parse_day_first: bool = False) -> str` | Format a date with Babel in the date format. `format_string` uses Babel patterns, for example `"dd MMM yyyy"`. `None` gives `""`. |
| `formatdate` | Alias of `format_date`. |
| `format_time(time_string=None, format_string: str \| None = None) -> str` | Format a time in the time format. |
| `format_datetime(datetime_string: DateTimeLikeObject, format_string: str \| None = None) -> str` | Format a datetime in the date and time format. No time zone conversion. |
| `global_date_format(date, format="long")` | Localized date in the current language. `"long"` gives `"January 15, 2026"`, `"short"` gives `"1/15/26"`. |
| `pretty_date(iso_datetime: datetime.datetime \| str) -> str` | Relative time from Babel, for example `"2 hours ago"`. A string must be in `yyyy-mm-dd hh:mm:ss.ffffff` format. |
| `get_user_date_format() -> str` | `frappe.db.get_default("date_format")`, else `"yyyy-mm-dd"`. Cached in `frappe.local` for the request. |
| `get_user_format` | Alias of `get_user_date_format`. |
| `get_user_time_format() -> str` | `frappe.db.get_default("time_format")`, else `"HH:mm:ss"`. Cached in `frappe.local`. |

## Durations and timedelta

| Function | Description |
|---|---|
| `format_duration(seconds, hide_days=False)` | `93784` gives `"1d 2h 3m 4s"`. `hide_days=True` gives `"26h 3m 4s"`. |
| `duration_to_seconds(duration)` | `"1d 2h 3m 4s"` gives `93784`. |
| `validate_duration_format(duration)` | Raise `frappe.ValidationError` if the string is not in the `d h m s` format. |
| `format_timedelta(o: datetime.timedelta \| str) -> str` | Format as `h:mm:ss`. Hours can be more than 23 (`"25:01:02"`). Used for `Time` values in JSON. |
| `parse_timedelta(s: str) -> datetime.timedelta` | Parse `str(timedelta)` output such as `"1 day, 2:03:04"`. |

## Numbers and money

Rounding uses the `rounding_method` from System Settings. The default is `"Banker's Rounding (legacy)"`. The other values are `"Banker's Rounding"` and `"Commercial Rounding"`. Another value raises `frappe.InvalidRoundingMethod`.

| Function | Description |
|---|---|
| `rounded(num, precision=0, rounding_method=None)` | Round with the given or system rounding method. `rounded(2.5)` is `2.0`, `rounded(3.5)` is `4.0`. |
| `ceil(s)` | Round up. |
| `floor(s)` | Round down. |
| `safe_div(numerator: NumericType, denominator: NumericType, precision: int = 2) -> float` | Divide and round. A `0` denominator gives `0.0`. |
| `remainder(numerator: NumericType, denominator: NumericType, precision: int = 2) -> NumericType` | Remainder of the division, rounded. |
| `round_based_on_smallest_currency_fraction(value, currency, precision=2)` | Round to the `smallest_currency_fraction_value` of the Currency. |
| `fmt_money(amount: str \| float \| int \| None, precision: int \| None = None, currency: str \| None = None, format: str \| None = None) -> str` | Format with the `number_format` default or `format`. `precision` defaults to `currency_precision`, then the number format. Adds the currency symbol when `currency` is set, unless `hide_currency_symbol` is `"Yes"`. |
| `money_in_words(number: str \| float \| int, main_currency: str \| None = None, fraction_currency: str \| None = None)` | Amount in words, for example `"USD One Thousand, Two Hundred And Fifty and Fifty Cent only."`. The default currency is the `currency` default, else `INR`. A negative or invalid number gives `""`. |
| `in_words(integer: int, in_million=True) -> str` | Integer in words. `in_million=False` uses lakh and crore. |
| `get_number_format_info(format: str) -> tuple[str, str, int]` | `(decimal_separator, thousands_separator, precision)` of a number format, for example `('.', ',', 2)` for `"#,###.##"`. Unknown formats give `('.', ',', 2)`. |
| `number_format_info` | Dict of each supported number format to its `(decimal, thousands, precision)` tuple. |

## Strings

| Function | Description |
|---|---|
| `strip(val: str, chars: str \| None = None) -> str` | Remove all `﻿` and `​` characters, then strip whitespace or `chars` from both ends. `None` gives `""`. |
| `get_abbr(string: str, max_len: int = 2) -> str` | First letters of up to `max_len` words. Keeps the case. `"Example Corp"` gives `"EC"`. |
| `random_string(length: int) -> str` | Random letters and digits from `secrets`. |
| `generate_hash(*args, **kwargs) -> str` | Same as `frappe.generate_hash`. Hex string of 56 characters by default. Pass `length`. |
| `sha256_hash(input: str \| bytes) -> str` | SHA-256 hex digest. |
| `comma_and(some_list, add_quotes=True)` | `"'a', 'b' and 'c'"`. No comma before `and`. Translates each item except the last. |
| `comma_or(some_list, add_quotes=True)` | `"'a', 'b' or 'c'"`. |
| `comma_sep(some_list, pattern, add_quotes=True)` | Join all items but the last with commas, then join with the last using `pattern`, for example `"{0} or {1}"`. A value that is not a list or tuple comes back unchanged. |
| `new_line_sep(some_list)` | Join with `"\n "` (new line and a space). |
| `filter_strip_join(some_list: list[str], sep: str) -> str` | Drop empty items, strip each, then join with `sep`. |
| `list_to_str(seq, sep=", ")` | Convert each item to `str`, strip it, then join with `sep`. |
| `get_string_between(start: str, string: str, end: str) -> str` | Text between `start` and `end`. These are regex patterns. Returns `string` if there is no match. |
| `esc(s, esc_chars)` | Put a backslash before each of `esc_chars` in `s`. |
| `unesc(s, esc_chars)` | Remove the backslash before each of `esc_chars`. |
| `make_esc(esc_chars)` | Return a function that runs `esc(s, esc_chars)`. Has a "to be deprecated" comment but does not warn. |
| `sanitize_column(column_name: str) -> str` | Raise `frappe.ValidationError` if a column name or expression has unsafe SQL. Returns the name. |
| `get_table_name(table_name: str, wrap_in_backticks: bool = False) -> str` | `"ToDo"` gives `"tabToDo"`. Names that start with `__` stay the same. |

## Lists and dicts

| Function | Description |
|---|---|
| `unique(seq: Sequence[T]) -> list[T]` | Remove duplicates and keep the order. |
| `has_common(l1: Hashable, l2: Hashable) -> bool` | Common items as a `set`. The set is truthy when there is a match. |
| `is_subset(list_a: list, list_b: list) -> bool` | `True` if every item of `list_a` is in `list_b`. |
| `create_batch(iterable: Iterable, size: int) -> Generator[Iterable, None, None]` | Yield slices of at most `size` items. The input must support `len()` and slicing. |
| `dict_with_keys(dict, keys)` | New dict with only `keys`. |
| `remove_blanks(d: dict) -> dict` | Remove all falsy values, also `0`. Changes `d` in place. |
| `dict_to_str(args: dict[str, Any], sep: str = "&") -> str` | URL query string. `None` values become empty. |
| `groupby_metric(iterable: dict[str, list], key: str)` | Regroup `{category: [records]}` as `{record[key]: {category: [records]}}`. |
| `squashify(what)` | Return the only item of a one-item sequence. Other values come back unchanged. |

## Filters and comparison

These evaluate Frappe filters in Python, without a database query.

| Function | Description |
|---|---|
| `evaluate_filters(doc, filters: dict \| list \| tuple)` | `True` if `doc` matches all filters. `doc` needs no `doctype` key. Filter fields get no fieldtype, so values are compared as they are. |
| `compare(val1: Any, condition: str, val2: Any, fieldtype: str \| None = None) -> bool` | Compare with a filter operator. Casts with `cast()` when `fieldtype` is set. An unknown operator returns `False`. |
| `get_filter(doctype: str, f: dict \| list \| tuple, filters_config=None) -> frappe._dict` | Normalize the first filter to `{doctype, fieldname, operator, value, fieldtype}`. Raises `frappe.ValidationError` for an unknown operator. |
| `make_filter_tuple(doctype, key, value)` | `[doctype, key, "=", value]`, or `[doctype, key, op, val]` when `value` is `[op, val]`. |
| `make_filter_dict(filters)` | Convert `[[doctype, key, op, value], ...]` to `{key: (op, value)}`. |
| `sql_like(value: str, pattern: str) -> bool` | Python version of SQL `LIKE` with `%` at the start, the end or both. No `%` matches a substring. |
| `filter_operator_is(value: str \| None, pattern: str) -> bool` | `pattern` is `"set"` or `"not set"`. Another value raises `frappe.ValidationError`. |
| `filter_operator_timespan(value: str, pattern: str) -> bool` | `True` if the date is in `get_timespan_date_range(pattern)`. |
| `filter_operator_between(value: Any, pattern: list \| tuple) -> bool` | `True` if `pattern[0] <= value <= pattern[1]`. `None` gives `False`. |
| `convert_type_for_between_filters(value: DateTimeLikeObject, set_time: datetime.time) -> datetime.datetime` | Make a datetime from a date with `set_time`. A datetime comes back unchanged. |
| `get_safe_filters(filters)` | Parse a JSON filter string. Keeps the string if it does not parse or parses to a number. |
| `operator_map` | Dict of operator to function, used by `compare`: `^`, `in`, `not in`, `=`, `!=`, `>`, `<`, `>=`, `<=`, `not None`, `None`, `like`, `not like`, `is`, `Timespan`, `between`, `Between`. |

```python
from frappe.utils import compare, evaluate_filters

evaluate_filters({"doctype": "ToDo", "status": "Open", "priority": "High"},
	{"status": "Open", "priority": ["in", ["High", "Low"]]})          # True
compare("2026-01-02", ">", "2026-01-01", "Date")                     # True
compare(5, "between", [1, 10])                                        # True
```

## Email addresses and phone numbers

| Function | Description |
|---|---|
| `validate_email_address(email_str, throw=False)` | Return the valid addresses in the string, joined with `", "`. Splits on commas only. Invalid ones are dropped. `throw=True` raises `frappe.InvalidEmailAddressError` on an invalid one. |
| `split_emails(txt)` | Split a string of addresses on commas. Keeps quoted names with commas intact. A new line does not split (see the pitfalls). |
| `parse_addr(email_string)` | `(name, email)` from `"Name <email>"`. `name` falls back to the email. Gives `(None, email)` when no valid address is found. |
| `extract_email_id(email: str) -> str` | Only the address part. `""` for an empty value. |
| `check_format(email_id)` | Simple check: an `@`, a `.` after it, and more than four characters after `@`. |
| `get_name_from_email_string(email_string, email_id, name)` | Name part of an address string, with unsafe characters removed. |
| `sanitize_email(emails)` | Keep valid addresses and format each as `Name <email>`, joined with `", "`. |
| `validate_phone_number(phone_number, throw=False)` | `True` if the value has only digits, spaces and `+ _ - , . * # ( )`, up to 20 characters. `throw=True` raises `frappe.InvalidPhoneNumberError`. |
| `validate_phone_number_with_country_code(phone_number: str, fieldname: str) -> None` | Check with the `phonenumbers` library. Raise `frappe.InvalidPhoneNumberError` if not valid. |
| `validate_name(name, throw=False)` | `True` if a person name has only letters, digits, `'` and `-`, with single spaces. `throw=True` raises `frappe.InvalidNameError`. |

## Users

These read the `User` DocType. They do not check permissions.

| Function | Description |
|---|---|
| `get_fullname(user=None)` | First and last name. Falls back to the user ID. Defaults to the session user. Cached in `frappe.local.fullnames`. |
| `get_email_address(user=None)` | Email of the user from `frappe.db.get_value`. Defaults to the session user. |
| `get_formatted_email(user, mail=None)` | `"Full Name <email>"`. Uses the `get_sender_details` hook if an app defines it. `""` if there is no email. |
| `get_user_info_for_avatar(user_id: str) -> _UserInfo` | `{"email", "image", "name"}` of a user. A missing user gives the ID as email and name. |
| `add_user_info(user: str \| list[str] \| set[str], user_info: dict[str, _UserInfo]) -> None` | Add `fullname`, `image`, `name`, `email` and `time_zone` to `user_info` for each user ID not in it yet. |

## Validation

| Function | Description |
|---|---|
| `validate_url(txt: str, throw: bool = False, valid_schemes: str \| Container[str] \| None = None) -> bool` | `True` if the text has a scheme and a host or path, or starts with `/`. `valid_schemes` limits the scheme. `throw=True` raises `frappe.ValidationError`. |
| `has_unsafe_url_scheme(txt: str) -> bool` | `True` for `javascript:`, `vbscript:` and `data:`. |
| `validate_iban(iban: str, throw: bool = False) -> bool` | Check the IBAN checksum. `throw=True` raises `frappe.ValidationError`. |
| `is_valid_iban(iban: str) -> bool` | IBAN checksum check. Ignores spaces. |
| `validate_json_string(string: str) -> None` | Raise `frappe.ValidationError`, with no message, if the string is not valid JSON. |
| `validate_python_code(string: str, fieldname: str \| None = None, is_expression: bool = True) -> None` | Compile the code. Raise `frappe.ValidationError` with the line on a syntax error. `is_expression=False` checks a script. |
| `is_git_url(url: str) -> bool` | `True` for git, ssh, http and https repository URLs. |

## URLs and links

`get_url` reads `host_name` from site config first. Without it, it uses the request host, then the site name. It adds `http_port` or `webserver_port` from config, unless `restart_supervisor_on_update` or `restart_systemd_on_update` is set or the host has a port. The desk routes use `/app`.

| Function | Description |
|---|---|
| `get_url(uri: str \| None = None, full_address: bool = False, allow_header_override: bool = True) -> str` | Absolute site URL with `uri` joined. A full `http` or `https` URI comes back unchanged. |
| `get_request_site_address(full_address=False)` | Same as `get_url(full_address=full_address)`. |
| `get_host_name() -> str` | Site URL without the scheme, with the port if there is one, for example `"frappe.localhost:8001"`. |
| `get_host_name_from_request() -> str` | Scheme and host of the current request. Uses `X-Forwarded-Proto`. `None` outside a request. |
| `get_request_host(request: Request) -> str` | `request.host`, or the `Host` header. |
| `url_contains_port(url: str) -> bool` | `True` if the URL has a port. |
| `is_site_link(link: str) -> bool` | `True` for a path that starts with `/` or a URL on the site host. |
| `get_url_to_form(doctype: str, name: str) -> str` | Absolute URL of the desk form, for example `http://frappe.localhost:8001/app/todo/abc`. |
| `get_url_to_list(doctype: str) -> str` | Absolute URL of the desk list. |
| `get_url_to_report(name, report_type: str \| None = None, doctype: str \| None = None) -> str` | Absolute URL of a query report (`/app/query-report/<name>`). With `report_type="Report Builder"`, `/app/<doctype>/view/report/<name>`. |
| `get_url_to_report_with_filters(name, filters, report_type=None, doctype=None)` | Report URL with `filters` as the query string. |
| `get_absolute_url(doctype: str, name: str) -> str` | Desk route of the form, for example `/app/todo/abc`. |
| `get_link_to_form(doctype: str, name: str, label: str \| None = None) -> str` | HTML `<a>` to the desk form. `label` defaults to `name`. |
| `get_link_to_report(name: str, label: str \| None = None, report_type: str \| None = None, doctype: str \| None = None, filters: dict \| None = None) -> str` | HTML `<a>` to a report, with `filters` in the query string. |
| `get_filtered_list_url(doctype: str, docnames: list[str] \| None = None) -> str` | List URL filtered with `name in docnames`. |
| `get_filtered_list_link(doctype: str, docnames: list[str] \| None = None, label: str \| None = None) -> str` | HTML `<a>` to the filtered list. `label` defaults to the translated DocType. |
| `quoted(url: str) -> str` | URL-quote the URL. Keeps `/`, `:` and the other URL characters. |
| `expand_relative_urls(html: str) -> str` | Make relative `href`, `src` and CSS `url()` values absolute with `get_url()`. |
| `scrub_urls(html: str) -> str` | Same as `expand_relative_urls`. |
| `quote_urls(html: str) -> str` | URL-quote absolute `href` and `src` values in HTML. |
| `add_trackers_to_url(url: str, source: str, campaign: str, medium: str = "email") -> str` | Add `source`, `medium` and `campaign` query parameters. The names have no `utm_` prefix. `campaign` is skipped when empty. A `mailto:` URL comes back unchanged. |

## HTML and Markdown

| Function | Description |
|---|---|
| `escape_html(text: str) -> str` | Escape `& < > " '`. |
| `strip_html(text: str) -> str` | Remove everything between `<` and `>`, then strip. |
| `strip_html_tags(text)` | Remove HTML tags. |
| `is_html(text: str) -> bool` | Truthy if the text has a tag. Returns a `re.Match` or `None`, not a `bool`. |
| `sanitize_html(html, linkify=False, always_sanitize=False, disallowed_tags=None)` | Remove unsafe tags, attributes and styles with `bleach`. Escapes a removed tag instead of dropping it: `<script>b</script>` becomes `&lt;script&gt;b&lt;/script&gt;`. Allows the `cid`, `http`, `https` and `mailto` URL schemes. JSON and text without tags come back unchanged unless `always_sanitize=True`. `disallowed_tags` is a set of tags to remove, or `"*"` for all. `linkify` is not used. Defined in `frappe.utils.html_utils`. |
| `md_to_html(markdown_text: str) -> UnicodeWithAttrs \| None` | Convert Markdown to HTML with `markdown2`. Supports fenced code, tables, header IDs and a table of contents. Does not sanitize. `None` on a `MarkdownError`. |
| `markdown(text, sanitize=True, linkify=True)` | Convert Markdown to HTML, then sanitize. HTML input is only sanitized. Defined in `frappe/utils/__init__.py`. `frappe.utils.data.markdown(markdown_text)` is a different function that does not sanitize. |
| `is_markdown(text)` | `True` if the text has `<!-- markdown -->`, or has no `<p>` and no `<br>`. `<!-- html -->` gives `False`. |
| `to_markdown(html: str) -> str` | Convert HTML to Markdown with `html2text`. |
| `UnicodeWithAttrs(text)` | `str` subclass with `toc_html` and `metadata` from `markdown2`. |

Module `frappe.utils.html_utils`:

| Function | Description |
|---|---|
| `clean_html(html)` | Keep only `div p br ul ol li strong b em i u table thead tbody td tr`. Removes all attributes, `<script>` and `<style>` content, and comments. Uses `bleach`. |
| `clean_email_html(html)` | Clean HTML for email bodies. Keeps more tags, attributes and a set of CSS properties. |
| `clean_script_and_style(html)` | Remove `<script>` and `<style>` with BeautifulSoup. |
| `unescape_html(value)` | `html.unescape`. |
| `is_json(text)` | `True` if `json.loads` works. |
| `get_icon_html(icon, small=False)` | `<img>` for an image path, `<span>` for an emoji, else `<i class="...">`. |

## Images and thumbnails

| Function | Description |
|---|---|
| `is_image(filepath: str) -> bool` | `True` if the extension maps to an `image/*` type. Case does not matter. |
| `get_identicon(email: str) -> str` | Identicon PNG as a `data:image/png;base64,...` URI. |
| `get_thumbnail_base64_for_image(src)` | `{"base64", "width", "height"}` of a 50 px thumbnail for a `/files/...` image. Cached in Redis. |
| `image_to_base64(image, extn: str) -> bytes` | Base64 of a PIL image. |
| `pdf_to_base64(filename: str) -> bytes \| None` | Base64 of a `.pdf` file found with `get_file_path`. `None` for another extension or a path with `../`. |

Module `frappe.utils.image`:

| Function | Description |
|---|---|
| `optimize_image(content, content_type, max_width=1024, max_height=768, optimize=True, quality=85)` | Resize and compress image bytes. Keeps the original if the result is not smaller. SVG comes back unchanged. |
| `strip_exif_data(content, content_type)` | Apply the EXIF orientation, then remove EXIF data. |
| `resize_images(path, maxdim=700)` | Resize every `jpg`, `jpeg`, `png` and `gif` file under `path` in place. |

## Paths, sites and bench

Paths are relative to the bench `sites` folder, for example `./frappe.localhost/private`.

| Function | Description |
|---|---|
| `get_site_path(*path)` | Path inside the current site folder. |
| `get_site_base_path()` | `frappe.local.site_path`. |
| `get_path(*path, **kwargs)` | Join `path` to `base`, or to the site path. |
| `get_files_path(*path, **kwargs)` | Path inside `public/files`, or `private/files` with `is_private=1`. |
| `get_backups_path()` | `private/backups` of the site. |
| `get_bench_path()` | Bench root. Uses `FRAPPE_BENCH_ROOT` if set. |
| `get_bench_relative_path(file_path)` | Absolute path of a file relative to the bench or `sites` folder. Calls `sys.exit(1)` if the file does not exist. |
| `get_bench_id()` | `bench_id` from config, or the bench path with `/` replaced by `-`. |
| `get_site_id(site=None)` | `"<site>@<bench_id>"`. |
| `get_sites(sites_path=None)` | Sorted names of folders with a `site_config.json`. |
| `get_site_name(hostname)` | Host without the port. |
| `get_site_url(site)` | `host_name` from the site config, or `http://<site>:<webserver_port>`. |
| `get_disk_usage()` | Size of the public files folder in MB, from `du`. |
| `is_cli() -> bool` | `True` if the process runs in a terminal. |

## Files on disk

| Function | Description |
|---|---|
| `get_file_size(path, format=False)` | Size in bytes, or a string such as `"731.0B"` or `"1.2KiB"` with `format=True`. |
| `get_file_timestamp(fn)` | Modification time as a `str`, or `None` if the file does not exist. |
| `touch_file(path)` | Make the file if missing and update its time. Returns `path`. |

To read files and make folders, use `frappe.read_file`, `frappe.get_file_json`, `frappe.get_file_items` and `frappe.create_folder`.

## Hooks, modules and apps

| Function | Description |
|---|---|
| `get_hook_method(hook_name, fallback=None)` | Object of the first value of a hook, else `fallback`. |
| `call_hook_method(hook, *args, **kwargs)` | Call each function of a hook. Returns the first truthy result. Calls all functions. |
| `get_frappe_version() -> str` | `frappe.__version__`, for example `"15.127.1"`. |
| `get_app_version(app_name: str) -> str` | `<app>.__version__`, or `"0.0.1"` on error. |
| `get_installed_apps_info()` | List of `{"app_name", "version", "branch"}` for all apps. |
| `get_site_info()` | Dict with installed apps, system users, country, language, time zone, setup and scheduler state, emails sent, space usage and recent logins. |
| `get_build_version()` | Modification time of `assets/assets.json`, or a random string. |
| `get_assets_json()` | Merged `assets.json` and `assets-rtl.json`. Cached in Redis outside developer mode. |
| `get_html_format(print_path)` | Read a print format HTML file and resolve its `{% include %}` lines. |
| `is_a_property(x) -> bool` | `True` for `property` and `functools.cached_property`. |

## Database and defaults

| Function | Description |
|---|---|
| `get_defaults(key=None)` | Same as `frappe.db.get_defaults(key)`. |
| `set_default(key, val)` | Same as `frappe.db.set_default(key, val)`. Writes to `tabDefaultValue`. |
| `get_db_count(*args)` | `{doctype: count}` for each DocType. |
| `call(fn, *args, **kwargs)` | Call `frappe.call(fn, ...)` and return a JSON-safe result. Made for `bench execute`. |
| `attach_expanded_links(doctype: str, docs: list, fields_to_expand: list)` | Replace each Link value in `docs` with the full linked record. Uses `frappe.get_list` with `fields=["*"]`, so read permission applies. Changes `docs` in place. |

## Requests, shell and other

| Function | Description |
|---|---|
| `get_request_session(max_retries=5)` | `requests.Session` that retries on HTTP 500 up to `max_retries` times. |
| `set_request(**kwargs)` | Set `frappe.local.request` to a werkzeug test request built from `kwargs`. |
| `get_html_for_route(route)` | Render a website route with a fake `GET` request and return the HTML. |
| `get_test_client(use_cookies=True) -> Client` | werkzeug test client for the Frappe WSGI app. |
| `get_imaginary_pixel_response()` | Response dict for a 1x1 transparent PNG. Set it as `frappe.response` for tracking pixels. |
| `execute_in_shell(cmd, verbose=False, low_priority=False, check_exit_code=False)` | Run a shell command. A list is joined with `shlex.join`. Returns `(err, out)` as `bytes`. `check_exit_code=True` raises on a non-zero exit. |
| `update_progress_bar(txt, i, l, absolute=False)` | Print a progress bar in a terminal. Prints dots when `CI` is set. Does nothing in a web request. |
| `get_traceback(with_context=False) -> str` | Traceback of the current exception with the bench path removed. `with_context` adds local variables, only in developer mode. `""` if no exception. |
| `log(event, details)` | `frappe.logger(event).info(details)`. |
| `get_job_name(key: str, doctype: str \| None = None, doc_name: str \| None = None) -> str` | `"<key>_<doctype>_<doc_name>"`. |
| `CallbackManager()` | Queue of functions. Methods: `add(func)`, `run()` (runs and removes each in order) and `reset()`. Calling the instance with a function adds it. `frappe.db.after_commit` is one. |

## frappe.utils.file_manager

Legacy file helpers. For new code, use the `File` DocType. See [Files](../../developer-guide/data-model/files.md) and [Email, Files and Printing](email-files-and-printing.md).

| Function | Description |
|---|---|
| `save_file(fname, content, dt, dn, folder=None, decode=False, is_private=0, df=None)` | Make a `File` attached to `dt`/`dn` and return it. `decode=True` decodes base64 content. Inserts with `ignore_permissions`. Reuses the stored file when the content hash matches. Raises `MaxFileSizeReachedError` over the limit. |
| `get_file_doc(dt=None, dn=None, folder=None, is_private=None, df=None)` | Make a `File` from `frappe.form_dict` (`filedata` or `file_url`). |
| `save_uploaded(dt, dn, folder, is_private, df=None)` | Save the base64 `filedata` of the request. |
| `save_url(file_url, filename, dt, dn, folder, is_private, df=None)` | Make a `File` that points to a URL. Inserts with `ignore_permissions`. |
| `upload()` | Read the upload from `form_dict`, save it, and add an `Attachment` comment to the document. |
| `get_uploaded_content()` | `(filename, content)` from `form_dict.filedata`. |
| `add_attachments(doctype: str, name: str \| int, attachments: str \| list[str])` | Whitelisted, all methods. Copy existing `File` records as attachments. Needs write permission on the document and read permission on each File. |
| `get_file(fname)` | `[file_name, content]` of a file by File name, file name or URL. Text is decoded. |
| `get_file_path(file_name)` | Path on disk of a File by name, file name or URL. `None` for a name with `../`. |
| `get_file_url(file_data_name)` | `file_url` of a File, or its `file_name`. |
| `get_max_file_size()` | `max_file_size` from site config, else `10485760` (10 MB). |
| `check_max_file_size(content)` | Return the size. Raise `MaxFileSizeReachedError` if over the limit. |
| `write_file(content, fname, is_private=0)` | Write bytes to the files folder. Raises `frappe.ValidationError` if `fname` leaves the folder. |
| `save_file_on_filesystem(fname, content, content_type=None, is_private=0)` | Write the file and return `{"file_name", "file_url"}`. Default of the `write_file` hook. |
| `remove_all(dt, dn, from_delete=False, delete_permanently=False)` | Delete all files attached to a document. With `from_delete=True`, deletes them with `ignore_permissions` and adds no comment. |
| `remove_file(fid=None, attached_to_doctype=None, attached_to_name=None, from_delete=False, delete_permanently=False)` | Delete a File that is attached to a document and add an `Attachment Removed` comment. Does nothing for a File with no attachment or with `from_delete=True`. |
| `delete_file_data_content(doc, only_thumbnail=False)` | Call the `delete_file_data_content` hook, or `delete_file_from_filesystem`. |
| `delete_file_from_filesystem(doc, only_thumbnail=False)` | Delete the file and its thumbnail from disk. |
| `delete_file(path)` | Delete a file in `public/files` or `private/files` by URL. |
| `get_content_hash(content)` | MD5 hex digest of the content. |
| `get_file_data_from_hash(content_hash, is_private=0)` | `write_file_keys` values of a File with this hash, or `False`. |
| `get_file_name(fname, optional_suffix)` | `fname`, with `optional_suffix` before the extension if the name is taken. |
| `is_safe_path(path: str) -> bool` | `True` for `http`, `https` and `/api/method/` URLs and paths inside the site folder. |
| `safe_b64decode(binary: bytes) -> bytes` | Base64-decode with padding added. |
| `MaxFileSizeReachedError` | Subclass of `frappe.ValidationError`. |

## frappe.utils.password

Values of `Password` fields and user passwords live in the `__Auth` table. User passwords are hashed with `pbkdf2_sha256` (`argon2` is also accepted). Other `Password` fields are encrypted with Fernet and the site `encryption_key`.

| Function | Description |
|---|---|
| `get_decrypted_password(doctype, name, fieldname="password", raise_exception=True)` | Plain value of a `Password` field. Raises `frappe.ValidationError` if not found, unless `raise_exception=False` (then `None`). |
| `set_encrypted_password(doctype, name, pwd, fieldname="password")` | Encrypt and store a `Password` field value. Upserts the row. |
| `remove_encrypted_password(doctype, name, fieldname="password")` | Delete the stored value of one field. |
| `check_password(user, pwd, doctype="User", fieldname="password", delete_tracker_cache=True)` | Return the user name if the password matches. Raise `frappe.AuthenticationError` if not. Rehashes an old hash. |
| `update_password(user, pwd, doctype="User", fieldname="password", logout_all_sessions=False)` | Hash and store a login password. `logout_all_sessions=True` ends all other sessions of the user. |
| `encrypt(txt, encryption_key=None)` | Fernet-encrypt text with the site key or `encryption_key`. |
| `decrypt(txt, encryption_key=None, key: str \| None = None)` | Decrypt text from `encrypt`. Raises `frappe.ValidationError` if the key is wrong. `key` is a label for the message. |
| `get_encryption_key()` | Site `encryption_key`. Makes one and writes it to `site_config.json` if missing. |
| `delete_all_passwords_for(doctype, name)` | Delete all stored values of a document. |
| `rename_password(doctype, old_name, new_name)` | Move stored values to a new document name. |
| `rename_password_field(doctype, old_fieldname, new_fieldname)` | Move stored values to a new fieldname. |
| `delete_login_failed_cache(user)` | Reset the failed login count of the user. |
| `get_password_reset_limit()` | `password_reset_limit` from System Settings, else `3`. |
| `create_auth_table()` | Make the `__Auth` table. |

## frappe.utils.pdf

v15 renders PDF with `wkhtmltopdf` through `pdfkit`.

| Function | Description |
|---|---|
| `get_pdf(html, options=None, output: PdfWriter \| None = None, smart_shrinking: bool = False)` | Render HTML to PDF `bytes` with `wkhtmltopdf`. Adds `disable-javascript` and `disable-local-file-access`. Adds `disable-smart-shrinking` unless `smart_shrinking=True`. `options["password"]` encrypts the PDF. With `output`, adds pages to the `PdfWriter` and returns it. |
| `prepare_options(html, options)` | Add default `wkhtmltopdf` options, 15 mm side margins, the page size from Print Settings (`pdf_page_size`, default `A4`), options from `<meta name="pdfkit-...">` tags and the session cookie. Inlines private images. Returns `(html, options)`. |
| `read_options_from_html(html)` | Read `pdfkit-*` meta tags and the header and footer into options. Returns `(html, options)`. |
| `get_cookie_options()` | `cookie` option with the session ID, so `wkhtmltopdf` can load private files. |
| `inline_private_images(html) -> str` | Replace private image `src` values with base64 data. |
| `prepare_header_footer(soup: BeautifulSoup)` | Write the `#header-html` and `#footer-html` parts to temporary HTML files for `wkhtmltopdf`. |
| `cleanup(options)` | Delete the temporary header and footer files. |
| `get_file_data_from_writer(writer_obj)` | Bytes of a `PdfWriter`. |
| `is_wkhtmltopdf_valid()` | Whitelisted, all methods. `True` if the installed `wkhtmltopdf` is the patched Qt build. Cached in Redis for one hour. |
| `get_wkhtmltopdf_version()` | Version string of `wkhtmltopdf`, for example `"0.12.6"`. Cached in Redis. |
| `pdf_contains_js(file_content: bytes)` | `True` if the PDF has a `/JS` or `/JavaScript` entry. |
| `pdf_header_html(soup, head, content, styles, html_id, css)` | Render the PDF header template. Used by the print pipeline. |
| `pdf_footer_html(soup, head, content, styles, html_id, css)` | Render the PDF footer template. |
| `pdf_body_html(template, args, **kwargs)` | Render the PDF body template. |
| `get_print_format_styles(soup: BeautifulSoup) -> list[cssutils.css.Property]` | CSS properties of the `.print-format` class. |
| `toggle_visible_pdf(soup)` | Show `.visible-pdf` and remove `.hidden-pdf` elements. |
| `FrappePDFKit` | `pdfkit.PDFKit` subclass used by `get_pdf`. |
| `PDF_CONTENT_ERRORS` | `wkhtmltopdf` error names that `get_pdf` treats as broken image links. |

## frappe.utils.csvutils

| Function | Description |
|---|---|
| `read_csv_content(fcontent)` | Parse CSV `str` or `bytes` into a list of rows. Tries `utf-8`, `windows-1250` and `windows-1252`. Empty cells become `None`. Removes the formula escape quote. |
| `read_csv_content_from_attached_file(doc)` | Parse the newest file attached to `doc`. |
| `to_csv(data)` | CSV text of a list of rows. Quotes non-numeric values. Escapes cells that start with a formula character with a leading `'`. |
| `build_csv_response(data, filename)` | Set `frappe.response` to download `<filename>.csv`. |
| `send_csv_to_client(args: str \| dict[str, Any])` | Whitelisted, all methods. Same as `build_csv_response(args["data"], args["filename"])`. |
| `escape_formula_injection(value)` | Add `'` before a value that starts with a formula character. |
| `unescape_formula_injection(value)` | Remove that `'`. |
| `UnicodeWriter(encoding="utf-8", quoting=csv.QUOTE_NONNUMERIC)` | CSV writer with `writerow(row)` and `getvalue()`. Escapes formulas. |
| `get_csv_content_from_google_sheets(url)` | Download a public Google Sheet as CSV. |
| `validate_google_sheets_url(url)` | Raise `frappe.ValidationError` if not an `https://docs.google.com/spreadsheets/` URL. |
| `check_record(d)` | Check mandatory fields, Select options and dates of an import row. |
| `import_doc(d, doctype, overwrite, row_idx, submit=False, ignore_links=False)` | Insert or update one document from an import row. |
| `getlink(doctype, name)` | HTML `<a>` to `/app/Form/<doctype>/<name>`. |

## frappe.utils.xlsxutils

v15 writes XLSX files with `openpyxl`.

| Function | Description |
|---|---|
| `make_xlsx(data, sheet_name, wb=None, column_widths=None)` | Write rows to a new first sheet and return a `BytesIO`. Makes a write-only workbook when `wb` is `None`. The first row is bold. Converts HTML in cells to text, except for the `Data Import Template` and `Data Export` sheets. Dates use the System Settings format. |
| `build_xlsx_response(data, filename)` | Set `frappe.response` to download `<filename>.xlsx`. |
| `read_xlsx_file_from_attached_file(file_url=None, fcontent=None, filepath=None, *, read_only=False)` | Rows of the active sheet as lists of cell values. Reads one of `file_url`, `fcontent` or `filepath`. |
| `read_xls_file_from_attached_file(content)` | Rows of the first sheet of an old `.xls` file. |
| `get_excel_date_format()` | `(date_format, time_format)` from System Settings, with `mm` changed to `MM`. |
| `handle_html(data)` | Convert HTML in a cell to plain text. |

```python
from frappe.utils.xlsxutils import make_xlsx, read_xlsx_file_from_attached_file

content = make_xlsx([["Item", "Rate"], ["Sample Item", 10]], "Items").getvalue()
read_xlsx_file_from_attached_file(fcontent=content)  # [['Item', 'Rate'], ['Sample Item', 10]]
```

## frappe.utils.response

To send a file from a whitelisted method, set `frappe.response["type"]`, `frappe.response["filename"]` and `frappe.response["filecontent"]`. `build_response` picks the builder from `frappe.response["type"]`.

| `frappe.response["type"]` | Builder | Response |
|---|---|---|
| `"json"` | `as_json()` | JSON body. The default. |
| `"csv"` | `as_csv()` | `result` as `<doctype>.csv` attachment. |
| `"txt"` | `as_txt()` | `result` as `<doctype>.txt` attachment. |
| `"download"` | `as_raw()` | `filecontent`. MIME type from `content_type` or the file name. `display_content_as` defaults to `"attachment"`. |
| `"pdf"` | `as_pdf()` | `filecontent` as inline `application/pdf`. |
| `"binary"` | `as_binary()` | `filecontent` as `application/octet-stream` attachment. |
| `"page"` | `as_page()` | Rendered website page from `route`. |
| `"redirect"` | `redirect()` | Redirect to `frappe.response.location`. |

`respond_as_web_page` and `redirect_to_message` are on `frappe` in v15, not in this module.

| Function | Description |
|---|---|
| `download_private_file(path: str) -> Response` | Check that the user can read the File, add an Access Log, then send the file. Raises `Forbidden` for Guest. |
| `send_private_file(path: str) -> Response` | Send a file from the private folder. Uses `X-Accel-Redirect` when nginx asks. Forces download for HTML, SVG and XML. |
| `download_backup(path)` | Send a backup file. Only System Manager or Administrator. Adds an Access Log. |
| `json_handler(obj)` | JSON `default` for dates, times, `timedelta`, `Decimal`, iterables, documents (`__json__`) and others. |
| `build_response(response_type=None)` | Build the werkzeug response from `frappe.response`. |
| `report_error(status_code)` | Build a JSON error response. Adds the traceback when allowed. |
| `is_traceback_allowed()` | `True` for a system user when `allow_error_traceback` is set in System Settings, and `frappe.flags.disable_traceback` is not set (or on the dev server). |
| `make_logs()` | Add `frappe.message_log` and debug logs to `frappe.response`. |
| `handle_session_stopped()` | Return an "Updating" message page with HTTP 503. |

## frappe.utils.nestedset

Tree DocTypes use the nested set model with `lft`, `rgt` and a parent Link field. Set `nsm_parent_field` on the DocType. Without it, the parent field is `parent_<scrubbed doctype>`. See [DocTypes](../../developer-guide/data-model/doctypes.md).

| Function | Description |
|---|---|
| `NestedSet(Document)` | Base controller for tree DocTypes. On update, calls `update_nsm` and `validate_ledger`. On trash, refuses to delete a node with children. After insert, clears the user permission cache of the session user. |
| `NestedSet.get_ancestors()` | Names of all ancestors. |
| `NestedSet.get_parent() -> NestedSet` | Parent document, or `None`. |
| `NestedSet.get_children() -> Iterator[NestedSet]` | Child documents. Uses `get_list`, so read permission applies. |
| `NestedSet.validate_one_root()` | Raise `NestedSetMultipleRootsError` if there is more than one root. |
| `NestedSet.validate_ledger(group_identifier="is_group")` | Raise if a leaf node has children. |
| `get_root_of(doctype)` | Name of the root node. |
| `get_ancestors_of(doctype, name, order_by="lft desc", limit=None)` | Names of the ancestors. Uses `get_all`, so no permission check. |
| `get_descendants_of(doctype, name, order_by="lft desc", limit=None, ignore_permissions=False)` | Names of the descendants. Uses `get_list`, so read permission applies. |
| `rebuild_tree(doctype, parent_field=None)` | Whitelisted, all methods. Compute `lft` and `rgt` again for the whole tree. `parent_field` is ignored. See the pitfalls for the permission check. |
| `remove_subtree(doctype: str, name: str, throw=True)` | Delete a node and its descendants with SQL. Checks DocType delete permission. Runs no controller methods. |
| `update_nsm(doc)` | Update `lft` and `rgt` after a node is added or moved. |
| `update_add_node(doc, parent, parent_field)` | Insert a node into the tree. |
| `update_move_node(doc: Document, parent_field: str)` | Move a node to a new parent. |
| `rebuild_node(doctype, parent, left, parent_field)` | Set `lft` and `rgt` for a node and its children. |
| `validate_loop(doctype, name, lft, rgt)` | Raise `NestedSetRecursionError` if the node is its own ancestor. |
| `NestedSetRecursionError`, `NestedSetMultipleRootsError`, `NestedSetChildExistsError`, `NestedSetInvalidMergeError` | Subclasses of `frappe.ValidationError`. |

## frappe.utils.caching

For details and Redis cache methods, see [Jobs, Cache and Realtime](jobs-cache-and-realtime.md) and [Caching](../../developer-guide/runtime/caching.md).

| Function | Description |
|---|---|
| `redis_cache(ttl: int \| None = 3600, user: str \| bool \| None = None, shared: bool = False) -> Callable` | Cache return values in Redis for `ttl` seconds. `user=True` caches per session user. `shared=True` shares across sites. Use with or without parentheses. Adds `fn.clear_cache()`. |
| `site_cache(ttl: int \| None = None, maxsize: int \| None = None) -> Callable` | Cache in process memory per site. Not shared between workers. Arguments must be JSON-serializable. Adds `fn.clear_cache()`. |
| `request_cache(func: Callable) -> Callable` | Cache in `frappe.local.request_cache` until the request ends. Also `frappe.request_cache`. |

## frappe.utils.jinja

For templates and email, see [Email, Files and Printing](email-files-and-printing.md).

| Function | Description |
|---|---|
| `render_template(template, context=None, is_path=None, safe_render=True, *, restrict_globals=None)` | Render a template string or path with the sandboxed Jinja environment. With `safe_render`, a string with `.__` raises `frappe.ValidationError`. Also `frappe.render_template`. |
| `get_template(path)` | Load a template from an app path. Also `frappe.get_template`. |
| `get_jenv(*, restrict_globals=None)` | Jinja environment with Frappe globals, filters and the `jinja` hook. Cached in `frappe.local`. See the note below for `restrict_globals`. |
| `validate_template(html, restrict_globals=None)` | Raise `frappe.ValidationError` on a Jinja syntax error. |
| `get_email_from_template(name, args)` | `(html, text)` from `templates/emails/<name>.html` and `.txt`. `text` is `None` if missing. |
| `guess_is_path(template)` | `True` for one line that ends with `.html`, `.css`, `.scss`, `.py`, `.md`, `.json`, `.js`, `.xml` or `.txt`. |
| `get_jinja_hooks()` | `(methods, filters)` dicts from the `jinja` hook. |
| `get_jloader()` | Template loader for all installed apps. |
| `set_filters(jenv)` | Add the `json`, `len`, `int`, `str` and `flt` filters. |
| `safe_render_flags()` | Context manager that counts nested safe renders in `frappe.flags.in_render_safe_exec`. |

!!! note
    `restrict_globals=None` uses `frappe.utils.safe_exec.is_render_exec_enabled()`. It is `False` unless `disable_render_safe_exec` is set to `0` in `common_site_config.json`. So by default templates get the full safe-exec globals, for example `frappe.db.get_value`. `restrict_globals=True` gives only the smaller `render_safe_globals()` set.

```python
from frappe.utils.jinja import render_template

render_template("Hi {{ name }} {{ frappe.utils.flt('2.5') }}", {"name": "Customer A"})
# 'Hi Customer A 2.5'
```

## frappe.utils.print_format

Print and PDF endpoints. For print formats, see [Print Formats](../../developer-guide/output/print-formats.md).

| Function | Description |
|---|---|
| `download_pdf(doctype: str, name: str, format: str \| None = None, doc: Document \| None = None, no_letterhead: bool \| int = 0, language: str \| None = None, letterhead: str \| None = None, pdf_generator: Literal["wkhtmltopdf", "chrome"] \| None = None)` | Whitelisted, all methods, guests allowed. Checks print permission. Sends `<name>.pdf`. |
| `download_multi_pdf(doctype: str \| dict[str, list[str]], name: str \| list[str], format: str \| None = None, no_letterhead: bool = False, letterhead: str \| None = None, options: str \| None = None)` | Whitelisted, all methods. Sends one PDF for many documents. `name` is a JSON list. Or pass `doctype` as `{doctype: [names]}`. A document that fails, for example with no permission, is left out. |
| `download_multi_pdf_async(...)` | Whitelisted, all methods. Same arguments. Builds the PDF in a background job (`long` queue for more than 20 documents, else `short`) and returns `{"task_id": ...}` with HTTP 201. Sends progress with `frappe.publish_progress`. |
| `report_to_pdf(html: str, orientation: str = "Landscape")` | Whitelisted, all methods. Render report HTML to `report.pdf`. Adds an Access Log. |
| `render_letterhead_for_print(letterhead: str \| None = None, doc: dict \| str \| None = None) -> dict` | Whitelisted, all methods. `{"header", "footer"}` HTML of a Letter Head, or the default one. `{}` without read permission on Letter Head. |
| `print_by_server(doctype: str, name: str \| int, printer_setting: str, print_format: str \| None = None, doc: Document \| None = None, no_letterhead: bool \| int = 0, file_path: str \| None = None)` | Whitelisted, all methods. Send the PDF to a CUPS printer from Network Printer Settings. Needs `pycups`. `file_path` is ignored. |
| `standard_format` | `"templates/print_formats/standard.html"`. |

## Deprecated

These warn when called. The `deprecated` decorator is in `frappe/utils/deprecations.py`.

| Name | Use instead |
|---|---|
| `cast_fieldtype(fieldtype, value, show_warning=True)` | `cast()`. Prints a warning. |
| `get_gravatar(email)` | `get_identicon()`. Now returns an identicon. |
| `get_gravatar_url(email, default="mm")` | `get_identicon()`. Returns `""`. |
| `has_gravatar(email)` | Returns `""`. |
| `gzip_compress(data, compresslevel=9)`, `gzip_decompress(data)` | Python `gzip` module. |
| `frappe.utils.print_format.read_multi_pdf(output)` | `PdfWriter.write()`. |

## Pitfalls

- `add_to_date` with a date string and only `minutes` or `seconds` drops the time. `add_to_date("2026-01-31", minutes=30)` gives `"2026-01-31"`. Pass a datetime string, or use `hours`.
- `add_to_date` with a `date` object and a time unit returns a `datetime`.
- `getdate("")` and `getdate(None)` return today, not `None`. Check for empty values before you call it.
- `cint("1,234")` gives `0`. Use `cint(flt("1,234"))`.
- `flt(1.005, 2)` gives `1.0` with the default rounding method. Pass `rounding_method="Commercial Rounding"` to round half away from zero.
- `compare` needs the exact operator name. `"Timespan"` works, `"timespan"` returns `False`.
- `split_emails` and `validate_email_address` do not split on new lines. `validate_email_address("a@example.com\nb@example.com")` gives only `"a@example.com"`.
- `frappe.utils.markdown` sanitizes. `frappe.utils.data.markdown` does not.
- `sanitize_html` escapes `<script>` and other removed tags. The tag text stays in the output as escaped text.
- `rebuild_tree` checks for System Manager only when `frappe.form_dict.cmd` is `"rebuild_tree"`. A call to `/api/method/frappe.utils.nestedset.rebuild_tree` runs for any logged-in user.
- `get_ancestors_of` ignores permissions. `get_descendants_of` and `NestedSet.get_children` apply them.
- `NestedSet.get_parent`, `get_children` and `validate_one_root` read `self.nsm_parent_field`. That attribute is set only when the DocType sets `nsm_parent_field`.

## Related

- [frappe namespace](frappe.md)
- [Database](database.md)
- [Permissions and Sessions](permissions-and-sessions.md)
- [Jobs, Cache and Realtime](jobs-cache-and-realtime.md)
- [Email, Files and Printing](email-files-and-printing.md)
- [Date and Time Fields](../../developer-guide/fields/date-and-time-fields.md)
- [Files](../../developer-guide/data-model/files.md)
- [Whitelisted Methods](../../developer-guide/api/whitelisted-methods.md)
