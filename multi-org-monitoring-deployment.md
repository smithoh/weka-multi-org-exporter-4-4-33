# Multi-Org Monitoring Deployment Runbook — External (Native) Prometheus, Grafana & Loki

> Step-by-step runbook for a **fully per-Organization-isolated** monitoring stack for the Multi-Org Exporter (overview: [README.md](README.md)).
>
> **"External" = natively installed on the OS** (Grafana via the official yum repo; Prometheus and Loki via official release binaries + `systemd`) — **not** the bundled `weka-mon` Docker stack. Only the WEKA **exporter** stays as a container.
>
> Validated on **WEKA v4.4.33** (cluster `smith-25`). Exporter validation: [validation-report-v4.4.33.md](validation-report-v4.4.33.md).

## 1. Node Layout

| Node | Role | Software (install method) | Listen |
|---|---|---|---|
| backend | WEKA cluster mgmt | — | 14000 |
| **client-0** | Exporters (both Orgs) | `wekasolutions/export` (**Docker**) | 8001, 8002 |
| **client-1** | org1 metrics | **Prometheus** (binary) + **Grafana** (yum) | 9090, 3000 |
| **client-2** | org2 metrics | **Prometheus** (binary) + **Grafana** (yum) | 9090, 3000 |
| **client-3** | org1 events | **Loki** (binary) | 3100 |
| **client-4** | org2 events | **Loki** (binary) | 3100 |

Placeholders: `<MGMT>` backend mgmt IP, `<C0>`…`<C4>` = client-0…4 IPs. Example (smith-25, private IPs): `<MGMT>=10.0.100.10`, `<C0>=10.0.100.6`, `<C1>=10.0.100.9`, `<C2>=10.0.100.8`, `<C3>=10.0.100.5`, `<C4>=10.0.100.4`.

## 2. Data Flow

```
                         exporter (client-0, Docker)
                    ┌──────────────────────────────┐
                    │ export-org1 :8001            │
                    │ export-org2 :8002            │
                    └───┬───────────────────┬──────┘
        [metrics: PULL] │                   │ [events: PUSH]
        org1│      org2 │                   │ org1│        org2│
  ┌─────────▼──┐  ┌─────▼──────┐      ┌──────▼───┐   ┌─────▼────┐
  │ Prometheus │  │ Prometheus │      │ Loki     │   │ Loki     │
  │ (client-1) │  │ (client-2) │      │(client-3)│   │(client-4)│
  └─────┬──────┘  └─────┬──────┘      └────┬─────┘   └────┬─────┘
  ┌─────▼──────┐  ┌─────▼──────┐           │              │
  │ Grafana    │  │ Grafana    │           │              │
  │ (client-1) │◄─┼─ Loki(client-3) ───────┘              │
  │ org1       │  │ Grafana(client-2) ◄─ Loki(client-4) ──┘
  └────────────┘  └─────(org2)──┘
```

- **Metrics**: Prometheus *pulls* from the exporter (`<C0>:8001` / `<C0>:8002`); Grafana reads its local Prometheus (`http://localhost:9090`).
- **Events**: the exporter *pushes* WEKA events to its Org's Loki; each Org's Grafana adds that Loki as a data source.

Prerequisite on client-1..4: a recent OS with internet egress; `curl`, `tar`, `unzip` (install `unzip` if missing).

---

## 3. Loki (native binary) — client-3 (org1) & client-4 (org2)

```bash
command -v unzip >/dev/null || dnf install -y unzip
cd /tmp
curl -sSL -o loki-linux-amd64.zip \
  https://github.com/grafana/loki/releases/download/v2.8.10/loki-linux-amd64.zip
unzip -oq loki-linux-amd64.zip
install -m 0755 loki-linux-amd64 /usr/local/bin/loki
useradd --system --no-create-home --shell /sbin/nologin loki     # idempotent

# config: reuse the weka-mon loki-config.yaml (auth_enabled:false, http :3100, path_prefix /loki)
mkdir -p /etc/loki /loki/chunks /loki/rules
cp /opt/weka-mon/etc_loki/loki-config.yaml /etc/loki/loki-config.yaml
chown -R loki:loki /loki /etc/loki
```

**Edited parameter:** none inside `loki-config.yaml` (used as shipped). The only change vs the Docker stack is the **data path is real host dirs** (`/loki/...`, owned by the `loki` user) instead of a Docker volume.

`/etc/systemd/system/loki.service`:

```ini
[Unit]
Description=Grafana Loki
After=network-online.target
Wants=network-online.target
[Service]
User=loki
Group=loki
ExecStart=/usr/local/bin/loki -config.file=/etc/loki/loki-config.yaml
Restart=always
[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload && systemctl enable --now loki
curl -s http://localhost:3100/ready      # -> "ready" (after ~15s warmup)
```

Run identically on client-3 and client-4.

---

## 4. Exporters (Docker) — client-0

One exporter per Org, each pointing at its Org's Loki (`<C3>` for org1, `<C4>` for org2).

```bash
mkdir -p /opt/weka-mon/.weka
weka user login org1admin '<PW>' --org org1 -H <MGMT> --path /opt/weka-mon/.weka/org1-authtoken.json
weka user login org2admin '<PW>' --org org2 -H <MGMT> --path /opt/weka-mon/.weka/org2-authtoken.json
chmod 644 /opt/weka-mon/.weka/*.json
```

**`export-org1.yml`** — edited parameters in **bold** intent (org2 differs only in `auth_token_file` and `loki.host`):

```yaml
exporter:
  listen_port: 8001
  events_to_loki: true        # <-- enable event push
  events_to_syslog: false
  backends_only: true
loki:
  host: <C3>                   # <-- org1 Loki (client-3); org2 -> <C4>
  port: 3100
  protocol: http               # <-- Loki is http / auth_enabled:false
  path: /loki/api/v1/push
  user: null
  password: null
  org_id: null
cluster:
  auth_token_file: org1-authtoken.json   # <-- org2 -> org2-authtoken.json
  hosts: [ <MGMT> ]
  mgmt_port: 14000
  force_https: false
  verify_cert: false
stats:                         # <-- REQUIRED (missing -> KeyError: 'stats')
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

`docker-compose-export.yml` (both Org services, `user: "0:0"` so 644 tokens are readable):

```yaml
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
curl -s http://localhost:8001/metrics | grep '^weka_fs'    # org1
```

---

## 5. Prometheus (native binary) — client-1 (org1) & client-2 (org2)

```bash
V=2.50.1
cd /tmp
curl -sSL -o prom.tar.gz \
  https://github.com/prometheus/prometheus/releases/download/v${V}/prometheus-${V}.linux-amd64.tar.gz
tar xzf prom.tar.gz && cd prometheus-${V}.linux-amd64
install -m 0755 prometheus promtool /usr/local/bin/
useradd --system --no-create-home --shell /sbin/nologin prometheus     # idempotent
mkdir -p /etc/prometheus/consoles /etc/prometheus/console_libraries /var/lib/prometheus
cp -r consoles/* /etc/prometheus/consoles/
cp -r console_libraries/* /etc/prometheus/console_libraries/
```

**Edited parameter — `/etc/prometheus/prometheus.yml`** (the only per-Org change is the scrape `targets` + `org` label):

```yaml
global:
  scrape_interval: 60s
scrape_configs:
  - job_name: weka
    static_configs:
      - targets: ["<C0>:8001"]    # client-1 = org1 ; client-2 -> "<C0>:8002"
        labels:
          org: "org1"             # client-2 -> "org2"
```

```bash
chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus
```

`/etc/systemd/system/prometheus.service`:

```ini
[Unit]
Description=Prometheus
After=network-online.target
Wants=network-online.target
[Service]
User=prometheus
Group=prometheus
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries
Restart=always
[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload && systemctl enable --now prometheus
curl -s http://localhost:9090/api/v1/targets | grep -o '"health":"up"'   # target up
```

---

## 6. Grafana (native, yum) — client-1 (org1) & client-2 (org2)

```bash
cat > /etc/yum.repos.d/grafana.repo <<'R'
[grafana]
name=grafana
baseurl=https://rpm.grafana.com
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://rpm.grafana.com/gpg.key
sslverify=1
R
dnf install -y grafana
```

**Edited parameters — provisioning files** (this is where each Org's data sources and dashboards are wired):

`/etc/grafana/provisioning/datasources/weka.yml` — Prometheus is local; **Loki points at this Org's Loki node**:

```yaml
apiVersion: 1
datasources:
- name: Prometheus
  type: prometheus
  access: proxy
  url: http://localhost:9090
  isDefault: true
- name: Loki
  type: loki
  access: proxy
  url: http://<C3>:3100        # client-1 = org1 Loki (client-3); client-2 -> http://<C4>:3100
  isDefault: false
```

`/etc/grafana/provisioning/dashboards/weka.yml` — auto-load the WEKA dashboards:

```yaml
apiVersion: 1
providers:
- name: weka
  type: file
  options:
    path: /var/lib/grafana/dashboards
```

```bash
# reuse the 10 WEKA dashboards shipped in the weka-mon package
mkdir -p /var/lib/grafana/dashboards
cp /opt/weka-mon/var_lib_grafana/dashboards/*.json /var/lib/grafana/dashboards/
chown -R grafana:grafana /var/lib/grafana /etc/grafana/provisioning

systemctl daemon-reload && systemctl enable --now grafana-server
# set admin password (default admin/admin -> Passw0rd!) via API once it's up:
curl -s -X PUT -H "Content-Type: application/json" \
  -d '{"oldPassword":"admin","newPassword":"Passw0rd!","confirmNew":"Passw0rd!"}' \
  http://admin:admin@localhost:3000/api/user/password
```

> **Note:** `grafana-cli admin reset-admin-password` did **not** take in this run; the reliable method is the `/api/user/password` API call above (default login `admin`/`admin`).

Grafana UI: `http://<C1>:3000` (org1) / `http://<C2>:3000` (org2), login `admin` / `Passw0rd!`. The provisioned data sources (`Prometheus` local + `Loki` this Org) and 10 dashboards (`Weka Cluster Overview`, `Weka Filesystems Detail`, `Weka Metrics Exporter Stats`, `Weka Logs`, …) load automatically.

---

## 7. Verification (smith-25, WEKA v4.4.33)

| Check | client-1 (org1) | client-2 (org2) |
|---|---|---|
| Prometheus target | `<C0>:8001` **up** | `<C0>:8002` **up** |
| `count(weka_fs)` | 6 | 6 |
| Grafana | healthy (v13.1.0), **10 dashboards**, datasources = Prometheus(local)+Loki | same |
| Loki labels (`/loki/api/v1/labels`) | from client-3: `category, cluster, event_type, severity, source` | from client-4: same |

With an FIO workload (org1 random read / org2 random write), `weka_fs_stats` populates per Org in each Prometheus, and the exporter's events appear in each Org's Loki — visible in that Org's Grafana (metrics dashboards + `Weka Logs`). Neither Org's stack references the other's data.

## 8. Operations Notes

- **Bring-up order:** Loki (client-3/4) → exporters (client-0, push to those Loki) → Prometheus + Grafana (client-1/2).
- **Service mgmt (native):** `systemctl status|restart loki|prometheus|grafana-server`; logs via `journalctl -u <svc>`.
- **Exporter (Docker):** `docker compose -f /opt/weka-mon/docker-compose-export.yml {up -d|down}`.
- **Versions used:** Loki 2.8.10, Prometheus 2.50.1, Grafana 13.1.0 (yum latest), exporter image `20260216`.
- **Data dirs / perms:** `/loki` owned by `loki`; `/var/lib/prometheus` owned by `prometheus`; `/var/lib/grafana` owned by `grafana`.
- **Network:** Prometheus(C1/C2)→exporter(C0):8001/8002; exporter(C0)→Loki(C3/C4):3100 must be reachable.
- **Scaling:** add `export-orgN` (next port) on client-0 + a dedicated Prometheus/Grafana + Loki node per new Org.
