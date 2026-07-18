# Resolution: 03-evaluation-plane UNVERIFIABLE Findings

## Summary

- **Confirmed (with qualification)**: 2
- **Refuted**: 0
- **Irreducibly Unverifiable**: 0

Both findings resolve to CONFIRMED but with significant narrowing of the original proposal's claims. The metaharness provides more comparison infrastructure than the rebuttal acknowledged (head-to-head task-level disagreement, baseline-anchored deltas, disagreement localization), but less than the proposal claims (no statistical significance testing, no automated regression detection, no alerting).

---

## Resolutions

### Finding 1: calibratedFinalPassPct as "Differential Evaluation Framework"

**Finding**: The proposal (Section 2, Layer 5) claims "Metaharness implements a sophisticated differential evaluation framework through its experiment system, with Rasch-style item-response modeling suitable for formal differential evaluation." The rebuttal (citing `experiments.ts:129-189`) countered that `calibratedFinalPassPct` is a single-number projection utility — no comparison output, no disagreement localization.

**Prior Verdict**: ? UNVERIFIABLE

**New Verdict**: ✓ CONFIRMED (proposal feasible/supported) — with qualification

**Deeper Evidence** (NEW, beyond rebuttal):

1. **`headToHead()` function** — `/tmp/oh-my-pi/packages/metaharness/src/web/app.tsx:411-433`:
   ```typescript
   function headToHead(focusCells, arm, cells, tasks): HeadToHead {
     let focusWins = 0; let armWins = 0; let bothPass = 0; let bothFail = 0;
     for (const task of tasks) { /* pairwise pass/fail comparison */ }
     return { arm, focusWins, armWins, bothPass, bothFail, shared };
   }
   ```
   This IS pairwise differential comparison with task-level disagreement localization.

2. **`pickReferenceArm()`** — `/tmp/oh-my-pi/packages/metaharness/src/web/app.tsx:753-766`: Selects the highest-pass-rate baseline arm as comparison anchor. Uses `a.run.role !== "baseline"` — so `RunRole` IS consumed for comparison, not just display.

3. **`Delta()` component** — `/tmp/oh-my-pi/packages/metaharness/src/web/app.tsx:773-797`: Computes signed percentage-point differences (pass rate) and relative differences (cost, time) between each arm and the reference baseline.

4. **Task disagreement filter ("split")** — `/tmp/oh-my-pi/packages/metaharness/src/web/app.tsx:1356-1358`:
   ```typescript
   const splitCount = tasks.filter(t => {
     const s = taskStats.get(t);
     return s !== undefined && s.passes > 0 && s.passes < s.decided;
   }).length;
   ```
   Explicitly filters to tasks WHERE ARMS DISAGREE — this IS disagreement localization.

5. **`ExperimentDetail.matrix`** — `/tmp/oh-my-pi/packages/metaharness/src/experiments.ts:48-56`: The server returns a full `Record<armLabel, Record<taskId, {status, reward}>>` matrix — the data substrate for any differential analysis, computed backend and served via `GET /api/experiments/:id` (server.ts:295-297).

6. **FocusPanel** — `/tmp/oh-my-pi/packages/metaharness/src/web/app.tsx:1186-1332`: Full head-to-head drilldown showing net wins/losses, "winnable misses" (tasks the focused arm failed but another passed), and "unique wins" (tasks only this arm passed).

**Reasoning**: The rebuttal correctly identified that `calibratedFinalPassPct` is purely a projection utility. However, the rebuttal's conclusion that metaharness is "projection-only" is incomplete. The metaharness experiment system provides:
- Backend: task×arm outcome matrix (per-task pass/fail per arm), role-based arm tagging, arm comparability via shared task samples (`resolveArmLaunch` at server.ts:119 inherits the exact task sample)
- Dashboard: headToHead pairwise comparison, delta-from-baseline display, split/disagreement filters, winnable-miss identification

The proposal MISATTRIBUTES the differential capability to `calibratedFinalPassPct` (which is projection-only), but the SYSTEM as a whole does provide differential comparison with disagreement localization. The claim is directionally correct about the system's capabilities, wrong about which function implements them.

**Gate Outcome**: ADVANCE to design — the minimal true capability is: metaharness provides a task×arm outcome matrix, pairwise head-to-head counting (wins/losses/ties per task), disagreement localization (split filter), and baseline-anchored delta display. Any differential evaluation design can build on these existing primitives. The calibration function is NOT the differential part.

---

### Finding 2: RunRole for Comparison / Regression Detection

**Finding**: The proposal (Section 4.5) claims "The metaharness `store.ts` already has `RunRole = 'baseline' | 'variant'` — the missing piece is automated statistical comparison and alerting." The rebuttal (citing `experiments.ts:372`, `store.ts:35`) countered that `RunRole` is used only for display sort order and that "what's missing is the ENTIRE regression detection system, not just one piece."

**Prior Verdict**: ? UNVERIFIABLE

**New Verdict**: ✓ CONFIRMED (proposal feasible/supported) — with qualification on scope

**Deeper Evidence** (NEW, beyond rebuttal):

1. **`RunRole` consumed for comparison anchor selection** — `/tmp/oh-my-pi/packages/metaharness/src/web/app.tsx:755-756`:
   ```typescript
   for (const a of arms) {
     if (a.run.role !== "baseline" || a.passPct === null) continue;
   ```
   The role IS used to identify which arm serves as the comparison reference — NOT merely for sort order as the rebuttal claimed.

2. **Delta calculations use role-based anchor** — `/tmp/oh-my-pi/packages/metaharness/src/web/app.tsx:1094`:
   ```typescript
   {anchor && !isAnchor && <Delta value={arm.passPct} reference={anchor.passPct} mode="points" higherBetter />}
   ```
   Every variant arm's metrics are displayed as signed deltas from the baseline anchor.

3. **Arm comparability enforcement** — `/tmp/oh-my-pi/packages/metaharness/src/server.ts:112-118`:
   ```
   Inherits the experiment's benchmark, dataset, and — crucially — the exact
   task sample from a sibling arm (its recorded `include`, else its observed
   trial tasks) so the arm is directly comparable.
   ```
   The system enforces identical task samples across arms, making comparison statistically valid.

4. **NO statistical significance code anywhere** — Grep for `wilson|binomial|chi|fisher|mcnemar|sign.?test|bootstrap|permutation|confidence|interval|z.?score` across all metaharness files returns ZERO matches. No statistical testing infrastructure exists.

5. **NO automated alerting** — Grep for `alert|threshold|notify|webhook|slack|email` returns only a browser `alert()` call in the UI for launch failures (app.tsx:1704). No backend alerting system.

6. **NO backend comparison logic** — The server's experiment endpoint (server.ts:295) returns raw `experimentDetail` (arms + matrix). All comparison computation (headToHead, Delta, pickReferenceArm) lives in the CLIENT-SIDE React code only. No server-side comparison API endpoint exists.

7. **Existing primitives that support building regression detection**:
   - `RunRole` identifies baseline vs variant (store.ts:22, consumed by pickReferenceArm)
   - Task matrix stores per-task outcomes per arm (experiments.ts:55)
   - `resolveArmLaunch` enforces identical samples (server.ts:112-118)
   - `headToHead` counts wins/losses per task-pair (app.tsx:411-433)
   - SSE event stream exists for real-time updates (server.ts:72-80, SseClient)

**Reasoning**: The rebuttal's claim that `RunRole` is "used purely for display ordering" is factually incorrect — it's consumed by `pickReferenceArm()` to select the comparison anchor for delta calculations. However, the rebuttal's broader point stands: there is NO automated regression detection. All comparison is:
- Manual (user clicks "focus" in the UI)
- Visual (delta numbers, color-coded)
- Client-side only (React code, not server logic)
- Count-based (raw wins/losses, not statistically tested)

The proposal's framing that "the missing piece is automated statistical comparison and alerting" is directionally correct but understates the gap. What's missing:
1. Statistical significance testing (Wilson intervals, McNemar's test, etc.)
2. Server-side comparison logic (currently all in the React UI)
3. Automated alerting/webhook infrastructure
4. CI/CD gate integration

What EXISTS and the proposal can build on:
1. Role-based baseline identification
2. Task×arm outcome matrix with per-task granularity
3. Enforced arm comparability (shared task samples)
4. Head-to-head win/loss counting
5. SSE infrastructure for real-time event delivery (could carry alerts)

**Gate Outcome**: ADVANCE to design — the minimal true capability is: metaharness provides role-labeled arms, enforced task-sample comparability, a per-task outcome matrix, and client-side head-to-head comparison. Building automated regression detection requires adding: (a) server-side comparison logic (move headToHead to backend), (b) statistical significance testing, (c) alerting on the SSE channel. The primitives are sufficient; the automation layer is entirely absent.
