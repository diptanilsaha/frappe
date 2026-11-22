---
title: Website
description: Serve web pages from an app with www/ files, route rules, redirects, base templates, DocTypes with a web view, page renderers and Jinja hooks.
version: v16
sources:
  - frappe/website/serve.py
  - frappe/website/path_resolver.py
  - frappe/website/router.py
  - frappe/website/page_renderers/base_template_page.py
  - frappe/website/page_renderers/template_page.py
  - frappe/website/page_renderers/document_page.py
  - frappe/website/page_renderers/list_renderer.py
  - frappe/website/page_renderers/print_page.py
  - frappe/website/page_renderers/static_page.py
  - frappe/website/page_renderers/web_form.py
  - frappe/website/website_generator.py
  - frappe/website/utils.py
  - frappe/website/doctype/website_settings/website_settings.py
  - frappe/website/doctype/web_template/web_template.py
  - frappe/www/list.py
  - frappe/www/sitemap.py
  - frappe/templates/base.html
  - frappe/templates/web.html
  - frappe/utils/jinja.py
  - frappe/utils/safe_exec.py
  - frappe/model/meta.py
  - frappe/hooks.py
---

# Website

Every request that is not `/api/*`, `/assets/*`, `/files/*` or `/desk` is a website request. Frappe resolves the path to a page and renders it with Jinja. Use the website layer for public pages, portal pages for website users, and pages that show documents of a DocType. For a form that guests fill in, use a [Web Form](web-forms.md). For desk screens, use [Desk Pages](../desk-ui/desk-pages.md).

## How a path is resolved

`frappe.website.serve.get_response()` creates a `PathResolver` for the request path. `PathResolver.resolve()` does these steps in order:

1. If the path is `desk` or starts with `desk/`, it renders the desk.
2. If the URL is in the cached 404 list, it returns the 404 page.
3. It applies redirects from the `website_redirects` hook and from Website Settings (`route_redirects`).
4. It maps the path to an endpoint. If an app sets the `website_path_resolver` hook, that function does this. Otherwise `resolve_path()` uses the home page for an empty path, removes `.html`, and applies `website_route_rules`.
5. It tries each renderer in this order and uses the first whose `can_render()` returns true.

| Order | Renderer | Renders |
|---|---|---|
| 1 | Classes from the `page_renderer` hook | Anything your app decides |
| 2 | `StaticPage` | Binary files (images, fonts, PDF) in an app `www/` folder |
| 3 | `WebFormPage` | Published Web Forms at their `route` |
| 4 | `DocumentPage` | A document of a DocType with `has_web_view`, or a Web Page with a dynamic route |
| 5 | `TemplatePage` | A `.html` or `.md` file in `www/` or `templates/pages/` |
| 6 | `PrintPage` | `/<DocType>/<name>`, the print view of a document |
| 7 | `ListPage` | `/<DocType>`, the portal list of a DocType |

If no renderer matches, Frappe shows `NotFoundPage`. `frappe.Redirect`, `frappe.PermissionError` and `frappe.PageDoesNotExistError` raised during rendering show the redirect, "not permitted" and 404 pages.

## Pages in www/

Put a file in `<app>/<app>/www/`. The file path is the route. `TemplatePage` looks for these names, in this order: `<path>`, `<path>.html`, `<path>.md`, `<path>/index.html`, `<path>/index.md`. It looks in `www/` first, then `templates/pages/`. Apps are checked from the last installed app to the first, so a later app can replace a page of an earlier app.

```text
sample_app/sample_app/www/
├── orders.html        -> /orders
├── orders.py          -> context for /orders
├── orders.js          -> added to the page (no {% block script %} in orders.html)
├── orders.css         -> added to the page (no {% block style %} in orders.html)
└── help/
    ├── index.md       -> /help
    └── shipping.md    -> /help/shipping
```

### Context with get_context

A `.py` file with the same name (hyphens become underscores) is the page controller. Frappe calls its `get_context(context)` function before it renders the template. Change `context` in place or return a dict.

```python
# sample_app/sample_app/www/orders.py
import frappe
from frappe import _

no_cache = 1


def get_context(context):
	if frappe.session.user == "Guest":
		frappe.throw(_("Log in to see your orders"), frappe.PermissionError)

	context.title = _("My Orders")
	context.orders = frappe.get_list(
		"Sample Order",
		filters={"owner_user": frappe.session.user},
		fields=["name", "order_date", "status", "total"],
		order_by="order_date desc",
	)
```

```jinja
{# sample_app/sample_app/www/orders.html #}
{% extends "templates/web.html" %}

{% block page_content %}
<h1>{{ title }}</h1>
{% for order in orders %}
	<p>{{ order.name }}: {{ frappe.format(order.total, {"fieldtype": "Currency"}) }}</p>
{% endfor %}
{% endblock %}
```

If the template does not contain `{% extends %}` and does not contain `</body>`, Frappe wraps it in the base template inside `{% block page_content %}`. So a plain `.html` or `.md` file also gets the navbar and footer.

These module-level variables in the controller set page properties:

| Name | Effect |
|---|---|
| `no_cache` | Set to `1` to never cache the rendered HTML. |
| `sitemap` | Set to `1` to add the page to `/sitemap.xml`. |
| `base_template_path` | Use another base template for this page. |
| `template` | Render another template instead of the file. |
| `condition_field` | Read by the router for page info. |

### Frontmatter

A `.md` or `.html` file can start with YAML frontmatter. The keys go into `context`.

```markdown
---
title: Shipping
add_breadcrumbs: 1
show_sidebar: 1
---

# Shipping

Orders ship in 2 days.
```

Useful context keys: `title`, `no_cache`, `sitemap`, `add_breadcrumbs`, `parents`, `show_sidebar`, `website_sidebar`, `sidebar_items`, `no_header`, `no_breadcrumbs`, `http_status_code`, `base_template_path`, `safe_render`.

### Context that every page gets

`init_context()` starts the context from `get_website_settings()` (navbar, footer, theme, `web_include_js`, `web_include_css`) and from `website_context` in `site_config.json`. After `get_context`, Frappe adds `metatags`, the base template, the title prefix, the `update_website_context` hook output, `path`, `pathname`, `canonical` and the current user (`user`, `fullname`, `user_image`).

## Templates and blocks

`templates/base.html` is the HTML document. `templates/web.html` extends it and is the usual parent for pages. These are the blocks you override most:

| Template | Blocks |
|---|---|
| `templates/base.html` | `meta_block`, `title`, `favicon`, `head`, `head_include`, `style`, `banner`, `navbar`, `content`, `footer`, `base_scripts`, `script`, `body_include` |
| `templates/web.html` | `hero`, `breadcrumbs`, `page_container`, `header`, `header_actions`, `page_content`, `page_footer`, `page_sidebar` |

To change the base template for all pages, set `base_template` in `hooks.py`. The last app wins. To change it for some routes only, use `base_template_map`. The longest matching regex wins.

```python
# sample_app/hooks.py
base_template = "sample_app/templates/sample_base.html"

base_template_map = {
	r"orders.*": "sample_app/templates/orders_base.html",
}
```

### Web Templates

A Web Template is a reusable block for Web Pages and Website Settings (navbar, footer). A standard Web Template lives in `<module>/web_template/<name>/` with `<name>.html` and `<name>.json`. Set `standard` to `1` in developer mode to export it. See `frappe/website/web_template/` for the templates that Frappe ships.

## Route rules and redirects

`website_route_rules` maps a URL pattern to an endpoint. The syntax is the Werkzeug rule syntax. Matched parts go into `frappe.form_dict`, and a match with arguments turns off the page cache.

```python
# sample_app/hooks.py
website_route_rules = [
	{"from_route": "/orders/<name>", "to_route": "order"},
	{"from_route": "/shop/<path:rest>", "to_route": "shop", "defaults": {"view": "grid"}},
]
```

With this rule, `/orders/SO-00001` renders `www/order.html`, and `frappe.form_dict.name` is `SO-00001`. Frappe itself uses this hook, for example `{"from_route": "/profile", "to_route": "me"}`.

`website_redirects` sends a redirect. `source` is a regex that must match the full path. `target` can use groups.

```python
# sample_app/hooks.py
website_redirects = [
	{"source": "/old-orders", "target": "/orders"},
	{"source": r"/catalog/(.*)", "target": r"/shop/\1", "redirect_http_status": 302},
	{"source": r"/app/(.*)", "target": r"/desk/\1", "forward_query_parameters": True},
]
```

| Key | Default | Description |
|---|---|---|
| `source` | required | Regex for the path, without the leading `/`. |
| `target` | required | URL or path. Can use `\1` for regex groups. |
| `redirect_http_status` | `301` | HTTP status of the redirect. |
| `forward_query_parameters` | `False` | Add the request query string to the target. |
| `match_with_query_string` | `False` | Match `source` against `path?query`. |

Users can also add redirects in Website Settings > Route Redirects. Resolved redirects are cached in the `website_redirects` cache key.

## DocTypes with a web view

A DocType can show each document as a web page.

1. Set `has_web_view` on the DocType.
2. Set `route` on the DocType, for example `orders`. This is also the URL of the list.
3. Optional: set `allow_guest_to_view` so that guests can see published documents.
4. Optional: set `is_published_field` to a Check field, for example `published`.
5. Make the controller extend `WebsiteGenerator`.

```python
# sample_app/sample_app/doctype/sample_item/sample_item.py
import frappe
from frappe.website.website_generator import WebsiteGenerator


class SampleItem(WebsiteGenerator):
	website = frappe._dict(
		condition_field="published",
		page_title_field="item_name",
	)

	def get_context(self, context):
		context.related = frappe.get_all(
			"Sample Item",
			filters={"category": self.category, "name": ("!=", self.name)},
			fields=["item_name", "route"],
			limit=5,
		)
```

`WebsiteGenerator.validate()` sets `route` when the document is published and `route` is empty. The route is `<DocType route>/<scrubbed title>`. `DocumentPage` renders the document with `templates/<doctype>.html` in the DocType folder. The list at `/<DocType route>` uses `templates/<doctype>_row.html` for each row.

```text
sample_app/sample_app/doctype/sample_item/templates/
├── sample_item.html       # one document
└── sample_item_row.html   # one row in the list
```

`DocumentPage` shows a document when one of these is true: the DocType has `allow_guest_to_view`, the user has read permission, or `frappe.has_website_permission(doc)` returns true. Use the `has_website_permission` hook or a `has_website_permission` controller method for portal access rules.

```python
# sample_app/hooks.py
has_website_permission = {
	"Sample Order": "sample_app.portal.order_has_website_permission",
}
```

```python
# sample_app/portal.py
def order_has_website_permission(doc, ptype, user, verbose=False):
	return doc.owner_user == user
```

## Portal list pages

`ListPage` renders `/<DocType>` with `www/portal.html` (`frappe/www/list.py`). It renders for DocTypes with `has_web_view`, and for standard DocTypes whose module has a `get_list_context(context)` function. Return a dict to change the list.

```python
# sample_app/sample_app/doctype/sample_order/sample_order.py
def get_list_context(context=None):
	return {
		"title": "My Orders",
		"show_sidebar": True,
		"row_template": "sample_app/templates/includes/order_row.html",
	}
```

Add portal menu items for website users with the `standard_portal_menu_items` hook. Portal Settings syncs them into its menu.

```python
# sample_app/hooks.py
standard_portal_menu_items = [
	{"title": "Orders", "route": "/orders", "reference_doctype": "Sample Order", "role": "Sample Role"},
]
```

## Home page

`get_home_page()` picks the page for `/`, in this order:

1. The `home_page` of the user's roles (logged-in users).
2. Portal Settings `default_portal_home` (logged-in users).
3. The `get_website_user_home_page` hook, then `website_user_home_page`, then `role_home_page`, then `home_page`.
4. Website Settings `home_page`.
5. `login` for guests, `me` for others. System users go to `desk`. Website users with a portal role go to `portal`.
6. If the user has a default workspace, Frappe uses the route of that workspace.

```python
# sample_app/hooks.py
role_home_page = {"Sample Role": "orders"}
```

## Custom page renderers

Add a class to the `page_renderer` hook to render paths your own way. It runs before all built-in renderers. The class gets `(path, http_status_code)` and must have `can_render()` and `render()`. Extend `BaseRenderer` or `BaseTemplatePage`.

```python
# sample_app/hooks.py
page_renderer = ["sample_app.renderers.FeedRenderer"]
```

```python
# sample_app/renderers.py
import frappe
from frappe.website.page_renderers.base_renderer import BaseRenderer


class FeedRenderer(BaseRenderer):
	def can_render(self):
		return self.path == "orders.json"

	def render(self):
		data = frappe.as_json(frappe.get_all("Sample Order", fields=["name", "status"], limit=20))
		return self.build_response(data, headers={"Content-Type": "application/json"})
```

Frappe uses this hook for Studio apps (`frappe.studio.doctype.studio_app.studio_app.StudioAppRenderer`).

## Jinja

`frappe.render_template(template, context)` renders a template string or a template path. Website pages, print formats, email templates and notifications all use it.

```python
frappe.render_template("Hello {{ name }}", {"name": "Customer A"})
# 'Hello Customer A'

frappe.render_template("{{ '2026-01-15' | global_date_format }}", {})
# 'January 15, 2026'

frappe.render_template("templates/includes/footer/footer_powered.html", {})
```

Frappe guesses that the string is a path when it looks like one. Pass `is_path=True` to be sure. With `safe_render=True` (the default), a template string that contains `.__` raises "Illegal template".

Templates run in a sandboxed Jinja environment. The `disable_render_safe_exec` key in `common_site_config.json` selects the globals (`get_jenv()` in `frappe/utils/jinja.py`):

| `disable_render_safe_exec` | Globals |
|---|---|
| not set or truthy (default) | `get_safe_globals()`, the same API that Server Scripts get. This includes `frappe.db.sql`. |
| `0` | `render_safe_globals()`, a smaller set for rendering only. |

Both sets include `frappe.format`, `frappe.get_doc`, `frappe.get_list`, `frappe.get_all`, `frappe.db.get_value`, `frappe.db.get_single_value`, `frappe.db.exists`, `frappe.db.count`, `frappe.session.user`, `frappe.form_dict`, `frappe.utils` (date and number helpers) and `_`. `frappe.get_list` and `frappe.db.get_value` apply the user's permissions. Both sets are in `frappe/utils/safe_exec.py`.

!!! note "Version"
    In later versions, templates always use `render_safe_globals()`.

### Add methods and filters

Use the `jinja` hook. Each value is a module path or a function path. For a module path, Frappe adds every function defined in that module (not imported into it).

```python
# sample_app/hooks.py
jinja = {
	"methods": [
		"sample_app.jinja.methods",
		"sample_app.utils.get_order_total",
	],
	"filters": [
		"sample_app.jinja.filters",
	],
}
```

```python
# sample_app/jinja/filters.py
def as_order_code(value):
	return f"ORD/{value}"
```

```jinja
{{ doc.name | as_order_code }}
{{ get_order_total(doc.name) }}
```

### Change context for all pages

`update_website_context` runs for every page after `get_context`. Return a dict to merge it into the context.

```python
# sample_app/hooks.py
update_website_context = ["sample_app.website.update_context"]
website_context = {"favicon": "/assets/sample_app/images/favicon.png"}
```

```python
# sample_app/website.py
def update_context(context):
	return {"support_email": "support@example.com"}
```

## Caching

`TemplatePage` and `DocumentPage` cache the HTML in Redis per path and per language for 30 minutes (`cache_html` in `frappe/website/utils.py`). The response gets `Cache-Control: private,max-age=300,stale-while-revalidate=10800` and `X-From-Cache`. Frappe does not cache when one of these is true:

- `developer_mode` or `disable_website_cache` is set in site config.
- The request has a query string.
- The page sets `no_cache`, or a route rule matched with arguments.
- The request sends `Cache-Control: no-cache`.

`WebsiteGenerator.clear_cache()` clears the cache for the document route after commit. Run `frappe.website.utils.clear_cache(path)` to clear one path, or `bench --site <site> clear-website-cache` for all.

## Sitemap

`/sitemap.xml` (`frappe/www/sitemap.py`) lists:

- `www/` pages whose controller or frontmatter sets `sitemap = 1`.
- Published documents of DocTypes with `has_web_view`, `allow_guest_to_view` and a condition field. Routes blocked in Website Settings `robots_txt` are left out.

## Pitfalls

- A `.py` file in `www/` is not a page. Only `.html` and `.md` files are pages. The `.py` file only gives context.
- Pages are cached for guests and users alike. Set `no_cache = 1` on pages that show user data.
- `get_context` runs as the current user. Check permissions in it, for example with `frappe.get_list` or `frappe.has_permission`.
- `website_route_rules` and the DocType route map are cached. Run `bench --site <site> clear-cache` after you change them on a running site.
- `website_path_resolver` replaces route rules and home page handling. The last function in the hook list wins.

## Related

- [Web Forms](web-forms.md)
- [Hooks](../building-apps/hooks.md)
- [Print Formats](../output/print-formats.md)
- [Caching](../runtime/caching.md)
- [Website (user guide)](../../user-guide/website-and-integrations/website.md)
- [Hooks reference](../../reference/configuration/hooks.md)
