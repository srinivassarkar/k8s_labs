# 🛡️ Kubernetes NetworkPolicy 

> *"By default, every pod trusts every pod. NetworkPolicies are how you fix that."*

---

## 0. First Principles 🧠

The mental model that never changes:

**Kubernetes networking has three immutable truths:**

1. **All pods can reach all pods by default** — no firewall, no isolation, flat L3 mesh.
2. **NetworkPolicy is additive and declarative** — you whitelist, never blacklist. Once a pod is *selected*, everything not explicitly allowed is denied.
3. **Kubernetes doesn't enforce policies** — it stores them. Enforcement is entirely the CNI plugin's job. No CNI support = policies are silently ignored.

**The mental model:**
```
No Policy Selected  →  All traffic allowed (default open)
Policy Selects Pod  →  All traffic DENIED except explicit allows
Multiple Policies   →  Rules are ADDITIVE (union of all allows)
```

**The statefulness rule:** If you allow egress from Pod A to Pod B on port 3306, the *reply* packets from B→A are automatically permitted. You never need to write a reverse rule for responses. Think AWS Security Groups, not NACLs.

---

## 1. Reality Constraints ⚠️

### What Kubernetes ACTUALLY Does

| Fact | Reality |
|------|---------|
| Default behavior | All pods can talk to all pods — zero isolation |
| Policy enforcement | Delegated 100% to CNI plugin |
| Policy scope | Layer 3 (IP) and Layer 4 (port/protocol) only |
| Statefulness | Stateful — responses automatically allowed |
| Multiple policies | Additive — all matching policies are unioned |
| Empty egress rules | Explicit `policyTypes: [Egress]` with no `egress:` = **deny ALL egress** |
| Node-to-pod traffic | Not controlled by NetworkPolicy by default |
| Inter-namespace | Allowed by default unless policies restrict it |

### CNIs That Support NetworkPolicy

| CNI | NetworkPolicy Support |
|-----|----------------------|
| **Calico** | ✅ Full support + GlobalNetworkPolicy extension |
| **Cilium** | ✅ Full support + L7 HTTP policies |
| **Antrea** | ✅ Full support |
| **AWS VPC CNI** | ✅ With aws-network-policy-agent |
| **OVN-Kubernetes** | ✅ Full support |
| **Flannel** | ❌ No support |
| **Kindnet** (KIND default) | ❌ No support |
| **Weave Net** | ❌ Deprecated/no support |

> 🚨 **Critical exam trap:** Applying a NetworkPolicy to a cluster with Flannel/Kindnet = the YAML applies but **zero enforcement happens**. The cluster won't warn you.

### What NetworkPolicy CANNOT Do

- Layer 7 (HTTP path/method) — needs Cilium or a service mesh
- Encrypt traffic — needs mTLS/WireGuard
- Control traffic between nodes themselves
- Override host-network pods (pods with `hostNetwork: true` bypass CNI policies)

---

## 2. Decision Logic 🗺️

### When to Use Which Selector

```
Traffic is from/to...
├── Same namespace, different pods?
│   └── podSelector only
├── Different namespace, all pods in it?
│   └── namespaceSelector only
├── Different namespace, specific pods in it?
│   └── namespaceSelector AND podSelector (single from: entry)
├── External IP range (outside cluster)?
│   └── ipBlock with CIDR
└── Combination of above?
    └── Multiple from:/to: entries (OR logic)
```

### AND vs OR — The Most Common Exam Gotcha

| Structure | Logic | Meaning |
|-----------|-------|---------|
| Two separate entries in `from:` | **OR** | Either condition satisfies the rule |
| `namespaceSelector` + `podSelector` in ONE `from:` entry | **AND** | BOTH must be true |

```yaml
# ❌ OR — too permissive
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: kube-system
  - podSelector:
      matchLabels:
        k8s-app: kube-dns

# ✅ AND — precisely scoped (CoreDNS only)
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: kube-system
    podSelector:          # ← same list item, not a new dash
      matchLabels:
        k8s-app: kube-dns
```

### policyTypes Decision Table

| `policyTypes` value | `ingress:` defined? | `egress:` defined? | Effect |
|---------------------|--------------------|--------------------|--------|
| `[Ingress]` | Yes | — | Ingress: allow listed; Egress: unrestricted |
| `[Egress]` | — | Yes | Egress: allow listed; Ingress: unrestricted |
| `[Ingress, Egress]` | Yes | Yes | Both directions restricted |
| `[Ingress, Egress]` | Yes | **No** | Ingress: allow listed; **Egress: DENY ALL** |
| Omitted | Yes, no egress rules | — | Defaults to `[Ingress]` |

> 💡 **Always declare `policyTypes` explicitly.** Relying on defaults is a production incident waiting to happen.

---

## 3. Internal Working ⚙️

### How a Packet Traverses the Policy Engine

1. **Pod A** initiates a TCP connection to **Pod B** on port 3306.
2. The packet leaves Pod A's `veth` pair into the host network namespace.
3. **CNI agent (e.g., Felix/Calico)** intercepts via `iptables`/`eBPF` hooks installed on the node.
4. Felix checks: *Does Pod A have any NetworkPolicy with `policyTypes: [Egress]`?*
   - **No policy** → packet passes through.
   - **Policy exists** → evaluate egress rules.
5. Felix evaluates each `to:` entry in the egress rules (OR between entries).
6. If a match is found → packet **allowed**; conntrack entry created (stateful).
7. Packet arrives at **Pod B's** node. Felix checks ingress rules for Pod B.
8. Both egress from A **and** ingress to B must allow the traffic — both must pass.
9. Return traffic from B → A: **conntrack match** → automatically allowed, no rule re-evaluation.

### How Calico Specifically Works

```
API Server → Calico kube-controllers → Calico datastore (CRDs in Kubernetes API)
                                                ↓
                                         Felix (per node)
                                                ↓
                                    iptables / eBPF rules on host
                                                ↓
                                       veth pair (pod interface)
```

- **Felix** = the per-node agent that translates NetworkPolicy into kernel rules
- **BIRD** = BGP daemon that handles pod route advertisement between nodes
- **calico-cni-plugin** = handles IP allocation (IPAM) at pod creation time
- **calico-kube-controllers** = watches K8s API and syncs state to Calico datastore

### The KIND + Calico Setup Flow

```
kind create cluster --config kind-cluster.yaml   # Creates nodes, NO CNI
  ↓
kubectl get nodes → NotReady (CNI plugin not initialized)
  ↓
kubectl apply -f calico.yaml                     # Installs Felix, BIRD, CNI binary
  ↓
calico-node DaemonSet pods come up               # Felix starts, installs iptables rules
  ↓
kubectl get nodes → Ready                        # Nodes can now route pod traffic
  ↓
NetworkPolicy YAML applied                       # Felix reads it, updates iptables/eBPF
```

---

## 4. Hands-On 🛠️

### KIND Cluster Config (Default CNI Disabled)

```yaml
# kind-cluster.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  image: kindest/node:v1.33.1@sha256:050072256b9a903bd914c0b2866828150cb229cea0efe5892e2b644d5dd3b34f
  extraPortMappings:
  - containerPort: 31000
    hostPort: 31000
- role: worker
  image: kindest/node:v1.33.1@sha256:050072256b9a903bd914c0b2866828150cb229cea0efe5892e2b644d5dd3b34f
- role: worker
  image: kindest/node:v1.33.1@sha256:050072256b9a903bd914c0b2866828150cb229cea0efe5892e2b644d5dd3b34f
networking:
  disableDefaultCNI: true   # Disable Kindnet; we'll install Calico
```

```bash
kind create cluster --config kind-cluster.yaml --name=cka-2025
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.2/manifests/calico.yaml
kubectl wait --for=condition=ready node --all --timeout=120s
```

### Namespace

```yaml
# 00-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: app1-ns
  labels:
    app: app1
    kubernetes.io/metadata.name: app1-ns  # Required for namespaceSelector matching
```

### Frontend (Deployment + NodePort Service)

```yaml
# 01-frontend.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deploy
  namespace: app1-ns
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app1
      role: frontend
  template:
    metadata:
      labels:
        app: app1
        role: frontend
    spec:
      containers:
      - name: frontend-container
        image: nginx:1.25-alpine
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
          limits:
            cpu: 200m
            memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
  namespace: app1-ns
spec:
  type: NodePort
  selector:
    app: app1
    role: frontend
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 31000
```

### Backend (Deployment + ClusterIP Service)

```yaml
# 02-backend.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-deploy
  namespace: app1-ns
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app1
      role: backend
  template:
    metadata:
      labels:
        app: app1
        role: backend
    spec:
      containers:
      - name: backend
        image: python:3.11-slim
        command: ["python3", "-m", "http.server", "5678"]
        ports:
        - containerPort: 5678
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
          limits:
            cpu: 200m
            memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
  namespace: app1-ns
spec:
  type: ClusterIP
  selector:
    app: app1
    role: backend
  ports:
  - protocol: TCP
    port: 5678
    targetPort: 5678
```

### Database (StatefulSet + Headless Service)

```yaml
# 03-db.yaml
apiVersion: v1
kind: Service
metadata:
  name: db-svc
  namespace: app1-ns
spec:
  clusterIP: None    # Headless — DNS returns pod IPs directly
  selector:
    app: app1
    role: db
  ports:
  - protocol: TCP
    port: 3306
    targetPort: 3306
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db-sts
  namespace: app1-ns
spec:
  serviceName: db-svc   # Must match headless service name
  replicas: 1
  selector:
    matchLabels:
      app: app1
      role: db
  template:
    metadata:
      labels:
        app: app1
        role: db
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        ports:
        - containerPort: 3306
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        resources:
          requests:
            cpu: 250m
            memory: 512Mi
          limits:
            cpu: 500m
            memory: 1Gi
```

### NetworkPolicy — DB Tier (Deny All, Allow Only Backend)

```yaml
# 04-db-netpol.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
  namespace: app1-ns
spec:
  podSelector:
    matchLabels:
      app: app1
      role: db
  policyTypes:
  - Ingress
  - Egress         # No egress rules defined = DENY ALL EGRESS
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: app1
          role: backend
    ports:
    - protocol: TCP
      port: 3306
  # No egress: block intentional — DB should never initiate outbound connections
```

### NetworkPolicy — Backend Tier (Frontend In, DB + DNS Out)

```yaml
# 05-backend-netpol.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: app1-ns
spec:
  podSelector:
    matchLabels:
      app: app1
      role: backend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: app1
          role: frontend
    ports:
    - protocol: TCP
      port: 5678
  egress:
  # Allow to DB pods
  - to:
    - podSelector:
        matchLabels:
          app: app1
          role: db
    ports:
    - protocol: TCP
      port: 3306
  # Allow DNS resolution — AND logic: must be kube-dns IN kube-system
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:                    # ← same list item = AND
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

### NetworkPolicy — Frontend Tier (External In, Backend + DNS Out)

```yaml
# 06-frontend-netpol.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-policy
  namespace: app1-ns
spec:
  podSelector:
    matchLabels:
      app: app1
      role: frontend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 0.0.0.0/0    # Restrict in prod to LB CIDR, e.g., 10.0.0.0/8
    ports:
    - protocol: TCP
      port: 80
  egress:
  # Allow to backend pods
  - to:
    - podSelector:
        matchLabels:
          app: app1
          role: backend
    ports:
    - protocol: TCP
      port: 5678
  # Allow DNS (AND logic)
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

### Verification Commands

```bash
# Apply everything
kubectl apply -f 00-namespace.yaml
kubectl config set-context --current --namespace=app1-ns
kubectl apply -f 01-frontend.yaml 02-backend.yaml 03-db.yaml
kubectl apply -f 04-db-netpol.yaml 05-backend-netpol.yaml 06-frontend-netpol.yaml

# Check all resources
kubectl get pods,svc,netpol -n app1-ns

# Test connectivity (before policies — should all succeed)
FE_POD=$(kubectl get pod -l role=frontend -o jsonpath='{.items[0].metadata.name}')
BE_POD=$(kubectl get pod -l role=backend -o jsonpath='{.items[0].metadata.name}')
DB_POD=$(kubectl get pod -l role=db -o jsonpath='{.items[0].metadata.name}')

# Frontend → Backend (should succeed after policy)
kubectl exec -it $FE_POD -- sh -c "apt-get install -y telnet -q && telnet backend-svc 5678"

# Frontend → DB (should FAIL after policy)
kubectl exec -it $FE_POD -- sh -c "telnet db-sts-0.db-svc.app1-ns.svc.cluster.local 3306"

# Backend → DB (should succeed after policy)
kubectl exec -it $BE_POD -- sh -c "apt-get install -y telnet -q && telnet db-sts-0.db-svc.app1-ns.svc.cluster.local 3306"

# Backend → Frontend (should FAIL after policy)
kubectl exec -it $BE_POD -- sh -c "telnet frontend-svc 80"

# Inspect policies
kubectl describe netpol db-policy
kubectl describe netpol backend-policy
kubectl describe netpol frontend-policy
```

---

## 5. Production Flow 🏗️

### Reference Architecture: 3-Tier App with Zero Trust

```
Internet
    │
    ▼
[Load Balancer / Ingress Controller]
    │  port 443 (TLS termination)
    ▼
[frontend pods]  ←── NetworkPolicy: allow 443 in from LB CIDR only
    │  port 5678
    ▼
[backend pods]   ←── NetworkPolicy: allow 5678 in from frontend only
    │  port 3306
    ▼
[db pods]        ←── NetworkPolicy: allow 3306 in from backend only
                      No egress allowed
```

### DNS — The Policy Nobody Remembers

Every pod that uses service names (99% of them) **needs DNS egress**. This is the most common reason NetworkPolicies silently break applications.

```yaml
# Reusable DNS egress rule — add to every policy that restricts Egress
egress:
- to:
  - namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: kube-system
    podSelector:
      matchLabels:
        k8s-app: kube-dns
  ports:
  - protocol: UDP
    port: 53
  - protocol: TCP
    port: 53
```

### Multi-Namespace Pattern

When DB is in a separate `db-ns` namespace:

```yaml
# backend policy egress to separate db namespace
egress:
- to:
  - namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: db-ns
    podSelector:
      matchLabels:
        role: db
  ports:
  - protocol: TCP
    port: 3306
```

```yaml
# db-policy in db-ns — allow ingress from app1-ns backend pods
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: app1-ns
    podSelector:
      matchLabels:
        role: backend
  ports:
  - protocol: TCP
    port: 3306
```

### Default Deny All (Baseline Policy — Apply to Every Namespace)

```yaml
# default-deny-all.yaml — baseline isolation for every namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: app1-ns
spec:
  podSelector: {}    # Selects ALL pods in the namespace
  policyTypes:
  - Ingress
  - Egress
  # No ingress/egress rules = DENY EVERYTHING
```

> 🏭 **Production pattern:** Apply `default-deny-all` first, then layer specific allow policies on top. This ensures new pods are isolated by default until explicitly permitted.

---

## 6. Mistakes 💥

### Mistake 1: Forgetting DNS Egress

**Symptom:** Service works locally (by IP), breaks when using service name. `nslookup backend-svc` times out.

**Root Cause:** Egress policy is applied, but port 53 to CoreDNS is not allowed. The pod can't resolve `backend-svc` to a ClusterIP.

**Fix:** Add DNS egress rule (see Section 4 for the exact YAML).

---

### Mistake 2: OR vs AND Selector Confusion

**Symptom:** Policy allows more traffic than expected. Unrelated pods can reach the target.

**Root Cause:** Using two separate `from:` entries instead of one combined entry.

```yaml
# ❌ WRONG — OR logic
from:
- namespaceSelector: { matchLabels: { env: prod } }
- podSelector: { matchLabels: { role: backend } }
# Allows: any pod in "env:prod" namespaces OR any pod labeled "role:backend" anywhere
```

```yaml
# ✅ CORRECT — AND logic
from:
- namespaceSelector: { matchLabels: { env: prod } }
  podSelector: { matchLabels: { role: backend } }
# Allows: only pods labeled "role:backend" that are ALSO in "env:prod" namespaces
```

---

### Mistake 3: Declaring Egress policyType Without Egress Rules

**Symptom:** Pod suddenly can't reach anything. All outbound connections fail.

**Root Cause:**

```yaml
policyTypes:
- Ingress
- Egress    # ← This with no egress: block = DENY ALL EGRESS
```

**Fix:** Either add `egress:` rules, or remove `Egress` from `policyTypes` if you don't need to control outbound traffic.

---

### Mistake 4: Using a CNI That Doesn't Support NetworkPolicy

**Symptom:** Policies apply successfully (`kubectl get netpol` shows them), but traffic flows through anyway.

**Root Cause:** Default KIND (Kindnet) or Flannel — these CNIs simply ignore NetworkPolicy objects.

**Fix:** Install Calico, Cilium, or Antrea. Verify with:
```bash
kubectl get ds -n kube-system | grep -E "calico|cilium|antrea"
```

---

### Mistake 5: Namespace Label Not Set

**Symptom:** `namespaceSelector` rules match nothing, traffic is denied even when it should be allowed.

**Root Cause:** The namespace doesn't have the label you're selecting on.

**Fix:**
```bash
# Check labels on namespace
kubectl get ns kube-system --show-labels

# Add missing label
kubectl label ns kube-system kubernetes.io/metadata.name=kube-system

# Note: kubernetes.io/metadata.name is auto-set in K8s 1.21+; older clusters need it manually
```

---

### Mistake 6: Headless Service DNS Format

**Symptom:** `telnet db-svc 3306` works, but `db-sts-0` style DNS doesn't resolve.

**Root Cause:** The StatefulSet's `serviceName` doesn't match the headless service name, or you're using the wrong FQDN format.

**Correct FQDN:** `<pod-name>.<service-name>.<namespace>.svc.cluster.local`
Example: `db-sts-0.db-svc.app1-ns.svc.cluster.local`

---

## 7. Interview Answers 🎤

**Q: What is a NetworkPolicy in Kubernetes?**

> "A NetworkPolicy is a Kubernetes API object that controls traffic flow to and from pods at Layer 3 and Layer 4 — meaning IP addresses and ports. By default, all pods in a cluster can communicate freely. Once a NetworkPolicy selects a pod, only traffic that explicitly matches an ingress or egress rule is allowed; everything else is denied. Importantly, Kubernetes itself doesn't enforce these policies — that's entirely delegated to the CNI plugin. If you're running Flannel or Kindnet, NetworkPolicies are stored in etcd but have zero effect on traffic."

---

**Q: What's the difference between ingress and egress in a NetworkPolicy?**

> "Ingress rules control traffic *coming into* a pod — who is allowed to initiate a connection to it. Egress rules control traffic *going out of* a pod — where it's allowed to send traffic. Both are from the perspective of the pod the policy is applied to. NetworkPolicies are stateful, so if an outgoing connection is permitted by an egress rule, the corresponding response packets are automatically allowed back in without needing a separate ingress rule."

---

**Q: How does AND vs OR work in NetworkPolicy selectors?**

> "This is a subtle but critical distinction. When you have multiple entries in a `from:` list — each starting with a dash — Kubernetes evaluates them as OR conditions: traffic is allowed if it matches *any* of the entries. However, when you combine `namespaceSelector` and `podSelector` within a *single* `from:` entry — at the same indentation level, without a new dash — Kubernetes applies AND logic: the traffic source must satisfy *both* conditions simultaneously. The most common production use of AND logic is scoping DNS access: you want to allow egress only to pods that are *both* in the `kube-system` namespace *and* labeled `k8s-app=kube-dns`."

---

**Q: What happens if you define `policyTypes: [Ingress, Egress]` but don't include any `egress:` rules?**

> "All egress traffic from the selected pods is immediately denied. This is a common footgun. The moment you declare `Egress` in `policyTypes`, Kubernetes treats the absence of `egress:` rules as 'allow nothing outbound.' This means DNS queries fail first, and then nothing works. The fix is either to add explicit egress rules — including DNS to CoreDNS — or to remove `Egress` from `policyTypes` if you only need to control ingress."

---

**Q: Why do most pods need DNS egress allowed, and how do you write that rule correctly?**

> "Service-to-service communication in Kubernetes relies on DNS. When a pod calls `backend-svc`, it first sends a DNS query to CoreDNS on port 53 to resolve that name to a ClusterIP. If your NetworkPolicy restricts egress and doesn't permit DNS traffic, the resolution fails and the pod can't reach anything by name. The correct rule allows egress to port 53 — both UDP and TCP — scoped to pods labeled `k8s-app=kube-dns` in the `kube-system` namespace, using AND logic with both a `namespaceSelector` and `podSelector` in the same `to:` entry."

---

**Q: How does Calico enforce NetworkPolicy under the hood?**

> "Calico uses a per-node agent called Felix that watches the Kubernetes API for NetworkPolicy objects. When a policy is created or updated, Felix translates it into either iptables rules or eBPF programs, depending on the dataplane mode, and installs them on the host's network stack. Traffic from pods flows through veth pairs into the host network namespace where these rules intercept it. Calico also runs BIRD, a BGP daemon, to advertise pod routes between nodes so cross-node traffic knows where to go. The kube-controllers component syncs Kubernetes state — like node and pod events — into Calico's own CRD-based datastore."

---

**Q: What's a "default deny all" policy and when should you use it?**

> "A default-deny-all policy selects all pods in a namespace using an empty `podSelector: {}` and declares both Ingress and Egress in `policyTypes` with no corresponding rules. This denies all traffic for every pod in that namespace. You should apply this as your baseline in any production namespace before deploying workloads, then layer specific allow policies on top. This ensures that any new pod added to the namespace is isolated by default until an explicit policy grants it access — which is the zero-trust model you want in production."

---

## 8. Debugging 🔍

### Symptom → Diagnosis Path

```
Pod can't reach a service
│
├─ 1. Is the target pod running?
│     kubectl get pods -n <ns> -l <selector>
│     kubectl describe pod <pod> | grep -A5 Conditions
│
├─ 2. Does the service have healthy endpoints?
│     kubectl get endpoints <svc-name> -n <ns>
│     # Empty SUBSETS = selector doesn't match any pods
│
├─ 3. Does the CNI support NetworkPolicy?
│     kubectl get ds -n kube-system | grep -E "calico|cilium|antrea"
│     # No match = policies are silently ignored
│
├─ 4. Are there any NetworkPolicies selecting the pod?
│     kubectl get netpol -n <ns>
│     kubectl describe netpol <policy-name> -n <ns>
│     # Look at "Allowing ingress/egress" sections
│
├─ 5. Is DNS working?
│     kubectl exec -it <pod> -- nslookup backend-svc
│     # Fails = missing DNS egress rule (port 53 to kube-dns)
│
├─ 6. Is it a namespace label problem?
│     kubectl get ns <ns> --show-labels
│     # Missing label = namespaceSelector matches nothing
│
└─ 7. Check if it's AND vs OR selector problem
      kubectl describe netpol <policy> | grep -A20 "Allowing"
      # Trace which from:/to: entries actually match your pod
```

### Quick Diagnostic Commands

```bash
# Show all NetworkPolicies and their selectors
kubectl get netpol -A -o wide

# Describe a specific policy — shows which pods it selects
kubectl describe netpol <policy-name> -n <namespace>

# Check if a pod is selected by any policy
kubectl get netpol -n <ns> -o json | \
  jq '.items[] | select(.spec.podSelector.matchLabels | to_entries[] | .value == "backend") | .metadata.name'

# Check namespace labels (critical for namespaceSelector)
kubectl get ns --show-labels

# Check node-level iptables rules (Calico iptables mode)
# Run on the node where the pod is scheduled:
iptables -L -n | grep -i calico | head -30

# Calico-specific: check Felix logs on the node
kubectl logs -n kube-system -l k8s-app=calico-node -c calico-node | grep -i "policy\|felix" | tail -50

# Test connectivity directly from inside a pod (without tools)
kubectl exec -it <pod> -- sh -c "echo > /dev/tcp/<target-ip>/<port>" && echo "OPEN" || echo "BLOCKED"

# Verify CoreDNS pods have the right label
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check if StatefulSet DNS format is correct
kubectl exec -it <pod> -- nslookup db-sts-0.db-svc.app1-ns.svc.cluster.local
```

### Common Failure Patterns

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Connection hangs indefinitely | Traffic blocked by NetworkPolicy | Add allow rule or check selector |
| `nslookup` fails | Missing DNS egress rule | Add port 53 egress to kube-dns |
| Policy applied but traffic flows anyway | CNI doesn't support NetworkPolicy | Install Calico/Cilium |
| `namespaceSelector` matches too much | OR instead of AND logic | Combine selectors in single `from:` |
| New pod has no network | `default-deny-all` active | Add specific allow policy for new pod |
| `telnet db-svc 3306` works, `db-sts-0.db-svc` doesn't | Wrong DNS format or headless service mismatch | Check `serviceName` in StatefulSet matches headless service |

---

## 9. Kill Switch ⚡

**10-second recall — the absolute minimum:**

```
1. Default = all traffic open. Policy selected = deny all except listed.
2. Rules are ADDITIVE across policies. You can only allow, never deny.
3. NetworkPolicy is STATEFUL — responses are auto-allowed.
4. CNI enforces policies, not Kubernetes. Flannel/Kindnet = ignored.
5. policyTypes: [Egress] with no egress: rules = DENY ALL OUTBOUND.
6. AND = combine in one from:/to: item. OR = separate items with dash.
7. ALWAYS allow DNS: port 53 UDP+TCP to kube-system/k8s-app=kube-dns.
8. namespaceSelector matches on LABELS, not namespace NAME.
9. Empty podSelector: {} = select ALL pods in namespace.
10. Default-deny-all first, then layer specific allows on top.
```

---

## 10. Appendix — Quick Reference 📋

### NetworkPolicy YAML Structure

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: <name>
  namespace: <namespace>
spec:
  podSelector:           # Which pods this policy applies to
    matchLabels:
      <key>: <value>     # Empty {} = ALL pods in namespace
  policyTypes:
  - Ingress              # Control incoming traffic
  - Egress               # Control outgoing traffic
  ingress:
  - from:
    - podSelector: {}          # From pods (same namespace)
    - namespaceSelector: {}    # From entire namespace
    - ipBlock:                 # From external CIDR
        cidr: <CIDR>
        except:
        - <excluded-CIDR>
    ports:
    - protocol: TCP
      port: <port>
  egress:
  - to:
    - podSelector: {}
    - namespaceSelector: {}
    - ipBlock:
        cidr: <CIDR>
    ports:
    - protocol: TCP
      port: <port>
```

### Selector Type Comparison

| Selector | Scope | Matches By |
|----------|-------|-----------|
| `podSelector` | Current namespace | Pod labels |
| `namespaceSelector` | Across cluster | Namespace labels |
| `ipBlock` | External | CIDR ranges |
| Combined (AND) | Single `from:` entry | BOTH namespace label AND pod label |
| Separate (OR) | Multiple `from:` entries | EITHER condition |

### StatefulSet DNS Format

```
<pod-name>.<service-name>.<namespace>.svc.cluster.local
db-sts-0.db-svc.app1-ns.svc.cluster.local
```

### Essential Commands Cheatsheet

```bash
# CRUD
kubectl apply -f netpol.yaml
kubectl get netpol -n <ns>
kubectl describe netpol <name> -n <ns>
kubectl delete netpol <name> -n <ns>

# Debug
kubectl get endpoints <svc> -n <ns>
kubectl get ns --show-labels
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl exec -it <pod> -- nslookup <service>

# Test connectivity
kubectl exec -it <pod> -- sh -c "echo > /dev/tcp/<ip>/<port> 2>/dev/null && echo OPEN || echo CLOSED"

# KIND cluster management
kind get clusters
kind delete cluster --name=<name>
kind create cluster --config kind-cluster.yaml --name=cka-2025

# Calico health check
kubectl get ds calico-node -n kube-system
kubectl get deployment calico-kube-controllers -n kube-system
kubectl logs -n kube-system -l k8s-app=calico-node -c calico-node --tail=50
```

### Policy Evaluation Flow (Mental Model)

```
Packet arrives at node for Pod B
         │
         ▼
Is Pod B selected by any NetworkPolicy with policyTypes [Ingress]?
         │
    No ──┼── Yes
    │    │      │
    │    │      ▼
    │    │   Does packet match ANY ingress rule?
    │    │      │
    │    │  Yes ┼ No
    │    │  │   │
    │    │  │   └── DROP
    │    │  │
    └────┴──┴── ALLOW (conntrack entry created)
```