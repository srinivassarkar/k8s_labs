# Kubernetes Resource Management 

---

## 0. The Five Rules Everything Else is Built On

Before anything else, memorize these five facts. Every behavior in this guide comes from them.

**Rule 1 — The scheduler only looks at requests, never limits.**
When Kubernetes decides which server (Node) to place your app (Pod) on, it only looks at `requests`. It never checks `limits`. It never checks real CPU/memory usage. Only `requests`.

**Rule 2 — The Linux kernel, not Kubernetes, enforces limits.**
Limits are not a Kubernetes feature at runtime. They are low-level OS boundaries (cgroups) set by the kubelet. When a container crosses a memory limit, the kernel kills it. Kubernetes just watches it happen and restarts the container.

**Rule 3 — CPU and memory behave completely differently.**
CPU is "compressible" — you can slow a process down without killing it. Memory is "incompressible" — you cannot take memory away from a process without terminating it. This single fact explains every OOM (Out of Memory) kill you will ever see.

**Rule 4 — LimitRange and ResourceQuota fire at creation time, not at runtime.**
These two tools only check Pods when they are being created. They have zero effect on Pods that are already running.

**Rule 5 — `requests` = the reservation. `limits` = the ceiling.**
Kubernetes guarantees a Pod its `requests` when placing it. `limits` are enforced by the kernel at runtime. Nothing above `requests` is guaranteed by Kubernetes.

---

## 1. What Kubernetes Actually Does (and Does NOT Do)

| Layer | What It DOES | What It Does NOT Do |
|---|---|---|
| Scheduler | Uses `requests` to find a Node with enough free Allocatable resources | Does not check `limits`; does not check real utilization |
| Kubelet | Translates `limits` into cgroup settings (`cpu.cfs_quota_us`, `memory.limit_in_bytes`) | Does not enforce `requests` at runtime |
| Kernel | Throttles CPU at the cgroup quota; OOM-kills when memory limit is breached | Does not understand Pods, namespaces, or QoS classes |
| API Server (Admission) | Runs LimitRange and ResourceQuota checks before saving the Pod | Does not retroactively affect running Pods when quotas change |
| LimitRange | Injects defaults, enforces min/max per container at creation time | Does not affect already-running Pods |
| ResourceQuota | Tracks total namespace usage and blocks creation if the cap is hit | Does not schedule, does not place, does not affect existing Pods |

### Important Edge Cases You Must Know

- **No limits + No LimitRange** → A Pod can consume the entire Node's memory. This is the "noisy neighbor" problem.
- **`limits` set, `requests` omitted** → Kubernetes automatically sets `requests == limits`. This gives you Guaranteed QoS (explained in Section 2).
- **`limit` < `request`** → The API Server immediately rejects the Pod with a validation error.
- **ResourceQuota exists but Pod has no `requests`** → Pod is rejected. The quota system can't track what it can't measure.
- **LimitRange + ResourceQuota together** → LimitRange injects defaults first, then ResourceQuota evaluates those injected values. So a Pod with no resources set can still be admitted if LimitRange fills in values that fit within the quota.
- **Reducing a quota does NOT evict existing Pods.** It only blocks new creations.
- **Node Capacity ≠ Allocatable.** The scheduler uses Allocatable. Capacity is the physical total; Allocatable subtracts what the kubelet and OS keep for themselves.

---

## 2. Decision Logic — When to Use What

### Which tool do you need?

```
Do you need to control a single Pod's resource usage?
  └── YES → Use requests/limits on the Pod spec
      └── Do you want defaults auto-applied in the namespace?
            └── YES → Use LimitRange (also enforces min/max per container)

Do you need to control total consumption across all Pods in a namespace?
  └── YES → Use ResourceQuota
      └── Do you also want to force Pods to declare resources?
            └── YES → Use ResourceQuota (quota with requests.cpu forces declarations)
                  + LimitRange (injects defaults so unlabeled Pods still pass)

Are you in a multi-tenant cluster with team namespaces?
  └── YES → LimitRange (per-Pod guardrails) + ResourceQuota (per-namespace cap) TOGETHER
```

### What happens step by step when you submit a Pod

```
Pod submitted to API Server
        │
        ▼
LimitRange Admission (inject defaults, check min/max)
        │
        ▼
ResourceQuota Admission (check namespace totals)
        │  If exceeded → 403 Forbidden (Pod never created)
        ▼
Pod written to etcd (status: Pending)
        │
        ▼
Scheduler watches Pending Pods
        │
        ▼
For each Node: Does Allocatable - AlreadyRequested >= Pod.requests?
        │  NO  → Skip node
        │  YES → Score node (affinity, taints, etc.)
        ▼
Best node selected → Pod bound to node
        │
        ▼
Kubelet sees bound Pod → sets cgroups (limits) → starts container
```

### QoS Classes — How Kubernetes Prioritizes Killing Pods Under Pressure

| Condition | QoS Class | Killed When? |
|---|---|---|
| `requests == limits` for ALL containers (both CPU and memory) | Guaranteed | Killed last |
| Any container has `requests < limits`, or only one resource type is set | Burstable | Killed second |
| No `requests` or `limits` on any container | BestEffort | Killed first |

> You cannot manually set the QoS class. Kubernetes assigns it automatically based purely on the relationship between your `requests` and `limits`.

### Full Feature Comparison

| Feature | Scope | Enforced By | Blocks Pod Creation? | Affects Scheduling? | Retroactive? |
|---|---|---|---|---|---|
| `requests` | Per-container | Scheduler (soft) | No | Yes (primary input) | N/A |
| `limits` | Per-container | Kernel (cgroups) | No (only if < request) | No | N/A |
| LimitRange | Namespace | Admission Controller | Yes (min/max violation) | Indirect (via injected requests) | No |
| ResourceQuota | Namespace | Admission Controller | Yes (when quota exceeded) | No | No |

---

## 3. How It Works Internally

### How the Scheduler Uses Requests (Step by Step)

1. The scheduler watches the API server for Pods where `nodeName == ""` (not yet placed).
2. For each candidate Node, it runs filter plugins. The `NodeResourcesFit` filter checks:
   - `Node.Allocatable.CPU - Sum(all existing Pod requests on node).CPU >= Pod.requests.CPU`
   - `Node.Allocatable.Memory - Sum(all existing Pod requests on node).Memory >= Pod.requests.Memory`
3. Nodes that pass the filter enter the **score** phase. Scoring plugins (`NodeResourcesBalancedAllocation`, `LeastAllocated`) also score based on — still — `requests`, not `limits`.
4. The highest-scored Node is selected. The Pod's `nodeName` is set (binding).

> The scheduler never reads actual CPU/memory utilization. Even if a Node is at 100% real CPU usage, if its request accounting shows headroom, the scheduler will still place Pods there.

### How Limits Are Enforced by the Kernel

When the kubelet starts a container:

1. It creates a cgroup hierarchy under `/sys/fs/cgroup/`.
2. **For a CPU limit of `500m`:** it sets `cpu.cfs_quota_us = 50000` and `cpu.cfs_period_us = 100000`. This means the container gets 50ms of CPU per 100ms window — exactly 0.5 cores, regardless of whether the rest of the Node is idle.
3. **For a memory limit of `256Mi`:** it sets `memory.limit_in_bytes = 268435456`. When the process tries to allocate beyond this, the kernel's OOM killer fires within the cgroup and kills the container process.
4. Kubernetes detects the exit code → sets container status to `OOMKilled` → kubelet restarts the container per the Pod's `restartPolicy`.

### How LimitRange Admission Works

1. User submits a Pod to the API Server.
2. The LimitRanger admission plugin intercepts it before it is stored.
3. For each container missing `requests` or `limits`: the plugin injects the default values from the matching LimitRange.
4. The plugin then validates: `min <= requests <= limits <= max`. If violated → `422 Unprocessable Entity`.
5. The modified Pod (now with injected values) proceeds to ResourceQuota admission.

### How ResourceQuota Tracks Usage

1. ResourceQuota stores a `status.used` counter in etcd for each tracked resource.
2. On every Pod creation/deletion, the quota controller increments/decrements `used`.
3. At admission time, the ResourceQuota plugin checks: `status.used + incoming Pod's resources <= spec.hard`.
4. If the check fails → `403 Forbidden`. The Pod object is never written to etcd.

> `used` is updated asynchronously by the quota controller. There is a brief window during high-throughput creation where quota enforcement can be slightly delayed (optimistic concurrency). In practice, this is handled via retry + conflict detection.

---

## 4. Configuration Examples

### Pod with Full Resource Specification

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
  namespace: team-a
spec:
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        cpu: "250m"
        memory: "128Mi"
      limits:
        cpu: "1"
        memory: "256Mi"
```

### LimitRange — Full Production Spec

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: team-a-limits
  namespace: team-a
spec:
  limits:
  - type: Container
    default:           # Applied when limits are omitted
      cpu: "500m"
      memory: "256Mi"
    defaultRequest:    # Applied when requests are omitted
      cpu: "100m"
      memory: "128Mi"
    min:               # Request/limit cannot go below this
      cpu: "50m"
      memory: "64Mi"
    max:               # Request/limit cannot exceed this
      cpu: "2"
      memory: "1Gi"
    maxLimitRequestRatio:  # limits/requests ratio cannot exceed this
      cpu: "4"
      memory: "2"
  - type: Pod          # Aggregate across all containers in Pod
    max:
      cpu: "4"
      memory: "2Gi"
  - type: PersistentVolumeClaim
    min:
      storage: "1Gi"
    max:
      storage: "10Gi"
```

### ResourceQuota — Full Production Spec

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: team-a
spec:
  hard:
    # Compute
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
    # Object counts
    pods: "20"
    services: "10"
    services.loadbalancers: "2"
    services.nodeports: "0"
    configmaps: "20"
    secrets: "30"
    persistentvolumeclaims: "10"
    # Storage
    requests.storage: "50Gi"
    # Scoped quota (only count BestEffort pods)
    # count/pods.BestEffort: "0"  # Uncomment to ban BestEffort pods
```

### Scoped ResourceQuota (QoS-Aware)

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: guaranteed-only
  namespace: production
spec:
  hard:
    pods: "10"
  scopeSelector:
    matchExpressions:
    - operator: In
      scopeName: PriorityClass
      values:
      - high-priority
```

### Storage Quota

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storage-quota
  namespace: team-a
spec:
  hard:
    requests.storage: "100Gi"
    persistentvolumeclaims: "5"
    # StorageClass-scoped
    gold.storageclass.storage.k8s.io/requests.storage: "50Gi"
    gold.storageclass.storage.k8s.io/persistentvolumeclaims: "2"
```

### Essential Commands

```bash
# Check node allocatable vs allocated
kubectl describe node <node-name> | grep -A 10 "Allocated resources"

# View quota status in a namespace
kubectl describe quota -n team-a

# View all LimitRanges in a namespace
kubectl describe limitrange -n team-a

# Check QoS class of a pod
kubectl get pod <name> -o jsonpath='{.status.qosClass}'

# Check what resources a pod actually has (after LimitRange injection)
kubectl get pod <name> -o yaml | grep -A 10 resources

# Live resource usage (requires metrics-server)
kubectl top pods -n team-a --sort-by=memory
kubectl top nodes

# Check metrics-server API registration
kubectl get apiservice v1beta1.metrics.k8s.io -o yaml

# Watch pod events for scheduling failures
kubectl describe pod <name> | grep -A 5 Events

# Check quota usage across all namespaces
kubectl get quota --all-namespaces

# Force delete and observe quota decrement
kubectl delete pod <name> && kubectl describe quota -n team-a
```

---

## 5. Production Setup

### Multi-Tenant Namespace Architecture

```
Cluster
├── namespace: team-frontend
│   ├── LimitRange: frontend-limits
│   │   └── Container defaults: 100m/128Mi req, 500m/512Mi lim
│   └── ResourceQuota: frontend-quota
│       └── requests.cpu: 8, requests.memory: 16Gi, pods: 50
│
├── namespace: team-backend
│   ├── LimitRange: backend-limits
│   │   └── Container defaults: 250m/256Mi req, 1/1Gi lim
│   └── ResourceQuota: backend-quota
│       └── requests.cpu: 16, requests.memory: 32Gi, pods: 30
│
└── namespace: team-data
    ├── LimitRange: data-limits
    │   └── Container max: 4 CPU, 8Gi memory
    └── ResourceQuota: data-quota
        └── requests.cpu: 20, requests.memory: 64Gi, pods: 10
```

### Recommended Resource Settings for Production

**For stateless web services:**

```yaml
resources:
  requests:
    cpu: "100m"      # What scheduler reserves — set to P50 actual usage
    memory: "128Mi"  # What scheduler reserves — set to P95 actual usage
  limits:
    cpu: "500m"      # Throttle ceiling — NOT set to prevent CPU throttling at burst
    memory: "256Mi"  # Hard memory cap — always set to prevent OOM propagation
```

> Many SRE teams intentionally omit `limits.cpu` on latency-sensitive services. CPU throttling from `cfs_quota` causes p99 latency spikes. Memory limits are always kept because unbounded memory growth is catastrophic.

**For batch/job workloads (Guaranteed QoS):**

```yaml
resources:
  requests:
    cpu: "2"
    memory: "4Gi"
  limits:
    cpu: "2"      # requests == limits → Guaranteed QoS
    memory: "4Gi" # Protected from OOM eviction under node pressure
```

### Monitoring Integration

- Set `requests` at **P50** of actual usage (from Prometheus `container_cpu_usage_seconds_total`)
- Set memory limits at **P99.9** of actual usage + 20% headroom
- Alert on `container_cpu_throttled_seconds_total > 25%` (CPU limit too low)
- Alert on `container_memory_usage_bytes / limits.memory > 0.85` (approaching OOM)

---

## 6. Common Mistakes

### Mistake 1: Setting requests too low / limits too high

**Symptom:** Pod is scheduled on a Node, but actual CPU usage far exceeds `requests`. Node becomes overcommitted. All Pods on the Node experience CPU contention.

**Root cause:** The scheduler packed too many Pods because `requests` were under-declared. `limits` had no effect on placement.

**Fix:** Set `requests` to reflect real P50 usage. Use VPA (Vertical Pod Autoscaler) recommendations as a baseline.

---

### Mistake 2: CPU limits causing latency spikes

**Symptom:** Service p99 latency is high, but average CPU usage is well below the limit. `container_cpu_throttled_seconds_total` metric is elevated.

**Root cause:** The Linux CFS scheduler enforces `cfs_quota` in 100ms windows. A short burst beyond the per-period quota causes the container to be throttled for the rest of that window — even if the Node has idle CPU sitting unused.

**Fix:** Either remove `limits.cpu` (and rely on `requests` for scheduling density) or significantly increase the CPU limit. Use `--cpu-cfs-quota=false` on the kubelet only for specific Nodes as a last resort.

---

### Mistake 3: ResourceQuota without LimitRange

**Symptom:** Pod with no resource declarations fails with: `must specify requests.cpu`

**Root cause:** When ResourceQuota tracks `requests.cpu`, the API server rejects Pods that don't declare it — because the quota controller cannot compute the delta. Without LimitRange, there are no defaults to inject.

**Fix:** Always deploy a LimitRange alongside ResourceQuota in the same namespace to inject defaults.

```bash
# Diagnose: check if quota exists without limitrange
kubectl get quota -n <ns>
kubectl get limitrange -n <ns>
```

---

### Mistake 4: Container OOMKilled despite Node having free memory

**Symptom:** `kubectl describe pod` shows `OOMKilled`. `kubectl top nodes` shows plenty of free memory on the Node.

**Root cause:** Memory limits are enforced by cgroups at the **container** level, not the Node level. The container hit its `limits.memory` ceiling regardless of what the Node has available.

**Fix:** Increase `limits.memory`. Check if the application has a memory leak. Set JVM heap (`-Xmx`) or Go `GOMEMLIMIT` to below the container limit.

```bash
# Confirm OOM kill
kubectl describe pod <name> | grep -i oom
kubectl get pod <name> -o jsonpath='{.status.containerStatuses[*].lastState.terminated.reason}'
```

---

### Mistake 5: Reducing ResourceQuota below current usage

**Symptom:** Existing Pods continue running. New Pods fail. Team reports "quota is at 0 but nothing is scheduled."

**Root cause:** Quota changes are not retroactive. Existing Pods hold their allocation. Reducing the quota only blocks incoming Pods until existing usage drops below the new hard limit.

**Fix:** This is expected behavior. Scale down existing Deployments before reducing the quota if you need headroom for new workloads.

---

### Mistake 6: LimitRange applied after Pods exist

**Symptom:** LimitRange is created but existing Pods don't have defaults applied.

**Root cause:** LimitRange is an admission-time control. It only fires during Pod creation, not retroactively.

**Fix:** Re-create Pods to trigger LimitRange injection.

```bash
kubectl rollout restart deployment/<name> -n <namespace>
```

---

## 7. Interview Questions & Answers

**Q: How does the Kubernetes scheduler decide where to place a Pod?**

The scheduler uses only the Pod's `requests` — not `limits` and not actual CPU or memory utilization. It looks at each Node's Allocatable resources, subtracts the sum of all existing Pod requests on that Node, and checks whether the remaining headroom can satisfy the incoming Pod's `requests`. If yes, the Node passes the filter phase. Nodes are then scored and the highest scorer wins. This means a Node running at 100% real CPU usage can still receive new Pods if the scheduler's request accounting shows headroom.

---

**Q: What happens when a container exceeds its CPU limit versus its memory limit?**

CPU and memory behave completely differently because CPU is compressible and memory is not. When a container exceeds its CPU limit, the Linux CFS scheduler throttles it — the process slows down but continues running. When a container exceeds its memory limit, the kernel's OOM killer terminates the container process immediately. Kubernetes then observes the exit, sets the container status to `OOMKilled`, and restarts it per the restart policy. You cannot take memory back from a process without killing it, which is why memory over-limit results in termination while CPU over-limit results only in throttling.

---

**Q: What are the three QoS classes and when does each matter?**

Guaranteed QoS is assigned when `requests` equal `limits` for every resource on every container. Burstable is assigned when at least one container has `requests` below `limits`, or has only one resource type specified. BestEffort is assigned when no `requests` or `limits` are set at all. This matters under Node memory pressure: the kubelet's eviction manager kills BestEffort Pods first, then Burstable, and protects Guaranteed Pods last. In production, critical stateful workloads should be Guaranteed to survive Node-level memory pressure events.

---

**Q: What is the difference between LimitRange and ResourceQuota?**

LimitRange is per-container and per-Pod — it injects default `requests` and `limits`, and enforces minimum and maximum values per container at creation time. ResourceQuota is per-namespace — it caps the total aggregate consumption of all Pods in the namespace. LimitRange prevents any single Pod from being too large or too small. ResourceQuota prevents the namespace as a whole from consuming more than its allocation. In production you use both together: LimitRange to ensure every Pod declares resources and has sane bounds, ResourceQuota to ensure the team's namespace stays within its cluster allocation.

---

**Q: What happens if you apply a ResourceQuota to a namespace that already has Pods with no resource requests?**

The existing Pods continue running — ResourceQuota changes are never retroactive. However, any new Pod created without resource `requests` will be rejected by the API server with an error saying it must specify `requests.cpu` or `requests.memory`, because the quota controller cannot track consumption without declared values. The fix is to also apply a LimitRange with defaults so the admission controller injects values before the quota check runs.

---

**Q: A Pod is stuck in Pending. Walk me through your diagnosis.**

First I run `kubectl describe pod <name>` and look at the Events section. The most common causes are:
- **Insufficient CPU or memory** — meaning the scheduler couldn't find a Node with enough headroom
- **Taints and tolerations mismatch** — the Pod doesn't tolerate a taint on all candidate Nodes
- **NodeSelector or affinity mismatch** — no Nodes match the required labels
- **ResourceQuota exceeded** — though in this case the Pod wouldn't reach Pending, it would be rejected at admission

If it's a resource issue, I run `kubectl describe nodes` to check Allocatable versus Allocated, and `kubectl get quota -n <namespace>` to check if namespace limits are the constraint.

---

## 8. Debugging Flowcharts

### Pod Stuck in Pending

```
kubectl get pod <name>  →  STATUS: Pending
        │
        ▼
kubectl describe pod <name>  →  check Events section
        │
        ├── "Insufficient cpu" or "Insufficient memory"
        │       └── kubectl describe nodes | grep -A 5 "Allocated resources"
        │           └── Is Allocated ≈ Allocatable? → node is full
        │               └── Options: scale node group, reduce Pod requests, add node
        │
        ├── "0/N nodes are available: N node(s) had untolerated taint"
        │       └── kubectl describe node <node> | grep Taints
        │           └── Add toleration to Pod or remove taint from node
        │
        ├── "0/N nodes are available: N node(s) didn't match node selector"
        │       └── Check Pod.spec.nodeSelector or affinity rules
        │           └── kubectl get nodes --show-labels | grep <required-label>
        │
        └── No events / empty events
                └── kubectl get events -n <namespace> --sort-by='.lastTimestamp'
                    └── Check for quota admission errors
```

### Pod OOMKilled / CrashLoopBackOff

```
kubectl get pod <name>  →  CrashLoopBackOff or Error
        │
        ▼
kubectl describe pod <name>  →  check Last State
        │
        ├── reason: OOMKilled
        │       └── Container hit memory limit
        │           ├── kubectl top pod <name>  (check actual usage trend)
        │           ├── Check app logs for memory leak: kubectl logs <name> --previous
        │           └── Increase limits.memory in Pod spec
        │
        └── Exit code: non-zero, not OOM
                └── kubectl logs <name> --previous
                    └── Application error — fix at app level
```

### Pod Rejected at Creation (403 / Quota Error)

```
kubectl apply -f pod.yaml  →  Error from server (Forbidden)
        │
        ▼
Error message contains "exceeded quota"
        │
        └── kubectl describe quota -n <namespace>
            └── Compare Used vs Hard for each resource
                └── Which resource is exhausted?
                    ├── requests.cpu / requests.memory  →  scale down other workloads or increase quota
                    ├── pods  →  delete unused pods or increase pod quota
                    └── configmaps/secrets  →  clean up unused objects
```

### ResourceQuota Admission Error (must specify requests)

```
Error: "must specify requests.cpu"
        │
        ▼
kubectl get quota -n <namespace>  →  quota tracks requests.cpu
        │
        ▼
kubectl get limitrange -n <namespace>  →  no LimitRange exists?
        │
        ├── No LimitRange → create one with defaultRequest.cpu
        │
        └── LimitRange exists → check if it has defaultRequest
            kubectl describe limitrange -n <namespace>
            └── Missing defaultRequest → add it
```

### kubectl top Fails

```
kubectl top pods  →  Error: Metrics API not available
        │
        ▼
kubectl get apiservice v1beta1.metrics.k8s.io
        │
        ├── Not found → metrics-server not installed
        │       └── kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
        │
        └── Found but STATUS: False
                └── kubectl describe apiservice v1beta1.metrics.k8s.io
                    └── Check "Conditions" for root cause
                        ├── TLS error → add --kubelet-insecure-tls to metrics-server args
                        └── metrics-server pod not running → kubectl logs -n kube-system -l k8s-app=metrics-server
```

---

## 9. Ten-Second Recall

If you remember nothing else, remember these ten lines:

```
SCHEDULING   →  requests only. Limits invisible to scheduler.
RUNTIME      →  CPU limit = throttle. Memory limit = OOM kill.
COMPRESSIBLE →  CPU (can throttle). INCOMPRESSIBLE → Memory (must kill).
QOS ORDER    →  BestEffort killed first. Guaranteed killed last.
LIMITRANGE   →  per-container, namespace-scoped, creation-time only, not retroactive.
RESOURCEQUOTA → per-namespace, blocks creation, tracks used vs hard.
QUOTA TRAP   →  quota exists → Pod MUST declare requests.
COMBO        →  LimitRange injects defaults → Quota evaluates totals. Order matters.
ALLOCATABLE  →  scheduler uses Allocatable, NOT Capacity.
LIMIT RULE   →  limits must be >= requests. Always.
```

---

## 10. Appendix

### Quick Command Reference

```bash
# Node inspection
kubectl describe node <node>                            # Full node detail including Allocatable and Allocated
kubectl get nodes -o custom-columns=NAME:.metadata.name,CPU:.status.allocatable.cpu,MEM:.status.allocatable.memory

# Pod resource inspection
kubectl get pod <name> -o yaml | grep -A 10 resources  # View injected resources
kubectl get pod <name> -o jsonpath='{.status.qosClass}' # QoS class
kubectl describe pod <name> | grep -A 20 Events         # Scheduling/admission events

# Quota and LimitRange
kubectl get quota --all-namespaces                      # All quotas across cluster
kubectl describe quota -n <namespace>                   # Quota with used vs hard
kubectl describe limitrange -n <namespace>              # LimitRange min/max/defaults
kubectl get quota <name> -n <namespace> -o yaml         # Raw quota spec + status

# Metrics (requires metrics-server)
kubectl top nodes                                       # Node CPU/memory actual usage
kubectl top pods -n <namespace> --sort-by=memory       # Pod usage sorted by memory
kubectl top pods -n <namespace> --containers            # Per-container breakdown

# API service
kubectl get apiservice v1beta1.metrics.k8s.io          # Metrics server registration
kubectl get apiservice | grep False                     # Any unhealthy API services

# Events
kubectl get events -n <namespace> --sort-by='.lastTimestamp' | tail -20
kubectl get events -n <namespace> --field-selector reason=FailedScheduling
```

### Resource Unit Reference

| Unit | Value |
|---|---|
| 1 CPU | 1 vCPU / 1 core |
| 1000m CPU | 1 vCPU (1000 millicores) |
| 100m CPU | 0.1 vCPU |
| Ki | Kibibytes (1024 bytes) |
| Mi | Mebibytes (1024² bytes) |
| Gi | Gibibytes (1024³ bytes) |
| K, M, G | Decimal prefixes (avoid — use Ki/Mi/Gi) |

### QoS Class Cheat Sheet

```
requests.cpu == limits.cpu  AND  requests.memory == limits.memory  →  Guaranteed
requests < limits  OR  only one resource defined                   →  Burstable
No requests, no limits                                             →  BestEffort
```

### ResourceQuota Trackable Resources

```yaml
# Compute
requests.cpu          limits.cpu
requests.memory       limits.memory

# Object counts
pods                  services               replicationcontrollers
configmaps            secrets                persistentvolumeclaims
services.loadbalancers                       services.nodeports

# Storage
requests.storage
<storageclass>.storageclass.storage.k8s.io/requests.storage
<storageclass>.storageclass.storage.k8s.io/persistentvolumeclaims
```

### LimitRange Types

```yaml
spec:
  limits:
  - type: Container   # Per-container limits
  - type: Pod         # Aggregate across all containers in the Pod
  - type: PersistentVolumeClaim  # Storage request bounds
```

### Admission Controller Order (Relevant Subset)

```
API Server receives request
    → Authentication
    → Authorization (RBAC)
    → Mutating Admission (LimitRanger injects defaults)  ← LimitRange fires here
    → Schema Validation
    → Validating Admission (ResourceQuota checks totals) ← ResourceQuota fires here
    → Written to etcd
```

> This order is why LimitRange defaults are visible to ResourceQuota — mutation runs before validation.

### Node Allocation Math

```
Given: Node with 4 CPU, 8Gi memory
Given: Pod requests 1 CPU, 2Gi memory

Max pods by CPU:    4 / 1 = 4 pods
Max pods by memory: 8 / 2 = 4 pods
Effective max:      min(4, 4) = 4 pods

Note: Pod limits (e.g., 3 CPU) have NO effect on this calculation.
      Scheduler only divides by requests.
```
