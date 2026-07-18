# Resolution: Governance Plane — Unverifiable Findings

## Summary

| Verdict | Count |
|---------|-------|
| CONFIRMED | 2 |
| REFUTED | 0 |
| IRREDUCIBLY UNVERIFIABLE | 1 |

Two of three proposals are feasible with caveats. One (hot-reload of governance policies) remains irreducibly unverifiable because it depends on whether a future `/reload` command refactoring would re-bucket rules — a design intent the source cannot decide.

---

## Resolutions

### Finding 1: Session-File Audit Persistence via `appendEntry`

```
Finding: Proposal Sections 3.5 and 4.6 claim `pi.appendEntry("governance-audit", { ... })` provides
"in-session persistence" suitable for audit trails. Rebuttal cited loader.ts:219-221 but could not
confirm durability (survives compaction? crash? overwrite?).

Prior Verdict: ? UNVERIFIABLE

New Verdict: ✓ CONFIRMED (proposal feasible/supported)

Deeper Evidence:

1. WRITE PATH — Synchronous durability guarantee:
   /tmp/oh-my-pi/packages/coding-agent/src/session/session-manager.ts:837-841
     #recordEntry(entry: SessionEntry): void {
       this.#entries.push(entry);
       this.#index.insert(entry);
       this.#appendToSessionFile(entry);       // <-- immediate disk write
       this.#notifyEntryAppended(entry);
     }

   /tmp/oh-my-pi/packages/coding-agent/src/session/session-manager.ts:704-712
     // Hot path: append synchronously so the entry is durable the instant this
     // returns (file/memory writers perform the write in-body). Never routed
     // through the async disk chain — durability must hold without a flush().
     void this.#appendWriter()
       .append(this.#lineFor(entry))
       .catch(err => this.#noteDiskFailure(err));

   The comment explicitly guarantees "durable the instant this returns" — synchronous write to the
   session JSONL file via the kernel page cache.

2. COMPACTION SURVIVAL — All entries survive in the session file:
   /tmp/oh-my-pi/packages/coding-agent/src/session/session-manager.ts:554-558
     #fileBody(): string {
       let body = this.#titleSlotLine();
       body += this.#lineFor(this.#header);
       for (const entry of this.#entries) body += this.#lineFor(entry);
       return body;
     }

   The `#entries` array is NEVER pruned. Compaction adds a new CompactionEntry with
   `firstKeptEntryId` designating which entries to send to the LLM, but ALL entries —
   including custom entries before the compaction boundary — remain in the session file.
   
   Confirmed by the compaction logic:
   /tmp/oh-my-pi/packages/agent/src/compaction/compaction.ts:124-141
     getMessageFromEntry for type "custom" returns `undefined` — custom entries are
     invisible to the LLM context but remain persisted in the entry array.

   /tmp/oh-my-pi/packages/agent/src/compaction/compaction.ts:499-501
     case "custom":
     case "custom_message":
     case "label":
     // These fall through the switch with no action — they are valid cut points
     // only for custom_message, NOT for plain "custom" entries.

   Plain "custom" entries (what appendEntry produces) are not cut points, not
   summarized, not deleted — they simply persist inertly in the session file.

3. CRASH DURABILITY:
   /tmp/oh-my-pi/packages/coding-agent/src/session/session-manager.ts:594
     #rewriteSynchronously():
       `writeTextSync` returns with bytes in kernel page cache — software-crash durable.
   
   The hot-path append writes synchronously. If the process crashes mid-write of a
   single JSONL line, the worst case is a truncated last entry — all prior entries
   survive because JSONL is append-only (each line is self-contained JSON).

4. NO DELETION/OVERWRITE:
   /tmp/oh-my-pi/packages/coding-agent/src/session/session-manager.ts:569-586
     #elideSupersededCompactionsOnBranch only touches CompactionEntry summaries
     (replaces their text with "[Superseded...]"), never custom entries.

Reasoning:
The session file is an append-only JSONL log. `appendCustomEntry` writes synchronously
to disk via the hot path. The `#entries` array is never pruned — not by compaction, not
by rewrite, not by any maintenance pass. Custom entries persist for the lifetime of the
session file. Compaction merely marks a `firstKeptEntryId` boundary that determines what
gets sent to the LLM; entries below that boundary remain on disk forever.

This makes session-file-backed audit persistence FEASIBLE for:
- Append-only structured logging of governance decisions
- Queryable history (parse the JSONL file)
- Crash durability (synchronous writes)

Limitations that survive from the rebuttal:
- No tamper-evident guarantees (file is writable by the same user)
- Session files can be manually deleted by the operator
- No cross-session aggregation (each session is a separate file)
- The 2-second `session_shutdown` timeout still applies for FLUSHING to remote backends

Gate Outcome: ADVANCE to design — session-file-backed audit trail via appendEntry is durable
and survives compaction. The minimal true capability: append-only, per-session, local-file
governance audit records that persist for the session file's lifetime.
```

---

### Finding 2: Hot-Reload of Governance Policies (Section 2.6)

```
Finding: Section 2.6 states rules are loaded once at session creation with no hot-reload.
Rebuttal found rules load once at createAgentSession() and no file-watcher, but `/reload` /
`ctx.reload()` exist. Question: does ctx.reload() re-bucket rules?

Prior Verdict: ? UNVERIFIABLE

New Verdict: ⚠ IRREDUCIBLY UNVERIFIABLE

Deeper Evidence:

1. RULES LOADED ONCE — confirmed:
   /tmp/oh-my-pi/packages/coding-agent/src/sdk.ts:1511-1515
     const rulesResult = ... await loadCapability<Rule>(ruleCapability.id, { cwd });
     const { rulebookRules, alwaysApplyRules } = bucketRules(rulesResult.items, ttsrManager, {
       builtinRules: ttsrSettings.builtinRules,
       disabledRules: ttsrSettings.disabledRules,
     });

   /tmp/oh-my-pi/packages/coding-agent/src/sdk.ts:1772
     setActiveRules([...rulebookRules, ...alwaysApplyRules, ...ttsrManager.getRules()]);

   `bucketRules` and `setActiveRules` are called exactly ONCE, during `createAgentSession`.

2. ctx.reload() IMPLEMENTATION — does NOT re-bucket rules:
   /tmp/oh-my-pi/packages/coding-agent/src/session/agent-session.ts:16273-16276
     async reload(): Promise<void> {
       const sessionFile = this.sessionFile;
       if (!sessionFile) return;
       await this.switchSession(sessionFile);
     }

   `reload()` delegates to `switchSession(sessionFile)` which:
   - Re-reads session entries from the file
   - Restores model, thinking level, messages
   - Emits session_switch event to extensions
   - Does NOT call `bucketRules()`, `loadCapability(ruleCapability.id, ...)`, or `setActiveRules()`
   
   Full trace of switchSession (lines 16285-16490): it rebuilds the conversation context
   from the session file, restores model/thinking state, reconciles bash sessions, and
   emits session_switch — but never touches the rule/TTSR system.

3. /reload-plugins DOES NOT re-bucket rules either:
   /tmp/oh-my-pi/packages/coding-agent/src/slash-commands/builtin-registry.ts:2250-2257
     name: "reload-plugins",
     description: "Reload all plugins (skills, commands, hooks, tools, agents, MCP)",
     handle: async (_command, runtime) => {
       await runtime.reloadPlugins();
       await runtime.output("Plugins reloaded.");
     }

   `reloadPlugins` clears plugin caches and refreshes skills/commands/MCP but does NOT
   re-invoke `bucketRules` or `setActiveRules`. The TTSR rules remain frozen from session
   creation.

4. NO MECHANISM EXISTS to re-bucket rules mid-session:
   /tmp/oh-my-pi/packages/coding-agent/src/capability/rule.ts:276-278
     /** Replace the active rule snapshot. Called once per top-level session. */
     export function setActiveRules(value: readonly Rule[]): void {
       activeRules = value;
     }

   The comment says "Called once per top-level session." There is no `reloadRules()`
   function, no file-watcher on rule directories, and no event that triggers re-bucketing.

5. EXTENSION COMMAND CONTEXT has `reload()` but hooks CANNOT call it:
   /tmp/oh-my-pi/packages/coding-agent/src/extensibility/hooks/types.ts:207-210
     // Parallel to ExtensionCommandContext: hooks intentionally omit
     // `switchSession`, `reload`, `compact`, and `getContextUsage` — those are
     // safe only from extension command handlers, not from the hook execution context.

   Only extension COMMANDS (slash commands registered by extensions) can call reload(),
   not event handlers (tool_call, session_start, etc.). A governance extension's tool_call
   handler cannot trigger reload.

Reasoning:
The source definitively proves that:
- Rules are loaded once at createAgentSession (CONFIRMED)
- ctx.reload() does NOT re-load rules (CONFIRMED — it only reloads session state)
- /reload-plugins does NOT re-load rules (CONFIRMED)
- No mechanism exists for mid-session rule re-bucketing (CONFIRMED)

However, the question "is policy hot-reload FEASIBLE?" depends on whether a future
implementation COULD add rule re-bucketing to the reload path. The existing `setActiveRules`
API is a simple setter — technically, an extension command COULD call `loadCapability` +
`bucketRules` + `setActiveRules` if those were exported/accessible. But:
- `bucketRules` requires a `TtsrManager` instance (not exposed to extensions)
- `setActiveRules` is a module-scoped function (not exposed to extensions)
- The TtsrManager's registered rules have no public `clear()` or `replaceAll()` method

Whether adding this capability is a "supported extension" or a "core change" is a
design-intent question the source cannot answer. The source shows it's architecturally
not impossible (the primitives exist) but not currently wired, and the extension API
surface intentionally does not expose the rule-loading pipeline.

Gate Outcome: RE-INVESTIGATE with named missing fact — whether the team intends for
`setActiveRules` and `bucketRules` to be accessible from extensions (a single exported
function + TtsrManager access would enable it). The source proves current infeasibility
but cannot determine future design intent.
```

---

### Finding 3: Declarative Approval Matrix via `alwaysApply` Rule

```
Finding: Section 3.2 proposes encoding an approval matrix as XML inside an `alwaysApply: true`
rule's body content, with a governance extension reading it from `session_start`. Rebuttal cited
shared-events.ts:28-30 (SessionStartEvent has no payload) and questioned whether the extension
can read rule content by any other means.

Prior Verdict: ? UNVERIFIABLE

New Verdict: ✓ CONFIRMED (proposal feasible/supported — via filesystem read, not session_start)

Deeper Evidence:

1. SessionStartEvent carries NO rule content (rebuttal correct):
   /tmp/oh-my-pi/packages/coding-agent/src/extensibility/shared-events.ts:27-30
     /** Fired on initial session load */
     export interface SessionStartEvent {
       type: "session_start";
     }

   Empty event — no rule body, no configuration payload.

2. Extensions HAVE `exec()` — can read any file:
   /tmp/oh-my-pi/packages/coding-agent/src/extensibility/extensions/loader.ts:223-224
     exec(command: string, args: string[], options?: ExecOptions) {
       return execCommand(command, args, options?.cwd ?? this.cwd, options);
     }

   An extension can run `exec("cat", [".omp/rules/approval-matrix.md"])` to read the
   rule file content at any time. This is documented API surface.

3. Extensions HAVE `cwd` — know where to look:
   /tmp/oh-my-pi/packages/coding-agent/src/extensibility/extensions/types.ts:419-420
     /** Current working directory */
     cwd: string;

4. Extensions ARE full Node.js modules — can import `fs`:
   /tmp/oh-my-pi/packages/coding-agent/src/extensibility/extensions/loader.ts:4-5
     import type * as fs1 from "node:fs";
     import * as fs from "node:fs/promises";

   Extensions load as standard ESM/CJS modules in the same Node.js process. They have
   unrestricted access to `import * as fs from "node:fs"` and can read any file the
   process user has access to.

5. Extension `session_start` handler receives `ctx` with both `cwd` and `exec`:
   /tmp/oh-my-pi/packages/coding-agent/src/extensibility/extensions/types.ts:1071
     on(event: "session_start", handler: ExtensionHandler<SessionStartEvent>): void;

   The handler signature is `(event: SessionStartEvent, ctx: ExtensionContext)` where
   `ctx.cwd` and `ctx.exec(...)` are available.

6. CONCRETE FEASIBLE PATTERN for reading the matrix at session_start:
   ```typescript
   import * as fs from "node:fs";
   import * as path from "node:path";

   export default function (pi: ExtensionAPI) {
     let matrix: ApprovalMatrix;

     pi.on("session_start", async (_event, ctx) => {
       // Read the rule file directly from disk
       const matrixPath = path.join(ctx.cwd, ".omp/rules/approval-matrix.md");
       const content = fs.readFileSync(matrixPath, "utf-8");
       // Parse the XML matrix from the rule body (after frontmatter)
       matrix = parseApprovalMatrix(content);
     });

     pi.on("tool_call", async (event, ctx) => {
       // Enforce the matrix on every tool call
       const decision = matrix.evaluate(event.toolName, event.input);
       if (decision === "deny") {
         return { block: true, reason: "Denied by approval matrix" };
       }
     });
   }
   ```

7. `alwaysApply` rules DO get injected into system prompt — dual purpose:
   /tmp/oh-my-pi/packages/coding-agent/src/capability/rule-buckets.ts:33
     export function bucketRules(rules, ttsrManager, options) { ... }

   An `alwaysApply: true` rule's body is injected into the system prompt (instructing
   the LLM about the policy) AND the same file can be read by the governance extension
   from disk (enforcing the policy mechanically). The dual-use pattern works.

Reasoning:
The proposal's MECHANISM is wrong (reading from session_start event payload) but the
INTENT is fully achievable through a different path: filesystem read. Extensions are
full Node.js modules with unrestricted fs access and the `ctx.exec()` API. At
`session_start`, the extension can read `.omp/rules/approval-matrix.md` from disk,
parse the embedded XML matrix, and cache it for enforcement during `tool_call` events.

The `alwaysApply: true` frontmatter ensures the matrix is ALSO injected into the LLM's
system prompt (advisory compliance) while the extension enforces it mechanically
(blocking non-compliant tool calls). This is a clean dual-enforcement pattern.

Gate Outcome: ADVANCE to design — declarative approval matrix via alwaysApply rule +
extension filesystem read is feasible. The minimal true capability: extension reads
rule file from disk at session_start, parses embedded matrix DSL, enforces via tool_call
blocking. The LLM sees the matrix as advisory context; the extension enforces it hard.
```
