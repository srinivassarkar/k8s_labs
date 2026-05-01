# Lab 06 — RBAC

## What This Lab Is About

RBAC (Role-Based Access Control) is how Kubernetes controls who can do what in the cluster. Every `kubectl` command you run, every API call a pod makes to the Kubernetes API server — all of it goes through RBAC. Get it wrong in one direction and you have a security hole. Get it wrong in the other direction and your apps can't function, your CI/CD pipeline breaks, and you spend an hour chasing a `403 Forbidden` that has nothing to do with your application code.

This lab covers the 4 most critical RBAC failure modes in real clusters. A pod that can't talk to the Kubernetes API because its ServiceAccount has no permissions. A `403 Forbidden` error with no obvious cause. Overly permissive RBAC that creates a security risk. And the correct production RBAC pattern — least privilege, scoped to exactly what each workload needs.

> In prod, RBAC errors don't announce themselves. They show up as mysterious application failures, broken operators, and CI/CD pipelines that silently do nothing.

---

## Environment

- **Cluster:** KIND (Kubernetes IN Docker)
- **K8s Version:** v1.27.3
- **Node:** Single node (`kind-control-plane`)
- **Namespace for this lab:** `lab06`

```bash
kubectl create namespace lab06
kubectl config set-context --current --namespace=lab06
```

---

## The RBAC Object Map — Know This Before Starting

```
ServiceAccount
  └── Represents an identity for a pod or process
  └── Every pod runs as a ServiceAccount (default if not specified)
  └── Lives in a namespace

Role
  └── A set of permissions (rules) scoped to ONE namespace
  └── Defines: which verbs on which resources are allowed

ClusterRole
  └── Same as Role but cluster-wide (all namespaces + cluster-level resources)
  └── Can also be used in a single namespace via RoleBinding

RoleBinding
  └── Connects a ServiceAccount (or User/Group) to a Role or ClusterRole
  └── Scoped to ONE namespace

ClusterRoleBinding
  └── Connects a ServiceAccount to a ClusterRole
  └── Applies cluster-wide — use carefully
```

```
The permission chain:

Pod → runs as → ServiceAccount
                     │
                     └── bound by → RoleBinding → Role → rules (verb + resource)

If the chain is broken at any point: 403 Forbidden
```

---

## The 4 Scenarios

| # | Scenario | What You'll Learn |
|---|---|---|
| 01 | Pod can't talk to K8s API — no permissions | ServiceAccount, Role, RoleBinding from scratch |
| 02 | 403 Forbidden — wrong verb or resource | Diagnosing and fixing permission gaps |
| 03 | Overly permissive RBAC — security risk | Why `cluster-admin` on a pod is dangerous |
| 04 | Production RBAC pattern — least privilege | The correct way to structure RBAC for real workloads |

---

## Scenario 01 — Pod Can't Talk to K8s API — No Permissions

### What You'll Break

A pod that needs to list pods in its namespace (common in operators, CI runners, monitoring agents). It uses the default ServiceAccount which has no permissions by default. Every API call returns `403 Forbidden`. The application logs show permission errors and the feature that depends on it silently fails.

### Apply the Broken State

```bash
# Deploy a pod that tries to list pods via the K8s API
# Uses kubectl inside the container to simulate an app calling the API
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: api-caller
  namespace: lab06
spec:
  serviceAccountName: default     # default SA — has no permissions
  containers:
  - name: app
    image: bitnami/kubectl:latest
    command: ["sh", "-c"]
    args:
    - |
      echo "Attempting to list pods..."
      kubectl get pods -n lab06
      echo "Exit code: $?"
      sleep 3600
EOF
```

```bash
# Wait for pod to start
kubectl get pod api-caller -n lab06

# Check the logs — see the 403
kubectl logs api-caller -n lab06
# Attempting to list pods...
# Error from server (Forbidden): pods is forbidden:
# User "system:serviceaccount:lab06:default" cannot list resource "pods"
# in API group "" in the namespace "lab06"
# Exit code: 1
```

The error message is very specific — it tells you exactly:
- **Who** is being denied: `system:serviceaccount:lab06:default`
- **What** they tried to do: `list` the `pods` resource
- **Where** they tried to do it: namespace `lab06`

### Investigate

```bash
# Step 1 — Check what ServiceAccount the pod is using
kubectl get pod api-caller -n lab06 \
  -o jsonpath='{.spec.serviceAccountName}'
# default

# Step 2 — Check what permissions the default SA has
kubectl auth can-i list pods \
  --as=system:serviceaccount:lab06:default \
  -n lab06
# no

# Step 3 — Check all permissions for this SA
kubectl auth can-i --list \
  --as=system:serviceaccount:lab06:default \
  -n lab06
# Resources  Non-Resource URLs  Resource Names  Verbs
# (very limited — mostly just self-info)

# Step 4 — Check if any RoleBindings exist for the default SA
kubectl get rolebindings -n lab06
kubectl get clusterrolebindings | grep lab06
# Likely nothing relevant
```

### Fix — Create ServiceAccount + Role + RoleBinding

```bash
# Step 1 — Create a dedicated ServiceAccount (don't use default in prod)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pod-reader-sa
  namespace: lab06
EOF

# Step 2 — Create a Role with exactly the permissions needed
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: lab06
rules:
- apiGroups: [""]           # "" means core API group (pods, services, configmaps)
  resources: ["pods"]       # Which resources
  verbs: ["get", "list", "watch"]  # Which actions
EOF

# Step 3 — Bind the Role to the ServiceAccount
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: lab06
subjects:
- kind: ServiceAccount
  name: pod-reader-sa       # The SA being granted permissions
  namespace: lab06
roleRef:
  kind: Role
  name: pod-reader          # The Role being granted
  apiGroup: rbac.authorization.k8s.io
EOF

# Step 4 — Verify the permission is now granted
kubectl auth can-i list pods \
  --as=system:serviceaccount:lab06:pod-reader-sa \
  -n lab06
# yes

# Step 5 — Redeploy the pod with the new ServiceAccount
kubectl delete pod api-caller -n lab06

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: api-caller
  namespace: lab06
spec:
  serviceAccountName: pod-reader-sa   # Fixed: use dedicated SA
  containers:
  - name: app
    image: bitnami/kubectl:latest
    command: ["sh", "-c"]
    args:
    - |
      echo "Attempting to list pods..."
      kubectl get pods -n lab06
      echo "Exit code: $?"
      sleep 3600
EOF

# Check logs — should work now
kubectl logs api-caller -n lab06
# Attempting to list pods...
# NAME         READY   STATUS    RESTARTS   AGE
# api-caller   1/1     Running   0          10s
# Exit code: 0
```

### How the Token Gets Into the Pod

```bash
# K8s automatically mounts the SA token into every pod
kubectl exec api-caller -n lab06 -- \
  ls /var/run/secrets/kubernetes.io/serviceaccount/
# ca.crt    namespace    token

# The token is what the app (or kubectl inside the pod) uses to authenticate
# kubectl inside the pod uses it automatically via KUBECONFIG env or in-cluster config
kubectl exec api-caller -n lab06 -- \
  cat /var/run/secrets/kubernetes.io/serviceaccount/namespace
# lab06
```

### Prod Wisdom

**Never use the `default` ServiceAccount for anything that calls the Kubernetes API.** Create a dedicated ServiceAccount per application role. The `default` SA is shared by all pods in a namespace — granting it permissions means every pod in the namespace gets those permissions, including ones you didn't intend. One ServiceAccount per workload type, scoped to exactly what it needs.

---

## Scenario 02 — 403 Forbidden — Wrong Verb or Resource

### What You'll Break

A ServiceAccount that has some permissions but not the specific one the app needs. This is subtler than Scenario 01 — the SA exists, the RoleBinding exists, but the Role is missing a verb or a resource. The application works partially and fails in specific code paths. These are the hardest RBAC bugs to find.

### Apply the Broken State

```bash
# Create a SA with a Role that's missing some verbs
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: configmap-manager-sa
  namespace: lab06
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: configmap-manager
  namespace: lab06
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]      # Missing: "create", "update", "delete"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: configmap-manager-binding
  namespace: lab06
subjects:
- kind: ServiceAccount
  name: configmap-manager-sa
  namespace: lab06
roleRef:
  kind: Role
  name: configmap-manager
  apiGroup: rbac.authorization.k8s.io
EOF

# Deploy a pod that needs to GET and CREATE configmaps
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: configmap-worker
  namespace: lab06
spec:
  serviceAccountName: configmap-manager-sa
  containers:
  - name: worker
    image: bitnami/kubectl:latest
    command: ["sh", "-c"]
    args:
    - |
      echo "--- Test: list configmaps ---"
      kubectl get configmaps -n lab06
      echo ""
      echo "--- Test: create configmap ---"
      kubectl create configmap test-config \
        --from-literal=key=value -n lab06
      echo ""
      echo "--- Test: delete configmap ---"
      kubectl delete configmap test-config -n lab06
      sleep 3600
EOF
```

```bash
kubectl logs configmap-worker -n lab06
# --- Test: list configmaps ---
# NAME               DATA   AGE
# kube-root-ca.crt   1      5m
# (success)

# --- Test: create configmap ---
# Error from server (Forbidden): configmaps is forbidden:
# User "system:serviceaccount:lab06:configmap-manager-sa"
# cannot create resource "configmaps" in API group ""
# in the namespace "lab06"

# --- Test: delete configmap ---
# (never reached)
```

List works. Create fails. The app partially works — making this extremely confusing without knowing to look at RBAC.

### Investigate

```bash
# Step 1 — Read the error message carefully
# "cannot create resource configmaps" — verb is "create"

# Step 2 — Check current permissions for this SA
kubectl auth can-i --list \
  --as=system:serviceaccount:lab06:configmap-manager-sa \
  -n lab06
# Resources    Non-Resource URLs  Resource Names  Verbs
# configmaps   []                 []              [get list]
# Only get and list — create, update, delete are missing

# Step 3 — Test specific permissions
kubectl auth can-i get configmaps \
  --as=system:serviceaccount:lab06:configmap-manager-sa \
  -n lab06
# yes

kubectl auth can-i create configmaps \
  --as=system:serviceaccount:lab06:configmap-manager-sa \
  -n lab06
# no

kubectl auth can-i delete configmaps \
  --as=system:serviceaccount:lab06:configmap-manager-sa \
  -n lab06
# no

# Step 4 — Look at the Role definition
kubectl describe role configmap-manager -n lab06
# PolicyRule:
#   Resources   Non-Resource URLs  Resource Names  Verbs
#   configmaps  []                 []              [get list]
# Missing verbs confirmed
```

### Fix — Update the Role

```bash
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: configmap-manager
  namespace: lab06
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
EOF

# Verify all permissions granted
kubectl auth can-i create configmaps \
  --as=system:serviceaccount:lab06:configmap-manager-sa \
  -n lab06
# yes

kubectl auth can-i delete configmaps \
  --as=system:serviceaccount:lab06:configmap-manager-sa \
  -n lab06
# yes

# Restart the pod to re-run the tests
kubectl delete pod configmap-worker -n lab06

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: configmap-worker
  namespace: lab06
spec:
  serviceAccountName: configmap-manager-sa
  containers:
  - name: worker
    image: bitnami/kubectl:latest
    command: ["sh", "-c"]
    args:
    - |
      echo "--- Test: list configmaps ---"
      kubectl get configmaps -n lab06
      echo ""
      echo "--- Test: create configmap ---"
      kubectl create configmap test-config \
        --from-literal=key=value -n lab06
      echo ""
      echo "--- Test: delete configmap ---"
      kubectl delete configmap test-config -n lab06
      echo ""
      echo "All tests passed."
      sleep 3600
EOF

kubectl logs configmap-worker -n lab06
# All three operations succeed
```

### The Verbs You Need to Know

```
READ operations:
  get    → retrieve a single resource by name
  list   → retrieve all resources of a type
  watch  → stream changes to resources (used by controllers)

WRITE operations:
  create → create a new resource
  update → replace a resource entirely
  patch  → partially update a resource
  delete → delete a single resource
  deletecollection → delete multiple resources at once

SPECIAL:
  use        → use a PodSecurityPolicy or StorageClass
  bind       → create RoleBindings (sensitive)
  escalate   → modify roles to have more permissions than you have (dangerous)
  impersonate → act as another user/SA (dangerous)
```

### Prod Wisdom

`kubectl auth can-i --list --as=system:serviceaccount:<namespace>:<name>` is the single most useful command for RBAC debugging. Run it immediately when you see a `403`. It shows you every permission the SA currently has in tabular form — the missing verb jumps out instantly. Keep this command in your muscle memory — it replaces 10 minutes of guessing with 10 seconds of clarity.

---

## Scenario 03 — Overly Permissive RBAC — Security Risk

### What You'll Break (and Understand)

The temptation when facing a `403` is to grant `cluster-admin` to make it go away. This is one of the most dangerous things you can do in a K8s cluster. A pod with `cluster-admin` can read all secrets in all namespaces, modify any resource, create new admin accounts, and essentially own the cluster. This scenario shows what that looks like and why it matters.

### Apply the Dangerous Pattern

```bash
# What many engineers do when they can't figure out RBAC
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dangerous-sa
  namespace: lab06
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: dangerous-cluster-admin
subjects:
- kind: ServiceAccount
  name: dangerous-sa
  namespace: lab06
roleRef:
  kind: ClusterRole
  name: cluster-admin          # Full cluster admin — every verb on every resource
  apiGroup: rbac.authorization.k8s.io
EOF
```

### Observe What This SA Can Do

```bash
# This SA can now do EVERYTHING
kubectl auth can-i create secrets \
  --as=system:serviceaccount:lab06:dangerous-sa \
  -n kube-system
# yes — can read ALL secrets in kube-system including service account tokens

kubectl auth can-i delete nodes \
  --as=system:serviceaccount:lab06:dangerous-sa
# yes — can delete nodes

kubectl auth can-i create clusterrolebindings \
  --as=system:serviceaccount:lab06:dangerous-sa
# yes — can grant itself or others any permission

kubectl auth can-i '*' '*' \
  --as=system:serviceaccount:lab06:dangerous-sa
# yes — literally everything
```

If a pod running with this SA is compromised:
- Attacker reads all Secrets across all namespaces (DB passwords, API keys, TLS certs)
- Attacker creates new ClusterRoleBindings to maintain access
- Attacker modifies or deletes other workloads
- Attacker can pivot to cloud provider credentials mounted in secrets

### The Audit — Finding Dangerous Bindings in Your Cluster

```bash
# Find all ClusterRoleBindings to cluster-admin
kubectl get clusterrolebindings -o json | \
  jq '.items[] | select(.roleRef.name=="cluster-admin") | 
  {name: .metadata.name, subjects: .subjects}'

# Find ServiceAccounts with cluster-admin
kubectl get clusterrolebindings -o json | \
  jq '.items[] | select(.roleRef.name=="cluster-admin") |
  .subjects[] | select(.kind=="ServiceAccount")'

# Find any SA with wildcard permissions
kubectl get roles,clusterroles -A -o json | \
  jq '.items[] | select(.rules[]?.verbs[]? == "*") | 
  {name: .metadata.name, namespace: .metadata.namespace}'
```

### Fix — Replace With Least Privilege

```bash
# Delete the dangerous binding
kubectl delete clusterrolebinding dangerous-cluster-admin
kubectl delete serviceaccount dangerous-sa -n lab06

# Replace with a properly scoped SA
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: scoped-sa
  namespace: lab06
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: scoped-role
  namespace: lab06
rules:
# Only what this workload actually needs
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "create", "update"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: scoped-binding
  namespace: lab06
subjects:
- kind: ServiceAccount
  name: scoped-sa
  namespace: lab06
roleRef:
  kind: Role
  name: scoped-role
  apiGroup: rbac.authorization.k8s.io
EOF

# Verify scoped permissions — yes to what it needs
kubectl auth can-i list pods \
  --as=system:serviceaccount:lab06:scoped-sa \
  -n lab06
# yes

# No to everything else
kubectl auth can-i list secrets \
  --as=system:serviceaccount:lab06:scoped-sa \
  -n lab06
# no

kubectl auth can-i list pods \
  --as=system:serviceaccount:lab06:scoped-sa \
  -n kube-system
# no — scoped to lab06 only
```

### Prod Wisdom

**`cluster-admin` on a pod ServiceAccount is a critical security vulnerability.** It violates the principle of least privilege and turns any pod compromise into a full cluster compromise. In a real prod audit, any ClusterRoleBinding that binds a ServiceAccount (not a human user) to `cluster-admin` is an immediate finding. The correct pattern: always start with no permissions, add only what the specific workload needs, scope to the specific namespace, use Role not ClusterRole unless cross-namespace access is truly required.

---

## Scenario 04 — Production RBAC Pattern — Least Privilege

### What You'll Build

A complete, production-grade RBAC setup for a real workload pattern — a deployment controller that needs to read deployments, manage configmaps in its own namespace, and read secrets it owns by name. This is the level of precision that proper RBAC requires and what you should aim for in every real workload.

### The Complete Production Setup

```bash
# Step 1 — Dedicated ServiceAccount per workload
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: deploy-controller-sa
  namespace: lab06
  labels:
    app: deploy-controller
  annotations:
    description: "SA for deploy-controller — manages configmaps and reads deployments"
EOF

# Step 2 — Scoped Role with only required permissions
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deploy-controller-role
  namespace: lab06
rules:
# Permission 1 — read deployments to watch for changes
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch"]

# Permission 2 — manage configmaps (its primary job)
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

# Permission 3 — read specific secrets by name only (not list all secrets)
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["deploy-controller-secret"]  # Only THIS secret, nothing else
  verbs: ["get"]

# Permission 4 — create events (for status reporting)
- apiGroups: [""]
  resources: ["events"]
  verbs: ["create", "patch"]
EOF

# Step 3 — RoleBinding in the correct namespace
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: deploy-controller-binding
  namespace: lab06
subjects:
- kind: ServiceAccount
  name: deploy-controller-sa
  namespace: lab06
roleRef:
  kind: Role
  name: deploy-controller-role
  apiGroup: rbac.authorization.k8s.io
EOF

# Step 4 — Create the secret it's allowed to read
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: deploy-controller-secret
  namespace: lab06
type: Opaque
stringData:
  api-key: "super-secret-key-123"
EOF
```

### Verify Every Permission Precisely

```bash
# What it CAN do
kubectl auth can-i list deployments \
  --as=system:serviceaccount:lab06:deploy-controller-sa \
  -n lab06
# yes

kubectl auth can-i create configmaps \
  --as=system:serviceaccount:lab06:deploy-controller-sa \
  -n lab06
# yes

kubectl auth can-i get secrets \
  --as=system:serviceaccount:lab06:deploy-controller-sa \
  -n lab06
# yes (but only for the named secret)

# What it CANNOT do
kubectl auth can-i list secrets \
  --as=system:serviceaccount:lab06:deploy-controller-sa \
  -n lab06
# no — list all secrets: denied

kubectl auth can-i delete deployments \
  --as=system:serviceaccount:lab06:deploy-controller-sa \
  -n lab06
# no — it can only read deployments, not modify them

kubectl auth can-i list deployments \
  --as=system:serviceaccount:lab06:deploy-controller-sa \
  -n kube-system
# no — Role scoped to lab06 only

kubectl auth can-i list nodes \
  --as=system:serviceaccount:lab06:deploy-controller-sa
# no — no cluster-level permissions
```

### Deploy and Verify the Workload

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-controller
  namespace: lab06
spec:
  replicas: 1
  selector:
    matchLabels:
      app: deploy-controller
  template:
    metadata:
      labels:
        app: deploy-controller
    spec:
      serviceAccountName: deploy-controller-sa   # Explicit SA
      automountServiceAccountToken: true         # Explicitly opt-in
      containers:
      - name: controller
        image: bitnami/kubectl:latest
        command: ["sh", "-c"]
        args:
        - |
          echo "=== Testing scoped permissions ==="
          echo ""
          echo "1. List deployments (allowed):"
          kubectl get deployments -n lab06
          echo ""
          echo "2. Create configmap (allowed):"
          kubectl create configmap controller-config \
            --from-literal=env=prod -n lab06 2>&1
          echo ""
          echo "3. Read specific secret (allowed):"
          kubectl get secret deploy-controller-secret -n lab06 2>&1
          echo ""
          echo "4. List all secrets (denied by design):"
          kubectl get secrets -n lab06 2>&1
          echo ""
          echo "5. Access kube-system (denied by design):"
          kubectl get pods -n kube-system 2>&1
          echo ""
          echo "=== Permission check complete ==="
          sleep 3600
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
EOF

kubectl logs $(kubectl get pod -n lab06 -l app=deploy-controller \
  -o jsonpath='{.items[0].metadata.name}') -n lab06
```

Expected output:
```
=== Testing scoped permissions ===

1. List deployments (allowed):
NAME                READY   UP-TO-DATE   AVAILABLE   AGE
deploy-controller   1/1     1            1           30s

2. Create configmap (allowed):
configmap/controller-config created

3. Read specific secret (allowed):
NAME                       TYPE     DATA   AGE
deploy-controller-secret   Opaque   1      2m

4. List all secrets (denied by design):
Error from server (Forbidden): secrets is forbidden: ...cannot list resource "secrets"

5. Access kube-system (denied by design):
Error from server (Forbidden): pods is forbidden: ...cannot list resource "pods" in the namespace "kube-system"

=== Permission check complete ===
```

Items 4 and 5 failing is **correct behaviour** — the workload is properly constrained.

### The Production RBAC Checklist

```
For every new workload that calls the K8s API:

□ Create a dedicated ServiceAccount — never use default
□ Create a Role (not ClusterRole) scoped to the workload's namespace
□ Grant only the verbs the workload actually uses
□ Use resourceNames to scope secret access where possible
□ Bind with RoleBinding (not ClusterRoleBinding) unless cross-namespace is required
□ Run kubectl auth can-i --list to verify granted permissions
□ Verify denied permissions are actually denied
□ Add description annotation to the ServiceAccount explaining its purpose
□ Disable automountServiceAccountToken on pods that don't need API access
```

### Disable Token Mounting for Pods That Don't Need It

```bash
# Most application pods don't need to call the K8s API at all
# Disable the token mount entirely for them
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: no-api-needed
  namespace: lab06
spec:
  automountServiceAccountToken: false   # No token mounted — most secure
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

# Verify no token is mounted
kubectl exec no-api-needed -n lab06 -- \
  ls /var/run/secrets/kubernetes.io/serviceaccount/ 2>&1
# ls: cannot access '/var/run/secrets/kubernetes.io/serviceaccount/': No such file or directory
```

### Prod Wisdom

Three RBAC patterns that define production-grade clusters:

**1. Dedicated SA per workload, never `default`.** The default SA is a shared identity — granting it any permissions affects every pod in the namespace. Create one SA per application role, name it meaningfully, annotate it with its purpose.

**2. `resourceNames` for secret access.** Instead of `verbs: ["get"] on resources: ["secrets"]` (which lets the SA get ANY secret), add `resourceNames: ["my-specific-secret"]`. This limits the blast radius if the SA is compromised to only the secrets it genuinely needs.

**3. `automountServiceAccountToken: false` on pods that don't call the API.** Most application pods — web servers, workers, databases — have no reason to call the Kubernetes API. Disable the token mount entirely. An attacker who compromises that pod cannot use the mounted token to escalate privileges within the cluster.

---

## Key Commands Reference — Lab 06

```bash
# The RBAC debug command — use this immediately when you see 403
kubectl auth can-i --list \
  --as=system:serviceaccount:<namespace>:<sa-name> \
  -n <namespace>

# Test a specific permission
kubectl auth can-i <verb> <resource> \
  --as=system:serviceaccount:<namespace>:<sa-name> \
  -n <namespace>

# Who can do what in a namespace
kubectl auth can-i list pods --as=system:serviceaccount:lab06:my-sa -n lab06

# Inspect Role rules
kubectl describe role <name> -n <namespace>
kubectl describe clusterrole <name>

# List all RoleBindings in a namespace
kubectl get rolebindings -n <namespace>
kubectl describe rolebinding <name> -n <namespace>

# List all ClusterRoleBindings
kubectl get clusterrolebindings

# Find what's bound to a ServiceAccount
kubectl get rolebindings -n <namespace> -o json | \
  jq '.items[] | select(.subjects[]?.name=="<sa-name>")'

# Audit: find all cluster-admin bindings
kubectl get clusterrolebindings -o json | \
  jq '.items[] | select(.roleRef.name=="cluster-admin") | .metadata.name'

# Check ServiceAccount token mount
kubectl get pod <name> -n <namespace> \
  -o jsonpath='{.spec.automountServiceAccountToken}'
```

---

## Prod Wisdom — The Senior Engineer Mindset

Three things that define senior-level RBAC thinking:

**1. They read the 403 error message completely.** The K8s API server tells you exactly who was denied, what verb they tried, and on which resource. `system:serviceaccount:lab06:default cannot create resource "secrets" in namespace "lab06"` gives you everything — the SA name, the missing verb, the resource, and the namespace. The fix is in the error message.

**2. They always use `kubectl auth can-i` before and after fixing.** Before: to confirm what's missing. After: to confirm the fix works AND that you haven't over-granted. Both directions matter — check that allowed is allowed AND that denied is still denied.

**3. They scope RBAC to the minimum viable permission surface.** Not just the right resource and verb — but also the right namespace (Role vs ClusterRole), and the right resource names for sensitive resources like Secrets. Every permission you grant is potential blast radius if that workload is compromised. Minimize it deliberately.

The RBAC debug order:
```
Read 403 message → identify SA + verb + resource
→ kubectl auth can-i --list (see what it has)
→ fix Role verbs or add missing Role rule
→ kubectl auth can-i (verify fix)
→ verify denied permissions still denied
```

---

## Cleanup

```bash
kubectl delete namespace lab06
```

---

*Lab 06 complete. Move to Lab 07 — Ingress when ready.*