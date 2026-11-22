---
title: Print Settings
description: Field reference for Print Settings, with the effect of each field on the print view, PDF generation, letter heads, bulk print and email attachments.
version: v16
sources:
  - frappe/printing/doctype/print_settings/print_settings.json
  - frappe/printing/doctype/print_settings/print_settings.py
  - frappe/printing/doctype/print_settings/print_settings.js
  - frappe/printing/page/print/print.js
  - frappe/printing/page/report_print/report_print.js
  - frappe/public/js/frappe/list/bulk_operations.js
  - frappe/public/js/frappe/model/model.js
  - frappe/public/js/frappe/model/meta.js
  - frappe/public/js/frappe/views/communication.js
  - frappe/public/js/print_format_builder/PrintFormatBuilder.vue
  - frappe/www/printview.py
  - frappe/templates/styles/standard.css
  - frappe/templates/print_formats/standard_macros.html
  - frappe/templates/print_format/print_format.html
  - frappe/utils/print_utils.py
  - frappe/utils/print_format.py
  - frappe/utils/print_format_generator.py
  - frappe/utils/pdf_generator/browser.py
  - frappe/utils/pdf_generator/pluto.py
  - frappe/utils/pdf.py
  - frappe/templates/print_formats/standard.html
  - frappe/printing/doctype/print_format/print_format.py
  - frappe/email/doctype/notification/notification.py
  - frappe/boot.py
  - frappe/hooks.py
---

# Print Settings

`Print Settings` is a Single DocType that sets the site defaults for printing: PDF engine, paper size, letter head, draft and cancelled prints, print style and font. Use this page to find what each field does and which code reads it. For Print Formats and the PDF engines, see [Print Formats](../../developer-guide/output/print-formats.md). For concepts that apply to all settings DocTypes, see [How settings work](how-settings-work.md).

## Print Settings

Open it at `/desk/print-settings`, or from the **Print Settings** menu item of the print view. Only `System Manager` can read and write it. The Print Format Builder also opens a **Print Settings** dialog that saves these fields with `frappe.client.set_value`, if the user has write permission.

![Print Settings form with PDF Settings and Page Settings sections](../../assets/images/user-guide/settings-print-settings.png)

The desk gets all values at boot. `load_print()` in `frappe/boot.py` adds the document as `:Print Settings` to the boot docs. Client code reads it with `frappe.model.get_doc(":Print Settings", "Print Settings")`. Server code mostly reads `frappe.get_cached_doc("Print Settings")`.

### PDF Settings

| Field | Default | Effect |
|---|---|---|
| Send Print as PDF (`send_print_as_pdf`) | `1` | `frappe.attach_print()` (`frappe/utils/print_utils.py`) makes a `.pdf` file. When off, it attaches the print as an `.html` file. This affects email attachments of documents, notifications and workflow actions. |
| Repeat Header and Footer (`repeat_header_footer`) | `1` | Repeats the letter head header and footer on each PDF page. For Jinja and standard formats, `templates/print_formats/standard.html` puts them in the `header-html` and `footer-html` blocks that the PDF engine repeats. When off, they print once, inline. Builder formats read it in `frappe/utils/print_format_generator.py`, and report PDFs in `frappe/utils/print_format.py`. The print view shows "Repeat Header and Footer is disabled" under the Letter Head field when it is off. |
| Print the PDF (`print_generated_pdf`) | `0` | The **Print** button in the print view prints the generated PDF instead of the browser page (`printit()` in `frappe/printing/page/print/print.js`). |
| PDF Generator (`pdf_generator`) | `chrome` | The site default PDF engine: `wkhtmltopdf`, `chrome` or `PlutoPrint`. The form says that `wkhtmltopdf` is deprecated and will be removed in version 17. `resolve_pdf_generator()` in `frappe/utils/print_utils.py` uses it only when the call and the Print Format give no engine. If this field is also empty, it uses `wkhtmltopdf`. It uses `chrome` instead of `PlutoPrint` when PlutoPrint is not installed or the format uses JavaScript or CSS grid. Print Format Builder formats ignore it: they keep `PlutoPrint`, `Typst` or `WeasyPrint`, or else use `chrome`. |
| Max Bulk Print Documents (`max_bulk_print_docs`) | `100` | Maximum documents in one bulk PDF (`download_multi_pdf` and `download_multi_pdf_async` in `frappe/utils/print_format.py`). If `0`, the `max_bulk_print_docs` site config key is used, then `100`. |
| Max Concurrent Bulk Exports Per User (`max_concurrent_bulk_exports`) | `5` | Maximum background bulk PDF jobs one user can have queued or running. If `0`, the `max_concurrent_bulk_exports` site config key is used, then `5`. |
| PDF Page Size (`pdf_page_size`) | `A4` | The default paper size for PDFs (`frappe/utils/pdf.py` for wkhtmltopdf, `frappe/utils/pdf_generator/browser.py` for chrome, `page_size_mm()` in `frappe/utils/print_format_generator.py`). Also the default of **Page Size** in the bulk print dialog and the report print page. The print view **Paper Size** field and a builder layout can override it for one print. Options: A0 to A9, B0 to B10, C5E, Comm10E, DLE, Executive, Folio, Ledger, Legal, Letter, Tabloid, Custom. |
| PDF Page Height (in mm) (`pdf_page_height`) | empty | Page height when the size is `Custom`. Shows only then. Builder formats use 297 mm when it is empty. |
| PDF Page Width (in mm) (`pdf_page_width`) | empty | Page width when the size is `Custom`. Shows only then. Builder formats use 210 mm when it is empty. |

Validation: when `pdf_page_size` is `Custom`, height and width must not be zero. The controller throws "Page height and width cannot be zero".

After a save with an engine other than `wkhtmltopdf`, the form shows how many Print Formats still use `wkhtmltopdf`. Its **Use ...** button calls `follow_print_settings_pdf_generator`, which clears **PDF Generator** on those formats so that they follow this setting. Only `System Manager` can call it.

### Page Settings

| Field | Default | Effect |
|---|---|---|
| Print with letterhead (`with_letterhead`) | `1` | Used when the caller does not say if a letter head is wanted: `frappe/www/printview.py`, the Print Format Builder renderer (`frappe/utils/print_format_generator.py`), the email composer, and Notification print attachments (`print_letterhead`). |
| Allow Print for Draft (`allow_print_for_draft`) | `1` | Lets users print draft documents of submittable DocTypes. Checked on the server in `get_rendered_template()` and `validate_print_for_docstatus()` in `frappe/www/printview.py`, which `attach_print` and Notification attachments also call. In the desk, `frappe.model.can_print_docstatus()` hides print for drafts, and bulk print skips them. |
| Always add "Draft" Heading for printing draft documents (`add_draft_heading`) | `1` | Adds a "Draft" heading to prints of draft submittable documents (`templates/print_formats/standard_macros.html`, `templates/print_format/print_format.html`). |
| Allow page break inside tables (`allow_page_break_inside_tables`) | `0` | When off, `templates/styles/standard.css` adds `page-break-inside: avoid` to `table td div`, so a cell does not split across pages. |
| Allow Print for Cancelled (`allow_print_for_cancelled`) | `0` | Lets users print cancelled documents. Same server and desk checks as `allow_print_for_draft`. |

The `Administrator` user can select draft and cancelled documents in bulk print, but the server check still applies.

### Print Server

| Field | Default | Effect |
|---|---|---|
| Enable Print Server (`enable_print_server`) | `0` | The print view gets a **Select Network Printer** menu item. **Print** sends the PDF to a CUPS printer from `Network Printer Settings` with `frappe.utils.print_format.print_by_server`. The server needs `pycups`. |

!!! warning
    The field has `depends_on: enable_print_server`, so it depends on itself. While it is off, the form hides it. Set it from code if you need it.

### Raw Printing

| Field | Default | Effect |
|---|---|---|
| Enable Raw Printing (`enable_raw_printing`) | `0` | Shows Print Formats with **Raw Printing** in the print view format list (`get_print_formats()` in `frappe/public/js/frappe/model/meta.js`). Adds the **Raw Printing Setting** menu item, which maps formats to printers for QZ Tray. |

### Print Style

| Field | Default | Effect |
|---|---|---|
| Print Style (`print_style`) | `Redesign` | The default `Print Style`. Its CSS is added after `templates/styles/standard.css` (`get_print_style()` in `frappe/www/printview.py`). The Print Format Builder renderer uses it when the print has no style. Boot puts the rendered CSS in `frappe.boot.print_css`. A disabled Print Style is ignored. |
| Print Style Preview (`print_style_preview`) | HTML | Shows the preview image of the selected Print Style in the form. Client side only. |

### Fonts

| Field | Default | Effect |
|---|---|---|
| Font (`font`) | `Default` | Font of HTML prints when the Print Format sets no font (`get_font()` in `frappe/www/printview.py`). Options: Default, Helvetica Neue, Arial, Helvetica, Inter, Verdana, Monospace. `Default` uses Inter and system fonts. The CSS in `frappe.boot.print_css` always uses the default font. |
| Font Size (`font_size`) | empty | In points. When set, `get_print_style()` adds `.print-format { font-size: <n>pt; }` after the Print Style CSS. Empty or `0` keeps the size of the style. |

### For developers

Read values in Python:

```python
import frappe

size = frappe.db.get_single_value("Print Settings", "pdf_page_size")  # "A4"
settings = frappe.get_cached_doc("Print Settings")
settings.with_letterhead  # 1
```

Read values in the desk:

```js
const print_settings = frappe.model.get_doc(":Print Settings", "Print Settings");
print_settings.allow_print_for_draft;
```

- `on_update` calls `frappe.clear_cache()`. Reload the desk to get the new boot values.
- A print request can override some values for one print with the `settings` argument (JSON). `get_allowed_print_settings_override()` in `frappe/www/printview.py` keeps only `pdf_page_size`, `pdf_page_height`, `pdf_page_width`, `page_orientation`, `print_font`, and the print toggles that the document returns from `get_print_settings()`. It never accepts `allow_print_for_draft` or `allow_print_for_cancelled`. Builder formats and the print style use this filter. The Jinja render path (`get_rendered_template`) does not: it merges the whole `settings` dict.
- `before_print(self, settings)` on a controller gets the Print Settings dict.
- The `pdf_generator` hook in `frappe/hooks.py` lists the engine functions. An app can add an engine. See [Print Formats](../../developer-guide/output/print-formats.md#the-pdf_generator-hook).
- `frappe.printing.doctype.print_settings.print_settings.is_print_server_enabled` is a whitelisted method that returns `enable_print_server`. Frappe code does not call it.

## Pitfalls

- A Print Format with its own **PDF Generator** wins over this setting. A format with an empty **PDF Generator** follows this setting at each print.
- A Print Format Builder format with no stored engine is saved with `chrome`. A new format that asks for `PlutoPrint` but uses JavaScript or CSS grid is saved with `chrome`.
- The `settings` argument of `/printview` and `frappe.utils.print_format.download_pdf` is merged into Print Settings before the draft and cancelled checks, for Jinja and standard formats. A caller can pass `{"allow_print_for_cancelled": 1}` to print a cancelled document when the setting is off. This was checked on this site.
- `allow_print_for_draft` and `allow_print_for_cancelled` apply only to submittable DocTypes.
- Desk users keep the old boot values until they reload the page.

## Related

- [Printing](../data-and-reports/printing.md)
- [Print Formats](../../developer-guide/output/print-formats.md)
- [How settings work](how-settings-work.md)
