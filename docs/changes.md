---
title: What's New in develop
description: Changes in Frappe Framework develop (17.0.0-dev) compared with version 16, verified against both code bases.
version: develop
sources:
  - frappe/__init__.py
  - frappe/app_state.py
  - frappe/storage
  - frappe/api/v2.py
  - frappe/hooks.py
---

# What's New in develop

This page lists the changes in develop (17.0.0-dev) since version 16. Each item was checked in both code bases. For steps to move an app, see [Upgrading](developer-guide/operations/upgrading.md).

!!! warning "Unreleased"
    develop is not a release. APIs on this page can change before version 17.

## New features

| Area | Change | Docs |
|---|---|---|
| Apps | An app can be disabled on a site and stay installed. New commands `bench enable-app` and `bench disable-app`. New hooks `before_enable`, `after_enable`, `before_disable`, `after_disable`. New functions `frappe.get_active_apps()` and `frappe.get_disabled_apps()`. | [App State](developer-guide/building-apps/app-state.md) |
| Files | Storage v2 (alpha): pluggable drivers (`storage_drivers` hook), the `File Blob` DocType, signed URLs. It is off unless the `storage_v2` site config flag is set. New hooks `after_file_upload` and `ignore_file_permissions`. | [Files](developer-guide/data-model/files.md) |
| Jobs | `frappe.enqueue_task()` and the `Background Task` DocType. `MapReduce Job` and `MapReduce Task` DocTypes. | [Background Jobs](developer-guide/runtime/background-jobs.md) |
| Permissions | `frappe.read_doc()` gets a document and checks read permission. | [Permissions](developer-guide/security/permissions.md) |
| REST API | v2 bulk endpoints: `/api/v2/document/<doctype>/bulk_delete`, `/api/v2/document/<doctype>/bulk_update`, `/api/v2/method/bulk_delete`, `/api/v2/method/bulk_update`. The HTTP `QUERY` method is accepted for list calls. | [REST API](developer-guide/api/rest-api.md) |
| Whitelisted methods | Hook `require_type_annotated_api_methods`: when true, a whitelisted method without type annotations fails when called. New apps set it to true. Hook `use_json_request_body`. | [Whitelisted Methods](developer-guide/api/whitelisted-methods.md) |
| Requests | Hook `streaming_request_paths`. | [Hooks reference](reference/configuration/hooks.md) |
| Printing | The default PDF generator is PlutoPrint. wkhtmltopdf is removed. A patch moves sites that used it to Chrome. | [Print Formats](developer-guide/output/print-formats.md) |
| Logs | Error Log, Access Log, Activity Log, Route History and Web Page View rows are stored in `sites/<site>/private/files/sqlite.db`. A database backup does not include them. Back up files too. | [Backups](user-guide/system-administration/backups.md) |
| Desk | The list sidebar group-by is replaced by "Customize Quick Filters". Saved list layouts are `List Filter` documents. New Kanban boards use Kanban v2. | [List View](user-guide/basics/list-view.md) |

## New DocTypes

`Background Task`, `Custom Icon`, `Document Template`, `DocType Layout Child Table`, `File Blob`, `MapReduce Job`, `MapReduce Task`, `Notification Webhook URL`, `Server Script Variable`, `Workflow Condition`.

## Moved, renamed or removed

| Item | Change |
|---|---|
| `DocType Layout`, `DocType Layout Field` | Moved from the Custom module to the Core module. |
| `Slack Webhook URL` | Renamed to `Notification Webhook URL`. The patches `rename_slack_webhook_url` and `migrate_slack_notifications` move existing data. A Notification with channel `Slack` in an app fixture or a JSON export is converted when it loads. |
| `Sidebar Item Group`, `Sidebar Item Group Link` | Removed. |
| `frappe/__init__.py` | Public names load lazily from other modules. `frappe.<name>` still works. Path and app helpers (`get_app_path`, `get_site_path`, `get_module_path`, `get_installed_apps`, `get_attr`, `scrub` and others) are defined in other modules now. Import them from `frappe`, not from their new module. |
| Build | esbuild is updated to version 0.28. `@frappe/esbuild-plugin-postcss2` is removed. |
| Tests | The `cypress/` folder is removed. |

## Unchanged requirements

Python `>=3.14,<3.15` and Node `>=24`, the same as version 16. See [Installation](getting-started/installation.md).
