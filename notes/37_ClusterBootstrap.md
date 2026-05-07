# ☸️ Kubernetes Cluster Bootstrap with kubeadm (v1.32)


> **Scope:** One control-plane node, two workers, Ubuntu 22/24, containerd, Calico (VXLAN, operator-managed).
> Every section builds on the last — read top to bottom the first time, then jump by number.

---

## 0 · First Principles 🧠

> *The mental model that never changes, no matter the version.*

| Principle | What it means in practice |
|---|---|
| **Kubernetes doesn't network itself** | After `kubeadm init`, pods can't talk to each other until a CNI plugin fills that gap. The cluster waits — silently. |
| **kubelet owns the node** | Every pod, including control-plane pods, is started and watched by kubelet. It's the one process you can't containerize. |
| **kubeadm is a bootstrapper, not an operator** | It wires up TLS, etcd, and the static pod manifests exactly once. Day-2 ops (upgrades, cert rotation) are still your job. |
| **cgroup driver must match end-to-end** | containerd → kubelet → kernel: all three must agree on `systemd`. Mismatch = silent OOM kills and node NotReady. |
| **Swap breaks resource guarantees** | Kubernetes QoS classes assume memory is finite and physical. Swap lies about that. kubelet refuses to start with swap unless explicitly told otherwise. |
| **Pod CIDR is a contract** | Whatever you pass to `--pod-network-cidr` must exactly match what you tell your CNI. Break this contract and pod-to-pod routing silently fails. |
| **Every node needs a container runtime** | kubeadm doesn't install containerd/docker for you. It expects a CRI socket to already exist. |

---

## 1 · Reality Constraints 🔒

> *What Kubernetes actually does and doesn't do. These are the edges that bite.*

### What kubeadm does NOT do
- Install containerd, Docker, or any CRI
- Install a CNI plugin (CoreDNS stays Pending until you do)
- Configure firewall rules or kernel modules
- Set up HA etcd (single `kubeadm init` = single etcd)
- Manage certificates beyond the initial 1-year issuance

### Immutable facts about this stack

```
kubeadm init  →  writes static pod manifests to /etc/kubernetes/manifests/
kubelet       →  watches that dir and starts API server, etcd, scheduler, controller-manager
containerd    →  runs all of the above as containers (pause + component)
Calico        →  provides veth pairs, routes, and iptables/eBPF rules for pod networking
```

### Hard limits you'll hit

| Constraint | Default value | Why it matters |
|---|---|---|
| Control-plane min CPU | **2 vCPU** | kubeadm preflight fails below this |
| Token TTL | **24 hours** | `kubeadm join` tokens expire; regenerate with `kubeadm token create` |
| Certificate validity | **1 year** | Certs auto-renew on upgrade; otherwise `kubeadm certs renew` |
| etcd data dir | `/var/lib/etcd` | Loss = full cluster state loss. Backup before anything destructive. |
| Pod CIDR overlap | Must not overlap node CIDR or service CIDR | Silent routing hell if it does |
| Calico BGP + VXLAN | Mutually exclusive intent | BGP enabled with VXLAN encapsulation = BIRD readiness failures |

### Port matrix (what must be open — no exceptions)

#### Control-plane node inbound

| Component | Port/Protocol | Source |
|---|---|---|
| Kubernetes API server | TCP 6443 | Workers + your machine |
| Calico Typha | TCP 5473 | Workers (and CP itself if DaemonSet runs there) |
| VXLAN overlay | UDP 4789 | Workers |
| SSH | TCP 22 | Your IP |
| etcd peer (HA only) | TCP 2379–2380 | Other CP nodes |

#### Worker node inbound

| Component | Port/Protocol | Source |
|---|---|---|
| kubelet API | TCP 10250 | Control plane |
| VXLAN overlay | UDP 4789 | Control plane + other workers |
| Calico Typha | TCP 5473 | Other workers + CP |
| NodePort range | TCP 30000–32767 | External clients (testing only) |
| SSH | TCP 22 | Your IP |

> ⚠️ **Miss TCP 5473?** `calico-node` DaemonSet runs on control-plane nodes by default (it tolerates the control-plane taint). So TCP 5473 must be open on the *control-plane SG too*.

---

## 2 · Decision Logic 🗺️

> *When to use what — no guessing required.*

### Should I use kubeadm for this?

```
Need a real, production-grade cluster? ──────────────────► Yes → kubeadm
Need a quick local dev loop? ──────────────────────────── Yes → kind / minikube
Need a managed control plane? ─────────────────────────── Yes → EKS / GKE / AKS
CKA exam environment? ─────────────────────────────────── Always kubeadm
```

### Which CNI? (Calico vs others)

| CNI | Encapsulation | BGP needed | Typha | Best for |
|---|---|---|---|---|
| **Calico (VXLAN)** | VXLAN overlay | ❌ No | ✅ Yes | Multi-cloud, easy setup |
| **Calico (BGP)** | None (native routing) | ✅ Yes | ✅ Yes | Bare-metal, same L2 fabric |
| **Flannel** | VXLAN | ❌ No | ❌ No | Simplicity, no NetworkPolicy |
| **Cilium** | eBPF (optional) | Optional | ❌ No | Observability, L7 policy |
| **WeaveNet** | Encrypted mesh | ❌ No | ❌ No | Simplicity + encryption |

> **This guide uses Calico VXLAN** — best default for cloud VMs on different subnets.

### BGP enabled or disabled?

```
Are all nodes in the same L2 broadcast domain AND you want native routing?
  → Keep BGP Enabled, open TCP 179, configure node-to-node mesh

Are you running on cloud VMs (different subnets) OR just want it to work?
  → Disable BGP, use VXLAN (UDP 4789), close TCP 179
  → This guide's path ✅
```

### Which encapsulation mode?

| Mode | When | Overhead |
|---|---|---|
| `VXLAN` | Nodes on different L3 subnets | ~50 bytes/packet |
| `VXLANCrossSubnet` | Mix of same and cross-subnet nodes | Minimal within-subnet |
| `IPIPCrossSubnet` | Linux-only, prefer VXLAN for clouds | Similar to VXLAN |
| `None` | Same L2 + BGP peering | Zero |

---

## 3 · Internal Working ⚙️

> *What actually happens under the hood, step by step.*

### Phase 1 — `kubeadm init` internals

```
1. Preflight checks
   ├── OS: swap off, kernel modules loaded, ports free
   ├── Runtime: CRI socket reachable (containerd via /run/containerd/containerd.sock)
   └── Resources: ≥2 CPU, ≥1700MB RAM

2. Certificate Authority generation
   ├── /etc/kubernetes/pki/ca.{crt,key}         — cluster CA
   ├── /etc/kubernetes/pki/etcd/ca.{crt,key}    — etcd CA
   └── /etc/kubernetes/pki/front-proxy-ca.{crt,key}

3. Static pod manifests written to /etc/kubernetes/manifests/
   ├── kube-apiserver.yaml
   ├── kube-controller-manager.yaml
   ├── kube-scheduler.yaml
   └── etcd.yaml

4. kubelet (already running) detects new manifests → starts containers via containerd

5. Bootstrap token created → stored in kube-system ConfigMap

6. RBAC rules written to authorize kubeadm join workflow

7. kubeconfig files written:
   ├── /etc/kubernetes/admin.conf   (cluster-admin)
   └── /etc/kubernetes/kubelet.conf (node identity)
```

### Phase 2 — `kubeadm join` internals

```
1. Worker contacts API server on port 6443 using the bootstrap token
2. Downloads cluster CA cert, validates against --discovery-token-ca-cert-hash
3. Generates kubelet client cert via CSR (signed by cluster CA)
4. kubelet starts, registers node object with API server
5. kube-proxy DaemonSet pod scheduled onto new node
6. Calico calico-node DaemonSet pod scheduled onto new node
7. Node transitions: NotReady → Ready (once CNI is up)
```

### Phase 3 — Calico operator bootstrap

```
1. tigera-operator Deployment starts (watches for Installation CR)
2. You create Installation CR with your pod CIDR spec
3. Operator creates:
   ├── calico-system namespace
   ├── calico-node DaemonSet (Felix + BIRD)
   ├── calico-typha Deployment (proxy between Felix and datastore)
   └── calico-kube-controllers Deployment
4. Felix programs iptables/nftables rules on each node
5. VXLAN VTEP interface (vxlan.calico) created on each node
6. IP pool allocated per node (/26 block from 192.168.0.0/16)
7. CoreDNS transitions: Pending → Running
```

### Why node NotReady happens

```
kubelet reports NotReady when:
  ├── CNI binary not found in /opt/cni/bin/
  ├── CNI config not in /etc/cni/net.d/
  ├── Container runtime unreachable
  └── Node lease not renewed (kubelet crash/resource pressure)
```

---

## 4 · Hands-On 🛠️

> *Production-quality commands. Nothing simplified. Labels show exactly where to run.*

### Step 1 — Disable swap & kernel networking `[ALL NODES]`

```bash
# Disable swap immediately and persist across reboots
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Load required kernel modules now and on reboot
echo -e "overlay\nbr_netfilter" | sudo tee /etc/modules-load.d/k8s.conf
sudo modprobe overlay
sudo modprobe br_netfilter

# Enable bridged traffic visibility and IP forwarding
cat <<'EOF' | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
EOF
sudo sysctl --system

# Verify
lsmod | grep -E "overlay|br_netfilter"
sysctl net.ipv4.ip_forward net.bridge.bridge-nf-call-iptables
```

### Step 2 — Install and configure containerd `[ALL NODES]`

```bash
sudo apt-get update && sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null

# Switch to systemd cgroup driver (must match kubelet)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# Pin the pause image version (prevents sandbox churn during upgrades)
sudo sed -i 's#sandbox_image = ".*"#sandbox_image = "registry.k8s.io/pause:3.9"#' \
  /etc/containerd/config.toml

sudo systemctl daemon-reload
sudo systemctl enable --now containerd
sudo systemctl status containerd --no-pager

# Verify CRI socket is reachable
sudo ctr version
```

### Step 3 — Install kubeadm, kubelet, kubectl v1.32 `[ALL NODES]`

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update && sudo apt-get install -y kubelet kubeadm kubectl

# Pin versions — never let apt auto-upgrade your cluster binaries
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable kubelet

# Verify
kubeadm version
kubelet --version
kubectl version --client
```

**Optional but highly recommended — kubectl aliases & completion:**

```bash
sudo apt-get install -y bash-completion
echo 'source /usr/share/bash-completion/bash_completion' >> ~/.bashrc
echo 'source <(kubectl completion bash)'                  >> ~/.bashrc
echo 'alias k=kubectl'                                    >> ~/.bashrc
echo 'complete -F __start_kubectl k'                      >> ~/.bashrc
source ~/.bashrc
```

### Step 4 — Initialize the control plane `[CONTROL PLANE ONLY]`

```bash
# Replace <CP_PRIVATE_IP> with your actual private IP
export CP_IP=<CP_PRIVATE_IP>

sudo kubeadm init \
  --control-plane-endpoint="${CP_IP}:6443" \
  --apiserver-advertise-address="${CP_IP}" \
  --pod-network-cidr=192.168.0.0/16

# Set up kubeconfig for your user
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Verify control plane components
kubectl get nodes
kubectl -n kube-system get pods
```

> **1 vCPU workaround (demos only):**
> ```bash
> sudo kubeadm init \
>   --control-plane-endpoint="${CP_IP}:6443" \
>   --apiserver-advertise-address="${CP_IP}" \
>   --pod-network-cidr=192.168.0.0/16 \
>   --ignore-preflight-errors=NumCPU
> ```

### Step 5 — Install Calico via Operator `[CONTROL PLANE ONLY]`

```bash
# Install CRDs and operator
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.2/manifests/operator-crds.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.2/manifests/tigera-operator.yaml

# Wait for operator to come up
kubectl -n tigera-operator rollout status deploy/tigera-operator

# Apply the Installation CR — this is what triggers everything
cat <<'EOF' | kubectl apply -f -
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default          # must be exactly "default" — singleton resource
spec:
  calicoNetwork:
    bgp: Disabled        # VXLAN-only — disables BIRD to avoid BGP readiness failures
    ipPools:
    - cidr: 192.168.0.0/16          # must match --pod-network-cidr
      natOutgoing: Enabled          # SNAT pods going to external IPs
      blockSize: 26                 # /26 = 64 pod IPs per node block
      encapsulation: VXLANCrossSubnet  # VXLAN across subnets, native within same L2
      nodeSelector: all()           # apply to every node
EOF

# Watch it come up (takes ~2 minutes)
watch kubectl -n calico-system get pods -o wide
```

### Step 6 — Join workers `[WORKER ONLY]`

```bash
# On control plane — generate fresh join command (tokens expire after 24h)
kubeadm token create --print-join-command
```

Run the printed command on **each** worker:

```bash
# Example output — yours will have different token and hash
sudo kubeadm join <CP_PRIVATE_IP>:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

### Step 7 — Verify and smoke test `[CONTROL PLANE]`

```bash
# All nodes Ready?
kubectl get nodes -o wide

# All system pods Running?
kubectl -n kube-system get pods -o wide
kubectl -n calico-system get pods -o wide

# Deploy and expose a test app
kubectl create deploy web --image=nginx --replicas=3
kubectl expose deploy/web --port=80 --type=NodePort

# Find the NodePort
kubectl get svc web -o jsonpath='{.spec.ports[0].nodePort}'; echo

# Hit it from your machine (both workers should respond)
curl -I http://<worker-1-public-ip>:<nodeport>
curl -I http://<worker-2-public-ip>:<nodeport>
```

### Calico Troubleshooting `[CONTROL PLANE]`

**Symptom:** `calico-node` pods stuck in `Running (0/1)`, logs show BIRD/BGP errors or Felix/Typha connection refused.

```bash
# Step 1: Check if BGP is enabled (empty output or "Enabled" means ON)
kubectl get installation.operator.tigera.io default \
  -o jsonpath='{.spec.calicoNetwork.bgp}{"\n"}'

# Step 2: Disable BGP (idempotent — safe to run even if already Disabled)
kubectl patch installation.operator.tigera.io default --type=merge \
  -p '{"spec":{"calicoNetwork":{"bgp":"Disabled"}}}'

# Step 3: Restart DaemonSet to pick up change
kubectl -n calico-system rollout restart ds/calico-node
kubectl -n calico-system rollout status ds/calico-node

# Step 4: If still failing — check Typha connectivity
kubectl -n calico-system get deploy,svc,endpoints -l k8s-app=calico-typha
kubectl -n calico-system logs ds/calico-node -c calico-node | grep -i typha

# Step 5: Nuclear reset (workers first, then control plane)
# WORKERS:
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d /var/lib/cni /var/lib/kubelet/pki ~/.kube
for i in cni0 vxlan.calico tunl0; do sudo ip link del "$i" 2>/dev/null || true; done
sudo systemctl restart containerd

# CONTROL PLANE:
sudo kubeadm reset -f
sudo rm -rf /etc/kubernetes /var/lib/etcd /etc/cni/net.d /var/lib/cni \
            /var/lib/kubelet/pki ~/.kube
for i in cni0 vxlan.calico tunl0; do sudo ip link del "$i" 2>/dev/null || true; done
sudo systemctl restart containerd
```

### Copy kubeconfig to worker (optional)

```bash
# On control-plane:
sudo cp /etc/kubernetes/admin.conf /tmp/kubeconfig
sudo chown $USER:$USER /tmp/kubeconfig
scp /tmp/kubeconfig ubuntu@worker-1:~/.kube/config

# On worker-1:
mkdir -p ~/.kube
chmod 600 ~/.kube/config
kubectl get nodes   # works now (cluster-admin access — labs only)
```

---

## 5 · Production Flow 🏗️

> *Real-world architecture — what you'd actually build and why.*

### Reference architecture (this guide)

```
┌──────────────────────────────────────────────────────────────┐
│  Your Machine                                                │
│  kubectl → port 6443 → control-plane public IP              │
└──────────────────────────────────────────────────────────────┘
                            │
                     ┌──────▼──────┐
                     │ control-plane│  2 vCPU / 4GB
                     │  TCP 6443   │  /etc/kubernetes/pki/
                     │  TCP 5473   │  /var/lib/etcd/
                     │  UDP 4789   │  static pods: apiserver
                     └──────┬──────┘           etcd
                            │                  scheduler
          ┌─────────────────┼──────────────────┐ controller-mgr
          │                 │                  │
   ┌──────▼──────┐   ┌──────▼──────┐          ...
   │  worker-1   │   │  worker-2   │
   │ TCP 10250   │   │ TCP 10250   │
   │ UDP 4789    │   │ UDP 4789    │
   │ TCP 5473    │   │ TCP 5473    │
   └─────────────┘   └─────────────┘
         │                  │
    vxlan.calico ◄──────────► vxlan.calico
    192.168.x.x/26           192.168.y.y/26
         │                  │
    [pod IPs]           [pod IPs]
```

### Startup order matters

```
1. containerd      (must be Running before kubelet starts)
2. kubelet         (reads /etc/kubernetes/manifests/, starts static pods)
3. etcd            (static pod — started by kubelet)
4. kube-apiserver  (static pod — needs etcd healthy first)
5. kube-scheduler + kube-controller-manager (static pods)
6. calico-node     (DaemonSet — needs API server to schedule)
7. CoreDNS         (Deployment — needs calico-node for networking)
8. Your workloads  (need CoreDNS for service discovery)
```

### Production additions not covered here

| Gap | Production solution |
|---|---|
| Single etcd | External etcd cluster or stacked HA (`--upload-certs`) |
| Single control plane | 3 CP nodes behind a load balancer (pass LB IP to `--control-plane-endpoint`) |
| Manual cert renewal | `kubeadm certs renew all` in cron, or upgrade annually |
| No audit logs | Add `--audit-log-path` and policy file to apiserver manifest |
| No image pull secrets | `imagePullSecrets` in ServiceAccount or pod spec |
| NodePort exposure | Use a LoadBalancer service or Ingress controller |
| No RBAC scoping | Never hand out `admin.conf` — use namespaced RoleBindings |

---

## 6 · Mistakes 💥

> *What actually breaks in real systems — root cause and fix, not just symptoms.*

### Mistake 1: Swap still enabled
- **Root cause:** `/etc/fstab` wasn't updated, swap comes back after reboot
- **Symptom:** kubelet fails to start, `journalctl -u kubelet` shows "swap is enabled"
- **Fix:** `sudo swapoff -a && sudo sed -i '/ swap / s/^/#/' /etc/fstab`

### Mistake 2: cgroup driver mismatch
- **Root cause:** containerd uses `cgroupfs`, kubelet expects `systemd` (or vice versa)
- **Symptom:** Pods start and immediately die; `journalctl -u kubelet` shows cgroup errors
- **Fix:** Ensure `SystemdCgroup = true` in `/etc/containerd/config.toml`, then `sudo systemctl restart containerd`

### Mistake 3: Pod CIDR mismatch between kubeadm and Calico
- **Root cause:** `--pod-network-cidr=10.244.0.0/16` but Calico CR uses `192.168.0.0/16`
- **Symptom:** Pods get IPs in the wrong range, cross-node routing fails silently
- **Fix:** Must match exactly. Easiest fix is full cluster reset and reinstall.

### Mistake 4: BGP enabled in VXLAN Calico install
- **Root cause:** Operator default enables BGP even when using VXLAN
- **Symptom:** `calico-node` Running but 0/1, logs show "BGP not established" / BIRD errors
- **Fix:** `kubectl patch installation.operator.tigera.io default --type=merge -p '{"spec":{"calicoNetwork":{"bgp":"Disabled"}}}'`, then restart DaemonSet

### Mistake 5: TCP 5473 not open → Typha connection refused
- **Root cause:** Security group / firewall blocks Typha port between nodes
- **Symptom:** `calico-node` logs show "timeout connecting to Typha" on port 5473
- **Fix:** Open TCP 5473 between all nodes (including CP↔workers)

### Mistake 6: Join token expired
- **Root cause:** Default token TTL is 24 hours; waited too long
- **Symptom:** `kubeadm join` returns "token not found" or "unauthorized"
- **Fix:** `kubeadm token create --print-join-command` on the control plane

### Mistake 7: Missing kernel modules
- **Root cause:** `overlay` or `br_netfilter` not loaded (skipped Step 1)
- **Symptom:** Containers fail to start, containerd logs show overlay filesystem errors
- **Fix:** `sudo modprobe overlay br_netfilter` and add to `/etc/modules-load.d/k8s.conf`

### Mistake 8: Running `kubeadm join` without `sudo`
- **Root cause:** Permission error writing to `/etc/kubernetes/`
- **Symptom:** `join` fails immediately with permission denied
- **Fix:** Always prefix with `sudo`

### Mistake 9: Calico leftover state after reset
- **Root cause:** Old CNI config/interfaces remain after `kubeadm reset`
- **Symptom:** New pods get duplicate IPs, networking broken from first boot
- **Fix:** Remove `/etc/cni/net.d`, `/var/lib/cni`, delete `cni0`, `vxlan.calico`, `tunl0` interfaces

### Mistake 10: kubeconfig not set up after init
- **Root cause:** Skipping the `mkdir ~/.kube && cp admin.conf` step
- **Symptom:** `kubectl get nodes` returns "connection refused" or "no config"
- **Fix:** `mkdir -p $HOME/.kube && sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config && sudo chown $(id -u):$(id -g) $HOME/.kube/config`

---

## 7 · Interview Answers 🎤

> *Full spoken sentences. Deliver these verbatim.*

---

**Q: Walk me through what happens when you run `kubeadm init`.**

"When you run `kubeadm init`, it first runs preflight checks — verifying swap is off, kernel modules are loaded, the container runtime is reachable, and you have at least two CPUs. Then it generates a certificate authority and all the component certificates under `/etc/kubernetes/pki`. It writes static pod manifests for the API server, etcd, scheduler, and controller manager to `/etc/kubernetes/manifests`. The kubelet — which is already running — detects those manifests and asks containerd to start those containers. Once the API server is up, kubeadm bootstraps RBAC, writes kubeconfig files, and creates a short-lived join token in a ConfigMap. At the end, it prints the `kubeadm join` command for you to run on workers."

---

**Q: Why do you need to disable swap before installing Kubernetes?**

"Kubernetes makes scheduling and resource-limit decisions assuming physical memory is finite and predictable. When swap is enabled, the kernel can silently page memory to disk, which breaks kubelet's memory accounting — a pod that's over its limit might not get OOM-killed when it should be. kubelet explicitly checks for swap and refuses to start unless you pass `--fail-swap-on=false`, which is not something you want in production. So you disable it permanently: `swapoff -a` for immediate effect, and comment it out in `/etc/fstab` to survive reboots."

---

**Q: Why must the cgroup driver match between containerd and kubelet?**

"Both containerd and kubelet need to agree on how they talk to the Linux kernel's cgroup subsystem. If containerd uses `cgroupfs` and kubelet uses `systemd`, they each create and track cgroup hierarchies independently. This causes resource accounting to be out of sync — containers might get double-counted or escape limits entirely — and can lead to cryptic pod crashes. The correct configuration is `SystemdCgroup = true` in containerd's config, which makes systemd the single source of truth for cgroup management, and kubelet defaults to systemd on modern distros. This is why we patch the containerd config before touching kubeadm."

---

**Q: What is Calico Typha, and why does it need its own firewall port?**

"Typha is a fan-out proxy that sits between the Calico datastore — which is backed by the Kubernetes API server — and the Felix agents running on every node. Without Typha, every Felix instance would independently watch the API server for policy updates, and in a large cluster you'd overwhelm the API server with watch connections. Typha aggregates those watches and fans the updates out to Felix over TCP 5473. In cloud environments with security groups, this means you need to explicitly open TCP 5473 between nodes so Felix can reach Typha — otherwise `calico-node` sits at Running but never becomes Ready."

---

**Q: How does a worker node join the cluster securely?**

"The join process uses a two-factor trust model. The first factor is the bootstrap token — a short-lived credential that the worker uses to authenticate to the API server for the first time. The second factor is the CA certificate hash — a SHA-256 fingerprint of the cluster's CA cert — which the worker uses to verify it's talking to the right API server and not a man-in-the-middle. Once authenticated, the worker generates a CSR, the controller manager signs it with the cluster CA, and kubelet gets a permanent client certificate. After that, the bootstrap token is no longer used."

---

**Q: CoreDNS is Pending after `kubeadm init`. What's wrong?**

"CoreDNS stays Pending until a CNI plugin is installed. Kubernetes has no built-in pod networking — it delegates that entirely to the CNI. Without a CNI, pods can't get network interfaces, so the scheduler marks them unschedulable on any node. Once you install Calico — or any CNI — it creates the network plumbing, the nodes transition to Ready, and CoreDNS gets scheduled and starts running. The fix is always: install your CNI immediately after `kubeadm init`, before doing anything else."

---

**Q: What is `br_netfilter` and why do you load it?**

"`br_netfilter` is a kernel module that causes the Linux bridge to call iptables for bridged traffic — traffic that flows between pods on the same node through the virtual bridge. Without it, iptables rules (which Kubernetes uses for Services and NetworkPolicy) are simply bypassed for bridged packets. You also need `net.bridge.bridge-nf-call-iptables=1` in sysctl to activate that behavior. Together they ensure that kube-proxy's iptables rules actually intercept pod-to-pod and pod-to-Service traffic on the same node."

---

## 8 · Debugging 🔍

> *Diagnosis paths — not a list of commands, but a decision tree you follow in order.*

### Path A — Node stuck in NotReady

```
kubectl get nodes                       # identify which node
kubectl describe node <name>            # read "Conditions" section
         │
         ├── "KubeletNotReady: container runtime not ready"
         │       └── ssh to node → sudo systemctl status containerd
         │                        sudo journalctl -u containerd -n 50
         │
         ├── "NetworkPluginNotReady: network plugin is not ready"
         │       └── CNI not installed or broken
         │           kubectl -n calico-system get pods -o wide
         │           → go to Path B
         │
         ├── "Kubelet stopped posting node status"
         │       └── ssh → sudo systemctl status kubelet
         │                 sudo journalctl -u kubelet -n 100
         │                 → check for swap / cgroup errors
         │
         └── Node lease timeout
                 └── Check node resources: df -h, free -m, top
```

### Path B — calico-node not Ready (0/1)

```
kubectl -n calico-system get pods -o wide           # which pods affected?
kubectl -n calico-system describe pod <calico-pod>  # Events section
kubectl -n calico-system logs <calico-pod> -c calico-node --tail=100
         │
         ├── "BGP not established" or BIRD errors
         │       └── kubectl get installation default -o jsonpath='{.spec.calicoNetwork.bgp}'
         │           → If not "Disabled":
         │             kubectl patch installation.operator.tigera.io default \
         │               --type=merge -p '{"spec":{"calicoNetwork":{"bgp":"Disabled"}}}'
         │             kubectl -n calico-system rollout restart ds/calico-node
         │
         ├── "timeout connecting to Typha" or "connection refused" on 5473
         │       └── Check security groups / firewall for TCP 5473
         │           kubectl -n calico-system get endpoints calico-typha
         │           → Open TCP 5473 between all nodes
         │
         └── Image pull errors
                 └── sudo systemctl status containerd
                     sudo ctr images pull <image>     # test pull manually
                     → Check NAT/internet access from node
```

### Path C — Pod stuck in Pending

```
kubectl describe pod <pod>     # read Events section
         │
         ├── "0/3 nodes available: insufficient cpu/memory"
         │       └── kubectl top nodes   (needs metrics-server)
         │           kubectl get nodes -o custom-columns='NAME:.metadata.name,CPU:.status.capacity.cpu,MEM:.status.capacity.memory'
         │
         ├── "0/3 nodes available: node had taints that pod didn't tolerate"
         │       └── kubectl describe nodes | grep -A3 Taints
         │           → Add toleration to pod spec, or
         │           → kubectl taint node <name> <key>-  (remove taint)
         │
         └── "Unable to mount volumes"
                 └── kubectl describe pvc <name>   # check PVC bound?
```

### Path D — kubeadm join fails

```
kubeadm join error message:
         │
         ├── "token not found" / "unauthorized"
         │       └── Token expired (24h TTL)
         │           On CP: kubeadm token create --print-join-command
         │
         ├── "connection refused" on port 6443
         │       └── API server not running, or SG blocking TCP 6443
         │           On CP: kubectl get pods -n kube-system | grep apiserver
         │           Check security group / firewall
         │
         └── "x509: certificate signed by unknown authority"
                 └── --discovery-token-ca-cert-hash is wrong
                     Re-run: kubeadm token create --print-join-command
```

### Essential one-liners for quick diagnosis

```bash
# All pods across all namespaces with status
kubectl get pods -A -o wide

# Events sorted by time (usually tells you what just broke)
kubectl get events -A --sort-by='.lastTimestamp' | tail -30

# kubelet logs on any node (SSH first)
sudo journalctl -u kubelet -n 100 --no-pager

# containerd logs
sudo journalctl -u containerd -n 50 --no-pager

# CNI config on a node
cat /etc/cni/net.d/*.conflist

# Pod IP routing check
kubectl exec -it <pod> -- ip route
kubectl exec -it <pod> -- ping <other-pod-ip>

# Is Typha reachable?
kubectl -n calico-system get endpoints calico-typha -o wide

# Node conditions at a glance
kubectl get nodes -o custom-columns='NAME:.metadata.name,STATUS:.status.conditions[-1].type,READY:.status.conditions[-1].status'
```

---

## 9 · Kill Switch 🔑

> *10-second recall — the absolute minimum to hold in memory.*

```
┌────────────────────────────────────────────────────────────────────┐
│  THE 5 LAWS OF kubeadm BOOTSTRAP                                  │
│                                                                    │
│  1. Swap off, modules on, sysctl applied  →  BEFORE containerd    │
│  2. containerd SystemdCgroup = true       →  BEFORE kubelet       │
│  3. kubeadm init pod-cidr == Calico cidr  →  192.168.0.0/16       │
│  4. BGP Disabled when using VXLAN         →  or BIRD will cry     │
│  5. Open 6443, 5473, 4789                 →  or nothing works     │
└────────────────────────────────────────────────────────────────────┘

BROKEN CALICO? → Disable BGP → Restart calico-node DaemonSet → Open TCP 5473

NODE NOT READY? → kubelet logs → containerd status → CNI config present?

JOIN FAILED?  → Token expired → Re-run: kubeadm token create --print-join-command

SCORCHED EARTH RESET ORDER: Workers → Control Plane (never CP first)
```

---

## 10 · Appendix 📋

> *Quick reference — copy-paste ready.*

### Node setup (run in order, all nodes)

```bash
# 1. Swap
sudo swapoff -a && sudo sed -i '/ swap / s/^/#/' /etc/fstab

# 2. Modules
echo -e "overlay\nbr_netfilter" | sudo tee /etc/modules-load.d/k8s.conf
sudo modprobe overlay && sudo modprobe br_netfilter

# 3. Sysctl
cat <<'EOF' | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
EOF
sudo sysctl --system

# 4. containerd
sudo apt-get update && sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo sed -i 's#sandbox_image = ".*"#sandbox_image = "registry.k8s.io/pause:3.9"#' /etc/containerd/config.toml
sudo systemctl enable --now containerd

# 5. kubeadm stack
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update && sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable kubelet
```

### Control plane only

```bash
sudo kubeadm init --control-plane-endpoint=<CP_IP>:6443 \
  --apiserver-advertise-address=<CP_IP> --pod-network-cidr=192.168.0.0/16
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Calico install

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.2/manifests/operator-crds.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.2/manifests/tigera-operator.yaml
```

```yaml
# Installation CR — save as calico-install.yaml
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    bgp: Disabled
    ipPools:
    - cidr: 192.168.0.0/16
      natOutgoing: Enabled
      blockSize: 26
      encapsulation: VXLANCrossSubnet
      nodeSelector: all()
```

### Useful kubectl one-liners

```bash
# Regenerate join command
kubeadm token create --print-join-command

# Check certs expiry
kubeadm certs check-expiration

# Renew all certs
sudo kubeadm certs renew all

# Get all pods not Running
kubectl get pods -A --field-selector=status.phase!=Running

# Watch pods come up
watch -n2 kubectl get pods -A -o wide

# Describe all events for a namespace
kubectl get events -n <ns> --sort-by='.lastTimestamp'

# Force delete stuck pod
kubectl delete pod <name> -n <ns> --force --grace-period=0

# Shell into a pod
kubectl exec -it <pod> -n <ns> -- /bin/sh

# Get logs from crashed container
kubectl logs <pod> -n <ns> --previous

# Check resource usage
kubectl top nodes
kubectl top pods -A
```

### Port reference card

| Port | Protocol | Direction | Component |
|---|---|---|---|
| 6443 | TCP | → Control Plane | Kubernetes API |
| 2379–2380 | TCP | CP ↔ CP | etcd (HA only) |
| 10250 | TCP | → Worker | kubelet API |
| 10257 | TCP | Internal | controller-manager |
| 10259 | TCP | Internal | scheduler |
| 5473 | TCP | All ↔ All | Calico Typha |
| 4789 | UDP | All ↔ All | VXLAN overlay |
| 179 | TCP | All ↔ All | BGP (disabled in this guide) |
| 30000–32767 | TCP | → Worker | NodePort services |

### File locations cheatsheet

| File | Purpose |
|---|---|
| `/etc/kubernetes/pki/` | All TLS certs and keys |
| `/etc/kubernetes/manifests/` | Static pod definitions (apiserver, etcd, etc.) |
| `/etc/kubernetes/admin.conf` | Cluster-admin kubeconfig |
| `/var/lib/etcd/` | etcd data directory — back this up! |
| `/etc/containerd/config.toml` | containerd configuration |
| `/etc/cni/net.d/` | CNI config (written by Calico) |
| `/opt/cni/bin/` | CNI plugin binaries |
| `/etc/modules-load.d/k8s.conf` | Kernel modules to load at boot |
| `/etc/sysctl.d/99-kubernetes-cri.conf` | Kernel parameters for networking |
| `~/.kube/config` | User kubeconfig |

---
