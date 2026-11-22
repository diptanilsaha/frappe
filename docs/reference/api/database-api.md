---
title: Database API
description: Methods on frappe.db with signatures, grouped by theme, and the differences between MariaDB, Postgres and SQLite.
version: develop
sources:
  - frappe/database/database.py
  - frappe/database/mariadb/database.py
  - frappe/database/postgres/database.py
  - frappe/database/sqlite/database.py
  - frappe/database/__init__.py
---

# Database API

`frappe.db` is the database connection of the current site. It is an instance of `MariaDBDatabase`, `PostgresDatabase` or `SQLiteDatabase`. All three extend `Database` in `frappe/database/database.py`. Use this page to find a method and its exact signature. For usage, filters and transactions, see [Database](../../developer-guide/database/database.md).

The signatures come from `inspect.signature` on a develop site (MariaDB). Internal helpers are left out. `frappe.db.db_type` returns `"mariadb"`, `"postgres"` or `"sqlite"`.

!!! note
    Methods on this page do not check permissions, unless the description says so. `set_value`, `delete` and `sql` do not run controller hooks.

## Read values

| Method | Signature | Description |
|---|---|---|
| `get_value` | `(doctype: str, filters=None, fieldname='name', ignore=False, as_dict=False, debug=False, order_by='KEEP_DEFAULT_ORDERING', cache=False, for_update=False, *, run=True, pluck=False, distinct=False, skip_locked=False, wait=True)` | Return one value, or a tuple of values if `fieldname` is a list, from the first matching row. `filters` is a name, a dict or a list. Use `None` for a Single DocType. |
| `get_values` | `(doctype: str, filters=None, fieldname='name', ignore=False, as_dict=False, debug=False, order_by='KEEP_DEFAULT_ORDERING', update=None, cache=False, for_update=False, *, run=True, pluck=False, distinct=False, limit=None, skip_locked=False, wait=True)` | Return values from all matching rows. |
| `get` | `(doctype, filters=None, as_dict=True, cache=False)` | Return `get_value` with `fieldname='*'`. |
| `get_single_value` | `(doctype: str, fieldname: str, cache: bool = True, *, debug=False, for_update=False, run=True)` | Return a field of a Single DocType, cast to its fieldtype. Cached in the request by default. |
| `get_singles_value` | `(*args, **kwargs)` | Alias of `get_single_value`. |
| `get_singles_dict` | `(doctype, debug=False, *, for_update=False, cast=False)` | Return all fields of a Single DocType as a dict. |
| `get_list` | `(*args, **kwargs)` | Calls `frappe.get_list`. Checks permissions. |
| `get_all` | `(*args, **kwargs)` | Calls `frappe.get_all`. Does not check permissions. |
| `exists` | `(dt, dn=None, cache=False, *, debug=False)` | Return the name of a matching document, or `None`. `dn` is a name or a dict of filters. `cache` works only when `dt` and `dn` are strings. |
| `count` | `(dt, filters=None, debug=False, cache=False, distinct: bool = True)` | Return `COUNT(*)` for the DocType and filters. |
| `estimate_count` | `(doctype: str) -> int` | Return the estimated row count from table statistics. Cached in Redis for 60 minutes. |
| `a_row_exists` | `(doctype)` | Return `True` if the table has at least one row. |
| `get_creation_count` | `(doctype, minutes)` | Return the number of records created in the last `minutes`. |
| `get_last_created` | `(doctype)` | Return the `creation` timestamp of the last record. |
| `get_descendants` | `(doctype, name)` | Return the descendants of a group node in a tree DocType. |

## Write values

| Method | Signature | Description |
|---|---|---|
| `set_value` | `(dt: str, dn, field: str, val=None, modified=None, modified_by=None, update_modified=True, debug=False)` | Update columns with SQL. No controller hooks or validation. Updates `modified` and `modified_by` unless `update_modified=False`. `field` can be a dict of field and value. |
| `set_single_value` | `(doctype: str, fieldname: str \| dict, value=None, *, modified=None, modified_by=None, update_modified=True, debug=False)` | Set fields of a Single DocType. |
| `bulk_update` | `(doctype: str, doc_updates: dict, *, chunk_size=100, modified=None, modified_by=None, update_modified=True, debug=False)` | Update many documents. `doc_updates` maps each name to a dict of values. |
| `bulk_insert` | `(doctype: str, fields: list[str], values: Iterable[Sequence], ignore_duplicates=False, *, chunk_size=1000)` | Insert many rows in one query per chunk. No controller hooks. |
| `delete` | `(doctype: str, filters: dict \| list \| None = None, debug=False, **kwargs)` | Run `DELETE` for matching rows. No controller hooks. |
| `truncate` | `(doctype: str)` | Run `TRUNCATE TABLE`. This is DDL and cannot be rolled back. |

## Raw SQL and query builder

| Method | Signature | Description |
|---|---|---|
| `sql` | `(query, values=<no value>, *, as_dict=0, as_list=0, debug=0, ignore_ddl=0, auto_commit=0, update=None, explain=False, run=True, pluck=False, as_iterator=False)` | Run a query and fetch all rows. Pass parameters in `values`, never with string formatting. `query` can also be a query builder object. |
| `sql_list` | `(query, values=(), debug=False, **kwargs)` | Return the first column of each row as a list. |
| `sql_ddl` | `(query, debug=False)` | Commit, then run a DDL query. |
| `multisql` | `(sql_dict, values=(), **kwargs)` | Run the query for the current `db_type` from a dict like `{"mariadb": ..., "postgres": ..., "*": ...}`. |
| `qb` | attribute | Query builder class for the current database. Same as `frappe.qb`. |
| `mogrify` | `(query, values)` | Return the query string with values substituted. |
| `escape` | `(s, percent=True)` | Escape quotes, and `%` if `percent=True`. |
| `explain_query` | `(query, values=<no value>)` | Print `EXPLAIN` of the query in the error log. |
| `unbuffered_cursor` | `()` | Context manager. Use a cursor that streams rows instead of fetching all of them. |
| `get_description` | `()` | Return the column metadata of the last query. |

## Transactions

See [Database](../../developer-guide/database/database.md) for when Frappe commits for you.

| Method | Signature | Description |
|---|---|---|
| `begin` | `(*, read_only=False)` | Start a transaction. |
| `commit` | `(*, chain=False)` | Commit and start a new transaction. Runs `before_commit` and `after_commit` callbacks. Inside `read_only()` it rolls back. |
| `rollback` | `(*, save_point=None, chain=False)` | Roll back the transaction, or to a savepoint. |
| `savepoint` | `(save_point)` | Make a savepoint. Rollback to it undoes only database changes, not files. |
| `release_savepoint` | `(save_point)` | Release a savepoint. |
| `read_only` | `()` | Context manager. Runs the block in a read-only transaction. The open transaction must have no writes. |
| `execution_timeout` | `(seconds: float)` | Context manager. Sets a statement timeout for the block, then restores the old one. |
| `set_execution_timeout` | `(seconds: float)` | Set the statement timeout of the session. |
| `get_execution_timeout` | `() -> float` | Return the statement timeout in seconds. |
| `before_commit` | `CallbackManager` | `frappe.db.before_commit.add(fn)` runs `fn` before the next commit. |
| `after_commit` | `CallbackManager` | `frappe.db.after_commit.add(fn)` runs `fn` after the next commit. |
| `before_rollback` | `CallbackManager` | Runs callbacks before the next rollback. |
| `after_rollback` | `CallbackManager` | Runs callbacks after the next rollback. |
| `advisory_lock` | `(key, *, timeout=10)` | Context manager. Holds a session-level named lock for the block. Raises `QueryTimeoutError` after `timeout` seconds. |
| `transaction_advisory_lock` | `(key, *, timeout=10)` | Take a lock that ends with the transaction. Postgres only. |

`frappe.database.database.savepoint(catch=Exception)` is a context manager and decorator. It wraps a block in a savepoint and rolls back to it if `catch` is raised:

```python
from frappe.database.database import savepoint

for name in ["Keyboard", "Monitor"]:
	with savepoint(catch=frappe.DuplicateEntryError):
		frappe.get_doc({"doctype": "Sample Item", "item_name": name}).insert()
```

## Defaults and globals

| Method | Signature | Description |
|---|---|---|
| `get_default` | `(key, parent='__default')` | Return a default value, or a list if there are many. |
| `get_defaults` | `(key=None, parent='__default')` | Return all defaults, or the values of one key. |
| `set_default` | `(key, val, parent='__default', parenttype=None)` | Set a global or user default. |
| `add_default` | `(key, val, parent='__default', parenttype=None)` | Add a value. A key can have many values. |
| `get_global` | `(key, user='__global')` | Return a global value. |
| `set_global` | `(key, val, user='__global')` | Save a global value. |
| `get_system_setting` | `(key)` | Return a value from System Settings. |

## Schema

| Method | Signature | Description |
|---|---|---|
| `table_exists` | `(doctype, cached=True)` | Return `True` if the table of the DocType exists. |
| `has_table` | `(doctype)` | Return `True` if the table exists. |
| `has_column` | `(doctype, column)` | Return `True` if the column exists. |
| `field_exists` | `(dt, fn)` | Return `True` if a DocField with this fieldname exists in the DocType. |
| `get_tables` | `(cached=True)` | Return all table names. |
| `get_table_columns` | `(doctype)` | Return column names of the DocType table. |
| `get_db_table_columns` | `(table) -> list[str]` | Return column names of a table by table name. |
| `get_table_columns_description` | `(table_name)` | Return columns with type, default, index and unique flags. |
| `get_column_type` | `(doctype, column)` | Return the database type of a column. |
| `describe` | `(doctype: str)` | Return the `DESCRIBE` output of the table. |
| `add_index` | `(doctype: str, fields: list, index_name=None, using=None, where=None, include=None)` | Make an index if it does not exist. `using`, `where` and `include` work on Postgres only. |
| `add_unique` | `(doctype, fields, constraint_name=None)` | Add a unique constraint. |
| `has_index` | `(table_name, index_name)` | Return `True` if the index exists. |
| `get_column_index` | `(table_name: str, fieldname: str, unique: bool = False)` | Return the index on a column, if any. |
| `change_column_type` | `(doctype: str, column: str, type: str, nullable: bool = False)` | Change the type of a column. |
| `rename_column` | `(doctype: str, old_column_name, new_column_name)` | Rename a column. |
| `rename_table` | `(old_name: str, new_name: str)` | Rename a table. |
| `drop_columns` | `(doctype: str, columns: list[str])` | Drop columns with `ALTER TABLE`. |
| `updatedb` | `(doctype, meta=None)` | Sync the table to the DocType: make it, add and change columns and indexes. |
| `get_row_size` | `(doctype: str) -> int` | Return the estimated maximum row size in bytes. MariaDB only. |
| `get_database_size` | `()` | Return the database size in MB. |

## Sequences

| Method | Signature | Description |
|---|---|---|
| `create_sequence` | `(*args, **kwargs)` | Make a sequence. Used by `autoname: autoincrement`. |
| `get_next_sequence_val` | `(*args, **kwargs)` | Return the next value of a sequence. |
| `set_next_sequence_val` | `(*args, **kwargs)` | Set the next value of a sequence. |

## Error checks

Use these in `except` blocks to test a database error without database-specific codes. Each takes the exception and returns `bool`:

`is_deadlocked`, `is_timedout`, `is_statement_timeout`, `is_read_only_mode_error`, `is_table_missing`, `is_missing_table`, `is_missing_column`, `is_missing_table_or_column`, `is_duplicate_entry`, `is_duplicate_fieldname`, `is_primary_key_violation`, `is_unique_key_violation`, `is_data_too_long`, `is_db_table_size_limit`, `is_syntax_error`, `is_access_denied`, `cant_drop_field_or_key`, `is_interface_error`.

`is_column_missing` is deprecated. Use `is_missing_column`.

```python
try:
	frappe.db.sql("insert into `tabSample Item` (name) values ('Keyboard')")
except Exception as e:
	if not frappe.db.is_duplicate_entry(e):
		raise
```

## Differences between databases

| Method | MariaDB | Postgres | SQLite |
|---|---|---|---|
| `advisory_lock` | `GET_LOCK` with a hashed key | `pg_advisory_lock`, polled until `timeout` | Raises `NotImplementedError` |
| `transaction_advisory_lock` | Raises `NotImplementedError` | `pg_advisory_xact_lock` | Raises `NotImplementedError` |
| `set_execution_timeout` | `set session max_statement_time` | `set local statement_timeout` (ends with the transaction) | `PRAGMA busy_timeout` |
| `bulk_insert` | Multi-row `INSERT` | `COPY ... FROM STDIN`, default `chunk_size=10_000`. Falls back to `INSERT` with `ignore_duplicates=True` | Multi-row `INSERT` |
| `add_index` | Ignores `using`, `where`, `include` | Supports trigram (`using`), partial (`where`) and covering (`include`) indexes | Skips a `using` index. Ignores `where` and `include` |
| `get_row_size` | Supported | Raises `NotImplementedError` | Raises `NotImplementedError` |
| `truncate` | `TRUNCATE TABLE` | `TRUNCATE TABLE` | `DELETE FROM` and resets `sqlite_sequence` |
| `drop_columns` | One `ALTER TABLE` | One `ALTER TABLE` | One statement per column in a savepoint. Drops indexes on those columns first |

## Related

- [Database](../../developer-guide/database/database.md)
- [Query Builder](../../developer-guide/database/query-builder.md)
- [Python API](python-api.md)
- [Performance](../../developer-guide/testing-and-performance/performance.md)
