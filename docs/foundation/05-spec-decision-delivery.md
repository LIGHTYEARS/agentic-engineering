# Spec + Decision + Delivery Planes — Confirmed Foundation

> Source: research/05 + rebuttals/05 + resolutions/05
> Funnel: 5 survived adversarial · 2 re-confirmed · 8 discarded · 0 open decision

## ✅ Confirmed buildable capabilities

### C-sdd-1: Plan mode as spec-authoring with approval gate
- **What it is**: Plan mode constrains the agent to read-only tools while drafting a `local://<slug>-plan.md` file, requires explicit human approval via `xd://propose`, and offers three approval paths (execute immediately / compact+execute / keep context)
- **oh-my-pi substrate**: `packages/coding-agent/src/prompts/system/plan-mode-active.md:2-9` — enforces read-only working-tree semantics, restricts writes to `local://` artifacts only, mandates `xd://propose` as sole approval mechanism
- **Provenance**: survived adversarial review

### C-sdd-2: Todo tool as hierarchical requirement register
- **What it is**: Phase-based task decomposition with `TodoPhase[]` containing named phases and ordered `TodoItem[]` tasks, enforcing single-active-task sequencing via `normalizeInProgressTask()`
- **oh-my-pi substrate**: `packages/coding-agent/src/tools/todo.ts:25-33` — `TodoItem { content: string; status: TodoStatus }`, `TodoPhase { name: string; tasks: TodoItem[] }`
- **Scope limit**: TodoItem has NO evidence field, NO acceptance-criteria attachment, NO metadata beyond content+status; completed/abandoned tasks are stripped on session resume (not a living-spec register)
- **Provenance**: survived adversarial review

### C-sdd-3: Ask tool as structured decision elicitation
- **What it is**: Structured multi-option selection with options, optional `recommended` index, multi-select, timeout-based auto-selection of recommended option, and structured result (`selectedOptions`, `customInput`, `timedOut`)
- **oh-my-pi substrate**: `packages/coding-agent/src/tools/ask.ts:69` — `recommended?: number` in schema; lines 161-166 — `getAutoSelectionOnTimeout` auto-selects recommended on timeout
- **Scope limit**: `recommended` is agent-supplied (LLM-authored in tool call args), not memory-injected; no ADR emission, no custom_message side-channel, no post-completion hook; `concurrency = "exclusive"` prevents parallel multi-criteria elicitation
- **Provenance**: survived adversarial review

### C-sdd-4: Session tree/branch as decision exploration
- **What it is**: `/tree` navigates the session entry tree (same-file leaf movement preserving abandoned paths via `branchWithSummary`); `/branch` creates a new session file linked via `parentSession` for forked exploration
- **oh-my-pi substrate**: `docs/session-tree-plan.md:14` — "Branching does not rewrite history; it only changes where the leaf points"; `packages/coding-agent/src/session/session-entries.ts:35` — `SessionHeader.parentSession?: string`; lines 48-52 — `SessionEntryBase { id, parentId, timestamp }`
- **Provenance**: survived adversarial review

### C-sdd-5: GitHub tool PR creation + run_watch CI polling
- **What it is**: `pr_create` with `draft: true` support, and `run_watch` for real-time CI monitoring with job-level failure detection, failed-job log tailing, and poll interval management (3s fast → 15s slow)
- **oh-my-pi substrate**: `docs/tools/github.md:30` — `draft: boolean, No, defaults to false`; lines 204-221 — run_watch flow with poll intervals, failure grace period, and full failed-job log artifact
- **Scope limit**: `run_watch` monitors CI for a specific commit, NOT post-deploy runtime monitoring; it is pre-merge verification, not canary deployment
- **Provenance**: survived adversarial review

### C-sdd-6: Commit agent + split_commit for local git commits
- **What it is**: Conventional commit generation with optional split into multiple topologically-ordered commits; `git.commit()` + optional `git.push()` — operates at the local git level only
- **oh-my-pi substrate**: `packages/coding-agent/src/commit/agentic/index.ts:239-244` — `runSingleCommit` calls `git.commit(ctx.cwd, commitMessage)` then optionally `git.push(ctx.cwd)`
- **Scope limit**: Local commits only, zero GitHub PR integration; split_commit dependencies field determines commit ORDER, not PR dependency chains; no per-group PR creation
- **Provenance**: survived adversarial review

### C-sdd-7: Release script as multi-gate delivery pipeline
- **What it is**: CLI script (`scripts/release.ts`) implementing: preflight checks → version bump → changelog → commit → atomic tag+push → CI watch with job-level failure detection and retry guidance
- **oh-my-pi substrate**: `scripts/release.ts:26-122` — `watchCI()` polls `gh run list`, detects failed jobs, tails last N lines of failed job logs, returns false on first failure
- **Scope limit**: Standalone CLI script invoked via `bun scripts/release.ts`, not an agent tool; no structured feedback to extension/hook system; no automated rollback (prints retry instructions only)
- **Provenance**: survived adversarial review

### C-sdd-8: Cross-plane traceability substrate (minimal)
- **What it is**: Session entry `id/parentId` trees form implicit audit trails; `SessionHeader.parentSession` links forked sessions; `planReferencePath` threads plan file identity from spec-phase into delivery-phase subagents; `CustomEntry`/`CustomMessageEntry` provide general-purpose persistence slots for arbitrary typed cross-plane records
- **oh-my-pi substrate**: `packages/coding-agent/src/session/session-entries.ts:48-52` — `SessionEntryBase { id, parentId, timestamp }`; line 35 — `parentSession?: string`; lines 124-128 — `CustomEntry<T> { type: "custom", customType, data? }`; lines 201-209 — `CustomMessageEntry<T> { type: "custom_message", customType, content, details?, display }`; `packages/coding-agent/src/session/agent-session.ts:1927` — `#planReferencePath`
- **Scope limit**: This is a MINIMAL identifier-threading base, not a full traceability system; no automated constraint→option pipeline, no ADR→delivery-gate mechanism, no signal→acceptance feedback loop; explicit traceability must be DESIGNED on top of these primitives
- **Provenance**: re-investigation CONFIRMED

### C-sdd-9: Runtime TTSR rule generation (mid-session)
- **What it is**: `TtsrManager.addRule(rule)` is callable mid-session to register new TTSR rules that take immediate effect; the OmfgController demonstrates this pattern: write rule file to `.omp/rules/` + call `addRule()` = enforcement in the same session
- **oh-my-pi substrate**: `packages/coding-agent/src/export/ttsr.ts:304-340` — `addRule(rule: Rule): boolean` with no lifecycle guard preventing mid-session calls; `packages/coding-agent/src/modes/controllers/omfg-controller.ts:260-262` — `#registerLive(rule) { this.ctx.session.ttsrManager?.addRule(rule); }`
- **Scope limit**: No file-watch auto-discovery of newly written rule files (writing `.omp/rules/*.md` alone does NOT activate in current session without explicit `addRule` call); no "constraint→regex translator" exists — the translation from plan constraints to regex patterns is pure logic that must be built
- **Provenance**: re-investigation CONFIRMED

## ❌ Must build net-new (gaps)

### G-sdd-1: Acceptance criteria formalism
- **What's missing**: No mechanism to attach verifiable acceptance predicates to todo tasks or plan items; TodoItem is `{ content, status }` only — no evidence field, no success/failure conditions
- **Evidence of absence**: `packages/coding-agent/src/tools/todo.ts:25-28` — TodoItem has only `content: string; status: TodoStatus`; grep: zero matches for "evidence" or "acceptance" in todo.ts schema
- **Nature of build**: extension (schema extension to TodoItem + todo tool `done` op + session serialization)

### G-sdd-2: Hard/soft constraint taxonomy
- **What's missing**: No declarative data model for "this requirement is hard" vs "this is a preference"; existing enforcement (tool whitelists, TTSR, plan-mode guard) is all mechanism, not specification
- **Evidence of absence**: grep: zero matches for "hard.*constraint" or "soft.*constraint" as a data model anywhere in plan-mode or todo code
- **Nature of build**: greenfield subsystem (constraint schema definition + validation pipeline + integration with plan-mode approval)

### G-sdd-3: ADR (Architecture Decision Record) persistence
- **What's missing**: Decisions made via `ask` or branching are captured in session transcripts but not extracted into structured, queryable decision logs; ask tool returns `toolResult()` only — no custom_message side-channel emission
- **Evidence of absence**: `packages/coding-agent/src/tools/ask.ts` — zero hits for `custom_message`, `customMessage`, `appendCustom`, or ADR-related emission
- **Nature of build**: extension (hook on ask-tool completion to emit CustomMessageEntry with decision-record type + extraction pipeline to docs/adr/)

### G-sdd-4: Constraint→regex translator for TTSR auto-generation
- **What's missing**: While `addRule()` enables mid-session rule registration, there is no logic to parse plan-file constraints (natural language) into TTSR condition regexes
- **Evidence of absence**: grep: zero matches for "constraint" in `packages/coding-agent/src/export/ttsr.ts` or plan-mode code paths that invoke `addRule`
- **Nature of build**: extension (pure logic function: NL constraint → regex pattern; hook on plan-approval to generate and register rules)

### G-sdd-5: Progressive/canary deployment orchestration
- **What's missing**: CI pipeline is all-or-nothing per release; no built-in mechanism for deploying to a subset of users, monitoring, then promoting; no environment model (staging→production)
- **Evidence of absence**: grep: zero matches for "canary", "progressive", "staging", "environment" in `scripts/release.ts` or `src/tools/gh.ts`
- **Nature of build**: greenfield subsystem (environment model + promotion gates + post-deploy signal monitoring beyond CI)

### G-sdd-6: Automated rollback on runtime signals
- **What's missing**: CI failure produces retry instructions for humans but doesn't auto-revert tags or initiate patch releases; `run_watch` returns text result with no event emission to extension system
- **Evidence of absence**: `scripts/release.ts` prints `console.log` retry guidance on failure; no `git revert` or auto-rollback code path exists
- **Nature of build**: extension (hook on run_watch failure → auto-generate revert commit/PR with approval gate)

## 🗑 Discarded overreach (do NOT re-propose)

- **Constraint frontmatter on plan files** — REFUTED: plans are opaque markdown read as raw `.text()` with title-regex extraction only; no frontmatter parsing in plan-mode code path (`packages/coding-agent/src/plan-mode/plan-handoff.ts:28-36`)
- **Checkpoint/rewind as side-effect-free speculation** — REFUTED: rewind restores ONLY conversation message prefix; filesystem, git state, artifacts, blob-store are NOT restored (`docs/tools/rewind.md:87-93`)
- **Checkpoint enabled by default for speculation** — REFUTED: `checkpoint.enabled` defaults to `false`, must be explicitly opted-in (`packages/coding-agent/src/config/settings-schema.ts:3666-3675`)
- **Ask tool ADR/custom_message emission** — REFUTED: ask tool returns standard `toolResult()` only; no `appendCustomMessageEntry` call, no decision-record emission exists (`packages/coding-agent/src/tools/ask.ts` — zero hits for custom_message)
- **Todo `done` evidence field** — REFUTED: TodoItem is `{ content: string; status: TodoStatus }` with no evidence field; `done` op simply flips status (`packages/coding-agent/src/tools/todo.ts:25-28,447-449`)
- **Split_commit→PR pipeline (progressive delivery)** — REFUTED: split_commit creates local git commits only, zero GitHub PR integration; no per-group PR creation, no inter-PR dependency tracking (`packages/coding-agent/src/commit/agentic/index.ts:239-244`)
- **Delivery_signal extension event** — REFUTED: does not exist in extension event system; runner's event list is session-lifecycle events only; run_watch emits to TUI renderer, not extension bus (`packages/coding-agent/src/extensibility/extensions/runner.ts:596-604`)
- **Memory pipeline for option-shape retrieval → ask `recommended`** — REFUTED: mnemopi stores free-text facts/transcripts, no structured decision records, no option-shape similarity matching; `recommended` is LLM-authored with no memory injection path (`packages/mnemopi/src/core/extraction.ts:42-69`, `packages/coding-agent/src/tools/ask.ts:69`)

## ⚠ Open design decision

None.
