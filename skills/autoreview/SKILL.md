---
name: autoreview
role: Closeout Reviewer
trigger: /autoreview
summary: Run a structured second-pass review as a closeout check before commit, PR update, or ship. Uses Copilot-native review agents by default; external helpers are optional.
---

# /autoreview — Structured Closeout Review

Run a structured code review as a closeout check on local, branch, commit, or PR work before commit/ship. This is advisory review, not approval routing.

This skill is adapted from OpenClaw's `autoreview` workflow, but made local to Kevin's Copilot/kstack setup: prefer Copilot's `code-review` subagent and available local/GitHub tooling; only use external autoreview helpers if they are already installed and explicitly requested or clearly available.

## Use when

- The user asks for autoreview, second-model review, closeout review, Codex review, Claude review, or review-before-ship.
- You have made non-trivial code edits and want an independent pass before final/commit/ship.
- Reviewing a local branch, PR branch, or specific commit after fixes.

## Iron Laws

1. **Advisory, not automatic.** Treat review output as evidence. Never blindly apply a finding.
2. **Verify every accepted finding.** Read the real code path, adjacent files, tests, and dependency contracts before fixing or reporting.
3. **Reject weak findings.** Discard speculative risks, unrealistic edge cases, broad rewrites, style nits, and fixes that over-complicate the codebase.
4. **Keep roles separate.** The reviewer finds risk; the main agent decides, verifies, fixes, tests, and reports.
5. **Iterate only on real fixes.** If a review-triggered fix changes code, rerun focused validation and rerun autoreview. Stop once there are no accepted/actionable findings.
6. **Do not push just to review.** Push only when the user requested push/ship/PR update.
7. **No nested review loops.** Do not ask the review agent to invoke built-in review panels or additional reviewers. One frozen diff, one review pass unless fixes require a rerun.

## Step 0: Adapt to this ask

Before running the workflow, state briefly:

- What target is being reviewed: dirty local diff, branch vs base, commit, or PR.
- Which review engine/path will be used: Copilot `code-review` subagent by default, or an installed external helper if explicitly requested/available.
- What validation already ran or still needs to run.

## Pick target

Choose the narrowest target that matches the work:

1. **Dirty local work**: inspect `git status --short`, `git diff`, `git diff --cached`, and untracked relevant files.
2. **Branch/PR work**: compare against the actual PR base when available:
   ```bash
   base=$(gh pr view --json baseRefName --jq .baseRefName)
   git diff --stat "origin/$base"...HEAD
   ```
   If no PR exists, use the repository's configured upstream/default base.
3. **Committed single change**: review `git show --stat --oneline <commit>` and the commit diff.
4. **Already-landed main work**: review the explicit commit(s), not an empty `main` vs `origin/main` diff.

## Default Copilot workflow

1. Identify target and collect enough local context for the reviewer: current branch, base, changed files, intent/plan, relevant test command/output if known.
2. Invoke the `code-review` subagent on that exact target. Ask for only production-relevant findings: bugs, security, data loss, concurrency, incorrect behavior, missing error handling, broken contracts, untested critical paths.
3. Verify each finding yourself by reading the relevant files and dependency docs/source/types when needed.
4. Classify each finding:
   - **Accepted**: real issue; fix or report as blocker.
   - **Rejected**: not a real risk, intentional invariant, out of scope, or over-complicated. Keep the rationale brief.
   - **Needs user decision**: behavior/scope ambiguity that should not be assumed.
5. If accepted fixes are made, rerun focused tests/validation and rerun autoreview on the updated diff.
6. Stop when the final review pass has no accepted/actionable findings, or when remaining issues are consciously rejected/deferred with rationale.

## Optional external helper workflow

If an autoreview helper exists in the repo or a known local install, prefer invoking it directly rather than reimplementing its bundling logic:

```bash
.agents/skills/autoreview/scripts/autoreview --help
skills/autoreview/scripts/autoreview --help
~/.codex/skills/agent-scripts/autoreview/scripts/autoreview --help
```

Use examples:

```bash
<autoreview-helper> --mode local
<autoreview-helper> --mode branch --base origin/main
<autoreview-helper> --mode commit --commit HEAD
```

Rules for helper use:

- Keep the requested engine/model. If the user asked for Codex or Claude specifically, do not silently switch.
- Be patient with long model calls. Heartbeat lines like `review still running: ... elapsed=... pid=...` are healthy progress.
- Do not terminate a review because it is quiet for a few minutes. Inspect only after missed heartbeats, an obvious subprocess failure, or an unusually long run.
- If helper output is noisy, summarize only after it exits. Do not ask another reviewer to rerun the same review for nicer wording.
- If the helper exits nonzero with actionable findings, verify before fixing/reporting.

## Parallel closeout

It is OK to run tests and review in parallel after formatting, as long as both use the same frozen code state. If tests or review lead to edits, rerun affected tests and rerun autoreview.

## Output format

```markdown
## /autoreview — <target>

**Review command/path:** <code-review subagent | helper command>
**Validation:** <tests/proof run, or why not run>
**Result:** <clean | findings accepted | findings rejected | blocked>

### Accepted findings
- **[severity] path:line — summary.** Why it is real; fix applied or required.

### Rejected findings
- **path:line — summary.** Brief reason rejected.

### Remaining action
- <only include if something still needs user/code action>
```

If clean, keep it short: state the target, review path, validation, and `No accepted/actionable findings.`

## Anti-patterns

- Treating review output as authoritative without reading the code.
- Filing style, taste, naming, formatting, or speculative architecture comments.
- Reviewing a clean branch with an empty diff and calling it meaningful.
- Running multiple reviewers by default just to feel safer.
- Pushing just to make a review target easier.
- Adding comments solely to appease a rejected finding.

## Exit criteria

Either:

- `No accepted/actionable findings` for the final reviewed target, or
- A concise list of accepted blockers/risks with concrete fixes or user decisions needed.
