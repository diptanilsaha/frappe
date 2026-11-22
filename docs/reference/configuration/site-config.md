---
title: Site Config Reference
description: Every site_config.json and common_site_config.json key that the Frappe Framework reads, with its default and meaning.
version: v16
sources:
  - frappe/config.py
  - frappe/__init__.py
  - frappe/app.py
  - frappe/auth.py
  - frappe/boot.py
  - frappe/database/database.py
  - frappe/database/mariadb/database.py
  - frappe/database/postgres/database.py
  - frappe/database/mariadb/setup_db.py
  - frappe/database/postgres/setup_db.py
  - frappe/database/db_manager.py
  - frappe/utils/background_jobs.py
  - frappe/utils/redis_wrapper.py
  - frappe/utils/redis_queue.py
  - frappe/utils/scheduler.py
  - frappe/utils/backups.py
  - frappe/utils/data.py
  - frappe/utils/print_utils.py
  - frappe/utils/pdf_generator/chrome_pdf_generator.py
  - frappe/utils/global_search.py
  - frappe/database/__init__.py
  - frappe/realtime/config.py
  - frappe/email/doctype/email_account/email_account.py
  - frappe/core/api/file.py
  - frappe/flow/daemon/lifecycle.py
  - frappe/integrations/frappe_providers/cloud_settings.py
  - frappe/utils/telemetry/pulse/client.py
---

# Site Config Reference

This page lists every key in `site_config.json` and `common_site_config.json` that the framework reads in this version.
For how the two files merge, how to set keys and how environment variables override them, see [Configuration](../../developer-guide/operations/configuration.md).

## How to read this page

- Frappe reads `sites/common_site_config.json` first, then `sites/<site>/site_config.json`. A site key overrides a common key with the same name. The result is `frappe.conf` (same object as `frappe.local.conf`).
- **Scope** "Common" means the key is read without a site (through `frappe.get_common_site_config()`, or `frappe.get_conf()` outside a site). Set it in `common_site_config.json`. "Site" keys work in either file.
- "Default" is the value the code uses when the key is missing. An empty cell means the feature is off or the value is empty.
- Set a key with `bench --site <site> set-config <key> <value>`, or `bench set-config -g <key> <value>` for the common file. Add `-p` to parse the value as JSON or a Python literal.

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
| `db_type` | Site | `mariadb` | `mariadb`, `postgres` or `sqlite`. Env: `FRAPPE_DB_TYPE`. |
| `db_name` | Site | value of `db_user` | Database name. Env: `FRAPPE_DB_NAME`. |
| `db_user` | Site | value of `db_name` | Database user. Env: `FRAPPE_DB_USER`. |
| `db_password` | Site | | Database password. Env: `FRAPPE_DB_PASSWORD`. |
| `db_host` | Site | `127.0.0.1` | Database host. Env: `FRAPPE_DB_HOST`. |
| `db_port` | Site | 3306 (MariaDB) or 5432 (Postgres) | Database port. Env: `FRAPPE_DB_PORT`. |
| `db_socket` | Site | | Unix socket path. Env: `FRAPPE_DB_SOCKET`. |
| `db_schema` | Site | `public` | Postgres schema. |
| `db_ssl_ca` | Site | | MariaDB: CA file. Turns on SSL. |
| `db_ssl_cert`, `db_ssl_key` | Site | | MariaDB: client certificate and key. Both are needed. |
| `db_ssl_check_hostname` | Site | | MariaDB: check the server host name in SSL. |
| `use_mysqlclient` | Site | 1 | MariaDB driver. `0` uses PyMySQL, which is deprecated and prints a warning. |
| `read_from_replica` | Site | | Send read-only requests and `frappe.read_only` code to the replica. |
| `replica_host` | Site | | Host of the read replica. |
| `replica_db_port` | Site | | Port of the read replica. |
| `different_credentials_for_replica` | Site | | Use `replica_db_user` (or `replica_db_name`) and `replica_db_password` for the replica. |
| `replica_db_user`, `replica_db_name`, `replica_db_password` | Site | | Replica credentials. |
| `max_writes_per_transaction` | Site | 200000 | Max write queries in one transaction before Frappe raises an error. |
| `enable_db_statement_timeout` | Site | | Set a query timeout: 1.5 x `http_timeout` for requests, 1.5 x job timeout for jobs. |
| `http_timeout` | Site | 300 | Request timeout in seconds, used for the statement timeout. |
| `rds_db` | Site | | On new site, grant a limited privilege set instead of `ALL PRIVILEGES` (for Amazon RDS). |
| `root_login`, `root_password` | Common | | Database root credentials for `new-site`, `drop-site`, `restore`. |
| `mariadb_root_login`, `mariadb_root_password` | Common | | MariaDB root credentials. |
| `postgres_root_login`, `postgres_root_password` | Common | | Postgres root credentials. |

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
| `client_cache_max_bytes` | Site | 0 (no limit) | Max size of the in-process client cache. |
| `workers` | Common | `{}` | Custom queues: `{"<queue>": {"timeout": <seconds>}}`. |
| `max_queued_jobs` | Site | framework constant | Max jobs in a queue before `enqueue` fails. |
| `rq_failed_jobs_limit` | Site | framework constant | Max failed jobs kept in the failed registry. |
| `rq_job_failure_ttl`, `rq_results_ttl` | Site | framework constants | Seconds RQ keeps failed jobs and job results. |
| `background_process_niceness` | Common | framework constant | Nice increment for worker processes. |
| `disable_scheduler` | Site | | Do not enqueue scheduled jobs for this site. |
| `pause_scheduler` | Site | | Pause the scheduler (set by `bench scheduler pause`). |
| `scheduler_tick_interval` | Common | framework constant | Seconds between scheduler ticks. |
| `scheduler_interval` | Site | 240 | Seconds between runs of `all` events. |
| `maintenance_mode` | Site | | Requests run read-only (or stop, see next key). Scheduled jobs do not run. Set by `bench set-maintenance-mode`. |
| `allow_reads_during_maintenance` | Site | `True` | In maintenance mode, run requests in read-only mode. When `false`, requests stop with `SessionStopped`. |

## Web server and realtime

| Key | Scope | Default | Description |
|---|---|---|---|
| `default_site` | Common | | Site for commands without `--site`. Set by `bench use`. |
| `webserver_port` | Common | | Port of `bench serve` and the dev server. |
| `webserver_host` | Common | | Host the realtime server uses to call the web server. |
| `http_port` | Site | | Port added to site URLs. |
| `socketio_port` | Common | 9000 | Port of the realtime server. Also sent to the browser. |
| `socketio_uds` | Common | | Unix socket for the realtime server. |
| `socketio_worker_threads` | Common | 4 | Threads of the realtime server. |
| `host_name` | Site | | Base URL of the site, for example `https://erp.example.com`. Used by `frappe.utils.get_url()`. `hostname` is also read. |
| `ssl_certificate` | Site | | Set by bench for SSL sites. Makes URLs use `https`. |
| `wildcard` | Site | | Dict with `domain` and `ssl_certificate` for wildcard SSL. |
| `restart_supervisor_on_update`, `restart_systemd_on_update` | Common | | Production markers. When both are unset, URLs include the port. |
| `allow_cors` | Site | | List of allowed origins (or `"*"`) for CORS on `/api`. |
| `allowed_referrers` | Site | `[]` | Referrers or origins that pass the CSRF check without a token. |
| `ignore_csrf` | Site | | Turn off the CSRF token check. Do not use in production. |
| `rate_limit` | Site | | Dict `{"limit": <seconds>, "window": <seconds>}`. Max request time per window. Applied in `before_request`. |
| `max_file_size` | Site | 25 MB | Max request body and upload size in bytes. |
| `file_chunk_size` | Site | 25 MB | Chunk size in bytes for chunked uploads. |
| `max_zip_extract_size`, `max_zip_extract_members` | Site | members: 1000 | Limits for unzipping uploaded files. |
| `deny_multiple_sessions` | Site | | One session per user. The System Settings field does the same. |
| `disable_website_cache` | Site | | Do not cache rendered web pages. |
| `robots_txt` | Site | | Path to a `robots.txt` file to serve. |
| `website_context` | Site | | Dict added to the context of every web page. |
| `google_analytics_id`, `google_analytics_anonymize_ip` | Site | | Analytics settings for desk and website, when not set in Website Settings. |
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
| `allow_tests` | Site | | Allow `bench run-tests` (also allowed when the `CI` env variable is set) and `frappe.print_sql`. |
| `live_reload` | Site | | Reload the browser after an asset build. Env: `LIVE_RELOAD`. |
| `esbuild_target` | Common | | esbuild target for `bench build`. Env: `ESBUILD_TARGET`. |
| `mute_emails` | Site | 0 | Do not send any email. |
| `admin_password` | Site | | Administrator password for `new-site`, `reinstall` and tests. A value in the config wins over the password passed to the installer. |
| `install_apps` | Site | `[]` | Apps to install on `new-site` after frappe. |
| `skip_setup_wizard` | Site | | Mark setup complete on install. Do not show the setup wizard. |
| `ngrok_authtoken` | Site | | Token for `bench ngrok`. |
| `extra_config` | Site | | Dotted path (or list) of functions. Each returns a dict merged into the config. |
| `disable_global_search` | Site | | Do not update or query the global search table. |

## Security and secrets

| Key | Scope | Default | Description |
|---|---|---|---|
| `encryption_key` | Site | made on first use | Key for Password fields and other encrypted values. Frappe writes it to `site_config.json` when it is missing. Keep it with your backups. |
| `backup_encryption_key` | Site | | Key to encrypt backups. |
| `secret` | Site | `encryption_key` | Key to sign verified command URLs. |
| `throttle_user_limit` | Site | 60 | Max new users per minute. |
| `notify_admin_access_to_system_manager` | Site | | Email System Managers when Administrator logs in. |
| `<provider>_login` | Site | | Dict `{"client_id", "client_secret"}` for social login, for example `google_login`. |

## Email

The `mail_*` keys make a default outgoing Email Account from site config when `mail_server` is set.

| Key | Scope | Default | Description |
|---|---|---|---|
| `mail_server` | Site | | SMTP server. |
| `mail_port` | Site | | SMTP port. |
| `mail_login` | Site | | SMTP login. Also the sender when `auto_email_id` is not set. |
| `mail_password` | Site | | SMTP password. |
| `use_tls`, `use_ssl` | Site | | Use TLS or SSL. |
| `auto_email_id` | Site | `notifications@example.com` | Sender address. |
| `email_sender_name` | Site | `Frappe` | Sender name. |
| `always_use_account_email_id_as_sender`, `always_use_account_name_as_sender_name` | Site | 0 | Always send from the account address and name. |
| `disable_mail_smtp_authentication` | Site | 0 | Send without SMTP login. |
| `smtp_no_ehlo_after_auth` | Site | | Do not send `EHLO` again after login. |
| `email_queue_batch_size` | Site | 500 | Emails sent per run of the email queue job. |
| `imap_timeout`, `pop_timeout` | Site | | Timeouts for incoming mail, when the Email Account has none. |
| `error_report_email` | Site | | Address for error reports from desk. |
| `setup_wizard_exception_email` | Site | | Address that gets setup wizard errors. |

## Files and backups

| Key | Scope | Default | Description |
|---|---|---|---|
| `backup` | Site | `{}` | Dict with `includes` and `excludes` (DocType lists) for `bench backup`. |
| `backup_path` | Site | `private/backups` | Backup folder, relative to the site. |
| `keep_backups_for_hours` | Site | 23 | Age in hours after which downloadable backups are deleted. |
| `data_import_batch_size` | Site | 1000 | Rows per batch in Data Import. |
| `max_bulk_print_docs`, `max_concurrent_bulk_exports` | Site | | Limits for bulk PDF export. |
| `max_reports_per_user` | Site | System Settings value, then 20 | Max enabled Auto Email Reports per user. |
| `limits` | Site | | Dict. `space_usage` is shown in site usage info. |

## PDF (Chromium)

| Key | Scope | Default | Description |
|---|---|---|---|
| `chromium_path` | Common | | Path to a Chromium binary to use instead of the downloaded one. |
| `chromium_websocket_url` | Common | | DevTools URL of an external Chromium. |
| `chromium_max_concurrent` | Common | 1 | Open connections to one Chromium process. |
| `use_persistent_chromium` | Common | `False` | Keep Chromium running between PDFs. |
| `chromium_start_timeout` | Common | 10 | Seconds to wait for Chromium to start. |
| `chromium_download_url` | Common | | Full download URL for `bench setup-chrome`. |
| `chromium_version`, `chromium_download_base_url`, `playwright_chromium_version`, `playwright_chromium_download_base_url` | Common | built-in | Override the Chromium build that `setup-chrome` downloads. |

## Automation, Flow and AI

| Key | Scope | Default | Description |
|---|---|---|---|
| `automation_disabled` | Site | | Turn off the automation engine. |
| `ai_keys` | Site | | List of `{"gateway_url", "api_key"}`. Frappe Cloud writes it. Flow uses the first entry. |
| `flow_server_url` | Common | `http://127.0.0.1:<webserver_port or 8000>` | Base URL that the Flow server uses to call back the web server. |
| `flow_server_secret` | Site | | Shared secret between the site and the Flow server. |
| `flow_server_manager` | Common | `local` | `local` or `supervisor`. How `bench flow-server` runs the daemon. |
| `flow_redis_url` | Site | value of `redis_queue` | Redis for Flow events. |
| `flow_callback_timeout`, `flow_background_tool_timeout` | Site | | Timeouts for Flow tool calls. |
| `push_relay_server_url` | Site | | Relay server for push notifications. |

## Frappe Cloud and telemetry

These keys are set by Frappe Cloud. You do not set them on a self-hosted site.

| Key | Scope | Description |
|---|---|---|
| `cloud_provider`, `cloud_server_url`, `cloud_account_url`, `cloud_settings_embed_url`, `cloud_settings_embed_version`, `cloud_settings_timeout` | Site | Cloud settings page in desk. |
| `pilot_endpoint`, `pilot_auth_token` | Site | Cloud control API. |
| `fc_communication_secret`, `fc_base_url`, `saas_billing_site_name` | Site | Billing. `fc_base_url` and `saas_billing_site_name` apply in developer mode only. |
| `mock_fc_billing`, `mock_fc_trial_days`, `mock_fc_consulting_cta` | Site | Mock billing in developer mode. |
| `subscription` | Site | Subscription data sent in boot. |
| `fc_team`, `pulse_api_key`, `pulse_host`, `pulse_force_enabled`, `pulse_identify_endpoint`, `pulse_ingest_endpoint`, `pulse_alias_endpoint` | Site | Telemetry client. |

## Pitfalls

- `frappe.conf` is cached for up to 60 seconds in long-running processes. Restart workers after you change Redis or database keys.
- A key in `site_config.json` hides the same key in `common_site_config.json`.
- The `auth_method` key for the site config email account is not read. The code passes `("auth_method")`, a string, not a tuple, so it looks up each character.

## Related

- [Configuration](../../developer-guide/operations/configuration.md)
- [CLI Reference](cli.md)
- [Deployment](../../developer-guide/operations/deployment.md)
- [Security](../../developer-guide/security/security.md)
- [Background Jobs](../../developer-guide/runtime/background-jobs.md)
