# Kubernetes Multi-Container Pods 

> **Scope:** Init Containers · Sidecar · Ambassador · Adapter · Shared Network · Shared Volumes · Failure Behavior

---

## 0. First Principles

> *The mental model that never changes, regardless of Kubernetes version.*

**One Pod = One Execution Context.**
A Pod is not a container — it is a shared execution environment (network namespace + IPC namespace + optional volume mounts) that one or more containers inhabit simultaneously.

**The immutable truths:**

| Principle | What it means |
|-----------|---------------|
| **Shared fate** | Containers in a Pod are scheduled together on the same Node — always |
| **Shared network** | Every container in a Pod shares the same IP address, loopback, and port space |
| **Shared IPC** | Containers can communicate via System V semaphores and POSIX shared memory |
| **Independent lifecycle** | A container crashing does not automatically terminate other containers |
| **Init is sequential, always** | Init containers run one at a time, in order, to completion before any app container starts |
| **Volumes are the glue** | The only way containers share filesystem state is via explicitly mounted volumes |

**The core mental split:**

```
initContainers  →  "before the world begins"   — setup, gating, dependency checks
containers      →  "while the world runs"       — application, helpers, proxies, translators
```

Everything in multi-container pod design is a consequence of these six truths.

---

## 1. Reality Constraints

> *What Kubernetes actually enforces — not what the docs imply.*

### What Kubernetes DOES enforce

- **Init containers run strictly sequentially.** There is no parallelism option.
- **A failed init container blocks the pod indefinitely** (subject to `restartPolicy`). The main containers never start.
- **Init containers are restarted on failure** (with backoff) if `restartPolicy` is `Always` or `OnFailure`. With `restartPolicy: Never`, one failure terminates the pod permanently.
- **Resource accounting is separate.** Kubernetes tracks init container resource requests independently from app containers. The effective pod resource request is `max(sum(app containers), max(init containers))`.
- **Init containers do NOT support `livenessProbe`, `readinessProbe`, or `startupProbe`.** These fields are silently ignored if set. Only `lifecycle` hooks are permitted.
- **Sidecar containers are independent processes.** The kubelet does not understand "sidecar" as a first-class concept pre-1.28. From Kubernetes 1.28+, native sidecar support (`initContainers` with `restartPolicy: Always`) exists as a GA feature in 1.29.
- **Port conflicts within a pod are illegal.** Two containers cannot bind the same port on the shared network namespace.
- **Volume mounts are per-container.** Declaring a volume at the pod level does not automatically mount it — each container must declare its own `volumeMounts`.

### What Kubernetes does NOT do

| Misconception | Reality |
|---------------|---------|
| "Sidecar stops when main container dies" | Sidecar keeps running; it is independent |
| "Init containers can be parallelized with some flag" | No such option exists |
| "Containers share filesystem by default" | No — only via explicitly defined volumes |
| "A crashed sidecar kills the pod" | Only if the pod's `restartPolicy` causes the pod to enter `CrashLoopBackOff` and exhaust retries |
| "Init containers have their own IP" | No — init containers share the pod IP |
| "You can liveness-probe an init container" | Not supported (pre-1.28 native sidecars) |

### Resource Calculation (Often Missed)

```
Effective pod CPU request = max(
    sum of all app container CPU requests,
    max individual init container CPU request
)
```

This matters for scheduling. An init container with a very high resource request can block scheduling even if app containers are tiny.

---

## 2. Decision Logic

> *When to use which pattern — unambiguous rules.*

### Pattern Selection Flowchart

```
Does the workload need something completed BEFORE the app starts?
│
├── YES → Does it need network access or external dependencies?
│           ├── YES → Init Container (with curl/nslookup/wait loops)
│           └── NO  → Init Container (file prep, config generation, schema migration)
│
└── NO → Does the helper need to run ALONGSIDE the app?
            │
            ├── Does it translate/adapt data format for the main app?
            │     └── YES → Adapter Pattern
            │
            ├── Does it proxy requests TO external services on behalf of main app?
            │     └── YES → Ambassador Pattern
            │
            └── Does it assist (logging, monitoring, health checking)?
                      └── YES → Sidecar Pattern
```

### Pattern Comparison Table

| Property | Init Container | Sidecar | Ambassador | Adapter |
|----------|---------------|---------|------------|---------|
| **Runs at** | Before app starts | Same time as app | Same time as app | Same time as app |
| **Lifecycle** | Exits on success | Runs forever | Runs forever | Runs forever |
| **Typical image** | curl, busybox, migrate | fluentd, envoy, datadog | envoy, nginx | custom transformer |
| **Shared volume** | Yes (pass data to app) | Yes (read app logs) | Rarely | Sometimes |
| **Network access** | Yes | localhost only (via pod IP) | Bridges external | localhost only |
| **CKA frequency** | ★★★★★ | ★★★★☆ | ★★☆☆☆ | ★★☆☆☆ |

### When to Use Each Pattern

```
Init Container    → DB schema migration, secret fetch, external service readiness gate
Sidecar           → Log shipping (Fluentd), metrics exposition, certificate rotation
Ambassador        → Abstracting cloud-specific APIs, service mesh ingress per-pod
Adapter           → Prometheus format conversion, log normalization for legacy systems
```

---

## 3. Internal Working

> *How Kubernetes actually orchestrates this, step by step.*

### Pod Startup Sequence (Full)

```
1. API Server validates Pod spec → stores in etcd
2. Scheduler assigns Pod to Node → writes nodeName to spec
3. kubelet on Node detects unscheduled Pod via watch
4. kubelet calls container runtime (containerd/CRI-O):
   a. Create pause container (the "infra" container)
      - Holds network namespace
      - Holds IPC namespace
      - Never runs user code
      - Has the Pod IP
   b. All subsequent containers JOIN this namespace
5. kubelet starts initContainers[0]
   - Waits for exit code 0
   - On failure: restarts with exponential backoff (if restartPolicy allows)
6. kubelet starts initContainers[1] (only after [0] exits 0)
   ... continues sequentially ...
7. ALL init containers succeeded → kubelet starts all containers[] simultaneously
8. kubelet begins health checking (liveness, readiness, startup probes)
9. Pod enters Running state when at least one container is running
10. Pod enters Ready when all readiness probes pass
```

### What the Pause Container Actually Does

```
┌─────────────────────────────────────┐
│               POD                   │
│                                     │
│  ┌──────────────────┐               │
│  │  pause container │  ← holds     │
│  │  (infra/sandbox) │    netns      │
│  │  PID 1 = pause   │    ipcns      │
│  │  IP: 10.0.0.5    │               │
│  └──────────────────┘               │
│          ↑ all containers join ↑    │
│  ┌────────────┐  ┌───────────────┐  │
│  │ main-app   │  │   sidecar     │  │
│  │ nginx:80   │  │ health-logger │  │
│  └────────────┘  └───────────────┘  │
│                                     │
│  Shared: IP, loopback, /dev/shm    │
│  NOT shared: PID ns (by default)   │
└─────────────────────────────────────┘
```

### Init Container Restart Behavior by `restartPolicy`

| restartPolicy | Init container fails | Behavior |
|---------------|---------------------|----------|
| `Always` | Retried with backoff | Pod stays alive, init retried |
| `OnFailure` | Retried with backoff | Same as Always for init |
| `Never` | Pod fails permanently | No retry; pod enters Failed state |

### Volume Lifecycle

```
emptyDir volume:
  Created when Pod is assigned to Node
  Lives as long as Pod lives on that Node
  Survives container restarts within the Pod
  Deleted when Pod is deleted or rescheduled

configMap / secret volume:
  Projected from etcd
  Updated in-place (with some propagation delay ~60s)
  All containers mounting it see updates simultaneously
```

---

## 4. Hands-On

> *Production-quality YAML + commands. Nothing simplified.*

### Lab 1 — Init Container as Dependency Gate

```yaml
# init-gate.yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-gate
  labels:
    app: init-gate
spec:
  restartPolicy: OnFailure
  initContainers:
  - name: wait-for-db
    image: busybox:1.36
    command:
    - sh
    - -c
    - |
      echo "Waiting for database at db-svc:5432..."
      until nc -z db-svc 5432; do
        echo "DB not ready, sleeping 5s..."
        sleep 5
      done
      echo "Database is reachable"
  containers:
  - name: main-app
    image: nginx:1.25
    ports:
    - containerPort: 80
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "250m"
        memory: "256Mi"
```

### Lab 2 — Multiple Init Containers (Sequential Proof)

```yaml
# init-sequential.yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-sequential
spec:
  initContainers:

  - name: step-1-fetch-config
    image: busybox:1.36
    command:
    - sh
    - -c
    - |
      echo "[step-1] Fetching config..."
      sleep 10
      echo "db_host=postgres-svc" > /shared/config.env
      echo "[step-1] Config written"
    volumeMounts:
    - name: shared-data
      mountPath: /shared

  - name: step-2-validate-config
    image: busybox:1.36
    command:
    - sh
    - -c
    - |
      echo "[step-2] Validating config..."
      cat /shared/config.env
      grep -q "db_host" /shared/config.env && echo "[step-2] Config valid" || exit 1
    volumeMounts:
    - name: shared-data
      mountPath: /shared

  containers:
  - name: main-app
    image: busybox:1.36
    command:
    - sh
    - -c
    - |
      echo "Main app starting with config:"
      cat /shared/config.env
      sleep 3600
    volumeMounts:
    - name: shared-data
      mountPath: /shared

  volumes:
  - name: shared-data
    emptyDir: {}
```

### Lab 3 — Sidecar with Shared Volume (Log Shipping Pattern)

```yaml
# sidecar-logging.yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-logging
spec:
  volumes:
  - name: app-logs
    emptyDir: {}

  containers:

  - name: main-app
    image: busybox:1.36
    command:
    - sh
    - -c
    - |
      while true; do
        echo "[$(date -Iseconds)] request processed status=200" >> /var/log/app/access.log
        sleep 3
      done
    volumeMounts:
    - name: app-logs
      mountPath: /var/log/app
    resources:
      requests:
        cpu: "50m"
        memory: "64Mi"

  - name: log-shipper
    image: busybox:1.36
    command:
    - sh
    - -c
    - |
      tail -F /var/log/app/access.log
    volumeMounts:
    - name: app-logs
      mountPath: /var/log/app
    resources:
      requests:
        cpu: "25m"
        memory: "32Mi"
```

### Lab 4 — Sidecar Health Logger (Shared Network Namespace)

```yaml
# sidecar-health.yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-health
spec:
  containers:

  - name: main-app
    image: nginx:1.25
    ports:
    - containerPort: 80
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"

  - name: health-logger
    image: curlimages/curl:8.4.0
    command:
    - sh
    - -c
    - |
      while true; do
        STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:80)
        if [ "$STATUS" = "200" ]; then
          echo "[$(date -Iseconds)] main-app healthy (HTTP $STATUS)"
        else
          echo "[$(date -Iseconds)] main-app UNHEALTHY (HTTP $STATUS)"
        fi
        sleep 10
      done
    resources:
      requests:
        cpu: "25m"
        memory: "32Mi"
```

### Native Sidecar (Kubernetes 1.29+)

```yaml
# native-sidecar-k8s129.yaml
# initContainer with restartPolicy: Always = native sidecar
# Starts before app containers, restarts on failure, runs for pod lifetime
apiVersion: v1
kind: Pod
metadata:
  name: native-sidecar-demo
spec:
  initContainers:
  - name: log-shipper-sidecar
    image: busybox:1.36
    restartPolicy: Always          # <-- this makes it a native sidecar (K8s 1.29+)
    command:
    - sh
    - -c
    - tail -F /var/log/app/access.log
    volumeMounts:
    - name: app-logs
      mountPath: /var/log/app

  containers:
  - name: main-app
    image: nginx:1.25
    volumeMounts:
    - name: app-logs
      mountPath: /var/log/app

  volumes:
  - name: app-logs
    emptyDir: {}
```

### Essential Commands

```bash
# Apply all manifests
kubectl apply -f init-gate.yaml
kubectl apply -f sidecar-logging.yaml

# Watch pod startup in real time
kubectl get pods -w

# Detailed pod state — shows init container status, events, conditions
kubectl describe pod <pod-name>

# List all containers in a pod (init + app)
kubectl get pod <pod-name> -o jsonpath='{.spec.initContainers[*].name}'
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[*].name}'

# Logs from a specific container
kubectl logs <pod-name> -c <container-name>
kubectl logs <pod-name> -c <container-name> --previous   # last terminated instance
kubectl logs <pod-name> -c <container-name> -f           # follow

# Exec into a specific container
kubectl exec -it <pod-name> -c <container-name> -- sh

# Check init container exit code
kubectl get pod <pod-name> -o jsonpath='{.status.initContainerStatuses[*].state}'

# Check all container statuses
kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[*]}'

# Force delete a stuck pod
kubectl delete pod <pod-name> --grace-period=0 --force
```

---

## 5. Production Flow

> *Real-world architecture and design patterns with rationale.*

### Pattern 1 — Database Migration Init (Most Common Production Use)

```
┌──────────────────────────────────────────────────────────┐
│                    Deployment Pod                        │
│                                                          │
│  initContainers:                                         │
│  ┌─────────────────────────────┐                         │
│  │ wait-for-db                 │  nc -z postgres:5432    │
│  │ busybox                     │  (loop until success)   │
│  └─────────────┬───────────────┘                         │
│                │ exits 0                                 │
│  ┌─────────────▼───────────────┐                         │
│  │ run-migrations              │  alembic upgrade head   │
│  │ app-image:latest            │  (or flyway, liquibase) │
│  └─────────────┬───────────────┘                         │
│                │ exits 0                                 │
│  containers:                                             │
│  ┌─────────────▼───────────────┐                         │
│  │ api-server                  │  uvicorn app:main       │
│  │ app-image:latest            │                         │
│  └─────────────────────────────┘                         │
└──────────────────────────────────────────────────────────┘

Key: migrations run exactly once per pod start
     no migration = no app start (guaranteed)
```

### Pattern 2 — Fluentd Log Shipping Sidecar (Production Standard)

```
┌─────────────────────────────────────────────────────────┐
│                    Application Pod                      │
│                                                         │
│  volumes:                                               │
│  └── app-logs: emptyDir                                 │
│                                                         │
│  containers:                                            │
│  ┌──────────────────┐    ┌───────────────────────────┐  │
│  │   main-app       │    │      fluentd sidecar      │  │
│  │                  │    │                           │  │
│  │  writes to       │    │  tail /var/log/app/*.log  │  │
│  │  /var/log/app/   │───▶│  → parse → enrich         │  │
│  │  access.log      │    │  → forward to Elasticsearch│  │
│  └──────────────────┘    └───────────────────────────┘  │
│        shared emptyDir volume                           │
└─────────────────────────────────────────────────────────┘
```

### Pattern 3 — Envoy Ambassador Proxy

```
┌─────────────────────────────────────────────────────────┐
│                    Application Pod                      │
│                                                         │
│  containers:                                            │
│  ┌──────────────────┐    ┌───────────────────────────┐  │
│  │   main-app       │    │   envoy (ambassador)      │  │
│  │                  │    │                           │  │
│  │  calls           │    │  listens on localhost:9000│  │
│  │  localhost:9000  │───▶│  → TLS termination        │  │
│  │                  │    │  → retries                │  │
│  │  (no TLS code)   │    │  → circuit breaking       │  │
│  └──────────────────┘    │  → forwards to external   │  │
│                          └───────────────────────────┘  │
│  Main app stays cloud-agnostic                          │
└─────────────────────────────────────────────────────────┘
```

### Pattern 4 — Prometheus Adapter

```
┌─────────────────────────────────────────────────────────┐
│                    Application Pod                      │
│                                                         │
│  containers:                                            │
│  ┌──────────────────┐    ┌───────────────────────────┐  │
│  │   legacy-app     │    │   metrics-adapter         │  │
│  │                  │    │                           │  │
│  │  exposes metrics │    │  scrapes /metrics/statsd  │  │
│  │  in StatsD fmt   │───▶│  → converts to Prometheus │  │
│  │  on :8125        │    │  → exposes on :9102       │  │
│  └──────────────────┘    └───────────────────────────┘  │
│                                           ↑             │
│                              Prometheus scrapes :9102   │
└─────────────────────────────────────────────────────────┘
```

### Real Production Decision Matrix

| Situation | Pattern | Reason |
|-----------|---------|--------|
| App needs DB schema current before starting | Init | Guarantees ordering |
| App emits logs, need centralized shipping | Sidecar + volume | Decouples log infra from app code |
| App needs mTLS to external service, no code change | Ambassador | Proxy handles auth transparently |
| Legacy app emits StatsD, monitoring stack needs Prometheus | Adapter | Format translation at pod boundary |
| App needs secrets from Vault on startup | Init | Fetch once, write to emptyDir, app reads |
| Cert rotation while app runs | Sidecar | Watches cert expiry, rotates, signals app |

---

## 6. Mistakes

> *What actually breaks in real systems — root cause + fix.*

### Mistake 1 — Init Container in `containers:` block

**What happens:** YAML is valid. Kubernetes schedules it. Init container runs concurrently with main app — no sequencing, no gating.

```yaml
# WRONG — init logic placed in containers block
containers:
- name: check-db       # ← This is NOT an init container
  image: busybox
  command: ["sh", "-c", "nc -z db:5432"]
- name: main-app
  image: nginx
```

```yaml
# CORRECT
initContainers:
- name: check-db
  image: busybox
  command: ["sh", "-c", "until nc -z db 5432; do sleep 2; done"]
containers:
- name: main-app
  image: nginx
```

**Diagnosis:** `kubectl describe pod` → check `Init Containers:` section exists and is separate from `Containers:`.

---

### Mistake 2 — Port Conflict Between Containers

**Root cause:** Two containers in the same pod try to bind the same port. The second container crashes with `EADDRINUSE`.

**Symptom:** One container in `CrashLoopBackOff`, other is `Running`.

```bash
kubectl logs <pod> -c <crashing-container>
# Error: bind: address already in use
```

**Fix:** Each container must bind a unique port within the pod's shared network namespace.

---

### Mistake 3 — Init Container with `livenessProbe`

**Root cause:** Developer adds a liveness probe to an init container expecting health checking during startup.

**What happens:** Kubernetes silently ignores the probe (pre-1.28). The init container runs but is never health-checked. If it hangs, pod hangs forever.

**Fix:** Do not add probes to init containers. Use a timeout in the command itself:

```bash
timeout 30 sh -c "until nc -z db 5432; do sleep 2; done"
```

---

### Mistake 4 — Volume Not Mounted in Init Container

**Root cause:** Volume declared at pod level but `volumeMounts` forgotten in the init container spec.

**Symptom:** Init container writes to its own filesystem. Main container sees an empty volume.

```bash
kubectl exec <pod> -c main-app -- ls /shared
# (empty)
```

**Fix:** Every container (init or app) that needs a volume must declare its own `volumeMounts` entry.

---

### Mistake 5 — `restartPolicy: Never` on a Pod with Init Containers

**Root cause:** Batch job pod set to `Never`. Init container fails transiently (network blip). Pod enters `Failed` state permanently — no retry.

**Fix:** For jobs with init containers that may fail transiently, use `restartPolicy: OnFailure` or wrap retry logic inside the init container command.

---

### Mistake 6 — Assuming Sidecar Terminates When Main App Dies

**Root cause:** Developer assumes pod death = all containers die simultaneously.

**Reality:** When main app container exits, sidecar continues running until the kubelet acts on the pod's `restartPolicy`. The sidecar may keep running for several seconds, logging errors.

**Implication for log shippers:** Sidecar may flush remaining logs after main app crashes — this is actually desirable behavior.

---

### Mistake 7 — High-Resource Init Container Blocking Scheduling

**Root cause:** Init container requests `4Gi` memory for a migration job. No node has 4Gi free. Pod stays `Pending`.

```bash
kubectl describe pod <pod>
# Events: 0/3 nodes are available: 3 Insufficient memory
```

**Fix:** Profile init container actual memory usage. Set requests to actual P95 usage. Init containers should be lean — they run once.

---

## 7. Interview Answers

> *Compressed, verbatim-ready answers for common interview questions.*

---

**Q: What is an init container and how does it differ from a regular container?**

> "An init container is a specialized container that runs to completion before any of the main application containers start. The key differences are threefold: first, init containers are strictly sequential — if you have three init containers, they run one at a time in declared order, not in parallel. Second, each init container must exit with code zero for the next one to proceed, acting as a gate that blocks the main containers. Third, init containers do not support liveness or readiness probes. They're designed for one-time setup tasks like database migrations, config fetching, or waiting for external dependencies to become available."

---

**Q: How do containers in the same pod communicate with each other?**

> "Containers within the same pod share a network namespace, which means they share the same IP address and loopback interface. So a sidecar can reach the main application simply by calling localhost on the port the main container is listening on — no service, no DNS, no cluster networking involved. For filesystem sharing, containers need to mount a common volume — typically an emptyDir — because the filesystem namespace is not shared by default. IPC namespace is shared, so they can also communicate via shared memory or POSIX semaphores, though that's rare in practice."

---

**Q: What happens to a sidecar container when the main application container crashes?**

> "The sidecar keeps running — it is completely independent. A container crash in one container does not propagate to other containers in the pod. What happens next depends on the pod's restartPolicy. If it's set to Always, the kubelet will restart the crashed main container, and the sidecar continues running throughout. This is actually useful for log shippers — the sidecar can flush the final logs from the dying main container before the pod is restarted. The sidecar only stops when the pod itself is terminated."

---

**Q: What is the difference between the Sidecar, Ambassador, and Adapter patterns?**

> "All three involve co-located helper containers, but they serve different purposes. A sidecar extends or assists the main container — classic examples are Fluentd for log shipping or Datadog for metrics. An ambassador acts as an outbound proxy on behalf of the main container — for example, Envoy handling TLS, retries, and circuit breaking so the main app can call localhost and stay cloud-agnostic. An adapter exposes the main container's functionality to the outside world in a standardized format — for example, a container that translates a legacy app's StatsD metrics into Prometheus format for scraping. The boundary between them is somewhat architectural — they all technically run the same way in Kubernetes."

---

**Q: How does Kubernetes calculate resource requests for a pod with init containers?**

> "The effective resource request for a pod is the maximum of two quantities: the sum of all app container resource requests, and the maximum individual init container resource request. So if your three app containers each request 100m CPU totaling 300m, but your init container requests 500m CPU, the pod is scheduled as if it needs 500m CPU. This trips people up because they assume init containers don't affect scheduling — they absolutely do, especially memory, which is where migration jobs are often undersized or oversized."

---

**Q: What is a native sidecar in Kubernetes 1.29+?**

> "Before 1.29, Kubernetes had no first-class concept of a sidecar — it was just a convention of putting helper containers in the containers block. The problem is that regular sidecars have no guaranteed startup order relative to app containers, and their shutdown is not coordinated. Native sidecars, introduced as GA in 1.29, are defined as init containers with restartPolicy: Always. This gives them three guarantees regular sidecars lack: they start before app containers, they restart on failure like app containers, and they receive a SIGTERM signal to shut down gracefully after app containers terminate. This matters for service mesh proxies like Envoy, which would otherwise die before the app finishes draining connections."

---

**Q: A pod is stuck in Init:0/2 — how do you debug it?**

> "I'd start by running kubectl describe pod to check the Events section — this usually tells me immediately if the init container is failing, waiting for a resource, or hitting an image pull issue. Then I'd check the logs of the stuck init container with kubectl logs pod-name -c init-container-name. Common causes are: the init container is in an infinite wait loop because a dependency like a database or service isn't reachable, the image can't be pulled, or the init container command exits non-zero due to a bug. If the init container is still running, I'd exec into it with kubectl exec -it pod -c init-container-name -- sh and manually test the dependency check. The key question is always: what is the init container waiting for, and does that thing actually exist?"

---

## 8. Debugging

> *Fast diagnosis paths — commands + decision trees.*

### Diagnostic Entry Point

```bash
kubectl get pod <pod-name>
# Read STATUS column — this is your first signal
```

### Status → Root Cause Decision Tree

```
STATUS = Init:N/M
  │
  ├── Is N < M? (some inits done, one is stuck)
  │     └── kubectl logs <pod> -c <init-container-N>
  │           ├── Loop output? → dependency not available
  │           │     └── kubectl exec -it <pod> -c <init-N> -- nc -z <host> <port>
  │           ├── No output?  → container not started (image pull issue)
  │           │     └── kubectl describe pod → Events → ImagePullBackOff?
  │           └── Error/exit? → command failure
  │                 └── kubectl logs <pod> -c <init-N> --previous
  │
STATUS = Init:Error
  │     └── kubectl logs <pod> -c <failed-init> --previous
  │           → fix command or dependency, kubectl delete pod + reapply
  │
STATUS = CrashLoopBackOff (app container)
  │     └── kubectl logs <pod> -c <main-container> --previous
  │           ├── Exit 1? → application error
  │           ├── OOMKilled? → increase memory limit
  │           └── Port conflict? → check other containers' ports
  │
STATUS = Pending
        └── kubectl describe pod → Events
              ├── Insufficient memory/cpu → init container over-requested resources
              ├── No nodes match → node selector / taint issue
              └── PVC not bound → volume dependency unmet
```

### Fast Diagnosis Commands

```bash
# 1. Get all container names (init + app)
kubectl get pod <pod> -o jsonpath='{range .spec.initContainers[*]}{.name}{"\n"}{end}'
kubectl get pod <pod> -o jsonpath='{range .spec.containers[*]}{.name}{"\n"}{end}'

# 2. Check init container exit codes
kubectl get pod <pod> -o jsonpath='{range .status.initContainerStatuses[*]}{.name}: exitCode={.lastState.terminated.exitCode} reason={.lastState.terminated.reason}{"\n"}{end}'

# 3. Check all container restart counts
kubectl get pod <pod> -o jsonpath='{range .status.containerStatuses[*]}{.name}: restarts={.restartCount}{"\n"}{end}'

# 4. Events sorted by time (most useful for stuck pods)
kubectl describe pod <pod> | grep -A 20 "Events:"

# 5. Tail logs from specific container
kubectl logs <pod> -c <container> --tail=50 -f

# 6. Check previous (terminated) container logs
kubectl logs <pod> -c <container> --previous

# 7. Exec into running init container (while it's still running)
kubectl exec -it <pod> -c <init-container-name> -- sh

# 8. Check resource requests vs node capacity
kubectl describe node <node> | grep -A 10 "Allocated resources"
```

### Common Scenarios and One-Line Fixes

| Symptom | Command to confirm | Fix |
|---------|-------------------|-----|
| Init stuck in loop | `kubectl logs pod -c init-name` | Fix dependency or add correct service |
| Init exits non-zero | `kubectl logs pod -c init-name --previous` | Fix init container command |
| Pod Pending | `kubectl describe pod \| grep Events` | Fix resource request or node issue |
| Sidecar not seeing logs | `kubectl exec pod -c sidecar -- ls /shared/` | Add volumeMounts to sidecar spec |
| Port already in use | `kubectl logs pod -c crashing-container` | Change one container's bind port |
| Image pull failure | `kubectl describe pod \| grep -i imagepull` | Fix image name or add imagePullSecret |

---

## 9. Kill Switch

> *10-second recall — the absolute minimum to hold in memory.*

```
┌─────────────────────────────────────────────────────────────┐
│              MULTI-CONTAINER PODS — 10 SECOND RECALL        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  initContainers → sequential → must exit 0 → before app    │
│  containers     → simultaneous → run forever → share netns  │
│                                                             │
│  Init  = before    Sidecar = helper                         │
│  Ambs  = proxy     Adapter = translator                     │
│                                                             │
│  Shared:   IP address, loopback, IPC                        │
│  NOT:      filesystem (needs emptyDir volume)               │
│                                                             │
│  Init resource request DOES affect pod scheduling           │
│  Init containers do NOT support liveness/readiness probes   │
│  Sidecar death ≠ pod death (independent)                    │
│                                                             │
│  K8s 1.29+: initContainer + restartPolicy: Always = native  │
│                                                             │
│  Debug: describe → logs -c → exec -it -c → logs --previous  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 10. Appendix

> *Quick reference card — commands, formats, cheatsheets.*

### YAML Structure Quick Reference

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-name
spec:
  restartPolicy: Always         # Always | OnFailure | Never

  volumes:                      # Declare volumes at pod level
  - name: shared-data
    emptyDir: {}

  initContainers:               # Run sequentially, before containers
  - name: init-one
    image: busybox:1.36
    command: ["sh", "-c", "echo done"]
    volumeMounts:               # Each container mounts explicitly
    - name: shared-data
      mountPath: /data
    # NO livenessProbe, readinessProbe here

  - name: init-two              # Runs AFTER init-one succeeds
    image: busybox:1.36
    command: ["sh", "-c", "echo done2"]

  containers:                   # Run simultaneously, after all inits
  - name: main-app
    image: nginx:1.25
    ports:
    - containerPort: 80
    volumeMounts:
    - name: shared-data
      mountPath: /data
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "256Mi"
    livenessProbe:
      httpGet:
        path: /healthz
        port: 80
      initialDelaySeconds: 10
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /ready
        port: 80

  - name: sidecar
    image: busybox:1.36
    command: ["sh", "-c", "tail -f /data/app.log"]
    volumeMounts:
    - name: shared-data
      mountPath: /data
```

### Command Cheatsheet

```bash
# --- Observation ---
kubectl get pods -w                                    # watch pod status changes
kubectl describe pod <pod>                             # full state, events, conditions
kubectl get pod <pod> -o yaml                          # raw spec + status

# --- Logs ---
kubectl logs <pod> -c <container>                      # specific container
kubectl logs <pod> -c <container> -f                   # follow
kubectl logs <pod> -c <container> --previous           # last terminated
kubectl logs <pod> -c <container> --tail=100           # last N lines

# --- Exec ---
kubectl exec -it <pod> -c <container> -- sh            # shell into container
kubectl exec <pod> -c <container> -- cat /etc/config   # one-shot command

# --- Status Queries ---
kubectl get pod <pod> -o jsonpath='{.status.phase}'
kubectl get pod <pod> -o jsonpath='{.status.initContainerStatuses[*].state}'
kubectl get pod <pod> -o jsonpath='{.spec.initContainers[*].name}'
kubectl get pod <pod> -o jsonpath='{.spec.containers[*].name}'

# --- Cleanup ---
kubectl delete pod <pod>
kubectl delete pod <pod> --grace-period=0 --force      # stuck pods
```

### Pod Status Reference

| Status | Meaning |
|--------|---------|
| `Pending` | Waiting to be scheduled or for volumes/images |
| `Init:0/2` | 0 of 2 init containers complete |
| `Init:1/2` | 1 of 2 init containers complete |
| `Init:Error` | An init container exited non-zero |
| `Init:CrashLoopBackOff` | An init container is crash-looping |
| `PodInitializing` | All inits done, app containers starting |
| `Running` | At least one container is running |
| `Completed` | All containers exited 0 (Jobs) |
| `CrashLoopBackOff` | App container keeps crashing |
| `OOMKilled` | Container exceeded memory limit |
| `Evicted` | Node resource pressure caused eviction |

### Pattern Summary Card

| Pattern | Keyword | Real Example | Key Trait |
|---------|---------|-------------|-----------|
| Init Container | `initContainers:` | DB migration, secret fetch | Sequential, exits before app |
| Sidecar | `containers:` (helper) | Fluentd, Datadog agent | Co-runs, shares network/volume |
| Ambassador | `containers:` (proxy) | Envoy, nginx | Outbound proxy to external |
| Adapter | `containers:` (translator) | StatsD→Prometheus | Format translation inbound |
| Native Sidecar | `initContainers:` + `restartPolicy: Always` | Envoy in Istio (1.29+) | Ordered start, graceful shutdown |

### emptyDir vs Other Volume Types for Multi-Container

| Volume Type | Survives container restart? | Survives pod restart? | Use case |
|-------------|----------------------------|----------------------|----------|
| `emptyDir` | Yes | No | Temp sharing between containers |
| `emptyDir: medium: Memory` | Yes | No | High-speed sharing (tmpfs) |
| `configMap` | Yes | Yes | Read-only config injection |
| `secret` | Yes | Yes | Read-only secret injection |
| `persistentVolumeClaim` | Yes | Yes | Durable data across pods |
