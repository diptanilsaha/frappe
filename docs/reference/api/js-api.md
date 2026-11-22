---
title: JavaScript API
description: Client-side frappe namespaces in desk (frappe.call, frappe.db, frappe.model, frappe.ui, frappe.datetime, frappe.utils and more) with signatures.
version: develop
sources:
  - frappe/public/js/frappe/request.js
  - frappe/public/js/frappe/db.js
  - frappe/public/js/frappe/translate.js
  - frappe/public/js/frappe/provide.js
  - frappe/public/js/frappe/assets.js
  - frappe/public/js/frappe/router.js
  - frappe/public/js/frappe/socketio_client.js
  - frappe/public/js/frappe/defaults.js
  - frappe/public/js/frappe/dom.js
  - frappe/public/js/frappe/microtemplate.js
  - frappe/public/js/frappe/ui/messages.js
  - frappe/public/js/frappe/ui/dialog.js
  - frappe/public/js/frappe/ui/field_group.js
  - frappe/public/js/frappe/ui/page.js
  - frappe/public/js/frappe/model/model.js
  - frappe/public/js/frappe/model/create_new.js
  - frappe/public/js/frappe/model/meta.js
  - frappe/public/js/frappe/model/perm.js
  - frappe/public/js/frappe/form/form.js
  - frappe/public/js/frappe/form/script_manager.js
  - frappe/public/js/frappe/form/formatters.js
  - frappe/public/js/frappe/utils/datetime.js
  - frappe/public/js/frappe/utils/number_format.js
  - frappe/public/js/frappe/utils/utils.js
  - frappe/public/js/frappe/utils/user.js
---

# JavaScript API

This page lists the client-side `frappe` namespaces that desk loads. Use it to find a function name and its arguments. The names come from `Object.keys()` on each namespace in a logged-in desk at `/desk`. The signatures come from the source files listed in the frontmatter. Internal helpers are left out.

For usage and examples, see [Client API](../../developer-guide/desk-ui/client-api.md) and [Form Scripts](../../developer-guide/desk-ui/form-scripts.md).

!!! note
    Functions that talk to the server return a Promise or a jQuery deferred. Await them, or pass a callback where the signature has one.

## Server calls

| Function | Signature | Description |
|---|---|---|
| `frappe.call` | `(opts)` or `(method, args, callback, headers)` | Call a whitelisted method at `/api/method/<method>`. Returns a Promise. The result is in `r.message`. |
| `frappe.xcall` | `(method, params, type = "POST", opts = {})` | Call a whitelisted method. Resolves with `r.message`. Rejects on error. |

Options of `frappe.call(opts)`:

| Option | Description |
|---|---|
| `method` | Dotted path of the whitelisted method. |
| `args` | Arguments object. |
| `type` | HTTP method. Default `"POST"`. |
| `callback(r)` | Called on success. |
| `error(r)` | Called on error. |
| `always(r)` | Called after success or error. |
| `freeze`, `freeze_message` | Block the screen while the call runs. |
| `btn` | Button to disable while the call runs. |
| `doc` | Call `method` as a controller method of this document (`run_doc_method`). |
| `async` | `false` makes a synchronous request. |
| `quiet` | Do not show the spinner. |
| `debounce` | Skip the call if the same args were sent less than this many ms ago. |
| `cache` | Allow the browser to cache a `GET` response. |
| `api_version` | Use `/api/<version>/method/` instead of `/api/method/`. |
| `headers`, `error_handlers`, `silent`, `url` | Extra headers, handlers per error type, no error dialog, custom URL. |

```js
const r = await frappe.call({
	method: "frappe.client.get_value",
	args: { doctype: "Sample Item", filters: { category: "Hardware" }, fieldname: ["name", "rate"] },
});
// r.message -> { name: "Keyboard", rate: 45 }

const pong = await frappe.xcall("frappe.ping"); // "pong"
```

## frappe.db

Each function calls a method in `frappe.client` or `frappe.desk.reportview`. Permissions of the current user apply.

| Function | Signature | Description |
|---|---|---|
| `get_value` | `(doctype, filters, fieldname, callback, parent_doc)` | Return field values of the first match. The result is in `r.message`. |
| `get_single_value` | `(doctype, field)` | Resolve with one field of a Single DocType. |
| `get_list` | `(doctype, { fields, filters, order_by, limit, ... })` | Resolve with a list of records. Default `fields` is `["name"]`. Default `limit` is 20. |
| `get_doc` | `(doctype, name, filters)` | Resolve with the full document. Adds it to `locals`. |
| `exists` | `(doctype, nameOrFilters)` | Resolve with `true` or `false`. |
| `count` | `(doctype, { filters, limit } = {}, cache = false)` | Resolve with the number of matching records. |
| `insert` | `(doc)` | Insert a new document. `doc` must have `doctype`. |
| `set_value` | `(doctype, docname, fieldname, value, callback)` | Set one value, or many with a dict as `fieldname`. Saves the document on the server. |
| `delete_doc` | `(doctype, name)` | Delete a document. |
| `get_link_options` | `(doctype, txt = "", filters = {}, page_length = 0, include_image = false)` | Resolve with Link search results. |

```js
const items = await frappe.db.get_list("Sample Item", {
	fields: ["name", "rate"],
	filters: { category: "Hardware" },
	limit: 2,
});
// [{ name: "Keyboard", rate: 45 }, { name: "Monitor", rate: 250 }]
```

## Messages and dialogs

| Function | Signature | Description |
|---|---|---|
| `frappe.msgprint` | `(msg, title, is_minimizable, re_route)` | Show a message dialog. `msg` can be an object with `message`, `title`, `indicator`, `primary_action`. |
| `frappe.throw` | `(msg)` | Show a red message, then throw an `Error`. |
| `frappe.show_alert` | `(message, seconds = 7, actions = {})` | Show a toast. `message` can be `{ message, indicator }`. |
| `frappe.toast` | same as `show_alert` | Alias of `show_alert`. |
| `frappe.confirm` | `(message, confirm_action, reject_action, primary_label, secondary_label)` | Show a Yes and No dialog. |
| `frappe.warn` | `(title, message_html, proceed_action, primary_label, is_minimizable, secondary_label)` | Show a warning dialog with a proceed button. |
| `frappe.prompt` | `(fields, callback, title, primary_label)` | Ask for values in a dialog. `fields` is one field definition or a list. |
| `frappe.show_progress` | `(title, count, total = 100, description, hide_on_completion = false)` | Show or update a progress dialog. |
| `frappe.hide_progress` | `()` | Close the progress dialog. |
| `frappe.hide_msgprint` | `(instant)` | Close the message dialog. |
| `frappe.verify_password` | `(callback)` | Ask the user for their password, then call `callback`. |

### frappe.ui.Dialog

`new frappe.ui.Dialog(opts)` makes a dialog. `Dialog` extends `frappe.ui.FieldGroup`.

| Option | Description |
|---|---|
| `title` | Dialog title. |
| `fields` | List of field definitions (`fieldname`, `fieldtype`, `label`, `options`, `reqd`, `default`, ...). |
| `size` | `"small"`, `"large"` or `"extra-large"`. |
| `primary_action_label`, `primary_action(values)` | Main button. `values` comes from `get_values()`. |
| `secondary_action_label`, `secondary_action` | Second button. |
| `minimizable` | Allow the user to minimize the dialog. |

| Method | Description |
|---|---|
| `show()`, `hide()` | Open or close the dialog. |
| `get_values(ignore_errors, check_invalid)` | Return field values. Returns nothing if a mandatory field is empty. |
| `get_value(key)`, `set_value(key, val)`, `set_values(dict)` | Read or set fields. |
| `get_field(fieldname)` | Return the control of a field. |
| `set_df_property(fieldname, prop, value)` | Change a field property. |
| `set_query(fieldname, opt1, opt2)` | Set the query of a Link field. |
| `set_primary_action(label, click)` | Change the main button. |
| `disable_primary_action()`, `enable_primary_action()` | Toggle the main button. |
| `set_title(t)`, `set_message(text)`, `set_alert(text, alert_class = "info")` | Change title, message or alert. |
| `clear()` | Clear all values. |
| `add_custom_action(label, action, css_class)` | Add a button to the footer. |

```js
const d = new frappe.ui.Dialog({
	title: __("New Sample Item"),
	fields: [
		{ fieldname: "item_name", fieldtype: "Data", label: __("Item Name"), reqd: 1 },
		{ fieldname: "rate", fieldtype: "Currency", label: __("Rate") },
	],
	primary_action_label: __("Create"),
	primary_action(values) {
		frappe.db.insert({ doctype: "Sample Item", ...values }).then(() => d.hide());
	},
});
d.show();
```

## Translation

| Function | Signature | Description |
|---|---|---|
| `__` | `(txt, replace, context = null)` | Return the translated string. `replace` is a list for `{0}`, `{1}`. Same as `frappe._`. |

```js
__("Hello {0}", ["World"]); // "Hello World"
```

## Forms

| Function | Signature | Description |
|---|---|---|
| `frappe.ui.form.on` | `(doctype, fieldname, handler)` or `(doctype, events)` | Add form event handlers. See [Form Scripts](../../developer-guide/desk-ui/form-scripts.md). |
| `frappe.ui.form.off` | `(doctype, fieldname, handler)` | Remove a handler. |
| `frappe.ui.form.trigger` | `(doctype, fieldname)` | Run the handlers of a field event on the current form. |
| `frappe.ui.form.make_quick_entry` | `(doctype, after_insert, init_callback, doc, force, skip_insert)` | Open the quick entry dialog. |
| `frappe.ui.form.make_control` | `(opts)` | Make one form control outside a form. |
| `frappe.new_doc` | `(doctype, opts, init_callback)` | Open a new form, or the quick entry dialog. `opts` sets field values. |

Common methods of `frm` (`frappe.ui.form.Form`, defined in `frappe/public/js/frappe/form/form.js`):

| Method | Signature |
|---|---|
| `set_value` | `(field, value, if_missing, skip_dirty_trigger = false)` |
| `refresh_field` | `(fname)` |
| `refresh_fields` | `()` |
| `set_query` | `(fieldname, opt1, opt2)` |
| `set_df_property` | `(fieldname, property, value, docname, table_field, table_row_name = null)` |
| `toggle_display` | `(fnames, show)` |
| `toggle_reqd` | `(fnames, mandatory)` |
| `toggle_enable` | `(fnames, enable)` |
| `add_custom_button` | `(label, fn, group, type = "default")` |
| `remove_custom_button` | `(label, group)` |
| `clear_custom_buttons` | `()` |
| `add_child` | `(fieldname, values)` |
| `clear_table` | `(fieldname)` |
| `call` | `(opts, args, callback)` |
| `save` | `(save_action, callback, btn, on_error)` |
| `savesubmit` | `(btn, callback, on_error)` |
| `savecancel` | `(btn, callback, on_error)` |
| `reload_doc` | `()` |
| `is_new` | `()` |
| `is_dirty` | `()` |
| `dirty` | `()` |
| `set_intro` | `(txt, color)` |
| `get_field` | `(field)` |
| `has_perm` | `(ptype)` |
| `enable_save` / `disable_save` | `()` / `(set_dirty = false)` |
| `set_read_only` | `()` |
| `scroll_to_field` | `(fieldname, focus = true)` |
| `get_selected` | `()` |
| `trigger` | `(event, doctype, docname)` |
| `add_fetch` | `(link_field, source_field, target_field, target_doctype)` |

The global `cur_frm` points to the current form. Prefer the `frm` argument of event handlers.

## frappe.model

Documents in the browser live in `locals[doctype][name]`.

| Function | Signature | Description |
|---|---|---|
| `get_doc` | `(doctype, name)` | Return a document from `locals`. Also `frappe.get_doc`. |
| `get_list` | `(doctype, filters)` | Return documents in `locals` that match. Also `frappe.get_list`. |
| `with_doctype` | `(doctype, callback, async)` | Load DocType metadata, then call `callback`. Returns a Promise. |
| `with_doc` | `(doctype, name, callback, error_callback)` | Load a document into `locals`. Returns a Promise. |
| `get_new_doc` | `(doctype, parent_doc, parentfield, with_mandatory_children)` | Make a new local document with defaults. Its name starts with `new-`. |
| `add_child` | `(parent_doc, doctype, parentfield, idx)` | Add a child row to a local document. |
| `clear_table` | `(doc, parentfield)` | Remove all rows of a child table. |
| `copy_doc` | `(doc, from_amend, parent_doc, parentfield)` | Make a local copy of a document. |
| `set_value` | `(doctype, docname, fieldname, value, fieldtype, skip_dirty_trigger = false)` | Set a value on a local document and run change triggers. |
| `get_value` | `(doctype, filters, fieldname, callback)` | Read from `locals`. With a callback, read from the server. |
| `delete_doc` | `(doctype, docname, callback)` | Ask, then delete on the server. |
| `rename_doc` | `(doctype, docname, callback)` | Open the rename dialog. |
| `open_mapped_doc` | `(opts)` | Call a mapper method (`opts.method`, `opts.frm`) and open the new document. |
| `round_floats_in` | `(doc, fieldnames)` | Round float fields to their precision. |
| `get_doc_title` | `(doc)` | Return the title of a document. |
| `is_submittable`, `is_table`, `is_single`, `is_tree` | `(doctype)` | Check DocType flags. |
| `can_read`, `can_write`, `can_create`, `can_delete`, `can_submit`, `can_cancel` | `(doctype)` | Check the user permission on a DocType. |
| `can_print`, `can_email`, `can_export`, `can_import`, `can_share` | `(doctype, frm)` | Check other rights. |
| `has_workflow` | `(doctype)` | Return `true` if the DocType has an active workflow. |
| `scrub`, `unscrub` | `(txt)` | Same as the Python helpers. |

`frappe.get_meta(doctype)` returns the metadata from `locals.DocType`. It returns `undefined` until the DocType is loaded. Call `frappe.model.with_doctype` first.

## Routing

| Function | Signature | Description |
|---|---|---|
| `frappe.set_route` | `(...parts)` or `(route_array)` or `(route_string)` | Go to a desk route, for example `frappe.set_route("Form", "ToDo", name)`. |
| `frappe.get_route` | `()` | Current route as an array. |
| `frappe.get_route_str` | `()` | Current route joined with `/`. |
| `frappe.get_prev_route` | `()` | Previous route. |
| `frappe.route_options` | object | Set before `set_route`. The next list applies them as filters, the next new form as values. |
| `frappe.router.on` | `(evt, handler)` | Listen to router events such as `change`. |
| `frappe.router.off` / `once` | `(evt, handler)` | Remove or run once. |
| `frappe.router.slug` | `(name)` | Route slug of a DocType name. |
| `frappe.utils.get_form_link` | `(doctype, name, html = false, display_text = null, query_params_obj = {})` | Desk URL of a form, for example `/desk/todo/<name>`. With `html=true`, an `<a>` tag. |

## Realtime

`frappe.realtime` is a `RealTimeClient` (Socket.IO). See [Realtime](../../developer-guide/runtime/realtime.md).

| Method | Signature | Description |
|---|---|---|
| `on` | `(event, callback)` | Listen to an event sent by `frappe.publish_realtime`. |
| `off` | `(event, callback)` | Stop listening. |
| `emit` | `(event, ...args)` | Send an event to the Socket.IO server. |
| `publish` | `(event, message)` | Send an event to other clients through the server. |
| `doc_subscribe` / `doc_unsubscribe` | `(doctype, docname)` | Join or leave the room of a document. |
| `doctype_subscribe` / `doctype_unsubscribe` | `(doctype)` | Join or leave the room of a DocType (list updates). |
| `task_subscribe` / `task_unsubscribe` | `(task_id)` | Join or leave the room of a task. |

## Dates and times

`frappe.datetime`. Strings are in `yyyy-mm-dd` (system format) unless the name says `user`.

| Function | Signature | Description |
|---|---|---|
| `get_today` | `()` | Today as `yyyy-mm-dd`. |
| `nowdate` | `()` | Same as `get_today`. |
| `now_date` / `now_time` / `now_datetime` | `(as_obj = false)` | Now in the user time zone. |
| `system_datetime` | `(as_obj = false)` | Now in the system time zone. |
| `add_days` | `(d, days)` | Add days. |
| `add_months` | `(d, months)` | Add months. |
| `get_diff` / `get_day_diff` | `(d1, d2)` | Days between `d1` and `d2` (`d1` minus `d2`). |
| `get_hour_diff` / `get_minute_diff` | `(d1, d2)` | Hours or minutes between two datetimes. |
| `str_to_obj` / `obj_to_str` | `(d)` | String to `Date`, `Date` to string. |
| `str_to_user` | `(val, only_time = false, only_date = false)` | System format to user format. |
| `user_to_str` | `(val, only_time = false)` | User format to system format. |
| `user_to_obj` / `obj_to_user` | `(d)` | Convert between user format and `Date`. |
| `month_start` / `month_end` | `()` | First or last day of this month. |
| `week_start` / `week_end`, `quarter_start` / `quarter_end`, `year_start` / `year_end` | `()` | Boundaries of this week, quarter or year. |
| `global_date_format` | `(d)` | Localized long date. |
| `comment_when` | `(datetime, mini)` | Relative time HTML, for example `2 hours ago`. |
| `prettyDate` | `(date, mini)` | Relative time text. |
| `convert_to_user_tz` / `convert_to_system_tz` | `(date, format)` | Convert between time zones. |
| `get_user_date_fmt` / `get_user_time_fmt` / `get_user_fmt` | `()` | Format strings of the user. |
| `validate` | `(d)` | Return `true` if the value is a valid date. |

```js
frappe.datetime.add_days("2026-01-31", 1);              // "2026-02-01"
frappe.datetime.get_day_diff("2026-03-01", "2026-02-01"); // 28
frappe.datetime.str_to_user("2026-01-15");              // "15-01-2026" with date format dd-mm-yyyy
```

## Numbers and formatting

| Function | Signature | Description |
|---|---|---|
| `frappe.format` | `(value, df, options, doc)` | Format a value by its field definition `df`. `options.inline` returns plain output. |
| `frappe.form.get_formatter` | `(fieldtype)` | Return the formatter function of a fieldtype. |
| `flt` | `(v, decimals, number_format, rounding_method)` | Global. Convert to float. |
| `cint` | `(v, def)` | Global. Convert to integer. |
| `cstr` | `(s)` | Global. Convert to string. |
| `format_currency` | `(v, currency, decimals)` | Global. Format with a currency symbol. |
| `format_number` | `(v, format, decimals)` | Global. Format with digit groups. |
| `precision` | `(fieldname, doc)` | Global. Precision of a field. |
| `in_list` | `(list, item)` | Global. Return `true` if `item` is in `list`. |
| `is_null` | `(v)` | Global. Return `true` for `null`, `undefined` or `""`. |

```js
format_currency(1234.5, "USD"); // "$ 1,234.50"
flt("1,234.567", 2);            // 1234.57
```

## frappe.utils

Selected helpers from `frappe/public/js/frappe/utils/utils.js`:

| Function | Signature | Description |
|---|---|---|
| `escape_html` / `unescape_html` | `(txt)` | Escape or unescape HTML. |
| `xss_sanitise` | `(string, options)` | Remove script vectors from a string. |
| `html2text` | `(html)` | Text content of HTML. |
| `is_html`, `is_url`, `is_json` | `(txt)` | Type checks. |
| `parse_json` | `(str)` | Parse JSON. Returns the input if it is not a string. |
| `get_url_arg` | `(name)` | Query string value of the current URL. |
| `get_query_params` | `(query_string)` | Parse a query string into an object. |
| `make_query_string` | `(obj, encode = true)` | Build a query string. |
| `get_form_link` | `(doctype, name, html = false, display_text = null, query_params_obj = {})` | See Routing. |
| `copy_to_clipboard` | `(string, message)` | Copy text and show an alert. |
| `debounce` | `(func, wait, immediate)` | Debounce a function. |
| `throttle` | `(func, wait, options)` | Throttle a function. |
| `sleep` | `(time)` | Promise that resolves after `time` ms. |
| `unique` | `(list)` | Remove duplicates. |
| `sum` | `(list)` | Sum of a list. |
| `deep_equal` | `(a, b)` | Deep compare two values. |
| `comma_and` / `comma_or` | `(list)` | Join a list for display. |
| `icon` | `(icon_name, size = "sm", icon_class = "", icon_style = "", svg_class = "")` | SVG markup of a desk icon. |
| `get_indicator_color` | `(state)` | Indicator color of a status. |
| `shorten_number` | `(number, country, min_length = 4, max_no_of_decimals = 2)` | Short form, for example `1.2K`. |
| `seconds_to_duration` | `(seconds, duration_options)` | Split seconds into days, hours, minutes and seconds. |
| `get_formatted_duration` | `(value, duration_options = null)` | Duration text. |
| `print` | `(doctype, docname, print_format, letterhead, lang_code)` | Open the print view. |
| `play_sound` | `(name)` | Play a desk sound. |
| `is_mac`, `is_xs`, `is_sm`, `is_md` | `()` | Platform and screen size checks. |

## Session, user and defaults

| Name | Signature | Description |
|---|---|---|
| `frappe.session.user` | string | Current user name. Also `user_fullname`, `user_email`. |
| `frappe.boot` | object | Boot data loaded with desk: `sysdefaults`, `user`, installed apps and more. |
| `frappe.sys_defaults` | object | System defaults, such as `date_format` and `currency`. |
| `frappe.user.has_role` | `(rl)` | Return `true` if the user has the role, or one of a list of roles. |
| `frappe.user.full_name` / `image` / `abbr` | `(uid)` | Info of a user. |
| `frappe.user_info` | `(uid)` | Return info about a user, such as `fullname`, `image` and `abbr`. |
| `frappe.user_roles` | array | Roles of the current user. |
| `frappe.defaults.get_default` | `(key)` | Return a default from `frappe.boot.user.defaults`. Respects user permissions. |
| `frappe.defaults.get_user_default` | `(key)` | User default. |
| `frappe.defaults.get_global_default` | `(key)` | Global default. |
| `frappe.defaults.get_user_permissions` | `()` | User permissions of the current user. |
| `frappe.perm.has_perm` | `(doctype, permlevel = 0, ptype = "read", doc)` | Check a permission. |
| `frappe.perm.get_perm` | `(doctype, doc)` | Permissions per permlevel. |

## Pages, assets and DOM

| Function | Signature | Description |
|---|---|---|
| `frappe.ui.make_app_page` | `(opts)` | Make a desk page in `opts.parent`. See [Desk Pages](../../developer-guide/desk-ui/desk-pages.md). |
| `frappe.provide` | `(namespace)` | Make a nested namespace object, for example `frappe.provide("sample_app.utils")`. |
| `frappe.require` | `(items, callback)` | Load JS or CSS assets. Returns a Promise. |
| `frappe.render_template` | `(name, data)` | Render a client template (`.html` in the bundle). |
| `frappe.run_serially` | `(tasks)` | Run functions one after the other. Each can return a Promise. |
| `frappe.timeout` | `(seconds)` | Promise that resolves after `seconds`. |
| `frappe.dom.freeze` / `unfreeze` | `(msg, css_class)` / `()` | Block the screen with a message. |
| `frappe.ui.keys.on` | `(key, handler)` | Bind a keyboard shortcut, for example `"ctrl+s"`. |
| `frappe.ui.keys.add_shortcut` | `({ shortcut, action, description, page, ... })` | Add a shortcut shown in the shortcuts dialog. |
| `frappe.is_mobile` | `()` | Return `true` on a small screen. |

## Classes

`frappe.ui` and `frappe.views` hold classes. These are the ones apps use most:

| Class | Description |
|---|---|
| `frappe.ui.Dialog` | Modal dialog with fields. |
| `frappe.ui.FieldGroup` | Fields rendered in any container. |
| `frappe.ui.Page` | Desk page with title, buttons and menu. |
| `frappe.ui.FileUploader` | File upload dialog. |
| `frappe.ui.Scanner` | Barcode and QR scanner. |
| `frappe.ui.form.MultiSelectDialog` | Dialog to pick many documents. |
| `frappe.ui.form.LinkSelector` | Dialog to search and pick a linked document. |
| `frappe.Chart` | Frappe Charts. |
| `frappe.DataTable` | Frappe DataTable. |
| `frappe.views.ListView` | List view. Extend it through `frappe.listview_settings[doctype]`. See [List View Settings](../../developer-guide/desk-ui/list-view-settings.md). |
| `frappe.views.QueryReport` | Script and query report view. |

## Related

- [Client API](../../developer-guide/desk-ui/client-api.md)
- [Form Scripts](../../developer-guide/desk-ui/form-scripts.md)
- [Python API](python-api.md)
- [Field Types](../data-model/field-types.md)
