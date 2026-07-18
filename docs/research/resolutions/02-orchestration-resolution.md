# Resolution: Orchestration Rebuttal — UNVERIFIABLE Findings

## Summary

- **CONFIRMED:** 1 (Finding #1 — isolation + merge workflow is feasible via diff-collection)
- **REFUTED:** 1 (Finding #2 — automated context assembly is NOT feasible on existing primitives)
- **Irreducibly Unverifiable:** 0

---

## Resolutions

### Finding 1: pi-iso isolation + cross-workspace shared state (Blackboard proposal §5.2.4)

**Finding:** The proposal wants candidate-diversity/merge workflows that preserve isolation guarantees while introducing cross-workspace shared state. The rebuttal deemed this UNVERIFIABLE because the `IsolationBackend` trait has no shared-channel concept between instances, isolated agents are torn down at completion, and IRC messaging is unstructured text with a 100-message cap.

**Prior Verdict:** ? UNVERIFIABLE

**New Verdict:** ✓ CONFIRMED (proposal feasible via diff-collection, NOT via live shared state)

**Deeper Evidence:**

1. **`/tmp/oh-my-pi/crates/pi-iso/src/diff.rs:29-57`** — The `Diff` struct is a `Vec<FileChange>` with a `unified_text()` method that concatenates all text-representable changes. This is a *structured* output: each `FileChange` carries `path: PathBuf`, `op: ChangeKind` (Added/Modified/Removed), and `diff: Option<String>`. This is NOT unstructured text — it is a machine-parseable delta record.

2. **`/tmp/oh-my-pi/crates/pi-iso/src/lib.rs:247-259`** — The `diff` method on `IsolationBackend` is async (designed for heavy I/O) and returns `IsoResult<Diff>`. Every backend inherits a `default_diff` that shells to `git diff` for git trees. This means **diff IS the post-hoc shared-state channel** — each candidate's output is fully captured as a structured diff before teardown.

3. **`/tmp/oh-my-pi/packages/coding-agent/src/task/isolation-runner.ts:118-128`** — `writeIsolationPatch()` calls `captureDeltaPatch(isolationDir, baseline)` and writes the result to `${artifactsDir}/${agentId}.patch`. This patch is available to the parent AFTER the isolated agent completes but BEFORE the merge decision.

4. **`/tmp/oh-my-pi/packages/coding-agent/src/task/isolation-runner.ts:147-219`** — `runIsolatedSubprocess()` returns a `SingleResult` that carries either `branchName` (branch mode) or `patchPath` (patch mode). The parent receives these results from ALL candidates before calling `mergeIsolatedChanges()`. This is a complete multi-candidate collection point.

5. **`/tmp/oh-my-pi/packages/coding-agent/src/task/worktree.ts:836-928`** — `mergeTaskBranches()` receives an array of `{ branchName, taskId, description, baseSha }` and processes them sequentially. The array is ordered by the caller. A quality-based selector could reorder this array, inspect patches via `git diff baseSha..branchName`, and filter candidates BEFORE merge.

6. **`/tmp/oh-my-pi/packages/coding-agent/src/task/isolation-runner.ts:250-303`** — `mergeIsolatedChanges()` is the merge gate. It runs AFTER the isolated subprocess returns and BEFORE changes hit the parent repo. In patch mode (lines 307-350), the patch text is read from disk and can be inspected/compared before `git.patch.applyText()` fires. This is the architectural insertion point for a candidate evaluator.

**Reasoning:**

The rebuttal correctly identified that `pi-iso` has no *live* shared channel between isolated instances. But the proposal's feasibility does not require live cross-workspace communication. The existing architecture already implements a **post-hoc diff-collection workflow**:

1. N candidates run in isolation (each gets a CoW snapshot).
2. Each candidate's changes are captured as structured diffs (git patches or `Diff` structs) before teardown.
3. The parent holds ALL candidate patches/branches simultaneously after all runs complete.
4. The merge gate (`mergeIsolatedChanges`) is called per-candidate with full access to the result.

A "Blackboard" for search-plane diversity does NOT need live cross-workspace state. The diff-collection point after candidate completion serves as a post-hoc blackboard where all candidates' outputs are available for comparison, scoring, and selective merge. The proposal's merge workflow is feasible because the *diff IS the shared state* — captured per-candidate and aggregated in the parent scope.

The rebuttal's concern about "breaking CoW isolation invariant" is valid for LIVE shared state during execution, but irrelevant for post-hoc diff collection, which is already implemented.

**Gate Outcome:** ADVANCE to design — The minimal true capability: isolated candidates produce structured diffs that are collected post-hoc in the parent scope, enabling quality-based selection/reordering before sequential cherry-pick merge. No live cross-workspace channel is needed.

---

### Finding 2: read tool's summarization → automated context assembly (§5.1.3)

**Finding:** The proposal references the `read` tool's structural summarization as a foundation for query-driven context packs. The rebuttal deemed this UNVERIFIABLE because it couldn't determine whether summarization is file-type-triggered or query-driven, and found no `ContextPack`/`contextAssembl` APIs.

**Prior Verdict:** ? UNVERIFIABLE

**New Verdict:** ✗ REFUTED (automated context assembly is NOT feasible on existing primitives)

**Deeper Evidence:**

1. **`/tmp/oh-my-pi/crates/pi-ast/src/summary.rs:164-211`** — `summarize_code()` takes `SummaryOptions { code, lang, path, min_body_lines, min_comment_lines, unfold_until_lines, unfold_limit_lines }`. There is NO query parameter, NO relevance input, NO "what does the caller need" parameter. The function is purely structural: it parses AST nodes, identifies elidable bodies (function bodies, class bodies, blocks), and produces kept/elided segments based on syntax structure alone.

2. **`/tmp/oh-my-pi/crates/pi-ast/src/summary.rs:247-326`** — `collect_elidable_tree()` walks the AST and builds an `ElidableForest` based on node types (`is_elidable_kind`) and line thresholds. The criteria are exclusively syntactic: "is this node a `statement_block`? Is it ≥ 4 lines?" No semantic, query-driven, or relevance-based logic exists.

3. **`/tmp/oh-my-pi/crates/pi-ast/src/summary.rs:419-745`** — `is_elidable_kind()` is a pure pattern match on language × AST node kind. It answers "can this node's body be hidden?" — not "is this node relevant to a query?"

4. **`/tmp/oh-my-pi/packages/coding-agent/src/tools/read.ts:1998-2035`** — `#trySummarize()` in the read tool triggers summarization when: (a) `read.summarize.enabled` is true, (b) no explicit selector was used (the read has `parsed.kind === "none"`), (c) file is not prose (unless `read.summarize.prose` is on), (d) file size/line count is within thresholds. The trigger is entirely file-characteristic-based — NOT query-driven.

5. **`/tmp/oh-my-pi/packages/coding-agent/src/tools/read.ts:2459-2492`** — The summarization path is entered when `parsed.kind === "none"` (no selector), confirming it fires as a *default behavior for bare reads*, not as a response to any context need or query.

6. **`/tmp/oh-my-pi/packages/coding-agent/src/config/settings-schema.ts:3092-3165`** — The `read.summarize.*` settings expose: `enabled` (bool), `prose` (bool), `minBodyLines` (number), `minCommentLines` (number), `minTotalLines` (number), `unfoldUntil` (number), `unfoldLimit` (number). These are all structural thresholds — zero semantic/relevance parameters.

7. **No relevance-selection API exists.** Grepping for `contextAssembl`, `ContextPack`, `relevance.*select`, `query.*context`, `semantic.*retriev`, `information.*need` across all of `/tmp/oh-my-pi/packages/coding-agent/src/` returns zero hits related to context assembly. The closest concept is `mnemopi` (long-term memory recall by embedding similarity), which is scoped to memory fragments, not codebase file selection.

8. **`/tmp/oh-my-pi/packages/coding-agent/src/task/types.ts:130-144`** — `TaskItem` schema: `{ name?, agent?, task, outputSchema?, schemaMode?, isolated? }`. No `needs`, `contextNeeds`, or information-requirement declaration field exists.

**Reasoning:**

The `read` tool's summarization is a **fixed structural transform** — it takes source code and produces a declarations-only view by eliding function/method bodies that exceed a line threshold. It is:
- Triggered by file characteristics (extension, size, absence of explicit selector)
- Controlled by numeric thresholds (min lines, unfold targets)
- Determined by AST node types per language
- Completely unaware of any caller's "needs" or query

The proposal's `ContextAssembler` requires:
1. A relevance-scoring mechanism (which file content matters for a given task?) — **does not exist**
2. A token-budget-aware selector (pick the most relevant N tokens) — **does not exist**
3. A declared-needs input from the task specification — **does not exist in TaskItem or AgentDefinition**
4. A query-driven summarization variant (show me the parts of this file relevant to X) — **does not exist**

The structural summarizer provides a *compression* primitive (show declarations, hide bodies) but NOT a *selection* primitive (choose which files/sections are relevant to a task). These are fundamentally different capabilities. Building automated context assembly would require:
- A semantic index over the codebase (embeddings, symbol graphs, or dependency analysis)
- A query→relevance scoring function
- A budget-aware greedy selection algorithm
- A `needs` field in the task specification schema

All of this is net-new infrastructure. The existing summarizer contributes only a final formatting step (compress selected files for token efficiency) — it cannot drive the selection itself.

**Gate Outcome:** DISCARD — The proposal's claim that existing `read` summarization provides a "foundation" for automated context assembly is false. The summarizer is a fixed structural compressor with zero query-driven or relevance-aware behavior. Automated context assembly requires a greenfield semantic retrieval system; the existing primitives contribute at most a final formatting stage.
