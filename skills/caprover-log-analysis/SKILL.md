---
name: caprover-log-analysis
description: >-
  Use this skill for any request to read, tail, search, count, or make sense of
  logs on a CapRover or Docker Swarm server (directly or over SSH) — including
  when the user just says an app on the server is broken or unreachable and
  wants its logs checked. Covers: inspecting a CapRover app's (srv-captain--*)
  recent errors; jointly analyzing related containers (app + database + worker)
  around a crash or suspected restart; statistics over captain-nginx access
  logs — per-domain traffic, status-code / 404 / 5xx counts, suspicious IPs or
  vulnerability-scan probes; debugging failed deployments via captain-captain;
  answering how to view Swarm service logs at all (e.g. docker logs fails with
  "No such container"); and writing incident or postmortem reports backed by
  server log evidence, in the user's language. Not for performance or config
  tuning (use performance-optimize), and not for analyzing a standalone log
  file already sitting on the user's local machine.
---

# CapRover Log Analysis

## Operating Model

Treat every log investigation as read-only work on a production system. The only Docker commands you need are `docker service logs`, `docker service ls`, `docker service ps`, `docker inspect`, and `docker ps`. Never run `docker service update`, `scale`, `rm`, `restart`, or anything that mutates state — a log question never justifies a state change. If the investigation reveals a tuning or config problem, report it and hand off to the performance-optimize skill instead of fixing it yourself.

Decide where commands run before running anything:

1. **Already on the CapRover host** (you can run `docker service ls` directly) — run commands locally.
2. **Remote host with SSH access** (the common case) — wrap every command as `ssh <host> '<command>'`. Confirm the SSH alias or address with the user if not obvious from context.
3. **No SSH / docker access** — fall back to the `caprover` CLI or the CapRover HTTP API (see the fallback section below). These only expose per-app logs, not captain-captain or captain-nginx, so say so when you are forced onto this path.

Mental model: CapRover is a thin management layer on Docker Swarm. Every app and every system component is a Swarm *service*, and logs live in the service's task containers — so `docker service logs` is the universal read primitive. Always bound reads with `--since` and/or `--tail`; an unbounded read on a busy service can return hundreds of megabytes.

## Log Topology — What Lives Where

Three kinds of services matter, and knowing which one holds the answer is most of the job:

- **`captain-captain`** — the CapRover main program. Deployment logs, image build output, app lifecycle events (create/update/scale), certificate issuance, and CapRover's own errors. **First stop for any failed deployment or build.**
- **`captain-nginx`** — the front proxy for all HTTP(S) traffic. Inside the container, `access.log` and `error.log` are symlinks to `/dev/stdout` and `/dev/stderr`, so **the only way to read nginx logs is `docker service logs captain-nginx`** — do not `docker exec` into the container hunting for log files; there are none. Each access-log line begins with the virtual-host name (the app's domain), which is the key for filtering traffic per app.
- **`srv-captain--<appname>`** — one Swarm service per app. An "app" with a database or worker is *several sibling services* (e.g. `srv-captain--kimai`, `srv-captain--kimai-db`, `srv-captain--kimai-fpm`). Discover the whole family before analyzing:

  ```bash
  docker service ls --filter name=srv-captain--<app>
  ```

There is also `captain-certbot` (TLS issuance) and, on the host, the `/captain/` directory tree (persistent config in `data/`, regenerated artifacts in `generated/`). For config-vs-log questions read `references/caprover-internals.md`.

## Reading Logs — Command Reference

The workhorse — bounded, timestamped, with stderr merged:

```bash
docker service logs --since 30m --timestamps <service-name> 2>&1
docker service logs --tail 200 --timestamps <service-name> 2>&1
```

Replica and task health (crash loops, failed deploys show up here, not in logs):

```bash
docker service ls --format "{{.Name}} {{.Replicas}} {{.Image}}"
docker service ps --no-trunc <service-name>       # ERROR column explains task failures
```

Pull several related services in one pass (write output to a local file per service):

```bash
for s in $(ssh <host> 'docker service ls --format "{{.Name}}"' | grep -iE '<app-pattern>|captain-(captain|nginx)'); do
  echo "== $s =="
  ssh <host> "docker service logs --since 40m --tail 150 --timestamps $s 2>&1"
done
```

Fallbacks when `docker service logs` is unavailable or empty:

- `docker ps --filter name=<service>` then `docker logs <container-id>` — reads a single task's container directly; useful when the service was just recreated and `service logs` lags.
- `caprover logs --appName <app>` (CLI) or `GET /api/v2/user/apps/appData/<app>/logs` (HTTP API with auth token) — per-app logs only; last resort without shell access.

### Pitfalls

- App service names **require the `srv-captain--` prefix**. `docker service logs myapp` fails; it's `srv-captain--myapp`.
- `docker service logs` writes to **stderr**. Without `2>&1` your pipes and greps see nothing.
- Swarm services are **not systemd units** — `journalctl -u srv-captain--myapp` returns nothing. Use `docker service logs`.
- Production images are often minimal: `docker exec <c> sh` may fail because there is no shell. Prefer reading logs from outside; treat `exec` as a last resort and ask the user before using it.
- Never use `--follow` in non-interactive execution — it never returns. Use `--since`/`--tail` snapshots.
- With multiple replicas, `service logs` interleaves all tasks. The task name prefix on each line (`<service>.<slot>.<task-id>`) tells them apart.

## Combined Analysis Workflow

Single-service tails answer single-service questions; real incidents almost always need logs from 2+ services correlated. Work through these steps:

1. **Pin down the symptom and time window.** What is failing, and since when? Default to the last 30–60 minutes if the user gave no window; widen only with evidence. Convert to a `--since` value (`30m`, `2h`, or RFC3339).
2. **Inventory services and replica health.** `docker service ls` for the big picture, `docker service ps --no-trunc` on anything suspicious. A replica count like `2/1` or `0/1` is itself a finding — record it.
3. **Pull logs per service to local files.** One file per service in a local working directory (never write files on the server). Bounded windows, timestamps on, `2>&1`.
4. **Aggregate quantitatively.** Count exact error strings, group nginx lines by vhost and status code, extract top-N offenders, then sample a few representative raw lines per pattern. Correlate across services by timestamp and by business IDs/domains appearing in multiple logs. Never report "there are some errors" — report *which* error, *how many times*, *when*, with sample lines.
5. **Route to the right playbook.** For the four common analysis shapes — multi-container correlation, cross-service error aggregation, nginx access-log analysis, deployment/build failure — follow `references/analysis-playbooks.md`.

Logs show what the server did with the requests it routed — they cannot show a
misrouted vhost, a wrong TLS cert, or a placeholder page served with a 200. When
user-facing behavior is itself in question, a couple of read-only probes from your
local machine (`curl -sk -o /dev/null -w "%{http_code} %{size_download}\n" https://<domain>/`,
`openssl s_client` for the served cert) are a legitimate cross-check; each probe also
writes an access-log line you can then locate to confirm the routing path. Keep it to
a handful of GETs against a production app, and never POST.

## Reporting

Write the final report in the language the user is conversing in. Use an incident-report structure:

1. **Time window & scope** — what range was examined, which services.
2. **Log sources** — exact services/commands the evidence came from.
3. **Findings, ordered by severity** — each with occurrence counts and time distribution.
4. **Key log excerpts** — a few representative raw lines per finding, trimmed.
5. **Probable cause(s)** — reasoned from the correlated evidence; mark speculation as such.
6. **Recommended next actions** — investigation or fixes; anything mutating goes to the user (or performance-optimize), not executed here.

If the logs show no anomalies, state that explicitly ("no 5xx in the window, no error-level lines in N lines examined") — an empty-but-verified result is a valid finding. Every conclusion must be backed by a count or a quoted log line from this run.

## Resource Selection

- `references/analysis-playbooks.md` — full command pipelines and interpretation notes for the four analysis shapes (multi-container correlation, error aggregation, nginx access-log analysis, deployment/build debugging). Read the relevant playbook before starting step 5 above.
- `references/caprover-internals.md` — the `/captain/` directory layout, config-to-runtime chain, and caprover CLI / HTTP API details. Read when the question involves where config lives, or when you must use the CLI/API fallback.

## Safety Defaults

- Allowed: `docker service logs / ls / ps`, `docker ps`, `docker inspect`, `docker logs`, read-only file reads under `/captain/`. Forbidden: `service update/scale/rm/rollback`, `restart`, `stack`, writes of any kind on the server.
- Ask before any `docker exec`, even a read-only one — it enters a production container.
- On a busy service, probe with `--tail 50` first to gauge volume before requesting a large `--since` window.
- Store intermediate files locally (working directory or scratchpad), never on the server.
- If the user's real need turns out to be tuning ("nginx worker limits", "make it handle more connections"), stop and hand off to performance-optimize with your findings attached.
