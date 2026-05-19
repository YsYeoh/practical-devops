# Chapter 16: ConfigMaps, Secrets & Configuration

In this chapter, you will learn how to manage application configuration and sensitive data separately from container images using ConfigMaps and Secrets.

---

## Part 1: ConfigMaps — Non-Sensitive Configuration

ConfigMaps store configuration data that is **not** secret: environment names, API URLs, feature flags, configuration files.

### Create a ConfigMap from Literal Values

```bash
kubectl create configmap app-config \
  --from-literal=NODE_ENV=production \
  --from-literal=PORT=3000 \
  --from-literal=API_URL=http://backend-service
```

### Create a ConfigMap from a File

```bash
# Create a config file
cat <<EOF > app.properties
NODE_ENV=production
PORT=3000
API_URL=http://backend-service
EOF

# Create ConfigMap from the file
kubectl create configmap app-config --from-file=app.properties
```

### Create a ConfigMap from YAML

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  NODE_ENV: "production"
  PORT: "3000"
  API_URL: "http://backend-service"
  nginx.conf: |
    server {
      listen 80;
      location / {
        proxy_pass http://nextjs:3000;
      }
    }
```

Apply it:

```bash
kubectl apply -f configmap.yaml
```

### Use a ConfigMap in a Pod

**As environment variables:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-demo
spec:
  containers:
    - name: app
      image: nginx:latest
      envFrom:
        - configMapRef:
            name: app-config
```

**As a mounted file:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-demo
spec:
  containers:
    - name: app
      image: nginx:latest
      volumeMounts:
        - name: config
          mountPath: /etc/config
  volumes:
    - name: config
      configMap:
        name: app-config
```

Each key in the ConfigMap becomes a file in `/etc/config/`:

```bash
kubectl exec configmap-demo -- ls /etc/config/
# NODE_ENV  PORT  API_URL  nginx.conf
```

---

## Part 2: Secrets — Sensitive Data

Secrets work like ConfigMaps but are intended for sensitive data (passwords, API keys, tokens). Values are stored as Base64-encoded strings.

### Create a Secret from Literal Values

```bash
kubectl create secret generic app-secret \
  --from-literal=MONGODB_PASSWORD='MyStr0ng!Pass' \
  --from-literal=JWT_SECRET='your-jwt-secret'
```

### Create a Secret from a File

```bash
echo -n 'MyStr0ng!Pass' > ./mongodb_password.txt
kubectl create secret generic app-secret --from-file=./mongodb_password.txt
```

### Create a Secret from YAML

Values must be Base64-encoded:

```bash
echo -n 'MyStr0ng!Pass' | base64
# TXlTdHIwbmchUGFzcw==
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  MONGODB_PASSWORD: TXlTdHIwbmchUGFzcw==
  JWT_SECRET: eW91ci1qd3Qtc2VjcmV0
```

### Use a Secret in a Pod

**As environment variables:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-demo
spec:
  containers:
    - name: app
      image: nginx:latest
      env:
        - name: MONGODB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: MONGODB_PASSWORD
```

**As a mounted file:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-demo
spec:
  containers:
    - name: app
      image: nginx:latest
      volumeMounts:
        - name: secrets
          mountPath: /run/secrets
          readOnly: true
  volumes:
    - name: secrets
      secret:
        secretName: app-secret
```

Secrets mounted as files are automatically Base64-decoded. Your application reads them:

```javascript
// Node.js
const fs = require('fs');
const password = fs.readFileSync('/run/secrets/MONGODB_PASSWORD', 'utf8').trim();
```

---

## Part 3: Complete Example — Configuring Our NextJS App

Create `configmap-and-secret.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nextjs-config
data:
  NODE_ENV: "production"
  PORT: "3000"
  MONGODB_URI: "mongodb://admin:$(MONGODB_PASSWORD)@mongodb-service:27017"
---
apiVersion: v1
kind: Secret
metadata:
  name: nextjs-secret
type: Opaque
data:
  MONGODB_PASSWORD: TXlTdHIwbmchUGFzcw==
---
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
          command: ["sleep", "3600"]
          ports:
            - containerPort: 3000
          envFrom:
            - configMapRef:
                name: nextjs-config
          env:
            - name: MONGODB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: nextjs-secret
                  key: MONGODB_PASSWORD
```

---

## Part 4: Best Practices

| Do | Don't |
|---|---|
| Store all config in ConfigMaps | Hardcode values in the Docker image |
| Use Secrets for passwords, API keys, tokens | Store secrets in ConfigMaps |
| Use environment-specific ConfigMaps/Secrets (dev, staging, prod) | Use the same config for all environments |
| Keep ConfigMaps and Secrets in version control (without real secrets) | Commit real secret values to Git |
| Use external secret stores for production (Vault, AWS Secrets Manager) | Rely only on Kubernetes Secrets for sensitive production data |

### Managing Secrets Safely

For real secrets in production:

1. **Do not commit raw Secret YAML** to Git
2. Use tools like:
   - **Sealed Secrets** — encrypt Secrets into Git-safe CRDs
   - **External Secrets Operator** — sync Secrets from AWS/GCP/Azure
   - **Vault** with CSI driver — mount secrets from HashiCorp Vault

---

## Verification Checklist

Before moving on, confirm:

- [x] You can create a ConfigMap and use it as env vars in a Pod
- [x] You can create a Secret and inject it as env vars or files
- [x] You understand that Secret values are Base64-encoded (not encrypted)
- [x] You can verify the Secret is accessible inside a running container
- [x] You know how to use `envFrom` with ConfigMapRef and `valueFrom.secretKeyRef` with secrets

---

### ConfigMap vs Secret Quick Reference

| | ConfigMap | Secret |
|---|---|---|
| Content type | Plain text | Base64-encoded |
| Size limit | 1 MiB | 1 MiB |
| Encryption at rest | No (unless etcd encrypted) | Yes (if encryption configured) |
| Use for | App config, URLs, feature flags | Passwords, API keys, tokens |

---

**Next:** [Chapter 17 — StatefulSets & Persistent Storage](17-statefulsets-and-persistent-storage.md)
