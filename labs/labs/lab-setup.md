# KIND Kubernetes Lab Setup

## 1. Install KIND

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64

chmod +x ./kind

sudo mv ./kind /usr/local/bin/kind
```

Verify:

```bash
kind version
```

---

## 2. Create Multi-Node Cluster

Create config:

```bash
cat <<EOF > kind-cluster.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4

nodes:
- role: control-plane
- role: worker
- role: worker
EOF
```

Create cluster:

```bash
kind create cluster --config kind-cluster.yaml --name k8s-labs
```

---

## 3. Install kubectl

If `kubectl` is not installed, you'll see:

```text
-bash: kubectl: command not found
```

Install the latest stable version:

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

chmod +x kubectl

sudo mv kubectl /usr/local/bin/
```

Verify installation:

```bash
kubectl version --client
```

---

## 4. Verify Cluster

Check cluster access:

```bash
kubectl cluster-info --context kind-k8s-labs
```

List nodes:

```bash
kubectl get nodes
```

Expected:

```text
NAME                        STATUS
k8s-labs-control-plane      Ready
k8s-labs-worker             Ready
k8s-labs-worker2            Ready
```

---

## 5. Fix kubeconfig (If `sudo` Was Used)

KIND may write kubeconfig to root's home.

Create local kube directory:

```bash
mkdir -p ~/.kube
```

Copy config:

```bash
sudo cp /root/.kube/config ~/.kube/config
```

Fix ownership:

```bash
sudo chown $(whoami):$(whoami) ~/.kube/config
```

Test:

```bash
kubectl get nodes
```

You should now use `kubectl` without `sudo`.

---

## 6. Useful Verification Commands

Current context:

```bash
kubectl config current-context
```

Cluster info:

```bash
kubectl cluster-info
```

Nodes:

```bash
kubectl get nodes -o wide
```

System pods:

```bash
kubectl get pods -A
```

Docker containers (Kind nodes):

```bash
docker ps
```

List KIND clusters:

```bash
kind get clusters
```

---

## 7. Lab Environment Ready

Check:

```bash
kubectl get nodes
kubectl get pods -A
```

If all nodes are **Ready** and all system pods are **Running**, your Kubernetes lab environment is ready for:

- Pods
- Deployments
- ReplicaSets
- Services
- ConfigMaps
- Secrets
- Volumes
- Networking
- Ingress
- Troubleshooting
- Scheduling
- Resource Limits
- RBAC
- Stateful Workloads

🚀 Happy Labbing!

---

# 🧹 Clean Up

## 8. Delete Cluster

Delete the specific cluster:

```bash
kind delete cluster --name k8s-labs
```

Verify removal:

```bash
kind get clusters
```

Delete all KIND clusters:

```bash
kind get clusters | xargs -I{} kind delete cluster --name {}
```
