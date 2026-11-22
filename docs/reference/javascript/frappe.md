---
title: frappe (JavaScript)
description: Reference of the global frappe namespace in desk - server calls, messages, routing, realtime, boot, session, user, defaults, translation, metadata, toolbar and keyboard shortcuts.
version: v16
sources:
  - frappe/public/js/frappe/request.js
  - frappe/public/js/frappe/provide.js
  - frappe/public/js/frappe/assets.js
  - frappe/public/js/frappe/translate.js
  - frappe/public/js/frappe/ui/messages.js
  - frappe/public/js/frappe/router.js
  - frappe/public/js/frappe/event_emitter.js
  - frappe/public/js/frappe/socketio_client.js
  - frappe/public/js/frappe/desk.js
  - frappe/public/js/frappe/defaults.js
  - frappe/public/js/frappe/sys_defaults.js
  - frappe/public/js/frappe/utils/user.js
  - frappe/public/js/frappe/utils/common.js
  - frappe/public/js/frappe/model/meta.js
  - frappe/public/js/frappe/microtemplate.js
  - frappe/public/js/frappe/dom.js
  - frappe/public/js/frappe/utils/tools.js
  - frappe/public/js/frappe/utils/preview_email.js
  - frappe/public/js/frappe/ui/keyboard.js
  - frappe/public/js/frappe/ui/alt_keyboard_shortcuts.js
  - frappe/public/js/frappe/ui/toolbar/toolbar.js
  - frappe/public/js/frappe/form/print_utils.js
  - frappe/boot.py
  - realtime/handlers.js
---

# frappe (JavaScript)

This page lists the functions and properties of the global `frappe` object in desk (`/desk`). Use it to look up a signature, a default or a side effect. The names come from `Object.keys()` on a logged-in desk page on v16. The signatures come from the source files in the frontmatter. For how-to guides, see [Client API](../../developer-guide/desk-ui/client-api.md) and [Form Scripts](../../developer-guide/desk-ui/form-scripts.md).

Other parts of the client API have their own pages:

| Namespace | Page |
|---|---|
| `frappe.utils`, `frappe.datetime`, `frappe.format`, `flt`, `cint`, `frappe.dom` | [Utils and Formatting](utils-and-formatting.md) |
| `frappe.ui.form.Form` (`frm`), grids, form events | [Form](form.md) |
| `frappe.model`, `frappe.db`, `locals`, `frappe.perm` | [Model and DB](model-and-db.md) |
| `frappe.ui.Dialog`, `frappe.ui.Page`, other UI classes | [UI Components](ui-components.md) |
| List, report, Kanban, calendar and tree views | [List and Report Views](list-and-report-views.md) |
| REST routes that these calls use | [REST Endpoints](../http/rest-endpoints.md) |

## Server calls

### frappe.call

```js
frappe.call(opts)
frappe.call(method, args, callback, headers)
```

Calls a whitelisted server method. The default URL is `/api/method/<method>`. With the string form, the four arguments become `opts.method`, `opts.args`, `opts.callback` and `opts.headers`.

Returns the jQuery `jqXHR` of the request. It is a thenable. It resolves with the response body (`r.message` holds the return value). On an HTTP error it rejects with the `jqXHR` object. When `debounce` skips the call, it returns `Promise.resolve()`.

| Option | Type | Default | Description |
|---|---|---|---|
| `method` | string | | Dotted path of the whitelisted method. With `doc`, the name of the controller method. |
| `args` | object | `{}` | Arguments. The body is always form-encoded. Objects and arrays are sent as JSON strings. |
| `type` | string | `"POST"` | HTTP method. |
| `callback(r, response_text)` | function | | Runs after a 200 response. |
| `error(r)` | function | | Runs after an error. The argument depends on the status code. For 417 it is the parsed body. For 404 and 403 it is `undefined`. |
| `always(r)` | function | | Runs after success or error, with the parsed body or `null`. |
| `doc` | object | | Run `method` as a controller method of this document. Sends `cmd=run_doc_method` and the document from `locals` as `docs`. |
| `module`, `page` | string | | With both set, calls `<module>.page.<page>.<page>.<method>`. |
| `btn` | element | | Disable this button while the request runs. |
| `freeze` | bool | `false` | Show a full-page overlay (`frappe.dom.freeze`) while the request runs. `args.freeze` and `args.freeze_message` also turn it on. They are still sent to the server. |
| `freeze_message` | string | | Text on the overlay. |
| `headers` | object | `{}` | Extra request headers. |
| `error_handlers` | object | `{}` | Map of exception name (`exc_type`) to handler for this call. See below. |
| `silent` | bool | `false` | Do not show `_server_messages` from the response. Status-code dialogs (403, 404, 500 and others) still show. |
| `async` | bool | `true` | `false` makes a blocking request. |
| `debounce` | number | | Skip the call if a call with the same args was sent less than this many ms ago. |
| `cache` | bool | `false` | Let the browser cache the response. Ignored when `window.dev_server` is set. |
| `api_version` | string | | Use `/api/<api_version>/method/`. With `"v2"`, messages are read from `r.messages`. |
| `url` | string | | Request this URL instead of `/api/method/<method>`. |
| `queued(data)` | function | | Runs when the server returns a `task_id` (a queued call). The client subscribes to the task with `frappe.realtime.subscribe`. |
| `quiet` | bool | | Sets `opts.no_spinner`. `frappe.call` does not pass it on, so it has no effect. |

If the browser is offline when the call starts, a "Connection Lost" toast shows. The request is still sent.

Each request sends the headers `X-Frappe-CSRF-Token`, `Accept: application/json` and `X-Frappe-CMD`. If `args.doctype` is set, it also sends `X-Frappe-Doctype`.

What happens with the response:

1. `docs` and `docinfo` in the response are synced into `locals` (`frappe.model.sync`).
2. `__messages` are merged into the translation dictionary. `_link_titles` are merged into the link title cache.
3. Handlers for `r.exc_type` run: first the global ones from `frappe.request.on_error`, then the one in `opts.error_handlers`.
4. If no handler ran and `silent` is not set, `_server_messages` are shown with `frappe.msgprint`.
5. Some status codes show their own dialog: 403 "Not permitted", 404 "Not found", 413 "File too big", 500 "Server Error" (with a copy or report button), 502, 504 and 508. A 401 or an expired session opens the session-expired dialog.
6. `QueryTimeoutError` and `QueryDeadlockError` show a "please try again" dialog.

```js
const r = await frappe.call({
	method: "frappe.client.get_count",
	args: { doctype: "ToDo" },
	type: "GET",
});
// r.message -> number of ToDo records

frappe.call("frappe.ping").then((r) => console.log(r.message)); // "pong"
```

### Related functions

| Function | Description |
|---|---|
| `frappe.xcall(method, params, type, opts = {})` | Promise wrapper of `frappe.call`. `type` defaults to `"POST"`. Resolves with `r.message`. Rejects with `r?.message` of the error argument, which is often `undefined`. `opts` is spread into the call options. |
| `frappe.request.on_error(error_type, handler)` | Add a global handler for an exception name, for example `"DoesNotExistError"`. It runs for every call that returns that `exc_type`. While a handler exists, `_server_messages` of that call are not shown. |
| `frappe.after_ajax(fn)` | Return a Promise that resolves with `fn()` after all running requests finish. Runs at once if no request is running. |
| `frappe.after_server_call()` | Return a Promise that resolves when running requests finish, or `null` if none run. |
| `frappe.last_request` | Data of the last request sent. |
| `frappe.last_response` | Parsed body of the last response. |
| `frappe.csrf_token` | CSRF token sent with each request. |

```js
const pong = await frappe.xcall("frappe.ping"); // "pong"

frappe.request.on_error("DoesNotExistError", (r) => {
	console.log("Missing record", r.exc_type);
});
```

!!! note "Version"
    v16 always sends a form-encoded body. Later versions can send a JSON body for apps that opt in.

## Namespaces and assets

| Function | Description |
|---|---|
| `frappe.provide(namespace)` | Make nested objects on `window` for a dotted name. Return the last object. Existing objects are kept. |
| `frappe.require(items, callback)` | Load JS or CSS files. `items` is a path or a list. Names with `.bundle.` resolve through `frappe.boot.assets_json`. Returns a Promise. Calls `callback` after the Promise resolves. Freezes the page while files load. A file that fails to load still resolves. Each path loads once. |
| `frappe.assets.bundled_asset(path, is_rtl = null)` | Return the built file path of a bundle name, for example `"desk.bundle.css"`. |

```js
// sample_app/public/js/sample.js
frappe.provide("sample_app.utils");
sample_app.utils.greet = () => frappe.show_alert(__("Hello"));

await frappe.require("sample_chart.bundle.js");
```

## Messages and dialogs

| Function | Returns | Description |
|---|---|---|
| `frappe.msgprint(msg, title, is_minimizable, re_route)` | `frappe.msg_dialog` | Show a message dialog. `msg` is a string, a JSON string or an options object (below). A second call before the dialog closes appends the message below a line. Also `window.msgprint`. |
| `frappe.throw(msg)` | | Call `frappe.msgprint` with a red indicator and the title "Error", then throw `Error(msg.message)`. `msg` is a string or an options object. |
| `frappe.hide_msgprint(instant)` | | Clear and close the message dialog. `instant` skips the fade. |
| `frappe.update_msgprint(html)` | | Replace the content of the open message dialog. Opens a new one if none is open. |
| `frappe.show_alert(message, seconds = 7, actions = {})` | jQuery element | Show a toast. `message` is a string or `{ message, indicator, subtitle, body }`. `indicator` maps to a toast type: `green` success, `red` error, `orange` or `yellow` warning, `blue` info. `actions` maps a `data-action` value in the message to a click handler. |
| `frappe.toast(...)` | jQuery element | Same function as `frappe.show_alert`. |
| `frappe.confirm(message, confirm_action, reject_action, primary_label, secondary_label)` | `frappe.ui.Dialog` | Show a dialog with "Yes" and "No". `reject_action` runs when the dialog closes without the primary action. <kbd>Enter</kbd> clicks "Yes". |
| `frappe.warn(title, message_html, proceed_action, primary_label, is_minimizable, secondary_label)` | `frappe.ui.Dialog` | Show a warning with a red primary button. `secondary_label` defaults to "Cancel". |
| `frappe.verify_password(callback)` | | Ask for the password of the current user ("Verify Password" dialog). Calls `frappe.core.doctype.user.user.verify_password`. Runs `callback()` if the password is correct. |
| `frappe.prompt(fields, callback, title, primary_label)` | `frappe.ui.Dialog` | Ask for values. `fields` is a field definition, a list of them, or a label string (makes one mandatory Data field named `value`). `callback(values)` runs on submit. Default title "Enter Value", default button "Submit". |
| `frappe.show_progress(title, count, total = 100, description, hide_on_completion = false)` | `frappe.ui.Dialog` | Show or update a progress dialog. A call with the same `title` updates the open dialog. With `hide_on_completion`, it closes 500 ms after 100 percent. |
| `frappe.hide_progress()` | | Close the progress dialog. |
| `frappe.messages.waiting(parent, msg)` | jQuery element | Append a centered message box to `parent`. |
| `frappe.messages.get_waiting_message(msg)` | string | HTML of that message box. |

Options of `frappe.msgprint` and `frappe.throw` when `msg` is an object:

| Key | Description |
|---|---|
| `message` | Text or HTML. A list shows each item with its own `msgprint` call. In a list from the server, items with `raise_exception` show first and the others are skipped. |
| `title` | Dialog title. Default "Message". |
| `indicator` | Color of the title indicator. Default `blue`. |
| `as_list` | Show `message` (a list) as a bulleted list. |
| `as_table` | Show `message` (a list of rows) as a table. |
| `alert`, `toast` | Show as a toast with `frappe.show_alert` instead of a dialog. |
| `clear` | Clear earlier messages in the open dialog. |
| `wide` | Use the normal dialog width instead of the narrow one. |
| `is_minimizable` | Allow the user to minimize the dialog. |
| `re_route` | Go to the previous route when the dialog closes. |
| `primary_action` | `{ label, action }`. Or `{ label, server_action, args, hide_on_success }` to call a whitelisted method. Or `{ label, client_action, args }` with a dotted path of a global function. |
| `secondary_action` | `{ label, action }`. Default label "Close". |

```js
frappe.msgprint({
	title: __("Items"),
	message: [__("Keyboard"), __("Monitor")],
	as_list: 1,
	indicator: "green",
});

frappe.confirm(__("Close this order?"), () => {
	frappe.show_alert({ message: __("Closed"), indicator: "green" });
});

frappe.prompt(__("Item Name"), ({ value }) => console.log(value));
```

## Translation

| Function | Description |
|---|---|
| `__(txt, replace, context = null)` | Return the translation of `txt` from `frappe._messages`, or `txt` if none exists. With `context`, look up `"<txt>:<context>"` first. `replace` is a list (or object) that fills `{0}`, `{1}`. A non-string `txt` is returned as it is. |
| `frappe._(txt, replace, context = null)` | Same function as `__`. |
| `frappe.get_languages()` | Return `[{ label, value }]` from `frappe.boot.lang_dict`, sorted by `value` (language code). |
| `frappe._messages` | Translation dictionary of the current language. Responses can add to it (`__messages`). |

```js
__("Hello {0}", ["World"]); // "Hello World"
__("Confirm", null, "Title of confirmation dialog");
```

See [Translations](../../developer-guide/building-apps/translations.md).

## Routing

Desk routes start with `/desk`. A route is an array, for example `["List", "ToDo", "List"]` or `["Form", "ToDo", "<name>"]`.

| Function | Description |
|---|---|
| `frappe.set_route(...parts)` | Go to a route. Accepts parts (`"Form", "ToDo", name`), one array, or one path string (`"todo/view/report"` or `"/desk/todo?status=Open"`). A plain object in the parts becomes `frappe.route_options`. Returns a Promise that resolves after the new page loads. Clears `frappe.route_flags`. |
| `frappe.get_route()` | Current route array (`frappe.router.current_route`). |
| `frappe.get_route_str()` | Current route joined with `/`, for example `"List/ToDo/List"`. |
| `frappe.get_prev_route()` | Previous route array from `frappe.route_history`, or `[]`. |
| `frappe.set_re_route(...parts)` | Call `frappe.set_route` and record a redirect from the old sub path to the new one in `frappe.re_route`. |
| `frappe.has_route_options()` | Return `true` if `frappe.route_options` has keys. |

| Property | Description |
|---|---|
| `frappe.route_options` | Object or `null`. Set it before `set_route`. Its keys become the query string. A list view reads them as filters. A new form reads them as values. The router sets it from the query string of the URL. |
| `frappe.route_flags` | Flags for the next route. `replace_route` uses `history.replaceState` instead of `pushState`. `jump` marks a route picked from outside the sidebar. |
| `frappe.open_in_new_tab` | Set to `true` before `set_route` to open the route in a new tab. Reset after the call. |
| `frappe.route_history` | List of visited routes. |
| `frappe.route_titles` | Map of sub path to page title. |
| `frappe.re_route` | Map of old sub path to new sub path. |
| `frappe.route_hooks` | Object for app code. Empty by default. |

Route conversion (`frappe.router.convert_from_standard_route`):

| Route | URL |
|---|---|
| `["List", "Sample Item"]` | `/desk/sample-item` |
| `["List", "Note", "Report"]` | `/desk/note/view/report` |
| `["Form", "Sample Item", "ITEM-1"]` | `/desk/sample-item/ITEM-1` |
| `["Tree", "Sample Item"]` | `/desk/sample-item/view/tree` |

The router can add a sidebar shell segment to the URL after `/desk`, for example `/desk/build/todo`. `frappe.get_route()` does not contain it.

`frappe.router` members that app code can use:

| Member | Description |
|---|---|
| `on(evt, handler)` | Run `handler(router)` on an event. The router triggers `"change"` after each route change. |
| `once(evt, handler)` | Run `handler` on the next event only. |
| `off(evt, handler)` | Does not remove the handler. See Pitfalls. |
| `trigger(evt, data)` | Trigger an event. |
| `current_route` | Current route array. |
| `slug(name)` | Lowercase `name` and replace spaces with `-`. `"Sample Order Item"` becomes `"sample-order-item"`. |
| `routes` | Map of slug to `{ doctype }` for each DocType the user can read. Each DocType Layout in `frappe.boot.doctype_layouts` adds its own slug with `{ doctype, doctype_layout }`. |
| `list_views` | View names valid after `/view/`: `list`, `kanban`, `report`, `calendar`, `tree`, `gantt`, `dashboard`, `image`, `inbox`, `map`. |
| `get_sub_path(route)` | Current path after `/desk/`, decoded. |
| `make_url(params)` | Build `/desk/<parts>` from an array. |
| `convert_from_standard_route(route)` | Convert a route array to URL parts (table above). |

```js
frappe.route_options = { status: "Open" };
await frappe.set_route("List", "ToDo");
// URL: /desk/<shell>/todo?status=Open

frappe.router.on("change", () => console.log(frappe.get_route_str()));
```

## Realtime

`frappe.realtime` is a `RealTimeClient` that wraps a Socket.IO client. `frappe.socketio` is the same object. It connects to `<origin>/<sitename>`, on port `frappe.boot.socketio_port` when `window.dev_server` is set. If `frappe.boot.disable_async` is set, it does not connect and all methods do nothing. See [Realtime](../../developer-guide/runtime/realtime.md).

| Method | Description |
|---|---|
| `on(event, callback)` | Listen to an event that the server sends with `frappe.publish_realtime`. |
| `off(event, callback)` | Remove `callback`. Without `callback`, Socket.IO removes only the first listener of the event. |
| `emit(event, ...args)` | Send an event to the Socket.IO server. The server only handles its own events (below). |
| `publish(event, message)` | Same as `emit(event, message)`. The server does not relay it to other clients. |
| `doc_subscribe(doctype, docname)` | Join the room of a document. The server checks read permission. Calls within 1 second of the last `doc_subscribe` are dropped. Forms call it on load. |
| `doc_unsubscribe(doctype, docname)` | Leave the room of a document. |
| `doc_open(doctype, docname)` / `doc_close(doctype, docname)` | Join or leave the "who is viewing" room. Forms call them on refresh and unload. |
| `doctype_subscribe(doctype)` / `doctype_unsubscribe(doctype)` | Join or leave the room of a DocType (list updates). Callers are not counted. One `doctype_unsubscribe` leaves the room for all views. |
| `task_subscribe(task_id)` / `task_unsubscribe(task_id)` | Join or leave the room of a task. |
| `subscribe(task_id, opts)` | Join the task and progress rooms. When the task status changes, call `opts.success`, `opts.callback`, `opts.always` or `opts.error`. `frappe.call` uses it for queued calls. |

| Property | Description |
|---|---|
| `socket` | The Socket.IO client. |
| `disabled` | `true` when `disable_async` is set. |
| `open_docs` | Set of `"<doctype>:<name>"` keys of subscribed documents. |

Events that the client handles by default: `msgprint` (calls `frappe.msgprint`), `progress` (calls `frappe.show_progress`), `task_status_change` and `task_progress`. After a reconnect, the client does not join its DocType and document rooms again. Reload the page to subscribe again.

```js
frappe.realtime.on("sample_order_ready", (data) => {
	frappe.show_alert(__("Order {0} is ready", [data.order]));
});
```

```python
# server side
frappe.publish_realtime("sample_order_ready", {"order": "SO-0001"}, user="Administrator")
```

## Boot and session

`frappe.boot` is the object that `frappe/boot.py` (`get_bootinfo`) sends when desk loads. Apps add keys with the `boot_session` hook. These keys are used most:

| Key | Description |
|---|---|
| `user` | Current user: `name`, `email`, `roles`, `defaults`, `user_type`, `language`, `mute_sounds`, `default_workspace`, `user_permissions`, permission lists (`can_read`, `can_write`, `can_create`, `can_submit`, `can_cancel`, `can_delete`, `can_print`, `can_email`, `can_export`, `can_import`, `can_search`, `can_get_report`, `can_select`, `in_create`, `all_read`), `allow_modules`, `permitted_modules`, `impersonated_by`. |
| `user_info` | Map of user name to `{ fullname, image, name, email, time_zone }`. |
| `sysdefaults` | System defaults: `date_format`, `time_format`, `number_format`, `currency`, `currency_precision`, `float_precision`, `rounding_method`, `country`, `time_zone`, `first_day_of_the_week`, `setup_complete` and more. |
| `sitename` | Site name. |
| `lang`, `lang_dict` | Language code of the user. Map of language name to code. |
| `time_zone` | `{ system, user }` time zone names. |
| `server_date` | Server date as `yyyy-mm-dd`. |
| `versions` | Map of app name to version. |
| `app_data` | Installed apps with `app_name`, `app_title`, logo and route. |
| `single_types`, `nested_set_doctypes`, `tree_view_doctypes` | DocType name lists. |
| `module_app` | Map of scrubbed module name to app. |
| `max_file_size`, `file_chunk_size` | Upload limits in bytes. |
| `socketio_port` | Port of the Socket.IO server. |
| `doctype_layouts` | DocType Layout records (`name`, `document_type`) that the router turns into routes. |
| `assets_json` | Map of bundle name to built file. |
| `desk_settings`, `navbar_settings` | Desk and navbar settings. |
| `home_page` | Default page after login. |
| `read_only` | `true` when the site is in read-only mode. |
| `developer_mode` | Set when developer mode is on. |
| `disable_async` | Set when realtime is turned off. |

Globals that desk sets from the boot (`set_globals` in `frappe/public/js/frappe/desk.js`):

| Name | Value |
|---|---|
| `frappe.session.user` | `frappe.boot.user.name`. `"Guest"` for website visitors. |
| `frappe.session.logged_in_user` | Same as `user` at boot. |
| `frappe.session.user_email` | `frappe.boot.user.email`. |
| `frappe.session.user_fullname` | Full name from `frappe.user_info()`. |
| `frappe.user_roles` | `frappe.boot.user.roles`. |
| `frappe.user_defaults` | `frappe.boot.user.defaults`. |
| `frappe.sys_defaults` | `frappe.boot.sysdefaults`. |
| `frappe.ui.py_date_format` | System date format in Python `strftime` form, for example `%d-%m-%Y`. |

## User

`frappe.user` holds the current user name and helpers. `String(frappe.user)` returns the name, so `frappe.user == "Administrator"` works.

| Member | Description |
|---|---|
| `frappe.user.name` | Current user name. `"Guest"` before boot. |
| `frappe.user.has_role(rl)` | Return `true` if the user has the role, or one of a list of roles. Returns `undefined`, not `false`, when there is no match. |
| `frappe.user.is_report_manager()` | `has_role(["Administrator", "System Manager", "Report Manager"])`. |
| `frappe.user.full_name(uid)` | Full name of a user. Returns the translated "You" for the current user. |
| `frappe.user.image(uid)` | Image URL of a user, or `null`. |
| `frappe.user.abbr(uid)` | Initials of a user. |
| `frappe.user.get_formatted_email(email)` | `"Full Name <email>"`. The name is quoted if it has special characters. It looks up `email` as a user name. |
| `frappe.user.get_emails()` | Email addresses of all users in `frappe.boot.user_info`. |
| `frappe.user_info(uid)` | Return `{ fullname, image, name, email, time_zone, abbr, color }` for a user. Defaults to the current user. An unknown user gets `{ fullname: uid, abbr, color }`. |
| `frappe.update_user_info(user_info)` | Merge a map of user name to info into `frappe.boot.user_info`. |
| `frappe.session_alive` | `false` after 30 seconds without mouse movement. |

```js
if (frappe.user.has_role("Sample Role")) {
	frm.add_custom_button(__("Approve"), () => {});
}
frappe.user_info("Administrator").fullname; // "Administrator"
```

## Defaults

`frappe.defaults` reads defaults from the boot. User defaults are in `frappe.boot.user.defaults`. System defaults are in `frappe.sys_defaults`.

| Function | Description |
|---|---|
| `get_default(key)` | User default for `key`. Parses JSON values. For a DocType name key (for example `"Company"`), uses the single User Permission value, or the scrubbed key. Returns `undefined` if the value is not allowed by User Permissions. |
| `get_user_default(key)` | User default for `key`. For a DocType name key, prefers the default User Permission. Returns the first value of a list. Does not parse JSON. |
| `get_user_defaults(key)` | All allowed values for `key`, as a list. |
| `get_global_default(key)` | System default for `key`. Returns the first value of a list. |
| `get_global_defaults(key)` | System default for `key`, as a list. |
| `is_enabled(key)` | `true` if the system default is `1`. |
| `set_user_default_local(key, value)` | Set a user default in the browser only. Does not save it. |
| `get_user_permissions()` | User Permissions of the current user: `{ <doctype>: [{ doc, is_default, ... }] }`. |
| `update_user_permissions()` | Load User Permissions again from the server (`get_current_user_permissions`). |
| `load_user_permission_from_boot()` | Load User Permissions from `frappe.boot.user.user_permissions`, or from the server if missing. |
| `is_a_user_permission_key(key)` | `true` if `key` has no `:` and is not already scrubbed, for example `"Company"`. |
| `in_user_permission(key, value)` | `true` if `value` is allowed for the DocType `key`, or if there is no User Permission for it. |

```js
frappe.defaults.get_default("date_format");     // "dd-mm-yyyy" on a site with that format
frappe.defaults.get_global_default("currency"); // system currency
```

## Metadata

| Function | Description |
|---|---|
| `frappe.get_meta(doctype)` | DocType document from `locals.DocType`. Returns `undefined` until the DocType is loaded. Load it with `frappe.model.with_doctype` (see [Model and DB](model-and-db.md)). |

`frappe.meta` reads the fields of loaded DocTypes. `name` is a document name. With a name, the functions use a per-document copy of the fields, so property changes apply to one document only.

| Function | Description |
|---|---|
| `get_docfield(doctype, fieldname, name)` | Field definition, or `null`. |
| `get_field(doctype, fieldname, name)` | Like `get_docfield`, but also finds standard fields such as `owner` and `creation`. |
| `get_docfields(doctype, name, filters, docfield_list = null)` | Field definitions sorted by `idx`. `filters` is an object such as `{ fieldtype: "Link" }`. |
| `get_fieldnames(doctype, name, filters)` | Fieldnames that match `filters`. |
| `has_field(dt, fn)` | The field definition if it exists, else `undefined`. |
| `get_table_fields(dt, include_computed = false)` | Table fields. Virtual tables only with `include_computed`. |
| `get_parentfield(parent_dt, child_dt)` | Fieldname of the table field for `child_dt`. Throws if none. |
| `get_doctype_for_field(doctype, key)` | DocType (parent or child) that has the field `key`. |
| `get_linked_fields(doctype)` | `options` of all Link fields. |
| `get_label(dt, fn, dn)` | Label of a field. Standard fields have fixed labels, translated with `__()`, for example `owner` is "Created By". Returns `fn` if no label. |
| `get_translated_label(dt, fn, dn)` | Translated label, with `dt` as context. |
| `get_field_precision(df, doc)` | Precision of a field: `df.precision`, else for Currency the `currency_precision` default or the currency number format, else `float_precision` (default 3). |
| `get_field_currency(df, doc)` | Currency of a Currency field, from `df.options` and `doc`. Defaults to the system currency or `"USD"`. |
| `get_print_formats(doctype)` | Print format names available for the DocType. "Standard" first, the default print format before it. JS print formats, raw printing formats (unless enabled) and restricted formats are left out. Print formats with `print_format_for` = "Report" are not left out. |
| `get_restricted_print_formats(doctype)` | Print formats that the user cannot use because of their roles. |
| `get_print_sizes()` | Page sizes: `"A0"` to `"A9"`, `"B0"` to `"B10"`, `"Letter"`, `"Legal"`, `"Custom"` and others. |
| `set_formatter(doctype, fieldname, name, formatter)` | Set `df.formatter` on a field. |
| `set_indicator_formatter(doctype, fieldname, name, get_text, get_color)` | Set a formatter that shows an indicator pill. |
| `get_docfield_copy(doctype, name, docfield_list = null)` | Map of fieldname to field definition for one document. |
| `make_docfield_copy_for(doctype, docname, docfield_list = null)` | Make that per-document copy. |
| `sort_docfields(docs)` | Sort field definitions by `idx`. |
| `sync(doc)` | Add the fields of a DocType document to the maps. Called when a DocType loads. |
| `add_field(df)` | Add one field definition to the maps. |
| `get_fields_to_check_permissions(doctype)` | Link fields that check User Permissions, plus an ID entry. |

| Property | Description |
|---|---|
| `frappe.meta.docfield_map` | `{ doctype: { fieldname: df } }`. |
| `frappe.meta.docfield_list` | `{ doctype: [df] }`. |
| `frappe.meta.docfield_copy` | `{ doctype: { docname: { fieldname: df } } }`. |

```js
await frappe.model.with_doctype("Sample Order");
frappe.meta.get_label("Sample Order", "owner");                // "Created By"
frappe.meta.get_parentfield("Sample Order", "Sample Order Item"); // "items"
```

## Toolbar

`frappe.ui.toolbar` helpers. The v16 desk has no navbar with `#navbar-<name>` menus or a `.navbar-right` element, so the menu helpers find nothing.

| Function | Description |
|---|---|
| `clear_cache()` | Clear local storage caches, call `frappe.sessions.clear`, show the result and reload. Throttled to once every 10 seconds. Shortcut <kbd>Shift</kbd>+<kbd>Ctrl</kbd>+<kbd>R</kbd>. |
| `show_about()` | Open the About dialog. |
| `route_to_user()` | Open the User form of the current user. |
| `view_website()` | Open `/index` in a new tab. |
| `toggle_full_width()` | Toggle full-width mode. Saves it in `localStorage.container_fullwidth`. Triggers `toggleFullWidth` on `body`. |
| `set_fullwidth_if_enabled()` | Apply the saved full-width mode. |
| `show_shortcuts(e)` | Open the keyboard shortcuts dialog. |
| `fetch_session_defaults()` | Load Session Default Settings into `frappe.boot.session_defaults`. |
| `setup_session_defaults()` | Open the Session Defaults dialog. Saving it clears the cache. |
| `add_dropdown_button(parent, label, click, icon)` | Add an item to the menu `#navbar-<parent>`. |
| `get_menu(label)` | jQuery of `#navbar-<label>`. |
| `add_menu_divider(menu)` | Add a divider to a menu. |
| `add_icon_link(route, icon, index, class_name)` | Add an icon link to `.navbar-right`. Throws on v16 because the element does not exist. |

## Keyboard shortcuts

`frappe.ui.keys` binds keys on `window`. A key name is lowercase. Modifiers come first in the order `alt+`, `shift+`, `ctrl+`. On macOS, <kbd>Cmd</kbd> counts as `ctrl`. Examples: `"ctrl+s"`, `"shift+ctrl+r"`, `"alt+shift+ctrl+s"`, `"shift+/"`.

| Function | Description |
|---|---|
| `add_shortcut({ shortcut, action, description, page, target, condition, ignore_inputs = false })` | Add a shortcut and list it in the shortcuts dialog. Details below. |
| `on(key, handler)` | Add a raw handler `handler(e)`. It also runs while an input has focus. If it returns `false`, the default action is blocked. |
| `off(key, page)` | Remove the handlers of `key` that are attached to `page`. Without `page`, removes all handlers of `key`. |
| `get_key(e)` | Key name of a keyboard event, for example `"shift+ctrl+s"`. |
| `get_shortcut_label(shortcut)` | Display label. On macOS, `Ctrl+`, `Alt+` and `Shift+` become `⌘`, `⌥` and `⇧`. |
| `show_keyboard_shortcut_dialog()` | Open the shortcuts dialog. Shortcut <kbd>Shift</kbd>+<kbd>/</kbd>. |
| `get_shortcut_groups()` | Global, page and grid shortcut groups for the dialog. |
| `get_shortcut_group(parent)` | `AltShortcutGroup` of an object. Used for <kbd>Alt</kbd>+letter shortcuts in menus and the sidebar. |

| Property | Description |
|---|---|
| `handlers` | `{ key: [handler] }`. |
| `standard_shortcuts` | Shortcuts added with `add_shortcut`. |
| `key_map` | Map of key code to name: letters, `enter`, `escape`, `tab`, `space`, arrows, `f1` to `f5`, `/`, `<`, `>`, `shift`, `ctrl`, `alt`, `meta`, `backspace`. |
| `frappe.ui.keyCode` | Key codes: `ESCAPE`, `LEFT`, `RIGHT`, `UP`, `DOWN`, `ENTER`, `TAB`, `SPACE`, `BACKSPACE`. |

Options of `add_shortcut`:

| Option | Default | Description |
|---|---|---|
| `shortcut` | | Key name. |
| `action(e)` | | Runs on the key. If it returns nothing or a truthy value, the default browser action is blocked. |
| `description` | | Text in the shortcuts dialog. Without it, the shortcut is not listed. |
| `page` | | A page object (for example `frm.page` or `cur_page.page.page`). The shortcut runs only while `page.wrapper` is visible. |
| `target` | | A jQuery element. The shortcut clicks it. Replaces `action`. |
| `condition()` | `() => true` | The shortcut runs only when this returns a truthy value. |
| `ignore_inputs` | `false` | Also run while an input, select, textarea or editable element has focus. |

`add_shortcut` first calls `off(shortcut, page)`. A new shortcut with the same key and page replaces the old one. A global shortcut (no `page`) removes all other handlers of that key, including built-in ones.

Global shortcuts on v16:

| Key | Action |
|---|---|
| `ctrl+s` | Trigger the primary action (for example Save). |
| `ctrl+k` | Toggle the awesomebar. |
| `ctrl+g` | Open global search. |
| `shift+/` | Show keyboard shortcuts. |
| `shift+ctrl+r` | Clear cache and reload. |
| `shift+t` | Open the console (needs write access to System Console). |
| `ctrl+/` | Toggle the sidebar. |
| `shift+ctrl+g` | Switch the theme. |
| `shift+ctrl+/` | Open the app dock. |
| `escape` | Close the open grid row or dialog. |
| `enter` | Click the primary button of a confirm dialog. |
| `ctrl+up` / `ctrl+down` | Open the previous or next grid row. |

```js
frappe.ui.form.on("Sample Order", {
	refresh(frm) {
		frappe.ui.keys.add_shortcut({
			shortcut: "shift+ctrl+a",
			action: () => frm.trigger("approve"),
			description: __("Approve order"),
			page: frm.page,
		});
	},
});
```

## Other functions

| Function | Returns | Description |
|---|---|---|
| `frappe.render_template(name, data)` | string | Render a client template. `name` is a key of `frappe.templates` (built from `.html` files in the bundle), or a template string if it has a space. Throws if not found. |
| `frappe.render(str, data, name)` | string | Compile and render a template string. Supports `{{ expr }}`, `{% if %}`, `{% else %}`, `{% endif %}`, `{% for x in list %}`, `{% endfor %}` and plain JavaScript in `{% %}`. |
| `frappe.template.compile(str, name)` | function | Compile a template. Cached by `name` or by `str`. |
| `frappe.run_serially(tasks)` | Promise | Run functions one after the other. A function can return a Promise. Falsy entries are skipped. |
| `frappe.timeout(seconds)` | Promise | Resolve after `seconds`. |
| `frappe.load_image(src, onload, onerror, preprocess = () => {})` | | Load an image. `onload(img)` runs when it loads. |
| `frappe.markdown(txt)` | string | Convert Markdown to HTML with showdown. Removes tags and attributes that are not on an allow list (for example `javascript:` links). |
| `frappe.get_abbr(txt, max_length)` | string | Initials of the words, up to `max_length` (default 2). Returns `""` for empty input. |
| `frappe.get_palette(txt)` | array | `[bg_var, color_var]` CSS variable names, picked from a hash of `txt`. |
| `frappe.avatar(user, css_class, title, image_url = null, remove_color = false, filterable = false)` | string | Avatar HTML of a user. Default class `avatar-small`. |
| `frappe.get_avatar(css_class, title, image_url = null, remove_color, data_attributes)` | string | Avatar HTML from a title and an image. |
| `frappe.avatar_group(users, limit = 4, options = {})` | jQuery | Group of avatars. Extra users show as `+N`. `options`: `css_class`, `filterable`, `action_icon`, `action`, `align`, `overlap`. |
| `frappe.get_data_pill(label, target_id = null, remove_action = null, image = null, colored = false)` | jQuery | A pill button. `remove_action(target_id or label, $pill)` adds a remove icon. |
| `frappe.get_modal(title, content)` | jQuery | Bare Bootstrap modal markup. Use `frappe.ui.Dialog` instead. |
| `frappe.create_shadow_element(wrapper, html, css, js)` | | Render HTML, CSS and JS inside a shadow DOM in `wrapper`. Script and style tags are removed from `html`. |
| `frappe.get_cookie(name)` | string | Value of a cookie. |
| `frappe.get_cookies()` | object | All cookies that JavaScript can read. |
| `frappe.is_mobile()` | bool | `true` when the window is narrower than 768 px. |
| `frappe.is_large_screen()` | bool | `true` when the window is taller than 1180 px. |
| `frappe.is_online()` | bool | `navigator.onLine`. Always `true` in developer mode. |
| `frappe.preview_email(template, args, header, with_container = false, only_html = false)` | Promise | Render an email template on the server (`frappe.email.email_body.get_email_html`). Shows it in a dialog, or resolves with the HTML if `only_html`. |
| `frappe.render_pdf(html, opts = {})` | | Post `html` to `frappe.utils.print_format.report_to_pdf` and download the PDF. `opts`: `orientation`, `report_name` (file name). |
| `frappe.download_report_pdf(args, filename)` | | Post `args` to `frappe.utils.print_format.download_report_pdf` and download the PDF. |
| `frappe.get_module(m, default_module)` | object | Module definition from `frappe.modules`. |
| `frappe.printing.convert_to_builder(doc)` | | Convert a Print Format document to the new print format builder, after a confirm dialog, then open `print-format-builder-beta`. A standard format (outside developer mode) is copied first, with a prompt for the new name. |

## Pitfalls

- `frappe.call` rejects with the `jqXHR` object, not with an `Error`. Read `xhr.responseJSON` for details.
- `silent: true` hides only `_server_messages`. A 404 or 403 still shows a dialog.
- `quiet` has no effect.
- `debounce` stores all calls under one log key (`"undefined"`), because `args.cmd` is removed before the check. The check still compares the full args.
- `frappe.router.off(evt, handler)` does not remove the handler. It unbinds a new wrapper function. Use `once` or a flag in the handler.
- `frappe.realtime.off(event)` without a callback removes only one listener.
- `frappe.realtime.doc_subscribe` drops a second call within 1 second. Only the first document is subscribed.
- `frappe.realtime.doctype_unsubscribe` does not count callers. When one view leaves a DocType room, other open views of that DocType stop getting list updates.
- `frappe.ui.keys.off(key)` without `page` removes every handler of `key`, including built-in ones.
- `frappe.user.full_name()` returns "You" for the current user. `frappe.user.get_formatted_email()` uses it, so the current user gets `"You <...>"`.
- `frappe.user.has_role()` returns `undefined` instead of `false`.
- `frappe.meta.get_fields_to_check_permissions()` uses the global `window.name` as a fieldname. The ID entry has an empty `fieldname`.
- `frappe.get_meta()` returns `undefined` for a DocType that is not loaded yet.

## Related

- [Client API](../../developer-guide/desk-ui/client-api.md)
- [Form Scripts](../../developer-guide/desk-ui/form-scripts.md)
- [Desk Pages](../../developer-guide/desk-ui/desk-pages.md)
- [Realtime](../../developer-guide/runtime/realtime.md)
- [Whitelisted Methods](../../developer-guide/api/whitelisted-methods.md)
- [Utils and Formatting](utils-and-formatting.md)
- [Model and DB](model-and-db.md)
