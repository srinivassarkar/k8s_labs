# Lab 01 — Pod Troubleshooting

## What This Lab Is About

Pods are the heartbeat of Kubernetes. Everything runs inside a pod — and when something breaks in prod, the first thing you're staring at is a pod in a bad state. This lab covers the 7 most common pod failure modes you will encounter in real clusters. Not theory — actual broken states, actual symptoms, actual fixes.

The goal is not just to fix it. The goal is to develop an **instinct** — the moment you see a status, you already know the next 3 commands to run. That instinct only comes from repetition.

> "I fear not the man who has practiced 10,000 kicks once, but I fear the man who has practiced one kick 10,000 times." — Bruce Lee

---

## Environment

- **Cluster:** KIND (Kubernetes IN Docker)
- **K8s Version:** v1.27.3
- **Node:** Single node (`kind-control-plane`)
- **Namespace for all labs:** `lab01` (create it before starting)

```bash
kubectl create namespace lab01
kubectl config set-context --current --namespace=lab01
```

---

## The 7 Scenarios

| # | Failure Mode | What You'll Learn |
|---|---|---|
| 01 | `ImagePullBackOff` | Wrong image name or tag |
| 02 | `CrashLoopBackOff` | App crashes on start |
| 03 | `Pending` — No Resources | Pod can't be scheduled |
| 04 | `Pending` — Missing ConfigMap | Volume mount failure |
| 05 | `OOMKilled` | Memory limit too low |
| 06 | `CreateContainerConfigError` | Bad env var from Secret |
| 07 | `RunContainerError` — Bad Command | Wrong entrypoint |

---

## Scenario 01 — `ImagePullBackOff`

### What You'll Break

A pod referencing an image that doesn't exist. This is one of the most frequent failures in real environments — typos in image names, wrong tags after a release, private registry credentials missing.

### Apply the Broken State

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: bad-image-pod
  namespace: lab01
spec:
  containers:
  - name: app
    image: nginx:thisdoesnotexist999
EOF
```

### Symptoms You Will Observe

```bash
kubectl get pod bad-image-pod -n lab01
```

You will see:
```
NAME           READY   STATUS             RESTARTS   AGE
bad-image-pod  0/1     ImagePullBackOff   0          30s
```

Sometimes it shows `ErrImagePull` first, then transitions to `ImagePullBackOff`. This is K8s backing off the retry interval — it starts fast, then slows down exponentially.

### Investigate

Run these in order. Build the habit:

```bash
# Step 1 — What is the status?
kubectl get pod bad-image-pod -n lab01

# Step 2 — What are the events saying?
kubectl describe pod bad-image-pod -n lab01

# Step 3 — Look specifically at Events section at the bottom of describe output
# You will see: Failed to pull image "nginx:thisdoesnotexist999": rpc error...
```

The `describe` output is your best friend here. Scroll to the **Events** section at the bottom — it will tell you exactly what failed and why.

### Fix

```bash
# Delete the broken pod
kubectl delete pod bad-image-pod -n lab01

# Apply with correct image
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: bad-image-pod
  namespace: lab01
spec:
  containers:
  - name: app
    image: nginx:1.25
EOF
```

Verify:
```bash
kubectl get pod bad-image-pod -n lab01
# Should show Running
```
### Possible Qs to Ask yourself : 
1. Is the image name/tag correct?
2. Is the registry reachable?
3. Is authentication required?
4. Is the CRI healthy?
5. Is the node healthy (disk, network, resources)?
6. Is imagePullPolicy preventing the pull?

### Prod Wisdom

In real environments `ImagePullBackOff` is also caused by **missing imagePullSecrets** when pulling from a private registry (ECR, GCR, ACR, Docker Hub private). Always check: is the image name right? Is the tag right? Does the ServiceAccount have the right pull secret attached?

---

## Scenario 02 — `CrashLoopBackOff`

### What You'll Break

A pod that starts, crashes immediately, starts again, crashes again — infinitely. This happens when the application itself exits with a non-zero code right after starting. Common causes: missing env vars the app expects, wrong startup command, app bug.

### Apply the Broken State

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: crash-pod
  namespace: lab01
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ["sh", "-c", "echo starting && exit 1"]
EOF
```

### Symptoms You Will Observe

```bash
kubectl get pod crash-pod -n lab01
```

Output progression:
```
NAME        READY   STATUS             RESTARTS   AGE
crash-pod   0/1     CrashLoopBackOff   3          45s
```

The `RESTARTS` count keeps climbing. The backoff delay increases: 10s → 20s → 40s → 80s → 160s → capped at 300s (5 minutes). In prod, a pod stuck at `RESTARTS: 15` is a serious alert.

### Investigate

```bash
# Step 1 — Current status and restart count
kubectl get pod crash-pod -n lab01

# Step 2 — Read the logs (this is KEY — logs survive the crash)
kubectl logs crash-pod -n lab01

# Step 3 — Read previous container logs (what did it print before dying?)
kubectl logs crash-pod -n lab01 --previous

# Step 4 — Describe for events
kubectl describe pod crash-pod -n lab01
```

The `--previous` flag is critical. When a container is restarting, `kubectl logs` shows the current (possibly empty) run. `--previous` shows the last run that crashed — that's where the error message lives.

### Fix

```bash
kubectl delete pod crash-pod -n lab01

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: crash-pod
  namespace: lab01
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ["sh", "-c", "echo starting && sleep 3600"]
EOF
```

Verify:
```bash
kubectl get pod crash-pod -n lab01
# Should show Running with RESTARTS: 0
```

### Prod Wisdom

When you see `CrashLoopBackOff` in prod, **always check logs with `--previous` first** before anything else. If the app printed a useful error before dying, that's your answer in 10 seconds. Also check if there's a `readinessProbe` or `livenessProbe` that might be killing the container before the app is ready — that's a very common trap covered in Lab 04.

---

## Scenario 03 — `Pending` — No Resources

### What You'll Break

A pod requesting more CPU or memory than any node in the cluster can provide. The scheduler cannot place it anywhere, so it stays in `Pending` forever.

### Apply the Broken State

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: too-hungry-pod
  namespace: lab01
spec:
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        memory: "100Gi"
        cpu: "50"
EOF
```

### Symptoms You Will Observe

```bash
kubectl get pod too-hungry-pod -n lab01
```

```
NAME             READY   STATUS    RESTARTS   AGE
too-hungry-pod   0/1     Pending   0          2m
```

Note: `Pending` with `RESTARTS: 0` and no progression. The pod is just sitting there. This is different from `CrashLoopBackOff` — the container never even starts.

### Investigate

```bash
# Step 1 — Confirm it's Pending
kubectl get pod too-hungry-pod -n lab01

# Step 2 — Describe to find the scheduler message
kubectl describe pod too-hungry-pod -n lab01

# Look for this in Events:
# Warning  FailedScheduling  default-scheduler  0/1 nodes are available:
# 1 Insufficient memory, 1 Insufficient cpu.

# Step 3 — Check what nodes actually have available
kubectl describe nodes | grep -A 5 "Allocated resources"

# Step 4 — Check node capacity
kubectl get nodes -o custom-columns="NAME:.metadata.name,CPU:.status.capacity.cpu,MEMORY:.status.capacity.memory"
```

### Fix

```bash
kubectl delete pod too-hungry-pod -n lab01

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: too-hungry-pod
  namespace: lab01
spec:
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        memory: "64Mi"
        cpu: "100m"
      limits:
        memory: "128Mi"
        cpu: "200m"
EOF
```

Verify:
```bash
kubectl get pod too-hungry-pod -n lab01
# Should show Running
```

### Prod Wisdom

In real clusters with multiple nodes, the scheduler message tells you **exactly** how many nodes failed and why — `0/5 nodes available: 3 Insufficient memory, 2 node(s) had taint that the pod didn't tolerate`. Read it carefully. `Pending` with scheduling failure is different from `Pending` waiting for PVC binding — `describe` always tells you which one it is.

---

## Scenario 04 — `Pending` — Missing ConfigMap

### What You'll Break

A pod that references a ConfigMap as a volume mount, but the ConfigMap doesn't exist. The pod stays in `Pending` — it never even gets to the scheduling phase properly because the volume can't be set up.

### Apply the Broken State

```bash
# Apply the pod WITHOUT creating the ConfigMap first
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: missing-config-pod
  namespace: lab01
spec:
  containers:
  - name: app
    image: nginx:1.25
    volumeMounts:
    - name: config-vol
      mountPath: /etc/config
  volumes:
  - name: config-vol
    configMap:
      name: app-config-that-doesnt-exist
EOF
```

### Symptoms You Will Observe

```bash
kubectl get pod missing-config-pod -n lab01
```

```
NAME                 READY   STATUS    RESTARTS   AGE
missing-config-pod   0/1     Pending   0          1m
```

Looks identical to resource starvation from `get` output. You must `describe` to find the difference.

### Investigate

```bash
# Step 1
kubectl get pod missing-config-pod -n lab01

# Step 2 — Describe reveals the real reason
kubectl describe pod missing-config-pod -n lab01

# In Events you will see:
# Warning  FailedMount  kubelet  MountVolume.SetUp failed for volume "config-vol":
# configmap "app-config-that-doesnt-exist" not found
```

This is a very important distinction — two pods both `Pending`, two completely different root causes. `describe` is the separator.

### Fix

```bash
# First create the missing ConfigMap
kubectl create configmap app-config-that-doesnt-exist \
  --from-literal=APP_ENV=production \
  --from-literal=LOG_LEVEL=info \
  -n lab01
```

Watch the pod recover automatically — no need to delete and recreate:

```bash
kubectl get pod missing-config-pod -n lab01 -w
# Watch it transition from Pending → ContainerCreating → Running
```

### Prod Wisdom

`Pending` has multiple root causes. Always `describe`. The three most common `Pending` reasons in prod are: insufficient resources on nodes, missing volumes/secrets/configmaps, and node taints the pod doesn't tolerate. A junior engineer restarts the pod. A senior engineer reads the Events section first.

---

## Scenario 05 — `OOMKilled`

### What You'll Break

A pod with a memory limit that's too low for what the application actually needs. The Linux kernel OOM killer terminates the process. K8s reports this as `OOMKilled`.

### Apply the Broken State

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: oom-pod
  namespace: lab01
spec:
  containers:
  - name: memory-hog
    image: polinux/stress:latest
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
    resources:
      limits:
        memory: "50Mi"
EOF
```

This container tries to allocate 150MB but the limit is 50MB. The kernel will kill it.

### Symptoms You Will Observe

```bash
kubectl get pod oom-pod -n lab01
```

```
NAME      READY   STATUS      RESTARTS   AGE
oom-pod   0/1     OOMKilled   0          5s
```

It may briefly show `OOMKilled` and then transition to `CrashLoopBackOff` as K8s restarts it. The key signal is `OOMKilled` in the status or in the describe output.

### Investigate

```bash
# Step 1 — Check status
kubectl get pod oom-pod -n lab01

# Step 2 — Describe shows exit code 137 (OOM signal)
kubectl describe pod oom-pod -n lab01

# Look for:
# Last State: Terminated
#   Reason: OOMKilled
#   Exit Code: 137

# Step 3 — Check resource limits currently set
kubectl get pod oom-pod -n lab01 -o jsonpath='{.spec.containers[0].resources}'
```

Exit code 137 = killed by signal 9 (SIGKILL) = OOM. This is a hard K8s rule — when memory limit is breached, the kernel kills it immediately, no grace period.

### Fix

```bash
kubectl delete pod oom-pod -n lab01

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: oom-pod
  namespace: lab01
spec:
  containers:
  - name: memory-hog
    image: polinux/stress:latest
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
    resources:
      requests:
        memory: "100Mi"
      limits:
        memory: "200Mi"
EOF
```

### Prod Wisdom

`OOMKilled` in prod is often not obvious at first — the pod shows as `CrashLoopBackOff` and you have to read `describe` to find `Reason: OOMKilled` in the `Last State` section. Always check `Last State` when you see high restart counts. And remember: **requests** are what the scheduler uses to place the pod; **limits** are what the kernel enforces at runtime. Getting these numbers right for your app in prod is a continuous tuning exercise.

---

## Scenario 06 — `CreateContainerConfigError`

### What You'll Break

A pod referencing a Secret in an environment variable that doesn't exist. K8s cannot construct the container config because a required Secret is missing.

### Apply the Broken State

```bash
# Apply pod WITHOUT creating the Secret
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: bad-secret-pod
  namespace: lab01
spec:
  containers:
  - name: app
    image: nginx:1.25
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
EOF
```

### Symptoms You Will Observe

```bash
kubectl get pod bad-secret-pod -n lab01
```

```
NAME            READY   STATUS                       RESTARTS   AGE
bad-secret-pod  0/1     CreateContainerConfigError   0          10s
```

Unlike `Pending`, this pod was scheduled to a node and the kubelet tried to start it — but couldn't build the container config because the Secret reference resolved to nothing.

### Investigate

```bash
# Step 1
kubectl get pod bad-secret-pod -n lab01

# Step 2 — Describe for the error message
kubectl describe pod bad-secret-pod -n lab01

# Events will show:
# Warning  Failed  kubelet  Error: secret "db-credentials" not found

# Step 3 — Confirm the secret doesn't exist
kubectl get secret db-credentials -n lab01
# Error from server (NotFound): secrets "db-credentials" not found
```

### Fix

```bash
# Create the missing Secret
kubectl create secret generic db-credentials \
  --from-literal=password=supersecretpassword123 \
  -n lab01
```

The pod will automatically recover — watch it:

```bash
kubectl get pod bad-secret-pod -n lab01 -w
# Transitions: CreateContainerConfigError → Running
```

### Prod Wisdom

`CreateContainerConfigError` is always a missing or misconfigured Secret or ConfigMap referenced via `envFrom` or `env.valueFrom`. The pattern is: pod is scheduled (unlike `Pending`), but kubelet can't start the container. Always check what the pod is referencing and whether those resources exist in the **same namespace**. Cross-namespace references for Secrets don't work in K8s — a Secret in `default` is invisible to a pod in `production`.

---

## Scenario 07 — `RunContainerError` — Bad Command

### What You'll Break

A pod with a command pointing to a binary that doesn't exist in the container image. The container runtime cannot start the process.

### Apply the Broken State

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: bad-command-pod
  namespace: lab01
spec:
  containers:
  - name: app
    image: nginx:1.25
    command: ["/bin/this-binary-does-not-exist"]
EOF
```

### Symptoms You Will Observe

```bash
kubectl get pod bad-command-pod -n lab01
```

```
NAME              READY   STATUS              RESTARTS   AGE
bad-command-pod   0/1     RunContainerError   0          5s
```

Or sometimes it shows as `Error` or `CrashLoopBackOff` depending on K8s version.

### Investigate

```bash
# Step 1
kubectl get pod bad-command-pod -n lab01

# Step 2
kubectl describe pod bad-command-pod -n lab01

# Events will show:
# Warning  Failed  kubelet  Error: failed to create containerd task:
# failed to create shim task: OCI runtime create failed:
# container_linux.go: starting container process caused: exec: "/bin/this-binary-does-not-exist":
# stat /bin/this-binary-does-not-exist: no such file or directory

# Step 3 — Confirm what commands DO exist in this image
kubectl run debug-check --image=nginx:1.25 --rm -it --restart=Never -- ls /bin
```

### Fix

```bash
kubectl delete pod bad-command-pod -n lab01

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: bad-command-pod
  namespace: lab01
spec:
  containers:
  - name: app
    image: nginx:1.25
    command: ["nginx", "-g", "daemon off;"]
EOF
```

Verify:
```bash
kubectl get pod bad-command-pod -n lab01
# Should show Running
```

### Prod Wisdom

`RunContainerError` with exec/binary errors almost always means one of two things in prod: the command or entrypoint path is wrong for the image, or the image was updated and the binary location changed. Always validate by running the image interactively first — `kubectl run debug --image=<image> --rm -it --restart=Never -- sh` — before writing the deployment YAML. This one mistake has caused many "why did prod break after an image update" incidents.

---

## Key Commands Reference — Lab 01

```bash
# The diagnostic flow — run in this order every time
kubectl get pod <pod-name> -n <namespace>
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous

# Watch a pod in real time
kubectl get pod <pod-name> -n <namespace> -w

# Check all pods in namespace
kubectl get pods -n lab01

# Get pod details as JSON/YAML (for deep inspection)
kubectl get pod <pod-name> -n lab01 -o yaml
kubectl get pod <pod-name> -n lab01 -o jsonpath='{.status}'

# Check node resources
kubectl describe nodes | grep -A 5 "Allocated resources"

# Run a debug container (ephemeral)
kubectl run debug --image=busybox:1.35 --rm -it --restart=Never -- sh

# Delete all pods in namespace (cleanup)
kubectl delete pods --all -n lab01
```

---

## Prod Wisdom — The Senior Engineer Mindset

Three things separate a senior K8s engineer from a junior one in incident response:

**1. They always read Events first.** `kubectl describe` → scroll to the bottom → read Events. This single habit solves 70% of pod issues in under 60 seconds.

**2. They use `--previous` on logs.** When a container is restarting, the error is in the *previous* run's logs, not the current one.

**3. They know the difference between `Pending` failure modes.** Resource starvation, missing volumes, and missing secrets all look the same at `kubectl get` level. `describe` separates them in 5 seconds.

The diagnostic order is always the same regardless of the symptom:
```
get → describe → logs → logs --previous → exec (if running)
```

Burn this order into your muscle memory. In a prod incident at 2am, you want zero thinking — just execution.

---

## Cleanup

```bash
kubectl delete namespace lab01
```

---

*Lab 01 complete. Move to Lab 02 — Deployments & Rollouts when ready.*
