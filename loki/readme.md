# 🗺️ Grafana Loki — Mastery Roadmap
*A structured, no-gaps learning path from "what is it" to production-grade operation.*

---

## 0. The One-Sentence Mental Model
> **Loki = Prometheus, but for logs.** It does NOT index log content (full-text) — it only indexes **labels** (metadata), and stores the actual log lines as compressed blobs. This single design choice explains almost every architectural decision below.

Why this matters: every other log system (ELK/Elasticsearch, Splunk) indexes every word in every log line → expensive, huge index. Loki indexes only labels (like `job`, `namespace`, `pod`) → tiny index, cheap storage, but queries must scan chunks (mitigated by good label design + parallelism).

---

## PHASE 1 — Core Concepts (Every DevOps Engineer MUST Know)

### 1.1 What Loki Is / Isn't
| Is | Isn't |
|---|---|
| A log **aggregation** system | A full-text search engine (not Elasticsearch) |
| Index = labels only | Index = every word (that's ELK) |
| Built to pair with Prometheus + Grafana | A metrics system (that's Prometheus/Mimir) |
| Cheap because object storage does the heavy lifting | Cheap if you use it with bad label design (cardinality kills it) |

### 1.2 The LGTM Stack (know where Loki fits)
```
Logs      → Loki
Metrics   → Prometheus / Mimir
Traces    → Tempo
Dashboard → Grafana (the "G" — the single pane of glass over all three)
```
Understanding this stack relationship is a baseline DevOps interview question.

### 1.3 Streams — the fundamental unit
A **stream** = a unique set of label key/value pairs. Every log line belongs to exactly one stream.
```
{job="nginx", namespace="prod", pod="nginx-7f4-abc"}
```
- Log lines within a stream are stored together, compressed, in order.
- A **new label value = a new stream** → this is the #1 source of cardinality explosions.

### 1.4 Labels vs Content
- **Labels** = structured metadata, indexed, used to *find* the right streams (cheap, small cardinality: `app`, `env`, `namespace`, `level`).
- **Content** = the raw log line, NOT indexed, searched via grep-like scanning at query time (LogQL line filters).
- **Golden rule:** never put unbounded/high-cardinality values (user ID, trace ID, IP, timestamp, request path) into a label. Put them in the log line and filter on them with LogQL instead.

### 1.5 LogQL — the query language
Two query types:
1. **Log queries** — return raw matching log lines
   ```logql
   {namespace="prod", app="checkout"} |= "error" != "timeout"
   ```
2. **Metric queries** — aggregate logs into numeric time series (Prometheus-style)
   ```logql
   sum(rate({namespace="prod", app="checkout"} |= "error" [5m])) by (pod)
   ```

Core LogQL building blocks:
| Stage | Purpose | Example |
|---|---|---|
| Stream selector | Pick streams by label | `{app="api"}` |
| Line filter | Grep-style filter on raw text | `\|= "500"`, `!= "healthcheck"`, `\|~ "regex"` |
| Parser | Extract fields from line into labels at query time | `\| json`, `\| logfmt`, `\| regexp`, `\| pattern` |
| Label filter | Filter on parsed/extracted fields | `\| status_code >= 500` |
| Line format | Reshape output | `\| line_format "{{.status}}"` |
| Metric functions | Turn logs into numbers | `rate()`, `count_over_time()`, `bytes_rate()`, `quantile_over_time()` |

### 1.6 Log Collection Agents (the "how logs get in")
| Agent | Notes |
|---|---|
| **Promtail** | Loki's original purpose-built agent (being phased toward Alloy) |
| **Grafana Alloy** | Newer unified agent (OTel-based) — the current recommended path forward |
| **Fluent Bit / Fluentd** | Common in k8s, has a Loki output plugin |
| **OpenTelemetry Collector** | Vendor-neutral, growing standard — has a Loki exporter |
| **Docker driver / systemd journal** | Direct log shipping for containers/VMs |

Every agent does the same 3 jobs: **discover** targets → **attach labels** → **push to Loki's `/loki/api/v1/push` endpoint**.

### 1.7 Retention & Cost model
- Retention configured per-tenant, enforced by the **Compactor**.
- Storage cost ≈ near-zero marginal cost because it's just object storage (S3/GCS/Azure Blob) — this is Loki's core value proposition vs. ELK.

---

## PHASE 2 — Architecture (the part that separates "used Loki" from "understands Loki")

### 2.1 The Write Path (how a log line gets stored)
```
Agent (Promtail/Alloy)
   → Distributor (validates, hashes stream → consistent hashing to ingesters)
   → Ingester (buffers in memory, builds compressed "chunks", writes WAL for crash-safety)
   → Chunk flushed to Object Storage (S3/GCS/Azure Blob) when full or time-based
   → Index entry written (chunk ↔ label mapping) to Index store
```

### 2.2 The Read Path (how a query runs)
```
Grafana / LogCLI
   → Query Frontend (splits query into shards, queues, caches results)
   → Querier (fetches matching chunks from ingesters [recent] + object storage [historical])
   → Merges & returns results
```

### 2.3 Core Components — know each one's ONE job
| Component | Job |
|---|---|
| **Distributor** | Entry point for writes; validates & hashes streams to the right ingester (consistent hashing ring) |
| **Ingester** | Holds recent log data in memory, batches into chunks, flushes to object storage, writes WAL |
| **Query Frontend** | Splits large queries into smaller shards, parallelizes, caches, protects queriers from overload |
| **Querier** | Executes the actual LogQL query against ingesters (recent) + storage (historical), merges results |
| **Index Gateway** | Serves index lookups (which chunks match these labels?) — used with TSDB/boltdb-shipper index |
| **Compactor** | Merges small index files into larger ones, applies retention (deletes expired data) |
| **Ruler** | Evaluates recording rules & alerting rules directly on log data (LogQL → alerts) |
| **Query Scheduler** (optional, large-scale) | Decouples frontend from queriers, smarter queueing |

### 2.4 The Hash Ring
- Distributor and Ingesters participate in a **consistent hash ring** (like Cassandra/DynamoDB style) to decide which ingester owns which stream.
- Enables horizontal scaling of ingesters without a central bottleneck.
- Ring state stored in etcd / Consul / memberlist (gossip).

### 2.5 Deployment Modes — pick based on scale
| Mode | Description | When to use |
|---|---|---|
| **Monolithic** | Single binary runs all components (`-target=all`) | Dev, small teams, POC, <~few GB/day |
| **Simple Scalable (SSD)** | 3 targets: `read`, `write`, `backend` — each independently scalable | Most production use cases (recommended default now) |
| **Microservices** | Every component (distributor, ingester, querier, etc.) is its own deployment/scaling group | Very large scale (100s of GB–TB/day), needs fine-grained scaling |

---

## PHASE 3 — Storage Architecture (deep dive — the part most engineers skip and regret)

### 3.1 Two Storage Concerns — DO NOT confuse them
1. **Chunk storage** — the actual compressed log data (always object storage: S3, GCS, Azure Blob, or filesystem for dev)
2. **Index storage** — maps labels → which chunks contain them (this is what's evolved over Loki's versions — see below)

### 3.2 Index Evolution (know the history — comes up in interviews & migrations)
| Era | Index type | Notes |
|---|---|---|
| Old (Loki <2.0) | Cassandra / BigTable / DynamoDB / BoltDB (single node) | External DB dependency, operational overhead |
| Loki 2.x | **boltdb-shipper** | Index built locally as BoltDB files, "shipped" to object storage periodically — no external DB needed |
| Loki 2.9+ / 3.x (current best practice) | **TSDB index** | Same idea as boltdb-shipper but faster, more efficient index format (borrowed from Prometheus TSDB) — **this is the current recommended default** |

**Key insight:** Loki moved from "needs an external index database" → "index lives in object storage too" — this is what makes Loki genuinely simple to operate at scale (single storage dependency: object storage).

### 3.3 Chunk Lifecycle
1. Ingester receives log lines for a stream, appends to an in-memory chunk (compressed with gzip/snappy/lz4/zstd).
2. Chunk is flushed to object storage when: it hits `max_chunk_age`, size threshold, or ingester needs to free memory / is shutting down.
3. Compactor later merges small per-ingester index files into a compact, per-day index.

### 3.4 WAL (Write-Ahead Log)
- Same purpose as Prometheus's WAL: protects in-memory ingester data from being lost on crash/restart.
- On restart, ingester replays the WAL to rebuild its in-memory chunks before accepting new writes.

### 3.5 Schema Config — the "gotcha" every beginner hits
Loki's `schema_config` defines **which index/storage format applies from which date forward**. This is append-only — you add a new schema entry with a future `from` date rather than editing history. Mis-set schema is one of the most common real-world Loki misconfig issues.
```yaml
schema_config:
  configs:
    - from: 2024-01-01
      store: tsdb
      object_store: s3
      schema: v13
      index:
        prefix: index_
        period: 24h
```

### 3.6 Retention & Compaction
- Retention is enforced by the **Compactor**, not by the ingesters.
- Two retention modes: global (`limits_config.retention_period`) or per-tenant overrides.
- Compactor also handles **deletion requests** (GDPR-style "delete these logs" API).

### 3.7 Caching layers (for query performance at scale)
| Cache | What it speeds up |
|---|---|
| Chunk cache | Avoid re-fetching the same chunk from object storage |
| Index cache | Avoid re-querying the index store |
| Results cache | Cache full query results (huge win for repeated dashboard queries) |
Typically backed by **Memcached** or **Redis** in production.

### 3.8 Multi-Tenancy
- Every request carries an `X-Scope-OrgID` header — Loki is multi-tenant by design (even single-tenant setups just use one fixed org ID).
- Enables shared clusters across teams with isolated limits, retention, and data.

---

## PHASE 4 — Production Operations (where the real gaps usually are)

- [ ] **Cardinality control** — audit label sets before shipping; use `logfmt`/`json` parsing at query time instead of extra labels
- [ ] **Rate limiting & limits_config** — per-tenant ingestion rate, max streams, max query length/parallelism
- [ ] **Alerting via Ruler** — write LogQL-based alerting rules (e.g., error rate thresholds), routed through Alertmanager
- [ ] **Recording rules** — precompute expensive LogQL metric queries (same concept as Prometheus recording rules)
- [ ] **Sizing ingesters/queriers** — replication factor, `chunk_target_size`, `max_chunk_age`
- [ ] **Backend choice** — S3 vs GCS vs Azure Blob vs MinIO (self-hosted S3-compatible) for storage
- [ ] **Observing Loki itself** — Loki exposes its own Prometheus metrics; monitor ingester memory, flush failures, compactor lag
- [ ] **Upgrades** — schema_config migrations, boltdb-shipper → TSDB index migration path
- [ ] **Security** — auth via reverse proxy (Loki has no built-in auth), TLS, per-tenant isolation

---

## PHASE 5 — Hands-On Mastery Checklist
Work through these in order; each builds on the last.

1. **Local single-binary Loki** (`docker-compose` with Loki + Promtail + Grafana) — ship your own container logs, view in Grafana Explore.
2. **Write LogQL** — practice line filters, `json`/`logfmt` parsers, label filters, and one metric query (`rate`, `count_over_time`).
3. **Deploy Simple Scalable mode** — split into `read`/`write`/`backend` targets, point at real S3/MinIO for storage.
4. **Configure TSDB index + schema_config** correctly with a future `from` date.
5. **Set retention** globally, then override per-tenant.
6. **Write a Ruler alert** — e.g., "error rate > 5% for 5m" — and route it through Alertmanager.
7. **Add caching** (Memcached) for index/chunk/results and benchmark query latency before/after.
8. **Cause and diagnose a cardinality explosion on purpose** (label with something unbounded) — watch ingester memory spike — then fix it. This lesson sticks better than reading about it.
9. **Migrate an agent** from Promtail to Grafana Alloy — understand the OTel-native direction Loki is heading.
10. **Build a full LGTM demo** — Loki + Prometheus + Tempo + Grafana, with trace-to-log correlation (`traceID` linking) — this is the "I actually get the whole stack" milestone.

---

## Quick Reference — One-Page Recall
```
Loki = Prometheus for logs → indexes LABELS only, not content
Write path:  Agent → Distributor → Ingester → Object Storage (chunks) + Index
Read path:   Grafana → Query Frontend → Querier → merges chunks+index results
Components:  Distributor, Ingester, Query Frontend, Querier, Index Gateway, Compactor, Ruler
Index evolution: Cassandra/BigTable → boltdb-shipper → TSDB (current default)
Deployment:  Monolithic → Simple Scalable (recommended) → Microservices
Storage:     Object storage (S3/GCS/Azure Blob) for BOTH chunks and index now
Retention:   Enforced by Compactor, per-tenant configurable
Cardinality: #1 operational risk — never label unbounded values
LogQL:       {selector} |= "filter" | parser | label_filter | metric_fn()
```
