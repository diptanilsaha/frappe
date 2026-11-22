---
title: Background Jobs
description: Run code outside the web request with frappe.enqueue and frappe.enqueue_doc, on RQ queues served by bench workers.
version: v15
sources:
  - frappe/utils/background_jobs.py
  - frappe/__init__.py
  - frappe/core/doctype/rq_job/rq_job.py
  - frappe/core/doctype/rq_job/rq_job.json
  - frappe/commands/scheduler.py
  - frappe/hooks.py
---

# Background Jobs

Use a background job when work takes longer than a web request should wait: bulk updates, imports, reports, emails, calls to slow external services. Frappe puts the job on a Redis queue (RQ). A worker process (`bench worker`) takes the job and runs it in a fresh site context. Use `frappe.enqueue` for a function and `frappe.enqueue_doc` for a controller method.

## Concepts

- **Queue**: a named RQ queue. Frappe has three built-in queues: `short`, `default` and `long`. You can add more in `common_site_config.json` (see [Queues and timeouts](#queues-and-timeouts)).
- **Worker**: a process that takes jobs from one or more queues. Start it with `bench worker`. In production, supervisor or systemd runs the workers (see [Deployment](../operations/deployment.md)).
- **Job ID**: every job has an ID. Frappe adds the site name as a prefix, so the ID `my_job` becomes `<site>::my_job`. If you do not give an ID, Frappe makes a UUID.
- **RQ Job**: a virtual DocType that shows jobs in Redis. Open it at `/app/rq-job`. It shows the status (`queued`, `started`, `finished`, `failed`, `deferred`, `scheduled`, `canceled`), arguments, timeout and exception.

## Enqueue a function

```python
# sample_app/sample_app/api.py
import frappe


@frappe.whitelist()
def recalculate_rates(category: str):
	frappe.only_for("System Manager")
	frappe.enqueue(
		"sample_app.sample_app.jobs.recalculate_item_rates",
		queue="long",
		timeout=1800,
		category=category,  # passed to the job as a keyword argument
	)
```

```python
# sample_app/sample_app/jobs.py
import frappe


def recalculate_item_rates(category: str):
	for name in frappe.get_all("Sample Item", {"category": category}, pluck="name"):
		item = frappe.get_doc("Sample Item", name)
		item.rate = item.rate * 1.1
		item.save()
```

You can pass the dotted path as a string or the function object. All extra keyword arguments go to the function. They must be picklable.

What the worker does for each job (`execute_job` in `frappe/utils/background_jobs.py`):

1. Connects to the site and sets the user to the user who enqueued the job (`frappe.session.user` at enqueue time).
2. Sets `frappe.job` with `site`, `method`, `job_name`, `kwargs` and `user`.
3. Runs every `before_job` hook.
4. Runs the function.
5. Commits the transaction if the function returns. Rolls back, writes an Error Log and raises again if it fails.
6. Runs every `after_job` hook.

The job does not check permissions for you. The job runs with the roles of the user who enqueued it, so check permissions in the whitelisted method before you enqueue.

### Retries on deadlock

If the function raises a deadlock, a lock wait timeout, or `frappe.RetryBackgroundJobError`, the worker rolls back and runs the function again. It tries up to 5 more times and waits 1, 2, 3, 4 and 5 seconds between attempts. Frappe has no other retry option in `frappe.enqueue`. A retry does not pass the user again, so it runs as `Administrator`.

## Enqueue after commit

When the job reads a document that the current request creates, enqueue it after commit. Otherwise the worker can start before the data is in the database.

```python
def on_submit(self):
	frappe.enqueue(
		"sample_app.sample_app.jobs.notify_order",
		order=self.name,
		enqueue_after_commit=True,
	)
```

With `enqueue_after_commit=True`, `frappe.enqueue` returns `None`. If the transaction rolls back, the job is not queued.

## Deduplicate jobs

Give a `job_id` and set `deduplicate=True` to skip the call when the same job is queued or running.

```python
frappe.enqueue(
	"sample_app.sample_app.jobs.rebuild_index",
	queue="long",
	job_id="sample_rebuild_index",
	deduplicate=True,
)
```

If a job with that ID is `queued` or `started`, `frappe.enqueue` writes a log line and returns `None`. If a finished or failed job with that ID exists, Frappe deletes it and queues a new one. `deduplicate=True` without `job_id` raises `ValidationError`.

Check a job from code:

```python
from frappe.utils.background_jobs import get_job_status, is_job_enqueued

is_job_enqueued("sample_rebuild_index")  # True while queued or started
get_job_status("sample_rebuild_index")   # rq JobStatus, or None if not found
```

Pass the ID without the site prefix. These functions add it.

## Run a controller method in the background

`frappe.enqueue_doc` loads the document in the worker and calls one of its methods.

```python
frappe.enqueue_doc("Sample Order", "SO-0001", "send_reminders", queue="short", timeout=300)
```

The worker calls `frappe.get_doc(doctype, name).<method>(**kwargs)`. The default `timeout` of `enqueue_doc` is 300 seconds.

## The `frappe.task` decorator

`@frappe.task(...)` adds an `enqueue` attribute to a function. The decorator arguments become `frappe.enqueue` arguments.

```python
@frappe.task(queue="long", timeout=1800)
def rebuild_index(category=None):
	...


rebuild_index.enqueue(category="Hardware")
```

## Show progress to the user

Version 15 has no task DocType for user-facing jobs. To show progress, send realtime events from the job with `frappe.publish_progress` or `frappe.publish_realtime`. See [Realtime](realtime.md).

## `frappe.enqueue` parameters

| Name | Type | Default | Description |
|---|---|---|---|
| `method` | `str` or callable | required | Function to run. |
| `queue` | `str` | `"default"` | Queue name. Must be a known queue, or `frappe.enqueue` raises `ValidationError`. |
| `timeout` | `int` | queue timeout | Seconds before RQ stops the job. |
| `event` | `str` | `None` | Label stored with the job. Used by `bench purge-jobs --event`. |
| `is_async` | `bool` | `True` | `False` runs the job in the current process. Deprecated outside tests, use `now`. |
| `job_name` | `str` | `None` | Deprecated. Use `job_id`. |
| `now` | `bool` | `False` | Run the function now with `frappe.call`, in the current process and transaction. |
| `enqueue_after_commit` | `bool` | `False` | Queue the job only after the current transaction commits. |
| `on_success` | callable | `None` | RQ success callback. |
| `on_failure` | callable | `truncate_failed_registry` | RQ failure callback. |
| `at_front` | `bool` | `False` | Put the job at the front of the queue. |
| `job_id` | `str` | UUID | Job ID. Frappe adds `<site>::` before it. |
| `deduplicate` | `bool` | `False` | Do not queue if a job with `job_id` is queued or started. |
| `**kwargs` | | | Arguments for the function. |

Return value: the `rq.job.Job`. It is `None` when the job is deduplicated or queued after commit. With `now=True`, it is the return value of the function.

## Queues and timeouts

| Queue | Default timeout |
|---|---|
| `short` | 300 seconds |
| `default` | 300 seconds |
| `long` | 1500 seconds |

Add a queue in `sites/common_site_config.json` under `workers`:

```json
{
 "workers": {
  "sample_queue": {"timeout": 900}
 }
}
```

Then start a worker for it: `bench worker --queue sample_queue`. In production, run `bench setup supervisor` or `bench setup systemd` again so a worker serves the new queue.

Queue names in Redis are `<bench_id>:<queue>`, so two benches can share one Redis.

If you set `max_queued_jobs`, `frappe.enqueue` raises `frappe.QueueOverloaded` when a queue holds that many jobs. If the key is not set, there is no limit.

## Workers and CLI

| Command | Description |
|---|---|
| `bench worker` | Start a worker for all queues. |
| `bench worker --queue short,default` | Start a worker for some queues. Other options: `--burst` (exit when the queues are empty), `--strategy round_robin\|random`, `--quiet`, `-u/--rq-username`, `-p/--rq-password`. |
| `bench worker-pool --num-workers 4` | Start a pool of workers (experimental). The default is 2 workers. |
| `bench doctor` | Show scheduler status and the number of workers online. |
| `bench --site <site> show-pending-jobs` | Show jobs that wait in the queues. |
| `bench purge-jobs --site <site> [--queue] [--event]` | Delete pending jobs. |

Workers run with a lower CPU priority than web processes. The default niceness increment is 10. Change it with `background_process_niceness` in `common_site_config.json`.

## Configuration keys

| Key | Default | Description |
|---|---|---|
| `workers` | none | Extra queues and their timeouts. `common_site_config.json` only. |
| `max_queued_jobs` | unset (no limit) | Queue size at which `frappe.enqueue` fails. |
| `rq_job_failure_ttl` | 7 days | Seconds to keep failed jobs in Redis. |
| `rq_results_ttl` | 600 | Seconds to keep finished job results in Redis. |
| `rq_failed_jobs_limit` | 1000 | Maximum number of failed jobs kept per queue. |
| `background_process_niceness` | 10 | CPU niceness increment for workers and the scheduler. |
| `use_rq_auth`, `rq_username`, `rq_password` | off | Connect to the queue Redis with an ACL user. |

## Pitfalls

- Do not pass documents or open files as arguments. Pass names and load the data in the job.
- A job that runs with `now=True` shares the caller's transaction. It does not commit.
- The worker commits only when the job ends. A long loop holds its writes until then. Commit in batches if you must, with `frappe.db.commit()`.
- Code changes do not reach running workers. Restart the workers after you deploy.

## Related

- [Scheduler](scheduler.md)
- [Realtime](realtime.md)
- [Configuration](../operations/configuration.md)
- [Logs and monitoring](../../user-guide/system-administration/logs-and-monitoring.md)
- [Performance](../testing-and-performance/performance.md)
- [Hooks reference](../../reference/configuration/hooks.md)
