# Kubernetes Autoscaling
> Covers: HPA · VPA · Cluster Autoscaler · Metrics Server · Scaling Math · Production Patterns

---

## 0. First Principles

> These are the invariants. Everything else is implementation detail built on top of these.

**1. Autoscaling is a control loop, not an event.**
Every autoscaler operates on a reconcile loop — observe state, compare to desired state, act. HPA loops every 15 seconds. VPA's Recommender runs continuously. Cluster Autoscaler loops every 10 seconds. None of them react instantaneously. Understanding loop frequency is understanding latency.

**2. HPA scales out (horizontal). VPA scales up (vertical). They measure differently.**
HPA observes aggregate utilization across running Pods and adjusts replica count. VPA observes historical per-container usage and adjusts resource requests. They share the same workload but operate on fundamentally different dimensions.

**3. Requests are the currency of the entire autoscaling system.**
HPA computes CPU% as `usage / request`. VPA recommends new `request` values. Cluster Autoscaler determines node headroom based on `requests`. If requests are wrong, every autoscaler downstream is operating on corrupt input.

**4. Scaling up is reactive. Scaling down is conservative.**
All three autoscalers scale up aggressively and scale down cautiously (with cooldown windows). This asymmetry is intentional — false negatives (under-scaling) are more dangerous than false positives (over-scaling).

**5. The three autoscalers form a hierarchy, not a flat system.**
VPA right-sizes the Pod → HPA right-scales the replica count → Cluster Autoscaler right-sizes the node pool. Each layer depends on the layer below being correctly calibrated. A misaligned layer corrupts all layers above it.

**6. Kubernetes autoscaling knows nothing about your traffic. It reacts to symptoms.**
CPU utilization is a lagging indicator of load. By the time CPU spikes, users are already being throttled. Production systems use custom metrics (RPS, queue depth, latency) to get ahead of the curve.

---

## 1. Reality Constraints

### What Each Autoscaler Actually Does and Doesn't Do

| Autoscaler | Scales | Requires | Does NOT Do |
|---|---|---|---|
| **HPA** | Pod replica count | metrics-server (or custom metrics adapter) | Change CPU/memory requests; scale StatefulSets safely; predict load |
| **VPA** | Pod CPU/memory requests | Its own metrics pipeline (not metrics-server) | Change replica count; work conflict-free alongside HPA on CPU |
| **Cluster Autoscaler** | Node count in a node group | Cloud provider integration (or Cluster API) | Work in bare-metal environments without provider plugin; remove nodes with non-movable Pods |

### Critical Behavioral Boundaries

- **HPA cannot function without `resources.requests.cpu` defined.** CPU% is computed as `usage / request`. No request = division by zero = HPA stays idle.
- **HPA cannot scale below `minReplicas` or above `maxReplicas`.** These are hard walls. HPA will not breach them even if the algorithm says it should.
- **HPA scale-down has a default stabilization window of 5 minutes.** Spikes that resolve in under 5 minutes will not cause unnecessary scale-down. This is configurable.
- **VPA in `Auto` mode evicts and recreates Pods.** This is a disruption event. On a single-replica Deployment, this causes downtime.
- **VPA and HPA on CPU metrics simultaneously cause oscillation.** VPA reduces the request → HPA sees higher CPU% on the same usage → HPA scales up replicas → VPA recalculates across more Pods → loop.
- **Cluster Autoscaler will not remove a node that has Pods with local storage, or Pods with restrictive PodDisruptionBudgets.**
- **Cluster Autoscaler scale-up is triggered by Pending Pods, not by high node utilization.** A node at 95% CPU does not trigger Cluster Autoscaler. A Pod stuck in Pending does.
- **`kubectl top` requires metrics-server. HPA requires metrics-server.** VPA uses its own metrics pipeline. These are different data paths.
- **metrics-server in KIND (local) requires `--kubelet-insecure-tls`.** Production cloud clusters use proper TLS and do not need this.

---

## 2. Decision Logic

### Which Autoscaler to Use

```
What do you need to scale?
│
├── Replica count (more/fewer pods of the same size)
│     └── Use HPA
│           └── Based on what metric?
│                 ├── CPU utilization  → Resource metric, type: Utilization
│                 ├── Memory           → Resource metric, type: Utilization (unreliable — see §6)
│                 └── RPS / queue / latency → External or Custom metric (requires adapter)
│
├── Pod resource requests (right-size each pod's CPU/memory)
│     └── Use VPA
│           └── What disruption tolerance?
│                 ├── None           → updateMode: "Off" (recommendation only)
│                 ├── On new pods    → updateMode: "Initial" (inject at creation, no eviction)
│                 └── Full control   → updateMode: "Auto" (evict + recreate — disruptive)
│
└── Node count (cluster capacity itself)
      └── Use Cluster Autoscaler
            └── Triggered when: HPA has created Pending pods with no schedulable node
```

### HPA + VPA Compatibility Rules

```
HPA on CPU   + VPA on CPU   → NEVER. Oscillation guaranteed.
HPA on CPU   + VPA Off mode → SAFE. VPA recommends, human applies.
HPA on RPS   + VPA Auto     → SAFE. Different metrics, no feedback loop.
HPA on CPU   + VPA Initial  → MARGINAL RISK. Requests only change at pod creation.
No HPA       + VPA Auto     → SAFE for right-sizing single-replica workloads.
```

### Scaling Decision Flowchart

```
Traffic increases
        │
        ▼
CPU usage rises above HPA target threshold
        │
        ▼
HPA controller computes desiredReplicas
        │  desiredReplicas = currentReplicas × (currentMetric / targetMetric)
        ▼
Does desiredReplicas exceed maxReplicas?
  YES → cap at maxReplicas, some pods may remain overloaded
  NO  → update Deployment replica count
        │
        ▼
New Pods created by ReplicaSet controller
        │
        ▼
Can scheduler fit new Pods on existing nodes?
  YES → Pods scheduled, HPA done
  NO  → Pods enter Pending state
        │
        ▼
Cluster Autoscaler detects unschedulable Pods
        │
        ▼
Cloud provider API called → new node provisioned
        │
        ▼
Node joins cluster → kubelet registers → scheduler places Pending pods
```

---

## 3. Internal Working

### HPA Control Loop — Step by Step

1. The `kube-controller-manager` runs the HPA controller, which reconciles every **15 seconds** (configurable via `--horizontal-pod-autoscaler-sync-period`).
2. HPA queries the **Metrics API** (`metrics.k8s.io`) for current Pod CPU usage. This is served by metrics-server aggregating kubelet summary stats.
3. For each Pod in the target Deployment, it computes: `podCPU% = currentUsage / requests.cpu × 100`.
4. It averages this across all ready Pods (Pods not yet ready are excluded from the denominator to avoid scaling artifacts during rollout).
5. It applies the scaling formula:
   ```
   desiredReplicas = ceil(currentReplicas × (avgUtilization / targetUtilization))
   ```
6. The result is clamped to `[minReplicas, maxReplicas]`.
7. For **scale-up**: there is a short stabilization window (default **0 seconds** — scale-up is immediate).
8. For **scale-down**: the stabilization window is **300 seconds (5 minutes)** by default. HPA tracks the maximum replica recommendation over this window and will not scale below it until the window is stable.
9. If the computed delta is within a tolerance band (default ±10%), HPA skips the update to prevent thrashing.

### VPA Architecture — How the Three Components Interact

```
Historical CPU/memory usage
(from Prometheus or metrics-server)
        │
        ▼
┌─────────────────┐
│  Recommender    │  Watches all Pod metrics continuously.
│                 │  Applies histogram-based algorithm (percentile targeting).
│                 │  Writes recommendations to VPA object status.
└────────┬────────┘
         │  VPA status.recommendation updated
         ▼
┌─────────────────┐
│  Updater        │  Reads VPA recommendations.
│                 │  Compares current Pod requests to target.
│                 │  If drift exceeds threshold → evicts the Pod.
└────────┬────────┘
         │  Pod evicted → ReplicaSet creates new Pod
         ▼
┌─────────────────────────┐
│  Admission Controller   │  Intercepts new Pod creation via webhook.
│  (MutatingWebhook)      │  Injects VPA-recommended requests/limits.
└─────────────────────────┘
         │
         ▼
Pod starts with corrected resource requests
```

### Cluster Autoscaler — Scale-Up and Scale-Down Logic

**Scale-up trigger:**
- Cluster Autoscaler runs its loop every **10 seconds**.
- It watches for Pods in `Pending` state where the failure reason includes `Insufficient cpu` or `Insufficient memory`.
- It simulates placing these Pods on a hypothetical new node of each configured node group type.
- It selects the node group that satisfies the pending Pods with the least wasted resources.
- It calls the cloud provider API to provision the node and waits for it to register.

**Scale-down trigger:**
- A node is considered for removal when its utilization (sum of all Pod requests / allocatable) is below the threshold (default: **50%**) for a sustained period (default: **10 minutes**).
- Cluster Autoscaler simulates evicting all Pods from the node and placing them elsewhere. If this simulation succeeds → node is cordoned, Pods drained, node deleted.
- Nodes are **never removed** if they contain: Pods with local storage, Pods not managed by a controller, Pods with restrictive PodDisruptionBudgets that would be violated, or Pods with the `cluster-autoscaler.kubernetes.io/safe-to-evict: "false"` annotation.

### How CPU% Is Computed (The Number HPA Operates On)

```
Pod has: requests.cpu = 100m
Pod is using: 80m (from metrics-server)

CPU% = 80m / 100m = 80%

If target is 50%:
desiredReplicas = ceil(2 × (80 / 50)) = ceil(3.2) = 4

Now VPA changes request to 25m:
CPU% = 80m / 25m = 320%   ← same workload, same usage, 4× the apparent spike
HPA sees: desiredReplicas = ceil(2 × (320 / 50)) = ceil(12.8) = 13

This is the oscillation problem.
```

---

## 4. Hands-On

### KIND Multi-Node Cluster

```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: scaling-lab
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

```bash
kind create cluster --config kind-config.yaml
kubectl get nodes
```

### Metrics Server — Install and Patch for KIND

```bash
# Install
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Patch for KIND (insecure kubelet TLS — never do this in production)
kubectl patch deployment metrics-server -n kube-system \
  --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'

# Verify
kubectl rollout status deployment/metrics-server -n kube-system
kubectl top nodes
kubectl top pods -A
```

### Target Deployment

```yaml
# nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-hpa
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-hpa
  template:
    metadata:
      labels:
        app: nginx-hpa
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
        ports:
        - containerPort: 80
        readinessProbe:           # Critical: HPA excludes non-ready pods from avg calculation
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
```

```bash
kubectl apply -f nginx-deployment.yaml
kubectl expose deployment nginx-hpa --port=80 --name=nginx-hpa-svc
```

### HPA — autoscaling/v2 Full Spec

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-hpa
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: "200Mi"
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0      # Scale up immediately
      policies:
      - type: Percent
        value: 100                        # Double replicas per period at most
        periodSeconds: 15
    scaleDown:
      stabilizationWindowSeconds: 300    # Wait 5 min before scaling down
      policies:
      - type: Pods
        value: 1                          # Remove at most 1 pod per period
        periodSeconds: 60
```

```bash
kubectl apply -f hpa.yaml
kubectl get hpa nginx-hpa
kubectl describe hpa nginx-hpa   # Shows current metrics and conditions
```

### VPA — Full Spec with All Modes

```yaml
# vpa-recommend.yaml — safe for production observation
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: nginx-vpa
  namespace: default
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-hpa
  updatePolicy:
    updateMode: "Off"      # "Off" | "Initial" | "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: nginx
      minAllowed:
        cpu: "50m"
        memory: "64Mi"
      maxAllowed:
        cpu: "2"
        memory: "1Gi"
      controlledResources: ["cpu", "memory"]
```

```bash
kubectl apply -f vpa-recommend.yaml

# Wait for Recommender to gather data, then:
kubectl describe vpa nginx-vpa

# Read the recommendation
kubectl get vpa nginx-vpa -o jsonpath='{.status.recommendation.containerRecommendations}'
```

### VPA Install (Production — Not KIND Demo)

```bash
# Clone autoscaler repo for correct version
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler

# Install using the provided script (handles CRDs, RBAC, deployments)
./hack/vpa-up.sh

# Verify all three components
kubectl get pods -n kube-system | grep vpa
# Expected: vpa-recommender, vpa-updater, vpa-admission-controller
```

### Load Generation

```bash
# Generate sustained CPU load
kubectl run load-generator \
  --image=busybox \
  --restart=Never \
  -- /bin/sh -c "while true; do wget -q -O- http://nginx-hpa-svc; done"

# Watch HPA react in real time
watch -n 5 kubectl get hpa nginx-hpa

# Watch pods scale
kubectl get pods -w

# Clean up load
kubectl delete pod load-generator
```

### Essential Commands

```bash
# HPA status with current metrics
kubectl get hpa -o wide
kubectl describe hpa <name>

# VPA recommendation
kubectl describe vpa <name>
kubectl get vpa <name> -o yaml | grep -A 20 recommendation

# Cluster Autoscaler logs (on cloud)
kubectl logs -n kube-system -l app=cluster-autoscaler --tail=50

# Metrics server status
kubectl get apiservice v1beta1.metrics.k8s.io
kubectl top pods --sort-by=cpu
kubectl top pods --containers   # per-container breakdown

# Check HPA events
kubectl describe hpa <name> | grep -A 10 Events

# Simulate what scheduler sees (pending pods for CA)
kubectl get pods --field-selector=status.phase=Pending -A
```

---

## 5. Production Flow

### Layered Autoscaling Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                        Traffic Layer                         │
│              (Load Balancer / Ingress / API Gateway)         │
└─────────────────────────┬────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────────┐
│                   Application Pods                           │
│         nginx / api-server / worker / etc.                   │
└──────┬──────────────────┬────────────────────────────────────┘
       │                  │
       ▼                  ▼
┌────────────┐    ┌───────────────┐    ┌─────────────────────┐
│    HPA     │    │  VPA (Off)    │    │  Cluster Autoscaler │
│ Replica    │    │  Recommender  │    │  Node Provisioner   │
│ Scaling    │    │  Only         │    │  (cloud provider)   │
└────────────┘    └───────────────┘    └─────────────────────┘
       │                  │                      ▲
       │ scale replicas   │ engineer reviews     │
       │                  │ adjusts requests     │ Pending pods
       │                  ▼                      │ trigger CA
       └──────────────────────────────────────────┘
```

### Recommended Strategy by Workload Type

| Workload | HPA | VPA | Cluster Autoscaler | Notes |
|---|---|---|---|---|
| Stateless web / API | CPU or RPS metric | Off mode | Yes | Bread and butter |
| Background workers | Queue depth metric | Off mode | Yes | Custom metrics required |
| Stateful apps (DBs) | Manual or none | Off mode | Careful | CA may not drain DB nodes safely |
| Batch / ML jobs | None | Auto mode | Yes | VPA right-sizes per job run |
| Latency-critical APIs | RPS / latency metric | Off mode | Yes | Never use CPU-only HPA |

### Production-Safe Resource Pattern

```yaml
# For latency-sensitive stateless services:
resources:
  requests:
    cpu: "200m"      # Set at P50 of observed usage (from VPA recommendation)
    memory: "256Mi"  # Set at P95 of observed usage
  limits:
    cpu: "1"         # Burst headroom; omit if CFS throttling causes latency issues
    memory: "512Mi"  # Always set — unbounded memory = OOM risk to neighbors
```

> **Production Note on CPU Limits:** Many SRE teams at scale (Google, Netflix, Cloudflare) intentionally omit `limits.cpu` for latency-sensitive services. Linux CFS quota enforcement happens in 100ms windows — a 1ms burst beyond quota causes the container to be throttled for the remaining 99ms of that window, spiking p99 latency. The tradeoff: no limit means the noisy neighbor risk. Mitigate with proper node resource reservations and horizontal isolation via dedicated node pools.

### VPA-Informed Manual Request Tuning Workflow

```
1. Deploy with conservative requests (slightly over-provisioned)
2. Apply VPA in Off mode
3. Run for 7–14 days to capture traffic patterns
4. Review VPA recommendation (Target field)
5. Set requests = VPA Target, limits = VPA Upper Bound
6. Deploy updated resource spec via GitOps
7. Monitor for OOM or throttling
8. Repeat quarterly
```

---

## 6. Mistakes

### Mistake 1: Running HPA + VPA on CPU simultaneously

**Symptom:** Replica count oscillates wildly. Scaling events fire every few minutes with no load change. HPA shows high CPU% even when actual load is low.

**Root cause:** VPA reduces `requests.cpu` to match actual usage. HPA recomputes CPU% as `usage / newRequest`. The same absolute CPU usage now appears as a much higher percentage → HPA scales up replicas → VPA recalculates on the new fleet → oscillation loop.

**Fix:** Never combine HPA (CPU target) with VPA (Auto or Initial mode). Choose one of:
- HPA on CPU + VPA in `Off` mode (observe-only)
- HPA on custom metrics (RPS/latency) + VPA in `Auto` mode
- VPA alone for single-replica workloads

---

### Mistake 2: HPA defined but pods never scale

**Symptom:** `kubectl get hpa` shows `<unknown>` in the TARGETS column. No scaling occurs.

**Root cause (most common):** metrics-server is not installed or not healthy. Without `v1beta1.metrics.k8s.io` registered, HPA cannot fetch CPU data.

**Root cause (second):** The target Deployment's Pods have no `resources.requests.cpu`. HPA cannot compute CPU% without a denominator.

**Fix:**
```bash
kubectl get apiservice v1beta1.metrics.k8s.io   # should show Available=True
kubectl top pods                                  # should return data
kubectl get deploy <name> -o yaml | grep -A 5 resources  # verify requests exist
```

---

### Mistake 3: VPA in Auto mode causing downtime

**Symptom:** Single-replica Deployment has periodic downtime. VPA updater evicts the only running Pod to apply new resource values. No replacement is ready before eviction.

**Root cause:** VPA Updater evicts Pods that are outside recommendation bounds. On a single-replica Deployment with no PodDisruptionBudget, this causes a brief gap.

**Fix:**
- Use `updateMode: "Initial"` for single-replica workloads (inject at creation, never evict)
- Or define a PodDisruptionBudget with `minAvailable: 1`
- Or keep `updateMode: "Off"` and apply changes manually at deploy time

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: nginx-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: nginx-hpa
```

---

### Mistake 4: Cluster Autoscaler does not remove idle nodes

**Symptom:** Load drops. HPA scales replicas to minimum. Nodes sit at low utilization. Cluster Autoscaler does not remove them. Cloud bill stays high.

**Root cause (most common):** A DaemonSet Pod, system Pod, or Pod with `cluster-autoscaler.kubernetes.io/safe-to-evict: "false"` annotation is running on the node. CA cannot safely drain it.

**Fix:**
```bash
# Check what's on the suspected idle node
kubectl get pods --all-namespaces -o wide | grep <node-name>

# Annotate pods that are safe to evict (e.g., log collectors)
kubectl annotate pod <name> cluster-autoscaler.kubernetes.io/safe-to-evict="true"
```

---

### Mistake 5: HPA scale-up too slow under sudden traffic spike

**Symptom:** Traffic spikes rapidly. HPA takes 2–3 minutes to provision enough replicas. Users see errors during the ramp.

**Root cause:** Default HPA behavior is conservative. The 15-second sync period means at best one scaling decision per 15 seconds. New Pods also take time to reach Ready state (image pull, init containers, readiness probe).

**Fix:**
- Pre-scale before known traffic events (`kubectl scale deployment`)
- Set `behavior.scaleUp.stabilizationWindowSeconds: 0` (already default for scale-up)
- Increase `behavior.scaleUp.policies[].value` to allow larger jumps per period
- Use KEDA (Kubernetes Event-Driven Autoscaler) for event-driven workloads that need faster reaction

---

### Mistake 6: Memory-based HPA is unreliable

**Symptom:** HPA based on memory utilization does not scale down even after load drops.

**Root cause:** Memory is not released to the OS by most runtimes (JVM, Go, Python) even after garbage collection. The process holds the pages. `kubectl top` shows high memory usage even when the application is idle. HPA sees high memory → never scales down.

**Fix:** Use memory HPA only for workloads where memory directly correlates with and is released after load (rare). Prefer CPU or custom metrics for scaling decisions.

---

## 7. Interview Answers

**Q: How does HPA decide how many replicas to create?**

"HPA runs a control loop every 15 seconds. It queries the Metrics API for current CPU usage across all ready Pods in the target Deployment, then divides each Pod's usage by its declared CPU request to get a utilization percentage. It averages that across all ready Pods and applies the formula: desired replicas equals current replicas multiplied by current utilization divided by target utilization, with the result rounded up to the nearest integer. The result is then clamped to the min and max replica bounds. So if you have 2 replicas at 100% utilization with a 50% target, HPA computes 2 times 100 over 50, which is 4 replicas. This is why CPU requests are mandatory for HPA to function — without them, the denominator is zero and the calculation breaks."

---

**Q: What is the difference between HPA and VPA, and can you run them together?**

"HPA and VPA operate on different dimensions. HPA adjusts the number of replicas — it scales out. VPA adjusts the CPU and memory requests of each individual container — it scales up. You can run them together, but with a critical constraint: never run HPA with a CPU utilization target alongside VPA in Auto or Initial mode. The reason is an oscillation feedback loop. VPA will reduce the CPU request to match observed usage. HPA computes CPU percentage as usage divided by request. When the request drops, the same absolute CPU usage appears as a much higher percentage, causing HPA to scale up replicas. VPA then recalculates across the larger fleet, potentially adjusting requests again. The system oscillates. The safe combination is HPA on CPU with VPA in Off mode — VPA only recommends, a human applies the changes — or HPA on custom metrics like requests-per-second with VPA in Auto mode, since they're operating on independent inputs."

---

**Q: How does Cluster Autoscaler know when to add a node?**

"Cluster Autoscaler does not react to high CPU or memory utilization on existing nodes. It reacts specifically to Pods in Pending state where the scheduling failure is due to insufficient resources. Its loop runs every 10 seconds, and when it finds unschedulable Pods, it simulates placing those Pods onto a new hypothetical node of each configured node group type. It selects the node group that satisfies the Pods with the least waste, calls the cloud provider API to provision the node, and waits for it to register with the cluster. This means Cluster Autoscaler is downstream of HPA — HPA creates the Pods, those Pods go Pending because no node can fit them, and that's the signal Cluster Autoscaler acts on."

---

**Q: Why might HPA not scale even though the app is clearly under load?**

"There are four common causes. First, metrics-server might not be running — HPA cannot query utilization without it, and the TARGETS column will show unknown. Second, the target Deployment's Pods might not have CPU requests defined — HPA computes utilization as usage divided by request, so without a request there's nothing to compute against. Third, the HPA might already be at maxReplicas — it will not breach that boundary. Fourth, the stabilization window might be preventing scale-down from a previous event, though this wouldn't apply to scale-up. I'd start by running kubectl describe hpa and reading the Conditions and Events sections — the controller writes clear diagnostic messages there."

---

**Q: What are the three VPA components and what does each one do?**

"VPA has three independent components that work in a pipeline. The Recommender watches historical CPU and memory usage for all Pods targeted by VPA objects. It runs a percentile-based algorithm — effectively a histogram — and computes target, lower bound, and upper bound recommendations, which it writes to the VPA object's status. The Updater reads those recommendations and compares them to the current Pod's actual requests. If the drift is significant, it evicts the Pod — effectively telling the ReplicaSet controller to create a replacement. The Admission Controller is a mutating webhook that intercepts the creation of that replacement Pod and injects the recommended resource values before it starts. So the flow is: Recommender computes, Updater evicts, Admission Controller injects. If you run VPA in Off mode, only the Recommender runs — you get the recommendations without any eviction or injection."

---

**Q: What is CFS throttling and why does it matter for Kubernetes?**

"CFS stands for Completely Fair Scheduler. When you set a CPU limit on a Kubernetes container, the kubelet translates that into a cgroup quota expressed in Linux's CFS accounting units. Specifically, it sets cpu.cfs_quota_us and cpu.cfs_period_us, where the period is typically 100 milliseconds. If your container is limited to 500 millicores, it gets 50ms of CPU time per 100ms window. The problem is that even a brief burst beyond that quota in a given window causes the kernel to throttle the container for the remainder of that window — even if the node has completely idle CPU capacity. This is a kernel-level enforcement, not a Kubernetes-level one. The practical consequence is that latency-sensitive services with tight CPU limits experience p99 latency spikes that are invisible in average CPU metrics. The fix is to either set limits well above typical peak usage or, for latency-critical workloads, omit CPU limits entirely and rely on requests for scheduling density."

---

## 8. Debugging

### Diagnosis Path: HPA Shows `<unknown>` in TARGETS

```
kubectl get hpa  →  TARGETS: <unknown>/50%
        │
        ▼
kubectl describe hpa <name>  →  check Conditions section
        │
        ├── "unable to get metrics for resource cpu"
        │       └── metrics-server problem
        │           ├── kubectl get apiservice v1beta1.metrics.k8s.io
        │           │       └── Available: False → metrics-server not healthy
        │           │           └── kubectl logs -n kube-system -l k8s-app=metrics-server
        │           └── kubectl top pods
        │                   └── If fails → metrics-server not working
        │                       Fix: install or patch metrics-server
        │
        └── "missing request for cpu"
                └── Target Deployment pods have no resources.requests.cpu
                    └── kubectl get deployment <name> -o yaml | grep -A 5 resources
                        Fix: add resources.requests.cpu to Deployment spec
```

### Diagnosis Path: HPA Not Scaling Up Under Load

```
kubectl get hpa  →  TARGETS: 80%/50%  but replicas not increasing
        │
        ▼
kubectl describe hpa <name>  →  check Conditions and Events
        │
        ├── Condition: AbleToScale=False, reason=BackoffBoth
        │       └── HPA is in backoff due to rapid scaling events
        │           Wait for backoff to expire (check events for last scale time)
        │
        ├── Replicas at maxReplicas
        │       └── Cannot scale further — increase maxReplicas or reduce target
        │
        ├── Pods are not Ready
        │       └── kubectl get pods  →  pods in Init or not Ready
        │           HPA excludes non-ready pods, so avg CPU appears low
        │           Fix: investigate why pods are not becoming Ready
        │
        └── stabilizationWindowSeconds blocking
                └── Check behavior.scaleUp.stabilizationWindowSeconds in HPA spec
                    kubectl get hpa <name> -o yaml | grep -A 10 behavior
```

### Diagnosis Path: VPA Recommendation Shows No Data

```
kubectl describe vpa <name>  →  no recommendation section / empty
        │
        ▼
Is the Recommender running?
kubectl get pods -n kube-system | grep vpa-recommender
        │
        ├── Not running → VPA not installed or crash looping
        │       └── kubectl logs -n kube-system <vpa-recommender-pod>
        │
        └── Running → check if target pods have been running long enough
                VPA needs ~24 hours of history for reliable recommendations
                kubectl describe vpa <name> | grep -i "last update"
                Check targetRef matches deployment name exactly (case-sensitive)
```

### Diagnosis Path: Cluster Autoscaler Not Adding Nodes

```
Pods stuck in Pending  +  Cluster Autoscaler not reacting
        │
        ▼
kubectl describe pod <pending-pod>  →  confirm reason is resource insufficiency
        │
        ├── "Insufficient cpu/memory" → CA should be triggering
        │       └── kubectl logs -n kube-system -l app=cluster-autoscaler | grep -i "scale up"
        │           └── Look for: "No candidates for scale-up"
        │               ├── All node groups at maxSize → increase max node count
        │               ├── Node group labels don't match Pod nodeSelector → fix affinity
        │               └── Cloud provider API error → check CA logs for API failures
        │
        └── Other reason (taint, affinity)
                └── CA does not resolve taint/affinity issues by adding nodes
                    Fix the Pod spec or the node configuration
```

### Diagnosis Path: Cluster Autoscaler Not Removing Idle Nodes

```
Nodes underutilized for >10 minutes but CA not removing them
        │
        ▼
kubectl get pods -o wide | grep <node-name>
        │
        ├── DaemonSet Pods only → these are non-evictable system pods
        │       └── Expected behavior — CA skips nodes with only DaemonSet pods
        │           unless node truly has 0 non-DaemonSet pods
        │
        ├── Pod with annotation safe-to-evict: "false"
        │       └── kubectl get pods -o yaml | grep safe-to-evict
        │           Fix: add "true" annotation if safe, or remove the pod
        │
        └── kubectl logs -n kube-system -l app=cluster-autoscaler | grep "scale down"
                └── Look for: "pod ... blocking scale-down" → that pod is preventing removal
```

---

## 9. Kill Switch

> Ten-second recall. If you remember nothing else, remember this.

```
HPA       →  scales REPLICAS. Uses requests.cpu as denominator. Needs metrics-server.
VPA       →  scales REQUESTS. Has Recommender, Updater, Admission Controller.
CA        →  scales NODES. Triggered by PENDING pods, NOT high utilization.

HPA formula:  desiredReplicas = currentReplicas × (currentMetric / targetMetric)
CPU%:         usage / request  — wrong request = wrong scaling.

HPA + VPA CPU = OSCILLATION. Never do it.
HPA + VPA Off = SAFE. VPA recommends, human applies.
HPA custom + VPA Auto = SAFE. Different inputs.

Scale-UP:   immediate (0s stabilization).
Scale-DOWN: 5 minutes stabilization window default.

VPA modes:  Off (recommend only) | Initial (inject at creation) | Auto (evict + recreate).
CA blocks:  local storage pods, PDB violations, safe-to-evict=false annotation.

No metrics-server = no HPA. No requests.cpu = no HPA.
VPA needs ~24h of history for reliable recommendations.
```

---

## 10. Appendix

### Quick Command Reference

```bash
# HPA
kubectl get hpa -o wide                                   # Current replicas and targets
kubectl describe hpa <name>                               # Conditions, events, scaling history
kubectl get hpa <name> -o yaml | grep -A 20 behavior     # Scaling behavior config
kubectl autoscale deployment <name> --min=1 --max=5 --cpu-percent=50  # Quick HPA create

# VPA
kubectl describe vpa <name>                               # Full recommendation
kubectl get vpa <name> -o jsonpath='{.status.recommendation.containerRecommendations[*]}'
kubectl get vpa --all-namespaces

# Cluster Autoscaler
kubectl logs -n kube-system -l app=cluster-autoscaler --tail=100 | grep -i "scale"
kubectl get pods --field-selector=status.phase=Pending -A                # CA triggers
kubectl describe node <name> | grep -i "unschedulable"                   # Cordon status

# Metrics
kubectl top nodes
kubectl top pods --containers --sort-by=cpu
kubectl top pods -A --sort-by=memory
kubectl get apiservice v1beta1.metrics.k8s.io              # Metrics server registration

# Events
kubectl get events -n <ns> --sort-by='.lastTimestamp' | grep -i "Scaled\|FailedScale"
kubectl describe hpa <name> | grep -A 20 Events
```

### HPA Scaling Formula Reference

```
desiredReplicas = ceil(currentReplicas × (avgCurrentMetric / targetMetric))

Tolerance band: if ratio is within [0.9, 1.1], no action taken.

CPU% = (sum of pod CPU usage) / (sum of pod CPU requests) × 100
     = NOT based on node capacity
     = NOT based on limits
```

### VPA Recommendation Fields

| Field | Meaning | Action |
|---|---|---|
| `lowerBound` | Safe minimum — risk of OOM below this | Never set requests below this |
| `target` | Ideal value based on observed usage | Set as your `requests` value |
| `upperBound` | Safe maximum for burst protection | Set as your `limits` value |
| `uncappedTarget` | Raw algorithm output before min/max policy | Informational only |

### HPA Behavior Configuration Reference

```yaml
behavior:
  scaleUp:
    stabilizationWindowSeconds: 0       # 0 = scale up immediately
    selectPolicy: Max                   # Max = use the policy that scales most
    policies:
    - type: Pods
      value: 4                          # Add at most 4 pods per period
      periodSeconds: 60
    - type: Percent
      value: 100                        # Or double replicas per period
      periodSeconds: 60
  scaleDown:
    stabilizationWindowSeconds: 300     # 5 min cooldown before scaling down
    selectPolicy: Min                   # Min = use the policy that scales least
    policies:
    - type: Pods
      value: 1                          # Remove at most 1 pod per 60s
      periodSeconds: 60
```

### autoscaling/v2 Metric Types

```yaml
# CPU / Memory (built-in)
- type: Resource
  resource:
    name: cpu
    target:
      type: Utilization          # % of request
      averageUtilization: 50
      # OR
      type: AverageValue         # Absolute value per pod
      averageValue: "500m"

# Custom metric (e.g., requests per second — needs custom adapter)
- type: Pods
  pods:
    metric:
      name: requests_per_second
    target:
      type: AverageValue
      averageValue: "100"

# External metric (e.g., SQS queue depth)
- type: External
  external:
    metric:
      name: sqs_queue_depth
      selector:
        matchLabels:
          queue: my-queue
    target:
      type: AverageValue
      averageValue: "30"
```

### HPA Conditions Reference

| Condition | Meaning |
|---|---|
| `AbleToScale=True` | No backoff or constraints blocking scaling |
| `AbleToScale=False` | Backoff in effect or min/max blocking |
| `ScalingActive=True` | Metrics available and scaling is active |
| `ScalingActive=False` | Usually means metrics unavailable |
| `ScalingLimited=True` | Desired replicas were clamped by min/max |

### Autoscaler Comparison Matrix

| Dimension | HPA | VPA | Cluster Autoscaler |
|---|---|---|---|
| What it changes | Replica count | requests/limits | Node count |
| Trigger | Metric threshold | Usage vs request drift | Pending Pods |
| Loop frequency | 15 seconds | Continuous (recommender) | 10 seconds |
| Requires | metrics-server | VPA components | Cloud provider plugin |
| Disrupts running pods? | No | Yes (in Auto mode) | Yes (drains nodes) |
| Works without cloud | Yes | Yes | No (needs node groups) |
| Safe with stateful apps | Risky | Yes (Off mode) | Risky |
| Scale-down delay | 5 min (default) | N/A | 10 min (default) |
