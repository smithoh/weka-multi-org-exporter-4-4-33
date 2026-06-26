# WEKA Multi-Org Exporter — Overview & Configuration Guide

> **Status: DRAFT.** Fact-based draft for review before wider distribution (team / tools repository).
> Step-by-step deployment is in the [deployment runbook](multi-org-monitoring-deployment.md).

## TL;DR

The WEKA Prometheus exporter container (`wekasolutions/export`) reports **cluster-wide** metrics by default. In a **multi-Organization (multi-tenant)** cluster, a single exporter cannot give each tenant a view limited to *their own* Organization's filesystems.

The **Multi-Org Exporter** pattern solves this by running **one exporter container per Organization**, each authenticated with that Org's own token, on a different port. Each instance then exposes per-filesystem capacity and I/O metrics (with `fs_id` / `fs_name` labels) scoped to a single Org, ready for per-tenant Prometheus scraping and Grafana dashboards.

---

## 1. The Problem

WEKA supports **Organizations** for multi-tenancy: tenants are isolated and an Org admin can only see their own Org's resources.

The exporter, however, is designed for a **single, cluster-wide** view:

- It logs into the cluster with one identity and reports metrics for the whole cluster.
- Giving each tenant a Grafana view restricted to *their* filesystems is not achievable from a single shared exporter instance.

Teams and customers running multi-Org clusters (e.g. Coupang, and others raising the same need) want **per-Org dashboards** — each tenant sees only their filesystems' capacity, IOPS, throughput, and latency.

## 2. The Solution — one exporter per Org

Only the **deployment** changes: run the exporter container **once per Organization**, each with:

- a dedicated **Org auth token** (the exporter logs into the cluster as that Org's admin → it only sees that Org's filesystems), and
- a dedicated **host port** (e.g. 8001 for org1, 8002 for org2).

```
                         WEKA cluster (mgmt API :14000)
                          ├─ Org1 (org1fs1 ...)
                          └─ Org2 (org2fs1 ...)
                                   ▲           ▲
              org1 token ──────────┘           └────────── org2 token
                                   │           │
  ┌───────────────────────────────┴───┐  ┌────┴──────────────────────────────┐
  │ export-org1  (container)           │  │ export-org2  (container)          │
  │   /weka/export.yml = export-org1.yml│  │   /weka/export.yml = export-org2.yml│
  │   listens :8001                     │  │   listens :8001                    │
  └───────────────┬─────────────────────┘  └──────────────┬────────────────────┘
       host :8001 │                                host :8002
                  ▼                                       ▼
          /metrics (org1 only)                    /metrics (org2 only)
                  └───────────────┬───────────────────────┘
                                  ▼
                           Prometheus (scrape job per Org)
                                  ▼
                              Grafana (per-Org dashboards)
```

Both containers listen on `8001` internally; the host ports `8001`/`8002` separate them via Docker port mapping.

## 3. Prerequisites & Compatibility

| Item | Notes |
|---|---|
| Exporter image | `wekasolutions/export` — see version note below |
| Host | Any host with Docker + network reach to a WEKA backend management IP (`:14000`); typically a WEKA client node |
| WEKA Organizations | `org1`, `org2`, … created, each with an admin user and at least one filesystem |
| Per-Org auth tokens | One token file per Org (§5.1) |

**Exporter image / WEKA version compatibility** (validated by us):

| WEKA version | Image | per-fs `weka_fs_stats` | Status |
|---|---|---|---|
| 4.4.10.171 | `wekasolutions/export:20260216` | OK | ✅ validated |
| 4.4.33 | `wekasolutions/export:20260216` | OK | ✅ validated (3 independent deployments) |
| 5.1.0 | (exporter originally written against 5.1.0 API) | OK | per upstream |

> The per-filesystem stats collection differs between the WEKA 4.4.x line and 5.1.0 APIs. The `20260216` image carries the 4.4.x compatibility handling and is the recommended image for 4.4.x clusters. Always re-validate the exact image against the exact WEKA build you deploy.

## 4. Metrics Exposed

Each per-Org exporter exposes, on its `/metrics` endpoint:

| Metric | Type | Key labels | Meaning |
|---|---|---|---|
| `weka_fs` | gauge | `cluster`, `name`, `stat` | Per-filesystem **capacity** (`available_total`, `used_total`, `available_ssd`, `used_ssd`, `total_percent_used`, `ssd_percent_used`) |
| `weka_fs_stats` | gauge | `cluster`, `fs_id`, `fs_name`, `stat`, `unit` | Per-filesystem **I/O** (`READS`, `READ_BYTES`, `READ_LATENCY`, `THROUGHPUT`, `WRITES`, `WRITE_BYTES`, `WRITE_LATENCY`) |
| `weka_stats` | gauge | `category`, `stat`, `unit`, host labels | General WEKA statistics selected via the config `stats:` section |

`fs_id` / `fs_name` on `weka_fs_stats` are what make per-filesystem (and therefore per-Org) breakdown possible.

### 4.1 How the `stats:` section relates to `weka stats`

The `stats:` section of `export.yml` is a **whitelist that selects which WEKA statistics the exporter publishes as `weka_stats`** (the per-node metric). Each entry maps directly to a `weka stats` category/statistic — the stock template states this explicitly: *"if you are familiar with 'weka stats', these are `--category <category> --stat <statistic>`."*

| `export.yml` (`stats:`) | Equivalent `weka stats` query | Published as |
|---|---|---|
| `ops: READS` | `weka stats --category ops --stat READS` | `weka_stats{category="ops",stat="READS",unit="iops",...}` |
| `ops: READ_LATENCY` | `weka stats --category ops --stat READ_LATENCY` | `weka_stats{category="ops",stat="READ_LATENCY",unit="microsecs",...}` |
| `cpu: CPU_UTILIZATION` | `weka stats --category cpu --stat CPU_UTILIZATION` | `weka_stats{category="cpu",...}` |
| `ssd: SSD_READ_REQS` | `weka stats --category ssd --stat SSD_READ_REQS` | `weka_stats{category="ssd",...}` |

The full catalog of categories/statistics is in WEKA's [list of statistics](https://docs.weka.io/usage/statistics/list-of-statistics). The exporter only emits the entries you uncomment in `stats:`.

> **Important — `weka_fs_stats` is separate from `stats:`.** The per-filesystem metric `weka_fs_stats` (labels `fs_id`/`fs_name`) is **not** controlled by the `stats:` section; the exporter always collects it via the internal `fs_stats` category. So enabling/disabling entries in `stats:` changes only `weka_stats`, never `weka_fs_stats`.

> **Verified (smith-25, live):** the exporter mirrors `weka stats` **1:1, no extra aggregation**. At the same instant, backend `weka stats --category fs_stats --stat READS` returned **`59611.98 ops/s`** and the exporter's `weka_fs_stats{fs_name="org1fs1",stat="READS"}` read **`59611.98 iops`** — identical value, same unit (`ops/s` = `iops`). The exporter republishes WEKA's computed value as-is.

## 5. Configuration

> This section is the generalized procedure. The complete exporter config files and Docker Compose are in the [deployment runbook](multi-org-monitoring-deployment.md) (§4).

### 5.1 Generate a per-Org auth token

```bash
weka user login <orgNadmin> '<PASSWORD>' --org <orgN> \
  -H <WEKA_MANAGEMENT_IP> --path /opt/weka-mon/.weka/<orgN>-authtoken.json
```

The exporter logs into the cluster using this token, so it only sees that Org's filesystems.

### 5.2 Place the token and fix permissions

```bash
chmod 644 /opt/weka-mon/.weka/*.json
```

> `weka user login --path` writes the token as `0600 root`. The exporter container runs as a non-root user, so with `0600` it cannot read the token and restart-loops with `Permission denied`. Fix with `chmod 644` **or** run the container as root (`user: "0:0"` in `docker-compose.yml`). Prefer `user: "0:0"` in production over world-readable credentials.

### 5.3 Per-Org config file (`export-orgN.yml`)

Key fields (full file in Appendix A):

```yaml
exporter:
  listen_port: 8001          # container-internal port (same for all Orgs)
cluster:
  auth_token_file: <orgN>-authtoken.json
  hosts: [ <WEKA_MANAGEMENT_IP> ]
  mgmt_port: 14000
stats:
  # cpu / ops / ops_driver — selects which weka stats to publish (see §4.1)
```

> **The `stats:` section is required.** If it is missing, the exporter aborts with `KeyError: 'stats'` and restart-loops. Every listed category must have at least one metric under it.

### 5.4 Docker Compose (one service per Org)

Each Org is a service mapping a unique host port to container `8001`:

```yaml
services:
  export-org1:
    image: wekasolutions/export:20260216
    volumes:
      - ${PWD}/.weka:/weka/.weka
      - ${PWD}/export-org1.yml:/weka/export.yml
    command: -v
    ports: ["8001:8001"]
  export-org2:
    image: wekasolutions/export:20260216
    volumes:
      - ${PWD}/.weka:/weka/.weka
      - ${PWD}/export-org2.yml:/weka/export.yml
    command: -v
    ports: ["8002:8001"]
```

Adding an Org = add another `export-orgN` service with its token, config file, and the next host port.

### 5.5 Run & verify

```bash
docker compose up -d
curl -s http://localhost:8001/metrics | grep '^weka_fs'   # org1
curl -s http://localhost:8002/metrics | grep '^weka_fs'   # org2
```

`weka_fs_stats` only carries data while the filesystem has I/O — mount the Org's filesystem and run a workload to see it populate.

## 6. External (Native) Prometheus, Grafana & Loki Integration (implemented)

A full **per-Org-isolated** monitoring stack has been built and validated (WEKA v4.4.33). Here **"external" means natively installed on the OS** — Grafana from the official yum repo, Prometheus and Loki from official release binaries with `systemd` — **not** the bundled `weka-mon` Docker stack. Only the WEKA exporter stays a container. Each Org gets its own Prometheus + Grafana (metrics) and its own Loki (events). Full runbook:

➡️ **[multi-org-monitoring-deployment.md](multi-org-monitoring-deployment.md)**

Layout (validated on `smith-25`):
- **client-0** — per-Org exporters, Docker (`:8001` org1, `:8002` org2)
- **client-1 / client-2** — native Prometheus (binary) + Grafana (yum), one per Org
- **client-3 / client-4** — native Loki (binary), one per Org (exporter pushes events here)

Two data flows, isolated per tenant:
- **Metrics**: exporter → Prometheus (per Org) → Grafana → `Weka Cluster Overview` / `Weka Filesystems Detail` etc.
- **Events**: exporter → Loki (per Org) → Grafana → `Weka Logs`

The runbook above has the exact install commands and the edited parameters per component.

## 7. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Container restart-loops; log `KeyError: 'stats'` | `export.yml` has no `stats:` section (or a duplicate / empty category) | Add a single valid `stats:` section; every category needs ≥1 metric |
| Container restart-loops; log `Permission denied: .../*-authtoken.json` | Token file is `0600 root`, unreadable by the non-root container | `chmod 644` the token, or set `user: "0:0"` |
| `weka_fs` present but `weka_fs_stats` empty | No filesystem I/O at scrape time | Run a read/write workload; per-fs stats only appear with activity |
| `weka_fs_stats` empty + repeating `Exception looking up node-host map: 'node-host'` | Exporter/WEKA version mismatch in per-fs stats collection | Use the image validated for your WEKA build (`20260216` for 4.4.x); re-validate per exact build |
| `docker compose up` mounts a directory at `/weka/export.yml` | The `export-orgN.yml` bind-mount source file does not exist | Create the config file before `up` |

## 8. Security & Operations

- **Isolation** comes from the token: an Org-admin token only exposes that Org's filesystems. Use the correct per-Org token for each exporter.
- **Credentials**: protect token files; do not commit them. Prefer `user: "0:0"` over world-readable `644` in production.
- **Scaling**: one service + one host port per Org; keep a consistent naming convention (`export-orgN`, `orgN-authtoken.json`, port `800N`).

## 9. Validation Status

- WEKA `v4.4.10.171` — validated (stable across a 30-minute workload).
- WEKA `v4.4.33` — validated on three independent fresh deployments (stable; per-Org separation and workload direction correct).

## 10. References

- WEKA external monitoring documentation: https://docs.weka.io/monitor-the-weka-cluster/external-monitoring
- WEKA list of statistics: https://docs.weka.io/usage/statistics/list-of-statistics
- Exporter image: `wekasolutions/export`

## Done / Validated

- [x] Live 1:1 `weka stats` ↔ `weka_fs_stats` value/unit comparison — exact match on smith-25 (§4.1).
- [x] External (native) Prometheus + Grafana + Loki deployment — full runbook in [multi-org-monitoring-deployment.md](multi-org-monitoring-deployment.md) (§3–7).
- [x] Per-Org Grafana dashboard for `weka_fs_stats` — `Per-Org Filesystem IO` (deployment runbook §8.4); plus a custom-panel example (§8.2).
