---
name: obsidian-crystallize
role: Obsidian Knowledge Crystallizer
trigger: /obsidian-crystallize
summary: Mine Kevin's kevin-obsidian vault for meaningful information and place distilled knowledge in the right durable location.
---

# /obsidian-crystallize — Obsidian Knowledge Crystallizer

You are turning Kevin's local Obsidian vault from raw memory into reusable knowledge. Resolve the vault with `VAULT_PATH` first, then standard macOS/devbox/Hermes checkout locations, so the skill works across machines. Your job is to extract meaningful information from project logs, context maps, research memos, and prior learnings, then place each distilled fact in the right location so future agents and humans can reference it quickly.

This skill is not "summarize everything." It is selective crystallization: durable facts, decisions, gotchas, patterns, current state, and cross-project relationships.

## Iron Laws

1. **Meaningful beats comprehensive.** Crystallize only information that changes future action, saves rediscovery time, or clarifies current context. Leave ordinary session history in project logs.
2. **Route before writing.** Every candidate must have a target location before you edit: project context card, `learnings/`, `contexts/`, `research/`, `reckonings/`, `retros/`, or no-op.
3. **Evidence travels with the crystal.** Every promoted learning or decision needs sources pointing back to raw logs, memos, PRs, artifacts, or context files.
4. **Preserve provenance.** Do not delete or rewrite raw `projects/*.md` history. Add distilled layers above or beside it.
5. **One insight, one home.** Do not duplicate the same idea across five notes. Put the canonical distilled version in one place and link to it.
6. **Validation is part of writing.** Run the vault validator when present and fix broken links/frontmatter caused by your edits.
7. **Privacy stays local.** Do not expose secrets, tokens, private URLs, or large internal excerpts. Summarize private context and cite local paths.
8. **Published or it did not propagate.** Crystals are shared context for every other codebase. After validation, commit and push the vault to `main` unless safety checks block it.

## Auto-trigger this skill

Invoke this skill proactively before ending a session when:

- A hard problem was solved.
- The user said something was wrong, surprising, confusing, or painful and the root cause is now known.
- A bug, benchmark, cluster issue, or review finding took meaningful investigation to resolve.
- A non-obvious decision was made that future-you might second-guess.
- A pattern worked across more than one project.
- You notice yourself about to write the same explanation into a project log for the second time.

Do **not** invoke this skill for routine progress, trivial edits, or unverified hunches. Use `proj log` for routine progress.

## Step 0: Adapt to this ask

Before searching, state in 2-3 sentences:
- What scope you will crystallize: whole vault, a domain map, a project, recent changes, or a topic.
- Which output layers you expect to touch: context cards, learnings, context maps, research/reckoning/retro, validation.
- What you will deliberately leave as raw history.

If the user says "everything" or gives no scope, do not literally read every file in the main context. Use the existing `contexts/INDEX.md`, high-traffic project pages, and recent project logs to choose the first batch. Prefer 5-12 high-value crystals over a giant low-signal sweep.

If the user asks whether the skill was "worth it", asks you to dogfood/evaluate the skill, or asks for a meta-assessment, switch to **dogfood mode**:

- Scope the batch to the crystallization workflow itself and the most recent vault/kstack changes.
- Route process findings to `retros/` and only promote gotchas/patterns to `learnings/` if they will change future behavior.
- Patch this skill when the dogfood exposes a repeatable workflow gap.
- Do not run a broad vault mining pass unless the user explicitly asks for more crystals.

## Routing table

Classify every candidate before editing:

| Candidate | Durable home | When to write |
|---|---|---|
| Current project status, next action, validation command, key repo/file | Project `## Context card` | It helps a future session re-enter a project in under 60 seconds. |
| Cross-project gotcha | `learnings/<slug>.md` with `type: gotcha` | It cost time once and plausibly will again. |
| Reusable approach that worked in multiple places | `learnings/<slug>.md` with `type: pattern` | It is a method future agents should repeat. |
| Non-obvious choice future-you might second-guess | `learnings/<slug>.md` with `type: decision` | There is a chosen path and credible alternatives. |
| Research conclusion from a larger memo | `learnings/<slug>.md` with `type: research-finding` | The memo is too long, but one conclusion should be findable. |
| Service/subsystem strategic call | `reckonings/<slug>.md` | It should end in HARDEN / EXTEND / REFACTOR / DEPRECATE / FREEZE. Use `/reckon` if substantial. |
| External research question | `research/<slug>.md` | It needs multi-source evidence and a verdict. Use `/research` if substantial. |
| Session/process friction | `retros/<slug>.md` or `RETROS.md` equivalent | It should change kstack or workflow behavior. Use `/retro` if substantial. |
| Cross-project relationship or read path | `contexts/<domain>-context.md` | It changes where agents should start reading. |
| Routine session event | No-op or `proj log` | It is useful provenance but not evergreen knowledge. |

## Value rubric

Promote a candidate when at least two are true:

- **Recurring:** It appears in multiple projects or sessions.
- **Expensive:** Rediscovering it would cost meaningful time, cluster money, or review churn.
- **Surprising:** It corrected an intuitive but wrong assumption.
- **Decision-changing:** It affects architecture, workflow, platform defaults, or research direction.
- **Verified:** There are logs, tests, benchmark numbers, PRs, or live-cluster evidence.
- **Current:** It changes what the next agent should do now.
- **Connective:** It explains how projects relate.

Do not promote when the note is merely a date-stamped activity, a one-off implementation detail, or a stale plan that was never validated.

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
$VAULT/_meta/tag-taxonomy.md
$VAULT/contexts/INDEX.md
```

Then check state:

```bash
cd "$VAULT"
git --no-pager status --short
proj context
```

Preserve unrelated edits. If a file has existing user changes, patch around them.

### 2. Build a crystallization queue

Use a queue so broad mining stays controlled. If SQL is available, create a table like:

```sql
CREATE TABLE IF NOT EXISTS crystals (
  id TEXT PRIMARY KEY,
  source TEXT NOT NULL,
  candidate TEXT NOT NULL,
  route TEXT NOT NULL,
  value_score INTEGER NOT NULL,
  status TEXT DEFAULT 'pending',
  evidence TEXT
);
```

Seed candidates from:

1. `contexts/INDEX.md` and domain maps.
2. High-traffic project pages in the current center of gravity.
3. `learnings/INDEX.md` gaps.
4. Recent `proj` notes from active projects.
5. User-specified topic/project, if any.

For each candidate, write a one-line thesis and route. Drop low-value candidates instead of hoarding them.

Before writing new learnings, dedupe against existing crystals:

```bash
cd "$VAULT"
grep -RInE '<key terms from candidate>' learnings contexts projects --include='*.md'
```

If an existing learning already owns the idea, update discovery surfaces or the existing note instead of creating a near-duplicate.

### 3. Crystallize in batches

Process the queue in small batches:

1. Read the source sections needed for evidence.
2. Decide the route using the routing table.
3. Dedupe against existing learnings and context maps.
4. Write the smallest durable artifact that captures the insight.
5. Add backlinks from context cards or indexes when the crystal is high-value.
6. Mark the candidate done.

Default batch size: 5-12 crystals. Stop before the batch becomes a rewrite project.

### 4. Write learnings correctly

For `learnings/*.md`, use the vault schema:

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

Body sections must be:

```markdown
## Context
## Learning
## Evidence
## Related
```

Use `status: validated` only when the evidence includes tests, benchmark results, live validation, merged PRs, or repeated successful use. Otherwise use `draft`.

### 5. Update routing surfaces

After writing crystals, update only the routing surfaces that benefit:

- `learnings/INDEX.md` for high-value learnings, not every new note.
- Relevant `contexts/*-context.md` if read paths changed.
- Relevant project context cards if the crystal changes current state, gotchas, or read-next links.
- `_meta/tag-taxonomy.md` only if a new tag is used more than twice and is not a synonym.

### 6. Validate

Run:

```bash
cd "$VAULT"
python3 scripts/validate_vault.py
```

If validation fails, fix your own broken links/frontmatter. If failures pre-existed, state that and do not hide them.

### 7. Record the batch

Use `proj log`, not hand-edited project logs:

```bash
proj log kevin-obsidian "Crystallized <N> durable notes from <scope>: <short summary>."
```

If the batch created a reusable insight about the crystallization process itself, consider a learning under `learnings/`.

### 8. Publish the vault to main

Crystallization changes are only useful to future agents in other codebases after the vault is published. After validation and `proj log`, commit and push the vault:

```bash
cd "$VAULT"
git --no-pager status --short
git branch --show-current
git add <files touched by this batch>
git commit -m "Crystallize <scope>"
GH_TOKEN=$(gh auth token -u chokevin) git push origin main
```

Rules:

- Push only after `python3 scripts/validate_vault.py` passes and the vault branch is `main`.
- Do not force-push. If the branch is not `main`, the remote rejects, or unrelated dirty work makes a safe commit ambiguous, stop and state the blocker.
- Include the validated crystals, context/index updates, and `proj log` entry in the commit; do not include unrelated WIP from other repositories.
- If this batch also changes `~/nonwork/kstack`, validate and publish that repo separately with its own commit.

## Output format

Return a concise batch report:

```markdown
Crystallized <N> items from <scope>.

| Crystal | Route | Why it matters |
|---|---|---|
| [[slug]] | learning/context/card | <one-line value> |

Validation: `<command>` passed/failed.
Published: <commit/push status, or blocker>.
Left raw: <what you intentionally did not promote>.
```

## Anti-patterns

- **Giant summaries.** A 2,000-line "summary of everything" is just another raw log.
- **No-route writing.** If you cannot name the durable home, do not write the note yet.
- **Weak learning spam.** Ten shallow learnings are worse than three durable ones.
- **Duplicate crystals.** If a learning already owns the idea, enrich or link it; do not mint a synonym.
- **Evidence-free takeaways.** "Seems like..." is not a crystal. Cite the source.
- **Chronology as structure.** Crystals are organized by use, not by date.
- **Breaking Obsidian links with clever filenames.** Use stable lowercase kebab-case ids.
- **Replacing `proj`.** Routine session logging stays in `proj`; crystallization is the promotion layer.
- **Local-only crystals.** A validated note that never reaches `main` will not help the next repo's agent.

## Exit criteria

- A bounded batch of meaningful candidates was routed.
- Durable artifacts were written in the correct locations.
- Raw logs were preserved.
- High-value crystals are discoverable from `contexts/` or `learnings/INDEX.md`.
- Validation ran and the final answer states the result.
- The vault batch was committed and pushed to `main`, or the blocker is explicit.

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
