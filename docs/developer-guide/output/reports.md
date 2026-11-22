---
title: Reports
description: Build Script Reports and Query Reports in an app, define filters, return charts and summaries, run reports from code, and use prepared reports and report roles.
version: v16
sources:
  - frappe/desk/query_report.py
  - frappe/core/doctype/report/report.py
  - frappe/core/doctype/report/report.json
  - frappe/core/doctype/report/boilerplate/controller.py
  - frappe/core/doctype/report/boilerplate/controller.js
  - frappe/core/doctype/report_filter/report_filter.json
  - frappe/core/doctype/prepared_report/prepared_report.py
  - frappe/core/report/database_storage_usage_by_tables/database_storage_usage_by_tables.py
  - frappe/core/report/database_storage_usage_by_tables/database_storage_usage_by_tables.js
  - frappe/website/report/website_analytics/website_analytics.py
  - frappe/public/js/frappe/views/reports/query_report.js
  - frappe/public/js/frappe/utils/utils.js
---

# Reports

A report is a `Report` document. Its `report_type` decides how it gets data. Use a Script Report when you need Python logic, joins or computed columns. Use a Query Report for one SQL query. Report Builder and Custom Report are saved views that users make in the desk. For the desk side, see [Reports (user guide)](../../user-guide/data-and-reports/reports.md).

| `report_type` | Data comes from | Made by |
|---|---|---|
| `Report Builder` | A saved list view of `ref_doctype` (`json` field) | Users, in Report view |
| `Query Report` | The SQL in `query` | Developers or System Managers |
| `Script Report` | `execute(filters)` in a Python file (standard) or `report_script` (not standard) | Developers |
| `Custom Report` | A saved copy of a Query or Script Report with its own columns and filters | Users, with "Save As" |

## Script Report in an app

A standard Script Report (`is_standard = "Yes"`) lives in the module folder of the app.

```text
sample_app/sample_app/selling/report/sample_order_summary/
├── __init__.py
├── sample_order_summary.json   # the Report document
├── sample_order_summary.py     # execute(filters)
└── sample_order_summary.js     # filters and client options
```

1. Turn on `developer_mode` in site config.
2. Log in as Administrator.
3. Create a Report with `report_type` "Script Report", `ref_doctype` "Sample Order" and a module of your app.
4. Save. In developer mode, Frappe sets `is_standard` to "Yes" and writes the JSON, `.py` and `.js` boilerplate.
5. Fill in the `.py` and `.js` files.
6. Commit the folder. `bench migrate` syncs the report to other sites.

Frappe finds the module as `<app>.<module>.report.<report_name>.<report_name>` (`get_report_module_dotted_path`).

### The Python file

`execute(filters)` gets the filter values as a `frappe._dict` and returns a tuple or list. Frappe reads up to six items, in this order:

| Position | Name | Type | Description |
|---|---|---|---|
| 0 | `columns` | list | Column definitions. Dicts or strings. |
| 1 | `data` | list | Rows. Each row is a dict (keys are column `fieldname`) or a list in column order. |
| 2 | `message` | str | HTML shown above the table. |
| 3 | `chart` | dict | Chart shown above the table. |
| 4 | `report_summary` | list | Number cards shown above the table. |
| 5 | `skip_total_row` | bool | Do not add the total row for this run. |

```python
# sample_app/sample_app/selling/report/sample_order_summary/sample_order_summary.py
import frappe
from frappe import _


def execute(filters=None):
	filters = frappe._dict(filters or {})
	columns = get_columns()
	data = get_data(filters)
	return columns, data, None, get_chart(data), get_summary(data)


def get_columns():
	return [
		{"label": _("Order"), "fieldname": "name", "fieldtype": "Link", "options": "Sample Order", "width": 140},
		{"label": _("Customer"), "fieldname": "customer_name", "fieldtype": "Data", "width": 180},
		{"label": _("Order Date"), "fieldname": "order_date", "fieldtype": "Date", "width": 110},
		{"label": _("Status"), "fieldname": "status", "fieldtype": "Data", "width": 100},
		{"label": _("Total"), "fieldname": "total", "fieldtype": "Currency", "width": 120},
	]


def get_data(filters):
	conditions = {"docstatus": ["<", 2]}
	if filters.status:
		conditions["status"] = filters.status
	if filters.from_date and filters.to_date:
		conditions["order_date"] = ["between", [filters.from_date, filters.to_date]]

	return frappe.get_list(
		"Sample Order",
		filters=conditions,
		fields=["name", "customer_name", "order_date", "status", "total"],
		order_by="order_date desc",
	)


def get_chart(data):
	return {
		"data": {
			"labels": [row.name for row in data],
			"datasets": [{"name": _("Total"), "values": [row.total for row in data]}],
		},
		"type": "bar",
	}


def get_summary(data):
	return [
		{"label": _("Orders"), "value": len(data), "datatype": "Int"},
		{"label": _("Total"), "value": sum(row.total for row in data), "datatype": "Currency", "indicator": "Green"},
	]
```

A column can also be a string in the form `"Label:Fieldtype/Options:Width"`, for example `"Order:Link/Sample Order:150"`. Frappe converts it to a dict and makes the `fieldname` from the label.

The chart dict goes to Frappe Charts: `data.labels`, `data.datasets[].name`, `data.datasets[].values`, and `type` (for example `bar`, `line`, `pie`, `percentage`, `axis-mixed`). A summary item has `label`, `value` and `datatype`. It can also have `indicator` (`Green`, `Red`, `Blue` and other colors) and `currency`.

The module can also define these functions:

| Function | Used for |
|---|---|
| `execute_snapshot_report(filters)` | Called instead of `execute` when the report has `snapshot_report` set. |
| `get_xlsx_styles(metadata)` | Styles for the Excel export. |

`Report.get_module_method()` allows only these two names and `execute`.

### The JS file

The `.js` file sets `frappe.query_reports["<Report Name>"]`. The key must be the report name.

```js
// sample_app/sample_app/selling/report/sample_order_summary/sample_order_summary.js
frappe.query_reports["Sample Order Summary"] = {
	filters: [
		{
			fieldname: "status",
			label: __("Status"),
			fieldtype: "Select",
			options: ["", "Draft", "Open", "Completed"],
		},
		{
			fieldname: "from_date",
			label: __("From Date"),
			fieldtype: "Date",
			default: frappe.datetime.add_months(frappe.datetime.get_today(), -1),
		},
		{
			fieldname: "to_date",
			label: __("To Date"),
			fieldtype: "Date",
			default: frappe.datetime.get_today(),
		},
	],
	formatter(value, row, column, data, default_formatter) {
		value = default_formatter(value, row, column, data);
		if (column.fieldname === "status" && data && data.status === "Open") {
			value = `<b>${value}</b>`;
		}
		return value;
	},
	onload(report) {
		report.page.add_inner_button(__("Refresh"), () => report.refresh());
	},
};
```

A filter is a field definition (`fieldname`, `label`, `fieldtype`, `options`, `default`, `reqd`, `depends_on`, `on_change`, `get_query`). Read a value with `frappe.query_report.get_filter_value("status")`. Set one with `frappe.query_report.set_filter_value("status", "Open")`.

Other keys that the report view reads: `onload`, `formatter`, `get_datatable_options`, `after_datatable_render`, `after_refresh`, `get_chart_data`, `get_pdf_format`, `tree`, `parent_field`, `initial_depth`, `collapsible_filters`, `separate_check_filters`, `export_hidden_cols`. A `<report_name>.html` file in the report folder is the print template of the report.

### Filters table

The Report has a `filters` table (`Report Filter`) with `fieldname`, `label`, `fieldtype`, `options`, `mandatory`, `default` and `wildcard_filter`. The desk uses these rows only when the JS file does not set `filters`. Frappe does not check the filter values on the server. Check them in `execute`.

### Script Report without a file

A Script Report with `is_standard = "No"` runs `report_script` with `safe_exec`. Only a `Script Manager` can save it, and server scripts must be on (see [Server and Client Scripts](../low-code-and-ai/server-and-client-scripts.md)). The script gets `filters`. It sets `data` (a full result like the return value of `execute`) or `result` (rows only, with columns from the `columns` table).

## Query Report

A Query Report runs `query` with the filter values as SQL parameters. Write the column definition as the column alias.

```sql
select
	name as "Order:Link/Sample Order:150",
	customer_name as "Customer::150",
	total as "Total:Currency:120"
from `tabSample Order`
where docstatus < 2
	and customer_name like %(customer)s
```

`execute_query_report()` checks the query with `check_safe_sql_query`, then runs it with `frappe.db.sql`. The filters in the JS file (or the `filters` table) give the values for `%(customer)s`. The result rows come back as dicts, with `fieldname` made from the label (`order`, `customer`, `total`).

## Run a report from code

`frappe.desk.query_report.run` is the method that the desk calls. It is whitelisted and read-only.

```python
from frappe.desk.query_report import run

result = run("Sample Order Summary", filters={"status": "Open"})
result["columns"]   # list of column dicts
result["result"]    # list of row dicts, plus a total row (a list) when add_total_row is set
```

The returned dict has `result`, `columns`, `message`, `chart`, `report_summary`, `skip_total_row`, `status`, `execution_time` and `add_total_row`. A snapshot report also has `snapshot_report` and `snapshot_at`. For a prepared report, it has `prepared_report` and `doc` and the saved result.

```bash
curl -b cookies.txt -G "http://<site>/api/method/frappe.desk.query_report.run" \
  --data-urlencode 'report_name=Sample Order Summary' \
  --data-urlencode 'filters={"status": "Open"}'
```

| Parameter | Type | Default | Description |
|---|---|---|---|
| `report_name` | str | required | Name of the Report. |
| `filters` | dict or JSON str | `None` | Filter values. |
| `ignore_prepared_report` | bool | `False` | Run now, also for a prepared report. |
| `custom_columns` | list | `None` | Extra columns from linked DocTypes. |
| `is_tree`, `parent_field` | bool, str | `False`, `None` | Build the total row for a tree report. |
| `are_default_filters` | bool | `True` | Use the saved filters of a Custom Report. |
| `js_filters` | list | `None` | Filter definitions from the client, used to check Link filter permissions. |

## Permissions and roles

`get_report_doc()` checks these before a run:

1. `Report.is_permitted()`. The user must have one of the roles in the `roles` table. A `Custom Role` for the report replaces that table. When no roles are set, all users are allowed.
2. The user has `report` permission on `ref_doctype`.
3. The report is not `disabled`.

A new report that is not standard gets the permlevel 0 roles of `ref_doctype` in `roles` (`set_doctype_roles`). After the run, `get_filtered_data()` removes rows with Link values that the user's [user permissions](../security/permissions.md) do not allow. Still check permissions in `execute`. Use `frappe.get_list`, or `frappe.only_for("System Manager")` for admin reports.

## Prepared reports

A prepared report runs in a background job and saves the result in a `Prepared Report` document with a file attachment. Use it for slow reports.

- Set `prepared_report` on the Report to always run it in the background.
- A Script Report that runs for more than 15 seconds turns on `prepared_report` by itself. Set `disable_prepared_report_automation` to stop this.
- The job runs on the `long` queue. `timeout` on the Report sets the job timeout. The default is 1500 seconds (`REPORT_TIMEOUT`).
- `run()` returns the last completed result for the same filters and user. Users start a new run with "Generate New Report" (`frappe.core.doctype.prepared_report.prepared_report.make_prepared_report`).
- The user gets a notification when the job is done.
- If the job fails, Frappe sets the Prepared Report `status` to "Error" and saves the traceback in `error_message`.

## Pitfalls

- `run()` has `@frappe.read_only`. With `read_from_replica` in site config, it reads from the replica. Do not write to the database in `execute`.
- Use the same `fieldname` in columns and row dicts. Rows with other keys show empty cells.
- A Query Report cannot skip a condition when a filter is empty. Use a Script Report for optional filters.
- `add_total_row` adds a list row at the end of `result`. Code that reads `result` must handle it.
- The report name, the folder name and the JS key must match. The folder name is the scrubbed report name.

## Related

- [Permissions](../security/permissions.md)
- [Background Jobs](../runtime/background-jobs.md)
- [Print Formats](print-formats.md)
- [Server and Client Scripts](../low-code-and-ai/server-and-client-scripts.md)
- [Reports (user guide)](../../user-guide/data-and-reports/reports.md)
- [Dashboards (user guide)](../../user-guide/data-and-reports/dashboards.md)
