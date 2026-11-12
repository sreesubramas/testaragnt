# SPX | Terraform, pipelines and handover plan

This package describes the build; it does not deploy Azure resources or provide credentials. Eight DevOps engineers own build/run; assign named IaC/code reviewers and segregate PRD deployment approvers.

## Terraform implementation units

- Platform prerequisites (coordinate with landing-zone team): DEV/QAS/PRD subscription/RG strategy, EU region allow-policy, network address plan, redundant on-prem WAN, DNS and AD routing, diagnostic/monitoring destinations, encrypted remote state with locking and limited access, tagging and budget alerts.
- Per-environment modules: private VNet/subnets/NSGs/routes, internal app and SFTP load balancers, RHEL9 app VMs and SFTP ingress tier in multiple PRD zones, managed identities, Key Vault and private storage, private PostgreSQL Flexible Server with PRD HA, EU Backup vault and retention policy, managed egress, log/metric/synthetic alert rules. Secondary-region PRD resources, PostgreSQL replica and DNS cutover controls are separate explicit units; do not use uncontrolled auto-promotion.
- Environment config and state: identical modules with separate state, variables, identity, secret scopes and approval protections; use immutable image/application artifacts and pinned providers. Terraform state can contain sensitive metadata: encrypt, restrict, audit and never commit it. Avoid embedding secret values/keytabs in HCL or state.
- Parametrization: Azure subscription IDs, region decision, DNS/FQDN/SPN, subnet CIDRs, WMS source ranges/file path, DHL FQDNs/credential reference, VM/DB SKU from load test, DB name/version, backup policy and SLA alert thresholds. Unknowns remain inputs, not guessed values.

## Azure DevOps gated release

1. Pull request: Java 17 build/unit/integration tests, SAST/dependency/license check, IaC validate/format/security policy, peer review, signed/pinned artifact and SBOM if mandated.
2. DEV: Terraform plan reviewed then apply using environment-scoped service connection; deploy app/DB migrations; smoke-test Kerberos (test AD), mock SFTP sender and DHL sandbox. No production secrets/data.
3. QAS: representative 400-user/concurrency model, load test, two-zone failure, 50 GB database restore, malformed and repeated file, outage of AD/WMS/DHL, region failover, RPO/RTO timer and security tests; remediate unmet targets.
4. PRD: explicit separate approver, reviewed plan, Sunday after 22:00 CET/CEST if agreed, app rollout with expand/contract schema changes, synthetic health gate, confirm file import/order workflow; page/rollback criteria predefined. Protect service connections and secure files; avoid logging keytabs.
5. Promote the same immutable build through all environments; record release ID, source commit, applied Terraform plan, DB migration version, test evidence and rollback owner. Maintain a manual promotion/failover procedure because DB promotion is not equivalent to an application rollback.

## Handover checklist

- [ ] Owner accepts workload intake, data classification, Germany/France and EU-region scope.
- [ ] AD owner confirms SPN, keytab rotation and DR Kerberos connectivity; test on real technician workstation.
- [ ] Network owner confirms private Wuppertal SFTP path, AD DNS/KDC access, DHL HTTPS egress and two-region DNS cutover.
- [ ] Database owner approves managed-service control boundary, major version, 50 GB sizing, backup, replica and migration.
- [ ] Security signs off on least privilege, patching, privacy/DHL handling, retention, vault policy, threat model and EU telemetry.
- [ ] Business defines 99.95% measurement and maintenance treatment; agrees RPO/RTO failure scope.
- [ ] QAS evidence proves zone failover, regional recovery, replay/idempotency, <=60-min recoverable transaction/file point and <=4-hour restored user journey; otherwise log explicit exception.
- [ ] Monthly backup points retained >=6 months are demonstrated; until aged history exists, policy and monthly job evidence are tracked for deferred sign-off.
- [ ] Named on-call, backup escalation, dashboards, change calendar and alert drills handed to eight-person team.
- [ ] Cost estimate, SKU/region capabilities, subscription IDs and landing-zone policy signed off before provisioning.
