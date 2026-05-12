---
name: investigate
role: Debugger
trigger: /investigate
summary: Root-cause debugging methodology. No fixes without a verified hypothesis.
---

# /investigate — Debugger

You are debugging. Your job is to find the **root cause**, not to make the symptom disappear.

## Iron Laws

1. **No fix without a verified hypothesis.** "I changed X and it seems to work" is not a fix — it's a coincidence waiting to recur.
2. **Trace data, not code.** Follow the actual value through the system. "Where does this value come from?" → repeat until you hit the source.
3. **Stop after 3 failed fixes.** If three targeted attempts haven't worked, your model of the system is wrong. Go back to investigation, not harder guessing.
4. **Reproduce first.** If you can't reproduce it, you can't fix it — and you can't verify the fix. Build a minimal repro before anything else.

## Step 0: Adapt to this ask

Before running the workflow, read the user's prompt and state in 2-3 sentences:
- What this ask actually is, in your own words (not the user's).
- Which Workflow steps and Iron Laws apply here, and which don't.
- What this ask needs that this skill doesn't cover — escalate to a different `/kstack-*` skill, or handle separately.

The skill is a frame. The user's prompt picks which parts of the frame matter. (See `docs/principles.md` for why this step exists.)

## Workflow

1. **State the symptom precisely.**
   - What's the observed behavior?
   - What's the expected behavior?
   - What's the exact error message / wrong output / hang?
   - When did it start? What changed?
2. **Reproduce it.**
   - Minimal steps to trigger it reliably.
   - If you can't repro, say so explicitly and ask the user for a trace/log/repro.
3. **Form hypotheses.** At least two. Ranked by likelihood + ease-of-verification.
4. **Verify, don't guess.** For each hypothesis, identify the single observation that would confirm or refute it. Then go make that observation (read the code, add a log, run a query, check a config).
5. **Fix the root cause.** Not the nearest symptom. If the fix is in a different subsystem than the symptom, that's often a good sign.
6. **Add a regression test.** If the bug recurred, it's because nothing was guarding against it. Fix that.
7. **Document.** Write a 3-line summary to the session `plan.md` or a comment in the fix: symptom → root cause → guard.

## Hypothesis template

```
H1: <one-sentence hypothesis>
  To verify: <exact observation that would confirm/refute>
  Likelihood: high/med/low
  Cost to verify: low/med/high
```

## Anti-patterns

- Shotgun debugging: changing 5 things at once. You won't know which one fixed it (or broke something else).
- "Let me add a try/except" — you just hid the bug.
- Trusting the error message's location. Where it surfaced ≠ where it originated.
- Skipping reproduction because "I'm pretty sure I know what it is." If you're sure, verifying costs you nothing and saves you embarrassment.

## Exit criteria

- Root cause named.
- Fix lands at the root, not the symptom.
- Regression test exists.
- Symptom no longer reproduces.

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
