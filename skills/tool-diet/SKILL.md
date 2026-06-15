---
name: tool-diet
role: Token Cost Auditor
trigger: /tool-diet
summary: Inspect recent tool use for bash-heavy browsing and tell the agent how to switch back to structured, low-context tools.
---

# /tool-diet — Token Cost Auditor

You are auditing the current session for avoidable token/cost waste from tool usage. Your job is to interrupt bash-heavy browsing before it turns into a bloated context window.

## Iron Laws

1. **Evidence, not vibes.** Count actual tool calls before giving advice.
2. **Browsing is not execution.** `bash` is acceptable for builds, tests, package managers, git metadata, runtime diagnostics, logs, and live systems. It is wasteful for `cat`, `head`, `tail`, `find`, `grep`, `rg`, `sed -n`, and `awk` code browsing.
3. **One correction, then resume.** Do not produce a lecture. Name the observed pattern, give the replacement tool path, and hand control back.
4. **Prefer structured tools.** Replacement path is `glob` for filenames, `rg` for content search, and `view` for file/line reads. Batch independent reads.
5. **Escalate broad reads.** If the session is doing exploratory reading across >5 files or multiple repos, recommend `/explore` or an explore subagent so only a summary returns to the main context.

## Workflow

1. Inspect recent tool use for the active session, preferably from `tool_requests` if available.
2. Count:
   - total `bash` calls
   - bash calls whose arguments include browsing commands: `cat`, `head`, `tail`, `find`, `grep`, `rg`, `sed -n`, `awk`
   - structured browsing calls: `glob`, `rg`, `view`
3. If bash browsing is low, say so in one sentence and stop.
4. If bash browsing is material, output:

```text
Tool diet: <N> bash calls, <M> look like file/code browsing. Switch now:
- filenames: glob
- content: rg tool
- file sections: view
- broad exploration: /explore or explore subagent
Resume the task using those tools unless a command must execute something.
```

## Suggested trigger points

- Around turn 20 of a long coding session.
- After three consecutive `bash` calls that only print repository files.
- Before invoking large skills that tend to browse or run workflows: `/plan`, `/investigate`, `/kernel-build`, `/airun-triage`, `/research`, `/agent-merge`.
