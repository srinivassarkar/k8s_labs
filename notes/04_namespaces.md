# Kubernetes Namespaces — CKA + Senior Interview Master Guide
---

## 0. First Principles

> *The mental model — what never changes, regardless of cluster size or Kubernetes version.*

**A namespace is a virtual cluster inside a physical cluster.** It is not a VM, not a network boundary, not a security perimeter by itself. It is a Kubernetes API grouping mechanism that scopes names, access control, and resource quotas — nothing more, nothing less. Every security or isolation guarantee you associate with namespaces must be explicitly enforced through additional primitives: NetworkPolicy, RBAC, LimitRanges, ResourceQuotas.

**Three laws that never change:**

1. **Names are scoped, not objects.** Two Deployments named `backend-deploy` can coexist in `dev-ns` and `prod-ns` simultaneously. The name is unique *within* a namespace, not across the cluster. Cluster-scoped objects (Nodes, PersistentVolumes, ClusterRoles, StorageClasses) have no namespace — they exist once, globally.

2. **DNS is the namespace contract.** Every Service gets a fully-qualified domain name: `<service>.<namespace>.svc.cluster.local`. Within the same namespace, the short name `<service>` resolves via DNS search domains. Cross-namespace access requires the FQDN. This is the mechanism, not a side effect.

3. **Namespaces do not enforce network isolation.** A pod in `dev-ns` can reach a pod in `prod-ns` at the network layer unless a NetworkPolicy explicitly blocks it. The namespace boundary is logical, not physical.

**The mental model in one diagram:**

```
Cluster (physical boundary)
│
├── kube-system ns    ← Kubernetes control plane components
├── kube-public ns    ← Publicly readable cluster info
├── kube-node-lease ns ← Node heartbeat Lease objects
├── default ns        ← Catch-all for unscoped resources
│
├── app1-ns           ← Your application namespace
│     ├── backend-deploy (Deployment)
│     ├── backend-svc   (Service: ClusterIP)
│     ├── frontend-deploy
│     └── frontend-svc  (Service: NodePort)
│
└── app2-ns           ← Another team / another app
      └── ...

Cross-cutting (no namespace):
  Nodes, PersistentVolumes, StorageClasses, ClusterRoles, Namespaces themselves
```

Everything flows from this model. Quotas apply per namespace. RBAC RoleBindings apply per namespace. DNS search domains apply per namespace. Master this and every interview question becomes a variation.

---

## 1. Reality Constraints

> *What Kubernetes actually does and doesn't do — the gaps that cause production incidents.*

### What Namespaces Guarantee

| Guarantee | Detail |
|---|---|
| Name uniqueness | Two resources of the same kind cannot share a name within one namespace |
| DNS short-name resolution | `curl backend-svc:9090` works from within the same namespace via CoreDNS search domains |
| ResourceQuota enforcement | If a quota is set, the API server rejects resource creation that exceeds it |
| RBAC scoping | A `Role` and `RoleBinding` in namespace A cannot grant access to namespace B |
| `kubectl` context scoping | `kubectl config set-context --current --namespace=X` makes X the default for all subsequent commands |

### What Namespaces Do NOT Guarantee

| Misconception | Reality |
|---|---|
| Namespaces = network isolation | Pods across namespaces can communicate freely. NetworkPolicy is required for isolation |
| Deleting a namespace is instant | Kubernetes must delete every resource inside first. Large namespaces can hang in `Terminating` for minutes |
| `kube-system` is protected from user workloads | Without RBAC restrictions, any user with cluster-admin can modify kube-system |
| Namespace-scoped RBAC blocks API access | A ClusterRoleBinding can grant cluster-wide access even if a RoleBinding is namespace-scoped |
| Cross-namespace DNS with short name | `curl backend-svc:9090` from a different namespace **fails**. You must use `backend-svc.app1-ns.svc.cluster.local:9090` |
| Resources inherit namespace labels | Labels on a namespace object do NOT propagate to resources inside it |

### Cluster-Scoped vs. Namespace-Scoped Resources

This distinction is probed heavily in CKA. Misplacing a `namespace:` field on a cluster-scoped resource causes a schema validation error.

| Namespace-Scoped | Cluster-Scoped |
|---|---|
| Pod, Deployment, ReplicaSet | Node |
| Service, Endpoints | PersistentVolume |
| ConfigMap, Secret | StorageClass |
| ServiceAccount | Namespace itself |
| Role, RoleBinding | ClusterRole, ClusterRoleBinding |
| LimitRange, ResourceQuota | CustomResourceDefinition |
| NetworkPolicy | PriorityClass |

### DNS Resolution Mechanics

```
Pod in app1-ns queries: backend-svc
CoreDNS search domains (injected via /etc/resolv.conf):
  search app1-ns.svc.cluster.local svc.cluster.local cluster.local

Resolution chain:
  backend-svc → backend-svc.app1-ns.svc.cluster.local → ClusterIP ✅

Pod in default ns queries: backend-svc
  backend-svc → backend-svc.default.svc.cluster.local → NXDOMAIN ❌
  Must use: backend-svc.app1-ns.svc.cluster.local ✅
```

---

## 2. Decision Logic

> *When to use what — clear rules before writing a single line of YAML.*

### When to Create a New Namespace

```
Is there a reason to isolate this workload?
│
├── Different team owns it → separate namespace
│     Reason: independent RBAC, independent quota, independent lifecycle
│
├── Different environment (dev/staging/prod) → separate namespace per env
│     Reason: prevent cross-env interference, allow different quotas
│
├── Different security posture → separate namespace + NetworkPolicy
│     Reason: PCI, HIPAA, or internal compliance boundary
│
├── Different billing unit → separate namespace + ResourceQuota + labels
│     Reason: chargeback reporting per namespace
│
└── Same team, same app, same env, no isolation needed → same namespace
      Reason: don't over-partition; operational complexity grows with namespace count
```

### Namespace Naming Convention

```
Pattern: <team>-<app>-<env>-ns

Examples:
  payments-api-prod-ns
  data-pipeline-staging-ns
  platform-tools-dev-ns

Anti-patterns:
  ns1, test, myapp     ← ambiguous, not searchable
  prod                 ← too broad; multiple teams share it
```

### kubectl -n Flag Decision

```
Are you working in the default namespace?
│
├── Yes → no flag needed (but verify with: kubectl config view --minify | grep namespace)
│
├── No, working in one namespace for a session → set context:
│     kubectl config set-context --current --namespace=app1-ns
│     (removes need for -n on every command)
│
└── No, one-off command in a different namespace → use -n flag:
      kubectl get pods -n app1-ns
```

### ResourceQuota vs. LimitRange — Decision Table

| Need | Use |
|---|---|
| Cap total CPU/memory/object count per namespace | ResourceQuota |
| Set default requests/limits per container | LimitRange |
| Prevent any single pod from using too much | LimitRange (max) |
| Prevent namespace from consuming all cluster resources | ResourceQuota |
| Both | Both — they are complementary, not alternatives |

---

## 3. Internal Working

> *How it actually happens under the hood — step by step.*

### Namespace Creation

1. You apply a Namespace object → **API Server** validates and persists to etcd.
2. The **Namespace controller** (inside kube-controller-manager) watches for new Namespace objects. When it detects one in `Active` phase, it creates the default ServiceAccount (`default`) inside the namespace and provisions any default Secrets (prior to Kubernetes 1.24, a token Secret was auto-created).
3. **CoreDNS** does not need to be updated — it dynamically resolves `<svc>.<ns>.svc.cluster.local` by querying the API server via its watch cache.
4. **RBAC admission** begins enforcing namespace scope for all subsequent requests targeting this namespace.

### How CoreDNS Resolves Cross-Namespace Names

1. Every pod gets `/etc/resolv.conf` injected by kubelet at startup containing:
   ```
   nameserver <CoreDNS ClusterIP>
   search <pod-namespace>.svc.cluster.local svc.cluster.local cluster.local
   options ndots:5
   ```
2. When a pod resolves `backend-svc`, the OS stub resolver appends search domains in order until a match is found. The first match (`backend-svc.<pod-namespace>.svc.cluster.local`) resolves if a Service named `backend-svc` exists in the same namespace.
3. For cross-namespace, `backend-svc.app1-ns` expands to `backend-svc.app1-ns.svc.cluster.local` — CoreDNS finds the Service record and returns the ClusterIP.
4. `ndots:5` means any name with fewer than 5 dots is treated as relative and search domains are tried first. A FQDN with a trailing dot bypasses search domains entirely.

### Namespace Deletion — Why It Can Hang

1. `kubectl delete namespace app1-ns` sends a DELETE to the API server.
2. API server sets the namespace phase to `Terminating` and adds a `deletionTimestamp`.
3. The **Namespace controller** begins deleting all resources inside: Pods, Deployments, Services, ConfigMaps, Secrets, etc.
4. Pods with finalizers (`foregroundDeletion`, custom finalizers from operators) block deletion until their finalizers are removed.
5. Resources with finalizers owned by CRDs that no longer exist can block the namespace indefinitely — this is the most common hang scenario.
6. Only after ALL resources are gone does the API server remove the namespace object from etcd.

### ResourceQuota Admission Webhook Flow

```
kubectl apply (Pod/Deployment)
    ↓
API Server → Admission Controllers
    ↓
ResourceQuota admission plugin checks:
  namespace quota object → current usage + requested <= limit?
  │
  ├── Yes → proceed to etcd write
  └── No  → reject with 403 Forbidden: exceeded quota
```

---

## 4. Hands-On

> *Production-quality YAML + commands — nothing simplified.*

### Namespace Creation

```yaml
# app1-ns.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: app1-ns
  labels:
    team: platform
    env: production
    cost-center: "1042"
```

```bash
kubectl apply -f app1-ns.yaml

# Set as default for current context (eliminates -n flag for session)
kubectl config set-context --current --namespace=app1-ns

# Verify
kubectl config view --minify | grep namespace:
```

### Backend Deployment + ClusterIP Service (Namespace-Aware)

```yaml
# backend-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-deploy
  namespace: app1-ns
  labels:
    app: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend-container
          image: hashicorp/http-echo
          args:
            - "-text=Hello from Backend"
            - "-listen=:5678"
          ports:
            - containerPort: 5678
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "200m"
              memory: "128Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
  namespace: app1-ns
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - protocol: TCP
      port: 9090
      targetPort: 5678
```

### Frontend Deployment + NodePort Service (Namespace-Aware)

```yaml
# frontend-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deploy
  namespace: app1-ns
  labels:
    app: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend-container
          image: nginx
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "200m"
              memory: "128Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
  namespace: app1-ns
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 31000
```

### ResourceQuota (Production Must-Have)

```yaml
# app1-ns-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: app1-quota
  namespace: app1-ns
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
    pods: "20"
    services: "10"
    persistentvolumeclaims: "5"
```

### LimitRange (Default Container Limits)

```yaml
# app1-ns-limitrange.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: app1-limitrange
  namespace: app1-ns
spec:
  limits:
    - type: Container
      default:
        cpu: "200m"
        memory: "256Mi"
      defaultRequest:
        cpu: "50m"
        memory: "64Mi"
      max:
        cpu: "2"
        memory: "4Gi"
```

### Apply and Verify

```bash
kubectl apply -f app1-ns.yaml
kubectl apply -f app1-ns-quota.yaml
kubectl apply -f app1-ns-limitrange.yaml
kubectl apply -f backend-deploy.yaml
kubectl apply -f frontend-deploy.yaml

# Verify all resources in namespace
kubectl get all -n app1-ns

# Verify quota consumption
kubectl describe resourcequota app1-quota -n app1-ns

# Verify endpoints (confirms selector is working)
kubectl get endpoints -n app1-ns
```

### Connectivity Tests — Same and Cross Namespace

```bash
# Test 1: Same-namespace short-name resolution (should work)
kubectl run test-pod -n app1-ns --image=busybox --rm -it --restart=Never \
  -- wget -qO- http://backend-svc:9090
# Expected: Hello from Backend

# Test 2: Cross-namespace short name (should FAIL)
kubectl run test-pod -n default --image=busybox --rm -it --restart=Never \
  -- wget -qO- http://backend-svc:9090
# Expected: wget: bad address 'backend-svc'

# Test 3: Cross-namespace FQDN (should work)
kubectl run test-pod -n default --image=busybox --rm -it --restart=Never \
  -- wget -qO- http://backend-svc.app1-ns.svc.cluster.local:9090
# Expected: Hello from Backend

# Test 4: External NodePort access
curl http://<NodeIP>:31000
# Expected: nginx welcome page

# Test 5: Inspect DNS search domains injected into a pod
kubectl exec -n app1-ns <any-pod> -- cat /etc/resolv.conf
```

---

## 5. Production Flow

> *Real-world architecture and design patterns beyond the exam.*

### Multi-Namespace Architecture Pattern

```
Cluster
│
├── ingress-ns          ← Ingress controller only (nginx/traefik)
│     └── ingress-svc (LoadBalancer) ← single external IP
│
├── monitoring-ns       ← Prometheus, Grafana, Alertmanager
│
├── payments-prod-ns    ← Production: strict quotas, NetworkPolicy, PodSecurityPolicy
│     ├── backend-deploy
│     ├── backend-svc (ClusterIP)
│     └── ResourceQuota + LimitRange + NetworkPolicy
│
├── payments-staging-ns ← Staging: relaxed quotas, no prod data
│
└── payments-dev-ns     ← Dev: permissive, ephemeral
```

### NetworkPolicy — Enforcing Namespace Isolation

Without this, namespace isolation is purely logical:

```yaml
# Deny all ingress to app1-ns except from within app1-ns
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-cross-namespace
  namespace: app1-ns
spec:
  podSelector: {}       # applies to ALL pods in app1-ns
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector: {} # allow from pods in same namespace only
```

```yaml
# Allow ingress-ns to reach app1-ns (for Ingress controller passthrough)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-ingress
  namespace: app1-ns
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-ns
```

### RBAC Scoped to Namespace (Least Privilege)

```yaml
# Developer can read/write in app1-ns only
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app1-developer
  namespace: app1-ns
rules:
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app1-developer-binding
  namespace: app1-ns
subjects:
  - kind: User
    name: alice
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: app1-developer
  apiGroup: rbac.authorization.k8s.io
```

### Cross-Namespace Service Access Pattern

In production, avoid coupling namespaces directly. The recommended pattern:

```
Option A: FQDN (simple, no extra objects)
  frontend in app1-ns → backend-svc.app2-ns.svc.cluster.local:9090

Option B: ExternalName Service (hides FQDN behind local alias)
  apiVersion: v1
  kind: Service
  metadata:
    name: backend-alias      # local name in app1-ns
    namespace: app1-ns
  spec:
    type: ExternalName
    externalName: backend-svc.app2-ns.svc.cluster.local
  # Now frontend uses: curl http://backend-alias:9090
```

---

## 6. Mistakes

> *What actually breaks in real systems — root cause + fix.*

### Mistake 1: Namespace Stuck in `Terminating`

**Symptom:** `kubectl get namespace app1-ns` shows `Terminating` indefinitely.

**Root cause:** A resource inside the namespace has a finalizer that cannot be satisfied — typically a CRD-based resource whose controller is no longer running, or a PVC with a `kubernetes.io/pvc-protection` finalizer with a pod still mounting it.

**Fix:**
```bash
# Identify what's blocking
kubectl api-resources --verbs=list --namespaced -o name | \
  xargs -I{} kubectl get {} -n app1-ns --ignore-not-found 2>/dev/null

# Force-remove finalizer from a stuck resource (DANGEROUS — data loss risk)
kubectl patch <resource-type> <name> -n app1-ns \
  -p '{"metadata":{"finalizers":[]}}' --type=merge

# Nuclear option: remove namespace finalizer directly via API
kubectl get namespace app1-ns -o json \
  | jq '.spec.finalizers = []' \
  | kubectl replace --raw "/api/v1/namespaces/app1-ns/finalize" -f -
```

---

### Mistake 2: Cross-Namespace DNS Fails Silently

**Symptom:** `curl backend-svc:9090` from a different namespace returns `connection refused` or `name not resolved`, but the service is running.

**Root cause:** Short-name DNS resolution uses the pod's own namespace search domain. `backend-svc` expands to `backend-svc.default.svc.cluster.local`, which doesn't exist.

**Fix:** Always use FQDN for cross-namespace communication:
```bash
curl http://backend-svc.app1-ns.svc.cluster.local:9090
```

---

### Mistake 3: ResourceQuota Blocks Deployment Silently

**Symptom:** `kubectl apply -f deployment.yaml` succeeds, but pods never appear. `kubectl get pods` shows nothing.

**Root cause:** The Deployment and ReplicaSet are created (no quota on those object counts), but Pod creation is blocked by ResourceQuota. The failure is on the ReplicaSet controller, not the apply command.

**Diagnosis:**
```bash
kubectl describe replicaset <rs-name> -n app1-ns
# Look for: "exceeded quota" in Events
kubectl describe resourcequota -n app1-ns
```

**Fix:** Either increase the quota or reduce pod resource requests.

---

### Mistake 4: Namespace Field Omitted — Resources Land in Wrong Namespace

**Symptom:** Applied a manifest but can't find the resource. It exists — in the wrong namespace.

**Root cause:** `namespace:` field missing from manifest metadata. Resource went to the current context's default namespace.

**Fix:**
```bash
# Find the orphaned resource
kubectl get <resource> --all-namespaces | grep <name>
# Always set namespace in metadata when writing namespace-scoped YAMLs
# AND always verify context before applying
kubectl config view --minify | grep namespace:
```

---

### Mistake 5: LimitRange Missing → ResourceQuota Blocks All Pods

**Symptom:** After setting a ResourceQuota on CPU/memory, all new pods fail with `must specify limits`.

**Root cause:** ResourceQuota with `limits.cpu` or `limits.memory` requires every pod to have explicit resource limits. Pods without limits are rejected by admission. If developers don't set limits, all deployments fail.

**Fix:** Always pair a ResourceQuota with a LimitRange that provides defaults:
```yaml
# LimitRange sets defaults so pods without explicit limits still pass quota admission
spec:
  limits:
    - type: Container
      default:
        cpu: "200m"
        memory: "256Mi"
      defaultRequest:
        cpu: "50m"
        memory: "64Mi"
```

---

### Mistake 6: Deleting Namespace Instead of Resources

**Symptom:** Teammate runs `kubectl delete namespace app1-ns` thinking it clears the deployments. Entire namespace is wiped including Services, ConfigMaps, Secrets, PVCs.

**Root cause:** Namespace deletion cascades to all contained resources. No confirmation prompt.

**Fix / Prevention:**
```bash
# Use RBAC to prevent namespace deletion for non-admins
# Developers should only have Role (namespace-scoped), never ClusterRole with namespace deletion

# To clear all deployments without deleting namespace:
kubectl delete deployments --all -n app1-ns
```

---

## 7. Interview Answers

> *Compressed, verbatim-ready answers. Practice speaking these aloud.*

---

**Q: What is a Kubernetes namespace and what does it actually isolate?**

"A namespace is a virtual partition inside a Kubernetes cluster that scopes names, RBAC, and resource quotas. What it actually isolates depends on what additional controls you layer on top. By itself, a namespace provides name uniqueness — two Deployments can share the same name in different namespaces — and it scopes DNS short-name resolution. What it does NOT provide on its own is network isolation: pods across namespaces can communicate freely at the IP level unless you enforce NetworkPolicies. So I'd say a namespace is an organizational and access-control boundary, not a security perimeter by itself."

---

**Q: How does DNS work across namespaces in Kubernetes?**

"Every pod gets a `/etc/resolv.conf` injected by kubelet that points to CoreDNS and includes search domains for the pod's own namespace — something like `<pod-namespace>.svc.cluster.local`. When a pod resolves a short name like `backend-svc`, CoreDNS tries appending those search domains in order. So from within the same namespace, the short name resolves successfully. From a different namespace, that expansion fails because the service doesn't exist in that namespace. Cross-namespace access requires the full DNS name: `backend-svc.app1-ns.svc.cluster.local`. This is a very common failure mode — people assume DNS works cluster-wide with short names, but it's namespace-scoped by default."

---

**Q: What's the difference between a Role and a ClusterRole, and when would you use each?**

"A Role is namespace-scoped — it grants permissions to resources within one specific namespace. A ClusterRole is cluster-scoped — it can grant permissions to namespace-scoped resources across all namespaces, or to cluster-scoped resources like Nodes or PersistentVolumes that have no namespace. You bind a Role with a RoleBinding, and a ClusterRole with either a ClusterRoleBinding for cluster-wide access, or a RoleBinding to scope a ClusterRole to a single namespace. In practice, most developers get a Role in their own namespace. SREs might get a ClusterRole bound via RoleBinding to each namespace they manage. Cluster administrators get a ClusterRoleBinding."

---

**Q: What happens when you delete a namespace and why can it get stuck?**

"When you delete a namespace, Kubernetes sets its phase to Terminating and begins cascading deletion of every resource inside — Pods, Services, ConfigMaps, Secrets, everything. The namespace object itself is only removed from etcd after all contained resources are fully deleted. It gets stuck when a resource inside has a finalizer that cannot be resolved. The most common causes are CRD-based resources whose controller is no longer running, or PVCs still mounted by running pods. When this happens, you need to identify the stuck resources using kubectl and manually remove their finalizers — which you should treat as a last resort since finalizers usually protect against data loss."

---

**Q: What is a ResourceQuota and how does it interact with a LimitRange?**

"A ResourceQuota caps the total consumption of resources within a namespace — total CPU, total memory, total number of Pods, Services, and so on. A LimitRange, on the other hand, operates at the individual container level: it sets minimum and maximum values and can inject defaults. They're complementary. The key interaction to know is this: if you set a ResourceQuota with `limits.cpu`, the API server will reject any pod that doesn't explicitly declare a CPU limit. So if you set a quota without a LimitRange providing default limits, every developer who forgets to set limits gets a rejected deployment. Best practice is always to pair a ResourceQuota with a LimitRange."

---

**Q: How would you prevent one team's namespace from affecting another team's workloads?**

"I'd apply three layers. First, ResourceQuotas on each namespace to prevent any one team from consuming all cluster resources. Second, LimitRanges to ensure every pod has resource requests and limits set. Third, NetworkPolicies to explicitly define which namespaces can communicate with which — by default, all cross-namespace pod traffic is permitted. I'd also apply RBAC so each team can only operate within their own namespace and cannot read Secrets or modify resources in another team's namespace. On the node level, for harder multi-tenancy, I'd consider node taints and tolerations or node selectors to physically separate workloads."

---

## 8. Debugging

> *Fast diagnosis paths — not a list of commands, but an actual decision tree.*

### Cross-Namespace DNS Failure — Diagnosis Path

```
Pod cannot reach a service in another namespace
│
├── Step 1: Confirm the service exists and has endpoints
│     kubectl get svc -n <target-ns>
│     kubectl get endpoints -n <target-ns>
│     → No endpoints → label selector mismatch (separate issue)
│     → Has endpoints → DNS problem, continue
│
├── Step 2: Test DNS resolution from the failing pod
│     kubectl exec -n <source-ns> <pod> -- nslookup backend-svc
│     → NXDOMAIN → expected, short name won't resolve cross-namespace
│
│     kubectl exec -n <source-ns> <pod> -- \
│       nslookup backend-svc.app1-ns.svc.cluster.local
│     → Resolves → use FQDN in application config
│     → NXDOMAIN → CoreDNS issue (go to Step 3)
│
├── Step 3: Verify CoreDNS is healthy
│     kubectl get pods -n kube-system -l k8s-app=kube-dns
│     kubectl logs -n kube-system <coredns-pod>
│     → CrashLoopBackOff → CoreDNS config error or OOMKilled
│
└── Step 4: Check /etc/resolv.conf in the failing pod
      kubectl exec -n <source-ns> <pod> -- cat /etc/resolv.conf
      → nameserver should be CoreDNS ClusterIP
      → search domains should include pod's own namespace
      kubectl get svc -n kube-system kube-dns
      → Confirm ClusterIP matches nameserver in resolv.conf
```

### Namespace Stuck Terminating — Diagnosis Path

```
kubectl get namespace app1-ns → STATUS: Terminating (for > 5 minutes)
│
├── Step 1: Find what's still inside
│     kubectl get all -n app1-ns
│     → If empty → finalizer on namespace object itself (go to Step 3)
│     → Resources still present → go to Step 2
│
├── Step 2: Find resources with finalizers
│     kubectl get <resource> -n app1-ns -o yaml | grep -A5 finalizers
│     Common culprits:
│       PVC with kubernetes.io/pvc-protection → pod still mounting it
│       Custom resources from operators → controller not running
│
├── Step 3: Force-remove finalizers from blocking resources
│     kubectl patch <type> <name> -n app1-ns \
│       -p '{"metadata":{"finalizers":[]}}' --type=merge
│
└── Step 4: If namespace object itself has finalizer
      kubectl get namespace app1-ns -o json \
        | jq '.spec.finalizers = []' \
        | kubectl replace --raw "/api/v1/namespaces/app1-ns/finalize" -f -
```

### ResourceQuota Blocking Pods — Diagnosis Path

```
kubectl apply -f deploy.yaml → success, but no pods appear
│
├── Step 1: Check ReplicaSet events (not Pod events — pods don't exist yet)
│     kubectl describe rs -n app1-ns
│     → "exceeded quota" in Events → quota problem confirmed
│
├── Step 2: Check quota usage
│     kubectl describe resourcequota -n app1-ns
│     → Which resource is at limit? (cpu, memory, pods, etc.)
│
├── Step 3: Options
│     a) Increase quota → kubectl edit resourcequota app1-quota -n app1-ns
│     b) Reduce pod resource requests in Deployment spec
│     c) Delete other workloads to free quota
│
└── Step 4: If missing limits are the problem
      kubectl describe replicaset <n> -n app1-ns
      → "must specify limits" error
      → Apply LimitRange with defaults to the namespace
```

### Quick Diagnostic Commands

```bash
# List all namespaces and their status
kubectl get namespaces

# All resources across all namespaces (dangerous on large clusters — use with label filter)
kubectl get all --all-namespaces

# Find a resource when you don't know which namespace it's in
kubectl get pods --all-namespaces | grep <name>

# Check current context namespace
kubectl config view --minify | grep namespace:

# Describe quota consumption
kubectl describe resourcequota -n app1-ns

# Check what DNS name a pod would use
kubectl exec -n <ns> <pod> -- cat /etc/resolv.conf

# Check CoreDNS logs for DNS errors
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50

# List all cluster-scoped resources (useful when debugging "namespace not found" errors)
kubectl api-resources --namespaced=false

# List namespace-scoped resources for a given namespace
kubectl api-resources --namespaced=true --verbs=list -o name | \
  xargs -I{} kubectl get {} -n app1-ns --ignore-not-found
```

---

## 9. Kill Switch

> *10-second recall — the absolute minimum to hold in memory under exam pressure.*

```
NAMESPACE = name scope + DNS scope + quota scope + RBAC scope
  NOT a network boundary (need NetworkPolicy for that)

DEFAULT NAMESPACES:
  default, kube-system, kube-public, kube-node-lease

DNS RULE:
  same-ns   → short name works    (backend-svc:9090)
  cross-ns  → FQDN required       (backend-svc.app1-ns.svc.cluster.local:9090)

CLUSTER-SCOPED (no namespace): Node, PV, StorageClass, ClusterRole, Namespace itself

SET DEFAULT NAMESPACE:
  kubectl config set-context --current --namespace=app1-ns

QUOTA + LIMITRANGE must go together → quota needs limits → limitrange provides defaults

STUCK TERMINATING → find finalizers → patch them out → nuclear: replace finalize API

VERIFY CONTEXT BEFORE EVERY APPLY:
  kubectl config view --minify | grep namespace:
```

---

## 10. Appendix

### Quick Reference: Default Namespaces

| Namespace | Purpose | Touch in Production? |
|---|---|---|
| `default` | Catch-all for unscoped resources | Avoid — use named namespaces |
| `kube-system` | Control plane components (coredns, kube-proxy, etcd) | Read-only; treat as immutable |
| `kube-public` | Cluster info readable by all (incl. unauthenticated) | Read-only |
| `kube-node-lease` | Node heartbeat Lease objects for faster failure detection | Never |
| `local-path-storage` | KIND-specific local PV provisioner | KIND only |

### Quick Reference: Namespace Commands

```bash
# Create namespace imperatively
kubectl create namespace app1-ns

# Create with YAML
kubectl apply -f namespace.yaml

# Set default namespace for session
kubectl config set-context --current --namespace=app1-ns

# Verify current default namespace
kubectl config view --minify | grep namespace:

# Get all resources in a namespace
kubectl get all -n app1-ns

# Get all resources across all namespaces
kubectl get all --all-namespaces

# Delete namespace (cascades to all resources inside)
kubectl delete namespace app1-ns

# Get namespace labels (useful for NetworkPolicy namespaceSelector)
kubectl get namespace app1-ns --show-labels

# Add label to namespace (required for namespaceSelector in NetworkPolicy)
kubectl label namespace app1-ns team=platform

# List all namespaced resource types
kubectl api-resources --namespaced=true -o name

# List all cluster-scoped resource types
kubectl api-resources --namespaced=false -o name
```

### Quick Reference: DNS Name Formats

| From | To | Format | Works? |
|---|---|---|---|
| same namespace | same namespace | `<service>:<port>` | ✅ |
| same namespace | same namespace | `<service>.<namespace>.svc.cluster.local:<port>` | ✅ |
| different namespace | target namespace | `<service>:<port>` | ❌ |
| different namespace | target namespace | `<service>.<namespace>:<port>` | ✅ |
| different namespace | target namespace | `<service>.<namespace>.svc.cluster.local:<port>` | ✅ |
| external | cluster service | N/A (use Ingress or LoadBalancer) | — |

### YAML Skeleton: Namespace with Quota + LimitRange

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: app1-ns
  labels:
    team: platform
    env: production
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: app1-quota
  namespace: app1-ns
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
    pods: "20"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: app1-limitrange
  namespace: app1-ns
spec:
  limits:
    - type: Container
      default:
        cpu: "200m"
        memory: "256Mi"
      defaultRequest:
        cpu: "50m"
        memory: "64Mi"
      max:
        cpu: "2"
        memory: "4Gi"
```

### Common Gotchas Cheatsheet

| Gotcha | Rule |
|---|---|
| Short-name DNS cross-namespace | Always use FQDN: `<svc>.<ns>.svc.cluster.local` |
| ResourceQuota without LimitRange | Always pair them — quota requires limits; LimitRange provides defaults |
| Namespace field on cluster-scoped resource | Will be rejected by API server with schema error |
| `kubectl apply` without checking context | Run `kubectl config view --minify \| grep namespace:` first |
| Namespace `Terminating` hang | Find and remove finalizers; use the `/finalize` API as last resort |
| Deleting namespace vs deleting resources | `kubectl delete ns` cascades everything — usually unintended |
| NetworkPolicy not blocking cross-ns traffic | Default: all traffic allowed. Must explicitly deny with `podSelector: {}` |
| `--all-namespaces` on large cluster | Can time out or overload API server; add label filter: `-l app=backend` |
