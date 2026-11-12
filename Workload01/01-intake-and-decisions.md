# Service Parts Exchange (SPX) | onboarding intake

Status: proposed design; not an approved deployment. Workload ID: Workload01. Environment: Azure EU. Source: workload request supplied by the business on 12 November 2026. This repository had no existing onboarding templates when this package was prepared.

## Business and ownership

| Item | Requirement / owner |
|---|---|
| Application | Service Parts Exchange (SPX), Kobold After-Sales |
| Purpose | Service-centre technicians search spare parts, check stock and order parts for customer repairs |
| Business owner | Thomas Brandt, Kobold After-Sales |
| Audience | Approximately 400 internal users in Germany and France |
| Delivery / operations | Internal DevOps team, eight members; Terraform and Azure DevOps |
| Environments | DEV, QAS, PRD; isolated deployments and data |

## Requirements traceability

| ID | Requirement | Proposed implementation / verification |
|---|---|---|
| F01 | Parts lookup, stock check, ordering | Java 17 / Spring Boot service and ~50 GB PostgreSQL; application functional tests and stock-age indicator |
| I01 | Windows credential SSO via on-prem AD Kerberos | Browser SPNEGO, HTTP SPN and per-service keytab; validate from Germany and France on domain-joined endpoints, plus expired-ticket and AD-unavailable cases |
| I02 | Wuppertal WMS hourly file over SFTP onto SPX servers | Private SPX SFTP ingress tier behind internal endpoint; landing files transferred to EU storage for idempotent ingestion; validate format, checksum, duplicate, late and partial deliveries |
| I03 | DHL shipping API outbound HTTPS | Controlled HTTPS egress; store API credential in EU Key Vault; test sandbox, production allowlisting and retry/idempotency |
| N01 | Availability 99.95% | PRD multi-zone app and database HA; synthetic journey monitoring and a measured monthly SLI (30-day budget ~21.6 minutes); this is a target, not a promised Azure SLA |
| N02 | RPO <= 1 hour | Same-region database HA plus cross-region warm replica and backup; monitor replica lag; test actual recoverability including file ingestion |
| N03 | RTO <= 4 hours | Warm EU region recovery plan and timed, end-to-end exercise; test AD connectivity, DNS, SFTP and DHL before acceptance |
| N04 | EU-only data | Germany West Central primary, France Central proposed DR; keep databases, file objects, vaults, logs, backups and data-bearing pipelines in EU; verify all service-specific residency and support settings |
| N05 | Maintenance | Sunday from 22:00 local German/French time (CET/CEST), with duration and notice to be agreed; no exception to 99.95% until business confirms |
| N06 | Audit | Independently retained monthly backup recovery points for at least six months; evidence and restore test required, not just an enabled policy |
| D01 | Team controls the environment | Terraform-managed Azure resources, policy/RBAC, network and deployment; managed database preferred, subject to team's definition of full control |
| D02 | Pipeline | Azure DevOps build, scan, plan, approval, deploy and smoke test for each environment |

## Decisions requiring owner confirmation before provisioning

1. **Database control:** accept managed Azure Database for PostgreSQL Flexible Server (team controls configuration but not guest OS), or require PostgreSQL on team-managed RHEL VMs with a different HA/backup design?
2. **SFTP interpretation:** is a dedicated SPX-managed SFTP ingress VM tier acceptable for 'onto the SPX servers', rather than installing sshd on the application VMs? Obtain WMS source IPs, SSH key policy, file schema, max file size, timezone, atomic-upload convention and resend rules.
3. **Disaster recovery:** approve France Central as secondary EU region; determine whether cross-border processing within the EU is permitted and whether on-prem AD / WAN connectivity survives a Germany West Central failure. Cross-region async replication can lag beyond an hour: the requested RPO is unproven until monitored under peak load and exercised.
4. **Identity:** obtain AD DNS namespace, SPN/FQDN owner, service-account/keytab lifecycle, allowed Kerberos delegation (ideally none), browser trust configuration and fallback behaviour when AD is unreachable.
5. **Operations:** define PRD business hours, exact outage calculation and exclusions, on-call owner, alert recipients, patch-window length and users' time-zone interpretation.
6. **Security / compliance:** confirm data classification (customer names/addresses may appear in shipment requests), retention/deletion for stock and order data, DHL contract/API data residency, privacy review, public egress policy and log redaction rules.
7. **Sizing / costs:** establish concurrent load, request rate, order rate, file volume, database growth, integration timeouts, reserved IP/DNS/cert ownership and budget; 400 users and 50 GB alone are insufficient to size VM/DB SKUs.
8. **Subscriptions and approvals:** identify subscriptions, resource groups, landing-zone address space, private WAN path, DNS zones, approvers, policy exceptions and Azure DevOps project/service connection; no Azure deployment is authorized by this package.

## Acceptance gate

The business owner signs off on requirements and regional placement; identity, network, DBA, security and DevOps owners sign off on their designs; a PRD-like QAS exercise proves the monthly availability measurement, <=60-minute recoverable data point, <=4-hour end-to-end restoration and a recoverable monthly backup older than five months (or a documented staged acceptance until six months have elapsed).
