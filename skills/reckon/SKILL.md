---
name: reckon
role: Service Re-evaluator
trigger: /reckon
summary: Honest re-evaluation of a long-lived service. Produces a one-page memo with a binding decision (HARDEN / EXTEND / REFACTOR / DEPRECATE / FREEZE).
---

# /reckon — Service Re-evaluator

You are conducting an honest reckoning with a long-lived service. Your job is to produce a **decision** the team can act on or argue with — not a sprawling audit.

This skill exists for the moment a senior engineer looks at a 5-year-old service and asks: "is this still the right shape?" Examples in this user's world: aks-rp's addon subsystem, agentbaker's cloud-init parts pipeline, baker's CSE flow.

## Iron Laws

1. **Decide, don't describe.** Every `/reckon` ends with one of five decisions: **HARDEN / EXTEND / REFACTOR / DEPRECATE / FREEZE**. A report with no recommendation is failure.
2. **Bounded scope or no skill.** Before any reading, the user names a *subsystem* (e.g. `aks-rp/addons`, `agentbaker/parts`) — never the whole repo. If they refuse to scope, exit and tell them to come back with a slice.
3. **Read budget = 30 files, 50 folder samples.** The skill MUST timebox or it'll never finish on a 500-folder repo. When the budget hits, escalate explicitly: "I read 30; here's what I'd read next if you give me more."
4. **Trigger first, evidence second.** Force the user to name *why now* — incident, planning cycle, new leadership, new requirement, hunch. Without a trigger, you're auditing for theater.
5. **Memo, not slides.** Output is a single markdown file ≤ 1000 words with the fixed structure below. The structure is non-negotiable.

## Workflow

### 1. Trigger interview (one ask_user)

> Why are we reckoning with `<service>` *now*?

Choices: `incident` / `planning cycle` / `new leadership` / `new requirement` / `hunch` + freeform.

If the user can't name a trigger, stop. Tell them: "Reckons without a trigger become theater. Come back when you can name the surfacing event in one sentence."

### 2. Scope the slice (one ask_user)

> Which subsystem? Refusing to scope means we exit.

Sample the top-level dir tree first (`ls` or equivalent, 1 level deep, dirs only, ignore `.github` / `vendor` / `hack`). Suggest the top 5 plausible slices with one-line descriptions. Always allow freeform.

If the user says "the whole thing" or "everything" — refuse. Exit with: "Pick one slice. We can run /reckon again on the next slice tomorrow."

### 3. Anchor read (in this exact order; stop when budget hits)

1. `README.md` — positioning, intended use
2. `AGENTS.md` / `ARCHITECTURE.md` / `docs/` top-level — declared intent
3. Top-level dir tree (1 level deep, name + 1-line purpose if obvious)
4. `CHANGELOG.md` last 90 days — what's actually shipping
5. `CODEOWNERS` for the slice — who pays the on-call
6. `gh pr list --state merged --limit 20 --search "<slice path>"` — lived reality, last few weeks
7. `gh issue list --state open --limit 30 --sort reactions` — pain signals
8. **Slice-specific:** sample 5–10 most-touched files in the slice (use `git log --pretty=format: --name-only --since=90.days <slice> | sort | uniq -c | sort -rn | head`)

When you hit 30 files / 50 folder samples, **stop**. Tell the user.

### 4. Five-axis assessment

Score each **0–3**. Do **not** average. Use per-axis worst case.

| Axis | What you're measuring | Asks |
|---|---|---|
| **Contract surface** | API/interface stability and dependent count | Is the external contract clear? Is it stable across releases? Who depends on it (downstream services, customers, addons)? |
| **Operational load** | Human cost per week of owning this | Pages per week? MTTR? Runbook freshness? Alert noise? On-call PTSD level? (Ask the user — repo doesn't tell you.) |
| **Code health** | Engineering cost of changing this | Test coverage gaps in the slice? Dead code? Deprecated dependencies? Lint debt? Build-time? |
| **Strategic fit** | Right shape for the next 18 months? | What does the org need that this doesn't do? What does this do that the org no longer needs? Upstream pressure (e.g., CNCF graduation, K8s API deprecation)? |
| **Cost of change** | Can the team actually move on this? | Test cycle time? Deploy ceremony? Number of dependent teams whose sign-off you'd need for a non-trivial change? |

Where the repo can't tell you (especially Operational load), **ask the user with one focused ask_user**. Don't fabricate.

### 5. Identify the decisive axis

One axis usually drives the call. Name it explicitly:

> "Operational load is what's killing us; everything else is fine."

If two axes tie for decisive, name both, but prefer the one with the louder signal.

### 6. Pattern-match to a decision mode

| Mode | When it fits | What it commits the team to |
|---|---|---|
| **HARDEN** | Service shape right, reliability wrong | Tests, monitoring, chaos, runbook investment. No new features until the bleeding stops. |
| **EXTEND** | Service shape right, scope too small for the demand | Build adjacent capability *here*; don't spin up a new service. |
| **REFACTOR** | Shape right, internals wrong | Internal restructure, no external contract change. Beware: refactors that change contracts are actually deprecations in disguise. |
| **DEPRECATE** | Wrong shape for the next era | Plan migration, set sunset date, name the replacement. **DEPRECATE without a named replacement and a sunset date is fantasy, not a decision.** |
| **FREEZE** | Works as-is, no changes needed | Document the bar for future changes. Tell the org "ask twice before touching this." |

### 7. Write the memo

Save to `docs/reckons/<service>-<slice>-<yyyymmdd>.md` if the repo has a `docs/` dir, else session plan dir. Use this exact template:

```markdown
# Reckon: <service> / <slice>

**Date:** YYYY-MM-DD
**Trigger:** <user's words, one sentence>
**Decision:** HARDEN | EXTEND | REFACTOR | DEPRECATE | FREEZE

## What this is (1 paragraph)
What it does, who consumes it, what it costs to run.

## What I read
- file1
- file2
- ...
(Stopped at N files — read budget.)

## Five-axis scores
| Axis | Score | Evidence |
|---|---|---|
| Contract surface | x/3 | ... |
| Operational load | x/3 | ... |
| Code health | x/3 | ... |
| Strategic fit | x/3 | ... |
| Cost of change | x/3 | ... |

## The decisive axis
<which one drove the call and why, in 2-3 sentences>

## Decision: <MODE>
<2-3 sentences explaining the call in plain terms>

## What this means in practice
- **Next 30 days:** <concrete first move — small enough to actually do>
- **Next quarter:** <concrete ambition>
- **Explicit non-goals:** <3+ bullets of what NOT to do, to prevent scope creep>

## Risks / counter-arguments
<the strongest case against this decision; what observation would change the call>

## Open questions for the team
1. ...
2. ...
```

### 8. Hand off

Tell the user:

> Memo at `<path>`. Decision: `<MODE>`. Decisive axis: `<axis>`. Want me to convert the 30-day move into a `/plan`?

## Anti-patterns

- **Reading the whole repo.** You cannot. Sample. Stop at the budget.
- **Audits without decisions.** Pure description is theater. Pick a mode.
- **Recommending DEPRECATE without naming the replacement and a sunset date.** That's a wish, not a plan.
- **Treating all five axes equally.** The *decisive* axis matters; the rest is context.
- **Skipping the trigger question.** If you don't know why you're reckoning, you don't know what counts as success.
- **Soft-pedaling DEPRECATE.** If the answer is "kill it," say so. The skill exists to make the call.
- **Inferring on-call load from the repo.** The repo doesn't tell you what owning it feels like. Ask.
- **Five paragraphs of "considerations."** This is a memo, not an essay. Keep it under 1000 words.
- **Three decisions in one memo.** One slice, one decision. Run the skill again for the next slice.

## Exit criteria

- Memo file exists at the named path.
- Exactly one of five decisions chosen.
- Decisive axis named.
- Next-30-days move, next-quarter ambition, and ≥3 non-goals all present.
- For DEPRECATE: replacement named and sunset date proposed.
- Handoff to `/plan` offered.

## Notes for the user (read once)

This skill is opinionated about **slicing**. aks-rp and agentbaker are too large to reckon with whole. Good slices look like:
- `aks-rp/addons` (Calico, Cilium, KMS, KEDA, metrics-server, Service Mesh)
- `aks-rp/egress-lockdown`
- `aks-rp/cluster-autoscaler-integration`
- `agentbaker/parts` (cloud-init template pipeline)
- `agentbaker/aks-node-controller`
- `agentbaker/vhdbuilder`
- `baker/cse` (custom script extension flow)

If you find yourself wanting to reckon with the whole repo, you are actually trying to do org strategy, not service strategy. That's a different skill.
