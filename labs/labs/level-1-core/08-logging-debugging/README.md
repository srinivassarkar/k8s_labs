# Lab 08 — Logging & Debugging Workflow

## What This Lab Is About

Every lab so far has used logging and debugging commands. This lab makes them explicit. It is about building a **systematic, repeatable debugging workflow** that you run the same way every single time — regardless of the symptom. Not random commands. Not guessing. A disciplined sequence that eliminates layers one by one until the root cause is exposed.

In prod, the cost of debugging inefficiency is measured in downtime. An engineer who runs the right 5 commands in the right order solves the incident in 5 minutes. An engineer who runs 30 random commands solves the same incident in 45 minutes. This lab is about becoming the first engineer.

There are three scenarios. A multi-container pod where logs from the wrong container mislead you. A pod that has already crashed and the evidence is gone — unless you know where to look. And a live production-style incident where you get only a symptom and must find the root cause using nothing but `kubectl`.

> The best debuggers are not the ones who know the most commands. They are the ones who run the fewest commands in the right order.

---

## Environment

- **Cluster:** KIND (Kubernetes IN Docker)
- **K8s Version:** v1.27.3
- **Node:** Single node (`kind-control-plane`)
- **Namespace for this lab:** `lab08`

```bash
kubectl create namespace lab08
kubectl config set-context --current --namespace=lab08
```

---

## The 3 Scenarios

| # | Scenario | What You'll Learn |
|---|---|---|
| 01 | Multi-container pod — logs from wrong container | `-c` flag, init containers, sidecar logging |
| 02 | Dead pod — evidence after the crash | `--previous`, events TTL, ephemeral containers |
| 03 | Live incident — symptom to root cause | The full systematic workflow under pressure |

---

## The Master Debug Workflow — Learn This First

Before the scenarios — internalize this order. It is the backbone of every debugging session in this lab and every prod incident you will face.

```
LEVEL 1 — What is the state?
  kubectl get pod <name> -n <ns>
  → Status, Ready, Restarts, Age
  → If Pending: go to scheduling (Lab 05)
  → If CrashLoopBackOff: go to Level 2 + logs --previous
  → If Running but 0/1 Ready: go to probes (Lab 04)

LEVEL 2 — Why is it in that state?
  kubectl describe pod <name> -n <ns>
  → Events section (bottom of output) — always read this first
  → Container state, last state, exit code, reason
  → Probe configuration if probe-related

LEVEL 3 — What did the app say?
  kubectl logs <name> -n <ns>
  kubectl logs <name> -n <ns> --previous
  → App-level error messages
  → Stack traces, connection errors, config errors

LEVEL 4 — What is it doing right now?
  kubectl exec -it <name> -n <ns> -- sh
  → Check process, ports, filesystem, env vars
  → Only for Running containers

LEVEL 5 — What does the cluster see?
  kubectl get events -n <ns> --sort-by='.lastTimestamp'
  → All events across all resources in the namespace
  → Catches things describe misses
```

---

```bash
# Create the lab namespace
kubectl create namespace lab07
kubectl config set-context --current --namespace=lab07
```

## Scenario 01 — Multi-Container Pod — Logs From the Wrong Container

### What You'll Break

A pod with multiple containers — a main app container and a sidecar. The main app is crashing but `kubectl logs` without specifying a container gives you the sidecar logs — which are perfectly healthy. This sends you on a false trail. Meanwhile the real error is in the main container and you're not looking there.

### Apply the Broken State

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-app
  namespace: lab08
spec:
  # Init container — runs first, must complete before main containers start
  initContainers:
  - name: db-migration
    image: busybox:1.35
    command: ["sh", "-c"]
    args:
    - |
      echo "[db-migration] Running database migrations..."
      sleep 3
      echo "[db-migration] Migrations complete."

  containers:
  # Main app — crashes immediately
  - name: main-app
    image: busybox:1.35
    command: ["sh", "-c"]
    args:
    - |
      echo "[main-app] Starting application..."
      echo "[main-app] ERROR: Cannot connect to database at db:5432"
      echo "[main-app] FATAL: Startup failed"
      exit 1
    resources:
      requests:
        memory: "32Mi"
        cpu: "50m"
      limits:
        memory: "64Mi"
        cpu: "100m"

  # Sidecar — log shipper, runs fine
  - name: log-shipper
    image: busybox:1.35
    command: ["sh", "-c"]
    args:
    - |
      echo "[log-shipper] Starting log collection..."
      while true; do
        echo "[log-shipper] Collecting logs... $(date)"
        sleep 10
      done
    resources:
      requests:
        memory: "32Mi"
        cpu: "50m"
      limits:
        memory: "64Mi"
        cpu: "100m"
EOF
```

### Symptoms You Will Observe

```bash
kubectl get pod multi-container-app -n lab08
# NAME                    READY   STATUS             RESTARTS   AGE
# multi-container-app     1/2     CrashLoopBackOff   3          45s
# 1/2 means: 1 of 2 containers is Ready (log-shipper), 1 is not (main-app)
```

### The Wrong Approach — What a Junior Does

```bash
# Without -c flag, kubectl logs picks the first container alphabetically
# or the first listed — which may NOT be the crashing one
kubectl logs multi-container-app -n lab08
# [log-shipper] Starting log collection...
# [log-shipper] Collecting logs... Mon Jan 1 00:00:10 UTC 2024
# [log-shipper] Collecting logs... Mon Jan 1 00:00:20 UTC 2024
# Everything looks fine — you'd chase a ghost
```

### The Right Approach — Systematic Container Identification

```bash
# Step 1 — How many containers? What are their names?
kubectl get pod multi-container-app -n lab08 \
  -o jsonpath='{.spec.containers[*].name}'
# main-app log-shipper

# Step 2 — Check init containers too
kubectl get pod multi-container-app -n lab08 \
  -o jsonpath='{.spec.initContainers[*].name}'
# db-migration

# Step 3 — Check the state of EACH container
kubectl get pod multi-container-app -n lab08 -o json | \
  jq '.status.containerStatuses[] | {name: .name, ready: .ready, restarts: .restartCount, state: .state}'
# {
#   "name": "log-shipper",
#   "ready": true,
#   "restarts": 0,
#   "state": {"running": {...}}
# }
# {
#   "name": "main-app",
#   "ready": false,
#   "restarts": 4,
#   "state": {"waiting": {"reason": "CrashLoopBackOff"}}
# }
# main-app has restarts — THAT is the problem container

# Step 4 — Describe to see all container states at once
kubectl describe pod multi-container-app -n lab08
# Look for the "Containers:" section
# main-app:
#   State: Waiting (CrashLoopBackOff)
#   Last State: Terminated (Exit Code: 1)
#   Restart Count: 4
# log-shipper:
#   State: Running
#   Restart Count: 0

# Step 5 — Get logs from the RIGHT container
kubectl logs multi-container-app -n lab08 -c main-app
# [main-app] Starting application...
# [main-app] ERROR: Cannot connect to database at db:5432
# [main-app] FATAL: Startup failed

# Step 6 — Get previous run logs (before the last crash)
kubectl logs multi-container-app -n lab08 -c main-app --previous
# Same error — confirms it's consistent

# Step 7 — Get init container logs (useful for migration failures)
kubectl logs multi-container-app -n lab08 -c db-migration
# [db-migration] Running database migrations...
# [db-migration] Migrations complete.
# Init container was fine — problem is in main-app startup
```

### All Log Flags You Need

```bash
# Specify container in multi-container pod
kubectl logs <pod> -n <ns> -c <container-name>

# Previous container instance (after crash/restart)
kubectl logs <pod> -n <ns> -c <container-name> --previous

# Stream logs in real time
kubectl logs <pod> -n <ns> -c <container-name> -f

# Last N lines only
kubectl logs <pod> -n <ns> -c <container-name> --tail=50

# Logs since a time period
kubectl logs <pod> -n <ns> -c <container-name> --since=1h
kubectl logs <pod> -n <ns> -c <container-name> --since-time="2024-01-01T00:00:00Z"

# All containers in pod at once (prefix shows which container)
kubectl logs <pod> -n <ns> --all-containers=true --prefix=true

# Logs from all pods in a deployment
kubectl logs deployment/<name> -n <ns> --all-containers=true

# Logs from all pods matching a label
kubectl logs -l app=web-app -n <ns> --all-containers=true --prefix=true
```

### The Init Container Sequence

```
Pod lifecycle with init containers:

[Init: db-migration]  → runs first, must exit 0
        ↓
[Init: config-setup]  → runs second (if multiple init containers)
        ↓
[Running: main-app + log-shipper]  → all main containers start together

If an init container fails:
  Pod status shows: Init:CrashLoopBackOff  or  Init:Error
  Main containers NEVER start
  Logs: kubectl logs <pod> -c <init-container-name>
```

```bash
# Simulate init container failure
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: init-fail-pod
  namespace: lab08
spec:
  initContainers:
  - name: failing-init
    image: busybox:1.35
    command: ["sh", "-c", "echo 'Init failed'; exit 1"]
    resources:
      requests:
        memory: "32Mi"
        cpu: "50m"
      limits:
        memory: "64Mi"
        cpu: "100m"
  containers:
  - name: main-app
    image: nginx:1.25
    resources:
      requests:
        memory: "64Mi"
        cpu: "100m"
      limits:
        memory: "128Mi"
        cpu: "200m"
EOF

kubectl get pod init-fail-pod -n lab08
# NAME            READY   STATUS                  RESTARTS   AGE
# init-fail-pod   0/1     Init:CrashLoopBackOff   2          30s
# Status "Init:" prefix means an init container is failing

kubectl logs init-fail-pod -n lab08 -c failing-init
# Init failed

kubectl logs init-fail-pod -n lab08 -c failing-init --previous
# Init failed  ← same error, confirms it's consistent
```

### Prod Wisdom

In any multi-container pod, **always check container names and their restart counts before reading logs.** `kubectl describe pod` shows the restart count per container — the one with non-zero restarts is where your logs investigation starts. Using `kubectl logs` without `-c` in a multi-container pod is a trap. The first container alphabetically is returned, not the crashing one. Also always check init containers when `Status` starts with `Init:` — the main app never ran, so the problem is entirely in the init phase.

---

## Scenario 02 — Dead Pod — Finding Evidence After the Crash

### What You'll Break

A pod that crashes, gets past its restart backoff limit, and is either gone or so deep in backoff that you can't run `exec` on it. The evidence is ephemeral — K8s only retains the last terminated container's logs and events for a limited time. This scenario teaches you where to look before the evidence disappears.

### Apply the Broken State

```bash
# A pod that crashes immediately every time
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: evidence-pod
  namespace: lab08
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ["sh", "-c"]
    args:
    - |
      echo "Starting up..."
      echo "Loading config from /etc/app/config.yaml"
      echo "ERROR: config file not found"
      echo "PANIC: nil pointer dereference at 0x00000000"
      exit 139   # Simulates a segfault (SIGSEGV)
    resources:
      requests:
        memory: "32Mi"
        cpu: "50m"
      limits:
        memory: "64Mi"
        cpu: "100m"
EOF
```

```bash
kubectl get pod evidence-pod -n lab08 -w
# Watch the restart cycle
# STATUS: Running → CrashLoopBackOff → Running → CrashLoopBackOff...
# RESTARTS climbs: 1 → 2 → 3 → 4 → 5...
# Backoff delay grows: 10s → 20s → 40s → 80s → 160s → 300s cap
```

### Extracting Evidence Before It Disappears

```bash
# TECHNIQUE 1 — Previous container logs (most important)
# Available as long as the pod object exists
kubectl logs evidence-pod -n lab08 --previous
# Starting up...
# Loading config from /etc/app/config.yaml
# ERROR: config file not found
# PANIC: nil pointer dereference at 0x00000000

# TECHNIQUE 2 — Exit code tells you HOW it died
kubectl get pod evidence-pod -n lab08 -o json | \
  jq '.status.containerStatuses[0].lastState.terminated'
# {
#   "exitCode": 139,          ← 139 = SIGSEGV (segmentation fault)
#   "finishedAt": "...",
#   "reason": "Error",
#   "startedAt": "..."
# }

# Exit code reference:
# 0   = clean exit (success)
# 1   = generic error
# 137 = OOMKilled (SIGKILL from kernel)
# 139 = Segmentation fault (SIGSEGV)
# 143 = SIGTERM (graceful shutdown — expected during pod termination)
# 130 = SIGINT (Ctrl+C)

# TECHNIQUE 3 — Describe for last state + events
kubectl describe pod evidence-pod -n lab08
# Containers:
#   app:
#     Last State: Terminated
#       Reason:    Error
#       Exit Code: 139
#       Started:   ...
#       Finished:  ...
#     Restart Count: 5

# Events section:
# Warning  BackOff  kubelet  Back-off restarting failed container

# TECHNIQUE 4 — Cluster events for this pod
kubectl get events -n lab08 \
  --field-selector involvedObject.name=evidence-pod \
  --sort-by='.lastTimestamp'
# LAST SEEN   TYPE      REASON    OBJECT         MESSAGE
# 5m          Normal    Pulled    Pod/evidence-pod  Successfully pulled image
# 5m          Normal    Started   Pod/evidence-pod  Started container app
# 5m          Warning   BackOff   Pod/evidence-pod  Back-off restarting failed container
```

### The Ephemeral Debug Container — For Running Pods With No Shell

```bash
# When a pod is Running but has no shell (distroless images, scratch containers)
# You can inject an ephemeral debug container without restarting the pod

# Example: pod running a distroless image with no shell
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: no-shell-pod
  namespace: lab08
spec:
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        memory: "64Mi"
        cpu: "100m"
      limits:
        memory: "128Mi"
        cpu: "200m"
EOF

kubectl wait --for=condition=ready pod/no-shell-pod -n lab08 --timeout=30s

# Inject an ephemeral debug container (K8s 1.23+)
kubectl debug -it no-shell-pod \
  --image=busybox:1.35 \
  --target=app \
  -n lab08 \
  -- sh

# Inside the ephemeral container:
# You can inspect the same process namespace as the app
# ps aux
# netstat -tlnp
# ls /proc/1/fd    (app's open file descriptors)
# cat /proc/1/environ | tr '\0' '\n'  (app's environment variables)
# exit
```

### Namespace-Wide Event Sweep

```bash
# When you don't know which resource is broken — sweep the whole namespace
kubectl get events -n lab08 --sort-by='.lastTimestamp'
# Shows ALL events in the namespace, most recent last
# Warning events are your signal

# Filter only warnings
kubectl get events -n lab08 \
  --field-selector type=Warning \
  --sort-by='.lastTimestamp'

# Get more detail on events
kubectl get events -n lab08 -o wide --sort-by='.lastTimestamp'

# Watch events in real time
kubectl get events -n lab08 -w
```

### When the Pod Is Completely Gone

```bash
# If the pod is deleted, logs are gone — that's why you need external log aggregation
# But the ReplicaSet/Deployment may have created a replacement
# Find the replacement pod
kubectl get pods -n lab08 -l app=<label> --sort-by='.metadata.creationTimestamp'

# Get logs from a specific previous pod via label if still in terminating state
kubectl logs -l app=<label> -n lab08 --previous

# Check if a Job completed or failed
kubectl get jobs -n lab08
kubectl describe job <name> -n lab08
# Look for: Succeeded / Failed count
```

### Prod Wisdom

`--previous` is your most important forensics tool. It retrieves logs from the last terminated instance of the container — available as long as the pod object still exists in the cluster. The moment someone runs `kubectl delete pod`, it's gone. In prod, always have external log aggregation (ELK, Loki, CloudWatch) — `kubectl logs` is ephemeral by nature. But in the heat of an incident, `--previous` and exit codes are what you reach for first.

---

## Scenario 03 — Live Incident — Symptom to Root Cause

### What You'll Build and Break

This is the capstone of the entire Level 1 lab series. A realistic prod-style broken environment — multiple interacting resources, several things wrong at once, no hints. You get a symptom and a namespace. Your job is to find the root cause using nothing but the systematic workflow.

### Set Up the Broken Environment

```bash
# DO NOT READ AHEAD — apply this and then start debugging from scratch
# Treat this as a real incident

cat <<EOF | kubectl apply -f -
# Resource 1 — ConfigMap with app config
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: lab08
data:
  DB_HOST: "postgres-svc"
  DB_PORT: "5432"
  LOG_LEVEL: "info"
---
# Resource 2 — Secret (missing a required key)
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: lab08
type: Opaque
stringData:
  DB_USER: "appuser"
  # DB_PASSWORD is intentionally missing
---
# Resource 3 — Backend database (actually just nginx simulating it)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: lab08
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: nginx:1.25
        ports:
        - containerPort: 5432
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
---
# Resource 4 — Database Service (wrong selector)
apiVersion: v1
kind: Service
metadata:
  name: postgres-svc
  namespace: lab08
spec:
  selector:
    app: database          # WRONG: pods have label app=postgres
  ports:
  - port: 5432
    targetPort: 5432
---
# Resource 5 — Main application (references missing secret key)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: lab08
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: app
        image: busybox:1.35
        command: ["sh", "-c"]
        args:
        - |
          echo "Starting web-app..."
          echo "DB_HOST: ${DB_HOST}"
          echo "DB_USER: ${DB_USER}"
          echo "Connecting to ${DB_HOST}:${DB_PORT}..."
          sleep 3600
        envFrom:
        - configMapRef:
            name: app-config
        env:
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: DB_USER
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: DB_PASSWORD     # This key doesn't exist in the secret
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
---
# Resource 6 — Service for web-app (correct)
apiVersion: v1
kind: Service
metadata:
  name: web-app-svc
  namespace: lab08
spec:
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
---
# Resource 7 — HPA (referencing a deployment that can't start)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
  namespace: lab08
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
EOF
```

**The symptom you've been given:**
> "Users can't reach the web-app. Pods seem to be failing but we're not sure why. The database might also be having issues."

**Now debug. Use only the workflow. No reading ahead.**

---

### The Debug Walkthrough — Step by Step

```bash
# ============================================================
# LEVEL 1 — What is the state of everything in this namespace?
# ============================================================

kubectl get all -n lab08
# NAME                          READY   STATUS                       RESTARTS
# pod/web-app-xxx-aaa           0/1     CreateContainerConfigError   0
# pod/web-app-xxx-bbb           0/1     CreateContainerConfigError   0
# pod/postgres-xxx-ccc          1/1     Running                      0
#
# NAME                TYPE        CLUSTER-IP    PORT(S)
# service/postgres-svc ClusterIP  10.96.x.x     5432/TCP
# service/web-app-svc  ClusterIP  10.96.y.y     80/TCP
#
# DEPLOYMENT          READY
# postgres            1/1     ← healthy
# web-app             0/2     ← 0 of 2 pods ready

# First signal: web-app pods in CreateContainerConfigError
# postgres looks fine

# ============================================================
# LEVEL 2 — Why is web-app in CreateContainerConfigError?
# ============================================================

kubectl describe pod $(kubectl get pod -n lab08 -l app=web-app \
  -o jsonpath='{.items[0].metadata.name}') -n lab08

# Events section:
# Warning  Failed  kubelet  Error: secret "app-secrets" not found
# OR:
# Warning  Failed  kubelet  couldn't find key DB_PASSWORD in Secret lab08/app-secrets

# Root cause #1 found: Secret "app-secrets" exists but key "DB_PASSWORD" is missing

# ============================================================
# LEVEL 3 — Verify the secret
# ============================================================

kubectl get secret app-secrets -n lab08 -o json | \
  jq '.data | keys'
# ["DB_USER"]
# DB_PASSWORD key is missing from the secret

# Fix #1 — Add the missing key to the secret
kubectl patch secret app-secrets -n lab08 \
  --type='json' \
  -p='[{"op":"add","path":"/data/DB_PASSWORD","value":"'$(echo -n "securepassword" | base64)'"}]'

# Verify fix
kubectl get secret app-secrets -n lab08 -o json | jq '.data | keys'
# ["DB_PASSWORD", "DB_USER"]

# ============================================================
# LEVEL 4 — Did fixing the secret fix the pods?
# ============================================================

kubectl get pods -n lab08 -l app=web-app
# Now pods should start — watch them
kubectl get pods -n lab08 -l app=web-app -w
# web-app pods now transition to Running

kubectl logs $(kubectl get pod -n lab08 -l app=web-app \
  -o jsonpath='{.items[0].metadata.name}') -n lab08
# Starting web-app...
# DB_HOST: postgres-svc
# DB_USER: appuser
# Connecting to postgres-svc:5432...
# App is running — now investigate the database connectivity

# ============================================================
# LEVEL 5 — Is the database service actually reachable?
# ============================================================

kubectl get endpoints postgres-svc -n lab08
# NAME           ENDPOINTS   AGE
# postgres-svc   <none>      10m
# No endpoints — Service selector doesn't match any pods

# Root cause #2 found: postgres-svc has wrong selector

kubectl describe svc postgres-svc -n lab08
# Selector: app=database

kubectl get pods -n lab08 -l app=postgres
# pod/postgres-xxx   1/1   Running
# Pods have label app=postgres, Service selects app=database — mismatch

# Fix #2 — Correct the Service selector
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: postgres-svc
  namespace: lab08
spec:
  selector:
    app: postgres           # Fixed: matches pod label
  ports:
  - port: 5432
    targetPort: 5432
EOF

kubectl get endpoints postgres-svc -n lab08
# ENDPOINTS: 10.244.x.x:5432 ← now populated

# ============================================================
# LEVEL 6 — Final verification sweep
# ============================================================

kubectl get all -n lab08
# web-app: 2/2 Ready
# postgres: 1/1 Ready
# All services have endpoints

kubectl get events -n lab08 \
  --field-selector type=Warning \
  --sort-by='.lastTimestamp'
# No new Warning events

# Verify HPA is functioning
kubectl get hpa -n lab08
# NAME          REFERENCE        TARGETS   MINPODS   MAXPODS   REPLICAS
# web-app-hpa   Deployment/web-app   x%/70%   2         10        2
# HPA now has a functioning deployment to target
```

### Post-Incident Summary — What Was Wrong

```
Issue #1: Secret "app-secrets" missing key "DB_PASSWORD"
  Signal:   CreateContainerConfigError on web-app pods
  Command:  kubectl describe pod → Events showed missing secret key
  Fix:      kubectl patch secret to add DB_PASSWORD key

Issue #2: postgres-svc selector mismatch
  Signal:   kubectl get endpoints postgres-svc showed <none>
  Command:  kubectl describe svc → Selector: app=database (wrong)
            kubectl get pods --show-labels → pods have app=postgres
  Fix:      Update Service selector to app=postgres
```

### The Workflow That Found Both Issues

```
kubectl get all          → spotted web-app at 0/2 Ready
kubectl describe pod     → Events revealed missing secret key
kubectl get secret       → confirmed key was missing
(fix #1)
kubectl logs             → app started, confirmed DB connection attempt
kubectl get endpoints    → postgres-svc showed <none>
kubectl describe svc     → wrong selector
kubectl get pods --show-labels → confirmed correct label
(fix #2)
kubectl get all          → final verification sweep
```

**9 commands. 2 root causes found. Zero guessing.**

---

## The Complete kubectl Debugging Reference

### Pod State Commands

```bash
# Full status overview of everything in namespace
kubectl get all -n <namespace>

# Pod state with restart count
kubectl get pods -n <namespace>
kubectl get pods -n <namespace> -w        # Watch in real time
kubectl get pods -n <namespace> -o wide   # Include node and IP

# Specific pod detail
kubectl describe pod <name> -n <namespace>

# Pod YAML (full spec + status)
kubectl get pod <name> -n <namespace> -o yaml

# Pod IP and node
kubectl get pod <name> -n <namespace> \
  -o jsonpath='{.status.podIP} {.spec.nodeName}'
```

### Log Commands

```bash
# Current logs
kubectl logs <pod> -n <ns>
kubectl logs <pod> -n <ns> -c <container>

# Previous container instance (after restart)
kubectl logs <pod> -n <ns> --previous
kubectl logs <pod> -n <ns> -c <container> --previous

# Stream
kubectl logs <pod> -n <ns> -f
kubectl logs <pod> -n <ns> -c <container> -f

# Last N lines
kubectl logs <pod> -n <ns> --tail=100

# Time-bounded
kubectl logs <pod> -n <ns> --since=30m
kubectl logs <pod> -n <ns> --since-time="2024-01-01T12:00:00Z"

# All containers in a pod
kubectl logs <pod> -n <ns> --all-containers=true --prefix=true

# All pods matching a label (across a deployment)
kubectl logs -l app=<label> -n <ns> --all-containers=true --prefix=true

# Deployment logs
kubectl logs deployment/<name> -n <ns>
```

### Exec and Debug Commands

```bash
# Shell into running container
kubectl exec -it <pod> -n <ns> -- sh
kubectl exec -it <pod> -n <ns> -c <container> -- sh

# Run a single command
kubectl exec <pod> -n <ns> -- curl -s http://localhost:8080/health
kubectl exec <pod> -n <ns> -- env | grep DB_
kubectl exec <pod> -n <ns> -- cat /etc/app/config.yaml
kubectl exec <pod> -n <ns> -- ss -tlnp

# Ephemeral debug container (for pods with no shell)
kubectl debug -it <pod> --image=busybox:1.35 --target=<container> -n <ns> -- sh

# Run a one-off debug pod in the cluster
kubectl run debug --image=busybox:1.35 --rm -it --restart=Never -n <ns> -- sh

# Debug a specific node
kubectl debug node/<node-name> -it --image=busybox:1.35
```

### Event Commands

```bash
# All events in namespace
kubectl get events -n <namespace>

# Sorted by time (most recent last)
kubectl get events -n <namespace> --sort-by='.lastTimestamp'

# Only warnings
kubectl get events -n <namespace> --field-selector type=Warning

# Events for a specific resource
kubectl get events -n <namespace> \
  --field-selector involvedObject.name=<pod-name>

# Watch events in real time
kubectl get events -n <namespace> -w

# Events across all namespaces
kubectl get events -A --sort-by='.lastTimestamp'
```

### Resource Inspection Commands

```bash
# Check if a resource exists
kubectl get <resource> <name> -n <namespace>

# Full resource detail
kubectl describe <resource> <name> -n <namespace>

# Raw YAML (for diffing or deep inspection)
kubectl get <resource> <name> -n <namespace> -o yaml

# JSONPath extraction
kubectl get pod <name> -n <ns> -o jsonpath='{.status.phase}'
kubectl get pod <name> -n <ns> -o jsonpath='{.spec.containers[*].name}'
kubectl get pod <name> -n <ns> -o jsonpath='{.status.containerStatuses[*].restartCount}'

# Check RBAC (from Lab 06)
kubectl auth can-i <verb> <resource> \
  --as=system:serviceaccount:<ns>:<sa> -n <ns>

# Check resource usage (requires metrics-server)
kubectl top pods -n <namespace>
kubectl top nodes
```

---

## The Exit Code Reference — Memorise This

```
Exit Code  Signal        Meaning
──────────────────────────────────────────────────────────
0          –             Clean exit (success)
1          –             Generic application error
2          –             Misuse of shell command
126        –             Command cannot be executed
127        –             Command not found
128        –             Invalid exit argument
130        SIGINT        Container interrupted (Ctrl+C)
137        SIGKILL       OOMKilled or force-killed
139        SIGSEGV       Segmentation fault
143        SIGTERM       Graceful termination (expected on pod deletion)
──────────────────────────────────────────────────────────
137 = OOMKilled  →  increase memory limit (Lab 05)
139 = Segfault   →  application bug or memory corruption
143 = SIGTERM    →  expected — pod is being gracefully shut down
anything else    →  read the application logs
```

---

## Prod Wisdom — The Senior Engineer Mindset

Three things that define a senior engineer's debugging approach:

**1. They run the same sequence every time, regardless of the symptom.** `get → describe → logs → logs --previous → events`. No shortcuts. No jumping to conclusions. The sequence is fast precisely because it's automatic — no decision-making overhead at 2am.

**2. They read exit codes before reading logs.** Exit code 137 = OOM, go straight to memory limits. Exit code 1 = app error, read the logs. Exit code 143 = normal termination, check why the pod was terminated, not why the app exited. The exit code narrows the search space before you read a single log line.

**3. They always end with a sweep.** After fixing the immediate issue, they run `kubectl get all` and `kubectl get events --field-selector type=Warning` to make sure nothing else is silently broken. The incident you just fixed may have masked a second problem — find it now, not in 20 minutes when users complain again.

The master debug sequence — run it the same way every time:
```
kubectl get all -n <ns>
  → identify the broken resource

kubectl describe pod <name> -n <ns>
  → read Events section, exit code, last state

kubectl logs <name> -n <ns> --previous
  → what did the app say before it died?

kubectl get events -n <ns> --field-selector type=Warning --sort-by='.lastTimestamp'
  → what has the cluster been saying about this namespace?

kubectl get endpoints <svc> -n <ns>
  → is traffic actually reaching pods?
```

Five commands. Every incident. Every time.

---

## Cleanup

```bash
kubectl delete namespace lab08
```

---

## Level 1 Complete.

You have now built and broken all 8 core labs:

| Lab | Topic | Status |
|-----|-------|--------|
| 01 | Pod Troubleshooting | ✓ |
| 02 | Deployments & Rollouts | ✓ |
| 03 | Services & DNS | ✓ |
| 04 | Probes | ✓ |
| 05 | Resource Limits & Scheduling | ✓ |
| 06 | RBAC | ✓ |
| 07 | Ingress | ✓ |
| 08 | Logging & Debugging Workflow | ✓ |

**Do not move to Level 2 until you can run every scenario in Level 1 from memory.**

Break them again. Fix them faster. The goal is zero hesitation.

*Level 2 — The Gap 7 begins when you are ready.*
