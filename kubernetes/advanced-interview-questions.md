1. How Does Kubernetes Work Internally? What Happens When You Apply a Manifest?
The Question: “Walk me through what happens when I run kubectl apply -f deployment.yaml"

Why It Matters: This question reveals if you understand Kubernetes as a declarative system with reconciliation loops, not just a container orchestrator.
```
Act 1: Client Request
deployment.yaml Creation:
A user defines the desired state in a YAML manifest. This includes the deployment name, replica count, and the pod specification (e.g., container image, ports).
kubectl Execution:
The kubectl apply -f deployment.yaml command is executed.
kubectl parses the YAML, converts it to a JSON payload, and makes a REST API call to the kube-apiserver.


Act 2: Control Plane - Initial Processing
kube-apiserver Receives Request:
Authentication: Verifies the identity of the client (e.g., via a user's kubeconfig or a service account token).
Authorization: Checks if the authenticated client is permitted to perform the requested action (e.g., create a Deployment) on the specified resource, typically using Role-Based Access Control (RBAC).
Admission Control: Executes a sequence of validation and mutation checks. This ensures the object conforms to cluster policies before it is persisted.
Persistence to etcd:
After the request is fully validated, the kube-apiserver writes the Deployment object to the etcd key-value store.
This write action solidifies the desired state as the cluster's new source of truth. etcd then notifies subscribers (like the API server itself) of the change.


Act 3: Control Plane - Reconciliation Loops
Deployment Controller (in kube-controller-manager):
Watch: This controller constantly watches the API Server for changes to Deployment objects.
Goal: To align the actual state with the desired state defined in the Deployment object, primarily by managing ReplicaSet versions.
Action:
It evaluates the Deployment and determines a ReplicaSet needs to be created or updated.
It generates a new ReplicaSet object with a name derived from the Deployment and a Pod template copied from the Deployment's specification.
It sends this ReplicaSet object to the kube-apiserver to be stored in etcd.

ReplicaSet Controller (in kube-controller-manager):
Watch: This controller watches for changes to ReplicaSet objects.
Goal: To ensure that the number of Pods matching its selector always equals the .spec.replicas count.
Action:
It compares the desired replica count from its spec (e.g., 3) against the number of currently active Pods it manages (currently 0).
Based on the difference, it creates the required number of Pod objects from its Pod template.
It sends these Pod objects to the kube-apiserver. The Pods are stored in etcd in the Pending phase because the .spec.nodeName field is not yet set.


Act 4: Control Plane - Scheduling
kube-scheduler:
Watch: The scheduler's sole focus is watching for Pod objects that have an empty .spec.nodeName field.
Goal: To assign each un-scheduled Pod to the most suitable worker Node.
Action (for each Pod):
Filtering: It identifies a list of viable Nodes that can run the Pod. This involves checking for constraints like resource requirements (CPU, memory), node selectors, taints/tolerations, and volume compatibility.
Scoring: It ranks the viable Nodes by applying a set of priority functions. These functions score nodes based on factors like resource utilization or trying to spread Pods across failure domains.
Binding: It commits to the highest-scoring Node and notifies the kube-apiserver. This action updates the Pod object in etcd, setting the .spec.nodeName to the chosen Node.


Act 5: Node-Level Execution
Kubelet (on the assigned Node):
Watch: Each Kubelet agent watches the API Server for Pods that have been scheduled to its node.
Goal: To ensure the containers described in a Pod's specification are running and healthy on its Node.
Action:
It detects the new Pod assigned to it.
It interacts with the Container Runtime (via the Container Runtime Interface - CRI) to perform the following:
Image Pull: Instructs the runtime to pull the container image (nginx:latest) if it's not cached locally.
Container Creation: Instructs the runtime to create and run the container based on the image and Pod spec.
Resource Setup: Configures the Pod's sandbox environment, including setting up networking (via CNI plugin) and mounting any specified data volumes.

Status Reporting:
The Kubelet continuously monitors the status of the Pod's containers.
It reports the Pod's status, events, and IP address back to the kube-apiserver, which updates the Pod's .status field in etcd. This makes the state visible to the rest of the cluster.
The Key Insight: This entire process, from controllers to the scheduler to the kubelet, is a series of reconciliation loops. Each component continuously watches the "desired state" in etcd and works independently to make the "actual state" of the world match it.
```
The Key Insight: This entire process, from controllers to the scheduler to the kubelet, is a series of reconciliation loops. Each component continuously watches the “desired state” in etcd and works independently to make the “actual state” of the world match it.

2. How Does DNS Resolution Work in Kubernetes?

The Question: “Explain how a Pod resolves my-service.my-namespace.svc.cluster.local"

Why It Matters: DNS issues are common in multi-namespace applications and service mesh configurations.
```
When a Pod tries to resolve a service name:

- The Pod's /etc/resolv.conf is configured by kubelet to point to CoreDNS (typically at 10.96.0.10 or similar cluster IP).
- CoreDNS receives the DNS query and checks if it matches the cluster domain pattern (.svc.cluster.local).
- For cluster services, CoreDNS queries the Kubernetes API to find the Service object and returns its ClusterIP.
- For external domains, CoreDNS forwards the query to upstream DNS servers (defined in the CoreDNS ConfigMap).
- The Pod receives the IP and establishes a connection. If it's a Service ClusterIP, kube-proxy (or iptables/IPVS rules) handles the load balancing to backend Pods.

Common Pitfalls:

Using wrong namespace in service name
CoreDNS ConfigMap misconfiguration for external DNS
Network policies blocking DNS traffic on port 53
```
3. Taints/Tolerations vs Node Affinity: When to Use What?
The Question: “What’s the difference between taints/tolerations and node affinity? When would you use each?”

Why It Matters: Proper workload placement is critical for resource optimization and isolation.
```
Taints/Tolerations:

Purpose: Repel Pods from nodes unless they explicitly tolerate the taint
Direction: Node-centric (nodes push away Pods)
Use Cases:
- Dedicated nodes for specific workloads (e.g., GPU nodes)
- Keeping nodes reserved during maintenance
- Isolating problematic or experimental workloads



Node Affinity:

Purpose: Attract Pods to specific nodes based on labels
Direction: Pod-centric (Pods pull toward nodes)
Use Cases:
- Placing Pods on nodes with specific hardware (SSD, high-memory)
- Multi-tenant clusters with customer-specific node pools
- Co-locating related Pods for performance
```
