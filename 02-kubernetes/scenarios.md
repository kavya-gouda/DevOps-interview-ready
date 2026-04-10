# Kubernetes — 150 Scenarios

> **ID prefix:** K- | Types: Troubleshoot, Explain, Implement, Design | Difficulty: Medium → Expert

---

## Section 1: Control Plane Internals (K-001 to K-030)

---

### K-001 | kube-apiserver Request Flow
**Type:** Explain | **Difficulty:** Hard | ⬜

> Trace a `kubectl apply` command through every stage of the kube-apiserver pipeline.

**What you must cover:**
- Authentication: bearer token / client cert / OIDC → user identity established
- Authorization: RBAC — does this user have `create pods` in this namespace?
- Admission control: mutating webhooks (inject sidecar, set defaults) → validating webhooks (enforce policy)
- etcd write: serialized via protobuf, stored under `/registry/...` key
- Watch notification: controllers watching the resource type are notified via gRPC streaming watch
- Caching: apiserver has in-memory cache (watchcache) to avoid hitting etcd on every GET

---

### K-002 | etcd Raft Consensus Deep Dive
**Type:** Explain | **Difficulty:** Expert | ⬜

> Explain how etcd achieves consensus using Raft. What happens during a leader election? What is quorum and why does it matter for cluster sizing?

**What you must cover:**
- Raft: leader-based consensus. Leader receives all writes, replicates to followers.
- Leader election: follower times out waiting for heartbeat → becomes candidate → requests votes → majority wins
- Quorum: majority of nodes must acknowledge a write before it's committed. `floor(n/2)+1` nodes.
- 3-node etcd: quorum=2, can tolerate 1 failure. 5-node: quorum=3, tolerates 2.
- Network partition: minority partition blocks (refuses writes), majority partition continues
- Split-brain prevention: no node commits without quorum — durability guarantee
- etcd recommendation: always 3 or 5 nodes, never even numbers

---

### K-003 | etcd Compaction and Defrag
**Type:** Troubleshoot | **Difficulty:** Hard | ⬜

> Your etcd is consuming 6GB of storage and `etcdctl check perf` shows high latency. What do you do?

**What you must cover:**
- MVCC: etcd stores every revision of every key — grows indefinitely without compaction
- Compaction: `etcdctl compact $(etcdctl endpoint status --write-out json | jq '.[0].header.revision')` — removes old revisions
- Defragmentation: compaction releases logical space; defrag reclaims physical disk space: `etcdctl defrag --cluster`
- `--auto-compaction-retention`: set to `1h` or `1000` revisions — prevents runoff
- Quota: `--quota-backend-bytes=8589934592` (8GB) — when exceeded, etcd enters alarm state, rejecting writes
- Clearing alarm: after defrag — `etcdctl alarm disarm`
- Risk: never defrag all etcd members simultaneously — do them one at a time

---

### K-004 | kube-scheduler Deep Dive
**Type:** Explain | **Difficulty:** Expert | ⬜

> Explain every phase of the Kubernetes scheduler pipeline for a new pending pod.

**What you must cover:**
- Watch queue: scheduler watches for pods with `nodeName: ""` (unscheduled)
- Filtering (predicates): eliminate nodes that can't run the pod (insufficient resources, taints not tolerated, node selectors)
- Scoring (priorities): rank remaining nodes by score (resource availability, pod affinity, topology spread, image locality)
- Binding: scheduler writes `Binding` object to apiserver → kubelet picks up the assignment
- Scheduling queue: backoff queue for pods that fail repeatedly, priority queue for pod priority
- Extenders: webhook-based custom scheduler extensions
- Scheduler profile: configure which plugins are active per profile

---

### K-005 | Custom Scheduler Extender vs Custom Scheduler
**Type:** Design | **Difficulty:** Expert | ⬜

> You have a special GPU workload that needs custom scheduling logic (prioritize nodes with warm model cache). Should you write a scheduler extender or a custom scheduler? What are the trade-offs?

**What you must cover:**
- Extender: HTTP webhook called by default scheduler during filter/score phases. Simple integration, no Go required.
- Extender limitations: HTTP overhead on every scheduling decision, limited interface, must keep up with default scheduler
- Custom scheduler: separate binary, can handle all scheduling logic, can run alongside default scheduler (`schedulerName` field)
- Scheduling framework plugins: preferred — Go plugins that hook into specific scheduler extension points (Filter, Score, Reserve, Bind)
- Recommendation: scheduler framework plugin for performance-critical use; extender for quick iteration
- Risk: custom schedulers duplicate complex logic — prefer framework plugins

---

### K-006 | Informers and Controller Pattern
**Type:** Explain | **Difficulty:** Expert | ⬜

> Explain how Kubernetes controllers work internally — informers, work queues, and the reconciliation loop. Why is idempotency critical?

**What you must cover:**
- Informer: list+watch wrapper — initial LIST, then WATCH for events (add/update/delete)
- Local cache: informer maintains in-memory cache — controllers read from cache, not apiserver (reduces load)
- Event handler: OnAdd/OnUpdate/OnDelete callbacks push object key to work queue
- Work queue: rate-limited queue with retry/backoff (exponential backoff on error)
- Reconciliation: dequeue key → get object from cache → compare actual vs desired state → make changes
- Idempotency: reconcile() may run many times for same object — must be safe to re-run
- Optimistic concurrency: use `ResourceVersion` to detect conflicts before writing

---

### K-007 | Webhook Admission Controller
**Type:** Implement | **Difficulty:** Hard | ⬜

> Describe how to implement a mutating admission webhook that injects a label `environment: production` on all pods deployed to the `prod` namespace.

**What you must cover:**
- MutatingWebhookConfiguration: register webhook endpoint, namespace selector, resource (pods), operations (CREATE)
- Webhook server: HTTPS endpoint (TLS required), receives AdmissionReview JSON
- Mutation: return JSON patch (`[{"op":"add","path":"/metadata/labels/environment","value":"production"}]`)
- TLS: cert-manager issues certificate for webhook service, `caBundle` in webhook config
- Timeout: webhook must respond within `timeoutSeconds` (default 10s) or admission fails
- FailurePolicy: `Fail` (reject if webhook unavailable) vs `Ignore` (allow if unavailable)
- Testing: `kwok` or kind cluster + webhook integration test in CI

---

### K-008 | Leader Election for Controller HA
**Type:** Explain | **Difficulty:** Hard | ⬜

> How does a Kubernetes controller achieve high availability with leader election? What happens if the leader crashes?

**What you must cover:**
- `client-go` leader election: uses Lease resource (or ConfigMap/Endpoints for legacy)
- Leader holds lease: renews every `leaseDuration / 3` seconds
- Follower watches lease: if lease expires (leader crashed), contest for new lease
- Exactly-one-active: only leader runs reconciliation; followers are warm standby
- Lease parameters: `leaseDuration`, `renewDeadline`, `retryPeriod` — tune for failover speed vs spurious elections
- Failure scenario: leader crashes → followers detect expired lease within `leaseDuration` → new election → new leader starts reconciling
- kube-controller-manager: runs multiple controllers, each with independent leader election

---

### K-009 | Garbage Collection in Kubernetes
**Type:** Explain | **Difficulty:** Medium | ⬜

> Explain how Kubernetes garbage collection works. What are owner references and finalizers? Give a real scenario where a finalizer prevented accidental deletion.

**What you must cover:**
- Owner references: child object has `ownerReferences` pointing to parent — parent deleted → GC deletes children
- Cascading deletion: foreground (parent first) vs background (parent deleted, GC cleans children async) vs orphan
- Finalizers: strings on `.metadata.finalizers` — object not deleted until all finalizers removed
- Controller responsibility: controller that added finalizer must remove it after cleanup
- Real scenario: Persistent Volume Claim — finalizer prevents PVC deletion while pod is using it
- Real scenario: operator adds finalizer on custom resource — ensures external resource (Azure Key Vault) is cleaned up before K8s object deletion
- Stuck objects: finalizer added but controller crashed — must manually patch finalizers to unblock

---

### K-010 | Generation and ObservedGeneration
**Type:** Explain | **Difficulty:** Hard | ⬜

> Explain `metadata.generation` vs `status.observedGeneration` in Kubernetes. How do you use them in a controller to detect spec changes?

**What you must cover:**
- `metadata.generation`: incremented by apiserver on every spec change (not status)
- `status.observedGeneration`: last generation the controller successfully processed
- Pattern: controller checks `generation != observedGeneration` → spec has changed, needs reconciliation
- After reconcile: controller updates `status.observedGeneration = generation`
- Detecting in-progress: `generation > observedGeneration` means controller hasn't caught up
- Why not just compare spec: generation check is cheap, avoids deep spec diffing
- kubectl: `kubectl rollout status` uses this pattern to determine if a Deployment rollout is complete

---

### K-011 | API Server Request Throttling
**Type:** Troubleshoot | **Difficulty:** Hard | ⬜

> Your controllers are getting `429 TooManyRequests` errors from the apiserver. How do you diagnose and fix this?

**What you must cover:**
- API priority and fairness (APF): replaces per-client throttling — flow schemas, priority levels, queuing
- Diagnosis: `kubectl get flowschemas`, `kubectl get prioritylevelconfigurations`, `kubectl get --raw /metrics | grep apiserver_flowcontrol`
- Root cause: controller issuing too many LIST requests, not using informer cache properly
- Fix: ensure controller reads from informer cache (lister) instead of direct apiserver calls
- `resync-period`: full re-list on a timer — tune to 30m–1h, not the default which may be too frequent
- Workqueue: ensure rate-limited workqueue has proper backoff config

---

### K-012 | etcd Backup and Restore
**Type:** Implement | **Difficulty:** Expert | ⬜

> Walk through the exact commands to take an etcd snapshot and restore it to recover a cluster where the apiserver is down.

**What you must cover:**
```bash
# Snapshot
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%Y%m%d).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify snapshot
etcdctl snapshot status /backup/etcd-$(date +%Y%m%d).db --write-out=table

# Restore (on new/recovered node)
etcdctl snapshot restore /backup/etcd-$(date +%Y%m%d).db \
  --name=master-1 \
  --initial-cluster=master-1=https://10.0.0.1:2380 \
  --initial-cluster-token=etcd-cluster-1 \
  --initial-advertise-peer-urls=https://10.0.0.1:2380 \
  --data-dir=/var/lib/etcd-restore
```
- Update etcd manifest data-dir to restored path
- Restart etcd pod, verify cluster health
- For AKS: etcd is managed — use Velero + cluster backup for workload state

---

### K-013 | resourceVersion and Watch Semantics
**Type:** Explain | **Difficulty:** Expert | ⬜

> Explain resourceVersion in Kubernetes. What happens when a controller issues a WATCH with an old resourceVersion? What is the "too old resource version" error?

**What you must cover:**
- resourceVersion: monotonically increasing opaque token per resource — represents etcd revision
- WATCH `resourceVersion=X`: stream events since revision X (guaranteed ordering)
- "Too old resource version" (HTTP 410 Gone): compaction removed revisions older than X → client must restart with LIST + WATCH
- Client-go handles this transparently: on 410, reflector issues new LIST then WATCH from latest RV
- Watch bookmarks: `AllowWatchBookmarks=true` — periodic synthetic events to maintain watch RV position, reducing 410 risk
- Implications for controller restarts: informer re-lists and catches up from latest state

---

### K-014 | Webhook Performance Impact
**Type:** Design | **Difficulty:** Hard | ⬜

> Your cluster has 8 admission webhooks. Pod start time has increased by 3 seconds on average. Diagnose and fix.

**What you must cover:**
- Measurement: `kubectl get --raw '/metrics' | grep apiserver_admission_webhook_admission_duration`
- Identify slow webhook: histogram shows which webhook adds most latency
- Fix options: increase webhook container resources, optimize webhook logic (avoid external calls), cache responses
- `failurePolicy: Ignore` for non-critical webhooks (speed vs safety trade-off)
- `namespaceSelector` + `objectSelector`: narrow webhook scope — don't evaluate every pod in every namespace
- `timeoutSeconds`: reduce to 3–5s with `failurePolicy: Ignore` to prevent slow webhook blocking pods
- Parallel evaluation: mutating webhooks are sequential; validating are parallelized by apiserver

---

### K-015 | CRD Versioning and Conversion
**Type:** Explain | **Difficulty:** Expert | ⬜

> Your CRD has v1alpha1 and v1 versions. How do you migrate existing resources from v1alpha1 to v1 without breaking users?

**What you must cover:**
- CRD served versions: multiple versions served simultaneously, `storage: true` on newest
- Conversion webhook: translates between versions on read/write — must handle bi-directional conversion
- Hub-and-spoke: v1 is hub — all versions convert through v1 internally
- Migration strategy: add v1 served alongside v1alpha1, update clients to use v1, then remove v1alpha1 after migration window
- `kubectl convert` (deprecated): use conversion webhook instead
- Validation markers: new fields in v1 can have required validation not present in v1alpha1 — handle gracefully

---

## Section 2: Workloads and Scheduling (K-016 to K-040)

---

### K-016 | Pod Lifecycle Complete Walkthrough
**Type:** Explain | **Difficulty:** Medium | ⬜

> Walk through the complete lifecycle of a pod from `kubectl apply` to the pod serving traffic.

**What you must cover:**
- kubectl apply → apiserver validates → stores in etcd → event emitted
- kube-scheduler assigns pod to node (`nodeName` set in etcd)
- kubelet on that node detects pod via watch → pulls images → runs init containers
- Init containers: run sequentially, must complete successfully before main containers start
- Main containers: started, readinessProbe begins polling, pod in `Running` state but not yet Ready
- Service: `Ready` condition true → kube-proxy/iptables rules updated → traffic starts routing
- StartupProbe: protects slow-starting apps before liveness probe kicks in

---

### K-017 | Taints and Tolerations — Design Patterns
**Type:** Design | **Difficulty:** Medium | ⬜

> Design a taint/toleration strategy for a cluster with 4 node types: general workloads, GPU workloads, spot instances, and system components.

**What you must cover:**
```yaml
# GPU nodes
kubectl taint nodes gpu-node-1 nvidia.com/gpu=present:NoSchedule
# GPU pods tolerate: key: nvidia.com/gpu, operator: Exists, effect: NoSchedule

# Spot nodes
kubectl taint nodes spot-1 spot=true:NoSchedule 
# Spot-tolerant pods in Deployment spec + PodDisruptionBudget

# System node pool (for critical DaemonSets only)
kubectl taint nodes sys-1 CriticalAddonsOnly=true:NoSchedule
# kube-proxy, CNI, CSI DaemonSets have this toleration built-in
```
- Node affinity: use in addition to taints — ensures workload is scheduled to correct node type even without dedicated taint pool
- PreferNoSchedule: soft taint — prefer to avoid but allow if no other nodes available
- NoExecute: evicts already-running pods that don't tolerate

---

### K-018 | TopologySpreadConstraints Deep Dive
**Type:** Implement | **Difficulty:** Hard | ⬜

> Design topology spread constraints for a 5-replica stateless service that must spread across 3 AZs evenly and across nodes within each AZ.

**What you must cover:**
```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: my-service
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: my-service
```
- `maxSkew: 1` — allows at most 1 pod difference between zones
- `DoNotSchedule` vs `ScheduleAnyway`: strictness vs flexibility during rolling updates
- Pod anti-affinity: alternative but less flexible — can block scheduling when nodes are full
- `minDomains` (K8s 1.28): require minimum number of topology domains

---

### K-019 | ResourceQuota and LimitRange Strategy
**Type:** Design | **Difficulty:** Medium | ⬜

> Design a ResourceQuota + LimitRange strategy for a shared cluster with 20 teams, ensuring no single team starves others.

**What you must cover:**
```yaml
# ResourceQuota per namespace
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "100"
    persistentvolumeclaims: "20"

# LimitRange — defaults for pods with no resources set
spec:
  limits:
  - type: Container
    default: {cpu: 200m, memory: 256Mi}
    defaultRequest: {cpu: 100m, memory: 128Mi}
    max: {cpu: "4", memory: 8Gi}
```
- Without LimitRange: pods with no resource spec can consume unlimited resources in quota namespace
- Namespace quotas: enforce via `kubectl describe resourcequota -n team-a`
- Alert: Prometheus alert when team is at 80% of quota
- Node pressure: ResourceQuota doesn't reserve nodes — need capacity planning on top

---

### K-020 | Priority Classes and Preemption
**Type:** Design | **Difficulty:** Hard | ⬜

> The cluster is under pressure. Define a PriorityClass hierarchy for a platform and explain how preemption works.

**What you must cover:**
```yaml
# Platform critical (system components)
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata: {name: platform-critical}
value: 1000000
globalDefault: false

# Production workloads
value: 100000

# Staging workloads  
value: 10000

# Batch/dev workloads
value: 1000
```
- Preemption: if a high-priority pod can't schedule, scheduler evicts lower-priority pods to make room
- `preemptionPolicy: Never` on non-preempting priority class (queue position only, no eviction)
- Risk: overly aggressive preemption can destabilize cluster — use PodDisruptionBudgets on critical services
- Critical pods: `system-cluster-critical`, `system-node-critical` are built-in highest priorities

---

### K-021 | Horizontal Pod Autoscaler — Custom Metrics
**Type:** Implement | **Difficulty:** Hard | ⬜

> Your service should scale based on the number of messages in an Azure Service Bus queue, not CPU. Implement HPA with custom metrics.

**What you must cover:**
- KEDA preferred: ScaledObject CRD — richer than raw HPA, scale-to-zero support
```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
spec:
  scaleTargetRef:
    name: order-processor
  minReplicaCount: 0
  maxReplicaCount: 20
  triggers:
  - type: azure-servicebus
    metadata:
      queueName: orders
      queueLength: "100"
    authenticationRef:
      name: azure-servicebus-auth
```
- If raw HPA: requires `external.metrics.k8s.io` API via custom metrics adapter (KEDA provides this)
- Scaling formula: current replicas × (current metric / desired metric)
- Stabilization window: prevent rapid scale down on temporary queue drain

---

### K-022 | Vertical Pod Autoscaler Deep Dive
**Type:** Explain | **Difficulty:** Hard | ⬜

> Explain VPA modes, limitations, and how to use it safely in production without causing unnecessary pod disruptions.

**What you must cover:**
- VPA modes: `Off` (recommendations only), `Initial` (set at pod creation only), `Auto` (can evict + resize)
- `Auto` mode impact: VPA may evict pods to resize — dangerous with stateful workloads
- Goldilocks: run VPA in `Off` mode cluster-wide, use Goldilocks dashboard to show recommendations per namespace
- Resources: VPA sets both `requests` AND `limits` — check VPA limits policy (`containerResourcePolicy`)
- VPA + HPA conflict: don't use both on CPU/memory simultaneously. Use KEDA for custom metric HPA + VPA for right-sizing.
- Namespace: VPA must have the `VerticalPodAutoscaler` CRD installed (install VPA admission controller separately)

---

### K-023 | Cluster Autoscaler Behavior
**Type:** Troubleshoot | **Difficulty:** Hard | ⬜

> New pods are pending but Cluster Autoscaler isn't adding nodes. Diagnose.

**What you must cover:**
- Check CA logs: `kubectl logs -n kube-system deployment/cluster-autoscaler`
- Common reasons: pod has special node requirements (taint, large resource request) that no node group can satisfy
- Node group max reached: CA won't exceed nodegroup's `maxSize` setting
- Pod not schedulable for reasons CA can't fix: `PodAffinityConflict`, `VolumeZoneConflict`
- `cluster-autoscaler.kubernetes.io/safe-to-evict: "false"` — pod blocking scale-down may block scale-up flow (check for this)
- Pod has local storage (`emptyDir`): CA defaults to not evicting these — blocks scale-down which can affect burst
- Fix: check `ExpanderStrategy`, check node group annotations, verify resource headroom

---

### K-024 | StatefulSet vs Deployment — When to Use Each
**Type:** Explain | **Difficulty:** Medium | ⬜

> Explain the differences between StatefulSet and Deployment and give 3 production scenarios where StatefulSet is required.

**What you must cover:**
- StatefulSet guarantees: stable network identity (`<name>-0`, `<name>-1`), stable storage (PVC per pod), ordered deployment/scaling
- Deployment: any pod can replace any other, shared storage (ReadWriteMany), stateless
- Scenario 1: Kafka broker — each broker needs stable hostname for other brokers to find it
- Scenario 2: Elasticsearch data node — each node has dedicated shard storage
- Scenario 3: Distributed database (Redis cluster) — each node holds specific data partitions
- Headless service: ClusterIP: None — DNS returns individual pod IPs instead of single VIP
- Deletion order: StatefulSet deletes in reverse order (N → 0), critical for quorum systems

---

### K-025 | DaemonSet Use Cases and Limitations
**Type:** Explain | **Difficulty:** Medium | ⬜

> Explain DaemonSet and 5 production use cases. What happens to DaemonSet pods during node lifecycle events (cordon, drain, upgrade)?

**What you must cover:**
- DaemonSet: one pod per node (or per subset of nodes)
- Use case 1: Fluent Bit log shipping agent (needs access to host log files)
- Use case 2: Prometheus node-exporter (needs access to host metrics)
- Use case 3: Cilium/Calico CNI networking agent (must run on every node)
- Use case 4: NVIDIA GPU device plugin
- Use case 5: Falco runtime security agent
- Node drain: DaemonSet pods are NOT evicted by `kubectl drain` by default — use `--ignore-daemonsets`
- Node cordon: DaemonSet pods already running are NOT removed; new nodes get DaemonSet pod scheduled

---

### K-026 | Job and CronJob Best Practices
**Type:** Design | **Difficulty:** Medium | ⬜

> Design a CronJob that runs a database backup every 4 hours. What are the concurrency and failure handling considerations?

**What you must cover:**
```yaml
spec:
  schedule: "0 */4 * * *"
  concurrencyPolicy: Forbid  # Don't start new job if previous still running
  failedJobsHistoryLimit: 3
  successfulJobsHistoryLimit: 3
  startingDeadlineSeconds: 300  # Don't start if missed by > 5 min
  jobTemplate:
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 3600  # Job must complete within 1h
      template:
        spec:
          restartPolicy: OnFailure
```
- `ConcurrencyPolicy: Forbid` vs `Replace` vs `Allow` — know when each is correct
- Job history: K8s keeps completed job pods — must set history limits to avoid pod accumulation
- Alerting: alert on `kube_job_status_failed > 0` for any job in the namespace

---

### K-027 | Pod Disruption Budgets Design
**Type:** Design | **Difficulty:** Hard | ⬜

> Design PodDisruptionBudgets for a set of services that must maintain minimum availability during cluster upgrades.

**What you must cover:**
```yaml
# For a 3-replica API service — always keep 2 running
apiVersion: policy/v1
kind: PodDisruptionBudget
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: api-service

# Alternative: maxUnavailable
spec:
  maxUnavailable: 1
```
- `minAvailable` vs `maxUnavailable`: minAvailable takes precedence, absolute or percentage
- PDB with StatefulSet: `maxUnavailable: 1` for quorum-based systems (3-node k8s: `maxUnavailable: 1`)
- Upgrade behavior: `kubectl drain` respects PDB — will wait for pod to be replaced before draining
- Danger: PDB too strict (`minAvailable: 3` on 3-replica service) blocks drains permanently — plan for cluster upgrades

---

### K-028 | Init Container Patterns
**Type:** Implement | **Difficulty:** Medium | ⬜

> Give 3 production init container patterns and implement each.

**What you must cover:**
- Pattern 1: Wait for dependency (database ready check)
```yaml
initContainers:
- name: wait-for-db
  image: postgres:15-alpine
  command: ['sh', '-c', 'until pg_isready -h db.svc -p 5432; do sleep 2; done']
```
- Pattern 2: Schema migration runner (runs once before app starts)
- Pattern 3: Config file transformation (render template from Vault secret → file → main container mounts volume)
- Init containers share volumes with main container — useful for handoff
- Init container failure: pod stays in `Init:Error` — main container never starts
- Init containers run sequentially — each must complete before next starts

---

### K-029 | Resource Request vs Limit — CPU Throttling
**Type:** Explain | **Difficulty:** Hard | ⬜

> Explain why setting CPU limits too low causes performance problems even when total cluster CPU is available. What's the recommended strategy?

**What you must cover:**
- CPU requests: used for scheduling decisions — node allocated this amount
- CPU limits: enforced by CFS (Completely Fair Scheduler) via cgroups — hard cap
- CPU throttling: process paused when it exceeds limit, even if physical CPU is idle
- Metric: `container_cpu_throttled_seconds_total` / `container_cpu_cfs_throttled_periods_total` — alert if >25%
- OOMKilled: memory limit exceeded → process killed immediately
- Recommendation: always set memory limits (prevents OOM cascade). Consider no CPU limit (or very high) and rely on requests for scheduling.
- Java: JVM default heap = 1/4 of container memory limit — must set `-Xmx` or use `-XX:+UseContainerSupport`

---

### K-030 | QoS Classes and Eviction Policy
**Type:** Explain | **Difficulty:** Hard | ⬜

> Explain QoS classes in Kubernetes. In a memory pressure situation, which pods are evicted first and why?

**What you must cover:**
- Guaranteed: requests == limits for all containers — last to be evicted
- Burstable: requests set, limits ≠ requests OR only one of request/limit set — middle priority
- BestEffort: no requests or limits — first to be evicted
- Eviction order under memory pressure: BestEffort → Burstable (farthest over request) → Guaranteed
- Node pressure eviction: kubelet evicts when node hits `eviction.hard` threshold (default: `memory.available < 100Mi`)
- Eviction vs OOMKill: eviction is graceful, OOMKill is immediate. Node OOM killer bypasses K8s — prevent with proper limits.
- `kubectl describe node`: check `Conditions` for `MemoryPressure: True`

---

## Section 3: Networking Internals (K-031 to K-060)

---

### K-031 | Pod-to-Pod Networking Without CNI
**Type:** Explain | **Difficulty:** Expert | ⬜

> Explain at the kernel level what happens when Pod A sends a packet to Pod B on the same node — with no CNI, just the Linux network stack.

**What you must cover:**
- Each pod has its own network namespace, created by container runtime
- `veth pair`: one end in pod namespace (`eth0`), other end on host (`vethXXX`)
- `cbr0` / `docker0` bridge: host-side veth endpoints connected to a bridge
- Pod A → `eth0` → `vethA` → `cbr0` bridge → `vethB` → `eth0` Pod B
- ARP: Pod A ARPs for Pod B's IP, cbr0 responds with Pod B's MAC
- Same subnet: pods on same node are in the same subnet — direct bridge forwarding
- Different nodes: traffic leaves via host `eth0` → needs CNI for cross-node routing

---

### K-032 | CNI Plugin Comparison
**Type:** Explain | **Difficulty:** Expert | ⬜

> Compare Cilium, Calico, and Azure CNI for production AKS usage. When would you choose each?

**What you must cover:**
- Calico: BGP-based routing, mature IPAM, `ipipMode=Always` for cross-subnet, excellent NetworkPolicy support
- Cilium: eBPF-based data plane (replaces iptables/kube-proxy), superior performance, L7 network policies, Hubble for observability, best choice for high-traffic clusters
- Azure CNI: pods get VNet IPs (not overlaid), native Azure networking, required for some Azure features (Application Gateway Ingress), IP exhaustion risk  
- Azure CNI Overlay: pods get IPs from pod CIDR (not VNet), better scaling, no IP exhaustion
- CNI + Cilium on AKS: Microsoft supports Cilium as AKS data plane overlay (2023+)
- Trade-offs: Cilium = best perf/features, complex to debug; Calico = stable, widely known; Azure CNI = Azure-native, simpler networking model

---

### K-033 | kube-proxy iptables vs IPVS vs eBPF
**Type:** Explain | **Difficulty:** Expert | ⬜

> Explain how kube-proxy implements Services in iptables mode vs IPVS mode. Why is Cilium's eBPF approach faster?

**What you must cover:**
- iptables: linear rule traversal — `iptables -L -n` shows KUBE-SERVICES chain → KUBE-SVC-XXX → KUBE-SEP-XXX per endpoint
- iptables scale problem: 10,000 services = 10,000+ iptables rules — O(n) lookup, slow updates
- IPVS: hash table lookup O(1), purpose-built for L4 LB — much better for large clusters
- eBPF (Cilium): bypass kernel networking stack entirely — sockmap for pod-to-pod, direct packet manipulation, no iptables at all
- Performance comparison: iptables 100K rules = 100% CPU for simple packet forward; eBPF = 1% CPU
- Enabling IPVS mode: `kube-proxy --proxy-mode=ipvs` (requires `ip_vs` kernel modules)
- kube-proxy replacement: `--set kubeProxyReplacement=true` in Cilium (removes kube-proxy entirely)

---

### K-034 | Service Types Deep Dive
**Type:** Explain | **Difficulty:** Medium | ⬜

> Compare ClusterIP, NodePort, LoadBalancer, ExternalName, and Headless services. Give a scenario for each.

**What you must cover:**
- ClusterIP: internal VIP, within-cluster only. Scenario: backend service not exposed externally.
- NodePort: exposes on all nodes at static port (30000–32767). Scenario: dev/test, or with external LB doing the round-robin.
- LoadBalancer: provisions cloud LB (Azure LB, AWS NLB). Scenario: internet-facing API.
- ExternalName: CNAME to external hostname. Scenario: abstract an external database DNS behind a K8s Service name.
- Headless (ClusterIP:None): DNS returns pod IPs directly. Scenario: StatefulSet — each pod addressable by `<name>-0.svc.cluster.local`.
- `externalTrafficPolicy: Local` on LoadBalancer/NodePort: preserve source IP but adds load imbalance risk

---

### K-035 | CoreDNS Configuration and Troubleshooting
**Type:** Troubleshoot | **Difficulty:** Hard | ⬜

> Pods are experiencing slow DNS resolution (~1 second for internal lookups). Diagnose.

**What you must cover:**
- `ndots: 5` default: for short hostname `redis`, try `redis.namespace.svc.cluster.local`, `redis.svc.cluster.local`, `redis.cluster.local`, `redis.search.domain`, then `redis` — 5 DNS lookups before success
- Fix: set `ndots: 2` or use FQDN (`redis.namespace.svc.cluster.local.`)
- Negative caching: NXDOMAIN responses cached — if wrong search domain, repeated failures
- CoreDNS metrics: `coredns_dns_request_duration_seconds`, `coredns_dns_responses_total` by rcode
- CoreDNS affinity: if CoreDNS pod is far from pod (different node), add latency
- `dnsPolicy: None` + custom `dnsConfig`: full control for specific pods
- Conntrack table issue: conntrack race condition causes random DNS failures — mitigated by Cilium eBPF DNS proxy

---

### K-036 | NetworkPolicy — Default Deny and Exceptions
**Type:** Implement | **Difficulty:** Hard | ⬜

> Implement a comprehensive NetworkPolicy for a namespace that implements default-deny and allows: DNS, liveness probes from kubelet, Prometheus scraping, and internal app-to-db traffic.

**What you must cover:**
```yaml
# Default deny all ingress + egress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]

# Allow DNS egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
  - ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP

# Allow app → db
---
spec:
  podSelector:
    matchLabels: {role: frontend}
  policyTypes: [Egress]
  egress:
  - to:
    - podSelector:
        matchLabels: {role: db}
    ports:
    - port: 5432
```
- Kubelet probe traffic: comes from node IP — use `ipBlock` with node CIDR
- Prometheus scrape: ingress from monitoring namespace pod selector

---

### K-037 | Ingress Controller Deep Dive
**Type:** Explain | **Difficulty:** Hard | ⬜

> Explain how NGINX Ingress Controller works internally. How does it handle TLS termination and upstream health checks?

**What you must cover:**
- Controller watches Ingress resources → generates nginx.conf → reloads NGINX
- `nginx.conf` template: upstream block per Service backend, server block per Ingress rule
- TLS termination: cert from Secret (`kubernetes.io/tls` type) → mounted in pod → nginx ssl block
- Upstream keepalive: persistent connections to backend pods (avoids TCP setup per request)
- Health checks: NGINX actively checks backends via `keepalive` + passive detection (5xx triggers removal)
- Annotations: control NGINX behavior (`proxy_read_timeout`, `proxy_body_size`, `rewrite-target`)
- Session affinity: `nginx.ingress.kubernetes.io/affinity: cookie` → consistent hashing via NGINX cookie
- Hot reload: NGINX config reload without dropped connections (`nginx -s reload` uses graceful reload)

---

### K-038 | Gateway API vs Ingress
**Type:** Explain | **Difficulty:** Hard | ⬜

> Explain why the Kubernetes Gateway API was created to replace Ingress. What are the key new capabilities?

**What you must cover:**
- Ingress limitations: single resource, too many vendor-specific annotations, no traffic splitting, single namespace
- Gateway API: role-based — infrastructure (GatewayClass), cluster ops (Gateway), app team (HTTPRoute)
- HTTPRoute: header-based matching, traffic splitting (canary), retry policies — all first-class, no annotations
- Cross-namespace routing: HTTPRoute in app namespace, Gateway in infra namespace — `ReferenceGrant` enables this
- Protocol support: HTTPRoute, TCPRoute, TLSRoute, GRPCRoute — Ingress only handles HTTP/HTTPS
- Implementations: Istio, Cilium, Contour, nginx-gateway-fabric, Envoy Gateway
- Migration: Ingress still valid, not deprecated, but new features won't be added

---

### K-039 | Service Mesh Trade-offs in Depth
**Type:** Architecture | **Difficulty:** Expert | ⬜

> Your team is debating Istio vs not using a service mesh at all. Make the case for both sides with concrete data.

**What you must cover:**
- Case for Istio: mTLS (zero-trust networking), uniform observability (RED metrics per service without code change), traffic management (canary, circuit breaker), L7 authorization policies
- Case against: ~30–50MB RAM per sidecar (1000 pods = 30–50GB RAM just for sidecars), ~3ms latency per hop, complex debugging, Envoy config is arcane, upgrade coordination
- Ambient mesh (Istio 1.21+): removes sidecar overhead — ztunnel node agent for mTLS, waypoint proxy only for L7 features
- Alternative to full mesh: mTLS via cert-manager + SPIFFE, observability via OTel SDK, traffic management via Argo Rollouts + NGINX — explicit but lower overhead
- Decision: service mesh justified for 50+ services with strict zero-trust requirements; overhead unjustified for small clusters

---

### K-040 | Multi-Cluster Service Discovery
**Type:** Design | **Difficulty:** Expert | ⬜

> Services in cluster A need to call services in cluster B. Design a service discovery and routing solution.

**What you must cover:**
- Istio multi-cluster: flat network (shared pod CIDR peered) or gateways (east-west gateway for cross-cluster traffic)
- Submariner: connects cluster pod and service CIDRs via encrypted tunnel (WireGuard or IPsec)
- Skupper / Service mirroring: mirror remote service as local Service in each cluster — transparent to application
- DNS federated: `*.cluster-b.svc.cluster.local` routed to cluster B's DNS proxy
- Global load balancer: expose services via global LB (Azure Traffic Manager, Cloudflare) — HTTP-level routing between clusters
- mTLS between clusters: SPIFFE identity federation (trust bundle exchange), Istio PeerAuthentication cross-cluster

---

## Section 4: Storage (K-041 to K-055)

---

### K-041 | CSI Driver Lifecycle
**Type:** Explain | **Difficulty:** Hard | ⬜

> Explain how a CSI driver provisions a PersistentVolume end-to-end — from PVC creation to pod mounting.

**What you must cover:**
- PVC created → PV controller (external-provisioner sidecar watches PVCs, calls CSI `CreateVolume`)
- Azure Disk created in Azure → PV object created with Azure Disk resource ID
- Attacher (external-attacher sidecar): calls `ControllerPublishVolume` → disk attached to node (Azure VM)
- Mounter (kubelet plugin): calls `NodeStageVolume` (format + stage to global mount point) → `NodePublishVolume` (bind mount into pod)
- Pod starts: container sees `/data` as a mounted filesystem
- Deletion: reverse: `NodeUnpublishVolume` → `NodeUnstageVolume` → `ControllerUnpublishVolume` → `DeleteVolume`
- gRPC API: CSI driver implements Identity, Controller, Node gRPC services

---

### K-042 | Storage Class Parameters for Production
**Type:** Implement | **Difficulty:** Hard | ⬜

> Design StorageClasses for 3 use cases: high-IOPS database (Azure Premium SSD), bulk storage (Azure Standard HDD), and shared read-write (Azure Files NFS).

**What you must cover:**
```yaml
# High-IOPS database storage
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: premium-ssd
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS
  kind: Managed
  cachingMode: ReadOnly
reclaimPolicy: Retain  # Don't auto-delete production data
volumeBindingMode: WaitForFirstConsumer  # Schedule pod first, then provision in same AZ
allowVolumeExpansion: true

# Azure Files NFS for ReadWriteMany
provisioner: file.csi.azure.com
parameters:
  skuName: Premium_LRS
  protocol: nfs
```
- `WaitForFirstConsumer`: critical for multi-AZ clusters — provision in same zone as pod
- `Retain` vs `Delete`: use `Retain` for production data — manual cleanup prevents accidents
- `allowVolumeExpansion: true`: can resize PVC by editing resource request (no downtime for most CSI drivers)

---

### K-043 | PVC Expansion and Volume Resize
**Type:** Troubleshoot | **Difficulty:** Medium | ⬜

> A pod's PVC is 80% full. Walk through the steps to expand it without downtime.

**What you must cover:**
```bash
# Check StorageClass allows expansion
kubectl get sc premium-ssd -o yaml | grep allowVolumeExpansion

# Edit PVC (increase size)
kubectl patch pvc my-data -p '{"spec":{"resources":{"requests":{"storage":"50Gi"}}}}'

# Monitor expansion
kubectl describe pvc my-data  # Conditions: FileSystemResizePending → Resizing → Bound
kubectl get pvc my-data  # CAPACITY should update after filesystem resize
```
- CSI driver: `ControllerExpandVolume` (expand underlying disk) → `NodeExpandVolume` (resize filesystem in pod)
- Pod restart: Azure Disk requires NodeExpandVolume which happens when pod is running — usually no restart needed
- Alert: alert on `kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes > 0.8`
- Automatic expansion: VolumeAutoScaler project (not built-in K8s)

---

### K-044 | StatefulSet Storage Patterns
**Type:** Design | **Difficulty:** Hard | ⬜

> Design a StatefulSet deployment for a 3-node Kafka cluster on AKS with dedicated PVCs per broker.

**What you must cover:**
```yaml
apiVersion: apps/v1
kind: StatefulSet
spec:
  serviceName: kafka
  replicas: 3
  podManagementPolicy: OrderedReady
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ReadWriteOnce]
      storageClassName: premium-ssd
      resources:
        requests:
          storage: 500Gi
```
- `volumeClaimTemplates`: each pod (`kafka-0`, `kafka-1`, `kafka-2`) gets its own PVC (`data-kafka-0`, `data-kafka-1`, `data-kafka-2`)
- PVC lifecycle: PVCs are NOT deleted when StatefulSet is deleted — intentional (data preservation)
- `podManagementPolicy: Parallel`: all pods start simultaneously (RollingUpdate still sequential with Parallel)
- Deletion: to clean up, delete StatefulSet + PVCs manually

---

### K-045 | Volume Snapshot and Clone
**Type:** Implement | **Difficulty:** Hard | ⬜

> Walk through taking a volume snapshot and creating a new PVC from that snapshot — for use as a pre-populated test database.

**What you must cover:**
```yaml
# VolumeSnapshot
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: db-snapshot-2026-04-07
spec:
  volumeSnapshotClassName: csi-azure-vsc
  source:
    persistentVolumeClaimName: postgres-data

# PVC from snapshot
apiVersion: v1
kind: PersistentVolumeClaim
spec:
  dataSource:
    kind: VolumeSnapshot
    name: db-snapshot-2026-04-07
    apiGroup: snapshot.storage.k8s.io
  resources:
    requests:
      storage: 100Gi
  storageClassName: premium-ssd
```
- VolumeSnapshotClass: CSI driver-specific (`csi-azure-vsc`)
- VolumeSnapshotContent: provisioned by CSI driver, bound to VolumeSnapshot
- Use case: nightly snapshot of prod DB → restore to staging for testing

---

## Section 5: Security (K-046 to K-065)

---

### K-046 | RBAC Design for Multi-Tenant Cluster
**Type:** Design | **Difficulty:** Hard | ⬜

> Design an RBAC model for a cluster with 20 teams, where each team should manage their own namespace resources but cannot access other teams' namespaces or cluster-level resources.

**What you must cover:**
```yaml
# Team-scoped Role (per namespace)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: team-a
  name: team-developer
rules:
- apiGroups: ["", "apps", "networking.k8s.io"]
  resources: ["pods", "deployments", "services", "ingresses", "configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]  # View only — no create/delete on secrets

# RoleBinding: bind to team's Azure AD group
kind: RoleBinding
subjects:
- kind: Group
  name: "team-a-developers@company.com"
  apiGroup: rbac.authorization.k8s.io
```
- Audit: `kubectl auth can-i create pods --as=system:serviceaccount:team-a:default -n team-b` — verify isolation
- ClusterRole + RoleBinding: ClusterRole defines permissions, RoleBinding scopes to namespace
- No wildcards (`*`): least privilege — enumerate allowed resources explicitly

---

### K-047 | ServiceAccount Best Practices
**Type:** Implement | **Difficulty:** Hard | ⬜

> Describe the security risks of the default ServiceAccount and implement a least-privilege ServiceAccount for a deployment that only needs to read ConfigMaps.

**What you must cover:**
```yaml
# Create dedicated SA
apiVersion: v1
kind: ServiceAccount
metadata:
  name: config-reader
  namespace: app
automountServiceAccountToken: false  # Opt-out globally

# Role: only read configmaps
kind: Role
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "watch"]

# Deployment
spec:
  serviceAccountName: config-reader
  automountServiceAccountToken: true  # Opt-in at pod level
```
- Default SA risk: `default` SA often has no RBAC but token is automounted — attack surface if pod is compromised
- Projected service account tokens: short-lived (1h) JWT, audience-specific — safer than legacy secret-based tokens
- Workload Identity (AKS): federate SA with Azure AD managed identity — no long-lived tokens, access Azure resources natively

---

### K-048 | Pod Security Standards Implementation
**Type:** Implement | **Difficulty:** Hard | ⬜

> Enable Pod Security Admission at the cluster level. Implement `restricted` policy for app namespaces and `privileged` for system namespaces.

**What you must cover:**
```yaml
# Namespace labels for Pod Security Admission
kubectl label namespace app-team-a \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/audit=restricted

kubectl label namespace kube-system \
  pod-security.kubernetes.io/enforce=privileged
```
- `restricted` requires: runAsNonRoot, no privileged, drop ALL capabilities, seccompProfile RuntimeDefault, no hostPath, no hostNetwork/PID/IPC
- Start with `warn` + `audit` modes in existing namespaces — identify violations before enforcing
- Kyverno alternative: more granular control, per-resource exceptions, better reporting
- `enforce` mode: pods violating policy are rejected at admission — workloads stop deploying until fixed

---

### K-049 | Workload Identity (AKS)
**Type:** Implement | **Difficulty:** Hard | ⬜

> Walk through configuring AKS Workload Identity to give a pod access to Azure Key Vault without storing any credentials.

**What you must cover:**
```bash
# Enable OIDC + Workload Identity on AKS cluster
az aks update --enable-oidc-issuer --enable-workload-identity -n myaks -g myrg

# Create managed identity
az identity create --name app-identity --resource-group myrg

# Federate: K8s SA → Azure Managed Identity
az identity federated-credential create \
  --name my-app-federated-cred \
  --identity-name app-identity \
  --issuer $(az aks show -n myaks -g myrg --query oidcIssuerProfile.issuerUrl -o tsv) \
  --subject system:serviceaccount:default:my-app-sa

# Grant Key Vault access to managed identity
az role assignment create --role "Key Vault Secrets User" \
  --assignee $(az identity show -n app-identity --query clientId -o tsv) \
  --scope $(az keyvault show -n mykeyvault --query id -o tsv)
```
- Pod annotation: `azure.workload.identity/client-id: <managed-identity-client-id>`
- SA annotation: `azure.workload.identity/client-id: <managed-identity-client-id>`
- Token exchange: Azure AD OIDC validates K8s SA projected token → issues Azure access token

---

### K-050 | OPA Gatekeeper Policy Examples
**Type:** Implement | **Difficulty:** Expert | ⬜

> Write an OPA Gatekeeper ConstraintTemplate that requires all Deployments to have a `team` label.

**What you must cover:**
```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: requireteamlabel
spec:
  crd:
    spec:
      names:
        kind: RequireTeamLabel
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package requireteamlabel
      violation[{"msg": msg}] {
        not input.review.object.metadata.labels.team
        msg := "Deployment must have a 'team' label"
      }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: RequireTeamLabel
metadata:
  name: require-team-label-deployments
spec:
  match:
    kinds:
    - apiGroups: ["apps"]
      kinds: ["Deployment"]
  enforcementAction: deny
```
- `enforcementAction: warn` first for audit, then `deny` after fixing violations
- `kubectl get constraints` — shows violations
- Mutation policies: Gatekeeper Assign CRD can automatically add default labels

---

### K-051 | Kyverno Policy Examples
**Type:** Implement | **Difficulty:** Hard | ⬜

> Write a Kyverno policy that automatically adds `istio-injection: enabled` label to new namespaces and disallows containers running as root.

**What you must cover:**
```yaml
# Add label to new namespaces
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-istio-injection
spec:
  rules:
  - name: add-label
    match:
      any:
      - resources: {kinds: [Namespace]}
    mutate:
      patchStrategicMerge:
        metadata:
          labels:
            istio-injection: enabled

# Disallow root containers (validation)
  - name: disallow-root
    match:
      any:
      - resources: {kinds: [Pod]}
    validate:
      message: "Containers must not run as root"
      pattern:
        spec:
          containers:
          - (name): "*"
            securityContext:
              runAsNonRoot: true
```
- `generate` rule: create NetworkPolicy/ResourceQuota when new namespace is created
- `PolicyException`: namespace-scoped exception for specific workloads
- `kyverno test`: unit test policies in CI

---

### K-052 | Falco Rules in Practice
**Type:** Implement | **Difficulty:** Hard | ⬜

> Write a Falco rule that alerts when a container reads sensitive files (`/etc/shadow`, `/etc/passwd`) at runtime.

**What you must cover:**
```yaml
- rule: Read Sensitive File Untrusted
  desc: Attempt to read sensitive files without privilege
  condition: >
    open_read
    and sensitive_files
    and not trusted_containers
    and not proc_is_new_exe_thread
  output: >
    Sensitive file opened for reading by untrusted program
    (user=%user.name command=%proc.cmdline file=%fd.name
     container_id=%container.id image=%container.image.repository)
  priority: WARNING
  tags: [filesystem, mitre_credential_access]
```
- Falco outputs: syslog, file, Slack webhook, Falcosidekick (fan-out to many outputs)
- Kubernetes audit integration: Falco can also analyze K8s audit log for suspicious API calls
- Base rules: `/etc/falco/falco_rules.yaml` — review and add custom rules in `/etc/falco/falco_rules.local.yaml`
- False positives: tune with macro exceptions for trusted processes (package managers in build stage)

---

### K-053 | seccomp Profile Implementation
**Type:** Implement | **Difficulty:** Expert | ⬜

> Create a custom seccomp profile for an NGINX container that restricts it to only the syscalls it needs.

**What you must cover:**
```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": [
    {
      "names": ["accept4","bind","brk","clone","close","epoll_create1",
                "epoll_ctl","epoll_pwait","execve","exit_group","fcntl",
                "fstat","futex","getpid","getrlimit","getuid","ioctl",
                "listen","mmap","mprotect","munmap","nanosleep","open",
                "openat","poll","prctl","read","recvfrom","recvmsg",
                "rt_sigaction","rt_sigprocmask","rt_sigreturn","sendmsg",
                "sendto","set_robust_list","setsockopt","setuid","socket",
                "socketpair","stat","uname","write","writev"],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```
```yaml
# K8s SeccompProfile (K8s 1.19+)
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/nginx-restricted.json
```
- RuntimeDefault: use as baseline first — allows common syscalls, blocks dangerous ones
- `audit` mode: log violations without blocking — build allowlist from logs
- `strace` to discover syscalls: `strace -c nginx -g nginx.conf` in a test environment

---

### K-054 | Image Pull Secrets and Private Registry
**Type:** Implement | **Difficulty:** Medium | ⬜

> Workloads need to pull images from a private Azure Container Registry. Implement both the credential and the pull secret — and explain the AKS-native approach.

**What you must cover:**
```bash
# Option 1: Explicit imagePullSecrets
kubectl create secret docker-registry acr-pull \
  --docker-server=myregistry.azurecr.io \
  --docker-username=myregistry \
  --docker-password=$(az acr credential show -n myregistry --query passwords[0].value -o tsv)

# Reference in pod spec
spec:
  imagePullSecrets:
  - name: acr-pull
```
- AKS native (preferred): `az aks update --attach-acr myregistry` — grants AKS kubelet managed identity `AcrPull` role — no secret needed
- Image pull credential rotation: managed identity approach is credential-free — preferred
- Per-namespace: configure pull secret + attach to default SA via `imagePullSecrets` on SA object
- Harbor/registry with OIDC: use OIDC-based auth instead of username/password where possible

---

### K-055 | Audit Log Configuration
**Type:** Implement | **Difficulty:** Hard | ⬜

> Configure the Kubernetes API server audit policy to log: all requests to secrets resources, all failed authorization events, and executive-level access.

**What you must cover:**
```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
# Log all access to secrets at RequestResponse level
- level: RequestResponse
  resources:
  - group: ""
    resources: ["secrets"]

# Log all failed requests
- level: Request
  omitStages: [RequestReceived]
  users: []
  verbs: []
  failedRequestsOnly: true

# Log privileged users at high verbosity
- level: RequestResponse
  users: ["admin@company.com", "cluster-admin"]

# Default: log metadata only (no request/response body)
- level: Metadata
```
- Audit log destination: file (disk) or webhook (audit sink to SIEM)
- AKS: audit logs → Azure Monitor Diagnostic Settings → Log Analytics → Sentinel
- Alert on audit: Sentinel analytic rule for `kubectl exec` into production, secret reads by CI accounts

---

## Section 6: Operations and Troubleshooting (K-056 to K-110)

---

### K-056 | CrashLoopBackOff Investigation
**Type:** Troubleshoot | **Difficulty:** Medium | ⬜

> A pod is in CrashLoopBackOff. Walk through the exact investigation steps.

```bash
# Step 1: Understand what's happening
kubectl get pod <pod> -n <ns> -o wide
kubectl describe pod <pod> -n <ns>  # → Exit Code, Last State, Events

# Step 2: Get logs
kubectl logs <pod> -n <ns>              # Current container logs
kubectl logs <pod> -n <ns> --previous   # Previous container's logs (crucial)

# Step 3: Exit code analysis
# Exit 1: application error (check app logs)
# Exit 137 = SIGKILL = OOMKilled (check requests/limits)
# Exit 139 = Segfault
# Exit 143 = SIGTERM (graceful shutdown that failed)

# Step 4: Start debugging container without liveness probe interference
kubectl debug -it <pod> -n <ns> --image=nicolaka/netshoot --copy-to=debug-pod

# Common causes
# - App crash: bug, missing env var, config error
# - OOMKilled: memory limit too low
# - Liveness probe too aggressive (starts before app ready)
# - Missing dependency (DB not ready, secret not mounted)
```

---

### K-057 | OOMKilled Investigation and Prevention
**Type:** Troubleshoot | **Difficulty:** Hard | ⬜

> Multiple pods are being OOMKilled during peak load. Walk through diagnosis and a long-term fix.

```bash
# Diagnosis
kubectl describe pod <pod> -n <ns>
# Look for: Last State: Terminated, Reason: OOMKilled, Exit Code: 137

# Node-level
kubectl describe node <node>  
# Look for: memory usage vs allocatable, recent OOMKill events

# Metrics
# container_oom_events_total (if available)
# container_memory_working_set_bytes vs container_spec_memory_limit_bytes
# VPA recommendations for that container

# Short-term fix
kubectl set resources deployment <name> -n <ns> \
  --requests=memory=512Mi --limits=memory=1Gi

# Long-term fix
# 1. Profile: use VPA Recommender to get actual P95 usage
# 2. Increase limits appropriately
# 3. Java: set -Xmx and -XX:MaxRAMPercentage=75 relative to limit
# 4. Add JVM/runtime memory metrics to dashboards
```

---

### K-058 | ImagePullBackOff Investigation
**Type:** Troubleshoot | **Difficulty:** Medium | ⬜

> Pods are failing with `ImagePullBackOff`. Walk through the complete investigation tree.

```bash
kubectl describe pod <pod> | grep -A 10 Events
# "Failed to pull image": dig into message

# Common causes and fixes:

# 1. Image doesn't exist
docker pull myregistry.azurecr.io/app:v1.2.3  # Test outside K8s

# 2. Wrong tag (typo in helm values or CI)
kubectl get pod <pod> -o jsonpath='{.spec.containers[0].image}'

# 3. No imagePullSecret / wrong credentials
kubectl get secret acr-pull -n <ns> -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d

# 4. Registry not reachable from cluster (network policy, private endpoint)
kubectl run test --image=busybox -it --rm -- nslookup myregistry.azurecr.io
kubectl run test --image=busybox -it --rm -- wget -qO- https://myregistry.azurecr.io/v2/

# 5. Rate limiting (Docker Hub)
# Use a mirror / authenticated pull secret
```

---

### K-059 | Pod Stuck in Pending — Scheduling Failure
**Type:** Troubleshoot | **Difficulty:** Hard | ⬜

> A pod has been Pending for 10 minutes. Walk through the full investigation.

```bash
# Step 1: Why is it pending?
kubectl describe pod <pod> -n <ns>
# Events section will show scheduler failure reason

# Failure reasons and fixes:
# "Insufficient cpu" → node doesn't have enough CPU
kubectl top nodes
kubectl describe node <node> | grep -A 5 "Allocated resources"

# "Taints that pod does not tolerate"
kubectl describe node <node> | grep Taint
# Fix: add toleration to pod spec

# "No nodes are available that match all of the following predicates"
# "node(s) didn't match pod anti-affinity rules"
kubectl get pods -o wide | grep myapp  # How many pods already running, where?

# "PVC not bound" → check PVC
kubectl get pvc -n <ns>
kubectl describe pvc <pvcname> -n <ns>  # Check events on PVC

# "Insufficient nvidia.com/gpu"
kubectl describe nodes | grep -A 5 "Capacity:" | grep gpu

# Node count too low → Cluster Autoscaler
kubectl logs -n kube-system deployment/cluster-autoscaler | tail -50
```

---

### K-060 | Pod Stuck in Terminating
**Type:** Troubleshoot | **Difficulty:** Hard | ⬜

> A pod has been in `Terminating` state for 30 minutes. Walk through investigation and resolution.

```bash
# Check what's blocking termination
kubectl describe pod <pod> -n <ns>
# Look for: finalizers on the pod, preStop hook, terminationGracePeriodSeconds

# Step 1: Check finalizers
kubectl get pod <pod> -n <ns> -o jsonpath='{.metadata.finalizers}'
# If finalizer present and controller dead: manually remove
kubectl patch pod <pod> -n <ns> -p '{"metadata":{"finalizers":[]}}' --type merge

# Step 2: Check if node is NotReady (pod termination stuck because kubelet can't reach apiserver)
kubectl get nodes
kubectl describe node <node> | head -20

# Step 3: Force delete (last resort — can cause duplicate pod if node comes back)
kubectl delete pod <pod> -n <ns> --grace-period=0 --force

# Common causes:
# - Stale finalizer from crashed controller
# - preStop hook hung (service mesh proxy not terminating)
# - terminationGracePeriodSeconds too short for graceful shutdown
# - Node is partitioned / NotReady — pod can't be garbage collected
```

---

### K-061 | Service Not Reachable — Debugging DNS and kube-proxy
**Type:** Troubleshoot | **Difficulty:** Hard | ⬜

> A pod cannot reach a Service by DNS name. Walk through the full debugging tree.

```bash
# Step 1: DNS resolution
kubectl exec -it debug-pod -- nslookup my-service.my-namespace.svc.cluster.local
kubectl exec -it debug-pod -- cat /etc/resolv.conf

# Step 2: If DNS resolves, test TCP connectivity  
kubectl exec -it debug-pod -- wget -qO- http://my-service:80/health
kubectl exec -it debug-pod -- nc -zv my-service 80

# Step 3: Check Service and Endpoints
kubectl get svc my-service -n my-namespace -o yaml
kubectl get endpoints my-service -n my-namespace
# If endpoints are empty: label selector mismatch!
kubectl get pods -n my-namespace -l app=my-service  # Do labels match Service selector?

# Step 4: Check iptables rules (on the node)
sudo iptables -L KUBE-SERVICES -n -v | grep <ClusterIP>
sudo iptables -L KUBE-SVC-<hash> -n -v  # DNAT rules to endpoints

# Step 5: NetworkPolicy blocking?
kubectl get networkpolicy -n my-namespace
# Test without NetworkPolicy: temporarily add allow-all, see if connectivity works

# Step 6: kube-proxy status
kubectl get pods -n kube-system | grep kube-proxy
kubectl logs -n kube-system kube-proxy-<id>
```

---

### K-062 | Node NotReady Troubleshooting
**Type:** Troubleshoot | **Difficulty:** Hard | ⬜

> A node has gone from Ready to NotReady. Walk through investigation from both inside the cluster and on the node itself.

```bash
# From cluster
kubectl describe node <node>
# Conditions section: MemoryPressure, DiskPressure, PIDPressure, Ready
# Events: last seen kubelet events

# On the node itself (SSH in)
systemctl status kubelet
journalctl -u kubelet -f --since "10 minutes ago"

# Common kubelet errors:
# "Error creating container" → runtime issue
# "Failed to pull image" → registry connectivity
# "ContainerCreating takes too long" → CSI volume attach issue
# "dial tcp <apiserver>:443: connect: connection refused" → apiserver connectivity

# Node disk pressure
df -h               # Check disk space
du -sh /var/log/pods/*  # Pod log disk usage
crictl images       # Unused images consuming disk
crictl ps -a        # Stopped containers not cleaned up

# Node memory pressure
free -m
cat /proc/meminfo | grep -E "MemTotal|MemFree|MemAvailable"

# Restart kubelet (if config issue fixed)
systemctl restart kubelet
```

---

### K-063 | HPA Not Scaling
**Type:** Troubleshoot | **Difficulty:** Hard | ⬜

> CPU utilization is above the HPA threshold but the deployment isn't scaling. Diagnose.

```bash
# Check HPA status
kubectl describe hpa <name> -n <ns>
# Look for: Conditions, Current/Desired replicas, Metrics

# Common issues:
# "unable to fetch metrics" 
kubectl get apiservice v1beta1.metrics.k8s.io  # Is metrics-server registered?
kubectl top pods -n <ns>  # Can metrics-server see pod metrics?

# HPA at max replicas already
# Check: spec.maxReplicas vs current replicas

# Insufficient resource requests
# HPA CPU% = current CPU / requested CPU
# If requests.cpu=0 (not set), HPA can't calculate percentage → set requests!

# Scale-down cooldown vs scale-up
# scaleDown.stabilizationWindowSeconds: 300 (default)
# scaleUp.stabilizationWindowSeconds: 0 (fast up) vs pod availability

# KEDA queue check
kubectl describe scaledobject <name> -n <ns>
kubectl get triggerauthentication -n <ns>
```

---

### K-064 | etcd High Latency — Impact and Response
**Type:** Troubleshoot | **Difficulty:** Expert | ⬜

> etcd is showing high commit latency (>100ms). What's the impact and how do you fix it?

```bash
# Measure etcd latency
etcdctl endpoint status --cluster --write-out=table
# Look for: Raft term, DB size, Leader

# etcd latency metrics
# etcd_disk_backend_commit_duration_seconds_bucket — disk flush latency
# etcd_network_peer_round_trip_time_seconds — peer latency

# Common causes:
# 1. Slow disk: etcd MUST be on fast SSD (NVMe preferred)
#    fio --filename=/var/lib/etcd/test --rw=write --bs=4k --ioengine=libaio --iodepth=1
#    Sequential write latency must be <10ms

# 2. DB too large (old revisions not compacted)
etcdctl check perf
etcdctl compact $(etcdctl endpoint status --write-out json | jq '.[0].header.revision')
etcdctl defrag --cluster

# 3. Too many objects (Helm release history bloat)
kubectl get secret -A | grep "helm.sh/release" | wc -l
# Fix: helm history --max 5 or set --history-max during install

# 4. Noisy neighbor (CPU/disk contention on same node)
# etcd should be on dedicated nodes with QoS
```

---

### K-065 | Helm Release Debugging
**Type:** Troubleshoot | **Difficulty:** Medium | ⬜

> A `helm upgrade` succeeded but the application is still running the old version. How do you investigate?

```bash
# Check helm release status
helm status <release> -n <ns>
helm history <release> -n <ns>

# Verify what's actually deployed
kubectl get deployment <name> -n <ns> -o jsonpath='{.spec.template.spec.containers[0].image}'

# Is rollout complete?
kubectl rollout status deployment/<name> -n <ns>

# Check for stuck rollout
kubectl describe deployment <name> -n <ns>
# Look for: Conditions, Available replicas, surge pods

# Did new pods come up?
kubectl get pods -n <ns> -l app=<name> --sort-by=.metadata.creationTimestamp

# Is HPA overriding replica count?
# Helm sets replicas=3 but HPA might be pinned at minReplicas=1
kubectl get hpa -n <ns>

# Did imagePullPolicy: Never or cached layer serve old image?
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].imageID}'
```

---

### K-066 to K-080 | K8s Rapid-Fire Troubleshooting

---

### K-066 | A pod is running but health check endpoint returns 200 yet the Pod is still marked NotReady. Why? | Troubleshoot | Hard | ⬜
**Hint:** `readinessProbe.successThreshold`, `readinessProbe.initialDelaySeconds`, container port mismatch between probe and service.

### K-067 | Two pods in the same namespace can't communicate. NetworkPolicy is the likely culprit. How do you quickly confirm? | Troubleshoot | Medium | ⬜
**Hint:** `kubectl exec -it debug -- curl <pod-ip>:<port>` both ways. `kubectl describe netpol -A`. Temporarily add allow-all netpol.

### K-068 | A `kubectl exec` works but the same command via the K8s API returns "forbidden". Why and how do you fix? | Troubleshoot | Hard | ⬜
**Hint:** RBAC verb `exec` on pod resource. Check `kubectl auth can-i create pods/exec --as=<user>`.

### K-069 | A Deployment's pod template was updated but no rollout happened. Why? | Troubleshoot | Medium | ⬜
**Hint:** Labels were changed but pod selector is immutable. New configmap applied but pods not updated (no `checksum/config` annotation pattern). `updateStrategy: OnDelete`.

### K-070 | cert-manager is not issuing certificates. Walk through investigation. | Troubleshoot | Hard | ⬜
**Hint:** `kubectl describe certificate`, `kubectl describe certificaterequest`, `kubectl describe order`, `kubectl describe challenge`. Check ACME HTTP-01 Ingress object exists and is reachable.

### K-071 | A node is reporting DiskPressure. How do you recover without losing running pods? | Troubleshoot | Hard | ⬜
**Hint:** `crictl rmi --prune` (unused images), `journalctl --vacuum-size=500M`, log rotation config, `kubectl drain --ignore-daemonsets` for cordon + cleanup.

### K-072 | Describe how you'd debug a 5xx error that only occurs on certain nodes (not all endpoints). | Troubleshoot | Expert | ⬜
**Hint:** Endpoint slice shows specific pod IPs receiving traffic. Pod on that node has a local disk volume issue or node-level DNS problem.

### K-073 | A Helm Chart's `helm upgrade` fails with "rendered manifests contain a resource that already exists". How do you fix? | Troubleshoot | Medium | ⬜
**Hint:** Resource was created outside Helm (manually). Add `helm.sh/resource-policy: keep` annotation, or adopt resource with `helm upgrade --force`, or use `kubectl annotate` to add Helm ownership labels.

### K-074 | ArgoCD shows an application as Healthy but OutOfSync. Explain what this means and how to reconcile. | Troubleshoot | Medium | ⬜
**Hint:** OutOfSync means live state differs from Git. Healthy means app is running fine. Auto-sync disabled or manual sync gates. `argocd app sync <name>` or enable `automated.selfHeal`.

### K-075 | A rolling update is stuck at 50% rollout. Pods are not progressing. Why? | Troubleshoot | Hard | ⬜
**Hint:** PodDisruptionBudget blocking eviction. New pods failing readiness probe. HPA at minReplicas=1 can't find room. `kubectl describe deployment` Conditions: ProgressDeadlineExceeded.

---

## Section 7: Advanced Kubernetes Patterns (K-076 to K-110)

---

### K-076 | Multi-Container Pod Patterns
**Type:** Explain | **Difficulty:** Medium | ⬜

> Explain 4 multi-container pod patterns: sidecar, ambassador, adapter, and init. Give a real production scenario for each.

**What you must cover:**
- Sidecar: co-located helper that extends main container (Envoy proxy, Vault agent, Fluent Bit file tailer)
- Ambassador: proxy adapts outside world to main container protocol (Cloud SQL proxy, legacy SOAP→REST)
- Adapter: transforms main container output format for external systems (nginx→metrics adapter for Prometheus)
- Init: setup/precondition before main container (wait-for-DB, schema migration, clone Git repo for content)
- Native sidecar containers (K8s 1.29 stable): sidecar containers have lifecycle tied to main container, start before it, die after it — fixes historical ordering problems

---

### K-077 | Native Sidecar Containers (K8s 1.29+)
**Type:** Explain | **Difficulty:** Hard | ⬜

> Explain the native sidecar container feature (K8s 1.29+) and how it improves over the traditional approach.

**What you must cover:**
- Before 1.29: sidecar = init container pattern or regular container, but init containers complete before main starts
- Native sidecar: `initContainers` with `restartPolicy: Always` — starts before main, lives alongside it, terminates after main
- Problem solved: Istio/Vault sidecar needs to be ready before main container connects to anything. Previously unreliable.
- Job use case: Job pod terminates when main container exits, but sidecar kept pod alive — native sidecar fixes this
- Signal handling: graceful shutdown — sidecar receives SIGTERM after main container exits

---

### K-078 | Kubernetes API Aggregation Layer
**Type:** Explain | **Difficulty:** Expert | ⬜

> Explain the Kubernetes API aggregation layer. How does `metrics.k8s.io` work? How would you add custom API groups?

**What you must cover:**
- APIService: extends Kubernetes API by registering custom API groups with the aggregation layer
- Request flow: kubectl → kube-apiserver → detects path is registered → proxies to extension API server
- example: `v1beta1.metrics.k8s.io` registered, requests proxied to metrics-server pod
- Custom API server: implement `k8s.io/apiserver` library, register via APIService CR
- Health: `kubectl get apiservice v1beta1.metrics.k8s.io` — check Available=True
- CRD vs aggregation: CRD is simpler (no custom server), aggregation allows custom storage, custom subresources, custom validation logic

---

### K-079 | Kubernetes Events Deep Dive
**Type:** Explain | **Difficulty:** Medium | ⬜

> Explain Kubernetes events — what generates them, how to query effectively, and how to build alerting on events.

```bash
# View recent events sorted by time
kubectl get events -n <ns> --sort-by='.lastTimestamp'

# Filter by reason
kubectl get events -n <ns> --field-selector reason=OOMKilling

# Watch for new events
kubectl get events -n <ns> -w

# AlertManager rule on events via kube-state-metrics
# kube_event_count{reason="BackOff",namespace=~"prod.*"} > 50
```
- Event TTL: events are stored in etcd with 1-hour TTL by default — query immediately after incident
- High-volume events: avoid storing high-frequency events (can flood etcd) — use `Events: false` in compact clusters
- event-exporter: export K8s events to metrics or log aggregation for historical analysis
- Alerting: `EventRouter` or `event-exporter` sends events to Slack/PagerDuty based on reason/severity

---

### K-080 | Kubernetes Upgrade Strategy for Production
**Type:** Design | **Difficulty:** Expert | ⬜

> Design a Kubernetes version upgrade strategy for a 20-node AKS production cluster. What are the risks and how do you test?

**What you must cover:**
- Target: upgrade within 2 minor versions of latest stable (K8s support policy)
- Process:
  1. Upgrade dev cluster (same config) — validate workloads, deprecated APIs
  2. Check addon/operator compatibility (cert-manager, ArgoCD, Prometheus operator)
  3. AKS control plane upgrade (zero-downtime, managed)
  4. Node pool upgrade: blue-green node pool (add new pool, cordon old, workloads migrate)
  5. Validate all PDBs respected during node drain
- Deprecated APIs: `kubectl convert` or `pluto` tool to find deprecated APIs in manifests
- AKS node image: upgrade node OS image without upgrading K8s version (monthly)
- PodDisruptionBudget: verify every stateful workload has PDB before upgrade

---

### K-081 to K-100 | Advanced K8s Rapid Scenarios

---

### K-081 | Explain `kubectl drain` vs `kubectl cordon` vs `kubectl taint`. When do you use each? | Explain | Medium | ⬜

### K-082 | A pod needs access to the Docker socket on the host. Why is this dangerous? What's the secure alternative? | Explain | Hard | ⬜
**Hint:** Host docker socket = root-level container escape. Alternatives: Kaniko, BuildKit rootless, ko for Go.

### K-083 | Explain how Kubernetes garbage collects nodes that have been unreachable for >5 minutes. What happens to pods on that node? | Explain | Hard | ⬜
**Hint:** Node lifecycle controller, pod eviction toleration `node.kubernetes.io/not-ready:NoExecute:300s`.

### K-084 | You need to run 10,000 batch jobs that each take ~10 seconds. How do you design this efficiently on Kubernetes? | Design | Expert | ⬜
**Hint:** Job parallelism, IndexedJob mode, Volcano gang scheduling, Kueue for queue management, spot nodes.

### K-085 | A secrets/configmap change should automatically restart pods that consume it. How do you implement this? | Implement | Medium | ⬜
**Hint:** `stakater/Reloader` — watches ConfigMaps/Secrets and adds rolling annotation to Deployments on change.

### K-086 | Explain kubectl rollout restart and how it achieves zero-downtime restarts. | Explain | Medium | ⬜
**Hint:** Adds `kubectl.kubernetes.io/restartedAt` annotation → triggers rolling update → new pods with new SA token/config.

### K-087 | Design a multi-arch container build pipeline (amd64 + arm64) for AKS with ARM node pools. | Design | Hard | ⬜
**Hint:** Docker Buildx with `--platform linux/amd64,linux/arm64` multi-platform build. OCI image index manifest.

### K-088 | A Service uses `externalTrafficPolicy: Local` — why are some backends receiving no traffic? | Troubleshoot | Hard | ⬜
**Hint:** NodePort mode: only sends to pods on the same node as the incoming traffic. If no local pod, drop. Fix: ensure Daemonset or change to `Cluster`.

### K-089 | Explain IRSA (IAM Roles for Service Accounts) on EKS and its equivalent on AKS. | Explain | Hard | ⬜
**Hint:** IRSA = OIDC token from EKS, federated to IAM role. AKS = Workload Identity (same OIDC mechanism → Azure Managed Identity).

### K-090 | A pod is consuming too much CPU and impacting neighbors on the same node. How do you isolate and fix? | Troubleshoot | Hard | ⬜
**Hint:** CPU limits (throttling), `cpuset` pinning via Topology Manager, dedicated node pool with taint+toleration.

### K-091 | Explain the difference between `spec.replicas` and `status.readyReplicas`. A Deployment shows `2/3 ready` — how do you investigate? | Troubleshoot | Medium | ⬜

### K-092 | Implement a pre-stop hook that ensures the service is removed from the load balancer before the container terminates. | Implement | Hard | ⬜
**Hint:** `preStop: exec: command: ["sleep", "15"]` — time for kube-proxy to remove iptables rules before pod accepts SIGTERM.

### K-093 | How do you debug a pod that exits in <1 second (before you can exec into it)? | Troubleshoot | Hard | ⬜
**Hint:** `kubectl run debug --image=<same-image> --command -- sleep 3600` — override entrypoint. `kubectl debug` copy-pod.

### K-094 | Explain how K8s handles node pressure eviction — sequence, thresholds, pod selection. | Explain | Expert | ⬜
**Hint:** Soft eviction (`eviction-soft`) vs hard eviction (`eviction-hard`). Eviction candidates: BestEffort first, then Burstable.

### K-095 | A 3rd party operator is consuming excessive memory on your cluster. How do you audit and constrain it? | Troubleshoot | Hard | ⬜
**Hint:** `kubectl top pods -n operator-ns`, set resource requests/limits on operator deployment, use VerticalPodAutoscaler recommender.

### K-096 | Explain `kubectl apply` vs `kubectl replace` vs `kubectl patch`. When would each fail? | Explain | Medium | ⬜

### K-097 | Implement a GitOps-compatible strategy to promote a config change from dev → staging → prod with approval gates. | Design | Hard | ⬜
**Hint:** ArgoCD ApplicationSet per env + GitHub PR for promotion + environment protection rules (required approvers).

### K-098 | A Kubernetes cluster's API server is throwing 503s. You cannot kubectl. How do you diagnose? | Troubleshoot | Expert | ⬜
**Hint:** SSH to control plane node, `crictl ps`, `systemctl status kube-apiserver`, `tail /var/log/pods/kube-system_kube-apiserver*/kube-apiserver/0.log`, check etcd health.

### K-099 | Explain projected volumes in Kubernetes. What types are supported and when would you use them? | Explain | Hard | ⬜
**Hint:** ServiceAccount token, Secret, ConfigMap, DownwardAPI combined into single mount path. Use case: inject multiple sources into one directory cleanly.

### K-100 | Design a Kubernetes disaster recovery drill plan. How do you test your DR without affecting production? | Design | Expert | ⬜

---

### K-101 to K-110 | AKS-Specific Scenarios

---

### K-101 | AKS Node Pool Upgrade — Blue-Green Strategy
**Type:** Implement | **Difficulty:** Hard | ⬜

> Walk through upgrading an AKS node pool from Ubuntu 20.04 to 22.04 using a blue-green pool strategy.

```bash
# Create new node pool (green)
az aks nodepool add \
  --resource-group myrg \
  --cluster-name myaks \
  --name nodepool2 \
  --node-count 5 \
  --node-vm-size Standard_D4s_v5 \
  --os-type Linux \
  --kubernetes-version 1.30.0

# Cordon old node pool (prevent new scheduling)
kubectl cordon -l agentpool=nodepool1

# Drain old nodes (respect PDB)
kubectl drain -l agentpool=nodepool1 --ignore-daemonsets --delete-emptydir-data

# Verify workloads on new pool
kubectl get pods -o wide | grep nodepool2

# Delete old node pool
az aks nodepool delete --resource-group myrg --cluster-name myaks --name nodepool1
```

---

### K-102 | AKS Private Cluster Connectivity
**Type:** Design | **Difficulty:** Expert | ⬜

> Your AKS cluster is private (API server not reachable from internet). How does kubectl work for developers? How does CI/CD connect for deployments?

**What you must cover:**
- `az aks command invoke`: runs kubectl commands via Azure API without VPN (most secure for CI)
- VPN/Bastion: developers connect to VNet via VPN Gateway or Azure Bastion, kubectl from within VNet
- Private endpoint: AKS API server has private DNS zone, resolves within VNet only
- CI/CD: self-hosted GitHub Actions runners / GitLab runners deployed within the VNet
- ArgoCD: runs inside cluster, no outbound API call needed for push-based GitOps
- `--attach-acr`: ACR private endpoint in same VNet, no public registry access

---

### K-103 | AKS CNI Overlay vs Azure CNI
**Type:** Explain | **Difficulty:** Hard | ⬜

> Compare Azure CNI vs Azure CNI Overlay for an AKS cluster with 500 pods per node.

**What you must cover:**
- Azure CNI: pods get VNet IPs → IP exhaustion with large clusters (each pod consumes a VNet IP)
- Azure CNI Overlay: pods get IPs from separate Pod CIDR (not VNet) → no IP exhaustion, scales easily
- NAT: Overlay adds NAT at node boundary for pod-to-VNet traffic (small perf overhead)
- Network Policy: Cilium works with Overlay, Azure NPM works with both
- Direct integration: Azure services requiring VNet-native pod IPs (some Azure LB configs) need non-overlay
- AKS recommendation (2024+): CNI Overlay for most clusters; plain Azure CNI only when VNet-native pod IPs required

---

### K-104 | AKS Workload Identity Troubleshooting
**Type:** Troubleshoot | **Difficulty:** Hard | ⬜

> A pod using AKS Workload Identity is getting 401 Unauthorized from Azure Key Vault. Walk through diagnosis.

```bash
# Step 1: Verify pod has workload identity annotations
kubectl get pod <pod> -o jsonpath='{.metadata.labels}'
# Should have: azure.workload.identity/use: "true"

# Step 2: Verify service account annotation
kubectl get sa <sa-name> -n <ns> -o jsonpath='{.metadata.annotations}'
# Should have: azure.workload.identity/client-id: <client-id>

# Step 3: Verify AKS OIDC issuer configured
az aks show -n myaks -g myrg --query oidcIssuerProfile

# Step 4: Verify federated credential
az identity federated-credential list --identity-name myid --resource-group myrg

# Step 5: Check subject match
# subject must be: system:serviceaccount:<namespace>:<sa-name>

# Step 6: Check Key Vault RBAC
az role assignment list --scope /subscriptions/<>/...<keyvault-id>
# Should show: Key Vault Secrets User on managed identity

# Step 7: Verify from inside pod
kubectl exec -it <pod> -- env | grep AZURE_FEDERATED_TOKEN_FILE
kubectl exec -it <pod> -- cat $AZURE_FEDERATED_TOKEN_FILE  # JWT token should exist
```

---

### K-105 | AKS KEDA Scaling with Azure Service Bus
**Type:** Implement | **Difficulty:** Hard | ⬜

> Implement KEDA ScaledObject to scale a consumer deployment from 0 to 50 replicas based on Azure Service Bus queue length.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: servicebus-auth
  namespace: consumers
type: Opaque
stringData:
  connection: "Endpoint=sb://...SharedAccessKey=..."
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: servicebus-triggerauth
  namespace: consumers
spec:
  secretTargetRef:
  - parameter: connection
    name: servicebus-auth
    key: connection
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-processor-scaler
  namespace: consumers
spec:
  scaleTargetRef:
    name: order-processor
  pollingInterval: 15
  cooldownPeriod: 60
  minReplicaCount: 0
  maxReplicaCount: 50
  triggers:
  - type: azure-servicebus
    metadata:
      queueName: orders
      queueLength: "100"           # Scale 1 replica per 100 messages
      activationQueueLength: "5"   # Activate from 0 replicas at 5 messages
    authenticationRef:
      name: servicebus-triggerauth
```

---

### K-106 | AKS Cost Optimization
**Type:** Design | **Difficulty:** Hard | ⬜

> Your AKS cluster costs $50,000/month. Design a cost reduction strategy targeting 30% savings without impacting reliability.

**What you must cover:**
- Right-sizing: VPA recommender + Goldilocks dashboard — identify over-provisioned pods (typically 40% of pods)
- Spot node pools: add spot pool for dev, batch, and fault-tolerant workloads (60–80% cheaper)
- Scale to zero: KEDA for idle consumers, `minReplicaCount: 0` for non-critical workloads at night
- Reserved instances: save plan or RI for baseline node count (1-year = ~40% discount)
- Cluster autoscaler: ensure scale-down is working (check `--scale-down-delay-after-add`, `--skip-nodes-with-local-storage`)
- Node consolidation: Karpenter consolidation (pack multiple pods into fewer nodes)
- Idle hours: dev clusters scaled to 0 outside business hours via scheduled scale down

---

### K-107 | AKS Monitoring with Azure Monitor and Prometheus
**Type:** Implement | **Difficulty:** Hard | ⬜

> Configure AKS managed Prometheus + Grafana for cluster monitoring. What metrics are collected by default and what additional configuration is needed?

**What you must cover:**
- Enable: `az aks update --enable-azure-monitor-metrics` — installs managed Prometheus (Azure Monitor workspace)
- Default scrape: kubelet, cadvisor, kube-state-metrics, node-exporter, API server
- Custom PodMonitor/ServiceMonitor: create CRs — managed Prometheus picks them up
- Grafana workspace: link to Azure Monitor workspace, team dashboards
- Prometheus remote write: existing Prometheus → remote write to Azure Monitor workspace
- Cost: Azure Monitor pricing by ingested metrics volume — be careful with high-cardinality metrics
- Alert rules: PrometheusRuleGroup CR (managed) vs raw AlertManager config

---

### K-108 | AKS Security Hardening Checklist
**Type:** Design | **Difficulty:** Expert | ⬜

> Walk through the AKS security hardening checklist for a production cluster serving financial data.

**What you must cover:**
- Private cluster: API server not internet-facing
- AAD integration: RBAC backed by Azure AD groups
- Workload Identity: no static credentials in pods
- ACR private endpoint: no public registry access
- Network policies: Cilium or Azure NPM, default-deny per namespace
- Pod Security Standards: `restricted` on app namespaces
- Secrets: Azure Key Vault + CSI secrets driver, encrypted at rest
- Node hardening: no SSH access (Azure Bastion only), auto-patching enabled
- Audit logs: Diagnostic Settings → Log Analytics → Sentinel
- Defender for Containers: runtime threat detection, vulnerability assessment
- RBAC: no wildcard roles, team-scoped RoleBindings, audit ClusterRoleBindings regularly

---

### K-109 | AKS Cluster Autoscaler Tuning
**Type:** Design | **Difficulty:** Hard | ⬜

> Your AKS cluster is slow to scale up (3-5 minutes for new nodes). How do you optimize for faster scale-up?

**What you must cover:**
- Node image pre-pull: pre-pull application images in node startup script — avoids 1-2min pull during first pod start
- Overprovisioning: deploy low-priority pause pods that consume spare capacity — real pods evict them, capacity already exists
- Fast node type: choose VMs with fast disk I/O and smaller boot time (DS series vs D series)
- Cluster autoscaler settings:
```yaml
--scale-down-delay-after-add=10m  # Don't scale down new nodes immediately
--max-node-provision-time=15m     # Timeout if node doesn't join in 15 min
```
- Karpenter (alternative): just-in-time provisioning with 30-second node launch vs 2-minute for CAS
- Proactive scaling: HPA triggers scale-up via KEDA before queue backlog causes impact

---

### K-110 | Multi-Cluster ArgoCD Management
**Type:** Design | **Difficulty:** Expert | ⬜

> You're using ArgoCD on a management cluster to deploy to 50 target clusters. Walk through the registration, ApplicationSet design, and security model.

**What you must cover:**
```bash
# Register target cluster
argocd cluster add <context-name> --name prod-eastus-01

# ApplicationSet with cluster generator
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
spec:
  generators:
  - clusters:
      selector:
        matchLabels:
          environment: production
  template:
    spec:
      project: production
      source:
        repoURL: https://github.com/company/gitops
        targetRevision: main
        path: clusters/{{name}}  # Cluster-specific kustomize overlay
      destination:
        server: '{{server}}'
        namespace: argocd
```
- Security: ArgoCD uses service account token per target cluster, minimal RBAC (only what ArgoCD deploys)
- Cluster secret: `argocd-cluster-<name>` secret stores kubeconfig → use ESO or Vault to manage these
- Multi-tenancy: use ArgoCD Projects to limit which repos/paths/namespaces each team's apps can target
- Drift detection: `app.kubernetes.io/managed-by: argocd` label on all managed resources — drift shown in UI
