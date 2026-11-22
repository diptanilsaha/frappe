---
title: Testing API
description: Test classes, assertions, context managers, test record helpers and API test helpers in frappe.tests and frappe.test_runner, with their exact signatures and behavior.
version: v15
sources:
  - frappe/tests/__init__.py
  - frappe/tests/utils.py
  - frappe/tests/test_api.py
  - frappe/tests/ui_test_helpers.py
  - frappe/test_runner.py
  - frappe/commands/utils.py
  - frappe/__init__.py
---

# Testing API

This page lists the public classes and helpers for tests in Frappe version 15. Use it to look up a signature or a side effect. For how to write and run tests, read [Testing](../../developer-guide/testing-and-performance/testing.md). For `bench run-tests` options, read [CLI: run-tests](../configuration/cli.md#run-tests).

The base class and the helpers are in `frappe.tests.utils`. The test records helpers are in `frappe.test_runner`.

```python
from frappe.tests.utils import FrappeTestCase, change_settings, patch_hooks, timeout
```

!!! note "Version"
    Version 16 adds `frappe.tests.UnitTestCase` and `frappe.tests.IntegrationTestCase`, and deprecates `FrappeTestCase`. Version 15 has only `FrappeTestCase`.

## Test classes

| Class | Base | Description |
|---|---|---|
| `frappe.tests.utils.FrappeTestCase` | `unittest.TestCase` | Commits pending changes before the class, and rolls back at the end of the class. |
| `frappe.tests.utils.MockedRequestTestCase` | `FrappeTestCase` | Starts a `responses.RequestsMock` in each `setUp` as `self.responses`. Outgoing `requests` calls fail unless you register a mock. |
| `frappe.tests.test_api.FrappeAPITestCase` | `FrappeTestCase` | HTTP helpers for REST tests. It is in a test module. |

If you override `setUpClass`, call `super().setUpClass()`.

### FrappeTestCase

| Member | Description |
|---|---|
| `setUpClass()` | Sets `TEST_SITE` and `ADMIN_PASSWORD`, keeps the current connection as the primary connection, and commits. Adds class cleanups that roll back and reset `frappe.local`. It does not make test records and does not change the user. |
| Class cleanup | Clears `frappe.db.value_cache`, rolls back, and resets `frappe.local` flags, `error_log`, `message_log`, `debug_log`, `conf`, `cache`, `lang` and `preload_assets`. Removes `frappe.local.request`. |
| `TEST_SITE` | `"test_site"`. Replaced by `frappe.local.site` when set. |
| `ADMIN_PASSWORD` | `admin_password` from the site config. |
| `SHOW_TRANSACTION_COMMIT_WARNINGS` | `False`. Set `True` to print a stack trace on each commit. |
| `maxDiff` | `10_000`. |
| `assertDocumentEqual(expected, actual)` | Compare only the fields in `expected` (a dict or a Document). Child tables must have the same length. Floats use the field precision. Integers and booleans are compared with `cint`. Dates and times are compared as strings. |
| `assertQueryEqual(first: str, second: str)` | Compare SQL after `normalize_sql`. |
| `assertSequenceSubset(larger: Sequence, smaller: Sequence, msg=None)` | Assert that `smaller` is a subset of `larger`. |
| `assertQueryCount(count)` | Context manager. Fails if more than `count` queries run through `frappe.db.sql`. Lists the queries on failure. |
| `assertRowsRead(count)` | Context manager. Fails if queries touch more than `count` rows (cursor `rowcount`). |
| `assertRedisCallCounts(count)` | Context manager. Fails if more than `count` Redis commands run. |
| `normalize_html(code: str) -> str` | Format HTML with BeautifulSoup for string comparison. |
| `normalize_sql(query: str) -> str` | Format SQL with `sqlparse`: upper-case keywords, reindent, strip comments. |
| `primary_connection()`, `secondary_connection()` | Context managers. Switch `frappe.local.db` between two connections to test locks and concurrency. After `secondary_connection` is used, both connections roll back at the end of the test. |
| `set_user(user: str)` | Context manager. Call `frappe.set_user(user)`, then switch back to the previous user. |
| `switch_site(site: str)` | Context manager. Connect to another site for the block, then connect back. Drops the current connection. |
| `freeze_time(time_to_freeze, *args, **kwargs)` | Context manager. Freeze the clock with `freezegun`. The time is read in the system time zone. |
| `enable_safe_exec()` | Class method. Set `server_script_enabled` to 1 in `common_site_config.json`. Sets it to 0 at the end of the class. Writes the file. |

Rollback happens once, at the end of the class. Records that one test writes are visible to the next tests of the same class.

```python
# sample_app/sample_app/doctype/sample_order/test_sample_order.py
import frappe
from frappe.tests.utils import FrappeTestCase, change_settings


class TestSampleOrder(FrappeTestCase):
	def test_total(self):
		order = frappe.get_doc({
			"doctype": "Sample Order",
			"customer_name": "Customer A",
			"order_date": "2026-01-15",
			"items": [{"item": "_Test Sample Item", "qty": 2}],
		}).insert()
		self.assertDocumentEqual({"customer_name": "Customer A", "items": [{"qty": 2}]}, order)

	def test_query_budget(self):
		with self.assertQueryCount(1):
			frappe.db.get_value("Sample Item", "_Test Sample Item", "rate")

	@change_settings("System Settings", {"float_precision": 4})
	def test_precision(self):
		...

	def test_user_cannot_delete(self):
		with self.set_user("user3@example.com"):
			self.assertRaises(frappe.PermissionError, frappe.delete_doc, "Role", "Sample Role")
```

### FrappeAPITestCase

`frappe.tests.test_api.FrappeAPITestCase` sends requests to the site through a Werkzeug test client in another thread.

| Member | Description |
|---|---|
| `version` | `""` (v1), `"v1"` or `"v2"`. Picks the path prefix. |
| `TEST_CLIENT` | Werkzeug test client from `frappe.utils.get_test_client()`. |
| `site_url` | Property. `get_url()`. |
| `sid` | Cached property. Logs in as Administrator and returns the sid. Pass it as a cookie or `?sid=`. |
| `resource(*parts)`, `method(*parts)`, `doctype_path(*parts)`, `get_path(*parts)` | Build URLs such as `/api/resource/...` or `/api/v2/document/...`. |
| `get(path, params=None, **kwargs)`, `post(path, data, **kwargs)`, `put(path, data, **kwargs)`, `patch(path, data, **kwargs)`, `delete(path, **kwargs)` | Send a request. `params` and `data` go in a JSON body. Return a `TestResponse`. |
| `tearDown()` | Rolls back after each test. |

The module also has `make_request(target, args=None, kwargs=None, site=None)`, which runs a request in a `ThreadWithReturnValue`. When `frappe.tests.test_api.authorization_token` is set, the thread sends `Authorization: token <value>`.

## Context managers and decorators

All in `frappe.tests.utils`.

| Name | Description |
|---|---|
| `change_settings(doctype, settings_dict=None, /, commit=False, **settings)` | Context manager and decorator. Set fields on a Single DocType, save with `ignore_permissions`, and restore the old values after the block. With `commit=True`, commits both saves. When `settings_dict` is given, `**settings` is ignored. |
| `patch_hooks(overridden_hoooks)` | Context manager. Make `frappe.get_hooks(hook)` return the given value for the listed hooks. The parameter name has this spelling in the code. |
| `timeout(seconds=30, error_message="Test timed out.")` | Decorator. Raise `Exception(error_message)` after `seconds`. Works as `@timeout` (message `"Test timed out"`) or `@timeout(60)`. Uses `SIGALRM`: Unix and the main thread only. Not a context manager. |
| `check_orpahned_doctypes()` | Raise if a standard DocType in the database has no controller. The name has this spelling in the code. |

## Test records

These functions are in `frappe.test_runner`. `bench run-tests --doctype` calls `make_test_records` with `commit=True` before it runs the tests. `--module` makes records for the `test_dependencies` of the module. `--app` also makes records for the DocType of each test file in a DocType folder.

| Function | Description |
|---|---|
| `make_test_records(doctype, verbose=0, force=False, commit=False)` | Make test records for each DocType that `get_dependencies` returns, depth first. Does nothing when `frappe.flags.skip_test_records` is set. |
| `make_test_records_for_doctype(doctype, verbose=0, force=False, commit=False)` | Make records for one DocType. Skips a DocType in the test record log unless `force=True`. |
| `make_test_objects(doctype, test_records=None, verbose=None, reset=False, commit=False)` | Insert the given records, or the records of `test_records.json`. Submits a record with `docstatus` 1. Returns names. Skips a record that exists unless `reset=True`. |
| `get_dependencies(doctype)` | Sorted list of the DocType, its Link targets (also in child tables) and `test_dependencies`, without `test_ignore`. |
| `get_modules(doctype)` | `(module, test_module)` of a DocType. Reloads the test module. |
| `print_mandatory_fields(doctype)` | Print the autoname and the mandatory fields of a DocType that has no test records. |
| `add_to_test_record_log(doctype)`, `get_test_record_log()` | Write and read `sites/<site>/.test_log`, the list of DocTypes that already have test records. |
| `frappe.get_test_records(doctype)` | Return the list in `test_records.json` of the DocType folder, or `[]`. |

Where the records come from, for each DocType:

1. `_make_test_records(verbose)` in the test module. It must return a list of names.
2. `test_records` in the test module.
3. `test_records.json` in the DocType folder.

Test module variables: `test_dependencies` (DocTypes to add) and `test_ignore` (DocTypes to skip).

!!! warning
    The test record log lists DocTypes, not records. If you delete test records, the next run does not make them again. Delete `sites/<site>/.test_log`, or pass `--force`.

## Other helpers

| Name | Description |
|---|---|
| `frappe.tests.ui_test_helpers.whitelist_for_tests(fn=None, methods=None)` | Decorator. Same as `@frappe.whitelist(methods=methods)`. When the module is imported in a web request that is not a test and not `bench start`, it raises. |
| `frappe.tests.global_test_dependencies` | `["User"]`. |
| `frappe.tests.update_system_settings(args, commit=False)` | Update System Settings with `ignore_mandatory` and save. Commits with `commit=True`. |
| `frappe.tests.get_system_setting(key)` | `frappe.db.get_single_value("System Settings", key)`. |

## Pitfalls

- `bench run-tests` runs only when the site config has `allow_tests` or the `CI` environment variable is set.
- Test records are committed. They stay in the site database after the run.
- `bench run-tests --doctype <name> --force` deletes every record of the DocType before it makes test records.
- `make_test_objects` calls `frappe.db.rollback()` when a record exists and `reset` is false. This also rolls back other uncommitted changes.
- A test that commits (`frappe.db.commit()`) breaks the class rollback. Set `SHOW_TRANSACTION_COMMIT_WARNINGS = True` to find commits.
- `enable_safe_exec` writes `common_site_config.json` and sets the key to 0 afterwards, also if it was 1 before.
- `change_settings` restores values with a second save. Without `commit=True`, both saves roll back with the class.
- `FrappeAPITestCase.get(path, params)` sends `params` as a JSON body, not as a query string.

## Related

- [Testing](../../developer-guide/testing-and-performance/testing.md)
- [CLI](../configuration/cli.md)
- [Performance](../../developer-guide/testing-and-performance/performance.md)
- [Jobs, Cache and Realtime API](jobs-cache-and-realtime.md)
