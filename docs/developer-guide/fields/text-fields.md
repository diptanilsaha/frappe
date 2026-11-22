---
title: Text Fields
description: How Data, Small Text, Text, Long Text, Text Editor, Markdown Editor, HTML Editor, Code, JSON, Password and Read Only fields store values, behave in the desk and validate on save.
version: v16
sources:
  - frappe/public/js/frappe/form/controls/data.js
  - frappe/public/js/frappe/form/controls/text.js
  - frappe/public/js/frappe/form/controls/text_editor.js
  - frappe/public/js/frappe/form/controls/markdown_editor.js
  - frappe/public/js/frappe/form/controls/html_editor.js
  - frappe/public/js/frappe/form/controls/code.js
  - frappe/public/js/frappe/form/controls/json.js
  - frappe/public/js/frappe/form/controls/password.js
  - frappe/public/js/frappe/form/controls/base_control.js
  - frappe/public/js/frappe/form/controls/base_input.js
  - frappe/public/js/frappe/form/formatters.js
  - frappe/model/document.py
  - frappe/model/meta.py
  - frappe/utils/__init__.py
  - frappe/utils/data.py
  - frappe/utils/formatters.py
  - frappe/utils/password.py
  - frappe/utils/html_utils.py
  - frappe/client.py
  - frappe/core/doctype/file/utils.py
  - frappe/core/doctype/doctype/doctype.py
  - frappe/core/doctype/docfield/docfield.json
  - frappe/core/doctype/user/user.json
---

# Text Fields

Use these fieldtypes for strings: short values, multi-line text, rich text, source code, JSON and secrets. Each one has a column in the parent table. In Python and in REST JSON, the value is a `str` or `None`. For column types and `options`, see [Field Types](../../reference/data-model/field-types.md). For properties that apply to all fields, see [Field Properties](field-properties.md).

## Summary

| Fieldtype | Desk control | Server on save |
|---|---|---|
| `Data` | One-line input | Length check. Validates `Email`, `Name`, `Phone`, `URL` and `IBAN` options. |
| `Small Text` | Text area, 150px high | HTML sanitized |
| `Text` | Text area, 300px high | HTML sanitized |
| `Long Text` | Same control as `Text` | HTML sanitized |
| `Text Editor` | Quill rich text editor | HTML sanitized. Inline images saved as `File`. |
| `Markdown Editor` | Ace editor in Markdown mode, with preview | HTML sanitized |
| `HTML Editor` | Ace editor in HTML mode, with preview | HTML sanitized |
| `Code` | Ace editor | Not sanitized. `Python` and `PythonExpression` are parsed. |
| `JSON` | Ace editor in JSON mode | Not sanitized. A `dict` is stored as JSON text. A `list` is rejected. |
| `Password` | Password input | Value encrypted in `__Auth`. Column keeps `*` characters. |
| `Read Only` | Display only | Length check. Non-string values become `str`. |

## Sanitization on save

`_sanitize_content()` in `frappe/model/document.py` runs on every insert and save. It changes each string value that contains `<` or `>` with `sanitize_html()`. Script tags and event attributes such as `onerror` are removed. Safe tags such as `<b>` stay.

The server skips a field when one of these is true:

- The field has `ignore_xss_filter = 1`.
- The fieldtype is `Code`, `JSON`, `Attach`, `Attach Image` or `Barcode`.
- The fieldtype is `Data`, `Small Text` or `Text` with options `Email`.
- The document is cancelled, or submitted and the field has no `allow_on_submit`.
- The whole value is valid JSON text, for example `{"a": "<b>x</b>"}`. `sanitize_html()` returns JSON unchanged.

```python
doc = frappe.get_doc({"doctype": "Sample Item", "item_name": "A", "description": "Hi <script>x</script><b>B</b>"})
doc.insert()
doc.description  # 'Hi <b>B</b>'
```

## `Data`

- **Stores**: a string up to `length` characters, 140 by default.
- **Desk**: a text input with `maxlength` set to `length`. A paste longer than the limit shows a "Data Clipped" message. The change event runs 500 ms after typing stops, except in a grid. If the DocType uses `autoname: field:<fieldname>`, a new document shows "already exists" under the field when the name is taken.
- **Properties**: `length`, `options`, `unique`, `translatable`, `in_global_search`, `alignment`, `placeholder`, `mask`.
- **Server**: `_validate_length()` throws `frappe.CharacterLengthExceededError` when the value is longer than `length`. A list value throws "cannot be a list". Single DocTypes skip the length check.
- **JS**: `frappe.ui.form.ControlData`. The `with_copy_button` df key adds a copy button. Set it in a dialog or field group.

### `Data` options

Set `options` to one of these values. Any other value shows an "Invalid Data Field" message when you save the DocType, but the save continues.

| Options | Desk | Server check on save |
|---|---|---|
| `Email` | Red border if an address is not valid. Accepts a list separated by commas. | `validate_email_address()` on each address. Throws `frappe.InvalidEmailAddressError`. |
| `Name` | Red border if not a valid person name | `validate_name()`: letters, digits, `'` and `-`, words split by one space |
| `Phone` | Red border if not valid | `validate_phone_number()`: 1 to 20 of digits, spaces and `+ _ - , . * # ( )` |
| `URL` | An "Open Link" button shows for a valid URL | `validate_url()`. Needs a scheme, or a path that starts with `/`. Any scheme passes, also `javascript:`. |
| `Barcode` | A "Scan" button opens the camera scanner | None |
| `IBAN` | Shows the value in groups of 4. Removes spaces before it sets the value. | `validate_iban()` checks the checksum |

The desk only marks the field red. The server rejects the value when you save. The `Email` check is skipped when the owner and the value are both standard users, such as `Administrator`.

```json
{"fieldname": "contact_email", "fieldtype": "Data", "label": "Contact Email", "options": "Email"}
```

## `Small Text`, `Text` and `Long Text`

- **Stores**: a string. `Long Text` has a larger column on MariaDB.
- **Desk**: a `<textarea>`. `Small Text` is 150px high. `Text` and `Long Text` are 300px high. The read-only view keeps line breaks and indentation. In list view and grid, new lines show as `<br>` if the value has no `<p>`, `<div>`, `<br>` or `<table>` tag.
- **Properties**: `max_height` (for example `200px`) limits the height. The DocType save throws if the unit is not `px` or `em`. `translatable` and `in_global_search` apply. `search_index` is not allowed.
- **JS**: `ControlSmallText` and `ControlText`. `ControlLongText` is the same class as `ControlText`.

## `Text Editor`

- **Stores**: HTML. The control wraps the content in `<div class="ql-editor read-mode">...</div>`.
- **Desk**: a Quill editor with a toolbar: headings, font size, bold, italic, underline, strike, colors, quote, code block, text direction, link, image, lists, alignment, indent and tables. In a grid row, the toolbar is hidden. Pasted or dropped images become inline `data:` URLs. The editor removes `<script>` and `<style>` before it sets the value.
- **Properties**: `max_height`, `placeholder` (default "Type something..."), `translatable`, `in_global_search`, `ignore_xss_filter`.
- **Server**: `_extract_images_from_text_editor()` saves each `data:` image as a private `File` attached to the document. It then puts the file URL in the `src`. Files are public if the DocType has `make_attachments_public`. URLs in the text do not become links.
- **JS**: `frappe.ui.form.ControlTextEditor`. Set `enable_mentions: 1` on the df to show `@` user mentions. Set `get_toolbar_options` on the df to a function that returns a Quill toolbar array.
- **Format**: Python `frappe.format_value()` returns `<div class='ql-snow'>...</div>`.

## `Markdown Editor` and `HTML Editor`

- **Stores**: Markdown or HTML text.
- **Desk**: an Ace editor with line wrap and a "Preview" button. The button changes to "Edit" while the preview shows. The HTML preview removes `<script>` and `<style>`. Drop an image on the editor to upload it to `Home/Attachments`. The control inserts `![](/files/sample.png)` at the cursor. Other file types are rejected.
- **Server**: sanitized like other text, unless the value contains `<!-- markdown -->` and no HTML tags.
- **Format**: Python `frappe.format_value()` converts `Markdown Editor` to HTML with `frappe.utils.markdown()`. The desk list view shows the raw text.
- **JS**: `ControlMarkdownEditor` and `ControlHTMLEditor`. Both extend `ControlCode`. `ControlHTMLEditor` always sets `options` to `HTML`.

!!! note
    The `HTML Editor` control also inserts the Markdown syntax `![](/files/sample.png)` for a dropped image, not an `<img>` tag.

## `Code`

- **Stores**: the source text, not sanitized.
- **Desk**: an Ace editor, 300px high, with an "Expand" button that changes the height to 600px. In read mode, a copy button shows. The keyboard mode comes from the `code_editor_type` field of the user: `vscode` (default), `vim` or `emacs`. In list view and grid, the value shows escaped inside `<pre>`.
- **Properties**: `options` sets the language. See [Field Types](../../reference/data-model/field-types.md) for the list. An unknown value logs a console warning and uses plain text. `max_height` applies. `search_index` is not allowed.
- **Server**: `_validate_code_fields()` parses `Python` as a script and `PythonExpression` as one expression. A syntax error throws "Invalid python code on line N".
- **JS**: `frappe.ui.form.ControlCode`. The df keys `max_lines`, `min_lines` and `wrap` set Ace options. With `max_lines`, `min_lines` or `max_height`, the "Expand" button does not show. To add autocompletion, set `autocompletions` to a list of strings or `{value, caption, meta, score}` objects, or to a function that returns one:

```js
// sample_app/sample_module/doctype/sample_item/sample_item.js
// "script" is a Code field. Server Script uses the same call for its own script field.
frappe.ui.form.on("Sample Item", {
	refresh(frm) {
		frm.set_df_property("script", "autocompletions", ["frappe.db.get_value", "frappe.get_doc"]);
	},
});
```

## `JSON`

- **Stores**: JSON text. You can set a `dict` in Python. `get_valid_dict()` stores it as compact JSON (`{"a":1}`). A `list` raises "Value for ... cannot be a list". Set `json.dumps([1, 2])` instead.
- **Python**: after a reload, `frappe.get_doc` and `frappe.db.get_value` return a `str`, not a `dict`. Use `frappe.parse_json(doc.data)`. REST JSON also returns the string.
- **Desk**: an Ace editor in JSON mode. The control does not check that the JSON is valid.
- **Server**: not sanitized. Frappe does not parse the value. On MariaDB, the `json` column rejects text that is not valid JSON with a database error.

```python
doc.settings = {"a": 1}
doc.save()
doc.reload()
doc.settings  # '{"a":1}'
frappe.parse_json(doc.settings).a  # 1
```

## `Password`

- **Stores**: the column holds one `*` for each character. The real value is encrypted with the site encryption key in the `__Auth` table.
- **Desk**: an `<input type="password">` with an eye button to show the text. While you type, a strength bar shows. It calls `frappe.core.doctype.user.user.test_password_strength`. Call `disable_password_checks()` on the control to turn the bar off.
- **Server**: `_save_passwords()` runs on save. A new value is encrypted, then the field is set to `*` characters. A value of only `*` characters keeps the stored password. An empty value removes the stored password. Set `doc.flags.ignore_save_passwords` to `True`, or to a list of fieldnames, to skip this.
- **Read the value**: in Python, call `doc.get_password("fieldname")`. Over HTTP, call `frappe.client.get_password` with `doctype`, `name` and `fieldname`. Only a System Manager can call it. `frappe.get_doc`, `frappe.db.get_value` and REST return the `*` characters.

```python
# "api_secret" is a Password field of the DocType
doc = frappe.get_doc("Sample Item", "ITEM-0001")
api_secret = doc.get_password("api_secret", raise_exception=False)
```

## `Read Only`

- **Stores**: a string up to `length` characters, 140 by default. Fill it with `fetch_from` or from code.
- **Desk**: always shows as text, never as an input, also for a new document.
- **Server**: `get_valid_dict()` converts a value that is not a `str` to `str` before it writes. After `insert()`, the object in memory keeps the value you set. A reload returns the string. `_validate_length()` applies. `unique` is allowed.
- **JS**: `frappe.ui.form.ControlReadOnly` is the same class as `ControlData`.

## Pitfalls

- The desk marks a bad `Email` or `URL` red but still sets the value. Only the server blocks it.
- A `URL` value with a `javascript:` scheme passes the server check. The list view and grid show it as a link. Check the scheme in `validate` when users enter URLs.
- A text value that is valid JSON is not sanitized.
- Text fields keep safe HTML. Do not render them with `|safe` in Jinja or `innerHTML` unless the field must hold HTML.
- `ignore_xss_filter` turns off sanitization for every writer of the field, also REST clients.
- A `JSON` value comes back as a string. Parse it before you use it.
- Never log or return a `Password` value from `get_password()` in an API response.

## Related

- [Field Types](../../reference/data-model/field-types.md)
- [Field Properties](field-properties.md)
- [DocField Properties](../../reference/data-model/docfield-properties.md)
- [Controls](controls.md)
- [Number Fields](number-fields.md)
- [Security](../security/security.md)
