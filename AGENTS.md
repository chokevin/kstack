# kstack — Agent Instructions

When this repo is loaded as a skill library (installed at `~/.kstack/` or `~/.copilot/skills/kstack/`), the agent has access to the following slash commands. Each is a self-contained Markdown skill with its own persona and workflow.

## Available skills

- **`/plan <ask>`** — Tech Lead. Scope, non-goals, SQL todo list. See `skills/plan/SKILL.md`.
- **`/research <question>`** — Researcher. Deep multi-source research (arxiv, GitHub, Reddit, X, HN, postmortems, talks). Steelmans the opposite. Ends with ADOPT / ADAPT / SKIP / FURTHER-RESEARCH. See `skills/research/SKILL.md`.
- **`/reckon <service>/<slice>`** — Service Re-evaluator. Honest re-evaluation of a long-lived service. Produces a one-page memo with a binding decision (HARDEN / EXTEND / REFACTOR / DEPRECATE / FREEZE). See `skills/reckon/SKILL.md`.
- **`/review [ref]`** — Staff Engineer. Rigorous review of changes. See `skills/review/SKILL.md`.
- **`/investigate <symptom>`** — Debugger. Root-cause methodology. See `skills/investigate/SKILL.md`.
- **`/explore <question>`** — Subagent Foreman. Fan out parallel read-only subagents to map / spec / compare without burning main context. Modeled on `huggingface/ml-intern`'s research-subagent pattern. See `skills/explore/SKILL.md`.
- **`/unstuck`** — Loop-breaker. Stops a doom loop (same fix, same error, third iteration), names it, forces a layer change. Modeled on `huggingface/ml-intern`'s `doom_loop.py` detector. See `skills/unstuck/SKILL.md`.
- **`/ship`** — Release Engineer. Lint → test → commit → push → PR. See `skills/ship/SKILL.md`.
- **`/retro`** — Coach. Session retrospective + kstack improvement log. See `skills/retro/SKILL.md`.

### Domain skills (only useful in specific contexts)

- **`/airun-triage <symptom>`** — Distributed Cluster Triage Engineer. Layered triage (Ray → Kueue → k8s scheduler → MIG → node-pool/region) for stuck/failed/slow jobs on the airun multi-region AKS cluster. See `skills/airun-triage/SKILL.md`.
- **`/kernel-recall <question>`** — Kernel Optimization Archivist. Searches the local `kevin-obsidian` vault for prior GPU/ML-kernel optimization notes, benchmark results, profiling lessons, and experiment history. See `skills/kernel-recall/SKILL.md`.
- **`/cpu-kernel-recall <question>`** — OS/Linux Kernel Module Archivist. Searches local memory and source references for CPU/host kernel modules, Linux driver configs, Secure Boot/AppArmor, OFED/MOFED, and driver-runtime gotchas that can confound GPU benchmarks. See `skills/cpu-kernel-recall/SKILL.md`.
- **`/profiler-handoff <profile/artifacts>`** — Profiler Artifact Release Engineer. Packages PyTorch/Perfetto/Nsight/HTA profiler results and torchview/Netron/Graphviz model graphs into Hermes-ready `artifact_handoff` events. See `skills/profiler-handoff/SKILL.md`.
- **`/obsidian-improve [scope]`** — Obsidian Context Gardener. Edits the local `kevin-obsidian` vault to improve fast context loading: indexes, context cards, promoted learnings, and validation. See `skills/obsidian-improve/SKILL.md`.
- **`/obsidian-crystallize [scope]`** — Obsidian Knowledge Crystallizer. Mines raw vault material and routes durable knowledge to the right home: learnings, context cards, context maps, research, reckonings, or retros. See `skills/obsidian-crystallize/SKILL.md`.
- **`/upstream-friction <boundary>`** — Integration Boundary Architect. Finds downstream hacks that should be upstream contract fixes, forces an owner decision, and prevents workaround sprawl. See `skills/upstream-friction/SKILL.md`.

## Rules when using kstack skills

1. **Read the skill's `SKILL.md` before acting.** Don't improvise a persona — the skill has one.
2. **Skills compose.** `/plan` writes to the session plan. `/review` reads it. `/ship` verifies against it.
3. **Never skip a skill's verification step.** If `/ship` says run tests, run tests — don't claim they pass because they "look fine."
4. **If a skill doesn't exist for the task, don't fake it.** Do the work directly and add a `/retro` note about the missing skill.

## Proactive skill triggers

Use these even when the user does not spell out the slash command:

| Signal | Skill |
|--------|-------|
| User reports something wrong/broken/flaky/regressed/confusing | `/investigate` first; `/airun-triage` for airun scheduling failures. |
| Third repeated attempt with same error/fix loop | `/unstuck`. |
| Hard-won fix, surprising benchmark, costly gotcha, non-obvious decision, or solved >15 minute problem | `/obsidian-crystallize` before ending. |
| Downstream repo is adding a workaround for an upstream platform/API/tooling gap | `/upstream-friction`. |
| Vault/project context feels stale, hard to navigate, or too log-heavy | `/obsidian-improve`. |
| GPU/ML compute-kernel or performance work starts | `/kernel-recall` before fresh work. |
| Profiler trace/export/result or model graph should be sent to Hermes | `/profiler-handoff` before emitting the artifact handoff. |
| OS/Linux kernel module, CPU-side driver, kernel config, Secure Boot/AppArmor, OFED/MOFED, or `open-gpu-kernel-modules` work starts | `/cpu-kernel-recall` before fresh work. |
| Agent/skill workflow friction appears | `/retro`. |

Do not turn every session into a vault update. `proj log` routine progress; crystallize only reusable knowledge.
