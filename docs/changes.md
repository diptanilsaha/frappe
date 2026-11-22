---
title: What's New in v15
description: Changes in Frappe Framework version 15 compared with version 14, verified against both code bases.
version: v15
sources:
  - pyproject.toml
  - package.json
  - frappe/__init__.py
  - frappe/hooks.py
  - frappe/modules.txt
  - frappe/api/__init__.py
  - frappe/api/v2.py
  - frappe/commands/gettext.py
  - frappe/commands/scheduler.py
  - frappe/tests/utils.py
  - frappe/patches/v15_0/remove_event_streaming.py
  - frappe/core/doctype/system_settings/system_settings.py
---

# What's New in v15

This page lists the changes in version 15 since version 14. The base for the comparison is the `v14.99.9` tag. For the steps to move an app, see [Upgrading](developer-guide/operations/upgrading.md).

## Requirements

| | v14 | v15 |
|---|---|---|
| Python | `>=3.10` | `>=3.10,<3.15` |
| Node | `>=14` | `>=18` |
| Vue (desk bundles) | `2.6.14`, Vuex 3, Vue Router 2 | `^3.3.0`, Vuex 4, Vue Router 4, Pinia |
| New Python dependency | | `duckdb` |

## New features

| Area | Change | Docs |
|---|---|---|
| REST API | Versioned routes `/api/v1/*` and `/api/v2/*`. `/api/*` stays on v1. v2 adds `/api/v2/document/<doctype>/...`, `/api/v2/doctype/<doctype>/meta` and `/api/v2/doctype/<doctype>/count`. `frappe/api.py` is now the `frappe/api` package. | [REST API](developer-guide/api/rest-api.md), [REST Endpoints](reference/http/rest-endpoints.md) |
| DocType | Form Builder in the DocType form (`frappe/public/js/form_builder`). New DocType options `queue_in_background`, `protect_attached_files`, `grid_page_length`, `rows_threshold_for_grid_search`. | [DocTypes](developer-guide/data-model/doctypes.md) |
| Workflow | Workflow Builder page at `/app/workflow-builder`. | [Workflow](user-guide/customization/workflow.md) |
| Submission | Submission Queue DocType runs submit in a background job when the DocType has `queue_in_background` set. | [Background Jobs](developer-guide/runtime/background-jobs.md) |
| Apps screen | Page `/apps` lists installed apps. Hook `add_to_apps_screen`. System Settings and User get a `default_app` field. | [Hooks](reference/configuration/hooks.md) |
| Reports | Snapshot reports: DuckDB Sync and DuckDB Sync Item DocTypes, System Settings `enable_snapshot_reports` and `frequency`. Prepared Report Analytics report. Prepared Report stores `peak_memory_usage`. | [Reports](user-guide/data-and-reports/reports.md) |
| Monitoring | Recorder is now a virtual DocType (Recorder, Recorder Query, Recorder Suggested Index). System Settings has a new `log_api_requests` field, but no v15 code reads it. | [Logs and Monitoring](user-guide/system-administration/logs-and-monitoring.md) |
| Permissions | Permission Inspector DocType checks one permission type of a user on a DocType or document. | [Permissions](user-guide/users-and-permissions/permissions.md) |
| Users | User Invitation DocType and hook `user_invitation`. User Role DocType. | [Users and Roles](user-guide/users-and-permissions/users-and-roles.md) |
| Reminders | Reminder DocType (`user`, `reminder_doctype`, `reminder_docname`, `remind_at`). | [Notifications](user-guide/automation/notifications.md) |
| Notifications | Notification Type, Notification Type Preference and Event Notifications DocTypes. Hooks `notification_self_notify_types`, `notification_skip_email_types`. Push Notification Settings DocType and `frappe/push_notification.py`. | [Notifications](user-guide/automation/notifications.md) |
| Naming | Amended Document Naming Settings table in Document Naming Settings. | [Naming](developer-guide/data-model/naming.md) |
| Security | Security Settings DocType builds `security.txt` content (contacts, languages, policy, expiry). A daily job checks the expiry. | [Security](developer-guide/security/security.md) |
| Website | Web Form Request DocType. Marketing Campaign DocType in the Website module. Hooks `website_path_resolver`, `get_web_pages_with_dynamic_routes`. | [Website](developer-guide/website/website.md), [Web Forms](developer-guide/website/web-forms.md) |
| Printing | Hooks `pdf_generator`, `on_print_pdf`, `pdf_body_html`, `get_print_format_template`. | [Print Formats](developer-guide/output/print-formats.md) |
| Search | `frappe/search/sqlite_search.py` builds SQLite full-text indexes for classes listed in hook `sqlite_search`. Hook `awesomebar_search`. | [Search](user-guide/basics/search.md) |
| Translations | Gettext PO files in `frappe/locale` and the `frappe/gettext` package. Hook `ignore_translatable_strings_from`. | [Translations](developer-guide/building-apps/translations.md) |
| Typing | Hook `export_python_type_annotations` writes field type hints into controllers. Package `frappe/types`. | [Controllers](developer-guide/data-model/controllers.md) |
| Other hooks | `after_build`, `after_app_build`, `default_log_clearing_doctypes`, `never_skip_patches`, `persistent_cache_keys`, `standard_navbar_items`, `standard_help_items`. | [Hooks](reference/configuration/hooks.md) |
| Python API | `frappe.get_single_value`, `frappe.is_setup_complete`, `frappe.get_app_source_path`, `frappe.get_common_site_config`, `frappe._lt` (lazy translation), `frappe.N_`, `frappe.job`. `frappe.get_lazy_doc` is an alias of `frappe.get_doc`. | [Python API](reference/python/frappe.md) |
| Cache | `frappe.cache` is a `RedisWrapper` object. `frappe.cache()` still works. | [Caching](developer-guide/runtime/caching.md) |
| CLI | `worker-pool`, `generate-pot-file`, `compile-po-to-mo`, `migrate-csv-to-po`, `update-po-files`, `create-po-file`, `update-csv-from-po`. | [CLI](reference/configuration/cli.md) |
| Tests | `MockedRequestTestCase` in `frappe/tests/utils.py`. `FrappeTestCase` stays the base class. | [Testing](developer-guide/testing-and-performance/testing.md) |
| Scheduler | Scheduler Event DocType. | [Scheduler](developer-guide/runtime/scheduler.md) |

## Removed

| Item | Notes |
|---|---|
| Event Streaming module (Event Producer, Event Consumer, Event Update Log, Event Sync Log, Document Type Mapping and their child tables) | Patch `frappe.patches.v15_0.remove_event_streaming` deletes them unless the `event_streaming` app is installed. |
| Error Snapshot DocType | Removed. |
| Pages `background_jobs`, `recorder` and `activity` | Use the RQ Job list and the Recorder DocType. |
| `bench import-csv` | Removed. Use Data Import. |
| Hooks `communication_doctypes` and `get_translated_dict` | No longer read. |

## Deprecated

`frappe/utils/deprecations.py` gives the `@deprecated` decorator and `deprecation_warning()`. Examples that warn in v15: `frappe.handler.uploadfile` (use `upload_file`), the `cmd` form parameter for RPC calls (use `/api/method/<cmd>`), `frappe.db.set_value` on a single DocType (use `frappe.db.set_single_value`), the `job_name` argument of `frappe.enqueue` (use `job_id`), and the gravatar helpers in `frappe.utils`.
