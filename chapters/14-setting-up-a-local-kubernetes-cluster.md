# Chapter 14: Setting Up a Local Kubernetes Cluster

In this chapter, you will install Minikube and kubectl to run a local Kubernetes cluster for learning and development.

---

## Choosing a Local K8s Tool

For learning, you have several options:

| Tool | Best For |
|---|---|
| **Minikube** | Single-node cluster, full K8s features, beginner-friendly |
| **kind** (Kubernetes in Docker) | Fast startup, CI/CD testing, multi-node clusters |
| **K3s** | Lightweight, production-capable, resource-friendly |
| **Docker Desktop** | Built-in K8s, convenient if you already use Docker Desktop |

This guide uses **Minikube** — it is the most feature-complete for learning, with add-ons for ingress, monitoring, and dashboards.

---

## Step 1: Install kubectl

`kubectl` is the command-line tool for interacting with any Kubernetes cluster.

### Linux (Ubuntu)

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

### macOS

```bash
brew install kubectl
```

### Windows

```powershell
curl.exe -LO "https://dl.k8s.io/release/v1.31.0/bin/windows/amd64/kubectl.exe"
```

Add `kubectl.exe` to your PATH.

### Verify

```bash
kubectl version --client
```

Expected output:

```
Client Version: v1.31.x
Kustomize Version: v5.x.x
```

---

## Step 2: Install Minikube

### Linux

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

### macOS

```bash
brew install minikube
```

### Windows

```powershell
choco install minikube
```

Or download the installer from the [Minikube releases page](https://github.com/kubernetes/minikube/releases).

### Verify

```bash
minikube version
```

Expected:

```
minikube version: v1.34.x
```

---

## Step 3: Start the Cluster

```bash
minikube start --cpus=2 --memory=4096
```

**What this does:** Creates a single-node Kubernetes cluster inside a VM (or Docker container, depending on your driver).

| Flag | Purpose |
|---|---|
| `--cpus=2` | Allocate 2 CPU cores |
| `--memory=4096` | Allocate 4 GB RAM |
| `--driver=docker` | (default on many systems) Runs K8s in a Docker container |
| `--driver=virtualbox` | Runs K8s in a VirtualBox VM (more isolated) |

> If you already have Docker installed, the `docker` driver is the quickest option.

### Check Cluster Status

```bash
minikube status
```

Expected:

```
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
```

### Verify Nodes

```bash
kubectl get nodes
```

Expected:

```
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   1m    v1.31.x
```

---

## Step 4: Explore the Cluster

### List All Namespaced Resources

```bash
kubectl get all --all-namespaces
```

This shows system Pods running in the cluster (CoreDNS, kube-proxy, storage provisioner, etc.).

### View Cluster Info

```bash
kubectl cluster-info
```

### View the Dashboard (Web UI)

```bash
minikube dashboard
```

This opens a web-based Kubernetes dashboard in your browser. You can view Pods, Deployments, Services, logs, and more.

---

## Step 5: Run Your First Pod

Deploy a simple Nginx Pod to confirm the cluster works:

```bash
kubectl create deployment hello-k8s --image=nginx:latest
kubectl expose deployment hello-k8s --type=NodePort --port=80
```

Check that the Pod is running:

```bash
kubectl get pods
```

Expected output (STATUS should be `Running`):

```
NAME                         READY   STATUS    RESTARTS   AGE
hello-k8s-7d4f8d6b5f-abc12   1/1     Running   0          30s
```

Access the Nginx page:

```bash
minikube service hello-k8s
```

This opens the Nginx welcome page in your browser.

Clean up:

```bash
kubectl delete service hello-k8s
kubectl delete deployment hello-k8s
```

---

## Step 6: Enable Minikube Add-ons

Minikube comes with useful add-ons that extend cluster capabilities. Enable these for later chapters:

```bash
# Ingress controller (for HTTP routing — Chapter 18)
minikube addons enable ingress

# Metrics server (for autoscaling — Chapter 20)
minikube addons enable metrics-server

# Storage provisioner (for persistent volumes — Chapter 17)
minikube addons enable storage-provisioner
```

View all available add-ons:

```bash
minikube addons list
```

---

## Step 7: Set kubectl Context

Minikube automatically configures `kubectl` to point to your Minikube cluster. Verify:

```bash
kubectl config current-context
```

Expected: `minikube`

If you ever need to switch back to Minikube:

```bash
kubectl config use-context minikube
```

---

## Step 8: Stop and Delete the Cluster

When you are done working:

```bash
# Stop the cluster (preserves VMs and data)
minikube stop

# Start it again later
minikube start
```

To completely delete the cluster and all data:

```bash
minikube delete
```

---

## Minikube Command Reference

| Command | Purpose |
|---|---|
| `minikube start` | Start cluster |
| `minikube stop` | Stop cluster (preserves state) |
| `minikube delete` | Delete cluster (all data lost) |
| `minikube status` | Check cluster health |
| `minikube dashboard` | Open web UI |
| `minikube addons enable <name>` | Enable an add-on |
| `minikube service <name>` | Expose a service via tunnel |
| `minikube ip` | Get cluster IP address |
| `minikube ssh` | SSH into the cluster node |

---

## Verification Checklist

Before moving on, confirm:

- [x] `kubectl version --client` works
- [x] `minikube version` works
- [x] `minikube start` creates a cluster successfully
- [x] `kubectl get nodes` shows one Ready node
- [x] You ran a test Nginx Pod and accessed it via `minikube service`
- [x] The `ingress`, `metrics-server`, and `storage-provisioner` add-ons are enabled

---

**Next:** [Chapter 15 — Pods, Deployments & Services](15-pods-deployments-and-services.md)
