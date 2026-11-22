---
title: List and Report Views (JavaScript)
description: Reference of the client registries that customize desk views - listview_settings, ListView members, query_reports, calendar, gantt, tree, kanban, map, dashboard chart sources and number cards.
version: develop
sources:
  - frappe/public/js/frappe/list/base_list.js
  - frappe/public/js/frappe/list/list_view.js
  - frappe/public/js/frappe/list/list_factory.js
  - frappe/public/js/frappe/list/list_view_select.js
  - frappe/public/js/frappe/model/indicator.js
  - frappe/public/js/frappe/views/reports/query_report.js
  - frappe/public/js/frappe/views/reports/report_view.js
  - frappe/public/js/frappe/views/reports/report_utils.js
  - frappe/public/js/frappe/views/calendar/calendar.js
  - frappe/public/js/frappe/views/gantt/gantt_view.js
  - frappe/public/js/frappe/views/treeview.js
  - frappe/public/js/frappe/views/kanban_v2/settings.js
  - frappe/public/js/frappe/views/kanban_v2/kanban_page.js
  - frappe/public/js/frappe/views/map/map_view.js
  - frappe/public/js/frappe/views/dashboard/dashboard_view.js
  - frappe/public/js/frappe/widgets/chart_widget.js
  - frappe/public/js/frappe/widgets/number_card_widget.js
  - frappe/desk/query_report.py
  - frappe/desk/calendar.py
  - frappe/desk/doctype/todo/todo_calendar.js
  - frappe/desk/doctype/event/event_calendar.js
  - frappe/desk/doctype/dashboard_chart_source/dashboard_chart_source.py
  - frappe/desk/doctype/number_card/number_card.json
  - frappe/utils/dashboard.py
---

# List and Report Views (JavaScript)

Each desk view reads a settings object from a global registry. An app fills the registry in a JavaScript file next to the DocType or the report. This page lists every key each view reads, with its arguments, and the view methods you can call from those keys. For where the files go, the hooks and worked examples, see [List View Settings](../../developer-guide/desk-ui/list-view-settings.md) and [Reports](../../developer-guide/output/reports.md).

| Registry | Filled by | Read by |
|---|---|---|
| `frappe.listview_settings[doctype]` | `<doctype>_list.js`, hook `doctype_list_js`, Client Script with `view` = `List` | List, Report, Gantt, Map, Kanban, Image and other list-based views, indicators |
| `frappe.query_reports[report_name]` | `<report_name>.js` in the report folder, or the `javascript` field of a Report | Query and Script Report view |
| `frappe.views.calendar[doctype]` | `<doctype>_calendar.js`, hook `doctype_calendar_js` | Calendar and Gantt views |
| `frappe.treeview_settings[doctype]` | `<doctype>_tree.js`, hook `doctype_tree_js` | Tree view |
| `frappe.kanban_v2.settings[doctype]` | `<doctype>_kanban.js`, hook `doctype_kanban_js` | Kanban boards on Kanban v2 |
| `frappe.dashboards.chart_sources[source]` | `<source>.js` in `<module>/dashboard_chart_source/<source>/` | Dashboard Charts with `chart_type` = `Custom` |

## listview_settings

`frappe.listview_settings[doctype]` is read when a view is set up, so changes after that need a new view (reload the page).

| Key | Type | Read by | Description |
|---|---|---|---|
| `add_fields` | list | list, report | Extra fields to fetch for each row. Add every field that `get_indicator`, `button` or `formatters` reads. |
| `filters` | list | list | Default filters, `[fieldname, operator, value]` or `[doctype, fieldname, operator, value]`. Used only when the user has no saved filters for the view. |
| `hide_name_column` | bool | list | Hide the "ID" column. That column shows only when the DocType has a `title_field` other than `name`. |
| `hide_name_filter` | bool | list | Hide the "ID" filter in the standard filter bar. |
| `custom_filter_configs` | list or function | list | Extra standard filter fields, as field definitions. A function can return a list or a Promise of one. |
| `get_indicator(doc)` | function | indicators | Return `[label, color, filter]`. `filter` is `"fieldname,operator,value"`, applied on click. Return nothing to fall through to the next rule. |
| `has_indicator_for_draft` | bool | indicators | Use `get_indicator` for drafts of a submittable DocType. |
| `has_indicator_for_cancelled` | bool | indicators | Use `get_indicator` for cancelled documents. |
| `formatters` | object | list | `{ fieldname: (value, df, doc) => html }`. On the subject (first) column, the result shows as text. |
| `button` | object | list | Row button: `show(doc)`, `get_label(doc)`, `get_description(doc)`, `action(doc)`. All four are required. |
| `dropdown_button` | object | list | Row dropdown: `get_label` (a string) and `buttons`, a list of `{ get_label, show(doc), get_description(doc), action(doc) }`. `get_label` of a button is a string. `show` and `get_description` are optional. |
| `onload(listview)` | function | list, report | Runs once when the view is set up, after the columns. |
| `before_render()` | function | list | Runs before each render of the rows. No arguments. |
| `refresh(listview)` | function | list, report | Runs after each data load and render. |
| `primary_action()` | function | list | Replaces the "Add" button action and the "Create" button of the empty state. |
| `get_form_link(doc)` | function | list | Return the URL a row opens. Default `/desk/<doctype-slug>/<name>`. |
| `reports` | list | view switcher | Extra reports in the switcher: `{ name, title, report_type, ref_doctype, route }`. |
| `get_coords_method` | any | view switcher | Shows "Map" in the view switcher. The Map view does not call it. |
| `gantt_custom_popup_html(task, doc)` | function | gantt | Return the HTML of the task popup. |

The indicator rules in `frappe/public/js/frappe/model/indicator.js` run in this order: unsaved, workflow state, draft and cancelled (submittable only), DocType states that match `doc.status`, `get_indicator`, submitted, `status`, `enabled`, `disabled`.

```js
// sample_app/sample_module/doctype/sample_order/sample_order_list.js
frappe.listview_settings["Sample Order"] = {
	add_fields: ["status", "total"],
	hide_name_filter: true,
	has_indicator_for_draft: true,
	get_indicator(doc) {
		const colors = { Draft: "gray", Open: "orange", Completed: "green" };
		return [__(doc.status), colors[doc.status] || "gray", "status,=," + doc.status];
	},
	formatters: {
		total: (value) => `<b>${format_currency(value)}</b>`,
	},
	dropdown_button: {
		get_label: __("More"),
		buttons: [{ get_label: __("Copy Name"), action: (doc) => frappe.utils.copy_to_clipboard(doc.name) }],
	},
	onload(listview) {
		listview.page.add_actions_menu_item(__("Mark Completed"), () =>
			listview.call_for_selected_items("sample_app.api.mark_completed")
		);
	},
};
```

## ListView members

`frappe.views.ListView` extends `frappe.views.BaseList`. The settings functions get the instance as `listview`. Other ways to reach it:

| Name | Description |
|---|---|
| `window.cur_list` | The list-based view on screen, or `null`. |
| `frappe.get_list_view(doctype)` | The List view of `doctype` made in this session (`frappe.views.list_view["List/<doctype>/List"]`). |
| `frappe.views.list_view[route]` | All list-based views made, by route string. |

| Member | Description |
|---|---|
| `doctype`, `meta`, `view_name` | DocType, its meta and the view name (`"List"`, `"Report"`, `"Kanban"`, ...). |
| `page` | The `frappe.ui.Page`. See [UI Components](ui-components.md#frappeuipage). |
| `data` | Rows of the current page. |
| `settings` | The `listview_settings` object, or `{}`. |
| `list_view_settings` | The **List View Settings** record (`disable_count`, `disable_auto_refresh`, ...). |
| `start`, `page_length` | Paging. `page_length` is 20, or 100 on a large screen. |
| `sort_by`, `sort_order` | Current sort. |
| `refresh()` | Load and render the data again. Returns a Promise. |
| `get_checked_items(only_docnames)` | Checked rows, or their names. |
| `clear_checked_items()` | Uncheck all rows. |
| `call_for_selected_items(method, args = {})` | `frappe.call(method)` with `args.names` set to the checked names, with freeze. Refreshes on success. |
| `get_filter_value(fieldname)` | Value of a filter. `like` values lose their `%`. |
| `get_args()` | Arguments of the `frappe.desk.reportview.get` call. |
| `make_new_doc()` | Open a new form. Copies `=` filters into the new document. |
| `filter_area` | Filter controller. See below. |

| `filter_area` method | Returns | Description |
|---|---|---|
| `get()` | list | All filters, standard filters included, as `[doctype, fieldname, operator, value]`. |
| `add(filters, refresh = true)` | Promise | Add filters (a list of 4-item lists, or one filter as arguments). Filters that exist are skipped. |
| `set(filters)` | Promise | Add filters without a refresh. |
| `remove(fieldname)` | Promise | Remove the filter of a field. |
| `remove_filters(filters)` | | Remove the filters of the fields in `filters`. |
| `clear(refresh = true)` | Promise | Remove all filters. |
| `exists(filter)` | bool | Check a filter. |

```js
await cur_list.filter_area.add([["Sample Order", "status", "=", "Draft"]]);
cur_list.get_filter_value("status"); // "Draft"
```

Other registries: `frappe.views.view_modes` lists the view names (`List`, `Report`, `Dashboard`, `Gantt`, `Kanban`, `Calendar`, `Image`, `Inbox`, `Tree`, `Map`). `frappe.views.is_valid(view_mode)` checks one.

### Which view needs what

| View | Route | Needs |
|---|---|---|
| Report | `/desk/<doctype>/view/report` | Reads `onload`, `refresh` and `add_fields` from `listview_settings`. |
| Image | `/desk/<doctype>/view/image` | `image_field` of the DocType. |
| Map | `/desk/<doctype>/view/map` | A Geolocation field named `location`, or fields `latitude` and `longitude`. |
| Calendar | `/desk/<doctype>/view/calendar/default` | `frappe.views.calendar[doctype]`, or `is_calendar_and_gantt` on the DocType. A `Calendar View` record adds a named calendar. |
| Gantt | `/desk/<doctype>/view/gantt` | `frappe.views.calendar[doctype]`, or `is_calendar_and_gantt` on the DocType. |
| Tree | `/desk/<doctype>/view/tree` | `frappe.treeview_settings[doctype]` or `is_tree` on the DocType. |
| Dashboard | `/desk/<doctype>/view/dashboard` | Dashboard Charts and Number Cards of the DocType. |

## query_reports

`frappe.query_reports[report_name]` configures a Query Report or Script Report. The report view loads it with `frappe.desk.query_report.get_script` (whitelisted). For the report files and the Python side, see [Reports](../../developer-guide/output/reports.md).

| Key | Type | Description |
|---|---|---|
| `filters` | list | Filter field definitions. See below. |
| `onload(report)` | function | Runs after the filters are made and route values set, before the first run. Add page buttons here. |
| `formatter(value, row, column, data, default_formatter, filter)` | function | Format a cell. Call `default_formatter(value, row, column, data)` for the standard result. |
| `get_datatable_options(options)` | function | Change the DataTable options. Return the options. |
| `after_datatable_render(datatable)` | function | Runs after each DataTable render. |
| `after_refresh(report)` | function | Runs after each run of the report, also on error. |
| `get_chart_data(columns, result)` | function | Return a chart config with `data.labels` and `data.datasets`. Replaces the `chart` from the server. |
| `tree` | bool | Rows form a tree. Sent to the server as `is_tree` (no total row for trees). |
| `parent_field` | string | Parent column of a tree. Sent to the server. |
| `initial_depth` | number | Tree depth to expand on render. `0` collapses all. |
| `collapsible_filters` | bool | Show one row of filters with a toggle for the rest. |
| `separate_check_filters` | bool | Put Check filters in their own row. |
| `export_hidden_cols` | bool | Offer hidden columns in the export dialog. |
| `get_pdf_format(report, bundled)` | function | Return the print template (string or Promise) for PDF and print. `bundled` is the `<report_name>.html` template. |

The view also sets these keys from the server: `html_format`, `has_python_formatter`, `execution_time`. If the Report has `validate_filters`, the server filters are merged into `filters`.

### Filter definitions

A filter is a field definition passed to `page.add_field`. These keys have a special meaning:

| Key | Description |
|---|---|
| `fieldname`, `label`, `fieldtype`, `options`, `default` | As for any field. `fieldtype` `"Break"` is skipped. |
| `reqd` or `mandatory` | The report does not run until the filter has a value. |
| `get_query()` | Query of a Link filter. |
| `on_change(report)` | Runs instead of the refresh when the value changes. Call `report.refresh()` yourself. |
| `depends_on` | `eval:` expression on the filter values (`doc`), a fieldname, or a boolean. A hidden filter is reset to its default. |
| `wildcard_filter` | Send the value as `%value%`. |
| `hidden` | Hide the filter. Its value is still sent. |

```js
// sample_app/sample_module/report/sample_order_summary/sample_order_summary.js
frappe.query_reports["Sample Order Summary"] = {
	filters: [
		{ fieldname: "status", label: __("Status"), fieldtype: "Select", options: "\nDraft\nOpen\nCompleted" },
		{ fieldname: "from_date", label: __("From Date"), fieldtype: "Date", reqd: 1, default: frappe.datetime.month_start() },
	],
	formatter(value, row, column, data, default_formatter) {
		value = default_formatter(value, row, column, data);
		return column.fieldname === "total" && data?.total > 1000 ? `<b>${value}</b>` : value;
	},
	onload(report) {
		report.page.add_inner_button(__("Open Orders"), () => report.set_filter_value("status", "Open"));
	},
};
```

### frappe.query_report

`frappe.query_report` is the `frappe.views.QueryReport` instance of the report page.

| Member | Description |
|---|---|
| `report_name`, `report_settings`, `report_doc` | Name, the `query_reports` object, the Report record. |
| `page` | The `frappe.ui.Page`. |
| `filters` | Filter controls. Each has `df`, `get_value()`, `set_value()`. |
| `columns`, `data` | Columns and rows of the last run. |
| `datatable` | The DataTable instance, or `null` before the first render. |
| `refresh()` | Run the report with the current filters. Throws `"Filter missing"` if a mandatory filter is empty. |
| `get_filter_values(raise)` | Object of filter values. Empty values are left out, unless the Report has `validate_filters`. With `raise`, missing mandatory filters show a message and throw. Also `get_values`. |
| `get_filter_value(fieldname, warn = true)` | Value of one filter. |
| `get_filter(fieldname, warn = true)` | Filter control. |
| `set_filter_value(fieldname, value)` or `set_filter_value({ fieldname: value })` | Set filter values. Each change runs the filter `on_change` or refreshes the report. |
| `toggle_filter_display(fieldname, flag)` | `true` hides the filter. |
| `get_checked_items(only_docnames)` | Checked rows. Needs `checkboxColumn: true` in the DataTable options. |
| `get_visible_columns()` | Columns not hidden by the user. |
| `expand_all_rows()`, `collapse_all_rows()` | Tree reports. |

## Calendar settings (frappe.views.calendar)

`frappe.views.calendar[doctype]` configures the Calendar view and, with `gantt`, the Gantt view.

| Key | Description |
|---|---|
| `field_map` | Maps `start`, `end`, `id`, `title`, `allDay`, `convertToUserTz` to fieldnames. Default `{ id: "name", start: "start", end: "end", allDay: "all_day", convertToUserTz: "convert_to_user_tz" }`. |
| `get_events_method` | Whitelisted method for events. Default `frappe.desk.calendar.get_events`. It gets `doctype`, `start`, `end`, `field_map`, `filters`, `fields`, `order_by`. |
| `update_event_method` | Whitelisted method called on drag or resize. Default `frappe.desk.calendar.update_event` (gets `args`, `field_map`, saves the document). |
| `fields` | Fields to fetch. The `field_map` start, end and title and `name` are added. |
| `get_css_class(event)` | Return a color: `danger`, `success`, `warning`, `default`, a color name (`red`, `green`, ...) or a hex. Without it, a Color field of the DocType colors the event. |
| `options` | FullCalendar options, merged last. |
| `apply_to_custom_calendar_views` | Also apply `fields` and `get_css_class` to `Calendar View` records. |
| `gantt` | `true` shows the Gantt view. An object also overrides keys in the Gantt view. |
| `order_by` | Gantt sort field (ascending). Default is the user sort, else `field_map.start`. |

The Gantt `field_map` adds `progress` (a fieldname or a function `(doc) => number`), `color`, `is_milestone` and `depends_on`. Defaults: `progress`, `color`, `is_milestone`, `depends_on_tasks`.

```js
// sample_app/sample_module/doctype/sample_order/sample_order_calendar.js
frappe.views.calendar["Sample Order"] = {
	field_map: { start: "order_date", end: "order_date", id: "name", title: "customer_name", allDay: "all_day" },
	get_css_class: (event) => (event.status === "Completed" ? "success" : "default"),
	gantt: true,
};
```

## Tree view (frappe.treeview_settings)

`frappe.treeview_settings[doctype]` is merged into the options of `frappe.views.TreeView`. The DocType needs an `is_group` field or settings. `window.cur_tree` is the `frappe.ui.Tree` on screen.

| Key | Default | Description |
|---|---|---|
| `get_tree_nodes` | `frappe.desk.treeview.get_children` | Method that returns child nodes. |
| `add_tree_node` | `frappe.desk.treeview.add_node` | Method that adds a node. |
| `get_tree_root` | `true` | Ask `get_tree_nodes` for the root. `false` skips it. |
| `root_label` | | Root label. Also the filter key whose value replaces the root. |
| `title` | `"<DocType> Tree"` | Page title. |
| `filters` | | Filter fields: field definitions with `onchange`, `disable_onchange`. A value is sent to the methods as an argument and becomes the root label. |
| `fields`, `ignore_fields` | | Fields of the "Add Child" dialog. Default: mandatory fields of the DocType. |
| `disable_add_node` | | Hide "Add Child" and the primary button. |
| `toolbar` | | Node actions: `{ label, icon, click(node), condition(node), get_disabled_reason(node), danger, btnClass }`. |
| `extend_toolbar` | | `true` adds `toolbar` to the standard actions (same label replaces). Without it, `toolbar` replaces them. |
| `menu_items` | | Page menu items: `{ label, action, condition }`. `condition` is a function or a string. |
| `onload(treeview)` | | Runs after setup. |
| `post_render(treeview)` | | Runs after the tree renders. |
| `onrender(node)`, `on_node_render(node, deep)`, `on_get_node(nodes)` | | Node hooks. |
| `click(node)` | | Runs on node click. |
| `get_label(node)` | | Node label. |
| `view_template` | | Template name to render `node.data` beside the tree. |
| `show_expand_all` | `true` | Show "Expand All". |
| `use_row_actions`, `row_style` | | Passed to `frappe.ui.Tree`. |
| `do_not_make_page`, `page` | | Render inside another page. |
| `do_not_setup_menu` | | No standard menu items. |

## Kanban settings (frappe.kanban_v2.settings)

Applies to Kanban boards that use Kanban v2. Keys:

| Key | Description |
|---|---|
| `card_context_menu(card, page)` | Return extra card menu items (`frappe.ui.Dropdown` items). They show before "Delete". |
| `bulk_actions(ids, page)` | Return extra buttons for selected cards: `{ label, icon, theme, variant, condition(), onclick(ids, page, done) }`. |
| `callbacks` | Engine callbacks: `canMoveCard`, `onBeforeCardMove` (can be async, `false` cancels the drag), `onCardMove`, `onAfterCardMove`, `onCardClick(card, ev)`, `onCardOpen(card)`, `onSelectionChange(ids)`, `onMoveError(move)`, `onAddCard(columnId)`. |
| `select_styles` | `{ value: theme }` or `{ value: { theme, icon } }` for Select values on cards. |
| `open_on_title_click` | `false` stops the card title opening the document. |
| `renderCard(card, el, ctx, page)`, `renderColumnHeader`, `renderEmptyState` | Custom renderers. |
| `options` | Extra engine options. |
| `boards` | `{ board_name: settings }` overrides for one board. |

`frappe.kanban_v2.extend_settings(doctype, patch)` changes the settings without losing the base. `patch` is an object (menu and bulk items append, maps merge) or a function `(base) => settings`.

## Dashboard chart sources and number cards

A Dashboard Chart with `chart_type` = `Custom` uses a **Dashboard Chart Source**. Its config file is `<module>/dashboard_chart_source/<source>/<source>.js`, served by `frappe.desk.doctype.dashboard_chart_source.dashboard_chart_source.get_config`.

| Key of `frappe.dashboards.chart_sources[source]` | Description |
|---|---|
| `method` | Whitelisted method that returns the chart data. It gets `chart_name`, `filters`, `refresh`, `time_interval`, `timespan`, `from_date`, `to_date`, `heatmap_year`. |
| `filters` | Filter field definitions for the chart filter dialog. |

```js
// sample_app/sample_module/dashboard_chart_source/sample_orders_by_status/sample_orders_by_status.js
frappe.provide("frappe.dashboards.chart_sources");
frappe.dashboards.chart_sources["Sample Orders by Status"] = {
	method: "sample_app.sample_module.dashboard_chart_source.sample_orders_by_status.sample_orders_by_status.get",
	filters: [{ fieldname: "status", label: __("Status"), fieldtype: "Select", options: "\nDraft\nOpen\nCompleted" }],
};
```

```python
# sample_app/sample_module/dashboard_chart_source/sample_orders_by_status/sample_orders_by_status.py
import frappe
from frappe.utils.dashboard import cache_source


@frappe.whitelist()
@cache_source
def get(chart_name=None, chart=None, no_cache=None, filters=None, from_date=None, to_date=None,
		timespan=None, time_interval=None, heatmap_year=None):
	rows = frappe.get_list("Sample Order", fields=["status", {"COUNT": "*", "as": "count"}], group_by="status")
	return {
		"labels": [r.status for r in rows],
		"datasets": [{"name": "Orders", "values": [r.count for r in rows]}],
		"type": "bar",
	}
```

`cache_source` caches the result in `chart-data:<chart name>` until a refresh.

A **Number Card** with `type` = `Custom` calls its `method` field (a whitelisted dotted path) with `filters`. The method returns a number, or `{ value, fieldtype, route_options, route }`.

## Pitfalls

- `listview_settings` from a file applies only to DocTypes in an app. For a custom DocType, use a Client Script with `view` = `List`.
- A field that `get_indicator`, `button` or `formatters` reads is `undefined` unless it is a column or in `add_fields`.
- `dropdown_button.get_label` and each button `get_label` are strings. A function shows as its source text.
- `filters` in `frappe.views.calendar` is not read. Neither is `style_map`.
- `get_coords_method` only adds "Map" to the view switcher. The Map view needs `location` or `latitude` and `longitude` fields.
- `set_filter_value` with an object can refresh the report once per changed filter.
- `frappe.query_report.get_checked_items()` throws before the first render, because `datatable` is `null`.
- Query report charts from `get_chart_data` need `data.labels`. Without labels no chart is drawn.

## Related

- [List View Settings](../../developer-guide/desk-ui/list-view-settings.md)
- [Reports](../../developer-guide/output/reports.md)
- [UI Components](ui-components.md)
- [frappe (JavaScript)](frappe.md)
- [Hooks reference](../configuration/hooks.md)
- [Views](../../user-guide/basics/views.md)
