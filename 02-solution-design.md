# SPX | proposed Azure solution design

Design status: proposed, not provisioned. See [intake and decisions](01-intake-and-decisions.md) for unresolved dependencies; [architecture diagram](architecture.mmd) is the diagram source. Choose resource sizes only after a representative load test.

## Placement and topology

- Separate DEV, QAS and PRD resource groups and virtual networks (prefer separate subscriptions if required by landing-zone policy); no production data copied into lower environments without anonymization and approval. Put PRD primary in Germany West Central; propose France Central as an EU secondary only after data-location and failover approval. Validate the selected SKUs, zone support, subscription quotas and region-pair rules before Terraform apply.
- Private corporate ingress via approved ExpressRoute or site-to-site VPN, with redundant paths, on-prem DNS forwarding and AD/KDC reachability. No public application, SFTP or database endpoint. Use internal L4 load balancer for application HTTPS and a separate internal SFTP endpoint. Terminate HTTPS on each RHEL VM so Kerberos HTTP service principal matches the shared application FQDN; maintain valid certificate/hostname on every node.
- PRD application: two or more Red Hat Enterprise Linux 9 VMs in different availability zones, with supported Java 17 JDK and Spring Boot release; zonal internal load balancer, health probe, autoscaling only after session and keytab behavior is validated. Keep the app stateless; do not store the only copy of stock files, order records or sessions on a VM. Patch and rotate VMs one at a time. QAS mirrors topology enough for failover and restore testing; DEV may be smaller.
- Database: proposed private Azure Database for PostgreSQL Flexible Server, general-purpose tier with zone-redundant HA if supported in the selected region and SKU. Private networking and private DNS, application TLS, least-privilege database roles. Test actual 50 GB migration, transaction load and schema evolution. PostgreSQL HA protects a regional AZ failure, not regional loss; keep backups and a separate, warm cross-region read replica for DR, with promotion documented. An async replica has no hard RPO guarantee: alert on lag and verify at failover.
- Wuppertal WMS pushes an hourly file to an SPX-operated SFTP ingress VM tier behind the private SFTP endpoint. SFTP VMs keep a transient local upload only, then atomically validate and move the completed file to an EU Blob landing area, returning an acknowledgment only after durable transfer (agree protocol with WMS). Ingestion tracks file ID/hash and processing state in PostgreSQL for replay-safe import. Plan multi-zone SFTP nodes with per-node local staging and a reconciler; exercise node loss mid-upload and resend. This avoids coupling the application HTTP tier to sshd or a single shared VM disk.
- DHL: application calls DHL HTTPS through controlled Azure Firewall/NAT egress; store credentials in EU Key Vault, restrict destination where technically supportable, bound retries/backoff, make shipment creation idempotent using an order reference and reconcile ambiguous timeouts with DHL rather than blind re-POST. Avoid customer identifiers in diagnostic logs.
- DR: provision Terraform-managed application/ingress network and compute capacity in the approved secondary EU region, plus private DNS/cert/key access, SFTP destination and database read replica. Keep DR app cold or warm only if tested cutover <=4 hours; keep replica running. Promotion is an explicit irreversible failover decision after checking replication lag and fencing original writers. Switch internal DNS/private routing after DB promotion; re-establish WMS delivery and DHL egress. Treat AD/WAN dependencies as part of the DR test. Reverse replication and failback need a separate runbook.

## Data flow and boundaries

1. Domain-joined technician browser -> corporate DNS -> private LB -> SPX HTTPS -> Kerberos challenge/ticket validated on RHEL VM using SPX AD service-account keytab. The application authorizes users using an agreed role mapping; login success does not alone grant ordering rights.
2. SPX VM -> private PostgreSQL endpoint for parts, stock and order transactions. VM managed identity retrieves permitted secrets/certs from EU Key Vault; central logs/metrics to EU Log Analytics and Application Insights with sensitive fields excluded.
3. WMS -> private WAN -> SFTP endpoint -> transient SPX SFTP VM -> encrypted EU landing storage -> validated import -> database. Indicate timestamp of last successfully processed stock file in UI and alert on stale stock.
4. Order in database -> outbox/idempotency state -> DHL HTTPS -> shipping ID stored in database; capture failures for retry/manual review without silently creating duplicate shipments. Outbox processing must be deployed in both regions but only one region may write after failover.

## Reliability and recovery boundaries

| Failure | Proposed response | Acceptance evidence |
|---|---|---|
| VM/AZ | LB drains failed node; app keeps serving; database zone-HA failover | Quarterly AZ/node and DB failover drill, user journey and error-budget analysis |
| WMS file delayed/duplicate | Retain last known stock with timestamp; alert after >75 minutes since last successful import (tune to schedule); dedupe on file ID/hash | Scheduled missed-file and resend tests |
| DHL unavailable | Queue/retry with cap and order status 'shipment pending'; do not claim shipment completed | API fault-injection, duplicate prevention |
| Region outage | Promote warm EU database replica; activate app/ingress; switch DNS and verify dependencies | Timed RTO/RPO exercise <=4h and <=1h; not guaranteed by design alone |
| Logical corruption / deletion | Point-in-time operational restore or separately retained monthly recovery point | Restore sample and checksum / application-level consistency test |

## Product documentation used to bound the proposal

- Azure PostgreSQL read replicas are asynchronous and can lag for hours under load: https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/concepts-read-replicas
- Azure Backup for PostgreSQL Flexible Server GA long-term solution uses scheduled logical backups and supports retention beyond the operational 35-day limit; v2 is preview and must not be assumed production-ready: https://learn.microsoft.com/en-us/azure/backup/backup-azure-database-postgresql-flex-overview
- Validate current HA, region, backup and network support against Azure documentation and tenant policies again at implementation time; product availability changes.
