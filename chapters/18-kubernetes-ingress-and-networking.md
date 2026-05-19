# Chapter 18: Kubernetes Ingress & Networking

In this chapter, you will learn how to expose your application to the outside world using Ingress — replacing the Nginx reverse proxy you used with Docker Compose.

---

## The Problem: How to Reach Your App

In Docker Compose, you exposed your app via `nginx` on port 80.

In Kubernetes, you have several options:

| Method | What It Does | When to Use |
|---|---|---|
| **kubectl port-forward** | Forwards a local port to a Pod | Debugging only |
| **NodePort Service** | Exposes on `<NodeIP>:<high-port>` | Development |
| **LoadBalancer Service** | Creates a cloud load balancer | Cloud production |
| **Ingress** | HTTP/HTTPS routing with host/path rules | Production (self-hosted or cloud) |

**Ingress** is the Kubernetes-native way to handle HTTP/HTTPS traffic — it replaces the Nginx reverse proxy from Docker Compose.

---

## Part 1: What Is Ingress?

Ingress is a Kubernetes object that manages external HTTP/HTTPS access to Services. It provides:

- Host-based routing (`app.example.com` → Service A, `api.example.com` → Service B)
- Path-based routing (`/api/*` → backend, `/*` → frontend)
- TLS/SSL termination
- Load balancing

### Ingress vs Nginx in Docker Compose

| Docker Compose | Kubernetes |
|---|---|
| Manual Nginx config in `nginx/default.conf` | Declarative Ingress YAML |
| `proxy_pass http://nextjs:3000` | Service-based routing |
| Manual reload (`docker compose restart nginx`) | Automatic reconciliation |
| Single Nginx instance | Ingress Controller manages Nginx |

---

## Part 2: Architecture

Ingress requires two parts:

```
Ingress Controller (running in cluster)
         +  Ingress Rules (YAML configuration)
         =  Working Ingress
```

| Component | Role | Example |
|---|---|---|
| **Ingress Controller** | A Pod that runs the actual reverse proxy software | nginx-ingress, traefik, haproxy |
| **Ingress Resource** | YAML rules that tell the controller how to route | `ingress.yaml` |

> **Note:** Installing an Ingress resource without an Ingress Controller does nothing. The Controller processes the rules.

---

## Part 3: Enable the Ingress Controller

If you enabled the `ingress` add-on in Chapter 14:

```bash
minikube addons enable ingress
```

Wait for the controller Pod to be ready:

```bash
kubectl get pods -n ingress-nginx
```

Expected:

```
NAME                                        READY   STATUS
ingress-nginx-controller-7c4b8d4b8f-abc12   1/1     Running
```

> On Minikube, the ingress add-on automatically installs the NGINX Ingress Controller.

---

## Part 4: Create an Ingress Resource

Now create rules to route traffic to your services.

Create `ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: app.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nextjs-service
                port:
                  number: 80
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend-service
                port:
                  number: 80
```

Apply it:

```bash
kubectl apply -f ingress.yaml
```

Verify:

```bash
kubectl get ingress
```

Expected output:

```
NAME          CLASS   HOSTS       ADDRESS        PORTS   AGE
app-ingress   nginx   app.local   192.168.49.2   80      10s
```

### Understanding the Ingress Rules

| Field | Purpose |
|---|---|
| `host: app.local` | Route only requests with this Host header |
| `path: /` | Root path → `nextjs-service` |
| `path: /api` | API path → `backend-service` |
| `pathType: Prefix` | Match the path prefix (not exact) |

### Test the Ingress

On Minikube, get the Ingress IP:

```bash
# This creates a tunnel so the Ingress is accessible locally
minikube tunnel
```

In another terminal, add the domain to your hosts file:

```bash
# Linux/macOS
echo "127.0.0.1 app.local" | sudo tee -a /etc/hosts

# Windows (PowerShell as Admin)
Add-Content -Path C:\Windows\System32\drivers\etc\hosts -Value "`n127.0.0.1 app.local"
```

Test:

```bash
curl -H "Host: app.local" http://localhost
```

---

## Part 5: Path-Based Routing

You can split traffic between services by HTTP path:

```yaml
spec:
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend-service
                port:
                  number: 80
          - path: /monitoring
            pathType: Prefix
            backend:
              service:
                name: grafana-service
                port:
                  number: 3000
```

| Path Type | Behavior |
|---|---|
| `Prefix` | Matches the path prefix (`/api` matches `/api/users`, `/api/health`) |
| `Exact` | Matches exactly (`/api` matches only `/api`) |
| `ImplementationSpecific` | Varies by Ingress Controller |

---

## Part 6: Host-Based Routing

For multiple domains pointing to different services:

```yaml
spec:
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: backend-service
                port:
                  number: 80
```

---

## Part 7: TLS/SSL with Ingress

To add HTTPS, reference a Secret containing your TLS certificate:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress-tls
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - app.example.com
      secretName: app-tls-secret
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nextjs-service
                port:
                  number: 80
```

Create the TLS Secret:

```bash
kubectl create secret tls app-tls-secret \
  --cert=fullchain.pem \
  --key=privkey.pem
```

> For production, use **cert-manager** to automatically issue and renew Let's Encrypt certificates — similar to Certbot on your VM.

---

## Part 8: Complete Example — Our Full Stack with Ingress

Create `full-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: fullstack-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: myapp.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nextjs-service
                port:
                  number: 80
```

---

## Verification Checklist

Before moving on, confirm:

- [x] The Ingress Controller is running (`kubectl get pods -n ingress-nginx`)
- [x] You created an Ingress resource pointing to `nextjs-service`
- [x] The Ingress has an IP address (`kubectl get ingress`)
- [x] You can access the application through the Ingress
- [x] You understand how Ingress replaces the Nginx reverse proxy from Docker Compose

---

## Quick Reference: Exposing Apps in K8s

| Method | Command / YAML | Best For |
|---|---|---|
| `kubectl port-forward` | `kubectl port-forward svc/nextjs-service 8080:80` | Quick debugging |
| NodePort | `type: NodePort` in Service YAML | Dev/test |
| LoadBalancer | `type: LoadBalancer` in Service YAML | Cloud prod |
| Ingress | `kind: Ingress` + Ingress Controller | Production (self-hosted or cloud) |

---

**Next:** [Chapter 19 — Helm Package Manager](19-helm-package-manager.md)
