---
title: Deployment
description: Run Frappe in production with the bench CLI - processes, Procfile, supervisor or systemd, nginx, workers and queues, socketio, maintenance mode and DNS multitenancy.
version: v16
sources:
  - frappe/commands/scheduler.py
  - frappe/commands/utils.py
  - frappe/commands/site.py
  - frappe/app.py
  - frappe/utils/background_jobs.py
  - frappe/realtime/README.md
  - frappe/boot.py
---

# Deployment

A Frappe bench runs as a set of processes: a web server, background workers, a scheduler, a realtime (socketio) server and two Redis servers. In development, `bench start` runs them from the `Procfile`. In production, the bench CLI generates supervisor (or systemd) and nginx configuration. The bench CLI is a separate package (`frappe-bench`). The commands on this page were checked against bench CLI 5.31.1.

## Processes

| Process | Command | Description |
|---|---|---|
| Web | `bench serve` (development), `gunicorn ... frappe.app:application` (production) | Serves desk, website and API requests. |
| Worker | `bench worker --queue <queues>` | Runs background jobs from Redis queues. |
| Scheduler | `bench schedule` | Enqueues Scheduled Job Types at their time. It does not run the jobs. |
| Realtime | `bench socketio` | Socket.IO server for realtime events. |
| Redis cache | `redis-server config/redis_cache.conf` | Cache, locks, rate limit counters, realtime pub/sub. |
| Redis queue | `redis-server config/redis_queue.conf` | Background job queues. |

All processes serve all sites of the bench. The web process finds the site from the `X-Frappe-Site-Name` header or from the host name of the request (`init_request` in `frappe/app.py`).

### Workers

`bench worker` options:

| Option | Description |
|---|---|
| `--queue` | Queues to consume, comma-separated, for example `short,default`. Without it, the worker consumes all queues. |
| `--burst` | Stop when the queues are empty. |
| `--strategy` | `round_robin` or `random` dequeue order. |
| `--quiet` | Hide log output. |
| `-u`, `--rq-username` / `-p`, `--rq-password` | Redis ACL user and password. |

`bench worker-pool --queue <queues> --num-workers 2` starts a pool of workers in one process. The standard queues are `short`, `default` and `long`. Add your own queues in `common_site_config.json` with the `workers` key. See [Background Jobs](../runtime/background-jobs.md).

### Realtime

`bench socketio` reads `socketio_backend` from `common_site_config.json`. The value `node` (default) runs `apps/frappe/socketio.js` with Node. The value `python` runs `python -m frappe.realtime.server` from the bench environment. If the Python server is not available, the command falls back to Node. The desk reads `socketio_port` from the boot info to connect.

## Development: Procfile

`bench init` and `bench setup procfile` write `Procfile` in the bench root. `bench start` runs it with honcho.

```bash
redis_cache: redis-server config/redis_cache.conf
redis_queue: redis-server config/redis_queue.conf
web: bench serve --port 8000
socketio: bench socketio
watch: bench watch
schedule: bench schedule
worker: bench worker 1>> logs/worker.log 2>> logs/worker.error.log
```

Each entry under `workers` in `common_site_config.json` adds a line `worker_<name>: bench worker --queue <name>`.

## Production setup

Run these commands from the bench root as the bench user. They need `sudo`.

1. Turn off developer mode on each site: `bench --site <site> set-config developer_mode 0`.
2. Build assets: `bench build --production`.
3. Run `sudo bench setup production <user>`. `<user>` is the system user that owns the bench.
4. Check that the processes run: `sudo supervisorctl status`.

`bench setup production <user>`:

1. Installs Ansible, fail2ban, nginx and supervisor if they are missing.
2. Writes `config/supervisor.conf` (or systemd units if `restart_systemd_on_update` is set).
3. Writes `config/nginx.conf`.
4. Links `config/supervisor.conf` into the supervisor config folder and `config/nginx.conf` to `/etc/nginx/conf.d/<bench-name>.conf`.
5. Reloads supervisor and nginx.

You cannot set both `restart_supervisor_on_update` and `restart_systemd_on_update`. `bench disable-production` removes the links.

### supervisor

`config/supervisor.conf` has these programs:

| Program | Command |
|---|---|
| `<bench>-frappe-web` | `gunicorn -b 127.0.0.1:<webserver_port> -w <gunicorn_workers> ... frappe.app:application --preload` |
| `<bench>-frappe-schedule` | `bench schedule` |
| `<bench>-frappe-short-worker` | `bench worker --queue short,default` |
| `<bench>-frappe-long-worker` | `bench worker --queue long,default,short` |
| `<bench>-frappe-default-worker` | `bench worker --queue default` (only if multi-queue consumption is off) |
| `<bench>-frappe-<name>-worker` | `bench worker --queue <name>`, one per `workers` entry |
| `<bench>-redis-cache`, `<bench>-redis-queue` | Redis servers |
| `<bench>-node-socketio` | `bench socketio` |

Groups: `<bench>-web` (web and socketio), `<bench>-workers` (scheduler and workers), `<bench>-redis`.

Bench config keys in `common_site_config.json` that change this file:

| Key | Default | Description |
|---|---|---|
| `webserver_port` | `8000` | Port of gunicorn. |
| `gunicorn_workers` | CPU count x 2 + 1 | gunicorn worker processes. |
| `gunicorn_max_requests` | computed | Restart a gunicorn worker after this many requests. |
| `http_timeout` | `120` | gunicorn timeout in seconds. |
| `background_workers` | `1` | Number of processes for each worker program. |
| `workers` | `{}` | Extra queues: `{"<name>": {"timeout": 300, "background_workers": 2}}`. |
| `socketio_backend` | `node` | `node` or `python`. |

After you change a key, run `bench setup supervisor` and `sudo supervisorctl reread && sudo supervisorctl update`.

### nginx

`config/nginx.conf` serves `/assets` and public files from disk. It sends `/socket.io` to the socketio server and other requests to gunicorn. It sets `X-Frappe-Site-Name` on proxied requests. Private files are served through an internal `/protected/` location after Frappe checks permission. Run `bench setup nginx` and `sudo bench setup reload-nginx` after you add a site or a domain.

## Multitenancy and DNS

- **DNS multitenancy** (`bench config dns_multitenant on`): all sites share port 80. nginx routes by host name. The site folder name must match the domain, for example `sites/erp.example.com`.
- **Port-based multitenancy** (default): each site gets its own port. Set it with `bench set-nginx-port <site> <port>`.
- Add more domains to a site with `bench setup add-domain <domain> --site <site>`.
- Get TLS certificates with `sudo bench setup lets-encrypt <site>`, or `bench setup wildcard-ssl <domain>`.

Run `bench setup nginx` after each change, then reload nginx.

## Updates and maintenance

`bench update` pulls apps, installs requirements, runs `migrate` on all sites, builds assets and restarts. Options include `--pull`, `--patch`, `--build`, `--requirements`, `--restart-supervisor`, `--restart-systemd`, `--no-backup` and `--reset`.

For a manual upgrade of one site:

```bash
bench --site erp.example.com set-maintenance-mode on
bench --site erp.example.com ready-for-migration
bench --site erp.example.com migrate
bench --site erp.example.com set-maintenance-mode off
bench restart
```

- `set-maintenance-mode on` sets `maintenance_mode` in the site config. The scheduler stops enqueuing jobs. Requests can still read: Frappe opens a read-only transaction, or connects to the replica when `read_from_replica` is set. Set `allow_reads_during_maintenance` to `0` to stop all requests.
- `ready-for-migration` exits with code 1 if the site still has pending background jobs.
- `bench --site <site> scheduler pause` and `scheduler resume` set `pause_scheduler`. `scheduler disable` and `scheduler enable` change the setting in System Settings. `scheduler status` shows the state.
- `bench doctor` shows worker and scheduler status. `bench --site <site> show-pending-jobs` lists queued jobs.

See [Upgrading](upgrading.md) for version changes.

## Scaling

- Raise `gunicorn_workers` for more parallel requests. Each worker is a process with its own memory.
- Raise `background_workers`, or add queues under `workers`, for more job throughput. Put slow jobs on `long` so that they do not block `short`.
- Run more benches behind a load balancer only with a shared database, shared Redis and shared file storage. All benches must run the same app versions.
- Point `db_host`, `redis_cache` and `redis_queue` in `common_site_config.json` to external servers when you move them off the bench host.

## Pitfalls

- Do not run `bench start` in production. It runs `bench watch` and the development server.
- Do not keep `developer_mode` on in production. In developer mode, saving a standard DocType writes its files to the app folder.
- Restart the workers after a code change. They keep the old code in memory.
- Keep backups of `sites/*/site_config.json`. They hold the database password and `encryption_key`.

## Related

- [Architecture](../../getting-started/architecture.md)
- [Installation](../../getting-started/installation.md)
- [Background Jobs](../runtime/background-jobs.md)
- [Scheduler](../runtime/scheduler.md)
- [Configuration](configuration.md)
- [Backups](../../user-guide/system-administration/backups.md)
- [CLI reference](../../reference/configuration/cli.md)
