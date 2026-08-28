---
num: "11"
title: "Probes and Alerts in OpenObserve"
desc: "Thin host and in-cluster probes that remote-write into OpenObserve — crashloops, Postgres, port exhaustion, Tailscale, and the k3d-to-host path."
tags: ["Observability", "OpenObserve", "Prometheus", "Alerting"]
color: warn
pubDate: 2026-08-28
---

## Context

The [logging recipe](/homelab/openobserve-logging) put every pod's stdout into OpenObserve. That's the search half. It will not page you. A CrashLoop of a bad image may write nothing useful. Postgres can accept a TCP handshake and still refuse `SELECT 1`. The Mac Mini can run out of ephemeral ports while every container looks `Running`. Tailscale can show a process and pass no traffic. And the homelab's most common lie is already in the [PostgreSQL recipe](/homelab/postgres-and-backups): pods time out to `host.k3d.internal:5432` while the database itself is fine.

So you want a thin metrics and probe layer that remote-writes into the OpenObserve you already run, then alerts from there. OpenObserve stays the brain — logs, metrics, and the page. You do not stand up Grafana OnCall (archived), and you do not stand up a second pager product for v1. A webhook into ntfy, Pushover, or Telegram is enough. GoAlert, Keep, or n8n can wait until you actually want auto-heal.

## Why OpenObserve stays the brain

The well-trodden move is Prometheus + Alertmanager + Grafana + a paging product. That's four new things to keep alive on a Mac Mini that already chose OpenObserve over Loki because it was *one binary*.

This recipe adds only what OpenObserve cannot do by itself: scrape kube-state-metrics, hit Postgres from two places, and read host counters the cluster cannot see. Prometheus (or an OpenTelemetry Collector) is a scraper with `remote_write`. Give it a couple of hours of local retention so a sink blip doesn't back up, then forget it as a UI. Alerts live in OpenObserve, next to the logs you will open when one fires.

## The architecture

```mermaid
flowchart TB
    subgraph k3d["k3d · cluster homelab"]
        KSM["kube-state-metrics"]
        PGK["pg-from-k3d<br/>postgres_exporter + pg_isready"]
    end
    subgraph host["Mac Mini / Colima"]
        NE["node_exporter"]
        PGH["pg-host-local<br/>postgres_exporter + pg_isready"]
        TS["Tailscale + TIME_WAIT textfile"]
    end
    Prom["Prometheus or OTel Collector<br/>scrape + remote_write"]
    OO["OpenObserve · metrics + SQL/PromQL alerts"]
    WH["webhook · ntfy / Pushover / Telegram"]

    KSM --> Prom
    PGK --> Prom
    NE --> Prom
    PGH --> Prom
    TS --> Prom
    Prom -->|"remote_write /api/org/prometheus/api/v1/write"| OO
    OO --> WH
```

Two rules that keep this from becoming a second observability stack:

- **Do not create the k3d cluster on the Docker compose/bridge network** so pods can "just" reach host containers. The [cluster-host recipe](/homelab/cluster-host) creates `homelab` the normal way. Pods reach host Postgres at `host.k3d.internal:5432`. Exporters on the host are published the same way, and Prometheus scrapes them at `host.k3d.internal:<port>`.
- **Ship every probe as a metric *and* one JSON log line.** Metrics page you. The log line is what you `str_match` next Tuesday when you want to know whether the path flickered at 03:12.

## Step 1 — Label prod and ignore the noise

Production apps in this cookbook live in namespace `apps` (the [PostgreSQL recipe](/homelab/postgres-and-backups) already puts the backup CronJob there). Label it so you can say "prod" without paging on CoreDNS or a k3d helper:

```bash
kubectl label namespace apps env=prod --overwrite
```

Every PromQL below filters `namespace="apps"` (or `namespace=~"apps|prod-.*"` if you split that way). kube-system and k3d internals stay out of the page. A single sidecar CrashLoop in `apps` is a warn. Available replicas = 0 on a Deployment that asked for replicas is a page.

## Step 2 — Secrets and the `monitoring` namespace

Prometheus needs the same OpenObserve credentials Fluent Bit already uses (the [secrets recipe](/homelab/1password-secrets)). Sync them into `monitoring` as well as `openobserve` / `fluent-bit`. The in-cluster Postgres probe needs the app DSN in `apps` — the same connection string the apps already use, not a new one.

```bash
kubectl create namespace monitoring
# re-run your vault sync so `monitoring` gets the OpenObserve user/password
```

## Step 3 — Host exporters (Colima + a Mac textfile)

node_exporter's Linux collectors (`node_nf_conntrack_*`, `node_sockstat_TCP_tw`) see the **Colima VM**, not macOS. The port-exhaustion incident in the [cluster-host recipe](/homelab/cluster-host) was macOS `TIME_WAIT` / `net.inet.ip.portrange`. You want both.

Run node_exporter as a host Docker container on the default bridge, published on the host — the same pattern as host Postgres. Do not join it to a k3d network.

```bash
mkdir -p "$HOME/homelab/probes"

docker run -d --name node-exporter --restart unless-stopped \
  --pid=host \
  --net=host \
  -v /:/host:ro \
  -v "$HOME/homelab/probes:/textfile" \
  quay.io/prometheus/node-exporter:latest \
  --path.rootfs=/host \
  --collector.textfile.directory=/textfile
```

`--net=host` binds `:9100` on the Colima VM, which is `host.k3d.internal:9100` from a pod.

`brew install node_exporter` and `brew services start node_exporter` also work if you would rather not run another container. From k3d you still scrape `host.k3d.internal:9100` only if that process is listening inside Colima or the port is published there. The Docker form matches how Postgres is already published.

Host Postgres containers sit on the default Docker bridge with `5432` published. Point a postgres_exporter at that published port and stamp it as the host vantage:

```bash
docker run -d --name postgres-exporter --restart unless-stopped \
  -p 9187:9187 \
  -e DATA_SOURCE_NAME="postgresql://appuser:${PG_PASSWORD}@172.17.0.1:5432/appdb?sslmode=disable" \
  -e PG_EXPORTER_CONSTANT_LABELS="vantage_point=pg-host-local" \
  quay.io/prometheuscommunity/postgres-exporter:latest
```

Fetch `$PG_PASSWORD` from the vault, don't type it. `172.17.0.1` is the default-bridge gateway from another container's point of view — the published host port, not the Postgres container's private network. If that address doesn't route on your Colima VM, use `host.docker.internal` instead. If you run more than one host Postgres, give each exporter (or each DSN) its own `vantage_point` / job label. Do not put k3d on that bridge so this "just works."

A `launchd` or `brew services` script on the Mac writes the signals macOS owns into the textfile directory Colima already shares (home directory — same rule as the Postgres data path):

```bash
#!/usr/bin/env bash
set -euo pipefail
# /usr/local/bin/homelab-host-probes.sh  (or ~/homelab/bin/...)
OUT="$HOME/homelab/probes/mac.prom"
PEER="${TAILSCALE_PEER:-your-laptop}"   # a stable MagicDNS name, not this machine

tw=$(netstat -an -p tcp 2>/dev/null | grep -c TIME_WAIT || true)
first=$(sysctl -n net.inet.ip.portrange.first)
last=$(sysctl -n net.inet.ip.portrange.last)

if pg_isready -h 127.0.0.1 -p 5432 >/dev/null 2>&1; then pg_local=1; else pg_local=0; fi

ts_json=$(tailscale status --json 2>/dev/null || echo '{}')
backend=$(printf '%s' "$ts_json" | python3 -c 'import json,sys; s=json.load(sys.stdin); print(1 if s.get("BackendState")=="Running" else 0)')
online=$(printf '%s' "$ts_json" | python3 -c 'import json,sys; s=json.load(sys.stdin); print(1 if (s.get("Self") or {}).get("Online") else 0)')
if tailscale ping -c 1 --timeout 5s "$PEER" >/dev/null 2>&1; then ping_ok=1; else ping_ok=0; fi
if ifconfig 2>/dev/null | grep -q 'utun'; then tun=1; else tun=0; fi

umask 022
cat > "$OUT.$$" <<EOF
# HELP mac_tcp_time_wait TIME_WAIT sockets on the Mac host
# TYPE mac_tcp_time_wait gauge
mac_tcp_time_wait $tw
# HELP mac_ephemeral_port_first sysctl net.inet.ip.portrange.first
# TYPE mac_ephemeral_port_first gauge
mac_ephemeral_port_first $first
# HELP mac_ephemeral_port_last sysctl net.inet.ip.portrange.last
# TYPE mac_ephemeral_port_last gauge
mac_ephemeral_port_last $last
# HELP probe_pg_up pg_isready from the Mac to 127.0.0.1:5432
# TYPE probe_pg_up gauge
probe_pg_up{vantage_point="pg-host-local"} $pg_local
# HELP probe_tailscale_backend_running tailscale status BackendState==Running
# TYPE probe_tailscale_backend_running gauge
probe_tailscale_backend_running $backend
# HELP probe_tailscale_self_online Self.Online from tailscale status --json
# TYPE probe_tailscale_self_online gauge
probe_tailscale_self_online $online
# HELP probe_tailscale_peer_ping tailscale ping to a stable peer
# TYPE probe_tailscale_peer_ping gauge
probe_tailscale_peer_ping{peer="$PEER"} $ping_ok
# HELP probe_tailscale_tun_up a utun interface exists
# TYPE probe_tailscale_tun_up gauge
probe_tailscale_tun_up $tun
EOF
mv "$OUT.$$" "$OUT"
```

Run it once a minute from launchd. node_exporter's textfile collector scrapes `*.prom` in that directory on the next pass.

The same script should POST one JSON object per check to the log stream you already have (`POST /api/<org>/default/_json`, same basic auth as Fluent Bit). In-cluster probes can skip the POST — they write to stdout and Fluent Bit ships them.

```bash
# one line per check; do not invent extra streams
curl -sS -u "$OO_USER:$OO_PASS" \
  -H 'Content-Type: application/json' \
  -X POST "http://127.0.0.1:5080/api/default/default/_json" \
  -d "[{\"probe\":\"pg_isready\",\"vantage_point\":\"pg-host-local\",\"ok\":$pg_local,\"host\":\"127.0.0.1\",\"port\":5432},
       {\"probe\":\"tailscale\",\"vantage_point\":\"host\",\"ok\":$online,\"backend_running\":$backend,\"peer_ping\":$ping_ok},
       {\"probe\":\"time_wait\",\"vantage_point\":\"mac\",\"ok\":1,\"time_wait\":$tw}]"
```

If OpenObserve is only reachable in-cluster, point that curl at `logs.otterpond.dev` from the tailnet, or at the published host port you already use for debugging. The payload is the point: one JSON line, fields you can `WHERE probe = 'pg_isready'` later.

## Step 4 — In-cluster kube-state-metrics and the k3d Postgres probe

Install a thin Prometheus. Disable Alertmanager and the pushgateway — those are the second pager this recipe is trying not to grow. Disable the chart's node-exporter DaemonSet; the host container above already covers Colima, and k3d's containerized nodes are not the Mac.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install prometheus prometheus-community/prometheus \
  -n monitoring \
  -f prometheus-values.yaml
```

The keys that matter in `prometheus-values.yaml`:

```yaml
alertmanager:
  enabled: false
prometheus-pushgateway:
  enabled: false
prometheus-node-exporter:
  enabled: false
kube-state-metrics:
  enabled: true
  metricLabelsAllowlist:
    - namespaces=[env]
    - pods=[app,env]
    - deployments=[app,env]
server:
  retention: 2h
  persistentVolume:
    size: 2Gi
  extraSecretMounts:
    - name: openobserve
      secretName: openobserve
      mountPath: /etc/prometheus/secrets/openobserve
      readOnly: true
  remoteWrite:
    - url: http://openobserve.openobserve.svc:5080/api/default/prometheus/api/v1/write
      basic_auth:
        username_file: /etc/prometheus/secrets/openobserve/username
        password_file: /etc/prometheus/secrets/openobserve/password
      queue_config:
        max_samples_per_send: 10000
extraScrapeConfigs: |
  - job_name: host-node
    static_configs:
      - targets: ["host.k3d.internal:9100"]
  - job_name: host-postgres
    static_configs:
      - targets: ["host.k3d.internal:9187"]
  - job_name: pg-from-k3d
    static_configs:
      - targets: ["pg-probe.apps.svc:9187"]
```

Copy the remote-write URL from your OpenObserve install if the service name or org differs. The path is always `/api/<org>/prometheus/api/v1/write`. An OpenTelemetry Collector with a Prometheus receiver and a `prometheusremotewrite` exporter is the same idea if you would rather not run Prometheus at all; the scrape targets and the OpenObserve URL do not change.

kube-state-metrics can also be installed on its own (`helm install kube-state-metrics prometheus-community/kube-state-metrics -n monitoring`) if you already have a scraper. The subchart is fewer moving parts.

The k3d vantage is a tiny Deployment in `apps` that talks the **app DSN** — `host.k3d.internal:5432` — not `127.0.0.1`. TCP-only is necessary and not sufficient; `postgres_exporter` runs `SELECT 1` (`pg_up`), and a sidecar `pg_isready` writes the JSON log line Fluent Bit will catch.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pg-probe
  namespace: apps
  labels:
    app: pg-probe
    env: prod
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pg-probe
  template:
    metadata:
      labels:
        app: pg-probe
        env: prod
    spec:
      containers:
        - name: postgres-exporter
          image: quay.io/prometheuscommunity/postgres-exporter:latest
          env:
            - name: DATA_SOURCE_NAME
              valueFrom:
                secretKeyRef:
                  name: app-db          # the same DSN the apps use
                  key: dsn
            - name: PG_EXPORTER_CONSTANT_LABELS
              value: vantage_point=pg-from-k3d
          ports:
            - containerPort: 9187
              name: metrics
        - name: pg-isready
          image: postgres:18-alpine   # client major must match the server
          envFrom:
            - secretRef:
                name: app-db
          command: ["/bin/sh", "-c"]
          args:
            - |
              while true; do
                if pg_isready -h host.k3d.internal -p 5432 >/dev/null 2>&1; then ok=1; else ok=0; fi
                printf '{"probe":"pg_isready","vantage_point":"pg-from-k3d","ok":%s,"host":"host.k3d.internal","port":5432}\n' "$ok"
                sleep 30
              done
---
apiVersion: v1
kind: Service
metadata:
  name: pg-probe
  namespace: apps
spec:
  selector:
    app: pg-probe
  ports:
    - name: metrics
      port: 9187
      targetPort: 9187
```

A CronJob that only prints `pg_isready` is fine if you do not want a standing exporter in `apps`; you lose a scrape target and keep the log line. Prefer the Deployment so the split-brain alert is a PromQL `and`, not a join across logs.

Optional: `helm install prometheus-blackbox-exporter prometheus-community/prometheus-blackbox-exporter -n monitoring` and scrape `tcp_connect` against `host.k3d.internal:5432` and a fixed tailnet IP / MagicDNS name. Treat blackbox TCP as the weaker signal — it is the handshake, not `SELECT 1`.

## Step 5 — Webhook destination, then one alert per probe

Create a webhook destination against the OpenObserve API. Do not click through a product UI you cannot version.

```bash
# template — row labels become the payload
curl -sS -u "$OO_USER:$OO_PASS" \
  -H 'Content-Type: application/json' \
  -X POST "http://openobserve.openobserve.svc:5080/api/default/alerts/templates" \
  -d '{
    "name": "homelab-webhook",
    "type": "http",
    "body": "{\"alert\":\"{alert_name}\",\"namespace\":\"{namespace}\",\"pod\":\"{pod}\",\"probe\":\"{probe}\",\"vantage_point\":\"{vantage_point}\",\"rows\":\"{rows}\"}"
  }'

# destination — ntfy shown; Pushover and Telegram are the same POST with a different URL/body
curl -sS -u "$OO_USER:$OO_PASS" \
  -H 'Content-Type: application/json' \
  -X POST "http://openobserve.openobserve.svc:5080/api/default/alerts/destinations" \
  -d '{
    "name": "ntfy",
    "type": "http",
    "url": "https://ntfy.sh/your-homelab-topic",
    "method": "post",
    "template": "homelab-webhook",
    "headers": {"Content-Type": "application/json"}
  }'
```

Set each alert's **row template** so `{namespace}`, `{pod}`, `{probe}`, and `{vantage_point}` come off the PromQL series labels. OpenObserve's built-in placeholders are `{alert_name}`, `{stream_name}`, `{rows}` — the four keys above are filled from the matching row, not from magic.

One scheduled PromQL (or SQL) alert per probe. Frequency 1–5 minutes. OpenObserve does not have Prometheus `for:`. `period` is lookback, and lengthening it makes a PromQL alert *more* willing to fire on a single blip, not less. Approximate pending with `min_over_time(...[5m:])` (or `max_over_time` when the bad state is `== 1`) so a 90-second rollout does not page. Two to five minutes is the window; pick 5m for deploys, 2m for "Postgres is gone."

### Severity

| Page now | Warn |
|---|---|
| `apps` Deployment available replicas = 0 while spec replicas > 0 | Single-sidecar / single-container CrashLoop in `apps` |
| Host Postgres down (`pg-host-local` and `pg-from-k3d` both fail) | Conntrack or `TIME_WAIT` > 80% of the pool (critical at 95%) |
| k3d path down while the host probe is up | Tailscale down — **unless** a probe's path actually goes over the tailnet, in which case page |

In this stack the app DSN is `host.k3d.internal`, not a Tailscale IP, so Tailscale-down is a warn (SSH, `logs.otterpond.dev`, CI). Page it only if you have pointed a probe at a tailnet address on purpose, or the k3d-path alert is firing and you already know Docker iptables vs Tailscale is the suspect.

## The five probes

Each one: why logs miss it, what to scrape, the PromQL, a 2–5 minute pending stand-in, and what to do when it fires.

### 1. CrashLoop, restarts, zero ready replicas

**Why logs miss it.** A container that never gets past the entrypoint may write nothing. A pod that is Running but not Ready looks healthy in Fluent Bit (it has a log file) and is not serving. OOMKilled is an exit reason, not a log line you can count on.

**Scrape.** kube-state-metrics. When an alert fires, pull the same window from logs and events — do not page on the metric alone and then guess:

```bash
kubectl -n apps get events --sort-by=.lastTimestamp | tail
kubectl -n apps logs <pod> --previous
```

```promql
# CrashLoopBackOff right now (pending: max_over_time 5m)
max by (namespace, pod, container) (
  max_over_time(
    kube_pod_container_status_waiting_reason{
      namespace="apps", reason="CrashLoopBackOff"
    }[5m]
  )
) == 1

# Restart storm
increase(kube_pod_container_status_restarts_total{namespace="apps"}[15m]) >= 3

# Asked for replicas, serving none
(
  kube_deployment_spec_replicas{namespace="apps"} > 0
)
and
(
  kube_deployment_status_replicas_available{namespace="apps"} == 0
)

# Running but not Ready
(
  kube_pod_status_phase{namespace="apps", phase="Running"} == 1
)
and on (namespace, pod)
(
  kube_pod_status_ready{namespace="apps", condition="true"} == 0
)

# OOMKilled split — last reason is sticky; require a recent restart
increase(kube_pod_container_status_restarts_total{namespace="apps"}[15m]) >= 1
and on (namespace, pod, container)
kube_pod_container_status_last_terminated_reason{
  namespace="apps", reason="OOMKilled"
} == 1
```

Zero available replicas on prod is a page. A single sidecar CrashLoop is a warn. OOMKilled is its own alert so you raise a memory limit instead of chasing a "crash."

**When it breaks.** Do not auto-delete or auto-restart a CrashLoop of a bad image. Kubernetes is already restarting it. Bounce-on-alert turns a bad tag into a tight loop you cannot read. Fix the image or the config, deploy, and let the pending window cover the rollout.

### 2. Postgres down vs not accepting queries

**Why logs miss it.** App logs say `connection refused`, `too many clients`, or `no route to host` — which is the client, not the verdict. TCP on `5432` can succeed while the postmaster is not accepting queries.

**Scrape.** `pg_up` from postgres_exporter (a real `SELECT 1`) and `pg_isready` from **two** places:

| Probe | Where it runs | Target |
|---|---|---|
| `pg-host-local` | Docker host | `127.0.0.1:5432` (or the socket) |
| `pg-from-k3d` | pod in `apps` | the app DSN, `host.k3d.internal:5432` |

```promql
# Host Postgres is down (both vantages dead) — page
min_over_time(pg_up{vantage_point="pg-host-local"}[2m]) == 0
and
min_over_time(pg_up{vantage_point="pg-from-k3d"}[2m]) == 0

# Same idea if you also export probe_pg_up from the Mac script
min_over_time(probe_pg_up{vantage_point="pg-host-local"}[2m]) == 0

# Accepting connections, nearly out of them
pg_stat_activity_count / pg_settings_max_connections > 0.8
```

Pair the metrics with a SQL alert on the log stream for `too many clients` / `no route to host`:

```sql
SELECT * FROM default
WHERE str_match(body, 'too many clients')
   OR str_match(body, 'no route to host')
   OR str_match(body, 'connection refused')
```

**When it breaks.** Host fail + k3d fail → Postgres or the Docker host. Work from the container out, the same ladder as the [PostgreSQL recipe](/homelab/postgres-and-backups):

```bash
docker ps --filter ancestor=postgres   # host Postgres containers on the default bridge
docker port <container>                # 5432 published?
pg_isready -h 127.0.0.1 -p 5432
```

Host ok + k3d fail is **not** this alert. That is probe 5.

### 3. Port / connection exhaustion on the Mac host

**Why logs miss it.** The failure shows up as `Cannot assign requested address` on whatever tried to open a socket — `cloudflared`, an app, a backup job. The cause is a pool on the Mac, usually after a crawler swarm, documented in the [cluster-host recipe](/homelab/cluster-host).

**Scrape.** node_exporter on Colima plus the Mac textfile:

```promql
# Linux / Colima side
node_nf_conntrack_entries / node_nf_conntrack_entries_limit > 0.8
node_sockstat_TCP_tw

# Mac side (textfile) — the incident that actually happened
mac_tcp_time_wait > 20000
(mac_ephemeral_port_last - mac_ephemeral_port_first) < 20000
```

Confirm the old-fashioned way when it fires:

```bash
netstat -an | grep TIME_WAIT | wc -l
sysctl net.inet.ip.portrange.first net.inet.ip.portrange.last
```

Warn above 80% of the pool (or `TIME_WAIT` in the tens of thousands). Critical at 95%. Pending 5 minutes — a crawler burst that drains and recovers should not page.

**When it breaks.** Widen the pool / conntrack. Do not reboot. `portrange.first=10000` and the `dev.otterpond.portrange` launchd plist are already the fix in [cluster-host](/homelab/cluster-host). Reboot clears `TIME_WAIT` and also clears everything else you were using to diagnose it.

### 4. Tailscale dropped

**Why logs miss it.** `tailscaled` can be running while `BackendState` is not `Running`, `Self.Online` is false, or a peer ping fails. Process-up is not mesh-up. `logs.otterpond.dev` is tailnet-only; a "can't reach the logging UI" ticket is often this, not OpenObserve.

**Scrape.** The Mac script (`tailscale status --json`, `tailscale ping` to a stable peer, utun present). Optional blackbox HTTP/TCP to a fixed tailnet IP or MagicDNS name. Do not alert only on the process.

```promql
max_over_time(probe_tailscale_backend_running[2m]) == 0
  or max_over_time(probe_tailscale_self_online[2m]) == 0
  or max_over_time(probe_tailscale_peer_ping[2m]) == 0
  or max_over_time(probe_tailscale_tun_up[2m]) == 0
```

Require the condition to hold for a couple of minutes. After any heal, re-probe for 30 seconds before you trust the page.

**When it breaks.** First move is the same reset as the [Tailscale recipe](/homelab/tailscale):

```bash
tailscale down
tailscale up --reset
sleep 30
tailscale status --json | python3 -c 'import json,sys; s=json.load(sys.stdin); print(s.get("BackendState"), (s.get("Self") or {}).get("Online"))'
```

If you are off the tailnet and need onto the box, use the LAN IP fallback in that recipe. Restarting `tailscaled` is the next step, not the first.

### 5. k3d cannot reach Postgres on the Docker host

**Why logs miss it.** Every app in `apps` logs a database timeout at once. That looks like Postgres. It is usually the path: stale or missing `host.k3d.internal` in CoreDNS NodeHosts, a k3d network recreate, CoreDNS itself, or Docker iptables fighting Tailscale. k3d start can hang on hostAliases injection and still come up `Ready` with NodeHosts wrong — treat "nodes are Ready" as unrelated.

**Scrape.** The two vantages from probe 2. This is a **first-class alert** with its own name. It is not "Postgres is down."

```promql
# Host can SELECT 1; the cluster cannot — page as k3d-path
min_over_time(pg_up{vantage_point="pg-host-local"}[2m]) == 1
and
max_over_time(pg_up{vantage_point="pg-from-k3d"}[2m]) == 0
```

Optional third check, from a debug pod, to see *which* path died — Docker-bridge / `host.k3d.internal` vs a Tailscale IP (only useful if you published `5432` on the tailnet, which this cookbook does not):

```bash
# resolve, then bypass DNS with the raw IP
kubectl -n apps exec deploy/pg-probe -c pg-isready -- \
  getent hosts host.k3d.internal
kubectl -n kube-system get configmap coredns -o yaml | grep -A20 NodeHosts

# from a node, does the VM IP still match?
docker exec k3d-homelab-server-0 cat /etc/hosts | grep host.k3d
colima ssh -- ip -4 addr show eth0 | grep inet
```

**When it breaks.** Follow the stale-`host.k3d.internal` steps in the [cluster-host recipe](/homelab/cluster-host). Do not restart Postgres and do not rebuild the cluster as the first move. If NodeHosts is missing the name after a start that hung on hostAliases injection, fix CoreDNS / the node `/etc/hosts` entries; the cluster being Ready does not mean the injection finished.

## Optional later: heal, with blast radius

An OpenObserve Action or an n8n / Keep / GoAlert step can restart `tailscaled`, or bounce CoreDNS, once you are tired of doing it by hand. Write the blast radius on the action before you enable it:

- **Restart tailscaled** — drops every tailnet client for a few seconds, including CI and your SSH session if that was the path in. Have the LAN IP first.
- **Bounce k3d DNS** (`kubectl -n kube-system rollout restart deploy/coredns`) — brief name-resolution blips for the whole cluster. Fine. Recreating the k3d network or the cluster is not fine as an automated heal.
- **CrashLoop** — must not auto-delete pods and must not auto-roll a Deployment. You will delete the evidence and still have the bad image.

v1 is webhook only. Add heal when a specific alert has woken you more than once and the action is a one-liner you already trust.

## When it breaks

### Symptom: metrics never show up in OpenObserve

Work along the pipeline. Is Prometheus running, and can it see the targets?

```bash
kubectl -n monitoring get pods
kubectl -n monitoring logs -l app.kubernetes.io/name=prometheus --all-containers | grep -i -e remote -e error
```

Remote-write 401/403 is the OpenObserve secret in `monitoring`. Connection refused is the service URL — org name, namespace, or port. Targets stuck down are almost always `host.k3d.internal` (same failure as Postgres) or a published port you forgot (`9100`, `9187`).

### Symptom: host Postgres is fine, every `apps` pod pages anyway

That is probe 5, not probe 2. If you only built a `pg_up == 0` alert with no `vantage_point`, you will page "Postgres down" for a DNS problem. Split the vantages before you add more signals.

### Symptom: alerts fire on every deploy

Pending is too short, or you used OpenObserve `period` as if it were Prometheus `for:`. Wrap the condition in `min_over_time` / `max_over_time` over 2–5 minutes. Frequency stays 1–5 minutes; the range vector is what eats the rollout.

### Symptom: kube-system CrashLoops wake you

The PromQL is missing `namespace="apps"`. kube-state-metrics sees the whole cluster on purpose. The label in Step 1 is useless unless the alerts use it.

### Symptom: Tailscale alert fires, database path is fine

Correct, for this cookbook. The app does not reach Postgres over the tailnet. Silence or downgrade that destination to warn. If `logs.otterpond.dev` is how you debug, you will feel this one anyway — you cannot open the UI until the [Tailscale reset](/homelab/tailscale) lands.

### Symptom: `Cannot assign requested address` in app logs, exporters look green

The Mac textfile probe is not running or node_exporter is not reading `$HOME/homelab/probes`. Check `mac_tcp_time_wait` exists, then go back to the [cluster-host ephemeral-port step](/homelab/cluster-host). The Linux conntrack ratio can be calm while macOS `TIME_WAIT` is not.
