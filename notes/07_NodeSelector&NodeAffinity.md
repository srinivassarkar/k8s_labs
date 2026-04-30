# Kubernetes Node Affinity & NodeSelector

> **Scope:** Controlling pod placement through node labels — from nodeSelector primitives to weighted affinity scoring, operators, OR logic, and production isolation patterns.

---

## 0. First Principles

> The mental model that never changes, regardless of Kubernetes version.

**The core invariant:**

> Placement mechanisms answer one question: *which nodes is this pod allowed to land on?* They differ only in how strictly and expressively they answer it.

`nodeSelector`, `requiredAffinity`, and `preferredAffinity` are all label-matching mechanisms on nodes. They differ in expressiveness, flexibility, and enforcement. None of them *pull* a pod to a node — they *filter* the candidate set the scheduler may use.

**The three axioms:**

| Axiom | Statement |
|-------|-----------|
| **Labels are the substrate** | Every placement rule ultimately resolves to: does this node's label set satisfy this expression? No labels, no placement. |
| **None of this is retroactive** | All three mechanisms use `IgnoredDuringExecution`. A pod already running on a node is never re-evaluated if labels change. Only `NoExecute` taints cause eviction. |
| **Affinity is additive, not prescriptive** | Affinity narrows the candidate set. Resource availability, taints, topology, and other plugins still apply within that set. |

**The mental hierarchy:**

```
nodeSelector          → "only nodes with this exact label"     (blunt instrument)
requiredAffinity      → "only nodes matching this expression"  (expressive, still hard)
preferredAffinity     → "prefer nodes matching this, weighted" (soft guidance to scorer)

All three: label-based, scheduling-time only, not retroactive.
Escalating expressiveness, same underlying mechanism.
```

**The AND / OR model — get this exactly right:**

```
nodeSelector:                      → all keys = AND, no OR possible

requiredAffinity:
  nodeSelectorTerms:               → multiple terms = OR (between terms)
    - matchExpressions:            → multiple expressions = AND (within a term)
      - key: zone, In: [east]
      - key: env, In: [prod]       → (zone=east AND env=prod)
    - matchExpressions:
      - key: tier, In: [frontend]  → OR (tier=frontend)

Result: (zone=east AND env=prod) OR (tier=frontend)
```

---

## 1. Reality Constraints

> What Kubernetes actually does and does not do — the gaps that cause production incidents.

### What Kubernetes Does

- Evaluates `nodeSelector` and `requiredAffinity` during the **Filter phase** of scheduling. Nodes that fail are eliminated from the candidate set.
- Evaluates `preferredAffinity` during the **Score phase**. Nodes that match accumulate weight; the highest-scoring node wins (subject to all other scoring plugins).
- Re-evaluates `requiredAffinity` on **newly created pods only** — `IgnoredDuringExecution` means existing pods are never re-checked.
- Allows combining `nodeSelector` and `nodeAffinity` on the same pod — **both must be satisfied simultaneously**.

### What Kubernetes Does NOT Do

| Assumption | Reality |
|------------|---------|
| "Removing a node label evicts pods on that node" | No. `IgnoredDuringExecution` means running pods are never re-evaluated. The pod keeps running even if the node no longer satisfies the rule. |
| "preferredAffinity guarantees placement on the preferred node" | No. It adds weight to the score. If the preferred node is resource-constrained, a lower-scored node wins. Weight is one input among many. |
| "Multiple `matchExpressions` within one term are OR logic" | No. Multiple expressions within a single `nodeSelectorTerms` entry are AND. OR only exists between separate `nodeSelectorTerms` entries. |
| "nodeSelector and nodeAffinity are alternatives — use one" | If both are set, both must be satisfied. They are additive, not substitutes. |
| "Gt/Lt operators work with any label value" | No. The label value must be a parseable integer string. A non-numeric value causes the expression to fail silently, leaving the pod Pending. |
| "preferredAffinity alone prevents pods from landing on unpreferred nodes" | No. Without a `requiredAffinity` or taint, pods can land anywhere schedulable. Preferred only biases, it does not block. |
| "nodeSelectorTerms with no matching nodes leaves pods on current nodes" | No. New pods from the same Deployment will go Pending. Running pods are unaffected (IgnoredDuringExecution). |

### The IgnoredDuringExecution Guarantee (and its consequence)

```
Node label change scenario:
  1. Pod scheduled onto worker1 because label env=prod existed
  2. Admin removes label: kubectl label nodes worker1 env-
  3. Pod continues running — no eviction, no warning
  4. New pods from the same Deployment → Pending (no matching node)

This is NOT a bug. It is intentional. RequiredDuringSchedulingRequiredDuringExecution
(which would evict) exists in the API spec but is NOT yet implemented in any
Kubernetes release as of 1.30.
```

### The Weight Scoring Reality

```
preferredDuringSchedulingIgnoredDuringExecution weight range: 1–100

Final node score = sum of:
  - Affinity weight matches
  - LeastRequestedPriority (CPU/memory)
  - BalancedResourceAllocation
  - ImageLocalityPriority
  - TaintToleration scorer
  - InterPodAffinity scorer
  + others

A weight: 100 preference can be overridden by a heavily-loaded preferred node
vs. an empty unpreferred node. Resource scoring frequently wins.
```

---

## 2. Decision Logic

> When to use what — hard rules, not opinions.

### Choosing the Right Mechanism

```
Do you need pods on specific nodes?
│
├── Is the requirement hard (pod MUST NOT go elsewhere)?
│   ├── Simple key=value match, no OR, no operators needed?
│   │   └── nodeSelector  (least expressive, least config)
│   └── Need OR logic, operators (In/NotIn/Gt/Lt/Exists), or multiple conditions?
│       └── requiredAffinity (nodeSelectorTerms + matchExpressions)
│
└── Is the requirement soft (prefer certain nodes but allow fallback)?
    └── preferredAffinity with weight
        └── Need to also hard-block certain nodes?
            └── requiredAffinity (filter) + preferredAffinity (score) together
```

### Choosing the Right Operator

| Operator | Use When | Label Requirement | Returns True When |
|----------|----------|-------------------|-------------------|
| `In` | Node must have key with one of the listed values | Must exist | Key exists AND value ∈ values list |
| `NotIn` | Exclude nodes with specific values | May or may not exist | Key absent OR value ∉ values list |
| `Exists` | Node must have the key (any value) | Must exist | Key is present (value irrelevant) |
| `DoesNotExist` | Node must NOT have the key | Must be absent | Key is absent |
| `Gt` | Node's label value must exceed threshold | Must be numeric string | Integer(label) > Integer(value) |
| `Lt` | Node's label value must be below threshold | Must be numeric string | Integer(label) < Integer(value) |

### AND vs. OR Construction Rules

```yaml
# AND within a term — pod requires BOTH conditions simultaneously
nodeSelectorTerms:
- matchExpressions:
  - key: zone         # Condition A
    operator: In
    values: [east]
  - key: env          # Condition B (AND with A)
    operator: In
    values: [prod]
# Result: zone=east AND env=prod

---

# OR between terms — pod accepts EITHER condition
nodeSelectorTerms:
- matchExpressions:
  - key: zone
    operator: In
    values: [east]    # Term 1
- matchExpressions:
  - key: env
    operator: In
    values: [dev]     # Term 2 (OR with Term 1)
# Result: zone=east OR env=dev

---

# OR within a single expression's values list
- key: storage
  operator: In
  values:
  - ssd
  - hdd             # storage=ssd OR storage=hdd (values list = OR)
```

### Combining Required + Preferred (The Production Pattern)

```
Use Case: Must be on storage nodes (hard), prefer SSD over HDD (soft)

required  → defines the candidate pool (filter)
preferred → biases placement within that pool (score)

Never use preferred alone for isolation.
Never use required + preferred on conflicting conditions (defeats the point).
```

---

## 3. Internal Working

> How it actually happens under the hood — step by step.

### The Scheduling Pipeline

```
Pod created (Pending) → kube-scheduler watch loop picks it up
              │
              ▼
┌─────────────────────────────────────────────────────┐
│ FILTER PHASE — eliminates ineligible nodes          │
│                                                     │
│  Plugin: NodeSelector                               │
│  → For each node: does label set contain all        │
│    key=value pairs in pod.spec.nodeSelector?        │
│    No → eliminate node                              │
│                                                     │
│  Plugin: NodeAffinity                               │
│  → Evaluate requiredDuringSchedulingI...            │
│    For each nodeSelectorTerms entry (OR):           │
│      Evaluate all matchExpressions (AND):           │
│        Does node satisfy every expression?          │
│    At least ONE term must be fully satisfied.       │
│    No term satisfied → eliminate node               │
│                                                     │
│  Other filter plugins run in parallel:              │
│  TaintToleration, NodeResourcesFit,                 │
│  VolumeBinding, NodePorts, etc.                     │
│                                                     │
│  Result: filtered node list                         │
└─────────────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────┐
│ SCORE PHASE — ranks remaining nodes                 │
│                                                     │
│  Plugin: NodeAffinity scorer                        │
│  → For each preferredDuringScheduling term:         │
│    Does node match the preference?                  │
│    Yes → add weight to node's score                 │
│    No  → add 0                                      │
│                                                     │
│  Other score plugins run in parallel:               │
│  LeastAllocated, BalancedAllocation,                │
│  ImageLocality, TaintToleration, etc.               │
│                                                     │
│  All plugin scores normalized → weighted sum        │
│  Result: scored node list                           │
└─────────────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────┐
│ BIND PHASE                                          │
│  Scheduler writes pod.spec.nodeName                 │
│  kubelet on that node picks up the pod              │
└─────────────────────────────────────────────────────┘
```

### How NodeAffinity Scoring Works Internally

```
Node scores from NodeAffinity plugin are normalized to [0, MaxNodeScore].
The raw affinity score = sum of weights of all matching preferred terms.

Example:
  Node has: storage=ssd, zone=east

  preferredTerms:
  - weight: 80, preference: storage=ssd   → matches → +80
  - weight: 20, preference: zone=east     → matches → +20
  - weight: 50, preference: env=prod      → no match → +0

  Raw affinity score = 100

  This is then normalized alongside scores from all other plugins.
  Final placement is the highest aggregate score.
```

### Label Evaluation at the API Server

```
kubectl label nodes worker1 storage=ssd
→ Writes to node.metadata.labels in etcd

Scheduler reads from its local informer cache (not etcd directly).
Cache is updated via watch events from API server.
Label propagation to scheduler: near-instantaneous (milliseconds) in healthy clusters.
```

### What IgnoredDuringExecution Actually Means in Code

```
The affinity rule field name encodes its semantics:
  requiredDuringScheduling  → enforced at scheduling time
  IgnoredDuringExecution    → NOT re-evaluated while pod is running

The "RequiredDuringExecution" variant exists as an API field
but kubelet/scheduler do not implement it yet (as of k8s 1.30).
When implemented, it would watch node label changes and trigger eviction.
```

---

## 4. Hands-On

> Production-quality YAML and commands. Nothing simplified.

### 4.1 Node Label Management

```bash
# Apply labels
kubectl label nodes worker1 storage=ssd env=prod zone=east
kubectl label nodes worker2 storage=hdd env=dev zone=west

# Overwrite existing label
kubectl label nodes worker1 storage=nvme --overwrite

# Remove label (trailing dash)
kubectl label nodes worker1 env-

# Verify
kubectl get nodes --show-labels
kubectl get nodes -o=jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.labels}{"\n"}{end}'

# Filter nodes by label
kubectl get nodes -l storage=ssd
kubectl get nodes -l 'env in (prod,staging)'
```

### 4.2 nodeSelector (Simple Hard Placement)

```yaml
# deploy-nodeselector.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ssd-workload
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ssd-workload
  template:
    metadata:
      labels:
        app: ssd-workload
    spec:
      nodeSelector:
        storage: ssd      # AND
        env: prod         # AND — both must match
      containers:
      - name: app
        image: nginx:1.25
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "200m"
            memory: "256Mi"
```

### 4.3 Required Node Affinity — Single Term (AND)

```yaml
# deploy-required-and.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: required-and
spec:
  replicas: 3
  selector:
    matchLabels:
      app: required-and
  template:
    metadata:
      labels:
        app: required-and
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: storage        # Condition A
                operator: In
                values:
                - ssd
              - key: env            # Condition B — AND with A
                operator: In
                values:
                - prod
      containers:
      - name: app
        image: nginx:1.25
```

### 4.4 Required Node Affinity — Multiple Terms (OR)

```yaml
# deploy-required-or.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: required-or
spec:
  replicas: 4
  selector:
    matchLabels:
      app: required-or
  template:
    metadata:
      labels:
        app: required-or
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:         # Term 1
              - key: zone
                operator: In
                values:
                - east
            - matchExpressions:         # Term 2 — OR with Term 1
              - key: env
                operator: In
                values:
                - dev
      containers:
      - name: app
        image: nginx:1.25
```

### 4.5 Numeric Operators: Gt and Lt

```bash
# Label nodes with numeric values
kubectl label nodes worker1 cpu-count=8
kubectl label nodes worker2 cpu-count=2
```

```yaml
# pod-gt.yaml — only nodes with cpu-count > 4
apiVersion: v1
kind: Pod
metadata:
  name: high-cpu-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: cpu-count
            operator: Gt
            values:
            - "4"           # Must be a string — scheduler parses to int
  containers:
  - name: app
    image: nginx:1.25
```

```yaml
# pod-lt.yaml — only nodes with cpu-count < 4
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: cpu-count
            operator: Lt
            values:
            - "4"
```

### 4.6 Anti-Placement: NotIn and DoesNotExist

```yaml
# Avoid dev nodes entirely
- key: env
  operator: NotIn
  values:
  - dev
  - staging

# Avoid nodes with a dedicated key (any value)
- key: dedicated
  operator: DoesNotExist
```

### 4.7 Required + Preferred Combined (Production Standard)

```yaml
# deploy-combined.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prod-app
  namespace: production
spec:
  replicas: 6
  selector:
    matchLabels:
      app: prod-app
  template:
    metadata:
      labels:
        app: prod-app
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: storage          # Hard requirement: must be storage node
                operator: In
                values:
                - ssd
                - hdd
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 80
            preference:
              matchExpressions:
              - key: storage          # Strong preference for SSD
                operator: In
                values:
                - ssd
          - weight: 20
            preference:
              matchExpressions:
              - key: zone             # Weaker preference for east zone
                operator: In
                values:
                - east
      containers:
      - name: app
        image: nginx:1.25
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
```

---

## 5. Production Flow

> Real-world architecture and design patterns.

### Pattern 1: Tiered Storage Placement

```
Scenario: Databases on NVMe SSD, caches on SSD, batch jobs on HDD.

Node labels:
  storage-tier=nvme   → database nodes
  storage-tier=ssd    → cache nodes
  storage-tier=hdd    → batch nodes

Database Deployment:
  required: storage-tier In [nvme]
  + taint: storage-tier=nvme:NoSchedule (prevent non-DB workloads)

Cache Deployment:
  required: storage-tier In [ssd]
  preferred: zone In [east] (weight: 60) for latency

Batch Deployment:
  required: storage-tier In [hdd]
  OR logic: storage-tier In [hdd, ssd] (spill to SSD if HDD full)
```

### Pattern 2: Multi-Zone High Availability

```yaml
# Require pod to be in either zone, prefer primary
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: topology.kubernetes.io/zone
          operator: In
          values:
          - us-east-1a
          - us-east-1b
          - us-east-1c
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      preference:
        matchExpressions:
        - key: topology.kubernetes.io/zone
          operator: In
          values:
          - us-east-1a    # Primary zone preferred
```

### Pattern 3: Environment Isolation (Strict)

```
Architecture:
  Prod nodes:    taint env=prod:NoSchedule + label env=prod
  Staging nodes: taint env=staging:NoSchedule + label env=staging
  Dev nodes:     taint env=dev:NoSchedule + label env=dev

Each Deployment carries:
  1. Toleration for its environment's taint
  2. requiredAffinity for its environment's label

Result:
  - No cross-environment drift (even under surge/rescheduling)
  - Toleration without affinity = taint block lifted but pod can still drift
  - Affinity without taint = preferred pool but other pods can enter
  - Both together = true isolation
```

### Pattern 4: Capacity Tier Routing with Gt

```
Scenario: Memory-intensive ML inference requires nodes with high RAM.
Label all nodes with their memory tier:
  kubectl label nodes <node> mem-gb=128
  kubectl label nodes <node> mem-gb=32

Inference Deployment:
  required:
  - key: mem-gb
    operator: Gt
    values: ["64"]    # Only 128GB+ nodes
```

### Pattern 5: Spot/Preemptible Node Preference

```yaml
# Prefer cheaper spot nodes, fall back to on-demand
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/os
          operator: In
          values: [linux]
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 80
      preference:
        matchExpressions:
        - key: node.kubernetes.io/capacity-type
          operator: In
          values: [spot]
    - weight: 20
      preference:
        matchExpressions:
        - key: node.kubernetes.io/capacity-type
          operator: In
          values: [on-demand]
```

### Pattern 6: Combining nodeSelector + nodeAffinity (Legacy Migration)

```
When migrating from nodeSelector to nodeAffinity gradually:
  Both fields can coexist — the pod must satisfy BOTH.
  This is useful for a phased rollout where some pods still use nodeSelector.

Do NOT combine them on the same pod if you intend them as alternatives.
The scheduler enforces AND between nodeSelector and nodeAffinity.
```

---

## 6. Mistakes

> What actually breaks in real systems — root cause and fix.

### Mistake 1: OR Logic Inside matchExpressions (Most Common Interview Trap)

**Symptom:** Pod is Pending despite some nodes matching "one of the conditions."

**Root cause:** Developer put two expressions in one `matchExpressions` block expecting OR. They get AND.

```yaml
# WRONG — this is (zone=east AND zone=west) — impossible
- matchExpressions:
  - key: zone
    operator: In
    values: [east]
  - key: zone
    operator: In
    values: [west]

# CORRECT — separate terms for OR
nodeSelectorTerms:
- matchExpressions:
  - key: zone
    operator: In
    values: [east]
- matchExpressions:
  - key: zone
    operator: In
    values: [west]
```

---

### Mistake 2: Non-Numeric Label Value with Gt/Lt

**Symptom:** Pod is Pending. Node has the label. No obvious error in `describe`.

**Root cause:** Label value is not a parseable integer. E.g., `cpu=8-core` instead of `cpu=8`.

**Diagnosis:**
```bash
kubectl describe pod <pod> | grep -A5 "Events"
# Look for: "Invalid value" or "failed to parse" in events
kubectl get node <node> --show-labels | grep cpu
```

**Fix:** Relabel with a clean integer string.
```bash
kubectl label nodes worker1 cpu-count=8 --overwrite
```

---

### Mistake 3: nodeSelector and nodeAffinity Both Set — Silent AND

**Symptom:** Pod Pending even though nodeAffinity expression matches available nodes.

**Root cause:** nodeSelector is also present and points to a label that doesn't exist, silently eliminating all nodes.

**Diagnosis:**
```bash
kubectl get pod <pod> -o yaml | grep -A5 nodeSelector
kubectl get pod <pod> -o yaml | grep -A20 affinity
```

**Fix:** Remove `nodeSelector` if `nodeAffinity` is the intended mechanism.

---

### Mistake 4: preferredAffinity Without requiredAffinity for Isolation

**Symptom:** Pods intended for prod nodes land on dev nodes under load.

**Root cause:** Preferred affinity only biases the scheduler. If prod nodes are saturated, the scheduler places pods on the next best node, regardless of environment.

**Fix:** Add `requiredAffinity` to hard-restrict the candidate pool, keep `preferredAffinity` only for tie-breaking within that pool.

---

### Mistake 5: Removing Node Label After Pods Are Running — Pending New Pods

**Symptom:** Deployment has 3 running replicas. A 4th is needed. New pod is Pending. Old pods are fine.

**Root cause:** Node label was removed. Old pods: IgnoredDuringExecution (still running). New pods: affinity re-evaluated at scheduling time → no matching node.

**Diagnosis:**
```bash
kubectl describe pod <new-pod> | grep -A5 Events
# "didn't match Pod's node affinity"
kubectl get nodes --show-labels   # Confirm label is gone
```

**Fix:** Re-apply the label.
```bash
kubectl label nodes worker1 env=prod
```

---

### Mistake 6: Weight Values Out of Range

**Symptom:** YAML validation error when applying Deployment.

**Root cause:** Weight must be an integer between 1 and 100 inclusive. `weight: 0` and `weight: 101` are invalid.

**Fix:** Keep weights in the 1–100 range. Use relative weighting (e.g., 80/20, 60/40) rather than arbitrary large numbers.

---

### Mistake 7: Forgetting Default Node Labels

**Symptom:** Affinity rule using `kubernetes.io/hostname` unexpectedly matches or fails.

**Root cause:** Kubernetes automatically applies well-known labels to all nodes. Forgetting they exist causes unexpected behavior.

```bash
# Always check default labels before designing affinity rules
kubectl get nodes --show-labels | grep "kubernetes.io"
```

Common defaults: `kubernetes.io/hostname`, `kubernetes.io/os`, `kubernetes.io/arch`, `topology.kubernetes.io/zone`, `topology.kubernetes.io/region`, `node.kubernetes.io/instance-type`.

---

## 7. Interview Answers

> Compressed, verbatim-ready answers for common questions.

---

**Q: What is the difference between nodeSelector and nodeAffinity?**

> "Both are label-based placement mechanisms, but nodeAffinity is the more expressive evolution of nodeSelector. nodeSelector only supports exact key-value matches with AND logic — it's a simple dictionary match. nodeAffinity supports rich operators like In, NotIn, Exists, DoesNotExist, Gt, and Lt. More importantly, nodeAffinity supports OR logic through multiple nodeSelectorTerms entries, which nodeSelector cannot express at all. nodeAffinity also has a soft variant, preferredDuringScheduling, that allows weighted preferences — nodeSelector has no equivalent. For anything beyond a simple equality match, you should use nodeAffinity."

---

**Q: How does OR logic work in nodeAffinity?**

> "This is a common interview trap. OR logic in nodeAffinity comes from having multiple entries in the `nodeSelectorTerms` list — if any single term is fully satisfied, the pod can be scheduled on that node. Within a single term, multiple `matchExpressions` entries are AND — all of them must be satisfied simultaneously. So the structure is: terms are OR'd together, expressions within a term are AND'd together. A values list inside a single expression is also OR'd — matching any one value satisfies that expression. Getting these three levels right is what separates someone who's read the docs from someone who's used it in production."

---

**Q: What is the difference between requiredDuringScheduling and preferredDuringScheduling?**

> "Required is a hard constraint — it eliminates nodes that don't satisfy it during the Filter phase. If no node satisfies the rule, the pod stays Pending indefinitely. Preferred is a soft constraint — it runs during the Score phase and adds weight to nodes that match, but no nodes are eliminated. If the preferred node is unavailable or resource-constrained, the pod still schedules elsewhere. The practical implication is that preferred alone cannot guarantee isolation. For strict workload segregation, you always need required affinity, optionally combined with preferred for tie-breaking within the qualifying pool."

---

**Q: Is node affinity retroactive? What if you change a node's labels after pods are running?**

> "No, it's not retroactive. Both required and preferred affinity use the `IgnoredDuringExecution` variant, which means the scheduler only evaluates the rule when the pod is first being scheduled. If you remove a label from a node after pods are already running there, those pods will continue running — they are never re-evaluated. However, any new pods from the same Deployment will be Pending because no node satisfies the affinity rule. The only mechanism in Kubernetes that retroactively affects running pods is the NoExecute taint effect. There is a `RequiredDuringExecution` variant in the API spec, but it is not implemented in any released Kubernetes version yet."

---

**Q: How do Gt and Lt operators work, and what's the failure mode?**

> "Gt and Lt are numeric comparison operators available in node affinity. They compare the integer value of a node's label against a threshold. For example, `key: cpu-count, operator: Gt, values: ['4']` matches nodes where the cpu-count label has a value greater than 4. The critical detail is that the label value must be a parseable integer string. If you label a node with something like `cpu=8-core` instead of `cpu=8`, the Gt operator fails to parse it, the expression evaluates to false, and the pod goes Pending with a potentially confusing event message. The value in the YAML must also be a string in quotes even though it represents a number — that's a common YAML formatting mistake."

---

**Q: Can you have both nodeSelector and nodeAffinity on the same pod?**

> "Yes, you can, but you should understand the semantics. When both are specified, the pod must satisfy both conditions simultaneously — it's an AND relationship between the two mechanisms. The scheduler first checks nodeSelector, then nodeAffinity, and both must pass. In practice, you'd typically use this during a migration where some legacy pods still use nodeSelector and you're introducing nodeAffinity without breaking existing behavior. But for new design, there's no reason to use nodeSelector — nodeAffinity with the In operator on a single key-value pair is strictly equivalent and more maintainable."

---

**Q: How does preferredAffinity interact with other scoring plugins?**

> "preferredAffinity weight is just one input into the overall node scoring. The scheduler runs multiple scoring plugins in parallel — LeastAllocated for CPU and memory, BalancedResourceAllocation, ImageLocality, TaintToleration scoring, and others. Each produces a score that gets normalized and aggregated. A preferredAffinity weight of 100 doesn't mean the preferred node always wins — it means affinity contributes 100 units of weight relative to its scoring range. If the preferred node is at 90% CPU utilization and an unpreferred node is empty, LeastAllocated will score the empty node much higher, and it may well win. This is why preferredAffinity alone cannot guarantee placement."

---

## 8. Debugging

> Fast diagnosis paths — commands and decision trees.

### Pod Pending — Affinity-Related Diagnosis

```
kubectl get pods → STATUS: Pending
          │
          ▼
kubectl describe pod <pod> → Events section
          │
          ├── "0/N nodes are available: N node(s) didn't match Pod's node affinity"
          │         │
          │         ▼
          │   Step 1: Inspect the pod's affinity rules
          │   kubectl get pod <pod> -o yaml | grep -A40 affinity
          │         │
          │         ▼
          │   Step 2: List all nodes and their labels
          │   kubectl get nodes --show-labels
          │         │
          │         ▼
          │   Step 3: Manually evaluate: does any node satisfy the rule?
          │   kubectl get nodes -l '<key> in (<values>)'
          │         │
          │         ├── No nodes returned → Labels missing or wrong value
          │         │   → kubectl label nodes <node> <key>=<value>
          │         │
          │         ├── Nodes returned but pod still Pending
          │         │   → Check if nodeSelector is ALSO set (silent AND)
          │         │   → Check if taints are blocking those nodes
          │         │   kubectl describe node <node> | grep -A5 Taint
          │         │
          │         └── Gt/Lt in use → verify label is numeric string
          │             kubectl get node <node> --show-labels | grep <key>
          │
          ├── "0/N nodes are available: N node(s) didn't match nodeSelector"
          │         │
          │         ▼
          │   kubectl get pod <pod> -o yaml | grep -A5 nodeSelector
          │   kubectl get nodes -l '<key>=<value>'
          │   → If no nodes: apply or fix the label
          │
          ├── Mixed reason: "N affinity + M Insufficient cpu"
          │         → Affinity block AND resource block
          │         → Fix affinity first, reassess resources
          │
          └── "0/N nodes are available: N Unschedulable"
              → Node cordoned — kubectl uncordon <node>
              → Or node has unschedulable taint
```

### Pod Running on Wrong Node

```
kubectl get pods -o wide → Pod on unexpected node
          │
          ▼
Was the affinity rule required or preferred?
  ├── required → Should not be possible. Check if rule was actually applied.
  │              kubectl get pod <pod> -o yaml | grep -A30 affinity
  │              (Rule may be missing from template spec)
  │
  └── preferred → Expected behavior — scheduler chose another node.
                  Was resource pressure the cause?
                  kubectl top nodes
                  → If preferred node was saturated, rescheduled elsewhere
                  → Upgrade to required + preferred combo for stricter control
```

### Label Change Caused New Pods to Pend (Running Pods Unaffected)

```
Symptom: Deployment at 3/4 replicas. 3 old pods running. 1 new pod Pending.

kubectl describe pod <new-pod> | grep -A5 Events
→ "didn't match Pod's node affinity"

kubectl get nodes --show-labels | grep <label-key>
→ Label missing

This is IgnoredDuringExecution in action.
Fix: kubectl label nodes <node> <key>=<value>
Watch: kubectl get pods -w
```

### Useful Diagnostic Commands

```bash
# Full affinity spec of a running pod
kubectl get pod <pod> -o yaml | grep -A50 affinity

# Check what labels a node actually has
kubectl get node <node> --show-labels

# Filter nodes by label selector
kubectl get nodes -l 'storage in (ssd,hdd)'
kubectl get nodes -l 'env=prod,zone=east'   # AND
kubectl get nodes -l 'dedicated'            # key exists
kubectl get nodes -l '!dedicated'           # key does not exist

# Events sorted by time
kubectl get events -n <namespace> --sort-by='.lastTimestamp'

# Scheduler decisions (if scheduler logging is enabled)
kubectl logs -n kube-system -l component=kube-scheduler --tail=50

# Simulate scheduling (dry run — reports which node would be chosen)
kubectl apply -f pod.yaml --dry-run=server

# Check all pending pods across cluster
kubectl get pods -A --field-selector=status.phase=Pending
```

### Common Event Messages and Meaning

| Event Message | Meaning | Fix |
|---------------|---------|-----|
| `N node(s) didn't match Pod's node affinity` | No node satisfies required affinity | Check/re-apply node labels |
| `N node(s) didn't match nodeSelector` | nodeSelector labels missing | Check/re-apply node labels |
| `N node(s) had untolerated taint` | Taint blocking, affinity may be fine | Add toleration or remove taint |
| `N Insufficient cpu/memory` | Resources exhausted on qualifying nodes | Scale nodes or reduce requests |
| `N node(s) didn't match Pod's node affinity, M Insufficient cpu` | Mixed — affinity + resource both blocking | Fix affinity first |
| `Invalid value` in affinity | Gt/Lt with non-numeric label value | Relabel node with integer string |

---

## 9. Kill Switch

> 10-second recall — the absolute minimum to hold in memory.

```
PLACEMENT HIERARCHY (least → most expressive):
  nodeSelector        → exact key=value, AND only, hard
  requiredAffinity    → operators + OR logic, hard
  preferredAffinity   → operators + OR logic, soft (weighted)

AND vs OR:
  Within one nodeSelectorTerms entry (matchExpressions) → AND
  Between nodeSelectorTerms entries                      → OR
  Within a values list on one expression                 → OR

OPERATORS: In / NotIn / Exists / DoesNotExist / Gt / Lt
  Gt/Lt: label value MUST be a numeric string

PREFERRED ≠ GUARANTEED:
  Weight is one score among many.
  Resource pressure can override preference.
  Preferred alone ≠ isolation.

TRUE ISOLATION = required affinity + taint on node

NOT RETROACTIVE:
  Label changes after scheduling → running pods unaffected.
  Only NoExecute taint evicts running pods.

LABEL REMOVE: kubectl label nodes <node> <key>-
```

---

## 10. Appendix

> Quick reference card — commands, formats, cheatsheets.

### Label Management Commands

```bash
# Apply
kubectl label nodes <node> <key>=<value>

# Overwrite
kubectl label nodes <node> <key>=<value> --overwrite

# Remove (trailing dash)
kubectl label nodes <node> <key>-

# View
kubectl get nodes --show-labels
kubectl get node <node> -o=jsonpath='{.metadata.labels}'

# Filter by label
kubectl get nodes -l <key>=<value>
kubectl get nodes -l '<key> in (<v1>,<v2>)'
kubectl get nodes -l '!<key>'              # key absent
```

### Full Operator Reference

```yaml
# In — value must be in list
- key: storage
  operator: In
  values: [ssd, nvme]

# NotIn — value must NOT be in list (or key absent)
- key: env
  operator: NotIn
  values: [dev, test]

# Exists — key must be present (any value)
- key: dedicated
  operator: Exists

# DoesNotExist — key must be absent
- key: tainted
  operator: DoesNotExist

# Gt — label value > threshold (numeric strings only)
- key: cpu-count
  operator: Gt
  values: ["4"]

# Lt — label value < threshold (numeric strings only)
- key: cpu-count
  operator: Lt
  values: ["8"]
```

### Feature Comparison Table

| Feature | nodeSelector | requiredAffinity | preferredAffinity |
|---------|-------------|-----------------|------------------|
| Hard constraint | ✅ | ✅ | ❌ |
| Soft constraint (weighted) | ❌ | ❌ | ✅ |
| AND logic | ✅ | ✅ | ✅ |
| OR logic | ❌ | ✅ (multi-term) | ✅ (multi-term) |
| In / NotIn operators | ❌ | ✅ | ✅ |
| Exists / DoesNotExist | ❌ | ✅ | ✅ |
| Gt / Lt operators | ❌ | ✅ | ✅ |
| Weight/scoring | ❌ | ❌ | ✅ (1–100) |
| Retroactive (evicts pods) | ❌ | ❌ | ❌ |
| Can combine with other | ✅ (AND with affinity) | ✅ | ✅ |

### Well-Known Node Labels (Built-In)

| Label | Example Value | Purpose |
|-------|--------------|---------|
| `kubernetes.io/hostname` | `worker1` | Node name |
| `kubernetes.io/os` | `linux` | Operating system |
| `kubernetes.io/arch` | `amd64` | CPU architecture |
| `topology.kubernetes.io/zone` | `us-east-1a` | Availability zone |
| `topology.kubernetes.io/region` | `us-east-1` | Cloud region |
| `node.kubernetes.io/instance-type` | `m5.xlarge` | Cloud instance type |
| `node-role.kubernetes.io/control-plane` | `""` | Control plane role |

### Full Isolation Template (Taint + Required Affinity)

```bash
# Node setup
kubectl taint nodes <node> <taint-key>=<taint-value>:NoSchedule
kubectl label nodes <node> <label-key>=<label-value>
```

```yaml
# Deployment
spec:
  template:
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
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            preference:
              matchExpressions:
              - key: "<secondary-preference-key>"
                operator: In
                values:
                - "<value>"
```

### Scheduler Phase Summary

| Phase | Mechanism | Effect |
|-------|-----------|--------|
| Filter | `nodeSelector` | Eliminates non-matching nodes (hard) |
| Filter | `requiredAffinity` | Eliminates non-matching nodes (hard) |
| Filter | TaintToleration | Eliminates untolerated tainted nodes |
| Filter | NodeResourcesFit | Eliminates under-resourced nodes |
| Score | `preferredAffinity` | Adds weight to matching nodes (soft) |
| Score | LeastAllocated | Prefers less-utilized nodes |
| Score | ImageLocality | Prefers nodes that already have the image |
| Bind | — | Writes `nodeName` to pod spec |

---
