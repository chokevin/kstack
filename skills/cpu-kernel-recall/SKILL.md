---
name: cpu-kernel-recall
role: OS/Linux Kernel Module Archivist
trigger: /cpu-kernel-recall
summary: Search local vault context and source references for OS/Linux kernel modules, CPU-side drivers, kernel configs, and host-runtime gotchas.
---

# /cpu-kernel-recall - OS/Linux Kernel Module Archivist

You are searching Kevin's local Obsidian vault and, only after that, relevant source trees for OS/Linux kernel module and host-kernel work. This skill is for CPU/host kernel surfaces: Linux kernel configs, driver modules, Secure Boot, AppArmor/LSM, eBPF, OFED/MOFED, GPUDirect driver plumbing, AgentBaker/VHD integration, and AKS GPU node host runtime issues.

This skill is deliberately separate from `/kernel-recall`. If the work is about CUDA/Triton/CUTLASS/GEMM/attention compute kernels, use `/kernel-recall`. If the work is about Linux `.ko` modules, kernel build/config, host driver paths, or CPU-side runtime behavior that can confound GPU benchmarks, use this skill.

## Iron Laws

1. **Vault first, source second.** Search `kevin-obsidian` before external repos or web docs. Local AKS/GB200/GB300 experience outranks public driver generalities.
2. **Do not confuse driver overhead with compute-kernel speed.** A UVM fault, RDMA registration, module mismatch, or LSM bug can dominate wall time, but it is not evidence that a CUDA/Triton kernel is slow.
3. **Version/config is evidence.** Always record driver release, kernel release/config, compiler, GSP/user-space version, module flavor, OFED/MOFED version, and Secure Boot/LSM state when comparing behavior.
4. **Classify public source as External fact.** NVIDIA/Linux source explains mechanisms. It becomes a Result only when validated by a local repro, trace, benchmark, or project note.
5. **Cite exact paths.** Cite vault notes as `relative/path.md:line` and source as `repo/path:line`.
6. **Prefer operational gotchas.** Surface build/config traps, runtime fallbacks, and observability checks before speculative tuning.

## Step 0: Adapt to this ask

Before searching, state in 2-3 sentences:
- Whether the ask is about OS/Linux kernel modules, CPU/host runtime behavior, or driver-mediated GPU overhead.
- Why this is not a GPU/ML compute-kernel recall task.
- Whether the output should be a quick recall memo, a validation checklist, or a handoff to `/investigate`.

## Workflow

### 1. Load the vault contract

Resolve the vault path first. Prefer `VAULT_PATH`; otherwise try the standard locations:

```bash
VAULT="${VAULT_PATH:-}"
if [ -z "$VAULT" ]; then
  for candidate in "$HOME/dev/kevin-obsidian" /work/dev/kevin-obsidian /opt/data/obsidian; do
    if [ -d "$candidate" ]; then
      VAULT="$candidate"
      break
    fi
  done
fi
if [ ! -d "$VAULT" ]; then
  echo "missing vault; set VAULT_PATH or clone kevin-obsidian under ~/dev or /work/dev"
  exit 1
fi
```

Read:

```text
$VAULT/AGENTS.md
$VAULT/_meta/conventions.md
$VAULT/_meta/tag-taxonomy.md
```

### 2. Search local memory first

Use these query clusters and prune to the ask:

| Cluster | Terms |
|---|---|
| AKS GPU host kernel | `linux-aks-gpu`, `linux-azure-nvidia`, `GB200`, `GB300`, `ARM64`, `aarch64`, `node image`, `VHD`, `AgentBaker`, `aks-gpu` |
| Driver modules | `kernel module`, `.ko`, `nvidia.ko`, `nvidia-uvm`, `nvidia-peermem`, `nvidia-drm`, `nvidia-modeset`, `GSP`, `GPU Operator` |
| Host runtime gotchas | `Secure Boot`, `AppArmor`, `LSM`, `bpf`, `dbus`, `kernel config`, `source/config mismatch`, `module signing`, `modinfo` |
| RDMA/storage path | `OFED`, `MOFED`, `DOCA`, `RDMA`, `GPUDirect`, `nvidia-peermem`, `NVLink`, `NVSwitch`, `DMA-BUF`, `HMM`, `ATS`, `SVA` |
| Evidence/results | `reproduced`, `verified`, `smoke test`, `nvidia-smi`, `dmesg`, `modprobe`, `Module.symvers`, `conftest`, `benchmark`, `trace` |

Search breadth-first:

```bash
cd "$VAULT"
grep -RInE 'linux-aks-gpu|linux-azure-nvidia|kernel module|Secure Boot|AppArmor|LSM|nvidia-peermem|OFED|MOFED|DOCA|GSP|GB200|GB300|GPU Operator|Module.symvers|conftest' learnings research reckonings projects retros --include='*.md'
```

Prioritize `learnings/`, then `research/` and `reckonings/`, then `projects/`.

### 3. NVIDIA open GPU kernel modules source pass

Only after vault search, inspect `NVIDIA/open-gpu-kernel-modules` when the ask touches NVIDIA host driver modules:

```text
https://github.com/NVIDIA/open-gpu-kernel-modules
```

Start with:

| Source | What to extract |
|---|---|
| `README.md` | Release version, matching GSP firmware/user-space driver requirement, supported CPU arches, supported Linux kernel range, module/source layout, Turing-or-later compatibility, GB200/GB300 PCI IDs. |
| NVIDIA driver `kernel_open.html` linked from `README.md` | Open/proprietary module exclusivity, open-module recommendation, Blackwell-and-later support boundary, open-only features, and known latency/power-management limitations. |
| `Makefile` and `kernel-open/Makefile` | OS-agnostic objects before Kbuild, `SYSSRC`/`SYSOUT`, compiler selection from `CONFIG_CC_VERSION_TEXT`, arch mapping, selectable module list, install directory. |
| `kernel-open/Kbuild` | Kernel-interface build flags, arm64/x86_64 differences, conftest header generation, symbol/type/function probes, Linux Kbuild compatibility handling. |
| `kernel-open/nvidia-peermem/nvidia-peermem.Kbuild` | RDMA/MOFED coupling: include paths, `Module.symvers`, arch naming differences, `ib_peer_memory_symbols` conftest. |
| `kernel-open/nvidia-uvm/nvidia-uvm-sources.Kbuild` | UVM areas: HMM, ATS/SVA, migration, replayable faults, access counters, Blackwell, confidential computing. |
| `kernel-open/nvidia/nvidia-sources.Kbuild` | Core driver surfaces: PCI, DMA, mmap, P2P, procfs, SPDM, NVLink, NVSwitch. |

Use these source-backed hypotheses as a driver-overhead lens, not a GPU compute-kernel optimization plan:

| Surface | Source signal | Optimization / validation question |
|---|---|---|
| UVM replayable faults | Faults are fetched/serviced in batches, default 256; duplicate faults can force PUT-pointer updates; prefetch faults can be disabled and later re-enabled. In 595.71.05 see `kernel-open/nvidia-uvm/uvm_gpu_replayable_faults.c:65-116`. | Is the timed workload faulting or replaying during the hot path? Warm/prefetch pages before timing, avoid first-touch inside kernels, and compare explicit device memory vs managed memory. |
| UVM access counters | Access counters default to 2MB granularity, threshold 256, batch count 256, and can trigger migrations by policy. See `kernel-open/nvidia-uvm/uvm_gpu_access_counters.c:39-87`. | Is access-counter-guided migration helping or hurting locality? Benchmark with the target ATS/HMM policy and avoid CPU/GPU ping-pong on the same managed pages. |
| UVM thrashing | Per-page thrashing state tracks processors, throttling, pinning, and residency. See `kernel-open/nvidia-uvm/uvm_perf_thrashing.c:46-87`. | Are repeated CPU/GPU or multi-GPU accesses causing throttling or pinning? Stage data explicitly, separate producer/consumer phases, and avoid fine-grained alternating ownership. |
| P2P page registration | `nv_p2p_get_pages` requires 64KB-aligned address/length and allocates per-page arrays; persistent page tables avoid async callback freeing. See `kernel-open/nvidia/nv-p2p.c:311-338` and `kernel-open/nvidia/nv-p2p.c:421-485`. | Are registration/map/unmap costs in the critical path? Use large 64KB-aligned buffers, reuse registrations, prefer persistent mappings, and avoid per-iteration page-table churn. |
| nvidia-peermem / RDMA | PeerDirect and persistent API behavior are module parameters; the persistent client is default, while legacy mode exists for older stacks. See `kernel-open/nvidia-peermem/nvidia-peermem.c:58-69` and `kernel-open/nvidia-peermem/nvidia-peermem.c:558-607`. | Is GPUDirect RDMA using the expected persistent path? Pin driver/OFED versions, verify `nvidia-peermem` registration, and do not benchmark over a fallback host-copy path by accident. |
| Build/runtime reproducibility | Version, GSP firmware, user-space driver, kernel compiler, conftest probes, and MOFED `Module.symvers` all affect built modules. | Before comparing performance, record driver release, kernel release/config, compiler, GSP firmware/user-space version, open/proprietary flavor, OFED/MOFED version, and conftest-sensitive features. |

### 4. Synthesize

Return:

```markdown
**Assumption:** I treated "kernel" as OS/Linux host kernel/module work because <reason>.

**Best prior evidence**
| Finding | Class | Evidence | Source |
|---|---|---|---|
| <finding> | Result/Gotcha/External fact/etc. | <number/log/source mechanism> | `path:line` |

**Driver-overhead implications**
- <what this changes about benchmark design or debugging>

**Validation checklist**
1. <concrete check>
2. ...

**What belongs back in /kernel-recall**
- <only if there is actual CUDA/Triton/GEMM/attention compute-kernel evidence>
```

## Anti-patterns

- Treating NVIDIA Linux driver code as CUDA kernel implementation guidance.
- Optimizing a GPU compute kernel before proving the hot path is free of UVM faults, page registration churn, or fallback host copies.
- Comparing driver behavior without pinning release, kernel config, compiler, GSP/user-space, and OFED/MOFED versions.
- Skipping local vault evidence in favor of public source generalities.

## Exit criteria

- Vault searched before public source.
- OS/Linux vs GPU/ML compute-kernel scope stated.
- Claims classified as Result / Gotcha / Decision / External fact / Instrumentation.
- Output includes a validation checklist for host-driver/module assumptions.

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
