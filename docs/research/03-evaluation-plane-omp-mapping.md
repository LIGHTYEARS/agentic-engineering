# Evaluation Plane — oh-my-pi Architecture Mapping

## 1. Inventory of Evaluation/Verification Mechanisms in oh-my-pi

### 1.1 LSP Diagnostics (Static Analysis Layer)

```
Claim: oh-my-pi exposes full Language Server Protocol diagnostics as a first-class verification primitive.
Rationale: The `lsp` tool with `action: "diagnostics"` provides workspace-wide type checking (tsc --noEmit, cargo check, go build, pyright) and per-file linting via Biome/SwiftLint/custom LSP linters.
Evidence: /tmp/oh-my-pi/docs/tools/lsp.md lines 68-84 — workspace diagnostics runs cargo/tsc/go/pyright, per-file diagnostics queries real LSP servers and custom linter clients.
Confidence: VERIFIED
```

```
Claim: LSP diagnostics serve as a deterministic verification oracle — "OK" vs structured error list with line/col/severity.
Rationale: Single-file diagnostics return "OK" when clean or a grouped diagnostic list with severity sorting. This binary outcome can be used as a pass/fail gate.
Evidence: /tmp/oh-my-pi/docs/tools/lsp.md lines 78-83 — "Single target with no issues: OK" vs "summary + grouped diagnostics".
Confidence: VERIFIED
```

### 1.2 Bash Tool (Test Execution Surface)

```
Claim: The bash tool is the primary mechanism for executing test suites, with support for synchronous, async, PTY, and auto-background modes.
Rationale: Test runners (bun test, pytest, go test, cargo test) are invoked via the bash tool with configurable timeouts up to 3600s.
Evidence: /tmp/oh-my-pi/docs/tools/bash.md lines 122-129 — timeout clamp 1..3600s, default 300s. Exit code is returned in details.exitCode.
Confidence: VERIFIED
```

```
Claim: Bash has an interceptor system that can block patterns and redirect to better tools, functioning as a lightweight policy enforcement layer.
Rationale: checkBashInterception() applies regex rules to commands before execution, enabling allowlist/blocklist patterns for evaluation contexts.
Evidence: /tmp/oh-my-pi/packages/coding-agent/src/tools/bash-interceptor.ts lines 37-60 — regex-based command blocking with redirect messaging.
Confidence: VERIFIED
```

### 1.3 Eval Tool (Sandboxed Code Execution Kernels)

```
Claim: The eval tool provides persistent, cell-based Python and JavaScript runtimes with structured output capture (display(), images, JSON) suitable for programmatic verification.
Rationale: Both runtimes persist state across cells, support tool.() proxies for calling any agent tool, and capture structured JSON outputs for verification.
Evidence: /tmp/oh-my-pi/docs/tools/eval.md lines 24-53 — cells array with per-cell language, timeout, reset. Lines 130-137 — prelude installs display, read, write, tool proxy, agent() and parallel().
Confidence: VERIFIED
```

```
Claim: Eval kernels can spawn subagents (agent()) and run parallel evaluation pipelines (parallel(), pipeline()), enabling multi-agent evaluation orchestration.
Rationale: The agent() helper spawns subagents through the same executor used by the task tool, with depth-3 recursion cap and parallel pool bounded by task.maxConcurrency.
Evidence: /tmp/oh-my-pi/docs/tools/eval.md lines 189-203 — agent() spawns via runSubprocess, parallel() uses bounded pool, pipeline() chains stages.
Confidence: VERIFIED
```

### 1.4 Debug Tool (Runtime Verification via DAP)

```
Claim: The debug tool provides full Debug Adapter Protocol integration for runtime assertion, memory inspection, and program state verification.
Rationale: Supports launch/attach, breakpoints (source, function, instruction, data), expression evaluation in REPL context, stack inspection, memory read/write, and disassembly.
Evidence: /tmp/oh-my-pi/docs/tools/debug.md lines 29-80 — full action enum with evaluate, stack_trace, variables, read_memory, disassemble.
Confidence: VERIFIED
```

### 1.5 Advisor/Watchdog (Adversarial Review)

```
Claim: The advisor system provides a structurally independent review agent that watches the primary agent's output and can interrupt with concern/blocker severity.
Rationale: The advisor is a separate Agent instance with its own ToolSession, model, and tool grants. It reviews each primary turn's delta and delivers advice through a steering channel.
Evidence: /tmp/oh-my-pi/docs/advisor-watchdog.md lines 1-10 — "reviews the primary agent's transcript after each turn, inspects the workspace with its own tools". Lines 88-95 — severity levels nit/concern/blocker with interruption semantics.
Confidence: VERIFIED
```

```
Claim: WATCHDOG.yml supports multi-advisor rosters with per-advisor specialization, tool grants, and model selection — enabling multi-perspective adversarial evaluation.
Rationale: Each entry in WATCHDOG.yml gets its own name, model, tools, and instructions. The "Fixer" example shows an advisor with edit+bash that can "run tests to prove a fix locally".
Evidence: /tmp/oh-my-pi/docs/advisor-watchdog.md lines 208-237 — WATCHDOG.yml schema with advisors[].name/model/tools/instructions.
Confidence: VERIFIED
```

### 1.6 TTSR (Time-Traveling Stream Rules)

```
Claim: TTSR provides real-time pattern matching against the agent's streaming output, with abort-and-retry semantics that function as invariant guards.
Rationale: When a TTSR condition matches (regex or ast-grep), the stream is aborted, the rule body is injected as correction guidance, and the request is retried.
Evidence: /tmp/oh-my-pi/packages/coding-agent/src/export/ttsr.ts lines 1-7 — "when condition pattern matches the agent's output, stream is aborted, rule injected, request retried." Lines 55-63 — settings with repeatMode, interruptMode.
Confidence: VERIFIED
```

### 1.7 Extension Tool Wrapper (Interception & Gating)

```
Claim: The ExtensionToolWrapper provides pre/post tool execution hooks that can block, modify, or observe tool calls — enabling evaluation gates.
Rationale: Every tool call passes through the wrapper which emits tool_call events (can return {block: true}), checks approval policies, and emits tool_result events (can modify results).
Evidence: /tmp/oh-my-pi/packages/coding-agent/src/extensibility/extensions/wrapper.ts lines 86-90 — "Emits tool_call event before execution (can block), tool_result event after (can modify result)".
Confidence: VERIFIED
```

### 1.8 TypeScript Edit Benchmark

```
Claim: oh-my-pi has a dedicated edit verification system with byte-for-byte comparison, formatted-equivalence checking, and indent-distance scoring.
Rationale: The typescript-edit-benchmark package verifies edits against expected fixtures using normalized formatting comparison, produces compact diffs, and computes indentation distance scores.
Evidence: /tmp/oh-my-pi/packages/typescript-edit-benchmark/src/verify.ts lines 1-20 — VerificationResult with success, diff, diffStats, indentScore, formattedEquivalent. Lines 78-178 — full verification pipeline.
Confidence: VERIFIED
```

### 1.9 Metaharness (Meta-Evaluation Infrastructure)

```
Claim: Metaharness is a complete benchmark orchestration system supporting multiple benchmark types (Harbor/edit/snapcompact), experiment grouping, arm comparison, and live dashboard monitoring.
Rationale: Manages runs in SQLite, normalizes traces across benchmark adapters, supports experiment arms with calibrated projections, and exposes a REST+SSE API with web dashboard.
Evidence: /tmp/oh-my-pi/packages/metaharness/src/benchmarks.ts lines 23-45 — BENCHMARK_DEFINITIONS with harbor/edit/snapcompact. /tmp/oh-my-pi/packages/metaharness/src/experiments.ts lines 1-50 — experiment arms with projections. /tmp/oh-my-pi/packages/metaharness/src/server.ts lines 1-24 — REST API surface.
Confidence: VERIFIED
```

```
Claim: Metaharness implements trace-based evaluation with model-generated narrative reports using a two-stage map/reduce pipeline.
Rationale: The trace-report script maps each assistant turn through a cheap model for grounded descriptions, then reduces the Turn Log to a Story Arc with failure analysis.
Evidence: /tmp/oh-my-pi/packages/metaharness/scripts/trace-report.ts lines 1-28 — "Two-stage map/reduce over normalized trace JSON". Lines 42-68 — TraceEntry types for assistant/toolResult/notice.
Confidence: VERIFIED
```

### 1.10 Approval System (Execution Gating)

```
Claim: The tool approval system provides tiered execution control (always-ask/write/yolo) with per-tool deny/allow/prompt policies, functioning as a risk-graded verification gate.
Rationale: resolveApproval() checks the tool's approval tier against the configured mode and per-tool user policies before execution proceeds.
Evidence: /tmp/oh-my-pi/packages/coding-agent/src/extensibility/extensions/wrapper.ts lines 123-137 — approval resolution with deny/prompt/allow outcomes. /tmp/oh-my-pi/packages/coding-agent/src/config/settings-schema.ts lines 3433-3443.
Confidence: VERIFIED
```

---

## 2. Mapping to the 6-Layer Evaluation Pyramid

### Layer 1: Capability Contract Tests
*"Does the system have the declared capabilities?"*

| OMP Primitive | Mapping | Quality |
|---|---|---|
| LSP `status` / `capabilities` | Verifies language server availability and declared protocol support | **Strong** — binary presence check |
| Bash (exit code check) | Verifies build/compile success | **Strong** — deterministic |
| Eval (kernel availability) | Python/JS runtime availability check | **Strong** — `resolveBackend()` throws on unavailable |

```
Claim: Layer 1 is well-served by existing primitives — LSP capabilities, bash build checks, and eval backend resolution provide deterministic capability verification.
Rationale: All three mechanisms produce binary pass/fail outcomes that require no judgment.
Evidence: LSP capabilities dumps JSON of server capabilities. Bash exit code 0/non-0. Eval resolveBackend throws ToolError on missing backends.
Confidence: VERIFIED
```

### Layer 2: Invariant Tests
*"Are structural properties preserved across changes?"*

| OMP Primitive | Mapping | Quality |
|---|---|---|
| LSP `diagnostics` (workspace) | Type invariants via tsc/cargo/go/pyright | **Strong** — zero-error = invariant holds |
| TTSR rules (regex + ast-grep) | Pattern-based invariant guards on agent output | **Strong** — abort-on-violation semantics |
| Bash interceptor | Command-level invariant enforcement | **Moderate** — blocking only, no verification |
| Auto-generated file guard | Invariant: generated files stay untouched | **Strong** — `assertEditableFile()` blocks edits |

```
Claim: TTSR is the most novel primitive for invariant enforcement — it operates on the streaming output, not post-hoc, enabling "shift-left" invariant verification during generation.
Rationale: Unlike traditional testing which verifies after the fact, TTSR matches patterns in the agent's token stream and aborts generation immediately, preventing invariant-violating code from ever being committed.
Evidence: /tmp/oh-my-pi/packages/coding-agent/src/export/ttsr.ts lines 4-6 — "stream is aborted, rule injected as system reminder, request retried." Scope filtering allows tool:edit(*.rs) patterns.
Confidence: VERIFIED
```

### Layer 3: Scenario Matrix Tests
*"Do representative input/output pairs produce correct results?"*

| OMP Primitive | Mapping | Quality |
|---|---|---|
| Edit benchmark (verify.ts) | Fixture-based golden file comparison with formatting normalization | **Strong** — exact expected output |
| Metaharness Harbor runner | Terminal-bench task suite with reward-based scoring | **Strong** — standardized benchmark |
| Eval + agent() | Programmatic scenario execution with structured output validation | **Strong** — JSON schema validation on output |

```
Claim: The edit benchmark system implements "formatted equivalence" — a novel relaxation of byte-for-byte comparison that normalizes whitespace/formatting while still catching semantic errors.
Rationale: This is practically ideal for code generation evaluation: it forgives formatting differences while detecting any meaningful content divergence.
Evidence: /tmp/oh-my-pi/packages/typescript-edit-benchmark/src/verify.ts lines 135-161 — formats both expected and actual through the same formatter, compares post-format versions, but also computes an indentScore measuring raw formatting distance.
Confidence: VERIFIED
```

### Layer 4: Metamorphic Tests
*"Do transformations preserve semantic meaning?"*

| OMP Primitive | Mapping | Quality |
|---|---|---|
| None directly | **Gap** | — |
| Eval + parallel() | *Could* implement transformation + verification pairs | **Potential** |

```
Claim: oh-my-pi has no built-in metamorphic testing infrastructure, but the eval tool's parallel() and pipeline() helpers provide the execution substrate for implementing it.
Rationale: Metamorphic testing requires applying semantics-preserving transformations and verifying output equivalence. The eval tool can orchestrate this but has no metamorphic-specific framework.
Evidence: No grep results for "metamorphic" in the codebase. Eval's parallel() supports running transformed variants concurrently.
Confidence: INFERRED
```

### Layer 5: Differential Tests
*"Do two implementations agree on behavior?"*

| OMP Primitive | Mapping | Quality |
|---|---|---|
| Metaharness experiments (arms) | Multi-arm experiment comparison with Rasch-calibrated projections | **Strong** — statistically grounded |
| Advisor (multi-advisor roster) | Multiple models reviewing same output | **Moderate** — not structured as differential |
| Eval agent() + parallel() | Compare outputs from multiple models/approaches | **Potential** |

```
Claim: Metaharness implements a sophisticated differential evaluation framework through its experiment system, with per-task difficulty calibration using Rasch-style item-response modeling.
Rationale: The calibratedFinalPassPct function uses sibling arm outcomes as per-task difficulty signals, fits a one-parameter logistic model, and projects remaining performance — this is formally grounded differential evaluation.
Evidence: /tmp/oh-my-pi/packages/metaharness/src/experiments.ts lines 130-190 — calibratedFinalPassPct with sigma/logit, bisection fit, smoothed difficulty estimates.
Confidence: VERIFIED
```

### Layer 6: Rubric-Based Evaluation
*"Does the output satisfy qualitative criteria judged by model or expert?"*

| OMP Primitive | Mapping | Quality |
|---|---|---|
| Advisor system | Model-as-judge with configurable review priorities | **Strong** — structured severity output |
| Trace-report (metaharness) | LLM-generated narrative evaluation with failure analysis | **Moderate** — narrative, not structured rubric |
| WATCHDOG.md instructions | Domain-specific review criteria injected into advisor | **Strong** — project-specific quality bars |

```
Claim: The advisor system is structurally a rubric-based evaluator — WATCHDOG.md defines review criteria, and the advisor produces severity-graded assessments.
Rationale: The advisor receives project-specific "watch for" instructions, reviews against them, and produces nit/concern/blocker outcomes — directly analogous to a rubric evaluation.
Evidence: /tmp/oh-my-pi/docs/advisor-watchdog.md lines 148-165 — WATCHDOG.md defines review priorities. Lines 88-95 — three severity levels map to rubric grades.
Confidence: VERIFIED
```

---

## 3. Gap Analysis

### Gap 1: No Formal Test Pyramid Orchestrator

```
Claim: oh-my-pi has no mechanism to compose verification layers into an ordered pyramid where earlier (cheaper) layers gate later (more expensive) ones.
Rationale: Each verification mechanism (LSP, bash tests, advisor) operates independently. There is no framework that says "run LSP diagnostics first; only if clean, run unit tests; only if passing, run integration tests."
Evidence: No orchestration layer found linking lsp diagnostics → bash test → advisor review in a gated sequence. Each tool is invoked independently by the agent.
Confidence: VERIFIED
```

### Gap 2: No Metamorphic Testing Framework

```
Claim: There is no built-in infrastructure for generating semantics-preserving input transformations and verifying output equivalence.
Rationale: Metamorphic testing (e.g., "rename a variable and verify the same test still passes") requires transformation generators and equivalence oracles — neither exists.
Evidence: No metamorphic-related patterns found in codebase search. The eval tool provides execution substrate but no metamorphic abstractions.
Confidence: VERIFIED
```

### Gap 3: No Deterministic Diff Verification Protocol

```
Claim: While the edit benchmark has fixture-based diff verification, there is no general-purpose protocol for verifying that an agent's changes satisfy a specification expressed as constraints on the diff itself.
Rationale: Diff verification (e.g., "this change must only touch files in src/auth/", "no function should grow by more than 20 lines") requires a diff-aware constraint language that doesn't exist.
Evidence: The edit benchmark verifies against golden fixtures (expected final state), not diff-level constraints. No diff-constraint DSL found.
Confidence: VERIFIED
```

### Gap 4: No Adversarial Input Generation

```
Claim: The advisor system reviews agent output but does not generate adversarial test inputs designed to break the agent's solution.
Rationale: True adversarial evaluation requires generating edge cases, boundary conditions, and attack inputs specific to the agent's code — the advisor only observes and critiques.
Evidence: /tmp/oh-my-pi/docs/advisor-watchdog.md — advisor "reviews the primary agent's transcript" but has no protocol for generating test inputs that probe weaknesses.
Confidence: VERIFIED
```

### Gap 5: No Regression Detection Across Sessions

```
Claim: While metaharness tracks benchmark results over time, there is no automatic regression detection that alerts when a new model/config performs worse than a previous baseline on the same tasks.
Rationale: Regression detection requires storing baselines and comparing new results against them with statistical significance testing. Metaharness stores results but doesn't implement automated alerting.
Evidence: /tmp/oh-my-pi/packages/metaharness/src/store.ts stores RunRow with scores but no baseline comparison or alerting logic found. Experiment arms allow manual comparison but no automated regression gates.
Confidence: INFERRED
```

### Gap 6: No Trace-Level Behavioral Assertions

```
Claim: There is no mechanism to assert behavioral properties of agent traces (e.g., "the agent must read the file before editing it", "no more than 3 retries on a failed test").
Rationale: Trace-based eval requires a temporal logic or assertion language over the sequence of tool calls. The trace-report script narrates traces but doesn't verify them against behavioral specs.
Evidence: Metaharness trace-report generates narrative descriptions but no assertion framework. No temporal logic patterns found.
Confidence: VERIFIED
```

---

## 4. Proposal: Complete Evaluation System Using oh-my-pi Primitives

### 4.1 Evaluation Pyramid Orchestrator (Extension)

**Mechanism**: A new oh-my-pi extension that implements a gated evaluation pipeline.

**Architecture**:
```
Extension: evaluation-pyramid
├── hooks/
│   └── post-edit.ts       — triggers Layer 1-2 after any edit/write
├── tools/
│   └── verify.ts          — orchestrates full pyramid on demand
└── config/
    └── pyramid.yml        — layer definitions and gate conditions
```

**Design**:
- **Layer 1** (capability): LSP `diagnostics file:"*"` — must return "OK" or zero errors
- **Layer 2** (invariant): TTSR scan over changed files + custom invariant assertions
- **Layer 3** (scenario): Bash test execution with exit-code gating
- **Layer 4** (metamorphic): Eval-driven transformation + re-verification
- **Layer 5** (differential): Multi-model comparison via parallel agent() calls
- **Layer 6** (rubric): Advisor assessment with structured scoring

**Key insight**: Use the `tool_call` hook to intercept `yield` (completion) and inject pyramid verification before the agent can declare "done". This leverages the existing hook infrastructure:

```typescript
// hooks/pre-yield.ts
on("tool_call", async (event) => {
  if (event.toolName !== "yield") return;
  const pyramidResult = await runPyramid(event.context);
  if (!pyramidResult.passed) {
    return { block: true, reason: pyramidResult.feedback };
  }
});
```

### 4.2 Diff Constraint Verification (Tool)

**Mechanism**: A new tool that expresses and verifies constraints on git diffs.

**Design**:
```yaml
# .omp/rules/diff-constraints.md
---
condition: ["yield", "checkpoint"]
---
Verify the current diff satisfies:
- Only files matching `src/auth/**` are modified
- No file grows by more than 50 lines net
- All new functions have at least one test reference
```

**Implementation**: Use `eval` tool to compute `git diff --stat`, parse results, and apply declarative constraints. TTSR's ast-grep conditions already provide AST-level matching — extend this to diff-level AST assertions.

### 4.3 Metamorphic Testing Engine (Eval Pipeline)

**Mechanism**: An eval-based pipeline that generates semantics-preserving transformations and verifies output equivalence.

**Design** (using existing eval primitives):
```python
# Metamorphic test: variable rename should not change behavior
transformations = [
    {"type": "rename_var", "from": "result", "to": "output"},
    {"type": "reorder_imports"},
    {"type": "extract_constant"},
]
for t in transformations:
    transformed = apply_transform(original_code, t)
    assert run_tests(transformed) == run_tests(original_code)
```

**Key insight**: Use `agent()` subagents to apply transformations (they have full edit/write access), then verify with bash test execution. The parallel() helper enables concurrent metamorphic checks.

### 4.4 Trace Behavioral Assertions (Extension)

**Mechanism**: A post-execution hook that verifies temporal properties of the agent's tool call trace.

**Design**:
```yaml
# .omp/eval/trace-assertions.yml
assertions:
  - name: read-before-edit
    pattern: "edit(path=$P) must be preceded by read(path=$P)"
    severity: blocker
  - name: retry-budget
    pattern: "bash(command~test) failures <= 3 consecutive"
    severity: concern
  - name: no-blind-writes
    pattern: "write(path=$P) must be preceded by read(path=$P) OR glob(path~$P)"
    severity: concern
```

**Implementation**: The extension hook system already captures every tool_call and tool_result event. A post-turn hook can analyze the accumulated trace against a pattern language:

```typescript
// Extension: trace-assertions
on("tool_result", (event, ctx) => {
  traceBuffer.push({ tool: event.toolName, args: event.args, result: event.result });
  const violations = checkAssertions(traceBuffer, config.assertions);
  for (const v of violations) {
    ctx.advise(v.message, v.severity); // Use advisor channel
  }
});
```

### 4.5 Regression Detection Service (Metaharness Extension)

**Mechanism**: Extend the metaharness store with baseline tracking and significance testing.

**Design**:
- Each experiment arm declares a `role: "baseline"` (already supported in RunRow)
- On run completion, compare variant scores against baseline using Wilson score intervals
- Emit alerts through the SSE event stream when a variant is significantly worse

**Key insight**: The metaharness `store.ts` already has `RunRole = "baseline" | "variant"` — the missing piece is automated statistical comparison and alerting.

---

## 5. Adversarial Evaluation: Extending Advisor/Watchdog

### 5.1 Current Adversarial Capabilities

```
Claim: The advisor system already has the structural prerequisites for adversarial evaluation — independent model, independent tool session, structured severity output, and steering-channel interruption.
Rationale: An adversarial evaluator needs: (1) independence from the system under test, (2) ability to probe the system, (3) ability to halt and redirect. The advisor has all three.
Evidence: /tmp/oh-my-pi/docs/advisor-watchdog.md — independent ToolSession (lines 75-86), steering channel interruption (lines 88-103), configurable tool grants including bash/eval (lines 83-86).
Confidence: VERIFIED
```

### 5.2 Adversarial Extension: Test-Generation Advisor

**Design**: A new WATCHDOG.yml advisor personality specialized for adversarial test generation:

```yaml
advisors:
  - name: Adversary
    model: anthropic/claude-sonnet-4-5:high
    tools: [read, grep, glob, bash, eval]
    instructions: |
      You are an adversarial test designer. After each edit by the primary agent:
      1. Read the changed code and its test file
      2. Generate edge-case inputs that might break the implementation
      3. Write and execute a minimal test script via eval/bash
      4. If the test fails, raise a blocker with the failing case
      
      Focus on:
      - Boundary conditions (empty, null, max-int, unicode)
      - Race conditions in concurrent code
      - Error paths that may not be tested
      - Invariant violations from partial updates
```

### 5.3 Adversarial Extension: Mutation Testing Advisor

**Design**: An advisor that applies mutations to the agent's code and checks if tests catch them:

```yaml
advisors:
  - name: MutationTester
    model: anthropic/claude-sonnet-4-5:medium
    tools: [read, grep, bash, eval]
    instructions: |
      After the primary agent writes or edits code:
      1. Read the changed code
      2. Identify logical mutations (negate conditions, off-by-one, remove error handling)
      3. Use eval to apply each mutation and run tests
      4. If tests pass despite a mutation, raise concern: "Mutation survived: [description]"
      
      This identifies undertested code paths. Only mutate the agent's new code.
```

### 5.4 Emission Guard as Evaluation Quality Filter

```
Claim: The AdvisorEmissionGuard provides deduplication, rate-limiting, and content-filtering for advisor outputs — essential for preventing evaluation noise.
Rationale: Without the guard, adversarial advisors could flood the primary agent with repetitive or content-free evaluations. The guard's normalization + dedupe + per-update limit ensures evaluation signal quality.
Evidence: /tmp/oh-my-pi/docs/advisor-watchdog.md lines 108-118 — content-free phrase filter, exact-text dedupe (FIFO ring of 4096), per-update rate limit.
Confidence: VERIFIED
```

### 5.5 Multi-Advisor Adversarial Panel

**Architecture**: Deploy multiple specialized adversaries as a panel, each watching different aspects:

```yaml
# .omp/WATCHDOG.yml
advisors:
  - name: TypeSafety
    model: anthropic/claude-sonnet-4-5:medium
    tools: [read, grep, glob]
    instructions: "Watch for type unsoundness, unsafe casts, any types"
    
  - name: SecurityReview
    model: anthropic/claude-sonnet-4-5:high
    tools: [read, grep, glob, bash]
    instructions: "Probe for injection, SSRF, path traversal, secret exposure"
    
  - name: EdgeCaseTester
    model: anthropic/claude-sonnet-4-5:high
    tools: [read, grep, glob, bash, eval]
    instructions: "Generate and execute edge-case tests for new code"
    
  - name: PerformanceWatch
    model: anthropic/claude-sonnet-4-5:medium
    tools: [read, grep, glob]
    instructions: "Flag O(n²) patterns, unbounded allocations, missing caches"
```

**Key insight**: Each advisor is independently bounded by `advisor.immuneTurns` (default 3 turns between interruptions), and the emission guard prevents cross-advisor noise accumulation. The `syncBacklog` setting controls how much the primary agent waits for the panel.

### 5.6 Adversarial Evaluation in Metaharness

**Mechanism**: Extend metaharness benchmarks with adversarial components:

1. **Pre-verification adversarial probing**: Before running the standard verifier, inject adversarial test cases generated by a separate model
2. **Post-verification mutation analysis**: After a trial passes, apply mutations and verify test coverage
3. **Cross-arm adversarial transfer**: Use failures from one experiment arm to generate targeted tests for other arms

**Implementation**: The metaharness adapter system (benchmarks.ts) already normalizes different benchmark types into a common `BenchmarkSnapshot`. Add an `adversarial` adapter that wraps any other adapter with adversarial extensions:

```typescript
// New adapter: adversarial wrapper
function readAdversarialSnapshot(jobDir: string, baseAdapter: BenchmarkKind): BenchmarkSnapshot {
  const base = readBenchmarkSnapshot(baseAdapter, jobDir);
  const adversarialTraces = readAdversarialProbes(jobDir);
  return {
    ...base,
    metrics: {
      ...base.metrics,
      adversarial_survival_rate: computeSurvivalRate(adversarialTraces),
      mutation_kill_rate: computeMutationKillRate(adversarialTraces),
    }
  };
}
```

---

## Summary of Extension Points

| Evaluation Need | oh-my-pi Primitive | Extension Required |
|---|---|---|
| Deterministic type checking | LSP diagnostics | None — ready |
| Test execution + gating | Bash tool + exit codes | Pyramid orchestrator extension |
| Pattern-based invariants | TTSR (regex + ast-grep) | Rule authoring only |
| Fixture comparison | typescript-edit-benchmark | Generalize beyond TypeScript |
| Multi-model differential | Metaharness experiments | Automated regression alerting |
| Rubric evaluation | Advisor + WATCHDOG.md | Structured scoring schema |
| Adversarial test generation | Advisor + bash/eval tools | WATCHDOG.yml personalities |
| Trace behavioral assertions | Hook system (tool_call/tool_result) | Temporal assertion language |
| Metamorphic testing | Eval parallel() + agent() | Transformation framework |
| Diff-level constraints | TTSR scope system | Diff-aware constraint DSL |
| Pre-yield verification gate | Hook system (pre-yield block) | Pyramid hook extension |
