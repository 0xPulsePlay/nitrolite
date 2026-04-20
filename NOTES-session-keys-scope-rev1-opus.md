# Session Keys Scope Notes — Multi-Reviewer Audit

**Source document:** `NOTES-session-keys-scope.md`
**Date:** 2026-04-20
**Method:** Three independent parallel review agents examined the document against the actual codebase, each from a different angle: (1) code claim verification, (2) architectural feasibility, (3) security & edge cases. Findings consolidated below.

---

## Verdict: Is the plan sound or hallucinated?

**Mostly sound, but contains one critical factual error that undermines the threat model and several material inaccuracies.** The overall direction (allowlists, signed scope, default-deny rebalance, stateless-first checks) is architecturally valid. The "80/20" pitch is defensible as a product pitch but optimistic as a security claim. However, the document's baseline analysis of channel session keys is **wrong**, which means any scope design built on that baseline will be debating the wrong starting point.

---

## 1. CRITICAL: Factual errors (fix before sharing)

### 1a. Channel session key assets ARE enforced (all 3 reviewers agree)

The document claims (§5 table, §2 line 40, §4 implicit):
- Channel session key assets are "stored but NOT checked at verify time"
- `VerifyChannelSessionKePermissionsV1` callback is "never bound to real storage"
- Channel session keys have "effectively only expiry" as scope

**All three claims are wrong in the production clearnode.**

`channel_v1/handler.go:56-59` wires the callback to `tx.ValidateChannelSessionKeyForAsset`, which performs a JOIN on `channel_session_key_assets_v1` checking: asset match, metadata hash match, latest version, non-expired. The SQL lives at `channel_session_key_state.go:144-173`.

**Impact:** The document presents channel keys as far less scoped than they actually are. This means:
- The §5 threat assessment overstates channel key risk
- The §6 proposal for `allowed_channel_ops` is solving a different baseline than described
- Yellow engineers will immediately spot this and question the rest of the analysis

### 1b. Type name is wrong

Document says `AppSessionKeySigValidatorV1.Recover()`. The actual type is **`AppSessionKeyValidatorV1`**; the constructor is `NewAppSessionKeySigValidatorV1` which returns `*AppSessionKeyValidatorV1`.

### 1c. `create_app_session` does not "fund initial allocations"

§4 table says create "funds initial allocations." Code shows sessions are created with **0 allocations**; funding happens via `submit_deposit_state`. Minor but signals imprecision.

### 1d. `GatedAction()` is not "unused"

§9 parking lot says the method "hints this was on someone's mind" as if it's aspirational. In reality, `GatedAction()` is actively called from `submit_app_state.go:77` and feeds the action gateway's staking/rate-limit system. It is in production use.

---

## 2. Architectural feasibility assessment (per proposal element)

### Tier 1 proposals

| Proposal | Rating | Notes |
|----------|--------|-------|
| `allowed_intents` bitmask | **SOUND** | Intent is parsed before quorum verification. Clean insertion point exists inside or before `verifyQuorum`. Must also cover `submit_deposit_state` and `rebalance_app_sessions` (separate RPC entry points). |
| `expires_at` | **SOUND** | Already exists and enforced. |

### Tier 2 proposals

| Proposal | Rating | Notes |
|----------|--------|-------|
| `allowed_channel_ops` (2-bit) | **QUESTIONABLE** | Mechanically feasible, but `submit_state` covers many `TransitionType` values (transfer, resize, close, etc.). A 2-bit RPC-level flag is too coarse for "no resize/close" dreams unless transition-level checks are added later. Also: `request_creation` does NOT call `AllowAction`, while `submit_state` does — asymmetric. |
| `per_op_limits` per asset | **QUESTIONABLE** | Doc says "stateless, cheap, safe" but current allocations are fetched **after** `verifyQuorum`, not before. Implementing per-asset delta caps requires reordering DB reads or adding new ones before the limit check. Not free. |
| `allowed_app_templates` | **QUESTIONABLE** | `ApplicationID` is available in the handler, but `GetAppSessionKeyOwner` uses a subquery on `app_sessions_v1` to resolve the `application_id` branch. For a **not-yet-created** session (at `create_app_session` time), that subquery returns no row. Application-only scoped keys may be unable to co-sign session creation. Needs explicit create-path handling. |
| `allowed_counterparties` | **QUESTIONABLE** | Participant addresses are available before `verifyQuorum`. Feasible but semantics are underspecified: restrict all participants? only co-signers? "other than me"? |

### ActionGateway as scope integration point

| Proposal | Rating | Notes |
|----------|--------|-------|
| Reuse `actionGateway.AllowAction()` for scope | **FLAWED** | ActionGateway implements **staking-based rate limits per app owner**, not delegation constraints from signed session-key state. Wrong policy domain, wrong principal. Also: `GatedAction()` returns `""` for `close` and `rebalance` intents, meaning those bypass action gateway entirely. |

### Schema versioning

| Claim | Rating | Notes |
|-------|--------|-------|
| "They already have versioning" for v2 schema | **QUESTIONABLE** | DB has per-key `version` (supersession counter), NOT a format/schema version column. Adding scope fields means a new ABI packer (`PackAppSessionKeyStateV1` → v2), new junction tables, and SDK changes. The doc is right in spirit but there's no "flip a schema version flag" — it's a proper new signing version. |

### The "80/20" claim

**Optimistic.** The 20% cost estimate is plausible (junction tables + bitmask checks + handler reordering). The 80% security claim is defensible only if "dream" is interpreted as "intent gating + coarse channel RPC gating + some per-op caps." It's weak against:
- Cumulative drain (1000 × per-op cap still empties a session)
- Lateral value movement within `operate` (shuffle among participants, zero net per asset)
- Rebalance batch-level coupling (cross-session economic trust)
- Coarse `submit_state` channel permission

---

## 3. Security findings

### Critical / High severity

| # | Finding | Severity | Doc coverage |
|---|---------|----------|-------------|
| S1 | **App session keys: any intent once scoped** — verified correct. A key authorized for session X can sign operate, withdraw, AND close. No intent filtering exists. | HIGH | Correctly identified |
| S2 | **`per_op_limits` without cumulative limits is a false sense of security** — 1000 operations at $50 each = $50k drain. Per-op is rate-shaped control, not a loss cap. | HIGH | Partially acknowledged (defers to "phase 2") |
| S3 | **Rebalance needs batch-level constraints, not just default-deny** — each session's quorum signs only its own `PackAppStateUpdateV1`; conservation is global. A rebalance-capable key creates economic coupling across sessions. Default-deny is necessary but not sufficient for strict keys. Needs explicit batch policy (hash of all session updates in every signer's payload, or allowlisted session sets). | HIGH | Identified but no mitigation proposed |
| S4 | **No max TTL on `expires_at`** — provisioning rejects past timestamps but has no upper bound. Users/apps can issue arbitrarily long-lived keys. Combined with no revocation RPC, exposure window after compromise can be very large. | MEDIUM | Partially noted (revocation question) but max TTL gap missed |

### Medium severity

| # | Finding | Severity | Doc coverage |
|---|---------|----------|-------------|
| S5 | **`create_app_session` + application-only scoped keys** — `GetAppSessionKeyOwner` subquery on `app_sessions_v1` for the `application_id` branch returns no row for a not-yet-created session. Application-only keys may fail to co-sign creation. | MEDIUM | Not identified |
| S6 | **Channel vs app key rotation asymmetry** — channel keys bind `metadata_hash` (new version with new hash immediately invalidates old sigs). App keys use junction-table scope (rotation behavior differs). Doc doesn't analyze this asymmetry for scope design implications. | MEDIUM | Not identified |
| S7 | **`close` and `rebalance` bypass `ActionGateway` rate limits** — `GatedAction()` returns `""` for these intents, so `AllowAction` returns nil (no-op). Inconsistent with registry story. | MEDIUM | Not identified |
| S8 | **Key supersession race** — what happens if a new session key version is submitted while operations signed by the old key are in flight? DB lookup uses `MAX(version)`, so in-flight ops from old key would fail. Not a vulnerability per se but an availability/UX concern. | MEDIUM | Not analyzed |

### Low / Informational

| # | Finding | Severity | Doc coverage |
|---|---------|----------|-------------|
| S9 | **`[]`/`[]` empty scope is useless, not dangerous** — storage allows zero junction rows but `GetAppSessionKeyOwner` fails the JOIN. Recommend rejecting empty scope at provisioning to fail closed. | LOW | Correctly identified as "useless" |
| S10 | **No classic signature replay** — state versions are monotonic; anti-replay is "signed payload includes version." Standard and sufficient for this model. | INFORMATIONAL | Not analyzed (not needed) |
| S11 | **Cross-domain confusion** — different signer type bytes (`0xA1`/`0xA2` vs `0x00`/`0x01`) and different ABI packers prevent cross-replay between app and channel domains. | LOW | Noted in §9 |
| S12 | **Bitmask bit-flipping** — not an issue since bitmask is inside ECDSA-signed commitment; any change breaks signature. Real concern is forward compatibility (new intent bits, default behavior for old keys). | INFORMATIONAL | Not analyzed |
| S13 | **Legacy compat layer** — `sdk/ts-compat` is a client façade, not a server-side bypass path. No scope circumvention risk from compat. | INFORMATIONAL | Not analyzed |

---

## 4. Coordination with `per-intent-quorums-proposal.md`

All three reviewers agree: **complementary, not conflicting**, but integration is non-trivial.

- **Complement:** Per-intent quorums raise the cosigner bar for dangerous intents; session-key scope bitmasks cap which intents a key may participate in. Together they narrow both "who must agree" and "what a delegated key can do."
- **Shared touchpoint:** Both proposals reason about intent at the same chokepoint (`verifyQuorum` call sites). Good for cohesion.
- **Tension:** Per-intent quorums change packing / session identity (V2 packers). Session key scope changes `PackAppSessionKeyStateV1` and DB schema. May ship two signing-version bumps in one release window. SDKs need clear separation of version axes.

---

## 5. Verified claims (things the doc gets right)

The bulk of the document is accurate. Across 20+ specific code citations verified:

- File paths and line numbers: ~95% accurate (off-by-one on file lengths, minor line shifts)
- SQL OR logic in `GetAppSessionKeyOwner`: correct
- Rebalance mechanics (conservation check, batch semantics, test count): correct
- RPC method enumeration (2 channel + 4 app session): correct
- Intent enumeration (5 intents): correct
- `scope` field grep returning zero hits in production Go: correct
- `submit_app_version` using wallet-only validation: correct
- SDK method locations (Go + TS): correct
- Signer type bytes (`0xA1`/`0xA2` for app, `0x00`/`0x01` for channel): correct
- Deny-list criticism: well-reasoned, no existing deny-list patterns in codebase
- Rebalance "default-deny" recommendation: sound
- "Design from scratch" assessment (no legacy scope to preserve): correct

---

## 6. Recommended next steps

### Before sharing with Yellow

1. **Fix the channel key error** — correct §2, §4, §5 to reflect that channel session keys enforce asset allowlist + metadata hash + version + expiry via `ValidateChannelSessionKeyForAsset`. Update the §5 table accordingly.
2. **Fix minor inaccuracies** — type name (`AppSessionKeyValidatorV1`), create_app_session funding claim, GatedAction usage status.
3. **Add channel enforcement detail to §6** — the `allowed_channel_ops` proposal should acknowledge the existing asset+metadata enforcement as baseline and propose transition-type granularity as the incremental improvement.

### Design gaps to resolve

4. **Define cumulative limit semantics** — "What does 'spent' mean?" is the right question. Define it before Yellow asks: gross outflow, net, per-participant? This is the difference between a useful feature and a foot-gun.
5. **Solve the create-path key resolution** — application-only scoped keys + `create_app_session` needs an explicit code path or documented limitation.
6. **Design rebalance batch binding** — default-deny is necessary but not sufficient. Propose either (a) full batch hash in every signer's payload, (b) explicit session-set allowlists, or (c) operator-only enforcement.
7. **Propose max TTL** — simple upper bound on `expires_at` at provisioning time.
8. **Map `close`/`rebalance` through ActionGateway** — or explicitly document why they're exempt from rate limits.

### Architecture decisions needed from Yellow

9. **Is `submit_state` one permission or many?** — transition-type granularity would make channel scope meaningful beyond asset allowlisting.
10. **One version bump or two?** — coordinate session-key scope v2 with per-intent-quorums v2 packer changes.
11. **Should empty scope (`[]`/`[]`) be rejected at provisioning?** — fail-closed recommendation.
