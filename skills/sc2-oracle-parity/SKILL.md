---
name: sc2-oracle-parity
description: >
  Drive sc2-gymnasium native/Ocean fidelity against real PySC2 + StarCraft II
  oracle traces. Use when improving MoveToBeacon parity, comparing native
  outputs to golden traces, running the Docker/AKS SC2 oracle harness, or
  deciding whether a native SC2 claim is faithful.
---

# sc2-oracle-parity: PySC2 Oracle Parity Loop

You are improving `chokevin/sc2-gymnasium` toward a faithful native/Ocean
recreation of the PySC2 MoveToBeacon harness. Real PySC2 backed by the
StarCraft II binary is the oracle. The native C core and Ocean surface are
candidates that must earn parity claims through measured traces.

This skill is a control loop, not a one-shot test runner. Each loop captures or
reuses oracle evidence, classifies the current gap, closes one smallest gap, and
re-runs the oracle comparison before changing the claim.

## Scope gate

Run this skill only when all are true:

- Repository is `chokevin/sc2-gymnasium`.
- The work touches or evaluates PySC2 traces, `SC2GymEnv`, `golden_trace.py`,
  the C-native core, the Ocean C surface, Docker oracle runs, or AKS flex
  oracle runs.
- The desired output is a parity decision, a smaller PySC2-vs-native gap, or a
  faithful next implementation slice.

If the repository does not match, stop and say this skill is only for
`chokevin/sc2-gymnasium`.

## Iron Laws

1. **PySC2 is the oracle.** A schema proxy fixture is useful for CI, but it is
   never parity proof. Parity proof requires `source.kind=pysc2_capture`.
2. **Name the claim level.** Every closeout must say L0/L1/L2/L3/L4, the
   evidence path, and what is still not faithful.
3. **No native change without a classified oracle mismatch.** First identify
   the failing field, phase, seed, and gap class. Then change the candidate.
4. **Train and holdout seeds are required before raising a level.** Seed 7 can
   guide implementation; seed 8 or another holdout must confirm the claim.
5. **Assets stay out of repo and public images.** Do not commit Blizzard
   binaries, maps, full raw asset dumps, or EULA-protected payloads. Use
   mounted `SC2PATH`, Docker volumes, or cluster-local `emptyDir` installs.
6. **Save durable evidence before cleanup.** Keep traces, gap reports, command
   logs, image refs, seed list, and run metadata under session artifacts or PR
   notes.
7. **No green-washing.** If semantic observations, reward, terminal timing, or
   replay behavior still mismatch, say "not faithful yet" and name the gap.

## Parity levels

| Level | Name | Claim allowed | Required proof |
|---|---|---|---|
| L0 | Harness execution | "The real oracle harness runs." | SC2 launches, PySC2 captures `source.kind=pysc2_capture`, trace and logs are durable. |
| L1 | Contract parity | "Native/Ocean match the PySC2 adapter representation contract for this slice." | Direct comparison matches shapes, dtypes, action masks, function specs, selected observation dimensions; Ocean matches flat shape/dtype. |
| L2 | Transition parity | "Native/Ocean reproduce oracle transition summaries for the replayed slice." | For train and holdout seeds, selected feature-layer summaries, player vector summaries, last actions, reward, and terminal flags match for the same replay actions. |
| L3 | Episode parity | "Native policy behavior matches MoveToBeacon outcome for this scripted policy." | Scripted policy reaches equivalent reward/score/termination over enough steps on train and holdout seeds. |
| L4 | Generality | "The approach generalizes beyond the first slice." | Additional policies, seeds, or mini-games pass without target-specific hacks. |

When in doubt, report the lower level.

## Current known baseline

As of the Docker E2E run saved under session artifacts
`files/docker-e2e-202606140157/seed-{7,8}/`:

- SC2 4.10 launched inside linux/amd64 Docker.
- Real PySC2 MoveToBeacon traces were captured for seeds 7 and 8.
- Observed action slice is `function_ids=[0, 7, 331]`:
  - `0`: `no_op`
  - `7`: `select_army`
  - `331`: `Move_screen`
- Reset exposes only `no_op` and `select_army` in the target mask.
  `select_army` enables `Move_screen`.
- Direct comparison matches representation fields but still mismatches selected
  feature contents, player values, last-action values after movement, and reward.
- Ocean comparison matches flat shape/dtype but mismatches reward after reset.

Baseline claim: **L1 for the first observed MoveToBeacon action slice; not L2**.

## Gap taxonomy

Classify every mismatch before choosing a fix.

| Gap class | Examples | First response | Done when |
|---|---|---|---|
| Harness/infrastructure | SC2 fails to launch, `SC2PATH` missing, protobuf/absl failure, Docker architecture mismatch, PVC mount failure | Fix runner/image/env first; do not touch native semantics | Real `pysc2_capture` trace and gap reports are produced durably |
| Contract/shape | Feature shapes, dtypes, action mask size, Ocean flat size mismatch | Align adapter/native/Ocean metadata from trace; add schema test | Shape/dtype/flat-size fields match train and holdout traces |
| Action availability | Move action unavailable until selection, wrong prerequisite action | Trace PySC2 available actions and encode prerequisite policy | Replay action sequence uses PySC2-observed function IDs and availability masks |
| Action encoding | Wrong argument order, queued flag, screen coordinate, native action tuple | Compare action spec and trace action payloads; fix mapper | `action`, `native_action`, and `last_actions` match where expected |
| Observation semantics | Feature nonzero/sum/sha mismatch, sparse coordinates differ | Inspect sparse layer summaries; choose one layer/value to model | Target layer summary matches for train and holdout seeds |
| Player semantics | Player vector nonzero/sum/sha mismatch | Decode PySC2 player vector fields and map native state | Player summary or documented selected fields match |
| Transition/reward | Reward differs after select/move/no-op; terminal timing differs | Extend trace length if needed; model delayed movement/scoring | Reward and terminal fields match train and holdout replay traces |
| Generality | Seed-specific coordinates or hardcoded behavior passes only one seed | Add holdout seeds and reject seed-specific hacks | Same code passes train and holdout without conditionals on seed/trace ID |

If multiple classes fail, fix in this order:

1. Harness/infrastructure
2. Contract/shape
3. Action availability
4. Action encoding
5. Observation semantics
6. Player semantics
7. Transition/reward
8. Generality

## Workflow

### 1. Establish baseline

Record:

- git branch and changed files
- trace policy and seeds
- oracle image or Docker volume used
- artifact directory
- current parity level

Run local deterministic checks before oracle work:

```bash
uv run --extra test pytest
uv run --extra test python -m sc2_gymnasium.native_core --build --compare --seed 7
uv run --extra test python scripts/compare_native_to_trace.py \
  --trace tests/fixtures/pysc2_move_to_beacon_schema_trace.json \
  --mode direct \
  --build
uv run --extra test python scripts/compare_native_to_trace.py \
  --trace tests/fixtures/pysc2_move_to_beacon_schema_trace.json \
  --mode ocean \
  --build
```

### 2. Run or reuse real oracle traces

Prefer current durable traces when they directly cover the changed surface. If
the changed surface could affect capture, action policy, trace schema, native
comparison, reward, terminal behavior, or Ocean metadata, rerun the oracle.

Local Docker E2E shape:

```bash
ARTIFACT_DIR="$HOME/.copilot/session-state/<session>/files/sc2-oracle-$(date +%Y%m%d%H%M%S)"
mkdir -p "$ARTIFACT_DIR"

docker run --rm --platform linux/amd64 \
  -e SC2PATH=/opt/StarCraftII \
  -e PROTOCOL_BUFFERS_PYTHON_IMPLEMENTATION=python \
  -e PYTHONPATH=/worktree/src \
  -v sc2-gymnasium-starcraftii:/opt/StarCraftII:ro \
  -v "$PWD:/worktree" \
  -v "$ARTIFACT_DIR:/results" \
  sc2-gymnasium-oracle:move-to-beacon-parity-local-amd64-pip \
  /bin/sh -lc '
    set -eux
    cd /worktree
    PY=/workspace/.venv/bin/python
    for seed in 7 8; do
      run_dir="/results/seed-${seed}"
      mkdir -p "$run_dir"
      trace="$run_dir/pysc2_move_to_beacon_trace.json"
      "$PY" scripts/capture_pysc2_trace.py \
        --output "$trace" \
        --map-name MoveToBeacon \
        --spatial-dim 64 \
        --seed "$seed" \
        --step-mul 8 \
        --steps 4 \
        --policy move-to-beacon-once
      "$PY" scripts/compare_native_to_trace.py --trace "$trace" --mode direct --build > "$run_dir/gap-direct.txt"
      "$PY" scripts/compare_native_to_trace.py --trace "$trace" --mode ocean --build > "$run_dir/gap-ocean.txt"
    done
  '
```

If the SC2 Docker volume is absent and the user has explicitly accepted the EULA
for this run, install assets into the volume:

```bash
docker volume create sc2-gymnasium-starcraftii
docker run --rm --platform linux/amd64 \
  -e SC2_EULA_ACCEPTED=1 \
  -v sc2-gymnasium-starcraftii:/opt/StarCraftII \
  <oracle-image> \
  /workspace/scripts/install_sc2_linux.sh /opt/StarCraftII
```

AKS/flex E2E shape:

```bash
scripts/acr_build_oracle_image.sh > /tmp/sc2_oracle_image.txt

SC2_EULA_ACCEPTED=1 \
SC2_ORACLE_IMAGE="$(cat /tmp/sc2_oracle_image.txt)" \
ASSET_MODE=emptyDir \
TRACE_POLICY=move-to-beacon-once \
BASE_SEED=7 \
COMPLETIONS=2 \
PARALLELISM=2 \
scripts/run_aks_flex_sandbox_experiments.sh apply

SC2_ORACLE_IMAGE="$(cat /tmp/sc2_oracle_image.txt)" \
scripts/run_aks_flex_sandbox_experiments.sh logs
```

Do not use local `linux/amd64` QEMU `uv sync` failures as evidence against the
oracle harness; use ACR build for cluster images when needed.

### 3. Summarize the action trace

For each real trace, extract:

- `trace_id`
- `action_spec.function_ids`
- reset available action mask
- each step action ID/name/arguments
- each step native replay tuple
- reward and terminal fields
- target coordinates used for `Move_screen`

Reject hardcoded action assumptions when the trace disagrees. The current
MoveToBeacon first slice is three actions, not two.

### 4. Classify the current gap

Read `gap-direct.txt` and `gap-ocean.txt` for train and holdout seeds.

For each mismatch:

```text
seed=<seed>
mode=<direct|ocean>
phase=<reset|step-N>
field=<field>
gap_class=<taxonomy class>
candidate_owner=<native_core|golden_trace|env_adapter|ocean_surface|runner>
```

Choose one smallest gap by priority order. Do not change multiple semantic
layers in one loop unless one cannot be verified without the other.

### 5. Implement one gap

Before editing, write a one-sentence hypothesis:

```text
Hypothesis: If <candidate change>, then <specific fields> will match for seeds
7 and 8 because <oracle evidence>.
```

Make the smallest code change that can close the selected gap. Preserve tests
that prove schema proxy behavior, Python-vs-C toy parity, and real trace
comparison semantics.

### 6. Verify and update claim

Run:

- local deterministic checks
- real oracle comparison for train and holdout seeds, unless the change is
  provably docs-only or schema-only
- direct and Ocean gap summaries

Raise the parity level only when every required proof for the next level passes.
Otherwise keep the level and report the next gap.

## Claim ladder closeout

Every final answer must include this shape:

```markdown
**Parity level:** L<N> — <name>
**Evidence:** <artifact paths and commands>
**Passed:** <specific fields/seeds/modes>
**Still not faithful:** <remaining mismatch classes>
**Next smallest gap:** <one gap with seed/mode/phase/field>
```

Allowed wording:

- L0: "The real oracle harness runs."
- L1: "The native/Ocean representation contract matches this traced slice."
- L2: "The native/Ocean transition summaries match this replayed slice."
- L3: "The scripted MoveToBeacon episode outcome matches."
- L4: "The approach generalizes beyond the first slice."

Forbidden wording unless L2/L3 proof exists:

- "faithful recreation"
- "PySC2 parity"
- "native SC2 replacement"
- "gameplay matches"
- "reward parity"

## Current next recommendation

From the Docker E2E baseline, the next smallest L1 -> L2 gap is
**observation semantics for selected feature layers at reset**:

- direct mode, seeds 7 and 8
- reset phase
- mismatched fields:
  - `observation.feature_screen.nonzero`
  - `observation.feature_screen.sum`
  - `observation.feature_screen.sha256`
  - `observation.feature_minimap.nonzero`
  - `observation.feature_minimap.sum`
  - `observation.feature_minimap.sha256`
- reason: reward mismatches after steps are downstream of missing semantic
  state; last-action mismatches only appear after movement; reset observation
  semantics are the earliest stable semantic failure.

Recommended next loop:

1. Extend trace summaries only if current sparse layer details are insufficient.
2. Decode PySC2 selected `player_relative`, `selected`, `visibility_map`,
   `unit_hit_points_ratio`, `unit_density`, and minimap layers at reset.
3. Choose one layer to model first, preferably `feature_screen.player_relative`.
4. Make Python and C native cores match that layer summary for seeds 7 and 8.
5. Re-run direct comparison and keep L1 until the selected semantic fields match.

## Artifacts

Save new evidence under:

```text
$COPILOT_SESSION/files/sc2-oracle-parity/<yyyyMMdd-HHmmss>-<slug>/
```

If `$COPILOT_SESSION` is unavailable, use the session `files/` directory shown by
Copilot CLI, or `/tmp/sc2-oracle-parity-<timestamp>` as a last resort.

At minimum save:

- `run-metadata.txt`
- `seed-*/pysc2_move_to_beacon_trace.json`
- `seed-*/gap-direct.txt`
- `seed-*/gap-ocean.txt`
- command log or shell transcript
- short human summary of the parity level and next gap

