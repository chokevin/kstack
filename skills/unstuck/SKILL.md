---
name: unstuck
role: Loop-breaker
trigger: /unstuck
summary: You're spinning. Same fix, same error, three iterations in. Stop, diagnose the loop, change the layer of attack — don't iterate on the failing layer.
---

# /unstuck — Loop-breaker

You are mid-task and stuck. Same error keeps appearing. Same fix keeps failing. You've tweaked one parameter four times. The user just typed `/unstuck` because they noticed before you did.

This skill exists for the moment an agent does the same wrong thing repeatedly with growing confidence — what `huggingface/ml-intern`'s `agent/core/doom_loop.py` calls a "doom loop." Their fix is a programmatic detector that hashes tool calls and injects a corrective prompt. This skill is the manual version: stop, name the loop, change layer.

## Iron Laws

1. **Stop after the third identical attempt.** Three calls with the same tool name and substantially the same arguments is a loop, not progress. If you catch yourself doing it, halt before the fourth call.
2. **Name the loop before you break it.** In one sentence: "I have run *X* three times with *Y* hoping for *Z*. It keeps producing *W*." If you can't write that sentence, you don't understand the loop and any new attempt is also blind.
3. **Don't iterate on the failing layer.** If `npm test` keeps failing on the same assertion, don't tweak the assertion harder — go up a layer (is the test even testing the right thing?) or down a layer (is the dependency installed?). Loops happen because you're stuck at the wrong level of abstraction.
4. **No hypothesis, no retry.** Before any next action, state the *new* hypothesis explicitly. "I'll try X again but harder" is not a hypothesis. "X is failing because the env var isn't set in this shell" is.
5. **Ship a diagnosis even if you don't ship a fix.** A loop you understood and escalated is a successful `/unstuck`. A loop you broke by accident is not.

## The five common loop shapes

Most loops fit one of these. Identify which before reacting.

| Shape | Symptom | The actual problem |
|---|---|---|
| **Iterate-the-fix** | Same code change, slightly different each time, same test failure | Wrong layer — the bug isn't in the code you're editing |
| **Re-run-and-pray** | Re-running the same command hoping the answer changes (`kubectl get pods` 6 times) | Need to *change* what you're observing, not how often |
| **Tweak-one-knob** | Changing one config value up and down (timeout=30s, 60s, 90s, 30s) | The knob isn't the issue; you've ruled it out by attempt 2 |
| **Hallucinated-API** | Calling a method/flag/argument that keeps producing "unknown" or "not found" | Your knowledge is outdated — go read the actual current docs/source |
| **Compound-edits** | Each edit fixes one error and reveals a new one, three deep | The original change was wrong; revert and rethink, don't keep patching |

## Step 0: Adapt to this ask

Before running the workflow, state in 2-3 sentences:
- Which loop shape (above) this looks like, in your own words.
- Which Workflow steps and Iron Laws apply most. (Often #2 "name the loop" is the whole skill — once named, the way out is obvious.)
- What this loop needs that this skill doesn't cover — escalate to `/kstack-investigate` if root cause is unclear, or `/kstack-research` if your knowledge is outdated, or **stop and ask the user** if you don't have enough information to break the loop.

## Workflow

### 1. Halt

Stop the in-flight tool call if you can. If you can't, finish it but don't queue another. **Do not call any more tools** until you've completed step 2.

### 2. Name the loop (write it down)

Write a 4-line block — to the user, in chat:

```
LOOP: <one-line shape, e.g. "iterate-the-fix on test assertion X">
ATTEMPTS:
  1. <what I tried, what happened>
  2. <what I tried, what happened>
  3. <what I tried, what happened>
HYPOTHESIS THAT WAS WRONG: <the assumption that produced all 3 attempts>
```

If attempts 1-3 don't share a common wrong hypothesis, you're not in a loop — you're just slow. Continue normally.

### 3. Change layer

Pick exactly one move from this menu. **Don't pick "try attempt #4."**

| Move | When |
|---|---|
| **Go up a layer** — question the test/spec/contract itself | The thing you're trying to satisfy may be wrong |
| **Go down a layer** — check env, deps, file existence, network, perms | The thing under you may be broken |
| **Change tool** — different command, different lens (logs vs metrics, repo grep vs LSP, run vs strace) | Same lens has shown you everything it will |
| **Read the actual source** — go to the library/binary's source on disk or GitHub | Your mental model of how it works is wrong |
| **Revert and rethink** — git stash / git checkout — back to last known good | Compound edits are stacking; the original change was wrong |
| **Ask the user** | The loop is on a decision only the user can make (intent, preference, scope) |
| **Escalate to `/kstack-investigate`** | The root cause is genuinely opaque and you need the structured debugging frame |
| **Escalate to `/kstack-research`** | Your knowledge is outdated (hallucinated API loop) and you need to re-ground in current docs |

State the move in one sentence: "Changing layer: <move> because <why>."

### 4. State the new hypothesis

One sentence. **Different from the wrong hypothesis in step 2.** If the new hypothesis is just a refinement of the wrong one ("maybe the timeout needs to be even higher"), you haven't actually changed layer — go back to step 3.

### 5. Try once

Execute the new hypothesis. Exactly one attempt. If it works, exit. If it doesn't:

- **If the failure mode changed** — that's progress. New error, new cycle. Continue normally outside `/unstuck`.
- **If the failure mode is identical to the loop** — your layer change was illusory. Go back to step 3 and pick a different move. Do not repeat step 5 with the same hypothesis.

### 6. Hard cap: two `/unstuck` cycles

If you've run `/unstuck` twice on the same problem and you're still looping, **stop and hand off to the user**. Two failed unstuck attempts means the problem is outside what an agent can break alone. State:

- The original loop (from step 2 of the first cycle).
- Both layer changes you tried.
- Why each failed.
- What the user could provide that would unblock you (info, decision, access, environment fix).

This is a successful `/unstuck`. Recognizing the unbreakable loop is the goal.

## Anti-patterns

- **"Let me try one more time with a small tweak."** That's attempt #4. The skill exists to prevent exactly this sentence.
- **Naming the loop without changing layer.** Diagnosis without action is theater. Step 2 is necessary, not sufficient.
- **Layer-change in name only.** Switching from `kubectl get pods` to `kubectl describe pod` is the same layer (cluster state inspection). Switching to "read the controller's reconcile loop source" is a different layer.
- **Using `/unstuck` as a shrug.** Every cycle must produce either a fix, a new failure mode, or an escalation with specifics. "I tried something different and it didn't work" is not enough.
- **Burying the loop in chat.** The 4-line LOOP block in step 2 is for the *user* to see, in plain text. Don't hide it in tool output or internal reasoning.
- **Sandbagging the hard cap.** If two cycles failed, three won't succeed. Hand off.
- **Conflating slow with looping.** A long task with steady progress is not a loop. A loop has *negative* progress — same evidence, growing confusion. Don't `/unstuck` work that's actually moving forward.

## Exit criteria

Exactly one of:
- **Broke through:** layer-changed, new hypothesis, single attempt succeeded. Resume the original task.
- **Made progress:** failure mode genuinely changed; you're now debugging a *different* problem. Resume normally.
- **Handed off:** two unstuck cycles failed; user has been told the loop, the layer changes tried, and what they'd need to provide.

In all three cases, the LOOP block from step 2 is in the chat history so the user can see what happened.

## Notes for the user (read once)

- This skill is the manual analog of `huggingface/ml-intern`'s `agent/core/doom_loop.py`, which hashes tool calls and detects 3+ identical or 2-to-5-pattern repetitions, then injects a corrective `[SYSTEM: DOOM LOOP DETECTED]` prompt. The mechanism is the same: detect, name, force layer change. The difference is that `/unstuck` runs in the user's mind too — you can invoke it the moment *you* notice the agent is spinning, not just when the agent self-detects.
- Most loops the user catches are loops the agent didn't notice it was in. That's a feature of agent confidence, not a bug — but it does mean the user typing `/unstuck` is high-signal. Take it seriously; don't dismiss with "I think I almost have it."
- A productive `/unstuck` often *kills* the current approach entirely. That's fine. Sunk cost on three failed attempts is not a reason to make a fourth.
- Compose with `/kstack-investigate` (when the root cause is unclear), `/kstack-research` (when knowledge is outdated), and `/kstack-explore` (when you need to fan out parallel investigations to find a fresh angle).

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
