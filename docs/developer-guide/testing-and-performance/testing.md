---
title: Testing
description: Write and run tests for Frappe apps with FrappeTestCase, test records, test helpers, run-tests, run-parallel-tests and run-ui-tests.
version: v15
sources:
  - frappe/commands/utils.py
  - frappe/test_runner.py
  - frappe/parallel_test_runner.py
  - frappe/tests/utils.py
  - frappe/tests/__init__.py
  - frappe/core/doctype/doctype/boilerplate/test_controller._py
  - frappe/__init__.py
---

# Testing

Frappe runs Python tests with the standard `unittest` module, through `bench run-tests`. Tests run against a real site and a real database. Use a separate test site, because tests write records to the database. UI tests run with Playwright (or Cypress) through `bench run-ui-tests`.

## Test classes

Import the classes from `frappe.tests.utils`:

| Class | Use for | Database |
|---|---|---|
| `FrappeTestCase` | All tests of an app | Commits before the class starts, rolls back after the class |
| `MockedRequestTestCase` | Code that makes HTTP requests. It mocks them with the `responses` library | Same as `FrappeTestCase` |

When you make a DocType in developer mode, Frappe writes `test_<doctype>.py` with a class that extends `FrappeTestCase`.

```python
# sample_app/sample_app/doctype/sample_item/test_sample_item.py
import frappe
from frappe.tests.utils import FrappeTestCase


class TestSampleItem(FrappeTestCase):
	def test_rate_is_saved(self):
		item = frappe.get_doc(
			{"doctype": "Sample Item", "item_name": "Test Bolt", "category": "Hardware", "rate": 10}
		).insert()
		self.assertEqual(item.rate, 10)
```

If you override `setUpClass`, call `super().setUpClass()`. Otherwise the class does not roll back.

`FrappeTestCase` adds:

| Name | Description |
|---|---|
| `assertDocumentEqual(expected, actual)` | Compares a dict or a document with a document. Floats use the field precision. |
| `assertQueryEqual(first, second)` | Compares two SQL strings after formatting. |
| `assertSequenceSubset(larger, smaller)` | Fails if `smaller` is not a subset of `larger`. |
| `assertQueryCount(count)` | Context manager. Fails if more than `count` SQL queries run. |
| `assertRowsRead(count)` | Context manager. Fails if queries read more than `count` rows. |
| `assertRedisCallCounts(count)` | Context manager. Fails if more Redis calls run. |
| `primary_connection()`, `secondary_connection()` | Context managers. Switch between two database connections to test concurrency. |
| `set_user(user)` | Context manager. Run as another user, then switch back. |
| `switch_site(site)` | Context manager. Connect to another site for the block. |
| `freeze_time(time_to_freeze)` | Context manager. Freeze the clock with `freezegun`, in the system time zone. |
| `enable_safe_exec()` | Class method. Turn on server scripts for the tests of the class. |
| `SHOW_TRANSACTION_COMMIT_WARNINGS` | Class attribute. Set `True` to print a warning when code commits. |

!!! note "Version"
    v16 adds `UnitTestCase` and `IntegrationTestCase` in `frappe.tests`. `FrappeTestCase` still works there, but is deprecated.

## Other helpers

These are functions in `frappe/tests/utils.py`:

| Name | Description |
|---|---|
| `change_settings(doctype, settings_dict=None, /, commit=False, **settings)` | Context manager or decorator. Change a Single DocType for the block, then restore. |
| `patch_hooks(overridden_hooks)` | Context manager. Replace hook values for the block. |
| `timeout(seconds=30)` | Decorator. Fail if the function runs longer. |

```python
import frappe
from frappe.tests.utils import FrappeTestCase, change_settings


class TestSampleOrder(FrappeTestCase):
	def test_user_cannot_read_other_orders(self):
		with self.set_user("user1@example.com"):
			names = frappe.get_list("Sample Order", pluck="name")
		...

	@change_settings("System Settings", {"float_precision": 3})
	def test_precision(self):
		...
```

## Test records

Before the tests of a DocType run, the test runner creates test records for the DocType and, recursively, for the DocTypes of its Link fields (also Link fields in child tables). It commits them. A record whose name exists is not created again. For each DocType, Frappe looks for records in this order (`frappe/test_runner.py`):

1. A `_make_test_records(verbose)` function in the test module.
2. A `test_records` attribute in the test module.
3. A `test_records.json` file next to the DocType.

```json
[
	{
		"doctype": "Sample Item",
		"item_name": "_Test Sample Item",
		"category": "Hardware",
		"rate": 100
	}
]
```

Control the dependencies with module variables in the test module:

```python
test_dependencies = ["User"]
test_ignore = ["Sample Role"]
```

`frappe.local.test_objects[doctype]` holds the names of the records made. `--skip-test-records` skips the creation.

## Run tests

Tests are off by default. Turn them on for the site once:

```bash
bench --site test.localhost set-config allow_tests true
```

`run-tests` also runs when the `CI` environment variable is set. With `CI` set, the command exits with a non-zero code on failure.

```bash
bench --site test.localhost run-tests --app sample_app
bench --site test.localhost run-tests --doctype "Sample Item"
bench --site test.localhost run-tests --module sample_app.sample_app.doctype.sample_item.test_sample_item
bench --site test.localhost run-tests --module sample_app.tests.test_api --test test_get_count
```

| Option | Description |
|---|---|
| `--app` | Run all tests of an app. |
| `--doctype` | Run the tests of a DocType. |
| `--module-def` | Run the tests of all DocTypes in a Module Def. |
| `--doctype-list-path` | Path to a `.txt` file with DocType names, as `<app>/<path>`. |
| `--module` | Dotted path of a test module. |
| `--case` | Run one `TestCase` class of the module. Use with `--module`. |
| `--test` | Run one test method. Repeat for more. |
| `--failfast` | Stop at the first failure. |
| `--profile` | Run with cProfile. |
| `--coverage` | Collect coverage. |
| `--junit-xml-output` | Write a JUnit XML report to this file. |
| `--skip-test-records` | Do not create test records. |
| `--skip-before-tests` | Do not run `before_tests` hooks. |

With `--app` or no filter, Frappe finds files whose names start with `test_` and end with `.py`, in all folders of the app except `public`, `locals` and `__pycache__`.

During a test run, Frappe sets `frappe.flags.in_test`. It disables the scheduler and enables it again at the end, unless it was already disabled. Before the tests, it runs the `before_tests` hooks of the app.

```python
# sample_app/hooks.py
before_tests = "sample_app.tests.setup.before_tests"
```

## Parallel tests in CI

```bash
bench --site test.localhost run-parallel-tests --app sample_app --build-number 1 --total-builds 4
```

| Option | Default | Description |
|---|---|---|
| `--app` | `frappe` | The app to test. |
| `--build-number` | `1` | Number of this build. |
| `--total-builds` | `1` | Number of builds that share the tests. |
| `--with-coverage` | off | Write a coverage file. |
| `--use-orchestrator` | off | Get tests from an orchestrator service. |
| `--dry-run` | off | List tests, do not run them. |

The parallel runner also creates records for the DocTypes in `global_test_dependencies` of `<app>.tests` (`frappe/tests/__init__.py` sets `["User"]`).

## UI tests

`bench run-ui-tests <app>` runs Playwright if the app has a `playwright.config.*` file. It installs `@playwright/test` and the browser on first use.

```bash
bench --site test.localhost run-ui-tests sample_app --headless
bench --site test.localhost run-ui-tests sample_app --headless --browser firefox --spec tests/ui/sample_item.spec.js
```

| Option | Description |
|---|---|
| `--headless` | Run without the Playwright UI. |
| `--browser` | `chromium` (default), `firefox` or `webkit`. |
| `--spec` | One spec file. |
| `--parallel-site` | Another site to run tests on. Repeat for more. Each site gets a worker. |

Frappe sets `BASE_URL` to the site URL, and `ADMIN_PASSWORD` when the site has `admin_password`. Extra arguments go to `playwright test`. Apps with only a `cypress.config.*` file still run with Cypress, with a deprecation warning.

## Pitfalls

- Do not run tests on a site with real data. Test records are committed.
- Test as a normal user, not Administrator. Permission bugs do not show as Administrator. Use `frappe.get_list` in permission tests, not `frappe.get_all`.
- Use `example.com` addresses for test users and emails.
- Do not add `if frappe.flags.in_test` branches to app code. Use `patch_hooks`, `change_settings` or mocks in the test.
- A test that calls `frappe.db.commit()` makes its records permanent. The class rollback cannot undo them.
- `enable_safe_exec()` sets `server_script_enabled` to 0 after the class, even if it was 1 before.

## Related

- [Quickstart](../../getting-started/quickstart.md)
- [Fixtures and Patches](../building-apps/fixtures-and-patches.md)
- [Server Scripts and Client Scripts](../low-code-and-ai/server-and-client-scripts.md)
- [CLI reference](../../reference/configuration/cli.md)
