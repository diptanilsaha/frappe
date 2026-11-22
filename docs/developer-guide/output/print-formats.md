---
title: Print Formats
description: Ship Print Formats in an app, write Jinja print templates, render HTML or PDF with frappe.get_print, and know how wkhtmltopdf, WeasyPrint and the pdf_generator hook make PDFs.
version: v15
sources:
  - frappe/utils/print_utils.py
  - frappe/utils/pdf.py
  - frappe/utils/print_format.py
  - frappe/utils/weasyprint.py
  - frappe/www/printview.py
  - frappe/printing/doctype/print_format/print_format.py
  - frappe/printing/doctype/print_format/print_format.json
  - frappe/printing/doctype/print_settings/print_settings.json
  - frappe/website/page_renderers/print_page.py
  - frappe/__init__.py
  - frappe/hooks.py
---

# Print Formats

A Print Format decides how a document looks when printed, sent as a PDF, or attached to an email. Each `Print Format` document is linked to a DocType (`doc_type`) or to a report (`print_format_for = "Report"`). Users design most formats in the desk (see [Printing (user guide)](../../user-guide/data-and-reports/printing.md)). As a developer you ship formats in your app, render them from code, and extend the PDF pipeline.

## Kinds of Print Format

| Kind | Fields | Rendered by |
|---|---|---|
| Standard | No Print Format, or `"Standard"` | Layout of the DocType form, `frappe/www/printview.py` |
| Print Format Builder | `format_data` (layout JSON) | The standard template with the saved layout |
| Print Format Builder (Beta) | `print_format_builder_beta = 1`, `format_data` | `PrintFormatGenerator` in `frappe/utils/weasyprint.py` |
| Custom Jinja | `custom_format = 1`, `print_format_type = "Jinja"`, `html` | `get_rendered_template()` in `frappe/www/printview.py` |
| Raw printing | `custom_format = 1`, `raw_printing = 1`, `raw_commands` | Sent to a printer as raw commands |

`print_format_type` can also be `JS`, a format that the desk renders in the browser. Frappe does not check its `html` as Jinja. A format with `print_format_for = "Report"` is always custom and not standard.

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

`frappe.get_print()` (from `frappe/utils/print_utils.py`) returns HTML, or PDF bytes with `as_pdf=True`. It renders the `printview` page, so the print permission check applies to the current user.

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
| `output` | `None` | A `pypdf.PdfWriter`. Pages are added to it. |
| `no_letterhead` | `0` | Leave out the letter head. |
| `password` | `None` | Encrypt the PDF with this password. |
| `pdf_options` | `None` | wkhtmltopdf options, for example `{"page-size": "A5"}`. |
| `letterhead` | `None` | Letter Head name to use. |
| `pdf_generator` | `None` | `"wkhtmltopdf"` or a value that a `pdf_generator` hook of another app handles. Empty uses the format's `pdf_generator`, then `wkhtmltopdf`. |

`frappe.attach_print(doctype, name, print_format=None, ...)` returns `{"fname": ..., "fcontent": ...}` for an email attachment. It returns a PDF when Print Settings `send_print_as_pdf` is on (the default), else HTML. It skips the print permission check (`frappe.local.flags.ignore_print_permissions`). For a Print Format Builder (Beta) format, it makes the PDF with WeasyPrint.

`frappe.translate.print_language("de")` is a context manager that prints in another language.

### URLs

| URL | Returns |
|---|---|
| `/printview?doctype=Sample Order&name=SO-00001&format=Sample Order Slip` | HTML print view |
| `/Sample Order/SO-00001` | HTML print view (`PrintPage` renderer) |
| `/api/method/frappe.utils.print_format.download_pdf?doctype=Sample Order&name=SO-00001&format=...` | PDF download |
| `/api/method/frappe.utils.print_format.download_multi_pdf` | One PDF for many documents |
| `/api/method/frappe.utils.weasyprint.download_pdf` | PDF of a Print Format Builder (Beta) format |

`download_pdf` takes `doctype`, `name`, `format`, `no_letterhead`, `language`, `letterhead` and `pdf_generator`. It checks print permission on the document.

## PDF generation

Frappe makes PDFs with wkhtmltopdf through `pdfkit` (`frappe.utils.pdf.get_pdf`). The `pdf_generator` field of Print Format has one option, `wkhtmltopdf`, and that is the default. Print Settings has no engine field.

`get_pdf(html, options=None, output=None)` does this:

1. It turns relative URLs into absolute URLs. wkhtmltopdf loads images and CSS from the site URL.
2. It reads page options from the `.print-format` CSS rule in the HTML: `margin-top`, `margin-bottom`, `margin-left`, `margin-right`, `page-size`, `header-spacing`, `orientation`, `page-width`, `page-height`.
3. It sets `page-size` from Print Settings `pdf_page_size` (default `A4`), or `pdf_page_height` and `pdf_page_width` for `Custom`. The default left and right margins are `15mm`.
4. It turns off JavaScript and local file access in wkhtmltopdf.
5. It builds the repeating header and footer with the `pdf_header_html` and `pdf_footer_html` hooks.

### The pdf_generator hook

Frappe ships no `pdf_generator` hook. An app, for example Print Designer, adds one and an extra option for the `pdf_generator` field. When the engine is not `wkhtmltopdf`, `get_print(as_pdf=True)` calls each function in the hook. The first one that returns bytes wins. If none returns a value, Frappe falls back to wkhtmltopdf.

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

To let users pick the engine, add `Sample Engine` to the options of the `pdf_generator` field of Print Format, for example with a Property Setter in your fixtures.

### Other print hooks

| Hook | Called with | Use |
|---|---|---|
| `on_print_pdf` | `doctype`, `name`, `print_format` | Run code before wkhtmltopdf makes a PDF, for example to log it. |
| `get_print_format_template` | `jenv`, `print_format` | Return a compiled Jinja template to use instead of the format's HTML. The last app wins. |
| `pdf_body_html` | `jenv`, `template`, `print_format`, `args` | Render the body HTML. Frappe sets `frappe.utils.pdf.pdf_body_html`. The last app wins. |
| `pdf_header_html`, `pdf_footer_html` | `soup`, `head`, `content`, `styles`, `html_id`, `css` | Build the repeating header and footer. |

## Letter heads

`get_letter_head()` uses, in order: the `letterhead` argument, the document's `letter_head` field, then the `Letter Head` with `is_default = 1`. When `no_letterhead` is not passed, Print Settings `with_letterhead` decides. Frappe renders the letter head `content` and `footer` with Jinja and `{"doc": doc.as_dict()}`, so you can use `{{ doc.customer_name }}` in a letter head.

## Print styles

Frappe renders `templates/styles/standard.css`, then adds the `css` of the selected `Print Style`, then the `css` of the Print Format. The default style is `Redesign`. Frappe ships the styles in `frappe/printing/print_style/` (Classic, Modern, Monochrome, Redesign).

## Raw printing

A raw format sends printer commands (for example ESC/POS or ZPL) instead of HTML. Set `raw_printing` and write the commands in `raw_commands` with Jinja. `frappe.www.printview.get_rendered_raw_commands(doc, name, print_format)` returns the rendered commands. The desk sends them to the printer through QZ Tray when Print Settings `enable_raw_printing` is on.

## Pitfalls

- A draft document of a submittable DocType prints only when Print Settings `allow_print_for_draft` is on (the default). A cancelled one needs `allow_print_for_cancelled`.
- Fields with `print_hide` and fields above the user's permission level are not printed.
- Use `get_formatted()` instead of raw values. Raw values do not follow the user's date, number and currency format.
- wkhtmltopdf fetches images and CSS over HTTP from the site URL. If the server cannot reach its own host name, PDF generation hangs or images are missing.
- wkhtmltopdf runs with `disable-javascript`. Scripts in a print format do not run in the PDF.

## Related

- [Website](../website/website.md)
- [Email and Notifications](email-and-notifications.md)
- [Hooks](../building-apps/hooks.md)
- [Translations](../building-apps/translations.md)
- [Printing (user guide)](../../user-guide/data-and-reports/printing.md)
