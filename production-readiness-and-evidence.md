---
description: Due-diligence gates and evidence register for grants, investment review, and mainnet launch.
---

# Production Readiness & Evidence

## Readiness rule

{% hint style="danger" %}
Do not describe a component as production-ready solely because it appears in the architecture. Production status requires a deployed artifact, tests, independent review where applicable, operational ownership, and linked evidence.
{% endhint %}

## Launch gates

- [ ] Formal economic specification and parameter rationale approved
- [ ] Critical invariants represented in unit, property, and stateful fuzz tests
- [ ] Mixed-decimal, rounding, time-jump, dust, and loss scenarios pass
- [ ] Storage migrations pass against a production-shaped snapshot
- [ ] DEX routes and oracle sources independently verified
- [ ] Degraded oracle/route behavior and emergency exits tested
- [ ] Keeper redundancy and liquidation race tests pass
- [ ] Indexer replay, reconciliation, backup, and restore drills pass
- [ ] Multisig, timelock, guardian limits, and key rotation verified
- [ ] Independent findings resolved or publicly accepted
- [ ] Monitoring, paging, incident response, and status communication live
- [ ] Contract IDs, source commit, WASM hashes, and configuration published
- [ ] Vulnerability disclosure and funded bug bounty operational
- [ ] Conservative canary caps and cap-increase criteria approved

Cross-chain access and confidential positions have independent approval gates.

## Evidence register

Complete this table before submitting external materials.

| Evidence | Link / identifier | Status |
|---|---|---|
| Source repository and release tag | TBD | Required |
| Testnet contract IDs | TBD | Required |
| Mainnet contract IDs | TBD | When deployed |
| Reproducible WASM hashes | TBD | Required |
| CI and automated test report | TBD | Required |
| Formal economic specification | TBD | Required |
| Audit report and remediation commit | TBD | Required before mainnet |
| Deployment/configuration manifest | TBD | Required |
| Monitoring/status page | TBD | Required before mainnet |
| Vulnerability disclosure / bounty | TBD | Required before mainnet |
| Governance signers and timelock | TBD | Required before mainnet |

## Grant acceptance package

For each milestone, publish:

1. a repository tag and immutable commit;
2. deployed testnet or mainnet contract IDs;
3. reproducible artifact hashes;
4. automated test and benchmark output;
5. relevant audit or review evidence;
6. example transactions or a reproducible demo; and
7. a concise report mapping every acceptance criterion to evidence.

This structure gives assessors objective proof of delivery and gives investors a clean path from architecture claims to operational facts.
