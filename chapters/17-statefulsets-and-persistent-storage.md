# Chapter 17: StatefulSets & Persistent Storage

In this chapter, you will learn how to run stateful applications (like MongoDB) in Kubernetes using StatefulSets and PersistentVolumes.

---

## The Challenge of Stateful Workloads

Stateless applications (like NextJS) are easy in Kubernetes:

```
Deployment → multiple identical Pods → any Pod handles any request
```

Stateful applications (like MongoDB) are harder:

```
Each Pod has unique identity
Data must survive Pod restarts
Pods must start in a specific order
```

Kubernetes provides **StatefulSets** and **PersistentVolumes** to solve these problems.

---

## Part 1: Volumes in Kubernetes

In Docker Compose, you used a named volume (`mongodb_data:`). In Kubernetes, volumes work differently.

### Volume Types

| Volume Type | Description | Persistence |
|---|---|---|
| **emptyDir** | Temporary, tied to Pod lifecycle | Lost when Pod is deleted |
| **hostPath** | Directory on the node's filesystem | Survives Pod restarts, but tied to a specific node |
| **PersistentVolume (PV)** | Cluster-wide storage resource | Survives Pod/node failures |
| **PersistentVolumeClaim (PVC)** | Request for storage by a user | Abstracts the underlying storage |
| **CSI Driver** | Cloud-managed storage (EBS, EFS, GCE PD) | Fully managed, portable |

For production, you use **PersistentVolume (PV)** and **PersistentVolumeClaim (PVC)**.

---

## Part 2: PersistentVolume & PersistentVolumeClaim

### PersistentVolume (PV) — Cluster Storage Resource

A PV is storage provisioned by an administrator (or dynamically by a StorageClass):

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mongodb-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /data/mongodb
```

> On Minikube, the `storage-provisioner` add-on (enabled in Chapter 14) handles PV creation automatically. You rarely create PVs directly.

### PersistentVolumeClaim (PVC) — User's Storage Request

A PVC requests storage from the cluster:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

Kubernetes matches PVCs to PVs automatically.

### Access Modes

| Mode | Meaning |
|---|---|
| **ReadWriteOnce (RWO)** | Single node can read/write |
| **ReadOnlyMany (ROX)** | Multiple nodes can read |
| **ReadWriteMany (RWX)** | Multiple nodes can read/write |

For most databases, use **ReadWriteOnce**.

### Use a PVC in a Pod

```yaml
spec:
  containers:
    - name: mongodb
      image: mongo:7
      volumeMounts:
        - name: mongodb-storage
          mountPath: /data/db
  volumes:
    - name: mongodb-storage
      persistentVolumeClaim:
        claimName: mongodb-pvc
```

---

## Part 3: StatefulSet — Stateful Workloads

A StatefulSet is like a Deployment but for stateful applications. Key differences:

| Feature | Deployment | StatefulSet |
|---|---|---|
| Pod identity | Random hash (e.g., `nextjs-7d4f8d`) | Ordinal index (e.g., `mongodb-0`, `mongodb-1`) |
| Pod naming | `app-<hash>` | `<statefulset-name>-<ordinal>` |
| Pod ordering | Any order | Ordered creation/termination (0, 1, 2...) |
| Stable network ID | No (IP changes) | Yes (DNS `pod-name-0.service-name`) |
| Storage | Shared volume | Dedicated PVC per Pod |

### MongoDB StatefulSet

Create `statefulset-mongodb.yaml`:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
spec:
  serviceName: mongodb-service
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
        - name: mongodb
          image: mongo:7
          env:
            - name: MONGO_INITDB_ROOT_USERNAME
              value: admin
            - name: MONGO_INITDB_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secret
                  key: MONGODB_PASSWORD
          ports:
            - containerPort: 27017
          volumeMounts:
            - name: mongodb-data
              mountPath: /data/db
  volumeClaimTemplates:
    - metadata:
        name: mongodb-data
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 10Gi
```

Key differences from a Deployment:

| Field | Purpose |
|---|---|
| `serviceName` | Required — gives stable DNS names to Pods |
| `volumeClaimTemplates` | Creates a PVC automatically for each replica |

### Headless Service for StatefulSet

StatefulSets need a **headless Service** (no ClusterIP) to provide stable DNS names:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb-service
spec:
  clusterIP: None  # Headless
  selector:
    app: mongodb
  ports:
    - port: 27017
      targetPort: 27017
```

With a headless service, Pods are reachable as:

```
mongodb-0.mongodb-service   → Pod 0
mongodb-1.mongodb-service   → Pod 1
```

---

## Part 4: Full Stack on K8s

Apply all the resources:

```bash
kubectl apply -f configmap-and-secret.yaml
kubectl apply -f statefulset-mongodb.yaml
kubectl apply -f deployment-and-service.yaml
```

Check everything:

```bash
kubectl get pods
kubectl get pvc
kubectl get pv
```

Expected:

```
NAME       READY   STATUS    RESTARTS   AGE
mongodb-0   1/1    Running   0          1m
nextjs-...  1/1    Running   0          1m
nextjs-...  1/1    Running   0          1m

NAME                   STATUS   VOLUME                                     CAPACITY
mongodb-data-mongodb-0 Bound    pvc-xxxxx                                 10Gi
```

---

## Part 5: Verify Data Persistence

Create test data:

```bash
# Connect to MongoDB
kubectl exec -it mongodb-0 -- mongosh \
  --username admin \
  --password MyStr0ng!Pass

# Inside MongoDB shell:
use testdb
db.testcollection.insertOne({ name: "persistence-test", date: new Date() })
db.testcollection.find()
exit
```

Delete the Pod:

```bash
kubectl delete pod mongodb-0
```

The StatefulSet recreates it automatically. Wait for the new Pod:

```bash
kubectl wait --for=condition=Ready pod/mongodb-0
```

Verify data survived:

```bash
kubectl exec -it mongodb-0 -- mongosh \
  --username admin \
  --password MyStr0ng!Pass \
  --eval "use testdb; db.testcollection.find()"
```

The data should still be there — the PVC persists independently of the Pod.

---

## Key Takeaways

- **Deployments** for stateless apps (NextJS, Nginx, API servers)
- **StatefulSets** for stateful apps (MongoDB, PostgreSQL, Redis)
- **PVCs** provide persistent storage that survives Pod restarts
- **volumeClaimTemplates** in StatefulSets create one PVC per replica automatically
- **Headless Services** give stable DNS names to StatefulSet Pods

---

## Verification Checklist

Before moving on, confirm:

- [x] MongoDB StatefulSet is running with `mongodb-0` Pod
- [x] A PVC is automatically created for the StatefulSet
- [x] Data written to MongoDB survives Pod deletion and recreation
- [x] The NextJS Deployment and Service are running alongside MongoDB

---

**Next:** [Chapter 18 — Kubernetes Ingress & Networking](18-kubernetes-ingress-and-networking.md)
