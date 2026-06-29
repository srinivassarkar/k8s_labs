# Kubernetes CRDs & Custom Resources


---

## The Core Idea (Remember This Always)

**Kubernetes is a state reconciliation engine.**

Everything in Kubernetes follows this same pattern:

```
Desired State (what you want) → Stored in etcd → Controller watches → Makes it happen → Actual State (what really is)
```

This pattern works for built-in resources (Pods, Deployments) AND for custom resources you create.

**The Three Parts of a Custom Resource:**

```
CRD = The Schema (the blueprint)
CR  = The Instance (the actual object)
Controller = The Behavior (makes things happen)
```

> **Think of it like:** A class definition → an object instance → the code that runs it.

---

## What Kubernetes Does and Doesn't Do

### What Kubernetes DOES for you when you add a CRD

- **Registers** the new resource type in the API server
- **Creates REST endpoints** so you can use `kubectl` to get, create, update, and delete
- **Validates** objects against your schema
- **Stores** CRs in etcd just like built-in resources
- **Makes it appear** in `kubectl api-resources`

### What Kubernetes Does NOT Do

| **It Does NOT** | **Why This Matters** |
|---|---|
| Create a controller automatically | CRs are just stored data unless you build a controller |
| Add default values | You need a Mutating Admission Webhook for defaults |
| Run business logic | No scheduling, no pod creation, no S3 uploads |
| Update status | Your controller must write to `.status` |
| Put CRs in the core API group (`/api`) | CRs always live under `/apis/<group>` |

### Hard Rules to Remember

| **Rule** | **Details** |
|---|---|
| CRD name format | **Must be** `<plural>.<group>` – e.g., `backuppolicies.ops.mycompany.com` |
| API version | `apiextensions.k8s.io/v1` (required; `v1beta1` is gone) |
| OpenAPI schema | **Required** for validation |
| `storage: true` | Exactly **one** version must have this |
| `served: true` | Version must be served to accept requests |
| Scope options | Only `Namespaced` or `Cluster` – no other choices |

---

## When to Use CRDs vs Alternatives

### Decision Flowchart

```
Do you need a new Kubernetes-native object type?
│
├── YES → Does it represent desired state that can drift?
│   ├── YES → CRD + Custom Controller (or Operator)
│   └── NO → CRD alone (as a config store)
│
└── NO → Use ConfigMap / Secret / existing resource with labels
```

### CRD vs ConfigMap vs Aggregated API Server

| **Criteria** | **CRD** | **ConfigMap** | **Aggregated API** |
|---|---|---|---|
| Schema validation | ✅ Yes | ❌ No | ✅ Yes |
| `kubectl` support | ✅ Full | ✅ Full | ✅ Full |
| RBAC per kind | ✅ Yes | ✅ Yes | ✅ Yes |
| Custom status | ✅ Yes | ❌ No | ✅ Yes |
| Custom columns in `kubectl get` | ✅ Yes | ❌ No | ✅ Yes |
| Complexity | Low | None | Very High |
| Best for | Platform abstractions, operators | App config | Complex protocols |

### Example: BackupPolicy CRD vs CronJob

| **Feature** | **CronJob** | **BackupPolicy CRD + Controller** |
|---|---|---|
| Schedule execution | ✅ Built-in | ✅ Controller-driven |
| PVC awareness | ❌ Manual volumes | ✅ Declarative field |
| Retention policy | ❌ Not supported | ✅ `retentionDays` field |
| Backup validation hooks | ❌ External scripts | ✅ Encoded in controller |
| Status reporting | Limited | ✅ Controller writes status |

---

## How It Actually Works

### Phase 1: Registering the CRD

```
kubectl apply -f backup-policy-crd.yaml
│
▼
API Server validates the CRD
│
▼
CRD stored in etcd
│
▼
API server dynamically creates new REST endpoints:
/apis/ops.mycompany.com/v1/namespaces/{ns}/backuppolicies
│
▼
`kubectl api-resources` now shows BackupPolicy
```

### Phase 2: Creating a Custom Resource

```
kubectl apply -f mysql-backup.yaml
│
▼
API Server receives the request
│
▼
Admission Controllers run (mutating → validating)
│
▼
OpenAPI schema validation checks all fields
│
▼
Object written to etcd
│
▼
Custom Controller (if running) gets a watch event
│
▼
Controller runs reconcile loop → creates CronJob / updates status
```

### The Control Loop (What Controllers Do)

```
while true:
    # 1. Read desired state from API
    backupPolicy = getBackupPolicy("mysql-backup")
    
    # 2. Check what actually exists
    existingCronJob = getExistingCronJob(backupPolicy.targetPVC)
    
    # 3. Reconcile (make reality match desired)
    if existingCronJob == null:
        createCronJob(backupPolicy.spec)
    else if needsUpdate(existingCronJob, backupPolicy.spec):
        updateCronJob(existingCronJob, backupPolicy.spec)
    
    # 4. Update status to reflect what happened
    backupPolicy.status.lastReconciled = time.Now()
    updateStatus(backupPolicy)
```

### `.spec` vs `.status` – Who Owns What

| **Field** | **Owner** | **Purpose** |
|---|---|---|
| `.spec` | User / GitOps | Declares **what should happen** |
| `.status` | Controller | Reports **what actually happened** |
| `.metadata` | API server | Identity, labels, resource version |

> **Critical:** Controllers should **never** modify `.spec`. They only update `.status`. This is why `status` is a separate subresource – so `kubectl apply` can't accidentally overwrite controller-managed status.

### Built-in Resources vs CRDs

| **Aspect** | **Built-in (e.g., Deployment)** | **Custom (CRD)** |
|---|---|---|
| Schema defined | Compiled into API server | Applied as CRD YAML |
| Schema location | OpenAPI at `/openapi/v2` | In the CRD `.spec.versions[].schema` |
| Controller | Comes with Kubernetes | Must be deployed separately |
| `kubectl explain` | Works natively | Works if you add descriptions |

---

## Hands-On Examples

### CRD Definition – Full Example

```yaml
# backup-policy-crd.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition

metadata:
  name: backuppolicies.ops.mycompany.com   # MUST be <plural>.<group>

spec:
  group: ops.mycompany.com                  # API group for CRs

  names:
    plural: backuppolicies                  # URL path
    singular: backuppolicy                  # Used in CLI messages
    kind: BackupPolicy                      # Used as `kind:` in manifests
    shortNames:
      - bp                                  # `kubectl get bp`

  scope: Namespaced                         # Namespaced | Cluster

  versions:
  - name: v1
    served: true                            # Expose via API
    storage: true                           # Only ONE version can be true

    # Enable status subresource
    subresources:
      status: {}

    # Custom columns for `kubectl get bp`
    additionalPrinterColumns:
    - name: Schedule
      type: string
      jsonPath: .spec.schedule
    - name: Retention
      type: integer
      jsonPath: .spec.retentionDays
    - name: PVC
      type: string
      jsonPath: .spec.targetPVC
    - name: Age
      type: date
      jsonPath: .metadata.creationTimestamp

    schema:
      openAPIV3Schema:
        type: object
        required: ["spec"]
        properties:
          spec:
            type: object
            required: ["schedule", "retentionDays", "targetPVC"]
            properties:
              schedule:
                type: string
                description: "Cron expression (e.g., '0 1 * * *' = daily at 1AM)"
                pattern: '^(\*|[0-9,\-\*/]+)\s+(\*|[0-9,\-\*/]+)\s+(\*|[0-9,\-\*/]+)\s+(\*|[0-9,\-\*/]+)\s+(\*|[0-9,\-\*/]+)$'
              retentionDays:
                type: integer
                description: "Days to retain completed backups"
                minimum: 1
                maximum: 365
              targetPVC:
                type: string
                description: "Name of the PersistentVolumeClaim to back up"
          status:
            type: object
            properties:
              lastBackupTime:
                type: string
                format: date-time
              lastBackupStatus:
                type: string
                enum: ["Success", "Failed", "Running"]
              message:
                type: string
```

### Custom Resource Instance

```yaml
# mysql-backup.yaml
apiVersion: ops.mycompany.com/v1   # <group>/<version>
kind: BackupPolicy                  # From CRD spec.names.kind

metadata:
  name: mysql-backup
  namespace: default

spec:
  schedule: "0 1 * * *"             # Daily at 1:00 AM UTC
  retentionDays: 7
  targetPVC: mysql-data-pvc
```

### Full Lifecycle Commands

```bash
# --- CRD Management ---

# Apply CRD (registers the new type)
kubectl apply -f backup-policy-crd.yaml

# Verify CRD exists
kubectl get crd backuppolicies.ops.mycompany.com

# Inspect CRD
kubectl describe crd backuppolicies.ops.mycompany.com

# Check API endpoint is live
kubectl api-resources | grep backup
kubectl api-resources --api-group=ops.mycompany.com

# --- CR Management ---

# Wait for CRD to be ready
kubectl wait --for=condition=Established crd/backuppolicies.ops.mycompany.com

# Create CR
kubectl apply -f mysql-backup.yaml

# List CRs
kubectl get backuppolicies
kubectl get bp           # Short name works
kubectl get bp -n default

# Inspect CR
kubectl describe backuppolicy mysql-backup
kubectl get backuppolicy mysql-backup -o yaml

# Check status (set by controller)
kubectl get backuppolicy mysql-backup -o jsonpath='{.status}'

# --- Cleanup ---
# Always delete CRs first, then the CRD
kubectl delete -f mysql-backup.yaml
kubectl delete -f backup-policy-crd.yaml

# WARNING: Deleting a CRD deletes ALL instances of that CR!

# --- Exploration ---
kubectl explain backuppolicy.spec        # Works if descriptions are set
kubectl get --raw /openapi/v2 | jq .     # Full OpenAPI schema for all resources
```

---

## Production Architecture

### The Operator Pattern

```
┌─────────────────────────────────────────────────────────┐
│ KUBERNETES CLUSTER                                    │
│                                                       │
│ User runs: kubectl apply -f mysql-backup.yaml        │
│                      │                                │
│                      ▼                                │
│               ┌──────────────┐ stored in             │
│               │BackupPolicy  │◄─────────┐             │
│               │ (Custom Resource)│        │             │
│               └──────────────┘         │             │
│                      ▲                  │ etcd        │
│                      │ watch event      │             │
│                      │                  │             │
│               ┌───────────────────────────┐           │
│               │ Custom Controller (Pod)   │           │
│               │                           │           │
│               │ Informer → Queue → Loop   │           │
│               │       │                   │           │
│               │   Creates/Updates CronJob │           │
│               │   Enforces retentionDays  │           │
│               │   Writes status           │           │
│               └───────────────────────────┘           │
│                      │                                │
│                      ▼                                │
│               ┌──────────────┐                        │
│               │   CronJob    │──► Job ──► Pod ──► S3 │
│               └──────────────┘                        │
└─────────────────────────────────────────────────────────┘
```

### Real Production CRD Use Cases

| **Product** | **CRD Kinds** | **What It Manages** |
|---|---|---|
| cert-manager | `Certificate`, `Issuer`, `ClusterIssuer` | TLS certificates |
| Prometheus Operator | `ServiceMonitor`, `PrometheusRule` | Monitoring config |
| Argo CD | `Application`, `AppProject` | GitOps deployments |
| Istio | `VirtualService`, `DestinationRule` | Service mesh routing |
| KEDA | `ScaledObject`, `TriggerAuthentication` | Event-driven autoscaling |
| Velero | `Backup`, `Restore`, `Schedule` | Cluster backup & restore |

### RBAC for CRDs

```yaml
# Restrict who can create BackupPolicies
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: backuppolicy-admin
rules:
# Full permissions on the CR
- apiGroups: ["ops.mycompany.com"]
  resources: ["backuppolicies"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

# Only controller should update status
- apiGroups: ["ops.mycompany.com"]
  resources: ["backuppolicies/status"]
  verbs: ["get", "update", "patch"]
```

---

## Common Mistakes (What Breaks in Real Life)

### ❌ Mistake 1: Deleting CRD Before Deleting CRs

**What happens:** Kubernetes deletes ALL CR instances from etcd. Data is gone forever.

**Fix:**
```bash
# Always delete instances first
kubectl delete backuppolicies --all -A
# Then delete CRD
kubectl delete crd backuppolicies.ops.mycompany.com
```

### ❌ Mistake 2: Wrong CRD Name Format

**What happens:** `kubectl apply` fails with validation error.

**Fix:** `metadata.name` must be `<plural>.<group>`. Wrong: `backup.ops.mycompany.com` (singular). Right: `backuppolicies.ops.mycompany.com` (plural).

### ❌ Mistake 3: Missing `required` Fields

**What happens:** CRs created without critical fields; controller crashes.

**Fix:** Always define `required` in the schema:
```yaml
spec:
  type: object
  required: ["schedule", "retentionDays", "targetPVC"]
```

### ❌ Mistake 4: Controller Updates `.spec` Instead of `.status`

**What happens:** User changes to `.spec` are silently reverted every reconcile cycle.

**Fix:** Enable `status` subresource and use `client.Status().Update()` – NOT `client.Update()`.

### ❌ Mistake 5: CR Created Before CRD Exists

**What happens:** "no matches for kind 'BackupPolicy'"

**Fix:**
```bash
kubectl apply -f backup-policy-crd.yaml
kubectl wait --for=condition=Established crd/backuppolicies.ops.mycompany.com
kubectl apply -f mysql-backup.yaml
```

### ❌ Mistake 6: Two Versions Both Marked `storage: true`

**What happens:** CRD apply fails validation.

**Fix:** Exactly one version gets `storage: true`; others get `storage: false`.

---

## Interview Answers (Simple, Ready to Say)

### Q: What is the difference between a CRD and a Custom Resource?

> "A CRD – CustomResourceDefinition – is the schema registration. It tells the API server that a new resource type exists and what fields are valid. Think of it like a class definition. A Custom Resource is an instance of that type – an actual object you create with `kubectl apply`. The CRD is the blueprint; the CR is the object. Once you apply a CRD, the API server dynamically starts serving REST endpoints for that new type."

### Q: Can a Custom Resource do anything by itself?

> "By itself, a Custom Resource is passive – it's just data stored in etcd. Kubernetes will validate it, but it won't act on it. To give a CR behavior – like spinning up pods or scheduling jobs – you need to implement a Custom Controller. The controller watches for CR events, compares desired state in `.spec` with actual cluster state, and takes corrective action. The CR declares intent; the controller enforces it."

### Q: What is the control loop pattern?

> "The control loop is the heartbeat of Kubernetes automation. Every controller continuously does three things: observe the current state, compare it with the desired state defined in the spec, and take action to reconcile any difference. For Custom Resources, you write your own controller that applies this same pattern to your domain problem. This is what makes Kubernetes declarative rather than imperative."

### Q: Where do Custom Resources live in the API hierarchy?

> "Custom Resources always live under named API groups – the `/apis/<group>/<version>` path. They cannot be part of the core API group `/api`, which is reserved for built-in types like Pods and Services. So if your CRD has group `ops.mycompany.com`, your CRs become accessible at `/apis/ops.mycompany.com/v1/namespaces/{namespace}/backuppolicies`. The API server generates these REST endpoints dynamically when you apply the CRD."

### Q: How does built-in resource validation differ from CRD validation?

> "For built-in resources like Deployments, the schema is compiled directly into the API server binary. It's expressed as Go structs that get published as OpenAPI definitions. For CRDs, you explicitly define the schema as an OpenAPI v3 specification inside the CRD manifest under `.spec.versions[].schema.openAPIV3Schema`. Both are validated at admission time, but the mechanism differs: built-ins use native code, CRDs use a dynamic schema engine."

### Q: What happens when you delete a CRD?

> "Deleting a CRD cascades – it removes all instances of that Custom Resource from etcd. This is destructive and there's no automatic backup. In production, you should always delete all CR instances first, then delete the CRD. Use `kubectl delete backuppolicies --all -A` before `kubectl delete crd backuppolicies.ops.mycompany.com`."

### Q: What is the `.status` field and who owns it?

> "The `.status` field reflects the current observed state as determined by the controller – not the user. The user owns `.spec`, which declares intent. The controller owns `.status`, which reports reality: for example, the last time a backup ran or whether it succeeded. These are separated to prevent `kubectl apply` from accidentally overwriting controller-managed status fields. With the status subresource enabled, status updates go through a separate API endpoint."

---

## Debugging – Fast Diagnosis

### Problem: `no matches for kind "BackupPolicy"`

```
Step 1: Check if CRD exists
kubectl get crd | grep backup

Step 2: Check apiVersion in your CR matches CRD
kubectl get crd backuppolicies.ops.mycompany.com -o jsonpath='{.spec.group}'
kubectl get crd backuppolicies.ops.mycompany.com -o jsonpath='{.spec.versions[*].name}'

Step 3: Check if the version is served
kubectl get crd backuppolicies.ops.mycompany.com -o jsonpath='{.spec.versions[*].served}'
```

### Problem: CR Applied But Nothing Happens

```
Step 1: Is a controller running?
kubectl get pods -A | grep -i controller

Step 2: Check controller logs
kubectl logs -n <controller-ns> <controller-pod> --tail=50

Step 3: Check .status for errors
kubectl get backuppolicy mysql-backup -o jsonpath='{.status}'

Step 4: Check events
kubectl describe backuppolicy mysql-backup
kubectl get events --field-selector involvedObject.name=mysql-backup
```

### Problem: CRD Apply Fails

```
Check 1: Name format – must be <plural>.<group>
Check 2: apiVersion – must be apiextensions.k8s.io/v1
Check 3: storage version – exactly one version with storage: true
Check 4: Schema structure – openAPIV3Schema must start with type: object
```

### Full Diagnostic Commands

```bash
# CRD health
kubectl get crd backuppolicies.ops.mycompany.com
kubectl describe crd backuppolicies.ops.mycompany.com
kubectl get crd backuppolicies.ops.mycompany.com -o jsonpath='{.status.conditions[*].type}'

# CR inspection
kubectl get bp -A                            # All namespaces
kubectl get bp mysql-backup -o yaml          # Full object
kubectl describe bp mysql-backup             # Events + summary

# API reachability
kubectl api-resources --api-group=ops.mycompany.com
kubectl get --raw /apis/ops.mycompany.com/v1 | jq .

# Admission debug
kubectl apply -f mysql-backup.yaml --dry-run=server -v=8
```

---

## 30-Second Quick Revision

### The Essentials

```
CRD = schema registration (the blueprint)
CR  = instance of that type (the object)
Controller = makes things happen (the behavior)
Without a controller, a CR is just stored config
```

### Name Formats

```
CRD name:     <plural>.<group>     e.g., backuppolicies.ops.mycompany.com
CR apiVersion: <group>/<version>   e.g., ops.mycompany.com/v1
```

### Where They Live

```
CRs live at: /apis/<group>/<version>/namespaces/<ns>/<plural>
Core types live at: /api (Pods, Services – NOT CRs)
```

### Who Owns What

```
.spec   = user owns = desired state
.status = controller owns = actual state
```

### Version Rules

```
storage: true → exactly ONE version must have this
served: true → version is reachable via API
```

### Golden Rule

```
Deleting CRD = ALL CR instances deleted from etcd (no undo)
Always delete CRs first, then the CRD
```

---

## Appendix – Quick Reference

### CRD Manifest Skeleton

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: <plural>.<group>           # MUST match exactly
spec:
  group: <group>                    # e.g., ops.mycompany.com
  names:
    plural: <plural>                # e.g., backuppolicies
    singular: <singular>            # e.g., backuppolicy
    kind: <Kind>                    # e.g., BackupPolicy
    shortNames: [<alias>]           # e.g., bp
  scope: Namespaced | Cluster
  versions:
  - name: v1
    served: true
    storage: true
    subresources:
      status: {}
    additionalPrinterColumns:
    - name: <Col>
      type: string | integer | date | boolean
      jsonPath: .<path>
    schema:
      openAPIV3Schema:
        type: object
        required: ["spec"]
        properties:
          spec:
            type: object
            required: [<fields>]
            properties:
              <field>:
                type: string | integer | boolean | object | array
                description: "..."
                minimum: <n>
                maximum: <n>
                pattern: "..."
          status:
            type: object
            properties:
              <statusField>:
                type: string
```

### CR Manifest Skeleton

```yaml
apiVersion: <group>/<version>      # e.g., ops.mycompany.com/v1
kind: <Kind>                        # e.g., BackupPolicy
metadata:
  name: <name>
  namespace: <ns>                   # Omit for Cluster-scoped
spec:
  <field>: <value>
```

### Essential Commands

| **Task** | **Command** |
|---|---|
| Apply CRD | `kubectl apply -f crd.yaml` |
| Wait for CRD ready | `kubectl wait --for=condition=Established crd/<name>` |
| List CRDs | `kubectl get crd` |
| View CRD schema | `kubectl get crd <name> -o yaml` |
| Explore CR fields | `kubectl explain <kind>.spec` |
| List CRs | `kubectl get <plural> -A` |
| Describe CR | `kubectl describe <kind> <name>` |
| View CR YAML | `kubectl get <kind> <name> -o yaml` |
| Check status | `kubectl get <kind> <name> -o jsonpath='{.status}'` |
| Delete CR | `kubectl delete <kind> <name>` |
| Delete ALL CRs | `kubectl delete <plural> --all -A` |
| Delete CRD | `kubectl delete crd <name>` |
| Raw API list | `kubectl get --raw /apis/<group>/<version> \| jq .` |
| API groups | `kubectl api-resources --api-group=<group>` |

### OpenAPI Types

| **Kubernetes Type** | **`type:` value** | **Notes** |
|---|---|---|
| Text | `string` | Add `pattern:` for format |
| Whole number | `integer` | Add `minimum:` / `maximum:` |
| Decimal | `number` | |
| True/false | `boolean` | |
| Nested object | `object` | Add `properties:` |
| List | `array` | Add `items:` |

---
