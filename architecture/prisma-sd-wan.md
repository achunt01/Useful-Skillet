# Prisma SD-WAN

Architecture reference for Prisma SD-WAN — the cloud-managed, application-defined WAN-edge product (formerly CloudGenix). This is a **different product** from the PAN-OS SD-WAN plugin covered in [palo-sdwan-reference.md](../baselines/palo-sdwan-reference.md); read the next section before assuming the two are interchangeable, because they aren't.

## Prisma SD-WAN vs. PAN-OS SD-WAN — don't conflate them

Both are "Palo Alto SD-WAN," and that naming collision causes real design mistakes. They're separate products with separate hardware, control planes, and philosophies.

| | **Prisma SD-WAN** (this doc) | **PAN-OS SD-WAN plugin** ([other doc](../baselines/palo-sdwan-reference.md)) |
|---|---|---|
| Origin | CloudGenix acquisition | Native PAN-OS feature |
| Edge device | **ION** appliances (physical/virtual) | Existing PAN-OS NGFWs |
| Control plane | Prisma SD-WAN **Controller** (multitenant cloud) | **Panorama** |
| Model | App-defined fabric, "zero-routing," autonomous path selection | Link tags + path-quality/traffic-distribution profiles on firewalls |
| Best when | You want a dedicated, app-aware SD-WAN fabric and are building toward SASE with Prisma Access | You already run PAN-OS firewalls at branches and want SD-WAN as a feature of them |

If a design already standardizes on PAN-OS firewalls at the branch and just needs path steering, the plugin is the lighter answer. If the goal is a purpose-built, application-SLA-driven WAN that plugs cleanly into Prisma Access for SASE, Prisma SD-WAN is the product. Choosing by name rather than by these criteria is the mistake.

## Components

Prisma SD-WAN has three moving parts:

- **ION devices** ("Instant-On Network") — the physical or virtual appliances at the WAN edge (branches and data centers). Models span the **ION 1000 / 2000 / 3000 / 5000 / 7000 / 9000** range plus virtual IONs, sized by branch throughput and port needs. They terminate the WAN links and do the per-flow work.
- **Prisma SD-WAN Controller** — the multitenant cloud control/management plane (formerly the CloudGenix Controller). All configuration, policy, and monitoring happen here; there's no per-device CLI-driven build. This is what eliminates touching each branch box individually.
- **CloudBlades** — pre-built integrations that automate insertion of third-party services (cloud security, other SASE components) into the fabric.

## The app-defined fabric

The defining idea. Instead of routing packets and hoping application experience follows, an ION in **Control mode** takes policy-based action on **every flow**: it identifies the application, continuously measures KPIs (latency, jitter, packet loss, plus app-specific measures like MOS for voice and transaction time for web apps), and steers each flow onto the path that meets that application's SLA — this is the "**zero-routing**" architecture, where you express application intent and SLA rather than hand-crafting routes and PBR.

Practical consequences:

- Policy is written in terms of **applications and SLA**, not prefixes and next-hops. You define what "good" means for an app; the fabric enforces and alerts on it.
- The ION measures continuously and re-selects paths in real time, so brownouts (a link that's up but degraded) trigger failover the same as hard failures — the case traditional routing metrics miss.
- Because it's flow- and app-aware from the edge, the telemetry it reports up to the Controller is application-experience data, not just interface counters — which is most of the operational value.

### Analytics / measurement even without full control

An ION can be deployed in a mode where it sits in the path and **examines and measures** application traffic (building the app-experience baseline) before you hand it full control of path selection. Useful for a low-risk insertion: prove the visibility and the measurements first, then enable Control mode to let it act on what it sees.

## Deployment shape

- **Branches** get one or more IONs terminating their WAN links (broadband, MPLS, LTE, etc.). Dual-link branches are the common case; the fabric load-shares and fails over across links by application SLA.
- **Data centers / hubs** get IONs (often larger models like ION 7000/9000) to anchor the fabric and reach private applications.
- **Zero-touch provisioning:** IONs are "Instant-On" — ship to site, connect, and they call home to the Controller for config, so branch turn-up doesn't require a skilled engineer on-site.
- **Integration with Prisma Access:** the common SASE pattern is Prisma SD-WAN for the network fabric feeding **remote-network** connections into Prisma Access for cloud-delivered security. The SD-WAN edge is the IPsec termination device referenced in [prisma-access.md](prisma-access.md).

## Design notes

- **Start in measurement, then take control.** Let the fabric baseline real application behavior before enabling aggressive path-selection policy, for the same reason you deploy path-quality thresholds loose first on the PAN-OS plugin — day-one tight policy causes flapping that looks like a product problem and is actually a tuning problem.
- **Model the app-SLA policy on real applications**, not a generic template. The product's value is app-awareness; a flat "all traffic, best path" policy leaves most of it on the table.
- **Size IONs to branch throughput and port count**, with headroom, the same discipline as sizing an NGFW — the app-flow processing is real work.
- **Decide the security model explicitly.** Prisma SD-WAN does WAN edge and some security functions, but the SASE story pairs it with Prisma Access for the full NGFW stack. Know where inspection happens for each traffic class (local breakout vs. through Prisma Access).

## Gotchas

- **The naming collision is the biggest trap.** "Palo Alto SD-WAN" means two products. Confirm which one a requirement, a doc, or a colleague is talking about before designing against it.
- **Zero-routing is a mindset shift.** Engineers used to PBR/route-metrics will try to force explicit routing onto an app-defined fabric and fight the product. Express intent and SLA; let the fabric route.
- **Controller is cloud and multitenant.** Management assumes connectivity to the Prisma SD-WAN cloud; factor that into the branch's out-of-band/management story.
- **Generic policy wastes the product.** If you're not writing app- and SLA-specific policy, you've bought an expensive link-bonding box.
- **Plan the Prisma Access hand-off early** if SASE is the goal — the remote-network integration, bandwidth, and IP allowlisting concerns from [prisma-access.md](prisma-access.md) all apply at the seam.

## References

- Prisma SD-WAN documentation: [docs.paloaltonetworks.com/prisma-sd-wan](https://docs.paloaltonetworks.com/prisma-sd-wan)
- ION device documentation: [docs.paloaltonetworks.com/prisma-sd-wan/ion-devices](https://docs.paloaltonetworks.com/prisma-sd-wan)
- Prisma SD-WAN best practices: [live.paloaltonetworks.com/t5/prisma-sd-wan-articles/prisma-sdwan-best-practices/ta-p/587064](https://live.paloaltonetworks.com/t5/prisma-sd-wan-articles/prisma-sdwan-best-practices/ta-p/587064)
