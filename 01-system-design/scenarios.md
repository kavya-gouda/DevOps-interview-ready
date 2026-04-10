# System Design & Architecture — 100 Scenarios

> **ID prefix:** S- | Types: Design, Architecture, Explain | Difficulty: Medium → Expert

---

## Section 1: Distributed Systems Fundamentals (S-001 to S-020)

---

### S-001 | CAP Theorem Applied to Your Platform
**Type:** Explain | **Difficulty:** Medium | ⬜

> Your team is building a real-time inventory system for an e-commerce platform. An interviewer asks: "Given CAP theorem, what trade-offs did you make, and why?"

**What you must cover:**
- CAP theorem definitions — and why you can only pick 2 under network partition
- CP vs AP choice for inventory (stock correctness vs availability)
- How eventual consistency is handled at the application layer
- Real example: DynamoDB (AP), etcd (CP)

---

### S-002 | Designing for 99.99% Availability
**Type:** Design | **Difficulty:** Medium | ⬜

> Design a platform that guarantees 99.99% uptime for a payment processing service. What does 99.99% actually mean in downtime minutes per year, and how do you architect for it?

**What you must cover:**
- 99.99% = 52.6 min/year downtime budget
- Redundancy at every layer (compute, network, storage, DNS)
- Multi-AZ vs multi-region: when each is sufficient
- Health checks, failover, and RTO/RPO definitions
- Chaos engineering to validate assumptions

---

### S-003 | Leader Election in Distributed Systems
**Type:** Explain | **Difficulty:** Hard | ⬜

> Explain how etcd implements leader election and why this matters when designing a control plane component that must have exactly-one-active semantics.

**What you must cover:**
- Raft consensus algorithm: leader election, log replication, quorum
- Quorum = (n/2)+1 — why odd number of nodes
- etcd lease-based leader election for Kubernetes controllers
- What happens during a network partition (split-brain prevention)
- `client-go` leader election API usage

---

### S-004 | Idempotency in Distributed Systems
**Type:** Explain | **Difficulty:** Hard | ⬜

> You're designing a webhook processing system. Messages can be delivered more than once. How do you ensure operations are idempotent end-to-end?

**What you must cover:**
- Idempotency keys: client-generated UUID per request
- Deduplication store (Redis SET NX with TTL, or DB unique constraint)
- At-least-once vs exactly-once delivery trade-offs
- Idempotency in Kubernetes controllers (reconciliation is inherently idempotent)
- Database: upsert patterns, conditional writes

---

### S-005 | Consistent Hashing for Distributed Caching
**Type:** Explain | **Difficulty:** Hard | ⬜

> Your Redis cluster is scaling from 3 to 5 nodes. How does consistent hashing prevent a thundering-herd cache miss? How does this work in Redis Cluster?

**What you must cover:**
- Consistent hashing ring: virtual nodes, minimal key redistribution
- Redis Cluster: 16384 hash slots, slot assignment during resharding
- Cache stampede: thundering herd prevention (mutex/semaphore, probabilistic early expiration)
- Comparison to modulo hashing (why it fails on resize)

---

### S-006 | Event Sourcing vs State Storage
**Type:** Explain | **Difficulty:** Hard | ⬜

> When would you choose event sourcing over a traditional state-based database? Walk through a real DevOps tooling use case.

**What you must cover:**
- Event sourcing: append-only event log, state = replay of events
- Advantages: full audit trail, temporal queries, easy replay
- Disadvantages: complexity, eventual consistency, event schema evolution
- Real use case: Kubernetes uses event sourcing (etcd watch API, controller reconciliation)
- When not to use it: simple CRUD services, high-throughput writes without audit needs

---

### S-007 | CQRS Pattern in Practice
**Type:** Design | **Difficulty:** Hard | ⬜

> Design a deployment tracking system that handles 10,000 deploys/day with a read-heavy dashboard (100:1 read/write ratio). Apply CQRS.

**What you must cover:**
- Command side: write path (deployment events → Kafka → event store)
- Query side: read-optimized projections (PostgreSQL read replica, Elasticsearch for search)
- Eventual consistency between write and read models
- How to handle inconsistency window in the UI ("your deploy is in progress")
- Trade-off: complexity vs performance

---

### S-008 | Saga Pattern for Distributed Transactions
**Type:** Design | **Difficulty:** Expert | ⬜

> Design a provisioning workflow: create AKS cluster → configure DNS → deploy base workloads → notify team. Any step can fail. How do you ensure consistency without a distributed transaction?

**What you must cover:**
- Saga pattern: choreography vs orchestration
- Compensating transactions per step (delete cluster if DNS fails)
- Orchestration approach: Step Functions / Temporal / Argo Workflows
- Failure handling: retry with backoff, dead-letter queue for terminal failures
- Idempotency of each step

---

### S-009 | Circuit Breaker Pattern
**Type:** Explain | **Difficulty:** Medium | ⬜

> A downstream dependency (secret store) becomes slow. How does a circuit breaker prevent cascading failure, and how would you implement this in a Kubernetes-native service?

**What you must cover:**
- Circuit breaker states: Closed → Open → Half-Open
- Trip conditions: error rate threshold + sliding window
- Fallback behavior: cached response, default value, fail-fast
- Implementation options: Envoy circuit breaking (Istio DestinationRule), Resilience4j, custom middleware
- Monitoring: circuit state as a Prometheus metric

---

### S-010 | Backpressure and Rate Limiting
**Type:** Design | **Difficulty:** Hard | ⬜

> Your CI system can process 200 builds/hour. During a release period, 2000 build requests arrive. Design a backpressure mechanism that prevents the system from falling over.

**What you must cover:**
- Rate limiting algorithms: token bucket, leaky bucket, sliding window counter
- Queue with bounded length — reject beyond capacity (fast fail > slow failure)
- Priority queues: production deployments over feature branch builds
- Metrics and alerting on queue depth
- Kubernetes: resource quotas, admission webhooks for throttling

---

### S-011 | Write-Ahead Logging and Crash Recovery
**Type:** Explain | **Difficulty:** Hard | ⬜

> Explain how write-ahead logging (WAL) works in PostgreSQL and how it guarantees durability. How does etcd use a similar mechanism?

**What you must cover:**
- WAL: write to log before applying to data files (fsync)
- Crash recovery: replay WAL from last checkpoint
- etcd: WAL + snapshots, `--auto-compaction-retention`
- Implications for backup strategy (WAL archiving for PITR)
- Relation to Kubernetes: etcd data loss = cluster metadata loss

---

### S-012 | Read Replicas and Lag
**Type:** Design | **Difficulty:** Medium | ⬜

> You've added a read replica to your PostgreSQL database to handle reporting queries. A developer complains that "just-written" data isn't showing up in reports. Diagnose and design a fix.

**What you must cover:**
- Replication lag: async vs sync replication, measurement query
- Application-level read routing: write to primary, read from replica after delay
- Session consistency: read-your-writes achieved by routing reads to primary for same session
- Monitoring: `pg_stat_replication`, `max_lag` alert
- Trade-off: sync replication kills write throughput

---

### S-013 | Database Sharding Strategy
**Type:** Design | **Difficulty:** Expert | ⬜

> Your deployment metadata table has 10 billion rows. Single-node PostgreSQL can't handle query load. Design a sharding strategy.

**What you must cover:**
- Shard key selection (tenant ID, pipeline ID) — avoid hot partitions
- Hash sharding vs range sharding trade-offs
- Cross-shard queries: avoid or use scatter-gather
- Schema migrations across shards: coordination problem
- Alternatives to sharding: TimescaleDB for time-series, Citus for horizontal Postgres
- Operational complexity: rebalancing shards, monitoring

---

### S-014 | Multi-Region Active-Active vs Active-Passive
**Type:** Design | **Difficulty:** Expert | ⬜

> A fintech company wants zero-downtime DR. Design a multi-region strategy. Choose between active-active and active-passive and justify your choice.

**What you must cover:**
- Active-passive: simpler, RPO/RTO depends on replication lag + failover time
- Active-active: write conflicts (which region wins?), latency trade-offs, highest complexity
- Data consistency in active-active: conflict-free data types (CRDTs), last-write-wins, or avoid shared writes
- Global traffic routing: Azure Front Door, CloudFlare, Route 53 health-check-based routing
- State: stateless services easily active-active; stateful (DB) hardest
- Cost: active-active ≈ 2× infrastructure cost

---

### S-015 | Thundering Herd on Startup
**Type:** Design | **Difficulty:** Hard | ⬜

> After a cluster-wide restart, 5000 pods simultaneously try to pull secrets from Vault. Vault is overwhelmed and fails. Design a solution.

**What you must cover:**
- Jitter: random startup delay to spread load (`initialDelaySeconds` + random jitter)
- Circuit breaker on Vault client (stop hammering on error)
- Vault response caching: agent injector cache, ESO cache TTL
- Vault transit scalability: batch operations, performance standby nodes
- Pre-warming: secrets sync before pod start (ESO `refreshInterval`)

---

### S-016 | Gossip Protocol and Service Discovery
**Type:** Explain | **Difficulty:** Hard | ⬜

> How does Consul use gossip protocol for service discovery? Compare this to Kubernetes DNS-based service discovery. When would you choose one over the other?

**What you must cover:**
- Gossip: each node talks to k random peers, O(log n) propagation
- Consul: agent on each node, catalog + health checks + KV store
- Kubernetes: kube-dns/CoreDNS, service → ClusterIP → iptables/IPVS
- DNS TTL caching problem in K8s (ndots, negative cache)
- When Consul wins: multi-datacenter, non-K8s workloads, advanced health checks

---

### S-017 | Distributed Locking
**Type:** Design | **Difficulty:** Hard | ⬜

> Two deployment pipelines must not run simultaneously against the same environment. Design a distributed lock using Redis. What are the failure modes?

**What you must cover:**
- Redlock algorithm (Martin Kleppmann critique vs Antirez defense)
- Simple Redis `SET NX PX` with expiry: single-node implementation + failure modes
- Lock expiry: what if the lock holder crashes? TTL + watchdog refresh
- Fencing tokens: monotonically increasing token to prevent stale lock use
- etcd-based distributed locking via lease (more correct for safety-critical use)

---

### S-018 | Change Data Capture (CDC)
**Type:** Design | **Difficulty:** Hard | ⬜

> You need to sync changes from your PostgreSQL deployments table to Elasticsearch for search, in near-real-time. Design a CDC pipeline.

**What you must cover:**
- Debezium: reads PostgreSQL WAL (logical replication), emits change events to Kafka
- Kafka Elasticsearch Connector: consumes events, indexes into ES
- Schema registry: Avro/Protobuf for schema evolution safety
- At-least-once delivery + idempotent writes to ES (`_id` from PK)
- Bootstrap problem: initial snapshot + then CDC stream
- Lag monitoring: Kafka consumer group lag

---

### S-019 | Bulkhead Pattern
**Type:** Design | **Difficulty:** Hard | ⬜

> Your CI system has a shared pool of build agents. Heavy nightly batch builds starve daytime PR builds. Apply the bulkhead pattern.

**What you must cover:**
- Bulkhead: isolated resource pools per workload type (thread pools, agent pools)
- Kubernetes: separate node pools with taints/tolerations + resource quotas per namespace
- GitHub Actions: separate runner groups per workflow classification
- Failure isolation: batch agents failing doesn't impact PR agents
- Cost trade-off: reserved vs burst capacity

---

### S-020 | Two-Phase Commit vs Saga
**Type:** Explain | **Difficulty:** Expert | ⬜

> Explain 2PC and why it's avoided in microservices. What does the Saga pattern solve that 2PC doesn't?

**What you must cover:**
- 2PC: coordinator + participants, prepare phase → commit phase
- 2PC failure modes: coordinator crashes after prepare (blocking, uncertainty period)
- 2PC limitations: blocking protocol, coordinator SPOF, all participants must be available
- Saga: no distributed lock, compensating transactions, async, non-blocking
- When 2PC is still used: databases (XA transactions), same-machine transactions

---

## Section 2: Platform & Infrastructure Design (S-021 to S-060)

---

### S-021 | CI/CD Platform at Scale — 5000 Microservices
**Type:** Design | **Difficulty:** Expert | ⬜

> Design a CI/CD platform for a company with 5000 microservices, 500 engineers, and a requirement that any service can deploy to production in under 10 minutes.

**What you must cover:**
- Build: distributed caching (Bazel remote exec, layer caching in registry)
- Test: parallelism, test sharding, flaky test quarantine
- Runner infrastructure: Kubernetes-native runners (ARC), autoscaling
- Deployment: GitOps with ArgoCD ApplicationSets, progressive delivery
- Artifact management: OCI registry with promotion gates (dev → staging → prod)
- Governance: required approvals, CODEOWNERS, environment protection rules
- Observability: DORA metrics (deployment frequency, lead time, MTTR, change failure rate)

---

### S-022 | Internal Developer Platform (IDP)
**Type:** Design | **Difficulty:** Expert | ⬜

> Design an Internal Developer Platform that allows engineers to self-provision a new microservice with CI/CD, Kubernetes namespace, monitoring, and secrets in under 5 minutes via a portal.

**What you must cover:**
- Backstage (or similar): software catalog, scaffolder templates, TechDocs
- Scaffolder: creates Git repo from template, triggers Terraform for namespace/RBAC, registers in ArgoCD
- Golden path: standard deployment template (Helm chart or Kustomize)
- Self-service observability: auto-provision Grafana dashboard + AlertManager rules from template
- Secrets: auto-create Vault path + ESO ExternalSecret from template
- Guard rails: OPA admission policies, required labels, mandatory security scanning

---

### S-023 | Multi-Tenant Kubernetes Platform
**Type:** Design | **Difficulty:** Expert | ⬜

> Design a multi-tenant Kubernetes platform for 50 product teams, each needing isolation of compute, network, and blast radius.

**What you must cover:**
- Namespace-per-team with resource quotas and LimitRanges
- Network isolation: default-deny NetworkPolicy per namespace
- RBAC: team-scoped roles, no cluster-admin for tenants
- Admission control: OPA/Kyverno — enforce labels, image registries, security context
- Cost allocation: Kubecost or OpenCost labels mapped to team
- Node isolation: dedicated node pools with taints (optional, for strict isolation)
- Cluster-per-team vs namespace-per-team: trade-offs

---

### S-024 | Disaster Recovery Plan for Kubernetes Cluster
**Type:** Design | **Difficulty:** Expert | ⬜

> Your AKS cluster is the primary platform for production. Design a DR plan with RTO < 1 hour and RPO < 15 minutes.

**What you must cover:**
- etcd backup: Velero (workload state) + Azure Backup for managed cluster
- GitOps as source of truth: cluster state in Git, re-apply on new cluster
- Persistent volume backups: Velero with Azure Disk snapshot
- Cross-region cluster: warm standby with ArgoCD managing both
- DNS failover: Azure Front Door or Traffic Manager health probes
- Runbook: step-by-step failover procedure, tested quarterly
- RTO/RPO measurement: time from detection to traffic restored

---

### S-025 | Zero-Downtime Database Migration
**Type:** Design | **Difficulty:** Expert | ⬜

> You need to add a NOT NULL column to a table with 500 million rows in PostgreSQL. The service is live and cannot tolerate downtime. Design the migration strategy.

**What you must cover:**
- Expand/Contract pattern (parallel change):
  1. Add column as NULLable with default
  2. Backfill in small batches (avoid long lock)
  3. Update application to write to both old + new column
  4. Verify backfill complete
  5. Add NOT NULL constraint (fast in modern Postgres with default set)
  6. Remove old column in later migration
- `pg_repack` for live table rewrites
- `lock_timeout` and `statement_timeout` in migration scripts
- Blue/green database: dual-write + schema compatibility window

---

### S-026 | Observability Platform Design
**Type:** Design | **Difficulty:** Expert | ⬜

> Design an observability platform for 200 microservices with metrics, logs, and traces — from scratch. Choose your stack and justify it.

**What you must cover:**
- Metrics: Prometheus + Thanos/Mimir for long-term storage, Grafana dashboards
- Logs: Fluent Bit (DaemonSet) → Loki, or Elasticsearch for full-text search
- Traces: OpenTelemetry SDK (auto-instrumentation) → OTel Collector → Tempo
- Correlation: trace-to-logs, trace-to-metrics (exemplars in Prometheus)
- Cardinality budget: label design, metric naming conventions
- Cost: storage tiering, retention policies (7d hot, 30d warm, 90d cold)
- Alerting: Grafana Unified Alerting → PagerDuty routing

---

### S-027 | Secrets Management Architecture
**Type:** Design | **Difficulty:** Hard | ⬜

> Design a secrets management architecture for 100 microservices on Kubernetes. No secrets should exist in Git, namespaced K8s Secrets should be auto-synced, and secrets must rotate automatically.

**What you must cover:**
- Vault as source of truth: dynamic secrets (DB, cloud), PKI for TLS
- External Secrets Operator: ExternalSecret CR → pulls from Vault via K8s auth
- Auto-rotation: Vault dynamic secrets TTL + ESO `refreshInterval`
- Secret zero problem: K8s service account token federated to Vault (no bootstrap secret)
- Emergency break-glass: manual Vault token with alerts on use
- Audit: Vault audit log → SIEM

---

### S-028 | Feature Flag System Design
**Type:** Design | **Difficulty:** Hard | ⬜

> Design a feature flag system used by 300 services, supporting user-level targeting, gradual rollouts, and kill switches.

**What you must cover:**
- Flag evaluation: server-side (call to flag service) vs client-side SDK (local evaluation from downloaded ruleset)
- Targeting: user attributes (user ID, org, plan, geo), percentage rollout
- Kill switch: immediate flag flip propagated in <1 second (streaming or polling with short TTL)
- SDK design: circuit breaker around flag service, safe default on failure
- Auditability: flag change log with who changed what and when
- OpenFeature standard: vendor-neutral SDK interface
- Self-hosted options: flagd (CNCF), Unleash, Flipt; managed: LaunchDarkly

---

### S-029 | Platform Migration — Monolith to Microservices
**Type:** Design | **Difficulty:** Expert | ⬜

> An existing Rails monolith processes 10M requests/day. You're tasked with migrating it to microservices on Kubernetes without downtime. What's your strategy?

**What you must cover:**
- Strangler Fig pattern: new services behind feature flags, route traffic gradually
- Domain decomposition: identify bounded contexts (DDD), start with least-coupled domain
- Database: shared DB is an anti-pattern — plan for eventual DB decomposition per service
- Traffic: API gateway or service mesh to route to old vs new based on feature flag
- Observability: distributed tracing from day 1 to understand call chains
- Risk mitigation: shadow mode, canary, automatic rollback
- Timeline: expect 12–24 months for a real monolith

---

### S-030 | High-Throughput Messaging System
**Type:** Design | **Difficulty:** Expert | ⬜

> Design a deployment event pipeline that ingests 1 million deployment events/day, enriches them with metadata, and powers a real-time dashboard and a 90-day historical analytics feature.

**What you must cover:**
- Ingestion: Kafka — partitioned by service, 3-day retention
- Enrichment: stream processing (Kafka Streams or Flink) — join with CMDB/metadata
- Real-time: enriched events → in-memory store (Redis) → WebSocket dashboard
- Historical: Kafka → object storage (Parquet/Iceberg on Azure Data Lake) → Athena/Synapse for analytics
- Exactly-once semantics for enrichment: Kafka transactions or idempotent consumer
- Backpressure: downstream consumer lag monitoring, auto-scaling consumers

---

### S-031 | API Gateway Design
**Type:** Design | **Difficulty:** Hard | ⬜

> Design an API gateway for 150 microservices. It must handle auth, rate limiting, routing, observability, and circuit breaking — at 50,000 RPS.

**What you must cover:**
- Gateway options: Kong, NGINX+Lua, Envoy-based (Ambassador/Emissary), cloud-native (Azure APIM)
- Auth: JWT validation at gateway (public key cache), OAuth2 token introspection for opaque tokens
- Rate limiting: per-client sliding window, Redis-backed counter across gateway instances
- Routing: path-based + header-based to services, canary routing by weight
- Observability: access logs, latency by route, error rate per upstream service
- Circuit breaker: Envoy outlier detection, 503 on open circuit
- TLS termination at gateway, mTLS upstream

---

### S-032 | Designing for Idempotent Deployments
**Type:** Design | **Difficulty:** Hard | ⬜

> Your CD pipeline deploys a Helm chart. If the pipeline is re-run identically, the deployment must produce the same result and not cause restarts. How do you design for idempotency?

**What you must cover:**
- Helm: same chart + same values = idempotent (unless image tag is `latest` — fix with digest)
- `helm upgrade --install` for both first install and upgrade
- Immutable image tags: `sha256:` digest instead of `latest`
- `kubectl apply` vs `kubectl create`: apply is idempotent
- Admission webhooks with defaulting: ensure webhook defaults are deterministic
- ArgoCD self-heal: drift correction without manual trigger
- Canary: idempotent traffic weight updates via Argo Rollouts

---

### S-033 | FinOps Platform Design
**Type:** Design | **Difficulty:** Hard | ⬜

> Design a cost visibility and governance platform for 50 product teams using Kubernetes + Azure. Teams should see their costs daily, receive alerts on anomalies, and have automated enforcement of tagging.

**What you must cover:**
- Cost data: OpenCost/Kubecost for K8s cost allocation by namespace/label
- Azure Cost Management API: pull cost by tag, subscription, resource group
- Normalization: unify K8s allocation + Azure billing into a single model
- Dashboard: Grafana or internal portal — daily cost per team, trend, top spenders
- Anomaly detection: percentage increase over rolling average, Slack/PagerDuty alert
- Enforcement: Azure Policy (deny resources without mandatory tags) + OPA Gatekeeper (deny pods without cost labels)
- Chargeback vs showback: showback (informational) first, chargeback (billing) as maturity grows

---

### S-034 | Service Mesh Adoption Strategy
**Type:** Design | **Difficulty:** Expert | ⬜

> Your platform has 200 services. You want to adopt Istio for mTLS, traffic management, and observability. Design the adoption strategy without breaking running services.

**What you must cover:**
- Phase 1: install Istio in permissive mode (no mTLS enforcement), sidecar injection opt-in by namespace label
- Phase 2: enable strict mTLS per namespace progressively, test each before proceeding
- Phase 3: traffic management — migrate Ingress rules to VirtualService/Gateway
- Observability: Kiali for mesh topology, metrics from Envoy sidecars
- Risk: sidecar adds ~20ms latency + memory overhead per pod
- Escape hatch: `sidecar.istio.io/inject: "false"` for services that can't tolerate overhead
- Ambient mesh consideration: Istio 1.21+ ambient mode eliminates sidecar overhead

---

### S-035 | Platform SLO Design
**Type:** Design | **Difficulty:** Hard | ⬜

> Define SLOs for a Kubernetes platform team that manages the infra used by 100 product teams. What SLIs do you measure, and how do you handle error budget burn?

**What you must cover:**
- Platform SLIs: cluster API availability, pod scheduling latency, PVC provisioning time, DNS resolution success rate
- Target SLOs: API availability 99.95%, pod schedule P99 < 2s, PVC provision < 60s
- Measurement: Prometheus blackbox exporter + custom probes, not just K8s internal metrics
- Error budget: 0.05% downtime/month = ~22 min — policy when budget is 50% burned
- Burn rate alerting: multi-window (1h + 6h) to catch both fast burns and slow burns
- Customer-facing impact: link platform SLO breach to product team SLO impact

---

### S-036 | GitOps Architecture for 100 Clusters
**Type:** Design | **Difficulty:** Expert | ⬜

> You're managing 100 Kubernetes clusters across dev, staging, and production environments across multiple regions. Design a GitOps architecture that scales.

**What you must cover:**
- Hub-and-spoke ArgoCD: management cluster runs ArgoCD, targets all 100 child clusters
- ApplicationSet: generate Application CRs from cluster inventory (git generator, cluster generator)
- Repository structure: monorepo (all clusters) vs multirepo (per cluster) — trade-offs
- Promotion: overlays (Kustomize) per environment, PRs trigger promotion
- Secrets: ESO on each cluster pulls from central Vault
- Cluster bootstrap: cluster API or Terraform provisions cluster → ArgoCD registers cluster → app sync begins
- Drift detection: ArgoCD self-heal enabled in prod, PR-only promotions in staging

---

### S-037 | Designing a Rollback Strategy
**Type:** Design | **Difficulty:** Hard | ⬜

> A deployment to production has caused a 10% error rate increase. Design a rollback strategy that gets to zero errors within 5 minutes.

**What you must cover:**
- Automated rollback trigger: Argo Rollouts analysis template — `successCondition: result[0] < 0.05` on error rate
- Canary: only 10% of traffic routed to new version — rollback affects only those users
- Rollback mechanism: `argocd app rollback`, `kubectl rollout undo`, Helm `rollback`
- Database: schema must be backward-compatible (no breaking migrations before traffic switch)
- Feature flags: instant kill switch for new feature without redeployment
- Communication: auto-Slack message on rollback trigger with runbook link

---

### S-038 | Designing for Blast Radius Reduction
**Type:** Design | **Difficulty:** Hard | ⬜

> A change in one service brought down the entire platform. How would you redesign the platform to minimize blast radius for any single failure?

**What you must cover:**
- Bulkhead: separate thread pools, agent pools, node pools per service class
- Cell-based architecture: independent cells (each with its own DB, compute, queue), deploy to one cell first
- Deployment rings: ring 0 (canary cell) → ring 1 (5%) → ring 2 (25%) → ring 3 (100%)
- Strangler pattern: isolate new changes behind abstraction
- Dependency inversion: services depend on abstractions, not concrete implementations
- Circuit breaker + timeout on every downstream call
- NetworkPolicy: restrict cross-namespace traffic to what's explicitly needed

---

### S-039 | High-Cardinality Metrics Architecture
**Type:** Design | **Difficulty:** Expert | ⬜

> You want per-user latency metrics for 1 million users. Prometheus will explode with cardinality. Design a solution.

**What you must cover:**
- Why not `user_id` as a label: each unique value = a new time series, OOM on Prometheus
- Alternative: histogram by quantile (p50, p95, p99) without user dimension in Prometheus
- Per-user analytics: use Elasticsearch or ClickHouse for high-cardinality queries
- Sampling: trace a subset of users, analyze via distributed tracing (Tempo)
- Aggregation at ingestion: Prometheus recording rules to pre-aggregate before storage
- VictoriaMetrics: handles higher cardinality than Prometheus but still has limits

---

### S-040 | Stateful Workloads on Kubernetes
**Type:** Design | **Difficulty:** Expert | ⬜

> Your team wants to run PostgreSQL on Kubernetes. Walk through the design — storage, HA, backup, and whether you'd actually recommend it.

**What you must cover:**
- When NOT to run DB on K8s: if managed service (Azure Database, RDS) is available — strongly prefer it
- When it makes sense: cost optimization, hybrid environments, dev environments
- Storage: local SSD with `WaitForFirstConsumer` binding, `volumeClaimTemplates` in StatefulSet
- HA: CloudNativePG operator (preferred), Patroni, or Crunchy PostgreSQL operator
- Anti-affinity: pods on different nodes, different AZs
- Backup: WAL-G to Azure Blob, point-in-time restore
- Upgrade: rolling upgrades via operator managed lifecycle

---

## Section 3: Platform Engineering & SRE Design (S-041 to S-070)

---

### S-041 | SRE Team Structure and Toil Reduction
**Type:** Design | **Difficulty:** Hard | ⬜

> You're hired as the first SRE at a company with 50 engineers and no reliability culture. What do you do in your first 90 days? What systems do you put in place?

**What you must cover:**
- Day 1–30: observe, don't change — understand toil, pain points, current on-call burden
- Establish SLIs/SLOs for top 3 revenue-critical services
- Set up error budget dashboards (Grafana) — make reliability visible
- Toil audit: quantify % of on-call time spent on toil
- Alert audit: disable/tune noisy alerts — reduce alert fatigue
- Day 31–60: automate top 3 toil items, establish incident review process, blameless post-mortems
- Day 61–90: SLO-based on-call rotation, runbook library, capacity planning baseline

---

### S-042 | Incident Management System Design
**Type:** Design | **Difficulty:** Hard | ⬜

> Design an incident management system that covers on-call alerting, incident declaration, real-time communication, and post-mortem tracking for a 50-engineer company.

**What you must cover:**
- On-call: PagerDuty/OpsGenie — escalation policies, rotation schedules
- Alert routing: AlertManager → PagerDuty by severity (P1 pages immediately, P3 is Slack only)
- Incident channel: auto-create Slack channel on P1/P2, invite on-call + stakeholders
- Status page: Statuspage.io or self-hosted — auto-update on incident declare/resolve
- Post-mortem: blameless template in GitHub, mandatory for P1/P2, 5-day SLA for completion
- Runbook links: every alert must link to a runbook (enforce via alerting rule lint)

---

### S-043 | Platform API Design — Control Plane
**Type:** Design | **Difficulty:** Expert | ⬜

> Design a control plane API that allows product teams to provision, update, and delete cloud resources (AKS namespaces, storage, queues) declaratively via a GitOps workflow.

**What you must cover:**
- Kubernetes as the control plane: Custom Resource Definitions (CRDs) per resource type
- Controllers: watch CRs, reconcile against Azure (Crossplane or custom controller)
- GitOps: teams submit PR with CRD manifest, ArgoCD applies, controller reconciles
- Admission webhook: validate CR spec, enforce naming, quota, allowed regions
- Status conditions: controller updates `.status.conditions` — teams see progress/errors in `kubectl describe`
- RBAC: teams can only manage their own CRs (namespace-scoped)
- Audit: K8s audit log captures who created/modified/deleted each CR

---

### S-044 | Capacity Planning for Kubernetes
**Type:** Design | **Difficulty:** Hard | ⬜

> Your AKS cluster is at 80% CPU capacity. Design a capacity planning process that ensures you never hit resource exhaustion in production.

**What you must cover:**
- Metrics: actual usage vs requested (Goldilocks recommendations), node pressure metrics
- Trend analysis: 30/60/90-day growth extrapolation (linear + seasonal)
- Buffer policy: maintain 30% headroom for burst + scheduling headroom
- Cluster Autoscaler: max node count, scale-up speed, scale-down stabilization
- Node pool sizing: VM SKU selection — memory:CPU ratio for workload type
- Budget forecast: map capacity growth to cloud cost forecast
- Karpenter: just-in-time node provisioning for better bin-packing

---

### S-045 | Golden Path Template Design
**Type:** Design | **Difficulty:** Medium | ⬜

> Design a "golden path" service template that new microservices use as a scaffolded starting point. What must it include to be production-ready on day 1?

**What you must cover:**
- Repository structure: `src/`, `tests/`, `Dockerfile`, `Makefile`, `.github/workflows/`
- CI pipeline: lint → unit test → build → SAST → container scan → push to registry
- CD: Helm chart with production-safe defaults (`resources`, `livenessProbe`, `readinessProbe`, `topologySpreadConstraints`)
- Observability: auto-exposed `/metrics` (Prometheus), structured JSON logging, OpenTelemetry SDK
- Security defaults: non-root user, read-only root FS, drop ALL capabilities, no privilege escalation
- Secrets: empty ESO ExternalSecret template pointing to team's Vault path
- Backstage registration: `catalog-info.yaml` pre-populated

---

### S-046 | On-Call Runbook Design
**Type:** Design | **Difficulty:** Medium | ⬜

> Design a standard runbook format for your platform team's top 20 alerts. What makes a runbook actually useful at 3 AM?

**What you must cover:**
- Structure: Alert name → Severity → Impact → Immediate triage steps → Root cause decision tree → Fix → Escalation → Prevention
- Commands must be copy-paste ready — no pseudocode at 3 AM
- Decision tree: symptom → most likely cause (frequency-ordered) → verify command → fix
- Link to dashboards: pre-built Grafana link scoped to the time range of the alert
- Escalation path: who else to page if 15 min passes without resolution
- Post-mortem link: "if this is a recurring alert, link your post-mortem here"
- Automated testing: Markdown lint + link check in CI

---

### S-047 | Developer Experience (DX) Improvement
**Type:** Design | **Difficulty:** Medium | ⬜

> Developers complain that deployments take 45 minutes. Redesign the CI/CD pipeline to get under 10 minutes without sacrificing quality gates.

**What you must cover:**
- Profile current pipeline: find the slowest stages (usually: tests, Docker build, integration tests)
- Parallelization: run lint, unit tests, security scan in parallel
- Test optimization: test sharding, skip unchanged modules (affected tests only)
- Build cache: Docker BuildKit with registry cache, Bazel remote cache
- Incremental: only rebuild changed services (dependency graph)
- Test pyramid: move slow E2E tests to nightly, keep fast unit/integration in PR pipeline
- Measure: track P50/P95 pipeline duration in Grafana as a DX SLO

---

### S-048 | Designing a Platform Health Dashboard
**Type:** Design | **Difficulty:** Medium | ⬜

> Design a platform health dashboard that gives product teams a real-time, self-service view of the health of their services without needing to ask the platform team.

**What you must cover:**
- Signal aggregation: per-namespace RED metrics (Rate, Errors, Duration) from Prometheus
- SLO burn rate: current error budget remaining per service
- Infrastructure health: pod restarts, OOMKilled events, PVC utilization
- Deployment status: last deploy time, rollout progress, ArgoCD sync status
- Self-service: teams see only their namespace/service (Grafana org isolation or label filtering)
- Alerting: teams can subscribe to their own alert rules from a template library

---

### S-049 | Multi-Cloud Strategy Design
**Type:** Design | **Difficulty:** Expert | ⬜

> Leadership wants a multi-cloud strategy as insurance against Azure lock-in. How would you design the platform infrastructure to be portable — or would you push back?

**What you must cover:**
- Pushback case: multi-cloud doubles operational complexity, cost, and expertise requirements — is the risk worth it?
- If proceeding: abstract cloud-specific services behind interfaces (S3-compatible API, Kubernetes for compute)
- Avoid cloud-native services that don't have cross-cloud equivalents (or accept the portability cost)
- Terraform abstracts provisioning (provider-agnostic modules where possible)
- Kubernetes is the portability layer — but K8s flavors differ (managed addons, CNI, storage)
- Data: the hardest to move — egress costs, replication latency, data gravity
- Recommended: stay single-cloud, invest in vendor negotiations and SLA governance instead

---

### S-050 | Designing for Compliance (SOC2, PCI)
**Type:** Design | **Difficulty:** Expert | ⬜

> Your platform needs to pass a SOC2 Type II audit. What infrastructure controls must you implement? What does the auditor look at?

**What you must cover:**
- Access control: least-privilege IAM, MFA enforcement, access reviews quarterly
- Encryption: at rest (Azure Disk encryption, etcd encryption) + in transit (TLS everywhere + mTLS)
- Audit logging: K8s audit log, Azure Activity Log → SIEM (Sentinel) → immutable log storage
- Change management: all infra changes via PR (GitOps), no manual changes in prod, change log
- Vulnerability management: container scanning in CI, patch SLA (critical: 30 days)
- Incident response: documented IR process, post-mortems, annual test
- Monitoring: 24/7 alerting, anomaly detection on auth failures
- Evidence collection: automated evidence gathering (Drata, Vanta, Secureframe)

---

### S-051 | Progressive Delivery Platform Design
**Type:** Design | **Difficulty:** Expert | ⬜

> Design a progressive delivery platform that allows product teams to safely deploy to production using canary releases with automatic promotion and rollback.

**What you must cover:**
- Argo Rollouts: Rollout CR replaces Deployment, defines canary steps
- Analysis template: PromQL query on error rate, response time — auto-promote if passing
- Traffic: Nginx Ingress with canary annotations or Istio VirtualService weight-based routing
- Automatic rollback: if analysis fails, rollback without human intervention
- Notifications: Slack/Teams message at each promotion step + on rollback
- Manual gates: optional `pause` step for human approval at certain percentages (e.g., 50%)
- Dashboard: Argo Rollouts UI + Grafana panel showing old vs new version metrics side-by-side

---

### S-052 | Developer Namespaces (Ephemeral Environments)
**Type:** Design | **Difficulty:** Hard | ⬜

> Design a system that automatically creates and destroys ephemeral preview environments for every pull request — complete with the full application stack.

**What you must cover:**
- Trigger: GitHub Actions on PR `opened`/`synchronize` events
- Namespace lifecycle: create namespace → apply Kustomize overlay → expose via Ingress with PR-specific hostname
- DNS: wildcard DNS record (`*.preview.company.com`) + cert-manager for TLS
- Cleanup: destroy namespace on PR `closed` or after TTL (e.g., 24h of inactivity)
- Resource limits: ResourceQuota on preview namespaces (no runaway cost)
- Shared services: use stubs/mocks for expensive external services (Kafka, payment gateway)
- PR comment: bot posts preview URL when environment is ready (GitHub Deployments API)

---

### S-053 | Kubernetes Operator Design
**Type:** Design | **Difficulty:** Expert | ⬜

> You need to automate the lifecycle management of PostgreSQL clusters on Kubernetes. Design a Kubernetes Operator.

**What you must cover:**
- CRD: `PostgresCluster` resource with spec (replicas, version, storage, backup config)
- Controller: watches PostgresCluster events, reconciles desired vs actual state
- Reconciliation loop: idempotent — create/update/delete underlying StatefulSets, Services, Secrets
- Status conditions: `Ready`, `Degraded`, `BackupFailed` — updated by controller
- Finalizers: prevent deletion until graceful shutdown (backup taken, replicas drained)
- Versioned upgrades: operator manages minor version upgrades safely
- Existing solutions: CloudNativePG (don't reinvent — but explain how it works under the hood)

---

### S-054 | Designing a Platform for AI/ML Workloads
**Type:** Design | **Difficulty:** Expert | ⬜

> Platform teams are asked to support ML training jobs on Kubernetes with GPU nodes, large dataset access, and long-running distributed training. What platform capabilities do you add?

**What you must cover:**
- GPU node pools: NVIDIA plugin DaemonSet, `nvidia.com/gpu` resource, MIG for multi-tenant GPU
- Scheduling: gang scheduling (Volcano, YuniKorn) — all pods start together or none
- Storage: high-throughput access to training data (Azure Blob CSI → NFS for multi-pod access)
- Job lifecycle: Kubeflow Training Operator (TFJob, PyTorchJob) for distributed training
- Spot node pools: significant cost reduction for fault-tolerant training jobs (checkpointing required)
- Quotas: fair-share scheduling across teams, priority classes
- Observability: GPU utilization metrics (`dcgm-exporter`), training job duration tracking

---

### S-055 | Designing a Configuration Management System
**Type:** Design | **Difficulty:** Hard | ⬜

> 200 services have configuration that changes per environment. Design a configuration management system that enables environment-specific config without duplicating code.

**What you must cover:**
- Hierarchy: base config + environment overlay (Kustomize, Helm values per env)
- External config: ConfigMap for non-sensitive, ESO/Vault for sensitive values
- Config validation: schema enforcement (JSON Schema, OPA policy) before deploy
- Hot reload: config changes without pod restart — watch ConfigMap + SIGHUP or polling
- Audit: all config changes via GitOps (PR + review), history in Git
- Secrets: never in ConfigMap, always in Vault-backed ExternalSecret
- Feature flags for runtime config changes (flag service, not a redeploy)

---

### S-056 | Blue/Green Deployment at Database Level
**Type:** Design | **Difficulty:** Expert | ⬜

> You're doing a major application upgrade that requires a new database schema incompatible with the old application version. Design a zero-downtime blue/green deployment including the database.

**What you must cover:**
- Database challenge: can't run old + new schema simultaneously on same DB
- Solution: provision new DB instance (green DB), migrate data in background
- Dual-write period: both old and new app versions write to both DBs (via sync layer or CDC)
- Cutover: switch traffic to green (app + DB simultaneously), verify
- Rollback window: keep blue alive for 24–72 hours with read-only access
- Data validation: hash comparison between blue and green tables
- Operational complexity: this is expensive — usually avoid by designing backward-compatible schemas

---

### S-057 | Designing an Alert Routing System
**Type:** Design | **Difficulty:** Hard | ⬜

> Design an alerting system that routes alerts to the right team automatically based on the alerting service, severity, and time of day — without manual maintenance of routing rules.

**What you must cover:**
- Service ownership registry: each service has an owner (team), stored in Backstage catalog or Git
- AlertManager routes: use `service` label on alert → lookup owner from registry → route to team channel
- Severity routing: P1 = page on-call, P2 = Slack channel + email, P3 = Slack channel only
- Time-based routing: business hours vs after-hours escalation paths
- Automation: when new service registered in Backstage, auto-create AlertManager route + Slack channel
- Fallback: unmatched alerts go to platform team, trigger "missing ownership" alert

---

### S-058 | Designing for Developer On-Call Rotation
**Type:** Design | **Difficulty:** Medium | ⬜

> Propose a "you build it, you run it" on-call model for a company where currently only the ops team is on-call. How do you transition without burning out developers?

**What you must cover:**
- Prerequisite: observability must be self-service (developers can debug their own services)
- Phase 1: shadow rotation — developers on-call alongside ops, not primary
- Phase 2: product teams own their service SLOs, alerts scoped to their services
- Runbooks: product teams must write runbooks for their own alerts
- Alert quality: before handing on-call, alert volume must be below sustainable threshold (<5 pages/week)
- Escalation: platform team is always escalation path for infrastructure issues
- Incentive: tied to service reliability — team SLO burn rate is visible to leadership

---

### S-059 | Designing for GDPR Data Residency
**Type:** Design | **Difficulty:** Expert | ⬜

> You're launching in the EU. GDPR requires personal data to stay within EU borders. Design an architecture that enforces data residency while keeping global latency acceptable.

**What you must cover:**
- Identify data types: personal data (PII) must stay in EU, non-personal can be global
- Regional data plane: EU cluster in West Europe + North Europe (HA within region)
- Global control plane: non-PII config/routing data can be globally distributed
- Data layer isolation: separate DB instances per region, no cross-region replication of PII
- Enforcement: Azure Policy deny rule for storage resources outside EU regions
- Verification: data residency audit — trace data flows with OpenTelemetry
- Right to erasure: design for soft delete + eventual purge, not hard delete (referential integrity)

---

### S-060 | Log Aggregation Architecture at Scale
**Type:** Design | **Difficulty:** Hard | ⬜

> 500 services generate 50 TB of logs/day. Design a log aggregation architecture that's queryable, cost-effective, and retains 90 days.

**What you must cover:**
- Collection: Fluent Bit DaemonSet (lightweight), structured JSON output
- Hot path (last 7 days): Loki or Elasticsearch — fast query
- Cold path (7–90 days): Parquet files on Azure Data Lake (compressed ~10:1), queryable via Synapse Serverless
- Tiering: automatic rolloff from hot → cold after 7 days
- Cost: Loki much cheaper than Elasticsearch at scale (99% compression via chunks)
- Indexing: Loki uses labels only (low cardinality) — don't index request IDs as labels
- Query performance: common queries should hit recording rules or summary tables

---

## Section 4: Architecture Patterns Deep Dive (S-061 to S-080)

---

### S-061 | Cell-Based Architecture
**Type:** Architecture | **Difficulty:** Expert | ⬜

> Explain cell-based architecture and how it achieves better blast-radius isolation than traditional tiered architecture. Give a concrete implementation on Kubernetes.

**What you must cover:**
- Cell: independently deployable unit with own compute, storage, and routing
- Each cell serves a subset of users (by user ID hash, org ID, geography)
- Failure isolation: one cell's outage affects only its users
- Implementation: separate K8s namespace (or cluster) per cell, cell router at ingress
- Deployment rings: new release goes to cell 0 first, then progressively more cells
- Database: each cell has own schema/shard — no cross-cell joins
- Trade-off: cross-cell operations (user moves, global analytics) require indirection

---

### S-062 | Sidecar Pattern in Kubernetes
**Type:** Architecture | **Difficulty:** Medium | ⬜

> Explain the sidecar pattern with 3 real production use cases. What's the overhead, and when would you choose a sidecar vs a shared library vs a DaemonSet?

**What you must cover:**
- Sidecar = co-located container in same Pod, shared network/volume namespace
- Use case 1: Envoy sidecar (service mesh) — transparent mTLS, observability, traffic management
- Use case 2: Vault agent sidecar — secret injection, renewable dynamic secrets
- Use case 3: Fluent Bit sidecar — log shipping for services that write to files
- Overhead: each sidecar adds ~30–50MB RAM, 1–2% CPU, startup latency
- DaemonSet win: shared infra concerns per node (log shipping, monitoring agent)
- Shared library win: language-native features, no network hop, no container management

---

### S-063 | Ambassador Pattern
**Type:** Architecture | **Difficulty:** Medium | ⬜

> Explaining the Ambassador pattern and how it's used in Kubernetes (different from the Sidecar). Give a concrete example.

**What you must cover:**
- Ambassador: proxy that adapts external protocol/auth to internal format
- Example: pod can't speak to cloud SQL directly (different auth) — ambassador sidecar authenticates and proxies
- Cloud SQL proxy: GCP Cloud SQL Auth Proxy as ambassador sidecar
- Azure SQL: sidecar that acquires Managed Identity token and forwards connections
- vs Service Mesh: ambassador is per-pod, protocol-adapting; service mesh is cluster-wide, transparent
- Trade-off: adds container management overhead; justified when protocol translation is complex

---

### S-064 | Designing an Audit Logging System
**Type:** Design | **Difficulty:** Hard | ⬜

> Design an audit logging system that captures all privileged actions across your platform (kubectl access, Vault secret reads, Terraform applies, cloud console logins) in a tamper-proof, queryable store.

**What you must cover:**
- Sources: K8s audit log, Vault audit log, Azure Activity Log, CI/CD system events
- Centralization: all logs → Azure Event Hub → Azure Log Analytics / SIEM (Sentinel)
- Immutability: write-once storage (Azure Immutable Blob Storage), WORM policy
- Enrichment: user identity, source IP, resource affected, action, result
- Real-time alerting: alert on privilege escalation, secrets access by unusual users
- Retention: 1 year online, 7 years cold (compliance requirement)
- Query interface: Kusto Query Language (KQL) for Sentinel, saved queries for common audit questions

---

### S-065 | Data Plane vs Control Plane Separation
**Type:** Architecture | **Difficulty:** Hard | ⬜

> Explain the data plane vs control plane separation with examples from Kubernetes, Istio, and your CI/CD system. Why does this separation matter for reliability?

**What you must cover:**
- Control plane: where decisions are made (kube-apiserver, Istiod, ArgoCD controller)
- Data plane: where traffic actually flows (kubelet/pods, Envoy proxies, build runners)
- Key property: data plane must keep working even if control plane is unavailable
- Kubernetes: pods keep running if apiserver goes down; new pods cannot be scheduled
- Istio: Envoy sidecars cache last-known xDS config — traffic flows during Istiod restart
- ArgoCD: deployed workloads keep running; new syncs cannot happen during controller outage
- Failure isolation: control plane outage ≠ user traffic impacted

---

### S-066 | Designing a Multi-Region Database Strategy
**Type:** Design | **Difficulty:** Expert | ⬜

> Design a database strategy for a SaaS application that serves users in US, EU, and APAC — considering latency, data residency, and consistency.

**What you must cover:**
- Option 1: Global write + regional read replicas — low read latency, write latency to single primary
- Option 2: Regional primary per geography — writes are local, cross-region sync is async (eventual consistency)
- Option 3: Globally distributed DB (CockroachDB, Azure Cosmos DB) — automatic geo-distribution, tunable consistency
- Data residency: EU personal data cannot sync to US — partition by user region at the application layer
- Conflict resolution in multi-primary: last-write-wins, CRDTs, application-level conflict resolution
- Cost: global replication has significant data transfer costs
- Recommendation: Cosmos DB with multi-region writes + consistency level per operation type

---

### S-067 | Designing a Deployment Freeze System
**Type:** Design | **Difficulty:** Medium | ⬜

> Design a system that enforces deployment freezes during peak business periods (Black Friday, year-end close) while still allowing emergency hotfixes through.

**What you must cover:**
- Freeze configuration: calendar-based freeze windows stored in a config store (Git or feature flag system)
- Enforcement: ArgoCD: automated sync disabled during freeze (webhook or controller checks calendar)
- Emergency bypass: special "break-glass" label on PR, requires two senior engineers to approve
- Audit: every bypass is logged with reason and approvers
- Notifications: automated Slack reminder 24h before freeze starts + when freeze ends
- Testing: automated tests for freeze window edge cases in CI

---

### S-068 | Rate Limiting Architecture at Scale
**Type:** Design | **Difficulty:** Hard | ⬜

> Design a distributed rate limiting system for an API gateway serving 100K RPS with per-client rate limits that persist across multiple gateway instances.

**What you must cover:**
- Centralized counter: Redis with sliding window log, `ZADD` + `ZREMRANGEBYSCORE` per window
- Lua script for atomicity: `MULTI/EXEC` or Lua for atomic check-and-increment
- Redis cluster: hash slot per client — consistent routing, no cross-slot operations
- Local approximation: token bucket in memory per gateway node, sync to Redis every 100ms (good enough accuracy, lower Redis load)
- Algorithms: fixed window (simple, boundary attack), sliding window (accurate), token bucket (burst-friendly)
- Headers: `X-RateLimit-Remaining`, `X-RateLimit-Reset`, `Retry-After` on 429
- Quotas: tiered rates by API key tier (free/pro/enterprise)

---

### S-069 | Kubernetes Operator Lifecycle Management
**Type:** Design | **Difficulty:** Expert | ⬜

> Your team uses 15 different Kubernetes operators. Design a governance process for discovering, approving, upgrading, and deprecating operators safely.

**What you must cover:**
- OLM (Operator Lifecycle Manager): catalog sources, subscriptions, automatic upgrade channels
- Approval process: operator evaluated for security, resource usage, CRD stability before cluster-wide adoption
- Testing: operator upgrades tested in dev cluster first (staging gate required)
- CRD versioning: operators must support CRD version migration (v1alpha1 → v1beta1 → v1)
- Deprecation: announce 90 days before, provide migration guide, remove after all dependents migrate
- Monitoring: operator health (reconciliation errors, queue depth) as a platform SLO

---

### S-070 | Storage Strategy for a Stateful Kubernetes Workload
**Type:** Design | **Difficulty:** Hard | ⬜

> Design a storage strategy for a stateful Kafka deployment on Kubernetes with 10TB data, requiring high IOPS and cross-AZ durability.

**What you must cover:**
- Volume type: Azure Premium SSD (P40/P50) — 7500 IOPS / 250 MB/s per disk
- PVC per broker: `volumeClaimTemplates` in StatefulSet — each broker gets dedicated PV
- Replication: Kafka topic replication factor 3 across AZs (not storage-level replication) — ISR quorum
- Anti-affinity: `podAntiAffinity` with `topologyKey: topology.kubernetes.io/zone`
- Backup: Cruise Control for partition rebalance + object storage tier (Azure Blob via Tiered Storage)
- StorageClass: `WaitForFirstConsumer` — provision PV in same AZ as pod
- Resizing: `allowVolumeExpansion: true` on StorageClass

---

## Section 5: Trade-Off and Advanced Design (S-071 to S-100)

---

### S-071 | Kubernetes vs Serverless — Trade-Off Analysis
**Type:** Architecture | **Difficulty:** Hard | ⬜

> When would you choose Kubernetes over serverless (Azure Functions, Lambda) and vice versa? Give concrete decision criteria.

**What you must cover:**
- Kubernetes wins: long-running workloads, stateful, custom runtimes, fine-grained resource control, warm containers with latency SLO
- Serverless wins: event-driven, spiky traffic (scale to zero), fast time-to-market, no infra management
- Cost crossover: above certain consistent load, K8s is cheaper; below it, serverless is cheaper
- Hybrid: K8s for core services, Functions for async/event-driven sidecars
- Cold start: Functions cold start 200ms–2s; unacceptable for P99 < 50ms APIs
- Observability: serverless harder to trace, less control over runtime environment

---

### S-072 | Pull vs Push Metrics Collection
**Type:** Explain | **Difficulty:** Medium | ⬜

> Prometheus uses a pull model. Datadog uses a push model. Explain the trade-offs and when you'd choose each.

**What you must cover:**
- Pull (Prometheus): scraper polls targets → natural service discovery, easy to debug (curl the metrics endpoint), scalability limited by scraper
- Push (Datadog/StatsD): agents push to central collector → works for short-lived jobs, firewalls, ephemeral containers
- Pull problem: short-lived batch jobs complete before scrape interval → use Pushgateway
- Push problem: misconfigured agent floods collector; harder to discover what's sending data
- Hybrid: OpenTelemetry Collector accepts push (OTLP) and can scrape (Prometheus receiver), exports to either
- Scale consideration: pull model hard to scale to 1M endpoints; remote write + federation helps

---

### S-073 | Monorepo vs Polyrepo for Microservices
**Type:** Architecture | **Difficulty:** Medium | ⬜

> Your team of 200 engineers is debating monorepo vs polyrepo. Make the case for each and give your recommendation with conditions.

**What you must cover:**
- Monorepo wins: atomic cross-service refactoring, single CI system, shared tooling, dependency consistency (no version skew)
- Monorepo tools: Nx, Turborepo, Bazel — incremental builds, affected-only tests
- Polyrepo wins: team autonomy, independent deployment cadence, smaller repo = faster git operations
- Polyrepo problems: dependency version drift, cross-repo refactoring is painful, duplicated CI config
- Scale: Google/Meta/Microsoft use monorepos at extreme scale — but with custom tooling
- Recommendation: monorepo with Bazel/Nx for teams < 1000 engineers, with clear ownership via CODEOWNERS

---

### S-074 | Kubernetes Admission Control Pipeline Design
**Type:** Design | **Difficulty:** Expert | ⬜

> Design an admission control framework that enforces 30+ policies across a multi-tenant cluster without impacting deployment speed.

**What you must cover:**
- Layer 1: OPA Gatekeeper or Kyverno — policy-as-code, version-controlled policies
- Policy categories: security (non-root, no privilege), naming (labels, annotations), quotas
- Dry-run mode: `warn` mode first — identify violations before enforcing
- Performance: webhook timeout defaults to 10s — keep policy evaluation fast (<1s)
- Testing: `kyverno test` or OPA conftest for offline policy testing in CI
- Exception handling: policy exceptions as CRs (e.g., `PolicyException` in Kyverno) with audit trail
- Monitoring: admission webhook latency + error rate as platform SLO signal

---

### S-075 | GitOps Secret Management Trade-Off
**Type:** Architecture | **Difficulty:** Hard | ⬜

> In GitOps, all desired state should be in Git. But secrets can't be in Git. Explain the three main patterns for handling secrets in a GitOps workflow and their trade-offs.

**What you must cover:**
- Pattern 1: Sealed Secrets — encrypt in Git, decrypt in cluster. Simple, no external dependency. But: key rotation is complex, Bitnami vendor.
- Pattern 2: External Secrets Operator — CRD in Git references external secret (Vault, Azure KV). Secret material never in Git. But: requires external secret store availability.
- Pattern 3: SOPS (git-crypt) — encrypt files at rest in Git using age/GPG keys or cloud KMS. Transparent to GitOps tooling. But: KMS dependency, key management.
- Recommendation: ESO + Vault for production (dynamic secrets, audit, rotation); SOPS for bootstrap secrets (Vault initial config)

---

### S-076 | Kubernetes Network Policy Design
**Type:** Design | **Difficulty:** Hard | ⬜

> Design a network policy architecture for a multi-tenant cluster where each tenant's namespace is fully isolated by default, with explicitly declared exceptions.

**What you must cover:**
- Default-deny policy: apply to every namespace on creation (Kyverno generates it automatically)
- Allow patterns: DNS (UDP 53 to kube-dns), health check probes (kubelet CIDR)
- Cross-namespace: explicit ingress/egress NetworkPolicy for approved cross-namespace traffic
- Monitoring namespace: allow Prometheus scraping from monitoring namespace
- Ingress namespace: allow traffic from ingress-nginx namespace to any app namespace
- Cilium: Cilium NetworkPolicy for additional features — DNS-based policies, L7 policies
- Testing: `kubectl exec` into pod and verify network connectivity matches expectation

---

### S-077 | Container Image Build Best Practices
**Type:** Implement | **Difficulty:** Medium | ⬜

> Walk through an optimized, secure Dockerfile for a Python API service. Identify and fix 5 common anti-patterns.

**What you must cover:**
- Multi-stage build: builder stage (full SDK) → runtime stage (slim/distroless)
- Non-root user: `USER 1000:1000` — no root at runtime
- Read-only root filesystem (enforce via securityContext in K8s, not Dockerfile)
- Layer ordering: `COPY requirements.txt` before `COPY src/` — cache pip install
- Pin base image: `python:3.12.3-slim-bookworm@sha256:...` not `python:3.12-slim`
- No secrets in ENV, COPY, or ARG — use runtime injection
- `.dockerignore`: exclude `.git`, `__pycache__`, test files, `.env`
- HEALTHCHECK: `HEALTHCHECK CMD curl -f http://localhost:8080/health || exit 1`

---

### S-078 | Horizontal vs Vertical Pod Autoscaling
**Type:** Explain | **Difficulty:** Medium | ⬜

> Explain HPA vs VPA, when to use each, and what happens if you use both simultaneously. Design a resource sizing strategy for a new microservice.

**What you must cover:**
- HPA: scales replica count based on CPU/memory/custom metrics. Requires horizontally scalable service.
- VPA: adjusts resource requests/limits based on observed usage. Recreates pods by default (Recreate mode).
- VPA modes: Off (recommendation only), Initial (set at pod creation), Auto (recreates pods)
- Problem with HPA + VPA: both trying to manage pod resources → conflict. Use KEDA + VPA in Off mode instead.
- New service sizing: start with Goldilocks (run VPA in recommendation mode), use P95 usage as request, set limit at 2× request
- KEDA: HPA on custom metrics (queue depth, Kafka lag) — more flexible than CPU-based scaling

---

### S-079 | Designing for MTTR Reduction
**Type:** Design | **Difficulty:** Hard | ⬜

> Your team's MTTR (Mean Time To Restore) is 45 minutes. Design a set of platform capabilities that get it under 10 minutes.

**What you must cover:**
- Detection speed: reduce time from failure to alert — tight alert thresholds, synthetic monitors
- Alert quality: every alert has severity, context, dashboard link, runbook link — no investigation needed to start
- Runbook automation: common fixes are automated (runbook scripts executable from Slack)
- Rollback speed: one-click rollback via ArgoCD UI or `argocd rollback` command — test it!
- Blast radius: canary deployments limit impact to 5–10% of users initially
- War room setup: auto-create incident channel, invite on-call + service owner, post last deploy commit
- Post-mortem: MTTR tracking as a metric in Grafana — make improvement visible

---

### S-080 | Multi-Cluster Networking Design
**Type:** Design | **Difficulty:** Expert | ⬜

> Design a networking model for 5 Kubernetes clusters (dev, staging, 3 regional prod) that allows services to call each other cross-cluster securely.

**What you must cover:**
- Option 1: Istio multi-cluster (flat network) — shared control plane or replicated, east-west gateway
- Option 2: Submariner — connects cluster pod CIDRs via cross-cluster tunnel
- Option 3: Service mirroring — ArgoCD or Skupper mirrors remote services as local Services
- Security: mTLS between clusters (SPIFFE/SPIRE federation), service authorization policies
- Service discovery: cross-cluster DNS (`svc.cluster1.global`) vs central service registry (Consul)
- Latency: cross-region calls add 50–150ms — design for this, don't hide it
- Recommendation: Istio east-west gateway for same-cloud clusters; VPN/ExpressRoute + service registry for multi-cloud

---

### S-081 to S-100 | Rapid-Fire Design Questions

---

### S-081 | How would you design healthcheck endpoints for a microservice? What's the difference between liveness, readiness, and startup probes in K8s? | **Type:** Explain | **Difficulty:** Medium | ⬜

### S-082 | Design a caching strategy for an API that calls an expensive downstream service. Include cache invalidation. | **Type:** Design | **Difficulty:** Medium | ⬜

### S-083 | Your Prometheus server's disk is filling up daily. How do you fix this without losing data? | **Type:** Troubleshoot | **Difficulty:** Hard | ⬜

### S-084 | Design a build artifact lifecycle — from build, to registry, to cleanup of old images. | **Type:** Design | **Difficulty:** Medium | ⬜

### S-085 | How would you design a platform-wide enforced maintenance window for non-critical workloads? | **Type:** Design | **Difficulty:** Medium | ⬜

### S-086 | Explain the Kubernetes Deployment rollout algorithm. How does `maxSurge` and `maxUnavailable` interact during a rolling update? | **Type:** Explain | **Difficulty:** Hard | ⬜

### S-087 | Design a secure container registry strategy: scanning, signing, promotion, and access control. | **Type:** Design | **Difficulty:** Hard | ⬜

### S-088 | You need to run database migrations as part of your CD pipeline. Design a safe migration execution strategy. | **Type:** Design | **Difficulty:** Hard | ⬜

### S-089 | A service has a P99 latency of 2 seconds but P50 of 50ms. What are the likely causes? How do you investigate? | **Type:** Troubleshoot | **Difficulty:** Hard | ⬜

### S-090 | Design an alert severity classification system (P1–P4) with routing and SLA requirements for each level. | **Type:** Design | **Difficulty:** Medium | ⬜

### S-091 | How do you ensure secrets are rotated automatically for a service that uses a PostgreSQL password stored in Vault? | **Type:** Design | **Difficulty:** Hard | ⬜

### S-092 | Design a chaos engineering practice for your Kubernetes platform — tooling, runbook, scope, and guardrails. | **Type:** Design | **Difficulty:** Expert | ⬜

### S-093 | Explain gRPC vs REST vs GraphQL for service-to-service communication. When does each win? | **Type:** Explain | **Difficulty:** Medium | ⬜

### S-094 | Your Kubernetes cluster's etcd is 80% full. What are your options, and what's the immediate risk? | **Type:** Troubleshoot | **Difficulty:** Expert | ⬜

### S-095 | Design a cost-optimized log storage tier using object storage and a query layer. | **Type:** Design | **Difficulty:** Hard | ⬜

### S-096 | How would you design a platform update cadence for Kubernetes version upgrades across 20 clusters without coordinated downtime? | **Type:** Design | **Difficulty:** Expert | ⬜

### S-097 | Explain PodDisruptionBudgets. Design a PDB strategy for a 3-replica stateless service and a 2-replica StatefulSet. | **Type:** Explain | **Difficulty:** Hard | ⬜

### S-098 | Design a developer feedback loop that makes test failures visible within 5 minutes of a commit. | **Type:** Design | **Difficulty:** Medium | ⬜

### S-099 | How would you design a platform for running untrusted customer code safely (like CI jobs from external contributors)? | **Type:** Design | **Difficulty:** Expert | ⬜

### S-100 | Design a self-healing runbook that automatically detects and resolves the top 5 most common production incidents without human intervention. | **Type:** Design | **Difficulty:** Expert | ⬜
