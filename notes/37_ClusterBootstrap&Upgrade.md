# 🚀 Kubernetes Cluster Bootstrap & Upgrade — CKA Master Guide
> *kubeadm + Calico + Zero-Downtime Upgrades, from first principles to production scars*

---

## 0. First Principles — The Mental Model That Never Changes

> Before a single command runs, internalize these truths. Everything else is just detail.

**The cluster is a distributed system.** Nodes are unreliable. Networks partition. Components restart. Kubernetes is designed to tolerate all of that — but only if *you* set it up correctly.

**kubeadm is a bootstrapper, not a babysitter.** It gets you to a working cluster. It won't auto-heal, auto-upgrade, or auto-configure your firewall. That's your job.

**Order matters. Always.** Control plane before workers. One minor version at a time. Cordon before drain. These aren't preferences — they're invariants.

**The CNI is not optional.** Without a CNI (like Calico), pods can't talk. CoreDNS won't start. Your cluster looks alive but is functionally dead.

**Version skew is a real constraint.** No component can be *newer* than the API server. kubelet can lag by two minors. kubectl by one. Violate this and things break in subtle, painful ways.

| Mental Model | What It Means in Practice |
|---|---|
| "Control plane is the brain" | Upgrade it first; workers can tolerate brief divergence |
| "Workers are cattle" | Drain → upgrade → uncordon, one at a time |
| "CNI owns pod networking" | Wrong firewall ports = `Running (0/1)` forever |
| "kubeadm manages static pods" | Upgrade = rewrite manifest files, kubelet restarts them |
| "PDBs are promises, not locks" | Drain will stall if PDB can't be satisfied — that's correct behavior |

---

## 1. Reality Constraints — What Kubernetes Actually Does (and Doesn't)

### What kubeadm does ✅
- Generates PKI (certs + kubeconfig files)
- Writes static pod manifests for control-plane components
- Bootstraps etcd (stacked topology)
- Creates the bootstrap token for worker joins
- On upgrade: rewrites static pod manifests with new image tags

### What kubeadm does NOT do ❌
- Configure your firewall / security groups
- Install or upgrade your CNI
- Manage your container runtime (containerd)
- Handle HA load balancer setup
- Back up etcd (you do that)

### Hard constraints you cannot ignore

| Constraint | Rule | Consequence of Violation |
|---|---|---|
| Swap | Must be OFF | kubelet refuses to start |
| CPU (control plane) | ≥ 2 vCPU | Preflight error (bypassable but unstable) |
| Skipping minor versions | Never skip (1.31→1.33 invalid) | Unsupported, likely broken |
| Component version skew | No component newer than API server | Silent failures, API incompatibilities |
| CNI pod CIDR match | Must match `--pod-network-cidr` | Pods get wrong IPs or can't communicate |
| SystemdCgroup alignment | containerd and kubelet must agree | cgroup errors, pod evictions |

---

## 2. Decision Logic — When to Use What

### Which networking mode for Calico?

```
Are you on a flat L2 network (all nodes same subnet)?
  ├── Yes → Use VXLANCrossSubnet (encapsulates only cross-subnet traffic)
  └── No  → Use VXLAN (always encapsulate)

Do you have a routed fabric (BGP-capable switches)?
  └── Yes → Use BGP (no overlay, pure L3 routing)
       └── But: open TCP 179 between nodes, configure route reflectors

Are you on a cloud provider with overlay restrictions?
  └── Check provider docs; VXLAN is generally safest default
```

### Which upgrade strategy?

```
Is this dev/test with no SLO?
  └── All at once (acceptable risk)

Is this production?
  ├── Can you afford node downtime one at a time?
  │   └── Yes → Rolling upgrade (best practice)
  └── Tight SLO or risky runtime/OS change?
      └── Blue/Green (spin up new nodes, shift traffic, retire old)
```

### Cordon vs Drain — when to use each

| Action | What It Does | When to Use |
|---|---|---|
| `kubectl cordon` | Marks node unschedulable (new pods won't land) | First step — always |
| `kubectl drain` | Evicts existing pods (respects PDBs), then cordons | When you're ready to actually move workloads |
| `kubectl uncordon` | Marks node schedulable again | After upgrade is verified |

> **Pro tip:** Always cordon *before* drain. It gives you a window to inspect what's running, check PDBs, and back out cleanly with `uncordon` if needed.

### Version skew rules at a glance

```
kube-apiserver (N)
  ├── kube-controller-manager: N or N-1
  ├── kube-scheduler:          N or N-1
  ├── kubelet:                 N, N-1, or N-2
  └── kubectl:                 N-1, N, or N+1
```

---

## 3. Internal Working — How It Actually Happens Under the Hood

### Cluster Bootstrap (kubeadm init)

```
1. Preflight checks
   └── CPU, RAM, swap, kernel modules, container runtime reachable?

2. Certificate generation
   └── /etc/kubernetes/pki/
       ├── ca.crt / ca.key           ← cluster root CA
       ├── apiserver.crt             ← API server TLS cert
       ├── etcd/ca.crt               ← etcd-specific CA
       └── sa.pub / sa.key           ← service account signing keys

3. Kubeconfig generation
   └── /etc/kubernetes/
       ├── admin.conf                ← cluster-admin (you use this)
       ├── kubelet.conf              ← kubelet identity
       ├── controller-manager.conf
       └── scheduler.conf

4. Static pod manifest generation
   └── /etc/kubernetes/manifests/
       ├── etcd.yaml
       ├── kube-apiserver.yaml
       ├── kube-controller-manager.yaml
       └── kube-scheduler.yaml
       (kubelet watches this dir and starts/restarts these pods automatically)

5. Bootstrap token created
   └── Stored as a Secret in kube-system

6. Cluster info written to ConfigMap
   └── kube-public/cluster-info (used by joining nodes)
```

### Worker Join (kubeadm join)

```
1. Worker contacts API server using bootstrap token
2. Downloads cluster CA cert (verified via --discovery-token-ca-cert-hash)
3. Gets a unique TLS client cert issued by the cluster CA
4. kubelet writes /var/lib/kubelet/config.yaml and starts
5. Node registers itself via POST to /api/v1/nodes
6. CNI assigns the node a pod CIDR block
7. Node transitions: NotReady → Ready (after CNI is working)
```

### What happens during `kubeadm upgrade apply`

```
1. Validates current cluster state
2. Pulls new control-plane images (or uses cache)
3. Rewrites static pod manifest files with new image tags
   └── /etc/kubernetes/manifests/*.yaml (image tags bumped)
4. kubelet detects manifest changes, kills old pods, starts new ones
5. Renews certificates (unless --certificate-renewal=false)
6. Updates kube-proxy DaemonSet + CoreDNS Deployment
7. Prints: "SUCCESS! A control plane node was upgraded to vX.Y.Z"
   └── kubelet on THIS node is NOT upgraded yet (Step 7 does that)
```

### How Calico Typha works

```
calico-node (Felix) on each node
    │
    │  TCP 5473
    ▼
calico-typha (aggregates datastore updates)
    │
    │  Kubernetes API
    ▼
etcd (via kube-apiserver)
```

> Without TCP 5473 open between nodes, Felix can't reach Typha → `calico-node` stays at `Running (0/1)` forever. This is the #1 Calico gotcha on cloud clusters.

---

## 4. Hands-On — Production-Quality YAML + Commands

### Step 1: Disable Swap & Kernel Networking `[ALL NODES]`

```bash
# Disable swap immediately + persist across reboots
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Load required kernel modules
echo -e "overlay\nbr_netfilter" | sudo tee /etc/modules-load.d/k8s.conf
sudo modprobe overlay && sudo modprobe br_netfilter

# Kernel sysctls for container networking
cat <<'EOF' | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
EOF
sudo sysctl --system
```

**Why each piece matters:**
- `swapoff` → kubelet assumes no swap for resource accounting
- `overlay` → union filesystem for container image layers
- `br_netfilter` → lets iptables see bridged pod traffic (Services won't work without it)
- `ip_forward=1` → allows the node to route packets between pods and the outside world

---

### Step 2: Install & Configure containerd `[ALL NODES]`

```bash
sudo apt-get update && sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null

# Critical: align cgroup driver with kubelet
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# Pin pause image version (prevents unnecessary pod sandbox churn)
sudo sed -i 's#sandbox_image = ".*"#sandbox_image = "registry.k8s.io/pause:3.9"#' \
  /etc/containerd/config.toml

sudo systemctl daemon-reload
sudo systemctl enable --now containerd
sudo systemctl status containerd
```

---

### Step 3: Install kubeadm, kubelet, kubectl `[ALL NODES]`

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update && sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl   # prevents accidental upgrades
sudo systemctl enable kubelet

# Verify
kubeadm version && kubelet --version && kubectl version --client
```

**Optional but highly recommended — bash completion:**

```bash
sudo apt-get install -y bash-completion
echo 'source /usr/share/bash-completion/bash_completion' >> ~/.bashrc
echo 'source <(kubectl completion bash)' >> ~/.bashrc
echo 'alias k=kubectl' >> ~/.bashrc
echo 'complete -F __start_kubectl k' >> ~/.bashrc
source ~/.bashrc
```

---

### Step 4: Initialize the Control Plane `[CONTROL PLANE ONLY]`

```bash
sudo kubeadm init \
  --control-plane-endpoint=<CP_PRIVATE_IP>:6443 \
  --apiserver-advertise-address=<CP_PRIVATE_IP> \
  --pod-network-cidr=192.168.0.0/16
```

```bash
# Set up kubeconfig for your user
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

> **1 vCPU bypass** (lab only — not recommended for stability):
> ```bash
> sudo kubeadm init ... --ignore-preflight-errors=NumCPU
> ```

---

### Step 5: Install Calico CNI via Operator `[CONTROL PLANE ONLY]`

```bash
# Install CRDs and the operator
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.2/manifests/operator-crds.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.2/manifests/tigera-operator.yaml
```

```yaml
# Apply the Installation CR — this is the key configuration object
cat <<'EOF' | kubectl apply -f -
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default          # Must be 'default' — this is a singleton resource
spec:
  calicoNetwork:
    bgp: Disabled        # Explicitly off for VXLAN-only clusters (avoids BIRD errors)
    ipPools:
    - cidr: 192.168.0.0/16          # Must match --pod-network-cidr from kubeadm init
      natOutgoing: Enabled           # SNAT pod traffic leaving the cluster
      blockSize: 26                  # /26 = 64 pod IPs per node
      encapsulation: VXLANCrossSubnet  # VXLAN where needed, native where not
      nodeSelector: all()
EOF
```

---

### Step 6: Join the Workers `[WORKER ONLY]`

```bash
# On control plane — generate a fresh join command
kubeadm token create --print-join-command
```

```bash
# Run the printed command on EACH worker (yours will have real values)
sudo kubeadm join <CP_PRIVATE_IP>:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

---

### Step 7: Verify `[Any node with kubeconfig]`

```bash
kubectl get nodes -o wide
kubectl -n kube-system get pods -o wide

# Deploy a quick smoke test
kubectl create deploy web --image=nginx --replicas=3
kubectl expose deploy/web --port=80 --type=NodePort
kubectl get svc web -o jsonpath='{.spec.ports[0].nodePort}'; echo
curl -I http://<worker-public-ip>:<nodeport>
```

---

### PodDisruptionBudget YAML (for upgrade safety)

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: red-deploy-pdb
spec:
  minAvailable: 3      # At least 3 pods must be Ready at all times
  # maxUnavailable: 1  # Alternative: allow at most 1 pod down at once
  selector:
    matchLabels:
      app: red-deploy
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: red-deploy
spec:
  replicas: 4
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
        image: nginx
```

---

## 5. Production Flow — Real-World Architecture & Design Patterns

### Full upgrade flow (1.32 → 1.33)

```
┌─────────────────────────────────────────────────────────────────┐
│  PRE-CHECKS                                                     │
│  ├── etcd snapshot: ETCDCTL_API=3 etcdctl snapshot save ...     │
│  ├── Tarball PKI:   tar czf k8s-pki.tar.gz /etc/kubernetes/     │
│  ├── Verify PDBs and readiness probes on critical workloads     │
│  └── Ensure capacity headroom (or scale up temporarily)         │
└─────────────────────┬───────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────────┐
│  CONTROL PLANE (one node; for HA do remaining CPs after)        │
│  1. Update APT repo: v1.32 → v1.33                              │
│  2. apt-mark unhold kubeadm && install kubeadm=1.33.4-1.1       │
│  3. kubeadm upgrade plan       (dry-run, shows all targets)     │
│  4. kubeadm upgrade apply v1.33.4  (rewrites manifests)         │
│  5. apt install kubelet=1.33.4-1.1 kubectl=1.33.4-1.1           │
│  6. systemctl restart kubelet                                   │
│  7. apt-mark hold kubeadm kubelet kubectl                       │
│  8. kubectl get nodes  → CP shows v1.33.4 ✓                     │
└─────────────────────┬───────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────────┐
│  WORKERS (one at a time, repeat for each)                       │
│  [Admin shell]  kubectl cordon worker-1                         │
│  [Admin shell]  kubectl drain worker-1 \                        │
│                   --ignore-daemonsets \                         │
│                   --delete-emptydir-data                        │
│  [worker-1]     Update APT repo → v1.33                         │
│  [worker-1]     apt install kubeadm=1.33.4-1.1                  │
│  [worker-1]     kubeadm upgrade node                            │
│  [worker-1]     apt install kubelet=1.33.4-1.1 kubectl=1.33.4-1.1│
│  [worker-1]     systemctl daemon-reload && restart kubelet      │
│  [Admin shell]  kubectl uncordon worker-1                       │
│  [Admin shell]  kubectl get nodes → worker-1 shows v1.33.4 ✓   │
└─────────────────────┬───────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────────┐
│  POST-VALIDATION                                                │
│  ├── kubectl version (client + server both v1.33.4)             │
│  ├── kubectl get nodes (all Ready, all v1.33.4)                 │
│  ├── kubectl -n kube-system get pods (CoreDNS, kube-proxy OK)   │
│  ├── kubectl -n calico-system get pods (all 1/1 Running)        │
│  └── Run synthetic traffic probe / smoke test                   │
└─────────────────────────────────────────────────────────────────┘
```

### Drain + PDB interaction (the diagram explained)

When you drain `worker-1` with `minAvailable: 3` and 4 replicas:

```
worker-1 has 2 red pods, worker-2 has 2 red pods
Total Ready: 4 ✓ (≥3, PDB satisfied)

Drain starts → evict first pod from worker-1
  Total Ready: 3 ✓ (still ≥3, PDB satisfied → eviction allowed)
  Replacement schedules on worker-2...
  Replacement becomes Ready → Total: 4 ✓
  Next eviction now allowed → repeat

If worker-2 has NO room:
  Evict first pod → Ready: 3 ✓
  Replacement can't schedule → stays Pending
  Second eviction attempt → Ready would drop to 2 ✗ (PDB BLOCKS)
  Drain stalls → you must: add capacity, scale replicas, or relax PDB
```

### Security Group reference (AWS / any cloud)

**Control Plane SG (inbound):**

| Purpose | Protocol/Port | Source |
|---|---|---|
| SSH | TCP 22 | Your IP |
| Kubernetes API | TCP 6443 | data-plane-sg |
| Calico Typha | TCP 5473 | data-plane-sg |
| VXLAN overlay | UDP 4789 | data-plane-sg |

**Worker SG (inbound):**

| Purpose | Protocol/Port | Source |
|---|---|---|
| SSH | TCP 22 | Your IP |
| Kubelet API | TCP 10250 | control-plane-sg |
| VXLAN overlay | UDP 4789 | data-plane-sg + control-plane-sg |
| Calico Typha | TCP 5473 | data-plane-sg + control-plane-sg |
| NodePort (optional) | TCP 30000-32767 | Your IP |

---

## 6. Mistakes — What Actually Breaks in Real Systems

### 💥 Mistake 1: `calico-node` stuck at `Running (0/1)` forever

**Symptom:** `kubectl -n calico-system get pods` shows calico-node as `0/1`. Logs show `"Error querying BIRD"` or `"BGP not established"`.

**Root cause:** Calico operator defaults BGP to *enabled*. In a VXLAN-only cluster, BIRD (the BGP daemon) can't establish sessions → readiness probe fails indefinitely.

**Fix:**
```bash
kubectl patch installation.operator.tigera.io default --type=merge \
  -p '{"spec":{"calicoNetwork":{"bgp":"Disabled"}}}'
kubectl -n calico-system rollout restart ds/calico-node
kubectl -n calico-system rollout status ds/calico-node
```

---

### 💥 Mistake 2: Felix can't reach Typha (TCP 5473 blocked)

**Symptom:** After disabling BGP, some calico-node pods still fail readiness. Logs mention Typha connection timeouts.

**Root cause:** Calico's Typha component (runs on nodes with hostNetwork) listens on TCP 5473. If your firewall blocks this between nodes, Felix can't sync.

**Fix:** Open TCP 5473 between all node security groups. In AWS, add the rule to both control-plane-sg and data-plane-sg.

---

### 💥 Mistake 3: Skipping minor versions during upgrade

**Symptom:** `kubeadm upgrade apply` fails or cluster behaves erratically after upgrade.

**Root cause:** Jumping from 1.31 → 1.33 is unsupported. The upgrade path must be sequential: 1.31 → 1.32 → 1.33.

**Fix:** Always check current version, plan one minor at a time.

---

### 💥 Mistake 4: Upgrading workers before control plane

**Symptom:** kubelet on worker is newer than API server. API calls start failing intermittently.

**Root cause:** Version skew violation. kubelet must never be newer than kube-apiserver.

**Fix:** Always upgrade control plane first, workers second.

---

### 💥 Mistake 5: `SystemdCgroup = false` in containerd

**Symptom:** Pods start but get evicted. cgroup-related errors in kubelet logs.

**Root cause:** containerd and kubelet must use the same cgroup driver. containerd ships with `SystemdCgroup = false` by default.

**Fix:**
```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
```

---

### 💥 Mistake 6: Drain blocks indefinitely due to PDB

**Symptom:** `kubectl drain` hangs. You see `"Cannot evict pod as it would violate the pod's disruption budget"`.

**Root cause:** PDB `minAvailable` can't be satisfied — usually because a replacement pod is Pending (insufficient capacity).

**Fix options:**
```bash
# Option 1: Add a node or scale down replicas temporarily
kubectl scale deploy red-deploy --replicas=5   # create headroom

# Option 2: Relax the PDB temporarily (not recommended in prod)
kubectl patch pdb red-deploy-pdb -p '{"spec":{"minAvailable":2}}'

# Option 3: Force drain (ONLY if you understand the risk)
kubectl drain worker-1 --ignore-daemonsets --delete-emptydir-data --force
# WARNING: --force deletes pods not managed by a controller (data loss risk)
```

---

### 💥 Mistake 7: Forgetting to `apt-mark hold` after upgrade

**Symptom:** `apt-get upgrade` pulls a newer kubeadm/kubelet and breaks version pinning.

**Fix:** Always re-hold after every planned upgrade:
```bash
sudo apt-mark hold kubeadm kubelet kubectl
```

---

## 7. Interview Answers — Compressed, Verbatim-Ready

**Q: Walk me through bootstrapping a Kubernetes cluster with kubeadm.**

> "First, I prepare all nodes by disabling swap, loading the overlay and br_netfilter kernel modules, and setting the bridge sysctls for iptables visibility. Then I install containerd and set SystemdCgroup to true so it aligns with kubelet's cgroup driver. I install kubeadm, kubelet, and kubectl from the official Kubernetes repo and hold them to prevent drift. On the control plane, I run kubeadm init with the pod network CIDR and the control plane endpoint. That generates the PKI, writes static pod manifests for etcd, API server, controller manager, and scheduler, and creates a bootstrap token. I set up kubeconfig, install Calico via the Tigera operator, then join each worker using the token printed by kubeadm token create --print-join-command."

---

**Q: What is the upgrade order and why does it matter?**

> "Control plane always goes first, then workers. This is because kubelet on workers can tolerate being up to two minor versions behind the API server, but nothing can ever be *newer* than the API server. If you upgraded workers first, you'd violate version skew and risk API compatibility failures. Within the control plane in an HA setup, you run kubeadm upgrade apply on exactly one node, then kubeadm upgrade node on the remaining control plane nodes, one at a time to preserve etcd quorum."

---

**Q: What does `kubeadm upgrade apply` actually do?**

> "It rewrites the static pod manifest files in /etc/kubernetes/manifests with updated image tags for the control plane components — the API server, controller manager, and scheduler. kubelet is watching that directory, so it automatically kills the old pods and starts new ones with the new images. It also updates kube-proxy as a DaemonSet and CoreDNS as a Deployment. Crucially, it does NOT upgrade kubelet itself — that's a separate apt install step because kubelet isn't a static pod; it's a systemd service."

---

**Q: What is a PodDisruptionBudget and how does it interact with drain?**

> "A PDB is a policy object that caps voluntary disruptions to a workload. You can express it as minAvailable — meaning at least N pods must be Ready at all times — or maxUnavailable — meaning at most N pods may be unavailable. When you run kubectl drain, it uses the Eviction API, which honors PDBs. If evicting a pod would violate the budget, the drain stalls and retries until capacity is restored. This means a PDB with minAvailable:3 on a 4-replica Deployment will let you evict one pod per round, but only after a replacement becomes Ready on another node. PDB counts Ready pods — Pending replacements don't count."

---

**Q: Why is BGP disabled in a VXLAN-only Calico cluster?**

> "Calico's BGP daemon, BIRD, is enabled by default in operator-based installs. In a VXLAN cluster you don't need BGP — pod traffic is encapsulated in UDP packets using VXLAN rather than advertised via routing protocols. When BGP is enabled but can't establish sessions — which happens if TCP 179 isn't open or there's no BGP peer — BIRD's health check fails, and the calico-node readiness probe fails, leaving the pod stuck at 0/1. Disabling BGP explicitly on the Installation CR removes BIRD from the equation entirely."

---

**Q: What is Calico Typha and why does TCP 5473 matter?**

> "Typha is an aggregation proxy that sits between the Kubernetes datastore and the Felix agents running on each node. Instead of every Felix instance watching the API server directly — which doesn't scale — they all connect to Typha, which fans out updates. Typha listens on TCP 5473 and typically runs with hostNetwork, so it uses the node's IP. If your firewall blocks 5473 between nodes, Felix can't connect to Typha, and calico-node pods fail their readiness probe. This is the second most common Calico issue after BGP not being disabled."

---

**Q: What's the supported version skew between components?**

> "The API server defines the cluster version. The controller manager and scheduler must not be newer than the API server and can be at most one minor older. kubelet on nodes must not be newer than the API server and can be up to two minors older — this is what allows you to have workers still on 1.32 while the control plane is already on 1.33 during a rolling upgrade. kubectl is the most flexible — it supports plus or minus one minor from the API server."

---

## 8. Debugging — Fast Diagnosis Paths

### Node stuck in NotReady

```
kubectl get nodes
  └── Node shows NotReady

kubectl describe node <node-name>
  └── Look at: Conditions, Events, Taints

# On the node itself
sudo systemctl status kubelet
journalctl -u kubelet -n 50 --no-pager

# Common causes:
  ├── CNI not installed → "network plugin not initialized"
  ├── containerd down → "failed to create sandbox"
  ├── Swap still on → "swap is enabled" in preflight
  └── Certificate expired → TLS errors in kubelet logs
```

---

### calico-node at `0/1` (the classic)

```
kubectl -n calico-system get pods
  └── calico-node-xxxxx  0/1  Running

kubectl -n calico-system logs ds/calico-node -c calico-node | tail -30
  ├── "Error querying BIRD" → BGP enabled but failing
  │     Fix: kubectl patch installation.operator.tigera.io default \
  │            --type=merge -p '{"spec":{"calicoNetwork":{"bgp":"Disabled"}}}'
  │          kubectl -n calico-system rollout restart ds/calico-node
  │
  └── "Typha connection failed" / timeout on 5473
        Fix: Open TCP 5473 in firewall between all nodes
             kubectl -n calico-system get deploy,svc,ep -l k8s-app=calico-typha
```

---

### Drain is stuck

```
kubectl drain worker-1 --ignore-daemonsets --delete-emptydir-data
  └── Hangs / "Cannot evict pod as it would violate PDB"

kubectl get pdb -A
  └── Which PDB, what are the current counts?

kubectl get pods -A -o wide | grep worker-1
  └── What's still running there?

kubectl get pods -A | grep Pending
  └── Is a replacement failing to schedule? (resource pressure? taints?)

kubectl describe pod <pending-pod>
  └── Events: "Insufficient cpu/memory" → need more capacity
              "Unschedulable" → check taints/tolerations
```

---

### Upgrade health check sequence

```bash
# After control plane upgrade
kubectl version                          # Server should be new version
kubectl get nodes                        # CP shows new version, workers still old
kubectl -n kube-system get pods          # CoreDNS and kube-proxy should be new version

# After worker upgrade
kubectl get nodes -o wide                # All nodes new version
kubectl -n calico-system get pods        # All 1/1 Running
kubectl get pods -A                      # No CrashLoopBackOff, no Pending

# After full upgrade
kubectl run test --image=nginx --restart=Never
kubectl exec test -- curl -s localhost    # Should return nginx HTML
kubectl delete pod test
```

---

### cert expiry check

```bash
# Check when certs expire (run on control plane)
sudo kubeadm certs check-expiration

# Renew all certs (if within 30 days or already expired)
sudo kubeadm certs renew all
sudo systemctl restart kubelet
```

---

## 9. Kill Switch — 10-Second Recall

```
BOOTSTRAP:
  swapoff → containerd (SystemdCgroup=true) → kubeadm/kubelet/kubectl
  → kubeadm init → kubeconfig → Calico CNI → kubeadm join

CALICO GOTCHAS:
  BGP enabled by default → disable it for VXLAN
  TCP 5473 = Typha port → must be open between ALL nodes
  UDP 4789 = VXLAN → must be open between ALL nodes

UPGRADE ORDER:
  Never skip minor versions
  Control plane FIRST, workers SECOND
  One node at a time

VERSION SKEW:
  Nothing newer than API server
  kubelet ≤ N-2, kubectl ≤ N±1

DRAIN FLOW:
  cordon → drain (honors PDB) → upgrade → uncordon

kubeadm upgrade apply = rewrites static pod manifests (NOT kubelet)
kubelet must be upgraded manually with apt + systemctl restart
```

---

## 10. Appendix — Quick Reference Cheatsheet

### Essential kubeadm commands

```bash
# Bootstrap
sudo kubeadm init --control-plane-endpoint=<IP>:6443 \
  --apiserver-advertise-address=<IP> --pod-network-cidr=192.168.0.0/16

# Get join command (run on control plane)
kubeadm token create --print-join-command

# Join worker
sudo kubeadm join <IP>:6443 --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>

# Upgrade dry-run (like terraform plan)
sudo kubeadm upgrade plan

# Apply upgrade (control plane only, first CP node)
sudo kubeadm upgrade apply v1.33.4

# Upgrade remaining CP nodes or workers
sudo kubeadm upgrade node

# Check cert expiry
sudo kubeadm certs check-expiration

# Renew all certs
sudo kubeadm certs renew all

# Full reset (careful — destroys cluster state)
sudo kubeadm reset -f
sudo rm -rf /etc/kubernetes /var/lib/etcd /etc/cni/net.d /var/lib/cni ~/.kube
```

---

### Node maintenance commands

```bash
kubectl cordon <node>                                          # Stop new pods
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data  # Evict pods
kubectl uncordon <node>                                        # Re-enable scheduling
kubectl get nodes -o wide                                      # Status overview
kubectl describe node <node>                                   # Conditions + events
```

---

### APT package management for Kubernetes

```bash
# Unhold (allow upgrade)
sudo apt-mark unhold kubeadm kubelet kubectl

# Install exact version
sudo apt-get update && sudo apt-get install -y kubeadm=1.33.4-1.1

# Re-hold after upgrade
sudo apt-mark hold kubeadm kubelet kubectl

# See available versions
sudo apt-cache madison kubeadm

# Change repo to new minor
sudo vim /etc/apt/sources.list.d/kubernetes.list
# Line should read: deb [...] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /
```

---

### Calico operator commands

```bash
# Check BGP status
kubectl get installation.operator.tigera.io default \
  -o jsonpath='{.spec.calicoNetwork.bgp}{"\n"}'

# Disable BGP
kubectl patch installation.operator.tigera.io default --type=merge \
  -p '{"spec":{"calicoNetwork":{"bgp":"Disabled"}}}'

# Restart calico-node DaemonSet
kubectl -n calico-system rollout restart ds/calico-node
kubectl -n calico-system rollout status ds/calico-node

# Check Typha
kubectl -n calico-system get deploy,svc,endpoints -l k8s-app=calico-typha

# Check calico-node logs for Typha errors
kubectl -n calico-system logs ds/calico-node -c calico-node | grep -i typha
```

---

### Kubernetes release cadence (memorize this)

| Item | Rule |
|---|---|
| New minor release | Every ~4 months |
| Supported versions | N, N-1, N-2 only |
| Support window | ~12 months per minor |
| Amazon EKS | Up to ~26 months (with extended support) |
| Upgrade path | One minor at a time, no skipping |
| Patch releases | ~Monthly; faster right after a new minor |

---

### Port reference card

| Port | Protocol | Component | Direction |
|---|---|---|---|
| 6443 | TCP | kube-apiserver | Workers → Control Plane |
| 10250 | TCP | kubelet | Control Plane → Workers |
| 2379–2380 | TCP | etcd | Control Plane ↔ Control Plane (HA) |
| 10257 | TCP | kube-controller-manager | Internal |
| 10259 | TCP | kube-scheduler | Internal |
| 4789 | UDP | VXLAN (Calico) | All nodes ↔ All nodes |
| 5473 | TCP | Calico Typha | All nodes → Typha pods |
| 179 | TCP | BGP (Calico) | All nodes ↔ (only if BGP enabled) |
| 30000–32767 | TCP | NodePort | External → Workers |

---
