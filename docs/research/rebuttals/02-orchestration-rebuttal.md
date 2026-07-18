# Rebuttal: Orchestration & Search Planes OMP Mapping

## Summary

- **Proposals refuted:** 4
- **Proposals unverifiable:** 2
- **Survived scrutiny:** The core inventory claims (§1) are largely accurate; the gap analysis (§4) is honest. The refutations below target the **adaptation strategies** (§5) which overstate integration feasibility and undercount architectural barriers.

---

## Refutations

### 1. Declarative Task DAG in Core (§5.1.1) — Integration Point Claim is Misleading

**Target Claim/Proposal:** "Adding dependency-aware scheduling would slot between resolution and the semaphore acquire. The swarm extension's `dag.ts` (Kahn's algorithm, wave builder) can be extracted into a shared utility."

**Original Confidence:** VERIFIED (§5.1.1 integration point)

**Verdict:** ✗ REFUTED

**Reason:** The proposal assumes that adding `waits_for` to `TaskItem` would "slot between resolution and the semaphore acquire" — but this fundamentally misunderstands the task tool's execution model. The task tool dispatches ALL batch items into independent background jobs immediately after resolution. Each job individually acquires the semaphore in its own closure. There is no centralized scheduling step between `resolveSpawnItems()` and job registration where dependency logic could intervene — jobs are registered in a loop (`for (const spawn of asyncSpawns)`) and each job's body independently calls `this.#getSpawnSemaphore().acquire()`.

Furthermore, the swarm extension's DAG system operates at a fundamentally different abstraction level: it runs entire `runSubprocess` calls sequentially in waves via `Promise.all` per wave (pipeline.ts:155). The task tool's async model registers independent jobs that each settle their own promises. Merging these two models would require replacing the task tool's fire-and-forget job registration with a wave-based barrier synchronization — a rewrite of the execution path, not a "slot between" insertion.

**Counter-Evidence:**
- `/tmp/oh-my-pi/packages/coding-agent/src/task/index.ts:854-879` — async spawns are registered in a simple loop with no scheduling intermediary; each job body acquires the semaphore independently
- `/tmp/oh-my-pi/packages/swarm-extension/src/swarm/pipeline.ts:155-194` — wave execution uses `Promise.all` blocking on all agents in a wave before proceeding; incompatible with the task tool's fire-and-forget model
- `/tmp/oh-my-pi/packages/coding-agent/src/task/types.ts:130-144` — `TaskItem` interface has only `name`, `agent`, `task`, `outputSchema`, `schemaMode`, `isolated`; no dependency field exists

---

### 2. Hierarchical Budget Allocator (§5.1.2) — Contradicts Subagent Isolation Design

**Target Claim/Proposal:** "The allocator would sit above this, dynamically computing per-agent budgets from the remaining global budget and redistributing when agents finish early."

**Original Confidence:** VERIFIED (§5.1.2 integration point)

**Verdict:** ✗ REFUTED

**Reason:** The proposal assumes a global budget allocator can dynamically adjust running subagents' budgets. However, subagents run with an **isolated settings snapshot** that is frozen at spawn time. The executor explicitly creates child sessions with `async.enabled = false` and `bash.autoBackground.enabled = false` forced. The `softRequestBudget` is read once during the budget enforcement setup and never re-queried from settings during execution.

More critically, `runSubprocess` captures `softRequestBudget` into a local `const` at the start of the run (via `settings.get("task.softRequestBudget")`) and the budget enforcement closure references that captured value for the entire subagent lifetime. There is no mechanism to "redistribute" budget to a running subagent — the child has no back-channel to receive updated budget allocations from its parent. The only budget interaction is the one-directional abort signal.

The proposal's "redistribute when agents finish early" requires a live bidirectional control channel between parent and child budget tracking — something that does not exist. The parent only receives the child's usage data AFTER the child has already terminated (in `SingleResult.tokens` and `SingleResult.requests`).

**Counter-Evidence:**
- `/tmp/oh-my-pi/packages/coding-agent/src/task/executor.ts:1000-1028` — `maxRuntimeMs` captured once, `softRequestBudget` enforced via closure-captured value with no dynamic update mechanism
- `/tmp/oh-my-pi/packages/coding-agent/src/config/settings-schema.ts:4242-4258` — `task.softRequestBudget` is a static per-subagent number (default 200); no global/tree-level aggregate
- `/tmp/oh-my-pi/packages/coding-agent/src/task/executor.ts:2104-2107` — `runSubprocess` receives options including settings as an immutable snapshot; child sessions cannot observe runtime budget changes from the parent

---

### 3. Early Termination & Cross-Candidate Signaling (§5.2.3) — "Minimal changes" is False

**Target Claim/Proposal:** "Adding a `SearchJobGroup` wrapper that monitors settled jobs and conditionally cancels remaining ones would require minimal changes — the cancel mechanism already exists (`cancel(id)`, `cancelAll(filter)`)."

**Original Confidence:** VERIFIED (§5.2.3 integration point)

**Verdict:** ✗ REFUTED

**Reason:** The proposal claims "minimal changes" by pointing at `cancel(id)` and `cancelAll(filter)`. But `cancel` is scoped by `ownerId` — a parent can only cancel jobs IT registered. Cross-agent cancellation is explicitly rejected at the manager level (`if (filter?.ownerId && job.ownerId !== filter.ownerId) return false`). This is a deliberate security boundary, not an incidental limitation.

More fundamentally, `AsyncJobManager` has no concept of job groups, completion callbacks that trigger conditional logic on sibling jobs, or quality-aware termination. It is a simple registry with individual job promises. The `onJobComplete` callback fires for delivery to the parent conversation (injecting the result as a message) — it is NOT a hook point for conditional orchestration logic. The delivery system uses retry queues with exponential backoff for failed deliveries to a busy parent — it is designed for reliable one-shot message injection, not for real-time monitoring and conditional cancellation.

A `SearchJobGroup` that monitors completions and conditionally cancels siblings would need to:
1. Add a group abstraction to `AsyncJobManager` (which has none)
2. Add a completion-watcher that runs before delivery (not after)
3. Add a quality evaluation step between job completion and the cancellation decision
4. Override the ownerId-scoping that currently prevents cross-job interference

This is not "minimal changes" — it requires a new abstraction layer inside the job manager.

**Counter-Evidence:**
- `/tmp/oh-my-pi/packages/coding-agent/src/async/job-manager.ts:260-268` — `cancel(id, filter?)` explicitly rejects cross-agent cancellation: `if (filter?.ownerId && job.ownerId !== filter.ownerId) return false`
- `/tmp/oh-my-pi/packages/coding-agent/src/async/job-manager.ts:30-59` — `AsyncJob` has no group field, no completion condition, no quality metric; it's a flat id→status registry
- `/tmp/oh-my-pi/packages/coding-agent/src/async/job-manager.ts:599-612` — `#enqueueDelivery` fires unconditionally on completion with no hook for conditional logic before delivery

---

### 4. Swarm DAG "Extraction" into Core — Architectural Mismatch

**Target Claim/Proposal:** "The swarm extension's `dag.ts` (Kahn's algorithm, wave builder) can be extracted into a shared utility" (§5.1.1) and the proposal treats swarm as proving that DAG execution is feasible in the task tool.

**Original Confidence:** VERIFIED (implicit throughout §2.1 and §5.1.1)

**Verdict:** ✗ REFUTED

**Reason:** The swarm extension's execution model is fundamentally different from the task tool's. The swarm calls `runSubprocess` directly (not via the task tool's `AsyncJobManager` path). Swarm agents:
1. Do NOT use isolation (`executor.ts` passes no `isolated` option)
2. Do NOT use hub messaging (no IRC bus interaction; agents communicate only through shared filesystem)
3. Do NOT register as async jobs (no `AsyncJobManager` integration)
4. Do NOT appear in the parent's agent registry (they use synthetic agent IDs like `swarm-<name>-<agent>-<iteration>`)
5. Are NOT revivable or messageable after completion

The swarm's DAG model requires all agents in a wave to complete before the next wave starts (`Promise.all` on wave agents → sequential wave loop). This blocking-waves model cannot be layered atop the task tool's fire-and-forget job model without fundamentally changing one or both systems. The proposal's framing of "extraction into shared utility" obscures this: the algorithm is trivially extractable, but the execution semantics it relies on (blocking wave barriers over `runSubprocess`) are incompatible with async job registration.

**Counter-Evidence:**
- `/tmp/oh-my-pi/packages/swarm-extension/src/swarm/executor.ts:66-79` — `runSubprocess` called directly with `cwd: workspace`, no isolation, no job manager, no hub
- `/tmp/oh-my-pi/packages/swarm-extension/src/swarm/pipeline.ts:155` — `await Promise.all(wave.map(...))` — blocking barrier that is incompatible with async job fire-and-forget
- `/tmp/oh-my-pi/packages/swarm-extension/README.md:397-401` — "The orchestrator starts and stops agents in the right order. It does NOT pass data between them. Agents communicate through files in the shared workspace."

---

### 5. Shared Search State / Blackboard via local:// (§5.2.4) — Contradicts Isolation Semantics

**Target Claim/Proposal:** "The `local://` protocol already provides shared filesystem access for non-isolated agents. For isolated agents, a new shared-state channel (implemented as a cross-workspace file or via the IRC bus with structured messages) would enable coordination without breaking isolation guarantees."

**Original Confidence:** VERIFIED (§5.2.4)

**Verdict:** ? UNVERIFIABLE

**Reason:** The proposal simultaneously claims to preserve isolation guarantees while introducing cross-workspace shared state. These are contradictory by definition. The pi-iso isolation contract is explicit: `merged` is a writable view that is independent of other workspaces. The `IsolationBackend` trait (start/stop/diff) has no concept of a shared channel between isolated instances.

For non-isolated agents, `local://` does provide shared access. But for isolated agents (which is the interesting case for search-plane diversity), there is no mechanism in `pi-iso` or the worktree lifecycle to create a cross-workspace communication channel. The IRC bus exists but:
1. Messages are unstructured text (no schema enforcement, no structured intermediate-result protocol)
2. The mailbox has a hard cap of 100 messages per agent (`MAILBOX_CAP`)
3. Isolated agents are torn down at completion and explicitly marked non-revivable

The proposal's suggestion of "a cross-workspace file" would require modifying the isolation setup to mount a shared directory — which breaks the CoW isolation invariant. The IRC-based alternative is technically possible but the proposal provides no evidence that text-only messaging with 100-message caps constitutes a viable "blackboard" for structured intermediate results during parallel search.

**Counter-Evidence:**
- `/tmp/oh-my-pi/crates/pi-iso/src/lib.rs:225-260` — `IsolationBackend` trait has `start(lower, merged)` / `stop(merged)` / `diff(lower, merged)` — no concept of shared channels between instances
- `/tmp/oh-my-pi/docs/tools/hub.md:110` — mailbox cap is 100 messages per agent
- `/tmp/oh-my-pi/docs/tools/task.md:41` — "Isolated agents are torn down at completion — not revivable"

---

### 6. Context Pack Assembly (§5.1.3) — No Foundation in Existing Code

**Target Claim/Proposal:** "A context assembler would replace the manual context authoring with a declarative approach, using the existing `read` tool's summarization capability and workspace tree to select relevant material."

**Original Confidence:** VERIFIED (§5.1.3)

**Verdict:** ? UNVERIFIABLE

**Reason:** The proposal references a "read tool's summarization capability" as a foundation for automated context assembly. However, the `read` tool's structural summarization (declarations-only elision) is a fixed behavior triggered by file type, not a query-driven relevance selector. There is no "summarization API" that takes a need declaration and returns relevant context.

The proposal's `ContextAssembler` interface assumes:
1. Subagents can declare `needs: ContextNeed[]` — no such field exists in `TaskItem` or `AgentDefinition`
2. A `budget`-aware assembly function can select material — no token-budget-aware file selection exists anywhere in the codebase
3. This replaces the `context` string field — but `context` is validated as non-empty in batch mode; replacing it with a declarative mechanism would break the batch validation contract

The "existing workspace tree" is indeed passed to subagents, but it is a flat file listing, not a queryable semantic index. The proposal presents this as if there's infrastructure to build on, but it would be a greenfield system.

**Counter-Evidence:**
- `/tmp/oh-my-pi/packages/coding-agent/src/task/types.ts:130-144` — `TaskItem` has no `needs` or `contextNeeds` field
- `/tmp/oh-my-pi/packages/coding-agent/src/task/index.ts:258-259` — batch validation requires `context` to be a non-empty string: `if (typeof params.context !== "string" || params.context.trim() === "")`
- No grep results for `contextAssembl`, `ContextPack`, `contextNeed`, `contextSelect` anywhere in the codebase

---

## Missing Coverage

The original proposal fails to analyze these subsystems that would constrain or invalidate its proposed extensions:

1. **Settings isolation boundary:** Subagent sessions receive frozen settings snapshots. Any "dynamic budget redistribution" or "runtime strategy adjustment" proposal must contend with the fact that child settings are immutable after spawn. The proposal never mentions this constraint.

2. **Process model:** The proposal never addresses that the task tool forces `async.enabled = false` in children. This means subagents cannot themselves spawn background jobs or participate in async coordination patterns. The "SearchJobGroup" proposal implicitly requires children to report intermediate quality signals back — but children have no async output channel to their parent until they terminate.

3. **Agent Registry scoping:** The proposal references `AgentRegistry.listVisibleTo()` (flat namespace, no hierarchy) but doesn't address that visibility is process-scoped. In a distributed or multi-process deployment, the registry provides no coordination surface.

4. **Swarm extension's complete isolation from the core task tool:** The proposal treats swarm and core-task as complementary layers but never acknowledges they share NO runtime state (no registry entries, no job manager, no hub messaging). They are entirely separate execution paths that happen to share `runSubprocess`.

5. **Merge ordering fragility:** The proposal's search-plane assumes branch-isolation enables "speculative parallel execution without merge until explicit selection." But `mergeTaskBranches` cherry-picks branches in the order of the `branches` array parameter — the caller controls the order, and the first conflict blocks all subsequent branches. The proposal never addresses how a quality-based selection (§5.2.2 `CandidateEvaluator`) would reorder branches BEFORE the merge, given that the merge function receives a pre-ordered array.

---

## Survived Scrutiny

### Core Inventory (§1) — Accurate
The technical inventory of existing mechanisms (§1.1–§1.7) is well-sourced and accurately represents the code. The pi-iso backends, the task tool's batch semantics, the hub messaging model, the semaphore, and the agent lifecycle are all correctly described.

**Upholding source:**
- `/tmp/oh-my-pi/crates/pi-iso/src/lib.rs:46-66` — BackendKind enum matches the proposal's list
- `/tmp/oh-my-pi/packages/coding-agent/src/task/parallel.ts:141-215` — Semaphore matches described behavior
- `/tmp/oh-my-pi/packages/swarm-extension/src/swarm/dag.ts:17-146` — DAG operations match described behavior

### Gap Analysis (§4) — Honest
The gap analysis sections (§4.1–§4.2) correctly identify what is missing. The claims about no declarative task DAG in core (§4.1.1), no cost-aware scheduling (§4.1.2), no global budget (§4.1.4), no diversity objective (§4.2.1), no early termination (§4.2.3) are all accurate and well-evidenced.

**Upholding source:**
- `/tmp/oh-my-pi/packages/coding-agent/src/task/types.ts:130-144` — TaskItem confirms no dependency fields
- `/tmp/oh-my-pi/packages/coding-agent/src/task/parallel.ts:141-215` — Semaphore confirms no weights/priorities
- `/tmp/oh-my-pi/packages/coding-agent/src/async/job-manager.ts:260-268` — No conditional cross-job termination logic
