# Spec, Decision, and Delivery Planes — OMP Mapping

## 1. Inventory of Spec/Decision/Delivery Mechanisms

### Spec-Relevant Mechanisms

| Mechanism | Source | Role |
|-----------|--------|------|
| `todo` tool | `packages/coding-agent/src/tools/todo.ts` | Phase-based task decomposition with status tracking |
| Plan mode | `packages/coding-agent/src/plan-mode/` | Structured requirement drafting → approval → execution pipeline |
| Prewalk | `src/task/executor.ts`, agent frontmatter | Pre-implementation planning phase before tool hand-off |
| Plan approval | `src/modes/interactive-mode.ts` | Gate between spec and execution: Approve & execute / compact / keep |
| Plan file (`local://`) | Plan-mode artifacts | Durable, addressable spec document persisted across compactions |

### Decision-Relevant Mechanisms

| Mechanism | Source | Role |
|-----------|--------|------|
| `ask` tool | `packages/coding-agent/src/tools/ask.ts` | Structured multi-option selection with recommendations |
| Session tree (`/tree`) | `src/session/session-manager.ts` | In-session branching for exploring alternatives |
| Session fork (`/branch`) | `src/session/session-manager.ts` | New session file fork at any decision point |
| Resolution devices | `src/tools/resolve.ts` | Binary accept/reject for staged previews |
| Checkpoint/Rewind | `src/tools/checkpoint.ts` | Investigate then commit or discard findings |
| Plan approval UI | `src/modes/interactive-mode.ts` | 3-way choice: execute, compact+execute, keep context |

### Delivery-Relevant Mechanisms

| Mechanism | Source | Role |
|-----------|--------|------|
| `github` tool | `src/tools/gh.ts` | PR creation, push, checkout, CI run watching |
| Commit agent | `src/commit/agentic/` | Conventional commit generation, split-commit with topo-sort |
| Release script | `scripts/release.ts` | Version bump → changelog → commit → tag → push → watch CI |
| CI pipeline | `.github/workflows/ci.yml` | Multi-stage: metadata → build → test → sign → publish → verify |
| Autoresearch | `src/autoresearch/` | Experiment keep/discard/revert workflow with git isolation |
| Session export/share | `src/export/` | Encrypted session snapshots for sharing |

---

## 2. Spec Plane Mapping

### 2.1 How Requirement/Constraint Management Works Now

#### Todo Tool as Requirement Register

Claim: The `todo` tool provides a hierarchical requirement decomposition structure with phases and tasks, enforcing single-active-task sequencing.
Rationale: Phases map to requirement categories; tasks map to individual requirements/acceptance criteria; the single-active-task invariant enforces sequential completion.
Evidence: `packages/coding-agent/src/tools/todo.ts` — `TodoPhase: { name: string, tasks: TodoItem[] }`, `TodoItem: { content: string, status: "pending" | "in_progress" | "completed" | "abandoned" }`. `normalizeInProgressTask()` enforces at most one active task.
Confidence: VERIFIED

Claim: Todo state persists across session resume and compaction, but completed/abandoned tasks are stripped on resume.
Rationale: This models a "done items roll off" pattern — not suitable for maintaining a living spec with acceptance criteria that reference past deliverables.
Evidence: `docs/tools/todo.md` line 152: "On session resume, `AgentSession.#syncTodoPhasesFromBranch()` strips completed and abandoned tasks before restoring the cached list."
Confidence: VERIFIED

#### Plan Mode as Spec Authoring

Claim: Plan mode is the closest existing mechanism to a formal specification workflow: it constrains the agent to read-only tools while it drafts a plan file, requires explicit human approval before execution begins, and persists the approved plan as an addressable `local://` artifact.
Rationale: The three approval paths (execute immediately, compact+execute, keep context) model different confidence levels in the spec. The plan file survives compaction and is referenced during execution.
Evidence: `src/plan-mode/state.ts`: `PlanModeState { enabled, planFilePath, workflow?, reentry? }`. Plan-mode active prompt restricts tools. `docs/resolve-tool-runtime.md`: plan approval via `/xdev/propose`. `docs/session-tree-plan.md` lines 195-205: approval reaches `#approvePlan(...)`.
Confidence: VERIFIED

#### Constraints Modeling (Hard vs Soft)

Claim: OMP has no explicit hard/soft constraint taxonomy. The closest primitives are: (1) tool whitelists during plan mode (hard constraint on actions), (2) TTSR rules that interrupt execution on regex match (hard behavioral gate), and (3) plan-mode write restrictions (hard write-path guard).
Rationale: These are all enforcement mechanisms, not declarative constraint specifications. There is no data model for "this requirement is hard" vs "this is a preference."
Evidence: `src/tools/plan-mode-guard.ts` — enforces plan-mode write policy. `docs/resolve-tool-runtime.md`: "the model can comply by writing to `/xdev/resolve` or `/xdev/reject`; a write to any other path is still a detour." TTSR rules in `.omp/rules/` can carry `condition:` regex.
Confidence: VERIFIED

### 2.2 Extension Strategy for Spec Plane

**Gap: No acceptance criteria formalism.** The todo tool tracks task completion status but not success/failure conditions. A Spec Plane needs verifiable acceptance predicates.

**Proposed Extension — Spec-Aware Plan Mode:**

1. **Constraint schema in plan file**: Extend the `local://<slug>-plan.md` format with structured frontmatter declaring hard constraints (MUST), soft constraints (SHOULD), and acceptance criteria (testable predicates).

2. **Todo phases as requirement tiers**: Map `phases` to requirement priority tiers (P0-critical, P1-important, P2-nice-to-have). The existing `init` op with `list` already supports multi-phase structure.

3. **Acceptance gate before `done`**: Extend the `todo` tool's `done` op to accept an optional `evidence` field — a reference to test output, bash result, or artifact that proves the acceptance criterion. Store in `details.completedTasks[].evidence`.

4. **Constraint violation detection via TTSR**: Write rules that fire when an edit touches a path or pattern that violates a declared hard constraint (e.g., "never modify generated files" → rule matches `tool:write(generated/*)` and interrupts).

---

## 3. Decision Plane Mapping

### 3.1 How Structured Choice Works Now

#### Ask Tool as Decision Elicitation

Claim: The `ask` tool implements a structured decision protocol with options, recommendations, multi-select, and timeout-based auto-selection — directly mapping to preference elicitation in the Decision Plane.
Rationale: It surfaces a set of alternatives, marks a recommendation, collects user preference, and records the choice in the transcript with full structured details.
Evidence: `packages/coding-agent/src/tools/ask.ts` — `Question { id, question, options: { label, description? }[], multi?, recommended? }`. Result includes `selectedOptions`, `customInput`, `timedOut`. Timeout auto-selects recommended option.
Confidence: VERIFIED

Claim: The `ask` tool's `recommended` field with timeout-based auto-select implements a rudimentary "default preference with override" pattern.
Rationale: This is analogous to "use the Pareto-optimal choice unless the human objects within a time window" — a progressive-trust mechanism.
Evidence: `docs/tools/ask.md` lines 47-48: "If a timeout fires before any selection/custom input, the tool auto-selects the recommended option, or the first option when no valid recommended index exists."
Confidence: VERIFIED

#### Session Branching as Decision Exploration

Claim: Session tree navigation (`/tree`) and session forking (`/branch`) implement a decision-exploration mechanism where multiple alternative paths can be pursued and compared without destroying the decision history.
Rationale: `/tree` moves the leaf pointer without rewriting history — abandoned paths are summarized and preserved. `/branch` creates an entirely new session file linked via `parentSession`. Both preserve the decision-point context.
Evidence: `docs/session-tree-plan.md`: "Branching does not rewrite history; it only changes where the leaf points." `branchWithSummary(branchFromId, summary)` preserves the abandoned path's summary. `createBranchedSession(leafId)` forks history up to a boundary.
Confidence: VERIFIED

#### Checkpoint/Rewind as Speculative Decision

Claim: The checkpoint/rewind mechanism implements a "try-then-decide" pattern where the agent can investigate an approach, capture findings in a report, then rewind all state changes — making it a side-effect-free decision exploration tool.
Rationale: Checkpoint saves message count + entry ID + timestamp; rewind restores context to that state with only the report surviving. This is pure speculative execution for decision-making.
Evidence: `src/tools/checkpoint.ts`: `CheckpointState { checkpointMessageCount, checkpointEntryId, startedAt }`. `RewindTool.execute()` returns `{ report, rewound: true }` and triggers context rollback to the checkpoint message count.
Confidence: VERIFIED

#### Resolution Devices as Binary Decisions

Claim: The `/xdev/resolve` and `/xdev/reject` devices implement a binary accept/reject decision gate for staged changes, with the rejection reason persisted.
Rationale: This is structurally identical to a code review approval gate — a proposed change is staged, reviewed (preview), then accepted or rejected with rationale.
Evidence: `docs/resolve-tool-runtime.md`: "dispatchResolutionDevice(session, 'resolve' | 'reject', text)" — the text field carries the decision rationale.
Confidence: VERIFIED

### 3.2 What's Missing for Decision Plane

**Gap: No Pareto comparison framework.** The `ask` tool presents options but provides no mechanism to evaluate trade-offs across multiple criteria simultaneously.

**Gap: No ADR (Architecture Decision Record) persistence.** Decisions made via `ask` or branching are captured in the session transcript but not extracted into a structured, queryable decision log.

**Gap: No preference learning.** Repeated decisions on similar topics don't inform future recommendations — each `ask` call starts fresh.

### 3.3 Extension Strategy for Decision Plane

1. **ADR auto-generation from ask decisions**: When the `ask` tool completes, emit a structured `custom_message` entry with type `"decision-record"` containing: context (what prompted the decision), options considered, selected option, rationale, and consequences. These can be extracted into `docs/adr/` files.

2. **Pareto frontier via checkpoint fan-out**: Use `checkpoint` → try approach A → `rewind` with report → `checkpoint` → try approach B → `rewind` with report → compare reports on multiple dimensions. The reports become Pareto candidates.

3. **Decision memory across sessions**: Leverage the OMP memory pipeline to retain decision patterns. When an `ask` tool with similar option shapes appears, inject prior decisions as context for the `recommended` field.

4. **Session branching as A/B comparison**: Formalize the pattern of `/branch` at a decision point → implement alternative A → score → resume original → implement alternative B → score → merge. The branch summaries contain enough context to compare.

---

## 4. Delivery Plane Mapping

### 4.1 How Git/PR/Release Infrastructure Works Now

#### GitHub Tool as Delivery Primitive

Claim: The `github` tool provides the atomic delivery operations needed for progressive exposure: PR creation (with draft mode), push, CI run watching, and branch checkout for review.
Rationale: Draft PRs are the "canary" equivalent in code delivery — visible to reviewers but not merged. `run_watch` provides real-time CI feedback (the "runtime signal" of the Delivery Plane). PR labels and reviewers formalize the approval gate.
Evidence: `src/tools/gh.ts` lines 260-285: `githubSchema` declares ops `pr_create`, `pr_push`, `pr_checkout`, `run_watch`. Fields include `draft?`, `reviewer?`, `label?`, `fill?`. `GhRunWatchViewDetails` tracks `state: "watching" | "completed"`, `failedLogs`, poll count.
Confidence: VERIFIED

#### Release Script as Progressive Delivery Pipeline

Claim: The release script (`scripts/release.ts`) implements a multi-gate delivery pipeline: preflight checks → version bump → changelog → commit → tag → atomic push → CI watch → manual retry guidance on failure.
Rationale: Each step is a gate: dirty working directory blocks release, version comparison blocks downgrades, CI failure provides retry instructions. The atomic push (`git push --atomic`) ensures branch and tag arrive together or not at all.
Evidence: `scripts/release.ts` lines 197-394: `cmdRelease()` — branch check, clean check, version comparison, package.json update, Cargo.toml update, changelog update, `bun run check`, commit, atomic tag+push, `watchCI()` with per-job failure log tailing. On failure: prints retry instructions preserving the release tag.
Confidence: VERIFIED

#### CI Pipeline as Multi-Stage Verification

Claim: The CI workflow implements progressive verification with early-exit on failure: metadata resolution → lint/typecheck → native build → test → binary assembly → GitHub release → binary verification → npm publish → Homebrew tap update.
Rationale: Each stage gates the next. The `release_github_verify` step downloads the published binary and asserts code signing — a post-deployment canary check. The Homebrew tap only updates after verify passes.
Evidence: `.github/workflows/ci.yml` lines 683-850: `release_github` → `release_github_verify` (codesign + smoke-test) → `release_npm` (gated on verify success) → `release_brew` (gated on verify success).
Confidence: VERIFIED

#### Checkpoint/Rewind + Git as Rollback

Claim: The checkpoint/rewind mechanism combined with git-based session state provides a rollback primitive: the agent can checkpoint, make changes, and rewind if verification fails — restoring both conversation context and (through git stash/reset) filesystem state.
Rationale: Checkpoint captures session message state; the autoresearch module demonstrates the full pattern: `git reset --hard` to baseline commit on experiment failure.
Evidence: `src/autoresearch/tools/log-experiment.ts`: `revertFailedExperiment()` calls `git.reset(cwd, { hard: true, target: "HEAD" })` then `git.clean(cwd)` on discard/crash/checks_failed. `src/tools/checkpoint.ts`: rewind reports are retained after context rollback.
Confidence: VERIFIED

#### Run Watch as Runtime Signal Monitoring

Claim: The `github` tool's `run_watch` operation implements real-time runtime signal monitoring — polling CI workflows, detecting job-level failures before workflow completion, and tailing failed job logs for rapid feedback.
Rationale: This is structurally identical to monitoring a canary deployment: watch metrics (CI status), detect anomalies (job failures), provide diagnostics (log tails), and terminate early (fail-fast on first job failure).
Evidence: `scripts/release.ts` lines 26-122: `watchCI()` polls `gh run list`, checks in-progress runs for failed jobs (`conclusion !== "success" && conclusion !== "skipped"`), tails last 20 lines of failed job logs, returns `false` on first failure.
Confidence: VERIFIED

### 4.2 What's Missing for Delivery Plane

**Gap: No progressive/canary deployment orchestration.** The CI pipeline is all-or-nothing per release. There is no built-in mechanism for deploying to a subset of users, monitoring, then promoting.

**Gap: No automated rollback on runtime signals.** CI failure produces retry instructions for humans but doesn't auto-revert the tag or initiate a patch release.

**Gap: No feature flag integration.** There is no mechanism to gate delivered features behind flags that can be toggled without redeployment.

**Gap: No deployment environment model.** The system knows about "release to npm/GitHub/Homebrew" but has no concept of staging → production promotion.

### 4.3 Extension Strategy for Delivery Plane

1. **PR-as-canary with `run_watch` feedback loop**: Extend the `github` tool's `pr_create` with a `canary: true` option that: (a) creates a draft PR, (b) enters `run_watch` mode, (c) auto-promotes to ready-for-review only when all checks pass, (d) auto-closes on persistent failure.

2. **Progressive merge via stacked PRs**: Implement a `split_delivery` operation (parallel to `split_commit`) that decomposes a large change into ordered PRs with dependency tracking, merging each only after its predecessor's CI passes.

3. **Rollback automation**: When `run_watch` detects failure after tag push, auto-generate the `git revert` command (or auto-execute with approval) rather than just printing retry instructions.

4. **Runtime signal extension point**: Add a `delivery_signal` event to the extension API that fires when `run_watch` detects state changes. Extensions can then implement custom promotion/rollback logic (e.g., "if npm download metrics drop after publish, auto-deprecate").

---

## 5. Gap Analysis Summary

| Plane | Has | Missing |
|-------|-----|---------|
| **Spec** | Phase-based task decomposition; plan-mode spec authoring; plan approval gate; tool restriction during spec phase | Acceptance criteria formalism; hard/soft constraint taxonomy; constraint satisfaction verification; spec versioning/diffing |
| **Decision** | Structured option selection (ask); branch-based exploration; checkpoint/rewind speculation; binary resolve/reject | Pareto multi-criteria comparison; ADR persistence & querying; preference learning; decision impact tracking |
| **Delivery** | PR creation with draft mode; CI run watching; conventional commit generation; multi-stage release pipeline; git-based rollback (autoresearch) | Canary/progressive deployment; automated rollback on signals; feature flags; environment promotion model; deployment health monitoring |

### Cross-Plane Gaps

| Gap | Planes Affected | Impact |
|-----|----------------|--------|
| No traceability from spec → delivery | Spec + Delivery | Cannot verify that delivered code satisfies declared requirements |
| No decision-to-spec feedback | Decision + Spec | Decisions don't update constraint models |
| No delivery-to-decision feedback | Delivery + Decision | Deployment outcomes don't inform future architectural choices |
| No constraint propagation | All three | A spec constraint like "must not break API" has no automated enforcement in delivery |

---

## 6. Proposal — Concrete Adaptation Strategies

### 6.1 Spec Plane: Structured Requirement Protocol

**Using existing primitives:**

- **Plan file format extension**: Add YAML frontmatter to `local://<slug>-plan.md` with:
  ```yaml
  constraints:
    hard:
      - "All existing tests must pass"
      - "No new dependencies without approval"
    soft:
      - "Prefer existing patterns over new abstractions"
  acceptance:
    - id: "AC-1"
      criterion: "PR passes CI"
      verification: "run_watch"
    - id: "AC-2"  
      criterion: "No regressions in test suite"
      verification: "bash:bun test"
  ```

- **Todo phases as requirement tiers**: Use the existing `init` op's multi-phase structure with convention: phase names encode priority (`P0: Critical`, `P1: Important`, `P2: Enhancement`).

- **Constraint enforcement via TTSR rules**: Auto-generate `.omp/rules/` from the plan's hard constraints. Example: constraint "no new dependencies" → rule with `condition: "tool:bash.*install|tool:write.*package.json"`.

**New primitives needed:**

- A `spec` tool (or `todo` extension) that associates acceptance criteria with tasks and tracks verification evidence.
- Plan-mode integration that validates constraints before approval.

### 6.2 Decision Plane: ADR + Preference Protocol

**Using existing primitives:**

- **Ask tool + custom_message = ADR**: After each `ask` completion, emit a `custom_message` with `customType: "adr"` containing the structured decision record. This survives compaction as a branch summary entry.

- **Session branching as decision exploration**: Formalize the pattern:
  1. Label current leaf as "Decision Point: <topic>"
  2. Branch to explore option A → summarize findings
  3. Branch back to decision point → explore option B → summarize findings
  4. Return to decision point with both summaries as context
  5. Use `ask` to select final direction

- **Checkpoint/rewind as cost estimation**: For each Pareto candidate:
  1. Checkpoint
  2. Attempt implementation
  3. Measure (lines changed, tests passing, time taken)
  4. Rewind with structured report of costs
  5. Compare reports across multiple dimensions

**New primitives needed:**

- A `decide` tool (extending `ask`) that takes evaluation criteria, weights them, and presents a Pareto-optimal recommendation.
- ADR extraction into `docs/adr/` from decision custom_messages (could be a compaction-time extension hook on `session_before_compact`).

### 6.3 Delivery Plane: Progressive Exposure Protocol

**Using existing primitives:**

- **Draft PR + run_watch = canary**: The `github` tool already supports `draft: true` and `run_watch`. Chain them: `pr_create(draft: true)` → `run_watch` → on success, use `bash: gh pr ready` to promote.

- **Split commit as progressive delivery**: The `split_commit` tool with `dependencies` field and topological sorting already implements ordered, atomic delivery of related changes. Extend to create one PR per commit group with inter-PR dependencies.

- **Autoresearch as deployment experiment framework**: The `log_experiment(keep/discard)` pattern with git-reset rollback is directly applicable to deployment experiments: deploy → measure → keep (promote) or discard (rollback).

- **Release script watchCI as health check**: The `watchCI()` loop with job-level failure detection and log tailing is a runtime signal monitor. Extend it with webhooks or polling for post-deployment metrics.

**New primitives needed:**

- A `deliver` tool orchestrating the full progressive pipeline: stage → monitor → promote/rollback.
- Environment model in session state: `{ environments: ["staging", "production"], current: "staging" }`.
- Promotion gates: configurable conditions (CI pass, N minutes without alerts, manual approval) before advancing to next environment.
- Rollback automation: on signal detection, auto-generate revert commit and PR rather than just logging.

### 6.4 Integration Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Nine-Plane Framework                       │
├───────────────┬──────────────────┬──────────────────────────┤
│   Spec Plane  │  Decision Plane  │     Delivery Plane       │
├───────────────┼──────────────────┼──────────────────────────┤
│ Plan mode     │ ask tool         │ github tool (PR/push)    │
│ + constraint  │ + ADR emission   │ + run_watch monitoring   │
│   frontmatter │                  │                          │
│               │ Session branching│ Commit agent             │
│ Todo phases   │ + comparison     │ + split delivery         │
│ + acceptance  │   protocol       │                          │
│   evidence    │                  │ Release pipeline         │
│               │ Checkpoint/rewind│ + progressive gates      │
│ TTSR rules    │ + Pareto eval    │                          │
│ = hard gates  │                  │ Autoresearch revert      │
│               │ Resolution device│ = rollback primitive     │
│ Spec tool     │ = binary gate    │                          │
│ (new)         │                  │ Deliver tool (new)       │
│               │ Decide tool (new)│                          │
└───────────────┴──────────────────┴──────────────────────────┘
         │                │                    │
         └────────────────┼────────────────────┘
                          │
              ┌───────────┴───────────┐
              │   Cross-Plane Wiring  │
              ├───────────────────────┤
              │ Spec→Decision:        │
              │   Constraints inform  │
              │   option generation   │
              │                       │
              │ Decision→Delivery:    │
              │   ADRs gate delivery  │
              │   scope               │
              │                       │
              │ Delivery→Spec:        │
              │   Runtime signals     │
              │   update acceptance   │
              │   criteria status     │
              └───────────────────────┘
```

### 6.5 Implementation Priority

| Priority | Component | Effort | Existing Foundation |
|----------|-----------|--------|---------------------|
| 1 | Plan file constraint frontmatter | Low | Plan mode already uses `local://` files with content |
| 2 | ADR emission from ask tool | Low | `custom_message` entries with custom types exist |
| 3 | Draft PR + run_watch pipeline | Low | Both `github` ops exist, just need orchestration |
| 4 | Todo acceptance evidence | Medium | Requires `todo` tool schema extension |
| 5 | TTSR auto-generation from constraints | Medium | TTSR rule system exists, need generation logic |
| 6 | Branching comparison protocol | Medium | Session branching exists, need formalization |
| 7 | Split delivery (PR per commit group) | Medium | Split commit exists, need PR integration |
| 8 | Decide tool (Pareto) | High | New tool, requires multi-criteria evaluation logic |
| 9 | Deliver tool (progressive) | High | New tool, requires environment + signal model |
| 10 | Cross-plane traceability | High | Requires new data model linking spec→decision→delivery |
