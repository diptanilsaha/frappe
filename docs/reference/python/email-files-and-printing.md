---
title: Email, Files and Printing API
description: Signatures and behavior of frappe.sendmail, the email queue, Communication make, the File DocType, file endpoints, get_print, attach_print, get_pdf and Jinja rendering.
version: v16
sources:
  - frappe/email/__init__.py
  - frappe/email/queue.py
  - frappe/email/doctype/email_queue/email_queue.py
  - frappe/email/email_body.py
  - frappe/core/doctype/communication/email.py
  - frappe/core/doctype/file/file.py
  - frappe/core/doctype/file/utils.py
  - frappe/core/api/file.py
  - frappe/handler.py
  - frappe/utils/print_utils.py
  - frappe/utils/pdf.py
  - frappe/www/printview.py
  - frappe/utils/jinja.py
---

# Email, Files and Printing API

This page lists the public functions for outgoing email, files, print output and Jinja templates. Use it to look up a signature, a default or a side effect. For how-to guides, read [Email and Notifications](../../developer-guide/output/email-and-notifications.md), [Files](../../developer-guide/data-model/files.md) and [Print Formats](../../developer-guide/output/print-formats.md).

**Whitelisted** means the function is callable at `/api/method/<dotted.path>`. "any" means the default set: `GET`, `POST`, `PUT`, `DELETE`.

## frappe.sendmail

`frappe.sendmail` is `frappe.email.sendmail`. It makes an `Email Queue` record. The scheduler sends queued emails (`frappe.email.queue.flush`).

```python
frappe.sendmail(
	recipients=None, sender="", subject="No Subject", message="No Message", as_markdown=False, delayed=True,
	reference_doctype=None, reference_name=None, unsubscribe_method=None, unsubscribe_params=None,
	unsubscribe_message=None, add_unsubscribe_link=1, attachments=None, content=None, doctype=None, name=None,
	reply_to=None, queue_separately=False, cc=None, bcc=None, message_id=None, in_reply_to=None, send_after=None,
	expose_recipients=None, send_priority=1, communication=None, retry=1, now=None, read_receipt=None,
	is_notification=False, inline_images=None, template=None, args=None, header=None, print_letterhead=False,
	with_container=False, email_read_tracker_url=None, x_priority: Literal[1, 3, 5] = 3, email_headers=None,
	raw_html=False, add_css=True, redact_message_after_send=False, wrapper=None,
) -> EmailQueue | None
```

| Parameter | Default | Description |
|---|---|---|
| `recipients`, `cc`, `bcc` | `None` | List, or a comma-separated string. Duplicates are removed. Unsubscribed addresses and `.invalid` addresses are dropped. |
| `sender` | `""` | Empty or `"Administrator"` uses the `default_sender` of the outgoing Email Account. |
| `subject` | `"No Subject"` | |
| `message`, `content` | `"No Message"`, `None` | HTML body. `content` wins over `message`. |
| `as_markdown` | `False` | Convert the body from Markdown. |
| `template`, `args` | `None` | Render `templates/emails/<template>.html` (and `.txt` if it exists) with `args`. Replaces `message`. |
| `delayed` | `True` | `False` sets `now=True`. |
| `now` | `None` | Send after the current transaction commits, not on the next scheduler run. |
| `reference_doctype`, `reference_name` | `None` | Link the queue to a document. `doctype` and `name` are aliases. Also used to pick the Email Account. |
| `attachments` | `None` | List of dicts. See below. |
| `reply_to`, `message_id`, `in_reply_to` | `None` | Headers. `in_reply_to` is a Communication name. Its `message_id` becomes `In-Reply-To`. |
| `send_after` | `None` | Datetime. An integer means that many days from today. |
| `expose_recipients` | `None` | `"header"` shows all To addresses. `"footer"` adds a "sent to" line. `None` hides To recipients from each other. |
| `queue_separately` | `False` | One Email Queue per To recipient, made in a `long` queue job. Always on for more than 100 recipients. Then the function returns `None`. |
| `send_priority` | 1 | Higher is sent first. |
| `x_priority` | 3 | `X-Priority` header: 1 highest, 3 normal, 5 lowest. |
| `add_unsubscribe_link`, `unsubscribe_method`, `unsubscribe_params`, `unsubscribe_message` | 1, `None`, `None`, `None` | The link is added only with a reference DocType and an `unsubscribe_message` (or for Newsletter). Default method `/api/method/frappe.email.queue.unsubscribe`. |
| `communication` | `None` | Communication name to link. Needs at least one recipient. |
| `read_receipt` | `None` | Add `Disposition-Notification-To`. |
| `is_notification` | `False` | Mark as a notification email. |
| `inline_images` | `None` | List of `{"filename", "filecontent"}`. `src` values are replaced with Content-IDs. |
| `header` | `None` | Header block added to the body. |
| `with_container`, `wrapper` | `False`, `None` | Wrap in a styled container. `wrapper` defaults to `templates/emails/standard.html`. |
| `print_letterhead` | `False` | Letterhead for print attachments. |
| `email_read_tracker_url` | `None` | URL for read tracking. |
| `email_headers` | `None` | Extra headers. `X-` is added when missing. |
| `raw_html`, `add_css` | `False`, `True` | Use the template as a full HTML file. Add CSS from the `email_css` hook. |
| `redact_message_after_send` | `False` | Replace the stored body after it is sent. |
| `retry` | 1 | Stored on the Email Queue as `retry`. |

Return value: the `Email Queue` document. An empty list when no recipient is left. `None` when `queue_separately` applies. Raises `frappe.OutgoingEmailError` when no outgoing Email Account exists.

Attachment dicts:

| Shape | Description |
|---|---|
| `{"fid": "<File name>"}` or `{"file_url": "/private/files/a.pdf"}` | An existing File. Read when the email is sent. |
| `{"print_format_attachment": 1, "doctype": ..., "name": ..., "print_format": ..., "lang": ...}` | A print rendered when the email is sent. `lang` defaults to the current language. |
| `{"fname": "a.pdf", "fcontent": b"..."}` | Content in memory, for example from `frappe.attach_print`. |

```python
frappe.sendmail(
	recipients=["user1@example.com"],
	subject="Order SO-00001",
	template="sample_order",
	args={"customer": "Customer A"},
	reference_doctype="Sample Order",
	reference_name="SO-00001",
	attachments=[frappe.attach_print("Sample Order", "SO-00001", file_name="SO-00001")],
)
```

## Email helpers

| Function | Description |
|---|---|
| `frappe.email.sendmail_to_system_managers(subject, content)` | `sendmail` to all enabled System Managers. |
| `frappe.email.get_system_managers()` | User IDs of enabled System Managers, without Administrator. Reads `Has Role` with SQL. |
| `frappe.email.get_contact_list(txt: str, page_length: int = 20, extra_filters: str \| None = None) -> list[dict]` | Whitelisted (any). Contact emails that match `txt`, as `{"value", "label", "description"}`. Applies permissions. |
| `frappe.email.get_recipient_avatars(emails: str) -> dict` | Whitelisted (any). `{"user_info": ..., "contact_images": ...}` for up to 100 addresses. |
| `frappe.email.relink(name: str, reference_doctype=None, reference_name: str \| int \| None = None)` | Whitelisted (any). Point a Communication at another document with SQL. Needs write on the Communication. Changes only rows with `communication_type` "Communication". |
| `frappe.email.get_communication_doctype(doctype, txt, searchfield, start, page_len, filters)` | Whitelisted (any). Link search for DocTypes the user can read. |
| `frappe.email.queue.flush()` | Send a batch of queued emails (`email_queue_batch_size`, default 500). Called by the scheduler. Does nothing when sending is suspended. |
| `frappe.email.queue.get_queue()` | The next batch of Email Queue rows to send. |
| `frappe.email.queue.retry_sending_emails()` | Requeue emails stuck in `Sending` for more than 15 minutes, up to the retry limit. Then sets `Error`. Commits. |
| `frappe.email.queue.get_emails_sent_this_month(email_account=None)`, `get_emails_sent_today(email_account=None)` | Counts of sent emails. |
| `frappe.email.queue.unsubscribe(doctype: str, name: str, email: str)` | Whitelisted (any, guest). The unsubscribe link target. Checks the signed URL. |
| `frappe.email.queue.get_unsubcribed_url(reference_doctype, reference_name, email, unsubscribe_method, unsubscribe_params)` | Signed unsubscribe URL. The name has this spelling in the code. |
| `frappe.email.doctype.email_queue.email_queue.send_now(name, force_send: bool = False)` | Whitelisted (any). Send one Email Queue now. Checks permission on it. |
| `frappe.email.doctype.email_queue.email_queue.retry_sending(queues)` | Whitelisted (any). Set `Error` rows back to `Not Sent`. |
| `frappe.email.doctype.email_queue.email_queue.toggle_sending(enable)` | Whitelisted (any). System Manager only. Sets the default `suspend_email_queue`. |
| `frappe.email.email_body.get_email_html(template: str, args: str, subject: str, header=None, with_container=False)` | Whitelisted (any). Render an email template to HTML. `args` is a JSON string. |
| `frappe.email.email_body.get_email(recipients, sender="", msg="", subject="[No Subject]", text_content=None, footer=None, print_html=None, formatted=None, attachments=None, content=None, reply_to=None, cc=None, bcc=None, email_account=None, expose_recipients=None, inline_images=None, header=None, x_priority=3)` | Build a MIME message object. Does not queue or send. |

### Communication make

```python
frappe.core.doctype.communication.email.make(
	doctype=None, name=None, content=None, subject=None, sent_or_received="Sent", sender=None,
	sender_full_name=None, recipients=None, communication_medium="Email", send_email=False,
	print_html=None, print_format=None, attachments=None, send_me_a_copy=False, cc=None, bcc=None,
	read_receipt=None, print_letterhead=True, letterhead=None, email_template=None, communication_type=None,
	send_after=None, print_language=None, now=False, raw_html=False, add_css=True, in_reply_to=None,
	print_style=None, print_settings=None, **kwargs,
) -> dict[str, str]
```

Whitelisted (any). This is the desk "New Email" action. It makes a Communication linked to the document and, with `send_email`, queues the email.

- Only System Users can call it. It checks `email` permission on the document, or `create` on Communication when no document is given.
- `print_format` attaches a print of the document. `attachments` takes File names or `{"fname", "fcontent"}` dicts.
- `sender` defaults to the session user. Extra `kwargs` show a deprecation warning.
- With an `email_template`, `raw_html` works only when the Email Template has "Use HTML" set. Without a template, `raw_html` is used as given.

## File DocType

`frappe.core.doctype.file.file.File`. Files are stored on the site file system, unless an app sets the `write_file` hook. Make a file with `frappe.get_doc({"doctype": "File", ...}).insert()`.

| Field | Description |
|---|---|
| `file_name` | Name of the file. |
| `file_url` | `/files/<name>`, `/private/files/<name>`, or an external URL. |
| `content` | Bytes or string to store on insert. Not saved as a field. |
| `is_private` | 1 for private storage. |
| `attached_to_doctype`, `attached_to_name`, `attached_to_field` | The document and field the file belongs to. |
| `folder` | Folder name. Default is `Home`, or `Home/Attachments` for attachments. |
| `is_folder` | 1 for a folder record. |
| `decode` | Set with `content` to decode base64 content. |

```python
file = frappe.get_doc({
	"doctype": "File",
	"file_name": "sample-order.txt",
	"content": "hello",
	"is_private": 1,
	"attached_to_doctype": "Sample Order",
	"attached_to_name": "SO-00001",
}).insert()
file.file_url      # "/private/files/sample-order.txt"
file.unique_url    # "/private/files/sample-order.txt?fid=<name>"
file.get_content() # "hello"
```

| Member | Description |
|---|---|
| `get_content(encodings=None) -> bytes \| str` | Return the stored content. Text is decoded with `utf-8-sig`, `utf-8`, `windows-1250`, `windows-1252`. Binary content stays bytes. Raises for folders. |
| `save_file(content=None, decode=False, ignore_existing_file_check=False, overwrite=False)` | Store content. Does nothing for remote files. Reuses the `file_url` of a File with the same `content_hash` and `is_private`, unless `ignore_existing_file_check` or `overwrite` is set. Calls the `before_write_file` and `write_file` hooks. |
| `save_file_on_filesystem()` | Write the content to `public/files` or `private/files` and return `{"file_name", "file_url"}`. |
| `write_file()` | Write the content to the path from `get_full_path()`. |
| `delete_file_data_content(only_thumbnail=False)` | Call the `delete_file_data_content` hook, or `delete_file_from_filesystem`. |
| `delete_file_from_filesystem(only_thumbnail=False)` | Delete the file and its thumbnail from disk. |
| `generate_content_hash()` | Set `content_hash` from the content, or from the file on disk. |
| `get_full_path()` | Disk path from `file_url` or `file_name`. Raises for an unsafe path or a `file_name` with `/`. |
| `exists_on_disk()` | `True` if `get_full_path()` exists. |
| `is_downloadable()` | `True` if the session user can read the file. |
| `get_extension()` | `os.path.splitext(file_name)`, for example `("sample-order", ".txt")`. |
| `is_remote_file` | Property. `True` when `file_url` starts with `http://`, `https://` or `/api/method/`, or there is no content. |
| `unique_url` | Property. For a private file, `file_url` plus `?fid=<name>`. Faster permission checks. |
| `create_attachment_copy(attached_to_doctype, attached_to_name, attached_to_field=None, ignore_permissions=False)` | Insert a new File for another document that reuses the same `file_url`. |
| `make_thumbnail(set_as_thumbnail=True, width=300, height=300, suffix="small", crop=False) -> str` | Make a thumbnail image. Returns its URL, or `None` when the image cannot be read. |
| `unzip() -> list[File]` | Extract a `.zip` into File records and delete the zip. Needs read permission. Limited by `max_zip_extract_size` and `max_zip_extract_members`. |
| `optimize_file()` | Whitelisted (document method). Compress a local image. Raises `NotImplementedError` for other files. Needs write permission. |
| `check_max_file_size()` | Raise `MaxFileSizeReachedError` above `get_max_file_size()`. Skip with `flags.skip_file_size_check`. |
| `File.zip_files(files) -> bytes` | Static. Zip the files (names or File objects) that the user can read. |

Reading a private file needs `File` read permission: owner, shared, or read on the attached document. See [Files](../../developer-guide/data-model/files.md).

## File endpoints

All in `frappe.core.api.file` unless noted.

| Function | Description |
|---|---|
| `frappe.handler.upload_file()` | Whitelisted (POST, guest when System Settings allow). No rate limit. The upload endpoint. See [Files](../../developer-guide/data-model/files.md). |
| `frappe.handler.download_file(file_url: str)` | Whitelisted (any, guest). No rate limit. Download a file with permission checks. |
| `unzip_file(name: str)` | Whitelisted (POST). `File.unzip()`. |
| `get_attached_images(doctype: str, names) -> _dict` | Whitelisted (any). `{docname: [file_url, ...]}` for the given documents. Applies permissions. |
| `get_files_in_folder(folder: str, start: int = 0, page_length: int = 20) -> dict` | Whitelisted (any). `{"files": [...], "has_more": bool}`. |
| `get_files_by_search_text(text: str) -> list[dict]` | Whitelisted (any). Up to 20 files that match the name. |
| `get_max_file_size() -> int` | Whitelisted (any, guest). Bytes. System Settings `max_file_size` (MB), else `max_file_size` in config (bytes), else 25 MB. |
| `get_max_extract_size() -> int` | Bytes. System Settings `max_zip_extract_size` (MB), else config, else 25 MB. |
| `get_max_extract_members() -> int` | Config `max_zip_extract_members`, default 1000. |
| `get_file_chunk_size() -> int` | Config `file_chunk_size`, default 25 MB. |
| `create_new_folder(file_name: str, folder: str) -> File` | Whitelisted (any). Insert a folder. |
| `move_file(file_list, new_parent: str, old_parent: str) -> None` | Whitelisted (any). Move files to another folder. |
| `zip_files(files: str)` | Whitelisted (any). Send a `files.zip` download. |

`frappe.utils.file_manager` helpers are in [Utilities](utils.md).

## Printing

### get_print

```python
frappe.get_print(
	doctype=None, name=None, print_format=None, style=None, as_pdf=False, doc=None, output=None,
	no_letterhead=0, password=None, pdf_options=None, letterhead=None,
	pdf_generator: Literal["wkhtmltopdf", "chrome", "PlutoPrint", "Typst", "WeasyPrint"] | None = None, settings=None,
)
```

`frappe.get_print` is `frappe.utils.print_utils.get_print`. It returns the print HTML (`str`), or PDF `bytes` with `as_pdf=True`.

- `print_format` empty or `"Standard"` uses the default print format of the DocType. A missing Print Format falls back to Standard.
- `doc` renders a document object instead of loading `doctype`/`name`.
- `password` encrypts the PDF.
- `output` is a `pypdf.PdfWriter`. Pages are added to it and it is returned.
- The PDF engine is picked by `resolve_pdf_generator`: the `pdf_generator` argument, then the Print Format setting, then Print Settings `pdf_generator` (field default `chrome`), else `wkhtmltopdf`. PlutoPrint falls back to `chrome` when it cannot render the format. A format made with the print format builder keeps `PlutoPrint`, `Typst` or `WeasyPrint`, else uses `chrome`.
- For `wkhtmltopdf`, the `print_font` key is removed from `settings`.
- `on_print_pdf` hooks run before a PDF. For an engine other than `wkhtmltopdf`, `pdf_generator` hooks can return the PDF. Else `frappe.utils.pdf.get_pdf` renders it with `wkhtmltopdf`.
- Permission: the user needs `read` or `print` on the document, or website permission, or a valid share `key`. Submittable drafts and cancelled documents need the Print Settings that allow them.

### Other print functions

| Function | Description |
|---|---|
| `frappe.attach_print(doctype, name, file_name=None, print_format=None, style=None, html=None, doc=None, lang=None, print_letterhead=True, password=None, letterhead=None, settings=None) -> dict` | Return `{"fname": ..., "fcontent": ...}` for `sendmail`. PDF when Print Settings `send_print_as_pdf` is set, else HTML. `file_name` defaults to `name`. Spaces are removed and `/` becomes `-`. Skips the print permission check. `lang` sets the print language. |
| `frappe.utils.pdf.get_pdf(html, options=None, output=None, smart_shrinking: bool = False)` | Render HTML to PDF bytes with `wkhtmltopdf`. `options["password"]` encrypts the PDF. With `output`, returns the writer. See [Utilities](utils.md). |
| `frappe.utils.pdf.encrypt_pdf(content: bytes, password: str) -> bytes` | Encrypt a PDF. |
| `frappe.utils.pdf.pdf_contains_js(file_content: bytes)` | `True` if the PDF has JavaScript. |
| `frappe.utils.print_utils.resolve_pdf_generator(print_format=None, pdf_generator=None) -> str` | Pick the PDF engine. See `get_print`. |
| `frappe.utils.print_utils.convert_uom(number, from_uom="px", to_uom="px", only_number=False)` | Convert between `px`, `mm`, `cm`, `in` at 96 px per inch. Returns a string like `"10.0mm"` unless `only_number`. |
| `frappe.utils.pdf.is_wkhtmltopdf_valid()` | Whitelisted (any). `True` if the installed `wkhtmltopdf` has the patched Qt build. |
| `frappe.utils.print_utils.setup_chromium()` | Download Chromium for the `chrome` engine if it is missing. `bench setup-chrome` calls it. |
| `frappe.utils.print_utils.parse_float_and_unit(input_text, default_unit="px")` | `"10mm"` becomes `{"value": 10.0, "unit": "mm"}`. |

!!! warning
    `get_pdf` returns before encryption when `output` is given. The password in `options` has no effect then.

## Jinja templates

All in `frappe.utils.jinja`. `frappe.render_template`, `frappe.get_template`, `frappe.get_jenv` and `frappe.get_email_from_template` are aliases.

| Function | Description |
|---|---|
| `render_template(template, context=None, is_path=None, safe_render=True, *, restrict_globals=None) -> str` | Render a template path or a template string. Returns `""` for an empty template. |
| `get_template(path)` | Return a compiled template from the `templates` folders of the apps. App templates win over frappe templates. |
| `get_jenv(*, restrict_globals=None)` | Return the sandboxed Jinja environment for the request. |
| `get_email_from_template(name, args) -> tuple` | Render `templates/emails/<name>.html` and `.txt`. Returns `(html, text or None)`. |
| `validate_template(html, restrict_globals=None)` | Raise `ValidationError` on a Jinja syntax error. |
| `guess_is_path(template) -> bool` | `True` for a one-line string that ends in `.html`, `.css`, `.scss`, `.py`, `.md`, `.json`, `.js`, `.xml` or `.txt`. |
| `get_jinja_hooks()` | `(methods, filters)` from the `jinja` hook. |
| `get_jloader()`, `set_filters(jenv)`, `safe_render_flags()` | Internal helpers. |

`render_template` behavior:

- The environment is a sandbox. `restrict_globals=None` gives the full safe-exec globals, unless `disable_render_safe_exec` is set to a false value in `common_site_config.json`. Then the template gets `render_safe_globals()`. `True` always gives `render_safe_globals()`.
- With `safe_render=True`, a template string that contains `.__` raises "Illegal template".
- An undefined variable renders as its source, for example `{{ missing }}`.
- A syntax error raises `ValidationError`. An error while rendering raises the original exception class. Users allowed to see tracebacks get the traceback as the message.

```python
frappe.render_template("Hello {{ name }}", {"name": "Customer A"})  # "Hello Customer A"
frappe.render_template("templates/emails/new_user.html", {"link": "/login"})
```

## Pitfalls

- `frappe.sendmail` does not send in the request. Use `now=True` only when the user waits for the email. It still sends after commit.
- `attach_print` skips the print permission check. Check permission before you attach a document the user asked for.
- `File.get_content()` returns `str` for text files and `bytes` for binary files.
- Template strings from users are rendered in the sandbox. Pass `restrict_globals=True` for them to get the smaller set of globals.

## Related

- [Email and Notifications](../../developer-guide/output/email-and-notifications.md)
- [Files](../../developer-guide/data-model/files.md)
- [Print Formats](../../developer-guide/output/print-formats.md)
- [Utilities](utils.md)
- [frappe namespace](frappe.md)
