# CapRover Internals — Directories, Config Chain, CLI/API Fallbacks

Read this when a log question turns into a "where does this config/data live" question,
or when you have no shell/docker access and must fall back to the caprover CLI or API.

## Service Topology Recap

CapRover ≈ Docker Swarm + a management container. Everything is a Swarm service:

| Service | Role | Log content |
|---------|------|-------------|
| `captain-captain` | CapRover core (Node.js) | Deploys, builds, app lifecycle, cert issuance, API errors |
| `captain-nginx` | Front proxy for all HTTP(S) | Access + error logs (stdout/stderr only) |
| `captain-certbot` | Let's Encrypt | Cert renewals and failures |
| `srv-captain--<app>` | One per app (siblings for db/worker) | The app's own stdout/stderr |

Docker Swarm keeps service logs in the container's json-file log driver on whichever
node runs the task; `docker service logs` aggregates them for you. Log history does
not survive container removal — if a task was recreated, its predecessor's logs may be
gone. `docker service ps` still shows the dead task's exit reason, which is often enough.

## The /captain Directory (on the host)

```
/captain/
├── data/        # PERSISTENT — source of truth. App definitions, nginx template
│                # overrides, TLS certs, registry config. Back up target.
├── generated/   # REGENERATED on every captain restart from data/ + templates.
│                # Rendered nginx configs live here. Never edit; edits are lost.
└── temp/        # Build workspaces and scratch. Safe to ignore for log work.
```

Config chain for nginx: CapRover EJS templates + per-app settings in `data/`
→ rendered into `generated/nginx/` → loaded by the captain-nginx container.
So when a log line points at odd routing, the *effective* config is in
`generated/`, but the thing to report/fix is in `data/` (or the CapRover UI).
Reading these files is fine; changing them is performance-optimize territory.

Nginx log files do NOT live under /captain: inside the captain-nginx container,
access.log/error.log symlink to /dev/stdout and /dev/stderr. The only log path is
`docker service logs captain-nginx`.

## caprover CLI Fallback

Requires `npm i -g caprover` locally and machine credentials (`caprover login`,
stored per named machine).

```bash
caprover logs --appName <app> --machineName <machine>   # app stdout/stderr
caprover list                                            # configured machines
```

Limits: per-app logs only — no captain-captain, no captain-nginx, no `service ps`
equivalents, no `--since` filtering. Good enough to answer "is my app throwing
errors", not enough for combined analysis. State this limitation in the report when
the CLI is all you have.

## HTTP API Fallback

The same API the web dashboard uses. Authenticate, then fetch logs:

```bash
# 1. Get a token (password auth)
curl -s -X POST https://captain.<root-domain>/api/v2/login \
  -H 'Content-Type: application/json' \
  -H 'x-namespace: captain' \
  -d '{"password":"<password>"}'          # → .data.token

# 2. App logs (last chunk of stdout/stderr)
curl -s https://captain.<root-domain>/api/v2/user/apps/appData/<app>/logs \
  -H 'x-namespace: captain' -H 'x-captain-auth: <token>'

# 3. App definitions (replica counts, env vars, domains — useful for inventory)
curl -s https://captain.<root-domain>/api/v2/user/apps/appDefinitions \
  -H 'x-namespace: captain' -H 'x-captain-auth: <token>'
```

Every response is JSON with `.status` (100 = OK) and `.data`. Same limitation as the
CLI: app-level only. Never log or echo the password/token into files that persist.

## Version Notes

Verified against CapRover 1.14.x. Service names (`captain-captain`, `captain-nginx`,
`srv-captain--` prefix) have been stable across 1.x. The API is namespaced `/api/v2/`;
if a future major version changes it, the CLI is the more stable fallback.
