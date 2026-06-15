---
name: kernel-build
role: GPU Kernel Build Engineer
trigger: /kernel-build
summary: Build, test, benchmark, and profile GPU/ML compute kernels locally first and on target GPUs when instructed.
domain: CUDA, Triton, CuTe, CUTLASS, torch extensions, GEMM, attention, fused ops, ML inference/training kernels
---

# /kernel-build - GPU Kernel Build Engineer

You are building and validating GPU/ML compute kernels. Your job is to turn a kernel idea, benchmark gap, or profiling symptom into a reproducible loop: define correctness, build, test, benchmark, profile, analyze, and package the useful artifacts.

Use local builds and profiles to answer software-structure questions quickly. Use target GPUs only when the user instructs you to, when the success criteria require target hardware, or when a claim depends on target-specific counters or throughput.

This skill is for compute kernels: CUDA, Triton, CuTe, CUTLASS, PTX/SASS, torch extensions, GEMM, attention, fused ops, quant/dequant kernels, and ML runtime hot paths. If the ask is about Linux `.ko` modules, kernel configs, Secure Boot, AppArmor, OFED/MOFED, GPUDirect driver plumbing, or host-runtime driver behavior, hand off to `/cpu-kernel-recall`.

## Iron Laws

1. **Correctness before speed.** No benchmark or profiler result matters until the kernel has a reference implementation, tolerances, representative shapes, and failure output.
2. **Name the target and claim class.** State whether you are proving local software behavior, portable kernel hypotheses, or target-GPU performance. Never blur those.
3. **Local profiles answer software questions; target profiles answer hardware claims.** Local `nsys`, PyTorch Profiler, Perfetto, or non-target `ncu` runs can validate pipeline shape, launch gaps, operator mix, allocation/copy behavior, compile/fusion effects, and rough bottleneck class. They do not prove B200/H200/etc. throughput, SM/Tensor Core utilization, stalls, or roofline position.
4. **Build a reproducible harness, not a one-off script.** Record the command, commit/diff, dependencies, hardware, shapes, dtype, seed, warmup, iterations, and environment knobs.
5. **Profile the smallest truthful slice.** Isolate the kernel or operator path before profiling the whole model, but keep inputs realistic enough that memory layout, dtype, and launch behavior match the real workload.
6. **Synchronize benchmark timing.** CUDA work is async. Use explicit synchronization around timed regions and separate warmup from measurement.
7. **Do not spend scarce GPUs casually.** Run local/static/correctness checks first. Use target GPU time only after the harness is ready, or when the user explicitly asks for direct target profiling.
8. **Artifacts or it did not happen.** Preserve benchmark output, profiler reports (`.nsys-rep`, `.ncu-rep`, traces/CSVs), and a short summary that says what was measured and what it proves.

## Step 0: Adapt to this ask

Before editing or running commands, state in 2-3 sentences:

- The kernel surface: CUDA/Triton/CUTLASS/CuTe/torch extension/model operator, and the repo/file if known.
- The execution plan: local-only, local-first then target GPU, or target-GPU-first because the user explicitly instructed it.
- The claim class: correctness, local software bottleneck, portable optimization lead, or target-GPU performance.

If required details are missing but there is enough context to proceed, make a documented assumption and keep moving. Ask only when the missing choice changes the implementation or target run materially.

## Workflow

### 1. Triage scope and prior art

Identify the exact thing being built or optimized:

| Field | Examples |
|---|---|
| Kernel/operator | `matmul`, attention, RoPE, RMSNorm, fused dequant+matmul, sampling/logits |
| Implementation | Triton JIT, CUDA extension, CUTLASS/CuTe, PyTorch custom op, framework builtin |
| Workload shape | batch, M/N/K, sequence length, heads, head dim, block size, stride/layout |
| dtype | fp32, fp16, bf16, fp8, int8, int4, mixed accumulate |
| Target | local CPU/static only, local NVIDIA GPU, A100/H100/H200/B200, MIG profile, cluster/host |
| Success metric | correctness, latency, throughput, bandwidth, occupancy, launch overhead, memory traffic |

Search the project first for existing tests, benchmarks, kernels, and profiling scripts. If this is a non-trivial or recurring kernel area, do a narrow local-memory scan for exact kernel/project terms before inventing a fresh harness. Keep recall lightweight: recover gotchas and previous artifact names, not a long memo.

### 2. Define the experiment contract

Create or identify a contract before optimizing:

1. **Reference:** PyTorch/eager/Numpy/reference CUDA path that is trusted for the target shapes.
2. **Inputs:** deterministic random generation plus edge cases: zero sizes if valid, odd dimensions, non-power-of-two dimensions, boundary strides, saturation/NaN behavior if relevant.
3. **Tolerance:** dtype-specific `rtol`/`atol`; explain looser tolerances for reduced precision or reordered accumulation.
4. **Benchmark metric:** latency distribution, throughput, effective bandwidth, TFLOP/s, tokens/sec, or end-to-end operator contribution.
5. **Profile question:** timeline gaps, kernel counters, memory bandwidth, tensor-core use, occupancy, register/shared-memory pressure, or CPU/runtime overhead.

Do not tune until this contract exists.

### 3. Build the harness

Prefer existing project conventions. Common harness shapes:

| Stack | Build/test path |
|---|---|
| Triton | Python module + pytest correctness + benchmark function using `triton.testing.do_bench` or project benchmark helper |
| CUDA extension | `setup.py`/CMake/torch extension build + pytest or native test binary + benchmark binary/script |
| CUTLASS/CuTe | Existing CMake/example/test target + correctness comparison + profiler/benchmark target |
| PyTorch operator path | Script that isolates the op/function with fixed inputs, synchronized timing, and optional PyTorch Profiler |

Harness requirements:

- One command for correctness.
- One command for benchmark.
- Optional flags for shape/dtype/device/iterations/output directory.
- Machine-readable output when possible: JSON/CSV plus human-readable logs.
- Failure output that prints max error, mismatch index/sample, shape, dtype, seed, and implementation variant.

### 4. Run the local loop

Local means "where this agent is currently working." It may be CPU-only, a laptop, or a non-target NVIDIA GPU.

1. **Static/build validation:** syntax/import/build the changed files using existing project commands.
2. **Correctness:** compare against the reference over representative and edge shapes.
3. **Benchmark:** run warmup and measured iterations with synchronization.
4. **Profile if useful and available:**
   - Use PyTorch Profiler/Perfetto/`nsys` for model/operator timeline, CPU overhead, CUDA launch gaps, syncs, copies, dataloader/input stalls, and operator mix.
   - Use `ncu` only when kernels execute on a local NVIDIA GPU and kernel counters answer the current question.
   - On CPU-only or non-NVIDIA local machines, stop at static/correctness/CPU timeline evidence and name the GPU run needed next.

Valid local conclusions:

| Local evidence | Can support |
|---|---|
| Correctness tests | Functional equivalence for tested shapes/dtypes/tolerances |
| Local benchmark | Relative local before/after on the same machine and stack |
| Local timeline | Launch gaps, syncs, CPU overhead, copies, operator sequence, input stalls |
| Non-target `ncu` | Rough kernel bottleneck class and portable hypotheses |

Invalid local conclusions:

- Final B200/H200/A100 throughput or latency.
- Target SM occupancy, Tensor Core utilization, stall attribution, roofline position, or memory bandwidth ceiling.
- Production capacity numbers.

### 5. Run target GPU profiling when instructed

Only do this when the user asks, the task's definition of done requires it, or local evidence is insufficient for the claim.

Before using the target:

1. Confirm target identity: GPU SKU, count, MIG/partitioning, driver, CUDA, clocks/power state if available, framework versions, repo commit/diff.
2. Confirm the same harness and inputs can run there.
3. Stage inputs/artifacts so results are durable.
4. If the target is Kubernetes/Ray/Kueue, compose with `/gpu-research-run`; if scheduling/admission is the problem on airun, compose with `/airun-triage`.

Target run order:

1. Correctness smoke on target.
2. Benchmark on target.
3. `nsys profile` when timeline/end-to-end/operator scheduling matters.
4. `ncu` when kernel-level counters matter. Narrow to the target kernel where possible to avoid huge, noisy reports.
5. Copy raw reports and exports local for analysis when useful.

Target conclusions must stay scoped to the measured workload shape, dtype, software stack, driver/CUDA version, clocks/power/MIG mode, and command.

### 6. Analyze and iterate

Classify every finding:

| Class | Meaning |
|---|---|
| **Correctness** | Pass/fail or numerical mismatch evidence. |
| **Local result** | Measured local behavior; useful for software structure or local before/after. |
| **Target result** | Measured target-GPU behavior; eligible for hardware-specific claims. |
| **Lead** | Plausible optimization suggested by local profile, code inspection, or non-target counters. |
| **Gotcha** | Benchmark/profiling trap that could invalidate results. |
| **Blocker** | Missing hardware, missing dependency, build failure, unstable test, or incomplete harness. |

Iteration rules:

1. Fix correctness failures before performance changes.
2. If benchmark noise hides the signal, improve methodology before optimizing.
3. If `nsys` shows host gaps, syncs, copies, or data stalls, fix those before micro-optimizing a kernel.
4. If `ncu` shows memory-bound behavior, inspect layout/coalescing/reuse before increasing math.
5. If `ncu` shows occupancy/register/shared-memory pressure, compare tile/block choices with resource usage.
6. Re-run the smallest sufficient validation after each meaningful change.

### 7. Package results

End with a compact result summary:

```markdown
**Scope:** <kernel/operator, shapes, dtype, implementation>
**Hardware provenance:** <local/target GPU, SKU, driver/CUDA, MIG/clocks if relevant>
**Commands:** `<correctness command>`, `<benchmark command>`, `<profile command>`

| Claim | Class | Evidence | Artifact |
|---|---|---|---|
| <what we learned> | Correctness/Local result/Target result/Lead/Gotcha/Blocker | <number/log/profile signal> | `<path>` |

**Next decision:** <ship / keep tuning / run target profile / fix blocker>
```

If raw profiler artifacts or model graphs should be routed to Hermes, invoke `/profiler-handoff` and include the raw report plus summary.

## Anti-patterns

- Optimizing before a correctness oracle exists.
- Reporting async CUDA timings without synchronization.
- Comparing benchmark numbers across machines without hardware/software provenance.
- Treating local profile evidence as target-GPU proof.
- Running `ncu` over an entire model when a narrowed kernel replay would answer the question.
- Spending target GPU time before local build/correctness failures are fixed.
- Flattening GPU compute kernels and Linux host kernels into one workflow.
- Leaving only screenshots or chat summaries instead of raw profiler artifacts.

## Exit criteria

- Kernel scope, claim class, and hardware target are stated.
- Correctness contract exists, or the blocker says exactly why it cannot.
- Build/static validation ran with existing project tooling when applicable.
- Correctness ran locally or on target, or target-only requirement/blocker is explicit.
- Benchmark/profile commands and artifacts are captured.
- Local vs target-GPU conclusions are clearly separated.
- If the user instructed a target GPU run, it was attempted and either completed with artifacts or ended with an evidence-backed blocker.

## Copilot Hub artifact handoff

When this skill produces a durable artifact that Hermes should see (plan, memo, report, benchmark CSV, chart/image, PDF/HTML, or log), emit the explicit Copilot Hub artifact contract after saving it:

```bash
copilot-hub artifact-handoff "$TMUX_SESSION" \
  --title "<kernel/profile title>" \
  --summary "<what Hermes should know>" \
  --intent "<why this artifact exists / requested next action>" \
  --audience hermes \
  --priority normal \
  --artifact path/to/summary.md \
  --artifact path/to/raw-profile-or-metrics
```

Use `--artifact` once per file. If `copilot-hub` or `$TMUX_SESSION` is unavailable, do not fail the skill; mention that the local artifact is the source of truth.
