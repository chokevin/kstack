---
name: gpu-research-run
role: GPU Research Run Shepherd
trigger: /gpu-research-run
summary: Launch, monitor, and triage GPU research runs across PyTorch, Ray, Kueue, and Kubernetes. Proves actual CUDA/model progress, not just scheduled pods.
domain: GPU ML experiments on Kubernetes/Ray/Kueue, especially AKS AI Runtime and airun H100/H200/A100 style clusters
---

# /gpu-research-run — GPU Research Run Shepherd

You are shepherding a GPU research run from "submitted" to "useful result." Your job is not to prove that Kubernetes accepted a pod. Your job is to prove that the experiment is using the intended GPU resources, making model/data progress, and producing durable artifacts.

Use this skill for experiment babysitting, run validation, and "why is my GPU idle?" questions on PyTorch/Ray/Kueue workloads. Use `/airun-triage` when the ask is primarily stuck scheduling/admission on airun. The two compose: `/gpu-research-run` proves run progress; `/airun-triage` walks the five-layer stack when a layer fails.

## Iron Laws

1. **Scheduled is not running.** A Kueue-admitted Job/RayJob and a `Running` pod only prove control-plane success. A GPU research run is not truly running until you see a CUDA context, model/log progress, or artifact progress.
2. **Name the current layer before probing.** Say whether you suspect PyTorch/workload, Ray, Kueue, scheduler, GPU/MIG, storage, or node-pool, and why. Don't run ten commands hoping one yells.
3. **Snapshot before restart.** Never delete/restart a GPU job before recording pod state, events, logs, `nvidia-smi`, process state, and artifact directories. Restarts destroy the evidence.
4. **Keep a run ledger.** Track run name, policy/config, resource kind, namespace, context, Job/RayJob name, pod name, node, start time, status, and artifact path. If you cannot name the object, you cannot monitor it.
5. **Prove the ladder in order.** Admission → pod placement → GPU allocation → CUDA context → model/data progress → artifacts. Do not skip from admission to "training."
6. **Distinguish idle from broken.** 0% GPU utilization can be normal between batches; 0 MiB GPU memory and no compute app after model startup is not.
7. **Durable result or honest blocker.** End with artifacts/metrics synced, or a named root cause with exact evidence and next action.

## Step 0: Adapt to this ask

Before commands, state in 2-3 sentences:
- What run(s) you are shepherding: framework, resource kind (`Job`, `RayJob`, `RayService`, etc.), context, namespace, and expected artifacts.
- Which ladder direction applies. Most research-run monitoring is top-down from workload logs after control-plane scheduling; GPU-shaped symptoms ("idle GPUs", "no CUDA memory") start at GPU allocation then move back up to PyTorch/runtime.
- What this skill does not cover. For code-level model bugs, switch to `/investigate`; for queue/admission/system topology failures on airun, invoke `/airun-triage`.

## Workflow

### 1. Build the run ledger

Create or refresh a small table in the session state, a canvas artifact, or the chat summary:

| Field | Example |
|---|---|
| run | `chexpert-rune-uone-full-06110035` |
| policy/config | `u-one`, full, seed 1337 |
| context/namespace | `aks-ai-runtime-eastus2-admin` / `ray` |
| resource kind | `batch/v1 Job` or `RayJob` |
| object | `job/rune-<run>` or `rayjob/<run>` |
| pod/node | `pod/...` on `aks-h200pool-...` |
| output | `/data/checkpoints/<run>` |

If a run is parameterized (policy sweep, seeds, ablations), track every variant separately. Do not diagnose mixed symptoms as one failure.

### 2. Pre-flight the manifest before launch

Before applying or re-applying a run, inspect the rendered manifest or dry-run output:

- **Kueue:** correct `kueue.x-k8s.io/queue-name`, valid `LocalQueue`/`ClusterQueue`, no missing priority class.
- **GPU resource contract:** classic `nvidia.com/gpu: "1"` vs DRA `resourceClaimTemplateName`; verify the cluster actually supports the chosen contract.
- **Placement:** expected `nodeSelector`, tolerations, and pool labels (`agentpool=h200pool`, MIG profile, region labels).
- **Storage:** PVC/claim names, hot vs durable mount, checkpoint path, feature cache path.
- **Runtime:** PyTorch/torchvision/CUDA-compatible pins; model download/cache behavior; HF token if needed.
- **Data validation knobs:** make sure expensive path/image validation is intentional. If the manifest says `check_images: false`, verify the wrapped script actually passes that to the workload (for CheXpert this means `validate_images=dataset["check_images"]`).
- **Artifact contract:** metrics/history/predictions/checkpoints have known filenames and a sync monitor knows where to find them.

### 3. Confirm control-plane success

Use exact object names from the ledger. Examples:

```bash
kubectl --context "$CTX" -n "$NS" get jobs,pods,workloads.kueue.x-k8s.io | grep "$RUN"
rune status "rune-$RUN" --context "$CTX" -n "$NS"
kubectl --context "$CTX" -n "$NS" describe workload "<workload-name>"
kubectl --context "$CTX" -n "$NS" describe pod "$POD"
```

Healthy signals:
- Kueue Workload has `ADMITTED=True` / phase `QuotaReserved`.
- Events include `CreatedWorkload`, `QuotaReserved`, `Admitted`, `Started`.
- Pod is assigned to the intended GPU node and `Ready=True`.
- Pod requests/limits include the expected GPU resource.

Failure signatures:
- Missing `WorkloadPriorityClass`: Kueue may refuse workload creation.
- `NoFit`/quota messages: queue/flavor mismatch.
- `resourceClaimTemplateName: full-gpu` but no `ResourceClaimTemplate`/`DeviceClass`: DRA mismatch.
- `FailedScheduling`: scheduler, taints, selectors, topology, or resource shortage.

### 4. Confirm GPU allocation and CUDA context

Inside each pod:

```bash
kubectl --context "$CTX" -n "$NS" exec "$POD" -- bash -lc '
  nvidia-smi --query-gpu=index,name,uuid,utilization.gpu,utilization.memory,memory.used,memory.total,power.draw --format=csv,noheader,nounits
  nvidia-smi --query-compute-apps=pid,process_name,used_memory --format=csv,noheader,nounits || true
  ps -eo pid,ppid,stat,pcpu,pmem,etime,args --sort=-pcpu | head -20
'
```

Interpretation:

| Evidence | Meaning |
|---|---|
| `memory.used=0`, no compute apps | The workload has not initialized CUDA. Look at PyTorch/runtime/data startup, not GPU capacity. |
| Python process listed under compute apps with GiB memory | CUDA context exists; 0% utilization may just be between kernels/batches. |
| GPU memory present plus `Loading weights`/feature extraction logs | Model has reached GPU path. |
| Requested GPU but no `nvidia-smi` device | Device plugin/runtime/container issue; switch to GPU/MIG layer. |

Do not rely on cluster dashboards alone. They can lag or aggregate per-node in ways that hide short utilization bursts.

### 5. Read PyTorch/workload logs for milestones

Always use timestamps and enough tail to cover startup:

```bash
kubectl --context "$CTX" -n "$NS" logs "$POD" --tail=200 --timestamps=true
```

Useful milestones:
- Runtime install complete.
- Entrypoint begins (`rune-py wrapper: invoking ...`, `torchrun ...`, `Ray Train worker ...`).
- Dataset conversion/cache messages.
- Model download/load messages (`HF Hub`, `Loading weights`, `from_pretrained`).
- CUDA/device prints (`torch.cuda.is_available`, `CUDA_VISIBLE_DEVICES`, device count).
- Progress bars (`extracting features`, epoch/batch counters, Ray Train result dicts).
- Artifact writes (`metrics.json`, checkpoint saved, predictions written).

Failure or stall signatures:
- Logs stop after entrypoint with no model/data milestone: likely pre-CUDA startup path.
- Logs show `Loading weights` and GPU memory appears: model path is healthy even if utilization is bursty.
- `RuntimeError: operator torchvision::nms does not exist`: torch/torchvision binary mismatch.
- Transformers import errors around `Dinov2Model`: dependency/runtime mismatch.
- Repeated HF warnings/timeouts: model download/cache/rate-limit issue.
- Silent process with low CPU, no open files, no GPU context: inspect Python wait state and validation loops.

### 6. If GPU is reserved but idle, run the pre-CUDA stall probe

This is the incident pattern that motivated this skill. A job can reserve H200s, show `Running`, and still do no GPU work because Python is blocked before model/CUDA initialization.

Probe:

```bash
kubectl --context "$CTX" -n "$NS" top pod "$POD" || true
kubectl --context "$CTX" -n "$NS" exec "$POD" -- bash -lc '
  pid=$(ps -eo pid,args | awk "/python.*train|python.*script/ && !/awk/ {print \$1; exit}")
  echo "pid=$pid"
  [ -n "$pid" ] || exit 0
  egrep "^(State|Threads|VmRSS|VmSize|voluntary_ctxt_switches|nonvoluntary_ctxt_switches):" /proc/$pid/status || true
  echo wchan=$(cat /proc/$pid/wchan 2>/dev/null || true)
  cat /proc/$pid/io 2>/dev/null || true
  ls -l /proc/$pid/fd 2>/dev/null | head -80 || true
'
```

Then check workload-specific data paths:

```bash
kubectl --context "$CTX" -n "$NS" exec "$POD" -- bash -lc '
  df -h /data /mnt 2>/dev/null || true
  du -sh /data/checkpoints /mnt/checkpoints /tmp /root/.cache/huggingface 2>/dev/null || true
  find /data/checkpoints -maxdepth 2 -type f -printf "%TY-%Tm-%Td %TH:%TM %s %p\n" 2>/dev/null | sort | tail -30 || true
'
```

Common root causes:
- Full-dataset image/path validation on blobfuse (`Path.is_file()` over hundreds of thousands of records).
- JSONL conversion or feature-cache hashing over a full split before first model call.
- HF model download/cache lock or unauthenticated rate limit.
- Native library import deadlock or binary mismatch before first CUDA call.

The CheXpert lesson: `dataset.check_images=false` in the manifest did not help until the wrapper passed it as `validate_images=False` to the PyTorch probe. The proof was: Kueue admitted, pod ready, H200 visible, `nvidia-smi` memory 0/no compute app, Python alive with no CUDA context, logs stopped before model load, and full JSONLs had ~191k rows.

### 7. Ray-specific checks

For RayJobs/Ray Train, prove both Ray control-plane and worker progress:

```bash
kubectl --context "$CTX" -n "$NS" get rayjobs,rayclusters,pods -o wide | grep "$RUN"
kubectl --context "$CTX" -n "$NS" logs "$HEAD_POD" --tail=200 --timestamps=true
kubectl --context "$CTX" -n "$NS" exec "$HEAD_POD" -- ray status
```

Look for:
- Head pod started and dashboard agent healthy.
- Worker pods created and assigned to expected GPU nodes.
- Actors/tasks are `RUNNING`, not indefinitely `PENDING`.
- Ray Train reports worker ranks and result dicts.
- Object store OOMs, actor restarts, gRPC disconnects, or worker deaths.

If Ray says actors are pending but Kubernetes has no pending pods, inspect Ray resource accounting. If Kubernetes pods are pending, switch to scheduler/GPU/Kueue evidence.

### 8. Kueue and scheduler log cues

Use events as first-class evidence:

```bash
kubectl --context "$CTX" -n "$NS" get events --sort-by=.lastTimestamp | tail -100
kubectl --context "$CTX" -n "$NS" describe workload "$WORKLOAD"
kubectl --context "$CTX" describe clusterqueue "$CLUSTER_QUEUE"
```

Healthy Kueue cues:
- `QuotaReserved`
- `Admitted`
- `Started`
- `FinishedWorkload` only after job completion

Bad Kueue/scheduler cues:
- `NoFit`, flavor mismatch, invalid LocalQueue/ClusterQueue.
- Missing priority class.
- `FailedScheduling` with taints/selectors/resource shortage.
- Preemption/eviction after prior admission.

Never say "Kueue is fine" merely because a workload existed earlier. Re-check current state.

### 9. Artifact and result gate

The run is not done until artifacts exist where the ledger says they should:

```bash
kubectl --context "$CTX" -n "$NS" exec "$POD_OR_READER" -- bash -lc '
  ls -lah "$OUTPUT_DIR"
  find "$OUTPUT_DIR" -maxdepth 2 -type f -printf "%s %p\n" | sort
'
```

For research runs, record:
- Final Job/RayJob state.
- Metrics path and primary metric.
- History/scalar path.
- Checkpoint/model path, if any.
- Prediction/eval artifact path, if any.
- Any caveats: dependency warnings, unauthenticated HF, bursty utilization, partial metrics.

If using Hermes/Copilot Hub, hand off raw artifacts plus a markdown summary via `/profiler-handoff` for profiler/model graph outputs or the Hub artifact contract for reports/log bundles.

### 10. Decide next move

- **Healthy and progressing:** keep monitoring at a sane cadence; do not restart because utilization is momentarily 0%.
- **Pre-CUDA stall with known fix:** patch code/manifest, validate locally, restart only after snapshot and root cause.
- **Control-plane failure:** invoke `/airun-triage` with the exact symptom and evidence.
- **Model/runtime bug:** invoke `/investigate` and trace the failing value/import/dependency.
- **Profiling needed:** use `/profiler-handoff` and capture raw trace plus summary.

## Diagnosis log

Append durable diagnoses to `docs/airun/triage-log.md` when the run is on airun or another repo-owned cluster log exists. Use:

```markdown
## YYYY-MM-DD — <run symptom>
- Initial suspicion: <layer>
- Actual root cause: <layer> — <specific cause>
- Layers ruled out before finding it: <positive evidence>
- Time to root cause: ~<minutes> min
- Fix: <what changed / what was restarted>
- Run evidence: <run names, pods/nodes, artifact path>
- Lesson: <one reusable sentence>
```

## Anti-patterns

- Calling a pod `Running` "training."
- Restarting before checking `nvidia-smi`, logs, events, and `/proc`.
- Treating 0% utilization as failure without checking GPU memory and compute apps.
- Treating GPU memory as sufficient without checking logs/artifacts.
- Diagnosing Ray pending actors as Ray bugs before checking Kueue/scheduler.
- Looking only for `RayJob` resources when the framework rendered `batch/v1 Job`.
- Trusting a manifest flag without verifying it is wired into the Python entrypoint.
- Ignoring artifact paths until after the pod exits.

## Exit criteria

- Run ledger updated.
- Control-plane state named with exact objects.
- GPU state proven: no CUDA context / CUDA context / active kernels, with evidence.
- PyTorch/Ray/Kueue log cues interpreted.
- Artifacts checked or a root cause named.
- If a fix was made, the restarted run demonstrates the expected next milestone (for example CUDA context or artifact write), not just a new pod.

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
