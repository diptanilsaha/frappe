---
title: Fixtures and Patches
description: Ship records with an app through the fixtures hook, change data on existing sites with patches, and understand the order of bench migrate.
version: v16
sources:
  - frappe/utils/fixtures.py
  - frappe/migrate.py
  - frappe/modules/patch_handler.py
  - frappe/modules/import_file.py
  - frappe/modules/utils.py
  - frappe/core/doctype/data_import/data_import.py
  - frappe/commands/site.py
  - frappe/commands/utils.py
---

# Fixtures and Patches

Fixtures and patches move data with your app. Use **fixtures** for records that must be the same on every site, such as Custom Fields, Property Setters, Roles or Workflows. Use **patches** for one-time changes to the data of sites that already exist, such as a field rename or a backfill. Both run during `bench migrate`.

## Fixtures

### Declare fixtures

Add the `fixtures` hook to `hooks.py`. Each item is a DocType name or a dict with filters.

```python
# sample_app/hooks.py
fixtures = [
	"Sample Role",
	{"dt": "Custom Field", "filters": [["module", "=", "Sample App"]]},
	{"dt": "Property Setter", "filters": [["module", "=", "Sample App"]]},
	{"doctype": "Workflow", "or_filters": [["name", "=", "Sample Order Approval"]], "prefix": "approval"},
]
```

| Key | Type | Description |
|---|---|---|
| `dt` or `doctype` | `str` | The DocType to export. |
| `filters` | list or dict | Filters for `frappe.get_all`. Exports only matching records. |
| `or_filters` | list or dict | OR filters for `frappe.get_all`. |
| `prefix` | `str` | Adds `<prefix>_` to the file name. Use it to export one DocType into more than one file. |

Without filters, every record of the DocType is exported. Filter Custom Fields and Property Setters by `module`, or you export the customizations of all apps.

`DocType` and `Page` cannot be fixtures. `export-fixtures` stops with an error for these. Create them in developer mode, and Frappe writes them to the app as files.

### Export fixtures

Run this on a site that has the records:

```bash
bench --site frappe.localhost export-fixtures --app sample_app
```

Frappe writes one JSON file per fixture to `sample_app/fixtures/<scrubbed_doctype>.json` (for example `custom_field.json`). It removes `owner`, `creation`, `modified_by`, `idx`, `lft` and `rgt` from each record. Without `--app`, Frappe exports the fixtures of all installed apps.

To control the import order, set `fixture_auto_order = True` in `hooks.py`. Frappe then adds a number to each file name, in the order of the `fixtures` list, for example `1_sample_role.json`.

### Import fixtures

`bench migrate` and `bench install-app` call `sync_fixtures`. It imports every `.json` file in `<app>/fixtures/`, sorted by file name.

- An existing record with the same name is deleted and inserted again. Changes made on the site to a fixture record are lost on the next migrate.
- A record that you remove from the JSON file is not deleted from sites.
- If the DocType of a file does not exist, Frappe prints "Skipping fixture syncing" and goes on.
- Emails are muted during the import. Each file is committed.
- `bench migrate --skip-fixtures` skips this step.

!!! warning
    `<app>/fixtures/custom_scripts/*.js` files are not imported. Frappe prints an error for each one. Export Client Scripts as a `Client Script` fixture instead.

### Fixtures or exported customizations

For Custom Fields and Property Setters you can also use **Export Customizations** in Customize Form. That writes `<app>/<module>/custom/<doctype>.json`. Files with `"sync_on_migrate": 1` sync on every migrate. See [Extending Other Apps](extending-apps.md).

## Patches

A patch is a Python module with an `execute()` function. Frappe runs each patch once per site and records it in **Patch Log**.

### Write a patch

```python
# sample_app/patches/v1_0/set_default_category.py
import frappe


def execute():
	"""Set category of old Sample Items to Hardware."""
	frappe.db.set_value(
		"Sample Item",
		{"category": ("is", "not set")},
		"category",
		"Hardware",
		update_modified=False,
	)
```

- Frappe prints the docstring when the patch runs.
- Frappe commits before and after the patch. If the patch raises an error, Frappe rolls back and stops the migrate.
- During a patch `frappe.flags.in_patch` is set and `developer_mode` is off. Files are not written back to the app.
- Server Script DocType events do not run during a migrate.

### Register the patch

Add the dotted path to `<app>/patches.txt`. Use the INI format with sections:

```ini
# sample_app/patches.txt
[pre_model_sync]
sample_app.patches.v1_0.rename_old_field

[post_model_sync]
sample_app.patches.v1_0.set_default_category
execute:frappe.db.set_single_value("System Settings", "enable_onboarding", 0)
```

| Section | Runs | Use for |
|---|---|---|
| `[pre_model_sync]` | Before DocType JSON is synced | Patches that need the old schema, for example to copy data out of a column that the new JSON removes. |
| `[post_model_sync]` | After DocType JSON is synced | Most patches. New fields and DocTypes exist. |

- A file without section headers is read in the old format. All lines are `pre_model_sync` patches.
- Both sections are required in INI format.
- Patches run before fixtures and exported customizations are synced. A patch cannot use Custom Fields or records that come from fixtures of the same migrate. Use an `after_migrate` hook for that work.
- A line that starts with `execute:` runs the Python code after the prefix.
- A line that starts with `finally:` runs after all other patches of the same pass.
- To run a patch again on all sites, change its line, for example add a comment: `sample_app.patches.v1_0.set_default_category #2`. Patch Log stores the full line.

### Run patches

```bash
bench --site frappe.localhost migrate
bench --site frappe.localhost run-patch sample_app.patches.v1_0.set_default_category
bench --site frappe.localhost run-patch --force sample_app.patches.v1_0.set_default_category
```

`run-patch` runs one patch if it is not in Patch Log. `--force` runs it again.

`bench migrate --skip-failing` writes a failing patch to Patch Log with `skipped` set and goes on. Frappe tries the patch again on the next migrate. To stop the migrate for a patch even with `--skip-failing`, list it in the `never_skip_patches` hook:

```python
# sample_app/hooks.py
never_skip_patches = ["sample_app.patches.v1_0.critical_data_fix"]
```

## Order of bench migrate

`SiteMigration` in `frappe/migrate.py` runs these steps for each site. It takes a file lock, so only one migrate runs at a time.

1. Check that Redis cache is running. If not, stop.
2. Register new modules of installed apps.
3. Run `before_migrate` hooks of each app.
4. Run `[pre_model_sync]` patches.
5. Sync DocType JSON and other standard files of all apps (`frappe.model.sync.sync_all`).
6. Run `[post_model_sync]` patches.
7. Sync Scheduled Job Types from `scheduler_events`.
8. Sync fixtures (unless `--skip-fixtures`).
9. Sync dashboards and exported customizations.
10. Sync languages, flush deferred inserts, remove orphan DocTypes.
11. Sync the portal menu and update Installed Applications.
12. Run `after_migrate` hooks of each app.

Steps 2 to 3, 4 to 6 and 7 to 12 each commit at the end, or roll back on error. After the migrate, Frappe clears the translation and website caches. It queues a rebuild of the website search index on the `long` queue, unless you pass `--skip-search-index`. Then it sends a `version-update` realtime event to open desks.

```python
# sample_app/hooks.py
before_migrate = "sample_app.migrate.before_migrate"
after_migrate = ["sample_app.setup.make_custom_fields"]
```

## Pitfalls

- Do not call `frappe.get_doc` and `doc.save()` on many records in a patch. Use `frappe.db.set_value` with filters, `frappe.qb.update` or `frappe.db.bulk_update`.
- A patch in `[pre_model_sync]` cannot use new fields. Put patches that read old data there, and check the column with `frappe.db.has_column` first, because new sites never had it.
- Fixture records are replaced on every migrate. Do not use fixtures for records that users edit.
- A `prefix` in a dict fixture is also used for the next plain string fixtures in the list (`frappe/utils/fixtures.py` does not reset it). Put plain string fixtures before dict fixtures with a `prefix`, or give every fixture a dict.

## Related

- [Apps](apps.md)
- [Hooks](hooks.md)
- [Extending Other Apps](extending-apps.md)
- [Upgrading](../operations/upgrading.md)
- [CLI reference](../../reference/configuration/cli.md)
