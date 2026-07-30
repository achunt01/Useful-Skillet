# Prisma Access

Architecture and design reference for Prisma Access — Palo Alto's cloud-delivered SASE. This is the general model: the three connection types, how they route to each other, capacity/bandwidth, and the design decisions that bite later. Prisma SD-WAN (the branch WAN-edge product that often feeds remote networks) is its own doc: [prisma-sd-wan.md](prisma-sd-wan.md).

## What it is, and what replaces what

Prisma Access delivers the NGFW security stack (App-ID, Threat Prevention, URL Filtering, WildFire, DNS Security, DLP, and the CASB features) from Palo Alto's cloud instead of from a box you rack. It's the "SSE/security" half of SASE; pair it with SD-WAN for the network half. The thing it most directly replaces is **backhaul-to-the-datacenter remote-access VPN** — instead of hauling every remote user and branch to a central firewall, users and sites connect to the nearest cloud location and get consistent policy there.

Management is via **Strata Cloud Manager** (the current cloud UI) or, in older/Panorama-managed tenants, via the Panorama Cloud Services plugin. New deployments land in Strata Cloud Manager.

## The three connection types

Everything in Prisma Access is built from three primitives. A full enterprise deployment usually runs all three at once.

| Type | Connects | Transport | Enforces security policy? |
|---|---|---|---|
| **Mobile Users** | Remote/roaming users | GlobalProtect app (or Prisma Access Agent), or clientless/explicit proxy | Yes |
| **Remote Networks** | Branches, sites, campuses | IPsec tunnel from an on-prem edge device (SD-WAN CPE, router, or firewall) | Yes |
| **Service Connections** | Data centers, private clouds, HQ — the resources users need to *reach* | IPsec tunnel from on-prem | **No** — routing/enablement only |

The distinction that trips people up: **service connections don't enforce policy and can't originate internet traffic.** They exist to make private resources reachable and to route between mobile users and remote networks. Security enforcement happens at the mobile-user and remote-network ingress points, not at the service connection.

### How they route to each other

Palo Alto's standing recommendation: **always create at least one service connection**, even if you think you don't need one, because the service connection is what stitches the routing fabric together so mobile users and remote networks can reach each other and reach private apps. Without it you get islands. The service connection to the data center is also typically how internal DNS and private-app routes get advertised into the Prisma Access fabric (via BGP or static).

Design the routing deliberately:

- Advertise **specific internal prefixes** over service connections, not a default route, unless you genuinely want to backhaul.
- Decide the internet-egress model per connection type — mobile users and remote networks generally egress locally from their Prisma Access location; that's the point of the architecture.
- Plan the overlap/summarization carefully; overlapping RFC1918 space across sites is the usual routing headache.

## Locations, compute locations, and capacity

Prisma Access runs in **locations** (geographic points of presence). Each location maps to a **compute location** — the underlying region where the security processing actually happens. This mapping matters for two reasons: latency (users connect to the nearest location) and bandwidth (allocation is done at the compute-location level).

Key capacity concepts:

- **Bandwidth is allocated per compute location**, and Prisma Access distributes it dynamically across the sites/tunnels homed to that compute location based on demand. You size and pay for aggregate bandwidth, not per-tunnel.
- **Mobile-user licensing** is by user count (or a units model depending on the license), and mobile-user capacity **autoscales**: as more users log in to a location, Prisma Access spins up additional mobile-user security processing nodes (MU-SPNs) automatically. You don't pre-provision per-location user capacity.
- Each customer gets a **dedicated dataplane**; one tenant's autoscaling event doesn't touch another tenant's capacity.

## The IP allocation gotcha (plan for autoscaling)

The single most under-planned thing in Prisma Access. Because the service **autoscales**, its **egress IP addresses change** — new nodes bring new public IPs, and each location has both active and reserved (standby) IP ranges.

If any SaaS provider, partner, or B2B integration does **source-IP allowlisting**, you must allowlist **both the active and the reserved** public IPs for the relevant locations, or access breaks the moment an autoscaling event moves users to a node whose IP wasn't on the list. Pull the current IP list from the tenant (it's exposed via API and in the UI) and build the allowlisting process around the fact that the set can grow. Treat "we allowlist Prisma's IPs somewhere" as a design input to capture on day one, not a surprise during the first scale event.

## Mobile Users: design notes

- **GlobalProtect** is the mainstream agent; the newer **Prisma Access Agent** is the direction of travel for unified SASE. Pick per your rollout maturity.
- Users connect to the **nearest location**; portal/gateway selection and internal-vs-external gateway logic determine on-net vs off-net behavior.
- **HIP checks** (posture: disk encryption, AV running, patch level) enforce endpoint state before granting access — carry the same HIP discipline over from on-prem GlobalProtect.
- Decide **split-tunnel vs. full-tunnel** deliberately and document it; it's the same trade-off as on-prem GP and drives how much traffic you're paying to inspect.

## Remote Networks: design notes

- Branches connect via **IPsec** from an on-prem termination device — an SD-WAN edge (commonly Prisma SD-WAN, see the sibling doc), a router, or a firewall.
- **ECMP** across multiple links to a location increases aggregate throughput and resilience, but if you enable **route summarization** on an ECMP location you must enable it on **all** links to that location or the commit fails.
- Bandwidth is drawn from the compute-location pool; size the aggregate, and remember a single remote-network tunnel has its own throughput ceiling, so very large sites may need multiple tunnels/links.

## Gotchas

- **No service connection = islands.** Mobile users and remote networks can't reach each other or private apps without the routing a service connection provides. Create one even in a mobile-users-only design if private-app access is in scope.
- **Service connections don't filter and can't egress to internet.** Don't design as if they're an enforcement point; they aren't.
- **Autoscaling IPs break SaaS allowlisting.** Allowlist active *and* reserved IPs for every location in scope, and treat the IP set as something that grows.
- **Route summarization on ECMP is all-or-nothing per location** — enable on every link or the commit errors, and the error text won't obviously point at this.
- **Overlapping private address space** across branches/DCs is the routing problem you'll actually spend time on; plan addressing and summarization before onboarding sites.
- **Mind the management plane you're on.** Strata Cloud Manager vs. Panorama-managed tenants differ in workflow and feature availability; know which one the tenant is before quoting a procedure.

## References

- Prisma Access Administration: [docs.paloaltonetworks.com/prisma-access/administration](https://docs.paloaltonetworks.com/prisma-access/administration)
- Use a Service Connection to enable access between Mobile Users and Remote Networks: [docs.paloaltonetworks.com/prisma-access/administration/prisma-access-service-connections/use-a-service-connection-to-enable-access-between-mobile-users-and-remote-networks](https://docs.paloaltonetworks.com/prisma-access/administration/prisma-access-service-connections/use-a-service-connection-to-enable-access-between-mobile-users-and-remote-networks)
- Prisma Access Locations: [docs.paloaltonetworks.com/prisma-access/administration/prisma-access-overview/list-of-prisma-access-locations](https://docs.paloaltonetworks.com/prisma-access/administration/prisma-access-overview/list-of-prisma-access-locations)
- Allocate Remote Network Bandwidth: [docs.paloaltonetworks.com/prisma-access/administration/prisma-access-remote-networks/allocate-remote-network-bandwidth](https://docs.paloaltonetworks.com/prisma-access/administration/prisma-access-remote-networks/allocate-remote-network-bandwidth)
- Prisma Access Infrastructure Management: [docs.paloaltonetworks.com/prisma-access/administration/prisma-access-overview/prisma-access-infrastructure-management](https://docs.paloaltonetworks.com/prisma-access/administration/prisma-access-overview/prisma-access-infrastructure-management)
