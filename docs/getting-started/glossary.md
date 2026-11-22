---
title: Glossary
description: Short definitions of the terms used in the Frappe Framework and its documentation.
version: develop
sources:
  - frappe/model/__init__.py
  - frappe/model/docstatus.py
  - frappe/model/document.py
  - frappe/core/doctype/doctype/doctype.json
  - frappe/core/doctype/docfield/docfield.json
  - frappe/core/doctype/module_def/module_def.json
  - frappe/core/doctype/report/report.json
  - frappe/core/doctype/user_permission/user_permission.json
  - frappe/desk/doctype/sidebar/sidebar.py
  - frappe/desk/doctype/dock/dock.json
  - frappe/desk/doctype/dock_item/dock_item.json
  - frappe/utils/fixtures.py
  - frappe/modules/patch_handler.py
  - frappe/__init__.py
  - frappe/hooks.py
---

# Glossary

Use this page to look up a Frappe term. Each entry has a short definition and a link to the page with the details. Names in backticks are exact DocType names, fieldnames, hook keys or Python names.

## Installation and structure

| Term | Definition |
|---|---|
| Bench | A folder that holds the apps, the Python environment and the sites. Also the name of the `bench` CLI. See [Architecture](architecture.md). |
| App | A Python package in `apps/` that adds DocTypes, pages, APIs and hooks. Frappe is also an app. See [Apps](../developer-guide/building-apps/apps.md). |
| Site | One tenant of a bench: one database, one `sites/<site>/` folder and one `site_config.json`. |
| Module | A group of DocTypes, reports, pages and workspaces inside an app. Stored as a `Module Def` record and listed in the app's `modules.txt`. |
| Site config | The JSON settings of a site (`site_config.json`), merged with `common_site_config.json`. Read in code with `frappe.conf`. See [Configuration](../developer-guide/operations/configuration.md). |
| Developer mode | The site config key `developer_mode`. When on, DocTypes and other standard records are saved as files in the app. |
| Hook | A variable in an app's `hooks.py` that Frappe reads to extend the framework, for example `doc_events` or `scheduler_events`. See [Hooks](../developer-guide/building-apps/hooks.md). |
| Fixture | Records that an app exports as JSON (in `<app>/fixtures/`) with the `fixtures` hook. Frappe imports them on install and migrate. See [Fixtures and patches](../developer-guide/building-apps/fixtures-and-patches.md). |
| Patch | A Python function listed in the app's `patches.txt` that `bench migrate` runs once per site. Sections `[pre_model_sync]` and `[post_model_sync]` set when it runs. |
| Migrate | `bench --site <site> migrate`. Runs patches, syncs DocType JSON to the database and imports fixtures. |

## Data model

| Term | Definition |
|---|---|
| DocType | The definition of a data type: fields, permissions, naming and layout. Each DocType has a table named `tab<DocType>`. See [DocTypes](../developer-guide/data-model/doctypes.md). |
| Document | One record of a DocType. In Python, an instance of `frappe.model.document.Document`. |
| DocField | One field of a DocType. Has a `fieldname`, a `fieldtype`, `options` and other properties. See [Field types](../reference/data-model/field-types.md). |
| Standard fields | Columns in every table: `name`, `owner`, `creation`, `modified`, `modified_by`, `docstatus`, `idx`. |
| Child Table | A DocType with `istable = 1`. Its rows belong to a parent document through the `parent`, `parenttype` and `parentfield` columns. |
| Single | A DocType with `issingle = 1`. It has one record and no table of its own. Its values are stored in `tabSingles`. Used for settings. |
| Tree | A DocType with `is_tree = 1`. Its records form a hierarchy. The controller uses `NestedSet`. |
| Virtual DocType | A DocType with `is_virtual = 1`. It has no table. The controller loads and saves the data. |
| Custom DocType | A DocType with `custom = 1`. It exists only in the site database, not in an app. |
| Standard | A record that belongs to an app and is saved as a file in it, for example a standard DocType or report. |
| Controller | The Python class of a DocType (`<doctype>.py`). Its methods, such as `validate` and `on_submit`, run during the document lifecycle. See [Controllers](../developer-guide/data-model/controllers.md). |
| Submittable | A DocType with `is_submittable = 1`. Its documents can be submitted and cancelled. |
| `docstatus` | The state of a document: `0` Draft, `1` Submitted, `2` Cancelled. See `frappe.model.docstatus.DocStatus`. |
| Amend | Make a new draft copy of a cancelled document. The copy links to the original in `amended_from`. |
| Naming | How the `name` (primary key) of a document is set, for example with `autoname`. See [Naming](../developer-guide/data-model/naming.md). |
| Link field | A field of type `Link` that stores the `name` of a document of another DocType, set in `options`. |
| Custom Field | A field added to a DocType for one site, stored in `Custom Field`. See [Customize Form](../user-guide/customization/customize-form.md). |
| Property Setter | A record that changes one property of a DocType or field for one site. |

## Desk and UI

| Term | Definition |
|---|---|
| Desk | The admin UI for System Users, served at `/desk`. See [Desk](../user-guide/basics/desk.md). |
| Workspace | A desk page with shortcuts, links, charts and number cards. Stored in `Workspace`. See [Workspaces, sidebar and dock](../developer-guide/desk-ui/workspaces-sidebar-dock.md). |
| Sidebar | The navigation of a module in the desk. An app can ship it as a `Sidebar` record. Otherwise Frappe computes it from the module's workspaces, DocTypes, reports and pages. |
| Dock | The rail of links of one app in the desk. Stored in `Dock`. Each `Dock Item` links to a sidebar, a workspace or a URL. |
| Desktop Icon | An icon on the desk home page (`/desk`). Stored in `Desktop Icon`. |
| Form view | The page that shows one document. See [Form view](../user-guide/basics/form-view.md). |
| List view | The page that shows the documents of a DocType. See [List view](../user-guide/basics/list-view.md). |
| Form script | JavaScript for a DocType form, in `<doctype>.js`, with `frappe.ui.form.on`. See [Form scripts](../developer-guide/desk-ui/form-scripts.md). |
| Client Script | A form or list script saved as a record on the site, not in an app. See [Server and client scripts](../developer-guide/low-code-and-ai/server-and-client-scripts.md). |
| Server Script | Python code saved as a record on the site. Runs in a restricted environment. |
| Report | A saved report. `report_type` is `Report Builder`, `Query Report`, `Script Report` or `Custom Report`. See [Reports](../developer-guide/output/reports.md). |
| Page | A custom desk page with its own JS, stored in `Page`. See [Desk pages](../developer-guide/desk-ui/desk-pages.md). |
| Web Form | A form on the website for Guests or Website Users that makes or edits documents. See [Web forms](../developer-guide/website/web-forms.md). |
| Print Format | A template that makes the print and PDF output of a document. See [Print formats](../developer-guide/output/print-formats.md). |

## Users and permissions

| Term | Definition |
|---|---|
| Administrator | The built-in super user of a site. |
| Guest | The user of a request that is not logged in. |
| System User | A user who can use the desk. |
| Website User | A user who can use only the website and portal, not the desk. |
| Role | A named set of permissions given to users, for example `System Manager`. See [Users and roles](../user-guide/users-and-permissions/users-and-roles.md). |
| Role permission | A row in a DocType's permissions: a role, a permission level and rights such as read, write, create, submit and delete. See [Permissions](../developer-guide/security/permissions.md). |
| Permission Level | The `permlevel` of a field (default `0`). Roles get rights per level, so you can restrict groups of fields. |
| User Permission | A `User Permission` record that limits a user to documents linked to one value, for example one company. See [User permissions](../user-guide/users-and-permissions/user-permissions.md). |
| Share | Access to one document given to one user, outside the role permissions. |

## Server and API

| Term | Definition |
|---|---|
| Whitelisted method | A Python function with the `@frappe.whitelist()` decorator. It can be called over HTTP at `/api/method/<dotted.path>`. See [Whitelisted methods](../developer-guide/api/whitelisted-methods.md). |
| REST API | The HTTP API under `/api/v1/` (also `/api/`) and `/api/v2/`. See [REST API](../developer-guide/api/rest-api.md). |
| `frappe.local` | Request-local state: site, database connection, user, form data. |
| `frappe.form_dict` | The parameters of the current request, from the query string and body. |
| Background job | A function that a worker runs outside the request, put in a queue with `frappe.enqueue`. See [Background jobs](../developer-guide/runtime/background-jobs.md). |
| Queue | A Redis job queue. Built-in queues: `short`, `default`, `long`. |
| Worker | A process (`bench worker`) that runs background jobs. |
| Scheduler | The process (`bench schedule`) that enqueues scheduled jobs. Jobs are set with the `scheduler_events` hook. See [Scheduler](../developer-guide/runtime/scheduler.md). |
| Realtime | Live events from the server to the browser over Socket.IO, sent with `frappe.publish_realtime`. See [Realtime](../developer-guide/runtime/realtime.md). |
| Cache | The Redis cache of the site, used with `frappe.cache`. See [Caching](../developer-guide/runtime/caching.md). |

## Related

- [Architecture](architecture.md)
- [Quickstart](quickstart.md)
- [Hooks reference](../reference/configuration/hooks.md)
- [DocTypes reference](../reference/data-model/doctypes.md)
