---
title: Permissions
description: How the permission engine decides access from role permissions, permission levels, user permissions, sharing and app hooks, and how to check permissions in code.
version: v15
sources:
  - frappe/permissions.py
  - frappe/__init__.py
  - frappe/model/document.py
  - frappe/model/db_query.py
  - frappe/share.py
  - frappe/handler.py
  - frappe/api/v1.py
  - frappe/api/v2.py
---

# Permissions

The permission engine decides if a user can do an action on a DocType or on one document. Use this page when you write server code that reads or changes data for a user, or when your app must add its own access rules. To set up roles and rules in the desk, see [Role permissions](../../user-guide/users-and-permissions/permissions.md) and [User Permissions](../../user-guide/users-and-permissions/user-permissions.md).

## How a decision is made

`frappe.has_permission` (in `frappe/__init__.py`) calls `frappe.permissions.has_permission`. The engine evaluates these layers in order:

1. `Administrator` gets every permission. No other check runs.
2. If `ptype` is `share` and **Disable Document Sharing** is on in System Settings, access is denied.
3. If the DocType is a child table, the check runs on the parent DocType. Pass `parent_doctype` when you do not pass a document.
4. Role permissions. The engine collects the `DocPerm` rows (or `Custom DocPerm` rows, if the DocType has any) at permission level 0 for the roles of the user. A right is granted if any row grants it.
5. If you pass a document, the engine also applies:
    1. `has_permission` hooks. They can only deny. If a hook denies, no other layer runs.
    2. "If Owner" rules. Some rights apply only when `doc.owner` is the user.
    3. User Permissions on the document itself and on its Link fields.
6. If access is still denied, the engine checks document sharing (`DocShare`). Only the rights `read`, `write`, `share`, `submit`, `email` and `print` can be shared. `email` and `print` are checked as `read` shares.
7. `select` is implied by `read`.

Without a document, the result answers "can the user do this on at least one document of this DocType". With a document, it answers for that document only.

### Rights

The rights are in the tuple `frappe.permissions.rights`:

`select`, `read`, `write`, `create`, `delete`, `submit`, `cancel`, `amend`, `print`, `email`, `report`, `import`, `export`, `share`.

v15 has no custom permission types. Only these rights exist.

`submit` is always false on a DocType that is not submittable. `import` is always false when `allow_import` is off.

### Automatic roles

`frappe.get_roles(user)` adds roles based on the user type. You do not assign them:

| Role | Who has it |
|---|---|
| `Guest` | Everybody, including users who are not logged in |
| `All` | Every logged-in user, including website users |
| `Desk User` | System users (desk users) |
| `Administrator` | The `Administrator` user |

```python
frappe.get_roles("user1@example.com")
# ['Sample Role', 'All', 'Guest', 'Desk User']
```

For `Administrator`, `frappe.get_roles` returns every `Role` record on the site.

## Check permissions in code

### frappe.has_permission

```python
frappe.has_permission(
    doctype=None, ptype="read", doc=None, user=None, throw=False,
    *, parent_doctype=None, debug=False, ignore_share_permissions=False,
)
```

| Name | Type | Default | Description |
|---|---|---|---|
| `doctype` | `str` | `None` | DocType to check. Optional if you pass `doc`. |
| `ptype` | `str` | `"read"` | Right to check. |
| `doc` | `Document` or `str` | `None` | Document or document name. Enables document-level checks. |
| `user` | `str` | current user | User to check for. |
| `throw` | `bool` | `False` | Raise `frappe.PermissionError` instead of returning `False`. |
| `parent_doctype` | `str` | `None` | Required for a child DocType when you do not pass `doc`. |
| `debug` | `bool` | `False` | Collect the reason for each step in `frappe.local.permission_debug_log`. |
| `ignore_share_permissions` | `bool` | `False` | Do not grant access through sharing. |

Output on the site, as `user1@example.com` (role `Sample Role`, which has read, write, create and delete on `Sample Item` and no rule on `Sample Order`):

```python
frappe.set_user("user1@example.com")
frappe.has_permission("Sample Item", "read")                  # True
frappe.has_permission("Sample Item", "write", doc="Laptop")   # True
frappe.has_permission("Sample Order", "read")                 # False
frappe.has_permission("Sample Order", "read", throw=True)     # raises frappe.PermissionError
```

When `throw=True` and access is denied for the current user, the engine also shows the reason with `frappe.msgprint`.

### Document methods

| Method | Description |
|---|---|
| `doc.has_permission(permtype="read", *, debug=False, user=None)` | Returns `bool`. Returns `True` if `doc.flags.ignore_permissions` is set. |
| `doc.check_permission(permtype="read", permlevel=None)` | Raises `frappe.PermissionError` if not permitted. |

`frappe.get_doc` does not check permissions. Load the document, then call `doc.check_permission("read")`.

`doc.insert()`, `doc.save()`, `doc.submit()`, `doc.cancel()` and `frappe.delete_doc()` check the matching right. Pass `ignore_permissions=True`, or set `doc.flags.ignore_permissions = True`, to skip the check. See [Controllers](../data-model/controllers.md).

### frappe.only_for

`frappe.only_for(roles, message=False)` raises `frappe.PermissionError` if the current user has none of the roles. `Administrator` always passes. The check is also skipped when `frappe.flags.in_test` is set. Set `message=True` to show a "This action is only allowed for ..." message.

```python
frappe.only_for("System Manager")
frappe.only_for(["System Manager", "Sample Role"], message=True)
```

## Permission levels

Each field has a `permlevel` (default 0). A role can read or write fields at level N only if it has a `DocPerm` row at level N. Level 0 rights decide access to the document itself. Rights at higher levels only control groups of fields.

- The REST API (`GET /api/resource/<doctype>/<name>` and `GET /api/v2/document/<doctype>/<name>`) removes values of fields the user cannot read (`doc.apply_fieldlevel_read_permissions()`). `frappe.get_doc` does not.
- `frappe.get_list` leaves out fields the user cannot read, also when you ask for `*`.
- `frappe.get_list` raises `frappe.PermissionError` if you sort or group by a field the user cannot read.
- `doc.save()` resets fields at a level the user has no access to back to their stored values (`Document.validate_higher_perm_levels`). `Administrator` and `ignore_permissions` skip this.

## Permissions in list queries

`frappe.get_list` adds conditions to the SQL query. The logic is in `DatabaseQuery.build_match_conditions` in `frappe/model/db_query.py`. `frappe.qb.get_query` does not add permission conditions in v15.

1. The user needs `select` or `read` on the DocType, or the query raises `frappe.PermissionError`.
2. If no role gives read or select, the query returns only documents shared with the user.
3. If read is "If Owner" only, the query adds `owner = <user>`.
4. Otherwise the query adds User Permission conditions for the DocType and its Link fields.
5. The query adds the conditions from `permission_query_conditions` hooks and from Server Scripts of type Permission Query.
6. If any condition from steps 3 to 5 applies, shared documents are added with `OR`.

`frappe.get_all` sets `ignore_permissions=True`. It skips all of the above.

```python
frappe.set_user("user1@example.com")
frappe.get_list("Sample Order")    # raises frappe.PermissionError
frappe.get_all("Sample Order", pluck="name")
# ['SO-00006', 'SO-00005', 'SO-00004', 'SO-00003', 'SO-00002', 'SO-00001']
```

Steps 3 to 5 run for `Administrator` too. Only `ignore_permissions` skips them.

After `frappe.share.add("Sample Order", "SO-00002", "user1@example.com", read=1)`:

```python
frappe.get_list("Sample Order", pluck="name")                         # ['SO-00002']
frappe.has_permission("Sample Order", "read", doc="SO-00002")         # True
frappe.has_permission("Sample Order", "write", doc="SO-00002")        # False
```

### User Permissions in queries

A User Permission restricts a user to some documents of a DocType. It applies to that DocType and to every Link field that points to it, unless the field has `ignore_user_permissions` set.

```python
from frappe.permissions import add_user_permission

add_user_permission("Sample Item", "Laptop", "user1@example.com")

frappe.set_user("user1@example.com")
frappe.get_list("Sample Item", pluck="name")                                 # ['Laptop']
frappe.has_permission("Sample Item", "read", doc="Monitor")                  # False
frappe.has_permission("Sample Item", "read")                                 # True, no doc
```

`frappe.get_list` has no `ignore_user_permissions` argument in v15. Passing it raises `TypeError`.

Helpers in `frappe/permissions.py`:

| Function | Description |
|---|---|
| `add_user_permission(doctype, name, user, ignore_permissions=False, applicable_for=None, is_default=0, hide_descendants=0)` | Creates a `User Permission`. Raises `frappe.DoesNotExistError` if the document does not exist. |
| `remove_user_permission(doctype, name, user, ignore_permissions=False)` | Deletes it. |
| `clear_user_permissions_for_doctype(doctype, user=None)` | Deletes all User Permissions for a DocType. |
| `get_user_permissions(user)` | Returns the User Permissions of a user, grouped by DocType. |

## Add rules from your app

### has_permission hook

Use the `has_permission` hook to deny access to single documents with Python logic. The hook runs only for document-level checks.

```python
# sample_app/hooks.py
has_permission = {
    "Sample Order": "sample_app.permissions.sample_order_has_permission",
}
```

```python
# sample_app/permissions.py
def sample_order_has_permission(doc, ptype=None, user=None, debug=False):
    if ptype == "write" and doc.status == "Completed":
        return False
    return None
```

Rules:

- The hook can only deny. It cannot grant a right that role permissions do not give.
- Return `False` to deny. `None` means "no decision", so Frappe asks the next handler. A function that ends without `return` does not deny.
- Handlers for `"*"` run for every DocType. Handlers run in reverse order. The first handler that returns a value other than `None` decides, and later handlers do not run.
- The hook is called with `frappe.call`, so it receives only the arguments it declares from `doc`, `ptype`, `user` and `debug`.

!!! warning
    The hook does not run for list queries. Add a matching `permission_query_conditions` hook, or `frappe.get_list` will show documents that `has_permission` denies.

### permission_query_conditions hook

Use `permission_query_conditions` to filter list queries. The function returns a SQL condition string. An empty value adds no condition.

```python
# sample_app/hooks.py
permission_query_conditions = {
    "Sample Order": "sample_app.permissions.sample_order_query",
}
```

```python
# sample_app/permissions.py
import frappe


def sample_order_query(user, doctype=None):
    if user == "Administrator" or "System Manager" in frappe.get_roles(user):
        return ""
    return f"`tabSample Order`.`owner_user` = {frappe.db.escape(user)}"
```

Rules:

- The function receives `user` and the keyword argument `doctype`.
- The function must return a SQL string. The engine joins all conditions with `AND` and wraps the result in parentheses.
- The hook also runs for `Administrator`. Return an empty string for users that must see everything.
- Escape every value with `frappe.db.escape`. The string goes into the SQL as it is.

## Ignore permissions safely

| Code | Effect |
|---|---|
| `frappe.get_all(...)` | No permission conditions. |
| `frappe.get_list(..., ignore_permissions=True)` | Same as `frappe.get_all`. |
| `doc.insert(ignore_permissions=True)`, `doc.save(ignore_permissions=True)` | Skips the right check for this call. |
| `doc.flags.ignore_permissions = True` | Skips the right check for all later actions on this object. |
| `frappe.db.*` methods | Never check permissions. |

Arguments that come from an HTTP request never reach your function as `ignore_permissions` or `flags`. `frappe.get_newargs` removes them before the call. Make sure you do not read them from `frappe.form_dict`.

## Pitfalls

- `frappe.get_all` and `frappe.db.*` return data the user may not see. In a whitelisted method, use `frappe.get_list`, or check `frappe.has_permission(..., throw=True)` first.
- A `has_permission` hook that returns `None` does not deny access. Return `False` explicitly.
- A `permission_query_conditions` hook without an `Administrator` check also filters the list for `Administrator`.
- `has_permission` without `doc` returns `True` if the user can access at least one document. It does not apply User Permissions or hooks.
- Sharing overrides User Permissions and query hooks in list queries.

## Related

- [Role permissions (desk)](../../user-guide/users-and-permissions/permissions.md)
- [User Permissions (desk)](../../user-guide/users-and-permissions/user-permissions.md)
- [Controllers](../data-model/controllers.md)
- [Whitelisted methods](../api/whitelisted-methods.md)
- [Database API](../database/database.md)
- [Security](security.md)
- [Hooks reference](../../reference/configuration/hooks.md)
