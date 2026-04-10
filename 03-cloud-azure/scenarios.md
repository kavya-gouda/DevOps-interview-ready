# Azure Cloud — 100 Scenarios

> **ID prefix:** A- | Types: Design, Implement, Troubleshoot, Explain | Difficulty: Medium → Expert

---

## Section 1: Azure Networking (A-001 to A-025)

---

### A-001 | Hub-Spoke VNet Topology Design
**Type:** Design | **Difficulty:** Hard | ⬜

> Design a Hub-Spoke VNet topology for a company with 3 business units, shared services (Azure Firewall, DNS, Bastion), and centralized egress control.

**What you must cover:**
- Hub VNet: Azure Firewall, Azure Bastion, VPN Gateway, Private DNS Resolver, shared AD
- Spoke VNets: one per BU or workload, peered to hub, no spoke-to-spoke peering (forces traffic through hub firewall)
- VNet Peering: hub↔spoke (non-transitive — spoke-to-spoke goes through hub firewall)
- Route table (UDR): spoke default route `0.0.0.0/0` → Azure Firewall private IP (all egress inspected)
- DNS: Private DNS Resolver in hub, conditional forwarders for on-premise
- Peering limits: VNet max 500 peerings — at scale, use Azure Virtual WAN instead
- Cost: Firewall at hub = one premium SKU services all spokes vs per-spoke NSG

---

### A-002 | Private Endpoints vs Service Endpoints
**Type:** Explain | **Difficulty:** Medium | ⬜

> Explain the difference between Private Endpoints and Service Endpoints. When must you use Private Endpoints?

**What you must cover:**
- Service Endpoint: no private IP, extends VNet identity to service, traffic still goes over Azure backbone to public IP, resource can still be accessed from other sources
- Private Endpoint: deploys a private NIC in your VNet with a private IP for the service, traffic stays entirely within VNet, completely blocks public access (when configured)
- Private Endpoint wins: regulatory requirement to block all public access, cross-region access, access from on-premise via ExpressRoute, approved by auditors for PCI/SOC2
- DNS: Private Endpoint requires private DNS zone integration (`privatelink.blob.core.windows.net`)
- Cost: Private Endpoint has hourly + data processing cost; Service Endpoint is free

---

### A-003 | NSG Rule Design and Effective Security Rules
**Type:** Implement | **Difficulty:** Medium | ⬜

> Design NSG rules for an AKS subnet that allows: ingress from Azure LB, egress to AKS control plane, egress to Azure services, and blocks everything else.

```
Inbound (allow):
- 100: Allow Azure LB → * port 80,443
- 200: Allow VNet → * (internal cluster traffic)
- 65000: Allow VNet (default)
- 65001: Allow Azure LB Health Probes
- 65500: Deny ALL (implicit)

Outbound (allow):
- 100: Allow * → AKS control plane (443, TCP) [use AKS service tag]
- 200: Allow * → Azure Storage, ACR (443) [AzureContainerRegistry, Storage service tags]
- 300: Allow * → Internet (443) [for package downloads — scope tightly]
- 65000: Allow VNet
- 65500: Deny ALL
```
- Service Tags: `AzureCloud`, `AzureKubernetesService` — managed IP ranges, auto-updated
- `az network nsg rule list --nsg-name <nsg>` to verify effective rules
- Effective rules: a subnet NSG + NIC NSG are both evaluated — most restrictive wins

---

### A-004 | ExpressRoute vs VPN Gateway
**Type:** Explain | **Difficulty:** Medium | ⬜

> Your company has a 10 Gbps on-premise data center and needs to connect to Azure. Make the case for ExpressRoute over VPN Gateway.

**What you must cover:**
- ExpressRoute: dedicated private circuit (not over internet), 50Mbps–100Gbps, SLA-backed, lower latency (<10ms)
- VPN Gateway: encrypted tunnel over internet, to 10Gbps (VpnGw5), dependent on internet latency
- ExpressRoute wins: predictable latency for database connections, high-throughput data transfer (>500Mbps), compliance that prohibits internet traversal
- VPN wins: cost (ER starts ~$55/month + circuit charges), quick setup, disaster recovery fallback
- ExpressRoute + VPN: ER as primary, VPN as failover — configure coexistence
- Global Reach: connect two offices via Azure backbone using ExpressRoute

---

### A-005 | Azure Front Door vs Application Gateway vs Traffic Manager
**Type:** Explain | **Difficulty:** Hard | ⬜

> Explain when to use Azure Front Door, Application Gateway, and Traffic Manager. Give a combined architecture.

**What you must cover:**
- Azure Front Door: global L7 LB, anycast edge PoPs, WAF, CDN, HTTP/2, WebSockets, multi-origin with health-based routing
- Application Gateway: regional L7 LB, WAF, URL routing, SSL offload, autoscale — within a single region
- Traffic Manager: DNS-based global routing (not a proxy!) — routing profiles: Priority, Weighted, Performance, Geographic
- Combined: Front Door (global L7 edge + CDN) → Application Gateway (regional WAF) → AKS Ingress → services
- Front Door health probes: detect failed origins and route away within seconds
- Traffic Manager latency: DNS TTL means failover isn't instant — AFD is better for active failover

---

### A-006 | Azure DNS Private Zones
**Type:** Implement | **Difficulty:** Medium | ⬜

> You have a private AKS cluster with a private endpoint to Azure Blob Storage. Walk through the DNS configuration so pods can resolve the storage endpoint.

```bash
# Private endpoint for storage
az network private-endpoint create \
  --name blob-pe \
  --resource-group myrg \
  --vnet-name myvnet \
  --subnet pe-subnet \
  --private-connection-resource-id $(az storage account show -n mysa -g myrg --query id -o tsv) \
  --group-ids blob

# Private DNS zone
az network private-dns zone create -g myrg -n "privatelink.blob.core.windows.net"
az network private-dns link vnet create -g myrg -n mylink -z "privatelink.blob.core.windows.net" \
  --virtual-network myvnet --registration-enabled false

# DNS record: pe-nic-ip → mystorage.privatelink.blob.core.windows.net
# Public DNS: mystorage.blob.core.windows.net → mystorage.privatelink.blob.core.windows.net (CNAME)
# Private DNS: mystorage.privatelink.blob.core.windows.net → 10.0.8.4 (PE NIC ip)
```
- CoreDNS on AKS: forward DNS for `blob.core.windows.net` through Azure DNS (168.63.129.16 — this is Azure's internal DNS)
- AKS private cluster: requires Private DNS zone for API server (`privatelink.{region}.azmk8s.io`)

---

### A-007 | DDoS Protection Strategy
**Type:** Design | **Difficulty:** Medium | ⬜

> Design Azure DDoS protection for a customer-facing platform. Compare DDoS Network Protection vs IP Protection vs WAF for different attack vectors.

**What you must cover:**
- Azure DDoS Network Protection: standard plan (~$2,944/month), VNet-level protection, adaptive tuning, telemetry, rapid response team
- DDoS IP Protection: per-public-IP plan ($200/IP/month), more targeted
- WAF (Application Gateway / Front Door): L7 protection (SQL injection, OWASP Top 10, rate limiting) — complement to DDoS
- Mitigation: volumetric attacks → DDoS Network Protection; application attacks → WAF; protocol attacks → NSG + Azure Firewall
- Monitoring: Azure Monitor DDoS metrics (`IfUnderDDoSAttack`), Alert on attack start/end
- SLA: DDoS Protection guarantee: credits if attack gets through after mitigation

---

### A-008 | Azure Firewall Policy Design
**Type:** Design | **Difficulty:** Hard | ⬜

> Design Azure Firewall policy for a hub-spoke topology with: outbound internet control, inbound port forwarding for AKS, and FQDN-based allow lists.

**What you must cover:**
- Rule collection priority: DNAT (lowest number = highest priority) → Network → Application
- DNAT rules: map public IP:port → AKS Ingress private IP (for inbound traffic)
- Network rules: allow AKS → AKS control plane (`AzureKubernetesService` service tag :443)
- Application rules with FQDN: allow pods → `*.docker.io`, `mcr.microsoft.com`, `*.azurecr.io`
- Threat Intelligence: enable block mode — known malicious IPs/FQDNs auto-blocked
- Policy hierarchy: base policy (org-wide baseline) → child policies per spoke (additional rules)
- UDR: spoke subnets have `0.0.0.0/0 → firewall-private-ip` to force all egress through firewall

---

### A-009 | Azure Load Balancer vs Application Gateway
**Type:** Explain | **Difficulty:** Medium | ⬜

> Explain when AKS should use Azure LB (LoadBalancer service type) vs Application Gateway Ingress Controller (AGIC).

**What you must cover:**
- Azure LB (`type: LoadBalancer`): L4, TCP/UDP, per-service LB, used by AKS automatically for service type LB — fast, simple
- AGIC: L7 HTTP/HTTPS, path-based routing, WAF, SSL termination at Azure boundary (before traffic enters VNet)
- AGIC advantage: SSL terminates at Application Gateway (TLS offload before cluster), WAF at gateway, single IP for multiple services
- AGIC disadvantage: configuration via Ingress annotations only, slower updates than nginx-ingress, AGIC must be in same VNet
- Recommendation: NGINX Ingress for flexibility + small clusters; AGIC for regulatory environments needing WAF at perimeter

---

### A-010 | Azure Virtual WAN
**Type:** Explain | **Difficulty:** Hard | ⬜

> When would you move from Hub-Spoke VNet peering to Azure Virtual WAN? What does vWAN simplify?

**What you must cover:**
- Hub-Spoke limit: 500 VNet peerings max per VNet — large enterprises hit this
- Virtual WAN: Microsoft-managed hub routers, hub-to-hub routing built-in, transitive routing (spoke-to-spoke via WAN hub)
- Simplification: no manual peering, no UDR management (automated), global connected hubs
- Secure Virtual Hub: Azure Firewall deployed inside vWAN hub — integrated security
- SD-WAN integration: branch offices connect via SDWAN NVA in vWAN
- Cost: vWAN has per-connection pricing — may cost more than simple peering for small deployments
- When to choose: 50+ spokes, global regions, branch-to-branch connectivity requirements

---

### A-011 to A-025 | Azure Networking Rapid-Fire

---

### A-011 | Explain Azure Private Resolver. How does it enable DNS forwarding between Azure and on-premise without deploying DNS VMs? | Explain | Hard | ⬜
**Hint:** Inbound endpoint (DNS queries from on-prem to Azure private zones), outbound endpoint (DNS queries from Azure to on-prem DNS). Ruleset per domain.

### A-012 | Design subnet sizing for an AKS cluster with 1000 pods, Azure CNI. How many IPs do you need? | Design | Hard | ⬜
**Hint:** Azure CNI: 1 IP per pod + 1 per node. 1000 pods + N nodes. Subnet must accommodate node scale too. Use /18 or larger.

### A-013 | A virtual machine can't reach the internet but NSG rules allow outbound. What else could be blocking? | Troubleshoot | Hard | ⬜
**Hint:** UDR override (route to firewall that's blocking), Azure Firewall policy denying FQDN, missing/expired NAT gateway, NVA in the path failing.

### A-014 | Explain Azure Bastion tiers (Basic vs Standard) and when you need Standard. | Explain | Medium | ⬜
**Hint:** Standard: native client support (SSH/RDP via local client), copy-paste, tunneling, shareable link. Basic: browser only.

### A-015 | Design a network architecture for an AKS cluster that must not have any public IP addresses (fully private). | Design | Expert | ⬜
**Hint:** Private cluster (API server), private ACR, private endpoints for all Azure dependencies, AzureFirewall for egress, no public LoadBalancer services.

### A-016 | ExpressRoute circuit shows connected but VMs can't communicate cross-circuit. What's missing? | Troubleshoot | Hard | ⬜
**Hint:** Global Reach required for circuit-to-circuit (office-to-office via Azure). Route filters not configured. BGP route not propagated.

### A-017 | Explain BGP route propagation in Azure. When does disabling it matter? | Explain | Hard | ⬜
**Hint:** Gateway route propagation: when enabled, VPN/ER gateway routes are added to VNet route table. Disable to prevent on-prem routes from overriding custom UDRs.

### A-018 | A subnet's effective routes show 0.0.0.0/0 next-hop is VNetLocal but traffic isn't reaching the internet. | Troubleshoot | Hard | ⬜
**Hint:** Missing NAT Gateway or public IP on subnet/VM. Azure doesn't automatically provide SNAT for all subnets — requires explicit configuration.

### A-019 | Design a cross-region failover for an AKS cluster using Azure Traffic Manager. | Design | Expert | ⬜
**Hint:** Traffic Manager DNS-based routing, Priority profile, health endpoint probe per region, TTL implications for failover speed.

### A-020 | Explain Azure Service Tags and how you use them in NSG rules to simplify management. | Explain | Medium | ⬜
**Hint:** Managed IP groups by Microsoft (e.g., `AzureKubernetesService`, `Storage.EastUS`). Auto-updated. Reference in NSG rule instead of maintaining IP lists.

### A-021 | An AKS cluster can't pull images from ACR despite the right ACR attachment. Diagnose. | Troubleshoot | Hard | ⬜
**Hint:** Node managed identity doesn't have AcrPull role. ACR uses Private Endpoint but cluster VNet isn't linked to Private DNS zone. Network policy blocking HTTPS.

### A-022 | Design a multi-region active-passive failover using Azure Front Door and paired AKS clusters. | Design | Expert | ⬜
**Hint:** AFD origin groups with priority. Health probe to `/health` endpoint. Database: Azure SQL geo-replication, RTO < 30 min, RPO < 5 min.

### A-023 | Explain the difference between Azure Firewall Standard and Premium. When do you need Premium? | Explain | Medium | ⬜
**Hint:** Premium: TLS inspection (decrypt + inspect, then re-encrypt), IDPS (Intrusion Detection), URL-filtering. Required for deep packet inspection compliance requirements.

### A-024 | How do you diagnose and fix asymmetric routing in a hub-spoke topology? | Troubleshoot | Expert | ⬜
**Hint:** All traffic must flow through firewall symmetrically (same path in and out). Asymmetric = firewall sees only one direction → drops TCP. Ensure return traffic UDR also points to firewall.

### A-025 | Explain Azure vNet Flow Logs and NSG Flow Logs. How do you use them for security investigation? | Explain | Medium | ⬜
**Hint:** NSG Flow Logs: per-rule, per-flow tuple (src/dst IP, port, protocol, allow/deny). Enable → Storage Account → Traffic Analytics → Log Analytics KQL queries.

---

## Section 2: AKS Deep Dive (A-026 to A-050)

---

### A-026 | AKS Cluster Architecture Design
**Type:** Design | **Difficulty:** Expert | ⬜

> Design a production AKS cluster for a fintech company: 99.99% availability, 10,000 pods, multi-AZ, PCI compliance.

**What you must cover:**
- Uptime SLA: 99.95% control plane (standard tier), system node pools across 3 AZs
- Node pools: system (Standard_D4s_v5, min 3 across 3 AZs), user (compute-optimized, autoscale)
- Networking: Azure CNI Overlay + Cilium, private cluster, Azure Firewall egress
- Security: AAD integration, Workload Identity, Defender for Containers, AzurePolicy add-on, Private ACR
- Storage: Premium SSD ZRS (zone-redundant) for stateful workloads
- Monitoring: Azure Monitor + managed Prometheus + Grafana
- PCI: pod security standards `restricted`, network encryption (Cilium), secret encryption at rest, audit log to Sentinel

---

### A-027 | AKS Node Pool Strategy
**Type:** Design | **Difficulty:** Hard | ⬜

> Design a multi-node pool strategy for: latency-critical API workloads, ML batch inference, and background workers with burst capacity.

**What you must cover:**
```bash
# System node pool (Kubernetes components only)
az aks nodepool add --name system --mode System \
  --node-vm-size Standard_D4s_v5 --node-count 3 --zones 1 2 3

# API node pool (latency-critical, memory-optimized)
az aks nodepool add --name api --mode User \
  --node-vm-size Standard_E8s_v5 --enable-cluster-autoscaler \
  --min-count 3 --max-count 20 --zones 1 2 3 \
  --node-taints "workload=api:NoSchedule"

# ML inference (GPU nodes)
az aks nodepool add --name gpu --mode User \
  --node-vm-size Standard_NC6s_v3 --enable-cluster-autoscaler \
  --min-count 0 --max-count 10 \
  --node-taints "sku=gpu:NoSchedule"

# Batch workers (spot nodes, burst, fault-tolerant)
az aks nodepool add --name spot --mode User \
  --node-vm-size Standard_D8as_v5 --priority Spot \
  --eviction-policy Delete --spot-max-price -1 \
  --enable-cluster-autoscaler --min-count 0 --max-count 100 \
  --node-taints "kubernetes.azure.com/scalesetpriority=spot:NoSchedule"
```

---

### A-028 | AKS Upgrade Planning
**Type:** Design | **Difficulty:** Hard | ⬜

> Design an AKS version upgrade plan for a cluster that's 2 minor versions behind. Define the sequence, pre-checks, and rollback plan.

**What you must cover:**
```bash
# Pre-checks
kubectl get nodes -o wide  # Current version
az aks get-upgrades --resource-group myrg --name myaks  # Available upgrades
kubectl get ingress,deployments -A -o yaml | pluto detect -  # Deprecated API check

# Step 1: Upgrade control plane
az aks upgrade --resource-group myrg --name myaks --kubernetes-version 1.30.0 --control-plane-only

# Step 2: Upgrade system node pool
az aks nodepool upgrade --resource-group myrg --cluster-name myaks \
  --name system --kubernetes-version 1.30.0

# Step 3: Upgrade user node pools (one at a time)
az aks nodepool upgrade --name api --kubernetes-version 1.30.0
```
- Rollback: control plane rollback not supported post-upgrade — upgrade dev first (dry run)
- Node pool upgrade: `maxSurge: 1` (default) — one extra node spun up, old node drained
- PDB check: `kubectl get pdb -A` — verify all critical workloads have PDB before draining nodes

---

### A-029 | Azure Monitor Container Insights vs Managed Prometheus
**Type:** Explain | **Difficulty:** Medium | ⬜

> Compare Container Insights and Managed Prometheus for AKS monitoring. When would you use each or both?

**What you must cover:**
- Container Insights: Log Analytics-based, pre-built dashboards, log collection (stdout/stderr), CMX metrics, no Prometheus knowledge needed
- Managed Prometheus: Prometheus-compatible TSDB in Azure Monitor, PromQL, Grafana integration, custom scrape config
- Container Insights wins: out-of-box experience, log correlation, Azure-native alerting without Prometheus expertise
- Managed Prometheus wins: full PromQL power, custom application metrics, Grafana dashboards, multi-cluster federation
- Both together: Managed Prometheus for metrics + Container Insights for logs — unified in Azure Monitor workspace
- Cost: Container Insights: per GB ingested into Log Analytics; Managed Prometheus: per metrics volume

---

### A-030 | AKS Networking Debugging
**Type:** Troubleshoot | **Difficulty:** Expert | ⬜

> A pod in AKS can reach the internet but cannot reach an internal Azure SQL private endpoint. Walk through debugging.

```bash
# Step 1: DNS resolution
kubectl exec -it debug -- nslookup mysqlserver.database.windows.net
# Expected: resolves to private IP (10.x.x.x)
# If resolves to public IP: Private DNS zone not linked to AKS VNet

# Step 2: Check Private DNS zone links
az network private-dns link vnet list --zone-name "privatelink.database.windows.net" -g myrg

# Step 3: Test TCP connectivity
kubectl exec -it debug -- nc -zv 10.0.8.4 1433

# Step 4: Check NSG on Private Endpoint subnet
az network nsg show -g myrg -n pe-nsg
# NSG on PE subnet: rule allowing inbound on 1433 from AKS subnet CIDR?

# Step 5: Check AKS VNet ↔ PE VNet peering (if different VNets)
az network vnet peering list --vnet-name aks-vnet -g myrg

# Step 6: Check Private Endpoint approval status
az network private-endpoint-connection list --type Microsoft.Sql/servers --name mysqlserver -g myrg
# Should be "Approved"
```

---

### A-031 to A-050 | AKS Rapid-Fire Scenarios

---

### A-031 | Explain AKS node OS disk types. When do you choose Ephemeral OS vs Managed OS disk? | Explain | Medium | ⬜
**Hint:** Ephemeral: uses VM local SSD (faster, no disk cost, OS data lost on node reimage). Managed: regular disk (slightly slower, more predictable). Ephemeral: stateless nodes, faster node add time.

### A-032 | AKS pod can't write to Azure Blob Storage. You see "Access denied" in logs. Walk through fix. | Troubleshoot | Hard | ⬜
**Hint:** Workload Identity: check federated credential subject, managed identity has Storage Blob Data Contributor, pod has `azure.workload.identity/use=true` label, SA has client-id annotation.

### A-033 | Design AKS cluster topology for dev/staging/prod with environment isolation. | Design | Hard | ⬜
**Hint:** Separate clusters per environment (not namespaces) — blast radius isolation. Shared ACR pull, separate VNets, ArgoCD multi-cluster management.

### A-034 | Explain AKS system node pool requirements. Why can you not remove the last system node pool? | Explain | Medium | ⬜
**Hint:** System node pool hosts: CoreDNS, metrics-server, tunnel-front/konnectivity-agent (for API server tunnel). Can't remove last one — cluster would lose critical components.

### A-035 | How do you enable Confidential Computing (CVM) nodes on AKS? Use case? | Explain | Hard | ⬜
**Hint:** `DCsv3` VM SKU with Intel SGX for enclaves; confidential containers; financial/healthcare workloads processing encrypted data in hardware-based enclaves.

### A-036 | AKS cluster was accidentally deleted. What can you recover and what's lost forever (without DR plan)? | Troubleshoot | Expert | ⬜
**Hint:** Lost: all resources deployed via kubectl (not GitOps), stateful data on Azure Disk (if ReclaimPolicy=Delete), etcd state. Recoverable: Azure managed resources (ACR images, Azure SQL) if separate, GitOps-managed state via reapply.

### A-037 | Implement Azure Policy add-on on AKS. What built-in policies would you apply for a CIS-hardened cluster? | Implement | Hard | ⬜
**Hint:** `Kubernetes cluster containers should not use forbidden sysctl interfaces`, `no privilege escalation`, `require readOnlyRootFilesystem`, `restrict host namespaces`.

### A-038 | Explain Open Service Mesh (now deprecated) vs Istio add-on on AKS. What's Microsoft's recommended mesh? | Explain | Medium | ⬜
**Hint:** OSM deprecated Jan 2024. Istio add-on (managed, tested against AKS versions, limited profiles). Cilium service mesh alternative. Microsoft recommends Istio add-on.

### A-039 | An AKS cluster's API server is experiencing high latency. How do you diagnose? | Troubleshoot | Expert | ⬜
**Hint:** etcd latency metrics (managed AKS via Container Insights), watch cache utilization, too many CRD objects, excessive LIST requests from controllers. `apiserver_request_duration_seconds`.

### A-040 | Design an AKS multi-tenancy model using Virtual Nodes (ACI) for burst capacity. | Design | Hard | ⬜
**Hint:** Virtual Kubelet + Azure Container Instances: schedule pods to ACI when cluster is full. Good for batch/burst. Limitations: no DaemonSets on virtual node, limited networking.

### A-041 | Explain AKS Start/Stop feature. Write automation to stop dev clusters at 6pm and start at 8am. | Implement | Medium | ⬜
**Hint:** `az aks stop/start`. GitHub Actions scheduled workflow or Azure Automation Runbook.

### A-042 | AKS Horizontal Pod Autoscaler is scaling but nodes are not being added. | Troubleshoot | Hard | ⬜
**Hint:** HPA scales replicas → pods become pending → CAS should add nodes. But: max node count reached, pod has constraints CAS can't meet, CAS backoff period.

### A-043 | Explain Azure Container Registry geo-replication. How does AKS pull from the nearest replica? | Explain | Hard | ⬜
**Hint:** ACR Geo-replication: automatic replication of images to secondary regions. AKS pulls from closest regional endpoint automatically (DNS-based). Reduces cross-region egress costs and latency.

### A-044 | Design a blue-green cluster upgrade strategy for production AKS (new cluster, not new node pool). | Design | Expert | ⬜
**Hint:** Provision new cluster with target K8s version, sync workloads via ArgoCD, test, switch DNS/traffic manager, decommission old cluster after validation window.

### A-045 | Implement AKS OIDC Issuer integration with an external OIDC provider (GitHub Actions). | Implement | Expert | ⬜
**Hint:** GitHub Actions OIDC → Azure federated credential → Service Principal or Managed Identity → no stored secrets in GitHub. `az ad app federated-credential create`.

### A-046 | Explain AKS Konnectivity agent. Why does the API server need to reach pods in private clusters? | Explain | Hard | ⬜
**Hint:** Konnectivity = tunnel from API server to cluster (for kubectl exec, log streaming, port-forward). Private cluster: API server peered via private endpoint, Konnectivity agent in cluster connects outbound to API server tunnel.

### A-047 | A node in AKS is showing "RegistrationFailed" status. How do you remediate? | Troubleshoot | Expert | ⬜
**Hint:** Custom script extension failure, cloud-init issue, disk quota exceeded during node bake, `az vmss get-instance-view`. Check VMSS Extension health.

### A-048 | Explain AKS Defender for Containers threat detection. Give 3 runtime alerts it would fire. | Explain | Medium | ⬜
**Hint:** Alerts: `Container running in privileged mode`, `Kubernetes service with external IP`, `Anomalous container executed command`. Integration with Sentinel for investigation.

### A-049 | How do you implement Azure role-based access to K8s resources using AAD groups? | Implement | Hard | ⬜
**Hint:** AKS AAD integration (`--enable-aad`, `--aad-admin-group-object-ids`). RoleBinding `subjects.kind: Group` with AAD group object ID. `kubelogin` for token acquisition.

### A-050 | AKS Spot node got preempted mid-job. What application patterns are necessary to handle this gracefully? | Design | Hard | ⬜
**Hint:** Checkpoint to Azure Blob every N minutes, idempotent job restart, `preStop` hook saves state, `PodDisruptionBudget` controls concurrent preemptions, `terminationGracePeriodSeconds: 30`.

---

## Section 3: Azure Security (A-051 to A-070)

---

### A-051 | Azure Key Vault — Access Pattern Design
**Type:** Design | **Difficulty:** Hard | ⬜

> Design a Key Vault access model for a company with 50 teams using AKS. Each team should access only their secrets.

**What you must cover:**
- One Key Vault per team vs shared: shared KV with RBAC scoped per secret prefix. Or per-team KV (better isolation, operational complexity)
- RBAC: `Key Vault Secrets User` assigned to team's managed identity, scoped to specific secrets via `--scope /subscriptions/.../secrets/<name>`
- ESO: ExternalSecret in team namespace points to their KV, uses WorkloadIdentity auth
- Audit: KV diagnostic logs → Log Analytics — who accessed what secret, when
- Network: Private Endpoint on KV, firewall to allow only cluster VNet + developer Bastion IP

---

### A-052 | Azure Managed Identity — Types and Patterns
**Type:** Explain | **Difficulty:** Medium | ⬜

> Explain System-assigned vs User-assigned Managed Identities. When must you use user-assigned?

**What you must cover:**
- System-assigned: tied to resource lifecycle (deleted when VM/AKS node deleted), one-to-one mapping
- User-assigned: independent resource, reusable across multiple workloads, persists after VM deletion
- AKS use case: AKS Workload Identity always uses user-assigned MI — controlled lifecycle, can be pre-authorized
- Share pattern: 100 pods that all need the same KV access → one user-assigned MI federated to 100 service accounts (or one SA)
- Avoid system-assigned in AKS: node pool scale-out creates new nodes with new system-assigned MIs → authorization lag

---

### A-053 | Microsoft Defender for Cloud — Security Score
**Type:** Design | **Difficulty:** Medium | ⬜

> Your Defender for Cloud secure score is 45%. Describe your approach to systematically improving it.

**What you must cover:**
- Prioritize by impact: Defender shows potential score increase per recommendation
- Quick wins: enable MFA enforcement, enable audit logging on all resources (usually high-impact, low-effort)
- Resource exemption: legitimate exceptions (dev/test resources) — mark exempt to avoid score drag
- Regulatory compliance: enable CIS, SOC2, PCI-DSS standard → mapped recommendations
- Automation: Defender auto-remediation for select recommendations (e.g., missing diagnostic settings)
- Workflow: export recommendations → JIRA/ADO tickets → SLA per severity
- Target: 80% as initial goal, address critical and high before medium

---

### A-054 | Azure Sentinel — SIEM Architecture for DevOps
**Type:** Design | **Difficulty:** Hard | ⬜

> Design a Sentinel-based SIEM for a DevOps platform. What data sources do you connect and what analytic rules do you create?

**What you must cover:**
- Data sources: Azure Activity Log, AAD Sign-in + Audit, AKS audit log, Key Vault access, GitHub audit log, Azure Firewall logs
- Analytic rules: `kubectl exec into production pod`, `Secret access by CI service principal`, `Admin RBAC granted`, `Login from unfamiliar IP`, `Vault access outside business hours`
- Automation: Sentinel Playbooks (Logic App) — auto-disable AAD account on credential stuffing alert
- UEBA: User Entity Behavior Analytics — baseline normal activity, alert on anomalies
- Incident triage: priority by confidence + severity, auto-assign to team via connector
- Cost: Log Analytics ingestion pricing — ingest only what you query. Use dedicated cluster for high-volume.

---

### A-055 to A-070 | Security Rapid-Fire

---

### A-055 | Explain Azure Confidential Computing (CVM) and when DevOps engineers need to care about it. | Explain | Hard | ⬜

### A-056 | Design a break-glass procedure for emergency admin access to a production AKS cluster with AAD RBAC. | Design | Expert | ⬜
**Hint:** Local kubeconfig with cluster-admin cert (offline access, stored in Azure Key Vault). Break-glass AAD account with strong MFA + alerting on use. Audit every use.

### A-057 | Your CI pipeline service principal has Owner role on the subscription. How do you fix this? | Implement | Hard | ⬜
**Hint:** Scope down to resource-group level + only needed roles (Contributor on RG, AcrPush on ACR, user-assignment on AKS). Use workload identity (OIDC) instead of service principal secret.

### A-058 | Implement Azure Policy to deny creation of any public IP address in production subscription. | Implement | Hard | ⬜
**Hint:** Built-in policy "Not allowed resource types" or custom Deny policy for `Microsoft.Network/publicIPAddresses`. Exclude Azure Firewall, App Gateway (with justification tags).

### A-059 | A developer accidentally pushed a secret to GitHub in a public repo. Walk through the incident response steps. | Troubleshoot | Hard | ⬜
**Hint:** Immediately revoke/rotate secret. Remove from Git history (BFG repo cleaner, git filter-branch). Audit logs for any usage since exposure. GitHub secret scanning retroactively alerts.

### A-060 | Explain PIM (Privileged Identity Management) and how you integrate it with AKS cluster admin access. | Explain | Hard | ⬜
**Hint:** PIM: just-in-time activation of privileged roles, requires MFA + justification, time-bounded. AKS: cluster-admin ClusterRoleBinding to AAD group that's PIM-managed. Audit activation events.

### A-061 | Design a secrets rotation strategy for a database password used by 100 microservices on K8s. | Design | Expert | ⬜
**Hint:** Vault dynamic secrets (DB secrets engine) — each pod gets short-lived unique credential. Vault agent renews before expiry. No shared password. No rotation needed (credential expires naturally).

### A-062 | Azure Defender for Containers fires "Anomalous container executed command". How do you investigate? | Troubleshoot | Hard | ⬜
**Hint:** Defender alert → Sentinel incident → audit trail via K8s audit log. Match alert timestamp to `kubectl exec` API call, source IP, user identity.

### A-063 | Implement image signing verification so AKS only runs signed images from trusted registry. | Implement | Expert | ⬜
**Hint:** Cosign sign image in CI. Ratify + Azure Policy or Kyverno verifyImages policy — reject pods with unverified images. Trust store: Sigstore/Notation.

### A-064 | Explain Zero Trust Network Access (ZTNA) for internal platforms. How does it differ from VPN? | Explain | Hard | ⬜
**Hint:** VPN: network-level access (once in VPN, can reach everything). ZTNA: identity + device posture verified per application. Never trust, always verify. Azure AD Conditional Access + App Proxy.

### A-065 | Design a compliance reporting system that automatically generates evidence for SOC2 audit. | Design | Expert | ⬜
**Hint:** Drata/Vanta/Secureframe: auto-pull evidence from Azure, GitHub, K8s. Screenshots of policies, access review reports, change history from GitOps.

### A-066 | Explain Azure Defender for DevOps. What does it scan and what does it add to traditional SAST tools? | Explain | Medium | ⬜
**Hint:** Integrates with GitHub/ADO. Scans IaC (Terraform, ARM, Bicep) and container images. Exposes findings in Defender for Cloud. Complements SAST for application code.

### A-067 | How do you implement audit-proof immutable log storage for compliance in Azure? | Implement | Hard | ⬜
**Hint:** Azure Immutable Blob Storage: WORM policy (time-based retention lock), legal hold mode. Write from diagnostic settings → storage account with WORM policy. Cannot modify or delete within retention period.

### A-068 | What is the Azure Security Benchmark? How do you use it to prioritize hardening? | Explain | Medium | ⬜
**Hint:** ASB: Microsoft's curated set of controls mapped to CIS, NIST, ISO. Defender for Cloud maps each recommendation to ASB control. Use to show auditors control coverage.

### A-069 | Design an emergency access (break-glass) account strategy for Azure subscriptions. | Design | Expert | ⬜
**Hint:** 2 emergency accounts, not synced from on-prem AD, MFA with FIDO2 key (not phone), Owner role at tenant level, stored offline, alerting on sign-in, reviewed quarterly.

### A-070 | A pod is running as root despite PSS restricted policy on the namespace. How is this possible? | Troubleshoot | Expert | ⬜
**Hint:** Pod was deployed before policy was applied (PSA doesn't retroactively evict). Namespace didn't have label at deploy time. Webhook not processing that namespace.

---

## Section 4: Azure Cost Optimization & FinOps (A-071 to A-085)

---

### A-071 | Azure Reserved Instances Strategy
**Type:** Design | **Difficulty:** Hard | ⬜

> You have a baseline of 50 AKS nodes running 24/7 and burst to 150 nodes during business hours. Design a compute cost optimization strategy.

**What you must cover:**
- Baseline 50 nodes: 1-year Reserved Instances (40% discount) or Azure Savings Plan
- Burst capacity (100 nodes): Spot VMs in separate node pool (60–80% discount, handle preemption gracefully)
- Commitment modeling: Azure Pricing Calculator + 90-day actual consumption analysis before committing
- VM type selection: `Dsv5`, `Dasv5` for general compute — compare pricing per vCPU at target size
- Savings Plan vs RI: Savings Plan more flexible (applies to any VM compute), RI tied to VM size/region
- Storage: reserved capacity for Azure Disk Premium SSD if predictable volumes

---

### A-072 | Azure Cost Anomaly Detection
**Type:** Design | **Difficulty:** Medium | ⬜

> Implement an automated cost anomaly detection and alerting system for Azure using native features.

**What you must cover:**
```bash
# Enable cost anomaly alert
az costmanagement alert create \
  --scope /subscriptions/<id> \
  --type Budget \  
  --name "monthly-budget-alert" \
  --amount 50000 \
  --time-grain Monthly \
  --threshold-type Actual \
  --threshold 80  # Alert at 80% of budget

# Cost anomaly detection (built-in) - enable in Cost Analysis
# Azure native: Cost Management → Cost Alerts → Anomaly Alerts
# Sends email when daily cost is anomalously high vs 7-day average
```
- `az costmanagement query`: export daily cost by resource group, tag
- Power BI: connect to Azure Cost Management connector for dashboards
- Tag enforcement: Azure Policy deny resources without `cost-center`, `team`, `environment` tags

---

### A-073 to A-085 | FinOps Rapid-Fire

---

### A-073 | How do you calculate the actual cost per team using shared AKS cluster infrastructure? | Design | Hard | ⬜
**Hint:** Kubecost: cost allocation by namespace label. Tag nodes with team label. Allocate shared costs (system node pool) proportionally by namespace usage.

### A-074 | A subscription's cost doubled overnight. Walk through investigation. | Troubleshoot | Hard | ⬜
**Hint:** Cost Analysis → drill by service → drill by resource. Check for new high-cost resource (VMSS scale-out, new P20 disk, cross-region data transfer spike). Check recent deployments/IaC changes.

### A-075 | Explain the difference between chargeback and showback. Which is harder to implement and why? | Explain | Medium | ⬜
**Hint:** Showback: report costs to teams (informational). Chargeback: actually bill/charge teams (affects budget). Chargeback requires business agreement, accounting integration, exception handling.

### A-076 | Design an automated cluster scale-down for non-business hours to reduce costs by 40%. | Design | Hard | ⬜
**Hint:** GitHub Actions/Azure Automation scheduled workflow: `kubectl scale deployment --replicas=0` for non-critical, `az aks nodepool update --min-count 0` for autoscaler, AKS stop/start for dev clusters.

### A-077 | Explain Azure Spot VMs eviction behavior in AKS context. What's the 30-second eviction notice? | Explain | Hard | ⬜
**Hint:** Azure sends IMDS (Instance Metadata Service) preemption notice 30 seconds before eviction. Node drain on SIGTERM. AKS registers preemption handler — graceful pod termination if within terminationGracePeriod.

### A-078 | How do you identify overprovisioned resources in an Azure subscription? | Implement | Medium | ⬜
**Hint:** Azure Advisor (CPU/memory utilization < 5% for 7 days = idle). Cloudyn (deprecated). Right-size recommendations. Custom query: `az monitor metrics list` for CPU utilization per VM.

### A-079 | A development team is keeping staging environments running 24/7 costing $20,000/month. Design a governance model. | Design | Medium | ⬜
**Hint:** Auto-shutdown policy on VMs. AKS cluster stop on schedule. Cost alert per team namespace. Required justification in ticket for 24/7 exceptions. Policy: no long-running ephemeral environments without tag.

### A-080 | Explain Azure Cost Management export and how to build a cost dashboard in Grafana. | Implement | Hard | ⬜
**Hint:** Cost export → Azure Blob Storage (CSV). Grafana JSON datasource or Azure Data Explorer → query costs. Alternative: Kubecost Grafana integration for K8s-specific cost.

### A-081 | How do you prevent cost overruns in a multi-team Azure environment using native guardrails? | Design | Hard | ⬜
**Hint:** Management Group hierarchy with Budgets per subscription/RG. Azure Policy: restrict SKU sizes (no P80 disks in dev). Spending limit on pay-as-you-go. Alert at 80% + 100% budget.

### A-082 | Optimize Azure Kubernetes Storage costs — what storage types should you default to for different workloads? | Design | Medium | ⬜
**Hint:** Dev/test: Standard_LRS HDD (cheapest). Staging: Standard_SSD_LRS. Production stateful (DB): Premium_SSD_LRS. High-IOPS: UltraSSD or Premium_SSD_v2. Don't overprovision disk size (you pay for provisioned, not used).

### A-083 | Your Azure Data Transfer costs tripled. How do you find the root cause? | Troubleshoot | Hard | ⬜
**Hint:** Cost Analysis by Meter subcategory → "Data Transfer Out" breakdown by region. Cross-region traffic is charged (within Azure). Identify services sending data cross-region. Fix: co-locate services, use Private Endpoints to avoid geo-redundant transfer.

### A-084 | Compare Azure Savings Plans vs Reserved Instances for an unpredictable workload. | Explain | Medium | ⬜
**Hint:** Savings Plan: compute commitment by hourly spend, applies to any compute service (VMs, AKS, ACI). RI: specific VM size + region + term. Savings Plan: more flexible, slightly less discount. RI: maximum discount for predictable workloads.

### A-085 | How do you track cloud cost as an engineering metric alongside reliability and velocity? | Design | Hard | ⬜
**Hint:** Cost/request (unit economics), cost trend vs traffic growth, cost per team per sprint. Dashboard in Grafana alongside latency SLO. Engineering OKR: "reduce cost/request by 20% this quarter".

---

## Section 5: Azure IaC, DevOps, and Platform Engineering (A-086 to A-100)

---

### A-086 | Azure Landing Zone Design
**Type:** Design | **Difficulty:** Expert | ⬜

> Design an Azure Landing Zone for 20 product teams using Cloud Adoption Framework (CAF). What management group hierarchy do you use?

**What you must cover:**
- Management Group hierarchy: Tenant Root → Platform (Connectivity, Identity, Management) → Landing Zones → Corp → Online → Sandbox
- Subscriptions: dedicated per team or workload (not resource groups for isolation)
- Platform subscriptions: Connectivity (hub VNet, firewall), Identity (AAD DS, Azure AD), Management (monitoring, security)
- Policy: inherited at management group level — applies to all child subscriptions
- Networking: Connectivity subscription owns hub VNet, spokes attached via peering
- Cost: budgets at subscription level, Management Group policy enforces tagging
- Tooling: CAF Terraform modules (Azure/caf-enterprise-scale), Bicep equivalents

---

### A-087 | Azure DevOps Pipeline vs GitHub Actions
**Type:** Explain | **Difficulty:** Medium | ⬜

> Compare Azure DevOps Pipelines with GitHub Actions for a platform team managing 50 microservices. What are the migration considerations?

**What you must cover:**
- Azure DevOps: integrated with JIRA/Boards/Repos, mature YAML pipelines, environments + approvals, private runners (agent pools), paid hosted agents
- GitHub Actions: stronger OSS ecosystem (Actions marketplace), native to GitHub repos, OIDC to Azure (no stored secrets), reusable workflows, ARC for K8s-native runners
- Migration: azure-pipelines.yml → .github/workflows/ (different syntax, mostly conceptual mapping)
- Service connections (ADO) → OIDC federated credentials (GHA) — no secrets stored
- Artifact feeds (ADO) → GitHub Packages / ACR — may need migration
- Cost: GHA: per-minute pricing on GitHub-hosted runners. ADO: parallel job pricing.

---

### A-088 | Azure Service Bus vs Event Hub vs Event Grid
**Type:** Explain | **Difficulty:** Medium | ⬜

> Compare Azure Service Bus, Event Hub, and Event Grid. Give a messaging pattern for each.

**What you must cover:**
- Service Bus: enterprise messaging (queues + topics), ordered delivery, dead-letter queue, delivery guarantee, up to 256KB/1MB messages. Pattern: command/control, task queue, work distribution
- Event Hub: high-throughput event streaming (millions/sec), partitioned log (like Kafka), consumer groups, capture to storage. Pattern: telemetry ingestion, log streaming, event sourcing
- Event Grid: pub/sub for Azure events (resource events, custom events), push delivery, 1-per-event model, serverless triggers. Pattern: react to Azure resource changes (blob created → function processes)
- Key difference: Service Bus guarantees single delivery; Event Hub is replay-able; Event Grid is fire-and-forget (retry but not guaranteed order)

---

### A-089 | Azure Functions vs Azure Container Apps vs AKS
**Type:** Explain | **Difficulty:** Medium | ⬜

> When would you choose Azure Functions vs Container Apps vs AKS for running a microservice?

**What you must cover:**
- Azure Functions: event-driven, scale-to-zero, consumption plan (pay per execution), fast to develop, cold start risk
- Container Apps: container-native, KEDA-powered autoscaling, scale-to-zero, Dapr integration, less operational overhead than AKS
- AKS: full control, Kubernetes-native patterns, stateful workloads, advanced networking, all K8s ecosystem tools
- Decision tree: event-driven + serverless → Functions. Container workload + less K8s complexity → Container Apps. Full K8s feature set/compliance → AKS.

---

### A-090 to A-100 | Azure Platform Rapid Scenarios

---

### A-090 | Implement blue/green deployment to Azure Container Apps using traffic weight splitting. | Implement | Hard | ⬜

### A-091 | Design an Azure API Management (APIM) layer for 50 microservices. What policies do you configure? | Design | Hard | ⬜
**Hint:** Rate limiting, JWT validation, request transformation, caching, backend circuit breaker, subscription keys per consumer app.

### A-092 | Explain Azure Managed Grafana and when it's preferred over self-hosted Grafana on AKS. | Explain | Medium | ⬜

### A-093 | Design a multi-region Azure deployment for GDPR compliance with data residency enforcement via Azure Policy. | Design | Expert | ⬜

### A-094 | A Logic App processing security alerts is failing intermittently. How do you debug and add resilience? | Troubleshoot | Hard | ⬜

### A-095 | Explain Bicep vs Terraform vs Pulumi for Azure-first organizations. When does each make sense? | Explain | Hard | ⬜
**Hint:** Bicep: Azure-native, no state file, ARM under the hood, great DX for Azure-only. Terraform: multi-cloud, mature ecosystem, drift detection, state management complexity. Pulumi: code-first (Python/TS), power of a programming language.

### A-096 | Implement Azure Event Grid to trigger an AKS workload when a file is uploaded to Blob Storage. | Implement | Hard | ⬜

### A-097 | Design a central log analytics workspace hierarchy for a company with 20 Azure subscriptions. | Design | Hard | ⬜
**Hint:** One central workspace per environment tier (dev, staging, prod), Diagnostic Settings from all subscriptions → central workspace. RBAC for team-scoped log access. Long-term: Azure Data Explorer for analytics.

### A-098 | Explain Azure Policy remediation tasks. How do you remediate non-compliant existing resources? | Implement | Medium | ⬜
**Hint:** Policy effect `DeployIfNotExists` + remediation task creates/modifies non-compliant resources. `Modify` effect patches properties. Not retroactive automatically — trigger remediation task manually or via automation.

### A-099 | Your Azure subscription hit its compute quota limit and deployments are failing. How do you resolve? | Troubleshoot | Medium | ⬜
**Hint:** `az vm list-usage`, identify which quota is exhausted, request increase via Azure portal (support ticket), or rearrange workloads to different region.

### A-100 | Design an Azure-native end-to-end DevSecOps pipeline integrating: Container scanning, IaC scanning, SAST, secret detection, and deployment to AKS. | Design | Expert | ⬜
**Hint:** GitHub Actions: `actions/checkout` → checkov (IaC) → Semgrep (SAST) → gitleaks (secrets) → Docker build → Trivy (container) → Push ACR → Cosign sign → ArgoCD sync → Defender audit.
