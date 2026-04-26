# Kubernetes Pod Scheduling: Manual Scheduling & Static Pods — CKA + Senior Interview Master Guide

---

## 0. First Principles

> *The mental model — what never changes, regardless of cluster version or configuration.*

**Kubernetes has exactly three actors involved in running a pod.** Every scheduling concept — normal, manual, static — is just a variation of which actors are present and who gets bypassed.

```
Actor 1: API Server   — stores desired state. Knows what should exist.
Actor 2: Scheduler    — decides WHERE it should run. Writes nodeName.
Actor 3: Kubelet      — makes it actually run. Talks to the container runtime.
```

**The normal pipeline:**

```
kubectl apply
    │
    ▼
API Server stores Pod object (nodeName = empty)
    │
    ▼
Scheduler watches for unbound pods (no nodeName)
Scheduler evaluates: taints, affinity, resources, topology
Scheduler writes nodeName to the Pod object
    │
    ▼
Kubelet on the assigned node watches for pods with its nodeName
Kubelet pulls image → calls container runtime → starts container
```

**Every other scheduling mode is a deliberate break in this pipeline:**

| Mode | Who is bypassed | Who owns the pod |
|---|---|---|
| Normal scheduling | Nobody | API Server + Scheduler |
| Manual scheduling (`nodeName`) | Scheduler only | API Server |
| Static pod | API Server + Scheduler | Kubelet directly |

**The single most important insight to carry into every question:**

> **Taints and tolerations, affinity rules, and scheduler policies are all enforced by the Scheduler. Bypass the Scheduler and every single one of those rules is irrelevant.** The kubelet does not consult them.

This is not a bug. It is architecture. The kubelet's job is to run whatever it is told to run, wherever it is. The scheduler's job is to decide the "wherever." Remove the scheduler from the equation and the kubelet has no gating mechanism to override.

---

## 1. Reality Constraints

> *What Kubernetes actually does and doesn't do — the gaps that cause exam failures and production incidents.*

### Manual Scheduling (`nodeName`) — What Actually Happens

| Claim | Reality |
|---|---|
| "Kubernetes deletes the pod if nodeName points to a non-existent node" | **False.** Pod stays in `Pending` forever. It is never garbage-collected by default |
| "Taints on the target node will block the pod" | **False.** Taints are a scheduler concept. `nodeName` bypasses the scheduler entirely |
| "The pod will be rescheduled if the node dies" | **False.** There is no controller watching the pod. If the node dies, the pod dies permanently |
| "You can set nodeName on a running pod to move it" | **False.** `nodeName` is immutable after pod creation. You must delete and recreate |
| "nodeName requires cluster-admin" | **False.** Any user with `create pods` permission can set `nodeName` |

### Static Pods — What Actually Happens

| Claim | Reality |
|---|---|
| "Static pods are managed by the API server" | **False.** The kubelet is the sole owner. The API server only receives a read-only mirror |
| "You can edit a static pod via kubectl" | **False.** `kubectl edit` appears to work but changes are immediately reverted by the kubelet |
| "kubectl delete removes a static pod" | **False.** The mirror pod is deleted briefly, then recreated within seconds. Delete the file |
| "Static pods can only run on control plane nodes" | **False.** Any node can run static pods if you place files in its manifest directory |
| "Static pods support all pod features" | **Mostly true** — but they cannot be part of a ReplicaSet, Deployment, or DaemonSet |

### The Mirror Pod Contract

A mirror pod is **read-only visibility**, nothing more. Understanding its limitations prevents exam mistakes:

```
Static pod file on disk
    │ (kubelet reads)
    ▼
Container running on node   ←──── SOURCE OF TRUTH
    │ (kubelet reports)
    ▼
Mirror pod in API server    ←──── READ-ONLY SHADOW
    │
    ▼
kubectl get pods shows it   ←──── DISPLAY ONLY

Consequence:
  kubectl delete pod <mirror>  →  kubelet recreates in seconds
  rm /etc/kubernetes/manifests/file.yaml  →  pod dies permanently
```

### What `nodeName` Actually Ignores

When you set `nodeName`, the following are **completely skipped**:

- Node taints and pod tolerations
- Node affinity and anti-affinity rules
- Pod topology spread constraints
- Resource availability checks (pod can be scheduled to an overloaded node)
- Node selector (`spec.nodeSelector`)
- Scheduler plugins (custom schedulers, scoring, filtering)

The kubelet still enforces: image pull policy, resource limits (if set), security contexts, and volume mounts.

---

## 2. Decision Logic

> *When to use what — clear rules before writing a single line of YAML.*

### Choosing a Scheduling Mode

```
Do you need Kubernetes to decide the node?
│
├── Yes (99% of cases) → Normal scheduling
│     Use Deployment/ReplicaSet, let the scheduler do its job
│     Add nodeSelector, affinity, or taints/tolerations to influence placement
│
└── No, you have a specific reason to override:
      │
      ├── "I need this pod on this exact node, and I know what I'm doing"
      │     → Manual scheduling (nodeName)
      │     Acceptable: debugging, one-off diagnostics, CKA exam tasks
      │     Not acceptable: production workloads
      │
      ├── "This pod must survive API server downtime"
      │     → Static pod
      │     Use case: control plane components, node-level monitoring agents
      │
      └── "I need this pod to run on every node, managed by Kubernetes"
            → DaemonSet (NOT static pods — DaemonSets are the production answer)
```

### When Static Pods Are the Right Answer

```
Is this a Kubernetes control plane component?
│
├── Yes (kube-apiserver, etcd, kube-scheduler, kube-controller-manager)
│     → Static pod. This is how kubeadm bootstraps every cluster.
│
└── No:
      │
      ├── Must run even if API server is down? → Static pod is viable
      │
      ├── Need it on every node, managed lifecycle? → DaemonSet (preferred)
      │
      └── Just want guaranteed node placement? → DaemonSet + nodeSelector
```

### Comparison Table: All Scheduling Modes

| Dimension | Normal | Manual (`nodeName`) | Static Pod | DaemonSet |
|---|---|---|---|---|
| Scheduler involved | ✅ | ❌ | ❌ | ✅ |
| API server required | ✅ | ✅ | ❌ | ✅ |
| Taints respected | ✅ | ❌ | ❌ | ✅ (with tolerations) |
| Rescheduled on node failure | ✅ (with controller) | ❌ | ❌ (restarts if node comes back) | ✅ |
| Survives API server down | ❌ | ❌ | ✅ | ❌ |
| Editable via kubectl | ✅ | ✅ | ❌ | ✅ |
| Production use | ✅ | Rare | Control plane only | ✅ |
| Source of truth | etcd | etcd | File on disk | etcd |

---

## 3. Internal Working

> *How it actually happens under the hood — step by step.*

### Manual Scheduling — Internal Flow

1. You submit a Pod manifest with `spec.nodeName` already set.
2. The **API Server** validates the manifest and persists it to etcd. The `nodeName` field is stored as-is.
3. The **Scheduler** runs its watch loop, looking for pods where `spec.nodeName == ""`. Your pod has a value — the scheduler skips it entirely. It never evaluates taints, affinity, or resources for this pod.
4. The **kubelet** on the specified node runs a watch against the API server, filtering for pods where `spec.nodeName == <this node's name>`. It finds your pod.
5. Kubelet calls the container runtime (containerd/CRI-O) to pull the image and start the container.
6. Kubelet writes pod status back to the API server.

**What happens if the nodeName node doesn't exist:**
- Kubelet on that non-existent node is never running → nobody picks up the pod
- Scheduler ignores it (nodeName is set)
- Pod stays `Pending` indefinitely — no garbage collection, no timeout

### Static Pod — Internal Flow

1. You place a YAML file in `/etc/kubernetes/manifests/` on the target node.
2. **Kubelet's file watcher** (inotify-based) detects the new file immediately. The poll interval is configurable (`--file-check-frequency`, default 20s) — for existing clusters, inotify events are near-instant.
3. Kubelet parses the YAML directly. No API server call occurs at this stage.
4. Kubelet instructs the container runtime to pull the image and start the container.
5. **If the API server is reachable**, kubelet creates a **mirror pod** in the API server — a read-only copy with `-<nodename>` appended to the name (e.g., `kube-apiserver-control-plane`). The mirror pod has `ownerReferences` pointing to the node, and an annotation `kubernetes.io/config.mirror` containing the file path hash.
6. If you `kubectl delete` the mirror pod, the API server deletes the mirror object. Kubelet detects the discrepancy — the file still exists — and recreates the mirror pod within seconds.
7. If you `rm` the file on disk, kubelet detects the file removal, stops the container, and does not recreate the mirror.

### How kubeadm Uses Static Pods to Bootstrap a Cluster

This is the answer to "how does Kubernetes start itself?":

```
kubeadm init runs on a bare node
    │
    ▼
kubeadm writes YAML files to /etc/kubernetes/manifests/:
    etcd.yaml
    kube-apiserver.yaml
    kube-controller-manager.yaml
    kube-scheduler.yaml
    │
    ▼
Kubelet (already running as a systemd service) detects files
    │
    ▼
Kubelet starts etcd container
Kubelet starts kube-apiserver container
    │
    ▼
API server comes up, kubelet connects to it
    │
    ▼
Kubelet creates mirror pods for all four components
    │
    ▼
kube-scheduler and kube-controller-manager come up
    │
    ▼
Cluster is alive — normal scheduling can now work
```

The bootstrap problem is solved by having the kubelet start before Kubernetes. The kubelet is a systemd service. Everything else — the entire control plane — is started by the kubelet as static pods.

### Kubelet's staticPodPath Configuration

The manifest directory is configurable. Its location matters for CKA tasks:

```bash
# Method 1: kubelet config file (kubeadm clusters)
cat /var/lib/kubelet/config.yaml | grep staticPodPath
# Output: staticPodPath: /etc/kubernetes/manifests

# Method 2: kubelet systemd service flags (older clusters)
systemctl cat kubelet | grep static-pod-path
# Or:
ps aux | grep kubelet | grep static-pod-path
```

---

## 4. Hands-On

> *Production-quality YAML + commands — nothing simplified.*

### Manual Scheduling — Pod with nodeName

```yaml
# manual-scheduled-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-manual
  labels:
    app: nginx
    scheduling: manual
spec:
  nodeName: my-cluster-worker-1   # Scheduler is completely bypassed
  containers:
    - name: nginx
      image: nginx:1.25
      ports:
        - containerPort: 80
      resources:
        requests:
          cpu: "50m"
          memory: "64Mi"
        limits:
          cpu: "200m"
          memory: "128Mi"
  # Note: No tolerations needed — taints are ignored when nodeName is set
```

```bash
# Apply
kubectl apply -f manual-scheduled-pod.yaml

# Verify it landed on the correct node
kubectl get pod nginx-manual -o wide

# Confirm scheduler was bypassed (nodeName was already set on creation)
kubectl get pod nginx-manual -o jsonpath='{.spec.nodeName}'

# Check Events to confirm no scheduler involvement
kubectl describe pod nginx-manual | grep -A10 Events
# You should see: "Started" directly — no "Scheduled" event from the scheduler
```

### Manual Scheduling — Assign Existing Unscheduled Pod via Binding API

If a pod is already `Pending` (scheduler is down or you removed the nodeName), you can bind it manually via the Binding subresource:

```bash
# Create a pod without nodeName (will stay Pending if scheduler is down)
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pending-pod
spec:
  schedulerName: non-existent-scheduler  # Forces Pending by naming a missing scheduler
  containers:
    - name: nginx
      image: nginx:1.25
EOF

# Bind it to a node via the Binding API (simulates what the scheduler does)
kubectl apply -f - <<EOF
apiVersion: v1
kind: Binding
metadata:
  name: pending-pod
target:
  apiVersion: v1
  kind: Node
  name: my-cluster-worker-1
EOF
```

### Static Pod — Create Directly on Node

```bash
# SSH onto the target node (or use kubectl debug for node access)
ssh user@<node-ip>

# Verify the staticPodPath
cat /var/lib/kubelet/config.yaml | grep staticPodPath

# Create the static pod manifest
cat <<EOF > /etc/kubernetes/manifests/static-nginx.yaml
apiVersion: v1
kind: Pod
metadata:
  name: static-nginx
  labels:
    app: static-nginx
    type: static
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      ports:
        - containerPort: 80
      resources:
        requests:
          cpu: "50m"
          memory: "64Mi"
        limits:
          cpu: "200m"
          memory: "128Mi"
EOF

# Verify kubelet picked it up (from your local machine)
kubectl get pods -A | grep static-nginx
# Name will be: static-nginx-<nodename>

# Verify it is a static pod (mirror pod annotation)
kubectl get pod static-nginx-<nodename> -o yaml | grep -A3 annotations
# Look for: kubernetes.io/config.mirror
```

### Confirm Static Pod Cannot Be Deleted via kubectl

```bash
# This will appear to work momentarily, then kubelet recreates it
kubectl delete pod static-nginx-<nodename>

# Watch the recreation (within ~5 seconds)
kubectl get pods -w | grep static-nginx

# Permanent deletion: remove the file on the node
ssh user@<node-ip>
rm /etc/kubernetes/manifests/static-nginx.yaml

# Confirm it's gone
kubectl get pods -A | grep static-nginx
```

### Inspect Existing Control Plane Static Pods

```bash
# List all static pod manifests on control plane
ls -la /etc/kubernetes/manifests/

# Inspect kube-apiserver static pod
cat /etc/kubernetes/manifests/kube-apiserver.yaml

# Confirm mirror pods in API server (note: name = <component>-<nodename>)
kubectl get pods -n kube-system | grep -E "etcd|kube-apiserver|kube-scheduler|kube-controller"

# Verify mirror pod annotation
kubectl get pod kube-apiserver-<nodename> -n kube-system \
  -o jsonpath='{.metadata.annotations.kubernetes\.io/config\.mirror}'
```

---

## 5. Production Flow

> *Real-world architecture and design patterns.*

### Where Static Pods Actually Appear in Production

```
Control Plane Node (kubeadm cluster)
├── /etc/kubernetes/manifests/
│     ├── etcd.yaml                    ← Static pod
│     ├── kube-apiserver.yaml          ← Static pod
│     ├── kube-controller-manager.yaml ← Static pod
│     └── kube-scheduler.yaml          ← Static pod
│
├── Kubelet (systemd service — the only non-pod control plane component)
│
└── /var/lib/kubelet/config.yaml       ← Contains: staticPodPath
```

**Why etcd is a static pod:** etcd stores all cluster state. If the API server could manage etcd, and etcd went down, the API server would lose the ability to bring etcd back. Static pods break this circular dependency.

### The Right Production Pattern for Node-Level Agents

Static pods are tempting for node-level agents (monitoring, log collectors, security scanners) but DaemonSets are almost always better:

```
Use Case: Run Fluentd on every node
│
├── Static pod approach
│     ✅ Survives API server downtime
│     ❌ No central lifecycle management
│     ❌ Cannot update via kubectl rollout
│     ❌ Cannot apply ResourceQuota
│     ❌ No rolling update support
│
└── DaemonSet approach (preferred)
      ✅ Managed rollouts
      ✅ Central configuration
      ✅ Tolerations for control plane nodes
      ✅ Works with RBAC and ResourceQuota
      ❌ Requires API server to be up

Decision rule: Use static pods only for components that must start
before or without the API server. Use DaemonSets for everything else.
```

### Modifying Control Plane Static Pods (Production Pattern)

kubeadm clusters are maintained by editing static pod manifests:

```bash
# Add a flag to kube-apiserver (e.g., enable audit logging)
# 1. Edit the manifest (kubelet will restart the component automatically)
vi /etc/kubernetes/manifests/kube-apiserver.yaml

# 2. Kubelet detects the file change and replaces the running container
# Watch the restart:
kubectl get pod kube-apiserver-<nodename> -n kube-system -w

# 3. Verify the flag took effect
kubectl get pod kube-apiserver-<nodename> -n kube-system \
  -o jsonpath='{.spec.containers[0].command}' | tr ',' '\n' | grep audit
```

### Manual Scheduling in Multi-Cluster / Debugging Workflows

```
Scenario: Scheduler is broken — pods are Pending, scheduler pod is crashlooping
│
├── Immediate mitigation: manually bind critical pods using nodeName
│     kubectl apply -f critical-pod-with-nodeName.yaml
│     (bypasses broken scheduler entirely)
│
└── Fix scheduler in parallel:
      kubectl describe pod kube-scheduler-<node> -n kube-system
      Check logs, fix config, kubelet auto-restarts it
      Once healthy, remove nodeName from pods and let scheduler take over
```

---

## 6. Mistakes

> *What actually breaks in real systems — root cause + fix.*

### Mistake 1: Editing Static Pod via kubectl — Changes Silently Revert

**Symptom:** You `kubectl edit pod kube-apiserver-<node> -n kube-system`, save changes, and they disappear within seconds.

**Root cause:** kubectl is editing the mirror pod in the API server. The kubelet's source of truth is the file on disk. Kubelet continuously reconciles the running container against the file — your changes are overwritten on the next sync cycle.

**Fix:**
```bash
# Always edit the file on the control plane node
vi /etc/kubernetes/manifests/kube-apiserver.yaml
# Kubelet detects the change and restarts the component automatically
```

---

### Mistake 2: Pod Stuck Pending — Assumed nodeName Would Be Rejected

**Symptom:** Pod applied with `nodeName: node-that-doesnt-exist` — you expected an error. Instead it silently sits `Pending` forever.

**Root cause:** Kubernetes assumes `nodeName` is intentional. No controller garbage-collects pods with a non-existent `nodeName`. The scheduler ignores it. No kubelet claims it.

**Fix:**
```bash
# Identify the issue
kubectl describe pod <n> | grep -E "Node:|Status:|Events"
# Events will show nothing — absence of events is the clue
kubectl get node <nodeName>  # Confirm node doesn't exist

# Fix: delete and recreate with correct nodeName
kubectl delete pod <n>
kubectl apply -f corrected-pod.yaml
```

---

### Mistake 3: Deleting Mirror Pod Instead of Static Pod File

**Symptom:** `kubectl delete pod static-nginx-worker1` appears to succeed. Pod returns within seconds.

**Root cause:** You deleted the mirror (shadow) in the API server. The kubelet's file watcher sees the manifest still on disk and recreates the mirror immediately.

**Fix:**
```bash
# SSH to the node where the static pod runs
ssh user@<node-ip>

# Remove the manifest file
rm /etc/kubernetes/manifests/static-nginx.yaml

# Pod disappears from API server (no file → kubelet stops container → mirror gone)
kubectl get pods -A | grep static-nginx
```

---

### Mistake 4: Assuming nodeName Respects Resource Limits on the Node

**Symptom:** Pod with `nodeName` is scheduled to a node with insufficient memory. Pod starts, immediately OOMKilled, or other pods on the node are disrupted.

**Root cause:** The scheduler checks resource availability before assigning a node. `nodeName` bypasses the scheduler entirely — no resource check is performed. The kubelet will still enforce container-level resource limits after the fact, but it does not refuse to run the pod.

**Fix:**
```bash
# Always verify node resources before using nodeName
kubectl describe node <nodeName> | grep -A10 "Allocated resources"
# Check "Requests" vs "Capacity" for CPU and Memory

# Only use nodeName if you're certain the node has headroom
```

---

### Mistake 5: Wrong staticPodPath — Static Pod Never Starts

**Symptom:** You placed a YAML in `/etc/kubernetes/manifests/` but the pod never appears.

**Root cause:** The `staticPodPath` in kubelet config may be different from the expected default, or you placed the file on the wrong node.

**Fix:**
```bash
# Verify staticPodPath on the target node
cat /var/lib/kubelet/config.yaml | grep staticPodPath

# Alternative: check kubelet process flags
ps aux | grep kubelet | tr ' ' '\n' | grep static

# Confirm file is in the correct directory with correct extension (.yaml or .json)
ls -la $(cat /var/lib/kubelet/config.yaml | grep staticPodPath | awk '{print $2}')

# Check kubelet logs for parsing errors
journalctl -u kubelet -f | grep -i static
```

---

### Mistake 6: Expecting nodeName to Work Like Node Affinity

**Symptom:** Pod with `nodeName` set runs on the wrong node — candidate assumes nodeName is just a "preference."

**Root cause:** `nodeName` is a **hard assignment**, not a preference. It is not the same as `nodeSelector` or `nodeAffinity`. It completely bypasses scheduling — the pod runs on exactly that node or nowhere (Pending).

**Comparison:**
```yaml
# nodeName = hard assignment, bypasses scheduler
spec:
  nodeName: worker-1

# nodeSelector = soft filter, scheduler still runs
spec:
  nodeSelector:
    disktype: ssd

# nodeAffinity = expressive filter, scheduler still runs
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: disktype
                operator: In
                values: ["ssd"]
```

---

## 7. Interview Answers

> *Compressed, verbatim-ready answers. Practice speaking these aloud.*

---

**Q: What is manual scheduling in Kubernetes and when would you use it?**

"Manual scheduling means setting the `spec.nodeName` field directly on a Pod, which tells Kubernetes to skip the scheduler entirely. The API server stores the pod, the scheduler ignores it because `nodeName` is already set, and the kubelet on the specified node picks it up and runs it. Critically, because the scheduler is bypassed, node taints, affinity rules, and resource availability checks are all ignored — the kubelet will run it regardless. You'd use this during debugging when the scheduler is broken and you need critical pods running, or in CKA exam tasks that explicitly ask for manual scheduling. In production with healthy clusters, you almost never use it."

---

**Q: What happens if you set nodeName to a node that doesn't exist?**

"The pod goes into Pending state and stays there indefinitely. Kubernetes does not garbage-collect it, does not surface an error, and does not retry. This is because Kubernetes treats the `nodeName` assignment as intentional — it assumes you know what you're doing. The scheduler ignores the pod since `nodeName` is already set. No kubelet ever claims it because no kubelet is running on a node with that name. The absence of events in `kubectl describe` is usually the first clue that this has happened."

---

**Q: What is a static pod and how is it different from a regular pod?**

"A static pod is a pod that the kubelet manages directly by watching a directory on disk — typically `/etc/kubernetes/manifests/` — rather than receiving instructions from the API server. The flow is completely different: you drop a YAML file on disk, the kubelet's file watcher detects it, and the kubelet starts the container without any API server involvement. If the API server is reachable, the kubelet creates a mirror pod in the API server for visibility, but that mirror is read-only — you can't edit or truly delete it via kubectl. The real difference from a regular pod is ownership: a regular pod is owned by etcd via the API server; a static pod is owned by a file on disk."

---

**Q: Why does `kubectl delete` not work on static pods?**

"Because `kubectl delete` removes the mirror pod from the API server, not the source file on disk. The kubelet's source of truth is the file in the manifest directory. When the kubelet detects that the mirror pod is gone but the file still exists, it recreates the mirror within seconds. The only way to permanently stop a static pod is to remove the file from the manifest directory on the node. This is a common exam trap — you need to SSH to the node and `rm` the file."

---

**Q: Why do control plane components like kube-apiserver run as static pods?**

"Because of the bootstrap problem: you can't use Kubernetes to manage Kubernetes before Kubernetes exists. kube-apiserver can't manage itself — if it went down, it couldn't restart itself via the normal pod lifecycle. Static pods solve this because they're managed entirely by the kubelet, which is a systemd service that runs on the node independently of the Kubernetes API. When you run `kubeadm init`, it writes YAML manifests for etcd, kube-apiserver, kube-controller-manager, and kube-scheduler into `/etc/kubernetes/manifests/`. The kubelet, already running, picks those up and starts the entire control plane. The cluster bootstraps itself from nothing."

---

**Q: Can a static pod run on a worker node? What about control plane taints?**

"Yes, absolutely. Static pods can run on any node that has a kubelet and a manifest directory configured. The control plane taint — `node-role.kubernetes.io/control-plane:NoSchedule` — is irrelevant for static pods because taints are enforced by the scheduler. Static pods bypass the scheduler entirely, so no taint check ever happens. The kubelet doesn't validate taints. This is actually the same reason that manual scheduling with `nodeName` also ignores taints."

---

**Q: What is the difference between nodeName and nodeSelector?**

"They operate at fundamentally different layers. `nodeName` is a hard assignment that completely bypasses the scheduler — you're telling Kubernetes 'run this on exactly this node, don't ask the scheduler.' `nodeSelector` is a filter that the scheduler uses during its node selection phase — it narrows the candidates, but the scheduler still runs, still evaluates taints and resource availability, and still writes the final `nodeName` to the pod. `nodeName` is also immutable after pod creation, while `nodeSelector` is part of the pod spec that influences scheduling but doesn't hardcode a destination. In production, `nodeSelector` and `nodeAffinity` are the right tools; `nodeName` is for surgical overrides."

---

## 8. Debugging

> *Fast diagnosis paths — not a list of commands, but an actual decision tree.*

### Pod Stuck Pending — Distinguish Manual Scheduling vs Scheduler Issue

```
Pod is Pending — why?
│
├── Step 1: Check if nodeName is set
│     kubectl get pod <n> -o jsonpath='{.spec.nodeName}'
│     → Empty string → scheduler hasn't assigned it (go to Step 2)
│     → Node name present → scheduler was bypassed, problem is with that node (go to Step 4)
│
├── Step 2: Is the scheduler running?
│     kubectl get pods -n kube-system | grep kube-scheduler
│     → Not running / CrashLooping → scheduler is the problem
│       Fix: kubectl describe pod kube-scheduler-<n> -n kube-system → check logs
│     → Running → scheduler rejected the pod (go to Step 3)
│
├── Step 3: Why did the scheduler reject it?
│     kubectl describe pod <n> | grep -A20 Events
│     Common messages:
│       "0/3 nodes are available: 3 node(s) had taint..."  → missing toleration
│       "0/3 nodes are available: Insufficient cpu"        → resource quota or node pressure
│       "0/3 nodes are available: node(s) didn't match"   → nodeSelector/affinity mismatch
│     Fix based on message:
│       Taint → add toleration or use nodeName to bypass
│       Resources → reduce requests or add nodes
│       Selector → fix labels on nodes or pod spec
│
└── Step 4: nodeName is set but pod is still Pending
      Does the node exist?
        kubectl get node <nodeName>
        → NotFound → wrong nodeName; delete pod, fix, reapply
        → Exists but NotReady → node is down; pod waits indefinitely
      Is kubelet running on that node?
        kubectl describe node <nodeName> | grep -A5 Conditions
        → MemoryPressure/DiskPressure → node can't accept pods
```

### Static Pod Not Appearing — Diagnosis Path

```
Expected static pod is missing from kubectl get pods
│
├── Step 1: Confirm file exists on the correct node
│     ssh user@<node>
│     ls /etc/kubernetes/manifests/
│     → File missing → create it
│     → File exists → continue
│
├── Step 2: Verify staticPodPath matches the directory
│     cat /var/lib/kubelet/config.yaml | grep staticPodPath
│     → Different directory → move file to correct location
│
├── Step 3: Check kubelet is running
│     systemctl status kubelet
│     → Inactive/Failed → restart: systemctl start kubelet
│     → Active but pod still missing → check Step 4
│
├── Step 4: Check kubelet logs for parsing errors
│     journalctl -u kubelet --since "5 minutes ago" | grep -i "static\|manifest\|error"
│     → YAML parse error → fix syntax in manifest file
│     → Permission denied → fix file permissions: chmod 644 /etc/kubernetes/manifests/file.yaml
│
└── Step 5: Force kubelet to re-read (touch the file)
      touch /etc/kubernetes/manifests/static-nginx.yaml
      # Or: systemctl restart kubelet (last resort)
```

### Quick Diagnostic Commands

```bash
# Check what node a pod is assigned to (and how)
kubectl get pod <n> -o wide
kubectl get pod <n> -o jsonpath='{.spec.nodeName}'

# Confirm whether a pod is a static pod (mirror pod check)
kubectl get pod <n> -o yaml | grep -E "config.mirror|ownerReferences"

# Find all static pods in the cluster (mirror pods have the mirror annotation)
kubectl get pods -A -o json | \
  jq '.items[] | select(.metadata.annotations["kubernetes.io/config.mirror"] != null) | .metadata.name'

# Check staticPodPath on a node
kubectl debug node/<nodename> -it --image=busybox -- \
  chroot /host cat /var/lib/kubelet/config.yaml | grep staticPodPath

# View kubelet logs for static pod activity
journalctl -u kubelet -f | grep -i static

# Check which scheduler a pod is using
kubectl get pod <n> -o jsonpath='{.spec.schedulerName}'
# "default-scheduler" = normal
# Anything else / empty (with nodeName set) = bypassed

# List events for a pending pod (absence of Scheduled event = manual or static)
kubectl describe pod <n> | grep -A20 Events

# Verify node resource availability before manual scheduling
kubectl describe node <nodename> | grep -A15 "Allocated resources"
```

---

## 9. Kill Switch

> *10-second recall — the absolute minimum to hold in memory under exam pressure.*

```
THREE ACTORS: API Server (stores) | Scheduler (places) | Kubelet (runs)

MANUAL SCHEDULING (nodeName):
  Bypasses: Scheduler only
  Ignores: ALL taints, affinity, resource checks
  If node missing: Pending FOREVER (never garbage collected)
  nodeName is IMMUTABLE after creation

STATIC POD:
  Bypasses: API Server + Scheduler
  Owner: File on disk (kubelet reads it)
  Mirror pod = READ-ONLY shadow (kubectl delete → recreates in seconds)
  Real delete: rm /etc/kubernetes/manifests/file.yaml
  Default path: /etc/kubernetes/manifests/
  Control plane uses it to bootstrap (kube-apiserver, etcd, scheduler, controller-manager)

TAINT RULE:
  Taints = scheduler concept
  nodeName bypasses scheduler → taints irrelevant
  Static pod bypasses scheduler → taints irrelevant

DEBUG:
  Pending + nodeName set → check if node exists
  Pending + no nodeName → check scheduler
  Static pod missing → check file path + kubelet logs
  kubectl delete static pod → file still there → kubelet recreates
```

---

## 10. Appendix

### Quick Reference: Scheduling Modes

| Mode | Set By | nodeName | Scheduler | API Server | Taint Check | Reschedule on Failure |
|---|---|---|---|---|---|---|
| Normal | Scheduler | Written by scheduler | ✅ | ✅ | ✅ | ✅ (with controller) |
| Manual | You | You set it | ❌ | ✅ | ❌ | ❌ |
| Static | Kubelet | N/A (local node) | ❌ | Optional (mirror) | ❌ | ❌ (restarts locally) |

### Quick Reference: Static Pod vs DaemonSet

| Feature | Static Pod | DaemonSet |
|---|---|---|
| Managed by | Kubelet | API Server + DaemonSet controller |
| Runs on all nodes | Only if file on each node | Automatically |
| Rolling update | Manual (edit file, cp to all nodes) | `kubectl rollout` |
| Survives API server down | ✅ | ❌ |
| kubectl edit works | ❌ (mirror reverts) | ✅ |
| Use in production | Control plane only | Node agents, logging, monitoring |

### Quick Reference: Key Files and Paths

```bash
# Static pod manifest directory (kubeadm default)
/etc/kubernetes/manifests/

# Kubelet config (contains staticPodPath)
/var/lib/kubelet/config.yaml

# Kubelet systemd unit file
/etc/systemd/system/kubelet.service.d/10-kubeadm.conf

# Find staticPodPath from kubelet config
cat /var/lib/kubelet/config.yaml | grep staticPodPath

# Find staticPodPath from process args (older clusters)
ps aux | grep kubelet | tr ' ' '\n' | grep static-pod-path
```

### Quick Reference: Key Commands

```bash
# Create pod on specific node (manual scheduling)
kubectl apply -f pod-with-nodeName.yaml

# Check which node a pod is on
kubectl get pod <n> -o wide

# Check if pod is a static pod
kubectl get pod <n> -o yaml | grep "config.mirror"

# List control plane static pods
kubectl get pods -n kube-system | grep -E "etcd|apiserver|scheduler|controller"

# Delete a static pod (permanent)
ssh <node>; rm /etc/kubernetes/manifests/<file>.yaml

# Find all mirror pods cluster-wide
kubectl get pods -A -o yaml | grep "config.mirror" -B5

# Check kubelet static pod logs
journalctl -u kubelet | grep -i "static\|manifest"

# Verify node has capacity before manual scheduling
kubectl describe node <nodename> | grep -A10 "Allocated resources"

# Force kubelet to re-read manifests
systemctl restart kubelet   # last resort only

# Get schedulerName (empty + nodeName = manual)
kubectl get pod <n> -o jsonpath='{.spec.schedulerName}'
```

### YAML Skeletons

```yaml
# Minimal manual scheduling pod
apiVersion: v1
kind: Pod
metadata:
  name: manual-pod
spec:
  nodeName: <exact-node-name>   # kubectl get nodes to find name
  containers:
    - name: nginx
      image: nginx:1.25
```

```yaml
# Minimal static pod (place in /etc/kubernetes/manifests/)
apiVersion: v1
kind: Pod
metadata:
  name: static-pod            # Final name will be: static-pod-<nodename>
spec:
  containers:
    - name: nginx
      image: nginx:1.25
  # No nodeName needed — kubelet infers local node
  # No tolerations needed — scheduler is not involved
```

### Common Gotchas Cheatsheet

| Gotcha | Rule |
|---|---|
| Pod Pending with nodeName set | Check if node exists — pod waits forever, never garbage collected |
| Taint blocks nodeName pod | It doesn't — taints are scheduler-only; nodeName bypasses scheduler |
| kubectl delete removes static pod | It doesn't — delete the file on disk |
| kubectl edit static pod works | It appears to, but reverts in seconds — edit the manifest file |
| nodeName is a preference like nodeSelector | No — nodeName is a hard binding, immutable after creation |
| Static pods only on control plane | Wrong — any node with kubelet + manifest directory works |
| staticPodPath is always /etc/kubernetes/manifests/ | Not guaranteed — verify with `cat /var/lib/kubelet/config.yaml` |
| Mirror pod name = pod name | Mirror pod name = `<metadata.name>-<nodename>` |
