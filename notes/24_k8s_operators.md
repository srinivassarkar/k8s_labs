# Kubernetes Operators


---

## The Core Idea (Remember This Always)

**Kubernetes is a reconciliation engine.** Everything follows this loop:

```
Observe desired state → Compare with actual state → Act to close the gap → Repeat
```

This is called the **reconciliation loop** or **control loop**. It never stops.

### The Three Rules of Operators

| **Rule** | **What It Means** |
|---|---|
| **Declarative intent** | Users say *what* they want, not *how* to do it |
| **Continuous reconciliation** | The controller never stops checking and fixing |
| **Codified human expertise** | An Operator is a runbook turned into code |

### Why Operators Exist

Kubernetes handles stateless workloads well with `Deployment` and `ReplicaSet`. But **stateful, complex systems** like databases, message queues, and observability stacks need human expertise that built-in controllers don't have.

The Operator pattern answers: *"How do we encode that human expertise into Kubernetes itself?"*

> **Mental shortcut:** A Deployment controller knows how to restart pods. An Operator knows how to perform a zero-downtime PostgreSQL major version upgrade. The difference is **domain knowledge**.

---

## What Kubernetes Handles vs What It Doesn't

### What Kubernetes Natively Handles (No Operator Needed)

| **Concern** | **Native Resource** |
|---|---|
| Stateless scaling | `Deployment` + `ReplicaSet` |
| Ordered, identity-preserving pods | `StatefulSet` |
| Periodic jobs | `CronJob` |
| Persistent storage binding | `PVC` + `StorageClass` |
| Service discovery | `Service` |

### What Kubernetes Cannot Handle Natively

- Database replication topology (master election, replica lag)
- Backup scheduling with retention and validation
- Application-aware failover (promoting read replica to primary)
- Schema migration during upgrades
- Custom health checks with business logic
- Self-healing with application-specific remediation

### Stateless vs Stateful Applications

```
STATELESS APPLICATION
┌──────────────────────────────────────────────────┐
│ Any pod is interchangeable                      │
│ State lives in external Redis/DB                │
│ Request → Pod-1 (crash) → Pod-2 serves same    │
│ Kubernetes can freely kill/reschedule pods      │
└──────────────────────────────────────────────────┘

STATEFUL APPLICATION
┌──────────────────────────────────────────────────┐
│ Pod-0 = Primary (writes)                        │
│ Pod-1 = Replica (reads)                         │
│ Pod-2 = Replica (reads)                         │
│ Identity matters. Order matters.                │
│ Kubernetes cannot freely kill without risk      │
└──────────────────────────────────────────────────┘
```

### Operator Scope Rules

- An Operator can be **namespace-scoped** or **cluster-scoped** based on its RBAC
- CRDs are **always cluster-scoped** (they extend the API globally)
- An Operator controller runs as a **Deployment** inside the cluster
- OLM (Operator Lifecycle Manager) is **optional but recommended** – it's not built into vanilla Kubernetes

---

## When to Use What – Decision Guide

### Should You Use an Operator?

```
Does your application have complex operational runbooks?
├── No → Use Deployment + ConfigMap + Helm. Done.
└── Yes → Does the runbook require runtime reconciliation?
    ├── No → Use Helm/Kustomize for templated installs
    └── Yes → Does it require deep application-level knowledge?
        ├── No → Use a generic controller or Job-based automation
        └── Yes → ✅ Build or adopt an Operator
```

### Operator vs Helm vs Kustomize

| **Dimension** | **Helm** | **Kustomize** | **Operator** |
|---|---|---|---|
| **Primary purpose** | Package + install manifests | Environment-specific patching | Runtime lifecycle management |
| **When it runs** | At install/upgrade time (one-shot) | At apply time (one-shot) | Continuously (infinite loop) |
| **State awareness** | None | None | Full (watches cluster state) |
| **Domain knowledge** | None | None | Deep (codified in controller) |
| **Use together?** | ✅ Deploy Operator via Helm | ✅ Template CRs via Kustomize | ✅ Core automation engine |

> **Rule:** Helm/Kustomize for templating. Operators for lifecycle. They complement each other.

### Which Operator Framework to Choose

| **Situation** | **Framework** | **Language** |
|---|---|---|
| Production-grade, full control | Operator SDK (Go) or Kubebuilder | Go |
| Have existing Helm charts | Operator SDK (Helm) | YAML/Helm |
| Strong Ansible automation background | Operator SDK (Ansible) | Ansible |
| Polyglot team, webhook-based | Metacontroller | Any (JSON webhooks) |
| Java microservices ecosystem | Java Operator SDK | Java |

---

## How Operators Actually Work

### The Operator Component Stack

```
┌─────────────────────────────────────────────────────────────┐
│ OPERATOR                                                    │
│ (logical construct – not a Kubernetes object)              │
│                                                             │
│ ┌───────────────────────────────────────────────────────┐   │
│ │ CUSTOM CONTROLLER                                     │   │
│ │ (runs as a Deployment in the cluster)                │   │
│ │                                                       │   │
│ │ ┌──────────────┐ ┌──────────────────────────┐       │   │
│ │ │ Informer /   │ │ Reconciliation Loop      │       │   │
│ │ │ Watcher      │─▶│ (compare desired vs     │       │   │
│ │ │ (list+watch  │ │ actual, take action)    │       │   │
│ │ │ CRs via API) │ └──────────────┬───────────┘       │   │
│ │ └──────────────┘                │                   │   │
│ │                                 ▼                   │   │
│ │ Creates / Updates / Deletes Kubernetes resources:   │   │
│ │ StatefulSets, Deployments, Services, PVCs,          │   │
│ │ CronJobs, Secrets, ConfigMaps, Jobs                │   │
│ └───────────────────────────────────────────────────────┘   │
│                                                             │
│ ┌───────────────┐ ┌────────────────────────────────┐       │
│ │ CRD           │ │ Custom Resources (CRs)         │       │
│ │ (schema /     │─▶│ (user-created instances of    │       │
│ │ blueprint)    │ │ the CRD – express intent)     │       │
│ └───────────────┘ └────────────────────────────────┘       │
└─────────────────────────────────────────────────────────────┘
```

### Step-by-Step: What Happens When You Apply a CR

```
Step 1: User applies a CR (e.g., SleepInfo, BackupPolicy)
└── kubectl apply -f sleepinfo.yaml

Step 2: API Server validates the CR against the CRD schema
└── Rejects if required fields missing or wrong types

Step 3: CR is saved to etcd

Step 4: Operator controller's Informer detects the new/changed CR
└── Event placed in the controller's work queue

Step 5: Reconciliation loop processes the event
└── Fetches current state of all related resources
└── Compares with desired state from CR spec

Step 6: Controller takes action to close the gap
└── Creates/updates/deletes resources
└── Updates CR .status with observed state

Step 7: Loop continues – controller re-watches for further drift
```

### How OLM Manages Operator Lifecycle

```
OLM Components:
├── CatalogSource – points to an index of available Operators
├── Subscription – declares intent to install a specific Operator
├── InstallPlan – OLM generates this; lists resources to create
├── ClusterServiceVersion (CSV) – Operator manifest: CRDs, RBAC, Deployment
└── OperatorGroup – scopes which namespaces the Operator manages

Flow:
Subscription created
└── OLM resolves latest CSV from CatalogSource
└── Creates InstallPlan
└── Applies CSV → creates CRDs, RBAC, Operator Deployment
└── Operator pod starts → controller loop begins
```

### kube-green Example (Concrete Use Case)

```
SleepInfo CR applied
└── kube-green controller watches SleepInfo resources
└── Reads sleepAt, wakeUpAt, weekdays, timeZone, excludeRef
└── Computes next sleep/wake events
└── At sleepAt time:
    ├── Annotates Deployments with original replica count
    ├── Patches Deployment spec.replicas = 0
    ├── Patches StatefulSet spec.replicas = 0
    └── Sets spec.suspend = true on CronJobs
└── At wakeUpAt time:
    ├── Reads saved replica count from annotations
    └── Restores spec.replicas to original value
└── Updates SleepInfo .status.lastScheduleTime
```

---

## Hands-On Examples

### Install OLM

```bash
# Install OLM v0.32.0
curl -sL https://github.com/operator-framework/operator-lifecycle-manager/releases/download/v0.32.0/install.sh \
| bash -s v0.32.0

# Verify OLM is running
kubectl get pods -n olm
kubectl get pods -n operators

# Verify OLM CRDs are registered
kubectl get crd | grep operators.coreos.com
```

### Install kube-green via OLM

```bash
# One-liner from OperatorHub
kubectl create -f https://operatorhub.io/install/kube-green.yaml
```

**Or using a Subscription manifest (preferred for GitOps):**

```yaml
# 00-kube-green-subscription.yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: my-kube-green
  namespace: operators
spec:
  channel: alpha
  name: kube-green
  source: operatorhubio-catalog
  sourceNamespace: olm
```

```bash
kubectl apply -f 00-kube-green-subscription.yaml
```

### Verify Operator Installation

```bash
# Check ClusterServiceVersion – must show PHASE: Succeeded
kubectl get csv -n operators
kubectl get csv -n operators -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'

# Confirm CRD is registered
kubectl api-resources | grep sleepinfo
kubectl get crd sleepinfos.kube-green.com -o yaml

# Check the controller pod is running
kubectl get pods -n operators -l app=kube-green
```

### Deploy Sample Application

```yaml
# 01-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        resources:
          requests:
            cpu: "50m"
            memory: "64Mi"
          limits:
            cpu: "100m"
            memory: "128Mi"
```

```bash
kubectl apply -f 01-app.yaml
kubectl get deployment nginx
```

### Create SleepInfo CR

```yaml
# 02-sleepinfo.yaml
apiVersion: kube-green.com/v1alpha1
kind: SleepInfo
metadata:
  name: sleepinfo-sample
  namespace: default
  labels:
    app: kube-green
spec:
  # Workloads matching excludeRef are never scaled down
  excludeRef:
  - apiVersion: apps/v1
    kind: Deployment
    name: critical-app
  
  # Scale down at 17:55 IST every day
  sleepAt: "17:55"
  wakeUpAt: "08:00"
  weekdays: "0-6"          # 0=Sunday, 6=Saturday
  suspendCronJobs: true
  suspendDeployments: true
  suspendStatefulSets: true
  timeZone: "Asia/Kolkata"
```

```bash
kubectl apply -f 02-sleepinfo.yaml

# Verify CR was accepted
kubectl get sleepinfo -n default
kubectl describe sleepinfo sleepinfo-sample -n default
```

### Observe Operator Behavior

```bash
# Watch deployment replica count in real time
kubectl get deployment nginx -w -n default

# Check status subresource on the CR
kubectl get sleepinfo sleepinfo-sample -n default -o jsonpath='{.status}'

# Inspect annotations kube-green adds to save original replica counts
kubectl get deployment nginx -n default \
  -o jsonpath='{.metadata.annotations}' | jq .

# Watch events from the controller
kubectl get events -n default --sort-by='.lastTimestamp' | grep -i sleep

# Check controller logs directly
kubectl logs -n operators \
  $(kubectl get pods -n operators -l app=kube-green -o jsonpath='{.items[0].metadata.name}') \
  --follow
```

### Custom BackupPolicy Operator Example

**CRD Definition:**
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: backuppolicies.ops.mycompany.com
spec:
  group: ops.mycompany.com
  names:
    kind: BackupPolicy
    plural: backuppolicies
    singular: backuppolicy
  scope: Namespaced
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            required: ["schedule", "retentionDays", "targetPVC"]
            properties:
              schedule:
                type: string
              retentionDays:
                type: integer
                minimum: 1
              targetPVC:
                type: string
          status:
            type: object
            properties:
              lastBackupTime:
                type: string
                format: date-time
              phase:
                type: string
    subresources:
      status: {}
```

**CR Instance:**
```yaml
apiVersion: ops.mycompany.com/v1
kind: BackupPolicy
metadata:
  name: mysql-backup
  namespace: default
spec:
  schedule: "0 1 * * *"
  retentionDays: 7
  targetPVC: mysql-data-pvc
```

---

## Production Architecture

### Production Operator Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│ Kubernetes Cluster                                             │
│                                                                 │
│ ┌──────────┐ ┌──────────────────────────────────────────┐     │
│ │ GitOps   │ │ operators namespace                      │     │
│ │ (ArgoCD  │─▶│ ┌──────────┐ ┌──────────────────────┐ │     │
│ │ /Flux)   │ │ │ OLM      │ │ kube-green-operator  │ │     │
│ └──────────┘ │ │ (manages │ │ (Deployment)         │ │     │
│              │ │ CSVs,    │ │ controller loop       │ │     │
│              │ │ upgrades)│ │ watches SleepInfo CRs │ │     │
│              │ └──────────┘ └──────────────────────┘ │     │
│              └──────────────────────────────────────────┘     │
│                                                                 │
│ ┌──────────────────────────────────────────────────────────┐   │
│ │ default namespace                                        │   │
│ │                                                          │   │
│ │ SleepInfo CR ──▶ kube-green watches ──▶ scales nginx=0  │   │
│ │ (excludes: critical-app)                                │   │
│ │                                                          │   │
│ │ Deployment: nginx (scaled to 0 at sleepAt)              │   │
│ │ Deployment: critical-app (EXCLUDED – stays at replicas) │   │
│ └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Real-World Operator Adoption Patterns

| **Pattern** | **Description** | **Example** |
|---|---|---|
| **Day-2 Operator** | Install app via Helm, manage lifecycle via Operator | Deploy Prometheus via Helm, manage alerts via Prometheus Operator |
| **Full lifecycle Operator** | Operator handles install + upgrade + backup + failover | Zalando PostgreSQL Operator |
| **Policy Operator** | Enforces cluster-wide policies via CRDs | Kyverno, OPA/Gatekeeper |
| **GitOps + Operator** | CRs stored in Git, applied by ArgoCD, acted on by Operator | ArgoCD Application CR |
| **Cost control Operator** | Automates resource scaling to reduce cost | kube-green SleepInfo |

### Mature Operator Checklist (Production Readiness)

```
☐ CRD has OpenAPI v3 schema validation (required fields, types, constraints)
☐ Controller has leader election (prevents split-brain in HA)
☐ CR .status subresource updated after every reconcile
☐ Controller emits Kubernetes Events on state changes
☐ Prometheus metrics exposed (/metrics endpoint)
☐ Graceful error handling with exponential backoff
☐ Finalizers set on CRs to handle cleanup before deletion
☐ RBAC is least-privilege (no cluster-admin for namespaced operators)
☐ OLM Subscription pinned to channel with upgrade approval strategy
☐ Integration tests validate reconciliation under failure scenarios
```

### Common Real-World Operators and Their CRDs

| **Operator** | **Key CRDs** | **What It Manages** |
|---|---|---|
| Prometheus Operator | `Prometheus`, `ServiceMonitor` | Monitoring stack |
| cert-manager | `Certificate`, `ClusterIssuer` | TLS certificate lifecycle |
| ArgoCD | `Application`, `AppProject` | GitOps deployments |
| Zalando PostgreSQL | `postgresql` | PostgreSQL clusters |
| Istio | `VirtualService`, `DestinationRule` | Service mesh |
| Kyverno | `ClusterPolicy`, `Policy` | Admission policy enforcement |
| Velero | `Backup`, `Restore`, `Schedule` | Cluster backup/restore |
| kube-green | `SleepInfo` | Off-hours workload scaling |

---

## Common Mistakes (What Breaks in Real Life)

### ❌ Mistake 1: Applying CR Before CRD is Registered

**Symptom:**
```
error: unable to recognize "sleepinfo.yaml": no matches for kind "SleepInfo"
```

**Root Cause:** CR applied before OLM finished installing the Operator.

**Fix:** Wait for CSV to reach `Succeeded` phase:
```bash
kubectl wait --for=jsonpath='{.status.phase}'=Succeeded \
  csv/kube-green.v0.7.0 -n operators --timeout=120s
kubectl apply -f sleepinfo.yaml
```

### ❌ Mistake 2: CR in Wrong Namespace

**Symptom:** CR applies successfully, but nothing happens.

**Root Cause:** OperatorGroup scopes which namespaces the Operator watches. If the CR is in a namespace not covered, the controller ignores it.

**Fix:** Check OperatorGroup:
```bash
kubectl get operatorgroup -n operators -o yaml
```
Ensure `targetNamespaces` includes your namespace, or set it to `""` for cluster-wide scope.

### ❌ Mistake 3: Wrong Timezone in sleepAt/wakeUpAt

**Symptom:** Workloads scaled down at wrong time; users report service unavailable during business hours.

**Root Cause:** `timeZone` not set (defaults to UTC) or set incorrectly.

**Fix:** Always explicitly set `timeZone` with IANA identifiers:
```bash
kubectl get sleepinfo sleepinfo-sample -o jsonpath='{.spec.timeZone}'
kubectl get sleepinfo sleepinfo-sample -o jsonpath='{.status.lastScheduleTime}'
```

### ❌ Mistake 4: CRD Schema Too Permissive

**Symptom:** CR applied with wrong values (e.g., `retentionDays: -1`), no error, but backup jobs never clean up.

**Root Cause:** CRD schema lacks `minimum` constraints or required field validation.

**Fix:** Add proper OpenAPI v3 validation:
```yaml
retentionDays:
  type: integer
  minimum: 1
  maximum: 365
```

### ❌ Mistake 5: Missing Finalizer Causes Orphaned Resources

**Symptom:** CR deleted, but the StatefulSet/CronJob/PVC it created remains.

**Root Cause:** Controller never set a finalizer on the CR.

**Fix:** Controller must add a finalizer during creation:
```
CR created → controller adds finalizer to metadata.finalizers
CR deleted → controller runs cleanup → removes finalizer → K8s completes deletion
```

### ❌ Mistake 6: Controller Without Leader Election in HA

**Symptom:** Multiple controller replicas all try to reconcile simultaneously.

**Root Cause:** Controller Deployment scaled to >1 replica without leader election.

**Fix:** Enable leader election:
```yaml
args:
- --leader-elect=true
```

### ❌ Mistake 7: excludeRef Not Working

**Symptom:** `critical-app` deployment scaled to 0 despite being in `excludeRef`.

**Root Cause:** `excludeRef` name field is case-sensitive and must match exactly.

**Fix:**
```bash
kubectl get deployment critical-app -n default -o jsonpath='{.metadata.name}'
# Compare exactly with excludeRef.name in SleepInfo CR
```

---

## Interview Answers (Simple, Ready to Say)

### Q: What is a Kubernetes Operator?

> "A Kubernetes Operator is a software extension that uses Custom Resource Definitions and a custom controller to encode domain-specific operational knowledge into Kubernetes itself. Instead of a human manually performing tasks like database failover or backup scheduling, an Operator runs as a controller inside the cluster, watches Custom Resources that declare intent, and continuously reconciles the actual state to match that intent. The key insight is that an Operator manages the full lifecycle of an application – including upgrades, scaling, backup, and self-healing – not just installation."

### Q: What is the difference between a CRD, a CR, and a controller?

> "A CRD is the schema – it tells the API server that a new resource type exists and what fields it accepts. A CR is an instance of that CRD – it's what the user creates to express their intent. A controller is the reconciliation loop that watches for CRs, compares desired state with actual state, and takes actions to close any gap. In a production Operator, the controller runs as a Deployment, uses the Kubernetes informer mechanism to efficiently watch for changes, and updates the CR's status subresource."

### Q: How is an Operator different from Helm?

> "Helm is a package manager – it templates and installs Kubernetes manifests at deploy time, then steps aside. It has no awareness of what happens after installation. An Operator runs continuously as a controller and actively manages the application throughout its entire lifecycle. You can actually use both together: use Helm to install the Operator itself, and use CRs to tell the Operator what to do at runtime. Helm for initial delivery, Operators for ongoing management."

### Q: What is the reconciliation loop and why is it important?

> "The reconciliation loop is the core pattern in Kubernetes controllers. It continuously does three things: observe the current state, compare it with the desired state expressed in a resource spec, and take action to eliminate any difference. It's important because it makes the system self-healing – if something drifts due to a node crash or manual change, the controller will detect it and correct it without human intervention. Operators build on this exact same pattern with application-specific logic."

### Q: What is OLM and why would you use it?

> "OLM stands for Operator Lifecycle Manager. It's a tool that manages the lifecycle of Operators themselves – installation, upgrades, dependency resolution, and RBAC. Without OLM, you'd manually apply CRDs, RBAC, and Deployment specs for each Operator. OLM automates that with Subscriptions (which Operator you want) and ClusterServiceVersions (which bundle all the Operator's metadata). It's especially useful in environments with many Operators or where automated upgrade management is required."

### Q: How does kube-green work?

> "kube-green is an Operator that automates resource conservation in non-production environments. You create a SleepInfo Custom Resource in a namespace, specifying a sleep time, wake-up time, days of the week, and timezone. The controller watches for SleepInfo resources and, at the configured sleep time, scales down all Deployments and StatefulSets to zero replicas and suspends CronJobs. Before scaling down, it saves the original replica counts as annotations. At wake-up time, it reads those annotations and restores the original counts. You can exclude specific workloads using excludeRef."

### Q: What makes an Operator production-grade?

> "A production-grade Operator needs proper status reporting – updating the CR's status subresource so users can observe what the controller has done. It needs to emit Kubernetes Events so operations are auditable. It needs leader election so multiple replicas don't cause race conditions. It needs finalizers on CRs to handle cleanup logic. It needs exponential backoff and retry logic for transient errors. It should expose Prometheus metrics for monitoring. And the CRD schema needs proper OpenAPI v3 validation to prevent invalid configurations."

---

## Debugging – Fast Diagnosis

### Operator Not Working – Primary Diagnosis Path

```
SYMPTOM: CR applied, nothing happens
│
├── Step 1: Is the CRD registered?
│   kubectl api-resources | grep <kind>
│   └── Not found → CRD not installed → check Operator install / CSV status
│
├── Step 2: Is the CSV in Succeeded phase?
│   kubectl get csv -n operators
│   └── Not Succeeded → OLM issue → check OLM pods, CatalogSource
│
├── Step 3: Is the controller pod running?
│   kubectl get pods -n operators
│   └── CrashLoopBackOff → check logs: kubectl logs <pod> -n operators
│
├── Step 4: Is the CR valid?
│   kubectl describe <cr-kind> <cr-name> -n <namespace>
│   └── Events show validation errors → fix CR spec
│
├── Step 5: Does the controller have RBAC to watch this namespace?
│   kubectl get operatorgroup -n operators -o yaml
│   └── targetNamespaces doesn't include your namespace → update OperatorGroup
│
└── Step 6: Check controller logs for reconcile errors
    kubectl logs -n operators <controller-pod> --follow | grep -i error
```

### kube-green Not Scaling Down

```
SYMPTOM: sleepAt time passed, deployment still has replicas
│
├── Is the SleepInfo CR in the same namespace as the Deployment?
│   kubectl get sleepinfo -n default
│   └── Not there → CR in wrong namespace
│
├── Is timeZone correct?
│   kubectl get sleepinfo -o jsonpath='{.spec.timeZone}'
│   └── Wrong timezone → workloads scaled at wrong wall clock time
│
├── Is the Deployment in excludeRef by mistake?
│   kubectl get sleepinfo -o jsonpath='{.spec.excludeRef}'
│   └── Name matches → remove from excludeRef
│
├── Did the controller observe the sleepAt event?
│   kubectl get sleepinfo -o jsonpath='{.status}'
│   └── lastScheduleTime not updated → controller not processing
│
└── Check controller logs
    kubectl logs -n operators <kube-green-pod> --follow | grep nginx
```

### Useful Diagnostic Commands

```bash
# Check all Operator-related resources at once
kubectl get csv,subscription,installplan -n operators

# Get detailed status of an InstallPlan
kubectl describe installplan -n operators

# Check if a CRD has proper schema
kubectl get crd sleepinfos.kube-green.com \
  -o jsonpath='{.spec.versions[0].schema.openAPIV3Schema.properties.spec}'

# Find all CRs of a type across all namespaces
kubectl get sleepinfo --all-namespaces

# Check annotations kube-green added before scaling
kubectl get deployment nginx -n default \
  -o jsonpath='{.metadata.annotations}' | python3 -m json.tool

# Watch events in a namespace for Operator activity
kubectl get events -n default -w --sort-by='.lastTimestamp'

# Check RBAC that an Operator controller has
kubectl get clusterrolebinding | grep kube-green
kubectl describe clusterrole kube-green-manager-role
```

---

## 30-Second Quick Revision

### The Essentials

```
OPERATOR = CRD + CR + Custom Controller
         = Schema + User Intent + Reconciliation Logic
```

### The Three Parts

```
CRD     → blueprint (API extension, cluster-scoped)
CR      → user's declaration of desired state
Controller → infinite loop: observe → diff → act
```

### Key Differences

```
Operator ≠ Helm (Helm is one-shot; Operator is continuous)
Operator = runs as a Deployment inside the cluster
OLM manages Operators the way Operators manage apps
```

### kube-green Quick Reference

```
SleepInfo CR → controller scales replicas=0 at sleepAt, restores at wakeUpAt
excludeRef → workloads never touched by kube-green
```

### The Reconciliation Loop

```
DESIRED (spec) vs ACTUAL (cluster) → ACTION → update STATUS
```

### Golden Rule

```
CRD is global. CR is namespaced. Controller watches CRs, manages resources.
```

---

## Appendix – Quick Reference

### OLM Commands

```bash
# Install OLM
curl -sL https://github.com/operator-framework/operator-lifecycle-manager/releases/download/v0.32.0/install.sh | bash -s v0.32.0

# Check OLM health
kubectl get pods -n olm
kubectl get pods -n operators

# List available Operators
kubectl get packagemanifest -n olm | grep kube-green

# Check installed Operators
kubectl get csv --all-namespaces
kubectl get csv -n operators -o wide

# Check Subscriptions
kubectl get subscription -n operators
kubectl describe subscription my-kube-green -n operators

# Check InstallPlans
kubectl get installplan -n operators
kubectl describe installplan <name> -n operators
```

### CRD Commands

```bash
# List all CRDs
kubectl get crd

# Inspect a CRD's schema
kubectl get crd sleepinfos.kube-green.com -o yaml

# Check API group/version
kubectl api-resources | grep sleepinfo

# Check CRD conditions
kubectl get crd sleepinfos.kube-green.com \
  -o jsonpath='{.status.conditions[*].type}'
```

### Custom Resource Commands

```bash
# List CRs
kubectl get sleepinfo -n default
kubectl get sleepinfo --all-namespaces

# Describe a CR (includes Events)
kubectl describe sleepinfo sleepinfo-sample -n default

# Check CR status
kubectl get sleepinfo sleepinfo-sample -n default -o jsonpath='{.status}'

# Edit a CR in place
kubectl edit sleepinfo sleepinfo-sample -n default

# Delete a CR (triggers finalizer logic)
kubectl delete sleepinfo sleepinfo-sample -n default
```

### kube-green SleepInfo Field Reference

| **Field** | **Type** | **Required** | **Default** | **Description** |
|---|---|---|---|---|
| `sleepAt` | string (HH:MM or cron) | ✅ | — | Time to scale workloads to 0 |
| `weekdays` | string (cron notation) | ✅ | — | Days to apply schedule |
| `wakeUpAt` | string (HH:MM or cron) | ❌ | — | Time to restore workloads |
| `timeZone` | string (IANA) | ❌ | UTC | Timezone for sleep/wake times |
| `excludeRef` | list | ❌ | — | Resources to exclude from scaling |
| `suspendCronJobs` | bool | ❌ | false | Suspend CronJobs during sleep |
| `suspendDeployments` | bool | ❌ | true | Scale Deployments to 0 |
| `suspendStatefulSets` | bool | ❌ | true | Scale StatefulSets to 0 |

### SleepInfo weekdays Cheatsheet

```
"1-5" → Monday to Friday
"0-6" → Every day (0=Sunday, 6=Saturday)
"1,3,5" → Monday, Wednesday, Friday
"0,6" → Weekend only
```

### IANA Timezone Examples

```
Asia/Kolkata    → IST (UTC+5:30)
Asia/Singapore  → SGT (UTC+8)
Asia/Tokyo      → JST (UTC+9)
Asia/Dubai      → GST (UTC+4)
Asia/Shanghai   → CST (UTC+8)
```

### Operator Development Frameworks

```
Go + Operator SDK   → operator-sdk init + operator-sdk create api
Go + Kubebuilder    → kubebuilder init + kubebuilder create api
Helm-based          → operator-sdk init --plugins helm
Ansible-based       → operator-sdk init --plugins ansible
Java                → Java Operator SDK
```

---

