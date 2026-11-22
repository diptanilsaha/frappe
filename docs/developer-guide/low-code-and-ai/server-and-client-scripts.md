---
title: Server Scripts and Client Scripts
description: Write Python Server Scripts and JavaScript Client Scripts in the desk, enable server scripts, and know the restricted API that safe_exec allows.
version: v15
sources:
  - frappe/core/doctype/server_script/server_script.py
  - frappe/core/doctype/server_script/server_script.json
  - frappe/core/doctype/server_script/server_script_utils.py
  - frappe/utils/safe_exec.py
  - frappe/handler.py
  - frappe/api/v2.py
  - frappe/custom/doctype/client_script/client_script.json
  - frappe/desk/form/meta.py
  - frappe/tests/utils.py
---

# Server Scripts and Client Scripts

**Server Script** and **Client Script** are DocTypes that store code in the database. You write them in the desk, without an app. Use them for small, site-specific changes. Use app code (controllers, hooks, form scripts) for logic that must be versioned, reviewed, tested, or shipped to more than one site.

| | Server Script | Client Script |
|---|---|---|
| Language | Restricted Python (RestrictedPython) | JavaScript |
| Runs | On the server | In the browser of the user |
| Role to create | `Script Manager` | `System Manager` |
| Must be enabled | Yes, `server_script_enabled` in `common_site_config.json` | No, but each script has `enabled` (default off) |
| Turn one script off | Set `disabled` | Clear `enabled` |

## Enable server scripts

Server scripts are off by default. Frappe reads the key only from `sites/common_site_config.json`, not from `site_config.json` (`is_safe_exec_enabled` in `frappe/utils/safe_exec.py`). The setting applies to all sites of the bench.

```bash
bench set-config -g server_script_enabled 1
```

When the key is off, every Server Script raises `ServerScriptNotEnabled` with the message "Server Scripts are disabled. Please enable server scripts from bench configuration." This also applies to other features that use `safe_exec`, such as Script Reports written in the desk. You can still save a Server Script while the key is off.

## Server Script types

Set **Script Type** (`script_type`):

| Type | Extra fields | Runs when | Variables in scope |
|---|---|---|---|
| `DocType Event` | `reference_doctype`, `doctype_event` | The document event fires | `doc` |
| `API` | `api_method`, `allow_guest`, rate limit fields | A request to `/api/method/<api_method>` | `frappe.form_dict` |
| `Scheduler Event` | `event_frequency`, `cron_format` | The scheduler runs the job | none |
| `Permission Query` | `reference_doctype` | `frappe.get_list` builds the query | `user`, set `conditions` |

### DocType Event

`doctype_event` maps to controller events (`EVENT_MAP` in `frappe/core/doctype/server_script/server_script_utils.py`):

| `doctype_event` | Controller event |
|---|---|
| Before Insert | `before_insert` |
| Before Validate | `before_validate` |
| Before Save | `validate` |
| After Insert | `after_insert` |
| After Save | `on_update` |
| Before Rename / After Rename | `before_rename` / `after_rename` |
| Before Submit / After Submit | `before_submit` / `on_submit` |
| Before Cancel / After Cancel | `before_cancel` / `on_cancel` |
| Before Delete / After Delete | `on_trash` / `after_delete` |
| Before Save (Submitted Document) / After Save (Submitted Document) | `before_update_after_submit` / `on_update_after_submit` |
| Before Print | `before_print` |
| On Payment Authorization | `on_payment_authorized` |

Scripts run after the controller and `doc_events` handlers for the same event. They do not run during `bench install-app` or `bench migrate`.

```python
# Server Script: DocType Event, Sample Order, Before Save
total = 0
for row in doc.items:
	row.amount = frappe.utils.flt(row.qty) * frappe.utils.flt(row.rate)
	total += row.amount
doc.total = total

if doc.total <= 0:
	frappe.throw(_("Total must be more than zero"))
```

`frappe.db.commit`, `frappe.db.rollback` and `frappe.db.add_index` are removed in DocType Event scripts.

### API

The script runs at `/api/method/<api_method>` and `/api/v2/method/<api_method>`. Set the response in `frappe.response["message"]`. If you set values on `frappe.flags`, Frappe returns the flags as the message instead.

```python
# Server Script: API, api_method = sample_item_count
category = frappe.form_dict.get("category")
frappe.response["message"] = frappe.db.count("Sample Item", {"category": category})
```

```bash
curl -b jar "http://frappe.localhost:8001/api/method/sample_item_count?category=Hardware"
```

```json
{"message": 3}
```

- With `allow_guest` off, a Guest gets `PermissionError`.
- With `enable_rate_limit` on, `rate_limit_count` defaults to 5 and `rate_limit_seconds` to 86400 when they are empty.
- An API script with `allow_guest` on and no rate limit can be called without limit by anyone. Turn on `enable_rate_limit` for guest scripts.
- The script has priority: an API script with the same path as a Python method replaces it.

### Scheduler Event

Saving the script creates or updates a **Scheduled Job Type** that points to it. `event_frequency` takes `All`, `Hourly`, `Daily`, `Weekly`, `Monthly`, `Yearly`, the `Long` variants, or `Cron` with `cron_format`. Setting `disabled` stops the job.

### Permission Query

Set `conditions` to a SQL condition string. Frappe adds it to the `WHERE` clause of `frappe.get_list` for `reference_doctype`.

```python
# Server Script: Permission Query, Sample Order
conditions = f"`tabSample Order`.owner_user = {frappe.db.escape(user)}"
```

## The restricted environment

`safe_exec` compiles the script with RestrictedPython and runs it with the globals from `get_safe_globals()`.

- `import` is not allowed. Use the names below.
- Attributes and keys that start with `_` are not allowed.
- `print()` writes to `frappe.log`, not to the console.
- `frappe.db.sql` allows only queries that start with `SELECT` or `EXPLAIN`, and `WITH` on MariaDB. It rejects `INTO OUTFILE`, `DUMPFILE` and queries on `__Auth`.

### Allowed API

Top-level names:

`frappe`, `json` (`loads`, `dumps`), `as_json`, `_`, `_dict`, `dict`, `log`, `args` (the form dict), `scrub`, `guess_mimetype`, `html2text`, `get_toc`, `get_next_link`, `get_visible_columns`, `run_script`, `FrappeClient`, `style`, `dev_server`, and the builtins `abs`, `all`, `any`, `bool`, `dict`, `enumerate`, `isinstance`, `issubclass`, `list`, `max`, `min`, `range`, `set`, `sorted`, `sum`, `tuple`.

`frappe.*`:

| Group | Names |
|---|---|
| Documents | `get_doc`, `new_doc`, `copy_doc`, `get_cached_doc`, `get_last_doc`, `get_mapped_doc`, `rename_doc`, `delete_doc`, `get_meta` |
| Lists | `get_list`, `get_all`, `qb` |
| Database (`frappe.db.*`) | `get_list`, `get_all`, `get_value`, `get_single_value`, `set_value`, `exists`, `count`, `escape`, `get_default`, `sql` (read-only), `commit`, `rollback`, `add_index`, `after_commit`, `before_commit`, `after_rollback`, `before_rollback` |
| Messages and errors | `msgprint`, `throw`, `log_error`, `errprint`, `log`, and all exception classes of `frappe.exceptions` (for example `frappe.ValidationError`) |
| Calls and jobs | `call` (calls a whitelisted method or API script), `enqueue` (enqueues a whitelisted method or API script), `is_job_queued` |
| HTTP | `make_get_request`, `make_post_request`, `make_put_request`, `make_patch_request`, `make_delete_request` |
| Email and print | `sendmail`, `get_print`, `attach_print`, `render_template` |
| Formatting | `format`, `format_value`, `format_date`, `bold`, `date_format`, `time_format`, `sanitize_html` |
| Session and request | `session` (`user`, `csrf_token`), `user`, `full_name`, `form_dict`, `request` (`path`), `response`, `flags`, `lang` |
| Users | `get_fullname`, `get_gravatar`, `get_identicon` |
| Other | `get_hooks`, `get_system_settings`, `get_url`, `website` (`abs_url`, `extract_title`, `get_boot_data`, `get_home_page`, `get_html_content_based_on_type`), `socketio_port`, `json_handler` |

`frappe.utils` contains a fixed list of helpers from `frappe/utils/data.py` (`VALID_UTILS` in `frappe/utils/safe_exec.py`). Examples: `flt`, `cint`, `cstr`, `getdate`, `get_datetime`, `nowdate`, `now`, `today`, `add_days`, `add_months`, `date_diff`, `fmt_money`, `money_in_words`, `strip_html`, `escape_html`, `get_url_to_form`, `generate_hash`. Other functions of `frappe.utils` are not available.

In Server Scripts, `frappe.get_doc`, `frappe.get_list`, `frappe.get_all` and the other document functions are the normal functions. They behave as in app code: `get_list` checks permissions of the session user, `get_all` does not, and `doc.save()` checks write permission.

!!! note "Version"
    In v15 Server Scripts, the HTTP helpers are the normal functions of `frappe.integrations.utils`. They do not block requests to private or local addresses. Give the `Script Manager` role only to trusted users.

## Client Script

A Client Script adds JavaScript to the form or list of a DocType. Set:

| Field | Description |
|---|---|
| `dt` | The DocType. |
| `view` | `Form` (default) or `List`. |
| `enabled` | Default off. Turn it on, or the script does not load. |
| `module` | Optional. Use it to filter the script in a fixture. Frappe does not check it when it loads scripts. |
| `script` | The code. |

```js
// Client Script: Sample Order, Form
frappe.ui.form.on("Sample Order", {
	refresh(frm) {
		if (frm.doc.docstatus === 0) {
			frm.add_custom_button(__("Recalculate"), () => frm.trigger("recalculate"));
		}
	},
});
```

Frappe appends all enabled scripts of the DocType, ordered by creation, to the form script or the list script (`add_custom_script` in `frappe/desk/form/meta.py`). For a `List` script, set `frappe.listview_settings["<DocType>"]`. See [Form Scripts](../desk-ui/form-scripts.md) and [List View Settings](../desk-ui/list-view-settings.md) for the API.

A Client Script runs only in the desk. Validation in a Client Script does not protect the data. The REST API, Data Import and server code skip it. Put rules that must hold on the server.

## Ship scripts with an app

Server Scripts and Client Scripts can be fixtures:

```python
# sample_app/hooks.py
fixtures = [
	{"dt": "Server Script", "filters": [["module", "=", "Sample App"]]},
	{"dt": "Client Script", "filters": [["module", "=", "Sample App"]]},
]
```

For code that you own, prefer controllers, `doc_events` and `doctype_js` in the app. They do not need `server_script_enabled`, and they run under your tests.

## Test with server scripts

Call the `enable_safe_exec` class method of `FrappeTestCase` in `setUpClass`. It sets `server_script_enabled` to 1 in `common_site_config.json`. A class cleanup sets it to 0 after the tests of the class, even if it was 1 before.

```python
# sample_app/sample_app/tests/test_sample_script.py
from frappe.tests.utils import FrappeTestCase


class TestSampleScript(FrappeTestCase):
	@classmethod
	def setUpClass(cls):
		super().setUpClass()
		cls.enable_safe_exec()

	def test_api_script(self):
		...
```

## Pitfalls

- `server_script_enabled` in `site_config.json` has no effect. Use `bench set-config -g`.
- Only one Permission Query script per DocType is used.
- A Server Script with an error in a `Before Save` event blocks all saves of the DocType. Test on a copy of the site first.
- Server Scripts are cached in `server_script_map`. Saving or deleting a script clears the cache.
- Do not build SQL from user input. Use `frappe.db.escape` or `frappe.qb`.

## Related

- [Extending Other Apps](../building-apps/extending-apps.md)
- [Form Scripts](../desk-ui/form-scripts.md)
- [Whitelisted Methods](../api/whitelisted-methods.md)
- [Security](../security/security.md)
- [Scheduler](../runtime/scheduler.md)
- [Site configuration reference](../../reference/configuration/site-config.md)
