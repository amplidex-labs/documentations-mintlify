# Research - Selective Position Confidentiality

## Research — Selective Position Confidentiality

> **Status:** Research track. Not part of the initial production security boundary.

AmpliDex is researching selective confidentiality for position parameters whose public disclosure could create an exploitable information advantage against traders.

The objective is **not** to make all protocol activity private or to provide complete transaction anonymity. Instead, the design aims to preserve public market transparency, protocol-level solvency visibility, LP risk monitoring, and permissionless liquidation while protecting position-specific information that may expose a trader's strategy, risk tolerance, or liquidation vulnerability.

Selective confidentiality is independently staged from the core AmpliDex protocol and is not required for initial production deployment.

### Design objective

Public blockchains provide strong transparency, but full visibility into leveraged positions may also expose economically sensitive information.

For example, observers may be able to use exact position parameters to infer:

* where a trader entered a market;
* the size of a trader's position;
* the amount of margin supporting the position;
* the trader's effective leverage;
* whether the position is profitable or under stress;
* how far the position is from liquidation; or
* where forced execution may occur.

AmpliDex therefore investigates whether selected trader-specific parameters can remain confidential while preserving the economic guarantees required by the protocol.

The target is:

> **Publicly verifiable solvency and liquidation eligibility without unnecessarily revealing trader-specific risk parameters.**

Confidentiality must never weaken protocol accounting, solvency enforcement, liquidation liveness, or LP transparency.

### Confidential state and derived information

Encrypted frontend state alone is insufficient for values that influence authoritative protocol decisions.

Confidential inputs or state used for protocol accounting, solvency, or liquidation must be cryptographically bound to the authoritative position state and enforced through proof-constrained transitions or another independently verifiable mechanism.

Not every confidential value must necessarily be represented as independently committed state.

Some values may be derived from confidential source parameters and simply not published.

For example:

```
Confidential source state:

collateral
debt
position holdings
entry information

        ↓

Derived information:

effective leverage
unrealized equity / PnL
margin ratio
liquidation price
distance to liquidation
```

The final design should minimize the amount of confidential authoritative state while avoiding unnecessary disclosure of derived trader information.

### Candidate confidential parameters

Candidate confidential source parameters may include:

* entry price or entry information;
* exact collateral or margin amount;
* exact debt amount;
* borrowed notional;
* exact position notional;
* exact position holdings; and
* private position-management parameters where supported.

Sensitive values that may instead be derived without being publicly disclosed include:

* effective leverage or borrow multiplier;
* unrealized equity;
* unrealized PnL;
* exact margin ratio;
* liquidation price;
* distance to liquidation; and
* private stop, close, or take-profit thresholds.

These parameters are candidates for confidentiality because their public disclosure may allow external participants to infer a trader's strategy, identify stress points, estimate forced-execution levels, or trade around liquidation conditions.

Not every field must be confidential in every implementation.

The confidentiality model may differ by market, position type, protocol version, or privacy mode.

### Public protocol state

Individual position confidentiality does not imply confidential protocol accounting.

Market-level and protocol-level information may remain public where required for transparency, solvency monitoring, LP risk evaluation, and permissionless protocol operation.

Public fields may include:

* market;
* position identifier or commitment;
* position status;
* protocol version;
* state version;
* proof-policy identifier;
* commitment;
* nullifier or nonce state;
* aggregate pool cash;
* aggregate debt;
* utilization;
* borrow index;
* aggregate open interest;
* market caps;
* oracle configuration;
* maintenance-margin requirements;
* protocol fees; and
* other market-level risk parameters.

Position direction, account identity, and coarse risk state may remain public or may be selectively hidden depending on the final privacy model.

For example, the protocol may expose:

```
Position:      #18429
Market:        XLM / USDC
Status:        Open
Risk state:    Healthy
Commitment:    0x83f...
Version:       7
```

without necessarily exposing:

```
Entry price
Exact collateral
Exact debt
Exact leverage
Exact position size
Current equity
Margin ratio
Liquidation price
```

This preserves protocol transparency while reducing the amount of information that can be directly used against an individual trader.

### Pool transparency

Selective confidentiality applies to individual position information, not necessarily to aggregate market accounting.

AmpliDex may continue to expose information such as:

```
total_cash
total_borrows
utilization
borrow_index
total_open_interest
market_caps
```

while withholding trader-specific values such as:

```
account collateral
entry price
exact borrowed amount
exact position size
margin ratio
liquidation threshold
```

This separation allows liquidity providers and market participants to evaluate protocol-level utilization, debt, liquidity, and risk without requiring the complete economic state of each trader to be publicly observable.

Pool accounting must remain independently reconcilable under any confidential-position design.

### Risk-state confidentiality

A selective-confidentiality design may expose a coarse, verifiable risk classification rather than exact underlying margin parameters.

For example:

```
HEALTHY
WARNING
LIQUIDATABLE
```

The protocol must still verify that the classification is correct, but external participants do not necessarily need access to the exact collateral, debt, equity, margin ratio, or liquidation threshold used to derive it.

Conceptually, a proof may establish:

```
margin_ratio > maintenance_margin
```

without revealing the exact margin ratio.

Similarly, liquidation eligibility may establish:

```
margin_ratio <= maintenance_margin
```

without necessarily revealing all underlying position parameters.

Risk classifications must never be treated as permanently valid.

Any published or proven risk classification is valid only against the:

* oracle observations;
* accrued debt state;
* borrow index;
* position commitment;
* protocol configuration;
* state version; and
* validity conditions

bound into that evaluation.

Liquidation eligibility must therefore be revalidated against authoritative state at execution.

### Proof-bound transitions

Where zero-knowledge or commitment-based mechanisms are used, a valid proof must bind the relevant state transition to its intended authorization and execution domain.

Depending on the operation, this may include:

* Stellar network;
* protocol contract;
* position ID;
* market;
* previous commitment;
* operation;
* new commitment;
* state version;
* nonce or nullifier;
* approved oracle inputs;
* relevant borrow index;
* relevant protocol configuration; and
* validity window.

A proof must not be replayable across:

* networks;
* contracts;
* positions;
* markets;
* commitments;
* state versions;
* authorization domains; or
* expired validity windows.

State transitions must either consume or otherwise invalidate the appropriate previous state so that stale proofs cannot modify an already-updated position.

### Economic invariants

Confidential-mode transitions must preserve the same economic rules as public positions.

These include:

* ownership;
* authorization;
* debt conservation;
* collateral accounting;
* LP and pool accounting;
* borrow-index consistency;
* interest accrual;
* fee accounting;
* solvency;
* margin requirements;
* position-state transitions;
* liquidation eligibility;
* liquidation settlement;
* residual collateral handling; and
* bad-debt accounting.

Selective confidentiality must never weaken the economic guarantees of the public protocol.

A transition that cannot demonstrate compliance with the required economic invariants must not be accepted solely because its underlying values are confidential.

### Liquidation

Selective confidentiality must preserve permissionless liquidation.

A confidential position cannot rely on secrecy to prevent or delay liquidation.

The system must provide a verifiable mechanism for determining whether a position satisfies the configured liquidation condition.

Conceptually, a healthy position may satisfy:

```
margin_ratio > maintenance_margin
```

while a liquidatable position satisfies:

```
margin_ratio <= maintenance_margin
```

without necessarily revealing the precise inputs used in those calculations.

However, proof of a historical risk state is insufficient for liquidation.

Before settlement, liquidation eligibility must be revalidated against the authoritative current:

* position state;
* debt state;
* borrow index;
* approved oracle observations;
* applicable execution valuation;
* protocol configuration; and
* liquidation parameters.

Liquidation settlement must preserve:

* current debt accounting;
* pool repayment;
* configured fees and incentives;
* residual user value;
* atomic eligibility revalidation; and
* explicit bad-debt recognition.

Confidentiality must not give a position weaker liquidation guarantees than an equivalent public position.

### Oracle and valuation binding

Any confidential solvency or liquidation proof that depends on market value must be bound to protocol-approved valuation inputs.

A proof must not allow a prover to substitute:

* stale prices;
* unauthorized oracle sources;
* unsupported markets;
* outdated protocol parameters; or
* favorable historical observations.

Where AmpliDex uses both independent reference pricing and realizable execution value, the confidential risk model must preserve the same conservative valuation rules used by public positions.

Privacy does not alter the protocol's underlying economic specification.

### Privacy limitations

Selective position confidentiality does not guarantee complete transaction anonymity or complete strategy secrecy.

Information may remain observable or inferable from:

* token transfers;
* DEX interactions;
* transaction timing;
* wallet activity;
* transaction frequency;
* execution size;
* price impact;
* liquidity changes;
* account relationships;
* access patterns; and
* other public blockchain metadata.

For example, hiding a position's exact notional in committed state does not necessarily prevent an observer from estimating its size if a corresponding DEX execution produces a clearly observable market impact.

AmpliDex therefore does not treat selective position confidentiality as equivalent to anonymous trading.

The objective is to reduce unnecessary disclosure of economically sensitive position information, not to claim that all associated activity is unobservable.

### Research path

Selective confidentiality is investigated progressively rather than as a single all-or-nothing privacy feature.

#### Stage 1 — Disclosure minimization

Reduce unnecessary publication of trader-specific information that is not required for authoritative protocol operation.

This may include avoiding unnecessary publication of:

* entry information;
* derived PnL;
* derived liquidation price;
* trader analytics; and
* other non-authoritative derived values.

This stage may not require zero-knowledge proofs.

#### Stage 2 — Private intents and commit-reveal

Evaluate mechanisms that reduce information leakage before execution, including:

* private intents;
* commit-reveal workflows;
* delayed parameter disclosure; and
* other mechanisms that reduce pre-execution strategy leakage.

#### Stage 3 — Selective position-parameter confidentiality

Evaluate commitment-based representation of economically sensitive position state while preserving public aggregate accounting.

Candidate protected values may include:

* collateral;
* debt;
* position size;
* position holdings; and
* related trader-specific parameters.

#### Stage 4 — Confidential risk-state proofs

Evaluate proofs that allow the protocol to verify solvency or liquidation conditions without exposing all underlying position parameters.

The target is to prove conditions such as:

```
equity > required_margin
```

or:

```
margin_ratio <= maintenance_margin
```

without publishing every input used in the calculation.

#### Stage 5 — Additional execution-privacy research

Evaluate stronger execution privacy only where technically feasible, economically justified, and compatible with Stellar and supported execution venues.

Full position or transaction anonymity is not a prerequisite for selective position confidentiality.

### Proof-system evaluation

No proof system is assumed solely because it provides zero-knowledge functionality.

Proof-system selection is based on measured:

* Soroban verifier cost;
* proof size;
* proof-generation latency;
* browser proving time;
* browser memory requirements;
* mobile feasibility;
* verification latency;
* setup assumptions;
* trusted-setup requirements where applicable;
* tooling maturity;
* test-vector quality;
* circuit complexity;
* verifier complexity;
* implementation maturity; and
* auditability.

The evaluation must also consider whether the confidentiality benefit justifies the additional:

* proving complexity;
* verification cost;
* contract complexity;
* operational complexity;
* user latency;
* failure modes; and
* security-review surface.

A simpler confidentiality mechanism is preferred where it provides the required protection without introducing unnecessary cryptographic complexity.

### Production activation

Selective position confidentiality is independently gated from the core AmpliDex production release.

Production activation requires evidence that the selected implementation:

* preserves the economic invariants of public positions;
* preserves authoritative pool accounting;
* does not weaken liquidation liveness;
* does not obscure protocol-level solvency;
* does not prevent LP or pool-risk monitoring;
* correctly binds proofs to authoritative oracle and debt state;
* prevents replay of confidential state transitions;
* safely invalidates stale proofs and commitments;
* preserves conservative valuation rules;
* does not introduce unacceptable Soroban verification costs;
* provides acceptable browser and mobile proving performance where applicable;
* has documented recovery and failure behavior; and
* passes independent review of the relevant circuit, commitment scheme, verifier, contract integration, and economic logic.

Production activation must also define explicit monitoring, incident-response, upgrade, and disablement procedures for the confidential mode.

Any selective-confidentiality capability remains opt-in and independently gated from the core AmpliDex production security boundary.

Failure or disablement of the confidentiality feature must not compromise the accounting, solvency, or operation of public AmpliDex markets.
