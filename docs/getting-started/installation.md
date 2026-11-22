---
title: Installation
description: Install the bench CLI, create a bench with Frappe v15, create a site and start the development server.
version: v15
sources:
  - pyproject.toml
  - package.json
  - frappe/commands/site.py
  - frappe/database/mariadb/setup_db.py
  - frappe/hooks.py
  - node_utils.js
  - realtime/index.js
---

# Installation

Use this page to set up a local development environment for Frappe v15 (branch `version-15`). You install the `bench` CLI, create a bench folder that holds Frappe, create a site, and start the processes. For production servers, see [Deployment](../developer-guide/operations/deployment.md).

## Requirements

| Software | Version | Source of the requirement |
|---|---|---|
| Python | 3.10 to 3.14 (`>=3.10,<3.15`) | `requires-python` in `pyproject.toml` |
| Node.js | 18 or newer | `engines.node` in `package.json` |
| Yarn | any current version | `bench` runs `yarn install` for app assets (`yarn.lock`) |
| MariaDB | 10.6 to 10.8 | `check_compatible_versions()` in `frappe/database/mariadb/setup_db.py` warns outside this range |
| PostgreSQL | limited support | `bench new-site --db-type postgres` |
| Redis | a current Redis server | Used for cache, job queue and realtime events |
| Git | any current version | `bench new-app` makes a Git repository for the new app |
| `bench` CLI | the `frappe-bench` package | Creates benches, gets apps and runs processes |

MariaDB is the default database. For PostgreSQL, `bench new-site` prints a note that PostgreSQL support is limited to Frappe v16 and above, and that fixes for earlier versions are not added.

## Install the bench CLI

1. Install the `frappe-bench` package as a tool:

    ```bash
    uv tool install frappe-bench
    ```

2. Make sure the command works:

    ```bash
    bench --version
    ```

## Create a bench

A bench is a folder that holds the apps, a Python virtual environment (`env/`), the sites (`sites/`), the Redis configuration (`config/`) and a `Procfile`.

1. Create the bench with the `version-15` branch of Frappe:

    ```bash
    bench init frappe-bench --frappe-branch version-15 --python python3.12
    ```

2. Go into the bench folder:

    ```bash
    cd frappe-bench
    ```

Useful `bench init` options:

| Option | Description |
|---|---|
| `--frappe-branch` (alias `--version`) | Branch of Frappe to clone. |
| `--python` | Path to the Python executable for `env/`. |
| `--frappe-path` | Clone Frappe from this path or URL. |
| `--skip-assets` | Do not build JS and CSS assets. |
| `--no-procfile` | Do not make a `Procfile`. |
| `--dev` | Turn on developer mode and install development dependencies. |

## Create a site

A site is one database plus one folder in `sites/`. One bench can serve many sites.

1. Create the site. Give the MariaDB root password and a password for `Administrator`:

    ```bash
    bench new-site sample.localhost --db-root-password <root-password> --admin-password <admin-password>
    ```

2. Make the new site the default site of the bench:

    ```bash
    bench use sample.localhost
    ```

3. If you develop apps on this site, turn on developer mode:

    ```bash
    bench --site sample.localhost set-config developer_mode 1
    ```

Common `bench new-site` options:

| Option | Default | Description |
|---|---|---|
| `--db-type` | `mariadb` | `mariadb` or `postgres`. |
| `--db-name` | generated | Database name. |
| `--db-password` | generated | Password of the site database user. |
| `--db-host`, `--db-port` | from config | Database server address. |
| `--db-socket` | `MYSQL_UNIX_PORT` env var | Database socket. |
| `--db-root-username` (alias `--mariadb-root-username`) | `root` | Root user that makes the database and the user. |
| `--db-root-password` (alias `--mariadb-root-password`) | prompt | Root password for MariaDB or PostgreSQL. |
| `--admin-password` | prompt | Password of the `Administrator` user. |
| `--install-app` | none | App to install after the site is made. Use more than once for more apps. |
| `--set-default` | off | Make the site the default site. |
| `--source-sql` | none | Start the database from an SQL file. |
| `--force` | off | Replace a site or database that already exists. |
| `--setup-db/--no-setup-db` | `--setup-db` | With `--no-setup-db`, do not create the database user and database. Only bootstrap. |

## Start the bench

1. Start all processes from the `Procfile`:

    ```bash
    bench start
    ```

2. Open `http://sample.localhost:8000` in the browser.
3. Log in as `Administrator` with the admin password.

The desk opens at `/app`. The route `/desk` redirects to `/app` (`website_redirects` in `frappe/hooks.py`).

!!! note "Version"
    In Frappe v16, the desk moved to `/desk`.

`bench start` runs these processes:

| Process | Command | Purpose |
|---|---|---|
| `redis_cache` | `redis-server config/redis_cache.conf` | Cache (`redis_cache` in `common_site_config.json`). |
| `redis_queue` | `redis-server config/redis_queue.conf` | Job queue and realtime pub/sub (`redis_queue`). |
| `web` | `bench serve --port 8000` | Python web server for all sites. |
| `socketio` | `bench socketio` | Realtime (Socket.IO) server on Node.js. |
| `watch` | `bench watch` | Rebuilds JS and CSS when files change. |
| `schedule` | `bench schedule` | Scheduler that enqueues scheduled jobs. |
| `worker` | `bench worker` | Background job worker. |

The site is found from the `Host` header of the request. A name that ends in `.localhost` points to your computer in most browsers. For other names, run `bench --site <site> add-to-hosts`.

The realtime server is the Node.js server in `realtime/index.js` (started from `socketio.js`). It listens on `socketio_port` (default `9000`, set in `node_utils.js`).

## Pitfalls

- Python 3.9 and older do not work. `pyproject.toml` requires Python 3.10 or newer.
- A MariaDB version below 10.6 or 10.9 and newer shows a warning during `bench new-site`. The site can still be made, but the version is not tested.
- `bench new-site` prompts for passwords when you do not give them. Give `--db-root-password` and `--admin-password` in scripts.
- Without `developer_mode`, you cannot make a standard (non-custom) DocType. You get the error "Not in Developer Mode".
- Do not name a site the same as an app in the bench. `bench new-site` stops with an error.

## Related

- [Architecture](architecture.md)
- [Quickstart](quickstart.md)
- [Configuration](../developer-guide/operations/configuration.md)
- [CLI reference](../reference/configuration/cli.md)
- [Deployment](../developer-guide/operations/deployment.md)
