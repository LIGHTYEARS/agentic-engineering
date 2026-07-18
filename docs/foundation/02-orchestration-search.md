# Orchestration & Search Planes — Confirmed Foundation

> Source: research/02 + rebuttals/02 + resolutions/02
> Funnel: 12 survived adversarial · 1 re-confirmed · 5 discarded · 0 open decision

## ✅ Confirmed buildable capabilities

### C-orch-1: Task fan-out with batch spawning
- **What it is**: The `task` tool dispatches a `{ context, tasks[] }` batch into independent background jobs via `AsyncJobManager`, bounded by a resizable `Semaphore` (default maxConcurrency=32, max 15 async jobs).
- **oh-my-pi substrate**: `packages/coding-agent/src/task/index.ts:854-879` — async spawns registered in loop; `packages/coding-agent/src/async/job-manager.ts` — job lifecycle registry
- **Provenance**: survived adversarial review

### C-orch-2: Hub/IRC mailbox coordination bus
- **What it is**: Process-global `IrcBus` providing peer-to-peer messaging between agents with delivery receipts, broadcast, and parked-agent revival on direct send; flat namespace via `AgentRegistry.listVisibleTo()`.
- **oh-my-pi substrate**: `packages/coding-agent/src/irc/bus.ts:1-33` — IrcBus with IrcMessage struct; `packages/coding-agent/src/registry/agent-registry.ts:170-174` — flat peer visibility
- **Provenance**: survived adversarial review

### C-orch-3: pi-iso CoW workspace isolation (8 backends)
- **What it is**: Native Rust PAL providing cross-platform copy-on-write workspace snapshots (APFS clonefile, btrfs, ZFS, Linux reflink, overlayfs kernel+fuse, ProjFS, Windows block-clone, recursive-copy fallback) with a resolution function that probes host capability.
- **oh-my-pi substrate**: `crates/pi-iso/src/lib.rs:46-66` — `BackendKind` enum; `crates/pi-iso/src/lib.rs:352-387` — resolve function with fallback chain
- **Provenance**: survived adversarial review

### C-orch-4: Resizable concurrency semaphore
- **What it is**: Abort-aware `Semaphore` class with runtime `resize()` that admits queued waiters on ceiling raise and drains naturally on lowering; plus `mapWithConcurrencyLimit` (fail-fast) and `mapWithConcurrencyLimitAllSettled` (all-settled) helpers.
- **oh-my-pi substrate**: `packages/coding-agent/src/task/parallel.ts:141-215` — Semaphore with acquire/release/resize; `packages/coding-agent/src/task/parallel.ts:26-84` — mapWithConcurrencyLimit
- **Provenance**: survived adversarial review

### C-orch-5: AsyncJobManager background job lifecycle
- **What it is**: Flat id→status registry managing background jobs with independent promise resolution, abort-controller-based cancellation, delivery-to-parent with retry queues, and ownerId-scoped cancel enforcement.
- **oh-my-pi substrate**: `packages/coding-agent/src/async/job-manager.ts:30-59` — AsyncJob struct (id, status, ownerId, abortController); `packages/coding-agent/src/async/job-manager.ts:260-268` — cancel with ownerId guard
- **Provenance**: survived adversarial review

### C-orch-6: Agent lifecycle state machine (park/revive)
- **What it is**: `AgentLifecycleManager` managing four states (running/idle/parked/aborted) with TTL-based parking to disk, on-demand revival on messaging, and concurrent park/revive coalescing.
- **oh-my-pi substrate**: `packages/coding-agent/src/registry/agent-lifecycle.ts:58-385` — full lifecycle manager; `packages/coding-agent/src/registry/agent-registry.ts:24` — AgentStatus type
- **Provenance**: survived adversarial review

### C-orch-7: Branch-mode isolation with git worktree merge
- **What it is**: Isolated agents commit to `omp/task/<id>` branches; `mergeTaskBranches` serializes cherry-picks via `git.withRepoLock` with stash/pop for working-tree safety; first conflict blocks subsequent branches.
- **oh-my-pi substrate**: `packages/coding-agent/src/task/worktree.ts:836-928` — mergeTaskBranches sequential cherry-pick; `packages/coding-agent/src/task/worktree.ts:731-817` — commitToBranch
- **Provenance**: survived adversarial review

### C-orch-8: Parallel candidate generation via batch + isolation
- **What it is**: Task batch fan-out + pi-iso isolation enables N agents to make conflicting filesystem changes without interference; each gets independent CoW snapshot; results collected as patches/branches.
- **oh-my-pi substrate**: `packages/coding-agent/src/task/worktree.ts:411-444` — ensureIsolation lifecycle; `packages/coding-agent/src/task/isolation-runner.ts:118-128` — writeIsolationPatch captures delta per agent
- **Provenance**: survived adversarial review

### C-orch-9: Resource-based termination (budget + wall-clock + abort)
- **What it is**: Multi-layered budget system: softRequestBudget (with 1.5x force-stop), maxRuntimeMs wall-clock timeout, max recursion depth=2, output size caps (500KB/5000 lines), and external abort signal.
- **oh-my-pi substrate**: `packages/coding-agent/src/config/settings-schema.ts:4160-4270` — all settings with defaults; `packages/coding-agent/src/task/executor.ts:1000-1028` — timeout enforcement
- **Provenance**: survived adversarial review

### C-orch-10: Agent role specialization via typed definitions
- **What it is**: Agents discovered from `.omp/agents/` directories with frontmatter-declared capabilities (tools, spawns, model, blocking behavior, output schema); task tool resolves agent type at spawn and enforces spawn policies.
- **oh-my-pi substrate**: `docs/task-agent-discovery.md:24-28` — AgentDefinition shape; bundled agents: scout, designer, reviewer, librarian, task, sonic
- **Provenance**: survived adversarial review

### C-orch-11: Context passing (context field + local:// + agent://)
- **What it is**: Three mechanisms for subagent context: (1) `context` string rendered into system prompt, (2) `local://` shared artifact filesystem, (3) `agent://` protocol for reading sibling outputs. Subagents do NOT inherit conversation history.
- **oh-my-pi substrate**: `packages/coding-agent/src/task/executor.ts:2496-2498` — systemPrompt template with context injection; `docs/tools/task.md:70-73` — agent:// and history:// protocols
- **Provenance**: survived adversarial review

### C-orch-12: Swarm DAG extension (separate execution path)
- **What it is**: YAML-defined multi-agent DAG with topological wave scheduling (Kahn's algorithm, cycle detection) and `Promise.all`-per-wave parallel execution; supports pipeline, sequential, and parallel modes.
- **oh-my-pi substrate**: `packages/swarm-extension/src/swarm/dag.ts:17-146` — buildDependencyGraph, detectCycles, buildExecutionWaves; `packages/swarm-extension/src/swarm/pipeline.ts:155-194` — wave execution
- **Scope limit**: Runs on separate `runSubprocess` path — shares NO runtime state with core task tool (no job manager, no hub, no registry, no isolation). Agents communicate only through shared filesystem. Cannot be trivially "extracted" into the task tool.
- **Provenance**: survived adversarial review

### C-orch-13: Post-hoc diff-collection for isolation+merge workflows
- **What it is**: pi-iso `diff()` returns structured `Diff{files: Vec<FileChange>}` per candidate (each `FileChange` carries `path`, `op: ChangeKind`, `diff: Option<String>`). Candidates run in isolation, diffs captured post-hoc before teardown, collected in parent scope. Parent holds ALL candidate patches/branches simultaneously after completion — this IS the shared state for quality-based selection/reordering before merge. No live cross-workspace channel needed.
- **oh-my-pi substrate**: `crates/pi-iso/src/diff.rs:29-57` — `Diff` struct with `Vec<FileChange>` and `unified_text()`; `crates/pi-iso/src/lib.rs:247-259` — async `diff()` trait method with default_diff; `packages/coding-agent/src/task/isolation-runner.ts:118-128` — writeIsolationPatch capturing delta per agent; `packages/coding-agent/src/task/isolation-runner.ts:250-303` — mergeIsolatedChanges as the architectural insertion point
- **Scope limit**: Post-hoc only — no live inter-candidate communication during execution. Candidates cannot observe each other's progress until all complete and diffs are collected in parent scope.
- **Provenance**: re-investigation CONFIRMED

## ❌ Must build net-new (gaps)

### G-orch-1: No declarative task DAG in core task tool
- **What's missing**: Core task tool batch items are independent (no `waits_for`, no inter-sibling dependencies). Cross-sibling coordination requires parent orchestration or ad-hoc hub messaging.
- **Evidence of absence**: `packages/coding-agent/src/task/types.ts:130-144` — TaskItem has only `name`, `agent`, `task`, `outputSchema`, `schemaMode`, `isolated`; no dependency field
- **Nature of build**: extension (new scheduling layer between resolveSpawnItems and job registration; cannot reuse swarm DAG execution model directly due to fire-and-forget vs blocking-wave incompatibility)

### G-orch-2: No cost-aware or priority-based scheduling
- **What's missing**: The semaphore counts slots uniformly — no weights, no priority queue, no cost model. A cheap scout and an expensive opus-model task consume one slot each.
- **Evidence of absence**: `packages/coding-agent/src/task/parallel.ts:141-215` — simple counting semaphore with no weight/priority parameters
- **Nature of build**: extension (weighted semaphore or priority queue wrapping the existing acquire/release)

### G-orch-3: No global/tree-level budget accounting
- **What's missing**: Per-subagent budgets only; no aggregate budget across a spawn tree. No "this entire task tree costs at most N tokens" with dynamic distribution.
- **Evidence of absence**: `packages/coding-agent/src/config/settings-schema.ts:4242-4258` — `task.softRequestBudget` is per-subagent, no global/tree aggregate; parent receives child usage only AFTER termination via `SingleResult`
- **Nature of build**: extension (new BudgetAllocator above executor; requires addressing the frozen-settings-snapshot constraint — child sessions cannot receive budget updates mid-run)

### G-orch-4: No query-driven context assembly
- **What's missing**: No mechanism to auto-select relevant context for a task. The `context` field is a manually authored string; `read` tool summarization is purely structural/AST-based (file-type-triggered, no relevance input).
- **Evidence of absence**: `crates/pi-ast/src/summary.rs:164-211` — `summarize_code()` has no query/relevance parameter; `packages/coding-agent/src/task/types.ts:130-144` — no `needs`/`contextNeeds` field; grep: zero matches for `contextAssembl`, `ContextPack`, `relevance.*select` in coding-agent/src
- **Nature of build**: greenfield subsystem (requires semantic index, query→relevance scoring, budget-aware selection algorithm, and new TaskItem schema field)

### G-orch-5: No solution comparison/ranking before merge
- **What's missing**: `mergeTaskBranches` cherry-picks in array-order; no quality evaluation, no Pareto comparison, no voting. First conflict blocks all subsequent.
- **Evidence of absence**: `packages/coding-agent/src/task/worktree.ts:854` — sequential for-loop over branches, order-dependent; no scoring/ranking logic in merge path
- **Nature of build**: extension (evaluator gate between diff-collection and merge — insertion point exists at `mergeIsolatedChanges` per C-orch-13, but evaluation logic is net-new)

### G-orch-6: No explicit diversity injection for parallel search
- **What's missing**: When N agents are spawned, nothing prevents convergence on same approach. No temperature variation, prompt perturbation, strategy constraints, or convergence detection.
- **Evidence of absence**: Batch spawning uses identical `context` for all items; per-item differentiation only via different `task` instructions and optional different `agent` types. No diversity metric or deduplication exists.
- **Nature of build**: extension (diversity-config layer above task batch that generates varied prompts/temperatures per spawn item)

### G-orch-7: No early-termination-on-sufficiency for parallel search
- **What's missing**: All spawned agents run to completion/budget regardless of sibling results. No automatic "one succeeded → cancel rest" mechanism. Parent must manually orchestrate via hub cancel.
- **Evidence of absence**: `packages/coding-agent/src/async/job-manager.ts:30-59` — no group abstraction, no completion-condition, no quality metric; jobs settle independently
- **Nature of build**: extension (SearchJobGroup abstraction with completion-watcher and conditional cancel; must also address the ownerId-scoping boundary)

### G-orch-8: No search strategy abstraction
- **What's missing**: No configurable search strategies (beam search, tournament, evolutionary). Only pattern is "spawn N, wait for all, merge first-fit."
- **Evidence of absence**: grep: zero matches for `SearchStrategy`, `beam`, `tournament` in `packages/coding-agent/src/task/`
- **Nature of build**: greenfield subsystem (strategy layer generating batch items with diversity config + wrapping with termination/selection logic)

## 🗑 Discarded overreach (do NOT re-propose)

- **DAG-in-core via `waits_for` on TaskItem "slots between resolution and semaphore acquire"** — REFUTED: Task tool dispatches all batch items into independent background jobs in a simple loop; each job acquires semaphore independently in its own closure. There is no centralized scheduling step where dependency logic could intervene. The swarm's wave-barrier model (`Promise.all` per wave) is architecturally incompatible with the task tool's fire-and-forget job registration. (`/tmp/oh-my-pi/packages/coding-agent/src/task/index.ts:854-879`)

- **Hierarchical/dynamic budget allocator with runtime redistribution** — REFUTED: Subagents run with frozen settings snapshots; `softRequestBudget` is captured into a `const` at run start and never re-queried. No back-channel exists for parent to update a running child's budget. Parent receives child usage data only AFTER termination. (`/tmp/oh-my-pi/packages/coding-agent/src/task/executor.ts:1000-1028`, `/tmp/oh-my-pi/packages/coding-agent/src/task/executor.ts:2104-2107`)

- **Early-termination via cross-agent cancel as "minimal changes"** — REFUTED: `cancel(id, filter?)` is explicitly ownerId-scoped; cross-agent cancellation is deliberately rejected (`if (filter?.ownerId && job.ownerId !== filter.ownerId) return false`). AsyncJobManager has no group abstraction, no completion-watcher hooks, no quality-aware termination. Building SearchJobGroup requires a new abstraction layer, not "minimal changes". (`/tmp/oh-my-pi/packages/coding-agent/src/async/job-manager.ts:260-268`)

- **Swarm DAG "extraction" into core task tool as shared utility** — REFUTED: Swarm runs on completely separate `runSubprocess` + `Promise.all` path sharing NO runtime state with task tool (no isolation, no hub, no job manager, no registry, no agent revival). The DAG algorithm is trivially extractable but the blocking-waves execution semantics it relies on are incompatible with the task tool's fire-and-forget async job model. (`/tmp/oh-my-pi/packages/swarm-extension/src/swarm/executor.ts:66-79`, `/tmp/oh-my-pi/packages/swarm-extension/src/swarm/pipeline.ts:155`)

- **Read-tool summarization → automated context assembly** — REFUTED: `summarize_code()` is a fixed structural transform triggered by file type/AST node kinds with zero query/relevance input. No `ContextPack`, no relevance-scoring, no budget-aware selection, no `needs` field in TaskItem. Automated context assembly is greenfield; the existing summarizer contributes at most a final compression stage. (`/tmp/oh-my-pi/crates/pi-ast/src/summary.rs:164-211`, `/tmp/oh-my-pi/packages/coding-agent/src/tools/read.ts:1998-2035`)

## ⚠ Open design decision

None.
