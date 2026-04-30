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
securityCon