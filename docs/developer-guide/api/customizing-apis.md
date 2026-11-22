---
title: Customizing APIs
description: Every way a v15 app can add or change API behavior in Frappe, from whitelisted methods and overrides to doc_events, permission hooks, request hooks, auth hooks, CORS, response formats, versioning and tests.
version: v15
sources:
  - frappe/app.py
  - frappe/api/__init__.py
  - frappe/api/v1.py
  - frappe/api/v2.py
  - frappe/handler.py
  - frappe/__init__.py
  - frappe/auth.py
  - frappe/permissions.py
  - frappe/model/db_query.py
  - frappe/model/document.py
  - frappe/client.py
  - frappe/rate_limiter.py
  - frappe/utils/response.py
  - frappe/website/path_resolver.py
  - frappe/hooks.py
  - frappe/tests/test_api.py
  - frappe/tests/utils.py
  - frappe/utils/__init__.py
---

# Customizing APIs

An app can add API endpoints and change how existing ones behave, without changes to Frappe or to other apps. This page lists each way, what it changes, and when it runs. For how a request is routed and answered, see [API Internals](api-internals.md). For the hook list, see [Hooks reference](../../reference/configuration/hooks.md).

The hook examples on this page were checked on the v15 site with the hooks added at runtime and a Werkzeug test client. The function names are examples.

## Choose a mechanism

| You want to | Use |
|---|---|
| Add an endpoint | `@frappe.whitelist()` function |
| Replace an endpoint of another app | `override_whitelisted_methods` |
| Change what REST create, read, update and delete do | Controller methods, `doc_events`, `override_doctype_class` |
| Hide documents or deny access | `has_permission`, `permission_query_conditions` |
| Run code on every request | `before_request`, `after_request` |
| Authenticate with your own header or token | `auth_hooks` |
| Limit calls | `@rate_limit`, site config `rate_limit` |
| Serve a route outside `/api` | `website_route_rules`, `page_renderer`, `www/` pages |
| Return a file, CSV or redirect | `frappe.response.type`, or return a `Response` |
| Allow browser calls from another origin | Site config `allow_cors` |

An app cannot add rules to the `/api` URL map (`API_URL_MAP` in `frappe/api/__init__.py`). New endpoints under `/api` are whitelisted methods.

## Add a whitelisted method

```python
# sample_app/api.py
import frappe


@frappe.whitelist(methods=["POST"])
def close_order(name: str) -> str:
	doc = frappe.get_doc("Sample Order", name)
	doc.check_permission("write")
	doc.status = "Completed"
	doc.save()
	return doc.status
```

The method is at `/api/method/sample_app.api.close_order` and `/api/v2/method/sample_app.api.close_order`. See [Whitelisted Methods](whitelisted-methods.md) for decorator options, type validation and permission checks.

## Override a method of another app

`override_whitelisted_methods` maps the dotted path that clients call to your function. Frappe applies it in `frappe.override_whitelisted_method()` before it imports the function, for v1 and v2.

```python
# sample_app/hooks.py
override_whitelisted_methods = {
	"frappe.auth.get_logged_user": "sample_app.api.get_logged_user",
}
```

```python
# sample_app/api.py
import frappe


@frappe.whitelist()
def get_logged_user() -> str:
	return "override:" + frappe.session.user
```

```json
{"message": "override:user1@example.com"}
```

- Your function must also be whitelisted. The whitelist and HTTP method checks run on your function, not on the original.
- If several apps override the same path, the last value in the hook list wins.
- Frappe looks for a Server Script of type API after it applies the override. A script with the original path does not run when the path is overridden. A script with the new path does.
- Python code that imports the original function still calls the original.
- Document routes (`/api/resource/...`, `/api/v2/document/...`) and fixed v2 routes such as `/api/v2/method/upload_file` do not use this hook.

## Change document behavior

REST create, update and delete call `doc.insert()`, `doc.save()` and `frappe.delete_doc()`. So every controller method and `doc_events` handler runs, the same as in the desk. Use these hooks to change what the REST routes do for a DocType of another app:

| Hook | Effect | See |
|---|---|---|
| `doc_events` | Run functions on document events (`validate`, `on_update`, `on_trash`, ...). An exception stops the request. | [Extending Other Apps](../building-apps/extending-apps.md#doc_events) |
| `override_doctype_class` | Replace the controller class. Last app wins. | [Extending Other Apps](../building-apps/extending-apps.md#override_doctype_class) |

v15 has no `extend_doctype_class` hook.

```python
# sample_app/hooks.py
doc_events = {
	"ToDo": {"validate": "sample_app.events.todo_validate"},
}
```

```python
# sample_app/events.py
import frappe


def todo_validate(doc, method):
	if doc.description == "forbidden":
		frappe.throw("Sample rule: forbidden description")
```

```bash
curl -s -X POST "$S/api/v2/document/ToDo" -H "$AUTH" -H "Content-Type: application/json" \
  -d '{"description": "forbidden"}' -w ' %{http_code}\n'
```

```json
{"errors": [{"type": "ValidationError", "message": "Sample rule: forbidden description", "title": "Message", "indicator": "red"}]} 417
```

A whitelisted method on the controller is callable with the document method routes. See [API Internals](api-internals.md#controller-methods).

Do not call `frappe.db.commit()` in these handlers. In v15 it commits at once, also when a later step of the request fails. The request commits when it succeeds.

## Change who can see and change documents

| Hook | Called with | Applies to |
|---|---|---|
| `has_permission` | `doc, ptype, user, debug` | One document: v1 and v2 read, create, update, delete and document methods. Frappe calls the hooks in reverse order. The first one that does not return `None` decides: a falsy value denies. `None` has no effect. A hook cannot grant. |
| `permission_query_conditions` | `user, doctype=` | Lists: v1 `/api/resource/<doctype>` and v2 `/api/v2/document/<doctype>`. Both call `frappe.client.get_list`. Returns an SQL condition string. |

```python
# sample_app/hooks.py
has_permission = {"ToDo": "sample_app.permissions.todo_has_permission"}
permission_query_conditions = {"ToDo": "sample_app.permissions.todo_query"}
```

```python
# sample_app/permissions.py
def todo_has_permission(doc, ptype, user, debug=False):
	if doc.description == "secret":
		return False
	return None


def todo_query(user, doctype=None):
	return "`tabToDo`.`description` != 'secret'"
```

Results for `user1@example.com`, who owns one ToDo "secret" and one ToDo "Sample visible":

| Request | Result |
|---|---|
| `POST /api/v2/document/ToDo` with `{"description": "secret"}` | HTTP 403. `has_permission` was called with `ptype="create"`. |
| `GET /api/v2/document/ToDo/<secret name>` | HTTP 403. Called with `ptype="read"`. |
| `GET /api/resource/ToDo/<secret name>` | HTTP 403 |
| `GET /api/v2/document/ToDo?fields=["description"]` | `{"data": [{"description": "Sample visible"}]}` |
| `GET /api/resource/ToDo?fields=["description"]` | `{"data": [{"description": "Sample visible"}]}` |

`has_permission` is not called for list rows. Use both hooks so that a list and a single read give the same answer. Frappe has no `write_permission_query_conditions` hook. See [Permissions](../security/permissions.md#add-rules-from-your-app).

## Run code on every request

| Hook | When | Called with |
|---|---|---|
| `before_request` | In `init_request()`, after `frappe.form_dict` and the cookie session are loaded, before `validate_auth()` | no arguments |
| `after_request` | In `application()`, after the commit or rollback, also for errors | `response=`, `request=` |

```python
# sample_app/hooks.py
before_request = ["sample_app.request.before_request"]
after_request = ["sample_app.request.after_request"]
```

```python
# sample_app/request.py
import frappe


def before_request():
	frappe.local.sample_started = frappe.utils.now()


def after_request(response, request):
	if response is not None:
		response.headers["X-Sample-App"] = "1"
```

The `X-Sample-App` header was on responses with HTTP 200, 401, 403, 404 and 417. It was not on the HTTP 405 response.

- In `before_request`, `frappe.session.user` is the cookie user, or `Guest`. A user from an `Authorization` header or `auth_hooks` is not set yet.
- An exception in `before_request` stops the request and returns an error response.
- `after_request` gets `response=None` when a `werkzeug` `HTTPException` (for example HTTP 405) ends the request. Check for `None`.
- An exception in `after_request` is logged and ignored. Writes in `after_request` are not committed by the request.
- These hooks run on every request that reaches `application()`, also for desk and website pages. Keep them fast.

## Authenticate with auth_hooks

`validate_auth()` in `frappe/auth.py` reads the `Authorization` header (token, Basic, Bearer), then calls each `auth_hooks` function with no arguments. A hook sets the user with `frappe.set_user()`.

```python
# sample_app/hooks.py
auth_hooks = ["sample_app.auth.validate_sample_key"]
```

```python
# sample_app/auth.py
import frappe


def validate_sample_key():
	if frappe.get_request_header("X-Sample-Key") != "let-me-in":
		return
	form_dict = frappe.local.form_dict
	frappe.set_user("user1@example.com")
	frappe.local.form_dict = form_dict
```

!!! warning
    `frappe.set_user()` resets `frappe.local.form_dict` to an empty dict. Save it before the call and put it back after, as the built-in token check does. Without this, the request loses all its arguments. In the test, `POST /api/v2/document/ToDo` then failed with `MandatoryError`.

- If the `Authorization` header has two parts and no check sets a user, Frappe raises `frappe.AuthenticationError` (HTTP 401) after the hooks.
- A header with one part (for example `Authorization: foo`) is ignored. The request runs as `Guest`.
- When a hook changes the user, Frappe checks the `restrict_ip` of that user.
- A real hook must compare a secret, for example a hash stored in a DocType. The fixed value above is only for the example.

See [Authentication](../security/authentication.md#custom-authentication).

## Rate limits

- `@rate_limit(...)` from `frappe.rate_limiter` limits calls of one method per IP, per form value (`key`), or both. Put it below `@frappe.whitelist()`. Over the limit: `frappe.RateLimitExceededError` (HTTP 429).
- Site config `rate_limit` (`{"limit": <seconds>, "window": <seconds>}`) limits total request time for the site. Responses get `X-RateLimit-Reset`, `X-RateLimit-Limit` and `X-RateLimit-Remaining` headers.

```python
# sample_app/api.py
import frappe
from frappe.rate_limiter import rate_limit


@frappe.whitelist(allow_guest=True)
@rate_limit(limit=5, seconds=60)
def request_demo(email: str):
	...
```

!!! warning
    The `@rate_limit` counter key contains `frappe.form_dict.cmd`. Only v1 sets `cmd`. Under `/api/v2/method/...`, all methods with `@rate_limit` share one counter for each IP. In the test, two calls of one method made the first call of another method fail with HTTP 429.

See [Whitelisted Methods](whitelisted-methods.md#rate-limits).

## Custom routes outside /api

`application()` sends every path that starts with `/api/` to the API URL map. Other `GET`, `HEAD` and `POST` paths, except `/backups`, `/private/files` and `/.well-known/security.txt`, go to the website router (`frappe.website.serve.get_response()`). So a custom route must be outside `/api`.

| Hook or folder | Use |
|---|---|
| `website_route_rules` | Map a URL pattern to a page. Path parts go into `frappe.form_dict`. |
| `page_renderer` | A class with `can_render()` and `render()`. It runs before the built-in renderers and can return JSON or any `Response`. |
| `www/` pages | HTML pages with `get_context`. |

These routes use the same session, commit rules and `before_request`/`after_request` hooks as `/api`. They do not use `@frappe.whitelist`. Check permissions in your code. Errors give an HTML page, unless the request has an `Accept` header and is AJAX or accepts `application/json`. See [Website](../website/website.md#custom-page-renderers).

## Custom response formats

A whitelisted method can return a file, a CSV, a redirect or a web page. Set `frappe.response.type`, or return a `werkzeug` `Response`:

```python
# sample_app/api.py
import frappe
from werkzeug.wrappers import Response


@frappe.whitelist()
def order_label(name: str):
	doc = frappe.get_doc("Sample Order", name)
	doc.check_permission("read")
	frappe.response.type = "download"
	frappe.response.filename = f"{doc.name}.txt"
	frappe.response.filecontent = f"Order {doc.name} for {doc.customer_name}"


@frappe.whitelist()
def ping_text():
	return Response("pong", mimetype="text/plain")
```

See [API Internals](api-internals.md#response-types) for all types and the keys they read.

## CORS

Site config `allow_cors` sets the origins that may call the site from a browser. The value is `"*"`, one origin, or a list. `process_response()` adds the headers only when the request has an `Origin` header that matches.

```bash
bench --site frappe.localhost set-config allow_cors '["https://app.example.com"]' --parse
```

A preflight `OPTIONS` request gets HTTP 200 and no body. Frappe does not run the endpoint. Response headers for a matching origin:

```http
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: https://app.example.com
Vary: Origin
Access-Control-Allow-Methods: POST
Access-Control-Allow-Headers: Content-Type
Access-Control-Max-Age: 86400
```

`Access-Control-Allow-Methods` and `Access-Control-Allow-Headers` repeat the values that the browser asked for. `Access-Control-Max-Age` is not sent in developer mode. An origin that does not match gets no CORS headers. In v15, only the site config sets the allowed origins. There is no per-request override.

## Version your API

Frappe has no version setting for app methods. The dotted path is the contract.

- Do not rename or move a whitelisted function. Clients call it by module path.
- Add new parameters with a default. Do not change the type or meaning of an existing parameter.
- For a breaking change, add a new function, for example `sample_app.api.v2.close_order`, and keep the old one.
- The v1 and v2 envelopes differ (`message` against `data`). Pick one for each client and keep it.

## Deprecate a method

1. Add the new function.
2. Keep the old function as a whitelisted wrapper that calls the new one.
3. Log each call, so that you know who still uses it.
4. Remove the old function in a later major version of your app.

```python
# sample_app/api.py
import frappe


@frappe.whitelist(methods=["POST"])
def close_order_v2(name: str, reason: str | None = None) -> str:
	...


@frappe.whitelist(methods=["POST"])
def close_order(name: str) -> str:
	frappe.logger("sample_app").warning(f"deprecated sample_app.api.close_order called by {frappe.session.user}")
	return close_order_v2(name)
```

You can also point the old path at the new function with `override_whitelisted_methods` in your own `hooks.py`. `frappe/utils/deprecations.py` is internal to Frappe. Do not import it in your app.

## Test an API

Call the function directly in a `FrappeTestCase` (`frappe.tests.utils`) to test its logic. Test the HTTP layer with a request, so that whitelist, HTTP method, type and permission checks run.

`frappe.tests.test_api.FrappeAPITestCase` is the base class that Frappe uses for its own API tests. It is in a test module, not a public API, so it can change. It sends requests to the site with the Werkzeug test client (`frappe.utils.get_test_client()`), in a separate thread:

| Member | Does |
|---|---|
| `version` | `""` (v1, default) or `"v2"` |
| `method(path)`, `resource(doctype, name)`, `doctype_path(...)` | Build the URL |
| `sid` | Session id of Administrator |
| `get`, `post`, `put`, `patch`, `delete` | Send a request. The data is sent as JSON. |

```python
# sample_app/tests/test_api.py
from frappe.tests.test_api import FrappeAPITestCase


class TestSampleAPI(FrappeAPITestCase):
	version = "v2"

	def test_logged_user(self):
		response = self.get(self.method("frappe.auth.get_logged_user"), {"sid": self.sid})
		self.assertEqual(response.status_code, 200)
		self.assertEqual(response.json["data"], "Administrator")
```

`TEST_CLIENT` is a class attribute, so all subclasses share one client and its cookies. After a request with `sid`, later requests without `sid` also run as Administrator. Also test as a normal user (for example with API keys of `user1@example.com`), not only as Administrator. See [Testing](../testing-and-performance/testing.md).

## Pitfalls

- An `auth_hooks` function that calls `frappe.set_user()` without saving `frappe.local.form_dict` drops all request arguments.
- `before_request` runs before token authentication. Do not check the user there for token requests.
- `override_whitelisted_methods` does not change document routes. Use `doc_events` or a controller hook.
- `has_permission` does not filter lists. Pair it with `permission_query_conditions`.
- A `has_permission` hook that returns `None` does not deny. Return `False` to deny.
- `doc_events` and controller methods run for desk, REST and Python calls alike. Check `frappe.request` only if you must know the source.
- A route outside `/api` is not protected by `@frappe.whitelist`. Check permissions in the renderer or page.

## Related

- [API Internals](api-internals.md)
- [Request Lifecycle](request-lifecycle.md)
- [Whitelisted Methods](whitelisted-methods.md)
- [REST API](rest-api.md)
- [Extending Other Apps](../building-apps/extending-apps.md)
- [Hooks reference](../../reference/configuration/hooks.md)
- [Permissions](../security/permissions.md)
- [Authentication](../security/authentication.md)
- [Website](../website/website.md)
- [Configuration](../operations/configuration.md)
