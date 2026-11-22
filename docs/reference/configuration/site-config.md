---
title: Site Config Reference
description: Every site_config.json and common_site_config.json key that Frappe Framework v15 reads, with its default and meaning.
version: v15
sources:
  - frappe/__init__.py
  - frappe/app.py
  - frappe/auth.py
  - frappe/boot.py
  - frappe/sessions.py
  - frappe/rate_limiter.py
  - frappe/monitor.py
  - frappe/database/database.py
  - frappe/database/mariadb/database.py
  - frappe/database/mariadb/setup_db.py
  - frappe/database/postgres/setup_db.py
  - frappe/database/db_manager.py
  - frappe/utils/background_jobs.py
  - frappe/utils/redis_wrapper.py
  - frappe/utils/redis_queue.py
  - frappe/utils/scheduler.py
  - frappe/utils/backups.py
  - frappe/utils/data.py
  - frappe/utils/password.py
  - frappe/utils/oauth.py
  - frappe/core/api/file.py
  - frappe/email/doctype/email_account/email_account.py
  - frappe/email/queue.py
  - frappe/core/doctype/scheduled_job_type/scheduled_job_type.py
  - frappe/integrations/frappe_providers/frappecloud_billing.py
  - frappe/utils/telemetry/pulse/client.py
  - node_utils.js
  - realtime/utils.js
---

# Site Config Reference

This page lists every key in `site_config.json` and `common_site_config.json` that the framework reads in v15.
For how the two files merge, how to set keys and how environment variables override them, see [Configuration](../../developer-guide/operations/configuration.md).

## How to read this page

- Frappe reads `sites/common_site_config.json` first, then `sites/<site>/site_config.json`. A site key overrides a common key with the same name. The result is `frappe.conf` (same object as `frappe.local.conf`).
- **Scope** "Common" means the key is read without a site (through `frappe.get_common_site_config()`, `frappe.get_conf()` outside a site, or the Node realtime server). Set it in `common_site_config.json`. "Site" keys work in either file.
- "Default" is the value the code uses when the key is missing. An empty cell means the feature is off or the value is empty.
- Set a key with `bench --site <site> set-config <key> <value>`, or `bench set-config -g <key> <value>` for the common file. Add `-p` to parse the value as a Python literal.

```bash
bench --site frappe.localhost set-config developer_mode 1
bench set-config -g redis_cache "redis://127.0.0.1:13000"
bench --site frappe.localhost set-config -p allow_cors '["https://app.example.com"]'
```

In code, read a key from `frappe.conf`. A missing key returns `None`.

```python
frappe.conf.developer_mode      # 1 or None
frappe.conf.get("max_file_size")
```

## Database

| Key | Scope | Default | Description |
|---|---|---|---|
| `db_type` | Site | `mariadb` | `mariadb` or `postgres`. Env: `FRAPPE_DB_TYPE`. |
| `db_name` | Site | | Database name. Frappe also uses it as the database user. |
| `db_password` | Site | | Database password. |
| `db_host` | Site | `127.0.0.1` | Database host. Env: `FRAPPE_DB_HOST`. |
| `db_port` | Site | 3306 (MariaDB) or 5432 (Postgres) | Database port. Env: `FRAPPE_DB_PORT`. |
| `db_socket` | Site | | Unix socket path. Env: `FRAPPE_DB_SOCKET`. |
| `db_schema` | Site | `public` | Postgres schema. |
| `db_ssl_ca` | Site | | MariaDB: CA file. Turns on SSL. |
| `db_ssl_cert`, `db_ssl_key` | Site | | MariaDB: client certificate and key. Both are needed. |
| `db_ssl_check_hostname` | Site | | MariaDB: check the server host name in SSL. |
| `read_from_replica` | Site | | Send read-only requests and `frappe.read_only` code to the replica. |
| `replica_host` | Site | | Host of the read replica. |
| `replica_db_port` | Site | | Port of the read replica. |
| `different_credentials_for_replica` | Site | | Use `replica_db_name` (as the user) and `replica_db_password` for the replica. |
| `replica_db_name`, `replica_db_password` | Site | | Replica credentials. |
| `max_writes_per_transaction` | Site | 200000 | Max write queries in one transaction before Frappe raises an error. |
| `enable_db_statement_timeout` | Site | | Set a query timeout: 1.5 x `http_timeout` for requests, 1.5 x job timeout for jobs. |
| `http_timeout` | Site | 300 | Request timeout in seconds, used for the statement timeout. |
| `rds_db` | Site | | On new site, grant a limited privilege set instead of `ALL PRIVILEGES` (for Amazon RDS). |
| `root_login` | Common | `root` (MariaDB) | Database root user for `new-site`, `drop-site`, `restore`. Only the Postgres setup reads this key. It asks for the user when the key is missing. |
| `root_password` | Common | | Database root password. When missing, Frappe asks for it. |

## Redis, workers and scheduler

| Key | Scope | Default | Description |
|---|---|---|---|
| `redis_cache` | Common | `redis://127.0.0.1:13311` | Redis URL for the cache. Env: `FRAPPE_REDIS_CACHE`. |
| `redis_queue` | Common | `redis://127.0.0.1:11311` | Redis URL for job queues and realtime. Env: `FRAPPE_REDIS_QUEUE`. |
| `redis_cache_sentinel_enabled` | Common | | Use Redis Sentinel for the cache. |
| `redis_cache_sentinels` | Common | `[]` | List of `"host:port"` Sentinel nodes. |
| `redis_cache_master_service` | Common | | Sentinel master name. |
| `redis_cache_sentinel_username`, `redis_cache_sentinel_password` | Common | | Sentinel credentials. |
| `redis_cache_master_username`, `redis_cache_master_password` | Common | | Master credentials. |
| `redis_queue_sentinel_enabled`, `redis_queue_sentinels`, `redis_queue_master_service`, `redis_queue_sentinel_username`, `redis_queue_sentinel_password`, `redis_queue_master_username`, `redis_queue_master_password` | Common | | Same as the `redis_cache_*` keys, for the queue. |
| `use_rq_auth` | Common | | Connect to the queue with a Redis ACL user. |
| `rq_username`, `rq_password` | Common | `rq_username`: the bench id | Redis ACL credentials. `bench create-rq-users` writes them. |
| `bench_id` | Common | bench path with `/` replaced by `-` | Bench identifier. |
| `workers` | Common | `{}` | Custom queues: `{"<queue>": {"timeout": <seconds>}}`. |
| `max_queued_jobs` | Site | 500 | Max jobs in a queue before `enqueue` fails. |
| `rq_failed_jobs_limit` | Site | 1000 | Max failed jobs kept in the failed registry. |
| `rq_job_failure_ttl` | Site | 604800 (7 days) | Seconds RQ keeps failed jobs. |
| `rq_results_ttl` | Site | 600 | Seconds RQ keeps job results. |
| `background_process_niceness` | Common | 10 | Nice increment for worker and scheduler processes. |
| `disable_scheduler` | Site | | Do not enqueue scheduled jobs for this site. |
| `pause_scheduler` | Site | | Pause the scheduler (set by `bench scheduler pause`). |
| `scheduler_tick_interval` | Common | 60 | Seconds between scheduler ticks. |
| `scheduler_interval` | Site | 240 | Seconds between runs of `all` events. Rounded down to whole minutes in the cron format. |
| `maintenance_mode` | Site | | Requests stop with `SessionStopped`, or run read-only (next key). Scheduled jobs do not run. Set by `bench set-maintenance-mode`. |
| `allow_reads_during_maintenance` | Site | | In maintenance mode, run requests in read-only mode instead of stopping them. |

## Web server and realtime

| Key | Scope | Default | Description |
|---|---|---|---|
| `default_site` | Common | | Site for commands without `--site`. Set by `bench use`. Env: `FRAPPE_SITE`. |
| `webserver_port` | Common | | Port of `bench serve`. Added to site URLs in development. The realtime server calls the web server on this port. |
| `webserver_host` | Common | | Host the realtime server uses to call the web server. |
| `http_port` | Site | | Port added to site URLs. Takes priority over `webserver_port`. |
| `socketio_port` | Common | | Port of the realtime server. Also sent to the browser in boot. Env: `FRAPPE_SOCKETIO_PORT`. |
| `socketio_uds` | Common | | Unix socket for the realtime server. Env: `FRAPPE_SOCKETIO_UDS`. |
| `file_watcher_port` | Common | | Port of the asset watcher. Sent to the browser in boot for live reload. |
| `host_name` | Site | | Base URL of the site, for example `https://erp.example.com`. Used by `frappe.utils.get_url()`. `hostname` is also read. |
| `ssl_certificate` | Site | | Set by bench for SSL sites. Makes URLs use `https`. |
| `wildcard` | Site | | Dict with `domain` and `ssl_certificate` for wildcard SSL. |
| `restart_supervisor_on_update`, `restart_systemd_on_update` | Common | | Production markers. When both are unset, URLs include the port. |
| `allow_cors` | Site | | List of allowed origins (or `"*"`) for CORS. |
| `ignore_csrf` | Site | | Turn off the CSRF token check. Do not use in production. |
| `rate_limit` | Site | | Dict `{"limit": <seconds>, "window": <seconds>}`. Max request time per window. |
| `max_file_size` | Site | 25 MB | Max request body size in bytes. Max upload size when System Settings has none. |
| `max_zip_extract_size` | Site | | Max unzipped size in bytes, when System Settings has none. |
| `max_zip_extract_members` | Site | 1000 | Max files in an uploaded zip. |
| `deny_multiple_sessions` | Site | | One session per user. The System Settings field does the same. |
| `disable_session_cache` | Site | | Build desk boot data on every load instead of reading it from the cache. |
| `disable_website_cache` | Site | | Do not cache rendered web pages. |
| `disable_global_search` | Site | | Do not update or query the global search table. |
| `robots_txt` | Site | | Path to a `robots.txt` file to serve, when Website Settings has none. |
| `website_context` | Site | | Dict added to the context of every web page. |
| `google_analytics_id`, `google_analytics_anonymize_ip` | Site | | Analytics settings for desk, and for the website when Website Settings has none. |
| `app_include_js`, `app_include_css` | Site | `[]` | Extra assets for desk, added after the hook values. |
| `disable_async` | Site | | Do not connect the browser to the realtime server. |
| `private_path` | Site | `private` | Folder for private files, relative to the site. |
| `monitor` | Site | | Log each request and job to the monitor queue. |
| `enable_frappe_logger` | Site | | Log each web request to the `frappe.web` logger. |
| `logging` | Site | | `2` logs every SQL query. Any value shows tracebacks on 404 responses. |

## Development and testing

| Key | Scope | Default | Description |
|---|---|---|---|
| `developer_mode` | Site | | Turns on developer mode: DocType JSON export, no asset cache, more tracebacks. |
| `allow_tests` | Site | | Allow `bench run-tests` and `frappe.print_sql`. |
| `live_reload` | Site | | Reload the browser after an asset build. Env: `LIVE_RELOAD`. |
| `mute_emails` | Site | 0 | Do not send any email. |
| `admin_password` | Site | | Administrator password for `new-site`, `reinstall` and tests. |
| `install_apps` | Site | `[]` | Apps to install on `new-site` after frappe. |
| `skip_setup_wizard` | Site | | Mark setup complete on install. Do not show the setup wizard. |
| `trigger_site_setup_in_background` | Site | | Run the setup wizard tasks in a background job. |
| `ngrok_authtoken` | Site | | Token for `bench ngrok`. |
| `extra_config` | Site | | Dotted path (or list) of functions. Each returns a dict merged into the config. |

## Security and secrets

| Key | Scope | Default | Description |
|---|---|---|---|
| `encryption_key` | Site | made on first use | Key for Password fields and other encrypted values. Frappe writes it to `site_config.json` when it is missing. Keep it with your backups. |
| `backup_encryption_key` | Site | made on first use | Key to encrypt backups. Frappe writes it when it is missing. |
| `secret` | Site | `encryption_key` | Key to sign verified command URLs. |
| `throttle_user_limit` | Site | 60 | Max new users per minute. |
| `notify_admin_access_to_system_manager` | Site | | Email System Managers when Administrator logs in. |
| `<provider>_login` | Site | | Dict `{"client_id", "client_secret"}` for social login, for example `google_login`. |
| `dropbox_access_key`, `dropbox_secret_key` | Site | | Dropbox app credentials, when Dropbox Settings has none. |

## Email

The `mail_*` keys make a default outgoing Email Account from site config when `mail_server` is set.

| Key | Scope | Default | Description |
|---|---|---|---|
| `mail_server` | Site | | SMTP server. |
| `mail_port` | Site | | SMTP port. |
| `mail_login` | Site | | SMTP login. Also the sender when `auto_email_id` is not set. |
| `mail_password` | Site | | SMTP password. |
| `use_tls` | Site | on when `mail_login` is set | Use TLS. |
| `auto_email_id` | Site | `notifications@example.com` | Sender address. |
| `email_sender_name` | Site | `Frappe` | Sender name. |
| `always_use_account_email_id_as_sender`, `always_use_account_name_as_sender_name` | Site | 0 | Always send from the account address and name. |
| `disable_mail_smtp_authentication` | Site | 0 | Send without SMTP login. |
| `email_queue_batch_size` | Site | 500 | Emails sent per run of the email queue job. |
| `imap_timeout`, `pop_timeout` | Site | | Timeouts for incoming mail, when the Email Account has none. |
| `error_report_email` | Site | | Address for error reports from desk. |
| `setup_wizard_exception_email` | Site | | Address that gets setup wizard errors. |
| `site_name` | Site | | Site name in the welcome email subject, when the `site_name` default is not set. |

## Files, backups and limits

| Key | Scope | Default | Description |
|---|---|---|---|
| `backup` | Site | `{}` | Dict with `includes` and `excludes` (DocType lists) for `bench backup`. |
| `backup_path` | Site | `private/backups` | Backup folder, relative to the site. |
| `keep_backups_for_hours` | Site | 23 | Age in hours after which downloadable backups are deleted. |
| `data_import_batch_size` | Site | 1000 | Rows per batch in Data Import. |
| `max_reports_per_user` | Site | System Settings value, then 20 | Max enabled Auto Email Reports per user. |
| `limits` | Site | | Dict. `space_usage` is shown in site usage info. |

## Frappe Cloud and telemetry

These keys are set by Frappe Cloud. You do not set them on a self-hosted site.

| Key | Scope | Description |
|---|---|---|
| `fc_communication_secret`, `fc_base_url`, `saas_billing_site_name` | Site | Billing. `fc_base_url` and `saas_billing_site_name` apply in developer mode only. |
| `subscription` | Site | Subscription data sent in boot. |
| `fc_team`, `pulse_api_key`, `pulse_host`, `pulse_ingest_endpoint` | Site | Telemetry client. |
| `push_relay_server_url` | Site | Relay server for push notifications. |

!!! note "Version"
    These later-version keys are not read in v15: `allowed_referrers`, `client_cache_max_bytes`, `file_chunk_size`, `storage_v2` and the `storage_*` keys, the `chromium_*` keys, `db_user`, `mariadb_root_*`, `postgres_root_*`, `socketio_worker_threads`, `smtp_no_ehlo_after_auth`, `use_ssl`, `esbuild_target`. The `FRAPPE_DB_NAME`, `FRAPPE_DB_USER` and `FRAPPE_DB_PASSWORD` environment variables are not read either.

## Pitfalls

- A process reads the config when it calls `frappe.init`. Restart the web server and workers after you change Redis or database keys.
- A key in `site_config.json` hides the same key in `common_site_config.json`.
- Without `allow_reads_during_maintenance`, maintenance mode stops all requests, not only writes.
- The `auth_method` key for the site config email account is not read. The code passes `("auth_method")`, a string, not a tuple, so it looks up each character.

## Related

- [Configuration](../../developer-guide/operations/configuration.md)
- [CLI Reference](cli.md)
- [Deployment](../../developer-guide/operations/deployment.md)
- [Security](../../developer-guide/security/security.md)
- [Background Jobs](../../developer-guide/runtime/background-jobs.md)
