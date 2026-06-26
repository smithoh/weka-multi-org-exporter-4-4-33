# WEKA Multi-Org Exporter — Overview

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
| Per-Org auth tokens | One token file per Org (see [deployment runbook](multi-org-monitoring-deployment.md)) |

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

## 5. Deployment

This document is an **overview only**. Every step-by-step procedure lives in the deployment runbook:

➡️ **[multi-org-monitoring-deployment.md](multi-org-monitoring-deployment.md)**

The runbook covers, end to end:
- per-Org auth token generation, exporter config (`export-orgN.yml`), and Docker Compose;
- native (OS-level) install of Prometheus, Grafana, and Loki — one isolated stack per Org — with the exact edited parameters for each component;
- custom dashboards (e.g. `Per-Org Filesystem IO` on `weka_fs_stats`);
- verification, troubleshooting, and day-2 operations (security, scaling).

## 6. References

- WEKA external monitoring documentation: https://docs.weka.io/monitor-the-weka-cluster/external-monitoring
- WEKA list of statistics: https://docs.weka.io/usage/statistics/list-of-statistics
- Exporter image: `wekasolutions/export`
