---
title: Layout and Display Fields
description: How the desk builds a form from Section Break, Column Break, Tab Break and Fold fields, and how Heading, HTML, Button, Attachment Gallery and Vue Component fields work.
version: develop
sources:
  - frappe/public/js/frappe/form/layout.js
  - frappe/public/js/frappe/form/section.js
  - frappe/public/js/frappe/form/column.js
  - frappe/public/js/frappe/form/tab.js
  - frappe/public/js/frappe/form/form.js
  - frappe/public/js/frappe/form/grid_row.js
  - frappe/public/js/frappe/form/controls/heading.js
  - frappe/public/js/frappe/form/controls/html.js
  - frappe/public/js/frappe/form/controls/button.js
  - frappe/public/js/frappe/form/controls/attachment_gallery.js
  - frappe/public/js/frappe/form/controls/vue_component.js
  - frappe/public/js/frappe/form/sidebar/attachments.js
  - frappe/core/doctype/doctype/doctype.py
  - frappe/core/doctype/docfield/docfield.json
  - frappe/desk/form/load.py
  - frappe/handler.py
  - frappe/model/__init__.py
---

# Layout and Display Fields

Layout fields split a form into tabs, sections and columns. Display fields show content but store no value. None of these fieldtypes has a database column. They are in `no_value_fields` in `frappe/model/__init__.py`, so they cannot be mandatory, in list view or in global search. For the list of fieldtypes, see [Field Types](../../reference/data-model/field-types.md).

| Fieldtype | Use it to |
|---|---|
| `Tab Break` | Start a tab. |
| `Section Break` | Start a section, optionally with a label and collapse. |
| `Column Break` | Start a new column in the current section. |
| `Fold` | Hide the rest of the form behind "Show more details". |
| `Heading` | Show the label as a heading. |
| `HTML` | Show static or rendered HTML. |
| `Button` | Run a form script handler or a controller method. |
| `Attachment Gallery` | Show the attachments of the document as a grid. |
| `Vue Component` | Mount a UI island inside the form. |

## How the form layout is built

`frappe.ui.form.Layout.render()` in `frappe/public/js/frappe/form/layout.js` reads the fields in `idx` order. Each field goes into the current tab, section and column.

1. If the DocType has a `Tab Break` and the first visible field is not one, the layout adds a tab "Details" (fieldname `__details`) at the start.
2. If the DocType has no tab and the first field is not a `Section Break`, the layout adds a section at the start.
3. A `Tab Break` starts a tab and a new section in it.
4. A `Section Break` starts a section. The next field starts the first column.
5. A `Column Break` starts a column in the current section.
6. A `Fold` puts the next sections on a hidden page.
7. Any other field goes into the current column.

Sections and columns that the layout adds get fieldnames such as `__section_1` and `__column_1`.

At display time:

- The columns of a section have equal width (`col-sm-<12 / n>`). Hidden columns do not count.
- A section with no visible field is hidden.
- A tab with no visible section is hidden. A tab is also hidden if the user has no read permission on its `permlevel`.
- If only one tab is visible, the tab bar is hidden.
- `depends_on` and `hidden` work on `Tab Break`, `Section Break` and `Column Break`. They hide the whole tab, section or column.

When you save a DocType, a layout field without a fieldname gets one. With a label, the fieldname is the label in snake case plus `_section`, `_column` or `_tab`. Without a label, it is `section_break_`, `column_break_` or `tab_break_` plus 4 random characters.

## `Tab Break`

- **Desk**: a tab in the tab bar. The label is the tab name.
- **Properties**: `show_dashboard` puts the form dashboard (connections, stats) at the top of this tab. Without it, the dashboard goes on the first tab.
- **JS**: `frm.layout.tabs` is the list of `Tab` objects. `frm.layout.select_tab(label_or_fieldname)` opens a tab. A URL hash with the tab fieldname, for example `#dates_tab`, opens that tab on load.

## `Section Break`

- **Desk**: a section with an optional label (head) and description.
- **Properties**:

| Property | Effect |
|---|---|
| `label` | Section head. A collapsible section needs a label. Without one, it does not collapse. |
| `description` | Text below the head. |
| `collapsible` | Click the head to collapse or expand. The section starts collapsed. |
| `collapsible_depends_on` | Expression. When it is true, the section is expanded. See [Field Properties](field-properties.md). |
| `hide_border` | No top border between this section and the one before. |

A collapsible section stays expanded if a mandatory field in it has no value, or if the user expanded it.

- **JS**: `frm.fields_dict.<section fieldname>` is a `Section` object, not a control. It has `wrapper`, `body`, `collapse(hide)`, `is_collapsed()`, `hide()` and `show()`. `frm.toggle_display("<section fieldname>", false)` hides the section.

```js
frappe.ui.form.on("Sample Order", {
	refresh(frm) {
		frm.fields_dict.more_info_section.collapse(false);
	},
});
```

## `Column Break`

- **Desk**: starts a column. A `label` shows above the column. A `description` shows below the label.
- **JS**: columns are not in `frm.fields_dict`. Use `depends_on` or `frm.toggle_display` on a column fieldname to hide it. The other columns then get wider.

## `Fold`

- **Desk**: a "Show more details" link. The sections after the `Fold` are hidden until the user clicks it.
- **Server**: when you save a DocType, `check_fold()` in `frappe/core/doctype/doctype/doctype.py` checks:
  - Only one `Fold` is in the form.
  - The next field is a `Section Break`.
  - The `Fold` is not the last field.
- **JS**: `frm.layout.unfold()` toggles the hidden part.

## `Heading`

- **Desk**: the translated label in an `<h4>`.
- **JS**: `frappe.ui.form.ControlHeading` (extends `ControlHTML`).

## `HTML`

- **Desk**: the content of `options`, translated with `__()` and rendered with `frappe.render()`. The template context is the control, so `doc`, `frm` and `df` are available.
- **Properties**: `options` holds the HTML.
- **JS**: `frappe.ui.form.ControlHTML` (extends `Control`). Set the content from a form script:

```js
frappe.ui.form.on("Sample Order", {
	refresh(frm) {
		frm.get_field("status_html").html(`<p>${__("Total")}: ${format_currency(frm.doc.total)}</p>`);
	},
});
```

| Method | Effect |
|---|---|
| `html(html)` | Replace the content. Without an argument, render `options` again. |
| `set_value(html)` | Set `df.options` to `html` and render it. A jQuery object is appended instead. The document value does not change. |
| `$wrapper` | jQuery element of the field. |

```json
{"fieldname": "help_html", "fieldtype": "HTML", "options": "<p>{{ doc.doctype }}: {{ __(\"Fill all fields\") }}</p>"}
```

!!! warning
    Escape document values before you put them in HTML. `frappe.render()` does not escape them.

## `Button`

- **Desk**: a button with the label. The field label is not shown above it. A button is always clickable, also on a read-only or submitted form.
- **Properties**:

| Property | Effect |
|---|---|
| `button_color` | `Default`, `Primary`, `Info`, `Success`, `Warning` or `Danger`. |
| `options` | Optional. Name of a whitelisted controller method. Used only when no form script handles the click. |

- **Click order**: the control calls the form script event with the fieldname of the button. If there is no handler and `options` is set, it calls `run_doc_method` with the method name. The source marks this path as deprecated. The method must be whitelisted.

```js
frappe.ui.form.on("Sample Order", {
	recalculate(frm) {
		frm.call("recalculate").then(() => frm.reload_doc());
	},
});
```

- **Child tables**: a `Button` in a child DocType triggers the event on the child DocType, with `(frm, cdt, cdn)`. `Button` and `HTML` fields can be `in_list_view` in a child table. In an editable grid, the button shows in every row.
- **Dialogs**: in a `frappe.ui.Dialog` or `FieldGroup`, set `click` on the field. See [Controls](controls.md).
- **JS**: `frappe.ui.form.ControlButton` (extends `ControlData`).

## `Attachment Gallery`

- **Desk**: a grid of the attachments of the document, sorted by file name. Images open in a lightbox. Other files open in a new tab. With write permission, the grid has an upload button and a delete button on each file. The upload button is hidden when the DocType `max_attachments` is reached. An unsaved document shows "Save the document to attach files."
- **Properties**:

| Property | Effect |
|---|---|
| `link_filters` | Optional JSON list of filters on `File`, for example `[["File", "file_type", "=", "PDF"]]`. A value that starts with `eval:` is evaluated with `doc`, `parent` and `frappe`. |
| `make_attachment_public` | Select "public" in the uploader by default. |

- **Server**: on DocType save, each filter must have 4 values and target `File`. With filters, the control calls `frappe.desk.form.load.get_filtered_attachments`. It checks read permission on the document.
- **Uploads**: the upload button makes a `File` with `attached_to_field` set to the gallery fieldname. Use a filter on `attached_to_field` to show only these files:

```json
{
	"fieldname": "photos",
	"fieldtype": "Attachment Gallery",
	"label": "Photos",
	"link_filters": "[[\"File\", \"attached_to_field\", \"=\", \"photos\"]]"
}
```

- **JS**: `frappe.ui.form.ControlAttachmentGallery` (extends `ControlInput`). It refreshes on the `attachments_change` event of the form.

## `Vue Component`

- **Desk**: mounts the UI island named in `options` with `frappe.ui.mount_island()`. If the island does not exist, the field shows `"<name>" is not available on this site.`
- **Props** passed to the island:

| Prop | Value |
|---|---|
| `doctype` | DocType of the document |
| `name` | Document name, `null` for a new document |
| `doc` | A copy of the document |
| `is_new` | `true` for a new document |
| `read_only` | `true` when the field is not editable |

- **Updates**: when the form becomes dirty, the control calls `island.update()` with new props, at most every 300 ms. The island is unmounted when the field is removed.
- **JS**: `frappe.ui.form.ControlVueComponent` (extends `Control`). To build an island, see [Islands](../desk-ui/islands.md).

## Pitfalls

- A `Section Break` with `collapsible` and no `label` does not collapse.
- `frm.fields_dict` has sections, but not columns or tabs.
- Changing `options` of an `HTML` field in Customize Form changes the content for all users. For content per document, set it in `refresh`.
- `depends_on` on layout fields runs only in the desk. The server does not evaluate it.

## Related

- [Form Scripts](../desk-ui/form-scripts.md)
- [Field Properties](field-properties.md)
- [Media and Special Fields](media-fields.md)
- [Controls](controls.md)
- [Islands](../desk-ui/islands.md)
- [Field Types](../../reference/data-model/field-types.md)
- [Customize Form](../../user-guide/customization/customize-form.md)
