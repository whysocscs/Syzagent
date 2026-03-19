# Syzagent

`Syzagent` is a minimal, runnable bundle around `source/syzdirect/Runner/run_hunt.py`.

The goal is not to preserve the entire original SyzDirect research tree. The goal is to keep only the pieces required to take a fresh Linux kernel CVE and run a practical hunting pipeline:

1. resolve or accept the target CVE metadata
2. prepare kernel source at the vulnerable or fixed revision
3. build LLVM bitcode
4. analyze kernel interfaces
5. analyze the target point
6. instrument distance
7. fuzz directly, or fuzz with an iterative agent loop

This repository is intended for "new CVE in, automated hunt out" workflows rather than replaying only a fixed paper dataset.

## What This Project Is

The original SyzDirect toolchain is split across several components:

- `syzdirect_function_model`: kernel interface extraction
- `syzdirect_kernel_analysis`: target point analysis
- `syzdirect_fuzzer`: Syzkaller-derived fuzzing side

`run_hunt.py` is the wrapper that connects these pieces into a single pipeline for a new CVE. This repository keeps the wrapper and the minimal binaries/configuration it needs, then adds practical automation around the fragile points that usually break when you try to use SyzDirect on a CVE that was not pre-curated in `dataset.xlsx`.

In practice, `Syzagent` is:

- a thin orchestration layer over the official SyzDirect binaries
- a CVE-oriented runner
- a restartable multi-stage pipeline
- a place to insert LLM-assisted syscall selection and iterative fuzz-template improvement

It is not:

- the full original research repository
- a polished production framework
- a replacement for Syzkaller itself

## Repository Layout

Included in this trimmed repository:

- `source/syzdirect/Runner/`
  - `run_hunt.py`: main entrypoint
  - `Fuzz.py`: fuzz launch logic
  - `Config.py`: runner config helpers
  - `cve_resolver.py`: CVE -> commit/function/file auto-resolution
  - `SyscallAnalyze/`: target-point and interface post-processing logic
- `source/syzdirect/kcov.diff`
- `source/syzdirect/bigconfig`
- `source/syzdirect/template_config`
- `source/syzdirect/syzdirect_fuzzer/`
  - `bin/`
  - `executor/`
  - `sys/`
  - `.descriptions`
- `source/syzdirect/syzdirect_function_model/build/lib/`
  - `interface_generator`
  - `configs/`
- `source/syzdirect/syzdirect_kernel_analysis/build/lib/`
  - `target_analyzer`
  - `configs/`

Excluded on purpose:

- old datasets
- old experiment outputs
- large ad hoc workdirs
- unrelated research or debugging notes

## High-Level Flow

The main pipeline in `run_hunt.py` is:

```text
source -> bitcode -> analyze -> target -> distance -> fuzz
```

The corresponding implementation is the `NewCVEPipeline` class.

### Stage 1. `source`

Prepare a Linux tree for the target commit.

What happens:

- clone from a local template repo if provided, otherwise clone `torvalds/linux`
- verify that the requested commit exists locally
- if it is missing, fetch the commit first
- check out the target revision
- apply `kcov.diff`
- relax some build constraints to make kernel building more automation-friendly

This matters because a common failure mode for fresh CVEs is asking Git to `checkout` a commit object that was never fetched. The runner now validates and fetches first instead of failing late with a vague checkout error.

### Stage 2. `bitcode`

Build the kernel with LLVM bitcode emission enabled.

What happens:

- generate an `emit-llvm.sh` wrapper
- copy the chosen kernel config, usually `bigconfig`
- append required build options
- run `olddefconfig`
- build the kernel and emit `.llbc` side products

The key expected artifact is:

- `arch/x86/boot/bzImage`

### Stage 3. `analyze`

Run the SyzDirect interface extraction step.

What happens:

- generate syzkaller feature signatures if needed
- run `interface_generator`
- build the kernel-to-syzkaller mapping (`kernelToSyscall.json`)

This is one of the most fragile stages on newer kernels. The wrapper adds a retry-and-quarantine flow for bad `.llbc` files:

- if `interface_generator` crashes or reports a bad `.llbc`
- the suspected file is moved into a quarantine directory outside the analysis tree
- a skip manifest is recorded
- the interface stage is retried

This keeps one bad bitcode file from repeatedly poisoning the whole run.

### Stage 4. `target`

Run target point analysis and prepare fuzzing inputs.

What happens:

- write the positive target point list for the requested function
- generate the SyzDirect target-relation map
- run `target_analyzer`
- ensure `target_functions_info.txt` exists
- invoke `PrepareForFuzzing`
- if that produces no usable callfile, fall back to generated inputs

The wrapper now defends against a subtle but dangerous failure mode: the stage may appear to complete, but `target_functions_info.txt` can still be empty and the fuzz stage may effectively do nothing. The runner now forces a minimal `0 <function> <file>` entry when necessary and verifies that required fuzz inputs exist before launching fuzzing.

### Stage 5. `distance`

Build the distance-instrumented kernel.

What happens:

- choose the target function index
- locate the computed distance directory
- generate the required `Makefile.kcov` wiring
- rebuild the kernel with distance instrumentation
- copy out instrumented `bzImage` and optionally `vmlinux`

### Stage 6. `fuzz`

Launch SyzDirect fuzzing.

What happens:

- verify `tfinfo`, `callfile`, and `bzImage` before starting
- regenerate missing callfiles when possible
- normalize syscall names so they match actual syzkaller names
- create per-target Syzkaller manager config
- run `syz-manager`

The runner now uses dynamic HTTP port allocation in normal fuzz mode so multiple runs do not collide on a fixed port.

## Modes of Operation

There are three useful modes.

### 1. `new`

The main path for a fresh CVE.

Example:

```bash
cd source/syzdirect/Runner

python3 run_hunt.py -j 8 -uptime 24 new \
  --cve CVE-2026-23126 \
  --commit b97d5eedf4976cc94321243be83b39efe81a0e15 \
  --function nsim_bpf_destroy_prog \
  --file drivers/net/netdevsim/netdevsim.c \
  --agent-rounds 3 \
  --agent-uptime 6
```

You can provide all metadata manually, or provide only the CVE and let the resolver try to fill the rest.

### 2. `dataset`

Compatibility path for the original SyzDirect dataset-based workflow.

Example:

```bash
python3 run_hunt.py dataset -dataset dataset.xlsx -j 8 \
  prepare_for_manual_instrument compile_kernel_bitcode \
  analyze_kernel_syscall extract_syscall_entry \
  instrument_kernel_with_distance fuzz
```

### 3. `fuzz`

Fuzz prebuilt targets only.

This is useful when the source/bitcode/analysis work is already complete and you only want to iterate on fuzzing.

Example:

```bash
python3 run_hunt.py fuzz --targets 0 -uptime 12 --agent-rounds 5
```

## How CVE Resolution Works

If `--commit`, `--function`, or `--file` is missing in `new` mode, `run_hunt.py` calls `cve_resolver.py`.

The resolver tries to find the fix commit from these sources, in order:

1. Linux kernel CVE list: `git.kernel.org/pub/scm/linux/security/vulns.git`
2. NVD API: `services.nvd.nist.gov`
3. GitHub commit search: `api.github.com`

Then it:

1. fetches the patch for the fix commit
2. parses the patch to infer the most relevant changed `.c` file
3. extracts the most likely function from hunk headers
4. uses `fix_commit~1` as the vulnerable checkout revision

Why `fix_commit~1` instead of the `Fixes:` tag?

- the `Fixes:` tag often points to the original introducing commit, which may be much older
- older commits are harder to fetch in shallow workflows
- `fix~1` is the most recent vulnerable state and is the right default for quick reproduction

### Example: auto-resolve from CVE only

```bash
python3 run_hunt.py -j 8 -uptime 24 new --cve CVE-2025-99999
```

This will attempt to infer:

- `commit`
- `function`
- `file`

Then the pipeline continues normally.

## Pre-Fix vs Post-Fix Runs

By default, `new --cve ...` runs in "reproduce on vulnerable kernel" mode:

- checkout is the vulnerable revision, usually `fix_commit~1`

If you pass `--verify-patch`, the checkout switches to the fixed commit instead:

- checkout is the actual fix commit

This is useful when you want to confirm that the bug no longer reproduces after the patch.

Example:

```bash
python3 run_hunt.py new \
  --cve CVE-2025-99999 \
  --verify-patch
```

## LLM-Assisted Syscall Selection

One of the main problems in one-day CVE hunting is not just finding the right target function. It is generating a decent initial callfile that gives the fuzzer a realistic path toward the vulnerable state.

`run_hunt.py` has two layers for this.

### Layer 1. Heuristic syscall guessing

`guess_syscalls(file_path)` maps source-file paths to rough syscall families.

Examples:

- `net/vmw_vsock/...` -> `connect$vsock_stream`
- `kernel/bpf/...` -> `bpf$PROG_LOAD`
- generic `fs/...` -> `openat`

This is cheap, deterministic, and works as a fallback.

### Layer 2. LLM syscall suggestion

`llm_analyze_cve()` builds a prompt using:

- CVE ID
- kernel commit
- target function
- file path

It then asks the local `claude` CLI for JSON of the form:

```json
{
  "syscalls": [
    {
      "Target": "name$variant",
      "Relate": ["setup1", "setup2"]
    }
  ]
}
```

The LLM is expected to propose:

- 1 to 3 target syscalls
- 3 to 6 related setup syscalls
- exact syzkaller syscall names

If the LLM call fails, times out, or returns bad JSON, the runner falls back to the heuristic mapping.

### Syscall normalization layer

Even when the LLM or heuristics are directionally correct, the names may not exactly match the syzkaller descriptions shipped in `syzdirect_fuzzer/sys/linux/gen/amd64.go`.

The runner therefore normalizes generated names before fuzzing. This absorbs common mismatches such as:

- `io_uring_register` -> a concrete syzkaller variant
- `connect$vsock` -> `connect$vsock_stream`
- `bind$vsock` -> `bind$vsock_stream`
- MPTCP-adjacent names into actual supported socket and operation combinations

This step is important because otherwise the fuzz stage can appear to run while the target call is silently unknown or disabled.

## How to Stop, Resume, and Restart From the Middle

The pipeline is intentionally stage-oriented.

You can restart from a specific stage with `--from-stage`:

```bash
python3 run_hunt.py new \
  --cve CVE-2025-99999 \
  --commit abc123 \
  --function vuln_func \
  --file net/core/sock.c \
  --from-stage distance
```

Valid stages are:

- `source`
- `bitcode`
- `analyze`
- `target`
- `distance`
- `fuzz`

This is useful when:

- source checkout and build are already done
- analysis succeeded and you only want to rebuild distance
- fuzzing failed and you only want to re-enter the last stage

## State Files

Each `new` run writes a state manifest under:

```text
<workdir>/state/case_0.json
```

The state file records:

- CVE
- commit
- function
- file
- config
- last stage
- phase
- key artifact paths
- last error, if any

Typical `phase` values are:

- `initialized`
- `running`
- `completed`
- `failed`

This is useful for:

- resumability
- debugging failed runs
- external orchestration that wants to monitor pipeline progress

## Agent Loop: Using LLM/Agents to Strengthen Fuzzing Mid-Run

The optional agent loop is the part that makes this repository more than a one-shot launcher.

When enabled, the flow becomes:

```text
fuzz -> assess health -> triage failure -> enhance callfile -> fuzz again
```

You enable it with:

```bash
python3 run_hunt.py new \
  --cve CVE-2025-99999 \
  --commit abc123 \
  --function vuln_func \
  --file net/core/sock.c \
  --agent-rounds 3 \
  --agent-uptime 6
```

Meaning:

- run up to 3 rounds
- each round fuzzes for 6 hours
- between rounds, analyze the result and mutate the callfile if needed

### Round structure

For each round:

1. run fuzzing and capture manager logs and metrics
2. check whether crashes were found
3. assess health from execution and coverage movement
4. classify failure into an R-class
5. call the appropriate agent to improve the callfile
6. write the enhanced callfile and continue to the next round

### Health assessment

The loop scores the round using:

- execution growth
- coverage growth
- crash count
- fatal log markers such as "all target calls are disabled"

The round is then treated as one of:

- `healthy`
- `stagnant`
- `fatal`

### Triage classes

The current logic uses three broad failure classes.

#### `R1`

Meaning:

- wrong syscall family
- target call never actually becomes runnable
- all target calls disabled
- nothing meaningful executed

Typical symptom:

- zero progress or immediate disablement

Action:

- broaden or replace related syscalls

#### `R2`

Meaning:

- syscall family may be right
- parameter or object generation is wrong

Typical symptom:

- repeated `EINVAL` or `EFAULT`

Action:

- synthesize better object setup or parameter scaffolding

#### `R3`

Meaning:

- executions happen
- but coverage or depth toward the target stalls
- likely missing dependency/context syscalls

Typical symptom:

- lots of execution, no meaningful movement

Action:

- augment the syscall chain with more related setup/context calls

### Which agent is called

Current mapping:

- `R1` or `R3` -> `RelatedSyscallAgent`
- `R2` -> `ObjectSynthesisAgent`

The agents are imported from `source/agent/`, which is expected to exist in a larger local environment. This trimmed repository documents the interface points, but the actual agent implementations are not bundled here.

### What "stop in the middle and strengthen fuzzing" means

Operationally, this repository supports that idea in two ways:

1. structured stopping between stages via `--from-stage`
2. structured stopping between fuzz rounds via the agent loop

That means you can:

- stop after `target` and inspect or replace the callfile manually
- stop after `distance` and only relaunch fuzz later
- let the agent loop run multiple short fuzz rounds instead of one long blind run
- inspect the enhanced callfile between rounds and decide whether to continue

This is a better fit for CVE hunting than a single 24-hour fuzz launch with no feedback loop.

## Fuzz Inputs and Why They Matter

Before fuzzing, the runner validates:

- `target_functions_info.txt`
- `inp_<xidx>.json` callfiles
- instrumented `bzImage`

If these are missing or empty, the runner now tries to repair them instead of silently continuing into a useless fuzz phase.

This matters because one of the easiest ways to waste a day is:

- target analysis prints something that looks plausible
- `tfinfo` ends up empty
- `syz-manager` never really runs the intended target call
- logs keep moving but the target path is effectively dead

The wrapper explicitly guards against this.

## Runtime Prerequisites

At minimum, you need:

- Python 3
- a Linux build environment capable of compiling kernels with Clang/LLVM
- QEMU/KVM-compatible runtime assets
- a VM image
- an SSH private key usable by the guest

Useful environment variables:

```bash
export SYZDIRECT_RUNTIME=/path/to/runtime
export SYZDIRECT_VM_IMAGE=/path/to/stretch.img
export SYZDIRECT_SSH_KEY=/path/to/stretch.id_rsa
```

Depending on your environment, you may also need:

- a local Linux repo template for faster cloning
- the `claude` CLI if you want LLM-assisted syscall generation
- network access for CVE auto-resolution

## Recommended Workflows

### Fast path: CVE only

```bash
python3 run_hunt.py -j 8 -uptime 24 new --cve CVE-2025-99999
```

Use this when:

- the CVE is in kernel.org or NVD
- the patch is parseable
- you want the runner to infer as much as possible

### Controlled path: manual commit/function/file

```bash
python3 run_hunt.py -j 8 -uptime 24 new \
  --cve CVE-2025-99999 \
  --commit abc123 \
  --function vuln_func \
  --file net/core/sock.c
```

Use this when:

- the CVE resolver found the wrong function
- the fix commit is non-mainline
- the patch touches multiple files and you want to force a target

### Analysis-first path

```bash
python3 run_hunt.py -j 8 new \
  --cve CVE-2025-99999 \
  --commit abc123 \
  --function vuln_func \
  --file net/core/sock.c \
  --from-stage target
```

Use this when:

- source and bitcode already exist
- you are iterating on target/callfile generation

### Fuzz-strengthening path

```bash
python3 run_hunt.py -j 8 -uptime 6 new \
  --cve CVE-2025-99999 \
  --commit abc123 \
  --function vuln_func \
  --file net/core/sock.c \
  --agent-rounds 4 \
  --agent-window 300 \
  --agent-uptime 6
```

Use this when:

- a single long fuzz run is too opaque
- you want iterative refinement of the callfile
- you expect the initial syscall chain to be incomplete

## Current Practical Improvements Over a Plain SyzDirect Script

This wrapper already adds several defensive behaviors for fresh CVEs:

- commit existence check before checkout
- fetch missing commit objects automatically
- clearer stage failure when a commit cannot be resolved locally
- `interface_generator` retry with bad `.llbc` quarantine
- persistent skip manifest for bad analysis files
- auto-generation of missing `template_config`
- auto-repair of missing or empty `target_functions_info.txt`
- regeneration or replication of missing callfiles
- syscall name normalization against real syzkaller names
- dynamic HTTP port assignment in normal fuzz mode
- state manifest recording for restart/debugging

## Limitations

Important limitations to understand up front:

- this repository still depends on shipped SyzDirect binaries rather than rebuilding every component from source
- CVE auto-resolution is best-effort; some CVEs still need manual `--commit`, `--function`, or `--file`
- LLM syscall selection is only as good as the local model/tooling you provide
- some kernels still fail in ways the wrapper cannot repair, especially low-level bitcode or runtime/coverage issues
- the agent loop expects external agent implementations under `source/agent/`
- normal fuzz mode uses dynamic HTTP ports, but other parts of a larger orchestration stack may still need their own port hygiene

## Minimal Quick Start

```bash
git clone https://github.com/whysocscs/Syzagent.git
cd Syzagent/source/syzdirect/Runner

export SYZDIRECT_RUNTIME=/path/to/runtime
export SYZDIRECT_VM_IMAGE=/path/to/stretch.img
export SYZDIRECT_SSH_KEY=/path/to/stretch.id_rsa

python3 run_hunt.py -j 8 -uptime 24 new \
  --cve CVE-2026-23126 \
  --commit b97d5eedf4976cc94321243be83b39efe81a0e15 \
  --function nsim_bpf_destroy_prog \
  --file drivers/net/netdevsim/netdevsim.c
```

If you already have artifacts and only want to re-enter fuzz:

```bash
python3 run_hunt.py -j 8 -uptime 12 new \
  --cve CVE-2026-23126 \
  --commit b97d5eedf4976cc94321243be83b39efe81a0e15 \
  --function nsim_bpf_destroy_prog \
  --file drivers/net/netdevsim/netdevsim.c \
  --from-stage fuzz
```

## Summary

If you only remember one thing, it should be this:

`Syzagent` is a CVE-oriented wrapper around SyzDirect that tries to turn a raw, newly disclosed Linux kernel CVE into a restartable, fuzzable hunting workflow, with optional LLM- and agent-assisted refinement when the first callfile is not good enough.
