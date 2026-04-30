# Kubernetes Taints & Tolerations

> **Scope:** Scheduling control via node taints and pod tolerations — from mental model to production patterns to live debugging.

---

## 0. First Principles

> The mental model that never changes, regardless of Kubernetes version.

**The core invariant:**

> A pod lands on a node only when the node *accepts* it AND the scheduler *places* it there.

Taints and tolerations govern the **acceptance** side. Everything else — resource availability, affinity, topology — governs the **placement** side. Never conflate them.

**The three axioms:**

| Axiom | Statement |
|-------|-----------|
| **Repulsion is the default** | A tainted node rejects all pods that do not explicitly declare tolerance. Silence equals rejection. |
| **Toleration is permissive, not directive** | A toleration says "I *can* run here." It does not say "I *must* run here." Without affinity, a tolerated pod may still land anywhere. |
| **NoExecute is the only retroactive effect** | `NoSchedule` and `PreferNoSchedule` only affect future scheduling decisions. `NoExecute` reaches back and evicts already-running pods. |

**The mental shorthand:**

```
Taint    = node says "stay away"
Toleration = pod says "I can handle that"
Affinity = pod says "put me here specifically"

Strict isolation requires all three.
```

---

## 1. Reality Constraints

> What Kubernetes actually does and does not do — the gaps that cause production incidents.

### What Kubernetes Does

- Evaluates taints/tolerations **at scheduling time** for every pending pod.
- Evaluates `NoExecute` taints **continuously** against running pods.
- Automatically adds two system tolerations to every pod via the `DefaultTolerations` admission plugin:
  - `node.kubernetes.io/not-ready:NoExecute` with `tolerationSeconds: 300`
  - `node.kubernetes.io/unreachable:NoExecute` with `tolerationSeconds: 300`
- Propagates control-plane taint `node-role.kubernetes.io/control-plane:NoSchedule` automatically on cluster init.

### What Kubernetes Does NOT Do

| Assumption | Reality |
|------------|---------|
| "Adding a taint evicts all pods immediately" | Only `NoExecute` triggers eviction. `NoSchedule` leaves existing pods running. |
| "A toleration guarantees pod placement on a specific node" | Toleration only removes the barrier. Without `nodeAffinity` or `nodeName`, the pod can go anywhere untainted. |
| "Removing a taint reschedules evicted pods" | No. Evicted pods must be rescheduled by their controller (Deployment, StatefulSet). Standalone pods are gone. |
| "PreferNoSchedule will never place a pod on the tainted node" | It will, if no other node is available. It is a soft preference, not a hard rule. |
| "Tolerations on a Deployment template apply to all replicas" | They do — but only if defined in `spec.template.spec.tolerations`, not at the Deployment level. |
| "Multiple taints on one node require only one matching toleration" | Every taint on a node must have a matching toleration. One unmatched taint blocks scheduling (for `NoSchedule`) or triggers eviction (for `NoExecute`). |

### Multiple Taints — The AND Rule

```
Node has:  taint-A:NoSchedule + taint-B:NoSchedule

Pod must tolerate BOTH taint-A AND taint-B to be scheduled.
Tolerating only one is insufficient.
```

### System-Injected Tolerations

Every pod automatically gets (inspect with `kubectl get pod <pod> -o yaml`):

```yaml
tolerations:
- effect: NoExecute
  key: node.kubernetes.io/not-ready
  operator: Exists
  tolerationSeconds: 300
- effect: NoExecute
  key: node.kubernetes.io/unreachable
  operator: Exists
  tolerationSeconds: 300
```

This gives pods a **5-minute grace window** before eviction when a node becomes unreachable — critical to understand for HA architecture.

---

## 2. Decision Logic

> When to use what — hard rules, not opinions.

### Choosing the Right Effect

```
Is the requirement hard (pod MUST NOT land here)?
├── Yes → NoSchedule
│         └── Are there already running pods you need to move?
│             ├── Yes → NoExecute (with tolerationSeconds if graceful)
│             └── No  → NoSchedule is sufficient
└── No  → Is it a preference (scheduler should try elsewhere first)?
          ├── Yes → PreferNoSchedule
          └── No  → No taint needed; use affinity instead
```

### Choosing the Right Operator

| Situation | Operator | Why |
|-----------|----------|-----|
| Taint has a specific value (e.g., `storage=ssd`) and pod must match exactly | `Equal` | Prevents pods tolerating wrong value variants |
| Taint key exists but value is irrelevant (e.g., any `storage` taint) | `Exists` | Broadens toleration across all values of that key |
| Pod should tolerate ALL taints on a node | `Exists` with no `key` | Nuclear option — use only for privileged system pods (DaemonSets) |

### Taint Alone vs. Taint + Affinity

| Goal | Mechanism | Limitation |
|------|-----------|------------|
| Block unwanted pods from a node | Taint only | Tolerated pods can still drift to untainted nodes |
| Force pods onto specific nodes | NodeAffinity only | Doesn't block non-tolerated pods from those nodes |
| Hard isolation — pods ONLY on designated nodes, AND no others | Taint + Required NodeAffinity | This is the production pattern |

### When to Use `tolerationSeconds`

Use `tolerationSeconds` on `NoExecute` tolerations when:
- The workload can survive brief node disruptions (e.g., transient network partition)
- You want graceful draining before eviction during rolling maintenance
- You are tuning HA failover timing (default 300s is often too slow for critical services)

```yaml
# Tight HA window — evict after 60s if node stays unhealthy
tolerations:
- key: "node.kubernetes.io/unreachable"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 60
```

---

## 3. Internal Working

> How it actually happens under the hood — step by step.

### Scheduling Pipeline (Simplified)

```
Pod created → kube-scheduler watch loop picks it up
       │
       ▼
1. FILTER phase
   └── TaintToleration plugin evaluates each node:
       For each taint on the node:
         Is there a matching toleration on the pod?
         ├── No + effect is NoSchedule → Node ELIMINATED
         └── Yes → Continue to next taint
       All taints matched → Node passes filter

       ▼
2. SCORE phase
   └── TaintToleration plugin scores remaining nodes:
       PreferNoSchedule taints with NO matching toleration → lower score
       (Pod may still land there if it's the only node available)

       ▼
3. BIND phase
   └── Scheduler writes nodeName to pod spec → kubelet picks it up
```

### NoExecute Eviction Pipeline

```
Taint (NoExecute) added to node
       │
       ▼
Node controller detects taint change
       │
       ▼
For each pod on the node:
  Does pod have matching NoExecute toleration?
  ├── No → Evict immediately
  └── Yes + tolerationSeconds defined → Start countdown timer
              │
              ▼
         Timer expires → Evict pod
              │
         Timer not expired → Pod remains running
```

### Key Components Involved

| Component | Role |
|-----------|------|
| `kube-scheduler` | Runs TaintToleration filter + scorer during scheduling |
| `kube-controller-manager` (NodeLifecycle controller) | Adds system taints when nodes become unhealthy; triggers NoExecute eviction |
| `kubelet` | Reports node conditions; does NOT enforce taints directly |
| API Server (admission) | `DefaultTolerations` admission plugin injects system tolerations |

### How the Control Plane Taint Works

```bash
# On cluster init, kubeadm applies:
kubectl taint nodes <control-plane> node-role.kubernetes.io/control-plane:NoSchedule

# DaemonSets (kube-proxy, CNI) bypass this via:
tolerations:
- operator: "Exists"   # tolerates ALL taints — why DaemonSets run everywhere
```

---

## 4. Hands-On

> Production-quality YAML and commands. Nothing simplified.

### 4.1 Verify Cluster State

```bash
# List all nodes with roles
kubectl get nodes -o wide

# Check all taints across all nodes
kubectl get nodes -o=jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.taints}{"\n"}{end}'

# Verbose taint inspection on specific node
kubectl describe node <node-name> | grep -A5 -i taint
```

### 4.2 Apply and Remove Taints

```bash
# Apply taint
kubectl taint nodes worker1 storage=ssd:NoSchedule
kubectl taint nodes worker2 storage=hdd:NoSchedule
kubectl taint nodes worker1 critical=true:NoExecute

# Remove taint (note the trailing dash)
kubectl taint nodes worker1 storage=ssd:NoSchedule-
kubectl taint nodes worker1 critical=true:NoExecute-

# Remove ALL taints of a key regardless of effect
kubectl taint nodes worker1 storage-
```

### 4.3 Pod with Exact-Match Toleration (Equal)

```yaml
# pod-ssd.yaml
apiVersion: v1
kind: Pod
metadata:
  name: ssd-pod
  labels:
    app: ssd-workload
spec:
  tolerations:
  - key: "storage"
    operator: "Equal"
    value: "ssd"
    effect: "NoSchedule"
  containers:
  - name: nginx
    image: nginx:1.25
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "200m"
        memory: "256Mi"
```

```bash
kubectl apply -f pod-ssd.yaml
kubectl get pods -o wide   # Confirm node placement
```

### 4.4 Pod with Exists Operator (Any Value)

```yaml
# pod-exists.yaml
apiVersion: v1
kind: Pod
metadata:
  name: exists-pod
spec:
  tolerations:
  - key: "storage"
    operator: "Exists"
    effect: "NoSchedule"
  containers:
  - name: nginx
    image: nginx:1.25
```

### 4.5 NoExecute with Graceful Eviction Window

```yaml
# pod-noexecute.yaml
apiVersion: v1
kind: Pod
metadata:
  name: noexecute-pod
spec:
  tolerations:
  - key: "critical"
    operator: "Equal"
    value: "true"
    effect: "NoExecute"
    tolerationSeconds: 30    # Stays 30s after taint is added, then evicted
  containers:
  - name: nginx
    image: nginx:1.25
```

```bash
kubectl apply -f pod-noexecute.yaml
kubectl get pods -w   # Watch eviction after 30s
```

### 4.6 Production: Deployment with Taint + Required NodeAffinity

```yaml
# deploy-prod.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prod-app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: prod-app
  template:
    metadata:
      labels:
        app: prod-app
    spec:
      tolerations:
      - key: "env"
        operator: "Equal"
        value: "prod"
        effect: "NoSchedule"
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: env
                operator: In
                values:
                - prod
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: prod-app
              topologyKey: kubernetes.io/hostname
      containers:
      - name: app
        image: nginx:1.25
        resources:
          requests:
            cpu: "250m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
```

```bash
# Prerequisite: label + taint the node
kubectl label nodes worker1 env=prod
kubectl taint nodes worker1 env=prod:NoSchedule

kubectl apply -f deploy-prod.yaml
kubectl get pods -o wide -n production
```

### 4.7 DaemonSet Pattern (Tolerate All Taints)

```yaml
# DaemonSets that must run on every node including tainted ones
spec:
  template:
    spec:
      tolerations:
      - operator: "Exists"   # No key/value — matches ALL taints
```

---

## 5. Production Flow

> Real-world architecture and design patterns.

### Pattern 1: GPU Node Isolation

```
Scenario: GPU nodes are expensive. Only ML training jobs should use them.

Node setup:
  kubectl taint nodes gpu-node-1 accelerator=gpu:NoSchedule
  kubectl label nodes gpu-node-1 accelerator=gpu

Pod requirement (ML job):
  tolerations:
  - key: "accelerator"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: accelerator
            operator: In
            values: [gpu]

Result: GPU jobs land on GPU nodes. All other pods cannot.
```

### Pattern 2: Environment Isolation (Prod / Dev / Staging)

```
Infrastructure:
  Node pool A → tainted env=prod:NoSchedule, labeled env=prod
  Node pool B → tainted env=staging:NoSchedule, labeled env=staging
  Node pool C → tainted env=dev:NoSchedule, labeled env=dev

Deployment per env:
  - Toleration matches its pool's taint
  - Required NodeAffinity locks it to the matching label

Guarantee: No cross-environment pod placement, even during surge scaling.
```

### Pattern 3: Node Maintenance Drain

```bash
# Step 1: Prevent new pods from scheduling
kubectl taint nodes worker1 maintenance=drain:NoSchedule

# Step 2: Evict existing pods gracefully (built-in drain does this)
kubectl drain worker1 --ignore-daemonsets --delete-emptydir-data --grace-period=60

# Step 3: Perform maintenance

# Step 4: Re-enable node
kubectl uncordon worker1
kubectl taint nodes worker1 maintenance=drain:NoSchedule-
```

> **Note:** `kubectl drain` implicitly cordons the node AND evicts pods. The manual taint is useful when you want to stop new scheduling without evicting current pods.

### Pattern 4: Critical System Workload Protection

```
Problem: A misbehaving tenant workload starves a critical monitoring pod.

Solution:
  1. Taint the dedicated monitoring node: monitoring=reserved:NoSchedule
  2. Only monitoring pods carry the matching toleration + required affinity
  3. No tenant pod can land on this node regardless of resource pressure
```

### Pattern 5: Spot/Preemptible Node Handling

```yaml
# Nodes provisioned on spot instances get auto-tainted by cloud providers:
# cloud.google.com/gke-spot:NoSchedule   (GKE)
# eks.amazonaws.com/capacityType=SPOT:NoSchedule  (EKS, via Karpenter)

# Batch jobs that can tolerate interruption:
tolerations:
- key: "cloud.google.com/gke-spot"
  operator: "Exists"
  effect: "NoSchedule"
- key: "node.kubernetes.io/unreachable"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 0    # Immediate failover — don't wait 300s
```

---

## 6. Mistakes

> What actually breaks in real systems — root cause and fix.

### Mistake 1: Taint Added, Existing Pods Not Evicted (Surprise)

**Symptom:** Added `NoSchedule` taint expecting running pods to leave. They didn't.

**Root cause:** `NoSchedule` does not evict running pods. Only `NoExecute` does.

**Fix:**
```bash
# If you need running pods to leave, use NoExecute OR manually drain
kubectl taint nodes worker1 key=value:NoExecute
# OR
kubectl drain worker1 --ignore-daemonsets
```

---

### Mistake 2: Toleration Without Affinity — Pod Lands on Wrong Node

**Symptom:** Pod tolerated the GPU taint but scheduled on a non-GPU node.

**Root cause:** Toleration only removed the scheduling barrier. Without affinity, the scheduler picked a cheaper/available untainted node.

**Fix:** Always pair a toleration with `requiredDuringSchedulingIgnoredDuringExecution` affinity for strict placement.

---

### Mistake 3: One of Multiple Taints Not Tolerated

**Symptom:** Pod is `Pending`. Node has two taints. Pod tolerates one.

**Root cause:** Every taint must be matched. Partial toleration = still blocked.

**Diagnosis:**
```bash
kubectl describe pod <pod> | grep -A10 "Events"
# Look for: "1 node(s) had untolerated taint"
```

**Fix:** Add the missing toleration.

---

### Mistake 4: Toleration in Wrong YAML Location

**Symptom:** Toleration defined in Deployment spec, pods still pending.

**Root cause:** Tolerations must be in `spec.template.spec.tolerations`, NOT in `spec.tolerations` (which doesn't exist at Deployment level).

**Fix:**
```yaml
# WRONG — this field doesn't exist
spec:
  tolerations: [...]

# CORRECT — inside template
spec:
  template:
    spec:
      tolerations: [...]
```

---

### Mistake 5: Removing Taint Without Trailing Dash

**Symptom:** Running `kubectl taint nodes worker1 key=value:NoSchedule` a second time throws "already exists" error. Trying to remove gives unexpected behavior.

**Fix:**
```bash
# The trailing dash is the removal syntax
kubectl taint nodes worker1 key=value:NoSchedule-

# Common error — forgetting the dash tries to ADD the taint again
kubectl taint nodes worker1 key=value:NoSchedule   ← WRONG (add)
kubectl taint nodes worker1 key=value:NoSchedule-  ← CORRECT (remove)
```

---

### Mistake 6: DaemonSet Pods Pending on Control Plane

**Symptom:** Custom DaemonSet pods are `Pending` on the control-plane node.

**Root cause:** Control plane has `node-role.kubernetes.io/control-plane:NoSchedule`. DaemonSet doesn't tolerate it.

**Fix:**
```yaml
tolerations:
- key: "node-role.kubernetes.io/control-plane"
  operator: "Exists"
  effect: "NoSchedule"
```

---

### Mistake 7: tolerationSeconds on NoSchedule (No Effect)

**Symptom:** Set `tolerationSeconds: 60` on a `NoSchedule` toleration, expecting time-limited access.

**Root cause:** `tolerationSeconds` is only valid for `NoExecute`. It is silently ignored on `NoSchedule` and `PreferNoSchedule`.

**Fix:** Use `NoExecute` if time-based eviction is the goal.

---

## 7. Interview Answers

> Compressed, verbatim-ready answers for common questions.

---

**Q: What is the difference between a taint and a toleration?**

> "A taint is applied to a node and acts as a repellent — it prevents pods from being scheduled there unless they explicitly declare they can tolerate it. A toleration is applied to a pod and says 'I'm okay with this taint.' The key insight is that a toleration is permissive, not directive — it removes a barrier but doesn't force placement. If you want a pod to actually land on a specific tainted node, you need a toleration *and* node affinity together."

---

**Q: What are the three taint effects and when do you use each?**

> "`NoSchedule` prevents new pods without a matching toleration from being scheduled on the node — it doesn't touch already-running pods. `PreferNoSchedule` is a soft version — the scheduler tries to avoid placing pods there but will if no other option exists. `NoExecute` is the aggressive one — it not only blocks new scheduling but also evicts existing pods that don't tolerate it. You can add `tolerationSeconds` to give a pod a grace window before eviction, which is useful for handling transient node conditions without immediately disrupting workloads."

---

**Q: If I taint a node with NoSchedule, will existing pods be evicted?**

> "No. `NoSchedule` only affects future scheduling decisions. Pods already running on the node will continue running — they are not retroactively evaluated. If you need to evict running pods, you must use `NoExecute`, which continuously evaluates all running pods against the node's taints and evicts those without a matching toleration."

---

**Q: How do you achieve strict workload isolation in Kubernetes?**

> "Tolerations alone are not enough because a tolerated pod can still land on any untainted node. For strict isolation, you need three things together: first, taint the dedicated node so unwanted pods can't land on it; second, add the matching toleration to your target pods; third, add required node affinity so your pods are *forced* onto that specific node. Without affinity, your tolerated pod is free to schedule anywhere the scheduler prefers, which defeats the isolation goal."

---

**Q: What happens to a pod when a NoExecute taint is added to its node?**

> "If the pod doesn't have a matching `NoExecute` toleration, it's evicted immediately. If it has the toleration but with a `tolerationSeconds` value, it's allowed to stay for that many seconds before being evicted. If it has the toleration without `tolerationSeconds`, it stays indefinitely. One important thing to note is that Kubernetes automatically injects two NoExecute tolerations on every pod for `node.kubernetes.io/not-ready` and `node.kubernetes.io/unreachable` with a 300-second window — this gives pods a 5-minute buffer before being evicted when a node goes down."

---

**Q: What is the difference between the Equal and Exists operators in tolerations?**

> "`Equal` requires that both the key and the value match exactly — if a node has `storage=ssd:NoSchedule`, an `Equal` toleration for `storage=hdd` won't match. `Exists` only checks for the presence of the key — it ignores the value entirely. So a toleration with `key: storage, operator: Exists` would match both `storage=ssd` and `storage=hdd`. If you omit the key entirely and use `Exists`, the pod tolerates every taint on the node — this is what DaemonSets typically use to ensure they run everywhere."

---

**Q: Can a pod with tolerations be blocked from a tainted node in any scenario?**

> "Yes. If the node has *multiple taints*, the pod must tolerate every one of them. Missing even a single taint's toleration will block scheduling. Also, toleration only addresses the taint barrier — a pod can still be blocked by insufficient resources, node selectors that don't match, or affinity rules that conflict. Tolerations are one layer in the scheduling stack, not a master override."

---

## 8. Debugging

> Fast diagnosis paths — commands and decision trees.

### Pod is Pending — Taint-Related Diagnosis

```
kubectl get pods → STATUS: Pending
          │
          ▼
kubectl describe pod <pod> → look at Events section
          │
          ├── "0/N nodes are available: N node(s) had untolerated taint"
          │         │
          │         ▼
          │   kubectl get nodes -o=jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.taints}{"\n"}{end}'
          │   → Identify which node has which taints
          │         │
          │         ▼
          │   kubectl get pod <pod> -o yaml | grep -A20 tolerations
          │   → Compare pod tolerations against node taints
          │         │
          │         ├── Missing toleration for a taint → Add toleration to pod spec
          │         ├── Wrong value on Equal toleration → Fix value or switch to Exists
          │         └── Toleration in wrong YAML path → Move to spec.template.spec.tolerations
          │
          ├── "0/N nodes are available: N Insufficient cpu/memory"
          │   → Taint is NOT the issue. Check resource requests.
          │
          └── Multiple reasons listed
              → Address taint issue first, then reassess
```

### Pod Running on Wrong Node

```
kubectl get pods -o wide → Unexpected NODE value
          │
          ▼
Does pod have correct toleration?
  ├── No → Pod shouldn't be on a tainted node at all (check node taints)
  └── Yes → Does pod have required nodeAffinity?
              ├── No → Pod can land anywhere — add required affinity
              └── Yes → Check if node label matches affinity expression
                        kubectl get node <node> --show-labels
```

### Pod Unexpectedly Evicted

```
kubectl describe pod <pod> → "Evicted" or "Reason: Evicted"
          │
          ▼
kubectl get events --field-selector reason=Evicted
          │
          ▼
Was a NoExecute taint recently added to the node?
  ├── Yes → Check if pod had matching NoExecute toleration
  │          kubectl get node <node> -o yaml | grep -A5 taints
  └── No  → Check if node became NotReady/Unreachable
             kubectl describe node <node> | grep -A5 Conditions
             → NodeLifecycle controller adds system NoExecute taints
                when node goes unhealthy
```

### Useful Diagnostic Commands

```bash
# All taints across all nodes
kubectl get nodes -o=jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.taints}{"\n"}{end}'

# Pending pods with reasons
kubectl get pods -A --field-selector=status.phase=Pending

# Describe pending pod — most useful single command
kubectl describe pod <pod-name>

# Node labels (needed to debug affinity + taint combos)
kubectl get nodes --show-labels

# Events filtered by type
kubectl get events -n <namespace> --sort-by='.lastTimestamp'

# Simulate scheduling without applying (dry run)
kubectl run test --image=nginx --dry-run=server -o yaml

# Check if a specific node would accept a pod (uses server-side validation)
kubectl apply -f pod.yaml --dry-run=server
```

### Common Event Messages and Meaning

| Event Message | Meaning |
|---------------|---------|
| `0/2 nodes are available: 2 node(s) had untolerated taint` | Pod missing toleration for every node's taint |
| `0/2 nodes are available: 1 node(s) had untolerated taint, 1 Insufficient memory` | Mixed issue — one node taint-blocked, one resource-blocked |
| `node(s) didn't match Pod's node affinity/selector` | Affinity misconfigured; node labels may not match |
| `Insufficient cpu` on all nodes | Not a taint issue — resource requests too high |

---

## 9. Kill Switch

> 10-second recall — the absolute minimum to hold in memory.

```
TAINT   → on NODE   → repels pods
TOLERATE → on POD   → permits landing

3 Effects:
  NoSchedule      = blocks new pods (existing stay)
  PreferNoSchedule = soft block (scheduler tries to avoid)
  NoExecute       = blocks new + evicts running (tolerationSeconds = grace period)

2 Operators:
  Equal  = key + value must match
  Exists = key presence only (no value check)
  (no key + Exists = tolerate everything)

GOLDEN RULE:
  Toleration alone ≠ isolation
  Taint + Toleration + Required NodeAffinity = TRUE isolation

REMOVE TAINT:  kubectl taint nodes <node> key=value:Effect-   ← note the dash

MULTIPLE TAINTS: Pod must tolerate ALL of them. Missing one = blocked.

AUTO-INJECTED: Every pod gets not-ready + unreachable NoExecute tolerations (300s)
```

---

## 10. Appendix

> Quick reference card — commands, formats, cheatsheets.

### Taint Management Commands

```bash
# Add
kubectl taint nodes <node> <key>=<value>:<effect>

# Remove (trailing dash)
kubectl taint nodes <node> <key>=<value>:<effect>-

# Remove all taints for a key
kubectl taint nodes <node> <key>-

# View taints
kubectl describe node <node> | grep -i taint
kubectl get node <node> -o=jsonpath='{.spec.taints}'
```

### Toleration YAML Reference

```yaml
tolerations:
# Exact match
- key: "mykey"
  operator: "Equal"
  value: "myvalue"
  effect: "NoSchedule"

# Key exists (any value)
- key: "mykey"
  operator: "Exists"
  effect: "NoSchedule"

# Tolerate ALL taints (DaemonSet pattern)
- operator: "Exists"

# NoExecute with grace period
- key: "mykey"
  operator: "Equal"
  value: "myvalue"
  effect: "NoExecute"
  tolerationSeconds: 60
```

### Effects Reference

| Effect | Blocks New Pods | Evicts Running Pods | Soft/Hard |
|--------|----------------|---------------------|-----------|
| `NoSchedule` | Yes | No | Hard |
| `PreferNoSchedule` | No (soft) | No | Soft |
| `NoExecute` | Yes | Yes (with timer option) | Hard |

### Operators Reference

| Operator | Key Required | Value Required | Matches |
|----------|-------------|----------------|---------|
| `Equal` | Yes | Yes | Exact key+value+effect |
| `Exists` | Optional | No | Key presence (or all if no key) |

### Well-Known System Taints

| Taint | Added By | Purpose |
|-------|----------|---------|
| `node-role.kubernetes.io/control-plane:NoSchedule` | kubeadm | Protects control plane |
| `node.kubernetes.io/not-ready:NoExecute` | NodeLifecycle controller | Node condition: NotReady |
| `node.kubernetes.io/unreachable:NoExecute` | NodeLifecycle controller | Node condition: Unknown |
| `node.kubernetes.io/memory-pressure:NoSchedule` | kubelet | Node memory pressure |
| `node.kubernetes.io/disk-pressure:NoSchedule` | kubelet | Node disk pressure |
| `node.kubernetes.io/pid-pressure:NoSchedule` | kubelet | Node PID pressure |
| `node.kubernetes.io/unschedulable:NoSchedule` | kubectl cordon | Manual cordon |

### Full Isolation Template

```yaml
# Node preparation (run before deploying)
# kubectl taint nodes <node> <key>=<value>:NoSchedule
# kubectl label nodes <node> <label-key>=<label-value>

apiVersion: apps/v1
kind: Deployment
metadata:
  name: isolated-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: isolated-app
  template:
    metadata:
      labels:
        app: isolated-app
    spec:
      tolerations:
      - key: "<taint-key>"
        operator: "Equal"
        value: "<taint-value>"
        effect: "NoSchedule"
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: "<label-key>"
                operator: In
                values:
                - "<label-value>"
      containers:
      - name: app
        image: <image>
```

---
