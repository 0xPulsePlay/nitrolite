# Per-Intent Quorums Proposal — Review Notes

Independent triple-blind review of `per-intent-quorums-proposal.md`. Three parallel reviewers each examined the full codebase against every claim in the proposal. Findings are synthesized below.

---

## Overall Assessment

The core idea — swap `.Quorum` for `.QuorumFor(intent)` at 4 call sites — is genuinely surgical and feasible. The architecture analysis in Part 1 is largely correct. However, the proposal undersells total implementation complexity by roughly 3x and has gaps that would cause silent failures if the touch list were followed as-is.

**Not hallucinated. Feasible. But incomplete.**

---

## Confirmed Accurate

These claims were verified against the actual codebase by all three reviewers:

| Claim | Verdict |
|-------|---------|
| `verifyQuorum` called from exactly 4 places | **Correct** — create, submit_app_state, submit_deposit_state, rebalance |
| Struct shapes for `AppDefinitionV1`, `AppSessionV1`, `AppParticipantV1` | **Correct** (minor field omissions in the proposal's snippets — `SessionData`, `CreatedAt`, `UpdatedAt` left out) |
| Intent enum: 5 values, iota-ordered | **Correct** — operate=0, deposit=1, withdraw=2, close=3, rebalance=4 |
| Session ID = `keccak256(abi.encode(app, participants, quorum, nonce))` | **Correct** |
| DB schema: single `quorum` column on `app_sessions_v1`, `signature_weight` on participants table | **Correct** |
| Smart contracts don't need changing | **Correct** — `ChannelHub.sol` has no quorum/app-session concept; quorum is purely off-chain |
| Concurrent state updates already serialize via version check | **Correct** — `appStateUpd.Version == appSession.Version+1` inside transaction |

---

## Critical Issues

### 1. `sdk/ts/` is missing from the touch list entirely

The proposal only mentions `sdk/ts-compat/src/app-signing.ts`. But the **primary** TypeScript SDK has its own independent packing implementations:

- `sdk/ts/src/app/packing.ts` — `packCreateAppSessionRequestV1`, `generateAppSessionIDV1`, `packAppStateUpdateV1`
- `sdk/ts/src/app/types.ts` — `AppDefinitionV1`, `AppStateUpdateIntent` enum
- `sdk/ts/src/utils.ts` — `transformAppDefinitionToRPC` / `transformAppDefinitionFromRPC`

These must stay byte-aligned with Go. **Two TS codepaths need updating, not one.** Missing this would cause mainline SDK users to produce wrong hashes and fail signature verification silently.

### 2. `get_app_definition.go` is missing from the touch list

`clearnode/api/app_session_v1/get_app_definition.go:38-43` manually constructs `rpc.AppDefinitionV1`:

```go
definition = rpc.AppDefinitionV1{
    Application:  session.ApplicationID,
    Participants: participants,
    Quorum:       session.Quorum,
    Nonce:        strconv.FormatUint(session.Nonce, 10),
}
```

This handler would return definitions **without** `intent_quorums` even after the change, silently hiding the feature from clients reading back their session config.

### 3. Sort order for ABI packing is dangerously under-specified

The proposal says "sort by Intent" but does not pin whether that means:
- **Numeric** `uint8` value: operate=0, deposit=1, withdraw=2, close=3, rebalance=4
- **Lexicographic** string name: close, deposit, operate, rebalance, withdraw

These produce completely different orderings. If Go sorts numerically and TS sorts lexicographically, **every signature silently fails** with an unhelpful "quorum not met" error.

**Required fix:** The spec must explicitly state "sort ascending by numeric `AppStateUpdateIntent` value (`uint8`)". Both Go and TS implementations must use numeric sort. Write a cross-language golden test vector before any packing code is touched.

### 4. Hydration failure = silent security downgrade

`QuorumFor()` falls back to `s.Quorum` when `IntentQuorums` is empty. If `Preload("IntentQuorums")` is missing from ANY read path:

- **If base quorum < intent-specific quorum:** silent security downgrade (e.g., withdraw was meant to require 2-of-2 but falls back to 1-of-2 base)
- **If base quorum > intent-specific quorum:** liveness failure (operations that should succeed get rejected)

This isn't theoretical — the codebase has multiple `GetAppSession` call sites and mock stores in tests that would all need updating. Consider making hydration failure **loud** (return error if DB has intent quorum rows but they weren't loaded) rather than silently falling back.

### 5. `requiredQuorum == 0` is not guarded in `verifyQuorum` itself

Today it's indirectly safe because `create_app_session.go:64-67` rejects `Quorum == 0`. But `verifyQuorum` at `handler.go:107-110` only checks `achievedQuorum < requiredQuorum` — if `requiredQuorum` is 0, it passes with zero signatures.

Edge case: migration bug or manual SQL sets `Quorum: 0`, or `IntentQuorums` row has `quorum: 0` that slips past validation. Add a defensive `requiredQuorum > 0` assertion inside `verifyQuorum`.

---

## Misleading Claims

### "Session ID packing and create signing hash are the same tuple"

They are **not** the same. Part 1, Section 3 says both ABI-pack `(application, participants, quorum, nonce)`. In reality:

- `GenerateAppSessionIDV1` packs `(application, participants, quorum, nonce)` — 4 fields
- `PackCreateAppSessionRequestV1` packs `(application, participants, quorum, nonce, sessionData)` — **5 fields**

The proposal conflates them with "Both ABI-pack (application, participants, quorum, nonce) and keccak the bytes." This matters because extending the definition affects **two different packers** with different field counts.

### "Rebalance is node-gated"

Rebalance calls `verifyQuorum` with participant signatures identically to operate/deposit/withdraw. The action gateway's `GatedAction()` returns `""` for rebalance and close (meaning they skip rate-limiting), but that's the staking/rate-limit layer, not quorum policy. Rebalance is participant-quorum-gated like everything else.

### "~5 lines of actual logic changed at runtime"

Technically true for the `.Quorum` → `.QuorumFor(intent)` swaps. Deeply misleading about total effort. The real implementation touches:

- 2 Go ABI packers (`GenerateAppSessionIDV1`, `PackCreateAppSessionRequestV1`)
- 2 TS SDK packing files (`sdk/ts/`, `sdk/ts-compat/`)
- DB migration + GORM model + preloading in all read paths
- Wire types (`pkg/rpc/types.go`)
- `get_app_definition.go` read path
- `unmapAppDefinitionV1` + `mapAppSessionInfoV1` mapping
- Validation logic in `create_app_session.go`
- Cross-language golden test vectors
- Stress test tooling
- Documentation (api.yaml, MCP server, examples)

### Line number for `AppSessionV1` struct

Proposal says `pkg/app/app_session_v1.go:117-142` for `AppSessionV1`. In reality, `AppSessionV1` is lines 117-128. The range 117-142 spans all three structs (`AppSessionV1`, `AppParticipantV1`, `AppDefinitionV1`). Minor but sloppy for a document that positions line numbers as precise references.

---

## Additional Missing Files

Files that reference quorum-related types/functions but are absent from the proposal's touch list:

| File | Why it matters |
|------|---------------|
| `sdk/ts/src/app/packing.ts` | **Primary TS SDK** — independent ABI packing, must match Go byte-for-byte |
| `sdk/ts/src/app/types.ts` | TS domain types for `AppDefinitionV1` |
| `sdk/ts/src/utils.ts` | `transformAppDefinitionToRPC` / `transformAppDefinitionFromRPC` |
| `clearnode/api/app_session_v1/get_app_definition.go` | Read path returns definitions without intent quorums |
| `clearnode/stress/storm.go` + `app_session.go` | Use packing functions directly — would produce wrong hashes |
| `sdk/go/examples/app_sessions/lifecycle.go` | Example code becomes stale |
| `sdk/ts/examples/` | Example code references `quorum` field |
| `sdk/mcp/src/index.ts` | Embeds quorum documentation and examples (discoverability surface) |
| `docs/legacy/API.md` | Multiple paragraphs describe quorum as single-value |
| `clearnode/store/database/test/postgres_integration_test.go` | Cleanup SQL must include new table |
| `pkg/app/app_session_v1_test.go` | Existing packing tests need new vectors |

---

## Design Concerns Worth Addressing

### Backward compatibility strategy

The proposal offers two options: (A) bump to V2 packers, or (B) conditional pack (skip new field if empty). Assessment:

- **Option A (V2)** is clearly safer. The repo already uses `...V1` naming throughout. A clean `V2` packer avoids any ambiguity about when the new field is included.
- **Option B (conditional pack)** is a footgun. If Go treats "unset" as omitted struct field but TS treats it as a zero-length `tuple[]`, the ABI encoding differs and signatures verify on one side only. This fails silently.

**Recommendation: Option A. Do not ship conditional packing.**

### Per-intent quorums must be in the signed create material

If intent quorums affect authorization but are **not** included in both `PackCreateAppSessionRequestV1` and `GenerateAppSessionIDV1`, then:
- The policy is not bound to the session ID participants think they joined
- A malicious or buggy node could change intent quorum thresholds after creation
- The trust model shifts from "cryptographic agreement" to "trust the DB"

All quorum policy that affects verification **must** appear in the signed definition.

### `ts-compat` `normalizeIntent` is already incomplete

`sdk/ts-compat/src/app-signing.ts:37-51` — `normalizeIntent` handles operate, deposit, withdraw, and `'close'` (string literal), but **throws for rebalance**. This is an existing bug/limitation, not introduced by the proposal, but it means compat-layer support for per-intent quorums on rebalance would require fixing this first.

### The "create" intent ambiguity

The proposal correctly notes that create uses `appDef.Quorum` (not an `AppStateUpdateIntent`). But it should explicitly state:
- `Quorum` field = create threshold (and fallback for any intent without an override)
- `IntentQuorums` = overrides for state-update intents only
- `Quorum == 0` with `IntentQuorums` populated = **invalid** (must be rejected at creation)
- `IntentQuorums` entry with `Quorum == 0` = **invalid** (must be rejected at creation)

---

## Recommendations Before Implementation

1. **Update the touch list** — add `sdk/ts/`, `get_app_definition.go`, stress tests, examples, MCP docs
2. **Pin sort order in the spec** — "ascending by numeric `uint8` intent value"
3. **Write cross-language golden test vectors first** — fixed input → expected hex hash, asserted from both Go and Jest, before any packing code changes
4. **Choose Option A (V2 packers)** — do not ship conditional packing
5. **Add defensive guards** — `requiredQuorum > 0` in `verifyQuorum`; loud error on hydration failure
6. **Include intent quorums in signed material** — both session ID and create signing hash
7. **Fix `ts-compat` `normalizeIntent`** for rebalance before or alongside this feature

---

*Reviewed 2025-04-20. Three independent parallel reviewers against the live codebase at HEAD.*
