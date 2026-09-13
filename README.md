# Production Kubernetes Cluster (`prod-cluster`)

GitOps repository and operational runbooks for the **Syntera Enterprise Production Kubernetes Cluster** (`syntera-k8s-prod`).

---

## 1. High-Level Architecture

The production cluster is a bare-metal/private-cloud **RKE2 (v1.35.7+rke2r1)** deployment configured for high availability across **9 dedicated Ubuntu 24.04.4 LTS nodes** (3 Control-Plane nodes + 6 Worker nodes).

```
                                [ MetalLB Layer 2 ]
                           (syntera-prod-pool: 10.73.31.96)
                                         |
                                         v
                         [ Istio Ingress Gateway (HA) ]
                                (10.73.31.96:443)
                                         |
                     +-------------------+-------------------+
                     |                                       |
                     v                                       v
          [ Production Workloads ]                   [ ArgoCD Server (HA) ]
        (swe-syntera, syntera-db, etc.)              (GitOps Reconciler)
                     |                                       |
                     +-------------------+-------------------+
                                         |
                                         v
                      [ Longhorn Distributed Block Storage ]
                          (3-Way Replication across 6 Workers)
```

---

## 2. Infrastructure Inventory & Node Profile

| Node | Role | IP | vCPU | RAM | Storage / Workload Role |
| :--- | :--- | :---: | :---: | :---: | :--- |
| `syntera-k8s-prod-cp-01` | `control-plane,etcd` | `10.73.31.86` | 4 | 16 GB | RKE2 API, etcd leader |
| `syntera-k8s-prod-cp-02` | `control-plane,etcd` | `10.73.31.87` | 4 | 16 GB | RKE2 API, etcd follower |
| `syntera-k8s-prod-cp-03` | `control-plane,etcd` | `10.73.31.88` | 4 | 16 GB | RKE2 API, etcd follower |
| `syntera-k8s-prod-worker-01` | `worker` | `10.73.31.89` | 12 | 16 GB | General workloads & storage |
| `syntera-k8s-prod-worker-02` | `worker` | `10.73.31.90` | 4 | 16 GB | Control workloads (Istio Pilot) |
| `syntera-k8s-prod-worker-03` | `worker` | `10.73.31.91` | 12 | 16 GB | General workloads & storage |
| `syntera-k8s-prod-worker-04` | `worker` | `10.73.31.92` | 12 | 16 GB | Ingress Gateway, ArgoCD Controller |
| `syntera-k8s-prod-worker-05` | `worker` | `10.73.31.93` | 12 | 16 GB | Ingress Gateway, General workloads |
| `syntera-k8s-prod-worker-06` | `worker` | `10.73.31.94` | 12 | 16 GB | General workloads & storage |

---

## 3. Core Cluster Components & Documentation

Detailed technical documentation for every cluster component is maintained in [`docs/`](docs/):

- 🏛️ **[Cluster Architecture](docs/ARCHITECTURE.md)**: Hardware breakdown, node topology, network CIDRs.
- 🚀 **[ArgoCD GitOps Engine](docs/ARGOCD.md)**: Production HA configuration, Redis HA Sentinel, App-of-Apps setup.
- 💾 **[Longhorn Storage](docs/LONGHORN.md)**: 3-Way replication, hard anti-affinity, StorageClass setup.
- ⚖️ **[MetalLB Load Balancing](docs/METALLB.md)**: IP address pool (`10.73.31.96-97`), L2 advertisements.
- 🌐 **[Istio Service Mesh & Gateway](docs/ISTIO.md)**: Ingress gateway, HPA scaling, mTLS configurations.

---

## 4. Repository Structure

```
prod-cluster/
├── README.md                          # Master overview & operational runbook
├── docs/                              # Technical guides (Architecture, Storage, Networking, GitOps)
├── bootstrap/                         # Production Helm values and deployment specs
│   ├── metallb/                       # IPAddressPool and L2Advertisement specs
│   ├── istio/                         # Istiod and Gateway HA production values
│   ├── longhorn/                      # Longhorn 3-way replication production values
│   └── argocd/                        # ArgoCD HA Redis and autoscaling production values
└── gitops/                            # ArgoCD Application manifests
    ├── root-app.yaml                  # Root App-of-Apps controller
    └── apps/
        ├── longhorn.yaml              # Declarative Longhorn Helm installation
        └── metallb.yaml               # Declarative MetalLB configuration
```

---

## 5. GitOps Quickstart

### Accessing ArgoCD:
```bash
# Retrieve initial admin password
kubectl --kubeconfig ~/.kube/config-prod -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo ""

# Forward ArgoCD UI
kubectl --kubeconfig ~/.kube/config-prod port-forward svc/argocd-server -n argocd 8080:443
```

### Applying Root Application (Self-Reconciling Cluster):
```bash
kubectl --kubeconfig ~/.kube/config-prod apply -f gitops/root-app.yaml
```
Once applied, ArgoCD synchronizes all applications in `gitops/apps/` directly from this Git repository.
