# MetalLB Load Balancer Guide: Production Cluster

This document outlines the configuration, management, and verification of MetalLB on the production cluster.

---

## 1. Overview & Components

MetalLB provides bare-metal / on-premise network load balancing for Kubernetes services of type `LoadBalancer`.

- **Controller**: Deployment running in `metallb-system`, handles IP allocation.
- **Speaker**: DaemonSet running on all nodes with `hostNetwork: true`, handles L2 ARP/NDP advertisement.
- **Manifest Location**: [`bootstrap/metallb/`](file:///home/mado/prod-cluster/bootstrap/metallb/)

---

## 2. IP Address Pool (`IPAddressPool`)

Defined in [`bootstrap/metallb/ipaddresspool.yaml`](file:///home/mado/prod-cluster/bootstrap/metallb/ipaddresspool.yaml):

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: syntera-prod-pool
  namespace: metallb-system
spec:
  addresses:
    - 10.73.31.96-10.73.31.97
  autoAssign: true
  avoidBuggyIPs: false
```

- **Allocated IPs**:
  - `10.73.31.96`: Reserved and bound to `istio-system/istio-ingressgateway`.
  - `10.73.31.97`: Available for secondary ingress or dedicated database/monitoring ingress.

---

## 3. Layer 2 Advertisement (`L2Advertisement`)

Defined in [`bootstrap/metallb/l2advertisement.yaml`](file:///home/mado/prod-cluster/bootstrap/metallb/l2advertisement.yaml):

```yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: syntera-prod-l2
  namespace: metallb-system
spec:
  ipAddressPools:
    - syntera-prod-pool
  nodeSelectors:
    - matchLabels:
        node-role.kubernetes.io/worker: worker
```

Traffic is advertised only from worker nodes, keeping control-plane interfaces dedicated to Kubernetes API and etcd.

---

## 4. Operational Commands

```bash
# Verify MetalLB status
kubectl --kubeconfig ~/.kube/config-prod get pods -n metallb-system

# Check IP address pool allocation
kubectl --kubeconfig ~/.kube/config-prod get ipaddresspools -n metallb-system
```
