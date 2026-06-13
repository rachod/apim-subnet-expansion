# APIM Subnet Expansion - Scale Azure API Management Beyond Subnet Limits

## Problem Statement

Azure API Management (Premium, Internal VNet mode) deployed in a constrained subnet cannot scale due to insufficient IP addresses. This repository provides a step-by-step guide to:

1. Identify the subnet IP exhaustion issue
2. Backup APIM configuration safely
3. Migrate APIM to a larger subnet with zero configuration loss
4. Scale APIM to the desired capacity

> **Note:** This guide is validated against APIM Premium tier with stv2 platform in Internal VNet mode on Southeast Asia region.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  VNet: 10.0.0.0/16                                          │
│                                                             │
│  ┌─────────────────────┐    ┌─────────────────────────────┐ │
│  │ snet-apim-small     │    │ snet-apim-large             │ │
│  │ 10.0.1.0/28         │    │ 10.0.2.0/25                 │ │
│  │ (11 usable IPs)     │    │ (123 usable IPs)            │ │
│  │                     │    │                             │ │
│  │ ❌ FULL at 5 units  │───►│ ✅ Scaled to 8+ units       │ │
│  └─────────────────────┘    └─────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

---

## IP Consumption per APIM Unit

| APIM Units | VMSS Instances | Load Balancer | Total IPs Required |
|-----------|----------------|---------------|-------------------|
| 1 | 2 | 1 | 3 |
| 3 | 6 | 1 | 7 |
| 5 | 10 | 1 | 11 |
| 8 | 16 | 1 | 17 |
| 12 | 24 | 1 | 25 |

> **Formula:** `IPs required = (units × 2) + 1`

---

## Subnet Sizing Guide

| Target Max Units | Minimum Subnet | Recommended Subnet | Usable IPs |
|-----------------|----------------|-------------------|------------|
| 1-3 units       | /27            | /26               | 27 / 59    |
| 4-6 units       | /26            | /25               | 59 / 123   |
| 7-12 units      | /25            | /24               | 123 / 251  |

---

## Step-by-Step Guide

| # | Document | Description |
|---|----------|-------------|
| 1 | [Prerequisites](docs/01-Prerequisites.md) | Requirements, environment setup, NSG and VNet creation |
| 2 | [Backup Configuration](docs/03-BackupConfiguration.md) | Backup APIM configuration before migration |
| 3 | [Subnet Migration](docs/04-SubnetMigration.md) | Migrate APIM to a larger subnet (CLI) with pre-flight checks |
| 4 | [Work Instruction — Portal](docs/05-WorkInstruction-Portal.md) | Step-by-step portal guide with screenshots |
| 5 | [Pre-Migration Checklist](docs/06-PreMigrationChecklist.md) | Production checklist with az commands per step |

---

## Contributing

Feel free to open issues or submit PRs for improvements.

## License

MIT
