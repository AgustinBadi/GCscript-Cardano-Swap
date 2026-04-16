# GCScript Issues to Report to Developer

## Pre-computed Values Required (injected via `args`)

### 1. Beacon asset names — `pairBeacon`, `offerBeacon`, `askBeacon`

Computed externally as SHA-256 hashes of specific byte sequences defined by the cardano-swaps protocol:

- `offerBeacon` = `sha256(0x01)`
- `askBeacon` = `sha256(0x02 || ask_policy_id_bytes || ask_asset_name_bytes)`
- `pairBeacon` = `sha256(0x00 || offer_policy_id_bytes || offer_asset_name_bytes || ask_policy_id_bytes || ask_asset_name_bytes)`

**Root cause**: ISL's `sha256()` only accepts UTF-8 strings, not raw byte arrays. The ADA policy ID in particular encodes as the zero byte `0x00` (not the string `"ada"`), which cannot be expressed as a UTF-8 string argument.

### 🎉 SOLUTION:

```json
{
  "title": "Binary Concat",
  "type": "script",
  "args": {
    "ask-header": "0002",
    "ask-policy-id": "d491234d8b40b63aceab0d7329c6db111c3634d9e0d3c6166c66c13b",
    "ask-asset-name-utf8": "USDx"
  },
  "return":{
    "mode":"last"
  },
  "exportAs": "data",
  "run": {
    "askData": {
      "type": "macro",
      "run": {
        "header": "{hexToByteArray(get('args.ask-header'))}",
        "policyId": "{hexToByteArray(get('args.ask-policy-id'))}",
        "assetName": "{hexToByteArray(strToHex(get('args.ask-asset-name-utf8')))}"
      }
    },
    "askHash": {
      "type": "macro",
      "run": "{sha256(flatten( get('cache.askData.header') , get('cache.askData.policyId') , get('cache.askData.assetName') ))}"
    }
  }
}

```
### 2. Datum CBOR — `datumHex`

The full `SwapDatum` encoded as CBOR hex, pre-computed offline.

**Root cause**: The `plutusData.fromJSON` step cannot accept integer values sourced from `args` macros. `args` values are always strings, so `{"int": "{get('args.priceNumerator')}"}` expands to `{"int": "1"}` — a string where an integer is required — causing the step to hang. There is no ISL function to cast a string to an integer.

### 🎉 SOLUTION:

```json

{
    "type": "script",
    "description": "From BigInt/BigNum Strings into Plutus Data Integer conversion (typed/untyped?). IMPORTANT: Overflow may occurr depending on the browser native implementation of JSON.parse(), need more input on this and ultimately will derive in a new ISL function bigNumToInt().",
    "title": "Creates a one-way swap UTxO on the Cardano-Swaps protocol via GCscript.",
    "exportAs": "datum",
    "args": {
        "priceNumerator": "1",
        "priceDenominator": "1",
        "offerQuantity": "3000000"
    },
    "run": {
        "datum": {
            "type": "plutusData",
            "data": {
                "fromJSON": {
                    "schema": 1,
                    "obj": {
                        "constructor": 0,
                        "fields": [
                            {
                                "int": "{jsonToObj(get('args.priceNumerator'))}"
                            },
                            {
                                "int": "{jsonToObj(get('args.priceDenominator'))}"
                            },                                                       
                            {
                                "int": "{jsonToObj(get('args.offerQuantity'))}"
                            }                          
                        ]
                    }
                }
            }
        },
        "datum2": {
            "type": "plutusData",
            "data": {
                "fromJSON": {
                    "schema": 1,
                    "obj": {
                        "list": [
                            {
                                "int": "{jsonToObj(get('args.priceNumerator'))}"
                            },
                            {
                                "int": "{jsonToObj(get('args.priceDenominator'))}"
                            },                                                       
                            {
                                "int": "{jsonToObj(get('args.offerQuantity'))}"
                            }
                        ]
                    }
                }
            }
        }          
  }
}
```

---

## Encoding Problems

### 3. Script hash discrepancy — `scriptHashHex` vs on-chain policy ID

This is the most significant issue, worth filing explicitly.

**The problem**: GCScript's `plutusScript` step computes `scriptHashHex` differently from how the Cardano node (and Aiken) computes the policy ID from the same CBOR bytes.

| | Formula | Input | Result |
|---|---|---|---|
| Cardano node / Aiken | `blake2b_224(0x02 \|\| cbor_bytestring_header \|\| flat_bytes)` | `5911ed<flat>` | `c4d7d117...` |
| GCScript internal | `blake2b_224(0x02 \|\| flat_bytes)` | strips `5911ed`, hashes `flat` | `705e07...` |

When `mints[].policyId` was hardcoded to Aiken's hash `c4d7d117...`, GCScript threw `missing_script_source` at build time because its internal script registry stored the script under `705e07...`.

**The workaround**: Double-wrap the `scriptHex`. Instead of providing `5911ed<flat>` (single CBOR), provide `5911f05911ed<flat>` (outer CBOR wrapping the single-CBOR). GCScript then strips the outer wrapper, hashes `5911ed<flat>`, and arrives at `c4d7d117...` — matching both `mints[].policyId` and the on-chain policy ID the node derives.

**Questions for the developer**:
- Is double-wrapping the intended `scriptHex` format for `plutusScript`? Is there a documented format specification?
- Is `scriptHashHex` intended to be usable as a `policyId` in `mints`? If so, why does it not match the Cardano-standard hash when single-CBOR scripts are provided?

### SOLUTION:

```
Some normalization is being applied for CBOR wrapping UPLC, but yes, as many other tooling sometimes the serialization being used differs between tools.

Try first on DSL and give me more insights to check, but if all fails wrapp/unwrapp manually using https://cardananium.github.io/cquisitor/ that uses the same underlying serialization lib

```

### 4. `args` inaccessible inside nested `script` blocks

`get('args.x')` and `get('cache.args.x')` both fail inside a nested `script` block's `run`. Only top-level `run` steps can access `args` directly. Steps that need `args` values must be placed at the top level, not inside nested scripts.

**Questions for the developer**:
- Is this by design? Is there a supported way to access `args` from within a nested `script` block?


### 🎉 SOLUTION:

```json
{
    "type": "script",
    "title": "Declarative argument proxy for nested scopes: \"args\":\"{get('args')}\"",
    "description": "",
    "exportAs": "willBeBaz",
    "args":{
        "foo":"bar",
        "bar":"baz"
    },
    "run": {
        "subScript":{
            "type": "script",
            "title": "",
            "description": "",            
            "args":"{get('args')}",
            "run": {
                "Hello_World":{
                    "type":"macro",
                    "run":"{get('args.bar')}"
                }
            }
        }        
    }
}
```
