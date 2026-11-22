---
title: Studio Development
description: Store Studio apps in an app repo, sync and export them, build them with bench, and pass boot data with the studio_app_boot hook.
version: v16
sources:
  - frappe/studio/sync.py
  - frappe/studio/export.py
  - frappe/studio/build.py
  - frappe/studio/utils.py
  - frappe/studio/watch.py
  - frappe/studio/sync_json.py
  - frappe/studio/api.py
  - frappe/studio/commands/__init__.py
  - frappe/studio/doctype/studio_app/studio_app.py
  - frappe/studio/doctype/studio_page/studio_page.py
  - frappe/studio/doctype/studio_component/studio_component.py
  - frappe/studio/ai/api.py
  - frappe/studio/ai/llm.py
  - frappe/commands/utils.py
  - frappe/hooks.py
  - studio/AGENTS.md
  - skills/studio-app-building/SKILL.md
  - skills/studio-app-building/ROUTING.md
  - hooks.md
---

# Studio Development

Studio is the visual app builder of Frappe. A Studio app is a `Studio App` document with `Studio Page` documents and source files. An app developer can store a Studio app in an app repo, so it installs and builds with the app. This page shows the file layout, how sync and export work, how to build, and the hooks. For the editor itself, see [Studio](../../user-guide/ai-and-app-builder/studio.md).

## Core concepts

| Name | What it is |
|---|---|
| `Studio App` | The app. Fields: `app_name`, `app_title`, `route`, `app_home`, `favicon`, `is_standard`, `frappe_app`. |
| `Studio Page` | One page. Fields: `page_title`, `route`, `tab_title`, `published`, `allow_guest`, `blocks`, `draft_blocks`, `resources`, `variables`, `script`. Its name is `page-<hash>`. |
| `Studio Component` | A reusable block tree with a script. With `studio_app` set, its name is `<studio_app>/<component_id>`. Without it, the component is global. |
| `Studio App File` | A source file of a custom app, stored in the database. |

There are two kinds of Studio app:

| Detail | Custom app (`is_standard` 0) | Standard app (`is_standard` 1) |
|---|---|---|
| Source | `Studio App File` records and the `script` field of each page | Files in `<app repo>/studio/<studio_app>/` |
| Languages | `.js`, `.json`, `.css` | `.js`, `.json`, `.css`, `.ts`, `.vue` |
| Edit | In Studio, on any site | The files, or Studio in developer mode |
| Build output | `sites/<site>/public/files/app_builds/<studio_app>/` | `<app>/public/app_builds/<studio_app>/` |
| Served from | `/files/app_builds/<studio_app>/` | `/assets/<app>/app_builds/<studio_app>/` |

An app ships standard apps. The rest of this page is about standard apps.

## File layout in an app repo

The `studio` folder is at the root of the app repo, next to the Python package. The folder name of each Studio app is `frappe.scrub(<studio app name>)`, so `my-app` is stored in `my_app`.

```text
apps/sample_app/
  sample_app/                          # the Python package
  studio/
    sample_portal/
      sample_portal.json               # the Studio App document
      tsconfig.json                    # written by Studio, maps @app/* to this folder
      router.ts                        # optional app router config
      tailwind.config.js               # optional extra Tailwind content paths
      components/OrderBadge.vue        # optional Vue components
      studio_page/
        orders/
          orders.json                  # the Studio Page document
          orders.ts                    # optional page script
      studio_components/
        order_card.json                # a Studio Component
        order_card.js                  # its script
```

Studio treats a subfolder of `studio/` as an app only if it holds `<folder>/<folder>.json`. Other folders are ignored.

The app JSON is a normal document export:

```json
{
 "app_home": "page-8599f810",
 "app_name": "sample_portal",
 "app_title": "Sample Portal",
 "doctype": "Studio App",
 "frappe_app": "sample_app",
 "is_standard": 1,
 "name": "sample_portal",
 "route": "portal"
}
```

Rules for the files:

- `name` in a page JSON is the scrubbed page title. `page_name` is the document ID. Do not change `page_name`.
- When you change a page title on disk, rename its folder, JSON file and script file to `frappe.scrub(page_title)`.
- A page script lives only in its `.ts` or `.js` file. The `script` field of an exported page stays empty.
- Keep one script file per module. Studio rejects a write that makes both `orders.ts` and `orders.js`.
- `app_home` holds a page document ID.

## Create a standard app

1. Turn on `developer_mode` in the site config.
2. Create the app in Studio, or insert a `Studio App` with `is_standard` set to 1 and `frappe_app` set to an installed app.
3. Save. Studio writes the export folder.

To convert a custom app, call `enable_app_export(target_app)` on the `Studio App` document. It sets `is_standard` and `frappe_app`, writes the JSON files, and moves the source files to disk. `disable_app_export(confirm_discarded=False)` makes the app custom again. It stops when files that a custom app cannot keep (such as `.ts` and `.vue`) exist, unless you confirm their deletion.

!!! warning
    Do not copy an export folder by hand to change the app type. A copy does not update `is_standard` or `frappe_app`.

## Export and sync

Export writes documents to disk. It runs in `on_update` of `Studio App`, `Studio Page` and `Studio Component`, only when `can_export` in `frappe/studio/export.py` is true:

- The document is standard.
- `developer_mode` is on.
- The site is not installing, uninstalling or importing.

Outside developer mode, a save of a standard app, page or component fails with "Standard apps can only be edited in developer mode." Install, migrate, import and patches can still write them.

Sync reads files into the database:

| When | Function | What it does |
|---|---|---|
| `bench migrate` | `frappe.studio.sync.after_migrate` | Imports the app, page and component JSON of every installed app. Deletes standard apps and pages whose files are gone. Rebuilds published custom apps. |
| App install | `frappe.studio.sync.after_app_install` | Imports the Studio apps of the installed app. |
| App uninstall | `frappe.studio.sync.before_app_uninstall` | Deletes the Studio apps whose `frappe_app` is the app. |
| `bench --site <site> watch-studio` | `frappe.studio.watch.watch` | Imports changed JSON and component scripts while it runs. |

`watch-studio` needs the `watchdog` package, a dev dependency. It finds `studio/` folders when it starts, so restart it after you add a folder. It does not sync deletions. Run it next to `bench start`, which runs the Vite dev server of Studio. The dev server serves page scripts and `.vue` files to the editor and preview.

## Build

| Command | Description |
|---|---|
| `bench build --app sample_app` | Builds the app assets. Then `frappe.studio.build.after_app_build` builds every standard Studio app of the built apps. A failed Studio build fails the command. |
| `bench --site <site> build-studio-app <studio_app>` | Builds one Studio app. It exits with code 1 on failure and logs the error in Error Log. |

You do not need a build to edit. The preview at `/dev/<app_route><page_route>` uses the editor renderer and shows drafts and unpublished pages. The user needs read access on `Studio Page`.

The published route `/<app_route>` uses the build. It serves an app only when the app has a published page. A guest can open it only when a published page has `allow_guest`. A guest who cannot open it is redirected to `/login`. Without a build, the published route falls back to the editor renderer.

The build output is generated. You can keep it out of git, for example with `sample_app/public/app_builds` in `.gitignore`.

## Hooks

Frappe registers these hooks for Studio in `frappe/hooks.py`:

| Hook | Value | Description |
|---|---|---|
| `page_renderer` | `frappe.studio.doctype.studio_app.studio_app.StudioAppRenderer` | Serves `/<app_route>` and `/dev/<app_route>`. |
| `website_route_rules` | `/studio/<path:app_path>` to `studio` | Opens the Studio editor. |
| `after_app_build` | `frappe.studio.build.after_app_build` | Builds the standard Studio apps of the built apps. |

An app can add one hook:

| Hook | Shape | Read by | Description |
|---|---|---|---|
| `studio_app_boot` | dict of Studio app name to dotted path | `StudioApp.get_boot` | The function returns a dict. The page gets it as the global `boot` before the first navigation. |

```python
# sample_app/hooks.py
studio_app_boot = {"sample_portal": "sample_app.studio_boot.get_boot"}
```

```python
# sample_app/studio_boot.py
import frappe


def get_boot() -> dict:
	return {
		"user_full_name": frappe.utils.get_fullname(frappe.session.user),
		"is_guest": frappe.session.user == "Guest",
	}
```

If several apps set a function for one Studio app, the last one wins. An exception is logged in Error Log and `boot` is empty. The whitelisted method `frappe.studio.api.get_app_boot(app_name)` returns the same dict.

Page scripts, bindings and the router config read `boot.<key>`. Keys can be missing, so use `?.` for nested keys.

## App router config

Put app-wide aliases, redirects and guards in `router.ts` or `router.js` at the root of the export folder. Use `export default`. Studio reads three optional keys: `routerOptions` (options for `createRouter`, except `routes` and `history`), `extendRoute(route)` and `setup(router)`.

```js
// studio/sample_portal/router.js
export default {
	routerOptions: { scrollBehavior: () => ({ top: 0 }) },
	setup(router) {
		router.beforeEach((to) => {
			if (boot.is_guest && to.name !== "Login") {
				return { name: "Login" }
			}
		})
	},
}
```

Each page is one route. The route `name` is the page title.

## File API

Use the whitelisted methods in `frappe/studio/api.py` to read and write source files. They work for both app types and take `frappe_app` and `studio_app`.

| Method | Description |
|---|---|
| `list_studio_files` | List the source files. |
| `read_studio_file` | Read one file. Returns `path`, `content` and `hash`. |
| `write_studio_file` | Write a file. Pass `known_hash` to reject the write when the file changed. |
| `create_studio_file` | Create an empty file. Fails if it exists. |
| `create_studio_folder` | Create a folder. |
| `rename_studio_file` | Rename a file or change its extension. |
| `delete_studio_file` | Delete a file. |

The write methods need write permission on `Studio Page`. They reject writes to the app, page and component JSON that Studio generates. Change those through the document fields.

## AI in Studio

The Studio editor has its own AI agent, separate from Flow. Its endpoints are in `frappe/studio/ai/api.py`. It reads an OpenRouter key from the `ai_api_key` field of `Studio Settings`. Without the key, the `run` endpoint fails. The frappe repo ships the Flow Skill `frappe/studio-app-building`, which tells Flow agents how to build Studio pages. See [Flow Development](flow-development.md).

## Pitfalls

- Saving a standard document in Studio writes the file on disk. Wait for the watcher to sync disk changes before you edit the same document in Studio.
- `bench migrate` deletes standard apps and pages whose files are gone. Do not delete an export folder you still need.
- A custom app cannot use `.ts` or `.vue` files, or packages outside `vue`, `vue-router`, `pinia` and `frappe-ui`. Export the app if it needs them.
- In a standard app, Tailwind scans only `studio/<studio_app>/`. Add a `tailwind.config.js` with `content` globs for other paths.
- The app name must be lowercase with hyphens. It is used in the editor URL and the export folder.

## Related

- [Studio](../../user-guide/ai-and-app-builder/studio.md)
- [Apps](../building-apps/apps.md)
- [Hooks](../building-apps/hooks.md)
- [Hooks reference](../../reference/configuration/hooks.md)
- [Website](../website/website.md)
- [CLI reference](../../reference/configuration/cli.md)
- [Islands](../desk-ui/islands.md)
