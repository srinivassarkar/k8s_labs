# ⚙️ Kubernetes Cluster Upgrades
> **Topic:** `kubeadm` cluster upgrades, version skew, PodDisruptionBudgets, drain semantics  
> **Scope:** CKA exam readiness + production-grade depth  
> **Cluster used in examples:** v1.32.8 → v1.33.4 (Ubuntu/Debian, Calico CNI, containerd CRI)

---

## 0. First Principles — The Mental Model That Never Changes

> *These are the axioms. Everything else is derived from them.*

**1. Kubernetes versions are a contract, not a suggestion.**  
The API server is the source of truth. Every other component must be ≤ its version. No exceptions.

**2. You can only upgrade one minor at a time.**  
`1.31 → 1.33` is not a valid path. `1.31 → 1.32 → 1.33` is. Kubernetes minor upgrades are sequential by design.

**3. Control plane always upgrades before workers.**  
A newer kubelet talking to an older API server is undefined behavior. The reverse (older kubelet, newer API server) is the supported rolling window.

**4. Kubernetes will not protect workloads you didn't protect yourself.**  
If you have no PDBs, no readiness probes, and one replica — drain will happily evict your pod into oblivion. The cluster is not responsible for your application's availability.

**5. `kubectl drain` uses the Eviction API, not force-kill.**  
This is why PDBs work. Eviction is polite. The API server checks PDB allowances before honoring eviction requests.

**6. Node version (kubelet) ≠ cluster version.**  
`kubectl get nodes` shows kubelet version. `kubectl version` shows the API server version. These can differ during a rolling upgrade — that's expected.

---

## 1. Reality Constraints — What Kubernetes Actually Does and Doesn't Do

### Version Skew Rules (Hard Limits)

| Component | Allowed Skew vs. kube-apiserver | Notes |
|---|---|---|
| `kube-controller-manager` | ≤ same minor, may be 1 minor behind | Must never be **newer** than apiserver |
| `kube-scheduler` | ≤ same minor, may be 1 minor behind | Same rule as controller-manager |
| `kubelet` | Up to **2 minors behind** apiserver | This is your rolling window for worker upgrades |
| `kubectl` | **±1 minor** from apiserver | Admin laptops can be N+1 or N-1 |
| `cloud-controller-manager` | ≤ same minor, may be 1 minor behind | Follows scheduler/controller-manager rule |

> **The golden rule:** Never run any component **newer** than the API server. Converge everything to the same minor as fast as operationally safe.

### Version Lifecycle

```
MAJOR.MINOR.PATCH  →  1.33.4
                       ^  ^  ^
                       |  |  +-- Patch: monthly, security/bugfix only
                       |  +----- Minor: ~every 4 months (~3/year), new features
                       +-------- Major: 1 since forever, semver theater
```

**Support window:** Only **N / N-1 / N-2** receive patches.

```
When 1.33 is current:
  ✅ 1.33 — supported (N)
  ✅ 1.32 — supported (N-1)
  ✅ 1.31 — supported (N-2)
  ❌ 1.30 — EOL, no more patches
```

### What `kubeadm upgrade` Does (and Doesn't Do)

**Does:**
- Updates static pod manifests for control-plane components (new image tags)
- Renews certificates it manages (by default)
- Updates RBAC rules and bootstrap tokens
- Updates CoreDNS and kube-proxy ConfigMaps

**Does NOT:**
- Upgrade `kubelet` or `kubectl` — you do that manually with `apt`
- Upgrade your CNI (Calico, Flannel, Cilium) — managed separately
- Upgrade your CSI drivers or ingress controllers
- Drain or cordon any node — that's your job
- Back up etcd — that's also your job

### Managed Kubernetes Reality Check

| Provider | Support Window | Auto-upgrade? |
|---|---|---|
| Upstream | N / N-1 / N-2 (~12 months) | No |
| Amazon EKS | ~14 months + 12 months extended (paid) | Optional |
| Azure AKS | Typically N-2 boundary | Auto-upgrade channels available |
| Google GKE | Typically N-2 boundary | Release channels (Rapid/Regular/Stable) |

> For managed clusters: **open tickets with your cloud provider**, not the Kubernetes community. Their SLA, their problem.

---

## 2. Decision Logic — When to Use What

### Should I Upgrade Now?

```
Is your current minor in N / N-1 / N-2?
  └── NO → Upgrade immediately. You are running unsupported software.
  └── YES → Are you on the latest patch for your minor?
              └── NO → Are you prod? Pin to validated patch. Otherwise, update now.
              └── YES → You're fine. Plan the next minor upgrade.
```

### Which Upgrade Strategy?

```
Is this a production cluster with an SLO?
  └── NO (dev/test) → "All at once" is fine
  └── YES → Do you need instant rollback capability?
              └── YES → Blue/Green (parallel cluster)
              └── NO → Rolling upgrade (one node at a time)
```

### When to Use `kubeadm upgrade apply` vs `kubeadm upgrade node`?

| Command | Use On | What It Does |
|---|---|---|
| `kubeadm upgrade apply <version>` | **First control-plane node only** | Upgrades the full cluster control plane |
| `kubeadm upgrade node` | All **other** control-plane nodes + all worker nodes | Aligns node to the already-upgraded cluster |

> **If you run `apply` on a second CP node, you'll confuse the cluster. Use `node` on everything except the first CP.**

### Cordon vs Drain — When to Use Each

```
Need to stop NEW pods from landing on a node?
  └── cordon only (reversible, non-disruptive)

Need to empty a node for maintenance?
  └── cordon + drain (evicts existing pods via Eviction API)
```

### PDB: `minAvailable` vs `maxUnavailable`

| Field | Meaning | Example (4 replicas) |
|---|---|---|
| `minAvailable: 3` | At least 3 pods must be Ready at all times | Allows 1 eviction at a time |
| `maxUnavailable: 1` | At most 1 pod may be unavailable at a time | Same effect here, but semantically clearer |
| `minAvailable: 75%` | Percentage-based (floor math) | 75% of 4 = 3 |

> **Pro tip:** `maxUnavailable` is often more intuitive for operations teams. `minAvailable` is easier to reason about during scale-up events.

---

## 3. Internal Working — How It Actually Happens Under the Hood

### What Happens During `kubeadm upgrade apply v1.33.4`

```
Step 1: Pre-flight checks
  - Verify kubeadm version matches target
  - Confirm etcd health (stacked or external)
  - Validate API server reachability
  - Check existing component versions

Step 2: Pull new control-plane images
  - API server, controller-manager, scheduler, etcd images
  - Equivalent to: kubeadm config images pull

Step 3: Update static pod manifests
  - Location: /etc/kubernetes/manifests/
  - Files: kube-apiserver.yaml, kube-controller-manager.yaml,
           kube-scheduler.yaml, etcd.yaml
  - Updates: image tag, feature flags, API deprecation flags

Step 4: kubelet detects manifest changes
  - kubelet watches /etc/kubernetes/manifests/ via inotify
  - Triggers pod stop → container pull → pod start
  - One component restarts at a time (graceful)

Step 5: Certificate renewal
  - kubeadm renews certificates in /etc/kubernetes/pki/
  - Controlled by --certificate-renewal flag (default: true)

Step 6: Update kube-proxy and CoreDNS
  - Updates their DaemonSet/Deployment image tags
  - Kubernetes controllers roll these out normally

Step 7: Print upgrade summary
  - Tells you what still needs manual action (kubelet, kubectl)
```

### What Happens During `kubectl drain`

```
kubectl drain worker-1 --ignore-daemonsets --delete-emptydir-data

Step 1: Cordon the node (automatic)
  - Node.spec.unschedulable = true
  - New pods won't land here

Step 2: List evictable pods
  - Skips: DaemonSet pods (--ignore-daemonsets)
  - Skips: Mirror pods (static pods via kubelet)
  - Includes: everything else (Deployments, StatefulSets, Jobs, etc.)

Step 3: For each pod, send Eviction request to API server
  - Uses: POST /api/v1/namespaces/{ns}/pods/{pod}/eviction
  - API server checks PDB allowances before honoring

Step 4: If PDB blocks eviction:
  - Request returns 429 Too Many Requests
  - drain retries (default: 20s intervals, indefinite timeout)
  - You'll see: "Cannot evict pod as it would violate the pod's disruption budget"

Step 5: If eviction is allowed:
  - Pod enters Terminating state
  - terminationGracePeriodSeconds countdown begins
  - Deployment controller notices → schedules replacement elsewhere
  - Once replacement is Ready, PDB allowance refreshes

Step 6: drain completes when node has no evictable pods
```

### How PDB Enforcement Works Internally

```
PDB: minAvailable: 3, Deployment replicas: 4

Eviction request arrives for pod-A on worker-1:

API server calculates:
  currentHealthy  = count pods where .status.phase == Running
                    AND .status.conditions[type=Ready].status == True
  expectedPods    = 4 (from controller)
  allowedDisruptions = max(0, currentHealthy - minAvailable)
                     = max(0, 4 - 3) = 1

allowedDisruptions > 0 → Eviction GRANTED
allowedDisruptions == 0 → 429 Returned, eviction BLOCKED
```

> **Key insight:** A pod in `Terminating` state is **not counted as Ready**. So if 3 pods are Running/Ready and 1 is Terminating, the Eviction API will block the next eviction until the replacement becomes Ready.

---

## 4. Hands-On — Production-Quality YAML + Commands

### PodDisruptionBudget + Deployment (Complete Example)

```yaml
# pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: red-deploy-pdb
  namespace: default
spec:
  minAvailable: 3          # Always keep >= 3 pods Ready
  # maxUnavailable: 1      # Alternative: allow at most 1 to be unavailable
  selector:
    matchLabels:
      app: red-deploy      # Must match your Deployment's pod labels
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: red-deploy
  namespace: default
spec:
  replicas: 4              # Must be > minAvailable, or drain will stall
  selector:
    matchLabels:
      app: red-deploy
  template:
    metadata:
      labels:
        app: red-deploy
    spec:
      containers:
      - name: web
        image: nginx:1.25
        readinessProbe:        # CRITICAL: PDB counts Ready pods — without this, 
          httpGet:             # replacement pods won't "count" until truly ready
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
```

---

### Full Control-Plane Upgrade (1.32.8 → 1.33.4)

#### Pre-flight: Back Up Everything

```bash
# 1. Snapshot etcd (stacked: run from a control-plane node)
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /backup/etcd-snapshot-$(date +%Y%m%d%H%M).db

# 2. Back up PKI and manifests
sudo tar czf /backup/k8s-config-$(date +%Y%m%d%H%M).tar.gz /etc/kubernetes/

# 3. Verify etcd health
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health
```

#### Step 1: Point APT to v1.33

```bash
# Check current repo config
pager /etc/apt/sources.list.d/kubernetes.list

# Edit: change 1.32 → 1.33
sudo vim /etc/apt/sources.list.d/kubernetes.list
# Line should read exactly:
# deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /

# Refresh package index
sudo apt-get update
```

#### Step 2: Find the Exact Target Version

```bash
sudo apt-cache madison kubeadm
# kubeadm | 1.33.4-1.1 | https://pkgs.k8s.io/core:/stable:/v1.33/deb  Packages
# kubeadm | 1.33.3-1.1 | ...
# Pick the latest: 1.33.4-1.1
```

#### Step 3: Upgrade kubeadm

```bash
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm=1.33.4-1.1
sudo apt-mark hold kubeadm
kubeadm version  # Verify: v1.33.4

# Optional: pre-pull images to minimize restart time
sudo kubeadm config images pull
```

#### Step 4: Dry-Run (Plan)

```bash
sudo kubeadm upgrade plan

# Output shows:
# - Components upgraded automatically (apiserver, scheduler, etcd, CoreDNS)
# - Components needing manual action (kubelet on each node)
# - Exact target versions

# See manifest diffs before committing:
sudo kubeadm upgrade diff v1.33.4
```

#### Step 5: Apply the Upgrade

```bash
sudo kubeadm upgrade apply v1.33.4

# Expected tail:
# [upgrade] SUCCESS! A control plane node of your cluster was upgraded to "v1.33.4".
# [upgrade] Now please proceed with upgrading the rest of the nodes...
```

#### Step 6: Verify Skew State

```bash
kubectl version
# Client Version: v1.32.8   (old kubectl still on PATH)
# Server Version: v1.33.4   ← API server upgraded

kubectl get nodes
# NAME            STATUS   ROLES           VERSION
# control-plane   Ready    control-plane   v1.32.8   ← kubelet not yet upgraded
# worker-1        Ready    <none>          v1.32.8
# worker-2        Ready    <none>          v1.32.8
```

#### Step 7: Upgrade kubelet + kubectl on Control Plane

```bash
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.33.4-1.1 kubectl=1.33.4-1.1
sudo systemctl daemon-reload && sudo systemctl restart kubelet
sudo apt-mark hold kubeadm kubelet kubectl

# Verify
kubectl get nodes
# control-plane   Ready    control-plane   v1.33.4   ← now updated
```

---

### Worker Node Upgrade (Repeat Per Node)

```bash
# ── FROM ADMIN SHELL ─────────────────────────────────────────
kubectl cordon worker-1
kubectl get pods -A -o wide | grep worker-1   # Review impact
kubectl drain worker-1 \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=60 \
  --timeout=10m

# ── ON THE WORKER NODE ───────────────────────────────────────
sudo vim /etc/apt/sources.list.d/kubernetes.list
# Ensure: .../v1.33/deb/ /

sudo apt-mark unhold kubeadm kubelet kubectl
sudo apt-get update
sudo apt-get install -y kubeadm=1.33.4-1.1
sudo kubeadm upgrade node          # ← NOT apply; align to upgraded cluster
sudo apt-get install -y kubelet=1.33.4-1.1 kubectl=1.33.4-1.1
sudo systemctl daemon-reload && sudo systemctl restart kubelet
sudo apt-mark hold kubeadm kubelet kubectl

# ── FROM ADMIN SHELL ─────────────────────────────────────────
kubectl uncordon worker-1
kubectl get nodes -o wide           # Verify worker-1 shows v1.33.4
```

---

### HA Control-Plane Upgrade (Additional Nodes)

```bash
# Run 'kubeadm upgrade apply' on EXACTLY ONE control-plane node (done above).
# For EACH additional CP node:

sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm=1.33.4-1.1
sudo apt-mark hold kubeadm
sudo kubeadm upgrade node   # ← 'node', not 'apply'

sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.33.4-1.1 kubectl=1.33.4-1.1
sudo systemctl daemon-reload && sudo systemctl restart kubelet
sudo apt-mark hold kubeadm kubelet kubectl
```

> **Load balancer note:** Your API LB should health-check `/readyz` on each CP node and pull it from rotation while its API server restarts. Never health-check `/healthz` — that returns 200 even during shutdown.

---

### Calico BGP Fix (VXLAN Clusters)

```bash
# Symptom: calico-node Running 0/1 after upgrade

# Check BGP state
kubectl get installation.operator.tigera.io default \
  -o jsonpath='{.spec.calicoNetwork.bgp}{"\n"}'

# Disable BGP (VXLAN doesn't need it)
kubectl patch installation.operator.tigera.io default --type=merge \
  -p '{"spec":{"calicoNetwork":{"bgp":"Disabled"}}}'

# Restart calico-node DaemonSet
kubectl -n calico-system rollout restart ds/calico-node
kubectl -n calico-system rollout status ds/calico-node

# If still failing: check Typha connectivity (port 5473/TCP)
kubectl -n calico-system logs ds/calico-node -c calico-node | grep -i typha
# Fix: open TCP 5473 between nodes in your Security Groups/NSGs
```

**Required ports summary:**

| Protocol | Port | Purpose | Required? |
|---|---|---|---|
| UDP | 4789 | VXLAN overlay | ✅ Always |
| TCP | 5473 | calico-node ↔ Typha | ✅ When Typha is deployed |
| TCP | 179 | BGP | ❌ Only if BGP enabled |

---

## 5. Production Flow — Real-World Architecture and Design Patterns

### Patch Management Policy (N / N-1 / N-2)

```
Environment    Minor     Patch Policy
──────────────────────────────────────────────────────────────
Prod (N)       1.33      Pin to validated patch (e.g. 1.33.3).
                         Promote new patches AFTER staging soak.
                         May intentionally LAG latest by 1-2 patches.

Staging (N)    1.33      Track latest patch. Purpose: validate before prod.

N-1 (Prod)     1.32      Run latest patch immediately. Less risk, more fixes.

N-2 (Prod)     1.31      Run latest patch. Approaching EOL — plan upgrade.
```

> **Real-world cadence:** Most mature teams upgrade every other minor (~6-8 months), with monthly patches in between. That keeps you inside N/N-1/N-2 without constant disruption.

### Pre-Upgrade Checklist

```
□ etcd snapshot completed and verified
□ /etc/kubernetes/ tarball on each CP node
□ PDBs in place for all critical workloads
□ Readiness probes defined on all workloads
□ Replica counts > 1 for everything that matters
□ Capacity headroom confirmed (or temp scale-up scheduled)
□ CNI/CSI/CRI compatibility verified against target minor
□ API deprecations reviewed (check: kubectl api-versions)
□ Upgrade window communicated to stakeholders
□ Rollback procedure documented and tested
□ Load balancer health-checks verified (/readyz not /healthz)
```

### Upgrade Strategy Decision Table

| Strategy | When to Use | Cost | Rollback Speed |
|---|---|---|---|
| **All at once** | Dev/test, no SLO | Low | Manual, slow |
| **Rolling (per node)** | Production, standard | Medium | Per-node uncordon |
| **Blue/Green** | High SLO, risky OS/CNI change | High | Instant (LB swap) |

### Managing Multiple kubectl Versions

```bash
# Option 1: Multiple binaries with aliases
kubectl-1.32 get nodes   # Add to PATH as kubectl-1.XX
kubectl-1.33 get nodes

# Option 2: asdf (auto-switches per project directory)
asdf install kubectl 1.33.4
asdf local kubectl 1.33.4

# Option 3: Docker (zero-install)
docker run --rm -v ~/.kube:/root/.kube bitnami/kubectl:1.33 get nodes

# Name your contexts with version to avoid confusion:
kubectl config rename-context kubernetes-admin@kubernetes prod-1.33
kubectl version  # Always sanity-check after context switch
```

---

## 6. Mistakes — What Actually Breaks in Real Systems

### Mistake 1: Skipping a Minor Version

**Symptom:** `kubeadm upgrade plan` or `apply` fails with a version skew error, or worse — it succeeds but API objects are silently migrated incorrectly.

**Root cause:** Internal API storage versions and migration hooks are only designed for N→N+1. Skipping breaks assumptions in the migration logic.

**Fix:** Always upgrade sequentially: `1.31 → 1.32 → 1.33`. No shortcuts.

---

### Mistake 2: Forgetting to Upgrade kubelet After `kubeadm upgrade apply`

**Symptom:** `kubectl get nodes` shows control-plane still at old version. Developers panic.

**Root cause:** `kubeadm upgrade apply` does NOT upgrade kubelet. It only updates static pod manifests. kubelet is a systemd service upgraded separately via apt.

**Fix:** Always complete Step 7 (apt install kubelet + systemctl restart kubelet) before moving to workers.

---

### Mistake 3: PDB Blocks Drain Forever

**Symptom:** `kubectl drain` hangs indefinitely with:
```
Cannot evict pod as it would violate the pod's disruption budget.
```

**Root cause:** Either (a) `minAvailable` equals total replicas leaving zero eviction allowance, or (b) replacement pods can't schedule (no capacity) so `allowedDisruptions` never rises above 0.

**Fix options (pick the appropriate one):**
```bash
# Option A: Temporarily relax the PDB
kubectl patch pdb red-deploy-pdb -p '{"spec":{"minAvailable":2}}'
# ... drain and upgrade ...
kubectl patch pdb red-deploy-pdb -p '{"spec":{"minAvailable":3}}'

# Option B: Scale up replicas to create headroom
kubectl scale deployment red-deploy --replicas=5
# Now allowedDisruptions = 5-3 = 2

# Option C: Force drain (loses PDB protection — last resort)
kubectl drain worker-1 --ignore-daemonsets --delete-emptydir-data --disable-eviction
# WARNING: This bypasses PDBs entirely
```

---

### Mistake 4: Running `kubeadm upgrade apply` on Multiple CP Nodes (HA)

**Symptom:** Second CP node fails with confusing errors about cluster state mismatch.

**Root cause:** `apply` assumes it's the first node and performs cluster-level operations. Running it twice corrupts the upgrade state machine.

**Fix:** `apply` on **exactly one** CP node. `node` on all others.

---

### Mistake 5: Not Verifying CNI Compatibility Before Upgrading

**Symptom:** After upgrade, pods can't communicate. calico-node or other CNI pods crashloop.

**Root cause:** CNI DaemonSets use CRDs and APIs that may have changed in the new minor. Some CNI versions only support specific Kubernetes minor versions.

**Fix:** Before upgrading, check:
- Calico: https://docs.tigera.io/calico/latest/getting-started/kubernetes/requirements
- Cilium: https://docs.cilium.io/en/stable/network/kubernetes/compatibility-table/
- General rule: upgrade CNI to a version that supports your target K8s minor **before or right after** the control-plane upgrade.

---

### Mistake 6: Draining Without Checking Pod Counts First

**Symptom:** Production traffic drops during drain because only one replica existed.

**Root cause:** Nobody checked what was running on the node before draining it.

**Fix:** Always run this before drain:
```bash
kubectl get pods -A -o wide | grep worker-1
# Review: any single-replica Deployments? Any StatefulSets without PDBs?
```

---

### Mistake 7: Forgetting `--ignore-daemonsets` During Drain

**Symptom:** `kubectl drain` refuses to proceed.

**Root cause:** DaemonSet pods can't be evicted in the normal sense (they'll just respawn). drain requires explicit acknowledgment to skip them.

**Fix:** Always use `--ignore-daemonsets`. DaemonSet pods are cluster infrastructure and should always run.

---

## 7. Interview Answers — Compressed, Verbatim-Ready

**Q: How does Kubernetes versioning work and why can't you skip minor versions?**

"Kubernetes uses semantic versioning with major, minor, and patch components. Minor versions are released roughly every four months and represent significant API and feature changes. You can't skip minor versions because each upgrade includes internal API migrations and storage version changes designed only for adjacent versions — going from 1.31 to 1.33 directly would bypass those migration hooks and can silently corrupt or misinterpret API objects. The supported upgrade path is always sequential, one minor at a time."

---

**Q: Walk me through a production kubeadm cluster upgrade.**

"I start with pre-flight work: snapshot etcd, tar up `/etc/kubernetes`, verify PDBs and readiness probes are in place, and confirm CNI and CRI compatibility with the target version. On the first control-plane node, I point APT to the new minor repo, upgrade kubeadm, run `kubeadm upgrade plan` to review the changes, then `kubeadm upgrade apply v1.33.4`. That updates the static pod manifests and restarts the control-plane components. Then I manually upgrade kubelet and kubectl with apt and restart kubelet. On remaining CP nodes in an HA setup, I run `kubeadm upgrade node` — not apply. For workers, I cordon then drain each one, upgrade kubeadm, run `kubeadm upgrade node`, upgrade kubelet and kubectl, restart kubelet, and uncordon. I do this one node at a time to maintain availability."

---

**Q: What's the difference between `kubectl cordon` and `kubectl drain`?**

"Cordon marks a node as unschedulable, preventing new pods from being placed there, but leaves existing pods running. It's reversible with uncordon and useful when you want to inspect a node before committing to maintenance. Drain does everything cordon does, plus it evicts existing pods using the Eviction API — which means it respects PodDisruptionBudgets. DaemonSet pods are skipped with `--ignore-daemonsets` because they're expected to run everywhere. The workflow I use is always cordon first, verify what's on the node, then drain."

---

**Q: How does a PodDisruptionBudget actually work? How does the API server enforce it?**

"A PDB defines either `minAvailable` or `maxUnavailable` for a set of pods selected by labels. When `kubectl drain` evicts a pod, it uses the Eviction API — it sends a POST to the pod's eviction subresource. The API server intercepts this and calculates `allowedDisruptions` as the difference between currently healthy pods — those in Running state with a Ready condition — and the `minAvailable` floor. If `allowedDisruptions` is greater than zero, eviction is granted. If it's zero, the API returns HTTP 429, drain retries, and the eviction is blocked until a replacement becomes Ready and restores the allowance. The key nuance is that a pod in Terminating state is not counted as Ready, so allowance doesn't restore until the replacement pod is actually serving traffic."

---

**Q: What is version skew and why does it matter?**

"Version skew defines the compatibility boundaries between Kubernetes components during rolling upgrades. The API server is the anchor — it defines the cluster version. kube-controller-manager and kube-scheduler must never be newer than the API server and can be at most one minor behind. Kubelet on worker nodes can be up to two minors behind the API server, which is the rolling window that allows us to upgrade workers after the control plane without immediate downtime. kubectl supports ±1 minor from the API server. The rule of thumb is: nothing can be newer than the API server, and you should converge everything to the same minor as quickly as operationally safe."

---

**Q: What does `kubeadm upgrade apply` actually do?**

"At a high level, it updates the static pod manifests in `/etc/kubernetes/manifests/` for the API server, controller manager, scheduler, and etcd — changing their image tags and any relevant flags. The kubelet, which watches that directory, detects the changes and restarts each component with the new image. It also updates CoreDNS and kube-proxy by patching their DaemonSet and Deployment image tags, and it renews certificates by default. Critically, it does NOT upgrade kubelet or kubectl — those are systemd services managed through the OS package manager."

---

**Q: What's the difference between `minAvailable` and `maxUnavailable` in a PDB?**

"Both achieve the same goal of limiting disruption, but from different directions. `minAvailable: 3` says 'at least 3 pods must be Ready at all times.' `maxUnavailable: 1` says 'at most 1 pod may be unavailable at any time.' For a Deployment with 4 replicas, both expressions allow exactly 1 eviction at a time. The practical difference shows up during scale events: `minAvailable` is an absolute floor and doesn't change as replicas scale, so if you scale down to 3 replicas with `minAvailable: 3`, you've just made the workload undrainable. `maxUnavailable` as a percentage scales proportionally, which is generally safer for variable-replica workloads."

---

## 8. Debugging — Fast Diagnosis Paths

### Upgrade Is Stuck or Failed — Decision Tree

```
kubeadm upgrade apply failed?
├── "etcd cluster is not healthy"
│   ├── Check: etcdctl endpoint health
│   ├── Check: kubectl get pods -n kube-system | grep etcd
│   └── Fix: restore from snapshot, or manually restart etcd pods
│
├── "version skew too large"
│   └── Fix: you tried to skip a version. Upgrade sequentially.
│
├── "connection refused" / API server unreachable
│   ├── Check: sudo crictl ps | grep kube-apiserver
│   ├── Check: sudo journalctl -u kubelet -n 50
│   └── Check: static pod manifest for syntax errors
│
└── "certificate error"
    ├── Check: kubeadm certs check-expiration
    └── Fix: kubeadm certs renew all
```

### Drain Is Blocked — Decision Tree

```
kubectl drain <node> hangs?
├── "Cannot evict pod: would violate disruption budget"
│   ├── Check: kubectl get pdb -A
│   ├── Check: kubectl describe pdb <name>
│   ├── Check: are replacement pods scheduling?
│   │   └── kubectl get pods -A -o wide  (look for Pending)
│   ├── Fix A: scale up deployment to create headroom
│   ├── Fix B: temporarily relax PDB (minAvailable--)
│   └── Fix C: kubectl drain --disable-eviction (bypass PDB, last resort)
│
├── "Pod has local storage that will be lost"
│   └── Fix: add --delete-emptydir-data flag
│
└── "Cannot delete DaemonSet-managed pod"
    └── Fix: add --ignore-daemonsets flag
```

### Node Stuck in NotReady After Upgrade

```bash
# Step 1: Check kubelet status
sudo systemctl status kubelet
sudo journalctl -u kubelet -n 100 --no-pager

# Step 2: Check for config drift
sudo kubeadm upgrade node  # Re-run in case it was skipped

# Step 3: Check CRI is healthy
sudo crictl info
sudo systemctl status containerd

# Step 4: Check CNI
kubectl get pods -n kube-system -o wide | grep <node-name>
kubectl describe node <node-name> | grep -A 5 Conditions

# Step 5: Hard reset (last resort)
sudo systemctl daemon-reload
sudo systemctl restart containerd
sudo systemctl restart kubelet
```

### Control-Plane Component Unhealthy After Upgrade

```bash
# Check static pod logs (these aren't regular pods)
sudo crictl ps -a | grep kube-apiserver
sudo crictl logs <container-id>

# Or via kubectl (if API server is still reachable)
kubectl logs -n kube-system kube-apiserver-<node-name>
kubectl describe pod -n kube-system kube-apiserver-<node-name>

# Check manifest for issues
sudo cat /etc/kubernetes/manifests/kube-apiserver.yaml

# Verify component health
kubectl get componentstatuses  # deprecated but sometimes useful
curl -k https://localhost:6443/readyz
curl -k https://localhost:6443/livez
```

### Version Mismatch Debugging

```bash
# What version is the API server?
kubectl version --short

# What version is each node's kubelet?
kubectl get nodes -o custom-columns='NAME:.metadata.name,VERSION:.status.nodeInfo.kubeletVersion'

# What packages are installed?
dpkg -l | grep -E 'kubeadm|kubelet|kubectl'

# Are any packages held?
apt-mark showhold
```

---

## 9. Kill Switch — 10-Second Recall

```
UPGRADE ORDER:    etcd backup → kubeadm → apply → kubelet/kubectl → workers
NEVER:            skip minors | upgrade workers before CP | run 'apply' twice in HA
SKEW RULES:       nothing newer than apiserver | kubelet ≤ N-2 | kubectl ±1
DRAIN SEQUENCE:   cordon → inspect → drain (--ignore-daemonsets --delete-emptydir-data)
PDB TRUTH:        counts READY pods | terminating ≠ ready | 429 = eviction blocked
kubeadm apply:    FIRST CP node only | does NOT touch kubelet | renews certs by default
kubeadm node:     ALL other nodes (CP + workers) | aligns to already-upgraded state
CALICO VXLAN:     disable BGP if you see BIRD errors | open TCP 5473 for Typha
```

---

## 10. Appendix — Quick Reference

### Upgrade Command Cheatsheet

```bash
# ── REPO ────────────────────────────────────────────────────
sudo vim /etc/apt/sources.list.d/kubernetes.list
# .../v1.33/deb/ /
sudo apt-get update

# ── FIND VERSION ────────────────────────────────────────────
sudo apt-cache madison kubeadm

# ── UPGRADE KUBEADM ─────────────────────────────────────────
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm=1.33.4-1.1
sudo apt-mark hold kubeadm
kubeadm version

# ── PLAN + APPLY (first CP only) ────────────────────────────
sudo kubeadm upgrade plan
sudo kubeadm upgrade diff v1.33.4  # optional: see manifest diffs
sudo kubeadm upgrade apply v1.33.4

# ── ALIGN (other CP nodes + workers) ───────────────────────
sudo kubeadm upgrade node

# ── UPGRADE KUBELET + KUBECTL ───────────────────────────────
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.33.4-1.1 kubectl=1.33.4-1.1
sudo systemctl daemon-reload && sudo systemctl restart kubelet
sudo apt-mark hold kubeadm kubelet kubectl

# ── DRAIN / UNCORDON ────────────────────────────────────────
kubectl cordon <node>
kubectl get pods -A -o wide | grep <node>
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data --grace-period=60 --timeout=10m
kubectl uncordon <node>
kubectl get nodes -o wide

# ── ETCD BACKUP ─────────────────────────────────────────────
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /backup/etcd-$(date +%Y%m%d).db

# ── VERIFY ──────────────────────────────────────────────────
kubectl version
kubectl get nodes -o wide
kubectl get pods -A | grep -v Running
kubeadm certs check-expiration
```

### Version Skew Quick Reference Card

```
kube-apiserver:          THE VERSION. Everything else is relative to this.
controller-manager:      ≤ apiserver, never newer. Prefer same minor.
scheduler:               ≤ apiserver, never newer. Prefer same minor.
kubelet:                 ≤ apiserver. Up to N-2 during rolling upgrade.
kubectl:                 ±1 minor from apiserver. OK to be N+1 on admin machines.
etcd:                    Follow kubeadm's recommendation. Don't diverge.
```

### PDB Formula

```
allowedDisruptions = currentHealthy - minAvailable
                   OR
allowedDisruptions = maxUnavailable - currentlyUnavailable

If allowedDisruptions > 0  →  eviction ALLOWED
If allowedDisruptions == 0 →  429 returned, drain WAITS
```

### Common Upgrade Errors → Quick Fix

| Error Message | Cause | Fix |
|---|---|---|
| `etcd cluster is not healthy` | etcd down/quorum lost | Restore snapshot or restart etcd |
| `version skew too large` | Tried to skip a minor | Upgrade one minor at a time |
| `Cannot evict: disruption budget` | PDB blocks drain | Scale up or relax PDB |
| `connection refused` (API) | API server crashed | Check crictl, journalctl, static manifests |
| `DaemonSet-managed pod` | Forgot flag | Add `--ignore-daemonsets` |
| `local storage will be lost` | Forgot flag | Add `--delete-emptydir-data` |
| Node stuck `NotReady` | kubelet not restarted | `systemctl restart kubelet` |
| Calico pods `0/1 Running` | BGP enabled on VXLAN | Patch Installation CR, disable BGP |

### Kubernetes Release Calendar Reference

| Event | Frequency | Note |
|---|---|---|
| Minor release | ~4 months | ~3 per year |
| Patch release | ~Monthly | Faster (1-2 weeks) just after a new minor |
| EOL of a minor | N-2 boundary | When N+1 releases, N-2 becomes unsupported |
| Cert rotation | 1 year by default | kubeadm renews on upgrade automatically |