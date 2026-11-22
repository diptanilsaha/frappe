---
title: Scheduler
description: Run functions on a schedule with the scheduler_events hook, Scheduled Job Type records and the bench schedule process.
version: develop
sources:
  - frappe/utils/scheduler.py
  - frappe/core/doctype/scheduled_job_type/scheduled_job_type.py
  - frappe/commands/scheduler.py
  - frappe/hooks.py
  - frappe/migrate.py
  - frappe/utils/background_jobs.py
---

# Scheduler

Use the scheduler to run a function at fixed times: each few minutes, hourly, daily, or on a cron expression. You declare the function in the `scheduler_events` hook of your app. Frappe stores each entry as a `Scheduled Job Type` record. The `bench schedule` process checks these records and puts due jobs on a [background queue](background-jobs.md). A worker then runs them.

## How it works

1. `bench migrate` (and app install) reads `scheduler_events` from every installed app. It creates or updates one `Scheduled Job Type` per method, and deletes records whose method is no longer in the hooks.
2. The `bench schedule` process wakes up each 4 minutes by default (`scheduler_tick_interval`).
3. For each site, it skips the site if the scheduler is inactive (see [Enable and disable](#enable-and-disable)).
4. For each `Scheduled Job Type` that is not `stopped`, it computes the next run from `last_execution` and the frequency. If the run is due, it enqueues the job.
5. The job ID is `scheduled_job||<Scheduled Job Type name>`. A job that is still queued or running is not queued again.
6. The worker runs the method, commits, and writes a `Scheduled Job Log` row (`Start`, `Complete` or `Failed`).

The scheduler only queues jobs. If no worker runs, no scheduled job runs.

## Declare scheduled events

```python
# sample_app/hooks.py
scheduler_events = {
	"all": [
		"sample_app.tasks.sync_stock",
	],
	"hourly": [
		"sample_app.tasks.send_reminders",
	],
	"daily": [
		"sample_app.tasks.close_old_orders",
	],
	"daily_long": [
		"sample_app.tasks.rebuild_rates",
	],
	"weekly": [],
	"monthly": [],
	"cron": {
		"*/15 * * * *": ["sample_app.tasks.poll_service"],
		"30 2 * * 1-5": ["sample_app.tasks.weekday_report"],
	},
}
```

```python
# sample_app/tasks.py
import frappe


def close_old_orders():
	for name in frappe.get_all("Sample Order", {"status": "Open", "docstatus": 0}, pluck="name"):
		frappe.get_doc("Sample Order", name).add_comment("Comment", "Checked by scheduler")
```

The method takes no arguments. It runs as the user `Administrator`, the user that the scheduler process uses when it enqueues. Run `bench migrate` after you change `scheduler_events`.

## Event keys

The key in `scheduler_events` sets the `frequency` of the `Scheduled Job Type`. Frappe converts `daily_long` to `Daily Long`.

| Key | Frequency | Runs | Queue |
|---|---|---|---|
| `all` | `All` | Each `scheduler_interval` seconds (default 240, so each 4 minutes) | `default` |
| `hourly` | `Hourly` | At minute 0 of each hour | `default` |
| `hourly_long` | `Hourly Long` | At minute 0 of each hour | `long` |
| `hourly_maintenance` | `Hourly Maintenance` | Each hour, at a minute offset that depends on the site | `long` |
| `daily` | `Daily` | At 00:00 | `default` |
| `daily_long` | `Daily Long` | At 00:00 | `long` |
| `daily_maintenance` | `Daily Maintenance` | Each day, at a minute offset that depends on the site | `long` |
| `weekly` | `Weekly` | Sunday 00:00 | `default` |
| `weekly_long` | `Weekly Long` | Sunday 00:00 | `long` |
| `monthly` | `Monthly` | 1st day of the month, 00:00 | `default` |
| `monthly_long` | `Monthly Long` | 1st day of the month, 00:00 | `long` |
| `yearly`, `annual` | `Yearly`, `Annual` | 1 January, 00:00 | `default` |
| `cron` | `Cron` | The cron expression (dict key) | `default`, or the `queue` field |

Times are in the system time zone of the site. The maintenance offset (0 to 59 minutes) comes from a hash of the site name. It spreads the load when many sites share one bench. Use the `_maintenance` keys for work that must run about once an hour or day but not at an exact time.

Cron expressions use the `croniter` syntax. An invalid expression fails validation on save with "Bad Cron Expression".

## Scheduled Job Type

Open the list at `/desk/scheduled-job-type`. Fields:

| Field | Description |
|---|---|
| `method` | Dotted path of the function. |
| `frequency` | See the table above. |
| `cron_format` | Cron expression. Required for `Cron`. |
| `queue` | Queue for `Cron` jobs. Empty means `default`. Cleared for other frequencies. |
| `stopped` | Check to stop this job. Migrate keeps this value. |
| `create_log` | Write a `Scheduled Job Log` for each run. Always on for frequencies other than `All` and `Cron`. |
| `last_execution` | Time of the last run. The next run is computed from it. |
| `server_script` | Set for jobs that come from a Server Script of type Scheduler Event. |

Actions in the form (System Manager only): **Execute** queues the job now (`execute_event`). **Skip Next Execution** moves `last_execution` to the next run time (`skip_next_execution`).

Run a job now from the CLI. This runs it in the current process, not in a worker:

```bash
bench --site frappe.localhost trigger-scheduler-event sample_app.tasks.close_old_orders
```

## Enable and disable

The scheduler is inactive for a site when one of these is true:

- `maintenance_mode` is set in site config.
- `pause_scheduler` is set in site config.
- `disable_scheduler` is set in site config.
- **System Settings > Enable Scheduler** (`enable_scheduler`) is off.

| Command | Effect |
|---|---|
| `bench --site <site> enable-scheduler` | Sets `enable_scheduler` = 1 in System Settings. |
| `bench --site <site> disable-scheduler` | Sets `enable_scheduler` = 0. |
| `bench --site <site> scheduler enable\|disable` | Same as the two commands above. |
| `bench --site <site> scheduler pause\|resume` | Sets or clears `pause_scheduler` in `site_config.json`. |
| `bench --site <site> scheduler status [-f json]` | Prints `enabled` or `disabled`. |
| `bench --site <site> set-maintenance-mode on\|off` | Sets `maintenance_mode`. Scheduled jobs do not run in maintenance mode. |
| `bench schedule` | Starts the scheduler process for all sites of the bench. |
| `bench doctor` | Shows scheduler status per site and workers online. |

The whitelisted method `frappe.utils.scheduler.activate_scheduler` (Administrator only) enables the scheduler and clears `pause_scheduler`. It fails in maintenance mode.

!!! note
    On Frappe Cloud, a site with no user activity for `dormant_days` (System Settings) runs scheduled jobs only once each 24 hours. This check is off in developer mode and on other hosts.

## Configuration keys

| Key | File | Default | Description |
|---|---|---|---|
| `scheduler_tick_interval` | `common_site_config.json` | 240 | Seconds between scheduler checks. |
| `scheduler_interval` | site or common | 240 | Interval of `all` events, in seconds. Converted to whole minutes. |
| `pause_scheduler` | site | unset | Pause scheduling for the site. |
| `disable_scheduler` | site | unset | Disable scheduling for the site. |
| `maintenance_mode` | site | unset | Stops the scheduler and blocks writes. |

## Pitfalls

- A disabled app keeps its `Scheduled Job Type` rows, but its jobs do not run. See [App State](../building-apps/app-state.md).
- A job that is still running when its next run is due is skipped, not queued twice.
- Long jobs on `default` hold up short ones. Use a `_long` key, or set `queue` on a cron job.
- `trigger-scheduler-event` runs in your shell. Errors show there, not in a worker log.

## Related

- [Background Jobs](background-jobs.md)
- [Hooks](../building-apps/hooks.md)
- [Server and Client Scripts](../low-code-and-ai/server-and-client-scripts.md)
- [Logs and monitoring](../../user-guide/system-administration/logs-and-monitoring.md)
- [CLI reference](../../reference/configuration/cli.md)
