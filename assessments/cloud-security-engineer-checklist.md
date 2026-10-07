# Cloud Security Engineer Checklist

Practical review checklist for cloud environments operated by security and platform engineering teams. Covers AWS and Azure, with a crosswalk to the CIS Foundations Benchmarks, NIST Cybersecurity Framework (CSF) 2.0, and Cloud Security Alliance (CSA) Cloud Controls Matrix (CCM).

This is an original working checklist, not an audit program or a substitute for the current official benchmarks. It does not reproduce benchmark control text or establish compliance. Confirm the applicable framework versions, scope, and evidence requirements with the organization before assessing. Cloud services and provider recommendations change; verify implementation details in current vendor documentation.

## Assessment context

- [ ] Record assessment date, assessor, business/service owner, cloud provider(s), accounts or subscriptions, regions, and environments in scope.
- [ ] Identify workloads, data classifications, internet-facing services, critical dependencies, and regulatory or contractual scope.
- [ ] Confirm the applicable CIS Benchmark and level, provider-native security baseline, NIST CSF 2.0 profile, and any CSA CCM or regulatory mappings.
- [ ] Document shared-responsibility boundaries for each managed service; assign controls to the cloud provider, platform team, workload owner, or another party.
- [ ] Identify approved exceptions, compensating controls, risk owners, and expiry/review dates.
- [ ] Collect evidence from live configuration and monitoring, not policy documents alone.

## Governance and inventory

- [ ] Cloud accounts/subscriptions are inventoried, assigned an owner and environment, and governed through an organization or management-group hierarchy.
- [ ] New account/subscription creation is controlled and applies required identity, logging, network, and security baselines automatically.
- [ ] Resource inventory is discoverable across all in-scope regions and subscriptions; orphaned, unsupported, and unowned resources are reviewed.
- [ ] Security policies are defined as code or centrally managed guardrails where practical; drift is detected and exceptions are tracked.
- [ ] Cloud assets and data have owners, classification, criticality, and lifecycle/retention requirements.
- [ ] Cost, quota, and service limits that could affect security controls or incident response are monitored.

## Identity and privileged access

- [ ] Human access uses centralized federation/SSO and phishing-resistant MFA for privileged roles where supported.
- [ ] Root/global administrator identities are protected, monitored, and used only for documented break-glass needs.
- [ ] Privileged access is time-bound or just-in-time, approved, and logged; standing administrator assignments are minimized and reviewed.
- [ ] Workloads use managed identities or short-lived federated credentials instead of embedded, long-lived secrets.
- [ ] Roles follow least privilege, separate duties where necessary, and avoid wildcard permissions or broad cross-account/tenant trust.
- [ ] Access reviews cover users, groups, service principals, roles, access keys, and external identities on a defined cadence.
- [ ] Emergency accounts are limited, strongly protected, tested, and generate high-priority alerts when used.

## Network security

- [ ] Public exposure is intentional, documented, owned, and limited to required protocols and sources.
- [ ] Administrative interfaces are not exposed directly to the internet; use approved private access paths and strong authentication.
- [ ] Network segmentation separates production, non-production, management, and sensitive workloads; east-west paths are understood.
- [ ] Firewall, security-group, NSG, and route rules are reviewed for broad access, stale entries, shadowing, and unintended bypass paths.
- [ ] Private connectivity is used for sensitive service-to-service traffic where appropriate; DNS and routing behavior are included in the design.
- [ ] Network flow and perimeter logs are collected, retained, and queryable for relevant assets.
- [ ] DDoS protections and public-ingress protections match service criticality and exposure.
- [ ] Egress paths are controlled or monitored, and unexpected internet egress can be detected.

## Data protection and key management

- [ ] Data stores are inventoried and classified; public access is disabled unless explicitly approved.
- [ ] Encryption in transit is enforced for sensitive service and administrative connections.
- [ ] Encryption at rest is enabled for sensitive data using provider-managed or customer-managed keys as required by policy.
- [ ] Key ownership, rotation, access, recovery, and revocation procedures are documented and tested.
- [ ] Secrets are kept in an approved secrets manager, access is scoped, and rotation is automated where supported.
- [ ] Backups and snapshots inherit appropriate encryption and access restrictions; restore access is separated from production administration.
- [ ] Retention, deletion, and data residency requirements are implemented for logs, backups, and application data.
- [ ] Public sharing, cross-account/tenant access, and external data transfers are monitored and periodically reviewed.

## Logging, monitoring, and threat detection

- [ ] Control-plane and administrative activity is logged across all in-scope accounts/subscriptions and regions.
- [ ] Logs are centralized into a protected destination with restricted delete/modify access and retention aligned to incident and compliance needs.
- [ ] Identity, network, DNS, workload, data-access, and security-service events are available to the security monitoring function.
- [ ] High-risk detections cover privilege escalation, unusual authentication, public exposure, disabling security controls, suspicious data access, and unexpected key use.
- [ ] Alert ownership, severity, escalation paths, and response expectations are documented and exercised.
- [ ] Security posture findings are assigned to owners, prioritized by exploitability and impact, and tracked to closure or an approved exception.
- [ ] Time synchronization and resource identity allow events to be correlated across cloud services and workloads.

## Workloads, containers, and software delivery

- [ ] VM images, containers, serverless functions, and managed services use supported versions and hardened configurations.
- [ ] Images and artifacts come from trusted registries and are scanned for vulnerabilities, malware, and exposed secrets before deployment.
- [ ] Workload runtime permissions are distinct from human administrative permissions and scoped to required resources.
- [ ] Kubernetes/API endpoints are private or strongly restricted; cluster RBAC, workload identity, admission controls, and audit logs are reviewed where applicable.
- [ ] CI/CD identities use short-lived federation, least privilege, and protected environments; deployment secrets are not stored in source or logs.
- [ ] Infrastructure-as-code is reviewed, scanned, and deployed through controlled pipelines; production changes are traceable and reversible.
- [ ] Critical vulnerabilities have documented remediation SLAs based on risk, with compensating controls for items that cannot be patched promptly.
- [ ] Endpoint, runtime, and host protections are enabled where the workload model requires them and report to a monitored security service.

## Vulnerability and configuration management

- [ ] Provider-native posture and threat-detection services are enabled at the organization/tenant level and cover in-scope assets.
- [ ] CIS Benchmark assessments use the current applicable version and level; failed or not-applicable checks have evidence and rationale.
- [ ] Provider-native benchmark findings are reconciled with the organization's policy rather than accepted or suppressed without review.
- [ ] Asset owners receive actionable findings with severity, remediation guidance, due date, and escalation path.
- [ ] Configuration changes are attributable to a person or workload identity and reviewed for security impact.
- [ ] Baseline changes are tested before broad rollout, and security policy changes have rollback or recovery procedures.

## Incident response and resilience

- [ ] Cloud-specific incident procedures identify who can isolate workloads, revoke credentials, disable keys, preserve evidence, and contact the provider.
- [ ] Incident responders can access required logs and snapshots without relying on potentially compromised production identities.
- [ ] Playbooks cover exposed credentials, public data exposure, compromised workload, destructive activity, and provider-control-plane incidents.
- [ ] Backup and recovery objectives are defined for critical services and tested; backup deletion is protected from ordinary workload administrators.
- [ ] Critical services are designed for availability-zone/region failures appropriate to business requirements.
- [ ] Incident exercises validate cloud audit-log completeness, notification, containment authority, evidence preservation, and recovery.

## AWS-specific checks

- [ ] AWS Organizations and account structure provide centralized governance; organization-wide security services and delegated administration are configured intentionally.
- [ ] CloudTrail management events are enabled across in-scope regions and accounts, with logs protected in a centralized destination.
- [ ] AWS Config recording and Security Hub CSPM coverage are enabled for in-scope resources and regions; findings have an operational owner.
- [ ] IAM root credentials are protected; MFA is enabled; unused credentials and excessive access are detected and removed.
- [ ] IAM Access Analyzer or equivalent external-access analysis is enabled for supported resource types.
- [ ] S3 Block Public Access is enabled at the organization/account level where appropriate; bucket policies and access points are reviewed for unintended exposure.
- [ ] VPC Flow Logs are enabled for relevant networks, and default security groups do not permit unintended traffic.
- [ ] EC2 instances use IMDSv2 where supported; public IPs and remote-administration ingress are restricted.
- [ ] EBS, RDS, and other sensitive stores are encrypted; KMS key policies and grants are reviewed for least privilege.
- [ ] AWS Backup or equivalent protection covers critical data, with cross-account or logically isolated copies where required.

## Azure-specific checks

- [ ] Management groups and subscriptions reflect governance boundaries; Azure Policy assignments and exemptions are reviewed at the appropriate scope.
- [ ] Microsoft Entra ID privileged roles use least privilege, strong authentication, and Privileged Identity Management or equivalent just-in-time controls.
- [ ] Defender for Cloud or equivalent posture and workload protections are enabled for the services in scope; recommendations are assigned and tracked.
- [ ] Azure Activity Logs and resource diagnostic logs are routed to a protected central workspace or SIEM with appropriate retention.
- [ ] Network Security Groups and Azure Firewall rules are reviewed for public administration access, broad source/destination ranges, and unintended routes.
- [ ] Private endpoints and private DNS are used where required for sensitive platform services; public network access is disabled when unnecessary.
- [ ] Storage accounts restrict public access and anonymous access; network access, shared-key usage, and SAS-token lifetimes are controlled.
- [ ] Key Vault access uses managed identities and least privilege; soft delete and purge protection are enabled for production vaults.
- [ ] Managed identities are preferred over client secrets for Azure workloads; credentials and application registrations are reviewed.
- [ ] Azure Backup or equivalent protects critical workloads, and restore procedures are tested.

## Standards alignment guide

Use the official documents for exact control statements, scope, and audit evidence. The checklist items above are grouped by practical engineering activity and are not a one-to-one reproduction of standard controls.

| Framework | How to use it in this review |
|---|---|
| CIS AWS Foundations Benchmark and CIS Microsoft Azure Foundations Benchmark | Select the applicable current version and level for core provider configuration. Add service-specific CIS Benchmarks when compute, database, or storage controls are in scope. |
| AWS Foundational Security Best Practices and Microsoft Cloud Security Benchmark | Use provider-native recommendations and continuous posture findings to identify cloud-service-specific configuration gaps. Review exceptions and coverage. |
| NIST CSF 2.0 | Use Govern, Identify, Protect, Detect, Respond, and Recover to organize risk ownership, asset coverage, preventive controls, detection, incident handling, and resilience. |
| CSA Cloud Controls Matrix (CCM) | Use cloud-specific control domains and responsibility assignments for broader cloud assurance and cross-framework mapping. |

### NIST CSF 2.0 review coverage

| CSF function | Checklist areas |
|---|---|
| Govern | Assessment context, policy ownership, risk exceptions, shared responsibility, and change governance |
| Identify | Governance and inventory, asset ownership, data classification, criticality, and dependency mapping |
| Protect | Identity, network, data protection, workload security, and configuration management |
| Detect | Logging, monitoring, threat detection, and security posture findings |
| Respond | Incident roles, containment capabilities, playbooks, and exercises |
| Recover | Backups, restoration tests, resilience objectives, and recovery exercises |

## Findings and evidence record

For each failed, partial, or not-applicable check, record:

| Field | Record |
|---|---|
| Asset / scope | Account, subscription, region, service, and resource identifiers |
| Evidence | Configuration export, policy evaluation, log query, screenshot, or test result with collection date |
| Standard reference | Framework, version, and control ID when applicable |
| Finding | Observed condition and affected data/service |
| Risk and priority | Likelihood, impact, exposure, exploitability, and severity |
| Remediation | Owner, action, target date, and validation method |
| Exception | Approver, rationale, compensating control, expiry, and review date |

Prioritize exposed credentials, public sensitive data, internet-accessible management planes, broad privileged access, disabled audit controls, and exploitable critical vulnerabilities before lower-impact hygiene findings.

## References

- [CIS Amazon Web Services Benchmarks](https://www.cisecurity.org/benchmark/amazon_web_services)
- [CIS Microsoft Azure Benchmarks](https://www.cisecurity.org/benchmark/azure)
- [AWS Foundational Security Best Practices in Security Hub CSPM](https://docs.aws.amazon.com/securityhub/latest/userguide/fsbp-standard.html)
- [CIS AWS Foundations Benchmark in Security Hub CSPM](https://docs.aws.amazon.com/securityhub/latest/userguide/cis-aws-foundations-benchmark.html)
- [Microsoft Cloud Security Benchmark overview](https://learn.microsoft.com/en-us/security/benchmark/azure/overview)
- [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework)
- [CSA Cloud Controls Matrix](https://cloudsecurityalliance.org/research/cloud-controls-matrix)
