# Lab 04 — Probes

## What This Lab Is About

Probes are how Kubernetes knows whether your container is alive, ready to serve traffic, and started successfully. Get them right and your app handles restarts, deployments, and failures gracefully. Get them wrong and you get one of the most frustrating class of bugs in K8s — the app looks healthy, pods are Running, but traffic is going to dead containers, or healthy containers are being killed in a loop.

There are three probe types. Each answers a different question:

- **Liveness** — Is this container still alive? If not, kill it and restart.
- **Readiness** — Is this container ready to receive traffic? If not, remove it from Service Endpoints.
- **Startup** — Has this container finished starting up? Blocks liveness and readiness until it passes.

This lab covers 4 scenarios: a liveness probe killing a healthy app, a readiness probe sending traffic to an unready pod, a missing startup probe crashing a slow-starting app, and the correct production probe configuration that protects your app without fighting it.

> A misconfigured probe is worse than no probe. No probe means K8s is blind. A wrong probe means K8s is actively working against your app.

---

## Environment

- **Cluster:** KIND (Kubernetes IN Docker)
- **K8s Version:** v1.27.3
- **Node:** Single node (`kind-control-plane`)
- **Namespace for this lab:** `lab04`

```bash
kubectl create namespace lab04
kubectl config set-context --current --namespace=lab04
```

---

## The 4 Scenarios

| # | Scenario | What You'll Learn |
|---|---|---|
| 01 | Liveness probe killing a healthy app | Wrong path, wrong port, too aggressive timing |
| 02 | Readiness probe — traffic to unready pods | Missing readiness lets broken pods receive traffic |
| 03 | Missing startup probe — slow app crashes in loop | `initialDelaySeconds` trap, startup probe solution |
| 04 | The correct prod probe configuration | All three probes working together properly |

---

## Scenario 01 — Liveness Probe Killing a Healthy App

### What You'll Break

A liveness probe configured with the wrong path or port. The container is running fine, the app is healthy — but Kubernetes thinks it's dead and kills it in a loop. This creates a `CrashLoopBackOff` that looks identical to an actual app crash, which makes it one of the hardest failures to diagnose if you don't know to check probe config.

### Apply the Broken State

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: bad-liveness
  namespace: lab04
spec:
  containers:
  - name: app
    image: nginx:1.25
    ports:
    - containerPort: 80
    livenessProbe:
      httpGet:
        path: /healthz          # WRONG: nginx has no /healthz endpoint
        port: 9090              # WRONG: nginx listens on 80, not 9090
      initialDelaySeconds: 3
      periodSeconds: 5
      failureThreshold: 2       # Only 2 failures before kill — too aggressive
EOF
```

### Symptoms You Will Observe

```bash
kubectl get pod bad-liveness -n lab04 -w
```

Watch the progression:
```
NAME           READY   STATUS    RESTARTS   AGE
bad-liveness   1/1     Running   0          8s
bad-liveness   1/1     Running   1          18s
bad-liveness   1/1     Running   2          28s
bad-liveness   0/1     CrashLoopBackOff   3   38s
```

The app is running perfectly. nginx is serving traffic. But Kubernetes is killing it every 10 seconds because the liveness probe keeps failing. `RESTARTS` climbs constantly.

```bash
kubectl get pod bad-liveness -n lab04
# NAME           READY   STATUS             RESTARTS   AGE
# bad-liveness   0/1     CrashLoopBackOff   5          1m
```

This looks identical to Scenario 02 from Lab 01 — a real app crash. The status is the same. The restart count is the same. You must dig deeper.

### Investigate

```bash
# Step 1 — Check restart count and status
kubectl get pod bad-liveness -n lab04

# Step 2 — Check logs (the app logs are clean — nginx is fine)
kubectl logs bad-liveness -n lab04
# Normal nginx access logs — no errors
# This is the KEY signal: logs are clean but pod is restarting = probe issue

kubectl logs bad-liveness -n lab04 --previous
# Also clean — app never actually crashed

# Step 3 — Describe the pod — look at Events AND Liveness probe config
kubectl describe pod bad-liveness -n lab04

# In Events you will see:
# Warning  Unhealthy  kubelet  Liveness probe failed:
# Get "http://10.244.x.x:9090/healthz": dial tcp 10.244.x.x:9090: connect: connection refused

# Also in the container spec section:
# Liveness:  http-get http://:9090/healthz delay=3s timeout=1s period=5s #success=1 #failure=2

# Step 4 — Confirm the app IS actually healthy
kubectl exec bad-liveness -n lab04 -- curl -s http://localhost:80
# Returns nginx HTML — app is perfectly healthy
```

The logs are clean. The app responds to curl. But Kubernetes is killing it. **This combination — clean logs + restarts — means probe misconfiguration.**

### Fix

```bash
kubectl delete pod bad-liveness -n lab04

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: bad-liveness
  namespace: lab04
spec:
  containers:
  - name: app
    image: nginx:1.25
    ports:
    - containerPort: 80
    livenessProbe:
      httpGet:
        path: /                 # Fixed: nginx serves / correctly
        port: 80                # Fixed: correct port
      initialDelaySeconds: 10   # Give app time to start
      periodSeconds: 10         # Check every 10s
      failureThreshold: 3       # 3 failures before kill — not 2
      timeoutSeconds: 5         # Wait up to 5s for response
EOF

kubectl get pod bad-liveness -n lab04 -w
# Stays Running, RESTARTS stays at 0
```

### The Liveness Probe Timing Parameters — Know Every One

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 10   # Wait this long after container starts before first probe
                             # Set this LONGER than your app's startup time
  periodSeconds: 10         # How often to run the probe
  timeoutSeconds: 5         # How long to wait for a response before counting as failure
  failureThreshold: 3       # Consecutive failures before action is taken
  successThreshold: 1       # Consecutive successes to be considered healthy (liveness: always 1)
```

### Prod Wisdom

If pods are restarting but the application logs show no errors — **check the liveness probe first**. Clean logs plus restarts is the signature of a probe killing a healthy container. In prod, liveness probe thresholds should be conservative: `failureThreshold: 3` minimum, `periodSeconds: 10` minimum. An aggressive liveness probe on a momentarily slow app will restart it mid-request, causing dropped traffic and making the incident worse.

---

## Scenario 02 — Readiness Probe — Traffic to Unready Pods

### What You'll Break

Two situations — first, a pod with no readiness probe receiving traffic before it's ready (causing request failures during startup). Second, a readiness probe that's too strict, removing healthy pods from rotation and causing unnecessary traffic drops. Readiness is about traffic routing, not container health.

### Break It Part A — No Readiness Probe (Traffic Too Early)

```bash
# Simulates an app that takes 20s to be ready (slow startup, DB connection warmup)
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: slow-app
  namespace: lab04
spec:
  replicas: 2
  selector:
    matchLabels:
      app: slow-app
  template:
    metadata:
      labels:
        app: slow-app
    spec:
      containers:
      - name: app
        image: nginx:1.25
        command: ["sh", "-c", "sleep 20 && nginx -g 'daemon off;'"]
        ports:
        - containerPort: 80
        # NO readinessProbe — K8s marks pod Ready immediately
---
apiVersion: v1
kind: Service
metadata:
  name: slow-app-svc
  namespace: lab04
spec:
  selector:
    app: slow-app
  ports:
  - port: 80
    targetPort: 80
EOF
```

```bash
# Watch — pods become 1/1 Running almost immediately despite app not being ready
kubectl get pods -n lab04 -w
# NAME               READY   STATUS    RESTARTS   AGE
# slow-app-xxx-aaa   1/1     Running   0          5s   <-- marked Ready, app is sleeping
# slow-app-xxx-bbb   1/1     Running   0          5s   <-- same

# Service already has these as Endpoints — traffic goes to them
kubectl get endpoints slow-app-svc -n lab04
# ENDPOINTS: 10.244.x.x:80, 10.244.y.y:80  <-- but app isn't serving yet
```

Any traffic hitting these pods in the first 20 seconds gets `connection refused`.

### Break It Part B — Readiness Probe Too Strict

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: strict-readiness
  namespace: lab04
spec:
  replicas: 1
  selector:
    matchLabels:
      app: strict-readiness
  template:
    metadata:
      labels:
        app: strict-readiness
    spec:
      containers:
      - name: app
        image: nginx:1.25
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /ready
            port: 80
          initialDelaySeconds: 2
          periodSeconds: 3
          failureThreshold: 1   # ONE failure = removed from Service immediately
          successThreshold: 5   # Needs 5 consecutive successes to be added back
EOF
```

```bash
kubectl get pod -n lab04 -l app=strict-readiness
# NAME                          READY   STATUS    RESTARTS   AGE
# strict-readiness-xxx-yyy      0/1     Running   0          30s
# Will flicker between 0/1 and 1/1 constantly
# nginx returns 404 for /ready which counts as failure
# 1 failure removes it, needs 5 successes to return — very unstable
```

### Investigate

```bash
# Step 1 — Check READY column (0/1 means readiness failing)
kubectl get pods -n lab04

# Step 2 — Check Endpoints — is pod in or out of rotation?
kubectl get endpoints slow-app-svc -n lab04
# Empty or populated tells you if pod is serving traffic

# Step 3 — Describe to see readiness probe failures
kubectl describe pod $(kubectl get pod -n lab04 -l app=strict-readiness \
  -o jsonpath='{.items[0].metadata.name}') -n lab04

# Events will show:
# Warning  Unhealthy  kubelet  Readiness probe failed:
# HTTP probe failed with statuscode: 404

# KEY DISTINCTION:
# Readiness failure → pod stays Running, just removed from Endpoints
# Liveness failure  → pod gets KILLED and restarted
# Check RESTARTS counter:
#   RESTARTS > 0 and climbing = liveness issue
#   RESTARTS = 0 but READY = 0/1 = readiness issue
```

### Fix — Correct Readiness Probe

```bash
kubectl delete deployment slow-app strict-readiness -n lab04

cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: production-app
  namespace: lab04
spec:
  replicas: 2
  selector:
    matchLabels:
      app: production-app
  template:
    metadata:
      labels:
        app: production-app
    spec:
      containers:
      - name: app
        image: nginx:1.25
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /             # Correct path that returns 200
            port: 80
          initialDelaySeconds: 5  # Wait for startup
          periodSeconds: 5        # Check every 5s
          failureThreshold: 3     # 3 failures before removal from Endpoints
          successThreshold: 1     # 1 success to be added back
EOF

kubectl get pods -n lab04
# Both pods show 1/1 Running — fully ready
```

### Liveness vs Readiness — The Critical Distinction

```
Liveness probe fails:
  → Container is KILLED
  → Pod restarts
  → RESTARTS counter increments
  → Use for: detecting a deadlocked or frozen app

Readiness probe fails:
  → Container stays RUNNING
  → Pod removed from Service Endpoints
  → Traffic stops going to this pod
  → RESTARTS stays at 0, READY shows 0/1
  → Use for: detecting an app not yet ready for traffic
             (startup, dependency unavailable, overloaded)

Rule: Never use liveness for "is the app ready" checks
      Never use readiness to restart a stuck app
```

### Prod Wisdom

**Readiness and liveness serve completely different purposes — never conflate them.** A common mistake in prod is configuring a liveness probe on the same endpoint as readiness with aggressive thresholds. When the app is temporarily slow or warming up, the liveness probe kills it — causing a restart storm exactly when you need stability. Readiness controls traffic. Liveness controls container lifecycle. Keep them separate and configure liveness conservatively.

---

## Scenario 03 — Missing Startup Probe — Slow App Crashes in Loop

### What You'll Break

An application that takes a long time to start (legacy app, JVM warmup, ML model loading — common in real prod). Without a startup probe, developers compensate with a large `initialDelaySeconds` on liveness. This is fragile — too short and the liveness probe kills the app during startup, too long and real deadlocks take forever to detect. The startup probe is the correct solution.

### Break It — Large initialDelaySeconds Anti-Pattern

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: slow-start-bad
  namespace: lab04
spec:
  containers:
  - name: app
    image: busybox:1.35
    # Simulates an app that takes 40s to start
    command: ["sh", "-c", "echo 'starting...'; sleep 40; echo 'ready'; sleep 3600"]
    livenessProbe:
      exec:
        command: ["sh", "-c", "echo alive"]
      initialDelaySeconds: 10   # Too short — liveness starts before app is ready
      periodSeconds: 5
      failureThreshold: 3
EOF
```

```bash
kubectl get pod slow-start-bad -n lab04 -w
# The liveness probe starts at 10s
# App isn't ready until 40s
# Between 10s and 40s, if liveness probe checks an app-specific endpoint it would fail
# With exec "echo alive" it passes — but in real life with HTTP endpoint it wouldn't
```

Now the real-world broken version:

```bash
kubectl delete pod slow-start-bad -n lab04

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: slow-start-broken
  namespace: lab04
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ["sh", "-c", "sleep 40 && httpd -f -p 8080"]
    livenessProbe:
      tcpSocket:
        port: 8080              # Port only opens after 40s sleep
      initialDelaySeconds: 5    # Probe starts at 5s — port not open yet
      periodSeconds: 5
      failureThreshold: 3       # 3 failures at 5s, 10s, 15s = killed at 15s
EOF
```

```bash
kubectl get pod slow-start-broken -n lab04 -w
# Pod starts, liveness begins at 5s
# Fails at 5s, 10s, 15s (3 failures = failureThreshold)
# Killed at ~15s and restarted — never gets to 40s to actually start
# CrashLoopBackOff — but the app itself has no bug
```

### Fix — Startup Probe Is the Right Solution

```bash
kubectl delete pod slow-start-broken -n lab04

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: slow-start-fixed
  namespace: lab04
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ["sh", "-c", "sleep 40 && httpd -f -p 8080"]
    startupProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 12      # 12 * 5s = 60s max startup time allowed
                                 # Liveness/Readiness are BLOCKED until this passes
    livenessProbe:
      tcpSocket:
        port: 8080
      periodSeconds: 10
      failureThreshold: 3       # Only runs AFTER startupProbe succeeds
    readinessProbe:
      tcpSocket:
        port: 8080
      periodSeconds: 5
      failureThreshold: 3       # Only runs AFTER startupProbe succeeds
EOF
```

```bash
kubectl get pod slow-start-fixed -n lab04 -w
# Startup probe runs every 5s starting at 5s
# Fails for the first ~40s (app still sleeping) — but that's OK
# At ~40s httpd starts, startup probe passes
# NOW liveness and readiness kick in
# Pod shows 1/1 Running — no restarts
```

### How Startup Probe Works — The Sequence

```
Container starts
      │
      ▼
startupProbe runs (every periodSeconds)
      │
      ├── fails → try again (up to failureThreshold * periodSeconds total time)
      │           if failureThreshold exceeded → container killed
      │
      └── passes → startupProbe stops running forever
                        │
                        ▼
              livenessProbe starts running
              readinessProbe starts running
              (normal operation from here)
```

### Calculating Startup Probe failureThreshold

```
Max allowed startup time = failureThreshold × periodSeconds

Example: App can take up to 2 minutes to start
  periodSeconds: 10
  failureThreshold: 12   → 12 × 10 = 120 seconds max
  initialDelaySeconds: 0 → start checking immediately

Add 20-30% buffer for slow nodes or heavy load:
  failureThreshold: 15   → 15 × 10 = 150 seconds — safer
```

### Probe Types — Three Ways to Check

```yaml
# HTTP GET — returns 2xx or 3xx = success, anything else = failure
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
    httpHeaders:
    - name: Authorization
      value: Bearer mytoken

# TCP Socket — connection established = success, refused = failure
# Use when app doesn't have HTTP (databases, message brokers)
livenessProbe:
  tcpSocket:
    port: 5432

# Exec — command exit code 0 = success, non-zero = failure
# Use for custom health logic
livenessProbe:
  exec:
    command:
    - sh
    - -c
    - "redis-cli ping | grep PONG"
```

### Prod Wisdom

Any application with a startup time over 20 seconds **must** have a startup probe — not a large `initialDelaySeconds`. The `initialDelaySeconds` approach is fragile because it's a fixed delay regardless of actual startup time. The startup probe adapts — it keeps checking until the app is ready, up to your configured limit, then hands off to liveness and readiness. JVM apps, Python ML services, apps with heavy DB migrations — all of them need this pattern in prod.

---

## Scenario 04 — The Correct Production Probe Configuration

### What You'll Build

This scenario puts it all together. A production-grade probe configuration with all three probes working correctly, with proper timing, proper endpoints, and proper thresholds. This is the pattern you will use as your default for every deployment from now on.

### The Complete Production-Grade Setup

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prod-app
  namespace: lab04
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: prod-app
  template:
    metadata:
      labels:
        app: prod-app
    spec:
      containers:
      - name: app
        image: nginx:1.25
        ports:
        - name: http
          containerPort: 80

        # Startup probe — runs first, blocks liveness and readiness
        # Gives app up to 30s to start (6 * 5s)
        startupProbe:
          httpGet:
            path: /
            port: http
          initialDelaySeconds: 5
          periodSeconds: 5
          failureThreshold: 6
          timeoutSeconds: 3

        # Readiness probe — controls traffic routing
        # Removed from Service Endpoints if failing
        # Conservative: 3 failures before removal, 1 success to return
        readinessProbe:
          httpGet:
            path: /
            port: http
          initialDelaySeconds: 0   # Startup probe handles the delay
          periodSeconds: 5
          failureThreshold: 3
          successThreshold: 1
          timeoutSeconds: 3

        # Liveness probe — controls container lifecycle
        # Very conservative: only kills if truly broken
        # Different path from readiness if possible (deeper health check)
        livenessProbe:
          httpGet:
            path: /
            port: http
          initialDelaySeconds: 0   # Startup probe handles the delay
          periodSeconds: 15        # Check less frequently than readiness
          failureThreshold: 3
          successThreshold: 1
          timeoutSeconds: 5        # More generous timeout for liveness
EOF
```

```bash
kubectl rollout status deployment/prod-app -n lab04
# deployment "prod-app" successfully rolled out

kubectl get pods -n lab04
# All 3 pods: 1/1 Running, RESTARTS: 0
```

### Verify Each Probe Is Working

```bash
# Step 1 — Confirm pods are fully ready
kubectl get pods -n lab04

# Step 2 — Check probe config is applied correctly
kubectl describe pod $(kubectl get pod -n lab04 -l app=prod-app \
  -o jsonpath='{.items[0].metadata.name}') -n lab04 | grep -A 8 "Liveness\|Readiness\|Startup"

# Step 3 — Watch that RESTARTS stays 0 over time
kubectl get pods -n lab04 -w
# Should be stable — no restarts, all 1/1

# Step 4 — Simulate a readiness failure (without killing the pod)
# Exec into pod and block the HTTP port temporarily
POD=$(kubectl get pod -n lab04 -l app=prod-app -o jsonpath='{.items[0].metadata.name}')

# Watch endpoints in one terminal
kubectl get endpoints -n lab04 -w &

# In another, confirm readiness probe is checking /
kubectl describe pod ${POD} -n lab04 | grep -A 5 "Readiness"
```

### The Decision Guide — Which Probe for What

```
Should I use a liveness probe?
  YES if: your app can deadlock or hang without crashing
          (stuck threads, memory leak causing unresponsiveness)
  NO if:  your app exits cleanly when it fails
          (K8s restarts it automatically on exit)

Should I use a readiness probe?
  YES if: your app has a startup phase before it can serve traffic
  YES if: your app can become temporarily unavailable (cache warming, dep down)
  YES if: you have rolling deployments (always — required for zero downtime)
  NO if:  you're okay with traffic going to starting pods

Should I use a startup probe?
  YES if: startup time > 20 seconds
  YES if: startup time is variable (sometimes fast, sometimes slow)
  YES if: you need liveness probe but app takes time to start
  NO if:  app starts in under 10 seconds consistently

Probe endpoint best practices:
  Liveness endpoint:  checks app is not deadlocked (lightweight, no DB calls)
  Readiness endpoint: checks app AND dependencies are ready (DB connected, cache warm)
  Use DIFFERENT endpoints for liveness and readiness when possible
```

### What Bad Probe Config Costs You in Prod

```
Liveness too aggressive:
  → Pods restart during traffic spikes (high latency = probe timeout = kill)
  → Rolling restarts cascade under load — incident gets worse
  → Data loss if app has in-flight requests

Readiness too strict:
  → Pods flicker in/out of Endpoints constantly
  → Load balancer routing becomes unstable
  → Clients see intermittent failures

Missing readiness on rolling deployment:
  → Traffic sent to pods still starting up
  → 502/503 errors during every deployment
  → Zero-downtime deployment becomes 30-second-downtime deployment

Missing startup probe on slow app:
  → App killed before it finishes starting
  → CrashLoopBackOff — looks like an app bug
  → Backoff delay means app never successfully starts
```

---

## Key Commands Reference — Lab 04

```bash
# Check probe configuration on a pod
kubectl describe pod <name> -n <namespace>
# Look for: Liveness, Readiness, Startup sections

# Watch restarts in real time (probe issues show as climbing RESTARTS)
kubectl get pods -n <namespace> -w

# Check if pod is in/out of Service rotation (readiness)
kubectl get endpoints -n <namespace>

# Get probe config as JSON
kubectl get pod <name> -n <namespace> \
  -o jsonpath='{.spec.containers[0].livenessProbe}'

# Check probe failure events
kubectl describe pod <name> -n <namespace> | grep -A 3 "Unhealthy"

# Exec into pod to manually test health endpoint
kubectl exec -it <name> -n <namespace> -- curl -s http://localhost:<port>/healthz

# Check previous container (killed by liveness) logs
kubectl logs <name> -n <namespace> --previous
```

---

## Prod Wisdom — The Senior Engineer Mindset

Three things that define senior probe configuration:

**1. They use different endpoints for liveness and readiness.** Liveness checks if the app is alive (fast, no external calls). Readiness checks if the app is ready to serve (includes dependency checks). Same endpoint for both is a common mistake — when the DB is down, readiness should fail (remove from rotation) but liveness should pass (don't restart, that won't fix the DB).

**2. They set liveness thresholds conservatively.** `failureThreshold: 3`, `periodSeconds: 15`, `timeoutSeconds: 5`. A probe that kills pods under momentary load is worse than no probe. Liveness is a last resort — it should only fire for genuinely broken containers.

**3. They always have a readiness probe on every Deployment.** No exceptions. Without it, rolling deployments send traffic to starting pods. This is the single most impactful probe to get right for zero-downtime deployments — more impactful than liveness.

The probe configuration order of importance:
```
1. Readiness probe — always, on every Deployment
2. Startup probe   — any app with startup > 20s
3. Liveness probe  — only for apps that can deadlock without exiting
```

---

## Cleanup

```bash
kubectl delete namespace lab04
```

---

*Lab 04 complete. Move to Lab 05 — Resource Limits & Scheduling when ready.*
