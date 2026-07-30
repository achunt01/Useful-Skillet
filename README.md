# useful-skillet

A working collection of Palo Alto Networks and network security engineering reference material: architecture design guides, hardening baselines, assessment methodology, upgrade runbooks, troubleshooting notes, and gotchas pulled from real client engagements and lab work.

Palo Alto focused — NGFW, Panorama, Prisma Access, and Prisma SD-WAN — with a few adjacent networking platforms (FortiGate, Meraki) kept alongside because the work is rarely single-vendor. The name nods to Iron Skillet.

Not an official Palo Alto Networks or Iron Skillet repo.

## Contents

### Architecture
Senior-level design references — the decisions that are expensive to reverse, and the reasoning behind them.

| Doc | Covers |
|---|---|
| [architecture/palo-alto-architecture.md](architecture/palo-alto-architecture.md) | Architecting Palo Alto NGFW — sizing, deployment mode, HA model, Panorama hierarchy, segmentation and policy model |
| [architecture/ngfw-in-azure.md](architecture/ngfw-in-azure.md) | Palo Alto NGFW in Azure — VM-Series vs. Cloud NGFW, transit VNet, Gateway Load Balancer, routing/HA, vWAN, bootstrap |
| [architecture/prisma-access.md](architecture/prisma-access.md) | Prisma Access (SASE) — mobile users, remote networks, service connections, compute locations, bandwidth, autoscaling IPs |
| [architecture/prisma-sd-wan.md](architecture/prisma-sd-wan.md) | Prisma SD-WAN (ION + Controller) — app-defined fabric, zero-routing, deployment shape, Prisma Access integration |

### Baselines
| Doc | Covers |
|---|---|
| [baselines/security-baseline.md](baselines/security-baseline.md) | Palo Alto NGFW/Panorama baseline reference — Iron Skillet, BPA, CIS Benchmarks, PAN-OS best practices |
| [baselines/palo-sdwan-reference.md](baselines/palo-sdwan-reference.md) | PAN-OS SD-WAN **plugin** — object model, hub-and-spoke design, Auto VPN, path quality tuning (distinct from Prisma SD-WAN) |
| [baselines/fortigate-hardening.md](baselines/fortigate-hardening.md) | FortiGate/FortiOS hardening checklist, CIS-aligned |
| [baselines/meraki-hardening.md](baselines/meraki-hardening.md) | Meraki Dashboard and device hardening, MX/MR/MS |

### Upgrades
| Doc | Covers |
|---|---|
| [upgrades/pan-os-cli-upgrade.md](upgrades/pan-os-cli-upgrade.md) | PAN-OS upgrades via CLI — command reference, upgrade-path rule, standalone and active/passive HA runbooks (11.1) |

### Assessments
| Doc | Covers |
|---|---|
| [assessments/bpa-review-checklist.md](assessments/bpa-review-checklist.md) | Running a Palo Alto BPA / health check engagement end to end |

### Gotchas
| Doc | Covers |
|---|---|
| [gotchas/gotchas.md](gotchas/gotchas.md) | Palo Alto quirks and misconfigs that aren't obvious |
| [gotchas/troubleshooting-methodology.md](gotchas/troubleshooting-methodology.md) | Vendor-agnostic approach to intermittent issues, log correlation, PCAP analysis |

### Configs
| Doc | Covers |
|---|---|
| [configs/zone-protection-cli.md](configs/zone-protection-cli.md) | PAN-OS zone protection CLI snippets |

### Reference
| File | Covers |
|---|---|
| [reference/Palo Alto Security Baseline Reference.docx](reference/Palo%20Alto%20Security%20Baseline%20Reference.docx) | Baseline reference in Word form |

## Why

Vendor documentation is scattered across official docs, community threads, KB articles, and PDFs, and the useful parts are rarely in the same place as the reasoning behind them. This repo is one place to keep what's actually useful day to day, in a format that's fast to search and reusable across engagements.

## Structure

```
useful-skillet/
├── architecture/   # senior-level design references (NGFW, Azure, Prisma Access, Prisma SD-WAN)
├── baselines/      # hardening checklists per platform
├── upgrades/       # PAN-OS upgrade runbooks
├── assessments/    # engagement methodology and templates
├── gotchas/        # things that broke, and why
├── configs/        # CLI/XML snippets
├── reference/      # source docs (docx, etc.)
└── README.md
```

## Notes

Baselines and design guides are starting points, not drop-in configs. Every environment needs its own segmentation model, risk tolerance, sizing, and documented deviations layered on top. Architecture docs target PAN-OS 11.1 as the current baseline; always confirm version-specific behavior against the release notes for your exact target.

## Status

Actively maintained. Added to as engagements and lab work turn up something worth keeping.
