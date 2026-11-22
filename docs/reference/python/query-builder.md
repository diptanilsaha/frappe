---
title: frappe.qb
description: Reference for frappe.qb and frappe.query_builder - tables, fields, terms, functions, Case, Interval, run() options and every argument of frappe.qb.get_query.
version: develop
sources:
  - frappe/query_builder/__init__.py
  - frappe/query_builder/builder.py
  - frappe/query_builder/utils.py
  - frappe/query_builder/functions.py
  - frappe/query_builder/custom.py
  - frappe/query_builder/terms.py
  - frappe/database/query.py
  - frappe/database/operator_map.py
  - frappe/database/utils.py
  - frappe/database/database.py
  - frappe/utils/safe_exec.py
  - frappe/boot.py
---

# frappe.qb

`frappe.qb` builds SQL from Python objects. It is a [PyPika](https://github.com/kayak/pypika) 0.51 dialect with Frappe additions. Use this page to look up a class, function or argument. For a guided introduction with joins, subqueries and updates, see [Query Builder](../../developer-guide/database/query-builder.md).

All SQL on this page was made on a develop site with MariaDB. Values are shown inline for reading. `run()` sends them as bound parameters.

!!! warning
    Queries built with `frappe.qb` skip permissions, validations, controller methods and `doc_events`. `UPDATE`, `INSERT` and `DELETE` do not set `modified` and do not clear the document cache. Only `frappe.qb.get_query(..., ignore_permissions=False)` checks permissions.

## The builder classes

`frappe.qb` is `frappe.local.qb`. `frappe.connect()` sets it to the class for `db_type` in the site config:

| `db_type` | Class | PyPika base |
|---|---|---|
| `mariadb` | `frappe.query_builder.builder.MariaDB` | `MySQLQuery` |
| `postgres` | `frappe.query_builder.builder.Postgres` | `PostgreSQLQuery` |
| `sqlite` | `frappe.query_builder.builder.SQLite` | `SQLLiteQuery` |

`frappe.db.qb` is the class for the dialect of that connection. Use it when the connection can be a replica or another database. `frappe.query_builder.get_query_builder(type_of_db: str)` returns the class for a `db_type`.

Class attributes and methods:

| Name | Description |
|---|---|
| `DocType(table_name: str, *args, **kwargs) -> Table` | Return the table `tab<table_name>`. Raises `frappe.ValidationError` ("Invalid DocType") if the name has characters other than letters, digits, `_`, space and `-`. |
| `Table(name, schema=None, alias=None)` | Return a PyPika table with the exact name, for tables such as `__Auth` or `tabSingles`. |
| `Schema(name)` | Return a schema. `frappe.qb.Schema("information_schema").tables` is a table in it. On Postgres, `information_schema.tables` maps to `pg_stat_all_tables`. |
| `Field(name, table=None, alias=None)` | Return a field. On Postgres, `table_name` maps to `relname` and `table_rows` to `n_tup_ins`. |
| `from_(table)` | Start a `SELECT`. A string is passed to `DocType()`. |
| `into(table)` | Start an `INSERT`. A string is passed to `DocType()`. |
| `update(table)` | Start an `UPDATE`. A string is passed to `DocType()`. |
| `select(*terms)` | Start a `SELECT` without a table. |
| `with_(table, name, recursive=False)` | Start a query with a `WITH` (CTE). `recursive=True` renders `WITH RECURSIVE`. |
| `get_query(...)` | Build a query from `fields` and `filters`. See [get_query](#get_query). |
| `max(dt, fieldname, filters=None, **kwargs)` | Return `MAX(fieldname)` for the filters, or `0`. `kwargs` go to `run()`. |
| `min(dt, fieldname, filters=None, **kwargs)` | Return `MIN(fieldname)`, or `0`. |
| `avg(dt, fieldname, filters=None, **kwargs)` | Return `AVG(fieldname)`, or `0`. |
| `sum(dt, fieldname, filters=None, **kwargs)` | Return `SUM(fieldname)`, or `0`. |
| `functions(name: str, *args, **kwargs) -> Function` | Return the SQL function `name` with the arguments. |
| `terms` | A `frappe._dict` of all public names in `pypika.terms`. |
| `asc`, `desc` | `Order.asc` and `Order.desc`. |
| `validate_doctype(doctype)` | The name check that `DocType()` runs. |
| `Tables(*names)`, `create_table`, `drop_table`, `create_index`, `drop_index`, `drop_database`, `drop_user`, `drop_view`, `load` | Inherited from PyPika `Query`. |

```python
frappe.qb.max("Sample Item", "rate")                                     # 900.0
frappe.qb.sum("Sample Item", "rate", filters={"category": "Hardware"})   # 1195.0

frappe.qb.from_(frappe.qb.Table("__Auth")).select("name").get_sql()
# SELECT `name` FROM `__Auth`
```

## The frappe.query_builder module

`frappe.query_builder` exports everything from `pypika` (with `from pypika import *`) and these Frappe names:

| Name | Description |
|---|---|
| `DocType(*args, **kwargs)` | Calls `frappe.qb.DocType`. |
| `Table(*args, **kwargs)` | Calls `frappe.qb.Table`. |
| `get_query(*args, **kwargs)` | Calls `frappe.database.query.Engine().get_query`. Same as `frappe.qb.get_query`. |
| `get_query_builder(type_of_db: str)` | Return the builder class for a `db_type`. |
| `Interval` | Frappe's `Interval`. It replaces PyPika's and also renders on SQLite. |
| `Field`, `Column` | PyPika `Field` and `Column`. |
| `ParameterizedValueWrapper`, `ParameterizedFunction` | Frappe classes that replace PyPika's `ValueWrapper` and `Function`, so values become bound parameters. |
| `patch_all()` | Apply the Frappe patches to PyPika. Runs on import. |

Common PyPika names from the same module: `Case`, `Criterion`, `EmptyCriterion`, `Order`, `Not`, `NULL`, `Tuple`, `Array`, `JSON`, `JoinType`, `CustomFunction`, `Parameter`, `Index`, `Rollup`, `Schema`, `AliasedQuery`, `Query`.

Importing `frappe.query_builder` patches PyPika:

- `table.field_name` and `table["field name"]` return a `Field` bound to the table.
- `query.run()` and `query.walk()` are added to every query and union.
- `.like()` and `.not_like()` render `ILIKE` and `NOT ILIKE` on Postgres.
- `.regex()` renders `REGEXP` on MariaDB and SQLite, and `~*` on Postgres.

## Tables and fields

```python
Item = frappe.qb.DocType("Sample Item")

Item.rate                 # Field "rate" of `tabSample Item`
Item["item_name"]         # same, for any name
Item.star                 # *
Item.as_("i")             # table alias: `tabSample Item` `i`
Item.rate.as_("price")    # field alias
```

## Criteria

| Python | SQL |
|---|---|
| `Item.rate == 10`, `!=`, `<`, `>`, `<=`, `>=` | `=`, `<>`, `<`, `>`, `<=`, `>=` |
| `(a) & (b)`, `(a) \| (b)` | `AND`, `OR` |
| `~(a)`, `Not(a)` | `NOT` |
| `Item.category.isin([...])`, `.notin([...])` | `IN`, `NOT IN`. The value can be a subquery. |
| `Item.name.like("L%")`, `.not_like("L%")` | `LIKE`, `NOT LIKE` (`ILIKE` on Postgres) |
| `Item.name.ilike("l%")` | `ILIKE` |
| `Item.name.regex("^L")` | `REGEXP` (MariaDB, SQLite), `~*` (Postgres) |
| `Item.description.isnull()`, `.isnotnull()` | `IS NULL`, `IS NOT NULL` |
| `Item.rate[50:150]`, `Item.rate.between(50, 150)` | `BETWEEN 50 AND 150` |
| `Tuple(Item.category, Item.disabled).isin([("Hardware", 0)])` | `(category, disabled) IN (('Hardware', 0))` |
| `Criterion.all([a, b])`, `Criterion.any([a, b])` | `a AND b`, `a OR b`. An empty list gives an `EmptyCriterion` that adds nothing. |

Put parentheses around each side of `&` and `|`. Python binds `&` before `==`.

`frappe.database.query.RawCriterion(sql_string)` is a criterion from a raw SQL string. It supports `&`, `|` and `~`. Never put user input in it.

## Query methods

These come from PyPika. Each returns a new query.

| Method | SQL |
|---|---|
| `.select(*terms)` | `SELECT`. A string is a column name. `"*"` is all columns. |
| `.where(criterion)` | `WHERE`. Many calls are joined with `AND`. |
| `.join(table, how=JoinType.inner).on(criterion)` | `JOIN ... ON`. |
| `.left_join(table)`, `.inner_join(table)`, `.right_join(table)`, `.outer_join(table)`, `.cross_join(table)` | Join of that type. |
| `.groupby(*terms)`, `.having(criterion)` | `GROUP BY`, `HAVING` |
| `.orderby(*terms, order=None)` | `ORDER BY`. Without `order`, no direction is written, so the database sorts `ASC`. |
| `.limit(n)`, `.offset(n)` | `LIMIT`, `OFFSET` |
| `.distinct()` | `SELECT DISTINCT` |
| `.for_update(nowait=False, skip_locked=False, of=())` | `FOR UPDATE`, with `NOWAIT` or `SKIP LOCKED` |
| `.delete()` | `DELETE` (after `from_`) |
| `.set(field, value)` | `SET` (after `update`) |
| `.columns(*names)`, `.insert(*rows)` | Columns and rows (after `into`) |
| `.ignore()` | Ignore duplicate rows. `frappe.db.bulk_insert` uses it on MariaDB and SQLite. |
| `.on_duplicate_key_update(field, value)` | `ON DUPLICATE KEY UPDATE` (MariaDB) |
| `.on_conflict(*fields).do_nothing()`, `.do_update(field, value=None)` | `ON CONFLICT` (Postgres, SQLite) |
| `query1 + query2`, `query1 * query2` | `UNION`, `UNION ALL` |
| `.get_sql()`, `str(query)` | The SQL with values inline. Use it to read the query only. |

```python
Item = frappe.qb.DocType("Sample Item")
(
    frappe.qb.from_(Item).select(Item.name).where(Item.rate > 500)
    + frappe.qb.from_(Item).select(Item.name).where(Item.rate < 50)
).run(pluck=True)
# ['Laptop', 'Keyboard']
```

## run() and walk()

`query.run(*args, **kwargs)` runs the query on `frappe.local.db`. To run on another connection, pass the query to that connection: `db.sql(query)`.

`run()` takes the keyword arguments of [frappe.db.sql](database.md#sql):

| Argument | Result |
|---|---|
| none | Tuple of tuples |
| `as_dict=True` | List of `frappe._dict` |
| `as_list=True` | List of lists |
| `pluck=True` | List of the first column |
| `update={...}` | Dict merged into each row. Needs `as_dict`. |
| `debug=True` | Logs the query and its run time |
| `run=False` | Returns the SQL with placeholders. The values are not returned. |
| `as_iterator=True` | Generator of rows. See `frappe.db.unbuffered_cursor`. |

After the query runs, `run()` also:

- runs the child queries of `get_query` and adds their rows (only with `as_dict=True`),
- masks the values of masked fields when the query came from `get_query(..., ignore_permissions=False)`,
- removes a `name` column that `get_query` added only for child queries.

`query.walk()` returns `(sql, params)`. The SQL has `%(paramN)s` placeholders. `frappe.db.sql(query)` calls `walk()` on a query object.

```python
frappe.qb.get_query("Sample Item", filters={"category": "Hardware"}).walk()
# ('SELECT `name` FROM `tabSample Item` WHERE `category`=%(param1)s', {'param1': 'Hardware'})
```

In a Server Script, `run()` raises `frappe.PermissionError` for a query that is not `SELECT` or `EXPLAIN`.

## Functions

Import functions from `frappe.query_builder.functions`. It exports all of `pypika.functions` and adds the names below. A name in the "Per database" column renders differently for each engine.

### Frappe functions

| Function | MariaDB | Postgres | SQLite |
|---|---|---|---|
| `GroupConcat(column, separator=",", alias=None)` | `GROUP_CONCAT(... SEPARATOR ',')` | `STRING_AGG(..., ',')` | `GROUP_CONCAT(..., ',')`. `DISTINCT` works only with `","` |
| `Match(column).Against(text)` | `MATCH(...) AGAINST ('+text*' IN BOOLEAN MODE)` | `TO_TSVECTOR('english', ...) @@ PLAINTO_TSQUERY('english', text)` | FTS5 `MATCH` |
| `Locate(needle, haystack)` | `LOCATE(needle, haystack)` | `STRPOS(haystack, needle)` | `INSTR(haystack, needle)` |
| `DateFormat(date, format)` | `DATE_FORMAT` | `TO_CHAR` | `STRFTIME`, or a Frappe function for other tokens |
| `Timestamp(term, time=None, alias=None)` | `TIMESTAMP(...)` | `TIMESTAMP(...)` | `DATETIME(...)` |
| `CombineDatetime(date, time)` | `TIMESTAMP(date, time)` | `date + time` (strings cast to `date` and `time`) | Frappe function |
| `UnixTimestamp(field)` | `unix_timestamp(field)` | `EXTRACT(EPOCH ...)` in the session time zone | Frappe function |
| `DateDiff(date1, date2)` | `DATEDIFF(date1, date2)` | `CAST(date1 AS date) - CAST(date2 AS date)` | `JULIANDAY` difference in days |
| `JSONExtract(field, path)` | `JSON_EXTRACT` | `field->path` | `JSON_EXTRACT` with MariaDB quoting |
| `JSONValue(field, path)` | `JSON_UNQUOTE(JSON_EXTRACT(...))` | `field->>path` | Unquoted scalar |
| `JSONContains(target, candidate)` | `JSON_CONTAINS` | `@>` | Frappe function |
| `Month(field)`, `Year(field)`, `Quarter(field)` | `MONTH`, `YEAR`, `QUARTER` | `date_part(...)` cast to integer | `STRFTIME` cast to integer |
| `MonthName(field)` | `MONTHNAME` | `to_char(field, 'FMMonth')` | `CASE` over `STRFTIME('%m')` |
| `Now()` | `NOW()` | `NOW()` | `NOW()` in the site time zone. `Now() + Interval(...)` uses `DATETIME` |
| `Coalesce(*terms)` | `COALESCE` | `COALESCE` | Returns the argument alone when there is one |
| `CurDate()` | `CURRENT_DATE` | `CURRENT_DATE` | `CURRENT_DATE` |
| `Cast_(value, as_type, alias=None)` | `CONCAT(value, '')` for `varchar`, else `CAST` | `CAST` | `CAST` |
| `Abs(term)` | `ABS` (scalar, not an aggregate) | same | same |
| `Round(term, decimal=0)` | `ROUND` | same | same |
| `Truncate(term, decimal)` | `TRUNCATE` | same | same |
| `Concat_ws(*terms)` | `CONCAT_WS` | same | same |
| `YearWeek(term)` | `YEARWEEK(term, 1)` | same | same |
| `Strpos(needle, haystack)`, `Instr(needle, haystack)` | `STRPOS(haystack, needle)`, `INSTR(haystack, needle)` | | |
| `Ifnull` | Alias of `IfNull` | | |

### PyPika functions

`Count`, `Sum`, `Avg`, `Max`, `Min`, `Std`, `StdDev`, `First`, `Last`, `ApproximatePercentile`, `IfNull`, `IsNull`, `NVL`, `NullIf`, `Coalesce`, `Concat`, `Upper`, `Lower`, `Length`, `Trim`, `Substring`, `Replace`, `Reverse`, `Insert`, `SplitPart`, `RegexpLike`, `RegexpMatches`, `Ascii`, `Bin`, `Floor`, `Sqrt`, `Cast`, `Convert`, `Signed`, `Unsigned`, `Date`, `DateAdd`, `TimeDiff`, `TimestampAdd`, `ToChar`, `ToDate`, `Extract`, `CurTime`, `CurTimestamp`, `UtcTimestamp`, `SqlTypes`.

### Custom functions and constants

| Name | Description |
|---|---|
| `CustomFunction(name, ["arg1", ...])` | Make a function class. From `frappe.query_builder`. |
| `frappe.qb.functions("UPPER", Item.name)` | Call any SQL function by name. |
| `frappe.query_builder.custom.ConstantColumn(value)` | A column with the same string in every row. Its default alias is the value. |
| `frappe.query_builder.terms.SubQuery(query, alias=None)` (also `subqry`) | Wrap a query as a criterion term, for `>` or `=` against a subquery. |

```python
from frappe.query_builder import CustomFunction
from frappe.query_builder.custom import ConstantColumn
from frappe.query_builder.functions import Count, GroupConcat

Item = frappe.qb.DocType("Sample Item")
DateOnly = CustomFunction("DATE", ["value"])

frappe.qb.from_(Item).select(
    Item.category,
    Count(Item.name).as_("count"),
    GroupConcat(Item.name, "; ").as_("names"),
).groupby(Item.category).get_sql()
# SELECT `category`,COUNT(`name`) `count`,GROUP_CONCAT(`name` SEPARATOR '; ') `names`
# FROM `tabSample Item` GROUP BY `category`

frappe.qb.from_(Item).select(DateOnly(Item.creation).as_("d"), ConstantColumn("Sample Item").as_("doctype")).get_sql()
# SELECT DATE(`creation`) `d`,'Sample Item' `doctype` FROM `tabSample Item`
```

## Case, Order and Interval

| Name | Use |
|---|---|
| `Case().when(criterion, value).else_(value)` | `CASE WHEN ... THEN ... ELSE ... END`. Chain more `.when()` calls. |
| `Order.asc`, `Order.desc` | The `order` argument of `.orderby()`. Also `frappe.qb.asc` and `frappe.qb.desc`. |
| `Interval(years=0, months=0, days=0, hours=0, minutes=0, seconds=0, microseconds=0, quarters=0, weeks=0, dialect=None)` | A date interval. Add it to or subtract it from a date term. |

```python
from frappe.query_builder import Case, Interval, Order
from frappe.query_builder.functions import Now

Item = frappe.qb.DocType("Sample Item")
band = Case().when(Item.rate >= 200, "High").else_("Low")

frappe.qb.from_(Item).select(Item.name, band.as_("band")).where(
    Item.creation > Now() - Interval(days=7)
).orderby(Item.rate, order=Order.desc).get_sql()
# SELECT `name`,CASE WHEN `rate`>=200 THEN 'High' ELSE 'Low' END `band` FROM `tabSample Item`
# WHERE `creation`>NOW()-INTERVAL '7' DAY ORDER BY `rate` DESC
```

## get_query

```python
frappe.qb.get_query(
    table: str | Table,
    fields: str | list | tuple | set | None = None,
    filters: dict | str | int | bool | list | Criterion | None = None,
    order_by: str | None = None,
    group_by: str | None = None,
    limit: int | None = None,
    offset: int | None = None,
    distinct: bool = False,
    for_update: bool = False,
    update: bool = False,
    into: bool = False,
    delete: bool = False,
    *,
    validate_filters: bool = False,
    skip_locked: bool = False,
    wait: bool = True,
    ignore_permissions: bool = True,
    ignore_user_permissions: bool = False,
    user: str | None = None,
    parent_doctype: str | None = None,
    reference_doctype: str | None = None,
    or_filters: dict | list | None = None,
    db_query_compat: bool = False,
) -> QueryBuilder
```

Build a query from the `fields` and `filters` syntax of `frappe.get_all`. The returned query is immutable. Each method you chain, such as `.where()`, returns a new query. `frappe.db.get_value`, `get_values`, `count`, `set_value` and `delete` use it.

`frappe.database.query.Engine(db=None).get_query(...)` is the class behind it. Pass `db` to build for a connection other than `frappe.local.db`.

| Argument | Default | Description |
|---|---|---|
| `table` | | DocType name or a `Table`. |
| `fields` | `["name"]` | Fields to select. See [Fields](#fields). |
| `filters` | `None` | Conditions joined with `AND`. See [Filters](#filters). |
| `order_by` | `None` | No `ORDER BY` when empty. `"KEEP_DEFAULT_ORDERING"` uses the sort settings of the DocType. See [Order and group](#order-and-group). |
| `group_by` | `None` | Comma-separated fields. |
| `limit` | `None` | Non-negative `int`. Another type raises `TypeError`. |
| `offset` | `None` | Non-negative `int`. Without `limit`, MariaDB gets `LIMIT 18446744073709551615` and SQLite gets `LIMIT -1`. |
| `distinct` | `False` | `SELECT DISTINCT`. On Postgres, an `order_by` field that is not selected is dropped with a `UserWarning`. |
| `for_update` | `False` | Add `FOR UPDATE`. |
| `skip_locked` | `False` | With `for_update`, add `SKIP LOCKED`. |
| `wait` | `True` | With `for_update`, `False` adds `NOWAIT`. |
| `update` | `False` | Return an `UPDATE` with the filters. Chain `.set()`. `fields` is ignored. |
| `into` | `False` | Return an `INSERT INTO`. Chain `.columns()` and `.insert()`. |
| `delete` | `False` | Return a `DELETE` with the filters. |
| `ignore_permissions` | `True` | `False` checks `select` or `read` permission, field permlevels, user permissions, `permission_query_conditions` and shares. Raises `frappe.PermissionError` when the user has no access. |
| `ignore_user_permissions` | `False` | With permissions on, skip User Permission conditions. |
| `user` | `frappe.session.user` | User for the permission checks. |
| `parent_doctype` | `None` | For a child DocType: check permissions on this parent and join it. |
| `reference_doctype` | `None` | DocType of a link field. Applies User Permissions with "Applicable For" this DocType. |
| `or_filters` | `None` | Conditions joined with `OR`, then added to `filters` with `AND`. |
| `db_query_compat` | `False` | `True` makes a field in `order_by` without a direction sort `ASC`, as in `frappe.get_list`. It also turns `None` in `in` and `not in` into `("",)`. |
| `validate_filters` | `False` | Deprecated. Not used. |

### Fields

| Form | Example | SQL |
|---|---|---|
| Field name | `"rate"` | `` `rate` `` |
| Comma-separated string | `"name, rate as price"` | `` `name`,`rate` `price` `` |
| Alias | `"rate as price"` | `` `rate` `price` `` |
| All columns | `"*"` | `*` (with permissions on: all permitted fields) |
| Table-qualified | ``"`tabSample Item`.`rate`"`` | `` `tabSample Item`.`rate` `` |
| Link field | `"item.category"` (in `Sample Order Item`) | `LEFT JOIN` on the linked DocType with an alias such as `` `tab_Sample Item_1` `` |
| Child table field | `"items.item"` (in `Sample Order`) | `LEFT JOIN` on the child table with `parent`, `parenttype` and `parentfield` |
| Child query | `{"items": ["item", "qty"]}` | A second query per run. Rows go in a list under `items`. Needs `run(as_dict=True)`. |
| Function | `{"COUNT": "*", "as": "n"}`, `{"SUM": "rate"}`, `{"IFNULL": ["description", "''"]}` | `COUNT(*) `n``, `SUM(`rate`)`, `IFNULL(`description`,'')` |
| Arithmetic | `{"MUL": ["rate", 2], "as": "double"}` | `` `rate`*2 `double` `` |
| Query builder term | `Count(Item.name)`, `Item.rate` | as built |

Function keys are `COUNT`, `SUM`, `AVG`, `MAX`, `MIN`, `ABS`, `EXTRACT`, `LOCATE`, `TIMESTAMP`, `IFNULL`, `CONCAT`, `NOW`, `NULLIF`, `MONTHNAME`, `QUARTER`, `MONTH`, `YEAR`. Arithmetic keys are `ADD`, `SUB`, `MUL` and `DIV`, with exactly two operands. Functions and operators can nest. A string argument is a field name, a number, `"*"` (`COUNT` only), or a quoted literal such as `"'text'"`. An alias must be a simple identifier.

A function written as a string, such as `"count(name) as c"`, raises `frappe.ValidationError`. Any other field string that is not a plain, backticked, table-qualified or aliased name raises `frappe.PermissionError`.

```python
frappe.qb.get_query(
    "Sample Item",
    fields=["category", {"COUNT": "*", "as": "n"}, {"SUM": "rate", "as": "total"}],
    group_by="category",
    order_by="total desc",
).run(as_dict=True)

frappe.qb.get_query("Sample Order", fields=["name", {"items": ["item", "qty"]}], limit=1, order_by="name asc").run(as_dict=True)
# [{'name': 'SO-00001', 'items': [{'item': 'Laptop', 'qty': 2.0}, {'item': 'Monitor', 'qty': 2.0}]}]
```

### Filters

| Form | Example | SQL |
|---|---|---|
| Name | `"Laptop"` | `name='Laptop'` |
| List of names | `["Laptop", "Monitor"]` | `name IN ('Laptop','Monitor')` |
| Dict | `{"category": "Hardware", "rate": [">", 100]}` | `category='Hardware' AND rate>100` |
| List of conditions | `[["rate", "between", [10, 100]], ["item_name", "like", "%a%"]]` | joined with `AND` |
| Condition with DocType | `[["Sample Order Item", "item", "=", "Laptop"]]` | joins the child table |
| Nested | `[["category", "=", "Hardware"], "or", [["rate", "<", 50], "and", ["disabled", "=", 0]]]` | `category='Hardware' OR (rate<50 AND disabled=0)` |
| Criterion | `Item.rate > 800` | as built |

A condition is `[field, value]`, `[field, operator, value]` or `[doctype, field, operator, value]`. A field can be a link or child table field (`"items.qty"`). A field that is not in the DocType but is in one of its child tables is looked up there, and the child table is joined.

Operators:

| Operator | Value | Notes |
|---|---|---|
| `=`, `!=`, `<`, `>`, `<=`, `>=`, `=<`, `=>` | value | `None` with `=` gives `IS NULL`. `None` with `!=` compares with the empty value of the field type. |
| `in`, `not in` | list, tuple, or comma-separated string | An empty list with `in` gives `1=0`. An empty list with `not in` gives `1=1`. `""` or `None` in an `in` list also matches `NULL`. |
| `like`, `not like`, `ilike` | pattern | `like` is `ILIKE` on Postgres. |
| `regex` | pattern | |
| `between` | `[from, to]` | Dates on a Datetime field become the start and end of the day. |
| `is` | `"set"` or `"not set"` | Compares with `NULL` and the empty value of the field type. |
| `timespan` | `"this month"`, `"last week"` and others | Becomes `between`. |
| `previous`, `next` | `"1 week"` and others | Becomes `between`. |
| `ancestors of`, `descendants of`, `not ancestors of`, `not descendants of`, `descendants of (inclusive)` | document name | For tree DocTypes. Uses `lft` and `rgt`. |
| `+`, `-`, `*`, `/` | value | Python arithmetic operators. |
| Custom | | Operators from the `filters_config` hook. See [Hooks](../configuration/hooks.md). |

A filter field name with characters other than letters, digits and `_` raises `frappe.ValidationError`. On `_assign` and `_liked_by`, `=` and `!=` become `like` and `not like` with `%value%`.

```python
frappe.qb.get_query(
    "Sample Item",
    filters={"disabled": 0},
    or_filters={"category": "Hardware", "rate": [">", 200]},
).get_sql()
# SELECT `name` FROM `tabSample Item` WHERE `disabled`=0 AND (`category`='Hardware' OR `rate`>200)
```

### Order and group

`order_by` and `group_by` take comma-separated fields: plain names, select aliases, result positions (`"1"`), link and child table fields, or backticked table-qualified names.

!!! warning
    In `order_by`, a field without a direction sorts `DESC`. `frappe.get_list` sorts it `ASC`. Pass `db_query_compat=True` or write the direction.

```python
frappe.qb.get_query("Sample Item", fields=["name"], order_by="rate").get_sql()
# SELECT `name` FROM `tabSample Item` ORDER BY `rate` DESC
frappe.qb.get_query("Sample Item", fields=["name"], order_by="KEEP_DEFAULT_ORDERING").get_sql()
# SELECT `name` FROM `tabSample Item` ORDER BY `modified` DESC
```

On Postgres with `group_by` or an aggregate, an `order_by` field that is not grouped or aliased is wrapped in `MAX()`.

### Permissions

With `ignore_permissions=False`, `get_query`:

1. Checks `select` permission on the DocType (or on `parent_doctype`) for `user`. Raises `frappe.PermissionError` if missing.
2. Removes fields above the permlevels the user can read. `"*"` becomes the permitted fields.
3. Adds the `if_owner` condition, or the User Permission conditions.
4. Adds the conditions of `permission_query_conditions` hooks.
5. Adds shared documents with `OR`. With no role permission, only shared documents match.

```python
frappe.qb.get_query("ToDo", fields=["name"], ignore_permissions=False, user="user1@example.com").get_sql()
# SELECT `name` FROM `tabToDo` WHERE ((`tabToDo`.allocated_to = 'user1@example.com'
#   or `tabToDo`.assigned_by = 'user1@example.com' or `tabToDo`.owner = 'user1@example.com'))
```

See [Permissions](../../developer-guide/security/permissions.md).

## Deprecated

| Name | Use instead |
|---|---|
| `get_query(..., validate_filters=...)` | Leave it out. It does nothing. |
| `frappe.query_builder.functions.Ifnull` | `IfNull` |

## Pitfalls

- `get_query` does not check that a field exists. A wrong field name fails when the query runs.
- `get_query` applies no `ORDER BY` by default. `frappe.get_list` sorts by the DocType settings.
- `run(run=False)` returns the SQL without its values. Use `walk()` to get both.
- `str(query)` and `get_sql()` put values in the SQL. Do not run that string with `frappe.db.sql`. Pass the query object.
- `query.run()` always uses `frappe.local.db`, even for a query from `frappe.db.qb` of a replica. Use `db.sql(query)` for another connection.

## Related

- [Query Builder](../../developer-guide/database/query-builder.md)
- [frappe.db](database.md)
- [Database](../../developer-guide/database/database.md)
- [Permissions](../../developer-guide/security/permissions.md)
- [Hooks](../configuration/hooks.md)
