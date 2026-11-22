---
title: Hooks Reference
description: Every hooks.py key that Frappe Framework v15 reads, with its shape, the file that reads it, and what it does.
version: v15
sources:
  - frappe/website/web_template/primary_navbar/primary_navbar.html
  - frappe/__init__.py
  - frappe/hooks.py
  - frappe/app.py
  - frappe/apps.py
  - frappe/auth.py
  - frappe/boot.py
  - frappe/sessions.py
  - frappe/installer.py
  - frappe/migrate.py
  - frappe/permissions.py
  - frappe/test_runner.py
  - frappe/parallel_test_runner.py
  - frappe/commands/utils.py
  - frappe/model/document.py
  - frappe/model/db_query.py
  - frappe/model/meta.py
  - frappe/model/sync.py
  - frappe/model/delete_doc.py
  - frappe/model/naming.py
  - frappe/modules/patch_handler.py
  - frappe/desk/form/meta.py
  - frappe/desk/form/load.py
  - frappe/desk/form/linked_with.py
  - frappe/desk/search.py
  - frappe/desk/notifications.py
  - frappe/desk/page/setup_wizard/setup_wizard.py
  - frappe/desk/page/leaderboard/leaderboard.py
  - frappe/desk/doctype/changelog_feed/changelog_feed.py
  - frappe/www/app.py
  - frappe/www/printview.py
  - frappe/www/list.py
  - frappe/www/login.py
  - frappe/website/path_resolver.py
  - frappe/website/router.py
  - frappe/website/utils.py
  - frappe/website/page_renderers/base_template_page.py
  - frappe/website/doctype/website_settings/website_settings.py
  - frappe/website/doctype/web_form/web_form.py
  - frappe/utils/__init__.py
  - frappe/utils/background_jobs.py
  - frappe/core/doctype/scheduled_job_type/scheduled_job_type.py
  - frappe/core/doctype/log_settings/log_settings.py
  - frappe/core/doctype/page/page.py
  - frappe/core/doctype/user_invitation/user_invitation.py
  - frappe/email/email_body.py
  - frappe/email/doctype/email_queue/email_queue.py
  - frappe/utils/file_manager.py
  - frappe/core/doctype/file/file.py
  - frappe/utils/print_utils.py
  - frappe/utils/pdf.py
  - frappe/utils/jinja.py
  - frappe/utils/fixtures.py
  - frappe/utils/change_log.py
  - frappe/gettext/translate.py
  - frappe/search/sqlite_search.py
---

# Hooks Reference

This page lists every key in an app's `hooks.py` that the framework reads in v15.
Use it to look up the shape of a hook value and the code that reads it.
For how hooks work and how values from many apps merge, see [Hooks](../../developer-guide/building-apps/hooks.md).

## How to read this page

- `frappe.get_hooks("key")` collects the key from all installed apps and returns a list. A single string value becomes a one-item list.
- A dict value is merged key by key. Each inner value also becomes a list. For example, `doc_events["*"]["on_update"]` is a list of dotted paths from all apps.
- "Last wins" means the framework uses only the last item (`[-1]`). Apps load in install order, so the app installed last wins.
- "First wins" means the framework uses only the first item (`[0]`).
- "All run" means the framework calls every dotted path in the list, in app order.
- "Per app" means the framework reads the key with `app_name=<app>` and uses only that app's value.
- Paths in the "Read in" column are relative to `apps/frappe`.

The **Audit System Hooks** report (`frappe/custom/report/audit_system_hooks`) shows the merged value of each hook on a site.
In code, check a value from the console:

```python
frappe.get_hooks("calendars")   # ['Event']
```

## App metadata

| Key | Shape | Read in | Description |
|---|---|---|---|
| `app_name` | str | `frappe/utils/boilerplate.py` | Python package name of the app. Written by `bench new-app`. |
| `app_title` | str | `frappe/utils/change_log.py`, `frappe/email/email_body.py`, `frappe/gettext/translate.py`, `frappe/desk/doctype/changelog_feed/changelog_feed.py`, `frappe/core/doctype/user_invitation/user_invitation.py` | Display name. Used in version info, in `.pot` headers, in the changelog feed, in invitation emails and as the email header title (last wins). |
| `app_publisher` | str | `frappe/modules/utils.py`, `frappe/gettext/translate.py` | Copyright holder in boilerplate files and `.pot` headers. Per app. |
| `app_description` | str | `frappe/utils/change_log.py` | Description shown in version info. Per app. |
| `app_email` | str | `frappe/gettext/translate.py` | Bug report address in the `.pot` catalog header. Per app. |
| `app_license` | str | `frappe/utils/boilerplate.py` | License name. Only the boilerplate generator uses it. |
| `app_logo_url` | str | `frappe/core/doctype/navbar_settings/navbar_settings.py` | Navbar logo when Navbar Settings has none. When exactly two apps set one, the second logo is used. |
| `<branch>_version` | str | `frappe/utils/change_log.py`, `frappe/commands/utils.py` | Version label for a branch, for example `develop_version`. Shown when the app is not on `master`, and by `bench version`. |
| `add_to_apps_screen` | list of dict | `frappe/apps.py` | Adds the app to the apps screen (`/apps`). Keys: `name`, `logo`, `title`, `route`, `has_permission` (dotted path). The tile always shows the app that sets the hook. `name` is used only to find a workspace of the app's modules when the user cannot open `route`. |
| `required_apps` | list of str | `frappe/installer.py`, `frappe/boot.py`, `frappe/core/doctype/installed_applications/installed_applications.py` | Apps to install before this app. `bench install-app` installs them first. `uninstall-app` refuses to remove an app that another app requires. |

```python
# sample_app/hooks.py
app_name = "sample_app"
app_title = "Sample App"
app_publisher = "Example Corp"
required_apps = ["frappe"]
add_to_apps_screen = [
	{
		"name": "sample_app",
		"logo": "/assets/sample_app/logo.svg",
		"title": "Sample App",
		"route": "/app/sample-item",
		"has_permission": "sample_app.api.check_app_permission",
	}
]
```

## App lifecycle (install, migrate, build, tests)

All keys in this table are lists of dotted paths. "Per app" keys run only the hooks of the app that the event is about.

| Key | Read in | Called with | Description |
|---|---|---|---|
| `before_install` | `frappe/installer.py` | nothing | Per app. Runs before the app is installed. If a function returns `False`, the install stops. |
| `after_install` | `frappe/installer.py` | nothing | Per app. Runs after the app's DocTypes sync. |
| `after_sync` | `frappe/installer.py` | nothing | Per app. Runs at the end of `install_app`, after fixtures sync. |
| `before_app_install` | `frappe/installer.py` | `app_name` | All apps. Runs before any app is installed. |
| `after_app_install` | `frappe/installer.py` | `app_name` | All apps. Runs after any app is installed. |
| `before_uninstall` | `frappe/installer.py` | nothing | Per app. Runs before the app is removed. |
| `after_uninstall` | `frappe/installer.py` | nothing | Per app. Runs after the app is removed. |
| `before_app_uninstall` | `frappe/installer.py` | `app_name` | All apps. Runs before any app is removed. |
| `after_app_uninstall` | `frappe/installer.py` | `app_name` | All apps. Runs after any app is removed. |
| `before_migrate` | `frappe/migrate.py` | nothing | Per app, for every installed app. Runs at the start of `bench migrate`. |
| `after_migrate` | `frappe/migrate.py` | nothing | Per app, for every installed app. Runs at the end of `bench migrate`. |
| `after_build` | `frappe/commands/utils.py` | nothing | Per app. Runs after `bench build` for the apps that were built. |
| `after_app_build` | `frappe/commands/utils.py` | list of built apps | Per app, for every app on the bench. Runs after a build. |
| `before_tests` | `frappe/test_runner.py`, `frappe/parallel_test_runner.py` | nothing | Per app. Runs before the tests of the app. `bench run-tests --skip-before-tests` skips it. |
| `never_skip_patches` | `frappe/modules/patch_handler.py` | not called | Patch paths that `bench migrate --skip-failing` cannot skip. |
| `fixtures` | `frappe/utils/fixtures.py`, `frappe/translate.py` | not called | Records to export with `bench export-fixtures` and import on migrate. Items are DocType names or dicts with `doctype`, `filters`, `or_filters`. |

```python
# sample_app/hooks.py
after_install = "sample_app.setup.install.after_install"
after_migrate = ["sample_app.setup.install.sync_defaults"]
fixtures = [{"doctype": "Custom Field", "filters": [["module", "=", "Sample App"]]}]
```

## Request and session

| Key | Shape | Read in | Description |
|---|---|---|---|
| `before_request` | list of dotted paths | `frappe/app.py` | All run before each HTTP request is handled. No arguments. |
| `after_request` | list of dotted paths | `frappe/app.py` | All run after each request. Called with `response=` and `request=`. |
| `auth_hooks` | list of dotted paths | `frappe/auth.py` | All run while Frappe validates the request auth. Use them to set the user from a custom header. No arguments. |
| `on_login` | list of dotted paths | `frappe/auth.py` | Runs after a successful login, before the session is made. Called with `login_manager=`. |
| `on_session_creation` | list of dotted paths | `frappe/auth.py` | Runs after a new session is made. Called with `login_manager=`. |
| `on_logout` | list of dotted paths | `frappe/auth.py` | Runs on logout, before the session is deleted. Called with `login_manager=`. |
| `override_whitelisted_methods` | dict: dotted path -> dotted path | `frappe/__init__.py` | Replaces a whitelisted method for `/api/method/<path>`. Last wins. |

```python
# sample_app/hooks.py
before_request = ["sample_app.utils.request.log_path"]
override_whitelisted_methods = {
	"frappe.desk.doctype.event.event.get_events": "sample_app.api.get_events",
}
```

## Documents and models

| Key | Shape | Read in | Description |
|---|---|---|---|
| `doc_events` | dict: DocType (or `"*"`, or tuple of DocTypes) -> event -> dotted paths | `frappe/__init__.py`, `frappe/model/document.py` | Runs functions on document events (`validate`, `on_update`, `on_submit`, ...). Called with `doc, method`. Handlers for the DocType run before `"*"` handlers. See [Controllers](../../developer-guide/data-model/controllers.md). |
| `override_doctype_class` | dict: DocType -> dotted class path | `frappe/model/document.py`, `frappe/model/sync.py` | Replaces the controller class of a DocType. Last wins. |
| `override_doctype_dashboards` | dict: DocType -> dotted paths | `frappe/model/meta.py` | Changes the connections dashboard of a standard DocType. Called with `data=`. Must return the new data. |
| `naming_series_variables` | dict: variable -> dotted path | `frappe/model/naming.py` | Custom parts in naming series, for example `.{branch}.` First wins per variable. Called with `doc, part`. |
| `ignore_links_on_delete` | list of DocTypes | `frappe/model/delete_doc.py` | Links from these DocTypes do not block delete or cancel. |
| `auto_cancel_exempted_doctypes` | list of DocTypes | `frappe/desk/form/linked_with.py` | DocTypes that "cancel all linked documents" does not cancel. |
| `export_python_type_annotations` | bool | `frappe/core/doctype/doctype/doctype.py` | Per app. `True` writes type hints into the controller when a DocType is saved in developer mode. |
| `domains` | dict: domain -> dotted module | `frappe/__init__.py`, `frappe/core/doctype/domain/domain.py`, `frappe/core/doctype/domain_settings/domain_settings.py` | Domain setup data. The module must have a `data` attribute. |

```python
# sample_app/hooks.py
doc_events = {
	"Sample Order": {
		"validate": "sample_app.events.sample_order.validate",
		"on_submit": ["sample_app.events.sample_order.notify"],
	},
	"*": {"on_trash": "sample_app.events.audit.log_delete"},
}
override_doctype_class = {"ToDo": "sample_app.overrides.todo.SampleToDo"}
```

## Permissions

| Key | Shape | Read in | Description |
|---|---|---|---|
| `has_permission` | dict: DocType or `"*"` -> dotted paths | `frappe/permissions.py` | Called with `doc, ptype, user, debug`. Frappe calls the methods in reverse order. The first result that is not `None` decides. `None` means "no opinion". A falsy result denies. A true result does not grant more than role permissions. Not called for Administrator. |
| `permission_query_conditions` | dict: DocType or `"*"` -> dotted paths | `frappe/model/db_query.py` | Called with `user, doctype=`. Returns an SQL condition string that `get_list` adds to the query. |
| `has_website_permission` | dict: DocType -> dotted paths | `frappe/__init__.py` | Permission check for portal pages of a DocType, when the controller has no `has_website_permission` method. |
| `user_invitation` | dict | `frappe/core/doctype/user_invitation/user_invitation.py`, `frappe/core/api/user_invitation.py` | Per app. Keys: `allowed_roles` (role -> roles it can invite), `extra_invite_params` (list), `after_accept` (dotted paths, called with `invitation=`, `user=`, `user_inserted=`). |

```python
# sample_app/hooks.py
has_permission = {"Sample Order": "sample_app.permissions.sample_order_has_permission"}
permission_query_conditions = {"Sample Order": "sample_app.permissions.sample_order_query"}
```

```python
# sample_app/permissions.py
def sample_order_query(user, doctype=None):
	return f"`tabSample Order`.owner_user = {frappe.db.escape(user)}"
```

## Desk, boot and assets

| Key | Shape | Read in | Description |
|---|---|---|---|
| `app_include_js` | list of str | `frappe/www/app.py` | JS bundles or paths loaded in desk (`/app`). Site config `app_include_js` adds more. |
| `app_include_css` | list of str | `frappe/www/app.py` | CSS loaded in desk. Site config `app_include_css` adds more. |
| `app_include_icons` | list of str | `frappe/www/app.py` | SVG sprite files loaded in desk. |
| `doctype_js` | dict: DocType -> path | `frappe/desk/form/meta.py` | Extra form script for a DocType of another app. Path is relative to the app package. |
| `doctype_list_js` | dict: DocType -> path | `frappe/desk/form/meta.py` | Extra list view script. |
| `doctype_tree_js` | dict: DocType -> path | `frappe/desk/form/meta.py` | Extra tree view script. |
| `doctype_calendar_js` | dict: DocType -> path | `frappe/desk/form/meta.py` | Extra calendar view script. |
| `page_js` | dict: page name -> path | `frappe/core/doctype/page/page.py` | Extra script for a desk Page. |
| `boot_session` | list of dotted paths | `frappe/boot.py` | Called with `bootinfo` while desk boot data is built. |
| `extend_bootinfo` | list of dotted paths | `frappe/sessions.py` | Called with `bootinfo=` after the boot data is built. |
| `calendars` | list of DocTypes | `frappe/boot.py` | DocTypes shown as calendars. |
| `treeviews` | list of DocTypes | `frappe/boot.py` | DocTypes that open in tree view. |
| `filters_config` | list of dotted paths | `frappe/boot.py` | Each returns a dict of custom filter operators. |
| `standard_queries` | dict: DocType -> dotted path | `frappe/desk/search.py` | Default Link field search query for a DocType. Last wins. |
| `awesomebar_search` | list of dotted paths | `frappe/desk/search.py`, `frappe/boot.py` | Extra awesomebar results. Called with the search text. Returns a list of items. |
| `additional_timeline_content` | dict: DocType or `"*"` -> dotted paths | `frappe/desk/form/load.py` | Extra timeline items. Called with `doctype, docname`. Returns a list. |
| `notification_config` | list of dotted paths | `frappe/desk/notifications.py` | Each returns a dict with `for_doctype`, `for_module`, `for_other`, `targets` for open counts. |
| `notification_skip_email_types` | list of str | `frappe/desk/doctype/notification_log/notification_log.py` | Notification Log types that do not send their own email. |
| `notification_self_notify_types` | list of str | `frappe/desk/doctype/notification_log/notification_log.py` | Notification Log types that reach the user who caused them. |
| `standard_navbar_items` | list of dict | `frappe/utils/install.py` | Items for the user (settings) dropdown, added to Navbar Settings on install. |
| `standard_help_items` | list of dict | `frappe/utils/install.py` | Items for the help dropdown. Keys: `item_label`, `item_type`, `route` or `action`, `is_standard`. |
| `sounds` | list of dict (`name`, `src`, `volume`) | `frappe/www/app.py` | Sounds that desk can play. |
| `global_search_doctypes` | dict: domain -> list of dict | `frappe/desk/doctype/global_search_settings/global_search_settings.py` | Default DocTypes for global search. Per app. |
| `leaderboards` | list of dotted paths | `frappe/desk/page/leaderboard/leaderboard.py` | Each returns a dict of leaderboard configs for the Leaderboard page. |
| `get_changelog_feed` | list of dotted paths | `frappe/desk/doctype/changelog_feed/changelog_feed.py` | Called with `since=`. Returns items with `title`, `app_name`, `link`, `creation`. Frappe stores up to 20 per hook as Changelog Feed records. |
| `clear_cache` | list of dotted paths | `frappe/__init__.py` | Runs on `frappe.clear_cache()` for the whole site. |
| `persistent_cache_keys` | list of str (glob) | `frappe/__init__.py` | Redis keys that `clear_cache` keeps. |
| `setup_wizard_requires` | list of str | `frappe/boot.py` | Assets the setup wizard loads first. |
| `setup_wizard_stages` | dotted path | `frappe/desk/page/setup_wizard/setup_wizard.py` | Per app. Returns the stages and tasks of the setup wizard. |
| `setup_wizard_complete` | list of dotted paths | `frappe/desk/page/setup_wizard/setup_wizard.py` | Run when the setup wizard completes. Called with the wizard args. |
| `setup_wizard_success` | list of dotted paths | `frappe/desk/page/setup_wizard/setup_wizard.py` | Run after setup succeeds. Called with the args. |
| `setup_wizard_exception` | list of dotted paths | `frappe/desk/page/setup_wizard/setup_wizard.py` | Run when setup fails. Called with `traceback, args`. |

```python
# sample_app/hooks.py
app_include_js = "sample_app.bundle.js"
doctype_js = {"ToDo": "public/js/todo.js"}
standard_queries = {"Sample Item": "sample_app.queries.sample_item_query"}
```

## Website and portal

| Key | Shape | Read in | Description |
|---|---|---|---|
| `home_page` | str | `frappe/website/utils.py` | Last wins. Website home page route. |
| `role_home_page` | dict: role -> route | `frappe/website/utils.py` | Home page for users with a role. |
| `website_user_home_page` | str | `frappe/website/utils.py` | Last wins. Home page for logged-in website users. |
| `get_website_user_home_page` | dotted path | `frappe/website/utils.py` | Last wins. Called with the user. Returns the home page route. |
| `website_route_rules` | list of dict (`from_route`, `to_route`, `defaults`) | `frappe/website/path_resolver.py` | Maps URL patterns to pages. |
| `website_redirects` | list of dict (`source`, `target`, `match_with_query_string`, `redirect_http_status`) | `frappe/website/path_resolver.py` | Regex redirects. Default status 301. Website Route Redirect records add more. |
| `website_path_resolver` | list of dotted paths | `frappe/website/path_resolver.py` | Replaces the path to endpoint lookup. Called with the path. |
| `page_renderer` | list of dotted class paths | `frappe/website/path_resolver.py` | Custom renderers. The class needs `can_render` and `render`. |
| `website_generators` | list of DocTypes | `frappe/website/router.py` | DocTypes that make web pages, in addition to DocTypes with `has_web_view`. |
| `base_template` | str | `frappe/website/page_renderers/base_template_page.py` | Last wins. Base template for web pages. Default `templates/base.html`. |
| `base_template_map` | dict: regex -> template | `frappe/website/router.py` | Base template per route pattern. The longest pattern is tried first. |
| `update_website_context` | list of dotted paths | `frappe/website/page_renderers/base_template_page.py` | Called with `context`. A returned dict updates the context. |
| `website_context` | dict | `frappe/website/doctype/website_settings/website_settings.py` | Static values added to the website context. |
| `web_include_js`, `web_include_css` | list of str | `frappe/website/doctype/website_settings/website_settings.py` | Assets on every website page. |
| `brand_html` | str (HTML) | `frappe/website/web_template/primary_navbar/primary_navbar.html` | First value wins (app install order). Shown as the website navbar brand only when Website Settings **Brand HTML** and **Banner Image** are empty. Frappe does not set it. |
| `webform_include_js`, `webform_include_css` | dict: DocType -> path | `frappe/website/doctype/web_form/web_form.py` | Extra script or style for Web Forms of a DocType. Path is relative to the app package. |
| `webform_list_context` | dotted path | `frappe/www/list.py` | First wins. Called with the module. Returns list context for a DocType. |
| `portal_menu_items` | list of dict | `frappe/website/utils.py` | Portal sidebar items, added to the items in Portal Settings. |
| `standard_portal_menu_items` | list of dict (`title`, `route`, `reference_doctype`, `role`) | `frappe/website/doctype/portal_settings/portal_settings.py` | Standard items synced into Portal Settings. A missing `role` is made. |
| `look_for_sidebar_json` | bool | `frappe/website/utils.py` | `True` loads `_sidebar.json` files for www pages. |
| `website_clear_cache` | list of dotted paths | `frappe/website/utils.py` | Called with `path` when the website cache is cleared. |
| `get_web_pages_with_dynamic_routes` | list of dotted paths | `frappe/website/doctype/web_page/web_page.py` | Each returns extra Web Pages with dynamic routes. |
| `signup_form_template` | str | `frappe/www/login.py` | Last wins. Template path (or dotted path) for the signup form. |
| `template_apps` | list of str | `frappe/utils/jinja.py` | App order for the Jinja template loader. Default: installed apps, reversed. |
| `jinja` | dict with `methods` and `filters` (dotted paths) | `frappe/utils/jinja.py` | Adds functions and filters to Jinja. A path to a module adds all functions defined in it. |
| `user_data_fields` | list of dict (`doctype`, `filter_by`, `redact_fields`, `partial`, `rename`, `strict`) | `frappe/website/doctype/personal_data_deletion_request/personal_data_deletion_request.py`, `frappe/website/doctype/personal_data_download_request/personal_data_download_request.py` | Where user data lives, for personal data download and delete. |

```python
# sample_app/hooks.py
website_route_rules = [{"from_route": "/orders/<name>", "to_route": "orders/order"}]
website_redirects = [{"source": "/old-items", "target": "/items"}]
jinja = {"methods": ["sample_app.jinja.get_rate"], "filters": ["sample_app.jinja.money"]}
```

## Background jobs and scheduler

| Key | Shape | Read in | Description |
|---|---|---|---|
| `scheduler_events` | dict: event -> dotted paths, `cron` -> dict: cron string -> dotted paths | `frappe/core/doctype/scheduled_job_type/scheduled_job_type.py` | Makes Scheduled Job Type records on migrate. Events: `all`, `hourly`, `hourly_long`, `hourly_maintenance`, `daily`, `daily_long`, `daily_maintenance`, `weekly`, `weekly_long`, `monthly`, `monthly_long`, `yearly`, `annual`, `cron`. `_long` and `_maintenance` events run on the `long` queue. |
| `before_job` | list of dotted paths | `frappe/utils/background_jobs.py` | Runs before each background job. Called with `method=`, `kwargs=`, `transaction_type="job"`. |
| `after_job` | list of dotted paths | `frappe/utils/background_jobs.py` | Runs after each job. Called with `method=`, `kwargs=`, `result=`. |
| `default_log_clearing_doctypes` | dict: DocType -> days | `frappe/core/doctype/log_settings/log_settings.py` | Default retention days added to Log Settings. Last wins per DocType. |
| `get_site_info` | list of dotted paths | `frappe/utils/__init__.py` | Called with `site_info`. A returned dict adds data to site usage info. |

```python
# sample_app/hooks.py
scheduler_events = {
	"daily": ["sample_app.tasks.close_old_orders"],
	"cron": {"0/30 * * * *": ["sample_app.tasks.sync_rates"]},
}
```

See [Scheduler](../../developer-guide/runtime/scheduler.md) and [Background Jobs](../../developer-guide/runtime/background-jobs.md).

## Email and notifications

| Key | Shape | Read in | Description |
|---|---|---|---|
| `email_css` | list of str | `frappe/email/email_body.py` | CSS inlined into outgoing email. |
| `default_mail_footer` | str | `frappe/email/email_body.py` | Footer HTML passed to the `email_footer` template. |
| `make_email_body_message` | list of dotted paths | `frappe/email/email_body.py` | Called with the `EMail` object before the message is built. |
| `override_email_send` | dotted path | `frappe/email/doctype/email_queue/email_queue.py` | First wins. Replaces SMTP send. Called with `email_queue, sender, recipient, message`. |
| `get_sender_details` | dotted path | `frappe/utils/__init__.py` | First wins. Returns `(sender_name, email)` for the default sender. |
| `welcome_email` | dotted path | `frappe/core/doctype/user/user.py` | Last wins. Returns the subject of the welcome email. |

## Files

| Key | Shape | Read in | Description |
|---|---|---|---|
| `before_write_file` | list of dotted paths | `frappe/utils/file_manager.py`, `frappe/core/doctype/file/file.py` | Called with `file_size=` before a file is written. |
| `write_file` | dotted path | `frappe/utils/file_manager.py`, `frappe/core/doctype/file/file.py` | First wins. Custom file writer, for example for remote storage. The File method passes the File document. |
| `delete_file_data_content` | dotted path | `frappe/utils/file_manager.py`, `frappe/core/doctype/file/file.py` | First wins. Custom file delete. Called with the File document and `only_thumbnail=`. |
| `write_file_keys` | list of str | `frappe/utils/file_manager.py` | File fields copied when a duplicate content hash is found. |

See [Files](../../developer-guide/data-model/files.md).

## Printing and PDF

| Key | Shape | Read in | Description |
|---|---|---|---|
| `pdf_generator` | list of dotted paths | `frappe/utils/print_utils.py` | Tried only when the PDF generator is not `wkhtmltopdf` (the default). Called with `print_format, html, options, output, pdf_generator`. The first non-empty result is the PDF. |
| `pdf_header_html`, `pdf_footer_html` | dotted path | `frappe/utils/pdf.py` | Last wins. Renders the wkhtmltopdf header or footer. Called with `soup, head, content, styles, html_id, css`. |
| `pdf_body_html` | dotted path | `frappe/www/printview.py` | Last wins. Renders the print body. Called with `jenv, template, print_format, args`. |
| `get_print_format_template` | dotted path | `frappe/www/printview.py` | Last wins. Called with `jenv, print_format`. Returns a Jinja template or `None`. |
| `on_print_pdf` | list of dotted paths | `frappe/utils/print_utils.py` | Called with `doctype, name, print_format` before wkhtmltopdf makes the PDF. |

## Search and translation

| Key | Shape | Read in | Description |
|---|---|---|---|
| `sqlite_search` | list of dotted class paths | `frappe/search/sqlite_search.py` | Full text search classes. Each must extend `SQLiteSearch`. |
| `ignore_translatable_strings_from` | list of app names | `frappe/gettext/translate.py` | Per app. Strings from these apps are left out of this app's `.pot` file. |

## Keys that frappe declares but does not read

`frappe/hooks.py` sets `email_append_to`, `otp_methods` and `require_type_annotated_api_methods`. No code in v15 reads them through `get_hooks`.
`jenv` is deprecated. A v13 patch only prints a warning when an app sets it. Use `jinja`.

!!! note "Version"
    These hooks exist only in later versions: `before_login`, `on_social_login`, `extend_doctype_class`, `workflow_methods`, `after_file_upload`, `storage_drivers`, `before_enable`, `after_enable`, `before_disable`, `after_disable`, `require_type_annotated_api_methods` (read), `use_json_request_body`, `send_sms`, `send_token_via_sms`, `doctype_kanban_js`. Setting them in a v15 app does nothing.

## Pitfalls

- A key with a typo does nothing and gives no error. Check the value with `frappe.get_hooks("<key>")`.
- Merged hooks are cached. Run `bench --site <site> clear-cache` after you change `hooks.py`.
- For "last wins" keys, the app install order decides which app wins.
- A `has_permission` hook that returns `True` stops the hooks of the apps installed before it. Return `None` when your app has no opinion.

## Related

- [Hooks](../../developer-guide/building-apps/hooks.md)
- [Extending Apps](../../developer-guide/building-apps/extending-apps.md)
- [Controllers](../../developer-guide/data-model/controllers.md)
- [Website](../../developer-guide/website/website.md)
- [Configuration](../../developer-guide/operations/configuration.md)
