# Deployment and Packaging Model (ADR)

**Status:** Accepted  
**Date:** 2026-07-20  
**Context:** Phase 1 ships without Backstage. Full Azure BYOC install was blocked by elevated subscription permissions. Product goal is Camel-based integration services, not vendor-operated SaaS.

---

## 1. Decision

The Platform is delivered as a **self-managed platform product**:

| Concern | Ownership |
|---------|-----------|
| Private core (IaC modules, GitOps bases, charts, images) | Platform vendor — clients do **not** fork R&D; they consume **versioned releases** |
| Tenant configuration (`platform.yaml`, overlays, CIDRs, BYO flags) | Client |
| Cloud identity bootstrap (OIDC, RG, scoped RBAC) | Client SecOps (once) |
| Platform component install / Day-2 sync | Client CI + GitOps (Argo CD), pulling vendor releases |
| Integration services (Tier C) | Client code, from vendor **templates** + **reference pipelines** |

We explicitly **do not** require:

- Vendor engineers holding Owner / subscription-wide RBAC Admin on the client cloud
- Clients receiving the full private core repository as their source of truth
- A custom Experience portal (Backstage) in Phase 1 — operators use native product UIs (see operator console map)

Managed / vendor-operated BYOC (vendor control plane in customer cloud) is **out of scope** until a full Azure cycle is proven.

---

## 2. Why this model (industry alignment)

| Model | Fit for OEIP |
|-------|----------------|
| Multi-tenant SaaS | Conflicts with “no vendor lock-in” and client data-plane control |
| Vendor-operated BYOC | Needs standing vendor access; SecOps friction; premature |
| **Self-managed + private artifacts** | Matches Tier A/B/C, component swap, and consulting-led delivery |

Least-privilege install follows common cloud IaC practice: client pre-creates identity and scopes Contributor (and narrow role-assignment rights if needed) to **platform resource groups**, not the whole subscription. Apply runs as **client CI via OIDC**, not as a vendor laptop.

---

## 3. Artifact flow

```text
platform-iac-core / platform-gitops-core   (private)
        │  tagged release (git ref / OCI chart / container image)
        ▼
client tenant repo                         (thin config only)
        │  client CI (OIDC → cloud)
        ▼
client cloud + cluster                     (configurable Tier A/B)
        │
        ▼
client integration repos                   (from Camel template)
```

Clients never need write access to core. Read access is limited to release artifacts required by their `platform.yaml`.

---

## 4. Configuration contract

Install is driven by a declarative `platform.yaml` (schema: `docs/operations/platform-yaml.schema.md`).

Capabilities are toggles aligned with Tier / component-swap assumptions (edge, identity, messaging BYO, etc.). Each client may enable a different subset; the installer must not assume a full greenfield subscription.

---

## 5. Cloud and OS scope

| Topic | Decision |
|-------|----------|
| First full cloud cycle | **Azure** (first potential client) |
| AWS / GCP | Provider modules deferred until Azure cycle works end-to-end; keep module layout ready (`modules/azure` today) |
| Local development | **Kind / Docker Compose** — primary path until a lab subscription exists |
| Client prep scripts | **Bash and PowerShell** equivalents (macOS, Linux, Windows) |

---

## 6. Permission principle (summary)

1. **Client prep (human, once):** create RGs, UAMI/App + OIDC federation, assign scoped roles.  
2. **Platform apply (automation):** OIDC identity with RG-scoped rights only.  
3. **Domain CI:** AcrPush (or equivalent) only — never subscription Contributor.  
4. **Vendor support:** no standing access; JIT / PR into tenant repo when needed.

Detailed matrix: `docs/operations/01-client-install-permission-matrix.md`.

---

## 7. Consequences

**Positive**

- SecOps can approve a one-page prep checklist instead of Owner-on-subscription
- Core IP stays private; clients still get auditable GitOps desired state
- Integration delivery unblocks on local golden path without waiting for Azure lab

**Trade-offs**

- Client (or partner) operates Day-2 platform components
- Release engineering (tags, changelogs, compatibility matrix) becomes mandatory
- AWS/GCP remain stubs until Azure is proven

---

## 8. Phase sequencing (binding)

0. Local Camel golden path (Compose/Kind)  
1. Operator UI runbook (no Backstage)  
2. Configurable self-managed installer (Azure first, Bash + PowerShell prep)  
3. Fake/first client thin tenant repo validating release pull  
4. Client-owned integration services from templates  
5. Later: Backstage, managed BYOC option, AWS/GCP providers  
