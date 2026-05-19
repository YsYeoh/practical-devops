# Chapter 20: Production Practices, CI/CD & Next Steps

This chapter covers the production practices that make Kubernetes reliable at scale — resource management, autoscaling, RBAC, and how to integrate K8s with your Jenkins CI/CD pipeline.

---

## Part 1: Resource Requests and Limits

Every container should specify how much CPU and memory it needs.

```yaml
resources:
  requests:
    cpu: "250m"     # 0.25 CPU cores
    memory: "128Mi"  # 128 MB
  limits:
    cpu: "500m"     # 0.5 CPU cores
    memory: "256Mi"  # 256 MB
```

| Term | Meaning |
|---|---|
| **requests** | Minimum resources guaranteed to the container. Used by the scheduler to decide which node to place the Pod on |
| **limits** | Maximum resources the container can use. If exceeded, the container may be throttled (CPU) or killed (memory) |

### Why They Matter

| Without requests/limits | With requests/limits |
|---|---|
| One app can starve others | Each app gets guaranteed resources |
| Node can be overloaded | Scheduler places Pods wisely |
| Unclear if the node is overcommitted | Cluster operator can see utilization |

### CPU Units

| Value | Meaning |
|---|---|
| `1` or `1000m` | 1 full CPU core |
| `500m` | Half a CPU core |
| `250m` | Quarter of a CPU core |
| `100m` | 0.1 CPU core |

### Memory Units

| Value | Meaning |
|---|---|
| `128M` or `128Mi` | 128 megabytes |
| `1G` or `1Gi` | 1 gigabyte |
| `512Mi` | 512 megabytes |

---

## Part 2: Horizontal Pod Autoscaling (HPA)

HPA automatically increases or decreases the number of Pod replicas based on CPU, memory, or custom metrics.

### Enable the Metrics Server

The HPA needs the metrics server to know resource usage. You enabled this in Chapter 14:

```bash
minikube addons enable metrics-server
```

Verify:

```bash
kubectl top pods
```

Expected output shows CPU and memory usage for each Pod.

### Create an HPA

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nextjs-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nextjs
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

Apply it:

```bash
kubectl apply -f hpa.yaml
```

Check the HPA:

```bash
kubectl get hpa
```

### How Autoscaling Works

```
CPU usage > 70% for 5 minutes → HPA adds replicas
CPU usage < 50% for 5 minutes → HPA removes replicas
```

The target replica count is calculated as:

```
desiredReplicas = ceil[currentReplicas × (currentMetricValue / targetMetricValue)]
```

### Generate Load to Test

```bash
# Run a load generator Pod
kubectl run load-test --rm -it --image=busybox -- /bin/sh

# Inside the container
while true; do
  wget -qO- http://nextjs-service
done
```

In another terminal, watch the HPA react:

```bash
kubectl get hpa -w
```

---

## Part 3: Namespaces — Organizing the Cluster

Namespaces partition a single cluster into multiple virtual clusters.

```bash
# List namespaces
kubectl get namespaces

# Create a namespace
kubectl create namespace staging
kubectl create namespace production

# Work within a namespace
kubectl get pods -n staging
kubectl apply -f deployment.yaml -n production

# Set a namespace permanently (context)
kubectl config set-context --current --namespace=staging
```

### Common Namespace Use Cases

| Namespace | Purpose |
|---|---|
| `default` | Default workspace (avoid using for production) |
| `kube-system` | System Pods (CoreDNS, etc.) |
| `kube-public` | Publicly readable config maps |
| `ingress-nginx` | Ingress controller Pods |
| `monitoring` | Prometheus, Grafana |

### Why Namespaces Matter

- **Isolation** — Resources in different namespaces do not conflict
- **Resource quotas** — Limit total CPU/memory per namespace
- **RBAC** — Control who can access each namespace
- **Organization** — Group resources by team, environment, or project

---

## Part 4: RBAC — Access Control Basics

RBAC (Role-Based Access Control) controls who can do what in the cluster.

> For a learning cluster, RBAC is not critical. But in a team environment, it is essential.

### Key RBAC Objects

| Object | Purpose |
|---|---|
| **Role** | Defines permissions within a namespace |
| **ClusterRole** | Defines permissions cluster-wide |
| **RoleBinding** | Binds a Role to a user/service account in a namespace |
| **ClusterRoleBinding** | Binds a ClusterRole cluster-wide |
| **ServiceAccount** | Identity for a Pod (not a human user) |

### Example: Read-Only Role for Developers

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: staging
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log", "services"]
    verbs: ["get", "list", "watch"]
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: staging
  name: read-pods
subjects:
  - kind: User
    name: developer@example.com
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### Common Verbs

| Verb | Permission |
|---|---|
| `get` | Read a specific resource |
| `list` | List resources |
| `watch` | Watch for changes (streaming) |
| `create` | Create a resource |
| `update` | Update a resource |
| `patch` | Partially update a resource |
| `delete` | Delete a resource |

---

## Part 5: CI/CD for Kubernetes

Now integrate your Jenkins pipeline to deploy to Kubernetes instead of Docker Compose.

### Jenkins + kubectl Setup

On your Jenkins server (or CI runner), install kubectl:

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

Copy the kubeconfig so Jenkins can authenticate:

```bash
# If Jenkins is on the same node as kubectl
mkdir -p /var/lib/jenkins/.kube
sudo cp ~/.kube/config /var/lib/jenkins/.kube/
sudo chown -R jenkins:jenkins /var/lib/jenkins/.kube
```

> In production, use a ServiceAccount token instead of copying the admin kubeconfig.

### Jenkins Pipeline for K8s Deployment

```groovy
pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "myapp:${BUILD_NUMBER}"
        K8S_NAMESPACE = "production"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'YOUR_GITHUB_REPO'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${DOCKER_IMAGE} ."
            }
        }

        stage('Push to Registry') {
            steps {
                sh """
                    docker tag ${DOCKER_IMAGE} registry.example.com/${DOCKER_IMAGE}
                    docker push registry.example.com/${DOCKER_IMAGE}
                """
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh """
                    kubectl set image deployment/nextjs \
                      nextjs=registry.example.com/${DOCKER_IMAGE} \
                      --namespace=${K8S_NAMESPACE}
                """
            }
        }

        stage('Verify Deployment') {
            steps {
                sh """
                    kubectl rollout status deployment/nextjs \
                      --namespace=${K8S_NAMESPACE} \
                      --timeout=120s
                """
            }
        }

        stage('Cleanup') {
            steps {
                sh 'docker image prune -af'
            }
        }
    }
}
```

### Blue-Green Deployment on K8s

Kubernetes supports rolling updates by default, but for true blue-green:

1. Create two Deployments: `nextjs-blue` and `nextjs-green`
2. Update the Service selector to point to the active deployment:
   ```bash
   kubectl patch service nextjs -p '{"spec":{"selector":{"version":"green"}}}'
   ```

Or use a tool like **Argo Rollouts** for advanced deployment strategies (blue-green, canary, A/B testing).

---

## Part 6: Production Readiness Checklist

| Practice | Why | Implemented? |
|---|---|---|
| Resource requests and limits | Prevents resource starvation | |
| Liveness and readiness probes | Self-healing and traffic control | |
| Pod Disruption Budgets | Prevent all Pods going down during maintenance | |
| Horizontal Pod Autoscaling | Handles traffic spikes | |
| Namespace isolation | Separate environments | |
| RBAC | Least-privilege access | |
| Network Policies | Restrict Pod-to-Pod traffic | |
| PersistentVolume backup | Backup database volumes | |
| Pod Security Standards | Restrict privileged containers | |
| Resource quotas per namespace | Fair resource allocation | |
| ImagePullSecrets for private registries | Access private Docker images | |

---

## Part 7: Kubernetes Learning Path

After completing this guide, continue with:

### Intermediate

| Topic | Why |
|---|---|
| **Custom Resource Definitions (CRDs)** | Extend Kubernetes with custom resources |
| **Operators** | Automate complex app management (e.g., Prometheus Operator) |
| **cert-manager** | Automatic TLS certificate management |
| **ArgoCD / Flux** | GitOps — deploy from Git automatically |
| **Service Mesh (Istio/Linkerd)** | Advanced traffic management, security, observability |

### Advanced

| Topic | Why |
|---|---|
| **Cluster Networking (CNI)** | Calico, Cilium, eBPF-based networking |
| **Multi-cluster management** | Deploy across multiple clusters |
| **Kubernetes Security** | Falco, OPA/Gatekeeper, Kyverno |
| **GPU scheduling** | Running AI workloads on K8s |
| **KEDA** | Event-driven autoscaling |

### Cloud-Native Ecosystem

| Tool | Purpose |
|---|---|
| **Prometheus + Grafana** | Monitoring (deeper dive) |
| **OpenTelemetry** | Distributed tracing |
| **Kustomize** | Native Kubernetes configuration customization |
| **Crossplane** | Infrastructure provisioning from Kubernetes |
| **Knative** | Serverless on Kubernetes |

---

## Verification Checklist

Before concluding, confirm:

- [x] You can create resource requests and limits for containers
- [x] HPA automatically scales Pods based on CPU usage
- [x] You understand how namespaces isolate environments
- [x] You can deploy to Kubernetes from a Jenkins pipeline
- [x] You know what to learn next (CRDs, Operators, ArgoCD, etc.)

---

## The Full Stack: Docker Compose → Kubernetes

```
Docker Compose                      Kubernetes
==============                      ==========
Single server                       Multi-node cluster
Manual management                   Self-healing (Deployments)
Basic compose networking            Advanced networking (Services, Ingress, NetworkPolicies)
Named volumes                       PersistentVolumes + StatefulSets
.env files                          ConfigMaps + Secrets
docker compose up -d                helm install + kubectl apply
docker compose logs -f              kubectl logs + Prometheus/Grafana
```

Both tools coexist in practice:
- **Docker Compose** for local development and testing
- **Kubernetes** for production, staging, and scale

You now know the foundations of both. Every concept from Docker Compose mapped directly to a Kubernetes equivalent — and the architecture knowledge transfers directly to cloud platforms (EKS, GKE, AKS) and production AI infrastructure.

---

## Appendix: All Kubernetes Commands from Chapters 13-20

```
kubectl get nodes | pods | deployments | services | ingress | hpa | pvc | pv
kubectl apply -f <file.yaml>
kubectl delete -f <file.yaml>
kubectl describe <resource> <name>
kubectl logs <pod-name>
kubectl exec -it <pod-name> -- sh
kubectl scale deployment <name> --replicas=N
kubectl set image deployment/<name> <container>=<image>:<tag>
kubectl rollout status deployment/<name>
kubectl rollout undo deployment/<name>
kubectl top pods
kubectl create configmap <name> --from-literal=key=value
kubectl create secret generic <name> --from-literal=key=value
kubectl port-forward svc/<name> <local>:<remote>
kubectl config use-context <context>
kubectl get all -n <namespace>
minikube start | stop | delete | status | dashboard | tunnel | ssh
helm install | upgrade | rollback | list | uninstall | repo add | search
```

---

**This concludes the Kubernetes extension of the Practical DevOps Training Guide.**
