# Contributing to kstack

kstack is primarily my personal stack — but if a skill or improvement would help others, PRs welcome.

## Principles

1. **One skill, one job.** If a skill does two unrelated things, it's two skills.
2. **Iron Laws first.** Every skill starts with 3-5 non-negotiable rules. If you can't name them, the skill isn't scoped yet.
3. **Anti-patterns matter.** Tell the agent what *not* to do. That's usually where the wins are.
4. **Agent-agnostic Markdown.** Skills are plain `.md` with YAML frontmatter. No host-specific escape hatches unless the skill genuinely requires them — and then gate them behind a "if host supports X" block.

## Adding a skill

1. Create `skills/<name>/SKILL.md` with frontmatter:
   ```yaml
   ---
   name: <name>
   role: <one-word persona>
   trigger: /<name>
   summary: <one sentence>
   ---
   ```
2. Body sections (use these headings so skills feel consistent):
   - `## Iron Laws` — 3-5 non-negotiables
   - `## Workflow` — numbered steps
   - `## Anti-patterns` — what not to do
   - `## Exit criteria` — how you know you're done
3. Add the skill to `AGENTS.md` and `README.md`.
4. Run `./bin/setup --dry-run` to sanity-check the install.

## Amending a skill

If real-world use surfaced a new failure mode, codify it in **Anti-patterns** or add an **Iron Law**. Don't let the skill drift into prose.

## Style

- Short sentences. Imperative voice. The reader is an AI agent, not a novel reader.
- No emoji in skill bodies (they inflate tokens).
- No "consider"-ing. Tell the agent what to do.
