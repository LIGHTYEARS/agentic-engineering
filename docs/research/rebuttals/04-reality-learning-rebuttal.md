# Rebuttal: Reality & Learning Planes OMP Mapping (Doc 04)

## Summary

| Verdict | Count |
|---------|-------|
| REFUTED | 5 |
| UNVERIFIABLE | 3 |
| Survived Scrutiny | 7 |

The proposal accurately identifies most existing mnemopi primitives but makes several incorrect specific claims about veracity types, overstates the feasibility of key adaptations by ignoring architectural constraints (the `consolidated_facts` table lives in a separate `VeracityConsolidator` module with its own database connection—not in the beam schema), and proposes a "cognitive state machine per session entry" that fundamentally conflicts with the append-only session model.

---

## Refutations

### Refutation 1: Veracity Types Mismatch — "5 veracity types" claim conflates two independent type systems

```
Target Claim/Proposal: "Mnemopi's veracity system already implements a multi-level provenance model with 5 veracity types weighted by reliability" — specifically that `Veracity` in `beam/types.ts` includes "stated|inferred|tool|imported|unknown" as a 5-type weighted system (Section 2.1)
Original Confidence: VERIFIED
Verdict: ✗ REFUTED
Reason: The `Veracity` type in `beam/types.ts:9-19` is a 9-member union (unknown|likely_true|true|false|stated|inferred|tool|imported|contested) plus an open `| string` escape hatch. The `VERACITY_WEIGHTS` in `veracity-consolidation.ts:5-11` only weights 5 of these (stated/inferred/tool/imported/unknown). The remaining 4 veracity values (likely_true, true, false, contested) have NO weight in the consolidation system — they are effectively dead labels in the Bayesian update path. The proposal maps these as a coherent "5 veracity types" system when in fact the type system declares 9 values and the weighting function ignores 4 of them. The `contested` state the proposal specifically references (Section 1.1 evidence bullet 3) is never assigned by any automated process and has no weight.
Counter-Evidence: /tmp/oh-my-pi/packages/mnemopi/src/core/beam/types.ts:9-19 (defines 9 Veracity values + open string); /tmp/oh-my-pi/packages/mnemopi/src/core/veracity-consolidation.ts:5-11 (only 5 weights defined); /tmp/oh-my-pi/packages/mnemopi/src/core/veracity-consolidation.ts:133-145 (clampVeracity maps unknown values to 'unknown' — contested/likely_true/true/false are all valid but unweighted)
```

### Refutation 2: `consolidated_facts` table is NOT in the beam schema — proposals assume unified database

```
Target Claim/Proposal: Section 5.1 proposes "Extend the `consolidated_facts` schema with a provenance DAG" via ALTER TABLE and new tables, implying consolidated_facts is part of the beam database. The proposal's architecture diagram (Section 5, Summary) treats veracity consolidation, episodic memory, and working memory as a unified store.
Original Confidence: VERIFIED
Verdict: ✗ REFUTED
Reason: The `consolidated_facts` and `conflicts` tables are created by `VeracityConsolidator.initTables()` in `veracity-consolidation.ts:176-207`, which takes its OWN `dbPath` parameter and opens its OWN database connection (`this.conn = conn ?? openDatabase(dbPath, ...)`). The beam schema (`beam/schema.ts:24-426`) does NOT create a `consolidated_facts` table anywhere. The `VeracityConsolidator` is only instantiated inside `PolyphonicRecallEngine` (`polyphonic-recall.ts:202`) — not in the main `BeamMemory` constructor where `veracityConsolidator` is explicitly set to `null` (`beam/index.ts:187`). This means the proposal's SQL alterations targeting `consolidated_facts` would need to operate on the polyphonic recall engine's database, which is a DIFFERENT subsystem from the beam memory store. The proposed `evidence_derivation` table referencing both `consolidated_facts` and session entries crosses database boundaries.
Counter-Evidence: /tmp/oh-my-pi/packages/mnemopi/src/core/beam/index.ts:187 (`this.veracityConsolidator = null`); /tmp/oh-my-pi/packages/mnemopi/src/core/polyphonic-recall.ts:202 (only place VeracityConsolidator is instantiated); /tmp/oh-my-pi/packages/mnemopi/src/core/beam/schema.ts:24-426 (entire beam schema — no consolidated_facts)
```

### Refutation 3: Cognitive State Machine per Session Entry is architecturally infeasible

```
Target Claim/Proposal: Section 5.1 proposes "ALTER TABLE episodic_memory ADD COLUMN cognitive_state TEXT" and "tag entries with cognitive phase" — a cognitive state machine that annotates each session entry with one of 7 states (perceiving|attending|reasoning|evaluating|deciding|acting|reflecting). Section 2.3 proposes "Tag entries with cognitive phase" as adaptation needed.
Original Confidence: INFERRED
Verdict: ✗ REFUTED
Reason: Session entries are append-only JSONL lines with a fixed schema per entry type. The session entry taxonomy (`docs/session.md:108-121`) defines a closed union: message|thinking_level_change|model_change|service_tier_change|compaction|branch_summary|custom|custom_message|label|ttsr_injection|session_init|mode_change. There is no extensible metadata field on `SessionEntryBase` — each entry type has a rigid shape. Adding a `cognitive_state` field to session entries would require either: (a) adding a new field to every entry type in the union (breaking the JSONL forward-compatibility contract since old readers would ignore it), or (b) creating a parallel tracking system. Furthermore, the proposal says to tag MEMORY entries (episodic_memory) with cognitive state, but cognitive state is a SESSION-level concept — by the time content reaches episodic_memory via `consolidateToEpisodic()`, it's been summarized across multiple tool calls and has no single cognitive phase. The `consolidateToEpisodic` function (`consolidate.ts:389-434`) takes a summary string and source working memory IDs — there's no insertion point for cognitive state metadata.
Counter-Evidence: /tmp/oh-my-pi/docs/session.md:108-121 (closed entry taxonomy, no extensible metadata); /tmp/oh-my-pi/packages/mnemopi/src/core/beam/consolidate.ts:389-434 (consolidateToEpisodic takes summary + sourceWmIds, no cognitive_state param); /tmp/oh-my-pi/packages/mnemopi/src/core/beam/types.ts:117-142 (RememberOptions has no cognitive_state field)
```

### Refutation 4: Proposal claims "Bayesian confidence updates" but the function is NOT Bayesian

```
Target Claim/Proposal: Section 2.1 evidence: "Bayesian confidence updates in `veracity-consolidation.ts:247-250`: `increment = (1.0 - current) * weight * 0.3`" — presented as a core Reality Plane mechanism for evidence grading.
Original Confidence: VERIFIED
Verdict: ✗ REFUTED
Reason: The `bayesianUpdate` method at `veracity-consolidation.ts:247-250` is named "Bayesian" but implements a simple heuristic increment formula: `(1.0 - current) * weight * 0.3`. This is NOT a Bayesian update. A Bayesian update requires a prior distribution, a likelihood function, and produces a posterior distribution via Bayes' theorem (P(H|E) = P(E|H)P(H)/P(E)). This formula is a fixed-proportion diminishing-returns increment with no likelihood ratio, no prior specification, and no normalization. Building a "Reality Plane" on the assumption this provides Bayesian evidence grading is unsound — the confidence score is merely monotonically increasing on repeated mentions, regardless of evidence quality or contradiction patterns. The proposal's gap table marks "Confidence scoring — Already functional" based on this mechanism, which overstates what it provides.
Counter-Evidence: /tmp/oh-my-pi/packages/mnemopi/src/core/veracity-consolidation.ts:247-251 (the actual bayesianUpdate method — no prior, no likelihood, just `(1-x)*w*0.3`)
```

### Refutation 5: Proposal's "Adaptive Forgetting" extension assumes recall_count modifies Weibull — but it doesn't

```
Target Claim/Proposal: Section 5.6 proposes "Make Weibull decay parameters adaptive based on recall frequency" claiming "Track recall frequency (already exists: `recall_count`, `last_recalled`)" and "Modify `weibullBoost()` to accept recall/validation counts" as implementation path steps 2-3.
Original Confidence: VERIFIED
Verdict: ✗ REFUTED
Reason: While `recall_count` and `last_recalled` fields DO exist in the schema (beam/schema.ts:283-286) and ARE incremented on recall (beam/recall.ts:887-889), the `weibullBoost()` function (weibull.ts:87-112) accepts ONLY `timestamp`, `queryTime`, `memoryType`, and `halflifeHours`. It has NO parameter for recall_count or validation_count. The proposal presents this as a simple modification ("Modify weibullBoost() to accept recall/validation counts") but the function's signature and call sites across the codebase are tightly coupled. More critically, `weibullBoost` is called from `beam/recall.ts` with per-row parameters at scoring time — adding recall_count would require threading it through the scoring pipeline. The proposal claims "already partially exists" for validation_count which IS in the schema but is NEVER read by ANY recall or decay function — it's written by the `memory_validations` table trigger system (`schema.ts:317-343`) and exposed in `MemoryRow` but has zero consumers in the decay/scoring path.
Counter-Evidence: /tmp/oh-my-pi/packages/mnemopi/src/core/weibull.ts:87-112 (weibullBoost signature — no recall/validation params); /tmp/oh-my-pi/packages/mnemopi/src/core/beam/recall.ts:728-730 (recencyDecay called with only timestamp + halflife, no recall_count)
```

---

## Unverifiable Claims

### Unverifiable 1: SHMR "harmony scoring" framing

```
Target Claim/Proposal: Section 3.3 states "SHMR performs knowledge evolution through clustering, belief generation, and harmony scoring" — implied as a sophisticated self-improving system.
Original Confidence: VERIFIED
Verdict: ? UNVERIFIABLE
Reason: The `harmonize()` function (shmr.ts:392-504) exists and does cluster + generate beliefs + compute harmony scores. However, the `deterministicBeliefs()` function (shmr.ts:250-292) that actually generates beliefs is a simple frequency counter — if an item appears 2+ times in a cluster, it becomes a belief. The "harmony score" (shmr.ts:294-311) just measures cosine similarity between belief embeddings and cluster centroids. Whether this constitutes "knowledge evolution" rather than "deduplication with extra steps" is a framing question that cannot be verified against code alone. The proposal builds extensively on this as a foundation for Learning Plane "belief formation" but the actual mechanism is more pedestrian than the description implies.
Counter-Evidence: /tmp/oh-my-pi/packages/mnemopi/src/core/shmr.ts:250-292 (deterministicBeliefs — frequency counting, not evolution); /tmp/oh-my-pi/packages/mnemopi/src/core/shmr.ts:294-311 (harmony score — simple cosine similarity metric)
```

### Unverifiable 2: "Polyphonic recall's voice disagreement is an untapped learning signal"

```
Target Claim/Proposal: Key Finding 5 states "When vector, graph, fact, and temporal voices strongly disagree, it indicates areas of knowledge uncertainty that should trigger focused evidence collection."
Original Confidence: INFERRED
Verdict: ? UNVERIFIABLE
Reason: The polyphonic recall system (polyphonic-recall.ts:182-533) combines voices via RRF (Reciprocal Rank Fusion) and applies diversity reranking. The `diversityRerank` method (line 412-430) uses Jaccard distance on voice-score presence to FILTER OUT similar results. But there is no mechanism to detect or measure "voice disagreement" as a signal — the combination is purely additive (combineVoices, line 390-411). To produce a "disagreement signal" would require comparing which memories are highly ranked by one voice but absent/low in others — the architecture doesn't expose this information. The claim that disagreement "indicates areas of knowledge uncertainty" is a theoretical hypothesis with no supporting implementation or measurement.
Counter-Evidence: /tmp/oh-my-pi/packages/mnemopi/src/core/polyphonic-recall.ts:390-411 (combineVoices is purely additive — no disagreement metric)
```

### Unverifiable 3: "session_reflect hook that fires at session end" feasibility

```
Target Claim/Proposal: Section 5.7 proposes "Add `session_end` hook in `agent-session.ts` that triggers reflection" as step 1 of the Reality→Learning feedback loop.
Original Confidence: INFERRED
Verdict: ? UNVERIFIABLE
Reason: The `dispose()` method in agent-session.ts (line 6712-6846) IS the session end path and it does fire autolearn capture drainage and mnemopi state disposal. However, it operates under strict timeout constraints (autolearn: 3s timeout at line 6664; mnemopi consolidate: configurable timeout at line 6841). The proposal's `SessionReflection` interface requires collecting "all cognitive state transitions, evidence gathered, failures encountered" — this would require traversing the full session tree at dispose time. The existing dispose path is already laden with timeout-bounded cleanup. Whether a "reflection" phase could be added without exceeding process-exit budgets on the `/quit` path (where `SHUTDOWN_CONSOLIDATE_BUDGET_MS = 1_500` at line 717) is unverifiable without performance profiling of actual session sizes.
Counter-Evidence: /tmp/oh-my-pi/packages/coding-agent/src/session/agent-session.ts:717 (SHUTDOWN_CONSOLIDATE_BUDGET_MS = 1500ms); /tmp/oh-my-pi/packages/coding-agent/src/session/agent-session.ts:6664 (autolearn 3s timeout); /tmp/oh-my-pi/packages/coding-agent/src/session/agent-session.ts:6841 (mnemopi dispose with configurable timeout)
```

---

## Missing Coverage

The original proposal failed to analyze these subsystems that would invalidate or significantly complicate its approach:

1. **Separate database topology**: The `VeracityConsolidator` creates its own tables in a database that may differ from the beam database. The proposal treats all mnemopi as one unified SQLite store, but the actual architecture has `beam/schema.ts` tables, `veracity-consolidation.ts` tables (created per PolyphonicRecallEngine instance), and `shmr.ts` tables (`harmonic_beliefs`, `memory_resonance_log`) — potentially in separate database files depending on initialization paths.

2. **`BeamMemory.veracityConsolidator = null`**: The main BeamMemory class explicitly sets `veracityConsolidator` to `null` (beam/index.ts:187). Only the `PolyphonicRecallEngine` instantiates it. This means veracity consolidation is gated behind polyphonic recall being enabled (controlled by `MNEMOPI_POLYPHONIC_RECALL` env var via `polyphonicRecallEnabled()`). The entire Reality Plane foundation (consolidated_facts, conflicts, Bayesian updates) is OPTIONAL and disabled by default unless polyphonic recall is active.

3. **Extraction categories don't include "resolved failures"**: The extraction.ts prompt (line 42-68) extracts: facts, instructions, preferences, timelines, kg. The proposal claims `docs/memory.md:47` shows "Phase 1 extracts resolved failures" — this is from the LOCAL memory backend's LLM-based session extraction (a different pipeline in `packages/coding-agent/src/memories/`), NOT from mnemopi's fact extraction. These are two completely separate extraction systems with different categories, timing, and storage targets.

4. **`ttsr_triggered` event has no failure context**: The proposal claims TTSR violations could "feed into the failure taxonomy" (Section 5.3), but the `TtsrTriggeredEvent` (extensibility/shared-events.ts:271-274) only carries `rules: Rule[]` — not what matched, not the offending content, and not any failure context. Building a failure taxonomy from TTSR would require significant event enrichment.

5. **No `session_end` event in the extension/hook system**: The proposal assumes a `session_end` hook can be added. Looking at the event types in hooks/types.ts and extensions/types.ts, the existing lifecycle events are: `session_start` (implied by tool creation), `ttsr_triggered`, `todo_reminder`, `tool_call`, `tool_result`, `auto_retry_*`, etc. There is NO `session_end` event. The dispose path fires no extension events — it tears down the extension runner itself. Adding a session_end hook would need to fire BEFORE dispose begins tearing down the infrastructure that hooks depend on.

---

## Survived Scrutiny

1. **Weibull decay parameters and formula** (Section 3.4): Accurately described. `weibull.ts:10-36` defines 20 memory types with the exact params cited. Formula `exp(-(ageHours/eta)^k)` at line 111 matches.

2. **Session entry taxonomy — 11 types** (Section 1.3): Confirmed at `docs/session.md:108-121`. The exact list matches. Append-only JSONL with tree semantics via parentId is correct.

3. **Tiered episodic degradation** (Section 3.4): `TIER2_DAYS=30`, `TIER3_DAYS=180`, `TIER3_MAX_CHARS=300` all confirmed at `consolidate.ts:57-60`. The degradation logic at lines 835-900 does exactly what's described.

4. **Auto-learn skill minting exists** (Section 1.7): `writeManagedSkill` at `managed-skills.ts:152` creates/overwrites SKILL.md in `~/.omp/agent/managed-skills`. Gated by `autolearn.enabled`. Skills DO get overwritten without version history — the "no versioning" gap is real.

5. **Polyphonic recall 4 voices with RRF** (Section 3.5): Confirmed — `PolyphonicVoice = "vector" | "graph" | "fact" | "temporal"` at `polyphonic-recall.ts:8`, weights at line 190-195 (vector=0.35, graph=0.25, fact=0.25, temporal=0.15), RRF at line 390-408.

6. **SHMR harmonize() clusters + beliefs + logging** (Section 3.3): Confirmed at `shmr.ts:392-504`. Does cluster by similarity, generate beliefs (deterministic path), apply to DB, log to `memory_resonance_log`.

7. **Managed skills have no version history** (Section 3.2): Confirmed — `writeManagedSkill` (`managed-skills.ts:152-229`) does a direct file overwrite (O_CREAT|O_EXCL for create, then open-for-update on subsequent). No journaling, no git, no version table.
