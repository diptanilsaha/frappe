---
title: Database API
description: Methods on frappe.db with signatures, grouped by theme, and the differences between MariaDB and Postgres.
version: v15
sources:
  - frappe/database/database.py
  - frappe/database/mariadb/database.py
  - frappe/database/postgres/database.py
  - frappe/database/__init__.py
  - frappe/database/schema.py
---

# Database API

`frappe.db` is the database connection of the current site. It is an instance of `MariaDBDatabase` or `PostgresDatabase`. Both extend `Database` in `frappe/database/database.py`. Use this page to find a method and its exact signature. For usage, filters and transactions, see [Database](../../developer-guide/database/database.md).

The signatures come from `inspect.signature` on a v15 site (MariaDB 10.6). Internal helpers are left out. `frappe.db.db_type` returns `"mariadb"` or `"postgres"`. The `db_type` key in site config selects the class. Any value other than `postgres` gives MariaDB. The MariaDB class uses the PyMySQL driver, so database errors are `pymysql.err` classes.

!!! note
    Methods on this page do not check permissions, unless the description says so. `set_value`, `delete` and `sql` do not run controller hooks.

## Read values

| Method | Signature | Description |
|---|---|---|
| `get_value` | `(doctype, filters=None, fieldname='name', ignore=None, as_dict=False, debug=False, order_by='KEEP_DEFAULT_ORDERING', cache=False, for_update=False, *, run=True, pluck=False, distinct=False, skip_locked=False, wait=True)` | Return one value, or a tuple of values if `fieldname` is a list, from the first matching row. `filters` is a name, a dict or a list. Use `None` for a Single DocType. |
| `get_values` | `(doctype, filters=None, fieldname='name', ignore=None, as_dict=False, debug=False, order_by='KEEP_DEFAULT_ORDERING', update=None, cache=False, for_update=False, *, run=True, pluck=False, distinct=False, limit=None, skip_locked=False, wait=True)` | Return values from all matching rows. |
| `get` | `(doctype, filters=None, as_dict=True, cache=False)` | Return `get_value` with `fieldname='*'`. |
| `get_single_value` | `(doctype, fieldname, cache=True)` | Return a field of a Single DocType, cast to its fieldtype. Cached in `frappe.db.value_cache` for the transaction by default. |
| `get_singles_value` | `(*args, **kwargs)` | Alias of `get_single_value`. |
| `get_singles_dict` | `(doctype, debug=False, *, for_update=False, cast=False)` | Return all fields of a Single DocType as a dict. |
| `get_list` | `(*args, **kwargs)` | Calls `frappe.get_list`. Checks permissions. |
| `get_all` | `(*args, **kwargs)` | Calls `frappe.get_all`. Does not check permissions. |
| `exists` | `(dt, dn=None, cache=False)` | Return the name of a matching document, or `None`. `dn` is a name or a dict of filters. `cache` works only when `dt` and `dn` are strings. |
| `count` | `(dt, filters=None, debug=False, cache=False, distinct: bool = True)` | Return `COUNT(*)` for the DocType and filters. With `cache=True` and no filters, the count is cached in Redis for 24 hours. |
| `estimate_count` | `(doctype: str)` | Return the estimated row count from table statistics. Not cached. |
| `a_row_exists` | `(doctype)` | Return `True` if the table has at least one row. |
| `get_creation_count` | `(doctype, minutes)` | Return the number of records created in the last `minutes`. |
| `get_last_created` | `(doctype)` | Return the `creation` timestamp of the last record, or `None`. |
| `get_descendants` | `(doctype, name)` | Return the descendants of a node in a tree DocType. |

## Write values

| Method | Signature | Description |
|---|---|---|
| `set_value` | `(dt, dn, field, val=None, modified=None, modified_by=None, update_modified=True, debug=False)` | Update columns with SQL. No controller hooks or validation. Updates `modified` and `modified_by` unless `update_modified=False`. `field` can be a dict of field and value. On a Single DocType it warns that this is deprecated and calls `set_single_value`. |
| `set_single_value` | `(doctype: str, fieldname: str \| dict, value=None, *, modified=None, modified_by=None, update_modified=True, debug=False)` | Set fields of a Single DocType. |
| `bulk_update` | `(doctype: str, doc_updates: dict, *, chunk_size: int = 100, modified=None, modified_by=None, update_modified=True, debug=False)` | Update many documents. `doc_updates` maps each name to a dict of values. |
| `bulk_insert` | `(doctype: str, fields: list[str], values: Iterable[Sequence], ignore_duplicates=False, *, chunk_size=10000)` | Insert many rows with one multi-row `INSERT` per chunk. No controller hooks. |
| `delete` | `(doctype: str, filters: dict \| list \| None = None, debug=False, **kwargs)` | Run `DELETE` for matching rows. No controller hooks. |
| `truncate` | `(doctype: str)` | Run `TRUNCATE` through `sql_ddl`. This commits first and cannot be rolled back. |

## Raw SQL

| Method | Signature | Description |
|---|---|---|
| `sql` | `(query, values=<no value>, *, as_dict=0, as_list=0, debug=0, ignore_ddl=0, auto_commit=0, update=None, explain=False, run=True, pluck=False, as_iterator=False)` | Run a query and fetch all rows. Pass parameters in `values`, never with string formatting. `query` can also be a query builder object. |
| `sql_list` | `(query, values=(), debug=False, **kwargs)` | Return the first column of each row as a list. |
| `sql_ddl` | `(query, debug=False)` | Commit, then run a DDL query. |
| `multisql` | `(sql_dict, values=(), **kwargs)` | Run the query for the current `db_type` from a dict like `{"mariadb": ..., "postgres": ...}`. There is no fallback key. |
| `mogrify` | `(query, values)` | Return the query string with values substituted. |
| `escape` | `(s, percent=True)` | Escape quotes, and `%` if `percent=True`. |
| `explain_query` | `(query, values=None)` | Print `EXPLAIN` of the query in the error log. |
| `unbuffered_cursor` | `()` | Context manager. Use a cursor that streams rows instead of fetching all of them. Use it with `sql(..., as_iterator=True)`. |
| `get_description` | `()` | Return the column metadata of the last query. |

`frappe.db` has no `qb` attribute in v15. Use `frappe.qb`. See [Query Builder](../../developer-guide/database/query-builder.md).

## Transactions

See [Database](../../developer-guide/database/database.md) for when Frappe commits for you.

| Method | Signature | Description |
|---|---|---|
| `begin` | `(*, read_only=False)` | Start a transaction. |
| `commit` | `()` | Commit and start a new transaction. Runs `before_commit` and `after_commit` callbacks. Clears `value_cache`. |
| `rollback` | `(*, save_point=None)` | Roll back the transaction, or to a savepoint. Runs `before_rollback` and `after_rollback` callbacks when there is no savepoint. |
| `savepoint` | `(save_point)` | Make a savepoint. Rollback to it undoes only database changes, not files. |
| `release_savepoint` | `(save_point)` | Release a savepoint. |
| `execution_timeout` | `(seconds: float)` | Context manager. Sets a statement timeout for the block, then restores the old one. |
| `set_execution_timeout` | `(seconds: float)` | Set the statement timeout of the session. |
| `get_execution_timeout` | `() -> float` | Return the statement timeout in seconds. `0.0` means no limit. |
| `before_commit` | `CallbackManager` | `frappe.db.before_commit.add(fn)` runs `fn` before the next commit. |
| `after_commit` | `CallbackManager` | `frappe.db.after_commit.add(fn)` runs `fn` after the next commit. |
| `before_rollback` | `CallbackManager` | Runs callbacks before the next rollback. |
| `after_rollback` | `CallbackManager` | Runs callbacks after the next rollback. |

`frappe.database.database.savepoint(catch=Exception)` is a context manager and decorator. It wraps a block in a savepoint and rolls back to it if `catch` is raised. `frappe.database` also exports it.

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
| `add_index` | `(doctype: str, fields: list, index_name: str \| None = None)` | Make an index if it does not exist. On MariaDB it commits the transaction first. |
| `add_unique` | `(doctype, fields, constraint_name=None)` | Add a unique constraint. |
| `has_index` | `(table_name, index_name)` | Return `True` if the index exists. |
| `get_column_index` | `(table_name: str, fieldname: str, unique: bool = False)` | Return the index on a column, if any. |
| `change_column_type` | `(doctype: str, column: str, type: str, nullable: bool = False)` | Change the type of a column. |
| `rename_column` | `(doctype: str, old_column_name, new_column_name)` | Rename a column. |
| `rename_table` | `(old_name: str, new_name: str)` | Rename a table. |
| `updatedb` | `(doctype, meta=None)` | Sync the table to the DocType: make it, add and change columns and indexes. |
| `get_row_size` | `(doctype: str) -> int` | Return the estimated maximum row size in bytes. MariaDB only. |
| `get_database_size` | `()` | Return the database size in MB. |

## Sequences

| Method | Signature | Description |
|---|---|---|
| `create_sequence` | `(*args, **kwargs)` | Make a sequence. Used by `autoname: autoincrement`. Calls `frappe/database/sequence.py`. |
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

| Method | MariaDB | Postgres |
|---|---|---|
| `estimate_count` | `table_rows` from `information_schema.tables` | `reltuples` from `pg_class` |
| `set_execution_timeout` | `set session max_statement_time` | `set local statement_timeout` (ends with the transaction) |
| `bulk_insert` with `ignore_duplicates=True` | `INSERT IGNORE` | `ON CONFLICT DO NOTHING` |
| `add_index` | If the index is missing: commit, then `ALTER TABLE ... ADD INDEX IF NOT EXISTS` | `CREATE INDEX IF NOT EXISTS` |
| `get_row_size` | Supported | Raises `NotImplementedError` |
| `escape` | PyMySQL `escape_string`. Also escapes backticks | psycopg2 `QuotedString`. `None` becomes an empty string |

## Related

- [Database](../../developer-guide/database/database.md)
- [Query Builder](../../developer-guide/database/query-builder.md)
- [Python API](python-api.md)
- [Performance](../../developer-guide/testing-and-performance/performance.md)
