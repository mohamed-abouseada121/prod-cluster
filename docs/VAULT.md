# HashiCorp Vault Architecture & Operations Guide (syntera-k8s-prod)

This document details the configuration, persistent storage model, network ingress, and data operations for HashiCorp Vault on the **syntera-k8s-prod** cluster.

---

## 1. Overview & Hardware Sizing

- **Namespace**: `vault`
- **Deployment Type**: StatefulSet (`vault-0`) + Vault Agent Injector Daemon/Deployment
- **Storage Engine**: `file` storage backend backed by Longhorn Distributed Block Storage
- **High Availability & Redundancy**: 3-way replication across worker nodes (`defaultReplicaCount: 3`) on StorageClass `longhorn`.

### Resource Allocations & Limits

| Component | CPU Requests | CPU Limits | Memory Requests | Memory Limits | Storage |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **Vault Server (`vault-0`)** | `250m` | `1000m` | `512Mi` | `1024Mi` | `20Gi` (Longhorn PVC) |
| **Agent Injector** | `100m` | `250m` | `128Mi` | `256Mi` | Stateless |

---

## 2. Network Layout & Ingress

- **Internal Service (ClusterIP)**: `vault.vault.svc.cluster.local:8200`
- **Internal Agent Injector**: `vault-agent-injector-svc.vault.svc.cluster.local:443`
- **Public Ingress Endpoint**: `https://syntera-vault.obelion.ai`
  - Ingress Controller: Istio Ingress Gateway (`10.73.31.96`)
  - TLS: Wildcard Certificate `obelion-ai-wildcard-tls` issued by Let's Encrypt via Cert-Manager.

---

## 3. Secret Engines & Schema

The production Vault instance hosts the following KV-v2 secret engines:

1. **`secret/` (KV v2)**:
   - `secret/data/bayan/secrets`: ClickHouse, PostgreSQL, Langfuse, and application API keys for Bayan.
   - `secret/data/swe-syntera/secrets`: PostgreSQL, MongoDB, Redis, Keycloak client secrets, and AI keys for SWE-Syntera.
   - `secret/data/agentic-builder/secrets`: Authentication and worker keys for Agentic Builder.
   - `secret/data/media-supporter/secrets`: Service credentials for Obelion Media Supporter.
2. **`syntera/` (KV v2)**:
   - `syntera/data/secrets`: Shared platform tokens and infrastructure credentials.

---

## 4. Kubernetes Authentication & Roles

Vault is configured with the `kubernetes` authentication method (`auth/kubernetes`), allowing workloads to authenticate using projected Kubernetes ServiceAccount tokens.

### Defined Roles:
- **`swe-app-role`**: Bound to namespace `swe-syntera`, service accounts `vault-auth`, `default`.
- **`bayan-role`**: Bound to namespace `bayan`, service accounts `vault-auth`, `default`.
- **`agentic-builder`**: Bound to namespace `agentic-builder`.
- **`media-supporter`**: Bound to namespace `media-supporter`.
- **`syntera-role`**: Bound to namespace `syntera`.

---

## 5. Security & Zero-Leak Policy

> [!CAUTION]
> Never commit unseal keys, root tokens, or plain-text application secrets to Git.
> - Production unseal keys and root tokens are stored locally under `bootstrap/vault/prod-vault-keys.json` which is ignored by `.gitignore`.
> - Always use `secrets.example.yaml` templates for public documentation.
