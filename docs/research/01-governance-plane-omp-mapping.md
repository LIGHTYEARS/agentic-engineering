# Governance Plane — oh-my-pi Architecture Mapping

## 1. Inventory of Governance Mechanisms Already in oh-my-pi

### 1.1 Tool Approval System (Tiered Permission Model)

```
Claim: OMP implements a three-tier permission model (read/write/exec) with configurable approval modes.
Rationale: Every tool declares an `approval` tier; the user selects an approval mode that auto-approves up to a given tier and prompts above it.
Evidence: packages/coding-agent/src/tools/approval.ts:28-38 defines TIER_RANK and APPROVAL_MODE_MAX_TIER; docs/approval-mode.md describes the three modes (always-ask, write, yolo).
Confidence: VERIFIED
```

The system provides:
- **Tool tier declarations**: `read`, `write`, `exec` — each tool self-declares its capability level.
- **Approval modes**: `always-ask` (auto-approves only `read`), `write` (auto-approves `read`+`write`), `yolo` (auto-approves everything).
- **Per-tool user overrides**: `tools.approval.<toolName>: allow | deny | prompt` — fine-grained per-tool policy regardless of mode.
- **Safety overrides**: Tools can dynamically escalate with `{ tier: "exec", override: true, reason: "..." }` to force a prompt even in permissive modes (non-yolo).

Key files:
- `packages/coding-agent/src/tools/approval.ts` — resolution logic
- `packages/coding-agent/src/extensibility/extensions/wrapper.ts` — enforcement in tool execution pipeline
- `packages/coding-agent/src/config/settings-schema.ts:3417-3460` — settings definitions

### 1.2 Critical Pattern Detection (Risk Classification)

```
Claim: OMP implements a hardcoded risk-pattern detector for bash commands that escalates dangerous patterns to require approval.
Rationale: The CRITICAL_BASH_PATTERNS array matches patterns like rm -rf /, fork bombs, dd to devices, remote-fetch-execute, etc. and returns { override: true } to force a prompt.
Evidence: packages/coding-agent/src/tools/bash.ts:82-128 defines CRITICAL_BASH_PATTERNS; line 380-383 returns the override decision.
Confidence: VERIFIED
```

Categories detected:
- Recursive destruction (`rm -rf /`, `sudo rm`)
- Fork bombs
- Disk/filesystem destruction (`mkfs`, `dd`, `shred`)
- System-config destruction (`> /etc/passwd`)
- Remote-fetch-then-execute (`curl | bash`)
- Process/host control (`shutdown`, `kill -9 1`)
- Network-shell exfiltration (`nc -e`)

### 1.3 Extension-Based Tool Blocking (Hook/Extension `tool_call` Interception)

```
Claim: Extensions and hooks can block any tool call pre-execution via the tool_call event, implementing arbitrary policy logic.
Rationale: The ExtensionToolWrapper emits tool_call before execution; if any handler returns { block: true }, execution is prevented. Errors in handlers also block (fail-closed).
Evidence: packages/coding-agent/src/extensibility/extensions/wrapper.ts:206-228; docs/hooks.md lines 113-134 (flow diagram).
Confidence: VERIFIED
```

This is the **primary extension point for governance policies**:
- Runs before tool execution
- Any handler can return `{ block: true, reason: "..." }`
- Fail-closed: handler errors block execution
- Has access to `toolName`, `toolCallId`, and `input` parameters

### 1.4 TTSR (Time Traveling Stream Rules) — Real-Time Stream Compliance

```
Claim: TTSR provides real-time regex/AST-grep monitoring of agent output streams, with the ability to interrupt mid-generation and inject corrective instructions.
Rationale: Rules with condition/astCondition fields are registered with TtsrManager and matched against streaming text/tool deltas. Matching rules trigger abort + retry with injected system-interrupt messages.
Evidence: docs/ttsr-injection-lifecycle.md (full document); packages/coding-agent/src/export/ttsr.ts; packages/coding-agent/src/sdk.ts (bucketRules call).
Confidence: VERIFIED
```

TTSR features relevant to governance:
- **Scope control**: Rules can target `text`, `thinking`, `tool` (or specific tools like `tool:edit(*.ts)`)
- **Interrupt modes**: `always` (abort + retry), `never` (inject reminder post-hoc), `prose-only`, `tool-only`
- **Repeat policy**: `once` or `after-gap` (measured in turns)
- **Glob-gating**: Rules can be restricted to specific file paths
- **AST conditions**: Structural pattern matching on edit/write tool streams

### 1.5 Rule System (Declarative Policy Framework)

```
Claim: OMP discovers rules from multiple provider formats, normalizes them into a canonical shape, and routes them to TTSR (enforcement), always-apply (mandatory context), or rulebook (advisory) buckets.
Rationale: The rulebook-matching-pipeline.md documents the full discovery→normalization→bucketing pipeline with seven providers at defined priority levels.
Evidence: docs/rulebook-matching-pipeline.md sections 1-5; packages/coding-agent/src/capability/rule.ts; packages/coding-agent/src/capability/rule-buckets.ts.
Confidence: VERIFIED
```

Rule properties relevant to governance:
- `alwaysApply: true` — injected into system prompt unconditionally
- `condition` — regex triggers for TTSR enforcement
- `astCondition` — structural pattern triggers
- `globs` — file-path scoping
- `scope` — stream surface targeting
- `interruptMode` — enforcement severity

### 1.6 MCP Server Denylist/Allowlist

```
Claim: OMP maintains user-level denylist and allowlist for MCP servers, providing a tool-source whitelist mechanism.
Rationale: The MCP config schema defines disabledServers (denylist, highest precedence) and enabledServers (allowlist, overrides source enabled:false but not denylist).
Evidence: packages/coding-agent/src/mcp/config-writer.ts:199-228 (read/write disabled servers); packages/coding-agent/src/config/mcp-schema.json:24-35 (schema definitions); packages/coding-agent/src/mcp/config.ts:108-122 (filtering logic).
Confidence: VERIFIED
```

### 1.7 Subagent Budget Controls

```
Claim: OMP implements soft and hard budget controls for subagents including request budgets, concurrency limits, recursion depth, and wall-clock timeouts.
Rationale: The task.* settings provide multi-dimensional resource controls for delegated work.
Evidence: packages/coding-agent/src/config/settings-schema.ts:4160-4270 (task.maxConcurrency, task.maxRecursionDepth, task.maxRuntimeMs, task.softRequestBudget, task.softRequestBudgetNotice).
Confidence: VERIFIED
```

Budget mechanisms:
| Setting | Default | Purpose |
|---------|---------|---------|
| `task.maxConcurrency` | 32 | Max parallel subagents |
| `task.maxRecursionDepth` | 2 | Max nesting depth |
| `task.maxRuntimeMs` | 0 (unlimited) | Wall-clock timeout |
| `task.softRequestBudget` | 200 | Soft request limit (steers wrap-up) |
| Force stop | 1.5× budget | Hard stop |

### 1.8 Session-Stop Hook (Post-Run Gate)

```
Claim: The session_stop extension event allows extensions to block session completion and inject continuation context, providing a post-execution governance checkpoint.
Rationale: The event supports returning { decision: "block", reason: "..." } or { continue: true, additionalContext: "..." } to prevent premature delivery.
Evidence: packages/coding-agent/src/extensibility/shared-events.ts:358-368 (SessionStopEventResult type); docs/extensions.md line 247-248 (session_stop description).
Confidence: VERIFIED
```

### 1.9 Settings Precedence Chain (Configuration Governance)

```
Claim: OMP implements a five-layer settings precedence chain allowing governance policies at multiple organizational levels.
Rationale: Settings merge from built-in defaults → global config → project config → CLI overlays → runtime overrides, with objects deep-merged and arrays replaced.
Evidence: docs/settings.md:92-107 (precedence documentation); packages/coding-agent/src/config/settings-schema.ts.
Confidence: VERIFIED
```

Governance-relevant layers:
1. **Built-in defaults** — vendor-supplied baseline
2. **Global config** (`~/.omp/agent/config.yml`) — organization/user policy
3. **Project config** (`<cwd>/.omp/config.yml`) — repository-specific policy
4. **CLI overlays** (`--config <file>`) — deployment/CI-specific
5. **Runtime overrides** (`--approval-mode`, etc.) — session-specific

### 1.10 Provider/Discovery Source Disabling

```
Claim: The disabledProviders setting can gate both model providers AND discovery sources, allowing policy control over which AI backends and configuration sources are active.
Rationale: disabledProviders operates on a shared namespace gating both model backends (anthropic, openai...) and discovery sources (claude, codex, cursor...) before credential checks.
Evidence: docs/settings.md:256-278 (Provider and source disabling); packages/coding-agent/src/config/settings-schema.ts.
Confidence: VERIFIED
```

### 1.11 Tool Enable/Disable Controls

```
Claim: Individual built-in tools can be toggled via settings (bash.enabled, eval.py, browser.enabled, etc.) and extensions can dynamically activate/deactivate tools via setActiveTools.
Rationale: Settings like bash.enabled gate tool registration; extensions use pi.setActiveTools() for dynamic toolset management.
Evidence: docs/settings.md:452-453 (tool toggles); packages/coding-agent/src/extensibility/extensions/types.ts:1216 (setActiveTools API); the defaultInactive field on tool definitions.
Confidence: VERIFIED
```

---

## 2. Gap Analysis — What the Governance Plane Requires but OMP Doesn't Provide

### 2.1 Formal Risk Grading Matrix

```
Claim: OMP has a three-tier tool classification (read/write/exec) but lacks a formal, multi-dimensional risk grading matrix that considers context, scope, reversibility, and blast radius.
Rationale: The current tier system is binary (auto-approve or prompt). It does not differentiate between "write one file in a sandbox" vs "write /etc/hosts" or between "exec ls" vs "exec docker push to production". Risk is either hardcoded in CRITICAL_BASH_PATTERNS or absent.
Evidence: packages/coding-agent/src/tools/approval.ts — only tier + override; no risk score, no contextual escalation beyond the bash-specific CRITICAL_BASH_PATTERNS.
Confidence: VERIFIED
```

**Gap**: Need a composable risk scoring function: `risk(tool, args, context) → RiskLevel` that considers:
- Target scope (sandbox vs production)
- Reversibility (read-only vs destructive)
- Blast radius (single file vs system-wide)
- Historical context (first time vs repeated operation)

### 2.2 Approval Matrices (Role × Operation → Decision)

```
Claim: OMP provides user-level tool policies but lacks role-based approval matrices mapping (who × what × where → approve/deny/escalate).
Rationale: The current system has one user with one policy set. There is no concept of "this agent runs as role=reviewer and cannot edit files" or "agent-A can only read, agent-B can also write, agent-C needs human approval for exec".
Evidence: All approval resolution in approval.ts operates on a single userConfig record; no role/identity parameter exists.
Confidence: VERIFIED
```

**Gap**: Need:
- Agent identity/role declarations
- Operation × role → policy mapping
- Escalation paths (auto → supervisor → human)
- Delegation chains (parent grants subset of own permissions to child)

### 2.3 Explicit Tool Whitelists (Positive Enumeration)

```
Claim: OMP uses deny-by-default only for unknown MCP tools (treated as exec tier). There is no positive-enumeration whitelist declaring "only these tools are available in this context".
Rationale: The MCP denylist is a negative filter; tool toggles (bash.enabled) are individual. There is no tools.allowedTools = ["read", "grep", "write"] setting that would restrict the active toolset to an explicit positive list.
Evidence: settings-schema.ts has no allowedTools/whitelist setting; the task tool inherits the full toolset minus opt-outs. setActiveTools exists but is imperative, not declarative policy.
Confidence: VERIFIED
```

**Gap**: Need declarative tool whitelists per context:
- Per-agent toolsets defined at spawn
- Project-level tool restrictions
- Phase-dependent tool availability (planning phase: read-only; implementation phase: full)

### 2.4 Token/Cost Budget Controls

```
Claim: OMP has per-subagent request budgets but lacks token-level or monetary cost budgets for sessions, tasks, or organizations.
Rationale: task.softRequestBudget counts assistant requests (turns), not tokens consumed or dollars spent. There is no setting for "max $5 per task" or "max 500K tokens per session".
Evidence: settings-schema.ts:4242-4268 defines softRequestBudget in request count only; no token/cost budget settings exist.
Confidence: VERIFIED
```

**Gap**: Need:
- Per-session token budget (input + output)
- Per-task monetary cost ceiling
- Organizational spending limits with alerting
- Budget inheritance/splitting across subagents

### 2.5 Audit Trail / Compliance Logging

```
Claim: OMP persists session entries and TTSR injection records but lacks a dedicated audit log for governance decisions (approvals, denials, escalations, policy violations).
Rationale: The tool_approval_requested/resolved events are observability-only (emitted to extension handlers). There is no structured audit log that records all governance decisions with timestamps, rationale, and outcome.
Evidence: wrapper.ts:152-172 emits tool_approval_requested/resolved events; these are fire-and-forget to handlers, not persisted to a governance audit store.
Confidence: VERIFIED
```

**Gap**: Need:
- Append-only audit log for all governance decisions
- Structured records: timestamp, agent, tool, args, policy, decision, approver, rationale
- Queryable history for compliance review
- Tamper-evident storage

### 2.6 Dynamic Policy Updates (Hot-Reload)

```
Claim: OMP loads rules at session creation and does not support hot-reloading governance policies mid-session without restart.
Rationale: bucketRules runs once in createAgentSession; TTSR rules are registered at startup. There is no file-watch or signal-based reload mechanism for rules.
Evidence: docs/ttsr-injection-lifecycle.md section 1 — rules loaded at session creation; no watcher mentioned. ttsr.disabledRules and builtinRules are read once.
Confidence: INFERRED (from documentation; no watcher code found)
```

**Gap**: Need:
- File-watch on rule directories for policy updates
- Signal/command to reload governance configuration
- Version-aware policy (apply-after vs apply-immediately)

### 2.7 Multi-Party Approval / Escalation

```
Claim: OMP supports only single-user approval (the operator confirms or denies). There is no multi-party approval workflow for high-risk operations.
Rationale: The approval flow in wrapper.ts goes directly to uiContext.select() for the one operator. No concept of "requires 2 approvals" or "escalate to team lead".
Evidence: wrapper.ts:188-203 — single select("Approve"/"Deny") call; no escalation routing.
Confidence: VERIFIED
```

**Gap**: Need:
- Multi-signatory approval for high-risk operations
- Escalation chains with timeout-based routing
- Integration with external approval systems (Slack, PagerDuty, etc.)

### 2.8 Scope/Boundary Enforcement

```
Claim: OMP allows project-level config but does not enforce filesystem or network boundaries — agents can access anything the OS user can.
Rationale: The bash tool runs with full user permissions. File read/write tools are not sandboxed to the project directory. task.isolation provides copy-on-write isolation for subagents but not containment.
Evidence: task.isolation.mode in settings-schema.ts:4039-4060 provides isolation modes (overlayfs, rcopy, worktree) for subagent file changes, but these are merge strategies, not access control. Bash has no chroot/namespace enforcement.
Confidence: VERIFIED
```

**Gap**: Need:
- Filesystem boundary enforcement (restrict to project root + declared exceptions)
- Network access control (allow/deny specific hosts/ports)
- Environment variable access control
- Secret injection without ambient access

---

## 3. Proposal — Strategies to Implement Missing Governance Features

### 3.1 Risk Grading Extension (via `tool_call` hook)

**Strategy**: Implement a governance extension that intercepts all `tool_call` events and computes a risk score using a composable evaluator chain.

```typescript
// .omp/extensions/governance/risk-grader.ts
export default function (pi: ExtensionAPI) {
  pi.on("tool_call", async (event, ctx) => {
    const risk = computeRisk(event.toolName, event.input, {
      cwd: ctx.cwd,
      session: ctx.sessionManager,
    });
    
    if (risk.level === "critical") {
      return { block: true, reason: `Risk level CRITICAL: ${risk.explanation}` };
    }
    if (risk.level === "high" && !ctx.hasUI) {
      return { block: true, reason: `High-risk operation requires interactive approval` };
    }
    // Log all risk assessments to audit trail
    await auditLog.record({ event, risk, timestamp: Date.now() });
  });
}
```

**Extension point**: `pi.on("tool_call", ...)` — the pre-execution interception surface.

**Risk evaluator dimensions**:
- Path sensitivity (production paths, system paths, secret paths)
- Operation destructiveness (delete > write > read)
- Data sensitivity (PII patterns, credential patterns)
- Blast radius (scope of potential damage)

### 3.2 Declarative Approval Matrix (via TTSR Rules + Extension)

**Strategy**: Define approval matrices as YAML rules discovered through the existing rule pipeline, enforced by a governance extension that reads the matrix at session start and applies it during `tool_call`.

```yaml
# .omp/rules/approval-matrix.md
---
alwaysApply: true
description: "Role-based approval matrix for production environments"
---
<governance-matrix>
  <role name="reviewer" tools="read,grep,web_search" />
  <role name="implementer" tools="read,grep,write,edit,bash" bash-deny="docker push|kubectl apply" />
  <role name="deployer" tools="*" requires-approval="bash:kubectl*,bash:docker push*" />
</governance-matrix>
```

The governance extension reads the matrix content from the `session_start` event and enforces it on every `tool_call`. Agent role is determined by:
- Explicit agent definition (in task spawn parameters)
- Inherited from parent context
- Declared in project config

### 3.3 Tool Whitelist via `setActiveTools` + TTSR Enforcement

**Strategy**: Combine two mechanisms:
1. **Declarative whitelist** in project config (new setting `tools.whitelist`)
2. **TTSR rule** that triggers if the agent attempts to use a non-whitelisted tool path

```yaml
# .omp/config.yml
tools:
  approvalMode: write
  whitelist:
    - read
    - grep
    - write
    - edit
    - bash
  # All other tools require explicit approval or are blocked
```

Implementation: A built-in governance extension reads `tools.whitelist` at session start, calls `setActiveTools()` to restrict the active set, and registers a `tool_call` handler as a defense-in-depth check.

### 3.4 Token/Cost Budget Extension

**Strategy**: Build a budget-tracking extension using `tool_result` events and `getContextUsage()`.

```typescript
export default function (pi: ExtensionAPI) {
  let totalTokens = 0;
  const MAX_TOKENS = Number(process.env.GOVERNANCE_TOKEN_BUDGET ?? Infinity);
  
  pi.on("turn_end", async (_event, ctx) => {
    const usage = ctx.getContextUsage();
    totalTokens = usage.totalTokens;
    
    if (totalTokens > MAX_TOKENS * 0.9) {
      pi.sendMessage({
        role: "user",
        content: [{ type: "text", text: "⚠️ Approaching token budget limit. Wrap up." }],
      }, { deliverAs: "steer" });
    }
    if (totalTokens > MAX_TOKENS) {
      // Force shutdown
      ctx.abort();
    }
  });
}
```

**Extension points used**:
- `pi.on("turn_end", ...)` — post-turn token accounting
- `ctx.getContextUsage()` — current token consumption
- `pi.sendMessage(..., { deliverAs: "steer" })` — mid-run budget warnings
- `ctx.abort()` — hard stop on budget exhaustion

### 3.5 Governance Audit Trail Extension

**Strategy**: Extension that subscribes to all governance-relevant events and appends structured records to a local audit log.

```typescript
export default function (pi: ExtensionAPI) {
  const auditLog: AuditRecord[] = [];
  
  pi.on("tool_call", async (event, ctx) => {
    auditLog.push({
      type: "tool_invocation",
      timestamp: Date.now(),
      tool: event.toolName,
      input: sanitize(event.input),
      session: ctx.sessionManager.getSessionId(),
    });
  });
  
  pi.on("tool_approval_requested", async (event) => {
    auditLog.push({ type: "approval_requested", ...event, timestamp: Date.now() });
  });
  
  pi.on("tool_approval_resolved", async (event) => {
    auditLog.push({ type: "approval_resolved", ...event, timestamp: Date.now() });
  });
  
  pi.on("ttsr_triggered", async (event) => {
    auditLog.push({
      type: "policy_violation",
      rules: event.rules.map(r => r.name),
      timestamp: Date.now(),
    });
  });
  
  pi.on("session_shutdown", async () => {
    await persistAuditLog(auditLog);
  });
}
```

**Extension points used**:
- `tool_call`, `tool_result` — tool activity
- `tool_approval_requested`, `tool_approval_resolved` — approval decisions
- `ttsr_triggered` — policy violations
- `session_shutdown` — flush to persistent storage
- `pi.appendEntry(customType, data)` — in-session persistence

### 3.6 Boundary Enforcement via TTSR Rules

**Strategy**: Use TTSR condition rules with scope targeting to detect filesystem/network boundary violations in real-time.

```yaml
# .omp/rules/boundary-enforcement.md
---
description: "Block operations outside project boundary"
condition:
  - "\\.\\./"
  - "/etc/"
  - "/usr/"
  - "/var/"
  - "~/"
scope: "tool:write(*), tool:edit(*), tool:bash(*)"
interruptMode: always
---
<system-interrupt>
STOP: You are attempting to access a path outside the project boundary.
Operations are restricted to the current working directory tree.
If you need to access external resources, explain why and request explicit approval.
</system-interrupt>
```

For network boundaries:
```yaml
# .omp/rules/network-boundary.md
---
description: "Detect network access attempts in bash"
condition:
  - "\\b(curl|wget|fetch|nc|ssh|scp|rsync)\\b"
scope: "tool:bash(*)"
interruptMode: never
---
<system-reminder>
Network access detected. Ensure this operation is within the approved network policy.
Approved hosts: github.com, registry.npmjs.org
Blocked: all others without explicit approval.
</system-reminder>
```

### 3.7 Multi-Party Approval via Extension + External Webhook

**Strategy**: Replace the single-user approval UI flow with an extension that routes high-risk approvals to an external system.

```typescript
export default function (pi: ExtensionAPI) {
  pi.on("tool_call", async (event, ctx) => {
    const risk = assessRisk(event);
    if (risk.requiresMultiParty) {
      // Block execution and initiate external approval
      const ticket = await createApprovalRequest({
        tool: event.toolName,
        args: event.input,
        risk: risk.level,
        requiredApprovers: risk.approverCount,
      });
      
      // Wait for external approval (with timeout)
      const decision = await pollApprovalDecision(ticket.id, { timeoutMs: 300_000 });
      if (!decision.approved) {
        return { block: true, reason: `Multi-party approval denied: ${decision.reason}` };
      }
    }
  });
}
```

---

## 4. Key Extension Points and Their APIs

### 4.1 `tool_call` Event (Primary Governance Gate)

```typescript
pi.on("tool_call", async (event: ToolCallEvent, ctx: ExtensionContext) => {
  // event.toolName: string — name of tool being invoked
  // event.toolCallId: string — unique invocation ID
  // event.input: Record<string, unknown> — tool arguments
  // return { block: true, reason?: string } to prevent execution
  // return undefined to allow
});
```

**Location**: `packages/coding-agent/src/extensibility/extensions/types.ts`
**Enforcement**: `packages/coding-agent/src/extensibility/extensions/wrapper.ts:206-228`
**Behavior**: First `block: true` wins; handler errors also block (fail-closed)

### 4.2 `tool_result` Event (Post-Execution Audit/Redaction)

```typescript
pi.on("tool_result", async (event: ToolResultEvent) => {
  // event.content, event.details, event.isError
  // Can modify output (redact secrets, add warnings)
  // return { content?, details?, isError? }
});
```

**Behavior**: Middleware-style; each handler sees prior modifications. Last result wins.

### 4.3 `session_stop` Event (Completion Gate)

```typescript
pi.on("session_stop", async (event, ctx) => {
  // Return { decision: "block", reason: "..." } to prevent delivery
  // Return { continue: true, additionalContext: "..." } to inject more work
  // Capped at 8 consecutive continuations
});
```

**Location**: `packages/coding-agent/src/extensibility/shared-events.ts:358-368`

### 4.4 TTSR Rule Registration (Declarative Stream Monitoring)

```yaml
# Rule file: .omp/rules/<name>.md
---
condition:
  - "regex_pattern"
astCondition:
  - "ast_grep_pattern"
scope: "text, tool:edit(*.ts)"
globs:
  - "src/**/*.ts"
interruptMode: always  # always | never | prose-only | tool-only
---
Content injected as system-interrupt on match.
```

**Location**: `packages/coding-agent/src/export/ttsr.ts` (TtsrManager)
**Lifecycle**: Registered at session creation via `bucketRules()` in `sdk.ts`

### 4.5 Settings System (Configuration-as-Policy)

```yaml
# .omp/config.yml (project-level)
tools:
  approvalMode: write
  approval:
    bash: prompt
    mcp__dangerous_tool: deny
  maxTimeout: 300
task:
  maxConcurrency: 4
  maxRecursionDepth: 1
  softRequestBudget: 100
  maxRuntimeMs: 600000
```

**Location**: `packages/coding-agent/src/config/settings-schema.ts`
**Precedence**: defaults → global → project → CLI overlay → runtime override

### 4.6 Extension Runtime Actions (Dynamic Governance)

```typescript
// Dynamically restrict tool availability
await pi.setActiveTools(["read", "grep", "write"]);

// Inject governance messages mid-run
pi.sendMessage(msg, { deliverAs: "steer" });

// Persistent state for governance tracking
pi.appendEntry("governance-audit", { ... });

// Context manipulation
pi.on("context", async (event) => {
  // Filter/inject messages before LLM call
  return { messages: filteredMessages };
});
```

### 4.7 `before_provider_request` Event (Request-Level Interception)

```typescript
pi.on("before_provider_request", async (event) => {
  // Can replace the provider request payload
  // Useful for enforcing token limits, injecting governance context
});
```

**Location**: `packages/coding-agent/src/extensibility/extensions/types.ts` (event surface)

---

## 5. Architecture Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                    GOVERNANCE PLANE (Proposed)                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ Risk Grading │  │ Approval     │  │ Tool Whitelists      │  │
│  │ Extension    │  │ Matrix Ext.  │  │ (setActiveTools)     │  │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────┘  │
│         │                  │                      │              │
│  ┌──────▼──────────────────▼──────────────────────▼───────────┐ │
│  │              tool_call Event (Pre-Execution Gate)           │ │
│  │              ExtensionToolWrapper.execute()                 │ │
│  └────────────────────────────┬───────────────────────────────┘ │
│                               │                                  │
│  ┌────────────────────────────▼───────────────────────────────┐ │
│  │         Built-in Approval System (tier + mode + policy)    │ │
│  │         packages/coding-agent/src/tools/approval.ts        │ │
│  └────────────────────────────┬───────────────────────────────┘ │
│                               │                                  │
│  ┌────────────────────────────▼───────────────────────────────┐ │
│  │  TTSR Stream Monitor (Real-time compliance)                │ │
│  │  .omp/rules/*.md with condition/astCondition               │ │
│  └────────────────────────────┬───────────────────────────────┘ │
│                               │                                  │
│  ┌────────────────────────────▼───────────────────────────────┐ │
│  │  Budget Controls (task.softRequestBudget, token tracking)  │ │
│  └────────────────────────────┬───────────────────────────────┘ │
│                               │                                  │
│  ┌────────────────────────────▼───────────────────────────────┐ │
│  │  Audit Trail Extension (all events → structured log)       │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                   │
│  Settings Layers: defaults → global → project → overlay → runtime│
└─────────────────────────────────────────────────────────────────┘
```

**Key insight**: OMP's extension system already provides the **enforcement hooks** needed for a Governance Plane. The gaps are primarily in:
1. **Policy expression** (no matrix DSL, no risk scoring framework)
2. **Budget granularity** (request-count only, no token/cost)
3. **Identity/role system** (single-user, no delegation)
4. **Audit persistence** (events available but not structured/stored)
5. **Boundary enforcement** (no sandboxing, only advisory TTSR rules)

All five gaps can be addressed through the extension API without modifying OMP core, making the Governance Plane implementable as a **plugin package** that composes:
- One or more extensions (risk grading, audit, budget, matrix enforcement)
- A set of TTSR rules (boundary detection, pattern compliance)
- Project-level settings (tool whitelists, approval modes, budget limits)
