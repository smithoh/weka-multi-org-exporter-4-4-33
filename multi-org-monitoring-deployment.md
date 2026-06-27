# Multi-Org Monitoring Deployment Runbook — External (Native) Prometheus, Grafana & Loki

> Step-by-step runbook for a **fully per-Organization-isolated** monitoring stack for the Multi-Org Exporter (overview: [README.md](README.md)).
>
> **"External" = natively installed on the OS** (Grafana via the official yum repo; Prometheus and Loki via official release binaries + `systemd`) — **not** the bundled `weka-mon` Docker stack. Only the WEKA **exporter** stays as a container.
>
> Validated on **WEKA v4.4.33** (cluster `smith-25`).

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

## 3. Deployment Procedure (overview)

Deploy in this order — each step produces what the next one consumes:

1. **Loki** — events store (client-3/4, §4). Brought up first so the exporter has a live push target.
2. **Exporters** — one container per Org (client-0, §5). The core data source: it pulls per-Org metrics from the WEKA cluster and pushes events to its Org's Loki. Everything downstream reads from here.
3. **Prometheus** — scrapes each Org's exporter endpoint (`<C0>:8001` / `:8002`) (client-1/2, §6).
4. **Grafana** — reads its local Prometheus + its Org's Loki and loads the dashboards (client-1/2, §7).
5. **Verify** per-Org isolation (§8); optionally **enable extra metrics / build custom dashboards** (§9).

> The exporter (client-0) is the heart of the stack — both metrics and events originate there. Loki is installed before it only because the exporter needs an existing push endpoint; if you start the exporter first, event pushes harmlessly retry until Loki is ready. Prometheus and Grafana must come after the exporter, since they have nothing to scrape/show until it is running.

---

## 4. Loki (native binary) — client-3 (org1) & client-4 (org2)

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

## 5. Exporters (Docker) — client-0

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

## 6. Prometheus (native binary) — client-1 (org1) & client-2 (org2)

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

## 7. Grafana (native, yum) — client-1 (org1) & client-2 (org2)

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

## 8. Verification (smith-25, WEKA v4.4.33)

| Check | client-1 (org1) | client-2 (org2) |
|---|---|---|
| Prometheus target | `<C0>:8001` **up** | `<C0>:8002` **up** |
| `count(weka_fs)` | 6 | 6 |
| Grafana | healthy (v13.1.0), **10 dashboards**, datasources = Prometheus(local)+Loki | same |
| Loki labels (`/loki/api/v1/labels`) | from client-3: `category, cluster, event_type, severity, source` | from client-4: same |

With an FIO workload (org1 random read / org2 random write), `weka_fs_stats` populates per Org in each Prometheus, and the exporter's events appear in each Org's Loki — visible in that Org's Grafana (metrics dashboards + `Weka Logs`). Neither Org's stack references the other's data.

## 9. Worked example — enable an extra metric and surface it on a dashboard

Enabling a stat in `stats:` makes the exporter/Prometheus **collect** it, but it only **appears on a dashboard if a panel queries it**. The bundled dashboards query only `cpu` / `ops` / `ops_driver(RDMA)` / `ops_nfs` from `weka_stats` (plus `weka_fs`, `weka_overview_*`, etc.) — they do **not** query `category=ssd` or `category=network`. So enabling those alone changes nothing on screen; you must add a panel. Full A→B example using `network / PUMPS_TXQ_FULL`.

### 9.1 (A) Enable the metric in the exporter config

On client-0, add the category/stat to each `export-orgN.yml` `stats:` section:

```yaml
stats:
  # ... existing categories ...
  network:
    PUMPS_TXQ_FULL: times
```

Restart and verify it reaches the exporter — note `PUMPS_TXQ_FULL` only increments under **network TX-queue pressure**, so it needs an active workload to show non-zero (with no I/O the series may be absent):

```bash
cd /opt/weka-mon && docker compose -f docker-compose-export.yml restart
curl -s http://localhost:8001/metrics | grep PUMPS_TXQ_FULL          # appears under load
# confirm in Prometheus (on client-1/2):
curl -s -G http://localhost:9090/api/v1/query \
  --data-urlencode 'query=count(weka_stats{category="network",stat="PUMPS_TXQ_FULL"})'
```

### 9.2 (B) Add a custom panel to a provisioned dashboard

Provisioned dashboards are JSON files under `/var/lib/grafana/dashboards/` (here `Weka_Cluster_Overview.json`, uid `WekaOverview`). Append a panel to the top-level `panels` array; Grafana re-imports on reload.

Panel essentials:
- `datasource: {"uid": "$weka_datasource"}` — reuse the dashboard's datasource template variable (don't hardcode a name)
- `gridPos.y` = below the current bottom panel (so it lands at the end)
- `targets[].expr` = the new metric, using the dashboard's `$cluster` variable

```json
{ "id": 109, "type": "row", "title": "Network (custom)", "collapsed": false,
  "gridPos": {"h":1,"w":24,"x":0,"y":159} },
{ "id": 110, "type": "timeseries", "title": "Network: PUMPS_TXQ_FULL by host (custom)",
  "datasource": {"uid": "$weka_datasource"},
  "gridPos": {"h":9,"w":24,"x":0,"y":160},
  "targets": [{ "refId":"A", "datasource":{"uid":"$weka_datasource"},
    "expr":"weka_stats{cluster=\"$cluster\", category=\"network\", stat=\"PUMPS_TXQ_FULL\"}",
    "legendFormat":"{{host_name}} {{node_role}}" }] }
```

Apply on each Grafana node (client-1/2):

```bash
chown grafana:grafana /var/lib/grafana/dashboards/Weka_Cluster_Overview.json
systemctl restart grafana-server      # or wait for the file provider to reload
```

> Rather than hand-editing a 60+ panel JSON, we used a small Python script that loads the file, computes the next `id` and the current bottom `y` (max `gridPos.y + h`), appends the row + timeseries, and writes it back. This avoids breaking the existing layout.

> **CLI (file edit) vs GUI — which to use (Grafana 11+/13 note, validated on smith-25).**
>
> - **For a permanent, provisioned dashboard, you MUST use the CLI / file edit above.** The files under `/var/lib/grafana/dashboards/` are the source of truth and use the **legacy** dashboard schema (top-level `panels[]` + `schemaVersion`). Editing that legacy JSON and reloading is the only way to permanently change a provisioned dashboard across all nodes.
> - **The GUI is read-only for provisioned dashboards.** You can build/preview a panel in the UI, but **Save is blocked** (*"cannot be saved … provisioned"*); the Save dialog only offers Copy/Export JSON.
> - **GUI Export is NOT a drop-in replacement for the file on Grafana 13.** Once a dashboard is opened/edited in Grafana 13 it is migrated in-memory to the **new v2 schema** (`elements` / `layout` / `vizConfig`, no `panels[]`). **Both** export models — *V2 Resource* **and** *Classic* — emit that new schema; the toggle only changes the outer resource envelope, not the body. Such a file does **not** match the legacy schema the provisioned files use, so dropping it into `/var/lib/grafana/dashboards/` is unreliable. → To update a provisioned dashboard permanently, edit the legacy JSON via CLI (above), **not** via GUI export.
> - **GUI Import is fine for a quick / per-node / experimental copy.** Dashboards → New → Import the JSON under a **new name + uid** (the provisioned uid `WekaOverview` is reserved, so import creates a *separate* dashboard). The copy lives in Grafana's DB (`provisioned: false`), is freely editable, exists only on that node, and leaves the provisioned original untouched — verified: imported `Weka Cluster Overview (new imported test)` showed the custom `PUMPS_TXQ_FULL` panel while the provisioned original kept its 63 panels unchanged.

### 9.3 Verify on the dashboard

```bash
# panel present in the dashboard model:
curl -s "http://admin:Passw0rd!@localhost:3000/api/dashboards/uid/WekaOverview" | grep -c "PUMPS_TXQ_FULL"
# data queryable (per Org):
curl -s -G http://localhost:9090/api/v1/query \
  --data-urlencode 'query=count(weka_stats{category="network",stat="PUMPS_TXQ_FULL"})'
```

Grafana → **Weka Cluster Overview → scroll to bottom → "Network (custom)"** shows `PUMPS_TXQ_FULL` per host (rises under workload). Validated on smith-25: panel present on both Org Grafanas, **20 series per Org**, values ~0.5–3 M `times` during a heavy FIO run.

**Takeaway:** dashboard visibility = (1) `stats:` enables collection → (2) Prometheus scrapes → (3) **a panel must query it**. Steps 1–2 are necessary but not sufficient; step 3 is what puts the metric on screen. (This is why enabling `ssd` alone showed nothing — no bundled panel queries it.)

### 9.4 Build a brand-new custom dashboard (per-Org filesystem I/O)

§9.2 adds a panel to an *existing* dashboard. This builds a **standalone dashboard** from scratch — the right approach for the multi-Org headline view: **per-filesystem I/O from `weka_fs_stats`** (the metric that has `fs_id`/`fs_name` and is **not** used by any bundled dashboard). The *same* dashboard JSON is deployed to both Org Grafanas; each shows only its own Org's filesystem because each reads only its own Org's Prometheus.

Provision it like any other dashboard: drop a JSON file into `/var/lib/grafana/dashboards/` (the file provider in §7 auto-loads it).

Dashboard essentials:
- Give it a unique `uid` (e.g. `perorgfsio`) and a `title`.
- Add a `datasource` **template variable** of `type: datasource`, `query: prometheus` (named `weka_datasource`, matching the bundled dashboards), and reference it in every panel/target as `{"uid": "$weka_datasource"}`. This auto-binds to the provisioned Prometheus without hardcoding a uid.
- One `timeseries` panel per metric family, querying `weka_fs_stats` by `fs_name`.

Panels and queries:

| Panel | unit | targets (`expr` → `legendFormat`) |
|---|---|---|
| IOPS by filesystem | `iops` | `weka_fs_stats{stat="READS"}` → `{{fs_name}} READS` ; `weka_fs_stats{stat="WRITES"}` → `{{fs_name}} WRITES` |
| Throughput by filesystem | `Bps` | `weka_fs_stats{stat="READ_BYTES"}` ; `weka_fs_stats{stat="WRITE_BYTES"}` |
| Latency by filesystem | `µs` | `weka_fs_stats{stat="READ_LATENCY"}` ; `weka_fs_stats{stat="WRITE_LATENCY"}` |

Minimal dashboard skeleton (one panel shown; repeat for the other two):

```json
{
  "uid": "perorgfsio",
  "title": "Per-Org Filesystem IO",
  "tags": ["weka","multi-org","custom"],
  "schemaVersion": 39, "version": 1, "refresh": "10s",
  "time": {"from": "now-30m", "to": "now"},
  "templating": {"list": [
    {"name":"weka_datasource","type":"datasource","query":"prometheus","refresh":1,"current":{}}
  ]},
  "panels": [
    {"id":1,"type":"row","title":"Per-Org Filesystem I/O (weka_fs_stats)","gridPos":{"h":1,"w":24,"x":0,"y":0}},
    {"id":2,"type":"timeseries","title":"IOPS by filesystem (READS / WRITES)",
     "datasource":{"uid":"$weka_datasource"},
     "gridPos":{"h":8,"w":24,"x":0,"y":1},
     "fieldConfig":{"defaults":{"unit":"iops"},"overrides":[]},
     "targets":[
       {"refId":"A","datasource":{"uid":"$weka_datasource"},"expr":"weka_fs_stats{stat=\"READS\"}","legendFormat":"{{fs_name}} READS"},
       {"refId":"B","datasource":{"uid":"$weka_datasource"},"expr":"weka_fs_stats{stat=\"WRITES\"}","legendFormat":"{{fs_name}} WRITES"}
     ]}
  ]
}
```

Deploy to each Org Grafana (identical file):

```bash
scp perorg_fsio.json smith_oh@<C1>:/tmp/      # and <C2>
# on each node:
cp /tmp/perorg_fsio.json /var/lib/grafana/dashboards/Per_Org_Filesystem_IO.json
chown grafana:grafana /var/lib/grafana/dashboards/Per_Org_Filesystem_IO.json
systemctl restart grafana-server
```

Verify (per Org):

```bash
curl -s "http://admin:Passw0rd!@localhost:3000/api/dashboards/uid/perorgfsio" | grep -c '"Per-Org Filesystem IO"'
curl -s -G http://localhost:9090/api/v1/query --data-urlencode 'query=weka_fs_stats'
```

Validated on smith-25 under FIO load (org1 random read / org2 random write):

| | client-1 (org1) | client-2 (org2) |
|---|---|---|
| Dashboard `perorgfsio` | loaded | loaded |
| Filesystem shown | **org1fs1 only** | **org2fs1 only** |
| READS | ~55,000 iops | 0 |
| WRITES | 0 | ~40,000 iops |
| READ_LATENCY | ~1.8 ms | 0 |
| WRITE_LATENCY | 0 | ~2.5 ms |

→ proves the multi-Org headline value: each tenant's Grafana shows **only its own filesystem's per-fs I/O**, drawn from `weka_fs_stats`. Keep a workload running while building/observing — `weka_fs_stats` is empty without I/O.

#### 9.4 (GUI alternative) — build the dashboard in the UI

Because this is a **new** dashboard (not provisioned), the GUI **Save works cleanly** — unlike §9.2:

1. **Dashboards → New → New dashboard → + Add visualization**; pick the **Prometheus** data source.
2. Build three **Time series** panels using the queries from the table above (Code-mode query + `{{fs_name}} …` legend), and set each panel's **Unit**: `iops` / `bytes/sec(SI)` / `microseconds (µs)`.
3. **Save** → title `Per-Org Filesystem IO`. It is stored in Grafana's DB on that node.

Trade-off vs the file method: a GUI-built dashboard lives in **one node's DB only** and is not portable — you'd rebuild it on each Org's Grafana. The file method above (same JSON dropped on every node) is the right choice for multi-node, provisioning-consistent deployment. For per-Org isolation it makes no difference which method you use — each node still reads only its own Org's Prometheus.

> **Note (Grafana 13):** to later make this **provisioned** (in `/var/lib/grafana/dashboards/`), use the **file/JSON above**, not a GUI *Export* — GUI Export emits the new v2 schema, which the file provider does not load reliably (see §9.2).

> **Two ways to add visualizations** (both in this runbook): **§9.2** = append a panel to an existing dashboard JSON; **§9.4** = ship a whole new dashboard JSON. Both are just files under `/var/lib/grafana/dashboards/` picked up by the file provider.

## 10. Operations Notes

- **Bring-up order:** Loki (client-3/4) → exporters (client-0, push to those Loki) → Prometheus + Grafana (client-1/2).
- **Service mgmt (native):** `systemctl status|restart loki|prometheus|grafana-server`; logs via `journalctl -u <svc>`.
- **Exporter (Docker):** `docker compose -f /opt/weka-mon/docker-compose-export.yml {up -d|down}`.
- **Versions used:** Loki 2.8.10, Prometheus 2.50.1, Grafana 13.1.0 (yum latest), exporter image `20260216`.
- **Data dirs / perms:** `/loki` owned by `loki`; `/var/lib/prometheus` owned by `prometheus`; `/var/lib/grafana` owned by `grafana`.
- **Network:** Prometheus(C1/C2)→exporter(C0):8001/8002; exporter(C0)→Loki(C3/C4):3100 must be reachable.
- **Scaling:** add `export-orgN` (next port) on client-0 + a dedicated Prometheus/Grafana + Loki node per new Org.
