# `platform.yaml` — Install Configuration Contract

Declarative input for a **self-managed** platform install. The client tenant repo holds a concrete file; the installer reads it and enables only the selected capabilities. Core modules/charts are pulled from **private vendor releases**, not developed in the tenant repo.

Related ADR: [13-deployment-and-packaging-model.md](../architecture/13-deployment-and-packaging-model.md).

---

## Example (Azure, greenfield-ish, Phase 1 subset)

```yaml
apiVersion: oeip.platform/v1alpha1
kind: PlatformInstance
metadata:
  client: alpha
  environment: dev
  # Pin to a vendor release — never floating main for production
  platformRelease: "v0.1.0"

spec:
  cloud:
    provider: azure          # azure | aws | gcp (only azure implemented in cycle 1)
    region: westeurope
    # Pre-created by client-prep (required for least-privilege apply)
    resourceGroups:
      management: rg-oeip-mgmt-alpha
      platform: rg-oeip-alpha-dev

  # How artifacts are consumed (client does not fork core)
  artifacts:
    iacModuleSource: "git::https://github.com/Open-Enterprise-Integration-Platform/platform-iac-core.git?ref=v0.1.0"
    gitopsSource: "https://github.com/Open-Enterprise-Integration-Platform/platform-gitops-core.git"
    gitopsPath: "apps/overlays/platform-azure-dev"
    containerRegistry: ""    # filled after foundation exists, or BYO

  networking:
    mode: provision          # provision | bring-your-own
    vnetCidr: "10.100.0.0/16"
    # when mode=bring-your-own:
    # existingVnetId: /subscriptions/.../resourceGroups/.../providers/Microsoft.Network/virtualNetworks/...
    # subnetIds: { aksNodes: ..., aksPods: ..., privateEndpoints: ..., runners: ... }

  cluster:
    mode: provision          # provision | bring-your-own
    private: true
    # when bring-your-own: existingClusterName + resourceGroup

  capabilities:
    # Phase 1 defaults — toggle per client reality
    edge:
      enabled: true
      product: apisix
    identity:
      enabled: true
      product: keycloak       # or bring-your-own corporate IdP (component swap)
      mode: provision        # provision | federate-existing
    secrets:
      enabled: true
      product: external-secrets
    observability:
      enabled: true
      metrics: kube-prometheus-stack
      logs: none             # none | loki (later)
      traces: none           # none | tempo (later)
    registry:
      schemas: apicurio
    serviceMesh:
      enabled: false         # Phase 1: off unless required
    experience:
      backstage: false       # Phase 1: native product UIs only

  dataPlane:
    postgres:
      mode: provision        # provision | bring-your-own | disabled
    redis:
      mode: provision
    messaging:
      mode: provision        # provision | bring-your-own | disabled
      product: azure-event-hubs   # azure-event-hubs | kafka-strimzi (later)

  gitops:
    engine: argocd
    # Repo that Argo watches for *this* tenant desired state (client-owned)
    tenantGitopsRepo: ""

  integrationGoldenPath:
    templateRef: "platform-iac-core/templates/quarkus-camel-app@v0.1.0"
    referencePipeline: true
```

---

## Field semantics

| Field | Meaning |
|-------|---------|
| `platformRelease` | Compatibility pin for IaC + GitOps + chart versions |
| `mode: provision` | Platform modules create the resource |
| `mode: bring-your-own` | Client supplies IDs/endpoints; modules only wire consumers |
| `mode: disabled` / `enabled: false` | Capability skipped entirely |
| `identity.mode: federate-existing` | Skip Keycloak; document OIDC settings for corporate IdP |
| `capabilities.experience.backstage` | Always false in Phase 1 |

Unknown `cloud.provider` values other than `azure` must fail fast with a clear message until cycle 1 completes.

---

## Validation rules (installer)

1. If `networking.mode=bring-your-own`, all required `subnetIds` must be set.  
2. If any `dataPlane.*.mode=provision`, `networking` must yield private endpoint (or equivalent) connectivity.  
3. `artifacts.iacModuleSource` / `gitopsSource` must reference a **tag or digest**, not `main`, for `environment: prod`.  
4. At least one of `edge` or `gitops` must be enabled (otherwise there is no operable control plane).  
5. Client prep outputs (`AZURE_CLIENT_ID`, RG names, OIDC subjects) must match `metadata.client` + `metadata.environment`.

---

## Mapping to Terraform / GitOps (cycle 1)

| `platform.yaml` | Azure cycle 1 |
|-----------------|---------------|
| `networking` + `cluster` | `modules/azure/foundation` |
| `dataPlane.postgres/redis` | `modules/azure/tier-b-data` |
| `dataPlane.messaging` | `modules/azure/tier-b-messaging` |
| `capabilities.secrets` | `modules/azure/security` + ESO GitOps app |
| `capabilities.*` product toggles | Kustomize overlays / Argo Applications |
| `integrationGoldenPath` | Client copies template; not applied by foundation TF |
