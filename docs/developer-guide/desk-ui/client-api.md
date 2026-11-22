---
title: Client API
description: Call the server, read data, show messages and dialogs, route, and format values from desk JavaScript with frappe.call, frappe.db, frappe.model, frappe.ui.Dialog and related helpers.
version: develop
sources:
  - frappe/public/js/frappe/request.js
  - frappe/public/js/frappe/db.js
  - frappe/public/js/frappe/model/model.js
  - frappe/public/js/frappe/model/create_new.js
  - frappe/public/js/frappe/ui/messages.js
  - frappe/public/js/frappe/ui/dialog.js
  - frappe/public/js/frappe/ui/field_group.js
  - frappe/public/js/frappe/router.js
  - frappe/public/js/frappe/form/formatters.js
  - frappe/public/js/frappe/utils/utils.js
  - frappe/public/js/frappe/utils/datetime.js
  - frappe/sessions.py
  - frappe/boot.py
---

# Client API

The desk loads a global `frappe` object in the browser. Form scripts, list settings, desk pages and Client Scripts use it to call the server, read documents, show messages and move between pages. All examples on this page were run in the desk browser console as Administrator. For a full list of functions, see the [JavaScript API reference](../../reference/api/js-api.md).

## Call the server

### frappe.call

`frappe.call(opts)` sends a request to `/api/method/<method>`. The method must be whitelisted. See [Whitelisted Methods](../api/whitelisted-methods.md).

```js
const r = await frappe.call({
	method: "frappe.client.get_value",
	args: { doctype: "Sample Item", filters: {}, fieldname: "item_name" },
});
console.log(r.message); // { item_name: "Antivirus" }
```

The short form is `frappe.call(method, args, callback)`. The return value is a jQuery promise that resolves with the full response object. The return value of the Python method is in `r.message`.

| Option | Type | Default | Description |
|---|---|---|---|
| `method` | string | | Dotted path of the whitelisted method. |
| `args` | object | `{}` | Arguments for the method. |
| `type` | string | `"POST"` | HTTP method. |
| `callback(r)` | function | | Runs on success. |
| `error(r)` | function | | Runs on failure. |
| `always(r)` | function | | Runs after success or failure. |
| `freeze` | bool | `false` | Block the page until the response comes back. |
| `freeze_message` | string | | Text shown while frozen. |
| `btn` | element | | Disable this button while the request runs. |
| `doc` | object | | Run a whitelisted controller method on this document (`run_doc_method`). |
| `async` | bool | `true` | Set `false` for a blocking request. Avoid it. |
| `headers` | object | `{}` | Extra HTTP headers. |
| `quiet` | bool | `false` | Do not show the request spinner. |
| `silent` | bool | `false` | Do not show error messages. |
| `api_version` | string | | Call `/api/<version>/method/<method>`, for example `"v2"`. |
| `url` | string | | Call this URL instead of `/api/method/<method>`. |
| `cache` | bool | | Let the browser cache a GET response. |

!!! note
    In develop, apps listed in `frappe.boot.json_request_apps` get a JSON request body. Other apps get form-encoded arguments. Set `json: true` or `json: false` to force one.

### frappe.xcall

`frappe.xcall(method, args, type)` is a shorter promise form. It resolves with `r.message` and rejects on error. The default type is `POST`.

```js
const count = await frappe.xcall("frappe.client.get_count", { doctype: "Sample Order" });
```

Use `frm.call()` in form scripts to call a controller method of the open document. See [Form Scripts](form-scripts.md).

## Read and write data: frappe.db

`frappe.db` wraps the `frappe.client` and `frappe.desk.reportview` methods. All functions return promises. The server checks permissions for each call.

| Function | Resolves with |
|---|---|
| `get_list(doctype, {fields, filters, order_by, limit, ...})` | List of rows. Default `fields: ["name"]`, `limit: 20`. |
| `get_value(doctype, name_or_filters, fieldname_or_list)` | The full response. The values are in `r.message`. |
| `get_single_value(doctype, field)` | The value of a field of a Single DocType. |
| `get_doc(doctype, name)` | The document. Also adds it to `locals`. |
| `exists(doctype, name_or_filters)` | `true` or `false`. |
| `count(doctype, {filters})` | A number. |
| `set_value(doctype, name, fieldname_or_object, value)` | The full response with the updated document in `r.message`. |
| `insert(doc)` | The inserted document. `doc` must have a `doctype` key. |
| `delete_doc(doctype, name)` | The server response. |
| `get_link_options(doctype, txt, filters)` | Link search results. |

```js
const orders = await frappe.db.get_list("Sample Order", {
	fields: ["name", "status"],
	filters: { status: "Open" },
	limit: 2,
});
// [{ name: "SO-00007", status: "Open" }, { name: "SO-00005", status: "Open" }]

const { message } = await frappe.db.get_value("Sample Order", "SO-00001", ["customer_name", "status"]);
// message = { customer_name: "Customer A", status: "Open" }

await frappe.db.count("Sample Order", { filters: { status: "Draft" } }); // 2
await frappe.db.exists("Sample Order", { customer_name: "Customer A" }); // true
```

!!! warning
    `frappe.db.get_value` and `frappe.db.set_value` resolve with the response object, not with the value. Read `r.message`.

## Documents in the browser: frappe.model

Loaded documents live in the global `locals[doctype][name]`. `frappe.model` works on these local copies.

| Function | Description |
|---|---|
| `frappe.get_doc(doctype, name)` | Get a loaded document from `locals`. |
| `frappe.model.with_doc(doctype, name)` | Load a document from the server if needed. Returns a promise. |
| `frappe.model.with_doctype(doctype)` | Load the meta of a DocType. Returns a promise. |
| `frappe.get_meta(doctype)` | Get a loaded meta. |
| `frappe.model.set_value(doctype, name, fieldname, value)` | Set a value on a local document and run form events. Use it for child rows. |
| `frappe.model.add_child(parent_doc, child_doctype, parentfield)` | Add a row to a local document. |
| `frappe.model.get_new_doc(doctype)` | Make a new local document with default values. |
| `frappe.new_doc(doctype, values)` | Open a new form (or quick entry dialog) with these values. |
| `frappe.model.sync(docs)` | Put documents from a server response into `locals`. |

## Messages

| Function | Description |
|---|---|
| `frappe.msgprint(message, title)` | Show a message dialog. |
| `frappe.msgprint({message, title, indicator, primary_action, wide})` | The same, with options. `indicator` is a color, for example `"red"`. |
| `frappe.throw(message)` | Show a red message and throw an `Error`. Stops the code that called it, for example a `validate` handler. |
| `frappe.show_alert({message, indicator}, seconds)` | Show a toast. Default 7 seconds. `frappe.toast` is the same function. |
| `frappe.confirm(message, on_yes, on_no)` | Ask Yes or No. Returns the dialog. |
| `frappe.warn(title, message, on_proceed, primary_label)` | A confirm dialog with a red primary button, for destructive actions. |
| `frappe.prompt(fields, callback, title, primary_label)` | Ask for values. `fields` is a label string, one field or a list of fields. |
| `frappe.show_progress(title, count, total, description)` | Show a progress bar dialog. `frappe.hide_progress()` closes it. |

```js
frappe.confirm(__("Mark all open orders as completed?"), () => {
	frappe.xcall("sample_app.api.complete_open_orders").then(() =>
		frappe.show_alert({ message: __("Done"), indicator: "green" })
	);
});

frappe.prompt(
	{ label: __("Customer"), fieldname: "customer_name", fieldtype: "Data", reqd: 1 },
	(values) => frappe.new_doc("Sample Order", values),
	__("New Order")
);
```

## Dialogs: frappe.ui.Dialog

`frappe.ui.Dialog` shows a modal with form fields. Fields use the same definitions as DocType fields: `fieldname`, `fieldtype`, `label`, `options`, `reqd`, `default`, `depends_on`, `read_only`, `description`.

```js
const d = new frappe.ui.Dialog({
	title: __("Enter details"),
	fields: [
		{ label: __("Customer"), fieldname: "customer_name", fieldtype: "Data", reqd: 1 },
		{ label: __("Date"), fieldname: "order_date", fieldtype: "Date", default: "Today" },
	],
	primary_action_label: __("Create"),
	primary_action(values) {
		// values = { customer_name: "Customer A", order_date: "2026-11-22" }
		frappe.new_doc("Sample Order", values);
		d.hide();
	},
});
d.show();
```

`primary_action` gets the values only if all mandatory fields have values. If it returns a promise, the button shows a busy state until the promise settles.

| Option | Description |
|---|---|
| `title` | Dialog title. |
| `fields` | List of field definitions. |
| `size` | `"small"`, `"large"` or `"extra-large"`. |
| `primary_action_label`, `primary_action(values)` | The main button. |
| `secondary_action_label`, `secondary_action()` | The second button. |
| `static` | Do not close on outside click or <kbd>Esc</kbd>. |
| `minimizable` | Let the user minimize the dialog. |
| `indicator` | Color of the title indicator. |

Methods: `show()`, `hide()`, `get_values()`, `get_value(fieldname)`, `set_value(fieldname, value)`, `set_values(object)`, `get_field(fieldname)`, `set_df_property(fieldname, property, value)`, `set_query(fieldname, fn)`, `set_primary_action(label, fn)`, `disable_primary_action()`, `enable_primary_action()`, `set_title(text)`, `set_message(text)`, `clear()`. Set `d.onhide = () => {}` to run code when the dialog closes.

## Routing

| Function | Description |
|---|---|
| `frappe.set_route(...parts)` | Go to a desk route. Returns a promise. |
| `frappe.get_route()` | The current route as a list, for example `["Form", "Sample Order", "SO-00001"]`. |
| `frappe.get_route_str()` | The current route as a string. |
| `frappe.route_options` | Values for the next view. A list view uses them as filters. A new form uses them as field values. |
| `frappe.get_prev_route()` | The previous route. |

```js
frappe.set_route("List", "Sample Order");
frappe.set_route("Form", "Sample Order", "SO-00001");
frappe.set_route("sample-order/new");

frappe.route_options = { status: "Open" };
frappe.set_route("List", "Sample Order"); // the list opens with status = Open
```

`frappe.set_route` accepts separate parts, a list or one string with slashes. `List`, `Form`, `Tree` and `Report` are converted to URL routes, for example `/desk/sample-order/SO-00001`. The desk can add the sidebar name to the URL. Do not build desk URLs by hand. Use `frappe.utils.get_form_link(doctype, name)` for a link.

## Formatting and utilities

```js
frappe.format(1234.5, { fieldtype: "Currency" });   // HTML with the formatted amount
frappe.format("2026-01-15", { fieldtype: "Date" });  // "15-01-2026" (user date format)
flt("12.3456", 2);                                   // 12.35
cint("7x");                                          // 7
frappe.datetime.get_today();                         // "2026-11-22"
frappe.datetime.add_days("2026-01-30", 3);           // "2026-02-02"
```

| Function | Description |
|---|---|
| `frappe.format(value, df, options, doc)` | Format a value as the desk shows a field of type `df.fieldtype`. Returns HTML. |
| `__(text, args)` | Translate a string. See [Translations](../building-apps/translations.md). |
| `flt(value, precision)`, `cint(value)`, `cstr(value)` | Convert to float, integer or string. |
| `frappe.datetime.get_today()`, `nowdate()`, `now_datetime()` | Today and now in system format. |
| `frappe.datetime.add_days(date, n)`, `add_months(date, n)`, `get_diff(d1, d2)` | Date math. |
| `frappe.datetime.str_to_user(value)`, `user_to_str(value)` | Convert between system and user date formats. |
| `frappe.utils.escape_html(text)` | Escape HTML. Use it for user data that you put into HTML. |
| `frappe.utils.debounce(fn, wait)`, `throttle(fn, wait)` | Limit how often a function runs. |
| `frappe.utils.copy_to_clipboard(text)` | Copy text and show an alert. |
| `frappe.utils.get_form_link(doctype, name)` | URL of a form. |
| `frappe.require(assets)` | Load JS or CSS bundles on demand. Returns a promise. |
| `frappe.run_serially(functions)` | Run functions one after the other. Waits for returned promises. |

## Session and boot

| Object | Description |
|---|---|
| `frappe.session.user` | The logged-in user, for example `"Administrator"`. |
| `frappe.session.user_email`, `frappe.session.user_fullname` | Email and full name. |
| `frappe.user.has_role(role)` | Check if the user has a role. |
| `frappe.boot` | Data loaded once at desk start: `sysdefaults`, `user`, `lang`, `time_zone`, `module_app`, `workspaces`, `dock`, `ui_islands` and more. |
| `frappe.sys_defaults` | System defaults such as date and number formats. |

To add your own keys to `frappe.boot`, use the `boot_session` hook. See [Extending Apps](../building-apps/extending-apps.md). Boot data is loaded once, so it is stale until the user reloads the page.

## Pitfalls

- Do not put permission checks only in the browser. The server must check them.
- `frappe.call` with `async: false` blocks the page. Use promises.
- `frappe.throw` stops your code with an exception. Catch it if the code must continue.
- `locals` holds only documents that the browser loaded. Use `frappe.model.with_doc` or `frappe.db.get_doc` before `frappe.get_doc`.
- Escape user data with `frappe.utils.escape_html` before you put it into HTML.

## Related

- [Form Scripts](form-scripts.md)
- [List View Settings](list-view-settings.md)
- [Desk Pages](desk-pages.md)
- [Whitelisted Methods](../api/whitelisted-methods.md)
- [Realtime](../runtime/realtime.md)
- [JavaScript API reference](../../reference/api/js-api.md)
