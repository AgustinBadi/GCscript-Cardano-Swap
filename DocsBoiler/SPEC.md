# Specification

<!-- One sentence: what problem does this application solve and for whom? -->

---

## 1. Goal & Scope

<!-- What is this system trying to achieve?
     Be concrete — describe the end state, not the journey.
     Example: "Allow users to lock ADA as collateral and mint a stablecoin against it." -->

### In Scope

<!-- List the features and behaviors this system WILL have. -->

### Out of Scope

<!-- Explicitly list what this system will NOT do.
     This is as important as the in-scope list — it prevents scope creep
     and resolves ambiguity during implementation. -->

---

## 2. Actors & Roles

<!-- Who interacts with this system?
     For each actor, describe: who they are, what they can do, and any trust assumptions.
     Examples: End User, Admin, Validator Node, Oracle, Smart Contract -->

| Actor | Description | Permissions / Trust Level |
|-------|-------------|---------------------------|
|       |             |                           |

---

## 3. Data Model

<!-- Define every piece of data the system stores or passes around.
     For on-chain data: specify Datum and Redeemer types for each validator.
     For off-chain data: specify any types the frontend/backend uses.
     Be explicit about field names, types, constraints, and what each field means. -->

### On-Chain

#### Datums

```
-- Example:
-- MyDatum {
--   owner    : PubKeyHash   -- the wallet that locked the funds
--   deadline : POSIXTime    -- after this time the funds can be reclaimed
-- }
```

#### Redeemers

```
-- Example:
-- data MyRedeemer = Claim | Reclaim
```

#### Minting Policy (if applicable)

```
-- Describe token name, amount rules, and who can mint/burn.
```

### Off-Chain

<!-- Types used by the frontend, backend, or transaction builder. -->

---

## 4. Business Rules & Invariants

<!-- The rules the system must NEVER violate.
     Write these as plain assertions, not code.
     These map directly to your validator logic.
     Example:
       - A user may only withdraw funds they deposited.
       - Minting is only allowed when collateral ratio > 150%.
       - Once locked, funds cannot be moved before the deadline. -->

-
-
-

---

## 5. Transactions

<!-- Define every transaction type the system supports.
     For each transaction, specify:
       - Purpose: what does this tx accomplish?
       - Inputs: UTxOs consumed (with datum/redeemer)
       - Outputs: UTxOs produced (with datum and value)
       - Minting/Burning: tokens minted or burned (if any)
       - Required Signers: which keys must sign
       - Validity Range: time constraints (if any)
       - Success Conditions: what must be true for the tx to validate
       - Failure Conditions: what causes the tx to be rejected -->

### Tx 1: [Name]

**Purpose:**

**Inputs:**
- UTxO at `[Script/Wallet]` with datum `[...]`, consumed with redeemer `[...]`

**Outputs:**
- UTxO at `[Script/Wallet]` with datum `[...]` and value `[...]`

**Required Signers:** 

**Validity Range:** 

**Success Conditions:**
-

**Failure Conditions:**
-

---

<!-- Copy the Tx block above for each additional transaction type. -->

---

## 6. Off-Chain Logic

<!-- Describe the application logic that runs outside the chain.
     This includes: transaction building, wallet interaction, indexing, API endpoints, UI flows.
     You don't need to specify implementation details — just behavior and responsibilities. -->

---

## 7. Error Cases

<!-- What can go wrong, and what should the system do?
     Cover both on-chain rejections and off-chain failures.
     Example:
       - Tx submitted with expired validity range → display "Transaction expired, please retry"
       - Insufficient collateral → block minting and show required amount -->

| Scenario | Expected Behavior |
|----------|-------------------|
|          |                   |

---

## 8. Open Questions

<!-- Unresolved decisions that must be answered before implementation begins.
     Do not start coding while this section is non-empty.
     Remove items as they are resolved — record the decision in the relevant section above. -->

- [ ] 
