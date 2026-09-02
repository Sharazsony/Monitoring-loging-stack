# kube-prometheus-stack — Quick Revision Guide

## 1. The Big Picture (4 stages)
| Stage | Tool |
|---|---|
| Collect | node-exporter, kube-state-metrics, cAdvisor |
| Store | Prometheus (TSDB) |
| Query/Visualize | PromQL + Grafana |
| Alert | Alertmanager |

## 2. Prometheus Core Ideas
- **Pull model**: Prometheus scrapes `/metrics` from targets (default every 30s). Apps don't push.
- **Metric = name + labels + value**, e.g. `node_cpu_seconds_total{cpu="0",mode="idle"} 45234.66`
- **4 metric types**:
  - Counter → only goes up (requests served)
  - Gauge → up/down (current memory)
  - Histogram → buckets (request latency)
  - Summary → like histogram, client-side quantiles (rarely used)
- **TSDB**: local time-series DB, default retention ~15d. For longer history → Thanos/Mimir/VictoriaMetrics via `remote_write`.

## 3. Prometheus Operator
- Problem: hand-editing `prometheus.yml` doesn't scale on dynamic K8s clusters.
- Fix: Operator watches **CRDs** and auto-rewrites Prometheus config + reloads.

| CRD | Purpose |
|---|---|
| `Prometheus` | Run a Prometheus server (replicas, retention, resources) |
| `ServiceMonitor` | Scrape pods behind a Service |
| `PodMonitor` | Scrape pods directly (no Service) |
| `PrometheusRule` | Alerting + recording rules |
| `Alertmanager` | Run Alertmanager cluster |
| `AlertmanagerConfig` | Routing/receivers per team |

⚠️ **#1 beginner trap**: `ServiceMonitor` labels must match Prometheus CRD's `serviceMonitorSelector` (usually `release: <helm-release-name>`), or it's silently ignored.

## 4. Stack Components
| Component | Type | Job |
|---|---|---|
| Prometheus Operator | Deployment | Watches CRDs, manages config |
| Prometheus server | StatefulSet | Scrapes, stores, evaluates rules |
| Alertmanager | StatefulSet | Routes/dedupes/notifies |
| Grafana | Deployment | Dashboards (read-only, queries Prometheus) |
| kube-state-metrics | Deployment | K8s **object** state (pod phase, replicas) — reads API |
| node-exporter | DaemonSet | **Hardware/OS** metrics (CPU, disk, net) — reads kernel |
| Metrics Server | Deployment (separate) | Powers `kubectl top` + HPA only, NOT Grafana |

**Memorize the difference**: node-exporter = machine facts · kube-state-metrics = Kubernetes object facts · Metrics Server = fast numbers for HPA only.

## 5. End-to-End Flow
```
Pod exposes /metrics → Prometheus scrapes+stores → PrometheusRule evaluates
→ alert fires → Alertmanager groups/routes → Slack/Discord/Email
```
Grafana runs in parallel: sends PromQL queries to Prometheus whenever a dashboard is opened.

## 6. Install (Helm) — Minimal Commands
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
kubectl create namespace monitoring
helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
kubectl get pods -n monitoring
kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring
```
Get Grafana password:
```bash
kubectl get secret monitoring-grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 --decode
```

## 7. PromQL Essentials
- `metric_name` → instant vector (latest value)
- `metric_name[5m]` → range vector (raw samples, last 5 min)
- `rate(counter[5m])` → per-second rate — **use for all counters**
- Aggregations: `sum()`, `avg()`, `max()/min()`, `count()`, `by(label)` = GROUP BY

Useful queries:
```promql
# CPU utilization %
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Crash-looping pods (last 1h)
increase(kube_pod_container_status_restarts_total[1h]) > 0

# p95 latency
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
```

## 8. Alertmanager Concepts
- **Prometheus detects** (condition true) → **Alertmanager decides notification** (who/how/how often). Kept separate on purpose.
- Route → Receiver → Grouping → Silence → Inhibition
- Key timers: `groupWait` (wait before first send) · `groupInterval` (wait before adding new alerts to a sent group) · `repeatInterval` (resend interval for still-firing alert)

## 9. Requests/Limits & HPA
- HPA compares usage against **requests**, not limits.
- Always set both `requests` and `limits` — no limit = pod can starve the whole node.
- HPA reads from **Metrics Server**, not Prometheus (needs fast/low-latency numbers).

## 10. Production Checklist (short version)
- Set resource requests/limits on every stack pod
- Tune `retention` + `retentionSize` explicitly
- Use real PVC/StorageClass (not emptyDir)
- 2+ Prometheus replicas, 3+ Alertmanager replicas
- Add Thanos/Mimir for >15d history
- Avoid high-cardinality labels (user IDs, request IDs)
- Grafana behind Ingress + TLS + SSO (never public NodePort)

## 11. Troubleshooting Quick Table
| Symptom | Likely Cause |
|---|---|
| Target missing in Prometheus | ServiceMonitor label mismatch |
| HPA "unable to fetch metrics" | Metrics Server not installed |
| Alert never fires | Wrong AlertmanagerConfig route/label |
| Grafana panel empty | Wrong data source / bad PromQL |
| Prometheus OOMKilled | High cardinality or retention too high for disk |
| node-exporter missing on some nodes | DaemonSet tolerations vs taints mismatch |

## 12. One-Line Glossary
- **Scrape** — Prometheus pulling `/metrics` over HTTP
- **Exporter** — process exposing metrics in Prometheus format
- **Cardinality** — # of unique label combos — the #1 scaling risk
- **Remote write** — streaming Prometheus data to long-term storage
