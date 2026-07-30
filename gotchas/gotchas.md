# Gotchas

Things that broke, misconfigs that weren't obvious, and fixes that aren't well documented elsewhere. Add to this as they come up.

## Zone protection / DoS

- Setting flood thresholds without first baselining CPS per zone causes legitimate traffic to get dropped during normal peak hours. Always pull `show running resource-monitor` or Panorama Health Monitor data before touching Activate/Maximum thresholds.
- Zone Protection profiles are applied on the ingress interface only. Applying a profile to the wrong zone (e.g., trust instead of untrust) gives a false sense of protection with no actual effect on inbound attacks.

## Security profiles

- Default Anti-Spyware/Vulnerability Protection severity actions for low/informational events are often left as "allow" rather than "default" out of the box. BPA flags this, but it's easy to miss on manual review since the profile still looks "enabled."
- WildFire file forwarding requires the right file type/size settings and an active subscription. Profile can look correctly configured but silently fail to forward if the subscription lapsed or size limits are misconfigured.

## Decryption

- SSL decryption exclusions accumulate over time and rarely get cleaned up. Review the exclusion list on every engagement. Old exclusions for decommissioned apps often stay in place indefinitely.
- Minimum TLS version changes can break legacy internal apps that only support TLS 1.0/1.1. Test before rolling out broadly, especially on internal decryption policies.

## Panorama / device groups

- Objects with the same name in Shared vs. a device group silently override each other depending on inheritance order. Naming collisions here are a common source of "why did this rule stop matching" tickets.
- Template stack ordering matters. A setting defined in a lower-priority template can silently overwrite one meant to come from a higher-priority template if the stack order isn't what you assumed.

## GlobalProtect

- Self-signed certs on the portal/gateway work fine for testing but cause client-side trust warnings and app connection failures at scale. Always plan for a proper cert.
- Split-tunnel configuration mismatches between portal and gateway configs cause inconsistent user experience depending on which gateway they connect to.

## SD-WAN

- SD-WAN plugin version compatibility with the base PAN-OS version isn't always obvious from the release notes. Confirm plugin/PAN-OS compatibility before upgrading either one independently.

## HA

- HA1/HA2 sharing a data-plane interface works until there's a traffic spike, at which point failover detection gets unreliable. Dedicated interfaces for HA links are worth the extra port even in smaller deployments.
