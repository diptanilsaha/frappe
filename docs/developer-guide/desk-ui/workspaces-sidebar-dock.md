---
title: Workspaces, Sidebar and Dock
description: Ship desk navigation from an app with Workspace, Sidebar and Dock JSON records and the add_to_apps_screen hook.
version: v16
sources:
  - frappe/desk/doctype/workspace/workspace.py
  - frappe/desk/doctype/workspace/workspace.json
  - frappe/desk/doctype/custom_workspace/custom_workspace.py
  - frappe/desk/doctype/sidebar/sidebar.py
  - frappe/desk/doctype/sidebar/sidebar.json
  - frappe/desk/doctype/sidebar_item/sidebar_item.json
  - frappe/desk/doctype/custom_sidebar/custom_sidebar.json
  - frappe/desk/doctype/dock/dock.py
  - frappe/desk/doctype/dock/dock.json
  - frappe/desk/doctype/dock_item/dock_item.json
  - frappe/desk/doctype/desktop_icon/desktop_icon.py
  - frappe/desk/doctype/desktop_settings/desktop_settings.py
  - frappe/desk/RETIRING.md
  - frappe/model/sync.py
  - frappe/commands/utils.py
  - frappe/apps.py
  - frappe/boot.py
  - frappe/hooks.py
  - frappe/dock/frappe/frappe.json
  - frappe/automation/sidebar/automation/automation.json
  - frappe/automation/workspace/automation/automation.json
  - frappe/public/js/frappe/ui/sidebar/sidebar.js
  - frappe/public/js/frappe/ui/sidebar/dock_manager.js
  - hooks.md
---

# Workspaces, Sidebar and Dock

The desk has three levels of navigation. The **dock** is the icon rail at the left edge. Each dock entry opens a **sidebar**, the list of links next to it. A sidebar link can open a **workspace**, a page with shortcuts, cards, charts and number cards. An app ships all three as JSON records in its folders. `bench migrate` imports them. The **apps screen** lists installed apps, and an app joins it with the `add_to_apps_screen` hook. For the user view, see [Desk](../../user-guide/basics/desk.md).

| Record | File in the app | Exported when |
|---|---|---|
| `Workspace` | `<app>/<module>/workspace/<name>/<name>.json` | Saved with `standard` = 1 and a `module`, in developer mode. |
| `Sidebar` | `<app>/<module>/sidebar/<name>/<name>.json`, or `<app>/sidebar/<name>/<name>.json` for a sidebar with no module | Saved with `standard` = 1, in developer mode. |
| `Dock` | `<app>/dock/<app>/<app>.json` | Saved with `standard` = 1, in developer mode. |

`<name>` is the scrubbed record name. During migrate, Frappe also deletes a standard `Workspace`, `Sidebar` or `Dock` record whose file is gone from the app.

## Workspaces

A workspace is a `Workspace` record. Its `content` field holds the page layout as a JSON list of blocks. The child tables hold the widgets that the blocks point to.

| Field | Description |
|---|---|
| `label`, `title` | Name of the workspace. The route is `/desk/<slug of the title>`, for example `/desk/automation`. |
| `module` | The module the workspace belongs to. Decides where it is exported and which sidebar lists it. |
| `icon`, `indicator_color` | Icon and color. |
| `public` | Visible to all users with access. A private workspace belongs to `for_user`. |
| `standard` | Shipped by an app. |
| `content` | JSON list of blocks: headers, paragraphs and widgets (`shortcut`, `card`, `chart`, `number_card`, `quick_list`, `custom_block`). |
| `shortcuts`, `links`, `charts`, `number_cards`, `quick_lists`, `custom_blocks` | Widget definitions. A block finds its row by label. |
| `roles` | Roles that can see the workspace. Empty means all. |
| `parent_page`, `sequence_id`, `is_hidden` | Nesting, order and visibility. |
| `type`, `link_type`, `link_to`, `external_link` | `Workspace` (default), or `Link` / `URL` for an entry that opens a DocType, Page, Report or URL. |

To make or change a standard workspace:

1. Turn on developer mode for the site.
2. Open the workspace in the desk and edit it.
3. Make sure `standard` is set and `module` points to a module of your app.
4. Save. Frappe writes the JSON file into the module folder.
5. Commit the file in your app.

Outside developer mode, a user cannot change the `content` of a standard workspace directly. The desk saves the change as a `Custom Workspace` record. That record holds the difference, so it survives app updates.

## Sidebars

A sidebar belongs to a module. There are two sources:

1. A `Sidebar` record that an app ships as JSON.
2. A computed sidebar, made from what the module holds: its workspaces, DocTypes, reports, dashboards and pages. Frappe uses it when the app ships no `Sidebar` for the module.

The desk then hides items the user cannot open, and applies `Custom Sidebar` records. These hold site and user changes on top of the base sidebar.

`Sidebar` fields:

| Field | Description |
|---|---|
| `title` | Name and record name (`autoname: field:title`). |
| `module` | Module that owns the sidebar. |
| `app` | App the sidebar is placed in. Derived from the module when set. |
| `header_icon` | Icon in the sidebar header. |
| `standard` | Shipped by an app. Needs developer mode to set. |
| `items` | Table of `Sidebar Item` rows. |

`Sidebar Item` fields:

| Field | Description |
|---|---|
| `type` | `Link`, `Section Break` or `Spacer`. |
| `label`, `icon` | Text and icon. |
| `link_type` | `DocType`, `Page`, `Report`, `Workspace`, `Dashboard` or `URL`. |
| `link_to` | The record the link opens. |
| `url` | Target of a `URL` link. |
| `route` | Path inside the page, for example `payroll` opens `/desk/<page>/payroll`. |
| `filters`, `route_options` | JSON filters or route options for the opened view. |
| `child`, `indent`, `collapsible`, `keep_closed` | Nesting under a Section Break. |
| `hidden` | Hide the item. |
| `open_in_new_tab` | Default 1. |

A shipped sidebar file looks like this (shortened):

```json
{
 "doctype": "Sidebar",
 "name": "Sample Module",
 "title": "Sample Module",
 "module": "Sample Module",
 "app": "sample_app",
 "header_icon": "package",
 "standard": 1,
 "items": [
  {"type": "Link", "label": "Sample Order", "link_type": "DocType", "link_to": "Sample Order", "icon": "list"},
  {"type": "Link", "label": "Sample Item", "link_type": "DocType", "link_to": "Sample Item", "icon": "box"}
 ]
}
```

To ship a sidebar:

1. Turn on developer mode for the site.
2. Open the module and select **Edit Sidebar** in the sidebar header.
3. Arrange the items.
4. Save to the layer named after your app (not "Just for me" or "For everyone"). Frappe saves the `Sidebar` record as standard and writes its JSON file.
5. Commit the file in your app.

### Old sidebar fixtures

In v16, the flat `<app>/workspace_sidebar/*.json` files are no longer imported. Convert them to per-module `Sidebar` files:

```bash
bench --site frappe.localhost convert-sidebar-fixtures --app sample_app --dry-run
bench --site frappe.localhost convert-sidebar-fixtures --app sample_app
```

The command does not overwrite existing files. Delete the old folder after you check the result. Until you convert, the modules of the app use computed sidebars. The `Workspace Sidebar` DocType stays only as an archive that this conversion reads. `Sidebar Item Group` belongs to that archive: it is a group row of a `Workspace Sidebar`, and the desk does not read it.

## Dock

The dock is a `Dock` record. An app ships one at `<app>/dock/<app>/<app>.json`. Its rows are the entries of the app on the rail. A module that the record does not name is not on the rail.

```json
{
 "doctype": "Dock",
 "name": "sample_app",
 "app": "sample_app",
 "standard": 1,
 "items": [
  {"link_type": "Sidebar", "link_to": "Sample Module", "icon": "package", "title": "Sample Module", "added": 0, "hidden": 0},
  {"link_type": "Workspace", "link_to": "Sample Reports", "icon": "chart", "title": "Reports", "added": 0, "hidden": 0},
  {"link_type": "URL", "url": "https://example.com/help", "icon": "help", "title": "Help", "added": 0, "hidden": 1}
 ]
}
```

`Dock` fields:

| Field | Description |
|---|---|
| `app` | The app whose dock this is. Required. |
| `standard` | The app's own dock, exported to the app. The record name is then the app name. |
| `user` | Empty for the app or site layer. A user name for that user's own arrangement. |
| `mount_on` | For a companion app: the host app whose rail gets this app's entries. Only on the standard record. |
| `items` | Table of `Dock Item` rows. Row order is the rail order. |

`Dock Item` fields:

| Field | Description |
|---|---|
| `link_type` | `Sidebar` (select that sidebar and open its landing route), `Workspace` (open that workspace and derive the sidebar from its module) or `URL` (leave the desk). |
| `link_to` | The sidebar or workspace name. |
| `url` | Target of a `URL` row. |
| `icon`, `title` | How the rail shows the entry. |
| `hidden` | Off by default. A site or a user can show it again in Manage Dock. |
| `added` | Set when the row adds a new entry, not a change to an entry from a lower layer. |

The site arrangement and each user's arrangement are stored as more `Dock` records on top of the app's dock. They can reorder, hide, relabel and add entries. They cannot change what an app row points to.

To author the dock:

1. Turn on developer mode for the site.
2. Open the user menu and select **Manage Dock**.
3. Arrange the entries.
4. Select **Export to app**. Frappe writes `<app>/dock/<app>/<app>.json`. Each later save updates the file.
5. Commit the file in your app.

If you delete the file and run `bench migrate`, the app has no dock. Its sidebar then shows an app switcher in the header.

## Apps screen: add_to_apps_screen

The `add_to_apps_screen` hook puts an app on the apps screen. It is a list with one dict per app.

```python
# sample_app/hooks.py
add_to_apps_screen = [
	{
		"name": "sample_app",
		"logo": "/assets/sample_app/images/logo.svg",
		"title": "Sample App",
		"route": "/desk/sample-module",
		"has_permission": "sample_app.api.has_app_permission",
	}
]
```

| Key | Description |
|---|---|
| `name` | App name. |
| `title` | Label on the apps screen. |
| `logo` | Logo URL. Falls back to the `app_logo_url` hook. |
| `route` | Where the icon goes. Falls back to the `app_home` hook. |
| `has_permission` | Dotted path of a function that returns `True` if the user may see the app. |
| `sequence_id` | Sort order. Lower shows first. Frappe uses 1000 to stay last. |
| `desk_route` | For an app whose UI is not the desk: a desk route shown as a second link, "Open in Desk". |

An app with `mount_on` in its dock does not get its own slot on the apps screen.

## Desktop icons

The `Desktop Icon` grid is being retired. Its rows are created, and `<app>/desktop_icon/*.json` fixtures are imported, only when **Desktop Settings** > `desktop_page` is `Desktop Icons`. The default is `Apps`. Use `add_to_apps_screen` and a `Dock` record in new apps.

## Pitfalls

- A standard record with no file in the app is deleted on the next `bench migrate`. Commit the exported files.
- Export works only in developer mode. On other sites, desk changes go to `Custom Workspace`, `Custom Sidebar` or site `Dock` records, not to files.
- A `Dock` row with `link_type` = `Sidebar` needs a matching `Sidebar` record or `Module Def`.
- The `get_desktop_icons` hook listed in old documentation is not read in v16.

## Related

- [Desk](../../user-guide/basics/desk.md)
- [Desk Pages](desk-pages.md)
- [Apps](../building-apps/apps.md)
- [Hooks](../building-apps/hooks.md)
- [Fixtures and Patches](../building-apps/fixtures-and-patches.md)
- [Hooks reference](../../reference/configuration/hooks.md)
