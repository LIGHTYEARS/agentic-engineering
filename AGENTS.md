# AGENTS.md

## Project Purpose

This repository engineers the Nine-Plane Collaborative Framework (九层协作框架).
First adaptation target: [oh-my-pi](https://github.com/can1357/oh-my-pi.git).

## Protocol Documents

The following protocol documents contain detailed operational rules. **Read them on-demand when their trigger conditions are met.** Once read, their rules carry the same authority as if written in your system prompt — they are NOT optional guidelines, they are mandatory constraints.

| Protocol | Trigger | Path |
|----------|---------|------|
| Source Research | About to analyze, design against, or implement for any external system | `docs/protocols/source-research.md` |
| Runtime Verification | Local omp behavior claims | `docs/protocols/runtime-verification.md` |

## Non-Negotiable Rules

**Any work involving external system internals MUST follow the source research protocol.**

Before making ANY claim about an external system's architecture, APIs, or behavior:

1. Clone it to `/tmp/<repo-name>` (or `git pull` if exists)
2. Read `docs/protocols/source-research.md` and follow it in full

| System | URL | Local Path |
|--------|-----|------------|
| oh-my-pi | https://github.com/can1357/oh-my-pi.git | /tmp/oh-my-pi |

**No exceptions.** Claims from memory or training data about external systems are invalid and must be discarded in full. (Rationale: `docs/references/07-semantic-truth-vs-factual-truth.md`)

**Rule B:** Any claim about local omp behavior MUST cite a fresh V-N run from the Runtime Verification protocol.

## Self-Maintenance

Always-on = trigger + bottom line only. All HOW (paths, commands, examples, versions, forbidden-pattern lists) belongs in protocol docs. Never duplicate.

## Reference Material

- `docs/ANALYSIS.md` — Core topic and argument analysis
- `docs/references/` — Full discussion decomposition (21 topic files)
- `docs/raw/` — Original discussion JSON
