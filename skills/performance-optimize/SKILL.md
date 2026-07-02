---
name: performance-optimize
description: Performance and high-concurrency tuning for Linux servers, Docker, Docker Swarm, CapRover, Nginx reverse proxies, and Go web services. Use when investigating bottlenecks, planning or applying kernel/sysctl/ulimit/container/proxy tuning, validating nofile or connection limits, preparing rollback-safe production changes, or optimizing for high concurrency such as tens of thousands or more mixed HTTP/WebSocket connections.
---

# Performance Optimize

## Operating Model

Treat performance work as production-risk work. Start with evidence, identify the current bottleneck, make narrow reversible changes, and validate both capacity and regression risk after each restart or rollout.

Do not apply host-level sysctl, Docker daemon, Swarm service, CapRover, or Nginx changes blindly. Confirm the target OS, traffic path, service names, downtime policy, rollback path, and whether the user wants a plan only or live execution.

## Workflow

1. Collect the traffic path and workload target: public listener, proxy chain, app runtime, expected concurrency, burst behavior, keepalive/WebSocket use, and acceptable restart windows.
2. Inspect current limits before editing: `ulimit -n`, `/proc/<pid>/limits`, `sysctl -a`, Docker daemon config, service-level ulimits, Nginx `nginx -T`, `ss -s`, `nstat -az`, and relevant service logs.
3. Identify the active cap. Prioritize hard ceilings such as Nginx `worker_processes`, `worker_connections`, process `nofile`, Docker default ulimits, listen backlogs, conntrack limits, and app server limits before micro-optimizing.
4. Back up every file or generated template that will be changed. Use timestamped backups and record the exact pre-change values.
5. Change persistent sources of truth, not generated artifacts, unless the generated file is the documented source. For CapRover, prefer `/captain/data` templates and config paths over one-off edits under `/captain/generated`.
6. Validate config syntax before restarting. Use service-specific validators such as `dockerd --validate --config-file /etc/docker/daemon.json` and `docker exec captain-nginx nginx -t`. Note that `sysctl --system` applies settings immediately rather than validating them, so review sysctl drop-ins before running it.
7. Restart or update the smallest safe set of components. When Docker or Swarm is involved, expect brief interruption and watch task replacement live.
8. Verify effective runtime state, not only files on disk. Check process limits, rendered Nginx config, listening ports, health checks, Swarm task status, logs, socket summaries, and network counters.
9. Report final state with before/after caps, changed files, validation commands, remaining risks, and rollback instructions.

## Resource Selection

Read `references/debian-docker-caprover-high-concurrency.md` when the server uses Debian, Docker or Docker Swarm, CapRover, and Nginx as the HTTPS reverse proxy, especially for tens of thousands or more concurrent mixed connections.

For other stacks, use the same workflow but adapt values to the workload, memory budget, kernel version, container runtime, proxy, and application server. Avoid copying CapRover-specific template paths into non-CapRover systems.

## Safety Defaults

- Keep changes reversible and scoped to the active bottleneck.
- Prefer additive drop-ins under `/etc/sysctl.d`, `/etc/security/limits.d`, `/etc/systemd/system.conf.d`, and service override directories.
- Merge JSON configs instead of overwriting them.
- Preserve existing app routing, TLS behavior, headers, WebSocket support, ACME behavior, and CapRover EJS variables.
- Keep `net.ipv4.tcp_tw_reuse` conservative under mixed or NAT traffic unless the user explicitly accepts the risk.
- Do not remove logging while tuning. If log volume is high, rotate and buffer it instead.
- Do not claim success until runtime limits and health checks confirm the new settings are effective.
