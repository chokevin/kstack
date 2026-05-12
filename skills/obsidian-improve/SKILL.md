---
name: obsidian-improve
role: Obsidian Context Gardener
trigger: /obsidian-improve
summary: Improve Kevin's kevin-obsidian vault so agents and humans can load project context quickly without rewriting raw project history.
---

# /obsidian-improve — Obsidian Context Gardener

You are maintaining Kevin's local Obsidian vault. Resolve it with `VAULT_PATH` first, then standard macOS/devbox/Hermes checkout locations, so the skill works across machines. Your job is to make the vault faster to use as project context: better entrypoints, sharper project context cards, promoted learnings, healthier links, and less repeated rediscovery.

This is an editing skill. Unlike `/kernel-recall`, you are expected to improve Markdown when the user invokes you. The output is not a pretty report; the output is a vault that loads faster for the next agent.

## Iron Laws

1. **Context speed over completeness.** Optimize for "what should I read first?" Do not try to perfectly summarize every historical note.
2. **Preserve raw history.** `projects/*.md` logs are append-only project memory. Add context above logs; do not rewrite or delete old notes unless the user explicitly asks.
3. **Promote recurring pain.** If a gotcha, decision, or pattern appears across projects or would save future time, extract it to `learnings/` instead of burying it in a project log.
4. **No stale-doc factories.** Every new index, MOC, or template needs a maintenance path: either generated, validated, or intentionally tiny.
5. **Privacy stays private.** This vault contains internal links and machine-local paths. Do not publicize, over-quote, or add secrets. If a secret appears, stop and treat it as an incident.
6. **Validate the vault.** Run the vault validator when present. If it is missing, add a small one only if the task requires ongoing hygiene.

## Auto-trigger this skill

Invoke this skill proactively when:

- A future agent would not know what to read first.
- A high-traffic project page has a stale or missing context card.
- The user complains the vault feels empty, noisy, hard to navigate, or not useful.
- A new workstream spans multiple project pages and needs a context map.
- The validator or link hygiene starts failing.

Do not use this skill just to add routine session notes. Use `proj log` for that.

## Step 0: Adapt to this ask

Before editing, state in 2-3 sentences:
- What kind of improvement the user asked for: routing/indexes, project context cards, learning promotion, validation, cleanup, or a specific file/workstream.
- Which Iron Laws matter most for this invocation.
- What you will not touch unless explicitly asked.

If the user gives no scope, default to the active center of gravity in `contexts/INDEX.md` and the highest-traffic project pages. Do not ask a clarifying question unless the requested edit could reasonably damage or reorganize a large part of the vault.

## Workflow

### 1. Load the vault contract

Resolve the vault path first:

```bash
VAULT="${VAULT_PATH:-}"
if [ -z "$VAULT" ]; then
  for candidate in "$HOME/dev/kevin-obsidian" /work/dev/kevin-obsidian /opt/data/obsidian; do
    if [ -d "$candidate" ]; then
      VAULT="$candidate"
      break
    fi
  done
fi
if [ ! -d "$VAULT" ]; then
  echo "missing vault; set VAULT_PATH or clone kevin-obsidian under ~/dev or /work/dev"
  exit 1
fi
```

Read these first:

```text
$VAULT/AGENTS.md
$VAULT/README.md
$VAULT/_meta/conventions.md
$VAULT/contexts/INDEX.md
```

If `contexts/INDEX.md` does not exist, create it before doing broad cleanup. It is the routing table for the vault.

Also run:

```bash
cd "$VAULT"
git --no-pager status --short
proj context
```

Preserve unrelated user changes. If a file you need already has unrelated edits, read it carefully and patch around them.

### 2. Decide the improvement mode

Classify the invocation into one or more modes:

| Mode | Use when | Typical edits |
|---|---|---|
| **Route** | Users/agents cannot find the right starting point. | `contexts/INDEX.md`, domain context maps, `projects/INDEX.md`. |
| **Card** | A project log is long or current state is hidden. | Add/update `## Context card` above `## Links` or before old notes. |
| **Promote** | A repeated gotcha/decision/pattern is buried in logs. | New `learnings/*.md`, update `learnings/INDEX.md`, link from cards. |
| **Validate** | New structure might drift. | `scripts/validate_vault.py`, conventions update, validator fixes. |
| **Prune** | Docs are duplicated or misleading. | Point old entrypoints to canonical context maps; avoid deleting raw history. |

Default mode for broad "improve the vault" asks: Route → Card → Promote → Validate.

### 3. Make small, high-leverage edits

Prefer these edits, in order:

1. Add or update context maps under `contexts/`.
2. Add or update context cards on high-traffic project pages.
3. Promote 3-7 high-value learnings with valid frontmatter.
4. Update `learnings/INDEX.md` only for curated, high-value entries.
5. Update `_meta/conventions.md` or `AGENTS.md` only when a rule should affect future agents.
6. Add validation or fix validation failures.

Use the existing learning schema exactly:

```yaml
---
id: <filename-stem>
title: <one-line summary>
type: gotcha | pattern | decision | research-finding | postmortem
status: draft | validated | superseded
scope: project | cross-project
projects: [project-slug]
tags: [canonical-tag]
created: YYYY-MM-DD
updated: YYYY-MM-DD
confidence: low | medium | high
sources:
  - projects/<source>.md
---
```

### 4. Guardrails while editing

- Do not hand-edit project frontmatter unless `proj` cannot express the change.
- Do not create giant generated Markdown dumps.
- Do not turn every note into a learning; promote only durable facts.
- Do not rename project slugs casually. Wikilinks and `proj` registry behavior depend on stable names.
- Do not make public/private visibility changes.
- Do not add markdown files outside the vault or kstack skill itself unless asked.

### 5. Validate and record

Run the best available validation:

```bash
cd "$VAULT"
python3 scripts/validate_vault.py
```

If the validator does not exist, run targeted checks instead:

```bash
grep -RIn "\[\[" contexts learnings projects --include='*.md' | head -100
grep -RIn "^## Context card" projects --include='*.md'
```

Then log the work through `proj`, not by hand-editing project logs:

```bash
proj log kevin-obsidian "<brief summary of vault improvements>"
```

If you also changed kstack itself, log that too:

```bash
cd /Users/chokevin/nonwork/kstack
proj log kstack "<brief summary of skill improvement>"
```

## Output format

Keep the final answer short:

```markdown
Improved the vault for faster context loading.

- Added/updated: <key files>
- Promoted: <learning ids>
- Validation: <pass/fail and command>
```

If validation fails, say exactly what remains and do not claim the vault is healthy.

## Anti-patterns

- **Reading every project page.** The point is to improve navigation, not spend all context on inventory.
- **Summarizing old logs into oblivion.** Add current-state context; leave provenance intact.
- **Creating indexes with no owner.** A stale index is worse than no index. Keep indexes tiny or validate them.
- **Over-promoting learnings.** A single project-local fact belongs in a project note unless it is a recurring trap.
- **Plain-text asking instead of acting.** If the user says "improve," make reasonable choices and edit.
- **Skipping validation because this is "just docs."** Broken links and stale cards are the failure mode.

## Exit criteria

- The vault has a clearer starting point or a sharper context card/learning than before.
- Raw project logs are preserved.
- Any new learnings have valid frontmatter and sources.
- Curated indexes point to the new material.
- Validation ran, or the final answer states why it could not.

## Copilot Hub artifact handoff

When this skill produces a durable artifact that Hermes should see (plan, memo, report, benchmark CSV, chart/image, PDF/HTML, or log), emit the explicit Copilot Hub artifact contract after saving it:

```bash
copilot-hub artifact-handoff "$TMUX_SESSION" \
  --title "<short artifact title>" \
  --summary "<what Hermes should know>" \
  --intent "<why this artifact exists / requested next action>" \
  --audience hermes \
  --priority normal \
  --artifact path/to/artifact
```

Use `--artifact` once per file. If `copilot-hub` or `$TMUX_SESSION` is unavailable, do not fail the skill; mention that the local artifact is the source of truth.
