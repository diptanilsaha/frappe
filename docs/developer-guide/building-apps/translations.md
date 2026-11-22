---
title: Translations
description: Mark strings with _() in Python and __() in JS, use context and lazy strings, keep PO/POT files in an app, and run the bench translation commands.
version: develop
sources:
  - frappe/utils/translations.py
  - frappe/translate.py
  - frappe/gettext/translate.py
  - frappe/gettext/extractors/README.md
  - frappe/types/lazytranslatedstring.py
  - frappe/commands/gettext.py
  - frappe/commands/translate.py
  - frappe/public/js/frappe/translate.js
  - frappe/desk/search.py
  - frappe/boot.py
  - frappe/model/meta.py
  - babel_extractors.csv
---

# Translations

Frappe translates user-facing strings at run time. You mark a string in code, and Frappe looks it up in the translations of the user's language. Translations come from PO files in each app, from older CSV files, and from `Translation` documents that users add in the desk. Mark every string that a user sees.

## Mark strings in Python

```python
import frappe
from frappe import _

frappe.throw(_("Rate must be more than zero"), title=_("Invalid Rate"))
frappe.msgprint(_("{0} is mandatory").format(_("Rate")))
_("Change", context="Coins")
```

`_(msg, lang=None, context=None)` returns the translation in `lang`, or in `frappe.local.lang` when `lang` is empty. When no translation exists, it returns `msg`.

```python
_("Save", lang="de")                           # 'Speichern'
_("{0} is mandatory", lang="de").format("Rate")  # 'Rate ist erforderlich'
_("Unknown sample string", lang="de")          # 'Unknown sample string'
```

Rules for the message:

- Pass a literal string. The extractor reads the source code, so `_(variable)` and f-strings are not extracted.
- Use `{0}`, `{1}` placeholders and call `.format()` on the result. Do not format before you translate.
- Use `context` when one English word has more than one meaning. Frappe looks up `"<msg>:<context>"` first, then `msg`.

### Lazy strings

At import time, the language is not known. Use `_lt()` for strings in module scope, for example in class attributes or constants. It returns a `_LazyTranslate` object that translates when it is turned into a string.

```python
from frappe import _lt

STATUS_LABELS = {"Open": _lt("Open"), "Completed": _lt("Completed")}

str(STATUS_LABELS["Open"])  # translated in the current language
```

`N_(msg)` marks a string for extraction and returns it unchanged. Use it when you translate the value later with `_()`.

### Print in another language

`frappe.translate.print_language(lang)` is a context manager. Inside it, `_()` and Jinja templates use `lang`.

```python
from frappe.translate import print_language

with print_language("de"):
	pdf = frappe.get_print("Sample Order", "SO-00001", as_pdf=True)
```

## Mark strings in JavaScript

`__(txt, replace, context)` is the client function (`frappe/public/js/frappe/translate.js`). `replace` is a list of values for `{0}`, `{1}`.

```js
frappe.msgprint(__("Rate must be more than zero"));
frappe.show_alert(__("{0} items added", [count]));
__("Change", null, "Coins");
```

The desk loads the translations of the user's language into `frappe._messages` at boot (`frappe.translate.get_boot_translations`, cached by the browser).

## Strings in Jinja, DocTypes and JSON

- In Jinja templates, use `{{ _("Total") }}`. `_` is a template global.
- DocType labels, descriptions, Select options, workspace labels, report names and Web Form labels are extracted from the JSON files. The desk translates them with `__()`.
- Field values are not translated. To show translated values of a Link field, set `translated_doctype` on the target DocType. The desk then translates link titles, and link search matches the translated text (`frappe/desk/search.py`).
- `translatable` on a DocField adds a translate button next to the field in the form. Users with write access to `Translation` can add a translation of the field value there.

## Where translations come from

`get_all_translations(lang)` merges these sources and caches the result in Redis:

1. CSV files: `<app>/translations/<lang>.csv` (older format).
2. MO files: `sites/assets/locale/<lang>/LC_MESSAGES/<app>.mo`, compiled from `<app>/locale/<lang>.po`.
3. `Translation` documents for the language (made in the desk).
4. Translated country names.

A child language (for example `es-GT`) first gets the parent language (`es`), then its own values. Later sources win.

## PO files in an app

```text
sample_app/sample_app/locale/
├── main.pot     # all strings of the app, made by generate-pot-file
├── de.po        # German translations
└── fr.po
```

1. Run `bench generate-pot-file --app sample_app` to extract strings into `main.pot`.
2. Run `bench create-po-file de --app sample_app` to start a new language.
3. Run `bench update-po-files --app sample_app` to sync existing PO files with the POT file.
4. Translate the `msgstr` entries in the PO file.
5. Run `bench compile-po-to-mo --app sample_app` to build the MO files.
6. Run `bench --site <site> clear-cache` so the site loads the new translations.

`bench build` also compiles PO files when they changed. Use `bench build --force` to compile all.

The extractor (Babel with `frappe/gettext/extractors`) finds `_()`, `_lt()` and `N_()` in Python, `__()` in JS, HTML and Vue files, and labels in DocType, report, workspace and Web Form JSON. The mapping of file patterns to extractors is in `babel_extractors.csv` at the repository root. An app can add its own `babel_extractors.csv` next to its package. Its rules run before the Frappe rules. Folders that start with `.` or `_`, and files ignored by git, are skipped.

To keep strings that another app already has out of your POT file, set this hook:

```python
# sample_app/hooks.py
ignore_translatable_strings_from = ["frappe"]
```

## Commands

| Command | Description |
|---|---|
| `bench generate-pot-file [--app APP]` | Extract strings into `<app>/locale/main.pot`. |
| `bench create-po-file LOCALE [--app APP]` | Create `<app>/locale/<locale>.po`. |
| `bench update-po-files [--app APP] [--locale LOCALE]` | Sync PO files with the POT file. |
| `bench compile-po-to-mo [--app APP] [--locale LOCALE] [--force]` | Compile PO files to MO files. |
| `bench migrate-csv-to-po [--app APP] [--locale LOCALE]` | Move old CSV translations into PO files. |
| `bench --site SITE get-untranslated LANG FILE [--app APP] [--all]` | Write untranslated strings (CSV workflow). |
| `bench --site SITE update-translations LANG UNTRANSLATED TRANSLATED [--app APP]` | Merge a translated file (CSV workflow). |
| `bench --site SITE import-translations LANG PATH` | Import a CSV file of translations. |
| `bench --site SITE migrate-translations SOURCE_APP TARGET_APP` | Move app-specific translations between apps. |
| `bench --site SITE build-message-files` | Rebuild the translation message files. |
| `bench --site SITE new-language LANG_CODE APP` | Create a CSV file for a language. |

Without `--app`, the gettext commands work on all apps and need a site (`bench --site <site> ...`).

## Pitfalls

- Translate the template, then format: `_("{0} saved").format(name)`, not `_(f"{name} saved")`.
- Do not join translated pieces into a sentence. Word order changes between languages. Translate the full sentence with placeholders.
- `_()` at module level runs before a request sets the language. Use `_lt()` there.
- Background jobs use the language of the user who queued the job. It is set on the first call to `_()`.
- After you change translations, clear the cache. Translations are cached per language.

## Related

- [Apps](apps.md)
- [Client API](../desk-ui/client-api.md)
- [Print Formats](../output/print-formats.md)
- [CLI reference](../../reference/configuration/cli.md)
