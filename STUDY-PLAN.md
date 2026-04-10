# Study Plan — 3-Month DevOps Interview Prep

**Start Date:** April 7, 2026  
**Target:** Senior / Staff / Principal DevOps & DevSecOps interviews  
**Pace:** Thorough (3+ months, ~10–15 hrs/week)

---

## Phases Overview

| Phase | Duration | Theme |
|---|---|---|
| **Phase 1** | Weeks 1–4 | Foundations: System Design + K8s Internals + Networking |
| **Phase 2** | Weeks 5–8 | Deep Dive: Azure + IaC + CI/CD + Observability |
| **Phase 3** | Weeks 9–12 | Confidence Gaps: Security + Scripting Rounds + FinOps |
| **Phase 4** | Weeks 13–16 | Execution: Behavioral + Mock Interviews + Company Prep |

---

## Phase 1 – Weeks 1–4: Foundations

### Week 1 — System Design Fundamentals
**Goal:** Be able to design any infrastructure or platform system from scratch under interview conditions.

| Day | Topic | Task |
|---|---|---|
| Mon | Distributed systems | Read CAP theorem, PACELC, consistency models. Write notes. |
| Tue | Availability patterns | Study failover, circuit breaker, bulkhead, retry patterns |
| Wed | Scaling patterns | Horizontal vs vertical, sharding, CDN, caching tiers (L1/L2, Redis) |
| Thu | Event-driven design | Kafka fundamentals, event sourcing, CQRS |
| Fri | Practice | Design a "Deploy at scale" system — CI/CD for 1000+ microservices |
| Weekend | Review | Awesome-scalability repo — read top 20 articles |

**Deliverable:** [01-system-design/case-studies/deploy-at-scale.md](01-system-design/case-studies/deploy-at-scale.md)

---

### Week 2 — System Design: Platform & SRE Focus
**Goal:** Handle platform-engineering-style design questions.

| Day | Topic | Task |
|---|---|---|
| Mon | Multi-region HA | Design a multi-region active-active AKS cluster |
| Tue | Observability system design | Design metrics + logs + traces platform from scratch |
| Wed | Internal developer platform | Design an IDP (like Backstage) — self-service infra |
| Thu | API gateway + service mesh | Design ingress, mTLS, traffic management |
| Fri | Practice | Design a zero-downtime database migration strategy |
| Weekend | Review | Revisit and improve previous case study docs |

**Deliverable:** [01-system-design/case-studies/](01-system-design/case-studies/)

---

### Week 3 — Kubernetes Internals
**Goal:** Answer any deep K8s internals question confidently.

| Day | Topic | Task |
|---|---|---|
| Mon | Control plane deep dive | kube-apiserver flow, etcd raft consensus, leader election |
| Tue | Scheduling | Scheduler extender, node affinity, taints/tolerations, topology spread |
| Wed | Networking internals | CNI deep dive (Cilium vs Calico), kube-proxy iptables vs eBPF |
| Thu | Storage internals | CSI drivers, dynamic provisioning, ReadWriteMany patterns |
| Fri | RBAC + Security | PSA, OPA Gatekeeper, Kyverno policies |
| Weekend | Hands-on | Spin up local cluster (kind/k3d), trace a pod scheduling end-to-end |

**Deliverable:** [02-kubernetes/internals/](02-kubernetes/internals/)

---

### Week 4 — Kubernetes Troubleshooting + Networking Fundamentals
**Goal:** Diagnose any K8s failure without hesitation. Nail networking fundamentals.

| Day | Topic | Task |
|---|---|---|
| Mon | K8s troubleshooting | CrashLoopBackOff, OOMKilled, ImagePullBackOff, Pending — root causes + fixes |
| Tue | K8s troubleshooting | Networking issues — DNS failure, service unreachable, readiness probe misconfig |
| Wed | DNS deep dive | Resolution chain, CoreDNS config, ndots, search domains |
| Thu | TLS/mTLS | Handshake walkthrough, cert-manager, SPIFFE/SPIRE, Istio mTLS |
| Fri | Load balancers | L4 vs L7, ALB vs NLB, Nginx Ingress vs Gateway API |
| Weekend | Practice | Write 10 troubleshooting runbooks in [02-kubernetes/troubleshooting/](02-kubernetes/troubleshooting/) |

**Deliverable:** 10 runbooks in `02-kubernetes/troubleshooting/`

---

## Phase 2 – Weeks 5–8: Deep Dives

### Week 5 — Azure Deep Dive
**Goal:** Own the Azure platform narrative — networking, security, cost.

| Day | Topic | Task |
|---|---|---|
| Mon | Azure Networking | VNet peering, Private DNS, Private Endpoints, Hub-Spoke topology |
| Tue | AKS advanced | Node pools, KEDA, workload identity, CNI Overlay vs Kubenet |
| Wed | Azure Policy | Built-in policies, custom policy, Blueprints, Defender for Cloud |
| Thu | Key Vault + Managed Identity | ESO, CSI secrets driver, workload identity federation |
| Fri | Cost Management | Budgets, cost allocation tags, advisor recommendations |
| Weekend | Practice | Write architecture diagram + decision log for a Landing Zone |

**Deliverable:** [03-cloud-azure/architecture/landing-zone.md](03-cloud-azure/architecture/landing-zone.md)

---

### Week 6 — Infrastructure as Code (Terraform)
**Goal:** Be the "IaC expert" in the room, not just a user.

| Day | Topic | Task |
|---|---|---|
| Mon | Terraform internals | State file structure, refresh, plan, apply cycle; locking mechanics |
| Tue | Module design | Versioned modules, input/output contracts, module registry |
| Wed | Testing Terraform | Terratest, checkov, tflint, infracost |
| Thu | Drift & remediation | Detecting drift at scale, import, moved blocks |
| Fri | Policy as Code | OPA + Conftest for Terraform, Sentinel basics |
| Weekend | Practice | Build a modular AKS deployment with CI/CD validation |

**Deliverable:** [04-infrastructure-as-code/terraform/aks-module/](04-infrastructure-as-code/terraform/aks-module/)

---

### Week 7 — CI/CD & GitOps
**Goal:** Design and explain enterprise-grade CI/CD pipelines end-to-end.

| Day | Topic | Task |
|---|---|---|
| Mon | GitHub Actions advanced | Reusable workflows, composite actions, OIDC auth to Azure |
| Tue | GitLab CI advanced | Pipeline includes, DAG, review apps, runners at scale |
| Wed | ArgoCD / Flux | App of Apps pattern, sync waves, rollback, multi-cluster |
| Thu | Release strategies | Blue/green, canary with Argo Rollouts, feature flags |
| Fri | Pipeline security | SAST/DAST/SCA integration, secret scanning, SBOM generation |
| Weekend | Practice | Draw a "secure supply chain" pipeline end-to-end diagram |

**Deliverable:** [05-cicd-pipelines/README.md](05-cicd-pipelines/README.md)

---

### Week 8 — Observability
**Goal:** Design and talk through full-stack observability for production systems.

| Day | Topic | Task |
|---|---|---|
| Mon | Prometheus deep dive | Scrape config, service discovery, TSDB, cardinality issues |
| Tue | PromQL | Write 20 production-useful queries, alerting rules |
| Wed | Distributed tracing | OpenTelemetry collector, Tempo/Jaeger, trace propagation |
| Thu | Log aggregation | Loki + Promtail, ELK vs Loki trade-offs, Fluent Bit config |
| Fri | SLO/SLI/Error Budget | Define SLOs for 3 real services, write error budget policy |
| Weekend | Practice | Build a Grafana dashboard spec for a K8s cluster |

**Deliverable:** [06-observability/slo-sli-sla/](06-observability/slo-sli-sla/)

---

## Phase 3 – Weeks 9–12: Confidence Gaps

### Week 9 — Security & DevSecOps
**Goal:** Speak fluently about supply chain, secrets, and runtime security.

| Day | Topic | Task |
|---|---|---|
| Mon | Supply chain | SBOM (CycloneDX/SPDX), Sigstore/Cosign image signing |
| Tue | Container scanning | Trivy in CI, Grype, base image hardening |
| Wed | Secrets management | HashiCorp Vault: dynamic secrets, PKI, K8s auth method |
| Thu | Runtime security | Falco rules, seccomp profiles, AppArmor, Pod Security Standards |
| Fri | Compliance | SOC2 controls mapped to DevOps tooling, audit log design |
| Weekend | Practice | Threat model a CI/CD pipeline using STRIDE |

**Deliverable:** [08-security-devsecops/threat-models/cicd-pipeline.md](08-security-devsecops/threat-models/cicd-pipeline.md)

---

### Week 10 — Scripting & Coding Rounds
**Goal:** Pass the coding screen. DevOps companies do test coding at senior level.

| Day | Topic | Task |
|---|---|---|
| Mon | Python DSA foundations | Arrays, hashmaps, two pointers — 5 problems on NeetCode |
| Tue | Python DSA continued | Sliding window, stacks/queues — 5 problems |
| Wed | Python automation | Write K8s health-check script using client library |
| Thu | Bash scripting | Write log-rotation, disk-alert, and deployment-verify scripts |
| Fri | Live coding simulation | 2 timed problems (45 min each) |
| Weekend | Review | Refactor scripts, add error handling, write tests |

**Deliverable:** [09-scripting-coding/](09-scripting-coding/)

---

### Week 11 — Cost Optimization & FinOps + Advanced K8s
**Goal:** Show strategic thinking on cost, not just technical depth.

| Day | Topic | Task |
|---|---|---|
| Mon | K8s resource tuning | VPA, Goldilocks, right-sizing requests/limits at scale |
| Tue | Spot/Preemptible strategy | Cluster autoscaler with spot, karpenter on AKS/EKS |
| Wed | FinOps principles | Cloud cost allocation, chargeback vs showback, tagging strategy |
| Thu | Idle resource cleanup | Automation patterns, cost anomaly detection |
| Fri | Practice | Design a FinOps governance model for a platform team |
| Weekend | Review | Update SKILLS.md with current levels |

---

### Week 12 — Behavioral Stories (STAR)
**Goal:** Have 15–20 polished, senior-level stories ready to fire in any direction.

| Day | Topic | Task |
|---|---|---|
| Mon | Story mining | List 30 real projects / incidents from your career |
| Tue | STAR format | Write 5 stories: technical leadership, complexity, ambiguity |
| Wed | STAR format | Write 5 stories: conflict, failure, influencing without authority |
| Thu | STAR format | Write 5 stories: delivery under pressure, reliability, cost impact |
| Fri | Practice | Record yourself answering 5 questions — watch back |
| Weekend | Polish | Trim each story to < 3 mins delivery |

**Deliverable:** [10-behavioral/star-stories/](10-behavioral/star-stories/)

---

## Phase 4 – Weeks 13–16: Execution

### Week 13 — Mock Interviews Round 1
- 2× system design mock interviews (self-timed, write up feedback after)
- 1× coding round simulation
- 1× behavioral round simulation

### Week 14 — Company-Specific Prep
- Research top 5 target companies: engineering blogs, tech stack, recent incidents
- Write company-specific notes in [11-mock-interviews/company-specific/](11-mock-interviews/company-specific/)
- Map each company's known interview format to your prep

### Week 15 — Mock Interviews Round 2 + Gap Fill
- Run mocks on identified weak areas
- Revisit lowest-rated SKILLS.md items
- Practice whiteboarding system design out loud

### Week 16 — Final Polish
- Review all case studies and runbooks
- Final pass on behavioral stories
- Logistics: resume, target list, referrals
- **You're ready.**

---

## Weekly Cadence Template

```
Monday    → Study new concept (1.5h)
Tuesday   → Deep dive / read primary source (1.5h)
Wednesday → Hands-on / lab / code (2h)
Thursday  → Write notes / case study / runbook (1.5h)
Friday    → Practice problem or mock question (1.5h)
Weekend   → Review + update SKILLS.md + plan next week (2h)
```

---

## Progress Log

| Week | Dates | Focus | Completed? | Notes |
|---|---|---|---|---|
| 1 | Apr 7–13 | System Design Fundamentals | ⬜ | |
| 2 | Apr 14–20 | System Design: Platform & SRE | ⬜ | |
| 3 | Apr 21–27 | Kubernetes Internals | ⬜ | |
| 4 | Apr 28–May 4 | K8s Troubleshooting + Networking | ⬜ | |
| 5 | May 5–11 | Azure Deep Dive | ⬜ | |
| 6 | May 12–18 | Terraform | ⬜ | |
| 7 | May 19–25 | CI/CD & GitOps | ⬜ | |
| 8 | May 26–Jun 1 | Observability | ⬜ | |
| 9 | Jun 2–8 | Security & DevSecOps | ⬜ | |
| 10 | Jun 9–15 | Scripting & Coding Rounds | ⬜ | |
| 11 | Jun 16–22 | FinOps + Advanced K8s | ⬜ | |
| 12 | Jun 23–29 | Behavioral Stories | ⬜ | |
| 13 | Jun 30–Jul 6 | Mock Interviews Round 1 | ⬜ | |
| 14 | Jul 7–13 | Company-Specific Prep | ⬜ | |
| 15 | Jul 14–20 | Mock Interviews Round 2 | ⬜ | |
| 16 | Jul 21–27 | Final Polish | ⬜ | |
