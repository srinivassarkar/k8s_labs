# Kubernetes Services & Deployments 
---

## 0. First Principles

> *What never changes — the mental model you carry into every question.*

**The Kubernetes Service contract is immutable:** a Service does not know about pods. It knows about **labels**. The moment a pod carries the right label, it is eligible for traffic — regardless of how it was created, what Deployment manages it, or what namespace it lives in (within scope). This is the single most probed concept in CKA and interviews.

**Three laws you must internalize:**

1. **Label = identity.** A pod without a label matching a Service selector is invisible to that Service. A pod with a matching label is immediately routed to — no restart, no reload.
2. **Service = stable endpoint, not load balancer.** kube-proxy programs iptables/IPVS rules on every node. The Service VIP (ClusterIP) never changes after creation. Pods are ephemeral; the Service is permanent.
3. **Deployment = desired state, not running state.** The Deployment controller perpetually reconciles reality toward your `spec.replicas`. It owns ReplicaSets; it does not own pods directly.

**The flow everything else is built on:**

```
Deployment spec → ReplicaSet → Pod (with labels)
                                    ↓
                             Service selector matches labels
                                    ↓
                             kube-proxy rules → traffic routed to pod IP:containerPort
```

This chain is the answer skeleton for almost every service-connectivity question.

---

## 1. Reality Constraints

> *What Kubernetes actually does and doesn't do — the gaps that cause production incidents.*

### What Kubernetes Guarantees

| Guarantee | Detail |
|---|---|
| ClusterIP is stable | Survives pod restarts, node failures, rolling updates |
| Service DNS is immediate | CoreDNS resolves `<svc-name>.<namespace>.svc.cluster.local` without restart |
| Endpoint slice updates are eventual | New pod IPs appear in endpoints within seconds (not milliseconds) |
| NodePort is on ALL nodes | Traffic on `:nodePort` is accepted on every node, even nodes with no matching pod |

### What Kubernetes Does NOT Guarantee

| Misconception | Reality |
|---|---|
| Service routes to healthy pods | Service routes to **Ready** pods. Readiness probe failure removes a pod from Endpoints — liveness failure does not immediately |
| `--port` and `--target-port` are the same | `--port` is what clients call. `--target-port` is the container port. They are independent |
| `nodePort` is optional in YAML | It IS optional — Kubernetes assigns a random port from `30000–32767`. But you cannot set it imperatively |
| Labels on Deployment template propagate to ReplicaSet | The `spec.selector` on a Deployment is **immutable after creation**. You cannot patch it |
| ClusterIP is reachable from outside the cluster | It is not. ClusterIP is a virtual IP routed only by kube-proxy inside the cluster |
| `kubectl expose` always works | `kubectl expose` cannot set `nodePort` value — it generates it randomly. Use YAML |

### Port Terminology (Critical for CKA)

```
Client → Service port (--port) → kube-proxy → Pod targetPort (--target-port / containerPort)
                        ↑
                NodePort (external) sits here for NodePort Services
```

| Field | Where it lives | What it means |
|---|---|---|
| `port` | Service spec | Port clients use to reach the Service |
| `targetPort` | Service spec | Port the container is actually listening on |
| `nodePort` | Service spec | External port opened on every node (30000–32767) |
| `containerPort` | Pod/container spec | Informational only — does NOT open a port |

---

## 2. Decision Logic

> *When to use what — clear rules before you write a single line of YAML.*

### Service Type Selection

```
Do you need to expose traffic?
│
├── No external access needed → ClusterIP (default)
│     └── Use case: backend APIs, databases, internal microservices
│
├── External access, single cluster, no cloud LB → NodePort
│     └── Use case: dev/test, on-prem, exam environments
│
├── External access, cloud provider available → LoadBalancer
│     └── Use case: production cloud workloads
│
└── Route based on hostname/path → Ingress (backed by ClusterIP Services)
      └── Use case: multi-service HTTP routing, TLS termination
```

### Imperative vs. Declarative Decision

```
Is this a one-shot operation (exam, quick test)?
│
├── Yes → Imperative with --dry-run=client -o yaml
│           kubectl create / expose / run → pipe to file → edit → apply
│
└── No → Declarative YAML from the start
          Tracked in git, repeatable, auditable
```

### When You MUST Use YAML (Cannot Do Imperatively)

| Requirement | Why imperative fails |
|---|---|
| Set a specific `nodePort` value | `kubectl expose` assigns random port |
| Add `args` to a container | `kubectl create deployment` has no `--args` flag |
| Set `readinessProbe` / `livenessProbe` | No imperative flag |
| Multiple containers in a pod | No imperative support |
| Set resource `requests` and `limits` | No imperative support for Deployments |
| `initContainers` | YAML only |

### Label Selector Rules

```
Does Service selector match pod labels exactly?
│
├── Yes (all key-value pairs match) → Pod is included in Endpoints
│
├── No (any mismatch, even case) → Pod is excluded; Service has no endpoints
│     └── Symptom: curl returns "connection refused" or times out
│
└── Label exists on pod but not in selector → irrelevant, pod still matches
```

---

## 3. Internal Working

> *How it actually happens under the hood — step by step.*

### When You Run `kubectl apply -f backend-deploy.yaml`

1. **API Server** receives the request, validates schema, persists to etcd.
2. **Deployment Controller** (in kube-controller-manager) watches etcd for Deployment objects. It creates a **ReplicaSet** with a `spec.selector` matching the Deployment's `spec.selector`.
3. **ReplicaSet Controller** watches its owned ReplicaSet, creates N **Pod** objects in etcd.
4. **Scheduler** sees unbound pods (no `nodeName`), scores nodes, patches each Pod with `spec.nodeName`.
5. **kubelet** on the assigned node watches for Pods scheduled to it, pulls the image, calls the container runtime (containerd), starts the container.
6. **kubelet** reports Pod status back → Pod reaches `Running`.

### When You Apply a Service

1. **API Server** persists the Service object. Kubernetes assigns a ClusterIP from the service CIDR (e.g., `10.96.0.0/12`).
2. **Endpoints/EndpointSlice Controller** watches for Services with selectors. It queries all pods across the namespace, finds pods whose labels match the selector, and creates an **EndpointSlice** object containing their `podIP:targetPort`.
3. **kube-proxy** on every node watches EndpointSlice objects. It programs **iptables** rules (or IPVS entries) that DNAT traffic destined for `ClusterIP:port` to one of the pod IPs in the endpoint slice (round-robin or random, depending on mode).
4. **CoreDNS** watches Service objects and creates a DNS A record: `<service-name>.<namespace>.svc.cluster.local → ClusterIP`.

### What Happens When a Pod Dies

1. Pod transitions to `Terminating` → kubelet sends SIGTERM, waits `terminationGracePeriodSeconds`.
2. Endpoint controller removes the pod IP from EndpointSlice **before** the pod fully terminates (configurable with `preStop` hooks).
3. kube-proxy removes the iptables rule for that pod IP.
4. ReplicaSet controller detects `current < desired`, creates a replacement pod.
5. New pod IP is added to EndpointSlice after it passes readiness checks.

### NodePort Traffic Path

```
External client → Node IP:31000
    → iptables PREROUTING DNAT rule (kube-proxy)
    → ClusterIP:port
    → iptables DNAT to Pod IP:targetPort
    → Pod (possibly on a different node — cross-node traffic via overlay network)
```

> **Key insight:** `kube-proxy --proxy-mode=iptables` (default) means traffic never actually passes through kube-proxy at runtime. kube-proxy only programs the rules; the kernel handles forwarding.

---

## 4. Hands-On

> *Production-quality YAML + commands — nothing simplified.*

### Backend Deployment + ClusterIP Service

```bash
# Step 1: Generate scaffolding
kubectl create deployment backend-deploy \
  --image=hashicorp/http-echo \
  --replicas=3 \
  --port=5678 \
  --dry-run=client -o yaml > backend-deploy.yaml

# Step 2: Append Service scaffolding (note: >> not >)
kubectl expose deployment backend-deploy \
  --type=ClusterIP \
  --port=9090 \
  --target-port=5678 \
  --name=backend-svc \
  --dry-run=client -o yaml >> backend-deploy.yaml
```

**Final `backend-deploy.yaml`** (after manual edits — this is what you submit):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-deploy
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
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - protocol: TCP
      port: 9090
      targetPort: 5678
```

> **Critical:** `hashicorp/http-echo` exits immediately without `-text` and `-listen` args. Always include them.

### Frontend Deployment + NodePort Service

```bash
# Step 1: Generate scaffolding
kubectl create deployment frontend-deploy \
  --image=nginx \
  --replicas=3 \
  --port=80 \
  --dry-run=client -o yaml > frontend-deploy.yaml

# Step 2: Append Service scaffolding
kubectl expose deployment frontend-deploy \
  --type=NodePort \
  --port=80 \
  --target-port=80 \
  --name=frontend-svc \
  --dry-run=client -o yaml >> frontend-deploy.yaml
```

**Final `frontend-deploy.yaml`:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deploy
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

### Apply and Verify

```bash
# Apply both manifests
kubectl apply -f backend-deploy.yaml
kubectl apply -f frontend-deploy.yaml

# Verify all resources
kubectl get all --show-labels

# Inspect endpoints (critical — confirms selector is working)
kubectl get endpoints backend-svc
kubectl get endpoints frontend-svc

# Full service inspection
kubectl describe svc backend-svc
kubectl describe svc frontend-svc

# Pod placement (which node each pod is on)
kubectl get pods -o wide
```

### Connectivity Tests

```bash
# Test 1: Frontend pod → backend service (in-cluster)
kubectl exec -it <frontend-pod-name> -- curl http://backend-svc:9090
# Expected: Hello from Backend

# Test 2: In-cluster using full DNS name (cross-namespace pattern)
kubectl exec -it <frontend-pod-name> -- \
  curl http://backend-svc.default.svc.cluster.local:9090

# Test 3: External access via NodePort
curl http://<NodeIP>:31000
# Expected: nginx welcome page

# Test 4: Get node IP if unknown
kubectl get nodes -o wide | awk '{print $6}'

# Test 5: Quick debug pod (no YAML needed)
kubectl run debug --image=curlimages/curl --rm -it --restart=Never \
  -- curl http://backend-svc:9090
```

---

## 5. Production Flow

> *Real-world architecture and design patterns beyond the exam.*

### Typical Frontend → Backend Architecture

```
Internet
    │
    ▼
[Ingress Controller] (e.g., nginx-ingress, NodePort :443/:80)
    │  routes by hostname/path
    ▼
[frontend-svc : ClusterIP]    [backend-svc : ClusterIP]
    │                               │
[frontend pods]   ──────────▶  [backend pods]
(nginx, React build)           (API, hashicorp/http-echo)
```

> In production, NodePort is almost never used directly. An Ingress or cloud LoadBalancer Service sits in front, and all downstream services are ClusterIP.

### Rolling Update Strategy (Default and Recommended)

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Max pods above desired during update
      maxUnavailable: 0  # Zero downtime: never reduce below desired
```

### Production Label Taxonomy

```yaml
# On pods, use multiple labels for fine-grained selection
metadata:
  labels:
    app: backend           # Service selector key
    version: "1.4.2"       # For canary/blue-green with separate Services
    tier: api              # For NetworkPolicy targeting
    env: production        # For monitoring aggregation
```

### NetworkPolicy Pattern (Least Privilege)

```yaml
# Only allow frontend pods to reach backend pods on port 5678
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-allow-frontend
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 5678
```

### Multi-Deployment, Single Service Pattern (Blue-Green)

```yaml
# Service selects on a "slot" label; you swap deployments by changing the label
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
spec:
  selector:
    app: backend
    slot: blue   # Change to "green" to cut traffic to the new deployment
```

---

## 6. Mistakes

> *What actually breaks in real systems — root cause + fix.*

### Mistake 1: Service Has No Endpoints

**Symptom:** `kubectl get endpoints backend-svc` shows `<none>`.

**Root cause:** Label selector mismatch. The Service's `spec.selector` does not match any running pod's `metadata.labels`.

**Common triggers:**
- Typo in label key or value (`app: Backend` vs `app: backend`)
- Label added to Deployment metadata but not to `spec.template.metadata.labels`
- Pod was created outside the Deployment and lacks the label

**Fix:**
```bash
# Compare selector vs pod labels
kubectl get svc backend-svc -o jsonpath='{.spec.selector}'
kubectl get pods --show-labels
# Patch label on a running pod (temporary)
kubectl label pod <pod-name> app=backend
```

---

### Mistake 2: `http-echo` Pod CrashLoopBackOff

**Symptom:** Pod starts and exits immediately. `kubectl logs` shows nothing.

**Root cause:** `hashicorp/http-echo` with no `args` has no default behavior — it exits 0.

**Fix:** Add args in the container spec:
```yaml
args:
  - "-text=Hello from Backend"
  - "-listen=:5678"
```

---

### Mistake 3: Deployment Selector Immutability Error

**Symptom:** `kubectl apply` returns: `field is immutable`.

**Root cause:** You tried to change `spec.selector.matchLabels` on an existing Deployment.

**Fix:** You must delete and recreate the Deployment. There is no patch path.
```bash
kubectl delete deployment backend-deploy
kubectl apply -f backend-deploy.yaml
```

---

### Mistake 4: NodePort Not Reachable from Outside

**Symptom:** `curl http://<NodeIP>:31000` times out.

**Root cause (in order of likelihood):**
1. Firewall / Security Group blocking the NodePort range (30000–32767)
2. Wrong node IP (using internal IP from a cloud environment where external IP differs)
3. `kube-proxy` is not running on that node
4. The NodePort was not set explicitly and you're hitting the wrong port

**Fix:**
```bash
# Confirm the actual NodePort assigned
kubectl get svc frontend-svc -o jsonpath='{.spec.ports[0].nodePort}'

# Confirm node external IP
kubectl get nodes -o wide

# Confirm kube-proxy is running
kubectl get pods -n kube-system | grep kube-proxy

# Check iptables rules for the service
iptables -t nat -L KUBE-SERVICES | grep frontend
```

---

### Mistake 5: Using `>>` vs `>` When Building Multi-Resource Files

**Symptom:** Only one resource in the file, or file is overwritten.

**Root cause:** Using `>` (overwrite) for the second command instead of `>>` (append).

**Fix:**
```bash
# First resource: use >
kubectl create deployment ... --dry-run=client -o yaml > deploy.yaml
# Second resource: use >>
kubectl expose deployment ... --dry-run=client -o yaml >> deploy.yaml
# Then manually add --- separator between the two documents
```

---

### Mistake 6: `containerPort` in Pod Spec Is Not What Opens the Port

**Symptom:** Removing `containerPort` from the pod spec causes no change in behavior — traffic still works. But adding it and mismatching it with `targetPort` causes confusion.

**Root cause:** `containerPort` in a pod spec is **documentation only**. It does not instruct the container runtime to open a port. The process inside the container must bind to that port. The Service's `targetPort` must match what the process actually listens on.

---

## 7. Interview Answers

> *Compressed, verbatim-ready answers. Practice speaking these aloud.*

---

**Q: How does a Kubernetes Service route traffic to pods?**

"A Service uses a label selector to identify eligible pods. The Endpoints controller watches for pods matching that selector and maintains an EndpointSlice containing their IP-port pairs. On every node, kube-proxy watches these EndpointSlices and programs iptables or IPVS rules to DNAT traffic destined for the Service's ClusterIP and port to one of the pod IPs. So at runtime, the kernel handles the forwarding — kube-proxy itself is only the rule programmer, not the data plane."

---

**Q: What's the difference between ClusterIP, NodePort, and LoadBalancer?**

"ClusterIP gives you a stable virtual IP accessible only inside the cluster — that's the baseline. NodePort builds on top of ClusterIP by opening a port, between 30000 and 32767, on every node in the cluster. Traffic hitting any node on that port gets forwarded to the ClusterIP, and then to a pod. LoadBalancer builds on top of NodePort by additionally provisioning an external cloud load balancer — it's a superset. In production, you rarely expose NodePort directly; instead, you put an Ingress controller in front, which itself uses a NodePort or LoadBalancer Service."

---

**Q: What happens if a Service selector matches no pods?**

"The Service still exists and gets a ClusterIP, but its Endpoints object will be empty — you'll see `<none>` if you run `kubectl get endpoints`. Any traffic sent to that ClusterIP will be dropped at the iptables level because there are no DNAT targets. This is a very common failure mode when labels are mismatched between the Service selector and the pod template."

---

**Q: Why is `spec.selector` on a Deployment immutable?**

"Because a Deployment manages ReplicaSets by ownership. If you could change the selector, the Deployment would lose track of its existing ReplicaSets — it wouldn't know which pods it owns. Kubernetes prevents this to avoid orphaned ReplicaSets and duplicate pods. If you need to change the selector, you must delete the Deployment and recreate it. In practice, this is one of the strongest arguments for GitOps — you'd delete and reapply from your updated manifest."

---

**Q: What is the difference between `port`, `targetPort`, and `nodePort` in a Service?**

"`port` is what clients inside the cluster use to reach the Service. `targetPort` is what the Service forwards traffic to on the pod — it must match the port the container process is actually listening on. `nodePort` is the external port opened on every cluster node for NodePort-type Services. These three can all be different numbers. A common pattern is port 80, targetPort 8080, nodePort 31000."

---

**Q: How would you expose a Deployment externally in a production Kubernetes cluster?**

"In production, I would not use NodePort directly. I'd deploy an Ingress controller — typically nginx-ingress or Traefik — which is itself exposed via a cloud LoadBalancer Service. Then I'd define an Ingress resource that routes HTTP/HTTPS traffic by hostname or path to ClusterIP Services backed by my Deployments. This gives me TLS termination, path-based routing, and a single external IP for multiple services. For non-HTTP traffic, I'd use a LoadBalancer Service directly."

---

**Q: Can you walk me through what happens step-by-step when you run `kubectl apply -f deployment.yaml`?**

"The kubectl client sends the manifest to the API server, which validates it against the schema and persists it to etcd. The Deployment controller, running inside kube-controller-manager, watches etcd for Deployment objects and creates a ReplicaSet. The ReplicaSet controller creates the Pod objects. The scheduler picks these up, scores nodes, and assigns each pod to a node by patching its `spec.nodeName`. The kubelet on that node watches for pods assigned to it, pulls the container image, calls the container runtime to start it, and then reports status back. Once the pod is Running and passes readiness checks, the Endpoints controller adds its IP to the EndpointSlice, and kube-proxy updates iptables — at which point the Service can route traffic to it."

---

## 8. Debugging

> *Fast diagnosis paths — not a list of commands, but an actual decision tree.*

### Service Connectivity Failure — Diagnosis Path

```
curl from pod to service fails (connection refused / timeout)
│
├─── Step 1: Is the Service reachable at all?
│    kubectl get svc <name>
│    → ClusterIP assigned? If "None", it's a headless service — expected
│    → No ClusterIP? Service may not have been created. Check: kubectl get svc
│
├─── Step 2: Does the Service have endpoints?
│    kubectl get endpoints <svc-name>
│    → Shows <none> → LABEL MISMATCH (go to Step 3)
│    → Shows pod IPs → Service is fine, problem is upstream or in the pod itself
│
├─── Step 3: Compare selector vs pod labels
│    kubectl get svc <name> -o jsonpath='{.spec.selector}'
│    kubectl get pods --show-labels
│    → Mismatch found → fix label on pod template or selector on service
│    → Match looks correct → check if pods are Ready
│
├─── Step 4: Are pods actually Ready?
│    kubectl get pods
│    → 0/1 Running → pod not ready, check readiness probe
│    → CrashLoopBackOff → kubectl logs <pod> (check for missing args, etc.)
│    → Pending → scheduling issue, kubectl describe pod <name>
│
├─── Step 5: Can you reach the pod IP directly?
│    POD_IP=$(kubectl get pod <name> -o jsonpath='{.status.podIP}')
│    kubectl exec -it <debug-pod> -- curl http://$POD_IP:5678
│    → Works → problem is in Service selector or port mapping
│    → Fails → problem is in the container (wrong port, not listening, crashed)
│
└─── Step 6: Validate port mapping
     kubectl describe svc <name> | grep -A5 "Port:"
     → port/targetPort mismatch with what container listens on?
     → Fix targetPort to match containerPort
```

### NodePort External Access Failure — Diagnosis Path

```
curl http://<NodeIP>:31000 fails
│
├─── Step 1: Confirm the NodePort
│    kubectl get svc frontend-svc -o jsonpath='{.spec.ports[0].nodePort}'
│
├─── Step 2: Confirm node IP (external vs internal)
│    kubectl get nodes -o wide
│    → Use EXTERNAL-IP column, not INTERNAL-IP
│
├─── Step 3: Check firewall/security group
│    Cloud environments: check Security Group allows 30000-32767 inbound
│    On-prem: iptables -L INPUT | grep 31000
│
├─── Step 4: Confirm kube-proxy is healthy
│    kubectl get pods -n kube-system -l k8s-app=kube-proxy
│    kubectl logs -n kube-system <kube-proxy-pod>
│
└─── Step 5: Confirm iptables rules exist
     iptables -t nat -L KUBE-NODEPORTS | grep 31000
     → No rule → kube-proxy hasn't synced; check kube-proxy logs
```

### Quick Diagnostic Commands

```bash
# All-in-one status view
kubectl get all --show-labels -n <namespace>

# Why is a pod not scheduled?
kubectl describe pod <name> | grep -A10 Events

# What IPs is the service sending to?
kubectl get endpoints <svc> -o yaml

# Test DNS resolution from inside the cluster
kubectl run dnstest --image=busybox --rm -it --restart=Never \
  -- nslookup backend-svc.default.svc.cluster.local

# Watch endpoint changes in real time
kubectl get endpoints -w

# Check kube-proxy mode
kubectl get configmap -n kube-system kube-proxy -o yaml | grep mode
```

---

## 9. Kill Switch

> *10-second recall — the absolute minimum to hold in memory under exam pressure.*

```
SERVICE = stable IP + label selector + port mapping
  port → targetPort → containerPort (must match process)

ClusterIP  = internal only
NodePort   = ClusterIP + host port (30000-32767), cannot set imperatively
LoadBalancer = NodePort + cloud LB

WORKFLOW:
  kubectl create/expose --dry-run=client -o yaml > file.yaml
  edit: add ---, labels, args, nodePort
  kubectl apply -f file.yaml

LABEL MISMATCH → no endpoints → traffic dropped
containerPort = docs only, does not open port
spec.selector on Deployment = immutable
http-echo NEEDS args: -text and -listen

DEBUG ORDER:
  svc exists? → endpoints? → pod ready? → pod IP reachable? → port correct?
```

---

## 10. Appendix

### Quick Reference: Service Types

| Type | ClusterIP | NodePort | External LB | Use Case |
|---|---|---|---|---|
| ClusterIP | ✅ | ❌ | ❌ | Internal services |
| NodePort | ✅ | ✅ | ❌ | Dev/test, on-prem |
| LoadBalancer | ✅ | ✅ | ✅ | Production cloud |
| ExternalName | ❌ | ❌ | ❌ | DNS alias to external |
| Headless (`ClusterIP: None`) | ❌ | ❌ | ❌ | StatefulSets, direct pod DNS |

### Quick Reference: Imperative Commands

```bash
# Create Deployment (scaffolding)
kubectl create deployment <name> \
  --image=<image> \
  --replicas=<n> \
  --port=<containerPort> \
  --dry-run=client -o yaml > deploy.yaml

# Expose Deployment as Service (scaffolding)
kubectl expose deployment <name> \
  --type=<ClusterIP|NodePort|LoadBalancer> \
  --port=<servicePort> \
  --target-port=<containerPort> \
  --name=<svcName> \
  --dry-run=client -o yaml >> deploy.yaml

# Run debug pod (ephemeral, auto-deleted)
kubectl run debug --image=curlimages/curl --rm -it --restart=Never \
  -- curl http://<service>:<port>

# Run debug pod with shell
kubectl run debug --image=busybox --rm -it --restart=Never -- sh

# Get pod labels
kubectl get pods --show-labels

# Label a pod imperatively
kubectl label pod <name> <key>=<value>

# Remove a label
kubectl label pod <name> <key>-

# Patch a Service type (e.g., ClusterIP → NodePort)
kubectl patch svc <name> -p '{"spec":{"type":"NodePort"}}'

# Scale Deployment
kubectl scale deployment <name> --replicas=5

# Watch rollout status
kubectl rollout status deployment/<name>

# Rollback
kubectl rollout undo deployment/<name>
```

### Quick Reference: Useful JSONPath Extractions

```bash
# Get ClusterIP
kubectl get svc <name> -o jsonpath='{.spec.clusterIP}'

# Get NodePort
kubectl get svc <name> -o jsonpath='{.spec.ports[0].nodePort}'

# Get all pod IPs
kubectl get pods -o jsonpath='{range .items[*]}{.status.podIP}{"\n"}{end}'

# Get node external IPs
kubectl get nodes -o jsonpath='{range .items[*]}{.status.addresses[?(@.type=="ExternalIP")].address}{"\n"}{end}'

# Get selector from a Service
kubectl get svc <name> -o jsonpath='{.spec.selector}'
```

### Quick Reference: Label Operations

```bash
# Show all resources with labels
kubectl get all --show-labels

# Filter by label selector
kubectl get pods -l app=backend
kubectl get pods -l 'app in (frontend, backend)'
kubectl get pods -l app!=backend

# Check endpoint membership (does this pod appear in the service?)
kubectl get endpoints <svc-name> -o yaml | grep <pod-ip>
```

### YAML Skeleton: Minimal Working Deployment + ClusterIP

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deploy
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp          # Must match template labels
  template:
    metadata:
      labels:
        app: myapp        # Must match selector
    spec:
      containers:
        - name: myapp
          image: nginx
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: my-svc
spec:
  type: ClusterIP
  selector:
    app: myapp            # Must match pod labels
  ports:
    - port: 80            # Service port (client-facing)
      targetPort: 80      # Container port (must match containerPort)
```

### Common Gotchas Cheatsheet

| Gotcha | Rule |
|---|---|
| `>>` not `>` | Second imperative command must append, not overwrite |
| `---` separator | Required between two resources in one file |
| `http-echo` args | Always add `-text` and `-listen` or pod crashes |
| `nodePort` in YAML | Must be between 30000–32767 or omit for random |
| `spec.selector` immutable | Cannot change after Deployment creation |
| Control plane taint | Remove with `kubectl taint node <node> node-role.kubernetes.io/control-plane:NoSchedule-` |
| DNS format | `<svc>.<namespace>.svc.cluster.local` |
| Readiness vs Liveness | Readiness failure removes from endpoints; liveness failure restarts pod |
