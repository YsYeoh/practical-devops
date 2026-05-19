# Chapter 15: Pods, Deployments & Services

In this chapter, you will learn how to run and connect applications in Kubernetes using Pods, Deployments, and Services — the three most essential objects.

---

## Part 1: Pods — The Smallest Unit

A Pod is one or more containers that share a network and storage. In practice, most Pods contain a single container.

### Create a Pod Directly

Create `pod-nginx.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx:latest
      ports:
        - containerPort: 80
```

Apply it:

```bash
kubectl apply -f pod-nginx.yaml
```

Check it:

```bash
kubectl get pods
kubectl describe pod nginx-pod
```

**Why you rarely create Pods directly:** If the node running this Pod fails, the Pod is gone forever. Deployments fix this by managing Pods with self-healing.

---

## Part 2: Deployments — Managed Pods

A Deployment ensures a desired number of identical Pods are always running. It handles:
- **Self-healing** — replaces failed Pods
- **Rolling updates** — updates Pods without downtime
- **Scaling** — increases or decreases replicas

### Create a Deployment

Create `deployment-nextjs.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nextjs
  labels:
    app: nextjs
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nextjs
  template:
    metadata:
      labels:
        app: nextjs
    spec:
      containers:
        - name: nextjs
          image: nginx:latest
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "250m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
```

Apply it:

```bash
kubectl apply -f deployment-nextjs.yaml
```

### Understanding the Deployment YAML

| Field | Purpose |
|---|---|
| `replicas: 2` | Run 2 identical Pods |
| `selector.matchLabels` | Tells the Deployment which Pods it manages |
| `template` | Defines the Pod template (labels + container spec) |
| `resources.requests` | Minimum CPU/memory guaranteed to each Pod |
| `resources.limits` | Maximum CPU/memory each Pod can use |

### Verify the Deployment

```bash
kubectl get deployments
kubectl get pods
kubectl get replicasets
```

You should see 2 Pods with `STATUS: Running`.

### Scale the Deployment

```bash
kubectl scale deployment nextjs --replicas=3
```

Check that a third Pod was created:

```bash
kubectl get pods
```

### Update the Image

When you push a new version of your app, update the Deployment:

```bash
kubectl set image deployment/nextjs nextjs=my-app:v2
```

Or edit the YAML directly:

```bash
kubectl edit deployment nextjs
```

Kubernetes performs a **rolling update** — Pods are replaced one by one, so the app stays available.

### Rollback a Deployment

If the update fails:

```bash
kubectl rollout undo deployment/nextjs
```

View rollout history:

```bash
kubectl rollout history deployment/nextjs
```

---

## Part 3: Services — Stable Networking

Pods are ephemeral — they get created, destroyed, and their IPs change. A Service provides a stable IP and DNS name that routes traffic to healthy Pods.

### Service Types

| Type | Access | Use Case |
|---|---|---|
| **ClusterIP** | Internal cluster only | Backend-to-backend communication |
| **NodePort** | External via `<NodeIP>:<Port>` | Development / debugging |
| **LoadBalancer** | External via cloud LB | Production on cloud |

### Create a ClusterIP Service

Create `service-nextjs.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nextjs-service
spec:
  selector:
    app: nextjs
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

Apply it:

```bash
kubectl apply -f service-nextjs.yaml
```

**How it works:** The Service selects Pods with `app: nextjs` label and load-balances traffic across them.

### Test the Service from Inside the Cluster

```bash
# Run a temporary Pod to test from within the cluster
kubectl run test-pod --rm -it --image=busybox -- /bin/sh

# Inside the container, curl the service by name
wget -qO- http://nextjs-service
```

DNS resolution works automatically: `nextjs-service` resolves to the Service's ClusterIP.

### Create a NodePort Service (External Access)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nextjs-nodeport
spec:
  type: NodePort
  selector:
    app: nextjs
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

Access it:

```bash
# Get the Minikube IP
minikube ip

# Visit in browser or curl
curl http://192.168.49.2:30080
```

> NodePort exposes the service on every node at the specified port (30080 in this case). Ports must be in the range 30000-32767.

---

## Part 4: Labels and Selectors

Labels are key-value pairs attached to Kubernetes objects. Selectors filter objects by labels.

```yaml
metadata:
  labels:
    app: nextjs
    environment: production
    tier: frontend
```

Use labels to:

- Target Pods with Services (as shown above)
- Group resources for monitoring
- Implement blue-green deployments (different labels for blue vs green)
- Organize by environment, team, or cost center

```bash
# List Pods matching a label
kubectl get pods -l app=nextjs

# List Pods matching multiple labels
kubectl get pods -l "app=nextjs,environment=production"
```

---

## Part 5: Liveness and Readiness Probes

Kubernetes uses probes to know when a container is alive and ready to serve traffic.

```yaml
spec:
  containers:
    - name: nextjs
      image: nginx:latest
      ports:
        - containerPort: 80
      livenessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 10
        periodSeconds: 10
      readinessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 5
```

| Probe | Purpose |
|---|---|
| **livenessProbe** | Is the container alive? If it fails, K8s restarts the container |
| **readinessProbe** | Is the container ready to serve traffic? If it fails, K8s removes it from the Service |
| **startupProbe** | For slow-starting containers. Defers liveness/readiness until startup is done |

---

## Full Example: NextJS Deployment + Service

Save these as `deployment-and-service.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nextjs
  labels:
    app: nextjs
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nextjs
  template:
    metadata:
      labels:
        app: nextjs
    spec:
      containers:
        - name: nextjs
          image: node:20-alpine
          command: ["sh", "-c", "echo 'Hello K8s' > index.html && python3 -m http.server 3000"]
          ports:
            - containerPort: 3000
          resources:
            requests:
              cpu: "250m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
          livenessProbe:
            httpGet:
              path: /
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: nextjs-service
spec:
  selector:
    app: nextjs
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
  type: ClusterIP
```

---

## Verification Checklist

Before moving on, confirm:

- [x] You can create a Deployment with 2+ replicas
- [x] Pods are running and self-heal if deleted (`kubectl delete pod <name>` — a new one replaces it)
- [x] You can scale a Deployment up and down
- [x] A Service routes traffic to Pods by label selector
- [x] You can access the app from inside the cluster via the Service DNS name
- [x] You understand the difference between liveness and readiness probes

---

**Next:** [Chapter 16 — ConfigMaps, Secrets & Configuration](16-configmaps-secrets-and-configuration.md)
