# Architecting Palo Alto Networks NGFW

Design reference for greenfield and refresh Palo Alto deployments, assuming PAN-OS 11.1. It sticks to the decisions that are hard to walk back later: platform sizing, the HA model, Panorama structure, and the segmentation/policy model. Feature-level hardening is in [security-baseline.md](../baselines/security-baseline.md); this doc sits above that.

## The decisions that are hard to undo

Most of what a firewall does day to day can be changed with a config push. A few things can't be changed without a maintenance window, a re-cable, or a rebuild, so it's worth getting them right at the start:

| Decision | Why it's sticky |
|---|---|
| Platform / model sizing | Throughput with all subscriptions and decryption on is roughly a third of the datasheet App-Mix number. Undersizing means a hardware swap, not a license bump. |
| Deployment mode (L3 / L2 / vwire / TAP) | Changing an in-path firewall's mode touches the L2/L3 config on every device next to it. Pick per interface based on what the insertion point needs. |
| HA model (A/P vs A/A) | Active/active brings in asymmetric routing, session ownership, and floating IPs. Most deployments don't need it, and it gets chosen by accident more often than on purpose. |
| Panorama device-group and template hierarchy | Re-parenting device groups after policy exists means untangling inheritance and shared-object collisions. Design the tree before the first rule. |
| Zone / segmentation model | Every security and NAT rule references zones. Resegmenting later is a rulebase rewrite. |

## Sizing

Size to the throughput you get with your features enabled, not the headline number. Threat Prevention, decryption, and logging each cost dataplane capacity, and the costs stack. The datasheet lists separate figures (App-ID only, Threat Prevention on, and so on); the number that matters for a fully-featured deployment is the lowest of those, with headroom taken off.

Pull these before you pick a model:

- Peak throughput and peak new connections per second (CPS). CPS is usually what kills an undersized box, not raw Gbps.
- Concurrent session count at peak.
- Decryption ratio: how much traffic you'll actually decrypt. This is the biggest single multiplier on capacity and the one most often left out of the sizing.
- Growth horizon. Size for the refresh cycle, typically three to five years, not for today.

Leave 30 to 50 percent headroom on whichever metric is the constraint. A firewall sitting at 85 percent dataplane CPU has nothing left to absorb an attack, a content update that shifts App-ID behavior, or an unplanned jump in decryption.

## Deployment mode, per insertion point

Mode is set per interface, and one firewall commonly mixes them.

- Layer 3 is the default and the right answer for most perimeter and inter-zone points. The firewall is a routed hop with its own IPs, does NAT, and terminates VPN.
- Virtual wire (vwire) inserts the firewall transparently between two devices with no IP renumbering. Use it when you're dropping a firewall into an existing path and can't change the L3 design, for example between an existing router and switch. It supports App-ID, security policy, and decryption without taking part in routing.
- Layer 2 has the firewall switching within a VLAN and enforcing policy between bridged VLANs. It's a niche choice; vwire is usually the cleaner transparent option.
- TAP mode takes a mirror/SPAN copy for visibility and App-ID reporting only, with no enforcement. It's useful for a pre-deployment traffic study, both to size the box and to see which App-IDs you'll be writing policy against.

Virtual routers (logical routers on newer PAN-OS) split the routing table. Use them to keep genuinely separate routing domains apart, such as an internet-edge VR and an internal VR on the same chassis, rather than running everything in one table and relying on policy to contain the leakage.

## High availability

Default to active/passive. Failover is deterministic, it's easier to reason about, and it covers the actual requirement (traffic keeps flowing if a box dies) for almost every design.

Use active/active only when you have a specific requirement it solves: asymmetric routing you can't get rid of, or a need to use both dataplanes at once. It adds session setup and ownership, HA3 packet forwarding, and floating-IP or ARP-load-sharing config, all of which you'll pay for in future troubleshooting.

Regardless of model, get the links right:

- Use dedicated HA1 (control) and HA2 (data/session-sync) links. Don't share them with dataplane interfaces. HA2 on a shared data interface works fine until a traffic spike makes failover detection unreliable. See [gotchas.md](../gotchas/gotchas.md).
- Add an HA1 backup link so one control-link failure doesn't cause split-brain.
- Tune link and path monitoring to real outage conditions, so failover fires on an actual upstream failure and not on a transient blip.
- Leave preemption off unless you have a reason for it. Automatic fail-back the moment the preferred box recovers just gives you two failovers instead of one, often before that box is really healthy.

## Panorama and management architecture

If there's more than one firewall, use Panorama. Managing a fleet box by box is how config drift and inconsistent policy happen.

Templates and template stacks carry device and network config (interfaces, zones, routing, log settings, server profiles). Stack them with a common baseline template at the bottom and site or role-specific templates overriding on top. Stack order is evaluated bottom to top, and a lower template can quietly overwrite a value you meant to set higher up, so set the order deliberately instead of discovering it later.

Device groups carry policy and objects and inherit top-down. Build the hierarchy around how policy actually differs:

- A Shared level for truly global objects and rules.
- Regional or functional parent groups for policy common to a set of sites.
- Leaf device groups per site or HA pair for local specifics.

Pre-rules and post-rules are the Panorama mechanism for this: pre-rules evaluate before local firewall rules, post-rules after. Put global guardrails such as a catch-all deny or management-access rules in Shared post-rules where a local admin can't shadow them, and put broadly-applicable allow policy in pre-rules.

Object hygiene is the recurring problem. An object defined in both Shared and a device group overrides silently depending on inheritance, and that's a common cause of "why did this rule stop matching" tickets. Agree on a naming convention and a rule for where objects live before the fleet grows.

Size and protect Panorama itself. Split the management and log-collector roles (or use dedicated log collectors) once log volume is real, and treat Panorama as a Tier-0 asset, because compromising it means compromising the whole fleet.

## Segmentation and the policy model

Zones are the backbone of the policy model, so zone design is segmentation design. Build purpose-driven zones (user, server, DMZ, guest, OT, management, and so on) rather than a flat trust/untrust pair, because every rule you write afterward either benefits from or is constrained by this.

A few things worth committing to up front:

- Interzone default-deny is the contract. Confirm the intrazone-allow / interzone-deny defaults are intact and haven't been neutralized by a broad any-any allow that crept in during a cutover.
- Use App-ID and User-ID from the start rather than "port rules now, App-ID later." The later migration is real work (see PANW's App-ID migration best practices) and tends not to happen. Writing identity- and application-based policy from day one avoids the rewrite.
- Apply Zone Protection to every ingress zone, especially internet-facing ones, with thresholds tuned to a measured CPS baseline. See [zone-protection-cli.md](../configs/zone-protection-cli.md).
- Treat decryption as an architecture decision. Where you decrypt sets what every downstream security profile can see. Plan the decryption zones, the exclusion process, and cert distribution before turning it on, because retrofitting decryption into a mature rulebase is disruptive.

A zero-trust posture is the direction of travel: segment by function, enforce on identity and application rather than IP and port, and shrink zones toward the workload rather than the subnet. It's an incremental program, not a switch you flip, and the zone and policy model above is what makes it reachable.

## Where the firewalls physically go

Common insertion points and the intent behind each:

- Internet edge / perimeter: the usual north-south enforcement point. L3, HA pair, decryption for outbound, threat prevention on everything.
- Data center / east-west: segmenting server tiers and containing lateral movement. This is where undersizing bites, because east-west volume dwarfs perimeter volume.
- Campus / inter-zone core: enforcement between user populations and internal services.
- Cloud: VM-Series or Cloud NGFW, see [ngfw-in-azure.md](ngfw-in-azure.md). The design principles carry over; the plumbing (routing, scaling, HA) is cloud-native and works differently.

## Gotchas

- Datasheet throughput is not deployment throughput. Plan against the all-features-enabled figure with your real decryption ratio, minus headroom.
- Active/active chosen by default causes problems later. If nobody can name the requirement it satisfies, use active/passive.
- Template stack order and Shared-vs-device-group object collisions generate the most confusing Panorama tickets. Both come down to design discipline up front, not cleverness later.
- HA1/HA2 on shared dataplane interfaces passes lab testing and fails under production load. Dedicate the interfaces.
- Deferring App-ID/User-ID and decryption turns each into a migration project instead of a property of the design. The cheapest time to adopt them is before the rulebase exists.

## References

- PAN-OS Best Practices: [docs.paloaltonetworks.com/best-practices](https://docs.paloaltonetworks.com/best-practices)
- Best Practices for Migrating to Application-Based Policy: [docs.paloaltonetworks.com/best-practices/app-id-best-practices](https://docs.paloaltonetworks.com/best-practices)
- HA overview (PAN-OS 11.1): [docs.paloaltonetworks.com/pan-os/11-1/pan-os-admin/high-availability](https://docs.paloaltonetworks.com/pan-os/11-1/pan-os-admin/high-availability)
- Panorama Administrator's Guide (11.1): [docs.paloaltonetworks.com/panorama/11-1/panorama-admin](https://docs.paloaltonetworks.com/panorama/11-1/panorama-admin)
- Zero Trust best practices: [docs.paloaltonetworks.com/best-practices/zero-trust-best-practices](https://docs.paloaltonetworks.com/best-practices)
