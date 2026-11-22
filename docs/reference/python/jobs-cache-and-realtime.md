---
title: Jobs, Cache and Realtime API
description: Signatures and behavior of frappe.enqueue, background job helpers, the scheduler, frappe.cache, caching decorators and frappe.publish_realtime.
version: develop
sources:
  - frappe/utils/background_jobs.py
  - frappe/utils/task_queue.py
  - frappe/core/doctype/background_task/background_task.py
  - frappe/utils/scheduler.py
  - frappe/utils/redis_wrapper.py
  - frappe/utils/caching.py
  - frappe/realtime/__init__.py
  - frappe/realtime/registry.py
  - frappe/realtime/socket.py
  - frappe/__init__.py
---

# Jobs, Cache and Realtime API

This page lists the public functions of the background job, scheduler, Redis cache and realtime modules. Use it to look up a signature, a default or a side effect. For how-to guides, read [Background Jobs](../../developer-guide/runtime/background-jobs.md), [Scheduler](../../developer-guide/runtime/scheduler.md), [Caching](../../developer-guide/runtime/caching.md) and [Realtime](../../developer-guide/runtime/realtime.md).

**Whitelisted** means the function is callable at `/api/method/<dotted.path>`. "any" means the default set: `GET`, `POST`, `PUT`, `DELETE`, `QUERY`.

## frappe.enqueue

`frappe.enqueue` is `frappe.utils.background_jobs.enqueue`.

```python
frappe.enqueue(
	method: str | Callable, queue: str = "default", timeout: int | None = None, event: str | None = None,
	is_async: bool = True, job_name: str | None = None, now: bool = False, enqueue_after_commit: bool = False,
	*, on_success: Callable | None = None, on_failure: Callable | None = None, at_front: bool = False,
	job_id: str | None = None, deduplicate=False, at_front_when_starved=False, rq_retry: Retry | None = None,
	**kwargs,
) -> Job | Any
```

| Parameter | Default | Description |
|---|---|---|
| `method` | | Dotted path or function. The worker calls it with `**kwargs`. |
| `queue` | `"default"` | `short`, `default`, `long`, or a queue in the `workers` key of `common_site_config.json`. Other names raise `ValidationError`. |
| `timeout` | `None` | Seconds. `None` uses the queue timeout: 300 for `short` and `default`, 1500 for `long`. |
| `event` | `None` | Stored on the job. Used by the scheduler. |
| `is_async` | `True` | `False` in tests runs the job in the same process through RQ. `False` outside tests calls the method directly and shows a deprecation warning. Use `now`. |
| `job_name` | `None` | Deprecated. Use `job_id`. |
| `now` | `False` | Call the method directly with `frappe.call` and return its result. |
| `enqueue_after_commit` | `False` | Add the job only after the current transaction commits. Returns `None`. |
| `on_success`, `on_failure` | `None` | RQ callbacks. `on_failure` defaults to `truncate_failed_registry`. |
| `at_front` | `False` | Put the job at the front of the queue. |
| `job_id` | `None` | Your ID. Stored as `<site>\|\|<job_id>`, with `:` replaced by `\|`. |
| `deduplicate` | `False` | Needs `job_id`. Returns `None` if a job with that ID is queued or started. A finished job with the ID is deleted first. |
| `at_front_when_starved` | `False` | Use `at_front` when the queue has more than 16 jobs. |
| `rq_retry` | `None` | `rq.Retry` object to retry a failed job. |
| `**kwargs` | | Arguments for `method`. `async` is read as `is_async`. |

Side effects and behavior:

- The job runs as the user who enqueued it, on the same site.
- The worker commits after the method returns. On an exception it rolls back, writes an Error Log, and commits the log.
- On a deadlock, lock timeout or `frappe.RetryBackgroundJobError`, the worker retries up to 5 times.
- `before_job` and `after_job` hooks run around the method. `frappe.local.job` holds `site`, `method`, `job_name`, `kwargs`, `user` and `after_job`.
- If the queue has `max_queued_jobs` jobs (default 500, plus 50 for each site), it raises `frappe.QueueOverloaded`.
- If Redis is down during `bench migrate`, the method runs directly.

```python
# sample_app/sample_app/api.py
import frappe


@frappe.whitelist(methods=["POST"])
def rebuild_totals(order: str):
	frappe.has_permission("Sample Order", "write", order, throw=True)
	frappe.enqueue(
		"sample_app.sample_app.jobs.rebuild_totals",
		queue="long",
		job_id=f"rebuild_totals::{order}",
		deduplicate=True,
		enqueue_after_commit=True,
		order=order,
	)
```

## Other enqueue functions

| Function | Description |
|---|---|
| `frappe.enqueue_doc(doctype, name=None, method=None, queue="default", timeout=300, now=False, **kwargs)` | Enqueue `run_doc_method`, which loads the document and calls `doc.<method>(**kwargs)`. |
| `frappe.utils.background_jobs.run_doc_method(doctype, name, doc_method, **kwargs)` | `getattr(frappe.get_doc(doctype, name), doc_method)(**kwargs)`. |
| `frappe.task(**task_kwargs)` | Decorator. Adds `f.enqueue(**kwargs)`, which calls `frappe.enqueue(f, **task_kwargs, **kwargs)`. |
| `frappe.utils.background_jobs.defer_enqueue_after_commit()` | Context manager. Holds `enqueue_after_commit` jobs across commits inside the block. On a clean exit, they move to the real commit callbacks. On an exception, they are dropped. Call `.cancel()` to drop them when you handle an error inside the block. |

### frappe.enqueue_task

`frappe.enqueue_task` is `frappe.utils.task_queue.enqueue_task`. It makes a `Background Task` document that users can follow, cancel and retry in the desk.

```python
frappe.enqueue_task(
	method: str | Callable, *, task_name: str | None = None, queue: str = "default", timeout: int | None = None,
	ref_doctype: str | None = None, ref_docname: str | None = None, deduplicate: bool = False,
	job_id: str | None = None, enqueue_after_commit: bool = True, now: bool = False,
	on_success: Callable | None = None, on_failure: Callable | None = None,
	retry_on: tuple[type[Exception], ...] = (), max_retries: int = 0, show_progress_bar: bool = True,
	allow_user_cancellation: bool = True, allow_user_retry: bool = True, at_front: bool = False,
	at_front_when_starved: bool = False, **kwargs,
) -> BackgroundTask
```

- Inserts the `Background Task` (status `Queued`) with `ignore_permissions`, then enqueues. `enqueue_after_commit` is `True` by default.
- `task_name` defaults to the dotted method path. `job_id` defaults to the task ID (a UUID).
- The worker sets status `Running`, then `Completed` (with `result` as JSON) or `Failed` (with the traceback). It commits each status change and sends a `task_update` realtime event to the user.
- `retry_on` with `max_retries > 0` retries on those exceptions through RQ.
- `on_success` and `on_failure` get the keyword arguments they declare from `task`, `result` or `exception`, and the job kwargs.

| Function | Description |
|---|---|
| `frappe.get_current_task() -> BackgroundTask \| None` | Return the `Background Task` of the running job. |
| `frappe.utils.task_queue.get_task_status(task_id: str) -> dict \| None` | Return `status`, `progress`, `stage`, `result`, `exception`, `started_at`, `ended_at`, with cached progress applied. |

`BackgroundTask` methods:

| Method | Description |
|---|---|
| `publish_progress(percent: int \| float, stage: str \| None = None) -> None` | Send progress. Calls under 100 percent are throttled to one each 0.2 seconds. |
| `update_stage(stage: str) -> None` | Send a stage text without a number. |
| `store_result(result) -> None` | Save `result` as JSON with `db_set`. |
| `attach_file(file_name: str, content: bytes, is_private: bool = True, doctype=None, docname=None) -> str` | Make a File attached to the reference document (or the task). Returns its URL. |

Whitelisted endpoints in `frappe.core.doctype.background_task.background_task`: `get_recent_tasks(limit: int = 15)` (any), `get_cached_task_status(task_id: str)` (any), `stop_task(task_id: str)` (any, owner or System Manager), `retry_task(task_id: str)` (POST, owner or System Manager).

```python
# sample_app/sample_app/jobs.py
import frappe


def export_orders():
	task = frappe.get_current_task()
	names = frappe.get_all("Sample Order", pluck="name")
	for i, name in enumerate(names, 1):
		...
		task.publish_progress(i * 100 / len(names))
```

## Jobs and queues

All in `frappe.utils.background_jobs`.

| Function | Description |
|---|---|
| `get_queues_timeout() -> dict[str, int]` | `{"short": 300, "default": 300, "long": 1500, ...}` plus queues from the `workers` config key. Cached for the process (`lru_cache`). |
| `get_queue(qtype: str, is_async: bool = True) -> Queue` | Return the RQ queue `<bench_id>:<qtype>`. Validates the name. |
| `get_queue_list(queue_list=None, build_queue_name=False) -> list` | Validate queue names, or return all. With `build_queue_name`, return full RQ names. |
| `validate_queue(queue: str, default_queue_list: list \| None = None) -> None` | Raise `ValidationError` if the queue does not exist. |
| `get_queues(connection=None) -> list[Queue]` | RQ queues of this bench. |
| `generate_qname(qtype: str) -> str` | `<bench_id>:<qtype>`. |
| `is_queue_accessible(qobj: Queue) -> bool` | `True` if the queue belongs to this bench. |
| `get_jobs(site=None, queue=None, key="method") -> dict` | `{site: [value of key, ...]}` for queued and running jobs. |
| `get_job(job_id: str) -> Job \| None` | Fetch a job by your `job_id`. |
| `get_job_status(job_id: str) -> JobStatus \| None` | RQ status, or `None` if the job is gone. |
| `is_job_enqueued(job_id: str) -> bool` | `True` if the job is `queued` or `started`. |
| `create_job_id(job_id: str \| None = None) -> str` | Namespace an ID to the site. Makes a UUID when empty. Idempotent. |
| `get_queue_args(job: Job) -> dict` | The arguments `enqueue` stored, or `{}` if they no longer load. |
| `get_workers(queue=None)` | RQ workers, for one queue or all. |
| `get_running_jobs_in_queue(queue)` | Current jobs of the workers of a queue. |
| `get_redis_conn(username=None, password=None)` | Connection to `redis_queue`. Retries 5 times. Uses RQ auth when `use_rq_auth` is set. |
| `get_redis_connection_without_auth()` | Shared connection to `redis_queue` without auth. |
| `execute_job(site, method, event, job_name, kwargs, user=None, is_async=True, retry=0)` | What a worker runs for each job. Do not call it. |
| `start_worker(queue=None, quiet=False, rq_username=None, rq_password=None, burst=False, strategy=DequeueStrategy.DEFAULT)` | Start one worker. Used by `bench worker`. |
| `start_worker_pool(queue=None, num_workers=1, quiet=False, burst=False)` | Start a worker pool. Experimental. Used by `bench worker-pool`. |
| `get_worker_name(queue)` | `<uuid>.<hostname>.<pid>.<queue>` when `queue` is given, else `None` (RQ default). |
| `set_niceness()` | Raise the process niceness by `background_process_niceness` (default 10). |
| `truncate_failed_registry(job, connection, type, value, traceback)` | Keep at most `rq_failed_jobs_limit` (default 1000) failed jobs. |
| `mapreduce(map_method, reduce_method, callback_method, data, document_type, document_name, job_name)` | Insert and submit a `MapReduce Job`. |
| `cancel_mapreduce_job(document_type: str, document_name: str)` | Cancel submitted MapReduce Jobs of a document and stop their tasks. |
| `remove_mapreduce_job(document_type: str, document_name: str)` | Cancel and delete MapReduce Jobs of a document. |
| `enqueue_test_job()`, `test_job(s)` | Enqueue a job that sleeps. For testing workers. |

| Constant | Value |
|---|---|
| `RQ_JOB_FAILURE_TTL` | 7 days. Config key `rq_job_failure_ttl`. |
| `RQ_RESULTS_TTL` | 600 seconds. Config key `rq_results_ttl`. |
| `RQ_FAILED_JOBS_LIMIT` | 1000. Config key `rq_failed_jobs_limit`. |
| `MAX_QUEUED_JOBS` | 500. Config key `max_queued_jobs`. |
| `QUEUE_STARVATION_THRESHOLD` | 16 |

## Scheduler

All in `frappe.utils.scheduler`. `bench schedule` runs `start_scheduler`. Workers of `bench worker-pool` also start it in a thread. It enqueues due `Scheduled Job Type` records. See [Scheduler](../../developer-guide/runtime/scheduler.md).

| Function | Description |
|---|---|
| `start_scheduler() -> NoReturn` | Loop forever: sleep one tick, then `enqueue_events_for_all_sites()`. Holds the file lock `config/scheduler_process`. Writes a heartbeat to Redis. |
| `enqueue_events_for_all_sites() -> None` | Call `enqueue_events_for_site` for each site in random order. |
| `enqueue_events_for_site(site: str) -> None` | Connect to the site and call `enqueue_events()` unless the scheduler is inactive. |
| `enqueue_events() -> list[str] \| None` | Enqueue each due job that is not stopped and whose app is enabled. Returns the enqueued methods. |
| `is_scheduler_inactive(verbose=True) -> bool` | `True` when `maintenance_mode` or `pause_scheduler` is set, or the scheduler is disabled. |
| `is_scheduler_disabled(verbose=True) -> bool` | `True` when `disable_scheduler` is set in config, or System Settings `enable_scheduler` is off. |
| `enable_scheduler()`, `disable_scheduler()`, `toggle_scheduler(enable)` | Set System Settings `enable_scheduler`. |
| `activate_scheduler()` | Whitelisted (any). Administrator only. Enable the scheduler and clear `pause_scheduler`. Raises in maintenance mode. |
| `get_scheduler_status()` | Whitelisted (any). `{"status": "active"}` or `{"status": "inactive"}`. |
| `is_scheduler_process_running() -> bool` | `True` if the scheduler heartbeat key exists in the queue Redis. |
| `get_scheduler_tick() -> int` | `scheduler_tick_interval` from config, default 240 seconds. |
| `sleep_duration(tick)` | Seconds until the next tick. With the default tick, aligns to the clock. |
| `schedule_jobs_based_on_activity(check_time=None)` | `False` for a dormant site that ran jobs in the last 24 hours. Cached 1 hour. |
| `is_dormant(check_time=None)` | `True` on Frappe Cloud when no user was active for `dormant_days`. Always `False` in developer mode or off Frappe Cloud. |
| `cprint(*args, **kwargs)` | Print only when a terminal is attached. |

## frappe.cache

`frappe.cache` is a `frappe.utils.redis_wrapper.RedisWrapper`, a subclass of `redis.Redis` connected to `redis_cache`. Values are pickled. `frappe.cache()` (called) is deprecated.

Key rules (`make_key(key, user=None, shared=False)`):

- The key becomes `<db_name>|<key>` (bytes). So each site has its own keys.
- `user=True` adds `user:<session user>:`. A user name adds `user:<name>:`.
- `shared=True` uses the key as given. All sites see it.

Most methods also keep the value in `frappe.local.cache` for the rest of the request. Most methods ignore `redis.exceptions.ConnectionError`.

| Method | Description |
|---|---|
| `set_value(key, val, user=None, expires_in_sec=None, shared=False) -> int` | Store a value. Returns the pickled size in bytes. |
| `get_value(key, generator=None, user=None, expires=False, shared=False, *, use_local_cache=True)` | Read a value. On a miss, call `generator()` and store the result. With `expires=True`, the generator is not called and the value is not kept locally. |
| `delete_value(keys, user=None, make_keys=True, shared=False)` | Delete one key or a list. `make_keys=False` takes full keys. |
| `delete_key(*args, **kwargs)` | Same as `delete_value`. |
| `delete_keys(key, user=None, shared=False)` | Delete keys that start with `key`. |
| `get_keys(key, user=None, shared=False)` | Full keys that start with `key`. |
| `get_all(key)` | Meant to return values of keys that start with `key`. See the warning below. |
| `exists(*names, user=None, shared=None) -> int` | Count of keys that exist. |
| `expire_key(key, time, *, user=None, shared=False)` | Set a TTL in seconds. |
| `hset(name, key, value, shared=False, *args, **kwargs)` | Set a field of a hash. Does nothing when `key` is `None`. |
| `hget(name, key, generator=None, shared=False)` | Read a field. On a miss, call `generator()` and store. |
| `hgetall(name) -> dict` | All fields, unpickled. Raises on connection errors. |
| `hexists(name, key, shared=False) -> bool` | `True` if the field exists. |
| `hdel(name, keys, shared=False, pipeline=None)` | Delete one field or a list. |
| `hdel_names(names, key)` | Delete one field from many hashes in one pipeline. |
| `hdel_keys(name_starts_with, key)` | Delete one field from hashes whose names start with a prefix. |
| `hkeys(name)` | Field names. |
| `lpush(key, value, user=None, shared=False)`, `lpop(key, user=None, shared=False)`, `blpop(key, timeout=0, user=None, shared=False)` | List operations. |
| `rpush(key, value)`, `rpop(key)`, `llen(key)`, `lrange(key, start, stop)`, `ltrim(key, start, stop)` | List operations. No `user` or `shared` argument. |
| `sadd(name, *values)`, `srem(name, *values)`, `sismember(name, value)`, `spop(name)`, `smembers(name)`, `srandmember(name, count=None)` | Set operations. Values are not pickled. |
| `ft(index_name="idx")` | RediSearch client with prefixed keys (`sugadd`, `suglen`, `sugdel`, `sugget`). |
| `connected() -> bool` | `True` if Redis answers `PING`. |

!!! warning
    `get_all(key)` returns `{key: None}`, not the values. `srandmember(name, count)` ignores `count` and returns one member.

```python
frappe.cache.set_value("sample_app:rates", {"Hardware": 10}, expires_in_sec=3600)
frappe.cache.get_value("sample_app:rates", expires=True)
frappe.cache.hset("sample_app:rate_by_item", "Laptop", 1200)
frappe.cache.hget("sample_app:rate_by_item", "Laptop")  # 1200
frappe.cache.delete_keys("sample_app:")
```

Module functions: `setup_cache() -> RedisWrapper` (makes `frappe.cache`, with Sentinel when `redis_cache_sentinel_enabled` is set) and `get_sentinel_connection(sentinels, sentinel_username=None, sentinel_password=None, master_username=None, master_password=None)`.

### frappe.client_cache

`frappe.client_cache` is a `ClientCache`: process memory in front of Redis, with Redis client tracking for invalidation. It is internal to Frappe and can change without notice. Use `frappe.cache` in apps.

| Member | Description |
|---|---|
| `ClientCache(maxsize=1024, ttl=600, monitor=None, maxsize_bytes=0)` | `maxsize_bytes` defaults to the `client_cache_max_bytes` config key. |
| `get_value(key, *, shared=False, generator=None)` | Read from memory, else Redis. Misses are not cached locally. |
| `set_value(key, val, *, shared=False)` | Write to Redis and memory. |
| `delete_value(key, *, shared=False)`, `delete_keys(pattern)` | Delete. |
| `get_doc(doctype: str, name: str \| None = None)` | Cached `frappe.get_doc`. Without `name`, a Single. |
| `erase_persistent_caches(*, doctype=None)` | Tell every process to clear controller caches. Without `doctype`, also clears `@site_cache`. |
| `clear_cache()`, `statistics`, `reset_statistics()` | Local cache control and hit counts. |

## Caching decorators

All in `frappe.utils.caching`. `frappe.request_cache` and `frappe.site_cache` are aliases.

| Decorator | Description |
|---|---|
| `request_cache(func)` | Cache results in `frappe.local.request_cache` for the request or job. Unhashable arguments skip the cache. Outside a request, calls the function. |
| `site_cache(ttl: int \| None = None, maxsize: int \| None = None)` | Cache results in process memory per site. All entries clear when `ttl` passes. `maxsize` removes the oldest entry. Adds `func.clear_cache()`. Works with or without parentheses. |
| `redis_cache(ttl: int \| None = 3600, user: str \| bool \| None = None, shared: bool = False)` | Cache results in Redis under `<module>.<qualname>::<hash of arguments>`. A `None` result is cached too. Adds `func.clear_cache()`. Works with or without parentheses. |
| `http_cache(*, public: bool = False, max_age: int \| None = None, stale_while_revalidate: int \| None = None)` | Set `Cache-Control` on GET responses of a whitelisted method. Does not cache on the server. |
| `deprecated_local_cache(namespace, key, generator, regenerate_if_none=False)` | The function behind the deprecated `frappe.local_cache`. |

!!! warning
    `redis_cache` builds the key with Python `hash()`. String hashes change between processes, so a result cached in one worker is not found by another worker for the same string arguments. Use `frappe.cache.get_value` with an explicit key when workers must share the result.

## Realtime

All in `frappe.realtime`. `frappe.publish_realtime` and `frappe.publish_progress` are aliases. Events go to the Redis channel `events` of `redis_queue`. Connection errors are ignored.

### publish_realtime

```python
frappe.publish_realtime(
	event: str | None = None, message: dict | None = None, room: str | None = None, user: str | None = None,
	doctype: str | None = None, docname: str | None = None, task_id: str | None = None, after_commit: bool = False,
)
```

The room is picked in this order:

1. Event `list_update` always uses `doctype:<doctype>` (from `doctype` or `message["doctype"]`). Event `docinfo_update` always uses `doc:<doctype>/<docname>`. These replace a `room` you pass.
2. `room`, when given.
3. `task_id` (or `frappe.local.task_id`) uses `task_progress:<task_id>`. Adds `task_id` to the message and sets `after_commit=False`.
4. `user` uses `user:<user>`. Event `msgprint` without `user` goes to the session user.
5. `doctype` and `docname` use `doc:<doctype>/<docname>`.
6. Else the site room `all` (System Users of the site).

`event=None` becomes `task_progress` with a task, else `global`. With `after_commit=True`, events are queued and sent after commit, and dropped on rollback. The same event, message and room are sent once.

### Other functions

| Function | Description |
|---|---|
| `publish_progress(percent, title=None, doctype=None, docname=None, description=None, task_id=None)` | Send a `progress` event with `percent`, `title`, `description`. Goes to the document room when `doctype` and `docname` are given, else to the session user. |
| `publish_to_user(user, event, message=None, *, after_commit=False)` | Room `user:<user>`. |
| `publish_to_doc(doctype, docname, event, message=None, *, after_commit=False)` | Room `doc:<doctype>/<docname>`. |
| `publish_to_doctype(doctype, event, message=None, *, after_commit=False)` | Room `doctype:<doctype>`. |
| `publish_task_progress(task_id, message=None, *, after_commit=False)` | Room `task_progress:<task_id>`. `after_commit` has no effect: task events are always sent now. |
| `publish_to_website(event, message=None, *, after_commit=False)` | Room `website`. |
| `publish_to_all(event, message=None, *, after_commit=False)` | Room `all` of the site. |
| `publish_to_room(room, event, message=None, *, after_commit=False)` | Any room. |
| `emit_via_redis(event, message, room)` | Publish now, with no room logic. |
| `flush_realtime_log()`, `clear_realtime_log()` | Send or drop the events queued with `after_commit`. |
| `get_doctype_room(doctype)`, `get_doc_room(doctype, docname)`, `get_user_room(user)`, `get_site_room()`, `get_task_progress_room(task_id)`, `get_website_room()` | Room names. |
| `has_permission(doctype: str, name: str, ptype: str = "read") -> bool` | Whitelisted (GET, guest). Used by the Socket.IO server. Raises `PermissionError`. |
| `get_user_info()` | Whitelisted (GET, guest). Used by the Socket.IO server. Returns `{}` unless the `X-Frappe-Socket-Secret` header matches. |
| `get_socketio_secret()` | Read or make the shared secret in Redis. |

```python
frappe.publish_realtime("sample_order_ready", {"name": "SO-0001"}, user="user1@example.com", after_commit=True)
```

### Python Socket.IO handlers

For the Python Socket.IO backend, `frappe.realtime.realtime` is a `Registry`. `realtime.on(event: str, *, frappe_context: bool = False, allow_guest: bool = False)` registers a handler. Frappe imports `<app>/realtime/handlers.py` of each installed app. The handler gets a `frappe.realtime.Socket` with `site`, `user`, `user_type`, `installed_apps`, `join(room)`, `leave(room)`, `emit(event, data=None, room=None)`, `get(key, default=None)`, `set(key, value)`, `participants(room)`, `user_of(sid)` and `has_permission(doctype, name=None)`. See [Realtime](../../developer-guide/runtime/realtime.md).

## Pitfalls

- A job that reads data from the current request needs `enqueue_after_commit=True`. Else the worker can start before the commit.
- `frappe.enqueue(..., now=True)` runs in the same transaction. The worker commit and retry rules do not apply.
- `get_queues_timeout` is cached for the process. Restart workers after you change `workers` in config.
- `site_cache` is per process. `func.clear_cache()` clears only the current process.
- `frappe.cache` keeps a request-local copy. A value changed by another process in the same request is not seen. Pass `use_local_cache=False` to read Redis.

## Related

- [Background Jobs](../../developer-guide/runtime/background-jobs.md)
- [Scheduler](../../developer-guide/runtime/scheduler.md)
- [Caching](../../developer-guide/runtime/caching.md)
- [Realtime](../../developer-guide/runtime/realtime.md)
- [frappe namespace](frappe.md)
- [Site Config](../configuration/site-config.md)
