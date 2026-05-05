# 🚪 Kubernetes Gateway API 

> *"Ingress was a door. Gateway API is an entire lobby with a concierge, access cards, and a sign-in sheet."*

---

## 0. First Principles — The Mental Model That Never Changes

> Before you memorize a single YAML key, burn these truths into your brain. Everything else is derived from them.

### The Three Laws of Gateway API

**Law 1: Separation of Concerns is the Whole Point**
Infrastructure (what handles traffic) and routing (where traffic goes) must be owned by different people, defined in different resources, and governed by different access controls. This is not a preference — it's the architectural DNA of the entire API.

**Law 2: The Spec IS the Feature Set**
In Ingress, features lived in annotations — unvalidated, controller-specific, non-portable. In Gateway API, every capability (weighted routing, header matching, retries, timeouts) is part of the API specification itself. The API server validates it. Any conformant controller must implement it.

**Law 3: Controllers Write Config; Data Planes Move Packets**
The controller (a pod in your cluster) watches `GatewayClass`, `Gateway`, and `Route` objects. It then configures a data plane — either a cloud load balancer it doesn't sit in front of, or in-cluster proxy pods it manages. **The controller is never in the hot path for traffic.**

### The Immutable Hierarchy

```
GatewayClass  ──► "What KIND of gateway am I?"  (Infrastructure Provider owns)
     │
     ▼
  Gateway     ──► "I AM a gateway, here are my ports"  (Cluster Operator owns)
     │
     ▼
  HTTPRoute   ──► "Route /iphone to iphone-svc"  (App Developer owns)
```

This maps exactly to org roles. GatewayClass is platform policy. Gateway is infrastructure provisioning. Routes are day-to-day application config. **These are designed to be owned by different teams with different RBAC.**

---

## 1. Reality Constraints — What Kubernetes Actually Does and Doesn't Do

### What Gateway API Is

| Fact | Detail |
|---|---|
| **Origin** | Kubernetes SIG-Network (community standard, not a vendor product) |
| **Delivery mechanism** | CRDs — must be installed; not built into Kubernetes by default |
| **Stability** | `gateway.networking.k8s.io/v1` is GA (stable) for HTTPRoute, Gateway, GatewayClass |
| **Controller requirement** | A compatible controller MUST be installed or nothing happens |
| **Validation** | Spec-defined fields are validated by the API server (unlike Ingress annotations) |

### What Gateway API Is NOT

| Misconception | Reality |
|---|---|
| "It replaces Ingress controllers" | No — NGINX, Envoy, and cloud LBs are still data planes; they just implement a different API |
| "It's available by default" | No — you must install CRDs + a compatible controller |
| "Annotations are gone" | Controllers may still have extension annotations; the *core features* no longer need them |
| "One controller for everything" | Each `GatewayClass` points to exactly one controller; you can have multiple classes |
| "TCPRoute/UDPRoute are everywhere" | Support varies by controller — NGINX Gateway Fabric, for example, supports only HTTPRoute and GRPCRoute today |

### Protocol Support Matrix

| Protocol | Route Kind | Notes |
|---|---|---|
| HTTP/HTTPS | `HTTPRoute` | GA, universally supported |
| gRPC | `GRPCRoute` | GA, supported by NGF and others |
| TLS passthrough | `TLSRoute` | Beta — SNI-based, gateway doesn't decrypt |
| TCP | `TCPRoute` | Experimental — L4 routing |
| UDP | `UDPRoute` | Experimental — DNS, syslog use cases |

### The Annotation Problem Gateway API Solves

```
Ingress + NGINX annotation world:
  nginx.ingress.kubernetes.io/rewrite-target: /
  alb.ingress.kubernetes.io/actions.forward: '{"targetGroups":[...]}'

Problems:
  ✗ Not validated by Kubernetes API server
  ✗ Typos fail silently
  ✗ Moving to a different controller = rewrite all annotations
  ✗ No RBAC at the feature level

Gateway API world:
  All of the above in spec fields — validated, portable, RBAC-able
```

---

## 2. Decision Logic — When to Use What

### Ingress vs Gateway API Decision Tree

```
Do you need multi-namespace routing from a single LB?
  ├─ YES ─► Gateway API (or controller-specific IngressGroup, but that's not portable)
  └─ NO ─► Continue...

Do you need TCP, UDP, or gRPC routing?
  ├─ YES ─► Gateway API
  └─ NO ─► Continue...

Do you need weighted traffic splitting or canary deployments in the spec (not annotations)?
  ├─ YES ─► Gateway API
  └─ NO ─► Continue...

Do you need different teams to manage their own routing rules independently?
  ├─ YES ─► Gateway API (separate HTTPRoute objects per team/namespace)
  └─ NO ─► Ingress is probably fine for simple HTTP routing
```

### Cloud-Managed vs In-Cluster Data Plane

| Scenario | Controller Location | Data Plane | When to Choose |
|---|---|---|---|
| **Cloud-Managed** | In-cluster pod | External cloud LB (ALB, GCLB, Azure App GW, VPC Lattice) | Cloud-native, managed scaling, no proxy pods to maintain |
| **In-Cluster** | In-cluster pod | Proxy pods (NGINX, Envoy, HAProxy, Istio) | On-prem, hybrid, need L7 inspection, cost control |

### Which Controller for Which Scenario?

| Controller | Data Plane | Best For |
|---|---|---|
| AWS Gateway API Controller | VPC Lattice | AWS-native, service mesh-like East-West traffic |
| AWS LBC (Ingress or GW mode) | ALB/NLB | Path/host routing on EKS |
| GKE Gateway Controller | Google Cloud LB | GKE clusters, multi-cluster |
| NGINX Gateway Fabric | In-cluster NGINX pods | On-prem, KIND, simple HTTP/gRPC |
| Envoy Gateway | In-cluster Envoy pods | Feature-rich, Envoy-based, open standard |
| Istio | In-cluster Envoy (sidecar) | Service mesh + ingress in one |

### Route Attachment Rules

```
HTTPRoute attaches to a Gateway listener when ALL of these are true:
  1. parentRefs[].name + namespace match the Gateway
  2. parentRefs[].sectionName matches the listener name
  3. The Gateway's allowedRoutes.namespaces allows the Route's namespace
  4. The controller accepts the HTTPRoute (reports Accepted=True)
```

---

## 3. Internal Working — How It Actually Happens Under the Hood

### The Full Reconciliation Loop (Step by Step)

```
Step 1: You install Gateway API CRDs
        → API server now knows about GatewayClass, Gateway, HTTPRoute etc.
        → Nothing runs yet; just schema definitions

Step 2: You install a controller (e.g., NGINX Gateway Fabric via Helm)
        → Controller pod starts
        → It calls List+Watch on GatewayClass, Gateway, HTTPRoute, GRPCRoute
        → It creates a GatewayClass resource pointing to itself
           (controller name: gateway.nginx.org/nginx-gateway-controller)

Step 3: You apply a Gateway resource
        → Controller's informer cache sees the new object
        → Controller checks: does this Gateway reference MY GatewayClass?
          ├─ YES → reconcile it (configure data plane, update status)
          └─ NO → ignore it
        → Controller programs the data plane:
            Cloud-managed: calls cloud API to create/configure LB
            In-cluster: writes nginx.conf / Envoy xDS config; proxy pods reload
        → Controller updates Gateway.status.conditions:
            Programmed=True, Ready=True (or similar, varies by controller)

Step 4: You apply HTTPRoute resources
        → Controller sees new HTTPRoute objects
        → Validates parentRefs (Gateway exists? Listener matches? Namespace allowed?)
        → If valid: pushes routing rules to data plane
        → Updates HTTPRoute.status.parents[].conditions:
            Accepted=True, ResolvedRefs=True

Step 5: Traffic flows
        Client → Data Plane (cloud LB or proxy pod)
        Data Plane matches request against programmed routing rules
        Data Plane forwards to backend Service → Pod
        Controller is NOT involved in live traffic
```

### Control Plane vs Data Plane — Concrete Example with NGF

```
Control Plane (nginx-gateway-fabric pod):
  - Watches k8s API
  - Translates Gateway + HTTPRoute → nginx config
  - Writes config to proxy pods via shared volume or xDS
  - Never touches a packet

Data Plane (nginx proxy pod):
  - Reads nginx.conf written by controller
  - Accepts TCP connections on port 80/443
  - Matches URL path → routes to backend Service ClusterIP
  - Returns response to client
```

### ReferenceGrant: The Cross-Namespace Safety Valve

Without `ReferenceGrant`, an HTTPRoute in `namespace-a` cannot reference a Service in `namespace-b`. This prevents namespace escape vulnerabilities.

```yaml
# Lives in the TARGET namespace (where the Service is)
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-from-app1-ns
  namespace: backend-ns       # namespace of the Service being referenced
spec:
  from:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    namespace: app1-ns        # namespace of the HTTPRoute that wants to reach here
  to:
  - group: ""
    kind: Service
```

> 🔐 Think of it as: "I (backend-ns) grant permission to HTTPRoutes from app1-ns to reference my Services."

---

## 4. Hands-On — Production-Quality YAML + Commands

### Full Demo Stack (KIND + NGINX Gateway Fabric)

#### Step 1: KIND Cluster with Port Mapping

```yaml
# 00-kind-cluster.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  image: kindest/node:v1.33.1@sha256:050072256b9a903bd914c0b2866828150cb229cea0efe5892e2b644d5dd3b34f
  extraPortMappings:
  - containerPort: 31000   # NodePort inside the KIND container
    hostPort: 31000        # Your laptop's port → traffic gets forwarded in
```

```bash
kind delete cluster --name=<existing> 2>/dev/null || true
kind create cluster --name=gateway-api --config=00-kind-cluster.yaml
```

#### Step 2: Install Gateway API CRDs

```bash
# Check the latest version at https://docs.nginx.com/nginx-gateway-fabric/install/helm/
# Replace v2.1.0 with the latest version listed there
kubectl kustomize \
  "https://github.com/nginx/nginx-gateway-fabric/config/crd/gateway-api/standard?ref=v2.1.0" \
  | kubectl apply -f -

# Verify CRDs installed
kubectl api-resources | grep gateway
```

> ⚠️ **Version lock warning:** CRD version must match controller version. Installing old CRDs with a new controller can crash the controller or produce missing resources like `BackendTLSPolicy`.

#### Step 3: Install NGINX Gateway Fabric Controller

```bash
helm install ngf oci://ghcr.io/nginx/charts/nginx-gateway-fabric \
  --create-namespace -n ngf-gatewayapi-ns \
  --set nginx.service.type=NodePort \
  --set-json 'nginx.service.nodePorts=[{"port":31000,"listenerPort":80}]'

# Verify controller deployed
kubectl get deploy -n ngf-gatewayapi-ns

# GatewayClass auto-created by the controller
kubectl get gatewayclasses.gateway.networking.k8s.io -o wide
# NAME    CONTROLLER                                    ACCEPTED
# nginx   gateway.nginx.org/nginx-gateway-controller   True
```

#### Step 4: Namespace + Application Deployments

```yaml
# 01-ns.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: app1-ns
```

```yaml
# 02-iphone.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: iphone-deploy
  namespace: app1-ns
spec:
  replicas: 2
  selector:
    matchLabels:
      app: iphone-page
  template:
    metadata:
      labels:
        app: iphone-page
    spec:
      containers:
      - name: python-http
        image: python:alpine
        command: ["/bin/sh", "-c"]
        args:
        - |
          mkdir -p /iphone && echo '<html><body><h1>iPhone Users</h1></body></html>' \
          > /iphone/index.html && cd / && python3 -m http.server 5678
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: iphone-svc
  namespace: app1-ns
spec:
  selector:
    app: iphone-page
  ports:
  - protocol: TCP
    port: 80
    targetPort: 5678
```

> Repeat the same pattern for `android-svc` (path `/android`) and `desktop-svc` (path `/`).

#### Step 5: The Gateway

```yaml
# 05-gateway.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: gateway
  namespace: ngf-gatewayapi-ns    # Lives with the controller, owned by Cluster Operator
spec:
  gatewayClassName: nginx          # References the GatewayClass → tells controller "this is yours"
  listeners:
  - name: http                     # Listener name — referenced by HTTPRoute.parentRefs.sectionName
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: All                  # Routes from ANY namespace can attach (tune per env)
```

```bash
kubectl apply -f 05-gateway.yaml
kubectl get gateway -n ngf-gatewayapi-ns
kubectl describe gateway gateway -n ngf-gatewayapi-ns  # look for Programmed=True
kubectl get svc -n ngf-gatewayapi-ns                   # look for NodePort 31000
```

#### Step 6: HTTPRoutes

```yaml
# 06-routes.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: iphone-routes
  namespace: app1-ns               # App Developer's namespace
spec:
  parentRefs:
  - name: gateway                  # Gateway to attach to
    namespace: ngf-gatewayapi-ns
    sectionName: http              # MUST match listener name in Gateway spec
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /iphone
    backendRefs:
    - name: iphone-svc
      port: 80
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: android-routes
  namespace: app1-ns
spec:
  parentRefs:
  - name: gateway
    namespace: ngf-gatewayapi-ns
    sectionName: http
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /android
    backendRefs:
    - name: android-svc
      port: 80
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: desktop-routes
  namespace: app1-ns
spec:
  parentRefs:
  - name: gateway
    namespace: ngf-gatewayapi-ns
    sectionName: http
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /               # Catch-all default route
    backendRefs:
    - name: desktop-svc
      port: 80
```

```bash
kubectl apply -f 06-routes.yaml
kubectl get httproutes -n app1-ns
kubectl describe httproutes.gateway.networking.k8s.io iphone-routes -n app1-ns
# Look for: Accepted=True, ResolvedRefs=True
```

#### Step 7: Verify Connectivity

```bash
# These should all return HTML
curl http://localhost:31000/iphone
curl http://localhost:31000/android
curl http://localhost:31000/

# Watch proxy logs to confirm traffic hits the data plane
kubectl logs -f -n ngf-gatewayapi-ns \
  $(kubectl get pods -n ngf-gatewayapi-ns -o name | grep nginx | head -1)
```

### Advanced YAML Patterns

#### Weighted Traffic Splitting (Canary)

```yaml
# Split 90% to stable, 10% to canary — no annotations needed
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: canary-route
  namespace: my-app-ns
spec:
  parentRefs:
  - name: gateway
    namespace: gateway-ns
    sectionName: http
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
    backendRefs:
    - name: api-stable-svc
      port: 80
      weight: 90
    - name: api-canary-svc
      port: 80
      weight: 10
```

#### Header-Based Routing

```yaml
rules:
- matches:
  - headers:
    - name: X-User-Type
      value: beta-tester
  backendRefs:
  - name: beta-svc
    port: 80
- backendRefs:                  # default fallback (no match condition)
  - name: stable-svc
    port: 80
```

#### HTTPRoute with Timeouts and Retries

```yaml
rules:
- matches:
  - path:
      type: PathPrefix
      value: /slow-api
  timeouts:
    request: 10s               # Fail request if backend doesn't respond in 10s
    backendRequest: 8s         # Per-attempt timeout
  backendRefs:
  - name: slow-api-svc
    port: 80
```

---

## 5. Production Flow — Real-World Architecture and Design Patterns

### Pattern 1: Multi-Tenant Platform (The Classic Use Case)

```
Platform Team owns:
  GatewayClass (nginx)  ← cluster-scoped, one per LB type
  Gateway (prod-gateway) in gateway-ns  ← provisions the LB, sets allowedRoutes

Team A owns:
  HTTPRoute in team-a-ns → attaches to prod-gateway → routes /app-a/* to their service

Team B owns:
  HTTPRoute in team-b-ns → attaches to prod-gateway → routes /app-b/* to their service

ReferenceGrant in team-b-ns (if Team A's route needs to reach Team B's service):
  Grants permission for HTTPRoutes from team-a-ns to reference Services in team-b-ns
```

**Why this is powerful:** Team A cannot see or modify Team B's HTTPRoute. Kubernetes RBAC at the resource level. No shared Ingress resource = no blast radius.

### Pattern 2: Cloud-Managed on AWS (Production EKS)

```
AWS Gateway API Controller (in-cluster) watches Gateway objects
  │
  └──► Calls AWS APIs to create VPC Lattice Service Networks / Services
         (Gateway spec → VPC Lattice config)

HTTPRoute → Lattice routing rules (host + path)

Traffic path:
  Client → Route 53 → VPC Lattice (AWS-managed) → Target Group → Pod

Controller never handles traffic. Fully managed data plane.
```

### Pattern 3: In-Cluster on KIND/On-Prem

```
NGINX Gateway Fabric controller pod (control plane):
  Watches Gateway + HTTPRoute
  Writes nginx.conf to proxy pod via shared memory / sidecar pattern

NGINX proxy pod (data plane):
  Exposed via NodePort (dev) or LoadBalancer (prod)
  Handles all L7 traffic

Traffic path:
  Client → NodePort:31000 → NGINX proxy pod → ClusterIP Service → Pod
```

### Traffic Flow Quick Reference

```
Cloud:    Client → Cloud LB → Kubernetes Service → Backend Pod
In-Cluster: Client → LoadBalancer/NodePort Svc → Proxy Pod → Backend Svc → Backend Pod
```

### TLS Termination at the Gateway

```yaml
# Gateway with HTTPS listener
spec:
  listeners:
  - name: https
    port: 443
    protocol: HTTPS
    tls:
      mode: Terminate         # Gateway decrypts TLS
      certificateRefs:
      - kind: Secret
        name: my-tls-secret   # TLS cert stored as k8s Secret
    allowedRoutes:
      namespaces:
        from: All
```

---

## 6. Mistakes — What Actually Breaks in Real Systems

### 💥 Mistake 1: CRD/Controller Version Mismatch

**Symptom:** Controller crashes on start, or resources like `BackendTLSPolicy` show "resource not found."

**Root Cause:** Helm installs the latest controller, but you installed older CRDs (or vice versa). They don't share a schema.

**Fix:**
```bash
# Check what CRD version is installed
kubectl get crd gateways.gateway.networking.k8s.io -o jsonpath='{.metadata.annotations}'
# Compare with controller version
kubectl get deploy -n ngf-gatewayapi-ns -o jsonpath='{.items[0].spec.template.spec.containers[0].image}'
# Reinstall CRDs matching the controller version
```

---

### 💥 Mistake 2: HTTPRoute Not Attaching (parentRefs Wrong)

**Symptom:** Routes exist but traffic returns 404 or default backend.

**Root Cause:** `sectionName` in `parentRefs` doesn't match the `name` field of the listener in the Gateway spec. These are case-sensitive exact matches.

**Fix:**
```bash
kubectl describe httproutes.gateway.networking.k8s.io <route-name> -n <ns>
# Look for: "No parent found" or "Listener not found"

# Cross-check:
kubectl get gateway <gw-name> -n <gw-ns> -o jsonpath='{.spec.listeners[*].name}'
# Must match the sectionName in your HTTPRoute parentRefs exactly
```

---

### 💥 Mistake 3: allowedRoutes Too Restrictive

**Symptom:** HTTPRoute in `app-ns` can't attach to Gateway in `gateway-ns`.

**Root Cause:** Gateway's `allowedRoutes.namespaces.from` is set to `Same` instead of `All` or `Selector`.

**Fix:**
```yaml
allowedRoutes:
  namespaces:
    from: Selector           # Or "All" for dev environments
    selector:
      matchLabels:
        shared-gateway-access: "true"
# Then label the namespace:
kubectl label namespace app-ns shared-gateway-access=true
```

---

### 💥 Mistake 4: Missing ReferenceGrant for Cross-Namespace Backend

**Symptom:** `ResolvedRefs=False` on HTTPRoute, error says "backend not permitted."

**Root Cause:** An HTTPRoute in `namespace-a` references a Service in `namespace-b` without a `ReferenceGrant` in `namespace-b`.

**Fix:** Create the `ReferenceGrant` in the target namespace (see Section 3).

---

### 💥 Mistake 5: Route Ordering / Priority Confusion

**Symptom:** `/` catch-all route intercepts traffic meant for `/iphone`.

**Root Cause:** In Gateway API, **more specific matches win**. `PathPrefix: /iphone` is more specific than `PathPrefix: /`. But if routes have overlapping specificity, behavior may be controller-dependent.

**Fix:** Order your routes from most specific to least specific. Use `Exact` match type for exact paths when needed.

```yaml
# More specific wins:
- path:
    type: Exact
    value: /iphone/special    # Matched first
- path:
    type: PathPrefix
    value: /iphone            # Matched second
- path:
    type: PathPrefix
    value: /                  # Catch-all, matched last
```

---

### 💥 Mistake 6: NodePort Mapping Missing in KIND

**Symptom:** `curl localhost:31000` connection refused even though Gateway is Ready.

**Root Cause:** The KIND cluster config didn't have `extraPortMappings`, or the NodePort number in Helm flags doesn't match the port mapping.

**Fix:**
```bash
# Check KIND port mappings
docker ps  # look at the PORTS column for the control-plane container
# Must show: 0.0.0.0:31000->31000/tcp

# Check NodePort value
kubectl get svc -n ngf-gatewayapi-ns -o jsonpath='{.items[0].spec.ports[0].nodePort}'
# Must equal 31000
```

---

### 💥 Mistake 7: Annotating Ingress-Style Thinking Into HTTPRoute

**Symptom:** Trying to add `nginx.ingress.kubernetes.io/rewrite-target` to an HTTPRoute.

**Root Cause:** Mental model still in Ingress-land.

**Fix:** Use `HTTPRoute` filters for rewrites:
```yaml
rules:
- filters:
  - type: URLRewrite
    urlRewrite:
      path:
        type: ReplacePrefixMatch
        replacePrefixMatch: /
  backendRefs:
  - name: my-svc
    port: 80
```

---

## 7. Interview Answers — Verbatim-Ready

### Q: What problem does Gateway API solve that Ingress couldn't?

*"Ingress had three core problems. First, it was namespace-scoped and couldn't natively support multi-tenant routing across namespaces without vendor-specific hacks like IngressGroups. Second, almost every advanced feature — SSL redirects, weighted routing, header matching — required controller-specific annotations that Kubernetes never validated, which led to silent misconfigurations and vendor lock-in. Third, there was no separation of concerns — developers and platform teams all had to share and mutate the same Ingress resource. Gateway API fixes all three: routing rules live in separate HTTPRoute objects with proper RBAC, advanced features are part of the validated API spec, and the resource model explicitly maps to organizational roles — GatewayClass for platform teams, Gateway for operators, and Routes for developers."*

---

### Q: Explain the three core resources of Gateway API and who owns them.

*"Gateway API has three core resources, each mapping to a different organizational role. GatewayClass is a cluster-scoped resource that defines a type of load balancer and points to the controller responsible for managing it — similar to a StorageClass for volumes. This is owned by the Infrastructure Provider or platform engineering team. Gateway is an instance of a GatewayClass — it provisions actual traffic-handling infrastructure like a cloud load balancer or in-cluster proxy, and defines listeners with ports and protocols. This is owned by the Cluster Operator. Finally, HTTPRoutes or other route types define the actual routing rules — which paths or headers go to which backend services. These are owned by application developers. The beauty of this model is that each layer has a distinct API boundary with its own RBAC, so teams can operate independently without stepping on each other."*

---

### Q: What is the difference between a cloud-managed and in-cluster data plane in Gateway API?

*"In both cases, the controller runs as a pod inside the cluster and watches Gateway API resources — so the control plane is always in-cluster. The difference is where packets actually flow. In a cloud-managed setup, like AWS VPC Lattice or GCP's Cloud Load Balancer with GKE Gateway, the controller programs an external, fully managed load balancer — it never sits in the traffic path itself. In an in-cluster setup, like NGINX Gateway Fabric or Envoy Gateway, the controller configures proxy pods running inside the cluster, and those proxy pods are the data plane that handles actual traffic. The controller is never in the hot path in either case — that's a critical design constraint. In-cluster is great for on-prem or environments needing fine-grained L7 control; cloud-managed is better for scalability and reduced operational overhead."*

---

### Q: How does an HTTPRoute attach to a Gateway?

*"An HTTPRoute uses a `parentRefs` field to declare which Gateway it wants to attach to. This includes the Gateway's name, namespace, and optionally a `sectionName` which must exactly match one of the listener names defined in the Gateway spec. The Gateway also has an `allowedRoutes` configuration that controls which namespaces can attach routes — it can be set to `Same`, `All`, or a label selector. The controller reconciles both sides: it reads the HTTPRoute, validates the parentRefs, checks that the Gateway allows the Route's namespace, and only then programs the routing rules into the data plane. The Route's status will reflect `Accepted=True` and `ResolvedRefs=True` when everything is wired up correctly."*

---

### Q: What is a ReferenceGrant and when do you need it?

*"A ReferenceGrant is a resource you create in the target namespace to explicitly permit another namespace to reference resources within it. You need it whenever an HTTPRoute in one namespace wants to forward traffic to a Service in a different namespace. Without it, the controller will reject the cross-namespace reference with a `ResolvedRefs=False` status. The ReferenceGrant lives in the namespace of the Service being referenced — this is important — and it specifies which source namespace and kind is permitted. This design prevents a rogue team from routing traffic to backends they don't own just by naming them in a Route."*

---

### Q: Why are Gateway API CRDs not installed by default?

*"Gateway API is intentionally separate from the core Kubernetes distribution because it's an evolving API maintained by SIG-Network that requires a compatible controller to do anything useful. Shipping it by default without a controller would just add confused users trying to apply HTTPRoute resources that nothing processes. The CRD separation also allows Gateway API to iterate faster than the Kubernetes release cycle — new features can ship in CRD updates without waiting for a Kubernetes minor version. In practice, you install the CRDs via kubectl kustomize or Helm, and install a compatible controller separately — NGINX Gateway Fabric, Envoy Gateway, or a cloud-provider controller."*

---

## 8. Debugging — Fast Diagnosis Paths

### The Gateway API Debugging Decision Tree

```
Problem: Traffic not routing correctly
│
├─ Is the GatewayClass Accepted?
│  └─ kubectl get gatewayclasses -o wide
│     ├─ ACCEPTED=False → Controller not running or not watching this class
│     │  → kubectl get pods -n <controller-ns>
│     │  → kubectl logs <controller-pod>
│     └─ ACCEPTED=True → continue
│
├─ Is the Gateway Programmed/Ready?
│  └─ kubectl get gateway -n <gw-ns>
│     kubectl describe gateway <name> -n <gw-ns>
│     ├─ Programmed=False → Controller error programming data plane
│     │  → kubectl logs <controller-pod>
│     │  → kubectl get events -n <gw-ns> --sort-by=.lastTimestamp
│     └─ Programmed=True → continue
│
├─ Is the HTTPRoute Accepted?
│  └─ kubectl describe httproutes.gateway.networking.k8s.io <name> -n <ns>
│     ├─ Accepted=False → parentRefs wrong
│     │  → Check sectionName matches listener name EXACTLY
│     │  → Check namespace is allowed by Gateway's allowedRoutes
│     ├─ ResolvedRefs=False → Backend not found or cross-namespace without ReferenceGrant
│     │  → Check Service exists in referenced namespace
│     │  → Check ReferenceGrant exists in target namespace
│     └─ Both True → continue
│
├─ Is the Service/Endpoints healthy?
│  └─ kubectl get endpoints <svc-name> -n <ns>
│     ├─ No endpoints → Pods not Ready, label selector mismatch
│     └─ Endpoints present → continue
│
└─ Is the data plane actually receiving traffic?
   └─ kubectl logs -f <proxy-pod> -n <controller-ns>
      ├─ No log entries → Traffic not reaching proxy (check NodePort, LB, DNS)
      └─ 502/503 errors → Backend unhealthy; check pod logs
```

### Quick Reference Commands

```bash
# --- Resource Health ---
kubectl get gatewayclasses -o wide
kubectl get gateway -A
kubectl get httproutes -A
kubectl get referencegrants -A

# --- Status Deep Dives ---
kubectl describe gateway <name> -n <ns>
kubectl describe httproutes.gateway.networking.k8s.io <name> -n <ns>

# --- Controller Logs ---
kubectl logs -n ngf-gatewayapi-ns \
  $(kubectl get pods -n ngf-gatewayapi-ns -o name | grep controller | head -1)

# --- Proxy/Data Plane Logs ---
kubectl logs -n ngf-gatewayapi-ns \
  $(kubectl get pods -n ngf-gatewayapi-ns -o name | grep nginx | head -1)

# --- Events (sorted, recent first) ---
kubectl get events -n <ns> --sort-by=.lastTimestamp | tail -20

# --- Test routing directly ---
curl -v http://localhost:31000/iphone
curl -v http://localhost:31000/android
curl -v http://localhost:31000/

# --- Check if NodePort is mapped correctly ---
kubectl get svc -n ngf-gatewayapi-ns -o wide
docker ps  # check KIND port bindings

# --- Verify CRDs are installed ---
kubectl api-resources | grep gateway.networking.k8s.io
```

### Status Condition Reference

| Resource | Condition | True Means | False Means |
|---|---|---|---|
| GatewayClass | `Accepted` | Controller is running and owns this class | Controller not found or not watching |
| Gateway | `Programmed` | Data plane configured and ready | Controller error or missing GatewayClass |
| HTTPRoute | `Accepted` | Route attached to Gateway listener | parentRefs mismatch or namespace not allowed |
| HTTPRoute | `ResolvedRefs` | All backend Services found | Service missing or cross-ns without ReferenceGrant |

---

## 9. Kill Switch — 10-Second Recall

> The absolute minimum. If you can hold this, you can reconstruct everything else.

```
GATEWAY API = Ingress++ with RBAC, multi-protocol, spec-defined features

3 RESOURCES:
  GatewayClass  → Infrastructure Provider (WHAT kind of LB)
  Gateway       → Cluster Operator (provision the LB + listeners)
  HTTPRoute     → App Developer (routing rules to Services)

FLOW:
  GatewayClass ← Controller watches it
  Gateway      → Controller programs data plane
  HTTPRoute    → attach via parentRefs.sectionName = listener.name
  Cross-ns backend? → Need ReferenceGrant in TARGET namespace

KEY FACTS:
  ✓ Not installed by default — install CRDs + controller
  ✓ Controller = control plane (never handles traffic)
  ✓ Cloud LB or proxy pods = data plane (ALWAYS handles traffic)
  ✓ Features are in spec (not annotations) → validated + portable
  ✓ allowedRoutes controls which namespaces can attach Routes

DEBUG ORDER:
  GatewayClass Accepted? → Gateway Programmed? → HTTPRoute Accepted+ResolvedRefs? → Endpoints? → Proxy logs
```

---

## 10. Appendix — Quick Reference Card

### Resource Cheatsheet

| Resource | Scope | API Version | Owned By |
|---|---|---|---|
| `GatewayClass` | Cluster | `gateway.networking.k8s.io/v1` | Infra Provider |
| `Gateway` | Namespace | `gateway.networking.k8s.io/v1` | Cluster Operator |
| `HTTPRoute` | Namespace | `gateway.networking.k8s.io/v1` | App Developer |
| `GRPCRoute` | Namespace | `gateway.networking.k8s.io/v1` | App Developer |
| `ReferenceGrant` | Namespace | `gateway.networking.k8s.io/v1beta1` | Owner of target namespace |
| `TLSRoute` | Namespace | `gateway.networking.k8s.io/v1alpha2` | App Developer |
| `TCPRoute` | Namespace | `gateway.networking.k8s.io/v1alpha2` | App Developer |

### HTTPRoute Match Types

| Field | Type Options | Example |
|---|---|---|
| `path.type` | `Exact`, `PathPrefix`, `RegularExpression` | `PathPrefix: /api` |
| `headers` | `Exact`, `RegularExpression` | `name: X-Version, value: v2` |
| `queryParams` | `Exact`, `RegularExpression` | `name: env, value: prod` |
| `method` | `GET`, `POST`, `PUT`, `DELETE`, etc. | `method: GET` |

### Gateway allowedRoutes Options

```yaml
allowedRoutes:
  namespaces:
    from: All        # Any namespace
    from: Same       # Same namespace as Gateway only
    from: Selector   # Namespaces matching a label selector
      selector:
        matchLabels:
          key: value
```

### Listener Protocol Options

| Protocol | TLS Required | Route Kind |
|---|---|---|
| `HTTP` | No | `HTTPRoute` |
| `HTTPS` | Yes (Terminate mode) | `HTTPRoute` |
| `TLS` | Yes (Passthrough mode) | `TLSRoute` |
| `TCP` | Optional | `TCPRoute` |
| `UDP` | No | `UDPRoute` |

### Common Commands Reference

```bash
# Install CRDs (check version at nginx gateway fabric docs first)
kubectl kustomize "https://github.com/nginx/nginx-gateway-fabric/config/crd/gateway-api/standard?ref=<VERSION>" | kubectl apply -f -

# Install NGF via Helm (NodePort mode for KIND)
helm install ngf oci://ghcr.io/nginx/charts/nginx-gateway-fabric \
  --create-namespace -n ngf-gatewayapi-ns \
  --set nginx.service.type=NodePort \
  --set-json 'nginx.service.nodePorts=[{"port":31000,"listenerPort":80}]'

# Get all Gateway API resources
kubectl api-resources | grep gateway.networking.k8s.io

# Watch resource status
kubectl get gateway,httproutes -A

# Detailed status
kubectl describe gateway <name> -n <ns>
kubectl describe httproutes.gateway.networking.k8s.io <name> -n <ns>

# Check controller + data plane pods
kubectl get pods -n ngf-gatewayapi-ns

# Tail proxy logs
kubectl logs -f -n ngf-gatewayapi-ns <proxy-pod-name>

# Test routing
curl -v http://localhost:31000/iphone
curl -v http://localhost:31000/android
curl -v http://localhost:31000/
```

### Ingress vs Gateway API — Side-by-Side

| Feature | Ingress | Gateway API |
|---|---|---|
| Multi-namespace routing | ❌ (or controller-specific) | ✅ native |
| RBAC per routing rule | ❌ | ✅ (separate Route objects) |
| TCP/UDP support | ❌ | ✅ |
| gRPC support | ❌ native | ✅ GRPCRoute |
| Weighted traffic splitting | ❌ native (annotations) | ✅ in spec |
| Header/query routing | ❌ native (annotations) | ✅ in spec |
| Timeouts & retries | ❌ native (annotations) | ✅ in spec |
| API validation | Annotations unvalidated | ✅ full schema validation |
| Portability | Low (annotation lock-in) | High (conformant spec) |
| Built into Kubernetes | ✅ | ❌ (CRDs + controller) |

---

