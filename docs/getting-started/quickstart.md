---
title: Quickstart
description: Build a small Frappe v15 app end to end with a DocType, a controller validation, a whitelisted method, a form script, a list view setting and a test.
version: v15
sources:
  - frappe/utils/boilerplate.py
  - frappe/commands/utils.py
  - frappe/commands/site.py
  - frappe/core/doctype/doctype/doctype.py
  - frappe/core/doctype/doctype/boilerplate/controller._py
  - frappe/core/doctype/doctype/boilerplate/test_controller._py
  - frappe/core/doctype/doctype/boilerplate/controller.js
  - frappe/modules/utils.py
  - frappe/desk/form/meta.py
  - frappe/public/js/frappe/model/indicator.js
  - frappe/__init__.py
  - frappe/utils/typing_validations.py
  - frappe/tests/utils.py
  - frappe/installer.py
---

# Quickstart

This page builds a small app, `sample_app`, with one DocType, `Sample Item`. You add a server validation, a whitelisted method, a form button, a list indicator and a test. Before you start, set up a bench and a site as shown in [Installation](installation.md). The examples use the site `sample.localhost`.

## 1. Create the app

1. In the bench folder, create the app:

    ```bash
    bench new-app sample_app
    ```

2. Answer the prompts:

    | Prompt | Default | Note |
    |---|---|---|
    | App Title | `Sample App` | Also the name of the first module. Must start with a letter. |
    | App Description | none | |
    | App Publisher | none | |
    | App Email | none | Must be a valid email address. |
    | App License | `mit` | |
    | Create GitHub Workflow action for unittests | `False` | Adds `.github/workflows/ci.yml` and `linter.yml`. |

`bench new-app` runs `bench make-app apps sample_app`. Then it installs the app into the bench Python environment and adds it to `sites/apps.txt`. Use `--no-git` to skip the Git repository.

The new app has this layout:

```text
apps/sample_app/
├── .editorconfig
├── .eslintrc
├── .gitignore
├── .pre-commit-config.yaml
├── README.md
├── license.txt
├── pyproject.toml
└── sample_app/
    ├── __init__.py
    ├── hooks.py
    ├── modules.txt          # "Sample App"
    ├── patches.txt
    ├── config/
    ├── patches/
    ├── public/
    │   ├── css/
    │   └── js/
    ├── templates/
    │   ├── includes/
    │   └── pages/
    ├── www/
    └── sample_app/          # module "Sample App"
        └── __init__.py
```

The generated `hooks.py` has most hooks as comments. To let Frappe write field type hints into your controllers, remove the `#` from this line:

```python
# sample_app/hooks.py
export_python_type_annotations = True
```

## 2. Install the app on the site

1. Install the app:

    ```bash
    bench --site sample.localhost install-app sample_app
    ```

2. Turn on developer mode, so that DocTypes you make are saved as files in the app:

    ```bash
    bench --site sample.localhost set-config developer_mode 1
    ```

3. Start the bench:

    ```bash
    bench start
    ```

## 3. Create the DocType

1. Open `http://sample.localhost:8000/app/doctype/new`.
2. Set **Name** to `Sample Item`.
3. Set **Module** to `Sample App`.
4. Leave **Custom?** cleared.
5. Add these fields:

    | Label | Fieldname | Type | Options | Mandatory |
    |---|---|---|---|---|
    | Item Name | `item_name` | Data | | Yes |
    | Category | `category` | Select | `Hardware`, `Software`, `Service` (one per line) | Yes |
    | Rate | `rate` | Currency | | No |
    | Disabled | `disabled` | Check | | No |
    | Description | `description` | Text Editor | | No |

6. In **Naming**, set **Auto Name** to `field:item_name`.
7. In **Permissions**, keep the row for `System Manager`.
8. Save.

Frappe makes the table `tabSample Item` and writes these files to `apps/sample_app/sample_app/sample_app/doctype/sample_item/`:

| File | Content |
|---|---|
| `sample_item.json` | DocType definition. |
| `sample_item.py` | Python controller. |
| `sample_item.js` | Form script. |
| `test_sample_item.py` | Test class. |
| `__init__.py` | Package marker. |

## 4. Add a validation and a whitelisted method

Edit the controller. If you turned on `export_python_type_annotations`, the file has a block between `# begin: auto-generated types` and `# end: auto-generated types`. Keep that block as it is. Frappe rewrites it when you save the DocType.

```python
# sample_app/sample_app/doctype/sample_item/sample_item.py
import frappe
from frappe import _
from frappe.model.document import Document
from frappe.utils import flt


class SampleItem(Document):
	def validate(self):
		if flt(self.rate) <= 0:
			frappe.throw(_("Rate must be greater than zero."), title=_("Invalid Rate"))


@frappe.whitelist()
def get_items_by_category(category: str) -> list[dict]:
	return frappe.get_list(
		"Sample Item",
		filters={"category": category, "disabled": 0},
		fields=["name", "item_name", "rate"],
		order_by="rate desc",
	)
```

- `validate()` runs before each insert and save. `frappe.throw()` raises `frappe.ValidationError` and stops the save.
- `get_items_by_category()` is callable at `/api/method/sample_app.sample_app.doctype.sample_item.sample_item.get_items_by_category`.
- `frappe.get_list()` applies the read permissions of the current user.
- In a request, Frappe checks the arguments against the type annotations (`frappe/utils/typing_validations.py`).

Test the method with `curl`. Log in first and keep the session cookie in the file `jar`:

```bash
curl -c jar -X POST http://sample.localhost:8000/api/method/login \
  -H 'Content-Type: application/json' -d '{"usr":"Administrator","pwd":"<admin-password>"}'
curl -b jar "http://sample.localhost:8000/api/method/sample_app.sample_app.doctype.sample_item.sample_item.get_items_by_category?category=Hardware"
```

The response has this shape:

```json
{"message": [{"name": "Laptop", "item_name": "Laptop", "rate": 900.0}]}
```

## 5. Add a form script

The form script runs in the desk when a `Sample Item` form opens.

```js
// sample_app/sample_app/doctype/sample_item/sample_item.js
frappe.ui.form.on("Sample Item", {
	refresh(frm) {
		if (frm.is_new()) return;

		frm.add_custom_button(__("Same Category"), () => {
			frappe.call({
				method: "sample_app.sample_app.doctype.sample_item.sample_item.get_items_by_category",
				args: { category: frm.doc.category },
				callback(r) {
					frappe.msgprint(r.message.map((d) => d.item_name).join(", "));
				},
			});
		});
	},
});
```

After you change JS files, reload the page. `bench watch` (part of `bench start`) rebuilds the assets.

## 6. Add a list view setting

Make a file `<doctype>_list.js` next to the controller. The desk loads it for the list view. Frappe does not make this file for you.

```js
// sample_app/sample_app/doctype/sample_item/sample_item_list.js
frappe.listview_settings["Sample Item"] = {
	add_fields: ["disabled"],
	get_indicator(doc) {
		if (doc.disabled) {
			return [__("Disabled"), "gray", "disabled,=,1"];
		}
		return [__("Enabled"), "green", "disabled,=,0"];
	},
};
```

`get_indicator` returns `[label, color, filter]`. A click on the indicator applies the filter. Open `/app/sample-item` to see the list.

## 7. Write a test

Edit the generated test file:

```python
# sample_app/sample_app/doctype/sample_item/test_sample_item.py
import frappe
from frappe.tests.utils import FrappeTestCase


class TestSampleItem(FrappeTestCase):
	def test_rate_must_be_positive(self):
		doc = frappe.get_doc(
			{"doctype": "Sample Item", "item_name": "Test Item", "category": "Hardware", "rate": 0}
		)
		self.assertRaises(frappe.ValidationError, doc.insert)

	def test_get_items_by_category(self):
		from sample_app.sample_app.doctype.sample_item.sample_item import get_items_by_category

		frappe.get_doc(
			{"doctype": "Sample Item", "item_name": "Test Mouse", "category": "Hardware", "rate": 20}
		).insert()
		names = [d.name for d in get_items_by_category("Hardware")]
		self.assertIn("Test Mouse", names)
```

1. Allow tests on the site:

    ```bash
    bench --site sample.localhost set-config allow_tests true
    ```

2. Run the tests of the DocType:

    ```bash
    bench --site sample.localhost run-tests --doctype "Sample Item"
    ```

`FrappeTestCase` (`frappe/tests/utils.py`) commits before the class starts and rolls back the database changes of the test class when the class is done.

## 8. Install the app on another site

1. Commit the app code to Git.
2. On the other bench, get the app with `bench get-app <git-url>`.
3. Install the app:

    ```bash
    bench --site <site> install-app sample_app
    ```

`install-app` makes the tables of all DocTypes in the app and runs the `after_install` hooks. After you pull new code, run `bench --site <site> migrate`.

## Pitfalls

- Without `developer_mode`, saving a DocType with **Custom?** cleared fails with "Not in Developer Mode".
- The dotted path of a DocType module repeats the app name: `sample_app.sample_app.doctype...`. The first part is the app package. The second part is the module folder.
- Without `allow_tests` in the site config (or the `CI` environment variable), `run-tests` stops and prints the command to set it.
- Do not edit the DocType JSON by hand. Change it in the desk, so Frappe updates the JSON and the table together.

## Related

- [Architecture](architecture.md)
- [Apps](../developer-guide/building-apps/apps.md)
- [DocTypes](../developer-guide/data-model/doctypes.md)
- [Controllers](../developer-guide/data-model/controllers.md)
- [Whitelisted methods](../developer-guide/api/whitelisted-methods.md)
- [Form scripts](../developer-guide/desk-ui/form-scripts.md)
- [List view settings](../developer-guide/desk-ui/list-view-settings.md)
- [Testing](../developer-guide/testing-and-performance/testing.md)
