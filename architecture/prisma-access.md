# Prisma Access

Architecture and design reference for Prisma Access, Palo Alto's cloud-delivered SASE. This covers the general model: the three connection types, how they route to each other, capacity and bandwidth, and the design decisions that cause trouble later. Prisma SD-WAN, the branch WAN-edge product that often feeds remote networks, has its own doc: [prisma-sd-wan.md](prisma-sd-wan.md).

## What it is, and what it replaces

Prisma Access delivers the NGFW security stack (App-ID, Threat Prevention, URL Filtering, WildFire, DNS Security, DLP, and the CASB features) from Palo Alto's cloud instead of from a box you rack. It's the security half of SASE; pair it with SD-WAN for the network half. What it most directly replaces is backhaul-to-the-datacenter remote-access VPN. Instead of hauling every remote user and branch back to a central firewall, users and sites connect to the nearest cloud location and get consistent policy there.

Management is through Strata Cloud Manager (the current cloud UI) or, in older Panorama-managed tenants, the Panorama Cloud Services plugin. New deployments land in Strata Cloud Manager.

## The three connection types

Everything in Prisma Access is built from three primitives. A full enterprise deployment usually runs all three at once.

| Type | Connects | Transport | Enforces security policy? |
|---|---|---|---|
| Mobile Users | Remote/roaming users | GlobalProtect app (or Prisma Access Agent), or clientless/explicit proxy | Yes |
| Remote Networks | Branches, sites, campuses | IPsec tunnel from an on-prem edge device (SD-WAN CPE, router, or firewall) | Yes |
| Service Connections | Data centers, private clouds, HQ, the resources users need to reach | IPsec tunnel from on-prem | No, routing/enablement only |

The part that trips people up: service connections don't enforce policy and can't originate internet traffic. They exist to make private resources reachable and to route between mobile users and remote networks. Security enforcement happens at the mobile-user and remote-network ingress points, not at the service connection.

### How they route to each other

Palo Alto's standing recommendation is to create at least one service connection even if you think you don't need one, because the service connection is what stitches the routing fabric together so mobile users and remote networks can reach each other and reach private apps. Without one you get islands. The service connection to the data center is also usually how internal DNS and private-app routes get advertised into the Prisma Access fabric, over BGP or static routes.

Design the routing deliberately:

- Advertise specific internal prefixes over service connections, not a default route, unless you actually want to backhaul.
- Decide the internet-egress model per connection type. Mobile users and remote networks normally egress locally from their Prisma Access location, which is the point of the architecture.
- Plan overlap and summarization carefully. Overlapping RFC1918 space across sites is the usual routing headache.

## Locations, compute locations, and capacity

Prisma Access runs in locations (geographic points of presence). Each location maps to a compute location, the underlying region where the security processing actually happens. This mapping matters for two reasons: latency (users connect to the nearest location) and bandwidth (allocation happens at the compute-location level).

The capacity concepts to know:

- Bandwidth is allocated per compute location, and Prisma Access distributes it dynamically across the sites and tunnels homed to that compute location based on demand. You size and pay for aggregate bandwidth, not per-tunnel.
- Mobile-user licensing is by user count (or a units model, depending on the license), and mobile-user capacity autoscales. As more users log in to a location, Prisma Access spins up more mobile-user security processing nodes (MU-SPNs) automatically, so you don't pre-provision per-location user capacity.
- Each customer gets a dedicated dataplane, so one tenant's autoscaling event doesn't touch another tenant's capacity.

## The IP allocation gotcha (plan for autoscaling)

This is the most under-planned thing in Prisma Access. Because the service autoscales, its egress IP addresses change. New nodes bring new public IPs, and each location has both active and reserved (standby) IP ranges.

If any SaaS provider, partner, or B2B integration does source-IP allowlisting, you have to allowlist both the active and the reserved public IPs for the relevant locations. Otherwise access breaks the moment an autoscaling event moves users to a node whose IP wasn't on the list. Pull the current IP list from the tenant (it's in the UI and exposed via API) and build the allowlisting process around the fact that the set can grow. Capture "we allowlist Prisma's IPs somewhere" as a design input on day one rather than finding it during the first scale event.

## Mobile Users: design notes

- GlobalProtect is the mainstream agent; the newer Prisma Access Agent is the direction of travel for unified SASE. Pick based on your rollout maturity.
- Users connect to the nearest location. Portal/gateway selection and internal-vs-external gateway logic determine on-net vs off-net behavior.
- HIP checks (disk encryption, AV running, patch level) enforce endpoint posture before granting access. Carry the same HIP discipline over from on-prem GlobalProtect.
- Decide split-tunnel vs. full-tunnel deliberately and document it. It's the same trade-off as on-prem GP and it drives how much traffic you're paying to inspect.

## Remote Networks: design notes

- Branches connect over IPsec from an on-prem termination device: an SD-WAN edge (often Prisma SD-WAN, see the sibling doc), a router, or a firewall.
- ECMP across multiple links to a location adds aggregate throughput and resilience, but if you enable route summarization on an ECMP location you have to enable it on all links to that location or the commit fails.
- Bandwidth comes from the compute-location pool. Size the aggregate, and remember a single remote-network tunnel has its own throughput ceiling, so very large sites may need multiple tunnels or links.

## Gotchas

- No service connection means islands. Mobile users and remote networks can't reach each other or private apps without the routing a service connection provides. Create one even in a mobile-users-only design if private-app access is in scope.
- Service connections don't filter and can't egress to the internet. Don't treat them as an enforcement point.
- Autoscaling IPs break SaaS allowlisting. Allowlist active and reserved IPs for every location in scope, and treat the IP set as something that grows.
- Route summarization on ECMP is all-or-nothing per location. Enable it on every link or the commit errors, and the error text won't obviously point at this.
- Overlapping private address space across branches and data centers is the routing problem you'll actually spend time on. Plan addressing and summarization before onboarding sites.
- Know which management plane the tenant is on. Strata Cloud Manager and Panorama-managed tenants differ in workflow and feature availability, so confirm which one you're on before quoting a procedure.

## References

- Prisma Access Administration: [docs.paloaltonetworks.com/prisma-access/administration](https://docs.paloaltonetworks.com/prisma-access/administration)
- Use a Service Connection to enable access between Mobile Users and Remote Networks: [docs.paloaltonetworks.com/prisma-access/administration/prisma-access-service-connections/use-a-service-connection-to-enable-access-between-mobile-users-and-remote-networks](https://docs.paloaltonetworks.com/prisma-access/administration/prisma-access-service-connections/use-a-service-connection-to-enable-access-between-mobile-users-and-remote-networks)
- Prisma Access Locations: [docs.paloaltonetworks.com/prisma-access/administration/prisma-access-overview/list-of-prisma-access-locations](https://docs.paloaltonetworks.com/prisma-access/administration/prisma-access-overview/list-of-prisma-access-locations)
- Allocate Remote Network Bandwidth: [docs.paloaltonetworks.com/prisma-access/administration/prisma-access-remote-networks/allocate-remote-network-bandwidth](https://docs.paloaltonetworks.com/prisma-access/administration/prisma-access-remote-networks/allocate-remote-network-bandwidth)
- Prisma Access Infrastructure Management: [docs.paloaltonetworks.com/prisma-access/administration/prisma-access-overview/prisma-access-infrastructure-management](https://docs.paloaltonetworks.com/prisma-access/administration/prisma-access-overview/prisma-access-infrastructure-management)
