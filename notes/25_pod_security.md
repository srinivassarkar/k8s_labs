# Kubernetes Pod Security – Security Context & Linux Capabilities


---

## The Core Idea (Remember This Always)

**Linux identity is UID/GID, not usernames.** The kernel cares about numbers, not names. A container running `uid=1000` has the same privileges whether that UID is called `appuser` or doesn't appear in `/etc/passwd`.

**Root is UID 0.** The name "root" is just a label. What matters is whether the process has UID 0. `runAsNonRoot: true` prevents UID 0, stopping the process before it can do damage.

**Linux capabilities split root privileges into small, separate pieces.** Instead of being all-powerful, a process can hold just one or two capabilities like `NET_ADMIN` without having full root access. This is how we apply least-privilege at the kernel level.

**Containers are not a full security boundary by default.** Namespaces hide resources, cgroups limit CPU/memory, and capabilities restrict privileges. Without explicit security settings, a root container on a shared node can easily compromise other workloads.

**Kubernetes only translates your security settings – the actual enforcement happens in the container runtime (containerd/CRI-O) and the Linux kernel.**

---

## What Kubernetes Does and Doesn't Do

| **Constraint** | **Reality** |
|---|---|
| Default pod runs as root | If the image doesn't specify a `USER`, the container starts with UID=0 |
| `runAsNonRoot` doesn't set a UID | It only rejects UID 0. You must also set `runAsUser` to a non-zero value |
| `fsGroup` is pod‑level only | You cannot set it per‑container. It applies to all volumes mounted in the pod |
| `capabilities` is container‑level only | You cannot set it at `spec.securityContext`. Misplacing it is a silent error |
| `allowPrivilegeEscalation` defaults to `true` | Even non‑root containers can become root via setuid binaries unless you set this to `false` |
| `readOnlyRootFilesystem` blocks `/tmp` too | Apps that write to `/tmp` will break. You must mount an `emptyDir` there |
| `privileged: true` bypasses almost everything | It disables seccomp, all capability restrictions, and gives full device access – nearly same as running on the host |
| Image `USER` instruction is advisory | A Dockerfile `USER 1000` sets a default, but a pod's `runAsUser` overrides it |
| `whoami` may return "unknown" | That’s normal – it just means the UID isn’t in `/etc/passwd`. Security uses UIDs, not names |
| Capability drops don’t work well when running as root | If `runAsUser: 0`, dropping capabilities may be ineffective because root can re‑acquire them |

---

## When to Use What – Decision Guide

### Which Level to Configure

```
Need uniform restrictions for the whole pod?
└─ Yes → Use spec.securityContext (pod‑level)
   ├─ runAsUser, runAsGroup, runAsNonRoot, fsGroup

Need to override for a specific container OR use fields that only exist per‑container?
└─ Yes → Use spec.containers[].securityContext (container‑level)
   ├─ allowPrivilegeEscalation, privileged, readOnlyRootFilesystem, capabilities

Overlap? (e.g., runAsUser at both levels)
└─ Container‑level ALWAYS wins
```

### Field Applicability Quick Reference

| **Field** | **Pod Level** | **Container Level** | **Notes** |
|---|---|---|---|
| `runAsUser` | ✅ | ✅ | Container overrides Pod |
| `runAsGroup` | ✅ | ✅ | Container overrides Pod |
| `runAsNonRoot` | ✅ | ✅ | Container overrides Pod |
| `fsGroup` | ✅ | ❌ | Pod‑only; sets group ownership on volumes |
| `allowPrivilegeEscalation` | ❌ | ✅ | Container‑only |
| `privileged` | ❌ | ✅ | Container‑only |
| `readOnlyRootFilesystem` | ❌ | ✅ | Container‑only |
| `capabilities` | ❌ | ✅ | Container‑only |

### When to Use `fsGroup`

```
Container needs to write to a mounted volume (PVC or emptyDir)?
└─ Yes → Set fsGroup = the GID that the container process belongs to
   → Volume files will be owned by that GID
   → The 's' bit (setgid) on the mount directory ensures new files inherit the GID

Container filesystem must be read‑only but app needs temporary writes?
└─ Yes → Use readOnlyRootFilesystem: true + an emptyDir volume at /tmp or /writable
   → Set fsGroup so the container can write to that volume
```

### Capabilities Decision Tree

```
Does the app need any Linux capability beyond just "running as a user"?
└─ No → drop: ["ALL"] and stop.
└─ Yes → drop: ["ALL"] first, then add: ONLY the specific capability needed

Common capabilities and when to use them:
├─ Bind to port < 1024 → NET_BIND_SERVICE
├─ Modify network interfaces → NET_ADMIN
├─ Change system time → SYS_TIME
├─ Change file ownership → CHOWN
├─ Mount filesystems → SYS_ADMIN (avoid if possible)
└─ Raw socket access → NET_RAW (drop this; it's dangerous and granted by default)
```

---

## How It Actually Works (Internal Mechanics)

### Security Context → OCI Runtime Spec Pipeline

```
1. User submits pod YAML with securityContext
   ↓
2. kube-apiserver validates and stores it in etcd
   ↓
3. kube-scheduler assigns the pod to a node
   ↓
4. kubelet on that node picks up the pod spec
   ↓
5. kubelet calls the CRI (containerd or CRI-O)
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
7. OCI runtime (runc/crun) starts the container with that spec
   ↓
8. Linux kernel enforces:
   ├─ UID/GID via the process credential struct
   ├─ Capabilities via capability bitmask (effective/permitted/inheritable sets)
   ├─ No new privs via PR_SET_NO_NEW_PRIVS prctl flag
   └─ Read‑only rootfs via a bind mount with MS_RDONLY flag
```

### How `fsGroup` Works Under the Hood

When `fsGroup: 2000` is set and a volume is mounted:

1. kubelet mounts the volume (e.g., emptyDir) at the host path.
2. kubelet runs `chown :2000 /path` to set the group ownership.
3. kubelet sets the setgid bit: `chmod g+s /path`.
4. The container starts with supplemental groups that include `fsGroup` (2000).
5. Any new file created inside that directory inherits the directory's GID (thanks to setgid).
6. The container process (which belongs to group 2000 as a supplemental group) can read and write.

Result: `drwxr-sr-x root 2000` — the `s` in `sr-x` is the setgid bit.

### How `allowPrivilegeEscalation: false` Works

Linux uses the `PR_SET_NO_NEW_PRIVS` prctl flag:

- Once set, it cannot be unset – it’s inherited by all child processes.
- It prevents `execve()` from granting more privileges via setuid binaries.
- A setuid binary (owned by root) will run with the caller's UID (e.g., 1000), not root's UID, making it harmless.

Without this flag, a setuid binary would run as root and could escalate privileges.

### Linux Capability Bitmask (Simplified)

Each process has three sets of capabilities:

- **Permitted** – the maximum set the process may have.
- **Effective** – the set currently active (what the kernel checks).
- **Inheritable** – what is passed to child processes via `execve()`.

Kubernetes maps:
- `capabilities.add: ["NET_ADMIN"]` → adds CAP_NET_ADMIN to permitted + effective.
- `capabilities.drop: ["ALL"]` → clears all bits in all three sets first.

---

## Hands-On Examples

### Default (Insecure) Pod – Baseline

```yaml
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
kubectl exec -it default-pod -- id
# uid=0(root) gid=0(root) groups=0(root) ← DANGER – running as root
```

### Secure Pod with Full Security Context

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:                # Pod‑level
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    runAsNonRoot: true
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    securityContext:              # Container‑level
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
```

```bash
kubectl exec -it secure-pod -- sh
id
# uid=1000 gid=3000 groups=2000,3000
mkdir /test_dir
# mkdir: can't create directory '/test_dir': Read-only file system
whoami
# whoami: unknown uid 1000   ← normal, not an error
```

### Secure Writable Volume Pattern

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-writable
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
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
      mountPath: /writable
    - name: tmp-vol
      mountPath: /tmp
  volumes:
  - name: writable-vol
    emptyDir: {}
  - name: tmp-vol
    emptyDir: {}
```

Now `/writable` and `/tmp` are writable, but the root filesystem remains read-only.

### Linux Capabilities – Drop All, Add Selectively

```yaml
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
        - ALL
        add:
        - NET_ADMIN
        - SYS_TIME
```

```bash
kubectl exec -it cap-pod -- sh
# Test SYS_TIME
date -s "2025-06-01 12:00:00"   # may succeed depending on runtime
# Test NET_ADMIN
ip link set eth0 down            # should succeed in permissive environments
# Test CHOWN (not added)
touch myfile && chown 1000:1000 myfile
# chown: Operation not permitted ← correct because CHOWN was dropped
```

### Strictest Hardening Profile (Production Baseline)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hardened-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    runAsGroup: 10001
    fsGroup: 10001
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
    volumeMounts:
    - name: tmp
      mountPath: /tmp
  volumes:
  - name: tmp
    emptyDir:
      medium: Memory
      sizeLimit: 50Mi
```

---

## Production Architecture & Patterns

### Layered Defense Architecture

```
Layer 1: Pod Security Standards (cluster‑wide admission)
   └─ Enforced via namespace labels:
      pod-security.kubernetes.io/enforce: restricted

Layer 2: Pod Security Context (per‑pod/container YAML)
   └─ Explicit runAsUser, runAsNonRoot, capabilities.drop: ALL

Layer 3: OPA Gatekeeper / Kyverno (policy engine)
   └─ Rejects pods missing required security fields

Layer 4: Seccomp / AppArmor (kernel‑level syscall filtering)
   └─ seccompProfile.type: RuntimeDefault blocks dangerous syscalls

Layer 5: Runtime Security (e.g., Falco)
   └─ Detects anomalous behavior at runtime
```

### Pod Security Standards Quick Reference

| **Level** | **What It Enforces** |
|---|---|
| `privileged` | No restrictions |
| `baseline` | Blocks privileged containers, host namespaces, dangerous volume types |
| `restricted` | Requires non‑root, drops ALL capabilities, requires seccomp |

```bash
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted
```

### Multi-Container Pod Security Inheritance

```yaml
spec:
  securityContext:                 # pod‑level defaults
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
  containers:
  - name: app                      # inherits pod settings
    securityContext:
      readOnlyRootFilesystem: true
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
  - name: log-shipper              # needs one extra capability
    securityContext:
      readOnlyRootFilesystem: true
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
        add: ["NET_BIND_SERVICE"]
```

### When to Use `privileged: true` (Rare Exceptions)

**Legitimate use cases:**
- CNI plugins (Calico, Cilium node agents)
- CSI drivers (storage drivers needing device access)
- Monitoring agents needing kernel access (Falco, eBPF)
- Node management daemonsets (device drivers, CUDA)

**Best practice:** Run privileged containers in a dedicated namespace with strict RBAC, NetworkPolicy, and audit logging.

---

## Common Mistakes (What Actually Breaks)

### ❌ Mistake 1 – `readOnlyRootFilesystem: true` Breaks the App

**Symptom:** Pod crashes. Logs show permission denied on `/tmp` or `/var/run`.

**Fix:** Mount `emptyDir` volumes at those paths.
```yaml
volumeMounts:
- name: tmp
  mountPath: /tmp
volumes:
- name: tmp
  emptyDir: {}
```

### ❌ Mistake 2 – `runAsNonRoot: true` Without `runAsUser`

**Symptom:** Pod fails to start: "container has runAsNonRoot and image will run as root".

**Fix:** Always pair `runAsNonRoot` with an explicit `runAsUser`.
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000   # required
```

### ❌ Mistake 3 – Putting `capabilities` at Pod Level

**Symptom:** YAML applies without error, but capabilities are not enforced.

**Fix:** Place `capabilities` under the container's `securityContext`, not the pod's.
```yaml
# WRONG
spec:
  securityContext:
    capabilities:     # ignored
      drop: ["ALL"]

# CORRECT
spec:
  containers:
  - name: app
    securityContext:
      capabilities:   # correct
        drop: ["ALL"]
```

### ❌ Mistake 4 – `fsGroup` Not Applied to Pre-Existing Volume Data

**Symptom:** Volume mounts correctly but existing files are not writable.

**Fix:** Use `fsGroupChangePolicy: "Always"` to force recursive chown on every mount.
```yaml
securityContext:
  fsGroup: 2000
  fsGroupChangePolicy: "Always"
```

### ❌ Mistake 5 – Dropping ALL Capabilities Breaks App Startup

**Symptom:** Container CrashLoopBackOff with "operation not permitted".

**Fix:** Start with `drop: ["ALL"]`, then add back only the needed capabilities.
```bash
# Check which capabilities a running process has
kubectl exec -it <pod> -- cat /proc/1/status | grep Cap
capsh --decode=0000000000003000
```

### ❌ Mistake 6 – `allowPrivilegeEscalation` Defaults to True

**Symptom:** Security audit flags containers that don't explicitly set it to `false`. The team says "we're non-root, so we're safe."

**Fix:** Always explicitly set `allowPrivilegeEscalation: false`. Even non-root UIDs (e.g., 1000) can execute setuid binaries to become root if this is not disabled.

---

## Interview Answers (Simple, Ready to Say)

### Q: What is the difference between `runAsUser` and `runAsNonRoot`?

> "`runAsUser` sets the numeric UID for the container process. `runAsNonRoot` is a validation check – it rejects the pod if the container would run as UID 0. However, `runAsNonRoot` doesn't set a UID by itself; you must also specify `runAsUser` with a non-zero value."

### Q: What does `allowPrivilegeEscalation` do?

> "It controls whether a process can gain more privileges than its parent through setuid binaries or execve. Setting it to `false` blocks these privilege escalations, so even if a non-root user runs a setuid root binary, it will run with the user's own UID, not root's. It's a safety net independent of the container's UID."

### Q: Why should I drop all capabilities and add only what I need?

> "Because capabilities are inherited by default – a container gets a broad set of Linux capabilities. Dropping all removes unnecessary privileges and reduces the attack surface. Then you add back only the exact capabilities the application requires (e.g., `NET_BIND_SERVICE`). This follows the principle of least privilege and is a best practice for security-hardened pods."

### Q: How do `fsGroup` and `supplementalGroups` differ?

> "`fsGroup` is a special supplemental group that Kubernetes uses to set ownership and setgid bits on mounted volumes, ensuring that the container process can write to those volumes. It's set at the pod level and applies to all volumes. `supplementalGroups` is a general list of additional GIDs that the container process will have, but they don't automatically change volume ownership. For volume access, `fsGroup` is usually sufficient."

### Q: Can a non-root container be privileged?

> "Yes, you can set `privileged: true` on a container that runs as non-root. This grants it all Linux capabilities and bypasses many restrictions, even though the effective UID is not 0. It's still a serious security risk. You should only use `privileged: true` for system-level daemons (like CNI agents) and never for application containers."

### Q: What is the difference between `securityContext` at pod level vs container level?

> "Pod-level settings apply to all containers in the pod and are a good place for uniform rules like `runAsUser`, `runAsGroup`, and `fsGroup`. Container-level settings override pod-level ones and are required for fields like `capabilities`, `allowPrivilegeEscalation`, and `readOnlyRootFilesystem` because those are specific to each container."

---

## Debugging – Fast Diagnosis

### Pod Fails to Start – Permission Errors

```
Check kubectl describe pod
│
├── Event: "container has runAsNonRoot and image will run as root"
│   └── Fix: add runAsUser: 1000 under securityContext
│
├── Event: "cannot create directory: read-only file system"
│   └── Fix: mount emptyDir at /tmp or add volume for writable paths
│
├── Event: "operation not permitted"
│   └── Check capabilities – you may have dropped a required one
│   └── Check allowPrivilegeEscalation: false – may block setuid binaries
│
└── Event: "chown ... operation not permitted"
    └── Fix: set fsGroupChangePolicy: "Always"
```

### Check Effective Security Settings on a Running Pod

```bash
# Show the full pod spec (including defaulted fields)
kubectl get pod <name> -o yaml

# Check the process UID/GID inside the container
kubectl exec -it <pod> -- id

# List effective capabilities
kubectl exec -it <pod> -- cat /proc/1/status | grep Cap
capsh --decode=<hex-value>

# Check mount permissions (for fsGroup)
kubectl exec -it <pod> -- ls -l <mountPath>
```

### Verify PodSecurity Standards Enforcement

```bash
# Check labels on the namespace
kubectl get ns <ns> --show-labels

# Dry-run to see if a pod would be blocked
kubectl apply --dry-run=server -f pod.yaml
```

---

## 30-Second Quick Revision

### The Essentials

```
Security Context = define UID/GID, capabilities, filesystem access
Linux UID = numeric, not a name – root is 0
runAsNonRoot = validation check, must be paired with runAsUser
fsGroup = group ownership on volumes (pod‑level only)
capabilities = per‑container, must be under containers[].securityContext
allowPrivilegeEscalation = always set to false (default is true!)
readOnlyRootFilesystem = true requires emptyDir for /tmp and other writable paths
```

### Golden Rules

```
Drop ALL capabilities, add only what you need
Set runAsNonRoot: true AND runAsUser: 1000+
Set allowPrivilegeEscalation: false
Use readOnlyRootFilesystem whenever possible
Never run application containers as privileged
```

### The Layered Approach

```
Pod Security Standards (namespace level)
    ↓
Security Context (pod/container YAML)
    ↓
Policy engines (Kyverno/Gatekeeper)
    ↓
Seccomp / AppArmor (kernel level)
    ↓
Runtime monitoring (Falco)
```

---

## Appendix – Quick Reference

### SecurityContext Fields at a Glance

| **Field** | **Level** | **Purpose** |
|---|---|---|
| `runAsUser` | Pod/Container | UID of the process |
| `runAsGroup` | Pod/Container | GID of the process |
| `runAsNonRoot` | Pod/Container | Reject if UID=0 |
| `fsGroup` | Pod only | Supplemental group for volume ownership |
| `fsGroupChangePolicy` | Pod only | When to apply chown (`Always` / `OnRootMismatch`) |
| `supplementalGroups` | Pod only | Additional GIDs for the process |
| `allowPrivilegeEscalation` | Container only | Block setuid privilege gain |
| `privileged` | Container only | Full host access (use rarely) |
| `readOnlyRootFilesystem` | Container only | Make rootfs read‑only |
| `capabilities` | Container only | Add/drop Linux capabilities |
| `seccompProfile` | Pod/Container | Syscall filtering profile |

### Capabilities Reference (Common Ones)

| **Capability** | **What It Allows** |
|---|---|
| `NET_BIND_SERVICE` | Bind to ports below 1024 |
| `NET_ADMIN` | Configure network interfaces |
| `SYS_TIME` | Change system clock |
| `CHOWN` | Change file ownership |
| `SYS_ADMIN` | Wide range of system admin (avoid if possible) |
| `NET_RAW` | Raw sockets (dangerous, drop by default) |
| `KILL` | Send signals to any process |
| `SETUID` | Set effective UID (dangerous) |

### Useful Commands Cheatsheet

```bash
# Check current security settings of a pod
kubectl get pod <name> -o yaml | grep -A 10 securityContext

# Decode capability bits
kubectl exec -it <pod> -- cat /proc/1/status | grep Cap
capsh --decode=<hex>

# Check PodSecurity Standards on a namespace
kubectl get ns <ns> --show-labels | grep pod-security

# Test with dry-run
kubectl apply --dry-run=server -f pod.yaml
```

---

