# Cert-Manager & SSL/TLS Certificates Guide: Production Cluster

This document details the configuration, ClusterIssuers, and Wildcard SSL/TLS certificates managed on the production cluster.

---

## 1. Overview & Architecture

- **Engine**: Jetstack Cert-Manager
- **Namespace**: `cert-manager`
- **Challenge Type**: ACME DNS-01 via Cloudflare API
- **Target Secret Location**: `istio-system` (Accessible by `istio-ingressgateway`)

---

## 2. ClusterIssuers

| ClusterIssuer Name | Domain Scope | Solver | Secret Name |
| :--- | :--- | :--- | :--- |
| `letsencrypt-cloudflare` | `*.obelion.ai` | Cloudflare DNS-01 | `cloudflare-api-token` |
| `syntera-dns-issuer` | `*.syntera.ai` | Cloudflare DNS-01 | `cloudflare-api-token-syntera` |

---

## 3. Wildcard Certificates

| Certificate Name | Namespace | Domains | TLS Secret Name | Issuer |
| :--- | :--- | :--- | :--- | :--- |
| `obelion-ai-wildcard` | `istio-system` | `*.obelion.ai` | `obelion-ai-wildcard-tls` | `letsencrypt-cloudflare` |
| `syntera-ai-wildcard-cert` | `istio-system` | `*.syntera.ai`, `syntera.ai` | `syntera-ai-wildcard-tls` | `syntera-dns-issuer` |

---

## 4. Verification & Troubleshooting

```bash
# 1. Check cert-manager pods
kubectl --kubeconfig ~/.kube/config-prod get pods -n cert-manager

# 2. Check ClusterIssuers status
kubectl --kubeconfig ~/.kube/config-prod get clusterissuer

# 3. Check Certificates readiness
kubectl --kubeconfig ~/.kube/config-prod get certificate -n istio-system

# 4. Check Challenges progress if pending
kubectl --kubeconfig ~/.kube/config-prod get challenges -A
```
