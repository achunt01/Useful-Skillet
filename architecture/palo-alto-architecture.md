# Architecting Palo Alto Networks NGFW

Design reference for greenfield and refresh Palo Alto deployments. Assumes PAN-OS 11.1 as the baseline. Focus is on the decisions that are expensive to reverse later: platform sizing, HA model, Panorama structure, and the segmentation/policy model. Feature-level hardening lives in [security-baseline.md](../baselines/security-baseline.md); this doc is the layer above that.

## The decisions that are hard to undo

Most of what a firewall does day to day can be changed with a config push. A handful of things can't be changed without a maintenance window, a re-cable, or a rebuild. Get these right up front:

| Decision | Why it's sticky |
|---|---|
| **Platform / model sizing** | Throughput with all subscriptions and decryption enabled is roughly a third of the datasheet App-Mix number. Undersizing means a hardware swap, not a license bump. |
| **Deployment mode** (L3 / L2 / vwire / TAP) | Changing an in-path firewall's mode touches every adjacent device's L2/L3 config. Pick per-interface based on what the insertion point actually needs. |
| **HA model** (A/P vs A/A) | Active/active pulls in asymmetric-routing handling, session ownership, and floating IPs. Almost nobody needs it; it's chosen by mistake far more often than by requirement. |
| **Panorama device-group and template hierarchy** | Re-parenting device groups after policy has been built means untangling inheritance and shared-object collisions. Design the tree before the first rule. |
| **Zone / segmentation model** | Zones are referenced by every security and NAT rule. Renaming or resegmenting later is a rulebase rewrite. |

## Sizing

Size to the *enabled-feature* throughput, not the headline number. Threat Prevention, decryption, and logging each cost dataplane capacity, and they compound. The datasheet gives you separate figures — App-ID only, Threat Prevention on, and so on — and the real-world number for a fully-featured deployment is the lowest of those, minus headroom.

What to actually pull before choosing a model:

- Peak throughput *and* peak new-connections-per-second (CPS) — CPS is what kills undersized boxes, not raw Gbps.
- Concurrent session count at peak.
- Decryption ratio: what fraction of traffic will actually be decrypted. This is the single biggest capacity multiplier and the most commonly ignored one.
- Growth horizon. Size for the refresh cycle (typically 3–5 years), not today.

Leave 30–50% headroom on the constraining metric. A firewall running at 85% dataplane CPU has no room to absorb an attack, a content update that changes App-ID behavior, or an unplanned decryption increase.

## Deployment mode, per insertion point

Mode is an interface-level decision, and a single firewall commonly mixes them.

- **Layer 3** — the default and the right answer for most perimeter and inter-zone insertion points. The firewall is a routed hop with its own IPs, does NAT, and terminates VPN.
- **Virtual wire (vwire)** — transparent insertion between two devices with no IP renumbering. The move when dropping a firewall into an existing path where you can't change the L3 design (e.g. between an existing router and switch). Supports App-ID, security policy, and decryption without participating in routing.
- **Layer 2** — the firewall switches within a VLAN and enforces policy between VLANs bridged through it. Niche; usually vwire is the cleaner transparent option.
- **TAP** — receives a mirror/SPAN copy. Visibility and App-ID reporting only, no enforcement. Useful for a pre-deployment traffic study to size and to see what App-IDs you'll actually be writing policy against.

Virtual routers (or logical routers on newer PAN-OS) partition the routing table. Use them to keep genuinely separate routing domains apart — e.g. an internet-edge VR and an internal VR on the same chassis — rather than leaning on one global table and hoping policy contains the leakage.

## High availability

Default to **active/passive**. It's simpler to reason about, failover is deterministic, and it covers the requirement ("the firewall keeps forwarding if a box dies") for the overwhelming majority of designs.

Reach for **active/active** only when you have a concrete requirement it uniquely solves — asymmetric routing you can't eliminate, or a need to use both boxes' dataplane concurrently. It brings session setup/ownership, HA3 packet forwarding, and floating-IP or ARP-load-sharing complexity that you will pay for in every future troubleshooting session.

Cabling and links that matter regardless of model:

- **Dedicated HA1 (control) and HA2 (data/session-sync) links.** Do not share these with dataplane interfaces. HA2 sharing a data interface works fine until a traffic spike makes failover detection unreliable — see [gotchas.md](../gotchas/gotchas.md).
- **HA1 backup link** so a single control-link failure doesn't cause split-brain.
- **Link and path monitoring** tuned to real outage conditions, so failover triggers on an actual upstream failure and not on a transient.
- **Preemption off** unless you have a specific reason for it. An automatic fail-back the moment the preferred box recovers just gives you two failovers instead of one, often before the recovered box is truly healthy.

## Panorama and management architecture

If there is more than one firewall, there is Panorama. Managing a fleet device-by-device is how config drift and inconsistent policy happen.

**Templates and template stacks** carry device/network config (interfaces, zones, routing, log settings, server profiles). Stack them so a common baseline template sits at the bottom and site/role-specific templates override on top. Stack order is evaluated bottom-to-top and a lower template can silently overwrite a value you meant to set higher up — order deliberately, don't discover it.

**Device groups** carry policy and objects, and they inherit top-down. Design the hierarchy around how policy actually differs:

- A **Shared** level for truly global objects and rules.
- Regional or functional parent groups for policy common to a set of sites.
- Leaf device groups per site/HA-pair for local specifics.

**Pre- and post-rules** are the Panorama mechanism: pre-rules evaluate before local firewall rules, post-rules after. Put global guardrails (e.g. a global deny, management-access rules) in Shared post-rules where a local admin can't shadow them, and put the broadly-applicable allow policy in pre-rules.

Object hygiene is the recurring failure mode. An object defined both in Shared and in a device group silently overrides depending on inheritance, and that's a top source of "why did this rule stop matching" tickets. Establish a naming convention and a rule about where objects live before the fleet grows.

Size and protect Panorama itself: separate the management and log-collector roles (or dedicated log collectors) once log volume is real, and treat Panorama as a Tier-0 asset — its compromise is fleet-wide.

## Segmentation and the policy model

Zones are the backbone of the policy model, so the zone design *is* the segmentation design. Build purpose-driven zones (user, server, DMZ, guest, OT, management, etc.) rather than a flat trust/untrust pair, because every future rule either benefits from or is constrained by this choice.

Principles worth committing to up front:

- **Interzone default-deny is the contract.** Confirm the intrazone-allow / interzone-deny defaults are intact and not neutralized by a broad any-any allow that crept in during a cutover.
- **App-ID and User-ID from day one**, not "port rules now, App-ID later." The later migration is real work (see PANW's App-ID migration best practices) and "later" tends not to arrive. Writing identity- and application-based policy from the start avoids the rewrite.
- **Zone Protection on every ingress zone**, internet-facing ones especially, with thresholds tuned to a measured CPS baseline. See [zone-protection-cli.md](../configs/zone-protection-cli.md).
- **Decryption is an architecture decision, not a checkbox.** Where you decrypt determines what every downstream security profile can actually see. Plan the decryption zones, the exclusion process, and the cert distribution before turning it on, because retrofitting decryption into a mature rulebase is disruptive.

A zero-trust posture is the direction of travel: segment by function, enforce on identity and application rather than IP and port, and shrink zones toward the workload rather than the subnet. It's an incremental program, not a mode you switch on, and the zone/policy model above is what makes it reachable.

## Where the firewalls physically go

Common insertion points and the intent behind each:

- **Internet edge / perimeter** — the classic north-south enforcement point. L3, HA pair, decryption for outbound, threat prevention on everything.
- **Data center / east-west** — segmenting server tiers and containing lateral movement. This is where undersizing bites, because east-west volume dwarfs perimeter volume.
- **Campus / inter-zone core** — enforcement between user populations and internal services.
- **Cloud** — VM-Series or Cloud NGFW; see [ngfw-in-azure.md](ngfw-in-azure.md). The design principles carry over; the plumbing (routing, scaling, HA) is cloud-native and different.

## Gotchas

- **Datasheet throughput is not deployment throughput.** The number you can plan against is the all-features-enabled figure with your real decryption ratio, minus headroom. Everything else is marketing math.
- **Active/active chosen by default is a self-inflicted wound.** If nobody can name the requirement it satisfies, the answer is active/passive.
- **Template stack order and Shared-vs-device-group object collisions** are the two Panorama issues that generate the most confusing tickets. Both are prevented by design discipline, not fixable by cleverness later.
- **HA1/HA2 on shared dataplane interfaces** survives lab testing and fails under production load. Dedicate the interfaces.
- **Deferring App-ID/User-ID and decryption** turns each into a migration project instead of a design property. Cheapest time to adopt them is before the rulebase exists.

## References

- PAN-OS Best Practices: [docs.paloaltonetworks.com/best-practices](https://docs.paloaltonetworks.com/best-practices)
- Best Practices for Migrating to Application-Based Policy: [docs.paloaltonetworks.com/best-practices/app-id-best-practices](https://docs.paloaltonetworks.com/best-practices)
- HA overview (PAN-OS 11.1): [docs.paloaltonetworks.com/pan-os/11-1/pan-os-admin/high-availability](https://docs.paloaltonetworks.com/pan-os/11-1/pan-os-admin/high-availability)
- Panorama Administrator's Guide (11.1): [docs.paloaltonetworks.com/panorama/11-1/panorama-admin](https://docs.paloaltonetworks.com/panorama/11-1/panorama-admin)
- Zero Trust best practices: [docs.paloaltonetworks.com/best-practices/zero-trust-best-practices](https://docs.paloaltonetworks.com/best-practices)
