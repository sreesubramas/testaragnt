# Workload01 - Service Parts Exchange (SPX)

Proposed Azure onboarding package for Kobold After-Sales; derived from the request of 12 November 2026. **Architecture only: nothing has been deployed, approved or costed.** The attached repository had no templates or existing Workload folders when this package was created, so these are self-contained draft artifacts.

| Artifact | Purpose |
|---|---|
| [01-intake-and-decisions.md](01-intake-and-decisions.md) | Business requirements, traceability, acceptance criteria and unresolved approvals |
| [02-solution-design.md](02-solution-design.md) | EU regional, app, database, SFTP, DHL and resilience design |
| [03-security-and-integrations.md](03-security-and-integrations.md) | Kerberos, network flows, secret protection, integration contracts |
| [04-operations-and-recovery.md](04-operations-and-recovery.md) | SLI/SLO, backup retention, alerts and tested recovery procedure |
| [05-delivery-plan.md](05-delivery-plan.md) | Terraform module scope, Azure DevOps deployment stages, handover gates |
| [architecture.mmd](architecture.mmd) | Mermaid architecture diagram source, renderable by Mermaid-compatible viewer |

First decisions: confirm whether managed PostgreSQL satisfies 'full control'; approve dedicated SPX SFTP ingress VMs; confirm France Central disaster recovery and end-to-end EU-only data handling; specify identity/network details; load-test and sign off on realistic RPO/RTO. See the numbered decisions in the intake. No Terraform apply or production connection should proceed before these choices are resolved.
