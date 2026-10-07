# Prisma Access

Architecture and design reference for Prisma Access, Palo Alto's cloud-delivered SASE. This covers the general model: the three connection types, how they route to each other, capacity and bandwidth, and the design decisions that cause trouble later. Prisma SD-WAN, the branch WAN-edge product that often feeds remote networks, has its own doc: [prisma-sd-wan.md](prisma-sd-wan.md).

## Naming: Prisma SASE vs Prisma Access

Prisma SASE is the portfolio name. Prisma Access is the security service edge (SSE) product inside it: the cloud-delivered NGFW stack. Prisma SD-WAN is the networking half. When a client says "we bought Prisma," find out which of the two they mean, because the architecture, licensing, and onboarding are different.

## What it is, and what it replaces

Prisma Access delivers the NGFW security stack (App-ID, Threat Prevention, URL Filtering, WildFire, DNS Security, DLP, and the CASB features) from Palo Alto's cloud instead of from a box you rack. It's the security half of SASE; pair it with SD-WAN for the network half. What it most directly replaces is backhaul-to-the-datacenter remote-access VPN. Instead of hauling every remote user and branch back to a central firewall, users and sites connect to the nearest cloud location and get consistent policy there.

Management is through Strata Cloud Manager (the current cloud UI) or, in older Panorama-managed tenants, the Panorama Cloud Services plugin. New deployments land in Strata Cloud Manager. See [Management plane](#management-plane-strata-cloud-manager-vs-panorama) below.

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

## Mobile users: connection methods

Four ways to onboard users, in rough order of security completeness:

- GlobalProtect agent. The mainstream option. Full tunnel (IPsec/SSL) to the nearest location, all ports and protocols inspected, HIP checks enforced. The only method that gives you the complete NGFW stack on the endpoint.
- Prisma Access Agent. The newer unified SASE agent and the direction of travel. Same connectivity as GlobalProtect with tighter integration into the SASE fabric. Pick based on rollout maturity and what the client already runs.
- Clientless VPN. Browser-based access for unmanaged devices and contractors. No agent install, SAML auth, limited to web apps. Useful as a stopgap, not a strategy.
- Explicit Proxy. PAC-file or proxy-mode agent configuration steering HTTP/HTTPS traffic to Prisma Access for SWG inspection. No tunnel, no all-port inspection. The migration path off legacy proxy estates.

Design notes:

- Users connect to the nearest location. Portal/gateway selection and internal-vs-external gateway logic determine on-net vs off-net behavior.
- HIP checks (disk encryption, AV running, patch level) enforce endpoint posture before granting access. Carry the same HIP discipline over from on-prem GlobalProtect.
- Decide split-tunnel vs. full-tunnel deliberately and document it. It's the same trade-off as on-prem GP and it drives how much traffic you're paying to inspect.

### Explicit Proxy

Explicit Proxy exists so shops coming off a legacy proxy estate can move web traffic to Prisma Access without re-architecting the network. The browser (via PAC file) or the agent in proxy mode sends HTTP/HTTPS to Prisma Access, which applies URL Filtering, Threat Prevention, and DLP to web traffic.

What to know before recommending it:

- It only covers HTTP/HTTPS. Non-web protocols bypass it entirely, so it is not a replacement for the agent where full inspection is needed.
- PAC file distribution is your problem. Push via GPO, MDM, or endpoint management; there is no portal-driven PAC push equivalent to gateway config.
- Plan how user identification works for policy, especially on shared machines and VDIs.
- Treat it as a phase, not a destination. The documented PANW position is that customers start on Explicit Proxy and move to the agent for all-port, all-protocol protection.

## Prisma Access Browser

Chromium-based enterprise browser, formerly Talon (acquired late 2023). PANW has started marketing it as Prisma Browser; the Prisma Access Browser name still appears in datasheets, so expect both names in the wild.

What it does: moves the control point into the browser itself. DLP at the point of use (copy/paste, download, upload, screenshot, watermarking), phishing and malicious-extension protection, and visibility into web/SaaS usage on devices the org does not manage. Primary use cases are BYOD, contractors, and unmanaged endpoints where the GlobalProtect agent cannot be installed.

Design notes:

- It complements the agent; it does not replace it. Browser-layer controls cover web/SaaS; the agent covers everything else.
- For GenAI governance it is the pragmatic answer on unmanaged devices: paste limits and blocking risky AI sites where network-layer DLP cannot see inside a personal browser session.
- Evaluate it against VDI and remote browser isolation for the contractor-access use case. It is cheaper than VDI and more usable than full isolation for most SaaS workflows.

## ZTNA and private app access

Prisma Access includes ZTNA for private apps: application-level access without a full network tunnel. Private apps are reached through service connections, which is another reason the "create at least one service connection" guidance matters.

Design notes:

- ZTNA policies are app-centric, not network-centric. Define the app, the users, and the posture; there is no flat network access to segment afterward.
- The on-prem side needs a reachable path for the app. Plan this alongside the service connection design, not after.
- ZTNA does not replace the agent for users who need broad private-network access. Use ZTNA for specific apps, the agent for full access.

## Remote Networks: design notes

- Branches connect over IPsec from an on-prem termination device: an SD-WAN edge (often Prisma SD-WAN, see the sibling doc), a router, or a firewall.
- ECMP across multiple links to a location adds aggregate throughput and resilience, but if you enable route summarization on an ECMP location you have to enable it on all links to that location or the commit fails.
- Bandwidth comes from the compute-location pool. Size the aggregate, and remember a single remote-network tunnel has its own throughput ceiling, so very large sites may need multiple tunnels or links.

## Management plane: Strata Cloud Manager vs Panorama

New tenants land in Strata Cloud Manager (SCM). Older tenants may still run on the Panorama Cloud Services plugin. One tenant uses one management plane; they are not run simultaneously for the same tenant.

What differs in practice:

- SCM is cloud-native: continuous best-practice assessments, ML-based config optimization, Command Center health views, API-first workflows. No plugin upgrades to track.
- Panorama-managed tenants need the Cloud Services plugin at a compatible version. Check plugin/PAN-OS compatibility before touching either side.
- Migration from Panorama to SCM is one-way. After migrating you cannot go back. The migration does not support every feature, so check the unsupported list against the tenant before starting. Data Filtering, FedRAMP, IoT Security, multitenant deployments, SSH Proxy, and separate portal/gateway auth have all been called out as unsupported at various points.
- SCM still provides monitoring visibility into Panorama-managed tenants even when Panorama owns the config.

Confirm which plane the tenant is on before quoting any procedure. Feature availability and click paths differ.

## Licensing and bandwidth

Prisma Access licensing meters:

- Per user, per year: mobile users. Covers GlobalProtect, clientless, and Explicit Proxy connectivity.
- Per site, per year: SASE branches. Includes the SD-WAN branch subscription with built-in redundancy and branch-to-branch connectivity.
- Per bandwidth (Mbps), per year: shared bandwidth pool across branches.

Editions bundle different feature sets (SWG, ZTNA, Enterprise with the secure browser and inline SaaS security). Edition names have shifted over time; confirm against the current licensing guide rather than quoting from memory.

Bandwidth notes:

- Bandwidth is allocated per compute location and shared dynamically across the sites homed to it. You size and pay for aggregate bandwidth, not per-tunnel.
- Starting with Prisma Access 6.0, remote networks can also be onboarded site-based: pick a site type from Very Small (25 Mbps) up to X-Large (2.5 Gbps), with 10 percent oversubscription allowed. Site-based and aggregate models have different planning math; know which one the tenant uses.
- Every edition includes a data transfer allowance (250 GB per year per unit, averaged across the environment). Heavy-egress designs should check this rather than assuming unlimited transfer.
- Location editions: Local (up to 5 locations) vs Worldwide (100+ locations), with minimum unit counts. A client with users on three continents on a Local edition is a design problem, not a config problem.
- Mobile-user capacity autoscales per location (MU-SPNs spin up with demand) and each customer gets a dedicated dataplane, so one tenant's scaling event does not affect another's.

## Clean Pipe

Clean Pipe is Prisma Access for service providers, MSSPs, and telcos that manage IT infrastructure for other organizations. The provider routes tenant traffic over a Partner Interconnect to a tenant-dedicated Clean Pipe instance, which applies security policy and egresses to the internet.

What to know:

- Separate Clean Pipe license, and tenants run in multi-tenant mode. An API exists for onboarding tenants quickly.
- QoS policies can be set per tenant (e.g., prioritize O365 or Windows Update traffic).
- This is a provider product. If the client is an enterprise, Clean Pipe is not their SKU; if the client is an MSSP, it probably is.

## QoS and user experience

- QoS policies apply to remote network traffic. Define what gets QoS treatment and the classes of service (priority, DSCP- or zone-based). Use it for voice, video, and business-critical SaaS, not as a substitute for enough bandwidth.
- ADEM (Autonomous Digital Experience Management) monitors end-user experience and can flag or remediate network problems. It is the first place to look when the complaint is "Prisma is slow" and policy is not the cause.
- App Acceleration exists for branch-to-cloud SaaS performance. Evaluate it where the complaint is latency to specific SaaS apps, not general throughput.

## Migration notes: from traditional VPN or on-prem firewall

- Inventory first: user count, concurrent VPN sessions today, branch bandwidth, apps that depend on source-IP allowlisting, internal DNS dependencies.
- Run parallel before cutover. Onboard a pilot group to Prisma Access while the existing VPN stays up; compare policy hits and user experience before migrating the fleet.
- Do not lift-and-shift an any-any VPN rulebase into Prisma Access. PAN-OS Policy Optimizer helps convert port-based rules to App-ID. The assessment will flag the any-any rule and the client will ask why they paid for App-ID.
- Decide internal vs external DNS per connection type early. Mobile users need internal DNS for private apps; get the DNS proxy and internal domain list right before users report resolution failures.
- Replicate the on-prem GlobalProtect HIP posture before cutover, or users who passed on-prem checks will fail cloud checks on day one.
- Decommission old concentrators only after Prisma Access traffic and logs show the full user population moved. Stragglers on the old VPN become the shadow IT the project was supposed to eliminate.

## The IP allocation gotcha (plan for autoscaling)

This is the most under-planned thing in Prisma Access. Because the service autoscales, its egress IP addresses change. New nodes bring new public IPs, and each location has both active and reserved (standby) IP ranges.

If any SaaS provider, partner, or B2B integration does source-IP allowlisting, you have to allowlist both the active and the reserved public IPs for the relevant locations. Otherwise access breaks the moment an autoscaling event moves users to a node whose IP wasn't on the list. Pull the current IP list from the tenant (it's in the UI and exposed via API) and build the allowlisting process around the fact that the set can grow. Capture "we allowlist Prisma's IPs somewhere" as a design input on day one rather than finding it during the first scale event.

## Gotchas

- No service connection means islands. Mobile users and remote networks can't reach each other or private apps without the routing a service connection provides. Create one even in a mobile-users-only design if private-app access is in scope.
- Service connections don't filter and can't egress to the internet. Don't treat them as an enforcement point.
- Autoscaling IPs break SaaS allowlisting. Allowlist active and reserved IPs for every location in scope, and treat the IP set as something that grows.
- Route summarization on ECMP is all-or-nothing per location. Enable it on every link or the commit errors, and the error text won't obviously point at this.
- Overlapping private address space across branches and data centers is the routing problem you'll actually spend time on. Plan addressing and summarization before onboarding sites.
- Know which management plane the tenant is on. Strata Cloud Manager and Panorama-managed tenants differ in workflow and feature availability, so confirm which one you're on before quoting a procedure.
- Explicit Proxy PAC files are distributed by you, not by the portal. If the PAC never reaches the endpoint, traffic silently bypasses Prisma Access with no error to chase.
- Migrating Panorama-managed Prisma Access to Strata Cloud Manager is one-way. Check the unsupported-feature list against the tenant before starting; there is no rollback.
- Site-based remote network onboarding (6.0+) and aggregate bandwidth licensing have different capacity math. Confirm which model the tenant uses before sizing anything.
- A tenant on Local edition (5 locations max) with users outside those locations will hairpin or fail. Check the edition against actual user geography.
- Edition names shift over time. Confirm the current licensing guide before quoting edition contents to a client.

## References

- Prisma Access Administration: [docs.paloaltonetworks.com/prisma-access/administration](https://docs.paloaltonetworks.com/prisma-access/administration)
- Use a Service Connection to enable access between Mobile Users and Remote Networks: [docs.paloaltonetworks.com/prisma-access/administration/prisma-access-service-connections/use-a-service-connection-to-enable-access-between-mobile-users-and-remote-networks](https://docs.paloaltonetworks.com/prisma-access/administration/prisma-access-service-connections/use-a-service-connection-to-enable-access-between-mobile-users-and-remote-networks)
- Prisma Access Locations: [docs.paloaltonetworks.com/prisma-access/administration/prisma-access-overview/list-of-prisma-access-locations](https://docs.paloaltonetworks.com/prisma-access/administration/prisma-access-overview/list-of-prisma-access-locations)
- Allocate Remote Network Bandwidth: [docs.paloaltonetworks.com/prisma-access/administration/prisma-access-remote-networks/allocate-remote-network-bandwidth](https://docs.paloaltonetworks.com/prisma-access/administration/prisma-access-remote-networks/allocate-remote-network-bandwidth)
- Prisma Access Infrastructure Management: [docs.paloaltonetworks.com/prisma-access/administration/prisma-access-overview/prisma-access-infrastructure-management](https://docs.paloaltonetworks.com/prisma-access/administration/prisma-access-overview/prisma-access-infrastructure-management)
- Migrate Prisma Access from Panorama to Strata Cloud Manager: [docs.paloaltonetworks.com/prisma-access/administration/prisma-access-overview/migrate-prisma-access-from-panorama-to-strata-cloud-manager](https://docs.paloaltonetworks.com/prisma-access/administration/prisma-access-overview/migrate-prisma-access-from-panorama-to-strata-cloud-manager)
- Onboard a site-based remote network: [docs.paloaltonetworks.com/prisma-access/activation-and-onboarding/onboard-prisma-access/onboard-a-site-based-remote-network](https://docs.paloaltonetworks.com/prisma-access/activation-and-onboarding/onboard-prisma-access/onboard-a-site-based-remote-network)
