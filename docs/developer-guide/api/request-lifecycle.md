---
title: Request Lifecycle
description: Follow one HTTP request through frappe.app.application, from site resolution, session and auth to dispatch, commit, response, cleanup and errors.
version: develop
sources:
  - frappe/app.py
  - frappe/__init__.py
  - frappe/auth.py
  - frappe/sessions.py
  - frappe/translate.py
  - frappe/config.py
  - frappe/hooks.py
  - frappe/rate_limiter.py
  - frappe/recorder.py
  - frappe/monitor.py
  - frappe/middlewares.py
  - frappe/api/__init__.py
  - frappe/handler.py
  - frappe/website/serve.py
  - frappe/storage/serve.py
  - frappe/utils/response.py
  - frappe/utils/error.py
  - frappe/utils/__init__.py
  - frappe/utils/data.py
  - frappe/utils/bench_helper.py
  - frappe/utils/background_jobs.py
  - frappe/utils/scheduler.py
  - frappe/database/database.py
  - frappe/database/utils.py
  - frappe/realtime/__init__.py
  - frappe/realtime/auth.py
  - frappe/permissions.py
  - frappe/exceptions.py
  - frappe/boot.py
  - frappe/commands/utils.py
  - frappe/commands/execute.py
  - realtime/middlewares/authenticate.js
---

# Request Lifecycle

This page follows one HTTP request through the code of `frappe/app.py`, step by step. Use it to debug a request, or to find the right place to change how your app handles requests. For how to call the API, see [REST API](rest-api.md) and [Whitelisted methods](whitelisted-methods.md). For the short overview, see [Architecture](../../getting-started/architecture.md).

## Core concepts

- **WSGI entry point**: `frappe.app.application`. Gunicorn calls it in production. `bench serve` calls it through the Werkzeug development server.
- **`frappe.local`**: state for one request, local to the thread. `frappe.init()` resets it at the start of each request. `frappe.destroy()` releases it at the end.
- **Transaction**: one database transaction for each request. `sync_database()` commits it or rolls it back after the handler returns.
- **Hooks**: `before_request`, `auth_hooks` and `after_request` let an app run code at fixed points. See [Hooks](../building-apps/hooks.md).

## The request, step by step

### 1. WSGI wrappers

`application` is wrapped twice in `frappe/app.py`:

1. `@Request.application` turns the WSGI call into a `werkzeug.Request` object.
2. `@after_response_wrapper` returns a `ClosingIterator`. It runs `run_after_response_callbacks()` and then `frappe.destroy()` after the server has sent the response body.

`bench serve` adds more middleware in `frappe.app.serve()`:

| Middleware | Condition | What it does |
|---|---|---|
| `SharedDataMiddleware` | Unless `NO_STATICS` is set | Serves `/assets/*` from `sites/assets`. |
| `StaticDataMiddleware` (`frappe/middlewares.py`) | Unless `NO_STATICS` is set | Serves `/files/*` from `sites/<site>/public/files`. Files that are not on disk go on to Frappe. |
| `ProxyFix` | `--proxy` or `USE_PROXY` | Reads `X-Forwarded-*` headers. |
| `ProfilerMiddleware` | `--profile` or `USE_PROFILER` | Prints a profile for each request. |

In production, nginx serves `/assets` and the files that exist in `/files`. When `FRAPPE_SENTRY_DSN` and `SENTRY_TRACING_SAMPLE_RATE` are set, `SentryWsgiMiddleware` also wraps the application.

### 2. Site resolution

`init_request()` picks the site name in this order:

1. The site given to `bench serve`. `bench serve` uses `--site`, then `FRAPPE_SITE`, then `default_site` from `common_site_config.json` (`get_sites()` in `frappe/utils/bench_helper.py`).
2. The `X-Frappe-Site-Name` request header.
3. The `Host` header without the port (`get_site_name(get_request_host(request))`).

`currentsite.txt` is not used. The CLI only prints a warning about it.

```bash
curl -s http://127.0.0.1:8000/api/method/ping -H 'X-Frappe-Site-Name: frappe.localhost'
# {"message":"pong"}

curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8000/api/method/ping
# 404   (body: "127.0.0.1 does not exist.")
```

If the site folder has no `site_config.json`, `get_site_config()` in `frappe/config.py` raises `IncorrectSitePath` and the response is HTTP 404. When `developer_mode` is set in `common_site_config.json`, that 404 page also lists all sites on the bench.

### 3. `frappe.init` and `frappe.connect`

1. `frappe.init(site, force=True, is_request=True)` releases the old `frappe.local` and fills it again. It loads the merged site config (cached for requests) into `frappe.local.conf`.
2. `init_request()` sets `frappe.local.request` and adds `request.after_response`, a `CallbackManager`.
3. `frappe.local.is_ajax` is `True` when `X-Requested-With` is `XMLHttpRequest`.
4. `frappe.connect(set_admin_as_user=False)` opens `frappe.local.db`. The user is not set yet.
5. If `maintenance_mode` is set in the site config, and `allow_reads_during_maintenance` is not false, `setup_read_only_mode()` sets `frappe.flags.read_only`. It connects to the read replica (`read_from_replica`) or starts a read-only transaction. If `allow_reads_during_maintenance` is false, the request fails with `frappe.SessionStopped` (HTTP 503).

These keys of `frappe.local` are set for each request:

| Key | Set by | Value |
|---|---|---|
| `site`, `site_path`, `conf` | `frappe.init` | Site name, site folder and merged config |
| `flags` | `frappe.init` | `_dict` with defaults such as `read_only`, `mute_messages`, `in_import`, `ignore_links` |
| `response` | `frappe.init` | `_dict({"docs": []})`. Becomes the JSON body for API calls. |
| `response_headers` | `frappe.init` | `Headers()`. Added to the response in `process_response()`. |
| `message_log`, `error_log`, `debug_log` | `frappe.init` | Messages from `frappe.msgprint`, `frappe.errprint` and debug output |
| `lang` | `frappe.init`, then `HTTPRequest.set_lang` | `conf.lang` or `en`, then the request language |
| `session` | `frappe.init`, then `Session` | `_dict(user="Guest", data=_dict())`, then the loaded session |
| `db` | `frappe.connect` | Database connection |
| `request` | `init_request` | The `werkzeug.Request` (also `frappe.request`) |
| `is_ajax` | `init_request` | `True` for `X-Requested-With: XMLHttpRequest` |
| `form_dict` | `make_form_dict` | Request parameters (`frappe.form_dict`) |
| `http_request` | `init_request` | `frappe.auth.HTTPRequest` instance (not for `OPTIONS`) |
| `request_ip` | `HTTPRequest.set_request_ip` | Client IP address |
| `cookie_manager` | `HTTPRequest.set_cookies` | `CookieManager` for cookies to set or delete |
| `login_manager` | `HTTPRequest.set_session` | `LoginManager` |
| `session_obj` | `LoginManager.make_session` | `frappe.sessions.Session` |
| `rate_limiter`, `monitor`, `_recorder` | `before_request` hooks | Only when the feature is on |

### 4. Body size and `frappe.form_dict`

`init_request()` sets `request.max_content_length` first:

- Upload paths (`/api/method/upload_file`, `/api/v1/method/upload_file`, `/api/v2/method/upload_file`): `get_max_file_size()`.
- A `PUT` to a path in the `streaming_request_paths` hook: no limit. The body is not read. `form_dict` gets only the query string.
- All other requests: `max_file_size` from the site config, or 25 MB.

Then `make_form_dict()` builds `frappe.form_dict`:

| Body | `frappe.form_dict` |
|---|---|
| JSON object (`Content-Type: application/json`) | The object. The query string is ignored. |
| JSON array | `{"data": [...]}` |
| Invalid JSON | `frappe.DataError` "Invalid request body" (HTTP 417) |
| Form (`application/x-www-form-urlencoded`) or multipart | Query string values, then form values on top. Files stay in `frappe.request.files`. |
| No body | The query string |

The key `_` (a jQuery cache buster) is always removed.

```bash
# The JSON body replaces the query string, so doctype is lost
curl -s -b cookies.txt -X GET "http://frappe.localhost:8000/api/method/frappe.client.get_count?doctype=ToDo" \
  -H 'Content-Type: application/json' -d '{"filters":{}}'
# HTTP 500: {"exception": "TypeError: get_count() missing 1 required positional argument: 'doctype'", ...}

# A form body is merged with the query string
curl -s -b cookies.txt -X GET "http://frappe.localhost:8000/api/method/frappe.client.get_count?doctype=ToDo" \
  -d 'filters={}'
# {"message":5}
```

### 5. Session and user: `frappe.auth.HTTPRequest`

For every method except `OPTIONS`, `init_request()` creates `HTTPRequest()`. It runs these steps in order:

1. `set_request_ip()`: the remote address under `bench serve`. Otherwise the first `X-Forwarded-For` value, then `REMOTE_ADDR`, then `127.0.0.1`.
2. `set_cookies()`: creates `frappe.local.cookie_manager`.
3. `set_session()`: creates `LoginManager()`.
    - For `/api/method/login`, it runs the login. See [Authentication](../security/authentication.md).
    - For other paths, `Session(resume=True)` reads the `sid` from `form_dict` or the `sid` cookie. It loads the session from the Redis cache, then from the `Sessions` table.
    - If the session is missing or expired, the user is `Guest`. The response gets `session_expired: 1` and the login cookies are deleted.
4. `set_lang()`: `get_language()` in `frappe/translate.py`. The order is `_lang` in the form, the user language (logged-in users), the `preferred_language` cookie, `Accept-Language`, then System Settings.
5. `validate_csrf_token()`. See [CSRF](#6-csrf-check).
6. `cookie_manager.init_cookies()`: queues the `sid` cookie.

A Guest request still gets `sid=Guest` and the cookies `system_user`, `full_name`, `user_id` and `user_lang`.

### 6. CSRF check

`HTTPRequest.validate_csrf_token()` raises `frappe.CSRFTokenError` (HTTP 400) only when all of these are true:

- The method is `POST`, `PUT`, `DELETE` or `PATCH`.
- `ignore_csrf` is not set in the site config.
- The session has a saved `csrf_token`. A session gets one when the desk or a website page calls `frappe.sessions.get_csrf_token()`.
- The `X-Frappe-CSRF-Token` header (or the `csrf_token` form value) does not match.
- `Referer` or `Origin` is not in `allowed_referrers` of the site config.

The check runs before `validate_auth()`, so it uses the cookie session. A token request has a Guest session with no saved token, so it is not checked.

```bash
# Cookie session after /desk was loaded once
curl -s -b cookies.txt -X POST http://frappe.localhost:8000/api/method/frappe.auth.get_logged_user
# HTTP 400: {"exception": "frappe.exceptions.CSRFTokenError: Invalid Request", "exc_type": "CSRFTokenError", ...}

curl -s -b cookies.txt -X POST http://frappe.localhost:8000/api/method/frappe.auth.get_logged_user \
  -H "X-Frappe-CSRF-Token: <csrf_token>"
# {"message":"Administrator"}
```

### 7. `before_request` hooks

`init_request()` then calls each function in the `before_request` hook with `frappe.call()`. Frappe registers these in `frappe/hooks.py`:

| Function | What it does |
|---|---|
| `frappe.recorder.record` | Starts the Recorder if it is on (Recorder in the desk, `frappe.recorder.start`). |
| `frappe.monitor.start` | Creates `frappe.local.monitor` if `monitor` is set in the site config. |
| `frappe.rate_limiter.apply` | Applies the site rate limit (`rate_limit` in the site config). |
| `frappe.integrations.oauth2.set_cors_for_privileged_requests` | Sets CORS for some OAuth and `/.well-known/` requests. |

App hooks run after these, in the order of installed apps. The user is the cookie session user here. Token and bearer authentication has not run yet. An exception in a `before_request` hook stops the request and goes to [error handling](#13-errors).

### 8. Rate limiting

`frappe.rate_limiter.apply()` reads `rate_limit = {"limit": <seconds>, "window": <seconds>}` from the site config. It counts the total request time of the site in microseconds, in a Redis key for each window. The counter is for the whole site, not for each user or IP. If the counter is above the limit, the request fails with `frappe.TooManyRequestsError` (HTTP 429).

- `process_response()` adds `X-RateLimit-Reset`, `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `Retry-After` for rejected requests.
- `frappe.rate_limiter.update` runs after the response and adds the time of this request to the counter.

The `@rate_limit` decorator and the `rate_limit` option of `@frappe.whitelist` are separate. They limit one method. See [Whitelisted methods](whitelisted-methods.md).

### 9. Token, bearer and auth hooks: `validate_auth()`

`application()` calls `validate_auth()` in `frappe/auth.py` after `init_request()`:

1. It splits the `Authorization` header on spaces.
2. With exactly two parts, `validate_oauth()` handles `Bearer <token>`.
3. With exactly two parts, `validate_auth_via_api_keys()` handles `token <key>:<secret>` and `Basic <base64>`. OAuth client credentials on the token, revoke and introspect endpoints are skipped here.
4. `validate_auth_via_hooks()` calls each function in `auth_hooks`.
5. With a two-part header, if the user is still `Guest`, Frappe raises `frappe.AuthenticationError` (HTTP 401).
6. If the user changed, Frappe checks the `restrict_ip` of the user and reads `X-Frappe-Agent`.

A token or bearer user is set with `frappe.set_user()`. No session row is made and no `sid` cookie is set for that user.

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://frappe.localhost:8000/api/method/frappe.auth.get_logged_user \
  -H 'Authorization: token 1a2b3c4d5e6f7a8:9f8e7d6c5b4a3f2'
# 401   (unknown key)

curl -s -o /dev/null -w "%{http_code}\n" http://frappe.localhost:8000/api/method/frappe.auth.get_logged_user \
  -H 'Authorization: token a b'
# 403   (three parts: the header is ignored and the request runs as Guest)
```

### 10. Dispatch by path

`application()` picks the handler in this order:

| Condition | Handler |
|---|---|
| `OPTIONS` | Empty `Response()` (CORS preflight). No session and no cookies. |
| `frappe.form_dict.cmd` is set | `frappe.handler.handle()`, then `build_response("json")`. Deprecated. |
| Path starts with `/api/` | `frappe.api.handle(request)`. See [API internals](api-internals.md). |
| Path starts with `/backups` | `frappe.utils.response.download_backup()`. Needs `System Manager`. |
| Path starts with `/private/files/` | `frappe.storage.serve.serve_private_file()`. Checks file permission. |
| Path starts with `/files/` | `frappe.storage.serve.serve_public_file()`. Only for files that nginx or `StaticDataMiddleware` did not find on disk. |
| `GET /.well-known/security.txt` | Text from `Security Settings`. HTTP 404 if the scheme is not `https`. |
| `GET /.well-known/*` | `handle_wellknown()` (OAuth metadata) |
| Other `GET`, `HEAD` or `POST` | `frappe.website.serve.get_response()`. Desk (`/desk`), `/app` redirects, `www/` pages, web forms and other website routes. |
| Anything else | `NotFound` (HTTP 404) |

`/assets/*` never reaches `application()`. nginx or `SharedDataMiddleware` serves it. For the website route order, see [Website](../website/website.md).

If the System Settings field `log_api_requests` is set, `frappe.api.handle()` also queues an `API Request Log` with `deferred_insert()`.

### 11. Commit or rollback

If the handler returns without an exception, `sync_database()` runs:

- For `POST`, `PUT`, `DELETE` and `PATCH` (`UNSAFE_HTTP_METHODS`), it commits.
- For other methods (`GET`, `HEAD`, `OPTIONS`, `QUERY`), it rolls back. If code set `frappe.local.flags.commit = True`, it commits. For example, `frappe/boot.py` sets it when the desk creates missing Notification Settings.
- It queues `session.update()` on `frappe.request.after_response`.

If the handler raises, `application()` calls `frappe.db.rollback(chain=True)` after it builds the error response.

Other rules:

- `frappe.db.commit()` in your code commits at once, for any HTTP method.
- `frappe.db.after_commit.add(fn)` runs `fn` after the next commit. A rollback drops it. `frappe.db.after_rollback.add(fn)` works the other way. A failing callback is written to the Error Log and does not change the response.
- `frappe.publish_realtime(..., after_commit=True)` queues the event on `after_commit`. A rollback drops the event.
- `frappe.enqueue(..., enqueue_after_commit=True)` enqueues the job only after the commit.
- In read-only mode (maintenance, or `frappe.db.read_only()`), a commit is a rollback.

See [Database API](../database/database.md) for transaction details.

```python
# sample_app/api.py
import frappe


@frappe.whitelist(methods=["POST"])
def close_item(name: str):
	frappe.db.set_value("Sample Item", name, "disabled", 1)
	# runs only if the request commits
	frappe.db.after_commit.add(lambda: frappe.publish_realtime("sample_item_closed", {"name": name}))
```

### 12. `after_request` hooks and the response

1. `run_after_request_hooks()` runs in a `finally` block, for success and error. It calls each `after_request` function with `frappe.call(fn, response=response, request=request)`. A function gets only the arguments it declares. An exception is logged and ignored. Frappe registers `frappe.monitor.stop` and `frappe.flow.launcher.add_to_page`.
2. `log_request()` writes one line to the `frappe.web` logger if `enable_frappe_logger` is set in the site config.
3. `process_response()` changes the response:
    - Sets `Cache-Control: no-store,no-cache,must-revalidate,max-age=0`, unless the handler set `Cache-Control`.
    - Adds the rate limit headers.
    - Adds `X-Frappe-Request-Id` when the monitor is on.
    - Adds CORS headers from `allow_cors` in the site config, or from `frappe.local.allow_cors`, if the `Origin` matches.
    - Adds `WWW-Authenticate` to 401 and 403 responses when OAuth resource metadata is on.
    - Adds the headers in `frappe.local.response_headers`.
    - Sets the queued cookies, unless `Cache-Control` contains `public`.

The `after_request` hook gets the final `werkzeug` `Response`. It can change headers or the body.

### 13. Errors

When an exception escapes the handler, `application()` makes the response:

- A `werkzeug.exceptions.HTTPException` (for example `NotFound`, `Forbidden`) uses its own response.
- Other exceptions go to `handle_exception()` in `frappe/app.py`.

`handle_exception()` works like this:

1. The status is `http_status_code` of the exception class, or 500. See `frappe/exceptions.py` and [REST API errors](rest-api.md#errors).
2. A `frappe.DoesNotExistError` for a DocType that the user cannot access becomes a `frappe.PermissionError` (HTTP 403). This hides which documents exist (`handle_does_not_exist_error` in `frappe/permissions.py`).
3. The response is JSON when the request has an `Accept` header and is AJAX or accepts `application/json`. It is also JSON for `/api/` paths, unless `Accept` starts with `text`. Then `report_error()` builds the body. See [API internals](api-internals.md).
4. Other requests get an HTML error page (`ErrorPage`) for 401, 403, 404 and 500. `frappe.SessionStopped` gets the maintenance page. 429 gets "Too Many Requests".
5. `frappe.AuthenticationError` also deletes the login cookies.
6. If the status is 500 or more, or `developer_mode` is on, `log_error_snapshot()` writes an Error Log (with `defer_insert`). Exceptions with `skip_error_log = True` (`AuthenticationError`, `CSRFTokenError`) are not logged.
7. In `developer_mode`, a non-JSON error prints the traceback to the console.

```bash
curl -s -o /dev/null -w "%{http_code} %{content_type}\n" -H 'Accept: text/html' \
  http://frappe.localhost:8000/api/method/frappe.auth.get_logged_user
# 403 text/html; charset=utf-8

curl -s http://frappe.localhost:8000/api/v2/nothing
# HTTP 404: {"errors":[{"type":"DoesNotExistError"}]}
```

The traceback goes into the JSON body (`exc` in v1, `exception` in v2) only if `is_traceback_allowed()` in `frappe/utils/response.py` is true:

- The site runs under `bench serve` (`frappe._dev_server`), for any user, Guest too.
- Or System Settings `allow_error_traceback` is on, the user is a System User, and `frappe.flags.disable_traceback` is not set.

`developer_mode` alone does not show tracebacks to clients. For 404 errors, the traceback is sent only when `logging` is set in the site config.

### 14. After the response and cleanup

After the server sends the body, `ClosingIterator` runs these in order:

1. `frappe.rate_limiter.update`: adds the request time to the rate limit counter.
2. `frappe.recorder.dump`: saves the recorded request if the Recorder is on.
3. Each function in `frappe.request.after_response`, in the order it was added. This includes `session.update()` from `sync_database()` and the queue of `frappe.db.commit_after_response`.
4. `frappe.destroy()`: closes `frappe.local.db` and releases `frappe.local`.

A failing callback is logged to the `after_response` logger. It cannot change the response, which is already sent.

```python
# sample_app/api.py
import frappe


@frappe.whitelist()
def get_report():
	# do slow work after the client has the response
	frappe.request.after_response.add(lambda: frappe.logger("sample_app").info("report sent"))
	return {"ok": True}
```

`commit_after_response(fn)` in `frappe/database/utils.py` runs `fn` after the response in a savepoint and then commits. Outside a request, or in tests, it runs `fn` at once.

## Request logging, Recorder and Monitor

| Tool | Turn on | Output |
|---|---|---|
| Request log | `enable_frappe_logger: 1` in the site config | One line for each request in the `frappe.web` logger: site, IP, user, path, method, status |
| API Request Log | System Settings `log_api_requests` | One `API Request Log` document for each `/api/` request |
| Recorder | Recorder in the desk (`frappe.recorder.start`). It turns off after 10 minutes. | SQL queries and timing of each request, in the desk |
| Monitor | `monitor: 1` in the site config | JSON for each request in Redis, written to `logs/monitor.json.log` by the scheduled job `frappe.monitor.flush` |

Send `X-Frappe-Request-Id` with a request to use your own trace ID in the monitor. It must match `TRACE_ID_PATTERN` in `frappe/monitor.py`.

## Other execution contexts

Code also runs outside `frappe.app.application`. These contexts have no `frappe.request`:

| Context | Start | User | Commit | Hooks |
|---|---|---|---|---|
| Background job | `frappe.utils.background_jobs.execute_job` | The user who called `frappe.enqueue` | Commits on success. On error, rolls back, writes the Error Log and commits it. Retries deadlocks and timeouts up to 5 times. | `before_job`, `after_job` |
| Scheduled job | The scheduler enqueues `Scheduled Job Type` jobs. Workers run them as background jobs. | `Administrator` | As background job | `before_job`, `after_job` |
| `bench execute` | `frappe/commands/execute.py` | `Administrator` | Commits after the call | None |
| `bench console` | `frappe/commands/utils.py` | `Administrator` | No auto commit. Rolls back on exit. Call `frappe.db.commit()`. | None |
| Socket.IO server | `bench socketio` | The user of the `sid` cookie or `Authorization` header | None | None |

Other differences:

- `is_valid_http_method()` in `frappe/handler.py` skips the HTTP method check in a background job.
- `frappe.form_dict` is empty unless the code fills it.
- The Socket.IO server does not run Frappe code for the socket. It calls `frappe.realtime.get_user_info` and `frappe.realtime.has_permission` over HTTP on the web server, with `X-Frappe-Site-Name`. These are normal requests.

## Pitfalls

- A JSON body replaces the query string. Send all parameters in the body, or none.
- A missing required argument raises `TypeError`, which is HTTP 500, not 417.
- A `GET` handler that writes data loses the writes, unless it calls `frappe.db.commit()` or sets `frappe.local.flags.commit`.
- An `Authorization` header that does not split into exactly two parts is ignored. The request runs as `Guest`.
- `before_request` hooks run before token authentication. Do not read `frappe.session.user` there for token requests.
- Under `bench serve`, Guests also get tracebacks. Do not judge production error output from a development server.
- The site rate limit is shared by all clients of the site.

## Related

- [Architecture](../../getting-started/architecture.md)
- [API internals](api-internals.md)
- [Customizing APIs](customizing-apis.md)
- [REST API](rest-api.md)
- [Whitelisted methods](whitelisted-methods.md)
- [Authentication](../security/authentication.md)
- [Hooks](../building-apps/hooks.md)
- [Database API](../database/database.md)
- [Background jobs](../runtime/background-jobs.md)
- [Realtime](../runtime/realtime.md)
- [Configuration](../operations/configuration.md)
- [REST endpoints reference](../../reference/http/rest-endpoints.md)
