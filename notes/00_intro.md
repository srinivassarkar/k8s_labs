# Kubernetes Architecture — Core, Extensions & the Plugin Model


---

## 0. First Principles — The Mental Model That Never Changes

Everything in Kubernetes architecture collapses into one line:

```
Kubernetes = Core (orchestration) + Extensions (runtime, network, storage)
```

Core decides **what** should happen. Extensions decide **how** it actually happens.

```
Core         →  "Run 3 replicas of this pod on healthy nodes"
CRI          →  actually starts the container process
CNI          →  actually gives it an IP and connects it to the network
CSI          →  actually attaches and mounts the persistent volume
```

The Kubernetes project owns the core. The ecosystem owns the extensions. This split is intentional and architectural — it is the reason Kubernetes runs on every cloud, with dozens of different runtimes, networks, and storage backends, without changing a line of core code.

**The three laws that never change:**

1. The API Server is the only entry point — nothing talks to etcd directly
2. Kubelet is the enforcer on every node — it owns the node's actual state
3. Extensions are swappable — Kubernetes core never hardcodes a runtime, network, or storage implementation

---

## 1. Reality Constraints — What Kubernetes Actually Does and Doesn't Do

### What Core Kubernetes Does

- Schedules pods to nodes
- Maintains desired state via controllers
- Serves and stores all cluster state through the API Server → etcd
- Manages service discovery and load balancing rules via kube-proxy
- Exposes extension points (CRI, CNI, CSI) for everything else

### What Core Kubernetes Does NOT Do

| Misconception | Reality |
|---|---|
| "Kubernetes runs containers" | No — kubelet calls CRI; the runtime runs containers |
| "Kubernetes handles pod networking" | No — CNI plugin assigns IPs and routes traffic |
| "Kubernetes manages volumes" | No — CSI driver creates, attaches, and mounts storage |
| "Kubernetes needs Docker" | No — Docker was removed in v1.24; containerd is the default |
| "Docker images don't work anymore" | Wrong — Docker images are OCI-compliant and work with any CRI |

### The Docker Clarification (Interview Trap)

```
What was removed:   dockershim (the adapter that let kubelet talk to Docker)
Removed in:         Kubernetes v1.24
What replaced it:   containerd (direct CRI integration)

What still works:   Docker-built images (they follow OCI standard)
Why they work:      containerd and CRI-O can pull and run any OCI image

Mental model:
  Docker (tool)  ≠  Docker image format (OCI standard)
  The tool is gone from the runtime path.
  The image format lives on everywhere.
```

### The Extension Contract

Each plugin type has a defined interface. Kubernetes core talks to the interface — never to a specific implementation:

```
kubelet → CRI interface → [containerd | CRI-O | gVisor | Kata]
kubelet → CNI interface → [Calico | Cilium | Flannel | Weave]
kubelet → CSI interface → [EBS | Azure Disk | Ceph | Longhorn]
```

Swap the plugin, keep the core. This is vendor neutrality by design.

---

## 2. Decision Logic — When to Use What

### Choosing a CRI (Container Runtime)

| Runtime | Use When | Avoid When |
|---|---|---|
| `containerd` | Default; general purpose; most clusters | Never — it's the right default |
| `CRI-O` | OpenShift / minimal footprint preferred | You need Docker CLI compatibility |
| `gVisor` (runsc) | Multi-tenant; untrusted workloads; sandbox isolation needed | High-syscall workloads (DBs, high perf) |
| `Kata Containers` | VM-level isolation required (compliance, shared infra) | Low latency or high density workloads |

### Choosing a CNI Plugin

| Plugin | Use When | Key Feature |
|---|---|---|
| `Calico` | NetworkPolicy enforcement is required; most production clusters | L3 routing + policy |
| `Cilium` | High performance; observability; eBPF-native required | eBPF dataplane, L7 policy |
| `Flannel` | Simple overlay; dev clusters; no NetworkPolicy needed | VXLAN overlay, minimal config |
| `Weave` | Simple multi-host networking with encryption | Mesh overlay |
| `AWS VPC CNI` | EKS — pods need native VPC IPs | AWS-native routing |

> **CKA exam note:** Flannel does not support `NetworkPolicy`. If a question involves NetworkPolicy, the cluster needs Calico or Cilium.

### Choosing a CSI Driver

| Driver | Use When |
|---|---|
| `AWS EBS CSI` | EKS / AWS-based clusters needing block storage |
| `Azure Disk CSI` | AKS / Azure-based clusters |
| `GCE PD CSI` | GKE / GCP-based clusters |
| `Ceph / Rook` | On-premise; need block, object, and file storage |
| `Longhorn` | On-premise; simpler setup; distributed block storage |
| `NFS CSI` | Shared `ReadWriteMany` volumes across pods |

### The Plugin Decision Triangle

```
Need to run workloads?      → CRI decision
Need pod-to-pod networking? → CNI decision
Need persistent data?       → CSI decision
```

---

## 3. Internal Working — How It Actually Happens

### Component Map: Control Plane

```
┌─────────────────────────────────────────────────────────────────┐
│                        CONTROL PLANE                            │
│                                                                 │
│  kube-apiserver    ← single entry point; validates + stores     │
│         ↕                                                       │
│       etcd         ← only persistent store; all state lives here│
│                                                                 │
│  kube-scheduler    ← watches unscheduled pods; picks a node     │
│  kube-controller-manager ← runs all controllers (Deployment,   │
│                            ReplicaSet, Node, Job, etc.)         │
│  cloud-controller-manager ← integrates with cloud APIs         │
│                             (LB, node lifecycle, routes)        │
└─────────────────────────────────────────────────────────────────┘
```

### Component Map: Node

```
┌──────────────────────────────────────────────────────┐
│                        NODE                          │
│                                                      │
│  kubelet      ← talks to API Server; enforces pods   │
│     ↓                                                │
│  CRI plugin   ← runs containers (containerd/CRI-O)   │
│  CNI plugin   ← sets up pod networking               │
│  CSI plugin   ← attaches/mounts volumes              │
│                                                      │
│  kube-proxy   ← programs iptables/ipvs for Services  │
└──────────────────────────────────────────────────────┘
```

### The Pod Creation Flow (Step by Step)

This is the most important sequence to understand end-to-end:

```
1. User runs: kubectl apply -f pod.yaml
2. kubectl sends HTTP POST to kube-apiserver
3. API Server:
   a. Authenticates the request
   b. Authorizes via RBAC
   c. Runs admission controllers (mutating → validating)
   d. Validates the object schema
   e. Writes Pod object to etcd with status: Pending
4. kube-scheduler:
   a. Watches for Pending pods with no nodeName
   b. Runs filtering (node has enough CPU/memory, taints/tolerations, etc.)
   c. Runs scoring (picks best node)
   d. Writes nodeName to the Pod spec in etcd
5. kubelet on the chosen node:
   a. Watches API Server for pods assigned to its node
   b. Sees the new pod
   c. Calls CRI → "create and start this container"
6. CRI (containerd):
   a. Pulls image if not cached
   b. Creates container sandbox (pause container)
   c. Starts the container process
7. CNI plugin is called by kubelet:
   a. Assigns an IP address to the pod
   b. Sets up veth pair (virtual ethernet)
   c. Programs routing so other pods can reach this IP
8. If volumes are requested, kubelet calls CSI:
   a. CSI controller: creates/attaches the volume to the node
   b. CSI node plugin: mounts the volume into the pod's filesystem
9. kubelet updates pod status → Running
10. kube-proxy (on all nodes):
    a. Watches Service objects
    b. Programs iptables/ipvs rules for service-to-pod routing
```

### The CRI Interface — What kubelet Actually Calls

```
kubelet → gRPC → CRI runtime (containerd)

CRI operations kubelet uses:
  RunPodSandbox()     → create the network namespace + pause container
  CreateContainer()   → create the app container inside the sandbox
  StartContainer()    → start the process
  StopContainer()     → graceful stop
  RemoveContainer()   → cleanup
  PullImage()         → pull image if not present
  ListContainers()    → report status back to API Server
```

### The CNI Interface — What kubelet Actually Calls

```
kubelet → exec → CNI binary (e.g. /opt/cni/bin/calico)

CNI operations:
  ADD   → called when pod is created: assign IP, set up routing
  DEL   → called when pod is deleted: release IP, remove routes
  CHECK → verify pod network is still correctly configured

CNI config lives at: /etc/cni/net.d/
CNI binaries live at: /opt/cni/bin/
```

### The CSI Interface — What kubelet Actually Calls

```
CSI has two components:

1. CSI Controller Plugin (runs as a Deployment on control plane or any node)
   → CreateVolume()    → provisions the disk in the cloud/storage system
   → AttachVolume()    → attaches the disk to the node at the hypervisor level
   → DeleteVolume()    → deprovisions the disk

2. CSI Node Plugin (runs as DaemonSet on every node)
   → NodeStageVolume()  → formats and mounts to a staging path on the node
   → NodePublishVolume() → bind-mounts into the pod's directory
   → NodeUnpublishVolume() → unmounts from pod
```

### Controller Manager: What Controllers Actually Do

```
ReplicaSet controller  → if actual pods < desired, creates new pods
                         if actual pods > desired, deletes excess pods
Deployment controller  → manages ReplicaSets for rollout/rollback
Node controller        → marks nodes NotReady if heartbeat stops;
                         evicts pods after timeout
Job controller         → creates pods for Jobs; retries on failure
ServiceAccount ctrl    → creates default SA in new namespaces
EndpointSlice ctrl     → keeps EndpointSlices in sync with pod IPs
```

### kube-proxy: How Services Actually Work

```
Service created → kube-proxy watches API Server
               → programs iptables (or ipvs) rules on every node

ClusterIP:
  iptables DNAT rule: ServiceIP:Port → one of the pod IPs (random selection)

NodePort:
  iptables rule on every node: NodeIP:NodePort → ServiceIP → PodIP

LoadBalancer:
  cloud-controller-manager creates cloud LB → forwards to NodePort → kube-proxy rules

Cilium replaces kube-proxy entirely with eBPF maps — faster, no iptables
```

---

## 4. Hands-On — YAML + Commands

### Inspect the Runtime (CRI)

```bash
# Check which CRI is in use on a node
kubectl get node <node-name> -o jsonpath='{.status.nodeInfo.containerRuntimeVersion}'
# Output example: containerd://1.7.2

# On the node — check containerd status
systemctl status containerd

# List running containers via containerd CLI
crictl ps

# List images
crictl images

# Inspect a specific container
crictl inspect <container-id>

# Pull an image manually via CRI
crictl pull nginx:latest

# Check kubelet's CRI socket config
cat /var/lib/kubelet/config.yaml | grep containerRuntime
# or
ps -aux | grep kubelet | grep container-runtime
```

### Inspect the Network (CNI)

```bash
# Check CNI config
ls /etc/cni/net.d/
cat /etc/cni/net.d/10-calico.conflist   # example for Calico

# Check CNI binaries
ls /opt/cni/bin/

# Check pod IPs
kubectl get pods -o wide

# Check Calico node status (if using Calico)
kubectl get pods -n calico-system
kubectl exec -n calico-system <calico-node-pod> -- calicoctl node status

# Check Cilium status (if using Cilium)
kubectl -n kube-system exec ds/cilium -- cilium status

# Test pod-to-pod connectivity
kubectl exec -it <pod-a> -- ping <pod-b-ip>

# Check NetworkPolicy is enforced
kubectl get networkpolicy -A
```

### Inspect the Storage (CSI)

```bash
# List CSI drivers installed
kubectl get csidrivers

# Check CSI node registrations
kubectl get csinodes

# List StorageClasses
kubectl get storageclass

# Check PV and PVC status
kubectl get pv
kubectl get pvc -A

# Describe a PVC to see which CSI driver handled it
kubectl describe pvc <pvc-name> -n <namespace>
# Look for: "ProvisionerName" and "Volume" fields

# Check CSI controller and node pods
kubectl get pods -n kube-system | grep csi

# Check volume attachment to a node
kubectl get volumeattachment
```

### StorageClass with CSI Driver

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ebs
provisioner: ebs.csi.aws.com        # ← CSI driver name
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer  # delays provisioning until pod is scheduled
allowVolumeExpansion: true
```

### PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
  namespace: production
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: fast-ebs
  resources:
    requests:
      storage: 20Gi
```

### Pod Using All Three Extensions

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: full-stack-pod
spec:
  runtimeClassName: gvisor        # ← CRI: use gVisor runtime instead of default
  containers:
  - name: app
    image: myregistry/app:v2
    resources:
      requests:
        cpu: 250m
        memory: 256Mi
      limits:
        cpu: 500m
        memory: 512Mi
    volumeMounts:
    - name: data
      mountPath: /data            # ← CSI: volume mounted here
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: app-data         # ← CSI: backed by EBS via StorageClass
  # CNI handles the pod IP and network — no YAML needed, it's automatic
```

### RuntimeClass (CRI Selection)

```yaml
# Define a RuntimeClass for gVisor
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc           # matches the handler name in containerd config

---
# Define a RuntimeClass for Kata
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata
handler: kata-qemu
```

### NetworkPolicy (CNI Enforcement)

```yaml
# Deny all ingress to a namespace, allow only from same namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}           # applies to all pods in namespace
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}       # allow from pods in same namespace only
```

```yaml
# Allow only specific pod to talk to a database pod
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-app-to-db
  namespace: production
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: app
    ports:
    - protocol: TCP
      port: 5432
```

### Check Node Component Status

```bash
# Control plane component health
kubectl get componentstatuses   # deprecated but still works on many clusters
kubectl get pods -n kube-system

# Kubelet status on a node
systemctl status kubelet
journalctl -u kubelet -f --no-pager

# kube-proxy status
kubectl get pods -n kube-system | grep kube-proxy
kubectl logs -n kube-system <kube-proxy-pod>

# Scheduler and controller manager
kubectl get pods -n kube-system | grep -E "scheduler|controller"
kubectl logs -n kube-system kube-scheduler-<node>
```

---

## 5. Production Flow — Real-World Architecture and Design Patterns

### Standard Production Stack

```
Control Plane (HA — 3 nodes):
  kube-apiserver (load balanced)
  etcd (3-node quorum)
  kube-scheduler
  kube-controller-manager

Node Components (all worker nodes):
  kubelet
  kube-proxy (or replaced by Cilium)
  containerd (CRI)
  Calico or Cilium (CNI)
  CSI node plugin (DaemonSet)

Cluster-wide:
  CSI controller plugin (Deployment)
  CoreDNS (cluster DNS)
  Ingress Controller (NGINX, AWS ALB, etc.)
  Metrics Server (for HPA)
```

### CNI Plugin Architecture (How Calico Works Internally)

```
Pod created → kubelet calls CNI ADD
Calico CNI binary:
  1. Creates veth pair (one end in pod netns, one on host)
  2. Assigns IP from IPAM pool
  3. Adds routes on host: "pod IP → veth"
  4. Programs BGP routes (Calico-specific) so other nodes know the pod IP

Cross-node traffic:
  Pod A (Node 1, 10.0.1.5) → Pod B (Node 2, 10.0.2.7)
  Calico uses BGP to advertise 10.0.2.0/24 is on Node 2
  Traffic goes: Pod A → Node 1 kernel → BGP route → Node 2 → Pod B
  (No overlay/encapsulation in default Calico — pure L3 routing)
```

### CSI Volume Lifecycle in Production

```
Developer creates PVC
        ↓
external-provisioner (CSI sidecar) calls CreateVolume() on CSI controller
        ↓
CSI controller plugin calls cloud API → EBS volume created in AWS
        ↓
PV object created, bound to PVC
        ↓
Pod scheduled to node
        ↓
attacher sidecar calls ControllerPublishVolume() → EBS attached to EC2 instance
        ↓
kubelet calls NodeStageVolume() via CSI node plugin → formats + mounts to staging path
        ↓
kubelet calls NodePublishVolume() → bind-mounts into pod's /data directory
        ↓
Pod starts, container sees /data as a local filesystem
```

### The Plugin Sidecar Pattern (CSI)

CSI drivers don't work alone — they use standardized sidecar containers:

```
CSI Controller Deployment:
  [CSI driver container]
  [external-provisioner]  ← watches PVCs, calls CreateVolume
  [external-attacher]     ← watches VolumeAttachment, calls ControllerPublish
  [external-snapshotter]  ← watches VolumeSnapshots
  [external-resizer]      ← watches PVC resize requests

CSI Node DaemonSet:
  [CSI driver container]
  [node-driver-registrar] ← registers driver with kubelet
  [liveness-probe]        ← health check for kubelet
```

### Why the Plugin Architecture Exists (The Four Reasons)

```
1. Interoperability
   Kubernetes runs identically whether your runtime is containerd, gVisor,
   or Kata — the core doesn't know or care.

2. Vendor Neutrality
   AWS, GCP, Azure, and on-prem Ceph all implement the same CSI interface.
   No cloud vendor controls the Kubernetes core.

3. Independent Innovation
   Cilium can add eBPF capabilities, image signing, L7 policies without
   waiting for a Kubernetes release. Plugins evolve at their own pace.

4. Maintainability
   A bug in the EBS CSI driver doesn't require a Kubernetes patch.
   A Calico upgrade doesn't require changing kubelet.
   Decoupling keeps the blast radius small.
```

### Multi-Runtime Clusters (Advanced Pattern)

```yaml
# Some pods run on containerd (default), others on gVisor
# RuntimeClass routes to the right handler

# Untrusted tenant workloads → gVisor (syscall sandbox)
spec:
  runtimeClassName: gvisor

# Compliance-sensitive workloads → Kata (VM isolation)
spec:
  runtimeClassName: kata

# Standard workloads → default (containerd)
spec: {}  # no runtimeClassName needed
```

---

## 6. Mistakes — What Actually Breaks in Real Systems

### Mistake 1 — CNI Plugin Not Installed, Pods Stuck in Pending/ContainerCreating

**What happens:** Cluster bootstrapped with kubeadm, but CNI not applied yet. All pods stuck in `ContainerCreating`. Nodes show `NotReady`.

**Root cause:** Kubernetes schedules the pod, kubelet tries to set up networking, but no CNI binary exists. The call fails silently from the pod's perspective.

**Fix:**
```bash
# Check the actual error
kubectl describe pod <pod-name>
# Look for: "network plugin is not ready" or "cni plugin not initialized"

# Apply CNI plugin (example: Calico)
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

# Verify CNI is running
kubectl get pods -n calico-system
# Wait until all calico-node pods are Running, then CoreDNS will start
```

### Mistake 2 — Wrong CRI Socket Path for kubelet

**What happens:** After installing containerd, kubelet fails to start. Error: "failed to create CRI shim" or "connection refused."

**Root cause:** kubelet defaults to a CRI socket path that doesn't match where containerd placed its socket.

**Fix:**
```bash
# Check what socket containerd is listening on
ls /run/containerd/containerd.sock

# Configure kubelet to use correct socket
# In /var/lib/kubelet/config.yaml:
containerRuntimeEndpoint: unix:///run/containerd/containerd.sock

# Or via kubelet args:
--container-runtime-endpoint=unix:///run/containerd/containerd.sock

systemctl restart kubelet
```

### Mistake 3 — NetworkPolicy Blocks Nothing Because CNI Doesn't Support It

**What happens:** Team applies `NetworkPolicy` resources. Security audit shows traffic is still flowing between namespaces that should be isolated. Policies appear to be working (no errors), but they have zero effect.

**Root cause:** Cluster is using Flannel, which does not implement NetworkPolicy. The objects are stored in etcd, but no component enforces them.

**Fix:**
```bash
# Check which CNI is installed
ls /etc/cni/net.d/
cat /etc/cni/net.d/*.conf | grep type

# Flannel conf shows "type": "flannel" — no policy enforcement
# Solution: migrate to Calico or Cilium

# Migration is non-trivial on live clusters — plan with maintenance window
```

### Mistake 4 — CSI Volume Stuck in Terminating State

**What happens:** PVC deleted, but it stays in `Terminating` indefinitely. The volume is gone from the cloud console but the PVC object won't disappear.

**Root cause:** The PVC has a finalizer (`kubernetes.io/pvc-protection`) that prevents deletion until no pod is using it. Or the CSI driver pod is down and can't complete the deletion lifecycle.

**Fix:**
```bash
# Check finalizers
kubectl get pvc <name> -o yaml | grep finalizers

# Check if any pod is still using it
kubectl get pods -A -o json | jq '.items[] | select(.spec.volumes[]?.persistentVolumeClaim.claimName == "<pvc-name>")'

# If no pod is using it and PVC is stuck, manually remove finalizer
kubectl patch pvc <name> -p '{"metadata":{"finalizers":null}}'

# Check CSI driver pods are healthy
kubectl get pods -n kube-system | grep csi
```

### Mistake 5 — Pod IP Not Reachable Across Nodes

**What happens:** Pod A on Node 1 can ping Pod B on Node 1. Pod A cannot ping Pod C on Node 2. Same Deployment, same cluster.

**Root cause:** CNI routes are not propagated between nodes. Common causes: BGP peer session down (Calico), VXLAN tunnel not forming (Flannel), Security Group blocking the CNI port (cloud-specific).

**Fix:**
```bash
# Check CNI pod logs on both nodes
kubectl logs -n calico-system <calico-node-on-node1>
kubectl logs -n calico-system <calico-node-on-node2>

# For Calico: check BGP peer status
kubectl exec -n calico-system <calico-node-pod> -- calicoctl node status

# For AWS: check that Security Group allows:
# - UDP 4789 (VXLAN for Flannel/Cilium overlay)
# - TCP/UDP 179 (BGP for Calico)
# - IP protocol 4 (IP-in-IP for Calico)

# Verify routes on the host
ip route | grep <pod-cidr>
```

### Mistake 6 — etcd Quorum Lost (Most Dangerous)

**What happens:** In a 3-node control plane, 2 etcd nodes go down. API Server stops responding to writes. Cluster is read-only or completely down.

**Root cause:** etcd requires quorum — majority of members must be healthy. 3 nodes need 2. 5 nodes need 3.

**Fix:**
```bash
# Check etcd member health
etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list

etcdctl endpoint health --endpoints=<all-three-endpoints>

# If quorum is lost, restoration from backup is required
# This is a DR scenario — regular etcd backups are mandatory
```

### Mistake 7 — Assuming Kubernetes Manages Docker Directly (Post-1.24)

**What happens:** Engineer SSHes onto a node and runs `docker ps` to see running containers. Gets an empty list. Thinks containers aren't running. Panics.

**Root cause:** Kubernetes now uses containerd directly. Containers exist in containerd's namespace, not Docker's.

**Fix:**
```bash
# Use crictl instead of docker
crictl ps              # list running containers
crictl logs <id>       # get logs
crictl inspect <id>    # inspect container

# Or use nerdctl (containerd's Docker-compatible CLI)
nerdctl ps
nerdctl logs <id>
```

---

## 7. Interview Answers — Compressed, Verbatim-Ready

### Q: Explain the Kubernetes architecture in 60 seconds.

> "Kubernetes has two layers: the control plane and the worker nodes. The control plane is the brain — it has the API Server, which is the single entry point for all operations; etcd, which is the key-value store holding all cluster state; the scheduler, which assigns pods to nodes; and the controller manager, which runs all the controllers that maintain desired state. On each worker node you have the kubelet, which is the agent that enforces pod specs by talking to the container runtime; and kube-proxy, which manages the networking rules for Services. The actual container execution, pod networking, and persistent storage are handled by plugin interfaces — CRI, CNI, and CSI — which are swappable and not part of Kubernetes core itself."

### Q: What is the difference between CRI, CNI, and CSI?

> "All three are plugin interfaces that extend Kubernetes capabilities. CRI is the Container Runtime Interface — it's what kubelet calls to actually start and stop containers. containerd is the most common CRI today. CNI is the Container Network Interface — it's called by kubelet to assign an IP address to a pod and set up routing so pods can communicate. Calico and Cilium are common CNI plugins. CSI is the Container Storage Interface — it handles volume lifecycle: creating disks in the cloud, attaching them to nodes, and mounting them into pods. AWS EBS, Azure Disk, and Ceph all implement CSI. The key point is that kubelet calls all three through standardized interfaces, so each layer is independently replaceable."

### Q: Docker was removed from Kubernetes. Does that mean Docker images don't work?

> "No — this is a common misconception. What was removed is dockershim, which was the adapter that let kubelet talk to the Docker daemon. That was removed in Kubernetes 1.24 because containerd, which Docker itself uses internally, supports the CRI interface directly. Docker images follow the OCI image specification, which is a standard that every CRI runtime — containerd, CRI-O, and others — can pull and run. So the tool is gone from the runtime path, but the image format is universal and works everywhere."

### Q: What happens step by step when you run `kubectl apply -f pod.yaml`?

> "The kubectl client sends the request to the API Server. The API Server authenticates it, authorizes it via RBAC, runs admission controllers, validates the schema, and writes the Pod object to etcd with status Pending. The scheduler watches for Pending pods without a node assignment, selects the best node based on resource availability and constraints, and writes the node name back to etcd. The kubelet on that node sees the pod is assigned to it and calls the CRI to pull the image and start the container. Once the container sandbox is created, kubelet calls the CNI plugin to assign an IP and set up routing. If there are volumes, kubelet calls the CSI driver to attach and mount them. Finally, kubelet reports the pod as Running back to the API Server, and kube-proxy updates its iptables rules if the pod backs a Service."

### Q: Why does Kubernetes use a plugin architecture instead of building everything in?

> "Four reasons. Interoperability — the same Kubernetes code runs whether you're using containerd or Kata, Calico or Cilium. Vendor neutrality — no single company controls the runtime or network layer; AWS, GCP, and Azure all implement the same CSI interface. Independent innovation — Cilium can add eBPF-based observability and L7 policies without waiting for a Kubernetes release cycle. And maintainability — a bug in an EBS CSI driver is fixed by updating that driver, not by patching kubelet. The interfaces define the contract; the implementations compete freely."

### Q: What does the controller manager actually do?

> "The controller manager runs a set of control loops, each responsible for a specific resource type. The ReplicaSet controller watches the actual number of running pods against the desired count and creates or deletes pods to reconcile the difference. The Deployment controller manages ReplicaSets to enable rolling updates and rollbacks. The Node controller monitors node heartbeats and marks nodes NotReady if they stop reporting in, then evicts pods after a configurable timeout. Every controller follows the same pattern: watch the current state, compare it to desired state, take action to close the gap."

### Q: What is etcd and what would happen if it failed?

> "etcd is the distributed key-value store that holds all of Kubernetes' state — every pod, node, service, secret, configmap, and cluster event. The API Server is the only component that talks to etcd directly. If etcd loses quorum — meaning more than half the members are unavailable — the API Server stops accepting write operations. The cluster goes read-only or completely unresponsive. Existing workloads continue running because kubelet and kube-proxy operate from their last-known state, but no new pods can be scheduled, no changes can be made, and the cluster cannot self-heal. This is why etcd requires regular backups and a minimum of three nodes in production."

---

## 8. Debugging — Fast Diagnosis Paths

### Pod Not Running — CRI, CNI, CSI Triage

```
Pod status is not Running
        │
        ├── Pending
        │     ├── "Unschedulable" → Scheduler issue
        │     │     kubectl describe pod <pod> | grep -A5 Events
        │     │     → insufficient CPU/memory → check node capacity
        │     │     → taints/tolerations mismatch → check node taints
        │     │     → no nodes match affinity → check affinity rules
        │     │
        │     └── No events / stuck → CNI issue
        │           kubectl describe pod | grep -i network
        │           kubectl get pods -n <cni-namespace>
        │           journalctl -u kubelet | grep cni
        │
        ├── ContainerCreating (stuck)
        │     kubectl describe pod <pod>
        │     → "network plugin not ready" → CNI pod not running
        │     → "unable to mount volumes" → CSI issue
        │     → "failed to pull image" → CRI/network issue
        │     → "rpc error: containerd" → CRI issue
        │
        ├── CrashLoopBackOff
        │     → Application crashing, not a platform issue
        │     kubectl logs <pod> --previous
        │
        └── Error / OOMKilled
              → Resource limits too low
              kubectl describe pod | grep -i "limits\|oom"
```

### CRI Debugging Path

```bash
# Is the runtime running?
systemctl status containerd

# Is kubelet able to talk to CRI?
journalctl -u kubelet | grep -i "cri\|runtime\|container"

# List containers via CRI
crictl ps -a                    # all containers including stopped
crictl ps --name <container>    # filter by name

# Inspect a failing container
crictl inspect <container-id>
crictl logs <container-id>

# Check image exists
crictl images | grep <image-name>

# Pull image manually to test registry access
crictl pull <image>

# Check CRI endpoint kubelet is using
cat /var/lib/kubelet/config.yaml | grep endpoint
```

### CNI Debugging Path

```bash
# Is CNI configured?
ls /etc/cni/net.d/
# Empty = no CNI installed

# Is CNI plugin running?
kubectl get pods -n kube-system | grep -E "calico|cilium|flannel|weave"

# Are nodes Ready? (CNI required for Ready status)
kubectl get nodes

# Pod network not working — check pod IP assigned
kubectl get pod <pod> -o wide
# If no IP → CNI failed to assign

# Test connectivity
kubectl exec <pod-a> -- ping <pod-b-ip>
kubectl exec <pod-a> -- curl <pod-b-ip>:<port>

# Check kubelet's CNI call logs
journalctl -u kubelet | grep -i cni

# Calico-specific
kubectl exec -n calico-system <calico-node> -- calicoctl node status
kubectl exec -n calico-system <calico-node> -- calicoctl get workloadendpoints

# Cilium-specific
kubectl -n kube-system exec ds/cilium -- cilium status
kubectl -n kube-system exec ds/cilium -- cilium endpoint list
```

### CSI Debugging Path

```bash
# PVC stuck in Pending
kubectl describe pvc <name>
# "waiting for a volume to be created" → provisioner not running or StorageClass wrong
# "no volume plugin matched" → CSI driver not installed or wrong provisioner name

# Check CSI driver is installed
kubectl get csidrivers

# Check CSI controller pods are healthy
kubectl get pods -n kube-system | grep csi

# Check provisioner logs
kubectl logs -n kube-system <csi-controller-pod> -c external-provisioner

# Volume attached to node but not mounting into pod
kubectl describe pod <pod>
# Look for: "MountVolume.SetUp failed" → CSI node plugin issue
kubectl logs -n kube-system <csi-node-pod> -c <csi-driver-container>

# Check VolumeAttachment object
kubectl get volumeattachment

# PVC stuck in Terminating
kubectl get pvc <name> -o yaml | grep -A5 finalizers
kubectl patch pvc <name> -p '{"metadata":{"finalizers":null}}'
```

### Control Plane Health Check

```bash
# Quick overview
kubectl get pods -n kube-system

# API Server
kubectl cluster-info
kubectl get --raw /healthz

# Scheduler
kubectl get pods -n kube-system | grep scheduler
kubectl logs -n kube-system <scheduler-pod>

# Controller Manager
kubectl get pods -n kube-system | grep controller
kubectl logs -n kube-system <controller-manager-pod>

# etcd health
etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Kubelet on a node
systemctl status kubelet
journalctl -u kubelet -f --no-pager --since "10 min ago"
```

---

## 9. Kill Switch — 10-Second Recall

```
Core = decides WHAT happens
Extensions = decide HOW it happens

Control Plane:
  API Server     → entry point, writes to etcd
  etcd           → all state, only the API Server touches it
  Scheduler      → assigns pods to nodes
  Controller Mgr → reconciles actual vs desired state
  Cloud CCM      → cloud LB, node lifecycle

Node:
  kubelet        → enforces pods, calls CRI + CNI + CSI
  kube-proxy     → iptables rules for Services

Plugin interfaces (swappable):
  CRI → runs containers     (containerd, CRI-O, gVisor, Kata)
  CNI → pod IP + routing    (Calico, Cilium, Flannel)
  CSI → volume lifecycle    (EBS, Azure Disk, Ceph, Longhorn)

Triage:
  Pod not starting?         → CRI issue (crictl ps, journalctl kubelet)
  No IP / no network?       → CNI issue (ls /etc/cni/net.d, cni pod logs)
  Volume not mounting?      → CSI issue (csi pod logs, volumeattachment)

Docker removed in v1.24 — images still work (OCI standard)
Flannel ≠ NetworkPolicy support (need Calico or Cilium)
etcd quorum lost = cluster goes read-only or down
```

---

## 10. Appendix — Quick Reference Card

### Component to Namespace Map

| Component | Namespace | Type |
|---|---|---|
| kube-apiserver | kube-system | Static Pod (kubeadm) |
| etcd | kube-system | Static Pod (kubeadm) |
| kube-scheduler | kube-system | Static Pod (kubeadm) |
| kube-controller-manager | kube-system | Static Pod (kubeadm) |
| kube-proxy | kube-system | DaemonSet |
| CoreDNS | kube-system | Deployment |
| calico-node | calico-system | DaemonSet |
| CSI controller | kube-system | Deployment |
| CSI node plugin | kube-system | DaemonSet |

### Static Pod Manifests (kubeadm)

```bash
/etc/kubernetes/manifests/kube-apiserver.yaml
/etc/kubernetes/manifests/etcd.yaml
/etc/kubernetes/manifests/kube-scheduler.yaml
/etc/kubernetes/manifests/kube-controller-manager.yaml
```

> Editing these files directly causes kubelet to restart the static pod automatically.

### Key File Paths

```bash
# CNI
/etc/cni/net.d/           ← CNI configuration files
/opt/cni/bin/             ← CNI plugin binaries

# Kubelet
/var/lib/kubelet/config.yaml          ← kubelet config
/var/lib/kubelet/kubeconfig           ← kubelet's API Server credentials
/var/lib/kubelet/pki/                 ← kubelet TLS certs

# etcd
/etc/kubernetes/pki/etcd/             ← etcd TLS certs

# Kubeconfig
~/.kube/config                        ← default kubeconfig
/etc/kubernetes/admin.conf            ← admin kubeconfig (kubeadm)
```

### crictl Commands

```bash
crictl ps                    # list running containers
crictl ps -a                 # all containers (including stopped)
crictl images                # list pulled images
crictl pull <image>          # pull image
crictl logs <container-id>   # container logs
crictl inspect <id>          # full container details
crictl pods                  # list pod sandboxes
crictl stopp <pod-id>        # stop pod
crictl rmp <pod-id>          # remove pod
```

### CRI → CNI → CSI Call Sequence

```
kubectl apply → API Server → etcd (Pending)
             → Scheduler → nodeName written
             → kubelet sees pod
             → kubelet → CRI: create sandbox + container
             → kubelet → CNI: assign IP, set up routes
             → kubelet → CSI: attach + mount volume
             → kubelet → API Server: status = Running
```

### NetworkPolicy Support by CNI

| CNI Plugin | NetworkPolicy Support |
|---|---|
| Calico | ✅ Yes |
| Cilium | ✅ Yes (+ L7) |
| Flannel | ❌ No |
| Weave | ✅ Yes |
| AWS VPC CNI | ✅ Yes (with Calico for policy) |

### Storage Access Modes

| Mode | Abbreviation | Description |
|---|---|---|
| ReadWriteOnce | RWO | One node can mount read-write |
| ReadOnlyMany | ROX | Many nodes can mount read-only |
| ReadWriteMany | RWX | Many nodes can mount read-write |
| ReadWriteOncePod | RWOP | One pod can mount read-write (k8s 1.22+) |

> EBS only supports RWO. For RWX, use NFS, CephFS, or Longhorn.

### etcd Quorum Reference

| etcd Nodes | Quorum Needed | Fault Tolerance |
|---|---|---|
| 1 | 1 | 0 |
| 3 | 2 | 1 |
| 5 | 3 | 2 |
| 7 | 4 | 3 |

---
