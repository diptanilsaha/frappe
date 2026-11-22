---
title: What's New in v16
description: Changes in Frappe Framework version 16 compared with version 15, verified against both code bases.
version: v16
sources:
  - pyproject.toml
  - package.json
  - frappe/hooks.py
  - frappe/__init__.py
  - frappe/deprecation_dumpster.py
  - frappe/patches.txt
---

# What's New in v16

This page lists the changes in version 16 since version 15. Each item was checked in both code bases. For the steps to move an app, see [Upgrading](developer-guide/operations/upgrading.md).

## Requirements

| | v15 | v16 |
|---|---|---|
| Python | `>=3.10,<3.15` | `>=3.14,<3.15` |
| Node | `>=18` | `>=24` |
| Databases | MariaDB, PostgreSQL | MariaDB, PostgreSQL, SQLite (experimental) |
| MariaDB driver | PyMySQL | mysqlclient (`use_mysqlclient: 0` selects PyMySQL, deprecated) |

## Desk

| Change | Docs |
|---|---|
| The desk moved from `/app` to `/desk`. `/app/...` redirects to `/desk/...`. | [Desk](user-guide/basics/desk.md) |
| New Desktop page (`/desk`), Sidebar, Workspace Sidebar and Dock. Each module ships a Sidebar and Workspace fixtures. | [Workspaces, Sidebar and Dock](developer-guide/desk-ui/workspaces-sidebar-dock.md) |
| Vue islands can be mounted in desk pages. | [Islands](developer-guide/desk-ui/islands.md) |

## New features

| Area | Change | Docs |
|---|---|---|
| Flow | AI agents with tools, skills, knowledge bases and triggers. Route `/flow`. Hooks `flow_agents`, `flow_tools`, `flow_triggers`. Command group `bench flow-server`. | [Flow](user-guide/ai-and-app-builder/flow.md), [Flow Development](developer-guide/low-code-and-ai/flow-development.md) |
| Studio | Visual app builder at `/studio`. Commands `build-studio-app` and `watch-studio`. | [Studio](user-guide/ai-and-app-builder/studio.md), [Studio Development](developer-guide/low-code-and-ai/studio-development.md) |
| Automation | Automation Flow DocType and the automation engine. Hooks `automation_events`, `automation_actions`, `automation_relationships`, `automation_relationship_ignore`. | [Automation Flow](user-guide/automation/automation-flow.md), [Automation Engine](developer-guide/low-code-and-ai/automation-engine.md) |
| MCP and discovery | MCP server at `/api/v2/mcp` (System Settings `enable_mcp`). API discovery at `/api/v2/discovery`. Hook `expose_discovery_source`. A patch turns on `enable_mcp` on upgraded sites. | [MCP and Discovery](developer-guide/api/mcp-and-discovery.md) |
| Extending | Hook `extend_doctype_class` adds a mixin to a DocType controller without replacing it. | [Extending Apps](developer-guide/building-apps/extending-apps.md) |
| Permissions | Permission Type and Permission Log DocTypes. | [Permissions](user-guide/users-and-permissions/permissions.md) |
| Printing | PDF generators: Chrome (default for Print Settings), PlutoPrint and wkhtmltopdf. Print Format also offers Typst and WeasyPrint (deprecated). Print Format Snippet DocType. | [Print Formats](developer-guide/output/print-formats.md) |
| Data Import | Value mapping, skipped rows, hooks `data_import_providers` and `importable_doctypes`. | [Data Import and Export](user-guide/data-and-reports/data-import-export.md) |
| Workflow | Workflow Transition Tasks. Hook `workflow_methods`. | [Workflow](user-guide/customization/workflow.md) |
| Website | UTM Source, UTM Medium and UTM Campaign DocTypes. | [Website](user-guide/website-and-integrations/website.md) |
| Tests | New test runner in `frappe/testing`. Test classes `IntegrationTestCase` and `UnitTestCase` (`frappe/tests/classes`). | [Testing](developer-guide/testing-and-performance/testing.md) |
| Python API | `frappe.toast`, `frappe.throw_permission_error`, `frappe.site_cache`, `frappe.concurrent_limit`. `frappe.get_lazy_doc` loads child tables on access (in v15 it was an alias of `get_doc`). | [Python API](reference/api/python-api.md) |
| CLI | `list-sites`, `bypass-patch`, `convert-sidebar-fixtures`, `setup-chrome`, `sqlite`, `sync-desktop-icons`, `build-studio-app`, `watch-studio`, `flow-server`. | [CLI](reference/configuration/cli.md) |
| Realtime | A Python Socket.IO server in `frappe/realtime`. `bench socketio` picks it when `socketio_backend` is set. `frappe/asgi.py` exists. | [Realtime](developer-guide/runtime/realtime.md) |
| Security | Session IDs are stored hashed. | [Authentication](developer-guide/security/authentication.md) |

## Removed

| Item | Notes |
|---|---|
| Blog (Blog Post, Blog Category, Blogger, Blog Settings) | Removed from frappe. |
| Newsletter (Newsletter, Newsletter Attachment, Newsletter Email Group), Marketing Campaign | Removed from frappe. |
| Energy Points (Social module: Energy Point Log, Rule, Settings, Review Level), leaderboard, `leaderboards` hook | Removed. |
| Dropbox Settings, Google Drive, S3 Backup Settings | Offsite backup integrations removed. |
| Changelog Feed, `get_changelog_feed` hook, Transaction Log | Removed. |
| `frappe.get_version()` | Removed. |
| `bench update-csv-from-po` | Removed. |

## Deprecated

`frappe/deprecation_dumpster.py` holds deprecated functions. They print a warning and are planned for removal in version 17. Examples: `frappe.connect(site=...)`, the `txt` argument of `frappe.generate_hash`, `frappe.get_test_records`, and the gravatar helpers in `frappe.utils`.
