# Multi-Tenant Amazon EKS Reference Architecture
**Dev / QA / Prod Isolation with No Noisy Neighbors**

## Overview

This repository documents a **reference architecture** for running **Dev, QA, and Prod workloads on a single Amazon EKS cluster** while maintaining:

- Strict environment isolation
- Predictable performance for Production
- Strong security boundaries
- Scalable platform operations

The design targets **enterprise DevOps / DevSecOps environments** where a shared Kubernetes control plane is required, but **runtime, network, and access isolation cannot be compromised**.

---

## Design Principles

1. **Isolation is layered, not singular**
   - Namespaces alone are insufficient
2. **Production must be protected from lower environments**
3. **Security and isolation are enforced by default**
4. **Automation over manual guardrails**
5. **Progressive evolution toward multi-cluster if needed**

---

## High-Level Architecture

Amazon EKS Control Plane (Shared)
│
├── Managed Node Group: Dev
│   └── Namespaces: dev/*
│
├── Managed Node Group: QA
│   └── Namespaces: qa/*
│
└── Managed Node Group: Prod
└── Namespaces: prod/*

The EKS control plane is shared, but **compute, network, IAM, and policy boundaries are environment-specific**.
## Control Plane

- Managed by **Amazon EKS**
- Platform team owns:
  - Cluster lifecycle
  - Admission controllers
  - RBAC bootstrap
  - Audit logging
- Application teams have **namespace-scoped access only**

> NOTE: This design assumes trust in the centralized control plane.  
> If regulatory or compliance requirements disallow this, separate clusters are recommended.

---

## Compute Isolation (No Noisy Neighbors)

### Dedicated Managed Node Groups

Each environment runs on **its own node group**:

| Environment | Instance Type Example | Purpose |
|-----------|----------------------|--------|
| Dev       | m6a.large           | Cost-efficient, bursty |
| QA        | m6i.large           | Stable performance |
| Prod      | c6i.xlarge          | Predictable latency |

### Scheduling Enforcement

- Nodes labeled by environment
- Taints applied to nodes
- Pods use:
  - `nodeSelector`
  - `tolerations`

This guarantees:
- Dev/QA workloads **can never run on Prod nodes**
- CPU and memory contention is eliminated across environments

---

## Namespace Strategy

Namespaces represent **policy and ownership boundaries**, not performance boundaries.

dev/
qa/
prod/

Each namespace has:
- Independent RBAC
- ResourceQuotas
- NetworkPolicies
- Pod security constraints

---

## Resource Management

### ResourceQuota

Applied per namespace to prevent runaway usage:

- CPU
- Memory
- Pod count
- Storage

### Quality of Service (QoS)

- **Production workloads use Guaranteed QoS**
- Requests == limits
- Ensures kernel-level prioritization and stable latency

---

## Network Isolation

### VPC CNI with Security Groups for Pods

- Each environment has **dedicated security groups**
- Prod pods:
  - Receive stricter ingress/egress rules
  - Are unreachable from Dev/QA by default
- Enforcement happens at **AWS networking layer (L3/L4)**

### Kubernetes NetworkPolicies

- Restrict pod-to-pod communication
- Enforce zero-trust inside the cluster
- Used as a second layer of defense

---

## IAM & Cloud Resource Isolation

### IAM Roles for Service Accounts (IRSA)

- Separate IAM roles per environment
- No cross-environment AWS API access
- Least-privilege permissions enforced

Examples:
- `eks-dev-app-role`
- `eks-qa-app-role`
- `eks-prod-app-role`

### Storage Isolation

- Separate EBS StorageClasses per environment
- Production volumes:
  - Encrypted with dedicated KMS keys
- PVCs scoped to namespaces

---

## Policy Enforcement (Admission Control)

Implemented using **OPA Gatekeeper or Kyverno**.

### Enforced Policies

- No `:latest` images
- Mandatory resource requests and limits
- Non-root containers
- Approved container registries
- PodDisruptionBudgets for Prod
- Environment-specific constraints

Policies are enforced **before workloads are admitted**, preventing misconfiguration rather than detecting it later.

---

## CI/CD & Release Strategy

### Deployment Model

| Environment | Deployment Strategy |
|-----------|---------------------|
| Dev       | Continuous deployment |
| QA        | Automated with approval |
| Prod      | Manual promotion only |

- Artifacts are **promoted, not rebuilt**
- GitOps recommended (Argo CD / Flux)
- Eliminates environment drift

---

## Observability & Blast Radius Control

### Telemetry Isolation

- Separate log streams for Dev, QA, and Prod
- Production logs:
  - Higher retention
  - SIEM integration
- Metrics segmented by namespace/environment

### Alert Routing

- Dev/QA: Chat-based notifications
- Prod: Incident management systems (PagerDuty, OpsGenie)

Dev noise never hides Production incidents.

---

## Limitations & Trade-offs

This design does **NOT** fully protect against:

- Kubernetes control-plane compromise
- etcd-level failures
- Cluster-wide misconfiguration

For:
- Regulated workloads (PCI, HIPAA)
- Strict compliance requirements
- Ultra-low-latency SLOs

👉 **Separate clusters per environment are recommended**

---

## When to Use This Architecture

✅ Platform Engineering teams  
✅ Large organizations with multiple product teams  
✅ Cost-conscious environments  
✅ Strong DevSecOps maturity  

---

## Summary

This reference architecture delivers:

- Strong environment isolation
- Zero noisy neighbors for Production
- Centralized platform operations
- Security by default, not by exception

This is **enterprise-grade EKS multi-tenancy**, not namespace-only segmentation.

---

## Next Steps (Optional Enhancements)

- Multi-cluster federation
- Cluster per environment evolution
- Service mesh integration
- Automated cost allocation
- Progressive Zero Trust enforcement

------

# Managing 10+ Kustomize Overlays Without Drift or Duplication


The goal is to:
- Avoid YAML duplication
- Prevent configuration drift
- Keep overlays thin and intentional
- Enforce consistency through structure and policy
- Scale as teams and environments grow

This approach is designed for **platform engineering and DevOps teams** operating Kubernetes at scale.


## Core Principles

1. **Overlays must be thin**
2. **Shared behavior belongs in the base or components**
3. **Overlays never redefine full resources**
4. **Rendered output is the source of truth**
5. **Policy enforces rules, not human discipline**

If overlays start copying manifests, the design is already failing.

## Repository Structure

```
.
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── hpa.yaml
│   ├── pdb.yaml
│   └── kustomization.yaml
│
├── components/
│   ├── autoscaling/
│   │   └── kustomization.yaml
│   ├── logging/
│   │   └── kustomization.yaml
│   ├── tracing/
│   │   └── kustomization.yaml
│   ├── pod-security/
│   │   └── kustomization.yaml
│   └── ingress/
│       └── kustomization.yaml
│
├── overlays/
│   ├── dev/
│   │   ├── kustomization.yaml
│   │   └── patches.yaml
│   ├── qa/
│   │   ├── kustomization.yaml
│   │   └── patches.yaml
│   ├── staging/
│   │   ├── kustomization.yaml
│   │   └── patches.yaml
│   └── prod/
│       ├── kustomization.yaml
│       └── patches.yaml
│
├── policies/
│   ├── gatekeeper/
│   └── kyverno/
│
└── README.md
```
## Base Layer (`/base`)

The base defines **what the application is**.

### Base owns:
- Resource definitions (Deployment, Service, HPA, PDB)
- Labels and selectors
- Probes
- Pod security context
- Default resource structure

### Base must NOT:
- Contain environment-specific values
- Reference environment-specific configs
- Be copied or forked

> If multiple bases exist for the same application, drift is guaranteed.

Components define **optional, reusable behaviors**.

Examples:
- Autoscaling policies
- Logging sidecars
- Tracing integration
- Pod security defaults
- Ingress configuration

Components allow behavior reuse without duplication.

### Example component usage:

```yaml
components:
  - ../../components/autoscaling
  - ../../components/logging
```
✅ Change once
✅ Applied to multiple environments
✅ No copy-paste overlays

### Overlays (/overlays)

Overlays define environment-specific intent, nothing more.
Allowed changes in overlays:
   - Replica counts
   - Resource requests / limits
   - Image tags or digests
   - Environment-specific ConfigMaps/Secrets
   - Ingress hostnames
   - Feature flags

Forbidden in overlays:

  - Full resource definitions
  - New Kubernetes objects
  - Selector changes
  - Security context modifications
  - ServiceAccount changes

If an overlay needs more power, the base or a component is incorrect.

## Patch-Only Overlays

Overlays use strategic merge or JSON patches only.

Example: replica change
```
patches:
  - target:
      kind: Deployment
      name: app
    patch: |
      - op: replace
        path: /spec/replicas
        value: 5

```
Why this matters:
   
-   Base changes propagate automatically
-   Structural drift is impossible
-   Reviews focus on intent, not YAML shape

Drift Prevention Strategy

Drift is detected at render time, not review time.

Every change must pass:
```
kustomize build overlays/<env>
```
Rendered output is:

-   Diffed against previous state
-   Compared across environments when needed
-   Validated by policy engines

Drift that never reaches the cluster is not drift.

## Policy Enforcement (/policies)
Discipline is enforced with policy-as-code, not trust.
Enforced rules:

- Overlays cannot introduce new resources
- Production requires requests and limits
- Non-root containers are mandatory
- :latest images are forbidden
- Approved registries only
- PDBs required for production

Violations fail CI/CD before deployment.

## Environment Ownership Model

Environments own overlays
Teams contribute to base and components
✅ Prevents forked realities
✅ Promotes platform consistency
✅ Reduces long-term maintenance cost
This is critical once you exceed 5–6 teams.

## Cross-Environment Drift Checks

Periodic validation:
```
kustomize build overlays/qa > qa.yaml
kustomize build overlays/prod > prod.yaml
diff qa.yaml prod.yaml
```
Only expected differences (replicas, image tags, hosts) are allowed.
Unexpected diffs require investigation.


## When This Approach Stops Scaling
Re-evaluate if:

- Overlays exceed ~15
- Per-tenant customization is needed
- Runtime parameters become dynamic
- Environments require legal or regulatory isolation

At that point:

- Introduce multiple clusters
- Or combine Helm + Kustomize
- Or adopt environment promotion pipelines
----
