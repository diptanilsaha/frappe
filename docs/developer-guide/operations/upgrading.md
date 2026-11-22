---
title: Upgrading
description: Notable breaking changes from v15 to v16 and from v16 to develop, how deprecations work in deprecation_dumpster, and the steps to upgrade a bench.
version: develop
sources:
  - frappe/deprecation_dumpster.py
  - frappe/hooks.py
  - pyproject.toml
  - package.json
  - frappe/model/qb_query.py
  - frappe/commands/site.py
  - frappe/commands/utils.py
  - frappe/commands/testing.py
  - frappe/desk/doctype/sidebar/convert_fixtures.py
  - frappe/model/sync.py
  - frappe/patches/v16_0/switch_default_sort_order.py
  - frappe/patches/v16_0/hash_session_ids.py
  - frappe/patches/v16_0/merge_studio_into_frappe.py
  - frappe/patches/v16_0/rename_slack_webhook_url.py
  - frappe/patches/v16_0/migrate_slack_notifications.py
  - frappe/patches/v16_0/hash_oauth_bearer_tokens.py
  - frappe/patches/v17_0/switch_wkhtmltopdf_to_chrome.py
  - frappe/patches/v17_0/keep_existing_sites_on_classic_charts.py
---

# Upgrading

This page lists changes that can break an app or a site when you move from v15 to v16, and from v16 to develop (17.0.0-dev). It lists only changes found by comparing the code of the three versions. Read it before you upgrade, and test the upgrade on a copy of the site.

## Upgrade steps

1. Take a backup with files: `bench --site <site> backup --with-files`. Keep the `encryption_key` from `site_config.json`.
2. Check the Python and Node versions in the table below. Install the new versions first.
3. Update each app to the matching branch (`version-16` or `develop`). All apps on a bench must support the same Frappe version.
4. Run `bench setup requirements` to install Python and Node dependencies.
5. Run `bench --site <site> migrate` for each site.
6. Run `bench build`.
7. Restart the processes: `bench restart` in production, or start `bench start` again in development.

In production, turn on maintenance mode before step 5 with `bench --site <site> set-maintenance-mode on`, and turn it off after step 7. `bench update` from the bench CLI runs the pull, requirements, migrate, build and restart steps for you.

## Requirements

| | v15 | v16 | develop |
|---|---|---|---|
| Python (`requires-python`) | `>=3.10,<3.15` | `>=3.14,<3.15` | `>=3.14,<3.15` |
| Node (`engines.node`) | `>=18` | `>=24` | `>=24` |

v16 needs Python 3.14. A v15 bench on an older Python needs a new virtual environment: `bench setup env --python python3.14`, then `bench setup requirements`.

## v15 to v16

### Desk route is /desk

The desk moved from `/app` to `/desk`. Frappe redirects `/app/...` to `/desk/...` with the query string (`website_redirects` in `frappe/hooks.py`). Update hardcoded links in apps, emails, print formats and external systems. In JavaScript, use `frappe.set_route` and `frappe.utils.get_form_link` instead of building `/app/` URLs.

### Modules moved to separate apps

These DocTypes are removed from frappe. Install the app to keep using them.

| Feature | DocTypes removed | App |
|---|---|---|
| Blog | Blog Post, Blog Category, Blogger, Blog Settings | `blogs` |
| Newsletter | Newsletter, Newsletter Attachment, Newsletter Email Group | `newsletter` |
| Energy Points | Energy Point Log, Energy Point Rule, Energy Point Settings, Review Level | `eps` |
| Offsite backups | Dropbox Settings, S3 Backup Settings, Google Drive | `offsite_backups` |

Also removed: Changelog Feed, Transaction Log, Marketing Campaign. The hooks `leaderboards` and `get_changelog_feed` are no longer read.

### get_list rejects SQL in fields

`frappe.get_list` and `frappe.get_all` now build queries with the query builder (`frappe.model.qb_query`). SQL functions written as strings in `fields` raise `ValidationError` ("SQL functions are not allowed as strings in SELECT"). Use the dict form:

```python
# v15
frappe.get_all("ToDo", fields=["count(name) as count"])

# v16 and develop
frappe.get_all("ToDo", fields=[{"COUNT": "name", "as": "count"}])
```

For other expressions, write a query with `frappe.qb`.

### Default sort is creation

The default `sort_field` of a new DocType is `creation`, not `modified`. A v16 patch (MariaDB only) adds an index on `creation` to the tables of all parent DocTypes, drops the `modified` index where the DocType does not sort by `modified`, and changes saved list settings from `modified` to `creation`. Code that expects lists sorted by last change must pass `order_by="modified desc"`.

### Sidebar fixtures

The workspace sidebar of an app moved from `<app>/workspace_sidebar/*.json` to one **Sidebar** per module, in `<app>/<module>/sidebar/`. The old folder is not imported. Until you convert, a module shows a generated sidebar. Convert the files once and commit the result:

```bash
bench --site <site> convert-sidebar-fixtures --app sample_app --dry-run
bench --site <site> convert-sidebar-fixtures --app sample_app
```

The command only reads the old folder and leaves it in place. Without `--app`, it converts every installed app that still has the old folder.

### Other changes

- Session IDs are stored as SHA-256 hashes. A patch hashes existing sessions, so users stay logged in. Code that reads `tabSessions` by the raw cookie value no longer finds the row.
- Studio is part of frappe. A site that had the separate `studio` app gets it from frappe after migrate, and `studio` is removed from the installed apps.
- `frappe.get_version` is removed.
- Test classes: use `frappe.tests.UnitTestCase` and `frappe.tests.IntegrationTestCase`. `frappe.tests.utils.FrappeTestCase` still works with a warning.

## v16 to develop

- **Website search index removed.** `frappe/search/website_search.py` and `full_text_search.py` are gone. `bench migrate --skip-search-index` has no effect and shows a warning. `bench build-search-index` does nothing. Remove them from scripts.
- **Slack Webhook URL renamed.** The DocType is now `Notification Webhook URL`. Notifications with channel `Slack` change to channel `Webhook`. Update code and fixtures that use the old names.
- **Sidebar Item Group removed.** The DocTypes `Sidebar Item Group` and `Sidebar Item Group Link` no longer exist.
- **PDF generator.** A patch sets `pdf_generator` to `chrome` for Print Formats that used `wkhtmltopdf` or had no value, and removes the site-wide `pdf_generator` of Print Settings. Check PDF output of custom print formats after upgrade.
- **OAuth bearer tokens** are stored as SHA-256 hashes. Existing tokens are hashed by a patch.
- **Classic charts.** Sites that finished setup keep the classic charts (`use_classic_charts` in System Settings). New sites get the new charts.
- **UI tests.** `bench run-ui-tests` runs Playwright. Apps with only a Cypress config still run with a warning. Cypress support is deprecated for removal in v17.
- **`frappe.read_doc`** is new. Use it to load a document for a user, with permission checks and field-level read permissions.
- **`rate_limit` in `@frappe.whitelist`.** The parameter only accepts `None`, to mark a guest method that is not rate limited. Any other value raises `TypeError`. Use the `@rate_limit` decorator to limit a method.
- **Server Script variables.** Server Scripts have a **Variables** table, read with `frappe.env`. API scripts with `allow_guest` must have a rate limit.
- **Removed deprecated functions.** `frappe.utils.error.get_default_args` and `frappe.utils.error.raise_error_on_no_output` are removed.

## Deprecations

Frappe keeps deprecated functions in `frappe/deprecation_dumpster.py`. Each one has the date it was marked and the version from which it is an error.

| Warning class | Behavior |
|---|---|
| `V15FrappeDeprecationWarning` | An error (subclass of `FrappeDeprecationError`). |
| `V16FrappeDeprecationWarning` | A warning. |
| `V17FrappeDeprecationWarning` | Ignored by default (`PendingFrappeDeprecationWarning`). |

To see pending warnings, or to turn an error back into a warning in an emergency, set `PYTHONWARNINGS`, for example `PYTHONWARNINGS="default::frappe.deprecation_dumpster.V17FrappeDeprecationWarning"`.

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
