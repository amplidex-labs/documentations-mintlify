# Overview

AmpliDex is a non-custodial leveraged markets and liquidity protocol designed for Stellar Soroban.

Liquidity providers supply capital to isolated lending pools, while traders post collateral and borrow assets to create long or short exposure through approved Stellar execution venues. Deterministic risk controls, collateral requirements, and liquidation rules are used to protect protocol solvency and return repaid or liquidated debt to the appropriate liquidity pool.

This documentation is intended for builders, engineers, security reviewers, grant assessors, protocol partners, and technical investors. It describes the AmpliDex system architecture and distinguishes between validated MVP capabilities, production-hardening requirements, staged protocol extensions, and longer-term research.

The technical architecture presented in this documentation represents the planned production system. Components that have already been implemented or validated are identified separately from functionality that remains under development.

<Info>
Before mainnet launch, AmpliDex will publish a complete evidence register containing repository tags, deployed contract IDs, audit reports, artifact hashes, test results, operational metrics, and other verifiable implementation evidence.
</Info>

## At a glance

<table data-search="false"><thead><tr><th>Property</th><th>Design</th></tr></thead><tbody><tr><td>Settlement</td><td>Stellar / Soroban</td></tr><tr><td>Primary accounting asset</td><td>Native USDC on Stellar</td></tr><tr><td>Custody</td><td>Non-custodial</td></tr><tr><td>Lending</td><td>Isolated pools with share and borrow-index accounting</td></tr><tr><td>Execution</td><td>Governed, bounded Aquarius/Soroswap adapters</td></tr><tr><td>Pricing</td><td>Independent reference sources plus DEX executable route values</td></tr><tr><td>Liquidation</td><td>Permissionless liquidation settlement with keeper-assisted automation</td></tr><tr><td>Cross-chain</td><td>EVM-controlled Soroban smart accounts and CCTP asset transpor</td></tr></tbody></table>

For the complete protocol design, system components, transaction flows, risk model, and implementation boundaries, see the [Technical Architecture](technical-architecture.md)
