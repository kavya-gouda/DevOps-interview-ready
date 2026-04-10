# Kubernetes Internals & Troubleshooting

> ⭐ Priority area — K8s is the core platform for most product companies

## Folder Structure

```
02-kubernetes/
├── internals/          # Control plane, networking, storage deep dives
├── troubleshooting/    # Runbooks: failure scenarios + root cause + fix
├── security/           # RBAC, PSA, OPA/Kyverno, runtime security
└── practice-scenarios/ # Hands-on labs and scenario prompts
```

## Internals Coverage Checklist

### Control Plane
- [ ] kube-apiserver request flow (authentication → authorisation → admission → etcd)
- [ ] etcd: raft consensus, quorum, compaction, defrag
- [ ] kube-scheduler: predicates, priorities, scoring, extenders, gang scheduling
- [ ] controller-manager: reconciliation loop pattern, informers, work queues
- [ ] cloud-controller-manager: node lifecycle, LB provisioning

### Networking
- [ ] Pod networking: veth pairs, network namespace, bridge
- [ ] CNI: Cilium (eBPF, L4LB), Calico (BGP, IPIP), comparison
- [ ] kube-proxy modes: iptables vs IPVS vs eBPF bypass
- [ ] DNS: CoreDNS configuration, ndots, search domains, negative caching
- [ ] Network Policies: ingress/egress rules, default-deny, Cilium cluster-wide
- [ ] Ingress vs Gateway API (HTTPRoute, TLSRoute)
- [ ] Service types: ClusterIP, NodePort, LoadBalancer, ExternalName, Headless

### Storage
- [ ] CSI driver lifecycle: provisioning, attaching, mounting
- [ ] StorageClass: parameters, reclaimPolicy, volumeBindingMode
- [ ] Volume modes: Filesystem vs Block
- [ ] Dynamic provisioning flow
- [ ] StatefulSet storage: PVC templates, stable identities

### Workloads & Scheduling
- [ ] Pod lifecycle: pending → scheduled → init containers → running → termination
- [ ] Resource requests vs limits: CPU throttling vs OOMKilled
- [ ] QoS classes: Guaranteed, Burstable, BestEffort
- [ ] HPA: custom metrics, external metrics, stabilization window
- [ ] VPA: Recreate vs Auto mode, LimitRange interaction
- [ ] KEDA: ScaledObject, ScaledJob, Azure Queue / Kafka scalers
- [ ] Cluster Autoscaler: scale-up triggers, scale-down safety, priority expanders
- [ ] Karpenter: NodePool, NodeClaim, disruption budgets

## Troubleshooting Runbooks

| Symptom | Runbook | Status |
|---|---|---|
| CrashLoopBackOff | [troubleshooting/crash-loop-backoff.md](troubleshooting/crash-loop-backoff.md) | 🔴 |
| OOMKilled | [troubleshooting/oomkilled.md](troubleshooting/oomkilled.md) | 🔴 |
| ImagePullBackOff / ErrImagePull | [troubleshooting/image-pull-errors.md](troubleshooting/image-pull-errors.md) | 🔴 |
| Pod stuck in Pending | [troubleshooting/pending-pod.md](troubleshooting/pending-pod.md) | 🔴 |
| Pod stuck in Terminating | [troubleshooting/terminating-pod.md](troubleshooting/terminating-pod.md) | 🔴 |
| Service not reachable | [troubleshooting/service-unreachable.md](troubleshooting/service-unreachable.md) | 🔴 |
| DNS resolution failure | [troubleshooting/dns-failure.md](troubleshooting/dns-failure.md) | 🔴 |
| Node NotReady | [troubleshooting/node-not-ready.md](troubleshooting/node-not-ready.md) | 🔴 |
| etcd high latency / quorum loss | [troubleshooting/etcd-issues.md](troubleshooting/etcd-issues.md) | 🔴 |
| HPA not scaling | [troubleshooting/hpa-not-scaling.md](troubleshooting/hpa-not-scaling.md) | 🔴 |

## Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Kubernetes the Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way)
- [Awesome Cloud Native Trainings](https://github.com/joseadanof/awesome-cloudnative-trainings)
- Killer.sh CKA / CKS practice environments
- [Ivan Velichko — iximiuz.com](https://iximiuz.com/) – internals deep dives
