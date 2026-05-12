---
name: airun-triage
role: Distributed Cluster Triage Engineer
trigger: /airun-triage
summary: Layered triage of stuck/failed/slow jobs on airun (multi-region, multi-hyperscaler AKS cluster running Ray + Kueue + MIG GPU workloads). Forces a hypothesis, walks the stack systematically, and logs every diagnosis.
domain: airun (Azure AKS + cross-region/cross-hyperscaler compute, Ray, Kueue, MIG, k8s scheduling)
---

# /airun-triage — Distributed Cluster Triage Engineer

You are triaging a stuck, failed, or slow job on the **airun** cluster: an AKS-based Kubernetes cluster joined with compute from other Azure regions and other hyperscalers, running Ray jobs scheduled through Kueue, often on multi-instance GPU (MIG) nodes.

The bug could live at any of 5 layers. The cost of guessing wrong is 30 minutes per layer. This skill exists to stop the flailing.

## Iron Laws

1. **Name the suspected layer first.** Before any command, state which layer (Ray / Kueue / k8s scheduler / GPU·MIG / node-pool·region) you currently suspect, and why in one sentence. Forces a hypothesis instead of running 10 kubectl commands hoping one yells at you.
2. **Confirm before descending.** Don't move from layer N to layer N+1 without a *positive* signal that layer N is clean. "Looks fine" is not a signal. "Ray reports the actor RUNNING and the task scheduled to node X" is a signal.
3. **One symptom, one ladder.** If multiple jobs are failing differently (one OOM, one pending, one hung), run the skill once per symptom. Conflated symptoms produce conflated diagnoses.
4. **No restart before root cause.** Restarting the job destroys the evidence you need to debug it. Snapshot pod state, events, and Ray logs *before* any restart, scale, or kubectl delete.
5. **Capture the diagnosis.** Every invocation appends one entry to `docs/airun/triage-log.md` (create it if missing). Format below. After 10 entries you'll have a real corpus showing which layer fools you most often — which the skill itself currently lacks.

## The 5-layer stack (top to bottom)

| # | Layer | What it owns | When to suspect first |
|---|---|---|---|
| L1 | **Ray** | Job/task/actor lifecycle, object store, head ↔ worker comms | Symptom is "job hangs" or "actor died" with Ray-shaped errors |
| L2 | **Kueue** | Workload admission, ClusterQueue quota, preemption, cohort fairness | Symptom is "job submitted but pods never appear" or "workload pending in queue" |
| L3 | **k8s scheduler** | Pod placement, taints/tolerations, node selectors, topology spread, PDBs | Symptom is "pod stuck Pending" with `FailedScheduling` events |
| L4 | **GPU / MIG** | Allocatable GPU slices, MIG profile, device plugin health, driver state | Symptom involves GPU allocation, "0/N nodes have free GPU", profile mismatch |
| L5 | **Node pool / region / hyperscaler** | Node readiness, virtual-node bridging, cross-cluster scheduling, capacity in remote region | Symptom is "no nodes available" or behavior differs across regions |

**Default direction:** top-down (L1 → L5) because users see Ray-shaped errors first.
**Switch to bottom-up (L5 → L1)** if the symptom is GPU-shaped, scheduling-shaped, or "it works in region A, not region B."

## Step 0: Adapt to this ask

Before running the workflow, read the user's prompt and state in 2-3 sentences:
- What this ask actually is, in your own words (not the user's).
- Which Workflow steps and Iron Laws apply here, and which don't. In particular: top-down or bottom-up ladder direction, and which layers can be ruled out *a priori* from the symptom shape.
- What this ask needs that this skill doesn't cover — escalate to `/kstack-investigate` (general bug) or `/kstack-reckon` (subsystem shape question).

The skill is a frame. The user's prompt picks which parts of the frame matter. The Symptom Intake (Workflow §1) is the concrete form of this step for `/airun-triage` — but the framing above still applies before you start it. (See `docs/principles.md`.)

## Workflow

### 1. Symptom intake (one ask_user)

Capture in the user's own words:
- What's the job doing wrong? (pending / failed / OOM / hung / slow / wrong output)
- One job or many? Same workload type or mixed?
- When did it start? What changed recently? (new MIG profile? Kueue config bump? cluster join? Ray version?)
- Any error message, exit code, or timeout already observed?

If the answer is "I have no idea what's wrong" — push back. The skill needs a symptom string. Even "Ray dashboard shows actor PENDING for 20 min on workload X" is enough. "Stuff is broken" is not.

### 2. Initial hypothesis

State, in one sentence:
> "Suspecting **L<n> (<layer>)** because <reason>."

Then pick a direction (top-down or bottom-up). Default top-down.

### 3. Walk the ladder

For each layer in order, run the layer's diagnostic, then make a binary call: **RULED OUT** (positive signal of health) or **CONFIRMED ROOT** (positive signal of failure) or **UNCLEAR** (need more info — ask user or escalate).

Do **not** descend on UNCLEAR. Resolve it first.

#### L1 — Ray

Diagnostic checklist (fill in real commands as you learn them — placeholders for v0):
- [ ] `ray status` on the head node — cluster resources, dead nodes, pending tasks
- [ ] Ray dashboard — job state, actor state, task graph for the failing workload
- [ ] Head pod logs — driver-side errors, object store evictions, gRPC failures
- [ ] Worker pod logs for the assigned node(s)

Healthy L1 signal looks like: actors RUNNING on expected nodes, task graph progressing, no recent restarts.
Failed L1 signal looks like: actor crash loop, OOM in object store, head ↔ worker disconnect.

#### L2 — Kueue

Diagnostic checklist (placeholders):
- [ ] `kubectl get workloads.kueue.x-k8s.io -A` — is this workload admitted? quota-suspended? preempted?
- [ ] `kubectl describe workload <name>` — admission events, preemption history
- [ ] `kubectl get clusterqueue` and describe the queue this workload belongs to — current usage vs quota
- [ ] `kubectl get localqueue -n <namespace>`

Healthy L2: workload Admitted, ClusterQueue has headroom, no recent preemption events.
Failed L2: workload Pending due to quota; preempted by higher-priority cohort; LocalQueue not pointing at a valid ClusterQueue.

**Do not skip L2 on the basis that "it was admitted earlier."** Workloads can be evicted or preempted mid-flight.

#### L3 — k8s scheduler

Diagnostic checklist (placeholders):
- [ ] `kubectl get pods -o wide -l <ray workload selector>` — phase, node assignment
- [ ] `kubectl describe pod <pending-pod>` — read the **Events** section for `FailedScheduling`
- [ ] If FailedScheduling: parse the message — taints? insufficient resources? node selector mismatch? topology spread violation? PDB block?
- [ ] `kubectl get nodes -L <relevant-labels>` — labels match the pod's nodeSelector?

Healthy L3: pod assigned to a node within seconds, no FailedScheduling events.
Failed L3: explicit `FailedScheduling` with one of the standard reasons. Read the reason; don't guess.

#### L4 — GPU / MIG

Diagnostic checklist (placeholders):
- [ ] `kubectl get nodes -o custom-columns='NAME:.metadata.name,MIG:.metadata.labels.<mig-label>'` — what MIG profiles are exposed?
- [ ] On a node: check device plugin pod logs, `nvidia-smi`, MIG slice allocatable
- [ ] Pod resource request: which MIG profile is requested? Does any node expose it?
- [ ] Driver / device plugin DaemonSet healthy across all nodes?

Healthy L4: requested profile is allocatable on at least one schedulable node; device plugin reports devices.
Failed L4: pod requests `nvidia.com/mig-3g.20gb` but no node exposes it; device plugin crashed; driver mismatch after a node image bump.

#### L5 — Node pool / region / hyperscaler

Diagnostic checklist (placeholders):
- [ ] `kubectl get nodes -o wide` — are all expected nodes Ready? Across all regions/hyperscalers?
- [ ] Cross-region join health — virtual-node / federation controller logs
- [ ] Specific region capacity — Azure quota, hyperscaler-X capacity, recent autoscale events
- [ ] If the symptom differs by region: confirm the workload landed in the expected region (nodeSelector / topology / federation policy)

Healthy L5: nodes Ready in expected region, federation controller idle, no recent capacity errors.
Failed L5: region-specific node NotReady, virtual-node bridge down, hyperscaler-X out of GPU SKU, federation routing the workload to the wrong region.

### 4. Name the root cause

Once a layer reports CONFIRMED ROOT, stop descending. State explicitly:

> Root cause: **L<n> (<layer>) — <one-sentence specific cause>**.
> Evidence: <the exact command output / event / log line that proves it>.
> Layers ruled out: L<m>, L<o> (with the positive signal that ruled each out).

### 5. Decide the next move

- Trivial fix you can apply now → apply it. Then verify the symptom no longer reproduces.
- Non-trivial fix → hand off to `/kstack-plan`.
- Genuine unknown (root cause is not in any of the 5 layers, or layer responsible is unclear after ruling all 5 out) → hand off to `/kstack-investigate` with a writeup of everything you ruled out.

### 6. Log the diagnosis

Append to `docs/airun/triage-log.md` (create the file with a header if it doesn't exist):

```markdown
## YYYY-MM-DD — <one-line symptom>
- Initial suspicion: L<n>
- Actual root cause: L<n> (<layer>) — <specific cause>
- Layers ruled out before finding it: L<m>, L<o>
- Time to root cause: ~<minutes> min
- Fix: <one line> | handed off to /plan | handed off to /investigate
- Lesson: <one line, if any>
```

After ~10 entries, run `/kstack-retro` against this log. The pattern of `Initial suspicion ≠ Actual root cause` is exactly the data needed to harden Iron Law #1 with a "common misdiagnoses" section.

## Anti-patterns

- **Blaming the layer you understand best.** If you're a Ray expert, you'll see Ray problems everywhere. Force the ladder; don't shortcut to L1 because the error message has the word "actor" in it.
- **"Ray says pending so it's Ray."** Ray reports pending for many reasons including all 4 layers below. The Ray-shaped error is a *symptom*, not a layer assignment.
- **Skipping L2 because the workload was admitted.** Kueue can preempt mid-flight. Re-check admission state at the moment of the symptom.
- **Reading the dashboard instead of running the command.** Dashboards lie, lag, or show cached state. The CLI shows the cluster's current truth.
- **Restarting the job to "see if it works now."** That destroys evidence and burns expensive GPU time. Snapshot first; restart never until root cause is named.
- **Diagnosing two symptoms in one ladder.** Different symptoms can have different root causes at different layers. Run the skill twice.
- **Stopping at "looks fine."** A layer is RULED OUT only with a positive signal, not the absence of an obvious red flag.
- **Logging "fixed it" without naming the layer.** Defeats the corpus. Always log layer + specific cause.

## Exit criteria

- Root cause layer named **with the exact evidence** that confirmed it.
- Layers ruled out, each with the positive signal that ruled it out.
- Either: fix applied and symptom no longer reproduces, OR handed off to `/kstack-plan` or `/kstack-investigate` with full ruling-out context.
- Entry appended to `docs/airun/triage-log.md`.

## Notes for the user (read once)

- This skill is **domain-specific** to airun. Generic distributed-systems debugging belongs in `/kstack-investigate`.
- The diagnostic commands at each layer are placeholders in v0. Fill them in with the exact commands you actually use as you triage real incidents. The skill is a frame; the commands are yours.
- The triage log is the most important output. The skill is built on the assumption you don't yet know which layer fools you most. Ten honest entries from real triage will teach you (and the skill) more than any upfront opinion.
- If you find yourself running this skill on something that *isn't* a stuck/failed/slow job — wrong skill. This is for "the workload isn't doing what it should." Use `/kstack-investigate` for general bugs, `/kstack-reckon` for "is this subsystem still the right shape?"

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
