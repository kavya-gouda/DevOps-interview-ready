# Skills Tracker

Rate yourself honestly. Revisit this file every 2 weeks and update levels.

**Scale:** 1 = Beginner · 2 = Aware · 3 = Working knowledge · 4 = Solid · 5 = Expert / Can teach

---

## 1. System Design & Architecture ⭐

| Topic | Current | Target | Priority | Status |
|---|:---:|:---:|---|---|
| Distributed systems fundamentals (CAP, BASE, consistency models) | 3 | 5 | High | 🔴 |
| Designing highly available, fault-tolerant systems | 3 | 5 | High | 🔴 |
| Microservices vs monolith trade-offs | 4 | 5 | Medium | 🔴 |
| Event-driven architecture (Kafka, queues) | 3 | 5 | High | 🔴 |
| Scaling patterns (horizontal, vertical, sharding, caching) | 3 | 5 | High | 🔴 |
| Multi-region / multi-AZ deployment strategies | 3 | 5 | High | 🔴 |
| API gateway, service mesh design | 3 | 5 | Medium | 🔴 |
| Data pipelines & streaming architectures | 2 | 4 | Medium | 🔴 |

**Resources:**
- [ ] [Awesome Scalability](https://github.com/binhnguyennus/awesome-scalability)
- [ ] *Designing Data-Intensive Applications* – Martin Kleppmann
- [ ] ByteByteGo System Design newsletter / videos

---

## 2. Kubernetes ⭐

| Topic | Current | Target | Priority | Status |
|---|:---:|:---:|---|---|
| Control plane components (kube-apiserver, etcd, scheduler, controller-manager) | 4 | 5 | High | 🔴 |
| Pod lifecycle, scheduling, eviction | 3 | 5 | High | 🔴 |
| Networking (CNI, kube-proxy, iptables vs eBPF) | 3 | 5 | High | 🔴 |
| Storage (PV, PVC, StorageClass, CSI) | 3 | 5 | Medium | 🔴 |
| RBAC, PSA / OPA / Kyverno | 3 | 5 | High | 🔴 |
| Helm, Kustomize | 4 | 5 | Medium | 🔴 |
| Cluster autoscaler, HPA, VPA, KEDA | 3 | 5 | High | 🔴 |
| Multi-tenancy patterns | 3 | 4 | Medium | 🔴 |
| AKS / EKS specifics | 4 | 5 | High | 🔴 |
| Troubleshooting (CrashLoopBackOff, OOMKilled, pending pods) | 4 | 5 | High | 🔴 |
| GitOps with ArgoCD / Flux | 3 | 5 | High | 🔴 |
| Service mesh (Istio / Linkerd basics) | 2 | 4 | Medium | 🔴 |

**Resources:**
- [ ] [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way)
- [ ] [Awesome Cloud Native Trainings](https://github.com/joseadanof/awesome-cloudnative-trainings)
- [ ] Killer.sh CKA/CKS simulators

---

## 3. Cloud – Azure

| Topic | Current | Target | Priority | Status |
|---|:---:|:---:|---|---|
| Networking (VNet, NSG, Private Endpoints, ExpressRoute) | 4 | 5 | High | 🔴 |
| AKS deep dive | 4 | 5 | High | 🔴 |
| Azure DevOps pipelines | 4 | 5 | Medium | 🔴 |
| Azure Monitor, Log Analytics, Application Insights | 4 | 5 | Medium | 🔴 |
| Azure Key Vault & Managed Identity | 4 | 5 | High | 🔴 |
| Azure Policy & Blueprints | 3 | 5 | High | 🔴 |
| Azure cost management & FinOps | 3 | 5 | High | 🔴 |
| Azure Landing Zone / CAF | 3 | 5 | Medium | 🔴 |

---

## 4. Infrastructure as Code (Terraform)

| Topic | Current | Target | Priority | Status |
|---|:---:|:---:|---|---|
| Terraform state management (remote state, locking) | 4 | 5 | High | 🔴 |
| Module design, versioning | 4 | 5 | High | 🔴 |
| Workspaces vs directory-per-env pattern | 3 | 5 | Medium | 🔴 |
| Terragrunt | 2 | 4 | Low | 🔴 |
| Drift detection & remediation | 3 | 5 | High | 🔴 |
| Terraform testing (Terratest, checkov, tflint) | 3 | 5 | High | 🔴 |
| Policy as Code with Sentinel / OPA | 2 | 4 | Medium | 🔴 |
| Pulumi / Crossplane awareness | 2 | 3 | Low | 🔴 |

---

## 5. CI/CD Pipelines

| Topic | Current | Target | Priority | Status |
|---|:---:|:---:|---|---|
| GitHub Actions (reusable workflows, composite actions) | 4 | 5 | High | 🔴 |
| GitLab CI (includes, templates, runners) | 4 | 5 | High | 🔴 |
| Jenkins (shared libraries, Jenkinsfile, pipelines as code) | 4 | 5 | Medium | 🔴 |
| GitOps deployment patterns (ArgoCD, Flux) | 3 | 5 | High | 🔴 |
| Container image build best practices (multi-stage, caching) | 4 | 5 | High | 🔴 |
| Release strategies (blue/green, canary, feature flags) | 3 | 5 | High | 🔴 |
| Pipeline security (SAST, DAST, SCA, secret scanning) | 3 | 5 | High | 🔴 |

---

## 6. Observability

| Topic | Current | Target | Priority | Status |
|---|:---:|:---:|---|---|
| Prometheus architecture (scrape, TSDB, PromQL) | 4 | 5 | High | 🔴 |
| Grafana dashboards, alerts | 4 | 5 | Medium | 🔴 |
| Datadog (APM, logs, synthetics) | 3 | 5 | Medium | 🔴 |
| Distributed tracing (OpenTelemetry, Jaeger/Tempo) | 3 | 5 | High | 🔴 |
| Log aggregation (Loki, ELK, Fluentd/Bit) | 3 | 5 | Medium | 🔴 |
| SLO / SLI / SLA definition and error budgets | 3 | 5 | High | 🔴 |
| On-call runbooks and alert fatigue reduction | 3 | 5 | Medium | 🔴 |

---

## 7. Networking ⭐

| Topic | Current | Target | Priority | Status |
|---|:---:|:---:|---|---|
| DNS (resolution, TTL, split-horizon, CoreDNS) | 3 | 5 | High | 🔴 |
| TLS/mTLS (handshake, cert management, rotation) | 3 | 5 | High | 🔴 |
| Load balancers (L4 vs L7, ALB, NLB, Nginx, Envoy) | 4 | 5 | High | 🔴 |
| TCP/IP fundamentals, subnetting | 3 | 5 | Medium | 🔴 |
| Ingress controllers, Gateway API | 3 | 5 | High | 🔴 |
| Network policies in Kubernetes | 3 | 5 | High | 🔴 |
| eBPF basics | 2 | 3 | Low | 🔴 |

---

## 8. Security & DevSecOps ⭐

| Topic | Current | Target | Priority | Status |
|---|:---:|:---:|---|---|
| Supply chain security (SBOM, Sigstore, Cosign) | 2 | 4 | High | 🔴 |
| Container image scanning (Trivy, Snyk, Grype) | 3 | 5 | High | 🔴 |
| Secrets management (Vault, SOPS, External Secrets Operator) | 3 | 5 | High | 🔴 |
| Zero-trust networking principles | 3 | 4 | Medium | 🔴 |
| SOC2 / ISO 27001 / PCI-DSS awareness | 2 | 4 | Medium | 🔴 |
| RBAC / IAM design patterns | 4 | 5 | High | 🔴 |
| Runtime security (Falco, seccomp, AppArmor) | 2 | 4 | High | 🔴 |
| Penetration testing / OWASP Top 10 | 2 | 4 | Medium | 🔴 |

---

## 9. Scripting & Coding ⭐

| Topic | Current | Target | Priority | Status |
|---|:---:|:---:|---|---|
| Python – automation & scripting | 4 | 5 | High | 🔴 |
| Python – data structures & algorithms (interview rounds) | 2 | 4 | High | 🔴 |
| Bash – advanced scripting, error handling | 4 | 5 | High | 🔴 |
| Python Kubernetes client (k8s SDK) | 3 | 5 | High | 🔴 |
| REST API automation (requests, FastAPI basics) | 3 | 4 | Medium | 🔴 |
| JSON/YAML parsing & jq | 4 | 5 | Medium | 🔴 |
| Concurrency basics (threading, asyncio) | 2 | 4 | Medium | 🔴 |

**Resources:**
- [ ] [NeetCode 150](https://neetcode.io/) – DSA for DevOps rounds
- [ ] [DevOps Exercises](https://github.com/bregman-arie/devops-exercises) – scripting section

---

## 10. Behavioral & Leadership ⭐

| Topic | Current | Target | Priority | Status |
|---|:---:|:---:|---|---|
| STAR format for senior-level stories | 3 | 5 | High | 🔴 |
| Conflict resolution & influencing without authority | 3 | 5 | High | 🔴 |
| Technical decision-making & trade-offs articulation | 3 | 5 | High | 🔴 |
| Driving reliability / SRE culture | 3 | 5 | Medium | 🔴 |
| Mentoring & team building stories | 3 | 5 | Medium | 🔴 |
| Incident post-mortem leadership | 4 | 5 | Medium | 🔴 |
| Stakeholder management & roadmap communication | 3 | 5 | Medium | 🔴 |

---

## 11. Cost Optimization & FinOps ⭐

| Topic | Current | Target | Priority | Status |
|---|:---:|:---:|---|---|
| Right-sizing compute & memory | 3 | 5 | High | 🔴 |
| Reserved Instances vs Spot / Preemptible strategy | 3 | 5 | High | 🔴 |
| Cloud cost allocation & tagging | 3 | 5 | Medium | 🔴 |
| Kubernetes resource requests/limits tuning | 3 | 5 | High | 🔴 |
| Idle resource detection & cleanup | 3 | 4 | Medium | 🔴 |

---

## Skill Level Summary (update bi-weekly)

| Date | Top 3 Gaps Addressed | Next Focus |
|---|---|---|
| 2026-04-07 | — | Starting System Design + K8s Internals |

