# Palo Alto NGFW in Azure

Design reference for running Palo Alto Networks firewalls in Azure. Covers the two product choices (VM-Series vs. Cloud NGFW for Azure), the transit/hub architectures, how traffic actually gets steered to the firewall, and the operational gotchas that don't exist on-prem. PAN-OS security design carries over from [palo-alto-architecture.md](palo-alto-architecture.md); what changes in cloud is the plumbing — routing, HA, and scale.

## First decision: VM-Series or Cloud NGFW

This is the fork that determines everything downstream. Pick it before drawing any VNets.

| | VM-Series | Cloud NGFW for Azure |
|---|---|---|
| **Model** | Self-managed NVA. You own the VMs, load balancers, PAN-OS/content updates, HA, and scaling. | Fully managed SaaS-style firewall. Palo Alto owns the infrastructure, PAN-OS, patching, HA, and scaling. |
| **Control** | Full control of PAN-OS version and every dataplane feature. | Abstracted — you get policy and inspection, not the box. |
| **Management** | Panorama or local, same as on-prem. | Azure-native (Rules Stacks) or Panorama-managed via Cloud Services plugin. |
| **vWAN** | Runs in a spoke; needs manual UDRs/static routes to insert. Not natively hub-resident. | Integrates into the vWAN hub with **routing intent** doing the traffic redirection. |
| **Best when** | You need a specific PAN-OS version for compliance, features not yet in Cloud NGFW, or hub IP/subnet-delegation constraints make the managed model impractical. | You want minimal operational overhead and clean vWAN integration, and the feature set covers your needs. |

The honest default in 2025+ for a greenfield Azure security posture is **Cloud NGFW if it fits**, because it removes the operational surface (scaling, HA, upgrades) that consumes most of the effort in a self-managed VM-Series deployment. Reach for VM-Series when a concrete requirement — version pinning, a specific feature, addressing constraints — rules the managed service out.

The rest of this doc is mostly about VM-Series, because that's where the architecture decisions live. Cloud NGFW's whole value proposition is that it makes most of them for you.

## VM-Series: the transit VNet (hub-and-spoke) model

The standard architecture is a **transit (hub) VNet** holding the VM-Series firewalls, with application workloads in **spoke VNets** peered to the hub. All inter-spoke, spoke-to-on-prem, and internet traffic is steered through the firewalls in the hub.

The common design uses a single set of VM-Series firewalls, each with interfaces in the hub's public (untrust), private (trust), and management subnets. Traffic is forced through them with **user-defined routes (UDRs)** on the spoke and gateway subnets pointing at an internal load balancer in front of the firewalls.

Why a load balancer and not a floating-IP HA pair like on-prem: Azure's fabric doesn't do gratuitous-ARP failover, and per-flow symmetry has to be engineered. The two patterns that work:

- **Azure load balancers (sandwich).** A public LB for inbound, an internal LB (ILB) for outbound/east-west, with the firewalls in the backend pool. Scales horizontally and is the classic pattern, but you own the routing, health probes, and session-symmetry design.
- **Azure Gateway Load Balancer (GWLB).** The newer and cleaner option. The firewalls sit in a GWLB backend pool and are inserted **transparently** into the path via service chaining — no per-spoke UDRs and no VNet-peering gymnastics to bend traffic through them. Requires **PAN-OS 10.1.4+ and VM-Series plugin 2.1.4+**. Behind the GWLB the firewall enforces a zone-based policy by mapping VNet-bound traffic to a trust zone and internet-bound traffic to untrust.

Prefer **GWLB** for new builds. It removes the most error-prone part of the classic model (hand-maintained UDRs) and makes horizontal scale a load-balancer concern rather than a routing-table one.

## VM-Series: HA in Azure is not on-prem HA

Do not port the active/passive floating-IP model into Azure. It can be done (with API-driven route-table updates on failover) but it's slow, fragile, and the wrong instinct. In cloud, **scale horizontally behind a load balancer** instead: multiple active firewalls in a backend pool, health-probed, with the LB handling distribution and instance loss. This gives you both availability and capacity from the same design, and it's how GWLB and the sandwich model both expect to be run.

Spread instances across **Availability Zones** so a zone outage doesn't take the whole security tier with it.

## Steering traffic: routing is the hard part

On-prem, the firewall is a cable you can't avoid. In Azure, traffic goes wherever the effective routes say, and the firewall is only in the path if you put it there. Getting this right is the bulk of the design work.

- **UDRs** on spoke subnets (and the gateway subnet, for on-prem-bound traffic) point default and inter-spoke routes at the firewall/ILB front-end IP.
- **Disable BGP route propagation** on route tables where you're forcing traffic to the firewall, or a learned route can bypass your UDR.
- **Symmetry matters.** Both directions of a flow must hit the same firewall instance, or stateful inspection drops the return traffic. GWLB and correctly-configured ILB HA-ports preserve this; hand-rolled multi-firewall UDR designs are where asymmetry bugs live.
- **NAT.** For internet-inbound, DNAT on the firewall (or via the public LB) to the workload. For outbound, SNAT on the untrust interface. Decide early whether internet egress is centralized through the hub or handled another way.

### vWAN

If the environment is built on **Azure Virtual WAN**, VM-Series is awkward — it lives in a spoke and needs manual static routes/UDRs to insert, because vWAN's clean declarative **routing intent** only steers to security solutions deployed *inside* the hub. **Cloud NGFW for Azure** is the intended fit here: it deploys into the vWAN hub and routing intent redirects traffic to it for inspection with no UDR maintenance. If the mandate is vWAN + Palo Alto, that's a strong signal toward Cloud NGFW rather than VM-Series.

## Bootstrap and lifecycle

VM-Series should be **bootstrapped**, not hand-built. A bootstrap package (via an Azure storage account / file share) supplies the initial config, licenses, content, and PAN-OS so an instance comes up production-ready and identical to its peers — which is what makes horizontal scale and instance replacement safe. Combined with an ARM/Terraform/Bicep deployment (Palo Alto publishes reference Terraform modules for the transit and GWLB patterns), the whole security tier becomes reproducible infrastructure rather than pets.

Licensing is either BYOL or PAYG from the Azure Marketplace; PAYG suits autoscaling because capacity you spin up is automatically entitled.

## Gotchas

- **Asymmetric routing is the #1 Azure firewall bug.** Every multi-instance design has to guarantee both directions of a flow land on the same box. Use GWLB or ILB HA-ports; don't improvise it with UDRs.
- **BGP route propagation left enabled** on a route table quietly creates a path around your UDR, so traffic bypasses the firewall and everything "works" — with no inspection.
- **On-prem HA muscle memory.** Floating-IP active/passive is the wrong pattern in Azure. Scale out behind a load balancer instead.
- **GWLB has version floors** — PAN-OS 10.1.4+ and VM-Series plugin 2.1.4+. Check the compatibility matrix before committing the design to it.
- **vWAN + VM-Series is a poor fit.** Routing intent won't steer to an NVA in a spoke. This is a Cloud NGFW use case; forcing VM-Series into it means hand-maintained routing you'll regret.
- **Un-bootstrapped instances can't scale.** If new firewalls aren't identical and production-ready on boot, autoscaling and instance replacement become manual events, defeating the point of running in cloud.
- **Sizing still applies.** Cloud doesn't repeal the enabled-feature throughput math from [palo-alto-architecture.md](palo-alto-architecture.md); pick VM-Series instance sizes against real decryption/throughput needs.

## References

- Set Up the VM-Series Firewall on Azure: [docs.paloaltonetworks.com/vm-series/deployment/public-cloud/set-up-the-vm-series-firewall-on-azure](https://docs.paloaltonetworks.com/vm-series/deployment/public-cloud/set-up-the-vm-series-firewall-on-azure)
- Deploy the VM-Series with the Azure Gateway Load Balancer: [docs.paloaltonetworks.com/vm-series/11-0/vm-series-deployment/set-up-the-vm-series-firewall-on-azure/deploy-the-vm-series-firewall-with-the-azure-gwlb](https://docs.paloaltonetworks.com/vm-series/11-0/vm-series-deployment/set-up-the-vm-series-firewall-on-azure/deploy-the-vm-series-firewall-with-the-azure-gwlb)
- Reference Architecture with Terraform — VM-Series in Azure (transit VNet): [pan.dev/terraform/docs/swfw/azure/vmseries/reference-architectures/vmseries_transit_vnet_common](https://pan.dev/terraform/docs/swfw/azure/vmseries/reference-architectures/vmseries_transit_vnet_common/)
- Cloud NGFW for Azure — introduction: [docs.paloaltonetworks.com/cloud-ngfw-azure/getting-started/introducing-cloud-ngfw-for-azure/cloud-ngfw-for-azure](https://docs.paloaltonetworks.com/cloud-ngfw-azure/getting-started/introducing-cloud-ngfw-for-azure/cloud-ngfw-for-azure)
- Cloud NGFW for Azure Virtual WAN: [docs.paloaltonetworks.com/cloud-ngfw-azure/deployment/cloud-ngfw-for-azure-deployment-architectures/cloud-ngfw-for-azure-virtual-wan](https://docs.paloaltonetworks.com/cloud-ngfw-azure/deployment/cloud-ngfw-for-azure-deployment-architectures/cloud-ngfw-for-azure-virtual-wan)
- Cloud NGFW for Azure vs VM-Series feature comparison: [live.paloaltonetworks.com/t5/cloud-ngfw-for-azure-articles/cloud-ngfw-for-azure-vs-vm-series-firewall-feature-comparison/ta-p/588332](https://live.paloaltonetworks.com/t5/cloud-ngfw-for-azure-articles/cloud-ngfw-for-azure-vs-vm-series-firewall-feature-comparison/ta-p/588332)
