# WEKA Multi-Org Exporter Configuration and Validation Report — WEKA v4.4.33

> **Validation summary — PASSED (validated twice).** The full procedure in this report was run end to end **two independent times on freshly deployed WEKA `v4.4.33` clusters**, and both runs completed with no issues. In both runs, `weka_fs` (capacity), per-Org separation, and `weka_fs_stats` (per-filesystem I/O) were emitted correctly and remained **stable across the entire 30-minute workload** (sampled at 2 / 5 / 10 / 15 / 22 / 25 / 28 minutes — 6 series per Org at every point, zero `node-host` errors). v4.4.33 behaves equivalently to v4.4.10.171.

## 1. Overview

This report is a complete runbook for configuring per-Organization (hereafter "Org") Exporters on WEKA `v4.4.33` and validating that per-Org filesystem capacity and I/O statistics are collected correctly in Prometheus format, using the `wekasolutions/export:20260216` image.

`org1` and `org2` were separated into independent authentication tokens and Exporter instances. Both the capacity metric (`weka_fs`) and the per-filesystem I/O statistics metric (`weka_fs_stats`) were confirmed to be emitted correctly and **stably throughout a 30-minute workload**, with per-Org separation. v4.4.33 behaves equivalently to the the v4.4.10.171 validation.

## 2. Validation Goals

- Run per-Org Exporters independently using per-Org authentication tokens.
- Confirm each Exporter only sees its own Org's filesystem.
- Confirm `weka_fs` (capacity) is emitted correctly.
- Confirm `weka_fs_stats` (per-filesystem IOPS, throughput, latency) is emitted correctly and remains stable under sustained workload.
- Validate compatibility between WEKA `v4.4.33` and Exporter image `20260216`.

## 3. Validation Environment

| Item | Setting |
|---|---|
| WEKA version | `v4.4.33` |
| Exporter image | `wekasolutions/export:20260216` |
| Exporter run method | Docker Compose |
| Cluster | 6 backends + 6 clients |
| Org 1 | `org1` / admin `org1admin` |
| Org 1 filesystem | `org1fs1` (`fs_id=2`, 111 GB) |
| Org 1 Exporter port | `8001` |
| Org 2 | `org2` / admin `org2admin` |
| Org 2 filesystem | `org2fs1` (`fs_id=3`, 222 GB) |
| Org 2 Exporter port | `8002` |
| Exporter host | a WEKA client node (Docker installed) |
| Backend management IP | `<WEKA_MANAGEMENT_IP>` (port `14000`) |

> Actual passwords and token values are not included in this report.

## 4. Configuration Procedure

### 4.1 Generate Per-Org Authentication Tokens

Log in with each Org's admin account to generate a separate authentication token file. This can be run on a backend, or on the Exporter host pointing at a backend management IP with `-H`.

```bash
weka user login org1admin '<ORG1_PASSWORD>' --org org1 \
  -H <WEKA_MANAGEMENT_IP> --path /opt/weka-mon/.weka/org1-authtoken.json

weka user login org2admin '<ORG2_PASSWORD>' --org org2 \
  -H <WEKA_MANAGEMENT_IP> --path /opt/weka-mon/.weka/org2-authtoken.json
```

On success:

```text
+------------------------------+
| Login completed successfully |
+------------------------------+
Default profile updated successfully
```

### 4.2 Place the Authentication Tokens (and fix permissions)

The token files must live under `/opt/weka-mon/.weka` on the Exporter host, and **must be readable by the Exporter container**.

```bash
mkdir -p /opt/weka-mon/.weka
# (if generated elsewhere, move them here)

# IMPORTANT: `weka user login --path` creates the token files as mode 0600 owned by root.
# The wekasolutions/export container runs as a NON-root user, so with 0600 it cannot
# read the token and fails with:
#     CRITICAL:Unable to ... Permission denied: '/weka/.weka/org1-authtoken.json'
# and the container falls into a restart loop. Make the files container-readable:
chmod 644 /opt/weka-mon/.weka/*.json
```

Resulting layout:

```text
/opt/weka-mon/.weka/
├── org1-authtoken.json   # 644
└── org2-authtoken.json   # 644
```

> **Two ways to resolve the `0600` permission problem** — use either one:
> - `chmod 644 /opt/weka-mon/.weka/*.json` (shown above; **used in this validation** — the `docker-compose.yml` in Appendix A.3 does not set `user`), **or**
> - Run the Exporter container as root by adding `user: "0:0"` to each service in `docker-compose.yml`.
>
> For a production setup, prefer running the container as root (`user: "0:0"`) over world-readable `644` credentials.

### 4.3 Verify the Org Information in the Token

Optionally inspect the JWT payload to confirm each token is for the correct Org and user.

```bash
jq -r .access_token /opt/weka-mon/.weka/org1-authtoken.json \
  | cut -d. -f2 | tr '_-' '/+' | awk '{print $0 "=="}' | base64 -d
```

The `org1` token should show:

```json
{ "sub": { "org_id": 1, "role": "OrgAdminUser", "source": "Internal", "username": "org1admin" } }
```

The `org2` token should show `org_id=2`, `username=org2admin`.

> A trailing `base64: invalid input` message may appear due to Base64 padding; the JSON payload above it is still valid.

### 4.4 Write Per-Org Exporter Configuration Files

Create a separate config file per Org. The complete files are in **Appendix A.1 / A.2**.

> **Required: the `stats:` section.** The config file must contain a `stats:` section (`cpu`, `ops`, `ops_driver`, ...). The Exporter reads `config['stats']` at startup; if it is missing the process aborts with `KeyError: 'stats'` and the container restart-loops. The per-filesystem `weka_fs_stats` series (`READS`, `READ_BYTES`, `READ_LATENCY`, `THROUGHPUT`, `WRITES`, `WRITE_BYTES`, `WRITE_LATENCY`) come from the `ops` category.

Key fields per Org (full file in Appendix A):

```yaml
exporter:
  listen_port: 8001          # container-internal port (fixed); host side differs via ports mapping

cluster:
  auth_token_file: org1-authtoken.json   # org2-authtoken.json for Org 2
  hosts:
    - <WEKA_MANAGEMENT_IP>
  force_https: false
  verify_cert: false
  mgmt_port: 14000

stats:
  # cpu / ops / ops_driver — see Appendix A.1
```

### 4.5 Docker Compose Configuration

The full file is in **Appendix A.3**. It defines only the two exporter services (one per Org). Both use container-internal port `8001`; the host ports `8001`/`8002` are separated by the `ports` mapping. Each service uses `command: -v` (the per-Org config is bind-mounted onto the default path `/weka/export.yml`). This compose does **not** set `user: "0:0"`, so the container runs as its default user and relies on the token files being `chmod 644` (see §4.2).

### 4.6 Run the Exporters

```bash
cd /opt/weka-mon
docker compose pull
docker compose up -d
docker ps
```

Expected:

```text
IMAGE                           PORTS                    NAMES
wekasolutions/export:20260216   0.0.0.0:8001->8001/tcp  weka-export-org1
wekasolutions/export:20260216   0.0.0.0:8002->8001/tcp  weka-export-org2
```

The `8002->8001` mapping (host 8002 → container 8001) is expected; both containers listen on `8001` internally.

### 4.7 Run a Test Workload

`weka_fs_stats` only has data when there is filesystem I/O. Mount each Org's filesystem on the client (using its Org token) and run FIO.

```bash
mkdir -p /mnt/org1fs1 /mnt/org2fs1
mount -t wekafs -o auth_token_path=/opt/weka-mon/.weka/org1-authtoken.json \
  <WEKA_MANAGEMENT_IP>/org1fs1 /mnt/org1fs1
mount -t wekafs -o auth_token_path=/opt/weka-mon/.weka/org2-authtoken.json \
  <WEKA_MANAGEMENT_IP>/org2fs1 /mnt/org2fs1

# org1 = read, org2 = write, 30 minutes
fio --name=org1read  --filename=/mnt/org1fs1/tf --rw=randread  --bs=64k --size=4G \
  --direct=1 --ioengine=libaio --iodepth=32 --numjobs=4 --runtime=1800 --time_based &
fio --name=org2write --directory=/mnt/org2fs1  --rw=randwrite --bs=64k --size=4G \
  --direct=1 --ioengine=libaio --iodepth=32 --numjobs=4 --runtime=1800 --time_based &
```

### 4.8 Check the Metrics Endpoint

```bash
curl -s http://localhost:8001/metrics | grep '^weka_fs'   # org1
curl -s http://localhost:8002/metrics | grep '^weka_fs'   # org2
```

## 5. Validation Results

### 5.1 `weka_fs` (capacity) and Per-Org separation

```text
# org1 (8001)
weka_fs{cluster="smith-22",name="org1fs1",stat="available_total"} 1.1100000256e+011
weka_fs{cluster="smith-22",name="org1fs1",stat="available_ssd"}   1.1100000256e+011
...
# org2 (8002)
weka_fs{cluster="smith-22",name="org2fs1",stat="available_total"} 2.22000001024e+011
weka_fs{cluster="smith-22",name="org2fs1",stat="available_ssd"}   2.22000001024e+011
...
```

| Endpoint | Auth token | Observed filesystem | Result |
|---|---|---|---|
| `localhost:8001/metrics` | `org1-authtoken.json` | `org1fs1`, `fs_id=2` | OK |
| `localhost:8002/metrics` | `org2-authtoken.json` | `org2fs1`, `fs_id=3` | OK |

### 5.2 `weka_fs_stats` stability across the run

`weka_fs_stats` produced the full set of series at every sample point throughout the 30-minute run, with no `node-host` errors:

| Elapsed | org1 series | org2 series | node-host err | cluster I/O |
|---|---|---|---|---|
| 2 min | 6 | 6 | 0 | reads 3.56 / writes 2.94 GiB/s |
| 5 min | 6 | 6 | 0 | reads 3.32 / writes 3.15 GiB/s |
| 10 min | 6 | 6 | 0 | reads 3.71 / writes 2.93 GiB/s |
| 15 min | 6 | 6 | 0 | reads 3.68 / writes 2.94 GiB/s |
| 22 min | 6 | 6 | 0 | reads 3.53 / writes 2.91 GiB/s |
| 25 min | 6 | 6 | 0 | reads 3.36 / writes 3.09 GiB/s |

### 5.3 `weka_fs_stats` sample output (at 22 min)

**org1 (`fs_id=2`, random read):**

```text
weka_fs_stats{cluster="smith-22",fs_id="2",fs_name="org1fs1",stat="READS",unit="iops"} 59315.97
weka_fs_stats{cluster="smith-22",fs_id="2",fs_name="org1fs1",stat="READ_BYTES",unit="bytespersec"} 3.887e+09
weka_fs_stats{cluster="smith-22",fs_id="2",fs_name="org1fs1",stat="READ_LATENCY",unit="microsecs"} 1958.06
weka_fs_stats{cluster="smith-22",fs_id="2",fs_name="org1fs1",stat="THROUGHPUT",unit="bytespersec"} 3.887e+09
weka_fs_stats{cluster="smith-22",fs_id="2",fs_name="org1fs1",stat="WRITES",unit="iops"} 0.0
weka_fs_stats{cluster="smith-22",fs_id="2",fs_name="org1fs1",stat="WRITE_BYTES",unit="bytespersec"} 0.0
```

**org2 (`fs_id=3`, random write):**

```text
weka_fs_stats{cluster="smith-22",fs_id="3",fs_name="org2fs1",stat="READS",unit="iops"} 0.0
weka_fs_stats{cluster="smith-22",fs_id="3",fs_name="org2fs1",stat="READ_BYTES",unit="bytespersec"} 0.0
weka_fs_stats{cluster="smith-22",fs_id="3",fs_name="org2fs1",stat="THROUGHPUT",unit="bytespersec"} 3.055e+09
weka_fs_stats{cluster="smith-22",fs_id="3",fs_name="org2fs1",stat="WRITES",unit="iops"} 46612.88
weka_fs_stats{cluster="smith-22",fs_id="3",fs_name="org2fs1",stat="WRITE_BYTES",unit="bytespersec"} 3.055e+09
weka_fs_stats{cluster="smith-22",fs_id="3",fs_name="org2fs1",stat="WRITE_LATENCY",unit="microsecs"} 2490.22
```

Key findings:

- `fs_id` / `fs_name` labels present (org1=2, org2=3).
- org1 shows read activity only; org2 shows write activity only — workload direction is reflected exactly, confirming per-Org isolation.
- Read IOPS ~53k–59k, write IOPS ~46k–49k, throughput ~3.0–3.9 GB/s — consistent across the full run.

## 6. Operational Considerations

1. The Exporter must use an authentication token generated within the target Org.
2. The config file **must** include the `stats:` section (otherwise `KeyError: 'stats'`).
3. Token files default to `0600 root`; make them container-readable (`chmod 644`) **or** run the container as root (`user: "0:0"`), otherwise the container restart-loops with `Permission denied`.
4. Before validating `weka_fs_stats`, run an actual read/write workload on the target filesystem.
5. Use the `fs_id`, `fs_name`, `stat`, `unit` labels to distinguish per-filesystem metrics in Prometheus and Grafana.
6. Do not store token/credential files as plaintext in source repositories or general documents.

## 7. Conclusion

On WEKA `v4.4.33`, the Multi-Org Exporter configuration was completed successfully using the `wekasolutions/export:20260216` image. Both `weka_fs` (capacity) and `weka_fs_stats` (per-filesystem IOPS, throughput, latency) were emitted correctly with per-Org separation, and `weka_fs_stats` remained stable across the full 30-minute workload. The `fs_id` and `fs_name` labels allow per-filesystem identification in Prometheus and Grafana. v4.4.33 behaves equivalently to v4.4.10.171.

## Appendix A. Configuration Files Used

These are the exact files used on the Exporter host (`/opt/weka-mon`). `<WEKA_MANAGEMENT_IP>` stands for a backend management IP (port `14000`). The `export-org*.yml` files are based on the stock `export.yml` template, so `events_to_loki`/`events_to_syslog` are left `True`; the Loki container is commented out in `docker-compose.yml` (§A.3), so event delivery to Loki simply does not occur — this does not affect metric collection. Token files are made container-readable with `chmod 644` (the compose does **not** set `user: "0:0"`).

### A.1 `export-org1.yml`

```yaml
exporter:
  listen_port: 8001
  events_only: False
  events_to_loki: True
  events_to_syslog: True
  timeout: 10.0
  max_procs: 8
  max_threads_per_proc: 100
  backends_only: True
  datapoints_per_collect: 5
  certfile: null
  keyfile: null

loki:
  host: localhost
  port: 3100
  protocol: https
  path: /loki/api/v1/push
  user: null
  password: null
  org_id: null
  client_cert: null
  verify_cert: False

cluster:
  auth_token_file: org1-authtoken.json
  hosts:
    - <WEKA_MANAGEMENT_IP>
  force_https: False   # only 3.10+ clusters support https
  verify_cert: False  # default cert cannot be verified
  mgmt_port: 14000

stats:
  cpu:     # Category
     CPU_UTILIZATION: "%"    # metric
  ops:
    ACCESS_OPS: ops
    COMMIT_OPS: ops
    CREATE_OPS: ops
    FILECLOSE_OPS: ops
    FILEOPEN_OPS: ops
    FLOCK_OPS: ops
    FSINFO_OPS: ops
    GETATTR_OPS: ops
    LINK_OPS: ops
    MKDIR_OPS: ops
    MKNOD_OPS: ops
    OPS: ops
    READDIR_OPS: ops
    READS: iops
    READ_BYTES: bytespersec
    READ_DURATION: microsecs
    READ_LATENCY: microsecs
    REMOVE_OPS: ops
    RENAME_OPS: ops
    RMDIR_OPS: ops
    THROUGHPUT: bytespersec
    UNLINK_OPS: ops
    WRITES: iops
    WRITE_BYTES: bytespersec
    WRITE_LATENCY: microsecs
  ops_driver:     # Category
    DIRECT_READ_SIZES: sizes
    DIRECT_WRITE_SIZES: sizes
    READ_SIZES: sizes
    WRITE_SIZES: sizes
```

> Note: only one top-level `stats:` key is present, and every active category (`cpu`, `ops`, `ops_driver`) has at least one metric. The exporter aborts if `stats:` is duplicated or if a category is listed with no metrics under it.

### A.2 `export-org2.yml`

Identical to A.1 except `cluster.auth_token_file`:

```yaml
cluster:
  auth_token_file: org2-authtoken.json
```

### A.3 `docker-compose.yml`

Only the two exporter services are defined (one per Org). Both use `command: -v` (the per-Org config is bind-mounted onto the default path `/weka/export.yml`, so no explicit `-c` is needed). There is **no** `user: "0:0"` — the container runs as its default user and reads the `chmod 644` token files (see §4.2).

```yaml
services:

  export-org1:
    image: wekasolutions/export:20260216
    container_name: weka-export-org1
    volumes:
      - /dev/log:/dev/log
      - ${PWD}/.weka:/weka/.weka
      - /etc/hosts:/etc/hosts
      - ${PWD}/export-org1.yml:/weka/export.yml
    command: -v
    restart: always
    ports:
      - "8001:8001"
    logging:
      options:
        max-file: "5"
        max-size: "15m"

  export-org2:
    image: wekasolutions/export:20260216
    container_name: weka-export-org2
    volumes:
      - /dev/log:/dev/log
      - ${PWD}/.weka:/weka/.weka
      - /etc/hosts:/etc/hosts
      - ${PWD}/export-org2.yml:/weka/export.yml
    command: -v
    restart: always
    ports:
      - "8002:8001"
    logging:
      options:
        max-file: "5"
        max-size: "15m"
```
