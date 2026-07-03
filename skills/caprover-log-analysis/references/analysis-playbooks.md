# Combined Analysis Playbooks

Four playbooks for the four common analysis shapes. Each assumes you already did the
workflow steps 1–3 from SKILL.md: symptom + time window pinned, services inventoried,
logs pulled to local files (one file per service, `--timestamps`, `2>&1`).

Commands below show the local form; prefix with `ssh <host> '...'` when remote.
`$W` stands for your local working directory.

---

## Playbook A — Multi-Container App Correlation

Use when one *app* is misbehaving and the app is really several sibling services
(web + database + worker/fpm). Example family: `srv-captain--kimai`,
`srv-captain--kimai-db`, `srv-captain--kimai-fpm`.

### 1. Discover the whole family — never assume it's one service

```bash
docker service ls --filter name=srv-captain--<app> --format "{{.Name}} {{.Replicas}} {{.Image}}"
```

If you analyze only the web container of a three-container app, you will misattribute
database failures to the app. The `--filter name=` match is a prefix match, so
`srv-captain--kimai` also returns `kimai-db` and `kimai-fpm`.

### 2. Pull all siblings plus the two system services over the SAME window

```bash
for s in srv-captain--<app> srv-captain--<app>-db srv-captain--<app>-fpm captain-nginx captain-captain; do
  docker service logs --since 2h --timestamps "$s" 2>&1 > "$W/${s}.log"
done
```

Same `--since` for every service — correlation is meaningless across different windows.

### 3. Build a cross-service timeline

Extract error-ish lines with timestamps from each file, then merge and sort:

```bash
grep -hiE 'error|fatal|panic|refused|timeout|denied|OOM' $W/srv-captain--<app>*.log \
  | sort > "$W/timeline.txt"
```

(`--timestamps` puts an RFC3339 timestamp at the start of each line, so a plain `sort`
is a chronological merge.)

Timezone trap: the Docker-added `--timestamps` prefix is always UTC, but timestamps
*inside* log lines follow each container's own TZ — e.g. a PHP-FPM image logging
`+0100` while the host and siblings log UTC. When you correlate by in-line timestamps
(nginx access lines, app-framework logs), check each container's offset first, or your
cross-service timeline silently misaligns by hours. The Docker prefix is the only
timebase guaranteed consistent across services.

### 4. Read the causal chain in the right direction

The typical dependency chain is **db → app → fpm/worker → nginx**. Root cause lives
upstream; symptoms live downstream:

- db logs show restarts / `too many connections` / OOM at T0
- app logs show `connection refused` / SQL errors starting T0+seconds
- nginx shows 502/504 for the app's vhost starting T0+seconds

If nginx 502s but the app logs are clean, check replica health (`service ps`) — the
task may be crash-looping so fast it logs nothing. If app errors but nginx is clean,
the problem is internal (cron, worker) and never surfaced to HTTP.

When a container restarted but nothing in the family explains *why*, look for an
external mutator before blaming the app: auto-updaters like **watchtower** SIGTERM
Swarm task containers when a floating tag (`:latest`, `mariadb:10`) gets a new
upstream release, then race Swarm's own task rebuild for volume locks — a classic
cause of "random" db crash-loops. Check `docker ps | grep -iE 'watchtower|ouroboros'`
and, if present, `docker logs watchtower --since 48h 2>&1 | tail -30` for
Stopping/Creating entries around the incident. `docker service inspect <svc>
--format '{{.UpdatedAt}}'` distinguishes a real CapRover deploy from an out-of-band
container replacement (deploys change UpdatedAt; watchtower doesn't).

### 5. Confirm against nginx by vhost

```bash
grep '<app-domain>' "$W/captain-nginx.log" | awk '{print $12}' | sort | uniq -c | sort -rn
```

(`$12` = status code with the standard Swarm `<task> |` prefix and no `--timestamps`;
verify the field index on a real line first — see Playbook C step 2.)

Compare the status-code mix during the incident window vs. before it.

---

## Playbook B — Error Aggregation Across Services

Use when the question is broad ("is anything wrong on this server?", "summarize errors
in the last hour") rather than about one app.

### 1. Dump every service to its own file

```bash
for s in $(docker service ls --format "{{.Name}}"); do
  docker service logs --since 60m --tail 300 --timestamps "$s" 2>&1 > "$W/$s.log"
done
```

`--tail 300` caps per-service volume so one chatty service can't blow up the pull.
If a service hits the cap, note it in the report ("truncated to last 300 lines").

### 2. Count exact error strings per service — numbers, not impressions

```bash
for f in $W/*.log; do
  n=$(grep -ciE 'error|fatal|panic|exception|refused|timeout' "$f")
  [ "$n" -gt 0 ] && echo "$n $(basename $f)"
done | sort -rn
```

### 3. Top-N distinct error patterns

Strip timestamps and variable parts, then count:

```bash
grep -hiE 'error|fatal|panic' $W/*.log \
  | sed -E 's/^[^ ]+ [^ ]+ //; s/[0-9a-f-]{8,}/<ID>/g; s/[0-9]{3,}/<N>/g' \
  | sort | uniq -c | sort -rn | head -20
```

The `sed` normalization (IDs/numbers → placeholders) is what turns 500 near-identical
lines into "1 pattern × 500 occurrences" — that is the difference between a report and
a log dump.

### 4. Sample representative raw lines

For each top pattern, `grep` back 2–3 *original* lines (with timestamps) as evidence.
A count without a sample line is unverifiable; a sample without a count is anecdote.

### 5. Correlate by business identifiers

Request IDs, user IDs, order numbers, domains — grep the same identifier across all
service files to trace one failing transaction through the stack:

```bash
grep -h '<some-request-id>' $W/*.log | sort
```

---

## Playbook C — Nginx Access-Log Analysis

Use for traffic questions: per-domain volume, status-code mix, suspicious scanning,
slow upstreams. **All nginx logs come from `docker service logs captain-nginx`** —
access.log/error.log inside the container are symlinks to stdout/stderr; there are no
files to find.

### 1. Pull one bounded window

```bash
docker service logs --since 30m captain-nginx 2>&1 > "$W/nginx.log"
```

Access lines and error lines arrive interleaved. Access lines start (after the Swarm
task prefix) with the **vhost** (the app's domain); error lines contain `[error]`/`[warn]`.
Split them first:

```bash
grep -v '\[error\]\|\[warn\]\|\[crit\]' "$W/nginx.log" > "$W/access.log"
grep    '\[error\]\|\[warn\]\|\[crit\]' "$W/nginx.log" > "$W/errors.log"
```

### 2. Field positions

A real line from CapRover 1.14's default `log_format main` (after `docker service
logs` prefixes it with `<task> |`):

```
captain-nginx.1.xyz@host | 1.71.140.154 - - [03/Jul/2026:00:20:08 +0000] "cleaner.bangapi.com" "GET / HTTP/1.1" 200 10158 "-" "Mozilla/..."
```

Client IP comes FIRST and the vhost (`$host`) is a separate **quoted** field after
the timestamp. With the `<task> |` prefix (and no `--timestamps`, which would shift
everything by one more field), awk fields: `$3`=client IP, `$8`=vhost (quotes
included — strip with `gsub(/"/,"")` or match as `"\"domain\""`), `$9`=method
(leading quote), `$10`=request path, `$12`=status, `$13`=bytes.

**Always verify on a real line first** — count fields with
`head -2 access.log | awk '{for(i=1;i<=13;i++) printf "%d=%s ", i, $i; print ""}'`.
The Swarm prefix, `--timestamps`, and any customized nginx template all shift
columns, and a wrong field index silently produces garbage stats. Note also that
CapRover's nginx uses `access_log ... if=$ip_in_log` with a geo map that excludes
private ranges (10/8, 172.16/12, 192.168/16) — container-to-container traffic never
appears in the access log, so absence of internal calls is not evidence of no traffic.

### 3. Standard pipelines

Traffic per vhost:

```bash
awk '{gsub(/"/,"",$8); print $8}' "$W/access.log" | sort | uniq -c | sort -rn
```

Status-code distribution (overall, then per vhost):

```bash
awk '{print $12}' "$W/access.log" | sort | uniq -c | sort -rn
awk '{gsub(/"/,"",$8); print $8, $12}' "$W/access.log" | sort | uniq -c | sort -rn | head -30
```

Top client IPs and top paths (scanning shows up here):

```bash
awk '{print $3}' "$W/access.log" | sort | uniq -c | sort -rn | head -15
awk '{print $10}' "$W/access.log" | sort | uniq -c | sort -rn | head -20
```

Cross-vhost IPs (one client touching many domains = crawler/scanner signal):

```bash
awk '{gsub(/"/,"",$8); print $3, $8}' "$W/access.log" | sort -u | awk '{print $1}' | uniq -c | sort -rn | head
```

Scanner heuristics: one IP hitting many 404s, paths like `/wp-login.php`, `/.env`,
`/admin`, `/phpmyadmin` on apps that aren't WordPress/PHP, or a single IP touching
many different vhosts.

### 4. Status-code semantics behind a reverse proxy

| Code | Meaning here | Usual root cause |
|------|-------------|------------------|
| 499 | Client closed connection before nginx answered | Slow upstream (check 499↔504 ratio) or impatient clients/health checks |
| 502 | Upstream refused / reset | App container down or crash-looping — check `service ps` and app logs |
| 504 | Upstream timed out | App alive but too slow — DB queries, upstream `proxy_read_timeout` |
| 503 | No healthy upstream / limits | Zero healthy replicas, or rate limiting |
| 404 spike | — | Scanning (many IPs/paths) or broken deploy (one path, many IPs) |

5xx concentrated on one vhost → that app's problem (go to Playbook A). 5xx across all
vhosts → nginx or host-level problem (also check `$W/errors.log`).

---

## Playbook D — Deployment / Build Debugging

Use when a deploy failed, a build hangs, or an app never becomes healthy after update.

### 1. captain-captain first — builds happen there, not in the app service

```bash
docker service logs --since 30m --timestamps captain-captain 2>&1 | grep -iE '<appname>|error|fail'
```

Build output (Dockerfile steps, npm/pip installs) and deploy orchestration live in
captain-captain's log. An app whose build failed may have **no new log lines at all**
in `srv-captain--<app>` — absence of app logs is itself a clue that the failure is
upstream in the build.

### 2. Task history explains "deployed but not running"

```bash
docker service ps --no-trunc srv-captain--<app>
```

Read the columns: `CURRENT STATE` (e.g. `Failed 12 seconds ago` repeating = crash
loop), `ERROR` (the actual reason — exit codes, `task: non-zero exit (137)` = OOM-kill,
`No such image` = registry/build problem), and `DESIRED STATE` vs current.
`--no-trunc` matters: the ERROR column is truncated to uselessness without it.

Replica column readings from `docker service ls`:

- `1/1` healthy; `0/1` nothing running (crashed or never started)
- `2/1` — more tasks than desired: an update is mid-flight, or an old task won't die,
  or a rolling update is stuck. Look at `service ps` for two tasks in conflicting states.

For any replica anomaly or suspected stuck deploy, read the service's update status —
it records *when* and *why* a rolling update stalled, even if that was months ago:

```bash
docker service inspect srv-captain--<app> --format \
  'update={{if .UpdateStatus}}{{.UpdateStatus.State}} started={{.UpdateStatus.StartedAt}} msg={{.UpdateStatus.Message}}{{else}}none{{end}} | created={{.CreatedAt}} | updated={{.UpdatedAt}}'
```

`State=paused` + a "failure or early termination of task ..." message means a deploy
failed, Swarm suspended the update, and the old task was never reclaimed — the direct
cause of long-lived `2/1` states. Container `StartedAt` times only tell you when tasks
last (re)started (e.g. a daemon restart), NOT when the update broke; anchor the
incident on `UpdateStatus.StartedAt`, not on container ages.

### 3. First lines of the new task's log

```bash
docker service logs --tail 100 --timestamps srv-captain--<app> 2>&1
```

Crash-loop culprits are almost always in the first ~20 lines each restart: missing
env var, unreachable DB, port already in use, bad migration.

### 4. If the image itself is suspect

```bash
docker service inspect srv-captain--<app> --format '{{.Spec.TaskTemplate.ContainerSpec.Image}}'
```

Compare the image tag/digest with what the deploy was supposed to produce (CapRover
tags builds as `img-captain--<app>:<version>`); a stale tag means the build never
completed and the service is still running the previous version.
