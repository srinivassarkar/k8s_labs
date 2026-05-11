# ☸️ Kubernetes Intro

> *"Kubernetes is to containers what an air traffic control tower is to planes — it doesn't fly the planes, it ensures none of them crash into each other."*

---

## 0. First Principles — The Mental Model That Never Changes

These are the truths that survive every version upgrade, every cloud provider, every tooling change.

| Principle | What It Means |
|---|---|
| **Desired State > Actual State** | You declare what you want. Kubernetes figures out how to get there — and stays there. |
| **Everything is an API object** | Pods, Deployments, Services — all are just records in etcd. Controllers react to changes in those records. |
| **Controllers watch, compare, act** | Every controller runs a reconciliation loop: *watch → diff → patch*. This is the heartbeat of the cluster. |
| **The API Server is the single source of truth** | No component talks to another directly. Everything goes through the API Server. etcd is write-only from controllers' perspective. |
| **Pods are ephemeral; abstractions are stable** | Never depend on a Pod's IP or name. Depend on Services, Deployments, StatefulSets. |
| **The data plane runs applications; the control plane runs Kubernetes** | A total control plane failure doesn't kill running Pods — but nothing new can be scheduled. |

> Every interview answer about Kubernetes should eventually trace back to one of these six principles.

---
# The Mental Model That Never Changes

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

## 1. Reality Constraints — What Kubernetes Actually Does (and Doesn't Do)

### ✅ What Kubernetes Actually Handles
- Scheduling Pods onto nodes based on resource fit, affinity, taints/tolerations
- Restarting failed containers via Kubelet health checks
- Rolling updates and rollbacks via Deployment controller
- Internal DNS resolution and service discovery via CoreDNS
- Load balancing to Pod backends via kube-proxy (iptables/IPVS)
- Secrets and ConfigMap injection into Pods
- Persistent volume lifecycle management

### ❌ What Kubernetes Does NOT Do Out of the Box
- **Build or push container images** — that's your CI/CD pipeline
- **Monitor application performance** — you need Prometheus, Datadog, etc.
- **Provide a service mesh** — Istio/Linkerd are separate installs
- **Autoscale Pods without metrics server** — HPA requires a metrics source
- **Manage cross-cluster networking** — that's Submariner, Cilium ClusterMesh, etc.
- **Automatically scale nodes** — Cluster Autoscaler is a separate component

### Docker's Gaps That Kubernetes Fills

| Docker Gap | Kubernetes Solution |
|---|---|
| No auto-scaling | HorizontalPodAutoscaler (HPA) |
| No native multi-host load balancing | Services (ClusterIP, NodePort, LoadBalancer) + kube-proxy |
| No self-healing | ReplicaSet Controller + Kubelet health checks |
| No rolling updates/rollbacks | Deployment controller with `maxSurge`/`maxUnavailable` |

---

## 2. Decision Logic — When to Use What

### Which Orchestrator Should You Use?

```
Is this a local dev/test environment?
├── YES → Docker Compose
└── NO → Is this a small team, simple app, all Docker-native?
         ├── YES → Docker Swarm (but consider K8s anyway — ecosystem wins)
         └── NO → Kubernetes
```

### Kubernetes vs Docker Swarm vs Docker Compose

| Feature | Kubernetes | Docker Swarm | Docker Compose |
|---|---|---|---|
| **Scope** | Multi-host cluster | Single Swarm cluster | Single host |
| **Scaling** | Auto + Manual (HPA) | Manual + limited auto | Manual only |
| **Load Balancing** | Advanced (kube-proxy, Ingress) | Basic (routing mesh) | Not built-in |
| **Self-Healing** | Robust (ReplicaSet + Kubelet) | Basic container restarts | None |
| **Networking** | Advanced (CNI, Network Policies, Service Mesh) | Overlay network | Docker default |
| **Stateful Workloads** | StatefulSets + PVCs | External storage only | Volumes only |
| **Learning Curve** | Steep | Moderate | Minimal |
| **Production-Ready** | ✅ Yes | ⚠️ Small-medium only | ❌ Dev only |
| **Ecosystem** | Helm, Prometheus, Istio, Argo | Docker tooling only | Docker Compose |

### Which Kubernetes Object Should You Use?

```
Do you need to run a workload?
├── Stateless app → Deployment
├── Stateful app (needs stable identity/storage) → StatefulSet
├── One pod per node → DaemonSet
└── Run once and done → Job / CronJob

Do you need to expose a workload?
├── Internal cluster traffic only → ClusterIP Service
├── External traffic, direct node port → NodePort Service
├── External traffic, cloud LB → LoadBalancer Service
└── HTTP/HTTPS routing with path rules → Ingress
```

---

## 3. Internal Working — How It Actually Happens Under the Hood

### The Kubernetes Deployment Lifecycle (Step by Step)

```
kubectl apply -f deployment.yaml
        │
        ▼
┌─────────────────────────────────────────┐
│  kubectl                                │
│  1. Validates YAML syntax + schema      │
│  2. Sends object to API Server          │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│  API Server                             │
│  1. Authenticates + Authorizes request  │
│  2. Validates against full K8s schema   │
│  3. Writes Deployment → etcd            │
│  4. Returns "deployment created"        │
└─────────────────┬───────────────────────┘
                  │  (watches API Server)
                  ▼
┌─────────────────────────────────────────┐
│  Deployment Controller (kube-ctrl-mgr)  │
│  1. Detects new Deployment object       │
│  2. Compares desired vs actual Pods     │
│  3. Creates ReplicaSet object → API Svr │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│  ReplicaSet Controller                  │
│  1. Detects new ReplicaSet object       │
│  2. Creates Pod objects → API Server    │
│  3. Pods stored in etcd (no containers  │
│     running yet — just metadata)        │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│  Scheduler                              │
│  1. Watches for Pods with no nodeName   │
│  2. Evaluates: resources, affinity,     │
│     taints/tolerations                  │
│  3. Updates Pod object with nodeName    │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│  Kubelet (on assigned node)             │
│  1. Watches API Server for its Pods     │
│  2. Pulls container image via CRI       │
│  3. Configures networking via CNI       │
│  4. Mounts volumes                      │
│  5. Starts containers                   │
│  6. Reports health status → API Server  │
└─────────────────────────────────────────┘
                  │
                  ▼
         🟢 Pod is Running
```

### Key Principle at Every Step
> **No component writes to etcd directly.** All state changes flow through the API Server. Controllers *watch* the API Server (via list/watch), not etcd.

### Pod Network Namespace — The Invisible Layer
- Every Pod gets its **own network namespace**: unique IP, interfaces, routing table
- All containers **within** a Pod share this namespace → they communicate via `localhost`
- Containers in **different** Pods communicate via their Pod IPs (routed by CNI)
- Services provide stable virtual IPs in front of dynamic Pod IPs

### How kube-proxy + CNI Handle Pod-to-Service Traffic

```
Python Frontend Pod
    │  "redis-service:6379"
    ▼
CoreDNS resolves → ClusterIP (e.g. 10.96.0.10)
    │
    ▼
kube-proxy intercepts (iptables / IPVS rules)
    │  looks up Endpoints object
    │  selects healthy Redis Pod (load balancing)
    │  rewrites destination IP → Redis Pod IP
    ▼
CNI routes packet across cluster network
    │
    ▼
Redis Pod processes request
    │
    ▼ (return path is DIRECT — kube-proxy not involved)
Python Frontend Pod receives response
```

> **Why is the return path direct?** Because connection tracking (conntrack) in the kernel already knows the mapping. The response doesn't need to be re-routed through kube-proxy.

---

## 4. Hands-On — Production-Quality YAML + Commands

### Minimal Production Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: python-frontend
  namespace: production
  labels:
    app: python-frontend
    version: "1.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: python-frontend
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0          # Zero downtime rolling update
  template:
    metadata:
      labels:
        app: python-frontend
        version: "1.0"
    spec:
      containers:
        - name: python-frontend
          image: myrepo/python-frontend:1.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
      terminationGracePeriodSeconds: 30
```

### ClusterIP Service for Redis

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-service
  namespace: production
spec:
  selector:
    app: redis
  ports:
    - protocol: TCP
      port: 6379
      targetPort: 6379
  type: ClusterIP
```

### Essential Commands

```bash
# Apply and verify
kubectl apply -f deployment.yaml
kubectl get deployments -n production
kubectl get pods -n production -o wide          # Shows which node each Pod runs on
kubectl describe deployment python-frontend -n production

# Check rollout status
kubectl rollout status deployment/python-frontend -n production
kubectl rollout history deployment/python-frontend -n production

# Rollback
kubectl rollout undo deployment/python-frontend -n production
kubectl rollout undo deployment/python-frontend --to-revision=2 -n production

# Scale
kubectl scale deployment python-frontend --replicas=5 -n production

# Inspect Pod networking
kubectl exec -it <pod-name> -- ip addr
kubectl exec -it <pod-name> -- cat /etc/resolv.conf  # Confirm CoreDNS is the resolver
```

---

## 5. Production Flow — Real-World Architecture

### The Three-Layer Abstraction Stack

```
┌─────────────────────────────────┐
│        Deployment               │  ← You manage this
│   (versioning + rolling update) │
├─────────────────────────────────┤
│        ReplicaSet               │  ← Deployment manages this
│   (desired replica count)       │
├─────────────────────────────────┤
│           Pod(s)                │  ← ReplicaSet manages this
│   (shared network namespace)    │
├─────────────────────────────────┤
│        Container(s)             │  ← Kubelet + CRI manage this
└─────────────────────────────────┘
```

### Production Architecture Pattern: Frontend + Backend

```
[User] → [Ingress Controller]
               │
               ▼
    [Frontend Service]  (ClusterIP)
               │
     ┌─────────┴──────────┐
     ▼         ▼          ▼
  [Pod 1]   [Pod 2]   [Pod 3]   ← Deployment (3 replicas)
     │         │          │
     └─────────┴──────────┘
               │ redis-service:6379
               ▼
    [Redis Service] (ClusterIP)
               │
            [Pod]              ← StatefulSet (for Redis)
               │
         [PersistentVolume]
```

### Where Each Control Plane Component Lives

| Component | Runs On | Failure Impact |
|---|---|---|
| etcd | Control plane node(s) | Total cluster failure — all state lost |
| API Server | Control plane node(s) | No new objects can be created/modified |
| Scheduler | Control plane node(s) | New Pods get stuck as `Pending` |
| Controller Manager | Control plane node(s) | No reconciliation — drift accumulates |
| Cloud Controller Manager | Control plane node(s) | Cloud LBs/volumes stop responding |
| Kubelet | Every worker node | Pods on that node stop being managed |
| kube-proxy | Every node (DaemonSet) | Service routing breaks on that node |
| CoreDNS | Worker nodes (Deployment) | DNS resolution fails cluster-wide |

---

## 6. Mistakes — What Actually Breaks in Real Systems

### Mistake 1: `ImagePullBackOff` on First Deployment
**Root Cause:** Wrong image tag, private registry with no pull secret, or registry unreachable from the node.
**Fix:**
```bash
kubectl describe pod <pod-name>   # Look at Events section
kubectl create secret docker-registry regcred \
  --docker-server=<registry> \
  --docker-username=<user> \
  --docker-password=<pass>
# Add imagePullSecrets to Pod spec
```

### Mistake 2: Pods Running but Service Unreachable
**Root Cause:** `selector` in Service doesn't match Pod labels — silent mismatch.
**Fix:**
```bash
kubectl get endpoints redis-service   # If empty → label mismatch
kubectl get pods --show-labels        # Compare with Service selector
```

### Mistake 3: Scheduler Leaves Pods in `Pending`
**Root Cause:** No node has enough CPU/memory, or taints block all nodes.
**Fix:**
```bash
kubectl describe pod <pod-name>   # Events: "Insufficient cpu"
kubectl describe nodes            # Check allocatable vs requested
kubectl get nodes -o wide         # Check if nodes are Ready
```

### Mistake 4: Deployment Not Rolling Back Properly
**Root Cause:** Revision history is missing; change-cause was never annotated.
**Fix:**
```bash
kubectl apply -f deployment.yaml
kubectl annotate deployment python-frontend kubernetes.io/change-cause="v2 release"
kubectl rollout history deployment/python-frontend
```

### Mistake 5: kube-proxy iptables Rules Stale
**Root Cause:** kube-proxy Pod crashed; old iptables rules remain, pointing to dead Pods.
**Fix:**
```bash
kubectl get pods -n kube-system | grep kube-proxy
kubectl logs -n kube-system <kube-proxy-pod>
kubectl rollout restart daemonset kube-proxy -n kube-system
```

### Mistake 6: Treating Pods as Durable
**Root Cause:** Hard-coding Pod IPs in application config.
**Fix:** Always use Service DNS names (`redis-service.production.svc.cluster.local`). Pod IPs are ephemeral — they change every restart.

---

## 7. Interview Answers — Verbatim-Ready

**Q: What is Kubernetes and why do we need it?**
> "Kubernetes is an open-source container orchestration platform that automates the deployment, scaling, and lifecycle management of containerized applications across a cluster of machines. We need it because Docker alone can run containers but can't handle what happens at scale — it has no built-in auto-scaling, no self-healing when a container crashes, no intelligent load balancing across hosts, and no way to do rolling updates without downtime. Kubernetes solves all of these by introducing a control loop architecture where the system continuously compares the desired state you declare against the actual state of the cluster, and automatically reconciles any differences."

**Q: How does Kubernetes differ from Docker Swarm?**
> "Both are container orchestrators, but they operate at different scales and with different depths. Docker Swarm is simpler to set up and works well for small-to-medium deployments where you're already deep in the Docker ecosystem. Kubernetes is significantly more complex but offers advanced features like custom scheduling, fine-grained network policies, native stateful workload support via StatefulSets, and a rich ecosystem of tools like Helm, Prometheus, and Istio. In production at scale, Kubernetes wins on every dimension except simplicity of initial setup."

**Q: What is a Pod, and why doesn't Kubernetes run containers directly?**
> "A Pod is the smallest deployable unit in Kubernetes, and it's a wrapper around one or more containers that share the same network namespace and storage volumes. Kubernetes doesn't run containers directly because a Pod provides the necessary abstraction layer — specifically, a shared network namespace means all containers in a Pod can communicate over localhost, and they share the same IP address. This is important for patterns like sidecar containers, where a log shipper or proxy needs to sit alongside the main app container as if they were on the same machine."

**Q: Walk me through what happens when you run `kubectl apply -f deployment.yaml`.**
> "First, kubectl validates the YAML for syntax and schema correctness, then sends the Deployment object to the API Server. The API Server authenticates and authorizes the request, validates the full schema, and writes the desired state to etcd. The Deployment Controller, which is part of the Controller Manager, detects the new object via a watch on the API Server. It compares desired vs actual state and creates a ReplicaSet object. The ReplicaSet Controller then creates the required Pod objects — at this point, only metadata exists in etcd, no containers are running. The Scheduler picks up these unscheduled Pods, evaluates nodes based on resources, affinity, and taints, and updates each Pod object with a nodeName. Finally, the Kubelet on the assigned node picks up the Pod, calls the container runtime via CRI to pull the image and start the containers, configures networking via CNI, and reports the running status back to the API Server."

**Q: What is the role of kube-proxy?**
> "kube-proxy runs as a DaemonSet on every node and is responsible for implementing Kubernetes Service routing. When traffic is sent to a Service's ClusterIP, kube-proxy uses either iptables or IPVS rules to intercept the packet, select a healthy backend Pod from the Service's Endpoints object, and rewrite the destination to that Pod's IP. It also handles basic load balancing — round-robin by default with iptables, more sophisticated algorithms with IPVS. Importantly, kube-proxy only handles the outbound path; return traffic flows directly back through the kernel's connection tracking without going through kube-proxy again."

**Q: What is the difference between the control plane and the data plane?**
> "The control plane is the brain of the cluster — it's responsible for all orchestration decisions. It consists of etcd for state storage, the API Server as the single communication hub, the Scheduler for Pod placement, the Controller Manager for reconciliation loops, and optionally the Cloud Controller Manager for cloud provider integration. The data plane is where applications actually run — on worker nodes. Each worker node has a Kubelet that manages containers via the container runtime, a kube-proxy for service routing, and a CNI plugin for pod networking. A useful way to think about it: if the control plane dies, running workloads keep running, but you lose the ability to schedule, scale, or update anything."

---

## 8. Debugging — Fast Diagnosis Paths

### Pod Not Starting

```
kubectl get pods -n <namespace>
        │
        ├── STATUS: Pending
        │     └── kubectl describe pod <name> → check Events
        │           ├── "Insufficient cpu/memory" → scale nodes or reduce requests
        │           ├── "No nodes available" / taint issue → check node taints
        │           └── "Unbound PVC" → check PersistentVolumeClaims
        │
        ├── STATUS: ImagePullBackOff / ErrImagePull
        │     └── Wrong tag? Private registry? Missing imagePullSecret?
        │           kubectl describe pod <name> → Events → image pull error detail
        │
        ├── STATUS: CrashLoopBackOff
        │     └── kubectl logs <pod> --previous → check last crash output
        │           ├── App startup error → fix app config
        │           ├── OOM killed → increase memory limit
        │           └── Liveness probe too aggressive → tune initialDelaySeconds
        │
        └── STATUS: Running but not reachable
              └── Go to Service Debugging below
```

### Service Not Routing Traffic

```
kubectl get endpoints <service-name> -n <namespace>
        │
        ├── Endpoints list is EMPTY
        │     └── Label mismatch between Service selector and Pod labels
        │           kubectl get pods --show-labels
        │           kubectl get svc <name> -o yaml | grep selector
        │
        └── Endpoints exist but still failing
              ├── Is the container port correct in Service targetPort?
              ├── kubectl exec into Pod → curl localhost:<port> → app up?
              └── kube-proxy healthy?
                    kubectl get pods -n kube-system | grep kube-proxy
                    kubectl logs -n kube-system <kube-proxy-pod>
```

### Cluster-Level Diagnosis

```bash
# Control plane health
kubectl get componentstatuses            # etcd, scheduler, controller-manager
kubectl get nodes                        # All nodes Ready?
kubectl describe node <name>             # Conditions, allocatable resources

# Core system pods
kubectl get pods -n kube-system          # All running?

# Recent events (great for catching cascading failures)
kubectl get events -n <namespace> --sort-by='.lastTimestamp'

# Audit recent API activity
kubectl get events --all-namespaces --sort-by='.lastTimestamp' | tail -30
```

---

## 9. Kill Switch — 10-Second Recall

```
☸️  K8s in One Breath:
    You declare desired state → API Server stores it in etcd
    → Controllers reconcile → Scheduler places Pods
    → Kubelet starts containers → kube-proxy routes Service traffic
    → CNI connects everything

🧱  The Stack:
    Deployment → ReplicaSet → Pod → Container(s)

🧠  Control Plane:  etcd | API Server | Scheduler | Controller Manager | CCM
⚙️  Data Plane:     Kubelet | kube-proxy | Container Runtime | CNI

🔑  Pod = shared network namespace (one IP, containers talk via localhost)
🔑  Service = stable virtual IP in front of ephemeral Pod IPs
🔑  kube-proxy = handles outbound Service routing (iptables/IPVS)
🔑  CoreDNS = resolves Service names to ClusterIPs
🔑  CNI = gives Pods IPs and routes cross-node traffic

⚠️  Golden Rules:
    - Never use Pod IPs directly
    - All traffic goes through API Server, not directly to etcd
    - Control plane failure ≠ running Pods die (but nothing new schedules)
    - Return traffic bypasses kube-proxy (conntrack handles it)
```

---

## 10. Appendix — Quick Reference Card

### kubectl Cheatsheet

```bash
# Cluster info
kubectl cluster-info
kubectl get nodes -o wide
kubectl get componentstatuses

# Workloads
kubectl get pods -A                                  # All namespaces
kubectl get pods -n <ns> -o wide
kubectl describe pod <name> -n <ns>
kubectl logs <pod> -n <ns> --previous               # Last crashed container
kubectl exec -it <pod> -n <ns> -- /bin/sh

# Deployments
kubectl apply -f <file.yaml>
kubectl get deployments -n <ns>
kubectl rollout status deployment/<name> -n <ns>
kubectl rollout history deployment/<name> -n <ns>
kubectl rollout undo deployment/<name> -n <ns>
kubectl scale deployment <name> --replicas=5 -n <ns>

# Services & Networking
kubectl get svc -n <ns>
kubectl get endpoints <svc-name> -n <ns>
kubectl describe svc <name> -n <ns>

# Events (best debugging start point)
kubectl get events -n <ns> --sort-by='.lastTimestamp'

# Force delete a stuck pod
kubectl delete pod <name> -n <ns> --grace-period=0 --force
```

### Kubernetes Object Hierarchy

```
Cluster
└── Namespace
    ├── Deployment
    │   └── ReplicaSet
    │       └── Pod
    │           └── Container(s)
    ├── Service
    │   └── Endpoints
    ├── ConfigMap
    ├── Secret
    └── PersistentVolumeClaim → PersistentVolume
```

### DNS Naming Convention

```
<service-name>.<namespace>.svc.cluster.local:<port>

Example:
redis-service.production.svc.cluster.local:6379

Short form (within same namespace):
redis-service:6379
```

### Control Plane Component → Responsibility Matrix

| Component | Watches | Acts On | Writes To |
|---|---|---|---|
| API Server | All requests | Auth/validation | etcd |
| etcd | — | — | Itself |
| Scheduler | Unscheduled Pods | Node selection | Pod (nodeName field) |
| Deployment Controller | Deployments | Diff desired/actual | ReplicaSet objects |
| ReplicaSet Controller | ReplicaSets | Diff desired/actual | Pod objects |
| Kubelet | Pods on its node | Container lifecycle | Pod status |
| kube-proxy | Services + Endpoints | iptables/IPVS rules | Node kernel |
| CoreDNS | Services | DNS resolution | In-memory DNS cache |

---
