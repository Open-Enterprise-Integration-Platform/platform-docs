# Internal: Platform Bootstrapping and IaC Guidelines

## Document Purpose
**INTERNAL USE ONLY.** This document is the definitive guide for Platform Engineers responsible for instantiating the Platform for a new client or tenant. It defines the Infrastructure as Code (IaC) repository structure, the strict sequencing of Day-0 (Provisioning) and Day-1 (Configuration) operations, and the handoff process to GitOps.

---

## 1. Infrastructure Repository Topology

To prevent state conflicts and ensure blast-radius isolation, the Platform infrastructure code must be strictly separated into dedicated repositories rather than a single monolith.

* **`platform-iac-foundation` (Terraform):** Contains the code for provisioning the base cloud resources (e.g., Virtual Networks, AKS Clusters, IAM Roles).
* **`platform-iac-tier-b` (Terraform):** Contains the code for provisioning managed cloud services acting as Tier B components (e.g., Azure Event Hubs, Azure Postgres, Azure Cache for Redis).
* **`platform-gitops-core` (ArgoCD/Helm):** Contains the Kubernetes manifests for the Control Plane (Tier A) and in-cluster infrastructure (APISIX, Keycloak, Observability stack).

---

## 2. The Bootstrapping Sequence (Order of Operations)

Standing up a new instance of the Platform requires a strict, sequential pipeline. Attempting to deploy higher-level components before foundational layers exist will result in deployment failures.

### Phase 1: Cloud Foundation (Day-0)
* **Tool:** Terraform (`platform-iac-foundation`)
* **Actions:**
  1. Provision the foundational networking (VNet/VPC, Subnets, Firewalls).
  2. Provision the base Kubernetes Cluster (e.g., Azure Kubernetes Service - AKS).
  3. Provision the central Container Registry (e.g., Harbor or ACR).
  4. Establish core Cloud Identity and permissions (e.g., Workload Identity).

### Phase 2: Tier B Managed Services (Day-0)
* **Tool:** Terraform (`platform-iac-tier-b`)
* **Actions:**
  1. Provision the Persistence Layer (Managed PostgreSQL, Managed Redis).
  2. Provision the Event/Messaging Layer (Managed Kafka / Event Hubs).
  3. Provision the Cloud Key Vault (for secret storage).
  4. *Output:* Terraform must output the connection strings and inject them directly into the Key Vault. No human should see or copy these credentials.

### Phase 3: GitOps Handoff & Core Operators (Day-1)
* **Tool:** CLI / Terraform (Bootstrapping Scripts)
* **Actions:**
  1. Install the GitOps Controller (ArgoCD) onto the raw Kubernetes cluster.
  2. Connect ArgoCD to the `platform-gitops-core` repository.
  3. *From this exact moment, `kubectl apply` is strictly prohibited. All further changes are pulled by ArgoCD.*

### Phase 4: Control Plane Instantiation (Day-1 via GitOps)
* **Tool:** ArgoCD
* **Actions:** ArgoCD automatically deploys the Platform layers in the following synchronized order:
  1. **Security Layer:** External Secrets Operator (connecting to Cloud Key Vault), Cert-Manager.
  2. **Observability Layer:** Prometheus, Loki, OpenTelemetry Collectors.
  3. **Identity Layer:** Keycloak (connecting to the Phase 2 Postgres database).
  4. **Edge Layer:** Apache APISIX and Ingress Controllers.

### Phase 5: The Golden Path Injection (ongoing delivery)
* **Tool:** CI/CD Automation
* **Actions:** 1. Instantiate the "Template Repositories" for the client.
  2. Provision the initial Domain Namespaces in Kubernetes (e.g., `platform-finance-prod`).
  3. Hand over the Developer Onboarding Guide to the client's Domain Teams.

---

## 3. "Secret Zero" Management

The "Secret Zero" problem refers to the initial credentials required to unlock the automation tools (e.g., the ArgoCD admin password or the initial Terraform State access keys).

* **Terraform State:** Must be stored in a highly restricted, encrypted remote backend (e.g., Azure Storage Account with restricted RBAC and Blob Versioning enabled).
* **Cluster Access:** The Platform Team accesses the cluster via temporary, federated OIDC tokens (e.g., Entra ID to AKS), never via static `kubeconfig` certificates.
* **Vault Unsealing:** If using HashiCorp Vault inside the cluster, it must be configured for Auto-Unseal using the native Cloud KMS (e.g., Azure Key Vault key) provisioned in Phase 1.
