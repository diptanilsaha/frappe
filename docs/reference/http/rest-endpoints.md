---
title: REST Endpoints Reference
description: Every route under /api in Frappe v15 (v1 and v2) and the most used whitelisted methods, with HTTP methods and permission checks.
version: v15
sources:
  - frappe/api/__init__.py
  - frappe/api/v1.py
  - frappe/api/v2.py
  - frappe/handler.py
  - frappe/auth.py
  - frappe/client.py
  - frappe/__init__.py
---

# REST Endpoints Reference

This page lists every route that Frappe v15 serves under `/api`, and the core whitelisted methods that clients call most.
For how to authenticate, filter and page, see [REST API](../../developer-guide/api/rest-api.md) and [Authentication](../../developer-guide/security/authentication.md).

## Route maps

`frappe/api/__init__.py` mounts three route sets:

| Prefix | Rules from | Response key |
|---|---|---|
| `/api/` | `frappe/api/v1.py` | `message` for methods, `data` for resources |
| `/api/v1/` | `frappe/api/v1.py` (same rules) | same as `/api/` |
| `/api/v2/` | `frappe/api/v2.py` | `data` for all routes. Errors are in `errors` |

Trailing slashes are optional. An unknown route raises `DoesNotExistError` (HTTP 404).
All routes run as the session user. Permission checks are those of the called function.

## API v1

| Route | HTTP methods | Does | Permission check |
|---|---|---|---|
| `/api/method/<dotted.path>` | any allowed by the method | Calls a whitelisted function. Also runs API Server Scripts with that name. Applies `override_whitelisted_methods`. | `@frappe.whitelist` rules (guest, HTTP methods) |
| `/api/resource/<doctype>` | `GET` | List records through `frappe.client.get_list`. | `get_list` permission rules |
| `/api/resource/<doctype>` | `POST` | Insert a document from the JSON body (or `data`). | `create` |
| `/api/resource/<doctype>/<name>` | `GET` | Read one document. `expand_links=1` adds linked documents. | `read` |
| `/api/resource/<doctype>/<name>` | `PUT` | Update fields and save. | `write` |
| `/api/resource/<doctype>/<name>` | `DELETE` | Delete. Returns HTTP 202 and `"ok"`. | `delete` |
| `/api/resource/<doctype>/<name>` | `POST` | Call the whitelisted controller method in `run_method`. | `write` |

List parameters for `GET /api/resource/<doctype>`: `fields`, `filters`, `or_filters`, `order_by`, `group_by`, `limit_start`, `limit_page_length` (or `limit`, default 20), `as_dict`, `expand`, `debug`.
`GET /api/resource/<doctype>/<name>?run_method=<method>` calls a document method with `read` permission (kept for old clients).

```bash
curl -b cookies.txt "http://frappe.localhost:8001/api/resource/ToDo?fields=%5B%22name%22,%22status%22%5D&limit=2"
```

```json
{"data": [{"name": "hqi0l08vpa", "status": "Open"}, {"name": "hqif81b6a2", "status": "Open"}]}
```

## API v2

| Route | HTTP methods | Does | Permission check |
|---|---|---|---|
| `/api/v2/method/login` | any | Does nothing by itself. See the warning below. | none |
| `/api/v2/method/logout` | any | Log out the session. | none |
| `/api/v2/method/ping` | any | Returns `"pong"`. | none |
| `/api/v2/method/upload_file` | any | Upload a file. Same as `frappe.handler.upload_file`. | see [Files](../../developer-guide/data-model/files.md) |
| `/api/v2/method/run_doc_method` | `GET`, `POST` | Run a whitelisted method on a `document` dict. Args: `method`, `document`, `kwargs`. | `read` (GET), `write` (POST) |
| `/api/v2/method/<dotted.path>` | any allowed by the method | Calls a whitelisted function or API Server Script. Applies `override_whitelisted_methods`. | `@frappe.whitelist` rules |
| `/api/v2/method/<doctype>/<method>` | any allowed by the method | Calls a whitelisted module function in the DocType's controller file. | `@frappe.whitelist` rules |
| `/api/v2/document/<doctype>` | `GET` | List records. Returns `data`. | `get_list` permission rules |
| `/api/v2/document/<doctype>` | `POST` | Insert a document from the JSON body. | `create` |
| `/api/v2/document/<doctype>/<name>` | `GET` | Read one document. | `read` |
| `/api/v2/document/<doctype>/<name>` | `PATCH`, `PUT` | Update fields and save. | `write` |
| `/api/v2/document/<doctype>/<name>` | `DELETE` | Delete. Returns HTTP 202 and `"ok"`. | `delete` |
| `/api/v2/document/<doctype>/<name>/copy` | `GET` | A copy of the document without `name` and other private properties, ready to insert. The route keeps `no_copy` fields. | `read` |
| `/api/v2/document/<doctype>/<name>/method/<method>` | `GET`, `POST` | Call a whitelisted controller method. The saved document goes in `docs`. | `read` (GET), `write` (POST) |
| `/api/v2/doctype/<doctype>/meta` | `GET` | DocType meta. | logged-in user (`frappe.only_for("All")`) |
| `/api/v2/doctype/<doctype>/count` | `GET` | Count with optional `filters`. | `frappe.desk.reportview.get_count` rules |

List parameters for `GET /api/v2/document/<doctype>`: `fields`, `filters`, `order_by`, `group_by`, `start` (default 0), `limit` (default 20), `as_dict`, `debug`.
The response has only `data`. To get the next page, raise `start` until a page has fewer rows than `limit`.

For `run_doc_method` with a document that is not saved, add `"__islocal": 1` to the `document` dict. Without it, Frappe tries to load the document from the database.

!!! warning "Login in v2"
    In v15, `LoginManager` logs in only when the path is `/api/method/login` or the form has `cmd=login`. A plain `POST /api/v2/method/login` returns `{}` and leaves the session as Guest. Use `/api/method/login`, or add `"cmd": "login"` to the v2 body.

```bash
curl -b cookies.txt -X POST http://frappe.localhost:8001/api/v2/document/ToDo \
  -H 'Content-Type: application/json' -d '{"description": "Sample REST todo"}'
curl -b cookies.txt -X PATCH http://frappe.localhost:8001/api/v2/document/ToDo/3u0lp3tjma \
  -H 'Content-Type: application/json' -d '{"status": "Closed"}'
curl -b cookies.txt "http://frappe.localhost:8001/api/v2/doctype/ToDo/count"
```

```json
{"data": 5}
```

!!! note "Version"
    v15 has no API discovery routes, no MCP route, and no v2 bulk routes. The `QUERY` HTTP method is not supported.

## Login, logout and files

These work on `/api/method/...` (v1). In this table, "any" means `GET`, `POST`, `PUT` and `DELETE`, the default of `@frappe.whitelist`. v1 rejects other methods, for example `PATCH`.

| Method path | HTTP methods | Guest | Description |
|---|---|---|---|
| `login` | `POST` | yes | Log in. Args: `usr`, `pwd`. Returns `message`, `home_page`, `full_name` and sets the `sid` cookie. `message` is `"No App"` for website users. |
| `logout` (`frappe.handler.logout`) | any | yes | Log out the current session. |
| `frappe.handler.web_logout` | any | yes | Log out and show a "Logged Out" web page. |
| `frappe.auth.get_logged_user` | any | no | Returns the user name of the session. |
| `frappe.ping` | any | yes | Returns `"pong"`. |
| `upload_file` (`frappe.handler.upload_file`) | any | yes | Upload a file as multipart form data (`file`, `doctype`, `docname`, `fieldname`, `is_private`, `folder`, `file_url`, `file_name`, `optimize`). Guests can upload only when System Settings `allow_guests_to_upload_files` is on. |
| `frappe.handler.download_file` | any | yes | Download a file by `file_url`, with session or token auth. |
| `frappe.core.doctype.user.user.reset_password` | `POST` | yes | Send a password reset link. Arg: `user`. |
| `frappe.core.doctype.user.user.update_password` | `POST` | yes | Set a new password with `key` or `old_password`. Args: `new_password`, `logout_all_sessions`. |
| `frappe.core.doctype.user.user.sign_up` | any | yes | Website sign up. Args: `email`, `full_name`, `redirect_to`. |

Short names such as `logout` and `upload_file` resolve to `frappe.handler` and print a deprecation warning. Use the full dotted path in new code.

```bash
curl -c cookies.txt -X POST http://frappe.localhost:8001/api/method/login \
  -H 'Content-Type: application/json' -d '{"usr": "Administrator", "pwd": "password"}'
```

```json
{"message": "Logged In", "home_page": "/app", "full_name": "Administrator"}
```

## frappe.client methods

Call them as `/api/method/frappe.client.<name>` or `/api/v2/method/frappe.client.<name>`. Defined in `frappe/client.py`.
"any" means `GET`, `POST`, `PUT` and `DELETE`, the default of `@frappe.whitelist`.

| Method | HTTP methods | Arguments | Description |
|---|---|---|---|
| `get_list` | any | `doctype`, `fields`, `filters`, `or_filters`, `group_by`, `order_by`, `limit_start`, `limit_page_length` (20), `parent`, `as_dict`, `expand`, `debug` | List records with permission checks. |
| `get_count` | any | `doctype`, `filters`, `cache`, `debug` | Count records. |
| `get` | any | `doctype`, `name` or `filters`, `parent` | Get one document as a dict. |
| `get_value` | any | `doctype`, `fieldname` (str or list), `filters`, `as_dict`, `parent`, `debug` | Get field values of one record. |
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
| `get_doc_permissions` | any | `doctype`, `docname` | Returns `{"permissions": {...}}`. |
| `get_password` | any | `doctype`, `name`, `fieldname` | Returns a decrypted Password field. System Manager only. |
| `get_time_zone` | any (guest) | none | Returns the system time zone. |
| `attach_file` | `POST`, `PUT` | `filename`, `filedata`, `doctype`, `docname`, `folder`, `decode_base64`, `is_private`, `docfield` | Attach file content to a document. |
| `is_document_amended` | any | `doctype`, `docname` | True when an amended copy exists. |
| `validate_link` | any | `doctype`, `docname`, `fields` | Check a Link value and fetch `fields` from the linked record. Needs `select` or `read` permission. |
| `get_js` | any | `items` | Deprecated. Returns the content of JS files under `assets/`. |

```bash
curl -b cookies.txt "http://frappe.localhost:8001/api/v2/method/frappe.client.get_value?doctype=User&filters=%22Administrator%22&fieldname=%5B%22full_name%22%5D"
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

`frappe` has about 530 whitelisted functions and controller methods in v15.
App developers make new ones with `@frappe.whitelist()`. See [Whitelisted Methods](../../developer-guide/api/whitelisted-methods.md).

## Pitfalls

- A method without `methods=` in `@frappe.whitelist` accepts `GET`, `POST`, `PUT` and `DELETE`. Write methods should set `methods=["POST"]`.
- v1 `PUT /api/resource/...` and v2 `PATCH` call `doc.save()`. They do not submit. Use the `submit` method or a document method.
- `frappe.client.get_list`, `/api/resource` and `/api/v2/document` return 20 rows by default. `limit_page_length=0` (v1) or `limit=0` (v2) removes the limit. Use it with care on large tables.
- System Settings has a `log_api_requests` field, but no code in v15 reads it. No API Request Log is written.

## Related

- [REST API](../../developer-guide/api/rest-api.md)
- [Authentication](../../developer-guide/security/authentication.md)
- [Whitelisted Methods](../../developer-guide/api/whitelisted-methods.md)
- [Files](../../developer-guide/data-model/files.md)
- [Site Config Reference](../configuration/site-config.md)
