# App Interface Guide

## Overview

Three GCScript transactions are implemented. The interface is responsible for computing and injecting `args` into each script before dispatching to the GameChanger wallet.

---

## `open_limit_order.json`

All values are direct user input. Nothing needs to be pre-computed — beacon hashes and the datum are computed inside the GCScript.

| Arg | Source |
|---|---|
| `offerPolicyId` | asset picker |
| `offerAssetName` | asset picker (hex-encoded) |
| `askPolicyId` | asset picker |
| `askAssetName` | asset picker (hex-encoded) |
| `priceNumerator` | user input |
| `priceDenominator` | user input |
| `offerQuantity` | user input |

---

## `swap_limit_order.json`

Two values require off-chain computation. Everything else is read from the selected UTxO's inline datum.

| Arg | Source |
|---|---|
| `swapTxHash` | query UTxOs by pair beacon |
| `swapTxIndex` | query UTxOs by pair beacon |
| `ownerStakeKeyHash` | decoded from inline datum |
| `offerPolicyId` | decoded from inline datum |
| `offerAssetName` | decoded from inline datum |
| `askPolicyId` | decoded from inline datum |
| `askAssetName` | decoded from inline datum |
| `priceNumerator` | decoded from inline datum |
| `priceDenominator` | decoded from inline datum |
| `remainingOfferQuantity` | **computed**: `current_offer - offer_taken` |
| `outputAdaQuantity` | **computed**: `utxo_lovelace + (offer_taken × priceNumerator / priceDenominator)` |

---

## `close_limit_order.json`

All values come from the selected UTxO and its inline datum.

| Arg | Source |
|---|---|
| `swapTxHash` | query user's own swap UTxOs |
| `swapTxIndex` | query user's own swap UTxOs |
| `ownerStakeKeyHash` | wallet (raw 28-byte key hash hex, not bech32) |
| `offerPolicyId` | decoded from inline datum |
| `offerAssetName` | decoded from inline datum |
| `askPolicyId` | decoded from inline datum |
| `askAssetName` | decoded from inline datum |

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
remainingOfferQuantity = current_offer_quantity - offer_taken
outputAdaQuantity      = utxo_lovelace + floor(offer_taken × priceNumerator / priceDenominator)
```

`utxo_lovelace` is the current ADA in the swap UTxO, read from the UTxO query response.
