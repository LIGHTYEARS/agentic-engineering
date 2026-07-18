# Orchestration & Search Planes: OMP Architecture Mapping

## 1. Inventory of Orchestration/Search Mechanisms

### 1.1 Task Fan-Out (Task Tool)

Claim: OMP's `task` tool is the primary orchestration primitive, supporting both synchronous inline execution and asynchronous background job fan-out with a session-scoped concurrency semaphore.
Rationale: The task tool handles batch spawning of subagents (`{ context, tasks[] }`), routing each to either blocking inline execution or background jobs via `AsyncJobManager`, bounded by a resizable `Semaphore`.
Evidence: `packages/coding-agent/src/task/index.ts` (entry), `packages/coding-agent/src/task/parallel.ts:141-215` (Semaphore class), `packages/coding-agent/src/async/job-manager.ts` (AsyncJobManager with max 15 running jobs default). Settings: `task.maxConcurrency` default 32, `task.maxRecursionDepth` default 2.
Confidence: VERIFIED

### 1.2 Hub Messaging (IRC Bus)

Claim: OMP provides a process-global mailbox-based messaging bus (`IrcBus`) that enables peer-to-peer communication between agents, with delivery receipts, broadcast, and parked-agent revival on direct send.
Rationale: The hub tool merges the former `irc`, `job`, and `launch` tools into a single coordination surface. Agents are visible to each other in a flat namespace (no hierarchy) via `AgentRegistry.listVisibleTo()`.
Evidence: `packages/coding-agent/src/tools/hub/index.ts` (entry), `packages/coding-agent/src/irc/bus.ts` (process-global bus), `packages/coding-agent/src/registry/agent-registry.ts:170-174` (flat peer visibility).
Confidence: VERIFIED

### 1.3 Workspace Isolation (pi-iso)

Claim: OMP provides cross-platform copy-on-write workspace isolation through a native Rust PAL (`pi-iso`) with 8 backend strategies, enabling each subagent to operate on an independent filesystem snapshot.
Rationale: The `crates/pi-iso` crate implements APFS clonefile, btrfs snapshots, ZFS clone, Linux reflink, overlayfs (kernel + fuse fallback), ProjFS (Windows), block-clone (NTFS/ReFS), and recursive copy. A resolution function probes host availability and falls through candidates.
Evidence: `crates/pi-iso/src/lib.rs:46-66` (BackendKind enum), `crates/pi-iso/src/lib.rs:352-387` (resolve function with fallback chain), `packages/coding-agent/src/task/worktree.ts:411-444` (ensureIsolation lifecycle).
Confidence: VERIFIED

### 1.4 Branch-Mode Isolation & Merge

Claim: Isolated agents can commit to dedicated git branches (`omp/task/<id>`) and their changes are cherry-picked back to HEAD, preserving per-commit authorship and message history.
Rationale: `commitToBranch` creates a task branch, replays filtered agent commits (handling baseline WIP), and `mergeTaskBranches` cherry-picks the range sequentially with stash/pop for dirty working trees.
Evidence: `packages/coding-agent/src/task/worktree.ts:731-817` (commitToBranch), `packages/coding-agent/src/task/worktree.ts:836-928` (mergeTaskBranches with serialized cherry-pick).
Confidence: VERIFIED

### 1.5 Swarm Extension (DAG Pipelines)

Claim: The swarm extension provides declarative YAML-defined multi-agent DAG orchestration with topological wave scheduling, iterative pipeline mode, and filesystem-based inter-agent communication.
Rationale: The swarm extension parses `waits_for`/`reports_to` dependencies into a DAG, detects cycles via Kahn's algorithm, builds execution waves, and runs agents within a wave in parallel using `Promise.all`. Supports pipeline (N iterations), sequential, and parallel modes.
Evidence: `packages/swarm-extension/src/swarm/dag.ts:17-146` (buildDependencyGraph, detectCycles, buildExecutionWaves), `packages/swarm-extension/src/swarm/pipeline.ts:155-194` (parallel wave execution), `packages/swarm-extension/README.md:120-136` (mode documentation).
Confidence: VERIFIED

### 1.6 Concurrency Control Primitives

Claim: OMP provides two concurrency-limited parallel execution helpers (`mapWithConcurrencyLimit` for fail-fast, `mapWithConcurrencyLimitAllSettled` for all-settled) and a resizable `Semaphore` class for cross-call bounding.
Rationale: These primitives form the substrate for both task fan-out (semaphore bounds spawns) and internal pipelines (map-reduce commit analysis). The semaphore supports abort-aware acquisition and runtime resizing.
Evidence: `packages/coding-agent/src/task/parallel.ts:26-84` (mapWithConcurrencyLimit), `packages/coding-agent/src/task/parallel.ts:100-127` (allSettled variant), `packages/coding-agent/src/task/parallel.ts:141-215` (Semaphore with resize).
Confidence: VERIFIED

### 1.7 Agent Lifecycle Management

Claim: OMP manages agent state transitions through a process-global registry with four states (running/idle/parked/aborted) and an `AgentLifecycleManager` that handles TTL-based parking, on-demand revival, and concurrent park/revive coalescing.
Rationale: Finished agents remain addressable (idle), are parked to disk after a TTL, and revive on messaging. This enables persistent agent memory across multi-turn interactions without unbounded resource consumption.
Evidence: `packages/coding-agent/src/registry/agent-lifecycle.ts:58-385` (full lifecycle manager), `packages/coding-agent/src/registry/agent-registry.ts:24` (AgentStatus type), settings `task.agentIdleTtlMs` default 420000ms.
Confidence: VERIFIED

---

## 2. Mapping to the Orchestration Plane

### 2.1 Task DAG

Claim: OMP supports implicit task DAGs through two mechanisms: (1) the task tool's hierarchical spawn tree (parent/child with depth control), and (2) the swarm extension's explicit `waits_for`/`reports_to` dependency graph.
Rationale: The task tool creates a tree rooted at "Main" with `task.maxRecursionDepth` limiting depth. The swarm extension adds arbitrary DAG topology with cycle detection. However, the task tool itself does not natively support cross-sibling dependencies - only parent-to-child relationships.
Evidence: Task depth gating in `packages/coding-agent/src/task/executor.ts` (depth computation + tool stripping), swarm DAG in `packages/swarm-extension/src/swarm/dag.ts:17-50` (explicit dependency graph).
Confidence: VERIFIED

Claim: The task tool's spawn tree is NOT a true DAG - it is a tree with optional messaging. Sibling coordination requires the parent to orchestrate or agents to use `hub` messaging.
Rationale: Task spawns create independent children. There's no native mechanism to declare "agent B waits for agent A" within a single task batch - all items in a batch are independent. Cross-agent dependencies are achieved informally through messaging or file-based signals.
Evidence: Task batch semantics in `docs/tools/task.md:29-31` ("one independent spawn per item"), hub messaging docs showing peer communication as the coordination mechanism.
Confidence: VERIFIED

### 2.2 Agent Assignment & Roles

Claim: OMP implements agent role specialization through typed agent definitions with frontmatter-declared capabilities (tools, spawns, model, blocking behavior, output schema).
Rationale: Agents are discovered from `.omp/agents/` directories (project > user > plugin > bundled), each with a specific role description, tool allowlist, and spawn policy. The task tool resolves the requested agent type at execution time and enforces spawn policies.
Evidence: `docs/task-agent-discovery.md:24-28` (AgentDefinition shape), bundled agents: `scout`, `designer`, `reviewer`, `librarian`, `task`, `sonic`. Agent source precedence: project `.omp/agents` > user `~/.omp/agents` > plugin > bundled.
Confidence: VERIFIED

Claim: Agent selection in OMP is explicit (caller names the agent type) rather than capability-matched. The orchestrator does not auto-select the best agent for a task based on requirements.
Rationale: The `agent` field in task items is a string naming a specific agent type. There is no capability matching, skill assessment, or dynamic assignment based on task requirements.
Evidence: `docs/tools/task.md:39` (agent field is a string naming the type), task validation rejects unknown agent names with "Unknown agent ... Available: ..." error.
Confidence: VERIFIED

### 2.3 Context Management

Claim: OMP provides three mechanisms for context passing to subagents: (1) the `context` field in batch calls rendered into each spawn's system prompt, (2) `local://` shared artifact filesystem, and (3) the `agent://` protocol for reading sibling outputs.
Rationale: Batch mode injects shared context as a `CONTEXT` section in the subagent's system prompt. The local:// root is shared between parent and children. The agent:// protocol resolves to the saved output markdown of any sibling.
Evidence: `packages/coding-agent/src/task/executor.ts:2496-2498` (systemPrompt template with context injection), executor.ts sharing local:// root, `docs/tools/task.md:70-73` (agent:// and history:// protocols).
Confidence: VERIFIED

Claim: Subagents do NOT inherit conversation history from the parent. They start fresh with only: workspace tree, skills, context field content, local:// files, and an optional plan reference.
Rationale: The executor explicitly creates child sessions without history. Context is deliberately scoped to prevent unbounded token growth.
Evidence: `docs/tools/task.md:164` ("Child sessions do not inherit conversation history").
Confidence: VERIFIED

### 2.4 Budgets & Resource Limits

Claim: OMP implements a multi-layered budget system for subagents: concurrency cap (task.maxConcurrency=32), request budget (task.softRequestBudget=200 with 1.5x force-stop), wall-clock timeout (task.maxRuntimeMs), recursion depth (task.maxRecursionDepth=2), and output size caps (500KB / 5000 lines).
Rationale: These form a defense-in-depth approach. The soft budget injects a steering notice then force-stops; the wall-clock cap is a hard abort. Concurrency is enforced via the resizable semaphore. Max jobs in the background manager is 15 (configurable via `async.maxJobs`, clamped 1-100).
Evidence: `packages/coding-agent/src/config/settings-schema.ts:4160-4270` (all settings with defaults), `packages/coding-agent/src/task/executor.ts:89-90` (SOFT_REQUEST_BUDGET per agent type: scout=100, sonic=50), `packages/coding-agent/src/task/types.ts` (MAX_OUTPUT_BYTES=500000, MAX_OUTPUT_LINES=5000).
Confidence: VERIFIED

---

## 3. Mapping to the Search Plane

### 3.1 Parallel Exploration

Claim: OMP enables parallel candidate generation through (1) task batch fan-out where multiple agents explore independently, and (2) the swarm extension's parallel wave execution where agents in the same wave run simultaneously.
Rationale: A task batch spawns N independent agents, each potentially exploring different solution approaches. The swarm's `parallel` mode runs all agents simultaneously. Both use filesystem or messaging to communicate results.
Evidence: `docs/tools/task.md:157-158` (parallelism via batch or parallel calls), `packages/swarm-extension/src/swarm/pipeline.ts:155-156` (Promise.all for wave agents), swarm README parallel mode: "all agents run in one wave".
Confidence: VERIFIED

### 3.2 Branch Isolation (Candidate Diversity)

Claim: OMP's isolation system enables parallel agents to make conflicting filesystem changes without interference, with each agent operating on a CoW snapshot of the workspace. Branch-mode isolation gives each agent its own git branch.
Rationale: When `isolated=true`, each agent gets an independent workspace via `ensureIsolation()`. In branch mode, changes commit to `omp/task/<id>` branches. This is structurally analogous to generating diverse solution candidates in isolation.
Evidence: `packages/coding-agent/src/task/worktree.ts:411-444` (ensureIsolation materializes a merged dir), branch naming `omp/task/${taskId}` in worktree.ts:747, `docs/tools/task.md:108-109` (isolation modes and merge strategies).
Confidence: VERIFIED

Claim: Branch isolation enables speculative parallel execution without merge until explicit selection. The `mergeTaskBranches` function cherry-picks branches sequentially and stops on first conflict.
Rationale: Branches can be independently reviewed before merge. The merge function serializes via `git.withRepoLock` and uses stash/pop for working-tree safety. Cherry-pick failures leave subsequent branches untouched, enabling partial merge of successful candidates.
Evidence: `packages/coding-agent/src/task/worktree.ts:836-928` (mergeTaskBranches: sequential cherry-pick, conflict detection, stash handling).
Confidence: VERIFIED

### 3.3 Candidate Diversity Mechanisms

Claim: OMP provides limited explicit diversity mechanisms - different agent types can be mixed in a single batch call, and model overrides enable different models per agent, but there is no explicit diversity objective or termination-on-sufficiency.
Rationale: A batch call can mix agent types (`agent` is per-item), and the swarm extension supports per-agent model overrides. However, these are caller-directed configurations, not automatic diversity maximization.
Evidence: `docs/tools/task.md:31` ("one call may mix agent types"), swarm README model override per agent, but no diversity metric or auto-selection logic found in codebase.
Confidence: VERIFIED

### 3.4 Termination Criteria

Claim: OMP's termination mechanisms are resource-based (budget exhaustion, wall-clock timeout, abort signal) and lifecycle-based (yield tool, reminder retries), not solution-quality-based.
Rationale: An agent terminates when: (1) it calls `yield`, (2) its soft budget (1.5x) triggers force-stop, (3) wall-clock timeout fires, (4) parent/external abort signal, or (5) 3 yield-reminder retries exhausted. There is no "good enough" solution-quality termination.
Evidence: `packages/coding-agent/src/task/executor.ts:1014-1028` (maxRuntimeMs timeout), `packages/coding-agent/src/task/executor.ts:1372-1383` (soft budget enforcement), `docs/tools/task.md:138` (MAX_YIELD_RETRIES=3).
Confidence: VERIFIED

---

## 4. Gap Analysis

### 4.1 Orchestration Plane Gaps

#### 4.1.1 No Declarative Task DAG in Core

Claim: The core task tool lacks a declarative DAG specification for inter-task dependencies. Dependencies between siblings must be orchestrated by the parent or through messaging.
Rationale: The swarm extension fills this gap but is a separate extension, not integrated into the core task tool. The core supports only tree-shaped parent/child relationships.
Evidence: Task batch items are independent (no `waits_for` field), swarm is in `packages/swarm-extension/` (a separate package), no DAG primitives in `packages/coding-agent/src/task/`.
Confidence: VERIFIED

#### 4.1.2 No Cost-Aware Scheduling

Claim: OMP's concurrency semaphore treats all agents equally regardless of resource cost. There is no cost model, priority queue, or weighted scheduling.
Rationale: The `Semaphore` counts slots uniformly. A cheap scout and an expensive opus-model task consume one slot each. There's no mechanism to prioritize low-cost explorations over expensive ones or to model estimated completion times.
Evidence: `packages/coding-agent/src/task/parallel.ts:141-215` (simple counting semaphore, no weights), `packages/coding-agent/src/task/index.ts:576-580` (semaphore sized only from maxConcurrency count).
Confidence: VERIFIED

#### 4.1.3 No Dynamic Context Pack Assembly

Claim: Context passed to subagents is static at spawn time. There is no mechanism for dynamically assembling minimal context packs based on the agent's declared information needs.
Rationale: The `context` field is a static string written by the parent at call time. While agents can read files on demand, there's no system that pre-selects relevant context based on the task's domain, minimizing token cost while maximizing relevance.
Evidence: `docs/tools/task.md:31-37` (context is a plain string field), executor injects it verbatim into system prompt.
Confidence: VERIFIED

#### 4.1.4 No Global Budget Accounting

Claim: OMP tracks per-subagent budgets but has no aggregate budget awareness across a spawn tree. Total cost accumulates without a global cap or allocation strategy.
Rationale: Each subagent has independent `softRequestBudget` and `maxRuntimeMs` limits. There's no mechanism to say "this entire task tree has a budget of N tokens" and distribute it among children.
Evidence: Settings are per-subagent (`task.softRequestBudget`, `task.maxRuntimeMs`), no global budget counter found in codebase. `SingleResult` reports per-spawn `tokens` and `requests` but no aggregation logic beyond parent display.
Confidence: VERIFIED

#### 4.1.5 No Adaptive Agent Re-Assignment

Claim: Once spawned, an agent's type cannot change. If an agent encounters work outside its specialization, there is no mechanism to hand off to a more suitable agent type (only messaging can request collaboration).
Rationale: Agent type is fixed at spawn time. The only dynamic is `prewalk` hand-off (model upgrade from smol to full), not type reassignment. An agent encountering a sub-problem must either solve it itself, spawn a child, or message a peer.
Evidence: Agent definition is immutable after resolution in `#runSpawn`, no re-assignment or morphing mechanism found.
Confidence: INFERRED

### 4.2 Search Plane Gaps

#### 4.2.1 No Explicit Diversity Objective

Claim: OMP provides no mechanism to ensure candidate diversity across parallel explorations. Multiple agents may converge on the same solution approach without detection.
Rationale: When multiple agents are spawned in parallel (e.g., for a coding task), nothing prevents them from exploring identical approaches. There's no diversity injection (temperature variation, prompt perturbation, strategy constraints) or convergence detection.
Evidence: Batch spawning uses identical `context` for all items; per-item differentiation is only through different `task` instructions and optional different `agent` types. No diversity metric or deduplication exists.
Confidence: VERIFIED

#### 4.2.2 No Solution Comparison or Selection

Claim: When multiple isolated agents produce solutions (on separate branches), OMP has no built-in mechanism to compare, rank, or select the best candidate. The merge is sequential (first-come order), not quality-ordered.
Rationale: `mergeTaskBranches` cherry-picks branches in the order provided. There's no evaluation of branch quality, no Pareto comparison, no voting mechanism. The first branch that conflicts blocks all subsequent ones.
Evidence: `packages/coding-agent/src/task/worktree.ts:854` (sequential for-loop over branches, order-dependent), no comparison/ranking logic in the merge path.
Confidence: VERIFIED

#### 4.2.3 No Early Termination on Sufficiency

Claim: OMP cannot terminate a parallel search when one candidate is "good enough." All spawned agents run to completion (or their budget) regardless of sibling results.
Rationale: Background jobs are independent. While `hub cancel` can kill jobs, there's no automatic mechanism triggered by one agent succeeding that cancels siblings. The parent would need to orchestrate this manually.
Evidence: AsyncJobManager's cancel is per-id or all-or-nothing (`cancelAll`); no conditional/cross-job termination logic. Each background job's promise resolves independently.
Confidence: VERIFIED

#### 4.2.4 No Search Strategy Configuration

Claim: OMP lacks configurable search strategies (beam search, Monte Carlo tree search, evolutionary selection). The only search pattern is "spawn N, wait for all, merge first-fit."
Rationale: The architecture enables parallel exploration via batch spawning and isolation, but provides no higher-level search strategy primitives. The swarm extension's pipeline mode (iterative repetition) is the closest analog but is statically configured.
Evidence: No search strategy abstraction found across `packages/coding-agent/src/task/`, `packages/swarm-extension/src/swarm/`. Swarm `target_count` repeats the same DAG N times.
Confidence: VERIFIED

#### 4.2.5 No Inter-Candidate Communication During Search

Claim: Isolated agents cannot observe each other's progress during execution. There is no shared blackboard, no intermediate result broadcast, and no collaborative filtering.
Rationale: Isolated workspaces are independent CoW snapshots. While non-isolated agents share the filesystem, isolated agents are deliberately walled off. Hub messaging exists but is not designed for structured intermediate-result sharing during search.
Evidence: `packages/coding-agent/src/task/worktree.ts:411-444` (each isolation is independent), messaging is text-only (no structured intermediate-result protocol).
Confidence: VERIFIED

---

## 5. Proposal: Adaptation Strategies

### 5.1 Orchestration Plane Extensions

#### 5.1.1 Declarative Task DAG in Core

**Strategy:** Extend the task batch schema to support inter-item dependencies:

```typescript
interface TaskItem {
  name: string;
  agent?: string;
  task: string;
  isolated?: boolean;
  // NEW: Orchestration plane additions
  waits_for?: string[];  // sibling task names in same batch
  priority?: number;     // scheduling hint
}
```

**Integration point:** `packages/coding-agent/src/task/index.ts` — the spawn list resolution already happens in `resolveSpawnItems`. Adding dependency-aware scheduling would slot between resolution and the semaphore acquire. The swarm extension's `dag.ts` (Kahn's algorithm, wave builder) can be extracted into a shared utility.

**Rationale:** This unifies the swarm extension's DAG capability with the core task tool, enabling ad-hoc agent-orchestrated DAGs without YAML file definitions.

#### 5.1.2 Hierarchical Budget Allocator

**Strategy:** Introduce a `BudgetAllocator` that tracks aggregate token/request spend across a spawn tree and distributes budgets dynamically:

```typescript
interface BudgetAllocator {
  totalBudget: { tokens: number; requests: number; wallClockMs: number };
  allocate(agentId: string, priority: number): AgentBudget;
  report(agentId: string, usage: Usage): void;
  remaining(): Budget;
  redistribute(): void;  // Reclaim from finished agents
}
```

**Integration point:** The `runSubprocess` function in `packages/coding-agent/src/task/executor.ts` already receives and enforces `softRequestBudget` and `maxRuntimeMs`. The allocator would sit above this, dynamically computing per-agent budgets from the remaining global budget and redistributing when agents finish early.

**Rationale:** Prevents runaway cost from deep spawn trees. Enables "spend no more than X on this entire investigation" semantics.

#### 5.1.3 Context Pack Assembly

**Strategy:** Add a `ContextAssembler` that builds minimal, relevant context packs for each subagent based on declared information needs:

```typescript
interface ContextAssembler {
  // Declare what this task needs
  needs: ContextNeed[];  // e.g., "file:src/auth.ts", "schema:User", "docs:authentication"
  // Build the pack from available sources
  assemble(needs: ContextNeed[], budget: number): ContextPack;
}
```

**Integration point:** The `context` field in batch calls is already rendered into the system prompt. A context assembler would replace the manual context authoring with a declarative approach, using the existing `read` tool's summarization capability and workspace tree to select relevant material.

**Rationale:** Reduces the parent agent's burden of manually curating context. Ensures subagents receive precisely what they need without token waste.

#### 5.1.4 Dynamic Agent Capability Matching

**Strategy:** Extend agent discovery to support capability declarations and match tasks to agents by requirements:

```yaml
# Agent definition frontmatter
name: security-reviewer
capabilities: [security-audit, dependency-analysis, cve-lookup]
cost: high
latency: medium
```

**Integration point:** `packages/coding-agent/src/task/discovery.ts` already parses agent frontmatter. Adding `capabilities` as an indexed field enables the task tool to accept `requires: [capability]` instead of explicit `agent: name`, with the system selecting the best available agent.

**Rationale:** Decouples task specification from agent identity. Enables pluggable specialization without callers knowing the agent roster.

### 5.2 Search Plane Extensions

#### 5.2.1 Diversity-Aware Parallel Search

**Strategy:** Add a `SearchStrategy` abstraction that configures how parallel explorations are diversified:

```typescript
interface SearchStrategy {
  type: "beam" | "diverse" | "tournament" | "iterative-refinement";
  width: number;  // parallel candidates
  diversityConfig?: {
    temperatureRange: [number, number];
    strategyPrompts: string[];  // Different approach directives
    modelMix?: string[];  // Different models for diversity
  };
  terminationCriteria: TerminationCriteria;
}
```

**Integration point:** This sits above the task batch mechanism. The `SearchStrategy` generates the batch items with diversity injections (varied prompts, temperature, model assignments) and wraps the batch call with termination logic.

**Rationale:** Makes the currently implicit "spawn and hope" pattern explicit and configurable. Different problems benefit from different search strategies.

#### 5.2.2 Solution Evaluator & Selector

**Strategy:** Add an evaluation gate between candidate generation (isolated branches) and merge:

```typescript
interface CandidateEvaluator {
  // Score each candidate branch
  evaluate(branches: BranchCandidate[]): Promise<ScoredCandidate[]>;
  // Select which to merge (Pareto-optimal, threshold, voting)
  select(scored: ScoredCandidate[], criteria: SelectionCriteria): string[];
}
```

**Integration point:** Between the isolation workspace teardown (where patches/branches are captured) and `mergeTaskBranches`. Currently the executor returns `SingleResult` with exit code and output — extending this with a scoring step that runs deterministic checks (tests pass, linter clean, diff size) before merge.

**Rationale:** Prevents merging inferior solutions. Enables quality-gated progressive exposure where only passing candidates advance.

#### 5.2.3 Early Termination & Cross-Candidate Signaling

**Strategy:** Extend `AsyncJobManager` with conditional cancellation triggers:

```typescript
interface SearchJobGroup {
  jobs: string[];
  terminateWhen: TerminationCondition;
  // e.g., "first_success" | "n_of_m_succeed" | "quality_threshold"
  onTerminate: "cancel_remaining" | "let_finish" | "reduce_budget";
}
```

**Integration point:** The `AsyncJobManager` already tracks job completion and delivers results. Adding a `SearchJobGroup` wrapper that monitors settled jobs and conditionally cancels remaining ones would require minimal changes — the cancel mechanism already exists (`cancel(id)`, `cancelAll(filter)`).

**Rationale:** Prevents wasting compute on parallel explorations after one candidate already satisfies requirements.

#### 5.2.4 Shared Search State (Blackboard)

**Strategy:** Add a structured shared-state mechanism for inter-candidate communication during search:

```typescript
interface SearchBlackboard {
  // Candidates can post partial findings
  post(candidateId: string, finding: Finding): void;
  // Other candidates can read (if strategy allows)
  read(filter?: FindingFilter): Finding[];
  // Prevents duplicate exploration
  claim(territory: string): boolean;
}
```

**Integration point:** The `local://` protocol already provides shared filesystem access for non-isolated agents. For isolated agents, a new shared-state channel (implemented as a cross-workspace file or via the IRC bus with structured messages) would enable coordination without breaking isolation guarantees.

**Rationale:** Enables collaborative search where candidates build on each other's partial findings rather than duplicating work.

#### 5.2.5 Iterative Refinement Loop

**Strategy:** Extend the swarm extension's pipeline mode with evaluation-driven iteration:

```yaml
swarm:
  mode: iterative-refinement
  max_iterations: 10
  termination:
    metric: test_pass_rate
    threshold: 0.95
  agents:
    generator:
      role: implementer
      task: "Implement based on feedback in feedback.md"
    evaluator:
      role: test-runner
      task: "Run tests, write pass rate and failures to feedback.md"
      reports_to: [generator]
```

**Integration point:** The `PipelineController` in `packages/swarm-extension/src/swarm/pipeline.ts` already supports `target_count` iterations. Adding conditional termination based on an evaluation metric (read from a file or structured output) would transform blind repetition into goal-directed search.

**Rationale:** Many coding tasks benefit from generate-test-fix loops. Making this a first-class pattern eliminates manual orchestration of the feedback cycle.

---

## 6. Architecture Summary

```
Current OMP Orchestration Stack:
================================

┌─────────────────────────────────────────────────────┐
│  Swarm Extension (YAML DAG, waves, iterations)      │  ← Declarative DAG
├─────────────────────────────────────────────────────┤
│  Task Tool (batch spawn, agent types, context)      │  ← Core spawner
├─────────────────────────────────────────────────────┤
│  Hub (IRC messaging, job control, processes)        │  ← Coordination
├─────────────────────────────────────────────────────┤
│  AsyncJobManager (background, delivery, retention)  │  ← Job lifecycle
├─────────────────────────────────────────────────────┤
│  Semaphore + mapWithConcurrencyLimit                │  ← Concurrency
├─────────────────────────────────────────────────────┤
│  AgentRegistry + AgentLifecycleManager              │  ← State machine
├─────────────────────────────────────────────────────┤
│  pi-iso (APFS/btrfs/ZFS/overlay/projfs/rcopy)      │  ← Isolation
├─────────────────────────────────────────────────────┤
│  Git worktree/branch (patch capture, cherry-pick)   │  ← Merge
└─────────────────────────────────────────────────────┘

Proposed Nine-Plane Extensions:
================================

┌─────────────────────────────────────────────────────┐
│  Search Strategy (beam/diverse/tournament)          │  ← NEW: Search Plane
├─────────────────────────────────────────────────────┤
│  Candidate Evaluator & Selector                     │  ← NEW: Search Plane
├─────────────────────────────────────────────────────┤
│  Budget Allocator (global tree accounting)          │  ← NEW: Orchestration
├─────────────────────────────────────────────────────┤
│  Task DAG (inter-sibling deps in core)             │  ← NEW: Orchestration
├─────────────────────────────────────────────────────┤
│  Context Assembler (need-based pack building)       │  ← NEW: Orchestration
├─────────────────────────────────────────────────────┤
│  SearchJobGroup (conditional termination)           │  ← NEW: Search Plane
├─────────────────────────────────────────────────────┤
│  Blackboard (structured inter-candidate state)      │  ← NEW: Search Plane
└─────────────────────────────────────────────────────┘
```

### Key Insight

OMP's architecture already provides the **substrate** for both the Orchestration and Search planes. The primitives are there: parallel execution, workspace isolation, branch-based candidate management, messaging, lifecycle control. What's missing is the **policy layer** — the higher-level abstractions that encode orchestration strategies (DAG scheduling, budget allocation, context selection) and search strategies (diversity injection, quality evaluation, conditional termination). The adaptation path is additive: build policy layers on top of existing primitives rather than replacing infrastructure.
