---
title: Notifications
description: Send email, in-app, SMS or Slack alerts when documents change, with the Notification DocType, Notification Log and Notification Settings.
version: v15
sources:
  - frappe/email/doctype/notification/notification.json
  - frappe/email/doctype/notification/notification.py
  - frappe/email/doctype/notification/notification.js
  - frappe/email/doctype/notification_recipient/notification_recipient.json
  - frappe/integrations/doctype/slack_webhook_url/slack_webhook_url.json
  - frappe/integrations/doctype/slack_webhook_url/slack_webhook_url.py
  - frappe/desk/doctype/notification_log/notification_log.json
  - frappe/desk/doctype/notification_log/notification_log.py
  - frappe/desk/doctype/notification_settings/notification_settings.json
  - frappe/desk/doctype/notification_type/notification_type.json
  - frappe/model/document.py
  - frappe/hooks.py
---

# Notifications

A Notification sends a message when something happens to a document: it is created, saved, submitted, cancelled, a field changes, or a date comes near. Use it to tell people about a document without code. The message can go by email, as an in-app (system) notification, by SMS, or to a Slack channel.

## Concepts

| DocType | Purpose |
|---|---|
| `Notification` | The rule: which document, which event, which condition, who gets it, and the message. |
| `Notification Log` | One in-app notification for one user. It shows in the bell menu in the desk. |
| `Notification Settings` | Per-user preferences. The name of the record is the user. |
| `Notification Type` | A category for in-app notifications, for example `Alert`. Used for filters and email preferences. |
| `Slack Webhook URL` | A saved Slack incoming webhook for the Slack channel. |

## Create a notification

1. Go to `/app/notification/new`.
2. Optional: enter a name. If you do not, Frappe uses the subject as the name.
3. Make sure that **Enabled** is set. It is set by default.
4. Select the **Channel**.
5. Enter the **Subject**.
6. Select **Send Alert On**.
7. Select the **Document Type**.
8. Optional: add a **Condition**.
9. Add rows in **Recipients**.
10. Enter the **Message**.
11. Save.

![Notification form for Sample Order with the Submit event and a condition](../../assets/images/user-guide/notifications-form.png)

For the **Days Before** and **Days After** events, a saved notification shows **Get Alerts for Today**. It lists the documents that would get the alert today.

## Channels

| Channel | Sends | Needs |
|---|---|---|
| Email | An email through the Email Queue. | An outgoing Email Account. Optional **Sender**. |
| Slack | The rendered message to a Slack channel. | A **Slack Channel** (`slack_webhook_url`), a `Slack Webhook URL` record. |
| System Notification | A `Notification Log` for each recipient user. | Recipients that resolve to users. |
| SMS | A text message through the SMS gateway. | SMS settings and recipients with a mobile number. |

For the other channels, select **Send System Notification** to also create an in-app notification. Set **Notification Title** and **Notification Message** for it.

### Slack Webhook URL

1. Make an incoming webhook in your Slack workspace.
2. Go to `/app/slack-webhook-url/new`.
3. Enter **Name** (`webhook_name`) and **Webhook URL** (`webhook_url`).
4. Optional: set **Show link to document** to add a "Go to the document" button to the message.
5. Save.
6. Select the record in **Slack Channel** on the Notification.

## Events

| Send Alert On | When it runs |
|---|---|
| New | After a document is inserted (`after_insert`). |
| Save | After every save (`on_update`). |
| Submit | After submit (`on_submit`). |
| Cancel | After cancel (`on_cancel`). |
| Value Change | After save, when the field in **Value Changed** has a new value. Not on insert. |
| Method | When the controller method in **Trigger Method** runs, for example `before_insert`. |
| Days Before / Days After | Once a day, for documents whose **Reference Date** field is the given number of days from today. |
| Custom | Never on its own. Code sends it, for example with `notification.send(doc)`. |

Date-based events also work on child table DocTypes. Other events do not. You cannot create a notification on `Email Queue`.

## Conditions

**Condition** is an optional Python expression. The notification is sent only when it is true. The expression gets `doc`, `nowdate` and a `frappe` object with `frappe.utils` only.

```python
doc.status == "Open" and doc.total > 500
```

## Recipients

Each row in **Recipients** can add people in these ways:

| Field | Description |
|---|---|
| Receiver By Document Field | `owner`, or a field of the document (or of a child table) that holds a user, an email address or a mobile number. |
| Receiver By Role | All users with this role. |
| CC / BCC | Email addresses, one on each line. Jinja is allowed. Email channel only. |
| Condition | A Python expression. The row is used only when it is true. |

Select **Send To All Assignees** to also email every user the document is assigned to (Email channel).

## Message

**Subject** and **Message** are Jinja templates. The context has `doc`, `alert` (the Notification) and `comments` (the latest comments on the document). Frappe renders them with restricted globals.

```jinja
<h3>Order {{ doc.name }}</h3>
<p>{{ doc.customer_name }} placed an order of {{ doc.total }}.</p>
```

Other options:

| Field | Description |
|---|---|
| Attach Print / Print Format | Attach a PDF of the document. |
| Attach Files / From Attach Field | `From Field` (one Attach field) or `All` (all attachments). |
| Set Property After Alert / Value To Be Set | Set a field on the document after the notification is sent. |
| Notification Type | The type of the in-app notification. Default `Alert`. |

## Notification Log and the bell menu

Each in-app notification is a `Notification Log` for one user (`for_user`). The bell icon in the desk lists them. A user can only read their own logs. Frappe does not create a log when the recipient is the user who made the change, except for types in the `notification_self_notify_types` hook (`Alert` and `Energy Point` in Frappe).

When a log is created, Frappe also emails it to the user if the user has email notifications on for that type. Types in the `notification_skip_email_types` hook (`Alert` in Frappe) do not send this extra email, because the Notification rule sends its own email.

## Notification Settings

Each user can open their settings from the gear icon in the bell menu, or at `/app/notification-settings/<user>`.

| Field | Default | Description |
|---|---|---|
| Enable System Notification (`enabled`) | 1 | Turn off all in-app notifications for this user. |
| Enable Email Notifications | 1 | Receive notification emails. |
| Email me for these notification types | | Types that send an email. |
| Mentions / Assignments / Energy Points / Document Share / Event Reminders | 1 | Email for these events. |
| Email Threads on Assigned Document | 1 | Email when a reply arrives on a document assigned to you. |
| Energy Points (`energy_points_system_notifications`) | 1 | In-app notifications for energy points. |
| Open Documents (`subscribed_documents`) | | DocTypes to follow in the notifications panel. |

## Pitfalls

- Notifications do not run during data import with emails muted, during patches, or during install.
- If the **Value Changed** field is removed from the DocType, Frappe disables the notification and logs an error.
- A Standard notification (`is_standard`) can only be edited in developer mode. To change one, disable it and duplicate it.
- An error in the template stops the save that triggered it, with "Error in Notification".
- An error while sending, for example a Slack request that fails, does not stop the save. Frappe logs "Failed to send Notification" in the Error Log.

## For developers

- Ship a notification with an app: set **Is Standard** and a **Module** in developer mode. Frappe exports it to `<module>/notification/<name>/` with the JSON, the message file (`.md`, `.html` or `.txt` by **Message Type**) and a `.py` file with `get_context(context)`.
- Send a notification from code: `frappe.get_doc("Notification", name).send(doc)`. It sends at once and does not check **Condition**.
- `trigger_daily_alerts` runs daily from `scheduler_events` in `frappe/hooks.py`.
- Create an in-app notification from code with `enqueue_create_notification(users, doc_dict)` in `frappe/desk/doctype/notification_log/notification_log.py`.
- See [Email and Notifications](../../developer-guide/output/email-and-notifications.md) for the developer guide.

## Related

- [Email](email.md)
- [Workflow](../customization/workflow.md)
- [Automation](automation.md)
- [Collaboration](../basics/collaboration.md)
