# ArgoCD GitOps Engine Guide: Production Cluster

This document outlines the architecture, high-availability profile, and credential management for ArgoCD on the production cluster.

---

## 1. High Availability (HA) Architecture

ArgoCD is installed in HA mode across the 6 worker nodes with strict redundancy:

- **Redis HA**: 3-node Redis cluster orchestrated with Sentinel (`redis-ha`), with pod anti-affinity across worker nodes.
- **ArgoCD Server**: 2+ replicas managed by Horizontal Pod Autoscaler (HPA max: 5).
- **ArgoCD Repo Server**: 2+ replicas managed by Horizontal Pod Autoscaler (HPA max: 5).
- **ArgoCD ApplicationSet Controller**: 2 replicas with active leader election.
- **ArgoCD Controller**: Scaled and dedicated to reconciling cluster state.
- **Manifest Location**: [`bootstrap/argocd/values-prod.yaml`](file:///home/mado/prod-cluster/bootstrap/argocd/values-prod.yaml)

---

## 2. Resource Allocations

| Component | CPU Request | Memory Request | CPU Limit | Memory Limit |
| :--- | :---: | :---: | :---: | :---: |
| **Application Controller** | `500m` | `1024Mi` | `2000m` | `2048Mi` |
| **API Server** | `100m` | `128Mi` | `500m` | `512Mi` |
| **Repo Server** | `250m` | `512Mi` | `1000m` | `1024Mi` |
| **Redis HA** | `100m` | `128Mi` | `300m` | `256Mi` |
| **ApplicationSet Controller**| `100m` | `128Mi` | `500m` | `512Mi` |
| **Notifications Controller** | `50m` | `64Mi` | `200m` | `256Mi` |

---

## 3. Initial Admin Credentials & Access

```bash
# Retrieve initial admin password
kubectl --kubeconfig ~/.kube/config-prod -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo ""

# Port-forward to ArgoCD UI
kubectl --kubeconfig ~/.kube/config-prod port-forward svc/argocd-server -n argocd 8080:443
```
Login with username `admin` and the retrieved password at `https://localhost:8080`.

---

## 4. App-of-Apps GitOps Workflow

The production cluster utilizes the **App-of-Apps** pattern:
- The root application is defined in [`gitops/root-app.yaml`](file:///home/mado/prod-cluster/gitops/root-app.yaml).
- It automatically detects and syncs all applications defined under [`gitops/apps/`](file:///home/mado/prod-cluster/gitops/apps/) (such as `longhorn.yaml`, `metallb.yaml`, etc.).
