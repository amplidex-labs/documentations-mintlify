---
description: Production target architecture, security model, and delivery plan for AmpliDex.
---

# AmpliDex Technical Architecture

AmpliDex is a non-custodial leveraged-markets and liquidity protocol designed for Stellar Soroban. Liquidity providers supply isolated pools; traders post collateral and borrow to create long or short exposure through approved Stellar execution venues; deterministic risk and liquidation rules settle debt back to the appropriate pool.

This GitBook is written for engineers, security reviewers, grant assessors, and technical investors. It describes the target production system and clearly separates validated MVP capabilities, production hardening, staged extensions, and research.

{% hint style="warning" %}
Architecture is not deployment evidence. Before external due diligence, complete the evidence register with repository tags, contract IDs, test reports, audit reports, artifact hashes, and operating metrics.
{% endhint %}

## At a glance

| Property | Design |
|---|---|
| Settlement | Stellar / Soroban |
| Primary accounting asset | Native USDC on Stellar |
| Custody | Non-custodial |
| Lending | Isolated pools with share and borrow-index accounting |
| Execution | Governed, bounded Aquarius/Soroswap adapters |
| Pricing | Independent reference sources plus executable route value |
| Liquidation | Permissionless settlement with keeper automation |
| Cross-chain | Staged EVM smart accounts and CCTP asset transport |
| Privacy | Staged, opt-in research track |

Start with [Technical Architecture](technical-architecture.md), then use the [Production Readiness & Evidence](production-readiness-and-evidence.md) page when preparing a grant or investment data room.
