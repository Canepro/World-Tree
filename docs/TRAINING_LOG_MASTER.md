# Kubernetes "World Tree" Training Log (TRAINING_LOG_MASTER.md) - Updated

**Cluster:** kind-cluster  
**Version:** v1.35.0  
**Date:** January 01, 2026  
**Author:** Vincent Mogah

---

## Phase 1: Infrastructure and the v1.35 Leap
The migration from a legacy v1.27 environment to v1.35 was executed successfully. This process necessitated a comprehensive disk prune and an upgrade of the Kind binary to v0.31.0.  
- **Key Lesson:** Kubernetes v1.35 imposes stricter cgroup v2 standards. Employing the latest Kind node images (SHA256: 452d707d4862f52530247495d180205e029056831160e22870e37e3f6c1ac31f) proved essential for ensuring operational stability.

## Phase 2: The Ingress "Connection Refused" Fix
**Problem:** Traffic from localhost failed to reach the Ingress controller, as it was scheduled on a worker node lacking the necessary port mappings.  
**Solution:** Implemented "control plane pinning" by patching the NGINX deployment with a nodeSelector to constrain it to the node equipped with host port mappings for ports 80 and 443.

## Phase 3: The Bitnami OCI and Registry War
**Problem:** Helm encountered failures in pulling images attributable to Bitnami's enhanced secure registry restrictions and "unrecognized image" errors.  
**Solution:** Transitioned to official images from registry.k8s.io and circumvented Bitnami chart security constraints by setting `global.security.allowInsecureImages=true`. This adjustment facilitated the Metrics Server's operation under v1.35.

## Phase 4: Elasticity and Load Testing
**Achievement:** A scaling event was induced successfully, escalating the cluster from 1 to 9 replicas.  
- **Requests per Second (RPS):** Approximately 1,936.  
- **Stability:** No failed requests occurred during a 200% CPU surge.  
- **Observation:** The Horizontal Pod Autoscaler requires approximately 60 seconds to respond to data from the Metrics API.

## Phase 5: Persistence and Node Affinity
**Achievement:** Demonstrated data durability across pod terminations utilizing a PostgreSQL StatefulSet backed by Rancher Local Path Provisioner.  
- **Discovery:** Data remains bound to a designated node (e.g., Worker 2). Attempts to access data from an alternative node yielded "No such file or directory" errors, illustrating the principles of local storage topology and the importance of node affinity in development environments.

## Phase 6: Final Verification and Hibernation
**Achievement:** Comprehensive cluster health verification confirmed all components operational prior to resource conservation. The `kubectl get all -A` output validated full readiness across namespaces. The cluster was then hibernated while preserving all data and configurations.  
- **Hibernation Commands:**  
  ```bash
  docker stop kind-cluster-control-plane kind-cluster-worker kind-cluster-worker2
  ```  
- **Resumption Commands:**  
  ```bash  
  docker start kind-cluster-control-plane kind-cluster-worker kind-cluster-worker2
  ```  
- **Status:** The "World Tree" is stable and ready for resumption at any time.

## Phase 7: Remote Connectivity and Firewall Traversal ("Saga of Port 6443")
**Achievement:** Established secure remote access to the Kind cluster API from a central Argo CD Hub on OKE, overcoming lab environment restrictions.  
- **Problem Summary:** API isolation to localhost, port restrictions (only 22, 80, 443, 6443 allowed), and active UFW firewall.  
- **Solution:** Deployed a socat-based Layer 4 proxy to forward traffic from port 6443 to the internal API port 41079, updated UFW rules, and applied TLS verification skip in kubeconfig.  
- **Key Lesson:** In restricted environments, proxy bridges and firewall alignments enable multi-cluster integrations like GitOps. Observability confirmed via metrics, logs, and node statuses.  
- **Restoration Script:** Created `~/restore-connectivity.sh` for reproducible recovery post-lab resets.

## Phase 8: Future Evolutionary Paths
The current cluster provides a solid foundation for local development and testing. The following enhancements are planned to evolve it toward production-grade capabilities, emphasizing automation, observability, and proactive operations.

- **GitOps Integration:** Transition to ArgoCD for declarative, automated application deployment and state reconciliation. This will enable git-based configuration management, drift detection, and continuous synchronization.
- **The LGTM Stack:** Deploy Loki for distributed logging, Grafana for unified visualization, Tempo for distributed tracing, and (optionally) Mimir for long-term metrics storage. This will provide full-spectrum observability beyond the current Metrics Server.
- **Alerting:** Implement Prometheus (replacing or supplementing Metrics Server) with Alertmanager to enable rule-based alerting, notification routing, and integration with external systems, closing the loop from metrics collection to actionable response.

These enhancements will be documented in subsequent phases of both the Training Log (narrative progress) and Reconstruction Guide (precise commands) as they are implemented.

This document remains a living record of the cluster's evolution. Future updates will detail the execution and outcomes of the planned phases.
