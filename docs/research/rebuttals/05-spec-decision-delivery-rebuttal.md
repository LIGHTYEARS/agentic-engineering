# Rebuttal: Spec, Decision, and Delivery Planes — OMP Mapping

## Summary

| Verdict | Count |
|---------|-------|
| REFUTED | 7 |
| UNVERIFIABLE | 3 |
| Survived scrutiny | 5 |

---

## Refutations

### 1. Plan File Constraint Frontmatter (Priority 1 Proposal — "Low Effort")

Target Claim/Proposal: "Extend the `local://<slug>-plan.md` format with structured frontmatter declaring hard constraints (MUST), soft constraints (SHOULD), and acceptance criteria (testable predicates)." (Section 6.1, Priority 1: "Low" effort)

Original Confidence: INFERRED (proposal, not a verified claim)

Verdict: REFUTED

Reason: Plan files in OMP are **plain markdown with no structured schema, no frontmatter parsing, and no validation**. The plan-mode system treats the file as an opaque markdown blob — it reads the content as a raw string (`Bun.file(resolved).text()`) and only ever extracts a title by regex-matching the first `# heading`. There is zero infrastructure for parsing YAML frontmatter from plan files, zero constraint schema, and zero validation pipeline. While OMP does have `parseFrontmatter()` for rules, skills, agents, and prompts, the plan-mode code path **never invokes it** on plan file content. The proposal's "Low effort" rating is misleading — it requires building: (a) a frontmatter parser integration for plan files, (b) a constraint schema definition, (c) constraint enforcement hooks, and (d) a propagation mechanism to TTSR rules. This is medium-to-high effort, not low.

Counter-Evidence:
- `/tmp/oh-my-pi/packages/coding-agent/src/plan-mode/plan-handoff.ts:28-36` — `resolveOverallPlanReference` reads plan content as raw text (`Bun.file(resolved).text()`) with no frontmatter parsing.
- `/tmp/oh-my-pi/packages/coding-agent/src/plan-mode/approved-plan.ts:165-166` — `resolveApprovedPlan` reads the plan via `input.readPlan(url)` and passes it to `finalizeApprovedPlan` which only calls `resolvePlanTitle` (heading extraction), never `parseFrontmatter`.
- `/tmp/oh-my-pi/packages/coding-agent/src/prompts/system/plan-mode-active.md:7` — The plan file is described as markdown (`local://<slug>-plan.md`); no structured frontmatter is mentioned in the model-facing prompt.

---

### 2. Checkpoint/Rewind as "Side-Effect-Free Decision Exploration"

Target Claim/Proposal: "The checkpoint/rewind mechanism implements a 'try-then-decide' pattern where the agent can investigate an approach, capture findings in a report, then rewind all state changes — making it a side-effect-free decision exploration tool." (Section 3.1) and "Pareto frontier via checkpoint fan-out" (Section 3.3)

Original Confidence: VERIFIED

Verdict: REFUTED

Reason: Checkpoint/rewind is explicitly **NOT side-effect-free**. The rewind tool documentation and source are unambiguous: rewind restores only the **conversation message prefix** — it does NOT restore filesystem contents, git state, artifacts, blob-store payloads, or any other external state. If the agent writes files, creates commits, or makes HTTP calls during the checkpoint window, those changes persist after rewind. The proposal's characterization as "pure speculative execution for decision-making" and "side-effect-free" is factually wrong. The "Pareto frontier via checkpoint fan-out" pattern (checkpoint → try A → rewind → checkpoint → try B → rewind → compare) would leave filesystem artifacts from approach A contaminating approach B unless the agent manually cleans up — which the proposal never addresses.

Counter-Evidence:
- `/tmp/oh-my-pi/docs/tools/rewind.md:72` — "Rewind restores only the message prefix recorded by `checkpointMessageCount`; there is no file restore, artifact restore, blob restore, or process restore path."
- `/tmp/oh-my-pi/docs/tools/rewind.md:87-93` — "Not restored: filesystem contents; git state; artifacts under `packages/coding-agent/src/session/artifacts.ts`; blob-store payloads..."
- `/tmp/oh-my-pi/docs/tools/rewind.md:94` — "There is no concurrent-edit reconciliation. If code or session-adjacent state changes during the checkpoint window, rewind does not merge or revert them; it only drops conversation context and rewires the session branch."

---

### 3. Checkpoint Enabled by Default for "Speculation"

Target Claim/Proposal: Checkpoint/rewind is proposed as a key mechanism for both Decision Plane (speculation, Pareto evaluation) and Delivery Plane (rollback). The proposal lists it as "existing primitives" ready for use.

Original Confidence: VERIFIED (the mechanism exists)

Verdict: REFUTED (as an "existing primitive" ready for use)

Reason: Checkpoint/rewind is **disabled by default** (`checkpoint.enabled` defaults to `false`). It must be explicitly opted-in via settings. The proposal presents it as a readily available primitive without noting this critical gate. Any framework building on checkpoint/rewind must first solve the enablement problem — users who haven't toggled the setting cannot use the Pareto or speculation patterns.

Counter-Evidence:
- `/tmp/oh-my-pi/packages/coding-agent/src/config/settings-schema.ts:3666-3675` — `"checkpoint.enabled": { type: "boolean", default: false, ... description: "Enable the checkpoint and rewind tools for context checkpointing" }`

---

### 4. ADR Auto-Generation from Ask Tool

Target Claim/Proposal: "When the `ask` tool completes, emit a structured `custom_message` entry with type `"decision-record"` containing: context, options considered, selected option, rationale, and consequences. These can be extracted into `docs/adr/` files." (Section 3.3, and Section 6.2: "Ask tool + custom_message = ADR")

Original Confidence: INFERRED

Verdict: REFUTED

Reason: The `ask` tool does **not** emit any `custom_message` entry after completion. It returns a standard `toolResult()` with structured `details` (selectedOptions, customInput, timedOut) — but this result stays in the tool result entry of the session transcript. There is no `appendCustomMessageEntry` call, no "decision-record" emission, and no hook that fires post-ask to create an ADR. The proposal conflates "the ask tool returns structured details" with "the ask tool emits a custom_message that can be extracted." These are architecturally different: tool results are part of the assistant turn and do not independently persist as extractable custom entries. Building this requires either: (a) modifying the ask tool to emit a side-channel custom_message, or (b) building an extension/hook that intercepts `tool_result` events for ask calls. Neither exists.

Counter-Evidence:
- `/tmp/oh-my-pi/packages/coding-agent/src/tools/ask.ts` — Full file search for `custom_message`, `customMessage`, `appendCustom`, `emit` yields zero hits related to decision-record emission. The only `emit` references are to the TUI gap-rendering helper (`emitGap`).
- `/tmp/oh-my-pi/packages/coding-agent/src/tools/ask.ts:8` — The tool's purpose is "Get decisions on implementation choices" but its output path is purely `toolResult()` → structured `details` → no custom_message side-channel.

---

### 5. Todo Tool "Acceptance Evidence" Extension

Target Claim/Proposal: "Extend the `todo` tool's `done` op to accept an optional `evidence` field — a reference to test output, bash result, or artifact that proves the acceptance criterion. Store in `details.completedTasks[].evidence`." (Section 2.2, point 3)

Original Confidence: INFERRED

Verdict: REFUTED

Reason: The `TodoItem` interface is `{ content: string; status: TodoStatus }` — there is no `evidence` field and no extension point for per-task metadata. The `done` op simply flips `task.status = "completed"` (line 449). The `TodoCompletionTransition` type is `{ phase: string; content: string }` — again no evidence field. The `completedTasks` array in `TodoToolDetails` records only which tasks transitioned, not any verification data. Adding an `evidence` field requires modifying: (a) the `TodoItem` interface, (b) the `todoSchema` (arktype), (c) the `done` case handler, (d) the `TodoCompletionTransition` type, (e) session serialization, and (f) the TUI renderer. This is not a minor extension — it touches the core data model of one of OMP's most-used tools.

Counter-Evidence:
- `/tmp/oh-my-pi/packages/coding-agent/src/tools/todo.ts:25-28` — `TodoItem { content: string; status: TodoStatus }` — no evidence field.
- `/tmp/oh-my-pi/packages/coding-agent/src/tools/todo.ts:35-38` — `TodoCompletionTransition { phase: string; content: string }` — no evidence field.
- `/tmp/oh-my-pi/packages/coding-agent/src/tools/todo.ts:447-449` — `case "done": { for (const task of getTaskTargets(phases, entry, errors)) { task.status = "completed"; } }` — no evidence parameter handling.

---

### 6. Split Commit as "Progressive Delivery" / "One PR per Commit Group"

Target Claim/Proposal: "Implement a `split_delivery` operation (parallel to `split_commit`) that decomposes a large change into ordered PRs with dependency tracking, merging each only after its predecessor's CI passes." (Section 4.3, point 2) and "The `split_commit` tool with `dependencies` field and topological sorting already implements ordered, atomic delivery of related changes. Extend to create one PR per commit group with inter-PR dependencies." (Section 6.3)

Original Confidence: VERIFIED (split_commit exists with topo-sort)

Verdict: REFUTED (the "progressive delivery" extension claim)

Reason: The split_commit tool creates **local git commits**, not PRs. It has no GitHub integration whatsoever — the entire commit agent pipeline operates at the git level (`git.commit`, `git.stage.hunks`, optional `git.push`). The `dependencies` field and topo-sort determine **commit ordering**, not PR dependency chains. There is zero infrastructure for: (a) creating multiple PRs from a split plan, (b) tracking inter-PR dependencies, (c) monitoring predecessor CI before merging successors, or (d) any GitHub API interaction in the commit agent path. The proposal's claim that extending split_commit to "one PR per commit group" is "Medium" effort dramatically underestimates the work — it requires building an entirely new orchestration layer that bridges the commit agent (local git operations) with the github tool (remote operations), plus a dependency-aware merge-when-ready system that doesn't exist anywhere in OMP.

Counter-Evidence:
- `/tmp/oh-my-pi/packages/coding-agent/src/commit/agentic/index.ts:239-244` — `runSingleCommit` calls `git.commit(ctx.cwd, commitMessage)` then optionally `git.push(ctx.cwd)` — no PR creation.
- `/tmp/oh-my-pi/packages/coding-agent/src/commit/agentic/index.ts:295-314` — The split commit executor iterates `order` and creates git commits with `git.commit(ctx.cwd, message)`, then does a single `git.push(ctx.cwd)` at the end — no per-group PR creation.
- `/tmp/oh-my-pi/packages/coding-agent/src/commit/agentic/topo-sort.ts:1-44` — `computeDependencyOrder` returns a flat number array (execution order) — it has no concept of PR dependencies, merge gates, or CI status.

---

### 7. "Delivery Signal" Extension Event

Target Claim/Proposal: "Add a `delivery_signal` event to the extension API that fires when `run_watch` detects state changes. Extensions can then implement custom promotion/rollback logic." (Section 4.3, point 4)

Original Confidence: INFERRED

Verdict: REFUTED

Reason: No `delivery_signal` event exists in OMP's extension event system. The extension API events are strictly session-lifecycle events (`session_before_compact`, `session_before_tree`, `session_before_branch`, `session_before_switch`, `session_shutdown`, `session_stop`, `session.compacting`, `agent_start`, `agent_end`, `tool_execution_start`, `tool_execution_end`, `tool_result`, `message_start`, `message_update`, `message_end`). The `run_watch` operation in the github tool is a pure polling loop that returns a final text result — it has no event emission mechanism, no hook integration, and no extension callback point. The tool doesn't emit `onUpdate` to extensions; it emits streaming updates to the TUI renderer only. Building this requires: (a) defining a new event type in the extension API, (b) wiring the github tool's internal polling loop to the extension event bus, (c) defining a structured payload for CI state transitions. This is a significant architectural addition, not a simple extension.

Counter-Evidence:
- `/tmp/oh-my-pi/packages/coding-agent/src/extensibility/extensions/runner.ts:596-604` — The runner's `#isSessionBeforeEvent` check exhaustively lists the "before" events: `session_before_switch`, `session_before_branch`, `session_before_compact`, `session_before_tree` — no delivery/run_watch events.
- `/tmp/oh-my-pi/packages/coding-agent/src/extensibility/shared-events.ts:64-65` — Session events defined: `session_before_compact` is the closest infrastructure event — there is no CI/delivery event type.
- `/tmp/oh-my-pi/docs/tools/github.md:55` — "`run_watch` is the only streaming op. It emits `onUpdate` snapshots while polling, then returns one final text result." — Updates go to the TUI, not the extension event bus.

---

## Unverifiable Claims

### 1. Cross-Plane Traceability

Target Claim/Proposal: "Spec→Decision: Constraints inform option generation. Decision→Delivery: ADRs gate delivery scope. Delivery→Spec: Runtime signals update acceptance criteria status." (Section 6.4 Integration Architecture)

Original Confidence: INFERRED

Verdict: ? UNVERIFIABLE

Reason: These are pure aspirational architecture without any mechanism specification. None of the three cross-plane flows has even a sketch of how it would work in OMP. There is no "constraint→option generation" pipeline, no "ADR gates delivery" mechanism, and no "runtime signal→acceptance criteria" feedback loop. The proposal acknowledges these as gaps but then presents them in an architecture diagram as if they're part of a coherent design — when they're actually entirely unspecified hand-waves.

---

### 2. Decision Memory Across Sessions via Memory Pipeline

Target Claim/Proposal: "Leverage the OMP memory pipeline to retain decision patterns. When an `ask` tool with similar option shapes appears, inject prior decisions as context for the `recommended` field." (Section 3.3, point 3)

Original Confidence: INFERRED

Verdict: ? UNVERIFIABLE

Reason: The OMP memory pipeline (mnemopi/hindsight) operates on agent turn boundaries (`agent_start`/`agent_end`) and extracts durable facts automatically — there is no mechanism to: (a) detect "similar option shapes" across sessions, (b) retrieve prior ask-tool decisions by structural similarity, or (c) inject that context into the `recommended` field of a future ask call. The memory system stores free-text facts, not structured decision records with option-shape signatures. The proposal assumes a retrieval capability that the memory system was never designed to provide.

---

### 3. TTSR Auto-Generation from Constraints

Target Claim/Proposal: "Auto-generate `.omp/rules/` from the plan's hard constraints. Example: constraint 'no new dependencies' → rule with `condition: \"tool:bash.*install|tool:write.*package.json\"`." (Section 6.1, Priority 5: "Medium" effort)

Original Confidence: INFERRED

Verdict: ? UNVERIFIABLE

Reason: While TTSR rules exist and support `condition:` regex patterns, there is no mechanism to programmatically generate rules from plan content at runtime. Rules are discovered from disk (`.omp/rules/*.md`) via the discovery/builtin pipeline. There is no API for an agent to dynamically register TTSR rules mid-session without writing files. The agent could write `.omp/rules/*.md` files using the `write` tool, but: (a) those rules are loaded at session init, not dynamically mid-session, and (b) there's no "constraint→regex" translator. The pattern is theoretically possible but requires the agent to manually write rule files and restart/reload — far from the automated pipeline the proposal implies.

---

## Missing Coverage

1. **Session persistence and compaction interaction with Spec Plane**: The proposal never addresses how plan-file constraints would survive compaction. Completed/abandoned todo tasks are stripped on session resume (`docs/tools/todo.md` line 152). If acceptance criteria are tracked as todo tasks, they disappear on resume — breaking the living-spec model.

2. **Concurrency model**: The proposal never addresses that `ask` has `concurrency = "exclusive"` — only one ask can run at a time. The "structured multi-option selection" cannot be parallelized for multi-criteria Pareto evaluation. Multiple dimensions must be evaluated sequentially.

3. **`run_watch` failure semantics vs. "canary"**: The proposal equates draft-PR + run_watch with canary deployment, but `run_watch` monitors CI for a **specific commit**, not a deployment to users. CI passing says nothing about runtime behavior in production. The "canary" analogy is a category error — CI is pre-merge verification, not post-deploy monitoring.

4. **Release script is not an agent tool**: The proposal treats `scripts/release.ts` as if it's an interactive agent-accessible primitive, but it's a standalone CLI script invoked via `bun scripts/release.ts`. It's not exposed as a tool, not callable from the agent session, and has no integration with the extension/hook system. The agent can invoke it via `bash`, but that gives no structured feedback or integration.

5. **Autoresearch isolation != deployment isolation**: The proposal maps autoresearch's git-reset pattern to "deployment experiments" but autoresearch operates on a dedicated `autoresearch/*` branch with hard reset to a known baseline. Production deployments don't have this luxury — you can't `git reset --hard` a running service.

---

## Survived Scrutiny

### 1. Todo as Hierarchical Requirement Register

The proposal's claim that the todo tool provides phase-based task decomposition is accurate. The `TodoPhase[]` structure with named phases and ordered tasks does map naturally to requirement tiers. Source: `/tmp/oh-my-pi/packages/coding-agent/src/tools/todo.ts:30-33`.

### 2. Plan Mode as Spec Authoring with Approval Gate

The proposal correctly identifies plan mode as the closest mechanism to formal spec authoring. The read-only tool restriction, explicit approval via `xd://propose`, and three approval paths (execute/compact+execute/keep context) are accurately described. Source: `/tmp/oh-my-pi/packages/coding-agent/src/prompts/system/plan-mode-active.md:3-9`.

### 3. Session Branching as Decision Exploration

The proposal's characterization of `/tree` (same-file navigation) and `/branch` (new session file) as decision-exploration mechanisms is accurate. Branching preserves abandoned paths via `branchWithSummary`, and the branch summaries survive in LLM context. Source: `/tmp/oh-my-pi/docs/session-tree-plan.md:14-16`.

### 4. GitHub Tool PR + run_watch as Delivery Operations

The proposal correctly identifies that `pr_create` supports `draft: true` and `run_watch` monitors CI. Both operations exist with the described fields. The chaining pattern (draft PR → run_watch → promote via bash) is technically feasible with existing primitives. Source: `/tmp/oh-my-pi/docs/tools/github.md:30-31` (draft field), `/tmp/oh-my-pi/docs/tools/github.md:204-221` (run_watch).

### 5. `pr://` Internal URL Protocol

The proposal mentions `read pr://` and the proposal's evidence about `pr://` reading is upheld. The internal URL protocol correctly supports `pr://<N>`, `pr://<owner>/<repo>/<N>`, and diff variants. Source: `/tmp/oh-my-pi/packages/coding-agent/src/internal-urls/issue-pr-protocol.ts:1-16`.
