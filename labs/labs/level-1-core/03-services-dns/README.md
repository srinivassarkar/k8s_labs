# Lab 03 — Services & DNS

## What This Lab Is About

Services are how pods talk to each other and how the outside world reaches your app. DNS is how pods find Services by name instead of IP. Together, they are the nervous system of a Kubernetes cluster — and when they break, **everything** breaks. Traffic stops. Apps can't reach databases. Health checks fail. And the symptoms are often misleading.

This lab covers the 5 most critical Service and DNS failure modes in real clusters. A wrong selector that silently drops all traffic. A port mismatch that returns connection refused. DNS resolution failures inside the cluster. A Service that exists but routes to zero pods. And the ClusterIP vs NodePort confusion that blocks external access.

> When networking breaks in K8s, everything looks the same — connection refused or timeout. Your job is to narrow it down in under 5 minutes. This lab builds that instinct.

---

## Environment

- **Cluster:** KIND (Kubernetes IN Docker)
- **K8s Version:** v1.27.3
- **Node:** Single node (`kind-control-plane`)
- **Namespace for this lab:** `lab03`

```bash
kubectl create namespace lab03
kubectl config set-context --current --namespace=lab03
```

---

## The 5 Scenarios

| # | Scenario | What You'll Learn |
|---|---|---|
| 01 | Wrong selector — Service routes to zero pods | Label matching, Endpoints object |
| 02 | Port mismatch — connection refused | `port` vs `targetPort`, container port |
| 03 | DNS resolution failure inside cluster | CoreDNS, FQDN, cross-namespace DNS |
| 04 | ClusterIP not reachable from outside | ClusterIP vs NodePort vs the KIND-specific access pattern |
| 05 | Service exists but app is unreachable — full debug flow | Combining all tools into one systematic workflow |

---

## Scenario 01 — Wrong Selector — Service Routes to Zero Pods

### What You'll Break

A Service with a label selector that doesn't match any pod. The Service exists, DNS resolves it, but there are zero Endpoints behind it. Every request gets `connection refused` or times out. This is one of the most common silent failures in K8s networking.

### Apply the Broken State

```bash
# Create a healthy deployment
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: lab03
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend        # <-- pod label is "backend"
    spec:
      containers:
      - name: app
        image: nginx:1.25
        ports:
        - containerPort: 80
EOF

# Create a Service with a WRONG selector
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
  namespace: lab03
spec:
  selector:
    app: backend-api        # <-- WRONG: looking for "backend-api", pods have "backend"
  ports:
  - port: 80
    targetPort: 80
EOF
```

### Symptoms You Will Observe

The Service exists. The pods are running. But traffic goes nowhere.

```bash
kubectl get svc -n lab03
# NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
# backend-svc   ClusterIP   10.96.x.x       <none>        80/TCP    10s

kubectl get pods -n lab03
# Both pods Running and Ready

# Try to reach the service from inside the cluster
kubectl run curl-test --image=curlimages/curl:latest --rm -it --restart=Never \
  -n lab03 -- curl -s --connect-timeout 5 http://backend-svc.lab03.svc.cluster.local
# curl: (28) Connection timed out after 5000 milliseconds
```

Timeout. App is running. Service exists. Traffic is not reaching anything.

### Investigate

```bash
# Step 1 — Check the Service
kubectl get svc backend-svc -n lab03

# Step 2 — THE KEY COMMAND: Check Endpoints
kubectl get endpoints backend-svc -n lab03
# NAME          ENDPOINTS   AGE
# backend-svc   <none>      2m

# "<none>" means zero pods matched the selector
# This is your smoking gun

# Step 3 — Describe the Service to see its selector
kubectl describe svc backend-svc -n lab03
# Selector: app=backend-api
# Endpoints: <none>

# Step 4 — Check what labels the pods actually have
kubectl get pods -n lab03 --show-labels
# NAME                       READY   STATUS    LABELS
# backend-xxx-yyy            1/1     Running   app=backend,...

# Step 5 — Confirm the mismatch
# Service selector: app=backend-api
# Pod label:        app=backend
# These don't match — that's the entire problem
```

The `Endpoints` object is the truth. If `ENDPOINTS` shows `<none>`, the selector is wrong. Period.

### Fix

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
  namespace: lab03
spec:
  selector:
    app: backend            # Fixed: matches pod label
  ports:
  - port: 80
    targetPort: 80
EOF

# Verify Endpoints are populated now
kubectl get endpoints backend-svc -n lab03
# NAME          ENDPOINTS                       AGE
# backend-svc   10.244.0.x:80,10.244.0.y:80    5s

# Test connectivity
kubectl run curl-test --image=curlimages/curl:latest --rm -it --restart=Never \
  -n lab03 -- curl -s http://backend-svc.lab03.svc.cluster.local
# Returns nginx welcome page — working
```

### Prod Wisdom

**Always check Endpoints first when a Service is unreachable.** `kubectl get endpoints <svc-name>` is a one-second check that immediately tells you whether the problem is a selector or networking issue. `<none>` endpoints = selector mismatch, always. Non-empty endpoints = traffic is reaching pods, problem is at the app layer. This single check cuts your debug time in half.

---

## Scenario 02 — Port Mismatch — Connection Refused

### What You'll Break

A Service where `targetPort` doesn't match the port the container is actually listening on. The Endpoints are populated (selector is correct), traffic reaches the pod, but the pod rejects it because nothing is listening on that port. Returns `connection refused` immediately — not a timeout.

### Apply the Broken State

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: lab03
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: nginx:1.25
        ports:
        - containerPort: 80    # nginx listens on port 80
---
apiVersion: v1
kind: Service
metadata:
  name: api-svc
  namespace: lab03
spec:
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 8080           # WRONG: container listens on 80, not 8080
EOF
```

### Symptoms You Will Observe

```bash
kubectl get endpoints api-svc -n lab03
# NAME      ENDPOINTS           AGE
# api-svc   10.244.0.x:8080     10s
```

Notice: Endpoints ARE populated — but with port `8080`. The selector matched correctly. The pods are running. But:

```bash
kubectl run curl-test \
  --image=curlimages/curl:latest \
  -it \
  --rm \
  --restart=Never \
  -n lab03 \
  -- sh

~ $ curl -v --connect-timeout 5 http://api-svc.lab03.svc.cluster.local

* Host api-svc.lab03.svc.cluster.local:80 was resolved.
* IPv4: 10.96.50.40
*   Trying 10.96.50.40:80...
* connect to 10.96.50.40 port 80 failed: Connection refused
curl: (7) Failed to connect to api-svc.lab03.svc.cluster.local:80 after 1 ms: Could not connect to server
```

`Connection refused` (not timeout) — this means the traffic reached the pod's IP but the port is closed.

### Investigate

```bash
# Step 1 — Endpoints are populated (selector is fine)
kubectl get endpoints api-svc -n lab03
# ENDPOINTS: 10.244.0.x:8080  <-- port 8080

# Step 2 — Check what port the container actually exposes
kubectl describe deployment api -n lab03 | grep -A 3 "Port"
# Port: 80/TCP    <-- container exposes 80

# Step 3 — Check the Service targetPort
kubectl describe svc api-svc -n lab03
# TargetPort: 8080/TCP   <-- Service targeting 8080

# The mismatch: container on 80, Service targeting 8080

# Step 4 — Confirm by exec into the pod and check listening ports
kubectl exec -it $(kubectl get pod -n lab03 -l app=api -o jsonpath='{.items[0].metadata.name}') \
  -n lab03 -- ss -tlnp
# You'll see nginx listening on 0.0.0.0:80, NOT 8080
```

The difference between Scenario 01 and 02:
- Scenario 01: `ENDPOINTS: <none>` → selector problem
- Scenario 02: `ENDPOINTS: 10.x.x.x:8080` → port problem

### Fix

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: api-svc
  namespace: lab03
spec:
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 80             # Fixed: matches container port
EOF

# Verify
kubectl get endpoints api-svc -n lab03
# ENDPOINTS: 10.244.0.x:80    <-- correct port now

kubectl run curl-test --image=curlimages/curl:latest --rm -it --restart=Never \
  -n lab03 -- curl -s http://api-svc.lab03.svc.cluster.local | head -5
# Returns nginx HTML — working
```

### Understanding the Port Fields

```yaml
ports:
- port: 80           # Port the SERVICE listens on (what clients call)
  targetPort: 80     # Port on the POD to forward traffic to
  nodePort: 30080    # Port on the NODE (only for NodePort/LoadBalancer type)
```

These three are independent. You can expose a Service on port 80 while the container runs on 3000:

```yaml
ports:
- port: 80
  targetPort: 3000   # Client calls :80, traffic hits pod :3000
```

### Prod Wisdom

Port mismatches are one of the top 3 most common K8s networking mistakes in prod. The tell is `connection refused` vs `timeout` — refused means traffic reached the pod and was rejected, timeout means traffic never reached anything. Always match `targetPort` to the actual `containerPort` in the pod spec. And use **named ports** in real deployments to make this self-documenting and change-safe:

```yaml
# In the container spec
ports:
- name: http
  containerPort: 80

# In the Service spec
ports:
- port: 80
  targetPort: http   # Reference by name, not number — survives port changes
```

---

## Scenario 03 — DNS Resolution Failure Inside Cluster

### What You'll Break

DNS is how pods find Services by name. CoreDNS runs in `kube-system` and handles all in-cluster name resolution. When DNS breaks — or when you use the wrong DNS name format — everything that depends on service discovery fails silently. This scenario covers the most common DNS failure patterns.

### Setup

```bash
# Ensure backend-svc from Scenario 01 is running
kubectl get svc backend-svc -n lab03
# If not, recreate it first
```

### Break It — Wrong DNS Name Formats

```bash
# Spin up a debug pod to test DNS from inside the cluster
kubectl run dns-debug --image=busybox:1.35 --rm -it --restart=Never -n lab03 -- sh
```

Inside the pod, run these tests:

```sh
# Test 1 — Short name (works only within same namespace)
nslookup backend-svc
# Should resolve — same namespace search domain applies

# Test 2 — Wrong namespace in FQDN
nslookup backend-svc.default.svc.cluster.local
# NXDOMAIN — service is in lab03, not default

# Test 3 — Correct FQDN
nslookup backend-svc.lab03.svc.cluster.local
# Address: 10.96.x.x — resolves correctly

# Test 4 — Typo in service name
nslookup backend-svcc.lab03.svc.cluster.local
# NXDOMAIN — doesn't exist

# Test 5 — Understand your search domains
cat /etc/resolv.conf
# nameserver 10.96.0.10        <-- CoreDNS ClusterIP
# search lab03.svc.cluster.local svc.cluster.local cluster.local
# This is WHY short names work within same namespace

exit
```

### Investigate DNS Issues

```bash
# Step 1 — Is CoreDNS running?
kubectl get pods -n kube-system -l k8s-app=kube-dns
# NAME                   READY   STATUS    RESTARTS   AGE
# coredns-xxx-yyy        1/1     Running   0          6d

# Step 2 — Check CoreDNS logs for errors
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50

# Step 3 — Test DNS resolution from inside cluster
kubectl run dns-debug --image=busybox:1.35 --rm -it --restart=Never -n lab03 -- \
  nslookup kubernetes.default.svc.cluster.local
# If this resolves: CoreDNS is healthy, issue is specific to your service name/namespace

# Step 4 — Verify the service exists in the right namespace
kubectl get svc -A | grep backend-svc

# Step 5 — Check the pod's DNS config
kubectl exec -it $(kubectl get pod -n lab03 -l app=backend -o jsonpath='{.items[0].metadata.name}') \
  -n lab03 -- cat /etc/resolv.conf
```

### The DNS Name Hierarchy — Know This Cold

```
# Short name (same namespace only)
backend-svc

# Namespace-qualified (cross-namespace — use this in prod configs)
backend-svc.lab03

# Full FQDN (always works everywhere — most explicit)
backend-svc.lab03.svc.cluster.local

# Format:
# <service-name> . <namespace> . svc . <cluster-domain>
#   backend-svc  .    lab03    . svc . cluster.local
```

### Cross-Namespace DNS Test

```bash
# Create a service in a different namespace
kubectl create namespace lab03-db

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: lab03-db
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
EOF

# From a pod in lab03, resolve the service in lab03-db
kubectl run dns-debug --image=busybox:1.35 --rm -it --restart=Never -n lab03 -- \
  nslookup postgres.lab03-db.svc.cluster.local
# Resolves — cross-namespace DNS works with FQDN
# Short name "postgres" would NOT resolve here

# Cleanup
kubectl delete namespace lab03-db
```

### Prod Wisdom

Always use the **namespace-qualified name** (`service.namespace`) in your application configs — never the short name. Short names only work within the same namespace. The moment you reference a service cross-namespace, or your team moves a service to a different namespace, short names silently fail with `NXDOMAIN`. When DNS issues arise, the first check is always `kubectl get pods -n kube-system -l k8s-app=kube-dns`. If CoreDNS is `0/1` or `CrashLoopBackOff`, your entire cluster's service discovery is down.

---

## Scenario 04 — ClusterIP Not Reachable From Outside

### What You'll Break

A Service of type `ClusterIP` that someone is trying to reach from outside the cluster. `ClusterIP` is internal-only by design — it is not accessible from your laptop, CI/CD runner, or anywhere outside the cluster network. This is an extremely common confusion that wastes hours in dev and staging environments.

### Apply the ClusterIP Service

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: internal-api
  namespace: lab03
spec:
  type: ClusterIP              # Default — internal only
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 80
EOF

kubectl get svc internal-api -n lab03
# NAME           TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
# internal-api   ClusterIP   10.96.x.x      <none>        80/TCP    5s
```

`EXTERNAL-IP: <none>` is the signal. No external IP means no external access.

```bash
# Attempting to curl the ClusterIP from your machine (outside the cluster)
CLUSTER_IP=$(kubectl get svc internal-api -n lab03 -o jsonpath='{.spec.clusterIP}')
curl http://${CLUSTER_IP}
# curl: (7) Failed to connect — ClusterIP is not routable outside the cluster
```

### Fix — NodePort for Direct Node Access

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: external-api
  namespace: lab03
spec:
  type: NodePort
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080            # Fixed port on every node (range: 30000-32767)
EOF

kubectl get svc external-api -n lab03
# NAME           TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
# external-api   NodePort   10.96.x.x     <none>        80:30080/TCP   5s
```

### Accessing NodePort in KIND

KIND runs inside Docker — the node is a Docker container. To reach a NodePort:

```bash
# Get the KIND node IP
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[0].address}')
echo $NODE_IP

# Reach the NodePort
curl http://${NODE_IP}:30080
```

### The Universal Fix — port-forward

`kubectl port-forward` works with **any Service type** — ClusterIP, NodePort, LoadBalancer. It is your best tool for local testing and debugging without changing Service types:

```bash
# Forward local port 8080 to Service port 80
kubectl port-forward svc/internal-api 8080:80 -n lab03 &
curl http://localhost:8080
# Works — nginx response
kill %1

# Port-forward directly to a pod (bypasses Service entirely)
POD=$(kubectl get pod -n lab03 -l app=backend -o jsonpath='{.items[0].metadata.name}')
kubectl port-forward pod/${POD} 8081:80 -n lab03 &
curl http://localhost:8081
kill %1
```

### Service Types — Know the Difference

```
ClusterIP (default)
  Accessible: inside cluster only
  Use for: databases, internal APIs, caches

NodePort
  Accessible: via <NodeIP>:<NodePort> from outside
  Port range: 30000-32767
  Use for: dev/staging, direct node access

LoadBalancer
  Accessible: via cloud load balancer external IP
  Use for: prod external-facing services on cloud

ExternalName
  Accessible: maps to an external DNS CNAME
  Use for: referencing external services by name inside cluster
```

### Prod Wisdom

In prod, **never change a ClusterIP to NodePort just to debug it**. Use `kubectl port-forward` instead — it gives you direct access without touching the Service definition or opening firewall ports on nodes. NodePort in prod is a security risk because it opens a port on every node in the cluster. For real external traffic in prod, use LoadBalancer type (on cloud) or Ingress (covered in Lab 07).

---

## Scenario 05 — Service Exists But App Unreachable — Full Debug Flow

### What You'll Break

This scenario combines multiple issues and walks through the **complete systematic debug workflow** a senior engineer runs every time a service is unreachable. No shortcuts. No guessing. Methodical elimination every time.

### Apply a Broken Setup With Multiple Issues

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  namespace: lab03
spec:
  replicas: 2
  selector:
    matchLabels:
      app: payment
  template:
    metadata:
      labels:
        app: payment-svc        # label is "payment-svc"
    spec:
      containers:
      - name: payment
        image: nginx:1.25
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: payment-svc
  namespace: lab03
spec:
  selector:
    app: payment                # selector "payment" — pod label is "payment-svc" — MISMATCH
  ports:
  - port: 443
    targetPort: 8443            # container on 80, Service targeting 8443 — MISMATCH
EOF
```

### The Systematic Debug Workflow — Run Every Step

```bash
# STEP 1 — Are the pods actually running?
kubectl get pods -n lab03 -l app=payment-svc
# If pods are not Running: go back to Lab 01 first

# STEP 2 — Does the Service exist in the right namespace?
kubectl get svc payment-svc -n lab03
# If not found: Service not created or wrong namespace

# STEP 3 — Check Endpoints — this is the fork in the road
kubectl get endpoints payment-svc -n lab03
# <none>         → go to Step 4 (selector issue)
# <ip>:<port>    → go to Step 5 (port or app issue)

# STEP 4 — Selector mismatch: compare Service selector vs Pod labels
kubectl describe svc payment-svc -n lab03 | grep Selector
# Selector: app=payment

kubectl get pods -n lab03 --show-labels
# Labels: app=payment-svc   <-- MISMATCH found

# STEP 5 — Port check: compare Endpoint port vs container port
kubectl get endpoints payment-svc -n lab03
# 10.x.x.x:8443   <-- endpoint port is 8443

kubectl describe deployment payment-service -n lab03 | grep -A 2 "Port"
# Port: 80/TCP    <-- container is on 80   MISMATCH found

# STEP 6 — Isolate: can you reach the pod directly via port-forward?
POD=$(kubectl get pod -n lab03 -l app=payment-svc -o jsonpath='{.items[0].metadata.name}')
kubectl port-forward pod/${POD} 9090:80 -n lab03 &
curl http://localhost:9090
# If this works: pod is fine, Service config is the problem
# If this fails: pod itself is the problem
kill %1

# STEP 7 — DNS check from inside cluster
kubectl run dns-debug --image=busybox:1.35 --rm -it --restart=Never -n lab03 -- \
  nslookup payment-svc.lab03.svc.cluster.local
# Resolves = DNS fine, problem is connection layer
# Fails    = CoreDNS issue
```

### Fix Both Issues

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  namespace: lab03
spec:
  replicas: 2
  selector:
    matchLabels:
      app: payment
  template:
    metadata:
      labels:
        app: payment            # Fixed: matches selector
    spec:
      containers:
      - name: payment
        image: nginx:1.25
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: payment-svc
  namespace: lab03
spec:
  selector:
    app: payment                # Correct
  ports:
  - port: 443
    targetPort: 80              # Fixed: matches container port
EOF

# Verify end to end
kubectl get endpoints payment-svc -n lab03
# ENDPOINTS: 10.244.0.x:80, 10.244.0.y:80

kubectl port-forward svc/payment-svc 9091:443 -n lab03 &
curl http://localhost:9091
# nginx HTML — fully working
kill %1
```

### The Decision Tree — Memorise This

```
Service unreachable?
│
├── kubectl get endpoints <svc> -n <ns>
│   │
│   ├── ENDPOINTS: <none>
│   │   └── Selector mismatch
│   │       → kubectl describe svc  (see selector)
│   │       → kubectl get pods --show-labels  (see pod labels)
│   │       → Fix the selector to match the labels
│   │
│   └── ENDPOINTS: <ip>:<port>
│       │
│       ├── Port wrong?
│       │   → compare endpoint port vs containerPort in pod spec
│       │   → fix targetPort in Service
│       │
│       └── Port looks right?
│           │
│           ├── kubectl port-forward pod/<name> <local>:<containerPort>
│           │   ├── Works   → Service config issue or NetworkPolicy
│           │   └── Fails   → App not listening, probe issue, app bug
│           │
│           └── DNS resolves?
│               → nslookup <svc>.<ns>.svc.cluster.local
│               ├── NXDOMAIN  → service name/namespace wrong
│               └── Timeout   → CoreDNS down
```

---

## Key Commands Reference — Lab 03

```bash
# The single most important networking debug command
kubectl get endpoints <svc-name> -n <namespace>

# Service inspection
kubectl get svc -n <namespace>
kubectl describe svc <name> -n <namespace>

# Pod label check
kubectl get pods -n <namespace> --show-labels

# Port-forward (universal access — works for any Service type)
kubectl port-forward svc/<name> <local-port>:<svc-port> -n <namespace>
kubectl port-forward pod/<name> <local-port>:<container-port> -n <namespace>

# DNS debug from inside cluster
kubectl run dns-debug --image=busybox:1.35 --rm -it --restart=Never \
  -n <namespace> -- sh
# Inside: nslookup <service>.<namespace>.svc.cluster.local
# Inside: cat /etc/resolv.conf

# CoreDNS health
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50

# What is the container actually listening on?
kubectl exec -it <pod-name> -n <namespace> -- ss -tlnp
```

---

## Prod Wisdom — The Senior Engineer Mindset

Three things that separate senior engineers in networking incidents:

**1. They check Endpoints before anything else.** `kubectl get endpoints <svc>` splits every networking problem into two clean buckets in one second — selector issue or connection issue. Without this, you're guessing.

**2. They use port-forward to isolate the layer.** Pod port-forward bypasses the Service entirely. If the pod responds but the Service doesn't — the Service is misconfigured. If the pod doesn't respond — the problem is in the app. This one isolation step replaces 20 minutes of random debugging.

**3. They always use FQDNs in app config.** `service.namespace.svc.cluster.local`. Short names silently break when namespaces change. FQDNs always work. Write FQDNs in your configs, your Helm values, your environment variables — every time.

The networking debug order — burn this in:
```
Endpoints → Port match → Pod port-forward → Service port-forward → DNS → NetworkPolicy
```

---

## Cleanup

```bash
kubectl delete namespace lab03
```

---

*Lab 03 complete. Move to Lab 04 — Probes when ready.*
