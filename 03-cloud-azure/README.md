# Cloud – Azure Deep Dive

## Folder Structure

```
03-cloud-azure/
├── networking/         # VNet, NSG, Private Endpoints, ExpressRoute
├── security/           # Azure Policy, Defender, Key Vault, Managed Identity
├── cost-optimization/  # Budgets, tagging, right-sizing
├── aks/                # AKS-specific deep dive
└── architecture/       # Landing Zone, CAF patterns
```

## Coverage Checklist

### Networking
- [ ] VNet fundamentals: subnets, CIDR planning, peering (regional + global)
- [ ] NSG: rules, priorities, effective security rules, flow logs
- [ ] Azure Firewall vs NVA vs NSG — when to use which
- [ ] Private Endpoints vs Service Endpoints — key differences
- [ ] Private DNS Zones: integration with private endpoints, auto-registration
- [ ] ExpressRoute vs VPN Gateway — trade-offs (latency, cost, SLA)
- [ ] Azure Front Door vs Application Gateway vs Traffic Manager
- [ ] Hub-Spoke topology: shared services, peering limits, routing

### AKS
- [ ] Node pool types: system vs user, spot, ARM64
- [ ] CNI modes: Kubenet vs Azure CNI vs Azure CNI Overlay vs Cilium
- [ ] Workload Identity: federated credentials, service account token projection
- [ ] KEDA on AKS: Azure Queue, Event Hub, Service Bus scalers
- [ ] AKS cluster upgrade strategy: node image, control plane, blue-green
- [ ] Egress control: UDR + Azure Firewall, private cluster
- [ ] Azure Monitor + Container Insights: metrics, logs, prometheus scraping
- [ ] Open Service Mesh / Istio add-on on AKS

### Security
- [ ] Azure AD / Entra ID: service principals vs managed identities
- [ ] Key Vault: access policies vs RBAC, soft delete, purge protection
- [ ] CSI Secrets Store driver: sync as Kubernetes secrets
- [ ] External Secrets Operator with Azure Key Vault
- [ ] Azure Policy: built-in initiative, custom policy, deny vs audit vs modify
- [ ] Microsoft Defender for Cloud: secure score, recommendations, CSPM
- [ ] Microsoft Defender for Containers: vulnerability assessment, runtime

### Cost Optimization
- [ ] Cost Management + Billing: budgets, alerts, cost analysis
- [ ] Azure Advisor: cost recommendations
- [ ] Reserved Instances vs Savings Plans vs Spot VMs
- [ ] AKS cost: right-sizing node pools, spot node pools, scale to zero
- [ ] Tagging strategy: mandatory tags, Azure Policy enforcement

## Resources

- [Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/)
- [Azure Landing Zones (CAF)](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/)
- [AKS Best Practices](https://learn.microsoft.com/en-us/azure/aks/best-practices)
- [Azure Security Benchmark](https://learn.microsoft.com/en-us/security/benchmark/azure/)
