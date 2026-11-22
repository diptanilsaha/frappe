---
title: Email and Notifications
description: Send email with frappe.sendmail and the Email Queue, use Email Templates, ship Notifications in an app, log Communications, handle incoming email and create system notifications.
version: v16
sources:
  - frappe/email/__init__.py
  - frappe/email/queue.py
  - frappe/email/email_body.py
  - frappe/email/receive.py
  - frappe/email/doctype/email_queue/email_queue.py
  - frappe/email/doctype/email_template/email_template.py
  - frappe/email/doctype/notification/notification.py
  - frappe/email/doctype/email_account/email_account.py
  - frappe/core/doctype/communication/communication.py
  - frappe/core/doctype/communication/email.py
  - frappe/desk/doctype/notification_log/notification_log.py
  - frappe/desk/doctype/notification_type/notification_type.py
  - frappe/model/document.py
  - frappe/utils/jinja.py
  - frappe/utils/print_utils.py
  - frappe/integrations/doctype/slack_webhook_url/slack_webhook_url.py
  - frappe/hooks.py
---

# Email and Notifications

Frappe sends all outgoing email through the `Email Queue`. Your code calls `frappe.sendmail()`, and the scheduler sends the queue in the background. Use a `Notification` to send email, system notifications, SMS or Slack messages when a document event happens, without code in the controller. Use `Communication` to keep an email thread on a document. For setup of email accounts and the desk side, see [Email (user guide)](../../user-guide/automation/email.md) and [Notifications (user guide)](../../user-guide/automation/notifications.md).

## Send email

```python
frappe.sendmail(
	recipients=["user1@example.com"],
	subject="Order SO-00001 is ready",
	message="<p>Your order is ready.</p>",
	reference_doctype="Sample Order",
	reference_name="SO-00001",
	attachments=[
		{
			"print_format_attachment": 1,
			"doctype": "Sample Order",
			"name": "SO-00001",
			"print_format": "Standard",
		}
	],
)
```

`frappe.sendmail()` builds an `Email Queue` document and returns it. It does not send the email now. The scheduler job `frappe.email.queue.flush` (event `all`) sends queued email. A default outgoing `Email Account` must exist, or Frappe raises `frappe.OutgoingEmailError`.

| Parameter | Default | Description |
|---|---|---|
| `recipients` | `[]` | List of email addresses, or a comma-separated string. |
| `sender` | `""` | From address. Empty uses the outgoing Email Account. |
| `subject` | `"No Subject"` | Subject line. |
| `message` / `content` | `"No Message"` | HTML body. `content` wins when both are set. |
| `as_markdown` | `False` | Convert the body from Markdown to HTML. |
| `template`, `args` | `None` | Render `templates/emails/<template>.html` (and `.txt`) with `args`. |
| `cc`, `bcc` | `[]` | Copy recipients. |
| `reply_to` | `None` | Reply-To address. |
| `reference_doctype`, `reference_name` | `None` | Link the email to a document. Also `doctype` and `name`. |
| `attachments` | `None` | List of attachment dicts. See below. |
| `delayed` | `True` | `False` sends after the transaction commits. Same as `now=True`. |
| `now` | `None` | Send after commit instead of waiting for the scheduler. |
| `send_after` | `None` | Do not send before this datetime. |
| `send_priority` | `1` | Higher values are sent first. |
| `queue_separately` | `False` | Make one Email Queue per recipient. |
| `expose_recipients` | `None` | `"header"` shows all To recipients. `"footer"` adds "This email was sent to ..." text. |
| `add_unsubscribe_link` | `1` | Add an unsubscribe link. |
| `unsubscribe_method`, `unsubscribe_params`, `unsubscribe_message` | `None` | Customize the unsubscribe link. |
| `header` | `None` | Add a header block to the email. |
| `with_container` | `False` | Wrap the body in a styled container. |
| `print_letterhead` | `False` | Add the letter head to print attachments. |
| `inline_images` | `None` | List of `{"filename", "filecontent"}`. |
| `read_receipt` | `None` | Ask for a read receipt. |
| `x_priority` | `3` | `1` highest, `3` normal, `5` lowest. |
| `email_headers` | `None` | Extra headers. Frappe adds `X-` when it is missing. |
| `raw_html` | `False` | Send the body as a full HTML document. |
| `add_css` | `True` | Add CSS from the `email_css` hook. |
| `redact_message_after_send` | `False` | Replace the stored body with a placeholder after it is sent. |
| `communication` | `None` | Link the Email Queue to a Communication. |
| `message_id`, `in_reply_to` | `None` | Threading headers. A reply to this email sends `message_id` back as `In-Reply-To`. |
| `wrapper` | `None` | Jinja template that wraps the body. Empty uses `templates/emails/standard.html`. |

### Attachments

| Shape | Description |
|---|---|
| `{"fname": "a.txt", "fcontent": b"..."}` | File content in memory. |
| `{"fid": "<File name>"}` or `{"file_url": "/files/a.pdf"}` | An existing `File`. Read when the email is sent. |
| `{"print_format_attachment": 1, "doctype": ..., "name": ..., "print_format": ...}` | A print of a document. Made with `frappe.attach_print()` when the email is sent. |

You can also pass the return value of `frappe.attach_print(doctype, name, print_format=...)` directly. See [Print Formats](print-formats.md).

### Email templates in files

`template="sample_order_ready"` renders `templates/emails/sample_order_ready.html` from any app with `args` as context. A `.txt` file with the same name gives the plain text part.

```jinja
{# sample_app/templates/emails/sample_order_ready.html #}
<p>{{ _("Hello") }} {{ customer_name }},</p>
<p>{{ _("Order {0} is ready.").format(order) }}</p>
```

```python
frappe.sendmail(
	recipients=["user1@example.com"],
	subject="Order ready",
	template="sample_order_ready",
	args={"customer_name": "Customer A", "order": "SO-00001"},
)
```

## Email Template

An `Email Template` is a document with a Jinja `subject` and `response` (or `response_html` when `use_html` is set). Users pick it in the email dialog. Render it from code with `get_formatted_email(doc)`.

```python
template = frappe.get_doc("Email Template", "Sample Order Ready")
email = template.get_formatted_email(frappe.get_doc("Sample Order", "SO-00001").as_dict())
# {'subject': 'Order SO-00001 is ready', 'message': 'Hello Customer A'}

frappe.sendmail(recipients=["user1@example.com"], **email)
```

The whitelisted method `frappe.email.doctype.email_template.email_template.get_email_template(template_name, doc, sender=None)` does the same for the client. It checks read permission on the template.

## Communication

A `Communication` is one email (or comment, call, chat) linked to a document with `reference_doctype` and `reference_name`. It shows in the document timeline. The desk email dialog calls `frappe.core.doctype.communication.email.make`.

```python
from frappe.core.doctype.communication.email import make

make(
	doctype="Sample Order",
	name="SO-00001",
	subject="Order update",
	content="<p>Your order ships today.</p>",
	recipients="user1@example.com",
	send_email=True,
	print_format="Standard",
)
```

`make()` only works for system users. It checks `email` permission on the reference document and `create` permission on Communication. With `send_email=True`, it queues the email. Without it, it only saves the Communication.

After a Communication is saved, Frappe calls `on_communication_update(communication)` on the reference document, if the controller has it. Otherwise it runs the default update (status and response time fields) and calls `notify_communication(communication)`.

```python
# sample_app/sample_app/selling/doctype/sample_order/sample_order.py
class SampleOrder(Document):
	def on_communication_update(self, communication):
		if communication.sent_or_received == "Received":
			self.db_set("status", "Open")
```

## Incoming email

An `Email Account` with `enable_incoming` pulls email every 10 minutes (`frappe.email.doctype.email_account.email_account.pull`). Each email becomes a `Communication`. To make documents from email:

1. Set `email_append_to` on your DocType.
2. Set `subject_field` and `sender_field` on the DocType. Optional: `sender_name_field`.
3. In the Email Account, set `Append To` to your DocType (or set it per IMAP folder).

For a new thread, `InboundMail` creates a document of that DocType with the subject and sender, with `ignore_mandatory`, and links the Communication to it. For a reply, it finds the document from the `In-Reply-To` header, or by subject and sender.

Frappe's `hooks.py` sets an `email_append_to` hook to `["Event", "ToDo", "Communication"]`, but no code reads it. Use the DocType property.

## Notifications in an app

A `Notification` sends a message when a document event happens. A standard Notification (`is_standard = 1`) is exported to the app when you save it in developer mode.

```text
sample_app/sample_app/selling/notification/sample_order_submitted/
├── __init__.py
├── sample_order_submitted.json   # the Notification document
├── sample_order_submitted.md     # message (.html or .txt for other message types)
└── sample_order_submitted.py     # get_context(context)
```

```python
# sample_app/sample_app/selling/notification/sample_order_submitted/sample_order_submitted.py
def get_context(context):
	context.items_count = len(context.doc.items)
```

```markdown
<!-- sample_order_submitted.md -->
Order **{{ doc.name }}** for {{ doc.customer_name }} has {{ items_count }} items.
```

The message, subject and condition get `doc`, `nowdate` and a safe `frappe` namespace. A standard Notification can only be enabled or changed in developer mode.

| Field | Values |
|---|---|
| `channel` | `Email`, `Slack`, `System Notification`, `SMS` |
| `slack_webhook_url` | `Slack Webhook URL` document to post to, for the `Slack` channel |
| `event` | `New`, `Save`, `Submit`, `Cancel`, `Value Change`, `Method`, `Days Before`, `Days After`, `Minutes Before`, `Minutes After`, `Custom` |
| `condition_type` | `Python` (a `condition` expression) or `Filters` (`filters` JSON) |
| `method` | Controller method name for the `Method` event, for example `on_payment` |
| `message_type` | `Markdown`, `HTML`, `Plain Text` |
| `recipients` | Table of `receiver_by_document_field`, `receiver_by_role`, `cc`, `bcc` and a `condition` per row |
| `send_to_all_assignees` | Also send to the users the document is assigned to |
| `attach_print`, `print_format` | Attach a print of the document |
| `send_system_notification` | Also create a system notification for email recipients |
| `set_property_after_alert`, `property_value` | Set a field after the notification is sent |

How events fire (`Document.run_notifications` in `frappe/model/document.py`):

| Event | Runs on |
|---|---|
| `New` | `after_insert` |
| `Save` | `on_update` |
| `Submit` | `on_submit` |
| `Cancel` | `on_cancel` |
| `Value Change` | `on_change`, not during insert or delete |
| `Method` | `doc.run_method("<method>")` with the method in `method` |
| `Days Before`, `Days After` | Daily job `trigger_daily_alerts` |
| `Minutes Before`, `Minutes After` | Job `trigger_offset_alerts`, every 5 minutes |
| `Custom` | Never by itself. Send it from code with `queue_send(doc)`. |

To fire a `Method` notification from code, call the method with `run_method`:

```python
doc = frappe.get_doc("Sample Order", "SO-00001")
doc.run_method("on_payment")
```

To send a `Custom` notification, or any notification without a document event, call `queue_send()`. It runs `evaluate_alert` in a background job after commit.

```python
notification = frappe.get_doc("Notification", "Sample Order Reminder")
notification.queue_send(frappe.get_doc("Sample Order", "SO-00001"))
```

Notifications do not run during install, patches, or imports with `mute_emails`.

## System notifications

A system notification is a `Notification Log` row. Users see it in the bell menu of the desk. Create them with `enqueue_create_notification`.

```python
from frappe.desk.doctype.notification_log.notification_log import enqueue_create_notification

enqueue_create_notification(
	["user1@example.com", "user2@example.com"],
	{
		"type": "Alert",
		"subject": "Order SO-00001 needs approval",
		"document_type": "Sample Order",
		"document_name": "SO-00001",
		"from_user": frappe.session.user,
	},
)
```

- `type` is a link to `Notification Type`. Frappe installs `Mention`, `Energy Point`, `Assignment`, `Share` and `Alert`.
- The logs are made in a background job after commit.
- A user does not get a notification for their own action (`for_user == from_user`), except for types in the `notification_self_notify_types` hook (`Alert`).
- Pass `dedupe_on=["document_type", "document_name"]` to skip users who already have a matching log.
- Notification Settings decide if a log also sends an email. Types in `notification_skip_email_types` (`Alert`) never send their own email.

## Hooks

| Hook | Use |
|---|---|
| `email_css` | CSS files added to every email (`["email.bundle.css"]` in Frappe). |
| `make_email_body_message` | Functions that get the email object before it is sent. Change headers or parts. |
| `override_email_send` | One function `(email_queue, sender, recipient, message)` that sends instead of SMTP. The first app that sets it wins (`get_hook_method`). |
| `default_mail_footer` | HTML added to the footer of every email. |
| `notification_config` | Function that gives open counts for the desk ("for_doctype", "for_module"). |
| `notification_self_notify_types` | Notification Types that also notify the user who acted. |
| `notification_skip_email_types` | Notification Types that never send their own email. |

## Pitfalls

- `frappe.sendmail()` queues the email in the current transaction. If the transaction rolls back, the email is not sent.
- `now=True` sends after commit, in the web request. Use it only for urgent email, like password reset.
- In tests (`frappe.in_test`), the queue is built but not sent over SMTP.
- Set `reference_doctype` and `reference_name` so replies link back to the document.
- Use `_()` in templates, and send with `frappe.translate.print_language` when the recipient uses another language.

## Related

- [Print Formats](print-formats.md)
- [Background Jobs](../runtime/background-jobs.md)
- [Scheduler](../runtime/scheduler.md)
- [Realtime](../runtime/realtime.md)
- [Email (user guide)](../../user-guide/automation/email.md)
- [Notifications (user guide)](../../user-guide/automation/notifications.md)
