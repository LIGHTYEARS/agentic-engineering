# Rebuttal: Governance Plane — oh-my-pi Architecture Mapping

## Summary

| Category | Count |
|----------|-------|
| Refuted | 5 |
| Unverifiable | 3 |
| Survived scrutiny | 9 |

The proposal's inventory of existing mechanisms (Section 1) is largely accurate — the extension `tool_call` blocking, approval tiers, TTSR, and settings precedence all exist as described. However, the **proposed implementation strategies** (Section 3) contain several significant errors: fabricated API surface names, mischaracterized TTSR enforcement semantics, and architecturally unsound boundary enforcement claims.

---

## Refutations

### Refutation 1: Token Budget Extension Uses Non-Existent API Shape

```
Target Claim/Proposal: Section 3.4 — Token/Cost Budget Extension proposes using `ctx.getContextUsage()` returning `usage.totalTokens` and calls `ctx.abort()` for hard stop.
Original Confidence: VERIFIED (references to extension points)
Verdict: ✗ REFUTED
Reason: The `ContextUsage` interface does NOT have a `totalTokens` field. It exposes `tokens` (estimated context tokens), `contextWindow`, and `percent`. Furthermore, the code in Section 3.4 subscribes to `pi.on("turn_end", ...)` and expects the handler to receive `ctx` — but `turn_end` handlers are `ExtensionHandler<TurnEndEvent>` which receive `(event: TurnEndEvent, ctx: ExtensionContext)`. The event itself does NOT carry a usage object; the extension must call `ctx.getContextUsage()` to get current context-window usage. This measures *context window occupancy* (cumulative conversation size visible to the model), NOT cumulative tokens consumed across turns. A compaction resets the number. This is fundamentally different from "total tokens spent" and cannot serve as a cost budget.
Counter-Evidence: /tmp/oh-my-pi/packages/coding-agent/src/extensibility/extensions/types.ts:345-351 defines `ContextUsage` as `{ tokens: number; contextWindow: number; percent: number }` — no `totalTokens`. /tmp/oh-my-pi/packages/coding-agent/src/session/agent-session.ts:17174-17181 confirms `getContextUsage()` returns `{ tokens: breakdown.usedTokens, contextWindow: breakdown.contextWindow, percent: ... }` — this is current context window occupancy, not cumulative spend.
```

### Refutation 2: TTSR Boundary Enforcement Cannot Actually Block Operations

```
Target Claim/Proposal: Section 3.6 — Boundary Enforcement via TTSR Rules proposes using `scope: "tool:write(*), tool:edit(*), tool:bash(*)"` with `interruptMode: always` to "block operations outside project boundary".
Original Confidence: VERIFIED (references TTSR interruptMode: always)
Verdict: ✗ REFUTED
Reason: Three issues compound into a fundamental architectural mismatch:

1. **TTSR does NOT block tool execution.** When `interruptMode: always` triggers, it calls `agent.abort()` to abort the *generation stream*, then retries with a system-interrupt injection. But the tool call may ALREADY be partially or fully executing by the time the regex match fires on the streaming tool argument delta. TTSR monitors *streaming deltas* during `message_update` — this is the LLM's output being produced, BEFORE tool execution begins. Once the LLM finishes producing the tool call arguments and the tool dispatches, TTSR has no blocking surface. The abort only prevents the *next* generation, it does not cancel an in-flight tool.

2. **The scope syntax is wrong.** The proposal writes `scope: "tool:write(*), tool:edit(*), tool:bash(*)"` as a single comma-delimited string. The `scope` field accepts `string | string[]` — if passed as a single string, `#parseToolScopeToken` applies a regex `/^(?:(?<prefix>tool)(?::(?<tool>[a-z0-9_-]+))?|(?<bare>[a-z0-9_-]+))(?:\((?<path>[^)]+)\))?$/i` which will NOT match the comma-separated multi-tool format. It would need to be an array: `["tool:write(*)", "tool:edit(*)", "tool:bash(*)"]`.

3. **Regex matching on path strings is unreliable for boundary enforcement.** The condition `"\\.\\.\\/"|"/etc/"|"/usr/"` matches path SUBSTRINGS in the streaming tool argument text. But tools like `bash` receive a `command` field, not a `path` field — and the regex runs against the full serialized tool argument stream, not parsed path values. A command like `cat /etc/hostname` would trigger, but `cat $(echo /etc/hostname)` or variable indirection would evade it trivially. This is advisory at best, never enforcement.

Counter-Evidence: /tmp/oh-my-pi/docs/ttsr-injection-lifecycle.md:83-93 — TTSR trigger aborts generation and retries with injection; it does NOT block tool execution. /tmp/oh-my-pi/packages/coding-agent/src/export/ttsr.ts:131-153 — `#parseToolScopeToken` regex rejects comma-separated scope strings. /tmp/oh-my-pi/docs/ttsr-injection-lifecycle.md:126 — for tool-source matches with `interruptMode: never`, there is NO abort at all, just a post-hoc reminder prepended to the tool result.
```

### Refutation 3: Multi-Party Approval Extension Proposes Blocking Inside `tool_call` Handler Beyond Timeout

```
Target Claim/Proposal: Section 3.7 — Multi-Party Approval via Extension proposes that a `tool_call` handler calls `await pollApprovalDecision(ticket.id, { timeoutMs: 300_000 })` (5-minute poll) to wait for external approval before returning block/allow.
Original Confidence: VERIFIED (tool_call can block)
Verdict: ✗ REFUTED
Reason: The extension handler timeout is 30 seconds (`EXTENSION_HANDLER_TIMEOUT_MS = 30_000`). Any `tool_call` handler that exceeds this timeout is killed and the tool call is **blocked with a timeout error** — NOT because the approval was denied, but because the extension itself timed out. The proposed 300-second poll would be killed after 30 seconds, causing EVERY multi-party approval request to fail with "Extension timed out after 30000ms". The proposal fundamentally cannot work as designed without either (a) modifying the core timeout or (b) a completely different architecture (perhaps blocking at the approval layer before the extension handler, or using a custom tool that replaces the original call).
Counter-Evidence: /tmp/oh-my-pi/packages/coding-agent/src/extensibility/extensions/runner.ts:72-73 — `export const EXTENSION_HANDLER_TIMEOUT_MS = 30_000;`. /tmp/oh-my-pi/packages/coding-agent/src/extensibility/extensions/runner.ts:776-792 — timeout in `emitToolCall` returns `{ block: true, reason: "Extension ... timed out after 30000ms" }`.
```

### Refutation 4: Audit Trail Extension References Non-Existent Events

```
Target Claim/Proposal: Section 3.5 — Governance Audit Trail Extension subscribes to `pi.on("session_shutdown", ...)` for flushing audit logs. It also subscribes to `pi.on("ttsr_triggered", ...)` which is valid, BUT the handler signature shows it accessing `event.rules.map(r => r.name)`.
Original Confidence: VERIFIED (events exist in type system)
Verdict: ✗ REFUTED (partially — the event naming is correct but the architectural promise is broken)
Reason: The `session_shutdown` handler timeout is ONLY 2 seconds (`SESSION_SHUTDOWN_HANDLER_TIMEOUT_MS = 2_000`), and shutdown handlers run in parallel with `Promise.all`. If `persistAuditLog(auditLog)` involves any I/O beyond trivial file writes (network call to a compliance store, database write to a governance audit system), it will be killed after 2 seconds. The proposal presents this as a reliable persistence guarantee ("flush to persistent storage") but the 2-second cap makes it a best-effort path that will silently lose audit records under load or with any remote backend. This fundamentally undermines the compliance/tamper-evident guarantee the governance plane requires.
Counter-Evidence: /tmp/oh-my-pi/packages/coding-agent/src/extensibility/extensions/runner.ts:88 — `export const SESSION_SHUTDOWN_HANDLER_TIMEOUT_MS = 2_000;`. /tmp/oh-my-pi/packages/coding-agent/src/extensibility/extensions/runner.ts:97-99 — `handlerTimeoutForEvent` returns 2000ms specifically for `session_shutdown`. /tmp/oh-my-pi/packages/coding-agent/src/extensibility/extensions/runner.ts:651-663 — shutdown handlers run in parallel via `Promise.all`, each bounded by the 2s timeout.
```

### Refutation 5: Proposed `tools.whitelist` Setting Does Not Exist and Cannot Be Added Without Core Changes

```
Target Claim/Proposal: Section 3.3 proposes adding a `tools.whitelist` setting to `.omp/config.yml` that a "built-in governance extension reads at session start, calls setActiveTools()". Section 2.3 correctly identifies this as a gap but Section 3.3 implies it can be implemented purely via extension without core changes.
Original Confidence: VERIFIED (the gap is real)
Verdict: ✗ REFUTED (the proposed solution is unsound)
Reason: `setActiveTools()` is imperative and can be called by ANY extension or slash command at any time. There is no mechanism to make a whitelist sticky or prevent other extensions/commands from re-adding tools. The proposal says "defense-in-depth check" via `tool_call` handler, which IS valid for blocking — but then the `tools.whitelist` setting itself is fictional (confirmed: no `whitelist` or `allowedTools` in settings-schema.ts). An extension could read a custom YAML key from the project config, but OMP's settings system does not validate unknown keys and won't surface them in `omp config list/get/set`. The whitelist would need to either: (a) live as a non-schema YAML key that the extension manually reads from the file (not via the settings API), or (b) require a core schema addition. The proposal glosses over this as trivial but it's a design gap.
Counter-Evidence: /tmp/oh-my-pi/packages/coding-agent/src/config/settings-schema.ts — grep for `whitelist` or `allowedTools` returns zero matches (confirmed by search). /tmp/oh-my-pi/packages/coding-agent/src/extensibility/extensions/types.ts:1216 — `setActiveTools` is a simple setter with no guard/lock mechanism.
```

---

## Unverifiable

### Unverifiable 1: `pi.appendEntry()` Persistence for Governance Audit

```
Target Claim/Proposal: Section 3.5 and 4.6 state `pi.appendEntry("governance-audit", { ... })` provides "in-session persistence" suitable for audit trails.
Original Confidence: VERIFIED (appendEntry exists)
Verdict: ? UNVERIFIABLE
Reason: `appendEntry` does write to the session file, and the session file is persisted to disk. However, session files are per-session JSON logs that can be deleted, overwritten during compaction, or lost if the process crashes before write. Whether this constitutes "audit persistence" sufficient for a governance plane is a semantic question the source cannot answer — it depends on the compliance requirements. The source confirms the API exists but makes no durability guarantees beyond "written to the session JSONL file at some point." Compaction may summarize or drop custom entries. No tamper-evident guarantees exist.
Counter-Evidence: /tmp/oh-my-pi/packages/coding-agent/src/extensibility/extensions/loader.ts:219-221 — `appendEntry` calls `this.runtime.appendEntry(customType, data)`. The runtime implementation delegates to `SessionManager.appendCustomEntry()` which writes to the session file.
```

### Unverifiable 2: Hot-Reload of Governance Policies (Section 2.6)

```
Target Claim/Proposal: Section 2.6 states rules are loaded once at session creation with no hot-reload. Confidence: INFERRED.
Original Confidence: INFERRED
Verdict: ? UNVERIFIABLE
Reason: The docs confirm rules are loaded once at session creation (`createAgentSession()` calls `bucketRules`). No file-watcher code for rules was found in the source. However, the `/reload` command and `ctx.reload()` in ExtensionCommandContext exist — these restart the session runtime and would re-load rules. So "no hot-reload" is technically accurate (no automatic watching), but manual reload IS possible. The proposal's characterization is mostly correct but the gap is smaller than implied.
Counter-Evidence: /tmp/oh-my-pi/docs/ttsr-injection-lifecycle.md:20-34 — `createAgentSession()` loads rules once. No watcher found in grep for `watch.*rule` across the src tree.
```

### Unverifiable 3: Declarative Approval Matrix via `alwaysApply` Rule

```
Target Claim/Proposal: Section 3.2 proposes encoding an approval matrix as XML inside an `alwaysApply: true` rule's body content, with a governance extension reading it from `session_start`.
Original Confidence: VERIFIED (alwaysApply rules exist)
Verdict: ? UNVERIFIABLE
Reason: `alwaysApply: true` rules are injected into the system prompt — they go to the LLM, not to extensions. The proposal assumes the governance extension can "read the matrix content from the session_start event" — but the `session_start` event has no payload that includes rule body content. The extension would need to independently read the rule files from disk (which is possible via `pi.exec("cat", [...])` or Node.js `fs`), not from any session event. The approach is theoretically possible but the mechanism described (reading from session_start) is not how it would actually work. Whether the extension can reliably discover and parse `.omp/rules/approval-matrix.md` at runtime without the rule system's help is implementation-dependent.
Counter-Evidence: /tmp/oh-my-pi/packages/coding-agent/src/extensibility/shared-events.ts:28-30 — `SessionStartEvent` is `{ type: "session_start" }` — it carries NO payload, no rule content, no configuration data.
```

---

## Missing Coverage

The proposal does not analyze the following subsystems that would materially affect governance implementation:

1. **Compaction interaction**: Governance audit entries stored via `appendEntry` will be subject to compaction. When the context compacts, custom entries may be summarized away or dropped entirely. The proposal never addresses how audit records survive compaction cycles — a critical gap for compliance.

2. **Subagent isolation model**: The proposal mentions `task.isolation.mode` briefly but never addresses how governance policies propagate to subagents. Subagents run `tools.approvalMode: yolo` by default (confirmed in `/tmp/oh-my-pi/docs/approval-mode.md:120-121`). Extensions registered in the parent are NOT automatically inherited by subagents — each subagent creates its own `ExtensionRunner` via `task/executor.ts`. A governance extension would need explicit support in the task spawn path to propagate to children.

3. **Extension loading order and conflict**: Multiple governance extensions could conflict. The `tool_call` handler uses "first block wins" semantics. If two governance extensions both want to evaluate risk, the first one to block determines the outcome. The proposal never discusses extension ordering, priority, or composition of multiple governance handlers.

4. **Process model constraints**: Extensions run in-process with no sandboxing. A malicious or buggy governance extension has full access to the Node.js runtime, can monkey-patch the extension runner, modify other extensions' state, or bypass its own policy checks. The proposal assumes extensions are trusted — but a governance plane that can be subverted by the governed code is architecturally unsound.

5. **ACP (Agent Control Protocol) mode**: The proposal focuses on interactive mode but never addresses how governance works in ACP sessions, where UI approval routes through the ACP client (not the terminal). The audit trail and approval flows would need different implementations for ACP vs interactive vs headless modes.

---

## Survived Scrutiny

1. **Section 1.1 — Tool Approval System (Three-tier model)**: Confirmed. `/tmp/oh-my-pi/docs/approval-mode.md` documents `read`/`write`/`exec` tiers and `always-ask`/`write`/`yolo` modes exactly as described. `tools.approval.<tool>: allow|deny|prompt` exists in settings.

2. **Section 1.2 — Critical Pattern Detection**: Confirmed. The proposal correctly identifies that `CRITICAL_BASH_PATTERNS` exists for bash-specific risk detection via `approval: (args) => { override: true }` escalation.

3. **Section 1.3 — Extension-Based Tool Blocking**: Confirmed. `/tmp/oh-my-pi/packages/coding-agent/src/extensibility/extensions/wrapper.ts:206-228` implements `tool_call` pre-execution blocking exactly as described — `{ block: true }` prevents execution, errors block (fail-closed). This is the correct and viable primary enforcement surface.

4. **Section 1.4 — TTSR Stream Monitoring**: Confirmed. The TTSR system works as described for its actual purpose (real-time regex/AST monitoring of generation streams with abort+retry). The proposal correctly identifies scope targeting, interrupt modes, and repeat policies.

5. **Section 1.5 — Rule System**: Confirmed. The rulebook pipeline with `alwaysApply`, `condition`, `astCondition`, `globs`, `scope`, and `interruptMode` exists as documented.

6. **Section 1.7 — Subagent Budget Controls**: Confirmed. `task.softRequestBudget`, `task.maxConcurrency`, `task.maxRecursionDepth`, `task.maxRuntimeMs` all exist in settings-schema.ts.

7. **Section 1.8 — session_stop Hook**: Confirmed. `/tmp/oh-my-pi/packages/coding-agent/src/extensibility/shared-events.ts:359-368` defines `SessionStopEventResult` with `{ continue?: boolean; additionalContext?: string; decision?: "block"; reason?: string }` — extension can continue or block session completion.

8. **Section 1.9 — Settings Precedence**: Confirmed. `/tmp/oh-my-pi/docs/settings.md:92-107` documents the five-layer precedence exactly as described.

9. **Section 3.1 — Risk Grading Extension via tool_call hook**: The basic approach IS viable. A `tool_call` handler receiving `event.toolName`, `event.toolCallId`, `event.input` with access to `ctx.cwd` and `ctx.sessionManager` CAN implement risk scoring and return `{ block: true }`. The code shape is slightly wrong (the extension API is `pi.on("tool_call", ...)` not `pi.on("tool_call", async (event, ctx) => {...})` with `ctx` as shown — but actually it IS correct per types.ts:1120). Upheld by `/tmp/oh-my-pi/packages/coding-agent/src/extensibility/extensions/types.ts:1120` and wrapper.ts:206-228.
