# Evaluation Plane — Confirmed Foundation

> Source: research/03 + rebuttals/03 + resolutions/03
> Funnel: 5 survived adversarial · 2 re-confirmed · 4 discarded · 0 open decision

## ✅ Confirmed buildable capabilities

### C-eval-1: LSP diagnostics as deterministic verification oracle
- **What it is**: Workspace-wide and per-file type checking (tsc/cargo/go/pyright) returning binary OK/error-list outcomes usable as pass/fail gates.
- **oh-my-pi substrate**: `/tmp/oh-my-pi/docs/tools/lsp.md:68-84` — workspace diagnostics runs cargo/tsc/go/pyright; per-file diagnostics queries real LSP servers and custom linter clients. Lines 78-83 confirm "Single target with no issues: OK" vs grouped diagnostic list.
- **Provenance**: survived adversarial review

### C-eval-2: Extension tool_call blocking (evaluation gating primitive)
- **What it is**: The `ExtensionToolWrapper` emits `tool_call` events before execution that can return `{ block: true, reason }` to prevent any tool invocation — usable as an evaluation gate.
- **oh-my-pi substrate**: `/tmp/oh-my-pi/packages/coding-agent/src/extensibility/extensions/wrapper.ts:206-228` — `tool_call` event can return `{ block: true, reason }` to prevent execution.
- **Provenance**: survived adversarial review

### C-eval-3: TTSR as streaming regex/AST-grep invariant guard
- **What it is**: Real-time pattern matching (regex + ast-grep) against the agent's streaming token output with abort-and-retry semantics; enforces structural invariants during generation.
- **oh-my-pi substrate**: `/tmp/oh-my-pi/packages/coding-agent/src/export/ttsr.ts:1-7` — stream abort + rule injection + retry on pattern match. Lines 55-63 — settings with repeatMode, interruptMode. Lines 354-365 — `checkDelta` appends to per-stream-key buffers and regex-matches current buffer.
- **Scope limit**: Operates ONLY on current-turn streaming token output (text_delta, thinking_delta, toolcall_delta). Buffer resets every turn (`resetBuffer()` at ttsr.ts:558-560). No cross-turn memory, no post-hoc trace analysis, no temporal ordering assertions.
- **Provenance**: survived adversarial review

### C-eval-4: Advisor/watchdog as rubric-based evaluator
- **What it is**: Structurally independent review agent with its own model, ToolSession, and tool grants that reviews primary agent output with configurable severity (nit/concern/blocker) and can interrupt via steering channel. WATCHDOG.yml supports multi-advisor rosters with per-advisor specialization.
- **oh-my-pi substrate**: `/tmp/oh-my-pi/docs/advisor-watchdog.md:75-86` — independent Agent instance with independent ToolSession and configurable tool grants. Lines 88-95 — severity levels nit/concern/blocker with interruption semantics. Lines 208-237 — WATCHDOG.yml schema with advisors[].name/model/tools/instructions.
- **Provenance**: survived adversarial review

### C-eval-5: Metaharness benchmark orchestration for multi-arm experiments
- **What it is**: Complete benchmark orchestration system managing runs in SQLite, normalizing traces across benchmark adapters (harbor/edit/snapcompact), supporting experiment arms with role-based labeling, and exposing REST+SSE API with web dashboard.
- **oh-my-pi substrate**: `/tmp/oh-my-pi/packages/metaharness/src/benchmarks.ts:23-45` — BENCHMARK_DEFINITIONS with harbor/edit/snapcompact. `/tmp/oh-my-pi/packages/metaharness/src/experiments.ts:48-56` — ExperimentDetail with arms, tasks, and task×arm matrix. `/tmp/oh-my-pi/packages/metaharness/src/server.ts:1-24` — REST API surface.
- **Provenance**: survived adversarial review

### C-eval-6: Metaharness differential comparison (pairwise head-to-head)
- **What it is**: Task-level pairwise win/loss/tie comparison between experiment arms, with disagreement localization (split filter) and baseline-anchored delta display for pass rate, cost, and time.
- **oh-my-pi substrate**: `/tmp/oh-my-pi/packages/metaharness/src/web/app.tsx:411-433` — `headToHead()` computes focusWins/armWins/bothPass/bothFail per task. `/tmp/oh-my-pi/packages/metaharness/src/web/app.tsx:1356-1358` — `splitCount` filters tasks where arms disagree (passes > 0 && passes < decided). `/tmp/oh-my-pi/packages/metaharness/src/web/app.tsx:773-797` — `Delta()` computes signed percentage-point and relative differences from baseline anchor. `/tmp/oh-my-pi/packages/metaharness/src/experiments.ts:48-56` — `ExperimentDetail.matrix`: `Record<armLabel, Record<taskId, {status, reward}>>`.
- **Scope limit**: This is NOT `calibratedFinalPassPct` (that function is projection-only for in-flight arms, not differential comparison). All comparison logic is CLIENT-SIDE React code only — no server-side comparison API. Visual/manual only; no automated thresholds or gating.
- **Provenance**: re-investigation CONFIRMED (with narrowing)

### C-eval-7: RunRole consumed for comparison anchoring
- **What it is**: `RunRole = "baseline" | "variant" | ""` is consumed by `pickReferenceArm()` to identify which arm serves as the comparison reference for delta calculations. Arms are forced comparable via identical task samples (`resolveArmLaunch` at server.ts:112-118).
- **oh-my-pi substrate**: `/tmp/oh-my-pi/packages/metaharness/src/web/app.tsx:753-766` — `pickReferenceArm()` selects highest-pass-rate baseline arm as comparison anchor using `a.run.role !== "baseline"`. `/tmp/oh-my-pi/packages/metaharness/src/store.ts:22` — `RunRole = "baseline" | "variant" | ""`. `/tmp/oh-my-pi/packages/metaharness/src/server.ts:112-118` — enforces identical task samples across arms.
- **Scope limit**: Zero statistical significance testing (grep: zero matches for wilson/binomial/chi/fisher/bootstrap/significance across all metaharness files). Zero automated regression detection or alerting (no backend comparison logic, no threshold-based alerts). Head-to-head is count-based raw wins/losses, not statistically tested.
- **Provenance**: re-investigation CONFIRMED (with narrowing)

## ❌ Must build net-new (gaps)

### G-eval-1: Metamorphic testing framework
- **What's missing**: No infrastructure for generating semantics-preserving input transformations and verifying output equivalence. No metamorphic-specific abstractions exist.
- **Evidence of absence**: grep: zero matches for "metamorphic" in the codebase. Eval's `parallel()` provides execution substrate but no transformation generators or equivalence oracles.
- **Nature of build**: greenfield subsystem (transformation library + equivalence oracle + eval-pipeline integration)

### G-eval-2: Adversarial input generation
- **What's missing**: The advisor system reviews agent output but cannot generate adversarial test inputs designed to break the agent's solution. No protocol exists for probing weaknesses with edge cases, boundary conditions, or attack inputs.
- **Evidence of absence**: `/tmp/oh-my-pi/docs/advisor-watchdog.md` — advisor "reviews the primary agent's transcript" but has no test-input generation protocol. Advisor personalities (WATCHDOG.yml) can be authored for adversarial review but the advisor architecture is observation-and-critique, not input-generation.
- **Nature of build**: extension (adversarial-generation WATCHDOG.yml personas are feasible within existing multi-advisor roster, but require careful design around `immuneTurns` rate-limiting constraints)

### G-eval-3: Statistical significance layer + automated regression alerting
- **What's missing**: (a) Statistical significance testing for arm comparisons (Wilson intervals, McNemar's test, etc.), (b) server-side comparison logic (currently all in React client), (c) automated alerting/webhook infrastructure, (d) CI/CD gate integration.
- **Evidence of absence**: grep: zero matches for `wilson|binomial|chi|fisher|bootstrap|significance` across all metaharness files. All comparison is client-side only (`headToHead`, `Delta`, `pickReferenceArm` live in web/app.tsx). No backend alerting system (grep: `alert|threshold|notify|webhook` returns only a browser `alert()` for launch failures at app.tsx:1704).
- **Nature of build**: extension (move headToHead to backend + add statistical significance testing + emit alerts on existing SSE channel; existing primitives — role-labeled arms, enforced task-sample comparability, per-task outcome matrix, SSE infrastructure — provide the substrate)

### G-eval-4: Gated pyramid orchestrator
- **What's missing**: No mechanism to compose verification layers into an ordered pyramid where earlier (cheaper) layers gate later (more expensive) ones. Each verification primitive (LSP, bash, advisor) operates independently with no sequencing framework.
- **Evidence of absence**: No orchestration layer found linking lsp diagnostics → bash test → advisor review in a gated sequence. Each tool is invoked independently by the agent. The `session_stop` event has a hard cap of 8 continuations and 30-second timeout per continuation — unsuitable for multi-layer pyramid iteration.
- **Nature of build**: extension (new extension using `tool_call` blocking to enforce layer ordering; must target the correct interception points — NOT `yield` which is subagent-only)

## 🗑 Discarded overreach (do NOT re-propose)

- **Pre-yield gate architecture** — REFUTED: `yield` is exclusively a subagent/task tool; main agent sessions never call it and have no "declare done" tool. The proposal's `on("tool_call", e => e.toolName === "yield")` hook targets the wrong mechanism entirely. (`/tmp/oh-my-pi/packages/coding-agent/src/tools/yield.ts:210` — YieldTool class for subagent result submission; `/tmp/oh-my-pi/packages/coding-agent/src/session/agent-session.ts:6317` — session_stop explicitly excluded for subagents)

- **Extensions injecting advisor advice via ctx.advise()** — REFUTED: `ExtensionAPI` exposes only `sendMessage()` and `sendUserMessage()` — no `advise()` method exists. `enqueueAdvice` is internal to `AdvisorRuntimeHost`, not exposed to extensions. Extensions cannot produce `<advisory>` elements with severity semantics. (`/tmp/oh-my-pi/packages/coding-agent/src/extensibility/extensions/types.ts:1191-1201` — ExtensionAPI surface; grep for `advise|enqueueAdvice` in extensibility/ returns zero matches)

- **TTSR for trace-level behavioral assertions** — REFUTED: TTSR operates on streaming token output only, buffer resets every turn (`resetBuffer()` at ttsr.ts:558-560), has no cross-turn memory, and cannot express temporal relationships like "X must precede Y". Its scope system filters which stream is monitored, not historical events. Building trace assertions requires an entirely new subsystem. (`/tmp/oh-my-pi/packages/coding-agent/src/export/ttsr.ts:558-560` — buffer reset on turn_start; `/tmp/oh-my-pi/docs/ttsr-injection-lifecycle.md:69-70` — "On turn_start, the stream buffer is reset")

- **Metaharness pluggable adapter wrapping system** — REFUTED: `BenchmarkKind` is a closed enum (`"harbor" | "edit" | "snapcompact"`) with hard-coded `BENCHMARK_DEFINITIONS`. No adapter registration API, no plugin system, no composability. Adding a new benchmark kind requires modifying the metaharness package source. (`/tmp/oh-my-pi/packages/metaharness/src/store.ts:19` — closed union type; `/tmp/oh-my-pi/packages/metaharness/src/benchmarks.ts:23-45` — hard-coded const array)

## ⚠ Open design decision

None.
