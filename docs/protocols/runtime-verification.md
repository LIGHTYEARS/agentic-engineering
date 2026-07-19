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

### V-11 — `pi-iso` isolated subagent produces a git patch artifact

Proves foundation `02-orchestration-search.md` C-ORCH-* runtime substrate: when a `task` subagent is spawned with `isolated: true` under `task.isolation.mode ≠ none`, it executes inside an APFS-clonefiled worktree at `~/.omp/wt/t<hash>/m` (source: `packages/coding-agent/src/task/worktree.ts:411-444`, `packages/utils/src/dirs.ts:601-603`), its diff is captured as a `.patch` file in the parent session's artifacts dir, then `merge: patch` (default) applies it via `git apply` to the real repo and destroys the worktree (source: `packages/coding-agent/src/task/isolation-runner.ts:305-341`).

Two probe-design facts, both critical:
- **`task.isolation.mode` is a schema switch, not a global gate.** Setting it to non-`none` only exposes an `isolated?: boolean` parameter on the `task` tool. The model MUST pass `isolated: true` in the tool call to enter iso (`packages/coding-agent/src/task/index.ts:596-611`, `packages/coding-agent/src/task/structured-subagent.ts:284-291`). No `isolated: true` = no iso, silently. The prompt MUST instruct the model explicitly.
- **Parent-workspace state is NOT a valid probe.** With `merge: patch` (default), the flow is: subagent edits inside worktree → patch captured → git-apply to parent → worktree cleaned. Parent being modified is *success*, not failure. Grep the `.patch` artifact instead.

```bash
d=/tmp/omp-v11-run && sdir=/tmp/omp-v11-sessions && rm -rf "$d" "$sdir" && mkdir "$d" "$sdir"
cat > /tmp/omp-v11-iso.yml <<'EOF'
task:
  isolation:
    mode: apfs
EOF
( cd "$d" && git init -q \
    && git config user.email v11@probe.local && git config user.name "V-11 Probe" \
    && echo "hello base" > file.txt && git add file.txt && git commit -q -m base \
    && timeout 240 omp -p --model 'local_llm/c_o_new_thinking__dev' --approval-mode yolo \
        --session-dir "$sdir" --config /tmp/omp-v11-iso.yml \
        "Call the task tool with parameters task='overwrite file.txt so its only content is the single line V11_ISO_EDIT (use the write tool)' and isolated=true. After the subagent finishes, reply only with V11_DONE." ) | tail -1
patch=$(find "$sdir" -name "*.patch" | head -1)
echo "patch_artifact_present=$([ -n "$patch" ] && echo yes || echo no)"
grep -qF -- '-hello base' "$patch" && echo "minus_line: yes" || echo "minus_line: no"
grep -qF -- '+V11_ISO_EDIT' "$patch" && echo "plus_line: yes"  || echo "plus_line: no"
grep -q 'diff --git a/file.txt b/file.txt' "$patch" && echo "diff_header: yes" || echo "diff_header: no"
rm -rf "$d" "$sdir" /tmp/omp-v11-iso.yml
```
Expected: last model output `V11_DONE`; `patch_artifact_present=yes`; `minus_line: yes`; `plus_line: yes`; `diff_header: yes`. The `.patch` file lives at `<session-dir>/<uuid>/<subagent-name>.patch` (subagent name is auto-generated, e.g. `SuccessfulTakin.patch`). Wall time ~130–140s (dominated by upstream thinking-model latency + iso lifecycle setup/teardown; iso overhead itself is small).

**Trap**: dev/non-thinking models frequently ignore `isolated=true` and either skip the task tool entirely or drop the parameter. Use a thinking-class model. Even so, the `apply=false` variant of the tool call is unreliable across runs — models tend to omit it. Rely on the `.patch` artifact as the ground-truth signal, not on parent-file state.

### V-12 — Mnemopi `retain` → cross-session `recall` round-trip + bank scoping

Proves foundation `09-learning.md` runtime substrate: facts retained via `retain` are durably persisted and retrievable by a **fresh subprocess** (`--no-session`) via semantic `recall`. Per-project bank scoping ensures memory isolation across different working directories.

**Prerequisite:** This probe requires a V12 token to have been retained ONCE from the target cwd. Do this manually or via the parent session before running the probe:
```bash
( cd ~/workspace/ai4se/agentic-engineering && timeout 60 omp -p --model 'local_llm/c_o_new__dev' \
    --approval-mode yolo --no-session \
    "Use the retain tool (xd://retain) with items=[{content:'V12_PROBE_TOKEN_ZYGOMORPHIC: The nine-plane framework uses zygomorphic lattice reduction as its foundational algebra. This sentence is uniquely retrievable.',context:'V-12 runtime verification probe.'}]. Reply only: V12_RETAINED." ) | tail -1
```
Expected: `V12_RETAINED`.

**Positive probe (same cwd = same bank → HIT):**
```bash
( cd ~/workspace/ai4se/agentic-engineering && timeout 120 omp -p --model 'local_llm/c_o_new__dev' \
    --approval-mode yolo --no-session \
    "Use the recall tool (xd://recall) with query 'zygomorphic lattice reduction nine-plane algebra'. If the result contains 'V12_PROBE_TOKEN_ZYGOMORPHIC', reply with exactly: V12_RECALL_HIT. Otherwise reply: V12_RECALL_MISS." ) | tail -1
```
Expected: `V12_RECALL_HIT`.

**Negative control (different cwd = different bank → MISS):**
```bash
( cd /tmp && timeout 120 omp -p --model 'local_llm/c_o_new__dev' \
    --approval-mode yolo --no-session \
    "Use the recall tool (xd://recall) with query 'zygomorphic lattice reduction nine-plane algebra'. If the result contains 'V12_PROBE_TOKEN_ZYGOMORPHIC', reply with exactly: V12_RECALL_HIT. Otherwise reply: V12_RECALL_MISS." ) | tail -1
```
Expected: `V12_RECALL_MISS`.

**What this proves:**
- `retain` writes survive process exit (not in-process-only)
- `recall` performs semantic retrieval (the query `zygomorphic lattice reduction nine-plane algebra` is a paraphrase of the stored content, not an exact match; FTS alone would miss it unless the tokenizer happens to overlap)
- Bank scoping is per-cwd: same project directory shares memories, different directory does not see them
- `--no-session` does NOT suppress mnemopi (session-file persistence and memory persistence are orthogonal concerns; source: `packages/coding-agent/src/mnemopi/backend.ts:70-71` — gate is `sessionId` existence, which `--no-session` still provides via in-memory session)

**Storage architecture note:** On this workstation `memory.backend = mnemopi` is the active setting. Despite the source code resolving to `getMemoriesDir(agentDir)/mnemopi/mnemopi.db`, no SQLite file by that name exists on disk — the retained token was found only inside the current session's JSONL file. The mnemopi implementation on this version may use session-embedded indexing with cross-session FTS against the session archive (under `~/.omp/agent/sessions/<project-slug>/`). The behavioral contract (retain in session A → recall in session B) is what this probe verifies; the internal persistence format is an implementation detail subject to version drift.

---

## 4. Known runtime traps

- **Native addon version sentinel mismatch after auto-update.** If `omp` auto-installs a new version mid-session (bun package resolves a newer `@oh-my-pi/pi-coding-agent`), the loader may briefly load an old `pi_natives.darwin-arm64.node` whose sentinel (`__piNativesV17_0_2` vs `__piNativesV17_0_4`) does not match. The error is verbose (`Failed to load pi_natives native addon for darwin-arm64`, three tried paths). Fix: re-run the same command once — the loader resolves the up-to-date `.node` and continues. Observed during this session: `17.0.2 → 17.0.4`.
- **`omp usage` says "No credentials found."** This is technically correct — the credential *broker vault* is empty because auth is via env-passed `apiKey` in `models.yml`. Not a failure signal.
- **`--model <fuzzy>` can resolve unexpectedly.** Fuzzy matching can pick a completely different provider if the string is ambiguous. Always qualify: `--model local_llm/<id>`.
- **Public-key routes 401.** `ANTHROPIC_API_KEY` is present in env but the value is an internal token; direct-to-Anthropic calls (e.g. `--model anthropic/claude-haiku-4-5`) return `401 invalid x-api-key`. Route via `local_llm/*` (LiteLLM handles upstream credentials).
- **Config keys are dotted paths, not YAML paths.** `omp config get modelRoles.default` does not work (`Unknown setting: modelRoles.default`). To inspect roles, read the YAML directly.
- **Iso probes need `isolated: true` in the tool call, not just the setting.** `task.isolation.mode = apfs` alone does nothing until a `task` tool call passes `isolated: true`. Prompting "use the task tool" is not enough — the model must be told to pass the parameter, AND it must be a thinking-class model (`local_llm/c_o_new_thinking__dev` +). Small/dev models routinely skip both the tool and the parameter.
- **Iso "success" looks like parent-file mutation.** `merge: patch` (default) applies the subagent's diff back to the real repo and destroys the worktree by design. Parent unchanged ≠ iso worked; parent changed ≠ iso failed. Grep `<session-dir>/<uuid>/*.patch` for the artifact — that is the ground truth.

---

## 5. Environment drift log

Record every observed divergence from §3 here. Do NOT edit §3's expected values silently.

- **2026-07-18T16:48Z** — mid-session auto-update: `omp/17.0.2 → 17.0.4`. Native addon sentinel briefly mismatched (loader tried V17.0.2 sentinel on V17.0.4 `.node`); one retry resolved it. TTSR rule count unchanged (29). LiteLLM catalog unchanged.

---

## 6. What this document does NOT prove

Being explicit so the doc is not overclaimed:

- Does NOT verify `<memories>` auto-injection at session start (only cross-session recall via explicit tool call). Proving injection would require inspecting the system prompt of a fresh subprocess, which is not observable from the outside.
- Does NOT verify `swarm-extension` behavior (only that it is *loaded* per `~/.omp/config.json`).
- Does NOT exercise checkpoint restore (`checkpoint.enabled = false` on this workstation).
- V-10 verifies JSONL persistence but does NOT verify the internal claim from foundation that `#entries` is never pruned — that requires forcing a compaction and inspecting `firstKeptEntryId` semantics; deferred.
- Does NOT verify the mnemopi embedding pipeline specifically (whether similarity is purely FTS-based or cosine-on-vector); the positive recall could be hitting on shared FTS tokens. A definitive vector-similarity proof would need a query with zero lexical overlap that still hits — deferred.

These are the highest-value next verification tasks; each becomes a new V-N entry when added, per the discipline in §2.
