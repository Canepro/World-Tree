# Kubernetes World Tree Reconstruction Guide (K8S_WORLD_TREE_GUIDE.md) - Updated

**Author:** Vincent Mogah  
**System Version:** Kubernetes v1.35.0 / Kind v0.31.0  
**Context:** Multi-node simulation on Ubuntu with host port mapping.  
**Date:** January 01, 2026  
**Purpose:** This document provides a precise, reproducible blueprint for reconstructing the "World Tree" cluster to its verified stable state, including infrastructure, ingress, observability, persistent storage with PostgreSQL, a sample application for testing, and remote connectivity configurations.

---

## 1. Preparation and Workspace Cleanup
Ensure a clean environment prior to cluster creation to optimize disk usage and avoid conflicts.

```bash
# Delete any existing clusters
kind delete cluster --name kind-cluster

# Prune unused Docker resources
docker system prune -a --volumes -f
```

## 2. Cluster Configuration
Define the multi-node cluster with host port mappings for external access.

File: cluster-config.yaml

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: kind-cluster
nodes:
- role: control-plane
  image: kindest/node:v1.35.0@sha256:452d707d4862f52530247495d180205e029056831160e22870e37e3f6c1ac31f
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
- role: worker
  image: kindest/node:v1.35.0
- role: worker
  image: kindest/node:v1.35.0
```

## 3. Create Cluster
Provision the cluster and label worker nodes for accurate role identification.

```bash
kind create cluster --config cluster-config.yaml --wait 5m

# Label worker nodes
kubectl label node kind-cluster-worker node-role.kubernetes.io/worker=worker
kubectl label node kind-cluster-worker2 node-role.kubernetes.io/worker=worker
```

## 4. Install Ingress Controller (With Control Plane Pinning)
Deploy NGINX Ingress and ensure it schedules on the control plane node for port accessibility.

```bash
# Install NGINX Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# Patch to pin to control plane
kubectl patch deployment ingress-nginx-controller -n ingress-nginx --type='json' -p='[
  {"op": "add", "path": "/spec/template/spec/nodeSelector", "value": {"ingress-ready": "true"}},
  {"op": "add", "path": "/spec/template/spec/tolerations", "value": [
    {"key": "node-role.kubernetes.io/control-plane", "operator": "Equal", "effect": "NoSchedule"},
    {"key": "node-role.kubernetes.io/master", "operator": "Equal", "effect": "NoSchedule"}
  ]}
]'
```

## 5. Install Metrics Server (Kind-Compatible)
Deploy the Metrics Server with necessary patches for Kind environments.

```bash
# Install Metrics Server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.8.0/components.yaml

# Patch for insecure TLS
kubectl patch deployment metrics-server -n kube-system --type 'json' -p '[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}]'

# Verify
kubectl top nodes
```

## 6. Install Local Path Provisioner
Enable dynamic local storage provisioning for development persistence. The default configuration uses `/opt/local-path-provisioner` as the base path.

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
```

## 7. Deploy Stateful PostgreSQL Database
Create persistent storage and deploy a single-replica PostgreSQL StatefulSet.

**Note:** PostgreSQL credentials are stored in a Kubernetes Secret. Create the secret first using `manifests/secrets/postgres-credentials-secret.yaml` before deploying the StatefulSet.

File: manifests/apps/postgres-pvc.yaml

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 1Gi
```

File: manifests/apps/postgres-db.yaml

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-db
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-credentials
              key: POSTGRES_PASSWORD
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-pvc
```

File: manifests/apps/postgres-service.yaml (Headless Service)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  clusterIP: None
  ports:
  - port: 5432
    name: postgres
  selector:
    app: postgres
```

Apply the manifests:

```bash
# Create PostgreSQL credentials secret first
kubectl apply -f manifests/secrets/postgres-credentials-secret.yaml

# Apply PostgreSQL resources
kubectl apply -f manifests/apps/postgres-pvc.yaml
kubectl apply -f manifests/apps/postgres-service.yaml
kubectl apply -f manifests/apps/postgres-db.yaml
```

## 8. Deploy Sample Application for Testing
Deploy a simple web application to validate ingress, scaling, and load handling.

File: manifests/apps/hello-app.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-app
spec:
  replicas: 1  # Initial replicas; adjust for testing
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
      - name: hello-app
        image: nginxdemos/hello:plain-text
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
---
apiVersion: v1
kind: Service
metadata:
  name: hello-service
spec:
  selector:
    app: hello
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-ingress
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hello-service
            port:
              number: 80
```

Apply the manifest:

```bash
kubectl apply -f manifests/apps/hello-app.yaml
```

## 9. Configure Horizontal Pod Autoscaler (Optional for Elasticity Testing)
Enable automatic scaling based on CPU utilization.

```bash
kubectl autoscale deployment hello-app --cpu-percent=50 --min=1 --max=10
```

## 10. Remote Connectivity Setup
Establish remote access to the cluster API for integration with external systems like Argo CD.

### Proxy Configuration
Run a socat proxy to forward traffic from port 6443 to the internal API.

```bash
docker run -d --name kind-proxy --net=host alpine/socat \
  TCP-LISTEN:6443,fork,reuseaddr TCP:127.0.0.1:41079
```

### Firewall Configuration
Allow traffic on port 6443 via UFW.

```bash
sudo ufw allow 6443/tcp
```

### Restoration Script
File: restore-connectivity.sh

```bash
#!/bin/bash
# 1. Clear old proxy
docker rm -f kind-proxy 2>/dev/null

# 2. Start Socat Proxy
echo "Starting Socat Proxy on 6443..."
docker run -d --name kind-proxy --net=host --restart always \
  alpine/socat TCP-LISTEN:6443,fork,reuseaddr TCP:127.0.0.1:41079

# 3. Ensure Firewall is open
echo "Configuring UFW..."
sudo ufw allow 6443/tcp

echo "Done. Test from laptop with: curl -k https://$(hostname):6443/version"
```

Make executable:

```bash
chmod +x ~/restore-connectivity.sh
```

In kubeconfig for remote access, set `insecure-skip-tls-verify: true`.

## 11. Storage Forensics (Locating Persistent Data)
Inspect the underlying storage location for verification or debugging.

```bash
# 1. Identify the hosting node
kubectl get pod postgres-db-0 -o wide

# 2. Retrieve the exact host path from the PV
kubectl describe pv $(kubectl get pvc postgres-pvc -o jsonpath='{.spec.volumeName}') | grep Path

# 3. Access the node container and inspect (replace placeholders; typical base path is /opt/local-path-provisioner)
docker exec -it <NODE_NAME> ls -F <EXACT_PATH_FROM_STEP_2>
```

## 12. Cluster Hibernation and Resumption
To conserve resources while preserving state:

```bash
# Hibernate
docker stop kind-cluster-control-plane kind-cluster-worker kind-cluster-worker2

# Resume
docker start kind-cluster-control-plane kind-cluster-worker kind-cluster-worker2
```

This blueprint reconstructs the cluster to its stable, verified configuration as of January 01, 2026. Additional enhancements, such as advanced observability or GitOps, may be incorporated in future revisions.
