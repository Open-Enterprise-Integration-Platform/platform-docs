# Phase 1 Operator Console Map (No Backstage)

Operators use **native product UIs**. Backstage is out of scope for Phase 1 (see ADR `13-deployment-and-packaging-model`).

| Product | Layer | Typical URL / entry | Day-1 / Day-2 tasks |
|---------|-------|---------------------|---------------------|
| **Argo CD** | Automation / GitOps | `https://argocd.<env>.<client>` | Sync status, app health, diff, manual sync, rollback via Git revert |
| **Apache APISIX Dashboard** | Edge | `https://apisix.<env>.<client>` | Routes, upstreams, auth plugins, rate limits |
| **Keycloak** | Identity | `https://auth.<env>.<client>` | Realms, clients, IdP federation, roles (or document corporate IdP if swapped) |
| **Grafana** | Observability | `https://grafana.<env>.<client>` | Dashboards, alerts, explore metrics (logs/traces when enabled) |
| **Apicurio Registry** | Registry | `https://registry.<env>.<client>` | Schemas / AsyncAPI / OpenAPI artifacts |
| **Cloud provider console** | Foundation | Azure Portal (cycle 1) | RG health, AKS, Key Vault, Event Hubs — SecOps owned |

### Access principles

- SSO via platform IdP (Keycloak) or corporate IdP when `identity.mode=federate-existing`
- No shared admin passwords in Git (replace any bootstrap defaults on first login)
- Break-glass local admin only for disaster recovery, vaulted by client

### When a capability is disabled in `platform.yaml`

Omit that row from the client’s runbook. Do not document URLs for products that were not installed.
