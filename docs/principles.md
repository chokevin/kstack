# kstack — Design Principles

This document captures *why* kstack is shaped the way it is. Read it once. The README says what kstack is; this says what kstack deliberately *isn't*, and why.

## Where this comes from

Two pieces of evidence shape the design:

1. **["How Well Do Agentic Skills Work in the Wild"](https://arxiv.org/html/2604.04323v1)** (Liu et al., 2026, UCSB + MIT). First rigorous benchmark of skill utility under realistic conditions. Headline finding: *skill benefits are fragile* — performance degrades from idealized (curated skills handed directly) to realistic (retrieval from large noisy collection), eventually approaching no-skill baseline. Best realistic gain: ~14% (Terminal-Bench 2.0, 57.7% → 65.5%).
2. **Dogfooding.** kstack started small on purpose. The HARDEN reckon memo (`docs/reckons/kstack-itself-20260418.md`) committed to staying small until v0 skills earn it.

## What the paper says, and what kstack does about it

The paper isolates three real-world challenges that prior skill benchmarks ignored:

| Challenge | What it means | kstack's stance |
|---|---|---|
| **Skill selection** | Agent doesn't know which loaded skill to use | **Sidestepped.** All skills invoked explicitly via `/kstack-<name>`. The user is the selector. |
| **Skill retrieval** | Agent must find relevant skills in a noisy collection | **Sidestepped.** 7 hand-written skills, all installed directly. No search, no embeddings, no agentic hybrid retrieval. |
| **Skill adaptation** | Retrieved skill only partially fits the task | **Addressed in every skill.** See `## Step 0: Adapt to this ask` in every SKILL.md. The paper's strongest recovery technique (query-specific refinement) is mandatory, not optional. |

The paper also names two bottlenecks:

- **Agents leave helpful skills unloaded.** Not a kstack problem — explicit invocation forces the load.
- **Retrieved content is too noisy or imprecise to help.** **This IS a kstack problem.** Placeholder commands (e.g. `/airun-triage` v0) are exactly this failure mode. The fix is content quality, not infrastructure: replace placeholders with real commands as soon as the skill is used.

## What kstack will not build

Each of these solves a problem kstack does not have. Building them would inflate surface area without proportionate benefit.

- **Agentic skill retrieval** (BM25, dense embeddings, hybrid search). Solves "find the right skill in 34k". kstack has 7.
- **"Pushy" skill descriptions** to fight under-triggering. Solves "agent doesn't auto-invoke." Slash commands eliminate this.
- **Eval harnesses with parallel subagent runs** (à la Anthropic's skill-creator). Worth understanding; not worth porting at this scale. Manual `/kstack-retro` after real invocations is enough until it isn't.
- **Skill marketplaces / discovery UIs.** kstack is a personal library of one. If a skill leaves my workflow, delete it.
- **Auto-generated skills from execution traces.** Premature. Skills should be earned by observed pain, not generated speculatively.

## What kstack does and why it's right

- **Small, hand-written, opinionated.** The paper's idealized setting (curated skills handed directly) is exactly where skills *do* help. kstack stays in that regime by being intentionally small. Don't scale.
- **Explicit invocation.** The user picks the skill. No selection bottleneck.
- **Composable artifacts.** `/plan` writes `plan.md`, `/review` reads it, `/ship` verifies it. Composition is the cheap multiplier; new skills are the expensive one.
- **Domain skills clearly demarcated.** Generic engineering primitives in `skills/{plan,review,investigate,ship,retro,reckon}`; domain skills (`airun-triage`) noted as such in README and AGENTS.md. Don't conflate.
- **Mandatory adaptation step.** Every SKILL.md begins with `## Step 0: Adapt to this ask`, forcing the agent to scope the skill to the actual prompt before executing. This is the paper's biggest single intervention.

## Bounded expectations

The paper's best realistic result is ~14% relative improvement on Terminal-Bench 2.0 with retrieval + refinement on top of Claude Opus 4.6. **That's the realistic ceiling on skill engineering.** Skills are useful, not transformative.

If a kstack skill claims (in retro or in your head) to deliver 2x or 10x productivity, you're miscounting. Worth ~10-15% on the tasks it fits, near 0% otherwise. Plan accordingly.

## When to revisit this doc

- Whenever a `/kstack-retro` produces a new principle or contradicts an existing one.
- Whenever a new piece of empirical evidence (paper, post-mortem, dogfood result) lands.
- Whenever you catch yourself wanting to add infrastructure that the paper says doesn't matter at this scale.

## Citation

Liu, Y., Ji, J., An, L., Jaakkola, T., Zhang, Y., Chang, S. (2026). *How Well Do Agentic Skills Work in the Wild: Benchmarking LLM Skill Usage in Realistic Settings.* arXiv:2604.04323. Code: https://github.com/UCSB-NLP-Chang/Skill-Usage
