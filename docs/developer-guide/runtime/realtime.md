---
title: Realtime
description: Push events from the server to browsers with publish_realtime, Socket.IO rooms and frappe.realtime.on, and add custom realtime handlers.
version: v16
sources:
  - frappe/realtime/__init__.py
  - frappe/realtime/README.md
  - frappe/realtime/handlers.py
  - frappe/realtime/config.py
  - frappe/realtime/server.py
  - frappe/public/js/frappe/socketio_client.js
  - frappe/model/document.py
  - frappe/commands/site.py
  - frappe/boot.py
---

# Realtime

Use realtime events to push data from the server to open browser tabs: a progress bar, a refresh after a background job, a message to one user. Server code calls `frappe.publish_realtime`. Frappe publishes the event on the Redis channel `events`. The Socket.IO server (`bench socketio`) reads the channel and sends the event to the browsers in the target room. Browser code listens with `frappe.realtime.on`.

## Concepts

- **Event**: a name, such as `doc_update` or `sample_order_ready`, and a JSON message.
- **Room**: a group of sockets on one site. The server sends each event to one room.
- **Socket.IO server**: a separate process. It listens on `socketio_port` (default 9000). There are two backends: the Node server `apps/frappe/socketio.js` and the Python server `python -m frappe.realtime.server`. The `bench socketio` command picks one from the `socketio_backend` key in `common_site_config.json` (`node` by default, or `python`).

### Rooms

| Room | Who is in it | How a socket joins |
|---|---|---|
| `user:<user>` | All sockets of that user on the site | On connect |
| `website` | All sockets on the site, guests too | On connect |
| `all` | Sockets of System Users on the site | On connect |
| `doctype:<doctype>` | Sockets that show a list of that DocType | `doctype_subscribe`, needs read permission |
| `doc:<doctype>/<name>` | Sockets that have the form open | `doc_subscribe`, needs read permission |
| `open_doc:<doctype>/<name>` | Sockets that view the form now. Used for `doc_viewers`. | `doc_open` |
| `task_progress:<task_id>` | Sockets that follow a task | `task_subscribe` |

The desk subscribes for you. A form subscribes to its document on load. A list view subscribes to its DocType.

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

### Named helpers

`frappe.realtime` has one helper for each room. Each calls `publish_realtime` and takes `after_commit` as a keyword-only argument.

```python
from frappe.realtime import (
	publish_task_progress,
	publish_to_all,
	publish_to_doc,
	publish_to_doctype,
	publish_to_room,
	publish_to_user,
	publish_to_website,
)

publish_to_doc("Sample Order", "SO-0001", "sample_order_ready", {"status": "Ready"}, after_commit=True)
publish_to_doctype("Sample Order", "sample_orders_changed")
publish_to_room("doc:Sample Order/SO-0001", "sample_order_ready")
```

`publish_task_progress(task_id, message)` has no event argument. It sends `task_progress`.

### Progress bars

`frappe.publish_progress` sends the `progress` event. The desk shows a progress dialog.

```python
def import_items(rows):
	for i, row in enumerate(rows, start=1):
		...
		frappe.publish_progress(i * 100 / len(rows), title="Importing items", description=f"Row {i}")
```

Signature: `publish_progress(percent, title=None, doctype=None, docname=None, description=None, task_id=None)`. With `doctype` and `docname`, everyone with the form open sees the progress. Without them, only the current user sees it.

Call it from a [background job](background-jobs.md) to show the progress of long work.

### Events the framework sends

| Event | Room | When | Message |
|---|---|---|---|
| `doc_update` | `doc:<doctype>/<name>` | After a document is saved (`notify_update`), after commit | `doctype`, `name`, `modified` |
| `list_update` | `doctype:<doctype>` | Same, for DocTypes that are not single, child or read-only | `doctype`, `name`, `user` |
| `docinfo_update` | `doc:<doctype>/<name>` | A Comment or Communication on the document changes | varies |
| `doc_viewers` | `open_doc:<doctype>/<name>` | A user opens or closes the form | `doctype`, `docname`, `users` |
| `msgprint` | `user:<user>` | `frappe.msgprint(..., realtime=True)` | message |
| `progress` | user or doc room | `frappe.publish_progress` | `percent`, `title`, `description` |

`notify_update` does not send events during import, patch, migrate and install. `doc.db_set(..., notify=True)` also calls it.

### From the CLI

```bash
bench --site frappe.localhost publish-realtime sample_event --message '{"status": "Open"}' --user Administrator
```

Options: `--message`, `--room`, `--user`, `--doctype`, `--docname`, `--after-commit`. The message is sent as a string.

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
| `on(event, callback)` | Listen for an event. Connects the socket if needed. |
| `off(event, callback)` | Stop listening. Without `callback`, removes all listeners for the event. |
| `emit(event, ...args)` | Send an event to the server. |
| `doctype_subscribe(doctype)` / `doctype_unsubscribe(doctype)` | Join or leave `doctype:<doctype>`. |
| `doc_subscribe(doctype, name)` / `doc_unsubscribe(doctype, name)` | Join or leave `doc:<doctype>/<name>`. |
| `task_subscribe(task_id)` / `task_unsubscribe(task_id)` | Join or leave `task_progress:<task_id>`. |

If `disable_async` is set in site config, the client does not connect.

!!! note
    In v16 the client does not join its DocType and document rooms again after a reconnect. A list or form that was open during a Socket.IO restart can miss `list_update` and `doc_update` events. Reload the page to subscribe again.

## Custom handlers (Python backend)

With the Python backend, an app can handle events that the browser emits. Put the handlers in `<app>/realtime/handlers.py`. The server imports this module for each installed app at startup. A handler runs only for sockets on sites where its app is installed.

```python
# sample_app/sample_app/realtime/handlers.py
from frappe.realtime import Socket, realtime


@realtime.on("sample_order_subscribe")
async def sample_order_subscribe(socket: Socket, order: str) -> dict:
	allowed = await socket.has_permission("Sample Order", order)
	if allowed:
		await socket.join(f"sample_order:{order}")
	return {"subscribed": allowed}
```

```js
frappe.realtime.emit("sample_order_subscribe", "SO-0001");
```

Then publish to the same room from server code: `publish_to_room("sample_order:SO-0001", "sample_order_ready", {...})`.

Rules:

- The first argument is the `Socket`. The other arguments are what the client sent.
- The return value answers the client's acknowledgement callback.
- `@realtime.on(event, frappe_context=False, allow_guest=False)`. With `allow_guest=False`, events from guests are dropped.
- Write handlers as `async def`. A plain `def` handler runs in a worker thread and gets a blocking socket with the same methods without `await`.
- `frappe_context=True` opens a site context with a database connection for each event. It needs a plain `def` handler. Use it only when you must query the database.
- An import error in `handlers.py` stops the server at startup.

`Socket` attributes: `site`, `user`, `user_type`, `installed_apps`. Methods: `join(room)`, `leave(room)`, `emit(event, data=None, room=None)`, `has_permission(doctype, name=None)`, `set(key, value)`, `get(key, default=None)`. `has_permission` calls the web process over HTTP.

## Configuration keys

All keys are in `common_site_config.json`.

| Key | Default | Description |
|---|---|---|
| `socketio_port` | 9000 | Port of the Socket.IO server. |
| `socketio_backend` | `node` | `node` or `python`. Read by `bench socketio`. |
| `socketio_uds` | none | Listen on this Unix socket instead of the port. |
| `socketio_worker_threads` | 4 | Threads for blocking work in the Python server. |
| `redis_queue` | `redis://127.0.0.1:11311` | Redis that carries the `events` channel. |

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
