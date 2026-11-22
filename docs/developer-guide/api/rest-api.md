---
title: REST API
description: Read and change documents and call whitelisted methods over HTTP with /api/v1 (resource, method) and /api/v2 (document, doctype, method), including list parameters, responses and errors.
version: v15
sources:
  - frappe/api/__init__.py
  - frappe/api/v1.py
  - frappe/api/v2.py
  - frappe/handler.py
  - frappe/client.py
  - frappe/app.py
  - frappe/exceptions.py
  - frappe/utils/response.py
  - frappe/desk/reportview.py
---

# REST API

Every Frappe site has a REST API at `/api`. It gives access to every DocType and every whitelisted method, with the permissions of the authenticated user. There are two versions:

- `/api/v1/...` (also served at `/api/...`). Routes `resource` and `method`. Results are in `data` or `message`.
- `/api/v2/...`. Routes `document`, `doctype` and `method`. Results are always in `data`. Errors are a list in `errors`.

All requests on this page were sent with curl to `http://frappe.localhost:8001` (a v15 site) as `user1@example.com` with token authentication. The responses are trimmed. To get a token, see [Authentication](../security/authentication.md).

```bash
S=http://frappe.localhost:8001
AUTH="Authorization: token <api_key>:<api_secret>"
```

## Routes

| Action | v1 | v2 |
|---|---|---|
| List documents | `GET /api/resource/<doctype>` | `GET /api/v2/document/<doctype>` |
| Create | `POST /api/resource/<doctype>` | `POST /api/v2/document/<doctype>` |
| Read | `GET /api/resource/<doctype>/<name>` | `GET /api/v2/document/<doctype>/<name>` |
| Update | `PUT /api/resource/<doctype>/<name>` | `PUT` or `PATCH /api/v2/document/<doctype>/<name>` |
| Delete | `DELETE /api/resource/<doctype>/<name>` | `DELETE /api/v2/document/<doctype>/<name>` |
| Call a document method | `POST /api/resource/<doctype>/<name>` with `run_method` | `GET` or `POST /api/v2/document/<doctype>/<name>/method/<method>` |
| Copy a document | | `GET /api/v2/document/<doctype>/<name>/copy` |
| Count | | `GET /api/v2/doctype/<doctype>/count` |
| DocType metadata | | `GET /api/v2/doctype/<doctype>/meta` |
| Call a whitelisted method | `/api/method/<dotted.path>` | `/api/v2/method/<dotted.path>` |
| Call a function in a DocType module | | `/api/v2/method/<doctype>/<function>` |

URL-encode spaces in DocType names (`Sample%20Item`). A trailing slash after the document name is accepted. v15 has no bulk routes. The full list is in [REST endpoints](../../reference/api/rest-endpoints.md).

## List documents

### v1

```bash
curl -s -G "$S/api/resource/Sample%20Item" -H "$AUTH" \
  --data-urlencode 'fields=["name","rate"]' \
  --data-urlencode 'filters=[["category","=","Hardware"],["rate","<",100]]' \
  --data-urlencode 'order_by=rate asc'
```

```json
{"data": [{"name": "Keyboard", "rate": 45.0}]}
```

`GET /api/resource/<doctype>` calls `frappe.client.get_list` with the query parameters:

| Parameter | Default | Description |
|---|---|---|
| `fields` | `["name"]` | JSON list. Same syntax as `frappe.get_list`. |
| `filters` | none | JSON dict or list. See [Filters](../database/database.md#filters). |
| `or_filters` | none | JSON dict or list, joined with `OR`. |
| `order_by` | DocType sort settings | For example `rate desc`. |
| `group_by` | none | Field name. |
| `limit_start` | `0` | Offset. |
| `limit_page_length` or `limit` | `20` | Page size. `0` returns all rows. |
| `as_dict` | `1` | `0` returns lists instead of objects. |
| `parent` | none | Parent DocType, required for a child DocType. |
| `expand` | none | JSON list of Link fields to replace with the linked document. |
| `debug` | `0` | Print the query for debugging. |

```bash
curl -s -G "$S/api/resource/Sample%20Item" -H "$AUTH" \
  --data-urlencode 'fields=["name","rate"]' --data-urlencode 'as_dict=0' --data-urlencode 'limit=2'
```

```json
{"data": [["Antivirus", 60.0], ["Installation", 80.0]]}
```

### v2

```bash
curl -s -G "$S/api/v2/document/Sample%20Item" -H "$AUTH" \
  --data-urlencode 'fields=["name","rate"]' \
  --data-urlencode 'filters={"category":"Hardware"}' \
  --data-urlencode 'order_by=rate desc' \
  --data-urlencode 'limit=2'
```

```json
{"data": [{"name": "Laptop", "rate": 900.0}, {"name": "Monitor", "rate": 250.0}]}
```

Next page, with `start=2`:

```json
{"data": [{"name": "Keyboard", "rate": 45.0}]}
```

v2 sets `limit_page_length` from `limit` (default `20`) and `limit_start` from `start` (default `0`). Then it calls `frappe.client.get_list`, so all v1 list parameters also work. The response has no `has_next_page` key. Ask for `limit + 1` rows to know if another page exists.

### Count

```bash
curl -s -G "$S/api/v2/doctype/Sample%20Item/count" -H "$AUTH" --data-urlencode 'filters={"category":"Hardware"}'
```

```json
{"data": 3}
```

The count route calls `frappe.desk.reportview.get_count`.

## Create a document

```bash
curl -s -X POST "$S/api/v2/document/Sample%20Item" -H "$AUTH" -H "Content-Type: application/json" \
  -d '{"item_name": "Sample Mouse", "category": "Hardware", "rate": 25}'
```

```json
{"data": {"name": "Sample Mouse", "owner": "user1@example.com", "creation": "2026-11-22 09:37:01.175667",
  "modified": "2026-11-22 09:37:01.175667", "modified_by": "user1@example.com", "docstatus": 0, "idx": 0,
  "item_name": "Sample Mouse", "category": "Hardware", "rate": 25.0, "disabled": 0, "description": null,
  "doctype": "Sample Item"}}
```

The body is the document as JSON. Include child rows as lists, for example `"items": [{"item": "Laptop", "qty": 1}]`. The document is inserted with `doc.insert()`, so validations, controller hooks and permission checks run. v1 `POST /api/resource/<doctype>` does the same.

## Read a document

```bash
curl -s "$S/api/v2/document/Sample%20Item/Sample%20Mouse" -H "$AUTH"
```

```json
{"data": {"name": "Sample Mouse", "owner": "user1@example.com", "docstatus": 0,
  "item_name": "Sample Mouse", "category": "Hardware", "rate": 25.0, "disabled": 0, "description": null,
  "doctype": "Sample Item"}}
```

The response has all fields and child tables. v15 ignores a `fields` parameter on this route. Both versions check `read` and remove fields that the user cannot read (`doc.apply_fieldlevel_read_permissions()`). v1 accepts `expand_links=1` to replace Link values with the linked documents.

## Update a document

```bash
curl -s -X PATCH "$S/api/v2/document/Sample%20Item/Sample%20Mouse" -H "$AUTH" -H "Content-Type: application/json" \
  -d '{"rate": 30}'
```

```json
{"data": {"name": "Sample Mouse", "modified": "2026-11-22 09:37:01.220335", "rate": 30.0, "doctype": "Sample Item"}}
```

Send only the fields to change. Frappe loads the document with `for_update=True`, applies the values and calls `doc.save()`, which checks `write`. A list for a table field replaces all child rows. v1 uses `PUT /api/resource/<doctype>/<name>` with the same body.

## Delete a document

```bash
curl -s -X DELETE "$S/api/v2/document/Sample%20Item/Sample%20Mouse" -H "$AUTH" -w '\n%{http_code}\n'
```

```text
{"data":"ok"}
202
```

## Call methods

```bash
curl -s "$S/api/method/frappe.auth.get_logged_user" -H "$AUTH"
# {"message":"user1@example.com"}
curl -s "$S/api/v2/method/frappe.auth.get_logged_user" -H "$AUTH"
# {"data":"user1@example.com"}
curl -s "$S/api/v2/method/User/get_current_user_roles" -H "$AUTH"
# {"data":["Sample Role","All","Guest","Desk User"]}
```

Send arguments as query parameters for `GET`, and as a JSON or form body for `POST`. Nested values (lists, dicts) need a JSON body or JSON-encoded parameters. See [Whitelisted methods](whitelisted-methods.md).

`frappe.client` has general methods, for example `frappe.client.get_list`, `frappe.client.get_value`, `frappe.client.insert`, `frappe.client.set_value`, `frappe.client.submit`, `frappe.client.cancel` and `frappe.client.delete`. They apply the permissions of the user.

```bash
curl -s -X POST "$S/api/v2/method/frappe.client.insert" -H "$AUTH" -H "Content-Type: application/json" \
  -d '{"doc": {"doctype": "Sample Item", "item_name": "Sample Mouse C", "category": "Hardware"}}'
```

## Responses

| Key | Where | Content |
|---|---|---|
| `data` | v1 resource, all v2 | Result |
| `message` | v1 method | Return value of the method |
| `docs` | v2 document methods | Changed documents |
| `_server_messages` | v1 | JSON string with messages from `frappe.msgprint` |
| `messages` | v2 | List of messages from `frappe.msgprint` |

## Errors

The HTTP status comes from `http_status_code` of the exception class (`frappe/exceptions.py`):

| Status | Exception |
|---|---|
| 400 | `frappe.CSRFTokenError` |
| 401 | `frappe.AuthenticationError`, `frappe.SessionExpired` |
| 403 | `frappe.PermissionError` |
| 404 | `frappe.DoesNotExistError` |
| 409 | `frappe.DuplicateEntryError` (subclass of `frappe.NameError`) |
| 417 | `frappe.ValidationError` and subclasses (for example `frappe.MandatoryError`, `frappe.TimestampMismatchError`), `frappe.FrappeTypeError` |
| 429 | `frappe.RateLimitExceededError`, `frappe.TooManyRequestsError` |
| 500 | Other exceptions |

v1 error:

```json
{"exception": "frappe.exceptions.PermissionError",
 "exc_type": "PermissionError",
 "_server_messages": "[\"{\\\"message\\\": \\\"User <strong>user1@example.com</strong> does not have doctype access via role permission for document <strong>Sample Order</strong>\\\", \\\"title\\\": \\\"Message\\\"}\"]",
 "_error_message": "No permission for Sample Order"}
```

v2 error:

```json
{"errors": [{"type": "PermissionError",
  "message": "User <strong>user1@example.com</strong> does not have doctype access via role permission for document <strong>Sample Order</strong>",
  "title": "Message"}]}
```

The traceback (`exc` in v1, `exception` in v2) is sent only when **Show Full Error and Allow Reporting of Issues to the Developer** (`allow_error_traceback`) is on in System Settings and the user is a system user (`is_traceback_allowed` in `frappe/utils/response.py`). It is not sent for HTTP 404 unless the `logging` site config is set.

## Transactions

A `POST`, `PUT`, `PATCH` or `DELETE` request commits on success. A `GET` request is always rolled back. Any error rolls back the request. See [Transactions](../database/database.md#transactions).

## Pitfalls

- The list default is 20 rows. Pass `limit=0` (v1) or a larger `limit` to get more.
- JSON parameters in a `GET` URL must be URL-encoded. Use `curl -G --data-urlencode`.
- v1 sends results in `data` for resources and in `message` for methods. v2 always uses `data`.
- v15 v2 list responses have no `has_next_page`, and v2 read ignores `fields`.
- The document method routes check only `read` (GET) or `write` (POST). They do not check the `methods` argument of `@frappe.whitelist`.

## Related

- [Authentication](../security/authentication.md)
- [Whitelisted methods](whitelisted-methods.md)
- [Database API](../database/database.md)
- [Permissions](../security/permissions.md)
- [Files](../data-model/files.md)
- [REST endpoints reference](../../reference/api/rest-endpoints.md)
