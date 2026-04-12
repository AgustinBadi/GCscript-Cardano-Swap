# CLAUDE.md

## Project Structure

- `SPEC.md` — foundational context, scope, protocol mechanics, tooling, and specification. Read this first.
- `DocsBoiler/SPEC.md` — spec template (reference only, not active).
- `DocsBoiler/IMPL.md` — impl plan template (reference only, not active).

## Workflow

1. Read `SPEC.md` in full before doing anything.
2. If the spec is ambiguous or incomplete, ask — do not assume.
3. Do not write code that isn't covered by the spec.

## TODO

- [ ] Resolve UTxO filtering question with GCscript developer — can filtering by asset pair be done inside GCscript or must it be done externally?
- [ ] Compile spending validator and beacon minting policy to CBOR using Aiken from the cardano-swaps source
- [ ] Implement Create GCscript — open a one-way swap (mint 3 beacons, build datum, output to personal DApp address)
- [ ] Implement Query GCscript — fetch open limit orders by beacon policy ID, filter by asset pair
- [ ] Implement Execute GCscript — fill a limit order (Swap redeemer, prev_input in output datum, invalid-hereafter)
