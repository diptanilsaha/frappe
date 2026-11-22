---
title: Database API
description: Read and write data with frappe.db and frappe.get_list / get_all, the filter syntax, and when Frappe commits or rolls back a transaction.
version: v15
sources:
  - frappe/database/database.py
  - frappe/database/query.py
  - frappe/database/operator_map.py
  - frappe/database/utils.py
  - frappe/model/db_query.py
  - frappe/utils/data.py
  - frappe/__init__.py
  - frappe/client.py
  - frappe/app.py
  - frappe/auth.py
  - frappe/model/document.py
---

# Database API

`frappe.db` is the connection to the site database for the current request or job. `frappe.get_list` and `frappe.get_all` query many rows of a DocType with filters. Use the document API (`frappe.get_doc`, `doc.save()`) when you need validation, hooks and permission checks. Use the functions on this page when you only need values, or when you need a fast write without controller logic.

All examples on this page were run in `bench --site frappe.localhost console` on a v15 site. The full list of methods is in the [Database API reference](../../reference/python/database.md).

## Two query engines

v15 has two engines. They accept different filter and field forms.

| Engine | Used by |
|---|---|
| `frappe.model.db_query.DatabaseQuery` (builds SQL strings) | `frappe.get_list`, `frappe.get_all`, `frappe.db.get_list`, `frappe.db.get_all`, `frappe.client.get_list`, the REST list endpoints |
| `frappe.database.query.Engine` (query builder) | `frappe.db.get_value`, `get_values`, `exists`, `count`, `set_value`, `delete`, and `frappe.qb.get_query` |

## Read values

### get_value

```python
frappe.db.get_value(doctype, filters=None, fieldname="name", ignore=None, as_dict=False,
    debug=False, order_by="KEEP_DEFAULT_ORDERING", cache=False, for_update=False,
    *, run=True, pluck=False, distinct=False, skip_locked=False, wait=True)
```

`filters` is a document name, a filter dict or list, or a query builder criterion. The method returns the first match, or `None`.

```python
frappe.db.get_value("Sample Item", "Laptop", "rate")
# 900.0
frappe.db.get_value("Sample Item", "Laptop", ["item_name", "rate"])
# ('Laptop', 900.0)
frappe.db.get_value("Sample Item", "Laptop", ["item_name", "rate"], as_dict=True)
# {'item_name': 'Laptop', 'rate': 900.0}
frappe.db.get_value("Sample Item", {"category": "Software", "rate": (">", 100)}, "name")
# 'Office Suite'
frappe.db.get_value("Sample Item", frappe.qb.DocType("Sample Item").rate > 800, "name")
# 'Laptop'
frappe.db.get_value("Sample Item", "Nope", "rate")
# None
```

| Option | Description |
|---|---|
| `cache=True` | Keep the result in `frappe.db.value_cache` until the next commit. |
| `for_update=True` | Lock the row (`SELECT ... FOR UPDATE`) until commit or rollback. |
| `skip_locked`, `wait` | With `for_update`: skip locked rows, or fail at once instead of waiting. |
| `ignore=True` | Return `None` instead of an error when the table or column is missing. |

### Other read methods

| Method | Returns |
|---|---|
| `frappe.db.get_values(doctype, filters, fieldname, ...)` | All matching rows as tuples, or dicts with `as_dict=True`. |
| `frappe.db.get(doctype, filters)` | The first matching row with all columns, as a dict. |
| `frappe.db.get_single_value(doctype, fieldname, cache=True)` | One field of a Single DocType, cast to the field type. Throws if the field does not exist. |
| `frappe.db.exists(doctype, name_or_filters)` | The name of the first match, or `None`. |
| `frappe.db.count(doctype, filters=None)` | The number of matching rows. |
| `frappe.db.sql(query, values)` | Raw SQL result. |

```python
frappe.db.get_values("Sample Item", {"category": "Hardware"}, ["name", "rate"], order_by="rate desc")
# (('Laptop', 900.0), ('Monitor', 250.0), ('Keyboard', 45.0))
frappe.db.get_single_value("System Settings", "time_zone")
# 'Asia/Calcutta'
frappe.db.exists("Sample Item", "Laptop")               # 'Laptop'
frappe.db.exists("Sample Item", {"category": "Hardware"})  # 'Keyboard'
frappe.db.count("Sample Item", {"category": "Hardware"})   # 3
```

## Query lists: get_list and get_all

`frappe.get_list(doctype, **kwargs)` applies the permissions of the current user. `frappe.get_all(doctype, **kwargs)` takes the same arguments and sets `ignore_permissions=True`. `frappe.db.get_list` and `frappe.db.get_all` are aliases. Both call `frappe.model.db_query.DatabaseQuery(doctype).execute(...)`.

```python
frappe.get_all("Sample Item",
    filters={"category": "Hardware"},
    fields=["name", "rate"],
    order_by="rate desc")
# [{'name': 'Laptop', 'rate': 900.0}, {'name': 'Monitor', 'rate': 250.0}, {'name': 'Keyboard', 'rate': 45.0}]
```

### Arguments

| Name | Type | Default | Description |
|---|---|---|---|
| `fields` | `list`, `str` | `["name"]` | Fields to select. See [Fields](#fields). |
| `filters` | `dict`, `list` | `None` | Conditions joined with `AND`. See [Filters](#filters). |
| `or_filters` | `dict`, `list` | `None` | Conditions joined with `OR`, then joined to `filters` with `AND`. |
| `order_by` | `str` | DocType sort settings | For example `"rate desc"` or `"category asc, rate desc"`. |
| `group_by` | `str` | `None` | For example `"category"`. |
| `limit` or `limit_page_length` or `page_length` | `int` | no limit | Maximum number of rows. |
| `limit_start` or `start` | `int` | `0` | Rows to skip. |
| `pluck` | `str` | `None` | Return a flat list of the values of one field. |
| `as_list` | `bool` | `False` | Return tuples instead of dicts. |
| `distinct` | `bool` | `False` | `SELECT DISTINCT`. |
| `ignore_permissions` | `bool` | `False` (`True` for `get_all`) | Skip permission conditions. |
| `user` | `str` | current user | Apply the permissions of another user. |
| `parent_doctype` | `str` | `None` | Required to query a child DocType with permissions. |
| `with_comment_count` | `bool` | `False` | Add `_comment_count` to each row. |
| `run` | `bool` | `True` | Set to `False` to get the SQL string instead of rows. |
| `debug` | `bool` | `False` | Print the query. |

`offset` and `ignore_user_permissions` are not arguments in v15. Passing them raises `TypeError`.

!!! warning
    `frappe.get_list` and `frappe.get_all` have no default limit. They return all matching rows, although the docstring says 20. Pass `limit` when the table can be large. `frappe.client.get_list` and the REST list endpoints use a default of 20.

```python
frappe.get_all("Sample Item", fields=["name", "rate"], order_by="rate desc", limit=2, as_list=True)
# (('Laptop', 900.0), ('Support Plan', 300.0))
frappe.get_all("Sample Item", order_by="rate desc", limit=2, start=2, pluck="name")
# ['Monitor', 'Office Suite']
```

### Filters

These forms work in `frappe.get_list`, `frappe.get_all` and the REST API:

| Form | Example | Meaning |
|---|---|---|
| Dict, equality | `{"category": "Hardware"}` | `category = 'Hardware'` |
| Dict, operator | `{"rate": [">", 100]}` or `{"rate": (">", 100)}` | `rate > 100` |
| List of conditions | `[["rate", ">=", 100], ["category", "in", ["Hardware", "Service"]]]` | Both, with `AND` |
| Condition with DocType | `[["Sample Order Item", "item", "=", "Laptop"]]` | Filter on a child table field |

```python
frappe.get_all("Sample Item", filters=[["rate", ">=", 100], ["category", "in", ["Hardware", "Service"]]], pluck="name")
# ['Support Plan', 'Monitor', 'Laptop']
frappe.get_all("Sample Item", or_filters={"category": "Service", "rate": [">", 800]}, pluck="name")
# ['Support Plan', 'Installation', 'Laptop']
frappe.get_all("Sample Order", filters=[["Sample Order Item", "item", "=", "Laptop"]], pluck="name")
# ['SO-00006', 'SO-00001']
```

`frappe.get_list` does not accept a list of names, a single name string, a query builder criterion, or nested `and`/`or` groups as `filters`. Use `or_filters` for `OR` logic. The `frappe.db.*` methods accept a criterion as `filters`, but not nested groups either.

#### Operators

`frappe.get_list` accepts the operators in `get_filter` in `frappe/utils/data.py`:

| Operator | Value | Notes |
|---|---|---|
| `=`, `!=`, `<`, `>`, `<=`, `>=` | scalar | |
| `in`, `not in` | list or comma-separated string | An empty list with `in` matches nothing. |
| `like`, `not like` | pattern with `%` | |
| `between` | `[from, to]` | Only for Date and Datetime fields and `creation` / `modified`. Dates in a Datetime field cover the full day. |
| `is` | `"set"` or `"not set"` | Compares with the empty string. |
| `timespan` | for example `"this year"`, `"last week"` | Converted to `between`. |
| `previous`, `next` | for example `"1 week"` | Converted to `between`. |
| `ancestors of`, `descendants of`, `not ancestors of`, `not descendants of`, `descendants of (inclusive)` | document name | For tree DocTypes. |

```python
frappe.get_all("Sample Item", filters={"item_name": ["like", "%top%"]}, pluck="name")
# ['Laptop']
frappe.get_all("Sample Item", filters={"category": ["not in", ["Hardware"]]}, pluck="name")
# ['Support Plan', 'Installation', 'Antivirus', 'Office Suite']
frappe.get_all("Sample Item", filters=[["rate", ">=", 50], ["rate", "<=", 150]], pluck="name")
# ['Installation', 'Antivirus', 'Office Suite']
```

!!! warning
    In `frappe.get_list`, `between` on a number field makes invalid SQL and raises `ProgrammingError`. Use `>=` and `<=` as in the last example. `frappe.db.get_values` and `frappe.db.count` accept `between` on numbers.

Apps can add operators with the `filters_config` hook.

### Fields

| Form | Example | Result |
|---|---|---|
| Field name | `"rate"` | Column of the DocType |
| Alias | `"rate as price"` | Column with another key |
| All fields | `"*"` | All permitted columns |
| Child table field | `"items.item"` or ``"`tabSample Order Item`.item"`` | One row per child row |
| Link field value | `"item.category"` | Field of the linked document (join) |
| SQL function as a string | `"count(name) as count"` | Aggregate function |

```python
frappe.get_all("Sample Order", fields=["name", "items.item", "items.qty"], filters={"name": "SO-00001"})
# [{'name': 'SO-00001', 'item': 'Laptop', 'qty': 2.0}, {'name': 'SO-00001', 'item': 'Monitor', 'qty': 2.0}]
frappe.get_all("Sample Order Item", fields=["item", "item.category"], limit=2)
# [{'item': 'Office Suite', 'category': 'Software'}, {'item': 'Laptop', 'category': 'Hardware'}]
frappe.get_all("Sample Item",
    fields=["category", "count(name) as count", "sum(rate) as total_rate"],
    group_by="category", order_by="category asc")
# [{'category': 'Hardware', 'count': 3, 'total_rate': 1195.0},
#  {'category': 'Service', 'count': 2, 'total_rate': 380.0},
#  {'category': 'Software', 'count': 2, 'total_rate': 180.0}]
```

!!! note "Version"
    v15 `frappe.get_list` takes SQL functions as strings. Dict fields such as `{"COUNT": "*"}` or `{"items": ["item"]}` raise `AttributeError` in v15. From v16, string SQL functions are rejected and the dict form is required.

## Raw SQL

```python
frappe.db.sql(query, values=(), *, as_dict=0, as_list=0, debug=0, ignore_ddl=0,
    auto_commit=0, update=None, explain=False, run=True, pluck=False, as_iterator=False)
```

Always pass values as parameters. Never format them into the query string.

```python
frappe.db.sql("select name, rate from `tabSample Item` where category=%s order by rate",
    ("Software",), as_dict=True)
# [{'name': 'Antivirus', 'rate': 60.0}, {'name': 'Office Suite', 'rate': 120.0}]
frappe.db.sql("select name from `tabSample Item` where rate > %(min)s order by rate",
    {"min": 200}, pluck=True)
# ['Monitor', 'Support Plan', 'Laptop']
```

Table names are `tab` plus the DocType name. `frappe.db.sql` also accepts a query builder object. Prefer [the query builder](query-builder.md), which works on MariaDB and Postgres.

## Write values

| Method | Description |
|---|---|
| `frappe.db.set_value(dt, dn, field, val=None, modified=None, modified_by=None, update_modified=True, debug=False)` | Updates columns with SQL. `dn` is a name or filters. `field` can be a dict of field values. |
| `frappe.db.set_single_value(doctype, fieldname, value=None, *, modified=None, modified_by=None, update_modified=True, debug=False)` | Updates a Single DocType. |
| `frappe.db.delete(doctype, filters=None, debug=False)` | Deletes matching rows. |
| `frappe.db.bulk_insert(doctype, fields, values, ignore_duplicates=False, *, chunk_size=10000)` | Inserts many rows with one multi-row `INSERT` per chunk of 10,000 rows, on MariaDB and Postgres. Does not set `creation`, `modified` or `owner`. |

```python
frappe.db.set_value("Sample Item", "Keyboard", "rate", 50)
frappe.db.set_value("Sample Item", "Keyboard", {"rate": 50, "disabled": 0}, update_modified=False)
```

`frappe.db.set_value` does not run validations, controller methods or `doc_events`. It does not check permissions and does not add a Version entry. It sets `modified` and `modified_by` unless `update_modified=False`. It clears the document cache. Use `doc.db_set` or `doc.save()` when hooks must run. See [Controllers](../data-model/controllers.md).

Calling `frappe.db.set_value` on a Single DocType is deprecated. Use `frappe.db.set_single_value`.

## Transactions

Frappe opens a transaction for each request and each background job. The code that runs in it shares one connection, `frappe.db`.

### When Frappe commits

| Context | On success | On error |
|---|---|---|
| HTTP request with `POST`, `PUT`, `DELETE` or `PATCH` | Commit | Rollback |
| HTTP request with `GET` or other safe methods | Rollback, unless `frappe.local.flags.commit` is set | Rollback |
| Background job (`frappe.enqueue`) | Commit | Rollback |
| `bench execute`, `bench console`, patches | You call `frappe.db.commit()` | |

The request logic is `sync_database` in `frappe/app.py`. `UNSAFE_HTTP_METHODS` is defined in `frappe/auth.py`.

!!! warning
    Writes in a `GET` request are rolled back. Declare `methods=["POST"]` on a whitelisted method that writes. See [Whitelisted methods](../api/whitelisted-methods.md).

### doc_events and controller hooks

Controller methods and `doc_events` handlers run inside the same transaction as the request. An exception in any of them rolls back the whole request, including writes made before the exception.

v15 does not block `frappe.db.commit()` inside `doc_events` handlers or controller methods. The call commits everything written so far. A later exception cannot roll that back. Do not commit in document events.

### Run code after commit

| API | Description |
|---|---|
| `frappe.db.after_commit.add(fn)` | Run `fn` after the next commit. Dropped on rollback. |
| `frappe.db.before_commit.add(fn)` | Run `fn` before the next commit. |
| `frappe.db.after_rollback.add(fn)`, `frappe.db.before_rollback.add(fn)` | Run `fn` around a rollback. |
| `frappe.enqueue(..., enqueue_after_commit=True)` | Put the job in the queue only after the commit. A worker then sees the committed data. |

```python
frappe.db.after_commit.add(lambda: print("after commit ran"))
frappe.db.commit()
# after commit ran
```

An exception in an after-commit callback is raised from `frappe.db.commit()`. The data is already committed at that point. See [Background jobs](../runtime/background-jobs.md).

### Manual control

| Method | Description |
|---|---|
| `frappe.db.commit()` | Commit and start a new transaction. |
| `frappe.db.rollback()` | Roll back and start a new transaction. |
| `frappe.db.savepoint(name)` | Create a savepoint. |
| `frappe.db.rollback(save_point=name)` | Undo changes after the savepoint only. |
| `frappe.db.release_savepoint(name)` | Remove a savepoint. |

```python
frappe.db.savepoint("sample_sp")
frappe.db.set_value("Sample Item", "Keyboard", "rate", 1)
frappe.db.get_value("Sample Item", "Keyboard", "rate")   # 1.0
frappe.db.rollback(save_point="sample_sp")
frappe.db.get_value("Sample Item", "Keyboard", "rate")   # 45.0
```

A rollback to a savepoint undoes database changes only. Callbacks and files are not undone.

## Pitfalls

- `frappe.get_all` and `frappe.db.*` do not check permissions.
- `frappe.get_list` returns all rows when you do not pass `limit`.
- `frappe.db.get_value` returns `None` when nothing matches. A tuple unpack of the result then fails.
- `frappe.db.exists` with filters returns a name, not a boolean. The query has no `ORDER BY`, so when many rows match, the database picks the row.
- The `regex` operator in `frappe.db.*` filters makes invalid SQL on MariaDB. Use `like`, or raw SQL with `REGEXP`.
- On MariaDB, `frappe.db.sql` raises `frappe.exceptions.ImplicitCommitError` for `CREATE`, `ALTER`, `DROP`, `TRUNCATE`, `START` or `BEGIN` after a write in the open transaction. Use `frappe.db.sql_ddl`, which commits first. Postgres has no such check.

## Related

- [Query builder](query-builder.md)
- [Permissions](../security/permissions.md)
- [Controllers](../data-model/controllers.md)
- [Background jobs](../runtime/background-jobs.md)
- [Performance](../testing-and-performance/performance.md)
- [Database API reference](../../reference/python/database.md)
