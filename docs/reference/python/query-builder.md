---
title: frappe.qb
description: Reference for frappe.qb and frappe.query_builder in version 15 - tables, fields, terms, functions, Case, Interval, run() options and every argument of frappe.qb.get_query.
version: v15
sources:
  - frappe/query_builder/__init__.py
  - frappe/query_builder/builder.py
  - frappe/query_builder/utils.py
  - frappe/query_builder/functions.py
  - frappe/query_builder/custom.py
  - frappe/query_builder/terms.py
  - frappe/database/query.py
  - frappe/database/operator_map.py
  - frappe/database/database.py
  - frappe/utils/safe_exec.py
---

# frappe.qb

`frappe.qb` builds SQL from Python objects. It is a [PyPika](https://github.com/kayak/pypika) 0.51 dialect with Frappe additions. Use this page to look up a class, function or argument. For a guided introduction with joins, subqueries and updates, see [Query Builder](../../developer-guide/database/query-builder.md).

All SQL on this page was made on a v15 site with MariaDB. Values are shown inline for reading. `run()` sends them as bound parameters.

!!! warning
    Queries built with `frappe.qb` skip permissions, validations, controller methods and `doc_events`. `UPDATE`, `INSERT` and `DELETE` do not set `modified` and do not clear the document cache. In v15, `frappe.qb.get_query` has no permission arguments. Nothing in `frappe.qb` checks permissions.

## The builder classes

`frappe.qb` is `frappe.local.qb`. `frappe.init()` sets it to the class for `db_type` in the site config:

| `db_type` | Class | PyPika base |
|---|---|---|
| `mariadb` | `frappe.query_builder.builder.MariaDB` | `MySQLQuery` |
| `postgres` | `frappe.query_builder.builder.Postgres` | `PostgreSQLQuery` |

`frappe.query_builder.get_query_builder(type_of_db: str)` returns the class for a `db_type`. `frappe.db` has no `qb` attribute in v15.

Class attributes and methods:

| Name | Description |
|---|---|
| `DocType(table_name: str, *args, **kwargs) -> Table` | Return the table `tab<table_name>`. Raises `frappe.ValidationError` ("Invalid DocType") if the name has characters other than letters, digits, `_`, space and `-`. |
| `Table(name, schema=None, alias=None)` | PyPika `Table` with the exact name, for tables such as `__Auth` or `tabSingles`. |
| `Tables(*names)` | PyPika: many tables at once. |
| `Schema(name)` | Return a schema. `frappe.qb.Schema("information_schema").tables` is a table in it. On Postgres, `information_schema.tables` maps to `pg_stat_all_tables`. |
| `Field(name, table=None, alias=None)` | Return a field. On Postgres, `table_name` maps to `relname` and `table_rows` to `n_tup_ins`. |
| `from_(table)` | Start a `SELECT`. A string is passed to `DocType()`. |
| `into(table)` | Start an `INSERT`. A string is passed to `DocType()`. |
| `update(table)` | Start an `UPDATE`. A string is passed to `DocType()`. |
| `select(*terms)` | Start a `SELECT` without a table. |
| `with_(table, name)` | Start a query with a `WITH` (CTE). |
| `get_query(...)` | Build a query from `fields` and `filters`. See [get_query](#get_query). |
| `max(dt, fieldname, filters=None, **kwargs)` | Return `MAX(fieldname)` for the filters, or `0`. `kwargs` go to `run()`. |
| `min(dt, fieldname, filters=None, **kwargs)` | Return `MIN(fieldname)`, or `0`. |
| `avg(dt, fieldname, filters=None, **kwargs)` | Return `AVG(fieldname)`, or `0`. |
| `sum(dt, fieldname, filters=None, **kwargs)` | Return `SUM(fieldname)`, or `0`. |
| `functions(name: str, *args, **kwargs) -> Function` | Return the SQL function `name` with the arguments. |
| `terms` | A `frappe._dict` of all public names in `pypika.terms`. |
| `asc`, `desc` | `Order.asc` and `Order.desc`. |
| `validate_doctype(doctype)` | The name check that `DocType()` runs. |
| `create_table`, `drop_table`, `create_index`, `drop_index`, `drop_database`, `drop_user`, `drop_view`, `load` | Inherited from PyPika `Query`. |

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
| `get_query(*args, **kwargs)` | Calls `frappe.database.query.Engine().get_query`. Same as `frappe.qb.get_query`. |
| `get_query_builder(type_of_db: str)` | Return the builder class for a `db_type`. |
| `Column` | Column helper from `frappe.query_builder.utils`. |
| `patch_query_execute()`, `patch_query_aggregation()` | Add `run()`, `walk()` and the aggregate helpers. `frappe.init()` calls them. |
| `ParameterizedValueWrapper`, `ParameterizedFunction` | Frappe classes that replace PyPika's `ValueWrapper` and `Function`, so values become bound parameters. |
| `Field` | PyPika `Field`. |

`Table` and `Interval` in this module are the PyPika classes. Other common PyPika names: `Case`, `Criterion`, `EmptyCriterion`, `Order`, `Not`, `NULL`, `Tuple`, `Array`, `JSON`, `JoinType`, `CustomFunction`, `Parameter`, `Index`, `Rollup`, `Schema`, `AliasedQuery`, `Query`, `DatePart`.

Importing `frappe.query_builder` patches PyPika:

- `table.field_name` and `table["field name"]` return a `Field` bound to the table. `table.field` is a `PseudoColumn("field")`.
- `query.run()` and `query.walk()` are added to every query and union.

There is no `LIKE` to `ILIKE` rewrite for Postgres, and no `REGEXP` rewrite. See [Criteria](#criteria).

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
| `Item.name.like("L%")`, `.not_like("L%")` | `LIKE`, `NOT LIKE` on both engines |
| `Item.name.ilike("l%")` | `ILIKE`. Postgres only. MariaDB has no `ILIKE`. |
| `Item.name.regex("^L")` | `REGEX`. Invalid SQL on MariaDB. See [Pitfalls](#pitfalls). |
| `Item.description.isnull()`, `.isnotnull()` | `IS NULL`, `IS NOT NULL` |
| `Item.rate[50:150]`, `Item.rate.between(50, 150)` | `BETWEEN 50 AND 150` |
| `Tuple(Item.category, Item.disabled).isin([("Hardware", 0)])` | `(category, disabled) IN (('Hardware', 0))` |
| `Criterion.all([a, b])`, `Criterion.any([a, b])` | `a AND b`, `a OR b`. An empty list gives an `EmptyCriterion` that adds nothing. |

Put parentheses around each side of `&` and `|`. Python binds `&` before `==`.

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
| `.ignore()` | `INSERT IGNORE` (MariaDB). `frappe.db.bulk_insert` uses it. |
| `.on_duplicate_key_update(field, value)` | `ON DUPLICATE KEY UPDATE` (MariaDB) |
| `.on_conflict(*fields).do_nothing()`, `.do_update(field, value=None)` | `ON CONFLICT` (Postgres) |
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

`query.run(*args, **kwargs)` calls `walk()`, then `frappe.db.sql(sql, params, *args, **kwargs)` on `frappe.local.db`. It takes the keyword arguments of [frappe.db.sql](database.md#sql):

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

After the query runs, `run()` also runs the child queries of `get_query` and adds their rows. This works only when the result rows are dicts with a `name` key, so use `run(as_dict=True)` and select `name`.

`query.walk()` returns `(sql, params)`. The SQL has `%(paramN)s` placeholders.

```python
frappe.qb.get_query("Sample Item", filters={"category": "Hardware"}).walk()
# ('SELECT `name` FROM `tabSample Item` WHERE `category`=%(param1)s', {'param1': 'Hardware'})
frappe.qb.from_(Item).select(Item.name).where(Item.name == "x").run(run=False)
# 'SELECT `name` FROM `tabSample Item` WHERE `name`=%(param1)s'
```

`frappe.db.sql(query_object)` also works, but it uses `str(query)`, so the values are written into the SQL. Frappe logs a warning to use `run()`.

In a Server Script, `run()` raises `frappe.PermissionError` ("Only SELECT SQL allowed in scripting") for a query that `check_safe_sql_query` rejects.

## Functions

Import functions from `frappe.query_builder.functions`. It exports all of `pypika.functions` and adds the names below. A name in the "Per database" rows renders differently for each engine.

### Frappe functions

| Function | MariaDB | Postgres |
|---|---|---|
| `GroupConcat(column, alias=None)` (MariaDB) / `GroupConcat(column, separator=",", alias=None)` (Postgres) | `GROUP_CONCAT(column)`. There is no separator argument. A second argument is the alias. | `STRING_AGG(column, separator)` |
| `Match(column).Against(text)` | `MATCH(...) AGAINST ('+text*' IN BOOLEAN MODE)` | `TO_TSVECTOR(...) @@ PLAINTO_TSQUERY(text)` |
| `DateFormat(date, format)` | `DATE_FORMAT` | `TO_CHAR` |
| `CombineDatetime(date, time)` | `TIMESTAMP(date, time)` | `date + time` (strings cast to `date` and `time`) |
| `UnixTimestamp(field)` | `unix_timestamp(field)` | `EXTRACT(EPOCH FROM field)` |
| `Cast_(value, as_type, alias=None)` | `CONCAT(value, '')` for `varchar`, else `CAST` | `CAST` |
| `Locate(*terms)` | `LOCATE(...)`. A first term that is not a string is converted with `get_sql()`. | same |
| `Ifnull(condition, term)` | `IFNULL`. Each argument that is not a string is turned into its SQL text and then quoted as a literal. Use PyPika `IfNull` for fields. | same |
| `Timestamp(term, time=None, alias=None)` | `TIMESTAMP(...)` | same |
| `Round(term, decimal=0)` | `ROUND` | same |
| `Truncate(term, decimal)` | `TRUNCATE` | same |
| `Concat_ws(*terms)` | `CONCAT_WS` | same |
| `SqlFunctions` | Enum of function names (`dayofyear`, `extract`, `locate`, `count`, `sum`, `avg`, `max`, `min`, `abs`, `timestamp`, `ifnull`). | |

`DateDiff`, `JSONExtract`, `JSONValue`, `JSONContains`, `Month`, `MonthName`, `Year`, `Quarter`, `YearWeek`, `Strpos` and `Instr` do not exist in v15. Use `CustomFunction` or `frappe.qb.functions`.

### PyPika functions

`Count`, `Sum`, `Avg`, `Max`, `Min`, `Std`, `StdDev`, `First`, `Last`, `ApproximatePercentile`, `IfNull`, `IsNull`, `NVL`, `NullIf`, `Coalesce`, `Concat`, `Upper`, `Lower`, `Length`, `Trim`, `Substring`, `Replace`, `Reverse`, `Insert`, `SplitPart`, `RegexpLike`, `RegexpMatches`, `Ascii`, `Bin`, `Floor`, `Sqrt`, `Abs`, `Cast`, `Convert`, `Signed`, `Unsigned`, `Date`, `DateAdd`, `DateDiff` (PyPika form with an interval argument), `TimeDiff`, `TimestampAdd`, `ToChar`, `ToDate`, `Extract`, `Now`, `CurDate`, `CurTime`, `CurTimestamp`, `UtcTimestamp`, `SqlTypes`.

### Custom functions and constants

| Name | Description |
|---|---|
| `CustomFunction(name, ["arg1", ...])` | Make a function class. From `frappe.query_builder`. |
| `frappe.qb.functions("UPPER", Item.name)` | Call any SQL function by name. |
| `frappe.query_builder.custom.ConstantColumn(value)` | A column with the same string in every row. |
| `frappe.query_builder.custom.GROUP_CONCAT`, `STRING_AGG`, `MATCH`, `TO_TSVECTOR` | The engine classes behind `GroupConcat` and `Match`. |
| `frappe.query_builder.terms.SubQuery(query, alias=None)` (also `subqry`) | Wrap a query as a criterion term, for `>` or `=` against a subquery. |
| `frappe.query_builder.terms.NamedParameterWrapper` | Collects the values for `walk()`. Internal. |

```python
from frappe.query_builder import CustomFunction
from frappe.query_builder.custom import ConstantColumn
from frappe.query_builder.functions import Count, GroupConcat

Item = frappe.qb.DocType("Sample Item")
DateOnly = CustomFunction("DATE", ["value"])

frappe.qb.from_(Item).select(
    Item.category,
    Count(Item.name).as_("count"),
    GroupConcat(Item.name).as_("names"),
).groupby(Item.category).get_sql()
# SELECT `category`,COUNT(`name`) `count`,GROUP_CONCAT(`name`) `names` FROM `tabSample Item` GROUP BY `category`

frappe.qb.from_(Item).select(DateOnly(Item.creation).as_("d"), ConstantColumn("Sample Item").as_("doctype")).get_sql()
# SELECT DATE(`creation`) `d`,'Sample Item' `doctype` FROM `tabSample Item`
```

## Case, Order and Interval

| Name | Use |
|---|---|
| `Case().when(criterion, value).else_(value)` | `CASE WHEN ... THEN ... ELSE ... END`. Chain more `.when()` calls. |
| `Order.asc`, `Order.desc` | The `order` argument of `.orderby()`. Also `frappe.qb.asc` and `frappe.qb.desc`. |
| `Interval(years=0, months=0, days=0, hours=0, minutes=0, seconds=0, microseconds=0, quarters=0, weeks=0, dialect=None)` | PyPika date interval. Add it to or subtract it from a date term. |

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
    fields: list | tuple | None = None,
    filters: dict | str | int | list | None = None,
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
) -> QueryBuilder
```

Build a query from a `fields` and `filters` syntax close to `frappe.get_all`. `frappe.db.get_value`, `get_values`, `exists`, `count`, `set_value` and `delete` use it. `frappe.database.query.Engine().get_query(...)` is the class behind it.

`ignore_permissions`, `user`, `or_filters`, `parent_doctype` and the other permission arguments do not exist in v15. Passing them raises `TypeError`.

| Argument | Default | Description |
|---|---|---|
| `table` | | DocType name or a `Table`. |
| `fields` | `["name"]` | Fields to select. See [Fields](#fields). |
| `filters` | `None` | Conditions joined with `AND`. See [Filters](#filters). |
| `order_by` | `None` | No `ORDER BY` when empty. `"KEEP_DEFAULT_ORDERING"` also gives no `ORDER BY`. See [Order and group](#order-and-group). |
| `group_by` | `None` | Comma-separated fields. |
| `limit` | `None` | Row limit. A string such as `"5"` also works. |
| `offset` | `None` | Rows to skip. Without `limit`, it renders `OFFSET n` alone, which MariaDB rejects. Pass `limit` too. |
| `distinct` | `False` | `SELECT DISTINCT`. |
| `for_update` | `False` | Add `FOR UPDATE`. |
| `skip_locked` | `False` | With `for_update`, add `SKIP LOCKED`. |
| `wait` | `True` | With `for_update`, `False` adds `NOWAIT`. |
| `update` | `False` | Return an `UPDATE` with the filters. Chain `.set()`. |
| `into` | `False` | Return an `INSERT INTO`. Chain `.columns()` and `.insert()`. |
| `delete` | `False` | Return a `DELETE` with the filters. |
| `validate_filters` | `False` | Reject filter field names with special characters (`frappe.ValidationError`, "Invalid filter"). Without it, such a name is quoted as it is. |

```python
frappe.qb.get_query("Sample Item", filters={"name": "Laptop"}, update=True).set("rate", 1).get_sql()
# UPDATE `tabSample Item` SET `rate`=1 WHERE `name`='Laptop'
frappe.qb.get_query("Sample Item", filters={"name": "Laptop"}, delete=True).get_sql()
# DELETE FROM `tabSample Item` WHERE `name`='Laptop'
```

### Fields

| Form | Example | SQL |
|---|---|---|
| Field name | `"rate"` | `` `rate` `` |
| Comma-separated string | `"name, rate as price"` | `` `name`,`rate` `price` `` |
| Alias | `"rate as price"` | `` `rate` `price` `` |
| All columns | `"*"` | `*` |
| Table-qualified | ``"`tabSample Item`.`rate`"`` | `` `tabSample Item`.`rate` `` |
| Link field | `"item.category"` (in `Sample Order Item`) | `LEFT JOIN` on `tabSample Item` |
| Child table field | `"items.item"` (in `Sample Order`) | `LEFT JOIN` on the child table with `parent` and `parenttype` |
| Child query | `{"items": ["item", "qty"]}` | A second query per run. Rows go in a list under `items`. Needs `run(as_dict=True)`. |
| SQL function as a string | `"count(name) as n"` | ``COUNT(`name`) `n` `` |
| Query builder term | `Count(Item.name)`, `Item.rate` | as built |

The dict function form of later versions, such as `{"COUNT": "*"}`, does not work in v15. A dict is read as a child query and raises `AttributeError`.

```python
frappe.qb.get_query("Sample Order", fields=["name", {"items": ["item", "qty"]}], filters={"name": "SO-00001"}).run(as_dict=True)
# [{'name': 'SO-00001', 'items': [{'item': 'Laptop', 'qty': 2.0}, {'item': 'Monitor', 'qty': 2.0}]}]
```

### Filters

| Form | Example | SQL |
|---|---|---|
| Name | `"Laptop"` | `name='Laptop'` |
| List of names | `["Laptop", "Monitor"]` | `name IN ('Laptop','Monitor')` |
| Dict | `{"category": "Hardware", "rate": [">", 100]}` | `category='Hardware' AND rate>100` |
| List of conditions | `[["rate", "between", [10, 100]], ["item_name", "like", "%a%"]]` | joined with `AND` |
| Condition with DocType | `[["Sample Order Item", "item", "=", "Laptop"]]` | joins the child table on `parent` and `parenttype` |
| Criterion | `Item.rate > 800` | as built |

Nested `"or"` and `"and"` groups are not supported in v15. A string `"or"` in a filter list is read as a name filter, `name='or'`.

Operators:

| Operator | Value | Notes |
|---|---|---|
| `=`, `!=`, `<`, `>`, `<=`, `>=`, `=<`, `=>` | value | `None` gives `IS NULL` with any operator, also with `!=`. |
| `in`, `not in` | list, tuple, or comma-separated string | An empty list gives `IN ('')` or `NOT IN ('')`. |
| `like`, `not like` | pattern | |
| `regex` | pattern | Renders `REGEX`, which is invalid SQL on MariaDB. |
| `between` | `[from, to]` | Values go through as given. Works on number fields. |
| `is` | `"set"` or `"not set"` | `"set"` gives `<>''`. `"not set"` gives `IS NULL OR =''`. |
| `timespan` | `"this year"`, `"last week"` and others | Becomes `BETWEEN` dates. |
| `ancestors of`, `descendants of`, `not ancestors of`, `not descendants of`, `descendants of (inclusive)` | document name | For tree DocTypes. Uses `lft` and `rgt`. |
| `+`, `-`, `*`, `/` | value | Python arithmetic operators. |

`ilike`, `previous`, `next` and operators from the `filters_config` hook are not supported in `get_query`. `previous` raises `KeyError`. `_assign` and `_liked_by` get no special handling: `=` stays `=`.

```python
frappe.qb.get_query("Sample Item", filters={"creation": ["timespan", "this year"]}).get_sql()
# SELECT `name` FROM `tabSample Item` WHERE `creation` BETWEEN '2026-01-01' AND '2026-12-31'
frappe.qb.get_query("Sample Item", filters={"name": ["in", "Laptop,Monitor"]}).get_sql()
# SELECT `name` FROM `tabSample Item` WHERE `name` IN ('Laptop','Monitor')
```

### Order and group

`order_by` and `group_by` take comma-separated fields, with an optional `asc` or `desc`.

!!! warning
    In `order_by`, a field without a direction sorts `DESC`. `frappe.get_list` sorts it `ASC`. Always write the direction.

```python
frappe.qb.get_query("Sample Item", fields=["name"], order_by="rate").get_sql()
# SELECT `name` FROM `tabSample Item` ORDER BY `rate` DESC
```

## Deprecated

| Name | Use instead |
|---|---|
| `frappe.query_builder.functions.Ifnull` | PyPika `IfNull`, which keeps fields as fields. |

## Pitfalls

- `get_query` does not check permissions in v15. Use `frappe.get_list` when permissions matter.
- `get_query` applies no `ORDER BY` by default, also with `"KEEP_DEFAULT_ORDERING"`. `frappe.db.get_value` adds `modified desc` itself.
- `.regex()` and the `regex` filter render `REGEX`, which MariaDB rejects with a syntax error. `RegexpLike` renders `REGEXP_LIKE`, which MariaDB also lacks. Use raw SQL with `REGEXP`.
- `GroupConcat(field, "; ")` on MariaDB uses `"; "` as the alias, not as the separator.
- `Ifnull(Item.description, "''")` renders `IFNULL('description', ...)`, a string literal, not the column.
- `run(run=False)` returns the SQL without its values. Use `walk()` to get both.
- `str(query)` and `get_sql()` put values in the SQL. Do not run that string with `frappe.db.sql`. Call `run()`.

## Related

- [Query Builder](../../developer-guide/database/query-builder.md)
- [frappe.db](database.md)
- [Database](../../developer-guide/database/database.md)
- [Permissions](../../developer-guide/security/permissions.md)
- [Hooks](../configuration/hooks.md)
