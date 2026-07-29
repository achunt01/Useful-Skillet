# Palo Alto Security Baseline Reference

Framework sources: Iron Skillet, Best Practice Assessment (BPA), CIS Benchmarks, PAN-OS Best Practices docs.

Working checklist for baseline engagements, not a copy-paste config. Every environment still needs its own zone model, rule base, and risk tolerance layered on top of this.

## Framework sources

| Source | What it gives you |
|---|---|
| **Iron Skillet** | Day-one XML config templates ([PaloAltoNetworks/iron-skillet](https://github.com/PaloAltoNetworks/iron-skillet)) that pre-load recommended security profiles, logging, and decryption settings before any use-case-specific policy is added. Deployable via Panorama, panhandler, or loaded directly on the firewall. |
| **Best Practice Assessment (BPA)** | PANW's own assessment tool. Parses a Tech Support File and runs 200+ pass/fail checks against current config, mapped to a capability adoption heatmap (App-ID, User-ID, Threat Prevention, URL Filtering, WildFire, logging). Used both to baseline a new build and to audit an existing one. |
| **CIS Benchmarks for PAN-OS** | Vendor-neutral, consensus-driven hardening checklist covering device management, zones, security policy, decryption, logging, and HA. Useful when a client needs to map to a recognized compliance standard rather than just PANW's own recommendations. |
| **PAN-OS Best Practices docs** | [docs.paloaltonetworks.com/best-practices](https://docs.paloaltonetworks.com/best-practices) — living best-practice guides per feature area: DoS/zone protection, decryption, security profiles, App-ID migration, etc. Where the reasoning and deployment sequencing for each control lives. |

## Baseline configuration categories

### Device management & hardening
- Management interface restricted to a dedicated network/jump host, no management access from untrust
- Idle timeout, login banner, and account lockout thresholds set on admin accounts
- Local admin accounts backed by MFA or tied to an external auth profile (RADIUS/SAML/LDAP) rather than local-only
- NTP, DNS, and update servers configured; scheduled dynamic content updates (App-ID, Threat, WildFire, Antivirus)
- SNMP/syslog service routes locked down to specific destinations, not left on default

### Zones and segmentation
- Purpose-built zones instead of one flat trust/untrust pair; segment by function (user, server, DMZ, guest, OT, etc.)
- Interzone default deny confirmed and not silently overridden by a broad allow rule
- Zone Protection profiles applied to every ingress zone, particularly internet-facing ones

### Zone protection & DoS
Measure-then-tune, not set-and-forget.
- Baseline average/peak connections-per-second (CPS) per zone before setting flood thresholds
- Flood protection enabled for TCP SYN, UDP, ICMP, ICMPv6, and other IP protocols, with Alarm/Activate/Maximum thresholds tuned to the measured baseline
- Reconnaissance protection enabled (port scan, host sweep) with block-ip action
- Packet Buffer Protection enabled to protect firewall resources under load

### Security policy
- App-ID used in place of port-based rules wherever feasible
- User-ID integrated for identity-based policy, not just IP-based
- Rulebase reviewed for shadowed/unused rules, overly broad "any/any" entries, and missing logging
- Log forwarding enabled on every rule (start and/or end, per PANW guidance) rather than left at defaults

### Security profiles
Where the BPA checks most heavily, and where Iron Skillet's recommended profiles are the fastest starting point.
- Antivirus, Anti-Spyware, and Vulnerability Protection profiles applied to all allowed traffic, with low/informational severity actions set to "default" rather than disabled
- URL Filtering profile blocking known-malicious and liability-risk categories, with credential submission control enabled
- WildFire profile forwarding unknown files for analysis, actions set per PANW recommended defaults
- File Blocking profile covering high-risk file types (executables, encrypted archives, etc.)

### Decryption
- SSL Forward Proxy decryption profile blocking sessions with expired/untrusted certs and unsupported protocol versions
- Minimum TLS version set to 1.2, weak ciphers (3DES, RC4, MD5) disabled
- Decryption exclusions documented and kept to the minimum needed (e.g., regulated traffic categories)

### Logging & visibility
- All log types (Traffic, Threat, System, Config, User-ID/HIP Match) forwarded to Panorama or an external log collector/SIEM, not left local-only
- Critical system events forwarded to email/alerting in addition to syslog
- Log retention aligned to the client's compliance or incident-response requirements

### Remote access (GlobalProtect)
- Gateway/portal certificates valid and not self-signed in production
- HIP checks enforced where endpoint posture matters (AV running, disk encryption, patch level)
- Split-tunnel vs. full-tunnel decision documented and matched to the client's risk posture

### Panorama structure
- Device groups and templates structured to avoid duplicated policy across firewalls
- Shared vs. device-group-specific objects kept clean to avoid naming collisions and rule sprawl
- Template stacks used to layer common baseline settings under site-specific overrides

### High availability
- HA1/HA2 links on dedicated interfaces, not sharing data-plane interfaces
- Path monitoring and link monitoring configured so failover actually triggers on real outage conditions
- Config sync and session sync verified, not just assumed from HA state

## Implementation approach

1. Load Iron Skillet day-one templates first on new/greenfield deployments to establish recommended profiles, logging, and decryption settings before any use-case policy exists.
2. Run a BPA against existing environments to get an objective pass/fail gap list against current config, rather than relying on a manual audit alone.
3. Cross-reference BPA findings against the CIS Benchmark when the client needs to map to a named compliance standard.
4. Prioritize remediation using the BPA heatmap and PANW best-practice docs for deployment sequencing (e.g., measure CPS before setting zone protection thresholds, don't just copy default values).
5. Document the resulting baseline as the client's own standard, including any accepted deviations and the reasoning behind them.

## Ongoing maintenance

- Re-run BPA on a recurring cadence (quarterly is common) to catch configuration drift
- Review zone protection thresholds after major traffic pattern changes, not just at initial deployment
- Track PAN-OS best-practice doc updates, since profile recommendations shift between releases

## References

- Iron Skillet: [github.com/PaloAltoNetworks/iron-skillet](https://github.com/PaloAltoNetworks/iron-skillet)
- Best Practice Assessment overview: [paloaltonetworks.com/services/bpa](https://www.paloaltonetworks.com/services/bpa)
- PAN-OS Best Practices: [docs.paloaltonetworks.com/best-practices](https://docs.paloaltonetworks.com/best-practices)
- CIS Benchmarks for Palo Alto Networks: [cisecurity.org/benchmark/palo_alto_networks](https://www.cisecurity.org/benchmark/palo_alto_networks)
