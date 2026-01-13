# World Tree Cluster Configuration

Kubernetes Kind cluster configuration for the "World Tree" spoke cluster in a Hub-and-Spoke observability architecture.

## Overview

This repository contains the complete configuration for a local Kind cluster that:
- Runs a multi-node Kubernetes cluster (v1.35.0)
- Integrates with a central observability Hub via Prometheus Remote Write
- Provides persistent PostgreSQL storage with local-path provisioning
- Includes sample applications for testing

## Quick Start

1. **Create the cluster:**
   ```bash
   kind create cluster --config cluster-config.yaml --wait 5m
   ```

2. **Install prerequisites:**
   - Ingress Controller (NGINX)
   - Metrics Server
   - Local Path Provisioner

3. **Deploy applications:**
   ```bash
   # PostgreSQL (create secret first)
   kubectl apply -f manifests/secrets/postgres-credentials-secret.yaml
   kubectl apply -f manifests/apps/postgres-pvc.yaml
   kubectl apply -f manifests/apps/postgres-service.yaml
   kubectl apply -f manifests/apps/postgres-db.yaml
   
   # Sample application
   kubectl apply -f manifests/apps/hello-app.yaml
   ```

4. **Set up observability (optional):**
   ```bash
   # Create monitoring namespace
   kubectl create namespace monitoring
   
   # Create observability credentials secret (required for Prometheus Agent)
   kubectl create secret generic observability-credentials -n monitoring \
     --from-literal=username="observability-user" \
     --from-literal=password="YOUR_PASSWORD_HERE"
   
   # Deploy Prometheus Agent
   kubectl apply -f manifests/monitoring/prometheus-agent-world-tree-spoke.yaml
   ```

For detailed instructions, see [docs/K8S_WORLD_TREE_GUIDE.md](docs/K8S_WORLD_TREE_GUIDE.md).

## Repository Structure

```
.
├── README.md                          # This file
├── cluster-config.yaml                # Kind cluster definition
│
├── manifests/                         # Kubernetes manifests
│   ├── apps/                          # Application deployments
│   │   ├── hello-app.yaml            # Sample web application
│   │   ├── postgres-db.yaml          # PostgreSQL StatefulSet
│   │   ├── postgres-pvc.yaml         # PostgreSQL PersistentVolumeClaim
│   │   └── postgres-service.yaml     # PostgreSQL headless Service
│   │
│   ├── infrastructure/                # Infrastructure components
│   │   └── ingress-nginx-patched.yaml # NGINX Ingress controller template
│   │
│   ├── monitoring/                    # Observability components
│   │   └── prometheus-agent-world-tree-spoke.yaml # Prometheus Agent
│   │
│   └── secrets/                       # Kubernetes Secrets
│       ├── postgres-credentials-secret.yaml
│       └── promtail-credentials-secret.yaml
│
├── helm/                              # Helm values files
│   ├── metrics-values.yaml            # Metrics Server values
│   └── promtail-kind-values.yaml      # Promtail Helm values
│
└── docs/                              # Documentation
    ├── K8S_WORLD_TREE_GUIDE.md        # Complete cluster reconstruction guide
    ├── TRAINING_LOG_MASTER.md         # Historical log of cluster evolution
    ├── NETWORKING-LAB.md              # Remote connectivity and firewall traversal
    └── WORLD-TREE-SPOKE-SETUP.md      # Prometheus Remote Write setup guide
```

## Prerequisites

- Docker (for Kind)
- kubectl
- Kind v0.31.0+
- Helm (optional, for Promtail)

## Key Features

- **Multi-node Kind cluster** (1 control-plane + 2 workers)
- **NGINX Ingress** with control plane pinning
- **PostgreSQL StatefulSet** with persistent storage
- **Prometheus Agent** for remote metrics push to Hub
- **Local Path Provisioner** for persistent volumes
- **Metrics Server** for HPA support

## Security Notes

- Credentials are stored in Kubernetes Secrets (see `manifests/secrets/*.yaml` files)
- **Important:** Create secrets before deploying applications that reference them
- **Note:** Secret manifests contain example credentials - replace with your own values for production use

## Cluster Information

- **Name:** kind-cluster
- **Kubernetes Version:** v1.35.0
- **Cluster Label:** `world-tree` (used for observability)
- **Namespace:** Default + monitoring

## Documentation

All detailed documentation is in the `docs/` folder:
- [Reconstruction Guide](docs/K8S_WORLD_TREE_GUIDE.md) - Step-by-step cluster setup
- [Training Log](docs/TRAINING_LOG_MASTER.md) - Historical development notes
- [Network Setup](docs/NETWORKING-LAB.md) - Remote connectivity configuration
- [Prometheus Setup](docs/WORLD-TREE-SPOKE-SETUP.md) - Observability integration

## License

Configuration files for local development and testing.
