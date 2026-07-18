# Rebuttal: 03-evaluation-plane-omp-mapping.md

## Summary

- **Refuted**: 4
- **Unverifiable**: 2
- **Survived scrutiny**: 5 (core primitives)

The proposal correctly identifies most oh-my-pi primitives and their raw capabilities. Its weaknesses emerge when it proposes composite systems built ON those primitives — particularly: (1) the "pre-yield gate" architecture misunderstands what `yield` is, (2) TTSR is proposed for "trace-level behavioral assertions" but only supports regex/AST-grep pattern matching on streaming token output — not post-hoc trace analysis, (3) the advisor system cannot be "extended" into adversarial test design by extensions since extensions have no API to inject advice, and (4) the metaharness has no pluggable adapter system that allows wrapping benchmarks with adversarial layers.

---

## Refutations

### Refutation 1: "Pre-Yield Gate" Architecture is Misdesigned

Target Claim/Proposal: Section 4.1 proposes intercepting `yield` via `tool_call` hook to inject pyramid verification: `on("tool_call", async (event) => { if (event.toolName !== "yield") return; ... return { block: true, reason: pyramidResult.feedback }; });`

Original Confidence: VERIFIED (claim about hook system)

Verdict: ✗ REFUTED

Reason: The `yield` tool is NOT a general-purpose "task completion" signal for main sessions. `yield` is exclusively a subagent/task tool — it exists only to return structured results from child agents spawned by the `task` tool. Main agent sessions do NOT call `yield` to signal completion; they simply stop generating (the `agent_end` event fires, then `session_stop` fires). The proposal's entire "pre-yield gate" architecture targets the wrong mechanism for main-session verification gating.

The correct interception point for "before the agent declares done" in a main session is the `session_stop` event, which fires when the main agent settles and CAN return `{ continue: true, additionalContext }` to force a continuation turn. But `session_stop` has a hard cap of 8 consecutive continuations (SESSION_STOP_CONTINUATION_CAP), explicitly never fires for task/subagent sessions, and its continuation mechanism is designed for one-shot corrections — NOT iterating through a 6-layer pyramid.

Counter-Evidence:
- `/tmp/oh-my-pi/packages/coding-agent/src/tools/yield.ts:210` — `readonly name = "yield"` is a `YieldTool` class for subagent result submission
- `/tmp/oh-my-pi/packages/coding-agent/src/session/agent-session.ts:6317` — `if (this.#agentKind === "sub" || !this.#extensionRunner?.hasHandlers("session_stop")) { return false; }` — session_stop explicitly excluded for subagents
- `/tmp/oh-my-pi/packages/coding-agent/src/extensibility/shared-events.ts:359-368` — `SessionStopEventResult` only allows `continue?: boolean` + `additionalContext?: string` or `decision?: "block"` + `reason?: string`
- `/tmp/oh-my-pi/docs/extensions.md:246` — `session_stop` "capped at 8 consecutive continuations and never fires for task/subagent sessions"

---

### Refutation 2: Extensions Cannot Inject Advice into Advisor Channel

Target Claim/Proposal: Section 4.4 proposes trace behavioral assertions via an extension that calls `ctx.advise(v.message, v.severity)` from a `tool_result` handler: `on("tool_result", (event, ctx) => { ... ctx.advise(v.message, v.severity); })`

Original Confidence: VERIFIED (claim about extension hook system)

Verdict: ✗ REFUTED

Reason: The `ExtensionContext` interface does NOT expose any `advise()` method. Only the advisor's own `advise` tool (internal to the advisor Agent instance) can inject advice into the primary transcript. Extensions can `sendMessage()` (which steers/follows-up as a user-role prompt) or `sendUserMessage()`, but neither produces the `<advisory>` element structure with severity semantics that the proposal relies on.

The `enqueueAdvice` method exists only on `AdvisorRuntimeHost` — an internal interface used by `AgentSession` to route advisor output to the primary transcript. It is not exposed to extensions. An extension attempting to surface evaluation findings would need to use `pi.sendMessage(...)` with `deliverAs: "steer"`, which enters as a user message, not an advisory — the agent treats it as instruction, not as "weigh, don't blindly obey" guidance.

Counter-Evidence:
- `/tmp/oh-my-pi/packages/coding-agent/src/extensibility/extensions/types.ts:1191-1201` — `ExtensionAPI` exposes `sendMessage` and `sendUserMessage` only; no `advise` method
- `/tmp/oh-my-pi/packages/coding-agent/src/advisor/runtime.ts:32` — `enqueueAdvice` is on `AdvisorRuntimeHost` interface, internal to session wiring
- Grep for `advise|enqueueAdvice` in `/tmp/oh-my-pi/packages/coding-agent/src/extensibility/` returns zero matches

---

### Refutation 3: TTSR Does NOT Support "Trace-Level Behavioral Assertions"

Target Claim/Proposal: Section 4.4 and Gap 6 propose building "trace behavioral assertions" (e.g., "edit(path=$P) must be preceded by read(path=$P)") and claim TTSR's "scope system" can be extended to "diff-aware constraint DSL" and temporal assertions over tool call sequences.

Original Confidence: VERIFIED (gap identification) / INFERRED (buildability)

Verdict: ✗ REFUTED

Reason: TTSR operates on the **streaming token output** of the LLM — it matches regex patterns against the text being generated in real-time (text deltas, thinking deltas, tool-call argument deltas). It does NOT have access to the historical tool call trace. It cannot express temporal relationships like "X must precede Y" because:

1. TTSR buffers are reset on every `turn_start` (`resetBuffer()`) — there is no cross-turn memory
2. TTSR matches against the current stream buffer content (text being generated NOW), not against a recorded history of past tool executions
3. TTSR's only "memory" is the injection record (`lastInjectedAt` for repeat gating) — it has no concept of tool call ordering or historical trace
4. The scope system (`allowText`, `allowThinking`, `allowAnyTool`, `toolScopes`) filters which STREAM is monitored — not which past events to analyze

Building temporal trace assertions requires an entirely new subsystem. The extension `tool_result` event DOES provide post-hoc access to individual tool completions, but assembling a trace requires the extension to maintain its own state buffer across turns — TTSR provides zero infrastructure for this.

Counter-Evidence:
- `/tmp/oh-my-pi/packages/coding-agent/src/export/ttsr.ts:558-560` — `resetBuffer(): void { this.#buffers.clear(); this.#lastAstSnapshots.clear(); }` — called on turn_start, no cross-turn persistence
- `/tmp/oh-my-pi/packages/coding-agent/src/export/ttsr.ts:354-365` — `checkDelta` appends to per-stream-key buffers and regex-matches the current buffer only
- `/tmp/oh-my-pi/docs/ttsr-injection-lifecycle.md:69-70` — "On `turn_start`, the stream buffer is reset"
- `/tmp/oh-my-pi/docs/ttsr-injection-lifecycle.md:77-80` — monitors `text_delta`, `thinking_delta`, and `toolcall_delta` — not completed tool results or historical traces

---

### Refutation 4: Metaharness Has No Pluggable Adapter Wrapping System

Target Claim/Proposal: Section 5.6 proposes an "adversarial adapter that wraps any other adapter" via `readAdversarialSnapshot(jobDir, baseAdapter)` that extends `BenchmarkSnapshot` with `adversarial_survival_rate` and `mutation_kill_rate` metrics.

Original Confidence: VERIFIED (claim about adapter system)

Verdict: ✗ REFUTED

Reason: The metaharness benchmark adapter system is a fixed enum (`BenchmarkKind = "harbor" | "edit" | "snapcompact"`) with hard-coded `BENCHMARK_DEFINITIONS` and a `readBenchmarkSnapshot` function that dispatches on the kind. There is no adapter registration API, no plugin system, and no composability (wrapping one adapter with another). Adding a new benchmark kind requires modifying the source code of the metaharness package itself.

Furthermore, `BenchmarkSnapshot.metrics` is typed as `Record<string, number | null>` which CAN hold arbitrary keys — but the `MetricDefinition[]` in `BENCHMARK_DEFINITIONS` controls what the dashboard displays. Adding `adversarial_survival_rate` requires modifying the `BENCHMARK_DEFINITIONS` array, not just returning extra data.

Counter-Evidence:
- `/tmp/oh-my-pi/packages/metaharness/src/store.ts:19` — `export type BenchmarkKind = "harbor" | "edit" | "snapcompact"` — closed union, no extension point
- `/tmp/oh-my-pi/packages/metaharness/src/benchmarks.ts:23-45` — `BENCHMARK_DEFINITIONS` is a hard-coded const array with three entries
- `/tmp/oh-my-pi/packages/metaharness/src/benchmarks.ts:60-73` — `BenchmarkSnapshot` is a plain data interface with no adapter composition hooks

---

## Unverifiable Claims

### Unverifiable 1: "Rasch-calibrated projections" Enabling "Differential Evaluation Framework"

Target Claim/Proposal: Section 2 (Layer 5) claims metaharness implements "a sophisticated differential evaluation framework" with "Rasch-style item-response modeling" suitable for "formal differential evaluation."

Original Confidence: VERIFIED

Verdict: ? UNVERIFIABLE

Reason: The `calibratedFinalPassPct` function does exist and does implement a one-parameter logistic model with bisection fitting — this is correctly described. However, calling it a "differential evaluation framework" overstates its scope. It is a **projection utility** for running experiments — it projects incomplete arms' final pass rates. It does NOT:
- Compare two implementations for behavioral agreement (differential testing)
- Generate shared test inputs to probe disagreements
- Report WHERE two arms disagree (only aggregate projected pass%)

The function is better characterized as "IRT-inspired extrapolation for experiment arms still in flight" — useful for experiment monitoring, but not a differential evaluation framework as the testing literature defines it.

Counter-Evidence:
- `/tmp/oh-my-pi/packages/metaharness/src/experiments.ts:129-189` — The function signature takes `decided`, `siblings`, `remaining`, `nTotal` and returns a single `number | null` (projected pass%). No comparison output, no disagreement localization.

---

### Unverifiable 2: RunRole "baseline" | "variant" Already Supporting Automated Regression Detection

Target Claim/Proposal: Section 4.5 claims "The metaharness `store.ts` already has `RunRole = 'baseline' | 'variant'` — the missing piece is automated statistical comparison and alerting."

Original Confidence: INFERRED

Verdict: ? UNVERIFIABLE

Reason: While `RunRole` does exist as `"baseline" | "variant" | ""`, it is used purely for **display ordering** in experiment views (`roleRank` in `experimentDetail` sorts baselines before variants). No code anywhere in metaharness consumes the `role` field for comparison logic, significance testing, or regression detection. The proposal's claim that "the missing piece is automated statistical comparison" is technically true but misleading — what's missing is the ENTIRE regression detection system, not just one piece. The role field is cosmetic metadata, not a statistical comparison hook.

Counter-Evidence:
- `/tmp/oh-my-pi/packages/metaharness/src/experiments.ts:372` — `const roleRank = (role: string) => (role === "baseline" ? 0 : role === "variant" ? 1 : 2)` — used only for sort order
- `/tmp/oh-my-pi/packages/metaharness/src/store.ts:35` — `role: RunRole` field in `RunRow` — no code reads it for comparison purposes beyond display

---

## Missing Coverage

The proposal does not analyze:

1. **Extension lifecycle constraints**: Extensions are loaded once at session start. The proposal assumes extensions can dynamically orchestrate multi-step evaluation pipelines (run LSP → run tests → run advisor) within a single `tool_call` or `session_stop` handler, but doesn't address whether blocking in handlers stalls the agent loop. The `session_stop` handler has a 30-second timeout on each continuation, limiting complex orchestration.

2. **Tool execution isolation**: The proposal assumes evaluation extensions can invoke arbitrary tools (LSP diagnostics, bash test execution) from within hook handlers. But `ExtensionContext` does NOT expose the tool registry or a way to invoke tools programmatically. Extensions can only `exec()` shell commands (via `pi.exec(...)`) — they cannot call `lsp diagnostics` or other agent tools directly.

3. **Advisor turn frequency constraints**: The proposal's "multi-advisor adversarial panel" ignores `advisor.immuneTurns` (default 3). After ANY advisor interrupts with a concern/blocker, ALL subsequent interrupts are suppressed for 3 turns. With 4+ adversarial advisors, most findings would be rate-limited out.

4. **Metaharness is standalone**: The metaharness package runs as a separate process managing benchmark jobs. It is NOT embedded in the coding-agent runtime. Extending it with "adversarial probing" at trial-verification time would require the metaharness runner to spawn additional agent instances — infrastructure that doesn't exist in its current `runner.ts`.

---

## Survived Scrutiny

1. **LSP diagnostics as verification oracle** (Section 1.1) — Correctly described. `/tmp/oh-my-pi/docs/tools/lsp.md` confirms workspace/per-file diagnostics with binary OK/error-list outcomes.

2. **Extension tool_call blocking** (Section 1.7) — Correctly described. `/tmp/oh-my-pi/packages/coding-agent/src/extensibility/extensions/wrapper.ts:206-228` confirms `tool_call` event can return `{ block: true, reason }` to prevent execution.

3. **TTSR as pattern-based invariant guard** (Section 2, Layer 2) — Correctly described for its actual capability: regex/AST-grep matching on streaming output with abort-and-retry. The proposal is accurate when describing what TTSR IS; it only fails when extending TTSR to things it isn't (trace assertions).

4. **Advisor structural independence** (Section 5.1) — Correctly described. `/tmp/oh-my-pi/packages/coding-agent/src/advisor/runtime.ts` and `/tmp/oh-my-pi/docs/advisor-watchdog.md:75-86` confirm independent Agent instance, independent ToolSession, configurable tool grants.

5. **WATCHDOG.yml multi-advisor roster** (Section 5.5 base capability) — Correctly described. `/tmp/oh-my-pi/docs/advisor-watchdog.md:208-237` confirms per-advisor name/model/tools/instructions. The concept of specialized adversary advisors is feasible within the existing WATCHDOG.yml system — the proposal's basic "adversarial test-generation advisor" design is sound IF one accepts the rate-limiting constraints from `immuneTurns`.
