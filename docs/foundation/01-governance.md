# Governance Plane — Confirmed Foundation

> Source: research/01 + rebuttals/01 + resolutions/01
> Funnel: 9 survived adversarial · 2 re-confirmed · 5 discarded · 1 open decision

## ✅ Confirmed buildable capabilities

### C-GOV-1: Pre-execution tool_call blocking (primary enforcement gate)
- **What it is**: Extensions intercept every tool invocation before execution; returning `{ block: true, reason }` prevents the call. Handler errors also block (fail-closed semantics).
- **oh-my-pi substrate**: `packages/coding-agent/src/extensibility/extensions/wrapper.ts:206-228` — `emitToolCall` emits event, checks `callResult?.block`, throws on block or handler error.
- **Provenance**: survived adversarial review

### C-GOV-2: Three-tier tool approval system
- **What it is**: Tools self-declare a tier (`read`/`write`/`exec`); user selects an approval mode (`always-ask`/`write`/`yolo`) that auto-approves up to a threshold. Per-tool overrides (`tools.approval.<tool>: allow|deny|prompt`) provide fine-grained policy. Dynamic escalation via `{ override: true }` forces prompt in non-yolo modes.
- **oh-my-pi substrate**: `packages/coding-agent/src/tools/approval.ts:28-38` — `TIER_RANK` and `APPROVAL_MODE_MAX_TIER`; `packages/coding-agent/src/config/settings-schema.ts:3417-3460` — settings definitions.
- **Provenance**: survived adversarial review

### C-GOV-3: Critical bash pattern detection
- **What it is**: Hardcoded risk-pattern array (`CRITICAL_BASH_PATTERNS`) matches destructive commands (rm -rf, fork bombs, dd, curl|bash, etc.) and returns `{ override: true }` to force approval even in permissive modes.
- **oh-my-pi substrate**: `packages/coding-agent/src/tools/bash.ts:82-128` — pattern definitions; lines 380-383 — override decision.
- **Provenance**: survived adversarial review

### C-GOV-4: Settings precedence chain (five-layer configuration governance)
- **What it is**: Settings merge from built-in defaults → global config (`~/.omp/agent/config.yml`) → project config (`<cwd>/.omp/config.yml`) → CLI overlays (`--config <file>`) → runtime overrides (`--approval-mode`, etc.), with objects deep-merged and arrays replaced. Governance policies can be enforced at any layer.
- **oh-my-pi substrate**: `docs/settings.md:92-107` — precedence documentation; `packages/coding-agent/src/config/settings-schema.ts` — schema definitions.
- **Provenance**: survived adversarial review

### C-GOV-5: session_stop gate (post-execution governance checkpoint)
- **What it is**: The `session_stop` extension event allows handlers to return `{ decision: "block", reason }` to prevent session delivery, or `{ continue: true, additionalContext }` to inject more work. Capped at 8 consecutive continuations.
- **oh-my-pi substrate**: `packages/coding-agent/src/extensibility/shared-events.ts:358-368` — `SessionStopEventResult` type definition.
- **Provenance**: survived adversarial review

### C-GOV-6: TTSR stream monitoring (real-time compliance detection)
- **What it is**: Rules with `condition`/`astCondition` fields are registered with TtsrManager and matched against streaming text/tool argument deltas. Matching triggers abort of the current generation + retry with injected system-interrupt message. Supports scope targeting, interrupt modes (`always`/`never`/`prose-only`/`tool-only`), repeat policies, and glob-gating.
- **oh-my-pi substrate**: `docs/ttsr-injection-lifecycle.md` — full lifecycle; `packages/coding-agent/src/export/ttsr.ts` — TtsrManager.
- **Scope limit**: TTSR does NOT block tool execution. It aborts the LLM generation stream and retries — if the tool call arguments are already fully produced, the tool may have dispatched before the abort fires. TTSR is advisory/corrective, not a pre-execution gate.
- **Provenance**: survived adversarial review

### C-GOV-7: Subagent budget controls
- **What it is**: Multi-dimensional resource controls for delegated work: `task.maxConcurrency` (default 32), `task.maxRecursionDepth` (default 2), `task.maxRuntimeMs` (default 0/unlimited), `task.softRequestBudget` (default 200 requests, hard stop at 1.5x).
- **oh-my-pi substrate**: `packages/coding-agent/src/config/settings-schema.ts:4160-4270` — all task.* settings.
- **Scope limit**: Budgets are request-count only, not token or monetary cost. No cross-session or organizational aggregation.
- **Provenance**: survived adversarial review

### C-GOV-8: Rule system (declarative policy framework)
- **What it is**: Rules discovered from multiple providers, normalized into canonical shape, and bucketed into TTSR (enforcement), always-apply (mandatory LLM context), or rulebook (advisory). Supports `alwaysApply`, `condition`, `astCondition`, `globs`, `scope`, `interruptMode`.
- **oh-my-pi substrate**: `docs/rulebook-matching-pipeline.md` — full pipeline; `packages/coding-agent/src/capability/rule.ts` — rule types; `packages/coding-agent/src/capability/rule-buckets.ts` — bucketing logic.
- **Provenance**: survived adversarial review

### C-GOV-9: Risk grading extension via tool_call hook
- **What it is**: A `tool_call` handler receiving `event.toolName`, `event.toolCallId`, `event.input` with access to `ctx.cwd` and `ctx.sessionManager` can implement arbitrary risk scoring logic and return `{ block: true }` to prevent high-risk operations.
- **oh-my-pi substrate**: `packages/coding-agent/src/extensibility/extensions/types.ts:1120` — handler type; `packages/coding-agent/src/extensibility/extensions/wrapper.ts:206-228` — enforcement.
- **Scope limit**: Extension must provide its own risk-scoring framework — no built-in risk matrix or scoring primitives exist.
- **Provenance**: survived adversarial review

### C-GOV-10: Session-file audit persistence via appendEntry
- **What it is**: `pi.appendEntry(customType, data)` writes structured records to the session JSONL file. Writes are synchronous to disk (durable the instant the call returns). Custom entries survive compaction — the `#entries` array is never pruned, and compaction only marks a `firstKeptEntryId` boundary for LLM context, not for file contents. Crash-durable (append-only JSONL, truncated last line is worst case).
- **oh-my-pi substrate**: `packages/coding-agent/src/session/session-manager.ts:837-841` — `#recordEntry` writes immediately; lines 704-712 — explicit durability comment ("durable the instant this returns"); lines 554-558 — `#fileBody` iterates all entries without pruning.
- **Scope limit**: No tamper-evident guarantees (file writable by same user). No cross-session aggregation. Session files can be manually deleted. Remote backend flushing at shutdown still subject to the 2-second `session_shutdown` timeout.
- **Provenance**: re-investigation CONFIRMED

### C-GOV-11: Declarative approval matrix via extension filesystem read at session_start
- **What it is**: An extension can encode an approval matrix as content in an `alwaysApply: true` rule file (`.omp/rules/approval-matrix.md`). At `session_start`, the extension reads the file from disk (via `fs.readFileSync` or `ctx.exec("cat", [...])`), parses the embedded matrix DSL, and enforces it via `tool_call` blocking. The `alwaysApply` frontmatter simultaneously injects the matrix as advisory LLM context — dual enforcement (mechanical + advisory).
- **oh-my-pi substrate**: `packages/coding-agent/src/extensibility/extensions/types.ts:1071` — `session_start` handler registration; `packages/coding-agent/src/extensibility/extensions/loader.ts:223-224` — `exec()` API; `packages/coding-agent/src/extensibility/extensions/types.ts:419-420` — `ctx.cwd` access.
- **Scope limit**: The `session_start` event itself carries no payload (empty `{ type: "session_start" }`). Matrix must be read from filesystem, not from event data.
- **Provenance**: re-investigation CONFIRMED

## ❌ Must build net-new (gaps)

### G-GOV-1: Formal multi-dimensional risk grading matrix
- **What's missing**: No built-in risk-scoring function that considers target scope, reversibility, blast radius, and historical context. Current system is binary (tier auto-approve or prompt) plus a fixed bash pattern list.
- **Evidence of absence**: `packages/coding-agent/src/tools/approval.ts` — only tier + override, no risk score; grep: zero matches for `riskScore|riskLevel|blastRadius` in settings-schema.ts.
- **Nature of build**: extension (composable risk evaluator chain implementable via `tool_call` hook)

### G-GOV-2: Role-based approval matrices (identity/delegation)
- **What's missing**: No agent identity or role concept. Approval resolution in `approval.ts` operates on a single `userConfig` record with no role/identity parameter. No delegation chains or per-agent permission subsets.
- **Evidence of absence**: `packages/coding-agent/src/tools/approval.ts` — no role parameter; grep: zero matches for `agentRole|delegation|roleMatrix` across src tree.
- **Nature of build**: extension (matrix enforcement via C-GOV-11 pattern) + potential core change for role identity propagation to subagents

### G-GOV-3: Token/monetary cost budget controls
- **What's missing**: Budget system counts assistant requests only (turns), not tokens consumed or monetary cost. No per-session token budget, no per-task cost ceiling, no organizational spending limits.
- **Evidence of absence**: `packages/coding-agent/src/config/settings-schema.ts:4242-4268` — `softRequestBudget` in request count only; `ContextUsage` (types.ts:345-351) exposes `tokens` (context window occupancy), NOT cumulative spend.
- **Nature of build**: extension (token tracking via turn_end + ctx.getContextUsage()) for context-window alerting; core change needed for true cumulative-spend tracking (requires provider-level metering not currently exposed)

### G-GOV-4: Filesystem/network boundary enforcement
- **What's missing**: No sandboxing. Bash runs with full user permissions. File read/write tools not restricted to project directory. `task.isolation` modes (overlayfs, rcopy, worktree) are merge strategies for subagent file changes, not access control.
- **Evidence of absence**: `packages/coding-agent/src/config/settings-schema.ts:4039-4060` — isolation modes; grep: zero matches for `chroot|namespace|sandbox|networkPolicy` in src.
- **Nature of build**: core change (requires OS-level sandboxing or container integration — TTSR regex matching on path strings is advisory only, trivially evadable)

### G-GOV-5: Structured governance audit log with cross-session aggregation
- **What's missing**: While per-session audit persistence works (C-GOV-10), there is no cross-session aggregation, no queryable governance decision history, and no tamper-evident storage. No dedicated governance event schema beyond raw custom entries.
- **Evidence of absence**: grep: zero matches for `auditStore|auditLog|complianceLog|governanceLog` in src; session files are isolated per-session JSONL.
- **Nature of build**: greenfield subsystem (aggregation layer over session files, or dedicated append-only governance store)

### G-GOV-6: Multi-party approval workflow
- **What's missing**: Single-user approval only. Extension handler timeout is 30 seconds (`EXTENSION_HANDLER_TIMEOUT_MS = 30_000`), making in-handler polling for external approval infeasible. No escalation chains or multi-signatory approval.
- **Evidence of absence**: `packages/coding-agent/src/extensibility/extensions/runner.ts:72-73` — 30s timeout; `wrapper.ts:188-203` — single `select("Approve"/"Deny")` call.
- **Nature of build**: core change (requires either raising/removing handler timeout for specific events, or a fundamentally different architecture — e.g., parking tool execution pending external decision)

### G-GOV-7: Governance policy propagation to subagents
- **What's missing**: Extensions registered in parent are NOT inherited by subagents. Each subagent creates its own `ExtensionRunner`. Subagents default to `approvalMode: yolo`. A governance extension must be explicitly loaded by each subagent.
- **Evidence of absence**: `docs/approval-mode.md:120-121` — subagent yolo default; subagent `ExtensionRunner` instantiation in `task/executor.ts` — no parent extension inheritance.
- **Nature of build**: core change (extension inheritance/propagation in task spawn path)

## 🗑 Discarded overreach (do NOT re-propose)

- **Token budget via `ctx.getContextUsage().totalTokens`** — REFUTED: `ContextUsage` has no `totalTokens` field; it exposes context-window occupancy (`tokens`, `contextWindow`, `percent`), not cumulative spend. Compaction resets the number. (`packages/coding-agent/src/extensibility/extensions/types.ts:345-351`)
- **TTSR as pre-execution blocker** — REFUTED: TTSR aborts the generation stream + injects a reminder on retry; it does NOT block tool execution. If tool call arguments are already fully produced, the tool dispatches before abort fires. Advisory/corrective only. (`docs/ttsr-injection-lifecycle.md:83-93`)
- **Multi-party approval via 5-minute poll in tool_call handler** — REFUTED: Extension handler timeout is 30 seconds; any handler exceeding this is killed and the tool call blocked with a timeout error — not a policy denial. 300-second poll dies at 30s every time. (`packages/coding-agent/src/extensibility/extensions/runner.ts:72-73, 776-792`)
- **Audit flush in session_shutdown as reliable persistence** — REFUTED: `SESSION_SHUTDOWN_HANDLER_TIMEOUT_MS = 2_000`; shutdown handlers run in parallel via `Promise.all`, each bounded by 2s. Any I/O beyond trivial local writes (network calls, remote audit stores) will be killed. Not a reliable compliance guarantee. (`packages/coding-agent/src/extensibility/extensions/runner.ts:88, 651-663`)
- **Fictional `tools.whitelist` setting / non-sticky setActiveTools** — REFUTED: No `whitelist` or `allowedTools` setting exists in `settings-schema.ts` (zero grep matches). `setActiveTools()` is imperative with no guard/lock — any extension or slash command can re-add tools at any time, making it non-sticky as a whitelist. (`packages/coding-agent/src/extensibility/extensions/types.ts:1216`; `settings-schema.ts` — no whitelist key)

## ⚠ Open design decision

### Governance policy hot-reload

**Current behavior (proven)**: Rules are loaded exactly ONCE during `createAgentSession()` via `bucketRules()` + `setActiveRules()`. Neither `ctx.reload()` nor `/reload-plugins` re-buckets rules:

- `reload()` delegates to `switchSession(sessionFile)` which restores session state (messages, model, thinking level) but never calls `bucketRules`, `loadCapability(ruleCapability.id, ...)`, or `setActiveRules`. (`packages/coding-agent/src/session/agent-session.ts:16273-16276`)
- `setActiveRules` is documented as "Called once per top-level session." (`packages/coding-agent/src/capability/rule.ts:276-278`)
- No file-watcher on rule directories exists.
- `bucketRules` requires a `TtsrManager` instance not exposed to extensions.
- `setActiveRules` is a module-scoped function not exposed to extensions.
- Extension event handlers (tool_call, session_start) intentionally omit `reload()` from their context — only extension COMMANDS can call it.

**The undecided question**: Whether adding mid-session rule re-bucketing is extension work or a core change.

**Three options for a decision-maker**:

1. **Try as extension first**: An extension command (slash command) could attempt to re-read rule files from disk and call internal APIs — but `bucketRules` and `TtsrManager` are not exposed to extensions. Would require exporting these as extension-accessible APIs (a narrow core surface change to enable an extension solution).

2. **Drop hot-reload from scope**: Accept that governance policies are session-scoped. Policy changes take effect on the next session. This matches the current design intent ("Called once per top-level session") and avoids complexity around mid-session rule consistency.

3. **Commit to a core change**: Add a `reloadRules()` function that re-runs the full discovery → normalization → bucketing pipeline, exposes it via extension command context, and optionally adds a file-watcher. Requires careful handling of in-flight TTSR rule state and already-matched rules.
