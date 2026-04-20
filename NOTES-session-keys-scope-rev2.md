# Yellow Network Session Keys — Scope Proposal & Spec Plan (rev2)

**Date:** 2026-04-20
**Supersedes:** `yellow-session-keys-scope-2026-04-20.md` (rev1)
**Reviewers incorporated:** three parallel rev1 reviews (Gemini, Codex, Opus) in `/home/claude/code/nitrolite/NOTES-session-keys-scope-rev1-{gemini,codex,opus}.md`
**Intended audience:** Yellow engineering, for an architectural discussion of scoped session keys in v1+.

---

## Changelog from rev1

Material corrections:
- **Channel session keys DO enforce asset + metadata-hash + latest-version + expiry today.** The rev1 claim that `channel_session_key_assets_v1` was unused and `VerifyChannelSessionKePermissionsV1` was unbound was factually wrong. In production, `channel_v1/handler.go:56-59` binds it to `tx.ValidateChannelSessionKeyForAsset`, which enforces all four. The rev1 baseline was understated.
- **Dropped the suggestion to reuse `actionGateway` / `GatedAction` for scope.** Wrong principal domain (owner-centric staking/rate-limits vs delegate-centric policy) and incomplete coverage (`close`/`rebalance` map to empty string and bypass the gateway).
- **Minor factual fixes:** type is `AppSessionKeyValidatorV1` (not `AppSessionKeySigValidatorV1`); `create_app_session` does not fund initial allocations (funding happens in `submit_deposit_state`); `GatedAction()` is in active production use at `submit_app_state.go:77`, not aspirational.

Material additions:
- Explicit **Current vs Proposed Enforcement Matrix** as the doc's entry point.
- **Per-signature intent enforcement** requirement (enforcement must happen inside `verifyQuorum`, not globally after it — quorums can mix wallet and session-key signatures with different policies).
- **`create_app_session` bootstrap problem** for application-only scoped keys.
- **Rebalance batch-binding** design — default-deny is necessary but not sufficient.
- **Max TTL policy** as an explicit invariant.
- **Schema versioning reality**: per-key `version` (supersession counter) is not a format-version column; v2 means a new ABI packer + migrations, not a flag flip.
- **Security honesty section** on what the proposed v2 does and does not cover.
- **Implementation plan** with DDL sketches, handler hook points, and SDK touchpoints.
- **Open-invariants section** — questions Yellow must answer in prose before anyone writes schema.

Revised claims:
- `allowed_channel_ops` as a 2-bit RPC flag is too coarse — replaced with "transition-type granularity within `submit_state`."
- `per_op_limits` is no longer labeled "trivially stateless" — it requires reordering allocation fetches ahead of `verifyQuorum`.
- The rev1 "80/20" claim is scoped honestly: v2 covers intent/method gating + per-op shaping + resource allowlists, not cumulative drain, lateral `operate` shuffles, or cross-session rebalance coupling.

---

## 0. Executive summary

Nitrolite already has **more scope enforcement than rev1 described**: channel keys enforce asset + metadata-hash + latest-version + expiry; app-session keys enforce resource (application / session) + expiry. The gaps are **action-level gating (intents/transitions), monetary shaping, counterparty restriction, app-template binding, and rotation/TTL hygiene.**

This rev2 proposes a three-pillar v2 schema (App Sessions / Channels / Limits), each with narrow stateless additions, and defers stateful cumulative limits to a phase 2 contingent on explicit "spent" semantics. It identifies five invariants Yellow must define in prose before schema work begins — cumulative semantics, rebalance batch binding, create-path resolution, max TTL, and empty-scope handling.

The v2 surface is intentionally small: five new policy dimensions, all allowlist-only, all enforced per-signature at verification time, riding one signing-version bump that should coordinate with the per-intent-quorums proposal.

---

## 1. Current vs Proposed Enforcement Matrix

Rows are policy dimensions; columns are per-domain status. ✓ = enforced today, ✗ = not enforced today, — = not applicable to this domain.

| Dimension | Channel session keys (today) | App-session keys (today) | Proposed v2 |
|---|---|---|---|
| Expiry | ✓ `expires_at > now()` in lookup | ✓ same | ✓ keep + **add max-TTL bound at provisioning** |
| Latest-version supersession | ✓ `MAX(version)` in lookup | ✓ same | ✓ keep + **document in-flight race** |
| Resource — asset allowlist | ✓ via `ValidateChannelSessionKeyForAsset` | — (app sessions are multi-asset by design) | ✓ keep |
| Resource — metadata-hash binding | ✓ precomputed Keccak(version, assets, expires_at) in lookup | — | ✓ keep |
| Resource — application / session allowlist | — | ✓ `GetAppSessionKeyOwner` SQL (app_session_ids OR application_ids) | ✓ keep + **fix create-path bootstrap** |
| Action — intent / transition gating | ✗ | ✗ | ✓ **ADD: bitmask, enforced per-signature** |
| Monetary — per-operation cap | ✗ | ✗ | ✓ **ADD: per-asset max (stateless-ish)** |
| Monetary — cumulative cap | ✗ (v0.5.x `allowances.used` precedent) | ✗ | ⏸ **phase 2, only after "spent" is defined** |
| Counterparty allowlist | — (counterparty is always the node) | ✗ | ✓ **ADD: enforced at create-time** |
| App-template allowlist (version hash) | — | ✗ (application_ids don't version-hash-bind) | ✓ **ADD: enforced at create-time** |
| Rebalance batch binding | — | ✗ (rebalance conservation is global-only) | ✓ **ADD: mechanism TBD — see §6.2** |
| Revocation | ✗ (only expiry + supersession) | ✗ (only expiry + supersession) | open question — see §11 |

Reading this table is the fastest way to align rev2 with Yellow: the **bolded cells in the proposed column are what this doc is asking them to sign off on**; the ⏸ cell is deliberately deferred.

---

## 2. Architecture baseline (corrected)

### 2.1 Two-domain model

Nitrolite has **two parallel session-key systems** with distinct tables, distinct signer-type bytes, and distinct verification plumbing. Storage is disjoint; but note: the app-session deposit path reuses channel signature validation for the underlying channel state transition, so "fully disjoint code" is overstated (rev1 error).

| | App session keys | Channel session keys |
|---|---|---|
| State table | `app_session_key_states_v1` | `channel_session_key_states_v1` |
| Junction tables | `app_session_key_applications_v1`, `app_session_key_app_sessions_v1` | `channel_session_key_assets_v1` |
| Signer-type byte (in sig prefix) | `0xA1` wallet / `0xA2` session-key | `0x00` wallet / `0x01` session-key |
| ABI packer | `PackAppSessionKeyStateV1` | direct-packed `(version, assets, expires_at)` with precomputed `metadata_hash` |
| Main validator type | `AppSessionKeyValidatorV1` (constructor: `NewAppSessionKeySigValidatorV1`) | `ChannelSigValidator` |
| Consumed by | `app_session_v1/*.go` handlers | `channel_v1/*.go` handlers + `app_session_v1/submit_deposit_state.go` |

### 2.2 Channel session keys — current enforcement (CORRECTED)

Contrary to rev1's claim, channel session keys are **meaningfully scoped today**. On every channel-domain signature verification (`ChannelSigValidator.Recover` at `pkg/core/channel_signer.go:159-216`, called from `submit_state.go:120` and `request_creation.go:157`):

1. Signer type byte decoded.
2. Session-key authorization signature verified against master wallet (recovers from `(sessionKey, metadataHash)`).
3. Session key's own signature verified against the operation data.
4. **Permission callback invoked** at line 202 — `verifyPermissions(walletAddr, sessionKeyAddr, metadataHashStr)`.
5. Callback is bound in `channel_v1/handler.go:56-59` to `tx.ValidateChannelSessionKeyForAsset`, which runs the SQL at `channel_session_key_state.go:144-173`:
   - asset ∈ junction-table allowlist for this key
   - metadata_hash matches stored
   - version = `MAX(version)` for this `(user, sessionKey)` pair
   - `expires_at > now()`

Any failure → signature rejected. **This is production-grade resource scoping.** The scope gap for channel keys is **action granularity**, not asset enforcement.

### 2.3 App-session keys — current enforcement

On every app-session quorum signature (`verifyQuorum` at `handler.go:64-113`):

1. For each signature, decode signer-type byte.
2. `AppSessionKeyValidatorV1.Recover(data, sigBytes)` recovers the signer.
3. **If session-key signer:** `tx.GetAppSessionKeyOwner(sessionKeyAddr, appSessionId)` runs the SQL at `app_session_key_state.go:192-219`:
   - `session_key` matches
   - `version = MAX(version)` for this key
   - `expires_at > now()`
   - `(app_session_id ∈ junction OR application_id ∈ junction)` where `application_id` is resolved from the app-session row

Any failure → returns error, quorum not achieved.

Gap: once a key is authorized for an app session, it can sign **any intent** (operate / withdraw / close) for that session. No action gating.

### 2.4 Code map

Navigation, updated from rev1:

**Data models**
- `clearnode/store/database/app_session_key_state.go:14-48` — app-session key schema + junction tables
- `clearnode/store/database/channel_session_key_state.go:12-48` — channel key schema + asset junction

**Verification hotspots (the scope-enforcement surface)**
- `clearnode/api/app_session_v1/handler.go:64-113` — `verifyQuorum` (app-session enforcement)
- `clearnode/store/database/app_session_key_state.go:192-219` — `GetAppSessionKeyOwner` SQL
- `pkg/core/channel_signer.go:159-216` — `ChannelSigValidator.Recover` (channel enforcement)
- `clearnode/api/channel_v1/handler.go:56-59` — **where the channel permission callback is bound to storage** (this is the line rev1 missed)
- `clearnode/store/database/channel_session_key_state.go:144-173` — `ValidateChannelSessionKeyForAsset` SQL

**Provisioning**
- `clearnode/api/app_session_v1/submit_session_key_state.go:17-140`
- `clearnode/api/channel_v1/submit_session_key_state.go:12-104`
- `pkg/core/session_key.go:119-150` — `ValidateChannelSessionKeyAuthSigV1` (verifies master-wallet delegation)

**Packers / validators**
- `pkg/app/session_key_v1.go:166-189` — app-session signer-type byte scheme (`0xA1`/`0xA2`)
- `pkg/core/channel_signer.go` — channel signer-type scheme (`0x00`/`0x01`)

**SDK**
- Go: `sdk/go/app_session.go` — `SignSessionKeyState`, `SubmitSessionKeyState`, `GetLastAppKeyStates`, `RebalanceAppSessions`
- TS: `sdk/ts/src/client.ts` — `submitSessionKeyState`, `signSessionKeyState`, `getLastKeyStates`, `rebalanceAppSessions`

**Related proposal**
- `per-intent-quorums-proposal.md` at repo root — active design work that touches the same verification plumbing

---

## 3. Complete enumeration of signable actions

No change from rev1 (verified correct by all three reviewers).

**Channel session keys — 2 RPC methods.** Verifier: `ChannelSigValidator.Recover()`.

| # | RPC method | Handler | Semantics |
|---|---|---|---|
| 1 | `channels.v1.request_creation` | `channel_v1/request_creation.go:16` | Opens a new on-chain channel; initial state signed |
| 2 | `channels.v1.submit_state` | `channel_v1/submit_state.go:14` | Submits a new channel state. Covers many `TransitionType` values — transfer, resize, close, etc. — which is why "2-bit channel method gating" is too coarse for real scope (see §5.2) |

**App-session keys — 4 methods, 5 intents.** Verifier: `AppSessionKeyValidatorV1.Recover()` via `verifyQuorum`.

| # | RPC method | Intent | Handler | Semantics |
|---|---|---|---|---|
| 3 | `app_sessions.v1.create_app_session` | — | `create_app_session.go:17` | Creates a new app session; all participants quorum-sign. Initial allocations are **zero** (rev1 said "funds initial allocations" — wrong) |
| 4a | `app_sessions.v1.submit_app_state` | `operate` | `submit_app_state.go:113` | Redistributes allocations (totals per asset unchanged) |
| 4b | `app_sessions.v1.submit_app_state` | `withdraw` | `submit_app_state.go:119` | Moves funds back to participants' channels |
| 4c | `app_sessions.v1.submit_app_state` | `close` | `submit_app_state.go:125` | Finalizes allocations, marks session closed |
| 5 | `app_sessions.v1.submit_deposit_state` | `deposit` | `submit_deposit_state.go:15` | Adds funds from a channel into an app session. Uses **both** validators (channel sig for underlying transition + app quorum sig) |
| 6 | `app_sessions.v1.rebalance_app_sessions` | `rebalance` | `rebalance_app_sessions.go:18` | Moves value atomically across ≥2 app sessions in one bundle (see §8) |

**Not signable by session keys:** `apps_v1.submit_app_version` (wallet-only); all `get_*` reads (no per-request signing); provisioning either kind of session key (master wallet only).

---

## 4. Legacy `scope` field — the v1 clean slate

No change from rev1 (verified correct).

`docs/legacy/SessionKeys.md:108, 123, 126` documents a `scope` field in v0.5.x `auth_request` with examples `"app.create"`, `"ledger.readonly"`. **It is documentation-only.** Grep `clearnode/` for `scope` returns zero hits in production Go. The field is not parsed, not stored, not enforced. The v1 API (`docs/api.yaml`) dropped it entirely.

Implication: no compat semantics to preserve. V2 scope can be designed from first principles.

---

## 5. Three-pillar v2 proposal

Framed around the three axes Yellow engineers will think in:

### 5.1 App Sessions pillar

**New policy fields on `AppSessionKeyStateV1`:**

- `allowed_intents` — bitmask over `AppStateUpdateIntent` (5 bits fit in a byte). Default-deny for any unset bit. Signed in the state commitment.
- `allowed_app_templates` — junction table `app_session_key_app_templates_v1 (session_key_state_id, template_hash)`. Empty = no template restriction; non-empty = allowlist of app version hashes from `apps_v1.submit_app_version`.
- `allowed_counterparties` — junction table `app_session_key_counterparties_v1 (session_key_state_id, wallet_address)`. Empty = unrestricted; non-empty = participants at `create_app_session` time must be a subset.

**Enforcement points:**

- **Intent check: inside `verifyQuorum`, per-signature.** This is a correctness requirement, not a style choice. A quorum can mix master-wallet and session-key signatures, potentially across multiple session keys with different policies. A global post-quorum check can't distinguish "which signer had which policy." The check must happen in the loop at `handler.go:86` after `Recover()` returns a wallet, branched on signer-type byte: if the signature was from a session key, load that key's policy and check the intent bit.
- **Template check:** `create_app_session.go:17` entry, after `appDef` is resolved. Stateless comparison against the junction table.
- **Counterparty check:** `create_app_session.go` entry, after participant list is resolved. Stateless subset check.

### 5.2 Channels pillar

**Baseline clarification:** channel keys already enforce asset + metadata + expiry + supersession. What's missing is action granularity.

**Proposed addition: `allowed_channel_ops` is NOT a 2-bit RPC-method flag.** `submit_state` covers multiple `TransitionType` values (transfer, resize, close, etc.) — an RPC-level flag doesn't deliver useful separation. Instead:

- Add `allowed_transition_types` to `ChannelSessionKeyStateV1`. A bitmask over the enumerated `TransitionType` values actually implemented in the channel state machine (`pkg/core/` — enumerate during impl).
- Enforcement: in `ChannelSigValidator.Recover` or immediately after, the decoded transition type is checked against the key's bitmask. Default-deny.
- `request_creation` is separate — it's its own RPC. Add a single `allow_channel_creation` boolean to the key state. Most session keys will set this to false.

### 5.3 Limits pillar

**Phase 1 (this proposal):**
- `per_op_limits` — junction table `session_key_per_op_limits_v1 (session_key_state_id, asset, max_amount)`. Enforced at each signed operation against the **delta** in the signed payload.
- Honest statefulness note: the current handler order fetches participant allocations **after** `verifyQuorum` (see `submit_app_state.go` — the `GetParticipantAllocations` call happens inside the per-intent branches starting line 113). Per-asset delta caps require either:
  - (a) moving the allocation fetch ahead of `verifyQuorum`, or
  - (b) computing the delta from the signed payload alone (sum of new allocations minus old, where old can be carried in the signed state itself, at the cost of payload size).
  Neither is disqualifying but the "trivially stateless" framing in rev1 was wrong.

**Phase 2 (deferred, contingent on §6.1):**
- `max_total` — cumulative per key.
- `max_per_session` — cumulative per key per app session.

These need a new usage-tracking table with updates inside every ledger transaction. **Hot-row contention is real.** Do not implement until the "spent" semantics question (§6.1) has a written answer.

### 5.4 What we are explicitly NOT proposing

Rev1's `denied_methods` / `denied_intents` twins are dead. Deny-lists in signed capabilities create conflict-resolution ambiguity and are fragile against protocol growth (a new intent or transition added later silently inherits "allowed" status). Default-deny + explicit allowlists only.

Rev1's `allowed_judge_sets` is dropped. Nitrolite has participant quorums with weights, not judges. The concept doesn't map.

Reuse of `actionGateway` / `GatedAction` as the scope integration point is dropped. Wrong principal (app owners vs delegate keys), wrong domain (economic rate-limits vs capability enforcement), incomplete coverage (`close`/`rebalance` return `""` and bypass the gateway).

---

## 6. Open invariants to define BEFORE schema work

These are questions Yellow must answer **in prose** before the first migration lands. Each has a concrete open decision with implications.

### 6.1 Cumulative "spent" semantics

What does `max_total` measure in a state-channel setting where allocations redistribute? Four candidates:

- **Gross outflow per asset:** every time the key's owner's allocation decreases for asset X, the diff adds to "spent." Symmetric inflows don't offset. Strict but conservative.
- **Net outflow per asset:** (initial allocation) − (current allocation), summed across all sessions touched by the key. More lenient; allows recovery-then-respend.
- **Per-counterparty gross:** cumulative diff routed to each counterparty, capped independently. Much more complex.
- **Gross aggregate value (oracle-priced):** sum of spends across assets in a common unit. Requires price oracle; probably not.

**Recommendation:** gross outflow per asset. Conservative, matches the v0.5.x `allowances.used` precedent, definable without oracles. Revisit if use cases require net.

### 6.2 Rebalance batch binding

Default-deny on `rebalance` (via intent bitmask) is necessary but not sufficient for session keys that **do** authorize rebalance. The problem: each session's participants sign only their own session's `PackAppStateUpdateV1`; conservation is a global property checked only by the handler at `rebalance_app_sessions.go:232`. A rebalance-capable key implicitly trusts the **other** session updates in the batch.

Three candidate mitigations:

- **(a) Full batch hash in every signer's payload.** Each `SignedAppStateUpdateV1` carries a hash over the entire batch (list of `(session_id, version)` pairs). Mismatched batch → signature fails. Strong binding, adds payload bytes, requires clients to construct the full batch before signing.
- **(b) Explicit session-set allowlist per key.** A rebalance-capable session key additionally stores an allowlist of peer session IDs it may participate in a rebalance with. Simpler but less flexible.
- **(c) Operator-only enforcement.** Rebalance is treated as operator-grade; user-owned keys may only authorize `rebalance` if the calling party is a known operator wallet. Operator-centric, limits end-user flexibility.

**Recommendation:** start with (b). It's the simplest addition, fits the junction-table pattern, and forces explicit trust scoping. Promote to (a) if flexible multi-party rebalance becomes a common use case.

### 6.3 Create-path resolution for application-only scoped keys

`GetAppSessionKeyOwner(sessionKey, appSessionId)` resolves the application_id via subquery on `app_sessions_v1`. At `create_app_session` time the row does not yet exist, so application-only-scoped keys (`app_session_ids=[]`, `application_ids=[A]`) can't authorize creation — the subquery returns no row.

Two resolutions:

- **Deterministic session ID pre-commit.** The session ID is derived deterministically from participants + nonce + app definition. The client computes it and includes it in the signed payload; the handler matches. Enforcement: on `create_app_session`, check the key's policy against the app definition's `application_id` directly (no subquery needed).
- **Additive session-ID arrays at provisioning.** Require creation-capable keys to list the specific future session ID(s) they'll authorize. More restrictive; kills the "auth any future session of app A" use case.

**Recommendation:** (a). The deterministic ID is already computable from the handler's inputs; match against `appDef.ApplicationID` directly.

### 6.4 Max TTL policy

`submit_session_key_state.go` rejects past `expires_at` but enforces no upper bound. Combined with no revocation RPC, a compromised key can have an arbitrarily long exposure window.

**Proposal:** config-level max TTL at provisioning (e.g., 30 days as a default, configurable by clearnode operators). Rejection at submit time, not at verify time — keeps the verify path hot and simple.

### 6.5 Supersession race / in-flight ops

When a new key version is submitted, `GetAppSessionKeyOwner` switches to `MAX(version)` immediately. In-flight operations signed by the old version fail at verify time. This is an availability concern, not a security one (the key owner's master wallet signed both versions).

**Recommendation:** accept as-is for rev2. Document the expected behavior so SDKs can retry with the new version. No schema change needed.

### 6.6 Empty-scope fail-closed

Today `app_session_ids=[]` + `application_ids=[]` stores successfully but the verify SQL returns zero rows — the key is effectively useless. Fail-open on misconfiguration is a foot-gun; fail-closed at provisioning is a safer default.

**Proposal:** at `submit_session_key_state`, reject states where **all** scope arrays are empty. Small additive check, prevents a common misconfiguration.

---

## 7. Schema & implementation plan

Enough detail to unblock an implementer, not so much that it's code.

### 7.1 New columns and junction tables

**`app_session_key_states_v1`** — add columns:
```
allowed_intents_bitmask  SMALLINT NOT NULL DEFAULT 0   -- bits: operate|deposit|withdraw|close|rebalance
allow_create             BOOLEAN  NOT NULL DEFAULT FALSE
```

**New junction tables:**
```
app_session_key_app_templates_v1 (session_key_state_id, template_hash)
app_session_key_counterparties_v1 (session_key_state_id, wallet_address)
session_key_per_op_limits_v1      (session_key_state_id, domain, asset, max_amount)
  -- domain: 'app_session' | 'channel' — same table serves both domains
```

**`channel_session_key_states_v1`** — add columns:
```
allowed_transition_types_bitmask  SMALLINT NOT NULL DEFAULT 0
allow_channel_creation            BOOLEAN  NOT NULL DEFAULT FALSE
```

(Per-op limits reuse `session_key_per_op_limits_v1` with `domain='channel'`.)

### 7.2 ABI packer versioning: V1 → V2

Adding any policy field to the signed state changes the ABI packer. This is a proper new signing version, **not** a flag flip — the v2 packer coexists with v1 during migration.

**Pack order (proposal for V2):**

App-session V2:
```
(user_address, session_key, version,
 application_ids[], app_session_ids[],
 allowed_intents_bitmask, allow_create,
 app_templates[], counterparties[],
 per_op_limits[(asset, max_amount)],
 expires_at)
```

Channel V2:
```
(user_address, session_key, version,
 assets[],
 allowed_transition_types_bitmask, allow_channel_creation,
 per_op_limits[(asset, max_amount)],
 expires_at)
```

Metadata-hash precomputation on channel side: recompute over the expanded tuple.

### 7.3 Enforcement hook points

App-session domain:
- **`verifyQuorum` at `handler.go:86`, per-signature:** after `Recover()`, if signer-type byte is `0xA2` (session key), load the key's state (`GetAppSessionKeyOwner` already returns the row via join — extend to return `allowed_intents_bitmask`), and check the current operation's intent against the bitmask. Reject the signature if the bit is clear. This is where the rev2 design deviates most from rev1.
- **`create_app_session.go:17` entry:** after participants and `appDef` are resolved, iterate over session-key signers in the quorum and check `allow_create`, `allowed_app_templates`, and `allowed_counterparties` (subset containment). Before `verifyQuorum`.
- **Per-op limits:** at each intent handler (`handleOperateIntent`, etc.), compute per-asset delta from the signed payload, check against `session_key_per_op_limits_v1`. Delta-from-payload avoids a pre-quorum DB read.

Channel domain:
- **`ChannelSigValidator.Recover`, extend `verifyPermissions`:** after the current `ValidateChannelSessionKeyForAsset` check, also pass the decoded transition type and check against the bitmask. Same DB hit — extend the SQL to return the bitmask alongside the boolean.
- **`request_creation.go:157`:** check `allow_channel_creation` before accepting the signature.

### 7.4 SDK changes

**Go:** `sdk/go/app_session.go` — extend `AppSessionKeyStateV1` struct and `SignSessionKeyState` to include new fields. Add builder helpers: `WithAllowedIntents(...)`, `WithAppTemplates(...)`, etc. Same pattern for channel keys.

**TS:** `sdk/ts/src/client.ts` — parallel additions. The dream-shape JSON in the user's original ask becomes the high-level input shape; the SDK translates to the ABI-packed state.

**Compat:** legacy `sdk/ts-compat` has no scope concept. No changes needed.

### 7.5 Migration approach

- V1 keys remain valid. No forced rotation.
- V2 keys are provisioned via a new method (`app_sessions.v2.submit_session_key_state` or a version flag on the existing method — Yellow's call).
- Enforcement: verify code tries V1 decoding first; if the signer-type byte or payload indicates V2, decode and apply V2 policy. Both paths coexist indefinitely; V1 deprecation is a separate decision.
- Coordinate the version bump with the per-intent-quorums proposal to avoid back-to-back packer changes.

---

## 8. Rebalance — what it does and why it matters for scope

### 8.1 Mechanics (unchanged from rev1)

`rebalance_app_sessions` moves value **atomically across ≥2 app sessions in a single signed bundle**, with a hard conservation invariant: **per-asset totals across all sessions in the batch must sum to zero**.

Flow (`rebalance_app_sessions.go`):
1. Array of `SignedAppStateUpdateV1`, each with `intent = rebalance` (line 56).
2. Duplicate-session rejection (line 63).
3. Each session's participants independently quorum-sign their own session's state change (line 142).
4. Per-participant, per-asset delta computation (lines 179-214).
5. **Conservation check** at line 232: per asset, sum of all changes = 0. Rejected otherwise.
6. Ledger entries written per participant (line 240); one aggregate transaction per session per asset routed through a synthetic `batchID` account (line 267).

### 8.2 Why it was missed in initial research

Three compounding reasons (from rev1, retained):

1. Not in `docs/legacy/*.md`. `grep -r rebalance docs/` returns only `api.yaml` and a mermaid diagram.
2. `docs/api.yaml` is OpenAPI-style, not tutorial. Method name doesn't surface in grep for "transfer between sessions" or "move funds between app sessions."
3. The canonical Go SDK example (`sdk/go/examples/app_sessions/lifecycle.go`) demonstrates the **old pattern**: withdraw-then-deposit across sessions. Rebalance has no example file of its own.

### 8.3 Scope implications

Rebalance is structurally different from operate/withdraw/close:

1. **Multi-session touch.** A scoped key's `allowed_app_session_ids` must apply to **all** sessions in the batch, not just one. §6.2 is the explicit response.
2. **Conservation ≠ counterparty trust.** Each session's quorum signs only its own update; a rebalance-capable key implicitly trusts peer sessions' participants. Without §6.2's binding, a malicious counterparty session can conserve globally while routing through unexpected participants.
3. **Per-op cap ambiguity.** A participant can have simultaneous positive deltas in one session and negative in another. `max_per_operation` needs an explicit definition (gross/net) — §6.1 covers this.

**Rev2 stance:** default-deny `rebalance` in the intent bitmask. It's operator-grade. Any session key that opts in must also satisfy §6.2's batch-binding requirement.

---

## 9. Coordination with `per-intent-quorums-proposal.md`

The per-intent-quorums proposal (at repo root) and this rev2 both:
- touch the same verification chokepoint (`verifyQuorum` call sites)
- require a new signing version / ABI packer
- hinge on the `AppStateUpdateIntent` enum as a first-class discriminator

**Complementary, not conflicting:**
- Per-intent quorums raise the co-signer bar for dangerous intents (protocol-level guard).
- Session-key scope caps which intents a delegated key may participate in (principal-level guard).
- Together they narrow both "who must agree" and "what a delegated key can do."

**Integration risk:** two packer-version bumps in close succession would force SDKs to juggle three schema versions (V1 / per-intent-V2 / session-scope-V2). **Strong recommendation: ship both changes in a single V2 packer bump.** This requires coordination between proposal authors on shared pack order and DB migration timing.

---

## 10. Security honesty — what this v2 covers and does not

Rev1's "80/20" claim was optimistic. Honest accounting:

**What v2 covers well:**
- Action-level misuse of a compromised or overreaching key (intent gating).
- Scope-creep as the protocol grows (allowlist + default-deny).
- Resource misuse (asset / template / counterparty allowlists).
- Per-transaction damage cap (per-op limits).
- TTL hygiene (max-TTL at provisioning).
- Creation-path misauthorization (bootstrap fix).

**What v2 does not cover:**
- **Cumulative drain via repeated legitimate operations.** 1000 × $50 per-op = $50k. Per-op is rate-shaped control, not a loss cap. Mitigated only by phase-2 cumulative limits (§6.1).
- **Lateral value movement inside `operate`.** Per-asset totals are conserved, but redistribution among participants (including the key owner's own allocation) is not capped without custom logic.
- **Cross-session rebalance coupling.** Mitigated partially by §6.2; complete mitigation requires option (a) full-batch-hash binding.
- **Revocation latency.** Without an explicit revocation RPC, the best you can do is new-version supersession. Between compromise detection and new-version submission, the key is usable.
- **Side-channel correlation.** V2 binds policy to keys, not to sessions. Observers correlating a session key across multiple app sessions still learn something.

Rev2 is a real step forward on capability confinement. It is not a complete solution to delegated-authority risk.

---

## 11. Questions for Yellow engineering

Reordered for the meeting flow:

### Facts to confirm
1. Confirm: channel session keys currently enforce asset + metadata + expiry + supersession. (Our read of the code; reviewers agree.)
2. Confirm: no revocation RPC exists in v1 — only supersession and expiry.
3. Confirm: the v0.5.x `scope` field was deliberately dropped in v1, not deferred.

### Design decisions Yellow owns
4. "What does `max_total` measure?" — see §6.1 candidates. Gross-per-asset is our recommendation.
5. Rebalance batch-binding mechanism — §6.2 options (a/b/c). We recommend (b) to start.
6. Create-path resolution — §6.3. We recommend deterministic session ID + direct `application_id` check.
7. Max-TTL default — §6.4. We suggest 30 days, configurable per clearnode.
8. Is `submit_state` one permission or many? — transition-type granularity is required for channel scope to be meaningful.
9. Empty-scope handling — reject at provisioning (fail-closed)?
10. One V2 packer bump or two? — coordinate with per-intent-quorums authors.

### Possible future features
11. Explicit revocation RPC — worth adding to V2, or keep supersession-only?
12. Phase-2 cumulative limits — is this on the roadmap, or deliberately out of scope for v1+?
13. Cross-domain scope (a key that covers both a channel and an app session it funds) — desirable, or keep domains disjoint?

---

## 12. Parking lot

Items not in the core proposal but worth raising:

- **Signer-type byte schemes differ** between app-session (`0xA1`/`0xA2`) and channel (`0x00`/`0x01`). No security issue — prevents cross-domain replay. Note it exists if cross-domain scope is ever proposed.
- **Key supersession race** (§6.5) — availability concern, document but don't fix in this proposal.
- **E2E tests beyond mocks** — the 11 rebalance tests use `MockSigValidator`. A live-clearnode smoke test for rebalance + scope combinations before shipping is recommended.
- **SDK local validation is not server enforcement** — SDK-side validation callbacks exist (e.g., in challenge flows) and are permissive by design. Server-side enforcement is the only source of truth. Worth flagging in SDK documentation to prevent misunderstanding.
