# FortiGate Hardening Baseline

Baseline hardening checklist for FortiGate / FortiOS. Cross-references Fortinet's own hardening guide and the CIS FortiGate benchmark.

## Administrative access

- [ ] Change default admin credentials immediately. Default creds are publicly known.
- [ ] Disable or rename the default `admin` account if not required
- [ ] Disable the `maintainer` admin account (serial-console recovery account, effectively a physical-access backdoor)
- [ ] Enable password policies (complexity, expiry, reuse)
- [ ] Lockout thresholds: CIS recommends minimum 3 retries and a 900-second lockout. FortiOS default is 3 retries and 60 seconds.
- [ ] Restrict admin access to specific trusted hosts, not `0.0.0.0/0`
- [ ] Use a reserved management interface where the platform supports it
- [ ] MFA on admin accounts (FortiToken or external auth)

## Attack surface reduction

- [ ] `System > Feature Visibility` — disable unused features/services
- [ ] Disable HTTP, Telnet, FTP, and unnecessary SNMP on all interfaces
- [ ] Disable unused interfaces entirely
- [ ] Disable unused protocols on interfaces that remain enabled
- [ ] Use **local-in policies** to close open ports and restrict access to services on the FortiGate itself. This is the control most often missed, since interface-level settings don't cover everything the device listens on.
- [ ] Disable auto USB firmware/config installation (disabled by default, verify it stayed that way)
- [ ] Physically secure the device. Physical access allows bypass or loading alternate firmware after a manual reboot.

## System settings

- [ ] NTP sync configured (log correlation is worthless without it)
- [ ] DNS configured to controlled resolvers
- [ ] Global commands set for stronger encryption (`strong-crypto`, TLS version floors)
- [ ] Custom private key specified for the encryption process
- [ ] Firmware kept current. FortiGate CVEs with CVSS scores above 9.0 land regularly, and VPN-facing services are a repeat target.

## Policies and security profiles

- [ ] Review firewall policies for overly broad `any/any` entries and unused rules
- [ ] Apply IPS sensors to relevant policies; confirm FortiGuard signature updates are current
- [ ] Enable Antivirus and Web Filtering on appropriate policies with current definitions
- [ ] Enable Botnet C&C domain blocking
- [ ] Enable the DoS policy
- [ ] Enable anomaly logging in monitor mode first, to baseline traffic before tuning thresholds. Note false positives and adjust rather than enforcing blind.

## VPN

- [ ] IPsec with AES-256, no legacy cipher suites
- [ ] Certificate-based authentication over pre-shared keys
- [ ] Restrict VPN access using address groups and user groups, not open access to all internal subnets
- [ ] MFA on remote access

## Logging and monitoring

- [ ] Auditing and logging configured and forwarded off-box (FortiAnalyzer, syslog, or SIEM)
- [ ] SNMP/NetFlow/sFlow for resource monitoring. Resource baselines are needed for tuning IPS signature rates, recognizing abnormal activity, and capacity planning.
- [ ] Configuration backups on a schedule. FortiManager for fleet config backup and automation.
- [ ] Register the device with Fortinet Support

## Self-audit

FortiGates with the Enterprise License Bundle (or another bundle plus the Attack Surface Security Rating package) self-audit against CIS controls on a schedule and report specific remediation steps. Worth enabling where licensed, because it also catches drift after you've brought a device into compliance. Without it, re-validation is manual and config quietly falls out of compliance with nobody noticing.

## References

- [Fortinet: Hardening your FortiGate](https://docs.fortinet.com/document/fortigate/7.6.0/best-practices/555436/hardening)
- [Fortinet hardening best practices (community)](https://community.fortinet.com/t5/FortiGate/Technical-Tip-Hardening-Best-Practices-Secure-Network-and/ta-p/392811)
- CIS Benchmark for Fortinet FortiGate (FortiOS 6.4+)
