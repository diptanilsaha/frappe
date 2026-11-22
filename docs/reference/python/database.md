---
title: frappe.db
description: Every public method and attribute of frappe.db with signatures, defaults, return values, side effects and the differences between MariaDB, Postgres and SQLite.
version: develop
sources:
  - frappe/database/database.py
  - frappe/database/mariadb/database.py
  - frappe/database/postgres/database.py
  - frappe/database/sqlite/database.py
  - frappe/database/__init__.py
  - frappe/database/utils.py
  - frappe/database/sequence.py
  - frappe/database/query.py
  - frappe/deprecation_dumpster.py
  - frappe/__init__.py
---

# frappe.db

`frappe.db` is the database connection of the current site. Use this page to look up the exact signature, return value and side effects of a method. For how to read, filter and write data, and for when Frappe commits, see [Database](../../developer-guide/database/database.md). For `frappe.db.qb` and `frappe.qb.get_query`, see [Query Builder reference](query-builder.md).

## Classes

`frappe.database.get_db()` makes the connection object from `db_type` in the site config. `frappe.connect()` stores it in `frappe.local.db`.

| `db_type` | Class | Module |
|---|---|---|
| `mariadb` (default) | `MariaDBDatabase(MariaDBConnectionUtil, MariaDBExceptionUtil, Database)` | `frappe/database/mariadb/database.py` |
| `postgres` | `PostgresDatabase(PostgresExceptionUtil, Database)` | `frappe/database/postgres/database.py` |
| `sqlite` | `SQLiteDatabase(SQLiteExceptionUtil, Database)` | `frappe/database/sqlite/database.py` |

All three extend `Database` in `frappe/database/database.py`. A method that the base class does not implement raises `NotImplementedError` on an engine that does not override it.

Unless a row says so, a method:

- does not check permissions,
- does not run controller methods, validations or `doc_events`,
- does not commit.

The signatures below come from `inspect.signature` on a develop site (MariaDB) and from the code of the other engines.

## Attributes

| Attribute | Type | Description |
|---|---|---|
| `db_type` | `str` | `"mariadb"`, `"postgres"` or `"sqlite"`. |
| `cur_db_name` | `str` | Name of the connected database. |
| `qb` | query builder class | Query builder for the dialect of this connection. `frappe.qb` is bound to the site's primary database. Use `frappe.db.qb` when the connection can be a replica. |
| `transaction_writes` | `int` | Count of `INSERT`, `UPDATE` and `DELETE` statements in the open transaction. Reset by `commit` and `rollback`. |
| `MAX_WRITES_PER_TRANSACTION` | `int` | `200_000`. Overridden by `max_writes_per_transaction` in the site config. |
| `auto_commit_on_many_writes` | `int` | `0`. When true, Frappe commits instead of raising `frappe.TooManyWritesError` at the write limit. |
| `value_cache` | `defaultdict` | Request cache used by `cache=True` reads. Cleared on commit and rollback. |
| `in_read_only_transaction` | `bool` | `True` inside `read_only()`. |
| `last_query` | `str` | The last query with its values. Set on MariaDB and Postgres after the first query. |
| `before_commit`, `after_commit`, `before_rollback`, `after_rollback` | `CallbackManager` | Callback queues. See [Transactions](#transactions). |
| `type_map` | `dict` | Field type to `(column type, length)` for this engine. |
| `VARCHAR_LEN` | `int` | `140`. |
| `MAX_COLUMN_LENGTH` | `int` | `64` (`63` on Postgres). |
| `DEFAULT_COLUMNS` | `tuple` | `("name", "creation", "modified", "modified_by", "owner", "docstatus", "idx")`. |
| `CHILD_TABLE_COLUMNS` | `tuple` | `("parent", "parenttype", "parentfield")`. |
| `OPTIONAL_COLUMNS` | `tuple` | `("_user_tags", "_comments", "_assign", "_liked_by")`. |
| `ProgrammingError`, `OperationalError`, `InternalError`, `DataError`, `SQLError`, `TableMissingError`, `InterfaceError`, `SequenceGeneratorLimitExceeded` | exception class | Driver exception classes of the engine. SQLite has no `InterfaceError`. |
| `InvalidColumnName` | exception class | Subclass of `frappe.ValidationError`. |

## Read values

### get_value

```python
frappe.db.get_value(
    doctype: str,
    filters: str | int | bool | dict | list | None = None,
    fieldname: str | list[str] = "name",
    ignore: bool = False,
    as_dict: bool = False,
    debug: bool = False,
    order_by: str = "KEEP_DEFAULT_ORDERING",
    cache: bool = False,
    for_update: bool = False,
    *,
    run: bool = True,
    pluck: bool = False,
    distinct: bool = False,
    skip_locked: bool = False,
    wait: bool = True,
)
```

Return a value from the first matching row. It calls `get_values` with `limit=1`.

| Parameter | Description |
|---|---|
| `filters` | A document name, a dict, a list of filters, or a query builder criterion. Use `None` for a Single DocType. |
| `fieldname` | A column, a list of columns, or `"*"`. `"*"` returns a dict. |
| `ignore` | Return `None` when the table or column does not exist. |
| `as_dict` | Return a `frappe._dict`. |
| `order_by` | The default sorts on `creation desc` for a name or dict filter. For a list filter it uses the sort settings of the DocType. Pass `None` for no `ORDER BY`. |
| `cache` | Keep the result in `value_cache` for the request. Works only when `filters` is a name. |
| `for_update` | Add `FOR UPDATE`. The row stays locked until commit or rollback. |
| `skip_locked`, `wait` | Add `SKIP LOCKED`, or `NOWAIT` when `wait=False`. `wait` is ignored for a list filter. |
| `run` | `False` returns the SQL with placeholders, not the result. |
| `pluck`, `distinct` | Passed to the query. `distinct=True` removes `ORDER BY`. |

Returns a single value when one field is asked for, a tuple for a list of fields, a dict with `as_dict=True`, or `None` when no row matches.

```python
frappe.db.get_value("Sample Item", "Laptop", "rate")                       # 900.0
frappe.db.get_value("Sample Item", "Laptop", ["item_name", "rate"])        # ('Laptop', 900.0)
frappe.db.get_value("Sample Item", {"category": "Hardware"}, "name")       # 'Keyboard'
frappe.db.get_value("System Settings", None, "date_format")                # 'dd-mm-yyyy'
frappe.db.get_value("Sample Item", "Laptop", "rate", run=False)
# 'SELECT `rate` FROM `tabSample Item` WHERE `name`=%(param1)s ORDER BY `creation` DESC LIMIT 1'
```

### Other read methods

| Method | Description |
|---|---|
| `get_values(doctype: str, filters=None, fieldname="name", ignore=False, as_dict=False, debug=False, order_by="KEEP_DEFAULT_ORDERING", update: dict \| None = None, cache=False, for_update=False, *, run=True, pluck=False, distinct=False, limit: int \| None = None, skip_locked=False, wait=True)` | Return all matching rows as a tuple of tuples, a list of dicts with `as_dict=True`, or a list with `pluck=True`. `update` adds keys to each dict row. Same filters and ordering as `get_value`. |
| `get(doctype, filters=None, as_dict=True, cache=False)` | Return `get_value(doctype, filters, "*")`. A dict of all columns. |
| `get_single_value(doctype: str, fieldname: str, cache: bool = True, *, debug=False, for_update=False, run=True)` | Return a field of a Single DocType, cast to its field type. Cached in `value_cache` by default. Raises `frappe.ValidationError` if the field does not exist. |
| `get_singles_value(*args, **kwargs)` | Alias of `get_single_value`. |
| `get_singles_dict(doctype, debug=False, *, for_update=False, cast=False)` | Return all stored fields of a Single DocType as a `frappe._dict`. Values are strings unless `cast=True`. |
| `get_values_from_single(fields, filters, doctype, as_dict=False, debug=False, update=None, *, run=True, pluck=False, distinct=False)` | Internal helper of `get_values` for Single DocTypes. |
| `get_list(*args, **kwargs)` | Calls `frappe.get_list`. Checks permissions. See [frappe](frappe.md). |
| `get_all(*args, **kwargs)` | Calls `frappe.get_all`. Does not check permissions. |
| `exists(dt, dn=None, cache=False, *, debug=False)` | Return the name of a matching document, or `None`. `dn` is a name or a dict of filters. `dt` can be a dict with a `doctype` key. Adds no `ORDER BY`. For a Single DocType with `dt == dn`, returns `dn` without a query. `cache` works only when `dt` and `dn` are strings. |
| `count(dt, filters=None, debug=False, cache=False, distinct: bool = True)` | Return `COUNT(*)` for the filters. `cache` works only without filters. |
| `estimate_count(doctype: str) -> int` | Return the row count from table statistics. Cached in Redis for 60 minutes. On SQLite it runs `COUNT(*)` with no cache. |
| `a_row_exists(doctype)` | Return a non-empty result if the table has at least one row. The value is a list of rows, not a `bool`. |
| `get_creation_count(doctype, minutes)` | Return the number of records with `creation` in the last `minutes`. |
| `get_last_created(doctype)` | Return the latest `creation` as a `datetime`, or `None`. |
| `get_descendants(doctype, name)` | Return the names below a node in a tree DocType. Returns `[]` if the node does not exist. |
| `get_system_setting(key)` | Return `frappe.get_system_settings(key)`. |

```python
frappe.db.get_values("Sample Item", {"category": "Hardware"}, ["name", "rate"], limit=2)
# (('Keyboard', 45.0), ('Monitor', 250.0))
frappe.db.exists("Sample Item", {"item_name": "Laptop"})    # 'Laptop'
frappe.db.count("Sample Item", {"category": "Hardware"})    # 3
```

## Write values

| Method | Description |
|---|---|
| `set_value(dt: str, dn: str \| int \| bool \| dict, field: str, val=None, modified=None, modified_by=None, update_modified=True, debug=False)` | Run an `UPDATE`. `dn` is a name or filters. `field` can be a dict of field and value. Sets `modified` and `modified_by` unless `update_modified=False`. Clears the document cache of `dn`, or of the whole DocType when `dn` is a filter. |
| `set_single_value(doctype: str, fieldname: str \| dict, value: str \| int \| None = None, *, modified=None, modified_by=None, update_modified=True, debug=False)` | Delete and insert the rows of the fields in `tabSingles`. Clears the document cache of the Single DocType. |
| `bulk_update(doctype: str, doc_updates: dict, *, chunk_size: int = 100, modified: str \| None = None, modified_by: str \| None = None, update_modified: bool = True, debug: bool = False)` | Update many documents with one `UPDATE ... CASE` per chunk. `doc_updates` maps each name to a dict of values. Does not clear the document cache. |
| `bulk_insert(doctype: str, fields: list[str], values: Iterable[Sequence], ignore_duplicates=False, *, chunk_size=1000)` | Insert many rows with one `INSERT` per chunk. Does not set `creation`, `modified` or `owner`. `ignore_duplicates` uses `INSERT IGNORE` (MariaDB, SQLite) or `ON CONFLICT DO NOTHING` (Postgres). |
| `delete(doctype: str, filters: dict \| list \| None = None, debug=False, **kwargs)` | Run `DELETE` for the matching rows. Without filters it deletes all rows. `conditions=` is an old name for `filters`. |
| `truncate(doctype: str)` | Commit, then run `TRUNCATE`. It cannot be rolled back. |

```python
frappe.db.set_value("Sample Item", "Keyboard", {"rate": 50, "disabled": 0}, update_modified=False)
frappe.db.set_single_value("System Settings", "country", "India")
frappe.db.bulk_update("Sample Item", {"Keyboard": {"rate": 40}, "Monitor": {"rate": 240}})
```

!!! warning
    `bulk_update` and `bulk_insert` leave the document cache as it is. `frappe.get_cached_doc` can return old values until the cache expires. Call `frappe.clear_document_cache(doctype, name)` after them.

## Raw SQL

### sql

```python
frappe.db.sql(
    query: str | QueryBuilder,
    values: tuple | list | dict | None = <no value>,
    *,
    as_dict=0, as_list=0, debug=0, ignore_ddl=0, auto_commit=0,
    update=None, explain=False, run=True, pluck=False, as_iterator=False,
)
```

| Parameter | Description |
|---|---|
| `query` | SQL with `%s` or `%(key)s` placeholders, or a query builder object. A query builder object is converted with `walk()`, so its values are bound. |
| `values` | Values for the placeholders. A single value that is not a tuple, list or dict is wrapped in a tuple. |
| `as_dict` | Return a list of `frappe._dict`. |
| `as_list` | Return a list of lists. |
| `pluck` | Return a list of the first column. |
| `update` | Dict to merge into each row. Works with `as_dict`. |
| `as_iterator` | Return a generator that fetches 1,000 rows at a time. Needs `as_dict` or `as_list` (or `pluck`). Use it in `unbuffered_cursor()` to keep memory low. |
| `debug` | Log the query and its run time. |
| `explain` | Return `None` without running the query. With `debug=True`, also log `EXPLAIN` of a `SELECT`. |
| `ignore_ddl` | Return `()` instead of raising when the table, column or key is missing. |
| `auto_commit` | Commit before and after the query. |
| `run` | `False` returns the query string. |

Returns a tuple of tuples by default, or `()` for a statement without a result set.

Side effects:

- `ifnull(` is rewritten to `coalesce(`.
- A deadlock raises `frappe.QueryDeadlockError`. A lock wait timeout raises `frappe.QueryTimeoutError`. A write in read-only mode raises `frappe.InReadOnlyMode`.
- A write after more than `MAX_WRITES_PER_TRANSACTION` writes raises `frappe.TooManyWritesError`.
- On MariaDB, `START`, `BEGIN`, `CREATE`, `ALTER`, `DROP` or `TRUNCATE` after a write raises `frappe.exceptions.ImplicitCommitError`. Use `sql_ddl` for DDL.
- In a Server Script, `frappe.db.sql` allows only `SELECT` and `EXPLAIN` queries. It cannot read `__Auth`.

```python
frappe.db.sql(
    "select name, rate from `tabSample Item` where category=%s limit 2", "Hardware", as_dict=True
)
# [{'name': 'Keyboard', 'rate': 45.0}, {'name': 'Laptop', 'rate': 900.0}]

with frappe.db.unbuffered_cursor():
    for row in frappe.db.sql("select name from `tabSample Item`", as_iterator=True, as_dict=True):
        print(row.name)
```

!!! warning
    Pass values in `values`. Never put user input in the query with string formatting.

### Other SQL helpers

| Method | Description |
|---|---|
| `sql_list(query, values=(), debug=False, **kwargs)` | Return `sql(..., pluck=True)`. |
| `sql_ddl(query, debug=False)` | Commit, then run the query. It commits even inside `doc_events`. On SQLite it also commits after the query. |
| `multisql(sql_dict, values=(), **kwargs)` | Run the query for the current `db_type` from a dict such as `{"mariadb": ..., "postgres": ..., "*": ...}`. `"*"` is the fallback. |
| `mogrify(query, values)` | Return the query with values filled in. For logs only. |
| `lazy_mogrify(query, values)` | Return a `LazyMogrify` object. It builds the string only when you call `str()` on it. |
| `escape(s, percent=True)` | Return `s` as a quoted SQL string literal. Doubles `%` when `percent=True`. |
| `explain_query(query, values=<no value>)` | Log `EXPLAIN` of the query. |
| `unbuffered_cursor()` | Context manager. Uses a server-side cursor for the block. Read all rows inside the block. |
| `get_description()` | Return the cursor description (column metadata) of the last query. |
| `execute_query(query, values=None)` | Run the query on the cursor without the checks of `sql`. Internal. |
| `fetch_as_dict(result) -> list[_dict]` | Convert rows of the last query to dicts. Internal. |
| `convert_to_lists(res)` | Convert tuples to lists. Internal. |
| `log_query(query, query_type, values=None, debug=False)` | Log a query as `sql` does. Internal. |
| `log_touched_tables(query, query_type)` | Record written tables in `frappe.flags.touched_tables` during migrate. Internal. |
| `check_transaction_status(query: str, query_type: str \| None = None)` | Apply the write limit and the implicit-commit check. Internal. |
| `check_implicit_commit(query: str, query_type: str)` | Raise `ImplicitCommitError` for implicit-commit statements after writes. MariaDB only. Postgres and SQLite do nothing. |
| `format_date(date)` | Return `"YYYY-MM-DD"`. |
| `format_datetime(datetime)` | Return `"YYYY-MM-DD HH:MM:SS.ffffff"`, or `"0001-01-01 00:00:00.000000"` for an empty value. SQLite uses `isoformat(" ")`. |

```python
frappe.db.escape("a'b%c")                 # "'a\\'b%%c'"  (MariaDB)
frappe.db.escape("a'b%c", percent=False)  # "'a\\'b%c'"
frappe.db.mogrify("select %s, %s", ("x", 1))   # "select 'x', 1"
```

## Transactions

See [Database](../../developer-guide/database/database.md) for when Frappe commits and rolls back for you.

| Method | Description |
|---|---|
| `begin(*, read_only=False)` | Run `START TRANSACTION`, or `START TRANSACTION READ ONLY`. `frappe.flags.read_only` also makes it read-only. |
| `commit(*, chain=False)` | Run `before_commit` callbacks, `COMMIT`, then start a new transaction. Clears `value_cache` and the rollback callbacks. Then runs `after_commit` callbacks. Inside `read_only()` it rolls back instead. |
| `rollback(*, save_point=None, chain=False)` | Run `before_rollback` callbacks, `ROLLBACK`, then start a new transaction. Clears the commit callbacks. Then runs `after_rollback` callbacks. With `save_point`, roll back to that savepoint only and run no callbacks. |
| `savepoint(save_point)` | Run `SAVEPOINT <save_point>`. |
| `release_savepoint(save_point)` | Run `RELEASE SAVEPOINT <save_point>`. |
| `read_only()` | Context manager. Rolls back, then runs the block in read-only transactions. A write raises `frappe.InReadOnlyMode`. Raises `frappe.ValidationError` if the open transaction has writes or `after_commit` callbacks. Rolls back at the end. |
| `run_after_transaction_callbacks(callbacks)` | Run the queued callbacks. A failing callback is logged in Error Log and the rest still run. |
| `execution_timeout(seconds: float)` | Context manager. Sets the statement timeout for the block, then restores the old value. |
| `set_execution_timeout(seconds: float)` | Set the statement timeout of the session. |
| `get_execution_timeout() -> float` | Return the statement timeout in seconds. `0` means no timeout. |
| `advisory_lock(key, *, timeout=10)` | Context manager. Holds a session-level named lock for the block. Polls every 0.1 seconds. Raises `frappe.QueryTimeoutError` after `timeout` seconds. |
| `transaction_advisory_lock(key, *, timeout=10)` | Take a lock that the database releases when the transaction ends. Postgres only. |

A commit or rollback is ignored with a warning while `_disable_transaction_control` is set. `Document.run_method` sets it while controller hooks and `doc_events` run.

Callbacks:

```python
frappe.db.after_commit.add(lambda: frappe.publish_realtime("sample_event"))
frappe.db.before_rollback.add(cleanup_files)
```

`frappe.database.database.savepoint(catch=Exception)` is a context manager and a decorator. It makes a savepoint with a random name, rolls back to it when `catch` is raised, and suppresses the exception:

```python
from frappe.database.database import savepoint

for name in ["Keyboard", "Webcam"]:
	with savepoint(catch=frappe.DuplicateEntryError):
		frappe.get_doc({"doctype": "Sample Item", "item_name": name}).insert()
```

!!! note
    A rollback to a savepoint undoes database changes only. Files written in the block stay.

Other helpers in `frappe/database/utils.py`:

| Function | Description |
|---|---|
| `commit_after_response(func)` | In a request, run `func` and commit after the response is sent. Each function runs in a savepoint, and errors go to Error Log. Outside a request or in tests, runs `func` at once. |
| `dangerously_reconnect_on_connection_abort(func)` | Decorator. After an `InterfaceError` or `OperationalError`, reconnects and calls `func` once more. The open transaction is lost. Use it only for logging or final steps of a job. |
| `drop_index_if_exists(table: str, index: str)` | Drop an index with `sql_ddl` if it exists. Prints the result. |

`frappe.database.database.get_query_execution_timeout()` returns the timeout set at connect time. It is `0` unless `enable_db_statement_timeout` is set in the site config. Then it is 1.5 times `http_timeout` (default 300) for a request, or 1.5 times the job timeout. See [Site Config](../configuration/site-config.md).

## Defaults and globals

These wrap `frappe.defaults`. See [Permissions and sessions](permissions-and-sessions.md).

| Method | Description |
|---|---|
| `get_default(key, parent="__default")` | Return a default value. Returns the first value if there are many. |
| `get_defaults(key=None, parent="__default")` | Return all defaults of `parent` as a dict, or the value of one key. Also tries `frappe.scrub(key)`. |
| `set_default(key, val, parent="__default", parenttype=None)` | Set a global or user default. |
| `add_default(key, val, parent="__default", parenttype=None)` | Add one more value for a key. |
| `get_global(key, user="__global")` | Return `get_default(key, user)`. |
| `set_global(key, val, user="__global")` | Call `set_default(key, val, user)`. |

## Schema

| Method | Description |
|---|---|
| `table_exists(doctype, cached=True)` | Return `True` if `tab<doctype>` exists. |
| `has_table(doctype)` | Same as `table_exists(doctype)`. |
| `get_tables(cached=True)` | Return all table names. Cached in the client cache. |
| `has_column(doctype, column)` | Return `True` if the column exists. Raises `TableMissingError` if the table does not exist. |
| `field_exists(dt, fn)` | Return the name of the DocField with fieldname `fn` in `dt`, or `None`. Custom Fields do not count. |
| `get_table_columns(doctype)` | Return the column names of `tab<doctype>`. Raises `TableMissingError` if the table does not exist. |
| `get_db_table_columns(table) -> list[str]` | Return the column names of a table by table name. Cached in the client cache. |
| `get_table_columns_description(table_name)` | Return a list of dicts with `name`, `type`, `default`, `index`, `unique`, `not_nullable`, `is_generated`. |
| `get_column_type(doctype, column)` | Return the column type, such as `decimal(21,9)` (MariaDB) or `numeric` (Postgres). |
| `describe(doctype: str)` | Return `DESC` on MariaDB, the column names on Postgres, `pragma_table_info` rows on SQLite. |
| `has_index(table_name, index_name)` | Return a non-empty result if the index exists. |
| `get_column_index(table_name: str, fieldname: str, unique: bool = False)` | Return the single-column index whose first column is `fieldname`, or `None`. |
| `get_index_name(fields)` | Return `"<field1>_<field2>_index"`. Removes prefix lengths such as `(10)`. |
| `add_index(doctype: str, fields: list, index_name: str \| None = None, using=None, where=None, include=None)` | Make an index if it does not exist. Commits first on MariaDB and Postgres. For one field outside install and migrate, it also makes a `search_index` Property Setter so that migrate keeps the index. |
| `add_unique(doctype, fields, constraint_name=None)` | Add a unique constraint if it does not exist. Commits first on MariaDB and Postgres. |
| `change_column_type(doctype: str, column: str, type: str, nullable: bool = False)` | Change the type of a column with `sql_ddl`. Postgres also takes `use_cast: bool = False`. |
| `rename_column(doctype: str, old_column_name, new_column_name)` | Rename a column with `sql_ddl`. |
| `rename_table(old_name: str, new_name: str)` | Rename `tab<old_name>` to `tab<new_name>`. On Postgres it also renames the indexes. |
| `drop_columns(doctype: str, columns: list[str])` | Drop columns with `sql_ddl`. |
| `updatedb(doctype, meta=None)` | Make or change the table to match the DocType: columns and indexes. Commits. Does nothing for a Single DocType. |
| `get_row_size(doctype: str) -> int` | Return the estimated maximum row size in bytes. MariaDB only. |
| `get_database_size()` | Return the database size in MB. |
| `get_database_list()` | Return the names of the databases on the server. |
| `get_routines()` | Return the names of stored functions and procedures. |
| `create_auth_table()`, `create_global_search_table()`, `create_user_settings_table()`, `create_sequence_table()` | Make the internal tables `__Auth`, `__global_search`, `__UserSettings` and the SQLite sequence table. Used by site setup. |
| `setup_type_map()` | Set `db_type` and `type_map`. Called by `__init__`. |
| `get_on_duplicate_update()` | Return `"ON DUPLICATE key UPDATE "` (MariaDB), `'ON CONFLICT ("name") DO UPDATE SET '` (Postgres, takes `key="name"`) or `"ON CONFLICT DO UPDATE SET "` (SQLite). |
| `is_type_number(code)`, `is_type_datetime(code)` | Return `True` if a cursor type code is a number or a datetime. |
| `clear_db_table_cache(query_type: str)` | Clear the cached table list after `CREATE` or `DROP`. Internal. |

## Sequences

Used by DocTypes with `autoname: autoincrement`. See [Naming](../../developer-guide/data-model/naming.md).

| Method | Description |
|---|---|
| `create_sequence(doctype_name: str, *, slug="_id_seq", temporary=False, check_not_exists=False, cycle=False, cache=0, start_value=0, increment_by=0, min_value=0, max_value=0) -> str` | Make the sequence `scrub(doctype_name + slug)` and return its name. Commits on MariaDB and Postgres (`sql_ddl`). |
| `get_next_sequence_val(doctype_name: str, slug="_id_seq") -> int` | Return the next value. Raises `SequenceGeneratorLimitExceeded` at the maximum. |
| `set_next_sequence_val(doctype_name: str, next_val: int, *, slug="_id_seq", is_val_used=False)` | Set the next value. With `is_val_used=True`, the next call returns `next_val + increment`. |

These call `create_sequence`, `get_next_val` and `set_next_val` in `frappe/database/sequence.py`.

## Connection

| Method | Description |
|---|---|
| `connect()` | Open the connection. Sets the statement timeout (see `get_query_execution_timeout`) and the session time zone. `sql` calls it when there is no connection. |
| `close()` | Close the connection. On Postgres it rolls back an open transaction first. |
| `use(db_name)` | Switch to another database. MariaDB only. |
| `get_connection()` | Return a new driver connection (PEP 249). |
| `set_session_time_zone(timezone: str)` | Set the session time zone so that `NOW()` matches the system time zone. |
| `create_connection()`, `get_connection_settings()` | MariaDB and SQLite connection helpers. Internal. |

## Error checks

Use these in an `except` block to test a database error without engine-specific codes. Each takes the exception and returns a `bool`.

| Method | `True` when |
|---|---|
| `is_deadlocked(e)` | The transaction was a deadlock victim. |
| `is_timedout(e)` | A lock wait timed out. |
| `is_statement_timeout(e)` | A statement ran longer than the statement timeout. |
| `is_read_only_mode_error(e)` | The transaction or server is read-only. |
| `is_table_missing(e)` | The table does not exist. |
| `is_missing_table(e)` | Same as `is_table_missing`. MariaDB and Postgres only. |
| `is_missing_column(e)` | The column does not exist. |
| `is_missing_table_or_column(e)` | Either of the two above. |
| `is_duplicate_entry(e)` | A unique or primary key was violated. |
| `is_primary_key_violation(e)` | The primary key was violated. |
| `is_unique_key_violation(e)` | A unique key was violated. |
| `is_duplicate_fieldname(e)` | A column with that name exists. |
| `is_data_too_long(e)` | A value is too long for the column. |
| `is_data_truncated(e)` | A value is out of range or of the wrong type. MariaDB and Postgres only. |
| `is_db_table_size_limit(e)` | The row size is above the limit. |
| `is_syntax_error(e)` | The SQL has a syntax error. |
| `is_access_denied(e)` | The database user has no access. |
| `cant_drop_field_or_key(e)` | A column or key to drop does not exist. |
| `is_interface_error(e)` | The connection is unusable. |
| `is_nested_transaction_error(e)` | A transaction is already active. SQLite only. |

```python
try:
	frappe.db.sql("insert into `tabSample Item` (name) values ('Laptop')")
except Exception as e:
	if not frappe.db.is_duplicate_entry(e):
		raise
```

## Differences between databases

| Topic | MariaDB | Postgres | SQLite |
|---|---|---|---|
| Implicit-commit check | `ImplicitCommitError` for DDL after writes | DDL runs in the transaction | DDL runs in the transaction |
| `sql` | Runs the query as written | Rewrites MariaDB syntax (backticks, `locate`). Rejects more than one statement in one query | Transpiles MariaDB SQL to SQLite |
| `set_execution_timeout` | `set session max_statement_time` | `set local statement_timeout` (ends with the transaction) | `PRAGMA busy_timeout` |
| `advisory_lock` | `GET_LOCK` on a SHA-256 key scoped to the database | `pg_try_advisory_lock`, polled | Raises `NotImplementedError` |
| `transaction_advisory_lock` | Raises `NotImplementedError` | `pg_try_advisory_xact_lock`, polled | Raises `NotImplementedError` |
| `bulk_insert` | Multi-row `INSERT` | `COPY ... FROM STDIN`, default `chunk_size=10_000`. Uses `INSERT` when `ignore_duplicates=True` | Multi-row `INSERT` |
| `add_index` | Skips the index when `using` is set. Ignores `where` and `include` | Supports `using` (`gin_trgm`, `gin_fulltext`, `btree`, `hash`, `gist`, `gin`, `brin`, `spgist`), partial (`where`) and covering (`include`) indexes. Index names include the table | Skips the index when `using` is set. Ignores `where` and `include` |
| `estimate_count` | `information_schema.tables.table_rows` | `pg_class.reltuples` | Exact `COUNT(*)`, not cached |
| `get_row_size` | Supported | Raises `NotImplementedError` | Raises `NotImplementedError` |
| `truncate` | `TRUNCATE` | `TRUNCATE` | `DELETE FROM` and resets `sqlite_sequence` |
| `drop_columns` | One `ALTER TABLE` | One `ALTER TABLE` | One statement per column. Drops indexes on those columns first. Commits |
| `begin` | `START TRANSACTION` | `START TRANSACTION` | `BEGIN IMMEDIATE` for a write request, else `BEGIN`. A read-only transaction opens a read-only connection |
| `use` | Supported | Not supported | Not supported |

## Deprecated

| Name | Use instead |
|---|---|
| `frappe.db.is_column_missing(e)` | `frappe.db.is_missing_column(e)` |
| `frappe.db.set_value(doctype, None, field, value)` on a Single DocType | `frappe.db.set_single_value(doctype, field, value)`. The old call still works and shows a deprecation warning. |

## Related

- [Database](../../developer-guide/database/database.md)
- [Query Builder reference](query-builder.md)
- [frappe](frappe.md)
- [Document](document.md)
- [Site Config](../configuration/site-config.md)
- [Performance](../../developer-guide/testing-and-performance/performance.md)
