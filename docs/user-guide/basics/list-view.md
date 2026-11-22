---
title: List View
description: Find, filter, sort and select documents in the Frappe v15 list view, use the group-by sidebar and saved filters, run bulk actions and set list settings.
version: v15
sources:
  - frappe/public/js/frappe/list/list_view.js
  - frappe/public/js/frappe/list/base_list.js
  - frappe/public/js/frappe/list/list_settings.js
  - frappe/public/js/frappe/list/list_sidebar.js
  - frappe/public/js/frappe/list/list_sidebar.html
  - frappe/public/js/frappe/list/list_sidebar_group_by.js
  - frappe/public/js/frappe/list/list_filter.js
  - frappe/public/js/frappe/list/list_view_select.js
  - frappe/public/js/frappe/list/bulk_operations.js
  - frappe/public/js/frappe/ui/sort_selector.js
  - frappe/public/js/frappe/utils/common.js
  - frappe/desk/doctype/list_view_settings/list_view_settings.json
  - frappe/desk/doctype/list_filter/list_filter.json
  - frappe/desk/doctype/bulk_update/bulk_update.py
  - frappe/desk/listview.py
  - frappe/desk/reportview.py
  - frappe/desk/form/meta.py
  - frappe/model/meta.py
---

# List View

The list view shows the documents of one DocType as rows. Use it to find documents, filter and sort them, and change many documents at the same time. Open it from a workspace, from the awesomebar, or at `/app/<doctype-slug>`, for example `/app/sample-order`.

![List view of Sample Order with the sidebar on the left](../../assets/images/user-guide/list-view.png)

## Parts of the list view

| Part | What it shows |
|---|---|
| Title | The DocType name, for example **Sample Order**. Click the menu icon next to it to hide or show the sidebar. |
| View switcher (**List View**) | Changes to another view. Refer to [Views](views.md). |
| Reload button | Loads the list again. |
| **...** menu | List and DocType settings. Refer to [List menu](#list-menu). |
| **Add Sample Order** | Creates a document. |
| Sidebar | **Filter By** (group-by counts), **Tags**, **Show Tags** and **Save Filter**. |
| Standard filters | Fields above the rows, for example **ID**, **Customer Name** and **Status**. |
| **Filter** | Adds filters on any field. The **x** next to it removes all filters. |
| Sort button | Sets the sort field and the sort order. |
| Rows | One row for each document. The right side of the row shows the assigned users, the time of the last change, the comment count and a heart (like). |
| Count | For example `6 of 6`: rows loaded and total rows that match the filters. |
| Page size | **20**, **100**, **500** or **2500** rows. |

Click the title of a row to open the document in the [form view](form-view.md).

## Filter

### Standard filters

Standard filters show above the list. Type or select a value to filter at once. These fields show as standard filters:

- **ID** (`name`), unless the list settings set `hide_name_filter`.
- The title field of the DocType.
- Fields with **In Standard Filter** (`in_standard_filter`) checked.

The **ID** filter and text fields match a part of the value (`like`). Select and Link fields match the exact value. A Link to a tree DocType matches the value and its descendants.

### Filter button

1. Click **Filter**.
2. Select a field, a condition (for example **Equals**) and a value.
3. To add one more condition, click **+ Add a Filter**.
4. To remove all conditions, click **Clear Filters**, or the **x** next to **Filter**.

You can also click a value in a row, for example a status indicator, to filter by that value.

Filters show in the URL as query parameters, for example `/app/sample-order?status=Open`. Share the URL to share the filtered list. The desk also remembers your last filters for each DocType.

!!! note "Large tables"
    The desk treats a table as large when it has more than 100,000 rows (estimated) and a document was modified in the last 30 days. On a large table with no filters, the list adds a filter for documents modified in the last 90 days. A message tells you about it. Turn this off with **Disable Automatic Recency Filters** in [List settings](#list-settings).

## Sidebar

The list sidebar shows under the title. Turn it off for a user with **Sidebar** (`list_sidebar`) in the user's **List Settings** section.

### Filter By

**Filter By** has a dropdown for **Assigned To** and **Created By**. Each dropdown lists the values with the number of matching documents. Click a value to filter by it. **Me** stands for you.

To add more group-by fields:

1. Click **Edit Filters**.
2. In **Select Filters**, select the fields. Fields of type Select, Link, Data, Int and Check are listed.
3. Click **Save**.

The selected fields are saved in your user settings for this DocType (`group_by_fields`). The counts come from `frappe.desk.listview.get_group_by_count`.

### Tags

The **Tags** dropdown lists the tags of the DocType with counts. Click a tag to filter by it. Click **Show Tags** to show the tags of each document in its row.

### Saved filters

Save the current filters with a name:

1. Set the filters.
2. Under **Save Filter**, type a **Filter Name**.
3. Select **Is Global** to share the filter with all users.
4. Press <kbd>Enter</kbd>.

Click **Show Saved** to see the saved filters. Click a filter to apply it. Click the **x** on a filter to delete it. Saved filters are `List Filter` documents (`reference_doctype`, `filter_name`, `for_user`, `filters`). A global filter has no `for_user`.

## Sort

Click the field name on the sort button to select the sort field. Click the icon on its left to change between ascending and descending order.

The sort options are **Last Updated On**, **ID**, **Created On**, **Most Used** (`idx`), the title field, the default sort field of the DocType, and every field that is mandatory, bold or in the list view. The default sort is the **Default Sort Field** and **Default Sort Order** of the DocType. A new DocType sorts by `modified`.

## Select rows and run bulk actions

Select the check box of a row to select it. Select the check box in the header to select all loaded rows. When you select rows, the **Add** button changes to **Actions**.

![Actions menu with two Sample Items selected](../../assets/images/user-guide/list-view-actions.png)

| Action | Shows when | What it does |
|---|---|---|
| **Edit** | The DocType has editable fields | Sets one field to one value on all selected documents. |
| **Export** | Always | Exports the selected rows. |
| **Assign To** | Always | Assigns the documents to users. Refer to [Collaboration](collaboration.md#assignments). |
| **Clear Assignment** | Always | Removes all assignments. |
| **Apply Assignment Rule** | Always | Runs the assignment rules on the documents. |
| **Add Tags** | Always | Adds tags to the documents. |
| **Print** | You can print the DocType | Prints all selected documents in one PDF. |
| **Submit** | Submittable DocType, you can submit, no workflow | Submits the documents. |
| **Cancel** | You can cancel, no workflow | Cancels the documents. |
| **Delete** | You can delete | Deletes the documents after you confirm. |

If the DocType has a workflow, **Edit** and **Delete** need **Allow Bulk Editing** in list settings.

Large jobs run in the background:

| Action | Runs in the background when |
|---|---|
| Edit, Submit, Cancel | 20 or more documents. More than 500 documents is not allowed. |
| Delete | More than 10 documents. |

## Like, comments and assignments in rows

- Click the heart on a row to like the document. Click the heart in the header to show only documents that you liked.
- The comment icon shows the number of comments.
- The avatars show the assigned users.

## Paging and refresh

The list loads 20 rows. On a screen taller than 1180 pixels, it loads 100 rows. Select a page size at the bottom to load more rows.

The list updates when another user changes a document. It also reloads every 5 minutes while it is open. Turn off live updates with **Disable Auto Refresh** in list settings.

## List menu

Click **...** to open the list menu.

| Item | Shows for | What it does |
|---|---|---|
| **Refresh** | All users | Loads the list again. |
| **Import** | Users who can import the DocType | Opens **Data Import** for this DocType. |
| **User Permissions** | System Manager | Opens the User Permission list for this DocType. |
| **Role Permissions Manager** | System Manager | Opens the permissions of this DocType. |
| **Customize** (<kbd>Ctrl</kbd>+<kbd>J</kbd>) | Users who can create Custom Field and Property Setter | Opens **Customize Form**, or the DocType form for a custom DocType. |
| **Toggle Sidebar** (<kbd>Ctrl</kbd>+<kbd>K</kbd>) | All users | Hides or shows the sidebar. |
| **Edit DocType** | System Manager in developer mode | Opens the DocType form. |
| **List Settings** | System Manager | Opens [List settings](#list-settings). |

## List settings

**List Settings** changes the list for all users of the DocType. It saves a `List View Settings` document named after the DocType.

| Field | Label | What it does |
|---|---|---|
| `total_fields` | Maximum Number of Fields | The number of columns, from 4 to 10. |
| `fields` | Fields | The columns of the list. Click **+ Add / Remove Fields** to change them. |
| `disable_count` | Disable Count | Hides the total count. |
| `disable_comment_count` | Disable Comment Count | Hides the comment count in rows. |
| `disable_sidebar_stats` | Disable Sidebar Stats | Removes the **Tags** dropdown from the sidebar. |
| `disable_auto_refresh` | Disable Auto Refresh | Stops live updates of the list. |
| `allow_edit` | Allow Bulk Editing | Allows bulk **Edit** and **Delete** when the DocType has a workflow. |
| `disable_automatic_recency_filters` | Disable Automatic Recency Filters | Stops the 90 day filter on large tables. |

## Keyboard shortcuts

| Shortcut | Action |
|---|---|
| <kbd>Down</kbd> / <kbd>Up</kbd> | Move to the next or previous row. |
| <kbd>Shift</kbd>+<kbd>Down</kbd> / <kbd>Shift</kbd>+<kbd>Up</kbd> | Select rows while you move. |
| <kbd>Space</kbd> | Select the current row. |
| <kbd>Enter</kbd> | Open the current row. |
| <kbd>Ctrl</kbd>+<kbd>J</kbd> | Open **Customize**. |
| <kbd>Ctrl</kbd>+<kbd>K</kbd> | Hide or show the sidebar. |

## For developers

- Change the list of a DocType in `<doctype>_list.js` next to the DocType JSON. It sets `frappe.listview_settings["Sample Order"]` with keys such as `add_fields`, `filters`, `get_indicator`, `onload`, `button`, `formatters`, `primary_action`, `hide_name_column` and `hide_name_filter`. Another app extends it with the `doctype_list_js` hook. `frappe/desk/form/meta.py` loads both. Refer to [List view settings](../../developer-guide/desk-ui/list-view-settings.md).
- The columns come from fields with `in_list_view`. Standard filters come from fields with `in_standard_filter`. The default sort comes from `sort_field` and `sort_order` on the DocType.
- The list gets its rows from `frappe.desk.reportview.get`. Bulk delete calls `frappe.desk.reportview.delete_items`. Bulk submit, cancel and edit call `frappe.desk.doctype.bulk_update.bulk_update.submit_cancel_or_update_docs`.
- The large table check is `Meta.check_if_large_table()` in `frappe/model/meta.py`. It sets `is_large_table` on the meta.

## Related

- [Views](views.md)
- [Form view](form-view.md)
- [Search](search.md)
- [Collaboration](collaboration.md)
- [Customize Form](../customization/customize-form.md)
- [List view settings](../../developer-guide/desk-ui/list-view-settings.md)
