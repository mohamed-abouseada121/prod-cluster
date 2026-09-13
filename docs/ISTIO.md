# Istio Service Mesh & Ingress Gateway Guide: Production Cluster

This document outlines the architecture, scaling policies, and operational commands for Istio on the production cluster.

---

## 1. Overview

Istio serves as both the Service Mesh control plane and the primary Ingress Gateway controller for all incoming external production traffic.

- **Version**: 1.30.3
- **Namespace**: `istio-system`
- **External Ingress IP**: `10.73.31.96` (via MetalLB `syntera-prod-pool`)
- **Manifest Location**: [`bootstrap/istio/`](file:///home/mado/prod-cluster/bootstrap/istio/)

---

## 2. Component Specifications

### 2.1 Control Plane (`istiod`)
- **Deployment**: `istiod`
- **Replicas**: 2 (Autoscaling via HPA: min 2, max 5)
- **Target CPU Utilization**: 80%
- **Resources**:
  - Requests: `cpu: 500m`, `memory: 1Gi`
  - Limits: `cpu: 2000m`, `memory: 2Gi`
- **Node Affinity**: Worker nodes only (`node-role.kubernetes.io/worker: worker`).

### 2.2 Ingress Gateway (`istio-ingressgateway`)
- **Deployment**: `istio-ingressgateway`
- **Service Type**: `LoadBalancer`
- **External IP**: `10.73.31.96`
- **Ports**:
  - `80`: HTTP traffic (redirected or routed to virtual services)
  - `443`: HTTPS TLS termination
  - `15021`: Health & status port
- **Replicas**: 2 (Autoscaling via HPA: min 2, max 5)
- **Resources**:
  - Requests: `cpu: 200m`, `memory: 256Mi`
  - Limits: `cpu: 1000m`, `memory: 1024Mi`

---

## 3. Operational Verification

```bash
# Check Istio deployments and pods
kubectl --kubeconfig ~/.kube/config-prod get pods -n istio-system -o wide

# Check Ingress Gateway LoadBalancer status
kubectl --kubeconfig ~/.kube/config-prod get svc -n istio-system istio-ingressgateway

# Check HPA status
kubectl --kubeconfig ~/.kube/config-prod get hpa -n istio-system
```
