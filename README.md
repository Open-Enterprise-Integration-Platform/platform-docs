# Open Enterprise Integration Platform

**Enterprise Integration Without Vendor Lock-In.** This repository contains the architecture, governance, and delivery standards for a complete, ready-to-deploy **Productized Enterprise Integration Operating Model**. The Platform is designed as an open, composable alternative to traditional, closed iPaaS/ESB solutions (such as MuleSoft, Boomi, or TIBCO).

With this Platform, organizations do not have to build their integration architecture from scratch. Instead, they implement a proven, scalable operating standard that guarantees full freedom from proprietary ecosystems.

## The Problem We Solve

Modern organizations often face system silos, a lack of standardization (observability and governance), and the high cost of maintaining point-to-point integrations. Classic enterprise integration platforms attempt to solve these issues but introduce severe new ones: exorbitant core-based licensing costs, "black box" infrastructure, and deep dependency on a single provider (vendor lock-in).

This Platform eliminates these barriers by utilizing a 100% open-source execution engine and a strict Layered Architecture, allowing for a drastic reduction in time-to-production while maintaining rigorous enterprise compliance.

## The Platform Manifesto: Core Paradigms

The Platform functions as a **Composable, Headless Integration Platform**. It is built upon the following non-negotiable paradigms:

1. **The 80/20 Rule & Tiered Composability:** 80% of the platform enforces a strict, opinionated operating standard (Tier A - The Control Plane). The remaining 20% allows organizations to safely replace infrastructure blocks (Tier B - Component Swap) with their corporate standards (e.g., swapping OSS Kafka for Azure Event Hubs) without rewriting business logic.
2. **Layered Technology Stack:** The architecture is strictly separated into dedicated layers: *Edge, Experience, Integration, Processing, Event, Persistence, Security, Registry, Automation,* and *Observability*.
3. **Stateless by Design & Thin Routes:** Integration code (Tier C) is built using standard **Apache Camel** (for routing and mediation) and **Quarkus** (for complex domain processing). All integration pods are entirely stateless and ephemeral.
4. **GitOps & IaC Delivery:** Manual configuration is strictly prohibited. Every infrastructure change and integration route is deployed declaratively via GitOps (e.g., ArgoCD), treating Git as the absolute Single Source of Truth.
5. **Native Observability:** Telemetry collection (OpenTelemetry tracing, Prometheus metrics, structured JSON logging) and Zero-Trust security are built-in platform primitives, not optional developer add-ons.

## Why Choose This Platform?

By adopting this architectural framework, you are not just acquiring a set of open-source tools. You are implementing a **Repeatable Operating Standard**.

* **Zero Vendor Lock-In:** Retain 100% ownership of your integration logic and infrastructure choices.
* **Golden Path Delivery:** Domain developers bootstrap integrations in minutes using certified templates with pre-configured CI/CD Quality Gates.
* **Multi-Cloud Readiness:** Deployable on any CNCF-compliant Kubernetes distribution (AKS, EKS, OpenShift, or On-Premise).
* **Enterprise Governance:** Built-in multi-tenancy isolation, Resource Quotas, and automated Software Bill of Materials (SBOM) generation.

---

## Documentation Structure

The documentation in this repository is divided into specific domains. Please refer to the directories below for detailed standards:

* `/architecture` - Core principles, layered infrastructure standards, support models, and component swap guidelines.
* `/delivery` - Developer onboarding, CI/CD Quality Gates, GitOps workflows, and "Thin Route" coding patterns.
* `/governance` - Team topologies, multi-tenancy isolation, contract lifecycle (OAS/AsyncAPI), and RBAC identity models.
* `/operations` - Observability standards, incident response (YBIYRI), disaster recovery, and state management.

### Current delivery decisions (2026-07)

* [Deployment & packaging model (ADR)](docs/architecture/13-deployment-and-packaging-model.md) — self-managed + private release artifacts
* [`platform.yaml` contract](docs/operations/platform-yaml.schema.md)
* [Client install permission matrix](docs/operations/01-client-install-permission-matrix.md)
* [Phase 1 operator console map (no Backstage)](docs/operations/04-phase1-operator-console-map.md)

Camel golden-path template (local Compose): `platform-iac-core/templates/quarkus-camel-app/`

---
*This repository serves as the definitive architectural blueprint. For implementation details, please refer to the specific Layer Standards within the documentation directories.*
