---
title: MCP and API Discovery
description: Connect AI agents to a site through the built-in MCP endpoint at /api/v2/mcp, and describe whitelisted methods with the /api/v2/discovery endpoints.
version: develop
sources:
  - frappe/mcp/__init__.py
  - frappe/mcp/transport.py
  - frappe/mcp/server.py
  - frappe/mcp/protocol.py
  - frappe/mcp/tools/__init__.py
  - frappe/mcp/tools/discover.py
  - frappe/mcp/tools/documents.py
  - frappe/mcp/tools/methods.py
  - frappe/api/discovery.py
  - frappe/api/v2.py
  - frappe/core/doctype/system_settings/system_settings.json
  - frappe/hooks.py
  - frappe/app.py
---

# MCP and API Discovery

Each site has a built-in Model Context Protocol (MCP) server at `/api/v2/mcp`. An AI agent (an MCP client) connects as a desk user and gets tools to search DocTypes, read and write documents, and call whitelisted methods. Every tool goes through the same permission checks as the REST API. The API discovery endpoints at `/api/v2/discovery` describe the whitelisted methods of the site, with their arguments and docstrings. MCP uses them, and you can call them from any API client.

## Enable MCP

MCP is on by default. The settings are in **System Settings > AI Agents > MCP Settings**:

| Field | Default | Description |
|---|---|---|
| `enable_mcp` | 1 | Off: `/api/v2/mcp` returns HTTP 404. |
| `server_name` | host name | Name sent to clients in `serverInfo`. The default is the host name with other characters changed to `_`, such as `frappe_localhost_8000`. |
| `agent_instructions` | empty | Text added to the server instructions that agents get when they connect. Tell agents what the site is for. |

## Connect a client

The endpoint uses the Streamable HTTP transport with plain JSON responses. It keeps no protocol session.

- URL: `https://<site>/api/v2/mcp`
- Method: `POST` only. Other methods get HTTP 405.
- Headers: `Content-Type: application/json`, and an `Accept` that allows `application/json`.
- Authentication: any method the REST API accepts: OAuth 2 bearer token, API key and secret (`Authorization: token <key>:<secret>`), or a session cookie. See [Authentication](../security/authentication.md).
- The user must have the `Desk User` role. Guests get HTTP 401. Users without the role get HTTP 403.
- A request with an `Origin` header from another host must match `allow_cors` in site config.

Without credentials, the response is HTTP 401 with `WWW-Authenticate: Bearer resource_metadata="<site>/.well-known/oauth-protected-resource"`. MCP clients use it to start an OAuth flow. The metadata endpoint is served when **OAuth Settings > Show Protected Resource Metadata** is on. See [Integrations](../../user-guide/website-and-integrations/integrations.md).

Protocol versions: `2025-11-25` (with `initialize`) and `2026-07-28` (with `server/discover` and the `MCP-Protocol-Version`, `Mcp-Method` and `Mcp-Name` headers). A request without `MCP-Protocol-Version` uses `2025-11-25`.

### Example with curl

```bash
curl -X POST "http://frappe.localhost:8000/api/v2/mcp" \
  -H "Authorization: token <api_key>:<api_secret>" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{"jsonrpc": "2.0", "id": 1, "method": "initialize",
       "params": {"protocolVersion": "2025-11-25", "capabilities": {},
                  "clientInfo": {"name": "sample-client", "version": "1.0"}}}'
```

```json
{"jsonrpc": "2.0", "id": 1, "result": {
  "protocolVersion": "2025-11-25",
  "capabilities": {"tools": {"listChanged": false}},
  "serverInfo": {"name": "frappe_localhost_8000", "version": "17.0.0-dev"},
  "instructions": "This is a Frappe Framework site at frappe.localhost:8000.\nYou are connected as Administrator (Administrator).\n..."}}
```

Call a tool:

```bash
curl -X POST "http://frappe.localhost:8000/api/v2/mcp" \
  -H "Authorization: token <api_key>:<api_secret>" \
  -H "Content-Type: application/json" -H "Accept: application/json" \
  -d '{"jsonrpc": "2.0", "id": 2, "method": "tools/call",
       "params": {"name": "list_documents",
                  "arguments": {"doctype": "Sample Item", "fields": ["name", "rate"],
                                "filters": {"category": "Hardware"}, "limit": 2}}}'
```

The result has `content` (JSON as text), `structuredContent` and `isError`. A tool that fails returns `isError: true` with the error text, for example `"Sample Item nope not found"`. Protocol errors (bad JSON, unknown method) return a JSON-RPC `error`.

## Tools

| Tool | Arguments | Annotation | Description |
|---|---|---|---|
| `search_doctypes` | `query`, `limit` | read-only | Find DocType names that contain the query. |
| `get_doctype_schema` | `doctype` | read-only | Fields and permissions of a DocType. |
| `search_methods` | `query` | read-only | Search whitelisted methods and document methods. |
| `get_method_schema` | `method`, `doctype` | read-only | Arguments and docstring of a method. |
| `list_document_methods` | `doctype` | read-only | Whitelisted document methods of a DocType. |
| `list_documents` | `doctype`, `filters`, `fields`, `order_by`, `start`, `limit` | read-only | Like `frappe.get_list`. |
| `get_document` | `doctype`, `name`, `fields` | read-only | One document with child tables. |
| `count_documents` | `doctype`, `filters` | read-only | Count matching documents. |
| `create_document` | `doctype`, `data`, `fields` | not destructive | Insert a document. |
| `update_document` | `doctype`, `name`, `data`, `fields` | destructive | Change fields of a document. |
| `delete_document` | `doctype`, `name` | destructive | Delete a document. |
| `call_method` | `method`, `args` | destructive | Call a whitelisted method or an API Server Script. |
| `call_document_method` | `doctype`, `name`, `method`, `args` | destructive | Call a whitelisted controller method, such as `submit`. |

Tool arguments take the same values as the matching [REST API](rest-api.md) v2 endpoints. Arguments are validated strictly: unknown arguments and wrong types fail. If a method returns a file, the tool describes it. Images up to 5 MB are sent to the model as image content.

Changes made through MCP with a token or OAuth are recorded in `Version` with the agent name. The agent name is the OAuth client name, else the `clientInfo.name` that the client sent, else `MCP`.

## API discovery endpoints

All endpoints use `GET` and need the `Desk User` role.

| Route | Returns |
|---|---|
| `/api/v2/discovery` | Counts of methods and document methods, and the links below. |
| `/api/v2/discovery/search?q=<words>` | Up to 50 methods whose path or docstring has all the words. |
| `/api/v2/discovery/method` | Index of all whitelisted methods. |
| `/api/v2/discovery/method/<dotted.path>` | One method: HTTP methods, `allow_guest`, parameters with types and defaults, endpoint, docstring and source. |
| `/api/v2/discovery/doctype/<doctype>` | Whitelisted controller methods of a DocType. |
| `/api/v2/discovery/doctype/<doctype>/method/<method>` | One controller method: endpoint, HTTP methods, required permission per HTTP method, parameters, docstring. |

```bash
curl "http://frappe.localhost:8000/api/v2/discovery/method/frappe.client.get_value" \
  -H "Authorization: token <api_key>:<api_secret>"
```

```json
{"data": {"type": "method", "kind": "rpc", "path": "frappe.client.get_value", "name": "get_value",
  "http_methods": ["GET", "POST", "PUT", "DELETE", "QUERY"], "allow_guest": false,
  "params": [{"name": "doctype", "required": true, "type": "str"},
             {"name": "fieldname", "required": true, "type": "str | list[str] | dict[str, Any]"}, "..."],
  "endpoint": "/api/v2/method/frappe.client.get_value",
  "docstring": "Return a value from a document. ...",
  "source": "@frappe.whitelist()\ndef get_value(..."}}
```

### The discovery cache

Frappe builds the method index in the background and keeps it in Redis for 24 hours (1 hour in developer mode). The job runs after each `bench migrate` (`after_migrate` hook) with the job ID `api-v2-discovery-build`. If the cache is empty, the endpoints start the job and return HTTP 503 with "API discovery cache is being generated. Please try again in 10 seconds."

The index is built by reading the Python files of each app for functions with `@frappe.whitelist`. API Server Scripts are listed for users who can read them.

### Make your methods discoverable

- Add type hints to every argument. Discovery shows them as `type`.
- Write a docstring. The first line is the `description` in search results. The search also reads the docstring.
- Source code is returned only when the app sets `expose_discovery_source = True` in `hooks.py`. Frappe sets it. Leave it out of a closed-source app.

```python
# sample_app/hooks.py
expose_discovery_source = True
```

```python
# sample_app/api.py
import frappe


@frappe.whitelist()
def get_open_orders(customer: str, limit: int = 20) -> list[dict]:
	"""Return open Sample Orders of a customer, newest first."""
	return frappe.get_list(
		"Sample Order",
		filters={"customer_name": customer, "status": "Open"},
		fields=["name", "order_date", "total"],
		order_by="order_date desc",
		limit=limit,
	)
```

## Pitfalls

- An agent can do everything its user can do. Give agents their own user with only the roles they need.
- `call_method` reaches every whitelisted method the user may call. Check permissions in your methods. See [Whitelisted Methods](whitelisted-methods.md).
- A new method does not show in discovery until the cache is rebuilt. Run `bench migrate`, or wait for the cache to expire.

## Related

- [REST API](rest-api.md)
- [Authentication](../security/authentication.md)
- [Whitelisted Methods](whitelisted-methods.md)
- [Flow (AI agents)](../../user-guide/ai-and-app-builder/flow.md)
- [REST endpoints reference](../../reference/http/rest-endpoints.md)
