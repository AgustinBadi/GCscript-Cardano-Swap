# Implementation Plan

## Actions

Three GCscript transactions implemented and working on Preview testnet:

### 1. Open Limit-Swap ✅

Create a one-way swap UTxO at the user's personal DApp address.

- Derive personal DApp address from spending validator + user staking credential
- Beacon names computed inside GCscript using `sha256(flatten(hexToByteArray(...), ...))` — no external pre-computation needed
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

### 2. Take Limit-Swap (Execute) ✅

Fill an existing open limit order.

- Two-step flow: interface queries UTxOs externally, user selects a swap, args are injected into the script
- Consume the swap UTxO with `Swap` redeemer (constructor 2)
- Construct output datum with `prev_input` set to the consumed UTxO's `TxOutRef` — built inside GCscript from `args.swapTxHash` and `args.swapTxIndex`
- Deposit ask asset, receive offer asset
- Set `invalid-hereafter` only if the swap datum has an `expiration`
- Permissionless — no signing requirement
- `outputAdaQuantity` must be the total output ADA (current UTxO ADA + deposit), not just the deposit — pre-computed by the interface

### 3. Close Limit-Swap ✅

Cancel an existing open limit order and reclaim assets.

- Consume the swap UTxO with `SpendWithMint` redeemer (constructor 0) on the spending validator
- Burn all 3 beacons via `CreateOrCloseSwaps` redeemer (constructor 1) on the beacon minting policy
- All 3 beacons must be burned — no stray beacons may remain
- Return remaining assets (ask asset accumulated + remaining offer asset) to owner via change
- Staking credential must sign — `requiredSigners` takes the raw 28-byte key hash hex (not bech32)

---

## Future Options

- **GCscript MCP server** — an MCP server wrapping GCscript tooling would close the development feedback loop: schema validation against the official JSON schema, GameChanger URL encoding (gzip + base64url), and script templates. Would allow Claude to validate and test GCscript inline during development without sending scripts to the wallet blindly. TypeScript + `@gamechanger-finance/gc` npm package is the natural implementation path.

## Architecture Decisions

- **Beacon names computed inside GCscript**: ISL `sha256()` accepts byte arrays via `hexToByteArray()` and `flatten()`. Beacon hashes are computed at runtime inside the script — no external pre-computation or `args` injection needed. ADA's empty policy ID is substituted with `hexToByteArray('00')` (single byte) before hashing.
- **Beacon hash formulas** (from Aiken source `one_way_swap/utils.ak`):
  ```
  pairBeacon  = sha256(offerPolicyId || offerAssetName || 0x00 || askAssetName)
  offerBeacon = sha256(0x01 || offerPolicyId || offerAssetName)
  askBeacon   = sha256(0x02 || askPolicyId || askAssetName)
  ```
  The `0x00` in `pairBeacon` substitutes ADA's empty policy ID — it is NOT a prefix byte.
- **Beacon script pre-parameterized offline**: `swap_script` has no parameters and is used as-is. `beacon_script` takes `dapp_hash` (the spending validator script hash) as a validator-level parameter. This is applied once offline via `aiken blueprint apply` and the resulting CBOR is hardcoded directly in the script — no runtime `scriptParams` needed. Parameterized beacon policy hash: `c4d7d117d9ebcde6db28db40837ff2b1401e9eaaa6eecea9e070e209`.
- **Script CBORs already compiled**: `contracts/cardano-swaps/aiken/plutus.json` contains the compiled CBOR for both validators. No Aiken compilation step needed.
- **Double-CBOR wrapping for spending validator**: The spending validator scriptHex must be double-wrapped (`591196591193...`) so GCScript's internal hash matches the Cardano-standard blake2b_224 hash. See TODO.md §3.
- **GCT-offer / ADA-ask pair chosen**: ADA-as-offer causes minimum UTxO ADA bumps when the UTxO gains a token type on execution, causing the net ADA received to be less than expected. Offering a native token (GCT) and asking for ADA avoids this — depositing ADA only increases lovelace without triggering minimum recalculation.
- **`prev_input` in output datum**: Built inside GCscript from `args.swapTxHash` and `args.swapTxIndex`. Encoded as `Some(OutputReference)` = constructor 0 wrapping constructor 0 `{ fields: [constructor 0 { fields: [bytes(txHash)] }, int(txIndex)] }`.
- **Inline datum — no datum in consumer witness**: With Plutus V2 inline datums, the `datum` field must NOT be included in the spend consumer witness. Adding it causes an `extraneousDatums` rejection by the node.
- **`requiredSigners` format**: GCScript's `buildTx.requiredSigners` accepts raw 28-byte key hash hex strings. Bech32 stake addresses fail with a CBOR deserialization error (expected 28 bytes, got 0).

## Open Issues

- **UTxO filtering**: confirm with GCscript developer whether filtering by asset pair can be done inside the query or must be handled externally between Query and Execute steps.
- ~~**`scriptHex` after `scriptParams`**~~: **Resolved.** GCscript returns the parameterized CBOR in `scriptHex` after applying `scriptParams`.
- ~~**Beacon names pre-computed externally**~~: **Resolved.** ISL `sha256(flatten(hexToByteArray(...), ...))` handles raw byte hashing inside the script.
- ~~**Execute datum requires dynamic `prev_input`**~~: **Resolved.** Constructed inside GCscript from `args.swapTxHash` and `args.swapTxIndex` using nested `plutusData` constructors.
- ~~**ADA-as-offer edge case**~~: **Avoided.** Switched to GCT-offer / ADA-ask pair. ADA-as-offer is deferred to a future script variant.
- **Deployment constants hardcoded in scripts**: The beacon policy ID and spending validator hash are currently hardcoded strings in all three scripts. They should be moved to `args` so the frontend controls them and a network/deployment change only requires updating one place.
  - `beacon-policy-id`: `c4d7d117d9ebcde6db28db40837ff2b1401e9eaaa6eecea9e070e209`
  - `spending-validator-hash`: `1d6cff26bcab91d2061aad0bd259cbb7d76d25ced2eeaed5926a42ad`
