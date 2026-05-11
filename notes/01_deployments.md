# ☸️ Kubernetes Study Guide — Kind Clusters, Contexts & Workload Controllers
### *CKA + Senior Interview Prep | Day 08 + Day 10*

> **How to use this doc:** Read top to bottom once. Then use sections 7–10 as your daily flashcard loop. Every section flows into the next — this is a story, not a dump.

---

## 0. First Principles 🧠
*The mental model — what never changes*

**Three laws that govern everything in Kubernetes:**

1. **Declare, don't command.** You tell Kubernetes *what you want*, not *how to get there*. A YAML file is a contract. The control plane figures out the rest.

2. **Labels are the glue.** Everything in Kubernetes finds everything else through labels and selectors. No labels → no relationships → nothing works as a system.

3. **Controllers reconcile reality toward desire.** A ReplicationController, ReplicaSet, or Deployment watches the cluster and asks one question in a loop: *"Is the actual state equal to the desired state?"* If not, it acts.

**The mental model for workload controllers:**

```
You write a Deployment
  └── Deployment manages a ReplicaSet
        └── ReplicaSet manages Pods
              └── Pods run your containers
```

Deployments didn't replace ReplicaSets. They *wrap* them and add version history, rolling updates, and rollback on top.

---

## 1. Reality Constraints ⚠️
*What Kubernetes actually does — and doesn't do*

| What People Assume | What Actually Happens |
|---|---|
| `kubectl create cluster` just works anywhere | Kind runs clusters **inside Docker containers** — it's for local dev only, not production |
| A ReplicationController does rolling updates | **It does not.** Updating the image on an RC leaves existing pods untouched. Only new pods get the new image. |
| A standalone ReplicaSet does rolling updates | **Also no.** Same problem. Rolling updates only happen when a Deployment manages the ReplicaSet. |
| `current-context` is always the cluster I'm working on | Only if you actively set it. Forgetting to switch context is one of the most common production accidents. |
| The annotation `kubernetes.io/change-cause` is automatic | **You must set it yourself** — either in the YAML or via `kubectl annotate`. Kubernetes does not write it for you. |
| Rollback restores the old pods | Rollback actually **scales up the old ReplicaSet** and scales down the current one. Old pods are gone; new pods matching the old spec are created. |

---

## 2. Decision Logic 🔀
*When to use what — clean rules, no ambiguity*

### Which workload controller do I use?

```
Do you need pods to run reliably and recover from failures?
│
├── YES → Do you need rolling updates / rollback?
│         ├── YES → Use a DEPLOYMENT (almost always this)
│         └── NO  → Do you need complex label matching (In, NotIn, Exists)?
│                   ├── YES → Use a REPLICASET (rare, standalone)
│                   └── NO  → Use a REPLICATIONCONTROLLER (legacy only, avoid)
│
└── NO  → You probably want a Job, DaemonSet, or StatefulSet (out of scope here)
```

### Which selector type do I use?

| Situation | Selector Type | Example |
|---|---|---|
| Match one exact label value | Equality-based (`matchLabels`) | `app: nginx` |
| Match one of several values | Set-based (`In`) | `app in (nginx, apache)` |
| Exclude certain values | Set-based (`NotIn`) | `environment notin (development)` |
| Just check if a label key exists | Set-based (`Exists`) | `tier` exists |
| Ensure a label key is absent | Set-based (`DoesNotExist`) | `debug` does not exist |

**Rule of thumb:** If you're writing a Deployment (which you almost always are), use `matchLabels` for simplicity. Pull out `matchExpressions` when you genuinely need multi-value or existence logic.

### Config file vs imperative command?

| Approach | When to use it | When NOT to use it |
|---|---|---|
| YAML / config file | Always in production; CI/CD pipelines; anything that needs to be tracked | Never the "quick one-off" in prod |
| Imperative command (`kubectl create`, `kubectl set image`) | Quick local experiments; debugging; forced exam constraints | Anywhere with version control or a team |

---

## 3. Internal Working 🔩
*How it actually happens under the hood — step by step*

### How a Kind cluster boots up

1. You run `kind create cluster --name my-first-cluster --config kind-cluster.yaml`
2. Kind reads the YAML — it sees 1 control-plane node and 2 worker nodes
3. Kind pulls the `kindest/node` Docker image (pinned by SHA256 in the config)
4. Three Docker containers spin up — one per node
5. Inside each container, `kubeadm` bootstraps the Kubernetes components
6. A `kubeconfig` entry is written to `~/.kube/config` on your machine
7. `kubectl` is now pointed at that cluster via the new context

### How a ReplicaSet maintains desired state

1. You apply a ReplicaSet manifest (3 replicas, selector `app: nginx`)
2. The ReplicaSet controller watches for pods matching `app: nginx`
3. It finds 0 pods → it creates 3 pods from the pod template
4. A pod dies → the controller sees 2 actual vs 3 desired → creates 1 new pod
5. You manually create a pod with label `app: nginx` → controller sees 4 vs 3 → **deletes 1** (even the one you created manually — this trips people up)

### How a Deployment rolling update works

1. You update the image: `kubectl set image deployment nginx-deployment nginx-container=nginx:1.20`
2. Kubernetes creates a **new ReplicaSet** (call it RS-v2) with 0 replicas
3. RS-v2 scales up 1 pod → waits for it to be Ready
4. Old ReplicaSet (RS-v1) scales down 1 pod
5. Repeat until RS-v2 has all 3 replicas, RS-v1 has 0
6. RS-v1 still exists (with 0 replicas) — this is how rollback works

### How rollback works

1. `kubectl rollout undo deployment nginx-deployment --to-revision=1`
2. Kubernetes looks up RS-v1 (the ReplicaSet from revision 1)
3. RS-v1 scales back up to 3 replicas
4. Current ReplicaSet scales down to 0
5. No pods are "restored" — fresh pods are created matching the old pod spec

### How kubeconfig context switching works

```
~/.kube/config
├── clusters:   [ cluster-1, cluster-2, cluster-3 ]   ← API server URLs + certs
├── users:      [ varun-user ]                         ← auth credentials
├── contexts:   [ dev-context, staging-context, prod-context ]  ← cluster + user + namespace triplets
└── current-context: dev-context                       ← the active one
```

`kubectl get pods` → reads `current-context` → resolves cluster + user → sends request to that API server.

---

## 4. Hands-On 🛠️
*Production-quality YAML and commands — nothing dumbed down*

### Kind cluster config (pinned version, multi-node)

```yaml
# kind-cluster.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    image: kindest/node:v1.31.4@sha256:2cb39f7295fe7eafee0842b1052a599a4fb0f8bcf3f83d96c7f4864c357c6c30
  - role: worker
    image: kindest/node:v1.31.4@sha256:2cb39f7295fe7eafee0842b1052a599a4fb0f8bcf3f83d96c7f4864c357c6c30
  - role: worker
    image: kindest/node:v1.31.4@sha256:2cb39f7295fe7eafee0842b1052a599a4fb0f8bcf3f83d96c7f4864c357c6c30
```

```bash
# Create cluster from config (always prefer this over bare command)
kind create cluster --name my-first-cluster --config kind-cluster.yaml

# Create cluster without config (defaults to latest image — avoid in production)
kind create cluster --name my-second-cluster
```

### ReplicationController (legacy — know it, don't use it)

```yaml
# rc.yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx-rc
spec:
  replicas: 3
  selector:
    app: nginx           # equality-based ONLY — this is all RC supports
  template:
    metadata:
      labels:
        app: nginx
        environment: development
    spec:
      containers:
        - name: nginx-container
          image: nginx
```

```bash
kubectl apply -f rc.yaml
kubectl get rc
kubectl describe rc nginx-rc
kubectl get pods --selector app=nginx
kubectl scale rc nginx-rc --replicas=4
```

### ReplicaSet with equality-based selector

```yaml
# rs.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx         # pods must have this label to be "adopted"
  template:
    metadata:
      labels:
        app: nginx
        environment: development
    spec:
      containers:
        - name: nginx-container
          image: nginx
```

### ReplicaSet with set-based selector (the interesting one)

```yaml
# rs-set-based.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: example-rs
spec:
  replicas: 3
  selector:
    matchExpressions:
      - key: app
        operator: In
        values:
          - nginx
          - apache
      - key: environment
        operator: NotIn
        values:
          - development
      - key: tier
        operator: Exists
      - key: debug
        operator: DoesNotExist
  template:
    metadata:
      labels:
        app: nginx
        environment: production
        tier: frontend
        # ⚠️ Do NOT add "debug" label here — it would violate the DoesNotExist rule
    spec:
      containers:
        - name: nginx-container
          image: nginx
```

### Deployment with rollout tracking

```yaml
# deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  annotations:
    kubernetes.io/change-cause: "Initial release with nginx 1.19"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
        environment: development
    spec:
      containers:
        - name: nginx-container
          image: nginx:1.19
```

### Full kubeconfig example

```yaml
# ~/.kube/config
apiVersion: v1
clusters:
  - name: cluster-1
    cluster:
      server: https://cluster-1-api-server
      certificate-authority-data: <certificate-data>
  - name: cluster-2
    cluster:
      server: https://cluster-2-api-server
      certificate-authority-data: <certificate-data>
  - name: cluster-3
    cluster:
      server: https://cluster-3-api-server
      certificate-authority-data: <certificate-data>
users:
  - name: varun-user
    user:
      client-certificate-data: <client-cert-data>
      client-key-data: <client-key-data>
contexts:
  - name: dev-context
    context:
      cluster: cluster-1
      user: varun-user
      namespace: dev
  - name: staging-context
    context:
      cluster: cluster-2
      user: varun-user
      namespace: staging
  - name: prod-context
    context:
      cluster: cluster-3
      user: varun-user
      namespace: prod
current-context: dev-context
```

---

## 5. Production Flow 🏗️
*Real-world patterns and architecture decisions*

### Multi-cluster context management (the Varun pattern)

A single engineer managing dev/staging/prod from one laptop:

```
Laptop (~/.kube/config)
├── dev-context     → cluster-1 (development)   namespace: dev
├── staging-context → cluster-2 (staging)       namespace: staging
└── prod-context    → cluster-3 (production)    namespace: prod
```

**Production rule:** In real environments, cluster name, user name, and context name are always **distinct**. Kind makes them the same (e.g., `kind-my-first-cluster` for everything) — that's a convenience shortcut that would confuse a team. In production (kubeadm, EKS, GKE, etc.), these are named separately and meaningfully.

### Deployment lifecycle pattern (declarative, version-controlled)

```
1. Write deploy.yaml with annotation kubernetes.io/change-cause: "Initial release nginx 1.19"
2. git commit && kubectl apply -f deploy.yaml
3. To update: edit deploy.yaml → change image tag + annotation → git commit → kubectl apply
4. Never use kubectl set image in production (it creates drift between git and cluster)
5. Use kubectl rollout history to audit; kubectl rollout undo to recover
```

### The ReplicaSet graveyard (and why it matters)

Every Deployment update creates a new ReplicaSet. Old ReplicaSets stick around at 0 replicas. This is by design — they're your rollback points. If you see 5 ReplicaSets for one Deployment, that's 5 revisions of history. This is normal and expected.

### Pause/Resume for batched changes

```bash
kubectl rollout pause deployment nginx-deployment
# make multiple changes here without triggering multiple rollouts
kubectl set image deployment nginx-deployment nginx-container=nginx:1.21
kubectl set resources deployment nginx-deployment -c=nginx-container --limits=cpu=200m,memory=512Mi
kubectl rollout resume deployment nginx-deployment
# one single rollout now applies all queued changes
```

---

## 6. Mistakes 💣
*What actually breaks in real systems — root cause + fix*

### Mistake 1: Wrong context → commands hit the wrong cluster

**What happens:** You run `kubectl delete pod prod-db-xxxx` thinking you're in dev. You're in prod.

**Root cause:** `current-context` wasn't updated after the last switch.

**Fix:**
```bash
kubectl config current-context   # always check before destructive operations
kubectl config use-context dev-context
```

**Prevention:** Add `kubectl config current-context` to your shell prompt (tools like `kube-ps1` do this).

### Mistake 2: Pod labels don't match ReplicaSet selector → pods never adopted

**What happens:** You apply a ReplicaSet expecting 3 pods. You get 3 pods from the RS **plus** your pre-existing pod that you thought would be adopted.

**Root cause:** The existing pod's labels didn't satisfy `matchExpressions`. The RS created its own.

**Fix:** Check the pod labels against all selector rules:
```bash
kubectl get pods --show-labels
kubectl describe rs example-rs   # look at "Selector" field
```

### Mistake 3: Adding a `debug` label to a pod managed by a DoesNotExist selector

**What happens:** The pod immediately falls out of scope. The RS sees replica count drop and creates a new pod to replace it.

**Root cause:** The pod no longer matches the selector → RS disowns it → RS creates a new replacement.

**Fix:** Never add/remove labels on running pods that are managed by a ReplicaSet unless you intend the consequence.

### Mistake 4: Using `kubectl set image` on an RC/standalone RS expecting rolling update

**What happens:** The image changes in the spec, but all running pods keep the old image. You think the update worked.

**Root cause:** RC and standalone RS do **not** replace running pods when the template changes. Only Deployments do.

**Fix:** Use Deployments for anything that needs live updates.

### Mistake 5: Forgetting `kubernetes.io/change-cause` → rollout history is useless

**What happens:** `kubectl rollout history` shows 5 revisions, all with `<none>` in the CHANGE-CAUSE column. You can't tell which revision is which.

**Fix:** Always include the annotation in your YAML. In a pinch:
```bash
kubectl annotate deployment nginx-deployment kubernetes.io/change-cause="Upgraded nginx 1.20 to 1.21"
```

---

## 7. Interview Answers 🎤
*Full spoken sentences — deliver these verbatim*

**Q: What is a Kubernetes context and why does it matter?**

"A Kubernetes context is a named combination of a cluster, a user, and a namespace stored in the kubeconfig file. It exists so that a single engineer can manage multiple clusters from one machine without rewriting credentials each time. The active context is tracked by the `current-context` field, and switching it with `kubectl config use-context` is what redirects all subsequent kubectl commands to a different cluster. In production, missing a context switch before a destructive command is one of the most common and dangerous operational mistakes."

**Q: What is the difference between a ReplicationController, a ReplicaSet, and a Deployment?**

"All three ensure a specified number of pod replicas are running. The key differences are in selector support and update behavior. A ReplicationController only supports equality-based selectors — it's legacy and rarely used today. A ReplicaSet adds set-based selectors with operators like In, NotIn, Exists, and DoesNotExist, giving you much finer control over which pods it manages. A Deployment wraps a ReplicaSet and layers on rolling updates, rollback history, and change annotations. In practice, you almost always write Deployments, which manage ReplicaSets for you behind the scenes."

**Q: How does a Kubernetes rolling update work internally?**

"When you update a Deployment — say, changing the container image — Kubernetes creates a brand new ReplicaSet for the new version with zero replicas. It then scales that new ReplicaSet up one pod at a time, waiting for each pod to become Ready before scaling down one pod from the old ReplicaSet. This continues until the new ReplicaSet has all the desired replicas and the old one has zero. The old ReplicaSet is kept around at zero replicas specifically to enable rollback — when you run `kubectl rollout undo`, Kubernetes just reverses the direction and scales the old one back up."

**Q: What is the difference between equality-based and set-based selectors?**

"Equality-based selectors match resources where a label key equals a specific value, using `=` or `!=`. They're simple and used in ReplicationControllers and as `matchLabels` in ReplicaSets and Deployments. Set-based selectors are more expressive — they use `In`, `NotIn`, `Exists`, and `DoesNotExist` operators and are specified under `matchExpressions`. For example, you can select pods where the `app` label is either `nginx` or `apache`, or where the `debug` label doesn't exist at all. Both types can be combined in the same selector block."

**Q: Why would you use a config file over an imperative command to create a cluster?**

"Configuration files give you version control, reproducibility, and a declarative source of truth. If you create a cluster imperatively, you have no record of exactly what was created — the node count, the Kubernetes version, and any custom settings live only in someone's head or shell history. With a YAML config file, you can commit it to Git, share it across the team, recreate the exact same cluster in any environment, and roll back if a config change causes problems. This is the same philosophy that drives Terraform, Helm, and Kubernetes manifests themselves."

**Q: What happens when you add a label to a pod that's already managed by a ReplicaSet, making it no longer match the selector?**

"The pod is disowned — it falls out of the ReplicaSet's management scope. The ReplicaSet sees that its actual replica count has dropped below the desired count and immediately creates a new pod to compensate. The original pod is now unmanaged and will not be cleaned up by the ReplicaSet. This is a real footgun — accidentally removing a matching label from a pod in production can trigger unexpected pod creation and leave orphaned pods running."

---

## 8. Debugging 🔬
*Fast diagnosis paths — commands and decision trees*

### My pods aren't being created by a ReplicaSet

```
kubectl get rs <rs-name>                   # check DESIRED vs CURRENT vs READY
│
├── DESIRED=3, CURRENT=0 → selector is broken
│   kubectl describe rs <rs-name>          # look at Events section
│   kubectl get pods --show-labels         # do existing pods match selector?
│
├── DESIRED=3, CURRENT=3, READY=0 → pods are crashing
│   kubectl get pods                       # look for CrashLoopBackOff or ImagePullBackOff
│   kubectl describe pod <pod-name>        # look at Events — image pull? OOM? probe fail?
│   kubectl logs <pod-name>               # application logs
│
└── DESIRED=3, CURRENT=6 → pre-existing pods with matching labels got adopted
    kubectl get pods --show-labels         # identify orphans
    kubectl label pod <pod-name> app-      # remove label to un-adopt (or delete the orphan)
```

### My rollout is stuck

```
kubectl rollout status deployment <name>   # is it progressing?
│
├── "Waiting for deployment to finish" → check pod health
│   kubectl get pods                       # new pods ready?
│   kubectl describe pod <new-pod>         # readiness probe failures?
│
├── Nothing happening → check if deployment is paused
│   kubectl get deployment <name> -o yaml | grep paused
│   kubectl rollout resume deployment <name>
│
└── Still stuck → check deployment events
    kubectl describe deployment <name>     # Events section at bottom
```

### I ran a command on the wrong cluster

```
kubectl config current-context            # verify where you are NOW
kubectl config get-contexts              # list all available contexts
kubectl config use-context <correct-ctx>  # switch to the right one

# If you deleted something accidentally:
kubectl get events --sort-by=.metadata.creationTimestamp   # what just happened?
# Restore from git-tracked YAML if available
kubectl apply -f <manifest-from-git>.yaml
```

### Rollout history is all `<none>`

```
# You forgot to set the annotation. Patch it retroactively:
kubectl annotate deployment <name> kubernetes.io/change-cause="describe what changed"

# View history after patching:
kubectl rollout history deployment <name>
```

### Context commands quick-reference

```bash
kubectl config current-context              # what cluster am I on?
kubectl config get-contexts                 # list all contexts
kubectl config use-context <ctx>           # switch context
kubectl config set-context --current --namespace=<ns>  # set default namespace
kubectl config view                        # dump full kubeconfig
```

---

## 9. Kill Switch ⚡
*10-second recall — the absolute minimum to hold in memory*

```
RC   → equality selectors only    → no rolling updates    → legacy, avoid
RS   → set-based selectors        → no rolling updates    → use inside Deployments
Deploy → wraps RS                 → rolling updates + rollback → always use this

Labels  = tags on objects
Selectors = filters to find objects by tags

Context = cluster + user + namespace triplet stored in ~/.kube/config
current-context = where kubectl is pointing right now

Rolling update = new RS scales up while old RS scales down
Rollback = old RS scales back up, current RS scales to 0

Config files > imperative commands. Always.
```

---

## 10. Appendix 📋
*Quick reference card*

### Kind commands

```bash
kind create cluster --name <name> --config <file.yaml>   # create with config
kind create cluster --name <name>                        # create (defaults to latest)
kind get clusters                                        # list clusters
kind delete cluster --name <name>                        # delete cluster
```

### Context management

```bash
kubectl config current-context                           # active context
kubectl config get-contexts                              # all contexts
kubectl config use-context <context-name>               # switch context
kubectl config set-context --current --namespace=<ns>   # set default namespace
kubectl config view                                      # view full kubeconfig
```

### ReplicationController

```bash
kubectl apply -f rc.yaml
kubectl get rc
kubectl describe rc <name>
kubectl scale rc <name> --replicas=5
kubectl delete rc <name>
```

### ReplicaSet

```bash
kubectl apply -f rs.yaml
kubectl get rs
kubectl describe rs <name>
kubectl scale rs <name> --replicas=5
kubectl get pods --selector app=nginx
kubectl delete rs <name>
```

### Deployment — full lifecycle

```bash
# Create
kubectl create deployment nginx-deployment --image=nginx:1.19   # imperative
kubectl apply -f deploy.yaml                                    # declarative (preferred)

# Inspect
kubectl get deployments
kubectl describe deployment <name>
kubectl get deployment <name> -o yaml

# Update
kubectl set image deployment/<name> <container>=nginx:1.20      # imperative
kubectl annotate deployment <name> kubernetes.io/change-cause="reason"

# Scale
kubectl scale deployment/<name> --replicas=5

# Rollout
kubectl rollout status deployment/<name>
kubectl rollout history deployment/<name>
kubectl rollout history deployment/<name> --revision=2          # inspect a specific revision
kubectl rollout undo deployment/<name>                          # rollback to previous
kubectl rollout undo deployment/<name> --to-revision=1         # rollback to specific revision

# Pause / Resume
kubectl rollout pause deployment/<name>
kubectl rollout resume deployment/<name>

# Edit live
kubectl edit deployment <name>

# Delete
kubectl delete deployment <name>
```

### Selector operators (set-based)

| Operator | YAML key | Meaning |
|---|---|---|
| `In` | `operator: In` | Label value is one of the listed values |
| `NotIn` | `operator: NotIn` | Label value is NOT any of the listed values |
| `Exists` | `operator: Exists` | Label key is present (any value) |
| `DoesNotExist` | `operator: DoesNotExist` | Label key is absent |

### Controller comparison

| Feature | ReplicationController | ReplicaSet | Deployment |
|---|---|---|---|
| API version | `v1` | `apps/v1` | `apps/v1` |
| Shortname | `rc` | `rs` | `deploy` |
| Selectors | Equality only | Equality + Set-based | Equality + Set-based |
| Rolling updates | ❌ No | ❌ No | ✅ Yes |
| Rollback | ❌ No | ❌ No | ✅ Yes |
| Use today? | ❌ Legacy | 🟡 Rarely (standalone) | ✅ Always |

### Annotations

```bash
# Add/update annotation
kubectl annotate deployment <name> kubernetes.io/change-cause="your message"

# In YAML (preferred)
metadata:
  annotations:
    kubernetes.io/change-cause: "Initial release with nginx 1.19"
```

### Useful inspection commands

```bash
kubectl api-resources                       # list all resource types with shortnames
kubectl explain deployment.spec             # field-by-field docs in terminal
kubectl get pods --show-labels             # see labels on pods
kubectl get all                            # everything in current namespace
kubectl get events --sort-by=.metadata.creationTimestamp   # recent events
```

---
