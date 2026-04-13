# Implementation Plan

## Actions

Three GCscript transactions to implement, in order of priority:

### 1. Open Limit-Swap

Create a one-way swap UTxO at the user's personal DApp address.

- Derive personal DApp address from spending validator + user staking credential
- Compute beacon names via SHA256 hashing inside GCscript
- Mint 3 beacons (pair, offer, ask) via `CreateOrCloseSwaps` redeemer
- Construct `SwapDatum` inline with offer asset, ask asset, price, beacon fields
- Output: swap UTxO at personal DApp address with beacons + offered asset + datum
- Staking credential must sign

### 2. Take Limit-Swap (Execute)

Fill an existing open limit order.

- Two-step flow: Query runs first (external to GCscript), user selects a swap, Execute URL is generated with that specific UTxO hardcoded
- Consume the swap UTxO with `Swap` redeemer
- Construct output datum with `prev_input` set to the consumed UTxO's `TxOutRef`
- Deposit ask asset, receive offer asset
- Set `invalid-hereafter` on the transaction
- Permissionless — no signing requirement

### 3. Close Limit-Swap

Cancel an existing open limit order and reclaim assets.

- Consume the swap UTxO with `SpendWithMint` redeemer on the spending validator
- Burn all 3 beacons via `CreateOrCloseSwaps` redeemer on the beacon minting policy
- Return remaining assets to owner
- Staking credential must sign

---

## Future Options

- **GCscript MCP server** — an MCP server wrapping GCscript tooling would close the development feedback loop: schema validation against the official JSON schema, GameChanger URL encoding (gzip + base64url), and script templates. Would allow Claude to validate and test GCscript inline during development without sending scripts to the wallet blindly. TypeScript + `@gamechanger-finance/gc` npm package is the natural implementation path.

## Open Issues

- **UTxO filtering**: confirm with GCscript developer whether filtering by asset pair can be done inside the query or must be handled externally between Query and Execute steps
- **Script CBORs**: compile spending validator and beacon minting policy from cardano-swaps Aiken source before any transaction can be built
