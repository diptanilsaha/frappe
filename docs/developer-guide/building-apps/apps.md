---
title: Apps
description: Create a Frappe app, understand its files and folders, install it on a site, and build its assets.
version: v16
sources:
  - frappe/utils/boilerplate.py
  - frappe/commands/utils.py
  - frappe/commands/site.py
  - frappe/installer.py
  - frappe/deprecation_dumpster.py
  - frappe/__init__.py
  - frappe/modules/utils.py
  - frappe/core/doctype/doctype/doctype.py
  - esbuild/esbuild.js
---

# Apps

An app is a Python package that adds DocTypes, pages, hooks, APIs and assets to a site. All your code goes in an app. Do not edit `frappe` or other apps that you do not own. Use an app when you need code-level changes. Use [Customize Form](../../user-guide/customization/customize-form.md) or [Server and Client Scripts](../low-code-and-ai/server-and-client-scripts.md) for small changes that a site admin can make.

## Concepts

- A bench holds many apps in `apps/`. The file `sites/apps.txt` lists the apps on the bench.
- A site uses only the apps that are installed on it. The list is stored in the `installed_apps` global default.
- An app has one or more modules. Each module is a folder in the app package. DocTypes, reports and pages belong to a module.
- `frappe` is always the first app. Other apps follow in the order of installation.

See [Architecture](../../getting-started/architecture.md) for the bench, site and app model.

## Create an app

1. Go to the bench root.
2. Run `bench new-app`.

```bash
cd ~/frappe-bench
bench new-app sample_app
```

`bench new-app` is a command of the `bench` CLI. It calls the Frappe command `bench make-app apps sample_app`. Then it installs the package into the bench virtual environment, adds the app to `sites/apps.txt` and builds assets.

`bench make-app DESTINATION APP_NAME` writes only the files. Use `--no-git` to skip `git init`. The app name is converted to snake case with `frappe.scrub`. The command stops if a site has the same name as the app.

The command asks these questions:

| Prompt | Default | Notes |
|---|---|---|
| App Title | App name in title case | Must start with a letter. Letters, numbers, spaces and underscores only. |
| App Description | none | |
| App Publisher | none | |
| App Email | none | Must be a valid email address. |
| App License | `mit` | Choices come from the GitHub licenses API. |
| Create GitHub Workflow action for unittests | `False` | Adds `.github/workflows/ci.yml` and `linter.yml`. |
| Branch Name | Current branch of `frappe` | Used for `git init` and the CI workflow. |

## Files and folders

`bench make-app` makes this tree (`frappe/utils/boilerplate.py`):

```text
apps/sample_app/                  # git repository
├── pyproject.toml
├── README.md
├── license.txt
├── .gitignore                    # not written with --no-git
├── .pre-commit-config.yaml
├── .editorconfig
├── .eslintrc
├── .github/workflows/            # only if you answer yes to the workflow prompt
└── sample_app/                   # Python package
    ├── __init__.py               # __version__ = "0.0.1"
    ├── hooks.py
    ├── modules.txt
    ├── patches.txt
    ├── config/__init__.py
    ├── patches/__init__.py
    ├── public/
    │   ├── .gitkeep
    │   ├── css/
    │   └── js/
    ├── templates/
    │   ├── __init__.py
    │   ├── includes/
    │   └── pages/__init__.py
    ├── www/
    └── sample_app/               # first module, named from the App Title
        ├── __init__.py
        └── .frappe
```

| Path | Purpose |
|---|---|
| `pyproject.toml` | Package metadata. Build backend is `flit_core`. The version is read from `__init__.py`. |
| `sample_app/__init__.py` | Holds `__version__`. |
| `sample_app/hooks.py` | App metadata and hooks. See [Hooks](hooks.md). |
| `sample_app/modules.txt` | One module name per line. `frappe.get_module_list` reads the modules of an app from this file. |
| `sample_app/patches.txt` | Patches that `bench migrate` runs. It has the sections `[pre_model_sync]` and `[post_model_sync]`. See [Fixtures and Patches](fixtures-and-patches.md). |
| `sample_app/public/` | Static assets. Served at `/assets/sample_app/`. |
| `sample_app/templates/` | Jinja templates for web pages and includes. |
| `sample_app/www/` | Web pages that map to URL paths. See [Website](../website/website.md). |
| `sample_app/<module>/` | A module folder. The boilerplate also writes an empty `.frappe` file in it. |

The first module folder name is the scrubbed App Title. For the title "Sample App" it is `sample_app`. This is the same as the package name, so the path is `sample_app/sample_app/sample_app/`.

### pyproject.toml

The generated file sets `requires-python = ">=3.14"` and `ruff` settings. Two bench-specific tables are present:

| Table | Use |
|---|---|
| `[tool.bench.dev-dependencies]` | Python packages that bench installs only when developer mode is on. |
| `[deploy.dependencies.apt]` | `packages` list of apt packages for Frappe Cloud. |

Do not add `frappe` to `dependencies`. Bench installs and manages it.

### Module folders

When you save a standard DocType in developer mode, Frappe writes its files into the module folder:

```text
sample_app/selling/                 # module "Selling"
└── doctype/
    └── sample_order/
        ├── __init__.py
        ├── sample_order.json       # the DocType definition
        ├── sample_order.py         # the controller
        ├── sample_order.js         # the form script
        └── test_sample_order.py    # the test file
```

Folder and file names are the scrubbed names (lowercase, spaces become underscores). See [DocTypes](../data-model/doctypes.md) for the JSON and [Controllers](../data-model/controllers.md) for the Python class.

To add a module, add its name to `modules.txt`, make the folder with an `__init__.py`, and run `bench --site <site> migrate`. `bench migrate` adds missing `Module Def` records for every installed app.

## Install an app on a site

1. Get the app into the bench. Use `bench new-app` or `bench get-app <git-url>`.
2. Install it on the site.

```bash
bench --site frappe.localhost install-app sample_app
```

`install-app` accepts more than one app. Use `--force` to install again over an existing install.

`frappe.installer.install_app` runs these steps in this order:

1. Install each app in `required_apps` first.
2. If the app is already installed, print a message and stop. `--force` skips this check.
3. Print a warning if the app overrides a DocType class that another app already overrides.
4. Check that the user has the `System Manager` role (not for `frappe`).
5. Run the `before_install` hooks of the app. If one returns `False`, stop.
6. Run the `before_app_install` hooks of the installed apps. Each gets the app name.
7. Add a `Module Def` for each module in `modules.txt`.
8. Sync all DocTypes of the app from their JSON files.
9. Add the app to `installed_apps` and commit.
10. Run the `after_install` hooks of the app.
11. Mark all patches in `patches.txt` as done. A new install does not run old patches.
12. Run the `after_app_install` hooks of the installed apps.
13. Sync scheduled jobs, fixtures, customizations and dashboards.
14. Run the `after_sync` hooks of the app.

If a step after step 9 fails, Frappe rolls back and removes the app from `installed_apps`.

To remove an app from a site, use `bench --site <site> uninstall-app sample_app`. Add `--dry-run` to list the DocTypes that it deletes. Other options are `--yes` (no prompt), `--no-backup` and `--force`.

## Required apps

Declare other apps that your app needs in `hooks.py`:

```python
# sample_app/hooks.py
required_apps = ["frappe/erpnext"]
```

An entry can be an app name (`erpnext`), an `org/repo` (`frappe/erpnext`) or a git URL. `install-app` installs the required apps first. `bench get-app --resolve-deps` also fetches them.

## Build assets

Frappe bundles files whose name matches `*.bundle.{js,ts,css,sass,scss,less,styl,jsx}` under `public/`. Other files in `public/` are served as they are.

```bash
bench build --app sample_app
```

| Option | Description |
|---|---|
| `--app TEXT` | Build assets for one app. |
| `--apps TEXT` | Build assets for some apps. |
| `--production` | Build in production mode. |
| `--hard-link` | Copy the files instead of making symlinks. |
| `--force` | Build the assets instead of downloading available ones. |
| `--save-metafiles` | Save esbuild metafiles to analyze bundle size. |
| `--using-cached` | Use cached build artifacts. Ignored in developer mode. |
| `--verbose` | Show more output. |

`sites/assets/sample_app` is a symlink to `apps/sample_app/sample_app/public`. Built bundles go into `public/dist/`. Include a bundle in the desk with the `app_include_js` or `app_include_css` hook:

```python
# sample_app/hooks.py
app_include_js = "sample_app.bundle.js"     # built from public/js/sample_app.bundle.js
app_include_css = "sample_app.bundle.css"
```

During development, `bench watch` rebuilds bundles when files change. The `watch` process in the bench `Procfile` runs it. Use `bench watch --apps sample_app` to watch some apps only.

## Pitfalls

- Do not delete `public/.gitkeep`. Without the `public` folder, `bench build` does not link the assets of the app.
- `bench make-app` does not install the app into the environment. Use `bench new-app`, or run `bench get-app` on the folder.
- A new module is not visible on a site until `modules.txt` lists it and you run `bench migrate`.
- You can save standard DocTypes only in developer mode. Set `developer_mode` to `1` in `site_config.json` on your development site. See [Configuration](../operations/configuration.md).

## Related

- [Quickstart](../../getting-started/quickstart.md)
- [Hooks](hooks.md)
- [DocTypes](../data-model/doctypes.md)
- [Fixtures and Patches](fixtures-and-patches.md)
- [CLI reference](../../reference/configuration/cli.md)
