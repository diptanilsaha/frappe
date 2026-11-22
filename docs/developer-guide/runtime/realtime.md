---
title: Realtime
description: Push events from the server to browsers with publish_realtime, Socket.IO rooms on the Node server, and frappe.realtime.on.
version: v15
sources:
  - frappe/realtime.py
  - socketio.js
  - realtime/index.js
  - realtime/handlers/frappe_handlers.js
  - realtime/middlewares/authenticate.js
  - node_utils.js
  - frappe/public/js/frappe/socketio_client.js
  - frappe/model/document.py
  - frappe/commands/site.py
  - frappe/sessions.py
---

# Realtime

Use realtime events to push data from the server to open browser tabs: a progress bar, a refresh after a background job, a message to one user. Server code calls `frappe.publish_realtime`. Frappe publishes the event on the Redis channel `events`. The Node Socket.IO server reads the channel and sends the event to the browsers in the target room. Browser code listens with `frappe.realtime.on`.

## Concepts

- **Event**: a name, such as `doc_update` or `sample_order_ready`, and a JSON message.
- **Room**: a group of sockets on one site. Each site is a Socket.IO namespace (`/<site>`).
- **Socket.IO server**: a separate Node process, `apps/frappe/socketio.js`. `bench socketio` starts it. It listens on `socketio_port` (default 9000). It reads the `events` channel from the `redis_queue` Redis.

!!! note "Version"
    v15 has only the Node server. It has no Python Socket.IO server and no custom handlers in Python apps.

### Authentication

On connect, the server calls `frappe.realtime.get_user_info` on the web server with the `sid` cookie or the `Authorization` header. The host and origin of the request must match. The socket gets `user` and `user_type` from the answer.

### Rooms

| Room | Who is in it | How a socket joins |
|---|---|---|
| `user:<user>` | All sockets of that user on the site | On connect |
| `website` | All sockets on the site, guests too | On connect |
| `all` | Sockets of System Users on the site | On connect |
| `doctype:<doctype>` | Sockets that show a list of that DocType | `doctype_subscribe`, needs read permission |
| `doc:<doctype>/<name>` | Sockets that have the form open | `doc_subscribe`, needs read permission |
| `open_doc:<doctype>/<name>` | Sockets that view the form now. Used for `doc_viewers`. | `doc_open`, needs read permission |
| `task_progress:<task_id>` | Sockets that follow a task | `task_subscribe` or `progress_subscribe` |

The desk subscribes for you. A form subscribes to its document on load. A list view subscribes to its DocType. The server checks permission with the whitelisted methods `frappe.realtime.can_subscribe_doc` and `frappe.realtime.can_subscribe_doctype`.

## Publish from the server

```python
frappe.publish_realtime(
	event="sample_order_ready",
	message={"order": "SO-0001", "status": "Ready"},
	user="user1@example.com",
	after_commit=True,
)
```

### Parameters

| Name | Type | Default | Description |
|---|---|---|---|
| `event` | `str` | `"task_progress"` if a task ID is set, else `"global"` | Event name. |
| `message` | `dict` | `{}` | JSON message. |
| `room` | `str` | see below | Room name. Overrides the other targets. |
| `user` | `str` | `None` | Send to the room `user:<user>`. |
| `doctype`, `docname` | `str` | `None` | Send to the room `doc:<doctype>/<docname>`. |
| `task_id` | `str` | `frappe.local.task_id` | Send to the room `task_progress:<task_id>`. Forces `after_commit=False`. |
| `after_commit` | `bool` | `False` | Send only after the transaction commits. Dropped on rollback. Duplicate events are sent once. |

When `room` is not given, the first match wins: `task_id`, then `user`, then `doctype` and `docname`. If none is given, the event goes to the `all` room (System Users of the site).

Special cases:

- `msgprint` without `user` goes to the current user.
- `list_update` always goes to `doctype:<doctype>`. The DocType comes from `doctype` or from `message["doctype"]`.
- `docinfo_update` always goes to `doc:<doctype>/<docname>`.

Use `after_commit=True` when the browser reloads data after the event. Otherwise the browser can read the database before your changes are committed.

`frappe/realtime.py` also has room name helpers: `get_user_room(user)`, `get_doc_room(doctype, docname)`, `get_doctype_room(doctype)`, `get_site_room()`, `get_website_room()` and `get_task_progress_room(task_id)`. Use them with `room=`:

```python
from frappe.realtime import get_website_room

frappe.publish_realtime("sample_banner", {"text": "Closed today"}, room=get_website_room())
```

### Progress bar

`frappe.publish_progress` sends the `progress` event. The desk shows a progress dialog.

```python
def import_items(rows):
	for i, row in enumerate(rows, start=1):
		...
		frappe.publish_progress(i * 100 / len(rows), title="Importing items", description=f"Row {i}")
```

Signature: `publish_progress(percent, title=None, doctype=None, docname=None, description=None, task_id=None)`. With `doctype` and `docname`, everyone with the form open sees the progress. Without them, only the current user sees it.

### Events the framework sends

| Event | Room | When | Message |
|---|---|---|---|
| `doc_update` | `doc:<doctype>/<name>` | After a document is saved (`notify_update`), after commit | `doctype`, `name`, `modified` |
| `list_update` | `doctype:<doctype>` | Same, for DocTypes that are not single, child or read-only | `doctype`, `name`, `user` |
| `docinfo_update` | `doc:<doctype>/<name>` | A Comment or Communication on the document changes | varies |
| `doc_viewers` | `open_doc:<doctype>/<name>` | A user opens or closes the form | `doctype`, `docname`, `users` |
| `msgprint` | `user:<user>` | `frappe.msgprint(..., realtime=True)` | message |
| `progress` | user or doc room | `frappe.publish_progress` | `percent`, `title`, `description` |

`notify_update` does not send events while patches run (`frappe.flags.in_patch`). Set `doc.flags.notify_update = False` to skip it for one save. `doc.db_set(..., notify=True)` also calls it.

### From the CLI

```bash
bench --site frappe.localhost publish-realtime sample_event --message '{"status": "Open"}' --user Administrator
```

Options: `--message`, `--room`, `--user`, `--doctype`, `--docname`, `--after-commit`. The message is sent as a string, not as an object.

## Listen in the browser

```js
// sample_app/public/js/sample_order.js
frappe.ui.form.on("Sample Order", {
	refresh(frm) {
		frappe.realtime.off("sample_order_ready");
		frappe.realtime.on("sample_order_ready", (data) => {
			if (data.order === frm.doc.name) {
				frm.reload_doc();
			}
		});
	},
});
```

Client API (`frappe.realtime`, also `frappe.socketio`):

| Method | Description |
|---|---|
| `on(event, callback)` | Listen for an event. Does nothing if the socket was not created. |
| `off(event, callback)` | Stop listening. Without `callback`, removes all listeners for the event. |
| `emit(event, ...args)` | Send an event to the server. |
| `publish(event, message)` | Same as `emit` with one argument. |
| `doctype_subscribe(doctype)` / `doctype_unsubscribe(doctype)` | Join or leave `doctype:<doctype>`. |
| `doc_subscribe(doctype, name)` / `doc_unsubscribe(doctype, name)` | Join or leave `doc:<doctype>/<name>`. Throttled to one call each second. |
| `task_subscribe(task_id)` / `task_unsubscribe(task_id)` | Join or leave `task_progress:<task_id>`. |

The client tries to reconnect 3 times. If `disable_async` is set in site config, the client does not connect.

The Node server accepts only the subscribe events in the table above, plus `ping` and `open_in_editor`. An app cannot add its own server-side handlers in v15. To reach a group of users, publish to a user, document or DocType room from Python.

## Configuration keys

| Key | File | Default | Description |
|---|---|---|---|
| `socketio_port` | `common_site_config.json` | 9000 | Port of the Socket.IO server. Also sent to the desk in boot. |
| `socketio_uds` | `common_site_config.json` | none | Listen on this Unix socket instead of the port. |
| `redis_queue` | `common_site_config.json` | none | Redis that carries the `events` channel. |
| `webserver_host`, `webserver_port` | `common_site_config.json` | none | Where the Socket.IO server sends permission checks. Without them, it uses the request origin. |
| `disable_async` | site | unset | Turn off the realtime client in the desk. |

Environment variables `FRAPPE_SOCKETIO_PORT`, `FRAPPE_SOCKETIO_UDS`, `FRAPPE_REDIS_QUEUE` and `FRAPPE_BENCH_ROOT` override the files.

## Pitfalls

- Events are not stored. A browser that is not connected when you publish misses the event.
- A user sees events in `user:<user>` and `all` without a permission check. Do not put data there that the user must not read.
- `publish_realtime` does nothing if Redis is down. It does not raise.
- In a background job, publish after your writes commit, or use `after_commit=True`.

## Related

- [Background Jobs](background-jobs.md)
- [Form Scripts](../desk-ui/form-scripts.md)
- [Client API](../desk-ui/client-api.md)
- [Deployment](../operations/deployment.md)
