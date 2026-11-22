---
title: Installation
description: Install the bench CLI, create a bench with Frappe version 16, create a site and start the development server.
version: v16
sources:
  - pyproject.toml
  - package.json
  - frappe/commands/site.py
  - frappe/commands/utils.py
  - frappe/database/mariadb/setup_db.py
  - frappe/realtime/config.py
  - frappe/realtime/server.py
  - frappe/hooks.py
---

# Installation

Use this page to set up a local development environment for Frappe version 16. You install the `bench` CLI, create a bench folder that holds Frappe, create a site, and start the processes. For production servers, see [Deployment](../developer-guide/operations/deployment.md).

## Requirements

| Software | Version | Source of the requirement |
|---|---|---|
| Python | 3.14 (`>=3.14,<3.15`) | `requires-python` in `pyproject.toml` |
| Node.js | 24 or newer | `engines.node` in `package.json` |
| Yarn | any current version | `bench` runs `yarn install` for app assets (`yarn.lock`) |
| MariaDB | 10.6 to 11.8 | `check_compatible_versions()` in `frappe/database/mariadb/setup_db.py` warns outside this range |
| PostgreSQL | supported, experimental | `bench new-site --db-type postgres` |
| SQLite | supported, experimental | `bench new-site --db-type sqlite` |
| Redis | a current Redis server | Used for cache, job queue and realtime events |
| Git | any current version | `bench new-app` makes a Git repository for the new app |
| `bench` CLI | the `frappe-bench` package | Creates benches, gets apps and runs processes |

MariaDB is the default database. For PostgreSQL and SQLite, `bench new-site` prints a note that the support is experimental.

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

1. Create the bench with the `version-16` branch of Frappe and Python 3.14:

    ```bash
    bench init frappe-bench --frappe-branch version-16 --python python3.14
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
| `--db-type` | `mariadb` | `mariadb`, `postgres` or `sqlite`. |
| `--db-name` | generated | Database name. |
| `--db-host`, `--db-port` | from config | Database server address. |
| `--db-root-username` | `root` | Root user that makes the database and the user. |
| `--db-root-password` | prompt | Root password for MariaDB or PostgreSQL. |
| `--admin-password` | prompt | Password of the `Administrator` user. |
| `--install-app` | none | App to install after the site is made. Use more than once for more apps. |
| `--set-default` | off | Make the site the default site. |
| `--source-sql` | none | Start the database from an SQL file. |
| `--force` | off | Replace a site or database that already exists. |

## Start the bench

1. Start all processes from the `Procfile`:

    ```bash
    bench start
    ```

2. Open `http://sample.localhost:8000` in the browser.
3. Log in as `Administrator` with the admin password.

The desk opens at `/desk`. The old routes `/app` and `/app/*` redirect to `/desk` (`website_redirects` in `frappe/hooks.py`).

!!! note "Version"
    In version 15, the desk is at `/app`. Version 16 moved it to `/desk`.

`bench start` runs these processes:

| Process | Command | Purpose |
|---|---|---|
| `redis_cache` | `redis-server config/redis_cache.conf` | Cache (`redis_cache` in `common_site_config.json`). |
| `redis_queue` | `redis-server config/redis_queue.conf` | Job queue and realtime pub/sub (`redis_queue`). |
| `web` | `bench serve --port 8000` | Python web server for all sites. |
| `socketio` | `bench socketio` | Realtime (Socket.IO) server. |
| `watch` | `bench watch` | Rebuilds JS and CSS when files change. |
| `schedule` | `bench schedule` | Scheduler that enqueues scheduled jobs. |
| `worker` | `bench worker` | Background job worker. |

The site is found from the `Host` header of the request. A name that ends in `.localhost` points to your computer in most browsers. For other names, run `bench --site <site> add-to-hosts`.

## Realtime server backend

Frappe version 16 ships two realtime servers: the Node.js server (`socketio.js`) and a Python Socket.IO server (`frappe.realtime.server`). `bench socketio` reads the `socketio_backend` key in `common_site_config.json`. The value is `node` (default) or `python`. If the value is `python` and the Python server is not available, `bench socketio` falls back to Node.js.

To use the Python server, set the key and restart the bench:

```bash
bench set-config -g socketio_backend python
```

The realtime server listens on `socketio_port` (default `9000`).

## Pitfalls

- Python 3.13 and older do not work. `pyproject.toml` requires Python 3.14.
- A MariaDB version outside 10.6 to 11.8 shows a warning during `bench new-site`. The site can still be made, but the version is not tested.
- `bench new-site` prompts for passwords when you do not give them. Give `--db-root-password` and `--admin-password` in scripts.
- Without `developer_mode`, you cannot make a standard (non-custom) DocType. You get the error "Not in Developer Mode".

## Related

- [Architecture](architecture.md)
- [Quickstart](quickstart.md)
- [Configuration](../developer-guide/operations/configuration.md)
- [CLI reference](../reference/configuration/cli.md)
- [Deployment](../developer-guide/operations/deployment.md)
