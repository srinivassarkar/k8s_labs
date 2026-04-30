# Kubernetes CRDs & Custom Resources 

> **Scope:** Custom Resources (CR), Custom Resource Definitions (CRD), Custom Controllers, Control Loop Pattern
> **Exam Weight:** Architecture & extensibility; appears in troubleshooting and design questions
> **Prerequisite mental model:** You must understand how the Kubernetes API server surfaces resources via `/api` and `/apis` groups before this topic makes sense.

---

## 0. First Principles

> *The mental model that never changes — anchor every answer here.*

**Kubernetes is, at its core, a state reconciliation engine.**

Everything in Kubernetes — Pods, Deployments, Services — follows the same pattern:

```
Desired State (spec) → Stored in etcd → Controller watches → Reconciles → Actual State (status)
```

This pattern is not limited to built-in types. CRDs and CRs extend this exact same model to **any domain problem you can define as a schema**.

| Principle | What It Means in Practice |
|---|---|
| **API is the contract** | Any object you `kubectl apply` becomes a resource the API server knows about |
| **etcd is the source of truth** | CRs are stored in etcd exactly like built-in resources |
| **Controllers are optional but necessary for behavior** | A CR without a controller is just stored data — inert config |
| **Desired state → Actual state is the loop** | Controllers exist to close the gap between `.spec` and the real world |
| **CRDs extend the API, not the binary** | No recompilation, no API server restart — fully dynamic |

> **The CRD is the schema. The CR is an instance. The controller is the behavior.**
> This maps exactly to: class definition → object instance → runtime logic.

---

## 1. Reality Constraints

> *What Kubernetes actually does and doesn't do — hard limits that trap engineers.*

### What Kubernetes DOES for you when you apply a CRD

- Registers the new kind in the API server
- Creates REST endpoints: `GET/POST/PUT/DELETE /apis/<group>/<version>/<plural>`
- Enables full `kubectl` compatibility: `get`, `describe`, `apply`, `delete`, `watch`, `-o yaml`
- Validates CRs against the OpenAPI v3 schema you define
- Stores CRs in etcd under the new group/version/kind
- Makes the resource appear in `kubectl api-resources`

### What Kubernetes does NOT do

- **No controller is created automatically.** CRs are passive storage unless you build and deploy a controller.
- **No defaulting logic** unless you implement a Mutating Admission Webhook.
- **No business logic** — scheduling a backup, creating a Pod, uploading to S3 — none of that happens.
- **No status updates** unless your controller writes to `.status`.
- **CRs cannot join the core API group (`/api`).** They always live under `/apis/<group>`.
- **CRDs are cluster-scoped objects** even when the CR itself is `Namespaced`.

### Hard Constraints to Memorize

| Constraint | Detail |
|---|---|
| CRD name format | **Must be** `<plural>.<group>` — e.g., `backuppolicies.ops.cloudwithvarjosh` |
| API version for CRDs | `apiextensions.k8s.io/v1` (required since k8s 1.16+; `v1beta1` is removed) |
| OpenAPI schema | Required in `v1`; validation is enforced at admission time |
| `storage: true` | Exactly **one** version must be marked `storage: true` |
| `served: true` | A version must be served to accept requests; can have served=false for migration |
| Scope options | Only `Namespaced` or `Cluster` — no other values |
| `kubectl explain` | Works on CRDs if you add `description` fields in the schema |

---

## 2. Decision Logic

> *When to use CRDs vs. alternatives — the decision tree engineers actually use.*

### When to Use a CRD

```
Do you need a new Kubernetes-native object type?
│
├── YES → Does it represent desired state that can drift?
│         ├── YES → CRD + Custom Controller (or Operator)
│         └── NO  → CRD alone (config store, policy declaration)
│
└── NO  → Use ConfigMap / Secret / existing resource with labels
```

### CRD vs. ConfigMap vs. Aggregated API Server

| Criteria | CRD | ConfigMap | Aggregated API Server |
|---|---|---|---|
| Schema validation | ✅ OpenAPI v3 | ❌ Free-form | ✅ Full control |
| `kubectl` native | ✅ Full verbs | ✅ Full verbs | ✅ Full verbs |
| RBAC per-kind | ✅ | ✅ (but only on ConfigMap) | ✅ |
| Custom status subresource | ✅ | ❌ | ✅ |
| Custom column display | ✅ `additionalPrinterColumns` | ❌ | ✅ |
| Complexity to implement | Low | None | Very High |
| Use case | Platform abstractions, operators | App config, flags | Complex protocol needs |

### BackupPolicy CRD vs. Native CronJob

| Feature | CronJob | BackupPolicy CRD + Controller |
|---|---|---|
| Schedule execution | ✅ Built-in | ✅ Controller-driven |
| PVC awareness | ❌ Manual volumeMounts | ✅ Declarative field |
| Retention policy | ❌ Not supported | ✅ `retentionDays` field |
| Backup validation hooks | ❌ External scripts | ✅ Encoded in controller |
| Declarative config store | ❌ Procedural Job YAML | ✅ CR is the config unit |
| `.status` reporting | Limited | ✅ Controller writes status |

---

## 3. Internal Working

> *What actually happens under the hood — step by step.*

### Phase 1: CRD Registration

```
kubectl apply -f backup-policy-crd.yaml
        │
        ▼
API Server receives POST to /apis/apiextensions.k8s.io/v1/customresourcedefinitions
        │
        ▼
API Server validates the CRD manifest (meta-schema validation)
        │
        ▼
CRD object stored in etcd
        │
        ▼
CRD controller (inside kube-controller-manager) detects new CRD
        │
        ▼
API server dynamically registers new REST endpoints:
  /apis/ops.cloudwithvarjosh/v1/namespaces/{ns}/backuppolicies
  /apis/ops.cloudwithvarjosh/v1/backuppolicies  (cluster-scoped list)
        │
        ▼
`kubectl api-resources` now shows BackupPolicy
```

### Phase 2: CR Creation

```
kubectl apply -f mysql-backup.yaml
        │
        ▼
API Server receives POST to /apis/ops.cloudwithvarjosh/v1/namespaces/default/backuppolicies
        │
        ▼
Admission Controllers run (MutatingWebhook → ValidatingWebhook)
        │
        ▼
OpenAPI v3 schema validation (enforced by API server using CRD schema)
  - schedule: must be string ✅
  - retentionDays: must be integer ✅
  - unknown fields: rejected if x-kubernetes-preserve-unknown-fields is not set
        │
        ▼
Object written to etcd with GVK: ops.cloudwithvarjosh/v1/BackupPolicy
        │
        ▼
Custom Controller (if deployed) receives watch event via informer
        │
        ▼
Controller reconcile loop runs → creates CronJob / updates .status
```

### The Control Loop — Internal Mechanics

```go
// Simplified reconcile loop (what a controller does internally)
for {
    // 1. Pull desired state from API cache (informer/lister)
    backupPolicy := lister.BackupPolicies(namespace).Get(name)

    // 2. Determine actual state
    existingCronJob := getExistingCronJob(backupPolicy.Spec.TargetPVC)

    // 3. Reconcile
    if existingCronJob == nil {
        createCronJob(backupPolicy.Spec)
    } else if cronJobNeedsUpdate(existingCronJob, backupPolicy.Spec) {
        updateCronJob(existingCronJob, backupPolicy.Spec)
    }

    // 4. Update .status to reflect what happened
    backupPolicy.Status.LastReconciled = time.Now()
    client.Status().Update(ctx, backupPolicy)
}
```

### .spec vs .status — The Separation of Concerns

| Field | Owner | Purpose |
|---|---|---|
| `.spec` | User / GitOps | Declares **what should happen** |
| `.status` | Controller | Reflects **what actually happened** |
| `.metadata` | API server | Identity, labels, resourceVersion |

> **Critical:** Controllers should never modify `.spec`. They own `.status` only. This is why `status` is typically a separate subresource — `kubectl apply` cannot accidentally overwrite controller-managed status.

### Built-in Resources vs. CRDs — How Definitions Differ

| Aspect | Built-in (e.g., Deployment) | Custom (CRD) |
|---|---|---|
| Schema defined | Compiled into API server binary | Applied as CRD YAML to the cluster |
| Schema location | OpenAPI spec at `/openapi/v2` | CRD `.spec.versions[].schema` |
| Controller | Ships with `kube-controller-manager` | Must be deployed separately |
| `kubectl explain` | Works natively | Works if `description` fields present |

---

## 4. Hands-On

> *Production-quality YAML and commands — nothing simplified.*

### CRD Definition — Full Production Quality

```yaml
# backup-policy-crd.yaml
apiVersion: apiextensions.k8s.io/v1            # Required since k8s 1.16
kind: CustomResourceDefinition

metadata:
  name: backuppolicies.ops.cloudwithvarjosh    # MUST be <plural>.<group>

spec:
  group: ops.cloudwithvarjosh                  # API group for apiVersion field in CRs

  names:
    plural: backuppolicies                     # URL path: /apis/.../backuppolicies
    singular: backuppolicy                     # Used in CLI output messages
    kind: BackupPolicy                         # PascalCase; used as `kind:` in CR manifests
    shortNames:
      - bp                                     # kubectl get bp

  scope: Namespaced                            # Namespaced | Cluster

  versions:
    - name: v1
      served: true                             # Expose via API
      storage: true                            # Only ONE version can be storage: true
      
      # Enables separate status updates (controller can update status without
      # conflicting with user updates to spec)
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
              required: ["schedule", "retentionDays", "targetPVC"]   # Enforce required fields
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
apiVersion: ops.cloudwithvarjosh/v1    # <group>/<version> from CRD
kind: BackupPolicy                     # kind from CRD spec.names.kind

metadata:
  name: mysql-backup
  namespace: default
  labels:
    app: mysql
    team: platform

spec:
  schedule: "0 1 * * *"       # Daily at 1:00 AM UTC
  retentionDays: 7
  targetPVC: mysql-data-pvc
```

### Full Lifecycle Commands

```bash
# --- CRD Management ---

# Apply CRD (registers the new type)
kubectl apply -f backup-policy-crd.yaml

# Verify CRD exists
kubectl get crd backuppolicies.ops.cloudwithvarjosh

# Inspect CRD definition
kubectl describe crd backuppolicies.ops.cloudwithvarjosh

# Confirm API endpoint is live
kubectl api-resources | grep backup
kubectl api-resources --api-group=ops.cloudwithvarjosh

# Verify via raw API
kubectl get --raw /apis/ops.cloudwithvarjosh/v1 | jq .

# --- CR Management ---

# Create CR
kubectl apply -f mysql-backup.yaml

# List CRs (short name works after CRD is applied)
kubectl get backuppolicies
kubectl get bp
kubectl get bp -n default

# Inspect CR
kubectl describe backuppolicy mysql-backup
kubectl get backuppolicy mysql-backup -o yaml

# Check .status written by a controller
kubectl get backuppolicy mysql-backup -o jsonpath='{.status}'

# Watch for changes
kubectl get bp -w

# Edit live
kubectl edit backuppolicy mysql-backup

# --- Cleanup ---
kubectl delete -f mysql-backup.yaml
kubectl delete -f backup-policy-crd.yaml

# WARNING: Deleting a CRD deletes ALL instances of that CR in etcd
# Always delete CR instances before deleting the CRD in production

# --- Exploration ---

# Explore built-in resource schema (equivalent of CRD for built-ins)
kubectl explain deployment.spec
kubectl explain backuppolicy.spec          # Works for CRDs too if descriptions are set
kubectl get --raw /openapi/v2 | jq .       # Full OpenAPI schema for all resources
```

---

## 5. Production Flow

> *Real-world architecture and design patterns.*

### Operator Pattern — CRD + Controller Together

```
┌─────────────────────────────────────────────────────────┐
│                    KUBERNETES CLUSTER                   │
│                                                         │
│  User/GitOps                                            │
│       │                                                 │
│       │  kubectl apply -f mysql-backup.yaml             │
│       ▼                                                 │
│  ┌──────────┐    stored in    ┌──────────┐              │
│  │BackupPolicy│◄──────────────│   etcd   │              │
│  │   (CR)   │                └──────────┘              │
│  └──────────┘                      ▲                   │
│       │                            │                   │
│       │  watch event               │ write .status      │
│       ▼                            │                   │
│  ┌───────────────────────────────────────────┐         │
│  │          Custom Controller (Pod)          │         │
│  │                                           │         │
│  │  Informer → Work Queue → Reconcile Loop   │         │
│  │         ↓                                 │         │
│  │  Creates/Updates CronJob                  │         │
│  │  Enforces retentionDays                   │         │
│  │  Writes status.lastBackupTime             │         │
│  └───────────────────────────────────────────┘         │
│       │                                                 │
│       ▼                                                 │
│  ┌──────────┐                                           │
│  │ CronJob  │──► Job ──► Pod ──► uploads to S3         │
│  └──────────┘                                           │
└─────────────────────────────────────────────────────────┘
```

### Real Production CRD Use Cases

| Product | CRD Kind | What It Manages |
|---|---|---|
| Velero | `Backup`, `Restore`, `Schedule` | Cluster backup & restore |
| cert-manager | `Certificate`, `Issuer`, `ClusterIssuer` | TLS certificate lifecycle |
| Prometheus Operator | `ServiceMonitor`, `PrometheusRule` | Monitoring config |
| Argo CD | `Application`, `AppProject` | GitOps deployments |
| Istio | `VirtualService`, `DestinationRule` | Service mesh routing |
| KEDA | `ScaledObject`, `TriggerAuthentication` | Event-driven autoscaling |

### RBAC for CRDs — Production Pattern

```yaml
# Restrict who can create BackupPolicies
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: backuppolicy-admin
rules:
  - apiGroups: ["ops.cloudwithvarjosh"]
    resources: ["backuppolicies"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["ops.cloudwithvarjosh"]
    resources: ["backuppolicies/status"]
    verbs: ["get", "update", "patch"]    # Only controller should patch status
```

---

## 6. Mistakes

> *What actually breaks in real systems — root cause and fix.*

### Mistake 1: Deleting CRD Before Deleting CR Instances

**Root cause:** Kubernetes garbage-collects all CR instances when the CRD is deleted. Data is gone from etcd.

**Symptom:** `kubectl get backuppolicy` returns "No resources found" after CRD deletion.

**Fix:**
```bash
# Always drain instances first
kubectl delete backuppolicies --all -A
# Then delete CRD
kubectl delete crd backuppolicies.ops.cloudwithvarjosh
```

---

### Mistake 2: Wrong CRD Name Format

**Root cause:** CRD `.metadata.name` must exactly match `<plural>.<group>`.

**Symptom:** `kubectl apply` fails with:
```
The CustomResourceDefinition "backup.ops" is invalid: metadata.name: Invalid value
```

**Fix:** Ensure `metadata.name` = `backuppolicies.ops.cloudwithvarjosh` (plural.group, not singular.group)

---

### Mistake 3: Missing `required` Fields Not Caught

**Root cause:** OpenAPI schema without `required: [...]` array — Kubernetes will accept CRs missing critical fields.

**Symptom:** CRs created without `schedule` field; controller crashes on nil pointer.

**Fix:** Always define `required` at the `spec` object level in the schema:
```yaml
spec:
  type: object
  required: ["schedule", "retentionDays", "targetPVC"]
```

---

### Mistake 4: Controller Updates `.spec` Instead of `.status`

**Root cause:** Controller uses the wrong client method and patches `.spec` fields, overwriting user intent.

**Symptom:** User changes to `.spec` are silently reverted on every reconcile cycle.

**Fix:** Enable the `status` subresource in CRD and use `client.Status().Update()` — not `client.Update()` — for status writes.

---

### Mistake 5: CR Created Before CRD Exists

**Root cause:** Applying CR manifest before CRD is registered.

**Symptom:**
```
error: unable to recognize "mysql-backup.yaml": no matches for kind "BackupPolicy"
in version "ops.cloudwithvarjosh/v1"
```

**Fix:** Apply CRD first, wait for it to be established:
```bash
kubectl apply -f backup-policy-crd.yaml
kubectl wait --for=condition=Established crd/backuppolicies.ops.cloudwithvarjosh
kubectl apply -f mysql-backup.yaml
```

---

### Mistake 6: Two Versions Both Marked `storage: true`

**Root cause:** Only one version can be the storage version in etcd.

**Symptom:** CRD apply fails with validation error about multiple storage versions.

**Fix:** Exactly one version gets `storage: true`; others get `storage: false`.

---

## 7. Interview Answers

> *Compressed, verbatim-ready answers for common questions.*

---

**Q: What is the difference between a CRD and a Custom Resource?**

"A CRD — CustomResourceDefinition — is the schema registration. It tells the Kubernetes API server that a new resource type exists, what its API group and version are, and what fields are valid. Think of it like a class definition. A Custom Resource is an instance of that type — an actual object you create using kubectl apply that conforms to the schema. The CRD is the blueprint; the CR is the object. Once you apply a CRD, the API server dynamically starts serving REST endpoints for that new type, and you can create, read, update, and delete CRs just like any built-in resource."

---

**Q: Can a Custom Resource do anything by itself?**

"By itself, a Custom Resource is passive — it's just data stored in etcd. Kubernetes will validate it against the OpenAPI schema defined in the CRD, but it won't act on it. To give a CR behavior — like spinning up pods, scheduling jobs, or uploading to S3 — you need to implement a Custom Controller. The controller runs a control loop where it watches for CR events, compares the desired state in `.spec` with the actual cluster state, and takes corrective action. The CR declares intent; the controller enforces it."

---

**Q: What is the control loop pattern and why does it matter?**

"The control loop is the heartbeat of Kubernetes automation. Every controller — built-in or custom — continuously does three things: observe the current state of the cluster, compare it with the desired state defined in a resource's spec, and take action to reconcile any difference. This is exactly how the Deployment controller ensures the right number of replicas is always running. For Custom Resources, you write your own controller that applies this same pattern to your domain problem. It's what makes Kubernetes declarative rather than imperative."

---

**Q: Where do Custom Resources live in the Kubernetes API hierarchy?**

"Custom Resources always live under the named API groups — the `/apis/<group>/<version>` path. They cannot be part of the core API group, which is the `/api` path reserved for original built-in types like Pods and Services. So when you define a CRD with group `ops.cloudwithvarjosh`, your CRs become accessible at `/apis/ops.cloudwithvarjosh/v1/namespaces/{namespace}/backuppolicies`. The API server generates these REST endpoints dynamically when you apply the CRD."

---

**Q: How does built-in resource validation differ from CRD validation?**

"For built-in resources like Deployments, the schema is compiled directly into the API server binary. It's not a separate file you can read — it's expressed as Go structs that get published as OpenAPI definitions you can query at `/openapi/v2`. For CRDs, you explicitly define the schema as an OpenAPI v3 specification inside the CRD manifest itself, under `.spec.versions[].schema.openAPIV3Schema`. Both are validated at admission time by the API server, but the mechanism differs: built-ins use native code, CRDs use a dynamic schema engine."

---

**Q: What happens when you delete a CRD?**

"Deleting a CRD cascades — it removes all instances of that Custom Resource from etcd. This is a destructive operation and there is no automatic backup. In production you should always delete all CR instances first, then delete the CRD. A good practice is to protect CRDs with a finalizer or use namespace-level RBAC to prevent accidental deletion. You can also wait for graceful cleanup using `kubectl wait`."

---

**Q: What is the `.status` field and who owns it?**

"The `.status` field reflects the current observed state of a resource as determined by the controller — not the user. The user owns `.spec`, which declares intent. The controller owns `.status`, which reports reality: for example, the last time a backup ran, whether it succeeded, or how many replicas are actually available. The reason these are separated, and why you enable the `status` subresource in a CRD, is to prevent `kubectl apply` from accidentally overwriting controller-managed status fields. With the subresource enabled, status updates go through a separate API endpoint and can only be written by callers with explicit `update` verb on `backuppolicies/status`."

---

## 8. Debugging

> *Fast diagnosis paths — commands and decision trees.*

### Problem: `no matches for kind "BackupPolicy"`

```
Symptom: kubectl apply gives "no matches for kind"

Step 1: Check if CRD exists
  kubectl get crd | grep backup
  → Not found → Apply CRD first, wait for Established condition
  → Found → Continue

Step 2: Check apiVersion in your CR matches CRD group/version
  kubectl get crd backuppolicies.ops.cloudwithvarjosh -o jsonpath='{.spec.group}'
  kubectl get crd backuppolicies.ops.cloudwithvarjosh -o jsonpath='{.spec.versions[*].name}'
  → CR apiVersion must be <group>/<version>

Step 3: Check if the version is served
  kubectl get crd backuppolicies.ops.cloudwithvarjosh -o jsonpath='{.spec.versions[*].served}'
  → Must be true
```

### Problem: CR Applied But Nothing Happens

```
Symptom: kubectl get bp shows object but no pods/jobs created

Step 1: Is a controller running?
  kubectl get pods -A | grep -i controller
  → No controller → You need to deploy one; CRDs are passive without controllers

Step 2: Check controller logs
  kubectl logs -n <controller-ns> <controller-pod> --tail=50

Step 3: Check .status field for controller errors
  kubectl get backuppolicy mysql-backup -o jsonpath='{.status}'

Step 4: Check events
  kubectl describe backuppolicy mysql-backup
  kubectl get events --field-selector involvedObject.name=mysql-backup
```

### Problem: CRD Apply Fails

```
Symptom: kubectl apply -f crd.yaml returns validation error

Check 1: CRD name format
  → metadata.name must be <plural>.<group>
  → Wrong: backuppolicy.ops.cloudwithvarjosh (singular used)
  → Right: backuppolicies.ops.cloudwithvarjosh

Check 2: apiVersion
  → Must be apiextensions.k8s.io/v1 (not v1beta1 — removed in 1.22)

Check 3: storage version
  → Exactly one version must have storage: true

Check 4: Schema structure
  → openAPIV3Schema must start with `type: object`
  → properties must be nested correctly under spec/status
```

### Problem: CR Fails Validation on Apply

```
Symptom: kubectl apply -f cr.yaml → "spec.retentionDays: Invalid value"

Step 1: Check what the CRD schema requires
  kubectl get crd backuppolicies.ops.cloudwithvarjosh -o jsonpath='{.spec.versions[0].schema}' | jq .

Step 2: Check required fields
  → Is the field marked required in the schema?
  → Is the type correct (string vs integer)?

Step 3: Test with explain
  kubectl explain backuppolicy.spec
  kubectl explain backuppolicy.spec.retentionDays
```

### Full Diagnostic Command Set

```bash
# CRD health
kubectl get crd backuppolicies.ops.cloudwithvarjosh
kubectl describe crd backuppolicies.ops.cloudwithvarjosh
kubectl get crd backuppolicies.ops.cloudwithvarjosh -o jsonpath='{.status.conditions[*].type}'

# CR inspection
kubectl get bp -A                              # all namespaces
kubectl get bp mysql-backup -o yaml            # full object including status
kubectl describe bp mysql-backup               # events + field summary

# API reachability
kubectl api-resources --api-group=ops.cloudwithvarjosh
kubectl get --raw /apis/ops.cloudwithvarjosh/v1 | jq .

# Admission debug (if CR is rejected at apply)
kubectl apply -f mysql-backup.yaml --dry-run=server -v=8

# Controller debug
kubectl logs -n <ns> <controller-pod> -f
kubectl get events -n default --sort-by=.lastTimestamp
```

---

## 9. Kill Switch

> *10-second recall — the absolute minimum to hold in memory.*

```
CRD  = schema registration → tells API server a new type exists
CR   = an instance of that type → stored in etcd like any resource
Controller = the loop that watches CR and makes things happen
Without a controller, a CR is just inert config

CRD name format:   <plural>.<group>       e.g. backuppolicies.ops.example.com
CRD apiVersion:    apiextensions.k8s.io/v1
CR  apiVersion:    <group>/<version>      e.g. ops.example.com/v1

CR lives at:       /apis/<group>/<version>/namespaces/<ns>/<plural>
Core types live at: /api  (Pods, Services — NOT CRs)

.spec  = user owns  = desired state
.status = controller owns = actual state

storage: true → exactly ONE version must have this
served: true  → version is reachable via API

Deleting CRD = ALL CR instances deleted from etcd (no undo)
```

---

## 10. Appendix

> *Quick reference card — commands, formats, cheatsheets.*

### CRD Manifest Skeleton

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: <plural>.<group>                         # MUST match exactly
spec:
  group: <group>                                 # e.g. ops.example.com
  names:
    plural: <plural>                             # e.g. widgets
    singular: <singular>                         # e.g. widget
    kind: <Kind>                                 # e.g. Widget (PascalCase)
    shortNames: [<alias>]                        # e.g. wg
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
                  minimum: <n>       # integers only
                  maximum: <n>
                  pattern: "..."     # strings only (regex)
            status:
              type: object
              properties:
                <statusField>:
                  type: string
```

### CR Manifest Skeleton

```yaml
apiVersion: <group>/<version>    # e.g. ops.example.com/v1
kind: <Kind>                     # e.g. Widget
metadata:
  name: <name>
  namespace: <ns>               # omit for Cluster-scoped CRDs
spec:
  <field>: <value>
```

### Essential Commands Cheatsheet

| Task | Command |
|---|---|
| Apply CRD | `kubectl apply -f crd.yaml` |
| Wait for CRD ready | `kubectl wait --for=condition=Established crd/<name>` |
| List all CRDs | `kubectl get crd` |
| View CRD schema | `kubectl get crd <name> -o yaml` |
| Explore CR fields | `kubectl explain <kind>.spec` |
| List CRs (all ns) | `kubectl get <plural> -A` |
| List CRs (short) | `kubectl get <shortname>` |
| Describe CR | `kubectl describe <kind> <name>` |
| View CR YAML | `kubectl get <kind> <name> -o yaml` |
| Check status | `kubectl get <kind> <name> -o jsonpath='{.status}'` |
| Delete CR | `kubectl delete <kind> <name>` |
| Delete ALL CRs | `kubectl delete <plural> --all -A` |
| Delete CRD | `kubectl delete crd <name>` |
| Raw API list | `kubectl get --raw /apis/<group>/<version> \| jq .` |
| Check API groups | `kubectl api-resources --api-group=<group>` |
| Dry-run server-side | `kubectl apply -f cr.yaml --dry-run=server` |

### OpenAPI Types Quick Reference

| Kubernetes Field Type | `type:` value | Notes |
|---|---|---|
| Free text | `string` | Add `pattern:` for format enforcement |
| Whole number | `integer` | Add `minimum:` / `maximum:` |
| Decimal | `number` | |
| True/false | `boolean` | |
| Nested object | `object` | Add `properties:` block |
| List | `array` | Add `items:` block with type |

### Controller Frameworks (Not in CKA Exam, But Useful for Interviews)

| Framework | Language | Use Case |
|---|---|---|
| Kubebuilder | Go | Full operator scaffolding, recommended |
| controller-runtime | Go | Lower-level library Kubebuilder uses |
| Operator SDK | Go / Ansible / Helm | OLM-compatible operators |
| Java Operator SDK | Java | JVM shops |
| kopf | Python | Lightweight scripts |

### Resources That Have Controllers vs. Don't

| Has Controller | No Controller |
|---|---|
| Deployment, ReplicaSet, StatefulSet | Pod (standalone) |
| DaemonSet, Job, CronJob | ConfigMap, Secret |
| HPA, VPA | Namespace |
| Endpoints (via Service) | ServiceAccount |
| PersistentVolume (provisioner) | LimitRange, ResourceQuota |
