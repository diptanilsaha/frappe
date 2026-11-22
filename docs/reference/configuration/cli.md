---
title: CLI Reference
description: Every bench command that the frappe app provides in v15, with its options and whether it needs a site.
version: v15
sources:
  - frappe/utils/bench_helper.py
  - frappe/commands/__init__.py
  - frappe/commands/site.py
  - frappe/commands/scheduler.py
  - frappe/commands/utils.py
  - frappe/commands/translate.py
  - frappe/commands/gettext.py
  - frappe/commands/redis_utils.py
  - frappe/exceptions.py
---

# CLI Reference

This page lists every `bench` command that the frappe app adds in v15.
The list and the options come from the click definitions in `frappe/commands/`.
Commands of the `bench` tool itself (`init`, `get-app`, `new-app`, `start`, `setup ...`) are not on this page.
Run all commands from the bench root (`frappe-bench/`).

## How commands find the site

`bench` loads `frappe.commands.commands` and the `commands` list of every other app (`<app>/commands/__init__.py` or `<app>/commands.py`).
It wraps them in one group that takes these options before the command name:

| Option | Description |
|---|---|
| `--site <site>` | Site to run on. `--site all` runs on every site, for commands that loop over sites. |
| `--verbose` | Verbose output. |
| `--force` | Force flag. Some commands read it (for example `run-patch`, `reload-doc`). |
| `--profile` | Run the command under `cProfile` and print the top 100 lines. |

When `--site` is not given, `bench` uses the `FRAPPE_SITE` environment variable, then `default_site` from `common_site_config.json` (set with `bench use <site>`).
A `sites/currentsite.txt` file is not supported. `bench` only prints a warning when it finds one.

```bash
bench --site frappe.localhost migrate
bench --site all clear-cache
bench --site frappe.localhost execute frappe.client.get_count --args '["ToDo"]'
```

In the tables below, "Needs site" means:

- **Yes**: the command runs on the site from `--site` (or the default site). Without one, it fails with "Please specify --site sitename".
- **Optional**: the command also works without a site, or takes `--site` after the command name.
- **No**: the command does not use a site.

Run `bench <command> --help` to see the same options on your bench.

!!! note "Version"
    v15 has 91 frappe commands. These develop commands do not exist in v15: `enable-app`, `disable-app`, `list-sites`, `bypass-patch`, `sync-desktop-icons`, `setup-chrome`, `run-microbenchmarks`. In v15 the test commands are in `frappe/commands/utils.py`.

## Sites and apps

Defined in `frappe/commands/site.py`.

| Command | Needs site | Description |
|---|---|---|
| [`add-system-manager`](#add-system-manager) | Yes | Add a new system manager to a site. |
| [`add-user`](#add-user) | Yes | Add user to a site. |
| [`add-database-index`](#add-database-index) | Yes | Adds a new DB index and creates a property setter to persist it. |
| [`describe-database-table`](#describe-database-table) | Yes | Show the schema, indexes and row count of a DocType table as JSON. With `--column`, also show distinct value counts. |
| [`backup`](#backup) | Yes | Make a database backup of the site. Optionally include files and config. |
| [`drop-site`](#drop-site) | No (`<site>` argument) | Remove a site from database and filesystem. |
| [`install-app`](#install-app) | Yes | Install a new app to site, supports multiple apps. |
| [`list-apps`](#list-apps) | Yes | List apps in site. |
| [`migrate`](#migrate) | Yes | Run patches, sync schema and rebuild files/translations. |
| [`migrate-to`](#migrate-to) | No | Move a site to a hosting provider (Frappe Cloud). Interactive. |
| [`new-site`](#new-site) | No (`<site>` argument) | Create a new site. |
| [`reinstall`](#reinstall) | Yes | Reinstall site ie. wipe all data and start over. |
| [`reload-doc`](#reload-doc) | Yes | Reload schema for a DocType. |
| [`reload-doctype`](#reload-doctype) | Yes | Reload schema for a DocType. |
| [`remove-from-installed-apps`](#remove-from-installed-apps) | Yes | Remove app from site's installed-apps list. |
| [`restore`](#restore) | Yes | Restore site database from an sql file. |
| [`run-patch`](#run-patch) | Yes | Run a particular patch. |
| [`set-password`](#set-password) | Yes | Set password for a user on a site. |
| [`set-admin-password`](#set-admin-password) | Yes | Set Administrator password for a site. |
| [`uninstall-app`](#uninstall-app) | Yes | Remove app and linked modules from site. |
| [`disable-user`](#disable-user) | Yes | Disable a user account on site. |
| [`use`](#use) | No (`<site>` argument) | Set a default site. |
| [`set-last-active-for-user`](#set-last-active-for-user) | Yes | Set users last active date to current datetime. |
| [`publish-realtime`](#publish-realtime) | Yes | Publish realtime event from bench. |
| [`browse`](#browse) | Optional | Open the site in a web browser, logged in as a user. |
| [`start-recording`](#start-recording) | Yes | Start Frappe Recorder. |
| [`stop-recording`](#stop-recording) | Yes | Stop Frappe Recorder. |
| [`add-to-hosts`](#add-to-hosts) | Yes | Add the site name to `/etc/hosts` (uses sudo). |
| [`ngrok`](#ngrok) | Yes | Start a ngrok tunnel to your local development server. |
| [`build-search-index`](#build-search-index) | Yes | Build the website search index for all routes of the site. |
| [`partial-restore`](#partial-restore) | Yes | Restore some tables from a partial backup file (made with `backup --only`). |
| [`trim-tables`](#trim-tables) | Yes | Remove columns from tables where fields are deleted from doctypes. |
| [`trim-database`](#trim-database) | Yes | Remove database tables for deleted DocTypes. |
| [`clear-log-table`](#clear-log-table) | Yes | Shrink a large log table. Recent rows are copied to a new table, which replaces the old one. |

### add-system-manager

```bash
bench --site <site> add-system-manager [OPTIONS] EMAIL
```

| Option | Default | Description |
|---|---|---|
| `--first-name` |  | First name of the user. |
| `--last-name` |  | Last name of the user. |
| `--password` |  | Password of the user. |
| `--send-welcome-email` | off | Send the welcome email. |

### add-user

```bash
bench --site <site> add-user [OPTIONS] EMAIL
```

| Option | Default | Description |
|---|---|---|
| `--first-name` |  | First name of the user. |
| `--last-name` |  | Last name of the user. |
| `--password` |  | Password of the user. |
| `--user-type` |  | User type, for example `System User`. |
| `--add-role` |  | Role to add. Repeat for many values. |
| `--send-welcome-email` | off | Send the welcome email. |

### add-database-index

```bash
bench --site <site> add-database-index [OPTIONS]
```

| Option | Default | Description |
|---|---|---|
| `--doctype` |  | DocType on which index needs to be added. |
| `--column` |  | Column to index. Multiple columns will create multi-column index in given order. To create a multiple, single column index, execute the command multiple times. Repeat for many values. |

### describe-database-table

```bash
bench --site <site> describe-database-table [OPTIONS]
```

| Option | Default | Description |
|---|---|---|
| `--doctype` |  | DocType to describe. |
| `--column` |  | Explicitly fetch accurate cardinality from table data. This can be slow on large tables. Repeat for many values. |

### backup

```bash
bench --site <site> backup [OPTIONS]
```

| Option | Default | Description |
|---|---|---|
| `--with-files` | off | Take backup with files. |
| `--include, --only, -i` |  | Specify the DocTypes to backup separated by commas. |
| `--exclude, -e` |  | Specify the DocTypes to not backup separated by commas. |
| `--backup-path` |  | Set path for saving all the files in this operation. |
| `--backup-path-db` |  | Set path for saving database file. |
| `--backup-path-files` |  | Set path for saving public file. |
| `--backup-path-private-files` |  | Set path for saving private file. |
| `--backup-path-conf` |  | Set path for saving config file. |
| `--ignore-backup-conf` | off | Ignore excludes/includes set in config. |
| `--verbose` | off | Add verbosity. |
| `--compress` | off | Compress private and public files. |
| `--old-backup-metadata` | off | Use older backup metadata. |

### drop-site

```bash
bench drop-site [OPTIONS] SITE
```

| Option | Default | Description |
|---|---|---|
| `--db-root-username, --mariadb-root-username, --root-login` |  | Root username for MariaDB or PostgreSQL, Default is "root". |
| `--db-root-password, --mariadb-root-password, --root-password` |  | Root password for MariaDB or PostgreSQL. |
| `--archived-sites-path` |  | Folder for the archived site. Default `<bench>/archived/sites`. |
| `--no-backup` | off | Do not take a backup before the drop. |
| `--force` | off | Force drop-site even if an error is encountered. |

### install-app

```bash
bench --site <site> install-app [OPTIONS] [APPS...]
```

| Option | Default | Description |
|---|---|---|
| `--force` | off | Install again even when the app is installed. |

### list-apps

```bash
bench --site <site> list-apps [OPTIONS]
```

| Option | Default | Description |
|---|---|---|
| `--format, -f` | `text` | One of `text`, `json`. |

### migrate

```bash
bench --site <site> migrate [OPTIONS]
```

| Option | Default | Description |
|---|---|---|
| `--skip-failing` | off | Skip patches that fail to run. |
| `--skip-search-index` | off | Skip search indexing for web documents. |

### migrate-to

```bash
bench migrate-to
```

### new-site

```bash
bench new-site [OPTIONS] SITE
```

| Option | Default | Description |
|---|---|---|
| `--db-name` |  | Database name. |
| `--db-password` |  | Database password. |
| `--db-type` | `mariadb` | Optional "postgres" or "mariadb". Default is "mariadb". One of `mariadb`, `postgres`. |
| `--db-host` |  | Database Host. |
| `--db-port` |  | Database Port. |
| `--db-root-username, --mariadb-root-username` |  | Root username for MariaDB or PostgreSQL, Default is "root". |
| `--db-root-password, --mariadb-root-password` |  | Root password for MariaDB or PostgreSQL. |
| `--db-socket, --mariadb-db-socket` |  | Database socket for MariaDB or folder containing database socket for PostgreSQL. |
| `--no-mariadb-socket` | off | DEPRECATED: Set MariaDB host to % and use TCP/IP Socket instead of using the UNIX Socket. |
| `--mariadb-user-host-login-scope` |  | Set the mariadb host for the user login scope if you do not want to use the current host as login scope which typically is ''@'localhost' - may be used when initializing a user on a remote host. See the mariadb docs on account names for more info. |
| `--admin-password` |  | Administrator password for new site. |
| `--verbose` | off | Verbose. |
| `--force` | off | Force restore if site/database already exists. |
| `--source-sql, --source_sql` |  | Initiate database with a SQL file. |
| `--install-app` |  | Install app after installation. Repeat for many values. |
| `--set-default` | off | Set the new site as default site. |
| `--setup-db, --no-setup-db` | off | Create the user and database in MariaDB or PostgreSQL. When false, only bootstrap. |

### reinstall

```bash
bench --site <site> reinstall [OPTIONS]
```

| Option | Default | Description |
|---|---|---|
| `--admin-password` |  | Administrator Password for reinstalled site. |
| `--db-root-username, --mariadb-root-username` |  | Root username for MariaDB or PostgreSQL, Default is "root". |
| `--db-root-password, --mariadb-root-password` |  | Root password for MariaDB or PostgreSQL. |
| `--yes` | off | Pass --yes to skip confirmation. |

### reload-doc

```bash
bench --site <site> reload-doc MODULE DOCTYPE DOCNAME
```

### reload-doctype

```bash
bench --site <site> reload-doctype DOCTYPE
```

### remove-from-installed-apps

```bash
bench --site <site> remove-from-installed-apps APP
```

### restore

```bash
bench --site <site> restore [OPTIONS] SQL_FILE_PATH
```

| Option | Default | Description |
|---|---|---|
| `--db-root-username, --mariadb-root-username` |  | Root username for MariaDB or PostgreSQL, Default is "root". |
| `--db-root-password, --mariadb-root-password` |  | Root password for MariaDB or PostgreSQL. |
| `--db-name` |  | Database name for site in case it is a new one. |
| `--admin-password` |  | Administrator password for new site. |
| `--install-app` |  | Install app after installation. Repeat for many values. |
| `--with-public-files` |  | Restores the public files of the site, given path to its tar file. |
| `--with-private-files` |  | Restores the private files of the site, given path to its tar file. |
| `--force` | off | Ignore the validations and downgrade warnings. This action is not recommended. |
| `--encryption-key` |  | Backup encryption key. |

### run-patch

```bash
bench --site <site> run-patch [OPTIONS] MODULE
```

| Option | Default | Description |
|---|---|---|
| `--force` | off | Run the patch even when it is in the Patch Log. |

### set-password

```bash
bench --site <site> set-password [OPTIONS] USER [PASSWORD]
```

| Option | Default | Description |
|---|---|---|
| `--logout-all-sessions` | off | Log out from all sessions. |

### set-admin-password

```bash
bench --site <site> set-admin-password [OPTIONS] [ADMIN_PASSWORD]
```

| Option | Default | Description |
|---|---|---|
| `--logout-all-sessions` | off | Log out from all sessions. |

### uninstall-app

```bash
bench --site <site> uninstall-app [OPTIONS] APP
```

| Option | Default | Description |
|---|---|---|
| `--yes, -y` | off | To bypass confirmation prompt for uninstalling the app. |
| `--dry-run` | off | List all doctypes that will be deleted. |
| `--no-backup` | off | Do not backup the site. |
| `--force` | off | Force remove app from site. |

### disable-user

```bash
bench --site <site> disable-user EMAIL
```

### use

```bash
bench use SITE
```

### set-last-active-for-user

```bash
bench --site <site> set-last-active-for-user [OPTIONS]
```

| Option | Default | Description |
|---|---|---|
| `--user` |  | Setup last active date for user. |

### publish-realtime

```bash
bench --site <site> publish-realtime [OPTIONS] EVENT
```

| Option | Default | Description |
|---|---|---|
| `--message` |  | Message payload. |
| `--room` |  | Room name. |
| `--user` |  | Send to this user. |
| `--doctype` |  | DocType of the document room. |
| `--docname` |  | Name of the document. |
| `--after-commit` |  | Publish after the database transaction commits. |

### browse

```bash
bench browse [OPTIONS] [SITE]
```

| Option | Default | Description |
|---|---|---|
| `--user` |  | Login as user. |
| `--session-end` |  | Session end (in ISO8601 format and timezone-aware - 2025-01-24T12:26:29.200853+00:00). |
| `--user-for-audit` |  | The user to mention in audit trail. |

### start-recording

```bash
bench --site <site> start-recording
```

### stop-recording

```bash
bench --site <site> stop-recording
```

### add-to-hosts

```bash
bench --site <site> add-to-hosts
```

### ngrok

```bash
bench --site <site> ngrok [OPTIONS]
```

| Option | Default | Description |
|---|---|---|
| `--bind-tls` | off | Returns a reference to the https tunnel. |
| `--use-default-authtoken` | off | Use the auth token present in ngrok's config. |

### build-search-index

```bash
bench --site <site> build-search-index
```

### partial-restore

```bash
bench --site <site> partial-restore [OPTIONS] SQL_FILE_PATH
```

| Option | Default | Description |
|---|---|---|
| `--verbose, -v` | off | Verbose output. |
| `--encryption-key` |  | Backup encryption key. |

### trim-tables

```bash
bench --site <site> trim-tables [OPTIONS]
```

| Option | Default | Description |
|---|---|---|
| `--dry-run` | off | Show what would be deleted. |
| `--format, -f` | `table` | Output format. One of `json`, `table`. |
| `--no-backup` | off | Do not backup the site. |

### trim-database

```bash
bench --site <site> trim-database [OPTIONS]
```

| Option | Default | Description |
|---|---|---|
| `--dry-run` | off | Show what would be deleted. |
| `--format, -f` | `text` | Output format. One of `json`, `text`. |
| `--no-backup` | off | Do not backup the site. |
| `--yes, -y` | off | To bypass confirmation prompt. |

### clear-log-table

```bash
bench --site <site> clear-log-table [OPTIONS]
```

| Option | Default | Description |
|---|---|---|
| `--doctype` |  | Log DocType. Required. |
| `--days` |  | Keep records for days. |
| `--no-backup` | off | Do not backup the table. |

## Scheduler and workers

Defined in `frappe/commands/scheduler.py`.

| Command | Needs site | Description |
|---|---|---|
| [`disable-scheduler`](#disable-scheduler) | Yes | Disable scheduler. |
| [`doctor`](#doctor) | Optional | Show diagnostic info about the scheduler and background workers. |
| [`enable-scheduler`](#enable-scheduler) | Yes | Enable scheduler. |
| [`purge-jobs`](#purge-jobs) | Optional | Delete pending jobs from the queues. Without `--event`, it deletes all jobs of the site. |
| [`ready-for-migration`](#ready-for-migration) | Yes | Exit with code 1 when the site has pending background jobs. Use it before a migrate. |
| [`scheduler`](#scheduler) | Yes | Control the scheduler. `STATE` is one of `pause`, `resume`, `disable`, `enable`, `status`. |
| [`set-maintenance-mode`](#set-maintenance-mode) | Yes | Put the site in maintenance mode, or take it out. `STATE` is `on` or `off`. |
| [`show-pending-jobs`](#show-pending-jobs) | Yes | Get diagnostic info about background jobs. |
| [`schedule`](#schedule) | No | Start scheduler process which is responsible for enqueueing the scheduled job types. |
| [`worker`](#worker) | No | Start a background worker. |
| [`worker-pool`](#worker-pool) | No | Start a pool of background workers. |
| [`trigger-scheduler-event`](#trigger-scheduler-event) | Yes | Trigger a scheduler event. |

### disable-scheduler

```bash
bench --site <site> disable-scheduler
```

### doctor

```bash
bench doctor [OPTIONS]
```

| Option | Default | Description |
|---|---|---|
| `--site` |  | Site name. Use it instead of the global `--site` option. |

### enable-scheduler

```bash
bench --site <site> enable-scheduler
```

### purge-jobs

```bash
bench purge-jobs [OPTIONS]
```

| Option | Default | Description |
|---|---|---|
| `--site` |  | Site name. Use it instead of the global `--site` option. |
| `--queue` |  | Queue name, for example `short`, `default` or `long`. |
| `--event` |  | Scheduler event, for example `daily` or `hourly_long`. |

### ready-for-migration

```bash
bench --site <site> ready-for-migration [OPTIONS]
```

| Option | Default | Description |
|---|---|---|
| `--site` |  | Site name. Use it instead of the global `--site` option. |

### scheduler

```bash
bench --site <site> scheduler [OPTIONS] STATE
```

| Option | Default | Description |
|---|---|---|
| `--site` |  | Site name. Use it instead of the global `--site` option. |
| `--format, -f` | `text` | Output format. One of `json`, `text`. |
| `--verbose, -v` | off | Verbose output. |

### set-maintenance-mode

```bash
bench --site <site> set-maintenance-mode [OPTIONS] STATE
```

| Option | Default | Description |
|---|---|---|
| `--site` |  | Site name. Use it instead of the global `--site` option. |

### show-pending-jobs

```bash
bench --site <site> show-pending-jobs [OPTIONS]
```

| Option | Default | Description |
|---|---|---|
| `--site` |  | Site name. Use it instead of the global `--site` option. |

### schedule

```bash
bench schedule
```

### worker

```bash
bench worker [OPTIONS]
```

| Option | Default | Description |
|---|---|---|
| `--queue` |  | Queue to consume from. Multiple queues can be specified using comma-separated string. If not specified all queues are consumed. |
| `--quiet` | off | Hide Log Outputs. |
| `-u, --rq-username` |  | Redis ACL user. |
| `-p, --rq-password` |  | Redis ACL user password. |
| `--burst` | off | Run Worker in Burst mode. |
| `--strategy` |  | Dequeuing strategy to use. One of `round_robin`, `random`. |

### worker-pool

```bash
bench worker-pool [OPTIONS]
```

| Option | Default | Description |
|---|---|---|
| `--queue` |  | Queue to consume from. Multiple queues can be specified using comma-separated string. If not specified all queues are consumed. |
| `--num-workers` | `2` | Number of workers to spawn in pool. |
| `--quiet` | off | Hide Log Outputs. |
| `--burst` | off | Run Worker in Burst mode. |

### trigger-scheduler-event

```bash
bench --site <site> trigger-scheduler-event EVENT
```

## Utilities and testing

Defined in `frappe/commands/utils.py`.

| Command | Needs site | Description |
|---|---|---|
| [`build`](#build) | No | Compile JS and CSS source files. |
| [`clear-cache`](#clear-cache) | Yes | Clear cache, doctype cache and defaults. |
| [`clear-website-cache`](#clear-website-cache) | Yes | Clear website cache. |
| [`db-console`](#db-console) | Yes | Enter into the Database console for given site. |
| [`transform-database`](#transform-database) | Yes | Change tables' internal settings changing engine and row formats. |
| [`jupyter`](#jupyter) | Yes | Start an interactive jupyter notebook. |
| [`console`](#console) | Yes | Start ipython console for a site. |
| [`destroy-all-sessions`](#destroy-all-sessions) | Yes | Clear sessions of all users (logs them out). |
| [`execute`](#execute) | Yes | Run a Python function on the site. Prints the return value. |
| [`export-csv`](#export-csv) | Yes | Export data import template with data for DocType. |
| [`export-doc`](#export-doc) | Yes | Export a single document to csv. |
| [`export-fixtures`](#export-fixtures) | Yes | Export fixtures. |
| [`export-json`](#export-json) | Yes | Export doclist as json to the given path, use '-' as name for Singles. |
| [`version`](#version) | No | Show the versions of all the installed apps. |
| [`data-import`](#data-import) | Yes | Import documents in bulk from CSV or XLSX using data import. |
| [`import-doc`](#import-doc) | Yes | Import (insert/update) doclist. If the argument is a directory, all files ending with .json are imported. |
| [`make-app`](#make-app) | No | Creates a boilerplate app. |
| [`create-patch`](#create-patch) | No | Make a new patch file and add it to `patches.txt`. Interactive. |
| [`mariadb`](#mariadb) | Yes | Enter into mariadb console for a given site. |
| [`postgres`](#postgres) | Yes | Enter into postgres console for a given site. |
| [`request`](#request) | Yes | Run a request as Administrator, from `--args` or a JSON file. |
| [`reset-perms`](#reset-perms) | Yes | Reset permissions for all doctypes. |
| [`run-tests`](#run-tests) | Yes | Run python unit-tests. |
| [`run-ui-tests`](#run-ui-tests) | Yes | Run UI tests with Playwright, or with Cypress for apps that still have a cypress config. |
| [`serve`](#serve) | Optional | Start development web server. |
| [`set-config`](#set-config) | Optional | Insert/Update a value in site_config.json. |
| [`show-config`](#show-config) | Yes | Print the merged site config of the site as a table or JSON. |
| [`watch`](#watch) | No | Watch and compile JS and CSS files as and when they change. |
| [`bulk-rename`](#bulk-rename) | Yes | Rename multiple records via CSV file. |
| [`add-to-email-queue`](#add-to-email-queue) | Yes | Add an email to the Email Queue. |
| [`rebuild-global-search`](#rebuild-global-search) | Yes | Rebuild the global search table. |
| [`run-parallel-tests`](#run-parallel-tests) | Yes | Split the tests of an app across CI builds and run one part. |

### build

```bash
bench build [OPTIONS]
```

| Option | Default | Description |
|---|---|---|
| `--app` |  | Build assets for app. |
| `--apps` |  | Build assets for specific apps. |
| `--hard-link` | off | Copy the files instead of symlinking. |
| `--production` | off | Build assets in production mode. |
| `--verbose` | off | Verbose. |
| `--force` | off | Force build assets instead of downloading available. |
| `--save-metafiles` | off | Saves esbuild metafiles for built assets. Useful for analyzing bundle size. More info: https://esbuild.github.io/api/#metafile. |

### clear-cache

```bash
bench --site <site> clear-cache
```

### clear-website-cache

```bash
bench --site <site> clear-website-cache
```

### db-console

```bash
bench --site <site> db-console [EXTRA_ARGS...]
```

### transform-database

```bash
bench --site <site> transform-database [OPTIONS]
```

| Option | Default | Description |
|---|---|---|
| `--table` |  | Comma separated name of tables to convert. To convert all tables, pass 'all'. Required. |
| `--engine` |  | Choice of storage engine for said table(s). One of `InnoDB`, `MyISAM`. |
| `--row_format` |  | Set ROW_FORMAT parameter for said table(s). One of `DYNAMIC`, `COMPACT`, `REDUNDANT`, `COMPRESSED`. |
| `--failfast` | off | Exit on first failure occurred. |

### jupyter

```bash
bench --site <site> jupyter
```

### console

```bash
bench --site <site> console [OPTIONS]
```

| Option | Default | Description |
|---|---|---|
| `--autoreload` | off | Reload changes to code automatically. |

### destroy-all-sessions

```bash
bench --site <site> destroy-all-sessions [OPTIONS]
```

| Option | Default | Description |
|---|---|---|
| `--reason` |  | Reason saved with the logout. |

### execute

```bash
bench --site <site> execute [OPTIONS] METHOD
```

| Option | Default | Description |
|---|---|---|
| `--args` |  | Positional arguments as a Python literal, for example `'["ToDo"]'`. |
| `--kwargs` |  | Keyword arguments as a Python dict literal. |
| `--profile` | off | Profile the call and print the stats. |

### export-csv

```bash
bench --site <site> export-csv DOCTYPE PATH
```

### export-doc

```bash
bench --site <site> export-doc DOCTYPE DOCNAME
```

### export-fixtures

```bash
bench --site <site> export-fixtures [OPTIONS]
```

| Option | Default | Description |
|---|---|---|
| `--app` |  | Export fixtures of a specific app. |

### export-json

```bash
bench --site <site> export-json [OPTIONS] DOCTYPE PATH
```

| Option | Default | Description |
|---|---|---|
| `--name` |  | Export only one document. |

### version

```bash
bench version [OPTIONS]
```

| Option | Default | Description |
|---|---|---|
| `-f, --format` | `legacy` | Output format. One of `plain`, `table`, `json`, `legacy`. |

### data-import

```bash
bench --site <site> data-import [OPTIONS]
```

| Option | Default | Description |
|---|---|---|
| `--file` |  | Path to import file (.csv, .xlsx). Consider that relative paths will resolve from 'sites' directory. Required. |
| `--doctype` |  | Required. |
| `--type` | `Insert` | Insert New Records or Update Existing Records. One of `Insert`, `Update`. |
| `--submit-after-import` | off | Submit document after importing it. |
| `--mute-emails` | off | Mute emails during import. |

### import-doc

```bash
bench --site <site> import-doc PATH
```

### make-app

```bash
bench make-app [OPTIONS] DESTINATION APP_NAME
```

| Option | Default | Description |
|---|---|---|
| `--no-git` | off | Do not initialize git repository for the app. |

### create-patch

```bash
bench create-patch
```

### mariadb

```bash
bench --site <site> mariadb [EXTRA_ARGS...]
```

### postgres

```bash
bench --site <site> postgres [EXTRA_ARGS...]
```

### request

```bash
bench --site <site> request [OPTIONS]
```

| Option | Default | Description |
|---|---|---|
| `--args` |  | arguments like `?cmd=test&key=value` or `/api/request/method?..`. |
| `--path` |  | path to request JSON. |

### reset-perms

```bash
bench --site <site> reset-perms
```

### run-tests

```bash
bench --site <site> run-tests [OPTIONS]
```

| Option | Default | Description |
|---|---|---|
| `--app` |  | For App. |
| `--doctype` |  | For DocType. |
| `--module-def` |  | For all Doctypes in Module Def. |
| `--case` |  | Select particular TestCase. |
| `--doctype-list-path` |  | Path to .txt file for list of doctypes. Example erpnext/tests/server/agriculture.txt. |
| `--test` |  | Specific test. Repeat for many values. |
| `--module` |  | Run tests in a module. |
| `--profile` | off | Profile the test run. |
| `--coverage` | off | Record code coverage. |
| `--skip-test-records` | off | Do not create test records. |
| `--skip-before-tests` | off | Do not run before tests hook. |
| `--junit-xml-output` |  | Destination file path for junit xml report. |
| `--failfast` | off | Stop the test run on the first error or failure. |

### run-ui-tests

```bash
bench --site <site> run-ui-tests [OPTIONS] APP [RUNNERARGS...]
```

| Option | Default | Description |
|---|---|---|
| `--headless` | off | Run UI Test in headless mode. |
| `--browser` |  | Browser to run tests in: chromium, firefox or webkit. |
| `--parallel-site` |  | Additional site to run tests on in parallel. Each site gets its own worker. Repeat for many values. |
| `--spec` |  | Spec file to run. |

### serve

```bash
bench serve [OPTIONS]
```

| Option | Default | Description |
|---|---|---|
| `--port` | `8000` | Port to listen on. |
| `--profile` | off | Profile each request. |
| `--proxy` | off | The development server may be run behind a proxy, e.g. ngrok / localtunnel. |
| `--noreload` | off | Do not reload on code change. |
| `--nothreading` | off | Handle one request at a time. |
| `--with-coverage` | off | Record code coverage. |

### set-config

```bash
bench --site <site> set-config [OPTIONS] KEY VALUE
```

| Option | Default | Description |
|---|---|---|
| `-g, --global` | off | Set value in bench config. |
| `-p, --parse` | off | Evaluate as Python Object. |

### show-config

```bash
bench --site <site> show-config [OPTIONS]
```

| Option | Default | Description |
|---|---|---|
| `--format, -f` | `text` | One of `text`, `json`. |

### watch

```bash
bench watch [OPTIONS]
```

| Option | Default | Description |
|---|---|---|
| `--apps` |  | Watch assets for specific apps. |

### bulk-rename

```bash
bench --site <site> bulk-rename DOCTYPE PATH
```

### add-to-email-queue

```bash
bench --site <site> add-to-email-queue EMAIL_PATH
```

### rebuild-global-search

```bash
bench --site <site> rebuild-global-search [OPTIONS]
```

| Option | Default | Description |
|---|---|---|
| `--static-pages` | off | Rebuild global search for static pages. |

### run-parallel-tests

```bash
bench --site <site> run-parallel-tests [OPTIONS]
```

| Option | Default | Description |
|---|---|---|
| `--app` | `frappe` | For App. |
| `--build-number` | `1` | Build number. |
| `--total-builds` | `1` | Total number of builds. |
| `--with-coverage` | off | Build coverage file. |
| `--use-orchestrator` | off | Use orchestrator to run parallel tests. |
| `--dry-run` | off | Do not actually run tests. |

## Translations (CSV)

Defined in `frappe/commands/translate.py`.

| Command | Needs site | Description |
|---|---|---|
| [`build-message-files`](#build-message-files) | Yes | Build message files for translation. |
| [`get-untranslated`](#get-untranslated) | Yes | Get untranslated strings for language. |
| [`import-translations`](#import-translations) | Yes | Update translated strings. |
| [`new-language`](#new-language) | Yes | Create lang-code.csv for given app. |
| [`update-translations`](#update-translations) | Yes | Update translated strings. |
| [`migrate-translations`](#migrate-translations) | Yes | Migrate target-app-specific translations from source-app to target-app. |

### build-message-files

```bash
bench --site <site> build-message-files
```

### get-untranslated

```bash
bench --site <site> get-untranslated [OPTIONS] LANG UNTRANSLATED_FILE
```

| Option | Default | Description |
|---|---|---|
| `--app` | `_ALL_APPS` | App to read. Default: all apps. |
| `--all` | off | Get all message strings. |

### import-translations

```bash
bench --site <site> import-translations LANG PATH
```

### new-language

```bash
bench --site <site> new-language LANG_CODE APP
```

### update-translations

```bash
bench --site <site> update-translations [OPTIONS] LANG UNTRANSLATED_FILE TRANSLATED_FILE
```

| Option | Default | Description |
|---|---|---|
| `--app` | `_ALL_APPS` | App to update. Default: all apps. |

### migrate-translations

```bash
bench --site <site> migrate-translations SOURCE_APP TARGET_APP
```

## Translations (gettext)

Defined in `frappe/commands/gettext.py`.

| Command | Needs site | Description |
|---|---|---|
| [`generate-pot-file`](#generate-pot-file) | Optional | Translation: generate POT file. |
| [`compile-po-to-mo`](#compile-po-to-mo) | Optional | Translation: compile PO files to MO files. |
| [`migrate-csv-to-po`](#migrate-csv-to-po) | Optional | Translation: migrate from CSV files (old) to PO files (new). |
| [`update-po-files`](#update-po-files) | Optional | Translation: sync PO files with POT file. |
| [`create-po-file`](#create-po-file) | Optional | Translation: create a new PO file for a locale. |
| [`update-csv-from-po`](#update-csv-from-po) | No | Add missing translations from PO file to CSV file. |

### generate-pot-file

```bash
bench generate-pot-file [OPTIONS]
```

| Option | Default | Description |
|---|---|---|
| `--app` |  | Only generate for this app. eg: frappe. |

### compile-po-to-mo

```bash
bench compile-po-to-mo [OPTIONS]
```

| Option | Default | Description |
|---|---|---|
| `--app` |  | Only compile for this app. eg: frappe. |
| `--force` | off | Force compile even if there are no changes to PO files. |
| `--locale` |  | Compile translations only for this locale. eg: de. |

### migrate-csv-to-po

```bash
bench migrate-csv-to-po [OPTIONS]
```

| Option | Default | Description |
|---|---|---|
| `--app` |  | Only migrate for this app. eg: frappe. |
| `--locale` |  | Compile translations only for this locale. eg: de. |

### update-po-files

```bash
bench update-po-files [OPTIONS]
```

| Option | Default | Description |
|---|---|---|
| `--app` |  | Only update for this app. eg: frappe. |
| `--locale` |  | Update PO files only for this locale. eg: de. |

### create-po-file

```bash
bench create-po-file [OPTIONS] LOCALE
```

| Option | Default | Description |
|---|---|---|
| `--app` |  | Only create for this app. eg: frappe. |

### update-csv-from-po

```bash
bench update-csv-from-po [OPTIONS] APP
```

| Option | Default | Description |
|---|---|---|
| `--locale` |  | Update CSV file only for this locale. eg: de. |

## Redis

Defined in `frappe/commands/redis_utils.py`.

| Command | Needs site | Description |
|---|---|---|
| [`create-rq-users`](#create-rq-users) | No | Create Redis Queue users and add to acl and app configs. |

### create-rq-users

```bash
bench create-rq-users [OPTIONS]
```

| Option | Default | Description |
|---|---|---|
| `--set-admin-password` | off | Set new Redis admin(default user) password. |
| `--use-rq-auth` | off | Enable Redis authentication for sites. |

## Pitfalls

- Put global options before the command: `bench --site s1 migrate`, not `bench migrate --site s1`. Only commands that define their own `--site` (for example `doctor`, `show-pending-jobs`, `ready-for-migration`, `purge-jobs`) accept it after the name.
- `bench execute` evaluates `--args` and `--kwargs` as Python literals. Quote them for the shell.
- `restore`, `reinstall`, `drop-site` and `uninstall-app` delete data. Take a backup first.

## Related

- [Installation](../../getting-started/installation.md)
- [Apps](../../developer-guide/building-apps/apps.md)
- [Configuration](../../developer-guide/operations/configuration.md)
- [Site Config Reference](site-config.md)
- [Testing](../../developer-guide/testing-and-performance/testing.md)
- [Deployment](../../developer-guide/operations/deployment.md)
- [Backups](../../user-guide/system-administration/backups.md)
