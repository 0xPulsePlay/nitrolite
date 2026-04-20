# Per-Intent Quorums for App Sessions — Rev2

**Status:** Design spec (buildable). Supersedes rev1.
**Scope:** Allow each app-session intent (create, deposit, operate, withdraw, close, rebalance) to enforce a different signature-weight quorum threshold, while preserving existing single-quorum sessions unchanged.

## Changelog vs rev1

- **Corrected factual error:** `PackCreateAppSessionRequestV1` (5 fields, incl. `sessionData`) and `GenerateAppSessionIDV1` (4 fields) pack *different* tuples. Rev1 conflated them.
- **Corrected factual error:** rebalance calls `verifyQuorum` with participant signatures identically to other intents. Rev1's "rebalance is node-gated" line was wrong.
- **Spec commitment:** single normative path is **V2 packers + V2 RPC methods**. Conditional V1 packing is dropped.
- **Determinism pinned:** sort by ascending numeric `uint8` intent value in both Go and TS. Never by stringified name.
- **Policy cryptographically bound:** `intent_quorums` is included in both the session-ID hash AND the create-signing hash, so policy cannot be mutated post-creation without breaking the session ID.
- **Hydration hardened:** missing `Preload("IntentQuorums")` returns an error rather than falling back to baseline quorum.
- **New defenses:** `requiredQuorum > 0` guard inside `verifyQuorum`; `totalWeights` widened from `uint8` to `uint16` to eliminate overflow.
- **Touch list expanded:** adds `sdk/ts/src/app/*`, `get_app_definition.go`, `clearnode/stress/*`, `sdk/mcp/`, examples, and golden test vectors.
- **New sections:** cross-language golden test vectors (blocking deliverable), rollout strategy, security/UX.
- **Prereq identified:** `sdk/ts-compat/src/app-signing.ts` `normalizeIntent` currently throws on rebalance and must be fixed.

---

## Part 1 — How quorum works today

### 1. Wire schema — what the client sends

`pkg/rpc/types.go:140-158`:

```go
type AppParticipantV1 struct {
    WalletAddress   string `json:"wallet_address"`
    SignatureWeight uint8  `json:"signature_weight"`
}
type AppDefinitionV1 struct {
    Application  string             `json:"application_id"`
    Participants []AppParticipantV1 `json:"participants"`
    Quorum       uint8              `json:"quorum"`
    Nonce        string             `json:"nonce"`
}
```

Canonical contract: `docs/api.yaml:165-172`.

### 2. Core domain model

`pkg/app/app_session_v1.go:117-128` (session), `:131-134` (participant), `:137-142` (definition):

```go
type AppSessionV1 struct {
    SessionID     string
    ApplicationID string
    Participants  []AppParticipantV1
    Quorum        uint8          // single session-wide threshold
    Nonce         uint64
    Status        AppSessionStatus
    Version       uint64
    SessionData   string
    CreatedAt     time.Time
    UpdatedAt     time.Time
}
type AppParticipantV1 struct {
    WalletAddress   string
    SignatureWeight uint8
}
type AppDefinitionV1 struct {
    ApplicationID string
    Participants  []AppParticipantV1
    Quorum        uint8
    Nonce         uint64
}
```

Intent enum, `app_session_v1.go:16-24`:

```go
const (
    AppStateUpdateIntentOperate   AppStateUpdateIntent = iota // 0
    AppStateUpdateIntentDeposit                                // 1
    AppStateUpdateIntentWithdraw                               // 2
    AppStateUpdateIntentClose                                  // 3
    AppStateUpdateIntentRebalance                              // 4
)
```

### 3. Two distinct ABI packers — session ID vs create-signing hash

These are **not the same tuple** (rev1 got this wrong):

**Session ID** — `pkg/app/app_session_v1.go:313-361` (`GenerateAppSessionIDV1`). Packs 4 fields:

```
(application, participants, quorum, nonce)
```

**Create signing hash** — `pkg/app/app_session_v1.go:200-251` (`PackCreateAppSessionRequestV1`). Packs 5 fields:

```
(application, participants, quorum, nonce, sessionData)
```

Both use Go's `go-ethereum/accounts/abi` package and `keccak256` the result. The distinction matters because any extension of the definition must update **both** packers consistently — and `sessionData` must remain in the signing hash but absent from the session ID.

The TypeScript SDK has independent implementations that must byte-match the Go output:

- `sdk/ts/src/app/packing.ts` — primary SDK packer
- `sdk/ts-compat/src/app-signing.ts:57-85` — compat-layer packer

### 4. Transport → domain translation

`clearnode/api/app_session_v1/utils.go:16-36` — `unmapAppDefinitionV1`.
`clearnode/api/app_session_v1/utils.go:169-176` — `getParticipantWeights` builds the `map[wallet]weight` used at verification time.

### 5. Creation — quorum bounds check

`clearnode/api/app_session_v1/create_app_session.go:64-88`:

- Rejects `Quorum == 0` (line 64-67).
- Sums participant weights into `var totalWeights uint8` (line 70).
- Rejects duplicate participants.
- Rejects `Quorum > totalWeights` (line 84-87).

**Known bug (fix as prereq):** `totalWeights uint8` overflows silently. Five participants at weight 60 each sum to 300 → wraps to 44. Widen to `uint16`; bounds-check that the final sum fits in `uint8` before comparing.

### 6. Persistence

`clearnode/store/database/app_session.go:14-40` (GORM models), `20251222000000_initial_schema.sql:111-135` (DDL). Single `quorum` column on `app_sessions_v1`, default 100. `signature_weight` on `app_session_participants_v1` (composite PK).

### 7. Enforcement — the single chokepoint

`clearnode/api/app_session_v1/handler.go:64-113` — `verifyQuorum`. Recovers each signer, checks participant membership, accumulates weights (dedup by signer), fails if `achievedQuorum < requiredQuorum`.

**No `requiredQuorum > 0` guard** — safe today only because callers validate at creation. Adding the guard inside `verifyQuorum` is defense in depth for rev2.

Four call sites, all pass `appSession.Quorum` (or `appDef.Quorum` at creation):

| Call site | File:line | Intent context |
|---|---|---|
| Create | `create_app_session.go:159` | create |
| Submit state | `submit_app_state.go:102` | operate / withdraw / close |
| Submit deposit | `submit_deposit_state.go:167` | deposit |
| Rebalance | `rebalance_app_sessions.go` | rebalance (participant-quorum-gated, identically to others — the `GatedAction()` returning `""` for rebalance is rate-limit layer only, not quorum policy) |

The intent is in scope at every call site. This is the surgical seam.

### 8. Client side

- `sdk/go/app_session.go:132` — `CreateAppSession(ctx, definition, sessionData, quorumSigs, opts...)`.
- `sdk/ts/src/app/packing.ts` + `sdk/ts/src/app/types.ts` + `sdk/ts/src/utils.ts` — primary TS SDK.
- `sdk/ts-compat/src/app-signing.ts:10-21` — compat-layer. **Existing bug:** `normalizeIntent` at line 37-51 throws on rebalance — a prereq fix.

---

## Part 2 — Proposal: per-intent quorums via V2 packers

### Design principles (observed in this codebase)

1. Typed enums with stringer methods (`AppStateUpdateIntent.String()`).
2. Immutable session definition baked into the session ID hash.
3. One policy-free chokepoint for verification (`verifyQuorum`); callers resolve the threshold.
4. Explicit `Pack...V1` functions with deterministic ABI encoding.
5. Versioned types (`...V1` suffix) for cross-version compatibility.
6. Reusable helpers (e.g., `getParticipantWeights`) — extend, don't duplicate.

### Shape

Extend the definition with a sorted slice of `(intent, quorum)` pairs. Empty slice = behaves as before. Ship as V2 packers and V2 RPC methods; V1 stays frozen.

### Step 1 — extend the domain model

`pkg/app/app_session_v1.go`:

```go
type IntentQuorumV1 struct {
    Intent AppStateUpdateIntent
    Quorum uint8
}

type AppDefinitionV1 struct {
    ApplicationID string
    Participants  []AppParticipantV1
    Quorum        uint8
    Nonce         uint64
    IntentQuorums []IntentQuorumV1 // optional; empty = use Quorum for all intents
}

type AppSessionV1 struct {
    // ... existing fields ...
    Quorum        uint8
    IntentQuorums []IntentQuorumV1
}

// QuorumFor returns the required quorum for a given intent, falling back
// to the session-wide Quorum if no per-intent override is set.
func (s *AppSessionV1) QuorumFor(intent AppStateUpdateIntent) uint8 {
    for _, iq := range s.IntentQuorums {
        if iq.Intent == intent {
            return iq.Quorum
        }
    }
    return s.Quorum
}
```

Slice not map: ABI packing is deterministic only with a fixed ordering, and Go maps have no defined iteration order.

### Step 2 — V2 packers (cryptographic commitment to policy)

Both new packers live alongside V1 without modifying it. V1 remains frozen for existing sessions.

`pkg/app/app_session_v1.go`:

```go
// GenerateAppSessionIDV2 packs 5 fields (adds IntentQuorums).
// Invariant: IntentQuorums MUST be sorted ascending by numeric uint8 intent value
// before packing. Callers must canonicalize; implementations must reject unsorted input.
func GenerateAppSessionIDV2(definition AppDefinitionV1) (string, error) { ... }

// PackCreateAppSessionRequestV2 packs 6 fields (adds IntentQuorums; retains sessionData).
func PackCreateAppSessionRequestV2(definition AppDefinitionV1, sessionData string) ([]byte, error) { ... }
```

ABI shape for `IntentQuorums` is a tuple slice:

```go
intentQuorumType, _ := abi.NewType("tuple", "", []abi.ArgumentMarshaling{
    {Name: "intent", Type: "uint8"},
    {Name: "quorum", Type: "uint8"},
})
```

**Sort order — this is the determinism contract:**

> `IntentQuorums` MUST be sorted in ascending order by the numeric value of `intent` (`uint8`), not by the stringified name. Both Go and TS implementations MUST enforce this ordering before hashing. A cross-language golden test vector (fixed input → expected hex keccak) freezes this invariant.

Cryptographic binding: including `IntentQuorums` in `GenerateAppSessionIDV2` means the policy is part of the session ID. A malicious node that later rewrites DB rows to lower `withdraw` quorum would break the session ID and fail validation at every participant.

### Step 3 — validation at creation

`clearnode/api/app_session_v1/create_app_session.go`. Full rule set for rev2:

```go
// Prereq: widen accumulator to prevent overflow
var totalWeights uint16
participantWeights := make(map[string]uint8)
for _, participant := range reqPayload.Definition.Participants {
    participantWallet := strings.ToLower(participant.WalletAddress)
    if _, exists := participantWeights[participantWallet]; exists {
        c.Fail(rpc.Errorf("duplicate participant address: %s", participant.WalletAddress), "")
        return
    }
    totalWeights += uint16(participant.SignatureWeight)
    participantWeights[participantWallet] = participant.SignatureWeight
}

// Baseline quorum validation (existing + overflow guard)
if reqPayload.Definition.Quorum == 0 {
    c.Fail(nil, "quorum must be greater than zero")
    return
}
if uint16(reqPayload.Definition.Quorum) > totalWeights {
    c.Fail(rpc.Errorf("quorum (%d) cannot exceed total weights (%d)",
        reqPayload.Definition.Quorum, totalWeights), "")
    return
}

// Per-intent quorum validation (new)
validIntents := map[app.AppStateUpdateIntent]bool{
    app.AppStateUpdateIntentOperate:   true,
    app.AppStateUpdateIntentDeposit:   true,
    app.AppStateUpdateIntentWithdraw:  true,
    app.AppStateUpdateIntentClose:     true,
    app.AppStateUpdateIntentRebalance: true,
}
seenIntents := make(map[app.AppStateUpdateIntent]bool)
for _, iq := range appDef.IntentQuorums {
    if !validIntents[iq.Intent] {
        c.Fail(rpc.Errorf("unknown intent: %d", iq.Intent), "")
        return
    }
    if seenIntents[iq.Intent] {
        c.Fail(rpc.Errorf("duplicate intent quorum: %s", iq.Intent), "")
        return
    }
    seenIntents[iq.Intent] = true
    if iq.Quorum == 0 {
        c.Fail(rpc.Errorf("intent quorum must be greater than zero for intent %s", iq.Intent), "")
        return
    }
    if uint16(iq.Quorum) > totalWeights {
        c.Fail(rpc.Errorf("intent quorum for %s (%d) exceeds total weights (%d)",
            iq.Intent, iq.Quorum, totalWeights), "")
        return
    }
}

// Server-side canonicalization: sort before packing & persistence
sort.Slice(appDef.IntentQuorums, func(i, j int) bool {
    return appDef.IntentQuorums[i].Intent < appDef.IntentQuorums[j].Intent
})
```

**`create` intent semantics:** `Quorum` is the create threshold AND the fallback for state-update intents without an override. `IntentQuorums` overrides state-update intents only; attempting to put `create` in `IntentQuorums` is rejected as an unknown intent (it isn't in the `AppStateUpdateIntent` enum).

### Step 4 — enforcement at every call site

`verifyQuorum` stays policy-free. Add one defensive guard at the top:

`clearnode/api/app_session_v1/handler.go:64-113`:

```go
func (h *Handler) verifyQuorum(..., requiredQuorum uint8, ...) error {
    if requiredQuorum == 0 {
        return rpc.Errorf("internal: requiredQuorum must be > 0")
    }
    // ... existing body ...
}
```

Call sites:

- `submit_app_state.go:102` → `appSession.QuorumFor(appStateUpd.Intent)`
- `submit_deposit_state.go:167` → `appSession.QuorumFor(app.AppStateUpdateIntentDeposit)`
- `rebalance_app_sessions.go` → `appSession.QuorumFor(app.AppStateUpdateIntentRebalance)`
- `create_app_session.go:159` → continues using `appDef.Quorum` (create baseline, unchanged)

### Step 5 — persistence with loud hydration failures

New migration `clearnode/config/migrations/postgres/<ts>_add_intent_quorums.sql`:

```sql
-- +goose Up
CREATE TABLE app_session_intent_quorums_v1 (
    app_session_id CHAR(66) NOT NULL,
    intent         SMALLINT NOT NULL,
    quorum         SMALLINT NOT NULL,
    PRIMARY KEY (app_session_id, intent),
    FOREIGN KEY (app_session_id) REFERENCES app_sessions_v1(id) ON DELETE CASCADE
);

ALTER TABLE app_sessions_v1
    ADD COLUMN has_intent_quorums BOOLEAN NOT NULL DEFAULT FALSE;

-- +goose Down
ALTER TABLE app_sessions_v1 DROP COLUMN has_intent_quorums;
DROP TABLE IF EXISTS app_session_intent_quorums_v1;
```

GORM model in `clearnode/store/database/app_session.go`:

```go
type AppSessionIntentQuorumV1 struct {
    AppSessionID string                   `gorm:"column:app_session_id;not null;primaryKey;priority:1"`
    Intent       app.AppStateUpdateIntent `gorm:"column:intent;not null;primaryKey;priority:2"`
    Quorum       uint8                    `gorm:"column:quorum;not null"`
}
func (AppSessionIntentQuorumV1) TableName() string { return "app_session_intent_quorums_v1" }

type AppSessionV1 struct {
    // ... existing ...
    HasIntentQuorums bool                         `gorm:"column:has_intent_quorums;not null;default:false"`
    IntentQuorums    []AppSessionIntentQuorumV1   `gorm:"foreignKey:AppSessionID;references:ID"`
}
```

**Hydration contract (security-critical):**

```go
// databaseAppSessionToCore returns an error if the DB has intent quorums
// but the caller forgot to Preload them. Silent fallback to baseline Quorum
// would be a security downgrade — fail loudly instead.
func databaseAppSessionToCore(dbSession *AppSessionV1) (*app.AppSessionV1, error) {
    if dbSession.HasIntentQuorums && len(dbSession.IntentQuorums) == 0 {
        return nil, fmt.Errorf(
            "session %s has intent quorums but none were preloaded — missing Preload()",
            dbSession.ID,
        )
    }
    // ... existing mapping ...
}
```

Every `GetAppSession` and `GetAppSessions` call site must add `.Preload("IntentQuorums")`. The `has_intent_quorums` flag turns every missed preload into an immediate error instead of a silent security downgrade.

`CreateAppSession` sets `HasIntentQuorums = len(session.IntentQuorums) > 0`.

### Step 6 — wire/transport

`pkg/rpc/types.go`:

```go
type IntentQuorumV1 struct {
    Intent app.AppStateUpdateIntent `json:"intent"` // numeric uint8, matches existing intent enum wire format
    Quorum uint8                    `json:"quorum"`
}
type AppDefinitionV1 struct {
    Application   string             `json:"application_id"`
    Participants  []AppParticipantV1 `json:"participants"`
    Quorum        uint8              `json:"quorum"`
    Nonce         string             `json:"nonce"`
    IntentQuorums []IntentQuorumV1   `json:"intent_quorums,omitempty"` // NEW; sorted asc by intent
}
```

Use the numeric `uint8` enum representation consistent with the existing `intent` field. The string/enum ambiguity in `docs/api.yaml` for `intent` is a separate pre-existing cleanup; rev2 uses numeric consistently to avoid compounding it.

`clearnode/api/app_session_v1/utils.go`:
- `unmapAppDefinitionV1` — copy over `IntentQuorums`, canonicalize sort.
- `mapAppSessionInfoV1` — emit current intent-quorum map so clients can read back.

`clearnode/api/app_session_v1/get_app_definition.go:38-43` — manual construction of `rpc.AppDefinitionV1` must include `IntentQuorums`. Without this, clients would be blind to configured per-intent quorums on read-back. **This was missing from rev1.**

### Step 7 — SDK & client

**Primary Go SDK** (`sdk/go/`):
- `sdk/go/utils.go:458,476` — extend mapping helpers.
- `sdk/go/app_session.go:132` — `CreateAppSession` signature unchanged (takes `app.AppDefinitionV1`).

**Primary TS SDK** (`sdk/ts/`) — **was missing from rev1 entirely:**
- `sdk/ts/src/app/packing.ts` — add `packCreateAppSessionRequestV2`, `generateAppSessionIDV2`. Must produce byte-identical output to Go `PackCreateAppSessionRequestV2` / `GenerateAppSessionIDV2` per the golden test vectors.
- `sdk/ts/src/app/types.ts` — add `IntentQuorumV1`, extend `AppDefinitionV1`.
- `sdk/ts/src/utils.ts` — extend `transformAppDefinitionToRPC` / `transformAppDefinitionFromRPC` to carry `intent_quorums`, sort on outbound.

**Compat SDK** (`sdk/ts-compat/`):
- `sdk/ts-compat/src/types.ts` — extend `RPCAppDefinition`, `AppSession`.
- `sdk/ts-compat/src/app-signing.ts` — extend packer + **fix existing `normalizeIntent` rebalance bug at lines 37-51**. This is a prereq for any rebalance-intent quorum support.

**MCP documentation surface:**
- `sdk/mcp/src/index.ts` — surfaces quorum documentation and examples to AI agents/IDEs. Update to describe `intent_quorums`.

**Examples:**
- `sdk/go/examples/app_sessions/lifecycle.go` — demonstrate per-intent quorums.
- `sdk/ts/examples/` — matching TS example.

**API docs:**
- `docs/api.yaml:160-172` — document `intent_quorums` on `app_definition`.

### Step 8 — cross-language golden test vectors (blocking deliverable)

Ship **before** any packer code changes:

```
/testdata/per-intent-quorums/
  vector-01-empty-intent-quorums.json          → expected_session_id_v2.hex
  vector-02-single-intent-quorum.json          → expected_session_id_v2.hex
  vector-03-all-intents.json                   → expected_session_id_v2.hex
  vector-04-unsorted-input-rejected.json       → expected_error
  vector-05-create-signing-hash-all-intents.json → expected_create_hash_v2.hex
  vector-06-v1-backcompat.json                 → expected_session_id_v1.hex (unchanged)
```

Each vector is asserted by:
- `pkg/app/app_session_v1_test.go` (Go)
- `sdk/ts/src/app/packing.test.ts` (primary TS, Jest)
- `sdk/ts-compat/src/app-signing.test.ts` (compat TS, Jest)

If any of the three assertions disagrees, the CI pipeline fails. This freezes the ABI before implementation code is written.

### Step 9 — rollout strategy

Partial-fleet mismatches on V2-aware clients vs V1-only servers are the main risk during deploy.

1. **Server first.** Deploy clearnode with V2 RPC methods before any SDK release emits `intent_quorums`.
2. **SDK capability gating.** New SDKs detect V2 support either via:
   - A server capabilities endpoint (preferred if one exists or can be added cheaply), or
   - Attempting `app_sessions.v2.create_app_session` and falling back to V1 on "method not found."
3. **Hard gate in SDKs.** If the caller passes a non-empty `intent_quorums` but the server is V1-only, the SDK MUST reject the call with a clear error — never silently strip the field and call V1 (that would create a session with weaker policy than the caller intended).
4. **No dual-write.** V2 sessions use V2 packers and V2 session IDs. Clients that want single-quorum sessions keep calling V1 methods — no change to their path.

### Step 10 — security & UX policy

Per-intent quorums are genuinely a governance decision, not neutral plumbing. Document this directly rather than hiding it in prose.

**The "rug" scenario:** a session created with `Quorum: 100` (all participants must sign) but `IntentQuorums: [{withdraw, 50}]` allows any coalition with ≥50 weight to drain funds. A naive user who thinks "100 means everyone has to agree" will not expect this.

Rev2 takes an explicit stance:

- **Asymmetric thresholds are allowed.** No hardcoded constraint like "withdraw ≥ operate." Different apps have legitimately different safety models (e.g., a game where rapid operates need fewer sigs but cashouts need unanimity).
- **Integrators own the policy.** Unsafe combinations are their responsibility.
- **Clients must surface the policy.** UI at session-join and withdraw time must display the per-intent quorum table. Hiding the map is a product footgun.
- **Readable on request.** `get_app_definition` and `get_app_sessions` both return the current `intent_quorums` map so auditors and wallets can check it.

## Critical files — touch list

| File | Change |
|---|---|
| `pkg/app/app_session_v1.go` | Add `IntentQuorumV1`, `QuorumFor`; add V2 packers alongside V1 |
| `pkg/rpc/types.go` | Add `IntentQuorumV1`; extend `AppDefinitionV1` |
| `clearnode/api/app_session_v1/handler.go` | Add `requiredQuorum > 0` guard in `verifyQuorum` |
| `clearnode/api/app_session_v1/create_app_session.go` | Widen overflow accumulator; full per-intent validation; canonicalize sort |
| `clearnode/api/app_session_v1/submit_app_state.go` | Call site → `QuorumFor(intent)` |
| `clearnode/api/app_session_v1/submit_deposit_state.go` | Call site → `QuorumFor(Deposit)` |
| `clearnode/api/app_session_v1/rebalance_app_sessions.go` | Call site → `QuorumFor(Rebalance)` |
| `clearnode/api/app_session_v1/utils.go` | Map/unmap `IntentQuorums`; surface on `AppSessionInfoV1` |
| `clearnode/api/app_session_v1/get_app_definition.go` | **NEW in rev2:** include `intent_quorums` in read-path response |
| `clearnode/store/database/app_session.go` | GORM model, `HasIntentQuorums` flag, preload discipline |
| `clearnode/store/database/utils.go` | Hydration with loud error on missed preload |
| `clearnode/config/migrations/postgres/<ts>_add_intent_quorums.sql` | New table + flag |
| `clearnode/store/database/test/postgres_integration_test.go` | Cleanup SQL for new table |
| `clearnode/stress/app_session.go` + `clearnode/stress/storm.go` | **NEW in rev2:** update to V2 packers |
| `sdk/go/utils.go` | Map helpers |
| `sdk/go/app_session.go` | No signature change; passes through `app.AppDefinitionV1` |
| `sdk/go/examples/app_sessions/lifecycle.go` | Example with per-intent quorums |
| `sdk/ts/src/app/packing.ts` | **NEW in rev2:** V2 packers, byte-aligned with Go |
| `sdk/ts/src/app/types.ts` | **NEW in rev2:** extend domain types |
| `sdk/ts/src/utils.ts` | **NEW in rev2:** RPC transforms |
| `sdk/ts/examples/` | **NEW in rev2:** TS example |
| `sdk/ts-compat/src/types.ts` | Extend RPC + domain types |
| `sdk/ts-compat/src/app-signing.ts` | Extend packer + **fix `normalizeIntent` rebalance bug** |
| `sdk/mcp/src/index.ts` | **NEW in rev2:** update AI/IDE doc surface |
| `docs/api.yaml` | Document `intent_quorums` |
| `pkg/app/app_session_v1_test.go` | Golden vectors (Go) |
| `sdk/ts/src/app/packing.test.ts` | Golden vectors (TS primary) |
| `sdk/ts-compat/src/app-signing.test.ts` | Golden vectors (TS compat) |
| `clearnode/api/app_session_v1/*_test.go` | Per-intent quorum enforcement + validation tests |
| `testdata/per-intent-quorums/*.json` | Cross-language fixtures |

**Honest scope:** ~30 files touched, determinism correctness is the dominant risk rather than code volume. Rev1's "~5 lines of runtime logic" framing is dropped.

## Reusable helpers (do not duplicate)

- `getParticipantWeights` — `clearnode/api/app_session_v1/utils.go:169-176`.
- `unmapAppDefinitionV1` / `mapAppSessionInfoV1` — extension points for transport mapping.
- `AppStateUpdateIntent.String()` — use for error messages only, never for sorting.
- `verifyQuorum` — stays policy-free; callers resolve the threshold.

## Verification

End-to-end test plan:

1. **Golden vectors pass in all three stacks** (Go, `sdk/ts`, `sdk/ts-compat`). Pre-implementation gate.
2. **Go unit tests** — V2 packers produce expected hashes; V1 packers unchanged (byte-equal to pre-change).
3. **Go handler tests** — each intent honors `QuorumFor(intent)`; fallback when map absent; all validation rejections (zero, duplicate, unknown, overflow).
4. **DB integration** — round-trip persistence; `HasIntentQuorums=true` with missing preload returns error.
5. **TS unit tests** — same golden vectors; `normalizeIntent` handles rebalance.
6. **Cross-stack end-to-end** — clearnode + Go SDK create V2 session, TS SDK submits withdraw update with per-intent quorum, signature verifies. "Quorum not met" indicates determinism drift.
7. **Rollout simulation** — new SDK with non-empty `intent_quorums` against V1-only server → rejected with explicit error, never silently downgraded.
8. **Regression** — every existing test using only `Quorum` continues to pass with V1 packers.

## Out of scope

- Mutable intent quorums post-creation (would require versioned participant weights and breaks the "session ID commits to policy" invariant).
- Per-participant veto rights or non-linear voting.
- Signer-type restrictions per intent.
- Smart-contract changes — ChannelHub has no quorum concept; this stays fully off-chain.
- Cleaning up the pre-existing `intent` enum string/uint8 ambiguity in `docs/api.yaml` (separate cleanup, not a rev2 dependency).
