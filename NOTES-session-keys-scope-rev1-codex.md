# Session Keys Scope Review (rev1, Codex)

Date: 2026-04-20  
Input reviewed: `NOTES-session-keys-scope.md`  
Method: three independent parallel reviews + direct source validation in this repo

---

## Executive verdict

**Verdict: FEASIBLE_WITH_MAJOR_FIXES**

The proposal direction is mostly feasible and contains strong ideas, but one foundational claim in the note is materially incorrect today. This is **not** pure hallucination, but it does include at least one critical misread that must be corrected before using the note as architecture truth.

---

## Consolidated findings by category

## 1) Critical (must fix before architecture decisions)

- The note says channel session-key assets are "stored but NOT checked" and that the channel permissions hook is "never bound to real storage." In current code, `channel_v1.Handler.getChannelSigValidator()` binds the callback to `ValidateChannelSessionKeyForAsset(...)`, and that DB query enforces latest version, expiry, metadata hash, and asset membership.
- Because of the above, the summary "channel session keys effectively have only expiry" is incorrect as a current-state statement.
- Sections that depend on this incorrect baseline should be rewritten (especially the current-state table and channel enforcement narrative).

## 2) High (likely to cause design drift or wrong implementation)

- "Two parallel, disjoint systems share no code" is overstated. Storage tables and provisioning RPCs are distinct, but key verification plumbing is shared in important paths (including app-session deposit path channel-state signature validation).
- Reusing `GatedAction`/`actionGateway` as the primary scope mechanism is incomplete today: `close` and `rebalance` intents map to empty gated action, and unknown gated actions no-op in `AllowAction`.
- Rebalance-specific scope semantics are underspecified for caps: gross vs net amount, per-session vs batch semantics, and whether all sessions in a rebalance batch must satisfy a key's scope.

## 3) Medium (important unknowns to resolve)

- `create_app_session` with application-scoped session keys needs explicit validation: owner lookup path depends on `GetAppSessionKeyOwner(sessionKey, appSessionId)` and the app-session row timing may matter for application-only scope.
- Supersession/rotation race semantics are not yet explicitly documented at policy level (single authoritative active version, transactional guarantees).
- Replay/idempotency behavior for repeated signed payload delivery is not treated in this note and should be made explicit for scope-cap features.
- SDK local validation examples (for challenge flows) include permissive callback patterns that should not be interpreted as server policy enforcement.

## 4) Strong parts (keep and build on)

- Legacy `scope` field analysis is directionally solid: v1 does not expose legacy scope semantics as a first-class API field.
- Rebalance analysis is valuable: multi-session atomicity and conservation constraints are real and materially affect scope design.
- Allowlist-only philosophy is strong. Avoid deny-lists in signed scope state.
- "Stateless first, stateful later" phased approach is pragmatic and implementable.

---

## Feasibility answer to the core question

- **Does the plan make sense?** Yes, the proposed direction (allowlist-only, intent/channel-op gating, stateless per-op caps first) is architecturally plausible.
- **Are there hallucinated critical pieces?** There is at least one critical incorrect claim about existing channel session-key enforcement. So: not wholesale hallucination, but not fully reliable as written.

---

## Recommended next steps (ordered)

1. Correct baseline facts in the source note for channel session-key enforcement and update the current-state matrix accordingly.
2. Add a "Current vs Proposed Enforcement Matrix" with explicit columns: expiry, resource, intent/method, per-op cap, cumulative cap, rebalance handling.
3. Define rebalance scope invariants in writing before schema work: batch-wide scope checks, cap semantics (gross/net), and default policy for rebalance capability.
4. Decide and document create-time scope semantics for app-session keys (application-level vs session-level scope during `create_app_session`) and add targeted tests.
5. Draft v2 schema/versioning shape with signed payload compatibility notes for Go/TS SDKs and DB migrations.
6. Implement in two phases:
   - Phase 1: stateless checks only (intent/channel-op bitmasks, per-op caps, allowlists).
   - Phase 2: optional stateful caps (`max_total`, `max_per_session`) only after load/locking strategy is validated.
7. Add end-to-end tests for rebalance + session-key scope combinations (not only mock-validator unit paths).

---

## Practical discussion framing for Yellow

- Start from "v1 already enforces channel asset allowlist + metadata hash + expiry + latest version."
- Then ask "what additional scope dimensions are missing and worth adding now?"
- Keep v2 goals narrow for first landing: deterministic, stateless, auditable enforcement with no deny-list ambiguity.
