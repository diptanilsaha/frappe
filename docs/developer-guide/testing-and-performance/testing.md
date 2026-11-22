---
title: Testing
description: Write and run tests for Frappe apps with UnitTestCase and IntegrationTestCase, test records, context managers, run-tests, run-parallel-tests and run-ui-tests.
version: develop
sources:
  - frappe/commands/testing.py
  - frappe/testing/README.md
  - frappe/testing/discovery.py
  - frappe/testing/runner.py
  - frappe/testing/environment.py
  - frappe/tests/__init__.py
  - frappe/tests/classes/unit_test_case.py
  - frappe/tests/classes/integration_test_case.py
  - frappe/tests/classes/context_managers.py
  - frappe/tests/utils/generators.py
  - frappe/tests/utils/test_capabilities.py
  - frappe/core/doctype/doctype/boilerplate/test_controller._py
  - frappe/deprecation_dumpster.py
---

# Testing

Frappe runs Python tests with the standard `unittest` module, through `bench run-tests`. Tests run against a real site and a real database. Use a separate test site, because integration tests write records to the database. UI tests run with Playwright through `bench run-ui-tests`.

## Test classes

Import the classes from `frappe.tests`:

| Class | Use for | Database |
|---|---|---|
| `UnitTestCase` | Code that does not need the database, or needs it only to read | Not rolled back |
| `IntegrationTestCase` | Code that reads and writes documents | Connected to the site, test records created, rolled back after the class |
| `MockedRequestTestCase` | Code that makes HTTP requests. It mocks them with the `responses` library | Same as `IntegrationTestCase` |

```python
# sample_app/sample_app/doctype/sample_item/test_sample_item.py
import frappe
from frappe.tests import IntegrationTestCase


class IntegrationTestSampleItem(IntegrationTestCase):
	def test_rate_is_saved(self):
		item = frappe.get_doc(
			{"doctype": "Sample Item", "item_name": "Test Bolt", "category": "Hardware", "rate": 10}
		).insert()
		self.assertEqual(item.rate, 10)
```

`UnitTestCase` adds these assertions: `assertDocumentEqual(expected, actual)`, `assertQueryEqual(first, second)`, `assertSequenceSubset(larger, smaller)`. In `setUpClass` it sets `cls.doctype` (from the folder of the test file) and sets the user to `Administrator`.

`IntegrationTestCase` adds:

| Name | Description |
|---|---|
| `assertQueryCount(count, query_type=None)` | Context manager. Fails if more than `count` SQL queries run. |
| `assertRowsRead(count)` | Context manager. Fails if queries read more than `count` rows. |
| `assertRedisCallCounts(count, exact=False)` | Context manager. Fails if more Redis calls run. |
| `primary_connection()`, `secondary_connection()` | Context managers. Switch between two database connections to test concurrency. |
| `globalTestRecords` | Read-only map of the test records made so far. |
| `SHOW_TRANSACTION_COMMIT_WARNINGS` | Class attribute. Set `True` to print a warning when code commits. |

If you override `setUpClass`, call `super().setUpClass()`.

`frappe.tests.utils.FrappeTestCase` is deprecated for removal in v17. Use `UnitTestCase` or `IntegrationTestCase`.

## Context managers

Import them from `frappe.tests`. Each context manager also works as a decorator. Most are also attributes of `UnitTestCase`, so you can use `self.<name>`. `change_settings` and `switch_site` are attributes of `IntegrationTestCase` only.

| Name | Description |
|---|---|
| `set_user(user)` | Run as another user, then switch back. |
| `change_settings(doctype, settings_dict=None, /, commit=False, **settings)` | Change a Single DocType for the block, then restore. |
| `patch_hooks(overridden_hooks)` | Replace hook values for the block. |
| `freeze_time(time_to_freeze, is_utc=False)` | Freeze the clock with `freezegun`. |
| `enable_safe_exec()` | Turn on server scripts for the block. |
| `switch_site(site)` | Connect to another site for the block. |
| `debug_on(*exceptions)` | Start `pdb` on the given exceptions. |
| `timeout(seconds=30)` | Decorator. Fail if the function runs longer. Not a class attribute. |
| `timeout_context(seconds=30)` | Context manager form of `timeout`. |
| `trace_fields(doc_class, ...)` | Trace changes to fields of a document class. |

```python
from frappe.tests import IntegrationTestCase, set_user


class IntegrationTestSampleOrder(IntegrationTestCase):
	def test_user_cannot_read_other_orders(self):
		with set_user("user1@example.com"):
			names = frappe.get_list("Sample Order", pluck="name")
		...

	@IntegrationTestCase.change_settings("System Settings", {"float_precision": 3})
	def test_precision(self):
		...
```

## Test records

When an `IntegrationTestCase` class starts, Frappe creates test records for its DocType and, recursively, for the DocTypes of its Link fields. It commits them. Frappe looks for records in this order (`frappe/tests/utils/generators.py`):

1. A `_make_test_records()` function in the test module.
2. A `test_records` attribute in the test module.
3. A `test_records.json` file next to the DocType.
4. A `test_records.toml` file next to the DocType.

```toml
# sample_app/sample_app/doctype/sample_item/test_records.toml
[["Sample Item"]]
item_name = "_Test Sample Item"
category = "Hardware"
rate = 100
```

Control the dependencies with module variables:

```python
EXTRA_TEST_RECORD_DEPENDENCIES = ["User"]
IGNORE_TEST_RECORD_DEPENDENCIES = ["Sample Role"]
```

The old names `test_dependencies` and `test_ignore` still work but show a deprecation warning. Records are loaded lazily. `--skip-test-records` is deprecated and has no effect.

## Run tests

Tests are off by default. Turn them on for the site once:

```bash
bench --site test.localhost set-config allow_tests true
```

`run-tests` also runs when the `CI` environment variable is set.

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
| `--doctype-list-path` | Path to a `.txt` file with DocType names. |
| `--module` | Dotted path of a test module. |
| `--case` | Run one `TestCase` class of the module. |
| `--test` | Run one test method. Repeat for more. |
| `--test-category` | `unit`, `integration` or `all` (default). |
| `--test-service` | Run only tests that declare a required service, for example `web-server`. |
| `--failfast` | Stop at the first failure. |
| `--debug` | Disable output buffer and start `pdb` on errors. |
| `--profile` | Run with cProfile. |
| `--coverage` | Collect coverage. |
| `--junit-xml-output` | Write a JUnit XML report to this file. |
| `--skip-before-tests` | Do not run `before_tests` hooks. |
| `--lightmode` | Skip the full test setup. |

`--doctype`, `--doctype-list-path`, `--module-def` and `--module` cannot be combined.

Frappe finds test files whose names start with `test_` and end with `.py`. It groups tests into `unit` and `integration` by their class and runs unit tests first. Before integration tests, it runs the `before_tests` hooks of the app and creates records for `global_test_dependencies`.

```python
# sample_app/hooks.py
before_tests = "sample_app.tests.setup.before_tests"
```

Detailed logs are written to a log file. `run-tests` prints its path at the start.

## Parallel tests in CI

```bash
bench --site test.localhost run-parallel-tests --app sample_app --build-number 1 --total-builds 4
```

| Option | Default | Description |
|---|---|---|
| `--app` | `frappe` | The app to test. |
| `--build-number` | `1` | Number of this build. |
| `--total-builds` | `1` | Number of builds that share the tests. |
| `--with-coverage` | off | Write a coverage file. Also read from `CAPTURE_COVERAGE`. |
| `--use-orchestrator` | off | Get tests from an orchestrator service. |
| `--dry-run` | off | List tests, do not run them. |
| `--lightmode` | off | Skip the test setup. |
| `--failfast` | off | Stop at the first failure. |

## UI tests

`bench run-ui-tests <app>` runs Playwright if the app has a Playwright config file. It installs `@playwright/test` and the browser on first use.

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

Frappe sets `BASE_URL` to the site URL, and `ADMIN_PASSWORD` when the site has `admin_password`. Extra arguments go to `playwright test`. Apps with only a Cypress config still run with Cypress, with a warning. Cypress support is deprecated for removal in v17.

## Pitfalls

- Do not run tests on a site with real data. Test records are committed.
- Test as a normal user, not Administrator. Permission bugs do not show as Administrator. Use `frappe.get_list` in permission tests, not `frappe.get_all`.
- Use `example.com` addresses for test users and emails.
- Do not add `if frappe.in_test` branches to app code. Use `patch_hooks`, `change_settings` or mocks in the test.
- A test that calls `frappe.db.commit()` makes its records permanent. The class rollback cannot undo them.

## Related

- [Quickstart](../../getting-started/quickstart.md)
- [Fixtures and Patches](../building-apps/fixtures-and-patches.md)
- [Server Scripts and Client Scripts](../low-code-and-ai/server-and-client-scripts.md)
- [CLI reference](../../reference/configuration/cli.md)
