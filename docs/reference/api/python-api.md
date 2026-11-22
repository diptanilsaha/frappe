---
title: Python API
description: Public functions and objects on the frappe module, grouped by theme, with signatures and one line each.
version: v15
sources:
  - frappe/__init__.py
  - frappe/model/document.py
  - frappe/model/meta.py
  - frappe/utils/background_jobs.py
  - frappe/utils/caching.py
  - frappe/utils/jinja.py
  - frappe/utils/print_utils.py
  - frappe/utils/formatters.py
  - frappe/utils/error.py
  - frappe/utils/redis_wrapper.py
  - frappe/utils/deprecations.py
  - frappe/realtime.py
  - frappe/query_builder/utils.py
---

# Python API

This page lists the public names on the `frappe` module. Use it to find the exact name and signature of a function. The list comes from `dir(frappe)`, read with `inspect.signature` on a v15 site. Internal helpers, re-exported standard library modules and exception classes are left out.

In v15, most functions are defined directly in `frappe/__init__.py`. Some are thin wrappers with the signature `(*args, **kwargs)` that import their module on first call. For these, the signature of the target function is shown, and the target file is named.

For `frappe.db` methods, see [Database API](database-api.md). For `frappe.utils` helpers, see [Utilities](utils.md).

## Context objects

These are request-local objects. They are proxies to `frappe.local` and are valid only after `frappe.init()` and `frappe.connect()`.

| Name | Type | Description |
|---|---|---|
| `frappe.local` | `Local` | Thread-local and request-local storage. All objects below live on it. |
| `frappe.db` | `LocalProxy` to `Database` | Database connection of the current site. See [Database API](database-api.md). |
| `frappe.qb` | `LocalProxy` | Query builder for the current database type. See [Query Builder](../../developer-guide/database/query-builder.md). |
| `frappe.session` | `LocalProxy` | Current session. `frappe.session.user` is the name of the current user. |
| `frappe.user` | `LocalProxy` | Proxy to `frappe.local.user`. It is always `None`. `set_user` does not set it. Use `frappe.session.user`. |
| `frappe.form_dict` | `LocalProxy` (`_dict`) | Request parameters (query string and form body). |
| `frappe.form` | `LocalProxy` | Alias of `frappe.form_dict`. |
| `frappe.request` | `LocalProxy` | The Werkzeug request object. |
| `frappe.response` | `LocalProxy` (`_dict`) | Response dict. Whitelisted methods put their return value in `frappe.response["message"]`. |
| `frappe.flags` | `LocalProxy` (`_dict`) | Request-scoped flags, for example `frappe.flags.in_test`, `frappe.flags.in_import`, `frappe.flags.in_migrate`. |
| `frappe.conf` | `LocalProxy` (`_dict`) | Merged `site_config.json` and `common_site_config.json`. See [Configuration](../../developer-guide/operations/configuration.md). |
| `frappe.lang` | `LocalProxy` | Language code of the current request. |
| `frappe.message_log` | `LocalProxy` (list) | Messages queued by `frappe.msgprint`. |
| `frappe.error_log` | `LocalProxy` (list) | Errors queued by `frappe.errprint`. |
| `frappe.debug_log` | `LocalProxy` (list) | Lines queued by `frappe.log`. |
| `frappe.job` | `LocalProxy` | Current background job info, if the code runs in a worker. |
| `frappe.cache` | `RedisWrapper` | Redis cache client. `frappe.cache()` also works and returns the same object, for older code. See [Caching](../../developer-guide/runtime/caching.md). |
| `frappe.STANDARD_USERS` | `tuple` | `("Guest", "Administrator")`. |

| Function | Signature | Description |
|---|---|---|
| `frappe.init` | `(site: str, sites_path: str = '.', new_site: bool = False, force=False) -> None` | Initialize frappe for a site. Resets `frappe.local`. |
| `frappe.init_site` | `(site=None)` | Context manager. Runs `frappe.init(site)` for the block, then `frappe.destroy()`. |
| `frappe.connect` | `(site: str \| None = None, db_name: str \| None = None, set_admin_as_user: bool = True) -> None` | Connect to the site database. If `site` is given, it calls `frappe.init(site)` first. |
| `frappe.connect_replica` | `() -> bool` | Connect to the read replica set in site config. |
| `frappe.destroy` | `()` | Close the database connection and release `frappe.local`. |
| `frappe.set_user` | `(username: str)` | Set the current user. Clears cached user permissions. |
| `frappe.get_user` | `()` | Return a `UserPermissions` object for the session user. |
| `frappe.get_roles` | `(username=None) -> list[str]` | Return the roles of a user. Default is the session user. |
| `frappe.get_request_header` | `(key, default=None)` | Return an HTTP request header. |
| `frappe.get_site_config` | `(sites_path: str \| None = None, site_path: str \| None = None) -> dict[str, Any]` | Return `site_config.json` merged with `common_site_config.json`. |
| `frappe.get_common_site_config` | `(sites_path: str \| None = None) -> dict[str, Any]` | Return `common_site_config.json`. |
| `frappe.get_conf` | `(site: str \| None = None) -> dict[str, Any]` | Return the config of a site. |
| `frappe.get_system_settings` | `(key: str)` | Return one value from System Settings. |
| `frappe.get_website_settings` | `(key)` | Return one value from Website Settings. |
| `frappe.get_site_path` | `(*joins)` | Return the path of the current site folder, joined with `joins`. |
| `frappe.is_setup_complete` | `()` | Return `True` if the setup wizard is complete. |
| `frappe.ping` | `()` | Return `"pong"`. |
| `frappe.logger` | `(module=None, with_more_info=False, allow_site=True, filter=None, max_size=100000, file_count=20)` | Return a Python logger that writes to the bench log folder. |
| `frappe.log` | `(msg: str) -> None` | Add a line to `frappe.debug_log`. |
| `frappe.errprint` | `(msg: str) -> None` | Add an error line. It is sent back as `exc` in the response. |
| `frappe.print_sql` | `(enable: bool = True) -> None` | Print every SQL query to the console. Use it to debug. |

## Documents

See [Controllers](../../developer-guide/data-model/controllers.md) for the `Document` class and its lifecycle.

| Function | Signature | Description |
|---|---|---|
| `frappe.get_doc` | `(doctype, name, /, *, for_update=None)` or `(doctype, /)` or `(dict)` or `(**kwargs)` | Return a `Document`. With a name, load it from the database. With a Single DocType name, load the Single. With a dict, make a new unsaved document. Calls `frappe.model.document.get_doc`. |
| `frappe.new_doc` | `(doctype: str, *, parent_doc=None, parentfield=None, as_dict=False, **kwargs) -> Document` | Return a new document with defaults set. Extra kwargs set field values. |
| `frappe.get_lazy_doc` | same as `get_doc` | Alias of `get_doc` in v15 (`get_lazy_doc = get_doc`). Child tables load at once. |
| `frappe.get_cached_doc` | `(*args, **kwargs) -> Document` | Same as `get_doc`, but return a copy from the cache if present. |
| `frappe.get_last_doc` | `(doctype, filters=None, order_by='creation desc', *, for_update=False)` | Return the last created document that matches the filters. Raises `DoesNotExistError` if none match. |
| `frappe.get_single` | `(doctype)` | Return the document of a Single DocType. |
| `frappe.copy_doc` | `(doc: Document, ignore_no_copy: bool = True) -> Document` | Return an unsaved copy of a document. Clears `name`, `owner`, `creation`, `modified`, `modified_by`. |
| `frappe.delete_doc` | `(doctype=None, name=None, force=False, ignore_doctypes=None, for_reload=False, ignore_permissions=False, flags=None, ignore_on_trash=False, ignore_missing=True, delete_permanently=False)` | Delete a document. Calls `frappe.model.delete_doc.delete_doc`. |
| `frappe.delete_doc_if_exists` | `(doctype, name, force=0)` | Delete a document if it exists. |
| `frappe.rename_doc` | `(doctype: str, old: str, new: str, force=False, merge=False, *, ignore_if_exists=False, show_alert=True, rebuild_search=True) -> str` | Rename a document and update all Link fields that point to it. |
| `frappe.get_version` | `(doctype, name, limit=None, head=False, raise_err=True)` | Return the change history of a document from `Version` records. Each item has `version`, `user` and `creation`. The DocType must track changes. |
| `frappe.get_meta` | `(doctype, cached=True)` | Return the metadata (DocType with custom fields and property setters applied). |
| `frappe.get_meta_module` | `(doctype)` | Return the Python module of a DocType controller. |
| `frappe.is_table` | `(doctype: str) -> bool` | Return `True` if the DocType is a child table (`istable`). |
| `frappe.get_doctype_app` | `(doctype)` | Return the app that owns a DocType. |
| `frappe.get_precision` | `(doctype: str, fieldname: str, currency: str \| None = None, doc=None) -> int` | Return the precision of a Float, Currency or Percent field. |
| `frappe.clear_document_cache` | `(doctype: str, name: str \| None = None) -> None` | Remove a document from the document cache. |
| `frappe.reload_doc` | `(module: str, dt=None, dn=None, force=False, reset_permissions=False)` | Reload a DocType or record from its JSON file in the module. Use it in patches. |
| `frappe.reload_doctype` | `(doctype, force=False, reset_permissions=False)` | Reload a DocType from its JSON file. |
| `frappe.import_doc` | `(path)` | Import documents from a JSON file, or every JSON file in a folder. |
| `frappe.make_property_setter` | `(args, ignore_validate=False, validate_fields_for_doctype=True, is_system_generated=True, *, module=None)` | Make a Property Setter that overrides a DocType or DocField property. |

## Queries

See [Database](../../developer-guide/database/database.md) for filter syntax.

| Function | Signature | Description |
|---|---|---|
| `frappe.get_list` | `(doctype, *args, **kwargs)` | Return records as a list of dicts. Applies permissions of the current user. The docstring says the default page length is 20, but a call without `limit` returns all rows. |
| `frappe.get_all` | `(doctype, *args, **kwargs)` | Same as `get_list`, but does not check permissions. Sets `limit_page_length=0` (no limit) if you do not pass it. Default `fields` is `["name"]`. |
| `frappe.get_value` | `(*args, **kwargs)` | Alias of `frappe.db.get_value`. |
| `frappe.get_cached_value` | `(doctype: str, name: str, fieldname: str = 'name', as_dict: bool = False)` | Return field values from the cached document. Returns `None` if the document does not exist. |
| `frappe.get_single_value` | `(setting: str, fieldname: str, /, *, as_dict: bool = False)` | Return a cached value from a Single DocType. |
| `frappe.set_value` | `(doctype, docname, fieldname, value=None)` | Set a value through `frappe.client.set_value`. Runs validations and saves the document. |
| `frappe.get_query` | `(*args, **kwargs) -> QueryBuilder` | Build a query builder object from doctype, fields and filters. Defined in `frappe/query_builder/utils.py`. See [Query Builder](../../developer-guide/database/query-builder.md). |
| `frappe.get_query_builder` | `(type_of_db: str)` | Return the query builder class for `mariadb` or `postgres`. |

## Messages and errors

| Function | Signature | Description |
|---|---|---|
| `frappe.msgprint` | `(msg, title=None, raise_exception=False, as_table=False, as_list=False, indicator=None, alert=False, primary_action=None, is_minimizable=False, wide=False, *, realtime=False) -> None` | Queue a message for the user. It is sent in `__server_messages` of the response. `indicator` is `blue`, `green`, `orange`, `red` or `yellow`. |
| `frappe.throw` | `(msg, exc=ValidationError, title=None, is_minimizable=False, wide=False, as_list=False, primary_action=None) -> None` | Show a message and raise `exc`. |
| `frappe.clear_messages` | `()` | Clear all queued messages. |
| `frappe.clear_last_message` | `()` | Remove the last queued message. |
| `frappe.get_message_log` | `() -> list[dict]` | Return the queued messages. |
| `frappe.log_error` | `(title=None, message=None, reference_doctype=None, reference_name=None, *, defer_insert=False)` | Make an Error Log. Without `message`, logs the current traceback. Defined in `frappe/utils/error.py`. |
| `frappe.get_traceback` | `(with_context: bool = False) -> str` | Return the traceback of the current exception. |
| `frappe.respond_as_web_page` | `(title, html, success=None, http_status_code=None, context=None, indicator_color=None, primary_action='/', primary_label=None, fullpage=False, width=None, template='message')` | Send the response as an HTML message page, not JSON. |
| `frappe.redirect_to_message` | `(title, html, http_status_code=None, context=None, indicator_color=None)` | Redirect to `/message?id=...` and show a message page. |
| `frappe.redirect` | `(url)` | Raise `frappe.Redirect`, which sends a 301 redirect to `url`. |

Exception classes such as `frappe.ValidationError`, `frappe.PermissionError`, `frappe.DoesNotExistError`, `frappe.DuplicateEntryError`, `frappe.MandatoryError`, `frappe.LinkValidationError` and `frappe.TimestampMismatchError` come from `frappe/exceptions.py`.

## Permissions

See [Permissions](../../developer-guide/security/permissions.md).

| Function | Signature | Description |
|---|---|---|
| `frappe.has_permission` | `(doctype=None, ptype='read', doc=None, user=None, throw=False, *, parent_doctype=None, debug=False, ignore_share_permissions=False)` | Return `True` if the user has permission `ptype` on the DocType or document. With `throw=True`, raise `PermissionError`. |
| `frappe.has_website_permission` | `(doc=None, ptype='read', user=None, verbose=False, doctype=None)` | Check website (portal) permission through the `has_website_permission` hook. |
| `frappe.only_for` | `(roles: list[str] \| tuple[str] \| str, message=False)` | Raise `PermissionError` if the user has none of the roles. |
| `frappe.only_has_select_perm` | `(doctype, user=None, ignore_permissions=False)` | Return `True` if the user has only `select` permission on the DocType. |
| `frappe.build_match_conditions` | `(doctype, as_condition=True)` | Return user permission conditions for a DocType as SQL or a list. |

## Whitelisting and request handling

See [Whitelisted Methods](../../developer-guide/api/whitelisted-methods.md).

| Function | Signature | Description |
|---|---|---|
| `frappe.whitelist` | `(allow_guest=False, xss_safe=False, methods=None)` | Decorator. Makes a function callable at `/api/method/<dotted.path>`. |
| `frappe.is_whitelisted` | `(method)` | Raise `PermissionError` if the function is not whitelisted (or not guest-allowed for Guest). |
| `frappe.override_whitelisted_method` | `(original_method: str) -> str` | Return the last override from the `override_whitelisted_methods` hook, or the original path. |
| `frappe.validate_and_sanitize_search_inputs` | `(fn)` | Decorator for custom Link search queries. Validates and cleans the search arguments. |
| `frappe.read_only` | `()` | Decorator. Marks a function as read-only so its reads can go to the replica (`read_from_replica`). |
| `frappe.write_only` | `()` | Decorator. Switches back to the primary connection while the function runs. |
| `frappe.call` | `(fn: str \| Callable, *args, **kwargs)` | Call a function by dotted path or object. Drops kwargs the function does not accept. |
| `frappe.get_newargs` | `(fn: Callable, kwargs: dict) -> dict` | Return only the kwargs that `fn` accepts. `frappe.call` uses it. |
| `frappe.get_attr` | `(method_string: str)` | Return the Python object for a dotted path. |

## Hooks, apps and modules

See [Hooks](../../developer-guide/building-apps/hooks.md) and [Apps](../../developer-guide/building-apps/apps.md).

| Function | Signature | Description |
|---|---|---|
| `frappe.get_hooks` | `(hook: str \| None = None, default='_KEEP_DEFAULT_LIST', app_name=None) -> _dict` | Return merged hook values from all installed apps, or the value of one hook. |
| `frappe.get_doc_hooks` | `()` | Return the merged `doc_events` hook as a dict per DocType. |
| `frappe.get_installed_apps` | `(*, _ensure_on_bench=False) -> list[str]` | Return apps installed on the current site. |
| `frappe.get_all_apps` | `(with_internal_apps=True, sites_path=None)` | Return all apps in `sites/apps.txt`. |
| `frappe.get_app_path` | `(app_name, *joins)` | Return the path of the app's Python package. |
| `frappe.get_app_source_path` | `(app_name, *joins)` | Return the path of the app repository. |
| `frappe.get_module_path` | `(module, *joins)` | Return the path of a module folder. |
| `frappe.get_pymodule_path` | `(modulename, *joins)` | Return the path of a Python module. |
| `frappe.get_module_list` | `(app_name)` | Return the modules in the app's `modules.txt`. |
| `frappe.get_module` | `(modulename)` | Import and return a Python module. |
| `frappe.get_active_domains` | `()` | Return active domains from Domain Settings. |
| `frappe.clear_cache` | `(user: str \| None = None, doctype: str \| None = None)` | Clear the cache of a user, a DocType, or the whole site. |
| `frappe.reset_metadata_version` | `()` | Reset the client build hash so desks reload metadata. |
| `frappe.get_desk_link` | `(doctype, name, show_title_with_name=False, open_in_new_tab=False)` | Return an HTML link to the document in desk. |

## Background jobs and realtime

See [Background Jobs](../../developer-guide/runtime/background-jobs.md) and [Realtime](../../developer-guide/runtime/realtime.md).

| Function | Signature | Description |
|---|---|---|
| `frappe.enqueue` | see below | Put a function on an RQ queue. Calls `frappe.utils.background_jobs.enqueue`. |
| `frappe.enqueue_doc` | `(doctype, name=None, method=None, queue='default', timeout=300, now=False, **kwargs)` | Enqueue a controller method of a document. |
| `frappe.task` | `(**task_kwargs)` | Decorator. Adds `.enqueue(**kwargs)` to the function, which calls `frappe.enqueue` with `task_kwargs`. |
| `frappe.publish_realtime` | `(event=None, message=None, room=None, user=None, doctype=None, docname=None, task_id=None, after_commit=False)` | Send a Socket.IO event to the browser. Calls `frappe/realtime.py`. |
| `frappe.publish_progress` | `(percent, title=None, doctype=None, docname=None, description=None, task_id=None)` | Send a progress event. Desk shows a progress bar. |

```python
# frappe/utils/background_jobs.py
frappe.enqueue(
	method: str | Callable, queue: str = "default", timeout: int | None = None, event=None,
	is_async: bool = True, job_name: str | None = None, now: bool = False, enqueue_after_commit: bool = False,
	*, on_success: Callable | None = None, on_failure: Callable | None = None, at_front: bool = False,
	job_id: str | None = None, deduplicate=False, **kwargs,
) -> Job | Any
```

`job_name` is deprecated. Use `job_id`. `is_async=False` outside of tests is deprecated. Use `now=True`.

## Caching

See [Caching](../../developer-guide/runtime/caching.md).

| Function | Signature | Description |
|---|---|---|
| `frappe.request_cache` | `(func: Callable) -> Callable` | Decorator. Caches return values for the rest of the request. Defined in `frappe/utils/caching.py`. |
| `frappe.local_cache` | `(namespace, key, generator, regenerate_if_none=False)` | Cache a value in `frappe.local.cache[namespace][key]` for the request. Calls `generator()` on a miss. |

`site_cache` and `redis_cache` are not on `frappe` in v15. Import them from `frappe.utils.caching`. See [Utilities](utils.md).

## Email, print and templates

See [Email and Notifications](../../developer-guide/output/email-and-notifications.md) and [Print Formats](../../developer-guide/output/print-formats.md).

| Function | Signature | Description |
|---|---|---|
| `frappe.sendmail` | see below | Queue an email through the default outgoing Email Account. Returns the Email Queue document. |
| `frappe.are_emails_muted` | `()` | Return `True` if `frappe.flags.mute_emails` or the `mute_emails` config is set. |
| `frappe.get_print` | `(doctype=None, name=None, print_format=None, style=None, as_pdf=False, doc=None, output=None, no_letterhead=0, password=None, pdf_options=None, letterhead=None, pdf_generator=None)` | Return the print HTML of a document, or PDF bytes with `as_pdf=True`. `pdf_generator` is `wkhtmltopdf` (default) or `chrome`. Defined in `frappe/utils/print_utils.py`. |
| `frappe.attach_print` | `(doctype, name, file_name=None, print_format=None, style=None, html=None, doc=None, lang=None, print_letterhead=True, password=None, letterhead=None)` | Return an attachment dict (`fname`, `fcontent`) of the print, for `sendmail`. |
| `frappe.render_template` | `(template, context=None, is_path=None, safe_render=True, *, restrict_globals=None)` | Render a Jinja template string or path. |
| `frappe.get_template` | `(path)` | Return a Jinja template object for a path. |
| `frappe.get_jenv` | `(*, restrict_globals=None)` | Return the Jinja environment with Frappe globals and filters. |
| `frappe.get_email_from_template` | `(name, args)` | Render `templates/emails/<name>.html` and `.txt`. Returns `(html, text)`. |
| `frappe.format_value` | `(value, df=None, doc=None, currency=None, translated=False, format=None)` | Format a value for display by fieldtype. Calls `frappe/utils/formatters.py`. |
| `frappe.format` | same as `format_value` | Alias of `format_value`. |

```python
# frappe/__init__.py
frappe.sendmail(
	recipients=None, sender="", subject="No Subject", message="No Message", as_markdown=False, delayed=True,
	reference_doctype=None, reference_name=None, unsubscribe_method=None, unsubscribe_params=None,
	unsubscribe_message=None, add_unsubscribe_link=1, attachments=None, content=None, doctype=None, name=None,
	reply_to=None, queue_separately=False, cc=None, bcc=None, message_id=None, in_reply_to=None, send_after=None,
	expose_recipients=None, send_priority=1, communication=None, retry=1, now=None, read_receipt=None,
	is_notification=False, inline_images=None, template=None, args=None, header=None, print_letterhead=False,
	with_container=False, email_read_tracker_url=None, x_priority: Literal[1, 3, 5] = 3, email_headers=None,
	redact_message_after_send=False,
) -> EmailQueue | None
```

## Translation

See [Translations](../../developer-guide/building-apps/translations.md).

| Function | Signature | Description |
|---|---|---|
| `frappe._` | `(msg: str, lang: str \| None = None, context: str \| None = None) -> str` | Return the translated string in the current language. |
| `frappe._lt` | `(msg: str, lang=None, context=None)` | Return a lazy string. It translates when converted to `str`. Use it at module level. |
| `frappe.N_` | `(msg: str, context: str \| None = None) -> str` | Mark a string for extraction. Returns it untranslated. |
| `frappe.set_user_lang` | `(user: str, user_language: str \| None = None) -> None` | Set `frappe.local.lang` for the user. |

## Strings, JSON and files

`cint`, `cstr` and `sbool` come from `frappe.utils.data`. The others are defined in `frappe/__init__.py` and are not on `frappe.utils` in v15. See [Utilities](utils.md) for the rest.

| Function | Signature | Description |
|---|---|---|
| `frappe.as_json` | `(obj: dict \| list, indent=1, separators=None, ensure_ascii=True) -> str` | Return a JSON string. Handles dates, decimals and documents. |
| `frappe.parse_json` | `(val)` | Parse a JSON string into a `_dict`. Returns other values as they are. |
| `frappe.cint` | `(s: int \| float \| str, default: int = 0) -> int` | Convert to integer. |
| `frappe.cstr` | `(s, encoding='utf-8')` | Convert to string. `None` becomes `""`. |
| `frappe.sbool` | `(x: str) -> bool \| Any` | Convert `"true"`, `"false"`, `"1"`, `"0"` strings to bool. |
| `frappe.scrub` | `(txt: str) -> str` | `Sales Order` becomes `sales_order`. |
| `frappe.unscrub` | `(txt: str) -> str` | `sales_order` becomes `Sales Order`. |
| `frappe.bold` | `(text)` | Wrap text in `<strong>` tags. |
| `frappe.as_unicode` | `(text, encoding: str = 'utf-8') -> str` | Convert bytes to `str`. |
| `frappe.safe_encode` | `(param, encoding='utf-8')` | Encode `str` to bytes. |
| `frappe.safe_decode` | `(param, encoding='utf-8', fallback_map: dict \| None = None)` | Decode bytes to `str`. |
| `frappe.safe_eval` | `(code, eval_globals=None, eval_locals=None)` | Evaluate an expression with restricted globals. |
| `frappe.generate_hash` | `(txt: str \| None = None, length: int = 56) -> str` | Return a random hex string. `txt` is not used. |
| `frappe.mock` | `(type, size=1, locale='en')` | Return fake data from the `faker` package. |
| `frappe.read_file` | `(path, raise_not_found=False, as_base64=False)` | Return the content of a file. |
| `frappe.get_file_json` | `(path)` | Read a file and parse it as JSON. |
| `frappe.get_file_items` | `(path, raise_not_found=False, ignore_empty_lines=True)` | Return the lines of a text file as a list. |
| `frappe.create_folder` | `(path, with_init=False)` | Make a folder. With `with_init=True`, add `__init__.py`. |

## Tests

| Function | Signature | Description |
|---|---|---|
| `frappe.get_test_records` | `(doctype)` | Return the records in `test_records.json` of the DocType folder, or `[]`. See [Testing](../../developer-guide/testing-and-performance/testing.md). |

## Deprecated

`frappe/utils/deprecations.py` has the `deprecated` decorator and `deprecation_warning()`. These calls on this page or nearby warn in v15:

| Call | Replacement |
|---|---|
| `frappe.enqueue(..., job_name=...)` | Pass `job_id`. |
| `frappe.enqueue(..., is_async=False)` outside tests | Pass `now=True`. |
| `frappe.db.set_value` on a Single DocType | `frappe.db.set_single_value`. See [Database API](database-api.md). |
| `super(Document, self).__init__(d)` in a controller | `self.init_from_dict(d)`. |

## Related

- [Database API](database-api.md)
- [Utilities](utils.md)
- [JavaScript API](js-api.md)
- [Hooks reference](../configuration/hooks.md)
