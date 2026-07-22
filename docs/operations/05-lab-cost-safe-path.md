# Lab path on a limited Azure credit (€45 VS) — cost-safe verification

**Verdict:** Full OEIP apply (private AKS + ACR Premium + Postgres Flexible + Redis Premium + Event Hubs + NAT) will typically **exceed €45 within hours/days**. Do **not** run `deploy.yaml` / full `main.tf` on this subscription.

Prep (RG + storage + UAMI) is cheap — keep it. Prove the **product path** (Camel → Postman) locally; use Azure only for optional cheap smoke later.

---

## Cost reality (order of magnitude)

| Component | Approx. burn |
|-----------|----------------|
| Prep (storage + identity) | cents–few €/month |
| Private AKS (even 1–2 nodes) | tens of €/day possible |
| ACR Premium | high fixed |
| Postgres Flexible + Redis Premium + EH + NAT | high fixed |
| **Local Docker Compose** | **€0 Azure** |

---

## Recommended test ladder (to Postman)

### A. Cost guard (do this first)

```bash
cd platform-iac-core/bootstrap
chmod +x cost-guard.sh
./cost-guard.sh -e tfraczak@sii.pl -a 35
```

Portal double-check: Subscription → **Budżety** / Cost Management.  
**Leave the VS spending limit enabled.**

### B. Microservice E2E without Azure (primary proof)

```bash
cd platform-iac-core/templates/quarkus-camel-app
docker compose up --build
```

Postman / curl:

- `POST http://localhost:8080/api/v1/ingest`
- Header: `Content-Type: application/json`
- Body: `{ "orderId": "lab-1", "amount": 10 }`
- Expect: **202** `{ "status": "accepted" }`
- Health: `GET http://localhost:8080/q/health`

This validates: template build → Kafka path → HTTP API (your consulting demo).

### C. Do **not** auto-apply full platform

`platform-client-alpha-infra` deploy workflow should be **manual only** (`workflow_dispatch`) so a push cannot burn credit.

### D. Optional later (still cheap): single ACI smoke

Only after Compose works — deploy one public container (no AKS). Separate script/doc when needed. Tear down the same day.

### E. Full platform install

Requires a **paid lab subscription** without a €45 ceiling (or sponsor credit ≥ several hundred €) + `client-teardown` after each session.

---

## What to improve for lower Azure cost (when you have budget)

| Today (expensive) | Lab / PoC cheaper |
|-------------------|-------------------|
| ACR Premium | ACR Basic |
| AKS private + NAT Gateway | Public AKS, `outbound_type=loadBalancer`, 1× `Standard_B2s` |
| Redis Premium | Skip or Azure Cache Basic |
| Postgres Flexible | Skip or Burstable B-series / local Postgres |
| Event Hubs Standard | Local Redpanda / single EH Basic |
| Always-on cluster | Destroy RG after each test day |

Track these as `lab_mode` flags in tenant TF (next increment).

---

## Cleanup prep resources (if pausing for weeks)

```bash
az group delete -n rg-alpha-dev-aks-nodes -y --no-wait
az group delete -n rg-alpha-dev-foundation -y --no-wait
# keep mgmt only if you still need tfstate/OIDC; otherwise:
# az group delete -n rg-oeip-mgmt-alpha -y --no-wait
az group delete -n rg-oeip-cost-guard -y --no-wait   # if cost-guard created it
```
