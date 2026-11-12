# SPX | security, identity and integration contract

## Zones and access

| Boundary | Required rule / safeguard | Validation |
|---|---|---|
| Technician -> SPX | Corporate managed device and internal network; HTTPS 443 to internal LB only; corporate DNS; TLS server certificate | Germany/France browser Kerberos test; no public listener |
| SPX -> AD | Private WAN to approved AD DNS and Kerberos KDCs (DNS 53; Kerberos 88 TCP/UDP; 464 for key rotation where needed); enumerate additional join/LDAP ports only if design requires them | Confirm real firewall matrix with AD admins; test AD outage and DR path |
| WMS -> SPX SFTP | Wuppertal DC source IPs to internal SFTP LB TCP 22; SSH public-key authentication, restricted upload account/chroot, per-environment credentials | Upload, atomic completion, host key change, duplicate and node-failure tests |
| SPX / importer -> PostgreSQL | Private DNS/routing to DB TCP 5432 with verified TLS; database roles split app, importer, migrations | No public database access; least-privilege tests |
| SPX -> DHL | Outbound HTTPS 443 via managed egress; verified DHL API FQDNs/IP policy and corporate proxy rules | Sandbox shipment and prod endpoint/allowlist test |
| SPX -> storage/secrets/telemetry | Private endpoints where supported; managed identities and narrowly scoped roles; approved EU destinations | DNS, firewall, secret-access and telemetry location tests |
| Administration | RBAC groups for eight DevOps members; least-privilege JIT/PIM, bastion/approved remote management; break-glass access audited | Access review and denied-access test |

No permanent credentials, keytabs, SSH private keys or DHL API secrets in Terraform state, source, pipeline variables or VM images. Store sensitive artifacts in an approved EU Key Vault, use managed identity for retrieval, prevent secret output in plan/logs, secure remote state and separate state by environment. Keytabs are security-sensitive: agree AD-issued SPN HTTP/<private-SPX-FQDN>, service account scope and rotation, provision the keytab to each RHEL VM with restrictive permissions, and ensure synchronized clocks. SPNEGO must be tested with Windows integrated-auth browser policy. Authorize via explicit group-to-role mapping; document group synchronization and account disable latency. Kerberos uses the client ticket without collecting Windows passwords.

For SFTP: use separate upload-only service accounts, restrict source network, disable password/interactive login, enforce host-key trust and cryptographic policy, antivirus/content inspection as required, filename/size quotas, atomic rename convention, checksum and business date validation. Protect raw, processed and quarantine objects with RBAC, lifecycle rules and encryption; agree retention and reprocessing window. A file is not successful until the full payload is durably stored and database import committed; reconcile any upload/import split-brain.

For DHL: agree API version, production and test endpoints, required shipping/customer personal data, request/response schema, quotas, IP allowlisting, authentication mode, data-processing terms, rate limits and webhook behavior. Handle timeout ambiguity using a stable client order key and shipment lookup; redact addresses, customer identifiers, tokens and Kerberos tickets from logs. Keep only necessary shipping fields with approved retention.

Threat-model the internal entry points (malicious file, credential theft, lateral movement, Kerberos replay, duplicate shipment), require RHEL/Java/PostgreSQL patching, VM disk and backup encryption, audit logs and vulnerability scans. Confirm regulatory classification with the privacy/security team; EU placement of Azure data alone does not prove DHL handling meets the data-residency requirement.
