# Per-Intent Quorums Proposal — Rev1 (Codex)

Date: 2026-04-20

## Verdict

**Feasible with changes. Not safe to implement as currently written.**

The core architectural seam is correct (`verifyQuorum` call sites are the right enforcement points), but the document currently contains several critical compatibility and specification issues that can break signatures/session IDs or cause unsafe rollout behavior.

## What Is Correct in the Proposal

- The core idea (resolve required quorum by intent at call sites, keep `verifyQuorum` generic) fits the current code structure.
- The identified enforcement chokepoints are accurate: create, submit state, submit deposit, rebalance.
- Deterministic ordering concerns for ABI packing are valid (slice + sort is the correct direction).
- The emphasis on Go/TS byte-for-byte parity is absolutely necessary.

## Critical Issues To Fix First

1. **Create hash vs session ID packing are not identical today**
   - The doc currently conflates these.
   - `PackCreateAppSessionRequestV1` includes `sessionData`.
   - `GenerateAppSessionIDV1` does not.
   - Any extension must preserve this split intentionally and update both functions consistently.
   - References: `pkg/app/app_session_v1.go`.

2. **Backward compatibility narrative is internally inconsistent**
   - One section says missing `intent_quorums` keeps hash behavior identical.
   - Another section correctly states adding an ABI arg (even empty array) changes hashes unless:
     - conditional packer branching is used, or
     - V2 methods/types are introduced.
   - The doc must make one normative strategy explicit, not optional prose.

3. **Touch list is incomplete**
   - Must update `sdk/ts/src/app/packing.ts` (primary TS SDK), not only `sdk/ts-compat`.
   - Must update `clearnode/api/app_session_v1/get_app_definition.go`, otherwise API consumers will not see `intent_quorums`.
   - Also re-check stress/example paths that construct app definitions and/or use packing helpers.

4. **Mixed-version rollout hazard is not addressed strongly enough**
   - New clients can send `intent_quorums` to old servers.
   - Old server payload translation can ignore unknown fields, causing silent client/server assumption drift.
   - This can lead to signature/session-ID mismatches or policy misunderstanding in production.
   - References: `pkg/rpc/message.go` and app session handlers.

5. **Weight summation overflow risk**
   - Current create validation sums `totalWeights` in `uint8`.
   - This can overflow and wrap, creating incorrect quorum checks.
   - Per-intent checks add more dependence on that value, increasing risk.
   - Reference: `clearnode/api/app_session_v1/create_app_session.go`.

6. **Security/governance impact of asymmetric intent quorums is understated**
   - Lower `withdraw`/`close` quorum is a major authorization policy decision, not just convenience.
   - The proposal should explicitly define allowed relationships (or clearly document no constraints and expected UX warnings).

## Important Warnings

- **API typing consistency:** `docs/api.yaml` describes app-state `intent` as string while Go wire types use enum values (`uint8`-backed). Avoid repeating this ambiguity in `intent_quorums`.
- **Validation completeness:** reject duplicate intents, zero quorum, quorum > total weight, and unknown intent values.
- **Data loading correctness:** ensure all enforcement read paths hydrate intent quorum rows; avoid fallback-to-baseline due to missing preload.
- **Deterministic canonicalization:** sort by numeric intent in both Go and TS before hashing; never sort by label/string.

## Recommended Implementation Contract

1. **Make a hard compatibility choice upfront**
   - Preferred: **V2 packers + V2 RPC methods** for clean semantics.
   - Alternative: conditional V1 branching only if you accept permanent complexity.

2. **If branching V1 is chosen, codify this invariant**
   - `len(intent_quorums) == 0` must produce byte-identical V1 hashes and IDs.
   - `len(intent_quorums) > 0` uses extended layout.
   - Freeze this with test vectors.

3. **Server-first rollout**
   - Deploy server support first.
   - Gate SDK emission of `intent_quorums` until server capability is confirmed.
   - Avoid partial-fleet ambiguity during rollout.

4. **Fix numeric safety before expanding quorum logic**
   - Change total weight accumulation/checking to a wider type (for example `uint16`/`int`) and bounds-check conversion to stored field sizes.

## Minimum Test Matrix Before Ship

- Cross-language vectors (Go + TS + ts-compat):
  - create hash with no `intent_quorums`
  - create hash with sorted `intent_quorums`
  - create hash with unsorted input (must canonicalize or reject deterministically)
  - session ID hash equivalents
- Backward compatibility vectors proving old behavior unchanged when no overrides are set.
- Validation tests:
  - duplicate intent entries
  - unknown intent value
  - zero quorum
  - quorum > total weight
- Enforcement tests:
  - operate/withdraw/deposit/rebalance each honoring `QuorumFor(intent)`
  - fallback to baseline quorum when override absent
- API shape tests:
  - `get_app_definition` and session info responses include `intent_quorums` when configured.

## Suggested Edits to the Original Proposal

- Correct the "how quorum works today" section to distinguish create-hash packing from session-ID packing.
- Replace ambiguous migration language with one explicit normative path.
- Expand touch list to include `sdk/ts/src/app/packing.ts` and `get_app_definition.go`.
- Add rollout strategy section (server-first, capability gating, mixed-version behavior).
- Add a security policy section describing whether asymmetric thresholds are allowed and why.

## Final Recommendation

Proceed only after tightening the spec into an implementation-safe contract. The architecture is viable, but the current draft has enough ambiguity that two competent teams could implement incompatible versions.
