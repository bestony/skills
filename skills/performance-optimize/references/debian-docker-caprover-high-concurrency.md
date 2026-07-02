# Debian Docker CapRover High-Concurrency Tuning

Use this reference for HTTPS traffic that enters CapRover Nginx and is proxied to a Go application, with a mixed workload target around 30000 concurrent connections. Brief Docker, CapRover, and Nginx restarts are acceptable.

## Baseline Findings To Confirm

Confirm these limits before applying the plan:

- CapRover Nginx may be capped by `worker_processes 1`, `worker_connections 1024`, or a container soft `nofile` limit of `1024`.
- Docker may have no daemon-level high-concurrency defaults.
- Swarm services may not inherit host `nofile` changes until service ulimits are updated or tasks are recreated.
- CapRover may regenerate Nginx config, so changes under `/captain/generated/nginx/nginx.conf` alone may not persist.

## Backup Checklist

Create timestamped backups before editing:

```bash
ts="$(date -u +%Y%m%dT%H%M%SZ)"
mkdir -p "/root/high-concurrency-backups/$ts"
cp -a /etc/sysctl.d "/root/high-concurrency-backups/$ts/sysctl.d"
cp -a /etc/systemd "/root/high-concurrency-backups/$ts/systemd"
test -f /etc/docker/daemon.json && cp -a /etc/docker/daemon.json "/root/high-concurrency-backups/$ts/daemon.json"
test -f /captain/data/config-captain.json && cp -a /captain/data/config-captain.json "/root/high-concurrency-backups/$ts/config-captain.json"
test -d /captain/generated/nginx && cp -a /captain/generated/nginx "/root/high-concurrency-backups/$ts/generated-nginx"
test -f /captain/data/nginx-shared.conf && cp -a /captain/data/nginx-shared.conf "/root/high-concurrency-backups/$ts/nginx-shared.conf"
test -f /captain/data/server-block-conf-override.ejs && cp -a /captain/data/server-block-conf-override.ejs "/root/high-concurrency-backups/$ts/server-block-conf-override.ejs"
```

Adjust CapRover paths if the installation uses different custom template filenames.

## Kernel And Network Tuning

Write persistent settings to `/etc/sysctl.d/99-high-concurrency-forwarder.conf`:

```conf
net.core.somaxconn = 65535
net.core.netdev_max_backlog = 65536
net.ipv4.tcp_max_syn_backlog = 65535

net.ipv4.ip_local_port_range = 10000 65535
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.ipv4.tcp_rmem = 4096 87380 134217728
net.ipv4.tcp_wmem = 4096 65536 134217728

net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 5

net.netfilter.nf_conntrack_max = 1048576
net.netfilter.nf_conntrack_tcp_timeout_established = 86400
net.netfilter.nf_conntrack_tcp_timeout_close = 30
net.netfilter.nf_conntrack_tcp_timeout_time_wait = 30
net.netfilter.nf_conntrack_tcp_timeout_fin_wait = 30

net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr

net.core.rps_sock_flow_entries = 32768
```

Before enabling BBR, try to load the modules:

```bash
modprobe tcp_bbr || true
modprobe sch_fq || true
sysctl --system
sysctl net.ipv4.tcp_congestion_control net.core.default_qdisc
```

Keep `net.ipv4.tcp_tw_reuse=2` unchanged unless the user explicitly wants to revisit the NAT and mixed-traffic tradeoff.

## Process And File Descriptor Limits

Create `/etc/security/limits.d/99-high-concurrency.conf`:

```conf
* soft nofile 1048576
* hard nofile 1048576
root soft nofile 1048576
root hard nofile 1048576
```

Create `/etc/systemd/system.conf.d/99-high-concurrency.conf`:

```ini
[Manager]
DefaultLimitNOFILE=1048576
```

Create `/etc/systemd/system/docker.service.d/10-limits.conf`:

```ini
[Service]
LimitNOFILE=1048576
```

Then reload systemd before restarting Docker:

```bash
systemctl daemon-reload
systemctl show --property DefaultLimitNOFILE
systemctl show docker --property LimitNOFILE
```

## Docker Daemon Defaults

Merge these keys into `/etc/docker/daemon.json`; do not overwrite unrelated keys:

```json
{
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Soft": 1048576,
      "Hard": 1048576
    }
  },
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "5"
  },
  "userland-proxy": false
}
```

Validate before restart:

```bash
dockerd --validate --config-file /etc/docker/daemon.json
systemctl restart docker
docker info --format '{{json .DefaultUlimits}}'
```

After setting `userland-proxy=false`, verify published ports still bind and receive traffic.

## Swarm Service Updates

After Docker restarts, update only the affected services first:

```bash
docker service update --ulimit-add nofile=1048576:1048576 captain-nginx
docker service update --ulimit-add nofile=1048576:1048576 captain-captain
docker service ps captain-nginx captain-captain --no-trunc
```

Force-refresh app services only when required by runtime validation or if they must inherit new Docker defaults.

## CapRover Nginx Persistence

Persist Nginx tuning through CapRover's supported config or template path. Do not rely only on editing `/captain/generated/nginx/nginx.conf`.

Base the custom config on the current CapRover template and preserve EJS variables, especially `base.dhparamsFilePath`. Tune the global Nginx settings:

```nginx
worker_processes auto;
worker_rlimit_nofile 1048576;

events {
    worker_connections 65535;
    multi_accept on;
}

http {
    map $http_upgrade $connection_upgrade {
        default upgrade;
        '' close;
    }

    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    client_header_timeout 15;
    client_body_timeout 30;
    send_timeout 30;
    ssl_session_cache shared:SSL:50m;
    access_log /var/log/nginx/access.log main buffer=256k flush=5s;
    open_file_cache max=200000 inactive=20s;
    open_file_cache_valid 30s;
    open_file_cache_min_uses 2;
    open_file_cache_errors on;
}
```

Create or update `/captain/data/server-block-conf-override.ejs` from the current default app template. Preserve existing CapRover behavior while adding proxy keepalive and WebSocket headers:

```nginx
proxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection $connection_upgrade;
proxy_connect_timeout 10s;
proxy_send_timeout 90s;
proxy_read_timeout 90s;
```

Validate rendered config before reload:

```bash
docker exec captain-nginx nginx -t
docker exec captain-nginx nginx -T | grep -E 'worker_processes|worker_connections|worker_rlimit_nofile|connection_upgrade|proxy_http_version'
```

## NIC RPS On Single-Queue Public Interface

For a single-queue public interface such as `eth0`, add a small systemd oneshot that sets:

```bash
echo ff > /sys/class/net/eth0/queues/rx-0/rps_cpus
echo 32768 > /sys/class/net/eth0/queues/rx-0/rps_flow_cnt
sysctl -w net.core.rps_sock_flow_entries=32768
```

Validate the real interface name first with `ip -br link` and `ethtool -l eth0` when available. Do not apply `ff` blindly on very small or very large CPU topologies; map the CPU mask to the host.

## Post-Restart Validation

Run these checks after each restart or service update:

```bash
docker service ls
docker service ps captain-nginx captain-captain --no-trunc

nginx_pid="$(docker exec captain-nginx sh -c 'pidof nginx | awk "{print \\$1}"')"
docker exec captain-nginx sh -c "cat /proc/$nginx_pid/limits | grep 'Max open files'"
docker exec captain-nginx nginx -T | grep -E 'worker_processes|worker_connections|worker_rlimit_nofile|connection_upgrade|proxy_http_version'

ss -ltnp | grep -E ':80|:443|:3000'
ss -s
nstat -az | grep -E 'Listen|ListenOverflows|ListenDrops|TCPAbort|TCPTimeout|Retrans|Syncookies|ConnTrack|IpExt'
docker service logs --since 10m captain-nginx
docker service logs --since 10m captain-captain
```

Check HTTP and HTTPS health endpoints from outside the host when possible.

## Acceptance Criteria

- Nginx capacity is not bounded by `1024` worker connections or a `1024` soft `nofile`.
- Docker-created containers inherit `nofile` soft and hard limits of `1048576`.
- CapRover Nginx changes survive restart and regeneration.
- Swarm services show no failed tasks after updates.
- Ports `80`, `443`, and `3000` are reachable when they are expected to be published.
- Logs and counters show no Nginx config errors, listen drops, conntrack pressure, reload loops, or CapRover regressions.

## Rollback

Restore the timestamped backups, validate config syntax, then restart the smallest affected component. If Docker daemon restart breaks published ports, revert `userland-proxy=false` first and restart Docker again. If CapRover routing breaks, revert the CapRover Nginx template override and confirm `docker exec captain-nginx nginx -t` before reloading.
