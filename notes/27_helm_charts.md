# Helm — CKA + Senior Interview Mastery Guide

> **Scope:** Helm v3 only. Every section connects to the next. Built for exam speed and interview depth.

---

## 0. First Principles

> **The mental model that never changes, regardless of Helm version or cluster size.**

**Helm = Package Manager for Kubernetes, with full application lifecycle management.**

Three immutable truths:

1. **Kubernetes has no concept of "application."** To Kubernetes, your app is just a bag of independent resources — Deployments, Services, ConfigMaps — with no awareness of how they relate. Helm imposes that structure.

2. **A Chart is a versioned blueprint. A Release is a live instance.** Just as a Docker image is a blueprint and a container is the running instance, a Helm chart is a blueprint and a release is the live, tracked deployment. This analogy is the mental model for everything that follows.

3. **Helm tracks state; `kubectl apply` does not.** `kubectl apply` is declarative but stateless from a release perspective. Helm wraps every install, upgrade, and rollback into a numbered revision stored as a Secret in Kubernetes — giving you a full audit trail and safe rollback path with a single command.

```
Repository  →  contains Charts  →  installed as Releases  →  tracked as Revisions
(like Docker Hub)  (like images)   (like containers)       (like image tags in use)
```

---

## 1. Reality Constraints

> **What Kubernetes and Helm actually do and don't do. Where the boundaries are.**

| Constraint | Detail |
|---|---|
| **Helm v3 has no Tiller** | Helm v3 communicates directly with the Kubernetes API server using your kubeconfig context. No cluster-side component required. |
| **Helm uses your current `kubectl` context** | Whatever context `kubectl config current-context` shows is where Helm operates. |
| **Helm stores release state as Secrets** | Each revision is stored as a `helm.sh/release.v1` Secret in the release's namespace. Deleting the namespace deletes release history. |
| **Release names are scoped to namespace** | You can have `app1` in `dev` and `app1` in `prod` — they are distinct releases. |
| **`Chart.yaml` is not templatable** | You cannot use `{{ .Values.something }}` in `Chart.yaml`. It is static metadata. |
| **`values.yaml` is merged, not replaced** | When you pass `-f custom.yaml`, Helm deep-merges it with the base `values.yaml`. Explicit `--set` values win over `-f` files. |
| **Resource names must be cluster-unique per namespace** | Without `{{ .Release.Name }}` in resource names, a second install of the same chart in the same namespace will fail with a conflict error. |
| **Rollback creates a new revision** | Rolling back to revision 1 does not delete revision 2; it creates revision 3 with revision 1's manifests. The history grows forward only. |
| **Helm does not manage resources created outside Helm** | If you `kubectl apply` a resource manually that Helm also owns, Helm may overwrite or conflict with it on the next `helm upgrade`. |
| **`helm uninstall` deletes all chart-managed resources** | Unlike `kubectl delete -f`, Helm knows exactly which resources belong to the release and removes all of them atomically. |

---

## 2. Decision Logic

> **When to use what. Clear rules for exam scenarios and architecture decisions.**

### Helm vs Kustomize — When to Choose Which

| Dimension | Kustomize | Helm |
|---|---|---|
| **Templating** | Patch/overlay only | Full Go templates (conditionals, loops, helpers) |
| **Lifecycle ops** | `kubectl apply/delete` only | install, upgrade, rollback, uninstall |
| **Release tracking** | None | Full revision history |
| **Packaging & distribution** | Not supported | `helm package`, chart repositories |
| **Learning curve** | Lower | Higher |
| **Best for** | Config overlays, GitOps patches | Full app lifecycle, third-party tools |

**Rule:** Use Kustomize when you need environment-specific config patches. Use Helm when you need versioned, releaseable, rollback-capable application packages.

---

### Values Precedence (highest to lowest)

```
--set key=value               # CLI wins absolutely
--set-string key=value        # Same precedence as --set, forces string type
-f custom-values.yaml         # Last -f file wins over earlier ones
values.yaml (chart default)   # Baseline, always overridden by the above
```

**Exam trap:** When multiple `-f` files are passed, the rightmost file wins on key conflicts.

---

### When to Use `--namespace` on `helm install`

```
Scenario                              Recommendation
------------------------------------------------------
Resources need to go to namespace X   Set metadata.namespace in templates via .Values.namespace
Helm release tracking in same ns      ALSO pass --namespace X to helm install
Production/CI-CD                      Always both: values-based namespace + --namespace flag
Quick test/dev                        Values-based only is acceptable
```

**Rule:** In production, always pass both. If you only use `metadata.namespace` in your YAML but don't pass `--namespace`, Helm tracks the release in the `default` namespace even though pods live in `dev`. This creates confusing output in `helm list`.

---

### Choosing the Right Verification Command

```
Goal                                      Command
-------------------------------------------------
Check chart structure + YAML syntax       helm lint ./chart
Preview rendered manifests (no release)   helm template ./chart
Preview with release name in manifests    helm template <release-name> ./chart
Full simulation (schema + API check)      helm install <name> ./chart --dry-run
Simulate with specific values file        helm install <name> ./chart -f env.yaml --dry-run
```

---

## 3. Internal Working

> **How Helm actually operates under the hood — step by step.**

### What Happens During `helm install`

```
1. Helm reads Chart.yaml → validates chart API version and metadata
2. Helm loads values.yaml → merges with any -f files → applies --set overrides
3. Helm renders all templates in templates/ using Go templating engine
   → .Values, .Release, .Chart, .Capabilities objects are populated
4. Rendered manifests are validated against the Kubernetes OpenAPI schema
5. Helm creates a Secret of type helm.sh/release.v1 in the target namespace
   → This Secret encodes the rendered manifest + metadata (revision=1, status=deployed)
6. Helm sends the manifests to the Kubernetes API server via kubectl-equivalent calls
7. Kubernetes controllers reconcile: Deployment controller creates ReplicaSets → Pods
8. Helm waits (if --wait flag used) for all resources to be Ready
9. NOTES.txt is rendered and printed to stdout
```

### What Happens During `helm upgrade`

```
1. Steps 1-4 same as install (renders new manifests)
2. Helm compares new rendered manifests to the stored previous release Secret
3. Helm sends a 3-way strategic merge patch to the Kubernetes API
   (old manifest → new manifest → current live state)
4. A new Secret is created: revision=N+1, status=deployed
5. Previous revision Secret is updated: status=superseded
```

### What Happens During `helm rollback <release> <revision>`

```
1. Helm fetches the Secret for the target revision number
2. Decodes the stored manifest from that revision
3. Sends that manifest to the Kubernetes API (effectively re-applying it)
4. Creates a NEW Secret: revision=N+1, status=deployed
   → Contains the old manifest — Helm never modifies existing revision Secrets
5. Previous revision's Secret: status=superseded
```

**Key insight:** Rollback in Helm is "replay the past" not "undo the present." Revision numbers always go forward.

### Where Helm Stores Release State

```bash
# Helm stores each revision as a Secret in the release namespace
kubectl get secrets -n <namespace> | grep helm.sh/release

# Decode a specific revision to inspect stored manifest
kubectl get secret <secret-name> -o jsonpath='{.data.release}' | base64 -d | base64 -d | gzip -d
```

### Template Rendering: Built-in Objects

| Object | Examples | Source |
|---|---|---|
| `.Values` | `.Values.replicaCount`, `.Values.image.tag` | `values.yaml` + overrides |
| `.Release` | `.Release.Name`, `.Release.Namespace`, `.Release.Revision`, `.Release.IsInstall`, `.Release.IsUpgrade` | Helm runtime |
| `.Chart` | `.Chart.Name`, `.Chart.Version`, `.Chart.AppVersion` | `Chart.yaml` |
| `.Capabilities` | `.Capabilities.KubeVersion.Version`, `.Capabilities.APIVersions.Has "networking.k8s.io/v1"` | Kubernetes API |
| `.Files` | `.Files.Get "config/app.conf"` | Non-template files in chart |

**Naming convention:** `.Values` is lowercase because you define it. Built-ins use PascalCase (`.Release`, `.Chart`) because Helm defines them.

---

## 4. Hands-On

> **Production-quality YAML and commands. Nothing simplified.**

### Complete Chart Structure

```
app1-chart/
├── Chart.yaml           # Required: chart metadata
├── values.yaml          # Required: default input values
├── values-dev.yaml      # Convention: environment override (not auto-loaded)
├── values-stage.yaml
├── values-prod.yaml
├── templates/
│   ├── _helpers.tpl     # Named templates (reusable blocks)
│   ├── deploy.yaml
│   ├── svc.yaml
│   ├── NOTES.txt        # Post-install message
│   └── tests/
│       └── test-connection.yaml
└── charts/              # Dependency sub-charts
```

---

### Chart.yaml (Production)

```yaml
apiVersion: v2
name: app1-chart
description: Multi-env NGINX web service with environment-specific values
type: application           # 'application' or 'library'
version: 0.2.0              # Chart version — increment on every chart change (SemVer)
appVersion: "1.22"          # App version — always reflects PRODUCTION app version
home: https://example.com
maintainers:
  - name: Your Name
    email: you@example.com
keywords:
  - nginx
  - multi-environment
```

> **`appVersion` rule:** In a multi-environment chart, `appVersion` must always reflect the production application version. Environment-specific versions are controlled via values files, not `Chart.yaml`, since `Chart.yaml` is not templatable.

---

### templates/deploy.yaml (Templatized)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-nginx-deploy
  namespace: {{ .Values.namespace }}
  labels:
    app: nginx
    helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version }}
    app.kubernetes.io/managed-by: {{ .Release.Service }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: {{ .Release.Name }}-cont
          image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: 80
```

---

### templates/svc.yaml (Templatized)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}-nginx-svc
  namespace: {{ .Values.namespace }}
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

---

### values-dev.yaml

```yaml
replicaCount: 1
namespace: dev
image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: "1.23"
```

### values-stage.yaml

```yaml
replicaCount: 2
namespace: stage
image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: "1.22"
```

### values-prod.yaml

```yaml
replicaCount: 3
namespace: prod
image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: "1.22"
```

---

### Core Helm Commands — Production Reference

```bash
# --- REPO MANAGEMENT ---
helm repo add my-bitnami https://charts.bitnami.com/bitnami
helm repo update                              # Refresh cached repo index
helm repo list
helm repo remove my-bitnami

# --- SEARCH ---
helm search repo my-bitnami/nginx             # Search local repos (offline)
helm search hub nginx                         # Search Artifact Hub (online)
helm search hub nginx | grep -i bitnami

# --- CHART SCAFFOLDING ---
helm create app1-chart                        # Scaffold boilerplate chart

# --- VERIFICATION (always run before install) ---
helm lint ./app1-chart
helm lint ./app1-chart -f ./app1-chart/values-dev.yaml
helm template ./app1-chart                    # Render with no release name
helm template app1-dev ./app1-chart -f ./app1-chart/values-dev.yaml
helm template app1-dev ./app1-chart --debug
helm install app1-dev ./app1-chart --dry-run

# --- INSTALL ---
helm install app1-dev ./app1-chart \
  -f ./app1-chart/values-dev.yaml \
  --namespace dev \
  --create-namespace                          # Creates namespace if not exists

helm install app1-stage ./app1-chart \
  -f ./app1-chart/values-stage.yaml \
  --namespace stage \
  --create-namespace

helm install app1-prod ./app1-chart \
  -f ./app1-chart/values-prod.yaml \
  --namespace prod \
  --create-namespace

# --- STATUS & HISTORY ---
helm list                                     # Releases in default namespace
helm list -A                                  # All releases across all namespaces
helm list -n dev
helm status app1-dev -n dev
helm history app1-dev -n dev
helm get values app1-dev -n dev               # Show values used in current release
helm get manifest app1-dev -n dev             # Show rendered manifests of current release

# --- UPGRADE ---
helm upgrade app1-dev ./app1-chart \
  -f ./app1-chart/values-dev.yaml \
  --namespace dev

helm upgrade app1-dev ./app1-chart \
  --set image.tag=1.24 \
  --namespace dev

helm upgrade app1-dev ./app1-chart \
  --reuse-values \                            # Keep all previously set values
  --set image.tag=1.24 \
  --namespace dev

# --- ROLLBACK ---
helm rollback app1-dev 1 -n dev              # Roll back to revision 1
helm rollback app1-dev 0 -n dev              # Roll back to previous revision (0 = last)

# --- UNINSTALL ---
helm uninstall app1-dev -n dev
helm uninstall app1-dev -n dev --keep-history # Preserves history Secrets

# --- PACKAGING & DISTRIBUTION ---
helm package ./app1-chart                    # Creates app1-chart-0.2.0.tgz
helm push app1-chart-0.2.0.tgz oci://registry.example.com/charts
helm pull my-bitnami/nginx --untar           # Download and unpack chart locally
```

---

## 5. Production Flow

> **Real-world architecture and design patterns.**

### Multi-Environment Pattern (Single Chart, Multiple Releases)

```
app1-chart/
├── values.yaml           ← base defaults (rarely used directly in prod)
├── values-dev.yaml       ← dev overrides: replicas=1, tag=1.23
├── values-stage.yaml     ← stage overrides: replicas=2, tag=1.22
└── values-prod.yaml      ← prod overrides: replicas=3, tag=1.22

helm install app1-dev   ./app1-chart -f values-dev.yaml   --namespace dev
helm install app1-stage ./app1-chart -f values-stage.yaml --namespace stage
helm install app1-prod  ./app1-chart -f values-prod.yaml  --namespace prod
```

Each release is fully independent: independent revision history, independent lifecycle, independent rollback.

---

### Environment Isolation Matrix

| Env | Release Name | Namespace | Helm Tracking NS | Image | Replicas |
|---|---|---|---|---|---|
| dev | `app1-dev` | `dev` | `dev` | nginx:1.23 | 1 |
| stage | `app1-stage` | `stage` | `stage` | nginx:1.22 | 2 |
| prod | `app1-prod` | `prod` | `prod` | nginx:1.22 | 3 |

---

### CI/CD Pipeline Integration Pattern

```bash
# Typical CI/CD upgrade flow
helm upgrade --install app1-prod ./app1-chart \   # --install = install if not exists
  -f values-prod.yaml \
  --set image.tag=${GIT_SHA} \                    # Dynamic tag from pipeline
  --namespace prod \
  --atomic \                                       # Roll back automatically if upgrade fails
  --timeout 5m \
  --wait                                           # Block until all resources are Ready
```

`--atomic` is the CI/CD safety net: if any resource fails to become Ready within `--timeout`, Helm automatically rolls back to the previous revision and exits non-zero.

---

### Dependency Management (Umbrella Chart Pattern)

```yaml
# Chart.yaml of parent chart
dependencies:
  - name: postgresql
    version: "12.1.6"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled       # Conditional — controlled via values
  - name: redis
    version: "17.3.2"
    repository: https://charts.bitnami.com/bitnami
```

```bash
helm dependency update ./my-app    # Pulls dependencies into charts/
helm dependency list ./my-app      # Verify dependency versions
```

---

### Helm Hooks for Lifecycle Events

```yaml
# templates/db-migrate-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-db-migrate
  annotations:
    "helm.sh/hook": pre-upgrade          # Runs before upgrade
    "helm.sh/hook-weight": "-5"          # Lower = runs earlier
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
          command: ["./migrate.sh"]
```

Common hook types: `pre-install`, `post-install`, `pre-upgrade`, `post-upgrade`, `pre-rollback`, `post-rollback`, `pre-delete`, `post-delete`, `test`.

---

## 6. Mistakes

> **What actually breaks in real systems — root cause and fix.**

### Mistake 1: Resource Name Conflict on Re-install

**Symptom:**
```
Error: INSTALLATION FAILED: rendered manifests contain a resource that already exists.
```
**Root cause:** Templates use hardcoded resource names (e.g., `name: nginx-deploy`) instead of `{{ .Release.Name }}-nginx-deploy`. When you install a second release from the same chart in the same namespace, names collide.

**Fix:** Always prefix resource names with `{{ .Release.Name }}`:
```yaml
metadata:
  name: {{ .Release.Name }}-nginx-deploy
```

---

### Mistake 2: Helm Release Tracked in Wrong Namespace

**Symptom:** `helm list -n prod` shows no releases, but pods are running in `prod`.

**Root cause:** Used `metadata.namespace: prod` in templates (via values) but did not pass `--namespace prod` to `helm install`. Helm tracked the release in `default`.

**Fix:** Always pass both:
```bash
helm install app1-prod ./chart -f values-prod.yaml --namespace prod
```

---

### Mistake 3: `appVersion` Set to Dev/Stage Version

**Symptom:** `helm list -A` shows `APP VERSION: 1.23` for the prod release, but prod is actually running `1.22`.

**Root cause:** `appVersion` in `Chart.yaml` was set to the dev image tag. Since `Chart.yaml` is not templatable, it applies to all environments.

**Fix:** `appVersion` in `Chart.yaml` must always reflect the production application version. Dev/stage tags are controlled via environment-specific values files.

---

### Mistake 4: Forgetting `--namespace` on `helm rollback`

**Symptom:**
```
Error: release: not found
```
**Root cause:** `helm rollback app1-prod 1` without `-n prod` looks for the release in the `default` namespace.

**Fix:**
```bash
helm rollback app1-prod 1 -n prod
```

---

### Mistake 5: `--dry-run` Doesn't Catch All Errors

**Symptom:** `--dry-run` passes, but actual `helm install` fails with a Kubernetes API error (e.g., invalid field, missing CRD).

**Root cause:** `helm template` and basic `--dry-run` validate YAML structure and known schema but cannot catch errors that require a live cluster (e.g., missing CRD types, RBAC violations, namespace not existing).

**Fix:** Use `--dry-run=server` (Helm 3.13+) for server-side validation:
```bash
helm install app1-prod ./chart --dry-run=server -n prod
```

---

### Mistake 6: Overwriting Manually Applied Resources

**Symptom:** A `kubectl apply` change to a resource is silently overwritten on next `helm upgrade`.

**Root cause:** Helm performs a 3-way merge against the last stored manifest, not against live cluster state. Manual changes that diverge from the stored manifest may be reverted.

**Fix:** Never manually edit resources managed by Helm. All changes go through `helm upgrade --set` or updated values files.

---

### Mistake 7: `helm uninstall` Destroys Release History

**Symptom:** After uninstall, `helm history app1-prod` returns `Error: release: not found`. You can no longer roll forward.

**Root cause:** `helm uninstall` by default deletes all release Secrets, not just the deployed resources.

**Fix:** Use `--keep-history` when there's any chance you need the history:
```bash
helm uninstall app1-prod -n prod --keep-history
```

---

## 7. Interview Answers

> **Full spoken sentences, verbatim-ready. Compressed for delivery under 90 seconds.**

---

**Q: What is Helm and why do we need it if we already have `kubectl apply`?**

"Helm is a package manager for Kubernetes that adds application lifecycle management on top of raw manifest deployment. `kubectl apply` is stateless — it sends manifests to the cluster but has no memory of what version was deployed, no concept of rollback, and no way to track an application as a whole. Helm solves this by grouping all your Kubernetes resources into a versioned unit called a Chart, and every deployment of that chart becomes a Release with a full revision history. This means you can upgrade with a single command, and roll back to any previous revision just as easily. In production, this is essential — especially when you're managing multiple environments or distributing applications across teams."

---

**Q: What is the difference between a Chart, a Release, and a Repository?**

"A Chart is the packaged blueprint — it contains templates, default values, and metadata, similar to a Docker image. A Repository is where charts are stored and shared, analogous to Docker Hub or an OCI registry. A Release is a live, running instance of a chart in a Kubernetes cluster — like a running container is to a Docker image. Every `helm install` creates a release, and that release is assigned a name and tracked independently with its own revision history, even if you install the same chart multiple times."

---

**Q: How does Helm track state? Where is it stored?**

"Helm stores each release revision as a Kubernetes Secret of type `helm.sh/release.v1` in the release's namespace. Each Secret contains the base64-encoded, gzip-compressed rendered manifest for that revision, plus metadata like the revision number, chart version, and status. When you run `helm rollback`, Helm decodes the target revision's Secret, re-applies those manifests, and creates a new Secret with the next revision number. This means rollback always moves forward in revision count — it never deletes or modifies existing revision Secrets."

---

**Q: What happens when you run `helm rollback`?**

"Helm rollback does not undo — it replays. When you roll back to revision 1, Helm fetches the stored manifest from that revision's Secret, sends it to the Kubernetes API, and records this as a new revision — say revision 3. The cluster state reverts to what it was at revision 1, but revision 2 remains in history with a 'superseded' status. This design is intentional: it preserves a complete, auditable changelog of every change made to a release."

---

**Q: How do you manage multiple environments with a single Helm chart?**

"The recommended pattern is to maintain a single chart with a base `values.yaml` and separate environment-specific values files — `values-dev.yaml`, `values-stage.yaml`, `values-prod.yaml`. Each file overrides the values relevant to that environment, like replica count, image tag, and namespace. You install each environment as a separate named release using `helm install app1-dev ./chart -f values-dev.yaml --namespace dev`. Each release has its own lifecycle, revision history, and upgrade path, but they all share the same chart codebase. This is far more maintainable than duplicating manifests per environment."

---

**Q: What is the difference between `helm template`, `helm lint`, and `helm install --dry-run`?**

"`helm lint` validates the chart's structure, metadata, and YAML syntax — it's a static analysis tool that runs entirely locally. `helm template` renders all templates into final Kubernetes manifests using supplied values, but makes no contact with the cluster and creates no release. `helm install --dry-run` goes a step further: it renders templates and simulates a full release install, validating rendered manifests against the Kubernetes API schema. With `--dry-run=server` available in Helm 3.13+, the validation is fully server-side, catching CRD-related errors and RBAC issues that client-side validation misses."

---

**Q: Why does rollback create a new revision rather than reverting to the old one?**

"Because Helm's revision model is append-only by design, similar to a git commit history. Every change — install, upgrade, or rollback — creates a new revision. This guarantees that the history is immutable and auditable. If Helm deleted revisions on rollback, you could lose the audit trail of what was deployed and when. By creating a new revision with the old manifest, Helm gives you a complete timeline: you can always see 'we upgraded to revision 2, then rolled back via revision 3.' This is critical in regulated environments where deployment history is required for compliance."

---

## 8. Debugging

> **Fast diagnosis paths — commands first, decision tree second.**

### Diagnosis Path: `helm install` Fails

```
helm install <name> ./chart --dry-run 2>&1
    │
    ├── YAML syntax error → helm lint ./chart
    │     └── Fix template syntax, check {{ }} blocks
    │
    ├── "resource already exists" → hardcoded resource names
    │     └── Add {{ .Release.Name }} prefix to metadata.name
    │
    ├── Schema validation error → check apiVersion/kind in templates
    │     └── helm install --dry-run=server for server-side schema check
    │
    └── Passes --dry-run but fails on actual install
          └── helm install --debug → verbose output
          └── kubectl get events -n <namespace> --sort-by='.lastTimestamp'
```

---

### Diagnosis Path: Release Exists But Resources Are Wrong

```
helm status <release> -n <namespace>
    │
    ├── STATUS: failed → helm history <release> -n <ns>
    │     └── helm rollback <release> <previous-good-revision> -n <ns>
    │
    ├── STATUS: deployed but pods failing
    │     └── kubectl get pods -n <ns>
    │     └── kubectl describe pod <pod> -n <ns>
    │     └── kubectl logs <pod> -n <ns>
    │
    └── Wrong values applied
          └── helm get values <release> -n <ns>      # What values are active?
          └── helm get manifest <release> -n <ns>    # What manifests are live?
          └── helm upgrade <release> ./chart -f corrected-values.yaml -n <ns>
```

---

### Diagnosis Path: `helm list` Shows Nothing But Resources Exist

```
helm list                                    # Default namespace only
helm list -A                                 # All namespaces — check here first

Release not in expected namespace?
    └── Helm installed without --namespace → tracked in default
    └── Fix: reinstall with correct --namespace flag

kubectl get secrets -A | grep helm.sh        # Find all release Secrets
kubectl get secret <helm-secret> -n <ns> -o yaml   # Inspect raw release data
```

---

### Diagnosis Path: Upgrade Fails Mid-Way

```
helm upgrade fails
    │
    ├── helm history <release> -n <ns>
    │     └── Latest revision shows STATUS: failed or pending-upgrade
    │
    ├── helm rollback <release> 0 -n <ns>    # Rollback to previous (0 = last good)
    │
    └── Root cause investigation
          kubectl get events -n <ns> --sort-by='.lastTimestamp'
          kubectl describe deployment <name> -n <ns>
          kubectl get pods -n <ns> -o wide
```

---

### Key Diagnostic Commands — Quick Reference

```bash
# Release state
helm list -A
helm status <release> -n <ns>
helm history <release> -n <ns>
helm get values <release> -n <ns>
helm get manifest <release> -n <ns>
helm get all <release> -n <ns>             # Values + manifest + hooks + notes

# Secrets (where Helm stores state)
kubectl get secrets -n <ns> -l owner=helm
kubectl get secret <secret> -n <ns> -o jsonpath='{.data.release}' \
  | base64 -d | base64 -d | gzip -d | python3 -m json.tool

# Kubernetes resource state
kubectl get deploy,svc,pods -n <ns>
kubectl describe deployment <name> -n <ns>
kubectl get events -n <ns> --sort-by='.lastTimestamp'
kubectl rollout status deployment/<name> -n <ns>
```

---

## 9. Kill Switch

> **10-second recall. The absolute minimum to hold in working memory.**

```
Chart      = versioned blueprint (like Docker image)
Release    = live instance of chart (like running container)
Repository = where charts live (like Docker Hub)
Revision   = numbered snapshot of every change — rollback creates NEW revision

Values precedence:  --set  >  -f file (rightmost wins)  >  values.yaml

Core flow:
  helm install <release> ./chart -f values.yaml --namespace <ns>
  helm upgrade <release> ./chart --set image.tag=x -n <ns>
  helm rollback <release> <revision> -n <ns>
  helm history <release> -n <ns>
  helm list -A

State lives in: Kubernetes Secrets (helm.sh/release.v1) in release namespace

Never forget:
  - --namespace goes on BOTH helm command AND metadata.namespace in YAML (prod)
  - appVersion in Chart.yaml = PRODUCTION version only
  - Rollback = new revision forward, never backward
  - helm uninstall deletes history unless --keep-history
```

---

## 10. Appendix

> **Quick reference card — commands, formats, cheatsheets.**

### Complete Command Cheatsheet

```bash
# REPO
helm repo add <alias> <url>
helm repo update
helm repo list
helm repo remove <alias>

# SEARCH
helm search repo <alias>/<chart>          # Local repos
helm search hub <keyword>                 # Artifact Hub

# CHART CREATION
helm create <chart-name>
helm package ./chart                      # Creates .tgz

# VERIFY
helm lint ./chart [-f values.yaml]
helm template [release-name] ./chart [-f values.yaml] [--debug]
helm install <name> ./chart --dry-run [-f values.yaml]

# INSTALL
helm install <release> <chart-ref> \
  [-f values.yaml] \
  [--set key=value] \
  [--namespace ns] \
  [--create-namespace] \
  [--atomic] \
  [--wait] \
  [--timeout 5m]

# UPGRADE
helm upgrade <release> <chart-ref> \
  [-f values.yaml] \
  [--set key=value] \
  [--reuse-values] \
  [--namespace ns] \
  [--atomic]

# UPGRADE OR INSTALL (idempotent — CI/CD standard)
helm upgrade --install <release> <chart-ref> -f values.yaml -n ns

# ROLLBACK
helm rollback <release> [revision] [-n ns]
# revision=0 or omitted → previous revision

# STATUS
helm list [-A] [-n ns]
helm status <release> [-n ns]
helm history <release> [-n ns]
helm get values <release> [-n ns] [--all]
helm get manifest <release> [-n ns]
helm get all <release> [-n ns]

# UNINSTALL
helm uninstall <release> [-n ns] [--keep-history]

# PLUGINS & MISC
helm plugin list
helm env                                  # Show Helm environment variables
helm version
```

---

### Chart.yaml Required Fields

```yaml
apiVersion: v2          # Always v2 for Helm 3
name: chart-name        # Must match directory name
version: 0.1.0          # Chart version (SemVer — increment on every chart change)
type: application        # 'application' or 'library'
```

### Chart.yaml Optional Fields

```yaml
appVersion: "1.22"      # Application version (informational, production version)
description: "..."
home: https://...
maintainers:
  - name: Name
    email: email@example.com
keywords: []
dependencies:
  - name: postgresql
    version: "12.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
```

---

### Helm Template Quick Reference

```
{{ .Release.Name }}           Release name
{{ .Release.Namespace }}      Release namespace
{{ .Release.Revision }}       Revision number
{{ .Release.IsInstall }}      true on first install
{{ .Release.IsUpgrade }}      true on upgrade
{{ .Values.<key> }}           Value from values.yaml
{{ .Chart.Name }}             Chart name
{{ .Chart.Version }}          Chart version
{{ .Chart.AppVersion }}       App version
{{ .Capabilities.KubeVersion.Version }}   K8s version

# Conditionals
{{- if .Values.ingress.enabled }}
  ... yaml ...
{{- end }}

# Default value
{{ .Values.replicaCount | default 1 }}

# String formatting
{{ printf "%s-%s" .Release.Name .Chart.Name }}
{{ .Values.name | upper }}
{{ .Values.name | trunc 63 | trimSuffix "-" }}

# Named template (define in _helpers.tpl)
{{- define "app.fullname" -}}
{{ .Release.Name }}-{{ .Chart.Name }}
{{- end }}

# Use named template
{{ include "app.fullname" . }}
```

---

### Hook Annotation Reference

```yaml
annotations:
  "helm.sh/hook": pre-upgrade          # When to run
  "helm.sh/hook-weight": "-5"          # Execution order (lower = earlier)
  "helm.sh/hook-delete-policy": hook-succeeded  # Cleanup policy
```

| Hook | Timing |
|---|---|
| `pre-install` | Before any resources are created |
| `post-install` | After all resources are created |
| `pre-upgrade` | Before upgrade manifests are applied |
| `post-upgrade` | After upgrade is complete |
| `pre-rollback` | Before rollback manifests are applied |
| `post-rollback` | After rollback is complete |
| `pre-delete` | Before resources are deleted |
| `post-delete` | After resources are deleted |
| `test` | On `helm test` command |

---

### Helm vs Docker Analogy Table

| Concept | Docker | Helm |
|---|---|---|
| Blueprint | Container Image | Helm Chart |
| Registry | Docker Hub / ECR | Chart Repository / Artifact Hub |
| Live instance | Running Container | Helm Release |
| Build | `docker build` | `helm package` (optional) |
| Push | `docker push` | `helm push` |
| Run | `docker run` | `helm install` |
| Update | `docker pull` + `docker run` | `helm upgrade` |
| Revert | Restart old container | `helm rollback` |

---

