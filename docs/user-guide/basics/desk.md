---
title: Desk
description: Find your way around the Frappe v15 desk at /app - login, navbar, workspace sidebar, workspaces, awesomebar, notifications, user menu and keyboard shortcuts.
version: v15
sources:
  - frappe/auth.py
  - frappe/apps.py
  - frappe/www/app.py
  - frappe/www/apps.py
  - frappe/hooks.py
  - frappe/public/js/frappe/ui/toolbar/toolbar.js
  - frappe/public/js/frappe/ui/toolbar/navbar.html
  - frappe/public/js/frappe/views/workspace/workspace.js
  - frappe/public/js/frappe/ui/notifications/notifications.js
  - frappe/public/js/frappe/ui/keyboard.js
  - frappe/public/js/frappe/ui/alt_keyboard_shortcuts.js
  - frappe/public/js/frappe/desk.js
  - frappe/desk/doctype/workspace/workspace.py
  - frappe/core/doctype/user/user.py
  - frappe/core/doctype/user/user.json
  - frappe/boot.py
---

# Desk

The desk is the back-office user interface of a Frappe site. System Users use it to read, create and change documents. It opens at `/app`. Use this page to learn the parts of the desk screen and how to move between them.

!!! note "Version"
    In Frappe v16, the desk moved to `/desk` and has a dock. In v15, `/desk` redirects to `/app`.

## Log in

1. Open `<site>/login` in the browser.
2. Type your email (or username) and password.
3. Click **Login**.

After login, a System User goes to the desk. `LoginManager.set_user_info` in `frappe/auth.py` picks the target:

1. If the user has a **Default Workspace** (`default_workspace`), go to `/app/<workspace>`.
2. Else, if a default app is set, go to the route of that app (`get_default_path()` in `frappe/apps.py`).
3. Else, go to `/app`.

A Website User does not get desk access and goes to the website instead.

## Desk layout

The desk screen has a navbar at the top, a sidebar on the left and the page in the main area.

![Desk at /app with the navbar, the workspace sidebar and the Build workspace](../../assets/images/user-guide/desk-layout.png)

| Part | Where | What it does |
|---|---|---|
| Navbar | Top | The logo (goes home), breadcrumbs, the awesomebar, the notification bell, the **Help** menu and your avatar. |
| Sidebar | Left | On a workspace: the list of workspaces. On a list or form: the list or form sidebar. |
| Page | Main area | Shows the workspace, list, form, report or page that you opened. |

Click the menu icon next to the page title to hide or show the sidebar.

### Workspaces

A workspace is a page with shortcuts, link cards, charts, number cards and other blocks. The sidebar lists the workspaces in two groups:

- **Public**: workspaces for all users.
- **Personal**: your private workspaces.

A workspace with a parent shows nested under its parent. Click a workspace to open it, for example `/app/build`.

When you open `/app`, the desk shows, in this order: your **Default Workspace**, the last workspace you opened, or the first workspace in the sidebar.

The buttons at the bottom of a workspace are:

| Button | What it does |
|---|---|
| **Edit** | Makes the blocks editable. Add blocks, drag them to move them, then click **Save** or **Discard**. |
| **New** | Opens the **New Workspace** dialog: **Title**, **Parent**, **Public**, **Icon** and **Indicator color**. |

In edit mode, each workspace in the sidebar has a settings menu with **Edit**, **Duplicate**, **Hide** and **Delete**. Only a user with the `Workspace Manager` role can edit, sort or make a public workspace. Other users can edit their personal workspaces and duplicate public ones.

### Apps screen

The page `/apps` shows one tile for each installed app that sets the `add_to_apps_screen` hook. Open it from **Apps** in the user menu. Frappe itself has no tile. On a site with only Frappe installed, `/apps` redirects to `/app`.

!!! note "For developers"
    An app adds its tile with the `add_to_apps_screen` hook in `hooks.py` (keys `name`, `logo`, `title`, `route`, `has_permission`). See [Workspaces and sidebar](../../developer-guide/desk-ui/workspaces-sidebar-dock.md).

## Search

Click the search box in the navbar, or press <kbd>Ctrl</kbd>+<kbd>G</kbd>, to open the awesomebar. Type a DocType name, a report name or `new <DocType>`. Refer to [Search](search.md).

## Notifications

Click the bell in the navbar to open the notifications panel. The bell shows a mark when you have unread notifications.

The panel has three tabs:

- **Notifications**: your notifications, for example assignments, mentions, shares and energy points.
- **Events**: your upcoming events.
- **What's New**: items from the changelog feed (`Changelog Feed`).

At the top of the panel, click the gear icon to open your **Notification Settings**. Click the check icon to mark all notifications as read. Click **View Full Log** to open the **Notification Log** list. Refer to [Collaboration](collaboration.md#notification-log).

## Help menu

Click **Help** in the navbar. The default items are **About**, **Keyboard Shortcuts** and **Frappe Support**. A System Manager changes these items in **Navbar Settings**.

## User menu

Click your avatar at the right of the navbar.

| Item | What it does |
|---|---|
| **My Profile** | Opens your user profile page (`/app/user-profile`). |
| **My Settings** | Opens your `User` form. |
| **Session Defaults** | Sets default values, for example a default company, for this session. |
| **Reload** | Clears the cache and reloads the desk. |
| **View Website** | Opens the website home page. |
| **Apps** | Opens the apps screen at `/apps`. |
| **Toggle Full Width** | Changes the page between fixed width and full width. |
| **Toggle Theme** | Opens the theme switcher (Light, Dark or Automatic). |
| **Log out** | Logs you out. |

A System Manager changes these items in **Navbar Settings**.

### Desk settings of a user

These fields on the `User` DocType change the desk for one user. A System Manager sets them in the **Settings** tab of the User form, in the sections **Desk Settings**, **Navigation Settings**, **List Settings** and **Form Settings**.

| Field | Label | Default | Effect |
|---|---|---|---|
| `desk_theme` | Desk Theme | none | `Light`, `Dark` or `Automatic`. |
| `mute_sounds` | Mute Sounds | 0 | Turns off desk sounds. |
| `search_bar` | Search Bar | 1 | Shows the awesomebar. |
| `notifications` | Notifications | 1 | Shows the notification bell. |
| `list_sidebar` | Sidebar (List Settings) | 1 | Shows the sidebar in list views. |
| `bulk_actions` | Bulk Actions | 1 | The desk in this version does not read this field. |
| `view_switcher` | View Switcher | 1 | Shows the view switcher in list views. |
| `form_sidebar` | Sidebar (Form Settings) | 1 | Shows the sidebar in forms. |
| `form_navigation_buttons` | Navigation Buttons | 1 | Shows the previous and next buttons in forms. |
| `timeline` | Timeline | 1 | Shows the activity timeline in forms. |
| `dashboard` | Dashboard | 1 | Shows the connections dashboard in forms. |

The desk reads these values at boot (`get_desk_settings()` in `frappe/boot.py`). Reload the desk after a change.

## Keyboard shortcuts

Press <kbd>Shift</kbd>+<kbd>/</kbd> (the <kbd>?</kbd> key), or select **Help** > **Keyboard Shortcuts**, to see the shortcuts for the current page. On macOS, <kbd>Cmd</kbd> works in place of <kbd>Ctrl</kbd>.

![Keyboard Shortcuts dialog on a list view](../../assets/images/user-guide/desk-keyboard-shortcuts.png)

These shortcuts work on every desk page:

| Shortcut | Action |
|---|---|
| <kbd>Ctrl</kbd>+<kbd>S</kbd> | Do the primary action of the page, for example **Save**. |
| <kbd>Ctrl</kbd>+<kbd>G</kbd> | Open the awesomebar. |
| <kbd>Ctrl</kbd>+<kbd>H</kbd> | Go to the desk home. |
| <kbd>Alt</kbd>+<kbd>S</kbd> | Open the user menu. |
| <kbd>Alt</kbd>+<kbd>H</kbd> | Open the **Help** menu. |
| <kbd>Shift</kbd>+<kbd>/</kbd> | Show keyboard shortcuts. |
| <kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>R</kbd> | Clear cache and reload. |
| <kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>G</kbd> | Switch theme. |
| <kbd>Esc</kbd> | Close the open dialog or the open child table row. |

On a workspace page, type any letter to open the awesomebar with that letter.

Hold <kbd>Alt</kbd> to see underlined letters in menus and the sidebar. Press <kbd>Alt</kbd> and the letter to click that item.

Shortcuts for lists and forms are on [List view](list-view.md#keyboard-shortcuts) and [Form view](form-view.md#keyboard-shortcuts).

!!! note "For developers"
    Add a shortcut with `frappe.ui.keys.add_shortcut({shortcut, action, description, page, condition, ignore_inputs})` (`frappe/public/js/frappe/ui/keyboard.js`). Pass `page` to limit the shortcut to one page. Only shortcuts with a `description` show in the dialog. A menu item gets a shortcut through the `shortcut` argument of `page.add_menu_item()`.

## Related

- [List view](list-view.md)
- [Form view](form-view.md)
- [Search](search.md)
- [Collaboration](collaboration.md)
- [Workspaces and sidebar](../../developer-guide/desk-ui/workspaces-sidebar-dock.md)
- [Users and roles](../users-and-permissions/users-and-roles.md)
