---
title: API Internals
description: How Frappe routes an /api request to a function, checks whitelist, guest and HTTP method rules, binds arguments, and turns frappe.response into JSON, files, redirects and errors.
version: develop
sources:
  - frappe/app.py
  - frappe/api/__init__.py
  - frappe/api/v1.py
  - frappe/api/v2.py
  - frappe/handler.py
  - frappe/__init__.py
  - frappe/auth.py
  - frappe/utils/response.py
  - frappe/utils/messages.py
  - frappe/utils/csvutils.py
  - frappe/utils/typing_validations.py
  - frappe/exceptions.py
  - frappe/permissions.py
---

# API Internals

This page shows what Frappe does with a request after the path starts with `/api/`. Read it to debug an API call, to understand an error response, or to return something other than JSON. For how to call the API, see [REST API](rest-api.md). For the steps before `/api` dispatch (site, session, `before_request`), see [Architecture](../../getting-started/architecture.md).

All examples were run on `http://frappe.localhost:8000` with `bench start`. Responses are shortened. Long tracebacks are shown as `"<traceback>"`.

## Where `/api` starts

`application()` in `frappe/app.py` sends every path that starts with `/api/` to `frappe.api.handle(request)`. Before that, `init_request()` has built `frappe.form_dict` and loaded the session, and `validate_auth()` has set the user from the `Authorization` header or `auth_hooks`.

!!! note
    A request with a `cmd` form value goes to `frappe.handler.handle()` before the `/api/` check. This form is deprecated. Use `/api/method/<dotted.path>`.

`frappe.api.handle()` (`frappe/api/__init__.py`) does this:

1. If System Settings `log_api_requests` is on, it queues an API Request Log with the path, user and HTTP method.
2. It matches the path against `API_URL_MAP`. No match raises `frappe.DoesNotExistError` (HTTP 404).
3. It calls the endpoint with the URL arguments (for example `doctype`, `name`, `method`).
4. If the endpoint returns a `werkzeug` `Response`, Frappe sends it as is.
5. Otherwise, a return value that is not `None` goes into `frappe.response["data"]`.
6. `build_response("json")` makes the response from `frappe.response`.

## URL map

`API_URL_MAP` is a Werkzeug `Map` with three mounts. `strict_slashes=False`, so a trailing slash is optional.

| Mount | Rules | Endpoints in |
|---|---|---|
| `/api` | v1 | `frappe/api/v1.py` |
| `/api/v1` | v1 (same rules) | `frappe/api/v1.py` |
| `/api/v2` | v2 | `frappe/api/v2.py` |

v1 rules:

| Rule | HTTP methods | Endpoint |
|---|---|---|
| `/method/<path:method>` | all | `handle_rpc_call` |
| `/resource/<doctype>` | `GET`, `QUERY` | `document_list` |
| `/resource/<doctype>` | `POST` | `create_doc` |
| `/resource/<doctype>/<path:name>` | `GET` | `read_doc` (or `execute_doc_method` with `run_method`) |
| `/resource/<doctype>/<path:name>` | `PUT` | `update_doc` |
| `/resource/<doctype>/<path:name>` | `DELETE` | `delete_doc` |
| `/resource/<doctype>/<path:name>` | `POST`, `QUERY` | `execute_doc_method` |

v2 rules (discovery and MCP rules are left out, see [MCP and API Discovery](mcp-and-discovery.md)):

| Rule | HTTP methods | Endpoint |
|---|---|---|
| `/method/login` | all | `login` (does nothing) |
| `/method/logout` | `POST` | `logout` |
| `/method/ping` | all | `frappe.ping` |
| `/method/upload_file` | `POST` | `frappe.handler.upload_file` |
| `/method/bulk_delete`, `/method/bulk_update` | `POST` | `bulk_delete`, `bulk_update` |
| `/method/run_doc_method` | `GET`, `POST`, `QUERY` | `run_doc_method` |
| `/method/<method>` | all | `handle_rpc_call` |
| `/method/<doctype>/<method>` | all | `handle_rpc_call` with `doctype` |
| `/document/<doctype>` | `GET`, `QUERY` | `document_list` |
| `/document/<doctype>` | `POST` | `create_doc` |
| `/document/<doctype>/bulk_delete`, `/document/<doctype>/bulk_update` | `POST` | `bulk_delete_docs`, `bulk_update_docs` |
| `/document/<doctype>/<path:name>` | `GET` | `read_doc` |
| `/document/<doctype>/<path:name>/copy` | `GET` | `copy_doc` |
| `/document/<doctype>/<path:name>` | `PATCH`, `PUT` | `update_doc` |
| `/document/<doctype>/<path:name>` | `DELETE` | `delete_doc` |
| `/document/<doctype>/<path:name>/method/<method>` | `GET`, `POST`, `QUERY` | `execute_doc_method` |
| `/doctype/<doctype>/meta` | `GET` | `get_meta` |
| `/doctype/<doctype>/count` | `GET`, `QUERY` | `count` |

Rules with a fixed path win over rules with variables. So `/api/v2/method/run_doc_method` goes to `run_doc_method`, not to `handle_rpc_call`. The v2 `<method>` part cannot contain `/`. The v1 `<path:method>` part can, and v1 drops everything after the first `/`.

The endpoints in `frappe/api/v2.py` are not whitelisted functions. The route itself is the permission to call them. They check document permissions in their own code.

### Route errors

| Request | Result |
|---|---|
| Path that no rule matches, for example `/api/nothing/here` | HTTP 404, JSON `{"exc_type": "DoesNotExistError"}` (v2: `{"errors": [{"type": "DoesNotExistError"}]}`) |
| Path matches, HTTP method does not, for example `DELETE /api/v2/doctype/ToDo/meta` | HTTP 405, Werkzeug HTML page, not JSON |
| `GET /api/v2/method/logout` | The `POST` rule does not match, so `/method/<method>` resolves `logout` as a dotted path. HTTP 417 `AppNotInstalledError` "App logout is not installed" |

```bash
curl -s -X DELETE "$S/api/v2/doctype/ToDo/meta" -b cookies.txt -w '\n%{http_code}\n'
```

```text
<!doctype html>
<html lang=en>
<title>405 Method Not Allowed</title>
...
405
```

## How a method call is resolved

`/api/method/<path>` and `/api/v2/method/<path>` both end in the same checks. v1 goes through `frappe.handler.handle()` and `execute_cmd()`. v2 calls `handle_rpc_call()` in `frappe/api/v2.py`.

1. v1 only: Frappe sets `frappe.form_dict.cmd` to the dotted path. If the path is `login`, nothing is called. The login already ran in `LoginManager` (see [Login](#login)).
2. v2 with `/method/<doctype>/<method>`: Frappe loads the controller module of the DocType and makes the path `<module>.<method>`.
3. `frappe.override_whitelisted_method(path)` replaces the path with the last value of `override_whitelisted_methods` for that key.
4. If an enabled Server Script of type API has this path as its method name, the script runs. The steps below are skipped.
5. `resolve_method()` imports the function with `frappe.get_attr`. A path without a dot is looked up in `frappe/handler.py` (v1 only, deprecated).
6. `frappe.is_whitelisted(fn)` checks the whitelist (see [Whitelist checks](#whitelist-checks)).
7. `is_valid_http_method(fn)` checks the HTTP method against the `methods` of `@frappe.whitelist`.
8. `frappe.call(fn, **frappe.form_dict)` calls the function (see [Argument binding](#argument-binding)).
9. v1 puts the return value in `frappe.response["message"]`. v2 puts it in `frappe.response["data"]`. A `None` return value adds no key.

`frappe.handler.run_doc_method` is the one exception in v1: `execute_cmd()` does not run steps 6 and 7 on it. It checks the whitelist and HTTP method on the controller method instead.

If the import fails in step 5, a user who may see tracebacks gets the reason. Other users get "Method not found". Both are `frappe.ValidationError` (HTTP 417).

```bash
curl -s -b cookies.txt "$S/api/v2/method/frappe.nonexistent.fn"
```

```json
{"errors": [{"type": "ValidationError", "exception": "<traceback>",
  "message": "Failed to get method for command frappe.nonexistent.fn with No module named 'frappe.nonexistent'",
  "as_table": false, "title": "Message", "indicator": "red"}]}
```

### Whitelist checks

`frappe.is_whitelisted(fn)` in `frappe/__init__.py` raises an error in these cases:

| Case | Exception | HTTP |
|---|---|---|
| The app of the function (first part of `fn.__module__`) is disabled on the site | `frappe.AppDisabledError` | 417 |
| `fn` has no `@frappe.whitelist()` | `frappe.PermissionError` "Function ... is not whitelisted." | 403 |
| The user is `Guest` and `fn` has no `allow_guest=True` | `frappe.PermissionError` | 403 |
| As above, and the session cookie has expired (`frappe.response.session_expired`) | `frappe.SessionExpired` | 401 |

For a guest call to a method without `xss_safe=True`, Frappe then sanitizes every string in `frappe.form_dict` with `frappe.utils.sanitize_html_payload`. `<script>x</script><b>hi</b>` becomes `<b>hi</b>`.

`is_valid_http_method(fn)` raises `frappe.PermissionError` "Not permitted" (HTTP 403) when the request method is not allowed. It does not check in a background job or in a Server Script (`frappe.flags.in_safe_exec`). `@frappe.whitelist()` without `methods` allows `GET`, `POST`, `PUT`, `DELETE` and `QUERY`. `PATCH` is not in this list.

```bash
curl -s -b cookies.txt -X PATCH "$S/api/method/frappe.auth.get_logged_user" -w ' %{http_code}\n'
```

```json
{"exception": "frappe.exceptions.PermissionError: Not permitted", "exc_type": "PermissionError", "exc": "<traceback>",
 "_server_messages": "[\"{\\\"message\\\":\\\"Not permitted\\\", ...}\"]"} 403
```

### Controller methods

A whitelisted method on a DocType controller runs through one of these routes:

| Route | Loads the document with | Permission checked |
|---|---|---|
| v1 `POST /api/resource/<doctype>/<name>` with `run_method` | `frappe.get_doc(doctype, name)` | `write` |
| v1 `GET` or `QUERY /api/resource/<doctype>/<name>` with `run_method` | same | `read` |
| v2 `/api/v2/document/<doctype>/<name>/method/<method>` | same | `read` for `GET`, `QUERY`. `write` for `POST` |
| v1 `/api/method/run_doc_method` (`frappe.handler.run_doc_method`), args `method`, `dt`, `dn` or `docs`, `args` | saved document with `check_permission=True` (read), or the `docs` dict | `read` |
| v2 `/api/v2/method/run_doc_method`, args `method`, `document`, `kwargs` | the `document` dict, checked with `check_if_latest()` | `read` for `GET`. `write` (`create` if not saved) for `POST` |

In each case Frappe checks `doc.is_whitelisted(method)` (or `frappe.is_whitelisted`) and the HTTP method, then calls `doc.run_method(method, ...)`. The document after the call is added to `frappe.response.docs`, with fields the user cannot read removed. v1 `run_doc_method` with `as_csv=1` returns the result as a CSV file.

## Argument binding

`init_request()` builds `frappe.form_dict` once per request (`make_form_dict()` in `frappe/app.py`):

| Request body | `frappe.form_dict` |
|---|---|
| `Content-Type: application/json`, a JSON object | The object only. The query string is ignored. |
| `Content-Type: application/json`, a JSON list | `{"data": [...]}` |
| Invalid JSON | `frappe.DataError` "Invalid request body" (HTTP 417) |
| Form or multipart body, or no body | Query string values, then form values (form wins) |

The key `_` is removed. Files of a multipart body are in `frappe.request.files`, not in `frappe.form_dict`.

`frappe.call(fn, **frappe.form_dict)` then uses `frappe.get_newargs()`:

- Keys that are not parameters of `fn` are dropped, unless `fn` has `**kwargs`.
- `ignore_permissions` and `flags` are always dropped.
- In v1, `cmd` is in `frappe.form_dict`, so a function with `**kwargs` gets `cmd` too.
- A missing required argument raises Python `TypeError` (HTTP 500).

The function that `@frappe.whitelist()` registers is a wrapper from `validate_argument_types()`. During a request, it converts and checks each annotated argument. A bad value raises `frappe.FrappeTypeError` (HTTP 417). See [Whitelisted Methods](whitelisted-methods.md#type-annotations).

Results of a test function `def r2_args(a: int, b: str = "x")`:

| Request | `message` |
|---|---|
| `GET ...?a=1&b=2&c=3` | `{"a": 1, "b": "2"}` |
| `POST ...?b=fromquery` with JSON `{"a": 5}` | `{"a": 5, "b": "x"}` |
| `POST ...?b=fromquery` with form `a=5` | `{"a": 5, "b": "fromquery"}` |
| `POST` with JSON `[1, 2]` | `TypeError: r2_args() missing 1 required positional argument: 'a'` (HTTP 500) |

## frappe.response

`frappe.response` (`frappe.local.response`) is a dict that `frappe.init()` creates for each request as `{"docs": []}`. Code adds keys to it. `as_json()` serializes all keys of the dict, so a custom key appears in the JSON. Keys that control the response:

| Key | Set by | Becomes |
|---|---|---|
| `message` | v1 method return value | `message` in JSON |
| `data` | v1 resource and all v2 return values | `data` in JSON |
| `docs` | document methods, v2 bulk update | `docs` in JSON. Removed when empty. |
| `has_next_page` | v2 list | `has_next_page` in JSON |
| `http_status_code` | your code, `delete_doc` (202), bulk routes (202) | HTTP status. Removed from the JSON. |
| `type` | your code | The response builder, see [Response types](#response-types) |
| `filename`, `filecontent`, `content_type`, `display_content_as` | your code | File for `download`, `pdf`, `binary` |
| `result`, `doctype` | `build_csv_response()` | Body and file name for `csv` and `txt` |
| `location` | your code | Target of `redirect` |
| `route`, `context` | `frappe.respond_as_web_page()` | Page for `page` |

```python
# test function in the console, then GET /api/method/<path>
@frappe.whitelist()
def r2_none():
    frappe.response["custom"] = 5
```

```json
{"custom": 5}
```

### Messages and logs

`make_logs()` in `frappe/utils/response.py` adds messages to JSON responses. The shape depends on the API version (`frappe.api.get_api_version()`: v2 when the path starts with `/api/v2`, else v1).

| Source | v1 key | v2 key |
|---|---|---|
| `frappe.msgprint()`, `frappe.throw()` (`frappe.local.message_log`) | `_server_messages`: a JSON string of a list of JSON strings | `messages`: a list of objects. A `throw` message moves into its error in `errors`. |
| `frappe.log()` (`frappe.local.debug_log`) | `_debug_messages`: a JSON string. Only if tracebacks are allowed. | `debug`: list of `{"message": ...}`. Only if tracebacks are allowed. |
| `frappe.errprint()` and error tracebacks (`frappe.local.error_log`) | `exc`: a JSON string of a list of tracebacks. `_exc_source`: the app that raised. Only if tracebacks are allowed. | not sent. The traceback is in `errors[].exception`. |
| `frappe.flags.error_message` (set by `frappe.has_permission(..., throw=True)`) | `_error_message` | not sent |

```python
@frappe.whitelist()
def r2_msg():
    frappe.msgprint("Saved two rows")
    frappe.log("debug line")
    return {"ok": 1}
```

v1 (`/api/method/...`):

```json
{"message": {"ok": 1},
 "_server_messages": "[\"{\\\"message\\\":\\\"Saved two rows\\\",\\\"as_table\\\":false,\\\"title\\\":\\\"Message\\\"}\"]",
 "_debug_messages": "[\"debug line\"]"}
```

v2 (`/api/v2/method/...`):

```json
{"data": {"ok": 1},
 "messages": [{"message": "Saved two rows", "as_table": false, "title": "Message"}],
 "debug": [{"message": "debug line"}]}
```

`frappe.msgprint(..., realtime=True)` sends the message over the realtime connection instead of the response.

### Serialization

`as_json()` uses `orjson` with `json_handler()`. Dates and times become strings, `timedelta` becomes `H:MM:SS`, `Decimal` becomes a float, a `Document` becomes its dict (`__json__`), and other iterables become lists. Another object raises `TypeError` (HTTP 500).

## Response types

Set `frappe.response.type` to use another builder (`build_response()` in `frappe/utils/response.py`). The return value of the function is ignored by these builders, so return `None`.

| `type` | Uses | HTTP response |
|---|---|---|
| `json` (default) | all keys | `application/json` |
| `download` | `filename`, `filecontent`, optional `content_type`, `display_content_as` (default `attachment`) | MIME type from `content_type` or the file name |
| `pdf` | `filename`, `filecontent` | `application/pdf`, `inline` |
| `binary` | `filename`, `filecontent` | `application/octet-stream`, `attachment` |
| `csv` | `result`, `doctype` (file name without `.csv`) | `text/csv`, `attachment; filename=<doctype>.csv` |
| `txt` | `result`, `doctype` | `text`, `attachment; filename=<doctype>.txt` |
| `redirect` | `location` | HTTP 302 to `location` |
| `page` | `route`, `context`, `http_status_code` | HTML page. `frappe.respond_as_web_page(title, html)` sets it. |

`download`, `pdf`, `binary`, `csv` and `txt` do not add messages, and they do not read `http_status_code`.

```python
# sample_app/api.py
import frappe
from frappe.utils.csvutils import build_csv_response


@frappe.whitelist()
def export_notes():
    frappe.response.type = "download"
    frappe.response.filename = "sample.txt"
    frappe.response.filecontent = "hello"


@frappe.whitelist()
def export_items():
    build_csv_response([["name", "rate"], ["Laptop", 900]], "SampleItems")
```

```text
GET /api/method/sample_app.api.export_notes -> 200
Content-Type: text/plain; charset=utf-8
Content-Disposition: attachment; filename=sample.txt

GET /api/method/sample_app.api.export_items -> 200
Content-Type: text/csv; charset=utf-8
Content-Disposition: attachment; filename=SampleItems.csv
```

### Return a Response object

A function can return a `werkzeug.wrappers.Response`. Both v1 and v2 send it without changes. No messages are added. Frappe still adds `Cache-Control`, CORS and cookie headers in `process_response()`.

```python
# sample_app/api.py
import frappe
from werkzeug.wrappers import Response


@frappe.whitelist()
def plain():
    return Response("plain text", mimetype="text/plain", status=201)
```

```text
GET /api/v2/method/sample_app.api.plain -> 201 text/plain; charset=utf-8
plain text
```

To change only the status of a JSON response, set `frappe.response.http_status_code`:

```python
@frappe.whitelist(methods=["POST"])
def create_sample():
    frappe.response.http_status_code = 201
    return "created"
```

```json
{"message": "created"}
```

## v1 and v2 envelopes

| | v1 (`/api/...`, `/api/v1/...`) | v2 (`/api/v2/...`) |
|---|---|---|
| Method result | `message` | `data` |
| Resource result | `data` | `data` |
| Messages | `_server_messages` (JSON string) | `messages` (list) |
| Error | `exc_type`, `exception`, `exc`, `_server_messages` | `errors`: list with one object |
| Error message text | in `_server_messages` | in `errors[0].message` |

v1 error keys (`report_error()` in `frappe/utils/response.py`):

| Key | Content | When |
|---|---|---|
| `exc_type` | Class name of the exception | always |
| `exception` | Last line of the traceback, for example `frappe.exceptions.PermissionError: Not permitted` | tracebacks allowed |
| `exc` | JSON string of a list with the full traceback | tracebacks allowed |
| `_server_messages` | The `throw` message | when the code called `frappe.throw` or `msgprint` |

v2 error object keys: `type`, `exception` (full traceback, when allowed), and the keys of the linked `throw` message (`message`, `title`, `indicator`, `as_table`).

Tracebacks are allowed (`is_traceback_allowed()`) on the development server (`bench start` sets `DEV_SERVER`), or when System Settings `allow_error_traceback` is on (default `1`) and the user is a System User. They are never sent for HTTP 404, unless site config `logging` is set. A CSRF error turns them off.

The same Guest call, on a production-like server (no `DEV_SERVER`):

```json
{"exc_type": "PermissionError",
 "_server_messages": "[\"{\\\"message\\\":\\\"<p>You are not permitted to access this resource. Login to access</p>...\\\", ...}\"]"}
```

!!! warning
    An error raised without `frappe.throw` has no message. For example, `frappe.FrappeTypeError` from type validation. Without tracebacks, a v2 client gets only `{"errors": [{"type": "FrappeTypeError"}]}`, and a v1 client gets only `exc_type`.

## Errors and HTTP status

An exception in the endpoint goes to `handle_exception()` in `frappe/app.py`:

1. The status is `http_status_code` of the exception class, or `500`.
2. If the exception is `frappe.DoesNotExistError` and the user has no permission on its DocType, Frappe raises `frappe.PermissionError` instead. A user without access does not learn if a document exists.
3. For `/api/` paths (unless `Accept` starts with `text`), and for requests with `Accept: application/json` or `X-Requested-With: XMLHttpRequest`, the response is JSON from `report_error()`. Other requests get an HTML error page.
4. `frappe.AuthenticationError` clears the login cookies.
5. For status 500 and higher, or in developer mode, Frappe saves an Error Log (`log_error_snapshot()`).
6. `application()` rolls back the transaction.

A `werkzeug.exceptions.HTTPException` (for example `NotFound`, `Forbidden`, `MethodNotAllowed`) skips `handle_exception()`. Werkzeug makes its own HTML response.

| HTTP | Exception classes (`frappe/exceptions.py`) |
|---|---|
| 200 | `frappe.RequestToken` |
| 400 | `frappe.CSRFTokenError` and its subclasses `InvalidAuthorizationHeader`, `InvalidAuthorizationPrefix`, `InvalidAuthorizationToken` |
| 401 | `frappe.AuthenticationError`, `frappe.SessionExpired` |
| 403 | `frappe.PermissionError` |
| 404 | `frappe.DoesNotExistError`, `frappe.PageDoesNotExistError` |
| 409 | `frappe.NameError`, `frappe.DuplicateEntryError` |
| 410 | `frappe.LinkExpired` |
| 415 | `frappe.UnsupportedMediaType` |
| 417 | `frappe.ValidationError` and subclasses without their own code (for example `MandatoryError`, `LinkValidationError`, `DataError`, `AppNotInstalledError`), `frappe.FrappeTypeError` |
| 429 | `frappe.RateLimitExceededError`, `frappe.TooManyRequestsError` |
| 500 | Other exceptions, `frappe.SessionBootFailed` |
| 501 | `frappe.OutgoingEmailError` |
| 503 | `frappe.SessionStopped`, `frappe.ServiceUnavailableError`, `frappe.InReadOnlyMode`, `frappe.QueueOverloaded` |
| 508 | `frappe.QueryTimeoutError`, `frappe.QueryDeadlockError` |

`frappe.Redirect(http_status_code=301)` takes the status as an argument. For your own exception, subclass one of these classes or set `http_status_code` on the class.

```python
# sample_app/exceptions.py
import frappe


class SampleQuotaExceeded(frappe.ValidationError):
    http_status_code = 402
```

### msgprint and throw in an API call

- `frappe.msgprint(msg)` adds a message and the call continues. The message is in `_server_messages` (v1) or `messages` (v2) of a successful response.
- `frappe.throw(msg, exc=frappe.ValidationError)` adds a message with `raise_exception` and raises `exc`. The status comes from `exc`. In v2, the message moves from `messages` into the error object.
- `frappe.throw_permission_error()` raises `frappe.PermissionError` with "Not permitted".
- When `frappe.flags.mute_messages` is set, no message is added, but the exception is still raised.

## Login

`LoginManager.__init__()` in `frappe/auth.py` logs in only when `frappe.request.path == "/api/method/login"`. It reads `usr` and `pwd` from `frappe.form_dict`. `frappe.handler.handle()` then skips the `login` command.

!!! warning
    `/api/v1/method/login` and `/api/v2/method/login` do not log in. They return HTTP 200 with `{}`, also for a wrong password, and start a Guest session. Use `POST /api/method/login`.

## File uploads

`upload_file` (`frappe.handler.upload_file`, also `POST /api/v2/method/upload_file`) reads a multipart body:

| Form field | Default | Description |
|---|---|---|
| `file` | none | The file content. Without it, `file_url` or `library_file_name` gives the file. |
| `is_private` | `1` | `0` saves under `/files/`, `1` under `/private/files/`. |
| `doctype`, `docname`, `fieldname` | none | Attach to this document. The user needs `write` on it. |
| `folder` | `Home` | File folder. |
| `file_url` | none | Make a File record for an existing URL. |
| `optimize`, `max_width`, `max_height` | none | Shrink images and PDFs. |
| `chunk_index`, `total_chunk_count`, `chunk_byte_offset`, `total_file_size` | none | Upload in parts. Only the last part creates the File. |
| `method` | none | Dotted path of a whitelisted function to call instead of creating a File. It reads `frappe.local.uploaded_file` and `frappe.local.uploaded_filename`. |

Rules:

- A Guest can upload only when System Settings `allow_guests_to_upload_files` is on. Otherwise HTTP 403.
- Guests and users without desk access can upload only the MIME types in `ALLOWED_MIMETYPES` (`frappe/handler.py`).
- The `after_file_upload` hooks run with `doc=` before the File is saved.
- The method is rate limited to 300 calls per 60 seconds per user.

```bash
curl -s -b cookies.txt -X POST "$S/api/v2/method/upload_file" -F "file=@sample-notes.txt" -F "is_private=0"
```

```json
{"data": {"name": "5da45e78f1", "file_name": "sample-notes.txt", "is_private": 0, "file_size": 13,
  "file_url": "/files/sample-notes.txt", "folder": "Home", "doctype": "File"}}
```

`frappe.handler.download_file?file_url=<url>` returns a file with `type = "download"`. It raises `frappe.PermissionError` when the user cannot read the File. See [Files](../data-model/files.md).

## Commit and after the response

`frappe.api.handle()` does not commit. `application()` does it after the endpoint returns (`sync_database()`):

- `POST`, `PUT`, `DELETE` and `PATCH` commit. `GET`, `HEAD`, `OPTIONS` and `QUERY` roll back, unless `frappe.local.flags.commit` is set.
- Any exception rolls back.
- `after_request` hooks run after the commit or rollback, also for errors.

See [Database](../database/database.md#transactions).

## Pitfalls

- A JSON body hides the query string. Send all arguments in the body, or use a form body.
- A wrong HTTP method on a route returns a Werkzeug HTML page with HTTP 405, not JSON. Check the status before you parse the body.
- `GET /api/v2/method/logout` returns HTTP 417, because only `POST` matches the `logout` rule.
- Do not return a value and also set `frappe.response.type = "download"`. The return value is not sent.
- Messages from `frappe.msgprint` are not in responses that you build as a `Response` object or a file.
- A Server Script of type API with the same method name replaces your function on both v1 and v2.

## Related

- [REST API](rest-api.md)
- [Whitelisted Methods](whitelisted-methods.md)
- [Customizing APIs](customizing-apis.md)
- [REST Endpoints Reference](../../reference/http/rest-endpoints.md)
- [Authentication](../security/authentication.md)
- [Architecture](../../getting-started/architecture.md)
