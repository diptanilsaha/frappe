---
title: Upgrading
description: Notable breaking changes from v15 to v16, how deprecations work in deprecation_dumpster, and the steps to upgrade a bench.
version: v16
sources:
  - frappe/deprecation_dumpster.py
  - frappe/hooks.py
  - frappe/patches.txt
  - pyproject.toml
  - package.json
  - frappe/model/qb_query.py
  - frappe/commands/utils.py
  - frappe/desk/doctype/sidebar/convert_fixtures.py
  - frappe/utils/print_utils.py
  - frappe/printing/doctype/print_format/print_format.json
  - frappe/printing/doctype/print_settings/print_settings.json
  - frappe/patches/v16_0/switch_default_sort_order.py
  - frappe/patches/v16_0/hash_session_ids.py
  - frappe/patches/v16_0/merge_studio_into_frappe.py
  - frappe/patches/v16_0/add_module_deprecation_warning.py
  - frappe/patches/v16_0/notify_apps_to_convert_sidebar_fixtures.py
  - frappe/patches/v16_0/set_classic_dashboards.py
  - frappe/patches/v16_0/sync_installed_apps_to_site_config.py
  - frappe/patches/v17_0/keep_existing_sites_on_classic_charts.py
---

# Upgrading

This page lists changes that can break an app or a site when you move from v15 to v16. It lists only changes found by comparing the code of the two versions. Read it before you upgrade, and test the upgrade on a copy of the site.

## Upgrade steps

1. Take a backup with files: `bench --site <site> backup --with-files`. Keep the `encryption_key` from `site_config.json`.
2. Check the Python and Node versions in the table below. Install the new versions first.
3. Update each app to its `version-16` branch. All apps on a bench must support the same Frappe version.
4. Run `bench setup requirements` to install Python and Node dependencies.
5. Run `bench --site <site> migrate` for each site.
6. Run `bench build`.
7. Restart the processes: `bench restart` in production, or start `bench start` again in development.

In production, turn on maintenance mode before step 5 with `bench --site <site> set-maintenance-mode on`. Turn it off after step 7. `bench update` from the bench CLI runs the pull, requirements, migrate, build and restart steps for you.

## Requirements

| | v15 | v16 |
|---|---|---|
| Python (`requires-python` in `pyproject.toml`) | `>=3.10,<3.15` | `>=3.14,<3.15` |
| Node (`engines.node` in `package.json`) | `>=18` | `>=24` |

v16 needs Python 3.14. A v15 bench on an older Python needs a new virtual environment: `bench setup env --python python3.14`, then `bench setup requirements`.

## Desk route is /desk

The desk moved from `/app` to `/desk`. Frappe redirects `/app`, `/app/...` and `/apps` to `/desk` with the query string (`website_redirects` in `frappe/hooks.py`). Update hardcoded links in apps, emails, print formats and external systems. In JavaScript, use `frappe.set_route` and `frappe.utils.get_form_link` instead of building `/app/` URLs.

## Modules moved to separate apps

These DocTypes are removed from frappe. Install the app to keep using them. During migrate, the patch `add_module_deprecation_warning` prints the app to install.

| Feature | DocTypes removed | App |
|---|---|---|
| Blog | Blog Post, Blog Category, Blogger, Blog Settings | `blogs` |
| Newsletter | Newsletter, Newsletter Attachment, Newsletter Email Group | `newsletter` |
| Energy Points | Energy Point Log, Energy Point Rule, Energy Point Settings, Review Level | `eps` |
| Offsite backups | Dropbox Settings, S3 Backup Settings, Google Drive | `offsite_backups` |

Also removed: `Changelog Feed`, `Transaction Log` and `Marketing Campaign`. The patch `migrate_to_utm` renames the `Marketing Campaign` DocType to `UTM Campaign`. v16 also adds `UTM Source` and `UTM Medium`. The hooks `leaderboards` and `get_changelog_feed` are no longer read.

## get_list rejects SQL in fields

`frappe.get_list` and `frappe.get_all` build queries with the query builder (`frappe.model.qb_query`). SQL functions written as strings in `fields` raise `ValidationError` ("SQL functions are not allowed as strings in SELECT"). Use the dict form:

```python
# v15
frappe.get_all("ToDo", fields=["count(name) as count"])

# v16
frappe.get_all("ToDo", fields=[{"COUNT": "name", "as": "count"}])
```

For other expressions, write a query with `frappe.qb`.

## Default sort is creation

The default `sort_field` of a new DocType is `creation`, not `modified`. The patch `switch_default_sort_order` (MariaDB only) does these things:

- It adds an index on `creation` to the table of every parent DocType.
- It drops the `modified` index where the DocType does not sort by `modified`.
- It changes saved list settings that sort by `modified` to the sort field of the DocType.

Code that expects lists sorted by last change must pass `order_by="modified desc"`.

## Print and PDF

In v15, the `pdf_generator` field of Print Format has one option, `wkhtmltopdf`. In v16, the options are `wkhtmltopdf`, `chrome`, `PlutoPrint`, `Typst` and `WeasyPrint`, with no default. Print Settings has a new site-wide `pdf_generator` (`wkhtmltopdf`, `chrome`, `PlutoPrint`, default `chrome`).

`frappe.utils.print_utils.resolve_pdf_generator` picks the engine in this order: an explicit choice, the field of the Print Format, the Print Settings value, then `wkhtmltopdf`. Print formats made in v15 keep their stored `wkhtmltopdf` value. Check the PDF output of custom print formats after you change the engine. See [Print Formats](../output/print-formats.md).

## Workspace sidebar fixtures

v16 adds a sidebar for each module (the `Sidebar` DocType). An app ships one per module, in `<app>/<module>/sidebar/`. The flat `<app>/workspace_sidebar/*.json` folder from earlier v16 releases is no longer imported. Until you convert it, the module shows a generated sidebar. Migrate prints the apps that still have the old folder. Convert the files once and commit the result:

```bash
bench --site <site> convert-sidebar-fixtures --app sample_app --dry-run
bench --site <site> convert-sidebar-fixtures --app sample_app
```

The command only reads the old folder and leaves it in place. Without `--app`, it converts every installed app that still has the old folder. See [Workspaces, Sidebar and Dock](../desk-ui/workspaces-sidebar-dock.md).

## Other changes

- Session IDs are stored as SHA-256 hashes. The patch `hash_session_ids` hashes existing sessions, so users stay logged in. Code that reads `tabSessions` by the raw cookie value no longer finds the row.
- Studio is part of frappe. On a site that had the separate `studio` app, migrate moves it into frappe and removes `studio` from the installed apps. Run `bench remove-app studio` after every site of the bench has migrated.
- Flow (AI agents), Automation Flow and MCP are new in v16. A patch sets `enable_mcp` in System Settings on upgrade. Turn it off if you do not want the MCP endpoint. See [MCP and Discovery](../api/mcp-and-discovery.md).
- Upgraded sites keep the classic dashboards (`dashboards` set to `Classic`) and the classic charts (`use_classic_charts` set). New sites get `Insights` dashboards and the new charts. Change both in System Settings.
- A patch writes `installed_apps` to `site_config.json`. Install and uninstall keep it in sync.
- `frappe.get_version` is removed.
- New app hooks include `extend_doctype_class`, `fixture_auto_order`, `importable_doctypes`, `data_import_providers`, `workflow_methods`, `has_comment_permission`, `on_social_login`, `studio_app_boot`, `flow_agents`, `flow_tools`, `flow_triggers` and the `automation_*` hooks. See [Hooks reference](../../reference/configuration/hooks.md).
- A new app marks each module with a `.frappe` file in the module folder. `modules.txt` is still written, for compatibility.
- Test classes: v16 adds `frappe.tests.UnitTestCase` and `frappe.tests.IntegrationTestCase`. `frappe.tests.utils.FrappeTestCase` still works with a deprecation warning. See [Testing](../testing-and-performance/testing.md).

## Deprecations

Frappe keeps deprecated functions in `frappe/deprecation_dumpster.py`. Each one has the date it was marked and the version from which it is an error.

| Warning class | Behavior |
|---|---|
| `V15FrappeDeprecationWarning` | An error (subclass of `FrappeDeprecationError`). |
| `V16FrappeDeprecationWarning` | A warning. |
| `V17FrappeDeprecationWarning` | Ignored by default (`PendingFrappeDeprecationWarning`). |

To see pending warnings, or to turn an error back into a warning in an emergency, set `PYTHONWARNINGS`. For example: `PYTHONWARNINGS="default::frappe.deprecation_dumpster.V17FrappeDeprecationWarning"`.

Functions marked for removal in v17 include:

| Deprecated | Use instead |
|---|---|
| `frappe.tests.utils.FrappeTestCase` | `frappe.tests.UnitTestCase` or `frappe.tests.IntegrationTestCase` |
| `frappe.tests.utils.change_settings`, `patch_hooks`, `debug_on`, `timeout` | The same names from `frappe.tests` |
| `frappe.tests_runner.*` | `frappe.tests.utils` functions, or no replacement |
| `frappe.get_test_records` | Test records load lazily |
| `frappe.db.is_column_missing` | `frappe.db.is_missing_column` |
| `frappe.gzip_compress`, `frappe.gzip_decompress` | The `gzip` module |
| `frappe.geo.country_info.get_translated_dict` | `frappe.geo.country_info.get_translated_countries` |
| `User.validate_roles` | `User.populate_role_profile_roles` |
| `frappe.utils.get_gravatar`, `get_gravatar_url`, `has_gravatar` | `frappe.utils.get_identicon`. Gravatar is removed. |
| `frappe.utils.error.get_default_args`, `raise_error_on_no_output` | No replacement |
| `frappe.utils.make_esc`, `frappe.client.get_js`, `frappe.utils.print_format.read_multi_pdf` | No replacement |

`frappe.utils.data.get_number_format_info` is marked for v16. Use `NumberFormat.from_string()` from `frappe.utils.number_format`.

In your own app, mark a public function as deprecated with `frappe.deprecation_dumpster.deprecation_warning(marked, graduation, msg)`. Keep it for one major version, then remove it.

## Related

- [Installation](../../getting-started/installation.md)
- [Fixtures and Patches](../building-apps/fixtures-and-patches.md)
- [Deployment](deployment.md)
- [Workspaces, Sidebar and Dock](../desk-ui/workspaces-sidebar-dock.md)
- [Testing](../testing-and-performance/testing.md)
- [Release changes](../../changes.md)
