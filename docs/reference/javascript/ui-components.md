---
title: UI Components (JavaScript)
description: Reference of desk UI classes - frappe.ui.Dialog, FieldGroup, MultiSelectDialog, LinkSelector, Page, FileUploader, Capture, Scanner, Tags, Tree, charts, Espresso components, notifications and the sidebar.
version: v16
sources:
  - frappe/public/js/frappe/ui/dialog.js
  - frappe/public/js/frappe/ui/field_group.js
  - frappe/public/js/frappe/form/layout.js
  - frappe/public/js/frappe/form/multi_select_dialog.js
  - frappe/public/js/frappe/form/link_selector.js
  - frappe/public/js/frappe/ui/page.js
  - frappe/public/js/frappe/file_uploader/file_uploader.bundle.js
  - frappe/public/js/frappe/file_uploader/FileUploader.vue
  - frappe/public/js/frappe/ui/capture.js
  - frappe/public/js/frappe/scanner/index.js
  - frappe/public/js/frappe/ui/tags.js
  - frappe/public/js/frappe/ui/tag_editor.js
  - frappe/public/js/frappe/ui/tree.js
  - frappe/public/js/frappe/ui/chart.js
  - frappe/public/js/frappe/ui/island/chart/chart.vue
  - frappe/public/js/frappe/ui/island/chart/legacy_config.js
  - frappe/public/js/frappe/ui/island/loader.js
  - frappe/public/js/frappe/ui/components/button.js
  - frappe/public/js/frappe/ui/components/badge.js
  - frappe/public/js/frappe/ui/components/alert.js
  - frappe/public/js/frappe/ui/components/avatar.js
  - frappe/public/js/frappe/ui/components/avatar_group.js
  - frappe/public/js/frappe/ui/components/breadcrumbs.js
  - frappe/public/js/frappe/ui/components/divider.js
  - frappe/public/js/frappe/ui/components/skeleton.js
  - frappe/public/js/frappe/ui/components/toast.js
  - frappe/public/js/frappe/ui/components/empty_state.js
  - frappe/public/js/frappe/ui/components/progress.js
  - frappe/public/js/frappe/ui/components/panel_header.js
  - frappe/public/js/frappe/ui/components/menu.js
  - frappe/public/js/frappe/ui/components/dropdown.js
  - frappe/public/js/frappe/ui/components/segmented_button.js
  - frappe/public/js/frappe/ui/components/context_menu.js
  - frappe/public/js/frappe/ui/components/tooltip.js
  - frappe/public/js/frappe/ui/components/popover.js
  - frappe/public/js/frappe/ui/components/hover_card.js
  - frappe/public/js/frappe/ui/components/combobox.js
  - frappe/public/js/frappe/ui/components/multi_combobox.js
  - frappe/public/js/frappe/ui/components/tabs.js
  - frappe/public/js/frappe/ui/components/tab_buttons.js
  - frappe/public/js/frappe/ui/components/stepper.js
  - frappe/public/js/frappe/ui/components/stat_card.js
  - frappe/public/js/frappe/ui/components/optional_sections.js
  - frappe/public/js/frappe/ui/components/bottom_sheet.js
  - frappe/public/js/frappe/ui/notifications/notifications.js
  - frappe/public/js/frappe/ui/sidebar/sidebar.js
  - frappe/public/js/desk.bundle.js
  - frappe/core/doctype/system_settings/system_settings.json
---

# UI Components (JavaScript)

This page lists the desk UI classes and helpers in `frappe.ui` that an app can use: dialogs, field groups, pages, uploaders, charts and the Espresso components (buttons, menus, toasts and more). Use it to look up options, defaults and method signatures. For how to build a page or a dialog, see [Desk Pages](../../developer-guide/desk-ui/desk-pages.md) and [Client API](../../developer-guide/desk-ui/client-api.md). For field definitions and control methods, see [Controls](../../developer-guide/fields/controls.md).

All names on this page exist in a logged-in desk at `/desk`. Names from a lazy bundle say which bundle to load with `frappe.require`.

## frappe.ui.Dialog

`new frappe.ui.Dialog(opts)` makes a Bootstrap modal with fields. `Dialog` extends `frappe.ui.FieldGroup`, which extends `frappe.ui.form.Layout`. The constructor builds the modal at once (`auto_make`), but does not show it.

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
		return frappe.db.insert({ doctype: "Sample Item", ...values }).then(() => d.hide());
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
| `primary_action(values)` | function | | Runs on click of the primary button, and on <kbd>Enter</kbd> in a text input. Gets `get_values()`. Does not run if a mandatory field is empty. If it returns a Promise, the button shows a busy state until it settles. |
| `primary_action_label` | string | `"Submit"` | Label of the primary button. |
| `primary_action_loading_label` | string | | Label while the primary button is busy. |
| `secondary_action` | function | | Click handler of the secondary button. |
| `secondary_action_label` | string | | Label of the secondary button. |
| `minimizable` | bool | | Show a minimize button. A click on the title restores the dialog. |
| `static` | bool | | No close on backdrop click or <kbd>Esc</kbd>. Hides the close button. |
| `centered` | bool | `false` | Center the dialog on the screen. |
| `animate` | bool | `true` | Fade in. |
| `auto_make` | bool | `true` | Build the modal in the constructor. |
| `keep_grid_form_open` | bool | `false` | Do not close an open grid row form when the dialog closes. |
| `sheet_on_mobile` | bool | `false` | On a phone, show as a `frappe.ui.BottomSheet`. Ignored for `static` and `minimizable` dialogs. |
| `indicator` | string | | Indicator color class in the header, for example `"green"`. |
| `on_hide`, `onhide` | function | | Runs when the dialog closes. |
| `on_page_show` | function | | Runs after the dialog is shown. |
| `on_minimize_toggle(is_minimized)` | function | | Runs after minimize or restore. |
| `no_submit_on_enter` | bool | | <kbd>Enter</kbd> does not run the primary action. |
| `no_focus` | bool | | Do not focus the first input on show. |
| `doc` | object | | Document to bind. If it is saved, field defaults are not applied. |

### Methods

| Method | Returns | Description |
|---|---|---|
| `show()` | the dialog | Open the dialog. Sets `window.cur_dialog` and adds the dialog to `frappe.ui.open_dialogs`. |
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
| `set_alert(text, alert_class = "info")` | | Put an alert (HTML) above the fields. `alert_class` is a Bootstrap alert class suffix. |
| `clear_alert()` | | Remove the alert. |
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

`new frappe.ui.FieldGroup({ fields, body })` renders fields in any element. Call `make()` after the constructor. [Controls](../../developer-guide/fields/controls.md#frappeuifieldgroup) describes the options, `get_values()` rules and default keywords.

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
| `set_query(fieldname, query)`, `set_query(fieldname, table_fieldname, query)` | | Set `get_query` of a Link field, or of a Link field in a Table field. |
| `add_fetch(link_field, source_field, target_field, target_doctype)` | | Copy `source_field` of the linked record into `target_field`. |
| `clear()` | | Set every field to its default or empty. |
| `refresh_dependency()` | | Evaluate `depends_on`, `mandatory_depends_on` and `read_only_depends_on` again. |
| `add_fields(fields)` | | Render more fields at the end. |
| `replace_field(fieldname, df, render)` | | Replace a field with a new definition. |
| `select_tab(label_or_fieldname)` | | Activate a Tab Break. |
| `show_message(html, color, permanent = false)` | jQuery | Show a message block above the fields. `color` is `yellow`, `blue`, `red`, `green`, `orange` or `white`. Empty `html` clears it. |

Properties: `fields_dict`, `fields_list`, `wrapper`, `dirty` (`true` after the user changes a value).

## frappe.ui.form.MultiSelectDialog

`new frappe.ui.form.MultiSelectDialog(opts)` opens a dialog to search and check many records of a DocType. The constructor loads the DocType meta, builds the dialog and shows it. Search uses `frappe.desk.search.search_widget`, so the user needs read access.

```js
new frappe.ui.form.MultiSelectDialog({
	doctype: "Sample Item",
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
| `get_results()` | Promise | Run the search again. |

Properties: `dialog` (the `frappe.ui.Dialog`), `results`, `page_length` (20, grows by 20 with "More").

## frappe.ui.form.LinkSelector

`new frappe.ui.form.LinkSelector(opts)` opens a "Select <DocType>" dialog with a search box and 10 results per page. A Link field opens it for advanced search. The grid **Add Multiple** button (`grid.set_multiple_add`) also opens it. Search uses `frappe.link_search` on `name`.

| Option | Description |
|---|---|
| `doctype` | DocType to search. The constructor loads its meta, then shows the dialog. |
| `target` | A Link control, or a grid. A click on a result sets the value in the control, or adds a row to the grid. |
| `txt` | Start text of the search box. |
| `fieldname` | Grid only: Link field of the child DocType to set. |
| `qty_fieldname` | Grid only: ask for a quantity and set it in this field. A row with the same value gets the new quantity. |

## frappe.ui.Page

`frappe.ui.make_app_page(opts)` makes a `frappe.ui.Page` in `opts.parent`, stores it as `opts.parent.page` and returns it. Forms, lists and reports use the same class, so `frm.page`, `listview.page` and `frappe.query_report.page` have these methods. See [Desk Pages](../../developer-guide/desk-ui/desk-pages.md) for a full page script.

```js
const page = frappe.ui.make_app_page({ parent: wrapper, title: __("Sample Board"), single_column: true });
page.set_primary_action(__("New Order"), () => frappe.new_doc("Sample Order"), "plus");
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

| Option | Type | Default | Description |
|---|---|---|---|
| `parent` | element | required | Wrapper element. |
| `title` | string | | Page title. HTML is stripped. Also sets the browser tab title. |
| `single_column` | bool | | No side section. |
| `sidebar_position` | string | left | `"Right"` puts the side section on the right. |
| `icon` | string | | Icon name shown before the title. |
| `make_page` | function | | Runs at the end of setup, with the page as `this`. |
| `hide_sidebar` | bool | `false` | Hide the desk sidebar on desktop. Closes the drawer on a small screen. |
| `hide_dock` | bool | `false` | Hide the dock. |
| `hide_mobile_nav` | bool | `false` | Hide the phone tab bar. |
| `required_libs` | list | | Assets that `load_lib(callback)` loads with `frappe.require`. |
| `show_breadcrumbs` | bool | | `false` draws no breadcrumb trail. |

### Parts

| Property | Description |
|---|---|
| `body`, `main` | Main content area (jQuery). |
| `sidebar` | Side section (jQuery). Empty with `single_column`. |
| `footer` | Footer below the main area. Hidden by default. |
| `page_form` | Row of filter fields made by `add_field`. |
| `fields_dict` | Controls made by `add_field`, by `fieldname` (or `label`). |
| `btn_primary`, `btn_secondary` | Header buttons. |
| `menu`, `actions` | Item stores of the "..." menu and the Actions menu. |
| `inner_toolbar` | Area of inner buttons. Same element as `custom_actions`. |
| `indicator` | Status pill next to the title. |
| `views`, `current_view` | Views made by `add_view`. |

### Title and indicator

| Method | Description |
|---|---|
| `set_title(title, icon = null, strip = true, tab_title = "", tooltip_label = "")` | Set the title (last breadcrumb) and the browser tab title. `strip` removes HTML. |
| `set_title_sub(txt)` | Set the subtitle. Empty `txt` hides it. |
| `get_title_area()` | jQuery of the title area. |
| `set_breadcrumbs(items)` | Set the trail. `items` take `frappe.ui.breadcrumbs` item keys. |
| `get_breadcrumbs()` | Current trail. |
| `set_indicator(label, color)` | Show the status pill. `color` is a theme name such as `green`, `red`, `orange`, `blue`, `gray`. |
| `clear_indicator()` | Hide the status pill. |

### Header buttons

A `click` function that returns a Promise (or jQuery deferred) puts its button in a busy state until it settles.

| Method | Returns | Description |
|---|---|---|
| `set_primary_action(label, click, icon, working_label)` | jQuery | Set the primary button. `label` can be `{ label, short_label, options }`. `short_label` shows on small screens. `options` adds a chevron menu with `frappe.ui.Dropdown` items. `working_label` shows while busy. |
| `set_primary_action_options(options)` | | Set or remove the chevron menu next to the primary button. |
| `set_secondary_action(label, click, icon, working_label)` | jQuery | Set the secondary button. |
| `clear_primary_action()`, `clear_secondary_action()`, `clear_actions()` | | Hide the buttons. |
| `add_action_icon(icon, click, css_class = "", tooltip_label)` | jQuery | Icon button in the header icon group. |
| `show_icon_group()`, `hide_icon_group()`, `clear_icons()` | | Icon group visibility. |

### Menus

| Method | Returns | Description |
|---|---|---|
| `add_menu_item(label, click, standard, shortcut, show_parent, group)` | jQuery `<a>` | Item in the "..." menu. `standard` items go after a divider. `shortcut` is a key string such as `"ctrl+p"` or a `frappe.ui.keys.add_shortcut` object, and is bound. `group` is a heading. A label that exists is not added again. |
| `add_custom_menu_item(parent, label, click, standard, shortcut, icon = null)` | jQuery `<a>` | Item in the store `parent`, for example one from `add_custom_button_group`. |
| `add_divider()` | jQuery | Divider in the "..." menu. |
| `clear_menu()`, `show_menu()`, `hide_menu()` | | "..." menu. |
| `add_action_item(label, click, standard, icon = null)` | jQuery `<a>` | Item in the Actions menu. Shows the menu. |
| `add_actions_menu_item(label, click, standard, shortcut, icon = null, theme = null)` | jQuery `<a>` | Item in the Actions menu. Does not show the menu. `theme` `"red"` marks a risky item. |
| `show_actions_menu()`, `hide_actions_menu()`, `clear_actions_menu()` | | Actions menu. |
| `add_dropdown_item({ label, click, standard, parent, shortcut, show_parent = true, icon, icon_right, theme, group })` | jQuery `<a>` | Generic form of the methods above. |
| `clear_user_actions()` | | Remove the non-standard items of the "..." menu. |

The menus render with `frappe.ui.Dropdown` each time they open. Changes to a returned `<a>` (`.text()`, `.toggle()`, `.addClass("disabled")`) show at the next open.

### Inner toolbar

| Method | Returns | Description |
|---|---|---|
| `add_inner_button(label, action, group, type = "default", align_right = false)` | jQuery | Button in the toolbar, or an item of the dropdown `group`. A label that exists is not added again. `type` is `default`, `secondary`, `light`, `primary`, `danger` or `ghost`. On a phone the button is also an item of the "..." menu. |
| `remove_inner_button(label, group)` | | Remove a button or a group item. `label` can be a list. A group with no items is removed. |
| `change_inner_button_type(label, group, type)` | | Change the look of a button. |
| `get_inner_group_button(label)` | jQuery | The group element. |
| `set_inner_btn_group_as_primary(label)` | | Make a group button solid. Hides the group if it has no items. |
| `add_divider_to_button_group(group)` | | Divider inside a group. |
| `add_inner_message(message)` | jQuery | Muted message (HTML) at the start of the toolbar. |
| `add_button(label, click, opts)` | jQuery | Button in the toolbar. `opts.btn_class` (`btn-primary`, `btn-danger`, ...), `opts.btn_size` (`btn-xs`), `opts.icon`. |
| `add_custom_button_group(label, icon, parent)` | jQuery `<ul>` | Dropdown button. Add items with `add_custom_menu_item(<returned ul>, ...)`. |
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

`new frappe.ui.FileUploader(opts)` opens the upload dialog. With `wrapper`, it renders inside that element without a dialog. Files go to `/api/method/upload_file` (whitelisted, `POST`), in chunks for files larger than `frappe.boot.file_chunk_size` (default 25 MB). Each upload makes a `File` record.

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
| `allow_toggle_private` | `true` | Show the private and public toggle. Off if the user cannot upload public files. |
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
| `frappe.ui.FileUploader.UploadOptions` | Static list of extra upload sources: `{ label, icon, action({ dialog, uploader, doctype, docname, fieldname }) }`. `icon` is SVG markup or a component. |

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
| `set_options(options)` | Merge options. Returns the instance. |

```js
const capture = new frappe.ui.Capture({ title: __("Photo") });
capture.submit((images) => console.log(images.length));
capture.show();
```

## frappe.ui.Scanner

`new frappe.ui.Scanner(options)` scans barcodes and QR codes with the camera. It loads `html5-qrcode` on demand and needs HTTPS.

| Option | Description |
|---|---|
| `dialog` | `true` opens a "Scan QRCode" dialog and starts scanning. |
| `container` | Element to scan in, without a dialog. Call `scan()` then. |
| `on_scan(result)` | Runs on a scan. `result.decodedText` is the text. |
| `multiple` | Default `false`. `false` stops after the first scan and closes the dialog. |
| `allow_keep_open` | Show a "keep open" toggle. Default is `true` with `dialog`. |
| `minimizable`, `primary_action_label`, `primary_action` | Passed to the dialog. |

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
| `toolbar` | | Node buttons: `{ label, icon, click(node), condition(node), danger }`. |
| `get_label(node)` | | Return the node label. |
| `on_click(node)`, `on_render(node)`, `on_node_render(node, deep)` | | Node hooks. |
| `use_row_actions` | `false` | Full-width rows with hover actions and a context menu. |
| `row_style` | `false` | Row look without actions. |
| `with_skeleton` | `1` | Show a loading skeleton. |
| `icon_set` | chevrons | `{ open, closed, leaf }` HTML. |

Methods: `refresh()` (reload the parent of the selected node), `load_children(node, deep = false)` (Promise), `expand_node(node, click = true)`, `toggle_node(node)`, `get_selected_node()`, `set_selected_node(node)`, `filter_nodes(txt)` (returns the match count). Properties: `root_node`, `nodes`.

## Charts

`frappe.ui.chart(el, config)` draws a chart into `el` and returns a handle at once. The chart is a frappe-ui chart, drawn by the `frappe.chart` island inside a shadow root. If System Settings has **Use Classic Charts** (`use_classic_charts`) on, a frappe-charts config is drawn with Frappe Charts and the handle is the Frappe Charts object.

```js
const chart = frappe.ui.chart(page.main.find(".sample-chart")[0], {
	type: "bar",
	data: [
		{ month: "Jan", total: 5 },
		{ month: "Feb", total: 7 },
	],
	x: "month",
	y: "total",
	height: 240,
});
await chart.ready;
chart.update({ type: "line", data: rows, x: "month", y: "total" });
chart.destroy();
```

| Config key | Description |
|---|---|
| `type` | `bar`, `line`, `area`, `donut`, `funnel`, `heatmap`, `calendar-heatmap`, `percentage-bar`, `sankey`, `scatter` or `number`. |
| `data` | List of rows. A plain object with `labels` and `datasets` is a Frappe Charts config, and is converted. |
| `x`, `y`, `y2` | Column names for axis charts. `y` can be a list. |
| `height` | Height in pixels. |
| `fieldtype`, `options`, `precision`, `currency` | Format values as desk formats a field of this type. A number with no type is a `Float`. |

Other keys are props of the frappe-ui chart component of that type.

| Name | Description |
|---|---|
| `frappe.ui.chart(el, config)` | Returns `{ update(config), destroy(), ready }`. A converted Frappe Charts config also has `data` and `draw()`. `update` of such a handle takes a full config or a `data` object. |
| `frappe.ui.chart.is_classic(config)` | `true` if this config is drawn with Frappe Charts. |
| `frappe.ui.chart.with_classic_palette(config)` | Add the theme palette when the chart is classic and has no `colors`. |

## Espresso components

These functions make the desk design system elements. Most come in three forms: a factory that returns a jQuery element (`frappe.ui.button(opts)`), an `.html(opts)` form that returns a string, and for stateful ones a class (`frappe.ui.Dropdown`). Labels and titles are escaped unless the table says HTML. Icon names are Lucide names.

Common keys: `css_class` (extra classes) and `attrs` (extra attributes, `{ name: value }`).

### Static elements

| Function | Main options | Description |
|---|---|---|
| `frappe.ui.button(opts)`, `.html(opts)` | `label`, `loading_label`, `variant` (`solid`, `subtle`, `outline`, `ghost`, default `subtle`), `size` (`xs`, `sm`, `md`, `lg`, default `sm`), `theme` (`gray`, `red`), `icon`, `icon_left`, `icon_right`, `title`, `tooltip`, `href`, `disabled`, `loading`, `type` (default `button`), `onclick` | Button. If `onclick` returns a Promise, the button is busy until it settles. With `href`, renders an `<a>`. |
| `frappe.ui.button.dress(el, opts)` | same as `button` | Turn an existing element into a button, or change its label, variant or icon. |
| `frappe.ui.badge(opts)`, `.html(opts)` | `label`, `theme` (`gray`, `blue`, `green`, `amber`, `red`, `violet`, `orange`), `size` (`sm`, `md`, `lg`), `variant`, `icon`, `icon_right`, `title` | Badge. |
| `frappe.ui.alert(opts)`, `.html(opts)` | `title`, `description`, `theme` (`yellow`, `blue`, `red`, `green`, `amber`), `variant` (`subtle`, `outline`), `icon`, `primary_action`, `secondary_action`, `footer`, `dismissible`, `on_dismiss` | Inline alert box. Actions take `frappe.ui.button` options. |
| `frappe.ui.avatar(opts)`, `.html(opts)` | `image`, `label`, `size` (`xs` to `3xl`), `shape` (`circle`, `square`), `theme` (`auto` picks by label), `indicator`, `title` | Avatar. Falls back to the first letter. |
| `frappe.ui.avatar_group(opts)`, `.html(opts)` | `avatars`, `max` (3), `size`, `shape`, `theme`, `onclick`, `hover_card`, `add` | Overlapping avatars with a "+N" circle. |
| `frappe.ui.breadcrumbs(opts)`, `.html(opts)` | `items`: `{ label, href, onclick, prefix, suffix, title }` | Breadcrumb trail. |
| `frappe.ui.divider(opts)`, `.html(opts)` | `orientation`, `flex_item`, `action`, `position` | Line, with an optional button on it. |
| `frappe.ui.skeleton(opts)`, `.html(opts)` | `width`, `height` | Loading placeholder. |
| `frappe.ui.empty_state(opts)`, `.html(opts)` | `icon`, `title`, `description`, `actions` (`{ label, variant, theme, icon, onclick, href }`) | Empty state block. |
| `frappe.ui.panel_header({ title, closable = true, on_close })` | | Returns `{ $header, $title, $items, $actions }`. |

### Toasts

`frappe.ui.toast(opts)` shows a toast and returns `{ id, $el, dismiss(), update(opts) }`.

| Option | Default | Description |
|---|---|---|
| `message` | | Text. |
| `description` | | Second line. |
| `type` | `"message"` | `message`, `info`, `success`, `warning` or `error`. |
| `duration` | `5000` | Milliseconds. `0` keeps the toast until closed. Hover pauses the timer. |
| `icon`, `icon_class` | | Icon and its classes. |
| `closable` | `true` | Close button. |
| `action` | | Button: `{ label, onclick }`. |
| `id` | random | A toast with the same `id` updates the open one. |

`frappe.ui.toast.promise(promise, { loading, success, error, duration })` shows a spinner toast until `promise` settles, then a success or error toast. `success` and `error` can be functions of the result. It returns `promise`.

`frappe.show_alert` also shows a toast. See [frappe (JavaScript)](frappe.md).

### Menus and overlays

Menu items have these keys: `label`, `icon`, `icon_right`, `image`, `letter`, `description`, `theme` (`gray`, `red`), `selected`, `disabled`, `shortcut` (display only), `onclick(e)`, `href`, `target`, `condition()` (checked at each open), `css_class`, `submenu` (list, or a function that returns a list or a Promise). A group is `{ group, hide_label, options }`. `options` can be a function, called at each open, that returns the items or a Promise.

| Class (factory) | Options | Methods |
|---|---|---|
| `frappe.ui.Dropdown` (`frappe.ui.dropdown(opts)` returns the trigger) | `trigger` or `button` (button options), `options`, `side` (`bottom`), `align` (`start`), `offset` (4), `empty_text`, `on_open`, `on_close(reason)`, `sheet_on_mobile` (`false`), `sheet_title()` | `open({ motion, focus })`, `close(reason)`, `toggle()`, `set_options(options)`, `destroy()`, `is_open` |
| `frappe.ui.SegmentedButton` (`frappe.ui.segmented_button(opts)` returns the wrapper) | `label`, `icon`, `variant`, `size`, `theme`, `onclick`, `options`, `menu_title`, `align` (`end`), `sheet_on_mobile` | `set_options(options)`, `destroy()` |
| `frappe.ui.ContextMenu` | `target`, `options`, `empty_text`, `on_open(event)`, `on_close(reason)` | `open_at(x, y)`, `close(reason)`, `set_options(options)`, `destroy()`, `is_open` |
| `frappe.ui.Popover` (`frappe.ui.popover(opts)` returns the trigger) | `trigger` or `button`, `content` (element, function or text), `side`, `align`, `offset`, `css_class`, `on_open`, `on_close(reason)` | `open()`, `close(reason)`, `toggle()`, `destroy()`, `is_open` |
| `frappe.ui.HoverCard(trigger, opts)` (`frappe.ui.hover_card(trigger, opts)`) | `content`, `side`, `align` (`center`), `offset`, `open_delay` (700), `close_delay` (300), `css_class`, `on_open`, `on_close` | `open()`, `close()`, `destroy()`, `is_open` |
| `frappe.ui.Tooltip(trigger, opts)` (`frappe.ui.tooltip(trigger, opts)`) | `text`, `only_on_overflow`, `shortcut`, `side` (`top`), `align` (`center`), `text_align`, `delay` (500), `offset`, `class` | `show()`, `hide()`, `set_text(text)`, `destroy()` |
| `frappe.ui.Tooltip.delegate(root, selector, opts)` | same as Tooltip | Tooltips for all matches in `root`. Returns a function that stops it. |

A factory that returns the trigger stores the instance on it: `.data("es-dropdown")`, `.data("es-popover")`, `.data("es-tooltip")`, `.data("es-hover-card")`.

### Inputs and navigation

| Class (factory) | Options | Methods |
|---|---|---|
| `frappe.ui.Combobox` (`frappe.ui.combobox(opts)` returns the trigger) | `options` (rows `{ label, value, description, icon, image, avatar, badge, disabled }`, groups, or a function `(query, { start })`), `page_size`, `value`, `placeholder`, `search_placeholder`, `filterable` (`true`), `hide_search`, `clear_button` (`true`), `tab_selects`, `disabled`, `filters`, `footer`, `chevron`, `value_input`, `open_on_focus`, `arrow_keys_open`, `actions`, `before_open`, `on_change(value, option)`, `on_open`, `on_close(reason)` | `get_value()`, `set_value(value, { label, image, avatar, silent = true })`, `clear()`, `set_options(options)`, `set_disabled(disabled)`, `focus()`, `open({ motion, query })`, `close(reason)` |
| `frappe.ui.MultiCombobox` (`frappe.ui.multi_combobox(opts)`) | Combobox options, plus `value` (list), `one_line`, `pill_href(value)`, `on_change(values, option, { added })` | `get_value()` (list), `set_value(values)`, `add_value(entry)`, `remove_value(value)`, `clear()` |
| `frappe.ui.Tabs` (`frappe.ui.tabs(opts)` returns the root, instance on `.data("es-tabs")`) | `tabs` (`{ label, icon, content, disabled }`), `active` (0), `vertical`, `on_change(index, tab)` | `set_active(index, { silent })`, `get_active()`, `destroy()` |
| `frappe.ui.TabButtons` (`frappe.ui.tab_buttons(opts)`, instance on `.data("es-tab-buttons")`) | `options` (`{ label, value, icon, icon_left, icon_right, active, disabled, title, onclick, prefix, suffix }`), `label`, `value`, `type` (`subtle`, `ghost`, `underline`, `browser-tab`), `size`, `vertical`, `direction`, `on_change(value, option)` | `get_value()`, `set_value(value, { silent })` |
| `frappe.ui.Progress` (`frappe.ui.progress(opts)`, `.html(opts)`, instance on `.data("es-progress")`) | `value` (0 to 100), `size` (`sm` to `xl`), `label`, `hint` (`true` or a function), `intervals`, `interval_count` (6) | `set_value(value)`, `get_value()`, `set_label(label)` |
| `frappe.ui.Stepper` (`frappe.ui.stepper(opts)` returns `$el`) | `steps` (`{ label }`), `current` (0), `label`, `is_locked(i)`, `is_completed(i)`, `on_step_click(i)`, `on_locked_click(i)`, `compact`, `hint`, `label_position` (`right`, `left`, `top`, `bottom`) | `set_current(index)`, `next_step()`, `prev_step()`, `refresh()` |

```js
const $tabs = frappe.ui.tabs({
	tabs: [
		{ label: __("Open"), content: () => $("<div>").text(__("Open orders"))[0] },
		{ label: __("Closed"), content: __("No closed orders") },
	],
	on_change: (index) => console.log(index),
});
$tabs.appendTo(page.main);
$tabs.data("es-tabs").set_active(1);
```

### Components in lazy bundles

| Name | Bundle | Description |
|---|---|---|
| `frappe.ui.stat_card(opts)`, `frappe.ui.stat_cards({ items, layout = "grid" })` | `desk_charts.bundle.js` | Metric card: `label`, `value`, `delta` (`{ value, positive_is_good, suffix }`), `caption`, `dot`, `icon`, `loading`, `onclick`. |
| `frappe.ui.OptionalSections` (`frappe.ui.optional_sections(opts)`) | `optional_sections.bundle.js` | Sections the user adds with a button: `sections` (`{ title, content, added }`), `label`, `remove_label`, `on_add`, `on_remove`. Methods `add(i)`, `remove(i)`, `is_added(i)`, `destroy()`. |
| `frappe.ui.BottomSheet` (`frappe.ui.bottom_sheet(opts)`) | `bottom_sheet.bundle.js` | Phone sheet: `title`, `subtitle`, `header`, `content`, `options`, `footer`, `height` (`auto`, `half`, `full`), `snap_points`, `show_close`, `dismissible` (`true`), `on_open`, `on_close(reason)`. Methods `open()`, `close(reason)`, `toggle()`, `push(step)`, `pop()`, `set_title(title, subtitle)`, `set_options(options)`, `set_height(height)`, `destroy()`. |

```js
await frappe.require("desk_charts.bundle.js");
frappe.ui.stat_card({ label: __("Open Orders"), value: "12", delta: { value: 8 } }).appendTo(page.main);
```

## Notifications and sidebar

The desk makes one `frappe.ui.Notifications` in the sidebar. An app does not make another. Use these helpers:

| Function | Description |
|---|---|
| `frappe.ui.notifications.get_notification_config()` | Promise of `frappe.desk.notifications.get_notification_info`. Caches it in `frappe.ui.notifications.config`. |
| `frappe.ui.notifications.show_open_count_list(doctype)` | Open the list of `doctype` with the "open" conditions of the `notification_config` hook as filters. |

`frappe.app.sidebar` is the `frappe.ui.Sidebar` of the desk. See [Workspaces, Sidebar and Dock](../../developer-guide/desk-ui/workspaces-sidebar-dock.md).

| Method | Description |
|---|---|
| `open()`, `close()` | Expand or collapse the sidebar. Saves the state. |
| `toggle_width()` | Switch between expanded and collapsed. Bound to <kbd>Ctrl</kbd>+<kbd>/</kbd>. |
| `toggle(hide)` | Hide or show the sidebar element. |
| `refresh()` | Pick the sidebar for the route and refresh the header and dock. |
| `get_item(label)` | Sidebar item object by label. |

## Deprecated

| Name | Use instead |
|---|---|
| `frappe.Chart` | `frappe.ui.chart`. Logs a warning once. |
| `frappe.ui.RealtimeChart(element, socketEvent, maxLabelPoints = 8, data)` | `frappe.ui.chart` and `update(data)`. |
| `frappe.utils.make_chart(wrapper, options)` | `frappe.ui.chart`. |
| `page.add_help_button(txt)` | Does nothing. |
| `dialog.add_custom_button()` | Does nothing. Use `add_custom_action`. |
| `frappe.app.sidebar.update_item()`, `remove_item()` | Do nothing. |

## Pitfalls

- The `values` option is ignored by `frappe.ui.Dialog` and `frappe.ui.FieldGroup`. Call `set_values()` after `make()` (or after the dialog constructor).
- `primary_action` does not close the dialog. Call `d.hide()`.
- `set_message()` sets plain text. `set_title()` and `set_alert()` set HTML. Escape user data.
- `frappe.ui.chart` draws inside a shadow root. `el.querySelector("svg")` finds nothing. Use the handle.
- `MultiSelectDialog` with `columns` and no `get_query().query` ignores `columns`.
- `frappe.ui.Tags.removeTag(label)` updates the list and calls `onTagRemove`, but leaves the tag element. A click on the tag close button removes both.
- `frappe.ui.TagEditor` reads `frm.doctype` and `frm.docname`. Options `doctype` and `docname` alone are not enough.
- `stat_card`, `OptionalSections` and `BottomSheet` are `undefined` until you load their bundle.

## Related

- [Controls](../../developer-guide/fields/controls.md)
- [Desk Pages](../../developer-guide/desk-ui/desk-pages.md)
- [Client API](../../developer-guide/desk-ui/client-api.md)
- [Islands](../../developer-guide/desk-ui/islands.md)
- [frappe (JavaScript)](frappe.md)
- [Form (JavaScript)](form.md)
- [List and Report Views](list-and-report-views.md)
