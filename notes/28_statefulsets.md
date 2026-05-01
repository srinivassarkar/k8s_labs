# Kubernetes StatefulSets 

## 0. First Principles

> *The mental model that never changes — anchor every decision here.*

**The Fundamental Contract of a StatefulSet:**

A StatefulSet makes a promise to each pod: *"You will always have the same name, the same DNS, and the same disk — across every restart, reschedule, and upgrade, forever."*

This promise is the entire reason StatefulSets exist. Everything else — ordered startup, headless services, volumeClaimTemplates — is infrastructure built to keep that promise.

```
┌─────────────────────────────────────────────────────────────────┐
│  THE STATEFULSET TRIAD                                          │
│                                                                 │
│   IDENTITY          DNS               STORAGE                   │
│   ────────          ───               ───────                   │
│   mysql-0           mysql-0.mysql     mysql-data-mysql-0        │
│   mysql-1     +     .mysql-ha    +    mysql-data-mysql-1        │
│   mysql-2           .svc.cluster      mysql-data-mysql-2        │
│                     .local                                      │
│                                                                 │
│   ☑ Stable across restarts   ☑ Survives node loss               │
│   ☑ Predictable ordinals     ☑ AZ-scoped reattachment           │
└─────────────────────────────────────────────────────────────────┘
```

**Why databases break without this triad:**

A MySQL primary stores replication configs using the replica's *hostname*. If `mysql-1` restarts as `mysql-4` (as it would in a Deployment), the primary sees a new, unauthorized node. Replication breaks silently. Data diverges. You find out on Friday night.

**The Three Immutable Laws:**

1. **Identity is sticky** — same ordinal, same PVC, same DNS record, every time
2. **Order is enforced** — pod `N` does not start until pod `N-1` is `Running` and `Ready`
3. **Volumes outlive pods** — PVCs are never deleted by the StatefulSet controller, only by you

---

## 1. Reality Constraints

> *What Kubernetes actually does and doesn't do — the guardrails you'll hit in production.*

### What StatefulSets Guarantee

| Guarantee | Mechanism | Caveat |
|---|---|---|
| Stable pod name | Ordinal suffix: `<sts-name>-<N>` | Name changes only if StatefulSet is renamed |
| Stable DNS per pod | Headless Service + CoreDNS | **You must create the headless service manually** |
| Ordered creation | `OrderedReady` policy (default) | Pod `N+1` waits for `N` to be `Ready` |
| Ordered deletion | Reverse ordinal (highest first) | Scale-down: `mysql-2` deleted before `mysql-1` |
| Volume per pod | `volumeClaimTemplates` | PVCs are **not** deleted when StatefulSet is deleted |
| Volume reattachment | PVC binding is sticky to pod name | Only works within the **same AZ** for block storage |

### What StatefulSets Do NOT Do

| Assumption | Reality |
|---|---|
| StatefulSet configures replication | ❌ It only provisions pods with stable identity — you or an Operator configures MySQL replication |
| Volume moves across AZs | ❌ EBS volumes are AZ-scoped — pod reschedule stays within original AZ |
| Headless service is auto-created | ❌ Must be created manually; `serviceName` in spec must match exactly |
| Data is safe on StatefulSet delete | ✅ PVCs persist — but EBS charges continue. You must manually delete PVCs to clean up |
| `Parallel` policy is safe for DBs | ❌ `Parallel` starts all pods simultaneously — unsafe for primary/replica ordering |

### EBS + StatefulSet AZ Binding — The Hard Wall

```
AZ us-east-2a          AZ us-east-2b          AZ us-east-2c
─────────────          ─────────────          ─────────────
  mysql-0                mysql-1                mysql-2
  EBS vol-A              EBS vol-B              EBS vol-C

If node in 2a dies:
  mysql-0 reschedules → MUST land in 2a (another node)
  If no node in 2a → pod stays Pending (NOT rescheduled to 2b)
  vol-A cannot cross AZ boundaries
```

This is why `WaitForFirstConsumer` on the StorageClass is **mandatory** — it defers volume creation until the pod is scheduled, binding the volume to the pod's AZ, not a random one.

---

## 2. Decision Logic

> *When to use what — hard rules, no ambiguity.*

### StatefulSet vs. Deployment Decision Tree

```
Does your workload need...
        │
        ├─ Persistent, pod-specific storage?
        │         YES → StatefulSet
        │
        ├─ Stable network identity (pod-level DNS)?
        │         YES → StatefulSet
        │
        ├─ Ordered startup/shutdown?
        │         YES → StatefulSet
        │
        ├─ Peer discovery by pod name (Kafka, MySQL, ES)?
        │         YES → StatefulSet
        │
        └─ None of the above → Deployment
```

### Anti-Affinity: Hard vs. Soft Rule

| Rule Type | Field | Behavior | When to Use |
|---|---|---|---|
| **Hard** | `requiredDuringSchedulingIgnoredDuringExecution` | Pod stays Pending if constraint cannot be met | Production HA — never co-locate DB replicas |
| **Soft** | `preferredDuringSchedulingIgnoredDuringExecution` | Best-effort spread, will still schedule | Dev/cost environments, more replicas than AZs |

**Rule of thumb:** If `replicas <= number of AZs`, use Hard. If `replicas > AZs`, use Soft or you will have Pending pods.

### Pod Management Policy

| Policy | Startup | Termination | Use Case |
|---|---|---|---|
| `OrderedReady` (default) | Sequential (0 → N) | Reverse (N → 0) | Databases, Kafka, Zookeeper |
| `Parallel` | Simultaneous | Simultaneous | Stateless-ish apps that happen to need PVCs |

### Access Mode Selection

| Mode | Meaning | EBS Support | Use Case |
|---|---|---|---|
| `ReadWriteOnce` (RWO) | One node at a time | ✅ | Single-node DB, general use |
| `ReadWriteOncePod` (RWOP) | One **pod** at a time (stricter) | ✅ | StatefulSet DBs — preferred |
| `ReadOnlyMany` (ROX) | Many pods, read-only | ❌ | Shared config, seed data |
| `ReadWriteMany` (RWX) | Many pods, read-write | ❌ | Shared logs — use EFS instead |

---

## 3. Internal Working

> *How it actually happens under the hood — step by step.*

### StatefulSet Controller Loop

```
1. You apply StatefulSet YAML
        │
        ▼
2. StatefulSet controller reads spec.replicas
        │
        ▼
3. For each ordinal (0, 1, 2...):
   a. Does PVC "mysql-data-mysql-N" exist?
      NO  → Create PVC from volumeClaimTemplate
      YES → Use existing PVC (key: data persistence on restart)
        │
   b. Create Pod "mysql-N" with PVC binding
        │
   c. Wait for Pod "mysql-N" to be Running + Ready
        │
   d. Move to ordinal N+1
        │
        ▼
4. StatefulSet reaches desired state
```

### Volume Provisioning Flow (WaitForFirstConsumer)

```
kubectl apply -f sts.yaml
        │
        ▼
PVC created (status: Pending — no AZ binding yet)
        │
        ▼
Scheduler places Pod "mysql-0" on node in us-east-2a
        │
        ▼
CSI provisioner gets AZ hint from node topology
        │
        ▼
EBS volume created in us-east-2a
        │
        ▼
PVC bound to volume (status: Bound)
        │
        ▼
Pod mounts volume and starts
```

### Pod Deletion + Reattachment Flow

```
kubectl delete pod mysql-1
        │
        ▼
StatefulSet controller detects desired ≠ actual
        │
        ▼
New pod "mysql-1" created (SAME name, guaranteed)
        │
        ▼
Scheduler looks for node in us-east-2b (AZ of original EBS volume)
        │
        ▼
EBS volume detaches from old node
        ▼
EBS volume attaches to new node (takes 10–30s typically)
        │
        ▼
Pod enters ContainerCreating → Running
        │
        ▼
Data is intact — PVC was never deleted
```

### DNS Resolution Path

```
App queries: mysql-0.mysql.mysql-ha.svc.cluster.local
        │
        ▼
CoreDNS lookup: mysql.mysql-ha.svc.cluster.local (headless)
        │
        ▼
Headless service returns A record for pod IP directly
(No VIP, no kube-proxy, no load balancing)
        │
        ▼
Connection goes directly to mysql-0 pod IP
```

**Key:** Without `clusterIP: None`, CoreDNS returns the service VIP and load-balances. The pod-level DNS record `mysql-0.mysql...` only exists when the service is headless.

---

## 4. Hands-On

> *Production-quality YAML + commands. Nothing simplified.*

### StorageClass — EBS gp3, WaitForFirstConsumer

```yaml
# 01-sc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc-gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
volumeBindingMode: WaitForFirstConsumer   # CRITICAL: AZ binding deferred to pod scheduling
reclaimPolicy: Delete                     # Deletes EBS vol when PVC is deleted
allowVolumeExpansion: true
```

### Namespace

```bash
kubectl create namespace mysql-ha
kubectl config set-context --current --namespace=mysql-ha
```

### Headless Service

```yaml
# 02-mysql-hs.yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql          # Must match StatefulSet spec.serviceName exactly
  namespace: mysql-ha
spec:
  clusterIP: None      # THIS makes it headless — no VIP, no load balancing
  selector:
    app: mysql
  ports:
    - port: 3306
      targetPort: 3306
```

### StatefulSet — Production-Aligned

```yaml
# 03-mysql-sts.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: mysql-ha
spec:
  serviceName: mysql                  # Must match headless service name
  replicas: 3
  podManagementPolicy: OrderedReady   # Sequential start — critical for primary/replica ordering
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      terminationGracePeriodSeconds: 10   # Allow MySQL to flush buffers before SIGKILL
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:   # Hard rule: one pod per AZ
            - labelSelector:
                matchLabels:
                  app: mysql
              topologyKey: topology.kubernetes.io/zone
      containers:
        - name: mysql
          image: mysql:8.0
          ports:
            - containerPort: 3306
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:           # Use a Secret in production, never plain value
                  name: mysql-secret
                  key: root-password
          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql
          readinessProbe:               # Required for OrderedReady — pod N must be Ready before N+1 starts
            exec:
              command:
                - bash
                - -c
                - "mysqladmin ping -u root -p${MYSQL_ROOT_PASSWORD}"
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          livenessProbe:
            exec:
              command:
                - bash
                - -c
                - "mysqladmin ping -u root -p${MYSQL_ROOT_PASSWORD}"
            initialDelaySeconds: 60
            periodSeconds: 15
  volumeClaimTemplates:
    - metadata:
        name: mysql-data               # PVC name = mysql-data-mysql-0, mysql-data-mysql-1, etc.
      spec:
        accessModes: ["ReadWriteOncePod"]   # Strictest isolation — one pod, one volume
        storageClassName: ebs-sc-gp3
        resources:
          requests:
            storage: 5Gi
```

### Secret (Referenced Above)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: mysql-ha
type: Opaque
stringData:
  root-password: "your-secure-password-here"
```

### EBS CSI Driver Installation

```bash
# Install the CSI driver (required for dynamic EBS provisioning)
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.44"

# Verify daemonset (runs on every node — handles volume attach/mount)
kubectl get daemonset ebs-csi-node -n kube-system

# Verify controller (central provisioning brain)
kubectl get deployment ebs-csi-controller -n kube-system
```

### Key Operational Commands

```bash
# Apply everything in order
kubectl apply -f 01-sc.yaml
kubectl apply -f 02-mysql-hs.yaml
kubectl apply -f 03-mysql-sts.yaml

# Watch ordered pod startup (you'll see 0 → 1 → 2 in sequence)
kubectl get pods -w

# Verify each pod landed in a different AZ
kubectl get pods -o wide
kubectl get nodes --show-labels | grep topology.kubernetes.io/zone

# Verify PVCs created per pod
kubectl get pvc
# Expected: mysql-data-mysql-0, mysql-data-mysql-1, mysql-data-mysql-2

# Verify PVs and their AZ binding
kubectl get pv -o custom-columns='NAME:.metadata.name,AZ:.metadata.labels.topology\.kubernetes\.io/zone,CAPACITY:.spec.capacity.storage'

# Verify DNS resolution from inside a pod
kubectl exec -it mysql-0 -- bash -c "nslookup mysql-0.mysql.mysql-ha.svc.cluster.local"

# Full DNS format
# <pod-name>.<service-name>.<namespace>.svc.cluster.local
```

---

## 5. Production Flow

> *Real-world architecture and design patterns.*

### Multi-AZ MySQL HA Architecture

```
                         KUBERNETES CLUSTER (us-east-2)
    ┌──────────────────────────────────────────────────────────────────┐
    │                                                                  │
    │   AZ: us-east-2a        AZ: us-east-2b        AZ: us-east-2c   │
    │   ┌────────────┐        ┌────────────┐        ┌────────────┐   │
    │   │  mysql-0   │        │  mysql-1   │        │  mysql-2   │   │
    │   │ (PRIMARY)  │───────▶│ (REPLICA)  │        │ (REPLICA)  │   │
    │   │            │        │            │◀───────│            │   │
    │   └─────┬──────┘        └─────┬──────┘        └─────┬──────┘   │
    │         │                     │                     │          │
    │   ┌─────▼──────┐        ┌─────▼──────┐        ┌─────▼──────┐  │
    │   │  EBS vol-A │        │  EBS vol-B │        │  EBS vol-C │  │
    │   │    5Gi     │        │    5Gi     │        │    5Gi     │  │
    │   └────────────┘        └────────────┘        └────────────┘  │
    │                                                                 │
    │   Headless Service "mysql" → pod-level DNS, no load balancing  │
    │   Pod Anti-Affinity → hard rule, one pod per AZ                │
    └─────────────────────────────────────────────────────────────────┘

    DNS Records (via headless service + CoreDNS):
    mysql-0.mysql.mysql-ha.svc.cluster.local → 10.0.1.4   (PRIMARY)
    mysql-1.mysql.mysql-ha.svc.cluster.local → 10.0.2.7   (REPLICA)
    mysql-2.mysql.mysql-ha.svc.cluster.local → 10.0.3.12  (REPLICA)
```

### Replication Topology (DevOps provisions pods; DBA configures replication)

```
Initial Clone (chained to reduce primary load):
  mysql-0 (primary, 5GB) ──clone──▶ mysql-1 ──clone──▶ mysql-2

Continuous Replication (direct — lower lag, fewer failure points):
  mysql-0 (binlog) ──stream──▶ mysql-1
  mysql-0 (binlog) ──stream──▶ mysql-2
```

### Responsibility Boundary

| Layer | Owner | Scope |
|---|---|---|
| StorageClass, StatefulSet YAML | DevOps | Pod identity, PVC lifecycle, ordering |
| Headless Service, RBAC | DevOps | Network identity, access control |
| Liveness/Readiness probes | DevOps | Health signals for Kubernetes |
| Replication topology config | DBA | MySQL-level: server-id, binlog, replica setup |
| Backup/restore strategy | DBA + DevOps | CronJob (DevOps), mysqldump/xtrabackup (DBA) |
| Failover/promotion logic | DBA | ProxySQL, Orchestrator, or Operator |
| Operators (e.g., Percona, Vitess) | Both | Automate DBA layer through Kubernetes CRDs |

### PVC Lifecycle — What Survives What

```
kubectl delete pod mysql-1       → PVC survives ✅ (pod recreated, PVC reattached)
kubectl delete statefulset mysql → PVC survives ✅ (intentional — protect data)
kubectl delete pvc mysql-data-mysql-1 → Data gone ❌ (reclaimPolicy: Delete)

Full cleanup order:
  1. kubectl delete statefulset mysql
  2. kubectl delete pvc -l app=mysql
  (EBS volumes deleted by reclaimPolicy: Delete)
```

---

## 6. Mistakes

> *What actually breaks in real systems — root cause + fix.*

### Mistake 1: Missing Headless Service (or Wrong `serviceName`)

**Symptom:** Pods start, but `mysql-0.mysql.mysql-ha.svc.cluster.local` doesn't resolve. Replication fails on DNS lookup.

**Root Cause:** `spec.serviceName: mysql` in StatefulSet must match the name of a headless service (`clusterIP: None`) in the same namespace. If the service doesn't exist or has the wrong name, CoreDNS never creates pod A records.

**Fix:**
```bash
kubectl get svc -n mysql-ha   # Verify service exists
kubectl describe svc mysql -n mysql-ha   # Confirm clusterIP is None
```

---

### Mistake 2: Pod Stuck in Pending After Node Loss

**Symptom:** Node in `us-east-2b` dies. `mysql-1` is rescheduled but stays `Pending` indefinitely.

**Root Cause:** EBS volume is in `us-east-2b`. No other nodes exist in `us-east-2b`. Pod cannot be scheduled because it needs to be in the AZ where the volume lives.

**Fix:** Add a node in `us-east-2b` (auto-scaler should handle this if configured). This is **expected behavior** — the system is protecting your data, not malfunctioning.

```bash
kubectl describe pod mysql-1   # Look for: "node(s) had no available volume zone"
kubectl get nodes --show-labels | grep zone   # Check available nodes per AZ
```

---

### Mistake 3: `terminationGracePeriodSeconds` Too Short

**Symptom:** Intermittent data corruption after pod restarts, especially under write load.

**Root Cause:** Default is 30s but was reduced "for faster restarts." MySQL needs time to flush the InnoDB buffer pool and write all pending transactions to disk.

**Fix:** Set `terminationGracePeriodSeconds: 60` or higher for write-heavy MySQL. Monitor actual shutdown time via `kubectl logs` before tuning.

---

### Mistake 4: Using `Parallel` PodManagementPolicy for Databases

**Symptom:** Replicas start before primary is ready. `mysql-1` tries to connect to `mysql-0` for initial clone, but `mysql-0` hasn't initialized yet. Replication never establishes.

**Root Cause:** `Parallel` policy ignores `Ready` state — all pods start simultaneously.

**Fix:** Always use `OrderedReady` for databases. Only use `Parallel` for stateless workloads that happen to need PVCs (e.g., distributed caches in dev).

---

### Mistake 5: `ReadWriteOnce` Instead of `ReadWriteOncePod`

**Symptom:** During a rolling update, two pods briefly mount the same volume. Data corruption.

**Root Cause:** `RWO` allows multiple pods on the same **node** to mount the same volume. `RWOP` (ReadWriteOncePod) restricts to exactly one pod, globally.

**Fix:** Use `ReadWriteOncePod` for all StatefulSet database volumes on Kubernetes 1.22+.

---

### Mistake 6: Storing Passwords in Plain `env.value`

**Symptom:** Password visible in `kubectl describe pod` output, stored in etcd unencrypted.

**Root Cause:** `env.value: "password123"` exposes secrets in pod spec.

**Fix:**
```yaml
env:
  - name: MYSQL_ROOT_PASSWORD
    valueFrom:
      secretKeyRef:
        name: mysql-secret
        key: root-password
```

---

### Mistake 7: Forgetting `WaitForFirstConsumer` on StorageClass

**Symptom:** Pod scheduled in `us-east-2a`, but PVC was provisioned in `us-east-2b`. Pod stays `Pending` forever with volume affinity conflict.

**Root Cause:** Default `Immediate` binding mode provisions the volume before the pod is scheduled, potentially in the wrong AZ.

**Fix:** Always use `volumeBindingMode: WaitForFirstConsumer` on any StorageClass used with StatefulSets in multi-AZ clusters.

---

## 7. Interview Answers

> *Compressed, verbatim-ready answers. Speak these.*

---

**Q: What is a StatefulSet and why do we need it?**

> "A StatefulSet is a Kubernetes workload controller designed specifically for applications that require stable identity, persistent storage, and ordered startup and shutdown. The core problem it solves is this: databases like MySQL rely on fixed hostnames for replication configuration. If you run MySQL in a Deployment, pods get random names on restart and the primary no longer recognizes the replica — replication breaks silently. StatefulSets solve this by giving each pod a stable ordinal name like `mysql-0` and `mysql-1`, a stable DNS record via a headless service, and a dedicated PVC that reattaches to the same pod name on every restart. Without these three guarantees, you simply cannot run a production database in Kubernetes safely."

---

**Q: What is a headless service and why does a StatefulSet need one?**

> "A headless service is a Kubernetes Service with `clusterIP: None`. A regular service returns a virtual IP and load-balances traffic across all matching pods — which breaks stateful apps that need to target a specific replica. With a headless service, CoreDNS creates individual A records for each pod in the format `<pod-name>.<service-name>.<namespace>.svc.cluster.local`. This is critical because replication configs in MySQL or peer discovery in Kafka need to address specific pods by name — not a random load-balanced endpoint. The StatefulSet's `spec.serviceName` field must match the headless service name exactly, otherwise pod-level DNS records are never created."

---

**Q: What happens to PVCs when a StatefulSet is deleted?**

> "PVCs are intentionally retained when a StatefulSet is deleted. This is a deliberate design choice — StatefulSets are meant to preserve identity and data across controller lifecycle events. If you delete and recreate a StatefulSet with the same name and the same `volumeClaimTemplate`, the new pods automatically rebind to the existing PVCs. `mysql-0` gets `mysql-data-mysql-0`, `mysql-1` gets `mysql-data-mysql-1`, and so on — no data loss. If you truly want to clean up, you must first delete the StatefulSet, then manually delete the PVCs. The underlying EBS volumes are then deleted by the `reclaimPolicy: Delete` on the StorageClass."

---

**Q: What is `WaitForFirstConsumer` and why is it critical in a multi-AZ setup?**

> "WaitForFirstConsumer is a `volumeBindingMode` setting on the StorageClass that defers volume provisioning until a pod is actually scheduled onto a node. In a multi-AZ cluster, EBS volumes are AZ-scoped — once provisioned in `us-east-2a`, they can never move. Without this setting, the PVC might be provisioned in `us-east-2b` before the scheduler even assigns the pod. If pod anti-affinity forces the pod into `us-east-2a`, you get a permanent conflict: the pod can't run in 2a because its volume is in 2b. With `WaitForFirstConsumer`, the provisioner gets the node's topology annotation after scheduling and creates the volume in the correct AZ."

---

**Q: Explain the difference between `OrderedReady` and `Parallel` pod management policies.**

> "With `OrderedReady`, which is the default, the StatefulSet controller starts pods sequentially — `mysql-0` must reach `Running` and `Ready` before `mysql-1` is created. Deletion happens in reverse: `mysql-2` is terminated before `mysql-1`. This is essential for databases because replicas need to initialize by cloning from an already-running primary. If replicas start before the primary, the initial data clone fails and replication never establishes. `Parallel` mode starts and terminates all pods simultaneously — it's faster but completely ignores readiness ordering. Use it only for stateless workloads that happen to need PVCs, never for databases or clustered systems."

---

**Q: Why can't a pod in a StatefulSet be rescheduled to a different AZ after node failure?**

> "Because EBS volumes are physically locked to a single availability zone. When `mysql-1` is scheduled into `us-east-2b`, the CSI provisioner creates the EBS volume in `us-east-2b`. If the node dies and there are no other nodes in `us-east-2b`, the pod will stay in `Pending` indefinitely — it cannot be moved to `us-east-2a` because the volume can't follow it. This is not a bug; it's the system protecting your data. The correct resolution is to ensure each AZ has at least two nodes, which is why we configure an auto-scaler. If you see a pod stuck Pending after a node loss, the first thing to check is whether there are available nodes in the same AZ as the pod's PVC."

---

**Q: What is pod anti-affinity and what does `requiredDuringSchedulingIgnoredDuringExecution` mean?**

> "Pod anti-affinity is a scheduling rule that tells the Kubernetes scheduler to avoid placing a pod on a topology — like a node or AZ — that already has a pod matching a given label selector. The field `requiredDuringSchedulingIgnoredDuringExecution` is a hard constraint: the scheduler will not place the pod unless the anti-affinity rule can be satisfied. The `IgnoredDuringExecution` part means if the rule becomes violated after the pod is already running — because an existing pod was rescheduled to the same AZ — Kubernetes will not evict the running pod. It only enforces the rule at scheduling time. The practical effect for StatefulSets is that MySQL replicas are spread across AZs, which protects against a single AZ failure taking down multiple replicas."

---

## 8. Debugging

> *Fast diagnosis paths — follow the chain, don't guess.*

### Path 1: Pod Stuck in `Pending`

```bash
kubectl describe pod mysql-1
```

Look for the `Events` section at the bottom:

| Message | Root Cause | Fix |
|---|---|---|
| `0/4 nodes available: 4 node(s) had volume node affinity conflict` | Volume AZ ≠ available nodes | Add node in same AZ as PVC |
| `0/4 nodes available: 4 node(s) didn't match pod affinity/anti-affinity` | Hard anti-affinity can't be satisfied (too many pods, too few AZs) | Use soft anti-affinity OR add nodes in missing AZ |
| `persistentvolumeclaim "mysql-data-mysql-1" not found` | PVC doesn't exist yet (first run) | Check StorageClass provisioner is working |
| `node(s) had taints that the pod didn't tolerate` | Node taints blocking scheduling | Add toleration to pod spec OR remove taint |

```bash
# Verify PVC status
kubectl get pvc mysql-data-mysql-1
# If Pending: check StorageClass and CSI driver

kubectl describe pvc mysql-data-mysql-1
# Look for: ProvisioningFailed or WaitForFirstConsumer (normal — waiting for pod)

# Check CSI driver is running
kubectl get pods -n kube-system | grep ebs-csi
```

---

### Path 2: Pod Stuck in `ContainerCreating` After Node Loss

```bash
kubectl describe pod mysql-1
# Look for: "AttachVolume.Attach failed" or "Multi-Attach error"
```

**Multi-Attach error** means the volume is still attached to the old (dead) node. EBS needs to detect node failure and detach first.

```bash
# Check if old node is in NotReady state
kubectl get nodes

# Force-delete the old node object to release volume attachment
kubectl delete node <dead-node-name>
# AWS will then release the EBS attachment
# Pod will resume ContainerCreating → Running in ~30-90s
```

---

### Path 3: DNS Resolution Failing Inside Pod

```bash
kubectl exec -it mysql-0 -- bash

# Test full DNS
nslookup mysql-0.mysql.mysql-ha.svc.cluster.local

# If fails:
nslookup mysql.mysql-ha.svc.cluster.local
# If this works but pod-level fails → headless service issue
```

```bash
# Verify headless service exists and is correct
kubectl get svc mysql -n mysql-ha -o yaml | grep clusterIP
# Must be: clusterIP: ""  (None)

# Verify service selects the right pods
kubectl get endpoints mysql -n mysql-ha
# Should list individual pod IPs, not empty
```

---

### Path 4: StatefulSet Not Progressing (Stuck at 1/3 Ready)

```bash
kubectl get statefulset mysql
# Check READY column

kubectl describe statefulset mysql
# Look for: "create Pod mysql-1 in StatefulSet mysql failed"

# Check the blocked pod
kubectl describe pod mysql-1
# Often: readiness probe failing — next pod won't start until this one is Ready
```

```bash
# Check readiness probe output directly
kubectl exec -it mysql-1 -- mysqladmin ping -u root -p
# If connection refused: MySQL hasn't finished init
# Check init logs
kubectl logs mysql-1 --previous
kubectl logs mysql-1
```

---

### Path 5: Data Missing After Pod Restart

```bash
# Confirm the same PVC is attached
kubectl describe pod mysql-1 | grep -A5 Volumes

# Confirm PVC is bound (not recreated)
kubectl get pvc mysql-data-mysql-1 -o yaml | grep -E "phase|volumeName|creationTimestamp"
# If creationTimestamp is recent: PVC was accidentally deleted and recreated — data is gone

# Verify the PV still exists and is not in Released state
kubectl get pv | grep mysql-data-mysql-1
```

---

### Quick Diagnosis Matrix

```
Pod Pending
  └─ Volume affinity conflict → Add node in correct AZ
  └─ Anti-affinity unsatisfied → Use soft rule or add AZ
  └─ PVC Pending → CSI driver issue

Pod ContainerCreating
  └─ Volume still attached to dead node → Force delete node object
  └─ Image pull error → Check image name/tag/registry

Pod CrashLoopBackOff
  └─ Check kubectl logs <pod> --previous
  └─ Often: bad env vars, wrong password, corrupted data dir

StatefulSet not scaling up
  └─ Pod N-1 not Ready → readiness probe failing
  └─ Check kubectl describe statefulset for events
```

---

## 9. Kill Switch

> *10-second recall — the absolute minimum to hold in memory.*

```
StatefulSet = Stable Name + Stable DNS + Stable Disk

1. Name:    mysql-0, mysql-1, mysql-2  (ordinal, never random)
2. DNS:     mysql-0.mysql.mysql-ha.svc.cluster.local  (needs headless svc)
3. Disk:    mysql-data-mysql-0  (PVC survives pod AND StatefulSet deletion)

Order: 0 → 1 → 2 on create. 2 → 1 → 0 on delete. (OrderedReady)
AZ:   Volume locks to pod's AZ. Node dies → reschedule within SAME AZ only.
EBS:  WaitForFirstConsumer = volume created in pod's AZ. Always use this.

Headless = clusterIP: None. serviceName in StatefulSet must match exactly.
RWOP > RWO for databases. Parallel policy is unsafe for DBs.
PVC cleanup: Delete StatefulSet first, then PVCs manually.
```

---

## 10. Appendix

> *Quick reference card — commands, formats, cheatsheets.*

### DNS Format Cheatsheet

```
Pod DNS:     <pod-name>.<svc-name>.<namespace>.svc.cluster.local
Service DNS: <svc-name>.<namespace>.svc.cluster.local

Example:
  mysql-0.mysql.mysql-ha.svc.cluster.local   ← specific replica
  mysql.mysql-ha.svc.cluster.local            ← all pods (headless = returns all IPs)
```

### PVC Naming Convention

```
Pattern: <volumeClaimTemplate.metadata.name>-<statefulset-name>-<ordinal>

Example:
  template name: mysql-data
  statefulset:   mysql
  ordinals:      0, 1, 2
  ─────────────────────────
  mysql-data-mysql-0
  mysql-data-mysql-1
  mysql-data-mysql-2
```

### Command Reference

```bash
# Lifecycle
kubectl apply -f 01-sc.yaml 02-mysql-hs.yaml 03-mysql-sts.yaml
kubectl delete statefulset mysql
kubectl delete pvc -l app=mysql   # Only after deleting StatefulSet

# Observation
kubectl get pods -w                          # Watch ordered startup
kubectl get pods -o wide                     # Node + IP info
kubectl get pvc                              # All PVCs
kubectl get pv                               # All PVs + AZ binding
kubectl get statefulset mysql                # Status: READY/DESIRED
kubectl describe statefulset mysql           # Events, error details

# DNS Testing (from inside a pod)
kubectl exec -it mysql-0 -- nslookup mysql-0.mysql.mysql-ha.svc.cluster.local
kubectl exec -it mysql-0 -- nslookup mysql.mysql-ha.svc.cluster.local

# Node AZ distribution
kubectl get nodes -L topology.kubernetes.io/zone

# Namespace shortcut
kubectl config set-context --current --namespace=mysql-ha

# Scaling
kubectl scale statefulset mysql --replicas=5   # Adds mysql-3, mysql-4 in order
kubectl scale statefulset mysql --replicas=2   # Removes mysql-4, then mysql-3

# Rolling restart (triggers ordered restart, keeps PVCs)
kubectl rollout restart statefulset mysql
kubectl rollout status statefulset mysql

# Chaos test: write data, kill pod, verify data survives
kubectl exec -it mysql-1 -- bash -c "echo 'persist-test' >> /var/lib/mysql/testfile"
kubectl delete pod mysql-1
kubectl exec -it mysql-1 -- cat /var/lib/mysql/testfile
```

### StatefulSet vs Deployment vs DaemonSet

| | Deployment | StatefulSet | DaemonSet |
|---|---|---|---|
| Pod naming | Random suffix | Ordinal index | One per node |
| Startup order | Parallel (default) | Sequential | N/A |
| PVC per pod | No | Yes (`volumeClaimTemplates`) | Optional |
| Stable DNS | No | Yes (headless svc) | No |
| Use case | Stateless apps | DBs, queues, clusters | Node agents, logging |
| Scale to zero | Yes | Yes | No (node-bound) |

### Kubernetes Objects Required for a Working StatefulSet

```
Mandatory:
  ✅ StatefulSet
  ✅ Headless Service (clusterIP: None, name = spec.serviceName)
  ✅ StorageClass (with WaitForFirstConsumer for multi-AZ)

Strongly Recommended in Production:
  ✅ Secret (for passwords)
  ✅ Readiness Probe (required for OrderedReady to work correctly)
  ✅ Liveness Probe (for health-based restart)
  ✅ PodDisruptionBudget (prevent all replicas from being evicted simultaneously)
  ✅ NetworkPolicy (restrict which pods can reach the DB)

Optional (advanced):
  ☐ Operator (Percona, Vitess, KubeDB) — automates DBA-layer tasks
  ☐ HorizontalPodAutoscaler — rarely useful for databases (scale-out ≠ automatic replication setup)
```

### PodDisruptionBudget for MySQL HA

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: mysql-pdb
  namespace: mysql-ha
spec:
  minAvailable: 2       # At least 2 pods must be running during voluntary disruptions
  selector:
    matchLabels:
      app: mysql
```

```

