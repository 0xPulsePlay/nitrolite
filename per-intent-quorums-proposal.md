# Per-Intent Quorums for App Sessions — Walkthrough & Proposal

Nitrolite app sessions currently enforce a single session-wide `quorum` threshold for every state transition (create, deposit, operate, withdraw, close, rebalance). This document walks through how quorum flows through the system today, then proposes a surgical extension that lets each intent carry its own required quorum while preserving the existing design patterns and backwards compatibility.

---

## Part 1 — How quorum works today, file by file

### 1. Wire schema — what the client sends

`pkg/rpc/types.go:140-158` — JSON-serializable transport types. This is what a participant sends over WebSocket JSON-RPC:

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

The API YAML at `docs/api.yaml:165-172` is the canonical contract — exact shape: `participants: []app_participant`, `quorum: integer`, `nonce: string`. Everything else (SDK types, DB) follows from this.

### 2. Core domain model — what the server works with

`pkg/app/app_session_v1.go:117-142`:

```go
type AppSessionV1 struct {
    SessionID     string
    ApplicationID string
    Participants  []AppParticipantV1
    Quorum        uint8          // ← single session-wide threshold
    Nonce         uint64
    Status        AppSessionStatus
    Version       uint64
    // ...
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

Intents are defined in the same file, `app_session_v1.go:16-24`:

```go
const (
    AppStateUpdateIntentOperate AppStateUpdateIntent = iota
    AppStateUpdateIntentDeposit
    AppStateUpdateIntentWithdraw
    AppStateUpdateIntentClose
    AppStateUpdateIntentRebalance
)
```

### 3. Deterministic session ID — why Quorum is immutable

`pkg/app/app_session_v1.go:313-361` (`GenerateAppSessionIDV1`) + `:202-251` (`PackCreateAppSessionRequestV1`). Both ABI-pack `(application, participants, quorum, nonce)` and keccak the bytes.

**Critical:** the session ID IS a hash over Quorum and participants. If you change the fields included in the pack, every existing session ID changes — the field set is effectively part of the on-wire contract.

This also means the TypeScript side has to match byte-for-byte. `sdk/ts-compat/src/app-signing.ts:57-85` packs the same tuple on the client for signing. If you extend the definition, you extend both packers identically.

### 4. Transport → domain translation

`clearnode/api/app_session_v1/utils.go:16-36` (`unmapAppDefinitionV1`) — maps `rpc.AppDefinitionV1` → `app.AppDefinitionV1`, pulling `Quorum` straight through.

`utils.go:169-176` (`getParticipantWeights`) builds the `map[wallet]weight` used during signature validation:

```go
func getParticipantWeights(participants []app.AppParticipantV1) map[string]uint8 {
    weights := make(map[string]uint8, len(participants))
    for _, p := range participants {
        weights[strings.ToLower(p.WalletAddress)] = p.SignatureWeight
    }
    return weights
}
```

### 5. Creation — the quorum bounds check

`clearnode/api/app_session_v1/create_app_session.go:64-88`:

- Rejects `Quorum == 0` (line 64-67)
- Sums participant weights → `totalWeights`
- Rejects duplicate participants
- Rejects `Quorum > totalWeights` (line 84-87, "target quorum cannot be greater than total sum of weights")

Then `line 159` calls `h.verifyQuorum(...)` against the packed create-request hash, using the same single `appDef.Quorum`. Session is persisted at `line 164-175` with that single `Quorum` field.

### 6. Persistence

`clearnode/store/database/app_session.go:20` — single `quorum` column on `app_sessions_v1`, default 100.

`clearnode/store/database/app_session.go:35` — `signature_weight` on `app_session_participants_v1` (composite PK `(app_session_id, wallet_address)`).

Schema DDL: `clearnode/config/migrations/postgres/20251222000000_initial_schema.sql:111-135`.

### 7. Enforcement — the single chokepoint

`clearnode/api/app_session_v1/handler.go:64-113` — `verifyQuorum`. Recovers each signer, checks they're a participant, accumulates weights (dedup by signer), fails if `achievedQuorum < requiredQuorum`.

**It's called from exactly four places**, always with `appSession.Quorum` (or the pre-persist `appDef.Quorum` for create):

| Call site | File:line | Intent |
|---|---|---|
| Create | `create_app_session.go:159` | create |
| Submit (operate/withdraw/close) | `submit_app_state.go:102` | operate / withdraw / close |
| Submit deposit | `submit_deposit_state.go:167` | deposit |
| Rebalance | `rebalance_app_sessions.go` | rebalance |

The intent is already in scope at every call site — `submit_app_state.go:102` passes `appSession.Quorum` immediately after dispatching on `appStateUpd.Intent`. This is the surgical seam.

### 8. Client side

`sdk/go/app_session.go:132` — `CreateAppSession(ctx, definition, sessionData, quorumSigs, opts...)` — takes `app.AppDefinitionV1` with Quorum pre-set by caller.

`sdk/ts-compat/src/app-signing.ts:10-21` — TS signing helper with `signatureWeight: number` and `quorum: number`, packs identically to the Go `PackCreateAppSessionRequestV1`.

---

## Part 2 — Surgical proposal for per-intent quorums

**Goal:** keep weights unchanged (or allow differing), but let each of {create, deposit, operate, withdraw, close} require a different quorum threshold.

**Design principles observed in this codebase:**
1. Typed enums with stringer methods (`AppStateUpdateIntent`)
2. Immutable session definition baked into the session ID hash
3. One chokepoint function for verification (`verifyQuorum`)
4. Explicit `Pack...V1` functions for signature determinism
5. Optional fields handled via pointers or zero-value-as-default (see how `SessionData` is treated as empty = keep previous)
6. Backward compatibility through versioned types (`...V1` suffix)

### Proposed shape

Extend `AppDefinitionV1` with an optional per-intent quorum map. Zero/missing value falls back to the existing single `Quorum`, so **every existing session keeps working identically** and the same session ID hashes correctly when the map is absent.

### Step 1 — extend the domain model

`pkg/app/app_session_v1.go`:

```go
// Add next to AppSessionStatus consts
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
    IntentQuorums []IntentQuorumV1 // same optional semantics
}
```

Use a slice (not a `map[Intent]uint8`) because **ABI packing requires a deterministic ordering** and maps don't give you one. Canonicalize order by sorting by `Intent` value at pack time.

Add a helper on the session:

```go
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

### Step 2 — extend the packers (critical for session ID + signatures)

`pkg/app/app_session_v1.go`: update `PackCreateAppSessionRequestV1` and `GenerateAppSessionIDV1`. Add the `intentQuorums` tuple slice after `quorum`:

```go
intentQuorumType, _ := abi.NewType("tuple", "", []abi.ArgumentMarshaling{
    {Name: "intent", Type: "uint8"},
    {Name: "quorum", Type: "uint8"},
})
args := abi.Arguments{
    {Type: abi.Type{T: abi.StringTy}},                         // application
    {Type: abi.Type{T: abi.SliceTy, Elem: &participantType}},  // participants
    {Type: abi.Type{T: abi.UintTy, Size: 8}},                  // quorum
    {Type: abi.Type{T: abi.SliceTy, Elem: &intentQuorumType}}, // intentQuorums (NEW)
    {Type: abi.Type{T: abi.UintTy, Size: 64}},                 // nonce
    {Type: abi.Type{T: abi.StringTy}},                         // sessionData (create only)
}
```

**Sort `intentQuorums` by `Intent` before packing** (and reject duplicate intents at validation time) so the hash is deterministic.

**Backward compat:** empty slice packs as a zero-length array — but this still changes the hash vs. the old packer. Two options:

- **(A) Bump to V2** (`PackCreateAppSessionRequestV2`, `GenerateAppSessionIDV2`, new RPC method). Safest; no ID collision with existing sessions. Matches the `...V1` convention already present.
- **(B) Conditional pack:** if `len(IntentQuorums) == 0`, use the old packing function; otherwise use the new one. Ugly but avoids a new method namespace.

Recommendation: **(A)**. The repo already has a precedent for versioned packers and the V1 RPC methods are the source of truth (`docs/api.yaml`). Shipping this as `app_sessions.v2.*` methods or just keeping V1 methods but allowing the new optional field is a judgement call — see "Migration path" below.

### Step 3 — validation at creation

`clearnode/api/app_session_v1/create_app_session.go:69-88`. After the existing `totalWeights` loop, add:

```go
seenIntents := make(map[app.AppStateUpdateIntent]bool)
for _, iq := range appDef.IntentQuorums {
    if seenIntents[iq.Intent] {
        c.Fail(rpc.Errorf("duplicate intent quorum: %s", iq.Intent), "")
        return
    }
    seenIntents[iq.Intent] = true

    if iq.Quorum == 0 {
        c.Fail(rpc.Errorf("intent quorum must be greater than zero for intent %s", iq.Intent), "")
        return
    }
    if iq.Quorum > totalWeights {
        c.Fail(rpc.Errorf("intent quorum for %s (%d) cannot be greater than total weights (%d)",
            iq.Intent, iq.Quorum, totalWeights), "")
        return
    }
    // Optional: reject intents that don't make sense (e.g., rebalance is node-gated)
}
```

Persist `IntentQuorums` alongside the session (new DB table — see Step 5).

Note: create itself is still gated by `appDef.Quorum` at `line 159`. If you want a per-intent quorum for `create` too, either (a) reuse `appDef.Quorum` as "the create quorum" and treat the intent-specific values as exclusively for state-update intents, or (b) let `IntentQuorums[Create]` override it. Option (a) is simpler and matches the existing mental model: `Quorum` = baseline / create threshold.

### Step 4 — enforcement at every call site

Change `verifyQuorum`'s fourth argument from `uint8` → either `uint8` via caller lookup, or keep the signature and just have callers pass `appSession.QuorumFor(intent)`. Keep `verifyQuorum` dumb and pass the resolved threshold — it's already policy-free, don't push logic into it.

Call sites:

- `submit_app_state.go:102`:

  ```go
  requiredQuorum := appSession.QuorumFor(appStateUpd.Intent)
  if err := h.verifyQuorum(tx, ..., participantWeights, requiredQuorum, packedStateUpdate, reqPayload.QuorumSigs); err != nil {
      return err
  }
  ```

- `submit_deposit_state.go:167` — same pattern, intent is always `Deposit` here so `appSession.QuorumFor(app.AppStateUpdateIntentDeposit)`.

- `rebalance_app_sessions.go` — use `QuorumFor(AppStateUpdateIntentRebalance)`.

- `create_app_session.go:159` — continues to use `appDef.Quorum` (the create baseline).

That's it for enforcement. Four lines changed, policy fully externalized into the session definition.

### Step 5 — persistence

New table `app_session_intent_quorums_v1` (match existing naming — `app_session_participants_v1` is the template):

```sql
CREATE TABLE app_session_intent_quorums_v1 (
    app_session_id CHAR(66) NOT NULL,
    intent SMALLINT NOT NULL,
    quorum SMALLINT NOT NULL,
    PRIMARY KEY (app_session_id, intent),
    FOREIGN KEY (app_session_id) REFERENCES app_sessions_v1(id) ON DELETE CASCADE
);
```

New migration file at `clearnode/config/migrations/postgres/<timestamp>_add_intent_quorums.sql` (goose format, `-- +goose Up` / `-- +goose Down`).

Add the GORM type in `clearnode/store/database/app_session.go`:

```go
type AppSessionIntentQuorumV1 struct {
    AppSessionID string                   `gorm:"column:app_session_id;not null;primaryKey;priority:1"`
    Intent       app.AppStateUpdateIntent `gorm:"column:intent;not null;primaryKey;priority:2"`
    Quorum       uint8                    `gorm:"column:quorum;not null"`
}
func (AppSessionIntentQuorumV1) TableName() string { return "app_session_intent_quorums_v1" }
```

Add to `AppSessionV1` struct:

```go
IntentQuorums []AppSessionIntentQuorumV1 `gorm:"foreignKey:AppSessionID;references:ID"`
```

Update `CreateAppSession` (line 43-71) to populate it, `GetAppSession` to `Preload("IntentQuorums")`, and `databaseAppSessionToCore` in `clearnode/store/database/utils.go` to map it back.

### Step 6 — wire/transport

`pkg/rpc/types.go`:

```go
type IntentQuorumV1 struct {
    Intent app.AppStateUpdateIntent `json:"intent"`
    Quorum uint8                    `json:"quorum"`
}
type AppDefinitionV1 struct {
    Application   string             `json:"application_id"`
    Participants  []AppParticipantV1 `json:"participants"`
    Quorum        uint8              `json:"quorum"`
    Nonce         string             `json:"nonce"`
    IntentQuorums []IntentQuorumV1   `json:"intent_quorums,omitempty"` // NEW
}
```

`clearnode/api/app_session_v1/utils.go`:

- `unmapAppDefinitionV1` — copy over `IntentQuorums`, converting intent strings/ints.
- `mapAppSessionInfoV1` — surface them on `AppSessionInfoV1` so clients can read current config.

Update `docs/api.yaml:160-172` (the `app_definition` type) to document the new optional field.

### Step 7 — SDK / client

- `sdk/go/utils.go:458,476` — add `IntentQuorums` to mapping helpers.
- `sdk/go/app_session.go:132` — `CreateAppSession` takes `app.AppDefinitionV1` already; no signature change needed.
- `sdk/ts-compat/src/types.ts` + `app-signing.ts` — add `intentQuorums` to `RPCAppDefinition` and update the signing hash packer to match Go byte-for-byte. Sort by intent before packing (document this invariant in a comment — this is one of the rare places a code comment is justified because the determinism requirement is non-obvious).

### Step 8 — tests

Follow the existing pattern in `clearnode/api/app_session_v1/submit_app_state_test.go` and `create_app_session_test.go`: add cases for

- different thresholds per intent
- missing `IntentQuorums` (backcompat — falls through to `Quorum`)
- validation errors (duplicate intent, zero quorum, exceeds total weights)

Sample sessions in `app_session_test.go` (`SignatureWeight: 100`, `Quorum: 100`) remain valid because they don't set `IntentQuorums`.

---

## Migration path recommendation

Ship behind `...V1` first, extending the existing methods — the new field is strictly additive (`omitempty`, empty slice packs to empty array), so **old clients creating sessions without `intent_quorums` get byte-identical behavior** as long as you branch the packer: if `len(IntentQuorums) == 0`, skip the new argument slot entirely; otherwise include it.

If you don't like the branching packer (legitimately ugly), introduce `app_sessions.v2.*` RPC methods with the new shape and leave V1 alone. V2 becomes the only way to opt into per-intent quorums.

## The touch list — summary

| File | Change |
|---|---|
| `pkg/app/app_session_v1.go` | Add `IntentQuorumV1`, extend `AppDefinitionV1`/`AppSessionV1`, add `QuorumFor` helper, extend packers |
| `pkg/rpc/types.go` | Add `IntentQuorumV1`, extend `AppDefinitionV1` with `intent_quorums` |
| `clearnode/api/app_session_v1/create_app_session.go` | Validate `IntentQuorums` (duplicates, zero, vs totalWeights); persist |
| `clearnode/api/app_session_v1/submit_app_state.go` | `appSession.QuorumFor(intent)` at line 102 |
| `clearnode/api/app_session_v1/submit_deposit_state.go` | `appSession.QuorumFor(Deposit)` at line 167 |
| `clearnode/api/app_session_v1/rebalance_app_sessions.go` | `appSession.QuorumFor(Rebalance)` |
| `clearnode/api/app_session_v1/utils.go` | Map/unmap `IntentQuorums` in `unmapAppDefinitionV1` + `mapAppSessionInfoV1` |
| `clearnode/store/database/app_session.go` | Add `AppSessionIntentQuorumV1` GORM model + persist in `CreateAppSession` + `Preload` in reads |
| `clearnode/store/database/utils.go` | Map DB rows → `app.IntentQuorumV1` in `databaseAppSessionToCore` |
| `clearnode/config/migrations/postgres/<ts>_add_intent_quorums.sql` | New table |
| `sdk/go/utils.go` | Add field in the two mapping helpers |
| `sdk/ts-compat/src/types.ts` + `app-signing.ts` | Match Go types + update hash packer (**must be byte-identical**) |
| `docs/api.yaml` | Document new optional field on `app_definition` |
| Tests | `create_app_session_test.go`, `submit_app_state_test.go`, `submit_deposit_state_test.go`, `app_session_test.go` |

**Lines of actual logic changed at runtime: ~5 (all `.Quorum` → `.QuorumFor(intent)`).** The rest is plumbing, persistence, and validation — pure additive surface area that doesn't alter the behavior of any existing session.

**One gotcha to flag before you start:** the signing hash on the client side (`sdk/ts-compat/src/app-signing.ts`) **must** match the Go packer byte-for-byte including intent sort order — if TS sorts `[deposit, operate]` but Go sorts `[operate, deposit]` (or by stringified name vs. numeric intent value), every signature fails verification with "quorum not met" in the least helpful way possible. Write a cross-language test vector (a fixed input → expected hex hash) and assert against it from both Go and Jest.
