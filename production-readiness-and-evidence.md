---
description: >-
  Due-diligence gates and evidence register for grants, investment review, and
  mainnet launch.
---

# Production Readiness & Evidence

AmpliDex distinguishes between **architectural design, implemented functionality, validated functionality, and production-ready functionality**.

A component is not considered production-ready solely because it appears in the technical architecture or has been implemented. Production readiness requires appropriate testing, deployment evidence, security review, operational ownership, monitoring, and documented verification.

This page defines the readiness criteria applied to AmpliDex and provides a public evidence register linking technical claims to verifiable implementation artifacts.

### Readiness classification

AmpliDex documentation uses the following classifications where implementation status needs to be distinguished:

| **Status**           | **Meaning**                                                                                                                       |
| -------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| **Designed**         | Architecture and expected behavior are specified, but implementation may not yet be complete.                                     |
| **Implemented**      | Functionality exists in source code and is undergoing integration or validation.                                                  |
| **Validated**        | Implementation has passed defined functional and technical validation requirements.                                               |
| **Production-ready** | Required testing, security review, deployment controls, monitoring, operational procedures, and supporting evidence are complete. |
| **Research**         | Experimental capability under evaluation and not part of the production security boundary.                                        |

These classifications allow the technical architecture to describe the intended production system without implying that every component has already completed the same stage of development.

***

### Production launch gates

AmpliDex will not classify the protocol as production-ready until the applicable launch gates have been satisfied.

#### Economic and protocol safety

* Formal economic specification and parameter rationale reviewed.
* Critical protocol and accounting invariants defined and tested.
* Supply, borrow, collateral, interest, repayment, and liquidation accounting validated.
* Mixed-decimal assets, rounding boundaries, dust, time jumps, bad debt, and loss scenarios tested.
* Protocol limits and initial market parameters defined conservatively.

#### Smart contract verification

* Unit and integration test suites passing.
* Property-based and stateful fuzz testing performed against critical invariants.
* Storage migrations tested against production-shaped state.
* Contract upgrade and rollback procedures verified where applicable.
* Reproducible Soroban WASM artifacts generated and independently identifiable.
* Deployed contract configuration verified against the approved deployment manifest.

#### Pricing and execution

* Approved oracle/reference pricing sources independently verified.
* Aquarius, Soroswap, and other enabled execution adapters tested against expected production behavior.
* Route limits, slippage controls, and execution bounds validated.
* Stale, unavailable, manipulated, or divergent price-source scenarios tested.
* Degraded execution and oracle behavior tested.
* Emergency exits and restricted operating modes verified.

#### Liquidation and automation

* Position-health and liquidation calculations tested across boundary conditions.
* Permissionless liquidation behavior verified.
* Keeper automation tested independently from the underlying liquidation mechanism.
* Multiple-keeper and liquidation-race scenarios tested.
* Keeper failure does not prevent eligible third parties from performing permissionless protocol actions.
* Recovery procedures for automation outages documented and tested.

#### Data and reconciliation

* Protocol events can be deterministically replayed from chain history.
* Indexed state is reconciled against authoritative on-chain state.
* Indexer interruption and recovery tested.
* Database backup and restore procedures tested.
* Operational metrics and reconciliation alerts implemented.

#### Governance and operational security

* Administrative permissions minimized and documented.
* Multisig configuration and signer responsibilities verified.
* Timelocks applied to applicable privileged operations.
* Guardian and emergency permissions explicitly bounded.
* Key rotation and recovery procedures tested.
* Deployment and configuration changes auditable.

#### Security and operations

* Independent security review completed where required.
* Critical and high-severity findings resolved before unrestricted deployment.
* Accepted residual findings documented with rationale.
* Production monitoring and alerting operational.
* Incident-response procedures documented.
* Public vulnerability disclosure process available.
* Funded bug bounty operational before unrestricted mainnet deployment.

#### Mainnet deployment

* Source commit and release tag published.
* Contract IDs published.
* Reproducible WASM hashes published.
* Deployment and configuration manifest published.
* Initial protocol caps and risk parameters approved.
* Mainnet deployment begins under conservative operating limits.
* Criteria for increasing caps or enabling additional markets defined in advance.

***

### Staged capabilities

Not every AmpliDex capability shares the same production-readiness boundary.

Core lending, leveraged-position management, execution, pricing, repayment, and liquidation must satisfy the primary protocol launch gates before mainnet operation.

Additional capabilities may be activated independently after satisfying their own security and operational requirements.

In particular:

**Cross-chain access** requires independent validation of bridge dependencies, message and asset settlement behavior, failure recovery, supported networks, and cross-chain account authorization.

**Confidential or ZK-enabled positions** require independent verification of proof construction, verifier correctness, circuit assumptions, failure behavior, and the interaction between private state and protocol solvency requirements.

A staged capability is not considered part of the production security boundary until its applicable approval gates have been completed.

***

### Evidence register

The evidence register provides a verifiable link between AmpliDex's technical claims and deployed implementation.

The register will be updated as components progress through implementation, validation, security review, and deployment.

| **Evidence**                      | **Requirement**                       | **Status**   | **Link / Identifier** |
| --------------------------------- | ------------------------------------- | ------------ | --------------------- |
| Source repository                 | Public implementation source          | Pending      | —                     |
| Release tag / immutable commit    | Required for releases                 | Pending      | —                     |
| Testnet contract IDs              | Required before production deployment | Pending      | —                     |
| Mainnet contract IDs              | Required when deployed                | Not deployed | —                     |
| Reproducible WASM hashes          | Required for deployed contracts       | Pending      | —                     |
| Automated test report             | Required                              | Pending      | —                     |
| Property / fuzz test results      | Required for critical invariants      | Pending      | —                     |
| Economic specification            | Required                              | Pending      | —                     |
| Deployment manifest               | Required                              | Pending      | —                     |
| Production configuration          | Required before mainnet               | Pending      | —                     |
| Security review / audit           | Required before unrestricted mainnet  | Pending      | —                     |
| Audit remediation commit          | Required where findings exist         | Pending      | —                     |
| Monitoring / status system        | Required before unrestricted mainnet  | Pending      | —                     |
| Vulnerability disclosure policy   | Required before unrestricted mainnet  | Pending      | —                     |
| Bug bounty                        | Required before unrestricted mainnet  | Pending      | —                     |
| Governance configuration          | Required before mainnet               | Pending      | —                     |
| Multisig / timelock configuration | Required before mainnet               | Pending      | —                     |

#### Evidence status

Evidence entries use four primary states:

* **Pending** — required evidence has not yet been published.
* **In progress** — implementation or verification is actively underway.
* **Verified** — evidence has been completed and linked to a reproducible or independently inspectable artifact.
* **Not applicable** — the requirement does not apply to the relevant release or component.

Evidence marked **Verified** should link to the underlying artifact rather than relying solely on a written assertion.

***

### Milestone evidence

For each significant implementation or deployment milestone, AmpliDex will publish an evidence package containing, where applicable:

1. an immutable source commit and release tag;
2. deployed testnet or mainnet contract IDs;
3. reproducible contract artifact and WASM hashes;
4. automated test, property-test, fuzz-test, and benchmark results;
5. relevant security review and remediation evidence;
6. deployment and configuration information;
7. example on-chain transactions or a reproducible demonstration; and
8. a concise mapping between the milestone's technical claims and supporting evidence.

This provides a traceable path from **architecture → implementation → deployment → verification → evidence**.

***

### Mainnet progression

Production readiness is not treated as a single binary event.

AmpliDex is expected to progress through controlled deployment stages, beginning with testnet validation and moving toward capped mainnet operation before unrestricted deployment.

Initial mainnet markets may therefore operate with conservative supply, borrow, position, and exposure limits. Limits are increased only after predefined operational, liquidity, risk, and reliability criteria have been satisfied.

This approach allows production evidence to be accumulated under controlled economic exposure while maintaining explicit criteria for broader protocol activation.
