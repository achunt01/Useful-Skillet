# Prisma Access Assessment Checklist

Working checklist for assessing a Prisma Access tenant on a client engagement. Companion to the architecture reference: [architecture/prisma-access.md](../architecture/prisma-access.md).

## Pre-engagement

- [ ] Confirm the management plane: Strata Cloud Manager or Panorama-managed (procedures and feature availability differ)
- [ ] Confirm edition and licensing meter: per-user, per-site, or per-bandwidth; Local vs Worldwide locations
- [ ] Confirm who holds Superuser on the managing plane and arrange tenant access or config export
- [ ] Inventory connection types in scope: mobile users, remote networks, service connections, Clean Pipe tenants

## Connectivity and routing

- [ ] At least one service connection exists if mobile users or remote networks need private-app access
- [ ] Service connections advertise specific internal prefixes, not a default route, unless backhaul is intentional
- [ ] BGP or static routing documented per connection; no unexplained overlapping RFC1918 space
- [ ] Remote network redundancy: secondary location in a separate compute location where the business requires it
- [ ] ECMP route summarization consistent across all links to a location (all-or-nothing per location)

## Policy posture

- [ ] Security Profile Groups applied on allow rules (no uninspected allows)
- [ ] No allow rules with application any + service any (App-ID bypass)
- [ ] Decryption policy covers internet-bound traffic; exclusions reviewed and current
- [ ] URL Filtering, Threat Prevention, DNS Security, WildFire enforced where the edition includes them
- [ ] ZTNA app definitions scoped to specific apps, not broad network ranges

## Mobile users

- [ ] Agent choice documented: GlobalProtect vs Prisma Access Agent, and why
- [ ] HIP checks enforced (disk encryption, AV, patch level), matching on-prem posture
- [ ] Split-tunnel vs full-tunnel decision documented and consistent across gateways
- [ ] Clientless VPN and Explicit Proxy usage inventoried; Explicit Proxy treated as a migration phase with a target end date

## Capacity and licensing

- [ ] Bandwidth allocation per compute location vs actual peak usage
- [ ] Remote network onboarding model confirmed: aggregate bandwidth vs site-based (6.0+)
- [ ] Data transfer allowance checked against actual egress
- [ ] SaaS and partner source-IP allowlists include active and reserved Prisma Access IPs for every location in scope

## Logging and visibility

- [ ] Logs forwarding to Strata Logging Service; retention meets client policy
- [ ] ADEM or Insights reviewed for chronic user-experience issues before blaming policy

## Findings triage

| Priority | Criteria |
|---|---|
| High | Uninspected allow rules, missing service connection breaking private-app access, decryption gaps on internet traffic |
| Medium | Bandwidth under-provisioned per compute location, stale IP allowlists, HIP gaps vs on-prem |
| Low | Naming, tagging, unused locations |

## Deliverable structure

- Executive summary (posture, top risks)
- Findings mapped to remediation steps and PANW doc links
- Remediation roadmap (quick wins vs longer-term)
- Re-assessment cadence recommendation

## Notes

- A Prisma Access assessment is a config and architecture review, not a live traffic analysis. Pair it with log review (Traffic, Threat, URL logs in Strata Logging Service) before declaring policy effective.
- Edition determines which findings are actionable. Do not recommend ZTNA hardening on an edition that does not include ZTNA; scope the findings to what the client bought.
