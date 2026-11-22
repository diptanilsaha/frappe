---
title: Database API
description: Read and write data with frappe.db and frappe.get_list / get_all, the filter syntax, and when Frappe commits or rolls back a transaction.
version: develop
sources:
  - frappe/database/database.py
  - frappe/database/query.py
  - frappe/database/operator_map.py
  - frappe/database/utils.py
  - frappe/model/qb_query.py
  - frappe/model/db_query.py
  - frappe/__init__.py
  - frappe/app.py
  - frappe/auth.py
  - frappe/model/document.py
  - frappe/utils/background_jobs.py
---

# Database API

`frappe.db` is the connection to the site database for the current request or job. `frappe.get_list` and `frappe.get_all` query many rows of a DocType with filters. Use the document API (`frappe.get_doc`, `doc.save()`) when you need validation, hooks and permission checks. Use the functions on this page when you only need values, or when you need a fast write without controller logic.

All examples on this page were run in `bench --site frappe.localhost console`. The full list of methods is in the [Database API reference](../../reference/python/database.md).

## Read values

### get_value

```python
frappe.db.get_value(doctype, filters=None, fieldname="name", ignore=False, as_dict=False,
    debug=False, order_by=DefaultOrderBy, cache=False, for_update=False,
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
frappe.db.get_value("Sample Item", "Nope", "rate")
# None
```

| Option | Description |
|---|---|
| `cache=True` | Keep the result in `frappe.db.value_cache` until the transaction ends. |
| `for_update=True` | Lock the row (`SELECT ... FOR UPDATE`) until commit or rollback. |
| `skip_locked`, `wait` | With `for_update`: skip locked rows, or fail at once instead of waiting. |
| `ignore=True` | Return `None` instead of an error when the table or column is missing. |

### Other read methods

| Method | Returns |
|---|---|
| `frappe.db.get_values(doctype, filters, fieldname, ...)` | All matching rows as tuples, or dicts with `as_dict=True`. |
| `frappe.db.get(doctype, filters)` | The first matching row with all columns, as a dict. |
| `frappe.db.get_single_value(doctype, fieldname, cache=True)` | One field of a Single DocType, cast to the field type. |
| `frappe.db.exists(doctype, name_or_filters)` | The name of the first match, or `None`. |
| `frappe.db.count(doctype, filters=None)` | The number of matching rows. |
| `frappe.db.sql(query, values)` | Raw SQL result. |

```python
frappe.db.get_values("Sample Item", {"category": "Hardware"}, ["name", "rate"], order_by="rate desc")
# (('Laptop', 900.0), ('Monitor', 250.0), ('Keyboard', 45.0))
frappe.db.get_single_value("System Settings", "time_zone")
# 'Asia/Kolkata'
frappe.db.exists("Sample Item", "Laptop")               # 'Laptop'
frappe.db.exists("Sample Item", {"category": "Hardware"})  # 'Keyboard'
frappe.db.count("Sample Item", {"category": "Hardware"})   # 3
```

## Query lists: get_list and get_all

`frappe.get_list(doctype, **kwargs)` applies the permissions of the current user. `frappe.get_all(doctype, **kwargs)` takes the same arguments and sets `ignore_permissions=True`. `frappe.db.get_list` and `frappe.db.get_all` are aliases.

In develop both run on the query builder engine (`frappe.model.qb_query.DatabaseQuery` and `frappe.database.query.Engine`). The older `frappe.model.db_query.DatabaseQuery` is still in the code but `frappe.get_list` does not use it.

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
| `filters` | `dict`, `list`, `str` | `None` | Conditions joined with `AND`. See [Filters](#filters). |
| `or_filters` | `dict`, `list` | `None` | Conditions joined with `OR`, then joined to `filters` with `AND`. |
| `order_by` | `str` | DocType sort settings | For example `"rate desc"` or `"category asc, rate desc"`. |
| `group_by` | `str` | `None` | For example `"category"`. |
| `limit` | `int` | no limit | Maximum number of rows. |
| `offset` | `int` | `0` | Rows to skip. |
| `pluck` | `str` | `None` | Return a flat list of the values of one field. |
| `as_list` | `bool` | `False` | Return tuples instead of dicts. |
| `distinct` | `bool` | `False` | `SELECT DISTINCT`. |
| `ignore_permissions` | `bool` | `False` (`True` for `get_all`) | Skip permission conditions. |
| `ignore_user_permissions` | `bool` | `False` | Skip User Permission conditions only. |
| `user` | `str` | current user | Apply the permissions of another user. |
| `parent_doctype` | `str` | `None` | Required to query a child DocType with permissions. |
| `with_comment_count` | `bool` | `False` | Add `_comment_count` to each row. |
| `run` | `bool` | `True` | Set to `False` to get the query object instead of rows. |
| `debug` | `bool` | `False` | Print the query. |

`limit_start`, `limit_page_length`, `start` and `page_length` still work but show a deprecation warning. Use `offset` and `limit`.

!!! warning
    In develop, `frappe.get_list` has no default limit. It returns all permitted rows. Pass `limit` when the table can be large. The REST endpoints and `frappe.client.get_list` keep a default of 20.

```python
frappe.get_all("Sample Item", fields=["name", "rate"], order_by="rate desc", limit=2, as_list=True)
# (('Laptop', 900.0), ('Support Plan', 300.0))
frappe.get_all("Sample Item", order_by="rate desc", limit=2, offset=2, pluck="name")
# ['Monitor', 'Office Suite']
```

### Filters

The same filter syntax works in `frappe.get_list`, `frappe.get_all`, `frappe.db.get_value`, `frappe.db.get_values`, `frappe.db.exists`, `frappe.db.count`, `frappe.db.set_value`, `frappe.db.delete`, `frappe.qb.get_query` and the REST API.

| Form | Example | Meaning |
|---|---|---|
| Dict, equality | `{"category": "Hardware"}` | `category = 'Hardware'` |
| Dict, operator | `{"rate": [">", 100]}` or `{"rate": (">", 100)}` | `rate > 100` |
| List of conditions | `[["rate", ">=", 100], ["category", "in", ["Hardware", "Service"]]]` | Both, with `AND` |
| Condition with DocType | `[["Sample Order Item", "item", "=", "Laptop"]]` | Filter on a child table field |
| Nested `and` / `or` | `[[["category", "=", "Service"], "or", ["rate", ">", 800]]]` | Grouped logic |
| List of names | `["Laptop", "Monitor"]` | `name in (...)` |
| One name | `"Laptop"` | `name = 'Laptop'` |
| Criterion | `frappe.qb.DocType("Sample Item").rate > 800` | Query builder term |

```python
frappe.get_all("Sample Item", filters=[["rate", ">=", 100], ["category", "in", ["Hardware", "Service"]]], pluck="name")
# ['Support Plan', 'Monitor', 'Laptop']
frappe.get_all("Sample Item", or_filters={"category": "Service", "rate": [">", 800]}, pluck="name")
# ['Support Plan', 'Installation', 'Laptop']
frappe.get_all("Sample Item", filters=[[["category", "=", "Service"], "or", ["rate", ">", 800]]], pluck="name")
# ['Support Plan', 'Installation', 'Laptop']
frappe.get_all("Sample Order", filters=[["Sample Order Item", "item", "=", "Laptop"]], pluck="name")
# ['SO-00006', 'SO-00001']
```

#### Operators

The operators are in `OPERATOR_MAP` in `frappe/database/operator_map.py`, plus the date and tree operators handled in `frappe/database/query.py`.

| Operator | Value | Notes |
|---|---|---|
| `=`, `!=`, `<`, `>`, `<=`, `>=` | scalar | `=<` and `=>` are also accepted. |
| `in`, `not in` | list or comma-separated string | An empty list with `in` matches nothing. With `not in` it matches everything. |
| `like`, `not like` | pattern with `%` | Case-insensitive on MariaDB (default collation) and on Postgres (uses `ILIKE`). |
| `ilike` | pattern | Postgres only. On MariaDB the query fails with a SQL syntax error. Use `like` there. |
| `regex` | pattern | `REGEXP` on MariaDB, `~*` on Postgres. There is no `not regex`. |
| `between` | `[from, to]` | Dates in a Datetime field cover the full day. |
| `is` | `"set"` or `"not set"` | `not set` also matches the empty value of the field type. |
| `timespan` | for example `"this year"`, `"last week"` | Converted to `between`. |
| `previous`, `next` | for example `"1 week"` | Converted to `between`. |
| `ancestors of`, `descendants of`, `not ancestors of`, `not descendants of`, `descendants of (inclusive)` | document name | For tree DocTypes. |

```python
frappe.get_all("Sample Item", filters={"rate": ["between", [50, 150]]}, pluck="name")
# ['Installation', 'Antivirus', 'Office Suite']
frappe.get_all("Sample Item", filters={"item_name": ["like", "%top%"]}, pluck="name")
# ['Laptop']
frappe.get_all("Sample Item", filters={"category": ["not in", ["Hardware"]]}, pluck="name")
# ['Support Plan', 'Installation', 'Antivirus', 'Office Suite']
```

Apps can add operators with the `filters_config` hook.

### Fields

| Form | Example | Result |
|---|---|---|
| Field name | `"rate"` | Column of the DocType |
| Alias | `"rate as price"` | Column with another key |
| All fields | `"*"` | All permitted columns |
| Child table field | `"items.item"` | One row per child row |
| Child table as list | `{"items": ["item", "qty"]}` | A list of child rows in each parent row |
| Link field value | `"item.category"` | Field of the linked document (join) |
| Function | `{"COUNT": "*", "as": "count"}` | Aggregate or SQL function |
| Arithmetic | `{"MUL": ["qty", "rate"], "as": "value"}` | `ADD`, `SUB`, `MUL`, `DIV` |

```python
frappe.get_all("Sample Order", fields=["name", {"items": ["item", "qty"]}], filters={"name": "SO-00001"})
# [{'name': 'SO-00001', 'items': [{'item': 'Laptop', 'qty': 2.0}, {'item': 'Monitor', 'qty': 2.0}]}]
frappe.get_all("Sample Order Item", fields=["item", "item.category"], limit=2)
# [{'item': 'Laptop', 'category': 'Hardware'}, {'item': 'Antivirus', 'category': 'Software'}]
frappe.get_all("Sample Item",
    fields=["category", {"COUNT": "*", "as": "count"}, {"SUM": "rate", "as": "total_rate"}],
    group_by="category", order_by="category asc")
# [{'category': 'Hardware', 'count': 3, 'total_rate': 1195.0},
#  {'category': 'Service', 'count': 2, 'total_rate': 380.0},
#  {'category': 'Software', 'count': 2, 'total_rate': 180.0}]
```

The supported functions are in `FUNCTION_MAPPING` in `frappe/database/query.py`: `COUNT`, `SUM`, `AVG`, `MAX`, `MIN`, `ABS`, `EXTRACT`, `LOCATE`, `TIMESTAMP`, `IFNULL`, `CONCAT`, `NOW`, `NULLIF`, `MONTHNAME`, `QUARTER`, `MONTH`, `YEAR`.

!!! warning
    SQL functions written as strings are rejected:

    ```python
    frappe.get_all("Sample Item", fields=["count(name) as c"])
    # frappe.ValidationError: SQL functions are not allowed as strings in SELECT: count(name) as c.
    # Use dict syntax like {'COUNT': '*'} instead.
    ```

## Raw SQL

```python
frappe.db.sql(query, values=(), *, as_dict=0, as_list=0, debug=0, pluck=False, as_iterator=False, ...)
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

Table names are `tab` plus the DocType name. `frappe.db.sql` also accepts a query builder object. Prefer [the query builder](query-builder.md), which works on MariaDB, Postgres and SQLite.

## Write values

| Method | Description |
|---|---|
| `frappe.db.set_value(doctype, name_or_filters, field, val=None, modified=None, modified_by=None, update_modified=True)` | Updates columns with SQL. `field` can be a dict of field values. |
| `frappe.db.set_single_value(doctype, fieldname, value=None, *, update_modified=True)` | Updates a Single DocType. |
| `frappe.db.delete(doctype, filters=None)` | Deletes matching rows. |
| `frappe.db.bulk_insert(...)` | Inserts many rows, one statement per chunk of 1,000 rows (`chunk_size`). Postgres uses `COPY`. |

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

Inside `doc_events` handlers, `frappe.db.commit()` and `frappe.db.rollback()` do nothing. `Document.run_method` increments `frappe.db._disable_transaction_control` while hooks run, and the database shows a warning instead. Do not commit in document events.

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

An exception in an after-commit callback is logged in Error Log. It does not fail the request. See [Background jobs](../runtime/background-jobs.md).

### Manual control

| Method | Description |
|---|---|
| `frappe.db.commit()` | Commit and start a new transaction. |
| `frappe.db.rollback()` | Roll back and start a new transaction. |
| `frappe.db.savepoint(name)` | Create a savepoint. |
| `frappe.db.rollback(save_point=name)` | Undo changes after the savepoint only. |
| `frappe.db.release_savepoint(name)` | Remove a savepoint. |
| `with frappe.db.read_only():` | Run a block in a read-only transaction. The open transaction must have no writes. |

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
- `frappe.db.exists` with filters returns the name of a matching row, not a boolean. It adds no `ORDER BY`, so do not depend on which row it returns.
- On MariaDB, a DDL statement (`CREATE`, `ALTER`) after writes in the same transaction raises `frappe.exceptions.ImplicitCommitError`, because it would commit them. Commit first, or run DDL before writes.

## Related

- [Query builder](query-builder.md)
- [Permissions](../security/permissions.md)
- [Controllers](../data-model/controllers.md)
- [Background jobs](../runtime/background-jobs.md)
- [Performance](../testing-and-performance/performance.md)
- [Database API reference](../../reference/python/database.md)
