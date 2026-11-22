---
title: Collaboration
description: Work on documents with other users - assignments and ToDo, sharing, comments and mentions, likes, follow, tags, activity and the notification log.
version: v16
sources:
  - frappe/public/js/frappe/form/sidebar/assign_to.js
  - frappe/public/js/frappe/form/sidebar/share.js
  - frappe/public/js/frappe/form/templates/set_sharing.html
  - frappe/public/js/frappe/form/footer/form_timeline.js
  - frappe/public/js/frappe/form/controls/text_editor.js
  - frappe/public/js/frappe/form/toolbar.js
  - frappe/public/js/frappe/ui/notifications/notifications.js
  - frappe/desk/form/assign_to.py
  - frappe/desk/doctype/todo/todo.py
  - frappe/desk/doctype/todo/todo_list.js
  - frappe/share.py
  - frappe/desk/form/utils.py
  - frappe/desk/notifications.py
  - frappe/desk/like.py
  - frappe/desk/form/document_follow.py
  - frappe/desk/doctype/tag/tag.py
  - frappe/desk/doctype/notification_log/notification_log.json
  - frappe/desk/doctype/notification_settings/notification_settings.json
  - frappe/core/doctype/user/user.json
  - frappe/hooks.py
---

# Collaboration

The desk has tools to work on documents with other users. Assign a document to a user, share it, comment on it, mention users, follow it and tag it. Each action can send a notification. Most tools are in the [form](form-view.md) sidebar and timeline. Some also work on many documents from the [list view](list-view.md).

## Assignments

An assignment gives a document to a user as a task. Each assignment is a `ToDo` document for that user.

To assign a document:

1. Open the document.
2. In the sidebar, click **+** next to **Assign**.
3. In **Assign To**, select one or more users. Select **Assign to me** to add yourself. Select a user group in **Assign To User Group** to add all its users.
4. Optional: set **Complete By**, **Priority** (**Low**, **Medium** or **High**) and a **Comment**.
5. Click **Add**.

To assign many documents, select them in the list view and click **Actions** > **Assign To**.

When you assign a document, Frappe:

- Creates a `ToDo` with `allocated_to`, `reference_type`, `reference_name`, `description`, `priority` and `date`. The default `date` is today. The default description is "Assignment for <DocType> <name>".
- Adds the user to the `_assign` field of the document. The list view shows the user's avatar on the row.
- Shares the document with the user (read only) if the user cannot read it. If document sharing is turned off in System Settings, it shows an error.
- Sends an `Assignment` notification to the user.
- Makes the user follow the document if **Auto follow documents that are assigned to you** is on for the user.

If the user already has an open ToDo for the document, Frappe does not make a second one.

Click the assigned users under **Assign** to open the **Assignments** dialog. In this dialog:

- Click the check icon (**Done**) next to your own assignment to complete it. This sets the ToDo to `Closed`. Only the assignee can do this.
- Click the cross icon (**Cancel**) to remove an assignment. You can remove your own assignment, or any assignment if you can write the document. This sets the ToDo to `Cancelled`.

In the list view, **Actions** > **Clear Assignment** removes all assignments of the selected documents.

### The ToDo list

Open the **ToDo** list to see your assignments. Click **Open** on a row to go to the assigned document. A user without a ToDo role permission sees only ToDos allocated to, assigned by or created by them.

Calendar and Kanban views are also available for ToDo. Refer to [Views](views.md).

!!! note "For developers"
    Assign from code with `frappe.desk.form.assign_to.add({"assign_to": ["user1@example.com"], "doctype": "Sample Order", "name": "SO-00001", "description": "Check the items"})`. Remove with `frappe.desk.form.assign_to.remove(doctype, name, assign_to)`. If the DocType has a field named `assigned_to`, Frappe also sets it. Use an [Assignment Rule](../automation/automation.md) to assign documents automatically.

## Sharing

Sharing gives one user access to one document, even if the user's role has no access to the DocType.

1. Open the document.
2. In the sidebar, click **+** next to **Share**.
3. In **Share With**, select a user.
4. Select the rights: **Read**, **Write**, **Submit** and **Share**.
5. Click **Add**.

Select **Everyone** to share the document with all users. You can share only if you have the share right on the document. You can only give the rights that you have yourself. The user gets a `Share` notification.

Each share is a `DocShare` document. A System Manager can turn off sharing for the site in System Settings.

!!! note "For developers"
    Share from code with `frappe.share.add(doctype, name, user, read=1, write=0, submit=0, share=0, everyone=0, notify=0)`. Read shares with `frappe.share.get_users(doctype, name)`. Refer to [Permissions](../../developer-guide/security/permissions.md).

## Comments and mentions

Every user who can read a document can comment on it.

1. Type in the comment box below the form.
2. To mention a user, type `@` and select the user.
3. Click **Comment**.

The comment shows in the activity of the document. You can edit your own comments. Administrator can edit all comments. You can delete a comment if you can delete `Comment` and you wrote it or have the System Manager role.

A mentioned user gets a `Mention` notification. A user gets mentions only if **Allowed In Mentions** is on for the user and the user is an enabled System User.

!!! note "For developers"
    A comment is a `Comment` document with `comment_type` = `Comment`. The desk calls `frappe.desk.form.utils.add_comment`. Mentions are handled by `frappe.desk.notifications.notify_mentions`.

## Likes

Click the heart in the form sidebar or on a list row to like a document. Click it again to remove the like. The list view header has a heart that shows only the documents you liked. Likes are stored in the `_liked_by` field and show in the activity.

## Follow

Follow a document to get emails about its changes.

1. Turn on **Send Notifications For Documents Followed By Me** in your User, in the **Document Follow** section.
2. Select a **Frequency**: **Hourly**, **Daily** or **Weekly**.
3. Open a document and select **Follow** in the form menu.

**Follow** shows only for DocTypes with **Track Changes**. Administrator cannot follow documents. Each follow is a `Document Follow` document. Select **Unfollow** to stop.

You can also follow documents automatically. Turn on these options in your User:

- **Auto follow documents that you create**
- **Auto follow documents that you comment on**
- **Auto follow documents that you Like**
- **Auto follow documents that are shared with you**
- **Auto follow documents that are assigned to you**

!!! note "For developers"
    The scheduler sends the follow emails with `frappe.desk.form.document_follow.send_hourly_updates`, `send_daily_updates` and `send_weekly_updates`. Follow from code with `frappe.desk.form.document_follow.follow_document(doctype, doc_name)`.

## Tags

A tag is a label on a document, for example `urgent`.

- In the form sidebar, click **+** next to **Tags** and type a tag.
- In the list view, select documents and click **Actions** > **Add Tags**.

To find tagged documents, type `#` and the tag in the awesomebar, or use a **Tags** quick filter in the list view. To show tags on list rows, turn on **Show Tags** in list settings.

Tags are stored in the `_user_tags` column of the document, and in `Tag` and `Tag Link` documents.

## Activity

The **Activity** section of a form shows the history of the document: creation, changes, attachments, likes, assignments, emails and comments. Field changes show only if the DocType has **Track Changes**. Refer to [Form view](form-view.md#comments-and-activity).

## Notification log

Each notification for a user is a `Notification Log` document. The notification types are `Alert`, `Share`, `Assignment`, `Mention` and `Energy Point` (DocType `Notification Type`).

To read notifications:

1. Click the bell in the dock. Opening the panel removes the dot from the bell.
2. Click a notification to open the document.
3. Click the check icon (**Mark all as read**) to mark all notifications as read. A click on one notification does not mark it as read.
4. Click **See all Activity** at the bottom of the list to open the **Notification Log** list.

### Notification settings

Click the gear icon in the notifications panel to open your **Notification Settings**.

| Field | Label | Default | What it does |
|---|---|---|---|
| `enabled` | Enable System Notification | 1 | Shows notifications in the desk. |
| `enable_email_notifications` | Enable Email Notifications | 1 | Also sends notifications by email. |
| `enable_email_mention` | Mentions | 1 | Emails for mentions. |
| `enable_email_assignment` | Assignments | 1 | Emails for assignments. |
| `enable_email_share` | Document Share | 1 | Emails for shares. |
| `enable_email_event_reminders` | Event Reminders | 1 | Emails for event reminders. |
| `enable_email_threads_on_assigned_document` | Email Threads on Assigned Document | 1 | Emails for email threads on documents assigned to you. |

Refer to [Notifications](../automation/notifications.md) to send your own notifications with the `Notification` DocType.

## Related

- [Form view](form-view.md)
- [List view](list-view.md)
- [Notifications](../automation/notifications.md)
- [Automation](../automation/automation.md)
- [Permissions](../users-and-permissions/permissions.md)
- [Email](../automation/email.md)
