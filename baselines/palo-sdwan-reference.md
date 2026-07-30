# Palo Alto SD-WAN Reference

Design and deployment reference for Panorama-managed Palo Alto SD-WAN using the SD-WAN plugin. Structured around a hub-and-spoke build with dual-ISP branches, which is the most common pattern in practice.

> **Not Prisma SD-WAN.** This is the **PAN-OS SD-WAN plugin** (SD-WAN as a feature of Panorama-managed NGFWs). The CloudGenix-derived **Prisma SD-WAN** (ION devices + cloud Controller) is a separate product — see [../architecture/prisma-sd-wan.md](../architecture/prisma-sd-wan.md).

## Configuration elements and how they relate

The plugin's object model trips people up because four things have to line up before a policy rule works:

| Element | Role |
|---|---|
| **Link tag** | Identifies the role of a physical link (e.g. `Primary`, `Secondary`, `MPLS`, `LTE`). A link can have exactly one tag. Tags control the *order* interfaces get used. |
| **SD-WAN interface profile** | Applied to the physical WAN interface. Specifies the tag, the link type (broadband, ADSL, cable, LTE, etc.), and max upload/download bandwidth. |
| **Path Quality profile** | Latency, jitter, and packet loss thresholds. When any single threshold is exceeded, the firewall picks a new preferred path. |
| **Traffic Distribution profile** | How the firewall selects the new best path. References link tags to narrow the candidate set. |
| **SD-WAN policy rule** | Where it all comes together: source/destination, applications/services, plus a Path Quality profile and a Traffic Distribution profile. |

Flow: tag goes on the interface profile → interface profile goes on the physical interface → tags get referenced by the Traffic Distribution profile → the policy rule references both profiles.

## Zones

The plugin uses four predefined zones: `zone-internal`, `zone-to-hub`, `zone-to-branch`, `zone-internet`. Names are **case sensitive**.

**Pre-provision these zones before enabling SD-WAN.** The plugin will create them, but creating them yourself first means you can add them to existing security policy rules ahead of the transition instead of scrambling after. For brownfield environments, plan the mapping from existing zones to the predefined zones as part of design.

Every transport or interface participating in SD-WAN needs to be in the same security zone.

## Deployment workflow

1. Pre-provision predefined zones and map existing zones to them
2. Create link tags, shared across hub and spoke device groups
3. Create SD-WAN interface profiles per circuit type, apply to physical WAN interfaces
4. Enable SD-WAN on branch WAN interfaces
5. Enable SD-WAN on hub WAN interfaces with the same tags and profiles
6. Define Path Quality and Traffic Distribution profiles
7. Create SD-WAN policy rules
8. Add hub and branch devices to the SD-WAN plugin (`Panorama > SD-WAN > Devices`) — device name, router name, link tags, BGP parameters. Bulk-add via CSV with serial number, device type, zone mappings, loopback address, prefixes to redistribute, AS number, router ID, virtual router name.
9. Create the VPN cluster (hub-and-spoke or mesh)
10. Validate tunnel creation, routing, and failover

Order matters at step 6: link tags must be committed and pushed to hubs and branches before Panorama can associate them in a Traffic Distribution profile.

## Auto VPN tunnel math

With Auto VPN, each hub-to-branch relationship builds a tunnel for every combination of links on each side. Dual ISP on both sides produces four tunnels:

- Hub primary ↔ Branch primary
- Hub primary ↔ Branch secondary
- Hub secondary ↔ Branch secondary
- Hub secondary ↔ Branch primary

All four are health-monitored continuously. Degradation or failure moves traffic to the next best tunnel with no administrative intervention, and traffic returns to the preferred path once health thresholds recover.

Note on cluster types: hub-and-spoke clusters need hub devices included. Mesh clusters build overlay tunnels from each remote site to every other remote site and don't require hubs. The plugin never builds hub-to-hub overlay tunnels.

## Traffic distribution methods

| Method | Use case |
|---|---|
| **Best Available Path** | Uses the healthiest path regardless of link cost. Good default for Phase 1 validation when you want failover proven before optimizing. |
| **Top Down Priority** | Strict tag ordering. The method for expensive backup links: put the LTE tag last so it's only used when the cheaper links are down or oversubscribed. |
| **Weighted Session Distribution** | Percentage split across tags. For deliberately load-sharing across circuits rather than treating one as backup. |

## Path Quality profile thresholds

Ranges: latency 10 to 2,000 ms, jitter 10 to 1,000 ms, packet loss as a percentage.

**Start conservative.** Tight thresholds on day one cause path flapping, which looks like an SD-WAN problem and is actually a tuning problem. Deploy loose, observe real path behavior through a full business cycle, then tighten based on what the applications actually need.

Any path exceeding a single threshold is excluded from selection. Remaining qualifying paths are then subject to the distribution method.

## Routing design

Decisions worth making explicitly rather than by default:

- **Advertise only what's needed.** Internal data center subnets over the overlay, not a default route. Sending a default route over SD-WAN backhauls all internet traffic to the hub, which is usually the opposite of what you want.
- **Local internet egress at branches.** General web and SaaS traffic exits locally via the primary DIA link with failover to secondary. Content filtering and security inspection stay enforced by the local firewall.
- **Branch transit.** Decide whether branches provide transit for each other. Usually no: all inter-site traffic through the hub keeps the routing model simple and the security enforcement point predictable.
- **BGP vs. static.** BGP is the default assumption in the plugin, but static routing works by omitting the BGP information in the plugin config and using normal virtual router static routes.

## Scaling with templates and variables

Bandwidth differs per site, but the design shouldn't. Use template variables for the per-site values (circuit bandwidth, IP addressing) so the interface profile structure stays identical across every branch. Validate this on the pilot site before onboarding the rest, because retrofitting variables across an already-deployed fleet is significantly more work than starting with them.

## Phased rollout

- Pick a pilot branch that's **representative** of the broader environment, not the easiest one. A single-circuit outlier site or the smallest branch won't surface the problems you need to find.
- Validate tunnel establishment, routing, and both failover *and* recovery before expanding. Recovery is the half that gets skipped and the half that generates tickets.
- Handle single-circuit sites as a documented exception rather than a special design.

## Gotchas

- **Plugin/PAN-OS version compatibility** is not always obvious from release notes. Confirm before upgrading either independently.
- **SaaS Quality profile and Error Correction profile are mutually exclusive** on the same SD-WAN policy rule (PAN-OS 10.0.2 behavior, plugin 2.0+). Associating one blocks the other.
- **Link tag is one-per-link.** Grouping multiple physical links under a shared tag is how you get bundle behavior; you can't multi-tag a single interface to achieve it.
- **Zone name case sensitivity** on the predefined zones causes commit failures that read as unrelated errors.
- Health checks run over a VPN tunnel, including for DIA circuits. Each DIA circuit gets a monitoring tunnel. Blocking that traffic upstream breaks path monitoring while leaving the data path apparently fine.

## References

- [SD-WAN deployment workflow](https://docs.paloaltonetworks.com/sd-wan/administration/sd-wan-deployment-workflow)
- [SD-WAN configuration elements](https://docs.paloaltonetworks.com/sd-wan/getting-started/sd-wan-configuration-elements)
- [Plan your SD-WAN configuration](https://docs.paloaltonetworks.com/sd-wan/1-0/sd-wan-admin/sd-wan-overview/plan-sd-wan-configuration)
- [SD-WAN traffic distribution profiles](https://docs.paloaltonetworks.com/sd-wan/administration/enable-sd-wan-without-auto-vpn/manage-sd-wan-link-failovers/sd-wan-traffic-distribution-profiles)
