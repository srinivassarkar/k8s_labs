# Kubernetes Deployments — Rolling Updates, Rollbacks & Lifecycle Management
### The Complete Mental Model — Interviews · CKA · Production

---

## 0. First Principles — The Mental Model That Never Changes

Kubernetes Deployments are built on a three-layer ownership chain:

```
Deployment  →  owns and manages  →  ReplicaSets
ReplicaSet  →  owns and manages  →  Pods
```

Every rule about Deployments flows from this hierarchy. Lock in these truths:

```
1. Deployment = desired state declaration for a versioned workload
2. ReplicaSet = the mechanism that keeps N pods running at all times
3. Pod = the actual running unit — ephemeral, replaceable

Every image change     →  new ReplicaSet created (old one kept, scaled to 0)
Rollback               →  old ReplicaSet scaled back up (not re-created)
Scale                  →  current ReplicaSet replica count adjusted
Pause/Resume           →  gates whether changes trigger a new RS immediately
```

The Deployment controller never touches Pods directly. It manages ReplicaSets. ReplicaSets manage Pods. This indirection is what enables zero-downtime rollouts and instant rollbacks.

**The invariant:**
```
A Deployment's revision history = its collection of ReplicaSets
Rolling back = switching which ReplicaSet is "active" (replicas > 0)
```

---

## 1. Reality Constraints — What Deployments Actually Do and Don't Do

### What Deployments Do

- Maintain a desired number of running pod replicas at all times
- Execute rolling updates by creating a new ReplicaSet and gradually shifting traffic
- Retain old ReplicaSets as rollback checkpoints (controlled by `revisionHistoryLimit`)
- Pause and batch multiple changes before triggering a rollout
- Expose rollout status, history, and revision tracking

### What Deployments Do NOT Do

| Misconception | Reality |
|---|---|
| "Rollback recreates the old pods from scratch" | No — it rescales the existing old ReplicaSet back up |
| "Scaling changes the Deployment revision" | No — scaling does not create a new ReplicaSet |
| "You can scale a ReplicaSet directly" | Technically yes, but Deployment will override it immediately |
| "Pause stops running pods" | No — pause gates future rollouts; existing pods keep running |
| "kubectl rollout undo restores the annotation" | No — change-cause annotation must be set manually after undo |
| "Every kubectl apply creates a new revision" | No — only changes to pod template spec create a new revision |
| "Deployment tracks all changes" | No — only `.spec.template` changes increment the revision |

### What Actually Increments the Revision Counter

```
Creates new revision (new ReplicaSet):
  ✅ Image change
  ✅ Environment variable change
  ✅ Resource limits change
  ✅ Any change to spec.template.*

Does NOT create new revision:
  ❌ Replica count change (scale)
  ❌ Label change on the Deployment metadata
  ❌ Annotation change
  ❌ Pause/Resume
```

### The `revisionHistoryLimit` Reality

```yaml
spec:
  revisionHistoryLimit: 10  # default — keeps 10 old ReplicaSets
```

Old ReplicaSets (scaled to 0) are kept as rollback checkpoints. Set too low and you lose rollback options. Set to 0 and rollback is impossible. In production, 5–10 is standard.

---

## 2. Decision Logic — When to Use What

### Deployment vs Other Workload Resources

| Resource | Use When |
|---|---|
| `Deployment` | Stateless workloads — web servers, APIs, workers |
| `StatefulSet` | Stateful workloads — databases, Kafka, ordered identity required |
| `DaemonSet` | One pod per node — log agents, monitoring, CNI plugins |
| `Job` | Run-to-completion tasks — batch jobs, migrations |
| `CronJob` | Scheduled run-to-completion tasks |

### Update Strategy Decision

```
How much downtime is acceptable?
        │
        ├── Zero downtime, gradual rollout → RollingUpdate (default)
        │         └── How fast?
        │               ├── Fast but risky → maxSurge high, maxUnavailable high
        │               └── Slow and safe → maxSurge low, maxUnavailable 0
        │
        └── Full replacement, brief downtime → Recreate
              └── Use when: old and new versions cannot run simultaneously
                  (e.g., DB schema migration, exclusive port binding)
```

### When to Use `kubectl rollout pause`

```
Making a single change?   →  No pause needed, rollout is atomic
Making multiple changes?  →  Pause → make all changes → Resume
                              Prevents an intermediate rollout
Testing before commit?    →  Pause, update, verify, then Resume
```

### Rollback Decision

```
Something broke after rollout?
        │
        ├── Broke immediately → kubectl rollout undo (go to previous revision)
        │
        ├── Need to go to a specific known-good version
        │         → kubectl rollout undo --to-revision=<N>
        │         → Check revision numbers first: kubectl rollout history deployment/<n>
        │
        └── Need to know what changed in each revision
                  → kubectl rollout history deployment/<n> --revision=<N>
```

---

## 3. Internal Working — How It Actually Happens

### The Deployment Controller Loop

The Deployment controller runs inside `kube-controller-manager` and continuously reconciles:

```
1. Watch API Server for Deployment objects
2. For each Deployment:
   a. Find all ReplicaSets owned by this Deployment (via ownerReferences)
   b. Identify the "active" RS (the one matching current pod template hash)
   c. Compare desired state (Deployment spec) vs actual state (RS + Pod counts)
   d. Take action to close the gap
```

### Rolling Update — Step by Step Internals

```
Initial state: Deployment with 3 replicas, image nginx:1.19
               RS-v1: replicas=3, pods=[pod1, pod2, pod3]

User runs: kubectl set image deployment/nginx nginx=nginx:1.20

1. Deployment controller computes new pod template hash
2. Creates RS-v2 (nginx:1.20) with replicas=0
3. Rolling update loop begins (respecting maxSurge and maxUnavailable):

   Iteration 1:
     RS-v2 scaled to 1 (total pods: 4 = surge of 1)
     Wait for RS-v2 pod to be Ready
     RS-v1 scaled to 2 (one old pod terminated)

   Iteration 2:
     RS-v2 scaled to 2
     Wait for new pod Ready
     RS-v1 scaled to 1

   Iteration 3:
     RS-v2 scaled to 3
     Wait for new pod Ready
     RS-v1 scaled to 0

4. Final state:
   RS-v1: replicas=0, image=nginx:1.19 (kept for rollback)
   RS-v2: replicas=3, image=nginx:1.20 (active)
```

### maxSurge and maxUnavailable — The Two Knobs

```
maxUnavailable: how many pods can be unavailable during rollout
maxSurge:       how many pods ABOVE desired can exist during rollout

Example with replicas=3, maxSurge=1, maxUnavailable=1:
  Minimum pods running at any time: 3 - 1 = 2
  Maximum pods at any time:         3 + 1 = 4

Example with replicas=3, maxSurge=0, maxUnavailable=1:
  Kills 1 old pod first, then creates 1 new pod
  Never exceeds 3 total pods — useful when node capacity is tight

Example with replicas=3, maxSurge=3, maxUnavailable=0:
  Creates all 3 new pods first, then kills all 3 old pods
  6 pods briefly — fastest rollout, needs spare capacity
```

### Rollback — What Actually Happens

```
kubectl rollout undo deployment/nginx

1. Controller identifies the previous revision (RS-v1, scaled to 0)
2. Scales RS-v1 back up using the same rolling update mechanism
3. Scales RS-v2 down to 0 simultaneously
4. The revision numbers SHIFT — the old revision becomes the new latest

IMPORTANT: After undo, revision numbers advance, they don't rewind.
Before undo: REVISION 1 (nginx:1.19), REVISION 2 (nginx:1.20) ← current
After undo:  REVISION 1 (nginx:1.19), REVISION 2 (nginx:1.20), REVISION 3 (nginx:1.19)
             Revision 3 is now current — it's the same image as Rev 1
```

### The Pod Template Hash — How ReplicaSets Are Identified

```
Each ReplicaSet gets a label: pod-template-hash=<hash>
Hash is computed from the pod template spec
Same template = same hash = same ReplicaSet
Different image = different hash = new ReplicaSet

kubectl get rs --show-labels
# NAME                        DESIRED   CURRENT   READY   LABELS
# nginx-deployment-75675f5897  3         3         3      pod-template-hash=75675f5897,app=nginx
# nginx-deployment-d7f7c5f4c   0         0         0      pod-template-hash=d7f7c5f4c,app=nginx
```

### How the Deployment Selector Works

```yaml
spec:
  selector:
    matchLabels:
      app: nginx        # ← immutable after creation
  template:
    metadata:
      labels:
        app: nginx      # ← must match selector
```

The selector is **immutable**. Changing it requires deleting and recreating the Deployment. The ReplicaSet selector inherits from the Deployment and adds `pod-template-hash`.

---

## 4. Hands-On — Production-Quality YAML + Commands

### Full Production Deployment Manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: production
  labels:
    app: nginx
    version: "1.19"
    team: platform
  annotations:
    kubernetes.io/change-cause: "Initial nginx:1.19 deployment"
spec:
  replicas: 3
  revisionHistoryLimit: 5          # keep 5 old RS for rollback
  progressDeadlineSeconds: 600     # fail if rollout takes > 10 min
  selector:
    matchLabels:
      app: nginx                   # immutable — cannot change after creation
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1                  # 1 extra pod above desired during rollout
      maxUnavailable: 0            # never reduce below desired — zero downtime
  template:
    metadata:
      labels:
        app: nginx
        version: "1.19"
    spec:
      containers:
      - name: nginx
        image: nginx:1.19
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 250m
            memory: 256Mi
        readinessProbe:            # critical — rollout waits for Ready before proceeding
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 15
          periodSeconds: 20
      terminationGracePeriodSeconds: 30
```

### Recreate Strategy (for Non-Compatible Version Changes)

```yaml
spec:
  strategy:
    type: Recreate    # kills ALL old pods before starting new ones
                      # brief downtime — use only when required
```

### Core Deployment Commands

```bash
# Create from manifest
kubectl apply -f nginx-deploy.yaml

# Imperative create (CKA quick-start, then export to YAML)
kubectl create deployment nginx-deployment \
  --image=nginx:1.19 \
  --replicas=3 \
  --dry-run=client -o yaml > nginx-deploy.yaml

# Verify state
kubectl get deployment nginx-deployment
kubectl get rs                              # watch ReplicaSets
kubectl get pods -o wide                   # see which node each pod is on
kubectl describe deployment nginx-deployment

# Watch rollout live
kubectl rollout status deployment/nginx-deployment
```

### Image Updates

```bash
# Update image (triggers new RS + rolling update)
kubectl set image deployment/nginx-deployment nginx=nginx:1.20

# Always annotate after update — this populates CHANGE-CAUSE in history
kubectl annotate deployment nginx-deployment \
  kubernetes.io/change-cause="Upgrade nginx 1.19 → 1.20" \
  --overwrite

# Update via edit (live YAML edit)
kubectl edit deployment nginx-deployment

# Update via apply (preferred — declarative)
# Edit the YAML file first, then:
kubectl apply -f nginx-deploy.yaml
```

### Rollout History and Inspection

```bash
# View history (requires change-cause annotations to be meaningful)
kubectl rollout history deployment/nginx-deployment
# REVISION  CHANGE-CAUSE
# 1         Initial nginx:1.19 deployment
# 2         Upgrade nginx 1.19 → 1.20
# 3         Upgrade nginx 1.20 → 1.21

# Inspect a specific revision
kubectl rollout history deployment/nginx-deployment --revision=2

# Check current rollout status
kubectl rollout status deployment/nginx-deployment
# Waiting for deployment "nginx-deployment" rollout to finish: 1 out of 3 new replicas have been updated...
# deployment "nginx-deployment" successfully rolled out
```

### Rollback Commands

```bash
# Rollback to immediately previous revision
kubectl rollout undo deployment/nginx-deployment

# Rollback to a specific revision
kubectl rollout undo deployment/nginx-deployment --to-revision=1

# Always annotate after rollback — undo does not restore the annotation
kubectl annotate deployment nginx-deployment \
  kubernetes.io/change-cause="Rollback to nginx:1.19 — production issue" \
  --overwrite

# Verify rollback succeeded
kubectl describe deployment nginx-deployment | grep Image
kubectl rollout history deployment/nginx-deployment
```

### Scale Operations

```bash
# Scale up/down
kubectl scale deployment nginx-deployment --replicas=5

# Autoscale (creates HPA)
kubectl autoscale deployment nginx-deployment \
  --min=2 --max=10 --cpu-percent=70

# Check HPA
kubectl get hpa
```

### Pause and Resume (Batching Multiple Changes)

```bash
# Pause — future changes won't trigger rollout yet
kubectl rollout pause deployment/nginx-deployment

# Make multiple changes safely
kubectl set image deployment/nginx-deployment nginx=nginx:1.22
kubectl set resources deployment/nginx-deployment \
  --containers=nginx \
  --requests=cpu=200m,memory=256Mi \
  --limits=cpu=500m,memory=512Mi

# Resume — triggers single rollout with ALL batched changes
kubectl rollout resume deployment/nginx-deployment

# Watch it roll out
kubectl rollout status deployment/nginx-deployment
```

### Export and Inspect

```bash
# Export current live state (useful for understanding defaults)
kubectl get deployment nginx-deployment -o yaml

# Get specific fields
kubectl get deployment nginx-deployment \
  -o jsonpath='{.spec.template.spec.containers[0].image}'

# Get all RS owned by a deployment
kubectl get rs -l app=nginx

# Get pods with their RS labels
kubectl get pods --show-labels -l app=nginx
```

### Cleanup

```bash
kubectl delete deployment nginx-deployment

# Delete with YAML (safer — deletes exactly what was applied)
kubectl delete -f nginx-deploy.yaml
```

---

## 5. Production Flow — Real-World Architecture and Design Patterns

### The GitOps Deployment Lifecycle

```
1. Developer pushes code → Git
2. CI pipeline builds image → tags with git SHA
3. Pipeline updates Deployment YAML image field
4. ArgoCD / Flux detects change → applies to cluster
5. Deployment controller executes rolling update
6. Readiness probes gate each pod's inclusion in the rollout
7. If health check fails → rollout stalls (progressDeadlineSeconds)
8. Alert fires → team runs kubectl rollout undo
```

### Why `readinessProbe` is Non-Negotiable in Production

```
Without readinessProbe:
  Rolling update creates new pod → pod starts → Deployment marks it Ready
  → scales down old pod → new pod crashes 30 seconds later
  → Service sends traffic to a pod that isn't serving → errors

With readinessProbe:
  Rolling update creates new pod → pod starts
  → readinessProbe must pass before pod is marked Ready
  → only then does Deployment scale down one old pod
  → if readinessProbe never passes → rollout stalls → old pods stay up
```

### `progressDeadlineSeconds` — The Safety Net

```yaml
spec:
  progressDeadlineSeconds: 600  # 10 minutes
```

```
If the rollout makes no progress for 600 seconds:
→ Deployment condition: Progressing=False, reason=ProgressDeadlineExceeded
→ The rollout does NOT automatically roll back — you must do it manually
→ Alert on this condition in production

Check stalled rollout:
kubectl get deployment nginx-deployment -o yaml | grep -A5 conditions
```

### The Blue-Green Pattern Using Deployments

```yaml
# Blue (current production)
metadata:
  name: nginx-blue
  labels:
    slot: blue

# Green (new version staging)
metadata:
  name: nginx-green
  labels:
    slot: green

# Service points to blue
spec:
  selector:
    app: nginx
    slot: blue   # ← change this to "green" to switch traffic
```

```bash
# Instant cutover (no rolling update — atomic switch)
kubectl patch service nginx-service \
  -p '{"spec":{"selector":{"slot":"green"}}}'
```

### The Canary Pattern Using Deployments

```bash
# Stable: 9 replicas
kubectl scale deployment nginx-stable --replicas=9

# Canary: 1 replica (10% of traffic)
kubectl scale deployment nginx-canary --replicas=1

# Both share the same Service selector label
# Service routes ~10% of requests to canary pod naturally
# When satisfied, scale canary to 10, scale stable to 0
```

### Deployment Topology: Spread Across Zones

```yaml
spec:
  template:
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: nginx
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: nginx
              topologyKey: kubernetes.io/hostname  # spread across nodes too
```

---

## 6. Mistakes — What Actually Breaks in Real Systems

### Mistake 1 — No `readinessProbe`, Bad Version Rolls Out

**What happens:** New image has a startup bug. Rolling update proceeds because Kubernetes marks pods Ready based on container running, not actually serving. Traffic hits broken pods. Errors spike. Rollout completes "successfully" with all broken pods.

**Root cause:** No readinessProbe — Kubernetes has no signal about whether the pod is actually serving.

**Fix:**
```yaml
readinessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 3   # 3 failures = NotReady → rollout stalls
```

### Mistake 2 — Rollout Stalls Silently, No Alert

**What happens:** New version deployed. One pod fails readiness. Rollout stalls at "1 of 3 updated." No alert fires. Engineers don't notice. Old pods serve traffic but the Deployment is stuck in a broken half-updated state. Next deployment attempt fails.

**Root cause:** `progressDeadlineSeconds` triggers but no monitoring on the condition.

**Fix:**
```bash
# Monitor this condition
kubectl get deployment <n> -o jsonpath='{.status.conditions[?(@.type=="Progressing")].reason}'
# Alert when: ProgressDeadlineExceeded

# Manual fix for stalled rollout
kubectl rollout undo deployment/<n>
```

### Mistake 3 — Scaling the ReplicaSet Directly

**What happens:** Engineer runs `kubectl scale rs nginx-deployment-75675f5897 --replicas=5`. It briefly works. Deployment controller notices the RS is out of sync with its desired count and immediately scales it back to 3.

**Root cause:** The Deployment controller is a continuous control loop. Anything that disagrees with the Deployment spec gets corrected.

**Fix:** Always scale the Deployment, never the RS:
```bash
kubectl scale deployment nginx-deployment --replicas=5
```

### Mistake 4 — Forgetting to Annotate After `set image` or `rollout undo`

**What happens:** Six months later, team looks at rollout history. All entries show `<none>` for CHANGE-CAUSE. No one knows what changed at revision 4. Debugging a production issue requires git archaeology.

**Root cause:** `kubectl set image` and `kubectl rollout undo` do not set the annotation automatically.

**Fix:** Make annotation a step in every runbook:
```bash
kubectl set image deployment/nginx nginx=nginx:1.22
kubectl annotate deployment nginx \
  kubernetes.io/change-cause="Deploy nginx:1.22 — CVE-2023-XXXX patch" \
  --overwrite
```

### Mistake 5 — `revisionHistoryLimit: 0` Set for "Cleanliness"

**What happens:** Team sets `revisionHistoryLimit: 0` to clean up old ReplicaSets. Production deployment breaks. Team runs `kubectl rollout undo`. Error: "no rollout history found." No old RS exists. Manual fix required.

**Fix:** Never set to 0. Use 3–10 depending on how many rollback points you want:
```yaml
spec:
  revisionHistoryLimit: 5  # minimum for production
```

### Mistake 6 — Using `Recreate` Strategy on a Stateless App

**What happens:** Team uses `Recreate` because they saw it online. Deployment update kills all 3 pods simultaneously. 30 seconds of downtime. Upstream service returns errors. SLA breach.

**Root cause:** `Recreate` is for workloads where old and new versions cannot co-exist (exclusive DB connections, port conflicts). Stateless HTTP servers should always use `RollingUpdate`.

**Fix:**
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0   # zero downtime guarantee
    maxSurge: 1
```

### Mistake 7 — Changing the Deployment Selector After Creation

**What happens:** Team wants to add a label to the selector. Runs `kubectl apply`. Error: "field is immutable."

**Root cause:** `spec.selector` is immutable in Deployments. The selector defines which pods the Deployment owns — changing it would orphan existing pods.

**Fix:** Delete and recreate the Deployment. Use `--cascade=orphan` if you want to keep running pods during the transition:
```bash
kubectl delete deployment nginx-deployment --cascade=orphan
# Pods keep running — not owned by any Deployment now
kubectl apply -f nginx-deploy-new-selector.yaml
# New Deployment takes ownership of pods matching new selector
```

### Mistake 8 — Pausing a Deployment and Forgetting to Resume

**What happens:** Engineer pauses deployment to make changes. Gets pulled into an incident. Deployment stays paused. Next day, team applies new changes. Nothing rolls out. Pods are running old version. Confusion.

**Root cause:** `kubectl rollout pause` gates all rollouts indefinitely until explicitly resumed.

**Fix:**
```bash
# Check if deployment is paused
kubectl get deployment nginx-deployment -o jsonpath='{.spec.paused}'
# "true" = paused

kubectl rollout resume deployment/nginx-deployment
```

---

## 7. Interview Answers — Compressed, Verbatim-Ready

### Q: What is a Deployment and how does it differ from a ReplicaSet?

> "A Deployment is a higher-level resource that owns and manages ReplicaSets, which in turn own and manage Pods. A ReplicaSet's only job is to ensure a specific number of identical pods are running at all times — it has no concept of update history or rollback. A Deployment adds the ability to perform rolling updates, track revision history, and roll back to a previous state. When you change the pod template in a Deployment — like updating an image — the Deployment controller creates a new ReplicaSet for the new version, gradually shifts replicas to it, and keeps the old ReplicaSet scaled to zero as a rollback checkpoint."

### Q: Walk me through what happens internally when you update a Deployment image.

> "The Deployment controller computes a hash of the new pod template. Since the hash is different, it creates a new ReplicaSet for the new version. Then the rolling update loop begins — it respects maxSurge and maxUnavailable. With maxSurge of 1 and maxUnavailable of 0 on a three-replica deployment, it creates one new pod first, waits for it to pass its readiness probe, then terminates one old pod. It repeats this cycle until the new ReplicaSet has all three replicas and the old one is at zero. The old ReplicaSet is kept in etcd, scaled to zero, as a rollback checkpoint."

### Q: What happens when you run `kubectl rollout undo`?

> "The controller identifies the previous revision — the most recently scaled-down ReplicaSet. It rescales that RS back up using the same rolling update mechanism, and simultaneously scales down the current active RS. The old pods are not recreated from scratch — the existing RS object is reused. Importantly, the revision numbers advance rather than rewind. So if you were on revision 3 and undo, you get revision 4 which has the same spec as revision 2. And the change-cause annotation is not restored automatically — you need to annotate manually after undoing."

### Q: What's the difference between maxSurge and maxUnavailable?

> "maxUnavailable controls how many pods can be unavailable during a rollout — meaning how far below the desired count you're willing to go. maxSurge controls how many extra pods above the desired count can exist at the same time. Setting maxUnavailable to zero means you never have fewer pods than desired — no downtime. Setting maxSurge to zero means you never exceed your desired count — useful when node capacity is tight. The combination defines the speed and safety of your rollout. For zero-downtime, I use maxUnavailable=0, maxSurge=1."

### Q: What happens to a Deployment if the readiness probe fails during a rollout?

> "The rolling update stalls. The Deployment controller will not proceed to scale down old pods or scale up more new pods because the new pod hasn't entered the Ready state. The old pods keep serving traffic. If the probe never passes within `progressDeadlineSeconds` — which defaults to ten minutes — the Deployment gets a condition of ProgressDeadlineExceeded, but it does not automatically roll back. You have to manually run kubectl rollout undo. This is why readiness probes are essential — without them, Kubernetes has no signal that a new pod is actually healthy, and a broken version can roll out completely."

### Q: Why does changing the Deployment selector fail after creation?

> "The selector is immutable because it defines pod ownership. The Deployment finds its pods by matching the selector label. If you changed the selector, the Deployment would no longer be able to find its existing pods — they'd become orphaned and the new selector would match nothing. To prevent this inconsistency, Kubernetes makes the selector immutable. If you genuinely need to change the selector, you have to delete the Deployment and recreate it."

### Q: When would you use `Recreate` strategy instead of `RollingUpdate`?

> "Recreate terminates all existing pods before starting any new ones — there's a window of complete downtime. You use it when old and new versions of your application absolutely cannot run at the same time. The classic example is a database migration where the schema change made by the new version would break the old version, or an application that holds an exclusive lock on a resource. For stateless HTTP services, you almost always want RollingUpdate with maxUnavailable zero to maintain availability throughout the update."

---

## 8. Debugging — Fast Diagnosis Paths

### Rollout Stuck — Diagnosis Path

```
kubectl rollout status deployment/<name>
"Waiting for deployment ... rollout to finish"
        │
        ├── Stuck for < progressDeadlineSeconds?
        │         → Wait, or check pod logs
        │
        └── Stuck for > progressDeadlineSeconds?
                  │
                  ├── Check condition
                  │   kubectl get deployment <n> -o yaml | grep -A10 conditions
                  │   → ProgressDeadlineExceeded = rollout stalled
                  │
                  ├── Check new RS pods
                  │   kubectl get pods -l app=nginx
                  │   kubectl describe pod <new-pod>
                  │   → ImagePullBackOff → wrong image name or registry
                  │   → CrashLoopBackOff → app startup failure
                  │   → Readiness probe failing → app not healthy
                  │
                  ├── Check pod logs
                  │   kubectl logs <new-pod>
                  │   kubectl logs <new-pod> --previous  (if crashed)
                  │
                  └── Roll back
                      kubectl rollout undo deployment/<n>
```

### Pod Not Spreading Across Nodes

```bash
# Check where pods landed
kubectl get pods -o wide

# Check node capacity
kubectl describe nodes | grep -A5 "Allocated resources"

# If all pods on one node → scheduling constraint or node pressure
kubectl get pods -o wide --show-labels
kubectl describe pod <pod> | grep -A10 Events
# Look for: "didn't match node selector" "insufficient cpu" "taint"
```

### Deployment Not Updating (Apply Has No Effect)

```bash
# Check if image actually changed
kubectl get deployment <n> -o jsonpath='{.spec.template.spec.containers[0].image}'

# Check if deployment is paused
kubectl get deployment <n> -o jsonpath='{.spec.paused}'

# Check rollout history for new revision
kubectl rollout history deployment/<n>

# Check if selector mismatch is preventing ownership
kubectl get rs --show-labels
kubectl get pods --show-labels
```

### Deployment Shows 0/3 Ready

```bash
# Check pods
kubectl get pods -l app=nginx

# Pod in Pending → scheduler issue (no resources, taint, affinity)
kubectl describe pod <pod> | grep -A10 Events

# Pod in CrashLoopBackOff → app crashing
kubectl logs <pod>
kubectl logs <pod> --previous

# Pod in ContainerCreating → CNI or image issue
kubectl describe pod <pod>
# "network plugin not ready" → CNI issue
# "ErrImagePull" / "ImagePullBackOff" → image issue

# Pod Running but not Ready → readiness probe failing
kubectl describe pod <pod> | grep -A5 "Readiness"
kubectl logs <pod>  # check app logs for startup errors
```

### Check Deployment Health at a Glance

```bash
# Full status overview
kubectl get deployment <n> -o wide
# READY column: 3/3 = healthy, 1/3 = problem

# Detailed conditions
kubectl get deployment <n> -o jsonpath='{.status.conditions}' | jq .

# All RS owned by deployment
kubectl get rs -l app=nginx --sort-by=.metadata.creationTimestamp

# Recent events
kubectl get events --field-selector involvedObject.name=<n> \
  --sort-by='.lastTimestamp'
```

---

## 9. Kill Switch — 10-Second Recall

```
Deployment → owns → ReplicaSet → owns → Pods

Image change     = new ReplicaSet (old kept at 0 for rollback)
Scale change     = NO new ReplicaSet (just adjusts replica count)
Rollback         = old RS rescaled up (revision counter advances)

RollingUpdate (default):  maxSurge=1, maxUnavailable=0 → zero downtime
Recreate:                 all old pods die first → brief downtime

readinessProbe = what gates rollout progress — NEVER skip it
progressDeadlineSeconds = how long before rollout declared failed (no auto-rollback)

History commands:
  kubectl rollout history deployment/<n>
  kubectl rollout history deployment/<n> --revision=2

Rollback commands:
  kubectl rollout undo deployment/<n>
  kubectl rollout undo deployment/<n> --to-revision=1

Pause/Resume = batch multiple changes into one rollout

revisionHistoryLimit: 0 = no rollback possible (never do this)
selector is immutable — cannot change after creation
```

---

## 10. Appendix — Quick Reference Card

### Core Commands Cheatsheet

```bash
# Create
kubectl create deployment <n> --image=<img> --replicas=3 --dry-run=client -o yaml
kubectl apply -f deploy.yaml

# Read
kubectl get deployment <n>
kubectl get deployment <n> -o wide
kubectl get deployment <n> -o yaml
kubectl describe deployment <n>
kubectl get rs -l app=<label>
kubectl get pods -o wide -l app=<label>

# Update
kubectl set image deployment/<n> <container>=<image>:<tag>
kubectl set resources deployment/<n> --containers=<c> --requests=cpu=100m
kubectl edit deployment/<n>
kubectl apply -f deploy.yaml                     # declarative (preferred)

# Annotate (always after update/rollback)
kubectl annotate deployment <n> \
  kubernetes.io/change-cause="<message>" --overwrite

# Scale
kubectl scale deployment <n> --replicas=5

# Rollout
kubectl rollout status deployment/<n>
kubectl rollout history deployment/<n>
kubectl rollout history deployment/<n> --revision=2
kubectl rollout undo deployment/<n>
kubectl rollout undo deployment/<n> --to-revision=1
kubectl rollout pause deployment/<n>
kubectl rollout resume deployment/<n>

# Delete
kubectl delete deployment <n>
kubectl delete -f deploy.yaml
```

### Deployment Spec Quick Reference

```yaml
spec:
  replicas: 3                          # desired pod count
  revisionHistoryLimit: 5              # old RS to keep for rollback
  progressDeadlineSeconds: 600         # max time for rollout progress
  selector:
    matchLabels:                       # IMMUTABLE after creation
      app: nginx
  strategy:
    type: RollingUpdate                # or Recreate
    rollingUpdate:
      maxSurge: 1                      # extra pods above desired
      maxUnavailable: 0                # pods below desired allowed
  template:
    metadata:
      labels:
        app: nginx                     # must match selector
    spec:
      containers:
      - name: nginx
        image: nginx:1.19
        readinessProbe: ...            # gates rollout — always set this
        livenessProbe: ...             # restarts unhealthy containers
        resources:
          requests: ...
          limits: ...
```

### Rollout History with Meaningful Output

```bash
# Without annotation — useless
REVISION  CHANGE-CAUSE
1         <none>
2         <none>

# With annotation — production-grade
REVISION  CHANGE-CAUSE
1         Initial nginx:1.19 deployment
2         Upgrade nginx 1.19 → 1.20 — performance fix
3         Rollback to nginx:1.19 — startup crash in 1.20
```

### maxSurge / maxUnavailable Combinations

| maxSurge | maxUnavailable | Behavior | Use When |
|---|---|---|---|
| 1 | 0 | Zero downtime, slight over-capacity | Production default |
| 0 | 1 | No extra pods, brief partial unavailability | Node capacity constrained |
| 25% | 25% | Kubernetes default — balanced | General purpose |
| 100% | 0 | All new pods before any old removed — fastest | Plenty of spare capacity |
| 0 | 0 | Invalid — would deadlock | Never |

### Deployment Conditions Reference

| Condition | Type | Meaning |
|---|---|---|
| `Available` | True | Min available replicas threshold met |
| `Progressing` | True | Rollout is in progress or recently completed |
| `Progressing` | False + `ProgressDeadlineExceeded` | Rollout stalled — intervention needed |

```bash
# Check conditions
kubectl get deployment <n> -o jsonpath='{.status.conditions[*].type}'
kubectl get deployment <n> -o jsonpath='{.status.conditions[*].reason}'
```

### ReplicaSet State During Rollout

```bash
kubectl get rs -l app=nginx --sort-by=.metadata.creationTimestamp

# During rollout:
NAME                        DESIRED   CURRENT   READY   AGE
nginx-deployment-75675f5897  1         1         0       10s   ← new, ramping up
nginx-deployment-d7f7c5f4c   2         2         2       5m    ← old, ramping down

# After rollout:
nginx-deployment-75675f5897  3         3         3       2m    ← active
nginx-deployment-d7f7c5f4c   0         0         0       7m    ← kept for rollback
```

---

