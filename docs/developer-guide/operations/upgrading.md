---
title: Upgrading
description: Upgrade a bench to Frappe v15 from v14, what the v15 patches change on a site, deprecation warnings in v15, and what to prepare for v16.
version: v15
sources:
  - frappe/patches.txt
  - frappe/patches/v15_0/remove_implicit_primary_key.py
  - frappe/patches/v15_0/remove_event_streaming.py
  - frappe/patches/v15_0/drop_modified_index.py
  - frappe/patches/v15_0/remove_prepared_report_settings_from_system_settings.py
  - frappe/patches/v15_0/copy_disable_prepared_report_to_prepared_report.py
  - frappe/patches/v15_0/move_event_cancelled_to_status.py
  - frappe/patches/v15_0/set_file_type.py
  - frappe/patches/v15_0/disable_default_website_signups.py
  - frappe/patches/v15_0/remove_background_jobs_from_dropdown.py
  - frappe/patches/v15_0/remove_workspace_settings_navbar_items.py
  - frappe/patches/v15_0/validate_newsletter_recipients.py
  - frappe/patches/v15_0/set_snapshot_report_frequency_to_weekly.py
  - frappe/patches/v15_0/set_contact_full_name.py
  - frappe/patches/v15_0/sanitize_workspace_titles.py
  - frappe/model/document.py
  - frappe/core/doctype/user/user.py
  - frappe/geo/country_info.py
  - frappe/client.py
  - frappe/utils/print_format.py
  - frappe/utils/deprecations.py
  - frappe/utils/redis_wrapper.py
  - frappe/utils/__init__.py
  - frappe/app.py
  - frappe/handler.py
  - frappe/database/database.py
  - frappe/utils/background_jobs.py
  - pyproject.toml
  - package.json
---

# Upgrading

This page shows how to upgrade a bench to v15 from v14, and what the v15 patches change on a site. It lists only changes that the v15 code and `frappe/patches.txt` show. The last section lists changes to prepare for before you move to v16. Read it before you upgrade, and test the upgrade on a copy of the site.

## Upgrade steps

1. Take a backup with files: `bench --site <site> backup --with-files`. Keep the `encryption_key` from `site_config.json`.
2. Check the Python and Node versions below. Install the new versions first.
3. Update each app to the `version-15` branch. All apps on a bench must support the same Frappe version.
4. Run `bench setup requirements` to install Python and Node dependencies.
5. Run `bench --site <site> migrate` for each site.
6. Run `bench build`.
7. Restart the processes: `bench restart` in production, or start `bench start` again in development.

In production, turn on maintenance mode before step 5 with `bench --site <site> set-maintenance-mode on`, and turn it off after step 7. In v15, maintenance mode blocks all web requests unless `allow_reads_during_maintenance` is set. `bench update` from the bench CLI runs the pull, requirements, migrate, build and restart steps for you. See [Deployment](deployment.md).

## Requirements

| | v15 |
|---|---|
| Python (`requires-python` in `pyproject.toml`) | `>=3.10,<3.15` |
| Node (`engines.node` in `package.json`) | `>=18` |

If the bench runs an older Python, make a new virtual environment: `bench setup env --python python3.11`, then `bench setup requirements`.

## Upgrading to v15

The `v15_0` patches in `frappe/patches.txt` make these changes when you run `bench migrate`:

| Change | Patch |
|---|---|
| Event Streaming is removed from frappe. If the `event_streaming` app is not installed, the patch deletes the Event Streaming module and its DocTypes (Event Producer, Event Consumer, Event Update Log, Event Sync Log and others). Install the `event_streaming` app before you migrate to keep them. | `remove_event_streaming` |
| Log tables that had an integer `name` column without `autoincrement` (for example Error Log, Version, Access Log, Email Queue) get a `varchar` `name` column. | `remove_implicit_primary_key` |
| On MariaDB, the `modified` index is dropped from all child tables. | `drop_modified_index` |
| The expiry of Prepared Reports moves from System Settings (`prepared_report_expiry_period`) to Log Settings. | `remove_prepared_report_settings_from_system_settings` |
| Reports with `disable_prepared_report` set get `prepared_report` 0 and `disable_prepared_report_automation` 1. | `copy_disable_prepared_report_to_prepared_report` |
| Events with Event Type `Cancelled` get Event Type `Private` and Status `Cancelled`. | `move_event_cancelled_to_status` |
| Files without `file_type` get it from the file extension. | `set_file_type` |
| Contacts without `full_name` get it from their name parts. | `set_contact_full_name` |
| Email Group Members with an invalid email are set to unsubscribed. | `validate_newsletter_recipients` |
| HTML is removed from Workspace titles. | `sanitize_workspace_titles` |
| The "Background Jobs" and "Workspace Settings" items are removed from the navbar menu. | `remove_background_jobs_from_dropdown`, `remove_workspace_settings_navbar_items` |
| If website signups were on only from the old default, the patch turns on **Disable Signups** in Website Settings and notifies System Managers. | `disable_default_website_signups` |
| The snapshot report frequency in System Settings is set to `Weekly`. | `set_snapshot_report_frequency_to_weekly` |

### Code changes to check in your app

- `frappe.cache` is an object. Call its methods directly, for example `frappe.cache.get_value(key)`. `frappe.cache()` still works for backward compatibility (`__call__` in `frappe/utils/redis_wrapper.py`).
- Gravatar support is removed. `frappe.utils.has_gravatar` and `frappe.utils.get_gravatar_url` are deprecated and return an empty string.
- Test classes extend `frappe.tests.utils.FrappeTestCase`. See [Testing](../testing-and-performance/testing.md).
- If your app has Report records with `disable_prepared_report` in fixtures, change them to `prepared_report` and `disable_prepared_report_automation`.

### Deprecation warnings in v15

v15 shows a `DeprecationWarning` (through `frappe.utils.deprecations.deprecation_warning`) for these calls. They still work in v15.

| Deprecated | Use instead |
|---|---|
| Sending `cmd` in the form data of a request | `/api/method/<dotted.path>` |
| A short method name in an RPC call (no dot) | The full dotted path |
| `frappe.db.set_value` on a Single DocType | `frappe.db.set_single_value` |
| `frappe.enqueue(..., job_name=...)` | `job_id` |
| `frappe.enqueue(..., is_async=False)` outside tests | `now=True` |
| `uploadfile` method | `upload_file`. The code says `uploadfile` is removed in v16. |
| `super(Document, self).__init__(d)` in a controller | `self.init_from_dict(d)` |
| `frappe.db.is_column_missing` | `frappe.db.is_missing_column` |
| `frappe.utils.gzip_compress`, `gzip_decompress` | The `gzip` module |
| `User.validate_roles` | `User.populate_role_profile_roles` |
| `frappe.geo.country_info.get_translated_dict` | `get_translated_countries` |
| `frappe.client.get_js`, `frappe.utils.print_format.read_multi_pdf` | No replacement |

To see the warnings in a run, set `PYTHONWARNINGS=default::DeprecationWarning`.

## Preparing for v16

These changes come from a comparison of the v15 code with the v16 code. Check them before you upgrade a v15 bench to v16.

- **Python 3.14 and Node 24.** v16 needs Python `>=3.14,<3.15` and Node `>=24`. Make a new virtual environment with `bench setup env --python python3.14`.
- **Desk route.** The desk moves from `/app` to `/desk`. v16 redirects `/app/...` to `/desk/...`. Use `frappe.set_route` and `frappe.utils.get_url_to_form` instead of building `/app/` URLs.
- **Modules move to separate apps.** Blog (Blog Post, Blog Category, Blogger, Blog Settings), Newsletter, Energy Points, and the Dropbox, S3 and Google Drive backups are not in v16 frappe. Install `blogs`, `newsletter`, `eps` or `offsite_backups` to keep them. The `leaderboards` hook is not read in v16.
- **SQL strings in `fields`.** v16 `frappe.get_list` and `frappe.get_all` raise `ValidationError` for SQL functions written as strings, for example `"count(name) as c"`. Use the dict form `{"COUNT": "name", "as": "c"}`, which does not work in v15.
- **Default sort.** New DocTypes sort by `creation` in v16. A v16 patch adds a `creation` index to all parent tables (MariaDB) and changes saved list sorts. Pass `order_by="modified desc"` where you need the last change first.
- **Session IDs** are stored as SHA-256 hashes. Code that reads `tabSessions` by the raw cookie value stops working.
- **`frappe.get_version`** is removed in v16.
- **Test classes.** v16 has `frappe.tests.UnitTestCase` and `frappe.tests.IntegrationTestCase`. `FrappeTestCase` still works with a deprecation warning.
- **Deprecations.** v16 moves deprecated functions to `frappe/deprecation_dumpster.py`, with a version after which each one is an error.

## Related

- [Installation](../../getting-started/installation.md)
- [Fixtures and Patches](../building-apps/fixtures-and-patches.md)
- [Deployment](deployment.md)
- [Testing](../testing-and-performance/testing.md)
- [Release changes](../../changes.md)
