# Multi-Org Monitoring Deployment — Per-Org Prometheus, Grafana & Loki

> Deployment guide for a **fully per-Organization-isolated** monitoring stack built on the Multi-Org Exporter (see [README.md](README.md) for the overview). Each tenant (Org) gets its own Prometheus + Grafana (metrics) and its own Loki (events), all fed from per-Org exporter containers.
>
> Validated on **WEKA v4.4.33** (cluster `smith-25`). The general concept doc is [README.md](README.md); the single-host exporter validation is [validation-report-v4.4.33.md](validation-report-v4.4.33.md).

## 1. Goal

Prove that per-Org observability can be isolated end to end: each Org's metrics **and** events are collected, stored, and visualized on separate, dedicated infrastructure — one tenant cannot see another's data.

## 2. Node Layout

| Node | Role | Services (Docker) | Ports |
|---|---|---|---|
| backend | WEKA cluster mgmt | — | 14000 (mgmt API) |
| **client-0** | Exporters (both Orgs) | `export-org1`, `export-org2` | 8001, 8002 |
| **client-1** | org1 metrics | `prometheus`, `grafana` | 9090, 3000 |
| **client-2** | org2 metrics | `prometheus`, `grafana` | 9090, 3000 |
| **client-3** | org1 events | `loki` | 3100 |
| **client-4** | org2 events | `loki` | 3100 |

Placeholders below: `<MGMT>` = backend mgmt IP, `<C0>`…`<C4>` = client-0…4 IPs.

## 3. Data Flow Architecture

```
                         exporter (client-0)
                    ┌──────────────────────────────┐
                    │ export-org1 :8001            │
                    │ export-org2 :8002            │
                    └───┬───────────────────┬──────┘
        [metrics: PULL] │                   │ [events: PUSH]
   Prometheus scrapes   │                   │ exporter pushes (events_to_loki)
        org1│      org2 │                   │ org1│        org2│
  ┌─────────▼──┐  ┌─────▼──────┐      ┌──────▼───┐   ┌─────▼────┐
  │ Prometheus │  │ Prometheus │      │ Loki     │   │ Loki     │
  │ (client-1) │  │ (client-2) │      │(client-3)│   │(client-4)│
  └─────┬──────┘  └─────┬──────┘      └────┬─────┘   └────┬─────┘
        │ datasource    │ datasource       │ datasource   │ datasource
  ┌─────▼──────┐  ┌─────▼──────┐           │              │
  │ Grafana    │  │ Grafana    │           │              │
  │ (client-1) │◄─┼─ Loki(client-3) ───────┘              │
  │ org1       │  │ Grafana(client-2) ◄─ Loki(client-4) ──┘
  └────────────┘  └─────(org2)──┘
```

- **Metrics**: Prometheus (client-1/2) *pulls* from the exporter (`<C0>:8001` / `<C0>:8002`); Grafana reads its local Prometheus.
- **Events**: exporter (client-0) *pushes* WEKA cluster events to the Org's Loki (client-3/4); each Org's Grafana adds that Loki as a data source.

## 4. Deployment

All nodes have Docker + Docker Compose. The Prometheus/Grafana/Loki nodes use the configs from the `weka-mon` package (extracted to `/opt/weka-mon`); only the few values below are changed.

### 4.1 Loki — client-3 (org1) and client-4 (org2)

`/opt/weka-mon/etc_loki/loki-config.yaml` ships with the package (`auth_enabled: false`, port 3100). Run only the Loki service:

```yaml
# /opt/weka-mon/docker-compose-loki.yml
services:
  loki:
    image: grafana/loki:2.8.10
    container_name: weka-loki
    volumes:
      - ${PWD}/etc_loki:/etc/loki
      - ${PWD}/loki_data:/loki
    command: -config.file=/etc/loki/loki-config.yaml
    restart: always
    ports: ["3100:3100"]
```

```bash
cd /opt/weka-mon
mkdir -p loki_data && chmod 777 loki_data      # data dir writable by the loki container
docker compose -f docker-compose-loki.yml up -d
curl -s http://localhost:3100/ready            # -> "ready"
```

### 4.2 Exporters — client-0 (both Orgs)

Generate per-Org tokens and point each Org's exporter at its Org's Loki:

```bash
mkdir -p /opt/weka-mon/.weka
weka user login org1admin '<PW>' --org org1 -H <MGMT> --path /opt/weka-mon/.weka/org1-authtoken.json
weka user login org2admin '<PW>' --org org2 -H <MGMT> --path /opt/weka-mon/.weka/org2-authtoken.json
chmod 644 /opt/weka-mon/.weka/*.json
```

`export-org1.yml` (org2 differs only by `auth_token_file` and `loki.host`):

```yaml
exporter:
  listen_port: 8001
  events_to_loki: true          # push events to this Org's Loki
  events_to_syslog: false
  backends_only: true
  # ... (timeout/procs/threads as default)
loki:
  host: <C3>                     # org1 Loki (client-3); org2 -> <C4> (client-4)
  port: 3100
  protocol: http                 # Loki is auth_enabled:false / http
  path: /loki/api/v1/push
  user: null
  password: null
  org_id: null
cluster:
  auth_token_file: org1-authtoken.json   # org2 -> org2-authtoken.json
  hosts: [ <MGMT> ]
  mgmt_port: 14000
  force_https: false
  verify_cert: false
stats:
  cpu: { CPU_UTILIZATION: "%" }
  ops:
    OPS: ops
    READS: iops
    READ_BYTES: bytespersec
    READ_DURATION: microsecs
    READ_LATENCY: microsecs
    THROUGHPUT: bytespersec
    WRITES: iops
    WRITE_BYTES: bytespersec
    WRITE_LATENCY: microsecs
  ops_driver:
    READ_SIZES: sizes
    WRITE_SIZES: sizes
```

```yaml
# /opt/weka-mon/docker-compose-export.yml
services:
  export-org1:
    image: wekasolutions/export:20260216
    container_name: weka-export-org1
    user: "0:0"
    volumes:
      - ${PWD}/.weka:/weka/.weka
      - /etc/hosts:/etc/hosts
      - ${PWD}/export-org1.yml:/weka/export.yml
    command: ["-v","-c","/weka/export.yml"]
    restart: always
    ports: ["8001:8001"]
  export-org2:
    image: wekasolutions/export:20260216
    container_name: weka-export-org2
    user: "0:0"
    volumes:
      - ${PWD}/.weka:/weka/.weka
      - /etc/hosts:/etc/hosts
      - ${PWD}/export-org2.yml:/weka/export.yml
    command: ["-v","-c","/weka/export.yml"]
    restart: always
    ports: ["8002:8001"]
```

```bash
cd /opt/weka-mon && docker compose -f docker-compose-export.yml up -d
curl -s http://localhost:8001/metrics | grep '^weka_fs'   # org1
curl -s http://localhost:8002/metrics | grep '^weka_fs'   # org2
```

### 4.3 Prometheus + Grafana — client-1 (org1) and client-2 (org2)

**Prometheus** — scrape only this Org's exporter:

```yaml
# /opt/weka-mon/etc_prometheus/prometheus.yml
global:
  scrape_interval: 60s
scrape_configs:
  - job_name: weka
    static_configs:
      - targets: ["<C0>:8001"]    # client-1=org1; client-2 -> <C0>:8002
        labels:
          org: "org1"             # client-2 -> "org2"
```

**Grafana** — the package provisions a `Prometheus` and a `Loki` data source plus the WEKA dashboards. Point the Loki data source at this Org's Loki node:

```yaml
# /opt/weka-mon/etc_grafana/provisioning/datasources/loki.yml
apiVersion: 1
datasources:
- name: Loki
  type: loki
  access: proxy
  url: http://<C3>:3100          # client-1=org1 Loki (client-3); client-2 -> http://<C4>:3100
  isDefault: false
```

(The `Prometheus` data source stays `http://prometheus:9090` — same Compose network.)

```yaml
# /opt/weka-mon/docker-compose-mon.yml
services:
  prometheus:
    image: prom/prometheus:v2.50.1
    container_name: weka-prometheus
    volumes:
      - ${PWD}/etc_prometheus:/etc/prometheus
      - ${PWD}/prometheus_data:/prometheus
    command: --config.file=/etc/prometheus/prometheus.yml --storage.tsdb.retention.size=20GB
    restart: always
    ports: ["9090:9090"]
  grafana:
    image: grafana/grafana:10.2.5
    container_name: weka-grafana
    volumes:
      - ${PWD}/etc_grafana/:/etc/grafana/
      - ${PWD}/var_lib_grafana/:/var/lib/grafana/
      - ${PWD}/usr_share_grafana_public_img:/usr/share/grafana/public/img/weka
    restart: always
    ports: ["3000:3000"]
```

```bash
cd /opt/weka-mon
mkdir -p prometheus_data && chmod 777 prometheus_data
chmod -R 777 var_lib_grafana
docker compose -f docker-compose-mon.yml up -d
```

Grafana: `http://<C1>:3000` (org1) / `http://<C2>:3000` (org2) — default login `admin` / `admin`. The bundled dashboards (`Weka_FS`, `Weka_Cluster_Overview`, `Weka_Exporter`, `Weka_Logs`, …) load automatically; `Weka_Logs` reads the Org's Loki.

## 5. Verification (smith-25, WEKA v4.4.33)

With an FIO workload running (org1 random read, org2 random write):

**Metrics (Prometheus):**

| Node | Prometheus target | `count(weka_fs_stats)` | sample |
|---|---|---|---|
| client-1 (org1) | `<C0>:8001` — **up** | **6** | `weka_fs_stats{fs_name="org1fs1",stat="READS"}` ≈ 43,473 iops, THROUGHPUT ≈ 2.85 GB/s |
| client-2 (org2) | `<C0>:8002` — **up** | **6** | `weka_fs_stats{fs_name="org2fs1",stat="WRITES"}` (write side) |

**Grafana:** both healthy (v10.2.5); data sources = `Prometheus` (local) + `Loki` (this Org's Loki node), verified via `/api/datasources`.

**Events (Loki):** both `client-3` and `client-4` returned labels `category, cluster, event_type, severity, source` from `/loki/api/v1/labels` — confirming the exporter is pushing WEKA events to each Org's Loki.

**Isolation:** client-1/Grafana sees only org1 (metrics from client-1 Prometheus, events from client-3 Loki); client-2/Grafana sees only org2 (client-2 Prometheus + client-4 Loki). Neither tenant's stack references the other's data.

## 6. Notes & Operations

- **Order of bring-up:** Loki (client-3/4) first → exporters (client-0, which push to those Loki) → Prometheus+Grafana (client-1/2).
- **Permissions:** `loki_data`, `prometheus_data`, and `var_lib_grafana` must be writable by their container users (`chmod 777` shown for a test cluster).
- **Token perms:** exporter token files at `chmod 644` (compose runs containers as root via `user: "0:0"`, so either works).
- **Network:** Prometheus(client-1/2) → exporter(client-0) ports; exporter(client-0) → Loki(client-3/4):3100 must be reachable.
- **Scaling to more Orgs:** add an `export-orgN` (next port) on client-0 and a dedicated Prometheus/Grafana + Loki node pair per Org.
