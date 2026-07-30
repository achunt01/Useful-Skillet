# Network Troubleshooting Methodology

Vendor-agnostic approach for working "weird network happenings": app issues, intermittent connectivity, partial outages, and the ones that don't produce a clean error anywhere.

## Before touching anything

Establish these four facts first. Skipping this is how an hour gets spent debugging the wrong layer.

1. **Scope** — one user, one site, one app, or everything? Single-user issues are rarely network.
2. **Timeline** — when did it start, and what changed in that window? Firmware, policy push, ISP maintenance, cert expiry, DHCP scope change, a new SaaS integration.
3. **Reproducibility** — consistent, intermittent, or time-of-day correlated? Intermittent points at capacity, path selection, or something with a timer.
4. **Path** — draw the actual traffic path end to end, including anything doing NAT, decryption, proxying, or inspection. Most "weird" behavior lives at a device nobody remembered was in the path.

## Work the path, not the symptom

Order of investigation that tends to converge fastest:

- **DNS first.** A surprising share of "the app is broken" tickets are resolution problems: split-horizon mismatches, stale records, a resolver change, DNS security sinkholing something legitimate.
- **Then reachability.** Layer 3 path, routing, asymmetric return paths, MTU.
- **Then policy.** Firewall rule match, App-ID/L7 misidentification, geo-blocking, decryption breaking the session.
- **Then the application.** TLS negotiation failure, cert chain problems, server-side rate limiting.

## Firewall / edge device logs

What to look at, in order:

- **Traffic logs** — is the session even reaching the firewall? Which rule matched? Is the session ending normally or being reset/aged out?
- **Threat logs** — was it dropped by a security profile rather than a policy rule? This is the case that looks like a network problem and isn't.
- **Session state** — is the session established but with wildly asymmetric byte counts? That's usually a return-path or MTU story.
- **System logs** — device-level events that correlate to the timeline: HA events, interface flaps, commit history, subscription/license state, cert expiry.

## When to reach for a PCAP

Go to packet capture when logs tell you *that* something failed but not *why*. Specifically:

- Session establishes then dies mid-transfer
- TLS handshake failures with no useful error client-side
- MTU / fragmentation suspicion
- App works from one location and not another with identical policy
- Anything where a vendor is going to ask for a capture eventually (get it while the issue is live)

### Capture discipline

- Capture at **both** sides of the suspect device where possible. A capture from one side only proves what arrived, not what was sent.
- Filter at capture time. Unfiltered captures on a busy link roll over before the event reproduces.
- Timestamp-align captures against log timestamps. This requires NTP to actually be working on every device involved, which is worth verifying before you need it.
- Note the capture point in the filename. Six captures later, "capture1.pcap" is useless.

### What to look for in the capture

| Pattern | Usually means |
|---|---|
| SYN with no SYN-ACK | Blocked upstream, or asymmetric routing |
| SYN-ACK arriving but session failing | Return path problem, or something resetting mid-stream |
| Retransmissions clustering at a consistent size | MTU / fragmentation |
| RST from an unexpected source IP | Inline device injecting resets (IPS, proxy, decryption failure) |
| TLS Client Hello then immediate close | Cipher/protocol mismatch, or cert validation failure |
| Duplicate ACKs and window changes | Congestion or a genuinely lossy path, not a policy issue |

## MTU specifically

Worth its own note because it accounts for a large share of "works for small stuff, breaks on large transfers" tickets:

- Symptom pattern: DNS and pings fine, web pages partially load, file transfers and VPN traffic fail
- Common causes: tunnel overhead (IPsec, GRE, SD-WAN), PMTUD blocked by an ICMP filter somewhere in the path
- Test with progressively larger DF-set pings to find the actual working MTU rather than guessing

## Correlating across sources

For anything spanning more than one system:

- Firewall/edge logs for the network path
- EDR/MDR for endpoint-side behavior on the affected client
- SIEM/XDR for correlation across sources and for the timeline view
- Identity logs (Entra ID sign-in logs) when the failure is auth-adjacent, which it often is when "the app broke" coincides with a policy change

## Documenting the outcome

Capture these for every non-trivial issue, both for the client and for future-you:

- Symptom as reported vs. actual root cause (these are frequently unrelated, and the gap is the useful part)
- What ruled out the wrong theories
- The fix, and whether it's a workaround or a real remediation
- Whether anything should be added to monitoring so this is detected rather than reported next time
