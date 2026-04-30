# Kubernetes Kustomize 

> **Scope**: Configuration management via Kustomize — base/overlay architecture, transformers, patching strategies, migration, and production patterns.

---

## 0. First Principles

> These are the truths that never change regardless of Kustomize version, cluster version, or environment.

**Kustomize exists to answer one question: how do you deploy the same application differently across environments without duplicating YAML?**

The answer is a two-layer model:

1. **Base** — what never changes (the application shape)
2. **Overlay** — what does change (the environment-specific delta)

Everything else in Kustomize is a mechanism that serves this model.

| Principle | What It Means |
|---|---|
| **DRY over duplication** | One base, N overlays. Never copy a full manifest to change one field. |
| **Plain YAML, always** | No templating language. No variables. No conditionals. Just YAML + patches. |
| **Overlay declares intent** | An overlay never describes the full resource — only the delta from base. |
| **Kustomize is a build tool** | It renders final manifests. It does not apply, track, or version them. |
| **Transformers before patches** | Transformers run first, patches run after. Patches always win conflicts. |
| **`kustomization.yaml` is the entry point** | Every directory Kustomize touches must contain exactly this file, exact name. |

---

## 1. Reality Constraints

> What Kubernetes and Kustomize actually enforce — the hard edges.

### What Kustomize IS

- A **manifest generator** — it outputs rendered YAML
- Built into `kubectl` since v1.14 (`kubectl apply -k`)
- Also available as a standalone binary (always newer than what's bundled in kubectl)
- A **GitOps-friendly** tool — output is deterministic and diffable

### What Kustomize IS NOT

- Not a package manager (no charts, no versions, no release tracking)
- Not a templating engine (no `{{ .Values.image }}`)
- Not a deployment tracker (no rollback, no hooks, no state)
- Not Helm (they solve adjacent but different problems)

### Hard Constraints You Will Hit

| Constraint | Detail |
|---|---|
| File must be named `kustomization.yaml` | Exact lowercase. `kustomization.yml`, `Kustomization.yaml` — all fail with `Error: no kustomization file found` |
| Paths are relative | `../../base` means two levels up from the overlay file's directory |
| `replace` on non-existent path is unsafe | Use `add` when the field doesn't exist in base. `replace` may silently fail or error depending on resource type. |
| `patchesJson6902` and `patchesStrategicMerge` are deprecated | Still processed, but emit warnings. Will eventually be removed. Migrate with `kustomize edit fix`. |
| `commonLabels` is deprecated | Use `labels:` with `includeSelectors: true` instead |
| Transformers cannot touch arbitrary nested fields | `images`, `replicas`, `namespace`, etc. — fixed scope. Anything inside `spec.template.spec.containers[]` needs a patch. |
| Kustomize runs entirely client-side | No server interaction during build. Applied resources still go through admission controllers. |

### Transformer Scope Limits — What You Cannot Do With Transformers Alone

```
✅ Transformers CAN:          ❌ Transformers CANNOT:
  - Change image tag             - Add env vars to containers
  - Set namespace                - Set resource limits/requests
  - Add labels/annotations       - Modify containerPort
  - Change replica count         - Remove a specific label key
  - Add name prefix/suffix       - Add volume mounts or probes
                                 - Modify any arbitrary nested field
```

---

## 2. Decision Logic

> When to use what. No ambiguity.

### Should I Use Kustomize or Helm?

```
Is the app distributed externally or consumed from a chart registry?
  YES → Helm (charts have versioning, dependency management)
  NO  → Is the configuration diff across environments small and structural?
          YES → Kustomize (simpler, native, no new syntax)
          NO  → Is complex logic, conditionals, or loops required?
                  YES → Helm
                  NO  → Kustomize
```

### Which Patch Type Should I Use?

```
Is this a new project or modernizing an existing one?
  NEW → Use patches: (generic form, inline)   ← always preferred
  EXISTING with patchesJson6902 → Run: kustomize edit fix
  EXISTING with patchesStrategicMerge → Run: kustomize edit fix

Do you need to reference an external patch file?
  YES → patches: with path: field (still modern syntax)
  NO  → patches: with inline patch: |- block

Is the field you're modifying deeply nested (env, resources, ports)?
  YES → patches: with RFC 6902 (add/replace/remove)
  NO  → A transformer may be sufficient (check transformer scope above)
```

### Transformer vs Patch Decision Table

| What You Want to Change | Use Transformer | Use Patch |
|---|---|---|
| Image tag | ✅ `images:` | Can override with patch |
| Replica count | ✅ `replicas:` | Can override with patch |
| Namespace | ✅ `namespace:` | Rarely needed |
| Name prefix/suffix | ✅ `namePrefix`/`nameSuffix` | No |
| Labels on all resources | ✅ `labels:` | No |
| Annotations on all resources | ✅ `commonAnnotations:` | No |
| Add env var to container | ❌ | ✅ `patches:` (op: add) |
| Set resource limits | ❌ | ✅ `patches:` (op: add) |
| Remove a specific label key | ❌ | ✅ `patches:` (op: remove) |
| Set a specific nodePort | ❌ | ✅ `patches:` (op: replace) |
| Add volume mount | ❌ | ✅ `patches:` (op: add) |
| Inject sidecar container | ❌ | ✅ `patches:` (op: add) |

### RFC 6902 Operation Selector

| Scenario | Correct Op | Wrong Op |
|---|---|---|
| Field exists in base, change value | `replace` | `add` (will error) |
| Field does NOT exist in base | `add` | `replace` (unsafe/may error) |
| Remove a field entirely | `remove` | N/A |
| Validate before patching | `test` | N/A |

---

## 3. Internal Working

> How Kustomize actually builds the final manifest — step by step under the hood.

### Build Pipeline (Ordered)

```
kubectl apply -k overlays/dev
         │
         ▼
[1] PARSE kustomization.yaml in overlays/dev/
         │
         ▼
[2] LOAD resources: (resolve relative paths, load base kustomization.yaml recursively)
         │
         ▼
[3] LOAD base resources: (deploy.yaml, svc.yaml, etc.) → build in-memory resource map
         │
         ▼
[4] RUN TRANSFORMERS (in this order):
     - namePrefix / nameSuffix
     - namespace
     - labels / commonLabels
     - commonAnnotations
     - images
     - replicas
         │
         ▼
[5] RUN PATCHES (patches:, patchesJson6902, patchesStrategicMerge)
     - Target resources are matched by: group, version, kind, name, namespace
     - RFC 6902 ops applied sequentially per resource
     - Strategic merge patches applied via Kubernetes schema merge rules
         │
         ▼
[6] OUTPUT: Final YAML stream (stdout or piped to kubectl apply)
```

**Critical ordering rule**: Transformers run in step 4, patches run in step 5. This means:
- If `images:` sets `nginx:latest` and a patch sets `nginx:1.22`, the patch wins.
- This is intentional — patches are for fine-grained overrides that must supersede broad transformers.

### How `target:` Matching Works in Patches

```yaml
patches:
  - target:
      group: apps       # API group (omit for core group: v1 resources like Service)
      version: v1       # API version
      kind: Deployment  # Resource kind
      name: nginx-deploy # metadata.name (before prefix/suffix are applied)
```

> **Important**: `name` in the target matches against the **base name**, not the final transformed name. If base has `nginx-deploy` and overlay adds `namePrefix: app1-`, the target still uses `name: nginx-deploy`.

### How Strategic Merge Patches Work

Strategic merge uses Kubernetes schema awareness. For lists (like `containers[]`), it merges by the **merge key** (`name` for containers). This means:

```yaml
# patch-container.yaml
spec:
  template:
    spec:
      containers:
        - name: nginx       # ← merge key: finds the container named "nginx"
          env:
            - name: LOG_LEVEL
              value: debug   # ← merges/adds this env var
```

It does NOT replace the entire `containers[]` list — it finds the matching entry and merges fields. This is the "strategic" part. JSON Patch (RFC 6902) has no such awareness — it operates on raw paths.

### Path Syntax in RFC 6902

```
/spec/template/spec/containers/0/env
      │                        │
      │                        └─ index 0 = first container (zero-based)
      └─ standard JSON Pointer syntax (RFC 6901)

/metadata/labels/app           ← dots replaced with /
/spec/ports/0/nodePort         ← array index access
```

---

## 4. Hands-On

> Production-quality YAML and commands. Nothing simplified.

### Complete Directory Structure

```
app1/
├── base/
│   ├── deploy.yaml
│   ├── svc.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── patch-container.yaml      # only if using file-based patches
    ├── stage/
    │   └── kustomization.yaml
    └── prod/
        └── kustomization.yaml
```

### base/deploy.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  labels:
    app: nginx
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
          image: nginx
          ports:
            - containerPort: 80
```

### base/svc.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

### base/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deploy.yaml
  - svc.yaml
```

### overlays/dev/kustomization.yaml — Full Production Example (Modern `patches:`)

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

namePrefix: app1-
nameSuffix: -dev

namespace: dev

labels:
  - pairs:
      env: dev
    includeSelectors: true  # propagates label to selector AND pod template

commonAnnotations:
  branch: dev
  support: 800-800-700
  managed-by: kustomize

images:
  - name: nginx
    newName: nginx
    newTag: "latest"

replicas:
  - name: nginx-deploy
    count: 2

patches:
  # Patch 1: Inject env var, set resource limits, remove stale label
  - target:
      group: apps
      version: v1
      kind: Deployment
      name: nginx-deploy
    patch: |-
      - op: add
        path: /spec/template/spec/containers/0/env
        value:
          - name: LOG_LEVEL
            value: debug
          - name: APP_ENV
            value: development
      - op: add
        path: /spec/template/spec/containers/0/resources
        value:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
      - op: remove
        path: /metadata/labels/unused

  # Patch 2: Harden NodePort to a known value
  - target:
      version: v1
      kind: Service
      name: nginx-svc
    patch: |-
      - op: replace
        path: /spec/ports/0/nodePort
        value: 31000
```

### overlays/prod/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

namePrefix: app1-
nameSuffix: -prod

namespace: prod

labels:
  - pairs:
      env: prod
    includeSelectors: true

commonAnnotations:
  branch: prod
  support: 800-800-700
  managed-by: kustomize

images:
  - name: nginx
    newName: nginx
    newTag: "1.22"   # pinned stable version in prod

replicas:
  - name: nginx-deploy
    count: 5

patches:
  - target:
      group: apps
      version: v1
      kind: Deployment
      name: nginx-deploy
    patch: |-
      - op: add
        path: /spec/template/spec/containers/0/resources
        value:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "2000m"
            memory: "1Gi"
```

### File-Based Patch (Modern `patches:` with `path:`)

```yaml
# overlays/dev/kustomization.yaml (excerpt)
patches:
  - path: patch-container.yaml
    target:
      group: apps
      kind: Deployment
      name: nginx-deploy
      version: v1
```

```yaml
# overlays/dev/patch-container.yaml
- op: replace
  path: /spec/template/spec/containers/0/image
  value: nginx:1.22
- op: add
  path: /spec/template/spec/containers/0/env
  value:
    - name: LOG_LEVEL
      value: debug
```

### Strategic Merge Patch (File-Based — Modern `patches:`)

```yaml
# overlays/dev/patch-container.yaml (strategic merge style)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
spec:
  template:
    spec:
      containers:
        - name: nginx
          env:
            - name: LOG_LEVEL
              value: debug
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
```

### Essential Commands

```bash
# Install latest standalone kustomize (always newer than kubectl-bundled version)
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash

# Render manifests without applying (dry run)
kubectl kustomize overlays/dev
kustomize build overlays/dev

# Apply rendered manifests
kubectl apply -k overlays/dev
kubectl apply -k overlays/stage
kubectl apply -k overlays/prod

# Render from current directory
cd overlays/dev && kubectl kustomize .

# Apply base directly (no overlays)
kubectl kustomize base
kubectl apply -k base

# Migrate deprecated patch fields to modern syntax
cd overlays/dev && kustomize edit fix

# Create namespace before applying (namespaces are not auto-created)
kubectl create namespace dev
kubectl create namespace stage
kubectl create namespace prod

# Diff what would change (GitOps-friendly)
kubectl diff -k overlays/prod

# Delete resources managed by an overlay
kubectl delete -k overlays/dev

# View the final generated YAML (pipe to file)
kustomize build overlays/prod > prod-manifests.yaml
```

---

## 5. Production Flow

> Real-world architecture and design patterns.

### Multi-App, Multi-Environment Repository Structure

```
infra/
├── apps/
│   ├── frontend/
│   │   ├── base/
│   │   │   ├── deploy.yaml
│   │   │   ├── svc.yaml
│   │   │   ├── hpa.yaml
│   │   │   └── kustomization.yaml
│   │   └── overlays/
│   │       ├── dev/kustomization.yaml
│   │       ├── stage/kustomization.yaml
│   │       └── prod/kustomization.yaml
│   ├── backend/
│   │   ├── base/
│   │   └── overlays/
│   └── payment/
│       ├── base/
│       └── overlays/
└── platform/
    ├── monitoring/
    └── ingress/
```

### GitOps Integration (ArgoCD / Flux Pattern)

```
Git Repository
     │
     ▼ (push to main)
CI Pipeline
     │ kustomize build overlays/prod | kubectl diff -
     │ (gate: no diff = no deploy)
     ▼
ArgoCD Application (points to overlays/prod)
     │
     ▼ kustomize build + kubectl apply
Kubernetes Cluster
```

ArgoCD natively understands Kustomize — point an Application to an overlay directory and it handles the build automatically.

### Environment Promotion Pattern

```
overlays/dev  →  overlays/stage  →  overlays/prod
  (latest tag)     (1.22 tag)         (1.20 tag, 5 replicas)
  (2 replicas)     (3 replicas)
  (debug logs)     (info logs)        (warn logs)
  (nodePort 31000) (nodePort 31001)   (ClusterIP, behind LB)
```

Each overlay only declares its delta. The base is inherited untouched.

### When to Break Out a Component into Its Own Base

Use separate bases when:
- A component is shared across multiple **applications** (not just environments)
- The component has its own release lifecycle
- Teams own different components (ownership boundary = base boundary)

### Secrets Management with Kustomize

Kustomize supports `secretGenerator:` but **never commit plaintext secrets**. Production pattern:

```yaml
# kustomization.yaml
secretGenerator:
  - name: db-secret
    envs:
      - secrets.env   # ← loaded from external secret manager at CI time
    options:
      disableNameSuffixHash: true
```

In practice, integrate with External Secrets Operator or Sealed Secrets for GitOps-safe secret management.

---

## 6. Mistakes

> What actually breaks in real systems — root cause and fix.

### Mistake 1: `replace` on a field that doesn't exist in base

**Symptom**: `kustomize build` silently produces incorrect output, or errors with `add operation does not apply: doc is missing path`

**Root cause**: RFC 6902 `replace` requires the path to exist. If the base manifest has no `resources:` block in the container spec, `replace` will fail or be ignored.

**Fix**:
```yaml
# Wrong
- op: replace
  path: /spec/template/spec/containers/0/resources
  value: ...

# Correct (field doesn't exist in base)
- op: add
  path: /spec/template/spec/containers/0/resources
  value: ...
```

---

### Mistake 2: `kustomization.yml` (wrong extension)

**Symptom**: `Error: no kustomization file found in directory`

**Root cause**: Kustomize only recognizes `kustomization.yaml` (exact lowercase, `.yaml` not `.yml`).

**Fix**: Rename the file. No workaround.

---

### Mistake 3: Wrong relative path in `resources:`

**Symptom**: `Error: accumulating resources: read .../<path>: no such file or directory`

**Root cause**: Path `../../base` is relative to the `kustomization.yaml` file's location, not where you ran the command from.

**Fix**: Verify directory depth. From `overlays/dev/`, two levels up `../..` lands at the project root, then `base` enters the base directory. Draw the tree if unsure.

---

### Mistake 4: `nameSuffix` breaks `target` name matching in patches

**Symptom**: Patch is silently not applied — no error, but resource is unchanged.

**Root cause**: The `name:` in a patch `target:` must match the **pre-transformation** name (the name in base), not the final suffixed name.

**Fix**:
```yaml
patches:
  - target:
      name: nginx-deploy   # ← base name, NOT "app1-nginx-deploy-dev"
```

---

### Mistake 5: Using `commonLabels` instead of `labels`

**Symptom**: Deprecation warnings; label not applied to selectors correctly.

**Root cause**: `commonLabels` is deprecated. It had edge cases with immutable selector fields.

**Fix**:
```yaml
# Deprecated
commonLabels:
  env: dev

# Correct
labels:
  - pairs:
      env: dev
    includeSelectors: true
```

---

### Mistake 6: Namespace not created before `kubectl apply -k`

**Symptom**: `Error from server (NotFound): namespaces "dev" not found`

**Root cause**: `namespace:` in Kustomize assigns namespace to resources but does NOT create the namespace object.

**Fix**:
```bash
kubectl create namespace dev
kubectl apply -k overlays/dev
# OR add a namespace.yaml to base/resources:
```

---

### Mistake 7: Using `patchesJson6902` with kubectl's bundled kustomize

**Symptom**: Deprecation warning during apply; future breakage risk.

**Root cause**: kubectl bundles an older kustomize version. The deprecated fields still work but will eventually be removed.

**Fix**: Install standalone kustomize binary and run `kustomize edit fix` to migrate all deprecated fields.

---

### Mistake 8: Transformer and patch conflict — unexpected image tag

**Symptom**: `images:` sets `nginx:latest`, but rendered output shows `nginx:1.22`.

**Root cause**: This is actually **correct behavior** — patches run after transformers and override them. But it surprises people who forget this.

**Rule**: If you define both a transformer and a patch that target the same field, the patch always wins. Use this intentionally; avoid it accidentally.

---

## 7. Interview Answers

> Full spoken sentences, verbatim-ready.

---

**Q: What is Kustomize and why does Kubernetes have it?**

"Kustomize is a native Kubernetes configuration management tool that allows you to customize raw YAML manifests for multiple environments without duplicating files or using a templating language. The Kubernetes community built it to solve the problem of managing environment-specific differences — like image tags, replica counts, labels, and namespaces — across dev, staging, and production. Instead of maintaining three separate copies of the same Deployment, you maintain one base and three overlays, where each overlay only describes what's different. This keeps your manifests DRY, auditable, and maintainable at scale."

---

**Q: What is the difference between a transformer and a patch in Kustomize?**

"Transformers are built-in mechanisms in Kustomize that modify well-known, top-level fields across all resources in a declarative way — things like `images`, `replicas`, `namespace`, `namePrefix`, and `labels`. They run first during the build pipeline. Patches, on the other hand, let you surgically modify any field, including deeply nested ones inside container specs, using either RFC 6902 JSON Patch operations or strategic merge. Patches run after transformers, so they always take precedence. If I want to change an image tag for all containers, I use a transformer. If I want to inject an environment variable or set resource limits on a specific container, I use a patch."

---

**Q: What patching strategies does Kustomize support, and which should you use?**

"Kustomize supports three patching approaches. The first is `patches`, which is the modern, recommended format — it supports both inline patches using RFC 6902 JSON Patch operations and file-based patches. The second is `patchesJson6902`, which was the older file-based JSON Patch method — it's now deprecated and you should migrate away from it using `kustomize edit fix`. The third is `patchesStrategicMerge`, which uses partial Kubernetes manifest documents to merge changes — also deprecated. For all new work, I use `patches` exclusively, and I run `kustomize edit fix` when modernizing legacy codebases."

---

**Q: What is the difference between `add` and `replace` in JSON Patch?**

"`replace` requires the path to already exist in the document — if the field isn't present, it will error or behave unexpectedly depending on the resource type. `add` creates the field if it doesn't exist. So the rule is: if the base manifest already has the field and you want to change its value, use `replace`. If you're introducing a new field — like adding resource limits to a container spec that has no resources block — use `add`. Getting this wrong is a common source of silent patch failures in Kustomize."

---

**Q: How does Kustomize differ from Helm?**

"Helm is a package manager for Kubernetes — it uses Go templating, supports conditionals and loops in manifests, tracks deployments as releases, enables rollbacks, and can pull charts from registries. Kustomize is a simpler overlay tool — it works with plain YAML, has no templating language, and focuses purely on customizing existing manifests for different environments. I use Kustomize when I own the application and need environment-specific config management. I use Helm when consuming third-party software from a chart registry or when the application requires complex conditional logic in its manifests. They're not mutually exclusive — many teams use Helm for third-party apps and Kustomize for their own services."

---

**Q: What does `kustomize edit fix` do?**

"`kustomize edit fix` is a migration command that automatically updates a `kustomization.yaml` file to replace deprecated fields with their modern equivalents. Most commonly, it converts `patchesJson6902` and `patchesStrategicMerge` to the unified `patches:` field. You run it from inside the overlay directory — the one containing the `kustomization.yaml` you want to migrate. It preserves the patch behavior while updating the syntax to comply with current Kustomize standards."

---

**Q: In what order does Kustomize apply transformers and patches, and why does it matter?**

"Kustomize always runs transformers first and patches second. Transformers handle broad, cross-cutting changes — things like adding a namespace or updating image tags across all resources. Patches then apply on top of the transformer output, so they can override transformer results. This ordering matters because it creates a clear mental model: use transformers for defaults, use patches for exceptions or overrides. For example, I might use the `images` transformer to set `nginx:latest` across all environments as a default, and then use a patch in the prod overlay to pin it to `nginx:1.22` — the patch wins."

---

## 8. Debugging

> Fast diagnosis paths. Follow the tree.

### Problem: `kubectl apply -k` fails immediately

```
Step 1: Check if kustomization.yaml exists in the target directory
  └─ ls overlays/dev/kustomization.yaml
       NOT FOUND → create it or fix the path
       FOUND → check the filename is exactly kustomization.yaml (not .yml)

Step 2: Run kustomize build to isolate the error
  └─ kustomize build overlays/dev 2>&1
       "no kustomization file found" → wrong directory or wrong filename
       "accumulating resources" error → bad relative path in resources:
       "json: cannot unmarshal" → invalid YAML syntax in kustomization.yaml or patch

Step 3: Validate YAML syntax
  └─ cat overlays/dev/kustomization.yaml | python3 -c "import sys,yaml; yaml.safe_load(sys.stdin)"
```

### Problem: Patch silently not applied

```
Step 1: Verify target match
  └─ Does target.name match the BASE resource name (before prefix/suffix)?
       NO  → fix the name in target:
       YES → continue

Step 2: Check target group/version/kind
  └─ For Deployments: group: apps, version: v1, kind: Deployment
  └─ For Services:    group: (omit), version: v1, kind: Service
  └─ For StatefulSets: group: apps, version: v1, kind: StatefulSet

Step 3: Render and grep
  └─ kustomize build overlays/dev | grep -A5 "name: app1-nginx-deploy-dev"
       Field present with wrong value → patch target mismatch or op mismatch
       Field absent → patch not reaching resource; re-check target selectors

Step 4: Test op correctness
  └─ Does the field exist in base?
       YES → use op: replace
       NO  → use op: add
```

### Problem: Wrong image tag in output

```
Step 1: Identify all sources that could set image
  └─ kustomization.yaml: images: field
  └─ Any patch with op: replace on /spec/template/spec/containers/0/image
  └─ Any patchesStrategicMerge file referencing the container

Step 2: Remember the rule — patches always override transformers
  └─ If patch sets image, transformer setting is irrelevant — patch wins

Step 3: Verify final output
  └─ kustomize build overlays/prod | grep "image:"
```

### Problem: `namespace not found` on apply

```
Step 1: Confirm the namespace exists
  └─ kubectl get namespace dev
       NotFound → kubectl create namespace dev → retry apply
       Found    → check if kustomization.yaml namespace: field matches exactly

Step 2: Check if namespace resource is in base
  └─ If you want Kustomize to manage the namespace, add a namespace.yaml to base/resources:
```

### Problem: `kustomize edit fix` breaks the build

```
Step 1: Check git diff after running edit fix
  └─ git diff overlays/dev/kustomization.yaml

Step 2: Verify patch files still exist if migration moved from inline to file-based
  └─ Some migrations produce references to files that don't exist yet

Step 3: Run build after fix to confirm
  └─ kustomize build overlays/dev > /dev/null && echo "OK"
```

### Diagnostic Commands Reference

```bash
# Full build with verbose error output
kustomize build overlays/dev 2>&1 | head -50

# Check what resources kustomize sees
kustomize build overlays/dev | grep "^kind:"

# Verify specific field in output
kustomize build overlays/dev | grep -A3 "replicas"
kustomize build overlays/dev | grep "image:"
kustomize build overlays/dev | grep "namespace:"

# Diff against cluster (requires running cluster)
kubectl diff -k overlays/dev

# Check kustomize version (bundled vs standalone)
kubectl version --client
kustomize version
```

---

## 9. Kill Switch

> 10-second recall. The absolute minimum to hold in memory.

```
MENTAL MODEL:
  base (what stays) + overlay (what changes) = final manifest
  transformers first → patches second → patches always win

FILE RULE:
  Must be named: kustomization.yaml (exact, always)

PATCH RULES:
  Field exists in base  → op: replace
  Field missing in base → op: add
  Delete a field        → op: remove
  Modern syntax         → patches: (not patchesJson6902, not patchesStrategicMerge)
  Migrate legacy        → kustomize edit fix

TRANSFORMER LIMITS:
  Can: image tag, replicas, namespace, name prefix/suffix, labels, annotations
  Cannot: env vars, resource limits, ports, volume mounts → use patches

COMMANDS:
  Build (no apply):  kustomize build overlays/dev
  Apply:             kubectl apply -k overlays/dev
  Diff:              kubectl diff -k overlays/dev
  Migrate:           kustomize edit fix (run inside overlay dir)

ORDER:
  1. transformers  2. patches  3. kubectl apply  4. admission controllers
```

---

## 10. Appendix

### Kustomize Field Reference — kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# ── RESOURCES ──────────────────────────────────────────────────
resources:
  - deploy.yaml              # local file
  - svc.yaml
  - ../../base               # relative path to another kustomize root
  - namespace.yaml           # kubernetes resource file

# ── TRANSFORMERS ───────────────────────────────────────────────
namePrefix: app1-
nameSuffix: -dev
namespace: dev

labels:
  - pairs:
      env: dev
    includeSelectors: true   # add label to selectors too

commonAnnotations:
  branch: dev
  owner: platform-team

images:
  - name: nginx              # matches container image name
    newName: nginx           # can change registry/repo
    newTag: "1.22"

replicas:
  - name: nginx-deploy       # matches Deployment/StatefulSet name
    count: 3

# ── PATCHES (MODERN — PREFERRED) ───────────────────────────────
patches:
  # Inline JSON patch
  - target:
      group: apps
      version: v1
      kind: Deployment
      name: nginx-deploy
    patch: |-
      - op: add
        path: /spec/template/spec/containers/0/env
        value:
          - name: LOG_LEVEL
            value: debug

  # File-based JSON patch
  - path: patch-container.yaml
    target:
      group: apps
      kind: Deployment
      name: nginx-deploy
      version: v1

  # File-based strategic merge patch (Kustomize auto-detects type)
  - path: patch-merge.yaml

# ── DEPRECATED (DO NOT USE IN NEW CODE) ────────────────────────
# patchesJson6902:           → use patches: instead
# patchesStrategicMerge:     → use patches: instead
# commonLabels:              → use labels: instead

# ── GENERATORS ─────────────────────────────────────────────────
configMapGenerator:
  - name: app-config
    literals:
      - LOG_LEVEL=info
      - APP_ENV=production

secretGenerator:
  - name: db-secret
    envs:
      - secrets.env
    options:
      disableNameSuffixHash: true
```

### RFC 6902 JSON Patch Operations Quick Reference

| Operation | Path Must Exist? | Use Case |
|---|---|---|
| `add` | No (creates it) | Inject new field, append to array |
| `replace` | Yes | Change existing field value |
| `remove` | Yes | Delete a field entirely |
| `copy` | Source must exist | Duplicate a value to a new path |
| `move` | Source must exist | Rename/relocate a field |
| `test` | Yes | Assert value before applying changes |

### Helm vs Kustomize Quick Comparison

| Feature | Kustomize | Helm |
|---|---|---|
| Built into kubectl | ✅ Yes | ❌ No |
| Templating language | ❌ None | ✅ Go templates |
| Values/variables | ❌ None | ✅ values.yaml |
| Package/chart registry | ❌ No | ✅ Yes |
| Release tracking | ❌ No | ✅ Yes |
| Rollback | ❌ No | ✅ Yes |
| Hooks/lifecycle | ❌ No | ✅ Yes |
| Learning curve | Low | Medium-High |
| Best for | Own apps, env config | Third-party apps, complex templating |

### Common Path Patterns

```
Container image:          /spec/template/spec/containers/0/image
First container env:      /spec/template/spec/containers/0/env
Container resources:      /spec/template/spec/containers/0/resources
Specific metadata label:  /metadata/labels/my-label-key
Deployment replicas:      /spec/replicas
Service nodePort:         /spec/ports/0/nodePort
Service type:             /spec/type
```

### Diagnostic Commands One-Liner Reference

```bash
kustomize build overlays/dev                          # render without apply
kustomize build overlays/dev | grep "image:"          # check image resolution
kustomize build overlays/dev | grep "replicas"        # check replica count
kustomize build overlays/dev | grep "namespace:"      # check namespace assignment
kubectl apply -k overlays/dev --dry-run=client        # dry-run apply
kubectl diff -k overlays/prod                         # diff against live cluster
cd overlays/dev && kustomize edit fix                  # migrate deprecated fields
kustomize version                                     # check standalone version
kubectl version --client                              # check bundled kustomize version
```
---
