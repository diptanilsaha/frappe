---
title: Query Builder
description: "Build SQL queries in Python with frappe.qb (PyPika): DocType tables, conditions, functions, joins, subqueries and frappe.qb.get_query."
version: v16
sources:
  - frappe/query_builder/__init__.py
  - frappe/query_builder/utils.py
  - frappe/query_builder/functions.py
  - frappe/query_builder/custom.py
  - frappe/query_builder/builder.py
  - frappe/query_builder/terms.py
  - frappe/query_builder/docs.md
  - frappe/database/query.py
  - frappe/database/database.py
---

# Query Builder

`frappe.qb` builds SQL queries from Python objects. It is a dialect of [PyPika](https://github.com/kayak/pypika) for MariaDB, Postgres and SQLite. Use it when `frappe.get_all` cannot express the query, for example for joins, subqueries, `CASE` or complex aggregates. Values are always sent as query parameters, so the query builder protects you from SQL injection.

All examples on this page were run in `bench --site frappe.localhost console`.

## Tables and fields

`frappe.qb.DocType(name)` returns the table of a DocType (`tab<name>`). Get a field as an attribute or with brackets.

```python
from frappe.query_builder import Order

Item = frappe.qb.DocType("Sample Item")

query = (
    frappe.qb.from_(Item)
    .select(Item.name, Item.rate)
    .where((Item.category == "Hardware") & (Item.rate > 100))
    .orderby(Item.rate, order=Order.desc)
)
query.get_sql()
# SELECT `name`,`rate` FROM `tabSample Item` WHERE `category`='Hardware' AND `rate`>100 ORDER BY `rate` DESC
query.run(as_dict=True)
# [{'name': 'Laptop', 'rate': 900.0}, {'name': 'Monitor', 'rate': 250.0}]
```

`frappe.qb.Table("Singles")` returns a table that is not a DocType. `Item["field name"]` works for field names that are not valid Python names.

## Run a query

`query.run()` runs the query on `frappe.db`. It takes the options of `frappe.db.sql`:

| Option | Result |
|---|---|
| none | Tuple of tuples |
| `as_dict=True` | List of dicts |
| `pluck=True` | Flat list of the first column |
| `debug=True` | Prints the query |

```python
frappe.qb.from_(Item).select(Item.name).where(Item.category.isin(["Software", "Service"])).run(pluck=True)
# ['Antivirus', 'Installation', 'Office Suite', 'Support Plan']
```

`query.walk()` returns the SQL with placeholders and the parameter dict:

```python
frappe.qb.from_(Item).select(Item.name).where(Item.name == "Laptop").walk()
# ('SELECT `name` FROM `tabSample Item` WHERE `name`=%(param1)s', {'param1': 'Laptop'})
```

!!! warning
    `query.get_sql()` and `str(query)` put the values into the SQL text. Use them to read the query, not to run it with `frappe.db.sql`.

## Conditions

| Python | SQL |
|---|---|
| `Item.rate == 10`, `!=`, `<`, `>`, `<=`, `>=` | comparison |
| `(a) & (b)`, `(a) \| (b)`, `~(a)` | `AND`, `OR`, `NOT` |
| `Item.category.isin([...])`, `.notin([...])` | `IN`, `NOT IN` |
| `Item.item_name.like("Lap%")`, `.not_like(...)` | `LIKE` (`ILIKE` on Postgres) |
| `Item.description.isnull()`, `.isnotnull()` | `IS NULL`, `IS NOT NULL` |
| `Item.rate[50:150]` | `BETWEEN 50 AND 150` |
| `Item.item_name.regex("^L")` | `REGEX`. MariaDB expects `REGEXP`, so this fails on MariaDB in v16. |

Put parentheses around each side of `&` and `|`. Python operator precedence binds `&` before `==`.

## Functions

Import functions from `frappe.query_builder.functions`. It exports all PyPika functions (`Count`, `Sum`, `Avg`, `Max`, `Min`, `Coalesce`, `IfNull`, ...) and adds database-aware ones such as `GroupConcat`, `Locate`, `DateFormat`, `Timestamp`, `UnixTimestamp`, `DateDiff`, `CombineDatetime`, `JSONExtract`, `Match`, `Round`, `Abs`.

```python
from frappe.query_builder.functions import Count, Sum

query = (
    frappe.qb.from_(Item)
    .select(Item.category, Count(Item.name).as_("count"), Sum(Item.rate).as_("total"))
    .groupby(Item.category)
    .orderby(Item.category)
)
query.run(as_dict=True)
# [{'category': 'Hardware', 'count': 3, 'total': 1195.0},
#  {'category': 'Service', 'count': 2, 'total': 380.0},
#  {'category': 'Software', 'count': 2, 'total': 180.0}]
```

`CASE` expressions:

```python
from frappe.query_builder import Case

band = Case().when(Item.rate >= 200, "High").else_("Low")
frappe.qb.from_(Item).select(Item.name, band.as_("band")).where(Item.category == "Hardware").orderby(Item.name).run()
# (('Keyboard', 'Low'), ('Laptop', 'High'), ('Monitor', 'High'))
```

Shortcut aggregates on `frappe.qb` take a DocType, a field and optional filters:

```python
frappe.qb.max("Sample Item", "rate")                                 # 900.0
frappe.qb.sum("Sample Item", "rate", filters={"category": "Hardware"})  # 1195.0
```

`frappe.qb.min` and `frappe.qb.avg` also exist.

## Joins

```python
SalesOrder = frappe.qb.DocType("Sample Order")
Row = frappe.qb.DocType("Sample Order Item")

query = (
    frappe.qb.from_(SalesOrder)
    .join(Row).on(Row.parent == SalesOrder.name)
    .select(SalesOrder.name, SalesOrder.customer_name, Row.item, Row.qty)
    .where(Row.item == "Laptop")
    .orderby(SalesOrder.name)
)
query.run(as_dict=True)
# [{'name': 'SO-00001', 'customer_name': 'Customer A', 'item': 'Laptop', 'qty': 2.0},
#  {'name': 'SO-00006', 'customer_name': 'Customer F', 'item': 'Laptop', 'qty': 1.0}]
```

`.join()` is an inner join. Use `.left_join()` for a left join. Child rows link to their parent with `parent`, `parenttype` and `parentfield`.

## Subqueries

A query object can be the value of `isin`:

```python
laptop_orders = frappe.qb.from_(Row).select(Row.parent).where(Row.item == "Laptop")

frappe.qb.from_(SalesOrder).select(SalesOrder.name).where(SalesOrder.name.isin(laptop_orders)).run(pluck=True)
# ['SO-00001', 'SO-00006']
```

## Updates, inserts and deletes

```python
frappe.qb.update(Item).set(Item.disabled, 1).where(Item.name == "Keyboard").run()
frappe.qb.into(Item).columns("name", "item_name").insert("Sample X", "Sample X").run()
frappe.qb.from_(Item).delete().where(Item.name == "Sample X").run()
```

These statements skip controllers, validations, `doc_events` and permission checks, like `frappe.db.set_value`. They also do not set `modified` or clear the document cache. Use them for bulk work in patches and jobs.

## frappe.qb.get_query

`frappe.qb.get_query` builds a query from the same `fields`, `filters` and `order_by` syntax as `frappe.get_all`. It returns a query object that you can extend with query builder methods.

```python
frappe.qb.get_query(
    table, fields=None, filters=None, order_by=None, group_by=None, limit=None, offset=None,
    distinct=False, for_update=False, update=False, into=False, delete=False,
    *, validate_filters=False, skip_locked=False, wait=True, ignore_permissions=True, ignore_user_permissions=False,
    user=None, parent_doctype=None, reference_doctype=None, or_filters=None, db_query_compat=False,
)
```

```python
query = frappe.qb.get_query(
    "Sample Item",
    fields=["name", "rate"],
    filters={"category": "Hardware"},
    order_by="rate desc",
    limit=2,
)
query.run(as_dict=True)
# [{'name': 'Laptop', 'rate': 900.0}, {'name': 'Monitor', 'rate': 250.0}]

frappe.qb.get_query("Sample Item", fields=["name"], filters={"category": "Hardware"}).where(Item.rate < 500).run(pluck=True)
# ['Keyboard', 'Monitor']
```

Differences from `frappe.get_list`:

- `ignore_permissions` defaults to `True`. Pass `ignore_permissions=False` to apply the permission rules of the current user (or of `user`). See [Permissions](../security/permissions.md).
- Without `order_by`, the query has no `ORDER BY`. `frappe.get_list` uses the sort settings of the DocType.
- The query is immutable after `get_query` returns. Each method such as `.where()` returns a new query.

```python
frappe.set_user("user1@example.com")
frappe.qb.get_query("Sample Order", fields=["name"], ignore_permissions=False).run()
# raises frappe.PermissionError
```

The REST endpoint `GET /api/v2/document/<doctype>` builds its query with `frappe.qb.get_query(..., ignore_permissions=False)`. A DocType controller can change that query with a static `get_list(query)` method. See [REST API](../api/rest-api.md).

## Use criteria in the database API

`frappe.db.get_value`, `frappe.db.get_values`, `frappe.db.count`, `frappe.get_all` and others accept a query builder criterion as `filters`:

```python
frappe.db.get_value("Sample Item", Item.rate > 800, "name")
# 'Laptop'
```

## Pitfalls

- Field names are not checked against the DocType. A wrong name fails only when the query runs.
- `frappe.qb` does not apply permissions. Check them before you return results to a user.
- In Server Scripts, only `SELECT` queries are allowed.
- `.regex()` builds `REGEX`, which MariaDB rejects. Use `.like()`, or `frappe.db.sql` with `REGEXP` and a parameter.

## Related

- [Database API](database.md)
- [Permissions](../security/permissions.md)
- [Security](../security/security.md)
- [Performance](../testing-and-performance/performance.md)
