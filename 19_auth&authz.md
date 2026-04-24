# Kubernetes API & Authorization 

---

# 🔥 1. Core Intuition (FIRST PRINCIPLES)

## 🧠 Mental Model

👉 EVERYTHING in Kubernetes = API call

* `kubectl` → API request
* Controller → API request
* kubelet → API request

👉 Flow:

```
Request → API Server → Auth → AuthZ → Admission → etcd
```

---

## ⚡ Golden Line 

👉 **API Server = Gatekeeper of the cluster**

* Authentication → Who are you
* Authorization → What can you do
* Admission → Should we allow this config

---

# 🌐 2. Kubernetes API (ENGINE)

## 💡 What it really is

* REST API
* Resource-based
* HTTPS secured

---

## 🔁 Example Mapping

```bash
kubectl get pods
```

👉 becomes:

```http
GET /api/v1/namespaces/default/pods
```

---

## 🧠 Key Insight

👉 You are NEVER talking to cluster directly
👉 You are ALWAYS talking to API server

---

# 🧭 3. API Endpoints (IMPORTANT)

## 🔑 Resource Endpoints

* `/api` → core group
* `/apis` → named groups

*`/api` → Core group / Legacy*
- *Original resources*: Holds foundational k8s objects like `pods`, `services`, `nodes`, `namespaces`
- *Empty group name*: Technical group is `""`, accessed via path `/api/v1`
- *Legacy design*: Exists for backwards compatibility with early k8s versions

```bash
kubectl get --raw /api/v1 | jq .resources[].name          # core resources
```

*`/apis` → Named groups / Extensions*
- *Newer resources*: Holds all extension resources like `deployments`, `jobs`, `ingress`, `roles`, `crds`
- *Explicit group names*: Uses clear names like `apps`, `batch`, `networking.k8s.io` at `/apis/<group>/<version>`
- *Extensible design*: Added to support new features without breaking the core API

```bash
kubectl get --raw /apis/apps/v1 | jq .resources[].name    # apps group resources
```

* So /api = OG resources, /apis = everything else that came after.
---

## ⚙️ Non-Resource Endpoints

* `/version`
* `/healthz`, `/livez`, `/readyz`
* `/metrics`

---

## ⚠️ Interview Trap

👉 `kubectl logs` ≠ API `/logs`

* logs come from **kubelet**, not API server

---

# 🧠 4. API Groups (VERY IMPORTANT)

## 🔹 Core Group

```yaml
apiVersion: v1
```

Examples:

* Pods
* Services
* ConfigMaps

👉 group = `""` (empty)

---

## 🔹 Named Groups

```yaml
apiVersion: apps/v1
```

Examples:

* Deployments
* StatefulSets
* Jobs

---

## ⚠️ Interview Trap

👉 Why no group in `v1`?

Answer:

* legacy design
* backward compatibility

---

# 🔍 5. Explore API (Hands-on)

## ▶️ Commands

```bash
kubectl api-resources
kubectl api-versions
```

---

## 🌐 Direct API Access

### Using curl (unsafe quick way)

```bash
curl -k https://<api-server>/version
```

---

### Using kubectl proxy (BEST WAY)

```bash
kubectl proxy
curl http://127.0.0.1:8001/api/v1/pods
```

👉 No TLS headache
👉 Auto auth

---

# 🔐 6. Authorization (CORE TOPIC)

## 💡 What it is

* Authentication → who are you
* Authorization → what can you do

---

## 🧠 Decision Factors

* user
* verb (get/create/delete)
* resource
* namespace
* API group

---

## ⚡ Failure Response

👉 If denied:

```
403 Forbidden
```

---

# ⚙️ 7. Authorization Modes (IMPORTANT)

---

# 🥇 1. RBAC (MOST IMPORTANT)

## 💡 Concept

* Role → permissions
* Binding → attach to user

---

## 🧾 Example

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get","list","watch"]
```

```yaml
kind: RoleBinding
subjects:
- kind: User
  name: seenu
roleRef:
  kind: Role
  name: pod-reader
```

---

## ⚠️ Interview Gold

👉 RoleBinding can bind ClusterRole
👉 But scope remains namespace

---

# 🥈 2. ABAC (LEGACY ❌)

* JSON file
* restart API server for changes
* not used in prod

👉 Just know → don’t use

---

# 🥉 3. Webhook Authorization (ADVANCED 🔥)

## 💡 Concept

* API server → external service
* external decides allow/deny

---

## 🔥 Real Tools

* OPA
* Gatekeeper
* Kyverno

---

## ⚠️ Interview Trick

👉 Webhook auth **cannot see full object**

* sees request metadata only
* NOT pod spec

---

# 🧠 4. Node Authorization

## 💡 Purpose

👉 Restrict kubelet access

* node1 → only its pods
* cannot access node2 data

---

## 🔥 Why important

* limits blast radius
* security isolation

---

# ☠️ 5. AlwaysAllow / AlwaysDeny

| Mode  | Behavior           |
| ----- | ------------------ |
| Allow | everything allowed |
| Deny  | everything blocked |

👉 Only for testing

---

# 🔁 8. Authorization Flow (CRITICAL)

## 🧠 Order Matters

```bash
--authorization-mode=Node,RBAC,Webhook
```

---

## ⚡ Decision Logic

| Result     | Meaning |
| ---------- | ------- |
| Allow      | stop    |
| Deny       | stop    |
| No opinion | next    |

---

## 🔥 Example Flow

1. Node → no opinion
2. RBAC → allow
   👉 STOP

Webhook never called

---

## ⚠️ Interview Trap

👉 Order decides behavior

---

# 🧪 9. Real Scenario (VERY IMPORTANT)

## 🎯 Requirement

* You can create pods
* Only from trusted registry

---

## 🧠 Solution

| Layer      | Tool            |
| ---------- | --------------- |
| Permission | RBAC            |
| Policy     | OPA (admission) |

---

## 🔥 Key Insight

👉 Authorization ≠ Admission

* Auth → can she create?
* Admission → what is she creating?

---

# 🚨 10. Production & Advanced Scenarios

---

## 🔍 Debugging Authorization Issues

```bash
kubectl auth can-i create pods --as=seenu
```

👉 BEST command for interviews

---

## 🧨 Scenario 1: Access denied

Check:

* Role
* Binding
* namespace

---

## 🧨 Scenario 2: Works in one namespace, not other

👉 RBAC is namespace scoped

---

## 🧨 Scenario 3: API call failing

Check:

* authentication
* RBAC rules
* API group mismatch

---

## ⚙️ Check API Server Config

```bash
cat /etc/kubernetes/manifests/kube-apiserver.yaml
```

Look for:

```bash
--authorization-mode=
```

---

## ☁️ Managed Cluster Reality

* EKS / GKE / AKS
  👉 Cannot access API server config

---

# 🧠 11. Real Production Patterns

| Problem             | Solution  |
| ------------------- | --------- |
| user access control | RBAC      |
| compliance rules    | OPA       |
| node isolation      | Node Auth |
| audit control       | Webhook   |

---

# ⚔️ 12. Interview Questions

---

### Q1

👉 What is Kubernetes API?

→ central REST interface

---

### Q2

👉 Difference: API vs API Server?

→ API = interface
→ API server = implementation

---

### Q3

👉 What happens when request comes?

→ Auth → AuthZ → Admission

---

### Q4 (IMPORTANT 🔥)

👉 RBAC vs Webhook?

→ RBAC = static permissions
→ Webhook = dynamic logic

---

### Q5 (HARD 🔥)

👉 Why RBAC + OPA together?

→ RBAC → access
→ OPA → policy

---

### Q6

👉 How to debug permission issue?

```bash
kubectl auth can-i
```
# 💣 13. KillerCoda Practice (MANDATORY)

## 🎯 Goal

Turn theory → muscle memory

---

## ⚔️ Task 1: RBAC Creation + Verification

👉 Create Role + RoleBinding

```bash
kubectl create ns rbac-test

kubectl create role pod-reader \
  --verb=get,list,watch \
  --resource=pods \
  -n rbac-test

kubectl create rolebinding read-pods \
  --role=pod-reader \
  --user=seenu\
  -n rbac-test
```

👉 Verify permissions

```bash
kubectl auth can-i get pods --as=seema -n rbac-test
```

✅ Expected:

```
yes
```

---

## 🧨 Task 2: Deny Access (IMPORTANT)

👉 Try forbidden action

```bash
kubectl auth can-i delete pods --as=seema -n rbac-test
```

❌ Expected:

```
no
```

👉 Interview insight:
👉 Authorization failure = **403 Forbidden**

---

## 🌐 Task 3: API Access via Proxy

👉 Start proxy

```bash
kubectl proxy
```

👉 Call API

```bash
curl http://127.0.0.1:8001/api/v1/namespaces/default/pods
```

🧠 Insight:
👉 kubectl = REST client
👉 This is actual Kubernetes API

---

## 🔍 Task 4: Explore API Groups (INTERVIEW GOLD)

```bash
kubectl api-resources
```

👉 Focus:

* core → `v1`
* apps → `apps/v1`
* batch → `batch/v1`
* rbac → `rbac.authorization.k8s.io/v1`

👉 Interview question:
“Where does Deployment belong?”

Answer:
👉 `apps/v1`

---

## 🧪 Task 5 (ADVANCED 🔥)

👉 Check non-resource permissions

```bash
kubectl auth can-i get /healthz --as=seema
```

👉 Learn:
👉 RBAC also controls **non-resource URLs**

---

# 🧩 Final Insight

👉 Kubernetes is NOT YAML
👉 Kubernetes is **API + control plane decisions**

