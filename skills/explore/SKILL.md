---
name: explore
role: Subagent Foreman
trigger: /explore
summary: Fan out parallel subagents to investigate without burning your main context. Each subagent gets its own context budget, read-only tools, and returns only a tight summary. The main thread stays clean.
---

# /explore — Subagent Foreman

You are coordinating an investigation that would blow your main conversation's context if you did it inline — many files to read, many repos to crawl, many APIs to spec out, many independent threads. Your job is **not to do the reading yourself**. It is to fan out subagents, give each a focused brief, and synthesize their summaries into a single answer.

This skill exists because context is the scarcest resource in long agent sessions. `huggingface/ml-intern`'s `agent/tools/research_tool.py` solves the same problem with a `research` subagent: independent message list, cheaper model, read-only tool subset, hard context budget, returns only the final summary to the main agent. `/explore` is the manual version of that pattern, expressed as a kstack skill.

## Iron Laws

1. **The main thread doesn't read.** If you find yourself opening files, grepping repos, or fetching URLs in the main conversation, you're doing the wrong thing. The whole point is delegation. The main thread orchestrates and synthesizes; subagents read.
2. **Every subagent has a single, falsifiable brief.** "Look into X" is not a brief. "Does TRL's `SFTTrainer` accept a `chat_template` arg in v0.12+? Cite the source line if yes, the changelog entry if no." is. If you can't write the brief in 1-2 sentences with a clear stop condition, you haven't decomposed the problem yet.
3. **Subagents are read-only by default.** They search, read, fetch, summarize. They do not edit, commit, run mutating commands, or call APIs with side effects. If a subagent needs to mutate, that's a `task` for `general-purpose`, not `explore`. State this distinction explicitly when delegating.
4. **Parallelize independent threads. Serialize dependent ones.** If three subagents could run in parallel without one needing the other's output, launch all three in the same response. If subagent B needs subagent A's findings, do A first, then craft B's brief from A's summary.
5. **Synthesize, don't concatenate.** The output is one coherent answer to the user's question, citing the subagents as evidence. Do not paste subagent summaries verbatim. If the synthesis is longer than the longest subagent summary, you're concatenating.
6. **Cap the fan-out.** Default: ≤5 parallel subagents per round, ≤3 rounds. More than that and you're doing a research project — escalate to `/kstack-research`.
7. **Trust but verify selectively.** If a subagent reports a critical fact (something a decision rests on), spot-check it once in the main thread (one targeted read or grep). Subagents hallucinate too. Don't audit everything — just the load-bearing claims.

## When to use `/explore` vs. just doing it yourself

Use `/explore` when **at least two** of these are true:

- You'd need to read >5 files or fetch >5 URLs to answer the question.
- The investigation has independent threads that could run in parallel.
- The reading is exploratory (most of what you read won't end up cited) — you're paying token cost for filtering.
- You're already past 50% context usage and need to preserve room for the actual work.
- The codebase / repo / domain is unfamiliar enough that you don't know which file is load-bearing.

Use the main thread directly when:

- You know exactly which 1-3 files have the answer. Just `view` them.
- The question needs synthesis with code you've already loaded into context. A subagent doesn't have that context.
- You need to *modify* the thing you're investigating in the same turn.
- It's a single grep / glob / API call. Subagent overhead isn't worth it.

The cost of a subagent call is real (round-trip latency, prompt tokens for the brief, summary tokens back). Don't burn it on trivial lookups.

## Step 0: Adapt to this ask

Before running the workflow, state in 2-3 sentences:
- What this ask actually is, in your own words. Is it a "map this codebase" question, a "spec this API surface" question, a "find all the places that touch X" question, or a "compare these N alternatives" question? The decomposition shape differs.
- Which Workflow steps and Iron Laws apply most. Some asks are one round of fan-out; some need 2-3.
- What this ask needs that this skill doesn't cover — if the question is "should we adopt X?", that's `/kstack-research`. If the question is "why is this broken?", that's `/kstack-investigate`. `/explore` is for **mapping** and **gathering**, not deciding or debugging.

## Workflow

### 1. Decompose

Write the user's question. Then list the **independent threads** of investigation that, if answered, would let you answer the question.

```
QUESTION: <one sentence>
THREADS:
  T1. <falsifiable brief, with clear stop condition>
  T2. <falsifiable brief, with clear stop condition>
  T3. <falsifiable brief, with clear stop condition>
DEPENDENCIES: <T2 needs T1's output | none — all parallel>
```

If you can't list at least 2 threads, you don't need `/explore` — just do it yourself. If you list >5, group or cut. Most exploration questions decompose into 3-4 threads.

### 2. Brief each subagent

For each thread, write a brief that includes:

- **The question.** One sentence, falsifiable.
- **Where to look.** Specific paths, repos, URLs, search terms. Don't make the subagent guess.
- **Stop condition.** "Stop when you've identified the function and read its body" or "Stop after reading at most 5 files." Without a stop condition, subagents over-explore.
- **Output shape.** "Return: function name, file:line, 3-line summary of what it does, and one quoted code snippet." Be specific — the more structured the requested output, the more usable the synthesis.
- **Read-only.** Explicit if the subagent should not modify anything. (`task` with `agent_type: explore` is read-only by design; `general-purpose` is not.)
- **Hard token guidance.** "Keep the response under ~500 words" — subagents will sprawl if not bounded.

A good brief reads like a tight Jira ticket. A bad brief reads like a wish.

### 3. Fan out (parallel where possible)

- Use the `task` tool with `agent_type: explore` for read-only investigation in unfamiliar code.
- Use `agent_type: general-purpose` only when the subagent genuinely needs to write code, run tests, or do multi-step work with side effects.
- Launch all independent subagents **in a single response** (parallel tool calls). Sequential launches waste wall-clock time.
- Do not launch the next round until all subagents from this round have returned.

For dependent threads: do the prerequisite first, read its summary, then craft the dependent brief using its actual output (not your guess at what its output will be).

### 4. Verify load-bearing claims

For each subagent summary, identify any claim the answer to the user's question depends on. For each such claim, do **one** targeted check in the main thread:

- "Subagent says function `foo` at `src/x.go:42` does Y." → `view` `src/x.go:40-60` to confirm.
- "Subagent says no callers of `bar`." → one `grep` for `\bbar\b`.
- "Subagent says the API endpoint returns 200 on empty input." → don't actually call the API; check the test or the route handler source.

Don't audit decorative claims. If a subagent says the file is "well-organized," who cares. Only audit the bricks the answer stands on.

### 5. Synthesize

Write the answer to the user's question. The structure is:

- **TL;DR** — 1-3 sentences. The actual answer.
- **What I found** — the synthesis, organized by what the user needs to know (not by which subagent reported it).
- **Evidence** — citations to specific files / lines / URLs, traceable back to the subagents and your spot-checks.
- **Caveats** — anything a subagent couldn't confirm, anything you didn't explore, anything that contradicted across subagents.

Do not include subagent summaries verbatim unless the user explicitly asks for the raw output. The subagent burned context so you wouldn't have to; pasting their full output undoes that.

### 6. Optional: another round

If the synthesis surfaces a new sub-question that the user actually needs answered (not a tangent), and there's budget, run one more round. Otherwise: hand off, name the open questions, let the user decide whether to drill further.

Hard cap: **3 rounds**. If you're at round 3 and still finding new questions, the original question was too broad — escalate to `/kstack-research` or push back on scope.

## Anti-patterns

- **"Let me just take a quick look myself."** That single `view` becomes ten and you've defeated the skill. Either fan out or commit to doing it inline — don't half-delegate.
- **Vague briefs.** "Investigate the auth module" produces a 2000-token wandering summary. "Find the function that mints session tokens; return the file path, line range, and what algo it uses" produces 80 tokens of signal.
- **Sequential fan-out.** Launching one subagent, waiting, launching the next, when they're independent. Always parallelize the independent ones in a single response.
- **No verification of load-bearing claims.** Subagents hallucinate. The whole synthesis can be wrong if you take an invented file path at face value. Spot-check the bricks.
- **Concatenated synthesis.** Three subagent summaries pasted under three headings. The user could have run the subagents themselves. Synthesis means the answer is *less* text than the inputs, not more.
- **Subagent sprawl.** No stop condition in the brief, no token guidance. Subagent reads 30 files, returns a 1500-word essay. Always set a budget in the brief.
- **Using `/explore` for decisions.** `/explore` maps and gathers. It does not decide. If you're picking between alternatives, `/explore` *feeds* `/kstack-research` or `/kstack-reckon`, but isn't a substitute.
- **Modifying things via `general-purpose` subagents when read-only would do.** Default to `agent_type: explore`. Reach for `general-purpose` only when mutation is genuinely required. Mutating subagents have side effects you can't see in their summary.
- **Spawning a subagent to read one file.** That's a `view` call. Subagent overhead is wasted on trivial work.

## Exit criteria

- Question stated explicitly at the top.
- Decomposition (≥2 threads) was written down before any subagent was launched.
- Each subagent had a falsifiable brief with a stop condition and output shape.
- Independent subagents were launched in parallel.
- Load-bearing claims were spot-checked in the main thread.
- Synthesis is shorter than the sum of subagent summaries and answers the original question.
- ≤3 rounds total. If more were needed, the user was told and given the choice to escalate.

## Notes for the user (read once)

- This skill is modeled on `huggingface/ml-intern`'s `agent/tools/research_tool.py` — the key insight from that code is that the subagent gets its own *independent context*, a *cheaper model*, and a *read-only tool subset*, and returns *only the final summary*. The main agent never sees the intermediate tool outputs. That's what keeps the main context clean. `/explore` enforces the same discipline manually.
- Composes naturally with `/kstack-research` (which is allowed to call `/explore` for its breadth pass), `/kstack-investigate` (which can fan out parallel "where does this value come from?" probes), and `/kstack-reckon` (which can fan out per-axis investigation of a service slice).
- The bounded-gain truth from `docs/principles.md` applies harder here than anywhere else: the upper bound on how much an agent can do is set by context, attention, and recovery. `/explore` directly buys back context budget. Use it on the right questions and the rest of the session breathes.
- If you find yourself wanting to use `/explore` on every question, you're over-applying. Most asks fit comfortably in the main thread. `/explore` is for the genuinely sprawling ones — codebase mapping, multi-repo audits, comparison matrices.

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
