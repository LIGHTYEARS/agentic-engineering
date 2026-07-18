# Resolution: Reality & Learning Planes — UNVERIFIABLE Findings (Doc 04)

## Summary

| Verdict | Count |
|---------|-------|
| CONFIRMED (feasible/supported) | 2 |
| REFUTED (infeasible/unsupported) | 1 |
| Irreducibly Unverifiable | 0 |

All three findings resolved to terminal verdicts through deeper source tracing.

---

## Resolutions

### Finding 1: SHMR harmonize() as "knowledge evolution" vs deduplication

```
Finding: Section 3.3 states "SHMR performs knowledge evolution through clustering,
belief generation, and harmony scoring" — the rebuttal questions whether this is
genuine belief evolution (supersession/revision over time) or merely
deduplication + clustering.

Prior Verdict: ? UNVERIFIABLE

New Verdict: ✓ CONFIRMED (proposal feasible/supported)

Deeper Evidence:
1. /tmp/oh-my-pi/packages/mnemopi/src/core/shmr.ts:28-35 — The Belief type
   defines `action?: "create" | "update" | "dampen"`. Three distinct mutation
   semantics: CREATE new beliefs, UPDATE existing facts (revise content +
   confidence), and DAMPEN contradicting facts (reduce their confidence).

2. /tmp/oh-my-pi/packages/mnemopi/src/core/shmr.ts:323-329 — applyBeliefs()
   implements DAMPEN: `UPDATE facts SET confidence = MAX(0.1, confidence - 0.15)
   WHERE fact_id = ?` — actively reduces confidence of contradicted facts.
   ALSO implements UPDATE: `UPDATE facts SET object = ?, confidence = ? WHERE
   fact_id = ?` — revises the CONTENT of an existing fact with new information.

3. /tmp/oh-my-pi/packages/mnemopi/src/core/shmr.ts:331-341 — beliefId is
   computed via SHA256 of `${clusterId}:${subject}:${predicate}:${object[:50]}`.
   The INSERT OR REPLACE semantics on `harmonic_beliefs` means repeated
   harmonization runs with DIFFERENT clusters produce DIFFERENT belief_ids, but
   same-cluster same-content beliefs get their confidence and provenance UPDATED
   (revision over time as new cluster members appear).

4. /tmp/oh-my-pi/packages/mnemopi/src/core/shmr.ts:467-479 — The iteration loop
   (up to SHMR_MAX_ITERATIONS=3) runs deterministicBeliefs() repeatedly per
   cluster. Each iteration can generate beliefs with different actions. The
   `totalContradictions` counter (line 477) tracks beliefs with action="dampen",
   proving the system actively resolves contradictions, not just clusters.

5. /tmp/oh-my-pi/packages/mnemopi/src/core/shmr.ts:233-248 — normalizeBelief()
   accepts LLM-generated belief JSON (via extractJsonFromLlmOutput at line
   212-227) with explicit action fields ("update"/"dampen"), showing the system
   was designed for an LLM-driven belief revision path beyond the deterministic
   frequency-counting fallback.

Reasoning: The rebuttal correctly identified that deterministicBeliefs() (the
fallback path) is a frequency counter that only emits action="create". However,
the rebuttal missed that:
(a) The Belief type explicitly supports "update" and "dampen" actions;
(b) applyBeliefs() implements both — UPDATE rewrites fact content, DAMPEN reduces
    confidence of contradicting facts;
(c) The LLM belief generation path (extractJsonFromLlmOutput + normalizeBelief)
    can produce "update"/"dampen" actions, which IS genuine belief revision;
(d) INSERT OR REPLACE on harmonic_beliefs means confidence evolves as new data
    clusters with old beliefs.

This is MORE than "dedup with extra steps." The system has three evolutionary
mechanisms: fact content revision (update), contradiction dampening (confidence
reduction), and belief confidence evolution (INSERT OR REPLACE). The proposal's
claim of "knowledge evolution" is supported by the architecture — it's just that
the DETERMINISTIC path alone is limited. The full harmonize pipeline including
LLM-generated beliefs provides genuine supersession/revision semantics.

Gate Outcome: ADVANCE to design — SHMR supports belief evolution via update/dampen
actions on existing facts, confidence dampening of contradictions, and INSERT OR
REPLACE on harmonic_beliefs. The minimal true capability: "SHMR can revise fact
content, reduce confidence of contradicted facts, and update belief confidence
over repeated harmonization passes."
```

---

### Finding 2: Polyphonic voice disagreement as learning signal

```
Finding: Key Finding 5 states "When vector, graph, fact, and temporal voices
strongly disagree, it indicates areas of knowledge uncertainty that should
trigger focused evidence collection." The rebuttal says no disagreement metric
exists and combineVoices is purely additive.

Prior Verdict: ? UNVERIFIABLE

New Verdict: ✓ CONFIRMED (proposal feasible/supported)

Deeper Evidence:
1. /tmp/oh-my-pi/packages/mnemopi/src/core/polyphonic-recall.ts:17-22 —
   PolyphonicResult interface retains `voiceScores: Partial<Record<PolyphonicVoice,
   number>>` as a per-memory breakdown of each voice's RRF contribution. This is
   NOT discarded — it persists through the full pipeline.

2. /tmp/oh-my-pi/packages/mnemopi/src/core/polyphonic-recall.ts:399-408 —
   combineVoices() stores EACH voice's contribution separately:
   `existing.voiceScores[result.voice] = ... + contribution`. The per-voice RRF
   rank score is retained as a named field on every combined result.

3. /tmp/oh-my-pi/packages/mnemopi/src/core/polyphonic-recall.ts:483-499 —
   hydrateResults() propagates voice_scores into the final PolyphonicMemoryResult:
   `voice_scores: sortedVoiceScores(result.voiceScores)`. The per-voice data
   survives all the way to the caller.

4. /tmp/oh-my-pi/packages/mnemopi/src/core/orchestrator.ts:24-29 —
   OrchestratedRecallResult interface exposes `voice_scores?:
   PolyphonicMemoryResult["voice_scores"]` to the consuming coding-agent layer.
   The per-voice rank divergence data is returned to the application.

5. /tmp/oh-my-pi/packages/mnemopi/src/core/polyphonic-recall.ts:431-443 —
   estimateSimilarity() already computes Jaccard intersection over voice
   membership (which voices found a given memory). This proves the architecture
   already reasons about per-voice presence/absence as a similarity signal —
   the inverse (low Jaccard = high disagreement) is trivially derivable.

Reasoning: The rebuttal's claim that "the combination is purely additive" and
"the architecture doesn't expose this information" is INCORRECT. The per-voice
RRF scores ARE retained in voiceScores on every PolyphonicResult, propagated
through hydrateResults() into the final output, and exposed via the orchestrator
interface all the way to the coding-agent consumer.

A disagreement signal is computable from the retained state: for any memory,
if voiceScores has entries for only 1-2 of 4 voices (high vector but absent from
graph/fact/temporal), that memory is controversial — ranked highly by one
retrieval strategy but invisible to others. The existing estimateSimilarity()
Jaccard computation on voice membership already demonstrates this pattern. No
intermediate state is discarded.

The proposal's claim is FEASIBLE: a disagreement metric (e.g., variance across
voice contributions, or count of voices that rank a memory in top-K vs those
that don't) can be computed from the existing voiceScores without any
architectural change — only a new computation over already-retained data.

Gate Outcome: ADVANCE to design — per-voice RRF scores are retained through
the full polyphonic recall pipeline. The minimal true capability: "Per-voice
rank contributions are preserved in voiceScores on every recall result; a
disagreement metric is computable from this existing retained state without
architectural modification."
```

---

### Finding 3: session_reflect hook at session end feasibility under timeout

```
Finding: Section 5.7 proposes "Add session_end hook in agent-session.ts that
triggers reflection" collecting "all cognitive state transitions, evidence
gathered, failures encountered" — the rebuttal questions feasibility under
SHUTDOWN_CONSOLIDATE_BUDGET_MS=1500ms and other hard timeouts.

Prior Verdict: ? UNVERIFIABLE

New Verdict: ✗ REFUTED (proposal infeasible/unsupported)

Deeper Evidence:
1. /tmp/oh-my-pi/packages/coding-agent/src/session/agent-session.ts:6717-6752 —
   The dispose() execution order is:
   (a) emit session_shutdown to extensions (line 6723-6724)
   (b) clearManagedTimers (line 6733)
   (c) abortRetry + abortCompaction (6747-6748)
   (d) cancelPostPromptTasks + agent.abort() (6749-6750)
   (e) await postPromptDrain (6751)
   (f) drainAutolearnCapture (6752) — 3s cap
   (g) cancelOwnAsyncJobs + dispose AsyncJobManager (6757-6768) — 3s cap
   (h) eval/ruby/julia kernel dispose (6769-6775)
   (i) browser tabs release (6783-6797) — 3s cap
   (j) MCP disconnect (6821-6831) — 3s cap
   (k) hindsight flush (6836-6838)
   (l) mnemopi dispose (6840-6841) — SHUTDOWN_CONSOLIDATE_BUDGET_MS=1500ms cap

2. /tmp/oh-my-pi/packages/coding-agent/src/extensibility/extensions/runner.ts:88 —
   SESSION_SHUTDOWN_HANDLER_TIMEOUT_MS = 2_000. Extension handlers for
   session_shutdown get a HARD 2-second budget. Any reflection hook registered
   as a session_shutdown extension handler MUST complete within 2 seconds.

3. /tmp/oh-my-pi/packages/coding-agent/src/session/agent-session.ts:717 —
   SHUTDOWN_CONSOLIDATE_BUDGET_MS = 1_500. This caps ONLY the mnemopi
   consolidation, not the entire dispose. The full dispose budget is the SUM
   of all sequential timeouts: 2s (session_shutdown) + 3s (autolearn) +
   3s (async jobs) + 3s (browser) + 3s (MCP) + 1.5s (mnemopi) = ~15.5s
   theoretical max, but the /quit path must remain responsive.

4. /tmp/oh-my-pi/packages/coding-agent/src/mnemopi/state.ts:550-560 —
   consolidate() during dispose runs with `{ full: false, extract: false,
   sleep: false }`. The LLM extraction is DISABLED on the shutdown path
   (extract: false). The sleep (working→episodic promotion) is SKIPPED
   (sleep: false). Only forceRetainCurrentSession + flushExtractions run.

5. /tmp/oh-my-pi/packages/coding-agent/src/session/agent-session.ts:6689-6699 —
   beginDispose() fires FIRST, setting #isDisposed=true. After this point,
   new eval starts throw, aside providers are detached, and the agent is
   aborted. The infrastructure for traversing the session tree (the agent,
   tool runners, eval kernels) is being torn down CONCURRENT with dispose steps.

6. /tmp/oh-my-pi/packages/coding-agent/src/extensibility/extensions/runner.ts:80-86 —
   Comment explicitly states: "session_shutdown is fire-and-forget teardown —
   extensions receive no result and the user has already asked to leave. A hung
   handler MUST NOT hold Ctrl+C / /exit hostage."

Reasoning: The proposal's SessionReflection interface requires:
- "all cognitive state transitions" → requires traversing the full session tree
- "evidence gathered" → requires correlating tool results across the session
- "failures encountered" → requires scanning session entries for error patterns
- "hypotheses resolved" → requires cross-referencing a hypothetical new table

This work profile (full session tree traversal + cross-reference) is incompatible
with the 2-second session_shutdown handler budget. The only available hook point
is session_shutdown (line 6723), which:
(a) Has a HARD 2s timeout (runner.ts:88);
(b) Fires at the VERY START of dispose, but beginDispose() has already aborted
    the agent and set isDisposed=true;
(c) Is explicitly documented as "fire-and-forget teardown" that "MUST NOT hold
    /exit hostage" (runner.ts:80-86);
(d) Cannot call LLM (agent is aborted), cannot run tools (eval execution is
    disposing), cannot write to mnemopi reliably (it's about to be disposed
    with a 1.5s cap).

Even if the hook skipped LLM calls and did pure in-memory work, traversing a
large session tree (sessions can have thousands of entries) and computing
structured reflection within 2 seconds is not guaranteed. More fundamentally,
the session_shutdown contract explicitly prohibits the kind of heavy processing
the proposal requires — it is designed for lightweight cleanup, not synthesis.

The proposal would need a fundamentally different architecture: a PRE-dispose
reflection phase that fires BEFORE beginDispose() tears down the infrastructure,
with its own dedicated time budget outside the existing dispose sequence. This
is not a minor adaptation — it requires restructuring the shutdown pipeline.

Gate Outcome: DISCARD — the session_shutdown hook has a hard 2s budget, fires
after infrastructure teardown begins, explicitly prohibits heavy processing,
and cannot access LLM or session tree traversal capabilities. Full reflection
at session end is architecturally infeasible within the existing shutdown model.
```
