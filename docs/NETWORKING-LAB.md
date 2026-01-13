# Kind Cluster: Remote Connectivity & Firewall Traversal (NETWORKING-LAB.md)

## Overview
This document outlines the strategy used to connect a local Kind cluster (running in a restricted lab environment) to a central Argo CD Hub running on OKE.

## The Problem
- **Isolation:** Kind API was bound to `127.0.0.1:41079`.
- **Restricted Egress/Ingress:** Lab provider blocked most ports except 22, 80, 443, and 6443.
- **Internal Security:** OS-level `ufw` was active and dropping packets by default.

## The Solution: Layer 4 Proxy Bridge
We implemented a `socat` proxy to bridge the gap between the public internet and the internal Kind network.

### 1. Proxy Configuration
A Podman/Docker container runs on the host to map the allowlisted port `6443` to Kind's internal API port.

```bash
docker run -d --name kind-proxy --net=host alpine/socat \
  TCP-LISTEN:6443,fork,reuseaddr TCP:127.0.0.1:41079
```

### 2. Firewall Alignment
The internal Ubuntu firewall (ufw) was updated to permit traffic on the gateway port:

```bash
sudo ufw allow 6443/tcp
```

### 3. TLS Hostname Workaround
Since the Kind certificate is issued for localhost and not the lab FQDN, we use `insecure-skip-tls-verify: true` in the kubeconfig to allow the handshake to complete.

## Observability Status
- **Metrics:** `kubectl top nodes` verified (Metrics-Server path is clear).
- **Logs:** Ingress logs verified via `kubectl logs` (Log streaming path is clear).
- **Control Plane:** Node status is Ready across all 3 nodes.

---

# README.md (Updated)

## 🌐 Multi-Cluster Infrastructure
Our environment consists of a Hub-and-Spoke architecture:

1. **Hub Cluster (OKE):** Hosts the Argo CD Instance and Grafana/Loki stack.
2. **Edge Cluster (Kind):** Remote cluster in Pluralsight Labs used for feature testing.

### Connectivity Restoration
If the Lab environment resets, run the following on the Lab Server:  
`~/restore-connectivity.sh`

### Verified Endpoints
- **Kind API:** `https://b0f08dc8213c.mylabserver.com:6443`
- **Kind Apps:** `http://b0f08dc8213c.mylabserver.com:80` (via Ingress-NGINX)

---
