# Mock Interviews & Full-Loop Scenarios

> 80 full-loop mock scenarios: system design prompts with constraints, on-call simulations, coding timers, and company-specific prep drills.

---

## Part 1: System Design Mock Prompts (30 timed questions)

Use the 55-minute interview format for each:
- 0–5 min: clarify requirements
- 5–10 min: scale estimation
- 10–30 min: high-level design
- 30–45 min: deep dive on 2 components
- 45–55 min: trade-offs, failure modes, cost

---

### M1. Design a global CI/CD platform for 500 microservices
**Constraints:** 10,000 developers, max build time 5 min P95, zero failed deployments should reach prod.
**Key areas to cover:** Pipeline orchestration, artifact registry, deployment strategy, rollback, observability, multi-region.
**Curveball:** "Now make it work for monorepo AND polyrepo setups."

---

### M2. Design a Kubernetes multi-tenant platform
**Constraints:** 50 product teams, each with dev/staging/prod environments, strict network isolation, cost chargeback per team.
**Key areas:** Namespace hierarchy, RBAC, network policies, resource quotas, admission control, cost allocation.
**Curveball:** "A team needs root access for a legacy workload. How do you handle it?"

---

### M3. Design an Internal Developer Platform (IDP)
**Constraints:** Self-service: teams should be able to provision a new service with CI/CD, observability, and DNS in < 15 minutes.
**Key areas:** Service catalog (Backstage), golden path templates, GitOps automation, secret injection.
**Curveball:** "How do you prevent teams from drifting from the golden path over time?"

---

### M4. Design a high-availability logging system for 1M events/second
**Constraints:** 30-day retention, sub-second search, multi-region, < $500k/year infra cost.
**Key areas:** Ingestion (Fluent Bit), storage (Loki/OpenSearch), indexing strategy, compaction, tiered storage.
**Curveball:** "The search index doubled in cost unexpectedly. How do you investigate?"

---

### M5. Design a secrets management system at scale
**Constraints:** 10,000 applications, each with dynamic DB credentials rotated every 24h, zero plaintext secrets in Git or registries.
**Key areas:** Vault architecture (HA, DR), K8s auth, dynamic secrets, audit trail, break-glass process.
**Curveball:** "Vault is down during a deployment. What happens?"

---

### M6. Design a zero-downtime database migration strategy
**Constraints:** 5TB PostgreSQL, 100k RPS, migrating to a new schema, no maintenance window.
**Key areas:** Expand/contract pattern, dual-write, CDC (Debezium), traffic shifting, rollback.
**Curveball:** "The migration falls 2 days behind schedule. Do you pause? Roll back? Continue?"

---

### M7. Design a multi-region active-active AKS deployment
**Constraints:** 3 regions, latency < 50ms P99, RPO < 1 min, RTO < 5 min.
**Key areas:** Global load balancer (AFD), data replication, conflict resolution, health checks, failover automation.
**Curveball:** "One region loses connectivity to the other two. How does the system behave?"

---

### M8. Design a container registry with CVE scanning and policy enforcement
**Constraints:** 500 image pushes/day, no image with CRITICAL CVEs should reach prod, SBOM required for compliance.
**Key areas:** Registry (ACR/ECR/Harbor), Trivy/Grype integration, admission webhook, image signing, audit log.
**Curveball:** "A critical CVE is published in a base image used by 200 services. What's your response?"

---

### M9. Design an observability stack for a microservices platform (100 services)
**Constraints:** Metrics (sub-1-min resolution), distributed traces (100% sampling for errors, 1% otherwise), logs (30 days), total cost < $200k/year.
**Key areas:** Prometheus + Thanos/Mimir, OTel collector, Tempo/Grafana, Loki, alert routing, SLO dashboards.
**Curveball:** "Prometheus cardinality explodes — one team added a high-cardinality label. How do you detect and fix this?"

---

### M10. Design a disaster recovery system for a critical payment service
**Constraints:** RPO = 0 (no data loss), RTO = 30 seconds, budget constraint: DR environment must be < 30% of prod cost when idle.
**Key areas:** Synchronous replication, warm standby, DNS-based failover, health checks, chaos testing.
**Curveball:** "Your DR test caused a partial failover in production. Walk me through what happened and how you prevent it."

---

### M11. Design a FinOps cost optimization system
**Constraints:** $5M/month cloud spend, 20% waste target, 15 teams with no visibility into their spend.
**Key areas:** Resource tagging enforcement, showback/chargeback, right-sizing automation (Goldilocks/VPA), savings plan management, idle resource cleanup.
**Curveball:** "One team is consistently 200% over budget. What do you do?"

---

### M12. Design a GitOps promotion pipeline (dev → staging → prod)
**Constraints:** Automated promotion to staging on merge, manual gate for prod, full audit trail, rollback in < 2 min.
**Key areas:** ArgoCD App of Apps, sync waves, image tag promotion automation (Image Updater), approval workflows.
**Curveball:** "A config drift is detected in prod that doesn't match Git. ArgoCD shows OutOfSync. What happened and what do you do?"

---

### M13. Design an ephemeral preview environment system
**Constraints:** Every PR spawns a full stack (app + DB + infra) in < 5 min, auto-destroyed when PR closes, cost-capped per environment.
**Key areas:** Namespace-per-PR, Helm install, database seeding, DNS (\*.preview.company.com), cost limit via resource quotas.
**Curveball:** "The preview environment DB has real data in it accidentally. How does this happen and how do you prevent it?"

---

### M14. Design a Kubernetes cluster autoscaling strategy for bursty workloads
**Constraints:** Black Friday-style traffic spikes (10x normal), must scale out in < 2 min, scale-in must not kill active requests.
**Key areas:** HPA with custom metrics, Cluster Autoscaler vs Karpenter, spot instance strategy, PodDisruptionBudget, preStop hooks.
**Curveball:** "During a spike, node provisioning takes 4 minutes and requests are timing out. What do you do?"

---

### M15. Design a supply chain security system for your container pipeline
**Constraints:** Every image must be signed, provenance must be verifiable, base images auto-updated weekly, SBOM stored and queryable.
**Key areas:** Cosign + Sigstore, GitHub Actions OIDC, policy admission (Kyverno/OPA), Dependabot/Renovate for base images.
**Curveball:** "A malicious package was injected into your base image upstream. How do you detect and respond?"

---

### M16. Design a global rate limiting system for an API gateway
**Constraints:** 50,000 RPS peak, per-user limits, distributed across 5 regions, < 5ms latency overhead.
**Key areas:** Token bucket algorithm, Redis cluster (CRDT or lua scripts), Envoy rate limit service, fallback behavior when Redis is unavailable.
**Curveball:** "Redis is degraded. Do you fail open or fail closed? What's the implication of each?"

---

### M17. Design a blue-green deployment system with automated traffic shifting
**Constraints:** Zero-downtime deploy, automatic rollback if error rate > 1% within 5 minutes of traffic shift, support for 3-minute canary phases.
**Key areas:** Argo Rollouts, Prometheus-based analysis template, traffic weights, header-based routing for testing.
**Curveball:** "The new version passes the canary phase but fails for 0.1% of users on a specific browser. How do you detect this?"

---

### M18. Design a distributed tracing system from scratch
**Constraints:** 200 services, 10,000 traces/second at peak, 7-day full retention, 30-day sampled retention, < $50k/month.
**Key areas:** OTel SDK + collector, Tempo architecture, sampling strategy (head vs tail), storage tiering (S3), Grafana integration.
**Curveball:** "A critical slowdown is reported but traces aren't showing it. Why might this happen?"

---

### M19. Design a RBAC system for a multi-tenant Kubernetes platform
**Constraints:** 50 product teams, 5 platform engineers, quarterly access review required, no ClusterAdmin for product teams.
**Key areas:** Namespace-scoped roles, aggregated ClusterRoles, Group-based binding (Entra ID groups), admission webhook to prevent privilege escalation, audit trail.
**Curveball:** "A team finds a way to escalate privileges via a misconfigured ClusterRoleBinding. What's your detection and response?"

---

### M20. Design a chaos engineering program for a payment platform
**Constraints:** Can't affect real financial transactions. Must prove system can handle node failure, network partition, and latency injection.
**Key areas:** Chaos Monkey / Chaos Mesh / LitmusChaos, steady-state hypothesis, game days, blast radius limiting, rollback.
**Curveball:** "A chaos experiment leaked into production and caused 30 minutes of degradation. How do you prevent this?"

---

### M21. Design an auto-remediation system for infrastructure incidents
**Constraints:** Common alerts (disk full, OOMKilled, pod restarts) should self-heal without paging an engineer. Risky actions require human approval.
**Key areas:** Alert → event bus → runbook-as-code (Robusta/Argo Events), risk classification, audit trail, feedback loop.
**Curveball:** "The auto-remediation worsened the incident by restarting pods that were healthy. How do you prevent false positives?"

---

### M22. Design a Kubernetes cluster upgrade strategy (zero downtime)
**Constraints:** 50 clusters, mixed workloads (stateful + stateless), prod can't have downtime, upgrades must complete in < 4h per cluster.
**Key areas:** Node image upgrade first, control plane upgrade, PDB compliance check, rollback path, canary cluster first.
**Curveball:** "Mid-upgrade, a new critical CVE is announced in the version you're upgrading to. What do you do?"

---

### M23. Design a centralized secrets rotation system
**Constraints:** 500 services, 3 cloud providers, DB passwords must rotate every 30 days, zero service downtime during rotation.
**Key areas:** Vault dynamic secrets, lease renewal, K8s annotation-triggered rotation, health check window for rotation, rollback.
**Curveball:** "A secret was rotated but 20 services didn't pick up the new value. How do you detect and resolve?"

---

### M24. Design a developer onboarding automation system
**Constraints:** A new engineer should have full local + cloud development environment set up in < 2 hours, zero manual IT tickets.
**Key areas:** GitHub provisioning, IaC for dev namespaces, Vault access grant, VPN, SSO, starter tasks in JIRA.
**Curveball:** "A contractor needs access but with restricted scope and time-limited. How does your system handle that?"

---

### M25. Design a network segmentation strategy for a Kubernetes platform
**Constraints:** 100 services, PCI workloads must be isolated, no east-west traffic between teams without explicit allowlist.
**Key areas:** Default-deny NetworkPolicy, Cilium cluster-wide network policy, namespace labeling, mTLS enforcement, audit logging.
**Curveball:** "A new service needs to talk to a PCI-scoped service. Walk me through the approval and implementation process."

---

### M26. Design a deployment frequency measurement and DORA metrics system
**Constraints:** 50 teams, multiple CI/CD tools (GitHub Actions + GitLab CI + Jenkins), track all 4 DORA metrics.
**Key areas:** Event collection (webhook → event store), deployment event schema, lead time calculation, MTTR tracking, dashboard.
**Curveball:** "Teams start gaming the metrics by deploying smaller, more frequent changes that don't reduce risk. How do you detect this?"

---

### M27. Design a multi-cloud cost allocation and showback system
**Constraints:** Azure + AWS, 30 product teams, costs allocated by namespace tag, monthly report auto-generated and emailed to team leads.
**Key areas:** Tag enforcement (Policy), Kubecost/OpenCost, cross-cloud normalization, allocation model (direct + shared infra split).
**Curveball:** "A team disputes their allocation is wrong. How do you audit and resolve?"

---

### M28. Design a certificate lifecycle management system
**Constraints:** 1,000 TLS certificates across cloud, K8s, and internal PKI, all must be rotated 30 days before expiry, zero manual steps.
**Key areas:** cert-manager with multiple issuers (ACME + Vault PKI + internal CA), expiry alerting, auto-renewal, rotation testing.
**Curveball:** "cert-manager fails to renew a certificate due to DNS challenge failure. What are your detection and manual recovery steps?"

---

### M29. Design a feature flag system integrated with your CD pipeline
**Constraints:** Flag changes deploy without a code release, different flag values per environment, kill switch deployable in < 30 seconds.
**Key areas:** OpenFeature standard, flag evaluation service (Flagd/LaunchDarkly), flag lifecycle (create → rollout → archive), audit trail.
**Curveball:** "A feature flag caused a production incident because it was set to the wrong value after a copy-paste error. How do you prevent?"

---

### M30. Design an infrastructure compliance scanning system
**Constraints:** Every Terraform plan is scanned before apply, every running K8s cluster is audited daily, compliance report generated weekly.
**Key areas:** checkov/OPA in CI, kube-bench + Polaris for runtime, policy-as-code versioning, exception management, ticketing integration.
**Curveball:** "A compliance scan is blocking 20 PRs because of a false positive. Engineers are annoyed. What do you do?"

---

## Part 2: On-Call Fire Drill Simulations (20 scenarios)

Simulate being paged at 2am. Walk through your response in under 3 minutes before diving into investigation.

---

### M31. PagerDuty fires: "API gateway 5xx rate > 10% for 5 minutes"
**Immediate:** What are your first 3 kubectl commands?
**Red herrings:** Recent deploy happened 2h ago (not the cause). Actual cause: upstream service OOM.
**Learning:** Don't assume most recent change is the cause.

---

### M32. Alert: "etcd latency P99 > 1 second"
**Immediate:** Check etcd member health, disk IOPS, raft index lag.
**Red herrings:** Network is fine. Actual cause: etcd disk is co-located with high-write workload.
**Learning:** etcd needs dedicated, fast SSDs. Control plane and data plane storage must be separated.

---

### M33. Alert: "PersistentVolume full — 95% usage on payment-service"
**Immediate:** Identify what's consuming space. Options: expand PVC, clean old data, move to object storage.
**Red herrings:** App team says "we don't write much." Actual cause: WAL logs not being cleaned up.
**Learning:** PVC expansion is online for most CSI drivers, but requires PVC `allowVolumeExpansion: true` on StorageClass.

---

### M34. Alert: "Kubernetes Node NotReady — 3 of 10 nodes"
**Immediate:** `kubectl describe node`, check kubelet logs, check containerd, check disk/memory pressure.
**Red herrings:** Cloud provider dashboard shows nodes are healthy. Actual cause: containerd socket hung due to a buggy DaemonSet.
**Learning:** Node NotReady ≠ VM down. kubelet and containerd can both fail while the VM is up.

---

### M35. Alert: "Deployment rollout stuck — 0 new pods becoming Ready"
**Immediate:** `kubectl rollout status`, `kubectl get pods`, check readiness probe, check image pull.
**Red herrings:** Image exists in registry. Actual cause: readiness probe HTTP endpoint returns 200 only after 60s warmup but probe `initialDelaySeconds` is 5.
**Learning:** Always check probe configuration when rollout stalls.

---

### M36. Alert: "Prometheus scrape target down — 40 targets"
**Immediate:** Check Prometheus targets page. Check ServiceMonitor CRD. Check pod labels match selector.
**Red herrings:** Pods are running fine. Actual cause: namespace label changed, ServiceMonitor selector no longer matches.
**Learning:** ServiceMonitor selectors must exactly match pod/service labels including namespace.

---

### M37. Alert: "TLS certificate expires in 6 hours — api.company.com"
**Immediate:** Is cert-manager running? Check Certificate resource status. Check ACME challenge. Check DNS propagation.
**Red herrings:** ACME challenge seems to succeed. Actual cause: DNS-01 challenge records are being created but in wrong zone (split-horizon DNS).
**Learning:** cert-manager DNS-01 challenges can fail silently if the solver is querying the wrong DNS server.

---

### M38. Alert: "Kafka consumer lag > 1 million messages on payment-processor"
**Immediate:** Check consumer group status. Check pod logs for errors. Check thread count / connection pool.
**Red herrings:** CPU and memory are fine. Actual cause: consumer group ID changed in a deployment, creating a new group starting from latest.
**Learning:** Consumer group ID changes reset the offset. Always validate consumer group continuity in deploys.

---

### M39. Alert: "ArgoCD application Out Of Sync and auto-sync failing"
**Immediate:** Check ArgoCD app events. Check Git commit that changed it. Check if resource has finalizer blocking deletion.
**Red herrings:** Git looks correct. Actual cause: A CRD was removed from the cluster but still exists in Git. ArgoCD can't delete what doesn't exist.
**Learning:** CRD deletion from clusters must be coordinated with Git state.

---

### M40. Alert: "Vault is sealed — 100% of dynamic secrets are failing"
**Immediate:** Check Vault pod status. Unseal if auto-unseal (Azure KMS) failed. Check Azure KMS key status.
**Red herrings:** Vault pods are running. Actual cause: Vault auto-unseal key was rotated in Key Vault without updating Vault's configuration.
**Learning:** Vault auto-unseal key rotation must be coordinated. Always keep the old key version active during transition.

---

### M41. Alert: "DNS resolution failing intermittently for all pods"
**Immediate:** Test from inside a pod: `nslookup kubernetes.default`. Check CoreDNS pod status and logs. Check CPU throttling on CoreDNS.
**Red herrings:** CoreDNS pods are running. Actual cause: CoreDNS is CPU-throttled due to no CPU limit (ironic — limit too low causes throttling).
**Learning:** DNS failures are often resource starvation on CoreDNS. Set appropriate requests AND limits.

---

### M42. Alert: "CI/CD pipeline 100% failure rate for 15 minutes"
**Immediate:** Check recent changes to pipeline config. Check runner health. Check secret availability.
**Red herrings:** Runners are healthy. Actual cause: GitHub Actions OIDC token endpoint is down (GitHub incident).
**Learning:** External dependencies (GitHub, DockerHub, cloud APIs) are failure points. Have fallback for runner auth.

---

### M43. Alert: "Memory usage on prod cluster nodes at 97%"
**Immediate:** `kubectl top nodes`. Find which namespace/pod is using most memory. Check for memory leak pattern.
**Red herrings:** Top consumer is a known service. Actual cause: A new deployment has no memory limit and a memory leak from a recent code change.
**Learning:** Always set memory limits. Unbounded memory leads to node pressure and eviction cascades.

---

### M44. Alert: "External load balancer health check failing — 429 Too Many Requests from backend"
**Immediate:** Check which service is rate-limiting. Check if the LB health check is misconfigured to hit a rate-limited endpoint.
**Red herrings:** App seems healthy from inside cluster. Actual cause: LB health check is hitting `/api/v1/search?q=*` (a slow endpoint) every 5 seconds, which counts toward rate limits.
**Learning:** Health check endpoints must be lightweight and exempt from business-logic rate limiting.

---

### M45. Alert: "Pod eviction spike — 50 pods evicted in 5 minutes"
**Immediate:** Check node conditions: `kubectl describe node | grep -A5 Conditions`. Check if nodes are under memory pressure.
**Red herrings:** Resource quotas look fine. Actual cause: A DaemonSet was updated and its memory request increased, causing node memory pressure.
**Learning:** DaemonSet resource requests consume node allocatable capacity for every pod. Small increases multiply by node count.

---

### M46. Alert: "Readiness probe failures across 30% of fleet — service degraded"
**Immediate:** Check which change preceded the alert. Check probe endpoint behavior. Check dependency health (DB, Redis, external API).
**Red herrings:** App code wasn't changed. Actual cause: DB connection pool exhausted due to connection leak from a recent config change.
**Learning:** Readiness probe failures can be symptoms of dependency issues, not the app itself.

---

### M47. Alert: "Terraform state lock not released — 2 hours"
**Immediate:** Check who holds the lock (Azure Blob lease metadata). Check if the `terraform apply` process is still running or was killed.
**Red herrings:** State file looks intact. Actual cause: The pipeline runner was OOM killed mid-apply, leaving the lease unreleased.
**Learning:** Terraform state locks must be manually released when runner is killed. Azure: `az storage blob lease break`.

---

### M48. Alert: "API latency P99 spike from 150ms to 4s"
**Immediate:** Check traces for the slow requests. Check DB query execution time. Check connection pool saturation.
**Red herrings:** Infrastructure metrics are fine. Actual cause: A missing index on a new DB column added in the last deploy. Full table scan on 10M rows.
**Learning:** Any schema change needs EXPLAIN ANALYZE validation. Missing indexes survive deployment unless you look for slow queries.

---

### M49. Alert: "Kubernetes API server response time > 5 seconds"
**Immediate:** Check etcd latency. Check number of objects in etcd (large object count = slow list). Check controller-manager reconciliation loop.
**Red herrings:** Nodes are healthy. Actual cause: A CRD watch is generating massive event streams; a controller has a bug that creates O(n²) API calls.
**Learning:** API server slowness is often etcd pressure or a misbehaving controller. Use `kubectl top` and API priority/fairness metrics.

---

### M50. Alert: "Cost spike — Azure spend 3x normal in last 24h"
**Immediate:** Check Azure Cost Management by resource group. Check for any new resource type. Check compute usage.
**Red herrings:** No new deployments. Actual cause: A load test was left running in prod by accident, spawning 200 extra VMs via autoscaling.
**Learning:** Load tests must never target prod. Cost anomaly detection should trigger in <1h for 2x deviations.

---

## Part 3: Coding Round Simulations (15 timed scenarios)

---

### M51. Coding Round: Log Aggregator (45 min)
**Prompt:** "Write a program that reads multiple log files in parallel and prints all ERROR lines with the originating filename, sorted by timestamp."
**Constraints:** Must be concurrent. Files can be large (10GB). Timestamps are ISO format.
**What's tested:** Concurrency, streaming file reads, sorting.

---

### M52. Coding Round: Kubernetes Resource Auditor (45 min)
**Prompt:** "Write a CLI tool that lists all deployments across all namespaces with their current vs desired replica count. Flag any with 0 ready replicas."
**Constraints:** Use the official K8s Python client. Output should be table-formatted.
**What's tested:** K8s client, output formatting, error handling.

---

### M53. Coding Round: Config Differ (30 min)
**Prompt:** "Given two Helm values.yaml files, write a function that outputs all keys that differ between them, including nested keys."
**Constraints:** Keys in only one file should show as 'added' or 'removed'. Depth can be arbitrary.
**What's tested:** Recursive dict traversal, clear output format.

---

### M54. Coding Round: Retry with Jitter (20 min)
**Prompt:** "Implement a retry decorator with exponential backoff and optional random jitter. It should accept configurable max retries, backoff factor, and exception types."
**What's tested:** Decorator pattern, randomness, edge cases (zero retries, immediate success).

---

### M55. Coding Round: On-Call Scheduler (45 min)
**Prompt:** "Given a list of engineers and a 4-week on-call schedule (1 primary, 1 secondary per week), write a function that generates a fair schedule ensuring no engineer is primary more than twice in 4 weeks and primary/secondary don't repeat consecutively."
**What's tested:** Constraint solving, greedy scheduling, fair distribution.

---

### M56. Coding Round: Alert Grouping (30 min)
**Prompt:** "Given a stream of alerts with fields {name, labels, fired_at}, group alerts with identical labels together into an 'alert group'. Alerts in the same group fired within 2 minutes of each other."
**What's tested:** Time-window grouping, dict operations, edge cases.

---

### M57. Coding Round: Kubernetes Manifest Validator (45 min)
**Prompt:** "Write a function that reads a Kubernetes YAML manifest and validates: (1) resource limits are set, (2) runAsNonRoot is true, (3) readinessProbe is defined for all containers."
**What's tested:** YAML parsing, traversal, clear violation reporting.

---

### M58. Coding Round: Cost Report Generator (30 min)
**Prompt:** "Given a JSON file with resource usage records {team, resource_type, cost_usd, date}, generate a monthly summary by team with total cost and top 3 most expensive resource types."
**What's tested:** Data aggregation, sorting, output formatting.

---

### M59. Coding Round: Parallel Health Checker (30 min)
**Prompt:** "Given a list of HTTP endpoints, check their health concurrently. Return a summary of OK, degraded (2xx but > 500ms), and failed endpoints."
**What's tested:** Concurrent requests, timeout handling, result categorization.

---

### M60. Coding Round: Simple Deployment Pipeline Simulator (60 min)
**Prompt:** "Implement a pipeline executor that runs stages sequentially, where each stage is a list of steps that run in parallel. If any step fails, the stage fails and subsequent stages don't run."
**What's tested:** Concurrency model, failure propagation, pipeline state management.

---

### M61. Coding Round: Semver Comparator and Upgrade Planner (30 min)
**Prompt:** "Given a list of services with their current and latest available versions (semver), identify which have breaking changes (major version bump) vs safe upgrades (minor/patch)."
**What's tested:** Semver parsing, comparison logic, categorization output.

---

### M62. Coding Round: Resource Quota Calculator (30 min)
**Prompt:** "Given a list of pods with their CPU/memory requests, calculate namespace-level totals and flag the top 3 pods consuming the most resources as potential candidates for right-sizing."
**What's tested:** Aggregation, sorting, unit conversion (m, Mi, Gi).

---

### M63. Coding Round: Pull Request Risk Scorer (45 min)
**Prompt:** "Write a function that scores the risk of a PR based on: files changed (weight: 0.3), lines changed (weight: 0.2), number of services impacted (weight: 0.3), age of code modified (weight: 0.2). Output: low/medium/high risk."
**What's tested:** Weighted scoring, normalization, configurable thresholds.

---

### M64. Coding Round: Event Deduplicator (20 min)
**Prompt:** "Write a function that deduplicates a stream of events where each event has an {id, type, timestamp}. Events with the same id within a 5-minute window should be deduplicated."
**What's tested:** Time-window deduplication, data structure choice.

---

### M65. Coding Round: YAML to Terraform Variable Converter (45 min)
**Prompt:** "Given a YAML config file, generate a Terraform `variables.tf` and `terraform.tfvars` file from it. Support string, number, bool, and list types."
**What's tested:** YAML parsing, type inference, Terraform HCL generation.

---

## Part 4: Behavioral Interview Simulations (15 prompts)

---

### M66. Full Behavioral Round Simulation (60 min)
**Format:**
- 0–15 min: "Tell me about yourself" + career arc
- 15–30 min: Two STAR stories (technical leadership + conflict)
- 30–45 min: Two STAR stories (failure + prioritization under pressure)
- 45–60 min: Questions you ask the interviewer

**Drill:** Record yourself on video. Watch for: hesitation, "we" instead of "I", vague results, no trade-offs named.

---

### M67. The "Why are you leaving?" drill
**Prompt:** Practice delivering a positive, forward-looking answer in < 90 seconds.
**Frame:** "I've accomplished X in my current role. I'm looking for Y (larger scale, platform engineering focus, product company impact). This role specifically excites me because Z."
**Never say:** Salary, politics, bad manager (even if true).

---

### M68. The "Tell me about yourself" drill (senior version)
**Target:** 2 minutes. Cover career arc (not CV recitation), core expertise, 1–2 major impact statements, why you're here.
**Practice until:** You don't sound scripted AND you don't forget your impact numbers.

---

### M69. The "Why this company?" drill
**Requirement:** Reference something specific — engineering blog post, a technical decision they made, a problem they're known for. Not generic praise.
**Bad:** "I love the culture and the impact you have."
**Good:** "I read your post on how you solved the distributed tracing cardinality problem at 10 billion spans/day. That's the kind of infrastructure challenge I want to work on."

---

### M70. The "What's your weakness?" drill
**Frame:** A real skill gap you're actively working on. Not a fake strength in disguise.
**Example approach:** "Historically I've been more comfortable in the technical layer than stakeholder communication. Over the last year I've been deliberate about X [specific practice]."

---

### M71. The reverse interview: practice asking questions
**Drill:** For each of these areas, formulate 1 targeted question:
- Engineering culture
- On-call experience
- Technical direction / biggest challenges
- Career growth
- How decisions get made

**Remember:** The last 5 minutes of your interview matter. Asking sharp questions signals intelligence.

---

### M72. The "Tough interviewer" simulation
**Scenario:** The interviewer pushes back on every answer. "But what if that didn't work?" "How would you have done that differently?" "What was YOUR specific contribution?"
**Drill:** Have a partner (or record yourself) play adversarial. Practice holding your position confidently when you're right, and updating gracefully when you learn something.

---

### M73. Whiteboarding system design out loud
**Drill:** Set a 5-minute timer. Draw and narrate simultaneously for a prompt (e.g., "Design a CI/CD system"). Don't pause to think silently — keep talking about what you're considering, even if uncertain.
**Evaluators want:** Structured thinking, not the perfect answer immediately.

---

### M74. The "Take-home" scenario — infrastructure architecture doc
**Prompt:** Companies sometimes ask for a 1–2 page architecture doc as homework. Practice writing one for: "Design the infrastructure for a new payments microservice."
**Must include:** Diagram (ASCII or Mermaid), component justifications, trade-offs, cost estimate, open questions.

---

### M75. The "Code review" round simulation
**Prompt:** You're handed a Terraform module or a Python script and asked to review it live.
**Practice:** Read it from top to bottom. Say out loud what you see (security issues, style, error handling, missing validation). Always comment on: secrets, error paths, idempotency.

---

## Part 5: Interview Debrief Templates

---

### M76. Post-Interview Debrief Form

```markdown
## Interview: [Company] — [Date] — [Round Type]

### What I was asked
1.
2.
3.

### Where I felt strong

### Where I fumbled

### Questions I couldn't answer

### Follow-up to study

### What I'd do differently
```

---

### M77. Company Interview Process Tracker

| Company | Stage | Date | Outcome | Notes |
|---|---|---|---|---|
| | Phone Screen | | | |
| | Technical Screen | | | |
| | System Design | | | |
| | Coding Round | | | |
| | Behavioral | | | |
| | Final / Offer | | | |

---

### M78. Offer Comparison Matrix

| Factor | Company A | Company B | Company C | Weight |
|---|---|---|---|---|
| Base Salary | | | | 25% |
| Stock / RSU | | | | 30% |
| Tech Stack Fit | | | | 15% |
| Growth Scope | | | | 15% |
| Culture Signal | | | | 10% |
| Work-Life / On-Call | | | | 5% |

---

### M79. 30-60-90 Day Plan Template (for offer acceptance)

```markdown
## 30 Days — Listen and Learn
- Meet everyone on the team
- Shadow on-call rotation
- Read the last 6 months of post-mortems
- Understand the top 3 reliability risks

## 60 Days — Contribute
- Fix one well-scoped problem
- Write documentation that was missing
- Propose one observability improvement

## 90 Days — Drive
- Lead one small but impactful initiative
- Present findings on a gap area
- Establish credibility with stakeholders outside the team
```

---

### M80. Final Pre-Interview Checklist

- [ ] Resume reviewed — numbers are accurate, no typos
- [ ] 15–20 STAR stories prepared and practiced out loud
- [ ] Top 5 system design patterns rehearsed with diagrams
- [ ] Research done: company engineering blog, tech stack, recent incidents/posts
- [ ] Questions prepared for each interviewer type (engineering, manager, VP)
- [ ] Environment ready: working internet, quiet room, pen+paper for whiteboard
- [ ] Know the interview format: how many rounds, duration, expectations
- [ ] Good night's sleep. You know this material.
