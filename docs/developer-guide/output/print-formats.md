---
title: Print Formats
description: Ship Print Formats in an app, write Jinja print templates, render HTML or PDF with frappe.get_print, choose the PDF engine and add one with the pdf_generator hook.
version: v16
sources:
  - frappe/utils/print_utils.py
  - frappe/utils/pdf.py
  - frappe/utils/print_format.py
  - frappe/utils/print_format_generator.py
  - frappe/utils/pdf_generator/pluto.py
  - frappe/utils/pdf_generator/browser.py
  - frappe/www/printview.py
  - frappe/printing/doctype/print_format/print_format.py
  - frappe/printing/doctype/print_format/print_format.json
  - frappe/printing/doctype/print_settings/print_settings.json
  - frappe/utils/weasyprint.py
  - frappe/utils/typst_emitter.py
  - frappe/printing/doctype/print_format/classic_converter.py
  - frappe/patches/v16_0/set_weasyprint_generator_for_beta_formats.py
  - frappe/website/page_renderers/print_page.py
  - frappe/hooks.py
---

# Print Formats

A Print Format decides how a document looks when printed, sent as a PDF, or attached to an email. Each `Print Format` document is linked to a DocType (`doc_type`) or to a report (`print_format_for = "Report"`). Users design most formats in the desk (see [Printing (user guide)](../../user-guide/data-and-reports/printing.md)). As a developer you ship formats in your app, render them from code, and extend the PDF pipeline.

## Kinds of Print Format

| Kind | Fields | Rendered by |
|---|---|---|
| Standard | No Print Format, or `"Standard"` | Layout of the DocType form, `frappe/www/printview.py` |
| Editor format | `print_format_builder_beta = 1`, `format_data` (layout JSON) | `PrintFormatGenerator` (`frappe/utils/print_format_generator.py`) |
| Custom Jinja | `custom_format = 1`, `print_format_type = "Jinja"`, `html` | `get_rendered_template()` in `frappe/www/printview.py` |
| Raw printing | `custom_format = 1`, `raw_printing = 1`, `raw_commands` | Sent to a printer as raw commands |

The classic Print Format Builder cannot create new formats. `create_custom_format()` in `frappe/printing/doctype/print_format/print_format.py` makes every new builder format with `print_format_builder_beta = 1`. Older formats that still have a classic layout keep rendering with the standard template. Prefer editor formats. Use a custom Jinja format only when the layout cannot be built from editor blocks.

## Print Formats in an app

Set `standard` to "Yes" and save in developer mode. `export_doc()` writes the JSON to the module folder. For a Jinja format, put the template in an `.html` file with the same name. `get_print_format()` reads that file before the `html` field.

```text
sample_app/sample_app/selling/print_format/sample_order_slip/
├── sample_order_slip.json
└── sample_order_slip.html
```

```jinja
{# sample_app/sample_app/selling/print_format/sample_order_slip/sample_order_slip.html #}
<h2>{{ doc.name }}</h2>
<p>{{ doc.customer_name }}, {{ doc.get_formatted("order_date") }}</p>
<table class="table table-bordered">
	{% for row in doc.items %}
	<tr>
		<td>{{ row.item }}</td>
		<td class="text-right">{{ row.get_formatted("qty") }}</td>
		<td class="text-right">{{ row.get_formatted("amount", doc) }}</td>
	</tr>
	{% endfor %}
</table>
<p class="text-right"><b>{{ _("Total") }}: {{ doc.get_formatted("total") }}</b></p>
```

Without developer mode, a standard Print Format cannot be saved ("Standard Print Format cannot be updated").

### Template context

`get_rendered_template()` renders the template with these variables:

| Name | Value |
|---|---|
| `doc` | The document. Use `doc.get_formatted("<fieldname>")` for values in the user's format. |
| `meta` | `frappe.get_meta(doc.doctype)` |
| `layout` | The form layout in sections and columns (`make_layout`) |
| `no_letterhead` | `1` when the letter head is off |
| `letter_head` | Rendered letter head HTML |
| `footer` | Rendered letter head footer HTML |
| `print_settings` | Print Settings as a dict |
| `trigger_print` | `1` when the page opens the print dialog |

The Jinja globals of [Website](../website/website.md) are also available, for example `frappe.db.get_value` and `_`. Methods and filters from the `jinja` hook work too. `validate_template` checks the Jinja when you save.

### Before print

Before it renders, Frappe sets `doc.flags.in_print` and calls the controller method `before_print(settings)`. Use it to set values that only the print shows.

```python
# sample_app/sample_app/selling/doctype/sample_order/sample_order.py
class SampleOrder(Document):
	def before_print(self, settings=None):
		self.print_heading = f"Order {self.name}"
```

## Render from code

`frappe.get_print()` returns HTML, or PDF bytes with `as_pdf=True`. It checks print permission for the current user.

```python
html = frappe.get_print("Sample Order", "SO-00001")
pdf = frappe.get_print("Sample Order", "SO-00001", print_format="Sample Order Slip", as_pdf=True)

with open("/tmp/so-00001.pdf", "wb") as f:
	f.write(pdf)
```

| Parameter | Default | Description |
|---|---|---|
| `doctype`, `name` | `None` | The document to print. |
| `print_format` | `None` | Print Format name. Empty uses the DocType default, then "Standard". |
| `style` | `None` | Print Style name. Empty uses Print Settings `print_style`. |
| `as_pdf` | `False` | Return PDF bytes instead of HTML. |
| `doc` | `None` | A document object to print instead of loading it. |
| `no_letterhead` | `0` | Leave out the letter head. |
| `letterhead` | `None` | Letter Head name to use. |
| `password` | `None` | Encrypt the PDF with this password. |
| `pdf_options` | `None` | Options for the PDF engine. |
| `pdf_generator` | `None` | `"wkhtmltopdf"`, `"chrome"`, `"PlutoPrint"`, `"Typst"` or `"WeasyPrint"`. Overrides the format and Print Settings for HTML formats. |
| `output` | `None` | A `pypdf.PdfWriter`. Pages are added to it. |

`frappe.attach_print(doctype, name, print_format=None, ...)` returns `{"fname": ..., "fcontent": ...}` for an email attachment. It returns a PDF when Print Settings `send_print_as_pdf` is on, else HTML. It skips the print permission check (`frappe.local.flags.ignore_print_permissions`).

`frappe.translate.print_language("de")` is a context manager that prints in another language.

### URLs

| URL | Returns |
|---|---|
| `/printview?doctype=Sample Order&name=SO-00001&format=Sample Order Slip` | HTML print view |
| `/Sample Order/SO-00001` | HTML print view (`PrintPage` renderer) |
| `/api/method/frappe.utils.print_format.download_pdf?doctype=Sample Order&name=SO-00001&format=...` | PDF download |
| `/api/method/frappe.utils.print_format.download_multi_pdf` | One PDF for many documents |

`download_pdf` takes `doctype`, `name`, `format`, `no_letterhead`, `language`, `letterhead`, `pdf_generator`, `style` and `settings`.

## PDF engines

`pdf_generator` on the Print Format picks the engine. Print Settings `pdf_generator` is the default for formats that do not set one. Print Settings offers `wkhtmltopdf`, `chrome` and `PlutoPrint`. Its field default is `chrome`.

| Value | Description |
|---|---|
| `wkhtmltopdf` | Renders HTML with the `wkhtmltopdf` binary (`frappe.utils.pdf.get_pdf()`). Install it on the server. The fallback when nothing else is set. |
| `chrome` | Renders HTML with a headless Chromium (`frappe/utils/pdf_generator/`). `bench setup-chrome` downloads it. |
| `PlutoPrint` | Renders HTML without a browser process. Falls back to `chrome` when PlutoPrint is not installed or the format has JavaScript or CSS grid (`plutoprint_blockers`). `get_plutoprint_pdf()` also uses `chrome` for a page range. |
| `Typst` | Editor formats only, without custom CSS in the Style box. `typst_blockers()` in `frappe/utils/typst_emitter.py` lists what stops it. |
| `WeasyPrint` | Deprecated. Kept only for editor formats that already use it. A new format cannot select it. |

`resolve_pdf_generator()` in `frappe/utils/print_utils.py` picks the engine:

1. For an editor format, it keeps `Typst`, `WeasyPrint` or `PlutoPrint` (when PlutoPrint can render it). Otherwise it uses `chrome`.
2. For other formats, it uses the `pdf_generator` argument, then the format's `pdf_generator`, then Print Settings `pdf_generator`, then `wkhtmltopdf`.
3. It changes `PlutoPrint` to `chrome` when PlutoPrint cannot render the format.

!!! note "Version"
    The patch `frappe/patches/v16_0/set_weasyprint_generator_for_beta_formats.py` sets existing editor formats to `WeasyPrint`, so they render as before. WeasyPrint shows a deprecation warning. Switch these formats to `chrome`. Later versions remove WeasyPrint and wkhtmltopdf.

### The pdf_generator hook

For an HTML format, `get_print(as_pdf=True)` renders the HTML. If the engine is not `wkhtmltopdf`, it calls each function in the `pdf_generator` hook. The first one that returns bytes wins. If none returns a value, Frappe uses `frappe.utils.pdf.get_pdf()`, which runs wkhtmltopdf.

```python
# frappe/hooks.py
pdf_generator = [
	"frappe.utils.pdf.get_chrome_pdf",
	"frappe.utils.pdf.get_plutoprint_pdf",
	"frappe.utils.print_format_generator.get_typst_pdf",
	"frappe.utils.weasyprint.get_weasyprint_pdf",
]
```

To add an engine:

1. Add your engine name to the options of the `pdf_generator` field of Print Format and Print Settings, for example with a Property Setter in your fixtures.
2. Add a function to the `pdf_generator` hook.
3. In the function, return `None` when `pdf_generator` is not your engine.

```python
# sample_app/hooks.py
pdf_generator = ["sample_app.pdf.get_sample_pdf"]
```

```python
# sample_app/pdf.py
def get_sample_pdf(print_format, html, options, output, pdf_generator=None):
	if pdf_generator != "Sample Engine":
		return None
	return render_with_sample_engine(html, options)  # must return PDF bytes
```

### Other print hooks

| Hook | Called with | Use |
|---|---|---|
| `on_print_pdf` | `doctype`, `name`, `print_format` | Run code before a PDF is made, for example to log it. |
| `get_print_format_template` | `jenv`, `print_format` | Return a compiled Jinja template to use instead of the format's HTML. The last app wins. |
| `pdf_body_html` | `jenv`, `template`, `print_format`, `args` | Render the body HTML. Frappe sets `frappe.utils.pdf.pdf_body_html`. The last app wins. |
| `pdf_header_html`, `pdf_footer_html` | `soup`, `head`, `content`, `styles`, `html_id`, `css` | Build the repeating header and footer. |

## Letter heads

`get_letter_head()` uses, in order: the `letterhead` argument, the document's `letter_head` field, then the `Letter Head` with `is_default = 1`. When neither argument is passed, Print Settings `with_letterhead` decides. Frappe renders the letter head `content` and `footer` with Jinja and `{"doc": doc.as_dict()}`, so you can use `{{ doc.customer_name }}` in a letter head.

## Print styles

Frappe renders `templates/styles/standard.css`, then adds the `css` of the selected `Print Style`, then the Print Settings `font_size`. A Jinja format can add CSS in its `css` field. Frappe ships the styles in `frappe/printing/print_style/`.

## Raw printing

A raw format sends printer commands (for example ESC/POS or ZPL) instead of HTML. Set `raw_printing` and write the commands in `raw_commands` with Jinja. `frappe.www.printview.get_rendered_raw_commands(doc, name, print_format)` returns the rendered commands. The desk sends them to the printer through QZ Tray when Print Settings `enable_raw_printing` is on.

## Pitfalls

- A draft document of a submittable DocType prints only when Print Settings `allow_print_for_draft` is on. A cancelled one needs `allow_print_for_cancelled`.
- Fields with `print_hide` and fields above the user's permission level are not printed.
- Use `get_formatted()` instead of raw values. Raw values do not follow the user's date, number and currency format.
- Render every format and look at the PDF before you ship it. wkhtmltopdf, PlutoPrint and Chromium can break pages differently.
- `wkhtmltopdf` does not use the `print_font` print setting. `get_print()` removes it.

## Related

- [Website](../website/website.md)
- [Email and Notifications](email-and-notifications.md)
- [Hooks](../building-apps/hooks.md)
- [Translations](../building-apps/translations.md)
- [Printing (user guide)](../../user-guide/data-and-reports/printing.md)
