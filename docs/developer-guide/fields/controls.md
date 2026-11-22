---
title: Controls
description: Use the desk control JavaScript API - control classes, make_control, control methods, FieldGroup and Dialog fields, dialog-only controls, and custom controls from an app.
version: develop
sources:
  - frappe/public/js/frappe/form/controls/control.js
  - frappe/public/js/frappe/form/controls/base_control.js
  - frappe/public/js/frappe/form/controls/base_input.js
  - frappe/public/js/frappe/form/controls/data.js
  - frappe/public/js/frappe/form/controls/link.js
  - frappe/public/js/frappe/form/controls/link_combobox.js
  - frappe/public/js/frappe/form/controls/autocomplete.js
  - frappe/public/js/frappe/form/controls/button.js
  - frappe/public/js/frappe/form/controls/date.js
  - frappe/public/js/frappe/form/controls/datetime.js
  - frappe/public/js/frappe/form/controls/date_range.js
  - frappe/public/js/frappe/form/controls/code.js
  - frappe/public/js/frappe/form/controls/text_editor.js
  - frappe/public/js/frappe/form/controls/dynamic_link.js
  - frappe/public/js/frappe/form/controls/multicheck.js
  - frappe/public/js/frappe/form/controls/multiselect.js
  - frappe/public/js/frappe/form/controls/multiselect_pills.js
  - frappe/public/js/frappe/form/controls/multiselect_list.js
  - frappe/public/js/frappe/form/controls/switch.js
  - frappe/public/js/frappe/form/controls/comment.js
  - frappe/public/js/frappe/form/controls/table.js
  - frappe/public/js/frappe/form/grid.js
  - frappe/public/js/frappe/form/layout.js
  - frappe/public/js/frappe/form/form.js
  - frappe/public/js/frappe/model/meta.js
  - frappe/public/js/frappe/ui/field_group.js
  - frappe/public/js/frappe/ui/dialog.js
  - frappe/core/doctype/system_settings/system_settings.json
---

# Controls

A control is the JavaScript object that renders one field in the desk. Forms, grids, dialogs, field groups, filters and web forms all build their fields as controls. Use this page when you build a dialog or a desk page, read or change a field from a form script, or add a control from an app. For the `frm` object and form events, refer to [Form Scripts](../desk-ui/form-scripts.md). For `frappe.ui.Dialog` options, refer to [Client API](../desk-ui/client-api.md#dialogs-frappeuidialog).

## Class hierarchy

Each fieldtype has a class `frappe.ui.form.Control<Fieldtype>`, with the spaces removed. The classes are in `frappe/public/js/frappe/form/controls/`. `control.js` imports all of them.

| Class | Extends | Adds |
|---|---|---|
| `Control` | - | Wrapper, display status (`get_status`), `set_value`, `get_value`, change event. Used directly by layout-only controls. |
| `ControlInput` | `Control` | Label, description, input area, read-only display area, mandatory and invalid styles. |
| `ControlData` | `ControlInput` | The `<input>`, change binding, `maxlength`, `Email`/`Phone`/`Name`/`URL`/`IBAN`/`Barcode` options. |
| `ControlInt` > `ControlFloat` > `ControlCurrency`, `ControlPercent`, `ControlRating` | `ControlData` | Number parsing and formatting. |
| `ControlDate` > `ControlDatetime`, `ControlTime` | `ControlData` | Date picker. |
| `ControlLink` > `ControlDynamicLink`, `ControlTableMultiSelect` | `ControlData` | Search dropdown, link validation, `fetch_from`. |
| `ControlText` > `ControlCode` > `ControlTextEditor`, `ControlMarkdownEditor` | `ControlData` | Text areas and editors. |
| `ControlAutocomplete` > `ControlMultiSelect`, `ControlMultiSelectPills` | `ControlData` | Dropdown from a list of options. |
| `ControlCheck` > `ControlSwitch` | `ControlData` | Checkbox, value `0` or `1`. |
| `ControlTable`, `ControlHTML`, `ControlImage`, `ControlMultiCheck`, `ControlVueComponent` | `Control` | No standard input. |

Aliases: `ControlReadOnly` is `ControlData`, `ControlLongText` is `ControlText`, `ControlLongInt` is `ControlInt`.

### Combobox variants

System Settings has the check box **Enable Combobox Fields (Beta)** (`enable_combobox_link_field`, default off). When it is on, `make_control` adds the suffix `Combobox` for these fieldtypes: `Link`, `Dynamic Link`, `Autocomplete`, `Table MultiSelect`, `MultiSelectPills` and `MultiSelect`. For example, a Link field then uses `ControlLinkCombobox`. Check the setting with `frappe.ui.form.is_combobox_link_enabled()`. The combobox controls accept the same `get_query` and `set_query` hooks. They also accept `map_options(rows, { query, start })` on the control or on the `df` to change, remove or group the rows before they show.

## Make a control

`frappe.ui.form.make_control(opts)` finds the class from `opts.df.fieldtype` and returns a new instance. It returns `undefined` and logs `Invalid Control Name` if no class exists.

```js
const control = frappe.ui.form.make_control({
	df: {
		fieldtype: "Link",
		fieldname: "item",
		label: __("Item"),
		options: "Sample Item",
		change() {
			console.log(control.get_value());
		},
	},
	parent: page.main.find(".sample-filters"),
	render_input: true,
});
control.set_value("Keyboard");
```

| Option | Description |
|---|---|
| `df` | The field definition. Same keys as a DocField, plus the keys in [Keys only for dialogs and field groups](#keys-only-for-dialogs-and-field-groups). |
| `parent` | Element or jQuery object. The control appends its wrapper to it. |
| `render_input` | If true, the constructor calls `refresh()` and builds the input. If false, the input is built on the first `refresh()`. |
| `only_input` | Render only the input, without label and description. |
| `frm`, `doctype`, `docname`, `doc` | Bind the control to a document. A form sets them. Without them the control is standalone. |

A standalone control (no `doctype` and no `docname`) has the status `Write` unless `df.hidden` or `df.read_only` is set. `set_value` then updates the input and `doc[fieldname]` if a `doc` is given. In a form, `set_value` calls `frappe.model.set_value` and runs the form events.

## Control methods and properties

| Name | Description |
|---|---|
| `get_value()` | The parsed value of the input when the status is `Write`. Otherwise the last set value. For example `ControlInt` returns a number. |
| `set_value(value, force)` | Validate, store and show the value, then run `df.change` or `df.onchange`. Returns a promise. Skips the change if the value is the same, unless `force` is true. |
| `set_input(value)` | Show a value in the input and the display area only. It does not change the document and does not run change events. |
| `get_input_value()` | The raw value of the input, before `parse`. |
| `parse(value)`, `validate(value)`, `format_for_input(value)` | Hooks that subclasses override. `validate` can return a promise, for example the link validation. |
| `refresh()` | Compute the status again and render the label, description, value and read-only state. |
| `refresh_input()` | Render the input part only. `refresh()` calls it. |
| `get_status(explain)` | `"Write"`, `"Read"` or `"None"`. Pass `true` to log the reason to the console. |
| `toggle(show)` | Set `df.hidden` and refresh. |
| `set_label(label)` | Change the label. |
| `set_description(text)` | Change the description under the field. |
| `set_new_description(html)`, `set_empty_description()` | Show other text in the help box, or clear it, without changing `df.description`. |
| `toggle_label(show)`, `toggle_description(show)` | Show or hide the label or the description. |
| `set_focus()` | Focus the input. |
| `df` | The field definition that the control uses. |
| `$wrapper` / `wrapper` | The outer `.frappe-control` element. It has `data-fieldname` and `data-fieldtype`. |
| `$input`, `input` | The input element, after the input is built. |
| `label_area`, `input_area`, `disp_area` | The label, the input container and the read-only display area. |
| `value`, `last_value` | The current and the previous value. |
| `frm`, `doc`, `layout` | The form, the document and the layout that own the control. |
| `grid`, `grid_row` | Set when the control is in a child table. |

`get_status` returns `"None"` for hidden fields. It returns `"Read"` for read-only fields, for virtual fields, for `Read Only` fields, and for fields in a read-only grid. In a document, it also checks the permission level. In a form, a read-only field with no value gets `"None"` if System Settings has **Hide Empty Read-Only Fields** on (default on). This does not apply to `HTML`, `Image`, `Button`, `Attachment Gallery`, `Geolocation` and `Vue Component`.

## Controls in a form

`frm.fields_dict[fieldname]` and `frm.get_field(fieldname)` return the same control. The `df` of a form control is the copy of the DocField for this document, from `frappe.meta.get_docfield(doctype, fieldname, docname)`. So `frm.set_df_property`, `frm.toggle_display`, `frm.toggle_reqd` and `frm.toggle_enable` change one document only. The DocType meta does not change. They set `hidden`, `reqd` and `read_only` on that copy and refresh the field. The full method list is in [Form Scripts](../desk-ui/form-scripts.md#the-frm-object).

```js
frappe.ui.form.on("Sample Order", {
	refresh(frm) {
		const field = frm.get_field("customer_name");
		field.set_description(__("Name as printed on the invoice"));
		frm.toggle_enable("order_date", frm.is_new());
	},
});
```

!!! tip
    To change a value in a form, use `frm.set_value`. `control.set_input` changes only what the user sees, and the value is not saved.

## frappe.ui.FieldGroup

`frappe.ui.FieldGroup` renders a list of field definitions as a layout without a modal. `frappe.ui.Dialog` extends it. Use it to put fields in a desk page or in an HTML field.

```js
const group = new frappe.ui.FieldGroup({
	fields: [
		{ fieldtype: "Data", fieldname: "customer_name", label: __("Customer"), reqd: 1 },
		{ fieldtype: "Date", fieldname: "order_date", label: __("Date"), default: "Today" },
		{ fieldtype: "Check", fieldname: "urgent", label: __("Urgent") },
		{ fieldtype: "Small Text", fieldname: "note", label: __("Note"), depends_on: (doc) => doc.urgent },
	],
	body: page.main.find(".sample-form"),
});
group.make();
group.get_values(true); // {order_date: "2026-11-22", urgent: 0}
group.get_values(); // null, and a "Missing Values Required" message for Customer
```

| Option | Description |
|---|---|
| `fields` | List of field definitions. A field with a `label` and no `fieldname` gets the label in lower case with `_` for spaces. |
| `body` or `parent` | Where to render. |
| `doc` | A document to bind to. If it is saved, defaults are not applied. |
| `no_submit_on_enter` | Do not run the primary action when the user presses <kbd>Enter</kbd>. |
| `no_focus` | Do not focus the first input. |

Methods: `make()`, `get_values(ignore_errors, check_invalid)`, `get_value(fieldname)`, `set_value(fieldname, value)`, `set_values(object)`, `get_field(fieldname)`, `has_field(fieldname)`, `get_input(fieldname)`, `set_df_property(fieldname, prop, value)`, `set_query(fieldname, fn)`, `set_query(child_field, table_field, fn)`, `add_fetch(link_field, source_field, target_field)`, `clear()`, `refresh_dependency()`. Property: `fields_dict`.

How `get_values` works:

- It leaves out fields with an empty value.
- If a mandatory field is empty, it shows **Missing Values Required** and returns `null`. With `ignore_errors` set, it returns the other values.
- With `check_invalid` set, a field with `df.invalid` (for example a wrong email) shows **Invalid Values** and returns `null`.
- If `df.include_default` is set and the value is empty, it uses `df.default`.

Defaults: `Today` (Date) and `now` (Datetime, Time) give the current date or time. `__user` or `user` gives the session user, and `user_fullname` gives the full name.

In a FieldGroup or Dialog, `depends_on`, `mandatory_depends_on` and `read_only_depends_on` are evaluated against the current values. `depends_on` also accepts a function `(values) => boolean` or a boolean, as well as the `eval:` string. Refer to [Field Properties](field-properties.md) for the expression syntax.

## Keys only for dialogs and field groups

A DocField does not store these keys. Set them in JavaScript on the `df` you pass to a Dialog, a FieldGroup or `make_control`. Some of them also work on a form `df` from a form script.

| Key | Fieldtypes | Description |
|---|---|---|
| `change`, `onchange` | All with an input | Function called after the value is set. `this` is the control. |
| `on_make` | All with an input | `on_make(control)`, called once after the input is built. |
| `get_status` | All | `get_status(control)` returns `"Write"`, `"Read"` or `"None"` and replaces the standard check. |
| `initial_value` | All with an input | Value set when the control is built. |
| `include_default` | All | `get_values` returns `default` if the value is empty. |
| `click` | Button | Function called on click. |
| `primary`, `btn_size`, `icon`, `title` | Button | Primary style, size suffix (`btn-<size>`, default `xs`), icon name, and tooltip. |
| `get_query` | Link, Autocomplete | Function that returns the query or filters for the dropdown. Same as `set_query`. Refer to [Link and Select Fields](link-and-select-fields.md). |
| `filters` | Link | Object of filters added to every search. |
| `only_select` | Link | Hide **Create a new ...** and **Advanced Search** in the dropdown. |
| `filter_description` | Link | Text shown at the end of the dropdown in place of the generated filter description. |
| `get_route_options_for_new_doc` | Link | `fn(control)` returns the values for a new document made from the dropdown. |
| `ignore_link_validation` | Link, Table MultiSelect | Do not check that the value exists. |
| `get_options` | Dynamic Link | `fn(control)` returns the DocType to search. In a dialog, the default reads the field named in `options`. |
| `get_data` | MultiCheck, MultiSelect, MultiSelectPills, MultiSelectList, Table | Function that returns the options (or the rows of a Table). It can return a promise. MultiSelectPills and MultiSelectList pass the typed text. |
| `max_items` | Autocomplete | Number of items in the dropdown. Default `99`. |
| `ignore_validation` | Autocomplete, MultiSelect | Accept a value that is not in the options. |
| `min_date`, `max_date`, `disabled_dates` | Date, DateRange | Limits of the date picker. `disabled_dates` is a list of `YYYY-MM-DD` strings (Date only). |
| `hide_timezone` | Datetime | Do not add the time zone to the description. |
| `min_lines`, `max_lines`, `wrap` | Code | Editor height and line wrap. |
| `enable_mentions`, `get_toolbar_options`, `theme` | Text Editor | `@` mentions, a custom Quill toolbar, the Quill theme (default `snow`). |
| `fields`, `data`, `in_place_edit`, `cannot_add_rows`, `cannot_delete_rows`, `on_setup`, `on_add_row` | Table | Child fields, rows, edit in the grid without a row form, block add or delete, and grid hooks. |
| `with_copy_button` | Data | Show a button that copies the value. |
| `input_css`, `input_class` | Data and subclasses | CSS and classes for the input. |
| `is_filter` | Data | Skip the `Email`, `Phone`, `Name` and `URL` checks. Filters use it. |
| `columns`, `select_all`, `on_change` | MultiCheck | Grid layout (`3`, `"15rem"` or `"15rem 5"`), **Select All** buttons, change callback. |

!!! warning
    `MultiCheck` does not call `change` or `onchange`. It calls `df.on_change()` when a box changes.

## Dialog-only controls

These controls have no DocField fieldtype. Use them in a Dialog, a FieldGroup or with `make_control`.

| Fieldtype | Value from `get_value` | Options |
|---|---|---|
| `MultiCheck` | Array of checked values, for example `["a"]`. | `options` is a list of `{label, value, checked}`, or a JSON string. Options are sorted by label unless `sort_options` is `false`. |
| `MultiSelect` | Comma-separated string, for example `"p, q"`. | `options` list, or `get_data()`. |
| `MultiSelectPills` | Array of values. | `get_data(txt)` returns `[{value, label}]`. |
| `MultiSelectList` | Array of selected values. | `options` or `get_data(txt)` returns strings or `{value, label, description}`. |
| `DateRange` | `["YYYY-MM-DD", "YYYY-MM-DD"]`. | `min_date`, `max_date`. |
| `Switch` | `1` or `0`. | A `Check` shown as a toggle switch. |
| `Comment` | HTML. | A Text Editor with the comment box frame. The form timeline uses it. |

```js
const d = new frappe.ui.Dialog({
	title: __("Export Orders"),
	fields: [
		{
			fieldtype: "MultiCheck",
			fieldname: "statuses",
			label: __("Status"),
			columns: 2,
			options: [
				{ label: __("Draft"), value: "Draft", checked: 1 },
				{ label: __("Submitted"), value: "Submitted" },
			],
		},
		{ fieldtype: "DateRange", fieldname: "period", label: __("Period") },
		{ fieldtype: "Switch", fieldname: "include_items", label: __("Include Items"), default: 1 },
	],
	primary_action(values) {
		// {statuses: ["Draft"], period: ["2026-01-01", "2026-01-31"], include_items: 1}
		d.hide();
	},
});
d.show();
```

A `Table` field in a dialog takes its columns from `fields` and its rows from `data`. `get_values()` returns the rows as a list of objects with `idx` and `name`. For grid details, refer to [Table Fields](table-fields.md).

## Add a custom control from an app

`make_control` looks up the class at call time. So an app can add a new class to `frappe.ui.form` from a desk bundle.

1. Make the bundle `sample_app/public/js/sample_app.bundle.js`.
2. Add it to `app_include_js` in `hooks.py`.
3. Define the class in the bundle.

```python
# sample_app/hooks.py
app_include_js = "sample_app.bundle.js"
```

```js
// sample_app/public/js/sample_app.bundle.js
frappe.ui.form.ControlSampleSlider = class ControlSampleSlider extends frappe.ui.form.ControlData {
	static input_type = "range";
	parse(value) {
		return cint(value);
	}
};
```

```js
new frappe.ui.Dialog({
	title: __("Priority"),
	fields: [{ fieldtype: "Sample Slider", fieldname: "level", label: __("Level"), default: 30 }],
}).show();
```

The fieldtype `Sample Slider` maps to `ControlSampleSlider`. `ControlData` reads the static properties `html_element` (default `input`), `input_type` (default `text`) and `input_mode`. Set `static trigger_change_on_input_event = false` to run the change event only on `change`, not while the user types.

!!! note
    A DocField `fieldtype` must be one of the standard fieldtypes, so a new control works only in dialogs, field groups and pages. To change a standard fieldtype in forms, assign a subclass to its name, for example `frappe.ui.form.ControlRating = class extends frappe.ui.form.ControlRating {}`. Classes that already extend it keep the old parent.

## Pitfalls

- `get_value()` returns `undefined` when the field is hidden or read-only and has no value. Read `frm.doc[fieldname]` in a form.
- `set_value` does nothing if the new value equals the current value. Pass `force = true` to run the change event again.
- A Data input runs its change event 500 ms after the user stops typing, not on each key.
- In a dialog, the mandatory style shows only after the user clicks the primary action.
- `frappe.ui.form.make_control` returns `undefined` for an unknown fieldtype. Check the result.

## Related

- [Form Scripts](../desk-ui/form-scripts.md)
- [Client API](../desk-ui/client-api.md)
- [Field Properties](field-properties.md)
- [Field Types](../../reference/data-model/field-types.md)
- [Link and Select Fields](link-and-select-fields.md)
- [Table Fields](table-fields.md)
- [Hooks](../building-apps/hooks.md)
