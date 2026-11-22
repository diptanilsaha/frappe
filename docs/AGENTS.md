---
title: For AI Agents
description: Rules, verified facts and pitfalls for AI agents that build or change Frappe apps on develop.
version: develop
sources:
  - frappe/model/document.py
  - frappe/__init__.py
  - frappe/permissions.py
  - frappe/database/database.py
  - frappe/utils/background_jobs.py
---

# For AI Agents

Read this page before you write or change code for a Frappe app on develop (17.0.0-dev). Every fact here was checked in this branch. For an index of all pages with one-line summaries, read [llms.txt](llms.txt).

## Know the version first

1. Run `bench version` or read `frappe/__init__.py` `__version__`. These docs are for `17.0.0-dev`.
2. Do not use an API from another version without checking it. For example, `frappe.read_doc`, `frappe.enqueue_task`, `bench enable-app` exist only in develop. `extend_doctype_class` exists in version 16 and later.

## Where to look

| Question | Page |
|---|---|
| Which hooks exist and what shape they take | [Hooks reference](reference/configuration/hooks.md) |
| Run order of controller methods and `doc_events` | [Controllers](developer-guide/data-model/controllers.md) |
| Signatures of `frappe.*`, `frappe.db.*`, `frappe.utils.*` | [Python API](reference/api/python-api.md), [Database API](reference/api/database-api.md), [Utils](reference/api/utils.md) |
| Client-side `frappe.*` and `frm` API | [JavaScript API](reference/api/js-api.md), [Form Scripts](developer-guide/desk-ui/form-scripts.md) |
| REST routes | [REST Endpoints](reference/api/rest-endpoints.md) |
| CLI commands and flags | [CLI](reference/configuration/cli.md) |
| Config keys | [Site Config](reference/configuration/site-config.md) |
| Field types and DB columns | [Field Types](reference/data-model/field-types.md) |

## Rules

- Use `frappe.session.user` for the current user. `frappe.user` is `None` on develop.
- Check permissions in every whitelisted method. Use `frappe.read_doc` or `doc.check_permission()`. `frappe.get_doc` does not check read permission.
- `frappe.get_all` ignores permissions. `frappe.get_list` applies them. Neither has a default row limit on develop. Pass `limit` (or `page_length`).
- `frappe.db.set_value`, `frappe.db.sql` and `frappe.qb` skip validations, controller methods and permissions. `frappe.db.set_value` clears the document cache. Raw SQL and `frappe.qb` updates do not.
- `frappe.qb.get_query` ignores permissions by default and adds no `ORDER BY`.
- Write SQL functions in `fields` in the dict form. A string such as `"count(name) as c"` raises `ValidationError`.
- Do not call `frappe.db.commit()` in a request handler or a controller method. The request commits when it succeeds. Inside `doc_events` handlers a commit is ignored with a warning. Writes in a `GET` request are rolled back.
- Use `enqueue_after_commit=True` when a job reads data that the current request writes.
- Add type annotations to every whitelisted method. Request arguments are validated against them. Apps made by `bench new-app` set `require_type_annotated_api_methods = True`, so a method without annotations fails.
- A whitelisted method without `methods=[...]` accepts `GET`. Restrict methods that write: `@frappe.whitelist(methods=["POST"])`.
- The server does not evaluate `depends_on`, `mandatory_depends_on` or `read_only_depends_on`. Only the desk does.
- Dotted paths in `hooks.py` fail only when the hook runs. Check that each path imports.
- `override_doctype_class`, `extend_doctype_class`, `doctype_js` and `doctype_list_js` do not apply to custom DocTypes (`custom = 1`).
- Deleting a DocType or a Custom Field leaves its database table or column in place.

## Lifecycle order (insert)

`before_insert` → `before_naming` → `autoname` → `before_validate` → `validate` → `before_save` → DB insert → `after_insert` → `on_update` → `on_change`. For each step, Frappe runs the controller method, then the `doc_events` handlers for the DocType, then the `"*"` handlers. See [Controllers](developer-guide/data-model/controllers.md) for save, submit, cancel, delete and the exact rules.

## Pitfalls seen in this code base

- `naming_series:` reads only the `naming_series` field.
- All `format:` names with `{#####}` and no fixed prefix share one counter in `tabSeries`.
- A Server Script with event "Before Save" runs on `validate`, not on `before_save`.
- `server_script_enabled` works only in `common_site_config.json`.
- Fixture import deletes and inserts again any record that already exists. Changes made on the site to a fixture record are lost at each `bench migrate`.
- `frappe.sendmail(now=True)` sends after the transaction commits.
- A query report runs in a read-only transaction. Uncommitted writes in the same transaction raise `ImplicitCommitError`.
- Error, access and activity logs are in `sites/<site>/private/files/sqlite.db`, not in the main database.
