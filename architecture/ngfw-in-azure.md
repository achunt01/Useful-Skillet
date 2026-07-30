# Palo Alto NGFW in Azure

Design reference for running Palo Alto Networks firewalls in Azure. It covers the two product choices (VM-Series vs. Cloud NGFW for Azure), the transit/hub architectures, how traffic actually gets steered to the firewall, and the operational problems that don't exist on-prem. PAN-OS security design carries over from [palo-alto-architecture.md](palo-alto-architecture.md); what changes in cloud is the plumbing, mainly routing, HA, and scale.

## First decision: VM-Series or Cloud NGFW

This choice drives everything downstream, so make it before you draw any VNets.

| | VM-Series | Cloud NGFW for Azure |
|---|---|---|
| Model | Self-managed NVA. You own the VMs, load balancers, PAN-OS/content updates, HA, and scaling. | Managed service. Palo Alto owns the infrastructure, PAN-OS, patching, HA, and scaling. |
| Control | Full control of the PAN-OS version and every dataplane feature. | Abstracted. You get policy and inspection, not the box. |
| Management | Panorama or local, same as on-prem. | Azure-native (Rules Stacks) or Panorama-managed via the Cloud Services plugin. |
| vWAN | Runs in a spoke and needs manual UDRs/static routes to insert. Not hub-resident. | Deploys into the vWAN hub, with routing intent handling the redirection. |
| Best when | You need a specific PAN-OS version for compliance, a feature Cloud NGFW doesn't have yet, or hub IP/subnet-delegation constraints rule out the managed model. | You want low operational overhead and clean vWAN integration, and the feature set covers what you need. |

For a greenfield Azure security posture, the default worth starting from is Cloud NGFW if it fits, because it removes the operational surface (scaling, HA, upgrades) that eats most of the effort in a self-managed VM-Series build. Go with VM-Series when a concrete requirement rules the managed service out: version pinning, a specific feature, or addressing constraints.

The rest of this doc is mostly about VM-Series, because that's where the architecture decisions live. Cloud NGFW's whole point is that it makes most of them for you.

## VM-Series: the transit VNet (hub-and-spoke) model

The standard architecture is a transit (hub) VNet holding the VM-Series firewalls, with application workloads in spoke VNets peered to the hub. All inter-spoke, spoke-to-on-prem, and internet traffic is steered through the firewalls in the hub.

The common design uses one set of VM-Series firewalls, each with interfaces in the hub's public (untrust), private (trust), and management subnets. Traffic is forced through them with user-defined routes (UDRs) on the spoke and gateway subnets pointing at an internal load balancer in front of the firewalls.

The reason there's a load balancer instead of an on-prem-style floating-IP HA pair: Azure's fabric doesn't do gratuitous-ARP failover, and per-flow symmetry has to be engineered. Two patterns work:

- Azure load balancers (the "sandwich"): a public LB for inbound, an internal LB (ILB) for outbound and east-west, with the firewalls in the backend pool. It scales horizontally and is the classic pattern, but you own the routing, health probes, and session-symmetry design.
- Azure Gateway Load Balancer (GWLB): the newer and cleaner option. The firewalls sit in a GWLB backend pool and get inserted into the path transparently through service chaining, with no per-spoke UDRs and no VNet-peering gymnastics to bend traffic through them. It needs PAN-OS 10.1.4 or later and VM-Series plugin 2.1.4 or later. Behind the GWLB the firewall enforces zone-based policy by mapping VNet-bound traffic to a trust zone and internet-bound traffic to untrust.

Prefer GWLB for new builds. It removes the most error-prone part of the classic model (hand-maintained UDRs) and turns horizontal scale into a load-balancer concern rather than a routing-table one.

## VM-Series: HA in Azure is not on-prem HA

Don't port the active/passive floating-IP model into Azure. It can be done, using API-driven route-table updates on failover, but it's slow and fragile and it's the wrong instinct. In cloud, scale horizontally behind a load balancer instead: several active firewalls in a backend pool, health-probed, with the LB handling distribution and instance loss. That gives you availability and capacity from the same design, which is how both GWLB and the sandwich model expect to run.

Spread instances across Availability Zones so a zone outage doesn't take the whole security tier with it.

## Steering traffic: routing is the hard part

On-prem, the firewall is a cable you can't route around. In Azure, traffic goes wherever the effective routes send it, and the firewall is only in the path if you put it there. Getting this right is most of the design work.

- Put UDRs on spoke subnets (and the gateway subnet, for on-prem-bound traffic) pointing default and inter-spoke routes at the firewall/ILB front-end IP.
- Disable BGP route propagation on route tables where you're forcing traffic to the firewall, or a learned route can bypass your UDR.
- Keep flows symmetric. Both directions of a flow have to hit the same firewall instance, or stateful inspection drops the return traffic. GWLB and correctly-configured ILB HA-ports preserve this; hand-rolled multi-firewall UDR designs are where asymmetry bugs come from.
- Handle NAT explicitly. For internet-inbound, DNAT on the firewall (or via the public LB) to the workload. For outbound, SNAT on the untrust interface. Decide early whether internet egress is centralized through the hub or handled another way.

### vWAN

If the environment is built on Azure Virtual WAN, VM-Series is awkward. It lives in a spoke and needs manual static routes and UDRs to insert, because vWAN's routing intent only steers to security solutions deployed inside the hub, not to an NVA in a spoke. Cloud NGFW for Azure is the intended fit here: it deploys into the vWAN hub and routing intent redirects traffic to it for inspection with no UDR maintenance. So if the requirement is vWAN plus Palo Alto, that points toward Cloud NGFW rather than VM-Series. You can force VM-Series into a vWAN design, but you'll be maintaining that routing by hand.

## Bootstrap and lifecycle

VM-Series should be bootstrapped, not hand-built. A bootstrap package (via an Azure storage account or file share) supplies the initial config, licenses, content, and PAN-OS, so an instance comes up production-ready and identical to its peers. That's what makes horizontal scale and instance replacement safe. Combined with an ARM/Terraform/Bicep deployment (Palo Alto publishes reference Terraform modules for the transit and GWLB patterns), the whole security tier becomes reproducible infrastructure instead of hand-configured servers.

Licensing is BYOL or PAYG from the Azure Marketplace. PAYG suits autoscaling, since capacity you spin up is entitled automatically.

## Gotchas

- Asymmetric routing is the most common Azure firewall bug. Every multi-instance design has to guarantee both directions of a flow land on the same box. Use GWLB or ILB HA-ports rather than improvising it with UDRs.
- BGP route propagation left enabled on a route table quietly creates a path around your UDR, so traffic bypasses the firewall and everything looks like it's working, with no inspection happening.
- On-prem HA habits don't transfer. Floating-IP active/passive is the wrong pattern in Azure. Scale out behind a load balancer instead.
- GWLB has version floors: PAN-OS 10.1.4+ and VM-Series plugin 2.1.4+. Check the compatibility matrix before you commit the design to it.
- vWAN and VM-Series don't fit well together. Routing intent won't steer to an NVA in a spoke, so this is a Cloud NGFW case; forcing VM-Series in means routing you maintain by hand.
- Un-bootstrapped instances can't scale. If new firewalls aren't identical and production-ready on boot, autoscaling and instance replacement become manual events, which defeats the point of running in cloud.
- Sizing still applies. Cloud doesn't repeal the enabled-feature throughput math from [palo-alto-architecture.md](palo-alto-architecture.md); pick VM-Series instance sizes against real decryption and throughput needs.

## References

- Set Up the VM-Series Firewall on Azure: [docs.paloaltonetworks.com/vm-series/deployment/public-cloud/set-up-the-vm-series-firewall-on-azure](https://docs.paloaltonetworks.com/vm-series/deployment/public-cloud/set-up-the-vm-series-firewall-on-azure)
- Deploy the VM-Series with the Azure Gateway Load Balancer: [docs.paloaltonetworks.com/vm-series/11-0/vm-series-deployment/set-up-the-vm-series-firewall-on-azure/deploy-the-vm-series-firewall-with-the-azure-gwlb](https://docs.paloaltonetworks.com/vm-series/11-0/vm-series-deployment/set-up-the-vm-series-firewall-on-azure/deploy-the-vm-series-firewall-with-the-azure-gwlb)
- Reference Architecture with Terraform, VM-Series in Azure (transit VNet): [pan.dev/terraform/docs/swfw/azure/vmseries/reference-architectures/vmseries_transit_vnet_common](https://pan.dev/terraform/docs/swfw/azure/vmseries/reference-architectures/vmseries_transit_vnet_common/)
- Cloud NGFW for Azure, introduction: [docs.paloaltonetworks.com/cloud-ngfw-azure/getting-started/introducing-cloud-ngfw-for-azure/cloud-ngfw-for-azure](https://docs.paloaltonetworks.com/cloud-ngfw-azure/getting-started/introducing-cloud-ngfw-for-azure/cloud-ngfw-for-azure)
- Cloud NGFW for Azure Virtual WAN: [docs.paloaltonetworks.com/cloud-ngfw-azure/deployment/cloud-ngfw-for-azure-deployment-architectures/cloud-ngfw-for-azure-virtual-wan](https://docs.paloaltonetworks.com/cloud-ngfw-azure/deployment/cloud-ngfw-for-azure-deployment-architectures/cloud-ngfw-for-azure-virtual-wan)
- Cloud NGFW for Azure vs VM-Series feature comparison: [live.paloaltonetworks.com/t5/cloud-ngfw-for-azure-articles/cloud-ngfw-for-azure-vs-vm-series-firewall-feature-comparison/ta-p/588332](https://live.paloaltonetworks.com/t5/cloud-ngfw-for-azure-articles/cloud-ngfw-for-azure-vs-vm-series-firewall-feature-comparison/ta-p/588332)
