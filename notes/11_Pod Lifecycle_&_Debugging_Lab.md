# Kubernetes Pod Lifecycle & Debugging

> **Scope:** Pod Phases · Container States · Restart Policies · Image Pull · Termination · Scheduling Failures · CrashLoopBackOff · OOMKilled · Production Debugging

---

## 0. First Principles

> *The mental model that never changes, regardless of Kubernetes version.*

**A Pod is a state machine, not a process.**
Kubernetes does not "run" pods the way an OS runs processes. It continuously reconciles *desired state* (spec) against *observed state* (status) and drives the pod toward the declared outcome. Every status field you see — phase, conditions, container states — is the control plane's best current observation, not a guarantee.

**The immutable truths:**

| Principle | What it means |
|-----------|---------------|
| **Phase is coarse** | Pod phase (Pending/Running/Succeeded/Failed/Unknown) is a high-level summary. Container states are the granular truth. |
| **restartPolicy governs container recovery** | It applies to all containers in the pod. You cannot set different policies per container. |
| **kubelet is the executor** | The API server records intent. The kubelet on the assigned node actually starts, monitors, and restarts containers. |
| **Backoff is exponential** | Restart delays start at 10s and double up to 5 minutes. This backoff resets after 10 minutes of successful running. |
| **SIGTERM before SIGKILL** | Kubernetes always attempts graceful shutdown first. Force-delete bypasses this for the API object but the node may still send SIGTERM. |
| **Scheduling is all-or-nothing** | A pod schedules to exactly one node. If no node satisfies all constraints, the pod waits in Pending indefinitely. |
| **ImagePullPolicy has defaults** | The default is NOT always `IfNotPresent` — it depends on the image tag. `latest` or no tag → `Always`. Explicit tag → `IfNotPresent`. |

**The two separate concerns you must never conflate:**

```
Phase      → what the Pod is doing (Pending, Running, Succeeded, Failed, Unknown)
Conditions → why it is or isn't ready (PodScheduled, Initialized, ContainersReady, Ready)
```

Understanding this split is what separates CKA candidates from senior engineers.

---

## 1. Reality Constraints

> *What Kubernetes actually enforces — not what the docs imply.*

### Phase vs. Display Status — The Critical Distinction

```
Internal phase (spec.status.phase) → what Kubernetes sees
kubectl get pods STATUS column     → what kubectl shows you
```

| Internal Phase | `kubectl get pods` STATUS | When |
|---------------|--------------------------|------|
| `Pending` | `Pending` | Not yet scheduled or image not pulled |
| `Running` | `Running` | At least one container running |
| `Succeeded` | `Completed` | All containers exited 0, restartPolicy≠Always |
| `Failed` | `Error` or `CrashLoopBackOff` | At least one container exited non-zero |
| `Unknown` | `Unknown` | kubelet communication lost |

**Succeeded ≠ Completed.** `Succeeded` is the internal phase. `Completed` is what `kubectl` displays. This distinction appears in CKA tasks that ask you to check pod phase programmatically.

```bash
# Internal phase — use this in scripts and exam tasks
kubectl get pod <pod> -o jsonpath='{.status.phase}'
# → Succeeded

# Display status — what you see in kubectl get pods
kubectl get pods
# → Completed
```

### Container States — The Real Signal

A pod in `Running` phase can have containers in any of these states:

| Container State | Meaning | Your next action |
|-----------------|---------|-----------------|
| `Waiting` | Not yet started (pulling image, init running) | Check `reason` field |
| `Running` | Process is executing | Normal |
| `Terminated` | Process exited | Check `exitCode` and `reason` |

```bash
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].state}'
```

### Restart Policy Enforcement

| restartPolicy | Container exits 0 | Container exits non-0 | Use case |
|--------------|-------------------|----------------------|----------|
| `Always` | Restarts | Restarts | Long-running services |
| `OnFailure` | Does NOT restart | Restarts | Jobs, batch tasks |
| `Never` | Does NOT restart | Does NOT restart | Debug pods, one-shot |

**Critical constraint:** `restartPolicy` applies at the Pod level, not per-container. All containers share it.

**Deployments always use `Always`.** You cannot set `restartPolicy: Never` in a Deployment — the controller will reject it.

### ImagePullPolicy Defaults (Frequently Tested)

```
image: nginx          → tag is implicitly "latest" → ImagePullPolicy: Always
image: nginx:latest   → explicit "latest"          → ImagePullPolicy: Always
image: nginx:1.25     → explicit version tag       → ImagePullPolicy: IfNotPresent
image: nginx@sha256:… → digest pinned              → ImagePullPolicy: IfNotPresent
```

**The exam trap:** Students set `imagePullPolicy: IfNotPresent` on a `latest` image and wonder why every pod restart still pulls — they forgot `Always` is the default for `latest`.

### Termination Grace Period

```
Default terminationGracePeriodSeconds = 30
Range: 0 to any positive integer
```

Setting `terminationGracePeriodSeconds: 0` does not mean "no signal" — it means SIGTERM and SIGKILL are sent simultaneously (effectively skipping graceful shutdown). This is the same behavior as `--grace-period=0` in kubectl.

### CrashLoopBackOff Backoff Schedule

```
Restart 1: 10s delay
Restart 2: 20s delay
Restart 3: 40s delay
Restart 4: 80s delay
Restart 5: 160s delay
Restart 6+: 300s (5 minutes, capped)
Reset: after 10 continuous minutes of Running
```

The pod stays `CrashLoopBackOff` during the backoff wait. It transitions to `Running` briefly when the container actually starts, then back to `CrashLoopBackOff` if it crashes again.

### What Kubernetes Does NOT Do

| Misconception | Reality |
|---------------|---------|
| "Running phase = all containers healthy" | Running = at least one container running. Others may be crashing. |
| "Force delete immediately frees the node" | `--force` removes the API object. The kubelet still runs the shutdown sequence on the node. |
| "OOMKilled is a Kubernetes decision" | The Linux kernel OOM killer terminates the process. Kubernetes just observes and records it. |
| "Pending means scheduling failed" | Pending means scheduling hasn't succeeded yet. It might still succeed once a node becomes available. |
| "ImagePullBackOff means wrong credentials" | It could be wrong name, wrong tag, private registry, or network unreachable. |

---

## 2. Decision Logic

> *Clear rules for restartPolicy, imagePullPolicy, and debugging entry points.*

### restartPolicy Selection

```
What does this pod do after its main process exits?

├── Should NEVER restart (run once, exit, done)?
│     └── restartPolicy: Never
│           → Debug pods, one-shot scripts, exam investigation pods

├── Should restart ONLY on failure (batch processing)?
│     └── restartPolicy: OnFailure
│           → Jobs, CronJobs, migration runners

└── Should ALWAYS restart regardless of exit code?
      └── restartPolicy: Always
            → Deployments, StatefulSets, DaemonSets
            → Any long-running service
```

### imagePullPolicy Selection

```
Is this a production deployment with a pinned version?
├── YES → image: app:v2.3.1 + imagePullPolicy: IfNotPresent
│         (fast starts, deterministic, cached on node)
└── NO  → image: app:latest + imagePullPolicy: Always
          (always fresh, slower start, good for dev/debug)

Special case — digest pinning (most secure):
└── image: app@sha256:abc123 → Always resolves to exact layer
```

### Debugging Entry Point by Symptom

| Symptom | Likely cause category | First command |
|---------|----------------------|---------------|
| `Pending` | Scheduling constraint | `kubectl describe pod` → Events |
| `CrashLoopBackOff` | Container crash | `kubectl logs --previous` |
| `ErrImagePull` / `ImagePullBackOff` | Registry issue | `kubectl describe pod` → Events |
| `OOMKilled` | Memory limit | `kubectl describe pod` → `lastState` |
| `Running` but unhealthy | App bug / probe config | `kubectl logs` + probe config |
| `Unknown` | Node communication lost | `kubectl get nodes` |
| `Terminating` stuck | Finalizer or force needed | `kubectl delete --force --grace-period=0` |

---

## 3. Internal Working

> *How Kubernetes actually drives the pod lifecycle, step by step.*

### Full Pod Lifecycle — Control Flow

```
1. User submits Pod spec → API Server validates → stored in etcd

2. Scheduler watch loop detects unscheduled pod (spec.nodeName = "")
   └── Runs predicates: node selector, taints/tolerations, resource fit, affinity
   └── Runs priorities: scoring remaining nodes
   └── Writes nodeName to pod spec → etcd

3. kubelet on assigned node detects pod via watch
   └── Creates pod sandbox (pause container → network namespace + cgroup)
   └── Pulls image if needed (respects imagePullPolicy)
   └── Starts initContainers sequentially (if any)
   └── Starts all containers[] simultaneously

4. kubelet runs probes (if configured):
   └── startupProbe  → gates liveness/readiness until app is started
   └── livenessProbe → kills container if unhealthy (triggers restartPolicy)
   └── readinessProbe → gates Ready condition (affects Service endpoints)

5. Container exits:
   └── exitCode 0 → restartPolicy governs
   └── exitCode non-0 → restartPolicy governs
   └── OOM → kernel kills process, Kubernetes sees Terminated/OOMKilled

6. Pod termination (kubectl delete):
   └── API: grace period set, pod enters Terminating
   └── kubelet sends SIGTERM to PID 1 of each container
   └── preStop hook executes (if configured)
   └── After terminationGracePeriodSeconds → SIGKILL
   └── kubelet removes pod sandbox → releases IP, volumes
   └── API Server removes pod object from etcd
```

### The Scheduler's Decision: Why Pods Stay Pending

```
Scheduler runs two phases for every unscheduled pod:

Phase 1 — Filtering (hard constraints, all must pass)
  ├── NodeSelector / nodeAffinity
  ├── Taints / Tolerations
  ├── Resource requests (CPU, memory, ephemeral storage)
  ├── Volume availability (PVC binding)
  └── Pod topology constraints

Phase 2 — Scoring (soft constraints, pick best node)
  ├── Least requested resources
  ├── Node affinity weight
  └── Inter-pod affinity

If Phase 1 eliminates ALL nodes → pod stays Pending
```

### CrashLoopBackOff — The Exact Sequence

```
t=0s    Container starts
t=1s    Container exits (non-0)
t=0s    kubelet observes exit → schedules restart (10s delay)
t=10s   Container starts again
t=10.1s Container exits again
t=10s   kubelet schedules restart (20s delay)
t=30s   Container starts again
...     (doubles each time, caps at 300s)

STATUS oscillates:
  Running (briefly) → Error → CrashLoopBackOff (waiting) → Running → ...
```

### SIGTERM → SIGKILL Termination Flow

```
kubectl delete pod nginx
         │
         ▼
API Server sets deletionTimestamp + deletionGracePeriodSeconds=30
         │
         ▼
kubelet sees deletionTimestamp
         │
         ├──▶ Runs preStop hook (if configured)
         │
         ├──▶ Sends SIGTERM to PID 1 of each container
         │         (app should catch this and drain/cleanup)
         │
         ├──▶ Waits terminationGracePeriodSeconds (30s default)
         │
         └──▶ Sends SIGKILL (cannot be caught or ignored)
                    │
                    ▼
              kubelet removes sandbox
              API Server removes etcd object
```

### Image Pull Decision Tree (Inside kubelet)

```
kubelet needs to start container with image X:tag

├── imagePullPolicy: Always
│     └── Always attempt registry pull
│           ├── Success → start container
│           └── Failure → ErrImagePull → backoff → ImagePullBackOff

├── imagePullPolicy: IfNotPresent
│     ├── Image cached on node? → use cache → start container
│     └── Not cached → pull from registry
│           ├── Success → cache + start container
│           └── Failure → ErrImagePull → backoff → ImagePullBackOff

└── imagePullPolicy: Never
      ├── Image cached on node? → use cache → start container
      └── Not cached → ErrImageNeverPull → pod stays in Waiting
```

---

## 4. Hands-On

> *Production-quality YAML + commands. Nothing simplified.*

### Observe Pod Lifecycle Phases

```bash
# Quick pod — may skip Running phase entirely
kubectl run ubuntu-ls --image=ubuntu --restart=Never -- ls

# Longer pod — observe all phases
kubectl run ubuntu-sleep --image=ubuntu --restart=Never -- sleep 30

# Watch in real time (separate terminal)
kubectl get pods -w

# Check internal phase vs display status
kubectl get pod ubuntu-sleep -o jsonpath='{.status.phase}'
kubectl describe pod ubuntu-sleep | grep "Status:"
```

### Restart Policy Lab YAML

```yaml
# restart-never.yaml — exits and stays dead
apiVersion: v1
kind: Pod
metadata:
  name: restart-never-demo
spec:
  restartPolicy: Never
  containers:
  - name: fail-container
    image: busybox:1.36
    command: ["sh", "-c", "echo 'exiting with failure'; exit 1"]
```

```yaml
# restart-always.yaml — crashes into CrashLoopBackOff
apiVersion: v1
kind: Pod
metadata:
  name: restart-always-demo
spec:
  restartPolicy: Always
  containers:
  - name: fail-container
    image: busybox:1.36
    command: ["sh", "-c", "echo 'crashing'; exit 1"]
```

```yaml
# restart-onfailure.yaml — restarts on non-zero, stops on zero
apiVersion: v1
kind: Pod
metadata:
  name: restart-onfailure-demo
spec:
  restartPolicy: OnFailure
  containers:
  - name: job-container
    image: busybox:1.36
    command: ["sh", "-c", "echo 'job done'; exit 0"]
```

### ImagePullBackOff Simulation

```yaml
# bad-image.yaml — wrong tag triggers ImagePullBackOff
apiVersion: v1
kind: Pod
metadata:
  name: bad-image-demo
spec:
  containers:
  - name: app
    image: nginx:this-tag-does-not-exist
    imagePullPolicy: Always
```

### Private Registry with imagePullSecret

```bash
# Step 1: Create the registry secret
kubectl create secret docker-registry regcred \
  --docker-server=myregistry.example.com \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=me@example.com
```

```yaml
# private-image.yaml — uses imagePullSecret
apiVersion: v1
kind: Pod
metadata:
  name: private-registry-demo
spec:
  imagePullSecrets:
  - name: regcred
  containers:
  - name: app
    image: myregistry.example.com/myapp:v1.0.0
    imagePullPolicy: IfNotPresent
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "256Mi"
```

### Termination Grace Period

```yaml
# graceful-shutdown.yaml — preStop hook + custom grace period
apiVersion: v1
kind: Pod
metadata:
  name: graceful-demo
spec:
  terminationGracePeriodSeconds: 60
  containers:
  - name: app
    image: nginx:1.25
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "nginx -s quit; sleep 5"]
    ports:
    - containerPort: 80
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
```

### Pending Pod — NodeSelector Mismatch

```yaml
# pending-demo.yaml — will stay Pending until label added
apiVersion: v1
kind: Pod
metadata:
  name: pending-demo
spec:
  nodeSelector:
    disktype: ssd
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
```

```bash
# Fix: add label to node
kubectl label node node01 disktype=ssd

# Remove label
kubectl label node node01 disktype-

# Watch pod transition from Pending to Running
kubectl get pods -w
```

### OOMKilled Simulation

```yaml
# oom-demo.yaml — memory limit far below what stress needs
apiVersion: v1
kind: Pod
metadata:
  name: oom-demo
spec:
  containers:
  - name: memory-hog
    image: polinux/stress
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "200M", "--vm-hang", "1"]
    resources:
      requests:
        memory: "50Mi"
      limits:
        memory: "64Mi"    # container needs 200M, limit is 64M → OOMKilled
```

```bash
# Confirm OOMKill
kubectl describe pod oom-demo | grep -A 5 "Last State"
# → Reason: OOMKilled
# → Exit Code: 137
```

### Full Debug Command Suite

```bash
# --- Phase 1: broad view ---
kubectl get pods
kubectl get pods -o wide                          # adds Node, IP columns
kubectl get pods --all-namespaces                 # if you don't know the namespace
kubectl get events --sort-by='.lastTimestamp'     # cluster-wide events, newest last

# --- Phase 2: describe ---
kubectl describe pod <pod>                        # full state, conditions, events

# --- Phase 3: logs ---
kubectl logs <pod>                                # current container stdout/stderr
kubectl logs <pod> --previous                     # last terminated container
kubectl logs <pod> -f                             # follow (tail -f equivalent)
kubectl logs <pod> --tail=50                      # last N lines
kubectl logs <pod> -c <container>                 # multi-container pods

# --- Phase 4: exec ---
kubectl exec -it <pod> -- sh                      # shell into container
kubectl exec -it <pod> -c <container> -- sh       # specific container

# --- Phase 5: precise state queries ---
kubectl get pod <pod> -o jsonpath='{.status.phase}'
kubectl get pod <pod> -o jsonpath='{.status.conditions}'
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].state}'
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].lastState}'
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].restartCount}'

# --- Phase 6: cleanup ---
kubectl delete pod <pod>
kubectl delete pod <pod> --grace-period=0 --force    # stuck Terminating pods
```

---

## 5. Production Flow

> *Real-world architecture and design patterns with rationale.*

### Pattern 1 — Graceful Shutdown for Web Services

In production, an nginx or app server receiving SIGTERM must drain in-flight connections before exiting. Default behavior (kill immediately on SIGTERM) causes 5xx errors for active requests.

```
kubectl delete pod / rolling update
         │
         ├──▶ Service removes pod from endpoints (readinessProbe fails or endpoint deleted)
         │         ← traffic stops flowing to this pod
         │
         ├──▶ SIGTERM sent to container
         │
         ├──▶ preStop hook: nginx -s quit (waits for active connections to finish)
         │
         ├──▶ Application finishes in-flight requests
         │
         └──▶ Container exits 0 (within grace period)
               terminationGracePeriodSeconds should be > longest expected request duration
```

**Rule of thumb:** `terminationGracePeriodSeconds` = max expected request duration + 10s buffer.

---

### Pattern 2 — CrashLoopBackOff in Production Deployments

Real production CrashLoopBackOff is almost never a bad command. It is almost always:

```
├── Missing environment variable (app fails to parse config on startup)
├── Database unreachable (app exits rather than waiting)
├── TLS cert mismatch (app validates and aborts)
├── OOMKilled (app needs more memory than limit allows)
└── Secret not mounted (volume not injected, app can't read credentials)
```

**Production response workflow:**

```
1. kubectl get pods → confirm CrashLoopBackOff
2. kubectl logs <pod> --previous → what did the app say before dying?
3. kubectl describe pod → check lastState.terminated.reason + exitCode
4. exitCode 137 → OOMKilled → increase memory limit
5. exitCode 1 → application error → read logs carefully
6. exitCode 126/127 → command not found → image or command issue
7. kubectl exec -it <pod> -- sh → if container stays up long enough
   OR use kubectl debug (ephemeral containers) in K8s 1.23+
```

---

### Pattern 3 — Scheduling Failure in Production

Production clusters have taints on nodes (GPU nodes, spot nodes, control-plane nodes). Pods without correct tolerations silently stay `Pending`.

```
Common taint/toleration mismatches:

Node: control-plane taint
  node-role.kubernetes.io/control-plane:NoSchedule
  → Pods without this toleration cannot schedule here

Node: GPU node taint
  nvidia.com/gpu=present:NoSchedule
  → Only ML pods with toleration can use GPU nodes

Node: spot instance taint
  cloud.google.com/gke-spot=true:NoSchedule
  → Only fault-tolerant workloads should tolerate this
```

**Diagnosis:**

```bash
kubectl describe pod <pending-pod> | grep -A 5 "Events:"
# "0/5 nodes are available: 2 node(s) had taint {node-role.kubernetes.io/control-plane: }, that the pod didn't tolerate"
# → Add toleration or change nodeSelector
```

---

### Pattern 4 — imagePullPolicy in CI/CD Pipelines

```
Development environment:
  image: myapp:latest
  imagePullPolicy: Always
  → Always get latest build, even if tag name unchanged

Staging / Production:
  image: myapp:v2.3.1
  imagePullPolicy: IfNotPresent
  → Deterministic, fast pod starts, no surprise updates

GitOps pattern:
  image: myapp@sha256:abc123def456
  → Immutable, cryptographically guaranteed, zero ambiguity
```

---

## 6. Mistakes

> *What actually breaks in real systems — root cause + fix.*

### Mistake 1 — Reading Phase Instead of Container State

**What happens:** Pod shows `Running`, engineer assumes everything is healthy. Sidecar container is CrashLoopBackOff.

```bash
# Wrong check
kubectl get pods   # → Running  (misleading)

# Correct check
kubectl get pod <pod> -o jsonpath='{range .status.containerStatuses[*]}{.name}: ready={.ready} restarts={.restartCount}{"\n"}{end}'
```

**Fix:** Always check container-level states, not just pod phase, when debugging.

---

### Mistake 2 — Force Deleting Stateful Pods

**Root cause:** StatefulSet pod stuck in `Terminating`. Engineer runs `kubectl delete pod --force --grace-period=0`.

**What happens:** API object is deleted instantly. If the node is partway through shutdown, two pods with the same identity may briefly exist, corrupting shared state (databases, distributed locks).

**Fix:** For StatefulSets, only force-delete after confirming the node is unreachable (`kubectl get nodes` → `NotReady`) and the pod cannot possibly still be running.

---

### Mistake 3 — `restartPolicy: Never` on a Deployment

**Root cause:** Developer copies a debug pod spec into a Deployment template.

**What happens:** Deployment controller rejects it. Error: `Unsupported value: "Never": supported values: "Always"`.

**Fix:** Deployments enforce `restartPolicy: Always`. Use `restartPolicy: OnFailure` or `Never` only in standalone Pods or Jobs.

---

### Mistake 4 — Misreading OOMKilled Exit Code

**Root cause:** Container exits with code 137 (128 + signal 9). Engineer reads logs, finds nothing wrong. Doesn't realize the process was killed by the kernel before it could log the error.

```bash
kubectl describe pod <pod>
# Last State: Terminated
#   Reason: OOMKilled
#   Exit Code: 137
```

**Fix:** Check `Reason: OOMKilled` in `lastState`, not exit code alone. Increase `resources.limits.memory`. Set `requests` equal to `limits` for predictable scheduling.

---

### Mistake 5 — Not Using `--previous` on CrashLoopBackOff

**What happens:** Engineer runs `kubectl logs <pod>`. Gets nothing or gets logs from the current (briefly running) container. Misses the actual error from the crash.

```bash
# Wrong — logs from current container (may be empty if it hasn't started yet)
kubectl logs crashloop-pod

# Correct — logs from last terminated container
kubectl logs crashloop-pod --previous
```

---

### Mistake 6 — Assuming `--force --grace-period=0` is Instant Cluster-Wide

**Root cause:** `kubectl delete pod --force --grace-period=0` removes the API object immediately. But the kubelet on the node still has up to several seconds of cleanup. The pod may still appear in `kubectl get pods` briefly on the node's local cache.

**Fix:** After force delete, verify with `kubectl get pod <name>` returns `NotFound`. Do not re-create immediately assuming the IP and volumes are free.

---

### Mistake 7 — Scheduling Failure Due to Resource Requests, Not Actual Usage

**Root cause:** Pod requests `8Gi` memory but only uses `512Mi`. All nodes have 4Gi free. Pod stays Pending forever.

**Observation:**
```bash
kubectl describe pod <pod>
# Events: 0/3 nodes are available: 3 Insufficient memory
```

**Fix:** Set resource `requests` based on actual P95 usage (from metrics), not worst-case theoretical. `limits` can be higher than `requests` for burstable QoS.

---

### Mistake 8 — preStop Hook Time Counts Against Grace Period

**Root cause:** Engineer sets `terminationGracePeriodSeconds: 30` and `preStop` hook that sleeps 30 seconds. Application gets 0 seconds to shut down after hook finishes.

**Fix:** `terminationGracePeriodSeconds` is the total budget for `preStop` + process shutdown. Set it to `preStop duration + expected drain time + buffer`.

---

## 7. Interview Answers

> *Compressed, verbatim-ready answers for common interview questions.*

---

**Q: Walk me through the Kubernetes pod lifecycle from `kubectl apply` to the pod running.**

> "When you apply a pod spec, the API server validates it and writes it to etcd. The scheduler watches for pods with no node name assigned, runs its filtering and scoring phases to find the best node, and writes the node name back to the pod spec. The kubelet on that node detects the assignment, creates the pod sandbox — which is the pause container that holds the network and IPC namespaces — then pulls the image if needed, runs any init containers sequentially, and finally starts all application containers simultaneously. The kubelet then begins executing any configured probes: startup probes first to gate the others, then liveness probes to kill unhealthy containers, and readiness probes to control whether the pod receives traffic from Services."

---

**Q: What is the difference between pod phase and container state?**

> "Pod phase is a coarse, pod-level summary — Pending, Running, Succeeded, Failed, or Unknown. It tells you the overall situation. Container state is the granular, per-container truth — Waiting, Running, or Terminated — and it includes the reason and exit code. A pod can be in Running phase while a container inside it is in Terminated state due to a crash. That's exactly the CrashLoopBackOff scenario: the phase says Running because at least one container is active, but the specific container is actually cycling between brief Running and Terminated states. The distinction matters in debugging — you should always check container state, not just pod phase."

---

**Q: What is CrashLoopBackOff and how does the backoff work?**

> "CrashLoopBackOff means a container is repeatedly crashing and the kubelet is applying an exponential backoff before restarting it. The first restart waits 10 seconds, the second 20, then 40, 80, 160, and it caps at 5 minutes. The backoff resets after the container runs successfully for 10 consecutive minutes. The term 'CrashLoopBackOff' is actually a display status in kubectl, not a real pod phase — the internal phase stays Running or Failed. The most common production causes are a missing environment variable the app needs at startup, a database being unreachable so the app exits rather than waiting, OOMKilled because the memory limit is too low, or a missing mounted secret."

---

**Q: What happens when you run `kubectl delete pod`?**

> "Kubernetes sets a deletion timestamp on the pod object and starts the grace period countdown, defaulting to 30 seconds. The kubelet sees this and runs any preStop lifecycle hook configured on the containers. Then it sends SIGTERM to PID 1 of each container — this is the signal the application should catch to drain connections and shut down cleanly. After the terminationGracePeriodSeconds expires, if the containers are still running, the kubelet sends SIGKILL, which the process cannot catch or ignore. Then the kubelet tears down the pod sandbox, releases the IP address and volumes, and the API server removes the pod object from etcd. Important nuance: the preStop hook time counts against the grace period budget, so if your hook runs for 20 seconds and the grace period is 30, your app only has 10 seconds to drain after the hook."

---

**Q: A pod is stuck in Pending. How do you diagnose it?**

> "My first command is `kubectl describe pod` and I go straight to the Events section at the bottom. Kubernetes spells out exactly why the pod can't schedule in plain English — it'll say something like '0 of 3 nodes are available: 3 Insufficient memory' or '2 node(s) didn't match node selector' or '1 node(s) had taint that the pod didn't tolerate.' From there I know whether I need to adjust resource requests, add a label to a node, add a toleration, or wait for a node to become available. I'll also run `kubectl get nodes` to make sure nodes are Ready, and `kubectl describe node` on the candidate nodes to see their available allocatable resources versus what's already allocated. The scheduler's decision is always deterministic — there's always an exact reason, and describe almost always surfaces it directly."

---

**Q: What is the difference between `ErrImagePull` and `ImagePullBackOff`?**

> "They are sequential states for the same underlying problem — the kubelet cannot pull the container image. ErrImagePull is the immediate failure on the first attempt. ImagePullBackOff is what you see after the kubelet has tried multiple times and is now backing off exponentially between retry attempts. The root cause is identical — you diagnose both the same way with `kubectl describe pod` looking at the Events section, which will tell you specifically whether it's an authentication failure, a non-existent tag, a non-existent repository, or a network connectivity issue to the registry. Common fixes are correcting the image name or tag, adding an imagePullSecret for private registries, or verifying the cluster's network access to the registry."

---

**Q: What are the three imagePullPolicy values and when does each apply?**

> "The three values are Always, IfNotPresent, and Never. Always means the kubelet calls the registry on every container start, even if the image is cached locally — this is the default for images tagged with 'latest' or with no tag at all. IfNotPresent means the kubelet uses the local cache if available and only pulls if the image isn't there — this is the default for any explicit version tag like nginx:1.25. Never means the kubelet refuses to pull and will fail the container if the image isn't already cached on the node — useful for air-gapped environments where you've pre-loaded images. The critical exam point is that the default is not always IfNotPresent: the tag determines the default, and 'latest' defaults to Always."

---

## 8. Debugging

> *Fast diagnosis paths — commands + decision trees.*

### Entry Point: Read the STATUS Column

```bash
kubectl get pods
# STATUS is your first signal — never skip this step
```

### Status → Root Cause Decision Tree

```
STATUS = Pending
  │
  └── kubectl describe pod <name> → Events section
        ├── "Insufficient cpu/memory"
        │     → resource requests too high
        │     → kubectl describe node → check Allocated vs Allocatable
        │     → Fix: lower requests or add nodes
        │
        ├── "didn't match node selector"
        │     → node missing required label
        │     → kubectl get nodes --show-labels
        │     → Fix: kubectl label node <node> <key>=<value>
        │
        ├── "had taint ... that the pod didn't tolerate"
        │     → pod missing toleration for node taint
        │     → Fix: add tolerations to pod spec
        │
        └── "0/N nodes available" with no specific reason
              → kubectl get nodes → any NotReady?
              → Fix: investigate node health

STATUS = CrashLoopBackOff
  │
  └── kubectl logs <pod> --previous
        ├── Application error message visible?
        │     → Fix the application config/code
        │
        ├── "No such file or directory" or "command not found"
        │     → wrong image or wrong command spec
        │     → Fix: correct image name or command
        │
        ├── Connection refused / timeout messages?
        │     → dependency (DB, cache) not reachable
        │     → kubectl exec -it <pod> -- sh → test connectivity
        │     → Fix: ensure dependency is up, add init container to wait
        │
        └── No logs at all / truncated?
              → kubectl describe pod → lastState.terminated.reason
              → Reason: OOMKilled? → exitCode: 137
              → Fix: increase memory limit

STATUS = ErrImagePull / ImagePullBackOff
  │
  └── kubectl describe pod <name> → Events → Failed to pull image
        ├── "not found" / "manifest unknown"
        │     → wrong image name or tag
        │     → Fix: correct the image reference
        │
        ├── "unauthorized" / "authentication required"
        │     → private registry, missing imagePullSecret
        │     → kubectl get secret regcred
        │     → Fix: create docker-registry secret + add imagePullSecrets
        │
        └── "connection refused" / timeout
              → registry network unreachable from cluster
              → kubectl exec node-pod -- curl registry
              → Fix: network policy, firewall, proxy settings

STATUS = OOMKilled (appears in describe, not get pods)
  │
  └── kubectl describe pod → lastState.terminated.reason = OOMKilled
        ├── exitCode: 137 confirms kernel OOM kill
        ├── Check resources.limits.memory in pod spec
        ├── Check actual usage: kubectl top pod <pod>
        └── Fix: increase memory limit OR fix memory leak in application

STATUS = Terminating (stuck)
  │
  └── kubectl describe pod → check Finalizers
        ├── Finalizers present?
        │     → something is blocking deletion (storage, custom controller)
        │     → kubectl patch pod <pod> -p '{"metadata":{"finalizers":[]}}' --type=merge
        │
        └── No finalizers but still stuck?
              → node may be unreachable
              → kubectl get nodes → NotReady?
              → kubectl delete pod <pod> --force --grace-period=0
```

### Fast Diagnosis Commands (Ordered by Information Density)

```bash
# 1. Broad view — always start here
kubectl get pods -o wide

# 2. Events — answers 80% of debug questions
kubectl describe pod <pod> | grep -A 30 "Events:"

# 3. Last container crash details
kubectl describe pod <pod> | grep -A 10 "Last State:"

# 4. Application error logs
kubectl logs <pod> --previous --tail=100

# 5. Container-level restart counts and ready status
kubectl get pod <pod> -o jsonpath='{range .status.containerStatuses[*]}{.name}: ready={.ready} restarts={.restartCount} reason={.lastState.terminated.reason}{"\n"}{end}'

# 6. Node resource capacity vs allocation
kubectl describe node <node> | grep -A 10 "Allocated resources:"

# 7. Cluster-wide events, most recent first
kubectl get events --sort-by='.lastTimestamp' -A | tail -30

# 8. Shell into running container
kubectl exec -it <pod> -- sh

# 9. Ephemeral debug container (K8s 1.23+ — use when main container has no shell)
kubectl debug -it <pod> --image=busybox --target=<container>

# 10. Check pod conditions (PodScheduled, Initialized, ContainersReady, Ready)
kubectl get pod <pod> -o jsonpath='{range .status.conditions[*]}{.type}: {.status} ({.reason}){"\n"}{end}'
```

### Exit Code Reference

| Exit Code | Signal | Meaning | Common cause |
|-----------|--------|---------|-------------|
| 0 | — | Success | Normal exit |
| 1 | — | Generic error | Application error |
| 126 | — | Command not executable | Permission issue |
| 127 | — | Command not found | Wrong image or path |
| 128+N | Signal N | Killed by signal N | Kernel/kubelet |
| 137 | SIGKILL (9) | Forcefully killed | OOMKilled or force delete |
| 143 | SIGTERM (15) | Graceful termination | Normal kubectl delete |

---

## 9. Kill Switch

> *10-second recall — the absolute minimum to hold in memory.*

```
┌─────────────────────────────────────────────────────────────────┐
│            POD LIFECYCLE & DEBUGGING — 10 SECOND RECALL         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PHASES:   Pending → Running → Succeeded/Failed/Unknown         │
│  DISPLAY:  Succeeded shows as "Completed" in kubectl            │
│                                                                 │
│  restartPolicy:                                                 │
│    Always    → services (Deployment)                            │
│    OnFailure → jobs / batch                                     │
│    Never     → debug / one-shot                                 │
│                                                                 │
│  imagePullPolicy:                                               │
│    latest tag → Always (default)                                │
│    version tag → IfNotPresent (default)                         │
│                                                                 │
│  Termination:  SIGTERM → wait grace period → SIGKILL            │
│  Default grace period: 30s                                      │
│  preStop hook time counts against grace period                  │
│                                                                 │
│  CrashLoopBackOff backoff: 10→20→40→80→160→300s (resets @10min) │
│  OOMKilled = exitCode 137 = kernel killed it                    │
│                                                                 │
│  Debug order: get pods → describe → logs --previous → exec      │
│  Pending → describe Events (scheduler tells you exactly why)    │
│  CrashLoop → logs --previous (not logs)                         │
│  ImagePullBackOff → describe Events (exact registry error)      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 10. Appendix

> *Quick reference card — commands, formats, cheatsheets.*

### Pod Phase Quick Reference

| Phase | `kubectl get pods` | Meaning |
|-------|-------------------|---------|
| `Pending` | `Pending` | Not scheduled or image not pulled |
| `Running` | `Running` | ≥1 container running |
| `Succeeded` | `Completed` | All containers exited 0, not restarting |
| `Failed` | `Error` / `CrashLoopBackOff` | ≥1 container exited non-0 |
| `Unknown` | `Unknown` | Node lost, status unknown |

### Pod Conditions Quick Reference

```bash
kubectl get pod <pod> -o jsonpath='{range .status.conditions[*]}{.type}: {.status}{"\n"}{end}'
```

| Condition | True means |
|-----------|-----------|
| `PodScheduled` | Node assigned |
| `Initialized` | All init containers completed |
| `ContainersReady` | All containers passing readiness |
| `Ready` | Pod can serve traffic |

### restartPolicy × Pod Controller Compatibility

| Controller | Allowed restartPolicy |
|-----------|----------------------|
| Deployment | `Always` only |
| StatefulSet | `Always` only |
| DaemonSet | `Always` only |
| Job | `OnFailure` or `Never` |
| CronJob | `OnFailure` or `Never` |
| Standalone Pod | `Always`, `OnFailure`, `Never` |

### imagePullPolicy YAML Skeleton

```yaml
containers:
- name: app
  image: myregistry.io/myapp:v1.2.3    # pinned tag
  imagePullPolicy: IfNotPresent         # use cache if available
```

```yaml
imagePullSecrets:                        # for private registries
- name: regcred
```

### Termination Configuration YAML Skeleton

```yaml
spec:
  terminationGracePeriodSeconds: 60      # total budget for preStop + drain
  containers:
  - name: app
    image: nginx:1.25
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "nginx -s quit; sleep 5"]
```

### Resource QoS Classes (Affects OOMKill Priority)

| QoS Class | Condition | OOMKill priority |
|-----------|-----------|-----------------|
| `Guaranteed` | requests == limits for all resources | Last to be killed |
| `Burstable` | requests < limits, or only some set | Middle priority |
| `BestEffort` | no requests or limits set | First to be killed |

```bash
# Check QoS class of a pod
kubectl get pod <pod> -o jsonpath='{.status.qosClass}'
```

### Complete Debug Command Cheatsheet

```bash
# --- Status ---
kubectl get pods
kubectl get pods -o wide
kubectl get pods -o wide --all-namespaces
kubectl get pods --field-selector=status.phase=Running
kubectl get events --sort-by='.lastTimestamp'

# --- Describe ---
kubectl describe pod <pod>
kubectl describe node <node>

# --- Logs ---
kubectl logs <pod>
kubectl logs <pod> --previous
kubectl logs <pod> -f
kubectl logs <pod> --tail=100
kubectl logs <pod> -c <container>
kubectl logs <pod> -c <container> --previous

# --- Exec ---
kubectl exec -it <pod> -- sh
kubectl exec -it <pod> -c <container> -- sh
kubectl exec <pod> -- env                         # print env vars
kubectl exec <pod> -- cat /etc/resolv.conf        # check DNS config

# --- Ephemeral debug (K8s 1.23+) ---
kubectl debug -it <pod> --image=busybox --target=<container>
kubectl debug node/<node> -it --image=ubuntu      # debug the node

# --- State queries ---
kubectl get pod <pod> -o jsonpath='{.status.phase}'
kubectl get pod <pod> -o jsonpath='{.status.qosClass}'
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[*].restartCount}'
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[*].lastState.terminated.reason}'

# --- Nodes ---
kubectl get nodes
kubectl get nodes --show-labels
kubectl describe node <node> | grep -A 10 "Allocated resources"
kubectl label node <node> <key>=<value>
kubectl label node <node> <key>-                  # remove label

# --- Cleanup ---
kubectl delete pod <pod>
kubectl delete pod <pod> --grace-period=0 --force
kubectl delete pod <pod> --wait=false
```

### Common Status → Fix Reference Card

| Status | First Command | Likely Fix |
|--------|--------------|------------|
| `Pending` | `kubectl describe pod` → Events | Add node label / lower resource requests / add toleration |
| `CrashLoopBackOff` | `kubectl logs --previous` | Fix app config / increase memory / fix command |
| `ImagePullBackOff` | `kubectl describe pod` → Events | Fix image name / add imagePullSecret |
| `OOMKilled` | `kubectl describe pod` → lastState | Increase `resources.limits.memory` |
| `Terminating` (stuck) | `kubectl describe pod` → Finalizers | Patch finalizers or `--force --grace-period=0` |
| `Unknown` | `kubectl get nodes` | Investigate node health |
| `Error` (restartPolicy:Never) | `kubectl logs --previous` | One-time failure — check exit code |
