# Reality & Learning Planes — Confirmed Foundation

> Source: research/04 + rebuttals/04 + resolutions/04
> Funnel: 7 survived adversarial · 2 re-confirmed · 6 discarded · 0 open decision

## ✅ Confirmed buildable capabilities

### C-rl-1: Weibull temporal decay with per-type parameters
- **What it is**: 20+ memory types with distinct Weibull survival-function decay curves (`exp(-(ageHours/eta)^k)`); fast decay for transient signals (request: k=1.5/eta=72h), slow decay for durable knowledge (profile: k=0.3/eta=8760h)
- **oh-my-pi substrate**: `packages/mnemopi/src/core/weibull.ts:10-36` — `WEIBULL_PARAMS` constant with all 20 types; `weibull.ts:87-112` — `weibullBoost(timestamp, queryTime, memoryType, halflifeHours)` scoring function
- **Scope limit**: Parameters are STATIC per type; weibullBoost accepts only timestamp/queryTime/memoryType/halflifeHours — no recall_count, no validation_count, no adaptive feedback. The function signature is tightly coupled to call-sites in `beam/recall.ts`
- **Provenance**: survived adversarial review

### C-rl-2: Polyphonic 4-voice RRF recall
- **What it is**: Multi-strategy retrieval combining vector (0.35), graph (0.25), fact (0.25), and temporal (0.15) voices via Reciprocal Rank Fusion with diversity reranking
- **oh-my-pi substrate**: `packages/mnemopi/src/core/polyphonic-recall.ts:8` — `PolyphonicVoice` type; `polyphonic-recall.ts:190-195` — voice weights; `polyphonic-recall.ts:390-408` — RRF combination; `polyphonic-recall.ts:431-443` — Jaccard-based diversity reranking
- **Provenance**: survived adversarial review

### C-rl-3: Veracity types (5 weighted, 9 total union)
- **What it is**: `Veracity` type is a 9-member union (unknown|likely_true|true|false|stated|inferred|tool|imported|contested) + open string escape hatch; only 5 carry operational weights in consolidation (stated=1.0, inferred=0.7, tool=0.5, imported=0.6, unknown=0.8)
- **oh-my-pi substrate**: `packages/mnemopi/src/core/beam/types.ts:9-19` — full 9-member Veracity union; `packages/mnemopi/src/core/veracity-consolidation.ts:5-11` — `VERACITY_WEIGHTS` for the 5 operational types
- **Scope limit**: The remaining 4 values (likely_true, true, false, contested) have NO weight in the consolidation system — they are valid labels but dead in the confidence-update path. The `contested` state is never assigned by automated processes
- **Provenance**: survived adversarial review

### C-rl-4: Episodic/working memory with tiered degradation
- **What it is**: Working memory with TTL-based expiry consolidates to episodic via sleep(); episodic degrades through tiers (Tier 2 at 30 days, Tier 3 at 180 days with truncation to 300 chars max)
- **oh-my-pi substrate**: `packages/mnemopi/src/core/beam/consolidate.ts:57-60` — `TIER2_DAYS=30`, `TIER3_DAYS=180`, `TIER3_MAX_CHARS=300`; `consolidate.ts:389-434` — `consolidateToEpisodic()` moves working→episodic with fact extraction
- **Provenance**: survived adversarial review

### C-rl-5: Auto-learn skill minting as file overwrites
- **What it is**: Post-session auto-learn captures lessons and mints managed SKILL.md files via direct file overwrite into `~/.omp/agent/managed-skills`, gated by `autolearn.enabled`
- **oh-my-pi substrate**: `packages/coding-agent/src/autolearn/managed-skills.ts:152` — `writeManagedSkill()` creates/overwrites SKILL.md with no version history; `autolearn/controller.ts:113-114` — triggers after `minToolCalls` (default 5)
- **Scope limit**: Skills are overwritten without any journaling, git tracking, or version table — no rollback possible. Deduplication is by skill name only
- **Provenance**: survived adversarial review

### C-rl-6: Append-only JSONL session tree
- **What it is**: Sessions are append-only JSONL with tree semantics (parentId chains), branching via pointer movement (no deletion), and a closed 11-type entry union
- **oh-my-pi substrate**: `docs/session.md:108-121` — entry types: message, thinking_level_change, model_change, service_tier_change, compaction, branch_summary, custom, custom_message, label, ttsr_injection, session_init, mode_change
- **Scope limit**: The entry taxonomy is a CLOSED union — no extensible metadata field on SessionEntryBase. Adding new annotation fields (cognitive state, etc.) requires either a new entry type or a parallel tracking system
- **Provenance**: survived adversarial review

### C-rl-7: SHMR harmonize() with clustering, belief generation, and resonance logging
- **What it is**: Semantic Harmonic Memory Resonance clusters related facts/episodes by embedding similarity, generates beliefs (deterministic frequency fallback + LLM-driven path), applies mutations to the fact store, and logs to `memory_resonance_log`
- **oh-my-pi substrate**: `packages/mnemopi/src/core/shmr.ts:392-504` — `harmonize()` full pipeline; `shmr.ts:250-292` — `deterministicBeliefs()` frequency counting fallback; `shmr.ts:294-311` — harmony score (cosine similarity)
- **Provenance**: survived adversarial review

### C-rl-8: SHMR genuine belief evolution — update/dampen actions revise facts
- **What it is**: `applyBeliefs()` supports three mutation semantics: CREATE (new beliefs), UPDATE (revise fact content + confidence), DAMPEN (reduce confidence of contradicting facts by 0.15). INSERT OR REPLACE on `harmonic_beliefs` means repeated harmonization evolves belief confidence over time as new cluster members appear
- **oh-my-pi substrate**: `packages/mnemopi/src/core/shmr.ts:28-35` — `Belief` type with `action?: "create" | "update" | "dampen"`; `shmr.ts:323-329` — DAMPEN: `UPDATE facts SET confidence = MAX(0.1, confidence - 0.15)`; UPDATE: `UPDATE facts SET object = ?, confidence = ?`; `shmr.ts:331-341` — INSERT OR REPLACE on `harmonic_beliefs` with SHA256 belief_id keyed on cluster+subject+predicate+object
- **Scope limit**: The deterministic fallback path (`deterministicBeliefs()`) only emits action="create". Full update/dampen actions require the LLM belief generation path (`extractJsonFromLlmOutput` + `normalizeBelief`). This is NOT just deduplication, but the evolutionary path depends on LLM availability during harmonization
- **Provenance**: re-investigation CONFIRMED

### C-rl-9: Polyphonic voice-disagreement signal is extractable from retained per-voice scores
- **What it is**: Per-voice RRF rank scores are retained through the full recall pipeline: stored in `voiceScores` on each `PolyphonicResult`, propagated via `hydrateResults()` into `voice_scores`, and exposed to the coding-agent layer via `OrchestratedRecallResult`
- **oh-my-pi substrate**: `packages/mnemopi/src/core/polyphonic-recall.ts:17-22` — `PolyphonicResult.voiceScores: Partial<Record<PolyphonicVoice, number>>`; `polyphonic-recall.ts:399-408` — `combineVoices()` stores each voice's contribution separately; `polyphonic-recall.ts:483-499` — `hydrateResults()` propagates as `voice_scores: sortedVoiceScores(result.voiceScores)`; `packages/mnemopi/src/core/orchestrator.ts:24-29` — `OrchestratedRecallResult.voice_scores` exposed to consumer
- **Scope limit**: The per-voice data EXISTS but no disagreement metric is currently COMPUTED. A variance/divergence function over the voice scores is trivially derivable from retained state without architectural change, but must be built
- **Provenance**: re-investigation CONFIRMED

## ❌ Must build net-new (gaps)

### G-rl-1: Skill version history and rollback
- **What's missing**: Managed skills are overwritten via `writeManagedSkill()` with no journaling, no git history, no version table. Cannot roll back a skill regression or track effectiveness over time
- **Evidence of absence**: `packages/coding-agent/src/autolearn/managed-skills.ts:152-229` — direct file create/overwrite with no version storage
- **Nature of build**: extension (add SQLite journal table alongside existing writes; hook into `writeManagedSkill` to capture prior content before overwrite)

### G-rl-2: Voice-disagreement metric computation
- **What's missing**: Per-voice RRF scores are retained (C-rl-9) but no function computes a disagreement/divergence signal from them. Needed to trigger uncertainty-aware evidence collection
- **Evidence of absence**: grep: zero matches for "disagreement|divergence|voice.*variance" in `packages/mnemopi/src/core/`
- **Nature of build**: extension (pure function over existing `voiceScores` data; no schema change)

### G-rl-3: Structured failure taxonomy
- **What's missing**: Failures are captured as unstructured prose via `learn` tool and memory extraction. No `failure_type`, `root_cause`, `severity`, or `resolution_pattern` fields exist anywhere in the schema
- **Evidence of absence**: grep: zero matches for "failure_type|root_cause|severity.*critical" in `packages/mnemopi/`; `tools/learn.ts` schema has only `memory` (text) + `context` (text)
- **Nature of build**: extension (new table in beam schema + classification prompt in extraction pipeline)

### G-rl-4: Hypothesis/assumption lifecycle tracking
- **What's missing**: Veracity types (C-rl-3) encode source reliability but have no "hypothesis" or "assumption" state with verification workflow (proposed→testing→confirmed/rejected)
- **Evidence of absence**: `packages/mnemopi/src/core/beam/types.ts:9-19` — Veracity union contains no hypothesis/testing/assumption value; grep: zero matches for "hypothesis|assumption" as type literals in mnemopi
- **Nature of build**: extension (add veracity values + hypotheses table + lifecycle tracking; builds on existing veracity infrastructure)

### G-rl-5: Evidence derivation graph (provenance DAGs)
- **What's missing**: Facts track `sources_json` but no reasoning-chain or tool-invocation-to-fact derivation graph exists. Must account for the separate-DB topology (VeracityConsolidator has its own DB, not beam's)
- **Evidence of absence**: `packages/mnemopi/src/core/beam/schema.ts:24-426` — no `evidence_derivation` or similar table; `beam/index.ts:187` — `veracityConsolidator = null` in main BeamMemory (only instantiated in PolyphonicRecallEngine at `polyphonic-recall.ts:202`)
- **Nature of build**: greenfield subsystem (new table(s) respecting DB topology; must decide whether to live in beam DB or polyphonic-recall DB or a third path)

### G-rl-6: Adaptive Weibull decay (recall-frequency feedback)
- **What's missing**: Weibull parameters are static per type. `recall_count` and `last_recalled` fields exist in the schema (`beam/schema.ts:283-286`) but are NOT consumed by `weibullBoost()` — the function signature (`weibull.ts:87-112`) has no such parameters
- **Evidence of absence**: `weibull.ts:87-112` — `weibullBoost(timestamp, queryTime, memoryType, halflifeHours)` — no recall_count/validation_count parameter
- **Nature of build**: core change (modify weibullBoost signature + thread recall_count through scoring pipeline in `beam/recall.ts`)

## 🗑 Discarded overreach (do NOT re-propose)

- Veracity as coherent 5-type system — REFUTED: the union has 9 members but only 5 are weighted; 4 (likely_true, true, false, contested) are dead labels in the consolidation path (`beam/types.ts:9-19` vs `veracity-consolidation.ts:5-11`)
- Extend `consolidated_facts` schema with provenance DAG (Section 5.1 SQL proposals) — REFUTED: `consolidated_facts` lives in VeracityConsolidator's SEPARATE database, not beam schema; cross-DB ALTER TABLE proposals are architecturally infeasible (`beam/index.ts:187` sets `veracityConsolidator = null`; only `polyphonic-recall.ts:202` instantiates it)
- Cognitive-state-machine per session entry — REFUTED: session entries use a closed 11-type union with no extensible metadata field; tagging each entry with cognitive phase conflicts with append-only JSONL forward-compatibility contract (`docs/session.md:108-121`)
- "Bayesian confidence updates" as evidence grading mechanism — REFUTED: `bayesianUpdate()` is `(1.0 - current) * weight * 0.3` — a fixed-proportion diminishing-returns heuristic, not Bayesian inference (no prior, no likelihood, no posterior normalization) (`veracity-consolidation.ts:247-251`)
- Adaptive Weibull via "Modify `weibullBoost()` to accept recall/validation counts" as simple extension — REFUTED: the function signature and all call-sites are tightly coupled; threading recall_count requires pipeline-wide changes, not a simple param addition (`weibull.ts:87-112`)
- `session_reflect` hook at session end for Reality→Learning feedback — REFUTED: infeasible under `SESSION_SHUTDOWN_HANDLER_TIMEOUT_MS = 2_000` hard cap; `session_shutdown` is fire-and-forget teardown that "MUST NOT hold /exit hostage"; agent/tools are already disposed when hook fires; full session tree traversal incompatible with 2s budget (`runner.ts:88`, `agent-session.ts:6717-6752`)

## ⚠ Open design decision

None.
