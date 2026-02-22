# CaddyManager Quadlet + Podman systemd setup

This folder contains the minimal systemd-quadlet layout for running CaddyManager with
Podman in production-like mode on Linux.

- `caddymanager.network`
- `caddymanager-backend.container`
- `caddymanager-frontend.container`
- `caddymanager.service`
- `backend.env.example`
- `frontend.env.example`

## 1) Architecture

### Units
- `caddymanager.network`  
  Creates a user-defined Podman network named `caddymanager` for backend/frontend communication.

- `caddymanager-backend.container`  
  Builds and runs the API image from the checked-out repository using:
  `podman build -t localhost/caddymanager-backend:dev /home/alfanick/Projects/caddymanager/backend`.
  - Internal API port: `3000`
  - Host bind: `127.0.0.1:12000:3000`
  - Uses shared config/data directory: `/etc/caddy/manager:/app/data:Z,U`
  - Reads env from: `/etc/caddy/manager/backend.env`
  - Auto-restart on failure (`Restart=always`, backoff `RestartSec=5s`)

- `caddymanager-frontend.container`  
  Builds and runs the frontend image from the checked-out repository using:
  `podman build -t localhost/caddymanager-frontend:dev /home/alfanick/Projects/caddymanager/frontend`.
  - Internal web port: `80`
  - Host bind: `127.0.0.1:12001:80`
  - Reads env from: `/etc/caddy/manager/frontend.env`
  - Starts after backend (`After=...caddymanager-backend.service`)
  - Auto-restart on failure (`Restart=always`, backoff `RestartSec=5s`)

- `caddymanager.service`  
  Orchestrator unit used as the boot entrypoint. It depends on:
  - `podman.service`
  - `caddy.service`
  - `tailscaled.service`
  and then starts both container units with `systemctl start ...` during startup.

### Runtime behavior
- Services run as root-managed systemd units (because Quadlet/systemd writes containers).
- Actual container processes run with container image users (`node` / `caddy`) and data is bound to `/etc/caddy/manager`.
- Logging is sent to journald in both containers (`StandardOutput=journal`, `StandardError=journal`, `LogDriver=journald`).
- Restart policy gives automatic recovery on failure.

## 2) Environment variables

#### Backend (`/etc/caddy/manager/backend.env`)
Use SQLite (no Mongo):

- `DB_ENGINE=sqlite`
- `SQLITE_DB_PATH=/app/data/caddymanager.sqlite`

Generate a JWT secret:

```bash
openssl rand -base64 64 | tr -d '\n'
```

Fill `JWT_SECRET=` with that value.

Start with `backend.env.example`.

#### Frontend (`/etc/caddy/manager/frontend.env`)
- `BACKEND_HOST=caddymanager-backend:3000`
- default UI flags (`APP_NAME`, `DARK_MODE`, etc.)

## 3) Step-by-step installation

Run these as root.

1. Install system files:

```bash
mkdir -p /etc/containers/systemd /etc/caddy/manager
cp -n systemd/backend.env.example /etc/caddy/manager/backend.env
cp -n systemd/frontend.env.example /etc/caddy/manager/frontend.env
chmod 600 /etc/caddy/manager/backend.env /etc/caddy/manager/frontend.env
```

2. Fill required values in both env files:

```bash
vim /etc/caddy/manager/backend.env
vim /etc/caddy/manager/frontend.env
```

At minimum:
- `JWT_SECRET` in backend env
- `DB_ENGINE=sqlite`
- `SQLITE_DB_PATH=/app/data/caddymanager.sqlite`

3. Symlink quadlet files into the system quadlet directory:

```bash
ln -s /home/alfanick/Projects/caddymanager/systemd/caddymanager.network /etc/containers/systemd/caddymanager.network
ln -s /home/alfanick/Projects/caddymanager/systemd/caddymanager-backend.container /etc/containers/systemd/caddymanager-backend.container
ln -s /home/alfanick/Projects/caddymanager/systemd/caddymanager-frontend.container /etc/containers/systemd/caddymanager-frontend.container
ln -s /home/alfanick/Projects/caddymanager/systemd/caddymanager.service /etc/systemd/system/caddymanager.service
```

4. Reload systemd + enable lingering for the `caddy` user (if needed):

```bash
systemctl daemon-reload
loginctl enable-linger caddy
```

5. Enable and start the stack:

```bash
systemctl enable --now caddymanager.service

systemctl is-active caddymanager.service caddymanager-backend.service caddymanager-frontend.service
```

`caddymanager.service` is the only unit that should be enabled for boot when using the orchestrator layout shown above.

If you still see "Unit ... not found", verify:

```bash
ls -l /etc/containers/systemd/caddymanager-*.container
systemctl list-unit-files | rg 'caddymanager-(backend|frontend)'
systemctl daemon-reload
```

6. Verify:

```bash
systemctl status caddymanager.service caddymanager-backend.service caddymanager-frontend.service --no-pager -l
systemctl is-active caddymanager.service caddymanager-backend.service caddymanager-frontend.service
ss -ltnp | rg '127\\.0\\.0\\.1:12000|127\\.0\\.0\\.1:12001'
journalctl -u caddymanager-backend.service -n 80 --no-pager
journalctl -u caddymanager-frontend.service -n 80 --no-pager
```

## 4) Updating / redeploying

- Rebuild from source on the next (re)start:

```bash
systemctl restart caddymanager-backend.service caddymanager-frontend.service
```

- If your checkout path is not `/home/alfanick/Projects/caddymanager`, update the
  `podman build ...` `ExecStartPre` paths in both `.container` files.

## 5) Hardening notes

- The host ports are currently loopback-only:
  - `127.0.0.1:12000` (backend)
  - `127.0.0.1:12001` (frontend)
- This avoids direct exposure on all interfaces and should be safe with your existing reverse-proxy/Tailscale path.
- If you need different host ports, edit `PublishPort` in both `.container` files.
