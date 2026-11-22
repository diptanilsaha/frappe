---
title: Desk Pages
description: Build a custom desk screen with the Page DocType, its JSON, JS, CSS and HTML files, frappe.pages events and frappe.ui.make_app_page.
version: v16
sources:
  - frappe/core/doctype/page/page.py
  - frappe/core/doctype/page/page.json
  - frappe/desk/desk_page.py
  - frappe/public/js/frappe/views/pageview.js
  - frappe/public/js/frappe/views/container.js
  - frappe/public/js/frappe/ui/page.js
  - frappe/public/js/frappe/microtemplate.js
  - frappe/bundler.py
  - frappe/desk/page/backups/backups.js
  - frappe/desk/page/backups/backups.json
  - frappe/utils/boilerplate.py
  - frappe/hooks.py
---

# Desk Pages

A desk page is a custom screen in the desk that is not a form, list or report. Examples in Frappe are `backups` (Download Backups) and `profile`. Use a page for a dashboard, a tool or a board that needs its own layout. A page is a record of the `Page` DocType plus files in your app. For a page drawn by a Vue component, see [Islands](islands.md).

## Page files

A standard page lives in the `page` folder of a module:

```text
sample_app/sample_module/page/order_board/
├── __init__.py
├── order_board.json     # the Page record
├── order_board.js       # page script
├── order_board.css      # optional styles
├── order_board.html     # optional templates
└── order_board.py       # optional whitelisted methods
```

The folder and file names are the scrubbed page name (`order-board` becomes `order_board`). `bench migrate` imports the JSON into the `Page` table.

## Create a page

1. Turn on developer mode for the site. A new Page needs it.
2. Log in as Administrator. Only Administrator can create or edit a Page.
3. Go to `/desk/page/new`.
4. Set **Page Name**, **Title** and **Module**. Keep **Standard** = `Yes`.
5. Add the roles that can open the page in **Roles**. With no roles, all desk users can open it.
6. Save.

On save, Frappe writes `<page_name>.json` and a starter `<page_name>.js` into the module folder. The record name comes from **Page Name**: lowercase, spaces become hyphens, and it is cut to 20 characters. The page opens at `/desk/<name>`.

## The page script

The starter script runs when the page loads for the first time:

```js
// sample_app/sample_module/page/order_board/order_board.js
frappe.pages["order-board"].on_page_load = function (wrapper) {
	const page = frappe.ui.make_app_page({
		parent: wrapper,
		title: __("Order Board"),
		single_column: true,
	});

	page.set_primary_action(__("New Order"), () => frappe.new_doc("Sample Order"), "add");
	page.add_inner_button(__("Refresh"), () => load_orders(page));

	page.status_field = page.add_field({
		label: __("Status"),
		fieldname: "status",
		fieldtype: "Select",
		options: ["", "Draft", "Open", "Completed"],
		change: () => load_orders(page),
	});

	$(page.body).html(`<div class="order-board"></div>`);
	load_orders(page);
};

frappe.pages["order-board"].on_page_show = function (wrapper) {
	load_orders(wrapper.page);
};

function load_orders(page) {
	frappe
		.xcall("sample_app.sample_module.page.order_board.order_board.get_orders", {
			status: page.status_field.get_value(),
		})
		.then((orders) => {
			$(page.body).find(".order-board").html(
				frappe.render_template("order_board", { orders })
			);
		});
}
```

`frappe.pages[name]` is the page wrapper element. The desk calls these functions on it:

| Function | When it runs |
|---|---|
| `on_page_load(wrapper)` | Once, after the page script is loaded. |
| `on_page_show(wrapper)` | Each time the page is shown, also the first time. |
| `refresh(wrapper)` | Each time the page is shown, after `on_page_show`. |

`frappe.ui.make_app_page(opts)` makes a `frappe.ui.Page` and stores it as `wrapper.page`. Options: `parent` (required), `title`, `single_column`. A `frappe.ui.Page` has these parts and methods:

| Member | Description |
|---|---|
| `page.body`, `page.main` | The main content area (a jQuery object). |
| `page.sidebar` | The side section, used when `single_column` is not set. |
| `set_title(title)`, `set_indicator(label, color)`, `clear_indicator()` | Title and status indicator. |
| `set_primary_action(label, fn, icon)`, `set_secondary_action(label, fn, icon)` | The main buttons. |
| `add_inner_button(label, fn, group)` | A toolbar button, or a dropdown item if `group` is set. |
| `add_menu_item(label, fn)` | An item in the "..." menu. |
| `add_action_item(label, fn)` | An item in the Actions menu. |
| `add_field(df)` | A filter control in the page form bar. Returns the control. `page.fields_dict` holds them. |
| `clear_primary_action()`, `clear_inner_toolbar()`, `clear_menu()` | Remove buttons. |

## Server methods

Put whitelisted functions in `<page_name>.py` and call them by dotted path:

```python
# sample_app/sample_module/page/order_board/order_board.py
import frappe


@frappe.whitelist()
def get_orders(status: str | None = None):
	filters = {"status": status} if status else {}
	return frappe.get_list("Sample Order", filters=filters, fields=["name", "customer_name", "status"])
```

The Page **Roles** control only who can open the page. They do not protect these methods. Use `frappe.get_list` or an explicit permission check in each method. See [Whitelisted Methods](../api/whitelisted-methods.md).

## CSS and HTML templates

- `<page_name>.css` is loaded with the page.
- Each `.html` file in the folder becomes `frappe.templates["<file name>"]`. Render it with `frappe.render_template("<file name>", data)`. The syntax is the desk micro-template: `{{ value }}`, `{% if %}`, `{% for item in list %}` and `{% endfor %}`, or plain JavaScript inside `{% %}`.
- If an `.html` file contains `<!-- jinja -->`, the server renders it with Jinja first. The context comes from `get_context(context)` in `<page_name>.py`. Such a page is not cached in the browser.

```html
<!-- sample_app/sample_module/page/order_board/order_board.html -->
{% for order in orders %}
	<div class="order-row">{{ order.customer_name }}: {{ order.status }}</div>
{% endfor %}
```

## Add code to another app's page

The `page_js` hook adds a script to a page that another app owns. The script runs after the page script.

```python
# sample_app/hooks.py
page_js = {"backups": "public/js/backups_extra.js"}
```

## Page fields

| Field | Type | Description |
|---|---|---|
| `page_name` | Data | Name used for the record name and route. |
| `title` | Data | Title in the breadcrumb and browser tab. |
| `module` | Link | Module that holds the page folder. |
| `standard` | Select | `Yes` for a page shipped in an app. |
| `roles` | Table (Has Role) | Roles that can open the page. Empty means all desk users. |
| `type` | Select | Empty for a script page, `Frappe UI` for an island page. |
| `island` | Data | For `Frappe UI` pages: the name of an island the app builds. Empty means the island is built from the page folder. |
| `system_page` | Check | The page opens with no sidebar. |
| `shared_page` | Check | The page opens in the sidebar that is on screen, instead of the sidebar of its module. Cannot be set with `system_page`. |

## Pitfalls

- Outside developer mode, the browser caches the page script in `localStorage`. Reload after a change. Run `bench --site <site> clear-cache` if it does not update.
- A Page can be deleted only in developer mode. In developer mode, deleting it also deletes its folder.
- The `frappe.pages[...]` key is the record name with hyphens, not the scrubbed folder name.
- A `Frappe UI` page does not run `<page_name>.js`. See [Islands](islands.md).

## Related

- [Islands](islands.md)
- [Client API](client-api.md)
- [Workspaces, Sidebar and Dock](workspaces-sidebar-dock.md)
- [Whitelisted Methods](../api/whitelisted-methods.md)
- [Apps](../building-apps/apps.md)
