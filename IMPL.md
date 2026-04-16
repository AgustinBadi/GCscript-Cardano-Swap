# Implementation Plan

## Actions

Three GCscript transactions to implement, in order of priority:

### 1. Open Limit-Swap

Create a one-way swap UTxO at the user's personal DApp address.

- Derive personal DApp address from spending validator + user staking credential
- Beacon names pre-computed externally and injected via `args` (ISL sha256 does not accept raw bytes)
- Mint exactly 1 of each beacon (pair, offer, ask) via `CreateOrCloseSwaps` redeemer (constructor 1)
- Any unused beacons must be burned in the same transaction
- Construct `SwapDatum` inline in GCscript using `args` and runtime cache values:
  - `beacon_id` == beacon policy id (hardcoded: `c4d7d117d9ebcde6db28db40837ff2b1401e9eaaa6eecea9e070e209`)
  - `pair_beacon`, `offer_beacon`, `ask_beacon` must match the minted beacon asset names
  - `offer_id` / `offer_name` and `ask_id` / `ask_name` must be different assets
  - `swap_price` numerator and denominator must both be > 0
  - `expiration` if set must be divisible by 60,000ms
- Output: swap UTxO at personal DApp address with beacons + offered asset + datum
- Output address must include a staking credential (no signing check at open time)
- No extraneous assets in the output UTxO

### 2. Take Limit-Swap (Execute)

Fill an existing open limit order.

- Two-step flow: Query runs first (external to GCscript), user selects a swap, Execute URL is generated with that specific UTxO hardcoded
- Consume the swap UTxO with `Swap` redeemer
- Construct output datum with `prev_input` set to the consumed UTxO's `TxOutRef`
- Deposit ask asset, receive offer asset
- Set `invalid-hereafter` only if the swap datum has an `expiration`
- Permissionless — no signing requirement

### 3. Close Limit-Swap

Cancel an existing open limit order and reclaim assets.

- Consume the swap UTxO with `SpendWithMint` redeemer on the spending validator
- Burn all 3 beacons via `CreateOrCloseSwaps` redeemer (constructor 1) on the beacon minting policy
- All 3 beacons must be burned — no stray beacons may remain
- Return remaining assets (ask asset accumulated + remaining offer asset) to owner
- Staking credential must sign

---

## Future Options

- **GCscript MCP server** — an MCP server wrapping GCscript tooling would close the development feedback loop: schema validation against the official JSON schema, GameChanger URL encoding (gzip + base64url), and script templates. Would allow Claude to validate and test GCscript inline during development without sending scripts to the wallet blindly. TypeScript + `@gamechanger-finance/gc` npm package is the natural implementation path.

## Architecture Decisions

- **Beacon names pre-computed externally**: ISL `sha256()` accepts UTF-8 strings, not raw bytes. The three beacon names require hashing raw byte arrays (including the ADA `0x00` substitution for the pair beacon). They must be computed outside GCscript and injected via `args`.
- **Beacon script pre-parameterized offline**: `swap_script` has no parameters and is used as-is. `beacon_script` takes `dapp_hash` (the spending validator script hash) as a validator-level parameter. This is applied once offline via `aiken blueprint apply` and the resulting CBOR is hardcoded directly in the script — no runtime `scriptParams` needed. Parameterized beacon policy hash: `c4d7d117d9ebcde6db28db40837ff2b1401e9eaaa6eecea9e070e209`.
- **Script CBORs already compiled**: `contracts/cardano-swaps/aiken/plutus.json` contains the compiled CBOR for both validators. No Aiken compilation step needed.

## Open Issues

- **UTxO filtering**: confirm with GCscript developer whether filtering by asset pair can be done inside the query or must be handled externally between Query and Execute steps
- ~~**`scriptHex` after `scriptParams`**~~: **Resolved.** GCscript returns the parameterized CBOR in `scriptHex` after applying `scriptParams`. Confirmed from the `Plutus Script Parametrization` example — the parameterized script produces a distinct `scriptHashHex`, meaning the CBOR itself changed. `cache.beaconPolicy.scriptHex` is safe to use as the minting witness script.
- **ADA-as-offer edge case**: the datum encodes ADA's policy as empty bytes `""`, but GCscript asset specs use `"ada"`. A single `args.offerPolicyId` cannot serve both roles without ISL conditionals (which don't exist). Needs two separate args or a dedicated ADA-offer script variant.
- **Execute datum requires dynamic `prev_input`**: The output datum must include `prev_input` set to the `TxOutRef` of the consumed swap UTxO (tx hash + output index). This value is not known until the UTxO is selected, so the full `SwapDatum` must be computed externally after UTxO selection and injected via `args.datumHex`.
