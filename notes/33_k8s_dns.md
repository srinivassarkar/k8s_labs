# 🌐 Kubernetes DNS & CoreDNS 

> *"Pods don't talk by IP. They talk by name. CoreDNS is the phone book. Understand it and nothing in Kubernetes networking will surprise you."*

---

## 0. First Principles 🧠

The mental model that never changes:

**Three truths about Kubernetes DNS:**

1. **Every Service gets a DNS A-record automatically.** The moment you create a Service, CoreDNS registers it. No configuration needed.
2. **Pods do NOT get DNS records by default.** You need the `pods` plugin enabled — and even then, it's IP-based, not name-based.
3. **Pods find their DNS server via `/etc/resolv.conf`.** Kubernetes injects this file automatically at pod creation, pointing to the `kube-dns` ClusterIP.

**The resolution model:**
```
Pod queries "backend-svc"
         │
         ▼
/etc/resolv.conf says: nameserver 10.96.0.10
         │
         ▼
CoreDNS at 10.96.0.10 appends search domains one by one:
  → backend-svc.default.svc.cluster.local  ✅ FOUND → returns ClusterIP
  (if not found, tries svc.cluster.local, cluster.local, then external forward)
```

**The hierarchy (read right-to-left):**
```
nginx-svc . default . svc . cluster.local
 hostname   namespace  type    cluster root
```

---

## 1. Reality Constraints ⚠️

### What Kubernetes DNS Actually Does and Doesn't Do

| Fact | Reality |
|------|---------|
| DNS server | CoreDNS — runs as a Deployment in `kube-system`, 2+ replicas |
| Service DNS | Automatically registered for **every** Service — always |
| Pod DNS | **Not created by default.** Requires `pods` plugin in Corefile |
| StatefulSet Pod DNS | **Automatically created** via headless service — no config needed |
| Cluster domain | `cluster.local` by default (configurable at cluster creation) |
| DNS server ClusterIP | Typically `10.96.0.10` — exposed via `kube-dns` Service |
| External DNS | CoreDNS forwards to upstream resolvers in its own `/etc/resolv.conf` |
| `nslookup`/`dig` | **Bypass `/etc/hosts`** — go directly to nameserver |
| `ping`/`curl` | **Consult `/etc/hosts` first**, then DNS (via nsswitch.conf) |
| Port 53 | UDP (primary) + TCP (large responses/zone transfers) |
| Port 9153 | Prometheus metrics for CoreDNS health/performance |

### What CoreDNS CANNOT Do (out of the box)

- Layer 7 DNS policies (no "route based on client identity")
- Per-pod DNS records without plugin configuration
- Dynamic record injection from outside Kubernetes
- TTL lower than `ttl 30` is *technically possible* but risks stale-record issues at scale

### The `ndots:5` Behavior — Frequently Misunderstood

The `/etc/resolv.conf` inside every pod contains `options ndots:5`. This means:

- If the name has **fewer than 5 dots**, the resolver tries search domains **first**, then attempts it as an absolute name.
- If the name has **5 or more dots**, it's treated as a FQDN first (no search domain appending).
- Practical impact: `my.service.com` (2 dots) gets search domains prepended → extra DNS round trips.
- Optimization: Use FQDNs ending in `.` (e.g., `nginx-svc.default.svc.cluster.local.`) to skip search domain iteration entirely.

---

## 2. Decision Logic 🗺️

### Which DNS Name Do I Use?

```
I want to resolve...
│
├── A Service in my OWN namespace?
│   └── Short name works: "nginx-svc"
│       (search domain appends namespace automatically)
│
├── A Service in a DIFFERENT namespace?
│   └── Must include namespace: "nginx-svc.other-ns"
│       (resolves to nginx-svc.other-ns.svc.cluster.local)
│
├── A specific StatefulSet pod?
│   └── Use: "<pod-name>.<headless-svc>.<namespace>.svc.cluster.local"
│       Example: db-0.db-svc.app1-ns.svc.cluster.local
│
├── A pod by IP address?
│   └── Requires pods plugin; use: "<hyphen-ip>.<namespace>.pod.cluster.local"
│       Example: 10-244-2-10.app1-ns.pod.cluster.local
│
└── An external domain (e.g., google.com)?
    └── CoreDNS forwards to upstream — works transparently
```

### FQDN Format Reference

| Resource | Format | Example |
|----------|--------|---------|
| Regular Service | `<name>.<ns>.svc.cluster.local` | `nginx-svc.default.svc.cluster.local` |
| Headless Service | `<name>.<ns>.svc.cluster.local` | `db-svc.app1-ns.svc.cluster.local` |
| StatefulSet Pod | `<pod>.<svc>.<ns>.svc.cluster.local` | `db-0.db-svc.app1-ns.svc.cluster.local` |
| Pod by IP (plugin) | `<hyphen-ip>.<ns>.pod.cluster.local` | `10-244-2-10.default.pod.cluster.local` |

### Cross-Namespace Resolution Rules

| Scenario | Works? | Query needed |
|----------|--------|-------------|
| Same-namespace short name | ✅ | `nginx-svc` |
| Cross-namespace short name | ❌ | Need: `nginx-svc.other-ns` |
| Cross-namespace with ns only | ✅ | `nginx-svc.other-ns` |
| Full FQDN anywhere | ✅ | `nginx-svc.other-ns.svc.cluster.local` |

### `pods` Plugin Mode — When to Use What

| Mode | Security | Use Case |
|------|----------|---------|
| `pods insecure` | ❌ Low | Dev/testing, kube-dns compat |
| `pods verified` | ✅ High | Production — validates against API server |
| `pods hostname` | ✅ Medium | StatefulSets, explicit hostname control |
| (disabled, default) | N/A | No pod-IP DNS; service-only resolution |

---

## 3. Internal Working ⚙️

### How CoreDNS Resolves a Name — Step by Step

**Scenario:** Pod in `default` namespace queries `nginx-svc`

```
1. Pod calls: getaddrinfo("nginx-svc")
        │
2. OS checks /etc/nsswitch.conf → "hosts: files dns"
        │
3. OS checks /etc/hosts → no match
        │
4. OS reads /etc/resolv.conf:
     nameserver: 10.96.0.10
     search: default.svc.cluster.local svc.cluster.local cluster.local
     options: ndots:5
        │
5. "nginx-svc" has 0 dots → LESS than ndots:5
   → Append first search domain first
   → Query CoreDNS: "nginx-svc.default.svc.cluster.local"
        │
6. CoreDNS receives query on port 53 (UDP)
        │
7. CoreDNS `kubernetes` plugin checks its in-memory cache/watch:
   → Looks for Service named "nginx-svc" in namespace "default"
   → FOUND: ClusterIP = 10.96.48.49
        │
8. CoreDNS returns A-record: nginx-svc.default.svc.cluster.local → 10.96.48.49
        │
9. Pod connects to 10.96.48.49 → kube-proxy routes to a backend pod
```

**Scenario:** Pod queries `google.com` (external)

```
1-5. Same as above, but "google.com" fails all search domain tries
        │
6. CoreDNS exhausts cluster.local lookups → no match
        │
7. CoreDNS `forward` plugin takes over
   → Sends query to upstream defined in CoreDNS node's /etc/resolv.conf
   → Upstream nameserver resolves google.com → real IP
        │
8. CoreDNS caches response (cache 30s)
        │
9. Returns IP to pod
```

### How CoreDNS Watches Kubernetes State

```
Kubernetes API Server
        │  (watch API — Services, Pods, Endpoints)
        ▼
CoreDNS `kubernetes` plugin (runs inside CoreDNS pod)
        │  Maintains in-memory index of:
        │  - All Services → ClusterIP / headless
        │  - All Pods (if pods plugin enabled) → IP
        │  - StatefulSet pods via headless svc → stable names
        ▼
DNS query arrives → O(1) in-memory lookup → response in microseconds
```

CoreDNS does **not** query etcd directly. It watches the API server over a long-lived HTTP/2 watch connection — the same mechanism used by controllers. This means DNS records are updated within milliseconds of a Service being created or deleted.

### How `/etc/resolv.conf` Gets Into a Pod

```
kubelet starts pod on node
        │
kubelet reads cluster DNS config (--cluster-dns flag)
        │
kubelet generates /etc/resolv.conf content:
  nameserver <kube-dns ClusterIP>
  search <namespace>.svc.cluster.local svc.cluster.local cluster.local
  options ndots:5
        │
kubelet injects file into pod's network namespace at creation time
        │
Pod sees /etc/resolv.conf on first read — already populated
```

The search domain format `<namespace>.svc.cluster.local` is why pods in different namespaces have different DNS search behavior — the namespace is baked in at pod creation.

---

## 4. Hands-On 🛠️

### Cluster Setup (KIND)

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
```

```bash
kind create cluster --config kind-cluster.yaml --name=cka-2025
```

### Inspect CoreDNS

```bash
# CoreDNS Deployment status
kubectl get deploy -n kube-system coredns

# CoreDNS Service (kube-dns)
kubectl get svc -n kube-system kube-dns

# CoreDNS pods and their labels
kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide

# Full Corefile config
kubectl get cm coredns -n kube-system -o yaml
```

### Set Up Test Services

```bash
# Create nginx service in default namespace
kubectl create deployment nginx-deploy --image=nginx
kubectl expose deployment nginx-deploy --name=nginx-svc --port=80

# Create service in a different namespace
kubectl create ns app1-ns
kubectl create deployment app1-deploy --image=nginx -n app1-ns
kubectl expose deployment app1-deploy --name=app1-svc --port=80 -n app1-ns

# Verify
kubectl get pods,svc
kubectl get pods,svc -n app1-ns
```

### DNS Testing with netshoot

```bash
# Launch debug pod in default namespace
kubectl run debug-pod -it --rm --image=nicolaka/netshoot -- bash

# Inside the pod:

# 1. Check DNS config
cat /etc/resolv.conf
# Expected: nameserver 10.96.0.10, search default.svc.cluster.local ...

cat /etc/hosts
# Shows pod IP, localhost, pod name

cat /etc/nsswitch.conf
# Shows: hosts: files dns

# 2. Resolve service in same namespace
nslookup nginx-svc
# Returns: nginx-svc.default.svc.cluster.local → ClusterIP

# 3. Resolve service in different namespace (from default ns)
nslookup app1-svc.app1-ns
# Works — resolves to app1-svc.app1-ns.svc.cluster.local

nslookup app1-svc
# FAILS — searches default.svc.cluster.local, not app1-ns

# 4. Resolve external DNS
nslookup kubernetes.io
# Works via CoreDNS forward plugin

# 5. Full FQDN lookup
nslookup nginx-svc.default.svc.cluster.local
dig nginx-svc.default.svc.cluster.local

# 6. Attempt pod name resolution (won't work — pods plugin disabled by default)
# Pod runs as "debug-pod" but:
nslookup debug-pod
# Returns: NXDOMAIN — no pod DNS record
```

### Cross-Namespace DNS Test

```bash
# Launch debug pod IN app1-ns
kubectl run debug-pod-app1 -n app1-ns -it --rm --image=nicolaka/netshoot -- bash

# Inside the pod — check its resolv.conf
cat /etc/resolv.conf
# search app1-ns.svc.cluster.local svc.cluster.local cluster.local ← ns changed!

# Short name works because namespace is in search domains
nslookup app1-svc
# Returns: app1-svc.app1-ns.svc.cluster.local → ClusterIP ✅

# Cross-namespace still needs namespace qualifier from here
nslookup nginx-svc.default
# Returns: nginx-svc.default.svc.cluster.local → ClusterIP ✅
```

### StatefulSet DNS Test

```bash
# Create headless service + StatefulSet
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: db-svc
  namespace: app1-ns
spec:
  clusterIP: None
  selector:
    app: db
  ports:
  - port: 3306
    targetPort: 3306
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db-sts
  namespace: app1-ns
spec:
  serviceName: db-svc
  replicas: 2
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "password123"
EOF

# Test StatefulSet pod DNS from debug pod in app1-ns
kubectl run dns-test -n app1-ns -it --rm --image=nicolaka/netshoot -- bash

# Inside pod:
nslookup db-sts-0.db-svc.app1-ns.svc.cluster.local
nslookup db-sts-1.db-svc.app1-ns.svc.cluster.local
# Each returns the individual pod IP — not the same ClusterIP
```

### Enable Pod DNS (pods verified mode)

```bash
# Edit CoreDNS configmap
kubectl edit cm coredns -n kube-system

# Change: pods insecure
# To:     pods verified
# (or add if not present)

# After saving, CoreDNS auto-reloads (reload plugin)
# Test pod IP DNS:
# Get pod IP
kubectl get pod debug-pod -o jsonpath='{.status.podIP}'
# e.g., 10.244.2.10

# From inside another pod:
nslookup 10-244-2-10.default.pod.cluster.local
# Should return: 10.244.2.10
```

### The Corefile (Production-Ready Reference)

```bash
# View current Corefile
kubectl get cm coredns -n kube-system -o jsonpath='{.data.Corefile}'
```

```
.:53 {
    errors
    health {
       lameduck 5s
    }
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
       pods insecure
       fallthrough in-addr.arpa ip6.arpa
       ttl 30
    }
    prometheus :9153
    forward . /etc/resolv.conf {
       max_concurrent 1000
    }
    cache 30 {
       disable success cluster.local
       disable denial cluster.local
    }
    loop
    reload
    loadbalance
}
```

---

## 5. Production Flow 🏗️

### CoreDNS High-Availability Architecture

```
                    ┌──────────────┐
                    │   Pod A      │
                    │ /etc/resolv  │
                    │ ns: 10.96.0.10│
                    └──────┬───────┘
                           │ DNS query (UDP 53)
                           ▼
                  ┌─────────────────┐
                  │  kube-dns Svc   │  (ClusterIP: 10.96.0.10)
                  │  ClusterIP      │
                  └────────┬────────┘
                           │ iptables/IPVS load balances
                    ┌──────┴──────┐
                    ▼             ▼
              ┌──────────┐  ┌──────────┐
              │ CoreDNS  │  │ CoreDNS  │  ← 2+ replicas
              │  Pod 1   │  │  Pod 2   │
              └────┬─────┘  └────┬─────┘
                   │              │
                   └──────┬───────┘
                          │ Watches API server
                          ▼
                  ┌────────────────┐
                  │  API Server    │
                  │  (Services,    │
                  │   Pods, NSes)  │
                  └────────────────┘
```

### Scaling CoreDNS for Large Clusters

For clusters with 500+ nodes or high DNS query rates:

```bash
# Scale CoreDNS replicas
kubectl scale deployment coredns -n kube-system --replicas=4

# Check current HPA (if configured)
kubectl get hpa -n kube-system

# Check DNS query rate via metrics (requires Prometheus)
# CoreDNS metrics endpoint
kubectl port-forward -n kube-system svc/kube-dns 9153:9153
curl localhost:9153/metrics | grep coredns_dns_requests_total
```

### Custom DNS for External Domains (stub zones)

```yaml
# Add to Corefile to forward specific domains to custom DNS
# kubectl edit cm coredns -n kube-system

.:53 {
    # ... existing config ...
    forward . /etc/resolv.conf
}

# Add a stub zone for internal corporate DNS
corp.internal:53 {
    errors
    cache 30
    forward . 10.10.0.53    # Your internal DNS server
}
```

### Caching Strategy

```
cluster.local queries → cache DISABLED (dynamic IPs change frequently)
external queries       → cache 30s (stable, safe to cache)
```

The `disable success cluster.local` and `disable denial cluster.local` lines in the Corefile prevent caching of Kubernetes-internal DNS — critical because pods and services are created/deleted rapidly and stale cache entries would cause connection failures.

### NetworkPolicy for DNS (Critical Production Requirement)

When applying restrictive NetworkPolicies, always allow DNS:

```yaml
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

---

## 6. Mistakes 💥

### Mistake 1: Forgetting That `/etc/hosts` Is Not Shared Between Containers

**Symptom:** Entry added to `/etc/hosts` in container A doesn't help container B in the same pod.

**Root Cause:** Each container has its own process namespace and filesystem view. `/etc/hosts` is mounted per-pod but the filesystem is per-container.

**Fix:** Use `hostAliases` in the Pod spec to inject custom `/etc/hosts` entries:

```yaml
spec:
  hostAliases:
  - ip: "1.2.3.4"
    hostnames:
    - "custom.internal"
```

---

### Mistake 2: Expecting Pod Name to Resolve via DNS

**Symptom:** `nslookup my-pod` returns NXDOMAIN even though the pod is running.

**Root Cause:** CoreDNS `pods` plugin is **disabled** by default. Pod names are not registered.

**Fix (if needed):** Enable `pods verified` in the Corefile. But in most production cases, **use Services**, not pod DNS.

---

### Mistake 3: Cross-Namespace Short Names

**Symptom:** `curl app1-svc` fails from a pod in `default` namespace, even though `app1-svc` exists.

**Root Cause:** The search domains are scoped to `default.svc.cluster.local`. `app1-svc` is in `app1-ns`, so `app1-svc.default.svc.cluster.local` doesn't exist.

**Fix:** Use `app1-svc.app1-ns` or the full FQDN.

---

### Mistake 4: `nslookup` Returns NXDOMAIN But `curl` Works

**Symptom:** `nslookup myservice` fails but `curl myservice` succeeds.

**Root Cause:** `/etc/hosts` has a static entry for `myservice`. `curl` follows `nsswitch.conf` (files → dns). `nslookup` skips `/etc/hosts` entirely.

**Lesson:** `nslookup`/`dig` are raw DNS tools. They don't respect `/etc/hosts`. Use `getent hosts <name>` to test the full resolution stack.

---

### Mistake 5: CoreDNS Pods Running But DNS Broken

**Symptom:** CoreDNS pods are `Running` and `Ready`, but DNS queries timeout.

**Root Cause (common):** NetworkPolicy is blocking UDP/TCP port 53 between pods and CoreDNS. Another common cause: loop detection triggered, CoreDNS restarting silently.

**Fix:**
```bash
# Check for DNS NetworkPolicy blockage
kubectl describe netpol -A | grep -A10 "Port"

# Check CoreDNS logs for "Loop" errors
kubectl logs -n kube-system -l k8s-app=kube-dns | grep -i loop

# Check if kube-dns Service has endpoints
kubectl get endpoints kube-dns -n kube-system
```

---

### Mistake 6: Modifying Corefile Without Using ConfigMap

**Symptom:** Changes to CoreDNS behavior are lost after pod restart.

**Root Cause:** CoreDNS pods mount the Corefile from the `coredns` ConfigMap. Editing the file inside the running container is ephemeral.

**Fix:** Always edit the ConfigMap:
```bash
kubectl edit cm coredns -n kube-system
# CoreDNS `reload` plugin picks up the change automatically
```

---

### Mistake 7: StatefulSet Pod DNS Not Resolving

**Symptom:** `nslookup db-sts-0.db-svc.app1-ns.svc.cluster.local` fails.

**Root Cause (3 possible):**
1. `serviceName` in StatefulSet doesn't match the headless Service name.
2. Headless Service doesn't have `clusterIP: None`.
3. Pod labels don't match the headless Service selector.

**Fix:**
```bash
kubectl describe statefulset db-sts -n app1-ns | grep ServiceName
kubectl describe svc db-svc -n app1-ns | grep -E "Type|ClusterIP|Selector"
kubectl get pods -n app1-ns -l app=db   # Verify label match
```

---

## 7. Interview Answers 🎤

**Q: What is CoreDNS and what role does it play in Kubernetes?**

> "CoreDNS is the default DNS server for Kubernetes clusters. It runs as a Deployment — typically two replicas for high availability — in the `kube-system` namespace, and it's exposed as a ClusterIP Service called `kube-dns`. Every pod in the cluster has its `/etc/resolv.conf` automatically configured to point to this ClusterIP as its nameserver. CoreDNS uses a plugin-based architecture; the `kubernetes` plugin watches the API server for Services and Pods, maintains an in-memory DNS table, and resolves short names like `nginx-svc` to their ClusterIPs. The `forward` plugin handles external queries by forwarding them to upstream resolvers. Without CoreDNS, pods would have to communicate using raw IP addresses, which breaks every time a Service is recreated."

---

**Q: What is the FQDN format for a Service and a StatefulSet pod in Kubernetes?**

> "For a regular Service, the FQDN follows the pattern `<service-name>.<namespace>.svc.cluster.local`. So a Service named `nginx-svc` in the `default` namespace resolves to `nginx-svc.default.svc.cluster.local`, and CoreDNS returns the Service's ClusterIP. For StatefulSet pods behind a headless Service — one with `clusterIP: None` — the format is `<pod-name>.<service-name>.<namespace>.svc.cluster.local`. So the first pod of a StatefulSet named `db-sts` using headless service `db-svc` in `app1-ns` is addressable as `db-sts-0.db-svc.app1-ns.svc.cluster.local`. This is critical because it resolves to the individual pod IP, not a virtual service IP, giving you stable, predictable addressing for stateful workloads."

---

**Q: Why can't you resolve a service from another namespace using just the service name?**

> "Every pod gets a `/etc/resolv.conf` with search domains scoped to its own namespace — something like `default.svc.cluster.local svc.cluster.local cluster.local`. When you use a short name like `app1-svc`, the resolver appends these domains in sequence. If `app1-svc` lives in `app1-ns`, not `default`, none of those search domain combinations will match, and you get NXDOMAIN. To resolve across namespaces, you need at minimum `app1-svc.app1-ns`, which the search domain turns into `app1-svc.app1-ns.svc.cluster.local`. Or you can use the full FQDN directly. This is why code that works fine within one namespace can silently break when the service is moved to another."

---

**Q: What is `ndots:5` in `/etc/resolv.conf` and why does it matter?**

> "The `ndots:5` option tells the resolver how to handle names that might be relative versus fully qualified. If a name has fewer than 5 dots, the resolver tries appending the search domains first before attempting it as an absolute name. If it has 5 or more dots, it's treated as a FQDN first. In Kubernetes, most service names — like `nginx-svc` or `nginx-svc.default.svc.cluster.local` — have fewer than 5 dots, so search domains are tried first. The downside is that external names like `api.mycompany.com` — 2 dots — also trigger search domain appending, causing multiple unnecessary DNS round trips before falling back to the actual name. You can eliminate this by always using trailing-dot FQDNs or pinning the exact FQDN in your configuration."

---

**Q: How does CoreDNS handle pod DNS records, and what modes are available?**

> "By default, CoreDNS does not create DNS records for individual pods. Only Services get A-records. However, CoreDNS has a `pods` directive within the `kubernetes` plugin that enables pod-level resolution. There are three modes. `pods insecure` enables IP-based lookups — like `10-244-2-10.default.pod.cluster.local` — without verifying with the API server. It's fast but can return stale or spoofed results, so it's mainly kept for backward compatibility with the old kube-dns system. `pods verified` only returns a result if the API server confirms there's an active pod with that IP in that namespace — this is the production-safe option. `pods hostname` resolves pods based on their hostname field, which is predictable for StatefulSets. In practice, for most production workloads, you don't need pod DNS — Services are the correct abstraction."

---

**Q: What's the difference between `/etc/hosts` and `/etc/resolv.conf`?**

> "`/etc/hosts` is a static local file containing name-to-IP mappings. It's checked first — before DNS — because `nsswitch.conf` defines the lookup order as `files dns`. Any entry in `/etc/hosts` short-circuits the DNS query entirely. In Kubernetes, this file includes the pod's own IP and hostname by default, and you can add custom entries using `hostAliases` in the Pod spec. `/etc/resolv.conf`, on the other hand, tells the system which DNS nameserver to use and which search domains to try — it doesn't contain any records itself. One important nuance: `nslookup` and `dig` bypass `/etc/hosts` completely and query the nameserver directly. So if something resolves with `ping` but not `nslookup`, it's almost certainly coming from `/etc/hosts`."

---

**Q: How do you troubleshoot DNS failures in a Kubernetes cluster?**

> "I follow a layered approach. First, I verify CoreDNS pods are running and healthy with `kubectl get pods -n kube-system -l k8s-app=kube-dns`. Then I check CoreDNS logs for errors — loop detection, forward failures, or crashed plugins. Next, I launch a debug pod with `nicolaka/netshoot` and test from inside: `nslookup kubernetes.default` for internal, and `nslookup google.com` for external. If internal works but external doesn't, the `forward` plugin or upstream DNS is the issue. If neither works, CoreDNS itself is broken or there's a NetworkPolicy blocking port 53. I then check `/etc/resolv.conf` inside the pod to confirm it points to the right nameserver. I also verify the `kube-dns` Service has endpoints with `kubectl get endpoints kube-dns -n kube-system`. Finally, I inspect the Corefile ConfigMap for any configuration mistakes. The most common causes I see are: NetworkPolicy blocking UDP 53, wrong namespace in a service query, and stale ConfigMap changes that weren't picked up."

---

## 8. Debugging 🔍

### Symptom → Diagnosis Path

```
DNS resolution failing
         │
         ├─ 1. Is CoreDNS running?
         │     kubectl get pods -n kube-system -l k8s-app=kube-dns
         │     → Not running: restart, check resource limits
         │     → CrashLoopBackOff: check logs (loop? config error?)
         │
         ├─ 2. Does kube-dns Service have endpoints?
         │     kubectl get endpoints kube-dns -n kube-system
         │     → No endpoints: CoreDNS pods not matching service selector
         │
         ├─ 3. Can the pod reach CoreDNS IP at all?
         │     kubectl exec -it <pod> -- nc -vz 10.96.0.10 53
         │     → Timeout: NetworkPolicy blocking port 53
         │     → Connection refused: CoreDNS not listening
         │
         ├─ 4. Test internal DNS
         │     kubectl exec -it <pod> -- nslookup kubernetes.default
         │     → NXDOMAIN: kubernetes plugin misconfigured in Corefile
         │     → Timeout: see step 3
         │
         ├─ 5. Test external DNS
         │     kubectl exec -it <pod> -- nslookup google.com
         │     → Fails (internal works): forward plugin misconfigured
         │     → Check CoreDNS node's /etc/resolv.conf for valid upstreams
         │
         ├─ 6. Check /etc/resolv.conf inside pod
         │     kubectl exec -it <pod> -- cat /etc/resolv.conf
         │     → Wrong nameserver: kubelet misconfigured (--cluster-dns)
         │     → Wrong search domains: namespace issue
         │
         ├─ 7. Cross-namespace resolution failure?
         │     Try: nslookup svc-name.other-ns (not just svc-name)
         │     → Short name in wrong ns = expected failure
         │
         └─ 8. StatefulSet pod DNS failure?
               Check serviceName matches headless svc name
               Check headless svc has clusterIP: None
               Check pod labels match svc selector
```

### Quick Diagnostic Commands

```bash
# CoreDNS health check
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl get endpoints kube-dns -n kube-system
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=100

# Check for loop errors
kubectl logs -n kube-system -l k8s-app=kube-dns | grep -i "loop\|error\|SERVFAIL"

# Launch netshoot debug pod
kubectl run dns-debug --rm -it --image=nicolaka/netshoot -- bash

# Inside debug pod: full DNS test suite
cat /etc/resolv.conf
nslookup kubernetes.default           # Test internal
nslookup google.com                   # Test external forward
nslookup nginx-svc.default            # Test service
nslookup nginx-svc.default.svc.cluster.local  # Test FQDN
dig nginx-svc.default.svc.cluster.local +short
getent hosts nginx-svc               # Tests FULL resolution stack (hosts + DNS)

# Test port 53 reachability
nc -vzu 10.96.0.10 53               # UDP
nc -vz 10.96.0.10 53                # TCP

# Check Corefile
kubectl get cm coredns -n kube-system -o jsonpath='{.data.Corefile}'

# Validate StatefulSet DNS setup
kubectl describe statefulset <name> -n <ns> | grep ServiceName
kubectl get svc -n <ns> | grep None    # Headless service has None ClusterIP
kubectl get pods -n <ns> --show-labels

# Check CoreDNS metrics (if Prometheus available)
kubectl port-forward -n kube-system svc/kube-dns 9153:9153
curl -s localhost:9153/metrics | grep -E "dns_requests|dns_responses|forward"
```

### Common Failures and Fixes

| Symptom | Root Cause | Fix |
|---------|-----------|-----|
| `nslookup svc` → NXDOMAIN (same ns) | Service doesn't exist or wrong name | `kubectl get svc -n <ns>` |
| `nslookup svc` → NXDOMAIN (diff ns) | Missing namespace qualifier | Use `svc-name.namespace` |
| External DNS fails, internal works | `forward` plugin misconfigured | Check Corefile `forward` directive |
| DNS times out entirely | NetworkPolicy blocking port 53 | Add DNS egress rule (see Section 5) |
| `nslookup` fails but `ping`/`curl` works | Answer in `/etc/hosts` | Use `getent hosts` to see full resolution |
| Pod IP DNS fails | `pods` plugin disabled | Enable `pods verified` in Corefile |
| StatefulSet pod DNS fails | `serviceName` mismatch | Match StatefulSet `serviceName` to headless svc name |
| CoreDNS CrashLoopBackOff | Loop detection triggered | Add `resolvConf /etc/resolv.conf` or fix upstream DNS |

---

## 9. Kill Switch ⚡

**10-second recall — the absolute minimum:**

```
1. CoreDNS = Deployment in kube-system, exposed as kube-dns ClusterIP (10.96.0.10)
2. Service FQDN: <name>.<ns>.svc.cluster.local → ClusterIP
3. StatefulSet pod FQDN: <pod>.<svc>.<ns>.svc.cluster.local → Pod IP (headless svc)
4. Pod DNS = NOT created by default. Needs `pods` plugin in Corefile.
5. /etc/resolv.conf injected by kubelet: nameserver + search domains + ndots:5
6. Search domains scoped to pod's namespace → cross-ns needs namespace qualifier
7. nslookup/dig bypass /etc/hosts. getent tests full stack.
8. Cache disabled for cluster.local (dynamic). Enabled for external (30s).
9. CoreDNS Corefile lives in ConfigMap `coredns` — edit there, reload plugin auto-applies.
10. DNS broken? → Check: CoreDNS running → endpoints exist → port 53 reachable → resolv.conf correct
```

---

## 10. Appendix — Quick Reference 📋

### FQDN Cheatsheet

```
Service:          <svc>.<ns>.svc.cluster.local
StatefulSet Pod:  <pod-name>.<headless-svc>.<ns>.svc.cluster.local
Pod by IP:        <a-b-c-d>.<ns>.pod.cluster.local   (requires pods plugin)
```

### /etc/resolv.conf Template (Generated by kubelet)

```
nameserver 10.96.0.10
search <namespace>.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

### CoreDNS Plugin Quick Reference

| Plugin | What it Does |
|--------|-------------|
| `kubernetes` | Resolves Service and Pod names from K8s API |
| `pods insecure/verified/hostname` | Controls pod IP DNS behavior |
| `forward` | Forwards unresolved queries to upstream DNS |
| `cache` | Caches external DNS responses |
| `errors` | Logs DNS errors |
| `health` | `/health` liveness endpoint |
| `ready` | `/ready` readiness endpoint |
| `prometheus` | Metrics on port 9153 |
| `loop` | Detects/prevents DNS loops |
| `reload` | Auto-reloads Corefile from ConfigMap |
| `loadbalance` | Round-robins multiple A records |

### Resolution Order Inside a Pod

```
1. /etc/hosts         (files in nsswitch.conf)
2. CoreDNS via /etc/resolv.conf nameserver
   a. Append search domains (if name has < ndots dots)
   b. Try as absolute FQDN if search domains fail
   c. Forward to upstream if not found in cluster.local
```

### Essential Commands Cheatsheet

```bash
# CoreDNS status
kubectl get deploy,pods,svc -n kube-system | grep dns
kubectl get endpoints kube-dns -n kube-system
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50

# Corefile inspection
kubectl get cm coredns -n kube-system -o yaml
kubectl get cm coredns -n kube-system -o jsonpath='{.data.Corefile}'

# Edit Corefile (changes auto-applied by reload plugin)
kubectl edit cm coredns -n kube-system

# Scale CoreDNS
kubectl scale deploy coredns -n kube-system --replicas=4

# Debug pod (the gold standard tool)
kubectl run dns-debug --rm -it --image=nicolaka/netshoot -- bash

# Inside debug pod
nslookup <svc>                                    # Short name (same ns)
nslookup <svc>.<ns>                               # Cross-namespace
nslookup <svc>.<ns>.svc.cluster.local             # Full FQDN
dig <svc>.<ns>.svc.cluster.local +short           # Clean IP output
dig <svc>.<ns>.svc.cluster.local ANY              # All record types
getent hosts <svc>                                # Full resolution (hosts + DNS)
cat /etc/resolv.conf                              # Check DNS config
cat /etc/hosts                                    # Check static overrides
nc -vzu 10.96.0.10 53                            # Test UDP 53 connectivity

# Service DNS records
kubectl get svc -A -o custom-columns=\
'NS:.metadata.namespace,NAME:.metadata.name,TYPE:.spec.type,IP:.spec.clusterIP'
```

### DNS Name Collision Debug Quick-Guide

```
Symptom: resolve fails with NXDOMAIN
Check sequence:
  1. kubectl get svc <name> -n <namespace>    → service exists?
  2. nslookup <name>.<namespace>              → try with namespace
  3. kubectl get pods -l app=<label> -n <ns>  → pods matching selector?
  4. kubectl get endpoints <svc> -n <ns>      → endpoints present?
  5. cat /etc/resolv.conf (inside pod)        → correct nameserver/search?
```