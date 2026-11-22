---
title: Whitelisted Methods
description: Expose Python functions over HTTP with @frappe.whitelist, validate arguments with type annotations, restrict access by role, rate limit, and override methods of other apps.
version: v16
sources:
  - frappe/__init__.py
  - frappe/handler.py
  - frappe/utils/typing_validations.py
  - frappe/rate_limiter.py
  - frappe/api/v1.py
  - frappe/api/v2.py
  - frappe/hooks.py
  - frappe/exceptions.py
  - frappe/core/doctype/user/user.py
---

# Whitelisted Methods

A whitelisted method is a Python function that clients can call over HTTP. Add `@frappe.whitelist()` to a function in your app, and it becomes available at `/api/method/<dotted.path>` and `/api/v2/method/<dotted.path>`. The desk calls these methods with `frappe.call`. Functions without the decorator cannot be called from a client.

## Write a method

```python
# sample_app/api.py
import frappe


@frappe.whitelist()
def get_items(category: str, limit: int = 20) -> list[dict]:
    return frappe.get_list(
        "Sample Item",
        filters={"category": category, "disabled": 0},
        fields=["name", "rate"],
        order_by="rate desc",
        limit=limit,
    )
```

Call it:

```bash
curl -G "http://frappe.localhost:8006/api/method/sample_app.api.get_items" \
  -H "Authorization: token <api_key>:<api_secret>" \
  --data-urlencode "category=Hardware" --data-urlencode "limit=2"
```

```json
{"message": [{"name": "Laptop", "rate": 900.0}, {"name": "Monitor", "rate": 250.0}]}
```

How the call works (`frappe/handler.py`):

1. The path after `/api/method/` is the dotted path of the function.
2. Frappe applies `override_whitelisted_methods` hooks.
3. If a Server Script of type API has the same method name, the script runs instead.
4. Frappe imports the function and checks that it is whitelisted and that the HTTP method is allowed.
5. Frappe calls the function with the request parameters (`frappe.form_dict`). Parameters that the function does not declare are dropped, unless it has `**kwargs`.
6. The return value goes into `message` (v1) or `data` (v2) of the JSON response.

The parameters `ignore_permissions` and `flags` are always removed before the call (`frappe.get_newargs`).

## Decorator options

```python
frappe.whitelist(allow_guest=False, xss_safe=False, methods=None)
```

| Name | Type | Default | Description |
|---|---|---|---|
| `allow_guest` | `bool` | `False` | Allow calls from users who are not logged in. |
| `xss_safe` | `bool` | `False` | With `allow_guest`: do not sanitize HTML in string arguments of guest calls. |
| `methods` | `list` | all | Allowed HTTP methods. Default: `GET`, `POST`, `PUT`, `DELETE`. |

### methods

A call with another HTTP method raises `frappe.PermissionError` (HTTP 403):

```bash
curl "http://frappe.localhost:8006/api/method/frappe.client.set_value?doctype=Sample%20Item&name=Laptop&fieldname=rate&value=1" \
  -H "Authorization: token <api_key>:<api_secret>"
```

```json
{"exception": "frappe.exceptions.PermissionError: Not permitted", "exc_type": "PermissionError"}
```

`frappe.client.set_value` declares `methods=["POST", "PUT"]`.

!!! warning
    Frappe rolls back the transaction at the end of every `GET` request. A method that writes data must declare `methods=["POST"]` (or `PUT`, `DELETE`). See [Transactions](../database/database.md#transactions).

### allow_guest

```python
# sample_app/api.py
import frappe
from frappe.rate_limiter import rate_limit


@frappe.whitelist(allow_guest=True, methods=["POST"])
@rate_limit(limit=5, seconds=60 * 60)
def subscribe(email: str):
    frappe.get_doc({"doctype": "Sample Subscriber", "email": email}).insert(ignore_permissions=True)
```

For guest calls, Frappe sanitizes HTML in every string argument of `frappe.form_dict` with `frappe.utils.sanitize_html`, unless `xss_safe=True`.

Declare `methods` and add a `@rate_limit(...)` below `@frappe.whitelist` on every guest method.

## Type annotations

The decorator wraps the function with `validate_argument_types` (`frappe/utils/typing_validations.py`). During a request (and in tests), Frappe validates each annotated argument with pydantic:

- Values are converted when the conversion is safe. The string `"2"` becomes `2` for an `int` argument.
- A value that does not match raises `frappe.FrappeTypeError` (HTTP 417).
- The type of the default value is also accepted. `limit: int = None` accepts `None`.
- String annotations and forward references are not validated. This includes all annotations in a module with `from __future__ import annotations`.
- Direct Python calls outside a request are not validated.

```bash
curl -G "http://frappe.localhost:8006/api/method/frappe.desk.search.get_names_for_mentions" \
  -H "Authorization: token <api_key>:<api_secret>" \
  --data-urlencode "search_term=user" --data-urlencode "page_length=two"
```

```json
{"exception": "frappe.exceptions.FrappeTypeError: Argument 'page_length' in 'frappe.desk.search.get_names_for_mentions' should be of type 'int' but got 'str' instead.", "exc_type": "FrappeTypeError"}
```

Arguments without an annotation are not validated.

!!! note "Version"
    `frappe/hooks.py` sets `require_type_annotated_api_methods = True`, but no code in v16 reads this hook. Whitelisted methods without annotations still work.

## Check access

The decorator only decides who can call the function. It does not check data permissions. Check them in the function:

```python
# sample_app/api.py
import frappe


@frappe.whitelist(methods=["POST"])
def close_order(name: str):
    doc = frappe.get_doc("Sample Order", name)
    doc.check_permission("write")
    doc.status = "Completed"
    doc.save()
    return doc.status
```

| Tool | Use it to |
|---|---|
| `frappe.only_for(roles, message=False)` | Allow only users with one of the roles. Raises `frappe.PermissionError`. |
| `frappe.has_permission(doctype, ptype, doc, throw=True)` | Check a right on a DocType or document. |
| `doc.check_permission(ptype)` | Check a right on a loaded document. |
| `frappe.get_list(...)` | Query with the permissions of the user. |

`frappe.core.doctype.user.user.generate_keys` is an example. It starts with `frappe.only_for("System Manager")`. A call from another user returns HTTP 403. See [Permissions](../security/permissions.md).

## Rate limits

`frappe.rate_limiter.rate_limit` limits calls per endpoint. It counts calls in Redis.

```python
rate_limit(key=None, limit=5, seconds=86400, methods="ALL", ip_based=True, endpoint=None, user_based=False)
```

| Name | Type | Default | Description |
|---|---|---|---|
| `key` | `str` | `None` | Name of a request parameter. Its value is added to the counter key, for example `"email"`. |
| `limit` | `int` or callable | `5` | Maximum calls in the window. A callable is evaluated on each call. |
| `seconds` | `int` | `86400` | Window length. |
| `methods` | `str` or `list` | `"ALL"` | Limit only these HTTP methods. |
| `ip_based` | `bool` | `True` | Count per client IP. |
| `user_based` | `bool` | `False` | Count per logged-in user. Guests are counted per IP. |
| `endpoint` | `str` | `None` | Counter name. Required when the function has no stable dotted path, for example a `functools.partial`. |

When the limit is reached, the call raises `frappe.RateLimitExceededError` (HTTP 429). The limit applies only to HTTP requests. Direct Python calls are not counted.

!!! warning
    Put `@rate_limit` below `@frappe.whitelist`. If it is above, the registered function is not the one that the module exports, and the method is not whitelisted.

The site config key `rate_limit` (`{"limit": <seconds>, "window": <seconds>}`) is a separate limit. It limits the total request processing time per site in each window. Over the limit, requests get HTTP 429, and responses carry `X-RateLimit-*` headers. See [Configuration](../operations/configuration.md).

## Controller methods

Whitelist a method of a DocType controller to call it on one document:

```python
# sample_app/sample_app/doctype/sample_order/sample_order.py
import frappe
from frappe.model.document import Document


class SampleOrder(Document):
    @frappe.whitelist()
    def mark_completed(self):
        self.status = "Completed"
        self.save()
```

| Route | Permission checked |
|---|---|
| `POST /api/resource/Sample Order/SO-00001` with `run_method=mark_completed` | `write` |
| `GET /api/resource/Sample Order/SO-00001?run_method=mark_completed` | `read` |
| `POST /api/v2/document/Sample Order/SO-00001/method/mark_completed` | `write` |
| `GET /api/v2/document/Sample Order/SO-00001/method/mark_completed` | `read` |

The method also needs `@frappe.whitelist()`, and the HTTP method must be in its `methods`. In the desk, call it with `frm.call("mark_completed")`. See [Form scripts](../desk-ui/form-scripts.md).

A module-level function in a DocType module can be called by DocType name in v2: `/api/v2/method/<DocType>/<function>`. For example `/api/v2/method/User/get_current_user_roles` calls `frappe.core.doctype.user.user.get_current_user_roles`.

## Override a method of another app

```python
# sample_app/hooks.py
override_whitelisted_methods = {
    "frappe.desk.doctype.event.event.get_events": "sample_app.overrides.get_events",
}
```

- The key is the dotted path that clients call. The value is your function.
- The new function must also be whitelisted.
- If several apps override the same path, the app installed last wins.
- The override applies to HTTP calls only. Python code that imports the original function still calls it.

See [Extending apps](../building-apps/extending-apps.md).

## Pitfalls

- A whitelisted method is a public endpoint. Validate every argument and check permissions in the function.
- Do not use `frappe.get_all` or `ignore_permissions=True` to return data to the caller unless all callers may see all rows.
- Do not build SQL from arguments. Use the [Query builder](../database/query-builder.md) or parameters.
- Return values must be JSON serializable.

## Related

- [REST API](rest-api.md)
- [Authentication](../security/authentication.md)
- [Permissions](../security/permissions.md)
- [Client API](../desk-ui/client-api.md)
- [Server and client scripts](../low-code-and-ai/server-and-client-scripts.md)
- [Security](../security/security.md)
