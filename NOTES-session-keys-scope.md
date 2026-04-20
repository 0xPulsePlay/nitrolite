# Yellow Network Session Keys — Architecture Map & Scope Design Notes

**Date:** 2026-04-20
**Context:** Prep material for an architectural discussion with the Yellow engineering team about scoped session keys. Based on a read-through of the nitrolite repo (`/home/claude/code/nitrolite`) at commit `bbc4f559`.
**Reader:** Liam (or future-me) going into that conversation with Yellow engineers. Self-contained — no prior session needed.

---

## 1. What session keys are in nitrolite

Off-chain authentication primitive. A user's master wallet authorizes a delegated signing key (the "session key"), which then signs RPC requests to the clearnode on the user's behalf. The delegation is captured in a **signed state** stored in the clearnode DB, versioned, and expirable.

**There are two parallel, disjoint session key systems** — they share no code and no table:

| | App session keys | Channel session keys |
|---|---|---|
| Table | `app_session_key_states_v1` | `channel_session_key_states_v1` |
| Used by | App-session handlers | Channel handlers |
| Scope hooks | `application_ids` + `app_session_ids` (enforced) | `assets` array (stored but NOT checked) |
| Provision via | `app_sessions.v1.submit_session_key_state` | `channels.v1.submit_session_key_state` |

A key of one type **cannot** sign operations in the other domain. That's the only coarse compartmentalization that exists today.

---

## 2. Key code locations

### Data models
- `clearnode/store/database/app_session_key_state.go:14-48` — app session key schema + junction tables `app_session_key_applications_v1` and `app_session_key_app_sessions_v1`
- `clearnode/store/database/channel_session_key_state.go:12-48` — channel session key schema + `channel_session_key_assets_v1` junction

### Provisioning handlers
- `clearnode/api/app_session_v1/submit_session_key_state.go:17-140` — app key submission (version sequencing, expiry check, signature verify)
- `clearnode/api/channel_v1/submit_session_key_state.go:12-104` — channel key submission
- `pkg/core/session_key.go:119-150` — `ValidateChannelSessionKeyAuthSigV1` verifies the master wallet's delegation signature

### Verification paths (the scope enforcement surface)
- `clearnode/api/app_session_v1/handler.go:64-113` — `verifyQuorum()`. Decodes signer-type byte (`0xA1` wallet / `0xA2` session key), recovers wallet, accumulates quorum weight. **This is the canonical enforcement point for app-session actions.**
- `clearnode/store/database/app_session_key_state.go:192-219` — `GetAppSessionKeyOwner()`. The SQL that checks `(app_session_id match OR application_id match) AND expires_at > now()`. **The only place any real scoping is enforced today.**
- `pkg/core/channel_signer.go:159-216` — `ChannelSigValidator.Recover()`. Supports session-key signatures for channel operations. **Calls a permissions hook at line 202 that is never bound to real storage** — so channel session keys currently have no effective scope beyond expiry.
- `pkg/core/session_key.go:27` — `VerifyChannelSessionKePermissionsV1` callback type. Intended extension point, unused.

### SDK surface
- Go: `sdk/go/app_session.go` — `SignSessionKeyState`, `SubmitSessionKeyState`, `GetLastAppKeyStates`
- TS: `sdk/ts/src/client.ts` — `submitSessionKeyState`, `signSessionKeyState`, `getLastKeyStates`

### Docs
- V1 source of truth: `docs/api.yaml` (OpenAPI-style YAML)
- Legacy v0.5.x compat: `docs/legacy/SessionKeys.md`, `docs/legacy/API.md`, `docs/legacy/Clearnode.protocol.md`
- Proposal already in repo: `per-intent-quorums-proposal.md` (at repo root — someone has been thinking about extending the intent model)

---

## 3. The `scope` field in legacy docs — what it actually is

`docs/legacy/SessionKeys.md:108, 123` documents a `scope` field in `auth_request` with example values `"app.create"` and `"ledger.readonly"`. Direct quote from line 123:

> `scope` (optional): Permission scope (e.g., "app.create", "ledger.readonly"). **Note:** This feature is not yet implemented

And line 11:

> When authenticating with an already registered session key, you must still provide all parameters in the `auth_request`. However, the configuration values (`application`, `allowances`, `scope`, and `expires_at`) from the request will be ignored...

### What the code actually does

Grepping `clearnode/` Go code for `scope` returns **zero hits** in any production file. The field:
- Is **not parsed** (no struct field reads it from the request)
- Is **not stored** (no DB column)
- Is **not enforced** (no check calls it)
- Is **only in v0.5.x legacy docs** — the v1 `docs/api.yaml` dropped it entirely

It is documentation-only. Purely aspirational. The v1 redesign deliberately didn't carry it forward. Good news: there's no legacy scope semantics to preserve — Yellow can design v1 scoping from scratch without compat constraints.

---

## 4. Complete enumeration of actions a session key can sign

### Channel session keys — 2 signable RPC methods
Verifier: `ChannelSigValidator.Recover()` — bound at `channel_v1/handler.go:56`.

| # | RPC method | Handler | What it does |
|---|---|---|---|
| 1 | `channels.v1.request_creation` | `channel_v1/request_creation.go:16` | Opens a new on-chain channel; signs the initial state |
| 2 | `channels.v1.submit_state` | `channel_v1/submit_state.go:14` | Submits a new channel state. **All off-chain value movement goes through this** — "transfers" between users, deposits from wallet into app sessions, withdrawals back to wallet. There is no separate `transfer` RPC method |

### App session keys — 4 methods, with 5 intents
Verifier: `AppSessionKeySigValidatorV1.Recover()` — called via `Handler.verifyQuorum()` at `app_session_v1/handler.go:64`.

Intents defined at `pkg/app/app_session_v1.go:16-24`.

| # | RPC method | Intent | Handler | What it does |
|---|---|---|---|---|
| 3 | `app_sessions.v1.create_app_session` | — | `create_app_session.go:17` | Creates a new app session, funds initial allocations; all participants quorum-sign |
| 4a | `app_sessions.v1.submit_app_state` | `operate` | `submit_app_state.go:113` | Redistributes allocations (totals per asset unchanged) — the "play the game" path |
| 4b | `app_sessions.v1.submit_app_state` | `withdraw` | `submit_app_state.go:119` | Moves funds from the app session back to participants' channels |
| 4c | `app_sessions.v1.submit_app_state` | `close` | `submit_app_state.go:125` | Finalizes allocations, marks session closed |
| 5 | `app_sessions.v1.submit_deposit_state` | `deposit` | `submit_deposit_state.go:15` | Adds funds from a channel into an app session. Uniquely uses **both** validators — channel sig for the underlying transition + app-session quorum sig |
| 6 | `app_sessions.v1.rebalance_app_sessions` | `rebalance` | `rebalance_app_sessions.go:18` | Moves value atomically across ≥2 app sessions in one signed bundle (see §7) |

### Things session keys CANNOT sign
- `apps_v1.submit_app_version` (registering app definitions) — wallet-only, uses plain `sign.TypeEthereumMsg` validator at `submit_app_version.go:75`
- All `get_*` reads — no per-request signing, auth'd at WebSocket session level
- Provisioning either type of session key — master wallet only; session keys cannot authorize other session keys
- **Revocation** — there is no v1 revocation RPC. Legacy `revoke_session_key` exists only in the v0.5.x compat layer. In v1, invalidation happens by expiry or by being superseded by a new version for the same `(user, session_key)` pair

---

## 5. What's actually scoped today

| Scope dimension | App session keys | Channel session keys |
|---|---|---|
| Expiry | ✓ enforced (`expires_at > now()` in lookup) | ✓ enforced |
| Resource (which app/session) | ✓ **enforced** via `GetAppSessionKeyOwner` SQL — key authorizes op on session X iff `app_session_ids` contains X **OR** `application_ids` contains X's owning app | partial — asset list stored in `channel_session_key_assets_v1` but **not checked** at verify time |
| Action/intent (operate vs withdraw vs close) | ✗ none — once authorized for a session, can sign any intent | ✗ none |
| Monetary cap | ✗ none in v1 (precedent exists from v0.5.x `allowances.used`) | ✗ none |
| Counterparty | ✗ none | ✗ none (not applicable — counterparty is the clearnode node itself) |

Takeaway: app session keys have structural scoping, channel session keys effectively have only expiry. Neither has action-level scoping.

### How app-session scope actually behaves

The authorizing SQL at `app_session_key_state.go:207` uses **OR**:

```
(app_session_key_app_sessions_v1.app_session_id = ?
 OR app_session_key_applications_v1.application_id = ?)
```

So at provisioning time, the arrays give:

| `app_session_ids` | `application_ids` | Effective scope |
|---|---|---|
| `[X]` | `[]` | Session X only |
| `[X,Y,Z]` | `[]` | Those three sessions only |
| `[]` | `[A]` | Any session of app A (including future ones) |
| `[X]` | `[A]` | Session X + anything in app A |
| `[]` | `[]` | **Zero rows match — useless** |

---

## 6. The "dream scope" analysis

Starting shape Liam floated:

```
{
  "application": "pulseplay",
  "allowed_methods": ["create_app_session", "submit_app_state"],
  "denied_methods": ["transfer", "withdraw", "close_channel", "resize_channel"],
  "allowed_app_session_intents": ["OPERATE", "DEPOSIT"],
  "denied_app_session_intents": ["WITHDRAW", "CLOSE"],
  "assets": { "USDC": { "max_total": "10000", "max_per_session": "500", "max_per_operation": "50" } },
  "allowed_counterparties": ["0xPulseHub"],
  "allowed_judge_sets": [["0xJudge1","0xJudge2","0xJudge3"]],
  "allowed_app_templates": ["0xTemplateHash"],
  "expires_at": 1760000000
}
```

### Architectural constraints that shape what's feasible

1. Session key state is **ABI-encoded and user-signed** — every field becomes part of the signed commitment. Adding fields = new state version (they already have versioning, so this is fine — plan on a `v2` schema).
2. Existing pattern is **junction tables for repeated fields** (`app_session_key_applications_v1`, etc.). New multi-value scope fields should follow this, not inline JSON.
3. **Stateless vs stateful checks** are very different beasts. Stateless (compare payload to allowlist) is cheap and race-free. Stateful (running totals) requires extra DB writes on every use, introduces hot rows, needs transaction isolation.

### Field-by-field feasibility

**Tier 1 — trivial, already matches the grain:**
- `application` → already exists (keep it).
- `allowed_app_session_intents` → **cleanest addition.** `AppStateUpdateIntent` enum at `pkg/app/app_session_v1.go:16-24` is already first-class. The unused `GatedAction()` method at `:43-54` hints this was on someone's mind. Add a bitmask column to the state row (5 intents fit in a byte), check it in `submit_app_state.go:112` and `submit_deposit_state.go`.
- `allowed_methods` (channel side) → **rename required.** No "transfer" method exists — transfers are `channels.v1.submit_state` with a new allocation. `close_channel` / `resize_channel` are encoded inside `submit_state` payloads, not distinct methods. Real channel-side actions are `request_creation` and `submit_state`. Collapses to a 2-bit flag.
- `expires_at` → already exists.

**Tier 2 — feasible, needs design care:**
- `allowed_methods` (app-session side) → small enum over `{create, submit_state, submit_deposit, rebalance}`. Works.
- `max_per_operation` per asset → **stateless, cheap, safe.** Check against the delta in the signed payload at verify time. Fails closed, no race conditions. Push hardest for this.
- `allowed_counterparties` → feasible at `create_app_session.go:17` (participants are in the request). App-session-scoped only; doesn't apply to channel transfers (counterparty is the node). Edge case: if a participant joins later, you'd need to re-check.
- `allowed_app_templates` → likely feasible. App definitions have content-addressable IDs (see `apps_v1/submit_app_version.go`). Check at `create_app_session` that the app's `ApplicationID` is in the allowed list. Clean stateless check. Best shape for "I trust this specific audited game contract, nothing else."

**Tier 3 — doable but costly:**
- `max_total` (cumulative per-key) and `max_per_session` → stateful. Needs a new table tracking `(session_key_id, asset, session_id?, spent_amount)` with UPDATEs on every op inside the same transaction as the ledger write. v0.5.x `allowances.used` is precedent. Costs: hot-row contention, subtle definition of "spent" in a state-channel where allocations redistribute, reorg semantics. Ask Yellow: **"what exactly would `max_total` measure?"** — the answer reveals how strict their accounting layer is.

**Tier 4 — concept mismatch:**
- `allowed_judge_sets` → nitrolite has no judges. It has participant quorums with weights (`participantWeights`, `requiredQuorum` at `handler.go:64`). If Liam means "restrict which co-signers validate alongside me," that folds into `allowed_counterparties`. If he means external arbitration oracles, that's contract-level work in `contracts/`, not clearnode scope. Either drop or re-slot.

### The deny-list footgun

The dream shape pairs `allowed_methods`+`denied_methods` and `allowed_intents`+`denied_intents`. **Drop all deny lists.**
- Redundant with allowlists.
- Conflict ambiguity: what happens if something's in both? in neither? in the protocol's future but not in either list?
- Fragile signed commitments: `deny=[close]` means the key can do everything else the protocol adds in the future. This is the scope-creep vulnerability that bit the v0.5.x `"clearnode"` special-case key.
- Existing codebase has no deny-list patterns anywhere. Stay consistent.

### Realistic v2 pitch

Five fields, all stateless at verify time, all allowlist-only, all mapping onto existing code paths and table patterns:

```
allowed_intents:         bitmask of AppStateUpdateIntent
allowed_channel_ops:     bitmask of {request_creation, submit_state}
per_op_limits:           junction table (session_key_id, asset, max_amount)
allowed_app_templates:   junction table (session_key_id, template_hash)
allowed_counterparties:  junction table (session_key_id, address)
```

That's ~80% of the dream shape's security value at ~20% of the engineering cost. Stateful monetary limits (`max_total`, `max_per_session`) as phase 2 if usage patterns justify the hot-row cost.

---

## 7. Rebalance discovery

### What rebalance_app_sessions does

Moves value **atomically across ≥2 app sessions in a single signed bundle**, with a hard conservation invariant: **per-asset totals across all sessions in the batch must sum to zero**.

Mechanics (`clearnode/api/app_session_v1/rebalance_app_sessions.go`):
1. Array of `SignedAppStateUpdateV1`, each with `intent = rebalance` (line 56).
2. Rejects duplicate sessions in the same batch (line 63).
3. Each session's own participants independently quorum-sign their own session's state change (line 142).
4. Computes per-participant, per-asset deltas (line 179-214).
5. **Conservation check** at line 232: per asset, sum of all changes across every session in the batch = 0. Rejected otherwise.
6. Writes ledger entries per participant (line 240), records one aggregate transaction per session per asset routed through a synthetic `batchID` account (line 267).

Concrete use: 3 sessions A/B/C. Alice has funds in A, wants some in C. Bob has funds in B, wants to top up A. One rebalance call, all quorum-signed, atomic all-or-nothing. No close-then-recreate, no half-committed intermediate state.

### Why it exists

- **Atomicity** — avoids stranded funds from partially-completed withdraw→deposit sequences
- **Operator efficiency** — almost certainly the clearnode's tool for its own market-making liquidity management across game sessions (the `batchID` synthetic account pattern reinforces this)
- **Multi-party coordination** — tournament prize pool redistribution in one atomic commit

### Status: fully production

| Layer | Evidence |
|---|---|
| Handler | 344 lines of real logic in `rebalance_app_sessions.go` |
| Router | `rpc_router.go:120` — registered and reachable |
| Method constant | `rpc.AppSessionsV1RebalanceAppSessionsMethod` in `pkg/rpc/methods.go` |
| Tests | `rebalance_app_sessions_test.go` — 1,256 lines, 11 test functions |
| Spec | `docs/api.yaml:758-771` |
| Go SDK | `sdk/go/app_session.go:259` — `Client.RebalanceAppSessions(ctx, signedUpdates)` |
| TS SDK | `sdk/ts/src/client.ts:1538` — `client.rebalanceAppSessions(updates)` |

### Why it was missed (Claude and Codex didn't find it)

1. **Not in legacy docs.** `grep -r rebalance docs/` returns only `api.yaml` and a mermaid diagram. `docs/legacy/*.md` doesn't mention it. Anyone relying on the legacy/public markdown docs never learns it exists.
2. **`docs/api.yaml` is an OpenAPI-style YAML dump**, not tutorial material. The method description "Rebalance multiple application sessions atomically" doesn't match what you'd grep for ("transfer between sessions," "move funds between app sessions").
3. **The canonical Go SDK example actively demonstrates the OLD pattern.** `sdk/go/examples/app_sessions/lifecycle.go` walks through a two-session scenario using **withdraw then deposit**, not rebalance. The header comment describes steps 6-7 as withdraw + deposit across sessions. Anyone reading "how do I work with multiple app sessions?" learns the old way first. Rebalance has no example file of its own.

**Unverified caveat:** the 11 tests use `MockSigValidator` (`testing.go`). Before depending on rebalance in production, smoke-test it against a running clearnode. Evidence is strong but not end-to-end.

### Why rebalance matters for scope design

Three ways it's structurally different from the other intents:

1. **Touches multiple sessions at once.** A scoped key's `allowed_app_session_ids` must apply to **all** sessions in the batch, not just one. Getting this wrong lets a key scoped to session A participate in a batch that moves funds in B too.
2. **Conservation invariant means counterparty scope leaks.** A rebalance-capable key implicitly trusts other sessions' participants to sign their halves correctly. Malicious counterparty session can still conserve globally while routing through unexpected participants.
3. **Per-operation limits get ambiguous.** A participant can have simultaneous positive deltas in one session and negative in another. What counts as "amount spent" for `max_per_operation`? Gross outflow? Net? Needs explicit definition.

**Default-deny stance on `rebalance` in the allowed-intents bitmask is defensible** — it's an operator-grade tool, not game-play-grade. Most end-user session keys should probably not have it.

---

## 8. Questions to bring to Yellow engineering

1. **Why was `scope` dropped from v0.5.x → v1?** The field is in legacy docs but entirely absent from the v1 code and `docs/api.yaml`. Deliberate deferral, or never-prioritized?
2. **What was the design intent for `channel_session_key_assets_v1`?** The schema exists but `VerifyChannelSessionKePermissionsV1` is unbound. Intended to land and got shelved, or different intent?
3. **Is `rebalance` meant for end-user keys, or operator-only?** Its absence from the tutorial example suggests the latter. If so, scope defaults should reflect that.
4. **What does "spent" mean for `max_total`-style accounting?** Net outflow per asset? Gross? Counterparty-restricted? This definition is load-bearing for any monetary-cap scope feature.
5. **Is there appetite for a `v2` session key state schema?** Per-intent quorums are already being discussed (`per-intent-quorums-proposal.md`) — scope fields could ride that same version bump.
6. **Revocation** — v1 has no RPC for it, only implicit supersession by new versions. Is explicit revocation planned, or is expiry+supersede the permanent design?

---

## 9. Parking lot (not discussed yet, worth raising)

- **Action gate already exists.** `core.GatedAction` concept and `h.actionGateway.AllowAction()` call at `submit_app_state.go:77`. There's a per-action gating framework already in use for the app registry's `CreationApprovalNotRequired` flag. Scope enforcement might reuse this plumbing rather than invent a parallel check.
- **Per-intent quorums proposal** (`per-intent-quorums-proposal.md` at repo root) is actively being designed. Scope design should coordinate — both proposals touch the same state schema and the same verification paths.
- **Signer type byte** (`0xA1` wallet / `0xA2` session key) at `pkg/app/session_key_v1.go:166-189` is app-session-only. Channel signatures use a different prefix scheme (`0x01`). Asymmetry worth keeping in mind if a unified scope model is ever proposed.
