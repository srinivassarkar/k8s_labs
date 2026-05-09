# Kubernetes Downward API 

---

## What is the Downward API and why does it exist?

Imagine you're running 3 copies of an nginx pod. Each one has a different IP address. How does a container inside the pod know its own IP? It can't call the Kubernetes API — that requires credentials and setup. It can't hardcode the value — IPs change every time a pod restarts.

**The Downward API solves this.** It lets Kubernetes inject information about the pod (like its name, IP, or CPU limits) directly into the container — either as environment variables or as files. No API calls. No permissions needed. It just works.

---

## Two things you can inject

| Type | What it gives you | Example values |
|---|---|---|
| `fieldRef` | Pod-level info | name, IP, namespace, labels |
| `resourceFieldRef` | Container resource info | CPU limit, memory request |

---

## Two ways to inject it

| Method | How it works | Updates after start? |
|---|---|---|
| Environment variable | Value is set when container starts | ❌ No — it's frozen |
| Volume file | Written as a file you can `cat` | ✅ Yes — updates in ~1 min |

---

## Method 1 — Environment Variable with `fieldRef`

**Goal:** Make each pod log its own IP address every 10 seconds.

**Why this is useful:** When you have 3 replicas running, you want to know *which pod* produced which log line. Injecting `POD_IP` lets the container log its own identity without you hardcoding anything.

**The YAML — save this as `downwardapi_env.yaml`:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      environment: production
  template:
    metadata:
      labels:
        app: nginx
        environment: production
    spec:
      containers:
        - name: write-container
          image: alpine
          command: ["/bin/sh"]
          args:
            - -c
            - |
              while true; do
                echo "$(date): $POD_IP" >> /proc/1/fd/1
                sleep 10
              done
          env:
            - name: POD_IP           # This is the env var name your container sees
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP   # This tells K8s what to inject
```

**What happens when you apply this:**
1. Kubernetes starts 3 pods
2. Before each container starts, Kubernetes looks up the pod's IP
3. It sets `POD_IP=192.168.x.x` as an environment variable inside the container
4. The container's script uses `$POD_IP` to print the IP every 10 seconds

**Deploy and verify:**

```bash
# Apply the file
kubectl apply -f downwardapi_env.yaml

# See your pods and their IPs
kubectl get pods -o wide

# Check logs — you should see the IP printed every 10 seconds
kubectl logs <pod-name>

# Confirm the env var is actually set inside the container
kubectl exec -it <pod-name> -- env | grep POD_IP
```

**Expected log output:**
```
Fri Dec 22 07:36:28 UTC 2023: 192.168.1.5
Fri Dec 22 07:36:38 UTC 2023: 192.168.1.5
```

---

## Adding more variables — pod name too

You can inject multiple values. Here's how to also include the pod's name in the log.

**Add this to the `env` section (same YAML, more entries):**

```yaml
env:
  - name: POD_IP
    valueFrom:
      fieldRef:
        fieldPath: status.podIP
  - name: POD_NAME
    valueFrom:
      fieldRef:
        fieldPath: metadata.name
```

**Update the log command to include both:**

```yaml
args:
  - -c
  - |
    while true; do
      echo "$(date): $POD_IP: $POD_NAME" >> /proc/1/fd/1
      sleep 10
    done
```

**New log output looks like:**
```
Fri Dec 22 07:36:28 UTC 2023: 192.168.1.5: nginx-deploy-abc123
```

Now you know the exact pod that produced each log line. Very useful when debugging across 3 replicas.

---

## Method 2 — Volume File with `fieldRef`

**Goal:** Instead of an env var, write the pod name into a file inside the container.

**Why use a file instead of env var?**
- Files update automatically if labels/annotations change — env vars don't
- Sidecars can read the same file (useful for log agents, metrics collectors)
- Easier to inspect: `cat /var/downward/pod_name` vs grepping env

**The key part of the YAML — the volume definition:**

```yaml
volumes:
  - name: downward-vol
    downwardAPI:           # This is a special volume type
      items:
        - path: pod_name   # This is the filename that will be created
          fieldRef:
            fieldPath: metadata.name   # This is what gets written into that file
```

**And the volumeMount to make it accessible in the container:**

```yaml
volumeMounts:
  - name: downward-vol
    mountPath: /var/downward   # The container can now read files under this path
```

**What Kubernetes does behind the scenes:**
1. Creates a file at `/var/downward/pod_name` inside the container
2. Writes the pod's name into it (e.g., `nginx-deploy-abc123`)
3. The file is **read-only** — Kubernetes manages it, you can only read it

**How to read it:**

```bash
kubectl exec -it <pod-name> -- cat /var/downward/pod_name
# Output: nginx-deploy-abc123
```

> ⚠️ **Common mistake:** Trying to write to this file. You'll get `Read-only file system`. It's managed by Kubernetes. If you need a writable file, use an `emptyDir` volume at a different path.

---

## Method 3 — `resourceFieldRef` for CPU and Memory

**Goal:** Inject the container's CPU limit and memory limit so the app knows what resources it has.

**Why this matters:** Many apps (especially Java/JVM, Python workers) need to tune themselves based on available CPU/memory. If you don't inject these, the app might try to use more than its limit and get OOM-killed.

> Note: For resource fields (cpu, memory), you **must** use `resourceFieldRef`, not `fieldRef`. And you **must** specify `containerName`.

**Full YAML — save as `downwardapi_resources.yaml`:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      environment: production
  template:
    metadata:
      labels:
        app: nginx
        environment: production
    spec:
      containers:
        - name: demo
          image: alpine
          command: ["/bin/sh"]
          args:
            - -c
            - |
              while true; do
                echo "$(date): IP=$POD_IP | CPU_LIMIT=$CPU_LIMIT | MEM_LIMIT=$MEM_LIMIT" >> /proc/1/fd/1
                sleep 10
              done
          resources:
            limits:
              memory: "128Mi"
              cpu: "500m"
            requests:
              memory: "64Mi"
              cpu: "250m"
          env:
            - name: POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: CPU_LIMIT
              valueFrom:
                resourceFieldRef:
                  containerName: demo    # Must match the container name above
                  resource: limits.cpu
            - name: MEM_LIMIT
              valueFrom:
                resourceFieldRef:
                  containerName: demo
                  resource: limits.memory
```

**Deploy and check:**

```bash
kubectl apply -f downwardapi_resources.yaml

# Check the injected values inside the container
kubectl exec -it <pod-name> -- env | grep -E 'CPU|MEM|POD'
```

**Expected output:**
```
POD_IP=192.168.1.5
CPU_LIMIT=1          # 500m becomes 1 (rounded up to whole cores in some versions)
MEM_LIMIT=134217728  # 128Mi converted to bytes
```

> ⚠️ Memory is always injected in **bytes**. `128Mi = 134217728 bytes`. This is by design.

---

## Injecting CPU/Memory as Volume Files

**Goal:** Same resource info, but written to files instead of env vars.

**Why:** The container script can `cat` the file dynamically. Better for scripts that re-read config, or for sidecar containers that share the same volume.

**The volume definition:**

```yaml
volumes:
  - name: downward-vol
    downwardAPI:
      items:
        - path: cpu_limit      # File will be at /var/downward/cpu_limit
          resourceFieldRef:
            containerName: demo
            resource: limits.cpu
        - path: cpu_request    # File will be at /var/downward/cpu_request
          resourceFieldRef:
            containerName: demo
            resource: requests.cpu
```

**Container script that reads these files:**

```yaml
args:
  - -c
  - |
    while true; do
      echo "CPU limit: $(cat /var/downward/cpu_limit) | Request: $(cat /var/downward/cpu_request)" >> /proc/1/fd/1
      sleep 10
    done
```

**Check the files:**

```bash
kubectl exec -it <pod-name> -- ls /var/downward/
# Output: cpu_limit  cpu_request

kubectl exec -it <pod-name> -- cat /var/downward/cpu_limit
# Output: 500m
```

---

## Quick Reference — What can you inject?

### Via `fieldRef` (pod-level info)

```
metadata.name          → pod name
metadata.namespace     → namespace it's in
metadata.uid           → unique pod ID
status.podIP           → pod's IP address
status.hostIP          → the node's IP
spec.nodeName          → which node it's on
metadata.labels        → all labels (volume only, not env)
metadata.annotations   → all annotations (volume only, not env)
```

### Via `resourceFieldRef` (container resources)

```
limits.cpu       → CPU limit
limits.memory    → memory limit
requests.cpu     → CPU request
requests.memory  → memory request
```

---

## When things go wrong

| Symptom | What happened | Fix |
|---|---|---|
| `Read-only file system` | Tried to write to a downwardAPI volume | Use `emptyDir` for writes |
| `container name must be specified` | Missing `containerName` in `resourceFieldRef` | Add `containerName: <your-container>` |
| `MEM_LIMIT=134217728` instead of `128Mi` | Expected — memory is always in bytes | Parse bytes in your app |
| `CPU_LIMIT=0` or `MEM_LIMIT=0` | No `resources.limits` defined | Add limits to your container spec |
| Env var shows old value after label update | Env vars don't refresh — they're frozen at start | Use a volume file instead |
| `metadata.labels` rejected in env var | Labels are a map, can't be one env var | Use a downwardAPI volume for labels |

---

## Cleanup

```bash
kubectl delete -f downwardapi_env.yaml
kubectl delete -f downwardapi_resources.yaml
```

---

## The one-paragraph summary

The Downward API lets a container know about itself — its name, its IP, its CPU limit — without calling the Kubernetes API. You inject this info either as environment variables (simple, but frozen at start) or as files in a volume (slightly more setup, but they update automatically). Use `fieldRef` for pod-level info like name and IP. Use `resourceFieldRef` for CPU and memory — and always include `containerName` there. Memory comes in as bytes, CPU in millicores. The volume is always read-only. That's the whole thing.