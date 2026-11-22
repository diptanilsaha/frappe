---
title: UI Components (JavaScript)
description: Reference of v15 desk UI classes - frappe.ui.Dialog, FieldGroup, MultiSelectDialog, Page, FileUploader, Capture, Scanner, Tags, Tree, Frappe Charts, notifications and the sidebar helper.
version: v15
sources:
  - frappe/public/js/frappe/ui/dialog.js
  - frappe/public/js/frappe/ui/field_group.js
  - frappe/public/js/frappe/form/layout.js
  - frappe/public/js/frappe/form/multi_select_dialog.js
  - frappe/public/js/frappe/ui/page.js
  - frappe/public/js/frappe/file_uploader/file_uploader.bundle.js
  - frappe/public/js/frappe/file_uploader/FileUploader.vue
  - frappe/public/js/frappe/ui/capture.js
  - frappe/public/js/frappe/scanner/index.js
  - frappe/public/js/frappe/ui/tags.js
  - frappe/public/js/frappe/ui/tag_editor.js
  - frappe/public/js/frappe/ui/tree.js
  - frappe/public/js/frappe/ui/chart.js
  - frappe/public/js/frappe/utils/utils.js
  - frappe/public/js/frappe/ui/notifications/notifications.js
  - frappe/public/js/frappe/ui/sidebar.js
---

# UI Components (JavaScript)

This page lists the desk UI classes and helpers in `frappe.ui` that an app can use in v15: dialogs, field groups, pages, uploaders, the camera and scanner, tags, trees and charts. Use it to look up options, defaults and method signatures. For how to build a page or a dialog, see [Desk Pages](../../developer-guide/desk-ui/desk-pages.md) and [Client API](../../developer-guide/desk-ui/client-api.md). For field definitions and control methods, see [Controls](../../developer-guide/fields/controls.md).

All names on this page exist in a logged-in v15 desk at `/app`. They were listed with `Object.keys(frappe.ui)` and the class prototypes.

!!! note "Version"
    v15 has no Espresso component functions (`frappe.ui.button`, `frappe.ui.toast`, `frappe.ui.Dropdown` and others) and no `frappe.ui.chart`. Use `frappe.show_alert` for toasts and `frappe.Chart` for charts.

## frappe.ui.Dialog

`new frappe.ui.Dialog(opts)` makes a Bootstrap modal with fields. `Dialog` extends `frappe.ui.FieldGroup`, which extends `frappe.ui.form.Layout`. The constructor builds the modal at once, but does not show it.

```js
const d = new frappe.ui.Dialog({
	title: __("New Sample Item"),
	fields: [
		{ fieldname: "item_name", fieldtype: "Data", label: __("Item Name"), reqd: 1 },
		{ fieldname: "rate", fieldtype: "Currency", label: __("Rate"), default: 10 },
	],
	primary_action_label: __("Create"),
	primary_action(values) {
		// values: { item_name: "Pen", rate: 10 }
		frappe.db.insert({ doctype: "Sample Item", ...values }).then(() => d.hide());
	},
});
d.show();
```

### Options

| Option | Type | Default | Description |
|---|---|---|---|
| `title` | string | | Title. Set as HTML. |
| `fields` | list | | Field definitions. See [Controls](../../developer-guide/fields/controls.md). |
| `size` | string | computed | `"small"`, `"large"` or `"extra-large"`. If not set: 4 or more Column Breaks in one section give `"extra-large"`, 2 or 3 give `"large"`, else normal. |
| `primary_action(values)` | function | | Runs on click of the primary button, and on <kbd>Enter</kbd> in a text input or select. Gets `get_values()`. Does not run if a mandatory field is empty. |
| `primary_action_label` | string | `"Submit"` | Label of the primary button. |
| `secondary_action` | function | | Click handler of the secondary button. |
| `secondary_action_label` | string | | Label of the secondary button. |
| `minimizable` | bool | | Show a minimize button. A click on the title restores the dialog. |
| `static` | bool | | No close on backdrop click or <kbd>Esc</kbd>. Hides the close button. |
| `animate` | bool | `true` | Fade in. |
| `indicator` | string | | Indicator color class in the header, for example `"green"`. |
| `on_hide`, `onhide` | function | | Runs when the dialog closes. |
| `on_page_show` | function | | Runs after the dialog is shown. |
| `on_minimize_toggle(is_minimized)` | function | | Runs after minimize or restore. |
| `no_submit_on_enter` | bool | | <kbd>Enter</kbd> does not run the primary action. |
| `no_focus` | bool | | Do not focus the first input on show. |
| `doc` | object | | Document to bind to the fields. |

### Methods

| Method | Returns | Description |
|---|---|---|
| `show()` | the dialog | Open the dialog. After it shows, sets `window.cur_dialog` and adds the dialog to `frappe.ui.open_dialogs`. |
| `hide()` | | Close the dialog. |
| `cancel()` | | Click the close button. |
| `no_cancel()` | | Hide the close button. |
| `set_title(t)` | | Set the title as HTML. |
| `set_primary_action(label, click)` | jQuery button | Set or replace the primary button. `click(values)` works as `primary_action`. |
| `set_secondary_action(click)` | jQuery button | Set the click handler of the secondary button and show it. |
| `set_secondary_action_label(label)` | | Set the label of the secondary button and show it. |
| `disable_primary_action()`, `enable_primary_action()` | | Disable or enable the primary button. |
| `get_primary_btn()`, `get_secondary_btn()`, `get_close_btn()`, `get_minimize_btn()` | jQuery | Footer and header buttons. |
| `add_custom_action(label, action, css_class = null)` | | Add a button to the footer. |
| `set_message(text)` | | Hide the fields and show `text` (plain text). |
| `clear_message()` | | Show the fields again. |
| `clear()` | | Set every field to its default or empty, and clear the message. |
| `toggle_minimize()` | | Minimize or restore. |

The dialog also has every FieldGroup method. See [frappe.ui.FieldGroup](#frappeuifieldgroup).

Properties: `$wrapper` (the `.modal`), `body` and `$body` (field area), `header`, `footer`, `fields_dict`, `display` (`true` while shown), `is_visible`, `is_minimized`.

Globals:

| Name | Description |
|---|---|
| `window.cur_dialog` | The dialog on top, or `null`. |
| `frappe.ui.open_dialogs` | List of open dialogs, the top one last. |
| `frappe.ui.hide_open_dialog()` | Close `cur_dialog`. A minimizable dialog is minimized instead. |

## frappe.ui.FieldGroup

`new frappe.ui.FieldGroup({ fields, body })` renders fields in any element. Call `make()` after the constructor. A field with a `label` and no `fieldname` gets the label in snake case as fieldname. [Controls](../../developer-guide/fields/controls.md#frappeuifieldgroup) describes the options, `get_values()` rules and default keywords.

| Method | Returns | Description |
|---|---|---|
| `make()` | | Render the fields and apply defaults. |
| `get_values(ignore_errors, check_invalid)` | object or `null` | Values of non-empty fields. `null` and a message if a mandatory field is empty. |
| `get_value(fieldname)` | any | Value of one field. |
| `set_value(fieldname, value)` | Promise | Set one value, then evaluate `depends_on`. |
| `set_values(dict)` | Promise | Set many values. Unknown keys are skipped. |
| `set_input(fieldname, value)` | Promise | Same as `set_value`. |
| `get_field(fieldname)` | control | The control object. |
| `has_field(fieldname)` | bool | |
| `get_input(fieldname)` | jQuery | The input element. |
| `set_df_property(fieldname, prop, value)` | | Change a field property and refresh the field. |
| `clear()` | | Set every field to its default or empty. |
| `refresh_dependency()` | | Evaluate `depends_on`, `mandatory_depends_on` and `read_only_depends_on` again. |
| `add_fields(fields)` | | Render more fields at the end. |
| `replace_field(fieldname, df, render)` | | Replace a field with a new definition. |
| `select_tab(label_or_fieldname)` | | Activate a Tab Break. |
| `show_message(html, color, permanent = false)` | | Show a message block above the fields. `color` is `yellow`, `blue`, `red`, `green` or `orange` (default `blue`). Empty `html` clears it. |

Properties: `fields_dict`, `fields_list`, `wrapper`, `dirty` (`true` after the user changes a value).

v15 FieldGroup has no `set_query` and no `add_fetch`. To filter a Link field, set `get_query` in the field definition.

## frappe.ui.form.MultiSelectDialog

`new frappe.ui.form.MultiSelectDialog(opts)` opens a dialog to search and check many records of a DocType. The constructor loads the DocType meta, builds the dialog and shows it. Search uses `frappe.desk.search.search_widget`, so the user needs read access.

```js
new frappe.ui.form.MultiSelectDialog({
	doctype: "Sample Item",
	target: null,
	setters: { category: "Hardware" },
	data_fields: [{ fieldname: "qty", fieldtype: "Int", label: __("Qty"), default: 1 }],
	get_query() {
		return { filters: { disabled: 0 } };
	},
	action(selections, args) {
		// selections: ["Monitor"], args: { category: "Hardware", qty: 1, filtered_children: [] }
		cur_dialog.hide();
	},
});
```

| Option | Type | Description |
|---|---|---|
| `doctype` | string | DocType to search. |
| `setters` | object or list | Filter fields above the results. An object maps fieldname to default value and copies the field definition from the meta. A list holds full field definitions. A `Data` setter in object form filters with `like`. |
| `read_only_setters` | list | Fieldnames of object setters to show read-only. |
| `action(selections, args)` | function | Primary action. `selections` is the list of checked names. `args` holds the setter values, `data_fields` values, filter group values and `filtered_children`. The dialog stays open. Call `cur_dialog.hide()` to close it. |
| `get_query()` | function | Returns `{ filters, query }`. `filters` is added to the search. `query` is a dotted path of a custom search method. |
| `columns` | list | Result columns. Used only when `get_query()` returns a `query`. Otherwise the columns are `name` and the setters. |
| `data_fields` | list | Extra fields below the results. Their values go to `args`. |
| `add_filters_group` | bool | Show a filter group for more conditions. |
| `primary_action_label` | string | Default `"Get Items"`. |
| `size` | string | Dialog size. |
| `allow_child_item_selection` | bool | Show a check box to pick child table rows instead of parents. Needs `child_fieldname` and `child_columns`. |
| `child_fieldname` | string | Table field of `doctype` that holds the child rows. |
| `child_columns` | list | Child fieldnames to show and search. |

The secondary button "Make <DocType>" sets `frappe.route_options` from the setters and opens a new form.

| Method | Returns | Description |
|---|---|---|
| `get_checked_values()` | list | Names of checked rows. |
| `get_checked_items()` | list | Checked result rows with all columns. |
| `get_results()` | | Run the search again. |

Properties: `dialog` (the `frappe.ui.Dialog`), `results`, `page_length` (20, grows by 20 with "More").

## frappe.ui.Page

`frappe.ui.make_app_page(opts)` makes a `frappe.ui.Page` in `opts.parent`, stores it as `opts.parent.page` and returns it. Forms, lists and reports use the same class, so `frm.page`, `listview.page` and `frappe.query_report.page` have these methods. See [Desk Pages](../../developer-guide/desk-ui/desk-pages.md) for a full page script.

```js
const page = frappe.ui.make_app_page({ parent: wrapper, title: __("Sample Board"), single_column: true });
page.set_primary_action(__("New Order"), () => frappe.new_doc("Sample Order"), "add");
const status = page.add_field({
	fieldname: "status",
	label: __("Status"),
	fieldtype: "Select",
	options: ["", "Open", "Closed"],
	change: () => load(page),
});
page.get_form_values(); // { status: "" }
```

### Options

| Option | Type | Description |
|---|---|---|
| `parent` | element | Wrapper element. Required. |
| `title` | string | Page title. HTML is stripped. Also sets the browser tab title. |
| `single_column` | bool | No side section. |
| `icon` | string | Font Awesome class shown before the title, for example `"fa fa-star"`. |
| `make_page` | function | Runs at the end of setup, with the page as `this`. |
| `card_layout` | bool | Show the main section as a card. |
| `disable_sidebar_toggle` | bool | Remove the sidebar toggle button. |
| `required_libs` | list | Assets that `load_lib(callback)` loads with `frappe.require`. |

### Parts

| Property | Description |
|---|---|
| `body`, `main` | Main content area (jQuery). |
| `sidebar` | Side section (jQuery). Empty with `single_column`. |
| `footer` | Footer below the main area. Hidden by default. |
| `page_form` | Row of filter fields made by `add_field`. |
| `fields_dict` | Controls made by `add_field`, by `fieldname` (or `label`). |
| `btn_primary`, `btn_secondary` | Header buttons. |
| `menu`, `actions` | Dropdown lists of the "..." menu and the Actions menu. |
| `inner_toolbar` | Area of inner buttons. Same element as `custom_actions`. |
| `indicator` | Status pill next to the title. |
| `views`, `current_view` | Views made by `add_view`. |

### Title and indicator

| Method | Description |
|---|---|
| `set_title(title, icon = null, strip = true, tab_title = "", tooltip_label = "")` | Set the title and the browser tab title. `strip` removes HTML. `icon` is a `frappe.utils.icon` name. |
| `set_title_sub(txt)` | Set the subtitle. Empty `txt` hides it. |
| `get_title_area()` | jQuery of the title area. |
| `set_indicator(label, color)` | Show the status pill. `color` is a class such as `green`, `red`, `orange`, `blue`, `gray`. |
| `clear_indicator()` | Hide the status pill. |

### Header buttons

A `click` function that returns a Promise (or jQuery deferred) disables its button until it settles.

| Method | Returns | Description |
|---|---|---|
| `set_primary_action(label, click, icon, working_label)` | jQuery | Set the primary button. `icon` is a `frappe.utils.icon` name. `working_label` is stored on the button as `data-working-label`. |
| `set_secondary_action(label, click, icon, working_label)` | jQuery | Set the secondary button. |
| `clear_primary_action()`, `clear_secondary_action()`, `clear_actions()` | | Hide the buttons. |
| `add_action_icon(icon, click, css_class = "", tooltip_label)` | jQuery | Icon button in the header icon group. |
| `show_icon_group()`, `hide_icon_group()`, `clear_icons()` | | Icon group visibility. |

### Menus

| Method | Returns | Description |
|---|---|---|
| `add_menu_item(label, click, standard, shortcut, show_parent)` | jQuery `<a>` | Item in the "..." menu. `standard` items go after a divider. `shortcut` is a key string such as `"Ctrl+P"` or a `frappe.ui.keys.add_shortcut` object, and is bound. A label that exists is not added again. |
| `add_custom_menu_item(parent, label, click, standard, shortcut, icon = null)` | jQuery `<a>` | Item in the dropdown list `parent`, for example one from `add_custom_button_group`. |
| `add_divider()` | jQuery | Divider in the "..." menu. |
| `clear_menu()`, `show_menu()`, `hide_menu()` | | "..." menu. |
| `add_action_item(label, click, standard)` | jQuery `<a>` | Item in the Actions menu. Shows the menu. |
| `add_actions_menu_item(label, click, standard, shortcut)` | jQuery `<a>` | Item in the Actions menu. Does not show the menu. |
| `show_actions_menu()`, `hide_actions_menu()`, `clear_actions_menu()` | | Actions menu. |
| `add_dropdown_item({ label, click, standard, parent, shortcut, show_parent = true, icon = null })` | jQuery `<a>` | Generic form of the methods above. |
| `clear_user_actions()` | | Remove the non-standard items of the "..." menu. |

### Inner toolbar

| Method | Returns | Description |
|---|---|---|
| `add_inner_button(label, action, group, type = "default")` | jQuery | Button in the toolbar, or an item of the dropdown `group`. A label that exists is not added again. `type` is a Bootstrap button suffix such as `default`, `primary` or `danger`. The button is also an item of the "..." menu on small screens. |
| `remove_inner_button(label, group)` | | Remove a button or a group item. `label` can be a list. A group with no items is removed. |
| `change_inner_button_type(label, group, type)` | | Change the look of a button. |
| `get_inner_group_button(label)` | jQuery | The group element. |
| `set_inner_btn_group_as_primary(label)` | | Make a group button primary. |
| `add_inner_message(message)` | jQuery | Muted message (HTML) at the start of the toolbar. |
| `add_button(label, click, opts)` | jQuery | Button in the toolbar. `opts.btn_class` (`btn-primary`, `btn-danger`, ...), `opts.btn_size` (default `btn-sm`), `opts.icon`. |
| `add_custom_button_group(label, icon, parent)` | jQuery `<ul>` | Dropdown button. Add items with `add_custom_menu_item(<returned ul>, ...)`. |
| `add_dropdown_button(parent, label, click, icon)` | | Calls `frappe.ui.toolbar.add_dropdown_button`. |
| `clear_inner_toolbar()`, `clear_custom_actions()` | | Remove all inner buttons and groups. |

### Page form and views

| Method | Returns | Description |
|---|---|---|
| `add_field(df, parent)` | control | Control in the page form (or in `parent`). Applies `df.default`. Stored in `fields_dict`. An `HTML` field returns nothing. Use `df.change` for a change handler. |
| `get_form_values()` | object | Values of all `add_field` controls. Empty values are included. |
| `add_select(label, options)`, `add_data(label)`, `add_date(label, date)` | jQuery input | Shortcuts that call `add_field`. |
| `add_check(label)` | jQuery input | Plain check box in the page form. |
| `add_label(label)`, `add_break()` | | Label, or a line break in the page form. |
| `clear_fields()`, `show_form()`, `hide_form()` | | Page form. |
| `add_view(name, html)` | jQuery | Add a content view. The first view is shown, the others are hidden. |
| `set_view(name)` | | Show one view. Triggers `view-change` on `page.wrapper`. |
| `get_empty_state(title, message, primary_action)` | jQuery | Card with a title, a message and a button label. Not attached. |
| `load_lib(callback)` | | `frappe.require(page.required_libs, callback)`. |

`frappe.ui.pages` maps a route string to the last page made on that route.

## frappe.ui.FileUploader

`new frappe.ui.FileUploader(opts)` opens the upload dialog. With `wrapper`, it renders inside that element without a dialog. Files go to `/api/method/upload_file` (whitelisted, `POST`), one request per file. Each upload makes a `File` record.

```js
new frappe.ui.FileUploader({
	doctype: "Sample Order",
	docname: frm.doc.name,
	folder: "Home/Attachments",
	restrictions: { allowed_file_types: [".pdf", "image/*"] },
	on_success(file_doc) {
		// file_doc: { doctype: "File", name: "405e5841b5", file_url: "/private/files/a.pdf", is_private: 1, ... }
		frm.reload_doc();
	},
});
```

| Option | Default | Description |
|---|---|---|
| `doctype`, `docname` | | Attach the file to this document. |
| `fieldname` | | Attach field to set. |
| `folder` | `"Home"` | File folder. |
| `method` | | Dotted path of a whitelisted method that `upload_file` calls with the file instead of saving it. |
| `on_success(file_doc, response)` | | Runs once per uploaded file. With `as_dataurl`, gets the file object with `dataurl`. |
| `allow_multiple` | `true` | Allow more than one file. |
| `as_dataurl` | `false` | Do not upload. Return base64 data URLs to `on_success`. |
| `restrictions` | `{}` | `max_file_size` (bytes, default from `frappe.core.api.file.get_max_file_size`), `max_number_of_files` (default `max_attachments` of the DocType), `allowed_file_types` (MIME types or extensions, default from System Settings `allowed_file_extensions`), `crop_image_aspect_ratio`. |
| `make_attachments_public` | | Make new files public. Files are private by default. |
| `allow_toggle_private` | `true` | Show the private and public toggle. |
| `allow_toggle_optimize` | `true` | Show the image optimize toggle. Images larger than 200 KB are optimized by default. |
| `allow_web_link` | `true` | Show "Link". |
| `allow_take_photo` | `true` | Show "Camera" when the browser has a camera API. |
| `allow_google_drive` | `true` | Show Google Drive when Google Settings has it on. |
| `disable_file_browser` | `false` | Hide "Library". |
| `upload_notes` | | Help text under the drop area. |
| `attach_doc_image` | `false` | Resize the image to 200 x 200 on the server. |
| `files` | | `File` objects to add at once. |
| `dialog_title` | `"Upload"` | Dialog title. |
| `wrapper` | | Render here instead of in a dialog. |
| `frm` | | Form. Its attachment limit is checked first. |

| Member | Description |
|---|---|
| `upload_files()` | Upload the selected files. Returns a Promise. |
| `dialog` | The `frappe.ui.Dialog`, if any. |
| `uploader` | The mounted Vue component. |
| `frappe.ui.FileUploader.UploadOptions` | Static list of extra upload sources: `{ label, icon, action({ dialog, uploader, doctype, docname, fieldname }) }`. |

## frappe.ui.Capture

`new frappe.ui.Capture(options)` takes photos with the camera. On a phone it opens the file input with `capture`. On desktop, or when the user default `force_web_capture_mode_for_uploads` is on, it shows a video stream.

| Option | Default | Description |
|---|---|---|
| `title` | `"Camera"` | Dialog title. |
| `animate` | `false` | Fade the dialog. |
| `error` | `false` | Show an alert when the camera fails to load. |

| Method | Description |
|---|---|
| `show()` | Open the camera. |
| `submit(fn)` | Set the callback. `fn(images)` gets a list of data URLs on "Submit". |
| `hide()` | Close the dialog and stop the stream. |
| `set_options(options)` | Merge options with `frappe.ui.Capture.OPTIONS`. Returns the instance. |

```js
const capture = new frappe.ui.Capture({ title: __("Photo") });
capture.submit((images) => console.log(images.length));
capture.show();
```

## frappe.ui.Scanner

`new frappe.ui.Scanner(options)` scans barcodes and QR codes with the camera. It loads `html5-qrcode` on demand. Browsers allow the camera only on HTTPS or `localhost`.

| Option | Description |
|---|---|
| `dialog` | `true` opens a "Scan QRCode" dialog and starts scanning. |
| `container` | Element to scan in, without a dialog. Call `scan()` then. |
| `on_scan(result)` | Runs on a scan. `result.decodedText` is the text. |
| `multiple` | Default `false`. `false` stops after the first scan and closes the dialog. |

Methods: `scan()`, `stop_scan()`.

## frappe.ui.Tags and frappe.ui.TagEditor

`frappe.ui.Tags` is a tag input. `frappe.ui.TagEditor` uses it for the document tags in the form sidebar and saves them with `frappe.desk.doctype.tag.tag.add_tag` and `remove_tag`.

| `frappe.ui.Tags` option | Description |
|---|---|
| `parent` | jQuery container. The add button goes into its `.form-sidebar-items` child. |
| `placeholder` | Content of the add button (HTML). |
| `tagsList` | Initial tags. |
| `onTagAdd(tag)`, `onTagRemove(tag)`, `onTagClick(tag)` | Callbacks. |

Methods: `addTag(label)`, `addTags(labels)`, `removeTag(label)`, `clearTags()`, `activate()` (show the input), `deactivate()`.

`frappe.ui.TagEditor` options: `parent`, `frm` (required, gives `doctype` and `docname`), `user_tags` (comma-separated string), `on_change(user_tags)`. Method: `refresh(user_tags)`.

## frappe.ui.Tree

`new frappe.ui.Tree(opts)` renders a lazy-loading tree. The tree view of a DocType uses it. See [List and Report Views](list-and-report-views.md#tree-view-frappetreeview_settings) for tree view settings.

| Option | Default | Description |
|---|---|---|
| `parent` | | Container element. |
| `label` | | Root label. |
| `root_value` | `label` | Value of the root node. |
| `method` | | Whitelisted method that returns children. Called with `args`, `parent` and `is_root`. Returns `[{ value, title, expandable }]`. |
| `args` | | Extra arguments for `method`. |
| `toolbar` | | Node buttons: `{ label, get_label(), click(node), condition(node), btnClass }`. |
| `get_label(node)` | | Return the node label. |
| `on_click(node)`, `on_render(node)`, `on_get_node(nodes, deep)` | | Node hooks. |
| `with_skeleton` | `1` | Show a loading skeleton. |
| `icon_set` | folder icons | `{ open, closed, leaf }` HTML. |

Methods: `refresh()` (reload the parent of the selected node), `load_children(node, deep = false)` (Promise, `deep` loads all levels with `frappe.desk.treeview.get_all_nodes`), `reload_node(node)`, `expand_node(node, click = true)`, `toggle_node(node)`, `get_selected_node()`, `set_selected_node(node)`. Properties: `root_node`, `nodes`.

## Charts

`frappe.Chart` is the Frappe Charts class. `new frappe.Chart(element, config)` draws an SVG chart into `element` (an element or a selector) and returns the chart.

```js
const chart = new frappe.Chart(page.main.find(".sample-chart")[0], {
	type: "bar",
	data: {
		labels: ["Jan", "Feb"],
		datasets: [{ name: __("Orders"), values: [5, 7] }],
	},
	height: 240,
});
chart.update({ labels: ["Jan", "Feb", "Mar"], datasets: [{ name: __("Orders"), values: [5, 7, 9] }] });
chart.addDataPoint("Apr", [3]);
chart.removeDataPoint(0);
```

| Config key | Description |
|---|---|
| `type` | For example `bar`, `line`, `pie`, `percentage`, `heatmap`, `donut` or `axis-mixed`. |
| `data` | `{ labels, datasets: [{ name, values, chartType }] }`. |
| `title`, `height`, `colors` | Title, height in pixels, list of color names or hex values. |
| `axisOptions`, `barOptions`, `lineOptions`, `tooltipOptions` | Frappe Charts options. |

Chart methods used in desk: `update(data)`, `addDataPoint(label, values, index)`, `removeDataPoint(index)`, `updateDataset(values, index)`, `export()`, `destroy()`.

| Name | Description |
|---|---|
| `frappe.utils.make_chart(wrapper, custom_options = {})` | Make a `frappe.Chart` with desk defaults: type `bar`, color `light-blue`, short Y axis numbers. Object options are merged. |
| `frappe.ui.RealtimeChart(element, socketEvent, maxLabelPoints = 8, data)` | `frappe.Chart` subclass. `start_updating()` listens to the realtime event `socketEvent` and adds each `{ label, points }` it gets. `stop_updating()` stops. Throws if `data` has more than `maxLabelPoints` points. |

## Notifications and sidebar

The desk navbar makes one `frappe.ui.Notifications`. An app does not make another. Use these helpers:

| Function | Description |
|---|---|
| `frappe.ui.notifications.get_notification_config()` | Promise of `frappe.desk.notifications.get_notification_info`. Caches it in `frappe.ui.notifications.config`. |
| `frappe.ui.notifications.show_open_count_list(doctype)` | Open the list of `doctype` with the "open" conditions of the `notification_config` hook as filters. |

`frappe.ui.Sidebar({ wrapper, css_class })` is a small helper that builds a list sidebar in `wrapper`. The workspace sidebar of v15 desk does not use it. See [Workspaces, Sidebar and Dock](../../developer-guide/desk-ui/workspaces-sidebar-dock.md).

| Method | Description |
|---|---|
| `add_item(item, section, h6 = false)` | Add `{ label, href, on_click, name }` or a jQuery element to the section with heading `section`. |
| `remove_item(name)` | Remove an item added with `name`. |
| `get_section(section_heading = "")` | Return the section list, or make it. |

## Deprecated

| Name | Use instead |
|---|---|
| `page.add_help_button(txt)` | Does nothing. |

## Pitfalls

- The `values` option is ignored by `frappe.ui.Dialog` and `frappe.ui.FieldGroup`. Call `set_values()` after `make()` (or after the dialog constructor).
- `primary_action` does not close the dialog. Call `d.hide()`.
- A `primary_action` that returns a Promise does not disable the dialog button. Disable it yourself with `disable_primary_action()`.
- `set_message()` sets plain text. `set_title()` sets HTML. Escape user data.
- `MultiSelectDialog` with `columns` and no `get_query().query` ignores `columns`.
- `frappe.ui.Tags.removeTag(label)` updates the list and calls `onTagRemove`, but leaves the tag element. A click on the tag close button removes both.
- `frappe.ui.TagEditor` reads `frm.doctype` and `frm.docname`. Options `doctype` and `docname` alone are not enough.
- `frappe.ui.Tree.toggle()` throws, because tree nodes have no `toggle` method. Use `toggle_node(node)`.

## Related

- [Controls](../../developer-guide/fields/controls.md)
- [Desk Pages](../../developer-guide/desk-ui/desk-pages.md)
- [Client API](../../developer-guide/desk-ui/client-api.md)
- [frappe (JavaScript)](frappe.md)
- [Form (JavaScript)](form.md)
- [List and Report Views](list-and-report-views.md)
