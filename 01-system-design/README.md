# System Design & Architecture

> ⭐ Priority area — expect 1–2 design rounds in every senior interview

## What to Expect

Senior DevOps / Platform interviews will ask you to design:
- CI/CD platforms at scale (1000s of microservices)
- Multi-region HA infrastructure
- Internal Developer Platforms (IDPs)
- Observability stacks
- Zero-downtime migration strategies
- Kubernetes platform with multi-tenancy

## Folder Structure

```
01-system-design/
├── concepts/           # Theory notes: CAP, BASE, patterns
├── case-studies/       # Actual design write-ups (your deliverables)
└── practice-problems/  # Problem prompts to practice from
```

## Key Concepts to Master

- [ ] CAP theorem, PACELC, consistency models (eventual, strong, causal)
- [ ] Availability patterns: failover, circuit breaker, bulkhead, retry with backoff
- [ ] Scaling: horizontal vs vertical, sharding, CDN, caching layers (Redis, Memcached)
- [ ] Load balancing: L4 vs L7, round-robin, least-connections, consistent hashing
- [ ] Message queues: Kafka (partitions, offsets, consumer groups), SQS/SNS
- [ ] Event-driven architecture: event sourcing, CQRS
- [ ] Multi-region: active-active vs active-passive, data residency, global traffic management
- [ ] Microservices: API gateway, service mesh, inter-service auth, versioning
- [ ] Data: read replicas, write-ahead logging, change data capture (CDC)

## Interview Framework (use every time)

```
1. Clarify requirements (functional + non-functional: scale, latency, availability)
2. Estimate scale (QPS, storage, bandwidth)
3. High-level design (boxes + arrows)
4. Deep dive on bottlenecks + trade-offs
5. Failure modes + observability
6. Cost & security considerations
```

## Resources

- [Awesome Scalability](https://github.com/binhnguyennus/awesome-scalability)
- *Designing Data-Intensive Applications* – Kleppmann
- [ByteByteGo](https://bytebytego.com/)
- [How They DevOps](https://github.com/bregman-arie/howtheydevops)
- [How They SRE](https://github.com/upgundecha/howtheysre)

## Case Studies (build these out)

| Problem | File | Status |
|---|---|---|
| CI/CD at scale — 1000+ microservices | [case-studies/cicd-at-scale.md](case-studies/cicd-at-scale.md) | 🔴 |
| Multi-region active-active AKS | [case-studies/multi-region-aks.md](case-studies/multi-region-aks.md) | 🔴 |
| Internal Developer Platform | [case-studies/internal-developer-platform.md](case-studies/internal-developer-platform.md) | 🔴 |
| Observability stack from scratch | [case-studies/observability-stack.md](case-studies/observability-stack.md) | 🔴 |
| Zero-downtime DB migration | [case-studies/zero-downtime-db-migration.md](case-studies/zero-downtime-db-migration.md) | 🔴 |
| Disaster recovery strategy | [case-studies/disaster-recovery.md](case-studies/disaster-recovery.md) | 🔴 |
