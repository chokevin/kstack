# kstack — Agent Instructions

When this repo is loaded as a skill library (installed at `~/.kstack/` or `~/.copilot/skills/kstack/`), the agent has access to the following slash commands. Each is a self-contained Markdown skill with its own persona and workflow.

## Available skills

- **`/plan <ask>`** — Tech Lead. Scope, non-goals, SQL todo list. See `skills/plan/SKILL.md`.
- **`/research <question>`** — Researcher. Deep multi-source research (arxiv, GitHub, Reddit, X, HN, postmortems, talks). Steelmans the opposite. Ends with ADOPT / ADAPT / SKIP / FURTHER-RESEARCH. See `skills/research/SKILL.md`.
- **`/reckon <service>/<slice>`** — Service Re-evaluator. Honest re-evaluation of a long-lived service. Produces a one-page memo with a binding decision (HARDEN / EXTEND / REFACTOR / DEPRECATE / FREEZE). See `skills/reckon/SKILL.md`.
- **`/review [ref]`** — Staff Engineer. Rigorous review of changes. See `skills/review/SKILL.md`.
- **`/investigate <symptom>`** — Debugger. Root-cause methodology. See `skills/investigate/SKILL.md`.
- **`/ship`** — Release Engineer. Lint → test → commit → push → PR. See `skills/ship/SKILL.md`.
- **`/retro`** — Coach. Session retrospective + kstack improvement log. See `skills/retro/SKILL.md`.

### Domain skills (only useful in specific contexts)

- **`/airun-triage <symptom>`** — Distributed Cluster Triage Engineer. Layered triage (Ray → Kueue → k8s scheduler → MIG → node-pool/region) for stuck/failed/slow jobs on the airun multi-region AKS cluster. See `skills/airun-triage/SKILL.md`.

## Rules when using kstack skills

1. **Read the skill's `SKILL.md` before acting.** Don't improvise a persona — the skill has one.
2. **Skills compose.** `/plan` writes to the session plan. `/review` reads it. `/ship` verifies against it.
3. **Never skip a skill's verification step.** If `/ship` says run tests, run tests — don't claim they pass because they "look fine."
4. **If a skill doesn't exist for the task, don't fake it.** Do the work directly and add a `/retro` note about the missing skill.
