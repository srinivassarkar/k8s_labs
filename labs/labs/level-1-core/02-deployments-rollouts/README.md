# Lab 02 — Deployments & Rollouts

## What This Lab Is About

In production, you rarely manage pods directly. You manage **Deployments**. A Deployment is the real unit of work — it owns the ReplicaSet, which owns the pods. Understanding how Deployments behave during updates, failures, and rollbacks is not optional — it is the difference between a clean release and a 3am incident.

This lab covers the 5 most critical Deployment scenarios: a bad image rollout that takes down your app, rollback mechanics, a broken rollout that stalls silently, a misconfigured strategy that causes downtime, and scaling under pressure. Every scenario is something that has happened in a real prod cluster.

> The Deployment is not just a pod manager. It is your release mechanism, your rollback button, and your availability contract — all in one YAML file.

---

## Environment

- **Cluster:** KIND (Kubernetes IN Docker)
- **K8s Version:** v1.27.3
- **Node:** Single node (`kind-control-plane`)
- **Namespace for this lab:** `lab02`

```bash
kubectl create namespace lab02
kubectl config set-context --current --namespace=lab02
```

---

## The 5 Scenarios

| # | Scenario | What You'll Learn |
|---|---|---|
| 01 | Bad image rollout → rollback | `rollout undo`, history, revision tracking |
| 02 | Stalled rollout — silent failure | `rollout status`, stuck `Pending`, `maxUnavailable` |
| 03 | Rollout strategy causing downtime | `RollingUpdate` vs `Recreate`, `maxSurge` / `maxUnavailable` |
| 04 | Deployment scaling — wrong way and right way | `scale`, HPA conflict awareness |
| 05 | Broken readiness blocks rollout forever | How a bad probe freezes the entire rollout |

---

## Scenario 01 — Bad Image Rollout → Rollback

### What You'll Break

You have a healthy Deployment running `nginx:1.25`. You push an update with a bad image tag. The rollout starts, new pods fail to start, and traffic is disrupted. You need to roll back to the last good revision.

### Apply the Base (Healthy) Deployment

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: lab02
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web
        image: nginx:1.25
        ports:
        - containerPort: 80
EOF
```

Wait for it to be healthy:
```bash
kubectl rollout status deployment/web-app -n lab02
# "deployment "web-app" successfully rolled out"

kubectl get pods -n lab02
# All 3 pods Running
```

### Break It — Push a Bad Image

```bash
kubectl set image deployment/web-app web=nginx:BAD-TAG-DOES-NOT-EXIST -n lab02
```

### Symptoms You Will Observe

```bash
kubectl get pods -n lab02
```

```
NAME                       READY   STATUS             RESTARTS   AGE
web-app-7d9f8b6c4-abc12    1/1     Running            0          2m
web-app-7d9f8b6c4-def34    1/1     Running            0          2m
web-app-7d9f8b6c4-ghi56    1/1     Running            0          2m
web-app-9b4c7d2f1-xyz99    0/1     ImagePullBackOff   0          20s
```

K8s starts a new pod with the bad image. Because the default strategy is `RollingUpdate` with `maxUnavailable: 25%`, it will NOT kill the old pods until the new one is ready. So your app is still serving — but the rollout is stuck.

```bash
kubectl rollout status deployment/web-app -n lab02
# Waiting for deployment "web-app" rollout to finish: 1 out of 3 new replicas have been updated...
# (hangs here — it will never finish)
```

### Investigate

```bash
# Step 1 — See the rollout is stuck
kubectl rollout status deployment/web-app -n lab02

# Step 2 — Check pod states
kubectl get pods -n lab02

# Step 3 — Check rollout history
kubectl rollout history deployment/web-app -n lab02
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         <none>

# Step 4 — Inspect the bad revision
kubectl rollout history deployment/web-app -n lab02 --revision=2

# Step 5 — See the ReplicaSets (old and new)
kubectl get replicasets -n lab02
# You'll see the old RS with 3 pods and new RS with 1 pod (stuck)
```

### Fix — Roll Back

```bash
# Roll back to previous revision
kubectl rollout undo deployment/web-app -n lab02

# Confirm rollback completed
kubectl rollout status deployment/web-app -n lab02
# "deployment "web-app" successfully rolled out"

# Verify all pods are Running with old image
kubectl get pods -n lab02
kubectl describe deployment web-app -n lab02 | grep Image
# Image: nginx:1.25
```

You can also rollback to a specific revision:
```bash
kubectl rollout undo deployment/web-app --to-revision=1 -n lab02
```

### Add Change-Cause for Better History (Prod Practice)

```bash
# Always annotate your rollouts in prod — this is what makes history useful
kubectl annotate deployment/web-app kubernetes.io/change-cause="nginx upgrade to 1.25" -n lab02

kubectl rollout history deployment/web-app -n lab02
# REVISION  CHANGE-CAUSE
# 1         nginx upgrade to 1.25
```

### Prod Wisdom

`rollout undo` is your emergency brake. But in real prod, you should **always set `kubernetes.io/change-cause` annotation** on every deployment so rollout history is human-readable. Seeing `revision 3: <none>` at 2am is useless. Seeing `revision 3: upgrade postgres driver to 4.2.1` tells you exactly what to undo and why.

---

## Scenario 02 — Stalled Rollout — Silent Failure

### What You'll Break

A Deployment update where the new pods get stuck in `Pending` because of a resource request that can't be satisfied. The rollout hangs silently — `rollout status` just waits forever. No alert, no error on the Deployment itself — just silence. This is the most dangerous type of failure.

### Apply the Base Deployment First

```bash
# Clean up from previous scenario first
kubectl delete deployment web-app -n lab02

cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  namespace: lab02
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
    spec:
      containers:
      - name: api
        image: nginx:1.25
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
EOF
```

Wait for healthy state:
```bash
kubectl rollout status deployment/api-server -n lab02
```

### Break It — Update With Impossible Resource Request

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  namespace: lab02
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
    spec:
      containers:
      - name: api
        image: nginx:1.26
        resources:
          requests:
            memory: "500Gi"
            cpu: "100"
          limits:
            memory: "500Gi"
            cpu: "100"
EOF
```

### Symptoms You Will Observe

```bash
kubectl rollout status deployment/api-server -n lab02
# Waiting for deployment "api-server" rollout to finish: 1 out of 2 new replicas have been updated...
# (hangs forever)
```

```bash
kubectl get pods -n lab02
```

```
NAME                          READY   STATUS    RESTARTS   AGE
api-server-5d8f9c7b2-aaa11    1/1     Running   0          3m
api-server-5d8f9c7b2-bbb22    1/1     Running   0          3m
api-server-9f3c2b1a4-ccc33    0/1     Pending   0          45s
```

The old pods are healthy, the new pod is `Pending`. The rollout is stuck but the app is still serving. **This is the most dangerous state in prod** — everything looks okay from a service perspective, but your deployment pipeline is blocked and no one notices until someone tries to deploy something else.

### Investigate

```bash
# Step 1 — Rollout is hanging
kubectl rollout status deployment/api-server -n lab02

# Step 2 — Find the Pending pod
kubectl get pods -n lab02

# Step 3 — Describe the Pending pod
kubectl describe pod <pending-pod-name> -n lab02
# Events: 0/1 nodes are available: 1 Insufficient memory, 1 Insufficient cpu.

# Step 4 — Check the deployment's rollout progress condition
kubectl describe deployment api-server -n lab02
# Look for Conditions section:
# Progressing: True (ReplicaSet is progressing)
# Available: True (old pods still serving)
# This tells you: alive but stuck

# Step 5 — Check ReplicaSets
kubectl get replicasets -n lab02
# Old RS: 2 ready, New RS: 0 ready (stuck at 1 desired)
```

### Fix

```bash
# Roll back to fix immediately
kubectl rollout undo deployment/api-server -n lab02

# Confirm
kubectl rollout status deployment/api-server -n lab02
kubectl get pods -n lab02
```

### Prod Wisdom

Silent stalled rollouts are why production teams set **`progressDeadlineSeconds`** on every Deployment. When the deadline expires, the Deployment condition flips to `Progressing: False` and your monitoring alerts fire. Without it — silence. Always add this to your Deployment spec:

```yaml
spec:
  progressDeadlineSeconds: 300  # 5 minutes — alert if rollout doesn't complete
```

---

## Scenario 03 — Wrong Rollout Strategy Causing Downtime

### What You'll Break

A Deployment using `Recreate` strategy — it terminates ALL existing pods first, then starts new ones. During the gap between termination and startup, your service has zero pods. This is a full outage window. Many engineers accidentally use `Recreate` or misconfigure `RollingUpdate` parameters without realizing the impact.

### Apply the Broken Strategy

```bash
kubectl delete deployment api-server -n lab02

cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: lab02
spec:
  replicas: 3
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: web
        image: nginx:1.25
EOF
```

Wait for healthy:
```bash
kubectl rollout status deployment/frontend -n lab02
```

### Observe the Downtime Window

Run this in a **separate terminal** to watch in real time:
```bash
kubectl get pods -n lab02 -w
```

Now trigger the update:
```bash
kubectl set image deployment/frontend web=nginx:1.26 -n lab02
```

In the watch terminal you will see:
```
frontend-abc123   1/1   Running   →   Terminating
frontend-def456   1/1   Running   →   Terminating
frontend-ghi789   1/1   Running   →   Terminating
# GAP HERE — zero pods running — full downtime
frontend-xyz111   0/1   Pending   →   ContainerCreating   →   Running
frontend-xyz222   0/1   Pending   →   ContainerCreating   →   Running
frontend-xyz333   0/1   Pending   →   ContainerCreating   →   Running
```

That gap is your outage window.

### Contrast — RollingUpdate With Safe Parameters

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: lab02
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Allow 1 extra pod above replicas during update
      maxUnavailable: 0  # NEVER take a pod down until a new one is fully Ready
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: web
        image: nginx:1.27
EOF
```

Watch the difference:
```bash
# In separate terminal
kubectl get pods -n lab02 -w
```

Now you will see: new pod comes up FIRST, becomes Ready, then old pod terminates. Zero gap. Zero downtime.

### Understanding maxSurge and maxUnavailable

```
replicas: 3, maxSurge: 1, maxUnavailable: 0

During update:
- Max pods running at any time: 3 + 1 = 4
- Min pods running at any time: 3 - 0 = 3
- Zero downtime guaranteed

replicas: 3, maxSurge: 0, maxUnavailable: 1 (K8s default)

During update:
- Max pods running at any time: 3 + 0 = 3
- Min pods running at any time: 3 - 1 = 2
- One pod is always down during the rollout — acceptable for non-critical apps
```

### When to Use Recreate

`Recreate` is NOT always wrong. It's correct when:
- Your app cannot run two versions simultaneously (database schema migrations)
- You have a singleton that breaks if two instances run at once
- You explicitly accept a brief downtime window for the update

The mistake is using it accidentally — not understanding what it does.

### Prod Wisdom

For any user-facing service in prod, the correct default is `RollingUpdate` with `maxUnavailable: 0` and `maxSurge: 1`. This costs one extra pod worth of resources during deployment but guarantees zero downtime. `maxUnavailable: 0` is the single most important setting for high-availability deployments.

---

## Scenario 04 — Scaling — Wrong Way and Right Way

### What You'll Break

Manually scaling a Deployment in a way that fights against an HPA, and also understanding what happens when you scale to zero. These are two common prod mistakes.

### Setup

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: worker
  namespace: lab02
spec:
  replicas: 3
  selector:
    matchLabels:
      app: worker
  template:
    metadata:
      labels:
        app: worker
    spec:
      containers:
      - name: worker
        image: nginx:1.25
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
EOF
```

### Break It — Scale to Zero Accidentally

```bash
# This is a common fat-finger mistake in prod
kubectl scale deployment worker --replicas=0 -n lab02
```

```bash
kubectl get pods -n lab02
# No resources found in lab02 namespace.

kubectl get deployment worker -n lab02
# NAME     READY   UP-TO-DATE   AVAILABLE   DESIRED
# worker   0/0     0            0           0
```

Your app is completely down. No pods. No error. Just gone.

```bash
# Recover
kubectl scale deployment worker --replicas=3 -n lab02
kubectl rollout status deployment/worker -n lab02
```

### Break It — HPA Conflict (Understanding the Pattern)

When an HPA is managing a Deployment, manually scaling it with `kubectl scale` creates a conflict — the HPA will override your manual change on its next reconciliation loop (every 15 seconds by default).

```bash
# Create an HPA targeting the worker deployment
cat <<EOF | kubectl apply -f -
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: worker-hpa
  namespace: lab02
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: worker
  minReplicas: 2
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
EOF
```

Now try to manually override it:
```bash
kubectl scale deployment worker --replicas=10 -n lab02

kubectl get deployment worker -n lab02
# READY shows 10 briefly...

# Wait 15-30 seconds
kubectl get deployment worker -n lab02
# HPA brought it back to minReplicas (2) — your manual scale was overridden silently
```

### Investigate the Conflict

```bash
# Check what HPA is doing
kubectl get hpa worker-hpa -n lab02
# NAME        REFERENCE          TARGETS         MINPODS   MAXPODS   REPLICAS
# worker-hpa  Deployment/worker  <unknown>/50%   2         5         2

# Describe HPA for event history
kubectl describe hpa worker-hpa -n lab02

# Check deployment events — you'll see scale-up then scale-down
kubectl describe deployment worker -n lab02
```

### The Right Way to Scale When HPA is Active

```bash
# Don't use kubectl scale — update the HPA bounds instead
kubectl patch hpa worker-hpa -n lab02 --patch '{"spec":{"minReplicas":4}}'

# Or update the HPA YAML directly
kubectl edit hpa worker-hpa -n lab02
```

### Cleanup HPA

```bash
kubectl delete hpa worker-hpa -n lab02
```

### Prod Wisdom

**Never use `kubectl scale` on a Deployment that has an HPA.** The HPA owns the replica count — your manual change will be silently reverted within seconds. This has caused many "why does my deployment keep going back to 2 replicas?" incidents in prod. The rule is: if HPA exists, change `minReplicas`/`maxReplicas` on the HPA. If no HPA, use `kubectl scale` or update the `replicas` field in the manifest.

---

## Scenario 05 — Broken Readiness Probe Blocks Rollout Forever

### What You'll Break

A Deployment update where the new pods have a misconfigured readiness probe — they never become `Ready`. Since K8s won't terminate old pods until new ones are `Ready`, the rollout freezes. Old pods keep serving, new pods are running but never ready, and your CI/CD pipeline is permanently blocked.

### Apply the Base Deployment

```bash
kubectl delete deployment worker -n lab02

cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-service
  namespace: lab02
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-service
  template:
    metadata:
      labels:
        app: web-service
    spec:
      containers:
      - name: web
        image: nginx:1.25
        readinessProbe:
          httpGet:
            path: /healthz
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
EOF
```

Wait for healthy:
```bash
kubectl rollout status deployment/web-service -n lab02
# Note: nginx:1.25 doesn't have /healthz but the probe will still pass
# because nginx returns 404 which is not a connection failure
# Let's check
kubectl get pods -n lab02
```

### Break It — Update With Bad Readiness Probe Path

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-service
  namespace: lab02
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: web-service
  template:
    metadata:
      labels:
        app: web-service
    spec:
      containers:
      - name: web
        image: nginx:1.26
        readinessProbe:
          httpGet:
            path: /this-endpoint-will-never-exist-ever
            port: 9999
          initialDelaySeconds: 3
          periodSeconds: 3
          failureThreshold: 3
EOF
```

### Symptoms You Will Observe

```bash
kubectl get pods -n lab02
```

```
NAME                           READY   STATUS    RESTARTS   AGE
web-service-old-aaa11          1/1     Running   0          3m
web-service-old-bbb22          1/1     Running   0          3m
web-service-new-ccc33          0/1     Running   0          45s
```

New pod is `Running` but `0/1 READY`. It will stay this way forever. Rollout is frozen.

```bash
kubectl rollout status deployment/web-service -n lab02
# Waiting for deployment "web-service" rollout to finish: 1 out of 2 new replicas have been updated...
# (hangs)
```

### Investigate

```bash
# Step 1 — Rollout is frozen
kubectl rollout status deployment/web-service -n lab02

# Step 2 — Spot the 0/1 Ready pod
kubectl get pods -n lab02

# Step 3 — Describe the new pod — readiness probe failing
kubectl describe pod <new-pod-name> -n lab02

# In Events look for:
# Warning  Unhealthy  kubelet  Readiness probe failed:
# Get "http://10.244.x.x:9999/this-endpoint-will-never-exist-ever": dial tcp: connection refused

# Step 4 — Check readiness probe config on deployment
kubectl get deployment web-service -n lab02 -o jsonpath='{.spec.template.spec.containers[0].readinessProbe}'

# Step 5 — Check deployment conditions
kubectl describe deployment web-service -n lab02
# Conditions:
#   Available: True   (old pods still serving)
#   Progressing: True (but not making progress)
```

### Fix

```bash
# Roll back immediately
kubectl rollout undo deployment/web-service -n lab02

kubectl rollout status deployment/web-service -n lab02
kubectl get pods -n lab02
# Both pods back to 1/1 Running
```

Then fix the probe in the deployment:
```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-service
  namespace: lab02
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: web-service
  template:
    metadata:
      labels:
        app: web-service
    spec:
      containers:
      - name: web
        image: nginx:1.26
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
          failureThreshold: 3
EOF
```

```bash
kubectl rollout status deployment/web-service -n lab02
# "deployment "web-service" successfully rolled out"
```

### Prod Wisdom

A readiness probe that never passes is a **deployment blocker, not a pod killer**. The pod runs forever, consuming resources, never serving traffic, and permanently blocking your pipeline. This is subtle because the pod shows `STATUS: Running` — only `READY: 0/1` betrays the problem. Always validate your readiness probe endpoint independently before deploying. And always set `progressDeadlineSeconds` on your Deployment — without it, a frozen rollout will block your pipeline silently for as long as the Deployment exists.

---

## Key Commands Reference — Lab 02

```bash
# Deployment status and history
kubectl rollout status deployment/<name> -n <namespace>
kubectl rollout history deployment/<name> -n <namespace>
kubectl rollout history deployment/<name> -n <namespace> --revision=<N>

# Rollback
kubectl rollout undo deployment/<name> -n <namespace>
kubectl rollout undo deployment/<name> -n <namespace> --to-revision=<N>

# Update image
kubectl set image deployment/<name> <container>=<image>:<tag> -n <namespace>

# Scale
kubectl scale deployment/<name> --replicas=<N> -n <namespace>

# Pause and resume a rollout (useful to stage a release)
kubectl rollout pause deployment/<name> -n <namespace>
kubectl rollout resume deployment/<name> -n <namespace>

# Watch pods in real time
kubectl get pods -n <namespace> -w

# Inspect ReplicaSets (shows old and new RS during rollout)
kubectl get replicasets -n <namespace>

# Check deployment conditions
kubectl describe deployment <name> -n <namespace>

# Annotate for history
kubectl annotate deployment/<name> kubernetes.io/change-cause="<message>" -n <namespace>
```

---

## Prod Wisdom — The Senior Engineer Mindset

Three things that separate senior engineers on Deployment handling:

**1. They always set `progressDeadlineSeconds`.** Without it, a stalled rollout is invisible to monitoring. With it, your alerting fires automatically when a deployment doesn't complete in time.

**2. They use `maxUnavailable: 0` for user-facing services.** The default (`maxUnavailable: 25%`) means K8s will kill pods before new ones are ready. For anything serving real traffic, that's acceptable only if you've consciously decided it is.

**3. They annotate every deployment with a change-cause.** Rollout history with `<none>` is useless in an incident. History with human-readable messages is your audit trail and your rollback decision guide.

The complete rollout safety spec for any prod Deployment:

```yaml
spec:
  progressDeadlineSeconds: 300
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

These four lines should be in every Deployment you write from now on. Not optional — default.

---

## Cleanup

```bash
kubectl delete namespace lab02
```

---

*Lab 02 complete. Move to Lab 03 — Services & DNS when ready.*