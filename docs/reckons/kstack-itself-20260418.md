# Reckon: kstack / itself

**Date:** 2026-04-18
**Trigger:** Skill of the skill — testing whether `/reckon` itself produces value.
**Decision:** HARDEN

## What this is

kstack is a 13-file repo (6 Markdown skills, 1 bash installer, README/AGENTS/CONTRIBUTING/LICENSE/VERSION/.gitignore), born ~5 hours before this memo. Modeled on Garry Tan's gstack but stripped to engineering primitives for Copilot CLI. Skills are symlinked into `~/.copilot/skills/kstack-*` and surfaced via a global instructions block. Currently 1 user (you), 1 host (Copilot CLI), 0 PRs, 0 issues. A second consumer ("outworked" repo) is planned but not yet integrated. There is no operational cost yet — nothing has been operated.

## What I read

- `README.md`, `AGENTS.md`, `CONTRIBUTING.md`, `LICENSE`, `VERSION`, `.gitignore`
- `bin/setup`
- `skills/{plan,review,investigate,ship,retro,reckon}/SKILL.md`
- `git log` (2 commits, both today)
- `gh issue list` and `gh pr list` (both empty)

13/30 files read. Anchor read complete; no budget pressure.

## Five-axis scores

| Axis | Score | Evidence |
|---|---|---|
| **Contract surface** | 1/3 | Skill format is implicit Markdown+YAML, no schema, no validator. Install contract is "symlink + global-instructions append," undocumented for second-host adapters. Will break the moment "outworked" consumes it without a defined surface. |
| **Operational load** | 0/3 | No pages, no users, no maintenance. Genuinely zero — and at this stage that's not a problem, it's the starting state. |
| **Code health** | 1/3 | The 6 skills are clean and consistent. But: no tests, no CI, no skill-format linter, no CHANGELOG. First malformed frontmatter ships silently and surfaces as runtime confusion. |
| **Strategic fit** | 2/3 | The 5 starter primitives (plan/review/investigate/ship/retro) are agent-agnostic engineering staples — they should transfer cleanly to your infra/multi-repo world. `/reckon` was built on theory before validation and is the weakest fit. Outworked as a second consumer is a strong positive signal *if* the substrate is hardened first. |
| **Cost of change** | 3/3 | 13 files, no tests to maintain, symlink install means edits are instant, no external dependencies. You can rewrite any skill in 5 minutes. Best possible. |

## The decisive axis

**Strategic fit**, by way of a specific anti-pattern: kstack is currently *accreting on theory*. `/reckon` was built before any of the v0.1 skills earned their keep on a real task. The shape of the next 30 days will determine whether kstack becomes a load-bearing tool you actually use or a wishful-thinking artifact that gets cargo-cult features from gstack until you abandon it.

## Decision: HARDEN

The shape is right (engineering primitives + Markdown + symlink install + agent-agnostic format). The reliability is unproven (zero real-task evidence). HARDEN means: stop adding skills, dogfood the existing five on real work, and put exactly enough scaffolding around the substrate to survive a second consumer ("outworked") without contract breakage.

## What this means in practice

- **Next 30 days:**
  1. Use each of the 5 starter skills on ≥2 real tasks (10 invocations total) across `aks-rp`, `agentbaker`, or `baker`.
  2. Log every invocation in a new `CHANGELOG.md` — what skill, what task, what worked, what hurt.
  3. Run `/kstack-retro` after each substantive use.
  4. Add a 20-line CI step that lints SKILL.md frontmatter (required keys: `name`, `role`, `trigger`, `summary`).

- **Next quarter:**
  - kstack survives outworked integration without breaking the SKILL.md contract.
  - At least 2 skills amended based on real `/retro` findings (proves the feedback loop works).
  - At least one new skill added — *driven by retro evidence, not theory*.

- **Explicit non-goals:**
  - Do **not** add new skills until the v0 five have ≥10 combined real invocations logged.
  - Do **not** port gstack's telemetry, preamble system, upgrade flow, or config CLI. Wrong tier of investment for one user.
  - Do **not** treat `/reckon` as load-bearing — it was built on theory; let it earn its keep or get pulled.
  - Do **not** add browser/QA tooling. Not your shape.

## Risks / counter-arguments

The strongest case against HARDEN is that the 5 skills might be the wrong primitives entirely, and dogfooding them will produce friction without insight. Counter: that *is* the insight. After 10 real invocations, `/retro` produces evidence about which skills earn keep. If 3 of 5 turn out useless, that's a successful HARDEN outcome — it tells you which 2 to double down on.

What would change the call: if outworked needs a fundamentally different skill format (e.g., executable skills, not Markdown prompts), the right call shifts to **REFACTOR** — re-shape the substrate around the contract outworked actually needs.

## Open questions for the team (you)

1. What is "outworked," and what does it actually need from kstack — the SKILL.md prose, the `/kstack-*` command names, or just the format conventions?
2. Should `/reckon` be removed in v0.2 since it's the most theory-heavy skill and least likely to be invoked on real work? Or kept as a probe?
3. What's the *first* real task this week where you'll invoke a kstack skill? Naming it makes the HARDEN plan concrete; leaving it abstract means HARDEN dies.
