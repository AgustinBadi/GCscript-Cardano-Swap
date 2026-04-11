# Implementation Plan

<!-- This document describes HOW the system in SPEC.md will be built.
     It is not the spec itself — it references SPEC.md as the source of truth.
     Update this as architectural decisions are made or revised.
     If something here contradicts SPEC.md, SPEC.md wins — fix the conflict before coding. -->

---

## 1. Technology Stack

<!-- What languages, frameworks, and tools will be used and why?
     Be explicit about version choices if they matter.
     Examples for Cardano:
       - Smart contract language: Aiken / Plutus / Helios / Marlowe
       - Off-chain: TypeScript + Lucid / Mesh / Cardano-js-sdk
       - Wallet connector: CIP-30 (Nami, Eternl, Lace...)
       - Node access: Blockfrost / Maestro / own node
       - Frontend: React / Svelte / plain HTML
       - Testing: Aiken built-in tests / Jest / Vitest -->

| Layer | Choice | Reason |
|-------|--------|--------|
| Smart contracts | | |
| Off-chain / tx building | | |
| Node / chain indexer | | |
| Frontend | | |
| Testing | | |
| Network target | Preview / Preprod / Mainnet | |

---

## 2. Repository Structure

<!-- How is the code organized? List top-level directories and their purpose.
     Example:
       /validators   — Aiken validator scripts
       /lib          — shared on-chain helper functions
       /offchain     — transaction builders and chain queries
       /frontend     — UI
       /tests        — test suite -->

```
/
├── ...
```

---

## 3. On-Chain Architecture

<!-- How do the validators fit together?
     - Which validators exist and what UTxO pattern does each guard?
     - How do validators reference each other (e.g. staking validator + spend validator)?
     - Any minting policies and how are they parameterized?
     - Diagram the UTxO flow if it helps. -->

### Validators

| Validator | Type (Spend/Mint/Withdraw) | Guards |
|-----------|---------------------------|--------|
| | | |

### UTxO Flow Diagram

```
<!-- ASCII or plain-text diagram showing how UTxOs move between wallets and scripts. -->
```

---

## 4. Off-Chain Architecture

<!-- How does the application layer interact with the chain?
     - How are transactions built and submitted?
     - How is chain state read (queries, event indexing)?
     - Is there a backend server, or is everything client-side?
     - How are errors surfaced to the user? -->

---

## 5. Transaction Builders

<!-- For each transaction in SPEC.md §5, describe the implementation approach.
     - Which library calls build the tx?
     - How are datums serialized?
     - How is fee estimation handled?
     - Any edge cases in construction worth noting? -->

### Tx: [Name — match SPEC.md name]

<!-- Implementation notes -->

---

<!-- Copy the Tx block above for each transaction in the spec. -->

---

## 6. Testing Strategy

<!-- How will correctness be verified against the spec?
     - Unit tests: individual validator logic
     - Integration tests: full transaction flows on emulator or testnet
     - What scenarios are mandatory to test before mainnet?
     Map tests back to the invariants in SPEC.md §4 — every invariant should have a test. -->

### On-Chain Tests

| Invariant (from SPEC §4) | Test Description | Status |
|--------------------------|------------------|--------|
| | | |

### Off-Chain / Integration Tests

| Scenario | Test Description | Status |
|----------|------------------|--------|
| | | |

---

## 7. Deployment Plan

<!-- Steps to go from dev to production.
     - How are validators compiled and hashed?
     - How is the script address derived and verified?
     - How are initial UTxOs seeded (e.g. reference scripts, protocol init)?
     - Testnet checklist before mainnet deploy. -->

### Pre-Mainnet Checklist

- [ ] All SPEC.md invariants covered by tests
- [ ] Validators audited / reviewed
- [ ] Script hashes verified against deployed contracts
- [ ] Testnet end-to-end run completed
- [ ] Emergency / upgrade path defined

---

## 8. Known Constraints & Trade-offs

<!-- Record architectural decisions that have a cost.
     For each: what was chosen, what was the alternative, and why this path was taken.
     This prevents future "why did we do it this way?" confusion.
     Example:
       - Chose inline datums over datum hashes: larger tx size but simpler indexing -->

| Decision | Alternative Considered | Reason for Choice |
|----------|------------------------|-------------------|
| | | |
