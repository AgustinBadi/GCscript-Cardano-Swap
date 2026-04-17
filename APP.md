# App Interface Guide

## Overview

Three GCScript transactions are implemented. The interface is responsible for computing and injecting `args` into each script before dispatching to the GameChanger wallet.

---

## `open_limit_order.json`

All values are direct user input. Nothing needs to be pre-computed — beacon hashes and the datum are computed inside the GCScript.

| Arg | Source |
|---|---|
| `offer-policy-id` | asset picker |
| `offer-asset-name` | asset picker (hex-encoded) |
| `ask-policy-id` | asset picker |
| `ask-asset-name` | asset picker (hex-encoded) |
| `price-numerator` | user input |
| `price-denominator` | user input |
| `offer-quantity` | user input |

---

## `swap_limit_order.json`

Two values require off-chain computation. Everything else is read from the selected UTxO's inline datum.

| Arg | Source |
|---|---|
| `swap-tx-hash` | query UTxOs by pair beacon |
| `swap-tx-index` | query UTxOs by pair beacon |
| `owner-stake-keyhash` | decoded from inline datum |
| `offer-policy-id` | decoded from inline datum |
| `offer-asset-name` | decoded from inline datum |
| `ask-policy-id` | decoded from inline datum |
| `ask-asset-name` | decoded from inline datum |
| `price-numerator` | decoded from inline datum |
| `price-denominator` | decoded from inline datum |
| `remaining-offer-quantity` | **computed**: `current_offer - offer_taken` |
| `output-ada-quantity` | **computed**: `utxo_lovelace + floor(offer_taken × price-numerator / price-denominator)` |

---

## `close_limit_order.json`

All values come from the selected UTxO and its inline datum.

| Arg | Source |
|---|---|
| `swap-tx-hash` | query user's own swap UTxOs |
| `swap-tx-index` | query user's own swap UTxOs |
| `owner-stake-key-hash` | wallet (raw 28-byte key hash hex, not bech32) |
| `offer-policy-id` | decoded from inline datum |
| `offer-asset-name` | decoded from inline datum |
| `ask-policy-id` | decoded from inline datum |
| `ask-asset-name` | decoded from inline datum |

---

## What the Interface Must Implement

### 1. Beacon hash computation
The same SHA-256 formulas used inside the GCScripts, implemented in JS/TS for the query step. Required to know which pair beacon to search for when listing available swaps.

```
pairBeacon  = sha256(offerPolicyId || offerAssetName || 0x00 || askAssetName)
offerBeacon = sha256(0x01 || offerPolicyId || offerAssetName)
askBeacon   = sha256(0x02 || askPolicyId || askAssetName)
```

> ADA policy ID is substituted with a single `0x00` byte (not the string `"ada"`).

### 2. UTxO query
Fetch UTxOs holding the pair beacon to list available swaps for a given asset pair.

- Blockfrost: `GET /assets/{pairBeaconUnit}/addresses` → then `GET /addresses/{addr}/utxos`
- Or: `GET /addresses/{dappAddress}/utxos` filtered by pair beacon asset

### 3. Inline datum parser
Decode the CBOR-encoded `SwapDatum` from each UTxO to extract swap parameters.

`SwapDatum` fields (in order):
1. `beacon_id` — policy ID bytes
2. `pair_beacon` — bytes
3. `offer_id` — policy ID bytes
4. `offer_name` — asset name bytes
5. `offer_beacon` — bytes
6. `ask_id` — policy ID bytes
7. `ask_name` — asset name bytes
8. `ask_beacon` — bytes
9. `swap_price` — constructor 0 `{ numerator: int, denominator: int }`
10. `prev_input` — `Some(OutputReference)` or `None`
11. `expiration` — `Some(POSIXTime)` or `None`

Use a CBOR library (`cbor-x`, `@emurgo/cardano-serialization-lib`, or similar) to decode.

### 4. Price math (for swap execution)
Given a user-chosen `offer_taken` amount:

```
remaining-offer-quantity = current_offer_quantity - offer_taken
output-ada-quantity      = utxo_lovelace + floor(offer_taken × price-numerator / price-denominator)
```

`utxo_lovelace` is the current ADA in the swap UTxO, read from the UTxO query response.
