# 📦 Prometheus TSDB: Blocks, Chunks & Write Flow (Local vs. Thanos)

---

## 1. Blocks & Chunks — The Core Concepts

### 🧱 Blocks
A **block** is a directory holding all time series data for a specific time window.

- Samples are first grouped into **2-hour blocks**.
- Background **compaction** later merges small blocks into larger ones — up to **10% of retention time or 31 days**, whichever is smaller.
- The **current (head) block** lives in memory, protected by a **Write-Ahead Log (WAL)** stored in `wal/` as **128 MB segments**.

**Each block directory contains:**

| File/Folder | Purpose |
|---|---|
| `chunks/` | The actual compressed sample data |
| `index` | Maps metric names & labels → time series |
| `tombstones` | Records deletions |
| `meta.json` | Block metadata |

**Typical data directory layout:**

```
./data
├── 01BKGV7JBM69T2G1BGBGM6KB12
│   └── meta.json
├── 01BKGTZQ1SYQJTR4PB43C8PD98
│   ├── chunks
│   │   └── 000001
│   ├── tombstones
│   ├── index
│   └── meta.json
├── chunks_head
│   └── 000001
└── wal
    ├── 000000002
    └── checkpoint.00000001
        └── 00000000
```

### 🔗 Chunks
A **chunk** is the unit of compressed sample storage inside a block's `chunks/` directory.

- Segment files in `chunks/` are up to **512 MB** by default.
- **Float (XOR) chunks** → limited to **1024 bytes**.
- **Native histogram chunks** → same 1024-byte limit, but at least **10 histograms** must be stored first before the limit kicks in (so they *can* exceed 1024 bytes).
- Encoding: **delta-of-delta (double delta)** for timestamps + **XOR** for float values — inspired by the **Gorilla** compression algorithm.

### 📋 Relationship Summary

| Concept | Role |
|---|---|
| **Block** | Time-bounded directory of data (starts at 2h, compacted later) |
| **Chunk** | Compressed samples for *one* time series, stored inside a block |
| **Segment file** | A file inside `chunks/` holding multiple chunks (up to 512 MB) |
| **WAL** | Protects the in-memory head block against crashes |

> 💡 **In short:** a block contains many chunks, and each chunk holds compressed samples for a single time series over part of that block's time range.

---

## 2. 🖥️ Local Prometheus Write Flow

```
   Target (app/exporter)
          │
          │  HTTP scrape (pull model)
          ▼
   Prometheus Retrieval
          │
          │  append sample
          ▼
   Write-Ahead Log (WAL)          ◄── crash protection, 128 MB segments
          │
          │  also written to
          ▼
   In-Memory Head Block           ◄── active 2-hour window, not yet persisted
          │
          │  every ~2 hours, head block is flushed
          ▼
   Persistent 2-hour Block        ◄── chunks/ + index + tombstones + meta.json
          │
          │  background compaction
          ▼
   Larger Compacted Blocks        ◄── up to 10% of retention time or 31 days
          │
          │  retention policy triggers
          ▼
   Block Deletion                 ◄── expired blocks removed (up to 2h delay)
```

### 🔑 Key Details at Each Step

| Step | Detail |
|---|---|
| **Scrape** | Pull-based; Prometheus scrapes targets at configured intervals |
| **WAL** | Stored in `wal/` directory, 128 MB segments, minimum 3 files kept |
| **Head block** | Kept in memory; secured by WAL for crash recovery |
| **2-hour block** | Written to disk with `chunks/`, `index`, `tombstones`, `meta.json` |
| **Compaction** | Merges small blocks into larger ones in the background |
| **Retention** | Controlled by `retention.time` (default 15d) or `retention.size` |

---

## 3. ☁️ Prometheus + Thanos Write Flow

Thanos extends the local flow by adding a **sidecar** that ships completed blocks to object storage.

```
   Target (app/exporter)
          │
          │  HTTP scrape (pull model)
          ▼
   Prometheus Retrieval
          │
          │  append sample
          ▼
   Write-Ahead Log (WAL)          ◄── same as local
          │
          ▼
   In-Memory Head Block           ◄── same as local
          │
          │  every ~2 hours
          ▼
   Persistent 2-hour Block on disk
          │
          ├─────────────────────────────┐
          ▼                             ▼
   Local Storage               Thanos Sidecar
   (retention applies)         (watches data dir)
                                        │
                                        │  uploads completed blocks
                                        ▼
                              Object Storage (S3, GCS, etc.)
                                        │
                          ┌─────────────┴─────────────┐
                          ▼                            ▼
                 Thanos Store Gateway          Thanos Compactor
                 (serves old blocks             (compacts blocks
                  from object storage)            in object storage)
                          │
                          ▼
                  Thanos Querier
             (fan-out queries across
              Prometheus + Store Gateway)
```

### 🔑 Key Details for the Thanos Path

| Component | Role |
|---|---|
| **Thanos Sidecar** | Runs alongside the Prometheus pod; uploads every new 2-hour block to object storage |
| **Object Storage** | S3, GCS, etc.; configured via a Kubernetes Secret |
| **Thanos Store Gateway** | Serves historical blocks from object storage to the Querier |
| **Thanos Compactor** | Handles compaction of blocks in object storage (singleton) |
| **Thanos Querier** | Provides a global query view across multiple Prometheus instances |

### ⚙️ Compaction Behavior Note

| Version Combo | Behavior |
|---|---|
| Prometheus **≥ v3.9.0** + Sidecar **≥ v0.42.0** | Local compaction **stays enabled**. Prometheus only compacts blocks already uploaded to object storage (coordinated via the sidecar's shipper meta file) — avoids memory/query overhead |
| Older versions | Sidecar **disables** local compaction (`--storage.tsdb.min-block-duration == --storage.tsdb.max-block-duration`); Thanos Compactor becomes solely responsible |

### 🔧 Minimal Thanos Sidecar Config (Prometheus Operator)

```yaml
spec:
  thanos:
    image: quay.io/thanos/thanos:v0.28.1
    objectStorageConfig:
      key: thanos.yaml
      name: thanos-objstore-config
```

**Create the object storage secret:**

```bash
kubectl -n monitoring create secret generic thanos-objstore-config \
  --from-file=thanos.yaml=/tmp/thanos-config.yaml
```

**Example `thanos-config.yaml`:**

```yaml
type: s3
config:
  bucket: thanos
  endpoint: ams3.digitaloceanspaces.com
  access_key: XXX
  secret_key: XXX
```

---

## 4. 🔀 Local vs. Thanos — Quick Comparison

| Aspect | Local Only | With Thanos |
|---|---|---|
| Data retention | Limited to local disk (`retention.time`/`retention.size`) | Extended indefinitely via object storage |
| Compaction | Handled entirely by Prometheus | Shared or offloaded to Thanos Compactor (version-dependent) |
| Querying | Single Prometheus instance | Global view across multiple instances via Querier |
| Historical data access | From local blocks only | From Store Gateway (object storage) |
| Durability | Tied to local disk | Durable via S3/GCS-backed object storage |

---

*Sources: Prometheus Storage docs, Thanos integration & sidecar config docs.*
