# kstack

> Personal AI software factory. Inspired by [gstack](https://github.com/garrytan/gstack), built for [GitHub Copilot CLI](https://github.com/github/copilot-cli) first, agent-agnostic by design.

**kstack** is my opinionated set of slash-command skills for AI coding agents. It turns a blank prompt into a structured sprint: **Plan → Build → Review → Ship**. Each skill is a Markdown file that encodes a specialist persona — planner, reviewer, debugger, release engineer — and a repeatable workflow.

I'm [@chokevin](https://github.com/chokevin). I ship infrastructure and platform code for a living, and I got tired of re-typing the same "be rigorous, check for X, verify Y" prompts every time. kstack is the muscle memory, written down.

This is a work-in-progress. The point is to dogfood it daily and let it evolve.

## Why another one?

[gstack](https://github.com/garrytan/gstack) is phenomenal and I borrow its sprint model shamelessly. But gstack targets Claude Code as the primary host, and it's opinionated around a founder/product workflow. kstack is:

- **Copilot CLI first** — skills are drop-in under `~/.copilot/skills/` or invoked via custom instructions
- **Infra-aware** — skills assume you're often working on Terraform, Kubernetes, Helm, Azure, and multi-repo CLIs, not just web apps
- **Small surface area** — a few skills I actually use, not a library I might use someday
- **Portable format** — each skill is plain Markdown with YAML frontmatter, works in Copilot CLI, Claude Code, Cursor, etc. with thin host adapters

## Skills (v0)

| Skill | Role | What it does |
|-------|------|--------------|
| `/plan` | Tech Lead | Turns a vague ask into a scoped plan with a SQL todo list and explicit non-goals. |
| `/review` | Staff Engineer | Rigorous review of uncommitted / branch changes. High signal-to-noise, no style nitpicks. |
| `/investigate` | Debugger | Root-cause methodology. Iron Law: no fixes without a verified hypothesis. |
| `/ship` | Release Engineer | Lint, test, commit, push, open PR. Verifies green CI before declaring done. |
| `/retro` | Coach | End-of-session retrospective: what worked, what to add to kstack next. |

More will land as I find friction in my own workflow.

## Install

**Requirements:** [GitHub Copilot CLI](https://github.com/github/copilot-cli), `git`, `bash`.

```bash
git clone --depth 1 https://github.com/chokevin/kstack.git ~/.kstack
~/.kstack/bin/setup
```

The setup script:
1. Symlinks `~/.kstack/skills/*` into `~/.copilot/skills/` (or your agent's skills dir)
2. Appends a `# kstack` section to `~/.copilot/instructions.md` listing the available skills

Run `~/.kstack/bin/setup --help` for other agents (Claude Code, Cursor, Codex).

## Use

In any Copilot CLI session:

```
/plan add retry + backoff to the baker provisioner
/review
/ship
```

Or reference a skill inline: "follow the kstack /investigate methodology on this flaky test."

## Philosophy

1. **One skill = one persona = one workflow.** No swiss-army knives.
2. **Skills compose.** `/plan` writes artifacts `/review` and `/ship` can read.
3. **Agent-agnostic Markdown.** If a skill needs host-specific tooling, it degrades gracefully.
4. **Ship the narrowest wedge.** Every skill starts as the 20% that covers 80%. Iterate from real use.

## License

MIT. Fork it, remix it, make it your own stack.
