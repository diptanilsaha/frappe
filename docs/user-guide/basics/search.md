---
title: Search
description: Find pages and documents with the awesomebar, global search, list filters and tags, and control what global search indexes.
version: v16
sources:
  - frappe/public/js/frappe/ui/toolbar/awesome_bar.js
  - frappe/public/js/frappe/ui/toolbar/search_utils.js
  - frappe/public/js/frappe/ui/toolbar/search.js
  - frappe/public/js/frappe/ui/toolbar/tag_utils.js
  - frappe/public/js/frappe/ui/keyboard.js
  - frappe/public/js/frappe/list/base_list.js
  - frappe/desk/search.py
  - frappe/utils/global_search.py
  - frappe/desk/doctype/global_search_settings/global_search_settings.py
  - frappe/desk/doctype/global_search_settings/global_search_settings.json
  - frappe/search/sqlite_search.py
  - frappe/hooks.py
---

# Search

The desk has three ways to search. Use the awesomebar to open a page or start a new document. Use global search to find text in documents of many DocTypes. Use the list view filters to find documents of one DocType.

## Awesomebar

Press <kbd>Ctrl</kbd>+<kbd>K</kbd>, or click **Search** in the dock, to open the awesomebar. Type at least two characters. Use <kbd>Up</kbd> and <kbd>Down</kbd> to select a result and <kbd>Enter</kbd> to open it. Hold <kbd>Ctrl</kbd> while you select a result to open it in a new tab.

![Awesomebar with the results for "new sam"](../../assets/images/user-guide/search-awesomebar.png)

| You type | You get |
|---|---|
| A DocType name, for example `sample order` | **Sample Order List** and other matching DocTypes. |
| `new` and a DocType, for example `new sample` | **New Sample Order**. It opens a new document. |
| A text, `in` and a DocType, for example `customer a in sample order` | **Find customer a in Sample Order**. It opens the list filtered by name. |
| A report, page, workspace or dashboard name | **Report ...**, **Open ...**, **Open ... Workspace** or **... Dashboard**. |
| A calculation, for example `= 12 * 4` or `12*4` | The result. |
| Text with `random` | **Generate Random Password**. |
| `#` and a tag, for example `#urgent` | Documents with that tag. |
| Any other text | **Search for ...**. It opens global search. |

On a list view, the awesomebar also offers **Find ... in ...** for the current DocType. It filters by the title field.

With an empty box, the awesomebar shows your recent pages and your most used links.

Turn off the awesomebar for a user with **Search Bar** (`search_bar`) on the User.

!!! note "For developers"
    Add results with the `awesomebar_search` hook. Each method gets `txt` and returns a list of dicts with `label`, `route` and optional `description`, `index` and `route_options`. The desk calls `frappe.desk.search.awesomebar_search` and shows the first 20 results of each method. A JavaScript function can show as a result with `frappe.search.utils.make_function_searchable(fn, label)`.

## Global search

Global search finds text in the fields of many DocTypes at the same time.

1. Press <kbd>Ctrl</kbd>+<kbd>G</kbd>. You can also type a text in the awesomebar and select **Search for ...**.
2. Type the text.
3. Click a DocType above the results to see results of that DocType only. Click **More** to see all DocTypes. Pin a DocType to keep it in the row.
4. Click a result to open the document.

To match more than one term, join the terms with `&`, for example `Customer A&Laptop`. Start with `#` to search by tag.

Global search shows only documents that you can read. Press <kbd>Ctrl</kbd>+<kbd>K</kbd> to go back to the awesomebar.

### What global search finds

- Only DocTypes in **Global Search Settings** > **Search Priorities**. A System Manager opens the settings from the gear icon in the global search dialog.
- Only fields with **In Global Search** (`in_global_search`) set. Set it in the DocType or in **Customize Form**. Fields of child tables also count.
- The document name too, if the DocType has **Make "name" searchable in Global Search** (`show_name_in_global_search`).
- Not cancelled documents, and not documents with `disabled` set or `enabled` not set.

When you save a document, the desk puts its text in a queue. A scheduled job (`frappe.utils.global_search.sync_global_search`) writes the queue to the `__global_search` table every 15 minutes. A new or changed document can take up to 15 minutes to show in the results.

!!! note "For developers"
    An app adds its DocTypes to the default search priorities with the `global_search_doctypes` hook. Set `disable_global_search` in `site_config.json` to stop indexing. The search API is `frappe.utils.global_search.search(text, start, limit, doctype)`. Refer to [Site config](../../reference/configuration/site-config.md).

## Search in a list

In the [list view](list-view.md), type in the quick filters at the top. The **ID** filter and text filters match a part of the value by default (**Like**). Click the match button next to the filter to change to **Equals**. Click **Filter** to search in any field.

In Link fields, type to search the linked DocType. The search uses the name, the title field and the **Search Fields** of the linked DocType.

## Tags

Tags are labels that users add to documents. To find documents by tag:

- Type `#` and the tag in the awesomebar or in global search.
- In the list view, add **Tags** with **Customize Quick Filters**, then select a tag.

Refer to [Collaboration](collaboration.md#tags).

## Full-text search for apps

Frappe also has a full-text search framework based on SQLite FTS5 in `frappe/search/sqlite_search.py`. The desk search does not use it. An app makes a subclass of `SQLiteSearch`, lists it in the `sqlite_search` hook, and builds its own search UI and API on it.

Frappe keeps the index current with hooks in `frappe/hooks.py`:

- `doc_events` `on_update` calls `frappe.search.sqlite_search.update_doc_index`. It indexes the document again when an indexed field changed.
- `doc_events` `on_trash` calls `frappe.search.sqlite_search.delete_doc_index`.
- A scheduled job every 3 hours calls `frappe.search.sqlite_search.build_index_if_not_exists`.
- `after_migrate` calls `frappe.search.sqlite_search.build_index_in_background`.

## Related

- [Desk](desk.md)
- [List view](list-view.md)
- [Collaboration](collaboration.md)
- [Customize Form](../customization/customize-form.md)
- [Hooks](../../developer-guide/building-apps/hooks.md)
