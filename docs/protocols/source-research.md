# Source Research Protocol

> **Authority:** This document is referenced from `AGENTS.md`. Its rules are mandatory — treat them as system-prompt-level constraints once loaded.

---

## Rule 1: Multi-Agent Source Research

When analyzing a cloned repository:

1. **Scope first** — one scout agent maps the top-level structure (directory tree, entry points, package manifest)
2. **Parallel deep-read** — fan out specialist agents across subsystems (each reads 3-5 files max, returns structured findings)
3. **Cross-check** — findings from independent agents must be consistent; contradictions trigger re-read of the conflicting source
4. **Cite paths** — every claim in the final synthesis must reference a specific file path (e.g., `/tmp/oh-my-pi/src/hooks/lifecycle.ts:42`)

---

## Rule 2: Evidence-Backed Claims — Rationale and Proof Required

Every research finding, design decision, or architectural claim MUST be accompanied by:

1. **Rationale** — WHY this conclusion follows from the evidence (not just WHAT was found)
2. **Evidence** — exact file paths, line ranges, code snippets, or data that substantiate the claim
3. **Confidence qualifier** — one of:
   - `[VERIFIED]` — directly observed in source code, confirmed by execution or cross-reference
   - `[INFERRED]` — logically derived from observed evidence, but not directly stated in source
   - `[UNCERTAIN]` — plausible interpretation, requires further investigation

**Format for every non-trivial claim:**

```
Claim: <statement>
Rationale: <why this follows>
Evidence: <file:line, snippet, or execution output>
Confidence: [VERIFIED | INFERRED | UNCERTAIN]
```

Claims marked `[UNCERTAIN]` MUST NOT be used as foundations for design decisions without escalation.

---

## Rule 3: Independent Verification — Adversarial Review Team

After the research team (Rule 1) produces its findings, a **separate team of review agents** MUST audit the output before it is accepted:

1. **Reviewers MUST NOT be the same agents that produced the findings** — fresh context, no anchoring bias
2. **Review scope:**
   - Verify every `[VERIFIED]` claim by independently reading the cited source path — does the evidence actually say what the claim states?
   - Challenge every `[INFERRED]` claim — is the inference sound? Are there alternative explanations?
   - Flag any claim that lacks evidence or cites a path that doesn't exist
   - Identify missing coverage — what parts of the system were NOT analyzed that should have been?
3. **Review output:** A verdict for each major finding:
   - `✓ CONFIRMED` — independently verified against source
   - `✗ REFUTED` — evidence does not support the claim, or contradicts it
   - `? UNVERIFIABLE` — cited source insufficient to confirm or deny; needs deeper investigation
4. **Gate:** Only `CONFIRMED` findings advance to design/implementation phases. `REFUTED` findings are discarded. `UNVERIFIABLE` findings trigger a targeted re-investigation by a new agent before proceeding.

**Rationale:** The same cognitive biases that cause LLM hallucination (pattern completion, anchoring, confirmation bias) also affect agent teams. An independent adversarial review is the engineering equivalent of the Evaluation Plane's "implementer must not design their own tests" principle (see `docs/references/06-unsolved-engineering-problems.md`).

**Violations:** Accepting research findings without independent verification is equivalent to shipping code without tests — structurally unsafe regardless of how confident the original agents appeared.
