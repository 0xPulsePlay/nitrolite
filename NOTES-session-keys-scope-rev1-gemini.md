# Session Keys Scope — Consolidated Review & Revisions

**Date:** 2026-04-20
**Reviewers:** 3 Parallel Agent Instances (Architecture, Security, Implementation)
**Source Document:** `NOTES-session-keys-scope.md`

## Executive Summary

The original document provides a mostly coherent and highly useful architectural map for discussing session key scoping with the Yellow engineering team. The split between app-session and channel keys, the legacy `scope` analysis, the rebalance mechanics, and the argument for stateless allowlists over stateful deny-lists are all directionally sound.

However, **there is one critical factual error** regarding channel session keys that must be corrected before the meeting, as well as a few implementation nuances that the original document oversimplifies. Presenting the document as-is would understate the current security posture of channel keys and propose reusing the wrong internal gateways.

---

## 1. Critical Hallucinations & Factual Corrections

### 🚨 Channel Session Key Assets ARE Enforced
The most significant hallucination in the original document is the repeated claim that channel session keys have no scope beyond expiry, that the `channel_session_key_assets_v1` table is "stored but not checked," and that `VerifyChannelSessionKePermissionsV1` is never bound to real storage. 

**The Reality:** In the production clearnode path, asset scoping *is* strictly enforced.
- `channel_v1` wires the validator to the DB using `tx.ValidateChannelSessionKeyForAsset`.
- This function actively joins the `channel_session_key_assets_v1` table and verifies that the operation's asset is in the allowlist, alongside checking the `metadata_hash`, version, and expiry.
- **Action Required:** Update Sections 1, 2, and 5 to reflect that channel keys already have **resource (asset) scoping + metadata binding + expiry**. The actual missing feature for channel keys is finer-grained *method* control (e.g., distinguishing between `request_creation` and `submit_state`), not asset enforcement.

### 🚨 Mischaracterization of `GatedAction` / Action Gateway
Section 9 suggests reusing the existing `actionGateway` and `core.GatedAction` plumbing for session-key intent scoping.
- **The Reality:** The existing `actionGateway` is designed for *owner-centric economic gating* (app owners), not delegate-centric policy enforcement (session keys). 
- Furthermore, intents like `close` and `rebalance` map to an empty string `""` in `GatedAction()`, meaning they completely bypass this gateway. Session key scoping will require a dedicated, per-signature evaluation path rather than piggybacking on the app registry's action gateway.

---

## 2. Architectural Strengths (What the Doc Gets Right)

- **The Deny-List Footgun:** The argument to drop `denied_methods` and `denied_intents` is spot-on. Signed capabilities must be default-deny with explicit allowlists. Deny-lists create ambiguous precedence and introduce severe scope-creep vulnerabilities when new RPCs are added to the protocol.
- **Stateless vs. Stateful Enforcement:** The phased approach—prioritizing cheap, stateless payload checks (like intent bitmasks and method allowlists) and deferring expensive, stateful checks (like `max_total` and `max_per_session` which introduce DB hot-rows)—is excellent engineering judgment.
- **Rebalance Mechanics:** The analysis of `rebalance_app_sessions` is highly accurate. The doc correctly identifies that rebalance batches touch multiple sessions simultaneously and rely on a strict zero-sum conservation invariant. The warning that stateful limits become ambiguous during offsetting rebalance flows is a crucial insight.
- **Legacy `scope`:** The conclusion that the legacy `scope` string is purely aspirational and safely ignored in v1 is correct, giving the team a clean slate for the v2 schema.

---

## 3. Implementation Nuances & Edge Cases

### The `create_app_session` Bootstrap Issue
Section 5 claims that an application-only scope (`[]` + `[A]`) allows the key to authorize "any future session of app A." 
- **The Catch:** `GetAppSessionKeyOwner` uses a subquery to look up the `application_id` of the existing session. During `create_app_session`, the session row doesn't exist yet, meaning the subquery will return no rows and fail. Authorizing creation requires the deterministic `app_session_id` to be explicitly included in the key's provisioned scope beforehand.

### Intent Bitmask Verification Depth
The "v2 pitch" suggests checking the intent bitmask at the handler level (e.g., in `submit_app_state.go`).
- **The Catch:** Because a quorum can mix master wallet signatures and session key signatures (potentially with different policies), intent verification cannot happen globally after `verifyQuorum`. It must be evaluated **per-signature** inside `verifyQuorum`, where the signer type is recovered and that specific key's policy is loaded.

---

## 4. Recommended Next Steps

1. **Patch the Document:** Update `NOTES-session-keys-scope.md` to correct the narrative around channel session keys. Acknowledge that asset enforcement exists, and pivot the "v2 channel scope" pitch to focus purely on method/operation bitmasks.
2. **Clarify Creation Semantics:** Add a note about the `create_app_session` bootstrap edge case to ensure the team designs the application-level scope correctly.
3. **Prepare for the Meeting:** Frame the v2 pitch around three core pillars:
   - **App Sessions:** Intent bitmasks (enforced per-signature) and App Template allowlists.
   - **Channels:** Method bitmasks (`request_creation` vs `submit_state`).
   - **Limits:** Reiterate that stateless per-operation limits are safe, but stateful aggregate limits require strict definitions of "spent" (especially during rebalances) and should be deferred to Phase 2.