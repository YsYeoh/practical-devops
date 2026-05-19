# Chapter 13: Kubernetes Introduction & Core Concepts

This chapter introduces Kubernetes (K8s), explains why it extends what you learned with Docker Compose, and covers the core concepts you need to get started.

---

## What Is Kubernetes?

Kubernetes (often abbreviated as K8s) is an open-source platform for automating deployment, scaling, and management of containerized applications.

If Docker Compose manages containers on **one** machine, Kubernetes manages containers across **many** machines (a cluster).

| Docker Compose | Kubernetes |
|---|---|
| Single host | Multi-host cluster |
| Manual scaling | Auto-scaling |
| No self-healing | Self-healing (restarts failed containers) |
| Simple networking | Advanced service mesh / ingress |
| Great for dev/staging | Production-grade orchestration |
| Docker-native | Works with any container runtime |

---

## Why Move from Docker Compose to Kubernetes?

### The Scaling Problem

Docker Compose runs all containers on one server. When you need to:

- Handle 10x more traffic
- Survive a server failure
- Roll out updates without any downtime
- Run different environments (dev/staging/prod) on shared infrastructure

...a single-server setup reaches its limits. Kubernetes solves these problems with its cluster architecture.

### What Kubernetes Adds

| Feature | Benefit |
|---|---|
| **Cluster** | Multiple servers working as one |
| **Self-healing** | Replaces failed containers automatically |
| **Auto-scaling** | Adjusts replica count based on CPU/memory |
| **Rolling updates** | Zero-downtime deployments by default |
| **Service discovery** | Built-in DNS and load balancing |
| **Declarative config** | Desired state → K8s makes it happen |

---

## Kubernetes Architecture

```
┌────────────────────────────────────────────┐
│           Control Plane (Master)            │
│  ┌────────┐  ┌──────────┐  ┌────────────┐ │
│  │  API   │  │ Scheduler│  │Controller  │ │
│  │ Server │  │          │  │  Manager   │ │
│  └───┬────┘  └──────────┘  └────────────┘ │
│      │                                      │
│  ┌───┴────┐                                 │
│  │  etcd  │  (cluster database)             │
│  └────────┘                                 │
└──────────────────┬─────────────────────────┘
                   │
    ┌──────────────┼──────────────┐
    ▼              ▼              ▼
┌──────────┐ ┌──────────┐ ┌──────────┐
│  Node 1  │ │  Node 2  │ │  Node 3  │
│ ┌──────┐ │ │ ┌──────┐ │ │ ┌──────┐ │
│ │kubelet│ │ │ │kubelet│ │ │ │kubelet│ │
│ │ Pods  │ │ │ │ Pods  │ │ │ │ Pods  │ │
│ └──────┘ │ │ └──────┘ │ │ └──────┘ │
└──────────┘ └──────────┘ └──────────┘
```

### Control Plane Components

| Component | Role |
|---|---|
| **API Server** | Front door to the cluster. All interactions go through the API |
| **Scheduler** | Decides which node runs each new Pod |
| **Controller Manager** | Watches cluster state and makes corrections (e.g., restarts a failed Pod) |
| **etcd** | Distributed key-value store holding all cluster data |

### Worker Node Components

| Component | Role |
|---|---|
| **kubelet** | Agent running on each node. Ensures containers are running as expected |
| **kube-proxy** | Handles networking rules and load balancing for Services |
| **Container Runtime** | Actually runs containers (Docker, containerd, etc.) |

---

## Core Kubernetes Objects

These are the building blocks you will use every day with Kubernetes.

### Pod

The smallest deployable unit. A Pod wraps one or more containers.

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
    - name: my-app
      image: nginx:latest
      ports:
        - containerPort: 80
```

**Key insight:** You rarely create Pods directly. Higher-level objects (Deployments) manage Pods for you.

### Deployment

Manages a set of identical Pods. Handles scaling, rolling updates, and self-healing.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: nginx:latest
          ports:
            - containerPort: 80
```

### Service

Provides stable networking for Pods. Pods get replaced and their IPs change — Services give you a fixed IP and DNS name.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```

Service types:

| Type | Accessible From | Use Case |
|---|---|---|
| **ClusterIP** (default) | Inside the cluster only | Internal communication |
| **NodePort** | External (node IP + port) | Development / debug |
| **LoadBalancer** | External (cloud LB) | Production on cloud |
| **Ingress** (special) | External (via Ingress controller) | HTTP/HTTPS routing |

### ConfigMap & Secret

Store configuration data separately from container images.

| Object | Data Type | Storage |
|---|---|---|
| ConfigMap | Plain text, non-sensitive | Key-value pairs or files |
| Secret | Base64-encoded sensitive data | Key-value pairs or files |

### Namespace

Virtual cluster within a physical cluster. Used to separate environments, teams, or projects.

```bash
kubectl get namespaces
# default, kube-system, kube-public, kube-node-lease
```

---

## Docker Compose → Kubernetes Mapping

| Docker Compose | Kubernetes |
|---|---|
| `services.nextjs` | `Deployment` + `Service` |
| `services.mongodb` | `StatefulSet` + `Service` |
| `services.nginx` | `Ingress` or `Deployment` + `Service` |
| `volumes` | `PersistentVolumeClaim` |
| `networks` | `NetworkPolicy` |
| `env_file` / `environment` | `ConfigMap` / `Secret` |
| `depends_on` | `initContainers` or startup probes |
| `restart: always` | `Deployment` (self-healing by default) |
| `ports` | `Service.ports` + `Ingress` |

---

## kubectl: The Kubernetes CLI

`kubectl` is your primary tool for interacting with a Kubernetes cluster.

### Essential Commands

| Command | Purpose |
|---|---|
| `kubectl get pods` | List all Pods |
| `kubectl get deployments` | List all Deployments |
| `kubectl get services` | List all Services |
| `kubectl get nodes` | List cluster nodes |
| `kubectl apply -f file.yaml` | Create or update resources from a YAML file |
| `kubectl delete -f file.yaml` | Delete resources defined in a YAML file |
| `kubectl logs pod-name` | View logs from a Pod |
| `kubectl describe pod pod-name` | Detailed info about a Pod |
| `kubectl exec -it pod-name -- sh` | Run a shell inside a container |
| `kubectl get all` | List all resources in a namespace |

---

## Verification Checklist

Before moving on, confirm:

- [x] You understand the difference between Docker Compose and Kubernetes
- [x] You know the main control plane components (API Server, Scheduler, Controller Manager, etcd)
- [x] You can explain the relationship between Pods and Deployments
- [x] You understand why Services are needed (Pod IPs change)
- [x] You know how Docker Compose concepts map to Kubernetes objects

---

**Next:** [Chapter 14 — Setting Up a Local Kubernetes Cluster](14-setting-up-a-local-kubernetes-cluster.md)
