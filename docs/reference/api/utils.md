---
title: Utilities
description: Commonly used helpers in frappe.utils for type conversion, dates, numbers, formatting, strings, URLs, files, HTML, passwords and caching, with signatures.
version: v15
sources:
  - frappe/utils/__init__.py
  - frappe/utils/data.py
  - frappe/utils/caching.py
  - frappe/utils/password.py
  - frappe/utils/response.py
  - frappe/utils/file_manager.py
  - frappe/utils/html_utils.py
  - frappe/utils/csvutils.py
  - frappe/utils/xlsxutils.py
  - frappe/utils/nestedset.py
  - frappe/utils/pdf.py
  - frappe/utils/image.py
  - frappe/utils/deprecations.py
---

# Utilities

`frappe.utils` holds helpers for type conversion, dates, numbers, formatting, strings, URLs and files. Most of them live in `frappe/utils/data.py`. `frappe/utils/__init__.py` re-exports them with `from frappe.utils.data import *`. Import them from `frappe.utils`:

```python
from frappe.utils import cint, flt, getdate, add_days, nowdate
```

The signatures come from `inspect.signature` on a v15 site. Type hints are shortened. `DateTimeLikeObject` means `str | datetime.date | datetime.datetime`. Helpers in other modules show the full import path.

!!! note
    In v15, `scrub`, `unscrub`, `bold`, `safe_encode`, `safe_decode`, `read_file` and `get_file_json` are on `frappe`, not on `frappe.utils`. Import them with `from frappe import scrub`. See [Python API](python-api.md).

## Example output

These results come from a v15 site with date format `dd-mm-yyyy`, number format `#,##,###.##` and rounding method `Banker's Rounding`:

```python
from frappe.utils import (
	cint, flt, cstr, getdate, add_days, date_diff, add_to_date, get_last_day, comma_and, fmt_money, format_date,
)

cint("12.7")                      # 12
cint("abc")                       # 0
cint(None, 5)                     # 5
flt("1,234.567", 2)               # 1234.57
cstr(None)                        # ''
getdate("2026-01-15")             # datetime.date(2026, 1, 15)
add_days("2026-01-31", 1)         # '2026-02-01'  (string in, string out)
add_days(getdate("2026-01-31"), 1)  # datetime.date(2026, 2, 1)
date_diff("2026-03-01", "2026-02-01")  # 28
add_to_date("2026-01-31", months=1)    # '2026-02-28'
get_last_day("2026-02-14")        # datetime.date(2026, 2, 28)
comma_and(["a", "b", "c"])        # "'a', 'b' and 'c'"
fmt_money(1234567.891, 2, "USD")  # '$ 12,34,567.89'
format_date("2026-01-15")         # '15-01-2026'
```

## Type conversion

| Function | Signature | Description |
|---|---|---|
| `cint` | `(s: int \| float \| str, default: int = 0) -> int` | Convert to integer. Returns `default` if conversion fails. |
| `flt` | `(s: int \| float \| str, precision: int \| None = None, rounding_method: str \| None = None) -> float` | Convert to float. Ignores commas in strings. Rounds to `precision` if given. |
| `cstr` | `(s, encoding='utf-8')` | Convert to string. `None` becomes `""`. |
| `sbool` | `(x: str) -> bool \| Any` | Convert `"true"`, `"false"`, `"1"`, `"0"` and similar strings to bool. |
| `cast` | `(fieldtype, value=None)` | Cast a value to the Python type of a Frappe fieldtype. |
| `parse_json` | `(val)` | Parse a JSON string into `_dict`. Returns other values as they are. |
| `parse_val` | `(v)` | Convert SQL result values to simple Python types. |
| `encode` | `(obj, encoding='utf-8')` | Encode a string to bytes. |

## Current date and time

All values are in the system time zone (System Settings).

| Function | Signature | Description |
|---|---|---|
| `nowdate` | `() -> str` | Current date as `yyyy-mm-dd`. |
| `today` | `() -> str` | Same as `nowdate`. |
| `nowtime` | `() -> str` | Current time as `hh:mm:ss`. |
| `now` | `() -> str` | Current datetime as `yyyy-mm-dd hh:mm:ss`. |
| `now_datetime` | `()` | Current datetime as a `datetime` object. |
| `get_datetime_in_timezone` | `(time_zone)` | Current datetime in the given time zone. |
| `get_system_timezone` | `() -> str` | The system time zone name. |
| `get_timestamp` | `(date)` | Unix timestamp of the date. |

## Parse dates and times

| Function | Signature | Description |
|---|---|---|
| `getdate` | `(string_date=None, parse_day_first=False) -> date \| None` | Convert to `datetime.date`. No argument returns today. |
| `get_datetime` | `(datetime_str=None) -> datetime \| None` | Convert to `datetime.datetime`. No argument returns now. |
| `get_time` | `(time_str: str) -> time` | Convert to `datetime.time`. |
| `get_timedelta` | `(time: str \| timedelta \| None = None) -> timedelta \| None` | Convert a time string to `datetime.timedelta`. |
| `to_timedelta` | `(time_str: str \| time) -> timedelta` | Convert a string or `time` to `timedelta`. |
| `get_date_str` | `(date_obj) -> str` | Return a date as `yyyy-mm-dd`. |
| `get_datetime_str` | `(datetime_obj) -> str` | Return a datetime as `yyyy-mm-dd hh:mm:ss`. |
| `get_time_str` | `(timedelta_obj) -> str` | Return a timedelta as `hh:mm:ss`. |
| `guess_date_format` | `(date_string: str) -> str` | Guess the format string of a date. |
| `is_invalid_date_string` | `(date_string: str) -> bool` | Return `True` if the string is empty or not a date. |

## Date arithmetic

Functions that take a date string return a string. Functions that take a `date` or `datetime` return the same type.

| Function | Signature | Description |
|---|---|---|
| `add_days` | `(date, days)` | Add days. |
| `add_months` | `(date, months)` | Add months. |
| `add_years` | `(date, years)` | Add years. |
| `add_to_date` | `(date=None, years=0, months=0, weeks=0, days=0, hours=0, minutes=0, seconds=0, as_string=False, as_datetime=False)` | Add any mix of units. |
| `date_diff` | `(string_ed_date, string_st_date) -> int` | Days between two dates (end minus start). |
| `days_diff` | `(string_ed_date, string_st_date) -> int` | Same as `date_diff`. |
| `month_diff` | `(string_ed_date, string_st_date)` | Months between two dates. |
| `time_diff` | `(string_ed_date, string_st_date)` | Difference as `timedelta`. |
| `time_diff_in_seconds` | `(string_ed_date, string_st_date)` | Difference in seconds. |
| `time_diff_in_hours` | `(string_ed_date, string_st_date)` | Difference in hours. |
| `get_first_day` | `(dt, d_years: int = 0, d_months: int = 0, as_str: bool = False)` | First day of the month. |
| `get_last_day` | `(dt)` | Last day of the month. |
| `get_first_day_of_week` | `(dt, as_str=False)` | First day of the week. The start day comes from System Settings, Sunday by default. |
| `get_last_day_of_week` | `(dt)` | Last day of the week. |
| `get_quarter_start` | `(dt, as_str: bool = False)` | First day of the quarter. |
| `get_quarter_ending` | `(date)` | Last day of the quarter. |
| `get_year_start` | `(dt, as_str=False)` | First day of the year. |
| `get_year_ending` | `(date) -> date` | Last day of the year. |
| `get_timespan_date_range` | `(timespan: str) -> tuple \| None` | `(start_date, end_date)` for `"last week"`, `"this month"`, `"today"` and the other timespan filter values. |
| `is_last_day_of_the_month` | `(dt)` | Return `True` if the date is the last day of its month. |
| `get_weekday` | `(datetime=None) -> str` | Weekday name, for example `"Sunday"`. |
| `get_weekdays` | `()` | All weekday names. |
| `get_month` | `(datetime=None) -> str` | Month name, for example `"January"`. |
| `convert_utc_to_system_timezone` | `(utc_timestamp)` | Convert a UTC datetime to the system time zone. |
| `convert_utc_to_timezone` | `(utc_timestamp, time_zone)` | Convert a UTC datetime to a time zone. |

## Format for display

These use the formats from System Settings and the user. Use `frappe.format_value` to format by fieldtype.

| Function | Signature | Description |
|---|---|---|
| `format_date` | `(string_date=None, format_string=None, parse_day_first=False) -> str` | Format a date in the user date format. |
| `formatdate` | same as `format_date` | Alias of `format_date`. |
| `format_time` | `(time_string=None, format_string=None) -> str` | Format a time in the user time format. |
| `format_datetime` | `(datetime_string, format_string=None) -> str` | Format a datetime in the user format. |
| `global_date_format` | `(date, format='long')` | Localized date such as `January 1, 2012`. |
| `pretty_date` | `(iso_datetime: datetime \| str) -> str` | Relative time such as `2 hours ago`. |
| `format_duration` | `(seconds: float \| int, hide_days: bool = False) -> str` | Format seconds as a duration such as `1d 2h 3m 4s`. |
| `duration_to_seconds` | `(duration)` | Convert a duration string to seconds. |
| `fmt_money` | `(amount, precision=None, currency=None, format=None) -> str` | Format a number with digit groups and an optional currency symbol. |
| `money_in_words` | `(number, main_currency=None, fraction_currency=None)` | Amount in words with currency, for example `USD One Thousand, Two Hundred And Fifty and Fifty Cent only.` |
| `in_words` | `(integer: int, in_million=True) -> str` | Integer in words. |
| `get_user_date_format` | `() -> str` | Date format of the current user. |
| `get_user_time_format` | `() -> str` | Time format of the current user. |

## Numbers

| Function | Signature | Description |
|---|---|---|
| `rounded` | `(num, precision=0, rounding_method=None)` | Round with the method from System Settings. If none is set, it uses `Banker's Rounding (legacy)`. With `Banker's Rounding`, `rounded(2.5)` is `2.0`. |
| `ceil` | `(s)` | Round up. |
| `floor` | `(s)` | Round down. |
| `safe_div` | `(numerator, denominator, precision: int = 2) -> float` | Divide. Returns `0` if `denominator` is `0`. |
| `remainder` | `(numerator, denominator, precision: int = 2)` | Remainder of a division. |
| `round_based_on_smallest_currency_fraction` | `(value, currency, precision=2)` | Round to the smallest fraction of the currency. |

## Strings and lists

| Function | Signature | Description |
|---|---|---|
| `strip` | `(val: str, chars: str \| None = None) -> str` | Strip characters from both ends. |
| `get_abbr` | `(string: str, max_len: int = 2) -> str` | Abbreviation of a string. |
| `random_string` | `(length: int) -> str` | Random alphanumeric string. |
| `generate_hash` | `(*args, **kwargs) -> str` | Random hex string. Same as `frappe.generate_hash`. |
| `sha256_hash` | `(input: str \| bytes) -> str` | SHA-256 hex digest. |
| `comma_and` | `(some_list, add_quotes=True)` | Join with commas and `and`. |
| `comma_or` | `(some_list, add_quotes=True)` | Join with commas and `or`. |
| `new_line_sep` | `(some_list)` | Join with new lines. |
| `unique` | `(seq) -> list` | Remove duplicates and keep the order. |
| `has_common` | `(l1, l2) -> bool` | Return `True` if the lists share an item. |
| `is_subset` | `(list_a: list, list_b: list) -> bool` | Return `True` if `list_a` is a subset of `list_b`. |
| `create_batch` | `(iterable, size: int)` | Yield batches of `size` items. |
| `dict_with_keys` | `(dict, keys)` | Return a new dict with only `keys`. |
| `remove_blanks` | `(d: dict) -> dict` | Remove keys with falsy values. Changes `d` in place. The docstring says `""` and `None`, but the code also removes `0` and `False`. |
| `compare` | `(val1, condition: str, val2, fieldtype=None) -> bool` | Compare two values with a filter operator such as `">"` or `"in"`. |
| `evaluate_filters` | `(doc, filters: dict \| list \| tuple)` | Return `True` if a document or dict matches the filters. |

## Email, names and users

| Function | Signature | Description |
|---|---|---|
| `validate_email_address` | `(email_str, throw=False)` | Return the valid email addresses in the string. With `throw=True`, raise on an invalid one. |
| `validate_phone_number` | `(phone_number, throw=False)` | Return `True` if the phone number is valid. |
| `validate_name` | `(name, throw=False)` | Return `True` if the name is valid. |
| `validate_url` | `(txt: str, throw: bool = False, valid_schemes=None) -> bool` | Return `True` if the text is a valid URL. |
| `split_emails` | `(txt)` | Split a string of email addresses into a list. |
| `parse_addr` | `(email_string)` | Return `(name, email)` from `Name <email>`. |
| `extract_email_id` | `(email: str) -> str` | Return only the email part. |
| `get_fullname` | `(user=None)` | Full name of a user. |
| `get_formatted_email` | `(user, mail=None)` | `Full Name <email>` of a user. |
| `get_email_address` | `(user=None)` | Email address of a user. |

## URLs and links

| Function | Signature | Description |
|---|---|---|
| `get_url` | `(uri=None, full_address=False, allow_header_override=True) -> str` | Absolute site URL, with `uri` appended. |
| `get_url_to_form` | `(doctype: str, name: str) -> str` | Absolute URL of the desk form. |
| `get_url_to_list` | `(doctype: str) -> str` | Absolute URL of the desk list. |
| `get_url_to_report` | `(name, report_type=None, doctype=None) -> str` | Absolute URL of a report. |
| `get_absolute_url` | `(doctype: str, name: str) -> str` | Desk route of the form, for example `/app/todo/<name>`. |
| `get_link_to_form` | `(doctype: str, name: str, label=None) -> str` | HTML link to the desk form. |
| `get_link_to_report` | `(name: str, label=None, report_type=None, doctype=None, filters=None) -> str` | HTML link to a report. |
| `get_filtered_list_url` | `(doctype: str, docnames=None) -> str` | List URL filtered to the given names. |
| `get_host_name` | `() -> str` | Host name of the current site. |
| `quoted` | `(url: str) -> str` | URL-quote a URL. |
| `expand_relative_urls` | `(html: str) -> str` | Make relative URLs in HTML absolute. |
| `has_unsafe_url_scheme` | `(txt: str) -> bool` | Return `True` for schemes that run code, such as `javascript:`. |

## HTML and Markdown

| Function | Signature | Description |
|---|---|---|
| `escape_html` | `(text: str) -> str` | Escape HTML special characters. |
| `strip_html` | `(text: str) -> str` | Remove everything between `<` and `>`. |
| `strip_html_tags` | `(text)` | Remove HTML tags. |
| `is_html` | `(text: str) -> bool` | Return `True` if the text has HTML tags. |
| `sanitize_html` | `(html, linkify=False, always_sanitize=False, disallowed_tags=None)` | Remove unsafe tags, attributes and styles to prevent XSS. Defined in `frappe.utils.html_utils`. |
| `frappe.utils.html_utils.clean_html` | `(html)` | Keep only basic formatting tags. |
| `md_to_html` | `(markdown_text: str)` | Convert Markdown to HTML. |
| `to_markdown` | `(html: str) -> str` | Convert HTML to Markdown. |

## Paths and site

| Function | Signature | Description |
|---|---|---|
| `get_site_path` | `(*path)` | Path inside the current site folder. |
| `get_files_path` | `(*path, **kwargs)` | Path inside `public/files`, or `private/files` with `is_private=True`. |
| `get_bench_path` | `()` | Bench root. Uses `FRAPPE_BENCH_ROOT` if set. |
| `get_sites` | `(sites_path=None)` | All site names in the bench. |
| `get_site_url` | `(site)` | URL of a site from its config. |
| `get_request_site_address` | `(full_address=False)` | Site address of the current request. |
| `is_cli` | `() -> bool` | Return `True` if the code runs from a terminal. |
| `execute_in_shell` | `(cmd, verbose=False, low_priority=False, check_exit_code=False)` | Run a shell command. Returns `(err, out)` as bytes. |

## Files

Module `frappe.utils.file_manager`. To work with files in code, prefer the `File` DocType. See [Files](../../developer-guide/data-model/files.md).

| Function | Signature | Description |
|---|---|---|
| `save_file` | `(fname, content, dt, dn, folder=None, decode=False, is_private=0, df=None)` | Make a File document attached to a document. |
| `get_file` | `(fname)` | Return `[file_name, content]` for a file name or URL. |
| `get_file_path` | `(file_name)` | Return the file path on disk. |
| `remove_all` | `(dt, dn, from_delete=False, delete_permanently=False)` | Remove all attachments of a document. |
| `remove_file` | `(fid=None, attached_to_doctype=None, attached_to_name=None, from_delete=False, delete_permanently=False)` | Remove a file and its File document. |
| `add_attachments` | `(doctype: str, name: str \| int, attachments: str \| list[str])` | Copy File documents as attachments to a document. |
| `get_max_file_size` | `()` | Maximum upload size in bytes. |

From `frappe.utils`: `get_file_size(path, format=False)`, `touch_file(path)`, `is_image(filepath)`. From `frappe`: `read_file(path, raise_not_found=False, as_base64=False)`, `get_file_json(path)`.

From `frappe.utils.image`: `optimize_image(content, content_type, max_width=1024, max_height=768, optimize=True, quality=85)`, `strip_exif_data(content, content_type)`.

## CSV, Excel and PDF

| Function | Signature | Description |
|---|---|---|
| `frappe.utils.csvutils.read_csv_content` | `(fcontent)` | Parse CSV content into a list of rows. |
| `frappe.utils.csvutils.to_csv` | `(data)` | Return CSV text for a list of rows. |
| `frappe.utils.csvutils.build_csv_response` | `(data, filename)` | Set the response to download a CSV file. |
| `frappe.utils.xlsxutils.make_xlsx` | `(data, sheet_name, wb=None, column_widths=None)` | Build an Excel workbook from a list of rows. |
| `frappe.utils.xlsxutils.build_xlsx_response` | `(data, filename)` | Set the response to download an XLSX file. |
| `frappe.utils.xlsxutils.read_xlsx_file_from_attached_file` | `(file_url=None, fcontent=None, filepath=None, *, read_only=False)` | Read rows from an XLSX file. |
| `frappe.utils.pdf.get_pdf` | `(html, options=None, output: PdfWriter \| None = None, smart_shrinking: bool = False)` | Convert HTML to PDF bytes with wkhtmltopdf. |

## Responses

Module `frappe.utils.response`. `respond_as_web_page` and `redirect_to_message` are on `frappe` in v15. See [Python API](python-api.md).

| Function | Signature | Description |
|---|---|---|
| `send_private_file` | `(path: str) -> Response` | Send a file from the private folder. |
| `download_private_file` | `(path: str) -> Response` | Check permission, then send a private file. |
| `json_handler` | `(obj)` | JSON `default` handler for dates, decimals and documents. |

To send a file download from a whitelisted method, set `frappe.response["type"] = "download"`, `frappe.response["filename"]` and `frappe.response["filecontent"]`.

## Passwords and encryption

Module `frappe.utils.password`. Passwords live in the `__Auth` table, not in the DocType table.

| Function | Signature | Description |
|---|---|---|
| `get_decrypted_password` | `(doctype, name, fieldname='password', raise_exception=True)` | Return the plain value of a Password field. |
| `set_encrypted_password` | `(doctype, name, pwd, fieldname='password')` | Store an encrypted Password field value. |
| `check_password` | `(user, pwd, doctype='User', fieldname='password', delete_tracker_cache=True)` | Check a user password. Raises `frappe.AuthenticationError` if wrong. |
| `update_password` | `(user, pwd, doctype='User', fieldname='password', logout_all_sessions=False)` | Set the login password of a user. |
| `encrypt` | `(txt, encryption_key=None)` | Encrypt text with the site `encryption_key`. |
| `decrypt` | `(txt, encryption_key=None, key: str \| None = None)` | Decrypt text from `encrypt`. |

## Caching decorators

Module `frappe.utils.caching`. See [Caching](../../developer-guide/runtime/caching.md).

| Function | Signature | Description |
|---|---|---|
| `redis_cache` | `(ttl: int \| None = 3600, user: str \| bool \| None = None, shared: bool = False)` | Cache return values in Redis. `user=True` caches per user. `shared=True` shares across sites. Adds `fn.clear_cache()`. |
| `site_cache` | `(ttl: int \| None = None, maxsize: int \| None = None)` | Cache in process memory per site. |
| `request_cache` | `(func)` | Cache for the rest of the request. Also `frappe.request_cache`. |

## Trees

Module `frappe.utils.nestedset`. See [DocTypes](../../developer-guide/data-model/doctypes.md) for tree DocTypes.

| Function | Signature | Description |
|---|---|---|
| `get_ancestors_of` | `(doctype, name, order_by='lft desc', limit=None)` | Ancestors of a node. |
| `get_descendants_of` | `(doctype, name, order_by='lft desc', limit=None, ignore_permissions=False)` | Descendants of a node. |
| `get_root_of` | `(doctype)` | Root node of the tree. |
| `rebuild_tree` | `(doctype, parent_field=None)` | Rebuild `lft` and `rgt` for the whole tree. |

## Deprecated

`cast_fieldtype(fieldtype, value, show_warning=True)` prints a deprecation message. Use `cast`.

These are marked with `@deprecated` from `frappe/utils/deprecations.py`. They warn when called: `get_gravatar`, `get_gravatar_url`, `has_gravatar`, `gzip_compress`, `gzip_decompress`. `make_esc` has a "to be deprecated" comment but does not warn.

## Related

- [Python API](python-api.md)
- [Database API](database-api.md)
- [JavaScript API](js-api.md)
