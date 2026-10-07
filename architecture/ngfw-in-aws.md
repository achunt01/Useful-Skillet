# Palo Alto NGFW in AWS

Design reference for deploying Palo Alto Networks firewalls in AWS. It covers the product choice (VM-Series or Cloud NGFW for AWS), common VPC and Transit Gateway patterns, traffic steering, availability, and lifecycle. The security policy principles in [palo-alto-architecture.md](palo-alto-architecture.md) still apply; AWS changes the network plumbing, especially routing, appliance insertion, and scale.

## First decision: VM-Series or Cloud NGFW

Choose the operating model before designing the VPC topology.

| | VM-Series | Cloud NGFW for AWS |
|---|---|---|
| Model | Self-managed firewall instances. You operate PAN-OS, licensing, scaling, and deployment automation. | Palo Alto Networks-managed firewall service integrated with AWS. |
| Control | Direct control of PAN-OS and the VM-Series feature set. | Firewall infrastructure is abstracted; policy and integration follow the service's supported model. |
| Management | Local management or Panorama, depending on the design. | Cloud NGFW service integrations and supported Panorama options; verify current capabilities for the deployment. |
| Best when | You need VM-Series flexibility, control over software and lifecycle, or an architecture built around your own appliance fleet. | You want a managed service and its feature set, regional availability, and integration model meet your requirements. |

Compare supported features, regions, licensing, throughput, and management integrations against current vendor documentation before choosing. The managed-service boundary affects which firewall settings and lifecycle tasks you control.

## VM-Series: choose a traffic-insertion pattern

Two common patterns address different network needs. Avoid combining them without explicitly designing the route tables and return path.

### Gateway Load Balancer endpoints

Use AWS Gateway Load Balancer (GWLB) with Gateway Load Balancer endpoints (GWLBe) to insert a fleet of VM-Series appliances transparently into VPC traffic paths. Deploy the GWLB and firewall targets in the inspection VPC; create endpoints in the workload VPCs and route selected traffic to those endpoints.

This pattern suits centralized inspection for internet egress, ingress, or VPC-to-VPC flows when routes are deliberately designed through the endpoint. GWLB distributes flows across healthy targets and uses GENEVE encapsulation between the load balancer and appliances. Design for flow stickiness, health checks, Availability Zone placement, and the return route.

### Transit Gateway centralized inspection

Use AWS Transit Gateway (TGW) to connect workload VPCs and on-premises networks, with route tables directing selected traffic through a dedicated inspection VPC. This provides a central routing point for north-south and east-west traffic across multiple VPCs.

Plan TGW attachment associations, route propagation, and static routes so that only intended traffic enters the inspection VPC and the return path traverses the same stateful firewall. For an appliance VPC handling traffic through TGW, evaluate appliance mode on the inspection VPC attachment to preserve Availability Zone affinity for stateful flows.

## Routing and traffic symmetry

AWS does not put an appliance into a path just because it is attached to a VPC or TGW. Route tables determine whether traffic is inspected.

- Draw the forward and return paths for each traffic class: internet ingress, egress, inter-VPC, and on-premises connectivity.
- Use VPC subnet route tables and TGW route tables together; check effective routes at both layers.
- Preserve stateful-flow symmetry. A flow must return through the same logical inspection path and compatible firewall state.
- For VM-Series forwarding traffic, configure AWS network-interface and instance settings as required by the chosen deployment guide, including disabling source/destination checks where the appliance is routing transit traffic.
- Keep management access separate from data-plane paths and restrict it to approved management sources.
- Design NAT deliberately. Decide whether source NAT or destination NAT occurs on the firewall, an AWS load-balancing component, or the workload, and validate what addresses the firewall sees.
- Plan route changes and failover behavior as part of recovery testing, not only during initial deployment.

## Availability and scale

For GWLB-based designs, deploy firewall targets across Availability Zones and use the load balancer's health state and flow stickiness as intended by the design. For TGW-based appliance insertion, verify zone affinity and failover behavior for the inspection VPC attachment and firewall instances.

Do not assume that a traditional on-premises active/passive floating-IP design maps directly to AWS. Choose an AWS-supported deployment pattern and validate what happens to established sessions when a target or an Availability Zone fails. Size instances for enabled features, decryption ratio, throughput, and connection rates, then test scaling behavior under load.

## Bootstrap and lifecycle

Automate network interfaces, routes, firewall instances, and policy initialization with a repeatable deployment process such as CloudFormation or Terraform. Bootstrap VM-Series instances so replacements receive consistent configuration, licensing, and content settings instead of requiring manual setup.

Decide how Panorama, AWS integrations, and configuration automation fit together before production deployment. Document ownership for PAN-OS and content upgrades, image selection, licensing changes, backups, and recovery. Revalidate vendor and AWS compatibility before changing instance types, AMIs, plugins, or PAN-OS releases.

## Gotchas

- A TGW attachment alone does not force traffic through a firewall. TGW and VPC routes both need to direct the relevant forward and return paths through inspection.
- Asymmetric routing breaks stateful inspection. Validate AZ affinity, route propagation, and return paths, especially across multiple attachments and zones.
- GWLB is transparent insertion, not a substitute for routing design. Endpoints must be placed and referenced by route tables for the traffic that should be inspected.
- Source/destination checks can prevent a VM-Series instance from forwarding transit traffic when they have not been disabled as required.
- Health checks and target registration do not prove that application traffic is inspected. Test with real flows and confirm firewall logs and counters.
- An unbootstrapped replacement instance can turn an autoscaling or recovery event into a manual firewall build.
- AWS and firewall service capabilities, regional availability, and licensing change over time. Confirm current requirements and supported architectures before implementation.

## References

- [Set Up the VM-Series Firewall on AWS](https://docs.paloaltonetworks.com/vm-series/deployment/public-cloud/set-up-the-vm-series-firewall-on-aws)
- [AWS Transit Gateway](https://docs.aws.amazon.com/vpc/latest/tgw/what-is-transit-gateway.html)
- [AWS Gateway Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/gateway/introduction.html)
- [Access virtual appliances through AWS PrivateLink](https://docs.aws.amazon.com/vpc/latest/privatelink/vpce-gateway-load-balancer.html)
- [Cloud NGFW for AWS](https://www.paloaltonetworks.com/network-security/cloud-ngfw/aws)
- [Palo Alto Networks Terraform provider and examples](https://pan.dev/terraform/)
