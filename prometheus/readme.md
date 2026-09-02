# 🧭 Reading Order — 3 Prometheus Docs

| Order | File | Why |
|---|---|---|
| **1st** | `3-prometheus-and-promql-full-guide.md` | Foundations — architecture, config, PromQL basics, alerting. Everything else assumes this. |
| **2nd** | `2-local-storage-deep-dive.md` | Goes deep on the TSDB/WAL/compaction/backfill internals only briefly touched in Doc 3. |
| **3rd** | `1-scaling-and-long-term-storage.md` | Operational layer on top — what to do once one server + local disk isn't enough (sharding, Thanos, federation). |

**Rule of thumb:** Doc 3 = *learn Prometheus*, Doc 2 = *understand the disk*, Doc 1 = *scale it up*.
-e 
---

# 📘 Doc 3: Prometheus & PromQL — Complete Concept-by-Concept Guide
**Source file:** The big one — 7 parts, covers Prometheus end-to-end from zero

## What it's for
This is the **foundational, start-to-finish course**. If you're new to Prometheus, this is where you begin — Docs 1 & 2 assume you already know what's covered here.

## Contents (7 parts)
1. **Observability Fundamentals** — logs/metrics/traces, SLI/SLO/SLA, error budgets
2. **Prometheus Fundamentals** — architecture, install, Node Exporter, config (`prometheus.yml`), TLS/auth, data model (metric = name+labels+value+timestamp), Docker, cAdvisor, `promtool`
3. **PromQL** — the big one:
   - selectors/matchers, `offset`/`@` modifiers
   - operators (arithmetic, comparison, `bool`, `and/or/unless`)
   - **vector matching** (`on`, `ignoring`, `group_left/right`) — classic source of confusion
   - aggregation (`by`/`without`)
   - functions (`rate` vs `irate`, `absent`, etc.)
   - subqueries, histograms/summaries + `histogram_quantile()`
   - recording rules, HTTP API
4. **Dashboarding** — Expression Browser vs Console Templates vs Grafana
5. **Instrumentation** — client libraries, Flask example, Counter/Gauge/Histogram/Summary, labels, naming best practices
6. **Service Discovery** — file/EC2/k8s discovery, **relabeling** (`keep`/`drop`/`replace`/`labelmap`)
7. **Alerting** — alert lifecycle (Inactive→Pending→Firing), labels vs annotations, Alertmanager architecture, routing tree, receivers, silences

## Best read when
**Read this one FIRST.** It's the prerequisite for the other two — everything in Docs 1 & 2 (TSDB blocks, retention, remote write, PromQL basics) is introduced here first, just at intro depth.
-e 
---

# 📘 Doc 2: Prometheus Local Storage — Deep-Dive Reference
**Source file:** Based on official Prometheus Storage docs, expanded with diagrams/tables (v3.14)

## What it's for
A **detailed reference** on how Prometheus's built-in TSDB actually works on disk — deeper and more mechanical than Doc 1. Good for understanding internals, troubleshooting disk issues, and backfilling.

## Contents (in order)
1. **Local Storage Overview** — TSDB is Prometheus's own engine, no external DB needed
2. **On-Disk Layout** — 2h block cycle, `chunks/` segment files (512MB cap), tombstones (delete markers, not real deletes), the in-memory "head block"
3. **WAL (Write-Ahead Log)** — full detail: 128MB segments, min 3 files, guarantees ≥2h crash-recovery coverage, `wal-compression` flag
4. **Filesystem rules** — POSIX-only, **never NFS/EFS** (corruption risk)
5. **Compaction** — merges blocks up to `min(10% of retention, 31 days)`; temporarily uses *more* disk than your retention limit while running
6. **Operational Flags** — `--storage.tsdb.retention.time/size`, the `retention.size` gotcha (WAL + head chunks still count toward the total!)
7. **Right-Sizing Retention** — set `retention.size` to 80–85% of disk, leave buffer for compaction overshoot
8. **Remote Storage** — 4 integration directions (write/read × client/server), and the key limitation: PromQL always evaluates *locally*, remote is just a data pipe
9. **Backfilling** — from OpenMetrics format (`promtool tsdb create-blocks-from openmetrics`) and for recording rules (`...from rules`), plus their limitations
10. **Quick Reference table** — every key fact on one page

## Best read when
You need the **"why" behind disk usage, corruption, backups, or backfilling** — this is the internals doc, denser than Doc 1's TSDB intro.
-e 
---

# 📘 Doc 1: Prometheus Scaling & Long-Term Storage
**Source file:** *Prometheus Certified Associate · Scaling & Long-Term Storage Module* (15 pages)

## What it's for
Covers what to do **once one Prometheus server isn't enough** — scaling it and getting data out of local disk into cheaper long-term storage. Exam-prep style, with a "memory hook" per section.

## Contents (in order)
1. **TSDB & Retention** — block structure (`chunks`, `index`, `meta.json`, `tombstones`), retention flags, capacity formula (`Time × Samples/sec × Bytes`)
2. **Remote Read/Write** — pushing/pulling data to external stores (S3, InfluxDB, VictoriaMetrics)
3. **Thanos** — Sidecar → Store → Query architecture for global view + cheap long-term storage
4. **Scaling** — Vertical (different services, different servers) vs Horizontal (same service, split load)
5. **Horizontal Sharding** — `hashmod` relabeling recipe (3 steps) to split targets across instances
6. **Quick Wins** — widen `scrape_interval`, use `sample_limit` before reaching for bigger fixes
7. **Federation** — the *correct* pattern: aggregate locally with recording rules → label with origin → federate only the summary (never raw metrics)
8. **Final Cheat Sheet** — one-page recap of every flag/config above

## Best read when
You already know basic Prometheus and need to **scale it or ship data long-term** — i.e., an operational/architecture concern, not a beginner topic.
