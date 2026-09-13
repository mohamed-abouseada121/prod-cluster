# Production Kubernetes Cluster Architecture (syntera-k8s-prod)

This document details the hardware profile, networking model, and component layout for the 9-node production Kubernetes cluster.

---

## 1. Node Topology & Hardware Profile

The cluster is deployed on **Ubuntu 24.04.4 LTS** running **RKE2 (v1.35.7+rke2r1)** with a total capacity of **76 vCPUs** and **144 GB RAM**.

### High Availability Control Plane (3 Nodes)
| Node Name | IP Address | vCPU | Memory | Roles |
| :--- | :---: | :---: | :---: | :--- |
| `syntera-k8s-prod-cp-01` | `10.73.31.86` | 4 | 16 GB | `control-plane,etcd` |
| `syntera-k8s-prod-cp-02` | `10.73.31.87` | 4 | 16 GB | `control-plane,etcd` |
| `syntera-k8s-prod-cp-03` | `10.73.31.88` | 4 | 16 GB | `control-plane,etcd` |

### Worker Nodes (6 Nodes)
| Node Name | IP Address | vCPU | Memory | Roles |
| :--- | :---: | :---: | :---: | :--- |
| `syntera-k8s-prod-worker-01` | `10.73.31.89` | 12 | 16 GB | `worker` |
| `syntera-k8s-prod-worker-02` | `10.73.31.90` | 4 | 16 GB | `worker` |
| `syntera-k8s-prod-worker-03` | `10.73.31.91` | 12 | 16 GB | `worker` |
| `syntera-k8s-prod-worker-04` | `10.73.31.92` | 12 | 16 GB | `worker` |
| `syntera-k8s-prod-worker-05` | `10.73.31.93` | 12 | 16 GB | `worker` |
| `syntera-k8s-prod-worker-06` | `10.73.31.94` | 12 | 16 GB | `worker` |

---

## 2. Network Layout & Ingress Architecture

- **CNI**: Calico (Canal)
- **Pod CIDR**: `10.42.0.0/16`
- **Service CIDR**: `10.43.0.0/16`
- **Load Balancing (MetalLB)**:
  - Mode: Layer 2 (`L2Advertisement` targeting worker nodes)
  - Address Pool (`syntera-prod-pool`): `10.73.31.96 - 10.73.31.97`
- **Service Mesh & Ingress Gateway (Istio)**:
  - `istio-ingressgateway`: Assigned External IP `10.73.31.96` (Ports 80, 443, 15021)
  - Redundancy: 2 Replicas with Horizontal Pod Autoscaler (HPA max: 5)

---

## 3. Storage Architecture (Longhorn)

- **Engine**: Longhorn Distributed Block Storage
- **Replica Factor**: 3-way replication across worker nodes (`defaultReplicaCount: 3`)
- **Anti-Affinity**: Hard anti-affinity (`replicaSoftAntiAffinity: false`) ensures no two replicas reside on the same worker node.
- **Disk Path**: `/var/lib/longhorn`
