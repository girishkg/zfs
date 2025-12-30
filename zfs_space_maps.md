
## ZFS “Space Maps” – a practical guide  

When people refer to **“space maps”** in the ZFS world they’re almost always talking about the *space‑usage view* that ZFS gives you for a pool or a dataset.  It’s the high‑level summary that lets you see:

| What you see | Why it matters |
|--------------|----------------|
| **used** – the *physical* space actually written to the pool | How much of the raw pool you’re consuming |
| **referenced** – the space you could reclaim by destroying a snapshot | The “true” footprint of a dataset |
| **logical used** – the sum of all *referenced* data in a dataset (including descendants) | A more realistic number of bytes “owned” by the dataset |
| **avail** – how much *free* space the pool has | What’s left for new data |
| **quota / reservation** – limits you can impose | Governance / isolation of space |

These numbers come from ZFS’s **space map** mechanism – the internal bitmap that tracks free/used blocks.  The command‑line tools that surface this information are:

| Tool | What it shows | Typical syntax |
|------|---------------|----------------|
| `zfs space` | A concise per‑pool or per‑dataset summary | `zfs space <pool|dataset>` |
| `zfs list` | Hierarchical list of all datasets | `zfs list -r -o name,used,referenced,avail` |
| `zfs get` | Individual properties (used, referenced, quota, etc.) | `zfs get used,referenced,quota -r <pool>` |
| `zpool iostat` | Real‑time I/O & space usage | `zpool iostat -v` |

Below is a step‑by‑step tour of how to use these commands, interpret the output, and turn the numbers into a “map” that’s useful for day‑to‑day administration, monitoring, or capacity planning.

---

## 1. `zfs space` – The Quick‑Look

**Syntax**

```bash
# Show a single dataset (or entire pool if you give the pool name)
zfs space <pool_or_dataset>
```

**Typical output**

```
# zfs space tank
tank              20.00GB  4.50GB  0.00B   50.00GB
tank/var          15.00GB  4.00GB  0.00B   35.00GB
tank/home         5.00GB   0.50GB  0.00B   15.00GB
```

| Column | Meaning |
|--------|---------|
| `NAME` | Dataset name |
| `USED` | Physical blocks on the pool (bytes) |
| `REFERENCE` | Space that can be reclaimed by snapshot rollback (bytes) |
| `RESERVED` | Reserved space that cannot be used by other datasets (bytes) |
| `AVAIL` | Remaining free space in the pool (bytes) |

### What to look for

- **Large `USED` vs. `REFERENCE` gap**: This means a lot of your data is *snapshotted* and still occupies physical space.  If you’re okay with discarding old snapshots, you can free a lot of space by `zfs destroy -r tank@snap1`.
- **High `RESERVED`**: If you see 100 GB reserved for a dataset that only uses 5 GB, you probably set an unnecessary reservation.
- **Low `AVAIL`**: Once `AVAIL` dips below your “low‑watermark”, consider expanding the pool or cleaning up.

---

## 2. `zfs list` – A Hierarchical View

The `list` command is the workhorse.  Combine it with `-r` (recursive) and `-o` (output columns) to slice the data.

**Full dataset tree with space details**

```bash
zfs list -r -o name,used,referenced,avail,quota,reservation
```

Sample output:

```
NAME                  USED      REFERENCED      AVAIL    QUOTA  RESERVATION
tank                  20.00GB   15.00GB         50.00GB  none   none
tank/var              15.00GB   10.00GB         35.00GB  25GB   5GB
tank/var/log          5.00GB    3.00GB          20.00GB  10GB   2GB
tank/var/tmp          10.00GB   7.00GB          15.00GB  none   none
tank/home              5.00GB    2.00GB          15.00GB  5GB    0GB
```

### Handy flags

| Flag | Effect |
|------|--------|
| `-s` | Sort by the specified column (`-s used`, `-s referenced`) |
| `-t` | Show dataset type (filesystem, snapshot, volume) |
| `-p` | Print values in **bytes** instead of human‑readable units |
| `-H` | Hide headers (good for scripts) |

**Example: Show only datasets over 1 GB**

```bash
zfs list -H -o name,used -s used | awk '$2>1024*1024*1024 {print}'
```

---

## 3. `zfs get` – Inspect Individual Properties

While `list` shows aggregated numbers, `get` lets you drill into a single property:

```bash
zfs get -r used,referenced,quota,reservation -H tank
```

Typical output:

```
tank            used           20.00GB
tank            referenced     15.00GB
tank            quota          none
tank            reservation    none
```

Use this when you need the exact numeric value of a property for automation or reporting.

---

## 4. Understanding the Numbers

| Property | How ZFS Calculates It | What It Means |
|----------|-----------------------|---------------|
| **used** | Sum of all blocks that *contain data* (including snapshots) | The raw footprint on disk |
| **referenced** | Sum of *unique* blocks in a dataset (i.e., blocks that would be lost if the dataset were destroyed) | The “true” amount of data you own |
| **logical used** (seen in `zfs list`) | `referenced` + `referenced` of descendants | Total space “owned” by a dataset, ignoring snapshots |
| **quota** | Upper limit you can set | Prevents runaway growth |
| **reservation** | Hard limit you set | Guarantees space for a dataset, even if the pool is full |
| **avail** | Pool free space minus reservations | Space left for any dataset that doesn’t have a reservation |

**Key insight**: In a snapshot‑heavy environment, `used` can be several times larger than `referenced`.  Cleaning snapshots is often the quickest way to free physical space.

---

## 5. Practical “Space‑Map” Workflows

Below are common day‑to‑day scenarios that leverage the space‑map view.

### A. Spotting Snapshot “Bloat”

```bash
# Find datasets where snapshots are holding > 10 GB of space
zfs list -H -o name,used,referenced | \
  awk '$2>10*1024*1024*1024 {print $1, $2, $3}'
```

If you see a `tank/var@snap3` that is 1 TB in `USED` but only 100 MB in `REFERENCE`, you can reclaim a huge chunk:

```bash
zfs destroy -r tank@old_snapshot
```

### B. Creating a Capacity‑Planning Report

```bash
#!/usr/bin/env bash
# capacity_report.sh

POOL="tank"

# 1. Show total usage by each top‑level filesystem
zfs list -H -o name,used -s used | \
  awk -F'\t' '$1 ~ /^[^\/]+$/ {print $1, $2}'

# 2. Show snapshot usage per dataset
zfs list -r -t snapshot -o name,used | \
  awk -F'\t' '$2>0 {print $1, $2}'
```

Run this nightly via `cron` or integrate into your monitoring stack (e.g., Grafana with the *zfs* plugin).

### C. Using `zpool iostat` for Real‑Time Space Pressure

```bash
zpool iostat -v
```

You’ll see a line like:

```
tank  2.00K  0.00K  1.00B  30.00GB
```

Where `30.00GB` is the current free space (`AVAIL`).  Coupled with `zpool list`, you can watch the pool shrink in real time.

---

## 5. Script‑Friendly Cheat Sheet

```bash
# 1. Get the current pool space map (bytes)
zfs space -p tank > /tmp/space_map.txt

# 2. Get logical used for each dataset (bytes)
zfs list -r -H -o name,logicalused | awk '$2>0 {print}' > /tmp/usage.log

# 3. Find datasets exceeding their quota
zfs list -H -o name,used,quota | awk '$3=="none" {next} $2>$3 {print $1}'
```

Add these snippets to your alerting pipeline (e.g., Prometheus node‑exporter), or export to CSV for Excel capacity analysis.

---

## 6. Beyond the CLI – GUI & Monitoring

If you prefer a visual “map”:

| Tool | What it offers |
|------|----------------|
| **TrueNAS/FreeNAS** | Web UI → *Storage → Pools → Space Map* (graphical bar charts) |
| **ZFS‑Stat** | A community tool that draws tree‑like diagrams of space usage.  Install via `pkg install zfs-stat` (BSD) or compile from source (Linux). |
| **Grafana + Prometheus** | Export ZFS metrics via `zfs_exporter`; build dashboards that show used/referenced/avail over time. |
| **Alertmanager** | Trigger alerts when `avail` falls below a threshold (e.g., `node_zfs_pool_avail < 5GB`). |

---

## TL;DR Cheat Sheet

| Command | Best for | Key columns |
|---------|----------|-------------|
| `zfs space tank` | One‑liner pool/dataset summary | `USED`, `REFERENCE`, `RESERVED`, `AVAIL` |
| `zfs list -r -o name,used,referenced,avail,quota,reservation` | Full tree view | `USED`, `REFERENCED`, `AVAIL`, `QUOTA`, `RESERVATION` |
| `zfs get -r used,referenced,quota,reservation tank` | Drill‑in property | Exact numeric values |
| `zpool iostat -v` | Real‑time I/O & free space | Live `AVAIL` & I/O throughput |

**Rule of thumb**:  
- **If `USED` >> `REFERENCE`** → *Snapshot cleanup*  
- **If `QUOTA` is set too low** → *Consider increasing or removing the limit*  
- **If `RESERVATION` is high but dataset usage is low** → *Remove or lower the reservation*  

Use the above commands routinely, and you’ll have a clear “map” of where space lives in your ZFS pool—ready for quick action or detailed analysis. Happy mapping!
