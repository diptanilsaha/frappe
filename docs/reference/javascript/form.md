---
title: JavaScript Form API
description: Reference for frappe.ui.form and the frm object in desk JavaScript - form and child table events in trigger order, frm methods and properties, toolbar, dashboard, timeline, sidebar, ScriptManager, grid and quick entry.
version: develop
sources:
  - frappe/public/js/frappe/form/form.js
  - frappe/public/js/frappe/form/script_manager.js
  - frappe/public/js/frappe/form/script_helpers.js
  - frappe/public/js/frappe/form/save.js
  - frappe/public/js/frappe/form/toolbar.js
  - frappe/public/js/frappe/form/dashboard.js
  - frappe/public/js/frappe/form/workflow.js
  - frappe/public/js/frappe/form/quick_entry.js
  - frappe/public/js/frappe/form/grid.js
  - frappe/public/js/frappe/form/grid_row.js
  - frappe/public/js/frappe/form/layout.js
  - frappe/public/js/frappe/form/footer/footer.js
  - frappe/public/js/frappe/form/footer/base_timeline.js
  - frappe/public/js/frappe/form/footer/form_timeline.js
  - frappe/public/js/frappe/form/sidebar/form_sidebar.js
  - frappe/public/js/frappe/form/controls/table_multiselect.js
  - frappe/public/js/frappe/model/create_new.js
  - frappe/public/js/frappe/request.js
  - frappe/public/js/frappe/ui/page.js
  - frappe/handler.py
---

# JavaScript Form API

This page lists the client-side API of desk forms: `frappe.ui.form`, the form events, and the `frm` object (`frappe.ui.form.Form`). Use it to look up an exact name, its arguments and what it does. For how to write a form script and where the code goes, see [Form Scripts](../../developer-guide/desk-ui/form-scripts.md). For controls (`frm.fields_dict.<fieldname>`), see [Controls](../../developer-guide/fields/controls.md).

The names come from `Object.keys(frappe.ui.form)` and the prototypes of the classes in a logged-in desk at `/desk` on develop. The event order was checked in the browser on a `Sample Order` form.

## frappe.ui.form functions

| Function | Description |
|---|---|
| `on(doctype, events)` or `on(doctype, event, handler)` | Register form event handlers for `doctype`. `events` is an object `{event: handler}`. Handlers add up: many calls for the same event all run, in the order they were registered. Use `"*"` as `doctype` for all DocTypes. Also `frappe.ui.form.on_change`. |
| `off(doctype, event, handler)` | Remove all handlers of `event` for `doctype`, and the matching `frm.events` and `frm.cscript` entries. The `handler` argument is ignored. |
| `trigger(doctype, event)` | Run the handlers of `event` on `cur_frm`. Returns nothing. Use `frm.trigger(event)` to get a promise. |
| `set_controller(doctype, ControllerClass)` | Bind a class-based controller to the DocType. Its methods get `(doc, cdt, cdn)`. They run after the `frappe.ui.form.on` handlers of the same event. |
| `make_quick_entry(doctype, after_insert, init_callback, doc, force, skip_insert)` | Open the quick entry dialog, or the full form if quick entry does not apply. Returns a promise of the dialog object. See [Quick entry](#quick-entry). |
| `make_control(opts)` | Make one control outside a form. See [Controls](../../developer-guide/fields/controls.md). |
| `get_open_grid_form()` | Return the `GridRow` whose row form is open, or `undefined`. |
| `close_grid_form()` | Close the open row form and the editable grid row. |
| `save(frm, action, callback, btn)` | Low-level save. Checks mandatory fields, then calls `frappe.desk.form.save.savedocs` (or `frappe.desk.form.save.submit` for `Submit`, `frappe.desk.form.save.cancel` for `cancel`). Does not run `validate` or `before_save`. Use `frm.save()`. |
| `check_mandatory(frm)` | Return `false` and show the missing mandatory fields, or `true`. |
| `update_calling_link(newdoc)` | After a document was made from a Link field, set it in that Link field and go back. |
| `remove_old_form_route()` | Remove the current route from `frappe.route_history`. |
| `get_breadcrumbs(frm)` | Return the breadcrumb items of a form. |
| `get_event_handler_list(doctype, event)` | Return the list of handlers of an event. Makes an empty list if there is none. |
| `handlers`, `controllers` | Registered handlers by DocType and event, and classes set with `set_controller`. |

Related globals:

| Name | Description |
|---|---|
| `frappe.new_doc(doctype, opts, init_callback)` | Open a new document. `opts` is set as `frappe.route_options`, so its keys become field values. Opens the route in `frappe.create_routes[doctype]` if set, else calls `make_quick_entry`. For `File`, opens the file uploader. Returns a promise. |
| `frappe.quick_edit(doctype, name)` | Load a document and open it in the quick entry dialog. |
| `frappe.validated` | Set to `false` in `validate`, `before_save`, `before_submit`, `before_cancel` or `before_discard` to stop the action. |
| `frappe.route_hooks.after_load`, `after_save`, `after_submit` | Set a function `fn(frm)` before you open a form. The form calls it once after load, save or submit, then deletes it. |
| `cur_frm` | The form on screen. Prefer the `frm` argument of handlers. |

## Form events

Handlers of form events get `frm`. A handler that returns a promise delays the next handler and the rest of the action.

### Trigger order

| Action | Events, in order |
|---|---|
| Open a document for the first time in this session (new or saved) | `setup` (once per DocType), `before_load`, `onload`, `refresh`, `onload_post_render` |
| Open a document again | `refresh` |
| Change a field value | `<fieldname>` |
| Save | `validate`, `before_save`, server save, `after_save`, `refresh` |
| Submit | `before_submit`, `validate`, `before_save`, server submit, `after_save`, `on_submit`, `refresh` |
| Cancel | `before_cancel`, server cancel, `after_cancel`, `refresh` |
| Delete | `after_delete` |
| Workflow action | `before_workflow_action`, server action, `after_workflow_action` |

`timeline_refresh` also runs each time the timeline renders, for example after save.

### All form events

| Event | When it runs |
|---|---|
| `setup` | Once, when the form object of the DocType is made. Runs at once, not in series. |
| `before_load` | The first time a document opens in this session, before `onload`. |
| `onload` | The first time a document opens in this session. |
| `refresh` | Each time the form renders: after load, save, submit, cancel and reload. |
| `onload_post_render` | After the first `refresh` of a document. |
| `<fieldname>` | After the value of the field changes through `frm.set_value`, `frappe.model.set_value` or the user. |
| `<button fieldname>` | When the user clicks a `Button` field. |
| `validate` | Before save and before submit. Stop with `frappe.throw()` or `frappe.validated = false`. Not run for `Update` (save after submit). |
| `before_save` | After `validate`, before the request. |
| `after_save` | After the server saved or submitted the document. |
| `before_submit` | After the user confirms submit. Stop with `frappe.validated = false`. |
| `on_submit` | After the server submitted the document. |
| `before_cancel` | After the user confirms cancel. Stop with `frappe.validated = false`. |
| `after_cancel` | After the server cancelled the document. |
| `before_discard`, `after_discard` | Before and after the user discards a draft. |
| `after_delete` | After the document was deleted, before the browser goes back. |
| `before_workflow_action`, `after_workflow_action` | Before and after a workflow action. `frm.selected_workflow_action` holds the action. |
| `timeline_refresh` | After the timeline renders. |
| `dashboard_update` | After the connections dashboard loads its counts. |
| `on_tab_change` | After the active form tab changes. |
| `on_hide` | When the form page is hidden. |
| `<table fieldname>_on_form_rendered` | On the parent DocType, when a row form of that table opens. Old style. Prefer `form_render`. |

A `depends_on`, `mandatory_depends_on` or `read_only_depends_on` value of the form `fn:<event>` runs the form event `<event>` and uses its return value.

### Child table events

Register child table events on the child DocType. Handlers get `(frm, cdt, cdn)`. Read the row with `locals[cdt][cdn]`.

| Event | When it runs |
|---|---|
| `<child fieldname>` | After a field of a row changes. |
| `<table fieldname>_add` | After the user adds a row, or after `grid.add_new_row()`. Not after `frm.add_child()`. |
| `before_<table fieldname>_remove` | Before a row is removed. |
| `<table fieldname>_remove` | After a row is removed. |
| `<table fieldname>_delete` | After the user deletes selected rows or all rows. `cdn` is not set. |
| `<table fieldname>_move` | After the user drags a row to a new position. |
| `form_render` | When the row form of a row opens. |

A `Table MultiSelect` field runs `<fieldname>_add`, `before_<fieldname>_remove` and `<fieldname>_remove` on the child DocType when the user adds or removes one value.

```js
// sample_app/sample_module/doctype/sample_order/sample_order.js
frappe.ui.form.on("Sample Order Item", {
	qty(frm, cdt, cdn) {
		const row = locals[cdt][cdn];
		frappe.model.set_value(cdt, cdn, "amount", flt(row.qty) * flt(row.rate));
	},
	items_remove(frm) {
		frm.set_value("total", frm.get_sum("items", "amount"));
	},
});
```

### Old-style handlers

`frm.cscript` holds old-style handlers. A method `frm.cscript.<event>` or `frm.cscript.custom_<event>` runs with `(doc, cdt, cdn)` after the new-style handlers. `frm.events.<event>` is the last handler registered for the event while the form was on screen.

## frm properties

| Property | Description |
|---|---|
| `doc` | The document, the same object as `locals[doctype][docname]`. |
| `doctype`, `docname` | DocType and name of the open document. |
| `meta` | The DocType meta. |
| `fields_dict` | Controls by fieldname. `fields_dict.<table>.grid` is the grid. |
| `fields` | List of controls. |
| `perm` | Permissions of the document by permlevel, for example `frm.perm[0].write`. |
| `page` | The `frappe.ui.Page` of the form. |
| `layout` | The `frappe.ui.form.Layout`. |
| `toolbar`, `dashboard`, `sidebar`, `timeline`, `footer` | Form parts. See the sections below. |
| `script_manager` | The `frappe.ui.form.ScriptManager`. |
| `states` | The workflow helper (`frappe.ui.form.States`). |
| `attachments`, `assign_to`, `shared`, `tags`, `comment_box` | Sidebar and footer parts. `comment_box` is the comment control. |
| `custom_buttons` | Buttons added with `add_custom_button`, by label. |
| `grids` | Grids of the form. |
| `wrapper`, `$wrapper`, `body` | DOM elements of the form. |
| `opendocs` | Names of documents opened in this form object. |
| `read_only` | `true` if a workflow makes the document read-only. |
| `save_disabled` | `true` after `disable_save()`. |
| `state_fieldname` | Workflow state field, or `null`. |
| `selected_doc` | The document or row of the event that runs now. |
| `selected_workflow_action` | The workflow action that runs now. |
| `doctype_layout` | The active DocType Layout, or `null`. |
| `undo_manager` | Undo and redo history of field changes. |
| `refresh_if_stale_for` | Seconds after which an unchanged document reloads when the user opens it again. Default `120`. |
| `fetch_dict` | Fetch rules added with `add_fetch`. |

Properties that a form script can set:

| Property | Description |
|---|---|
| `custom_make_buttons` | `{doctype: button_label}`. The `+` button of that DocType in the connections dashboard clicks this custom button. |
| `make_methods` | `{doctype: fn(frm)}`. The `+` button of that DocType in the connections dashboard calls `fn`. |
| `can_make_methods` | `{doctype: fn(frm)}`. Return `true` to show the `+` button of that DocType. |
| `ignore_doctypes_on_cancel_all` | DocTypes that cancel does not cancel with the document. |
| `show_print_first` | Open the print view when a submitted document opens. |

## frm methods

### Values

| Method | Description |
|---|---|
| `set_value(fieldname, value, if_missing, skip_dirty_trigger = false)` or `set_value({fieldname: value, ...})` | Set parent fields and run their events. Returns a promise. With an array for a Table field, replaces all rows and renders the table. Standard keys such as `name`, `idx` and `parent` in the array are ignored. With `if_missing`, sets only empty fields. Throws if the field is not in the form. In the object form, unknown fields are skipped. |
| `get_field(fieldname)` | Return the control. Same as `fields_dict[fieldname]`. |
| `get_docfield(fieldname, child_fieldname)` | Return the DocField copy of this document. With two arguments, return the field of the child DocType of the Table field `fieldname`. |
| `get_formatted(fieldname)` | Return the formatted value as HTML. |
| `get_title()` | Return the title of the document. |
| `get_doc()` | Return `locals[doctype][docname]`. |
| `is_new()` | Return `doc.__islocal`: truthy for an unsaved document. |
| `is_dirty()` | Return `true` if the document has unsaved changes. |
| `dirty()` | Mark the document as changed. Adds the leave-page warning unless developer mode is on. |
| `add_fetch(link_field, source_field, target_field, target_doctype)` | When `link_field` changes, copy `source_field` of the linked document into `target_field`. Prefer `fetch_from` in the DocType. |
| `set_currency_labels(fields_list, currency, parentfield)` | Add `(<currency>)` to the labels of these fields. With `parentfield`, change the columns of that table. With no `currency`, reset the labels. |
| `reset_currency_labels(fields, parentfield)` | Restore the labels changed by `set_currency_labels`. |
| `set_fields_as_options(fieldname, reference_doctype, filter_function, default_options = [], table_fieldname)` | Load the fields of `reference_doctype` and set them as the options of a Select field. Returns a promise of the options. |

### Field properties

These change the DocField copy of the open document only, then render the field again.

| Method | Description |
|---|---|
| `set_df_property(fieldname, property, value)` | Set a field property, for example `read_only`, `hidden`, `reqd`, `options`, `label`, `description`. |
| `set_df_property(table_fieldname, property, value, docname, child_fieldname, row_name)` | Set a property of a child field. With `row_name`, for that row only. Pass `frm.docname` as `docname`. |
| `toggle_display(fieldnames, show)` | Set `hidden`. `fieldnames` is a fieldname, a list, or `"*"`. |
| `toggle_reqd(fieldnames, mandatory)` | Set `reqd`. |
| `toggle_enable(fieldnames, enable)` | Set `read_only`. |
| `set_query(fieldname, fn)` or `set_query(child_fieldname, table_fieldname, fn)` | Set the `get_query` of a Link field. `fn(doc, cdt, cdn)` returns `{filters}` or `{query, filters}`. |
| `set_indicator_formatter(fieldname, get_color, get_text)` | Show a Link column as a colored indicator. `get_color(doc)` returns a color class. |
| `field_map(fieldnames, fn)` | Call `fn(df)` for each DocField and render the fields that changed. |
| `refresh_field(fieldname)` | Render one field again. |
| `refresh_fields()` | Render all fields in the next animation frame. |
| `set_read_only()` | Remove write rights from `frm.perm` and render the fields. |
| `disable_form()` | Make all fields read-only and hide the save button. |

```js
frm.set_df_property("items", "read_only", 1, frm.docname, "qty", row.name);
frm.set_query("item", "items", () => ({ filters: { disabled: 0 } }));
```

### Child tables

| Method | Description |
|---|---|
| `add_child(table_fieldname, values)` | Add a row and return it. `idx` and `name` in `values` are ignored. Does not render and does not run `<table>_add`. Call `refresh_field(table_fieldname)`. |
| `clear_table(table_fieldname)` | Remove all rows. Does not render. |
| `get_sum(table_fieldname, fieldname)` | Sum a column with `+`. An empty value gives `NaN`. |
| `update_in_all_rows(table_fieldname, fieldname, value)` | Set `value` in each row where the field is empty. Runs the child field event. |
| `get_selected()` | Return the selected rows of all grids as `{table_fieldname: [row names]}`. |
| `open_grid_row()` | Return the `GridRow` whose row form is open. |

### Buttons and messages

| Method | Description |
|---|---|
| `add_custom_button(label, fn, group, type = "default")` | Add a button to the form toolbar. With `group`, add it to a dropdown with that label. Returns the button. Call it in `refresh`: the toolbar is cleared on each render. |
| `remove_custom_button(label, group)` | Remove one custom button. |
| `clear_custom_buttons()` | Remove all custom buttons and sidebar user actions. |
| `change_custom_button_type(label, group, type)` | Change the button style, for example `"primary"`. |
| `set_intro(text, color)` | Show a message at the top of the form. Call with no text to clear it. |
| `set_footnote(text)` | Show a note at the end of the form. |
| `add_web_link(path, label)` | Add a link to the sidebar. |
| `show_success_action()` | Show the Success Action message of the DocType. |
| `scroll_to_field(fieldname, focus = true)` | Open the tab, open the section, scroll to the field, focus it and highlight it. Returns `false` if the field is not found. |
| `set_active_tab(tab)`, `get_active_tab()` | Set or get the active `frappe.ui.form.Tab`. Use `frm.get_field(tab_fieldname).tab` to get a tab. |

### Document actions

| Method | Description |
|---|---|
| `save(action = "Save", callback, btn, on_error)` | Run `validate` and `before_save`, then save. `action` is `"Save"`, `"Submit"`, `"Update"` or `"Cancel"`. Returns a promise. Shows "No changes in document" if nothing changed. |
| `savesubmit(btn, callback, on_error)` | Ask, then run `before_submit` and submit. Returns a promise that resolves after `on_submit`. |
| `savecancel(btn, callback, on_error)` | Find submitted linked documents. If there are some, offer to cancel them all. Else ask, run `before_cancel` and cancel. |
| `savetrash()` | Find linked documents. Offer to delete them with the document, or ask, then delete. |
| `discard(btn, callback, on_error)` | Ask, then discard the draft with `frappe.desk.form.save.discard`. |
| `amend_doc()` | Make an amended copy of a cancelled document. Needs an `amended_from` field. |
| `copy_doc(onload, from_amend)` | Open a copy of the document. `onload(newdoc)` runs before the form opens. |
| `reload_doc()` | Remove the document from `locals`, load it again, then render. Returns a promise. |
| `save_or_update()` | Save a draft, or update a submitted document that has changes. |
| `enable_save()`, `disable_save(set_dirty = false)` | Show or hide the primary save button. Call `disable_save` in `refresh`. With `set_dirty`, changes still mark the form dirty. |
| `print_doc()` | Open the print view. |
| `email_doc(message, opts)` | Open the email composer. Returns the `frappe.views.CommunicationComposer`. |
| `rename_doc()` | Open the rename dialog. |
| `share_doc()` | Open the share dialog. |
| `navigate_records(prev)` | Open the next record (`prev` = `0`) or the previous record (`prev` = `1`) with the filters of the list view. |
| `execute_action(action)` | Run a DocType Action by label or row. A Server Action calls the method with `doc`, syncs a returned document and renders. |
| `make_new(doctype, fieldname)` | Make a new linked document from the connections dashboard. Uses `make_methods` or `custom_make_buttons` if set. |
| `can_create(doctype)` | Return `true` if the dashboard can show `+` for this DocType. |

### Permissions and data

| Method | Description |
|---|---|
| `has_perm(ptype)` | Return `true` if the user has `ptype` (`"read"`, `"write"`, `"submit"`...) at permlevel 0 on this document. |
| `get_perm(permlevel, access_type)` | Return `frm.perm[permlevel][access_type]`, or `null`. |
| `get_docinfo()` | Return `frappe.model.docinfo[doctype][docname]`. |
| `get_files()` | Return the attachments, sorted by file name. |
| `get_involved_users()` | Return users linked to the document, its comments, versions, assignments and emails. Leaves out Administrator and the current user. |
| `trigger(event, doctype, docname)` | Run the handlers of an event. Returns a promise. |
| `refresh(docname)` | Render the form. With `docname`, switch to that document first. |

### Call the server

`frm.call(method, args, callback)` or `frm.call(opts)` calls the server and returns the promise of `frappe.call`.

| Form | What it calls |
|---|---|
| `frm.call("method_name", args)` | The whitelisted method `method_name` of the controller class. The current document from `locals` is sent as `docs`. The server runs `frappe.handler.run_doc_method`, which checks read permission. |
| `frm.call({method, doc: frm.doc, args})` | Same as above. |
| `frm.call({method, args})` with no `doc` | A module function. A `method` without a dot is prefixed with the Python module of the DocType. If `r.message` is an object, its keys are set on the form with `set_value`. With `child`, they are set on that row. |

When you call a controller method, the response includes the document. Desk syncs it into `locals` and renders the fields again. The changes are not saved.

```python
# sample_app/sample_module/doctype/sample_order/sample_order.py
class SampleOrder(Document):
	@frappe.whitelist()
	def get_open_count(self):
		return frappe.db.count("Sample Order", {"customer_name": self.customer_name, "docstatus": 0})
```

```js
const r = await frm.call("get_open_count");
r.message; // a number
```

### Internal methods

Desk calls these. Do not call them from a form script: `setup`, `setup_meta`, `setup_std_layout`, `watch_model_updates`, `setup_notify_on_rename`, `setup_file_drop`, `setup_image_autocompletions_in_markdown`, `setup_doctype_actions`, `setup_docinfo_change_listener`, `add_form_keyboard_shortcuts`, `switch_doc`, `check_reload`, `trigger_onload`, `initialize_new_doc`, `render_form`, `render_fields`, `cleanup_refresh`, `refresh_header`, `apply_layout_defaults`, `onload_post_render`, `configure_breadcrumb_width`, `focus_on_first_input`, `run_after_load_hook`, `show_upload_first_review`, `validate_and_save`, `validate_form_action`, `handle_save_fail`, `trigger_link_fields`, `show_conflict_message`, `show_submit_message`, `show_web_link`, `show_report_bug_link`, `show_submission_queue_banner`, `show_workflow_read_only_banner`, `mark_mask_fields_readonly`, `fetch_permissions`, `has_read_permission`, `check_doctype_conflict`, `rename_notify`, `get_active_tab_hash`, `is_form_builder`, `scroll_to_element`, `set_link_field`, and the methods that start with `_`.

## frm.page

`frm.page` is a `frappe.ui.Page`. All its methods are in [UI Components](ui-components.md). These are the ones form scripts use most:

| Method | Description |
|---|---|
| `add_inner_button(label, action, group, type = "default", align_right = false)` | Add a toolbar button. `frm.add_custom_button` calls this. |
| `add_menu_item(label, click, standard, shortcut, show_parent, group)` | Add an item to the `...` menu. |
| `add_action_item(label, click, standard, icon)` | Add an item to the Actions dropdown. |
| `set_primary_action(label, click, icon, working_label)`, `set_secondary_action(...)` | Set the main or second button. The toolbar sets these on each render. |
| `set_indicator(label, color)`, `clear_indicator()` | Set or clear the status indicator next to the title. |
| `clear_actions_menu()` | Remove all items of the Actions dropdown. |

## frm.toolbar

`frappe.ui.form.Toolbar` builds the title, the indicator, the primary button and the `...` menu on each render.

| Method | Description |
|---|---|
| `refresh()` | Build the menu, title, indicator and primary action again. |
| `set_primary_action(dirty)` | Set the primary button for the current state: Save, Submit, Update, Cancel or Amend. |
| `get_action_status()` | Return `"Submit"`, `"Save"`, `"Update"`, `"Cancel"`, `"Amend"`, `"Edit"` or `null`. |
| `set_indicator()` | Set the indicator from `frappe.get_indicator(frm.doc)`. |
| `set_title()` | Set the form title. |
| `show_linked_with()` | Open the Links dialog. |
| `show_jump_to_field_dialog()` | Open the Jump to Field dialog. |
| `can_save()`, `can_submit()`, `can_update()`, `can_cancel()`, `can_amend()`, `can_repeat()`, `can_rename()` | Return `true` if the action applies now. |
| `has_workflow()`, `get_docstatus()` | Workflow check and `cint(frm.doc.docstatus)`. |

The other `add_*` and `make_*` methods build the menu.

Do not call `get_dropdown_menu(label)`. It calls `frm.page.add_dropdown`, which does not exist on develop, so it throws.

The form adds these keyboard shortcuts: `shift+ctrl+>` next record, `shift+ctrl+<` previous record, `ctrl+p` print, `shift+ctrl+z` redo.

## frm.dashboard

`frappe.ui.form.Dashboard` is the area at the top of the form. It holds messages, progress bars, indicators, the connections links, a heatmap and a chart. The connections data comes from the `<doctype>_dashboard.py` file of the DocType (`frm.meta.__dashboard`).

| Method | Description |
|---|---|
| `set_headline(html, color, permanent = false)` | Show a message at the top of the form. |
| `set_headline_alert(text, color, permanent = false)` | Same, wrapped in a `div`. With no text, clear it. `frm.set_intro` uses this. |
| `clear_headline()` | Clear the message. |
| `add_comment(text, alert_class, permanent)` | Show a message. Clears it after 10 seconds unless `permanent`. |
| `clear_comment()` | Clear the message. |
| `add_indicator(label, color)` | Add a colored indicator in the stats area. Returns the element. |
| `add_progress(title, percent, message)` | Add a progress bar. `percent` is a number or a list of `{title, width, progress_class}`. |
| `show_progress(title, percent, message)` | Add or update the progress bar with this title. |
| `hide_progress(title)` | Remove one progress bar, or all with no title. |
| `add_section(body_html, label = null, css_class = "custom", hidden = false)` | Add a collapsible section. Returns its body. |
| `add_transactions(opts)` | Add connection groups `{label, items}` to the connections area. |
| `render_graph(args)` | Draw a chart with `frappe.ui.chart` arguments. |
| `update_heatmap(data)` | Update the heatmap data points. |
| `show()`, `hide()` | Show or hide the dashboard. |
| `refresh()` | Render the dashboard again. Runs on each form render. |

```js
frappe.ui.form.on("Sample Order", {
	refresh(frm) {
		frm.dashboard.add_indicator(__("Total: {0}", [format_currency(frm.doc.total)]), "blue");
	},
});
```

## frm.timeline

`frm.timeline` is a `FormTimeline`. It exists when the DocType does not hide the toolbar and Desk Settings shows the timeline.

| Method | Description |
|---|---|
| `refresh()` | Render the timeline again, then run `timeline_refresh`. |
| `add_action_button(label, action, icon = null, btn_class = null)` | Add a button above the timeline. Returns the button. |
| `add_timeline_item(item, append_at_the_end = false)` | Add one item. `item` has `content` and `creation`, and can have `timeline_badge`, `icon`, `icon_size`, `hide_timestamp` and `is_card`. |
| `add_timeline_items(items, append_at_the_end = false)` | Add many items. |
| `add_timeline_items_based_on_creation(items)` | Insert items at the place of their `creation` time. |
| `compose_mail(communication_doc = null, reply_all = false)` | Open the email composer. |

Items added in `timeline_refresh` stay until the next render. To add items from the server, use the `additional_timeline_content` hook. See [Hooks](../configuration/hooks.md).

## frm.sidebar

`frappe.ui.form.Sidebar` exists when Desk Settings shows the form sidebar.

| Method | Description |
|---|---|
| `add_user_action(label, click)` | Add a link to the sidebar. Returns the link. |
| `clear_user_actions()` | Remove these links. |
| `refresh()` | Render the sidebar again. |
| `reload_docinfo(callback)` | Load `docinfo` again with `frappe.desk.form.load.get_docinfo`, then call `callback(docinfo)`. |

## frappe.ui.form.ScriptManager

`frm.script_manager` loads the form scripts of the DocType and runs events.

| Method | Description |
|---|---|
| `trigger(event_name, doctype, name)` | Run all handlers of the event, one after the other, and return a promise. `doctype` and `name` default to the form document. Pass the child DocType and row name for a child event. Sets `frm.selected_doc`. |
| `get_handlers(event_name, doctype)` | Return `{new_style: [...], old_style: [...]}`. New style: handlers of the DocType, then of `"*"`. Old style: names of `frm.cscript` methods. |
| `has_handlers(event_name, doctype)` | Return a truthy value if the event has a handler. |
| `has_handler(event_name)` | Return the new-style handler list of the form DocType for the event, if any. |
| `copy_from_first_row(parentfield, current_row, fieldnames)` | Copy fields of the first row into `current_row` with `frappe.model.set_value`. |
| `setup()` | Run the DocType JS, the layout client script and Client Scripts, bind the controller, set up `fetch_from`, then run `setup`. Desk calls it. |
| `bind_controller()`, `make(ControllerClass)` | Bind the class set with `set_controller`. |
| `log_error(caller, e)` | Show an alert and log the error. |

## Grid and GridRow

The grid of a Table field is `frm.fields_dict.<table_fieldname>.grid` or `frm.get_field(table_fieldname).grid`. The classes are not on `frappe.ui.form`. Their constructors are `Grid` and `GridRow`.

### Grid

| Method or property | Description |
|---|---|
| `add_new_row(idx, callback, show, copy_doc, go_to_last_page = false, go_to_first_page = false)` | Add a row, run `<table>_add` and render. With `idx`, insert at that position. With `show`, open the row form. `copy_doc` copies its values. Returns the row, or nothing if rows cannot be added. |
| `get_row(key)` | Return the `GridRow` by index (negative counts from the end) or by row name. |
| `get_grid_row(key)` | Same as `get_row`. |
| `get_selected()` | Return the names of the selected rows. |
| `get_selected_children()` | Return the selected rows. |
| `select_row(name)` | Select a row. |
| `get_data()` | Return the rows. |
| `get_docfield(fieldname)` | Return the child DocField copy of this form. |
| `get_field(fieldname)` | Return an object where you set `get_query` for a child Link field. `frm.set_query` uses it. |
| `update_docfield_property(fieldname, property, value)` | Set a child DocField property for all rows and new rows, then render. Throws if the field is not found. |
| `toggle_reqd(fieldname, reqd)`, `toggle_enable(fieldname, enable)`, `toggle_display(fieldname, show)` | Set `reqd`, `read_only` or `hidden` through `update_docfield_property`. |
| `set_column_disp(fieldname, show)` | Show or hide a column. Changes the shared DocField. `fieldname` can be a list. |
| `set_column_disp_in_list_view(fieldname, show)` | Show or hide a column in this grid only. |
| `toggle_checkboxes(enable)` | Enable or disable the row checkboxes. |
| `add_custom_button(label, click, position = "bottom")` | Add a button below the grid, or above it with `"top"`. Returns the button. |
| `clear_custom_buttons()` | Hide the custom buttons. |
| `set_multiple_add(link_fieldname, qty_fieldname)` | Show the Add Multiple button. It opens a search dialog for the Link field. |
| `only_sortable(status)` | Allow drag to reorder, but no other edits. |
| `is_editable()`, `is_sortable()`, `allow_on_grid_editing()` | Return the state of the grid. |
| `delete_rows()`, `delete_all_rows()`, `duplicate_rows()` | Delete the selected rows, ask and delete all rows, or copy the selected rows. Run the remove and delete events. |
| `refresh()`, `refresh_row(docname)`, `reset_grid()` | Render the grid, one row, or the grid with its columns again. |
| `set_focus_on_row(idx)` | Focus the first input of a row. |
| `grid_rows`, `grid_rows_by_docname`, `data`, `docfields`, `doctype`, `df`, `frm` | Rows, rows by name, row data, child DocFields, child DocType, Table DocField and form. |
| `cannot_add_rows` | Set `true` to block new rows. `df.cannot_add_rows` and `df.cannot_delete_rows` also work. |

### GridRow

| Method or property | Description |
|---|---|
| `doc` | The row data. |
| `toggle_view(show, callback)` | Open or close the row form. |
| `get_field(fieldname)` | Return the control of a field in the editable row or the open row form. Throws if neither exists. |
| `set_field_property(fieldname, property, value)` | Set a property on the controls of this row and render them. |
| `toggle_reqd(fieldname, reqd)`, `toggle_display(fieldname, show)`, `toggle_editable(fieldname, editable)` | Set `reqd`, `hidden` or `read_only` on the controls of this row. |
| `refresh()`, `refresh_field(fieldname, txt)` | Render the row or one cell. |
| `select(checked)` | Set `doc.__checked`. |
| `remove()` | Run `before_<table>_remove`, remove the row, run `<table>_remove`. |
| `insert(show, below, duplicate)` | Add a row above or below, or a copy. |
| `move()` | Ask for a row number and move the row. |
| `open_prev()`, `open_next()`, `has_prev()`, `has_next()` | Move between row forms. |

```js
frappe.ui.form.on("Sample Order", {
	refresh(frm) {
		const grid = frm.get_field("items").grid;
		grid.toggle_enable("rate", false);
		grid.add_custom_button(__("Set Qty to 1"), () => {
			for (const row of frm.doc.items || []) {
				frappe.model.set_value(row.doctype, row.name, "qty", 1);
			}
		});
	},
});
```

See [Table fields](../../developer-guide/fields/table-fields.md) for which calls render and which run events.

## Quick entry

`frappe.ui.form.make_quick_entry(doctype, after_insert, init_callback, doc, force, skip_insert)` opens a `frappe.ui.form.QuickEntryForm`, a `frappe.ui.Dialog` with the mandatory fields and the fields with `allow_in_quick_entry`.

| Argument | Description |
|---|---|
| `doctype` | DocType of the new document. |
| `after_insert(doc)` | Called after the insert. Without it, the new document opens unless you are in a list view. |
| `init_callback(dialog)` | Called after the dialog renders. If the full form opens, called with the document. |
| `doc` | A local document to start from. Default: a new document from `frappe.model.get_new_doc`. |
| `force` | Show the dialog even if quick entry does not apply. |
| `skip_insert` | Do not insert. Call `after_insert` with the values. |

The dialog shows only if the DocType has Quick Entry set, has no mandatory Table field, and has at least one field to show. Else the full form opens. The dialog saves with `frappe.client.save`. For a submittable DocType without a workflow, the primary button then changes to Submit.

To change the dialog of one DocType, define `frappe.ui.form.<DocType without spaces>QuickEntryForm` as a subclass of `frappe.ui.form.QuickEntryForm`. Frappe has `AddressQuickEntryForm` and `ContactQuickEntryForm`.

| Method | Description |
|---|---|
| `set_df_property(fieldname, prop, value)` | Set a field property in the dialog. |
| `toggle_display(fieldnames, show)`, `toggle_enable(fieldnames, enable)`, `toggle_reqd(fieldnames, mandatory)` | Change fields of the dialog. |
| `set_intro(text, color)` | Show a message in the dialog. |
| `insert(values)` | Save the document. Returns a promise of the document. |
| `open_doc(set_hooks)` | Close the dialog and open the full form with the values. |
| `update_doc(data)` | Copy dialog values into `doc`. |
| `render_dialog()`, `register_primary_action()`, `set_defaults()`, `is_quick_entry()` | Override these in a subclass to change the dialog. |

## Pitfalls

- `frappe.ui.form.off` removes all handlers of the event, not only the one you pass.
- `frm.add_child`, `frm.clear_table` and direct changes to `frm.doc` do not render. Call `frm.refresh_field()`.
- `frm.set_value` is asynchronous. Use `await` before you save or read values that other handlers set.
- `frm.get_sum` gives `NaN` if a row has no value in the column. Sum with `flt()` if values can be empty.
- `frm.set_value` sets parent fields only. Use `frappe.model.set_value(cdt, cdn, ...)` for a row.
- `frm.is_new()` returns `undefined` for a saved document, not `false`.
- Set field properties and buttons in `refresh`. `setup` runs once per DocType, and the toolbar is cleared on each render.
- A form script is not a security check. Repeat each rule in the controller.

## Related

- [Form Scripts](../../developer-guide/desk-ui/form-scripts.md)
- [Controls](../../developer-guide/fields/controls.md)
- [Table fields](../../developer-guide/fields/table-fields.md)
- [Model and DB API](model-and-db.md)
- [UI Components](ui-components.md)
- [frappe namespace](frappe.md)
- [Controllers](../../developer-guide/data-model/controllers.md)
