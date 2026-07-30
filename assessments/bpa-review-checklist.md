# BPA / Assessment Review Checklist

Working checklist for running a Best Practice Assessment or a general network/security health check engagement.

## Pre-engagement

- [ ] Confirm scope: single firewall, HA pair, or Panorama-managed fleet
- [ ] Request Tech Support File(s) (TSF) or arrange direct access to pull them
- [ ] Confirm PAN-OS/Panorama versions in scope (some checks are version-dependent)
- [ ] Set expectations with client on what BPA does and doesn't cover (config posture, not live traffic analysis)

## Running the BPA

- [ ] Upload TSF to the BPA tool
- [ ] Pull the Executive Summary: BPA system rating, CDSS capability adoption, Vulnerability Protection score
- [ ] Export full findings list (pass/fail per check, grouped by policies/objects/network/device)
- [ ] Cross-reference low-scoring categories against the [security baseline](../baselines/security-baseline.md)

## Findings triage

Group findings by effort vs. impact rather than presenting the raw fail list:

| Priority | Criteria |
|---|---|
| High | Security profile gaps (AV/anti-spyware/vuln protection not applied), no log forwarding, weak decryption settings |
| Medium | Zone protection thresholds not tuned, rule cleanup, object hygiene |
| Low | Cosmetic/administrative items (naming conventions, tag usage) |

## Deliverable structure

- Executive summary (posture score, top risks, trend if repeat assessment)
- Findings detail, mapped to remediation steps and PANW doc links
- Remediation roadmap (quick wins vs. longer-term projects)
- Recommendation for re-assessment cadence

## Notes

- BPA compares config against PANW best practices, not against the client's actual traffic or business requirements. Always sanity-check findings against what the environment actually needs before recommending a change.
- A low score isn't automatically a problem. Document any accepted risk / intentional deviation so it doesn't get flagged again next cycle.
