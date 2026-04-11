# Source: A Peer-to-Peer DeFi Application on Cardano using GCscript

## General Description

This application is an off-chain implementation of a P2P DeFi protocol on Cardano using GCscript.

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
- The beacon token (presence = validity signal)
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