# Networking (DNS, TLS, Load Balancers, Service Mesh)

> ⭐ Priority area — networking deep dives are common in senior rounds

## Folder Structure

```
07-networking/
├── dns/            # DNS resolution, CoreDNS, split-horizon
├── tls-ssl/        # TLS handshake, cert-manager, mTLS, SPIFFE
├── load-balancers/ # L4 vs L7, algorithms, Nginx, Envoy, ALB/NLB
└── service-mesh/   # Istio, Linkerd, traffic management
```

## Coverage Checklist

### DNS
- [ ] Resolution chain: stub resolver → recursive resolver → authoritative
- [ ] Record types: A, AAAA, CNAME, MX, TXT, SRV, PTR, NS, SOA
- [ ] TTL: impact on propagation, caching, negative caching (NXDOMAIN)
- [ ] Split-horizon DNS: different answers for internal vs external
- [ ] CoreDNS: ConfigMap, plugins (kubernetes, forward, cache, rewrite, log)
- [ ] K8s DNS resolution: `<svc>.<ns>.svc.cluster.local`, headless services
- [ ] ndots setting: impact on lookup performance, best practice
- [ ] DNS-based load balancing vs Anycast

### TLS / SSL
- [ ] TLS 1.2 vs 1.3 handshake: differences, 0-RTT
- [ ] Certificate chain: leaf → intermediate → root CA
- [ ] SNI (Server Name Indication): multi-cert hosting
- [ ] mTLS: client certificates, use in service mesh, SPIFFE/SPIRE
- [ ] cert-manager: ClusterIssuer, Certificate resource, ACME (Let's Encrypt), Vault issuer
- [ ] Certificate rotation: zero-downtime rotation strategies
- [ ] HSTS, OCSP stapling, certificate transparency logs
- [ ] TLS termination: at LB vs at pod (passthrough)
- [ ] Common TLS errors: certificate expired, chain incomplete, SAN mismatch

### Load Balancers
- [ ] L4 (transport) vs L7 (application) — key differences
- [ ] Algorithms: round-robin, least-connections, IP hash, weighted, random, p2c
- [ ] Azure LB (L4) vs Azure Application Gateway (L7) vs Azure Front Door (global L7)
- [ ] AWS NLB vs ALB — sticky sessions, timeouts, health checks
- [ ] Nginx Ingress: configuration snippets, lua, upstream keepalive
- [ ] Envoy: listeners, routes, clusters, filters, xDS API
- [ ] Connection draining / graceful shutdown: `preStop` hook + `terminationGracePeriodSeconds`
- [ ] Health checks: active vs passive, liveness vs readiness influence

### Ingress & Gateway API
- [ ] Ingress resource: rules, TLS, annotations (class-dependent)
- [ ] Ingress controllers: Nginx, Traefik, Kong, Contour comparison
- [ ] Gateway API: GatewayClass, Gateway, HTTPRoute, TLSRoute, ReferenceGrant
- [ ] Gateway API vs Ingress: why migrating makes sense

### Service Mesh
- [ ] Data plane vs control plane separation
- [ ] Istio: Envoy sidecar injection, VirtualService, DestinationRule, PeerAuthentication
- [ ] Istio traffic management: canary, weighted routing, fault injection, circuit breaker
- [ ] Linkerd: lightweight alternative, mRPC, no Envoy dependency
- [ ] Ambient mesh (Istio sidecarless): ztunnel, waypoint proxy
- [ ] Service mesh trade-offs: latency overhead, complexity, debuggability

## Key Troubleshooting Questions to Answer Confidently

- "A pod can't reach another service — how do you debug?" (DNS? kube-proxy? NetworkPolicy? CNI?)
- "TLS handshake fails between services — where do you look?"
- "Intermittent 5xx from load balancer — what are the possible causes?"

## Resources

- [The Book of Secret Knowledge — networking section](https://github.com/trimstray/the-book-of-secret-knowledge)
- [Ivan Velichko — container networking](https://iximiuz.com/)
- [Envoy Proxy documentation](https://www.envoyproxy.io/docs/)
- [cert-manager documentation](https://cert-manager.io/docs/)
