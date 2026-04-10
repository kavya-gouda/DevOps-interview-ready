# Networking — 90 Scenarios

> **ID prefix:** N- | Types: Design, Implement, Troubleshoot, Explain | Difficulty: Medium → Expert

---

## Section 1: Kubernetes Networking (N-001 to N-035)

---

### N-001 | Pod Networking Model — The Fundamentals
**Type:** Explain | **Difficulty:** Hard | ⬜

> Explain how two pods on different nodes communicate in Kubernetes. Walk from packet origination to delivery.

**What you must cover:**
- Each pod has a flat routable IP (no NAT between pods)
- Packet flow: Pod A (node 1) → veth pair → Linux bridge/CBR0 → node routing table → overlay (VXLAN/BGP)
- CNI plugin creates the network: Calico (BGP routes), Flannel (VXLAN), Cilium (eBPF)
- Node routing: each node knows pod CIDRs for other nodes (if BGP) or encapsulates (if VXLAN)
- kube-proxy: maintains iptables/ipvs rules for Service ClusterIPs (not used for pod-to-pod)
- AKS: Azure CNI (pods get Azure VNet IPs, no overlay), or Kubenet (overlay + Azure route table)

---

### N-002 | Service Types — ClusterIP vs NodePort vs LoadBalancer vs ExternalName
**Type:** Explain | **Difficulty:** Medium | ⬜

> Explain each Kubernetes Service type with a concrete use case.

```yaml
# ClusterIP: internal communication only
kind: Service
spec:
  type: ClusterIP  # Default. Only accessible within cluster.
  
# NodePort: exposes on each node's IP:port (30000-32767)
spec:
  type: NodePort
  ports:
  - port: 80
    nodePort: 31000  # External traffic → any node:31000 → pod

# LoadBalancer: creates Azure/AWS LB, gets external IP
spec:
  type: LoadBalancer
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"  # Internal LB

# ExternalName: DNS alias to external service
spec:
  type: ExternalName
  externalName: mydb.postgres.database.azure.com
```
- Headless Service (`clusterIP: None`): no VIP, DNS returns individual pod IPs (StatefulSet, direct pod access)
- LoadBalancer: each Service creates its own Azure LB (expensive). Use Ingress instead for HTTP.

---

### N-003 | Ingress vs Gateway API
**Type:** Design | **Difficulty:** Hard | ⬜

> You have 30 services that need external HTTP access with different path-based routing. Design the ingress strategy.

```yaml
# NGINX Ingress (classic)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/rate-limit: "100"  # rate limiting
spec:
  ingressClassName: nginx
  rules:
  - host: api.company.com
    http:
      paths:
      - path: /users
        pathType: Prefix
        backend:
          service: {name: users-svc, port: {number: 80}}
      - path: /orders
        pathType: Prefix
        backend:
          service: {name: orders-svc, port: {number: 80}}
  tls:
  - hosts: [api.company.com]
    secretName: api-tls

# Gateway API (future standard, more expressive)
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
spec:
  rules:
  - matches: [{path: {type: PathPrefix, value: /users}}]
    backendRefs: [{name: users-svc, port: 80, weight: 90}]  # Weighted routing built-in
```

---

### N-004 | Network Policy — Zero-Trust Microsegmentation
**Type:** Implement | **Difficulty:** Hard | ⬜

> Implement a zero-trust network policy for a 3-tier app: frontend → API → database. Each tier can only call the next.

```yaml
# Default deny all ingress/egress in namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}   # All pods
  policyTypes: [Ingress, Egress]
---
# Frontend: allow ingress from internet, egress to API only
kind: NetworkPolicy
metadata:
  name: frontend-policy
spec:
  podSelector:
    matchLabels: {tier: frontend}
  policyTypes: [Ingress, Egress]
  ingress:
  - {}    # Allow from anywhere (internet → LB → pod)
  egress:
  - to:
    - podSelector:
        matchLabels: {tier: api}
    ports:
    - port: 8080
---
# API: ingress from frontend only, egress to DB only
kind: NetworkPolicy
metadata:
  name: api-policy
spec:
  podSelector:
    matchLabels: {tier: api}
  ingress:
  - from:
    - podSelector: {matchLabels: {tier: frontend}}
  egress:
  - to:
    - podSelector: {matchLabels: {tier: database}}
    ports:
    - port: 5432
  - to:  # Allow DNS
    - namespaceSelector: {matchLabels: {kubernetes.io/metadata.name: kube-system}}
    ports:
    - port: 53
      protocol: UDP
```

---

### N-005 | DNS Resolution in Kubernetes
**Type:** Troubleshoot | **Difficulty:** Hard | ⬜

> A pod can't resolve `postgres-svc.database.svc.cluster.local`. Walk through debugging.

```bash
# Test from pod
kubectl exec -it debug-pod -- nslookup postgres-svc.database.svc.cluster.local

# Check CoreDNS
kubectl get pods -n kube-system -l k8s-app=coredns
kubectl logs -n kube-system -l k8s-app=coredns --tail=50

# Verify service exists
kubectl get svc postgres-svc -n database

# Check CoreDNS ConfigMap
kubectl get configmap coredns -n kube-system -o yaml

# Test DNS resolution steps
# 1. Search domains: postgres-svc → postgres-svc.database.svc.cluster.local (ndots:5)
# 2. If NXDOMAIN: service doesn't exist or wrong namespace
# 3. If timeout: CoreDNS pod issue, network policy blocking port 53
```
- FQDN format: `<service>.<namespace>.svc.<cluster-domain>` (default `cluster.local`)
- Short names work due to `resolv.conf` search domains in pod

---

### N-006 | Calico vs Cilium vs Flannel
**Type:** Explain | **Difficulty:** Expert | ⬜

> Compare the major CNI plugins. When would you choose Cilium over Calico?

| Feature | Flannel | Calico | Cilium |
|---|---|---|---|
| Network Policy | No | Yes (full) | Yes (L7 aware) |
| Underlying tech | VXLAN | BGP / VXLAN / eBPF | eBPF only |
| Performance | Good | Better | Best (bypass iptables) |
| L7 visibility | No | No | Yes (HTTP, gRPC) |
| Encryption | No | WireGuard | WireGuard |
| Service mesh | No | No | Cilium Mesh (native) |

- Choose Cilium when: high performance needed, L7 network policy (FQDN, HTTP path), eBPF-based observability (Hubble), Kubernetes Gateway API
- Choose Calico when: BGP peering with on-prem network, simpler operations, large established ecosystems

---

### N-007 | Service Mesh — Istio Architecture
**Type:** Design | **Difficulty:** Expert | ⬜

> Explain Istio's architecture. What does the data plane do vs the control plane?

**Control plane (Istiod):**
- Pilot: service discovery, xDS API → distributes routing rules to Envoy proxies
- Citadel: certificate authority → issues mTLS certificates (SPIFFE-based)
- Galley: configuration validation

**Data plane (Envoy sidecars):**
- Intercepts all pod traffic (iptables redirect, port 15006 inbound, 15001 outbound)
- Enforces mTLS between services
- Applies traffic management: retries, circuit breaking, timeouts, canary routing
- Reports telemetry: traces to Jaeger, access logs to Loki

```yaml
# mTLS policy - require encrypted comms
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT  # All inter-service comms must be mTLS
```

---

### N-008 | CoreDNS Custom Configuration
**Type:** Implement | **Difficulty:** Hard | ⬜

> Pods need to resolve internal corporate domain `corp.internal` using an on-prem DNS server. Configure CoreDNS.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . /etc/resolv.conf     # Default: forward to node DNS
        cache 30
        loop
        reload
        loadbalance
    }
    corp.internal:53 {
        errors
        forward . 10.0.1.53 10.0.1.54  # On-prem DNS servers for corp.internal
        cache 10
    }
```
- Restart CoreDNS pods after ConfigMap update
- Test: `kubectl exec -it test-pod -- nslookup myhost.corp.internal`

---

### N-009 to N-035 | Kubernetes Networking Rapid-Fire

---

### N-009 | A pod's readiness probe is succeeding but it's still not receiving traffic. What are 5 possible causes? | Troubleshoot | Hard | ⬜
**Hint:** 1. Node network issue (pod IP not routing). 2. Service selector mismatch (label wrong). 3. NetworkPolicy blocking ingress. 4. kube-proxy not synced (iptables stale). 5. Endpoints object not populated (`kubectl get endpoints svc-name`).

### N-010 | Implement Kubernetes Ingress with cert-manager for automatic TLS certificate from Let's Encrypt. | Implement | Hard | ⬜
**Hint:** ClusterIssuer with ACME HTTP-01 or DNS-01 challenge. Ingress annotation: `cert-manager.io/cluster-issuer: letsencrypt-prod`. Cert-manager creates Certificate → Secret. Ingress TLS references the Secret.

### N-011 | What is a headless Service and when would you use it? | Explain | Medium | ⬜
**Hint:** `clusterIP: None`. DNS returns A records for each pod IP (not a single VIP). Use case: StatefulSets (each pod needs stable DNS). Custom load balancing (client-side). Direct pod addressing.

### N-012 | How does kube-proxy iptables mode work? What are its performance limitations? | Explain | Expert | ⬜
**Hint:** kube-proxy: for each Service, adds iptables DNAT rules. Packet → iptables chain → random selection of Endpoint → pod IP. Limitation: O(n) rules — 10,000 services = 10,000 iptables rules, slow updates. IPVS mode: better O(1) lookup.

### N-013 | A NetworkPolicy blocks a pod from reaching the Kubernetes API server. How do you diagnose and fix? | Troubleshoot | Hard | ⬜
**Hint:** API server communication: port 443 or 6443 to `kubernetes.default.svc`. Egress policy must allow: `to: namespaceSelector: {kubernetes.io/metadata.name: default}` port 443. Or allow `to: ipBlock: cidr: <api-server-ip>/32`.

### N-014 | Implement an ExternalDNS setup for automatically creating Azure DNS records for Kubernetes Services and Ingresses. | Implement | Hard | ⬜
**Hint:** ExternalDNS controller: watches Services (`type: LoadBalancer`) and Ingresses, creates Azure DNS records via Azure SDK. RBAC: workload identity to manage DNS zone. Annotation: `external-dns.alpha.kubernetes.io/hostname: api.example.com`.

### N-015 | Explain IPVS mode in kube-proxy. How is it better than iptables for large clusters? | Explain | Hard | ⬜
**Hint:** IPVS: Linux kernel load balancing. kube-proxy in IPVS mode: creates virtual server per Service ClusterIP, registers backends. Algorithms: round-robin, least-conn, source-hash. O(1) lookup vs O(n) iptables. Required for >3,000 Services.

### N-016 | A Kubernetes service with 3 pods: traffic always goes to one pod. What's happening? | Troubleshoot | Hard | ⬜
**Hint:** Session affinity (`service.spec.sessionAffinity: ClientIP`). Source IP NAT: all traffic appears from same client. IPVS source hash. Check: `kubectl get svc -o yaml | grep sessionAffinity`.

### N-017 | Implement global rate limiting in Kubernetes using NGINX Ingress and a shared Redis store. | Implement | Expert | ⬜
**Hint:** NGINX Ingress: `nginx.ingress.kubernetes.io/global-rate-limit: "100"` + `nginx.ingress.kubernetes.io/global-rate-limit-window: "1m"`. Requires Redis sidecar or external Redis. Rate limit applied across all NGINX replicas.

### N-018 | Explain how Istio's traffic management `DestinationRule` and `VirtualService` work together. | Explain | Hard | ⬜
**Hint:** VirtualService: routing rules (match on host/path/headers → route to subset). DestinationRule: defines subsets (v1, v2 with pod labels) + connection pool settings, circuit breaker, load balancing policy. Together: route 90% to v1, 10% to v2 (canary).

### N-019 | What is FQDN-based NetworkPolicy and which CNI supports it? | Explain | Expert | ⬜
**Hint:** Standard NetworkPolicy: only pod/namespace selectors — can't filter by domain name. Calico GlobalNetworkPolicy with `domains`, Cilium `CiliumNetworkPolicy` with `toFQDNs`: allow egress to `api.stripe.com` (resolves DNS, creates IP-based rules dynamically).

### N-020 | Implement multi-tenant network isolation in a single AKS cluster for two business units. | Design | Expert | ⬜
**Hint:** Separate namespaces. NetworkPolicy: default-deny, only allow within namespace. Separate node pools (taints/tolerations for workload isolation). Separate ingress controllers per BU. Virtual Node or namespace quotas for compute isolation.

### N-021 | How does Cilium Hubble provide network observability without code changes? | Explain | Expert | ⬜
**Hint:** Hubble: observability layer for Cilium. eBPF programs monitor every network packet at kernel level — no sidecar needed. Service dependency graph, per-connection HTTP/gRPC visibility, dropped packets, DNS queries. `hubble observe` CLI or Hubble UI.

### N-022 | A pod is unable to connect to an Azure SQL database. The connection times out. Debugging approach. | Troubleshoot | Hard | ⬜
**Hint:** 1. `kubectl exec -it pod -- nc -zv mydb.database.windows.net 1433` — timeout vs refused. 2. NetworkPolicy: egress to Azure SQL IP allowed? 3. Private endpoint: pod in same VNet as SQL PE? 4. DNS resolves to public IP (wrong: use private DNS zone). 5. Azure SQL firewall: allow subnet or service endpoint.

### N-023 | Implement a circuit breaker in Istio for a downstream service that's flaky. | Implement | Hard | ⬜
**Hint:** `DestinationRule.spec.trafficPolicy.outlierDetection`: `consecutive5xxErrors: 5`, `interval: 30s`, `baseEjectionTime: 30s`, `maxEjectionPercent: 50`. Istio ejects unhealthy endpoints from load balancing pool temporarily.

### N-024 | Explain how Kubernetes pods with `hostNetwork: true` differ from regular pods — security implications. | Explain | Hard | ⬜
**Hint:** `hostNetwork: true`: pod shares node's network namespace — sees `eth0`, same IP as node. Use case: network-level logging, node-level proxy. Security risk: pod can access all host network interfaces, listen on arbitrary ports, bypass NetworkPolicy.

### N-025 | Implement a multi-cluster Service mesh using Istio multi-primary on two AKS clusters. | Design | Expert | ⬜

### N-026 | How do you prevent DNS rebinding attacks in Kubernetes? | Explain | Expert | ⬜
**Hint:** CoreDNS: validate that external DNS doesn't return RFC1918 addresses (rebinding). Configure `policy safe` in CoreDNS. Egress NetworkPolicy: block unwanted internal CIDR access. Ingress validation.

### N-027 | Implement sticky sessions in Kubernetes for a stateful web application. | Implement | Medium | ⬜
**Hint:** NGINX Ingress: `nginx.ingress.kubernetes.io/affinity: cookie`, `nginx.ingress.kubernetes.io/session-cookie-name: route`. Or: Service `sessionAffinity: ClientIP`. Warning: sticky sessions complicate scaling.

### N-028 | A Service type LoadBalancer is stuck in pending state on AKS. Why? | Troubleshoot | Medium | ⬜
**Hint:** Azure Load Balancer controller can't create LB. Common: subscription quota for public IPs exceeded, subnet IP exhaustion (for internal LB), missing AKS service principal permissions on subnet (for non-managed identity).

### N-029 | Explain Kubernetes Endpoint slices and how they improve scalability vs Endpoints. | Explain | Hard | ⬜
**Hint:** Endpoints: one large object with all pod IPs — every update triggers full object replacement. EndpointSlice: max 100 endpoints per slice — updates are surgical, only changed slice updated. Required for >100 pod services.

### N-030 | Implement a custom timeout and retry policy for a specific API route using Istio VirtualService. | Implement | Hard | ⬜
**Hint:** `VirtualService.spec.http.timeout: 5s`. `retries: attempts: 3, perTryTimeout: 2s, retryOn: 5xx,reset,connect-failure`. Apply to specific `match: uri: prefix: /api/payments` only.

### N-031 | How do you implement East-West traffic encryption in Kubernetes without a service mesh? | Implement | Expert | ⬜
**Hint:** WireGuard at CNI level (Cilium WireGuard, Calico WireGuard). Encrypts pod-to-pod traffic node-to-node. No application changes. Alternative: mTLS in application code. Or: IPsec between nodes.

### N-032 | Explain how AKS Azure CNI works and its IP allocation implications for large clusters. | Explain | Expert | ⬜
**Hint:** Azure CNI: each pod gets an IP from the subnet (not an overlay). Implication: must pre-allocate large enough subnet. 100 nodes × 30 pods each = 3,000 IP addresses needed. Azure CNI Overlay: pods use private overlay IPs, only node IPs from subnet (solves IP exhaustion).

### N-033 | Implement a zero-downtime pod network migration from Flannel to Cilium on a live cluster. | Design | Expert | ⬜

### N-034 | How do you troubleshoot packet drops in Kubernetes? What eBPF tools help? | Troubleshoot | Expert | ⬜
**Hint:** `cilium monitor --type drop` — real-time drop events. `bpftool prog` — inspect eBPF programs. `tc -s qdisc show` — traffic shaping drops. `netstat -s` — TCP retransmits. Kernel: `perf stat -e 'net:*drop*'`.

### N-035 | Explain how hairpin NAT works and when you need it in Kubernetes. | Explain | Expert | ⬜
**Hint:** Hairpin: when a pod accesses its own Service ClusterIP, packet routes back to same pod. Without hairpin: pod talks to its own IP via DNAT → TCP rejects (same source/dest in NAT table). kube-proxy handles this via MASQUERADE rule.

---

## Section 2: Azure Networking (N-036 to N-070)

---

### N-036 | Hub-Spoke VNet Design
**Type:** Design | **Difficulty:** Hard | ⬜

> Design a hub-spoke Azure VNet topology for 3 business units with centralized egress and connectivity to on-premises.

```
On-Premises
    │
    │ ExpressRoute/VPN
    ▼
Hub VNet (10.0.0.0/16)
├── Azure Firewall Subnet (10.0.0.0/26) 
├── VPN/ExpressRoute Gateway Subnet (10.0.1.0/27)
├── Azure Bastion Subnet (10.0.2.0/27)
└── DNS Resolver Inbound Endpoint (10.0.3.0/28)
    │
    ├── VNet Peering → BU1 Spoke (10.1.0.0/16)
    │                    ├── AKS Subnet (10.1.1.0/22) - 1024 IPs
    │                    └── workloads
    ├── VNet Peering → BU2 Spoke (10.2.0.0/16)
    └── VNet Peering → BU3 Spoke (10.3.0.0/16)
```
- All egress: Spoke → Hub → Azure Firewall → Internet (centralized policy)
- User-Defined Route on spoke subnets: `0.0.0.0/0 → Azure Firewall private IP`
- Hub peering: `allowGatewayTransit: true` on hub side, `useRemoteGateways: true` on spoke side

---

### N-037 | Private Endpoints vs Service Endpoints
**Type:** Explain | **Difficulty:** Hard | ⬜

> A developer asks: "Should we use Private Endpoint or Service Endpoint for Azure SQL?" What do you recommend?

| Feature | Service Endpoint | Private Endpoint |
|---|---|---|
| IP type | Public IP of service | Private IP in your VNet |
| Data path | Still traverses public internet (Microsoft backbone) | Fully private (no public IP) |
| DNS | Public DNS | Private DNS Zone required |
| Cost | Free | ~$7/month per endpoint + data |
| Cross-region | Not supported | Supported |
| Exposure | Service still visible from internet | Service invisible from internet |

**Recommendation:** Private Endpoint for all production workloads. Service Endpoint only for dev/test where cost matters.

---

### N-038 | Azure Private DNS Resolution
**Type:** Implement | **Difficulty:** Hard | ⬜

> AKS pods can't resolve `mydb.privatelink.database.windows.net`. Walk through the DNS configuration required.

```
Configuration chain:
1. Private DNS Zone: privatelink.database.windows.net (linked to AKS VNet)
2. A Record: mydb → 10.1.5.4 (private endpoint IP)
3. AKS: CoreDNS forwards to Azure DNS server (168.63.129.16)
4. Azure DNS: authoritative for private DNS zone → returns 10.1.5.4
5. Pod: sees private IP → traffic stays inside VNet
```
- If Private DNS Zone NOT linked to AKS VNet: DNS returns public IP → traffic goes to internet (breaks if PE is private-only)
- Azure DNS Resolver: for conditional forwarding from AKS to on-prem DNS
- CoreDNS ConfigMap: `forward . 168.63.129.16` (Azure DNS — default, resolves private zones if VNet-linked)

---

### N-039 | Azure Firewall Policy Design
**Type:** Design | **Difficulty:** Hard | ⬜

> Design Azure Firewall rules for AKS cluster egress: allow required AKS API server traffic, Azure services, and your application's external API calls.

```
Application Rule Collections (FQDN based):
Priority 100 — AKS Required (Allow):
  - *.hcp.<region>.azmk8s.io:443    # AKS API server
  - mcr.microsoft.com, *.data.mcr.microsoft.com:443  # Microsoft Container Registry
  - management.azure.com:443        # ARM API
  - *.pkg.dev, gcr.io:443          # Container registries

Priority 200 — App External APIs (Allow):
  - api.stripe.com:443             # Payment API
  - api.sendgrid.com:443           # Email API

Network Rule Collections:
Priority 100 — DNS (Allow):
  - Protocol: UDP, Port: 53, Destination: 168.63.129.16  # Azure DNS
```

---

### N-040 to N-070 | Azure Networking Rapid-Fire

---

### N-040 | What is the difference between Azure Front Door, Application Gateway, and Traffic Manager? | Explain | Hard | ⬜
**Hint:** Traffic Manager: DNS-based global routing (no data plane, P2P to origin). App Gateway: regional L7 LB (WAF, SSL term, session affinity). Front Door: global L7 LB + CDN + WAF (edge presence in all regions). Use all three for global HA + WAF + regional failover.

### N-041 | Implement Azure Application Gateway Ingress Controller (AGIC) on AKS. What are the trade-offs vs NGINX? | Implement | Hard | ⬜
**Hint:** AGIC: maps Kubernetes Ingress → Azure Application Gateway native rules. Pro: Azure-native WAF, SSL cert from Key Vault, autoscaling. Con: one AppGW per cluster (expensive), limited NGINX annotation support, slower propagation.

### N-042 | AKS subnet is exhausted (no IPs left). How do you fix without downtime? | Troubleshoot | Expert | ⬜
**Hint:** Azure CNI: pods use subnet IPs. Expand subnet address space (Azure portal → CIDR expansion if adjacent space available). Or: migrate to Azure CNI Overlay (pods use private overlay IPs, decouple from subnet). Online migration possible in newer AKS versions.

### N-043 | How does Azure VNet peering handle transitive routing? What's the limitation? | Explain | Hard | ⬜
**Hint:** VNet peering: NOT transitive. A↔B, B↔C, but A cannot reach C (unless explicitly peered or using Azure Virtual WAN/Hub). Hub-spoke solves this via centralized routing (UDR through hub NVA or Azure Firewall).

### N-044 | Implement AKS with private cluster — pods need to pull images from a private ACR. Walk through the configuration. | Implement | Expert | ⬜
**Hint:** Private cluster: API server has only private IP. ACR private endpoint in same VNet. DNS: Private DNS Zone for `<registry>.azurecr.io` → PE IP. AKS node pool: in same or peered VNet. Managed identity: AKS → `AcrPull` role on ACR.

### N-045 | Design a multi-region active-active architecture on Azure for 99.99% availability. | Design | Expert | ⬜
**Hint:** Azure Front Door → two active regions (EUS + WEU). Session state: Cosmos DB global distribution. Database: configured for multi-region write or read replicas. Deploy: separate AKS clusters each region, ArgoCD multi-cluster. Failover: AFD health probe removes unhealthy origin.

### N-046 | Explain Azure ExpressRoute circuit redundancy and failover design. | Explain | Expert | ⬜
**Hint:** ExpressRoute circuit: 2 MSEE routers (primary + secondary port) — always two physical connections to the location. Failure: if primary MSEE fails, BGP reconverges to secondary. For full redundancy: two ExpressRoute circuits from different peering locations + VPN as cold backup.

### N-047 | A workload in AKS needs internet access but traffic must go through an inspection proxy. Implement the architecture. | Implement | Expert | ⬜
**Hint:** Azure Firewall as HTTP proxy: AKS → UDR 0.0.0.0 → Firewall → internet. OR: corporate proxy with `HTTP_PROXY` environment variable set on pods. Cluster: `--http-proxy-config` at AKS level.

### N-048 | What is Azure Virtual WAN and when should you use it over hub-spoke VNet peering? | Explain | Hard | ⬜
**Hint:** VWAN: Microsoft-managed hub (SD-WAN, automated routing). Supports: any-to-any connectivity (transitive), global backbone, branch office SD-WAN integration. Use when: >50 VNets, global deployment, transitive routing needed without managing NVA.

### N-049 | Implement an Azure DDoS Protection plan for an AKS cluster with a public ingress. | Implement | Hard | ⬜
**Hint:** Azure DDoS Standard: per-VNet protection plan. Enable on AKS VNet. Adaptive tuning: learns baseline traffic, auto-mitigates volumetric attacks. Pair with: Azure Firewall (L7 protection) + WAF (AppGW or AFD).

### N-050 | An AKS pod can reach the internet but can't reach an on-premises resource via ExpressRoute. Debug. | Troubleshoot | Expert | ⬜
**Hint:** 1. UDR on AKS subnet: does on-prem CIDR route via ExpressRoute gateway or via internet? 2. BGP propagation: ER gateway advertises on-prem routes — check effective routes on NIC. 3. NSG on gateway subnet: outbound blocked? 4. On-prem firewall: return traffic to AKS pod CIDR allowed?

### N-051 | How do you implement Azure Private Link Service to expose an internal AKS service to another tenant? | Implement | Expert | ⬜

### N-052 | Design an Azure network topology for PCI-DSS compliance — cardholder data environment isolated. | Design | Expert | ⬜
**Hint:** CDE VNet: completely isolated, private endpoints for all services. No peering to non-CDE VNets. All egress: via dedicated Azure Firewall in CDE hub. Zero inbound from internet to CDE (AFD WAF → internal LB via PrivateLink). NSG flow logs enabled for audit.

### N-053 | Explain BGP routing in Azure VPN/ExpressRoute — what routes are advertised? | Explain | Expert | ⬜
**Hint:** ER: Azure advertises VNet CIDRs + peered VNet CIDRs to on-prem. On-prem advertises its CIDR to Azure (BGP ASN). VPN: site-to-site, addresses exchanged in phase 2 configuration or BGP. Azure Border: AS 12076.

### N-054 | What are Network Security Groups vs Azure Firewall — when do you need both? | Explain | Hard | ⬜
**Hint:** NSG: stateful L4 filtering (IP/port/protocol). Firewall: stateful L4+L7 with FQDN, threat intelligence, IDS/IPS, URL filtering. Use both: NSG for subnet/NIC-level coarse filtering (cheap), Firewall for advanced egress inspection (expensive, ~$1.25/hour).

### N-055 | Implement Azure Bastion for secure RDP/SSH access to AKS nodes without public IPs. | Implement | Medium | ⬜
**Hint:** Azure Bastion: deployed in `AzureBastionSubnet` (/26+). Provides browser-based SSH/RDP. AKS nodes: no public IP, accessed via Bastion. `az network bastion ssh --name BastionName --resource-group RG --target-resource-id <nodeVmId>`.

### N-056 | How does Azure Load Balancer (internal) distribute traffic to AKS pods? | Explain | Hard | ⬜
**Hint:** Internal LB: frontend IP from AKS subnet (private). Backend pool: AKS node IPs (node-to-pod DNAT via kube-proxy). Health probe: TCP or HTTP to nodePort. Session persistence: optional (5-tuple hash by default for equal distribution).

### N-057 | Implement a WAF policy for Azure Application Gateway protecting an AKS service with OWASP rules. | Implement | Hard | ⬜
**Hint:** Application Gateway WAF v2: attach WAFPolicy (OWASP CRS 3.2). Custom rules: block by IP geolocation, rate limit by URI. Run in Detection mode first, review logs → switch to Prevention. Monitor: Azure Monitor WAF logs.

### N-058 | A cross-region VNet peering has higher latency than expected. How do you diagnose? | Troubleshoot | Hard | ⬜
**Hint:** Peering latency = Microsoft backbone. Expected: 10-30ms between EU regions. Check with: `aznping` tool or explicit latency measurements. Alternative: ExpressRoute circuit for lower latency to on-prem. Consider: regional colocate services.

### N-059 | How do you implement split-horizon DNS for an AKS service accessible both internally (private IP) and externally (public IP)? | Implement | Hard | ⬜
**Hint:** External DNS zone: `api.company.com → public IP (AFD/AppGW)`. Private DNS zone in Azure VNet: `api.company.com → private IP (internal service)`. Linked to AKS VNet: internal resolution gets private IP. Same FQDN, different answers based on network location.

### N-060 | Design network security for an AKS cluster handling financial transaction data. | Design | Expert | ⬜
**Hint:** Private cluster + private endpoint for all PaaS. Zero-trust NetworkPolicy. Calico/Cilium FQDN egress control. mTLS between services (Istio). Encryption in transit (WireGuard CNI or service mesh) and at rest. Azure Defender for Kubernetes + network threat detection.

### N-061 | What is SNAT exhaustion in Azure Load Balancer and how does it impact AKS workloads? | Explain | Expert | ⬜
**Hint:** SNAT: when pods connect to internet, node's IP used (NAT through LB). SNAT port pool per backend: limited (~1,024 ports per public IP per backend by default). Exhaustion: TCP connections fail, clients get connection reset. Fix: increase LB IPs (`outbound rule`), use NAT Gateway (dedicated SNAT), or egress via Azure Firewall (centralized SNAT).

### N-062 | Implement Azure NAT Gateway for AKS cluster egress with a static public IP. | Implement | Hard | ⬜
**Hint:** Create NAT Gateway with static public IP prefix. Associate with AKS subnet. All outbound SNAT from pods uses NAT Gateway IP (predictable, whitelistable). Benefit: high port availability, no SNAT exhaustion.

### N-063 | Explain the difference between Azure Private Endpoint DNS resolution on-premises vs in-cloud. | Explain | Expert | ⬜
**Hint:** In-cloud: Private DNS Zone linked to VNet → automatic resolution. On-premises: DNS conditional forwarder → Azure DNS Private Resolver (inbound endpoint IP). On-prem DNS forwards `privatelink.*` queries to Azure Private Resolver → returns PE IP.

### N-064 | How do you migrate an AKS cluster from kubenet to Azure CNI without downtime? | Design | Expert | ⬜

### N-065 | Implement network monitoring for a production AKS cluster using NSG flow logs and Network Watcher. | Implement | Hard | ⬜
**Hint:** NSG flow logs: enable on all AKS-related NSGs. Log Analytics workspace. Traffic Analytics: aggregated view of flows, top talkers, blocked flows. Network Watcher: connection monitor for synthetic network health probes.

### N-066 | How does Azure Front Door handle SSL/TLS termination and certificate management? | Explain | Hard | ⬜
**Hint:** AFD: terminates TLS at PoP (edge). Re-originates new TLS connection to origin (optional). Certificates: AFD-managed (auto-renewed) or BYOC from Key Vault. Minimum TLS 1.2 enforced. End-to-end encryption: PoP → origin with origin certificate validation.

### N-067 | AKS pod IP allocation is slow — new pods take 30 seconds to get an IP. Why? | Troubleshoot | Expert | ⬜
**Hint:** Azure CNI: node pre-allocates a pool of IPs. Queue empty → must request more IPs from Azure IPAM (slow API call). Set `--node-resource-group` max pods and `--max-pods` appropriately. Azure CNI `dynamicIpAllocation` (preview): faster allocation.

### N-068 | How do you implement cross-tenant Private Link access between your company and a partner? | Design | Expert | ⬜
**Hint:** Your side: Private Link Service in front of your LB. Partner side: requests Private Endpoint to your PLS → you approve (or auto-approve for specific subscriptions). DNS: partner creates A record for PE IP. Result: partner traffic reaches your service fully privately.

### N-069 | Explain Azure Route Server and where it fits in an AKS hybrid networking scenario. | Explain | Expert | ⬜
**Hint:** Azure Route Server: enables dynamic BGP route exchange between NVAs (firewalls/SD-WAN) and Azure VNet. Use case: NVA advertises on-prem routes to all VMs in VNet via BGP (no need to manually maintain UDRs). AKS: NVA → Route Server → routes propagate to AKS nodes automatically.

### N-070 | Design a network architecture that satisfies: all production traffic is encrypted, no public IPs on any AKS node, all egress is inspected. | Design | Expert | ⬜
**Hint:** Private AKS cluster (no public node IPs). NAT Gateway or Azure Firewall for egress (inspected). AFD WAF + Private Link for ingress. mTLS or CNI encryption for pod traffic. Result: no public IP surface, all egress inspected.

---

## Section 3: DNS, TLS, and Network Troubleshooting (N-071 to N-090)

---

### N-071 | TLS Certificate Lifecycle Management
**Type:** Design | **Difficulty:** Hard | ⬜

> Design a certificate lifecycle management system for 200 services across AKS, preventing certificate expiry incidents.

**What you must cover:**
- cert-manager: manages internal certs (self-signed CA, Let's Encrypt)
- Internal CA (SPIFFE/SPIRE): service-to-service mTLS with auto-rotating short-lived certs
- External certs: Key Vault integration via `CSI driver` or cert-manager ACME
- Monitoring: Prometheus cert expiry metric `ssl_certificate_expiry_seconds`
- Alert: `ssl_certificate_expiry_seconds < 30 * 86400` (30 days before expiry)
- Rotation: cert-manager auto-renews at 2/3 of lifetime
- Never: manually manage certificates for 200 services — 100% automation

---

### N-072 | Debugging TLS Handshake Failures
**Type:** Troubleshoot | **Difficulty:** Hard | ⬜

> An AKS service fails with "SSL_ERROR_HANDSHAKE_FAILURE". Walk through the debugging process.

```bash
# Test TLS handshake
openssl s_client -connect api.company.com:443 -servername api.company.com

# Useful output to check:
# - Certificate chain validity
# - Certificate expiry date
# - Cipher suite negotiation
# - SNI handling

# From inside a pod
kubectl exec -it debug-pod -- openssl s_client -connect postgres-svc:5432 -starttls postgres

# Common causes:
# 1. Certificate expired (check Not After date)
# 2. Certificate CN/SAN doesn't match hostname (SNI mismatch)
# 3. Server requires client cert (mTLS) — client not providing one
# 4. Cipher suite incompatibility (TLS 1.0 vs 1.3 negotiation)
# 5. Self-signed cert not trusted by client (add CA to trust store)
```

---

### N-073 to N-090 | Network Troubleshooting Rapid-Fire

---

### N-073 | Walk through the complete connection path for a user request: browser → Azure Front Door → AKS Ingress → pod. | Explain | Hard | ⬜
**Hint:** DNS → AFD CNAME → AFD PoP TLS termination → backend pool (AKS cluster IP or AppGW) → NGINX Ingress pod → Service ClusterIP → iptables DNAT → pod.

### N-074 | Implement a network connectivity test framework for validating cluster network policy correctness. | Implement | Expert | ⬜
**Hint:** `netpol-verify` or `network-policy-explorer`. Or: kubectl run test pods with curl/nc, assert expected allowed/denied connections. Integrate into CI: `kubectl apply -f netpol.yaml && run-network-tests.sh`.

### N-075 | DNS TTL is set to 5 minutes. Traffic is still going to old pod IP 30 minutes after pod replacement. Why? | Troubleshoot | Hard | ⬜
**Hint:** Client is caching DNS beyond TTL (OS-level cache ignores TTL). JVM: `networkaddress.cache.ttl=60` property. Node.js, Python: varies. Fix: reduce TTL + ensure clients respect it. Or: use Service ClusterIP (never changes) not direct pod IP.

### N-076 | AKS cluster has intermittent DNS resolution failures under load. Diagnose. | Troubleshoot | Expert | ⬜
**Hint:** CoreDNS overloaded: `kubectl top pods -n kube-system -l k8s-app=coredns`. Increase CoreDNS replicas. NodeLocal DNSCache (`node-local-dns`): reduces load on CoreDNS by caching at each node. Check: `coredns_dns_request_duration_seconds` histogram.

### N-077 | Implement mutual TLS (mTLS) for service-to-service communication in AKS without a service mesh. | Implement | Expert | ⬜
**Hint:** SPIRE: SPIFFE-compliant identity for pods. Each pod gets X.509 SVID (cert+key) via SPIRE Agent. Application: use SVID in TLS handshake. Or: cert-manager Certificate per service, mount as volume. Rotate certs with short TTLs.

### N-078 | A load balancer probe is failing, causing service health checks to show unhealthy. Debug. | Troubleshoot | Hard | ⬜
**Hint:** Azure LB probe: TCP or HTTP check on nodePort. Verify: nodePort is open on node firewall (NSG). kube-proxy: nodePort rule exists (`iptables -t nat -L KUBE-NODEPORTS`). App responds 200 on probe endpoint. Pod is ready (endpoints populated).

### N-079 | Implement network-level forensics for a suspected data exfiltration incident in AKS. | Design | Expert | ⬜
**Hint:** NSG flow logs (L4): identify suspicious external connections. Azure Firewall logs: blocked/allowed FQDN requests. Hubble (Cilium): L7 network topology, which pod connected to which external IP. Packet capture: `tcpdump -i eth0 -w capture.pcap` from pod.

### N-080 | How do you implement geographic load balancing and failover with Azure Traffic Manager? | Implement | Hard | ⬜
**Hint:** Traffic Manager profile: Geographic routing or Priority routing. Endpoints: AKS clusters in two regions (via AppGW or AFD). Failover: Priority routing — primary region serves all traffic, secondary is backup. Geographic: specific countries route to closest region.

### N-081 | What is ECMP (Equal-Cost Multi-Path) routing and how does Calico use it? | Explain | Expert | ⬜
**Hint:** ECMP: multiple equal-cost routes to same destination. Calico with BGP: distributes pod traffic across multiple nodes via ECMP for load distribution. For Service IPs: multiple paths to multiple backends.

### N-082 | Implement an Azure Network Watcher connection monitor to detect latency increases between AKS and on-premises. | Implement | Hard | ⬜
**Hint:** Connection Monitor: source (AKS pod/VM), destination (on-prem host:port), frequency (every 30s). Metrics: RTT, packet loss, checks passed/failed. Alert: latency > 50ms for 5+ minutes.

### N-083 | How does SNAT (Source Network Address Translation) work for an AKS pod reaching the internet? | Explain | Expert | ⬜
**Hint:** Pod IP → kube-proxy MASQUERADE → node IP → Azure Load Balancer SNAT → public IP (or NAT Gateway). Pod IP is not visible externally — only the SNAT IP. Implication: for external whitelisting, share the SNAT IPs.

### N-084 | Implement network access logging for PCI compliance — record every TCP connection to database pods. | Implement | Expert | ⬜
**Hint:** NSG flow logs V2 on database subnet: captures src/dst IP, port, protocol, allowed/denied, bytes. Log Analytics workspace. Retention: 12 months minimum for PCI. Azure Sentinel: alert on unauthorized connection attempts to DB subnet.

### N-085 | A multi-cluster setup has asymmetric routing — packets go in on node A but return on node B. What causes this? | Troubleshoot | Expert | ⬜
**Hint:** Multiple default gateways or ECMP without flow-level consistency. Azure LB: 5-tuple hash ensures symmetric in normal cases. Cross-zone routing: traffic enters one zone, response goes through another zone NVA. Fix: stateful inspection with connection tracking (NVA in HA pair with same state).

### N-086 | Explain Kubernetes EndpointSlice topology-aware routing and when to enable it. | Explain | Expert | ⬜
**Hint:** Topology-aware hints: kube-proxy preferentially routes traffic to endpoints in the same zone as the client (reduces cross-zone egress costs). Enable: `service.kubernetes.io/topology-mode: Auto`. Only effective when endpoints are spread across zones.

### N-087 | How do you implement network-based admission control in Kubernetes? | Explain | Hard | ⬜
**Hint:** NetworkPolicy: default deny before pod starts (policy applied when pod gets IP). Admission webhook: can reject pod creation based on label/annotation (validate NetworkPolicy exists for namespace). Kyverno: require NetworkPolicy before service deployment.

### N-088 | AKS pod has network connectivity but high packet loss (5%). Debug and fix. | Troubleshoot | Expert | ⬜
**Hint:** `ping pod-ip` from another pod — packet loss on overlay? Node saturation: tx queue drops (`ip -s link show eth0`). MTU mismatch: overlay adds headers, may exceed 1500 byte MTU → fragmentation → drops. Fix: set MTU correctly in CNI config. Or: hardware NIC issue → replace node.

### N-089 | Implement a network chaos test: inject packet loss between two AKS services during a load test. | Implement | Expert | ⬜
**Hint:** Chaos Mesh `NetworkChaos`: `action: loss`, `loss.loss: "20"`, `selector: namespace/label`. Duration: 5 minutes. Verify: application handles loss gracefully (retries, circuit breaker). Observe: latency/error rate in Prometheus.

### N-090 | Design a network layout for an AKS cluster that must meet: <5ms latency to Azure SQL, no public IPs, and 10Gbps throughput for data processing. | Design | Expert | ⬜
**Hint:** Proximity Placement Group: AKS node pool + Azure SQL PE in same datacenter. Azure SQL: Business Critical tier (premium SSD, `<1ms` I/O). Private Endpoint: PE in same subnet as AKS → local routing, no VNet hop. AKS network: Accelerated Networking on VMs (10Gbps+), Ultra SSD.
