# Runtime Verification Protocol — omp on this workstation

> **Authority:** Referenced from `AGENTS.md`. Rules mandatory once loaded.
> **Purpose:** `foundation/` is a source-code-derived spec (`file:line` citations from `/tmp/oh-my-pi`). This document is its **empirical counterpart**: what the *running* `omp` binary on this workstation actually does. Every claim below is a shell command you can re-run to reproduce.
>
> **Provenance:** Recorded 2026-07-18. Every command in §3 was executed against the live binary at the time of writing; expected outputs are pinned. If your re-run diverges, the environment has drifted — do NOT quietly amend claims; open an "environment drift" section at the bottom of this file.

---

## 1. Ground truth: how omp is wired on this box

**Do NOT infer routing from env vars.** `ANTHROPIC_API_KEY` / `AZURE_OPENAI_*` are present but they are *side-signals*, not the active route. The active route is defined by two YAML files:

| File | Role | Source of truth for |
|---|---|---|
| `~/.omp/agent/config.yml` | Global agent config | `modelRoles.*` (default/smol/slow/plan/task/vision/…), extensions, features |
| `~/.omp/agent/models.yml` | Provider + model registry | `providers.<name>.baseUrl`, `api` shape, per-model compat flags |
| `~/.omp/config.json` | Extension pin | `extensions: ["packages/swarm-extension"]` |

**The active route (verified):**

```
omp session
   │  modelRoles.default (config.yml)
   ▼
local_llm/c_o_new_thinking__dev
   │  providers.local_llm.baseUrl (models.yml)
   ▼
http://localhost:8082/v1        ← local LiteLLM
   │  (LiteLLM internal routing)
   ▼
upstream real models (c_o_new / gpt-5.4 / gemini-3.1-pro / kimi-k2.6 / glm-5.1 / …)
```

**`modelRoles`** (pinned; from `~/.omp/agent/config.yml`):

| Role | Model |
|---|---|
| `default` | `local_llm/c_o_new_thinking__dev` |
| `smol` | `local_llm/c_o_new__thinking__dev:off` |
| `slow` | `local_llm/gpt-5.4__272k` |
| `plan` | `local_llm/c_o_new__max` |
| `vision` | `local_llm/c_o_new__max` |
| `task` | `local_llm/c_o_new__dev:off` |
| `commit` | `local_llm/c_o_new__dev:off` |
| `designer` | `local_llm/c_o_new_thinking__dev:off` |
| `advisor` | `local_llm/gpt-5.4__272k` |

**`local_llm` provider declaration** (from `~/.omp/agent/models.yml`):

- `baseUrl: http://localhost:8082/v1`
- `api: openai-completions` (OpenAI Chat Completions shape, not Anthropic)
- Compat flags: `supportsStore: false`, `supportsDeveloperRole: false`, `supportsReasoningEffort: false`, `supportsStrictMode: false`, `toolStrictMode: none` — LiteLLM proxy does not surface these OpenAI-newer features.
- 18+ model IDs (`c_o_new__dev`, `openrouter-{1,2,3}o__{dev,max}`, `gpt-5.4__272k`, `gpt-5.5__800k`, `gpt-5.6-{terra,sol}__800k`, `gemini-3.1-pro-preview`, `kimi-k2.6`, `glm-5.1`, …).

**Environment at recording time:**

| Component | Version |
|---|---|
| `omp` | `17.0.4` |
| Bun | `1.3.14` |
| Node | `v24.12.0` |
| macOS | Darwin 25.3.0 arm64 (Apple M3 Pro) |
| Binary path | `/Users/bytedance/.bun/bin/omp` |

---

## 2. Method rules (do this, in this order)

1. **Never claim a route from env alone.** Read `~/.omp/agent/config.yml` `modelRoles` and `~/.omp/agent/models.yml` `providers` first.
2. **Probe the local endpoint before running omp.** If LiteLLM (`:8082`) is not answering, nothing else in §3 is meaningful.
3. **Verify with the omp *binary*, not source reading.** `/tmp/oh-my-pi` is one release; the installed binary may be ahead (see §4).
4. **Isolate every end-to-end run** with `--no-session` + a scratch `cwd` (`/tmp/omp-verify-*`) so nothing contaminates the parent session.
5. **Redact** every token/apikey line when reading or pasting config content: `sed -E 's/(apiKey|token|secret|Authorization)([^:]*): *.*/\1\2: <REDACTED>/gi'`.
6. **Run everything in §3** whenever you touch this document. Divergence must be recorded in §5, not silently patched into §3.

---

## 3. The verification checks (re-runnable, exact expected output)

Every command below was executed against the live binary. Copy-paste-run.

### V-1 — Binary is installed and self-reports a version

```bash
which omp && omp --version | tail -1
```
Expected shape: `/Users/bytedance/.bun/bin/omp` on line 1, `omp/<semver>` on line 2. Any error mentioning `pi_natives` version sentinel = the native addon just updated out from under the loader; re-run once and it self-heals (see §4).

### V-2 — Local LiteLLM endpoint is up

```bash
curl -sS -o /dev/null -w "http=%{http_code} t=%{time_total}s\n" http://localhost:8082/v1/models --max-time 3
```
Expected: `http=200 t=<something under 0.1s>` (sub-10ms is typical on localhost).

### V-3 — LiteLLM exposes the model catalog omp expects

```bash
curl -sS http://localhost:8082/v1/models --max-time 3 \
  | python3 -c 'import sys,json;d=json.load(sys.stdin);ms=d["data"];print("count=",len(ms));[print(" -",m["id"]) for m in ms[:12]]'
```
Expected: `count= >=18` and IDs matching `~/.omp/agent/models.yml` `providers.local_llm.models[].id` (`c_o_new__dev`, `c_o_new_thinking__dev`, `openrouter-{1,2,3}o__{dev,max}`, `c_o_new__max`, `c_o_new_thinking__max`, `glm-5.1`, `gpt-5.4__272k`, …).

### V-4 — omp reads the internal `omp://` doc URI

```bash
omp read "omp://" | head -3
```
Expected: `1|# Documentation` then `2|` (blank) then `3|118 files available:` (count may drift with version).

### V-5 — TTSR loads real rules including the user's custom one

```bash
omp ttsr list 2>&1 | head -4
```
Expected: `TTSR rules (29)` header, then the three `[native]` rules `feature-workflow-impl-gate`, `heavy-refactor-impl-gate`, `no-patch-mindset` (this last one has scope `text, thinking` and both Chinese and English keyword sets — it is the user's own rule).

### V-6 — TTSR text-source matching fires (advisory, not blocking)

```bash
omp ttsr test --source text 'let me just apply a quick fix, the simplest band-aid workaround' --json \
  | python3 -c 'import sys,json;d=json.load(sys.stdin);print("evaluated=",d["evaluated"]);print("triggered=",[t["name"] for t in d["triggered"]])'
```
Expected: `evaluated= 29` and `triggered= ['no-patch-mindset']`. The regex that fired is the English half (`\b(simplest|...|band.?aid|...)\b`). Text-source rules do NOT block execution — they inject a reminder on the next generation (this is the "advisory" behaviour that `foundation/03-evaluation.md` C-eval-3 pins).

### V-7 — TTSR tool-source matching fires (regex + AST)

```bash
omp ttsr test --source tool --tool edit --path /tmp/probe.ts 'const x: any = 1' --json \
  | python3 -c 'import sys,json;d=json.load(sys.stdin);print("triggered=",[t["name"] for t in d["triggered"]])'
```
Expected: `triggered= ['ts-no-any']` (regex-source rule).

AST-source rule (structural match via tree-sitter, not regex):
```bash
printf 'package main\nfunc f(n int){ for i := 0; i < n; i++ { _ = i } }\n' \
  | omp ttsr test --file - --path /tmp/probe.go --json \
  | python3 -c 'import sys,json;d=json.load(sys.stdin);print("triggered=",[t["name"] for t in d["triggered"]])'
```
Expected: `triggered= ['go-range-int']`.

### V-8 — Nested end-to-end run through the real route

Isolated scratch dir; real model call; real `read` tool; hostile approval-mode so no interactive prompts.

```bash
d=/tmp/omp-verify-$$ && mkdir "$d" && printf 'line-one\nline-two\nline-three\n' > "$d/sample.txt"
( cd "$d" && timeout 150 omp -p --model 'local_llm/c_o_new__dev' --approval-mode yolo --no-session \
    "Use the read tool to read sample.txt in the current directory, then reply with exactly the second line's text and nothing else." )
echo "exit=$?"
rm -rf "$d"
```
Expected: last non-empty line is exactly `line-two`, `exit=0`. Total wall time on a warm cache is ~15–25s (dominated by upstream model latency, not omp overhead).

### V-9 — Extension `tool_call` handler blocks tool execution before dispatch

Proves foundation `01-governance.md` C-GOV-* runtime substrate: an extension returning `{ block, reason }` from `tool_call` intercepts the call *before* it runs, and the reason string reaches the model as the tool result.

```bash
mkdir -p /tmp/omp-v9-probe && cat > /tmp/omp-v9-probe/block-bash.ts <<'TS'
import type { ExtensionAPI } from "@oh-my-pi/pi-coding-agent";
export default function v9Probe(pi: ExtensionAPI) {
  pi.setLabel("V-9 tool_call blocker");
  pi.on("tool_call", async (event) => {
    if (event.toolName !== "bash") return;
    const cmd = (event.input as { command?: string }).command ?? "";
    if (cmd.includes("V9_SENTINEL")) return { block: true, reason: "V9_BLOCKED_BY_EXTENSION" };
  });
}
TS
d=/tmp/omp-v9-run && rm -rf "$d" && mkdir "$d"
( cd "$d" && timeout 120 omp -p --model 'local_llm/c_o_new__dev' --approval-mode yolo --no-session \
    -e /tmp/omp-v9-probe/block-bash.ts \
    "Run this exact bash command using the bash tool: echo V9_SENTINEL. If the tool call fails or is blocked, reply with exactly the failure/block reason string and nothing else." ) | tail -1
rm -rf "$d"
```
Expected: last line is exactly `V9_BLOCKED_BY_EXTENSION`. The command never runs (the shell never echoes `V9_SENTINEL` back through the tool result). This is a *pre-execution* block, not a post-hoc filter.

### V-10 — Session-file audit persistence (append-only JSONL, resume appends)

Proves foundation `01-governance.md` C-GOV-1 runtime substrate: every message pass persists to disk as a line-delimited JSON entry; resuming an existing session appends new entries without touching prior ones (append-only).

```bash
d=/tmp/omp-v10-run && sdir=/tmp/omp-v10-sessions && rm -rf "$d" "$sdir" && mkdir "$d" "$sdir"
( cd "$d" && timeout 90 omp -p --model 'local_llm/c_o_new__dev' --approval-mode yolo \
    --session-dir "$sdir" \
    "Reply with exactly the token V10_TOKEN_BRAVO — nothing else." ) | tail -1
sess=$(find "$sdir" -maxdepth 1 -name "*.jsonl" -type f | head -1)
before_lines=$(wc -l <"$sess") ; sid=$(basename "$sess" .jsonl)
( cd "$d" && timeout 90 omp -p --model 'local_llm/c_o_new__dev' --approval-mode yolo \
    --session-dir "$sdir" --resume "$sid" \
    "Reply with exactly V10_TOKEN_CHARLIE — nothing else." ) | tail -1
after_lines=$(wc -l <"$sess")
echo "lines: $before_lines -> $after_lines ; BRAVO count: $(grep -c V10_TOKEN_BRAVO "$sess") ; CHARLIE count: $(grep -c V10_TOKEN_CHARLIE "$sess")"
rm -rf "$d" "$sdir"
```
Expected: two model outputs (`V10_TOKEN_BRAVO`, `V10_TOKEN_CHARLIE`); `lines: 8 -> 11` (±1 depending on `title`/`model_change` bookkeeping); `BRAVO count: 2` (still present after resume — proves append-only, not truncate-rewrite); `CHARLIE count: 2` (newly appended). Session filename shape is `<UTC-timestamp>_<uuid>.jsonl` (NOT `session.jsonl`).

Distinct entry types observed on a minimal one-turn session: `session`, `title`, `model_change`, `message` (user + assistant), `custom_message`, `custom` — the top-level `type` field discriminates.

---

## 4. Known runtime traps

- **Native addon version sentinel mismatch after auto-update.** If `omp` auto-installs a new version mid-session (bun package resolves a newer `@oh-my-pi/pi-coding-agent`), the loader may briefly load an old `pi_natives.darwin-arm64.node` whose sentinel (`__piNativesV17_0_2` vs `__piNativesV17_0_4`) does not match. The error is verbose (`Failed to load pi_natives native addon for darwin-arm64`, three tried paths). Fix: re-run the same command once — the loader resolves the up-to-date `.node` and continues. Observed during this session: `17.0.2 → 17.0.4`.
- **`omp usage` says "No credentials found."** This is technically correct — the credential *broker vault* is empty because auth is via env-passed `apiKey` in `models.yml`. Not a failure signal.
- **`--model <fuzzy>` can resolve unexpectedly.** Fuzzy matching can pick a completely different provider if the string is ambiguous. Always qualify: `--model local_llm/<id>`.
- **Public-key routes 401.** `ANTHROPIC_API_KEY` is present in env but the value is an internal token; direct-to-Anthropic calls (e.g. `--model anthropic/claude-haiku-4-5`) return `401 invalid x-api-key`. Route via `local_llm/*` (LiteLLM handles upstream credentials).
- **Config keys are dotted paths, not YAML paths.** `omp config get modelRoles.default` does not work (`Unknown setting: modelRoles.default`). To inspect roles, read the YAML directly.

---

## 5. Environment drift log

Record every observed divergence from §3 here. Do NOT edit §3's expected values silently.

- **2026-07-18T16:48Z** — mid-session auto-update: `omp/17.0.2 → 17.0.4`. Native addon sentinel briefly mismatched (loader tried V17.0.2 sentinel on V17.0.4 `.node`); one retry resolved it. TTSR rule count unchanged (29). LiteLLM catalog unchanged.

---

## 6. What this document does NOT prove

Being explicit so the doc is not overclaimed:

- Does NOT verify anything about `mnemopi` (memory) beyond what shows up in `config list`.
- Does NOT verify `swarm-extension` behavior (only that it is *loaded* per `~/.omp/config.json`).
- Does NOT exercise `pi-iso` isolation / `diff()` collection (`foundation/02-orchestration-search.md` C-ORCH-*). Attempted a V-11 probe: overlay `--config task.isolation.mode=apfs` was accepted, `task.isolation.mode` supports `apfs` on this APFS-mounted filesystem, but the test could not reliably force the target model (`local_llm/c_o_new__dev`, a small dev model) to spawn a `task` subagent — the model wrote directly to `file.txt` in the parent workspace via the `write` tool instead. This is a **probe-design gap, not evidence against iso**: the iso path was never actually entered, so its behavior remains unverified. Re-attempt requires either (a) a probing model reliably obedient to "use the task tool", or (b) a scripted programmatic task-tool invocation not routed through model choice. Not in scope for this pass.
- Does NOT exercise checkpoint restore (`checkpoint.enabled = false` on this workstation).
- V-10 verifies JSONL persistence but does NOT verify the internal claim from foundation that `#entries` is never pruned — that requires forcing a compaction and inspecting `firstKeptEntryId` semantics; deferred.

These are the highest-value next verification tasks; each becomes a new V-N entry when added, per the discipline in §2.
