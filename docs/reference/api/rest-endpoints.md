---
title: REST Endpoints Reference
description: Every route under /api in Frappe (v1, v2, discovery, MCP) and the most used whitelisted methods, with HTTP methods and permission checks.
version: v16
sources:
  - frappe/api/__init__.py
  - frappe/api/v1.py
  - frappe/api/v2.py
  - frappe/api/discovery.py
  - frappe/mcp/__init__.py
  - frappe/mcp/transport.py
  - frappe/mcp/tools/__init__.py
  - frappe/handler.py
  - frappe/auth.py
  - frappe/client.py
  - frappe/__init__.py
---

# REST Endpoints Reference

This page lists every route that Frappe serves under `/api`, and the core whitelisted methods that clients call most.
For how to authenticate, filter and page, see [REST API](../../developer-guide/api/rest-api.md) and [Authentication](../../developer-guide/security/authentication.md).

## Route maps

`frappe/api/__init__.py` mounts three route sets:

| Prefix | Rules from | Response key |
|---|---|---|
| `/api/` | `frappe/api/v1.py` | `message` for methods, `data` for resources |
| `/api/v1/` | `frappe/api/v1.py` (same rules) | same as `/api/` |
| `/api/v2/` | `frappe/api/v2.py` | `data` for all routes. Errors are in `errors` |

Trailing slashes are optional. When System Settings `log_api_requests` is on, each call writes an API Request Log.
All routes run as the session user. Permission checks are those of the called function.
An HTTP method that a route does not list returns HTTP 405.

## API v1

| Route | HTTP methods | Does | Permission check |
|---|---|---|---|
| `/api/method/<dotted.path>` | any allowed by the method | Calls a whitelisted function. Also runs API Server Scripts with that name. | `@frappe.whitelist` rules (guest, HTTP methods) |
| `/api/resource/<doctype>` | `GET` | List records through `frappe.client.get_list`. | `get_list` permission rules |
| `/api/resource/<doctype>` | `POST` | Insert a document from the JSON body (or `data`). | `create` |
| `/api/resource/<doctype>/<name>` | `GET` | Read one document. `expand_links=1` adds linked documents. | `read` (`doc.check_permission`) |
| `/api/resource/<doctype>/<name>` | `PUT` | Update fields and save. | `write` |
| `/api/resource/<doctype>/<name>` | `DELETE` | Delete. Returns HTTP 202 and `"ok"`. | `delete` |
| `/api/resource/<doctype>/<name>` | `POST` | Call the whitelisted controller method in `run_method`. | `write` |

List parameters for `GET /api/resource/<doctype>`: `fields`, `filters`, `or_filters`, `order_by`, `group_by`, `limit_start`, `limit_page_length` (or `limit`, default 20), `as_dict`, `expand`, `debug`.
`GET /api/resource/<doctype>/<name>?run_method=<method>` also calls a document method with `read` permission (kept for old clients).

```bash
curl -b cookies.txt "http://frappe.localhost:8006/api/resource/ToDo?fields=%5B%22name%22,%22status%22%5D&limit=2"
```

```json
{"data": [{"name": "hqcd9rprol", "status": "Open"}, {"name": "hqci4hjb0e", "status": "Open"}]}
```

## API v2

| Route | HTTP methods | Does | Permission check |
|---|---|---|---|
| `/api/v2/method/login` | any | Log in with `usr` and `pwd`. The login runs in `LoginManager` before the route. | none |
| `/api/v2/method/logout` | `POST` | Log out the session. | none |
| `/api/v2/method/ping` | any | Returns `"pong"`. | none |
| `/api/v2/method/upload_file` | `POST` | Upload a file. Same as `frappe.handler.upload_file`. | see [Files](../../developer-guide/data-model/files.md) |
| `/api/v2/method/run_doc_method` | `GET`, `POST` | Run a whitelisted method on a `document` dict that need not be saved. Args: `method`, `document`, `kwargs`. | `read` (GET), `write` or `create` (POST) |
| `/api/v2/method/<dotted.path>` | any allowed by the method | Calls a whitelisted function or API Server Script. Applies `override_whitelisted_methods`. | `@frappe.whitelist` rules |
| `/api/v2/method/<doctype>/<method>` | any allowed by the method | Calls a whitelisted module function in the DocType's controller file. | `@frappe.whitelist` rules |
| `/api/v2/document/<doctype>` | `GET` | List records. Returns `data` and `has_next_page`. | query permission rules |
| `/api/v2/document/<doctype>` | `POST` | Insert a document from the JSON body. | `create` |
| `/api/v2/document/<doctype>/<name>` | `GET` | Read one document. | `read` |
| `/api/v2/document/<doctype>/<name>` | `PATCH`, `PUT` | Update fields and save. | `write` |
| `/api/v2/document/<doctype>/<name>` | `DELETE` | Delete. Returns HTTP 202 and `"ok"`. | `delete` |
| `/api/v2/document/<doctype>/<name>/copy` | `GET` | A copy of the document without `name` and other private properties, ready to insert. The route keeps `no_copy` fields. | `read` |
| `/api/v2/document/<doctype>/<name>/method/<method>` | `GET`, `POST` | Call a whitelisted controller method. The saved document goes in `docs`. | `read` (GET), `write` (POST) |
| `/api/v2/doctype/<doctype>/meta` | `GET` | DocType meta. | logged-in user (`frappe.only_for("All")`) |
| `/api/v2/doctype/<doctype>/count` | `GET` | Count with optional `filters`. | `frappe.desk.reportview.get_count` rules |

List parameters for `GET /api/v2/document/<doctype>`: `fields`, `filters` (list or dict), `order_by`, `group_by`, `start` (default 0), `limit` (default 20), `as_dict` (default true), `debug`.
A controller can change the list query with a static `get_list(query)` method that returns a query builder object.

```bash
curl -b cookies.txt -X POST http://frappe.localhost:8006/api/v2/document/ToDo \
  -H 'Content-Type: application/json' -d '{"description": "Sample REST todo"}'
curl -b cookies.txt -X PATCH http://frappe.localhost:8006/api/v2/document/ToDo/2nvhphpj3b \
  -H 'Content-Type: application/json' -d '{"status": "Closed"}'
curl -b cookies.txt "http://frappe.localhost:8006/api/v2/doctype/ToDo/count"
```

```json
{"data": 4}
```

!!! note "Version"
    v16 has no v2 bulk routes (`bulk_delete`, `bulk_update`) and does not accept the `QUERY` HTTP method. Use `frappe.client.bulk_update` for many updates.

## API discovery

All discovery routes need the `Desk User` role. They are `GET` only.

| Route | Returns |
|---|---|
| `/api/v2/discovery` | Counts of methods and DocType methods, and links to the routes below. |
| `/api/v2/discovery/search?q=<text>` | Up to 50 methods whose text has every word of `q`. |
| `/api/v2/discovery/method` | All whitelisted methods the user can call. |
| `/api/v2/discovery/method/<dotted.path>` | One method: parameters, HTTP methods, docstring, and source when the app sets `expose_discovery_source`. |
| `/api/v2/discovery/doctype/<doctype>` | Whitelisted controller methods of a DocType. |
| `/api/v2/discovery/doctype/<doctype>/method/<method>` | One controller method. |

The discovery index is kept in the Redis cache for one hour.
When the index is missing, the route enqueues a build job on the `long` queue and returns HTTP 503 `DiscoveryCacheUnavailable`.
Try again after a few seconds. A background worker must run for the build to finish.
See [MCP and Discovery](../../developer-guide/api/mcp-and-discovery.md).

## MCP

| Route | HTTP methods | Does |
|---|---|---|
| `/api/v2/mcp` | `POST` (`GET` and `DELETE` return 405) | Model Context Protocol server over streamable HTTP with JSON responses. |

The route returns 404 unless System Settings `enable_mcp` is on.
A Guest request gets HTTP 401. A user without the `Desk User` role gets HTTP 403.
The request needs `Content-Type: application/json` (else 415) and an `Accept` header that allows `application/json` (else 406).
Tools: `search_doctypes`, `get_doctype_schema`, `search_methods`, `get_method_schema`, `list_document_methods`, `list_documents`, `get_document`, `count_documents`, `create_document`, `update_document`, `delete_document`, `call_method`, `call_document_method`.
Each tool runs with the permissions of the user.

## Login, logout and files

These work on `/api/method/...` (v1). In v2, use the routes in the v2 table.

| Method path | HTTP methods | Guest | Description |
|---|---|---|---|
| `login` | `POST` | yes | Log in. Args: `usr`, `pwd`. Returns `message`, `home_page`, `full_name` and sets the `sid` cookie. |
| `logout` | `POST` | yes | Log out the current session. |
| `frappe.handler.web_logout` | `POST` | yes | Log out and show a "Logged Out" web page. |
| `frappe.auth.get_logged_user` | any | no | Returns the user name of the session. |
| `frappe.ping` | any | yes | Returns `"pong"`. |
| `upload_file` (`frappe.handler.upload_file`) | `POST` | yes | Upload a file as multipart form data (`file`, `doctype`, `docname`, `fieldname`, `is_private`, `folder`, `file_url`). |
| `frappe.handler.download_file` | any | yes | Download a file by `file_url`, with session or token auth. |
| `frappe.core.doctype.user.user.reset_password` | `POST` | yes | Send a password reset link. Arg: `user`. |
| `frappe.core.doctype.user.user.update_password` | `POST` | yes | Set a new password with `key` or `old_password`. |
| `frappe.core.doctype.user.user.sign_up` | any | yes | Website sign up. Args: `email`, `full_name`, `redirect_to`. |

Short names such as `logout` and `upload_file` resolve to `frappe.handler` and are deprecated. Use the full dotted path in new code.

```bash
curl -c cookies.txt -X POST http://frappe.localhost:8006/api/method/login \
  -H 'Content-Type: application/json' -d '{"usr": "Administrator", "pwd": "password"}'
```

```json
{"message": "Logged In", "home_page": "desk", "full_name": "Administrator"}
```

## frappe.client methods

Call them as `/api/method/frappe.client.<name>` or `/api/v2/method/frappe.client.<name>`. Defined in `frappe/client.py`.
"any" means `GET`, `POST`, `PUT` and `DELETE`. This is the default of `@frappe.whitelist()`.

| Method | HTTP methods | Arguments | Description |
|---|---|---|---|
| `get_list` | any | `doctype`, `fields`, `filters`, `or_filters`, `group_by`, `order_by`, `limit_start`, `limit_page_length` (20), `parent`, `as_dict`, `expand`, `debug` | List records with permission checks. |
| `get_count` | any | `doctype`, `filters`, `cache`, `debug` | Count records. |
| `get` | any | `doctype`, `name` or `filters`, `parent` | Get one document as a dict. |
| `get_value` | any | `doctype`, `fieldname` (str or list), `filters`, `as_dict`, `parent` | Get field values of one record. |
| `get_single_value` | any | `doctype`, `field` | Get a field of a Single DocType. |
| `set_value` | `POST`, `PUT` | `doctype`, `name`, `fieldname` (str or dict), `value` | Set values and save the document. |
| `insert` | `POST`, `PUT` | `doc` | Insert a document. A child row needs `parenttype`, `parent`, `parentfield`. |
| `insert_many` | `POST`, `PUT` | `docs` | Insert many documents. |
| `save` | `POST`, `PUT` | `doc` | Save an existing document. |
| `rename_doc` | `POST`, `PUT` | `doctype`, `old_name`, `new_name`, `merge` | Rename a document. |
| `submit` | `POST`, `PUT` | `doc` | Submit a document. |
| `cancel` | `POST`, `PUT` | `doctype`, `name` | Cancel a submitted document. |
| `delete` | `DELETE`, `POST` | `doctype`, `name` | Delete a document. |
| `bulk_update` | `POST`, `PUT` | `docs` | Update many documents. Each item has `doctype`, `docname` and fields. |
| `has_permission` | any | `doctype`, `docname`, `perm_type` (`read`) | Returns `{"has_permission": bool}`. |
| `get_doc_permissions` | any | `doctype`, `docname` | Returns `{"permissions": {...}}`. Empty when the user cannot read the document. |
| `get_password` | any | `doctype`, `name`, `fieldname` | Returns a decrypted Password field. System Manager only. |
| `get_time_zone` | any (guest) | none | Returns the system time zone. |
| `attach_file` | `POST`, `PUT` | `filename`, `filedata`, `doctype`, `docname`, `folder`, `decode_base64`, `is_private`, `docfield` | Attach file content to a document. |
| `is_document_amended` | any | `doctype`, `docname` | True when an amended copy exists. |
| `validate_link_and_fetch` | `GET`, `POST` | `doctype`, `docname`, `fields_to_fetch`, `query`, `filters` | Check a Link value and fetch fields from the linked record. |

```bash
curl -b cookies.txt "http://frappe.localhost:8006/api/v2/method/frappe.client.get_value?doctype=User&filters=%22Administrator%22&fieldname=%5B%22full_name%22%5D"
```

```json
{"data": {"full_name": "Administrator"}}
```

## Other core methods

The desk calls these. They are useful for clients too.

| Method | HTTP methods | Description |
|---|---|---|
| `frappe.desk.reportview.get` | any | List view data. Args like `get_list` (`doctype`, `fields`, `filters`, `order_by`, `start`, `page_length`). |
| `frappe.desk.reportview.get_count` | any | Count for list view. |
| `frappe.desk.reportview.export_query` | any | Export Report Builder data as CSV or Excel. |
| `frappe.desk.reportview.delete_items` | `POST`, `DELETE` | Delete selected records. |
| `frappe.desk.form.load.getdoc` | any | Load a document with docinfo for the form. |
| `frappe.desk.form.load.getdoctype` | any | Load DocType meta and scripts for the form. |
| `frappe.desk.form.save.savedocs` | `POST`, `PUT` | Save, submit or update a document from the form. Args: `doc`, `action`. |
| `frappe.desk.form.save.cancel` | `POST`, `PUT` | Cancel from the form. |
| `frappe.desk.search.search_link` | any | Results for a Link field dropdown. |
| `frappe.desk.query_report.run` | any | Run a Query or Script Report. Args: `report_name`, `filters`. |
| `frappe.rename_doc` | `POST`, `PUT` | Rename and update links. |

`frappe` has about 720 whitelisted methods in this version. To list them, call `/api/v2/discovery/method` as a desk user.
App developers make new ones with `@frappe.whitelist()`. See [Whitelisted Methods](../../developer-guide/api/whitelisted-methods.md).

## Pitfalls

- A method without `methods=` in `@frappe.whitelist` accepts `GET`, `POST`, `PUT` and `DELETE`. Write methods should set `methods=["POST"]`.
- v1 `PUT /api/resource/...` and v2 `PATCH` call `doc.save()`. They do not submit. Use the `submit` method or a document method.
- v2 list returns 20 records by default. Check `has_next_page` and raise `start`.
- `frappe.client.get_list` and `/api/resource` return 20 rows by default. `limit_page_length=0` removes the limit. Use it with care on large tables.
- `run_doc_method` in v2 loads the saved copy of the document by `name`. Send a `name` (for example `new-todo-1`) also for an unsaved document. Without it, the call fails with "not found".

## Related

- [REST API](../../developer-guide/api/rest-api.md)
- [Authentication](../../developer-guide/security/authentication.md)
- [Whitelisted Methods](../../developer-guide/api/whitelisted-methods.md)
- [MCP and Discovery](../../developer-guide/api/mcp-and-discovery.md)
- [Files](../../developer-guide/data-model/files.md)
- [Site Config Reference](../configuration/site-config.md)
