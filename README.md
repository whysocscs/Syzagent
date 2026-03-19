# Syzagent

Minimal repository for running the `source/syzdirect/Runner/run_hunt.py` workflow without the rest of the original research tree.

## Included

- `source/syzdirect/Runner/`
  - `run_hunt.py`
  - `Fuzz.py`
  - `Config.py`
  - `cve_resolver.py`
  - `SyscallAnalyze/`
- `source/syzdirect/kcov.diff`
- `source/syzdirect/bigconfig`
- `source/syzdirect/template_config`
- `source/syzdirect/syzdirect_fuzzer/`
  - `bin/`
  - `sys/`
  - `executor/`
- `source/syzdirect/syzdirect_function_model/build/lib/`
  - `interface_generator`
  - `configs/`
- `source/syzdirect/syzdirect_kernel_analysis/build/lib/`
  - `target_analyzer`
  - `configs/`

## Usage

Run from `source/syzdirect/Runner`:

```bash
python3 run_hunt.py -j 8 -uptime 24 new \
  --cve CVE-2026-23126 \
  --commit b97d5eedf4976cc94321243be83b39efe81a0e15 \
  --function nsim_bpf_destroy_prog \
  --file drivers/net/netdevsim/netdevsim.c
```

## Runtime Prerequisites

- Python 3
- A Linux kernel source template or network access for cloning `torvalds/linux`
- A working VM image and SSH key
- Environment variables may be overridden if needed:

```bash
export SYZDIRECT_RUNTIME=/path/to/runtime
export SYZDIRECT_VM_IMAGE=/path/to/image.img
export SYZDIRECT_SSH_KEY=/path/to/key
```

## Notes

- This repo intentionally excludes old datasets, workdirs, and local experiment outputs.
- The layout is kept compatible with the current relative paths in `run_hunt.py`.
