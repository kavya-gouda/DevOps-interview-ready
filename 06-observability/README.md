# Observability (Prometheus, Grafana, OpenTelemetry, Datadog)

## Folder Structure

```
06-observability/
├── prometheus-grafana/   # PromQL, alerting rules, recording rules
├── distributed-tracing/  # OpenTelemetry, Tempo, Jaeger
├── log-aggregation/      # Loki, ELK, Fluent Bit
└── slo-sli-sla/          # SLO definitions, error budgets, policies
```

## Coverage Checklist

### Prometheus
- [ ] Architecture: scrape model, exporters, Pushgateway (when to use / avoid)
- [ ] Service discovery: Kubernetes SD, relabeling
- [ ] TSDB: block storage, compaction, retention
- [ ] Cardinality: high-cardinality labels, impact on performance
- [ ] Recording rules: precompute expensive queries
- [ ] Alerting rules: `for`, `labels`, `annotations`, routing to AlertManager
- [ ] AlertManager: routing tree, inhibition, silences, receivers (PagerDuty, Slack)
- [ ] Remote write: Thanos, Mimir, Cortex for long-term storage

### PromQL
- [ ] Selectors, label matchers
- [ ] Range vectors vs instant vectors
- [ ] Key functions: `rate()`, `irate()`, `increase()`, `histogram_quantile()`
- [ ] Aggregation: `sum by`, `avg by`, `topk`, `count`
- [ ] Joins: `on`, `ignoring`, `group_left`, `group_right`
- [ ] Common patterns: error rate, p99 latency, availability SLO burn rate

### Grafana
- [ ] Data sources: Prometheus, Loki, Tempo, Azure Monitor
- [ ] Dashboard best practices: USE method, RED method, SLO dashboards
- [ ] Alerting in Grafana: Unified Alerting, contact points, notification policies
- [ ] Tempo integration: trace-to-logs, trace-to-metrics correlations

### Distributed Tracing
- [ ] OpenTelemetry: SDK, instrumentation (auto vs manual), collector
- [ ] Collector: pipelines, exporters, processors, sampling
- [ ] Sampling strategies: head-based, tail-based, adaptive
- [ ] Propagation: W3C TraceContext, B3
- [ ] Jaeger vs Tempo: storage, query patterns

### Log Aggregation
- [ ] Loki architecture: ingesters, distributors, queriers, compactor
- [ ] LogQL: log stream selectors, filter expressions, metric queries
- [ ] Fluent Bit: inputs, filters, outputs, buffering
- [ ] ELK: Logstash pipeline, Elasticsearch index templates, Kibana
- [ ] Structured logging: JSON logs, correlation IDs, trace ID injection

### SLO / SLI / SLA
- [ ] SLI types: availability, latency, error rate, throughput
- [ ] SLO definition: window (rolling vs calendar), target
- [ ] Error budget: calculation, burn rate alerts (multi-window, multi-burn)
- [ ] Error budget policy: freeze deployments, accelerate reliability work
- [ ] Alerting on SLO burn rate (Google SRE approach)

## Key PromQL Queries to Know

```promql
# HTTP error rate (5xx)
sum(rate(http_requests_total{status=~"5.."}[5m])) /
sum(rate(http_requests_total[5m]))

# p99 latency
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))

# Availability (1 - error rate)
1 - (sum(rate(http_requests_total{status=~"5.."}[30d])) /
     sum(rate(http_requests_total[30d])))
```

## Resources

- [Prometheus Documentation](https://prometheus.io/docs/)
- [Google SRE Workbook — SLOs chapter](https://sre.google/workbook/alerting-on-slos/)
- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
- [Grafana Loki Documentation](https://grafana.com/docs/loki/)
