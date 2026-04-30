# Kubernetes Services — ClusterIP, NodePort, LoadBalancer, Headless & Ingress
### The Complete Mental Model — Interviews · CKA · Production

---

## 0. First Principles — The Mental Model That Never Changes

Pods are ephemeral. They die, restart, and get rescheduled. Their IP addresses change every time. Services exist to solve exactly this problem:

```
Service = stable network identity for a dynamic set of pods
```

The three guarantees a Service provides:

```
1. Stable virtual IP (ClusterIP) — does not change for the life of the Service
2. Stable DNS name — <service-name>.<namespace>.svc.cluster.local
3. Load balancing — traffic distributed across all healthy, matching pods
```

The mechanism that connects a Service to its pods:

```
Service → label selector → Endpoints / EndpointSlices → Pod IPs
```

**The two truths that never change:**

1. A Service does not route traffic to pods. It routes traffic to **endpoints**, which are the pod IPs that match the selector. The Service and the Endpoints are separate objects.
2. If the selector matches no pods, the Endpoints object is empty. Traffic to that Service goes nowhere, silently. This is the most common breakage pattern.

The mental hierarchy:

```
External Client
      ↓
LoadBalancer / NodePort / Ingress   ← entry into cluster
      ↓
ClusterIP Service                   ← stable internal IP + DNS
      ↓
Endpoints / EndpointSlices          ← real pod IPs (auto-synced)
      ↓
Pods                                ← the actual workload
```

---

## 1. Reality Constraints — What Services Actually Do and Don't Do

### What Services Do

- Provide a stable virtual IP (ClusterIP) that persists regardless of pod restarts
- Register a DNS record so pods can find each other by name
- Load balance across all matching pods using kube-proxy rules
- Automatically update endpoints as pods are created, deleted, or become ready
- Expose pods externally via NodePort or LoadBalancer types

### What Services Do NOT Do

| Misconception | Reality |
|---|---|
| "Services route directly to pods" | No — they route to Endpoints (pod IPs), maintained as a separate object |
| "Changing a pod IP breaks the Service" | No — EndpointSlice controller updates automatically |
| "Services inspect HTTP traffic" | No — Services are L4 (TCP/UDP). For L7 routing, you need Ingress or a service mesh |
| "LoadBalancer type works everywhere" | No — requires a cloud provider or bare-metal LB controller (MetalLB) |
| "Ingress is a type of Service" | No — Ingress is a separate API resource that sits in front of Services |
| "ExternalName does DNS and proxying" | No — ExternalName is a pure CNAME alias. No proxying, no ClusterIP, no endpoints |
| "Headless services can't be accessed" | Wrong — they're accessed by pod DNS name directly, bypassing the virtual IP |
| "NodePort range is configurable" | Yes, but default is 30000–32767 — changing requires API server flag `--service-node-port-range` |
| "Service health-checks pods" | No — kube-proxy includes pod IPs in endpoints. Readiness probes control whether a pod IP appears in endpoints |

### The Readiness Probe → Endpoints Connection

```
Pod readinessProbe passes   →  EndpointSlice controller adds pod IP to endpoints
Pod readinessProbe fails    →  pod IP removed from endpoints → Service stops sending traffic to it
Pod deleted                 →  pod IP removed from endpoints immediately
```

This is why readiness probes are not optional in production. They are the mechanism by which Services know which pods are healthy.

### kube-proxy and the Virtual IP Reality

```
ClusterIP is NOT a real IP on any interface.
It exists only as an iptables/IPVS rule programmed by kube-proxy on every node.

When a pod sends traffic to 10.96.0.1:80 (ClusterIP):
  → The kernel intercepts the packet before it leaves the node
  → iptables DNAT rule rewrites destination to one of the actual pod IPs
  → Packet is forwarded to the real pod
  → Pod responds directly back to the client (SNAT applies)
```

---

## 2. Decision Logic — When to Use What

### Service Type Selection

```
Who needs to access this service?
          │
          ├── Only pods inside the cluster
          │         └── ClusterIP (default)
          │
          ├── External access needed + cloud environment
          │         └── LoadBalancer (gets a public IP from cloud)
          │
          ├── External access needed + bare-metal / dev / no cloud
          │         └── NodePort (access via NodeIP:30000-32767)
          │
          ├── Accessing an external system (outside cluster)
          │         └── ExternalName (DNS CNAME to external hostname)
          │
          └── StatefulSet / need direct pod addressing (DBs, Kafka)
                    └── Headless (clusterIP: None — returns pod IPs via DNS)
```

### HTTP/HTTPS Routing Decision

```
Need path-based or host-based routing? (e.g. /api → svc-a, /web → svc-b)
          │
          ├── Yes → Ingress (requires Ingress Controller — NGINX, Traefik, ALB)
          │
          └── No → LoadBalancer or NodePort (one IP/port per service)
```

### Full Service Type Comparison

| Type | Internal | External | IP Type | Port Range | Use Case | Key Limitation |
|---|---|---|---|---|---|---|
| `ClusterIP` | ✅ | ❌ | Virtual (iptables) | Any | Microservice-to-microservice | Not reachable outside cluster |
| `NodePort` | ✅ | ✅ | Node IP | 30000–32767 | Dev/test, bare-metal | Exposes port on ALL nodes |
| `LoadBalancer` | ✅ | ✅ | Public IP (cloud) | Any | Production external access | Costs money; one LB per service |
| `ExternalName` | ❌ | ✅ | DNS alias only | N/A | Alias to external service | No proxying, no TLS termination |
| `Headless` | ✅ | ❌ | None (`clusterIP: None`) | Any | StatefulSets, DBs, Kafka | No load balancing — direct pod DNS |
| `Ingress` | ✅ | ✅ | Public IP/DNS | 80/443 | HTTP/HTTPS routing | Not a Service; needs controller |

### Headless vs ClusterIP: When Each Wins

```
Use ClusterIP when:
  - You want load balancing across pods
  - Client doesn't need to know which pod it's talking to
  - Stateless services (web servers, API servers)

Use Headless (clusterIP: None) when:
  - You need to talk to a SPECIFIC pod (not any pod)
  - Application manages its own clustering (Kafka, Cassandra, etcd)
  - StatefulSet pods need stable DNS identities
  - DNS returns all pod IPs and client picks one (client-side LB)
```

---

## 3. Internal Working — How It Actually Happens

### How kube-proxy Programs Traffic Rules

```
1. Service created → API Server writes Service object to etcd
2. kube-proxy watches API Server for Service + EndpointSlice changes
3. kube-proxy programs rules on EVERY node in the cluster:

   iptables mode (default):
   → PREROUTING chain: DNAT rule intercepts packets to ClusterIP
   → Randomly selects one of the endpoint pod IPs
   → Rewrites destination IP to pod IP before packet leaves node

   IPVS mode (higher performance):
   → Creates virtual server in kernel IPVS table
   → Uses more efficient hash-based load balancing
   → Supports additional algorithms: rr, lc, dh, sh, sed, nq

   Cilium (eBPF, replaces kube-proxy entirely):
   → Programs eBPF maps in kernel — no iptables at all
   → Significantly lower latency, better observability
```

### EndpointSlice Controller — How Endpoints Stay in Sync

```
Pod created + readiness probe passes:
  1. EndpointSlice controller detects new pod
  2. Checks pod labels match Service selector
  3. Adds pod IP:port to EndpointSlice
  4. kube-proxy on all nodes picks up the change
  5. iptables rules updated — new pod starts receiving traffic

Pod deleted or readiness probe fails:
  1. Controller removes pod IP from EndpointSlice
  2. kube-proxy updates rules on all nodes
  3. Pod no longer receives new connections
  4. In-flight connections to that pod may still complete (graceful)
```

### NodePort Internal Flow

```
External Client → NodeIP:31080
        ↓
Node receives packet on port 31080
        ↓
kube-proxy iptables rule: DNAT 31080 → ClusterIP:80
        ↓
ClusterIP rule: DNAT → one of the endpoint pod IPs
        ↓
Pod receives traffic
        ↓
Response goes directly back to client (SNAT may apply)

IMPORTANT: The pod that handles traffic may NOT be on the node
that received the packet. kube-proxy routes across nodes.
This cross-node hop is controlled by:
  externalTrafficPolicy: Cluster (default — can route to any node)
  externalTrafficPolicy: Local   (only routes to pods on receiving node)
```

### LoadBalancer Internal Flow

```
Cloud LB (e.g. AWS NLB) → NodeIP:NodePort (on any node)
        ↓
kube-proxy rules on the receiving node
        ↓
DNAT to ClusterIP → DNAT to pod IP
        ↓
Pod handles request

The LoadBalancer type is just NodePort + cloud-controller-manager
creating a cloud load balancer pointing at the NodePort.
```

### Headless Service DNS — How StatefulSet Pods Get Stable Identity

```
Regular ClusterIP Service DNS:
  my-svc.my-ns.svc.cluster.local → 10.96.0.5 (virtual ClusterIP)
  ALL pods behind the service share this single IP

Headless Service DNS (clusterIP: None):
  my-headless.my-ns.svc.cluster.local → [10.0.1.5, 10.0.2.7, 10.0.3.9]
  Returns ALL pod IPs directly — no ClusterIP involved

StatefulSet Pod DNS (requires headless service):
  pod-0.my-headless.my-ns.svc.cluster.local → 10.0.1.5  (pod-0 only)
  pod-1.my-headless.my-ns.svc.cluster.local → 10.0.2.7  (pod-1 only)
  pod-2.my-headless.my-ns.svc.cluster.local → 10.0.3.9  (pod-2 only)

This stable DNS identity survives pod restarts — the IP may change
but pod-0.my-headless... always resolves to the current pod-0 IP.
```

### ExternalName — What It Actually Does

```
Service type ExternalName creates a CNAME record in CoreDNS:
  my-ext-svc.my-ns.svc.cluster.local → CNAME → db.external-corp.com

No ClusterIP. No iptables rules. No proxying.
The DNS resolution happens in the pod itself.
Pod resolves my-ext-svc → db.external-corp.com → actual IP.

Use case: abstract an external dependency behind a Kubernetes DNS name
so you can change the external endpoint by updating the Service,
not every application config file.
```

---

## 4. Hands-On — Production-Quality YAML + Commands

### Full Architecture: Frontend (NodePort) → Backend (ClusterIP)

```
External Client → NodePort:31xxx → Frontend (nginx) Pod
                                          ↓
                               ClusterIP Service (backend-svc)
                                          ↓
                               Backend (http-echo) Pod
```

### Backend Deployment (with required args)

```yaml
# backend-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  labels:
    app: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: http-echo
        image: hashicorp/http-echo
        args:
          - "-text=hello from backend"
          - "-listen=:8080"
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 3
          periodSeconds: 5
        resources:
          requests:
            cpu: 50m
            memory: 32Mi
          limits:
            cpu: 100m
            memory: 64Mi
```

### Backend ClusterIP Service

```yaml
# backend-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
spec:
  type: ClusterIP        # default — internal only
  selector:
    app: backend         # must match pod labels exactly
  ports:
  - name: http
    port: 8080           # port the Service listens on
    targetPort: 8080     # port on the Pod container
    protocol: TCP
```

### Frontend Deployment

```yaml
# frontend-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
```

### Frontend NodePort Service

```yaml
# frontend-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
  - name: http
    port: 80             # ClusterIP port (internal)
    targetPort: 80       # container port
    nodePort: 31080      # fixed NodePort (optional — omit to auto-assign 30000-32767)
    protocol: TCP
```

### LoadBalancer Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-lb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"  # AWS-specific
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
  externalTrafficPolicy: Local   # preserve client IP; only route to local pods
```

### Headless Service (for StatefulSet)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: db-headless
spec:
  clusterIP: None        # makes it headless
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
```

### ExternalName Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
  namespace: production
spec:
  type: ExternalName
  externalName: rds.us-east-1.amazonaws.com  # CNAME target
  # No selector, no ports, no ClusterIP
```

### Ingress Resource (L7 HTTP Routing)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-svc
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: backend-svc
            port:
              number: 8080
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls-secret
```

### Core Service Commands

```bash
# Create service imperatively (exam speed)
kubectl expose deployment backend \
  --name=backend-svc \
  --port=8080 \
  --target-port=8080 \
  --type=ClusterIP

kubectl expose deployment frontend \
  --name=frontend-svc \
  --port=80 \
  --target-port=80 \
  --type=NodePort

# Apply declaratively
kubectl apply -f backend-deploy.yaml
kubectl apply -f backend-svc.yaml
kubectl apply -f frontend-deploy.yaml
kubectl apply -f frontend-svc.yaml

# Inspect services
kubectl get svc
kubectl get svc -o wide
kubectl describe svc backend-svc

# Get NodePort number
kubectl get svc frontend-svc -o jsonpath='{.spec.ports[0].nodePort}'

# Get node IP
kubectl get nodes -o wide

# Get endpoints (exam critical)
kubectl get endpoints backend-svc
kubectl get endpointslices
kubectl describe endpointslices -l kubernetes.io/service-name=backend-svc

# Test internal service from temporary pod
kubectl run testpod --image=busybox -it --rm -- sh
# Inside: wget -qO- http://backend-svc:8080
# Inside: wget -qO- http://backend-svc.default.svc.cluster.local:8080

# Test from inside a running pod
kubectl exec -it deploy/frontend -- bash
# Inside: curl http://backend-svc:8080

# Access NodePort externally
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
NODE_PORT=$(kubectl get svc frontend-svc -o jsonpath='{.spec.ports[0].nodePort}')
curl http://$NODE_IP:$NODE_PORT
```

---

## 5. Production Flow — Real-World Architecture and Design Patterns

### Standard Three-Tier Architecture

```
Internet
    ↓
[Cloud LoadBalancer]          ← provisioned by LoadBalancer Service or Ingress
    ↓
[Ingress Controller]          ← NGINX, Traefik, AWS ALB — does L7 routing
  /api → backend-svc
  /    → frontend-svc
    ↓
[ClusterIP Services]          ← internal stable endpoints
    ↓
[Pods]                        ← actual workloads
```

### Why Each Layer Exists

```
LoadBalancer / Ingress  → single entry point, TLS termination, routing
ClusterIP               → stable internal DNS, decouples pods from consumers
Readiness probes        → controls which pod IPs appear in Endpoints
```

### `externalTrafficPolicy` — The Production Decision

```yaml
externalTrafficPolicy: Cluster  # default
  → Traffic can hop to a pod on a different node
  → Client IP is NOT preserved (SNAT applied)
  → Load balanced evenly across all pods
  → Works even if no pod is on the receiving node

externalTrafficPolicy: Local
  → Traffic only routed to pods on the receiving node
  → Client IP IS preserved (no SNAT)
  → If no pod on receiving node → connection dropped
  → Requires pods on every node (use DaemonSet or affinity rules)
```

> Production rule: Use `Local` when you need client IP for logging/rate limiting. Use `Cluster` (default) when you need even traffic distribution and can accept SNAT.

### Session Affinity (Sticky Sessions)

```yaml
spec:
  sessionAffinity: ClientIP          # route same client IP to same pod
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800          # 3 hours default
```

> Note: Session affinity only works within a node in iptables mode. IPVS mode has more consistent behavior. For true sticky sessions at L7, use Ingress annotations.

### Multi-Port Service

```yaml
spec:
  ports:
  - name: http         # name is REQUIRED when multiple ports
    port: 80
    targetPort: 8080
  - name: metrics
    port: 9090
    targetPort: 9090
  - name: grpc
    port: 9000
    targetPort: 9000
```

### Named TargetPort (Best Practice)

```yaml
# In the Pod spec:
ports:
- name: http
  containerPort: 8080

# In the Service spec:
ports:
- port: 80
  targetPort: http     # references port by name, not number
                       # if container port number changes, Service doesn't need updating
```

### DNS Resolution in Practice

```
Pod in same namespace:
  curl http://backend-svc:8080
  → resolves to ClusterIP via CoreDNS

Pod in different namespace:
  curl http://backend-svc.production.svc.cluster.local:8080
  → must use FQDN

DNS search path (from /etc/resolv.conf in pod):
  search default.svc.cluster.local svc.cluster.local cluster.local
  → allows short names within same namespace

StatefulSet pod DNS:
  pod-0.my-headless-svc.default.svc.cluster.local
  pod-1.my-headless-svc.default.svc.cluster.local
```

---

## 6. Mistakes — What Actually Breaks in Real Systems

### Mistake 1 — Selector Mismatch (Most Common Production Break)

**What happens:** Service created, pods running, but no traffic gets through. Endpoints are empty.

**Root cause:** Label on the pod doesn't match the selector on the Service. Even a single typo (`App` vs `app`) causes this.

**Diagnosis + Fix:**
```bash
kubectl describe svc backend-svc | grep -A3 Selector
kubectl get pods --show-labels -l app=backend

# Compare exactly — case-sensitive
kubectl get svc backend-svc -o jsonpath='{.spec.selector}'
kubectl get pods -o jsonpath='{.items[*].metadata.labels}'

# Fix: update selector or pod label to match
kubectl label pod <pod-name> app=backend --overwrite
```

### Mistake 2 — Readiness Probe Failing → Empty Endpoints

**What happens:** Pods are Running but Service has no endpoints. Traffic goes nowhere.

**Root cause:** Pod containers are running but readiness probe is failing — so the pod IP is never added to Endpoints.

**Diagnosis + Fix:**
```bash
kubectl get endpoints backend-svc
# If "Endpoints: <none>" → probe failing or no pods match

kubectl describe pod <pod>
# Look for: "Readiness probe failed" in Events

kubectl get pod <pod> -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}'
# "False" = pod is running but not Ready
```

### Mistake 3 — Missing args on http-echo → CrashLoopBackOff

**What happens:** `hashicorp/http-echo` container crashes immediately. Pod enters CrashLoopBackOff.

**Root cause:** The http-echo image requires `-text` and `-listen` args. Without them, it exits with an error.

**Fix:**
```yaml
args:
  - "-text=hello from backend"
  - "-listen=:8080"
```

### Mistake 4 — Port Mismatch (port vs targetPort vs containerPort)

**What happens:** Service exists, endpoints are populated, but connections are refused.

**Root cause:** `targetPort` in the Service doesn't match `containerPort` in the pod.

**Diagnosis + Fix:**
```bash
# Check what port the container actually listens on
kubectl describe pod <pod> | grep -A5 Ports

# Check Service targetPort
kubectl describe svc backend-svc | grep -A5 Port

# They must match. Fix by updating the Service:
kubectl patch svc backend-svc -p '{"spec":{"ports":[{"port":8080,"targetPort":8080}]}}'
```

### Mistake 5 — Using ExternalName and Expecting Traffic to Work Like a Proxy

**What happens:** Team creates ExternalName service expecting it to proxy traffic to an external DB. Connection times out or SSL errors appear.

**Root cause:** ExternalName is just a DNS CNAME — it returns the external hostname for the pod to resolve. The pod's application must handle the actual connection including TLS to the external service. ExternalName does not add ClusterIP, does not do TLS termination, does not handle port mapping.

**Fix:** For actual proxying to external endpoints, use a Service with no selector and manually manage Endpoints:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  ports:
  - port: 5432
    targetPort: 5432
---
apiVersion: v1
kind: Endpoints
metadata:
  name: external-db      # must match Service name
subsets:
- addresses:
  - ip: 203.0.113.42     # actual external IP
  ports:
  - port: 5432
```

### Mistake 6 — NodePort Blocked by Firewall / Security Group

**What happens:** NodePort service created, correct port confirmed, but external access times out.

**Root cause:** Cloud security group or on-prem firewall blocks the NodePort range (30000–32767) on worker nodes.

**Fix:**
```bash
# Verify NodePort is assigned
kubectl get svc frontend-svc -o jsonpath='{.spec.ports[0].nodePort}'

# Test locally on the node first
curl localhost:<nodeport>

# If that works → firewall issue. Open the NodePort in the security group:
# AWS: Edit inbound rules on worker node security group
# GCP: gcloud compute firewall-rules create allow-nodeport --allow=tcp:30000-32767
```

### Mistake 7 — Service in Wrong Namespace

**What happens:** Pod calls `http://backend-svc:8080` but gets DNS NXDOMAIN. Backend Service exists but in a different namespace.

**Root cause:** Short DNS names only resolve within the same namespace.

**Fix:**
```bash
# Use FQDN for cross-namespace communication
curl http://backend-svc.backend-namespace.svc.cluster.local:8080

# Or move services to the same namespace
# Or create an ExternalName service in the calling namespace
```

### Mistake 8 — `externalTrafficPolicy: Local` with No Local Pods

**What happens:** LoadBalancer or NodePort with `externalTrafficPolicy: Local`. Some nodes have no matching pods. Connections to those nodes are dropped.

**Root cause:** `Local` policy only routes to pods on the same node. Nodes without a matching pod drop the connection.

**Fix:**
```bash
# Check pod distribution
kubectl get pods -o wide

# Either:
# 1. Switch to Cluster policy (accept SNAT trade-off)
kubectl patch svc frontend-svc -p '{"spec":{"externalTrafficPolicy":"Cluster"}}'
# 2. Ensure pods run on every node (DaemonSet or pod anti-affinity)
```

---

## 7. Interview Answers — Compressed, Verbatim-Ready

### Q: What is a Kubernetes Service and why does it exist?

> "Pods are ephemeral — they die and restart with new IP addresses. A Service provides a stable virtual IP and DNS name that persists regardless of what happens to the underlying pods. When you send traffic to a Service, it uses a label selector to find the matching pods, maintains an Endpoints object with their current IPs, and uses kube-proxy rules on every node to actually route the traffic. Without Services, every consumer would need to know and track the constantly-changing IP addresses of every pod."

### Q: What is the difference between ClusterIP, NodePort, and LoadBalancer?

> "All three build on each other. ClusterIP is the base — it's a virtual IP that's only reachable from inside the cluster, used for service-to-service communication. NodePort adds external reachability by opening a port in the range 30000 to 32767 on every node in the cluster — external traffic hits any node IP on that port and gets forwarded to the service. LoadBalancer builds on top of NodePort by also provisioning a cloud load balancer that points at the NodePort — you get a single public IP that hides the node-level complexity. In practice, ClusterIP is for internal microservices, NodePort is for dev or bare-metal, and LoadBalancer is for production external access in the cloud."

### Q: What is a headless service and when would you use it?

> "A headless service is created by setting `clusterIP: None`. It doesn't get a virtual IP or kube-proxy rules. Instead, when you do a DNS lookup, CoreDNS returns all the individual pod IPs directly. You use this primarily with StatefulSets, where each pod needs a stable DNS identity — things like databases, Kafka, or Zookeeper where the application itself manages clustering and needs to talk to specific peers. With a headless service and a StatefulSet named 'kafka', pod-0 is reachable at `kafka-0.kafka-headless.namespace.svc.cluster.local`, which survives pod restarts."

### Q: How does a Service actually route traffic under the hood?

> "A Service's virtual IP — the ClusterIP — isn't a real IP on any network interface. It only exists as an iptables or IPVS rule that kube-proxy programs on every node. When a pod sends a packet to the ClusterIP, the kernel intercepts it via the PREROUTING chain, rewrites the destination IP to one of the actual pod IPs using DNAT, and then forwards it. kube-proxy watches the API Server for changes to Services and EndpointSlices and updates these rules on all nodes whenever pods are added or removed. In clusters using Cilium, eBPF maps replace iptables entirely for better performance."

### Q: What is the difference between Ingress and a Service?

> "A Service operates at Layer 4 — TCP and UDP. It can route to pods but it has no understanding of HTTP, hostnames, or URL paths. Ingress operates at Layer 7 — it understands HTTP and HTTPS. It allows you to do host-based routing, path-based routing, and TLS termination all through a single external IP, routing to multiple backend Services based on rules. Ingress itself is just a configuration resource — you also need an Ingress Controller, like NGINX or Traefik, that reads those rules and does the actual routing. Without a controller, an Ingress resource has no effect."

### Q: What happens if a Service has no endpoints?

> "If the selector on the Service doesn't match any pod labels, or if all matching pods have failing readiness probes, the Endpoints object is empty. Traffic sent to the Service's ClusterIP is accepted by the iptables rule — so it doesn't fail at the network level — but there's no backend to forward it to, so the connection hangs or times out. This is a silent failure that's easy to miss. The correct way to diagnose it is to run `kubectl get endpoints <service-name>` and check whether pod IPs are listed."

### Q: What is `externalTrafficPolicy: Local` and when do you use it?

> "By default, NodePort and LoadBalancer services use `externalTrafficPolicy: Cluster`, which means traffic hitting any node can be forwarded to a pod on a completely different node. This performs source NAT so the original client IP is lost. Setting it to `Local` changes the behavior — traffic is only routed to pods on the same node that received the request. This preserves the original client IP, which you need for things like rate limiting or geo-based access control. The trade-off is that nodes without a matching pod will drop connections, so you need to ensure pod distribution matches your traffic routing."

### Q: Walk me through what happens when pod A calls `http://backend-svc:8080`.

> "Pod A sends the DNS lookup for `backend-svc` to its configured nameserver, which is CoreDNS running in the cluster. CoreDNS resolves it to the ClusterIP of the backend-svc Service — say 10.96.50.100. Pod A sends the HTTP request to 10.96.50.100:8080. The packet leaves the pod's network namespace and hits the node's iptables PREROUTING chain. kube-proxy has programmed a DNAT rule that intercepts packets to 10.96.50.100:8080 and rewrites the destination to one of the actual pod IPs in the Endpoints list — say 10.0.2.5:8080. The packet is forwarded to that pod. The backend pod processes the request and responds directly back to pod A's IP."

---

## 8. Debugging — Fast Diagnosis Paths

### Service Not Working — Primary Triage

```
Traffic not reaching pods through Service
        │
        ├── Step 1: Check endpoints
        │   kubectl get endpoints <svc-name>
        │   kubectl get endpointslices -l kubernetes.io/service-name=<svc-name>
        │   │
        │   ├── Endpoints EMPTY → selector mismatch or readiness probe failing
        │   │         │
        │   │         ├── Check selector vs pod labels
        │   │         │   kubectl describe svc <svc> | grep Selector
        │   │         │   kubectl get pods --show-labels
        │   │         │
        │   │         └── Check readiness
        │   │             kubectl describe pod <pod> | grep -A5 Readiness
        │   │             kubectl get pod <pod> -o jsonpath='{.status.conditions}'
        │   │
        │   └── Endpoints POPULATED → routing or port issue
        │             │
        │             ├── Check targetPort matches container port
        │             │   kubectl describe svc <svc> | grep Port
        │             │   kubectl describe pod <pod> | grep Port
        │             │
        │             └── Test direct pod IP (bypass Service)
        │                 kubectl run tmp --image=busybox -it --rm -- \
        │                   wget -qO- http://<pod-ip>:8080
        │                 If this works → Service routing issue
        │                 If this fails → app/container issue
        │
        └── Step 2: Test Service DNS
            kubectl run tmp --image=busybox -it --rm -- sh
            nslookup backend-svc          # same namespace
            nslookup backend-svc.default.svc.cluster.local
            # NXDOMAIN → Service doesn't exist or wrong namespace
            # Returns IP → DNS works, issue is elsewhere
```

### NodePort Not Reachable from Outside

```bash
# Step 1: Verify NodePort assigned
kubectl get svc <n> -o jsonpath='{.spec.ports[0].nodePort}'

# Step 2: Get node IP
kubectl get nodes -o wide

# Step 3: Test from the node itself (eliminates firewall)
ssh <node>
curl localhost:<nodeport>

# If works on node but not externally → firewall/SG blocking
# If fails on node → check kube-proxy and endpoints

# Step 4: Check kube-proxy logs
kubectl logs -n kube-system -l k8s-app=kube-proxy --tail=50

# Step 5: Verify iptables rules exist on node
iptables -t nat -L KUBE-SERVICES | grep <svc-name>
```

### DNS Resolution Failing Inside Pod

```bash
# Test DNS from inside a pod
kubectl run tmp --image=busybox -it --rm -- sh
nslookup kubernetes.default          # test cluster DNS works at all
nslookup backend-svc                 # test same-namespace resolution
nslookup backend-svc.other-ns.svc.cluster.local  # cross-namespace

# Check CoreDNS is running
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns

# Check pod's /etc/resolv.conf
kubectl exec <pod> -- cat /etc/resolv.conf
# Should show: nameserver pointing to CoreDNS ClusterIP
#              search <namespace>.svc.cluster.local svc.cluster.local cluster.local
```

### Endpoints Populated but Traffic Still Failing

```bash
# Test bypassing the Service entirely — hit pod IP directly
kubectl get endpoints <svc-name>
# Note a pod IP

kubectl run tmp --image=busybox -it --rm -- wget -qO- http://<pod-ip>:<port>

# If direct pod works but Service doesn't:
# → kube-proxy issue or iptables problem

# Check iptables DNAT rules for the service
kubectl get svc <svc> -o jsonpath='{.spec.clusterIP}'
iptables -t nat -L KUBE-SERVICES | grep <clusterIP>
iptables -t nat -L KUBE-SVC-<hash> -n  # check backends

# Restart kube-proxy if rules are missing
kubectl rollout restart daemonset kube-proxy -n kube-system
```

### Service-to-Service Cross-Namespace Failure

```bash
# Confirm service exists in expected namespace
kubectl get svc -A | grep backend-svc

# Test with full FQDN
kubectl exec -it <pod> -- curl http://backend-svc.production.svc.cluster.local:8080

# Check NetworkPolicy isn't blocking
kubectl get networkpolicy -A
kubectl describe networkpolicy <policy> -n <namespace>
```

---

## 9. Kill Switch — 10-Second Recall

```
Service = stable IP + DNS for dynamic pods
Traffic path: Service → Endpoints → Pod IPs (via iptables/IPVS)

ClusterIP     → internal only, default
NodePort      → NodeIP:30000-32767, external, any cloud
LoadBalancer  → cloud LB + NodePort, public IP
ExternalName  → CNAME alias only, no proxying
Headless      → clusterIP: None, returns pod IPs directly (StatefulSet)
Ingress       → NOT a Service; L7 HTTP routing, needs a controller

Endpoints empty?  → selector mismatch OR readiness probe failing
Port mismatch?    → targetPort ≠ containerPort
External blocked? → firewall / security group on NodePort range
Cross-namespace?  → use FQDN: svc.namespace.svc.cluster.local
ExternalName?     → no ClusterIP, no proxying, CNAME only

Debug order:
  1. kubectl get endpoints <svc>
  2. kubectl get pods --show-labels
  3. kubectl run testpod --image=busybox -it --rm -- wget <svc>:<port>
  4. Check port numbers match end-to-end
```

---

## 10. Appendix — Quick Reference Card

### Service Type Decision Table

| Need | Type |
|---|---|
| Pod-to-pod (same cluster) | `ClusterIP` |
| External access (cloud) | `LoadBalancer` |
| External access (bare-metal/dev) | `NodePort` |
| External DNS alias | `ExternalName` |
| StatefulSet / direct pod DNS | `Headless (clusterIP: None)` |
| L7 HTTP/HTTPS routing | `Ingress` |

### Port Terminology (Exam Critical)

| Field | Defined In | Meaning |
|---|---|---|
| `containerPort` | Pod spec | Port the application inside the container listens on |
| `targetPort` | Service spec | Port on the pod to forward to — must match `containerPort` |
| `port` | Service spec | Port the Service itself listens on (ClusterIP:port) |
| `nodePort` | Service spec | Port opened on every node (NodePort/LB type only, 30000–32767) |

### Imperative Service Commands

```bash
# ClusterIP
kubectl expose deployment <n> --port=<port> --target-port=<tp> --type=ClusterIP

# NodePort
kubectl expose deployment <n> --port=<port> --target-port=<tp> --type=NodePort

# LoadBalancer
kubectl expose deployment <n> --port=<port> --target-port=<tp> --type=LoadBalancer

# Generate YAML without applying
kubectl expose deployment <n> --port=80 --type=ClusterIP --dry-run=client -o yaml
```

### DNS Name Formats

```bash
# Same namespace (short name works)
http://backend-svc:8080

# Cross-namespace (FQDN required)
http://backend-svc.production.svc.cluster.local:8080

# Headless — all pod IPs
backend-headless.default.svc.cluster.local → [10.0.1.5, 10.0.2.7, ...]

# StatefulSet pod DNS (requires headless service)
pod-0.backend-headless.default.svc.cluster.local
pod-1.backend-headless.default.svc.cluster.local
```

### Endpoint Debug Commands

```bash
kubectl get endpoints <svc-name>
kubectl get endpointslices -l kubernetes.io/service-name=<svc-name>
kubectl describe endpointslices -l kubernetes.io/service-name=<svc-name>

# Populated = Service can route
# Empty     = selector mismatch or readiness probe failing
```

### kube-proxy Mode Reference

| Mode | Mechanism | Default | Performance | Notes |
|---|---|---|---|---|
| `iptables` | Kernel netfilter rules | Most clusters | Good | Random selection per connection |
| `ipvs` | Kernel IPVS virtual server | Some clusters | Better | Multiple LB algorithms |
| `eBPF (Cilium)` | BPF maps in kernel | Cilium clusters | Best | Replaces kube-proxy entirely |

### NetworkPolicy Impact on Services

```bash
# Services bypass NetworkPolicy for pod-to-pod routing IF:
# → No NetworkPolicy selects the target pod (default-allow)

# Services DO NOT bypass NetworkPolicy IF:
# → A NetworkPolicy exists that selects the target pod
# → The NetworkPolicy doesn't allow ingress from the source

# Always check NetworkPolicy when endpoints are populated but traffic fails
kubectl get networkpolicy -A
kubectl describe networkpolicy <policy>
```

### externalTrafficPolicy Comparison

| Policy | Client IP Preserved | Cross-node hop | Drop if no local pod |
|---|---|---|---|
| `Cluster` (default) | ❌ (SNAT applied) | ✅ | ❌ (routes elsewhere) |
| `Local` | ✅ (no SNAT) | ❌ | ✅ (connection dropped) |

---

