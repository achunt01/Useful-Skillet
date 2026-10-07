# AWS CIS Foundations Assessment Checklist

Practical working checklist for assessing foundational AWS account and service configuration against the CIS AWS Foundations Benchmark. It focuses on cloud security engineering evidence across identity, audit, network, and data controls.

This checklist is not the CIS Benchmark, does not reproduce its control text, and is not a compliance attestation. Select the exact benchmark version and Level (1 or 2) required for the assessment, and use the official benchmark for authoritative control language, applicability, procedures, and evidence. Example CIS control identifiers below are provided as navigation aids and may change between versions.

## Assessment setup

- [ ] Record assessment date, assessor, organization, account IDs, partitions, regions, and environments in scope.
- [ ] Select and record the CIS AWS Foundations Benchmark version and Level for each in-scope account.
- [ ] Identify exclusions, service availability differences, approved exceptions, and compensating controls before scoring results.
- [ ] Confirm whether the assessment includes only foundational AWS controls or also CIS service benchmarks for compute, databases, storage, containers, or operating systems.
- [ ] Gather read-only evidence with collection dates; redact secrets and sensitive customer data.
- [ ] Identify a technical owner and risk owner for each account and finding.

## Account and organization governance

- [ ] Account security contact information is current and routes to an actively monitored mailbox or response team. Example reference: `Account.1`.
- [ ] Accounts belong to the intended AWS Organization and organizational unit; production and security boundaries are reflected in account structure.
- [ ] Organization-wide guardrails prevent disabling required audit and security services or weakening mandatory controls without an approved exception.
- [ ] Security services have documented delegated administrators, coverage, and ownership across in-scope accounts and regions.
- [ ] Unused, suspended, or sandbox accounts are identified and have an appropriate access, data-retention, and closure process.

## Identity and access management

- [ ] Root-user credentials are protected, not used for routine work, and monitored; root access keys do not exist. Example reference: `IAM.4`.
- [ ] Root MFA is enabled and meets the selected benchmark's required assurance level. Example references: `IAM.6`, `IAM.9`.
- [ ] IAM users with console passwords have MFA where the benchmark applies. Prefer federated workforce access over creating long-lived IAM users. Example reference: `IAM.5`.
- [ ] Human and workload permissions follow least privilege; broad administrative policies and wildcard actions/resources are identified, justified, and minimized.
- [ ] IAM users do not have directly attached policies where prohibited by the selected benchmark. Example reference: `IAM.2`.
- [ ] Long-lived access keys are inventoried, rotated or removed under policy, and replaced with federation or short-lived credentials where possible. Example references: `IAM.3`, `IAM.22`.
- [ ] Unused credentials and identities are disabled or removed using the exact inactivity criteria in the selected benchmark.
- [ ] Password policy controls meet the selected benchmark when IAM users with passwords are permitted. Example references: `IAM.15`, `IAM.16`.
- [ ] External access to supported resources is reviewed with IAM Access Analyzer; findings have an owner and disposition. Example reference: `IAM.28`.
- [ ] AWS CloudShell permissions are reviewed and restricted according to organizational policy. Example reference: `IAM.27`.
- [ ] A support role and incident-access process are available so responders can engage AWS Support when needed. Example reference: `IAM.18`.
- [ ] IAM policies, role trust policies, permission boundaries, and cross-account access are reviewed for unintended privilege escalation paths.

## Audit logging and configuration history

- [ ] At least one multi-Region CloudTrail trail captures management events in all in-scope accounts and regions, including newly enabled regions. Example reference: `CloudTrail.1`.
- [ ] Trail coverage, event selectors, delivery status, and organization/account membership are monitored for gaps or unauthorized changes.
- [ ] CloudTrail log files are encrypted at rest using the configuration required by the selected benchmark. Example reference: `CloudTrail.2`.
- [ ] CloudTrail log file integrity validation is enabled where required. Example reference: `CloudTrail.4`.
- [ ] Access logging is enabled for the S3 bucket that stores CloudTrail logs where required. Example reference: `CloudTrail.7`.
- [ ] The CloudTrail log destination blocks unintended public access, restricts write/delete permissions, and retains logs for the defined period.
- [ ] AWS Config records the required resource types in all in-scope regions and uses the appropriate service-linked role. Example reference: `Config.1`.
- [ ] CloudTrail, AWS Config, and security-service changes produce alerts and are investigated.

## Network and compute configuration

- [ ] Default VPC security groups do not allow unintended inbound or outbound traffic. Example reference: `EC2.2`.
- [ ] VPC Flow Logs are enabled for all in-scope VPCs and delivered to a protected, monitored destination. Example reference: `EC2.6`.
- [ ] EBS default encryption is enabled in each applicable region. Example reference: `EC2.7`.
- [ ] EC2 instances require IMDSv2 where supported and compatible with the workload. Example reference: `EC2.8`.
- [ ] Network ACLs do not expose SSH or RDP to the internet. Example reference: `EC2.21`.
- [ ] Security groups do not expose remote-administration ports to `0.0.0.0/0` or `::/0` unless explicitly justified and mitigated. Example references: `EC2.53`, `EC2.54`.
- [ ] Public IPv4/IPv6 assignment is intentional; management access uses approved private paths and is logged.
- [ ] Security group rules are reviewed for overly broad CIDRs, unrestricted egress, stale rules, and references to unexpected security groups.
- [ ] Network routing and inspection paths are reviewed to verify that sensitive traffic cannot bypass required security controls.

## Storage, databases, and encryption

- [ ] S3 Block Public Access is configured at account and bucket levels consistent with the intended access model. Example references: `S3.1`, `S3.8`.
- [ ] S3 bucket policies require secure transport where applicable. Example reference: `S3.5`.
- [ ] S3 access policies, access points, ACLs, and cross-account sharing are reviewed for unintended public or external access.
- [ ] MFA Delete and object-level CloudTrail data events are enabled where required by the selected benchmark and justified by data criticality. Example references: `S3.20`, `S3.22`, `S3.23`.
- [ ] KMS key policies and grants are least-privilege; key rotation is enabled where required and operationally supported. Example reference: `KMS.4`.
- [ ] EFS encryption requirements are met for the selected benchmark and workload; review applicable controls such as `EFS.1` and `EFS.8`.
- [ ] RDS instances and clusters are not publicly accessible unless explicitly approved and protected. Example reference: `RDS.2`.
- [ ] RDS encryption at rest, Multi-AZ resilience, and automatic minor version upgrade settings meet applicable benchmark requirements. Example references: `RDS.3`, `RDS.5`, `RDS.13`, `RDS.15`.
- [ ] Backup and snapshot access, encryption, retention, and recovery testing align with data classification and recovery objectives.

## Continuous assessment and remediation

- [ ] The selected CIS benchmark is evaluated continuously or on a defined assessment cadence using an approved method.
- [ ] If AWS Security Hub CSPM is used, confirm the selected CIS standard version, account/region coverage, enabled controls, and finding workflow; do not assume it checks every benchmark requirement.
- [ ] Reconcile automated findings against the official benchmark and validate controls requiring manual evidence or organizational context.
- [ ] Findings record the account, region, resource, benchmark version and Level, CIS control identifier, evidence, severity, owner, and remediation date.
- [ ] Not-applicable controls include a rationale and supporting evidence; exceptions include an approver, compensating control, and expiry/review date.
- [ ] High-risk findings are prioritized based on exposure, data sensitivity, exploitability, and business impact, not score alone.
- [ ] Remediation is verified after changes, and recurring failures are addressed through account vending, policy-as-code, or other preventive guardrails.

## Assessment results

| CIS control / topic | Account and region | Result (Pass / Fail / N/A) | Evidence reference | Finding / exception ID | Owner and due date |
|---|---|---|---|---|---|
|  |  |  |  |  |  |
|  |  |  |  |  |  |
|  |  |  |  |  |  |

For each failure or exception, retain the exact evidence and assessment steps required by the chosen CIS Benchmark. Do not record credentials, secret values, or unnecessary personal information in the report.

## References

- [CIS Amazon Web Services Benchmarks](https://www.cisecurity.org/benchmark/amazon_web_services)
- [CIS AWS Foundations Benchmark in AWS Security Hub CSPM](https://docs.aws.amazon.com/securityhub/latest/userguide/cis-aws-foundations-benchmark.html)
- [AWS Foundational Security Best Practices in Security Hub CSPM](https://docs.aws.amazon.com/securityhub/latest/userguide/fsbp-standard.html)
- [AWS Organizations user guide](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_introduction.html)
- [AWS CloudTrail user guide](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-user-guide.html)
- [AWS Config developer guide](https://docs.aws.amazon.com/config/latest/developerguide/WhatIsConfig.html)
