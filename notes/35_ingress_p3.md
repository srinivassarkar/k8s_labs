# 🚦 Kubernetes Ingress: TLS & Subdomain Routing Mastery


> *"Ingress is where Kubernetes stops being a container platform and starts being a production system."*

---

## 0. First Principles 🧠
*The mental model that never changes — anchor everything here*

Kubernetes has no native "external traffic routing" concept inside the cluster spec. Ingress fills exactly that gap with one elegant contract:

> **One external entry point → Many internal services, routed by rules.**

The three laws of Ingress that never bend:

| Law | What It Means |
|-----|---------------|
| **Ingress ≠ Load Balancer** | Ingress is a *spec*. The Ingress Controller is the *engine* that makes it real. |
| **Controller = Cloud Native Glue** | On EKS, the AWS Load Balancer Controller reads your Ingress spec and provisions an ALB. The controller is doing the heavy lifting. |
| **Annotations = Controller Config** | Standard `networking.k8s.io/v1` Ingress has ~10 fields. Everything else (TLS redirect, health checks, target type) lives in controller-specific annotations. |

**The immutable hierarchy:**

```
Internet
    ↓
DNS (Route 53) resolves hostname → ALB IP
    ↓
ALB (provisioned by AWS LB Controller reading Ingress annotations)
    ↓
Listener Rules (path or host match → target group)
    ↓
Pod IPs directly (target-type: ip) OR Node ports (target-type: instance)
    ↓
Container port
```

This chain never changes. Debug from top to bottom, always.

---

## 1. Reality Constraints 🔒
*What Kubernetes actually does — and doesn't do*

### What Ingress IS

- A Kubernetes API object (`kind: Ingress`) that declares *desired* routing behavior
- A trigger for the Ingress Controller to reconcile infrastructure
- The single place where TLS termination, path rules, and host rules are declared

### What Ingress IS NOT

- A running process or daemon inside a pod (that's the controller)
- A replacement for NetworkPolicy (Ingress doesn't control pod-to-pod traffic)
- Automatically functional — **you MUST install an Ingress Controller** or nothing works

### EKS-Specific Reality

| Constraint | Detail |
|------------|--------|
| Controller must be installed separately | AWS Load Balancer Controller via Helm or manifests |
| IRSA required | Controller needs IAM permissions to create/modify ALBs — OIDC must be enabled on the cluster |
| One ALB per Ingress (default) | Multiple Ingresses can share one ALB via `IngressGroup` annotation |
| `target-type: ip` requires VPC CNI | Pods must be in the VPC CIDR, not overlay-networked |
| ACM certs must be in the same region | Cross-region ACM certs won't work with ALB |
| Apex domain (`cwvj.click`) ≠ wildcard (`*.cwvj.click`) | You need **two certificates** or a cert with both SANs |

### TLS Reality

```
Browser → ALB:443 → TLS terminates HERE → ALB → Pod:5678 (plain HTTP)
```

The ALB terminates TLS. Your pods speak plain HTTP. The cert lives in ACM, not in a Kubernetes Secret (for ALB). This is **different from nginx-ingress** where you'd use a `tls:` block with a Secret.

---

## 2. Decision Logic 🔀
*When to use what — clear rules, no ambiguity*

### Routing Strategy Decision Tree

```
Traffic arrives at domain...
         │
         ▼
Is the path different?          e.g. /iphone, /android, /
   YES ──────────────────────→  PATH-BASED ROUTING (Demo 2)
         │
        NO
         │
         ▼
Is the hostname different?      e.g. iphone.cwvj.click, android.cwvj.click
   YES ──────────────────────→  HOST-BASED (NAME-BASED) ROUTING (Demo 3)
         │
        NO
         │
         ▼
Single backend? → No Ingress needed → Use a LoadBalancer Service
```

### Path-Based vs Host-Based: When to Choose

| Factor | Path-Based | Host-Based |
|--------|-----------|------------|
| DNS overhead | Minimal — 1 DNS record | Higher — 1 record per subdomain |
| App isolation | Apps share the same domain | Each app has clean domain identity |
| TLS certificates | 1 cert for apex domain | Need wildcard or per-subdomain certs |
| URL structure | `app.com/iphone` | `iphone.app.com` |
| Production use | Internal tools, API versioning | SaaS products, multi-tenant apps |
| Backend complexity | Backend must serve content at subpath | Backend serves at `/` — simpler |

### ALB Target Type Decision

| Use `target-type: ip` when... | Use `target-type: instance` when... |
|-------------------------------|-------------------------------------|
| EKS with VPC CNI (standard EKS) | Kops, kubeadm, non-VPC-CNI setups |
| You want direct pod health checks | NodePort-based routing is acceptable |
| You need fine-grained pod lifecycle | |

> ✅ **Rule of thumb for EKS**: Always use `target-type: ip`. It routes directly to pod IPs, bypasses kube-proxy, and gives faster health convergence.

### Certificate Strategy

```
Scenario                          Certificate Strategy
─────────────────────────────────────────────────────────────────────
Single apex domain (cwvj.click)   1 cert: cwvj.click
Apex + wildcard subdomains        2 certs: cwvj.click AND *.cwvj.click
                                  OR 1 cert with both SANs
Multiple unrelated domains        1 cert per domain OR multi-SAN cert
```

---

## 3. Internal Working ⚙️
*How it actually happens, step by step*

### What Happens When You `kubectl apply -f ingress.yaml`

```
Step 1: kubectl → kube-apiserver
   → Validates Ingress spec against networking.k8s.io/v1 schema
   → Stores object in etcd

Step 2: AWS Load Balancer Controller (watching via Informer)
   → Detects new/updated Ingress with ingressClassName: alb
   → Reads all annotations to build desired ALB state

Step 3: Controller calls AWS EC2/ELB APIs
   → Creates ALB (if not exists) with name from annotation
   → Creates Target Groups (one per backend Service)
   → Registers pod IPs as targets (target-type: ip)
   → Creates Listeners (80, 443 per listen-ports annotation)
   → Attaches ACM cert to HTTPS listener
   → Creates Listener Rules (host + path conditions → target group)

Step 4: SSL Redirect Rule
   → Creates a redirect rule on HTTP:80 listener
   → 301 redirect all traffic → HTTPS (from ssl-redirect: '443')

Step 5: Health Check Convergence
   → ALB pings each pod at healthcheck-path every 15s
   → After 2 consecutive 200s → pod marked healthy
   → Only healthy targets receive traffic

Step 6: DNS (Route 53)
   → Alias record cwvj.click → ALB DNS name
   → TTL is low for alias records (inherits from ALB)
   → Browser resolves → ALB IP → request flows
```

### How Host-Based Routing Works in the ALB

The AWS LB Controller translates Ingress `rules[].host` + `rules[].http.paths` into **ALB Listener Rules** with conditions:

```
HTTPS:443 Listener Rules (in priority order):
┌──────────────────────────────────────────────────────────┐
│ Priority 1: IF host = iphone.cwvj.click AND path = /*    │
│             THEN forward → iphone-svc target group        │
├──────────────────────────────────────────────────────────┤
│ Priority 2: IF host = android.cwvj.click AND path = /*   │
│             THEN forward → android-svc target group       │
├──────────────────────────────────────────────────────────┤
│ Priority 3: IF host = cwvj.click AND path = /*           │
│             THEN forward → desktop-svc target group       │
├──────────────────────────────────────────────────────────┤
│ Default:    Return 404 (no match)                        │
└──────────────────────────────────────────────────────────┘

HTTP:80 Listener:
┌──────────────────────────────────────────────────────────┐
│ Default Action: Redirect to HTTPS:443 (301)              │
└──────────────────────────────────────────────────────────┘
```

### TopologySpreadConstraints — What It Does

Every deployment in these demos uses:
```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: ScheduleAnyway
```

This tells the scheduler: *"Try to spread pods evenly across AZs, but if you can't, schedule anyway."* With 2 replicas and 2 AZs (`us-east-2a`, `us-east-2b`), you get 1 pod per AZ — surviving an AZ failure.

---

## 4. Hands-On 🛠️
*Production-quality YAML + commands — nothing simplified*

### EKS Cluster Setup

```yaml
# eks-config.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: cwvj-ingress-demo
  region: us-east-2
  tags:
    owner: varun-joshi
    bu: cwvj
    project: ingress-demo
availabilityZones:
  - us-east-2a
  - us-east-2b
iam:
  withOIDC: true   # CRITICAL: enables IRSA for ALB controller
managedNodeGroups:
  - name: cwvj-eks-priv-ng
    instanceType: t3.small
    minSize: 2
    maxSize: 2
    privateNetworking: true
    volumeSize: 20
    iam:
      withAddonPolicies:
        albIngress: true   # Attaches ALB controller IAM policy
        autoScaler: true
        certManager: true
    labels:
      lifecycle: ec2-autoscaler
```

```bash
# Create cluster
eksctl create cluster -f eks-config.yaml

# Install AWS Load Balancer Controller
helm repo add eks https://aws.github.io/eks-charts
helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=cwvj-ingress-demo \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

### Namespace

```yaml
# 01-ns.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: app1-ns
```

### Demo 2 — Path-Based Routing with TLS

```yaml
# 02-iphone.yaml (path-based: content served at /iphone)
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
              <body><h1>iPhone Users</h1>
              <p>Welcome to Cloud With VarJosh</p>
              </body></html>' > /iphone/index.html && cd / && python3 -m http.server 5678
          ports:
            - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: iphone-svc
  namespace: app1-ns
  annotations:
    alb.ingress.kubernetes.io/healthcheck-path: /iphone/index.html
spec:
  selector:
    app: iphone-page
  ports:
    - protocol: TCP
      port: 80
      targetPort: 5678
```

```yaml
# 05-ingress.yaml (Demo 2: path-based + TLS)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-demo2
  namespace: app1-ns
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/load-balancer-name: cwvj-ingress-demo2
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}, {"HTTP":80}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-2:ACCOUNT:certificate/CERT-ID
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    alb.ingress.kubernetes.io/healthcheck-protocol: HTTP
    alb.ingress.kubernetes.io/healthcheck-port: traffic-port
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: '15'
    alb.ingress.kubernetes.io/healthcheck-timeout-seconds: '5'
    alb.ingress.kubernetes.io/healthy-threshold-count: '2'
    alb.ingress.kubernetes.io/unhealthy-threshold-count: '2'
    alb.ingress.kubernetes.io/success-codes: '200'
spec:
  ingressClassName: alb
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
          - path: /
            pathType: Prefix
            backend:
              service:
                name: desktop-svc
                port:
                  number: 80
```

### Demo 3 — Host-Based (Subdomain) Routing with TLS

```yaml
# 02-iphone.yaml (host-based: content at root /)
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
              echo '<html>
              <head><title>iPhone Users</title></head>
              <body><h1>iPhone Users</h1>
              <p>Welcome to Cloud With VarJosh</p>
              </body></html>' > /index.html && python3 -m http.server 5678
          ports:
            - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: iphone-svc
  namespace: app1-ns
  annotations:
    alb.ingress.kubernetes.io/healthcheck-path: /index.html  # root now
spec:
  selector:
    app: iphone-page
  ports:
    - protocol: TCP
      port: 80
      targetPort: 5678
```

```yaml
# 05-ingress.yaml (Demo 3: host-based + dual TLS certs)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-demo3
  namespace: app1-ns
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/load-balancer-name: cwvj-ingress-demo3
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}, {"HTTP":80}]'
    # Two certs: apex + wildcard (comma-separated)
    alb.ingress.kubernetes.io/certificate-arn: >-
      arn:aws:acm:us-east-2:ACCOUNT:certificate/APEX-CERT-ID,
      arn:aws:acm:us-east-2:ACCOUNT:certificate/WILDCARD-CERT-ID
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    alb.ingress.kubernetes.io/healthcheck-protocol: HTTP
    alb.ingress.kubernetes.io/healthcheck-port: traffic-port
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: '15'
    alb.ingress.kubernetes.io/healthcheck-timeout-seconds: '5'
    alb.ingress.kubernetes.io/healthy-threshold-count: '2'
    alb.ingress.kubernetes.io/unhealthy-threshold-count: '2'
    alb.ingress.kubernetes.io/success-codes: '200'
spec:
  ingressClassName: alb
  rules:
    - host: iphone.cwvj.click
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: iphone-svc
                port:
                  number: 80
    - host: android.cwvj.click
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: android-svc
                port:
                  number: 80
    - host: cwvj.click
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: desktop-svc
                port:
                  number: 80
```

### Key Commands

```bash
# Apply all resources in directory order
kubectl apply -f .

# Watch ingress get its ALB address
kubectl get ingress -n app1-ns -w

# Describe ingress for events and address
kubectl describe ingress ingress-demo3 -n app1-ns

# Check all pods are running and spread across AZs
kubectl get pods -n app1-ns -o wide

# Verify services and endpoints
kubectl get svc,ep -n app1-ns

# Test routing (replace with actual ALB DNS or domain)
curl -H "Host: iphone.cwvj.click" https://cwvj-ingress-demo3-XXXX.us-east-2.elb.amazonaws.com/ -k
curl https://iphone.cwvj.click/

# Check ALB controller logs
kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller --tail=50

# Cleanup
kubectl delete -f .
eksctl delete cluster --name cwvj-ingress-demo --region us-east-2
```

---

## 5. Production Flow 🏭
*Real-world architecture and design patterns*

### Production-Grade Ingress Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│  INTERNET                                                           │
│       │                                                             │
│  ┌────▼────────────────────────────────────────────────────────┐   │
│  │  Route 53                                                    │   │
│  │  iphone.cwvj.click  ──┐                                      │   │
│  │  android.cwvj.click ──┼──→ Alias A → ALB DNS name           │   │
│  │  cwvj.click         ──┘                                      │   │
│  └──────────────────────────────────────────────────────────────┘   │
│       │                                                             │
│  ┌────▼────────────────────────────────────────────────────────┐   │
│  │  AWS ALB (internet-facing, multi-AZ)                        │   │
│  │  ┌─────────────────────────┐  ┌──────────────────────────┐  │   │
│  │  │ HTTP:80 Listener        │  │ HTTPS:443 Listener       │  │   │
│  │  │ Default: Redirect→443   │  │ ACM Cert: cwvj.click     │  │   │
│  │  └─────────────────────────┘  │ ACM Cert: *.cwvj.click   │  │   │
│  │                               │ Rules:                   │  │   │
│  │                               │  iphone.* → iphone-tg   │  │   │
│  │                               │  android.* → android-tg  │  │   │
│  │                               │  cwvj.click → desktop-tg │  │   │
│  │                               └──────────────────────────┘  │   │
│  └────────────────┬──────────────────────┬──────────────────────┘   │
│                   │                      │                          │
│  ┌────────────────▼──────────────────────▼───────────────────────┐  │
│  │  EKS Private Node Group (us-east-2a + us-east-2b)            │  │
│  │  ┌──────────────────┐  ┌──────────────────┐  ┌────────────┐  │  │
│  │  │ iphone pods      │  │ android pods     │  │desktop pods│  │  │
│  │  │ AZ-a + AZ-b      │  │ AZ-a + AZ-b     │  │AZ-a + AZ-b│  │  │
│  │  │ port: 5678       │  │ port: 5678       │  │ port: 5678 │  │  │
│  │  └──────────────────┘  └──────────────────┘  └────────────┘  │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### Production Design Patterns

**Pattern 1: ExternalDNS for automated DNS**
Instead of manually creating Route 53 records, deploy ExternalDNS controller. It watches Ingress resources and auto-creates/deletes Route 53 records.

```bash
# ExternalDNS annotation on Ingress
external-dns.alpha.kubernetes.io/hostname: iphone.cwvj.click
```

**Pattern 2: IngressGroup for shared ALB**
Multiple Ingresses → single ALB. Reduces cost and IP footprint.

```yaml
annotations:
  alb.ingress.kubernetes.io/group.name: my-shared-alb
  alb.ingress.kubernetes.io/group.order: '10'  # Rule priority within group
```

**Pattern 3: WAF + Shield integration**
```yaml
annotations:
  alb.ingress.kubernetes.io/wafv2-acl-arn: arn:aws:wafv2:...
```

**Pattern 4: Cert-Manager for nginx-ingress (non-ALB)**
When not on AWS ALB, use cert-manager with Let's Encrypt:
```yaml
# On the Ingress spec
tls:
  - hosts:
      - iphone.example.com
    secretName: iphone-tls-secret   # cert-manager populates this
annotations:
  cert-manager.io/cluster-issuer: letsencrypt-prod
```

---

## 6. Mistakes 💀
*What actually breaks in real systems — root cause + fix*

### Mistake 1: Apex domain not covered by wildcard cert

**Symptom:** `https://cwvj.click` shows SSL error / browser warning. `https://iphone.cwvj.click` works fine.

**Root Cause:** `*.cwvj.click` does NOT cover `cwvj.click`. Wildcard only covers one subdomain level.

**Fix:** Request two certificates — one for `cwvj.click`, one for `*.cwvj.click` — and provide both ARNs comma-separated in the annotation.

```yaml
alb.ingress.kubernetes.io/certificate-arn: arn:...:APEX-CERT,arn:...:WILDCARD-CERT
```

---

### Mistake 2: Health check path mismatch between Demo 2 and Demo 3

**Symptom:** Targets stay unhealthy. ALB returns 502/504.

**Root Cause:**
- Demo 2 serves content at `/iphone/index.html` — health check must be `/iphone/index.html`
- Demo 3 serves content at `/index.html` — health check must be `/index.html`
- Using Demo 2's service annotations with Demo 3 deployments → 404 on health check → targets never become healthy.

**Fix:** Align health check path with actual file location per deployment version.

---

### Mistake 3: OIDC not enabled — ALB controller can't assume IAM role

**Symptom:** Controller pod is running but no ALB ever gets created. Controller logs show `UnauthorizedOperation`.

**Root Cause:** IRSA requires OIDC provider to be registered with IAM. Without `withOIDC: true`, service account annotations don't grant AWS API access.

**Fix:**
```bash
eksctl utils associate-iam-oidc-provider --cluster cwvj-ingress-demo --approve
```

---

### Mistake 4: `ingressClassName: alb` missing

**Symptom:** Ingress created, address stays empty (`<none>`). No ALB appears.

**Root Cause:** The AWS LB Controller only processes Ingresses that explicitly claim `ingressClassName: alb`. Without it, no controller picks up the object.

**Fix:** Add to spec:
```yaml
spec:
  ingressClassName: alb
```

---

### Mistake 5: Path ordering — specific paths after catch-all

**Symptom:** All traffic hits desktop (catch-all `/`) even when hitting `/iphone`.

**Root Cause:** In path-based routing, ordering matters. If `/` is listed first, it matches everything before `/iphone` is evaluated.

**Fix:** Always list specific paths before the catch-all:
```yaml
paths:
  - path: /iphone     # specific first
  - path: /android    # specific second
  - path: /           # catch-all last
```

---

### Mistake 6: HTTP health check on HTTPS listener

**Symptom:** Targets cycle between healthy/unhealthy. Intermittent 502s.

**Root Cause:** `healthcheck-protocol: HTTPS` with self-signed certs or no cert on the pod fails TLS validation.

**Fix:** Keep health check protocol as HTTP. ALB → pod traffic is HTTP (TLS terminates at ALB):
```yaml
alb.ingress.kubernetes.io/healthcheck-protocol: HTTP
```

---

## 7. Interview Answers 🎤
*Compressed, verbatim-ready — say these out loud*

---

**Q: What is a Kubernetes Ingress and how does it differ from a LoadBalancer Service?**

"An Ingress is a Kubernetes API object that defines layer-7 HTTP routing rules — things like path-based or host-based routing, TLS termination, and virtual hosting. A LoadBalancer Service operates at layer 4 and simply exposes a single service on an external IP. The key difference is that a LoadBalancer Service gives you one external IP per service, which is expensive and hard to manage at scale. An Ingress lets you share a single external entry point — like one ALB — across many services, routing traffic based on URL path or hostname. On EKS, the AWS Load Balancer Controller reads the Ingress spec and provisions the ALB automatically."

---

**Q: What is an Ingress Controller and why do you need one?**

"The Ingress resource is just a declaration — it's a YAML object that says what you want. The Ingress Controller is what actually does the work. It's a running process, usually a Deployment in the cluster, that watches for Ingress objects and reconciles the underlying infrastructure. On EKS, that's the AWS Load Balancer Controller creating and managing ALBs. On bare metal or other clouds, you'd use nginx-ingress, Traefik, or similar. Without a controller, you can apply Ingress manifests all day and nothing will happen — the objects just sit in etcd."

---

**Q: Explain the difference between path-based and host-based routing in Ingress.**

"Path-based routing routes traffic based on the URL path — so `app.com/iphone` goes to one service and `app.com/android` goes to another. Everything shares the same hostname. Host-based routing, also called name-based routing, routes based on the full hostname — so `iphone.app.com` goes to one service and `android.app.com` goes to another. Each app gets its own subdomain. In the Ingress spec, path-based uses the `paths` field without a `host`, while host-based adds a `host` field to each rule. For host-based routing on HTTPS, you need certificates that cover all the subdomains — typically a wildcard cert."

---

**Q: How does TLS work with the AWS ALB Ingress Controller?**

"TLS terminates at the ALB. You provision a certificate in AWS Certificate Manager, reference its ARN in the Ingress annotation, and the ALB presents that certificate to clients. The traffic from the ALB to your pods is plain HTTP — your containers don't need to speak TLS. To force HTTPS, you use the ssl-redirect annotation, which configures the HTTP:80 listener to send a 301 redirect to HTTPS. This is fundamentally different from nginx-ingress, where you'd put the certificate in a Kubernetes Secret and reference it in the Ingress tls block."

---

**Q: Why do you need two certificates for subdomain routing — one for the apex domain and one wildcard?**

A wildcard certificate like `*.cwvj.click` only covers one level of subdomains (e.g., `iphone.cwvj.click`, `api.cwvj.click`) and does **not** cover the apex domain `cwvj.click`. If a user accesses `cwvj.click` over HTTPS using only a wildcard cert, the browser will show a certificate mismatch error because the apex domain isn’t included.

To properly cover both the apex domain and all first-level subdomains, you have two options:

* **Single certificate with multiple SANs (recommended):**
  Request one certificate that includes both `cwvj.click` and `*.cwvj.click`.

* **Two separate certificates:**
  One certificate for `cwvj.click` and another wildcard certificate for `*.cwvj.click`.

With AWS Application Load Balancer (ALB), you can attach either a single certificate with both names or multiple certificates. The ALB uses SNI (Server Name Indication) to present the correct certificate based on the requested hostname.

**In short:** A wildcard certificate alone is not enough for the apex domain—but you don’t necessarily need two certificates; one certificate with both entries is sufficient.


---

**Q: What is `topologySpreadConstraints` and why is it used in these deployments?**

"TopologySpreadConstraints tells the Kubernetes scheduler how to distribute pods across topology domains — in this case, AWS availability zones. With `maxSkew: 1` and `topologyKey: topology.kubernetes.io/zone`, we're saying the difference in pod count between any two zones should be at most 1. With `whenUnsatisfiable: ScheduleAnyway`, if perfect spread isn't possible — say because a zone has no capacity — the scheduler will still place the pod rather than leave it Pending. The practical effect is that with 2 replicas across 2 AZs, you get one pod per AZ, giving you availability zone redundancy. If AZ-a goes down, AZ-b's pod keeps serving traffic."

---

**Q: What is IRSA and why does the ALB controller need it?**

"IRSA stands for IAM Roles for Service Accounts. It's a mechanism that lets Kubernetes service accounts assume AWS IAM roles without distributing long-lived credentials. The AWS Load Balancer Controller needs to make AWS API calls — creating ALBs, registering targets, managing security groups. To do this securely, we create an IAM role with the necessary permissions, annotate the controller's Kubernetes service account with that role ARN, and enable OIDC on the EKS cluster. The OIDC provider lets AWS IAM trust tokens issued by the EKS API server. Without this, the controller either uses instance role permissions — which is overly broad — or fails entirely."

---

## 8. Debugging 🔍
*Fast diagnosis paths — follow the chain*

### Master Diagnostic Flow

```
SYMPTOM: Traffic not reaching backend
         │
         ▼
1. Does the Ingress have an ADDRESS?
   kubectl get ingress -n app1-ns
   │
   ├─ NO ADDRESS → ALB not provisioned
   │   Check: kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
   │   Common causes: OIDC not enabled, IAM policy missing, ingressClassName wrong
   │
   └─ HAS ADDRESS → ALB exists, go deeper
         │
         ▼
2. Can you curl the ALB directly?
   curl http://<ALB-DNS>/ -H "Host: iphone.cwvj.click"
   │
   ├─ 404 → Rules not matching. Check listener rules in AWS Console.
   │   Check: pathType, host field spelling, path ordering
   │
   ├─ 502/503 → Targets unhealthy
   │   Check: AWS Console → Target Groups → health status
   │   kubectl exec -n app1-ns <pod> -- curl localhost:5678/index.html
   │   Verify: health check path matches actual file location
   │
   ├─ 301 on HTTPS → SSL redirect working (expected on HTTP port)
   │
   └─ TLS error → Certificate issue
         │
         ▼
3. DNS resolving correctly?
   dig iphone.cwvj.click
   nslookup iphone.cwvj.click
   Should return: CNAME or A → ALB DNS name
   If not: check Route 53 alias records
```

### Command Cheatsheet for Debugging

```bash
# 1. Check ingress status and events
kubectl describe ingress ingress-demo3 -n app1-ns

# 2. Check ALB controller logs (most useful for provisioning issues)
kubectl logs -n kube-system \
  -l app.kubernetes.io/name=aws-load-balancer-controller \
  --tail=100 | grep -i error

# 3. Check pod health directly
kubectl get pods -n app1-ns -o wide
kubectl exec -n app1-ns <pod-name> -- curl -s localhost:5678/index.html

# 4. Verify service endpoints are populated
kubectl get endpoints -n app1-ns
# Empty endpoints = label selector mismatch between Service and Deployment

# 5. Test routing with explicit Host header (bypass DNS)
curl -v -H "Host: iphone.cwvj.click" http://<ALB-DNS>/

# 6. Check for certificate binding on ALB
aws elbv2 describe-listeners \
  --load-balancer-arn <ALB-ARN> \
  --region us-east-2

# 7. Verify IAM role binding (IRSA)
kubectl get sa -n kube-system aws-load-balancer-controller -o yaml
# Should show: eks.amazonaws.com/role-arn annotation

# 8. DNS resolution test
dig +short iphone.cwvj.click
curl -v https://iphone.cwvj.click/ 2>&1 | grep -E "SSL|certificate|Connected"

# 9. Check events across namespace
kubectl get events -n app1-ns --sort-by='.lastTimestamp' | tail -20
```

### Quick Health Check Diagnosis Table

| Symptom | Check | Root Cause |
|---------|-------|------------|
| Ingress `ADDRESS` empty | LB controller logs | IRSA, OIDC, IAM policy |
| 502 Bad Gateway | Target group health in console | Health check path wrong |
| 404 Not Found | ALB listener rules | Path/host mismatch in Ingress |
| SSL cert warning | `dig` + cert subject | Apex not covered by wildcard |
| All traffic hits one backend | Path ordering in Ingress | Catch-all `/` listed first |
| Pods in Pending | `kubectl describe pod` | Node capacity, taints |
| Empty endpoints | `kubectl get ep` | Label selector mismatch |

---

## 9. Kill Switch ⚡
*10-second recall — burn this into memory*

```
INGRESS = RULES (in k8s) + CONTROLLER (does the work)
ALB CONTROLLER = reads Ingress → creates ALB + listener rules + target groups
PATH-BASED = paths[], no host field
HOST-BASED = host: iphone.cwvj.click in each rule
TLS on ALB = ACM cert ARN in annotation, terminates at ALB, pods get plain HTTP
WILDCARD *.cwvj.click ≠ apex cwvj.click → need 2 certs (or multi-SAN)
IRSA = service account + IAM role + OIDC (needed for controller to call AWS APIs)
HEALTH CHECK PATH must match actual file location in container
TOPOLOGY SPREAD = AZ redundancy, maxSkew:1 means ±1 pod per zone
target-type: ip = direct to pod IPs (EKS best practice)
ingressClassName: alb = required for controller to pick up the Ingress
```

---

## 10. Appendix 📋
*Quick reference card*

### Annotation Reference Card

```yaml
# === CORE ALB CONFIG ===
alb.ingress.kubernetes.io/scheme: internet-facing          # or internal
alb.ingress.kubernetes.io/load-balancer-name: my-alb      # custom ALB name
alb.ingress.kubernetes.io/target-type: ip                  # ip or instance

# === LISTENERS ===
alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}, {"HTTP":80}]'

# === TLS ===
alb.ingress.kubernetes.io/certificate-arn: arn:...,arn:...  # comma-separated
alb.ingress.kubernetes.io/ssl-redirect: '443'               # HTTP→HTTPS redirect

# === HEALTH CHECKS ===
alb.ingress.kubernetes.io/healthcheck-protocol: HTTP
alb.ingress.kubernetes.io/healthcheck-port: traffic-port
alb.ingress.kubernetes.io/healthcheck-path: /health         # OR per-Service annotation
alb.ingress.kubernetes.io/healthcheck-interval-seconds: '15'
alb.ingress.kubernetes.io/healthcheck-timeout-seconds: '5'
alb.ingress.kubernetes.io/healthy-threshold-count: '2'
alb.ingress.kubernetes.io/unhealthy-threshold-count: '2'
alb.ingress.kubernetes.io/success-codes: '200'

# === SHARING ALB ACROSS INGRESSES ===
alb.ingress.kubernetes.io/group.name: shared-alb
alb.ingress.kubernetes.io/group.order: '10'

# === SECURITY ===
alb.ingress.kubernetes.io/wafv2-acl-arn: arn:aws:wafv2:...
```

### Path Type Comparison

| `pathType` | Match Behavior | Example |
|-----------|---------------|---------|
| `Prefix` | Matches path and any subpath | `/foo` matches `/foo`, `/foo/bar`, `/foobar` |
| `Exact` | Matches exact path only | `/foo` matches only `/foo` |
| `ImplementationSpecific` | Controller decides | Varies — avoid in production |

> ⚠️ `Prefix` matching is greedy. `/` matches everything. Always put specific paths before `/`.

### Full Resource Checklist

```
Before deploying Ingress — verify:
[ ] OIDC enabled on EKS cluster
[ ] AWS LB Controller installed and running
[ ] IAM policy attached to controller's service account (IRSA)
[ ] ACM certificate issued and ACTIVE (not pending validation)
[ ] ingressClassName: alb present in Ingress spec
[ ] Service selectors match Deployment pod labels
[ ] Health check path exists in container at container startup
[ ] Correct pathType (Prefix for wildcards, Exact for strict matching)
[ ] Specific paths before catch-all (/) in path-based routing
[ ] Route 53 alias records created pointing to ALB DNS name
[ ] Both apex and wildcard certs if using apex + subdomains
```

### eksctl Quick Reference

```bash
# Create cluster from config
eksctl create cluster -f eks-config.yaml

# Enable OIDC (if forgotten during create)
eksctl utils associate-iam-oidc-provider \
  --cluster cwvj-ingress-demo \
  --approve

# Delete cluster (also removes node groups)
eksctl delete cluster --name cwvj-ingress-demo --region us-east-2
```

### Route 53 DNS Architecture

```
Record Type   Name                   Value                      Use
───────────────────────────────────────────────────────────────────────
A (Alias)     cwvj.click            → ALB DNS name             Apex domain
A (Alias)     iphone.cwvj.click     → ALB DNS name             Subdomain
A (Alias)     android.cwvj.click    → ALB DNS name             Subdomain
CNAME         _acme.cwvj.click      → ACM validation value     Cert validation
```

All subdomains point to the **same ALB**. The ALB's listener rules sort out who goes where based on the `Host:` header in the HTTP request.

---
