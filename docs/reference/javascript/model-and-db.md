---
title: JavaScript Model and DB API
description: Reference for frappe.model, locals, frappe.db, frappe.model.user_settings, frappe.perm and frappe.workflow in desk JavaScript, with signatures, return values and side effects.
version: develop
sources:
  - frappe/public/js/frappe/model/model.js
  - frappe/public/js/frappe/model/create_new.js
  - frappe/public/js/frappe/model/sync.js
  - frappe/public/js/frappe/model/user_settings.js
  - frappe/public/js/frappe/model/perm.js
  - frappe/public/js/frappe/model/workflow.js
  - frappe/public/js/frappe/db.js
  - frappe/public/js/frappe/provide.js
  - frappe/client.py
  - frappe/desk/reportview.py
  - frappe/desk/form/load.py
  - frappe/model/mapper.py
  - frappe/model/utils/user_settings.py
---

# JavaScript Model and DB API

This page lists the client-side API for documents in desk JavaScript. `frappe.model` works on documents that the browser has loaded into `locals`. `frappe.db` sends requests to the server and returns promises. Use `frappe.model` in form scripts to change the open document. Use `frappe.db` to read or write other records. For how-to guides, see [Client API](../../developer-guide/desk-ui/client-api.md) and [Form Scripts](../../developer-guide/desk-ui/form-scripts.md).

The names on this page come from `Object.keys()` on each namespace in a logged-in desk at `/desk` on develop. Every example was run in that desk.

## locals

`locals` is a global object. It holds every document that the browser loaded, by DocType and name: `locals[doctype][name]`.

| Key | Content |
|---|---|
| `locals[doctype][name]` | A document as a plain object. Child rows are also stored here, under their child DocType. |
| `locals.DocType[doctype]` | The meta of a loaded DocType. `frappe.get_meta(doctype)` reads it. |
| `locals[":" + doctype]` | Partial documents sent in boot, for example `locals[":Print Format"]`. |

Keys of a local document that start with `__` are client-only flags:

| Key | Meaning |
|---|---|
| `__islocal` | `1` if the document is not saved on the server yet. Its name starts with `new-`. |
| `__unsaved` | `1` if the document has changes that are not saved. |
| `__unedited` | `true` on a new grid row that the user has not changed. |
| `__last_sync_on` | `Date` of the last `frappe.model.sync` for this document. |
| `__run_link_triggers` | Set on a new document. The form runs Link field triggers for default values once. |
| `__newname` | The name that the user typed for a DocType with `autoname` = `prompt`. |
| `__needs_refresh` | Set when another user changed the document. The form reloads or shows a message. |
| `__checked` | `1` if the row is selected in a grid. |

!!! note
    `locals` holds only documents that the browser loaded. Call `frappe.model.with_doc` or `frappe.db.get_doc` before you read a record with `frappe.get_doc`.

## frappe.model: load documents and meta

| Function | Description |
|---|---|
| `with_doctype(doctype, callback, async)` | Load the meta of `doctype` and of its child DocTypes into `locals.DocType`, then call `callback(r)`. Returns a promise. If the meta is loaded, it calls `callback()` and returns a resolved promise. Calls `frappe.desk.form.load.getdoctype` (`GET`). Also loads list, calendar, tree and Kanban scripts and the user settings of the DocType. |
| `with_doc(doctype, name, callback, error_callback)` | Load a document, its child rows and its `docinfo` into `locals`. Returns a promise that resolves with the document. Calls `frappe.desk.form.load.getdoc` (`GET`), which checks read permission. Uses the local copy if the document and its `docinfo` are already loaded and you pass no `error_callback`. With no `name`, loads the Single DocType `doctype`. `callback(name, r)`. `error_callback(r, permission_denied)`. On `PermissionError`, removes the document from `locals`. |
| `sync(r)` | Put documents from a server response into `locals` and `docinfo` into `frappe.model.docinfo`. `r` is a document, a list of documents, or `{docs, docinfo}`. Updates a document that is already loaded in place. Renames a local document when the response has `localname`. Returns the list of documents. |
| `get_docinfo(doctype, name)` | Return the `docinfo` of a loaded document, or `null`. Keys include `attachments`, `comments`, `communications`, `versions`, `assignments`, `shared`, `permissions`, `tags`, `views` and `is_document_followed`. |
| `set_docinfo(doctype, name, key, value)` | Set one key of a loaded `docinfo`. Does nothing if the `docinfo` is not loaded. |
| `get_shared(doctype, name)` | Return `docinfo.shared` of a loaded document. Throws if the `docinfo` is not loaded. |
| `is_fresh(doc)` | Return `true` if the document was synced less than 5 seconds ago. |

```js
const doc = await frappe.model.with_doc("Sample Item", "Keyboard");
doc.rate; // 45
frappe.get_doc("Sample Item", "Keyboard") === doc; // true
```

## frappe.model: read local documents

These functions read `locals` only. They do not send a request, except `get_value` with a callback.

| Function | Description |
|---|---|
| `get_doc(doctype, name)` | Return `locals[doctype][name]`, or `null`. With no `name`, `name` is `doctype` (Single DocType). If `name` is an object, return the first local document that matches these filters. Also `frappe.get_doc`. |
| `get_list(doctype, filters)` | Return the local documents of `doctype` that match `filters`. Reads `locals[":" + doctype]` if `locals[doctype]` does not exist. Returns `[]` if nothing is loaded. Also `frappe.get_list`. |
| `get_children(doc, parentfield, filters)` or `get_children(doctype, parent, parentfield, filters)` | Return the rows of a child table of a local document, filtered by `filters` if given. Also `frappe.get_children`. |
| `get_value(doctype, filters, fieldname, callback)` | Without `callback`: return `fieldname` of the local document with name `filters`, or of the first local match of `filters`, or `null`. With `callback`: call `frappe.client.get_value` and pass `r.message` to `callback`. Returns `undefined` in that case, not a promise. |
| `has_value(doctype, name, fieldname)` | Return `true` if the field has a value. For a Table field, return `true` if the table has rows. |
| `get_doc_title(doc)` | Return `"New <DocType>"` for an unsaved document. Else return the value of the title field, or the name. Needs the meta of the DocType. |
| `get_title_from_title_field(doc, meta)` | Return the title field value. For a Link title field, return the cached link title. |
| `get_all_docs(doc)` | Return `[doc, ...all child rows]`. |
| `get_no_copy_list(doctype)` | Return the fieldnames that `copy_doc` skips: `name`, `amended_from`, `amendment_date`, `cancel_reason` and fields with `no_copy`. |

```js
frappe.model.get_value("Sample Item", "Keyboard", "rate");             // 45
frappe.model.get_value("Sample Item", { category: "Hardware" }, "name"); // "Keyboard"
frappe.model.get_list("Sample Item", { category: "Hardware" });          // only rows in locals
```

## frappe.model: create and change local documents

These functions change `locals` only. Save the document to write it to the database.

| Function | Description |
|---|---|
| `get_new_doc(doctype, parent_doc, parentfield, with_mandatory_children)` | Make a new local document with default values and add it to `locals`. Sets `docstatus: 0`, `__islocal: 1`, `__unsaved: 1`, `owner` and a `new-<doctype>-<random>` name. With `parent_doc`, adds it as a row of `parent_doc[parentfield]`. With `with_mandatory_children`, adds one empty row to each mandatory Table field. Applies `frappe.route_options` as values (only fields that exist and are not `no_copy`), then clears `frappe.route_options`. Returns the document. |
| `make_new_doc_and_get_name(doctype, with_mandatory_children)` | Same as `get_new_doc` for a parent document. Returns the name. |
| `get_new_name(doctype)` | Return a new local name, for example `new-sample-order-zglqhgctzg`. |
| `add_child(parent_doc, doctype, parentfield, idx)` or `add_child(parent_doc, parentfield)` | Add a new row to a child table and return it. With two arguments, the child DocType comes from the Table field. With `idx`, inserts before that position and renumbers. Marks `cur_frm` dirty if it shows `parent_doc`. Does not render the grid and does not run `<table>_add`. |
| `clear_table(doc, parentfield)` | Remove all rows of a child table from the document and from `locals`. Does not render. |
| `copy_doc(doc, from_amend, parent_doc, parentfield)` | Return a new local copy of a document and its child rows. Skips `name`, `amended_from`, `amendment_date`, `cancel_reason`, keys that start with `__`, and Password fields. Skips `no_copy` fields unless `from_amend` is set. Clears `creation`, `modified`, `lft` and `rgt`, and sets `owner` and `modified_by` to the current user. |
| `set_value(doctype, docname, fieldname, value, fieldtype, skip_dirty_trigger = false)` | Set a value on a local document and run the change handlers. `fieldname` can be an object `{fieldname: value}`. Returns a promise. Does nothing for a key whose value is the same, except for a Link or Dynamic Link when you pass that `fieldtype`. With `skip_dirty_trigger`, the form is not marked dirty. |
| `set_default_values(doc, parent_doc)` | Set default values on fields that are empty. Returns the list of fieldnames it set. |
| `get_default_value(df, doc, parent_doc)` | Return the default of one field. Checks user permissions, user defaults, the last selected value and `df.default` (`__user`, `user_fullname`, `Today`, `now`, `:DocType`). |
| `create_mandatory_children(doc)` | Add one empty row to each mandatory Table field. |
| `round_floats_in(doc, fieldnames)` | Round fields to their precision. Without `fieldnames`, rounds all Currency and Float fields. Needs the meta. |
| `validate_missing(doc, fieldname)` | Call `frappe.throw` if the field is empty. |

```js
await frappe.model.with_doctype("Sample Order");
const order = frappe.model.get_new_doc("Sample Order");
const row = frappe.model.add_child(order, "items");
row.doctype;     // "Sample Order Item"
row.parentfield; // "items"
await frappe.model.set_value(row.doctype, row.name, "qty", 2);
```

!!! warning
    Inside a form, `frappe.model.set_value` runs the form field events of the document. For a child row, it runs the event on the child DocType. It does not fetch `fetch_from` values of a Link field in a row. The server fetches them when the document is saved.

## frappe.model: change handlers

| Function | Description |
|---|---|
| `on(doctype, fieldname, fn)` | Run `fn(fieldname, value, doc, skip_dirty_trigger)` after `set_value` changes `fieldname` on a document of `doctype`. Use `"*"` for all fields. The form uses this to run form events. |
| `trigger(fieldname, value, doc, skip_dirty_trigger = false)` | Run the handlers of `fieldname` and `"*"` for `doc.doctype`, one after the other. Returns a promise. |
| `events` | The registered handlers, by DocType and fieldname. |

Prefer `frappe.ui.form.on` in form scripts. See [Form API](form.md).

## frappe.model: server actions

| Function | Description |
|---|---|
| `delete_doc(doctype, docname, callback)` | Show a confirm dialog. On confirm, call `frappe.client.delete`, remove the document from `locals`, then call `callback(r, rt)`. |
| `rename_doc(doctype, docname, callback)` | Open the rename dialog. On confirm, call `frappe.rename_doc` with the new name and the merge flag. Updates `locals` and triggers the `rename` event on `document`. Calls `callback(new_name)`. |
| `open_mapped_doc(opts)` | Call `frappe.model.mapper.make_mapped_doc` (`POST`) and open the new document. See the options below. |
| `add_mapped_doc_guard(fn)` | Add `fn(mapped_doc, opts)`. If it returns `false` (or a promise of `false`), `open_mapped_doc` does not open the new document. An error in a guard is logged and ignored. |
| `remove_mapped_doc_guard(fn)` | Remove a guard. |
| `should_open_mapped_doc(mapped_doc, opts)` | Run all guards. Returns a promise of `true` or `false`. |

Options of `open_mapped_doc(opts)`:

| Option | Description |
|---|---|
| `method` | Dotted path of a whitelisted mapper method on the server. |
| `frm` | The source form. If the form has unsaved changes, the call throws. Its name is the default `source_name`. Its selected grid rows are sent as `selected_children`. |
| `source_name` | Name of the source document. |
| `args` | Extra arguments for the mapper method. |
| `freeze_message` | Message while the screen is frozen. |
| `run_link_triggers` | Run Link field triggers on the new document when it opens. |

```js
// sample_app/sample_module/doctype/sample_order/sample_order.js
frappe.model.open_mapped_doc({
	method: "sample_app.sample_module.doctype.sample_order.sample_order.make_invoice",
	frm: cur_frm,
});
```

## frappe.model: DocType checks and permissions

The `can_*` functions read the lists in `frappe.boot.user`. They check the DocType, not one document. For one document, use `frm.has_perm(ptype)` or `frappe.perm.has_perm`.

| Function | Description |
|---|---|
| `can_create(doctype)`, `can_read(doctype)`, `can_write(doctype)`, `can_delete(doctype)`, `can_submit(doctype)`, `can_cancel(doctype)`, `can_select(doctype)`, `can_get_report(doctype)` | Return `true` if the user has this right on the DocType. |
| `can_import(doctype, frm, meta = null)` | `false` if `meta` is given and does not allow import. `true` for System Manager. With `frm`, reads `frm.perm[0].import`. |
| `can_export(doctype, frm)` | `true` for System Manager. With `frm`, reads `frm.perm[0].export`. |
| `can_print(doctype, frm)`, `can_email(doctype, frm)` | With `frm`, read `frm.perm[0]`. Else read `frappe.boot.user`. |
| `can_share(doctype, frm)` | `false` if document sharing is disabled and the user is not Administrator. With `frm`, reads `frm.perm[0].share`. |
| `can_print_docstatus(doctype, docstatus)` | Return `true` if a document in this `docstatus` can print, as set in Print Settings. |
| `can_print_doc(frm)` | Return `true` if the form document can print: docstatus, print permission and not a Single. |
| `is_submittable(doctype)`, `is_table(doctype)`, `is_tree(doctype)` | Return the flag from the loaded meta. Return `undefined` if the meta is not loaded. |
| `is_single(doctype)` | Return `true` if the DocType is in `frappe.boot.single_types`. |
| `has_workflow(doctype)` | Return the number of active Workflow records in `locals` for the DocType. |

!!! warning
    `frappe.model.can_share(doctype)` without `frm` throws a `TypeError` on develop, because `frappe.boot.user` has no `can_share` list. Pass `frm`, or use `frappe.perm.has_perm(doctype, 0, "share")`.

## frappe.model: fieldtype helpers and constants

| Name | Description |
|---|---|
| `scrub(txt)` | Replace spaces with `_` and make lowercase: `"Sample Order"` gives `"sample_order"`. |
| `unscrub(txt)` | Replace `_` and `-` with spaces and capitalize words. |
| `is_value_type(fieldtype)` | `false` for fieldtypes that store no value, such as `Table`, `Section Break` and `HTML`. Accepts a DocField. |
| `is_numeric_field(fieldtype)` | `true` for `Int`, `Float`, `Currency`, `Percent` and `Duration`. Accepts a DocField. |
| `is_non_std_field(fieldname)` | `false` for standard fields such as `name`, `owner`, `docstatus`, `parent`. |
| `get_std_field(fieldname, ignore = false)` | Return the DocField of a standard field. Shows a message for an unknown column unless `ignore` is set. |
| `get_full_column_name(fieldname, doctype)` | Return `` `tab<doctype>`.`<fieldname>` ``. |
| `get_server_module_name(doctype)` | Return the Python module of the DocType controller, for example `frappe.desk.doctype.todo.todo`. Needs the meta. |
| `all_fieldtypes`, `no_value_type`, `layout_fields`, `table_fields`, `numeric_fieldtypes`, `html_fieldtypes`, `plain_text_fieldtypes` | Lists of fieldtypes. |
| `std_fields`, `std_fields_list`, `std_fields_table`, `child_table_field_list`, `restricted_fields`, `core_doctypes_list` | Lists of standard fields and core DocTypes. |
| `docinfo`, `new_names`, `user_settings` | Loaded `docinfo` by DocType and name, map of local names to saved names, and loaded user settings. |
| `DocTypeController` | Form controller class of the DocType form. |

## frappe.model: internal functions

Desk calls these. Apps rarely need them: `init`, `init_doctype`, `add_to_locals`, `update_in_locals`, `remove_from_locals`, `clear_doc`, `delete_from_locals`, `rename_doc_in_locals`, `rename_after_save`, `sync_docinfo`, `sync_print_format_for_meta`, `get_default_from_boot_docs`, `set_default_views_for_doctype`.

`remove_from_locals(doctype, name)` and `clear_doc(doctype, name)` remove a document from `locals`. `clear_doc` also removes a child row from its parent and renumbers `idx`.

## frappe.model.user_settings

User settings store list, report and grid settings per user and DocType on the server. `frappe.model.user_settings[doctype]` holds the loaded settings.

| Function | Description |
|---|---|
| `get(doctype)` | Call `frappe.model.utils.user_settings.get`. Returns a promise of the settings object. |
| `save(doctype, key, value)` | Set `key`. If `value` is a plain object, merge it into the existing value. Saves only if something changed. Returns a promise. Does nothing for Guest. |
| `remove(doctype, key)` | Delete `key` and save. |
| `update(doctype, user_settings)` | Save the whole settings object with `frappe.model.utils.user_settings.save`, then store the response in `frappe.model.user_settings[doctype]`. |
| `frappe.get_user_settings(doctype, key)` | Return the loaded settings, or one key of them. Returns `{}` if not loaded. No request. |

```js
await frappe.model.user_settings.save("Sample Item", "sample_app", { compact: 1 });
frappe.get_user_settings("Sample Item", "sample_app").compact; // 1
await frappe.model.user_settings.remove("Sample Item", "sample_app");
```

## frappe.db

Each function calls a whitelisted method on the server. The server checks the permissions of the current user. Each function returns a promise.

| Function | Server method | Resolves with |
|---|---|---|
| `get_list(doctype, args)` | `frappe.desk.reportview.get_list` (`GET`) | List of row objects. |
| `get_value(doctype, filters, fieldname, callback, parent_doc)` | `frappe.client.get_value` (`GET`) | The response `r`. The values are in `r.message`. `callback(r.message)` if given. |
| `get_single_value(doctype, field)` | `frappe.client.get_single_value` (`GET`) | The value. |
| `get_doc(doctype, name, filters)` | `frappe.client.get` (`GET`) | The document. Also adds it to `locals`. Rejects on error. |
| `exists(doctype, name_or_filters)` | `get_value` or `count` | `true` or `false`. |
| `count(doctype, {filters, limit} = {}, cache = false)` | `frappe.desk.reportview.get_count` | A number. |
| `insert(doc)` | `frappe.client.insert` (`POST`) | The inserted document. |
| `set_value(doctype, docname, fieldname, value, callback)` | `frappe.client.set_value` (`POST`) | The response `r`. The saved document is in `r.message`. |
| `delete_doc(doctype, name)` | `frappe.client.delete` (`POST`) | `r.message`. Removes the document from `locals`. |
| `get_link_options(doctype, txt = "", filters = {}, page_length = 0, include_image = false)` | `frappe.desk.search.search_link` (`GET`) | List of `{value, label, description}`. |

### get_list

`args` keys:

| Key | Default | Description |
|---|---|---|
| `fields` | `["name"]` | Fields to read. |
| `filters` | none | Object, or list of `[doctype, field, operator, value]` or `[field, operator, value]`. |
| `or_filters` | none | Filters joined with `OR`. |
| `order_by` | server default | For example `"rate asc"`. |
| `group_by` | none | Group by a field. Write aggregates in `fields` in the dict form, for example `{COUNT: "*", as: "n"}`. |
| `limit` | `20` | Maximum rows. Pass `0` for no limit. |
| `start` or `limit_start` | `0` | Offset. |

The server passes the arguments to `DatabaseQuery.execute`, which applies permissions.

```js
await frappe.db.get_list("Sample Item", {
	fields: ["name", "rate"],
	filters: { category: "Hardware" },
	order_by: "rate asc",
	limit: 2,
});
// [{ name: "Keyboard", rate: 45 }, { name: "Monitor", rate: 250 }]
```

!!! warning
    `frappe.db.get_list` does not reject when the server returns an error. The promise stays pending. The error dialog shows, but code after `await` does not run. Use `frappe.xcall("frappe.client.get_list", {...})` if you must handle the error.

### get_value and exists

`filters` is a name or a filters object. `fieldname` is a fieldname or a list. For a child DocType, pass the parent DocType as `parent_doc`.

```js
const r = await frappe.db.get_value("Sample Item", "Keyboard", ["rate", "category"]);
r.message; // { rate: 45, category: "Hardware" }

(await frappe.db.get_value("Sample Item", "No Such Item", "rate")).message; // {}

await frappe.db.exists("Sample Item", "Keyboard");             // true
await frappe.db.exists("Sample Item", { category: "Service" }); // true
```

`exists` with a string calls `get_value` for `name`. With an object it calls `count` with `limit: 1`.

### count

`count` sends a `POST` request. With `cache = true` it sends a `GET` request that the browser can cache. If a filter is on a child DocType, the count is distinct. If you pass `limit`, the server stops counting at that number.

```js
await frappe.db.count("Sample Item", { filters: { category: "Hardware" } }); // 3
await frappe.db.count("Sample Order", { filters: [["Sample Order Item", "item", "=", "Keyboard"]] });
```

### insert, set_value, delete_doc

`insert(doc)` needs `doc.doctype`. It runs the full insert on the server, with validations and controller methods.

`set_value` loads the document on the server, updates it and calls `doc.save()`. Validations, controller methods and permission checks run. `fieldname` can be an object of values, then leave out `value`. Standard fields such as `name`, `owner` and `docstatus` are ignored. For a child DocType, the server saves the parent document.

```js
const doc = await frappe.db.insert({ doctype: "Sample Item", item_name: "Sample Probe", category: "Hardware", rate: 10 });
const r = await frappe.db.set_value("Sample Item", doc.name, { rate: 15, disabled: 1 });
r.message.rate; // 15
await frappe.db.delete_doc("Sample Item", doc.name);
```

## frappe.perm

`frappe.perm` evaluates permissions in the browser from the meta, `frappe.boot` and `docinfo`. The server checks again on every request.

| Function | Description |
|---|---|
| `has_perm(doctype, permlevel = 0, ptype = "read", doc)` | Return `true` if the user has `ptype` at `permlevel`. With a saved `doc`, applies owner rules, shares and the permissions in `docinfo`. |
| `get_perm(doctype, doc)` | Return a list of permission objects, one per permlevel, for example `perm[0].write`. Without `doc`, the result is cached per DocType once the meta is loaded. |
| `get_rights(doctype)` | Return the standard rights plus the custom permission types of the DocType. |
| `get_role_permissions(meta)` | Return the rights from the DocPerm rows of the roles of the user. |
| `get_match_rules(doctype, ptype)` | Return the user permission and owner rules that limit the records the user sees. |
| `get_field_display_status(df, doc, perm, explain)` | Return `"Write"`, `"Read"` or `"None"` for a field. Applies permlevel, `hidden`, docstatus, `allow_on_submit`, workflow, `read_only` and `set_only_once`. With `explain`, logs each step. |
| `is_visible(df, doc, perm)` | Return `true` unless the display status is `"None"`. `df` can be a fieldname. |
| `get_allowed_docs_for_doctype(user_permissions, doctype)` | Return the names allowed by user permissions for `doctype`. |
| `filter_allowed_docs_for_doctype(user_permissions, doctype, with_default_doc = true)` | Return `{allowed_records, default_doc}`, or only the list if `with_default_doc` is `false`. |
| `rights`, `doctype_perm` | Standard right names and the cache of `get_perm`. |

The globals `READ`, `WRITE`, `CREATE`, `DELETE`, `SUBMIT`, `CANCEL` and `AMEND` hold the lowercase right names, for old code.

## frappe.workflow

`frappe.workflow` reads the Workflow records that desk loaded into `locals` with the meta.

| Function | Description |
|---|---|
| `get_state_fieldname(doctype)` | Return the workflow state field, or `null` if the DocType has no workflow. |
| `get_workflow(doc)` | Return the workflow that applies to the document. With many workflows, the first by priority whose conditions match. |
| `has_workflow(doc)` | Return `true` if a workflow applies to the document. |
| `applies_to(workflow, doc)` | Return `true` if all conditions of the workflow match. |
| `get_state(doc)` | Return the state of the document, or the default state for its docstatus. |
| `get_default_state(doc, docstatus)` | Return the first state with this docstatus. `doc` can be a DocType name. |
| `get_transitions(doc)` | Call `frappe.model.workflow.get_transitions`. Returns a promise of the transitions the user can run. |
| `get_document_state_roles(doc, state)` | Return the roles that can edit in this state. |
| `is_read_only(doctype, name)` | Return `true` if the workflow state does not let the user edit the document. |
| `is_self_approval_enabled(doc)` | Return `allow_self_approval` of the workflow. |
| `get_update_fields(doctype)` | Return the fields that workflow states set. |
| `get_all_transitions(doctype)`, `get_all_transition_actions(doctype)` | Return the transitions or their action names of all workflows of the DocType. |
| `setup(doctype)` | Read the workflows of the DocType from `locals` into the caches `workflows`, `candidates`, `state_fields` and `avoid_status_override`. |

## Pitfalls

- `frappe.model.set_value(doc, fieldname, value)` with a document object and three arguments sets the field to the fieldname string on develop. Pass the DocType and name, or an object: `frappe.model.set_value(doc, { customer_name: "Customer A" })`.
- `frappe.db.get_value` and `frappe.db.set_value` resolve with the response. Read `r.message`.
- `frappe.model.get_value` with a callback returns `undefined`, not a promise. Use `frappe.db.get_value` with `await`.
- `frappe.model.get_list` and `frappe.get_list` read `locals` only. They are not the same as the server `frappe.get_list`. Use `frappe.db.get_list`.
- `frappe.model.get_doc_title`, `round_floats_in`, `is_submittable` and `get_server_module_name` need the meta. Call `frappe.model.with_doctype` first.
- `frappe.model.add_child` and `clear_table` do not render. In a form, call `frm.refresh_field(table_fieldname)`.

## Related

- [Client API](../../developer-guide/desk-ui/client-api.md)
- [Form Scripts](../../developer-guide/desk-ui/form-scripts.md)
- [Form API](form.md)
- [frappe namespace](frappe.md)
- [Table fields](../../developer-guide/fields/table-fields.md)
- [Permissions](../../developer-guide/security/permissions.md)
