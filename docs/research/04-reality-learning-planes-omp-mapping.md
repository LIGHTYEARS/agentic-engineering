# Reality & Learning Planes: OMP Architecture Mapping

## 1. Inventory of Knowledge/Memory Mechanisms

### 1.1 Mnemopi Memory Engine (`packages/mnemopi/`)

**Claim:** Mnemopi is a full-featured SQLite-backed memory engine with working memory, episodic memory, fact extraction, knowledge graphs, veracity tracking, and Weibull-based temporal decay.

**Rationale:** The package contains distinct subsystems for each cognitive function: `beam/store.ts` for working memory, `beam/consolidate.ts` for episodic consolidation, `veracity-consolidation.ts` for truth tracking, `shmr.ts` for harmonic belief formation, `weibull.ts` for temporal decay, `episodic-graph.ts` for relationship graphs, `triples.ts` for knowledge triples, and `polyphonic-recall.ts` for multi-voice retrieval.

**Evidence:**
- `packages/mnemopi/src/core/beam/types.ts` defines `MemoryScope`, `TrustTier` (STATED|OBSERVED|INFERRED|SYSTEM), `Veracity` (unknown|likely_true|true|false|stated|inferred|tool|imported|contested)
- `packages/mnemopi/src/core/weibull.ts` defines 20+ memory types with per-type decay parameters (profile: k=0.3/eta=8760h; decision: k=1.0/eta=336h; error: k=1.1/eta=336h)
- `packages/mnemopi/src/core/veracity-consolidation.ts` implements Bayesian confidence updates and conflict detection/resolution
- `packages/mnemopi/src/core/shmr.ts` implements Semantic Harmonic Memory Resonance — clustering + belief generation

**Confidence:** VERIFIED

### 1.2 Local Summary Memory Pipeline (`packages/coding-agent/src/memories/`)

**Claim:** The local memory backend runs a two-phase extraction pipeline at startup: Phase 1 extracts durable signal per-session, Phase 2 consolidates across sessions into `MEMORY.md`, `memory_summary.md`, and auto-generated skills.

**Rationale:** The pipeline is described in `docs/memory.md` and implemented in `packages/coding-agent/src/memories/index.ts`.

**Evidence:**
- `docs/memory.md:46-55` describes Phase 1 (per-session extraction) and Phase 2 (consolidation into MEMORY.md + skills/)
- Configuration limits: `maxRolloutAgeDays=30`, `minRolloutIdleHours=12`, `maxRolloutsPerStartup=64`
- Output is redacted for secrets before writing
- Skills generated in `skills/` subdirectories under the memory root

**Confidence:** VERIFIED

### 1.3 Session System (`packages/coding-agent/src/session/`)

**Claim:** Sessions are append-only JSONL trees with branching, compaction entries, and a rich entry taxonomy supporting full audit trail reconstruction.

**Rationale:** `docs/session.md` documents 11 entry types in a tree structure with `parentId` chains.

**Evidence:**
- Entry types: `message`, `thinking_level_change`, `model_change`, `service_tier_change`, `compaction`, `branch_summary`, `custom`, `custom_message`, `label`, `ttsr_injection`, `session_init`, `mode_change`
- Tree semantics: append-only with mutable leaf pointer, `branch()` moves pointer without deleting entries
- Version 3 format with migrations from v1/v2

**Confidence:** VERIFIED

### 1.4 Compaction & Snapcompact (`packages/snapcompact/`, `packages/agent/src/compaction/`)

**Claim:** Compaction preserves context through summarization (LLM or bitmap archival) while tracking file operations, split-turn context, and boundary logic.

**Rationale:** `docs/compaction.md` documents multiple strategies (context-full, snapcompact, handoff) and pre-compaction pruning.

**Evidence:**
- `docs/compaction.md:133-142` describes snapcompact: history serialized onto PNG frames using pixel fonts
- Six trigger paths: manual, overflow recovery, incomplete-output recovery, threshold maintenance, mid-turn maintenance, idle maintenance
- File-operation tracking: cumulative read/write sets persisted in `details.readFiles`/`details.modifiedFiles`
- Pruning policies protect recent tool output (40,000 tokens) and skill reads

**Confidence:** VERIFIED

### 1.5 Skills System (`packages/coding-agent/src/extensibility/skills.ts`)

**Claim:** Skills are file-backed capability packs discovered at startup with provider-priority-based deduplication, supporting both authored and auto-learned (managed) skills.

**Rationale:** `docs/skills.md` documents the discovery pipeline and `SKILL.md` frontmatter schema.

**Evidence:**
- 7+ discovery providers with priority: native(100) > omp-plugins(90) > claude(80) > agents/codex(70) > opencode(55) > github(30) > omp-managed(5)
- Managed skills at `~/.omp/agent/managed-skills` via autolearn; always defer to same-named authored skill
- `skill://` URL protocol for on-demand access

**Confidence:** VERIFIED

### 1.6 Context Files & Rules (`docs/context-files.md`)

**Claim:** Context files provide persistent instruction injection via Markdown files discovered automatically at multiple scopes (user, project, ancestor walk-up).

**Rationale:** `docs/context-files.md` documents the full discovery pipeline and `@`-import expansion.

**Evidence:**
- Native `.omp/AGENTS.md` + `RULES.md` (sticky, always-apply)
- Walk-up to repo root for native files
- Priority-based shadowing across 8+ providers
- `@path` imports with 5-hop depth limit, cycle detection

**Confidence:** VERIFIED

### 1.7 Auto-Learn System (`packages/coding-agent/src/autolearn/`)

**Claim:** The auto-learn system captures lessons and mints managed skills post-session via the `learn` and `manage_skill` tools, gated by `autolearn.enabled`.

**Rationale:** Grep results across `autolearn/controller.ts`, `tools/learn.ts`, `autolearn/managed-skills.ts`.

**Evidence:**
- `autolearn/controller.ts:113-114`: triggers after `minToolCalls` (default 5)
- `tools/learn.ts:51-105`: persists to mnemopi, local `learned.md`, or hindsight backend
- `autolearn/managed-skills.ts:24-27`: writes to isolated `~/.omp/agent/managed-skills`
- `learned.md` capped at 100 entries, newest-first, deduped, secret-redacted

**Confidence:** VERIFIED

### 1.8 TTSR (Time-Traveling Stream Rules)

**Claim:** TTSR provides real-time pattern matching on agent output streams, enabling rule-based behavioral intervention mid-generation.

**Rationale:** `export/ttsr.ts` and `capability/rule-buckets.ts` show the pipeline.

**Evidence:**
- `config/settings-schema.ts:2892-2968`: TTSR settings (enabled, contextMode, interruptMode, repeatMode, repeatGap, builtinRules)
- Rules have `condition` (regex) and `astCondition` (ast-grep) with `scope` for targeting specific tool streams
- Built-in defaults shipped via `discovery/builtin-defaults.ts`

**Confidence:** VERIFIED

---

## 2. Reality Plane Mapping

The Reality Plane concerns evidence collection, distinguishing fact from assumption, tracking provenance, and maintaining cognitive state awareness.

### 2.1 Evidence Collection & Provenance

**Claim:** Mnemopi's veracity system already implements a multi-level provenance model with 5 veracity types weighted by reliability.

**Rationale:** The `VERACITY_WEIGHTS` constant and `TrustTier` type directly encode evidence strength.

**Evidence:**
- `veracity-consolidation.ts:5-11`: `VERACITY_WEIGHTS = { stated: 1.0, inferred: 0.7, tool: 0.5, imported: 0.6, unknown: 0.8 }`
- `beam/types.ts:9`: `TrustTier = "STATED" | "OBSERVED" | "INFERRED" | "SYSTEM"`
- `beam/types.ts:9-19`: `Veracity` includes "contested" state for disputed facts
- Bayesian confidence updates in `veracity-consolidation.ts:247-250`: `increment = (1.0 - current) * weight * 0.3`

**Confidence:** VERIFIED

**Mapping to Reality Plane:**
| Reality Plane Concept | OMP Mechanism | Gap |
|---|---|---|
| Evidence grading | `Veracity` + `TrustTier` | Missing: tool output citation chains |
| Provenance tracking | `sources_json` in consolidated_facts | Missing: full derivation DAGs |
| Confidence scoring | Bayesian updates per fact | Already functional |
| Conflict detection | `conflicts` table + `recordConflict()` | Missing: automated resolution policies |
| Supersession | `superseded_by` field | Already functional |

### 2.2 Fact vs. Assumption Distinction

**Claim:** The system distinguishes facts from assumptions through veracity labels but lacks explicit "assumption" tracking at the session level.

**Rationale:** Veracity types (stated, inferred, tool, imported) encode source reliability but don't capture the epistemological status during reasoning (hypothesis vs. verified conclusion).

**Evidence:**
- `beam/types.ts:9-19`: Veracity enum includes "likely_true", "true", "false" but not "hypothesis" or "assumption"
- `extraction.ts:42-68`: Extraction prompt categorizes into facts, instructions, preferences, timelines, kg — no explicit assumption category
- `beam/consolidate.ts:31-37`: `CONTAMINATED_VERACITY` marks inferred/tool/imported/unknown/false as needing care

**Confidence:** VERIFIED

**Gap:** Need an explicit `hypothesis` or `assumption` veracity state with lifecycle tracking (proposed → tested → confirmed/rejected).

### 2.3 Cognitive State Tracking (Seven States)

The Nine-Plane framework defines 7 cognitive states. Here's how OMP maps:

**Claim:** OMP partially models cognitive states through session entry types, mode changes, and memory tiers, but does not have explicit cognitive state tracking.

**Rationale:** Different session entry types and memory tiers correspond loosely to cognitive phases.

**Evidence:**
- `session-entries.ts` entry types: `session_init` (perceiving), `message` (reasoning/acting), `compaction` (consolidating), `branch_summary` (evaluating alternatives), `mode_change` (planning vs. executing)
- `beam/types.ts:48`: `BeamConfig.workingMemoryLimit` and `workingMemoryTtlHours` — working memory has explicit capacity/decay
- `beam/consolidate.ts:57-59`: `TIER2_DAYS=30`, `TIER3_DAYS=180` — degradation tiers

**Confidence:** INFERRED

**Proposed Cognitive State Model:**

| Cognitive State | Existing OMP Analog | Adaptation Needed |
|---|---|---|
| Perceiving | `session_init`, tool results arriving | Tag entries with cognitive phase |
| Attending | TTSR rule matching, skill activation | Formalize attention gating |
| Reasoning | Assistant message generation | Track reasoning chains explicitly |
| Evaluating | Branch summary, compaction decisions | Make evaluation criteria explicit |
| Deciding | `mode_change`, model_change | Record decision rationale |
| Acting | Tool calls (edit, write, bash) | Already tracked in session |
| Reflecting | Compaction summary, auto-learn capture | Formalize reflection triggers |

### 2.4 Evidence Lifecycle

**Claim:** The existing memory system tracks evidence lifecycle through temporal fields (`first_seen`, `last_seen`, `valid_until`, `superseded_by`) and mention counting.

**Rationale:** `ConsolidatedFact` and `MemoryRow` both carry temporal metadata.

**Evidence:**
- `veracity-consolidation.ts:33-43`: `ConsolidatedFact` has `first_seen`, `last_seen`, `mention_count`, `superseded`
- `beam/types.ts:205-206`: `MemoryRow` has `valid_until`, `superseded_by`
- `weibull.ts:28-36`: Decay parameters per type (error: k=1.1/eta=336h means errors decay faster than profiles: k=0.3/eta=8760h)

**Confidence:** VERIFIED

---

## 3. Learning Plane Mapping

The Learning Plane concerns failure taxonomy, asset versioning, evaluation expansion, and controlled forgetting.

### 3.1 Failure Capture & Taxonomy

**Claim:** OMP captures failures through memory extraction and the `learn` tool but lacks a structured failure taxonomy with root-cause categorization.

**Rationale:** The extraction pipeline captures "resolved failures" but doesn't classify them by type (logic error, integration failure, specification gap, etc.).

**Evidence:**
- `docs/memory.md:47`: Phase 1 extracts "resolved failures" from past sessions
- `tools/learn.ts:10-11`: `learn` tool schema captures `memory` (the lesson) and `context` (source context)
- `weibull.ts:32-33`: `error` type (k=1.1, eta=336h) and `issue` type (k=1.1, eta=336h) — errors decay in ~14 days
- No explicit taxonomy: no `failure_type`, `root_cause`, or `resolution_pattern` fields

**Confidence:** VERIFIED

**Gap:** Need structured failure schema:
```typescript
interface FailureRecord {
  type: "logic" | "integration" | "specification" | "environment" | "resource" | "knowledge";
  severity: "critical" | "major" | "minor";
  rootCause: string;
  resolution: string;
  preventionStrategy: string;
  recurrenceRisk: number; // 0-1
  relatedFacts: string[]; // fact_ids
}
```

### 3.2 Asset Versioning

**Claim:** Skills have implicit versioning through managed-skill writes and memory consolidation passes, but no explicit version tracking or diff history.

**Rationale:** Managed skills are overwritten on update; the local memory pipeline regenerates `MEMORY.md` on each consolidation pass.

**Evidence:**
- `autolearn/managed-skills.ts`: `writeManagedSkill` creates/overwrites `SKILL.md` — no version history
- `docs/memory.md:48-53`: Phase 2 produces `MEMORY.md`, `memory_summary.md`, `skills/` — stale skills are "pruned automatically"
- Session files are append-only with version field (currently v3)
- No git-based or SQLite-journaled versioning of knowledge artifacts

**Confidence:** VERIFIED

**Gap:** Need explicit versioning:
- Skills need version numbers, changelogs, and rollback
- Facts need supersession chains (partially exists via `superseded_by`)
- Memory summaries need delta tracking between consolidation passes

### 3.3 Knowledge Evolution (Eval Expansion)

**Claim:** The SHMR (Semantic Harmonic Memory Resonance) system performs knowledge evolution through clustering, belief generation, and harmony scoring, but doesn't expand evaluation criteria.

**Rationale:** `shmr.ts` clusters related memories and generates consolidated beliefs, which is knowledge synthesis rather than evaluation expansion.

**Evidence:**
- `shmr.ts:392-504`: `harmonize()` clusters facts/episodes by similarity, generates beliefs, applies them, logs to `memory_resonance_log`
- `shmr.ts:37-44`: `HarmonizeStats` tracks clusters, beliefs, contradictions, harmony_score
- `veracity-consolidation.ts:418-438`: `runConsolidationPass()` auto-resolves conflicts where a high-mention fact supersedes lower-confidence alternatives
- No mechanism to expand test coverage or evaluation criteria based on failures

**Confidence:** VERIFIED

**Gap:** Need eval expansion:
- After failure resolution, automatically generate new evaluation criteria
- Track which assertions/tests were missing that would have caught the failure
- Feed failure patterns back into TTSR rules for prevention

### 3.4 Controlled Forgetting

**Claim:** OMP implements sophisticated controlled forgetting through Weibull decay, tiered degradation (working → episodic with tier 2/3 aging), and consolidation that prunes working memory.

**Rationale:** Multiple mechanisms cooperate: temporal decay scoring, sleep consolidation, and explicit TTL.

**Evidence:**
- `weibull.ts:1-36`: 20+ memory types with distinct decay curves:
  - Fast decay: `request` (k=1.5, eta=72h), `event` (k=1.2, eta=168h), `decision` (k=1.0, eta=336h)
  - Slow decay: `profile` (k=0.3, eta=8760h/1yr), `relationship` (k=0.35, eta=8760h)
  - Formula: `exp(-(ageHours/eta)^k)` — Weibull survival function
- `beam/consolidate.ts:57-59`: `TIER2_DAYS=30`, `TIER3_DAYS=180`, `TIER3_MAX_CHARS=300` — progressive degradation
- `beam/consolidate.ts:389-434`: `consolidateToEpisodic()` moves working memory to episodic with fact extraction and graph ingestion
- `beam/types.ts:224-228`: `EpisodicMemoryRow` has `tier`, `degraded_at` fields
- `memories/index.ts:1264`: `MAX_LEARNED_LESSONS=100` — hard cap on lesson count (FIFO forgetting)

**Confidence:** VERIFIED

**Existing forgetting hierarchy:**
1. Working memory → TTL-based expiry (`workingMemoryTtlHours`)
2. Working → Episodic consolidation during `sleep()` 
3. Episodic tier degradation (tier 1 → tier 2 at 30 days → tier 3 at 180 days)
4. Tier 3 truncation to 300 chars max
5. Weibull decay scoring reduces recall probability over time
6. Supersession marks old facts as replaced
7. Conflict resolution marks losing facts as superseded

### 3.5 Polyphonic Recall as Learning Signal

**Claim:** The 4-voice polyphonic recall system (vector, graph, fact, temporal) provides diversity in memory retrieval that could serve as a learning signal for identifying knowledge gaps.

**Rationale:** When voices disagree strongly, it indicates areas of uncertain or contradictory knowledge.

**Evidence:**
- `polyphonic-recall.ts:8`: `PolyphonicVoice = "vector" | "graph" | "fact" | "temporal"`
- `polyphonic-recall.ts:190-194`: Voice weights: vector=0.35, graph=0.25, fact=0.25, temporal=0.15
- `polyphonic-recall.ts:390-408`: RRF (Reciprocal Rank Fusion) combines voices
- `polyphonic-recall.ts:430-441`: Diversity reranking reduces redundancy using voice-score Jaccard distance

**Confidence:** INFERRED

---

## 4. Gap Analysis

### 4.1 Reality Plane Gaps

| Gap | Severity | Description |
|---|---|---|
| No explicit cognitive state machine | HIGH | Session entries don't declare which cognitive phase the agent is in |
| No derivation DAGs | HIGH | Facts track `sources_json` but not full reasoning chains |
| No hypothesis lifecycle | MEDIUM | Can't mark claims as hypotheses pending verification |
| No real-time evidence quality scoring | MEDIUM | Veracity is assigned at write time, not re-evaluated dynamically |
| No tool-output provenance links | MEDIUM | `tool` veracity exists but doesn't link to specific tool invocations |
| Limited conflict resolution automation | LOW | `runConsolidationPass()` only resolves when mention_count > 2 |

### 4.2 Learning Plane Gaps

| Gap | Severity | Description |
|---|---|---|
| No structured failure taxonomy | HIGH | Failures captured as prose, not categorized by type/severity |
| No skill versioning | HIGH | Managed skills overwritten without history |
| No eval expansion loop | HIGH | Failures don't generate new evaluation criteria |
| No cross-session failure patterns | MEDIUM | Each session's failures are extracted independently |
| No forgetting governance | MEDIUM | Weibull params are fixed; no adaptive decay based on recall frequency |
| No skill effectiveness tracking | MEDIUM | No metrics on whether a skill actually prevents recurrence |
| No explicit unlearning | LOW | Supersession exists but no mechanism to forcefully invalidate a belief cluster |

### 4.3 Integration Gaps

| Gap | Severity | Description |
|---|---|---|
| Reality→Learning feedback loop missing | HIGH | Failures don't automatically create learning entries with structured metadata |
| No evidence-gated actions | HIGH | Agent can act without meeting evidence thresholds |
| Session ↔ Memory temporal correlation | MEDIUM | Hard to trace which session interactions led to which memory consolidations |
| TTSR ↔ Learning disconnect | MEDIUM | TTSR rule violations don't feed into the failure taxonomy |

---

## 5. Proposal: Concrete Adaptation Strategies

### 5.1 Reality Plane: Evidence Registry Extension

**Strategy:** Extend the `consolidated_facts` schema with a provenance DAG and cognitive state annotations.

```sql
-- New table: evidence derivation graph
CREATE TABLE evidence_derivation (
  id TEXT PRIMARY KEY,
  fact_id TEXT NOT NULL REFERENCES consolidated_facts(id),
  derived_from_fact_id TEXT REFERENCES consolidated_facts(id),
  derived_from_tool_call TEXT,  -- session_entry_id of source tool call
  derivation_type TEXT NOT NULL, -- 'observed' | 'inferred' | 'synthesized' | 'imported'
  reasoning TEXT,               -- natural language reasoning chain
  session_id TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Extend memory rows with cognitive state
ALTER TABLE episodic_memory ADD COLUMN cognitive_state TEXT; 
-- values: perceiving|attending|reasoning|evaluating|deciding|acting|reflecting
```

**Implementation path:**
1. Add `cognitive_state` field to `RememberOptions` in `beam/types.ts`
2. The coding-agent wrapper tags memories based on context: tool results → `perceiving`, compaction → `reflecting`, learn captures → `deciding`
3. Extend `VeracityConsolidator` with a `derivation_graph` table
4. Store `session_entry_id` in fact sources when facts are extracted from specific tool outputs

### 5.2 Reality Plane: Hypothesis Lifecycle

**Strategy:** Add a `hypothesis` veracity state with explicit verification workflow.

```typescript
// New veracity states
type ExtendedVeracity = Veracity | "hypothesis" | "testing" | "confirmed" | "rejected";

interface HypothesisRecord {
  factId: string;
  hypothesis: string;
  evidenceRequired: string[];  // what would confirm/reject
  evidenceCollected: string[]; // fact_ids of supporting/contradicting evidence
  status: "proposed" | "testing" | "confirmed" | "rejected";
  confirmedAt?: string;
  rejectedReason?: string;
}
```

**Implementation path:**
1. Extend `Veracity` type in `types.ts` with `hypothesis`/`testing`/`confirmed`/`rejected`
2. Add `hypotheses` table to beam schema
3. TTSR rule: when agent states an assumption, prompt it to register as hypothesis
4. Session hook: after tool results, check if any hypothesis can be confirmed/rejected

### 5.3 Learning Plane: Failure Taxonomy Schema

**Strategy:** Create a structured failure table in mnemopi with categorization, and feed TTSR violations into it.

```sql
CREATE TABLE failure_records (
  id TEXT PRIMARY KEY,
  session_id TEXT NOT NULL,
  entry_id TEXT,               -- session entry where failure occurred
  failure_type TEXT NOT NULL,  -- logic|integration|specification|environment|resource|knowledge
  severity TEXT NOT NULL,      -- critical|major|minor
  description TEXT NOT NULL,
  root_cause TEXT,
  resolution TEXT,
  prevention_strategy TEXT,
  recurrence_risk REAL DEFAULT 0.5,
  related_fact_ids TEXT,       -- JSON array of fact_ids
  generated_rule_id TEXT,      -- TTSR rule created to prevent recurrence
  generated_skill_id TEXT,     -- managed skill capturing the fix
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  resolved_at TIMESTAMP
);

CREATE TABLE failure_patterns (
  id TEXT PRIMARY KEY,
  pattern_name TEXT NOT NULL,
  failure_type TEXT NOT NULL,
  occurrence_count INTEGER DEFAULT 1,
  last_occurrence TEXT,
  prevention_effectiveness REAL DEFAULT 0.0,  -- 0-1, tracks if rules/skills prevent recurrence
  related_failure_ids TEXT  -- JSON array
);
```

**Implementation path:**
1. Add `failure_records` and `failure_patterns` tables to beam schema
2. Hook into `ttsr_triggered` event to capture rule violations as minor failures
3. Hook into error-recovery paths (overflow, incomplete) as severity-graded failures
4. After each failure resolution, prompt the agent to classify and record via new `record_failure` tool
5. Pattern detection: during `sleep()`/consolidation, cluster failures by type and generate patterns

### 5.4 Learning Plane: Skill Versioning

**Strategy:** Add version tracking to managed skills using a SQLite journal alongside the SKILL.md files.

```sql
CREATE TABLE skill_versions (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  skill_name TEXT NOT NULL,
  version INTEGER NOT NULL,
  content_hash TEXT NOT NULL,
  content TEXT NOT NULL,       -- full SKILL.md at this version
  change_reason TEXT,
  effectiveness_score REAL,    -- 0-1, updated by feedback
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  UNIQUE(skill_name, version)
);

CREATE TABLE skill_effectiveness (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  skill_name TEXT NOT NULL,
  session_id TEXT NOT NULL,
  invoked_at TIMESTAMP,
  outcome TEXT,               -- 'success' | 'partial' | 'failure' | 'not_applicable'
  feedback TEXT
);
```

**Implementation path:**
1. Modify `writeManagedSkill` in `autolearn/managed-skills.ts` to journal each write
2. Track invocations via `skill://` reads (hook into `skill-protocol.ts`)
3. Add effectiveness feedback after sessions where skills were used
4. Implement rollback via `manage_skill` action `rollback`

### 5.5 Learning Plane: Eval Expansion Loop

**Strategy:** After failure resolution, automatically generate TTSR rules and evaluation criteria.

```typescript
interface EvalExpansionEntry {
  failureId: string;
  generatedRuleCondition?: string;   // regex for TTSR
  generatedTestCriteria?: string;    // acceptance criteria
  generatedSkillUpdate?: string;     // skill enhancement
  effectivenessTracking: {
    preventedRecurrences: number;
    falsePositives: number;
    lastEvaluated: string;
  };
}
```

**Implementation path:**
1. After `learn` tool captures a failure lesson, trigger eval expansion
2. Generate a candidate TTSR rule condition from the failure pattern
3. Write rule to `.omp/rules/` with `condition:` matching the failure's code pattern
4. Track whether the rule fires (true positives) and whether it false-alarms
5. Prune rules with high false-positive rates (adaptive forgetting of overly broad rules)

### 5.6 Learning Plane: Adaptive Forgetting

**Strategy:** Make Weibull decay parameters adaptive based on recall frequency and task relevance.

```typescript
// Current: fixed params per type
// Proposed: adaptive params based on usage
interface AdaptiveDecayParams {
  baseK: number;          // from WEIBULL_PARAMS
  baseEta: number;        // from WEIBULL_PARAMS
  recallBoost: number;    // each recall extends eta
  confirmationBoost: number; // each confirmation reduces k (more long-term)
  contradictionPenalty: number; // contradictions increase k (faster decay)
}

function adaptiveWeibullBoost(
  timestamp: string,
  memoryType: string,
  recallCount: number,
  validationCount: number,
  contradictionCount: number,
): number {
  const base = WEIBULL_PARAMS[memoryType];
  const adaptedEta = base.eta * (1 + recallCount * 0.1 + validationCount * 0.2);
  const adaptedK = base.k * (1 - validationCount * 0.05 + contradictionCount * 0.1);
  const ageHours = (Date.now() - Date.parse(timestamp)) / 3_600_000;
  return Math.exp(-((ageHours / adaptedEta) ** adaptedK));
}
```

**Implementation path:**
1. Extend `MemoryRow` with `validation_count` (already partially exists: `validator`, `validated_at`, `validation_count`)
2. Track recall frequency (already exists: `recall_count`, `last_recalled`)
3. Modify `weibullBoost()` to accept recall/validation counts
4. During `sleep()` consolidation, apply adaptive decay to decide which memories degrade

### 5.7 Integration: Reality→Learning Feedback Loop

**Strategy:** Create a `session_reflect` hook that fires at session end, correlating session events with knowledge updates.

```typescript
interface SessionReflection {
  sessionId: string;
  cognitiveTransitions: Array<{
    fromState: CognitiveState;
    toState: CognitiveState;
    trigger: string;
    entryId: string;
  }>;
  evidenceCollected: Array<{
    factId: string;
    source: "tool" | "user" | "inference";
    confidence: number;
  }>;
  failuresEncountered: FailureRecord[];
  lessonsLearned: string[];     // from learn tool calls
  hypothesesResolved: string[]; // confirmed or rejected
  skillsInvoked: string[];
  skillEffectiveness: Record<string, "helpful" | "irrelevant" | "harmful">;
}
```

**Implementation path:**
1. Add `session_end` hook in `agent-session.ts` that triggers reflection
2. Collect all cognitive state transitions, evidence gathered, failures encountered
3. Feed structured reflection into mnemopi as a special `reflection` memory type
4. Use reflection data to update skill effectiveness scores and failure patterns
5. Trigger eval expansion for any unresolved failures

---

## Summary Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        REALITY PLANE                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────┐   ┌───────────────┐   ┌──────────────────┐   │
│  │  Evidence     │   │  Hypothesis   │   │  Cognitive State │   │
│  │  Registry     │   │  Lifecycle    │   │  Machine         │   │
│  │  (veracity +  │   │  (proposed→   │   │  (7 states per  │   │
│  │   derivation  │   │   confirmed/  │   │   session entry) │   │
│  │   DAG)        │   │   rejected)   │   │                  │   │
│  └──────┬───────┘   └──────┬────────┘   └────────┬─────────┘   │
│         │                   │                      │              │
│         └───────────────────┼──────────────────────┘              │
│                             │                                      │
│         ┌───────────────────▼──────────────────────┐              │
│         │          Provenance Graph                  │              │
│         │  (fact→source→tool_call→session_entry)    │              │
│         └───────────────────┬──────────────────────┘              │
│                             │                                      │
└─────────────────────────────┼──────────────────────────────────────┘
                              │ feedback
┌─────────────────────────────▼──────────────────────────────────────┐
│                        LEARNING PLANE                               │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────┐   ┌───────────────┐   ┌──────────────────────┐  │
│  │  Failure      │   │  Skill        │   │  Eval Expansion      │  │
│  │  Taxonomy     │   │  Versioning   │   │  Loop                │  │
│  │  (typed,      │   │  (journal +   │   │  (failure→TTSR rule  │  │
│  │   severity,   │   │   effective-  │   │   + test criteria)   │  │
│  │   patterns)   │   │   ness)       │   │                      │  │
│  └──────┬───────┘   └──────┬────────┘   └────────┬─────────────┘  │
│         │                   │                      │                 │
│         └───────────────────┼──────────────────────┘                 │
│                             │                                         │
│         ┌───────────────────▼──────────────────────┐                 │
│         │       Adaptive Forgetting                  │                 │
│         │  (Weibull + recall-count + validation)     │                 │
│         │  (SHMR harmony + contradiction decay)      │                 │
│         └────────────────────────────────────────────┘                 │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## Key Findings

1. **OMP is remarkably well-positioned for Reality Plane implementation.** The veracity system, trust tiers, Bayesian confidence, conflict detection, and supersession already form a solid evidence management foundation. The primary gaps are explicit cognitive state tracking and full derivation DAGs.

2. **Learning Plane has strong primitives but lacks structure.** Weibull decay, tiered consolidation, SHMR harmonization, and auto-learn capture provide the building blocks, but failures are unstructured prose, skills lack versioning, and there's no feedback loop from failures to evaluation criteria.

3. **The integration between planes is the largest gap.** Reality events (evidence quality, hypothesis resolution) should trigger Learning updates (failure records, skill enhancements), but this causal bridge doesn't exist yet.

4. **TTSR is an underutilized governance→learning connector.** Rule violations could automatically feed the failure taxonomy, and failure patterns could automatically generate new TTSR rules — creating a closed learning loop.

5. **Polyphonic recall's voice disagreement is an untapped learning signal.** When vector, graph, fact, and temporal voices strongly disagree, it indicates areas of knowledge uncertainty that should trigger focused evidence collection (Reality) or hypothesis registration.
