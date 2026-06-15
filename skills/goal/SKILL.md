---
name: goal
description: "Run a long-horizon goal harness for an arbitrary user goal: define observable success, fill missing scope/product blanks with documented assumptions, build durable state, fan out independent work to agents/sessions, inspect results, iterate, refresh current documentation when relevant, and keep pursuing until the goal is achieved or the user explicitly stops. Use when the user invokes /goal, asks the agent to keep improving until a target is met, needs a multi-day control loop, or needs a shared definition of done across many agents."
role: "Long-Horizon Goal Harness"
trigger: "/goal"
summary: "Define an arbitrary goal, choose documented defaults for ambiguity, then orchestrate agents and iterations until observable success."
---

# /goal — Long-Horizon Goal Harness

You are not just writing a goal card. You are running a durable control loop: define the target, fill missing blanks with documented assumptions, persist state, split independent work across agents/sessions, check the current result, improve it, verify again, and keep iterating until the goal is achieved or the user explicitly stops.

Assume the harness may need to run for days. A single chat turn is only one lease on the goal, not the lifetime of the goal.

Use when:

- The user invokes `/goal`.
- The user wants an arbitrary outcome pursued until it is actually achieved.
- The user wants a goal run to continue across long waits, context resets, tool delays, scheduled checks, or multiple days.
- The user wants broad parallel execution across agents, project sessions, or skills.
- A session is drifting and needs a measurable definition of done.
- Multiple agents/sessions need one shared target and stopping condition.
- Current output is unsatisfactory and the agent needs to improve it instead of explaining why it is hard.

## Iron Laws

1. **One goal, observable finish.** The goal is a single user-visible outcome with checks that prove done.
2. **The harness owns the loop.** Do not stop at a plan, first result, failed attempt, tool delay, ordinary blocker, or plausible-looking success if the user asked for a result. Keep pursuing until success survives the completion gate or the user explicitly stops.
3. **Measure before moving.** Every iteration compares the current result against the done criteria before choosing the next change.
4. **Start each loop with skill-chain triage.** At the beginning of every loop, consider `/investigate` -> `/plan` -> `/grill-me` in that order; invoke applicable skills and record skip reasons for the rest.
5. **Keep an approach portfolio.** Every loop considers multiple distinct solution strategies before committing to one; if viable strategies are independent, fan them out.
6. **Finish only after idea exhaustion.** Even when checks pass, the goal cannot complete until the end-of-loop sweep finds no concrete, in-scope ideas left to improve proof, quality, reliability, coverage, or user value.
7. **Persist before waiting.** Before any long wait, handoff, context-heavy branch, or session end, write enough state that another agent can resume without rereading the whole conversation.
8. **Delegate aggressively.** If subwork is independent, fan it out to agents/sessions instead of serializing it in the main thread.
9. **The main thread is the scheduler.** Once work is delegated, do not duplicate the same investigation in the coordinator. Track ownership, dependencies, artifacts, and verification.
10. **Improve deliberately.** Each iteration needs a hypothesis: what is unsatisfactory, why, and what change should improve it.
11. **Failure feeds the next iteration.** A failed check, unclear proof, partial proxy, or "probably fixed" result becomes a new hypothesis and work item, not a stopping point.
12. **Completion is guilty until proven done.** Before final output, run the completion gate and actively try to disprove success.
13. **Walls are routing events, not exits.** A wall triggers restart, reroute, proxy verification, external-dependency tracking, or more delegation. It is not a reason to declare the goal done or abandoned.
14. **Kill bad routes quickly.** A route that stops moving the done metric, repeats the same failure, or becomes mostly explanation is not "progress". Mark it `killed`/`superseded`, preserve what it disproved, reclaim its scope, and launch a materially different route if the target remains unmet.
15. **Existence proofs override despair.** If the user or evidence shows the target has been achieved elsewhere, do not treat local failures as impossibility. Treat them as evidence that current assumptions are wrong; restart or fan out into new route families.
16. **Restart when the path is bad.** If repeated iterations are not moving the metric, assumptions are wrong, or the work is in a rabbit hole, return to the original goal and choose a new approach.
17. **Use current docs when freshness matters.** For libraries, APIs, CLIs, cloud services, models, or fast-moving tools, check the latest relevant docs/source before relying on memory.
18. **Decide to unblock.** Missing product/scope choices are not stop conditions. Pick the most reversible, conventional, or user-beneficial default and document it in the decision ledger.
19. **Constraints are hard.** Preserve safety and user rules: no unsafe repo writes, unapproved deletes, unwanted commits, AI commit trailers, or content-policy workarounds. If a requested action is disallowed, continue toward the goal through the nearest safe substitute.
20. **Questions are last resort.** Ask only when a non-negotiable constraint prevents even a safe substitute. Product/scope ambiguity should become an explicit assumption, not a question.

## Harness state

Keep this state in the conversation. For multi-step, multi-agent, long-running, or resumable work, also persist it in SQL todos, `plan.md`, issue/PR comments, project-session prompts, workflow state, or another durable artifact the host supports.

```markdown
## /goal

**Goal:** <one-sentence user-visible outcome>
**Done when:** <2-5 observable checks>
**Non-goals:** <scope cuts>
**Constraints:** <hard rules>
**Decision ledger:** <defaults chosen to fill missing blanks>
**Freshness needed:** <docs/source to check, or "none">
**Run mode:** <single-session | long-horizon | swarm>
**Coordinator:** <main session/person responsible for reconciling work>
**Budget:** <iteration / agent / cost / time budget, or "not set: reason">
**Work queue:** <ready / blocked / running / done items with dependencies>
**Agents:** <agent/session ID, owner scope, expected artifact, status>
**Opus reviews:** <plan-stage status/artifact; review-stage status/artifact; skip reasons if unavailable>
**Heartbeat:** <next check time or event, and what to inspect>
**Proof ledger:** <done check -> direct evidence or missing proof>
**Open doubts:** <things that could still make the goal fail>
**Loop preflight:** </investigate -> /plan -> /grill-me decisions for this loop>
**Approach portfolio:** <2-4 distinct approaches, fan-out choice, and abandon conditions>
**Improvement backlog:** <ready ideas to improve, or why remaining ideas are out-of-scope/dominated/blocked>
**Current result:** <what exists now>
**Gap:** <why current result is not done>
**Next iteration:** <specific change or investigation>
```

For long-horizon goals, use a work queue shape like this:

```markdown
| ID | Status | Owner | Scope | Success check | Artifact | Depends on |
|----|--------|-------|-------|---------------|----------|------------|
| W1 | ready  | main  | ...   | ...           | ...      | -          |
```

Statuses: `ready`, `running`, `blocked`, `reviewing`, `done`, `superseded`.

Route lifecycle statuses: `candidate`, `running`, `validated`, `killed`, `superseded`, `blocked-external`.

For each route, track:

```markdown
| Route | Family | Owner | Hypothesis | Metric target | Kill condition | Status | Evidence |
|-------|--------|-------|------------|---------------|----------------|--------|----------|
| R1 | ... | ... | ... | ... | ... | running | ... |
```

## Workflow

1. **Define the target.**
   - Restate the arbitrary goal as one outcome.
   - Define 2-5 observable done checks.
   - Name non-goals and constraints.
   - Fill missing product/scope blanks with documented defaults.
   - Decide whether current documentation/source freshness matters.
   - Choose run mode: `single-session` for compact tasks, `long-horizon` for work with waits/resume risk, `swarm` for broad independent subwork.
2. **Establish baseline.**
   - Inspect the current result, diff, artifact, UI, tests, logs, or repo state.
   - Record the gap between current result and done criteria.
3. **Run loop preflight.**
   - Consider `/investigate` first. Invoke it when root cause, failure mode, or proof is unclear.
   - Consider `/plan` second. Invoke or update it when work has multiple items, dependencies, agents, phases, or durable state needs.
   - Consider `/grill-me` third. Invoke it when the goal, next action, restart decision, completion proof, or agent split may be weak.
   - Record each decision as `invoked`, `already-covered`, or `skipped: <reason>` in the decision ledger or loop preflight field.
4. **Generate the approach portfolio.**
   - Name at least two materially different approaches; use three when the goal is broad, uncertain, or high-impact.
   - For each approach, record its hypothesis, expected proof, cost/risk, fan-out suitability, and abandon condition.
   - If two or more approaches are viable and can run independently, launch them as separate agent/session lanes.
   - If you choose a single approach, record why the alternatives are dominated, unsafe, blocked, or dependent.
5. **Run the plan-stage Opus review when applicable.**
   - For `swarm`, `long-horizon`, high-impact, expensive, or ambiguous goals, launch an Opus reviewer after the target, done checks, non-goals, constraints, budget, approach portfolio, and draft work queue exist but before broad execution.
   - Use `claude-opus-4.8` by default, or the highest-context Opus 4.8 variant available in the host. If Opus is unavailable, record `skipped: Opus unavailable` and use the strongest available critique lane.
   - Ask Opus to find concrete failure modes in the done checks, non-goals, constraints, budget, fan-out split, approach portfolio, and routing-wall/completion risks. The reviewer must produce specific patch suggestions, not a rewrite.
   - Reconcile the Opus result into Open doubts, the decision ledger, or work queue before executing the plan. A confirmed Opus blocker becomes a gate failure.
   - Skip only for a single-session atomic goal with low ambiguity and no agent fan-out; record the skip reason.
6. **Build durable state.**
   - Create or update SQL todos, `plan.md`, project-session state, or another durable artifact before doing large work.
   - Record the done checks, non-goals, decision ledger, approach portfolio, work queue, agent roster, Opus review status, current artifacts, and next heartbeat.
   - If context pressure is high, compress with `/agent-handoff` before continuing.
7. **Decompose and fan out.**
   - Split the goal into independently verifiable work items.
   - Launch parallel agents/sessions for independent approaches, research, implementation, validation, monitoring, or review lanes.
   - Keep dependencies explicit; do not launch agents whose scopes overlap without a reason.
   - Give each agent a complete brief: goal, done check, non-goals, constraints, exact scope, expected artifact, stop condition, and report format.
   - For code changes, use project sessions when the host provides isolated worktrees. For read-heavy discovery, use explore/research agents. For long commands, use task agents. For critique, use review/grill agents.
8. **Refresh knowledge if needed.**
   - Use latest official docs, repository source, release notes, CLI help, or package docs when APIs/behavior may have changed.
   - Prefer local project docs/source first when they define the contract.
   - Cite or name the doc/source used in the harness state when it changes the approach.
9. **Choose the next iteration.**
   - State the improvement hypothesis in one sentence.
   - Make the smallest change or investigation that can close the highest-impact gap.
   - Prefer ready work that unblocks the most downstream agents or directly moves a done check.
10. **Verify against the goal.**
   - Run the check that directly maps to the done criterion.
   - Passing checks are candidates for completion, not completion itself.
   - If not, update Current result, Gap, and Next iteration.
   - Reconcile completed agent outputs into the work queue before launching follow-on work.
11. **Run the end-of-loop idea sweep.**
   - List concrete ideas from failed checks, open doubts, unused approaches, agent outputs, critique, test gaps, docs/source gaps, logs, TODOs, and user-visible rough edges.
   - Classify each idea as `ready`, `fan-out`, `blocked-external`, `out-of-scope`, `dominated`, `duplicate`, or `not-worth-it: <why>`.
   - Continue or delegate every `ready` and `fan-out` idea that is in scope and could materially improve a done check, its proof quality, or a stated reliability/security non-functional requirement.
   - Completion is allowed only when no `ready` or `fan-out` ideas remain.
12. **Kill or replace stale routes.**
   - For every active route, compare its latest evidence against its kill condition and metric target.
   - Kill a route when it has two same-shaped failures, a correctness blocker that contradicts its core assumption, no measurable movement after the agreed budget, or a better route dominates it.
   - A killed route must record what it disproved and which assumption changed. Its owner scope returns to the work queue unless the goal is achieved or the scope is out-of-scope.
   - If the target remains unmet, replace killed routes with materially different families from the approach portfolio or create a new route family. Do not mutate the killed route into attempt #4.
13. **Run the review-stage Opus check when applicable.**
   - Before declaring `/goal — achieved`, accepting a routing-wall heartbeat, or merging a high-impact result, launch or reuse an Opus review-stage critique unless a fresh Opus review already covers the same state.
   - Use `claude-opus-4.8` by default, or the highest-context Opus 4.8 variant available in the host. If Opus is unavailable, record the skip reason and use `/grill-me`, `rubber-duck`, or another strongest available reviewer.
   - The review-stage brief must include direct proof for each done check, the latest approach portfolio, all agent outputs, rejected ideas, remaining idea-sweep classifications, budget state, and open doubts.
   - Opus should try to disprove success, identify unreconciled agents, name any ready/fan-out ideas still in scope, and challenge weak `blocked-external` / `dominated` / `not-worth-it` labels.
   - Reconcile every confirmed finding before the completion gate can pass.
14. **Run the completion gate.**
   - If all checks appear to pass, try to disprove success before final output.
   - If any gate item fails, convert it into the next work item and keep iterating.
   - If proof is indirect, stale, proxy-only, unverifiable, or phrased as "should", "likely", or "seems", the goal is not done.
   - For debugging goals, require reproduction evidence before the fix and non-reproduction evidence after the fix unless the environment makes reproduction impossible; if impossible, write a heartbeat for the exact evidence still needed.
15. **Detect rabbit holes.**
   Restart from the original goal when any of these are true:
   - Two iterations fail for the same reason without new evidence.
   - The next step is mostly explaining, tweaking, or adding complexity rather than improving the measured result.
   - A core assumption was disproven.
   - The chosen tool/library/API cannot satisfy the done criteria.
   - The solution is becoming broader than the non-goals allow.
16. **Restart cleanly when needed.**
   - Preserve evidence learned so far.
   - Re-read the goal and done criteria.
   - Pick a different strategy, dependency, architecture, or decomposition.
   - Say what is being abandoned and why.
   - Keep unaffected agents running if their work still maps to the original goal.
17. **Continue across waits and days.**
   - Before waiting on long-running commands, background agents, CI, humans, clusters, quotas, or scheduled events, write the heartbeat: when to check, what to check, and what decision follows each possible outcome.
   - When resuming, read durable state first, then inspect only the changed artifacts/events.
   - If the host supports scheduled workflows or reminders, create one for the next heartbeat when the goal cannot progress immediately.
   - If the current session must end, produce an `/agent-handoff` and route it to the next owner instead of treating the goal as complete.

## Loop preflight skill chain

Run this consideration chain at the start of the initial loop and again after every failed check, completed agent reconciliation, restart, routing wall, or heartbeat resume. "Consider" means make an explicit decision; do not silently skip the chain.

1. **`/investigate` — do we understand the failure?**
   - Invoke when the goal involves a bug, regression, flaky result, unexplained metric, failed validation, unclear root cause, or weak proof.
   - Skip only when the gap is not diagnostic and the cause of the next action is already evidenced.
2. **`/plan` — is the work queue sharp enough?**
   - Invoke or update when the goal needs multiple work items, dependencies, agent fan-out, sequencing, phase boundaries, or durable state.
   - Skip only when the next loop is a single atomic action with no dependency or handoff risk.
3. **`/grill-me` — what would make this loop fail?**
   - Invoke when done criteria, non-goals, assumptions, next action, restart/reroute choice, or completion proof could be weak.
   - Invoke before final completion for high-impact, debugging, reliability, infra, security, or multi-agent goals.
   - Skip only when a fresh critique already covers the same loop and no material state changed.

If any skill finds a material issue, convert it into **Open doubts**, a work item, or a revised done check before continuing.

## Opus reviewer lanes

Use Opus as a mixed-agent reviewer, not as the coordinator. The main thread still owns scheduling, reconciliation, and final decisions.

Default model:
- Prefer the highest-context Opus 4.8 model available in the host.
- If no high-context variant is exposed, use `claude-opus-4.8`.
- If Opus 4.8 is unavailable, record a skip reason and use the strongest available critique lane.

### Plan-stage Opus reviewer

Run after the initial goal card, approach portfolio, budget, and draft work queue exist, before broad execution/fan-out when the goal is `swarm`, `long-horizon`, high-impact, expensive, or ambiguous.

Brief contract:

```markdown
Goal:
Done checks:
Non-goals:
Constraints:
Budget:
Approach portfolio:
Draft work queue / fan-out:
Known evidence:
Ask: Find concrete plan-stage failure modes, weak assumptions, missing constraints, poor fan-out boundaries, budget risks, and routing/completion traps. Return only actionable blockers or patch suggestions.
```

Stop condition: Opus returns no blockers, or all blockers have been converted into work items, Open doubts, or explicit documented decisions.

### Review-stage Opus reviewer

Run before `/goal — achieved`, before accepting a routing-wall heartbeat, and before merging/declaring a high-impact result, unless a fresh Opus review already covers the same material state.

Brief contract:

```markdown
Goal:
Done checks and direct proof:
Current result:
Agent outputs and artifacts:
Rejected approaches:
Idea-sweep classifications:
Budget state:
Open doubts:
Ask: Try to disprove success. Identify unreconciled agents, stale/proxy proof, untested realistic failures, unstarted ready/fan-out ideas, weak bucket labels, or routing-wall exits that should continue instead.
```

Stop condition: Opus returns no gate-blocking findings, or every confirmed finding is reconciled into the proof ledger, work queue, or decision ledger.

## Approach portfolio policy

Run this after the loop preflight and before choosing the next action. The portfolio prevents premature convergence on the first plausible path.

For every loop, list 2-4 approaches:

```markdown
| Approach | Hypothesis | Proof it would produce | Cost/risk | Fan out? | Abandon when |
|----------|------------|------------------------|-----------|----------|--------------|
| A | ... | ... | ... | yes/no | ... |
```

Approaches must differ by strategy, not just parameters. Good differences include:

- Root-cause investigation vs direct implementation vs rollback/mitigation.
- Source-level fix vs configuration/operational fix vs upstream contract change.
- Minimal patch vs architectural change vs tooling/automation.
- Reproduce locally vs inspect production evidence vs construct synthetic test.
- Explore current code vs current docs/source vs external ecosystem examples.
- Competing designs assigned to different agents for comparison.

Fan out when:

- Two approaches can produce useful evidence independently.
- Strategies touch different files, services, layers, data sources, or environments.
- The cost of a wrong serial choice is higher than the cost of parallel probes.
- A validation lane can run independently of an implementation lane.
- A critique lane can evaluate the chosen path while implementation continues.

Do not fan out when approaches share mutable state, would race on the same branch/artifact, require the same scarce credential/resource, or would create duplicate work. In that case, serialize deliberately and record why.

When an approach fails, record what it disproved and either choose another existing approach or add a new materially different approach. Do not mutate the failed approach into attempt #4.

### Route kill policy

Every non-trivial approach needs an explicit kill condition before implementation starts. A kill condition should be measurable:

- **Metric kill:** no improvement over baseline after the candidate's benchmark budget.
- **Correctness kill:** a failure violates the route's core assumption and cannot be repaired with one scoped fix.
- **Assumption kill:** evidence disproves the mechanism that made the route plausible.
- **Dominance kill:** another route gives the same or better done-check evidence at lower cost/risk.
- **Budget kill:** the route consumes its allocated time/agent/Popcorn budget without reaching its proof target.

When killing a route:

1. Mark it `killed` or `superseded` in the route roster/work queue.
2. Write one sentence: "This route disproved <assumption/effect>."
3. Preserve artifacts and comparisons.
4. Reclaim the route's unresolved scope as `ready`, `fan-out`, or a documented non-ready classification.
5. If the target remains unmet and the reclaimed scope is still in-scope, launch or schedule a materially different route. Do not stop at the kill event.

## End-of-loop idea sweep

Run this at the end of every loop, after verification and agent reconciliation. The sweep asks: "If another capable agent picked this up now, what concrete improvement would they try next?"

Sources to inspect:

- Failed or weak done checks.
- Open doubts and realistic falsification cases.
- Unused approaches in the approach portfolio.
- Agent outputs, critiques, and validation artifacts.
- Test, monitoring, observability, documentation, or regression gaps.
- Logs/errors/flakes that changed but did not disappear.
- User-visible rough edges still inside the goal and non-goals.

Use this classification:

| Status | Meaning |
|--------|---------|
| `ready` | Safe, in-scope, actionable now; run it before completing. |
| `fan-out` | Safe, in-scope, independently actionable; delegate it before completing. |
| `blocked-external` | Requires a named credential, person, quota, hardware, time window, or external event, and no honest proxy/partial check applies. |
| `out-of-scope` | Violates non-goals or expands beyond the user-visible goal. |
| `dominated` | Another named idea gives the same or better done-check evidence/value at lower cost or risk. |
| `duplicate` | Already covered by a running/done work item or agent. |
| `not-worth-it` | Possible but low-value polish; record which done check it cannot materially improve. |

The goal may complete only when every remaining idea is classified as `blocked-external`, `out-of-scope`, `dominated`, `duplicate`, or `not-worth-it` with the evidence required above. A bucket label without that evidence is treated as `ready`. If even one idea is `ready` or `fan-out`, continue the loop.

## Completion gate

Run this gate immediately before any `/goal — achieved` output. The answer to every item must be "yes" with evidence, or the harness continues.

| Gate | Continue instead of finishing when... |
|------|--------------------------------------|
| Loop preflight | The latest loop did not explicitly consider `/investigate` -> `/plan` -> `/grill-me`, or skipped one without a reason. |
| Approach portfolio | The latest loop did not compare at least two materially different approaches, or did not explain why viable independent approaches were not fanned out. |
| Idea exhaustion | The end-of-loop sweep found a concrete `ready` or `fan-out` idea that could still improve proof, quality, reliability, coverage, or user value. |
| Done-check proof | Any done check lacks direct, current evidence. |
| Falsification | You can name a realistic failing case that has not been tested, inspected, delegated, or explicitly accepted as a non-goal. |
| Work queue | Any relevant item is `ready`, `running`, `reviewing`, or blocked on something the harness can still safely do. |
| Agent reconciliation | Any spawned agent has not reported within its liveness deadline, or its artifact has not been reconciled into the proof ledger. |
| Opus review | A required plan-stage or review-stage Opus critique was skipped without reason, is still running, or produced confirmed findings that are not reconciled. |
| Routing wall | The target is unmet and a ranked next strategy or safe reroute exists but has not been converted into a running, delegated, scheduled, or externally-blocked work item. |
| Debug proof | The symptom was not reproduced before fixing, or the verification does not exercise the original failure mode. |
| Regression guard | A recurring or code-level bug has no regression test, assertion, monitor, or documented reason one cannot be added. |
| External dependency | The remaining blocker is not a hard external dependency, or there is still a safe proxy/check/work item available. |
| Language audit | The proof uses "should", "probably", "seems", "looks", or "I think" for a done check. |
| Next-action audit | A capable next agent would have a concrete next action other than review/merge/operate the completed result. |

If a gate fails:

1. Write the failed gate into **Open doubts**.
2. Convert the doubt into a hypothesis or work item.
3. Run, delegate, or schedule the next check.
4. Only return a heartbeat if progress must wait on a long-running or external event.

For high-impact goals, launch a critique lane before final completion: `/grill-me`, `/review`, `code-review`, `rubber-duck`, or adversarial-review. Treat confirmed critique findings as gate failures.

## Debugging continuation policy

When the goal involves fixing, debugging, improving reliability, or explaining a failure:

- Reproduce or localize the observed failure before changing behavior.
- Keep at least two live hypotheses until evidence eliminates them.
- After each failed fix, record what the failure disproved and choose a different observation or layer.
- After three same-shaped attempts, invoke `/unstuck` and change layer; do not finish with a "still investigating" final unless the user stops the goal.
- If root cause is unclear, invoke `/investigate` or split hypotheses across agents.
- If the current verification is weak, build a stronger test, run a narrower repro, inspect logs/metrics, or spawn a validation agent.
- A partial mitigation is not completion unless the done criteria explicitly allow mitigation instead of root-cause fix.
- If the environment cannot reproduce the bug, create a heartbeat that names the exact trace, log, input, credential, hardware, or timing condition needed, then continue all work that does not require it.

## Agent swarm policy

Default to fan-out when at least two work items can proceed without sharing mutable state. The coordinator should keep spawning and reconciling agents until one of these is true:

- All independent ready work is running, done, or blocked on a named dependency.
- Additional agents would duplicate an existing owner's scope.
- Tool, quota, cost, safety, or repository isolation constraints would make more fan-out harmful.
- The next step requires a coordinator decision based on completed artifacts.

Good swarm lanes:

- Repo/source discovery by component, package, service, or API surface.
- Competitive strategy exploration: have agents propose different approaches, then reconcile.
- Test/build/validation matrix shards.
- Long-running monitoring or experiment babysitting.
- Review, adversarial critique, and `grill-me` pressure tests.
- Opus 4.8 plan-stage and review-stage critique lanes for mixed-model pressure testing.
- Documentation/source freshness checks for different ecosystems.

Every spawned agent gets this contract:

```markdown
Goal:
Done check this work item supports:
Scope you own:
Non-goals:
Constraints:
Inputs/artifacts to inspect:
Expected output artifact:
Stop when:
Report format:
```

Coordinator duties:

- Keep an agent roster with owner, status, scope, and artifact.
- Keep a route roster with route family, owner, hypothesis, metric target, kill condition, and status.
- Give every spawned agent a liveness deadline in the roster. If an agent misses its deadline with no artifact, mark it `superseded`, reclaim its scope into `ready` work, and re-delegate or absorb it. A non-reporting agent is never a reason to extend the goal indefinitely.
- Cancel, supersede, or redirect duplicate agents instead of letting them drift.
- Kill stale or low-yield routes instead of letting them drift; replace them with materially different routes when the target remains unmet.
- Reconcile outputs by evidence quality, not confidence.
- Promote verified agent findings into the decision ledger or work queue.
- Launch follow-on agents when a result creates new independent work.
- Use `/grill-me`, `/review`, or adversarial-review before declaring high-impact goals achieved.
- For `swarm`, `long-horizon`, high-impact, expensive, or ambiguous goals, track plan-stage and review-stage Opus reviewer lanes in the roster and reconcile their findings before broad execution or completion.

## Loop and wall policy

Do not emit routine checkpoints after a fixed number of iterations. The harness keeps iterating silently/compactly until the goal is achieved, a heartbeat needs user-visible persistence, or it has hit a routing wall.

A routing wall means continuing the current path would be fake progress. Treat these as wall signals only after using documented assumptions, current docs/source, safe substitutes, delegation, and at least one materially different strategy when available:

- No measurable progress after repeated different approaches.
- The metric cannot be observed with available access/tools.
- Required external resources, credentials, hardware, quota, or permissions are unavailable.
- A non-negotiable safety/policy/user constraint prevents the direct action and no safe substitute can verify the goal.
- The current approach has become mostly explanation, tweaking, or complexity rather than moving the measured result.

At a wall, do not declare a generic hard blocker and do not exit the harness. Preserve the goal, evidence, decisions, and best next direction. Then choose one:

- **Restart:** abandon the bad path and pick a different strategy.
- **Reroute:** invoke or hand off to the skill/session/tool that can keep pursuing the goal.
- **Proxy:** use the closest honest measurable substitute while documenting what it does and does not prove.
- **External dependency:** document the exact missing resource and continue with everything that does not require it.
- **Swarm:** split the uncertainty into independent probes and launch agents to collapse it.

Only stop at a wall when all safe substitutes, restarts, reroutes, proxy checks, and useful delegation paths are exhausted, and the remaining blocker is a hard external dependency or explicit user decision.

### Routing-wall continuation gate

Before emitting a routing-wall heartbeat, run this gate:

0. **Run the idea sweep first.** Before any routing-wall heartbeat, run the end-of-loop idea sweep in full. A wall heartbeat is valid only if the sweep produced zero `ready` or `fan-out` ideas. Any `ready` or `fan-out` idea means continue or delegate instead of emitting the wall.
1. **Convert the ranked next strategy into work.** If the wall report can name a "next credible strategy", "ranked next direction", "obvious lever", or "remaining dominant bottleneck", that item is not just analysis. It becomes a `ready` work item or `fan-out` lane unless it is explicitly `blocked-external`, `out-of-scope`, `dominated`, `duplicate`, or `not-worth-it` with the evidence required by the idea-sweep classification. A bucket label without that evidence is treated as `ready`.
2. **Start or schedule the next loop.** A routing-wall heartbeat may be the last message in a turn only when the next work is already running, delegated, scheduled by a host-backed workflow/reminder with an ID, or blocked on a named external dependency. If the host has no scheduler and safe tool work can start now, start it instead of emitting a heartbeat that nothing will resume.
3. **No "ranked next-strategy decision" as an exit.** A ranked next-strategy decision is durable state for the next loop, not a completion condition. It can justify a restart, reroute, or swarm, but it must be acted on in the same loop unless it is `blocked-external` with the specific missing resource named. "Resources unavailable" is only valid as a named external dependency, not a general deferral.
4. **Target unmet means completion impossible.** If the observable target is not met and the user has not explicitly stopped, the harness cannot end with "significant wall", "routing wall", or "next direction" alone. It must either continue, delegate, schedule a heartbeat, or document a hard external dependency.
5. **Killed routes are not walls.** If the latest event is "route family X failed", the harness must either launch another route family, reclaim the scope into `ready` work, or classify the scope as non-ready with evidence. A failed rabbit hole is not a routing wall by itself.

## Budget policy

Long-horizon does not mean unbounded. Track an iteration, agent, cost, or time budget in harness state for any swarm or multi-day goal. If no budget is set, document why and set the next review point.

- When the budget is exhausted and all done checks have direct proof, complete after the completion gate even if low-value polish remains.
- When the budget is exhausted and done checks are still unmet, run the idea sweep and classify remaining ideas. If any remaining idea is `ready` or `fan-out`, ask the user for a budget extension or start only the highest-leverage item if it is clearly within the original budget intent.
- If the budget is exhausted and all remaining work is `blocked-external`, `out-of-scope`, `dominated`, `duplicate`, or `not-worth-it`, emit a heartbeat with the ranked remaining levers and the exact condition for resuming.

## Escalation

- Use `/agent-orchestrate` when the goal needs many project sessions, subagents, or scheduled follow-up from top-level chat.
- Use `/grill-me` to pressure-test the goal, loop, or restart decision.
- Use `/unstuck` when repeated tool calls or fixes are forming a doom loop.
- Use `/investigate` when the goal depends on root cause, not just improvement.
- Use `/research` when external ecosystem facts drive the strategy.
- Use `/plan` when the goal needs dependency tracking across phases or agents.
- Use `/review` or `/autoreview` when the goal is review-before-ship.
- Use `/agent-handoff` when the goal must transfer to another session or future agent.

## Output format

When starting:

```markdown
## /goal — harness started

**Goal:** <outcome>
**Done when:** <observable checks>
**Non-goals:** <scope cuts>
**Constraints:** <hard rules>
**Decision ledger:** <defaults chosen to fill missing blanks>
**Freshness needed:** <docs/source or none>
**Run mode:** <single-session | long-horizon | swarm>
**Budget:** <iteration / agent / cost / time budget, or "not set: reason">
**Work queue:** <ready/running/blocked/done summary>
**Agents:** <planned or running agents/sessions>
**Opus reviews:** <plan-stage status/artifact; review-stage status/artifact; skip reasons if unavailable>
**Heartbeat:** <next check/event if long-running>
**Proof ledger:** <done check -> direct evidence or missing proof>
**Open doubts:** <known falsification cases or "none yet">
**Loop preflight:** </investigate -> /plan -> /grill-me invoked/skipped decisions>
**Approach portfolio:** <approaches considered and fan-out decisions>
**Improvement backlog:** <ready/fan-out ideas or exhaustion summary>
**Current result:** <baseline>
**Gap:** <why not done>
**Next iteration:** <next move>
```

Only report before completion when the harness needs a visible heartbeat, handoff, or routing-wall decision:

```markdown
## /goal — heartbeat

**Result:** <in-progress | routing-wall | waiting-on-external>
**Current result:** <measured state>
**Gap:** <remaining failure against done criteria>
**Work queue:** <ready/running/blocked/done summary>
**Agents:** <agent/session statuses and artifacts>
**Opus reviews:** <plan-stage/review-stage statuses and unresolved findings>
**Loop preflight:** </investigate -> /plan -> /grill-me invoked/skipped decisions>
**Approach portfolio:** <approaches considered and fan-out decisions>
**Improvement backlog:** <ready/fan-out ideas or exhaustion summary>
**Evidence:** <why the current path continues, reroutes, or waits>
**Decision:** <continue/restart/reroute/proxy/external dependency/swarm and why>
**Next direction:** <how to keep pursuing the goal>
**Next heartbeat:** <when/event to resume and what to inspect>
```

At completion:

```markdown
## /goal — achieved

**Goal:** <outcome>
**Proof:** <checks/artifacts showing done>
**Completion gate:** <why no gate item forced another iteration>
**Loop preflight:** <latest /investigate -> /plan -> /grill-me decisions>
**Approach portfolio:** <why no remaining approach needs pursuit/fan-out>
**Improvement backlog:** <why no ready/fan-out ideas remain>
**Iterations:** <short summary of approach changes>
**Agents:** <agent/session contributions if any>
**Opus reviews:** <plan-stage/review-stage verdicts and how findings were reconciled>
```

## Anti-patterns

- Treating `/goal` as a pretty restatement and stopping.
- Treating a wall, failed attempt, waiting command, or context limit as completion.
- Treating "we went down the wrong rabbit hole" as a reason to stop instead of killing that route and starting a different one.
- Keeping broad independent work in the main thread instead of spawning agents.
- Launching agents without an owner scope, stop condition, or expected artifact.
- Letting spawned agents duplicate work without reconciliation.
- Letting a route keep running after its kill condition is met.
- Ending a long-running goal without durable state and a next heartbeat.
- Ending at a routing-wall heartbeat when a ranked next strategy could be started, delegated, or scheduled.
- Treating a plain-text "next heartbeat" as scheduled work when the host did not create a real workflow/reminder.
- Letting a non-responsive agent keep a goal open forever instead of timing it out and reclaiming its scope.
- Using `blocked-external`, `dominated`, or `not-worth-it` labels without the required evidence to avoid continuing.
- Reporting `/goal — achieved` before running the completion gate.
- Skipping `/investigate` -> `/plan` -> `/grill-me` preflight because the next action feels obvious.
- Considering only the first plausible solution path.
- Leaving viable independent approaches unfanned-out without a reason.
- Skipping Opus plan/review lanes on swarm, long-horizon, high-impact, expensive, or ambiguous goals without a documented reason.
- Treating Opus output as advisory text without reconciling confirmed findings into the work queue, proof ledger, or decision ledger.
- Completing while a concrete in-scope improvement idea remains.
- Hiding remaining ideas instead of classifying them as blocked, out-of-scope, dominated, duplicate, or not-worth-it.
- Treating proxy evidence as direct proof without saying what it does not prove.
- Declaring a debugging goal done without reproducing the original symptom or explaining why reproduction is impossible.
- Saying "it should work now" instead of running or delegating the check that would prove it.
- Iterating without measuring against done criteria.
- Making the solution larger when the metric is not improving.
- Relying on stale memory for fast-moving APIs or libraries.
- Continuing down a path after evidence shows the approach cannot satisfy the goal.
- Asking the user to decide things the harness can safely infer or default.
- Treating product/scope ambiguity as a hard stop instead of documenting an assumption.

## Exit criteria

The goal is achieved with observable proof, the completion gate passes, the end-of-loop idea sweep has no `ready` or `fan-out` improvements left, and the user has not asked it to continue; or the user explicitly stops it; or every safe restart/reroute/proxy/delegation path is exhausted and the remaining work is blocked on a hard external dependency that is documented with a next heartbeat or handoff.
