---
title: Document
description: Every public method and property of frappe.model.document.Document in version 15, the hook run order of each operation, behavior flags and the frappe.model helper modules.
version: v15
sources:
  - frappe/model/document.py
  - frappe/model/base_document.py
  - frappe/model/docstatus.py
  - frappe/model/naming.py
  - frappe/model/mapper.py
  - frappe/model/rename_doc.py
  - frappe/model/delete_doc.py
  - frappe/model/utils/__init__.py
  - frappe/model/utils/rename_field.py
  - frappe/model/utils/user_settings.py
  - frappe/model/create_new.py
  - frappe/core/doctype/version/version.py
  - frappe/handler.py
---

# Document

`frappe.model.document.Document` is the base class of every controller. `frappe.get_doc`, `frappe.new_doc` and the other loaders in [frappe namespace](frappe.md#documents) return instances of it. This page lists every public method and property in version 15, the order of controller hooks for each operation, and the flags that change behavior. For the how-to and controller examples, see [Controllers](../../developer-guide/data-model/controllers.md).

`BaseDocument` was merged into `Document`. `frappe.model.document.BaseDocument` and `frappe.model.base_document.BaseDocument` are aliases of `Document`.

## Construction

| Call | Result |
|---|---|
| `Document(doctype, name)` | Load the document and its child rows. Raises `DoesNotExistError`. Pass `for_update=True` to lock the rows (`SELECT ... FOR UPDATE`). |
| `Document(doctype)` | Load a Single DocType. Uses defaults if no row exists. |
| `Document(dict)` or `Document(**fields)` | Make a new unsaved document. Raises `ValueError` ("Illegal arguments") without arguments. |

Controller code uses `frappe.get_doc` and `frappe.new_doc`, which pick the controller class with `get_controller(doctype)`. A custom DocType (`custom = 1`) uses `Document`, or `NestedSet` when `is_tree` is set.

| Function in `frappe.model.document` | Description |
|---|---|
| `get_controller(doctype)` | Return the controller class. Cached per site in `frappe.controllers`. Not cached in the development server or during migrate. Applies the `override_doctype_class` hook (last entry wins, must subclass `Document`). |
| `import_controller(doctype)` | Import the controller class without the cache. |
| `get_doc(*args, **kwargs)` | The function behind `frappe.get_doc`. |
| `bulk_insert(doctype: str, documents: Iterable[Document], ignore_duplicates: bool = False, chunk_size=10_000)` | Insert documents and their child rows with `frappe.db.bulk_insert`. Runs no hooks, no validation, no naming. Sets `owner`, `modified_by`, `creation` and `modified`. `documents` can be a generator. |
| `execute_action(__doctype, __name, __action, **kwargs)` | Worker side of `queue_action`. On error, rolls back and adds a comment "Action Failed". Then unlocks the document. |
| `unlock_document(doctype: str \| None = None, name: str \| None = None, args=None)` | Remove the lock of a queued document. Needs `write`. `args` is an old JSON form with `doctype` and `name`. Whitelisted: `POST`. |

## Properties

| Property | Type | Description |
|---|---|---|
| `doctype`, `name` | `str` | DocType and name. `name` is `None` until naming. |
| `owner`, `creation`, `modified`, `modified_by`, `idx` | | Standard fields. |
| `docstatus` | `DocStatus` | `0` draft, `1` submitted, `2` cancelled. Setting a plain `int` converts it. |
| `flags` | `_dict` | Per-document flags. See [Flags](#flags). |
| `meta` | `Meta` | Cached `frappe.get_meta(self.doctype)`. |
| `permitted_fieldnames` | `set[str]` | Fields the session user may read. Cached. |
| `parent_doc` | `Document \| None` | Parent of a child row. Weak reference, set by `append`. |
| `is_locked` | `bool` | `True` if a lock file exists and is younger than 3 hours. |
| `parent`, `parenttype`, `parentfield` | `str` | Set on child rows. |

`DocStatus` (`frappe.model.docstatus`) is an `int` subclass with `DocStatus.DRAFT`, `DocStatus.SUBMITTED`, `DocStatus.CANCELLED` and the methods `is_draft()`, `is_submitted()`, `is_cancelled()`.

## Persistence

| Method | Description |
|---|---|
| `insert(ignore_permissions=None, ignore_links=None, ignore_if_duplicate=False, ignore_mandatory=None, set_name=None, set_child_names=True) -> Document` | Insert a new document and its child rows. Checks `create` permission (and `submit` if `docstatus` is 1). Runs the [insert hooks](#insert). `ignore_permissions`, `ignore_links` and `ignore_mandatory` set the matching flags. `ignore_if_duplicate` returns quietly on a duplicate name. `set_name` forces a name. |
| `save(ignore_permissions=None, ignore_version=None) -> Document` | Insert if new, else update. Checks `write` permission. Runs the [save hooks](#save). `ignore_version` defaults to `frappe.flags.in_test`. The signature is `save(*args, **kwargs)`, passed to `_save`. |
| `submit() -> Document` | Set `docstatus` to 1 and `save()`. Checks `submit` permission. Whitelisted. |
| `cancel() -> Document` | Set `docstatus` to 2 and `save()`. Checks `cancel` permission. Whitelisted. |
| `delete(ignore_permissions=False, force=False, *, delete_permanently=False)` | Call `frappe.delete_doc` with this document's flags. See [delete_doc](frappe.md#delete_doc). |
| `rename(name: str \| int, merge: bool = False, force: bool = False, validate_rename: bool = True)` | Check `write` permission, call `frappe.model.rename_doc.rename_doc`, then reload. Whitelisted. `validate_rename` is not passed on. |
| `reload() -> Document` | Load all values again from the database. |
| `load_from_db() -> Document` | Same as `reload`. Called by the constructor. |
| `db_set(fieldname, value=None, update_modified=True, notify=False, commit=False)` | Set a value on the object and in the database without the save cycle. See [db_set](#db_set). |
| `db_get(fieldname)` | Return the stored value with `frappe.db.get_value`. |
| `get_db_value(key)` | Same as `db_get`. |
| `db_insert(ignore_if_duplicate=False)` | Raw `INSERT` of this row only. Names the document if `name` is empty. Raises `DuplicateEntryError` or `UniqueValidationError`. Retries up to 5 times for a hash name collision. |
| `db_update()` | Raw `UPDATE` of this row only. Calls `db_insert` for a new row. Does not clear `frappe.db.value_cache`. |
| `db_update_all()` | `db_update` of the parent and every child row. No validation, no hooks. |
| `update_modified()` | Set `modified` to now and write only that column. |
| `update_single(d)` | Replace the rows of a Single DocType in `tabSingles` with `d`. |
| `update_children()` | Write all child tables: delete removed rows, update or insert the others. |
| `update_child_table(fieldname: str, df=None)` | Write one child table. Skips DocTypes in `flags.ignore_children_type`. |
| `deferred_insert() -> None` | Push the row to Redis. A scheduled job inserts it later. Insertion is not guaranteed. |
| `queue_action(action, **kwargs)` | Lock the document and enqueue `action` (or `_action` if it exists, such as `_submit`) for a worker. `enqueue_after_commit` defaults to `True`. Other `kwargs` go to `frappe.enqueue`. |
| `lock(timeout=None)` | Make a lock file for the document. Raises `DocumentLockedError` if a lock younger than 3 hours exists. With `timeout`, retry once per second. |
| `unlock()` | Delete the lock file. |
| `check_if_locked()` | Throw `DocumentLockedError` if `is_locked`. After 30 minutes the message offers a "Force Unlock" action. |

Whitelisted methods run through `frappe.handler.run_doc_method`. The desk calls `/api/method/run_doc_method` with `dt` and `dn`. It loads the document with `frappe.get_doc` and then checks `read` permission. See [REST Endpoints](../http/rest-endpoints.md).

### db_set

| Parameter | Default | Description |
|---|---|---|
| `fieldname` | | Fieldname, or a dict of fieldnames and values. |
| `value` | `None` | New value. Ignored when `fieldname` is a dict. |
| `update_modified` | `True` | Set `modified` and `modified_by`. Not changed while the document is in `frappe.flags.currently_saving`, for example inside `on_update`. |
| `notify` | `False` | Call `notify_update()`. |
| `commit` | `False` | Call `frappe.db.commit()`. |

`db_set` loads `get_doc_before_save()` if it is not loaded, runs `before_change`, writes with `frappe.db.set_value` (or `set_single_value`), then runs `on_change`. It skips validation, permission checks, mandatory checks, Link checks and `Version`. On a new unsaved document it writes nothing to the database.

## Field access

| Method | Description |
|---|---|
| `get(key, filters=None, limit=None, default=None)` | Return a field value, or `default`. For a child table with a `filters` dict, return the matching rows. With a dict as `key`, filter rows of all child tables. A non-dict `filters` is used as `default`. |
| `getone(key, filters=None)` | Return the first row of `get(key, filters, limit=1)`. Raises `IndexError` if none match. |
| `set(key, value, as_value=False)` | Set a field. For a child table, replace all rows with `value` (list of dicts or documents), unless `as_value=True`. Ignores reserved keys such as `doctype` and `flags`. |
| `update(d) -> Document` | Set many fields from a dict. Child table values are turned into row documents. |
| `update_if_missing(d)` | Set fields from `d` only where the current value is `None`. Skips names in `self.dont_update_if_missing`. |
| `append(key: str, value=None, position: int = -1) -> Document` | Add a row to child table `key` and return it. Sets `parent`, `parenttype`, `parentfield`, `idx`, `docstatus`. With `position`, insert there and number all rows again. |
| `extend(key, value)` | Call `append` for each item. Raises `ValueError` if `value` is not iterable. |
| `remove(doc)` | Remove a child row from its table and number the rows again. |
| `delete_key(key)` | Remove an attribute from the object. |
| `get_value(fieldname)` | Return the value cast to the field type. |
| `cast(value, df)` | Cast a value to the type of DocField `df`. |
| `get_all_children(parenttype=None) -> list[Document]` | Return rows of all child tables, or of one child DocType. |
| `get_table_field_doctype(fieldname)` | Return the child DocType of a table field. |
| `get_parentfield_of_doctype(doctype)` | Return the first table fieldname whose options is `doctype`. |
| `get_valid_dict(sanitize=True, convert_dates_to_str=False, ignore_nulls=False, ignore_virtual=False) -> _dict` | Return the column values of this row, cast to their field types. |
| `as_dict(no_nulls=False, no_default_fields=False, convert_dates_to_str=False, no_child_table_fields=False, no_private_properties=False) -> dict` | Return all values and child rows as a `_dict`. Adds `__islocal`, `__onload`, `_user_tags`, `_liked_by`, `__unsaved`, `__run_link_triggers` when set, unless `no_private_properties`. |
| `as_json()` | Return `frappe.as_json(self.as_dict())`. |
| `is_new() -> bool` | Return `True` until the document is inserted (`__islocal` is set). |
| `init_child_tables()` | Set every child table attribute to `[]` if it is `None`. |
| `init_valid_columns()` | Set missing columns to `None`, `docstatus` and `idx` to `0`. |
| `get_valid_columns() -> list[str]` | Return the table columns of the DocType. |
| `init_from_dict(d: dict) -> None` | Set values from `d` without a database read. Calls `__setup__` if defined. |
| `set_onload(key, value)` | Add a value to `__onload`, which the desk form receives. Use it in `onload`. |
| `get_onload(key=None)` | Return one `__onload` value, or all. |
| `get_password(fieldname='password', raise_exception=True)` | Return the decrypted value of a Password field. |
| `is_dummy_password(pwd)` | Return `True` if `pwd` has only `*`. |

```python
doc = frappe.new_doc("Sample Order", customer_name="Customer A", order_date="2026-11-22")
doc.append("items", {"item": "Laptop", "qty": 3})
doc.append("items", {"item": "Monitor", "qty": 1})
[r.item for r in doc.get("items", {"qty": [">", 1]})]
# ['Laptop']
[r.item for r in doc.get({"item": "Monitor"})]
# ['Monitor']
```

## Change tracking

| Method | Description |
|---|---|
| `get_doc_before_save() -> Document \| None` | The stored copy loaded at the start of `save`, `submit`, `cancel` and `db_set`. `None` for a new document. |
| `has_value_changed(fieldname)` | `True` if the field differs from the stored copy. `True` for a new document. |
| `load_doc_before_save(*, raise_exception: bool = False)` | Load the stored copy into `_doc_before_save`. |
| `get_latest()` | Return the stored copy, loading it if needed. |
| `check_if_latest()` | Throw `TimestampMismatchError` if the stored `modified` differs from the loaded one. Then call `check_docstatus_transition`. |
| `check_docstatus_transition(to_docstatus)` | Allow 0 to 0 (save), 0 to 1 (submit), 1 to 1 (update after submit), 1 to 2 (cancel). Other changes raise `DocstatusTransitionError` or `ValidationError`. Checks `submit` or `cancel` permission. |
| `is_child_table_same(fieldname)` | `True` if a child table did not change, other than `modified`, `modified_by` and `creation`. |

## Permissions

| Method | Description |
|---|---|
| `check_permission(permtype='read', permlevel=None)` | Raise `PermissionError` if `has_permission(permtype)` is `False`. `permlevel` is not used. |
| `has_permission(permtype='read', *, debug=False, user=None) -> bool` | Call `frappe.permissions.has_permission` for this document. Returns `True` when `flags.ignore_permissions` is set. |
| `raise_no_permission_to(perm_type)` | Set `frappe.flags.error_message` and raise `PermissionError`. For a child row, names the parent. |
| `get_permissions()` | Return the DocPerm rows of the DocType, or of the parent for a child row. |
| `get_permlevel_access(permission_type='write')` | Return the permlevels where the session user's roles have `permission_type`. |
| `has_permlevel_access_to(fieldname, df=None, permission_type='read')` | `True` if the user has access to the field's permlevel. |
| `apply_fieldlevel_read_permissions()` | Remove values above the user's read permlevels. Does nothing for `Administrator`. |
| `copy_with_fieldlevel_read_permissions() -> Document` | Return a copy with `apply_fieldlevel_read_permissions` applied. |
| `validate_higher_perm_levels()` | Reset fields the user cannot write at permlevel above 0 to their stored or default values. |
| `reset_values_if_no_permlevel_access(has_access_to, high_permlevel_fields)` | Reset helper used by `validate_higher_perm_levels`. |
| `is_whitelisted(method_name)` | Raise `NotFound` if the method does not exist, or `PermissionError` if it is not whitelisted. |

## Validation helpers

Call these from `validate` and other controller methods.

| Method | Description |
|---|---|
| `validate_value(fieldname, condition, val2, doc=None, raise_exception=None)` | Throw if the field value does not meet `condition` (`=`, `!=`, `>`, `in`, `not in`, `^` ...) against `val2`. |
| `validate_table_has_rows(parentfield, raise_exception=None)` | Throw `EmptyTableError` if the child table is empty. |
| `validate_from_to_dates(from_date_field: str, to_date_field: str) -> None` | Throw `InvalidDates` if the "to" date is before the "from" date. |
| `round_floats_in(doc, fieldnames=None, do_not_round_fields=None)` | Round Currency, Float and Percent fields of `doc` to their precision. |
| `precision(fieldname, parentfield=None) -> int \| None` | Return the precision of a Currency, Float or Percent field. Cached per document. |
| `validate_set_only_once()` | Throw `CannotChangeConstantError` if a `set_only_once` field changed. |
| `validate_update_after_submit()` | Throw `UpdateAfterSubmitError` if a field without `allow_on_submit` changed on a submitted document. |
| `validate_workflow()` | Check the Workflow transition and set the workflow state. |
| `validate_amended_from()` | Throw if `amended_from` is not cancelled. |
| `get_invalid_links(is_submittable=False)` | Return `(invalid_links, cancelled_links)` of this row. Also fills `fetch_from` fields. |
| `set_fetch_from_value(doctype, df, values)` | Set one `fetch_from` field from fetched values. |
| `check_no_back_links_exist()` | Throw `LinkExistsError` if a submitted document links to this one. Runs on cancel. |
| `throw_length_exceeded_error(df, max_length, value)` | Throw `CharacterLengthExceededError`. |
| `show_unique_validation_message(e)` | Show "must be unique" and raise `UniqueValidationError`. |
| `get_field_name_by_key_name(key_name)` | MariaDB only. Return the column of a unique index. |
| `get_label_from_fieldname(fieldname)` | Return the translated label, or `None`. |

## Hooks and triggers

| Method | Description |
|---|---|
| `run_method(method: str, *args, **kwargs)` | Run the controller method, `doc_events` handlers for the DocType, then for `"*"`. Then run Notifications, Webhooks and DocType Event Server Scripts for `method`. Raises if `method` starts with `_`. Returns the combined return value. |
| `run_trigger(method, *args, **kwargs)` | Alias of `run_method`. |
| `run_notifications(method)` | Evaluate enabled Notifications of the DocType for the event of `method`. |
| `Document.hook(f)` | Static decorator. Makes `f` extensible by `doc_events`. Dict return values are merged. |
| `run_before_save_methods()` | Run `before_validate` and the `before_*` methods for the current action. |
| `run_post_save_methods()` | Run the `on_*` methods, clear cache, notify, update global search, save `Version`, run `on_change`. |

A `doc_events` handler always gets `(doc, method, *args)`. A handler that accepts only one argument raises `TypeError`. A `frappe.db.commit()` inside a handler or controller method really commits in v15.

## Naming and title

| Method | Description |
|---|---|
| `set_new_name(force=False, set_name=None, set_child_names=True)` | Name the document and its new child rows with `frappe.model.naming.set_new_name`. Does nothing if already named, unless `force`. |
| `set_name_in_children()` | Name child rows that have no name. |
| `set_parent_in_children()` | Set `parent` and `parenttype` on every child row. |
| `get_title()` | Return the value of the DocType title field, or `""`. |
| `set_title_field()` | When `title_field` is `title`, fill it from the field `options` or `default` template. |
| `set_user_and_timestamp()` | Set `modified`, `modified_by`, and for a new document `creation` and `owner`. Copies them to child rows. |
| `set_docstatus()` | Copy `docstatus` to child rows. |

## Activity, sharing and output

| Method | Description |
|---|---|
| `add_comment(comment_type: str = 'Comment', text: str \| None = None, comment_email: str \| None = None, comment_by: str \| None = None)` | Insert a `Comment` on the document with `ignore_permissions=True`. Returns it. Whitelisted. |
| `add_tag(tag)`, `remove_tag(tag)` | Add or remove a Tag. |
| `get_tags()` | Return the Tags as a list of strings. |
| `add_seen(user=None)` | Add the user to `_seen`, if `track_seen` is set. |
| `reset_seen()` | Set `_seen` to the session user. Runs on save when `track_seen` is set. |
| `add_viewed(user=None, force=False, unique_views=False)` | Insert a `View Log`, if `track_views` is set or `force`. |
| `get_liked_by()` | Return the users in `_liked_by`. |
| `get_assigned_users()` | Return the set of users with a ToDo on this document that is not Cancelled. |
| `notify_update()` | Publish `doc_update` (to the document room) and `list_update` after commit. Skipped in a patch. `list_update` is skipped for read-only, Single and child DocTypes. |
| `clear_cache()` | Call `frappe.clear_document_cache(self.doctype, self.name)`. |
| `save_version()` | Insert a `Version` with the changes, when `track_changes` is set and `flags.ignore_version` is not. |
| `log_error(title=None, message=None, *, defer_insert=False)` | Call `frappe.log_error` with this document as reference. |
| `get_url()` | Return the desk path, for example `/app/sample-order/SO-00001`. |
| `get_signature()` | Return a SHA-224 hash of `doctype:name`. Used for lock files and private links. |
| `get_document_share_key(expires_on=None, no_expiry=False)` | Return the key of a `Document Share Key` for this document. Inserts one if none exists. |
| `get_formatted(fieldname, doc=None, currency=None, absolute_value=False, translated=False, format=None)` | Return the display value of a field with `frappe.format_value`. |
| `is_print_hide(fieldname, df=None, for_print=True)` | `True` if the field is hidden in print. |
| `in_format_data(fieldname)` | Always `True`. Kept for old print templates. |
| `copy_attachments_from_amended_from()` | Copy the File attachments of `amended_from`. Runs on insert of an amendment. |
| `remove_unpicklable_values(state)` | Remove cached properties from a pickled state. |

## Hook run order

Verified on a v15 site by patching `run_method` on `Sample Order` and running each operation. For each step, Frappe runs the controller method, then `doc_events`, then Notifications, Webhooks and Server Scripts. For the standard checks between the steps, see [Controllers](../../developer-guide/data-model/controllers.md#run-order).

### Insert

`before_insert` > `before_naming` > `autoname` > `before_validate` > `validate` > `before_save` > row insert > `after_insert` > `on_update` > `on_change`

With `docstatus` 1 on insert, `before_submit` replaces `before_save` and `on_submit` runs after `on_update`.

### Save

`before_validate` > `validate` > `before_save` > row update > `on_update` > `on_change`

### Submit

`before_validate` > `validate` > `before_submit` > row update > `on_update` > `on_submit` > `on_change`

### Update after submit

`before_update_after_submit` > row update > `on_update_after_submit` > `on_change`

### Cancel

`before_cancel` > row update > `on_cancel` > back link check > `on_change`

### Delete

`on_trash` > `on_change` > row delete > `after_delete`

### db_set

`before_change` > `on_change`

### Rename

`before_rename(old, new, merge)` > rename > `after_rename(old, new, merge)`

With `flags.ignore_validate`, a save runs `before_validate` > `on_update` > `on_change`.

## Flags

Set these on `doc.flags` before the operation.

| Flag | Effect |
|---|---|
| `ignore_permissions` | `has_permission` returns `True`. Skips permlevel resets. Set by `insert(ignore_permissions=True)` and `save(ignore_permissions=True)`. |
| `ignore_links` | Skip Link validation and the back link check on cancel. Set by `insert(ignore_links=True)`. |
| `ignore_mandatory` | Skip the mandatory field check. Set by `insert(ignore_mandatory=True)`. |
| `ignore_validate` | Skip `validate`, `before_save`, `before_submit`, `before_cancel`, `before_update_after_submit` and `set_title_field`. `before_validate` and the standard checks still run. |
| `ignore_validate_update_after_submit` | Allow changes to fields without `allow_on_submit` on a submitted document. |
| `ignore_validate_constants` | Skip the `set_only_once` check. |
| `ignore_version` | Do not save a `Version`. Set by `save(ignore_version=...)`. |
| `ignore_children_type` | List of child DocTypes whose rows are not written on save. |
| `ignore_save_passwords` | `True`, or a list of fieldnames, to skip storing Password fields. |
| `ignore_permlevel_for_fields` | List of fieldnames not reset by permlevel checks. |
| `notify_update` | `False` skips `notify_update` after save. |
| `for_update` | Rows were loaded with `FOR UPDATE`. Set by `get_doc(..., for_update=True)`. |
| `updater_reference` | Dict stored on the `Version`. A new document gets a `Version` when this is set. |
| `in_insert` | Read only. `True` while insert runs. |
| `in_delete` | Read only. `True` in `on_change` and later during delete. |
| `name_set` | Read only. `True` after naming. |

## frappe.model.naming

| Function | Description |
|---|---|
| `set_new_name(doc)` | Name a document: `before_naming`, autoincrement, amendment suffix, Single name, Document Naming Rule, controller `autoname`, the DocType `autoname` property, else a hash. Validates the result. |
| `make_autoname(key='', doctype='', doc='', *, ignore_validate=False)` | Return the next name for a series key such as `SAMPLE-.YYYY.-.####`. `key="hash"` returns a 10-character random name. Increments `tabSeries`. |
| `parse_naming_series(parts, doctype=None, doc=None, number_generator=None) -> str` | Resolve a naming series (list of parts or `.`-separated string) to the next name. |
| `getseries(key, digits)` | Increment the `tabSeries` counter of `key` and return it zero-padded. |
| `revert_series_if_last(key, name, doc=None)` | Decrement the series counter if `name` is the last one. Runs on delete. |
| `set_name_by_naming_series(doc)` | Name from the `naming_series` field. Throws if it is empty. |
| `set_name_from_naming_options(autoname, doc)` | Name from an `autoname` value: `field:`, `naming_series:`, `Prompt`, `format:` or a series. |
| `set_naming_from_document_naming_rule(doc)` | Name from the first matching Document Naming Rule. |
| `get_default_naming_series(doctype: str) -> str \| None` | Return the first non-empty `naming_series` option. |
| `validate_name(doctype: str, name: int \| str)` | Strip and check a name. Throws `NameError` for empty names, `<` or `>`, or a name equal to the DocType. |
| `append_number_if_name_exists(doctype, value, fieldname='name', separator='-', filters=None)` | Return `value`, or `value-N` with the next free number. |
| `is_autoincremented(doctype: str, meta=None) -> bool` | `True` if `autoname` is `autoincrement`. |
| `determine_consecutive_week_number(datetime)` | Return the week number for the `WW` series part. |
| `has_custom_parser(e)` | `True` if the `naming_series_variables` hook has a parser for part `e`. |
| `NamingSeries(series: str)` | Class with `validate()` and `generate_next_name(doc, *, ignore_validate=False)`. |
| `InvalidNamingSeriesError` | Exception for an invalid series. |

```python
from frappe.model.naming import make_autoname

make_autoname("hash")                                  # '51dcotarfd'
make_autoname("SAMPLE-.YYYY.-.####", "Sample Order")   # for example "SAMPLE-2026-0001". Increments tabSeries.
```

See [Naming](../../developer-guide/data-model/naming.md).

## frappe.model.mapper

| Function | Description |
|---|---|
| `get_mapped_doc(from_doctype, from_docname, table_maps, target_doc=None, postprocess=None, ignore_permissions=False, ignore_child_tables=False, cached=False)` | Return a new unsaved document mapped from a source document. Checks `create` on the target and `read` on the source. Runs `before_mapping` and `after_mapping` on the target. |
| `map_doc(source_doc, target_doc, table_map, source_parent=None)` | Map one document with one table map. |
| `map_fields(source_doc, target_doc, table_map, source_parent)` | Copy fields with the same name, `field_map` fields and `fetch_from` values. |
| `map_fetch_fields(target_doc, df, no_copy_fields)` | Fill `fetch_from` fields of one Link field. |
| `map_child_doc(source_d, target_parent, table_map, source_parent=None)` | Map one child row and append it to the target. |
| `make_mapped_doc(method: str, source_name: str, selected_children: str \| None = None, args: str \| None = None)` | Call a whitelisted mapper method. `args` go to `frappe.flags.args`. Whitelisted. |
| `map_docs(method: str, source_names: str, target_doc, args: str \| None = None)` | Call a whitelisted mapper method once per source on one target. Whitelisted. |

`table_maps` is a dict keyed by source DocType. Keys of each map:

| Key | Description |
|---|---|
| `doctype` | Target DocType. Required. |
| `field_map` | Dict (or list of pairs) of source field to target field. |
| `field_no_map` | Fields not to copy. |
| `validation` | `{"field": ["=", value]}`. Throws if the source does not match. |
| `postprocess` | `fn(source, target, source_parent)` for this map. |
| `condition` | `fn(source_row)`. Rows where it returns falsy are skipped. |
| `filter` | `fn(source_row)`. Rows where it returns truthy are skipped. |
| `ignore` | Skip all rows of this child DocType. |
| `add_if_empty` | Map rows only if the target table is empty. |
| `reset_value` | Empty the target table first. |

A child table with no map is copied when source and target have the same fieldname and child DocType and neither field is `no_copy`.

```python
from frappe.model.mapper import get_mapped_doc

@frappe.whitelist()
def make_copy(source_name: str):
	return get_mapped_doc(
		"Sample Order",
		source_name,
		{
			"Sample Order": {"doctype": "Sample Order", "field_no_map": ["status"]},
			"Sample Order Item": {"doctype": "Sample Order Item"},
		},
	)
```

## frappe.model.rename_doc

| Function | Description |
|---|---|
| `rename_doc(doctype: str \| None = None, old: str \| None = None, new: str \| None = None, force: bool = False, merge: bool = False, ignore_permissions: bool = False, ignore_if_exists: bool = False, show_alert: bool = True, rebuild_search: bool = True, doc: Document \| None = None, validate: bool = True) -> str` | Rename a document. Pass `doctype`, `old`, `new`, or `doc`, `new`. See the steps below. |
| `validate_rename(doctype, old, new, meta, merge, force=False, ignore_permissions=False, ignore_if_exists=False, save_point=False, old_doc=None) -> str` | Check a rename and return the validated new name. |
| `update_document_title(*, doctype: str, docname: str, title: str \| None = None, name: str \| None = None, merge: bool = False, enqueue: bool = False, **kwargs) -> str` | Change the title, the name, or both. With `enqueue`, the rename runs in the `long` queue. Needs `write`. Whitelisted. |
| `bulk_rename(doctype: str, rows: list[list] \| None = None, via_console: bool = False) -> list[str] \| None` | Rename from rows of `(old, new, merge)`. Commits after each row. Max 500 rows unless `via_console`. |
| `get_link_fields(doctype: str) -> list[dict]` | Return Link fields in all DocTypes, Custom Fields and Property Setters that point to `doctype`. |
| `update_link_field_values`, `rename_dynamic_links`, `update_user_settings`, `update_attachments`, `rename_versions`, `rename_eps_records`, `update_assignments`, `update_customizations`, `rename_doctype`, `rename_parent_and_child`, `update_autoname_field`, `update_child_docs`, `update_options_for_fieldtype`, `get_select_fields`, `update_select_field_values`, `update_parenttype_values` | Steps of `rename_doc`. |

`rename_doc` steps:

1. With `validate`, run `before_rename(old, new, merge)`. A returned string, or a dict with `new`, changes the new name.
2. With `validate`, check: not a Single, `old` exists, `new` is different, `new` exists only with `merge` (unless `ignore_if_exists`), `write` permission on both, and `allow_rename` unless `force` or `ignore_permissions`.
3. Rename the row and child rows, or move assignments with `merge`.
4. Update Link and Dynamic Link values, user settings, attachments, versions and Energy Point Log records.
5. Run `after_rename(old, new, merge)`, rename stored passwords and add an "Edit" comment.
6. With `merge`, delete the old document.
7. Clear the cache of the whole site, enqueue the global search rebuild and show an alert. No realtime rename event is sent.

## frappe.model.delete_doc

`delete_doc` is described in [frappe namespace](frappe.md#delete_doc).

| Function | Description |
|---|---|
| `get_linked_docs(doc, method='Delete') -> list[dict]` | Return documents that link to `doc` through Link fields. |
| `get_dynamic_linked_docs(doc, method='Delete') -> list[dict]` | Return documents that link to `doc` through Dynamic Link fields. |
| `check_if_doc_is_linked(doc, method='Delete')` | Raise `LinkExistsError` if a document links to `doc`. On `Cancel`, only submitted documents count, and DocTypes in `doc.ignore_linked_doctypes` are skipped. |
| `check_if_doc_is_dynamically_linked(doc, method='Delete')` | Same for Dynamic Links. |
| `raise_link_exists_exception(doc, reference_doctype, reference_docname, row='')` | Throw the `LinkExistsError` message. |
| `check_permission_and_not_submitted(doc)` | Check `delete` permission and throw for submitted documents. |
| `add_to_deleted_document(doc)` | Insert a `Deleted Document` copy. |
| `update_naming_series(doc)` | Revert the naming series if this was the last number. |
| `delete_from_table(doctype: str, name: str, ignore_doctypes: list[str], doc)` | Delete the row and its child rows. |
| `update_flags(doc, flags=None, ignore_permissions=False)` | Copy `flags` to `doc.flags`. |
| `delete_dynamic_links(doctype, name)` | Delete ToDo, Comment, Version, DocShare, View Log and other rows that reference a deleted document. Clear Communication and Activity Log references. Enqueued by `delete_doc`. |
| `delete_references(doctype, reference_doctype, reference_name, reference_doctype_field='reference_doctype', reference_name_field='reference_name')` | Delete rows that reference a document. |
| `clear_references(doctype, reference_doctype, reference_name, reference_doctype_field='reference_doctype', reference_name_field='reference_name')` | Set references to a document to `NULL`. |
| `clear_timeline_references(link_doctype, link_name)` | Delete timeline links to a document. |
| `insert_feed(doc)` | Add a deletion feed entry. |
| `delete_controllers(doctype, module)` | Delete the controller folder of a DocType. |

## frappe.model.utils

| Function | Description |
|---|---|
| `set_default(doc, key)` | Set `is_default` on `doc` and clear it on all others with the same `key` value. |
| `set_field_property(filters, key, value)` | Set a DocField property on all matching fields, save the DocTypes and commit. |
| `get_fetch_values(doctype, fieldname, value)` | Return the `fetch_from` values for a Link value. |
| `is_virtual_doctype(doctype: str)` | `True` for a virtual DocType. Cached per site. |
| `is_single_doctype(doctype: str) -> bool` | `True` for a Single DocType. Cached per site. |
| `render_include(content)` | Resolve `{% include "app/path" %}` in JS content. Raises `InvalidIncludePath`. |
| `frappe.model.utils.rename_field.rename_field(doctype, old_fieldname, new_fieldname, validate=True)` | Move data from an old column to a new field after the DocType is synced. Updates reports, Property Setters and user settings. Use it in patches. |
| `frappe.model.utils.user_settings.get_user_settings(doctype, for_update=False)` | Return list view settings of the session user. `get` and `save` in that module are whitelisted. |

See [Fixtures and Patches](../../developer-guide/building-apps/fixtures-and-patches.md) for `rename_field` in patches.

## Deprecated

| Name | Status | Replacement |
|---|---|---|
| `super(Document, self).__init__(d)` in a controller | Warns | `self.init_from_dict(d)`. |
| `BaseDocument` | Alias | `Document`. |
| `DocStatus.draft()`, `submitted()`, `cancelled()` | Kept, no warning | `DocStatus.DRAFT`, `SUBMITTED`, `CANCELLED`. |
| `frappe.handler.runserverobj` | Alias | `run_doc_method`. |

## Pitfalls

- `doc.db_set` on a new unsaved document writes nothing. It returns `None` and gives no error.
- `frappe.get_doc` does not check permissions. Call `doc.check_permission()`.
- A `frappe.db.commit()` in a controller method or `doc_events` handler commits. A later error cannot roll it back.
- `db_update` and raw SQL do not clear `frappe.db.value_cache`. A cached `get_value` in the same transaction can return the old value.

## Related

- [frappe namespace](frappe.md)
- [Controllers](../../developer-guide/data-model/controllers.md)
- [DocTypes](../../developer-guide/data-model/doctypes.md)
- [Naming](../../developer-guide/data-model/naming.md)
- [Database](database.md)
- [Permissions and Sessions](permissions-and-sessions.md)
