# Lab 05 — Resource Limits & Scheduling

## What This Lab Is About

Resources and scheduling are the foundation of cluster stability. Every pod you run competes for CPU and memory on the nodes in your cluster. Get this wrong and you get OOMKilled pods, evicted workloads, a scheduler that can't place anything, or — worst of all — a noisy neighbor that starves every other app on the node.

This lab covers the 5 most critical resource and scheduling failure modes in real clusters. Pods with no limits that consume everything and kill neighbours. The difference between requests and limits and why it matters for scheduling. QoS classes and how they determine eviction order. LimitRange and ResourceQuota enforcement at the namespace level. And node affinity and taints that cause pods to be unschedulable in ways that are hard to read without knowing exactly where to look.

> In prod, resource misconfiguration is silent until it isn't. Then everything goes down at once.

---

## Environment

- **Cluster:** KIND (Kubernetes IN Docker)
- **K8s Version:** v1.27.3
- **Node:** Single node (`kind-control-plane`)
- **Namespace for this lab:** `lab05`

```bash
kubectl create namespace lab05
kubectl config set-context --current --namespace=lab05
```

---

## The 5 Scenarios

| # | Scenario | What You'll Learn |
|---|---|---|
| 01 | No resource limits — noisy neighbour | Why limits are mandatory, node pressure |
| 02 | Requests vs Limits — the scheduling trap | How scheduler uses requests, not limits |
| 03 | QoS classes and eviction order | BestEffort vs Burstable vs Guaranteed |
| 04 | LimitRange and ResourceQuota | Namespace-level enforcement and quota exceeded errors |
| 05 | Taints, Tolerations and Node Affinity | Why pods get stuck Pending with no obvious reason |

---

## Scenario 01 — No Resource Limits — Noisy Neighbour

### What You'll Break

A pod with no resource limits. It starts small, then consumes all available CPU and memory on the node. Other pods on the same node get starved, slowed down, or OOMKilled. This is the most common cause of mysterious performance degradation in shared clusters.

### Apply the Broken State

```bash
# Deploy a critical app with proper limits
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: critical-app
  namespace: lab05
spec:
  replicas: 1
  selector:
    matchLabels:
      app: critical-app
  template:
    metadata:
      labels:
        app: critical-app
    spec:
      containers:
      - name: app
        image: nginx:1.25
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
EOF

# Deploy a noisy neighbour with NO limits
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: noisy-neighbour
  namespace: lab05
spec:
  replicas: 1
  selector:
    matchLabels:
      app: noisy-neighbour
  template:
    metadata:
      labels:
        app: noisy-neighbour
    spec:
      containers:
      - name: app
        image: polinux/stress:latest
        command: ["stress"]
        args: ["--cpu", "4", "--vm", "1", "--vm-bytes", "512M", "--vm-hang", "1"]
        # NO resources block — no requests, no limits
        # This will consume everything it can get
EOF
```

### Symptoms You Will Observe

```bash
# Check what's happening on the node
kubectl top nodes
# NAME                 CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# kind-control-plane   1800m        90%    1900Mi          95%
# Node is under extreme pressure

kubectl top pods -n lab05
# NAME                          CPU(cores)   MEMORY(bytes)
# noisy-neighbour-xxx           1750m        480Mi    <-- consuming everything
# critical-app-xxx              10m          8Mi      <-- starved

# The critical app is barely getting any CPU
# On a real multi-pod node, other pods would be evicted
```

Note: `kubectl top` requires metrics-server. In KIND run:
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# KIND needs this patch for metrics-server to work
kubectl patch deployment metrics-server -n kube-system --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'

# Wait for metrics-server to be ready
kubectl rollout status deployment metrics-server -n kube-system

# Now top works
kubectl top pods -n lab05
```

### Investigate Node Pressure

```bash
# Step 1 — Check node conditions for memory/cpu pressure
kubectl describe node kind-control-plane | grep -A 10 "Conditions:"
# MemoryPressure: True  ← node is under memory pressure
# DiskPressure:   False
# PIDPressure:    False
# Ready:          True (but degraded)

# Step 2 — Check which pods have no limits
kubectl get pods -n lab05 -o json | \
  jq '.items[] | {name: .metadata.name, limits: .spec.containers[].resources.limits}'
# noisy-neighbour shows null limits

# Step 3 — Check node allocatable vs requested
kubectl describe node kind-control-plane | grep -A 8 "Allocated resources"
# Resource           Requests     Limits
# cpu                1850m/2      0/2       <-- limits are 0 (unbounded)
# memory             1700Mi/2Gi   128Mi/2Gi

# Step 4 — Events on the node during pressure
kubectl describe node kind-control-plane | grep -A 5 "Events"
```

### Fix

```bash
# Add resource limits to the noisy neighbour
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: noisy-neighbour
  namespace: lab05
spec:
  replicas: 1
  selector:
    matchLabels:
      app: noisy-neighbour
  template:
    metadata:
      labels:
        app: noisy-neighbour
    spec:
      containers:
      - name: app
        image: polinux/stress:latest
        command: ["stress"]
        args: ["--cpu", "4", "--vm", "1", "--vm-bytes", "512M", "--vm-hang", "1"]
        resources:
          requests:
            memory: "256Mi"
            cpu: "500m"
          limits:
            memory: "512Mi"     # Kernel kills this pod if it exceeds 512Mi
            cpu: "500m"         # CPU throttled at 500m, never starves others
EOF
```

### Prod Wisdom

In prod, **every container must have resource limits — no exceptions.** Without limits, a single misbehaving pod can take down every other workload on the node. The standard practice is to enforce this at the namespace level using `LimitRange` (covered in Scenario 04) so that pods without limits get default limits automatically. No limits is not a "we'll add it later" situation — it is a ticking time bomb.

---

## Scenario 02 — Requests vs Limits — The Scheduling Trap

### What You'll Break

A pod with a low memory request but a very high memory limit. The scheduler places the pod based on the request (low — fits easily), but at runtime the pod expands to use its full limit, causing the node to be overcommitted. Meanwhile, another pod with accurate requests can't be scheduled because the scheduler thinks there's not enough room — even though there physically is.

### Apply the Broken State

```bash
# First — understand the node's allocatable resources
kubectl describe node kind-control-plane | grep -A 6 "Allocatable:"
# cpu:    2
# memory: ~2Gi (varies in KIND)

# Deploy a pod with misleadingly low request but high limit
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: memory-deceiver
  namespace: lab05
spec:
  replicas: 3
  selector:
    matchLabels:
      app: memory-deceiver
  template:
    metadata:
      labels:
        app: memory-deceiver
    spec:
      containers:
      - name: app
        image: polinux/stress:latest
        command: ["stress"]
        args: ["--vm", "1", "--vm-bytes", "200M", "--vm-hang", "1"]
        resources:
          requests:
            memory: "10Mi"      # LIES: claims to need 10Mi
            cpu: "10m"
          limits:
            memory: "256Mi"     # Actually uses up to 256Mi
            cpu: "500m"
EOF
```

```bash
# Scheduler happily places all 3 (3 * 10Mi = 30Mi requested — fits easily)
kubectl get pods -n lab05 -l app=memory-deceiver
# All 3 Running

# But actual usage
kubectl top pods -n lab05 -l app=memory-deceiver
# Each using ~200Mi — total ~600Mi actual vs 30Mi declared to scheduler

# Now try to schedule a pod that honestly declares its needs
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: honest-pod
  namespace: lab05
spec:
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        memory: "500Mi"         # Honest about its needs
        cpu: "100m"
      limits:
        memory: "500Mi"
        cpu: "200m"
EOF
```

```bash
kubectl get pod honest-pod -n lab05
# NAME         READY   STATUS    RESTARTS   AGE
# honest-pod   0/1     Pending   0          30s

kubectl describe pod honest-pod -n lab05 | grep -A 5 "Events"
# Warning  FailedScheduling  default-scheduler
# 0/1 nodes are available: 1 Insufficient memory.

# Scheduler sees:
# Node allocatable: ~2Gi
# Already requested by other pods: ~1.9Gi (including memory-deceiver's 3 * 10Mi + system)
# Honest-pod needs 500Mi — doesn't fit on paper
# Reality: node has plenty of free memory — but scheduler only sees requests
```

### Investigate

```bash
# Step 1 — Check what the scheduler sees (requested vs actual)
kubectl describe node kind-control-plane | grep -A 10 "Allocated resources"
# Resource    Requests      Limits
# cpu         xxx/2000m     xxx/2000m
# memory      xxx/2Gi       xxx/2Gi
# This shows REQUESTED resources — not actual usage

# Step 2 — Compare with actual usage
kubectl top node kind-control-plane
# Shows ACTUAL cpu and memory usage

# Step 3 — Find pods with suspiciously low requests vs high limits
kubectl get pods -n lab05 -o json | jq '.items[] | {
  name: .metadata.name,
  req_mem: .spec.containers[].resources.requests.memory,
  lim_mem: .spec.containers[].resources.limits.memory
}'
```

### Fix

```bash
# Delete the deceptive pods
kubectl delete deployment memory-deceiver -n lab05
kubectl delete pod honest-pod -n lab05

# Redeploy with accurate requests
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: memory-honest
  namespace: lab05
spec:
  replicas: 2
  selector:
    matchLabels:
      app: memory-honest
  template:
    metadata:
      labels:
        app: memory-honest
    spec:
      containers:
      - name: app
        image: nginx:1.25
        resources:
          requests:
            memory: "64Mi"      # Accurate — what the app actually needs at rest
            cpu: "100m"
          limits:
            memory: "128Mi"     # Headroom — max it can use under peak load
            cpu: "200m"
EOF
```

### The Request/Limit Mental Model

```
REQUESTS — what the scheduler uses
  → Scheduler sums all requests on a node
  → Pod only placed if (node allocatable - sum of requests) >= pod's request
  → Set to: typical usage under normal load
  → Too low  = overcommitted node, OOMKill storm under pressure
  → Too high = pod can't schedule even when physical space exists

LIMITS — what the kernel enforces at runtime
  → CPU: throttled when exceeded (not killed)
  → Memory: killed (OOMKilled) when exceeded
  → Set to: maximum acceptable usage under peak load
  → Too low  = OOMKilled during traffic spikes
  → Too high = noisy neighbour risk, node instability

GOLDEN RULE:
  requests = what you need (baseline)
  limits   = what you allow (ceiling)
  Never set requests >> actual usage
  Never set limits << actual peak usage
```

---

## Scenario 03 — QoS Classes and Eviction Order

### What You'll Break

When a node runs out of memory, Kubernetes evicts pods to reclaim resources. The order of eviction is determined by QoS class — which is automatically assigned based on how you configure requests and limits. Understanding this determines which of your pods survive a node pressure event.

### Understanding QoS Classes

```
Guaranteed (highest priority — last to be evicted)
  Condition: requests == limits for ALL containers, both CPU and memory
  Example:
    resources:
      requests:
        memory: "128Mi"
        cpu: "500m"
      limits:
        memory: "128Mi"   # same as request
        cpu: "500m"       # same as request

Burstable (middle priority)
  Condition: at least one container has a request or limit set
             but requests != limits
  Example:
    resources:
      requests:
        memory: "64Mi"
      limits:
        memory: "128Mi"   # different from request = Burstable

BestEffort (lowest priority — first to be evicted)
  Condition: NO resources set at all — no requests, no limits
  Example:
    containers:
    - name: app
      image: nginx:1.25
      # no resources block at all
```

### Apply All Three QoS Classes

```bash
# BestEffort — first to be evicted
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: besteffort-pod
  namespace: lab05
spec:
  containers:
  - name: app
    image: nginx:1.25
    # No resources — BestEffort
EOF

# Burstable — evicted second
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: burstable-pod
  namespace: lab05
spec:
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        memory: "64Mi"
        cpu: "100m"
      limits:
        memory: "128Mi"   # limit != request = Burstable
        cpu: "200m"
EOF

# Guaranteed — last to be evicted
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: guaranteed-pod
  namespace: lab05
spec:
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        memory: "64Mi"
        cpu: "100m"
      limits:
        memory: "64Mi"    # limit == request = Guaranteed
        cpu: "100m"
EOF
```

### Verify QoS Classes

```bash
# Check QoS class assigned to each pod
kubectl get pod besteffort-pod -n lab05 \
  -o jsonpath='{.status.qosClass}'
# BestEffort

kubectl get pod burstable-pod -n lab05 \
  -o jsonpath='{.status.qosClass}'
# Burstable

kubectl get pod guaranteed-pod -n lab05 \
  -o jsonpath='{.status.qosClass}'
# Guaranteed

# Check all at once
kubectl get pods -n lab05 \
  -o custom-columns="NAME:.metadata.name,QOS:.status.qosClass"
# NAME              QOS
# besteffort-pod    BestEffort
# burstable-pod     Burstable
# guaranteed-pod    Guaranteed
```

### Simulate Eviction Order Understanding

```bash
# When node memory pressure hits:
# 1. BestEffort pods evicted first (no resource definition at all)
# 2. Burstable pods evicted next (sorted by how much they exceed their request)
# 3. Guaranteed pods evicted last (only if node is critically out of memory)

# Check node eviction thresholds
kubectl describe node kind-control-plane | grep -A 5 "eviction"

# Check if any pods have been evicted
kubectl get pods -n lab05 --field-selector=status.phase=Failed
# Evicted pods show as Failed with reason Evicted

# See eviction reason
kubectl describe pod <evicted-pod-name> -n lab05
# Message: The node was low on resource: memory.
# Threshold quantity: 100Mi, available: 80Mi
```

### Prod Wisdom

For any prod workload that cannot tolerate being evicted — **set requests equal to limits (`Guaranteed` QoS)**. The cost is less efficient bin packing, but the benefit is eviction immunity under node pressure. For background jobs, batch processors, and non-critical workloads — `Burstable` is fine. `BestEffort` should never be used for anything important in prod — it is the first casualty when the node sneezes.

---

## Scenario 04 — LimitRange and ResourceQuota

### What You'll Break

A namespace with no guardrails. Any pod can request unlimited resources, any team can deploy without limits, and the cluster gets exhausted. Then the fix — `LimitRange` to auto-inject defaults and `ResourceQuota` to cap total namespace consumption.

### Break It — No Guardrails

```bash
# Create a fresh namespace for this scenario
kubectl create namespace lab05-quota

# Deploy without any resource spec — accepted without complaint
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: unlimited-app
  namespace: lab05-quota
spec:
  replicas: 10
  selector:
    matchLabels:
      app: unlimited-app
  template:
    metadata:
      labels:
        app: unlimited-app
    spec:
      containers:
      - name: app
        image: nginx:1.25
        # No resources — accepted by default, QoS: BestEffort
EOF

kubectl get pods -n lab05-quota
# 10 pods, all Running, all BestEffort
# These will be the first evicted under any pressure
```

### Fix Part A — LimitRange (Default Limits)

`LimitRange` automatically injects default requests and limits into pods that don't specify them. It also enforces min/max bounds.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: lab05-quota
spec:
  limits:
  - type: Container
    default:                   # Applied when no limits specified
      memory: "128Mi"
      cpu: "200m"
    defaultRequest:            # Applied when no requests specified
      memory: "64Mi"
      cpu: "100m"
    max:                       # No container can exceed this
      memory: "512Mi"
      cpu: "1"
    min:                       # No container can go below this
      memory: "32Mi"
      cpu: "50m"
EOF

# New pods after this LimitRange will get default limits injected
# Existing pods are NOT affected — only new pods

# Delete and recreate to see defaults applied
kubectl delete deployment unlimited-app -n lab05-quota

cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: limited-app
  namespace: lab05-quota
spec:
  replicas: 2
  selector:
    matchLabels:
      app: limited-app
  template:
    metadata:
      labels:
        app: limited-app
    spec:
      containers:
      - name: app
        image: nginx:1.25
        # No resources specified — LimitRange injects defaults
EOF

# Check that defaults were injected
kubectl describe pod $(kubectl get pod -n lab05-quota -l app=limited-app \
  -o jsonpath='{.items[0].metadata.name}') -n lab05-quota | grep -A 6 "Limits\|Requests"
# Limits:
#   cpu:     200m     <-- injected by LimitRange
#   memory:  128Mi   <-- injected by LimitRange
# Requests:
#   cpu:     100m     <-- injected by LimitRange
#   memory:  64Mi    <-- injected by LimitRange

# Try to exceed the max
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: too-big
  namespace: lab05-quota
spec:
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        memory: "1Gi"          # Exceeds LimitRange max of 512Mi
EOF
# Error from server (Forbidden): pods "too-big" is forbidden:
# maximum memory usage per Container is 512Mi, but limit is 1073741824
```

### Fix Part B — ResourceQuota (Total Namespace Cap)

`ResourceQuota` caps the total resources consumed across all pods in a namespace.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: namespace-quota
  namespace: lab05-quota
spec:
  hard:
    requests.cpu: "1"          # Total CPU requests across all pods <= 1 core
    requests.memory: "512Mi"   # Total memory requests <= 512Mi
    limits.cpu: "2"            # Total CPU limits <= 2 cores
    limits.memory: "1Gi"       # Total memory limits <= 1Gi
    pods: "10"                 # Max 10 pods in this namespace
    services: "5"              # Max 5 services
EOF

# Check quota usage
kubectl get resourcequota -n lab05-quota
kubectl describe resourcequota namespace-quota -n lab05-quota
# Resource          Used    Hard
# limits.cpu        400m    2
# limits.memory     256Mi   1Gi
# pods              2       10
# requests.cpu      200m    1
# requests.memory   128Mi   512Mi
# services          0       5

# Try to exceed the quota
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: quota-buster
  namespace: lab05-quota
spec:
  replicas: 5
  selector:
    matchLabels:
      app: quota-buster
  template:
    metadata:
      labels:
        app: quota-buster
    spec:
      containers:
      - name: app
        image: nginx:1.25
        resources:
          requests:
            memory: "200Mi"    # 5 * 200Mi = 1000Mi > 512Mi quota
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
EOF

kubectl get pods -n lab05-quota
# Only some pods created — others blocked by quota

kubectl get replicaset -n lab05-quota
kubectl describe replicaset -n lab05-quota | grep -A 5 "Warning"
# Warning  FailedCreate  replicaset-controller
# Error creating: pods is forbidden: exceeded quota: namespace-quota,
# requested: requests.memory=200Mi, used: requests.memory=384Mi,
# limited: requests.memory=512Mi
```

### Investigate Quota Issues

```bash
# See current quota consumption
kubectl describe resourcequota namespace-quota -n lab05-quota

# See LimitRange in effect
kubectl describe limitrange default-limits -n lab05-quota

# Find pods that were blocked
kubectl get events -n lab05-quota --field-selector reason=FailedCreate

# Check if quota is why a deployment has fewer pods than desired
kubectl describe deployment quota-buster -n lab05-quota
# Conditions: ReplicaFailure — quota exceeded
```

### Cleanup Quota Namespace

```bash
kubectl delete namespace lab05-quota
```

### Prod Wisdom

In any multi-team cluster, **every namespace should have both a LimitRange and a ResourceQuota**. LimitRange protects individual nodes from unbounded containers. ResourceQuota protects the cluster from runaway namespaces. Without them, one team's bad deployment can exhaust cluster resources for everyone. Set `LimitRange` defaults conservatively — low enough to protect the cluster, high enough that apps can run without always needing explicit overrides.

---

## Scenario 05 — Taints, Tolerations and Node Affinity

### What You'll Break

Pods stuck in `Pending` because of node taints they don't tolerate, and pods that won't schedule because of node affinity rules that don't match any node. These are two of the most confusing scheduling failures because the error messages are not immediately obvious.

### Break It — Taint With No Toleration

```bash
# Add a taint to the KIND node
kubectl taint node kind-control-plane dedicated=gpu:NoSchedule

# Now deploy a pod without a toleration
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: no-toleration-pod
  namespace: lab05
spec:
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        memory: "64Mi"
        cpu: "100m"
      limits:
        memory: "128Mi"
        cpu: "200m"
EOF
```

```bash
kubectl get pod no-toleration-pod -n lab05
# NAME                  READY   STATUS    RESTARTS   AGE
# no-toleration-pod     0/1     Pending   0          30s

kubectl describe pod no-toleration-pod -n lab05 | grep -A 5 "Events"
# Warning  FailedScheduling  default-scheduler
# 0/1 nodes are available: 1 node(s) had untolerated taint
# {dedicated: gpu}. preemption: 0/1 nodes are available:
# 1 Preemption is not helpful for scheduling.
```

### Fix — Add Toleration

```bash
kubectl delete pod no-toleration-pod -n lab05

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: tolerating-pod
  namespace: lab05
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"       # Tolerate this specific taint
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        memory: "64Mi"
        cpu: "100m"
      limits:
        memory: "128Mi"
        cpu: "200m"
EOF

kubectl get pod tolerating-pod -n lab05
# NAME               READY   STATUS    RESTARTS   AGE
# tolerating-pod     1/1     Running   0          5s
```

### Remove the Taint After Testing

```bash
kubectl taint node kind-control-plane dedicated=gpu:NoSchedule-
# The trailing "-" removes the taint
```

### Break It — Node Affinity With No Matching Node

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: bad-affinity-pod
  namespace: lab05
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: node-type
            operator: In
            values:
            - high-memory       # No node has this label
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        memory: "64Mi"
        cpu: "100m"
      limits:
        memory: "128Mi"
        cpu: "200m"
EOF
```

```bash
kubectl get pod bad-affinity-pod -n lab05
# NAME               READY   STATUS    RESTARTS   AGE
# bad-affinity-pod   0/1     Pending   0          20s

kubectl describe pod bad-affinity-pod -n lab05 | grep -A 5 "Events"
# Warning  FailedScheduling  default-scheduler
# 0/1 nodes are available: 1 node(s) didn't match Pod's node affinity/selector.
```

### Fix — Label the Node or Relax the Affinity

```bash
kubectl delete pod bad-affinity-pod -n lab05

# Option A — Label the node to match the affinity rule
kubectl label node kind-control-plane node-type=high-memory

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: fixed-affinity-pod
  namespace: lab05
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: node-type
            operator: In
            values:
            - high-memory       # Now this label exists on the node
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        memory: "64Mi"
        cpu: "100m"
      limits:
        memory: "128Mi"
        cpu: "200m"
EOF

kubectl get pod fixed-affinity-pod -n lab05
# Running

# Option B — Use preferredDuringScheduling instead of required
# (soft preference — schedules anywhere if no match found)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: preferred-affinity-pod
  namespace: lab05
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: node-type
            operator: In
            values:
            - high-memory       # Preferred but not required — schedules anywhere
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        memory: "64Mi"
        cpu: "100m"
      limits:
        memory: "128Mi"
        cpu: "200m"
EOF

kubectl get pod preferred-affinity-pod -n lab05
# Running — even without the label, because it's preferred not required
```

### Taints vs Affinity — The Mental Model

```
Taints and Tolerations — NODE pushes pods away
  Taint on node: "I don't want most pods here"
  Toleration on pod: "I can handle this node's taint"

  Effects:
    NoSchedule    → New pods without toleration won't be scheduled here
    PreferNoSchedule → Scheduler tries to avoid this node, but will use it if needed
    NoExecute     → Existing pods without toleration are EVICTED immediately

Node Affinity — POD pulls toward nodes
  Required (hard): pod MUST land on a matching node — Pending if no match
  Preferred (soft): pod prefers matching node — schedules anywhere if no match

Combined pattern in prod:
  Taint a node for dedicated workloads (GPU, high-memory, spot)
  Add toleration to pods that should run there
  Add affinity to pods that MUST run there
  This ensures only the right pods reach the right nodes
```

### Diagnose Any Pending Pod — The Complete Scheduler Debug Flow

```bash
# Step 1 — Confirm it's Pending
kubectl get pod <name> -n <namespace>

# Step 2 — Read the scheduler message
kubectl describe pod <name> -n <namespace> | grep -A 10 "Events"

# Decode the message:
# "Insufficient cpu/memory"       → resource request too high
# "node(s) had untolerated taint" → add toleration to pod
# "node(s) didn't match affinity" → fix affinity or label the node
# "0/N nodes available"           → N = total nodes, message explains why each failed

# Step 3 — Check node labels (for affinity issues)
kubectl get nodes --show-labels

# Step 4 — Check node taints (for taint issues)
kubectl describe node <node-name> | grep -A 5 "Taints"

# Step 5 — Check node allocatable vs requested (for resource issues)
kubectl describe node <node-name> | grep -A 10 "Allocated resources"
```

### Prod Wisdom

Taints are how platform teams reserve nodes for specific workloads — GPU nodes, spot instances, dedicated infra nodes. In any real cluster you will encounter taints on system nodes (`node-role.kubernetes.io/control-plane:NoSchedule`) and purpose-built node pools. The single most common taint issue is system-level taints on control plane nodes — your pods refuse to schedule there for good reason and the error message tells you exactly why. Always read the full scheduler failure message: it tells you per-node why each node was rejected.

---

## Key Commands Reference — Lab 05

```bash
# Check node resource pressure
kubectl describe node <node-name> | grep -A 10 "Conditions:"
kubectl describe node <node-name> | grep -A 10 "Allocated resources"

# Real-time resource usage (requires metrics-server)
kubectl top nodes
kubectl top pods -n <namespace>

# Check QoS class
kubectl get pod <name> -n <namespace> -o jsonpath='{.status.qosClass}'

# Check all pods QoS at once
kubectl get pods -n <namespace> \
  -o custom-columns="NAME:.metadata.name,QOS:.status.qosClass"

# LimitRange and ResourceQuota
kubectl get limitrange -n <namespace>
kubectl get resourcequota -n <namespace>
kubectl describe resourcequota <name> -n <namespace>

# Node taints
kubectl describe node <name> | grep Taints
kubectl taint node <name> key=value:effect         # Add taint
kubectl taint node <name> key=value:effect-        # Remove taint (trailing -)

# Node labels (for affinity)
kubectl get nodes --show-labels
kubectl label node <name> key=value                # Add label
kubectl label node <name> key-                     # Remove label (trailing -)

# Scheduler failure diagnosis
kubectl describe pod <name> -n <namespace> | grep -A 10 "Events"
```

---

## Prod Wisdom — The Senior Engineer Mindset

Three things that separate senior engineers in resource management:

**1. They treat requests as a contract with the scheduler.** Requests must reflect actual baseline usage — not zero, not wildly inflated. Inaccurate requests cause either overcommitted nodes (OOMKill storms under pressure) or chronic Pending pods (scheduler thinks there's no room). Profile your app in staging, set requests to P50 usage, set limits to P99 usage.

**2. They enforce LimitRange and ResourceQuota on every namespace from day one.** These are not optional in a shared cluster. Without them, one bad deployment can destabilize the entire cluster. LimitRange as the per-pod guardrail. ResourceQuota as the per-namespace guardrail. Both. Always.

**3. They read the full scheduler failure message.** The message `0/3 nodes available: 2 Insufficient memory, 1 node(s) had untolerated taint` tells you everything — 2 nodes rejected for memory, 1 for taint. Most engineers stop at `Insufficient memory` and miss the taint. Read every line of the scheduler message, it tells you exactly what each node rejected and why.

The resource scheduling debug order:
```
describe pod → Events → scheduler message → node labels/taints → node allocated resources
```

---

## Cleanup

```bash
# Remove test labels and taints from node
kubectl taint node kind-control-plane dedicated=gpu:NoSchedule- 2>/dev/null || true
kubectl label node kind-control-plane node-type- 2>/dev/null || true

# Delete namespace
kubectl delete namespace lab05
```

---

*Lab 05 complete. Move to Lab 06 — RBAC when ready.*