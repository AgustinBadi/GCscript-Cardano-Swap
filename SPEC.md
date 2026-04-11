# Source: A Peer-to-Peer DeFi Application on Cardano using GCscript

## Goal & Scope

This application showcases the power of GCscript by implementing an off-chain interface for the Cardano-Swaps P2P DeFi protocol. The goal is to generate and submit transactions that interact with the protocol directly from a GameChanger wallet. The current implementation is focused on One-Way Swaps (Limit Orders).

### In Scope

This application supports the following operations on One-Way Swaps (Limit Orders):

* **Create** — open a limit order specifying: offer asset, ask asset, price (as a ratio), and change address (where leftover ADA is returned).
* **Query** — fetch open limit orders by beacon policy ID and filter by asset pair.
* **Execute** — fill a limit order by depositing the ask asset and receiving the offer asset.

### Out of Scope

This application WILL NOT address:

* Production readiness — this is a prototype only.
* A user interface — interactions are driven directly via GCscript URLs.
* One-way swap composition.
* Any operation regarding Two-Way Swaps (Liquidity Swaps).

### Future Options

* **Close/Cancel swap** — closing an existing limit order (burning beacons, reclaiming assets) is not in scope for this prototype but is a natural next operation to implement.

### Application Flow

This is a prototype with no UI. Each flow is triggered by opening a GCscript URL directly in the GameChanger wallet. A UI layer is optional and out of scope.

**As a Swap Owner:**
1. *(Optional UI)* Specify offer asset, ask asset, price, and change address → generates GCscript URL
2. Open GCscript URL in GameChanger wallet
3. Wallet derives personal DApp address, builds and submits Open Swap transaction → 3 beacons minted, swap UTxO created

**As a Taker:**
1. *(Optional UI)* Query open limit orders by beacon policy ID, filter by asset pair, select a swap → generates GCscript URL
2. Open GCscript URL in GameChanger wallet
3. Wallet builds and submits Execute Swap transaction → offer asset received, ask asset deposited into swap UTxO

---

## Foundational Concepts

### Beacon Tokens & Distributed DApps (CIP-0089)

Reference: https://github.com/cardano-foundation/CIPs/blob/master/CIP-0089/README.md

The DeFi protocol this application interfaces with follows the CIP-0089 pattern for distributed DApps (dDApps).

#### Address Model

Every user gets a personal DApp address derived from two components:

```
User DApp Address = DApp spending script (payment credential)
                 + User's staking credential (pubkey, native script, or Plutus script)
```

There is no single shared contract address. Each user's funds live at their own unique address while still being governed by the same spending script. The user retains staking delegation control over their locked assets.

#### Beacon Token Model

Beacon tokens are native assets that tag valid protocol UTxOs, enabling on-chain discovery without centralized indexers.

Lifecycle:
- **Minted** when a DApp UTxO is created
- **Burned** when that UTxO is spent
- **Never circulate freely** — a UTxO carrying the beacon is by definition valid

This invariant means invalid UTxOs cannot carry the beacon token, so filtering by beacon token is sufficient to find all live, valid protocol UTxOs.

#### UTxO Structure

Each protocol UTxO contains:
- The beacon token (presence = validity signal; a protocol may use more than one per UTxO)
- A datum encoding the user's intent (e.g. swap offer, loan terms)
- The locked asset(s)

#### Discovery Mechanism

Because the protocol is distributed (no single address), off-chain logic must discover UTxOs by querying the chain for the beacon token across all addresses, then filtering results client-side. There is no centralized indexer — the beacon token itself is the index.

#### Implications for This Application

- Transaction building must derive the correct personal DApp address per user (spending script + user stake key)
- UTxO discovery queries by beacon token policy ID, not by address
- The application is censorship-resistant and requires no backend infrastructure beyond chain access

---

### Cardano-Swaps Protocol

Reference: https://github.com/fallen-icarus/cardano-swaps

#### Overview

Cardano-Swaps is a P2P DEX implementing CIP-0089. Each user's swap lives at their own personal DApp address. There is no shared contract address and no centralized batcher — swaps are discovered and filled peer-to-peer via beacon tokens.

Two swap types exist:
- **One-way swap** (limit order): trade in one direction at a minimum price
- **Two-way swap** (liquidity provision): trade in either direction with a spread

This application focuses on **one-way swaps**.

#### Personal DApp Address

```
Address = one-way swap spending validator (payment credential)
        + user's staking credential (pubkey or script)
```

All of a user's swap UTxOs live at this single address. The spending validator enforces all swap rules. The staking credential authorizes owner operations.

#### One-Way Swap UTxO Structure

Each active swap UTxO contains:
- **3 beacon tokens** (qty = 1 each):
  - Pair beacon: identifies the trading pair
  - Offer beacon: identifies the offered asset
  - Ask beacon: identifies the requested asset
- **Offered asset**: the tokens being sold
- **Min 2 ADA**: for storage
- **Inline datum**: full `SwapDatum`

#### SwapDatum Fields

```
SwapDatum {
  beacon_policy_id  : PolicyId       -- beacon minting policy
  beacon_name       : AssetName      -- sha2_256(offer ++ ask)
  offer_asset       : (PolicyId, AssetName)
  ask_asset         : (PolicyId, AssetName)
  offer_beacon_name : AssetName      -- "01" ++ sha2_256(offer)
  ask_beacon_name   : AssetName      -- "02" ++ sha2_256(ask)
  price             : Rational       -- ask/offer as (numerator, denominator)
  prev_input        : Option<TxOutRef>
  expiration        : Option<POSIXTime>  -- must be divisible by 60,000 ms
}
```

ADA is represented with an empty policy ID (`""`) in asset configs.

#### Beacon Naming

```
pair_beacon  = sha2_256(offer_policy ++ offer_name ++ ask_policy ++ ask_name)
offer_beacon = "01" ++ sha2_256(offer_policy ++ offer_name)
ask_beacon   = "02" ++ sha2_256(ask_policy ++ ask_name)
```

#### Target Network

**Preprod testnet.**

#### Beacon Policy IDs (Preprod)

```
one-way beacon policy: 47cec2a1404ed91fc31124f29db15dc1aae77e0617868bcef351b8fd
two-way beacon policy: 84662c22dc5c0cadad7b2ebf9757ce9ea61dbd8fe64bc8c43c112a40
```

#### Transaction Flows

**Open Swap** (owner only)
- Mint 3 beacons via `CreateOrCloseSwaps` redeemer
- Output: swap UTxO at personal DApp address with beacons + offered asset + datum
- Staking credential must approve

**Execute Swap** (permissionless)
- Redeemer: `Swap`
- Taker deposits ask asset, takes offer asset
- Swap UTxO stays alive (partial fills allowed)
- Price constraint must hold: `offer_taken × price_num ≤ ask_given × price_den`
- `invalid-hereafter` must be set (for expiration validation)
- No approval required
- **`prev_input` handling** *(non-obvious)*: the output datum must include the `prev_input` field set to the `TxOutRef` (tx hash + output index) of the consumed swap input. The validator checks this explicitly. In GCscript, the consumed UTxO's reference must be read from the input and written into the output datum — it cannot be omitted or left as `None`.

**Close Swap** (owner only)
- Redeemer: `SpendWithMint` on spending validator + `CreateOrCloseSwaps` on beacon policy
- Burn all 3 beacons
- Owner reclaims remaining assets
- Staking credential must approve

#### Price Constraint

```
offer_taken × price_numerator ≤ ask_given × price_denominator
```

Example: price = 1/2 (offer 1 ADA per 2 DJED taken)
- Taking 100 ADA requires depositing ≥ 50 DJED
- `100 × 1 ≤ 50 × 2` → `100 ≤ 100` ✓

#### Key Invariants

- Beacons are non-transferable — minted to and burned from the swap UTxO only
- A UTxO carrying a beacon is by definition valid
- Exactly 3 beacons per swap, each with quantity = 1
- Price numerator and denominator must both be > 0
- Expiration (if set) must align to 1-minute boundaries (divisible by 60,000 ms)
- Only offer asset may leave the UTxO during execution; only ask asset (and ADA) may enter
- Up to 25 swaps can be batched in a single transaction

---

### GCscript (GameChanger DSL)

Reference: https://github.com/GameChangerFinance/gamechanger.wallet/tree/main
API Docs: https://wallet.gamechanger.finance/doc/api/v2

#### What It Is

GCscript is a JSON-based DSL for building and submitting Cardano transactions through the GameChanger wallet. It is:
- **Non-Turing complete** — deliberately limited, no arbitrary loops or recursion
- **Client-side** — executes entirely in the user's browser/wallet, not on a backend
- **Transport-agnostic** — delivered via URL, QR code, NFC, or redirect
- **Permission-based** — user reviews and approves every transaction before signing

Scripts are JSON objects where every node is a function call. The wallet parses, validates, executes, and presents the result to the user for approval.

#### Script Structure

```json
{
    "type": "script",
    "exportAs": "result",
    "run": {
        "build": { "type": "buildTx", "tx": { ... } },
        "sign":  { "type": "signTxs", "txs": "{get('cache.build')}" },
        "submit":{ "type": "submitTxs", "txs": "{get('cache.sign')}" }
    }
}
```

#### Transaction Building (`buildTx`)

```json
{
    "type": "buildTx",
    "tx": {
        "outputs":      [...],   // UTxOs to create
        "inputs":       [...],   // Optional: manually specified UTxOs to consume
        "mints":        [...],   // Native asset minting/burning
        "withdrawals":  [...],   // Stake reward withdrawals
        "certificates": [...],   // Delegation certificates
        "auxiliaryData":{...},   // Metadata
        "ttl":          12345    // Validity range (slot)
    }
}
```

Coin selection and change outputs are handled automatically.

#### Outputs

```json
{
    "address": "addr1...",
    "assets": [
        { "policyId": "ada", "quantity": "2000000" },
        { "policyId": "<hex>", "assetName": "<hex>", "quantity": "1" }
    ],
    "datum": { ... }   // inline datum as Plutus data
}
```

#### Inputs (manual UTxO selection)

```json
{
    "txHash": "<hex>",
    "index": 0,
    "address": "<addr>",
    "assets": [...],
    "redeemer": {
        "data": { ... },
        "budget": { "mem": "500000", "cpu": "200000000" }
    }
}
```

#### Minting / Burning

```json
{
    "policyId": "<hex>",
    "assetName": "<hex>",
    "quantity": "1",       // negative = burn
    "redeemer": {
        "data": { ... },
        "budget": { "mem": "...", "cpu": "..." }
    },
    "script": {
        "type": "plutusScript",
        "script": { "type": "cbor", "cbor": "<compiled_script_hex>" }
    }
}
```

#### Plutus Data

Datums and redeemers are expressed as Plutus data objects:

```json
{ "constructor": 0, "fields": [ { "int": 42 }, { "bytes": "deadbeef" } ] }
```

Primitive types: `int`, `bytes`, `list`, `map`, `constructor`.

#### Inline Scripting Language (ISL)

Dynamic values are expressed inline with `{...}` syntax:

```json
"address": "{get('cache.myStep.address')}"
"quantity": "{addBigNum(get('args.amount'), '1000000')}"
```

Key ISL functions:
- `get(path)` — read from cache or exports
- `addBigNum / subBigNum / mulBigNum` — arbitrary precision arithmetic
- `sha256() / sha512()` — hashing
- `strToHex() / hexToStr()` — encoding
- `addressBech32ToPlutusDataObj()` — convert address to Plutus data
- `getAddressInfo()` — extract payment/staking credential from address
- `fail(message)` — abort execution

No conditionals or loops in ISL — but hashing, address derivation, datum construction, and chain queries can all be handled inside the script itself.

#### Execution Flow

1. Dapp constructs GCscript JSON
2. Encodes it (gzip + base64url) into a GameChanger URL
3. User opens URL in browser → GameChanger wallet parses script
4. Wallet executes functions, builds tx, presents to user
5. User approves → wallet signs and submits
6. Result optionally returned via `returnURLPattern` redirect

#### Capabilities Relevant to This Application

- **Hashing**: SHA256/SHA512 available in ISL — beacon names can be computed inside the script
- **Address derivation**: `buildAddress` + staking credential functions can derive the personal DApp address internally; addresses can also be converted to Plutus data directly
- **UTxO queries by policy ID**: chain query functions support querying by beacon policy ID; filtering may be handled within GCscript as well
- **Plutus data serialization**: datum construction can be done inside the script
- **Execution budget estimation**: `buildTx` handles this automatically via Ogmios — no manual budget specification needed
- **Compiled scripts**: the spending validator and beacon minting policy must be provided as pre-compiled CBOR hex. Scripts are compiled from the Aiken source code in the cardano-swaps repository (https://github.com/fallen-icarus/cardano-swaps) using the Aiken compiler.