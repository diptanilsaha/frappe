---
title: Security
description: Write secure Frappe app code - permission checks in whitelisted methods, safe SQL, escaping, private files, rate limits, CSRF, secrets and the risks of ignore_permissions.
version: develop
sources:
  - frappe/__init__.py
  - frappe/model/document.py
  - frappe/rate_limiter.py
  - frappe/auth.py
  - frappe/utils/response.py
  - frappe/core/doctype/file/file.py
  - frappe/utils/data.py
  - frappe/utils/html_utils.py
  - frappe/utils/password.py
  - frappe/utils/safe_exec.py
  - AGENTS.md
  - code_review.md
---

# Security

Every whitelisted method is an HTTP endpoint. Any logged-in user, and with `allow_guest` any visitor, can call it with any arguments. This page lists the rules that keep app code safe. The rules come from `code_review.md` and `AGENTS.md` in the frappe repository, and each one is checked against the code.

## Check permissions in whitelisted methods

Frappe checks that the method is whitelisted. It does not check permissions on the documents that your method reads or changes. Do it yourself, before you read data.

```python
# sample_app/api.py
import frappe


@frappe.whitelist(methods=["GET"])
def get_order(name: str):
	doc = frappe.read_doc("Sample Order", name)
	return doc.as_dict()


@frappe.whitelist(methods=["POST"])
def close_order(name: str):
	doc = frappe.get_doc("Sample Order", name)
	doc.check_permission("write")
	doc.status = "Closed"
	doc.save()
```

| API | Checks |
|---|---|
| `frappe.read_doc(doctype, name)` | Raises `PermissionError` without read permission. Removes fields the user cannot read. The returned document cannot be saved. |
| `frappe.get_doc(doctype, name)` | No check on load. `doc.save()`, `doc.insert()`, `doc.submit()` and `doc.delete()` check permission, unless `ignore_permissions` is set. |
| `doc.check_permission(ptype)` | Raises `PermissionError`. |
| `frappe.has_permission(doctype, ptype, doc=None, user=None, throw=False)` | Returns `True` or `False`. Raises with `throw=True`. |
| `frappe.only_for(roles)` | Raises `PermissionError` if the user has none of the roles. |
| `frappe.get_list(...)` | Applies role permissions, user permissions and permission query conditions. |
| `frappe.get_all(...)` | No permission check. Use it only for internal queries. |
| `frappe.db.get_value(...)`, `frappe.db.sql(...)` | No permission check. |

- Take the user from `frappe.session.user`. Never take the user ID from an argument.
- Restrict `methods` for each endpoint. Use `GET` for reads and `POST` for writes. A `GET` endpoint that writes can be triggered by a link.
- Put validation in the controller (`validate`, `on_update`, `on_trash`), not in one endpoint. Bulk actions, Data Import, the REST API and `doc.save()` skip a check that lives in one endpoint.

See [Permissions](permissions.md) for the permission engine.

## ignore_permissions

`doc.insert(ignore_permissions=True)`, `doc.flags.ignore_permissions = True` and `frappe.get_all` skip permission checks. Use them only for writes that the system makes, such as a log record, never for a value that comes from the request. Do not use `ignore_permissions=True` or a blanket role grant to fix a workflow. Give the user the right permission instead.

## SQL injection

Use the ORM or the query builder. They escape values.

```python
# Good
frappe.get_list("Sample Order", filters={"customer_name": customer}, fields=["name"])

Order = frappe.qb.DocType("Sample Order")
frappe.qb.from_(Order).select(Order.name).where(Order.customer_name == customer).run()

# Good: values as parameters
frappe.db.sql("select name from `tabSample Order` where customer_name = %s", (customer,))

# Bad: value in the string
frappe.db.sql(f"select name from `tabSample Order` where customer_name = '{customer}'")
```

- Pass values in the second argument of `frappe.db.sql`, never with f-strings or `%` formatting.
- A field name or sort order from the request is not a value. Check it against `frappe.get_meta(doctype).get_valid_columns()` or a fixed list.
- `frappe.get_list` rejects SQL functions as strings in `fields`. Use the dict form, for example `{"COUNT": "*"}`.
- In Server Scripts, `frappe.db.sql` allows only read queries.

## XSS

Escape at render time, for the context of the output. Do not strip HTML from data at storage, because that loses data.

- In Jinja templates, values are not escaped by default. Use `{{ value | e }}` for user data in HTML.
- In Python, use `frappe.utils.escape_html(text)` for text in HTML, and `frappe.utils.sanitize_html(html)` for HTML that users write.
- In desk JavaScript, use `frappe.utils.escape_html(value)` before you put a value into HTML.
- For a guest method, Frappe sanitizes the arguments in `frappe.form_dict` unless the method is whitelisted with `xss_safe=True`. Use `xss_safe=True` only when the method escapes its output.

## Files

- A file with `is_private` set is served from `/private/files/`. Frappe checks read permission on the File record, and logs the access. Guests always get `403`.
- The File permission allows Administrator, the owner, users the file is shared with, and users who can read the document that the file is attached to.
- A public file (`/files/...`) is readable by anyone who knows the URL. Make uploads private when they hold personal or business data.
- Frappe forces download, not display, for file types such as `.html`, `.svg` and `.xml`, so they cannot run scripts on the site domain.
- Do not open a file path that comes from the request. Look up the `File` document and check its permission.

See [Files](../data-model/files.md).

## Rate limits

Every guest endpoint needs a rate limit. Use the `rate_limit` decorator below `@frappe.whitelist`:

```python
# sample_app/api.py
import frappe
from frappe.rate_limiter import rate_limit


@frappe.whitelist(allow_guest=True, methods=["POST"])
@rate_limit(limit=5, seconds=60 * 60)
def contact(email: str, message: str):
	...
```

`rate_limit(key=None, limit=5, seconds=86400, methods="ALL", ip_based=True, endpoint=None, user_based=False)`:

| Name | Default | Description |
|---|---|---|
| `key` | `None` | A form field name. Its value becomes part of the counter, for example an email address. |
| `limit` | `5` | Requests allowed in the window. Can be a callable. |
| `seconds` | `86400` | Window length. |
| `methods` | `"ALL"` | HTTP methods to limit. |
| `ip_based` | `True` | Count per client IP. |
| `user_based` | `False` | Count per logged-in user. Guests are counted by IP. |
| `endpoint` | `None` | Counter name. Default is the dotted path of the function. |

Over the limit, Frappe raises `RateLimitExceededError` (HTTP 429). Counters are in Redis cache. Calls outside a request, such as tests and background jobs, are not limited.

For a guest method that must not be limited, write `@frappe.whitelist(allow_guest=True, rate_limit=None)`. This marks the choice. Any other value for `rate_limit` raises `TypeError`.

The site-wide limit in `site_config.json` (`"rate_limit": {"limit": <seconds>, "window": <seconds>}`) is different. It limits the total request time of the site in each window, not the number of requests.

## CSRF

For a logged-in session, Frappe checks the CSRF token on `POST`, `PUT`, `DELETE` and `PATCH` requests (`validate_csrf_token` in `frappe/auth.py`). The check runs only when the session holds a CSRF token. The desk sends the token in the `X-Frappe-CSRF-Token` header, or a form sends it as `csrf_token`. Requests whose `Referer` or `Origin` matches the `allowed_referrers` config also pass.

Do not set `ignore_csrf` in production. It turns the check off for all requests.

## Secrets

- Store passwords and API secrets in `Password` fields. Frappe encrypts them with the site's `encryption_key` and stores them in `__Auth`. Read them with `doc.get_password("fieldname")` or `frappe.utils.password.get_decrypted_password(doctype, name, fieldname)`.
- Keep a copy of `encryption_key` from `site_config.json`. Without it, a restored backup cannot decrypt the passwords.
- Never return a secret from an endpoint. Return a masked value or a boolean.
- Put bench-wide secrets in `site_config.json` or `common_site_config.json`, not in the app repository. Read them with `frappe.conf.<key>`.
- Do not log secrets or full request bodies with `frappe.log_error`.

## Other rules

- Fail loudly. Catch the specific exception. Do not wrap code in a blanket `try/except`.
- Never destroy data in silence. Disable records instead of deleting them, and confirm bulk actions.
- Server Scripts run restricted Python. Allow the `Script Manager` role only to trusted users. See [Server Scripts and Client Scripts](../low-code-and-ai/server-and-client-scripts.md).

## Related

- [Permissions](permissions.md)
- [Whitelisted Methods](../api/whitelisted-methods.md)
- [Authentication](authentication.md)
- [Files](../data-model/files.md)
- [Configuration](../operations/configuration.md)
