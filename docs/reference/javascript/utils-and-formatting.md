---
title: Utils and Formatting (JavaScript)
description: Reference of client-side helpers in the v15 desk - frappe.utils, frappe.datetime, frappe.format, number and currency formatting, flt, cint, cstr, frappe.dom, frappe.urllib and other globals.
version: v15
sources:
  - frappe/public/js/frappe/utils/utils.js
  - frappe/public/js/frappe/utils/common.js
  - frappe/public/js/frappe/utils/datatype.js
  - frappe/public/js/frappe/utils/number_format.js
  - frappe/public/js/frappe/utils/number_systems.js
  - frappe/public/js/frappe/utils/datetime.js
  - frappe/public/js/frappe/utils/pretty_date.js
  - frappe/public/js/frappe/utils/urllib.js
  - frappe/public/js/frappe/utils/tools.js
  - frappe/public/js/frappe/utils/datatable.js
  - frappe/public/js/frappe/query_string.js
  - frappe/public/js/frappe/event_emitter.js
  - frappe/public/js/frappe/meta_tag.js
  - frappe/public/js/frappe/format.js
  - frappe/public/js/frappe/form/formatters.js
  - frappe/public/js/frappe/model/meta.js
  - frappe/public/js/frappe/dom.js
---

# Utils and Formatting (JavaScript)

This page lists the client-side helpers that the v15 desk loads: type conversion, number and currency formatting, field formatting, dates, `frappe.utils`, `frappe.urllib`, `frappe.dom` and some globals on `window`. The names come from `Object.keys()` on a logged-in desk page on v15. Each example ran on the v15 site. Results that depend on site settings name the setting. The example site uses date format `dd-mm-yyyy`, number format `#,##,###.##`, currency `INR`, float precision 3 and "Banker's Rounding".

For the `frappe` namespace (calls, messages, routing, boot), see [frappe (JavaScript)](frappe.md). For field types, see [Number Fields](../../developer-guide/fields/number-fields.md) and [Date and Time Fields](../../developer-guide/fields/date-and-time-fields.md).

## Type conversion

These are globals on `window`.

| Function | Description |
|---|---|
| `flt(v, decimals, number_format, rounding_method)` | Convert to a float. Strips a currency symbol before a space and the group separators of `number_format` (default: system format). Returns `0` for `null`, `""` or a value that does not parse. Rounds with `_round` when `decimals` is given. |
| `cint(v, def)` | Convert to an integer with `parseInt`. `true` is `1`, `false` is `0`. Returns `def` (default `0`) if the value does not parse. |
| `cstr(s)` | Convert to a string. `null` and `undefined` become `""`. |
| `_round(num, precision, rounding_method)` | Round a number. `rounding_method` defaults to the `rounding_method` system default, else `"Banker's Rounding (legacy)"`. Other values: `"Banker's Rounding"`, `"Commercial Rounding"`. Throws for an unknown method. |
| `roundNumber(num, precision)` | `_round(num, precision)`. Kept for old code. |
| `precision(fieldname, doc)` | Precision of a field of `doc` (default `cur_frm.doc`). Without an open form, returns `frappe.boot.sysdefaults.float_precision`, which is a string. |
| `remainder(numerator, denominator, precision)` | Remainder, rounded to `precision`. |
| `round_based_on_smallest_currency_fraction(value, currency, precision)` | Round to the `smallest_currency_fraction_value` of the Currency. Without one, `_round(value)`. |
| `is_null(v)` | `true` for `null`, `undefined` or a blank string. Returns `undefined` (not `false`) otherwise. |
| `in_list(list, item)` | `list.includes(item)`. |
| `copy_dict(d)` | Shallow copy of an object. |

```js
flt("1,234.567", 2);              // 1234.57
flt("1.234,56", 2, "#.###,##");   // 1234.56
flt("₹ 1,200.50");                // 1200.5
flt(2.5, 0);                      // 2 (Banker's Rounding)
flt(2.5, 0, null, "Commercial Rounding"); // 3
cint("12.9");                     // 12
cint("abc", 5);                   // 5
cstr(null);                       // ""
```

## Number and currency formatting

Globals on `window`.

| Function | Description |
|---|---|
| `format_number(v, format, decimals)` | Format a number with the group and decimal separators of `format`. Without `format`, uses the system number format and, if `decimals` is not given, the `float_precision` default (else 3). Without `decimals`, uses the decimals of `format`. |
| `format_currency(v, currency, decimals)` | `format_number` with the currency symbol before the value, or after it when the Currency has `symbol_on_right`. `decimals` defaults to the `currency_precision` system default. No symbol when `hide_currency_symbol` is set. |
| `get_currency_symbol(currency)` | Symbol from the Currency in `locals`, else the currency code. `currency` defaults to the system currency. Returns `null` when `hide_currency_symbol` is `"1"` or `"Yes"`. |
| `get_number_format(currency)` | Number format of the Currency when `use_number_format_from_currency` is enabled. Else the system `number_format`, else `"#,###.##"`. |
| `get_number_format_info(format)` | `{ decimal_str, group_sep, precision }` of a format. Unknown formats get `.` and `,`. |
| `strip_number_groups(v, number_format)` | Remove group separators and change the decimal separator to `.`. Returns a string. |
| `convert_old_to_new_number_format(v, old_number_format, new_number_format)` | Change the separators of a formatted string from one format to another. |
| `fmt_money(v, format)` | Deprecated. Calls `format_currency(v, format)`. |

`frappe.number_format_info` lists the supported formats: `#,###.##`, `#.###,##`, `# ###.##`, `# ###,##`, `#'###.##`, `#, ###.##`, `#,##,###.##`, `#,###.###`, `#.###`, `#,###`.

```js
format_number(1234567.891, "#,###.##");     // "1,234,567.89"
format_number(1234567.891, "#.###,##", 1);  // "1.234.567,9"
format_number(1234567.891);                 // "12,34,567.891" (format #,##,###.##, precision 3)
format_currency(1234.5, "USD");             // "$ 1,234.50"
format_currency(1234.5, "INR", 0);          // "₹ 1,234"
get_number_format_info("#.###,##");         // { decimal_str: ",", group_sep: ".", precision: 2 }
```

## Field formatting

### frappe.format

```js
frappe.format(value, df, options, doc)
```

Format a value for display by its field definition `df`. Returns HTML (or a plain value for some options). Without `df`, the value is formatted as Data. A `Dynamic Link` is formatted as a Link to `doc[df.options]`. `df.formatter`, if set, replaces the standard formatter. The result has `<script>`, `<style>` and similar tags removed (`frappe.dom.remove_script_and_style`). Other HTML in a Data value is not escaped.

For a Currency field, `df.options` is the name of the currency field in `doc`, or `DocType:field:currency_field`. It is not a currency code. Without a match, the system currency is used.

| Option | Description |
|---|---|
| `inline` | Return numbers without the right-aligned `<div>` wrapper. |
| `only_value` | Like `inline`. For Link, return the title or value as text, not an `<a>` tag. |
| `for_print` | For Link, return text, not an `<a>` tag. |
| `always_show_decimals` | For Float, keep decimals on whole numbers. By default `2.000` shows as `2`. |
| `label` | For Link, the link text. |

Formatters exist for: `Data`, `Autocomplete`, `Select`, `Float`, `Int`, `Percent`, `Rating`, `Currency`, `Check`, `Link`, `Date`, `DateRange`, `Datetime`, `Text`, `Time`, `Duration`, `LikedBy`, `Tag`, `Comment`, `Assign`, `SmallText`, `TextEditor`, `Code`, `WorkflowState`, `Email`, `FileSize`, `TableMultiSelect`, `Color`, `Icon`, `Attach`, `AttachImage`. Other field types use `Data`.

```js
frappe.format(1234.5, { fieldtype: "Currency" }, { inline: 1 }); // "₹ 1,234.50" (system currency INR)
frappe.format(2, { fieldtype: "Float" });            // "<div style='text-align: right'>2</div>"
frappe.format("2026-01-15", { fieldtype: "Date" });  // "15-01-2026" (date format dd-mm-yyyy)
frappe.format(3661, { fieldtype: "Duration" });      // "1h 1m 1s"
frappe.format(1536, { fieldtype: "Int", options: "File Size" }); // "1.50K"
frappe.format("ToDo", { fieldtype: "Link", options: "DocType" });
// '<a href="/app/doctype/ToDo" data-doctype="DocType" data-name="ToDo" data-value="ToDo">ToDo</a>'
```

### Related helpers

| Name | Description |
|---|---|
| `frappe.form.get_formatter(fieldtype)` | Formatter function of a field type. Spaces are removed (`"Small Text"` gives `SmallText`). Unknown types get `Data`. |
| `frappe.form.formatters` | Map of field type to `formatter(value, df, options, doc)`. Assign a key to add or replace a formatter. |
| `frappe.form.link_formatters[doctype]` | `fn(value, doc, docfield)` that changes the display value of Links to `doctype`. Frappe sets one for `User` that shows `doc.full_name` or `<fieldname>_full_name` from `doc`. |
| `frappe.get_format_helper(doc)` | Copy of `doc` with `get_formatted(fieldname)`, which calls `frappe.format` with `{ inline: 1 }`. |
| `frappe.meta.set_formatter(doctype, fieldname, name, formatter)` | Set `df.formatter` on one field. See [frappe (JavaScript)](frappe.md#metadata). |
| `$.format(str, args)` | Replace `{0}`, `{1}` (or `{}` in order) with items of `args`. Used by `__()`. |

## Dates and times

`frappe.datetime` uses moment.js. The system string formats are `frappe.defaultDateFormat` (`YYYY-MM-DD`), `frappe.defaultTimeFormat` (`HH:mm:ss`) and `frappe.defaultDatetimeFormat`. `moment.defaultFormat` is set to `YYYY-MM-DD`, so functions that call `moment().format()` return a date only.

| Function | Description |
|---|---|
| `get_today()` / `nowdate()` / `now_date(as_obj = false)` | Today in the user time zone, as `yyyy-mm-dd`, or a `Date` with `as_obj`. |
| `now_time(as_obj = false)` | Current time in the user time zone, `HH:mm:ss`. |
| `now_datetime(as_obj = false)` | Current date and time in the user time zone. |
| `system_datetime(as_obj = false)` | Current date and time in the system time zone. |
| `add_days(d, days)` | Add days. Returns `yyyy-mm-dd`. The time part is dropped. |
| `add_months(d, months)` | Add months. Returns `yyyy-mm-dd`. |
| `get_diff(d1, d2)` / `get_day_diff(d1, d2)` | Whole days of `d1` minus `d2`. |
| `get_hour_diff(d1, d2)` / `get_minute_diff(d1, d2)` | Whole hours or minutes of `d1` minus `d2`. |
| `week_start()` / `week_end()` | First or last day of this week. The week starts on Sunday (moment locale `en`). |
| `month_start()` / `month_end()` | First or last day of this month. |
| `quarter_start()` / `quarter_end()` | First or last day of this quarter. |
| `year_start()` / `year_end()` | First or last day of this year. |
| `str_to_obj(d)` | System string to `Date`. |
| `obj_to_str(d)` | `Date` to `yyyy-mm-dd`. |
| `obj_to_user(d)` | `Date` to a date string in the user format. |
| `str_to_user(val, only_time = false, only_date = false)` | System string to user format. A datetime is also converted from the system to the user time zone. |
| `user_to_str(val, only_time = false)` | User format to system format. Also accepts a 2-digit year. |
| `user_to_obj(d)` | User format to `Date`. |
| `get_datetime_as_string(d)` | `Date` to `yyyy-mm-dd` plus the system time format. |
| `global_date_format(d)` | Long date, for example `15th January 2026`. Adds the time for a datetime. |
| `get_time(timestamp)` | Time as `hh:mm A`. |
| `convert_to_user_tz(date, format)` | System time zone to user time zone. Returns a string, or a moment object with `format === false`. |
| `convert_to_system_tz(date, format)` | User time zone to system time zone. Same return as above. |
| `is_system_time_zone()` / `is_timezone_same()` | `true` if the user and system time zones have the same offset now. |
| `get_user_date_fmt()` / `get_user_fmt()` | System `date_format`, for example `dd-mm-yyyy`. Default `yyyy-mm-dd`. |
| `get_user_time_fmt()` | System `time_format`. Default `HH:mm:ss`. |
| `validate(d)` | `true` if `d` is a valid date, datetime or time in system format (strict parse). |
| `get_first_day_of_the_week_index()` | Index (0 is Sunday) of the `first_day_of_the_week` system default. |
| `prettyDate(date, mini)` | Relative time, for example `2 hours ago` or `yesterday`. With `mini`, short form such as `2 h`. Returns `""` for a future date. |
| `comment_when(datetime, mini)` | `<span class="frappe-timestamp">` with the relative time and the full date as `title`. Also `window.comment_when`. |
| `refresh_when()` | Update the text of all `.frappe-timestamp` elements. Runs every 60 seconds. |
| `moment_to_date_obj(moment_obj)` | moment object to `Date` with the same wall-clock values. |

```js
frappe.datetime.add_days("2026-01-31", 1);                  // "2026-02-01"
frappe.datetime.add_months("2026-01-31", 1);                // "2026-02-28"
frappe.datetime.get_day_diff("2026-03-01", "2026-02-01");   // 28
frappe.datetime.str_to_user("2026-01-15 10:30:00");         // "15-01-2026 10:30:00"
frappe.datetime.user_to_str("15-01-2026");                  // "2026-01-15"
frappe.datetime.global_date_format("2026-01-15");           // "15th January 2026"
```

## frappe.utils

### Strings and HTML

| Function | Description |
|---|---|
| `escape_html(txt)` | Escape `&`, `<`, `>`, `"`, `'`, the backtick and `=`. `null` and `undefined` give `""`. |
| `unescape_html(txt)` | Reverse of `escape_html`. |
| `html2text(html)` | Text content of HTML. |
| `is_html(txt)` | `true` if `txt` has an element node. |
| `xss_sanitise(string, options)` | Remove `<script>` blocks and `alert()`, `confirm()`, `prompt()` calls (strategy `"js"`). Then escape `< > " ' /` (strategy `"html"`). Default `{ strategies: ["html", "js"] }`. Does not remove `on*` attributes. |
| `has_unsafe_url_scheme(value)` | `true` for a `javascript:`, `vbscript:` or `data:` URL. Tabs, newlines and leading control characters are removed before the check. |
| `strip_whitespace(html)` | Remove empty `<p>` tags and collapse repeated `<br>`. |
| `encode_tags(html)` | Escape `& < >`. |
| `replace_newlines(t)` | Replace `\n` with `<br>`. |
| `strip_original_content(txt)` | Remove trailing quoted lines (starting with `>`) of an email body. |
| `toggle_blockquote(txt)` | Hide nested blockquotes behind a toggle link. |
| `to_title_case(string, with_space = false)` | Capitalize words. Remove `-` and `_`, or replace them with a space when `with_space`. |
| `split_emails(txt)` | Split on commas and newlines, but not inside quotes. Returns a list. |
| `string_to_boolean(string)` | `true` for `t`, `true`, `y`, `yes`, `1`. `false` for `f`, `false`, `n`, `no`, `0`. Else the input. |
| `is_json(str)` | `true` if `JSON.parse` succeeds. |
| `parse_json(str)` | Parsed value, or the input if it does not parse. |
| `is_url(txt)` | `true` if `txt` starts with `http://` or `https://`. |
| `get_random(len)` | Random string of letters. |
| `get_decoded_string(dataURI)` | Decode the base64 part of a data URI. |
| `validate_type(val, type)` | Test `val` against a pattern. `type`: `phone`, `name`, `number`, `digits`, `alphanum`, `email`, `url`, `dateIso`. Returns `false` for an empty string or an unknown type. |

```js
frappe.utils.escape_html('<a href="x">');          // "&lt;a href&#x3D;&quot;x&quot;&gt;"
frappe.utils.to_title_case("sample_item-name", true); // "Sample item Name"
frappe.utils.split_emails('a@x.com, "B, C" <b@x.com>'); // ["a@x.com", "\"B, C\" <b@x.com>"]
```

### Lists and objects

| Function | Description |
|---|---|
| `unique(list)` | Remove duplicates. Keys are compared as strings. |
| `sum(list)` | Sum with `flt`. |
| `all(lst)` | `true` if every item is truthy. |
| `remove_nulls(list)` | Remove items for which `is_null` is true (also blank strings). |
| `dict(keys, values)` | List of objects from a key list and a list of rows. |
| `arrays_equal(arr1, arr2)` | Compare two arrays item by item. Nested arrays are compared too. |
| `deep_equal(a, b)` | Deep compare (`fast-deep-equal`). |
| `deep_clone(obj, customizer)` | Deep copy with lodash `cloneDeepWith`. |
| `intersection(a, b)` | Sorted items in both lists. |
| `range(start, end)` | `[start, ..., end - 1]`. With one argument, `[0, ..., start - 1]`. |
| `parse_array(array)` | The array, or `undefined` if empty. |
| `sort(list, key, compare_type, reverse)` | Sort a list of objects by `key` in place. `compare_type` is `"string"` or `"number"` (guessed from the first item). |
| `filter_dict(dict, filters)` | Items of a list or object that match `filters`, for example `{ status: "Open" }` or `{ a: ["in", [1, 2]] }`. A string filter returns `[dict[filters]]`. |
| `comma_and(list)` / `comma_or(list)` | `"a, b and c"` or `"a or b"`. Translated. |
| `comma_sep(list, sep)` | Join with `, ` and `sep` before the last item. |
| `mask_passwords(obj)` | Replace values of keys that contain `password` or `passphrase` with `*****`, in place. Returns nothing. |

```js
frappe.utils.unique([1, 2, 2, "a", "a"]);       // [1, 2, "a"]
frappe.utils.dict(["a", "b"], [[1, 2], [3, 4]]); // [{a: 1, b: 2}, {a: 3, b: 4}]
frappe.utils.comma_and(["a", "b", "c"]);        // "a, b and c"
```

### URLs, links and routes

| Function | Description |
|---|---|
| `get_form_link(doctype, name, html = false, display_text = null, query_params_obj = null)` | Desk URL of a form, for example `/app/sample-item/Item%20A`. With `html`, an `<a>` tag. `query_params_obj` adds a query string. |
| `generate_route(item)` | Desk URL of a workspace link item (`type`: `doctype`, `report`, `page`, `dashboard`, plus `doc_view`, `route_options`, `is_query_report` and others). |
| `get_route_label(route_str)` | Display label of a route string, for example `ToDo List`. |
| `get_url_arg(name)` | Value of a query parameter of the current URL, or `""`. |
| `get_query_string(url)` | Part of `url` after `?`. |
| `get_query_params(query_string)` | Object of query parameters. Default: the current URL. Repeated keys give a list. `+` becomes a space. |
| `make_query_string(obj, encode = true)` | `?key=value&...`. Skips `undefined`, `null` and `""`. Objects become JSON. |
| `get_url_from_dict(args)` | `key=value&...` without `?`. Skips `null`. |
| `get_args_dict_from_url(txt)` | Object from `a=1&b=2`. |
| `sanitise_redirect(url)` | `url` if it is on the same origin, else `""`. Keeps the relative or absolute form. |
| `strip_url(url)` | Remove characters before the protocol, `//` or `#`. |
| `get_file_link(filename)` | `files/<filename>` for a bare name. URLs and paths are returned as they are. |
| `get_link_title(doctype, name)` | Cached title of a Link value, or `undefined`. |
| `add_link_title(doctype, name, value)` | Add a title to the cache. |
| `fetch_link_title(doctype, name)` | Get the title from `frappe.desk.search.get_link_title` and cache it. Resolves with the title, or `name` on error. |
| `get_page_view_count(route)` | Call `frappe.website.doctype.web_page_view.web_page_view.get_page_view_count`. Returns the call. |

```js
frappe.utils.get_form_link("Sample Item", "Item A", true, "Open");
// '<a href="/app/sample-item/Item%20A">Open</a>'
frappe.utils.make_query_string({ a: 1, b: "x y", c: null }); // "?a=1&b=x%20y"
frappe.utils.get_query_params("a=1&b=x+y&a=2");             // { a: ["1", "2"], b: "x y" }
```

### Numbers and durations

| Function | Description |
|---|---|
| `shorten_number(number, country, min_length = 4, max_no_of_decimals = 2)` | Short form with the number system of `country`: `K`, `M`, `B`, `T` by default. `K`, `L`, `Cr` for Bangladesh, India, Myanmar and Pakistan. Nepal adds `Ar` and `Kh`. Numbers with fewer than `min_length` digits are returned as strings without change. |
| `get_number_system(country)` | List of `{ divisor, symbol }` for a country. |
| `get_number_of_decimals(number)` | Count of decimal digits. |
| `format_chart_axis_number(label, country)` | `shorten_number(label, country, 3)`. `country` defaults to the system country. |
| `seconds_to_duration(seconds, duration_options)` | `{ days, hours, minutes, seconds }`. `hide_days` moves days into hours. `hide_seconds` rounds seconds into minutes. |
| `duration_to_seconds(days = 0, hours = 0, minutes = 0, seconds = 0)` | Total seconds. |
| `get_formatted_duration(value, duration_options = null)` | Text such as `1d 1h 1m 1s`. |
| `get_duration_options(docfield)` | `{ hide_days, hide_seconds }` of a Duration field. |
| `get_formatted_iban(value)` | IBAN in groups of 4 characters. Not changed for BI, SV, EG and LY. |
| `only_allow_num_decimal(input)` | Remove characters other than digits, `.` and `-` from a jQuery input as the user types. |
| `report_column_total(values, column, type)` | Total of a report column. Mean for Percent or `type === "mean"`. `null` for non-number columns. |

```js
frappe.utils.shorten_number(1234567, "India");         // "12.35 L"
frappe.utils.shorten_number(1234567, "United States"); // "1.23 M"
frappe.utils.get_formatted_duration(90061);            // "1d 1h 1m 1s"
```

### Filters

| Function | Description |
|---|---|
| `get_filter_as_json(filters)` | Filter list `[[doctype, field, op, value], ...]` to a JSON string `{ field: [op, value] }`. `null` for an empty list. |
| `get_filter_from_json(filter_json, doctype)` | Parse a filter string saved by a dashboard chart or card. With `doctype`, returns a filter list. Without it, an object keyed by field. |
| `process_filter_expression(filter)` | Evaluate a filter string as JavaScript with `new Function`. |

### Code evaluation

| Function | Description |
|---|---|
| `eval(code, context = {})` | Evaluate an expression. The keys of `context` are variables. Throws on error. It does not remove an `eval:` prefix. |
| `make_event_emitter(object)` | Add `on`, `once`, `off`, `trigger` (jQuery events) to an object. `off` does not work, see Pitfalls. |
| `debug.watch_property(obj, prop, callback = console.trace)` | Call `callback` each time `obj[prop]` is set. For debugging only. |

```js
frappe.utils.eval("doc.qty > 2", { doc: { qty: 3 } }); // true
```

### Timing

| Function | Description |
|---|---|
| `sleep(time)` | Promise that resolves after `time` ms. |
| `debounce(func, wait, immediate)` | Debounced function. |
| `throttle(func, wait, options)` | Throttled function. `options.leading` and `options.trailing` (`false` to turn off). |
| `setup_timer(start, end, $element)` | Count from `start` to `end`, one step each second, in the text of `$element`. |

### Desk UI helpers

| Function | Description |
|---|---|
| `icon(icon_name, size = "sm", icon_class = "", icon_style = "", svg_class = "")` | SVG markup of a desk icon (`#icon-<name>`). A name that starts with `es-` uses the Espresso icon `#<name>`. `size`: `xs`, `sm`, `md`, `lg` and others, or `{ width, height }`. |
| `flag(country_code)` | `<img>` of a country flag from `flagcdn.com`. |
| `set_title(title)` | Set `document.title`, with the prefix of `set_title_prefix`. |
| `set_title_prefix(prefix)` | Set a title prefix and apply it. |
| `copy_to_clipboard(string)` | Copy text and show the green alert "Copied to clipboard.". |
| `get_clipboard_data(clipboard_paste_event)` | Text of a paste event. |
| `play_sound(name)` | Play the `#sound-<name>` audio element. Skipped when the user has `mute_sounds`. |
| `scroll_to(element, animate = true, additional_offset, element_to_be_scrolled, callback, highlight_element = false)` | Scroll to an element or a pixel offset. Skipped when `frappe.flags.disable_auto_scroll` is set. |
| `get_scroll_position(element, additional_offset)` | Scroll offset of an element, minus the navbar and page head height. |
| `add_custom_button(html, action, class_name = "", title = "", btn_type, wrapper, prepend)` | Add a small button to `wrapper`. `btn_type` defaults to `btn-secondary`. |
| `add_select_group_button(wrapper, actions, btn_type, icon = "", prepend)` | Add a split button. `actions` is `[{ label, description, action }]`. Returns the jQuery element. |
| `bind_actions_with_object($el, object)` | Clicks on `[data-action]` inside `$el` call `object[action](e, $target)`. |
| `setup_search($wrapper, el_class, text_class, data_attr)` | Filter elements of `$wrapper` with its `[data-element="search"]` input. |
| `set_footnote(footnote_area, wrapper, txt)` | Set or remove a footnote area in `wrapper`. |
| `build_summary_item(summary)` | jQuery of a report summary item (`label`, `value`, `datatype`, `currency`, `indicator`). |
| `guess_style(text, default_style, _colour)` | Indicator style from status words: `warning`, `danger`, `success`, `info`, or `default_style`. |
| `guess_colour(text)` | Color for status words: `orange`, `red`, `green`, `blue`, else `gray`. |
| `get_indicator_color(state)` | Promise of the color of a Workflow State, from its `style` or `guess_colour`. |
| `print(doctype, docname, print_format, letterhead, lang_code)` | Open `/printview` in a new window and start printing. |
| `resize_image(reader, callback, max_width, max_height)` | Resize the image of a `FileReader` (default 600 x 400). `callback(dataURL)` gets a JPEG data URL. |
| `csv_to_array(strData, strDelimiter)` | Parse CSV text into rows. Default delimiter `,`. |
| `is_image_file(filename)` / `is_video_file(filename)` | Check the file extension. Query strings are ignored. |
| `file_name_ellipsis(filename, length)` | Shorten a file name and keep its extension. |
| `load_video_player()` | Load `video_player.bundle.js`. Returns a Promise. |
| `new_auto_repeat_prompt(frm)` | Ask for a frequency and dates, then make an Auto Repeat for `frm.doc`. |
| `generate_tracking_url()` | Ask for UTM values and copy a tracking URL. |
| `set_meta_tag(route)` | Open the Website Route Meta of `route`, or a new one. |
| `warn_page_name_change()` | Show a message that a new page name breaks old URLs. |
| `datatable.get_translations()` | Translations for Frappe DataTable in the current language. |

### Environment

| Name | Description |
|---|---|
| `is_mac()` | `true` if `navigator.platform` is `MacIntel`. |
| `is_xs()`, `is_sm()`, `is_md()` | Document width below 768, 768 to 990, or 991 to 1198 px. |
| `is_rtl(lang = null)` | `true` for `ar`, `he`, `fa` and `ps`. Default: `frappe.boot.lang`. |
| `get_browser()` | `{ name, version }` from the user agent. |
| `is_current_user(user)` | `user === frappe.session.user`. |
| `supportsES6` | `true` if the browser parses ES6 arrow functions. |
| `map_defaults` | Default center, zoom, OpenStreetMap tiles and options for Geolocation maps. |

### Charts

| Function | Description |
|---|---|
| `make_chart(wrapper, custom_options = {})` | Make a `frappe.Chart` with default options (bar chart, short Y axis numbers, `format_chart_axis_number`) merged with `custom_options`. |
| `set_space_label_ratio(chart_args)` | Set `seriesLabelSpaceRatio` to 0.9 when there are more than 10 labels. |

## frappe.urllib

| Function | Description |
|---|---|
| `get_arg(name)` | Value of a query parameter of the current URL, or `""`. Also `window.get_url_arg`. |
| `get_dict()` | Object of the query parameters of the current URL. Also `window.get_url_dict`. |
| `get_base_url()` | `frappe.base_url` or `window.location.origin`, without a trailing `/`. |
| `get_full_url(url)` | Absolute URL of a path. URLs with `http://` or `https://` are returned as they are. |

`window.open_url_post(URL, PARAMS, new_window)` submits a hidden form with `POST` and the CSRF token. Values that are not strings are sent as JSON. Use it to download files from a whitelisted method.

```js
frappe.urllib.get_full_url("/files/a.png"); // "http://<site>/files/a.png"
```

## frappe.dom

| Function | Description |
|---|---|
| `freeze(msg, css_class)` | Show a full-page overlay `#freeze` with a message. Calls are counted. |
| `unfreeze()` | Decrease the count. Remove the overlay at 0. |
| `set_style(txt, id)` | Add a `<style>` tag to `<head>`. With `id`, replace the tag with that id. Returns the element. |
| `remove_script_and_style(txt)` | Remove `script`, `style`, `noscript`, `title`, `meta`, `base`, `head` and stylesheet `link` tags from HTML. |
| `eval(txt)` | Run code in a new `<script>` tag in `<head>` (global scope). |
| `by_id(id)` | `document.getElementById(id)`. |
| `get_unique_id()` | `"unique-<n>"`. |
| `set_unique_id(ele)` | Give the element a unique id if it has none. Return the id. |
| `add(parent, newtag, className, cs, innerHTML, onclick)` | Make an element and append it to `parent`. For `img`, `className` is the `src`. |
| `css(ele, s)` | Assign the style object `s` to `ele.style`. |
| `activate($parent, $child, common_class, active_class = "active")` | Move the active class to `$child`. |
| `is_element_in_viewport(el, tolerance = 0)` | `true` if the element is fully visible. |
| `is_element_in_modal(element)` | `true` if the element is inside a `.modal`. |
| `is_touchscreen()` | `true` if `ontouchstart` is in `window`. |
| `save_selection()` / `restore_selection(savedSel)` | Save and restore the text selection. |
| `handle_broken_images(container)` | Add the class `no-image` to images that fail to load. |
| `scroll_to_bottom(container)` | Scroll a container to the end. |
| `scroll_to_section(section_name)` | Open and scroll to a form section by its label. |
| `file_to_base64(file_obj)` | Promise of a data URL of a `File`. |
| `pixel_to_inches(pixels)` | Convert pixels to inches with the screen DPI. |

## Other helpers

| Name | Description |
|---|---|
| `frappe.scrub(text, spacer = "_")` | Replace spaces with `spacer` and lowercase. Hyphens are kept (`"Sample-Item"` gives `"sample-item"`). |
| `frappe.unscrub(txt)` | Replace `-` and `_` with spaces and capitalize words. |
| `frappe.ellipsis(text, max)` | Cut `text` at `max` characters (default 20) and add `...`. |
| `frappe.tools.downloadify(data, roles, title)` | Download a list of rows as `<title>.csv`. |
| `frappe.tools.to_csv(data)` | CSV text of a list of rows. Strings are quoted. |
| `repl(s, dict)` | Replace `%(key)s` with values of `dict`. |
| `replace_all(s, t1, t2)` | Replace every `t1` with `t2`. |
| `strip(s, chars)` / `lstrip(s, chars)` / `rstrip(s, chars)` | Remove characters from the ends. Default: newline, tab, space. |
| `strip_html(txt)` | Remove HTML tags. |
| `toTitle(str)` | Capitalize the first letter of each word. |
| `nth(number)` | Ordinal: `"1st"`, `"2nd"`, `"3rd"`, `"4th"`. |
| `has_words(list, item)` | `true` if `item` contains a word of `list`. `true` for an empty `item`. |
| `has_common(list1, list2)` | `true` if the lists share an item. |
| `validate_email(txt)`, `validate_phone(txt)`, `validate_name(txt)`, `validate_url(txt)` | `frappe.utils.validate_type` with that type. |
| `Array.prototype.uniqBy(key)` | Remove items with the same `key(item)`. |
| `Array.prototype.move(from, to)` | Move an item in place. |
| `Object.prototype.setDefault(key, default_value)` | Like Python `dict.setdefault`. |
| `String.prototype.plural(revert)` | English plural, or singular with `revert`. Knows irregular words such as `person`, `child`, `man`. `"Category".plural()` gives `"Categories"`. |
| `NEWLINE`, `TAB`, `UP_ARROW`, `DOWN_ARROW` | `"\n"`, `9`, `38`, `40`. |

## Deprecated

| Name | Use instead |
|---|---|
| `fmt_money(v, format)` | `format_currency(v, currency, decimals)` |
| `roundNumber(num, precision)` | `flt(num, precision)` or `_round` |
| `frappe.datetime.get_user_fmt()` | `frappe.datetime.get_user_date_fmt()` |

## Pitfalls

- `frappe.datetime.add_days` and `add_months` drop the time part of a datetime.
- `get_user_date_fmt()` and `get_user_time_fmt()` return the system formats. There is no per-user date format.
- `frappe.scrub` does not replace hyphens. The Python `frappe.scrub` does.
- `frappe.format` with a Currency field does not read a currency code from `df.options`. `{ options: "USD" }` gives the system currency.
- `filter_dict` with `<`, `<=`, `>` or `>=` compares against the whole `[op, value]` array, so these filters do not match.
- `frappe.utils.eval("eval:...")` throws a `SyntaxError`. Remove the `eval:` prefix first.
- `nth(11)`, `nth(12)` and `nth(13)` return `"11st"`, `"12nd"` and `"13rd"`.
- `"Person".plural()` returns `"people"`. Irregular words lose their capital letter.
- `frappe.tools.downloadify` checks `has_common(roles, roles)`, which is always true, so the role check never blocks.
- `make_event_emitter` `off()` does not remove handlers. It unbinds a new wrapper function.
- `frappe.format` does not escape HTML in a Data value. Escape user input with `frappe.utils.escape_html` before you show it. The `Tag` and `Assign` formatters also show values without escaping.
- `precision()` returns a string when no form is open.
- `is_null` returns `undefined` instead of `false`.

## Related

- [frappe (JavaScript)](frappe.md)
- [Client API](../../developer-guide/desk-ui/client-api.md)
- [Number Fields](../../developer-guide/fields/number-fields.md)
- [Date and Time Fields](../../developer-guide/fields/date-and-time-fields.md)
- [Model and DB](model-and-db.md)
- [UI Components](ui-components.md)
