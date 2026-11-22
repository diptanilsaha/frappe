---
title: Configuration
description: Configure a site with site_config.json and common_site_config.json, read values with frappe.conf, change them with bench set-config, and override them with environment variables.
version: v16
sources:
  - frappe/config.py
  - frappe/installer.py
  - frappe/commands/utils.py
  - frappe/boot.py
  - frappe/utils/background_jobs.py
  - frappe/utils/scheduler.py
  - frappe/utils/redis_wrapper.py
  - frappe/utils/redis_queue.py
  - frappe/utils/safe_exec.py
  - frappe/realtime/config.py
  - frappe/database/database.py
  - frappe/app.py
  - frappe/auth.py
  - frappe/rate_limiter.py
  - frappe/core/api/file.py
  - frappe/email/doctype/email_account/email_account.py
  - frappe/utils/backups.py
  - frappe/integrations/doctype/webhook/webhook.py
---

# Configuration

Frappe reads its configuration from two JSON files in the `sites` folder. `sites/common_site_config.json` holds values for every site on the bench. `sites/<site>/site_config.json` holds values for one site and overrides the common file. Use configuration for things that differ per server or site: credentials, Redis URLs, ports, feature flags. Use a Single DocType such as System Settings for things that a user changes in the desk.

## How the values merge

`frappe.get_site_config()` builds the configuration in this order. A later step wins.

1. `common_site_config.json`.
2. `site_config.json` of the site.
3. Environment variables (see [Environment variables](#environment-variables)).
4. Defaults: `db_type` = `mariadb`, `db_host` = `127.0.0.1`, `db_port` = the default port of the database, `db_user` = `db_name` (and the other way round), `redis_queue` = `redis://127.0.0.1:11311`, `redis_cache` = `redis://127.0.0.1:13311`.
5. `extra_config` hooks: if the config has `extra_config` (a dotted path or a list of paths), Frappe calls each function and merges the dict it returns.

## Read configuration in code

```python
import frappe

frappe.conf.developer_mode           # current site, merged config. A missing key returns None.
frappe.conf.get("max_file_size", 0)  # with a default

frappe.get_site_config()             # merged config of the current site, read from disk
frappe.get_common_site_config()      # common_site_config.json only
frappe.get_conf("frappe.localhost")  # config of another site
frappe.get_conf()                    # current site, or common config when no site is set
```

`frappe.conf` is `frappe.local.conf`. It is loaded once per request or job by `frappe.init`. The result is a `frappe._dict`, so attribute access returns `None` for a missing key.

Some keys are read with `frappe.get_common_site_config()` or before a site is set. These keys work only in `common_site_config.json`. They are marked **common** in the tables below.

!!! warning
    Site config holds secrets: `db_password`, `encryption_key`, mail and Redis passwords. Never send `frappe.conf` to the browser. `frappe.boot` gets only `developer_mode`, `socketio_port`, `socketio_backend`, `file_watcher_port`, `max_file_size` and `file_chunk_size`.

## Change configuration

```bash
bench --site frappe.localhost set-config developer_mode 1
bench --site frappe.localhost set-config allow_cors '["https://app.example.com"]' --parse
bench set-config -g scheduler_tick_interval 60        # common_site_config.json
bench --site frappe.localhost set-config pause_scheduler None   # remove the key
bench --site frappe.localhost show-config                       # print the merged config
```

- `-g`, `--global`: write to `common_site_config.json`.
- `-p`, `--parse`: read the value with `ast.literal_eval` as a Python literal (`True`, `False`, numbers, lists, dicts). Lowercase `true` fails with `--parse`.
- Without `--parse`, `"0"` and `"1"` are saved as integers, and `"true"` and `"false"` as booleans. The value `None` removes the key.

From code, use `frappe.installer.update_site_config(key, value)`. It locks the file, writes it, and clears the cached config. Running processes read the new value on their next request. Restart workers and the Socket.IO server for keys they read at startup (`workers`, `socketio_*`, `redis_*`).

## Keys by purpose

This is the list of keys that you are most likely to set. For every key read by the framework, see the [site config reference](../../reference/configuration/site-config.md).

### Database

| Key | Default | Description |
|---|---|---|
| `db_type` | `mariadb` | `mariadb`, `postgres` or `sqlite`. |
| `db_name` | `db_user` | Database name. Also the Redis key prefix of the site. |
| `db_user`, `db_password` | `db_name`, none | Database credentials. |
| `db_host`, `db_port`, `db_socket` | `127.0.0.1`, default port, none | Where the database server is. |
| `db_ssl_ca`, `db_ssl_cert`, `db_ssl_key`, `db_ssl_check_hostname` | none | TLS for MariaDB. |
| `db_schema` | none | Schema for Postgres. |
| `read_from_replica` | off | Send read-only requests to a replica. Needs `replica_host`. |
| `replica_host`, `replica_db_port` | none | Replica server. |
| `different_credentials_for_replica`, `replica_db_name`, `replica_db_user`, `replica_db_password` | off | Use other credentials for the replica. |
| `max_writes_per_transaction` | framework limit | Raise the limit of writes in one transaction. |
| `enable_db_statement_timeout` | off | Stop queries after 1.5 times the request timeout (`http_timeout`, default 300 s) or the job timeout. |
| `rds_db` | off | Grant a reduced set of privileges when making the site database, for managed databases. |

### Redis, queues and workers

| Key | Default | Description |
|---|---|---|
| `redis_cache` | `redis://127.0.0.1:13311` | Redis for `frappe.cache`. |
| `redis_queue` | `redis://127.0.0.1:11311` | Redis for job queues and realtime events. **common** |
| `redis_cache_sentinel_enabled`, `redis_cache_sentinels`, `redis_cache_master_service`, ... | off | Connect to the cache through Redis Sentinel. The same keys exist with the `redis_queue_` prefix. |
| `workers` | none | Extra queues: `{"<queue>": {"timeout": <seconds>}}`. **common** |
| `use_rq_auth`, `rq_username`, `rq_password` | off | Redis ACL user for the queue. **common** |
| `max_queued_jobs` | 500 | Queue size at which `frappe.enqueue` fails. |
| `rq_job_failure_ttl`, `rq_results_ttl`, `rq_failed_jobs_limit` | 7 days, 600 s, 1000 | How long RQ keeps job data. |
| `background_process_niceness` | 10 | CPU niceness increment for workers and the scheduler. **common** |
| `client_cache_max_bytes` | no limit | Memory limit of `frappe.client_cache` per process. |

See [Background Jobs](../runtime/background-jobs.md).

### Scheduler and maintenance

| Key | Default | Description |
|---|---|---|
| `scheduler_tick_interval` | 240 | Seconds between scheduler checks. **common** |
| `scheduler_interval` | 240 | Interval of `all` scheduler events, in seconds. |
| `pause_scheduler` | off | Pause scheduled jobs. |
| `disable_scheduler` | off | Disable scheduled jobs. |
| `maintenance_mode` | off | Block writes and stop the scheduler. |
| `allow_reads_during_maintenance` | on | In maintenance mode, keep the site open for reads. |

See [Scheduler](../runtime/scheduler.md).

### Web server, URLs and realtime

| Key | Default | Description |
|---|---|---|
| `webserver_port` | none | Port of `bench serve`. Used in URLs when `host_name` is not set. **common** |
| `host_name` (or `hostname`) | none | Full site URL, such as `https://erp.example.com`. Used by `frappe.utils.get_url`. |
| `http_port` | none | Port added to URLs made from the request host. |
| `default_site` | none | Site used when no `--site` is given. **common** |
| `socketio_port` | 9000 | Port of the Socket.IO server. **common** |
| `socketio_backend` | `node` | `node` or `python`. Read by `bench socketio`. **common** |
| `socketio_uds`, `socketio_worker_threads` | none, 4 | Socket.IO Unix socket and Python server threads. **common** |
| `live_reload`, `file_watcher_port` | off | Development asset watcher. **common** |
| `restart_supervisor_on_update`, `restart_systemd_on_update` | off | Restart services after `bench update`. |

See [Realtime](../runtime/realtime.md) and [Deployment](deployment.md).

### Security and requests

| Key | Default | Description |
|---|---|---|
| `encryption_key` | made at site creation | Key for encrypted Password fields and backups. Keep a copy. Without it, encrypted data cannot be read. |
| `allow_cors` | none | `"*"`, one origin, or a list of origins allowed to call the API from a browser. |
| `allowed_referrers` | `[]` | Extra referrers accepted by the CSRF check. |
| `ignore_csrf` | off | Turn off the CSRF check. Only for development. |
| `deny_multiple_sessions` | off | Log out other sessions of a user at login. Also in System Settings. |
| `rate_limit` | none | `{"limit": <seconds>, "window": <seconds>}`. Total request time the site may use in each window. Over the limit, requests get HTTP 429. |
| `throttle_user_limit` | 60 | Maximum users created in 60 seconds. |
| `server_script_enabled` | off | Allow Server Scripts and safe exec. **common** |
| `notify_admin_access_to_system_manager` | off | Email System Managers when Administrator logs in. |

See [Security](../security/security.md).

### Files

| Key | Default | Description |
|---|---|---|
| `max_file_size` | 25 MB | Maximum upload size in bytes. System Settings `max_file_size` (in MB) wins when set. |
| `file_chunk_size` | 25 MB | Chunk size in bytes for uploads from the desk. |
| `max_zip_extract_size`, `max_zip_extract_members` | 25 MB, 1000 | Limits for unzipping a File. System Settings `max_zip_extract_size` (in MB) wins when set. |
| `private_path` | `private` | Folder of private files, relative to the site folder. |

### Email

These keys make a default outgoing account when `mail_server` is set and no Email Account exists for sending.

| Key | Description |
|---|---|
| `mail_server`, `mail_port` | SMTP server and port. |
| `mail_login`, `mail_password` | SMTP login. |
| `use_tls`, `use_ssl` | SMTP encryption. |
| `auto_email_id` | Sender address. Default: `mail_login`, else `notifications@example.com`. |
| `email_sender_name` | Sender name. Default `Frappe`. |
| `always_use_account_email_id_as_sender`, `always_use_account_name_as_sender_name` | Force the account sender. |
| `disable_mail_smtp_authentication` | Send without SMTP login. |
| `mute_emails` | Do not send emails. |
| `email_queue_batch_size` | Emails sent per queue flush. Default 500. |
| `pop_timeout`, `imap_timeout` | Timeouts for incoming mail. |

### Development and logging

| Key | Default | Description |
|---|---|---|
| `developer_mode` | off | Write DocType and other standard changes to app files. Turns off website cache. |
| `allow_tests` | off | Allow `bench run-tests` on the site. Not needed when `CI` is set. |
| `logging` | 0 | `2` logs each SQL query. A true value also shows tracebacks on 404 responses. |
| `enable_frappe_logger` | off | Write request logs with `frappe.logger`. |
| `monitor` | off | Record request and job metrics (`frappe.monitor`). |
| `disable_async` | off | The desk does not connect to the Socket.IO server. |
| `disable_website_cache` | off | Do not cache rendered website pages. |
| `disable_global_search` | off | Do not update the global search index. |
| `error_report_email` | none | Recipient of error reports that users send from the desk. |

### Install and backups

| Key | Default | Description |
|---|---|---|
| `admin_password` | none | Administrator password used by `bench new-site` and tests. |
| `install_apps` | none | Apps to install with every new site (after `frappe`). |
| `skip_setup_wizard` | off | Do not open the setup wizard after install. |
| `backup` | none | `{"includes": [...], "excludes": [...]}`: tables for a partial backup. |
| `keep_backups_for_hours` | 23 | Hours before temporary backup files in the backup folder are deleted. |
| `lang` | `en` | Language used before a user is known. |

Other groups: `chromium_*` and `use_persistent_chromium` (PDF with Chromium), `flow_*` (Flow server), `pulse_*` (telemetry), `cloud_*` and `fc_*` (Frappe Cloud). See the [site config reference](../../reference/configuration/site-config.md).

## Environment variables

| Variable | Overrides |
|---|---|
| `FRAPPE_DB_TYPE`, `FRAPPE_DB_HOST`, `FRAPPE_DB_PORT`, `FRAPPE_DB_SOCKET`, `FRAPPE_DB_NAME`, `FRAPPE_DB_USER`, `FRAPPE_DB_PASSWORD` | The matching `db_*` keys. |
| `FRAPPE_REDIS_QUEUE`, `FRAPPE_REDIS_CACHE` | `redis_queue`, `redis_cache`. |
| `FRAPPE_SITE` | Default site for `bench` commands. |
| `SITES_PATH` | Sites folder used by `frappe.get_conf()` without a site. |
| `FRAPPE_SENTRY_DSN` | Send errors from workers to Sentry. |
| `FRAPPE_BACKGROUND_WORKERS_NOFORK` | `bench worker-pool` runs jobs without fork. |

Environment variables win over both JSON files.

## Pitfalls

- A key in `site_config.json` hides the same key in `common_site_config.json` for that site.
- **common** keys in `site_config.json` have no effect.
- `frappe.conf` is loaded per request. A long-running process (worker, scheduler, Socket.IO) reads some keys only at startup.
- `bench set-config` writes valid JSON. Hand edits that break the JSON stop every request with "site_config.json is invalid".
- Do not commit `site_config.json` or `common_site_config.json` to a repository.

## Related

- [Site config reference](../../reference/configuration/site-config.md)
- [CLI reference](../../reference/configuration/cli.md)
- [Deployment](deployment.md)
- [Security](../security/security.md)
- [System Settings](../../user-guide/settings/system-settings.md)
