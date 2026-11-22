---
title: Files
description: Store and attach files with the File DocType, upload them over HTTP, choose public or private storage, and know the alpha File Storage v2 flag.
version: develop
sources:
  - frappe/core/doctype/file/file.py
  - frappe/core/doctype/file/file.json
  - frappe/core/doctype/file/file_v1.py
  - frappe/core/doctype/doctype/doctype.json
  - frappe/handler.py
  - frappe/core/api/file.py
  - frappe/utils/file_manager.py
  - frappe/storage/__init__.py
  - frappe/storage/SPEC.md
  - frappe/storage/driver.py
---

# Files

Every uploaded or generated file in Frappe has a `File` document. The document stores the name, URL, size, folder, privacy and the document it is attached to. The bytes are on disk in the site folder (storage v1, the default). Use the `File` DocType from code when you make, attach or read files. Use the `upload_file` endpoint from a client.

## Public and private files

| | Public | Private |
|---|---|---|
| `is_private` | 0 | 1 |
| Disk path | `sites/<site>/public/files/<file_name>` | `sites/<site>/private/files/<file_name>` |
| `file_url` | `/files/<file_name>` | `/private/files/<file_name>` |
| Who can read | Anyone with the URL. nginx serves it with no permission check. | Users who pass the `File` permission check. Guests get HTTP 403. |

Read permission on a private file is given when one of these is true:

- The user is `Administrator`, or the owner of the file.
- The file is shared with the user.
- The file is attached to a document and the user can read that document. Write, create and delete on the file need write permission on the document.

The `ignore_file_permissions` hook turns off these checks for all files.

Make files private unless they must be public, such as website images. DocType settings change the defaults:

- **Force Private Attachments** (`force_private_attachments`): files attached to this DocType cannot be public.
- **Make Attachments Public by Default** (`make_attachments_public`): the desk uploader selects "public" by default.
- System Settings **Only Allow System Managers to Upload Public Files** (`only_allow_system_managers_to_upload_public_files`).

## File fields

| Field | Description |
|---|---|
| `file_name` | File name. |
| `file_url` | URL of the file. For a remote file, the full `http(s)` URL. |
| `is_private` | Private file. |
| `file_size` | Size in bytes. |
| `file_type` | Extension in capitals, such as `PDF`. |
| `content_hash` | MD5 hash of the content. |
| `folder` | Parent folder, such as `Home` or `Home/Attachments`. |
| `is_folder` | The record is a folder. |
| `attached_to_doctype`, `attached_to_name` | Document the file is attached to. |
| `attached_to_field` | Attach field of that document that holds the file URL. |
| `thumbnail_url` | Thumbnail of an image. |
| `blob` | Link to `File Blob`. Only set with storage v2. |

## Make a file from code

```python
file = frappe.get_doc(
	{
		"doctype": "File",
		"file_name": "order-summary.txt",
		"attached_to_doctype": "Sample Order",
		"attached_to_name": "SO-00001",
		"content": b"Total: 100",
		"is_private": 1,
	}
).insert()

file.file_url        # /private/files/order-summary.txt
file.get_content()   # b"Total: 100"
file.get_full_path() # ./<site>/private/files/order-summary.txt
```

- `content` can be `bytes` or `str`. Set `"decode": True` if the content is base64.
- The insert checks write permission on the attached document, unless you set `ignore_permissions=True`.
- If a different file with the same name exists on disk, Frappe adds a suffix to the new file name. The suffix is the last 6 characters of the content hash, or a random one if that name is taken too.
- If a file with the same content hash and privacy exists, the new `File` row uses the same file on disk.
- Without `folder`, an attached file goes to `Home/Attachments` and a file that is not attached goes to `Home`.
- `get_content()` returns `str` for text files and `bytes` for binary files.
- Deleting a `File` row deletes the file on disk after commit, when no other `File` row has the same `file_url`.
- You cannot delete a `File` while its URL is still set in an Attach field of the attached document. Clear the field first, or set `file.flags.force_delete = True`.

For a remote file, set `file_url` to the full URL and no `content`.

### Attach fields

An `Attach` or `Attach Image` field stores the `file_url`. To fill one from code, make the `File` with `attached_to_field` set, then set the field:

```python
order = frappe.get_doc("Sample Order", "SO-00001")
file = frappe.get_doc(
	{
		"doctype": "File",
		"file_name": "signed.pdf",
		"attached_to_doctype": "Sample Order",
		"attached_to_name": order.name,
		"attached_to_field": "signed_copy",  # an Attach field on Sample Order
		"content": pdf_bytes,
		"is_private": 1,
	}
).insert()
order.db_set("signed_copy", file.file_url)
```

An `Attach Image` field accepts only image files.

### Legacy helper

`frappe.utils.file_manager.save_file(fname, content, dt, dn, folder=None, decode=False, is_private=0, df=None)` makes a `File` and returns it. New code can make the `File` document directly.

## Upload over HTTP

`POST /api/method/upload_file` takes `multipart/form-data`. It is rate-limited to 300 calls per minute per user.

```bash
curl -X POST "http://frappe.localhost:8000/api/method/upload_file" \
  -H "Authorization: token <api_key>:<api_secret>" \
  -F "file=@summary.pdf" \
  --form-string "doctype=Sample Order" \
  --form-string "docname=SO-00001" \
  --form-string "is_private=1"
```

The response `message` is the new `File` document.

| Form field | Default | Description |
|---|---|---|
| `file` | | The file. |
| `is_private` | `1` | `0` for a public file. |
| `doctype`, `docname` | none | Attach to this document. Needs write permission on it. |
| `fieldname` | none | Attach field to fill. |
| `folder` | `Home` | Target folder. |
| `file_url` | none | Make a `File` for an existing URL instead of uploading bytes. |
| `file_name` | file name | Name for a `file_url` upload. |
| `library_file_name` | none | Name of an existing `File` to attach again. |
| `optimize` | off | Compress an image or a PDF. Use `max_width` and `max_height` for images. |
| `method` | none | Whitelisted method to call instead of making a `File`. It reads `frappe.local.uploaded_file` and `frappe.local.uploaded_filename`. |
| `chunk_index`, `total_chunk_count`, `chunk_byte_offset`, `total_file_size` | none | Chunked upload. The desk sends chunks of `file_chunk_size` bytes. |

Rules:

- Guests can upload only when System Settings **Allow Guests to Upload Files** is on. **Allowed DocTypes for Guest Uploads** limits the target DocTypes.
- Guests and website users can upload only JPG, PNG, GIF, PDF, TXT, CSV and Microsoft Office files.
- The size limit is System Settings **Max File Size** in MB, else `max_file_size` in site config in bytes, else 25 MB.
- Each function in the `after_file_upload` hook gets `doc=<File>` before save and must return the document.

To download a file with a token, call `GET /api/method/download_file?file_url=/private/files/<name>`. It applies the same permission checks.

## Storage hooks

| Hook | Description |
|---|---|
| `before_write_file` | Called with `file_size` before content is written. |
| `write_file` | Replaces how file content is written. Used by storage v1 and `save_file`. |
| `delete_file_data_content` | Replaces how file content is deleted. |
| `after_file_upload` | Changes the `File` document in `upload_file` before save. |
| `ignore_file_permissions` | Any true value turns off the File permission checks. |
| `storage_drivers` | Registers File Storage v2 drivers. |

## File Storage v2 (alpha)

!!! warning "Alpha"
    File Storage v2 is experimental. Its design, APIs, URLs and tables can change or be removed. Do not build on it outside Frappe's own apps.

Storage v2 is off by default. Turn it on per site with the `storage_v2` site config key:

```bash
bench --site frappe.localhost set-config storage_v2 1
```

With the flag on:

- New uploads make a `File Blob` row (checksum, size, sniffed MIME type, driver, key) and the `File.blob` link.
- Bytes go through a storage driver. Core has `local` (default), `s3` and `memory` (tests). Select one with `storage_driver`, and configure it with `storage_driver_config`.
- New file URLs have two parts: `/files/<blob>/<file_name>` and `/private/files/<blob>/<file_name>`. The random blob name makes public URLs hard to guess.
- Private files are served after a permission check. Drivers such as `s3` can serve them through signed, expiring URLs.
- Old v1 files keep their URLs and stay readable, so you can turn on the flag on an existing site.
- `frappe.storage.gc.collect_garbage` runs daily and deletes blobs that no `File` row uses.

Check the flag in code with `frappe.storage.enabled()`. The full design is in `frappe/storage/SPEC.md`.

## Pitfalls

- Public files have no permission check. Anyone with the URL can download them.
- A file name is not unique. Find a `File` by `file_url` and the attached document, not by name alone.
- Changing `is_private` moves the file and changes its URL. Frappe updates the `attached_to_field` of the attached document. Other places that store the old URL keep it.
- Files that you write to disk without a `File` row are not listed in the desk and are never cleaned up.

## Related

- [REST API](../api/rest-api.md)
- [Security](../security/security.md)
- [Configuration](../operations/configuration.md)
- [Form View](../../user-guide/basics/form-view.md)
- [Field types reference](../../reference/data-model/field-types.md)
