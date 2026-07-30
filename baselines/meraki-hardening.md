# Meraki Hardening Baseline

Meraki is cloud-managed, so the hardening surface splits into two parts: the Dashboard (org/account security) and the devices themselves. The Dashboard half is the one most often skipped.

## Dashboard and organization security

- [ ] Two-factor authentication enforced org-wide for all admins
- [ ] Password policy configured (complexity, periodic change, reuse restrictions)
- [ ] Role-based administration — scope admins to networks/tags they actually need instead of full org access
- [ ] Review the admin list on a schedule. Departed staff and stale partner accounts accumulate here.
- [ ] **Meraki Support access set to BLOCKED** by default. Enable it only for the duration of an active support case, then re-block.
- [ ] Login IP ranges restricted where the client's access pattern allows it
- [ ] Configuration change alerts enabled and going somewhere a human reads
- [ ] Review the change log after any maintenance window

## Local Status Page

Worth calling out separately because the default is weak: LSP credentials default to the device **serial number as username with an empty password**, with no login attempt limiting. That's low enough entropy to brute-force. If reached, an attacker can modify config, cause a DoS, or pull low-privileged info.

- [ ] Disable the Local Status Page where it isn't operationally needed, or
- [ ] Set explicit LSP credentials and restrict management VLAN access to it

## Network design and segmentation

- [ ] Treat management access as its own trust zone, not part of the general user VLAN
- [ ] VLANs mapped to actual functional zones before touching a single firewall rule. Firewall projects derail when the first step is "log into Dashboard and add a rule."
- [ ] Guest networks: internet-only, no access to internal DNS or file shares
- [ ] Auto VPN scope limited to subnets that actually need site-to-site reachability. VPNs connect sites; they don't need to expose every subnet everywhere.
- [ ] Templates used for consistency across sites, with local overrides documented

## MX security appliance

- [ ] L3 and L7 firewall rules built from the zone/flow design, least-privilege from the start
- [ ] IDS/IPS enabled with an appropriate ruleset
- [ ] AMP (anti-malware) enabled
- [ ] Content filtering configured
- [ ] Geo-IP firewalling where the business case supports it
- [ ] SD-WAN / multi-uplink policies configured per traffic type rather than left at defaults, so performance-sensitive traffic fails over appropriately

## Wireless (MR)

- [ ] WPA2-Enterprise or WPA3 where client support allows; no PSK on corporate SSIDs
- [ ] Guest SSID isolated with client isolation enabled
- [ ] Rogue AP / air marshal alerting reviewed, not just enabled

## Switching (MS)

- [ ] STP bridge priority set intentionally based on topology, not left to default election
- [ ] Unused ports disabled or dropped into an unrouted VLAN
- [ ] Port security / access policies on user-facing ports where the environment supports it

## Monitoring and troubleshooting

- [ ] Syslog or SIEM forwarding configured. Dashboard event log alone is not a retention strategy.
- [ ] Filter the event log on security appliance events to review key permits and denies
- [ ] Packet capture on MX or MR when a specific app or flow misbehaves. Built-in capture is one of the better parts of the platform, use it before escalating.
- [ ] Firmware kept current on a defined cadence rather than ad hoc

## Drift

Templates, tags, and local tweaks drift over time. Keep a running gotchas/change-notes list per client, because Dashboard doesn't make override history obvious.

## References

- [Meraki Best Practice Design — MX Security and SD-WAN](https://documentation.meraki.com/Architectures_and_Best_Practices/Cisco_Meraki_Best_Practice_Design/Best_Practice_Design_-_MX_Security_and_SD-WAN/General_MX_Best_Practices)
- [Meraki Trust and security tools](https://meraki.cisco.com/trust/)
- [Cisco advisory: Meraki Local Status Page configuration hardening](https://sec.cloudapps.cisco.com/security/center/content/CiscoSecurityAdvisory/cisco-sa-meraki-lsp-7xySn6pj)
