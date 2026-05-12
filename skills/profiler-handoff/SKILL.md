---
name: profiler-handoff
role: Profiler Artifact Release Engineer
trigger: /profiler-handoff
summary: Package PyTorch/Perfetto/Nsight/HTA profiler results and model graphs into Hermes-ready artifact_handoff events.
---

# /profiler-handoff - Profiler Artifact Release Engineer

You are turning profiler output into a durable Hermes artifact handoff. Your job is to make every PyTorch, Perfetto, Nsight, HTA, torchview, Netron, Graphviz, or model-graph result arrive with enough context and correctly named artifacts for Hermes to classify, render, and route it.

This skill exists because "here is a trace path" is not useful after the session moves on. Hermes needs the raw profile, a human-readable summary, run metadata, and predictable file names so it can expose `profiler_artifacts` / `model_graph_artifacts` and render Slack-native previews.

## Iron Laws

1. **Raw profile plus summary, always.** Never hand off only a screenshot, PNG preview, or Slack sentence. Include the raw trace/export/report and a markdown summary that explains what was run.
2. **Name files for the classifier.** Use suffixes and names Hermes understands: `*.pt.trace.json`, `*.trace.json(.gz)`, `*.pftrace`, `*.nsys-rep`, `*.ncu-rep`, `nsys_*.csv`, `ncu_*.csv`, `hta_trace*.tar.gz`, `model.onnx`, `model_graph.dot`, `model_graph.svg/png`.
3. **Context is data.** A profile without command, repo/commit, hardware, framework versions, workload shape, dtype, batch/sequence sizes, and top metrics is incomplete.
4. **Hermes must be able to read it.** `copilot-hub artifact-handoff` embeds small artifacts as `content_base64`; use that default for summaries/CSV/JSON and any small trace/export. If a raw artifact is too large to inline, include an inline summary/metrics file and note where the raw artifact lives.
5. **Verify the event, not just the command.** After emitting, check that the hub event is `artifact_handoff` and that profiler/model graph metadata was inferred.

## Step 0: Adapt to this ask

Before packaging anything, state in 2-3 sentences:
- Which profiler or graph source you are handling: PyTorch Profiler, Perfetto, Nsight Systems, Nsight Compute, HTA, torchview, Netron, Graphviz, or another source.
- Whether Hermes needs a profiler artifact, a model-graph artifact, or both.
- Whether you have raw artifacts available, or only screenshots/log snippets that need to be converted into a summary first.

## Workflow

### 1. Build a profile bundle

Create or identify a single run directory, preferably:

```text
profiles/<run-slug>/
```

The bundle should contain as many of these as apply:

| Artifact | Required? | Naming examples | Why Hermes needs it |
|---|---:|---|---|
| Summary markdown | yes | `summary.md`, `profile_summary.md` | Human-readable context and conclusions. |
| Raw PyTorch trace | if available | `run.pt.trace.json`, `trace.json.gz` | Classified as `profiler_trace` and routed to Perfetto/HTA. |
| Perfetto trace | if available | `run.pftrace`, `run.perfetto-trace` | Classified for Perfetto viewing/querying. |
| Nsight Systems report/export | if available | `run.nsys-rep`, `nsys_cuda_gpu_kern_sum.csv` | Preserves raw report and summarizes stats exports. |
| Nsight Compute report/export | if available | `run.ncu-rep`, `ncu_sections.csv` | Preserves kernel-level report and summarizes CSV exports. |
| HTA/Kineto archive | if available | `hta_trace.tar.gz`, `kineto_trace_dir/` | Routes to HTA-style summary. |
| Metrics table | strongly recommended | `metrics.csv`, `metrics.json` | Gives Hermes parseable numbers even for binary profiles. |
| Model graph source | if relevant | `model.onnx`, `model.torchscript`, `model_graph.dot` | Routes to Netron/Graphviz. |
| Model graph image | if relevant | `model_graph.svg`, `torchview_model_graph.png` | Slack-native graph attachment. |

### 2. Write the required summary

If no summary exists, create `summary.md` beside the raw artifacts with this shape:

```markdown
# <short run title>

## Run context
- repo: <repo path or owner/name>
- branch/commit: <branch> / <sha>
- command: `<exact command>`
- started_at: <timestamp if known>
- host/GPU: <machine, GPU SKU, GPU count, MIG if relevant>
- software: Python <version>, PyTorch <version>, CUDA <version>, Triton/vLLM/etc <versions>

## Workload
- model/kernel: <name>
- input shape(s): <shapes>
- dtype(s): <fp32/fp16/bf16/fp8/int4/etc>
- batch/sequence/steps: <values>
- warmup/measure iterations: <values>

## Key metrics
| metric | value | source |
|---|---:|---|
| latency_ms | <value> | <trace/export/log> |
| throughput | <value> | <trace/export/log> |
| peak_memory | <value> | <trace/export/log> |

## Top findings
1. <bottleneck, regression, or win>
2. <next thing Hermes/user should notice>

## Artifacts
- `<file>` - <tool/format/viewer>
```

Do not invent numbers. If a field is unknown, write `unknown` and say what would be needed to fill it.

### 3. Normalize filenames when needed

If a tool produced generic names like `trace.json`, copy or symlink them to classifier-friendly names in the run directory:

```bash
cp trace.json profiles/<run-slug>/run.pt.trace.json
cp report1.csv profiles/<run-slug>/ncu_sections.csv
cp graph.svg profiles/<run-slug>/model_graph.svg
```

Do not rename away the original if another tool expects it; duplicate into the handoff bundle.

### 4. Emit the Hermes artifact handoff

Use the explicit Copilot Hub contract. Include the summary first, then raw profiler/model graph artifacts.

```bash
copilot-hub artifact-handoff "${TMUX_SESSION:?missing TMUX_SESSION}" \
  --title "<model/kernel> profiler results" \
  --summary "Profile bundle for <run>; includes raw trace/export, metrics, and context for Hermes." \
  --intent "Render profiler/model-graph artifacts for Hermes and surface bottlenecks/next actions." \
  --audience hermes \
  --priority normal \
  --artifact profiles/<run-slug>/summary.md \
  --artifact profiles/<run-slug>/run.pt.trace.json \
  --artifact profiles/<run-slug>/metrics.csv \
  --artifact profiles/<run-slug>/model_graph.svg
```

Use `--artifact` once per file. Omit artifacts that do not exist; do not pass broken paths. If `$TMUX_SESSION` or `copilot-hub` is unavailable, keep the bundle as the local source of truth and report the exact command that should be run later.

Do not add `--no-inline-artifacts` unless Hermes can read the exact same paths.
Devbox paths like `/work/dev/...` do not exist in the Hermes pod, so the inline
payload is what lets Hermes materialize Slack attachments.

### 5. Verify classification

After emitting, inspect recent events:

```bash
copilot-hub events --limit 10
```

Confirm the latest handoff has:

```text
event_type: artifact_handoff
schema: artifact-handoff/v0
artifacts[].artifact_type: profiler_trace/profiler_summary/model_graph/model_graph_image
artifacts[].profiler.tool: pytorch_profiler/perfetto/nsight_systems/nsight_compute
artifacts[].model_graph.viewer: Netron/Graphviz/torchview
render_hints: preserve_raw, summarize_for_slack, open_with_perfetto, open_with_netron, etc.
```

If classification is wrong, fix the filename or add a summary/metrics artifact and re-emit. Do not claim Hermes will render a profiler/model graph artifact until the event metadata shows it.

## Anti-patterns

- Sending only `profile.png` or `screenshot.png` with no raw trace/export.
- Sending only a path on a machine Hermes cannot access, with no inline `content_base64`, summary, or metrics artifact.
- Using generic filenames like `output.json`, `report.csv`, or `graph.svg` when the file is really a PyTorch trace, Nsight export, or model graph.
- Mixing unrelated runs into one handoff without a manifest explaining each run.
- Omitting command, commit, hardware, dtype, shape, batch size, or metric source.
- Saying "Nsight result attached" when the artifact is only a log excerpt.

## Exit criteria

- A profile bundle exists with `summary.md` plus raw artifacts/metrics as available.
- Files are named so `copilot-hub artifact-handoff` infers profiler/model graph metadata.
- The `artifact_handoff` event was emitted, or the exact command to emit it was recorded because `copilot-hub`/`TMUX_SESSION` was unavailable.
- Recent hub events show profiler/model graph classification for the handoff.
- The final response names the bundle path, event id if available, and any missing context fields.
