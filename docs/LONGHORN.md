# Longhorn Distributed Block Storage Guide: Production Cluster

This document details the configuration, high-availability replication, and operational management of Longhorn on the production cluster.

---

## 1. Storage Architecture & HA Policy

Longhorn provides enterprise-grade distributed block storage across the 6 worker nodes.

- **StorageClass Name**: `longhorn` (Set as Cluster Default)
- **Replication Factor**: 3-Way Active Replication (`defaultReplicaCount: 3`)
- **Replica Anti-Affinity**: Hard anti-affinity (`replicaSoftAntiAffinity: false`) ensures that each replica of every volume is scheduled on a physically separate worker node.
- **Node Selection**: Worker nodes only (`node-role.kubernetes.io/worker: worker`).
- **Over-Provisioning**: 200% (`storageOverProvisioningPercentage: 200`).
- **Minimal Available Storage Threshold**: 15% (`storageMinimalAvailablePercentage: 15`).

---

## 2. Component Resource Allocations

| Component | Role | CPU Request | Memory Request | CPU Limit | Memory Limit |
| :--- | :--- | :---: | :---: | :---: | :---: |
| **Longhorn Manager** | DaemonSet on all nodes | `200m` | `512Mi` | `1000m` | `1024Mi` |
| **Longhorn CSI Driver** | Controller & Node plugins | `100m` | `256Mi` | `500m` | `512Mi` |
| **Longhorn UI** | Web Dashboard (2 Replicas) | `50m` | `64Mi` | `200m` | `256Mi` |

---

## 3. GitOps Management via ArgoCD

Longhorn is deployed and reconciled declaratively by ArgoCD using:
- **Application Manifest**: [`gitops/apps/longhorn.yaml`](file:///home/mado/prod-cluster/gitops/apps/longhorn.yaml)
- **Helm Values**: [`bootstrap/longhorn/values-prod.yaml`](file:///home/mado/prod-cluster/bootstrap/longhorn/values-prod.yaml)

---

## 4. Operational Commands

```bash
# Check Longhorn pods status
kubectl --kubeconfig ~/.kube/config-prod get pods -n longhorn-system

# Check StorageClasses
kubectl --kubeconfig ~/.kube/config-prod get sc

# Check Longhorn nodes
kubectl --kubeconfig ~/.kube/config-prod get nodes.longhorn.io -n longhorn-system

# Test creating a test PVC
kubectl --kubeconfig ~/.kube/config-prod apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
  namespace: default
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: longhorn
  resources: { requests: { storage: 1Gi } }
EOF
```
