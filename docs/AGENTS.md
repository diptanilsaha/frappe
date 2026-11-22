---
title: For AI Agents
description: Rules, verified facts and pitfalls for AI agents that build or change Frappe apps on version 16.
version: v16
sources:
  - frappe/model/document.py
  - frappe/__init__.py
  - frappe/permissions.py
  - frappe/app.py
  - frappe/database/query.py
---

# For AI Agents

Read this page before you write or change code for a Frappe app on version 16. Every fact here was checked in the `version-16-hotfix` branch. For an index of all pages with one-line summaries, read [llms.txt](llms.txt).

## Know the version first

1. Run `bench version` or read `frappe/__init__.py` `__version__`. These docs are for `16.x`.
2. Do not use an API from another version without checking it. `frappe.read_doc`, `frappe.enqueue_task`, `bench enable-app`, v2 bulk endpoints and storage v2 do not exist in version 16. `extend_doctype_class`, Flow, Studio, MCP and the `/desk` route are new in version 16.

## Where to look

| Question | Page |
|---|---|
| Which hooks exist and what shape they take | [Hooks reference](reference/configuration/hooks.md) |
| Run order of controller methods and `doc_events` | [Controllers](developer-guide/data-model/controllers.md) |
| Signatures of `frappe.*`, `frappe.db.*`, `frappe.utils.*` | [Python API](reference/python/frappe.md), [Database API](reference/python/database.md), [Utils](reference/python/utils.md) |
| Client-side `frappe.*` and `frm` API | [JavaScript API](reference/javascript/frappe.md), [Form Scripts](developer-guide/desk-ui/form-scripts.md) |
| REST routes | [REST Endpoints](reference/http/rest-endpoints.md) |
| CLI commands and flags | [CLI](reference/configuration/cli.md) |
| Config keys | [Site Config](reference/configuration/site-config.md) |
| Field types and DB columns | [Field Types](reference/data-model/field-types.md) |

## Rules

- Use `frappe.session.user` for the current user. `frappe.user` is `None`.
- Check permissions in every whitelisted method. Use `doc.check_permission("read")` or `frappe.has_permission`. `frappe.get_doc` does not check read permission.
- `frappe.get_all` ignores permissions. `frappe.get_list` applies them. Neither has a default row limit in Python. Pass `limit`. In JavaScript, `frappe.db.get_list` defaults to 20 rows.
- `frappe.db.set_value`, `frappe.db.sql` and `frappe.qb` skip validations, controller methods and permissions.
- `frappe.qb.get_query` ignores permissions by default (`ignore_permissions=True`).
- Write SQL functions in `fields` in the dict form. A string such as `"count(name) as c"` raises `ValidationError`.
- Do not call `frappe.db.commit()` in a request handler or a controller method. The request commits when it succeeds, but only for `POST`, `PUT`, `PATCH` and `DELETE` (or when `frappe.local.flags.commit` is set). Writes in a `GET` request are rolled back. Inside `doc_events` handlers transaction control is disabled.
- A `has_permission` hook can only deny. Any falsy return value, `None` included, denies access.
- Add type annotations to whitelisted methods. Request arguments are validated against them.
- `@frappe.whitelist()` takes `allow_guest`, `xss_safe` and `methods`. Without `methods`, the method accepts `GET`, `POST`, `PUT` and `DELETE`. Restrict methods that write: `@frappe.whitelist(methods=["POST"])`.
- The server does not evaluate `depends_on`, `mandatory_depends_on` or `read_only_depends_on`. Only the desk does.
- Dotted paths in `hooks.py` fail only when the hook runs. Check that each path imports.
- `override_doctype_class`, `extend_doctype_class`, `doctype_js` and `doctype_list_js` do not apply to custom DocTypes (`custom = 1`).
- The desk is at `/desk`. Old `/app/...` links redirect.

## Lifecycle order (insert)

`before_insert` → `before_naming` → `autoname` → `before_validate` → `validate` → `before_save` → DB insert → `after_insert` → `on_update` → `on_change`. For each step, Frappe runs the controller method, then the `doc_events` handlers for the DocType, then the `"*"` handlers. See [Controllers](developer-guide/data-model/controllers.md) for save, submit, cancel, delete and the exact rules.

## Pitfalls seen in this code base

- `require_type_annotated_api_methods` is set in frappe's own `hooks.py`, but no version 16 code reads it.
- The `regex` and `ilike` filters fail on MariaDB.
- `server_script_enabled` works only in `common_site_config.json`.
- Fixture import deletes and inserts again any record that already exists. Changes made on the site to a fixture record are lost at each `bench migrate`.
- A background job that is retried after a deadlock runs as Administrator.
- `frappe.flow.lib.session.new_session()` commits. An agent run inside a document event commits in the middle of the save.
