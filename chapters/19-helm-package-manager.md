# Chapter 19: Helm — The Kubernetes Package Manager

In this chapter, you will learn Helm, which packages Kubernetes YAML files into reusable charts — similar to how `apt` packages software or how `npm` packages libraries.

---

## The Problem Helm Solves

A real application stack needs multiple Kubernetes resources:

```
Deployment + Service + ConfigMap + Secret + Ingress + PVC + ...
```

Managing each YAML file individually becomes tedious. Helm packages all of them into a single **chart** that can be:

- Installed with one command
- Configured via values (no YAML editing)
- Versioned and shared
- Rolled back easily

### Docker Compose vs Helm

| Docker Compose | Helm |
|---|---|
| One `docker-compose.yml` for all services | One chart (or subchart) per service |
| Environment variables via `.env` | Configurable via `values.yaml` |
| `docker compose up -d` | `helm install` / `helm upgrade` |
| No versioning | Chart versions + release tracking |
| Single host | Multi-node cluster |

---

## Part 1: Install Helm

```bash
# Linux/macOS
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# macOS (alternative)
brew install helm

# Windows
choco install kubernetes-helm
```

Verify:

```bash
helm version
```

Expected:

```
version.BuildInfo{Version:"v3.16.x"}
```

---

## Part 2: Understanding Helm Charts

A Helm chart is a directory with this structure:

```
my-chart/
├── Chart.yaml          # Chart metadata (name, version, description)
├── values.yaml         # Default configuration values
├── charts/             # Subchart dependencies (optional)
└── templates/          # Go-template YAML files
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    ├── configmap.yaml
    ├── _helpers.tpl    # Template helper functions
    └── NOTES.txt       # Post-install instructions
```

### Chart.yaml

```yaml
apiVersion: v2
name: nextjs-app
description: A NextJS application with MongoDB
version: 0.1.0
appVersion: "1.0.0"
type: application
```

### values.yaml

```yaml
replicaCount: 2

image:
  repository: nginx
  tag: latest
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

resources:
  requests:
    cpu: 250m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi

mongodb:
  enabled: true
  image: mongo:7
  storage: 10Gi
```

### Template Example (templates/deployment.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "nextjs-app.fullname" . }}
  labels:
    app.kubernetes.io/name: {{ include "nextjs-app.name" . }}
    helm.sh/chart: {{ include "nextjs-app.chart" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app.kubernetes.io/name: {{ include "nextjs-app.name" . }}
  template:
    metadata:
      labels:
        app.kubernetes.io/name: {{ include "nextjs-app.name" . }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.port }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

Templates use Go templates (`{{ .Values.fieldName }}`) to inject values from `values.yaml`.

---

## Part 3: Create Your First Chart

```bash
# Scaffold a chart
helm create my-app

# View the chart structure
tree my-app/
```

This creates a complete, working Nginx chart with Deployment, Service, HPA, Ingress, and more.

### Install the Chart

```bash
helm install my-release ./my-app
```

### Override Default Values

```bash
# With --set
helm install my-release ./my-app --set replicaCount=3 --set image.tag=v2

# Or with a custom values file
echo 'replicaCount: 3' > custom-values.yaml
helm install my-release ./my-app -f custom-values.yaml
```

### List Installed Releases

```bash
helm list
```

Expected:

```
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART
my-release      default         1               2024-01-01 12:00:00                     deployed        my-app-0.1.0
```

### Upgrade a Release

```bash
helm upgrade my-release ./my-app --set replicaCount=5
```

### Rollback a Release

```bash
# Check revision history
helm history my-release

# Rollback to revision 1
helm rollback my-release 1
```

### Uninstall

```bash
helm uninstall my-release
```

---

## Part 4: Using Helm Repositories

Helm can install charts from remote repositories — useful for installing complex stacks (Prometheus, MongoDB, Nginx Ingress, etc.).

### Add a Repository

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

### Install MongoDB from Bitnami

```bash
# Install with default values
helm install mongodb bitnami/mongodb

# Install with custom values
helm install mongodb bitnami/mongodb \
  --set auth.rootPassword=MyStr0ng!Pass \
  --set persistence.size=20Gi
```

### Install Nginx Ingress Controller

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx
```

### Install Prometheus Stack

```bash
helm install prometheus prometheus-community/kube-prometheus-stack
```

This single command installs:
- Prometheus (metrics collection)
- Grafana (dashboards)
- AlertManager (alerts)
- Node Exporter (server metrics)
- kube-state-metrics (cluster state metrics)

---

## Part 5: Create a Chart for Our Stack

Create a chart structure for your full stack:

```bash
helm create fullstack-app
```

Modify `Chart.yaml`:

```yaml
apiVersion: v2
name: fullstack-app
description: NextJS + MongoDB + Ingress stack
version: 0.1.0
dependencies:
  - name: mongodb
    version: 15.x.x
    repository: https://charts.bitnami.com/bitnami
    condition: mongodb.enabled
```

Update dependencies:

```bash
helm dependency update
```

Modify `values.yaml` to expose MongoDB settings and your app config, then customize the templates for your NextJS Deployment, Service, and Ingress.

This is the real power of Helm — your entire stack becomes **one command**:

```bash
helm install production ./fullstack-app -f production-values.yaml
```

---

## Helm Command Reference

| Command | Purpose |
|---|---|
| `helm create <name>` | Scaffold a new chart |
| `helm install <release> <chart>` | Install a chart |
| `helm upgrade <release> <chart>` | Upgrade a release |
| `helm rollback <release> <revision>` | Rollback to a previous revision |
| `helm list` | List installed releases |
| `helm uninstall <release>` | Delete a release |
| `helm repo add <name> <url>` | Add a Helm repository |
| `helm repo update` | Update local repo cache |
| `helm search repo <keyword>` | Search for charts in repos |
| `helm show values <chart>` | View default values of a chart |
| `helm dependency update` | Update chart dependencies |
| `helm history <release>` | View revision history |
| `helm get values <release>` | View the values used by a release |
| `helm template <release> <chart>` | Render templates locally (dry run) |

---

## Verification Checklist

Before moving on, confirm:

- [x] `helm version` shows Helm v3
- [x] You scaffolded and installed a chart with `helm create` + `helm install`
- [x] You can override values with `--set` and `-f custom-values.yaml`
- [x] You can upgrade and rollback a release
- [x] You can install a third-party chart from a repo (e.g., Bitnami MongoDB)
- [x] You understand how `values.yaml` feeds into Go templates

---

**Next:** [Chapter 20 — Production Practices, CI/CD & Next Steps](20-production-practices-cicd-and-next-steps.md)
