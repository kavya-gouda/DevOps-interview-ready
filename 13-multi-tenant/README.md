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

---
