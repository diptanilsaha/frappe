---
title: frappe namespace
description: Every public name on the frappe Python module in v16 with signature, defaults, return value and side effects, grouped by topic.
version: v16
sources:
  - frappe/__init__.py
  - frappe/exceptions.py
  - frappe/model/document.py
  - frappe/model/delete_doc.py
  - frappe/model/rename_doc.py
  - frappe/model/meta.py
  - frappe/model/qb_query.py
  - frappe/client.py
  - frappe/utils/messages.py
  - frappe/utils/error.py
  - frappe/utils/translations.py
  - frappe/utils/caching.py
  - frappe/utils/local.py
  - frappe/utils/jinja.py
  - frappe/utils/print_utils.py
  - frappe/utils/formatters.py
  - frappe/utils/background_jobs.py
  - frappe/utils/typing_validations.py
  - frappe/realtime/__init__.py
  - frappe/email/__init__.py
  - frappe/config.py
  - frappe/cache_manager.py
  - frappe/core/doctype/system_settings/system_settings.py
  - frappe/concurrency_limiter.py
  - frappe/deprecation_dumpster.py
  - frappe/app.py
  - frappe/handler.py
---

# frappe namespace

This page lists every public name on the `frappe` module in v16 (`frappe.__version__` is `16.x`). Use it to look up an exact signature, default or side effect. The list comes from `dir(frappe)` on a v16 site, with signatures read by `inspect.signature`. For the `Document` class, see [Document](document.md). For `frappe.db`, see [Database](database.md). For `frappe.utils`, see [Utilities](utils.md).

## How names are loaded

- `frappe/__init__.py` imports every name when `frappe` is imported. There is no lazy loading.
- Names marked with a "Defined in" module other than `frappe` are re-exports. Import them from either place.
- Request-scoped objects (`frappe.db`, `frappe.session`, `frappe.form_dict` ...) are `LocalProxy` objects on `frappe.local`. They are valid only after `frappe.init()` and, for `db`, `frappe.connect()`.
- `frappe.*` re-exports every class from `frappe/exceptions.py` (`from .exceptions import *`) and every public name from `frappe/utils/messages.py`.

## Context objects

| Name | Type | Description |
|---|---|---|
| `frappe.local` | `Local` | Thread-local and request-local storage. Every object below is an attribute of it. `release_local(local)` clears it. |
| `frappe.db` | `LocalProxy` to `MariaDBDatabase`, `PostgresDatabase` or `SQLiteDatabase` | Connection of the current site. Set by `frappe.connect()`. See [Database](database.md). |
| `frappe.qb` | `LocalProxy` to `MariaDB`, `Postgres` or `SQLite` | Query builder for `conf.db_type`. Set by `frappe.init()`. See [Query Builder](query-builder.md). |
| `frappe.conf` | `LocalProxy` (`_dict`) | `site_config.json` merged over `common_site_config.json`. Set by `frappe.init()`. |
| `frappe.session` | `LocalProxy` (`_dict`) | Current session. `frappe.session.user` is the user name. `frappe.session.sid` is the session id. `frappe.session.data` holds session data. Default user after `init` is `Guest`. |
| `frappe.form_dict` | `LocalProxy` (`_dict` of `str`) | Request arguments (query string, form body, JSON body). Reset by `frappe.set_user()`. |
| `frappe.form` | `LocalProxy` | Alias of `frappe.form_dict`. |
| `frappe.request` | `LocalProxy` to `werkzeug.wrappers.Request` | The HTTP request. Not set in console, jobs or tests. |
| `frappe.response` | `LocalProxy` (`_dict`) | Response body. Starts as `{"docs": []}`. A whitelisted method's return value goes to `frappe.response["message"]`. |
| `frappe.flags` | `LocalProxy` (`_dict`) | Request-scoped flags. See [Global flags](#global-flags). |
| `frappe.lang` | `LocalProxy` (`str`) | Language of the current request. Default `conf.lang` or `"en"`. |
| `frappe.job` | `LocalProxy` (`_dict`) | Metadata of the running background job. Not set outside a worker. |
| `frappe.message_log` | `LocalProxy` (`list`) | Messages added by `frappe.msgprint`. Sent as `_server_messages`. |
| `frappe.error_log` | `LocalProxy` (`list`) | Entries added by `frappe.errprint`. Sent as `exc`. |
| `frappe.debug_log` | `LocalProxy` (`list`) | Lines added by `frappe.log`. |
| `frappe.user` | `LocalProxy` | Kept for compatibility. `frappe.init()` sets it to `None` and `set_user` does not change it. Use `frappe.session.user`. |
| `frappe.cache` | `RedisWrapper` | Redis cache client, global to the process, keys prefixed per site. See [Jobs, Cache and Realtime](jobs-cache-and-realtime.md). |
| `frappe.client_cache` | `ClientCache` | In-process cache in front of Redis for values that rarely change. |
| `frappe.in_test` | `bool` | `True` while tests run. Set by `frappe.tests.utils.toggle_test_mode()`. |
| `frappe.STANDARD_USERS` | `tuple` | `("Guest", "Administrator")`. |
| `frappe.SITE_NAME_PATTERN` | `re.Pattern` | `^[a-zA-Z0-9._-]+$`. `frappe.init` raises `ValueError` for other site names. |
| `frappe.__version__` | `str` | The v16 release, for example `"16.60.2"`. |
| `frappe.__title__` | `str` | `"Frappe Framework"`. |
| `frappe.controllers` | `dict` | Controller class cache, per site. |
| `frappe.lazy_controllers` | `dict` | Lazy controller class cache, per site. Used by `get_lazy_doc`. |
| `frappe.loggers` | `dict` | Logger cache used by `frappe.logger`. |
| `frappe.log_level` | `int \| None` | Log level for `frappe.logger`. Default `None`. |
| `frappe.whitelisted` | `set` | Functions decorated with `@frappe.whitelist`. |
| `frappe.guest_methods` | `set` | Whitelisted functions with `allow_guest=True`. |
| `frappe.xss_safe_methods` | `set` | Guest functions with `xss_safe=True`. |
| `frappe.allowed_http_methods_for_whitelisted_func` | `dict` | Function to list of allowed HTTP methods. |

### Values on frappe.local

`frappe.init()` sets these attributes. Read them as `frappe.local.<name>`.

| Attribute | Value after `init` |
|---|---|
| `site`, `site_name` | Site name. |
| `sites_path`, `site_path` | Path of the `sites` folder and of the site folder. |
| `conf`, `lang`, `flags`, `form_dict`, `session`, `response`, `qb` | See the table above. |
| `response_headers` | Empty `werkzeug.datastructures.Headers`. |
| `cache` | Empty `dict`. Request-scoped cache. Reset by `set_user`. |
| `request_cache` | `defaultdict(dict)`. Used by `@frappe.request_cache`. |
| `locked_documents` | Documents locked by `doc.lock()` in this request. |
| `request_ip`, `task_id`, `user`, `user_perms`, `all_apps` | `None`. |
| `role_permissions`, `valid_columns`, `new_doc_templates` | Empty `dict`. Per-request caches. |
| `app_modules`, `module_app` | App to module list and module to app maps. Set by `setup_module_map`. |
| `initialised` | `True`. A second `init` call returns early unless `force=True`. |

### Global flags

`frappe.init()` sets these keys on `frappe.flags`. Other code sets more keys during migrate, install and import.

| Flag | Default | Meaning |
|---|---|---|
| `currently_saving` | `[]` | `(doctype, name)` pairs inside a save. `db_set` does not change `modified` for these. |
| `redirect_location` | `""` | Target of `frappe.redirect`. |
| `in_install_db`, `in_import`, `new_site` | `False` | Set during database install, data import and new site creation. |
| `in_test` | `frappe.in_test` | `True` in tests. |
| `mute_messages` | `False` | `msgprint` does not queue messages. It still raises when asked to. |
| `mute_emails` | `False` | `frappe.are_emails_muted()` returns truthy. |
| `ignore_links` | `False` | Global flag key. Document Link checks use `doc.flags.ignore_links`. |
| `has_dataurl` | `False` | Set when a Text Editor value had a data URL. |
| `read_only` | `False` | Request runs in read-only mode. `log_error` and `add_viewed` use deferred inserts. |

Keys set elsewhere and read by `Document`: `in_migrate`, `in_install`, `in_patch`, `in_setup_wizard`, `print_messages` (print `msgprint` output to the console), `error_message` (message for the next `PermissionError`).

## Site, connection and user

| Function | Description |
|---|---|
| `init(site: str, sites_path: str = '.', new_site: bool = False, force: bool = False, *, is_request=False, is_job=False) -> None` | Set up `frappe.local` for a site: config, flags, session as `Guest`, query builder, Redis connection, module map. Does nothing if already initialised, unless `force=True`. |
| `connect(site: str \| None = None, db_name: str \| None = None, set_admin_as_user: bool = True) -> None` | Open the database connection in `frappe.local.db`. Sets the user to `Administrator` by default. `site` and `db_name` are deprecated. |
| `connect_replica() -> bool` | Open a connection to the read replica from `replica_host` and swap it into `frappe.db`. The primary stays in `frappe.local.primary_db`. Returns `False` if already swapped. |
| `destroy()` | Close the database connection and release `frappe.local`. |
| `init_site(site=None)` | Context manager. Calls `init(site)` on enter and `destroy()` on exit. `site=None` loads only `common_site_config.json`. |
| `setup_redis_cache_connection()` | Set `frappe.cache` and `frappe.client_cache`. Called by `init`. |
| `setup_module_map(include_all_apps: bool = True) -> None` | Build `local.app_modules` and `local.module_app`. Cached in Redis. |
| `set_user(username: str)` | Set `frappe.session.user` and `sid`. Clears `local.cache`, `form_dict`, session data, role permission and user permission caches. No permission check. |
| `get_user()` | Return the `UserPermissions` object of the session user. Cached in `local.user_perms`. |
| `get_roles(username=None) -> list[str]` | Return the roles of a user. Default is the session user. Returns `["Guest"]` when no session exists. |
| `get_request_header(key, default=None)` | Return an HTTP request header. Fails outside a request. |
| `get_site_config(sites_path: str \| None = None, site_path: str \| None = None, *, cached=False) -> _dict` | Return `site_config.json` merged over `common_site_config.json`. Defined in `frappe.config`. |
| `get_common_site_config(sites_path: str \| None = None, cached=False) -> _dict` | Return `common_site_config.json`. Defined in `frappe.config`. |
| `get_conf(site: str \| None = None) -> _dict` | Return `frappe.local.conf` if set. Else load the config of `site`, or the common config. Defined in `frappe.config`. |
| `get_system_settings(key: str)` | Return one System Settings value. Caches the document in `frappe.local.system_settings` and `client_cache`. Defined in `frappe.core.doctype.system_settings.system_settings`. |
| `get_website_settings(key)` | Return one Website Settings value. Caches the document in `frappe.local.website_settings`. |
| `get_site_path(*joins)` | Return `frappe.local.site_path` joined with `joins`. |
| `is_setup_complete()` | Return `True` if `is_setup_complete` is set on the `Installed Application` rows of `frappe` and `erpnext`. Cached per request. |
| `ping()` | Return `"pong"`. Whitelisted, guest allowed. |

## Documents

See [Document](document.md) for the methods of the returned objects and [Controllers](../../developer-guide/data-model/controllers.md) for the lifecycle.

| Function | Description |
|---|---|
| `get_doc(doctype, name, /, *, for_update=None, check_permission=None)` | Load a document and its child rows from the database. `name` can be a filters dict. Raises `DoesNotExistError`. No permission check unless `check_permission` is set (`True` checks `read`, a string checks that permission type). |
| `get_doc(doctype, /)` | Load a Single DocType. |
| `get_doc(dict)` or `get_doc(**kwargs)` | Make a new unsaved document. The dict must have `doctype`. Prefer `new_doc`. |
| `get_doc(document)` | Return the same `Document` object. |
| `new_doc(doctype: str, *, parent_doc=None, parentfield=None, as_dict=False, **kwargs) -> Document` | Return a new unsaved document with defaults and user defaults set. `kwargs` set field values. |
| `get_lazy_doc(doctype: str, name: str, *, for_update=None, check_permission=None) -> Document` | Same as `get_doc`, but child tables load on first access. `DocType` loads eagerly with a warning. |
| `get_cached_doc(*args, **kwargs) -> Document` | Same as `get_doc`, but read from Redis if present. Stores the document for 3600 seconds. Do not change and save the returned object. |
| `get_last_doc(doctype, filters=None, order_by='creation desc', *, for_update=False)` | Return the newest document that matches `filters`. Raises `DoesNotExistError` if none. No permission check. |
| `get_single(doctype)` | Return `get_doc(doctype, doctype)`. |
| `copy_doc(doc: Document, ignore_no_copy: bool = True) -> Document` | Return an unsaved deep copy. Clears `name`, `owner`, `creation`, `modified`, `modified_by`, `amended_from`, `amendment_date`, and `docstatus` (not in tests). With `ignore_no_copy=False`, also clears `no_copy` fields. |
| `delete_doc(doctype=None, name=None, force=False, ignore_doctypes=None, for_reload=False, ignore_permissions=False, flags=None, ignore_on_trash=False, ignore_missing=True, delete_permanently=False)` | Delete one document or a list of names. See [delete_doc](#delete_doc). |
| `delete_doc_if_exists` | Alias of `delete_doc`. `ignore_missing` is already `True` by default. |
| `rename_doc(doctype: str, old, new, force: bool = False, merge: bool = False, *, ignore_if_exists: bool = False, show_alert: bool = True, rebuild_search: bool = True) -> str` | Rename a document and update all Link and Dynamic Link values. Checks `write` permission and `allow_rename` (unless `force`). Whitelisted: `POST`, `PUT`. See [rename_doc](document.md#frappemodelrename_doc). |
| `get_meta(doctype, cached: bool = True) -> Meta` | Return the DocType definition with Custom Fields and Property Setters applied. Defined in `frappe.model.meta`. |
| `get_meta_module(doctype)` | Return the Python module of the DocType controller. |
| `is_table(doctype: str) -> bool` | Return `True` for a child table DocType (`istable`). Cached in `client_cache`. |
| `get_doctype_app(doctype)` | Return the app that owns the DocType module. |
| `get_precision(doctype: str, fieldname: str, currency: str \| None = None, doc=None) -> int` | Return the precision of a numeric field. |
| `can_cache_doc(args) -> str \| None` | Return the cache key for `get_cached_doc` arguments, or `None` if they cannot be cached. |
| `get_document_cache_key(doctype: str, name: str)` | Return `document_cache::<doctype>::<name>`. |
| `clear_document_cache(doctype: str, name: str \| None = None) -> None` | Remove a document (or all documents of the DocType) from Redis and `frappe.db.value_cache`. Runs again after commit and rollback. |
| `reload_doc(module: str, dt=None, dn=None, force=False, reset_permissions=False)` | Import a DocType or record from its JSON file in `<module>/<dt>/<dn>/<dn>.json`. Use it in patches. `force` reloads even when `modified` matches. |
| `reload_doctype(doctype, force=False, reset_permissions=False)` | Reload a DocType from its JSON file. Reads the module from the database. |
| `import_doc(path)` | Import documents from a JSON file or folder through `frappe.core.doctype.data_import.data_import.import_doc`. |
| `make_property_setter(args, ignore_validate=False, validate_fields_for_doctype=True, is_system_generated=True, *, module=None)` | Insert a Property Setter. `args` keys: `doctype`, `fieldname`, `property`, `value`, `property_type`, `doctype_or_field`, `row_name`. Without `doctype`, makes one for every DocType that has `fieldname`. |

```python
doc = frappe.get_doc("Sample Order", "SO-00001", check_permission="write")
item = frappe.new_doc("Sample Item", item_name="Cable", category="Hardware", rate=12)
```

!!! note "Version"
    v16 has no `frappe.read_doc` and no `frappe.get_docs`. To send a document to a user, call `doc.check_permission("read")`, then `doc.apply_fieldlevel_read_permissions()`.

### delete_doc

`frappe.delete_doc` calls `frappe.model.delete_doc.delete_doc`. For each name it does this:

1. Return `False` if the record does not exist and `ignore_missing=True`. Raise `DoesNotExistError` if `ignore_missing=False`.
2. Lock the row with `SELECT ... FOR UPDATE NOWAIT`.
3. Check `delete` permission and throw if the document is submitted. Skipped with `for_reload=True`.
4. Run `on_trash` and `on_change` (with `doc.flags.in_delete`). Skipped with `ignore_on_trash=True`.
5. Check that no document links to it. Skipped with `force=True`. Raises `LinkExistsError`.
6. Delete the row and child rows, run `after_delete`, delete attachments, passwords, global search entries and tags.
7. Add a `Deleted Document` unless `delete_permanently=True` or `for_reload=True`.
8. Enqueue `delete_dynamic_links` after commit (runs at once in tests), publish `doc_update` and `list_update`, and insert a feed entry.

| Parameter | Default | Description |
|---|---|---|
| `doctype` | `None` | DocType. If empty, read `dt` and `dn` from `frappe.form_dict`. |
| `name` | `None` | Name, or a list of names. |
| `force` | `False` | Skip the link check. Also allows deleting a standard DocType. |
| `ignore_doctypes` | `None` | Child DocTypes whose rows are not deleted. |
| `for_reload` | `False` | Used when a DocType is reloaded. Runs `before_reload`, skips permission and `on_trash`, deletes permanently. |
| `ignore_permissions` | `False` | Skip the `delete` permission check. |
| `flags` | `None` | Dict copied to `doc.flags`. `doc.delete()` passes its own flags. |
| `ignore_on_trash` | `False` | Skip `on_trash` and `on_change`. |
| `ignore_missing` | `True` | Return `False` instead of raising for a missing record. |
| `delete_permanently` | `False` | Do not keep a `Deleted Document` copy. |

!!! warning
    With a list of names, `delete_doc` returns `False` at the first missing name. Names after it in the list are not deleted.

## Queries

`frappe.get_list` and `frappe.get_all` call `frappe.model.qb_query.DatabaseQuery(doctype).execute()`. For filter syntax, see [Database](../../developer-guide/database/database.md). For `frappe.db.get_value` and other database methods, see [Database](database.md).

| Function | Description |
|---|---|
| `get_list(doctype, *args, **kwargs)` | Return rows as a list of `_dict`. Applies role permissions, user permissions and sharing of `user` (default session user). No default row limit. The docstring says 20, but the code applies no limit. |
| `get_all(doctype, *args, **kwargs)` | Same as `get_list` with `ignore_permissions=True`. Sets `limit_page_length=0` if not given. |
| `get_value(*args, **kwargs)` | Alias of `frappe.db.get_value`. No permission check. |
| `get_cached_value(doctype: str, name: str \| dict, fieldname: str \| Iterable[str] = 'name', as_dict: bool = False) -> Any` | Return field values from `get_cached_doc`. Returns `None` if the document does not exist. `as_dict=True` needs a list of fieldnames. |
| `get_single_value(setting: str, fieldname: str, /, *, as_dict: bool = False)` | Return a cached value of a Single DocType. |
| `set_value(doctype, docname, fieldname, value=None)` | Call `frappe.client.set_value`: load the document, update it, run `save()` with all validations and permission checks. `fieldname` can be a dict. |
| `get_query(*args, **kwargs) -> QueryBuilder` | Build a query from `table`, `fields`, `filters` and other options. Defined in `frappe.query_builder.utils`. See [Query Builder](query-builder.md). |
| `get_query_builder(type_of_db: str)` | Return the query builder class for `mariadb`, `postgres` or `sqlite`. |
| `build_match_conditions(doctype, as_condition=True)` | Return the user permission conditions of a DocType as SQL, or as a list. |

### get_list arguments

| Argument | Type | Default | Description |
|---|---|---|---|
| `fields` | `list \| tuple \| str` | `[pluck or "name"]` | Columns. Use the dict form for SQL functions, for example `{"COUNT": "*"}`. Child fields as `"items.qty"`. |
| `filters` | `dict \| list` | `None` | AND conditions. |
| `or_filters` | `dict \| list` | `None` | OR conditions. |
| `group_by` | `str` | `None` | Group by fields. |
| `order_by` | `str` | DocType `sort_field` and `sort_order` | Order by fields. |
| `limit` | `int` | `None` | Maximum rows. `None` or `0` returns all rows. |
| `offset` | `int` | `None` | Rows to skip. |
| `as_list` | `bool` | `False` | Return tuples, not dicts. |
| `pluck` | `str` | `None` | Return a flat list of one field. |
| `distinct` | `bool` | `False` | `SELECT DISTINCT`. |
| `ignore_permissions` | `bool` | `False` | Skip permission checks. `get_all` sets it. |
| `ignore_user_permissions` | `bool` | `False` | Skip only User Permissions. |
| `user` | `str` | session user | Check permissions of this user. |
| `parent_doctype` | `str` | `None` | Parent DocType, needed to query a child table with permissions. |
| `reference_doctype` | `str` | `None` | DocType whose link field context applies to user permissions. |
| `with_comment_count` | `bool` | `False` | Add `_comment_count` to each row. |
| `update` | `dict` | `None` | Merge this dict into each row. |
| `run` | `bool` | `True` | `False` returns the query object, not rows. |
| `debug` | `bool` | `False` | Print the query. |
| `save_user_settings`, `save_user_settings_fields`, `user_settings` | | `False`, `False`, `None` | Save list settings of the user. Used by the list view. |
| `ignore_ddl` | `bool` | `False` | Return `[]` if the table does not exist. |
| `limit_start`, `limit_page_length`, `start`, `page_length` | `int` | `0`, `None`, `None`, `None` | Deprecated aliases of `offset` and `limit`. They warn when set. |
| `with_childnames`, `join`, `ignore_ifnull`, `strict` | | | Accepted for compatibility. They do nothing. |

The first two arguments are swapped when `fields` looks like filters (a dict, or a list of lists).

```python
frappe.get_list("Sample Order", filters={"status": "Draft"}, fields=["name", "total"], order_by="creation desc", limit=20)
frappe.get_all("Sample Item", filters={"disabled": 0}, pluck="name")
frappe.get_list("Sample Order", fields=["status", {"COUNT": "*", "as": "count"}], group_by="status")
# [{'status': 'Completed', 'count': 1}, {'status': 'Draft', 'count': 2}, {'status': 'Open', 'count': 3}]
```

A function written as a string in `fields`, such as `"count(name) as c"`, raises `frappe.ValidationError`.

## Messages and errors

| Function | Description |
|---|---|
| `msgprint(msg, title=None, raise_exception=False, as_table=False, as_list=False, indicator=None, alert=False, primary_action=None, is_minimizable=False, wide=False, *, realtime=False, allow_dangerous_html=False) -> None` | Queue a message dialog for the user in `frappe.message_log`. See the parameter table below. |
| `throw(msg, exc=ValidationError, title=None, is_minimizable=False, wide=False, as_list=False, primary_action=None, *, allow_dangerous_html=False) -> None` | Call `msgprint` with `raise_exception=exc` and a red indicator. Always raises. |
| `toast(message: str, indicator=None)` | Call `msgprint(message, indicator=indicator, alert=True)`. Desk shows a short alert. |
| `throw_permission_error()` | Throw `PermissionError` with the message "Not permitted". |
| `clear_messages()` | Empty `frappe.message_log`. |
| `clear_last_message()` | Remove the last message from `frappe.message_log`. |
| `get_message_log() -> list[dict]` | Return a copy of `frappe.message_log`. |
| `log_error(title=None, message=None, reference_doctype=None, reference_name=None, *, defer_insert=False) -> ErrorLog` | Insert an `Error Log`. See below. Defined in `frappe.utils.error`. |
| `get_traceback(with_context: bool = False) -> str` | Return the traceback of the current exception. Defined in `frappe.utils`. |
| `errprint(msg: str) -> None` | Add `{"exc": msg}` to `frappe.error_log`. Prints it outside a request or in developer mode. |
| `log(msg: str) -> None` | Print to stderr and add to `frappe.debug_log`. |
| `logger(module=None, with_more_info=False, allow_site=True, filter=None, max_size=100000, file_count=20) -> Logger` | Return a cached Python logger that writes to `logs/<module>.log` in the bench and, with `allow_site`, to `<site>/logs/<module>.log`. |
| `print_sql(enable: bool = True) -> None` | Print every SQL query. Works only with `allow_tests` in site config. Stored in `client_cache`. |
| `respond_as_web_page(title, html, success=None, http_status_code=None, context=None, indicator_color=None, primary_action='/', primary_label=None, fullpage=False, width=None, template='message')` | Send the response as an HTML message page, not JSON. |
| `redirect_to_message(title, html, http_status_code=None, context=None, indicator_color=None)` | Store the message in Redis for 60 seconds. Redirect to `/message?id=<key>`, or return that URL in an Ajax request. |
| `redirect(url)` | Set `frappe.flags.redirect_location` and raise `frappe.Redirect` (301). |

### msgprint parameters

| Parameter | Default | Description |
|---|---|---|
| `msg` | | String, list (with `as_list`) or list of lists (with `as_table`). |
| `title` | `"Message"` | Dialog title. |
| `raise_exception` | `False` | `True` raises `ValidationError`. An exception class or instance raises that exception with `msg`. |
| `as_table`, `as_list` | `False` | Render a list of lists as a table, or a list as a bullet list. |
| `indicator` | `None` | `blue`, `green`, `orange`, `red` or `yellow`. Red if `raise_exception` is set. |
| `alert` | `False` | Show a toast, not a dialog. |
| `primary_action` | `None` | Dict for a button: `label`, and `server_action` or `client_action`, `args`, `hide_on_success`. |
| `is_minimizable`, `wide` | `False` | Dialog options. |
| `realtime` | `False` | Publish the message at once with the `msgprint` realtime event, not in the response. |
| `allow_dangerous_html` | `False` | Skip `clean_html` on the message. |

With `frappe.flags.mute_messages` set, `msgprint` queues nothing but still raises.

```python
frappe.throw(_("Rate must be positive"), title=_("Invalid Rate"))
frappe.msgprint(_("Order updated"), indicator="green", alert=True)
frappe.throw(_("Not allowed"), frappe.PermissionError)
```

### log_error

- `title` is the `method` field. `message` is the `error` field. Without `message`, Frappe logs the current traceback.
- If `title` contains a newline and `message` is set, the two are swapped.
- Inserts with `ignore_permissions=True`. Uses `deferred_insert` when `defer_insert=True`, in read-only mode, or in a read-only transaction.
- An error while the log is written is not caught. It goes to the caller.
- Prints the title and returns `None` if no database connection exists.

!!! warning
    `frappe.log_error(message="...")` without `title` raises `TypeError` in v16. Always pass `title`.

### Exceptions

All classes are in `frappe/exceptions.py` and on `frappe`. The HTTP status comes from `http_status_code`. A class without one inherits it from its base, or gives `500`.

| Exception | Base | HTTP |
|---|---|---|
| `ValidationError` | `Exception` | 417 |
| `FrappeTypeError` | `TypeError` | 417 |
| `AuthenticationError` | `Exception` | 401 (not written to Error Log) |
| `SessionExpired` | `Exception` | 401 |
| `PermissionError` | `Exception` | 403 |
| `DoesNotExistError(*args, doctype=None)` | `ValidationError` | 404 |
| `PageDoesNotExistError` | `ValidationError` | 404 |
| `NameError` | `Exception` | 409 |
| `DuplicateEntryError` | `NameError` | 409 |
| `LinkExpired` | `ValidationError` | 410 |
| `UnsupportedMediaType` | `Exception` | 415 |
| `TooManyRequestsError`, `RateLimitExceededError` | `Exception`, `ValidationError` | 429 |
| `CSRFTokenError` | `Exception` | 400 (not written to Error Log) |
| `InvalidAuthorizationHeader`, `InvalidAuthorizationPrefix`, `InvalidAuthorizationToken` | `CSRFTokenError` | 400 |
| `SessionBootFailed` | `ValidationError` | 500 |
| `OutgoingEmailError` | `Exception` | 501 |
| `SessionStopped`, `ServiceUnavailableError` | `Exception` | 503 |
| `InReadOnlyMode` | `ValidationError` | 503 (not written to Error Log) |
| `QueueOverloaded` | `ValidationError` | 503 |
| `RequestToken` | `Exception` | 200 |
| `Redirect(http_status_code: int = 301)` | `Exception` | 301 or the given code |

These inherit 417 from `ValidationError`: `DataError`, `MappingMismatchError`, `InvalidStatusError`, `MandatoryError`, `NonNegativeError`, `InvalidSignatureError`, `CannotChangeConstantError`, `CharacterLengthExceededError`, `UpdateAfterSubmitError`, `LinkValidationError`, `CancelledLinkError` (base `LinkValidationError`), `DocstatusTransitionError`, `TimestampMismatchError`, `EmptyTableError`, `LinkExistsError`, `InvalidEmailAddressError`, `InvalidNameError`, `InvalidPhoneNumberError`, `TemplateNotFoundError`, `UniqueValidationError`, `AppNotInstalledError`, `ImplicitCommitError`, `DocumentLockedError`, `CircularLinkingError`, `InvalidColumnName`, `IncompatibleApp`, `InvalidDates`, `DataTooLongException`, `DocumentAlreadyRestored`, `AttachmentLimitReached`, `PrintFormatError`, `InvalidDatabaseFile`.

These have no status code and give 500: `QueryTimeoutError`, `QueryDeadlockError`, `SiteNotSpecifiedError`, `DatabaseModificationError`, `UrlSchemeNotSupported`, `ImproperDBConfigurationError(reason, msg=None)`, `UnknownDomainError`, `RetryBackgroundJobError`, `SecurityException` (not written to Error Log), `FileAlreadyAttachedException`, `TooManyWritesError`, `InvalidRemoteException`, `CommandFailedError(message, out, err)`, `ExecutableNotFound` and `InvalidRoundingMethod` (base `FileNotFoundError`). `IncorrectSitePath` subclasses `werkzeug.exceptions.NotFound`. `frappe.NotFound` is the werkzeug class.

## Permissions

See [Permissions and Sessions](permissions-and-sessions.md) for `frappe.permissions` and [Permissions](../../developer-guide/security/permissions.md) for the concepts.

| Function | Description |
|---|---|
| `has_permission(doctype=None, ptype='read', doc=None, user=None, throw=False, *, parent_doctype=None, debug=False, ignore_share_permissions=False)` | Return `True` if `user` (default session user) has `ptype` on the DocType, or on `doc` (object or name). With `throw=True`, raise `PermissionError` and set `frappe.flags.error_message`. |
| `has_website_permission(doc=None, ptype='read', user=None, verbose=False, doctype=None)` | Check portal access. Calls the controller's `has_website_permission`, else every `has_website_permission` hook for the DocType. Returns `False` if neither exists. |
| `only_for(roles: list[str] \| tuple[str] \| str, message=False)` | Raise `PermissionError` if the session user has none of `roles`. `Administrator` always passes. With `message=True`, show a "This action is only allowed for" dialog. |
| `only_has_select_perm(doctype, user=None, ignore_permissions=False)` | Return `True` if the user has `select` but not `read` on the DocType. |

```python
@frappe.whitelist(methods=["POST"])
def close_order(name: str):
	doc = frappe.get_doc("Sample Order", name, check_permission="write")
	doc.db_set("status", "Closed")
```

## Whitelisting and calls

See [Whitelisted Methods](../../developer-guide/api/whitelisted-methods.md) and [REST Endpoints](../http/rest-endpoints.md).

| Function | Description |
|---|---|
| `whitelist(allow_guest=False, xss_safe=False, methods=None)` | Decorator. Makes a function callable at `/api/method/<dotted.path>`. See below. |
| `is_whitelisted(method)` | Raise `PermissionError` if `method` is not whitelisted, or is not guest-allowed for `Guest`. Raise `SessionExpired` when a whitelisted method is called as `Guest` after the session expired. For `Guest` and non-`xss_safe` methods, sanitize string values in `form_dict`. |
| `override_whitelisted_method(original_method: str) -> str` | Return the last path from the `override_whitelisted_methods` hook, or the original path. |
| `validate_and_sanitize_search_inputs(fn)` | Decorator for custom Link search query functions. Validates `searchfield`, converts `start` and `page_len` to `int`, and returns `[]` if `doctype` does not exist. |
| `read_only()` | Decorator. With `read_from_replica` in site config, run the function on the replica connection. Also sets `read_only = True` on the wrapper, which Flow reads. |
| `write_only()` | Decorator. Switch to the primary connection while the function runs, if a replica is active. |
| `call(fn: str \| Callable, *args, **kwargs)` | Call a function, or a dotted path, with only the keyword arguments it accepts. |
| `get_newargs(fn: Callable, kwargs: dict) -> dict` | Return `kwargs` without keys `fn` does not accept. Always removes `ignore_permissions` and `flags`. |
| `get_attr(method_string: str) -> Any` | Return the object at a dotted path. Throws `AppNotInstalledError` if the first part is not an installed app (except during install and uninstall). |

`@frappe.whitelist` details:

- `methods`: `None` allows `GET`, `POST`, `PUT` and `DELETE`. A list limits the methods.
- `allow_guest=True` lets `Guest` call it. Guest arguments are HTML-sanitized unless `xss_safe=True`.
- Arguments are converted and checked against the type annotations in a request or in tests (`frappe.utils.typing_validations.validate_argument_types`). Arguments without annotations are not checked.
- There is no `rate_limit` or `force_types` argument in v16. To rate limit, put `@frappe.rate_limiter.rate_limit(...)` below `@frappe.whitelist()`.

```python
# sample_app/api.py
import frappe

@frappe.whitelist(methods=["POST"])
def set_rate(item: str, rate: float) -> float:
	doc = frappe.get_doc("Sample Item", item, check_permission="write")
	doc.rate = rate
	doc.save()
	return doc.rate
```

## Hooks, apps and modules

See [Hooks](../../developer-guide/building-apps/hooks.md) and [Hooks reference](../configuration/hooks.md).

| Function | Description |
|---|---|
| `get_hooks(hook: str \| None = None, default='_KEEP_DEFAULT_LIST', app_name: str \| None = None) -> _dict` | Return all hooks merged from installed apps, or one hook. A missing hook gives `[]` unless `default` is passed. Cached in `client_cache` as `app_hooks`, or per site in developer mode, or per request with `app_name`. |
| `get_doc_hooks()` | Return `doc_events` keyed by DocType, with tuple keys expanded. Cached in `frappe.local.doc_events_hooks`. |
| `append_hook(target, key, value)` | Merge one hook value into a hooks dict. Lists are extended. Dicts are merged by key. |
| `get_installed_apps(*, _ensure_on_bench=False) -> list[str]` | Return apps installed on the site, from the `installed_apps` global. Cached per request. |
| `get_all_apps(with_internal_apps=True, sites_path=None)` | Return apps listed in `sites/apps.txt`, with `frappe` first. |
| `get_app_path(app_name, *joins)` | Return the path of the app Python package. |
| `get_app_source_path(app_name, *joins)` | Return the path of the app repository (`<app package>/..`). |
| `get_module_path(module, *joins)` | Return the path of a module folder. |
| `get_pymodule_path(modulename, *joins)` | Return the folder of a Python module. |
| `get_module_list(app_name)` | Return the module names in the app `modules.txt`. |
| `get_module(modulename: str)` | Import a module with `importlib.import_module`. |
| `get_domain_data(module)` | Return the `data` dict of a `domains` hook entry. |
| `get_active_domains()` | Return active domains from Domain Settings. |
| `clear_cache(user: str \| None = None, doctype: str \| None = None)` | Clear the cache of one user, one DocType, or the whole site. Defined in `frappe.cache_manager`. |
| `reset_metadata_version()` | Make a new client build hash so open desks reload metadata. Defined in `frappe.cache_manager`. |
| `get_desk_link(doctype, name, show_title_with_name=False, open_in_new_tab=False)` | Return an HTML `<a>` to `/desk/<doctype-slug>/<name>` with the DocType label and the title. |

```python
frappe.get_desk_link("Sample Item", "Laptop")
# '<a href="/desk/sample-item/Laptop" style="font-weight: bold;">Sample Item Laptop</a>'
```

## Translation

See [Translations](../../developer-guide/building-apps/translations.md).

| Function | Description |
|---|---|
| `_(msg: str, lang: str \| None = None, context: str \| None = None) -> str` | Return the translation in `lang` (default `frappe.local.lang`). Returns `msg` if none exists. |
| `_lt(msg: str, lang: str \| None = None, context: str \| None = None)` | Return a lazy object that translates when it is turned into a string. Use it at module level. |
| `N_(msg: str, context: str \| None = None) -> str` | Mark a string for extraction. Return it unchanged. |
| `set_user_lang(user: str, user_language: str \| None = None) -> None` | Set `frappe.local.lang` to the language of `user`, else to `user_language`. |

## Background jobs and realtime

See [Jobs, Cache and Realtime](jobs-cache-and-realtime.md) for all arguments and [Background Jobs](../../developer-guide/runtime/background-jobs.md) for the how-to.

| Function | Description |
|---|---|
| `enqueue(method, queue='default', timeout=None, event=None, is_async=True, job_name=None, now=False, enqueue_after_commit=False, *, on_success=None, on_failure=None, at_front=False, job_id=None, deduplicate=False, at_front_when_starved=False, **kwargs) -> Job \| Any` | Put `method` on an RQ queue with `kwargs`. `now=True` runs it in process and returns the result. Defined in `frappe.utils.background_jobs`. |
| `enqueue_doc(doctype, name=None, method=None, queue='default', timeout=300, now=False, **kwargs)` | Enqueue `frappe.utils.background_jobs.run_doc_method`, which loads the document and calls `doc.<method>(**kwargs)`. |
| `task(**task_kwargs)` | Decorator. Adds `fn.enqueue(**kwargs)`, which calls `frappe.enqueue(fn, **task_kwargs, **kwargs)`. |
| `concurrent_limit(limit: int \| None = None, wait_timeout: int = 10)` | Decorator. Limits how many calls of the function run at the same time. Defined in `frappe.concurrency_limiter`. |
| `publish_realtime(event=None, message=None, room=None, user=None, doctype=None, docname=None, task_id=None, after_commit=False)` | Send a Socket.IO event. `after_commit=True` sends it after the transaction commits. Defined in `frappe.realtime`. |
| `publish_progress(percent, title=None, doctype=None, docname=None, description=None, task_id=None)` | Send a `progress` event. Desk shows a progress bar. |

## Caching

| Function | Description |
|---|---|
| `request_cache(func: Callable) -> Callable` | Decorator. Cache return values in `frappe.local.request_cache` for the rest of the request. Defined in `frappe.utils.caching`. |
| `site_cache(ttl: int \| None = None, maxsize: int \| None = None) -> Callable` | Decorator. Cache return values in the process across requests, per site. |
| `local_cache(namespace, key, generator, regenerate_if_none=False)` | Cache a value in `frappe.local.cache[namespace][key]`. Defined as `deprecated_local_cache`. |

`redis_cache` and `http_cache` are in `frappe.utils.caching`. See [Jobs, Cache and Realtime](jobs-cache-and-realtime.md).

## Email, printing and templates

See [Email, Files and Printing](email-files-and-printing.md) for all arguments.

| Function | Description |
|---|---|
| `sendmail(recipients=None, sender='', subject='No Subject', message='No Message', ..., retry=1, ..., wrapper=None) -> EmailQueue \| None` | Queue an email through the outgoing Email Account. Defined in `frappe.email`. |
| `are_emails_muted()` | Return truthy if `frappe.flags.mute_emails` or the `mute_emails` config is set. |
| `get_print(doctype=None, name=None, print_format=None, style=None, as_pdf=False, doc=None, output=None, no_letterhead=0, password=None, pdf_options=None, letterhead=None, pdf_generator=None, settings=None)` | Return the print HTML of a document, or PDF bytes with `as_pdf=True`. `pdf_generator` is `wkhtmltopdf`, `chrome`, `PlutoPrint`, `Typst` or `WeasyPrint`. Defined in `frappe.utils.print_utils`. |
| `attach_print(doctype, name, file_name=None, print_format=None, style=None, html=None, doc=None, lang=None, print_letterhead=True, password=None, letterhead=None, settings=None)` | Return `{"fname": ..., "fcontent": ...}` of the print, for `sendmail(attachments=[...])`. |
| `render_template(template, context=None, is_path=None, safe_render=True, *, restrict_globals=None)` | Render a Jinja template string or path. Defined in `frappe.utils.jinja`. |
| `get_template(path)` | Return a Jinja template object for a path such as `sample_app/templates/x.html`. |
| `get_jenv(*, restrict_globals=None)` | Return the Jinja environment with Frappe globals and filters. |
| `get_jloader()` | Return the Jinja loader that reads templates from all apps. |
| `get_email_from_template(name, args)` | Render `templates/emails/<name>.html` and `.txt`. Returns `(html, text)`. |
| `format_value(value, df=None, doc=None, currency=None, translated=False, format=None)` | Format a value for display by its field type. Defined in `frappe.utils.formatters`. |
| `format` | Alias of `format_value`. |

## Strings, JSON and files

| Function | Description |
|---|---|
| `as_json(obj: dict \| list, indent=1, separators=None, ensure_ascii=True) -> str` | Return JSON with sorted keys. Converts dates, decimals and documents. |
| `parse_json(val: str)` | Parse a JSON string into a `_dict`. Return other values unchanged. Defined in `frappe.utils.data`. |
| `cint(s, default: int = 0) -> int` | Convert to `int`. |
| `cstr(s, encoding='utf-8') -> str` | Convert to `str`. `None` becomes `""`. |
| `sbool(x)` | Convert `"true"`, `"false"` and other strings to `bool` when possible. |
| `scrub(txt: str) -> str` | `Sample Order` becomes `sample_order`. Also replaces `-` with `_`. |
| `unscrub(txt: str) -> str` | `sample_order` becomes `Sample Order`. |
| `bold(text) -> str` | Wrap text in `<strong>`. |
| `as_unicode(text, encoding='utf-8') -> str` | Convert bytes to `str`. |
| `safe_encode(param, encoding='utf-8')` | Encode `str` to bytes. |
| `safe_decode(param, encoding='utf-8', fallback_map=None)` | Decode bytes to `str`. |
| `safe_eval(code, eval_globals=None, eval_locals=None)` | Evaluate an expression with restricted globals. |
| `generate_hash(txt: str \| None = None, length: int = 56) -> str` | Return a random hex string from `secrets.token_hex`. `txt` is deprecated and ignored. |
| `mock(type, size=1, locale='en')` | Return fake data from `faker`. |
| `read_file(path, raise_not_found=False, as_base64=False)` | Return file content as text or Base64. Returns `None` for a missing file unless `raise_not_found`. |
| `get_file_json(path)` | Read a file and parse it as JSON. |
| `get_file_items(path, raise_not_found=False, ignore_empty_lines=True)` | Return the lines of a text file. Skips empty lines and lines that start with `#`. |
| `create_folder(path, with_init=False)` | Make a folder. With `with_init=True`, add `__init__.py`. |

`scrub`, `unscrub`, `read_file` and `get_file_json` are defined in `frappe/__init__.py`. They are not on `frappe.utils` in v16.

## Classes and types

| Name | Description |
|---|---|
| `_dict` | `dict` with attribute access. From `frappe.types`. |
| `Local`, `LocalProxy`, `release_local(local)` | Context-local storage. From `frappe.utils.local`. |
| `init_site` | See [Site, connection and user](#site-connection-and-user). |
| `ConfType`, `SessionType`, `LogMessageType`, `JobMetaType`, `ResponseDict`, `FlagsDict`, `FormDict` | Type aliases for the context objects. |

`frappe` also exposes standard library and third-party names it imports (`os`, `sys`, `json`, `re`, `inspect`, `functools`, `importlib`, `threading`, `warnings`, `orjson`, `defaultdict`, `Callable`, `Iterable`, `Any`, `Optional`, `Union`, `TypeAlias`, `TYPE_CHECKING`, `Headers`). Do not use them through `frappe`.

## Submodules

These subpackages are attributes of `frappe` after import. Each has its own page.

| Module | Page |
|---|---|
| `frappe.model` | [Document](document.md) |
| `frappe.database`, `frappe.query_builder` | [Database](database.md), [Query Builder](query-builder.md) |
| `frappe.utils` | [Utilities](utils.md) |
| `frappe.permissions`, `frappe.share`, `frappe.defaults` | [Permissions and Sessions](permissions-and-sessions.md) |
| `frappe.cache_manager`, `frappe.realtime`, `frappe.concurrency_limiter` | [Jobs, Cache and Realtime](jobs-cache-and-realtime.md) |
| `frappe.email` | [Email, Files and Printing](email-files-and-printing.md) |
| `frappe.config` | [Site Config](../configuration/site-config.md) |

`frappe.exceptions` and `frappe.types` are covered on this page. The other subpackages that are attributes after import (`frappe.core`, `frappe.desk`, `frappe.modules`, `frappe.locale`, `frappe.integrations`, `frappe.automation_engine`, `frappe.flow`, `frappe.studio`, `frappe.monitor`, `frappe.commands`, `frappe.coverage`, `frappe.deprecation_dumpster`) hold app modules and CLI code. They have no API on `frappe` itself.

## Deprecated

| Name | Status | Replacement |
|---|---|---|
| `connect(site=...)`, `connect(db_name=...)` | Warns, removal in v17 | Call `frappe.init(site)`, then `frappe.connect()`. |
| `generate_hash(txt=...)` | Warns, `txt` ignored | `frappe.generate_hash(length=...)`. |
| `get_test_records(doctype)` | `@deprecated`, removal in v17 | `cls.globalTestRecords["Some Doc"]` in an `IntegrationTestCase`, else `frappe.tests.utils.load_test_records_for`. See [Testing](testing.md). |
| `local_cache(namespace, key, generator, regenerate_if_none=False)` | Named `deprecated_local_cache`, no warning | `frappe.request_cache`, `frappe.site_cache` or `frappe.client_cache`. |
| `get_list` / `get_all` arguments `limit_start`, `limit_page_length`, `start`, `page_length` | Warn when truthy | `offset` and `limit`. |
| `frappe.user` | Always `None` | `frappe.session.user`. |

## Related

- [Document](document.md)
- [Database](database.md)
- [Query Builder](query-builder.md)
- [Utilities](utils.md)
- [Permissions and Sessions](permissions-and-sessions.md)
- [Jobs, Cache and Realtime](jobs-cache-and-realtime.md)
- [Email, Files and Printing](email-files-and-printing.md)
- [Hooks reference](../configuration/hooks.md)
- [JavaScript frappe namespace](../javascript/frappe.md)
