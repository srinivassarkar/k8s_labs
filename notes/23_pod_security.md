# Pod Security in Kubernetes — Security Context & Linux Capabilities
---

## 0. First Principles

> *The mental model — what never changes, regardless of Kubernetes version.*

**Linux identity is UID/GID — not usernames.** The kernel enforces access decisions using numeric IDs. `/etc/passwd` is just a human-readable lookup table. A container running `uid=1000` has identical kernel-level permissions whether that UID is named `appuser` or has no name at all.

**Root is UID 0 — period.** The string `root` is irrelevant. What matters is whether the process has UID 0. `runAsNonRoot: true` enforces that UID 0 is rejected at container start, before the process can do any damage.

**Linux capabilities decompose root.** Traditional Unix was binary: you're root or you're not. Capabilities split root privilege into ~40 discrete tokens. A process can hold `NET_ADMIN` without holding `SYS_ADMIN`. This is how least-privilege works at the kernel level.

**The container boundary is not a security boundary by itself.** Namespaces isolate visibility; cgroups limit resources; capabilities limit privilege. Without explicit security context configuration, a container running as root on a shared node is a lateral movement risk.

**Security context is enforced by the container runtime (containerd/CRI-O), not Kubernetes.** Kubernetes translates your `securityContext` YAML into OCI runtime spec. The actual enforcement is at the kernel via Linux's credential system and capability bitmask.

---

## 1. Reality Constraints

> *What Kubernetes actually does and doesn't do — where the system surprises you.*

| Constraint | Reality |
|---|---|
| Default pod runs as root | If the container image has no `USER` instruction, UID=0 is used unless explicitly overridden |
| `runAsNonRoot` doesn't set a UID | It only rejects UID 0. You must also set `runAsUser` with a non-zero value, otherwise startup fails with no clear alternative |
| `fsGroup` is pod-level only | You cannot set it per-container. It applies ownership to ALL mounted volumes in the pod |
| `capabilities` is container-level only | Cannot be set at `spec.securityContext`. Misplacing it is a silent YAML error |
| `allowPrivilegeEscalation` defaults to `true` | Even for non-root containers, privilege escalation via setuid is allowed unless explicitly set to `false` |
| `readOnlyRootFilesystem` blocks `/tmp` too | Apps that write to `/tmp` will break. You must mount an `emptyDir` at `/tmp` explicitly |
| `privileged: true` bypasses almost everything | It disables seccomp, all capability dropping, and grants full device access. Equivalent to running directly on the host |
| Image `USER` instruction is advisory | A Dockerfile `USER 1000` sets a default but a pod spec's `runAsUser` overrides it completely |
| `whoami` may return "unknown" | This is not an error. It means the UID has no entry in `/etc/passwd`. Kubernetes security works on UIDs, not names |
| Capability changes don't apply to root | If `runAsUser: 0`, capability dropping is largely ineffective in many environments because root can re-acquire capabilities |
| Kind/Minikube may not enforce `SYS_TIME` | Some capabilities are intercepted by the container runtime or not allowed by the node's kernel — behavior varies by cluster |

---

## 2. Decision Logic

> *When to use what — clear rules before you write a single line of YAML.*

### Which Level to Configure

```
Need to restrict the entire pod uniformly?
  └─ Yes → Use spec.securityContext (pod-level)
       ├─ runAsUser, runAsGroup, runAsNonRoot, fsGroup

Need container-specific override OR a field that only exists at container level?
  └─ Yes → Use spec.containers[].securityContext (container-level)
       ├─ allowPrivilegeEscalation, privileged, readOnlyRootFilesystem, capabilities

Overlap? (e.g., runAsUser at both levels)
  └─ Container-level ALWAYS wins
```

### Field Applicability Quick Reference

| Field | Pod Level | Container Level | Notes |
|---|---|---|---|
| `runAsUser` | ✅ | ✅ | Container overrides Pod |
| `runAsGroup` | ✅ | ✅ | Container overrides Pod |
| `runAsNonRoot` | ✅ | ✅ | Container overrides Pod |
| `fsGroup` | ✅ | ❌ | Pod-level only; volume group ownership |
| `allowPrivilegeEscalation` | ❌ | ✅ | Container-level only |
| `privileged` | ❌ | ✅ | Container-level only |
| `readOnlyRootFilesystem` | ❌ | ✅ | Container-level only |
| `capabilities` | ❌ | ✅ | Container-level only |

### When to Use `fsGroup`

```
Container needs to write to a mounted volume (PVC or emptyDir)?
  └─ Yes → Set fsGroup = GID that the container process belongs to
         → Mount the volume at the desired path
         → Container's supplemental groups will include fsGroup
         → Volume files will be owned by that GID
         → The 's' bit (setgid) on the mount dir ensures new files inherit GID

Container filesystem must be read-only but app needs temp writes?
  └─ Yes → readOnlyRootFilesystem: true + emptyDir volume at /tmp or /writable
         → Set fsGroup to allow group-based write access to that volume
```

### Capabilities Decision Tree

```
Does the app need any Linux capability beyond "run as a user"?
  └─ No  → drop: ["ALL"] and stop. No add: needed.
  └─ Yes → drop: ["ALL"] first, then add: ONLY the specific capability needed

Common capability needs:
  ├─ Bind to port < 1024         → NET_BIND_SERVICE
  ├─ Modify network interfaces   → NET_ADMIN
  ├─ Change system time          → SYS_TIME
  ├─ Change file ownership       → CHOWN
  ├─ Mount filesystems           → SYS_ADMIN (avoid if possible)
  └─ Raw socket access           → NET_RAW (drop this; it's granted by default and dangerous)
```

---

## 3. Internal Working

> *How it actually happens under the hood — from YAML to kernel enforcement.*

### Security Context → OCI Runtime Spec Pipeline

```
1. User submits pod YAML with securityContext fields
        ↓
2. kube-apiserver validates the spec, stores in etcd
        ↓
3. kube-scheduler assigns pod to a node
        ↓
4. kubelet on target node picks up the pod spec
        ↓
5. kubelet calls CRI (containerd or CRI-O) via gRPC
        ↓
6. CRI translates Kubernetes securityContext → OCI runtime spec (config.json)
   ├─ runAsUser → "uid" in process.user
   ├─ runAsGroup → "gid" in process.user
   ├─ supplementalGroups + fsGroup → "additionalGids"
   ├─ capabilities.add/drop → "process.capabilities" bitmask
   ├─ allowPrivilegeEscalation → "process.noNewPrivileges: true"
   ├─ readOnlyRootFilesystem → "root.readonly: true"
   └─ privileged → all capabilities added, seccomp disabled
        ↓
7. OCI runtime (runc/crun) executes the container with that spec
        ↓
8. Linux kernel enforces:
   ├─ UID/GID via credential struct in process descriptor
   ├─ Capabilities via capability bitmask (effective/permitted/inheritable sets)
   ├─ No new privs via PR_SET_NO_NEW_PRIVS prctl flag
   └─ Read-only rootfs via bind mount with MS_RDONLY flag
```

### How `fsGroup` Actually Changes Volume Ownership

When `fsGroup: 2000` is set and a volume is mounted:

```
1. kubelet mounts the volume (e.g., emptyDir) at the host path
2. kubelet calls chown on the mount point: chown :<fsGroup> /path
3. kubelet sets setgid bit: chmod g+s /path
4. Container starts with supplementalGroups including fsGroup GID
5. Any new file created in that path inherits the directory's GID (setgid)
6. Container process (which belongs to GID 2000 as a supplemental group) can read/write
```

Result: `drwxr-sr-x root 2000` — the `s` in `sr-x` is the setgid bit that propagates ownership.

### How `allowPrivilegeEscalation: false` Works

```
Linux mechanism: PR_SET_NO_NEW_PRIVS
  ├─ Set via prctl(PR_SET_NO_NEW_PRIVS, 1, ...)
  ├─ Once set, cannot be unset — inherited by all child processes
  ├─ Prevents execve() from granting more capabilities via setuid binaries
  └─ Makes setuid binaries run with caller's credentials, not file owner's

Without this: a binary owned by root with setuid bit runs as UID 0 regardless of caller
With this: setuid bit is ignored — binary runs as UID 1000 (the container user)
```

### Capability Bitmask (Simplified)

```
Each Linux capability is a bit in three sets per process:
  ├─ Permitted:    the maximum set the process may have
  ├─ Effective:    the set currently active (what the kernel checks)
  └─ Inheritable: what's passed through execve()

Kubernetes translates:
  capabilities.add:  ["NET_ADMIN"] → adds CAP_NET_ADMIN to permitted + effective
  capabilities.drop: ["ALL"]       → clears all bits in all three sets first
```

---

## 4. Hands-On

> *Production-quality YAML + commands. Nothing simplified.*

### Default (Insecure) Pod — Baseline Comparison

```yaml
# Day41/00-default-pod.yaml
# Run this first to see the problem: uid=0(root)
apiVersion: v1
kind: Pod
metadata:
  name: default-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
```

```bash
kubectl apply -f 00-default-pod.yaml
kubectl exec -it default-pod -- id
# uid=0(root) gid=0(root) groups=0(root),10(wheel)  ← DANGER
```

---

### Secure Pod with Full Security Context

```yaml
# Day41/01-security-context.yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:           # Pod-level: applies to ALL containers
    runAsUser: 1000          # Run as UID 1000 (not root)
    runAsGroup: 3000         # Primary GID 3000
    fsGroup: 2000            # Volume ownership GID (supplemental)
    runAsNonRoot: true       # Fail fast if image tries to run as UID 0
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    securityContext:         # Container-level: these fields ONLY work here
      allowPrivilegeEscalation: false   # Block setuid privilege gain
      readOnlyRootFilesystem: true      # Immutable root filesystem
```

```bash
kubectl apply -f 01-security-context.yaml
kubectl exec -it secure-pod -- sh

# Verify identity
id
# uid=1000 gid=3000 groups=2000,3000

# Verify read-only filesystem
mkdir /test_dir
# mkdir: can't create directory '/test_dir': Read-only file system

# Verify no privilege escalation
sudo su
# sh: sudo: not found
su -
# su: must be suid to work properly

# Note: whoami returns "unknown" — not an error
whoami
# whoami: unknown uid 1000
```

---

### Secure Writable Volume Pattern (Production-Safe)

```yaml
# Day41/02-security-context.yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-writable
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000            # GID that gets write access to the volume
    runAsNonRoot: true
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
    volumeMounts:
    - name: writable-vol
      mountPath: /writable   # Only this path is writable
    - name: tmp-vol
      mountPath: /tmp        # Apps that need /tmp also need this
  volumes:
  - name: writable-vol
    emptyDir: {}
  - name: tmp-vol
    emptyDir: {}
```

```bash
kubectl apply -f 02-security-context.yaml
kubectl exec -it secure-writable -- sh

# Root filesystem is locked
mkdir /tmp/test
# mkdir: can't create directory '/tmp/test': Read-only file system

# But /writable is accessible (via fsGroup)
cd /writable
mkdir test_dir && touch myfile
ls -lthr
# drwxr-sr-x 2 1000 2000   4.0K test_dir   ← GID 2000 from fsGroup
# -rw-r--r-- 1 1000 2000      0 myfile     ← setgid inherits GID

# /tmp is writable via its own emptyDir mount
touch /tmp/scratch
# succeeds
```

---

### Linux Capabilities — Drop All, Add Selectively

```yaml
# Day41/03-capability-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: cap-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL                # Start from zero capabilities
        add:
        - NET_ADMIN          # Needed: network interface management
        - SYS_TIME           # Needed: system clock adjustment
```

```bash
kubectl apply -f 03-capability-pod.yaml
kubectl exec -it cap-pod -- sh

# Test SYS_TIME (may depend on runtime/environment)
date -s "2025-06-01 12:00:00"
# Expected: clock updates if runtime allows

# Test NET_ADMIN
ip link set eth0 down
# Expected: succeeds in permissive environments

# Test CHOWN (was dropped — ALL was dropped, CHOWN was not re-added)
touch myfile && chown 1000:1000 myfile
# chown: myfile: Operation not permitted  ← CORRECT — capability was dropped
```

---

### Strictest Hardening Profile (Production Baseline)

```yaml
# production-hardened-pod.yaml
# Maximum restriction — start here and relax only what the app needs
apiVersion: v1
kind: Pod
metadata:
  name: hardened-pod
  labels:
    app: hardened
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001         # High UID to avoid collision with system users
    runAsGroup: 10001
    fsGroup: 10001
    seccompProfile:
      type: RuntimeDefault   # Enable default seccomp filtering (Kubernetes 1.19+)
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      privileged: false
      capabilities:
        drop:
        - ALL
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    resources:
      requests:
        memory: "64Mi"
        cpu: "100m"
      limits:
        memory: "128Mi"
        cpu: "200m"
  volumes:
  - name: tmp
    emptyDir:
      medium: Memory         # RAM-backed tmp for sensitive scratch data
      sizeLimit: 50Mi
```

---

## 5. Production Flow

> *Real-world architecture and design patterns you must know for senior interviews.*

### Layered Defense Architecture

```
Layer 1: Pod Security Standards (cluster-wide admission)
  └─ Enforced via namespace labels:
     pod-security.kubernetes.io/enforce: restricted
     pod-security.kubernetes.io/warn: restricted
     pod-security.kubernetes.io/audit: restricted

Layer 2: Pod Security Context (per-pod/container spec)
  └─ Explicit runAsUser, runAsNonRoot, capabilities.drop: ALL

Layer 3: OPA Gatekeeper / Kyverno (policy engine)
  └─ Rejects pods missing required security fields
  └─ Enforces org-wide standards independent of developer YAML

Layer 4: Seccomp / AppArmor / SELinux (kernel-level syscall filtering)
  └─ seccompProfile.type: RuntimeDefault blocks dangerous syscalls
  └─ Applied regardless of capability settings

Layer 5: Runtime Security (Falco)
  └─ Detects anomalous behavior at runtime
  └─ E.g., unexpected shell spawn, file write to /etc
```

### Pod Security Standards (PSS) Quick Reference

| Level | What It Enforces |
|---|---|
| `privileged` | No restrictions — equivalent to no policy |
| `baseline` | Blocks privileged containers, host namespaces, certain volume types |
| `restricted` | Requires non-root, drops ALL capabilities, requires seccomp |

```bash
# Apply restricted PSS to a namespace
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest
```

### Multi-Container Pod Security Inheritance Pattern

```yaml
# Pattern: strict pod-level defaults, relaxed per-container where justified
spec:
  securityContext:
    runAsNonRoot: true    # Default for all containers
    runAsUser: 1000
    fsGroup: 2000
  containers:
  - name: app             # Inherits pod-level; adds container-level restrictions
    securityContext:
      readOnlyRootFilesystem: true
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]

  - name: log-shipper     # Needs one extra capability — explicit add
    securityContext:
      readOnlyRootFilesystem: true
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
        add: ["NET_BIND_SERVICE"]
```

### When to Use `privileged: true` (Rare, Documented Exceptions)

```
Legitimate use cases:
  ├─ CNI plugins (e.g., Calico, Cilium node agents)
  ├─ CSI drivers (storage drivers needing device access)
  ├─ Monitoring agents needing kernel access (e.g., Falco, eBPF tools)
  └─ Node management daemonsets (e.g., device drivers, CUDA)

Pattern: Run privileged containers in a dedicated namespace with:
  ├─ Separate RBAC — developers cannot deploy here
  ├─ NetworkPolicy — restrict egress from privileged pods
  └─ Audit logging — every action from privileged pods is logged
```

---

## 6. Mistakes

> *What actually breaks in real systems — root cause + fix.*

### Mistake 1: `readOnlyRootFilesystem: true` Breaks the App

**Symptom:** Pod starts but app crashes immediately. Logs show permission denied on `/tmp`, `/var/run`, `/etc/config`.

**Root Cause:** Application writes to standard filesystem paths that are now read-only.

**Fix:**
```yaml
volumeMounts:
- name: tmp-vol
  mountPath: /tmp
- name: run-vol
  mountPath: /var/run
volumes:
- name: tmp-vol
  emptyDir: {}
- name: run-vol
  emptyDir: {}
```

---

### Mistake 2: `runAsNonRoot: true` Without `runAsUser` Set

**Symptom:** Pod fails to start. Event shows: `container has runAsNonRoot and image will run as root`.

**Root Cause:** `runAsNonRoot` rejects UID 0 but doesn't assign an alternative UID. If the image has no `USER` instruction, it defaults to 0 and is rejected.

**Fix:** Always pair `runAsNonRoot: true` with an explicit `runAsUser`:
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000      # Required — tells the runtime which UID to use
```

---

### Mistake 3: Placing `capabilities` at Pod Level

**Symptom:** YAML applies without error, but capabilities are not enforced. `kubectl get pod -o yaml` shows the field missing from status.

**Root Cause:** `capabilities` is silently ignored at `spec.securityContext`. It's only valid under `spec.containers[].securityContext`.

**Fix:**
```yaml
# WRONG
spec:
  securityContext:
    capabilities:          # ← ignored here
      drop: ["ALL"]

# CORRECT
spec:
  containers:
  - name: app
    securityContext:
      capabilities:        # ← correct location
        drop: ["ALL"]
```

---

### Mistake 4: `fsGroup` Not Propagated to Pre-Existing Volume Data

**Symptom:** Volume mounts correctly, but files already present on the volume are not writable by the container.

**Root Cause:** By default, Kubernetes only applies fsGroup ownership recursively when the volume is first populated. For PVCs with existing data, this recursive chown can be controlled by `fsGroupChangePolicy`.

**Fix:**
```yaml
spec:
  securityContext:
    fsGroup: 2000
    fsGroupChangePolicy: "Always"   # Force recursive chown on every mount
    # Options: "Always" | "OnRootMismatch" (default, only if root dir doesn't match)
```

---

### Mistake 5: Dropping ALL Capabilities Breaks App Startup

**Symptom:** Container CrashLoopBackOff. Logs show operation not permitted on startup, often during network initialization or file ownership changes.

**Root Cause:** Default containers get capabilities like `CHOWN`, `SETUID`, `NET_BIND_SERVICE`, `KILL`. Dropping ALL removes these. Many apps rely on them implicitly.

**Fix — Iterative Approach:**
```bash
# 1. Run with drop: ["ALL"] and check what fails
# 2. Use strace or audit logs to find the failing syscall → map to capability
# 3. Add back only what's needed

# Useful: list capabilities a running process has
kubectl exec -it <pod> -- cat /proc/1/status | grep Cap
# CapEff: 0000000000003000   ← decode with capsh
capsh --decode=0000000000003000
```

---

### Mistake 6: `allowPrivilegeEscalation` Defaults to True

**Symptom:** Security audit flags containers that don't explicitly set `allowPrivilegeEscalation: false`. The team insists "we're running as non-root, we're safe."

**Root Cause:** Even a non-root user (UID 1000) can execute a setuid binary and gain root unless `allowPrivilegeEscalation: false` is set. This is independent of `runAsUser`.

**Fix:** Always explicitly set it:
```yaml
securityContext:
  allowPrivilegeEscalation: false   # Never omit this
```

---

## 7. Interview Answers

> *Compressed, verbatim-ready answers. Deliver exactly these — no padding.*

---

**Q: Why is running a container as root dangerous in Kubernetes?**

"Running as root means the container process has UID 0, which grants unrestricted access within the container's namespaces. If that container is compromised, an attacker can potentially escape the container, access host paths through mounted volumes, modify shared kernel state, or pivot to other workloads. It also violates the principle of least privilege, which is required by most compliance frameworks. The fix is to set `runAsUser` to a non-zero UID and enforce `runAsNonRoot: true` so the pod fails fast if the image tries to default to root."

---

**Q: What's the difference between pod-level and container-level security context?**

"Pod-level security context is defined at `spec.securityContext` and applies to all containers in the pod. It supports fields like `runAsUser`, `runAsGroup`, `fsGroup`, and `runAsNonRoot`. Container-level is defined at `spec.containers[].securityContext` and overrides pod-level for that specific container. Critically, some fields are *only* valid at the container level: `allowPrivilegeEscalation`, `privileged`, `readOnlyRootFilesystem`, and `capabilities`. If you put those at the pod level, they're silently ignored — which is a common and dangerous mistake."

---

**Q: How does `fsGroup` work and when do you need it?**

"`fsGroup` sets a supplemental group ID that is applied to all mounted volumes in the pod. Kubernetes uses it to chown the volume mount point to that GID and sets the setgid bit on the directory, so files created inside inherit that group ownership. You need it whenever you have `readOnlyRootFilesystem: true` but your application still needs to write to a volume — like logs, temp files, or cached data. You mount an `emptyDir` or PVC at a specific path, set `fsGroup` to a GID your process belongs to, and the container gets write access to that volume without needing root."

---

**Q: What are Linux capabilities and how do they apply in Kubernetes?**

"Linux capabilities decompose the monolithic root privilege into about 40 discrete tokens — things like `NET_ADMIN` for network configuration, `SYS_TIME` for clock modification, `CHOWN` for file ownership changes. In Kubernetes, you configure them under `spec.containers[].securityContext.capabilities` — pod level doesn't work. Best practice is `drop: ALL` first, which clears all capability bits, then `add` only the specific capabilities your application actually requires. This means even if the container is compromised, the attacker has minimal kernel privilege to work with. Containers that need nothing special should have no capabilities at all."

---

**Q: What does `allowPrivilegeEscalation: false` actually prevent?**

"It sets the `PR_SET_NO_NEW_PRIVS` flag on the container process, which prevents any child process from gaining more privileges than the parent through mechanisms like setuid binaries. Without this, a non-root user in your container could execute a binary owned by root with the setuid bit set and suddenly have root-level access. Setting it to false means the setuid bit is ignored — the binary runs with the caller's credentials, not the file owner's. It's independent of `runAsUser`, so you need both."

---

**Q: What's the difference between `privileged: true` and adding capabilities?**

"`privileged: true` is a sledgehammer — it grants the container all Linux capabilities, disables seccomp filtering, allows direct device access, and essentially removes all isolation. It's equivalent to running a process directly on the host kernel. Adding specific capabilities is a scalpel — you surgically grant only the kernel tokens your workload needs while keeping everything else locked down. In production, `privileged: true` should only appear for infrastructure components like CNI plugins or CSI drivers, running in dedicated namespaces with strict RBAC, not for application workloads."

---

**Q: How does Kubernetes translate security context into actual enforcement?**

"The kubelet passes the pod spec to the container runtime via CRI, which translates the security context into an OCI runtime spec in a file called `config.json`. Fields like `runAsUser` become the process UID in the spec, `capabilities` become a bitmask of permitted and effective capability sets, `allowPrivilegeEscalation` maps to `noNewPrivileges: true`, and `readOnlyRootFilesystem` maps to a read-only bind mount on the root. The actual enforcement then happens at the Linux kernel level when runc or crun executes the container with those constraints."

---

## 8. Debugging

> *Fast diagnosis paths — follow the decision tree, not a checklist.*

### Primary Diagnosis Flow

```
Pod fails to start (Pending/Error state)
        ↓
kubectl describe pod <name>
        ↓
Check Events section:
  ├─ "container has runAsNonRoot and image will run as root"
  │     → runAsNonRoot: true but no runAsUser set (or image defaults to UID 0)
  │     → Fix: add runAsUser: <non-zero UID>
  │
  ├─ "Operation not permitted" / "permission denied" during startup
  │     → Missing capability required by app init
  │     → Fix: check /proc/1/status CapEff, add specific capability back
  │
  ├─ "Read-only file system"
  │     → App writes to / or /tmp at startup
  │     → Fix: mount emptyDir at the specific write path
  │
  └─ Pod starts but crashes (CrashLoopBackOff)
        ↓
     kubectl logs <pod> --previous
          ↓
     Look for: permission denied, operation not permitted, read-only filesystem
```

### Capability Debugging

```bash
# Step 1: Exec into running pod and check effective capabilities
kubectl exec -it <pod> -- cat /proc/1/status | grep -E "Cap(Eff|Prm|Inh)"
# CapInh: 0000000000000000
# CapPrm: 0000000000003000
# CapEff: 0000000000003000

# Step 2: Decode the hex bitmask on your local machine
capsh --decode=0000000000003000
# Output: = cap_net_admin,cap_sys_time

# Step 3: If a specific operation fails, trace which capability it needs
# Run with strace (if image has it) or use a debug container
kubectl debug -it <pod> --image=nicolaka/netshoot --target=<container>
strace -e trace=process <command>
# Look for EPERM errors on specific syscalls → map to capability

# Capability-to-operation reference (common):
# EPERM on chown()     → needs CHOWN
# EPERM on bind(<1024) → needs NET_BIND_SERVICE
# EPERM on stime()     → needs SYS_TIME
# EPERM on ioctl(SIOCSIFFLAGS) → needs NET_ADMIN
```

### Volume Write Permission Debugging

```bash
# Step 1: Check the mount
kubectl exec -it <pod> -- mount | grep <mountpath>
# Verify it shows rw (not ro)

# Step 2: Check ownership of the mount directory
kubectl exec -it <pod> -- ls -la /writable
# drwxr-sr-x 2 root 2000 4096  ← GID should match fsGroup

# Step 3: Check what groups the process belongs to
kubectl exec -it <pod> -- id
# uid=1000 gid=3000 groups=2000,3000  ← 2000 must be in supplemental groups

# Step 4: If GID mismatch, check fsGroup in pod spec
kubectl get pod <pod> -o jsonpath='{.spec.securityContext.fsGroup}'

# Step 5: For PVCs with existing data, check fsGroupChangePolicy
kubectl get pod <pod> -o jsonpath='{.spec.securityContext.fsGroupChangePolicy}'
# Empty means "OnRootMismatch" (default) — change to "Always" if needed
```

### Security Context Field Placement Debugging

```bash
# Check what actually landed in the pod spec (catches silent YAML errors)
kubectl get pod <pod> -o yaml | grep -A 20 securityContext

# Validate YAML before applying
kubectl apply --dry-run=server -f pod.yaml

# Check if PSS/PSA is rejecting the pod
kubectl describe pod <pod> | grep -i "security\|forbidden\|policy"

# Check namespace PSS labels
kubectl get namespace <ns> --show-labels | grep pod-security
```

### Quick Triage Commands

```bash
# Full security context dump for a running pod
kubectl get pod <pod> -o jsonpath='{.spec.securityContext}' | jq .
kubectl get pod <pod> -o jsonpath='{.spec.containers[0].securityContext}' | jq .

# Check what UID the container actually runs as
kubectl exec -it <pod> -- id

# Check filesystem mutability
kubectl exec -it <pod> -- touch /testfile && echo "writable" || echo "read-only"

# View events for security-related failures
kubectl get events --field-selector reason=Failed --sort-by='.lastTimestamp'

# Audit log check (on control plane)
grep "securityContext\|capability\|privilege" /var/log/audit/audit.log
```

---

## 9. Kill Switch

> *10-second recall — the absolute minimum to hold in memory.*

```
IDENTITY:   runAsUser + runAsGroup (pod-level) → who runs the process
SAFEGUARD:  runAsNonRoot: true → fail if image defaults to UID 0
VOLUMES:    fsGroup (pod-level only) → GID ownership of all mounts
ESCALATION: allowPrivilegeEscalation: false (container-level only) → block setuid
FILESYSTEM: readOnlyRootFilesystem: true (container-level only) → immutable root
CAPS:       drop: ALL first, then add: [only what's needed]
OVERRIDE:   container-level always beats pod-level
PLACEMENT:  capabilities/privileged/readOnlyRootFilesystem = container-level ONLY
WRITABLE:   readOnlyRootFilesystem + emptyDir mount = safe write pattern
NUCLEAR:    privileged: true = no isolation — infrastructure use only
```

---

## 10. Appendix

> *Quick reference card — commands, YAML snippets, cheatsheet.*

### Minimal Secure Pod — Copy-Paste Template

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: <name>
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
  containers:
  - name: app
    image: <image>
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
    volumeMounts:
    - name: tmp
      mountPath: /tmp
  volumes:
  - name: tmp
    emptyDir: {}
```

### Common Linux Capabilities Reference

| Capability | What It Unlocks | Default? |
|---|---|---|
| `CHOWN` | Change file owner/group | Yes |
| `SETUID` | Change process UID | Yes |
| `SETGID` | Change process GID | Yes |
| `NET_BIND_SERVICE` | Bind to ports below 1024 | Yes |
| `KILL` | Send signals to arbitrary processes | Yes |
| `NET_RAW` | Raw socket access (dangerous, drop this) | Yes |
| `NET_ADMIN` | Modify network config, routes, firewall | No |
| `SYS_TIME` | Modify system clock | No |
| `SYS_ADMIN` | Mount filesystems, kernel params (avoid) | No |
| `DAC_OVERRIDE` | Bypass file permission checks | No |

> Capabilities marked "Yes" are granted to containers by default (even without `runAsUser: 0`). Drop them explicitly if your app doesn't need them.

### Security Context Field Cheatsheet

```
spec.securityContext:            (Pod-level)
  runAsUser: <uid>
  runAsGroup: <gid>
  fsGroup: <gid>
  runAsNonRoot: true|false
  fsGroupChangePolicy: Always|OnRootMismatch
  seccompProfile:
    type: RuntimeDefault|Localhost|Unconfined

spec.containers[].securityContext:   (Container-level)
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  privileged: false
  capabilities:
    drop: ["ALL"]
    add: ["CAP_NAME"]
  runAsUser: <uid>          # overrides pod-level
  runAsGroup: <gid>         # overrides pod-level
  runAsNonRoot: true|false  # overrides pod-level
```

### Diagnostic Commands Cheatsheet

```bash
# Identity
kubectl exec -it <pod> -- id
kubectl exec -it <pod> -- whoami

# Capabilities
kubectl exec -it <pod> -- cat /proc/1/status | grep Cap
capsh --decode=<hex>

# Filesystem
kubectl exec -it <pod> -- mount | grep " / "
kubectl exec -it <pod> -- ls -la /

# Volume ownership
kubectl exec -it <pod> -- ls -la /mountpath

# Pod spec inspection
kubectl get pod <pod> -o yaml | grep -A 30 securityContext
kubectl get pod <pod> -o jsonpath='{.spec.securityContext}' | jq .
kubectl get pod <pod> -o jsonpath='{.spec.containers[0].securityContext}' | jq .

# Events
kubectl describe pod <pod> | grep -A 10 Events
kubectl get events --sort-by='.lastTimestamp' -n <ns>

# Namespace PSS policy
kubectl get ns <ns> --show-labels | grep pod-security
```

### Pod Security Standards at a Glance

```bash
# Label a namespace with Restricted PSS (most hardened)
kubectl label namespace <ns> \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/audit=restricted

# Restricted PSS requires:
#   - runAsNonRoot: true
#   - allowPrivilegeEscalation: false
#   - capabilities.drop: ["ALL"]
#   - seccompProfile: RuntimeDefault or Localhost
#   - No hostPath volumes, no host namespaces
```
