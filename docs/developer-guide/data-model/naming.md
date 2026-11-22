---
title: Naming
description: How Frappe sets the name of a document, with autoname options, naming series, the autoname controller method and rename.
version: v15
sources:
  - frappe/model/naming.py
  - frappe/model/document.py
  - frappe/model/rename_doc.py
  - frappe/core/doctype/doctype/doctype.py
  - frappe/core/doctype/doctype/doctype.json
  - frappe/core/doctype/document_naming_settings/document_naming_settings.json
  - frappe/database/mariadb/schema.py
  - frappe/custom/doctype/customize_form/customize_form.py
  - frappe/public/js/frappe/doctype/index.js
  - frappe/__init__.py
---

# Naming

Every document has a `name`. It is the primary key of the table and the value that Link fields store. Frappe sets the name once, on insert, from the naming rule of the DocType. Choose the rule when you design the DocType, because changing it later does not rename old documents. Use [Rename](#rename) to change the name of one document.

## Naming rules

In the DocType form, the **Naming Rule** (`naming_rule`) field sets the **Auto Name** (`autoname`) property. Frappe uses `autoname` at runtime.

| Naming Rule | `autoname` value | Example name |
|---|---|---|
| Set by user | `prompt` | `my-name` |
| Autoincrement | `autoincrement` | `5` (an integer) |
| By fieldname | `field:item_name` | `Support Plan` |
| By "Naming Series" field | `naming_series:` | `SO-2026-00001` |
| Expression | `SO-.YYYY.-.#####` or `{customer_name}-.###` | `SO-2026-00001`, `Customer A-001` |
| Expression (old style) | `format:SO-{MM}-{customer_name}-{#####}` | `SO-11-Customer A-00001` |
| Random | `hash` | `n7jg1d7hvg` |
| By script | empty | Set in the `autoname` controller method. |

If `autoname` is empty and no method sets a name, Frappe uses `hash`. An expression without `#` and without `format:` is also not applied, so the name falls back to `hash`.

### Set by user (`prompt`)

The user types the name in the new form. In code, set `__newname`, or pass `set_name` to `insert`:

```python
doc = frappe.get_doc({"doctype": "Sample Code", "title": "A"})
doc.set("__newname", "CODE-A")
doc.insert()

frappe.get_doc({"doctype": "Sample Code", "title": "B"}).insert(set_name="CODE-B")
```

Without a name, insert fails with "Please set the document name".

### By fieldname (`field:<fieldname>`)

The name is the value of the field, with spaces trimmed at the ends. When you save the DocType, Frappe sets `unique` on that field. If the field is empty on insert, Frappe throws "<label> is required".

### Naming series (`naming_series:`)

The DocType needs a Select field with the fieldname `naming_series`. Each option is a series prefix. The user picks one, and Frappe adds `.#####` to it.

```text
SO-.YYYY.-
SR-.YYYY.-
```

If the document has no `naming_series` value, Frappe uses the first non-empty option. Frappe keeps the counters in the `Series` table (`tabSeries`), one row per prefix.

!!! note
    The field must be named `naming_series`. The desk describes the format as `naming_series:[fieldname]`, but the code reads only `doc.naming_series`.

### Expression

An expression has parts separated by dots. Frappe joins the parts:

| Part | Value |
|---|---|
| `#`, `##`, `#####` ... | Next number of the counter for the prefix built so far. The count of `#` sets the digits. |
| `YY`, `YYYY` | Year, 2 or 4 digits. |
| `MM`, `DD` | Month, day. |
| `WW` | Calendar week. |
| A key of the `naming_series_variables` hook | Value from the hook function. See below. |
| `timestamp` | Current date and time. |
| `{fieldname}` or `fieldname` | Value of that field of the document. |
| Any other text | Used as it is. For example `JJJ` is not a date part. |

`SO-.YYYY.-.#####` makes `SO-2026-00001`, then `SO-2026-00002`. The counter key is `SO-2026-`, so the count starts again each year.

`SO-.customer_name.-.####` uses a separate counter for each customer: `SO-Customer A-0001`, `SO-Customer B-0001`.

### Expression (old style, `format:`)

`format:` replaces every braced part and keeps other text. Frappe shows a warning when you save a DocType with `format:`.

!!! warning
    A braced number part such as `{#####}` evaluates with an empty prefix. All `format:` patterns that use `{#...}` share one counter, the row with an empty `name` in `tabSeries`. Use the dot syntax (`SO-.#####`) when you need a counter per prefix.

### Autoincrement

The name is an integer from a database sequence. The `name` column is an integer column (`bigint` on MariaDB). You can change to or from `autoincrement` only while the DocType has no records. Customize Form cannot change it. Frappe sets `allow_rename` to 0 for these DocTypes.

## Naming order

`Document.insert` calls `set_new_name` after `before_insert`. `frappe.model.naming.set_new_name` runs these steps:

1. Run `before_naming` on the document.
2. Clear `doc.name`, unless `autoname` is `prompt` or the document is imported.
3. If `autoname` is `autoincrement`, take the next sequence value and stop.
4. If the document has `amended_from`, make the amended name. See below.
5. If the DocType is a Single, use the DocType name.
6. Apply the enabled `Document Naming Rule` records for the DocType, by priority.
7. Run the `autoname` controller method.
8. Apply the `autoname` property of the DocType.
9. If there is still no name, use `hash`.
10. Validate the name. It cannot contain `<` or `>`, and it cannot equal the DocType name.

`insert(set_name="...")` and a `prompt` name with `__newname` skip these steps. Child rows are named with the same steps. Step 3 returns early, so the `autoname` method does not run for `autoincrement`. Step 2 means that a `name` you set in the dict before `insert` is ignored. Use `set_name` instead.

See [Automation](../../user-guide/automation/automation.md) for `Document Naming Rule` and `Document Naming Settings`.

## The autoname method

Set the name in code with an `autoname` method on the controller. Leave the naming rule empty or set it to "By script".

```python
# sample_app/selling/doctype/sample_order/sample_order.py
import frappe
from frappe.model.document import Document
from frappe.model.naming import make_autoname


class SampleOrder(Document):
	def autoname(self):
		prefix = "SO-" + frappe.scrub(self.customer_name)[:3].upper() + "-.####"
		self.name = make_autoname(prefix, doc=self)
```

Use `before_naming` to set fields that the naming rule reads, such as `naming_series`.

### Helpers

| Function | Description |
|---|---|
| `make_autoname(key, doctype="", doc="")` | Returns a name from an expression. `"hash"` returns a random 10-character name. |
| `NamingSeries(series).get_preview(doc=None)` | Returns three sample names without using the counter. |
| `NamingSeries(series).get_current_value()` | Returns the current counter value. |
| `NamingSeries(series).update_counter(new_count)` | Sets the counter. A wrong value can make duplicate names. |
| `append_number_if_name_exists(doctype, value)` | Returns `value`, or `value-1`, `value-2` ... if it exists. |

```python
from frappe.model.naming import NamingSeries, make_autoname

make_autoname("SAMPLE-.YY.-.#####")          # 'SAMPLE-26-00001', uses the counter
NamingSeries("SAMPLE-.YYYY.-.###").get_preview()
# ['SAMPLE-2026-001', 'SAMPLE-2026-002', 'SAMPLE-2026-003']
```

### Custom series parts

Add your own expression part with the `naming_series_variables` hook. The function gets the document and the part and returns a string.

```python
# sample_app/hooks.py
naming_series_variables = {
	"REGION": "sample_app.naming.get_region",
}
```

```python
# sample_app/naming.py
def get_region(doc, part):
	return doc.get("region_code") or "XX"
```

Now `SO-.REGION.-.####` makes `SO-EU-0001`.

## Amended documents

When you amend a cancelled document, the new document has `amended_from`. With the default setting (`Amend Counter` in the **Default Amendment Naming** field, `default_amend_naming`, of **Document Naming Settings**), the name is the original name with a counter: `SO-00016` becomes `SO-00016-1`, then `SO-00016-2`. Set `Default Naming` to use the normal naming rule instead. The **Amended Document Naming Settings** table sets this per DocType.

## Rename

Rename changes the `name` and updates every Link and Dynamic Link field that points to it.

```python
doc = frappe.get_doc("Sample Item", "Support Plan")
doc.rename("Sample Gadget")              # needs write permission. Also takes merge, force, validate_rename

frappe.rename_doc("Sample Item", "Sample Gadget", "Support Plan")
frappe.rename_doc("Sample Item", "Sample Old", "Support Plan", merge=True)
```

| Parameter of `frappe.rename_doc` | Default | Description |
|---|---|---|
| `doctype`, `old`, `new` | | DocType, current name, new name. |
| `force` | `False` | Rename even if `allow_rename` is off. |
| `merge` | `False` | Merge into an existing document `new`, then delete `old`. |
| `ignore_if_exists` | `False` | Do not throw if `new` exists. |
| `show_alert` | `True` | Show a message after rename. |
| `rebuild_search` | `True` | Enqueue a rebuild of the global search for the DocType. |

The rename runs `before_rename(old, new, merge)` and `after_rename(old, new, merge)` on the document. `before_rename` can return a new name. Rename checks write permission and the `allow_rename` property. Single DocTypes cannot be renamed. After the rename, Frappe adds a comment "renamed from ... to ..." on the document.

## Pitfalls

- Changing the naming rule does not rename existing documents.
- A `field:` name changes only by rename. Editing the field later does not change the name.
- Names from `hash` are random. Do not use them in URLs that users must type.
- Deleting the last document of a series can lower the counter by one (`revert_series_if_last`). Do not rely on gaps.
- A name that is a number is stored as a string, except with `autoincrement`.

## Related

- [DocTypes](doctypes.md)
- [Controllers](controllers.md)
- [Automation](../../user-guide/automation/automation.md)
