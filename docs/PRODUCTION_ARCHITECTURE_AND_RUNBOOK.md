# Obelion & Syntera Production Infrastructure Documentation

> **Environment:** Production (`syntera-prod`)  
> **Kubernetes Engine:** RKE2 `v1.35.7+rke2r1` (Ubuntu 24.04.4 LTS)  
> **Topology:** 3 Control-Plane Nodes + 6 Worker Nodes  
> **Offsite Backups:** AWS S3 (`s3://obelion-prod-cluster-backups` in `us-east-1`)  
> **Monitoring:** `https://monitoring-prod.obelion.ai` (Dedicated Host: `10.73.31.97`)  
> **Date:** September 2026 | **Status:** Audited & Active  

---

## 1. Executive Summary & Architectural Blueprint

The Obelion & Syntera production infrastructure is designed around high availability, zero single points of failure (No-SPOF), strict GitOps discipline via ArgoCD, automated offsite disaster recovery, and isolated out-of-band monitoring.

### Key Principles
1. **Zero Manual Drift (GitOps-only):** All workloads, configs, and storage rules are defined in Git repositories. Direct `kubectl apply` commands on production are prohibited.
2. **High Availability Compute & Storage:** Multi-master RKE2 control plane with distributed Longhorn block storage replicating data across independent physical worker nodes.
3. **Enterprise Database Resilience:** CloudNativePG 3-instance PostgreSQL HA cluster with synchronous replication, continuous WAL streaming, and automated failover.
4. **Automated Disaster Recovery:** Multi-tier scheduled backups to AWS S3 with automated non-destructive restore validation.
5. **Out-of-Band Observability:** An independent virtual machine hosting VictoriaMetrics, Loki, vmalert, Alertmanager, and Grafana exposed via SSL at `https://monitoring-prod.obelion.ai`.

---

## 2. Network Topology & IP Allocation

| Node / Identifier | Role | Internal IP | External IP | Specs / Notes |
| :--- | :--- | :--- | :--- | :--- |
| `syntera-k8s-prod-cp-01` | Control Plane | `10.73.31.86` | N/A | RKE2 Master 1, etcd Quorum |
| `syntera-k8s-prod-cp-02` | Control Plane | `10.73.31.87` | N/A | RKE2 Master 2, etcd Quorum |
| `syntera-k8s-prod-cp-03` | Control Plane | `10.73.31.88` | N/A | RKE2 Master 3, etcd Quorum |
| `syntera-k8s-prod-worker-01` | Worker Node | `10.73.31.89` | N/A | Workloads + Longhorn Storage |
| `syntera-k8s-prod-worker-02` | Worker Node | `10.73.31.90` | N/A | Workloads + Longhorn Storage |
| `syntera-k8s-prod-worker-03` | Worker Node | `10.73.31.91` | N/A | Workloads + Longhorn Storage |
| `syntera-k8s-prod-worker-04` | Worker Node | `10.73.31.92` | N/A | Workloads + Longhorn Storage |
| `syntera-k8s-prod-worker-05` | Worker Node | `10.73.31.93` | N/A | Workloads + Longhorn Storage |
| `syntera-k8s-prod-worker-06` | Worker Node | `10.73.31.94` | N/A | Workloads + Longhorn Storage |
| `syntera-k8s-ingress-vip` | MetalLB VIP | `10.73.31.96` | N/A | Istio IngressGateway (Ports 80/443) |
| `syntera-prod-monitor-01` | Monitoring VM | `10.73.31.97` | `185.139.123.210` | VictoriaMetrics, Loki, Grafana |
| `Default Gateway` | Router / NAT | `10.73.31.1` | `185.139.123.210` | Port-forwarding & Datacenter NAT |

---

## 3. Ingress & Domain Directory

All inbound domains are managed through Cloudflare (Proxied) and route to either the internal MetalLB VIP (`10.73.31.96`) via Istio IngressGateway, or to dedicated hosts:

| Public Domain | Namespace | Backend Service | Description |
| :--- | :--- | :--- | :--- |
| `bayan.obelion.ai` | `bayan-prod` | `frontend-app:3000` | Bayan Next.js Web Application |
| `bayan-api.obelion.ai` | `bayan-prod` | `backend-app:8000` | Bayan FastAPI Core Engine |
| `bayan-langfuse.obelion.ai` | `langfuse-bayan` | `langfuse-web:3000` | LLM Observability & Tracing |
| `syntera-studio.obelion.ai` | `swe-syntera-prod` | `swe-frontend:3000` | SWE Syntera Studio Web UI |
| `syntera-studio-api.obelion.ai` | `swe-syntera-prod` | `swe-backend:8000` | SWE Syntera Core API |
| `syntera-galaxy.syntera.ai` | `syntera-marketplace-prod` | `marketplace-ui:3000` | Syntera Marketplace UI |
| `syntera-galaxy-api.syntera.ai` | `syntera-marketplace-prod` | `marketplace-api:8000` | Syntera Marketplace API |
| `syntera-galaxy-agenticbuilder.syntera.ai` | `agentic-builder-prod` | `agentic-ui:3000` | Agentic Builder Frontend |
| `syntera-galaxy-agenticbuilder-api.syntera.ai` | `agentic-builder-prod` | `agentic-api:8000` | Agentic Builder Backend API |
| `syntera-argocd.obelion.ai` | `argocd` | `argocd-server:80` | GitOps Control Plane UI |
| `syntera-longhorn.obelion.ai` | `longhorn-system` | `longhorn-frontend:80` | Longhorn Storage Management |
| `syntera-vault.obelion.ai` | `vault` | `vault:8200` | HashiCorp Vault Secrets UI |
| `monitoring-prod.obelion.ai` | External (`10.73.31.97`) | Nginx -> Grafana:3000 | Central Observability Portal |

---

## 4. GitOps Repositories & Delivery Workflows

ArgoCD synchronizes state across several focused repositories:

1. **`prod-cluster`** (`github.com/mohamed-abouseada121/prod-cluster.git`):
   - Branch: `main`
   - Role: Root GitOps repo. Manages Istio gateways, MetalLB, Longhorn recurring jobs, ARC controllers, and monitoring RBAC.
2. **`Bayan`** (`github.com/Obelion-ai/Bayan.git`):
   - Branch: `prod`
   - Role: Bayan application, Langfuse tracing, and daily DB backup CronJob.
3. **`swe_syntera`** (`github.com/Obelion-ai/swe_syntera.git`):
   - Branch: `prod` & `main`
   - Role: Syntera Studio, Keycloak, CloudNativePG cluster definition, and MongoDB backup CronJob.
4. **`syntera-marketplace`** (`github.com/opex-sa/syntera-marketplace.git`):
   - Branch: `prod`
   - Role: Marketplace microservices, Discord bot, and multi-engine backup CronJob.
5. **`Agentic-builder`** (`github.com/opex-sa/Agentic-builder.git`):
   - Branch: `prod`
   - Role: Agentic builder engine, MySQL database, and backup CronJob.

---

## 5. Production Database Tier & HA Specifications

### 1. CloudNativePG (`syntera-db`)
- **Engine:** PostgreSQL 17.2
- **Topology:** 3 Instances (`syntera-postgres-1` [Primary], `2`, `3` [Standbys])
- **Endpoints:**
  - Read-Write: `syntera-postgres-rw.syntera-db:5432`
  - Read-Only: `syntera-postgres-ro.syntera-db:5432`
  - Connection Pooler: `syntera-postgres-pooler.syntera-db:5432`
- **Archiving & Backup:** Continuous WAL archiving to S3 + daily full backup at 01:00 UTC.

### 2. Bayan PostgreSQL (`bayan-prod`)
- **Engine:** PostgreSQL 18.0 (Alpine)
- **Service:** `postgres-service.bayan-prod:5432`
- **Backup:** Daily dump at 02:00 UTC to `databases/bayan/`.

### 3. SWE Syntera MongoDB (`swe-syntera-prod`)
- **Engine:** MongoDB 6.0.28
- **Service:** `mongo.swe-syntera-prod:27017`
- **Backup:** Daily archive at 02:30 UTC to `databases/swe-syntera/`.

### 4. Syntera Marketplace DBs (`syntera-marketplace-prod`)
- **Engines:** MongoDB 6.0.28 + PostgreSQL 18.0 (Discord Bot)
- **Backup:** Daily multi-dump at 03:00 UTC to `databases/marketplace/`.

### 5. Agentic Builder MySQL (`agentic-builder-prod`)
- **Engine:** MySQL 8.4.11 LTS
- **Service:** `dev-db.agentic-builder-prod:3306`
- **Backup:** Daily dump at 03:30 UTC to `databases/agentic-builder/`.

---

## 6. Disaster Recovery & Offsite Backup Strategy

### AWS S3 Storage Architecture
- **Bucket:** `s3://obelion-prod-cluster-backups`
- **Region:** `us-east-1`
- **Access Secret:** `aws-s3-backup-credentials` provisioned across all application namespaces.

### Backup Schedule Matrix

| Workload | Schedule (UTC) | Target S3 Path | Retention |
| :--- | :--- | :--- | :--- |
| **CloudNativePG HA** | `0 1 * * *` (01:00) | `cnpg/syntera-postgres/base/` | 30 Days + WAL stream |
| **Bayan PG18** | `0 2 * * *` (02:00) | `databases/bayan/` | 30 Days |
| **SWE Syntera Mongo** | `30 2 * * *` (02:30) | `databases/swe-syntera/` | 30 Days |
| **Marketplace DBs** | `0 3 * * *` (03:00) | `databases/marketplace/` | 30 Days |
| **Agentic Builder MySQL** | `30 3 * * *` (03:30) | `databases/agentic-builder/` | 30 Days |
| **Longhorn CSI Volumes** | `0 4 * * *` (04:00) | `longhorn/` | 7 Daily / 4 Weekly |
| **Longhorn Local Snapshots** | `0 * * * *` (Hourly) | Local Worker Disks | 24 Hourly |

---

## 7. Monitoring, Logging & Alerting

### Dedicated Host Specifications
- **Hostname:** `syntera-prod-monitor-01` (`10.73.31.97`)
- **Public Domain:** `https://monitoring-prod.obelion.ai`
- **Web Server:** Nginx 1.24 with Let's Encrypt SSL & daily auto-renew cron (`/etc/cron.d/certbot-renewal`).

### Stack Components
- **VictoriaMetrics (`:8428`):** Scrapes Prometheus metrics across nodes, pods, and cluster components.
- **vmalert (`:8880`):** Evaluates alerting rules every 15 seconds.
- **Alertmanager (`:9093`):** Deduplicates and routes alerts to Discord.
- **Grafana 11 (`:3000`):** Visual dashboards with preconfigured metrics and logs.
- **Loki 3 (`:3100`):** Log ingestion and querying.

### Real-Time Alert Catalog
- **Infrastructure:** `NodeNotReady`, `NodeDiskSpaceFillingUp`, `NodeHighMemoryPressure`, `KubePodCrashLooping`, `ContainerOOMKilled`, `DeploymentReplicasUnavailable`, `PersistentVolumeFillingUp`, `VaultSealedOrDown`.
- **Applications:** `BayanBackendDown`, `LangfuseClickHouseDiskPressure`, `KeycloakAuthenticationDown`, `SWEBackendDown`, `SWEMongoDBDown`, `MarketplaceBackendDown`, `AgenticBuilderBackendDown`.
- **Backups:** `DatabaseBackupJobFailed` (fires if any database backup CronJob exits with code != 0).
- **Notifications:** Discord Webhook with formatted embed cards and automatic resolution (`send_resolved: true`).

---

## 8. Operations & Emergency Runbook

### 1. Mandatory Kubeconfig Parameter
Always use the production kubeconfig:
```bash
kubectl --kubeconfig ~/.kube/config-prod <command>
```

### 2. Manual Backup Trigger
To trigger on-demand backups:
```bash
# Bayan
kubectl --kubeconfig ~/.kube/config-prod create job --from=cronjob/bayan-db-backup manual-bayan -n bayan-prod

# SWE Syntera
kubectl --kubeconfig ~/.kube/config-prod create job --from=cronjob/mongo-backup manual-swe -n swe-syntera-prod

# Marketplace
kubectl --kubeconfig ~/.kube/config-prod create job --from=cronjob/marketplace-db-backup manual-mkt -n syntera-marketplace-prod

# Agentic Builder
kubectl --kubeconfig ~/.kube/config-prod create job --from=cronjob/agentic-builder-db-backup manual-agentic -n agentic-builder-prod
```

### 3. Non-Destructive Restore Testing
Never overwrite live production tables. Always launch an ephemeral test container mounting the backup secret and verify row counts before deleting the test pod.
