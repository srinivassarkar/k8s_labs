# 🚦 Kubernetes Ingress 
> *"An Ingress without a controller is like a map without a navigator — declarative intent, zero action."*

---

## 0. First Principles — The Mental Model That Never Changes

> These truths hold regardless of cloud provider, controller version, or cluster size.

| Principle | What It Means |
|---|---|
| **Ingress is declarative, not active** | The `Ingress` resource is a YAML spec. It does nothing by itself. Traffic never passes *through* it. |
| **The controller is the engine** | The Ingress Controller watches Ingress objects and translates rules into real load balancer config. |
| **Layer 7 is the unlock** | Ingress operates at HTTP/HTTPS (L7). This enables host/path routing, TLS termination — things L4 (NodePort, LoadBalancer Service) simply cannot do. |
| **One LB, many services** | The whole point: one external load balancer fronts *n* services. Cost collapses, operations simplify. |
| **Annotations drive controller behavior** | Controller-specific behavior (TLS, health checks, target types) is expressed via `metadata.annotations`, not `spec`. |
| **IngressClass = controller selector** | When multiple controllers coexist, `ingressClassName` tells Kubernetes which one owns this Ingress. |

---

## 1. Reality Constraints — What Kubernetes Actually Does and Doesn't Do

### What Kubernetes Gives You (Core API)
- An `Ingress` API object at `networking.k8s.io/v1`
- Path-type semantics: `Exact`, `Prefix`, `ImplementationSpecific`
- A default backend fallback
- TLS secret reference support
- `IngressClass` resource for controller binding

### What Kubernetes Does NOT Give You
- ❌ Any actual routing logic — that's the controller's job
- ❌ A built-in Ingress Controller — you must install one
- ❌ L4 (TCP/UDP) routing — use `Service: LoadBalancer` for that
- ❌ Automatic TLS certificate provisioning — use cert-manager or ACM
- ❌ Rate limiting, auth, canary — all controller-specific, via annotations

### Service vs. Ingress: The Honest Comparison

| Feature | NodePort | LoadBalancer Service | Ingress |
|---|---|---|---|
| OSI Layer | L4 | L4 | L7 |
| Path-based routing | ❌ | ❌ | ✅ |
| Host-based routing | ❌ | ❌ | ✅ |
| TLS termination | ❌ | ❌ (app handles it) | ✅ |
| Cloud LB per service | N/A | ✅ (one per svc) | ✅ (one for all) |
| Cost at scale | Medium | High ($$$ per svc) | Low |
| Production-ready | Dev/test only | Sometimes | ✅ |

### Path Type Semantics (Production-Critical)

| pathType | Rule | Example: `/foo` matches |
|---|---|---|
| `Exact` | Must match exactly, no trailing slash | `/foo` only |
| `Prefix` | Matches path prefix, segment-aware | `/foo`, `/foo/`, `/foo/bar` |
| `ImplementationSpecific` | Controller decides — avoid in CKA | Controller-dependent |

> ⚠️ `Prefix` matching is segment-aware. `/fo` does NOT match `/foo`. `/foo` DOES match `/foo/bar`.

---

## 2. Decision Logic — When to Use What

### Should I Use Ingress?

```
Is this HTTP or HTTPS traffic?
    ├── NO  → Use Service type LoadBalancer (L4) or bare NodePort
    └── YES → Do you need path or host-based routing?
                  ├── NO  → Single service? Use LoadBalancer Service
                  └── YES → Use Ingress ✅
                                 ├── On AWS EKS? → AWS Load Balancer Controller (ALB)
                                 ├── On-prem / self-managed? → NGINX or HAProxy
                                 ├── Need service mesh + mTLS? → Istio Ingress Gateway
                                 └── Need auto-TLS (Let's Encrypt)? → Traefik
```

### Cloud-Native vs. 3rd-Party Controller

| Dimension | Cloud-Native (e.g., AWS LBC) | 3rd-Party (e.g., NGINX) |
|---|---|---|
| **LB provisioned** | External L7 ALB (outside cluster) | Internal controller pod (inside cluster) |
| **Exposed via** | Direct ALB provisioned from Ingress annotations | `Service: LoadBalancer` → L4 LB → Ingress Controller pod |
| **TLS termination** | At the ALB (e.g., ACM cert) | At the controller pod |
| **Routing logic runs** | In the cloud load balancer | Inside the cluster |
| **IAM/cloud permissions needed** | ✅ Yes (IRSA on EKS) | ❌ No |
| **Best for** | Managed cloud (EKS, GKE) | On-prem, hybrid, multi-cloud |

### ALB Target Type Decision

```
Are your Pods getting IPs directly from the VPC subnet (VPC CNI)?
    ├── YES → Use target-type: ip
    │              └── ALB → Pod IP directly (fewer hops, pod-level health checks)
    └── NO  → Use target-type: instance
                   └── ALB → EC2 NodePort → kube-proxy → ClusterIP → Pod
                              (works with any CNI: Calico, Weave, Flannel)
```

### Popular Ingress Controllers at a Glance

| Controller | Best For | Key Differentiator |
|---|---|---|
| **AWS Load Balancer Controller** | EKS production | Provisions real AWS ALB/NLB; deep AWS integration |
| **NGINX Ingress Controller** | Universal | Most widely used; highly customizable; strong community |
| **HAProxy Ingress** | High perf / low latency | Optimized HAProxy config generation |
| **Traefik** | Cloud-native / dynamic | Auto TLS via Let's Encrypt; real-time config reload |
| **Contour** | Multi-team / advanced traffic | Built on Envoy; fine-grained traffic policies |
| **Istio Ingress Gateway** | Service mesh environments | mTLS, observability, circuit breaking |

---

## 3. Internal Working — What Actually Happens Under the Hood

### Request Flow: Cloud-Native (AWS ALB) 🔵

```
[User Browser]
      │  HTTP GET myapp.com/iphone
      ▼
[DNS / Route 53]
      │  Resolves to ALB DNS name
      ▼
[AWS ALB] ← provisioned by AWS LBC watching the Ingress object
      │  Evaluates listener rules:
      │    /iphone → iphone-svc target group
      │    /android → android-svc target group
      │    / → desktop-svc target group
      ▼
[ALB Target Group (type: ip)]
      │  Direct to Pod IP via VPC ENI
      ▼
[Pod: iphone-deploy, port 5678]
      │  Serves /iphone/index.html
      ▼
[Response back to user]
```

**Controller reconciliation loop (step by step):**

1. Engineer applies `Ingress` object → stored in etcd
2. AWS LBC controller (Deployment in `kube-system`) has a **watch** on `Ingress` resources
3. LBC detects new/changed Ingress → reads annotations + spec
4. LBC calls AWS APIs to:
   - Create/update ALB (`CreateLoadBalancer`)
   - Create Listeners on port 80/443
   - Create Target Groups per backend Service
   - Register Pod IPs in Target Groups (if `target-type: ip`)
   - Configure routing rules matching paths → target groups
5. ALB DNS name written back to `Ingress.status.loadBalancer.ingress[].hostname`

### Request Flow: 3rd-Party (NGINX) 🟠

```
[User Browser]
      │  HTTP GET myapp.com/iphone
      ▼
[Cloud L4 Load Balancer] ← created by Service: LoadBalancer for NGINX controller
      │  Forwards TCP stream (no L7 inspection)
      ▼
[NGINX Ingress Controller Pod]
      │  Reads Ingress rules from Kubernetes API
      │  Performs L7 routing internally (path match)
      │  TLS termination happens HERE
      ▼
[ClusterIP Service: iphone-svc]
      ▼
[Pod: iphone-deploy]
```

---

## 4. Hands-On — Production-Quality YAML + Commands

### 4.1 Namespace

```yaml
# 01-ns.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: app1-ns
```

### 4.2 iPhone Deployment + Service

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
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: iphone-page
      containers:
      - name: python-http
        image: python:alpine
        command: ["/bin/sh", "-c"]
        args:
        - |
          mkdir -p /iphone && echo '<html>
          <head><title>iPhone Users</title></head>
          <body><h1>iPhone Users</h1></body>
          </html>' > /iphone/index.html && cd / && python3 -m http.server 5678
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: iphone-svc
  namespace: app1-ns
  annotations:
    # Per-service health check path — ALB reads this annotation
    alb.ingress.kubernetes.io/healthcheck-path: /iphone/index.html
spec:
  selector:
    app: iphone-page
  ports:
  - protocol: TCP
    port: 80
    targetPort: 5678
```

### 4.3 Android Deployment + Service

```yaml
# 03-android.yaml — mirrors iphone, path: /android
apiVersion: apps/v1
kind: Deployment
metadata:
  name: android-deploy
  namespace: app1-ns
spec:
  replicas: 2
  selector:
    matchLabels:
      app: android-page
  template:
    metadata:
      labels:
        app: android-page
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: android-page
      containers:
      - name: python-http
        image: python:alpine
        command: ["/bin/sh", "-c"]
        args:
        - |
          mkdir -p /android && echo '<html>
          <head><title>Android Users</title></head>
          <body><h1>Android Users</h1></body>
          </html>' > /android/index.html && cd / && python3 -m http.server 5678
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: android-svc
  namespace: app1-ns
  annotations:
    alb.ingress.kubernetes.io/healthcheck-path: /android/index.html
spec:
  selector:
    app: android-page
  ports:
  - protocol: TCP
    port: 80
    targetPort: 5678
```

### 4.4 Desktop Deployment + Service (catch-all)

```yaml
# 04-desktop.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: desktop-deploy
  namespace: app1-ns
spec:
  replicas: 2
  selector:
    matchLabels:
      app: desktop-page
  template:
    metadata:
      labels:
        app: desktop-page
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: desktop-page
      containers:
      - name: python-http
        image: python:alpine
        command: ["/bin/sh", "-c"]
        args:
        - |
          echo '<html>
          <head><title>Desktop Users</title></head>
          <body><h1>Desktop Users</h1></body>
          </html>' > /index.html && python3 -m http.server 5678
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: desktop-svc
  namespace: app1-ns
  annotations:
    alb.ingress.kubernetes.io/healthcheck-path: /index.html
spec:
  selector:
    app: desktop-page
  ports:
  - protocol: TCP
    port: 80
    targetPort: 5678
```

### 4.5 Ingress Resource (AWS ALB — Path-Based Routing)

```yaml
# 05-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-demo1
  namespace: app1-ns
  annotations:
    # --- ALB Scheme ---
    alb.ingress.kubernetes.io/scheme: internet-facing        # internet-facing | internal
    alb.ingress.kubernetes.io/load-balancer-name: cwvj-ingress-demo1

    # --- Target type: ip = direct to Pod IP; instance = via NodePort ---
    alb.ingress.kubernetes.io/target-type: ip

    # --- Listener config ---
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]'  # JSON array

    # --- Health check config ---
    alb.ingress.kubernetes.io/healthcheck-protocol: HTTP
    alb.ingress.kubernetes.io/healthcheck-port: traffic-port  # "traffic-port" = same as listener
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: '15'
    alb.ingress.kubernetes.io/healthcheck-timeout-seconds: '5'
    alb.ingress.kubernetes.io/healthy-threshold-count: '2'
    alb.ingress.kubernetes.io/unhealthy-threshold-count: '2'
    alb.ingress.kubernetes.io/success-codes: '200'
spec:
  ingressClassName: alb   # Binds this Ingress to the AWS ALB controller
  rules:
  - http:
      paths:
      - path: /iphone
        pathType: Prefix
        backend:
          service:
            name: iphone-svc
            port:
              number: 80
      - path: /android
        pathType: Prefix
        backend:
          service:
            name: android-svc
            port:
              number: 80
      - path: /           # MUST be last — catch-all default
        pathType: Prefix
        backend:
          service:
            name: desktop-svc
            port:
              number: 80
```

> ⚠️ **Rule ordering matters.** More specific paths must come before the catch-all `/`. The AWS LBC respects declaration order when creating ALB listener rules.

### 4.6 EKS Cluster Config (eksctl)

```yaml
# eks-config.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: cwvj-ingress-demo
  region: us-east-2
  tags:
    owner: varun-joshi
    project: ingress-demo

availabilityZones:
- us-east-2a
- us-east-2b

iam:
  withOIDC: true   # Required for IRSA (IAM Roles for Service Accounts)

managedNodeGroups:
- name: cwvj-eks-priv-ng
  instanceType: t3.small
  minSize: 4
  maxSize: 4
  privateNetworking: true     # Nodes in private subnets only
  volumeSize: 20
  iam:
    withAddonPolicies:
      autoScaler: true
      certManager: true
      albIngress: true        # Grants LBC the permissions it needs
  labels:
    lifecycle: ec2-autoscaler
```

### 4.7 Complete Setup Commands

```bash
# --- STEP 1: Provision EKS Cluster ---
eksctl create cluster -f eks-config.yaml

# Verify nodes are distributed across AZs
kubectl get nodes --show-labels | grep topology.kubernetes.io/zone

# --- STEP 2: Install AWS Load Balancer Controller ---

# 2a. Download and create IAM policy
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.13.3/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json

# 2b. Create IAM-backed service account (IRSA)
eksctl create iamserviceaccount \
  --cluster=cwvj-ingress-demo \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --region us-east-2 \
  --approve

# Verify the annotation exists (eks.amazonaws.com/role-arn)
kubectl get sa -n kube-system aws-load-balancer-controller -o yaml

# 2c. Install via Helm
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=cwvj-ingress-demo \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --version 1.13.0

# Verify controller is running (2/2)
kubectl get deployment -n kube-system aws-load-balancer-controller

# --- STEP 3: Deploy applications ---
kubectl apply -f 01-ns.yaml
kubectl config set-context --current --namespace=app1-ns
kubectl apply -f 02-iphone.yaml
kubectl apply -f 03-android.yaml
kubectl apply -f 04-desktop.yaml

# Verify pods distributed across AZs
kubectl get pods -o wide --sort-by='.spec.nodeName'

# --- STEP 4: Apply Ingress ---
kubectl apply -f 05-ingress.yaml

# Watch ALB provisioning (takes ~3 min)
kubectl get ingress -w

# --- STEP 5: Verify ---
kubectl describe ingress ingress-demo1
kubectl get ingressclass

# Test routes (replace with your ALB DNS)
curl http://<ALB_DNS>/iphone
curl http://<ALB_DNS>/android
curl http://<ALB_DNS>/

# --- STEP 6: Cleanup ---
kubectl delete -f .
# OR delete cluster entirely:
eksctl delete cluster --name cwvj-ingress-demo
```

---

## 5. Production Flow — Architecture and Design Patterns

### Pattern 1: Single-Domain Path-Based Routing (This Demo)

```
Internet
   │
   ▼
[AWS ALB: cwvj-ingress-demo1]
   │  /iphone ──────────────────▶ [iphone-svc] ──▶ [iphone-pod-az-a] [iphone-pod-az-b]
   │  /android ─────────────────▶ [android-svc] ──▶ [android-pod-az-a] [android-pod-az-b]
   │  / (catch-all) ────────────▶ [desktop-svc] ──▶ [desktop-pod-az-a] [desktop-pod-az-b]
   │
   └── Health checks per service via ALB target group
```

**Why it works in prod:** Every service gets its own ALB target group with independent health checks. A failing `/android` backend doesn't affect `/iphone`. 

### Pattern 2: Host-Based Routing (Coming in Part 3)

```
iphone.myapp.com  ──▶ [ALB] ──▶ iphone-svc
android.myapp.com ──▶ [ALB] ──▶ android-svc
```

```yaml
# Host-based rule in Ingress spec
rules:
- host: iphone.myapp.com
  http:
    paths:
    - path: /
      pathType: Prefix
      backend:
        service:
          name: iphone-svc
          port:
            number: 80
- host: android.myapp.com
  http:
    paths:
    - path: /
      pathType: Prefix
      backend:
        service:
          name: android-svc
          port:
            number: 80
```

### Pattern 3: HTTPS with TLS Termination (ACM)

```yaml
metadata:
  annotations:
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-2:ACCOUNT:certificate/CERT_ID
    alb.ingress.kubernetes.io/ssl-redirect: '443'
spec:
  tls:
  - hosts:
    - myapp.com
    secretName: myapp-tls   # For NGINX; ACM handles this differently
```

### Pattern 4: Multi-Tenant with Multiple IngressClasses

```
Cluster
├── Namespace: team-a
│   └── Ingress (ingressClassName: nginx)  ──▶ NGINX Controller Pod
└── Namespace: team-b
    └── Ingress (ingressClassName: alb)    ──▶ AWS LBC ──▶ AWS ALB
```

### TopologySpreadConstraints Pattern (Used in This Demo)

```yaml
topologySpreadConstraints:
- maxSkew: 1                            # Max difference between AZ pod counts
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: ScheduleAnyway    # Soft constraint (DoNotSchedule = hard)
  labelSelector:
    matchLabels:
      app: iphone-page
```

> `maxSkew: 1` means if AZ-a has 2 pods, AZ-b must have at least 1. This protects against all pods landing in one AZ.

---

## 6. Mistakes — What Actually Breaks in Real Systems

### 💥 Mistake 1: Catch-All `/` Path Not Last in Rules

**Symptom:** All requests go to `desktop-svc` regardless of path.

**Root Cause:** Kubernetes Ingress spec is evaluated top-down. If `/` (Prefix) appears first, it matches *everything* before `/iphone` is evaluated.

**Fix:**
```yaml
# WRONG: / first means it swallows everything
paths:
- path: /
  pathType: Prefix
  ...
- path: /iphone   # Never reached!
  pathType: Prefix
  ...

# CORRECT: specific paths first, catch-all last
paths:
- path: /iphone
- path: /android
- path: /    # last!
```

---

### 💥 Mistake 2: Missing `ingressClassName` in Multi-Controller Clusters

**Symptom:** Ingress created but no ALB provisioned. No events. `kubectl get ingress` shows empty ADDRESS.

**Root Cause:** Without `ingressClassName: alb`, no controller claims ownership of the Ingress. In older clusters, a `kubernetes.io/ingress.class` annotation was used. In Kubernetes ≥ 1.18, use `.spec.ingressClassName`.

**Fix:**
```yaml
spec:
  ingressClassName: alb   # must match an IngressClass resource name
```

Verify available classes:
```bash
kubectl get ingressclass
```

---

### 💥 Mistake 3: Wrong Health Check Path per Service

**Symptom:** Target group shows unhealthy targets. Pods are running fine.

**Root Cause:** ALB health check uses a default path (often `/`) but `iphone-svc` pods only serve `/iphone/index.html`. A `GET /` returns 404, failing the health check.

**Fix:** Set per-service health check annotations:
```yaml
# On the Service, NOT the Ingress
metadata:
  annotations:
    alb.ingress.kubernetes.io/healthcheck-path: /iphone/index.html
```

---

### 💥 Mistake 4: `target-type: ip` Without VPC CNI

**Symptom:** ALB shows targets as unhealthy or unreachable. Works with `target-type: instance` but not `ip`.

**Root Cause:** `target-type: ip` requires pods to have routable VPC IPs. This requires AWS VPC CNI. If using Calico or Flannel, pod IPs are not routable from the VPC, so the ALB can't reach them directly.

**Fix:** Use `target-type: instance` for non-VPC-CNI CNIs:
```yaml
alb.ingress.kubernetes.io/target-type: instance
```

---

### 💥 Mistake 5: IRSA Misconfiguration — LBC Has No AWS Permissions

**Symptom:** LBC pods are running but no ALB is created. Controller logs show `AccessDenied` or `UnauthorizedOperation`.

**Root Cause:** The ServiceAccount is not annotated with the correct IAM Role ARN, or the OIDC provider wasn't set up (`withOIDC: true` in eksctl).

**Fix:**
```bash
# Verify annotation exists on ServiceAccount
kubectl get sa -n kube-system aws-load-balancer-controller -o jsonpath='{.metadata.annotations}'

# Check controller logs for permission errors
kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
```

---

### 💥 Mistake 6: `helm list` Shows No LBC Installation

**Symptom:** After installing, `helm list` returns nothing.

**Root Cause:** The LBC is installed in `kube-system`, not `default`. `helm list` defaults to current namespace.

**Fix:**
```bash
helm list -n kube-system
# or
helm list -A   # all namespaces
```

---

### 💥 Mistake 7: `pathType: ImplementationSpecific` in CKA

**Symptom:** Behavior differs across environments; test clusters may not behave as expected.

**Root Cause:** `ImplementationSpecific` delegates matching semantics entirely to the controller. Behavior is undefined at the Kubernetes spec level.

**Fix:** Always use `Exact` or `Prefix` in CKA and production. Only use `ImplementationSpecific` if your controller docs explicitly require it.

---

## 7. Interview Answers — Compressed, Verbatim-Ready

**Q: What is the difference between a Kubernetes Service and an Ingress?**

> "A Service operates at Layer 4 and exposes a stable endpoint for a set of Pods — it handles TCP and UDP traffic. An Ingress operates at Layer 7 and provides HTTP-level routing features like path-based and host-based routing, TLS termination, and the ability to front multiple services with a single external load balancer. You'd use a Service for any traffic, but you'd use Ingress specifically for HTTP and HTTPS workloads where you want smarter, cheaper routing."

---

**Q: What is an Ingress Controller and why is one required?**

> "The Ingress resource is purely declarative — it describes what should happen but does nothing on its own. The Ingress Controller is the active component that watches Ingress objects in the API server and translates those rules into real load balancer configuration. Without a controller, the Ingress object just sits in etcd doing nothing. Kubernetes intentionally ships no built-in controller, giving you the flexibility to choose one appropriate for your environment — NGINX for on-prem, AWS Load Balancer Controller for EKS, Traefik if you need automatic TLS, and so on."

---

**Q: What's the difference between `target-type: ip` and `target-type: instance` in AWS ALB?**

> "With `target-type: ip`, the ALB registers individual Pod IPs directly in its target groups. This is possible on EKS because the VPC CNI plugin gives each Pod a real routable IP from the VPC subnet. Traffic goes straight from the ALB to the Pod, bypassing kube-proxy and NodePort. With `target-type: instance`, the ALB registers EC2 node IPs and routes to NodePorts, which then go through kube-proxy to reach pods. The `ip` mode has fewer hops and supports pod-level health checks, but requires VPC CNI. The `instance` mode works with any CNI but adds network overhead."

---

**Q: Why would you run multiple Ingress controllers in the same cluster?**

> "There are a few real-world reasons. First, protocol requirements — you might need NGINX for an application using non-HTTP traffic, while ALB handles your HTTP services. Second, multi-tenancy — in a shared cluster, separate teams may want their own ingress controller for operational independence. Third, environment segmentation — production might use ALB with strict compliance settings while staging uses NGINX with relaxed rules. The `IngressClass` resource is what makes this possible; each Ingress object declares which controller should handle it via `spec.ingressClassName`."

---

**Q: What is IngressClass and when is it important?**

> "IngressClass is a Kubernetes resource that maps a class name to a specific Ingress controller. It's the mechanism by which a cluster can have multiple controllers coexist without conflict. When you set `spec.ingressClassName: alb` on an Ingress, only the AWS Load Balancer Controller — which is registered as the handler for the `alb` class — will reconcile that Ingress. If you don't specify a class, Kubernetes will use whichever controller is marked as the default, which can lead to unpredictable behavior in multi-controller environments."

---

**Q: How does path-based routing work in Ingress?**

> "In an Ingress spec, you define a list of routing rules under `spec.rules`. Each rule can specify a `host` and a list of `paths`. Each path entry has a `pathType` — either `Exact`, `Prefix`, or `ImplementationSpecific` — and a `backend` pointing to a Kubernetes Service. When a request comes in, the Ingress controller (not Kubernetes itself) evaluates these rules in order. For AWS ALB, it creates listener rules in the ALB. For NGINX, it generates upstream config in nginx.conf. The key operational point: more specific paths must come before the catch-all, because a `Prefix: /` rule will match everything if it appears first."

---

**Q: What does `topologySpreadConstraints` do in these deployments?**

> "It's a scheduler hint that distributes pods evenly across topology domains — in this case, availability zones. `maxSkew: 1` means the difference in pod count between any two zones must be at most 1. `whenUnsatisfiable: ScheduleAnyway` makes it a soft constraint — if perfect balance isn't achievable, schedule anyway rather than block. The alternative, `DoNotSchedule`, is a hard constraint and can cause pods to pend indefinitely. In production, this matters because without it, all pods for a service could land in one AZ, creating a single point of failure that Kubernetes won't automatically detect or fix."

---

## 8. Debugging — Fast Diagnosis Paths

### The Ingress Debugging Decision Tree

```
Problem: "My Ingress doesn't work"
          │
          ▼
Step 1: Is the ADDRESS field populated?
  kubectl get ingress -n <ns>
          │
    ├── NO ADDRESS
    │       │
    │       ├── Does IngressClass exist?
    │       │   kubectl get ingressclass
    │       │   └── NO → install controller / check ingressClassName annotation
    │       │
    │       ├── Is controller running?
    │       │   kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
    │       │   └── NO → check Helm install, IRSA setup
    │       │
    │       └── Check controller logs for errors
    │           kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
    │           Common errors: AccessDenied, no OIDC, missing annotations
    │
    └── ADDRESS EXISTS but wrong routing
            │
            ├── Which path is broken?
            │   curl -v http://<ALB>/iphone
            │
            ├── Check Ingress spec rules (order matters!)
            │   kubectl describe ingress <name> -n <ns>
            │
            ├── Are backend Services healthy?
            │   kubectl get endpoints -n <ns>
            │   # Empty = no pods matching selector
            │
            ├── Are pods running?
            │   kubectl get pods -n <ns> -l app=iphone-page
            │
            ├── Is health check path correct?
            │   # Check ALB target group health in AWS Console
            │   # Or: kubectl describe svc iphone-svc → check annotations
            │
            └── target-type: ip but pods unreachable?
                # Verify VPC CNI is in use
                kubectl get pods -n kube-system | grep aws-node
                # If not found: switch to target-type: instance
```

### Essential Debug Commands

```bash
# --- Ingress status ---
kubectl get ingress -n app1-ns
kubectl describe ingress ingress-demo1 -n app1-ns   # Check events section!

# --- Controller health ---
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller --tail=50

# --- Service + endpoint check ---
kubectl get svc -n app1-ns
kubectl get endpoints -n app1-ns   # Empty = selector mismatch or no running pods

# --- Pod health ---
kubectl get pods -n app1-ns -o wide
kubectl describe pod <pod-name> -n app1-ns

# --- Test routing from inside cluster ---
kubectl run debug --rm -it --image=curlimages/curl -- sh
curl http://iphone-svc.app1-ns.svc.cluster.local/iphone

# --- IngressClass check ---
kubectl get ingressclass
kubectl describe ingressclass alb

# --- IRSA / ServiceAccount check ---
kubectl get sa -n kube-system aws-load-balancer-controller -o yaml
# Look for: eks.amazonaws.com/role-arn annotation

# --- Helm check ---
helm list -n kube-system
helm status aws-load-balancer-controller -n kube-system

# --- Node AZ distribution ---
kubectl get nodes -L topology.kubernetes.io/zone

# --- Pod AZ distribution ---
kubectl get pods -n app1-ns -o wide --sort-by='.spec.nodeName'
```

### Common Error → Fix Quick Reference

| Error / Symptom | Likely Cause | Fix |
|---|---|---|
| Ingress ADDRESS empty | No controller or wrong `ingressClassName` | Install controller; check `kubectl get ingressclass` |
| ALB created but 502/503 | Target group unhealthy (wrong health check path) | Fix `healthcheck-path` annotation on Service |
| All traffic goes to one backend | Catch-all `/` path is first in rules | Reorder: specific paths first, `/` last |
| `AccessDenied` in LBC logs | IRSA misconfigured; no IAM Role annotated | Re-run `eksctl create iamserviceaccount` |
| `target-type: ip` targets unhealthy | Non-VPC-CNI pod IPs not routable | Switch to `target-type: instance` |
| Pods all in one AZ | Missing `topologySpreadConstraints` | Add spread constraints to Deployment spec |
| NGINX works, ALB doesn't | Missing `ingressClassName: alb` annotation | Add `.spec.ingressClassName: alb` |

---

## 9. Kill Switch — 10-Second Recall

```
┌─────────────────────────────────────────────────────────┐
│                  INGRESS BRAIN DUMP                      │
├─────────────────────────────────────────────────────────┤
│  Ingress = L7 declarative spec (YAML only, no traffic)  │
│  Controller = the thing that actually routes traffic     │
│                                                         │
│  Cloud-native: ALB provisioned OUTSIDE cluster          │
│  3rd-party: NGINX runs INSIDE cluster                   │
│                                                         │
│  target-type: ip   = Pod IP direct (needs VPC CNI)      │
│  target-type: instance = via NodePort (any CNI)         │
│                                                         │
│  ingressClassName → picks which controller owns it      │
│                                                         │
│  PATH ORDER MATTERS: specific first, / last             │
│  HEALTH CHECK PATH: per-service annotation on SVC       │
│  IRSA: LBC needs IAM role via ServiceAccount annotation │
│                                                         │
│  Ingress broken? Check: ADDRESS → logs → endpoints      │
└─────────────────────────────────────────────────────────┘
```

---

## 10. Appendix — Quick Reference Card

### Ingress API Fields

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: <name>
  namespace: <ns>
  annotations: {}         # Controller-specific behavior
spec:
  ingressClassName: alb   # Which controller handles this
  defaultBackend:         # Optional: catch-all if no rules match
    service:
      name: fallback-svc
      port:
        number: 80
  tls:                    # Optional: TLS config
  - hosts:
    - myapp.com
    secretName: myapp-tls  # k8s Secret with tls.crt + tls.key
  rules:
  - host: myapp.com       # Optional: host-based matching
    http:
      paths:
      - path: /api
        pathType: Prefix   # Exact | Prefix | ImplementationSpecific
        backend:
          service:
            name: api-svc
            port:
              number: 80
```

### AWS LBC Key Annotations Cheatsheet

| Annotation | Values | Purpose |
|---|---|---|
| `alb.ingress.kubernetes.io/scheme` | `internet-facing`, `internal` | Public vs private ALB |
| `alb.ingress.kubernetes.io/target-type` | `ip`, `instance` | How targets registered in ALB |
| `alb.ingress.kubernetes.io/listen-ports` | `'[{"HTTP":80},{"HTTPS":443}]'` | ALB listener ports |
| `alb.ingress.kubernetes.io/certificate-arn` | `arn:aws:acm:...` | ACM cert for HTTPS |
| `alb.ingress.kubernetes.io/ssl-redirect` | `'443'` | Redirect HTTP → HTTPS |
| `alb.ingress.kubernetes.io/healthcheck-path` | `/health` | Health check URL |
| `alb.ingress.kubernetes.io/load-balancer-name` | `my-alb` | Custom ALB name in AWS |
| `alb.ingress.kubernetes.io/group.name` | `shared-alb` | Share ALB across Ingresses |

### pathType Quick Reference

| Input URL | path: `/foo`, pathType: `Prefix` | path: `/foo`, pathType: `Exact` |
|---|---|---|
| `/foo` | ✅ | ✅ |
| `/foo/` | ✅ | ❌ |
| `/foo/bar` | ✅ | ❌ |
| `/foobar` | ❌ | ❌ |

### Imperative Commands (CKA Exam)

```bash
# Create an ingress imperatively (basic)
kubectl create ingress my-ingress \
  --class=nginx \
  --rule="myapp.com/api*=api-svc:80" \
  --rule="myapp.com/*=frontend-svc:80" \
  --namespace=production

# Expose a deployment as ClusterIP service (pre-req for ingress)
kubectl expose deployment my-deploy --port=80 --target-port=8080 --name=my-svc

# Quick test from inside cluster
kubectl run tmp-debug --rm -it --image=curlimages/curl -- curl http://my-svc/api

# Get ingress with wide output
kubectl get ingress -A -o wide

# Check what IngressClasses are available
kubectl get ingressclass -o yaml
```

### Troubleshooting Shorthand

```bash
# The 4 commands to run first
kubectl get ingress -n <ns>                          # 1. Does ADDRESS exist?
kubectl describe ingress <name> -n <ns>              # 2. Any events? Rules correct?
kubectl get endpoints -n <ns>                        # 3. Do services have backends?
kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller  # 4. Controller errors?
```

---
