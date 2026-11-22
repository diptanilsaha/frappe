---
title: Performance
description: Keep Frappe apps fast with indexes, cached reads, fewer queries, bulk operations and background jobs, and find slow code with the Recorder, the monitor log and query profiling.
version: v15
sources:
  - frappe/database/database.py
  - frappe/database/mariadb/schema.py
  - frappe/database/schema.py
  - frappe/__init__.py
  - frappe/recorder.py
  - frappe/core/doctype/recorder/recorder.py
  - frappe/monitor.py
  - frappe/hooks.py
  - frappe/commands/utils.py
  - frappe/tests/utils.py
---

# Performance

Write code for tables with 100,000 rows or more. Most slow Frappe code makes too many queries, filters on columns without an index, or does long work inside a web request. This page shows the fixes and the tools to find the slow code. Measure before you change code, and measure again after.

## Keep work out of the hot path

Code that runs on every request, every save or every desk load affects all users.

- Keep `boot_session`, `before_request`, `doc_events` on `"*"` and permission hooks fast. Do not query per record in them.
- Read values the client needs on page load from `frappe.boot`, not with a server call on each form load.
- Load JavaScript that few users need on first use, not in the main bundle.

## Indexes

- Set **Index** (`search_index`) on fields that you filter or sort by. Frappe adds the index on migrate.
- Add a composite index in code with `frappe.db.add_index(doctype, fields, index_name=None)`. Call it from a patch or `after_migrate`.
- Add a unique constraint with `frappe.db.add_unique(doctype, fields, constraint_name=None)`.
- Sort by an indexed column. Every table of a parent DocType has an index on `modified`, and new DocTypes sort by `modified` (descending). If you set **Sort Field** to `creation`, Frappe adds an index on `creation` on migrate.
- A child table has an index on `parent`.

```python
# sample_app/patches/v1_0/add_order_indexes.py
import frappe


def execute():
	frappe.db.add_index("Sample Order", ["customer_name", "order_date"])
```

## Cached reads

| API | Use for |
|---|---|
| `frappe.get_cached_doc(doctype, name)` | Documents that change rarely, such as settings or masters. Cleared when the document is saved. |
| `frappe.get_cached_value(doctype, name, fieldname="name", as_dict=False)` | One or more fields of such a document. |
| `frappe.db.get_single_value(doctype, fieldname, cache=True)` | Values of a Single DocType. It caches the value on the database connection. |
| `frappe.db.get_value(doctype, name, fieldname, cache=True)` | Caches the value on the database connection, for a `name` given as a string. `cache` defaults to `False`. |

Do not edit and save a document from `get_cached_doc`. Load it with `frappe.get_doc`. See [Caching](../runtime/caching.md) for `frappe.cache`, `site_cache` and `redis_cache`. Add a cache only when you measured the cost it removes, and know how it is cleared.

## Avoid N+1 queries

Do not call `frappe.get_doc` or `frappe.db.get_value` inside a loop over records. Get all rows in one query.

```python
# Slow: one query per order
for name in frappe.get_all("Sample Order", pluck="name"):
	order = frappe.get_doc("Sample Order", name)
	total += order.total

# Fast: one query
total = sum(frappe.get_all("Sample Order", pluck="total"))

# Fast: aggregate in the database
frappe.get_all("Sample Order", fields=["sum(total) as total"])
```

- Ask only for the fields you need. `fields=["*"]` reads all columns.
- Use `pluck="field"` for a list of one field.
- Set a `limit` for lists. In v15, `frappe.get_list` and `frappe.get_all` return all matching rows when you give no limit (checked on a site with 3,165 DocField rows).
- Get the rows of a child table in one query with `frappe.get_all("Sample Order Item", filters={"parent": ["in", names]})`.

!!! note "Version"
    The dict form of aggregates (`fields=[{"SUM": "total"}]`) does not work in v15. It raises `AttributeError`. Use the string form.

## Bulk operations

| API | Description |
|---|---|
| `frappe.db.set_value(doctype, filters, field, value, update_modified=True)` | Updates all matching rows in one query. No controller events. |
| `frappe.db.bulk_update(doctype, doc_updates, *, chunk_size=100, update_modified=True)` | Updates many records with different values. `doc_updates` maps names to dicts of values. |
| `frappe.db.bulk_insert(doctype, fields, values, ignore_duplicates=False, *, chunk_size=10000)` | Inserts many rows. No controller events, no naming, no validation. |
| `frappe.qb.update(...)` | Query builder `UPDATE` with any condition. |

These skip `validate`, `on_update` and `doc_events`. Use them in patches and imports where you control the data. Do not use `frappe.db.delete` in place of `frappe.delete_doc` for normal deletes, because it skips the link checks and hooks.

In patches, commit in batches when you change many rows, so that one transaction does not hold locks for a long time.

## Background jobs

Move work that takes more than a few seconds, calls to third-party services, and bulk operations to a background job.

```python
frappe.enqueue(
	"sample_app.tasks.rebuild_totals",
	queue="long",
	enqueue_after_commit=True,
	order_name=doc.name,
)
```

See [Background Jobs](../runtime/background-jobs.md).

## Find slow code

### Recorder

The Recorder captures requests and background jobs, with every SQL query, its call stack and `EXPLAIN` output. Only Administrator can use it.

1. Open `/app/recorder`.
2. Start the Recorder. Set the request path filter if you need only some requests.
3. Do the slow action in another tab.
4. Stop the Recorder and open the captured request.

The Recorder stops after 5 minutes (`RECORDER_AUTO_DISABLE` in `frappe/recorder.py`). It stores data in Redis cache. For a captured request, the Recorder can suggest indexes (it runs the analysis on the `long` queue), and Administrator can add them from the same page.

`RecorderConfig` options: `record_requests`, `record_jobs`, `record_sql`, `capture_stack`, `profile` (cProfile), `explain`, `request_filter`, `jobs_filter`.

### Monitor log

Set `"monitor": 1` in `site_config.json`. Frappe then records each request and job: path or method, duration in microseconds, status code, and the wait time of jobs. The scheduler writes the records from Redis to `logs/monitor.json.log` in the bench (`frappe.monitor.flush` runs on the `all` event). Add your own keys to the current record with `frappe.monitor.add_data_to_monitor(**kwargs)`.

### Query profiling

- `frappe.db.sql(query, values, debug=True)` prints the query and its time.
- `frappe.db.sql(query, values, explain=True)` prints the `EXPLAIN` output.
- `bench --site frappe.localhost execute sample_app.tasks.rebuild_totals --profile` runs a function with cProfile.
- `bench --site frappe.localhost run-tests --module ... --profile` profiles tests.

### Guard performance in tests

`FrappeTestCase` has context managers that fail when code gets slower:

```python
import frappe
from frappe.tests.utils import FrappeTestCase


class TestSampleOrder(FrappeTestCase):
	def test_list_query_count(self):
		with self.assertQueryCount(3):
			frappe.get_list("Sample Order", limit=20)

		with self.assertRowsRead(100):
			frappe.get_list("Sample Order", limit=20)
```

## Pitfalls

- A filter on an unindexed column scans the whole table. Check with `explain=True`.
- `frappe.get_list` and `frappe.get_all` with no `limit` read every row.
- A `site_cache` is per process and is not cleared when data changes. Use it only for data that never changes while the process runs.
- Long requests that write hold row locks until the commit. Other users then wait. Move long writes to a background job.

## Related

- [Database](../database/database.md)
- [Query Builder](../database/query-builder.md)
- [Caching](../runtime/caching.md)
- [Background Jobs](../runtime/background-jobs.md)
- [Logs and Monitoring](../../user-guide/system-administration/logs-and-monitoring.md)
