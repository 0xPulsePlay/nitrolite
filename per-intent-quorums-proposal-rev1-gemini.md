# Critical Review: Per-Intent Quorums Proposal (Rev 1)

This document synthesizes a parallel review of the `per-intent-quorums-proposal.md` by three independent AI subagents focusing on Architecture & Consensus, Security & Attack Vectors, and Implementation & Performance.

## Executive Summary

The core premise of the proposal—storing a map of intents to thresholds and passing them to the existing `verifyQuorum` function—is **the correct architectural seam** and is entirely feasible. The author is **not hallucinating** the codebase; the file paths, code references, and identification of `verifyQuorum` as the single chokepoint are highly accurate.

However, the proposal **severely hand-waves the complexity** (calling it "~5 lines of logic") and contains internal contradictions and critical omissions that would result in broken signatures, split-brain SDKs, or severe security vulnerabilities if implemented exactly as written.

---

## 1. The Backward Compatibility Contradiction (Blocker)

The proposal contradicts itself on how ABI packing will work for existing sessions and the session ID hash.

*   **The Contradiction:** In Part 2 (Step 2), it correctly notes that adding `IntentQuorums` to the ABI changes the session ID hash, suggesting a "V2" bump or a conditional packing branch. But in the "Migration path" section, it claims the new field is "strictly additive" and "byte-identical" for old clients.
*   **The Reality:** You cannot just add an empty slice to an ABI tuple and get the same hash. You *must* choose either a clean **V2 RPC namespace/packer**, OR you must write a **conditional packer** (if `len == 0`, omit the field entirely).
*   **The Risk:** If the Go backend and the TypeScript SDK disagree even slightly on whether to include an empty array or how to sort the intents, **every signature will fail to verify** with a cryptic "quorum not met" error.
*   **Recommendation:** Force a decision before implementation. We highly recommend a clean **V2 bump** to avoid the fragile tech debt of conditional V1 packing across multiple languages. Furthermore, **cross-language test vectors** (hardcoded hex hashes mapping fixed inputs to outputs in both Go and TS) are absolutely mandatory.

## 2. The DB "Preload" Security Footgun

The proposal suggests adding a new `AppSessionIntentQuorumV1` table and using a `QuorumFor(intent)` fallback mechanism (`if missing, return s.Quorum`).

*   **The Risk:** In GORM, relational data is not loaded by default. If *any* code path in the system fetches an `AppSessionV1` from the database but forgets to append `.Preload("IntentQuorums")`, the slice will be empty in memory.
*   **The Exploit:** Because of the fallback logic, forgetting to preload would cause highly restricted intents (e.g., a 100% quorum requirement for `withdraw`) to silently downgrade to the baseline quorum (e.g., 50%). This is a massive security vulnerability.
*   **Recommendation:** The DB read path must be strictly wrapped so it is impossible to fetch a session for verification without its intent quorums attached. The fallback logic makes "missing rows" equivalent to "weaker security," so integrity and reliable loading of this table are paramount.

## 3. Game-Theoretic "Rug" Dynamics & Policy Asymmetry

The proposal allows per-intent quorums to be *lower* than the baseline creation quorum. It treats this as neutral plumbing, but it introduces a whole new class of adversarial coalition dynamics.

*   **The Risk:** If a session is created with a baseline quorum of `100` (so everyone must sign to create it), but `withdraw` is set to `50`, a minority of participants could drain the session without the others realizing.
*   **The UX Hazard:** Users and integrators may equate the "creation quorum" with "all actions need that many signatures." The mismatch between mental models and actual enforcement can lead to unexpected losses.
*   **Recommendation:** The protocol needs explicit documentation warning integrators about withdraw/close asymmetry. Consider adding validation rules at creation (e.g., "withdraw quorum cannot be lower than operate quorum" or "no intent quorum can be lower than baseline") unless the product requirements explicitly demand allowing weaker intent quorums.

## 4. Ambiguity Around "Rebalance" and "Create" Intents

The author is confused or underspecified about how certain intents fit into the new model.

*   **Rebalance:** The proposal states in passing that "rebalance is node-gated" (Step 3) but then wires up `QuorumFor(AppStateUpdateIntentRebalance)` (Step 4). If rebalance signatures are not comparable to other intents (e.g., different authority model), a per-intent quorum for rebalance may be meaningless or dangerous. The protocol must explicitly define if rebalance is part of this multisig security model or handled entirely differently.
*   **Create:** The proposal keeps `appDef.Quorum` for creation but allows intent quorums for state updates. This means `create` is special-cased and cannot have a dedicated "intent quorum" map entry. While coherent, the product narrative must explicitly state that the map is for *state-update only* to avoid confusion.

## 5. Scope Underestimation

The proposal characterizes this as "~5 lines of actual logic changed at runtime."

*   **The Reality:** The runtime diff inside `verifyQuorum` might be small, but the **security** of the system depends on the packers, the database schema, the migrations, the RPC mapping, and ensuring every single `QuorumFor(intent)` call site passes the *intended* operation. The surface area for bugs (especially cross-language determinism and database preload omissions) is massive.

## Next Steps for Implementation

Before merging any code based on this proposal, the following must be addressed:

1.  **Commit to a V2 namespace** (or formally accept the tech debt of a conditional V1 packer) to resolve the backward compatibility contradiction.
2.  **Mandate cross-language test vectors** to guarantee TS SDK and Go backend ABI packing determinism.
3.  **Harden the DB read path** to ensure `IntentQuorums` are always loaded when `AppSessionV1` is retrieved for signature verification.
4.  **Clarify the rules for Rebalance and Create** intents relative to the map.
5.  **Document the game-theoretic risks** of asymmetric quorums for integrators.
