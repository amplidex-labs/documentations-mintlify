# Overview

AmpliDex is a non-custodial leveraged-markets and liquidity protocol designed for Stellar Soroban. Liquidity providers supply liquidity on isolated pools; traders post collateral and borrow to create long or short exposure through approved Stellar execution venues; deterministic risk and liquidation rules settle debt back to the appropriate pool.

This documentation is written for builders, engineers, security reviewers, grant assessors, and technical investors. It describes the production system and clearly separates validated MVP capabilities, production hardening, staged extensions, and research.

{% hint style="info" %}
The presented technical architecture represent planned development. Before mainnet launch, a complete evidence register with repository tags, contract IDs, audit reports, artifact hashes, and operating metrics will be published.
{% endhint %}

## At a glance

<table data-search="false"><thead><tr><th>Property</th><th>Design</th></tr></thead><tbody><tr><td>Settlement</td><td>Stellar / Soroban</td></tr><tr><td>Primary accounting asset</td><td>Native USDC on Stellar</td></tr><tr><td>Custody</td><td>Non-custodial</td></tr><tr><td>Lending</td><td>Isolated pools with share and borrow-index accounting</td></tr><tr><td>Execution</td><td>Governed, bounded Aquarius/Soroswap adapters</td></tr><tr><td>Pricing</td><td>Independent reference sources plus DEXs executable route value</td></tr><tr><td>Liquidation</td><td>Permissionless settlement with keeper automation</td></tr><tr><td>Cross-chain</td><td>EVM - smart accounts and CCTP asset transport</td></tr></tbody></table>

For a full technical architecture of AmpliDex protocol, review the technical document at [Technical Architecture](technical-architecture.md)
