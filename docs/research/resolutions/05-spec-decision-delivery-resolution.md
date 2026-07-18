# Resolution: Spec, Decision, and Delivery Planes — Unverifiable Findings

## Summary

| Verdict | Count |
|---------|-------|
| ✓ CONFIRMED (partially feasible) | 2 |
| ✗ REFUTED | 1 |

---

## Resolutions

### Finding 1: Cross-Plane Traceability Flows

**Finding:** "Spec→Decision: Constraints inform option generation. Decision→Delivery: ADRs gate delivery scope. Delivery→Spec: Runtime signals update acceptance criteria status." (Section 6.4 Integration Architecture) — Rebuttal says these are aspirational hand-waves with no mechanism.

**Prior Verdict:** ? UNVERIFIABLE

**New Verdict:** ✓ CONFIRMED (minimal substrate exists that a design could build on)

**Deeper Evidence:**

1. **Session tree linkage via `parentId` chains:** Every `SessionEntry` carries `{ id, parentId, timestamp }` forming a tree (`/tmp/oh-my-pi/packages/coding-agent/src/session/session-entries.ts:48-53`). The `SessionHeader` carries `parentSession?: string` (`session-entries.ts:35`) linking forked/branched sessions to their parent. `SessionManager.#freshEntryFields()` sets `parentId: this.#index.leafId()` (`session-manager.ts:829-834`), creating a linear chain within a session. Branching creates `BranchSummaryEntry` nodes with `fromId` pointing back to the decision point (`session-manager.ts:1781-1785`).

2. **Plan reference threading from spec→delivery:** `AgentSession` maintains `#planReferencePath` (`agent-session.ts:1927`) set during plan approval via `setPlanReferencePath(path)` (`agent-session.ts:7985-7986`). This path is injected into subagent system prompts (`subagent-system-prompt.md:13-21`) via the template variable `{{planReferencePath}}` + `{{planReference}}`. The `task` executor receives `planReference: { path, content }` (`executor.ts:278`) and forwards it to subagents. This means the plan file identity propagates from the spec phase into every delivery-phase subagent.

3. **CustomEntry and CustomMessageEntry for extension-authored records:** `CustomEntry<T>` (`session-entries.ts:124-128`) persists arbitrary typed data with `customType` discriminator. `CustomMessageEntry<T>` (`session-entries.ts:201-209`) participates in LLM context. Both are appended via `appendCustomEntry(customType, data)` and `appendCustomMessageEntry(customType, content, display, details)` (`session-manager.ts:1600-1642`). These are general-purpose cross-entry storage slots that an extension or design could use to link decisions to deliveries.

4. **Session entry tree serves as an implicit audit trail:** The entire session transcript forms a tree where every tool call, ask-tool decision, plan approval, and commit are linked by `parentId`. Tool results from `ask` store `selectedOptions` in structured `details`. This chain is queryable post-hoc.

**Reasoning:** The proposal claims cross-plane traceability as if it's a coherent mechanism — it isn't. But the rebuttal's claim that there is "no mechanism" is also overstated. There IS a minimal substrate: (a) the session entry tree with `id/parentId` chains, (b) `parentSession` linkage across session forks, (c) `planReferencePath` threading from plan approval into subagent execution, (d) `CustomEntry`/`CustomMessageEntry` for arbitrary structured persistence. These don't form an automated pipeline, but they form a substrate a design could build on without new primitives. The proposal's specific claims about constraint→option, ADR→delivery-gate, and signal→acceptance loops are unimplemented, but the substrate for implementing them (entry linkage + custom entries + plan reference path) exists.

**Gate Outcome:** ADVANCE to design — the minimal true capability is: session entry trees provide implicit traceability; `planReferencePath` threads spec identity into execution; `CustomEntry` provides the persistence slot for cross-plane records. A design should build explicit traceability on these, not assume it exists.

---

### Finding 2: Memory Pipeline for Option-Shape Retrieval → Ask `recommended`

**Finding:** "Leverage the OMP memory pipeline to retain decision patterns. When an `ask` tool with similar option shapes appears, inject prior decisions as context for the `recommended` field." (Section 3.3, point 3) — Rebuttal says memory stores free-text facts, not structured decision records.

**Prior Verdict:** ? UNVERIFIABLE

**New Verdict:** ✗ REFUTED (no structured decision-record storage/retrieval; memory pipeline stores free-text transcripts, not option-shape indexed records)

**Deeper Evidence:**

1. **Memory extraction stores free-text categories, not structured records:** The mnemopi extraction pipeline (`/tmp/oh-my-pi/packages/mnemopi/src/core/extraction.ts:42-69`) uses an LLM prompt to extract `{ facts, instructions, preferences, timelines, kg }` — all string arrays plus KG triples. There is no "decision" category, no "option-shape" extraction, and no structured decision record format.

2. **Retention stores full conversation transcripts as opaque text blobs:** `MnemopiSessionState.retainMessages()` (`/tmp/oh-my-pi/packages/coding-agent/src/mnemopi/state.ts:464-491`) calls `prepareRetentionTranscript(messages, true)` which produces a flattened text transcript, then stores it via `rememberInScope(transcript, { source: "coding-agent-transcript", memoryType: "episode", ... })`. The stored content is a flat string with metadata like `session_id`, `message_count`, `cwd`. There is no ask-tool-specific extraction.

3. **MemoryRow has no decision-record schema:** The `MemoryRow` type (`/tmp/oh-my-pi/packages/mnemopi/src/types.ts:8-29`) has fields: `id`, `content` (string), `source`, `timestamp`, `session_id`, `importance`, `metadata_json`, `veracity`, `memory_type`, `trust_tier`. The `content` field is a plain string. There is no structured `options_shape`, `selected_option`, or `decision_context` field.

4. **Recall uses vector/FTS similarity, not structural option-shape matching:** The recall pipeline (`/tmp/oh-my-pi/packages/mnemopi/src/core/beam/recall.ts:1-80`) retrieves by cosine similarity on embeddings + full-text search scoring. It operates on text content. There is no mechanism to match by "option shape" (number of options, option labels, domain type).

5. **Ask tool's `recommended` field is agent-supplied, not memory-injected:** The `ask` tool schema (`/tmp/oh-my-pi/packages/coding-agent/src/tools/ask.ts:69`) defines `recommended?` as a number (index). It is set by the LLM in its tool call arguments. There is no pre-filling mechanism, no memory injection hook, and no code path that reads from the memory pipeline into the `recommended` field before the ask executes.

**Reasoning:** The proposal's claim requires: (a) structured decision-record storage (doesn't exist — memory stores text), (b) option-shape similarity retrieval (doesn't exist — recall uses text/vector similarity), (c) injection into the `recommended` field of future ask calls (doesn't exist — `recommended` is LLM-authored). All three prerequisites are absent from both the memory pipeline and the ask tool. The gap is not "could be built with minor extensions" — it requires a fundamentally different memory schema and a new injection mechanism.

**Gate Outcome:** DISCARD — the proposal assumes a memory architecture that doesn't exist and can't be achieved by extending the current pipeline without redesigning both the storage schema and the ask tool's parameter-sourcing path.

---

### Finding 3: Runtime TTSR Rule Generation from Plan Content

**Finding:** "Auto-generate `.omp/rules/` from the plan's hard constraints. Example: constraint 'no new dependencies' → rule with `condition: \"tool:bash.*install|tool:write.*package.json\"`." (Section 6.1, Priority 5: "Medium" effort) — Rebuttal says rules are disk-discovered at session init, no dynamic registration API.

**Prior Verdict:** ? UNVERIFIABLE

**New Verdict:** ✓ CONFIRMED (partially feasible — mid-session live registration IS possible via `TtsrManager.addRule()`)

**Deeper Evidence:**

1. **Rules ARE loaded at session init as a one-shot:** `createAgentSession()` in `/tmp/oh-my-pi/packages/coding-agent/src/sdk.ts:1502-1521` runs `loadCapability<Rule>(ruleCapability.id, { cwd })` followed by `bucketRules(rulesResult.items, ttsrManager, ...)` exactly once. The `TtsrManager` is created, populated, and then handed to the session. There is no periodic re-scan or file-watcher for new rule files.

2. **HOWEVER: `TtsrManager.addRule(rule)` IS exposed on the session object and callable mid-session:** The `OmfgController` (`/tmp/oh-my-pi/packages/coding-agent/src/modes/controllers/omfg-controller.ts:260-262`) demonstrates live mid-session rule registration:
   ```ts
   #registerLive(rule: Rule): void {
       this.ctx.session.ttsrManager?.addRule(rule);
   }
   ```
   This is called after writing a rule file to disk AND immediately registering it with the live TTSR manager. The OMFG flow: user complains → LLM generates a rule → rule is saved to `.omp/rules/` → `#registerLive(savedRule)` adds it to the active manager → the rule takes effect THIS session.

3. **`addRule` has no "init-only" guard:** `/tmp/oh-my-pi/packages/coding-agent/src/export/ttsr.ts:304-340` shows `addRule(rule: Rule): boolean` only checks: (a) TTSR is enabled, (b) rule has valid condition, (c) not a duplicate name. There is no lifecycle check preventing mid-session calls.

4. **The session exposes `ttsrManager` publicly:** `AgentSession` has a public getter `get ttsrManager(): TtsrManager | undefined` (`agent-session.ts:3957-3958`), making `session.ttsrManager?.addRule(rule)` callable from any code with access to the session object — extensions, hooks, custom tools, or internal controllers.

5. **No auto-discovery of newly written rule files exists:** Writing a `.omp/rules/foo.md` file via the `write` tool does NOT trigger re-discovery. The `loadCapability` call happened once at init. The OMFG controller works around this by doing BOTH: writing the file (for future sessions) AND calling `addRule` (for the current session).

**Reasoning:** The rebuttal was partially correct — there is no file-watch mechanism and no "constraint→regex translator." But the core claim that "rules are loaded only at session init with no dynamic registration" is WRONG. The `TtsrManager.addRule()` method is exposed on the session object and is actively used mid-session by the OMFG controller. An agent that writes a rule file AND constructs a `Rule` object and calls `session.ttsrManager.addRule(rule)` CAN register a TTSR rule that takes effect immediately in the current session. The missing piece is the "constraint→regex translator" (parsing plan constraints into regex patterns), which is pure logic with no infrastructure dependency.

**Gate Outcome:** ADVANCE to design — the minimal true capability is: `TtsrManager.addRule(rule)` provides mid-session dynamic rule registration accessible via `session.ttsrManager`. An extension, hook, or custom tool CAN generate TTSR rules from plan content and register them live. The design needs only: (a) a constraint→regex translation function, (b) a hook on plan-approval that generates rules and calls `addRule`. The OMFG controller proves this pattern works today.
