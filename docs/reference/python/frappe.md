---
title: frappe namespace
description: Every public name on the frappe Python module in version 15 with signature, defaults, return value and side effects, grouped by topic.
version: v15
sources:
  - frappe/__init__.py
  - frappe/exceptions.py
  - frappe/model/document.py
  - frappe/model/delete_doc.py
  - frappe/model/rename_doc.py
  - frappe/model/meta.py
  - frappe/model/db_query.py
  - frappe/client.py
  - frappe/app.py
  - frappe/utils/error.py
  - frappe/utils/caching.py
  - frappe/utils/jinja.py
  - frappe/utils/print_utils.py
  - frappe/utils/formatters.py
  - frappe/utils/background_jobs.py
  - frappe/utils/typing_validations.py
  - frappe/utils/redis_wrapper.py
  - frappe/utils/data.py
  - frappe/realtime.py
  - frappe/query_builder/utils.py
---

# frappe namespace

This page lists every public name on the `frappe` module in version 15 (`frappe.__version__` is `15.x`). Use it to look up an exact signature, default or side effect. The list comes from `dir(frappe)` on a v15 site, with signatures read by `inspect.signature`. For the `Document` class, see [Document](document.md). For `frappe.db`, see [Database](database.md). For `frappe.utils`, see [Utilities](utils.md).

## How names are loaded

- Most functions are defined in `frappe/__init__.py`. Version 15 has no lazy import table.
- Some functions are thin wrappers with the signature `(*args, **kwargs)`. They import the real function on the first call. This page shows the signature of the target function and names the module. Examples: `frappe.enqueue`, `frappe.publish_realtime`, `frappe.format_value`.
- Request-scoped objects (`frappe.db`, `frappe.session`, `frappe.form_dict` ...) are `LocalProxy` objects on `frappe.local`. They are valid only after `frappe.init()` and, for `db`, `frappe.connect()`.
- `frappe` re-exports every class from `frappe/exceptions.py` (`from .exceptions import *`).

## Context objects

| Name | Type | Description |
|---|---|---|
| `frappe.local` | `werkzeug.local.Local` | Thread-local and request-local storage. Every object below is an attribute of it. `release_local(local)` clears it. |
| `frappe.db` | `LocalProxy` to `MariaDBDatabase` or `PostgresDatabase` | Connection of the current site. Set by `frappe.connect()`. See [Database](database.md). |
| `frappe.qb` | `LocalProxy` to `MariaDB` or `Postgres` | Query builder for `conf.db_type`. Set by `frappe.init()`. See [Query Builder](query-builder.md). |
| `frappe.conf` | `LocalProxy` (`_dict`) | `site_config.json` merged over `common_site_config.json`. Set by `frappe.init()`. |
| `frappe.session` | `LocalProxy` (`_dict`) | Current session. `frappe.session.user` is the user name. `frappe.session.sid` is the session id. `frappe.session.data` holds session data. Default user after `init` is `Guest`. |
| `frappe.form_dict` | `LocalProxy` (`_dict`) | Request arguments. Reset by `frappe.set_user()`. |
| `frappe.form` | `LocalProxy` | Alias of `frappe.form_dict`. |
| `frappe.request` | `LocalProxy` to `werkzeug.wrappers.Request` | The HTTP request. Not set in console, jobs or tests. |
| `frappe.response` | `LocalProxy` (`_dict`) | Response body. Starts as `{"docs": []}`. A whitelisted method's return value goes to `frappe.response["message"]`. |
| `frappe.flags` | `LocalProxy` (`_dict`) | Request-scoped flags. See [Global flags](#global-flags). |
| `frappe.lang` | `LocalProxy` (`str`) | Language of the current request. Default `conf.lang` or `"en"`. |
| `frappe.job` | `LocalProxy` | Metadata of the running background job. Not set outside a worker. |
| `frappe.message_log` | `LocalProxy` (`list`) | Messages added by `frappe.msgprint`. Sent as `_server_messages`. |
| `frappe.error_log` | `LocalProxy` (`list`) | Entries added by `frappe.errprint`. Sent as `exc`. |
| `frappe.debug_log` | `LocalProxy` (`list`) | Lines added by `frappe.log`. |
| `frappe.user` | `LocalProxy` | Proxy to `frappe.local.user`. `frappe.init()` sets it to `None` and `set_user` does not change it. Use `frappe.session.user`. |
| `frappe.cache` | `RedisWrapper` | Redis cache client, global to the process, keys prefixed per site. `frappe.cache()` also works and returns the same object, for old code. See [Jobs, Cache and Realtime](jobs-cache-and-realtime.md). |
| `frappe.STANDARD_USERS` | `tuple` | `("Guest", "Administrator")`. |
| `frappe.SITE_NAME_PATTERN` | `re.Pattern` | `^[a-zA-Z0-9._-]+$`. `frappe.init` raises `ValueError` for other site names. |
| `frappe.__version__` | `str` | The release, for example `"15.127.1"`. |
| `frappe.__title__` | `str` | `"Frappe Framework"`. |
| `frappe.controllers` | `dict` | Controller class cache, per site. |
| `frappe.loggers` | `dict` | Logger cache used by `frappe.logger`. |
| `frappe.log_level` | `int \| None` | Log level for `frappe.logger`. Default `None`. |
| `frappe.whitelisted` | `list` | Functions decorated with `@frappe.whitelist`. |
| `frappe.guest_methods` | `list` | Whitelisted functions with `allow_guest=True`. |
| `frappe.xss_safe_methods` | `list` | Guest functions with `xss_safe=True`. |
| `frappe.allowed_http_methods_for_whitelisted_func` | `dict` | Function to list of allowed HTTP methods. |

### Values on frappe.local

`frappe.init()` sets these attributes. Read them as `frappe.local.<name>`.

| Attribute | Value after `init` |
|---|---|
| `site` | Site name. |
| `sites_path`, `site_path` | Path of the `sites` folder and of the site folder. |
| `conf`, `lang`, `flags`, `form_dict`, `session`, `response`, `qb` | See the table above. |
| `cache` | Empty `dict`. Request-scoped cache used by `frappe.local_cache`. Reset by `set_user`. |
| `locked_documents` | Documents locked by `doc.lock()` in this request. |
| `request_ip`, `task_id`, `user`, `user_perms` | `None`. |
| `role_permissions`, `valid_columns`, `new_doc_templates` | Empty caches. |
| `app_modules`, `module_app` | App to module list and module to app maps. Set by `setup_module_map`. |
| `initialised` | `True`. A second `init` call returns early unless `force=True`. |

### Global flags

`frappe.init()` sets these keys on `frappe.flags`. Other code sets more keys during migrate, install and import.

| Flag | Default | Meaning |
|---|---|---|
| `currently_saving` | `[]` | `(doctype, name)` pairs inside a save. `db_set` does not change `modified` for these. |
| `redirect_location` | `""` | Target of `frappe.redirect`. |
| `in_install_db`, `in_import`, `new_site` | `False` | Set during database install, data import and new site creation. |
| `in_test` | `False` | `True` while tests run. |
| `mute_messages` | `False` | `msgprint` does not queue messages. It still raises when asked to. |
| `mute_emails` | `False` | `frappe.are_emails_muted()` returns truthy. |
| `ignore_links` | `False` | Global flag key. Document Link checks use `doc.flags.ignore_links`. |
| `has_dataurl` | `False` | Set when a Text Editor value had a data URL. |
| `read_only` | `False` | Request runs in read-only mode. `log_error` and `add_viewed` use deferred inserts. `frappe.db.begin` starts a read-only transaction. |

Keys set elsewhere and read by `Document`: `in_migrate`, `in_install`, `in_patch`, `in_setup_wizard`, `print_messages` (print `msgprint` output to the console), `error_message` (message for the next `PermissionError`).

## Site, connection and user

| Function | Description |
|---|---|
| `init(site: str, sites_path: str = '.', new_site: bool = False, force=False) -> None` | Set up `frappe.local` for a site: config, flags, session as `Guest`, query builder, Redis connection, module map. Does nothing if already initialised, unless `force=True`. |
| `connect(site: str \| None = None, db_name: str \| None = None, set_admin_as_user: bool = True) -> None` | Open the database connection in `frappe.local.db`. With `site`, call `init(site)` first. The database user is `db_name` from the site config. Sets the user to `Administrator` by default. |
| `connect_replica() -> bool` | Open a connection to `replica_host` (port `replica_db_port`) and swap it into `frappe.db`. The primary stays in `frappe.local.primary_db`. Uses `replica_db_name` and `replica_db_password` when `different_credentials_for_replica` is set. Returns `False` if already swapped. |
| `destroy()` | Close the database connection and release `frappe.local`. |
| `init_site(site=None)` | Context manager class. Calls `init(site)` on enter and `destroy()` on exit. `site=None` loads only `common_site_config.json`. |
| `setup_redis_cache_connection()` | Set `frappe.cache` once per process. Called by `init`. |
| `setup_module_map(include_all_apps: bool = True) -> None` | Build `local.app_modules` and `local.module_app`. Cached in Redis as `app_modules` or `installed_app_modules`. |
| `set_user(username: str)` | Set `frappe.session.user` and `sid`. Clears `local.cache`, `form_dict`, session data, Jinja environments, role permission and user permission caches. No permission check. |
| `get_user()` | Return the `UserPermissions` object of the session user. Cached in `local.user_perms`. |
| `get_roles(username=None) -> list[str]` | Return the roles of a user. Default is the session user. Returns `["Guest"]` when no session exists. |
| `get_request_header(key, default=None)` | Return an HTTP request header. Fails outside a request. |
| `get_site_config(sites_path: str \| None = None, site_path: str \| None = None) -> dict[str, Any]` | Return `site_config.json` merged over `common_site_config.json`. See below. |
| `get_common_site_config(sites_path: str \| None = None) -> dict[str, Any]` | Return `common_site_config.json` as a `_dict`, or an empty `_dict`. |
| `get_conf(site: str \| None = None) -> dict[str, Any]` | Return `frappe.local.conf` if set. Else load the config of `site`, or the common config. |
| `get_system_settings(key: str)` | Return one System Settings value. Caches the document in `frappe.local.system_settings`. |
| `get_website_settings(key)` | Return one Website Settings value. Caches the document in `frappe.local.website_settings`. |
| `get_site_path(*joins)` | Return `frappe.local.site_path` joined with `joins`. |
| `is_setup_complete()` | Return `True` if `is_setup_complete` is set on the `Installed Application` rows of `frappe` and `erpnext`. Cached per request. |
| `ping()` | Return `"pong"`. Whitelisted: `GET`, `POST`, `PUT`, `DELETE`, guest allowed. |

`get_site_config` also sets these keys. An environment variable wins over the config value:

| Key | Environment variable | Default |
|---|---|---|
| `redis_queue` | `FRAPPE_REDIS_QUEUE` | `redis://127.0.0.1:11311` |
| `redis_cache` | `FRAPPE_REDIS_CACHE` | `redis://127.0.0.1:13311` |
| `db_type` | `FRAPPE_DB_TYPE` | `mariadb` |
| `db_socket` | `FRAPPE_DB_SOCKET` | none |
| `db_host` | `FRAPPE_DB_HOST` | `127.0.0.1` |
| `db_port` | `FRAPPE_DB_PORT` | `3306` (MariaDB) or `5432` (Postgres) |

With `extra_config` (a dotted path or a list of them) in the config, `get_site_config` calls each function and merges the returned dict. See [Site Config](../configuration/site-config.md).

## Documents

See [Document](document.md) for the methods of the returned objects and [Controllers](../../developer-guide/data-model/controllers.md) for the lifecycle.

| Function | Description |
|---|---|
| `get_doc(doctype, name, /, *, for_update=None)` | Load a document and its child rows from the database. `name` can be a filters dict. Raises `DoesNotExistError`. No permission check. |
| `get_doc(doctype, /)` | Load a Single DocType. |
| `get_doc(dict)` or `get_doc(**kwargs)` | Make a new unsaved document. The dict must have `doctype`. Prefer `new_doc`. |
| `get_doc(document)` | Return the same `Document` object. |
| `new_doc(doctype: str, *, parent_doc=None, parentfield=None, as_dict=False, **kwargs) -> Document` | Return a new unsaved document with defaults and user defaults set. `kwargs` set field values. |
| `get_lazy_doc` | Alias of `get_doc`. Child tables load at once. |
| `get_cached_doc(*args, **kwargs) -> Document` | Same as `get_doc`, but read from Redis if present. Stores the document in Redis with no expiry. `clear_document_cache` and `doc.save()` remove it. Do not change and save the returned object. |
| `get_last_doc(doctype, filters=None, order_by='creation desc', *, for_update=False)` | Return the newest document that matches `filters`. Raises `DoesNotExistError` if none. No permission check. |
| `get_single(doctype)` | Return `get_doc(doctype, doctype)`. |
| `copy_doc(doc: Document, ignore_no_copy: bool = True) -> Document` | Return an unsaved deep copy. Clears `name`, `owner`, `creation`, `modified`, `modified_by`, `amended_from`, `amendment_date`, and `docstatus` (not in tests). With `ignore_no_copy=False`, also clears `no_copy` fields. |
| `delete_doc(doctype: str \| None = None, name: str \| None = None, force: bool = False, ignore_doctypes: list[str] \| None = None, for_reload: bool = False, ignore_permissions: bool = False, flags: None = None, ignore_on_trash: bool = False, ignore_missing: bool = True, delete_permanently: bool = False)` | Delete one document or a list of names. See [delete_doc](#delete_doc). |
| `delete_doc_if_exists(doctype, name, force=0)` | Call `delete_doc(doctype, name, force=force, ignore_missing=True)`. Returns `None`. |
| `rename_doc(doctype: str, old: str, new: str, force: bool = False, merge: bool = False, *, ignore_if_exists: bool = False, show_alert: bool = True, rebuild_search: bool = True) -> str` | Rename a document and update all Link and Dynamic Link values. Checks `write` permission and `allow_rename` (unless `force`). Whitelisted: `POST`, `PUT`. See [rename_doc](document.md#frappemodelrename_doc). |
| `get_version(doctype, name, limit=None, head=False, raise_err=True)` | Meant to return the `Version` history of a document. It fails with `OperationalError` in v15. See [Pitfalls](#pitfalls). |
| `get_meta(doctype, cached=True)` | Return the `frappe.model.meta.Meta` of a DocType with Custom Fields and Property Setters applied. |
| `get_meta_module(doctype)` | Return the Python module of the DocType controller. |
| `is_table(doctype: str) -> bool` | Return `True` for a child table DocType (`istable`). The list of child DocTypes is cached in Redis as `is_table`. |
| `get_doctype_app(doctype)` | Return the app that owns the DocType module. Cached in `frappe.local.cache`. |
| `get_precision(doctype: str, fieldname: str, currency: str \| None = None, doc: Document \| None = None) -> int` | Return the precision of a numeric field. |
| `can_cache_doc(args) -> str \| None` | Return the cache key for `get_cached_doc` arguments, or `None` if they cannot be cached. |
| `get_document_cache_key(doctype: str, name: str)` | Return `document_cache::<doctype>::<name>`. |
| `clear_document_cache(doctype: str, name: str \| None = None) -> None` | Remove a document (or all documents of the DocType) from Redis. Runs again after commit and rollback. Also clears `local.system_settings` or `local.website_settings` for those DocTypes. |
| `reload_doc(module: str, dt: str \| None = None, dn: str \| None = None, force: bool = False, reset_permissions: bool = False)` | Import a DocType or record from its JSON file in `<module>/<dt>/<dn>/<dn>.json`. Use it in patches. `force` reloads even when `modified` matches. |
| `reload_doctype(doctype, force=False, reset_permissions=False)` | Reload a DocType from its JSON file. Reads the module from the database. |
| `import_doc(path)` | Import documents from a JSON file or folder through `frappe.core.doctype.data_import.data_import.import_doc`. |
| `make_property_setter(args, ignore_validate=False, validate_fields_for_doctype=True, is_system_generated=True, *, module=None)` | Insert a Property Setter. `args` keys: `doctype`, `fieldname`, `property`, `value`, `property_type`, `doctype_or_field`, `row_name`. Without `doctype`, makes one for every DocType that has `fieldname`. |

```python
doc = frappe.get_doc("Sample Order", "SO-00001")
doc.check_permission("write")
item = frappe.new_doc("Sample Item", item_name="Cable", category="Hardware", rate=12)
```

!!! warning
    `frappe.get_doc` has no `check_permission` argument in v15. `frappe.get_doc("Sample Item", "Laptop", check_permission=True)` loads the document and checks nothing. Call `doc.check_permission(...)` or `frappe.has_permission(...)`.

### delete_doc

`frappe.delete_doc` calls `frappe.model.delete_doc.delete_doc`. For each name it does this:

1. Return `False` if the record does not exist and `ignore_missing=True`. Raise `DoesNotExistError` if `ignore_missing=False`.
2. Delete stored passwords of the document.
3. Lock the row with `SELECT ... FOR UPDATE NOWAIT`. Throw `QueryTimeoutError` if another transaction holds it.
4. Check `delete` permission and throw if the document is submitted. Skipped with `for_reload=True`.
5. Run `on_trash` and `on_change` (with `doc.flags.in_delete`). Skipped with `ignore_on_trash=True` or `for_reload=True`.
6. Check that no document links to it. Skipped with `force=True`. Raises `LinkExistsError`. If the DocType has an `enabled` or `disabled` field, the message says to disable the record.
7. Revert the naming series if this was the last number, delete the row and child rows, run `after_delete`, delete attachments.
8. Enqueue `delete_dynamic_links` after commit (run at once in tests).
9. Clear the document cache, global search entry and tags.
10. Add a `Deleted Document` unless `delete_permanently=True` or `for_reload=True`.
11. Publish `doc_update` and `list_update` and insert a feed entry. Skipped with `for_reload=True` or in a patch.

| Parameter | Default | Description |
|---|---|---|
| `doctype` | `None` | DocType. If empty, read `dt` and `dn` from `frappe.form_dict`. |
| `name` | `None` | Name, or a list of names. |
| `force` | `False` | Skip the link check. Also allows deleting a standard DocType. |
| `ignore_doctypes` | `None` | Child DocTypes whose rows are not deleted. |
| `for_reload` | `False` | Used when a DocType is reloaded. Runs `before_reload` on a DocType, skips permission and `on_trash`, deletes permanently. |
| `ignore_permissions` | `False` | Skip the `delete` permission check. |
| `flags` | `None` | Dict copied to `doc.flags`. `doc.delete()` passes its own flags. |
| `ignore_on_trash` | `False` | Skip `on_trash` and `on_change`. |
| `ignore_missing` | `True` | Return `False` instead of raising for a missing record. |
| `delete_permanently` | `False` | Do not keep a `Deleted Document` copy. |

A virtual DocType must implement its own `delete()`. `delete_doc` calls it after the permission check.

!!! warning
    With a list of names, `delete_doc` returns `False` at the first missing name. Names after it in the list are not deleted.

## Queries

`frappe.get_list` and `frappe.get_all` call `frappe.model.db_query.DatabaseQuery(doctype).execute()`. For filter syntax, see [Database](../../developer-guide/database/database.md). For `frappe.db.get_value` and other database methods, see [Database](database.md).

| Function | Description |
|---|---|
| `get_list(doctype, *args, **kwargs)` | Return rows as a list of `_dict`. Applies role permissions, user permissions and sharing of `user` (default session user). No default row limit, although the docstring says 20. |
| `get_all(doctype, *args, **kwargs)` | Same as `get_list` with `ignore_permissions=True`. Sets `limit_page_length=0` if not given. |
| `get_value(*args, **kwargs)` | Alias of `frappe.db.get_value`. No permission check. |
| `get_cached_value(doctype: str, name: str, fieldname: str = 'name', as_dict: bool = False) -> Any` | Return field values from `get_cached_doc`. `fieldname` can also be a list. Returns `None` if the document does not exist. `as_dict=True` needs a list of fieldnames. |
| `get_single_value(setting: str, fieldname: str, /, *, as_dict: bool = False)` | Return a cached value of a Single DocType through `get_cached_value`. |
| `set_value(doctype, docname, fieldname, value=None)` | Call `frappe.client.set_value`: load the document, update it, run `save()` with all validations and permission checks. `fieldname` can be a dict. |
| `get_query(*args, **kwargs) -> QueryBuilder` | Call `frappe.database.query.Engine().get_query`. Defined in `frappe.query_builder.utils`. See [Query Builder](query-builder.md). |
| `get_query_builder(type_of_db: str)` | Return the query builder class for `mariadb` or `postgres`. |
| `build_match_conditions(doctype, as_condition=True)` | Return the user permission conditions of a DocType as SQL, or as a list. |

### get_list arguments

| Argument | Type | Default | Description |
|---|---|---|---|
| `fields` | `list \| tuple \| str` | `` [`tab<doctype>`.`<pluck or name>`] `` | Columns. SQL functions as strings, for example `"count(name) as count"`. Child fields as `"items.qty"`, link fields as `"item.category"`. |
| `filters` | `dict \| list` | `None` | AND conditions. |
| `or_filters` | `dict \| list` | `None` | OR conditions. |
| `group_by` | `str` | `None` | Group by fields. |
| `order_by` | `str` | DocType `sort_field` and `sort_order` | Order by fields. |
| `limit`, `limit_page_length`, `page_length` | `int` | `None` | Maximum rows. `None` or `0` returns all rows. |
| `limit_start`, `start` | `int` | `0` | Rows to skip. |
| `as_list` | `bool` | `False` | Return tuples, not dicts. |
| `pluck` | `str` | `None` | Return a flat list of one field. |
| `distinct` | `bool` | `False` | `SELECT DISTINCT`. |
| `ignore_permissions` | `bool` | `False` | Skip permission checks. `get_all` sets it. |
| `user` | `str` | session user | Check permissions of this user. |
| `parent_doctype` | `str` | `None` | Parent DocType, needed to query a child table with permissions. Keyword only. |
| `reference_doctype` | `str` | `None` | DocType whose link field context applies to user permissions. |
| `with_comment_count` | `bool` | `False` | Add `_comment_count` to each row. |
| `with_childnames`, `join` | `bool`, `str` | `False`, `"left join"` | Join child tables of the DocType with the given join type. |
| `update` | `dict` | `None` | Merge this dict into each row. |
| `run` | `bool` | `True` | `False` returns the SQL string, not rows. |
| `debug` | `bool` | `False` | Print the query. |
| `strict` | `bool` | `True` | Check the final query for subqueries, set operations, comments and stacked statements. `False` skips the check. |
| `save_user_settings`, `save_user_settings_fields`, `user_settings` | | `False`, `False`, `None` | Save list settings of the user. Used by the list view. |
| `ignore_ddl` | `bool` | `False` | Return `[]` if the table does not exist. |
| `docstatus`, `add_total_row`, `ignore_ifnull` | | | Accepted. `docstatus` and `add_total_row` have no effect. |

`offset`, `ignore_user_permissions` and dict fields such as `{"COUNT": "*"}` do not exist in v15. The first two arguments are swapped when `fields` looks like filters (a dict, or a list of lists).

```python
frappe.get_list("Sample Order", filters={"status": "Draft"}, fields=["name", "total"], order_by="creation desc", limit=20)
frappe.get_all("Sample Item", filters={"disabled": 0}, pluck="name")
frappe.get_list("Sample Order", fields=["status", "count(name) as count"], group_by="status")
```

## Messages and errors

| Function | Description |
|---|---|
| `msgprint(msg: str, title: str \| None = None, raise_exception: bool \| type[Exception] \| Exception = False, as_table: bool = False, as_list: bool = False, indicator: Literal['blue', 'green', 'orange', 'red', 'yellow'] \| None = None, alert: bool = False, primary_action: str \| None = None, is_minimizable: bool = False, wide: bool = False, *, realtime=False) -> None` | Queue a message dialog for the user in `frappe.message_log`. See the parameter table below. |
| `throw(msg: str, exc: type[Exception] \| Exception = ValidationError, title: str \| None = None, is_minimizable: bool = False, wide: bool = False, as_list: bool = False, primary_action=None) -> None` | Call `msgprint` with `raise_exception=exc` and a red indicator. Always raises. |
| `clear_messages()` | Empty `frappe.message_log`. |
| `clear_last_message()` | Remove the last message from `frappe.message_log`. |
| `get_message_log() -> list[dict]` | Return a copy of `frappe.message_log`. |
| `log_error(title=None, message=None, reference_doctype=None, reference_name=None, *, defer_insert=False)` | Insert an `Error Log` and return it. See below. Defined in `frappe.utils.error`. |
| `get_traceback(with_context: bool = False) -> str` | Return the traceback of the current exception. Calls `frappe.utils.get_traceback`. |
| `errprint(msg: str) -> None` | Add `{"exc": msg}` to `frappe.error_log`. Prints it outside a request, when `cmd` is not in `form_dict`, or in developer mode. |
| `log(msg: str) -> None` | Add to `frappe.debug_log`. Prints `repr(msg)` outside a request. |
| `logger(module=None, with_more_info=False, allow_site=True, filter=None, max_size=100000, file_count=20)` | Return a cached Python logger that writes to `logs/<module>.log` in the bench and, with `allow_site`, to `<site>/logs/<module>.log`. |
| `print_sql(enable: bool = True) -> None` | Print every SQL query. Works only with `allow_tests` in site config. Stored in Redis as `flag_print_sql`. |
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

With `frappe.flags.mute_messages` set, `msgprint` queues nothing but still raises. Version 15 does not sanitize the HTML of the message.

```python
frappe.throw(_("Rate must be positive"), title=_("Invalid Rate"))
frappe.msgprint(_("Order updated"), indicator="green", alert=True)
frappe.throw(_("Not allowed"), frappe.PermissionError)
```

### log_error

- `title` is the `method` field. `message` is the `error` field. Without `message`, Frappe logs the current traceback with context.
- If `message` is set and `title` contains a newline, the two are swapped.
- Inserts with `ignore_permissions=True`. Uses `deferred_insert` (and returns `None`) when `defer_insert=True` or in read-only mode.
- Returns `None` if no database connection exists.
- An error while inserting the log is not caught.

!!! warning
    `frappe.log_error(message="...")` without `title` raises `TypeError` in v15. Always pass `title`.

### Exceptions

All classes are in `frappe/exceptions.py` and on `frappe`. The HTTP status comes from `http_status_code`. A class without one inherits it from its base, or gives `500`. `frappe.app.handle_exception` reads it.

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
| `SessionStopped` | `Exception` | 503 |
| `InReadOnlyMode` | `ValidationError` | 503 (not written to Error Log) |
| `QueueOverloaded` | `ValidationError` | 503 |
| `RequestToken` | `Exception` | 200 |
| `Redirect(http_status_code: int = 301)` | `Exception` | 301 or the given code |

These inherit 417 from `ValidationError`: `DataError`, `MappingMismatchError`, `InvalidStatusError`, `MandatoryError`, `NonNegativeError`, `InvalidSignatureError`, `CannotChangeConstantError`, `CharacterLengthExceededError`, `UpdateAfterSubmitError`, `LinkValidationError`, `CancelledLinkError` (base `LinkValidationError`), `DocstatusTransitionError`, `TimestampMismatchError`, `EmptyTableError`, `LinkExistsError`, `InvalidEmailAddressError`, `InvalidNameError`, `InvalidPhoneNumberError`, `TemplateNotFoundError`, `UniqueValidationError`, `AppNotInstalledError`, `ImplicitCommitError`, `DocumentLockedError`, `CircularLinkingError`, `InvalidColumnName`, `IncompatibleApp`, `InvalidDates`, `DataTooLongException`, `DocumentAlreadyRestored`, `AttachmentLimitReached`, `PrintFormatError`, `InvalidDatabaseFile`.

These have no status code and give 500: `QueryTimeoutError`, `QueryDeadlockError`, `SiteNotSpecifiedError`, `UrlSchemeNotSupported`, `ImproperDBConfigurationError(reason, msg=None)`, `UnknownDomainError`, `RetryBackgroundJobError`, `SecurityException` (not written to Error Log), `FileAlreadyAttachedException`, `TooManyWritesError`, `InvalidRemoteException`, `CommandFailedError(message, out, err)`, `ExecutableNotFound` and `InvalidRoundingMethod` (base `FileNotFoundError`). A raw driver deadlock or lock wait timeout that reaches `handle_exception` as `InternalError` gives 508. `IncorrectSitePath` subclasses `werkzeug.exceptions.NotFound`. `frappe.NotFound` is the werkzeug class.

## Permissions

See [Permissions and Sessions](permissions-and-sessions.md) for `frappe.permissions` and [Permissions](../../developer-guide/security/permissions.md) for the concepts.

| Function | Description |
|---|---|
| `has_permission(doctype=None, ptype='read', doc=None, user=None, throw=False, *, parent_doctype=None, debug=False, ignore_share_permissions=False)` | Return `True` if `user` (default session user) has `ptype` on the DocType, or on `doc` (object or name). With `throw=True`, raise `PermissionError` and set `frappe.flags.error_message`. |
| `has_website_permission(doc=None, ptype='read', user=None, verbose=False, doctype=None)` | Check portal access. Calls the controller's `has_website_permission`, else every `has_website_permission` hook for the DocType. Returns `False` if neither exists. Returns `True` when `doc.flags.ignore_permissions` is set. |
| `only_for(roles: list[str] \| tuple[str] \| str, message=False)` | Raise `PermissionError` if the session user has none of `roles`. `Administrator` and tests always pass. With `message=True`, show a "This action is only allowed for" dialog. |
| `only_has_select_perm(doctype, user=None, ignore_permissions=False)` | Return truthy if the user has `select` but not `read` on the DocType. |

```python
@frappe.whitelist(methods=["POST"])
def close_order(name: str):
	doc = frappe.get_doc("Sample Order", name)
	doc.check_permission("write")
	doc.db_set("status", "Closed")
```

## Whitelisting and calls

See [Whitelisted Methods](../../developer-guide/api/whitelisted-methods.md) and [REST Endpoints](../http/rest-endpoints.md).

| Function | Description |
|---|---|
| `whitelist(allow_guest=False, xss_safe=False, methods=None)` | Decorator. Makes a function callable at `/api/method/<dotted.path>`. See below. |
| `is_whitelisted(method)` | Raise `PermissionError` if `method` is not whitelisted, or is not guest-allowed for `Guest`. For `Guest` and non-`xss_safe` methods, sanitize string values in `form_dict`. |
| `override_whitelisted_method(original_method: str) -> str` | Return the last path from the `override_whitelisted_methods` hook, or the original path. |
| `validate_and_sanitize_search_inputs(fn)` | Decorator for custom Link search query functions. Checks `searchfield`, casts `start` and `page_len` to `int`, and returns `[]` if `doctype` does not exist. |
| `read_only()` | Decorator. With `read_from_replica` in site config, run the function on the replica connection. |
| `write_only()` | Decorator. Switch to the primary connection while the function runs, if a replica is active. |
| `call(fn: str \| Callable, *args, **kwargs)` | Call a function, or a dotted path, with only the keyword arguments it accepts. |
| `get_newargs(fn: Callable, kwargs: dict[str, Any]) -> dict[str, Any]` | Return `kwargs` without keys `fn` does not accept. Always removes `ignore_permissions` and `flags`. |
| `get_attr(method_string: str) -> Any` | Return the object at a dotted path. Throws `AppNotInstalledError` if the first part is not an installed app (except during install and uninstall). |

`@frappe.whitelist` details:

- `methods`: `None` allows `GET`, `POST`, `PUT` and `DELETE`. A list limits the methods.
- `allow_guest=True` lets `Guest` call it. Guest arguments are HTML-sanitized unless `xss_safe=True`.
- In a request or in tests, arguments are converted and checked against the type annotations (`frappe.utils.typing_validations.validate_argument_types`). Annotations are optional.

```python
# sample_app/api.py
import frappe

@frappe.whitelist(methods=["POST"])
def set_rate(item: str, rate: float) -> float:
	doc = frappe.get_doc("Sample Item", item)
	doc.check_permission("write")
	doc.rate = rate
	doc.save()
	return doc.rate
```

## Hooks, apps and modules

See [Hooks](../../developer-guide/building-apps/hooks.md), [Hooks reference](../configuration/hooks.md) and [Apps](../../developer-guide/building-apps/apps.md).

| Function | Description |
|---|---|
| `get_hooks(hook: str \| None = None, default: Any \| None = '_KEEP_DEFAULT_LIST', app_name: str \| None = None) -> _dict` | Return all hooks merged from installed apps, or one hook. A missing hook gives `[]` unless `default` is passed. Cached in Redis as `app_hooks`. Not cached in developer mode. With `app_name`, reads only that app. |
| `get_doc_hooks()` | Return `doc_events` keyed by DocType, with tuple keys expanded. Cached in `frappe.local.doc_events_hooks`. |
| `append_hook(target, key, value)` | Merge one hook value into a hooks dict. Lists are extended. Dicts are merged by key. |
| `get_installed_apps(*, _ensure_on_bench=False) -> list[str]` | Return apps installed on the site, from the `installed_apps` global. Cached per request. Returns `[]` during database install. |
| `get_all_apps(with_internal_apps=True, sites_path=None)` | Return apps listed in `sites/apps.txt` (and the site `apps.txt`), with `frappe` first. |
| `get_app_path(app_name, *joins)` | Return the path of the app Python package. |
| `get_app_source_path(app_name, *joins)` | Return the path of the app repository (`<app package>/..`). |
| `get_module_path(module, *joins)` | Return the path of a module folder. |
| `get_pymodule_path(modulename, *joins)` | Return the folder of a Python module. |
| `get_module_list(app_name)` | Return the module names in the app `modules.txt`. |
| `get_module(modulename)` | Import a module with `importlib.import_module`. |
| `get_domain_data(module)` | Return the `data` dict of a `domains` hook entry. |
| `get_active_domains()` | Return active domains from Domain Settings. |
| `clear_cache(user: str \| None = None, doctype: str \| None = None)` | Clear the cache of one user, one DocType, or the whole site. A full clear keeps keys from the `persistent_cache_keys` hook and runs the `clear_cache` hooks. |
| `reset_metadata_version()` | Make a new client build hash so open desks reload metadata. Returns it. |
| `get_desk_link(doctype, name, show_title_with_name=False, open_in_new_tab=False)` | Return an HTML `<a>` to `/app/Form/<doctype>/<name>` with the DocType label and the title. |

## Translation

See [Translations](../../developer-guide/building-apps/translations.md).

| Function | Description |
|---|---|
| `_(msg: str, lang: str \| None = None, context: str \| None = None) -> str` | Return the translation in `lang` (default `frappe.local.lang`). Returns `msg` if none exists. |
| `_lt(msg: str, lang: str \| None = None, context: str \| None = None)` | Return a `_LazyTranslate` object that translates when it is turned into a string. Use it at module level. |
| `N_(msg: str, context: str \| None = None) -> str` | Mark a string for extraction. Return it unchanged. |
| `set_user_lang(user: str, user_language: str \| None = None) -> None` | Set `frappe.local.lang` to the language of `user`, else to `user_language`. |

## Background jobs and realtime

See [Jobs, Cache and Realtime](jobs-cache-and-realtime.md) for all arguments and [Background Jobs](../../developer-guide/runtime/background-jobs.md) for the how-to.

| Function | Description |
|---|---|
| `enqueue(method: str \| Callable, queue: str = 'default', timeout: int \| None = None, event=None, is_async: bool = True, job_name: str \| None = None, now: bool = False, enqueue_after_commit: bool = False, *, on_success: Callable \| None = None, on_failure: Callable \| None = None, at_front: bool = False, job_id: str \| None = None, deduplicate=False, **kwargs) -> Job \| Any` | Put `method` on an RQ queue with `kwargs`. `now=True` runs it in process and returns the result. Defined in `frappe.utils.background_jobs`. |
| `enqueue_doc(doctype, name=None, method=None, queue='default', timeout=300, now=False, **kwargs)` | Enqueue `frappe.utils.background_jobs.run_doc_method`, which loads the document and calls `doc.<method>(**kwargs)`. |
| `task(**task_kwargs)` | Decorator. Adds `fn.enqueue(**kwargs)`, which calls `frappe.enqueue(fn, **task_kwargs, **kwargs)`. |
| `publish_realtime(event: str \| None = None, message: dict \| None = None, room: str \| None = None, user: str \| None = None, doctype: str \| None = None, docname: str \| None = None, task_id: str \| None = None, after_commit: bool = False)` | Send a Socket.IO event. Default event is `task_progress` inside a task, else `global`. `after_commit=True` sends it after the transaction commits. Defined in `frappe.realtime`. |
| `publish_progress(percent, title=None, doctype=None, docname=None, description=None, task_id=None)` | Send a `progress` event. Desk shows a progress bar. |

## Caching

| Function | Description |
|---|---|
| `request_cache(func: Callable) -> Callable` | Decorator. Cache return values in `frappe.local.request_cache` for the rest of the request. Defined in `frappe.utils.caching`. |
| `local_cache(namespace, key, generator, regenerate_if_none=False)` | Cache a value in `frappe.local.cache[namespace][key]` for the request. Calls `generator()` on a miss. No warning. |

`site_cache` and `redis_cache` are not on `frappe` in v15. Import them from `frappe.utils.caching`. See [Jobs, Cache and Realtime](jobs-cache-and-realtime.md).

## Email, printing and templates

See [Email, Files and Printing](email-files-and-printing.md) for all arguments.

| Function | Description |
|---|---|
| `sendmail(recipients=None, sender='', subject='No Subject', message='No Message', as_markdown=False, delayed=True, ..., redact_message_after_send=False) -> EmailQueue \| None` | Queue an email through the outgoing Email Account. With `now=True`, send at once. |
| `are_emails_muted()` | Return truthy if `frappe.flags.mute_emails` or the `mute_emails` config is set. |
| `get_print(doctype=None, name=None, print_format=None, style=None, as_pdf=False, doc=None, output=None, no_letterhead=0, password=None, pdf_options=None, letterhead=None, pdf_generator: Literal['wkhtmltopdf', 'chrome'] \| None = None)` | Return the print HTML of a document, or PDF bytes with `as_pdf=True`. Defined in `frappe.utils.print_utils`. |
| `attach_print(doctype, name, file_name=None, print_format=None, style=None, html=None, doc=None, lang=None, print_letterhead=True, password=None, letterhead=None)` | Return `{"fname": ..., "fcontent": ...}` of the print, for `sendmail(attachments=[...])`. A PDF when Print Settings `send_print_as_pdf` is set, else HTML. |
| `render_template(template, context=None, is_path=None, safe_render=True, *, restrict_globals=None)` | Render a Jinja template string or path. Defined in `frappe.utils.jinja`. |
| `get_template(path)` | Return a Jinja template object for a path such as `sample_app/templates/x.html`. |
| `get_jenv(*, restrict_globals=None)` | Return the Jinja environment with Frappe globals and filters. |
| `get_jloader()` | Return the Jinja loader that reads templates from all apps. |
| `get_email_from_template(name, args)` | Render `templates/emails/<name>.html` and `.txt`. Returns `(html, text)`. |
| `format_value(value, df=None, doc=None, currency=None, translated=False, format=None)` | Format a value for display by its field type. Defined in `frappe.utils.formatters`. |
| `format` | Same as `format_value`. |

## Strings, JSON and files

`cint`, `cstr` and `sbool` come from `frappe.utils.data`. The others are defined in `frappe/__init__.py`. See [Utilities](utils.md).

| Function | Description |
|---|---|
| `as_json(obj: dict \| list, indent=1, separators=None, ensure_ascii=True) -> str` | Return JSON with sorted keys. Converts dates, decimals and documents. |
| `parse_json(val)` | Parse a JSON string into a `_dict`. Return other values unchanged. |
| `cint(s: int \| float \| str, default: int = 0) -> int` | Convert to `int`. |
| `cstr(s, encoding='utf-8')` | Convert to `str`. `None` becomes `""`. |
| `sbool(x: str) -> bool \| Any` | Convert `"true"`, `"false"` and other strings to `bool` when possible. |
| `scrub(txt: str) -> str` | `Sample Order` becomes `sample_order`. |
| `unscrub(txt: str) -> str` | `sample_order` becomes `Sample Order`. |
| `bold(text)` | Wrap text in `<strong>`. |
| `as_unicode(text, encoding: str = 'utf-8') -> str` | Convert bytes to `str`. `None` becomes `""`. |
| `safe_encode(param, encoding='utf-8')` | Encode `str` to bytes. |
| `safe_decode(param, encoding='utf-8', fallback_map: dict \| None = None)` | Decode bytes to `str`. |
| `safe_eval(code, eval_globals=None, eval_locals=None)` | Evaluate an expression with restricted globals. |
| `generate_hash(txt: str \| None = None, length: int = 56) -> str` | Return a random hex string from `secrets.token_hex`. `txt` is not used. |
| `mock(type, size=1, locale='en')` | Return fake data from `faker`. Faker is a bench dev dependency only. Without it, the call raises `ModuleNotFoundError`. |
| `read_file(path, raise_not_found=False, as_base64=False)` | Return file content as text or Base64, or `None`. |
| `get_file_json(path)` | Read a file and parse it as JSON. |
| `get_file_items(path, raise_not_found=False, ignore_empty_lines=True)` | Return the lines of a text file. Skips empty lines and lines that start with `#`. |
| `create_folder(path, with_init=False)` | Make a folder. With `with_init=True`, add `__init__.py`. |

## Classes and types

| Name | Description |
|---|---|
| `_dict` | `dict` with attribute access. From `frappe.types.frappedict`. |
| `Local`, `release_local(local)` | Context-local storage. From `werkzeug.local`. |
| `init_site` | See [Site, connection and user](#site-connection-and-user). |
| `lazy_import(name, package=None)` | Import a module on first attribute access. From `frappe.utils.lazy_loader`. |
| `patch_query_execute()`, `patch_query_aggregation()` | Add `run()`, `walk()`, `max`, `min`, `avg` and `sum` to the query builder. `init` calls them once per process. |

`frappe` also exposes standard library names it imports (`os`, `json`, `re`, `inspect`, `functools`, `importlib`, `traceback`, `warnings`, `gc`, `signal`, `faulthandler`, `unicodedata`, `click`, `Callable`, `Any`, `Literal`, `Optional`, `TypeAlias`, `TYPE_CHECKING`, `overload`) and `g0`, `g1`, `g2` (the garbage collector thresholds read at import). Do not use them through `frappe`.

## Submodules

These subpackages are attributes of `frappe` after import. Each has its own page.

| Module | Page |
|---|---|
| `frappe.model` | [Document](document.md) |
| `frappe.database`, `frappe.query_builder` | [Database](database.md), [Query Builder](query-builder.md) |
| `frappe.utils` | [Utilities](utils.md) |
| `frappe.defaults` | [Permissions and Sessions](permissions-and-sessions.md) |
| `frappe.cache_manager` | [Jobs, Cache and Realtime](jobs-cache-and-realtime.md) |

`frappe.exceptions` and `frappe.types` are covered on this page. The other subpackages that are attributes after import (`frappe.core`, `frappe.desk`, `frappe.monitor`, `frappe.commands`, `frappe.coverage`) hold app modules and CLI code. They have no API on `frappe` itself.

## Deprecated

`frappe/utils/deprecations.py` has the `deprecated` decorator and `deprecation_warning()`.

| Name | Status | Replacement |
|---|---|---|
| `enqueue(..., job_name=...)` | Warns | `job_id`. |
| `enqueue(..., is_async=False)` outside tests | Warns | `now=True`. |
| `get_list` / `get_all` arguments `limit_start`, `limit_page_length`, `start`, `page_length` | Accepted, no warning | `limit`. |
| `frappe.user` | Always `None` | `frappe.session.user`. |
| `frappe.cache()` | Accepted, no warning | `frappe.cache`. |

## Pitfalls

- `frappe.get_doc` ignores a `check_permission` argument. Check permissions yourself.
- `frappe.get_list` and `frappe.get_all` return all rows when you do not pass `limit`.
- `frappe.get_cached_doc` keeps the document in Redis with no expiry. Raw SQL or query builder updates do not clear it.
- `frappe.get_version` builds its `Version` query with `order_by` and `limit` inside `filters`. Every call with a DocType that tracks changes raises `OperationalError` ("Unknown column 'tabVersion.order_by'"). Query `Version` with `frappe.get_all` instead.
- `frappe.log_error(message=...)` without `title` raises `TypeError`.

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
