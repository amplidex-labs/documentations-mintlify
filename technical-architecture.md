# AmpliDex Technical Architecture

### 1. Purpose and Scope

AmpliDex is a non-custodial leveraged-markets and liquidity protocol designed for Stellar Soroban.

Liquidity providers supply assets to isolated lending pools. Traders post collateral and borrow from those pools to create long or short exposure through approved Stellar execution venues. Interest accrues through pool-level borrow indexes, while positions remain subject to deterministic solvency rules at every relevant protocol transition and may be independently monitored for permissionless liquidation eligibility.

AmpliDex is designed around five production principles:

1. keep financial authority and settlement on Stellar/Soroban;
2. isolate markets and external dependencies so failures remain bounded;
3. constrain, delay, and expose privileged actions;
4. preserve permissionless repayment, exits, and liquidation wherever safe; and
5. allow EVM users to access Stellar-native positions without introducing backend custody.

The core production design covers:

* isolated lending pools;
* LP share accounting;
* indexed borrowing and interest accrual;
* leveraged long and short positions;
* bounded DEX execution;
* resilient pricing;
* deterministic solvency checks;
* permissionless liquidation;
* bad-debt accounting;
* indexing and query infrastructure;
* governance and upgrade controls; and
* production monitoring and operations.

USDC serves as the primary protocol quote and reporting denomination. Individual markets may additionally support borrowable non-USDC assets where required for short exposure.

### 2. Scope and Maturity

#### 2.1 Capability Register

The architecture describes the intended production system. A capability appearing in this document does not imply that it has been deployed, audited, or classified as production-ready.

| **Capability**                        | **Current Status** | **Production Scope**     | **Release Gate**                                   |
| ------------------------------------- | ------------------ | ------------------------ | -------------------------------------------------- |
| Lending pools and LP shares           | MVP validated      | Core launch              | Economic review and invariant tests                |
| Borrow-index accounting               | MVP validated      | Core launch              | Rounding, overflow, accrual, and time-jump tests   |
| Long and short positions              | MVP validated      | Core launch              | Lifecycle, solvency, and insolvency tests          |
| Add margin, repay, partial/full close | MVP validated      | Core launch              | State-machine and property tests                   |
| Aquarius execution                    | MVP validated      | Core launch              | Adapter review and production-route verification   |
| Keeper-assisted liquidation           | MVP validated      | Core launch              | Independent keeper and liquidation-race tests      |
| Soroswap integration                  | Proposed           | Production hardening     | Adapter integration and route-bound testing        |
| Route failover                        | Proposed           | Production hardening     | Bounded fallback and failure-mode testing          |
| Multi-source oracle router            | Proposed           | Production hardening     | Freshness, deviation, and degraded-mode testing    |
| Event indexer and query API           | Proposed           | Production hardening     | Replay, reconciliation, backup, and SLO validation |
| Direct liquidation market             | Proposed           | Post-core activation     | Economic review and capped rollout                 |
| EVM-controlled smart accounts         | Proposed           | Cross-chain extension    | Authentication and authorization review            |
| Circle CCTP integration               | Proposed           | Cross-chain extension    | Recovery, replay, and supported-domain validation  |
| Confidential positions                | Research           | Future opt-in capability | Circuit benchmarks and dedicated verifier audit    |

“MVP validated” reflects implementation or validation claims associated with the current project scope. Before external production claims are made, each relevant capability must be linked to verifiable evidence such as a repository release, deployed contract ID, automated test report, transaction record, review artifact, or reproducible build.

For canonical release status and supporting artifacts, see [**Production Readiness & Evidence**](production-readiness-and-evidence.md).

#### 2.2 Release Non-Goals

The production release does not target:

* replicated leveraged-position state across multiple chains;
* arbitrary user-supplied DEX contracts;
* arbitrary user-supplied execution routes;
* arbitrary user-supplied price feeds;
* backend custody;
* server-held user signing keys;
* uncapped permissionless market listing;
* guaranteed LP withdrawal liquidity during high utilization;
* complete transaction anonymity;
* unreviewed automatic contract upgrades; or
* hidden socialization of bad debt across unrelated markets.

## 3. System Context

<svg version="1.1" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1690.395462207016 1806.3789764903486" width="1690.395462207016" height="1806.3789764903486"><!-- svg-source:excalidraw --><metadata></metadata><defs><style class="style-fonts">
      @font-face { font-family: Excalifont; src: url(data:font/woff2;base64,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); }</style></defs><rect x="0" y="0" width="1690.395462207016" height="1806.3789764903486" fill="#ffffff"></rect><g stroke-linecap="round" transform="translate(868.1996056721291 1692.122728442404) rotate(0 75.19347034366547 40.288594891758294)"><path d="M0 0 C45.29 -2.28, 89.33 -2.37, 150.39 0 M0 0 C43.03 -1.48, 87.55 -0.2, 150.39 0 M150.39 0 C151.96 20.68, 151.05 41.44, 150.39 80.58 M150.39 0 C151.37 19.38, 150.43 38.13, 150.39 80.58 M150.39 80.58 C109.21 81.28, 63.59 79.68, 0 80.58 M150.39 80.58 C114.73 79.99, 79.93 81.09, 0 80.58 M0 80.58 C-1.08 52.77, -0.77 24.81, 0 0 M0 80.58 C-0.25 61.9, 0.57 43.63, 0 0" stroke="#1971c2" stroke-width="2" fill="none"></path></g><g transform="translate(902.3331166041735 1719.9113233341614) rotate(0 41.059959411621094 12.5)"><text x="41.059959411621094" y="17.619999999999997" font-family="Excalifont, Xiaolai, sans-serif, Segoe UI Emoji" font-size="20px" fill="#1971c2" text-anchor="middle" style="white-space: pre;" direction="ltr" dominant-baseline="alphabetic">Aquarius</text></g><g stroke-linecap="round" transform="translate(1232.9499703807087 1687.828248916625) rotate(0 100.53459211014342 42.5)"><path d="M0 0 C73.46 -2.97, 142.32 -1.06, 201.07 0 M0 0 C78.23 0.01, 156.55 -0.14, 201.07 0 M201.07 0 C199.48 21.45, 199.41 45.15, 201.07 85 M201.07 0 C201 28.42, 199.55 58.23, 201.07 85 M201.07 85 C139.25 85.24, 78.62 85.28, 0 85 M201.07 85 C121.12 86.38, 41.98 86.36, 0 85 M0 85 C0.73 56.03, -1.48 27.5, 0 0 M0 85 C-0.87 51, -0.06 18.79, 0 0" stroke="#2f9e44" stroke-width="2" fill="none"></path></g><g transform="translate(1245.9046445831373 1705.328248916625) rotate(0 87.57991790771484 25)"><text x="87.57991790771484" y="17.619999999999997" font-family="Excalifont, Xiaolai, sans-serif, Segoe UI Emoji" font-size="20px" fill="#2f9e44" text-anchor="middle" style="white-space: pre;" direction="ltr" dominant-baseline="alphabetic">Primary Reference</text><text x="87.57991790771484" y="42.62" font-family="Excalifont, Xiaolai, sans-serif, Segoe UI Emoji" font-size="20px" fill="#2f9e44" text-anchor="middle" style="white-space: pre;" direction="ltr" dominant-baseline="alphabetic">Source</text></g><g stroke-linecap="round" transform="translate(1039.8696956451386 1693.415136183823) rotate(0 75.19347034366547 38.563573483216715)"><path d="M0 0 C40.04 2.67, 78.91 2.8, 150.39 0 M0 0 C60.02 1.61, 119.11 1.09, 150.39 0 M150.39 0 C148.96 17.06, 150.65 29.79, 150.39 77.13 M150.39 0 C151.57 22.97, 151.61 46.76, 150.39 77.13 M150.39 77.13 C100.51 77.39, 55.62 75.99, 0 77.13 M150.39 77.13 C113.75 78.55, 75.94 78.24, 0 77.13 M0 77.13 C-0.18 60.2, -0.46 43.31, 0 0 M0 77.13 C0.72 53.91, 0.42 30.02, 0 0" stroke="#1971c2" stroke-width="2" fill="none"></path></g><g transform="translate(1068.7432044409525 1719.4787096670407) rotate(0 46.31996154785156 12.5)"><text x="46.31996154785156" y="17.619999999999997" font-family="Excalifont, Xiaolai, sans-serif, Segoe UI Emoji" font-size="20px" fill="#1971c2" text-anchor="middle" style="white-space: pre;" direction="ltr" dominant-baseline="alphabetic">SoroSwap</text></g><g stroke-linecap="round" transform="translate(1452.9355438064558 1687.5405524233884) rotate(0 100.53459211014342 42.5)"><path d="M0 0 C45.72 1.41, 89.93 -0.58, 201.07 0 M0 0 C64.98 0.99, 130.55 2.29, 201.07 0 M201.07 0 C199.9 24.66, 199.16 50.76, 201.07 85 M201.07 0 C201.78 24.78, 202.21 47.23, 201.07 85 M201.07 85 C132.81 82.59, 65.67 83.92, 0 85 M201.07 85 C150.28 85.85, 99.66 86.57, 0 85 M0 85 C1.62 68.33, -0.91 48.67, 0 0 M0 85 C1.21 68.3, 0.08 49.6, 0 0" stroke="#2f9e44" stroke-width="2" fill="none"></path></g><g transform="translate(1469.210210074314 1705.0405524233884) rotate(0 84.25992584228516 25)"><text x="84.25992584228516" y="17.619999999999997" font-family="Excalifont, Xiaolai, sans-serif, Segoe UI Emoji" font-size="20px" fill="#2f9e44" text-anchor="middle" style="white-space: pre;" direction="ltr" dominant-baseline="alphabetic">Secondary</text><text x="84.25992584228516" y="42.62" font-family="Excalifont, Xiaolai, sans-serif, Segoe UI Emoji" font-size="20px" fill="#2f9e44" text-anchor="middle" style="white-space: pre;" direction="ltr" dominant-baseline="alphabetic">Reference Source</text></g><g stroke-linecap="round"><g transform="translate(942.8676343637835 1534.384593304614) rotate(0 0.21272082600535214 74.98467471367348)"><path d="M0 0 C0.16 55.97, 0.32 111.94, 0.43 149.97 M0 0 C0.14 50.96, 0.29 101.92, 0.43 149.97" stroke="#1971c2" stroke-width="2" fill="none"></path></g><g transform="translate(942.8676343637835 1534.384593304614) rotate(0 0.21272082600535214 74.98467471367348)"><path d="M0 0 L6.38 13.58 L-6.3 13.61 L0 0" stroke="none" stroke-width="0" fill="#1971c2" fill-rule="evenodd"></path><path d="M0 0 C2.38 5.07, 4.76 10.13, 6.38 13.58 M0 0 C2.17 4.61, 4.33 9.23, 6.38 13.58 M6.38 13.58 C3.3 13.59, 0.23 13.59, -6.3 13.61 M6.38 13.58 C2.59 13.59, -1.21 13.6, -6.3 13.61 M-6.3 13.61 C-4.43 9.56, -2.55 5.51, 0 0 M-6.3 13.61 C-4.29 9.27, -2.28 4.93, 0 0 M0 0 C0 0, 0 0, 0 0 M0 0 C0 0, 0 0, 0 0" stroke="#1971c2" stroke-width="2" fill="none"></path></g><g transform="translate(942.8676343637835 1534.384593304614) rotate(0 0.21272082600535214 74.98467471367348)"><path d="M0.43 149.97 L-5.95 136.39 L6.73 136.36 L0.43 149.97" stroke="none" stroke-width="0" fill="#1971c2" fill-rule="evenodd"></path><path d="M0.43 149.97 C-1.95 144.9, -4.34 139.84, -5.95 136.39 M0.43 149.97 C-1.74 145.36, -3.91 140.74, -5.95 136.39 M-5.95 136.39 C-2.88 136.38, 0.2 136.38, 6.73 136.36 M-5.95 136.39 C-2.16 136.38, 1.63 136.37, 6.73 136.36 M6.73 136.36 C4.85 140.41, 2.98 144.46, 0.43 149.97 M6.73 136.36 C4.72 140.7, 2.71 145.04, 0.43 149.97 M0.43 149.97 C0.43 149.97, 0.43 149.97, 0.43 149.97 M0.43 149.97 C0.43 149.97, 0.43 149.97, 0.43 149.97" stroke="#1971c2" stroke-width="2" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(993.1287812922783 1534.384593304614) rotate(0 60.91719234826178 75.79719515244597)"><path d="M0 0 C0 18.71, 0 37.41, 0 79.47 M0 0 C0 21.58, 0 43.16, 0 79.47 M0 79.47 C0 90.14, 5.33 95.47, 16 95.47 M0 79.47 C0 90.14, 5.33 95.47, 16 95.47 M16 95.47 C37.16 95.47, 58.32 95.47, 105.83 95.47 M16 95.47 C46.54 95.47, 77.09 95.47, 105.83 95.47 M105.83 95.47 C116.5 95.47, 121.83 100.81, 121.83 111.47 M105.83 95.47 C116.5 95.47, 121.83 100.81, 121.83 111.47 M121.83 111.47 C121.83 127.06, 121.83 142.65, 121.83 151.59 M121.83 111.47 C121.83 122.81, 121.83 134.16, 121.83 151.59" stroke="#1971c2" stroke-width="2" fill="none"></path></g><g transform="translate(993.1287812922783 1534.384593304614) rotate(0 60.91719234826178 75.79719515244597)"><path d="M0 0 L6.34 13.59 L-6.34 13.59 L0 0" stroke="none" stroke-width="0" fill="#1971c2" fill-rule="evenodd"></path><path d="M0 0 C1.49 3.2, 2.98 6.4, 6.34 13.59 M0 0 C1.72 3.69, 3.44 7.38, 6.34 13.59 M6.34 13.59 C1.41 13.59, -3.52 13.59, -6.34 13.59 M6.34 13.59 C1.33 13.59, -3.68 13.59, -6.34 13.59 M-6.34 13.59 C-5 10.72, -3.66 7.85, 0 0 M-6.34 13.59 C-4.09 8.77, -1.84 3.95, 0 0 M0 0 C0 0, 0 0, 0 0 M0 0 C0 0, 0 0, 0 0" stroke="#1971c2" stroke-width="2" fill="none"></path></g><g transform="translate(993.1287812922783 1534.384593304614) rotate(0 60.91719234826178 75.79719515244597)"><path d="M121.83 151.59 L115.5 138 L128.17 138 L121.83 151.59" stroke="none" stroke-width="0" fill="#1971c2" fill-rule="evenodd"></path><path d="M121.83 151.59 C120.34 148.39, 118.85 145.19, 115.5 138 M121.83 151.59 C120.11 147.9, 118.39 144.21, 115.5 138 M115.5 138 C120.43 138, 125.36 138, 128.17 138 M115.5 138 C120.51 138, 125.52 138, 128.17 138 M128.17 138 C126.83 140.87, 125.5 143.74, 121.83 151.59 M128.17 138 C125.92 142.82, 123.68 147.65, 121.83 151.59 M121.83 151.59 C121.83 151.59, 121.83 151.59, 121.83 151.59 M121.83 151.59 C121.83 151.59, 121.83 151.59, 121.83 151.59" stroke="#1971c2" stroke-width="2" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(1181.8223002368322 1536.8942520118544) rotate(0 75.76428051374933 71.36939473127131)"><path d="M0 0 C0 28.99, 0 57.97, 0 78.39 M0 0 C0 26.54, 0 53.08, 0 78.39 M0 78.39 C0 89.05, 5.33 94.39, 16 94.39 M0 78.39 C0 89.05, 5.33 94.39, 16 94.39 M16 94.39 C44.08 94.39, 72.16 94.39, 135.53 94.39 M16 94.39 C46.32 94.39, 76.63 94.39, 135.53 94.39 M135.53 94.39 C146.2 94.39, 151.53 99.72, 151.53 110.39 M135.53 94.39 C146.2 94.39, 151.53 99.72, 151.53 110.39 M151.53 110.39 C151.53 119.78, 151.53 129.17, 151.53 142.74 M151.53 110.39 C151.53 120.99, 151.53 131.6, 151.53 142.74" stroke="#2f9e44" stroke-width="2" fill="none"></path></g><g transform="translate(1181.8223002368322 1536.8942520118544) rotate(0 75.76428051374933 71.36939473127131)"><path d="M0 0 L6.34 13.59 L-6.34 13.59 L0 0" stroke="none" stroke-width="0" fill="#2f9e44" fill-rule="evenodd"></path><path d="M0 0 C2.34 5.03, 4.69 10.05, 6.34 13.59 M0 0 C2.15 4.6, 4.29 9.21, 6.34 13.59 M6.34 13.59 C2.54 13.59, -1.26 13.59, -6.34 13.59 M6.34 13.59 C3.08 13.59, -0.19 13.59, -6.34 13.59 M-6.34 13.59 C-4.92 10.55, -3.5 7.51, 0 0 M-6.34 13.59 C-4.88 10.47, -3.43 7.35, 0 0 M0 0 C0 0, 0 0, 0 0 M0 0 C0 0, 0 0, 0 0" stroke="#2f9e44" stroke-width="2" fill="none"></path></g><g transform="translate(1181.8223002368322 1536.8942520118544) rotate(0 75.76428051374933 71.36939473127131)"><path d="M151.53 142.74 L145.19 129.14 L157.87 129.14 L151.53 142.74" stroke="none" stroke-width="0" fill="#2f9e44" fill-rule="evenodd"></path><path d="M151.53 142.74 C149.18 137.71, 146.84 132.68, 145.19 129.14 M151.53 142.74 C149.38 138.14, 147.24 133.53, 145.19 129.14 M145.19 129.14 C148.99 129.14, 152.79 129.14, 157.87 129.14 M145.19 129.14 C148.45 129.14, 151.72 129.14, 157.87 129.14 M157.87 129.14 C156.45 132.19, 155.03 135.23, 151.53 142.74 M157.87 129.14 C156.41 132.27, 154.96 135.39, 151.53 142.74 M151.53 142.74 C151.53 142.74, 151.53 142.74, 151.53 142.74 M151.53 142.74 C151.53 142.74, 151.53 142.74, 151.53 142.74" stroke="#2f9e44" stroke-width="2" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(1224.7456098243674 1536.8942520118544) rotate(0 198.5535444198049 78.08169034209732)"><path d="M0 0 C0 13.99, 0 27.99, 0 48.6 M0 0 C0 16.91, 0 33.83, 0 48.6 M0 48.6 C0 59.27, 5.33 64.6, 16 64.6 M0 48.6 C0 59.27, 5.33 64.6, 16 64.6 M16 64.6 C95 64.6, 174 64.6, 381.11 64.6 M16 64.6 C126.88 64.6, 237.76 64.6, 381.11 64.6 M381.11 64.6 C391.77 64.6, 397.11 69.94, 397.11 80.6 M381.11 64.6 C391.77 64.6, 397.11 69.94, 397.11 80.6 M397.11 80.6 C397.11 106.95, 397.11 133.3, 397.11 156.16 M397.11 80.6 C397.11 96.37, 397.11 112.14, 397.11 156.16" stroke="#2f9e44" stroke-width="2" fill="none"></path></g><g transform="translate(1224.7456098243674 1536.8942520118544) rotate(0 198.5535444198049 78.08169034209732)"><path d="M0 0 L6.34 13.59 L-6.34 13.59 L0 0" stroke="none" stroke-width="0" fill="#2f9e44" fill-rule="evenodd"></path><path d="M0 0 C1.83 3.91, 3.65 7.83, 6.34 13.59 M0 0 C2.21 4.73, 4.41 9.46, 6.34 13.59 M6.34 13.59 C1.93 13.59, -2.48 13.59, -6.34 13.59 M6.34 13.59 C3.43 13.59, 0.51 13.59, -6.34 13.59 M-6.34 13.59 C-4.48 9.6, -2.61 5.61, 0 0 M-6.34 13.59 C-4.77 10.24, -3.21 6.88, 0 0 M0 0 C0 0, 0 0, 0 0 M0 0 C0 0, 0 0, 0 0" stroke="#2f9e44" stroke-width="2" fill="none"></path></g><g transform="translate(1224.7456098243674 1536.8942520118544) rotate(0 198.5535444198049 78.08169034209732)"><path d="M397.11 156.16 L390.77 142.57 L403.45 142.57 L397.11 156.16" stroke="none" stroke-width="0" fill="#2f9e44" fill-rule="evenodd"></path><path d="M397.11 156.16 C395.28 152.25, 393.46 148.34, 390.77 142.57 M397.11 156.16 C394.9 151.43, 392.69 146.7, 390.77 142.57 M390.77 142.57 C395.18 142.57, 399.59 142.57, 403.45 142.57 M390.77 142.57 C393.68 142.57, 396.59 142.57, 403.45 142.57 M403.45 142.57 C401.58 146.56, 399.72 150.56, 397.11 156.16 M403.45 142.57 C401.88 145.92, 400.32 149.28, 397.11 156.16 M397.11 156.16 C397.11 156.16, 397.11 156.16, 397.11 156.16 M397.11 156.16 C397.11 156.16, 397.11 156.16, 397.11 156.16" stroke="#2f9e44" stroke-width="2" fill="none"></path></g></g><mask></mask><g stroke-linecap="round" transform="translate(1005.0206533392629 435.39055345277484) rotate(0 77.86627983279777 42.5)"><path d="M0 0 C42.43 0.08, 86.19 0.83, 155.73 0 M0 0 C31.94 -0.2, 63.71 0.55, 155.73 0 M155.73 0 C156.9 28.87, 156.64 53.47, 155.73 85 M155.73 0 C156.3 27.97, 154.92 54.72, 155.73 85 M155.73 85 C102.32 86.66, 49.92 86.25, 0 85 M155.73 85 C94.48 85.57, 32.53 84.18, 0 85 M0 85 C-1.64 60.4, -1.23 39.05, 0 0 M0 85 C-0.34 55.09, 0.01 23.13, 0 0" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(1019.816956365421 442.89055345277484) rotate(0 63.069976806640625 35)"><text x="63.069976806640625" y="24.668" font-family="Excalifont, Xiaolai, sans-serif, Segoe UI Emoji" font-size="28px" fill="#1e1e1e" text-anchor="middle" style="white-space: pre;" direction="ltr" dominant-baseline="alphabetic">Circle</text><text x="63.069976806640625" y="59.668" font-family="Excalifont, Xiaolai, sans-serif, Segoe UI Emoji" font-size="28px" fill="#1e1e1e" text-anchor="middle" style="white-space: pre;" direction="ltr" dominant-baseline="alphabetic">CCTP V2</text></g><g stroke-linecap="round" transform="translate(1001.8664368720038 271.23955586907323) rotate(0 137.24810745371724 42.5)"><path d="M0 0 C104.09 0.55, 208.46 0.31, 274.5 0 M0 0 C71.4 0.62, 143.88 0.7, 274.5 0 M274.5 0 C272.42 26.3, 272.5 51.11, 274.5 85 M274.5 0 C275.11 18.44, 274.21 37.85, 274.5 85 M274.5 85 C204.2 86.1, 130.96 86.18, 0 85 M274.5 85 C217.75 84.85, 161.34 85.32, 0 85 M0 85 C-0.21 59.16, -0.71 35.28, 0 0 M0 85 C-1.54 65.41, -1.07 48.06, 0 0" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(1057.0745815571663 296.23955586907323) rotate(0 82.03996276855469 17.5)"><text x="82.03996276855469" y="24.668" font-family="Excalifont, Xiaolai, sans-serif, Segoe UI Emoji" font-size="28px" fill="#1e1e1e" text-anchor="middle" style="white-space: pre;" direction="ltr" dominant-baseline="alphabetic">EVM Wallets</text></g><g stroke-linecap="round" transform="translate(685.9733370768136 271.13560144879557) rotate(0 121.72249567608378 42.5)"><path d="M0 0 C66.58 -0.66, 131.66 0.41, 243.44 0 M0 0 C94.85 -0.88, 191.9 0.11, 243.44 0 M243.44 0 C242.61 34.71, 242.82 65.65, 243.44 85 M243.44 0 C242.83 22.37, 244.01 46.19, 243.44 85 M243.44 85 C178.56 85.27, 117.07 87.07, 0 85 M243.44 85 C156.81 85.06, 69.12 85.22, 0 85 M0 85 C-1.4 58.08, -0.13 35.15, 0 0 M0 85 C-0.68 62.66, 0.37 41.56, 0 0" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(710.4658828017255 296.13560144879557) rotate(0 97.22994995117188 17.5)"><text x="97.22994995117188" y="24.668" font-family="Excalifont, Xiaolai, sans-serif, Segoe UI Emoji" font-size="28px" fill="#1e1e1e" text-anchor="middle" style="white-space: pre;" direction="ltr" dominant-baseline="alphabetic">Stellar Wallets</text></g><g stroke-linecap="round" transform="translate(1001.1384800814103 589.936018356826) rotate(0 135.41553115938677 42.5)"><path d="M0 0 C104.53 -0.56, 210.82 0.07, 270.83 0 M0 0 C74.62 -1.94, 150.84 -1.36, 270.83 0 M270.83 0 C272.33 29.9, 271.18 56.06, 270.83 85 M270.83 0 C270.86 24.08, 270.88 49.68, 270.83 85 M270.83 85 C183.94 84.76, 97.14 84.35, 0 85 M270.83 85 C177.43 85.01, 85.29 86.12, 0 85 M0 85 C0.77 60.44, 0.73 39.62, 0 0 M0 85 C-1.04 67.76, -1.43 48.39, 0 0" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(1036.496050730544 597.436018356826) rotate(0 100.0579605102539 35)"><text x="100.0579605102539" y="24.668" font-family="Excalifont, Xiaolai, sans-serif, Segoe UI Emoji" font-size="28px" fill="#1e1e1e" text-anchor="middle" style="white-space: pre;" direction="ltr" dominant-baseline="alphabetic">Soroban Smart</text><text x="100.0579605102539" y="59.668" font-family="Excalifont, Xiaolai, sans-serif, Segoe UI Emoji" font-size="28px" fill="#1e1e1e" text-anchor="middle" style="white-space: pre;" direction="ltr" dominant-baseline="alphabetic">Account</text></g><g stroke-linecap="round" transform="translate(606.4759984952325 1220.5571154360223) rotate(0 81.61691130631789 42.5)"><path d="M0 0 C45.07 0.36, 92.03 0.2, 163.23 0 M0 0 C57.33 0.52, 115.67 -0.25, 163.23 0 M163.23 0 C165.12 22.6, 162.64 41.03, 163.23 85 M163.23 0 C163.26 30.92, 163.68 60.34, 163.23 85 M163.23 85 C114.55 85.45, 63.47 86.49, 0 85 M163.23 85 C99.24 83.82, 36.18 82.48, 0 85 M0 85 C-1.17 54.55, -1.92 19.44, 0 0 M0 85 C0.03 52.65, -0.49 20.61, 0 0" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(638.5889348870005 1228.0571154360223) rotate(0 49.50397491455078 35)"><text x="49.50397491455078" y="24.668" font-family="Excalifont, Xiaolai, sans-serif, Segoe UI Emoji" font-size="28px" fill="#1e1e1e" text-anchor="middle" style="white-space: pre;" direction="ltr" dominant-baseline="alphabetic">Lending</text><text x="49.50397491455078" y="59.668" font-family="Excalifont, Xiaolai, sans-serif, Segoe UI Emoji" font-size="28px" fill="#1e1e1e" text-anchor="middle" style="white-space: pre;" direction="ltr" dominant-baseline="alphabetic">Pools</text></g><g stroke-linecap="round" transform="translate(882.820621033321 1443.384593304614) rotate(0 85.3774499644669 42.5)"><path d="M0 0 C55.86 -0.65, 112.51 1.18, 170.75 0 M0 0 C52.19 -1.06, 105.84 -0.77, 170.75 0 M170.75 0 C173 18.92, 171.26 39, 170.75 85 M170.75 0 C171.48 22.6, 170.28 46.92, 170.75 85 M170.75 85 C134.49 84.51, 100.07 83.91, 0 85 M170.75 85 C106.26 84.01, 41.65 82.83, 0 85 M0 85 C2 58.34, 0.26 31.45, 0 0 M0 85 C0.39 55.07, -0.35 24.31, 0 0" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(900.8581051774763 1450.884593304614) rotate(0 67.3399658203125 35)"><text x="67.3399658203125" y="24.668" font-family="Excalifont, Xiaolai, sans-serif, Segoe UI Emoji" font-size="28px" fill="#1e1e1e" text-anchor="middle" style="white-space: pre;" direction="ltr" dominant-baseline="alphabetic">Execution</text><text x="67.3399658203125" y="59.668" font-family="Excalifont, Xiaolai, sans-serif, Segoe UI Emoji" font-size="28px" fill="#1e1e1e" text-anchor="middle" style="white-space: pre;" direction="ltr" dominant-baseline="alphabetic">Router</text></g><g stroke-linecap="round" transform="translate(895.3834128928356 1189.796340233721) rotate(0 181.98059013661896 74.51921740882062)"><path d="M0 0 C78.3 2.38, 154.98 1.54, 363.96 0 M0 0 C101.54 1.84, 202.5 1.51, 363.96 0 M363.96 0 C364.47 31.26, 364.51 65.95, 363.96 149.04 M363.96 0 C364.51 47.74, 364.59 96.36, 363.96 149.04 M363.96 149.04 C267.76 148.14, 171.85 148.53, 0 149.04 M363.96 149.04 C234.95 147.62, 105.81 147.84, 0 149.04 M0 149.04 C-2.55 100.98, -2.38 49.24, 0 0 M0 149.04 C0.4 105.91, 2.08 62.29, 0 0" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(954.7660416036742 1241.8155576425415) rotate(0 122.59796142578125 22.5)"><text x="122.59796142578125" y="31.716" font-family="Excalifont, Xiaolai, sans-serif, Segoe UI Emoji" font-size="36px" fill="#1e1e1e" text-anchor="middle" style="white-space: pre;" direction="ltr" dominant-baseline="alphabetic">AmpliDex Core</text></g><g stroke-linecap="round" transform="translate(1113.5414269043995 1445.8942520118526) rotate(0 92.76532177406261 42.5)"><path d="M0 0 C38.94 -0.32, 82.42 -0.89, 185.53 0 M0 0 C39.45 0.2, 77.51 1.64, 185.53 0 M185.53 0 C187.03 28.32, 187.26 60.51, 185.53 85 M185.53 0 C185.27 30.99, 185.26 61.79, 185.53 85 M185.53 85 C141.16 85.48, 97.15 82.86, 0 85 M185.53 85 C129.59 84.9, 74.2 85.07, 0 85 M0 85 C-0.58 65.39, -2.1 42.6, 0 0 M0 85 C0.19 66.59, 1.06 47.66, 0 0" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(1158.9027722380324 1453.3942520118526) rotate(0 47.40397644042969 35)"><text x="47.40397644042969" y="24.668" font-family="Excalifont, Xiaolai, sans-serif, Segoe UI Emoji" font-size="28px" fill="#1e1e1e" text-anchor="middle" style="white-space: pre;" direction="ltr" dominant-baseline="alphabetic">Oracle</text><text x="47.40397644042969" y="59.668" font-family="Excalifont, Xiaolai, sans-serif, Segoe UI Emoji" font-size="28px" fill="#1e1e1e" text-anchor="middle" style="white-space: pre;" direction="ltr" dominant-baseline="alphabetic">Router</text></g><g stroke-linecap="round" transform="translate(1393.3146883863155 1224.4444230592017) rotate(0 89.56614344986883 42.5)"><path d="M0 0 C66.24 -2.01, 134.09 -2.39, 179.13 0 M0 0 C56.4 0.21, 114.02 -1.21, 179.13 0 M179.13 0 C180.34 19.79, 181.13 36.85, 179.13 85 M179.13 0 C178.49 28.34, 178.5 57.48, 179.13 85 M179.13 85 C120.82 85.08, 64.42 85.87, 0 85 M179.13 85 C126.52 85.9, 73.58 87.1, 0 85 M0 85 C1.24 66.13, 1.32 45.92, 0 0 M0 85 C-0.64 65.55, 0.4 48.21, 0 0" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(1409.7728835329617 1231.9444230592017) rotate(0 73.10794830322266 35)"><text x="73.10794830322266" y="24.668" font-family="Excalifont, Xiaolai, sans-serif, Segoe UI Emoji" font-size="28px" fill="#1e1e1e" text-anchor="middle" style="white-space: pre;" direction="ltr" dominant-baseline="alphabetic">Liquidation</text><text x="73.10794830322266" y="59.668" font-family="Excalifont, Xiaolai, sans-serif, Segoe UI Emoji" font-size="28px" fill="#1e1e1e" text-anchor="middle" style="white-space: pre;" direction="ltr" dominant-baseline="alphabetic">Engine</text></g><g stroke-linecap="round"><g transform="translate(889.3834128928356 1264.2700742861944) rotate(0 -56.83679589248368 -0.19731887995840225)"><path d="M0 0 C-38.14 1.71, -78.53 0.14, -113.67 -1.14 M0 0 C-44.09 -0.75, -87.76 -0.45, -113.67 -1.14" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(889.3834128928356 1264.2700742861944) rotate(0 -56.83679589248368 -0.19731887995840225)"><path d="M1.63 1.53 L-13.56 4.61 L-14.79 -4.59 L-0.02 -1.49" stroke="none" stroke-width="0" fill="#1e1e1e" fill-rule="evenodd"></path><path d="M0 0 C-3.36 3.64, -8.42 5, -13.6 6.34 M0 0 C-5.68 2.25, -11.06 5.22, -13.6 6.34 M-13.6 6.34 C-12.57 2.88, -12.91 1.58, -13.59 -6.34 M-13.6 6.34 C-13.16 2.05, -13.49 -2.33, -13.59 -6.34 M-13.59 -6.34 C-8.53 -4.37, -4.99 -3.63, 0 0 M-13.59 -6.34 C-9.65 -5.34, -7.32 -2.55, 0 0 M0 0 C0 0, 0 0, 0 0 M0 0 C0 0, 0 0, 0 0" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(889.3834128928356 1264.2700742861944) rotate(0 -56.83679589248368 -0.19731887995840225)"><path d="M-112.04 0.39 L-99.95 -9.01 L-101.37 7.15 L-113.7 -2.64" stroke="none" stroke-width="0" fill="#1e1e1e" fill-rule="evenodd"></path><path d="M-113.67 -1.14 C-107.45 -1.97, -102.9 -5, -99.99 -7.28 M-113.67 -1.14 C-108.92 -3.64, -103.87 -5.43, -99.99 -7.28 M-99.99 -7.28 C-99.03 -5.12, -99.41 -0.81, -100.18 5.4 M-99.99 -7.28 C-99.75 -3.28, -100.14 0.62, -100.18 5.4 M-100.18 5.4 C-102.47 3.83, -106.36 1.03, -113.67 -1.14 M-100.18 5.4 C-103.11 3.25, -107.74 2.73, -113.67 -1.14 M-113.67 -1.14 C-113.67 -1.14, -113.67 -1.14, -113.67 -1.14 M-113.67 -1.14 C-113.67 -1.14, -113.67 -1.14, -113.67 -1.14" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(1381.4181757892711 1266.4150035332477) rotate(0 -59.1924719917979 -0.6289807608945921)"><path d="M0 0 C-26.4 -1.74, -54.19 -1.93, -118.38 -1.01 M0 0 C-28.53 0.36, -59.51 0.71, -118.38 -1.01" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(1381.4181757892711 1266.4150035332477) rotate(0 -59.1924719917979 -0.6289807608945921)"><path d="M0.03 -1.22 L-15.08 7.53 L-12.81 -8.09 L-1.88 -0.16" stroke="none" stroke-width="0" fill="#1e1e1e" fill-rule="evenodd"></path><path d="M0 0 C-3.09 0.1, -7.27 1.47, -13.73 6.04 M0 0 C-2.86 1.34, -7.5 3.29, -13.73 6.04 M-13.73 6.04 C-14.54 3.79, -13.76 -0.65, -13.45 -6.64 M-13.73 6.04 C-13.96 3.47, -13.91 0.43, -13.45 -6.64 M-13.45 -6.64 C-7.7 -3.77, -2.89 -2.78, 0 0 M-13.45 -6.64 C-10.34 -5.38, -5.2 -2.3, 0 0 M0 0 C0 0, 0 0, 0 0 M0 0 C0 0, 0 0, 0 0" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(1381.4181757892711 1266.4150035332477) rotate(0 -59.1924719917979 -0.6289807608945921)"><path d="M-118.36 -2.23 L-106 -5.54 L-104.3 4.18 L-120.27 -1.16" stroke="none" stroke-width="0" fill="#1e1e1e" fill-rule="evenodd"></path><path d="M-118.38 -1.01 C-115.48 -3.62, -113.56 -4.92, -104.65 -7.04 M-118.38 -1.01 C-114.33 -2.88, -112.17 -3.92, -104.65 -7.04 M-104.65 -7.04 C-105.56 -3.61, -104.9 -2.38, -104.94 5.64 M-104.65 -7.04 C-104.98 -4.06, -105.05 -1.55, -104.94 5.64 M-104.94 5.64 C-108.63 3.81, -113.22 0.17, -118.38 -1.01 M-104.94 5.64 C-109.46 2.95, -112.07 2.21, -118.38 -1.01 M-118.38 -1.01 C-118.38 -1.01, -118.38 -1.01, -118.38 -1.01 M-118.38 -1.01 C-118.38 -1.01, -118.38 -1.01, -118.38 -1.01" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(968.3582380068419 1437.384593304616) rotate(0 1.089578976404482 -49.83491255620993)"><path d="M0 0 C-0.79 -30.85, 0.93 -64.18, 2.38 -99.67 M0 0 C0.86 -40.17, 1.69 -79.94, 2.38 -99.67" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(968.3582380068419 1437.384593304616) rotate(0 1.089578976404482 -49.83491255620993)"><path d="M1.57 -1.59 L-7 -14.26 L4.59 -14.18 L-1.43 -1.21" stroke="none" stroke-width="0" fill="#1e1e1e" fill-rule="evenodd"></path><path d="M0 0 C-2.61 -3.27, -3.89 -8.44, -6.12 -13.69 M0 0 C-2.84 -5.91, -5.36 -11.47, -6.12 -13.69 M-6.12 -13.69 C-1.95 -12.46, 1.11 -12.83, 6.55 -13.49 M-6.12 -13.69 C-2.79 -14, -0.36 -13.88, 6.55 -13.49 M6.55 -13.49 C4.23 -7.15, 1.37 -2.71, 0 0 M6.55 -13.49 C4.85 -9.02, 2.17 -4.08, 0 0 M0 0 C0 0, 0 0, 0 0 M0 0 C0 0, 0 0, 0 0" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(968.3582380068419 1437.384593304616) rotate(0 1.089578976404482 -49.83491255620993)"><path d="M3.95 -101.26 L7.45 -86.46 L-6.32 -86.95 L0.95 -100.88" stroke="none" stroke-width="0" fill="#1e1e1e" fill-rule="evenodd"></path><path d="M2.38 -99.67 C3.93 -94.08, 6.57 -90.32, 8.32 -85.9 M2.38 -99.67 C4.18 -94.59, 6.46 -89.23, 8.32 -85.9 M8.32 -85.9 C5.47 -84.71, 1.5 -85.23, -4.35 -86.26 M8.32 -85.9 C5.9 -86.37, 2.57 -86.38, -4.35 -86.26 M-4.35 -86.26 C-1.62 -90.36, 0.72 -96.45, 2.38 -99.67 M-4.35 -86.26 C-1.53 -90.51, 0.06 -94.21, 2.38 -99.67 M2.38 -99.67 C2.38 -99.67, 2.38 -99.67, 2.38 -99.67 M2.38 -99.67 C2.38 -99.67, 2.38 -99.67, 2.38 -99.67" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(1206.2416423385785 1439.8942520118526) rotate(0 -1.0882893153657278 -50.7815533233661)"><path d="M0 0 C-1.57 -35.31, -3.16 -69.97, -1.42 -101.56 M0 0 C-0.25 -37.91, 0.23 -74.52, -1.42 -101.56" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(1206.2416423385785 1439.8942520118526) rotate(0 -1.0882893153657278 -50.7815533233661)"><path d="M0.68 -0.2 L-6.55 -12.04 L4.67 -13.01 L-1.83 -0.01" stroke="none" stroke-width="0" fill="#1e1e1e" fill-rule="evenodd"></path><path d="M0 0 C-2.9 -4.82, -6.05 -9.14, -6.74 -13.4 M0 0 C-2.71 -5.5, -4.43 -9.97, -6.74 -13.4 M-6.74 -13.4 C-2.21 -13.6, 0.97 -12.92, 5.93 -13.78 M-6.74 -13.4 C-2.25 -13.13, 3.34 -13.4, 5.93 -13.78 M5.93 -13.78 C5.09 -9.43, 1.61 -4.5, 0 0 M5.93 -13.78 C4.01 -9.11, 1.31 -3.33, 0 0 M0 0 C0 0, 0 0, 0 0 M0 0 C0 0, 0 0, 0 0" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(1206.2416423385785 1439.8942520118526) rotate(0 -1.0882893153657278 -50.7815533233661)"><path d="M-0.74 -101.76 L5.62 -86.86 L-8.51 -86.97 L-3.24 -101.58" stroke="none" stroke-width="0" fill="#1e1e1e" fill-rule="evenodd"></path><path d="M-1.42 -101.56 C0.48 -97.09, 2.02 -92.17, 5.42 -88.21 M-1.42 -101.56 C0.66 -97.14, 3.91 -91.82, 5.42 -88.21 M5.42 -88.21 C1.9 -88.35, -2.99 -87.4, -7.25 -87.74 M5.42 -88.21 C0.08 -87.38, -4.17 -87.31, -7.25 -87.74 M-7.25 -87.74 C-3.91 -92.58, -3.45 -96.88, -1.42 -101.56 M-7.25 -87.74 C-4.63 -93.65, -2.84 -98.4, -1.42 -101.56 M-1.42 -101.56 C-1.42 -101.56, -1.42 -101.56, -1.42 -101.56 M-1.42 -101.56 C-1.42 -101.56, -1.42 -101.56, -1.42 -101.56" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(1198.3427150275966 162.31667322328576) rotate(0 -0.39670304400533496 51.461441322893734)"><path d="M0 0 C-0.24 30.94, -0.48 61.88, -0.79 102.92 M0 0 C-0.3 38.38, -0.59 76.76, -0.79 102.92" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(1198.3427150275966 162.31667322328576) rotate(0 -0.39670304400533496 51.461441322893734)"><path d="M-0.79 102.92 L-7.03 89.28 L5.65 89.38 L-0.79 102.92" stroke="none" stroke-width="0" fill="#1e1e1e" fill-rule="evenodd"></path><path d="M-0.79 102.92 C-2.67 98.82, -4.54 94.72, -7.03 89.28 M-0.79 102.92 C-3.12 97.84, -5.44 92.75, -7.03 89.28 M-7.03 89.28 C-4.32 89.3, -1.62 89.32, 5.65 89.38 M-7.03 89.28 C-3.47 89.31, 0.08 89.33, 5.65 89.38 M5.65 89.38 C3.09 94.75, 0.53 100.13, -0.79 102.92 M5.65 89.38 C3.81 93.24, 1.97 97.11, -0.79 102.92 M-0.79 102.92 C-0.79 102.92, -0.79 102.92, -0.79 102.92 M-0.79 102.92 C-0.79 102.92, -0.79 102.92, -0.79 102.92" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(1082.6858304901525 362.23955586907323) rotate(0 0.05055134095346148 33.575498791850805)"><path d="M0 0 C0.03 17.8, 0.05 35.6, 0.1 67.15 M0 0 C0.03 19.62, 0.06 39.23, 0.1 67.15" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(1082.6858304901525 362.23955586907323) rotate(0 0.05055134095346148 33.575498791850805)"><path d="M0 0 L6.36 13.59 L-6.32 13.6 L0 0" stroke="none" stroke-width="0" fill="#1e1e1e" fill-rule="evenodd"></path><path d="M0 0 C1.69 3.6, 3.37 7.2, 6.36 13.59 M0 0 C1.86 3.97, 3.72 7.94, 6.36 13.59 M6.36 13.59 C1.44 13.59, -3.48 13.6, -6.32 13.6 M6.36 13.59 C1.91 13.59, -2.53 13.6, -6.32 13.6 M-6.32 13.6 C-4.58 9.85, -2.83 6.1, 0 0 M-6.32 13.6 C-4.07 8.77, -1.83 3.94, 0 0 M0 0 C0 0, 0 0, 0 0 M0 0 C0 0, 0 0, 0 0" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(1082.6858304901525 362.23955586907323) rotate(0 0.05055134095346148 33.575498791850805)"><path d="M0.1 67.15 L-6.26 53.57 L6.42 53.55 L0.1 67.15" stroke="none" stroke-width="0" fill="#1e1e1e" fill-rule="evenodd"></path><path d="M0.1 67.15 C-1.58 63.55, -3.27 59.95, -6.26 53.57 M0.1 67.15 C-1.76 63.18, -3.61 59.21, -6.26 53.57 M-6.26 53.57 C-1.34 53.56, 3.58 53.55, 6.42 53.55 M-6.26 53.57 C-1.81 53.56, 2.64 53.55, 6.42 53.55 M6.42 53.55 C4.68 57.3, 2.93 61.06, 0.1 67.15 M6.42 53.55 C4.17 58.38, 1.93 63.21, 0.1 67.15 M0.1 67.15 C0.1 67.15, 0.1 67.15, 0.1 67.15 M0.1 67.15 C0.1 67.15, 0.1 67.15, 0.1 67.15" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(1081.4408799012635 521.897315656528) rotate(0 0.05055134095346148 33.575498791850805)"><path d="M0 0 C0.04 25.63, 0.08 51.26, 0.1 67.15 M0 0 C0.03 20.24, 0.06 40.49, 0.1 67.15" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(1081.4408799012635 521.897315656528) rotate(0 0.05055134095346148 33.575498791850805)"><path d="M0 0 L6.36 13.59 L-6.32 13.6 L0 0" stroke="none" stroke-width="0" fill="#1e1e1e" fill-rule="evenodd"></path><path d="M0 0 C2.43 5.18, 4.85 10.37, 6.36 13.59 M0 0 C1.92 4.1, 3.83 8.19, 6.36 13.59 M6.36 13.59 C2.9 13.59, -0.56 13.6, -6.32 13.6 M6.36 13.59 C3 13.59, -0.35 13.6, -6.32 13.6 M-6.32 13.6 C-4.96 10.68, -3.6 7.76, 0 0 M-6.32 13.6 C-3.8 8.18, -1.28 2.76, 0 0 M0 0 C0 0, 0 0, 0 0 M0 0 C0 0, 0 0, 0 0" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(1081.4408799012635 521.897315656528) rotate(0 0.05055134095346148 33.575498791850805)"><path d="M0.1 67.15 L-6.26 53.57 L6.42 53.55 L0.1 67.15" stroke="none" stroke-width="0" fill="#1e1e1e" fill-rule="evenodd"></path><path d="M0.1 67.15 C-2.33 61.97, -4.75 56.78, -6.26 53.57 M0.1 67.15 C-1.82 63.06, -3.73 58.96, -6.26 53.57 M-6.26 53.57 C-2.8 53.56, 0.66 53.56, 6.42 53.55 M-6.26 53.57 C-2.9 53.56, 0.45 53.56, 6.42 53.55 M6.42 53.55 C5.06 56.47, 3.7 59.39, 0.1 67.15 M6.42 53.55 C3.9 58.97, 1.38 64.39, 0.1 67.15 M0.1 67.15 C0.1 67.15, 0.1 67.15, 0.1 67.15 M0.1 67.15 C0.1 67.15, 0.1 67.15, 0.1 67.15" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(1197.6080326717674 362.23955586907323) rotate(0 0.22753616592672188 110.8482312438764)"><path d="M0 0 C0.09 45.96, 0.19 91.93, 0.46 221.7" stroke="#1e1e1e" stroke-width="2.5" fill="none" stroke-dasharray="8 10"></path></g><g transform="translate(1197.6080326717674 362.23955586907323) rotate(0 0.22753616592672188 110.8482312438764)"><path d="M0 0 L6.37 13.58 L-6.31 13.61 L0 0" stroke="none" stroke-width="0" fill="#1e1e1e" fill-rule="evenodd"></path><path d="M0 0 C1.32 2.82, 2.64 5.63, 6.37 13.58 M6.37 13.58 C2.66 13.59, -1.04 13.6, -6.31 13.61 M-6.31 13.61 C-4.52 9.74, -2.72 5.87, 0 0 M0 0 C0 0, 0 0, 0 0" stroke="#1e1e1e" stroke-width="2.5" fill="none"></path></g><g transform="translate(1197.6080326717674 362.23955586907323) rotate(0 0.22753616592672188 110.8482312438764)"><path d="M0.46 221.7 L-5.91 208.11 L6.77 208.09 L0.46 221.7" stroke="none" stroke-width="0" fill="#1e1e1e" fill-rule="evenodd"></path><path d="M0.46 221.7 C-0.87 218.88, -2.19 216.06, -5.91 208.11 M-5.91 208.11 C-2.21 208.11, 1.5 208.1, 6.77 208.09 M6.77 208.09 C4.97 211.96, 3.18 215.82, 0.46 221.7 M0.46 221.7 C0.46 221.7, 0.46 221.7, 0.46 221.7" stroke="#1e1e1e" stroke-width="2.5" fill="none"></path></g></g><mask></mask><g transform="translate(1207.2213620390867 448.08920651950575) rotate(0 53.33995819091797 25)"><text x="53.33995819091797" y="17.619999999999997" font-family="Excalifont, Xiaolai, sans-serif, Segoe UI Emoji" font-size="20px" fill="#1e1e1e" text-anchor="middle" style="white-space: pre;" direction="ltr" dominant-baseline="alphabetic">Authorized</text><text x="53.33995819091797" y="42.62" font-family="Excalifont, Xiaolai, sans-serif, Segoe UI Emoji" font-size="20px" fill="#1e1e1e" text-anchor="middle" style="white-space: pre;" direction="ltr" dominant-baseline="alphabetic">Control</text></g><g stroke-linecap="round"><g transform="translate(838.8735723915233 168.29441546531416) rotate(0 0.012327765574809746 48.420592991740705)"><path d="M0 0 C0.01 23.79, 0.01 47.59, 0.02 96.84 M0 0 C0.01 33.74, 0.02 67.48, 0.02 96.84" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(838.8735723915233 168.29441546531416) rotate(0 0.012327765574809746 48.420592991740705)"><path d="M0.02 96.84 L-6.32 83.25 L6.36 83.24 L0.02 96.84" stroke="none" stroke-width="0" fill="#1e1e1e" fill-rule="evenodd"></path><path d="M0.02 96.84 C-1.53 93.5, -3.09 90.16, -6.32 83.25 M0.02 96.84 C-2.19 92.11, -4.39 87.37, -6.32 83.25 M-6.32 83.25 C-3.33 83.25, -0.33 83.25, 6.36 83.24 M-6.32 83.25 C-2.8 83.25, 0.71 83.25, 6.36 83.24 M6.36 83.24 C4.85 86.48, 3.34 89.72, 0.02 96.84 M6.36 83.24 C4.71 86.78, 3.07 90.31, 0.02 96.84 M0.02 96.84 C0.02 96.84, 0.02 96.84, 0.02 96.84 M0.02 96.84 C0.02 96.84, 0.02 96.84, 0.02 96.84" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(844.2724732583538 362.13560144879557) rotate(0 -0.1876367332015434 204.19822073714568)"><path d="M0 0 C-1.14 109.05, -1.44 218.79, 0.02 408.4 M0 0 C1.15 124.77, 0.33 249.93, 0.02 408.4" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(844.2724732583538 362.13560144879557) rotate(0 -0.1876367332015434 204.19822073714568)"><path d="M1.99 408.58 L-7.83 395.1 L5.27 395.61 L-1.37 409.15" stroke="none" stroke-width="0" fill="#1e1e1e" fill-rule="evenodd"></path><path d="M0.02 408.4 C-0.06 404.12, -2.16 400.86, -6.29 394.79 M0.02 408.4 C-1.44 404.01, -4.5 400.12, -6.29 394.79 M-6.29 394.79 C-2.21 394.33, 0.33 393.6, 6.39 394.82 M-6.29 394.79 C-3.71 395.34, -0.56 394.97, 6.39 394.82 M6.39 394.82 C2.78 399.36, 1.6 404.05, 0.02 408.4 M6.39 394.82 C4.28 399.96, 2.16 404.94, 0.02 408.4 M0.02 408.4 C0.02 408.4, 0.02 408.4, 0.02 408.4 M0.02 408.4 C0.02 408.4, 0.02 408.4, 0.02 408.4" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(1136.4540112407976 680.936018356826) rotate(0 -0.39353551751355553 44.79801228312954)"><path d="M0 0 C-1.35 17.35, 0.6 37.28, -0.66 89.6 M0 0 C-1.52 22.74, -0.3 44.72, -0.66 89.6" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(1136.4540112407976 680.936018356826) rotate(0 -0.39353551751355553 44.79801228312954)"><path d="M0.34 91.47 L-7.39 74.62 L6.4 77.58 L1.16 91.37" stroke="none" stroke-width="0" fill="#1e1e1e" fill-rule="evenodd"></path><path d="M-0.66 89.6 C-3.28 86.38, -2.99 85.16, -6.99 76 M-0.66 89.6 C-2.9 86.7, -3.43 83.19, -6.99 76 M-6.99 76 C-1.28 77.24, 2.98 77.31, 5.69 76 M-6.99 76 C-1.86 75.77, 2.36 75.88, 5.69 76 M5.69 76 C3.43 80, 2.91 85.46, -0.66 89.6 M5.69 76 C4.3 79.32, 2.1 84.26, -0.66 89.6 M-0.66 89.6 C-0.66 89.6, -0.66 89.6, -0.66 89.6 M-0.66 89.6 C-0.66 89.6, -0.66 89.6, -0.66 89.6" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g></g><mask></mask><g stroke-linecap="round" transform="translate(713.4016275246104 776.5320429230851) rotate(0 266.23534977770214 42.5)"><path d="M0 0 C189.73 0.19, 378.82 -0.18, 532.47 0 M0 0 C145.52 0.52, 292.22 1.1, 532.47 0 M532.47 0 C530.73 32.94, 533.55 64, 532.47 85 M532.47 0 C531.47 33.4, 532.45 65.58, 532.47 85 M532.47 85 C376.87 85.89, 221.96 86.58, 0 85 M532.47 85 C382.3 86.81, 232 87.45, 0 85 M0 85 C1.1 58.91, -1.77 34.51, 0 0 M0 85 C0.44 54.2, 1.27 25.58, 0 0" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(889.9950171582695 801.5320429230851) rotate(0 89.64196014404297 17.5)"><text x="89.64196014404297" y="24.668" font-family="Excalifont, Xiaolai, sans-serif, Segoe UI Emoji" font-size="28px" fill="#1e1e1e" text-anchor="middle" style="white-space: pre;" direction="ltr" dominant-baseline="alphabetic">AmpliDex App</text></g><g stroke-linecap="round" transform="translate(713.6644043849719 931.8690603780688) rotate(0 102.72114831305134 42.5)"><path d="M0 0 C75.72 -2.16, 150.16 -0.23, 205.44 0 M0 0 C59.27 0.84, 118.85 1.38, 205.44 0 M205.44 0 C203.93 16.91, 205.36 34.12, 205.44 85 M205.44 0 C205.28 18.67, 204.95 36.35, 205.44 85 M205.44 85 C135.88 84.82, 65.27 83.9, 0 85 M205.44 85 C156.24 87.19, 109.04 86.49, 0 85 M0 85 C0.4 64.58, -0.27 45.37, 0 0 M0 85 C0.44 67.53, 0.5 50.09, 0 0" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(723.0335934695086 956.8690603780688) rotate(0 93.35195922851562 17.5)"><text x="93.35195922851562" y="24.668" font-family="Excalifont, Xiaolai, sans-serif, Segoe UI Emoji" font-size="28px" fill="#1e1e1e" text-anchor="middle" style="white-space: pre;" direction="ltr" dominant-baseline="alphabetic">AmpliDex SDK</text></g><g stroke-linecap="round" transform="translate(994.2813805700298 932.2295802867375) rotate(0 121.72249567608378 42.5)"><path d="M0 0 C96.4 -1.94, 191.96 -2.39, 243.44 0 M0 0 C70.81 -2.39, 141.06 -1.14, 243.44 0 M243.44 0 C244.1 23.53, 241.01 44.72, 243.44 85 M243.44 0 C243.61 28.56, 244.4 56.62, 243.44 85 M243.44 85 C171.36 85.82, 97.4 86.9, 0 85 M243.44 85 C176.05 84.18, 110.17 84.1, 0 85 M0 85 C-0.02 51.52, 1.16 18.05, 0 0 M0 85 C-0.08 52.51, -0.26 18.33, 0 0" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(1037.4219136606644 957.2295802867375) rotate(0 78.58196258544922 17.5)"><text x="78.58196258544922" y="24.668" font-family="Excalifont, Xiaolai, sans-serif, Segoe UI Emoji" font-size="28px" fill="#1e1e1e" text-anchor="middle" style="white-space: pre;" direction="ltr" dominant-baseline="alphabetic">Stellar RPC</text></g><g stroke-linecap="round"><g transform="translate(831.1430349874208 867.5320429230851) rotate(0 -0.2875814842300315 29.168508727491826)"><path d="M0 0 C-0.16 15.6, -1.35 30.4, 0.05 58.34 M0 0 C-0.84 21.83, -0.51 42.73, 0.05 58.34" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(831.1430349874208 867.5320429230851) rotate(0 -0.2875814842300315 29.168508727491826)"><path d="M-0.51 59.05 L-7.62 44.64 L7.23 45.87 L-0.08 57.53" stroke="none" stroke-width="0" fill="#1e1e1e" fill-rule="evenodd"></path><path d="M0.05 58.34 C-1.2 56.08, -3.74 53.19, -6.65 44.92 M0.05 58.34 C-3.08 54.11, -5.26 49.16, -6.65 44.92 M-6.65 44.92 C-3.79 45.16, 0.74 44.98, 6.02 44.58 M-6.65 44.92 C-3.04 45.3, 1.31 44.33, 6.02 44.58 M6.02 44.58 C3.75 48.98, 3.52 51.78, 0.05 58.34 M6.02 44.58 C3.88 48.37, 2.76 54.23, 0.05 58.34 M0.05 58.34 C0.05 58.34, 0.05 58.34, 0.05 58.34 M0.05 58.34 C0.05 58.34, 0.05 58.34, 0.05 58.34" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(925.1067010110728 974.2690603780684) rotate(0 31.587339779479407 0.4463634747098695)"><path d="M0 0 C19.86 1.33, 36.53 1.04, 63.17 0.36 M0 0 C21.61 -0.07, 43.96 -0.1, 63.17 0.36" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(925.1067010110728 974.2690603780684) rotate(0 31.587339779479407 0.4463634747098695)"><path d="M62.94 0.25 L51.19 4.84 L48.37 -7.71 L61.22 -0.05" stroke="none" stroke-width="0" fill="#1e1e1e" fill-rule="evenodd"></path><path d="M63.17 0.36 C59.85 2.94, 54.15 4.46, 49.48 6.48 M63.17 0.36 C58.37 2.52, 54.13 4.5, 49.48 6.48 M49.48 6.48 C50.86 4.26, 50.44 0.74, 49.69 -6.2 M49.48 6.48 C49.02 2.94, 50.18 -0.68, 49.69 -6.2 M49.69 -6.2 C56.03 -3.99, 58.82 -1.46, 63.17 0.36 M49.69 -6.2 C53.39 -4.24, 56.84 -2.5, 63.17 0.36 M63.17 0.36 C63.17 0.36, 63.17 0.36, 63.17 0.36 M63.17 0.36 C63.17 0.36, 63.17 0.36, 63.17 0.36" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(249.4616442486349 1183.6189232309753) rotate(0 168.13045940211487 -0.13072644821386348)"><path d="M0 0 C108.07 1.68, 217.2 0.51, 336.26 0.86 M0 0 C76.43 -2.22, 153.13 -1, 336.26 0.86" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(249.4616442486349 1183.6189232309753) rotate(0 168.13045940211487 -0.13072644821386348)"><path d="M337.69 0.75 L322.56 6.98 L324.62 -5.43 L337.01 2.82" stroke="none" stroke-width="0" fill="#1e1e1e" fill-rule="evenodd"></path><path d="M336.26 0.86 C330.94 3.71, 326.75 4.11, 322.6 7.06 M336.26 0.86 C332.97 1.67, 329.91 4.22, 322.6 7.06 M322.6 7.06 C323.24 3.85, 323.76 -1.14, 322.73 -5.62 M322.6 7.06 C322.67 3.18, 322.24 0.71, 322.73 -5.62 M322.73 -5.62 C326.32 -4.82, 328.67 -2.23, 336.26 0.86 M322.73 -5.62 C327.56 -3.03, 331.34 -1.68, 336.26 0.86 M336.26 0.86 C336.26 0.86, 336.26 0.86, 336.26 0.86 M336.26 0.86 C336.26 0.86, 336.26 0.86, 336.26 0.86" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(585.7225630528646 1218.3463520040805) rotate(0 -168.13045940211487 0.3025342412192913)"><path d="M0 0 C-88.04 2.6, -175.39 1.65, -336.26 -0.93 M0 0 C-91.5 -0.31, -181.9 0.07, -336.26 -0.93" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(585.7225630528646 1218.3463520040805) rotate(0 -168.13045940211487 0.3025342412192913)"><path d="M-336.48 -2.92 L-322.55 -8.95 L-322.34 3.54 L-335.9 -0.48" stroke="none" stroke-width="0" fill="#1e1e1e" fill-rule="evenodd"></path><path d="M-336.26 -0.93 C-333.2 -1.99, -329.36 -4.4, -322.64 -7.21 M-336.26 -0.93 C-333.03 -3.23, -328.52 -4.23, -322.64 -7.21 M-322.64 -7.21 C-322.08 -3.36, -323.32 2.07, -322.7 5.47 M-322.64 -7.21 C-322.21 -4.25, -323.03 -1.45, -322.7 5.47 M-322.7 5.47 C-324.31 4.35, -327.78 3.04, -336.26 -0.93 M-322.7 5.47 C-325.73 4.4, -329.11 2.39, -336.26 -0.93 M-336.26 -0.93 C-336.26 -0.93, -336.26 -0.93, -336.26 -0.93 M-336.26 -0.93 C-336.26 -0.93, -336.26 -0.93, -336.26 -0.93" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(585.7225630528646 1355.2942174949058) rotate(0 -52.49989303613438 0.6393328746335101)"><path d="M0 0 C-22.83 0.76, -44.88 2.11, -105 0.58 M0 0 C-39.9 0.39, -77.12 0.37, -105 0.58" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(585.7225630528646 1355.2942174949058) rotate(0 -52.49989303613438 0.6393328746335101)"><path d="M-103.05 0.5 L-91.15 -7.14 L-92.14 6.56 L-105.9 0.29" stroke="none" stroke-width="0" fill="#1e1e1e" fill-rule="evenodd"></path><path d="M-105 0.58 C-102.72 -0.76, -99.83 -1.19, -91.44 -5.83 M-105 0.58 C-100.67 -0.94, -94.3 -3.49, -91.44 -5.83 M-91.44 -5.83 C-92.22 -0.13, -91.19 2.96, -91.37 6.85 M-91.44 -5.83 C-90.74 -2, -91.59 0.98, -91.37 6.85 M-91.37 6.85 C-97.33 5.5, -99.31 1.69, -105 0.58 M-91.37 6.85 C-94.88 5.07, -99.63 3.38, -105 0.58 M-105 0.58 C-105 0.58, -105 0.58, -105 0.58 M-105 0.58 C-105 0.58, -105 0.58, -105 0.58" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(474.72350643628306 1505.561672615122) rotate(0 55.49952830829079 0.8901259431286235)"><path d="M0 0 C36.69 1.07, 77.85 2.9, 111 0.88 M0 0 C41.14 -0.1, 84.85 0.87, 111 0.88" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(474.72350643628306 1505.561672615122) rotate(0 55.49952830829079 0.8901259431286235)"><path d="M112.17 2.29 L97.69 7.99 L98.25 -5.37 L110.39 2.55" stroke="none" stroke-width="0" fill="#1e1e1e" fill-rule="evenodd"></path><path d="M111 0.88 C104.91 3.29, 102.12 6.6, 97.36 7.11 M111 0.88 C105.07 2.81, 101.13 5.65, 97.36 7.11 M97.36 7.11 C96.34 4.28, 98.42 -0.01, 97.45 -5.57 M97.36 7.11 C97.41 4.12, 97.04 2.16, 97.45 -5.57 M97.45 -5.57 C104.06 -1.77, 108.36 -1.74, 111 0.88 M97.45 -5.57 C100.08 -4.56, 103.41 -2.59, 111 0.88 M111 0.88 C111 0.88, 111 0.88, 111 0.88 M111 0.88 C111 0.88, 111 0.88, 111 0.88" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g></g><mask></mask><g stroke-linecap="round" transform="translate(56.100291695809574 1298.5689311909337) rotate(0 79.81481856647406 51.72063201423589)"><path d="M-1.71 1.52 L160.23 -1.59 L157.72 104.31 L-0.42 103.28" stroke="none" stroke-width="0" fill="#a5d8ff"></path><path d="M0 0 C47.23 0.17, 93.22 -1.57, 159.63 0 M0 0 C33.69 0.1, 67.42 -0.28, 159.63 0 M159.63 0 C157.79 24.44, 159.34 49.67, 159.63 103.44 M159.63 0 C159.43 40.9, 159.66 81.36, 159.63 103.44 M159.63 103.44 C115.15 105.13, 73.4 103.5, 0 103.44 M159.63 103.44 C116.79 102.99, 73 101.35, 0 103.44 M0 103.44 C0.77 60.73, -2.35 18.9, 0 0 M0 103.44 C-1.36 72.69, 0.4 41.18, 0 0" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(89.49052682722504 1319.361486422491) rotate(0 46.424583435058594 30.928076782678545)"><text x="46.424583435058594" y="21.79810851643161" font-family="Excalifont, Xiaolai, sans-serif, Segoe UI Emoji" font-size="24.74246142614258px" fill="#1e1e1e" text-anchor="middle" style="white-space: pre;" direction="ltr" dominant-baseline="alphabetic">Event</text><text x="46.424583435058594" y="52.72618529910984" font-family="Excalifont, Xiaolai, sans-serif, Segoe UI Emoji" font-size="24.74246142614258px" fill="#1e1e1e" text-anchor="middle" style="white-space: pre;" direction="ltr" dominant-baseline="alphabetic">Indexer</text></g><g stroke-linecap="round" transform="translate(55.437106408196996 1382.766620005581) rotate(0 79.8228984277639 18.928584814773785)"><path d="M69.16 0 C80.51 -0.72, 95 -0.23, 106.47 0.9 C117.94 2.03, 129.55 4.7, 137.96 6.77 C146.37 8.84, 153.81 10.7, 156.95 13.3 C160.09 15.9, 159.41 19.41, 156.8 22.37 C154.18 25.33, 149.07 28.67, 141.24 31.08 C133.42 33.49, 121.31 35.61, 109.83 36.85 C98.34 38.08, 84.55 38.78, 72.34 38.49 C60.14 38.19, 47.02 36.72, 36.6 35.08 C26.18 33.44, 15.9 31.4, 9.8 28.67 C3.71 25.95, 0.29 21.78, 0.05 18.76 C-0.2 15.73, 2.77 13.14, 8.33 10.51 C13.89 7.89, 20.48 4.9, 33.41 3.01 C46.34 1.13, 74.55 -0.43, 85.9 -0.79 C97.25 -1.15, 101.59 0.44, 101.53 0.85 M59.11 1.35 C70.26 0.3, 86.27 -1.09, 98.23 -0.64 C110.2 -0.19, 121.78 1.99, 130.88 4.07 C139.98 6.14, 148.18 9.03, 152.85 11.82 C157.52 14.61, 160.24 17.69, 158.91 20.79 C157.59 23.89, 151.74 28.06, 144.92 30.41 C138.09 32.76, 128.51 33.67, 117.97 34.9 C107.42 36.14, 93.89 37.57, 81.65 37.81 C69.4 38.06, 55.94 37.73, 44.5 36.36 C33.07 35, 20.5 32.28, 13.01 29.64 C5.52 27.01, 1.08 23.57, -0.44 20.56 C-1.96 17.56, -0.4 14.46, 3.9 11.6 C8.2 8.74, 15.84 5.44, 25.35 3.42 C34.86 1.39, 55.05 -0.31, 60.95 -0.54 C66.84 -0.78, 60.67 1.31, 60.72 2.03" stroke="none" stroke-width="0" fill="#a5d8ff"></path><path d="M111.24 1.52 C122.7 2.43, 134.78 4.63, 142.62 6.83 C150.46 9.03, 155.99 11.93, 158.3 14.73 C160.61 17.52, 160.04 20.73, 156.48 23.63 C152.92 26.52, 145.64 30.04, 136.95 32.11 C128.26 34.17, 116.08 35.12, 104.32 36.01 C92.57 36.91, 78.49 37.84, 66.43 37.47 C54.37 37.11, 41.73 35.69, 31.97 33.81 C22.22 31.93, 13.08 28.81, 7.92 26.17 C2.76 23.53, 0.35 20.8, 1.02 17.97 C1.69 15.13, 5.73 11.76, 11.93 9.15 C18.13 6.55, 27.83 3.7, 38.22 2.35 C48.6 1, 60.91 1, 74.27 1.04 C87.64 1.07, 110.17 2.17, 118.41 2.55 C126.65 2.94, 124.59 3.17, 123.72 3.36 M64.87 0.21 C76.25 -0.61, 89.45 -0.15, 101.32 0.6 C113.19 1.34, 127.2 2.74, 136.08 4.66 C144.95 6.58, 150.81 9.19, 154.55 12.13 C158.29 15.08, 160.25 19.24, 158.5 22.32 C156.76 25.4, 151.34 28.16, 144.07 30.6 C136.8 33.03, 126.19 35.46, 114.88 36.91 C103.57 38.37, 88.62 39.42, 76.19 39.32 C63.76 39.23, 50.74 38.04, 40.3 36.33 C29.86 34.62, 20.04 31.75, 13.53 29.06 C7.01 26.36, 2.26 22.91, 1.19 20.17 C0.13 17.43, 2.67 15.1, 7.13 12.62 C11.6 10.13, 18.6 7.28, 27.98 5.25 C37.35 3.21, 57.66 1.34, 63.4 0.41 C69.14 -0.53, 61.87 -0.69, 62.4 -0.36" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g stroke-linecap="round" transform="translate(55.857342747362054 1286.1224363918518) rotate(0 80.68027265904675 15.132464801675269)"><path d="M117.63 2.39 C128.82 3.18, 139.06 3.92, 146.3 5.68 C153.54 7.45, 159.39 10.54, 161.07 12.97 C162.75 15.41, 160.7 17.95, 156.38 20.29 C152.06 22.63, 144.22 25.58, 135.17 27.03 C126.12 28.48, 114.33 28.61, 102.09 28.97 C89.85 29.33, 74.04 29.54, 61.71 29.21 C49.39 28.88, 37.37 28.47, 28.14 26.97 C18.92 25.47, 11.09 22.35, 6.38 20.21 C1.66 18.07, -1.2 16.26, -0.15 14.12 C0.89 11.99, 5.51 9.34, 12.63 7.41 C19.75 5.48, 31.45 3.77, 42.56 2.54 C53.66 1.31, 65.69 0.2, 79.26 0.03 C92.84 -0.14, 115.65 0.91, 124.03 1.55 C132.41 2.18, 130.23 3.46, 129.54 3.84 M71.66 1.45 C83.23 0.75, 98.21 -0.5, 110.17 0.19 C122.13 0.88, 134.97 3.83, 143.4 5.58 C151.83 7.34, 158.31 8.52, 160.75 10.73 C163.19 12.93, 161.61 16.51, 158.02 18.81 C154.42 21.11, 147.46 22.91, 139.2 24.54 C130.93 26.17, 119.97 27.51, 108.43 28.57 C96.88 29.63, 82.46 31.14, 69.93 30.91 C57.41 30.67, 43.39 28.53, 33.29 27.16 C23.19 25.79, 14.84 24.97, 9.33 22.69 C3.82 20.41, -0.06 15.77, 0.23 13.47 C0.52 11.18, 5.21 10.58, 11.07 8.92 C16.93 7.27, 25.31 5.09, 35.39 3.55 C45.48 2.01, 65.47 0.35, 71.6 -0.32 C77.73 -0.99, 71.76 -0.68, 72.19 -0.47" stroke="none" stroke-width="0" fill="#a5d8ff"></path><path d="M91.67 0.62 C103.55 0.58, 117.29 1.12, 127.44 2.31 C137.59 3.5, 147.02 5.44, 152.58 7.75 C158.15 10.06, 161.17 13.9, 160.83 16.19 C160.48 18.49, 156.58 19.68, 150.52 21.53 C144.46 23.37, 134.76 25.93, 124.45 27.26 C114.15 28.59, 101.01 29.07, 88.71 29.51 C76.41 29.95, 62.37 30.76, 50.67 29.9 C38.97 29.05, 26.53 26.41, 18.49 24.36 C10.46 22.31, 5.09 19.73, 2.47 17.61 C-0.16 15.5, -0.48 13.93, 2.75 11.67 C5.97 9.41, 13.29 5.98, 21.82 4.05 C30.34 2.11, 41.08 0.51, 53.9 0.05 C66.72 -0.4, 90.08 1.13, 98.73 1.32 C107.38 1.51, 106.08 1.04, 105.8 1.18 M108.7 -0.59 C120.64 0.32, 133.09 3.65, 141.2 5.64 C149.31 7.64, 154.36 9.09, 157.37 11.39 C160.37 13.69, 161.57 17.05, 159.23 19.44 C156.89 21.84, 151.41 24.36, 143.31 25.77 C135.21 27.19, 122.39 27.15, 110.61 27.95 C98.84 28.74, 85.13 30.8, 72.64 30.55 C60.16 30.31, 46.34 27.74, 35.7 26.48 C25.06 25.23, 14.8 24.63, 8.82 23.03 C2.84 21.42, -0.19 19.39, -0.2 16.84 C-0.21 14.28, 2.97 9.96, 8.76 7.69 C14.56 5.41, 24.32 4.56, 34.56 3.18 C44.79 1.81, 57.93 -0.17, 70.15 -0.56 C82.38 -0.94, 101.83 0.72, 107.91 0.88 C113.99 1.04, 107.16 0.02, 106.62 0.4" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g stroke-linecap="round"><g transform="translate(215.45483053440512 1304.4011175366813) rotate(0 -79.56292820173348 47.75480606491146)" fill-rule="evenodd"><path d="M0 0 L0.28 97.6 L-2.12 93.36 L-5.19 90.82 L-19.05 85.13 L-45.2 80.28 L-71.85 78.47 L-99.51 78.93 L-129.34 82.38 L-147.89 87.26 L-155.2 90.82 L-159.24 94.66 L-159.4 -2.09 L-157.83 0 L-152.62 3.01 L-136.95 7.37 L-114.29 10.45 L-87.35 11.9 L-59.4 11.54 L-33.8 9.4 L-13.65 5.75 L-1.36 1.02 L0 0" stroke="none" stroke-width="0" fill="#a5d8ff" fill-rule="evenodd"></path><path d="M0 0 C0.1 34.72, 0.2 69.45, 0.28 97.6 M0 0 C0.07 24.35, 0.14 48.7, 0.28 97.6 M0.28 97.6 C-0.21 96.75, -0.69 95.89, -2.12 93.36 M0.28 97.6 C-0.66 95.94, -1.6 94.27, -2.12 93.36 M-2.12 93.36 C-3.1 92.54, -4.09 91.73, -5.19 90.82 M-2.12 93.36 C-3.17 92.49, -4.23 91.61, -5.19 90.82 M-5.19 90.82 C-9.9 88.89, -14.61 86.95, -19.05 85.13 M-5.19 90.82 C-8.16 89.6, -11.14 88.37, -19.05 85.13 M-19.05 85.13 C-26.95 83.66, -34.85 82.2, -45.2 80.28 M-19.05 85.13 C-25.97 83.84, -32.9 82.56, -45.2 80.28 M-45.2 80.28 C-51.49 79.85, -57.77 79.43, -71.85 78.47 M-45.2 80.28 C-54.2 79.67, -63.2 79.06, -71.85 78.47 M-71.85 78.47 C-80.56 78.61, -89.27 78.76, -99.51 78.93 M-71.85 78.47 C-80.17 78.61, -88.5 78.75, -99.51 78.93 M-99.51 78.93 C-106.11 79.69, -112.72 80.46, -129.34 82.38 M-99.51 78.93 C-107.94 79.9, -116.37 80.88, -129.34 82.38 M-129.34 82.38 C-135.06 83.88, -140.77 85.39, -147.89 87.26 M-129.34 82.38 C-135.06 83.89, -140.79 85.39, -147.89 87.26 M-147.89 87.26 C-150.53 88.55, -153.17 89.83, -155.2 90.82 M-147.89 87.26 C-150.38 88.47, -152.87 89.69, -155.2 90.82 M-155.2 90.82 C-156.66 92.21, -158.12 93.59, -159.24 94.66 M-155.2 90.82 C-156.47 92.02, -157.73 93.22, -159.24 94.66 M-159.24 94.66 C-159.28 70.57, -159.32 46.49, -159.4 -2.09 M-159.24 94.66 C-159.3 58.74, -159.36 22.81, -159.4 -2.09 M-159.4 -2.09 C-158.96 -1.5, -158.51 -0.9, -157.83 0 M-159.4 -2.09 C-158.92 -1.45, -158.43 -0.8, -157.83 0 M-157.83 0 C-156.73 0.64, -155.63 1.28, -152.62 3.01 M-157.83 0 C-156.78 0.61, -155.72 1.22, -152.62 3.01 M-152.62 3.01 C-149.17 3.97, -145.72 4.93, -136.95 7.37 M-152.62 3.01 C-148.59 4.13, -144.56 5.25, -136.95 7.37 M-136.95 7.37 C-129.99 8.31, -123.02 9.26, -114.29 10.45 M-136.95 7.37 C-131.66 8.09, -126.38 8.81, -114.29 10.45 M-114.29 10.45 C-107.79 10.8, -101.29 11.15, -87.35 11.9 M-114.29 10.45 C-106.43 10.88, -98.58 11.3, -87.35 11.9 M-87.35 11.9 C-78.3 11.78, -69.25 11.67, -59.4 11.54 M-87.35 11.9 C-80.99 11.82, -74.62 11.74, -59.4 11.54 M-59.4 11.54 C-50.39 10.78, -41.37 10.03, -33.8 9.4 M-59.4 11.54 C-51.44 10.87, -43.48 10.21, -33.8 9.4 M-33.8 9.4 C-26.11 8.01, -18.42 6.61, -13.65 5.75 M-33.8 9.4 C-27.33 8.23, -20.86 7.06, -13.65 5.75 M-13.65 5.75 C-10.1 4.39, -6.56 3.02, -1.36 1.02 M-13.65 5.75 C-8.91 3.93, -4.18 2.11, -1.36 1.02 M-1.36 1.02 C-0.91 0.68, -0.46 0.34, 0 0 M-1.36 1.02 C-1.07 0.8, -0.77 0.58, 0 0 M0 0 C0 0, 0 0, 0 0 M0 0 C0 0, 0 0, 0 0" stroke="transparent" stroke-width="1" fill="none"></path></g></g><mask></mask><g stroke-linecap="round" transform="translate(263.28120981018037 1454.7272090833194) rotate(0 102.72114831305134 42.5)"><path d="M0 0 C61.34 0.84, 118.4 0.35, 205.44 0 M0 0 C60.59 1.25, 121.77 1, 205.44 0 M205.44 0 C206.09 26.15, 205.65 50.77, 205.44 85 M205.44 0 C205.23 30.39, 205.78 61.79, 205.44 85 M205.44 85 C158.32 85.94, 112.38 83.82, 0 85 M205.44 85 C160.95 85.6, 115.42 86.54, 0 85 M0 85 C-1.73 61.52, -0.23 33.04, 0 0 M0 85 C-0.15 58.48, -0.82 31.91, 0 0" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(311.61237399237325 1462.2272090833194) rotate(0 54.389984130859375 35)"><text x="54.389984130859375" y="24.668" font-family="Excalifont, Xiaolai, sans-serif, Segoe UI Emoji" font-size="28px" fill="#1e1e1e" text-anchor="middle" style="white-space: pre;" direction="ltr" dominant-baseline="alphabetic">Keeper</text><text x="54.389984130859375" y="59.668" font-family="Excalifont, Xiaolai, sans-serif, Segoe UI Emoji" font-size="28px" fill="#1e1e1e" text-anchor="middle" style="white-space: pre;" direction="ltr" dominant-baseline="alphabetic">Watcher</text></g><g stroke-linecap="round" transform="translate(269.280480354495 1313.478105496033) rotate(0 102.72114831305134 42.5)"><path d="M0 0 C73.74 1.12, 147.88 0.32, 205.44 0 M0 0 C47.14 -1.85, 95.6 -1.6, 205.44 0 M205.44 0 C204.78 27.49, 206.69 53.72, 205.44 85 M205.44 0 C204.47 32.65, 205.53 64.25, 205.44 85 M205.44 85 C152.24 85.02, 100.45 85.37, 0 85 M205.44 85 C145.76 83.15, 85.72 83.3, 0 85 M0 85 C1.23 59.1, -0.06 29.19, 0 0 M0 85 C0.21 51.28, 0.64 19.04, 0 0" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(319.4596517388363 1320.978105496033) rotate(0 52.54197692871094 35)"><text x="52.54197692871094" y="24.668" font-family="Excalifont, Xiaolai, sans-serif, Segoe UI Emoji" font-size="28px" fill="#1e1e1e" text-anchor="middle" style="white-space: pre;" direction="ltr" dominant-baseline="alphabetic">Event</text><text x="52.54197692871094" y="59.668" font-family="Excalifont, Xiaolai, sans-serif, Segoe UI Emoji" font-size="28px" fill="#1e1e1e" text-anchor="middle" style="white-space: pre;" direction="ltr" dominant-baseline="alphabetic">Indexer</text></g><g stroke-linecap="round" transform="translate(38.01934762253222 1153.2889696155144) rotate(0 102.72114831305134 42.5)"><path d="M0 0 C73.11 1.28, 141.73 2.08, 205.44 0 M0 0 C74.79 -0.76, 148.97 0.31, 205.44 0 M205.44 0 C206.96 21.87, 205.37 40.71, 205.44 85 M205.44 0 C206.16 28.23, 204.67 57.75, 205.44 85 M205.44 85 C161.92 86.83, 118.42 87.48, 0 85 M205.44 85 C124.1 86.2, 43.89 86.22, 0 85 M0 85 C1 50.96, -1.81 20.03, 0 0 M0 85 C1.03 64.88, 0.66 43.28, 0 0" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(69.14451815238135 1178.2889696155144) rotate(0 71.59597778320312 17.5)"><text x="71.59597778320312" y="24.668" font-family="Excalifont, Xiaolai, sans-serif, Segoe UI Emoji" font-size="28px" fill="#1e1e1e" text-anchor="middle" style="white-space: pre;" direction="ltr" dominant-baseline="alphabetic">Query API</text></g><g transform="translate(79.53379622324246 1339.819458462527) rotate(0 57.80995178222656 12.5)"><text x="57.80995178222656" y="17.619999999999997" font-family="Excalifont, Xiaolai, sans-serif, Segoe UI Emoji" font-size="20px" fill="#1e1e1e" text-anchor="middle" style="white-space: pre;" direction="ltr" dominant-baseline="alphabetic">PostgreSQL</text></g><g stroke-linecap="round"><g transform="translate(263.280480354495 1355.878105496031) rotate(0 -20.77527576286775 0.2874807286152645)"><path d="M0 0 C-14.12 0.65, -29.84 0.22, -41.55 0.57 M0 0 C-12.96 0.34, -25.94 0.59, -41.55 0.57" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(263.280480354495 1355.878105496031) rotate(0 -20.77527576286775 0.2874807286152645)"><path d="M-41.87 1.29 L-27.77 -5.9 L-27.51 6.63 L-41.57 -0.27" stroke="none" stroke-width="0" fill="#1e1e1e" fill-rule="evenodd"></path><path d="M-41.55 0.57 C-36.51 -1.1, -32.64 -3.81, -27.98 -5.82 M-41.55 0.57 C-37.11 -1.51, -32.73 -3.47, -27.98 -5.82 M-27.98 -5.82 C-28.18 -1.59, -27.77 1.7, -27.93 6.86 M-27.98 -5.82 C-28 -2.52, -27.74 1.14, -27.93 6.86 M-27.93 6.86 C-32.25 4.85, -36.33 2.99, -41.55 0.57 M-27.93 6.86 C-32.98 4.68, -38.29 1.92, -41.55 0.57 M-41.55 0.57 C-41.55 0.57, -41.55 0.57, -41.55 0.57 M-41.55 0.57 C-41.55 0.57, -41.55 0.57, -41.55 0.57" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(135.16000483595963 1426.62377271275) rotate(0 60.67247124594451 35.75669622913756)"><path d="M0 0 C0.47 15.99, -1.96 33.11, 0 54.5 M0 0 C-0.08 12.45, 0.28 25.37, 0 54.5 M0 54.5 C-0.25 63.92, 4.16 68.75, 16 70.5 M0 54.5 C-1.34 65.88, 4.09 71.2, 16 70.5 M16 70.5 C38.17 69.75, 60.8 73.12, 122.12 70.5 M16 70.5 C46.63 71.85, 76.83 70.51, 122.12 70.5" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(135.16000483595963 1426.62377271275) rotate(0 60.67247124594451 35.75669622913756)"><path d="M121.02 68.74 L106.75 75.89 L108.35 66.21 L120.91 68.91" stroke="none" stroke-width="0" fill="#1e1e1e" fill-rule="evenodd"></path><path d="M122.12 70.5 C118.82 72.42, 113 75.1, 108.57 76.94 M122.12 70.5 C118.73 72.08, 115.96 74.04, 108.57 76.94 M108.57 76.94 C107.51 72.55, 108.99 68.95, 108.48 64.26 M108.57 76.94 C108.1 72.81, 108.09 68.84, 108.48 64.26 M108.48 64.26 C112.61 67.36, 119.79 69.71, 122.12 70.5 M108.48 64.26 C112.11 65.8, 115.2 66.51, 122.12 70.5 M122.12 70.5 C122.12 70.5, 122.12 70.5, 122.12 70.5 M122.12 70.5 C122.12 70.5, 122.12 70.5, 122.12 70.5" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(1079.86177247446 1050.9992934663642) rotate(0 -0.10575418572534545 53.435452989881014)"><path d="M0 0 C0.73 26.86, -2.32 50.86, 0.31 106.87 M0 0 C-0.37 32.04, 1.16 62.47, 0.31 106.87" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(1079.86177247446 1050.9992934663642) rotate(0 -0.10575418572534545 53.435452989881014)"><path d="M1.46 105.17 L-4.33 94.44 L6.17 95.21 L-1.34 106.88" stroke="none" stroke-width="0" fill="#1e1e1e" fill-rule="evenodd"></path><path d="M0.31 106.87 C0.18 103.96, -3.71 98.9, -5.94 93.24 M0.31 106.87 C-1.91 103.16, -2.68 98.23, -5.94 93.24 M-5.94 93.24 C-1.8 92.63, 1.43 93.48, 6.74 93.32 M-5.94 93.24 C-1.41 93.16, 3.32 93.08, 6.74 93.32 M6.74 93.32 C4.87 99.05, 0.46 104.79, 0.31 106.87 M6.74 93.32 C4.33 97.1, 3.49 102.11, 0.31 106.87 M0.31 106.87 C0.31 106.87, 0.31 106.87, 0.31 106.87 M0.31 106.87 C0.31 106.87, 0.31 106.87, 0.31 106.87" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(1112.1202322351583 1050.9992934663642) rotate(0 -0.29622855507204804 53.435452989881014)"><path d="M0 0 C-0.21 43.41, -0.57 82.13, -0.15 106.87 M0 0 C-1.42 30.48, 0.21 60.37, -0.15 106.87" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(1112.1202322351583 1050.9992934663642) rotate(0 -0.29622855507204804 53.435452989881014)"><path d="M-1.47 -1.77 L6.41 13.5 L-6.8 15.23 L-1.99 -1.56" stroke="none" stroke-width="0" fill="#1e1e1e" fill-rule="evenodd"></path><path d="M0 0 C2.73 6.54, 4.96 9.53, 6.28 13.62 M0 0 C1 3.81, 4.05 7.21, 6.28 13.62 M6.28 13.62 C4.5 13.52, 1.52 13.98, -6.4 13.57 M6.28 13.62 C1.77 14.15, -2.07 13.46, -6.4 13.57 M-6.4 13.57 C-3.35 9.63, -3.1 5.46, 0 0 M-6.4 13.57 C-5.09 9.56, -2.64 7.56, 0 0 M0 0 C0 0, 0 0, 0 0 M0 0 C0 0, 0 0, 0 0" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(256.1825703546965 1126.7719909708412) rotate(0 112.72737799909828 -130.01448913524837)"><path d="M0 0 C-0.07 -52.44, -0.34 -102.25, 0 -243.98 M0 0 C0.86 -69.27, 1.99 -137.42, 0 -243.98 M0 -243.98 C-0.21 -254.15, 4.67 -259.11, 16 -259.98 M0 -243.98 C-0.04 -255.52, 4.65 -260.58, 16 -259.98 M16 -259.98 C75.92 -259.63, 140.36 -258.03, 225.63 -259.98 M16 -259.98 C74.01 -259.59, 132.6 -259, 225.63 -259.98" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(256.1825703546965 1126.7719909708412) rotate(0 112.72737799909828 -130.01448913524837)"><path d="M226.37 -261.78 L212.43 -253.69 L210.28 -264.36 L224.15 -261.93" stroke="none" stroke-width="0" fill="#1e1e1e" fill-rule="evenodd"></path><path d="M225.63 -259.98 C222.88 -259.27, 219.76 -256.04, 212.07 -253.55 M225.63 -259.98 C221.19 -258.75, 218.4 -256.39, 212.07 -253.55 M212.07 -253.55 C212.57 -257.25, 211.51 -261.27, 211.99 -266.23 M212.07 -253.55 C211.62 -257.52, 211.7 -259.87, 211.99 -266.23 M211.99 -266.23 C216.24 -263.87, 221.13 -260.57, 225.63 -259.98 M211.99 -266.23 C217.55 -264.58, 221.17 -261.64, 225.63 -259.98 M225.63 -259.98 C225.63 -259.98, 225.63 -259.98, 225.63 -259.98 M225.63 -259.98 C225.63 -259.98, 225.63 -259.98, 225.63 -259.98" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(301.1474174181749 1126.7719909708412) rotate(0 89.9835509732111 -106.23031624413943)"><path d="M0 0 C1.4 -66.08, -0.27 -129.37, 0 -196.2 M0 0 C-1.21 -75.04, -0.59 -152.37, 0 -196.2 M0 -196.2 C-1.98 -207.99, 6.58 -213.81, 16 -212.2 M0 -196.2 C-0.84 -209.08, 3.23 -211.14, 16 -212.2 M16 -212.2 C75.98 -212.22, 134.97 -210.16, 180.66 -212.2 M16 -212.2 C81.17 -210.61, 145.49 -210.29, 180.66 -212.2" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(301.1474174181749 1126.7719909708412) rotate(0 89.9835509732111 -106.23031624413943)"><path d="M0.42 -1.35 L-5.08 -15.15 L7.52 -14.64 L-1.81 1.06" stroke="none" stroke-width="0" fill="#1e1e1e" fill-rule="evenodd"></path><path d="M0 0 C-1.77 -5.54, -5.1 -8.96, -6.32 -13.6 M0 0 C-3.13 -4.62, -5.11 -10.94, -6.32 -13.6 M-6.32 -13.6 C-4.8 -14.14, -2.45 -14.84, 6.35 -13.59 M-6.32 -13.6 C-2.52 -13.36, 1.96 -13.49, 6.35 -13.59 M6.35 -13.59 C3.08 -9.19, 2.53 -6.11, 0 0 M6.35 -13.59 C3.54 -9.51, 2.37 -3.49, 0 0 M0 0 C0 0, 0 0, 0 0 M0 0 C0 0, 0 0, 0 0" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g></g><mask></mask><g stroke-linecap="round" transform="translate(1104.818990893611 71.31667322328576) rotate(0 93.64398670066112 42.5)"><path d="M0 0 C49.53 -1.92, 100.88 0.76, 187.29 0 M0 0 C54.37 1.86, 109.86 0.63, 187.29 0 M187.29 0 C189.34 19.43, 186.49 39.22, 187.29 85 M187.29 0 C186.06 18.66, 187.7 35.59, 187.29 85 M187.29 85 C136 83.6, 81.02 85.08, 0 85 M187.29 85 C133.34 84.32, 78.83 85.34, 0 85 M0 85 C1.67 63.75, -0.22 43.25, 0 0 M0 85 C0.96 68.1, -0.19 49.66, 0 0" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(1125.2430068911472 96.31667322328576) rotate(0 73.219970703125 17.5)"><text x="73.219970703125" y="24.668" font-family="Excalifont, Xiaolai, sans-serif, Segoe UI Emoji" font-size="28px" fill="#1e1e1e" text-anchor="middle" style="white-space: pre;" direction="ltr" dominant-baseline="alphabetic">EVM Users</text></g><g stroke-linecap="round" transform="translate(745.3295856908626 77.29441546531234) rotate(0 93.64398670066112 42.5)"><path d="M0 0 C52.43 1.61, 102.66 1.04, 187.29 0 M0 0 C42.94 -1.55, 85.32 -2.35, 187.29 0 M187.29 0 C188.11 32.14, 186.81 68.97, 187.29 85 M187.29 0 C187.11 22.05, 186.93 44, 187.29 85 M187.29 85 C114.88 85.95, 44.31 82.49, 0 85 M187.29 85 C130.22 86.12, 72.48 86.01, 0 85 M0 85 C-0.87 66.21, -1.84 43.31, 0 0 M0 85 C-0.77 61.69, -1.44 37.55, 0 0" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(750.5636145057815 102.29441546531234) rotate(0 88.40995788574219 17.5)"><text x="88.40995788574219" y="24.668" font-family="Excalifont, Xiaolai, sans-serif, Segoe UI Emoji" font-size="28px" fill="#1e1e1e" text-anchor="middle" style="white-space: pre;" direction="ltr" dominant-baseline="alphabetic">Stellar Users</text></g><g stroke-opacity="0.4" fill-opacity="0.4" stroke-linecap="round" transform="translate(683.4529243347024 41.09853758445752) rotate(0 334.422881864838 71.08334057855791)"><path d="M0 0 C241.08 0, 482.15 0, 668.85 0 M0 0 C153.94 0, 307.88 0, 668.85 0 M668.85 0 C668.85 29.6, 668.85 59.19, 668.85 142.17 M668.85 0 C668.85 37.18, 668.85 74.37, 668.85 142.17 M668.85 142.17 C445.22 142.17, 221.6 142.17, 0 142.17 M668.85 142.17 C424.47 142.17, 180.09 142.17, 0 142.17 M0 142.17 C0 96.73, 0 51.29, 0 0 M0 142.17 C0 93.72, 0 45.28, 0 0" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g stroke-opacity="0.4" fill-opacity="0.4" stroke-linecap="round" transform="translate(487.8079579214027 742.0641993998543) rotate(0 492.9866587492197 151.46754703325496)"><path d="M0 0 C334.63 0, 669.25 0, 985.97 0 M0 0 C350.83 0, 701.66 0, 985.97 0 M985.97 0 C985.97 87.18, 985.97 174.35, 985.97 302.94 M985.97 0 C985.97 68.65, 985.97 137.29, 985.97 302.94 M985.97 302.94 C763.67 302.94, 541.36 302.94, 0 302.94 M985.97 302.94 C684.75 302.94, 383.53 302.94, 0 302.94 M0 302.94 C0 184.77, 0 66.6, 0 0 M0 302.94 C0 209.64, 0 116.35, 0 0" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g stroke-opacity="0.4" fill-opacity="0.4" stroke-linecap="round" transform="translate(10 1132.7719909708394) rotate(0 246.28257035469687 221.40671422047126)"><path d="M0 0 C188.48 0, 376.96 0, 492.57 0 M0 0 C101.42 0, 202.83 0, 492.57 0 M492.57 0 C492.57 169.85, 492.57 339.69, 492.57 442.81 M492.57 0 C492.57 101, 492.57 202.01, 492.57 442.81 M492.57 442.81 C305.91 442.81, 119.26 442.81, 0 442.81 M492.57 442.81 C389.31 442.81, 286.05 442.81, 0 442.81 M0 442.81 C0 289.29, 0 135.78, 0 0 M0 442.81 C0 311.69, 0 180.57, 0 0" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g stroke-opacity="0.4" fill-opacity="0.4" stroke-linecap="round" transform="translate(830.9961718369341 1669.1134274080941) rotate(0 424.69964518504094 63.63277454112722)"><path d="M0 0 C217.45 0, 434.91 0, 849.4 0 M0 0 C181.34 0, 362.68 0, 849.4 0 M849.4 0 C849.4 34.73, 849.4 69.47, 849.4 127.27 M849.4 0 C849.4 41.03, 849.4 82.06, 849.4 127.27 M849.4 127.27 C510.16 127.27, 170.91 127.27, 0 127.27 M849.4 127.27 C613.39 127.27, 377.38 127.27, 0 127.27 M0 127.27 C0 90.16, 0 53.05, 0 0 M0 127.27 C0 85.28, 0 43.3, 0 0" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g stroke-opacity="0.4" fill-opacity="0.4" stroke-linecap="round" transform="translate(591.7225630528646 1163.8701994461244) rotate(0 500.43902966534824 199.6173049090212)"><path d="M0 0 C344.44 0, 688.88 0, 1000.88 0 M0 0 C372.78 0, 745.56 0, 1000.88 0 M1000.88 0 C1000.88 123.02, 1000.88 246.04, 1000.88 399.23 M1000.88 0 C1000.88 141.24, 1000.88 282.48, 1000.88 399.23 M1000.88 399.23 C649.9 399.23, 298.93 399.23, 0 399.23 M1000.88 399.23 C640.12 399.23, 279.36 399.23, 0 399.23 M0 399.23 C0 310.25, 0 221.26, 0 0 M0 399.23 C0 302.24, 0 205.24, 0 0" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(682.5914875548369 10) rotate(0 217.88986206054688 12.5)"><text x="217.88986206054688" y="17.619999999999997" font-family="Excalifont, Xiaolai, sans-serif, Segoe UI Emoji" font-size="20px" fill="#1e1e1e" text-anchor="middle" style="white-space: pre;" direction="ltr" dominant-baseline="alphabetic">USERS/LIQUIDITY PROVIDERS/KEEPERS</text></g><g transform="translate(489.68540009581375 706.2559015338629) rotate(0 75.10994720458984 12.5)"><text x="75.10994720458984" y="17.619999999999997" font-family="Excalifont, Xiaolai, sans-serif, Segoe UI Emoji" font-size="20px" fill="#1e1e1e" text-anchor="middle" style="white-space: pre;" direction="ltr" dominant-baseline="alphabetic">ACCESS LAYER</text></g><g transform="translate(12.427811343128269 1104.0750673437688) rotate(0 111.66993713378906 12.5)"><text x="111.66993713378906" y="17.619999999999997" font-family="Excalifont, Xiaolai, sans-serif, Segoe UI Emoji" font-size="20px" fill="#1e1e1e" text-anchor="middle" style="white-space: pre;" direction="ltr" dominant-baseline="alphabetic">DISCOVERY/HISTORY</text></g><g transform="translate(838.0409422745888 1642.2939798650277) rotate(0 98.07996368408203 12.5)"><text x="98.07996368408203" y="17.619999999999997" font-family="Excalifont, Xiaolai, sans-serif, Segoe UI Emoji" font-size="20px" fill="#1e1e1e" text-anchor="middle" style="white-space: pre;" direction="ltr" dominant-baseline="alphabetic">EXECUTION LAYER</text></g><g transform="translate(590.750372870114 1135.1732758190537) rotate(0 115.06993865966797 12.5)"><text x="115.06993865966797" y="17.619999999999997" font-family="Excalifont, Xiaolai, sans-serif, Segoe UI Emoji" font-size="20px" fill="#1e1e1e" text-anchor="middle" style="white-space: pre;" direction="ltr" dominant-baseline="alphabetic">AMPLIDEX PROTOCOL</text></g><g stroke-linecap="round"><g transform="translate(157.11546399058898 1280.8601768288336) rotate(0 0 -18.285603606659606)"><path d="M0 0 C0 -8.34, 0 -16.67, 0 -36.57 M0 0 C0 -7.49, 0 -14.98, 0 -36.57" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(157.11546399058898 1280.8601768288336) rotate(0 0 -18.285603606659606)"><path d="M0 -36.57 L6.34 -22.98 L-6.34 -22.98 L0 -36.57" stroke="none" stroke-width="0" fill="#1e1e1e" fill-rule="evenodd"></path><path d="M0 -36.57 C1.44 -33.47, 2.89 -30.37, 6.34 -22.98 M0 -36.57 C1.3 -33.79, 2.6 -31, 6.34 -22.98 M6.34 -22.98 C1.45 -22.98, -3.44 -22.98, -6.34 -22.98 M6.34 -22.98 C2.27 -22.98, -1.79 -22.98, -6.34 -22.98 M-6.34 -22.98 C-4.09 -27.79, -1.85 -32.6, 0 -36.57 M-6.34 -22.98 C-4.49 -26.94, -2.64 -30.9, 0 -36.57 M0 -36.57 C0 -36.57, 0 -36.57, 0 -36.57 M0 -36.57 C0 -36.57, 0 -36.57, 0 -36.57" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(116.89062833974094 1280.8601768288336) rotate(0 0 -18.285603606659606)"><path d="M0 0 C0 -7.32, 0 -14.63, 0 -36.57 M0 0 C0 -14.37, 0 -28.74, 0 -36.57" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g><g transform="translate(116.89062833974094 1280.8601768288336) rotate(0 0 -18.285603606659606)"><path d="M0 0 L-6.34 -13.59 L6.34 -13.59 L0 0" stroke="none" stroke-width="0" fill="#1e1e1e" fill-rule="evenodd"></path><path d="M0 0 C-1.27 -2.72, -2.54 -5.44, -6.34 -13.59 M0 0 C-2.49 -5.34, -4.98 -10.68, -6.34 -13.59 M-6.34 -13.59 C-2.71 -13.59, 0.93 -13.59, 6.34 -13.59 M-6.34 -13.59 C-1.46 -13.59, 3.43 -13.59, 6.34 -13.59 M6.34 -13.59 C4.26 -9.13, 2.18 -4.67, 0 0 M6.34 -13.59 C4.21 -9.03, 2.08 -4.47, 0 0 M0 0 C0 0, 0 0, 0 0 M0 0 C0 0, 0 0, 0 0" stroke="#1e1e1e" stroke-width="2" fill="none"></path></g></g><mask></mask></svg>

```mermaid
flowchart TB

subgraph CLIENTS["Users & Integrators"]
    direction LR
    U["Stellar Users"]
    EVMU["EVM Users"]
    LP["Liquidity Providers"]
    LB["Liquidation Buyers"]
    AG["Agents / Integrators"]
end

subgraph ACCESS["Access Layer"]
    direction LR
    UI["Web / Mobile"]
    SDK["AmpliDex SDK"]
    RPC["Stellar RPC"]
end

subgraph PROTOCOL["AmpliDex Protocol - Soroban"]
    direction TB

    CORE["AmpliDex Core"]

    subgraph SERVICES["Protocol Services"]
        direction LR
        POOL["Isolated Lending Pools"]
        EXEC["Execution Router"]
        ORACLE["Oracle Router"]
        LIQ["Liquidation Engine"]
    end

    CORE --> POOL
    CORE --> EXEC
    CORE --> ORACLE
    CORE --> LIQ
end

subgraph EXTERNAL["Execution & Pricing"]
    direction LR
    AQ["Aquarius"]
    SS["Soroswap"]
    O1["Primary Reference Source"]
    O2["Secondary Reference Source"]
end

subgraph DATA["Indexing & Automation"]
    direction LR
    IDX["Event Indexer"]
    DB[("PostgreSQL")]
    API["Query API"]
    KEEP["Independent Keepers"]
end

subgraph STAGED["Staged Cross-Chain Access"]
    direction LR
    EW["EVM Wallet"]
    SA["Soroban Smart Account"]
    CCTP["Circle CCTP V2"]
end

U --> UI
LP --> UI
LB --> UI
AG --> SDK

UI --> SDK
SDK --> RPC
RPC --> CORE

EXEC --> AQ
EXEC --> SS

ORACLE --> O1
ORACLE --> O2

CORE -->|"Events"| IDX

IDX --> DB
DB --> API
DB --> KEEP

API -.->|"Discovery / History"| UI
KEEP -->|"Submit Operations"| RPC

EVMU -.-> EW
EW -.->|"Authorized Control"| SA
SA -.->|"Scoped Invocation"| CORE
CCTP -.->|"Native USDC Transport"| SA
```

The authoritative financial state remains on Stellar. Off-chain systems improve discovery, monitoring, transaction preparation, and automation but cannot redefine balances, debt, position health, or liquidation eligibility.

### 3.1 Trust Boundaries

| **Component**     | **Trusted For**                                     | **Not Trusted For**                                       |
| ----------------- | --------------------------------------------------- | --------------------------------------------------------- |
| Soroban contracts | Balances, debt, solvency, settlement, authorization | External prices without validation                        |
| Governance        | Bounded configuration and scheduled upgrades        | Arbitrary transfer of user or LP funds                    |
| Guardian          | Narrow emergency restrictions                       | Economic changes, fee changes, arbitrary balance mutation |
| DEX adapters      | Invoking registered execution venues                | Defining risk policy or weakening user bounds             |
| Oracle adapters   | Reference observations                              | Sole pricing authority when stale or divergent            |
| Indexer / API     | Discovery, history, analytics                       | Authoritative balances or liquidation eligibility         |
| Keepers           | Candidate discovery and transaction submission      | Deciding whether a position is liquidatable               |
| Frontend / SDK    | Transaction construction, simulation, signing flow  | Contract-level validation                                 |
| CCTP              | Native USDC transport                               | Synchronizing AmpliDex debt or position state             |
| Smart accounts    | Scoped user-authorized Soroban execution            | Protocol governance or custody outside user authorization |

## 4. Architecture Principles

#### Single settlement domain

Liquidity, debt, positions, fees, and liquidations settle on Stellar/Soroban.

#### On-chain authority; off-chain acceleration

Backends, indexers, APIs, keepers, and interfaces improve usability and liveness but cannot rewrite financial truth.

#### Fail closed for added risk

Unsafe oracle, route, liquidity, market-cap, or execution conditions reject new risk.

#### Preserve risk reduction

Repayment, margin addition, position reduction, close, and liquidation remain available whenever they can be performed safely.

#### Least privilege

Governance, guardian, deployer, keeper, CI, infrastructure, and operational identities are separate and narrowly scoped.

#### Deterministic execution

Protocol-managed assets interact only with registered adapters and bounded execution routes.

#### Market isolation

Risk caps, pools, pricing rules, routes, liquidation behavior, and emergency controls are defined per market.

#### Atomicity and idempotency

On-chain financial transitions are atomic. Off-chain event processing is replay-safe and idempotent.

#### Explicit failure domains

A failure in a DEX, oracle, keeper, indexer, RPC provider, bridge, or staged extension must not implicitly corrupt unrelated market accounting.

#### Measurable operations

Safety-relevant signals have explicit metrics, owners, alert thresholds, and runbooks.

## 5. On-Chain Architecture

### 5.1 Contract Topology

```
AmpliDex Core
├── Market registry and configuration
├── Lending and LP accounting
├── Borrow and interest accounting
├── Position state machine
├── Risk evaluation
├── Settlement
└── Fee accounting


Execution Router
├── Aquarius adapter
├── Soroswap adapter
└── Governed route registry


Oracle Router
├── Primary reference adapter
├── Secondary reference adapter
├── Normalization logic
└── Validation / degraded-mode policy


Liquidation Engine
├── Eligibility evaluation
├── Liquidation quote construction
├── Direct buyer settlement
└── Keeper-assisted settlement


Extensions
├── Smart Account Factory
├── EVM-controlled Soroban account
└── CCTP ingress / egress integration
```

Accounting that shares critical invariants remains atomic within the authoritative protocol state transition.

External integrations are isolated behind narrow interfaces.

Whether routers or subsystems are implemented as separately deployed contracts or internal contract modules is a deployment decision. Authorization boundaries, upgrade boundaries, storage ownership, and failure assumptions are documented in either case.

### 5.2 Canonical Records

```
GlobalConfig
protocol version, governance roles, guardian

MarketConfig(asset)
status, caps, margin parameters, routes, oracle policy

PoolState(asset)
cash, total borrows, reserves, total shares, borrow index

LpBalance(asset, account)
LP share ownership

Position(position_id)
owner, market, side, collateral, debt, holdings, status, version

OwnerPosition(owner, ...)
bounded position-discovery index where practical

RouteConfig(route_id)
venue, adapter, asset path, limits, priority, status

OraclePolicy(market)
sources, freshness bounds, deviation threshold, degraded behavior
```

Storage migrations are:

* versioned;
* bounded;
* idempotent;
* migration-order aware; and
* tested against serialized production-shaped state.

Existing enum variants are never reordered.

Every deployed release records:

* source commit;
* release tag;
* compiler and toolchain versions;
* reproducible WASM hash;
* Stellar network;
* contract ID;
* migration version;
* deployment configuration; and
* governance authorization transaction.

### 5.3 Market Configuration

Each market defines:

* enabled operations;
* emergency state;
* base and quote assets;
* asset decimal scales;
* pool supply cap;
* borrow cap;
* utilization limits;
* maximum position size;
* market open-interest cap;
* maximum borrow multiplier;
* warning margin;
* maintenance margin;
* opening fee;
* closing fee;
* reserve factor;
* liquidation fee or incentive;
* approved execution routes;
* maximum route hops;
* route liquidity floors;
* user and protocol slippage limits;
* approved reference-price sources;
* maximum oracle age;
* price-deviation threshold;
* degraded-mode policy;
* minimum position size; and
* minimum residual-position size.

Configuration updates validate internal consistency.

For example:

```
warning_margin > maintenance_margin
maintenance_margin > 0
max_borrow_multiplier <= protocol_maximum
route_slippage <= protocol_maximum_slippage
```

Configuration changes may not retroactively mutate existing user balances, LP shares, or recognized debt.

## 6. Economic Model

### 6.1 Pool and LP Accounting

For an established pool, deposits mint LP shares using the pre-deposit exchange rate:

```
shares_minted =
floor(
  deposit_assets × total_shares
  / total_assets_before
)
```

The economic value represented by LP shares is:

```
assets_owned =
floor(
  user_shares × total_assets
  / total_shares
)
```

Immediately withdrawable cash is bounded by available pool liquidity:

```
available_cash =
token_balance - reserved_outflows
```

```
assets_redeemed <=
min(
  assets_owned,
  available_cash
)
```

The economic specification must explicitly define:

* empty-pool initialization;
* minimum initial liquidity;
* donation behavior;
* share-price manipulation protections;
* deposit rounding direction;
* withdrawal rounding direction;
* dust handling;
* token decimal normalization;
* fee-on-transfer token rejection;
* reserve treatment;
* loss realization; and
* behavior when losses reduce the LP exchange rate.

The interface must distinguish **economic ownership** from **immediately withdrawable liquidity**.

### 6.2 Debt and Interest

Debt is represented through a pool-level borrow index.

Conceptually:

```
scaled_debt =
ceil(
  borrow_amount × INDEX_SCALE
  / borrow_index_at_borrow
)
```

Current debt is:

```
current_debt =
ceil(
  scaled_debt × current_borrow_index
  / INDEX_SCALE
)
```

Borrowing and repayment round conservatively so debt cannot disappear through truncation.

Interest accrual is bounded by:

* elapsed-time limits;
* fixed-point arithmetic limits;
* maximum configured interest rate;
* overflow-safe multiplication/division; and
* defined maximum accrual intervals.

Accrual occurs before every operation that reads or mutates authoritative pool debt.

Utilization is conceptually:

```
U =
total_borrowed / total_assets
```

A kinked annualized borrow-rate model may be represented as:

```
if U <= U_optimal:

    rate =
      base
      + slope_1 × U / U_optimal

else:

    rate =
      base
      + slope_1
      + slope_2 × (U - U_optimal) / (1 - U_optimal)
```

For intuition only, supply rate may be approximated as:

```
supply_rate ≈
borrow_rate × U × (1 - reserve_factor)
```

The exact implementation specifies:

* time basis;
* compounding convention;
* fixed-point scale;
* interest-accrual granularity;
* maximum rate;
* reserve accounting;
* rounding direction; and
* reference test vectors.

These are normative economic parameters, not implicit implementation details.

### 6.3 Leverage Semantics

AmpliDex uses **borrow multiplier**, not gross-exposure multiplier.

```
borrowed_notional =
collateral × borrow_multiplier
```

Collateral remains committed as loss-absorbing margin and is not included\
in the initial traded notional.

For a fully deployed position:

position\_exposure = borrowed\_notional

Example:

```
Collateral / margin:  1,000 USDC
Borrow multiplier:    5.0×
Borrowed notional:    5,000 USDC
Position exposure:    5,000 USDC
```

The contract, SDK, interface, documentation, and risk disclosures use this terminology consistently.

### 6.4 Position State Machine

```mermaid
stateDiagram-v2

[*] --> Open: Validated open

Open --> Open: Increase
Open --> Open: Add margin
Open --> Open: Repay
Open --> Open: Valid partial close

Open --> Warning: Warning threshold crossed
Warning --> Open: Position recovers
Warning --> Warning: Add margin
Warning --> Warning: Repay
Warning --> Warning: Valid partial close

Open --> Liquidatable: Maintenance margin breached
Warning --> Liquidatable: Maintenance margin breached

Open --> Closed: Full close
Warning --> Closed: Full close

Liquidatable --> Liquidated: Atomic liquidation settlement

Closed --> [*]
Liquidated --> [*]
```

A long position borrows USDC and acquires the market asset.

A short position borrows the market asset and sells it into USDC or the configured quote asset.

Every open or increase operation validates, within the authoritative transaction:

* market status;
* user authorization;
* collateral;
* pool capacity;
* borrow cap;
* position cap;
* open-interest cap;
* price environment;
* execution-route bounds;
* route liquidity;
* realized balance changes;
* resulting holdings;
* fees; and
* resulting margin.

A position may not be opened or increased into an immediately liquidatable state.

Partial close allocates:

* collateral;
* holdings;
* debt;
* realized PnL;
* protocol fees; and
* residual state

using explicitly tested rounding rules.

A partial close rejects a residual position that is:

* dust-sized;
* structurally invalid;
* outside current caps;
* under minimum collateral requirements; or
* immediately liquidatable.

A full close repays current debt and required fees, returns eligible surplus to the owner, and permanently transitions the position to `Closed`.

#### 6.5 Position Risk Calculation

Risk evaluation considers both independent reference pricing and realizable execution value.

Collateral remains committed as loss-absorbing margin and is accounted for separately from the assets or proceeds representing the leveraged position.

For a long position:

```
long_equity =
    collateral
    + conservative_sale_value(held_asset)
    - current_debt
    - estimated_close_costs
```

For a short position:

```
short_equity =
    collateral
    + held_sale_proceeds
    - conservative_repurchase_cost(current_debt_asset)
    - estimated_close_costs
```

At position opening, initial position exposure is the borrowed notional deployed into the trade:

```
initial_position_exposure =
    borrowed_notional
```

Current position exposure is determined from the conservative current value of the leveraged position.

For a long:

```
long_current_exposure =
    conservative_sale_value(held_asset)
```

For a short:

```
short_current_exposure =
    conservative_repurchase_value(current_debt_asset)
```

Margin is conceptually:

```
margin_ratio =
    equity / current_position_exposure
```

The protocol uses the more conservative valid valuation where multiple relevant reference or executable values are available.

For example, a position opened with 1,000 USDC of collateral and a 5.0× borrow multiplier has 5,000 USDC of initial position exposure. The 1,000 USDC collateral remains as margin and is not included in the initial traded notional.

If a long position subsequently falls from 5,000 USDC to a conservative realizable value of 4,500 USDC, ignoring interest and closing costs:

```
equity =
    1,000
    + 4,500
    - 5,000
    = 500 USDC

current_position_exposure =
    4,500 USDC

margin_ratio =
    500 / 4,500
    ≈ 11.11%
```

Interest, execution costs, and applicable fees further reduce equity and are included when determining the actual margin ratio.

The normative economic specification defines:

* initial position exposure;
* current position exposure;
* positive and negative equity;
* treatment of collateral;
* treatment of short-sale proceeds;
* valuation units;
* margin denominator;
* warning and maintenance-margin thresholds;
* liquidation equality conditions;
* accrued-interest treatment;
* fee ordering;
* liquidation incentive priority;
* decimal normalization;
* rounding direction;
* residual collateral;
* negative-value handling; and
* behavior when no executable quote exists.

The **Soroban contract**, not the indexer, keeper, query API, or frontend, makes the final solvency and liquidation determination.

## 7. Execution and Pricing

### 7.1 Execution Router

The AmpliDex Core invokes only execution routes registered through governed configuration.

A route binds:

* market;
* direction;
* adapter;
* venue;
* input asset;
* output asset;
* path;
* maximum hops;
* quote expiration;
* liquidity floor;
* protocol slippage ceiling;
* optional route priority; and
* enabled status.

Adapters expose narrow operations such as:

```
quote_exact_input(...)
execute_exact_input(...)

quote_exact_output(...)
execute_exact_output(...)
```

A transaction succeeds only when the selected route:

1. is enabled;
2. matches the expected market and asset direction;
3. uses an approved adapter;
4. stays inside the user-signed maximum input or minimum output;
5. stays inside protocol-level slippage limits;
6. stays inside configured price-deviation limits;
7. satisfies freshness requirements;
8. meets configured liquidity requirements; and
9. produces an actual balance delta consistent with the executed operation.

User-signed bounds and protocol bounds are independent.

A retry may choose another approved route but may never loosen either set of constraints.

Fallback execution occurs across separately simulated and authorized attempts unless the complete fallback path can be proven atomic and deterministic within a single transaction.

***

### 7.2 Oracle Router

AmpliDex distinguishes between two pricing concepts.

#### Reference price

An independent, freshness-checked observation used for:

* solvency;
* margin evaluation;
* circuit breakers;
* price-deviation checks; and
* risk controls.

#### Executable price

A conservative approved-route quote used to estimate:

* realizable sale value;
* asset repurchase cost;
* liquidation proceeds;
* closing cost; and
* execution viability.

All sources are normalized to a documented fixed-point scale.

They are checked for:

* ledger or timestamp freshness;
* decimal normalization;
* source validity;
* cross-source deviation;
* route/reference deviation; and
* configured market policy.

Material disagreement is not averaged away.

#### Degraded Pricing Behavior

| **Pricing Condition**             | **New Exposure**              | **Repayment / Add Margin** | **Close / Risk Reduction**                   | **Liquidation**                               |
| --------------------------------- | ----------------------------- | -------------------------- | -------------------------------------------- | --------------------------------------------- |
| Healthy sources                   | Enabled                       | Enabled                    | Enabled                                      | Enabled                                       |
| One source stale                  | Market-policy dependent       | Enabled                    | Conservative route bounds                    | Conservative policy                           |
| Material source disagreement      | Disabled                      | Enabled                    | Risk-reducing path only                      | Restricted / predefined policy                |
| All reference sources unavailable | Disabled                      | Enabled                    | Emergency-exit policy                        | Predefined emergency policy                   |
| Route unavailable                 | Disabled where route required | Enabled                    | Alternate approved route or emergency policy | Alternate approved route or predefined policy |

Exact behavior is defined per market before activation.

An oracle incident never silently substitute an unbounded spot price.

## 8. Liquidation and Bad Debt

### 8.1 Liquidation Eligibility

Liquidation eligibility is recalculated atomically in the settlement transaction.

A keeper or liquidation buyer may identify and submit a candidate position, but cannot force a healthy position to liquidate.

No off-chain service has authority to override on-chain eligibility.

### 8.2 Liquidation Quote

A liquidation quote binds, where applicable:

* position ID;
* position version;
* market;
* required buyer payment;
* debt principal;
* accrued interest;
* protocol fees;
* liquidation incentive;
* collateral delivered;
* reference price;
* executable price;
* quote ledger;
* expiry; and
* relevant route constraints.

A buyer signs bounded terms such as:

```
max_payment
min_collateral_received
expiry
position_version
```

Settlement rechecks:

* authorization;
* current position version;
* current position health;
* quote validity;
* debt;
* fees;
* route bounds;
* payment;
* collateral delivery; and
* market state.

A successful liquidation:

1. collects settlement value;
2. repays the affected lending pool;
3. distributes only configured and permitted fees;
4. returns any eligible residual surplus to the position owner; and
5. marks the position as `Liquidated`.

### 8.3 Keeper Model

Keepers provide automation and liveness but are not protocol authorities.

Multiple independent keepers may:

* discover risky positions;
* simulate liquidation;
* submit liquidation transactions;
* rebroadcast after RPC failure; and
* monitor execution.

Keeper keys:

* do not control governance;
* do not control user accounts;
* do not decide liquidation eligibility;
* hold only bounded operating balances; and
* can be replaced without changing protocol solvency rules.

Permissionless third parties may submit valid liquidation transactions independently of AmpliDex-operated keepers.

### 8.4 Bad Debt

Bad debt is recognized only after position-held assets and collateral have been exhausted.

The intended loss waterfall is:

```
Position-held assets
        ↓
Position collateral
        ↓
Designated insurance reserve, if enabled
        ↓
Affected isolated pool loss
```

Losses are explicit in contract state and emitted events.

They are never silently assigned to unrelated markets or pools.

If insurance is enabled, it has explicit:

* balances;
* contribution rules;
* per-market limits;
* global limits;
* epoch payout limits; and
* governance controls.

## 9. Off-Chain Platform

### 9.1 Event Indexer

Soroban contract events feed a replayable indexer with a durable ledger checkpoint.

Event rows use deterministic identities such as:

```
(network, transaction_hash, event_index)
```

Event persistence and checkpoint advancement occur in one database transaction.

The indexer supports derived views for:

* account positions;
* pool activity;
* market history;
* utilization;
* liquidations;
* fees;
* execution activity;
* protocol metrics; and
* staged cross-chain status.

The indexer:

* resumes from the last durable checkpoint;
* tolerates duplicate event delivery;
* handles supported Stellar history semantics;
* detects gaps;
* replays deterministically;
* reconciles materialized state against contract reads; and
* exposes lag and reconciliation metrics.

Indexed state is never authoritative for financial transitions.

### 9.2 Query API

The query API is:

* versioned;
* read-oriented;
* paginated;
* rate-limited;
* cache-aware; and
* non-authoritative.

A canonical public path may use:

```
/v1/...
```

The API supports:

* markets;
* pools;
* positions;
* account history;
* liquidations;
* protocol metrics;
* route status;
* analytics; and
* cross-chain status where applicable.

Quote endpoints are advisory.

Any transaction derived from API data is simulated and revalidated against authoritative on-chain state before signing or submission.

### 9.3 Frontend and SDK

The non-custodial client is responsible for:

* wallet connection;
* market discovery;
* authoritative state refresh;
* transaction construction;
* simulation;
* signing;
* submission;
* lifecycle feedback; and
* failure recovery guidance.

A backend never signs financial transactions on behalf of the user.

Data precedence is:

1. **contract state** for balances, debt, configuration, position status, and eligibility;
2. **current simulation** for transaction-specific effects and resource costs; and
3. **indexed data** for discovery, history, and analytics.

SDK modules may include:

```
markets
pools
positions
liquidation
pricing
execution
simulation
contract bindings
smart accounts
CCTP
```

Generated bindings are pinned to compatible contract versions.

Clients fail explicitly when connected to an unsupported protocol version.

## 10. Cross-Chain Access

### 10.1 EVM-Controlled Soroban Smart Accounts

An EVM-controlled Soroban smart account may hold Stellar assets and authorize scoped Soroban invocations after validating an EVM signature.

An authorization binds at minimum:

* EVM signer;
* Stellar network passphrase;
* smart-account address;
* target contract;
* invocation tree or invocation hash;
* nonce;
* validity window; and
* account implementation version.

Nonces are single-use.

Signer rotation and recovery are explicit privileged flows with separate authorization requirements.

The EVM wallet controls a **Stellar-resident account**. It does not cause AmpliDex position state to exist on an EVM chain.

### 10.2 Circle CCTP

Circle CCTP transports native USDC to and from the user-controlled Stellar account.

CCTP does **not** transport:

* AmpliDex position state;
* debt;
* LP shares;
* liquidation state; or
* protocol governance state.

The bridge and trade are separate state machines.

Ingress:

```
EVM user authorizes CCTP transfer
        ↓
Source-chain approval / burn
        ↓
Message finality
        ↓
Circle attestation
        ↓
Destination mint on Stellar
        ↓
USDC held by user-controlled Soroban account
        ↓
User separately authorizes AmpliDex deposit
        ↓
AmpliDex deposit transaction is simulated
        ↓
Authorized AmpliDex invocation
        ↓
USDC deposited into AmpliDex
```

Egress:

```
USDC held in AmpliDex
        ↓
User authorizes AmpliDex withdrawal
        ↓
USDC returned to user-controlled Stellar account
        ↓
User separately authorizes CCTP transfer
        ↓
CCTP burn on Stellar
        ↓
Message finality / attestation
        ↓
Destination-chain mint
        ↓
USDC received by user-controlled destination account
```

A successful CCTP transfer does not authorize an AmpliDex operation.

After USDC is minted on Stellar, the assets remain under the control of the user's Soroban account until the user separately authorizes an AmpliDex deposit or other protocol operation.

If the CCTP transfer succeeds but the subsequent AmpliDex operation is not authorized, cannot be simulated successfully, or fails during execution, the USDC remains under user control.

The user may:

* retry the AmpliDex operation;
* hold the USDC in their Soroban account;
* transfer the USDC to another Stellar account;
* use the USDC with another Stellar application; or
* separately authorize a CCTP transfer to bridge the USDC to a supported destination chain.

Relayers may pay transaction fees or submit authorized transactions, but they never receive spending authority over the user's assets.

Each supported CCTP integration release documents:

* source domain;
* destination domain;
* supported chains;
* finality assumptions;
* message uniqueness and replay protection;
* duplicate-message behavior;
* attestation behavior;
* retry and idempotency handling;
* partial-completion and recovery procedures; and
* operational monitoring.

## 11. Governance and Upgrade Safety

Production control consists of:

* governance multisig or governor;
* timelock;
* narrowly scoped emergency guardian; and
* minimized deployment authority.

### 11.1 Role Boundaries

| **Role**               | **May Perform**                                                                                             | **May Not Perform**                                                       |
| ---------------------- | ----------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| Governance             | Schedule upgrades, add markets, update timelocked parameters, register approved adapters and oracle sources | Bypass contract invariants or arbitrarily seize user balances             |
| Guardian               | Pause new risk, disable markets, routes, or oracle sources under predefined emergency rules                 | Move user funds, change economic ownership, arbitrarily upgrade contracts |
| Deployer               | Execute an approved deployment or migration procedure                                                       | Retain unrestricted economic authority after handoff                      |
| Keeper                 | Discover and submit public protocol operations                                                              | Change configuration, upgrade contracts, or override eligibility          |
| Indexer / API operator | Operate data infrastructure                                                                                 | Authorize financial state transitions                                     |

### 11.2 Emergency States

Granular emergency modes include:

```
NORMAL

NEW_EXPOSURE_PAUSED

MARKET_RESTRICTED

ROUTE_RESTRICTED

ORACLE_RESTRICTED

LIQUIDATION_ONLY

EMERGENCY_EXIT
```

Emergency-state transitions:

* emit on-chain events;
* identify the initiating role;
* carry bounded authority;
* have documented entry criteria;
* have documented exit criteria; and
* map to operational runbooks.

### 11.3 Upgrade Process

Production upgrades follow:

```
Proposal
    ↓
Public technical review
    ↓
Security-delta review
    ↓
Governance approval
    ↓
Timelock
    ↓
Execution
    ↓
Migration verification
    ↓
Post-deployment verification
```

Storage migration and rollback or forward-fix procedures are rehearsed against production-shaped state before deployment.

Critical governance and upgrade roles use threshold or multisig custody with hardware-backed keys where applicable.

Key rotation procedures are documented and tested.

## 12. Security Model

### 12.1 Critical Invariants

Critical invariants include:

1. LP shares cannot create value through deposit or withdrawal rounding.
2. Withdrawals cannot exceed the user's economic ownership.
3. Withdrawals cannot exceed available pool cash.
4. Accrued debt cannot decrease except through repayment, settlement, or explicit recognized loss.
5. Borrowing cannot exceed pool, asset, market, or position limits.
6. A newly opened or increased position cannot be immediately liquidatable.
7. Partial close cannot leave invalid or unaccounted state.
8. Only registered adapters and bounded routes may move protocol-managed trade assets.
9. Stale or conflicting pricing conditions cannot create new leveraged exposure.
10. Liquidation eligibility is recalculated in the settlement transaction.
11. Fees cannot consume unrelated LP principal.
12. Fees cannot exceed configured or economically valid limits.
13. Bad debt is explicit and isolated to its configured loss domain.
14. Indexed or API state cannot authorize financial transitions.
15. User bounds cannot be loosened by route fallback.
16. An EVM authorization cannot replay across nonce, account, contract, version, or Stellar network.
17. A failed AmpliDex operation after successful CCTP mint leaves the minted funds under user control.
18. Privileged actors cannot arbitrarily transfer user or LP balances.
19. Emergency modes may reduce risk but may not silently rewrite economic ownership.

### 12.2 Threats and Controls

| **Threat**                        | **Primary Controls**                                                                 | **Failure Response**                                      |
| --------------------------------- | ------------------------------------------------------------------------------------ | --------------------------------------------------------- |
| DEX manipulation / low liquidity  | Independent price checks, route allowlist, caps, slippage bounds, liquidity floors   | Restrict route or market; use conservative close path     |
| Oracle compromise / staleness     | Multiple sources, freshness checks, deviation bounds, circuit breakers               | Stop new risk; enter predefined degraded mode             |
| Insolvency from volatility        | Conservative margins, caps, liquidation incentive, keeper diversity                  | Explicit bad-debt recognition and isolated loss           |
| Flash-loan / composability attack | End-state validation, conservative pricing, bounded routes, safe invocation ordering | Reject invalid transaction and investigate economic path  |
| Malicious token / adapter         | Asset review, adapter allowlist, balance-delta verification                          | Disable affected integration and isolate market           |
| Keeper outage / censorship        | Permissionless submission, multiple operators, direct buyers                         | Alert and activate independent fallback operators         |
| Governance compromise             | Multisig, timelock, narrow guardian, published hashes                                | Pause new exposure and rotate compromised authority       |
| Smart-account replay              | Full domain separation, nonce, expiry, invocation scope                              | Disable staged access path without affecting native users |
| Indexer / RPC corruption          | Reconciliation, redundant providers, on-chain revalidation                           | Mark data stale and fail transaction preparation safely   |
| CCTP partial completion           | Explicit state machine, idempotency, user-controlled destination                     | Recovery workflow and independent status verification     |

All financial arithmetic uses:

* checked integers;
* fixed-point representations;
* normalized decimals;
* overflow-safe `mul_div` patterns;
* explicit rounding rules; and
* no floating-point arithmetic.

External token callbacks and re-entrant invocation paths are treated as hostile.

Authorization and state transitions follow safe checks/effects/interactions ordering where applicable.

## 13. Verification and Assurance

### 13.1 Test Strategy

#### Unit tests

Cover:

* interest;
* borrow indexes;
* LP shares;
* fees;
* decimal normalization;
* margin;
* liquidation thresholds;
* slippage;
* configuration validation;
* authorization; and
* error conditions.

#### Property tests

Verify properties such as:

* asset conservation;
* debt monotonicity absent repayment;
* no rounding extraction;
* bounds never loosen;
* invalid routes cannot execute;
* healthy positions cannot liquidate; and
* user authorization cannot replay.

#### Stateful fuzz tests

Exercise long operation sequences including:

* deposit;
* withdrawal;
* borrow;
* repay;
* open;
* increase;
* add margin;
* repeated partial close;
* liquidation;
* loss events;
* extreme time jumps;
* dust;
* mixed decimals; and
* cap transitions.

#### Integration tests

Cover:

* Aquarius adapter;
* Soroswap adapter;
* oracle adapters;
* route fallback;
* pricing disagreement;
* RPC failure;
* indexer replay;
* reconciliation;
* contract migration; and
* staged cross-chain components where enabled.

#### Adversarial tests

Cover:

* manipulated DEX prices;
* stale reference sources;
* divergent oracle sources;
* front-running;
* keeper races;
* liquidation-buyer races;
* replay attempts;
* malicious token behavior;
* resource exhaustion; and
* invalid cross-domain authorization.

#### End-to-end tests

Validate the full lifecycle on testnet:

```
LP deposit
    ↓
Open position
    ↓
Interest accrual
    ↓
Margin change / repay
    ↓
Partial close
    ↓
Full close
```

and:

```
LP deposit
    ↓
Open position
    ↓
Risk deterioration
    ↓
Liquidation eligibility
    ↓
Keeper / buyer submission
    ↓
Atomic liquidation
    ↓
Pool repayment
```

Operational drills are included where relevant.

### 13.2 CI Requirements

CI blocks merge on applicable failures in:

* formatting;
* linting;
* contract build;
* unit tests;
* property tests;
* integration tests;
* known-vulnerability policy;
* generated-binding drift;
* reproducible artifact generation; and
* dependency-lock consistency.

Coverage is treated as a supporting signal, not a substitute for invariant, property, state-machine, and adversarial testing.

### 13.3 Independent Review

Independent security review is scoped by subsystem.

Required review domains include:

* core economics;
* contract accounting;
* position-state transitions;
* liquidation;
* DEX integrations;
* oracle integrations;
* governance;
* upgrade and emergency authorization;
* EVM smart-account authentication before activation; and
* CCTP recovery and integration behavior before cross-chain launch.

Critical and high-severity findings are resolved and retested before unrestricted activation.

Published security evidence should identify:

* reviewed commit;
* review scope;
* findings;
* remediation commits;
* retest status; and
* accepted residual risk where applicable.

## 14. Reliability, Observability, and Operations

### 14.1 Service Objectives

Targets are validated under production-shaped load and failure testing.

| **Service / Signal**                     | **Initial Target**                         |
| ---------------------------------------- | ------------------------------------------ |
| Public web and read API availability     | 99.9% monthly                              |
| Indexer lag under normal conditions      | Fewer than 3 ledgers                       |
| Indexer recovery point                   | Last durable finalized checkpoint          |
| Keeper candidate-to-submission latency   | Market-specific safety budget              |
| Critical on-call alert dispatch          | Under 2 minutes                            |
| Database recovery                        | Point-in-time recovery with tested restore |
| Backend dependency for protocol solvency | None                                       |

These values are operational targets and may evolve based on production measurements.

### 14.2 Metrics

Operational metrics include:

#### Pools

* cash;
* borrows;
* utilization;
* reserves;
* share price;
* deposit flow;
* withdrawal flow.

#### Positions

* open interest;
* side distribution;
* margin distribution;
* warning positions;
* liquidatable positions;
* close volume;
* bad debt.

#### Execution

* route usage;
* quote failures;
* execution failures;
* slippage;
* route fallback;
* liquidity-floor violations.

#### Pricing

* reference-source age;
* source availability;
* cross-source deviation;
* reference/executable deviation;
* degraded-mode transitions.

#### Liquidation

* candidate count;
* submission latency;
* success rate;
* buyer participation;
* keeper activity;
* race frequency;
* unpaid debt.

#### Infrastructure

* RPC latency and failure;
* indexer lag;
* indexer reconciliation;
* database health;
* API latency;
* keeper heartbeat;
* keeper operating balance.

#### Cross-chain

Where staged CCTP access is enabled:

* burn state;
* attestation duration;
* mint duration;
* retry count;
* unresolved transfers; and
* recovery events.

### 14.3 Alerts and Runbooks

Alerts define:

* severity;
* threshold;
* owner;
* escalation;
* suppression policy; and
* linked runbook.

Runbooks cover at minimum:

* oracle divergence;
* stale pricing;
* DEX or adapter failure;
* abnormal utilization;
* liquidation backlog;
* bad debt;
* RPC outage;
* indexer replay;
* indexer rebuild;
* reconciliation failure;
* database recovery;
* compromised keys;
* governance incident;
* contract-upgrade failure;
* CCTP delay;
* smart-account incident; and
* frontend compromise.

High-severity scenarios are rehearsed before increasing protocol risk caps.

### 14.4 Operational Security

Secrets are stored in managed secret systems.

Governance and upgrade keys use hardware-backed multisig or threshold custody where applicable.

Deployer authority is removed or minimized after handoff.

Keeper credentials, CI credentials, RPC secrets, and database credentials are:

* scoped;
* rotated;
* monitored; and
* independently revocable.

Backups are encrypted.

Restore procedures are tested rather than merely documented.

## 15. Deployment and Release

The production release process follows:

```
Reviewed change
        ↓
Deterministic build
        ↓
Artifact hash
        ↓
Unit / property / fuzz / integration gates
        ↓
Testnet deployment
        ↓
End-to-end validation
        ↓
Security-delta approval
        ↓
Governance proposal
        ↓
Timelock
        ↓
Capped mainnet canary
        ↓
Post-deployment verification
        ↓
Observation window
        ↓
Evidence-based cap increase
```

The initial mainnet configuration uses:

* few markets;
* conservative borrow multipliers;
* low supply caps;
* low borrow caps;
* low open-interest caps;
* conservative utilization;
* strict oracle-deviation limits;
* limited execution routes; and
* at least two independently operated keeper paths where keeper automation is required.

Caps increase only after predefined observation periods and measurable criteria covering:

* utilization;
* route execution;
* oracle behavior;
* liquidation latency;
* bad debt;
* infrastructure reliability; and
* incident history.

Release artifacts include:

* source tag;
* immutable source commit;
* dependency lockfiles;
* SBOM where applicable;
* compiler version;
* Stellar toolchain version;
* WASM hash;
* deployed contract IDs;
* configuration manifest;
* migration manifest;
* automated test summary;
* audit references;
* signer approval; and
* rollback or forward-fix plan.

Canonical release evidence is maintained in **Production Readiness & Evidence**.

## 16. Architecture Decisions

### ADR-001 — Stellar Is the Sole Financial Settlement Domain

**Decision**

Liquidity, debt, LP shares, positions, liquidation, and protocol settlement remain authoritative on Stellar/Soroban.

**Rationale**

Avoids fragmented solvency and cross-chain position-state synchronization.

### ADR-002 — CCTP Transports USDC, Not Protocol State

**Decision**

CCTP is used for native USDC movement only.

**Rationale**

Contains cross-chain failure and preserves user recovery without introducing cross-chain debt synchronization

### ADR-003 — The Indexer Is Non-Authoritative

**Decision**

Indexed state is used for discovery, history, monitoring, and analytics.

**Rationale**

Protocol solvency remains independent of backend availability and database correctness.

### ADR-004 — Execution Routes Are Governed and Bounded

**Decision**

Protocol assets execute only through registered adapters and approved routes.

**Rationale**

Prevents arbitrary external invocation and unbounded routing risk.

### ADR-005 — Material Oracle Disagreement Restricts Risk

**Decision**

Meaningfully conflicting observations trigger restricted operation instead of automatic averaging.

**Rationale**

Avoids manufacturing false confidence from inconsistent pricing inputs.

### ADR-006 — Direct Liquidation With Keeper Automation

**Decision**

Liquidation is permissionless, with keepers providing automation rather than exclusive authority.

**Rationale**

Broadens participation and improves liveness without creating a trusted liquidation operator.

### ADR-007 — EVM Users Control Soroban Smart Accounts

**Decision**

External wallet identity is translated into scoped Soroban account authorization.

**Rationale**

Provides EVM access without introducing backend custody or moving AmpliDex state to EVM chains.

### ADR-008 — Markets Are Isolated by Configuration and Loss Domain

**Decision**

Pools, caps, routes, pricing policies, emergency states, and losses are bounded per market wherever practical.

**Rationale**

Limits contagion and makes protocol risk easier to measure and control.

### ADR-009 — Off-Chain Services Accelerate but Do Not Authorize

**Decision**

Keepers, indexers, APIs, relayers, and interfaces may prepare and submit actions but cannot weaken on-chain validation.

**Rationale**

Ensures availability failures or infrastructure compromise do not redefine protocol ownership or solvency.

## Appendix A — Glossary

#### Borrow Multiplier

Borrowed notional divided by posted collateral.

```
borrow_multiplier =
borrowed_notional / collateral
```

#### Gross Exposure

Collateral-funded exposure plus borrowed exposure before fees and execution effects.

#### Reference Price

Independent price observation used for validation, margin evaluation, and risk controls.

#### Executable Price

Conservative value obtainable through an approved execution route under configured bounds.

#### Scaled Debt

Debt units normalized against the pool borrow index.

#### Isolated Pool

A lending pool whose liquidity, debt, caps, and loss accounting are bounded to a defined market or asset domain.

#### Liquidation Buyer

A permissionless participant that settles a qualifying position under bounded contract-enforced terms.

#### Keeper

Untrusted automation that discovers candidates and submits valid protocol operations.

#### Guardian

A restricted emergency role that may reduce risk but cannot arbitrarily seize assets or redefine economic ownership.

#### Execution Adapter

A narrow protocol integration responsible for interacting with a registered external execution venue.

#### Oracle Router

The subsystem responsible for validating and normalizing approved reference-price sources and applying market pricing policy.

#### Degraded Mode

A predefined restricted protocol state activated when a dependency such as an oracle or route no longer satisfies normal operating assumptions.

#### CCTP

Circle Cross-Chain Transfer Protocol, used by AmpliDex staged cross-chain access for native USDC movement.

#### Smart Account

A Soroban account contract that authorizes transactions according to defined authentication and authorization rules.

## Conclusion

AmpliDex is designed as a Stellar-native leveraged credit and execution protocol.

Isolated liquidity pools fund borrowing. Borrowed capital creates bounded long or short exposure through approved Stellar execution venues. Independent pricing and executable-route validation constrain risk. Deterministic margin rules define solvency. Permissionless liquidation returns debt to the affected pool. Bad debt is explicit and isolated. Off-chain infrastructure improves discovery, automation, and observability without becoming authoritative for financial state.

Cross-chain access is deliberately limited to user-controlled account authorization and asset transport. AmpliDex debt and position state remain authoritative on Stellar.

Production readiness is demonstrated through reproducible artifacts, invariant and adversarial testing, independent review, conservative deployment, measurable operations, and published evidence.
