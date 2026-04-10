# Observability — 80 Scenarios

> **ID prefix:** O- | Types: Design, Implement, Troubleshoot, Explain | Difficulty: Medium → Expert

---

## Section 1: Prometheus and PromQL (O-001 to O-025)

---

### O-001 | Prometheus Architecture and Scrape Flow
**Type:** Explain | **Difficulty:** Medium | ⬜

> Explain Prometheus's pull model, service discovery, and how metrics flow from an application to a Grafana dashboard.

**What you must cover:**
- Application: exposes `/metrics` endpoint in Prometheus exposition format
- Service discovery: Prometheus uses `kubernetes_sd_configs` to discover pods with annotation `prometheus.io/scrape: "true"`
- Scrape: Prometheus pulls metrics from each target every `scrape_interval` (default 1m)
- TSDB: time series stored locally in 2-hour blocks, compacted to longer-term chunks
- Remote write: Prometheus → Thanos/Cortex/Mimir for long-term storage
- PromQL: query at read time; Grafana queries Prometheus HTTP API
- Alertmanager: Prometheus evaluates `alert` rules, fires to Alertmanager, which routes to PagerDuty/Slack

---

### O-002 | Counter vs Gauge vs Histogram — Rate Calculation
**Type:** Explain | **Difficulty:** Hard | ⬜

> A developer asks: "Why is my request rate showing 0 even though requests are happening?" What's the issue?

```promql
# WRONG - instant value of a counter
http_requests_total

# CORRECT - rate of change over 5 min window
rate(http_requests_total[5m])

# Why 5m? Rule of thumb: at least 4× scrape_interval (typically 1m → use 5m)
# rate() handles counter resets (pod restarts) correctly
# increase() is rate() × range duration — for display only, not alerting
```
- `rate()` returns per-second average — always use for alerting
- `irate()` — instant rate (last 2 samples) — spiky, not good for alerting
- Counter resets: Prometheus `rate()` and `increase()` detect and handle automatically

---

### O-003 | PromQL — Real Alerting Rules
**Type:** Implement | **Difficulty:** Hard | ⬜

> Write PromQL alerting rules for: P99 latency > 500ms and error rate > 1%.

```yaml
# prometheus/rules/alerts.yml
groups:
- name: api-slos
  interval: 1m
  rules:
  
  - alert: HighP99Latency
    expr: |
      histogram_quantile(0.99,
        sum by (service, le) (
          rate(http_request_duration_seconds_bucket{job="api"}[5m])
        )
      ) > 0.5
    for: 5m
    labels:
      severity: warning
      team: platform
    annotations:
      summary: "P99 latency > 500ms for {{ $labels.service }}"
      description: "P99 latency is {{ $value | humanizeDuration }}"
  
  - alert: HighErrorRate
    expr: |
      (
        sum by (service) (rate(http_requests_total{status=~"5.."}[5m]))
        /
        sum by (service) (rate(http_requests_total[5m]))
      ) > 0.01
    for: 5m
    labels:
      severity: critical
```
- `for: 5m`: alert must be true for 5 min before firing — prevents flapping
- `histogram_quantile`: always apply to `rate()` of bucket, not instant values

---

### O-004 | Prometheus Cardinality Problem
**Type:** Troubleshoot | **Difficulty:** Expert | ⬜

> Prometheus is OOMing. `tsdb status` shows one metric has 2 million time series. What caused this and how do you fix it?

```bash
# Query TSDB status
curl http://prometheus:9090/api/v1/status/tsdb | jq .

# Find high-cardinality metrics
topk(10, count by (__name__)({__name__=~".+"}))

# Common culprit: user_id, request_id, trace_id as labels
http_requests_total{user_id="12345"}  # WRONG - unbounded cardinality
http_requests_total{service="api", status="200"}  # CORRECT
```
**Fixes:**
- Drop high-cardinality labels with `metric_relabel_configs` in scrape config
- Move high-cardinality data to log-based metrics or tracing
- Enforce label cardinality limits in instrumentation guidelines
- Prometheus: `--storage.tsdb.max-block-size` and OOM prevention via resource limits + VPA

---

### O-005 | Prometheus Federation and Thanos
**Type:** Design | **Difficulty:** Expert | ⬜

> Design a multi-cluster observability architecture with Prometheus for 20 clusters with 90-day retention.

**What you must cover:**
- Per-cluster: local Prometheus with short retention (2-7d) — scrapes cluster metrics
- Thanos Sidecar: attached to each Prometheus, uploads TSDB blocks to Azure Blob Storage
- Thanos Query: global query layer — fans out queries to all Sidecars or Store Gateways
- Thanos Store Gateway: reads from Blob Storage for historical queries (> local retention)
- Thanos Compactor: deduplicate, downsample, apply retention policies
- Thanos Ruler: evaluate alerting rules globally (cross-cluster aggregations)
- Grafana: points to Thanos Query as data source — queries across all clusters transparently

---

### O-006 | Recording Rules for Dashboard Performance
**Type:** Implement | **Difficulty:** Hard | ⬜

> A Grafana dashboard loads in 30 seconds. The main query aggregates sum over 10,000 pods. How do you fix?

```yaml
# Before: heavy query run at dashboard load time
sum(rate(container_cpu_usage_seconds_total[5m])) by (namespace, pod)  # Slow!

# After: pre-compute with recording rule
groups:
- name: k8s.aggregates
  interval: 1m
  rules:
  - record: namespace:container_cpu_usage_seconds:sum_rate5m
    expr: |
      sum by (namespace) (
        rate(container_cpu_usage_seconds_total[5m])
      )
```
- Recording rule computes at `interval`, stores as new metric
- Dashboard query: `namespace:container_cpu_usage_seconds:sum_rate5m` — instant lookup, <1s
- Naming convention: `level:metric:operations` (Prometheus docs standard)

---

### O-007 | SLI/SLO Implementation with Prometheus
**Type:** Design | **Difficulty:** Expert | ⬜

> Implement SLO tracking for 99.9% availability with burn rate alerting.

```yaml
# SLI: good requests = 2xx or 3xx. Bad = 5xx.
# SLO: 99.9% availability (error budget: 0.1% = 43.8 min/month)

rules:
# 1-hour burn rate (fast burn)
- alert: SLOFastBurn
  expr: |
    (
      1 - (
        sum(rate(http_requests_total{status!~"5.."}[1h]))
        / sum(rate(http_requests_total[1h]))
      )
    ) > (14.4 * 0.001)  # 14.4x burn rate = burn 2% budget in 1 hour
  for: 2m
  labels: {severity: critical, ticket: "yes"}

# 6-hour burn rate (slow burn)  
- alert: SLOSlowBurn
  expr: |
    (
      1 - (
        sum(rate(http_requests_total{status!~"5.."}[6h]))
        / sum(rate(http_requests_total[6h]))
      )
    ) > (6 * 0.001)  # 6x burn rate = burn 5% budget in 6 hours
  for: 15m
  labels: {severity: warning}
```
- Multi-window multi-burn-rate: Google's SRE workbook approach
- Error budget: track remaining in Grafana — freeze feature work when >50% burned

---

### O-008 | Prometheus Alertmanager Routing
**Type:** Implement | **Difficulty:** Hard | ⬜

> Configure Alertmanager to route critical alerts to PagerDuty, warning to Slack, and silence low-severity during business hours.

```yaml
# alertmanager.yaml
route:
  receiver: 'default-slack'
  group_by: ['alertname', 'service', 'env']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  
  routes:
  - match:
      severity: critical
    receiver: pagerduty-production
    continue: true   # Also send to Slack
  
  - match:
      severity: critical
    receiver: slack-critical
  
  - match_re:
      severity: warning|info
    receiver: slack-general
    active_time_intervals: [business-hours]

time_intervals:
- name: business-hours
  time_intervals:
  - weekdays: [monday:friday]
    times: [{start_time: '09:00', end_time: '18:00'}]

receivers:
- name: pagerduty-production
  pagerduty_configs:
  - routing_key: ${{ secrets.PD_KEY }}
    severity: '{{ .CommonLabels.severity }}'
    description: '{{ .CommonAnnotations.summary }}'
```

---

### O-009 to O-025 | Prometheus and PromQL Rapid-Fire

---

### O-009 | Write a PromQL query to find pods consuming more than 80% of their CPU limits. | Implement | Hard | ⬜
**Hint:** `container_cpu_usage_seconds_total` (rate) / `kube_pod_container_resource_limits_cpu_cores` > 0.8. Join on `pod`, `container`, `namespace`.

### O-010 | Prometheus scrape is returning zero metrics for an application. Walk through debugging. | Troubleshoot | Hard | ⬜
**Hint:** Check `prometheus.io/scrape: "true"` annotation. Target in Prometheus UI (Status → Targets). Port/path correct? `prometheus.io/port: "8080"`, `prometheus.io/path: /metrics`. Network policy blocking scrape port.

### O-011 | Explain the difference between `rate()`, `irate()`, `increase()`, and `delta()` in PromQL. | Explain | Hard | ⬜
**Hint:** `rate()`: per-second avg over window. `irate()`: per-second from last 2 samples. `increase()`: extrapolated total increase over window. `delta()`: for gauges — difference, not rate.

### O-012 | Your Prometheus is consuming 150GB RAM. Causes and remediation? | Troubleshoot | Expert | ⬜
**Hint:** High series count (cardinality explosion). Long retention. Wide scrape interval means large head block. Fix: reduce cardinality (drop labels), reduce retention, add memory limit + VPA, consider Prometheus WAL-E or Thanos.

### O-013 | Implement Prometheus scraping for a service that exposes metrics on multiple ports. | Implement | Medium | ⬜
**Hint:** `ServiceMonitor` with multiple `endpoints:` entries. Or: separate `PodMonitor` per container port. Each endpoint gets its own job label.

### O-014 | Write a PromQL query to detect when any deployment has fewer replicas than desired for > 5 minutes. | Implement | Hard | ⬜
**Hint:** `kube_deployment_status_replicas_available < kube_deployment_spec_replicas` — join by `deployment, namespace`.

### O-015 | How do you implement metric-based autoscaling in Kubernetes using KEDA with Prometheus? | Implement | Expert | ⬜
**Hint:** KEDA `ScaledObject` with `prometheus` trigger: queries Prometheus for custom metric, scales HPA target 0→N based on threshold.

### O-016 | Explain Prometheus remote_write quality guarantees — is it at-least-once or exactly-once? | Explain | Expert | ⬜
**Hint:** At-least-once. On failure, Prometheus retries. Deduplication responsibility is on the receiver (Mimir/Thanos/Cortex). WAL: Prometheus buffers unsent samples locally.

### O-017 | Implement a recording rule for request success rate, split by service, for a 30-day SLO report. | Implement | Hard | ⬜
**Hint:** `record: job:http_requests:success_rate5m` with `sum(rate(success[5m]))/sum(rate(total[5m])) by (job)`. For 30d trend in Grafana, query the recording rule.

### O-018 | Write a Prometheus alert that fires if a cron job hasn't run in the last 25 hours (based on job success metric). | Implement | Hard | ⬜
**Hint:** `time() - job_last_success_timestamp > 25*3600`. Where `job_last_success_timestamp` is a gauge updated by the cron job. Alert: CronJobMissed.

### O-019 | How do you test Prometheus alerting rules before deploying? | Implement | Hard | ⬜
**Hint:** `promtool test rules tests/alerting_test.yaml` — mock series + expected firing state at specific timestamps. Unit test alerts without live Prometheus.

### O-020 | Explain Prometheus exemplars and their relationship to distributed tracing. | Explain | Expert | ⬜
**Hint:** Exemplar: sample (timestamp, trace_id, span_id) attached to a histogram bucket. In Grafana: click latency spike → "trace" link → jumps to Jaeger/Tempo trace. Enables metrics-to-traces exploration.

### O-021 | Prometheus is missing 30 seconds of metrics after a pod restart. Why? | Troubleshoot | Medium | ⬜
**Hint:** Scrape interval: if pod restarts between scrapes, that scrape interval is missed. Staleness markers: Prometheus marks series as stale immediately. Use recording rules with `offset` to smooth over gaps.

### O-022 | Write a PromQL query that shows which nodes are most likely to be evicted in the next hour based on memory pressure. | Implement | Expert | ⬜
**Hint:** `container_memory_usage_bytes` / `node_memory_MemTotal_bytes` by node, sorted desc. Overlay: `node_memory_MemAvailable_bytes < 500Mi`. Alert: NodeMemoryPressureImminent.

### O-023 | How do you implement Prometheus operator PodMonitor vs ServiceMonitor — when to use each? | Explain | Medium | ⬜
**Hint:** ServiceMonitor: scrape via Service/Endpoints (most common). PodMonitor: scrape pods directly when no Service (batch jobs, DaemonSets). Both produce the same scrape targets.

### O-024 | Implement a multi-cluster Prometheus federation setup using Prometheus federation endpoint. | Implement | Hard | ⬜
**Hint:** Global Prometheus: `scrape_configs: job_name: federate, metrics_path: /federate, targets: [cluster1-prom:9090], params: match[]: ['{job="important-job"}']`. Federation: pull aggregates only, not all series.

### O-025 | Write a PromQL query to find which namespace is consuming the most memory relative to its limits. | Implement | Hard | ⬜
**Hint:** `sum(container_memory_usage_bytes) by (namespace)` / `sum(kube_pod_container_resource_limits{resource="memory"}) by (namespace)` — topk(5).

---

## Section 2: Grafana, Dashboards, and Tracing (O-026 to O-055)

---

### O-026 | Grafana Dashboard as Code with Grafonnet
**Type:** Implement | **Difficulty:** Hard | ⬜

> Design a standard Grafana dashboard-as-code approach for 50 service dashboards with shared panels.

**What you must cover:**
- Grafonnet (Jsonnet library): define dashboard in Jsonnet, generate JSON, commit to Git
- Dashboard provisioning: Grafana reads from `/etc/grafana/provisioning/dashboards/*.yaml`
- ConfigMap: dashboard JSONs in K8s ConfigMaps — Grafana sidecar (kiwigrid) syncs automatically
- Shared library: common panels (request rate, P99 latency, error rate, pod restarts) in shared Jsonnet
- Service dashboard: extends shared template, adds service-specific panels
- Helm: `grafana.dashboardsConfigMaps` → reference ConfigMaps in values.yaml

---

### O-027 | Grafana Alerting vs Prometheus Alertmanager
**Type:** Design | **Difficulty:** Hard | ⬜

> Your team wants to move alerting from Prometheus Alertmanager to Grafana Alerting. What changes and what stays the same?

**What you must cover:**
- Grafana Alerting: manage alert rules in Grafana UI, stored in Grafana DB (or Git-backed)
- Routing: Grafana contact points (Slack, PagerDuty, OpsGenie) replace Alertmanager receivers
- Multi-datasource: Grafana can alert on Loki, Prometheus, and other sources — Alertmanager can't
- Migration path: export Alertmanager rules → import to Grafana. Runs in parallel temporarily.
- Limitation: Grafana Alerting less mature for complex routing trees vs Alertmanager
- Recommendation: for Prometheus-only, Alertmanager is standard. For multi-datasource, Grafana Alerting is better.

---

### O-028 | Distributed Tracing with OpenTelemetry
**Type:** Design | **Difficulty:** Expert | ⬜

> Design a distributed tracing setup for 20 microservices using OpenTelemetry and Tempo.

**What you must cover:**
- Instrumentation: OpenTelemetry SDK in each service (auto-instrumentation for Python, Java, Node.js)
- Collector: OTel Collector DaemonSet — receives traces from pods, batches/filters, exports to Tempo
- Tempo: trace backend — stores compressed traces in Azure Blob Storage
- Grafana: Tempo data source — search by trace ID, service graph, span metrics
- Context propagation: W3C TraceContext headers (`traceparent`, `tracestate`) across services
- Sampling: head sampling (1% of traces) or tail sampling (Collector: sample 100% of errors, 1% of success)
- Service graph: Grafana Tempo service graph panel shows dependencies and error rates

---

### O-029 | Loki Log Aggregation
**Type:** Design | **Difficulty:** Hard | ⬜

> Design a log aggregation system using Loki for 50 Kubernetes clusters.

**What you must cover:**
- Loki stack: Promtail (per-cluster DaemonSet) → Loki (multi-tenant, object storage backend)
- Promtail: reads `/var/log/pods/`, adds labels (`namespace`, `pod`, `container`) — no log parsing at scrape time
- Loki: stores logs in chunks on Azure Blob Storage, indexes only labels (not log content) — differentiated from ES
- Cardinality: Loki label cardinality must be low — don't add request_id as label
- Log filtering: LogQL `{namespace="prod", app="api"} |= "error" | json | level = "error"`
- Grafana Explore: Loki + time range → correlate with Prometheus metrics via split panel
- Cost: Loki < Elasticsearch by 10-100× because inverted index is just labels, not full text

---

### O-030 | OpenTelemetry Collector Pipeline
**Type:** Implement | **Difficulty:** Expert | ⬜

> Configure an OTel Collector pipeline that: receives traces from apps, enriches with k8s metadata, filters health check spans, and exports to Tempo.

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc: {endpoint: 0.0.0.0:4317}
      http: {endpoint: 0.0.0.0:4318}

processors:
  k8sattributes:    # Enrich with pod/node/namespace metadata
    extract:
      metadata: [k8s.pod.name, k8s.namespace.name, k8s.node.name]
  
  filter/drop-health:   # Drop health check spans
    spans:
      exclude:
        match_type: regexp
        attributes:
          - key: http.url
            value: ".*(health|ready|ping).*"
  
  batch:
    send_batch_size: 10000
    timeout: 1s

exporters:
  otlp/tempo:
    endpoint: tempo.monitoring:4317
    tls: {insecure: true}

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [k8sattributes, filter/drop-health, batch]
      exporters: [otlp/tempo]
```

---

### O-031 to O-055 | Observability Rapid-Fire

---

### O-031 | Grafana dashboard shows "No data" for a metric that exists in Prometheus. Debug steps. | Troubleshoot | Medium | ⬜
**Hint:** Variable misconfigured (label mismatch). Query uses old metric name. Time range too narrow. Grafana data source wrong URL. PromQL returns no data (try in Explore with same query).

### O-032 | Implement Grafana templating variables so one dashboard covers all namespaces dynamically. | Implement | Medium | ⬜
**Hint:** Variable: `label_values(kube_pod_info, namespace)`. Use `$namespace` in queries. Multi-value: `namespace=~"$namespace"` with `|` join for regex.

### O-033 | A microservice has high P99 latency but normal P50. What does this tell you, and how do you investigate? | Troubleshoot | Hard | ⬜
**Hint:** Long tail — some requests are slow. Likely: outlier slow queries, GC pauses, lock contention, cold cache. Investigate: traces (find slow spans), per-path breakdown (some routes slow), dependency calls.

### O-034 | Implement log-based metrics in Loki for error rate calculation without changing application code. | Implement | Hard | ⬜
**Hint:** Loki `MetricsQL` recording rule or Promtail pipeline stages: `metrics:` block converts log lines to counters. Or: LogQL `count_over_time({app="api"} |= "ERROR" [5m])` in Grafana alert.

### O-035 | How does Grafana OnCall work and how does it differ from PagerDuty? | Explain | Medium | ⬜
**Hint:** Grafana OnCall: open-source, integrates with Grafana Alerting. On-call schedules, escalation policies, phone/Slack alerts — same features as PagerDuty. PagerDuty: mature enterprise, better integrations, SLA.

### O-036 | Design a RED method dashboard (Rate, Errors, Duration) for a gRPC service. | Implement | Hard | ⬜
**Hint:** Rate: `sum(rate(grpc_server_started_total[5m])) by (grpc_method)`. Errors: `sum(rate(grpc_server_handled_total{grpc_code!="OK"}[5m])) by (grpc_method)`. Duration: `histogram_quantile(0.99, rate(grpc_server_handling_seconds_bucket[5m]))`.

### O-037 | Implement alert correlation — group related alerts so on-call doesn't get paged 50 times for one failure. | Design | Hard | ⬜
**Hint:** Alertmanager `group_by: [alertname, service]`. Inhibition rules: if NodeDown firing, suppress pod-level alerts for pods on that node. Grafana OnCall: correlation windows.

### O-038 | How do you implement structured logging in Python so logs are parseable by Loki/Logstash? | Implement | Medium | ⬜
**Hint:** `structlog` or `python-json-logger`. Output: `{"timestamp": "...", "level": "error", "service": "api", "message": "...", "trace_id": "..."}`. Loki: `| json` in LogQL extracts fields.

### O-039 | A Prometheus alert fires at 3 AM but self-resolves in 2 minutes every night. How do you suppress overnight noise without ignoring real issues? | Design | Hard | ⬜
**Hint:** Investigate cause first (scheduled job, gc). If known benign: Alertmanager time interval to silence 2-4 AM. Or: increase `for: 10m` — 2-min flap won't fire. Or: adjust alert threshold for that time window.

### O-040 | Implement a Grafana SLO dashboard with error budget burn rate visualization. | Implement | Expert | ⬜
**Hint:** Burn rate: `1 - success_rate / SLO_target` * (window_duration / error_budget_duration). Gauge panel: remaining error budget %. Alert status panel from Alertmanager.

### O-041 | How do you correlate a specific user's request across 10 microservices using distributed tracing? | Explain | Hard | ⬜
**Hint:** Inject trace_id in HTTP headers at edge. Propagate `traceparent` (W3C) through all service calls. In Tempo: search by trace_id → see full trace with all service spans. Add user_id as span attribute for searchability.

### O-042 | Prometheus is returning ALERTS metric but PagerDuty is not receiving them. Debug. | Troubleshoot | Hard | ⬜
**Hint:** Check Alertmanager: `amtool alert query`. Alertmanager silenced? Check silence list. Alert routing: does alert match PagerDuty receiver route? PagerDuty integration key correct? Test: `amtool alert add` manually.

### O-043 | Implement health check probes (liveness, readiness, startup) and explain how they differ. | Implement | Medium | ⬜
**Hint:** Startup: pod is starting up (disable liveness/readiness until startup passes). Readiness: pod can serve traffic (remove from Service endpoints when fails). Liveness: pod is alive (restart if fails). Never make readiness dependent on external DB — it cascades.

### O-044 | Design an observability strategy for a service mesh (Istio). What do you get for free vs what needs instrumentation? | Design | Expert | ⬜
**Hint:** Free (Istio): HTTP request rate, latency, error rate, TCP bytes between services (from Envoy sidecars). Need instrumentation: business metrics, structured logs, trace context propagation (within service code), custom spans.

### O-045 | How do you implement continuous profiling for production Python services? | Implement | Expert | ⬜
**Hint:** Pyroscope or Grafana Phlare: continuous profiling agent, sends flame graphs to backend. CPU, goroutine (if Go), heap allocation profiles. Grafana: profile data source — correlate with metrics and logs.

### O-046 | What is Datadog APM's impact on application performance? How do you control overhead? | Explain | Hard | ⬜
**Hint:** Instrumentation overhead: 1-3% CPU, negligible latency for most services. Trace sampling reduces overhead: head sampling 10% of requests. Datadog Agent: separate process, not in-process overhead.

### O-047 | Write a Datadog monitor that alerts if p95 request latency exceeds 300ms for 3 of the last 5 minutes. | Implement | Hard | ⬜
**Hint:** Datadog monitor: metric `trace.web.request.duration.by.http_status.p95` > 0.3. Alert condition: `require`: at least 3 out of last 5 evaluation windows breach threshold.

### O-048 | How do you implement capacity forecasting using Prometheus data? | Design | Expert | ⬜
**Hint:** `predict_linear(node_disk_free_bytes[1w], 7*24*3600) < 0` — will disk fill in 7 days? Linear regression on historical data. Or: Grafana ML forecasting plugin. Alert: disk exhaustion predicted.

### O-049 | Explain OpenTelemetry semantic conventions for HTTP spans. Why do they matter? | Explain | Hard | ⬜
**Hint:** Standardized attribute names: `http.method`, `http.status_code`, `http.url`, `http.target`. Matter because: backends (Tempo, Jaeger, Datadog) can auto-generate service maps and dependency graphs from standard attributes.

### O-050 | Implement a dead man's switch alert: alert if no data is received from a monitoring system for 5 minutes. | Implement | Hard | ⬜
**Hint:** Prometheus: `absent(up{job="myservice"} == 1)`. Alertmanager: `ALERTS_FOR_STATE` metric. Or: heartbeat alert — if metric `watchdog` stops firing, trigger "MonitoringSilent" alert.

### O-051 | How do you use Loki to debug a production incident after the fact (logs disappeared from pods log stream)? | Troubleshoot | Medium | ⬜
**Hint:** Pod logs ephemeral — Loki persists them. Query: `{namespace="prod", app="api"} |= "error"` at incident time range. Pod restart? Loki has logs from before restart. Filter by container restart count (label).

### O-052 | Design on-call rotation and escalation policy for a 24x7 SLA. | Design | Hard | ⬜
**Hint:** Primary on-call: 1 week rotation. Escalation L1→L2 after 5 min no ack. L2 always senior. Secondary on-call: backup if primary unavailable. Biz hours: team also paged. Postmortem required for all Sev1/Sev2.

### O-053 | Explain the four golden signals and implement each as a Prometheus metric + alert for a gRPC service. | Implement | Expert | ⬜
**Hint:** Latency: `histogram_quantile(0.99, rate(grpc_duration_bucket[5m]))`. Errors: `rate(grpc_handled_total{code!="OK"}[5m]) / rate(grpc_handled_total[5m])`. Throughput: `sum(rate(grpc_handled_total[5m]))`. Saturation: CPU usage, queue depth.

### O-054 | How do you implement dynamic alerting thresholds using anomaly detection in Datadog? | Design | Expert | ⬜
**Hint:** Datadog `anomaly()` function: `anomaly(avg:system.cpu.user{*}, "agile", 2)` — alert when metric deviates 2σ from predicted baseline. Season-aware: accounts for day-of-week/time-of-day patterns.

### O-055 | Implement a Grafana alert that fires only during business hours using time range conditions. | Implement | Medium | ⬜
**Hint:** Grafana Alerting mute timings: schedule to mute outside business hours (nights/weekends). Or: Alertmanager `active_time_intervals`. Not a native PromQL feature.

---

## Section 3: Incident Response and AIOps (O-056 to O-080)

---

### O-056 | SRE Incident Severity Classification
**Type:** Design | **Difficulty:** Hard | ⬜

> Design an incident severity classification system (Sev1 to Sev4) with escalation triggers, communication SLAs, and MTTR targets.

| Severity | Impact | MTTR Target | On-Call Response | Customer Communication |
|---|---|---|---|---|
| SEV1 | Full service outage / data loss | 30 min | Immediate page | Status page in 5 min |
| SEV2 | Major feature broken, >20% users | 2 hours | Page primary | Status page in 15 min |
| SEV3 | Minor feature degraded | 8 hours | Ticket next business day | Internal ticket |
| SEV4 | Cosmetic / UX issue | SLA sprint | Backlog | None |

- SEV1/SEV2: incident commander + scribe + resolver roles declared
- War room: Zoom bridge open, dedicated Slack channel `#incident-YYYY-MM-DD-XXX`
- Postmortem: mandatory for SEV1/SEV2, optional SEV3

---

### O-057 | Runbook Design for Common Incidents
**Type:** Design | **Difficulty:** Hard | ⬜

> Design a runbook template for a high-error-rate incident. What must every runbook include?

**Required sections:**
1. **Alert:** Link to alert definition + Grafana dashboard
2. **Symptom:** What the user/system experiences
3. **Likely causes:** Ordered by frequency (most common first)
4. **Investigation:** Copy-paste commands with descriptions
5. **Mitigation:** Steps to reduce blast radius (feature flag off, scale up)
6. **Root cause analysis:** How to confirm each cause
7. **Fix:** Permanent fix per root cause
8. **Escalation:** When to escalate and who to page
9. **Prevention:** Long-term remediation / toil reduction

---

### O-058 | AIOps — Automated Anomaly Detection
**Type:** Design | **Difficulty:** Expert | ⬜

> Design an AIOps system that automatically detects deployment-correlated anomalies making manual triage unnecessary for 80% of incidents.

**What you must cover:**
- Deployment event: CD pipeline publishes event to Kafka/EventGrid
- Anomaly detector: compares metric distributions before/after deployment (change point detection)
- Correlation: alert fires within 10 min of deploy → auto-correlate "caused by deploy"
- Auto-rollback trigger: if SEV1 anomaly correlated with deploy → auto-trigger rollback
- Tools: Datadog Watchdog, Honeycomb BubbleUp, custom ML (Prophet for seasonality-aware anomaly detection)
- Feedback loop: on-call confirms/rejects auto-correlation to improve model

---

### O-059 to O-080 | Incident and AIOps Rapid-Fire

---

### O-059 | A deployment just happened and P99 latency spiked. Walk through the 5-minute investigation protocol. | Troubleshoot | Hard | ⬜
**Hint:** 1. Verify deploy time vs spike time (correlate). 2. Identify affected service (trace span latency breakdown). 3. Error rate (separate from latency issue?). 4. Dependency: downstream slow or upstream overload? 5. Rollback decision: based on error rate and user impact.

### O-060 | How do you implement a chaos engineering experiment using Chaos Mesh on Kubernetes? | Implement | Expert | ⬜
**Hint:** Chaos Mesh: `NetworkChaos` (add 100ms latency to pod), `PodChaos` (kill random pods), `StressChaos` (CPU burn). Run during business hours with blast radius control. Hypothesize → experiment → verify system behaves as designed.

### O-061 | Implement a canary analysis AnalysisRun in Argo Rollouts that fails deployment if error rate > 1% for 5 minutes. | Implement | Expert | ⬜

### O-062 | Design a logging strategy for GDPR compliance — no PII in logs. | Design | Hard | ⬜
**Hint:** Log schema review: scrub `user_email`, `IP addresses` (hash or drop), `payment info`. Structured logging: centralized sanitizer before shipping to Loki. Log retention policy: 30 days. Audit trail: separate secure log stream for compliance.

### O-063 | How do you monitor Kubernetes cluster health proactively (not just react to alerts)? | Design | Expert | ⬜
**Hint:** Node health: kube-state-metrics, node_exporter. Control plane: API server latency, etcd backup age, cert expiry. Capacity planning: headroom alerts (80% threshold). Upgrade readiness: deprecated API checks.

### O-064 | Implement a synthetic monitoring check for a critical user payment flow. | Implement | Hard | ⬜
**Hint:** Grafana k6 or Blackbox Exporter: HTTP probe hitting `/checkout/complete`. Run every minute from multiple regions (Azure Azure region probes). Alert: probe fails 2 consecutive checks. SLA: measured independently from metrics.

### O-065 | A Kubernetes node is NotReady and pods are being evicted. Walk through the incident response. | Troubleshoot | Hard | ⬜
**Hint:** 1. `kubectl describe node <node>` — conditions. 2. `kubectl get events -n kube-system`. 3. SSH to node: `journalctl -u kubelet --no-pager -n 100`. 4. Disk/memory pressure? Drain: `kubectl drain --ignore-daemonsets`. 5. Replace node (AKS: node pool upgrade/reimage).

### O-066 | How do you measure and report on MTTR improvements over time? | Design | Hard | ⬜
**Hint:** Incident timestamps from PagerDuty API: created at + resolved at = MTTR per incident. Aggregate by week/month. Trend: Grafana or BI dashboard. Correlate with investments: runbooks added, automation deployed.

### O-067 | What is the difference between monitoring and observability? | Explain | Medium | ⬜
**Hint:** Monitoring: known unknowns (you know what to alert on). Observability: unknown unknowns (you can ask new questions from telemetry without deploying new code). Observability = logs + metrics + traces + correlation tools.

### O-068 | Implement cost-based alerting — alert if a service's cloud cost spikes >20% week-over-week. | Implement | Hard | ⬜
**Hint:** Azure Cost Management API → export to storage (or pull via LogicApp). Prometheus: expose as gauge metric. Alert: `week_cost / last_week_cost > 1.2`.

### O-069 | Explain the USE method (Utilization, Saturation, Errors) and implement it for AKS nodes. | Implement | Hard | ⬜
**Hint:** Utilization: `node_cpu_seconds_total` (busy %). Saturation: run queue length `node_load1`. Errors: `node_network_receive_errs_total`. Container: CPU throttling = saturation. Memory evictions = saturation/error.

### O-070 | How do you reduce alert fatigue without missing real incidents? | Design | Expert | ⬜
**Hint:** Audit: which alerts are low-signal (fire but no action taken). Raise thresholds + extend `for:` duration. Delete alerts with no runbook. Implement SLO-based alerting (burn rate) — fewer, higher-quality alerts. Correlation: group related alerts.

### O-071 | Implement distributed log sampling — keep 100% of error logs but only 1% of info logs. | Implement | Expert | ⬜
**Hint:** Promtail pipeline stages: `sampling: rate: 0.01 drop_counter_reason: sampled` with `match: selector: '{level="info"}'`. Or OTel Collector: `filter` processor on severity. Always keep errors/warnings.

### O-072 | A service has high GC overhead visible in traces but not in metrics. How do you instrument to capture GC pauses? | Implement | Hard | ⬜
**Hint:** JVM: `jvm_gc_pause_seconds_sum` from Micrometer. Python: `gc.callbacks`. Go runtime metrics. GC pause > 100ms creates latency spikes visible in P99. Alert: `jvm_gc_pause_seconds > 0.1`.

### O-073 | How do you validate that an observability stack (Prometheus + Loki + Tempo) is working correctly before relying on it for production? | Design | Hard | ⬜
**Hint:** Synthetic load: generate test traces/metrics/logs. Verify end-to-end in Grafana Explore. Backup alerting: uptime checks from external provider (StatusCake). Alert on monitoring itself (watchdog alert, "dead man's switch").

### O-074 | Design a disaster recovery observability setup — your primary monitoring stack went down. | Design | Expert | ⬜
**Hint:** Monitoring stack in separate AKS cluster from production. Multi-region: replicate to secondary region. External: Datadog or New Relic as backup (different vendor). Status page: separate from internal monitoring.

### O-075 | Implement a Grafana OnCall integration that creates a Jira ticket for every SEV2+ incident automatically. | Implement | Hard | ⬜
**Hint:** Grafana OnCall webhook: on alert fire → HTTP POST to Jira REST API (`/rest/api/2/issue`). Template: issue title from alert name, description from annotations. Auto-close Jira ticket when alert resolves.

### O-076 | How do you track error budget consumption in real time and automate feature freeze when budget is exhausted? | Design | Expert | ⬜
**Hint:** Error budget remaining: Grafana gauge (live). When < 20%: Prometheus alert → GitHub Actions webhook → set `FEATURE_FREEZE=true` flag in Terraform or config. Restart after next period (30 days).

### O-077 | Implement a k8s audit log monitoring pipeline for detecting privilege escalation. | Implement | Expert | ⬜
**Hint:** AKS audit logs → Azure Monitor → Log Analytics. Alert on: `verb=bind`, `verb=escalate`, `verb=impersonate`. Kubernetes audit policy: log `RequestResponse` level for sensitive resources (secrets, clusterroles).

### O-078 | How do you do a chaos game day — plan and execute it safely? | Design | Expert | ⬜
**Hint:** Define hypothesis first: "if payment service restarts, checkout should degrade gracefully (no data loss, 5s MTTR)". Blast radius control: feature flag off, test in staging first, on-call on standby. Execute: inject failure → observe. Record: did system behave as designed?

### O-079 | A service's Prometheus metrics show normal but users are reporting errors. Explain the gap and debug. | Troubleshoot | Hard | ⬜
**Hint:** Synthetic vs real user monitoring: metrics are averages, users see outliers. Check: RUM (Real User Monitoring) in browser. CDN errors (not instrumented). Some user segments (geography, device) not in metrics labels. Check logs for correlating errors.

### O-080 | Design an observability platform that can onboard a new microservice in < 1 hour with zero manual work. | Design | Expert | ⬜
**Hint:** Auto-instrumentation: OTel init container or admission webhook injects OTel Java/Python agent. ServiceMonitor: created by Helm chart template (standard annotation). Grafana: auto-provisioned dashboard from service template. Alert: standard SLO alerts applied from policy. On-call: added to escalation policy via IDP API.
