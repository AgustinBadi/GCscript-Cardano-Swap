# GCscript Improvement Suggestions

Suggestions for the GCscript DSL to improve developer experience.

---

## 1. Comment Support

**Problem:** GCscript is JSON-based and JSON does not support comments. There is currently no way to annotate individual fields or function calls with explanatory notes, making complex scripts hard to read and maintain.

**Suggestion:** Add a reserved `_comment` field (or `description` field) that is ignored at runtime but preserved in the JSON structure. This would allow developers to document their scripts inline without affecting execution.

**Example:**
```json
{
    "type": "buildTx",
    "_comment": "Builds the Open Limit Order transaction — mints 3 beacons and creates the swap UTxO",
    "tx": { ... }
}
```

Alternatively, guarantee that `description` is accepted on every function call (not just script blocks) and is always ignored at runtime.

---

## 2. Rename `type` to `func`

**Problem:** Using `"type"` as the key for function calls is semantically confusing. In most contexts, `type` implies a data type or schema identifier, not a function invocation. Developers coming from any programming background expect `type` to describe *what something is*, not *what it does*.

**Suggestion:** Rename `type` to `func` (or `call`) to make the intent explicit and intuitive.

**Current:**
```json
{
    "type": "buildTx",
    "tx": { ... }
}
```

**Proposed:**
```json
{
    "func": "buildTx",
    "tx": { ... }
}
```

This reads naturally as "call the `buildTx` function" rather than "the type of this thing is `buildTx`". A backwards-compatible transition could support both `type` and `func` during a deprecation period.

---

## 3. Raw Byte Hashing in ISL

**Problem:** ISL's `sha256()` function accepts UTF-8 strings as input. However, many Cardano protocols require hashing raw byte arrays — for example, the cardano-swaps protocol computes beacon token names as:

```
pair_beacon  = sha2_256(offer_id* ++ offer_name ++ ask_id* ++ ask_name)
offer_beacon = sha2_256(0x01 ++ offer_id ++ offer_name)
ask_beacon   = sha2_256(0x02 ++ ask_id ++ ask_name)
```

Where `offer_id*` and `ask_id*` substitute ADA's empty policy ID (`""`) with a single byte `0x00` — but only for the pair beacon computation, not in the datum fields themselves.

Hashing the hex string representation of these bytes (e.g. `sha256("deadbeef")`) produces a different result than hashing the actual bytes (`0xDE 0xAD 0xBE 0xEF`), making it impossible to correctly compute on-chain beacon names inside GCscript today.

**Suggestions (either or both):**

1. **Add a `sha256Hex()` ISL function** that accepts a hex-encoded byte string and hashes the underlying bytes:
   ```
   sha256Hex("deadbeef01cafe...") -> hash of the actual bytes
   ```

2. **Allow pre-computed values as `args`** — let the caller pass raw bytes or pre-computed digests as script arguments, so the hashing can be done externally and the result injected at runtime:
   ```json
   {
       "type": "script",
       "args": {
           "pairBeacon": "<pre-computed sha256 hex>",
           "offerBeacon": "<pre-computed sha256 hex>",
           "askBeacon": "<pre-computed sha256 hex>"
       },
       "run": { ... }
   }
   ```
   This pattern keeps the script clean while allowing external computation for operations ISL cannot express natively.
