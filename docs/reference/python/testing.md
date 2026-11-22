---
title: Testing API
description: Test classes, assertions, context managers, test record helpers and API test helpers in frappe.tests, with their exact signatures and behavior.
version: develop
sources:
  - frappe/tests/__init__.py
  - frappe/tests/classes/unit_test_case.py
  - frappe/tests/classes/integration_test_case.py
  - frappe/tests/classes/mocked_request_test_case.py
  - frappe/tests/classes/context_managers.py
  - frappe/tests/utils/__init__.py
  - frappe/tests/utils/generators.py
  - frappe/tests/utils/test_capabilities.py
  - frappe/tests/test_api.py
  - frappe/deprecation_dumpster.py
---

# Testing API

This page lists the public classes and helpers in `frappe.tests`. Use it to look up a signature or a side effect. For how to write and run tests, read [Testing](../../developer-guide/testing-and-performance/testing.md). For `bench run-tests` options, read [CLI: run-tests](../configuration/cli.md#run-tests).

`frappe.tests` exports `UnitTestCase`, `IntegrationTestCase`, `MockedRequestTestCase` and the context managers below.

## Test classes

| Class | Base | Description |
|---|---|---|
| `UnitTestCase` | `unittest.TestCase` | For code that does not write to the database. Sets the user to `Administrator`. No rollback. |
| `IntegrationTestCase` | `UnitTestCase` | Connects to the site, makes test records, commits them, and rolls back at the end of the class. |
| `MockedRequestTestCase` | `IntegrationTestCase` | Starts a `responses.RequestsMock` in each `setUp` as `self.responses`. Outgoing `requests` calls fail unless you register a mock. |
| `frappe.tests.test_api.FrappeAPITestCase` | `IntegrationTestCase` | HTTP helpers for REST tests. It is in a test module, not in `frappe.tests`. |

If you override `setUpClass`, call `super().setUpClass()`.

### UnitTestCase

| Member | Description |
|---|---|
| `setUpClass()` | Sets `cls.doctype` from the test file folder (`.../doctype/<name>/test_<name>.py`), `cls.module`, and the user `Administrator`. Restores `Administrator` after the class. |
| `doctype` | DocType name read from the JSON file next to the test, or `None`. |
| `assertDocumentEqual(expected: dict \| Document, actual: Document) -> None` | Compare only the fields in `expected`. Child tables must have the same length. Floats use the field precision. Integers and booleans are compared with `cint`. Dates are compared as strings. |
| `assertQueryEqual(first: str, second: str) -> None` | Compare SQL after `normalize_sql`. |
| `assertSequenceSubset(larger: Sequence, smaller: Sequence, msg=None) -> None` | Assert that `smaller` is a subset of `larger`. |
| `normalize_html(code: str) -> str` | Static. Format HTML with BeautifulSoup for string comparison. |
| `normalize_sql(query: str) -> str` | Static. Format SQL with `sqlparse`. Backticks become double quotes on Postgres and SQLite. |
| `registerAs(_as)` | Class method. Used to attach the context managers to the class. |

### IntegrationTestCase

| Member | Description |
|---|---|
| `setUpClass()` | Calls `frappe.init` on the current site (or `TEST_SITE`). Makes test records for `cls.doctype` and its Link dependencies, or for `EXTRA_TEST_RECORD_DEPENDENCIES` when the module is not in a DocType folder. Commits. |
| `setUp()` | Starts a stack dump after 5 minutes (`faulthandler`) to find stuck tests. |
| Class cleanup | Clears `frappe.db.value_cache`, rolls back, and resets `frappe.local` flags, messages, conf, response, cache and `lang`. |
| `TEST_SITE` | `"test_site"`. Replaced by `frappe.local.site` when set. |
| `ADMIN_PASSWORD` | `admin_password` from the site config. |
| `globalTestRecords` | Read-only map `{doctype: [record, ...]}` of the test records loaded so far. |
| `SHOW_TRANSACTION_COMMIT_WARNINGS` | `False`. Set `True` to print a stack trace on each commit. |
| `maxDiff` | `10_000`. |
| `assertQueryCount(count: int, query_type: tuple[str] \| None = None)` | Context manager. Fails if more than `count` queries run through `frappe.db.sql`. `query_type` filters, for example `("select",)`. |
| `assertRowsRead(count: int)` | Context manager. Fails if queries touch more than `count` rows (cursor `rowcount`). |
| `assertRedisCallCounts(count: int, *, exact=False)` | Context manager. Fails if more than `count` Redis commands run. `exact=True` needs the exact number. |
| `primary_connection()`, `secondary_connection()` | Context managers. Switch `frappe.local.db` between two connections to test locks and concurrency. After `secondary_connection` is used, both connections roll back at the end of the test. |
| `change_settings(...)`, `switch_site(site)` | Context managers, registered on this class. See below. |

Rollback happens once, at the end of the class. Records that one test writes are visible to the next tests of the same class.

```python
# sample_app/sample_app/doctype/sample_order/test_sample_order.py
import frappe
from frappe.tests import IntegrationTestCase


class IntegrationTestSampleOrder(IntegrationTestCase):
	def test_total(self):
		order = frappe.get_doc({
			"doctype": "Sample Order",
			"customer_name": "Customer A",
			"items": [{"item": "_Test Sample Item", "qty": 2}],
		}).insert()
		self.assertDocumentEqual({"customer_name": "Customer A", "items": [{"qty": 2}]}, order)

	def test_query_budget(self):
		with self.assertQueryCount(5):
			frappe.get_cached_doc("Sample Item", "_Test Sample Item")
```

### FrappeAPITestCase

`frappe.tests.test_api.FrappeAPITestCase` sends requests to the site through a Werkzeug test client in another thread.

| Member | Description |
|---|---|
| `version` | `""` (v1), `"v1"` or `"v2"`. Picks the path prefix. |
| `site_url` | `get_url()`. |
| `sid` | Cached property. Logs in as Administrator and returns the sid. Pass it as a cookie or `?sid=`. |
| `resource(*parts)`, `method(*parts)`, `doctype_path(*parts)`, `get_path(*parts)` | Build URLs such as `/api/resource/...` or `/api/v2/document/...`. |
| `get(path, params=None, **kwargs)`, `post(path, data, **kwargs)`, `put(path, data, **kwargs)`, `patch(path, data, **kwargs)`, `delete(path, **kwargs)`, `query(path, data, **kwargs)` | Send a request with a JSON body. Return a `TestResponse`. |
| `tearDown()` | Rolls back after each test. |

On SQLite, `make_request` commits before the request so the request thread can write.

## Context managers

Each one is a context manager and also works as a decorator. Import them from `frappe.tests`. Most are also attributes of `UnitTestCase` (`self.set_user(...)`). `change_settings` and `switch_site` are attributes of `IntegrationTestCase` only. `timeout` is not a class attribute.

| Name | Description |
|---|---|
| `set_user(user: str)` | Call `frappe.set_user(user)`, then switch back to the previous user. |
| `change_settings(doctype, settings_dict=None, /, commit=False, **settings)` | Set fields on a Single DocType, save with `ignore_permissions`, and restore the old values after the block. With `commit=True`, commits both saves. When `settings_dict` is given, `**settings` is ignored. |
| `patch_hooks(overridden_hooks: dict)` | Make `frappe.get_hooks(hook)` return the given value for the listed hooks. |
| `freeze_time(time_to_freeze, is_utc: bool = False, *args, **kwargs)` | Freeze the clock with `freezegun`. The time is read in the system time zone unless `is_utc=True`. |
| `enable_safe_exec()` | Set `server_script_enabled` in `common_site_config.json` for the block, then restore it. Writes the file. |
| `switch_site(site: str)` | Connect to another site for the block, then connect back. |
| `debug_on(*exceptions)` | Print the traceback and start `pdb.post_mortem` on the exceptions (default `AssertionError`), then raise. |
| `timeout_context(seconds=30, error_message="Operation timed out.")` | Raise `Exception(error_message)` after `seconds`. Uses `SIGALRM`: Unix and the main thread only. |
| `timeout(seconds=30, error_message="Operation timed out.")` | Decorator form of `timeout_context`. Works as `@timeout` or `@timeout(60)`. |
| `trace_fields(doc_class, field_name=None, forbidden_values=None, custom_validation=None, **field_configs)` | Replace fields of a document class with traced properties for the block. Assigning a forbidden value, or a value that fails `custom_validation`, raises. |

```python
from frappe.tests import IntegrationTestCase, set_user


class IntegrationTestSampleItem(IntegrationTestCase):
	@IntegrationTestCase.change_settings("System Settings", {"float_precision": 4})
	def test_precision(self):
		...

	def test_user_cannot_delete(self):
		with set_user("user1@example.com"):
			self.assertRaises(frappe.PermissionError, frappe.delete_doc, "Sample Item", "_Test Sample Item")
```

## Test records

`frappe.tests.utils` re-exports these from `frappe.tests.utils.generators`.

| Function | Description |
|---|---|
| `make_test_records(doctype, force=False, commit=False) -> list` | Make test records for the DocType and, depth first, for the DocTypes it links to. Returns `[(doctype, count), ...]`. `force=True` makes them again. |
| `make_test_records_for_doctype(doctype, force=False, commit=False) -> list` | Make records for one DocType only. Returns names. |
| `make_test_objects(doctype=None, test_records=None, reset=False, commit=False) -> list` | Insert the given records (a list, or `{doctype: [records]}`), or load them from the DocType folder. Returns names. |
| `load_test_records_for(index_doctype) -> dict` | Read `test_records.json`, else `test_records.toml`, from the DocType folder. |
| `get_missing_records_doctypes(doctype, visited=None, _parent=None) -> list[str]` | DocTypes that still need records, in dependency order. |
| `get_missing_records_module_overrides(module) -> [list, list]` | `(to_add, to_remove)` from `EXTRA_TEST_RECORD_DEPENDENCIES` and `IGNORE_TEST_RECORD_DEPENDENCIES` of a test module. |
| `get_modules(doctype)` | `(module, test_module)` of a DocType. Cached. |
| `print_mandatory_fields(doctype, initial_doctype)` | Log the mandatory fields of a DocType that has no test records. |
| `TestRecordManager` | Keeps a log of made records in `sites/<site>/.test_records.jsonl`, so a second run does not make them again. |

Where the records come from, for each DocType:

1. `_make_test_records()` in the test module. It must yield `(tag, document)` pairs.
2. `test_records` in the test module.
3. `test_records.json`, then `test_records.toml`, in the DocType folder.

Test module variables: `EXTRA_TEST_RECORD_DEPENDENCIES` (DocTypes to add) and `IGNORE_TEST_RECORD_DEPENDENCIES` (DocTypes to skip). The old names `test_dependencies` and `test_ignore` still work with a warning. `IGNORE_TEST_RECORD_DEPENDENCIES` works only in a test module inside a DocType folder.

!!! warning
    If you delete a record listed in `.test_records.jsonl`, the next run raises `ValueError` ("had been deleted resulting in inconsistent global state"). Delete the file to start again.

## Other helpers

All in `frappe.tests.utils` unless noted.

| Function | Description |
|---|---|
| `whitelist_for_tests(**whitelist_kwargs)` | Decorator. Same as `@frappe.whitelist(...)`, but the method works only in tests, under `CI`, or on `bench start` with the `allow_tests` site config. |
| `toggle_test_mode(enable: bool)` | Set `frappe.in_test` and `frappe.local.flags.in_test`. |
| `wait_for_job(job_id: str, wait_timeout: float = 10)` | Wait until the job is no longer queued or started. Returns after `wait_timeout` seconds even if the job did not run. |
| `check_orpahned_doctypes()` | Raise if a standard DocType in the database has no controller. The name has this spelling in the code. |
| `frappe.tests.utils.test_capabilities.requires_test_service(service)` | Decorator for a test, class or module. Skips the test when the `TestService` is not available. `bench run-tests --test-service` selects these tests. |
| `frappe.tests.utils.test_capabilities.is_test_service_available(service) -> bool` | `True` if the service is available to this test process. |
| `frappe.tests.global_test_dependencies` | `["User"]`. |

## Deprecated

| Name | Use instead |
|---|---|
| `frappe.tests.utils.FrappeTestCase` | `UnitTestCase` or `IntegrationTestCase`. Removal planned in v17. |
| `frappe.tests.utils.change_settings`, `patch_hooks`, `debug_on`, `timeout` | The same names from `frappe.tests`. |
| `frappe.tests.utils.get_dependencies` | `get_missing_records_doctypes`. |
| `frappe.tests.update_system_settings(args, commit=False)` | `change_settings("System Settings", ...)`. |
| `frappe.tests.get_system_setting(key)` | `frappe.db.get_single_value("System Settings", key)`. |
| `frappe.get_test_records(doctype)` | `cls.globalTestRecords[doctype]`, or `load_test_records_for`. |
| `test_dependencies`, `test_ignore` | `EXTRA_TEST_RECORD_DEPENDENCIES`, `IGNORE_TEST_RECORD_DEPENDENCIES`. |

## Pitfalls

- Test records are committed. They stay in the site database after the run.
- A test that commits (`frappe.db.commit()`) breaks the class rollback. Set `SHOW_TRANSACTION_COMMIT_WARNINGS = True` to find commits.
- `enable_safe_exec` writes `common_site_config.json`. Other processes on the bench see the change during the block.
- `change_settings` restores values with a second save. Without `commit=True`, both saves roll back with the class.

## Related

- [Testing](../../developer-guide/testing-and-performance/testing.md)
- [CLI](../configuration/cli.md)
- [Performance](../../developer-guide/testing-and-performance/performance.md)
- [Jobs, Cache and Realtime API](jobs-cache-and-realtime.md)
