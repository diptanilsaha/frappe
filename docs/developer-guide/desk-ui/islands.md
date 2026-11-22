---
title: Islands
description: Draw desk pages and widgets with Vue islands, self-contained bundles that the desk mounts in a shadow root by name.
version: v16
sources:
  - frappe/utils/island.py
  - frappe/sessions.py
  - frappe/bundler.py
  - frappe/core/doctype/page/page.py
  - frappe/core/doctype/page/page.json
  - frappe/public/js/frappe/ui/island/loader.js
  - frappe/public/js/frappe/views/pageview.js
  - frappe/public/js/frappe/ui/chart.js
  - frappe/desk/page/profile/profile.island.js
  - frappe/desk/page/search/search.json
  - ui/README.md
  - ui/island/host.js
  - ui/island/mount.js
  - ui/island/decisions/README.md
  - ui/vite/island/pages.js
  - ui/vite/island/build-pages.js
  - island-toolchain/package.json
  - flow/build-islands.mjs
  - flow/package.json
  - esbuild/esbuild.js
---

# Islands

An island is a Vue UI unit that an app builds and the desk mounts in a shadow root. The island bundle carries its own Vue, frappe-ui and everything it imports, so the desk page needs nothing loaded first. Use an island when you want to write a desk page or widget in Vue with frappe-ui instead of jQuery. The desk pages `profile`, `search` and `notifications`, the Flow chat and desk charts are islands.

There are two ways to use islands:

| Way | You write | Who builds it | Island name |
|---|---|---|---|
| Page island | A `Page` with `type` = `Frappe UI`, plus a `.vue` file in the page folder | Frappe, in `bench build` | `<app>.page.<page name>` |
| App island | An entry file and a build script in your app frontend | Your app, from the `build` script of its `package.json` | Any name. Prefix it with the app name. |

## The island registry

An island is registered by its build. The build writes `<name>.island.js` (and `<name>.island.css`) into `sites/assets/assets.json`. `frappe/utils/island.py` reads that file and keeps only the islands of apps installed on the site. The list is in `frappe.boot.ui_islands`:

```js
frappe.boot.ui_islands;
// ["frappe.chart", "frappe.flow.chat", "frappe.page.notifications", "frappe.page.profile", "frappe.page.search"]
```

A host without desk boot can resolve a name with the whitelisted method `frappe.utils.island.get_island_assets(name)`. It returns `{js, css}`, or throws if the island is not on the site.

## Page islands

A page island draws the body of a desk page. The desk keeps its own page header and sets the title and menu from what the island reports.

1. Turn on developer mode for the site.
2. Create a `Page` as described in [Desk Pages](desk-pages.md). Set **Type** to `Frappe UI` and keep **Standard** = `Yes`.
3. Save. Frappe writes the starter files next to the page JSON.
4. Run `bench build`, or keep `bench watch` running.
5. Open `/desk/<page name>`.

```text
sample_app/sample_module/page/order_board/
├── order_board.json        # "type": "Frappe UI"
├── order_board.island.js   # the entry, written once
└── order_board.vue         # the component, yours to edit
```

The entry file exports `mount`:

```js
// sample_app/sample_module/page/order_board/order_board.island.js
import { mountVueIsland } from "@framework/ui/island";

import Page from "./order_board.vue";

export const mount = (el, context) => mountVueIsland(el, { ...context, component: Page });
```

The component gets the route below the page as props and reports its title and actions:

```vue
<!-- sample_app/sample_module/page/order_board/order_board.vue -->
<script setup>
import { onMounted } from "vue";
import { createResource, frappeRequest } from "frappe-ui";
import { useHost } from "@framework/ui/island";

defineProps({
	route: { type: Array, default: () => [] },
	query: { type: Object, default: () => ({}) },
});
const emit = defineEmits(["title", "actions"]);
const host = useHost();

const orders = createResource({
	url: "frappe.client.get_list",
	params: { doctype: "Sample Order", fields: ["name", "customer_name", "status"] },
	resourceFetcher: frappeRequest,
	auto: true,
});

onMounted(() => {
	emit("title", "Order Board");
	emit("actions", [
		{ label: "New Order", onClick: () => host.navigate("/desk/sample-order/new") },
	]);
});
</script>

<template>
	<div class="h-full overflow-y-auto p-5">
		<div v-for="order in orders.data" :key="order.name" class="text-sm text-ink-gray-8">
			{{ order.customer_name }}: {{ order.status }}
		</div>
	</div>
</template>
```

| Prop or event | Description |
|---|---|
| `route` (prop) | URL segments after the page name. |
| `query` (prop) | URL query parameters. |
| `title` (event) | A string or `null`. The desk shows it in the breadcrumb and browser tab. |
| `actions` (event) | A list of `{label, icon?, onClick}` or `{label, icon?, href}`. The desk shows each as a page menu row. An `href` opens in a new tab. |

A route change updates `route` and `query` in place. The island is not mounted again. The island stays mounted while the page is hidden.

!!! note
    Frappe builds all page islands of the bench in one build, rooted at `apps/frappe/island-toolchain/`. A page island can import only `vue`, `frappe-ui`, `@framework/ui` and what they bring. It cannot import code from your app frontend. If it must, build an app island instead.

A Frappe UI page does not run `<page_name>.js`. The first `bench build` runs `yarn install` in `apps/frappe/island-toolchain/`. If a page island build fails, `bench build` reports it and continues.

## App islands

An app island is built by your app. Use it to reuse your own components, or to mount an island in a place that is not a full page.

1. Add `@framework/ui` to your frontend as a linked package and add its Vite plugin. The steps are in `apps/frappe/ui/README.md`.
2. Write an entry that exports `mount`:

```js
// sample_app/frontend/src/islands/order_board.js
import { mountVueIsland } from "@framework/ui/island";
import OrderBoard from "./OrderBoard.vue";

export const mount = (el, context) => mountVueIsland(el, { ...context, component: OrderBoard });
```

3. Build all entries of the app in one script:

```js
// sample_app/frontend/build-islands.mjs
import { buildIslands } from "@framework/ui/vite/island";

await buildIslands({
	app: "sample_app",
	root: import.meta.dirname,
	entries: { "sample_app.order_board": "src/islands/order_board.js" },
	production: process.argv.includes("--production"),
	watch: process.argv.includes("--watch"),
});
```

4. Call the script from the `build` script in the `package.json` at the root of your app. `bench build` runs that script for each app. Output goes to `sites/assets/<app>/dist/island/`.

`mountVueIsland` also accepts `configure(app)` to register plugins, and `routes` and `history` for a vue-router inside the island.

### Mount an app island in the desk

Use `frappe.ui.mount_island(name, el, props)` in any desk code:

```js
const island = frappe.ui.mount_island("sample_app.order_board", page.body[0], {
	status: "Open",
	onTitle: (title) => page.set_title(title),
});
island.update({ status: "Completed" });
await island.ready;
island.unmount();
```

The third argument is the Vue props object. Event listeners are `on*` keys, for example `onTitle`. The handle returns at once. `ready` resolves when the island is on the page and rejects if the load fails. `frappe.ui.unmount_island(el)` removes the island from an element.

To use an app island as a whole desk page, create a `Page` with `type` = `Frappe UI` and set **Island** to the island name. Frappe then builds and scaffolds nothing for the page. The page does not need to be Standard. On save, Frappe checks that the island is in the registry.

## The host context

Inside an island component, `useHost()` from `@framework/ui/island` returns what the desk passes in. Every key is optional.

| Key | Description |
|---|---|
| `locale` | User language, from `frappe.boot.lang`. |
| `timezone` | User or system time zone. |
| `user` | The session user. |
| `base_url` | Site URL. |
| `navigate(route)` | Go to a desk route with `frappe.set_route`. Use it instead of plain links. |
| `format(value, df, doc)` | Format a value as the desk does, as text. |
| `developer_mode` | `true` on a developer-mode site. |
| `first_day_of_the_week` | From system defaults. |
| `theme` | The desk theme. Added by the mount contract. |

## Charts

Desk charts use the framework island `frappe.chart`. Draw a chart with `frappe.ui.chart(el, config)`. It returns `{update(config), destroy(), ready}`. `frappe.Chart`, `frappe.utils.make_chart` and `frappe.ui.RealtimeChart` are deprecated.

## Pitfalls

- Island names share one namespace on the bench. Prefix every name with your app name. Two builds that write the same key fail.
- A plain `<a href>` inside an island reloads the page, because the shadow root hides the click from the desk router. Use `host.navigate()`.
- `bench watch` runs the framework and page island builds in watch mode. An app island needs its own watch script, for example `node build-islands.mjs --watch`.
- A page island build that fails does not fail `bench build`. The page then shows its unbuilt state. Check the build output.
- An island named in a Page `island` field must be built and its app installed, or the Page does not save.

## Related

- [Desk Pages](desk-pages.md)
- [Client API](client-api.md)
- [Apps](../building-apps/apps.md)
- [Studio Development](../low-code-and-ai/studio-development.md)
- [JavaScript API reference](../../reference/javascript/frappe.md)
