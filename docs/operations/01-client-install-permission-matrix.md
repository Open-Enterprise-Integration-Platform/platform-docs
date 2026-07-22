# Client Install Permission Matrix

Self-managed install: **client SecOps prepares identity and scope once**; **client CI applies** using OIDC. The vendor does not retain Owner access.

Related: [13-deployment-and-packaging-model.md](../architecture/13-deployment-and-packaging-model.md), [platform-yaml.schema.md](./platform-yaml.schema.md).

---

## 1. Actors

| Actor | Role |
|-------|------|
| Client SecOps | Creates RGs, cloud identity, OIDC federation, scoped RBAC |
| Client Platform CI | Terraform / Helm / Argo bootstrap via OIDC |
| Client Domain CI | Build & push integration images (registry push only) |
| Vendor engineer | No standing cloud access; support via PR / pair session |

---

## 2. Azure (cycle 1) — who needs what

### A. Human: Client Prep (once per environment)

Run `platform-iac-core/bootstrap/client-prep.sh` (macOS/Linux) or `client-prep.ps1` (Windows).

Required to run prep:

| Permission | Why |
|------------|-----|
| Create Resource Groups | mgmt + platform + aks-nodes |
| Create User-Assigned Managed Identity + federated credential | Secret-zero OIDC for GitHub Actions |
| Assign RBAC on **those RGs only** | Scope automation without subscription Owner for vendors |

Typical built-in roles for the **person** running prep (client employee): Owner **or** User Access Administrator + Contributor **on the target RGs / subscription slice their landing zone allows** — decided by client SecOps, not by the vendor.

Prep does **not** need to be run by the vendor.


### B. Automation: Platform Apply identity (OIDC)

Scoped to RGs created by client-prep:

| Role | Scope | Why |
|------|-------|-----|
| Contributor | `rg-oeip-mgmt-*`, `rg-<client>-<env>-foundation`, `rg-<client>-<env>-aks-nodes` | Create AKS, ACR, data, ACI runner, etc. |
| Role Based Access Control Administrator *(narrow)* | Same RGs only | Terraform role assignments (ACR pull, KV access, Workload Identity) |
| Storage Blob Data Contributor | tfstate storage account | Remote state with `use_azuread_auth` |

**Avoid:** subscription-wide Contributor / RBAC Admin as the default product path.

Optional hardening (recommended later): separate **plan** (Reader) and **apply** (Contributor) identities per environment.

### C. Automation: Domain (integration) CI identity

| Role | Scope |
|------|-------|
| AcrPush (or vendor-neutral registry push) | Platform ACR / Harbor project only |

Never subscription Contributor.

### D. Workload identities (in-cluster)

| Workload | Role | Scope |
|----------|------|-------|
| External Secrets | Key Vault Secrets User | Platform Key Vault |
| App / Camel pods | As designed per integration (KV Secrets User, etc.) | Least privilege per namespace |

---

## 3. GitHub / Git hosting

| Secret / var | Who sets | Notes |
|--------------|----------|-------|
| `AZURE_CLIENT_ID` / `TENANT_ID` / `SUBSCRIPTION_ID` | Client after prep | Environment-scoped |
| OIDC (`id-token: write`) | Workflow permission | No long-lived ARM client secret |
| Registry credentials | Prefer OIDC / Workload Identity over static passwords | |
| Argo CD repo access | Deploy key or GitHub App (preferred over broad PAT) | Least privilege to tenant + release repos |

Replace wide `PAT_TOKEN` usage as part of installer hardening (Phase 2).

---

## 4. What the vendor never needs for steady state

- Owner on client subscription  
- Application Administrator in client Entra (after prep is done)  
- Permanent `kubectl` admin kubeconfig  
- Fork of client production data stores  

Break-glass support: time-bound role assignment **created by the client**, ticketed and revoked after the session.

---

## 5. Cross-platform prep scripts

| OS | Artifact |
|----|----------|
| macOS / Linux | `platform-iac-core/bootstrap/client-prep.sh` |
| Windows | `platform-iac-core/bootstrap/client-prep.ps1` |
| CI | Linux runners execute apply; prep remains human/SecOps |

Same outputs (`AZURE_CLIENT_ID`, RG names, storage). No logic drift between shells.

---

## 6. Mapping to previous blocker

The former `azure-bootstrap-tenant` path assumed **subscription Owner-class** rights and often **subscription RBAC Admin** for the deploy identity. That path is **lab-only / legacy**.

Product path = `bootstrap/client-prep.sh|.ps1` + this matrix + `platform.yaml` + release-pinned module sources.
