# Repository Guidelines

## Project Overview

This repository ships infrastructure for the **ISPD 2027 MCMM Post-CTS Buffering and Sizing Contest**: benchmarks, the ASAP7 platform, evaluation/scoring scripts, and a tool container. It is not an application or a contestant optimizer. Most implementation work belongs in `evaluator/`. Public designs are `aes` (top `aes_cipher_top`) and `jpeg` (top `jpeg_encoder`).

## Architecture & Data Flow

The evaluator is a **synchronous, file-backed pipeline**, not a service or framework:

1. `evaluator/run_submission.py` runs the contestant command, measures wall time, and requires `tool/<design>.def` and `tool/<design>.v`. It enforces an 18,000-second limit for both the tool and total flow, killing remaining process-group workers when a command finishes.
2. `evaluator/eval.sh` delegates to `internal/run.py`, which resolves tools and fixes evaluation settings. It disables Resizer and strips inherited overrides such as `SPEF_PREFIX` and `INSPECT_DB`.
3. `internal/core_run.sh` orchestrates OpenROAD through `evaluation.tcl`: baseline snapshot, Verilog check, physical evaluation, audit/formal proofs, then fresh-process final timing via `refresh_timing.py`.
4. SPEF validation and `metrics.py` produce `summary.csv`; `legality.py` seals evaluated artifacts with hashes. The submission harness additionally binds measured runtimes to the result.
5. `evaluator/score.py` independently verifies certification and computes the score from candidate metrics, packaged baseline/Resizer references, and runtimes. Higher is better.

`EVALUATION_COMPLETE` means evaluation finished; `SUBMISSION_COMPLETE` additionally means measured submission validation finished. Neither runs scoring. Inter-stage contracts are environment variables plus JSON/CSV/TSV/DEF/Verilog files. Schema or artifact changes require tracing producers, audit/refresh consumers, metrics, certificate sealing/verification, and scoring.

## Key Directories

- `evaluator/internal/`: Python orchestration/validation and OpenROAD Tcl extraction.
- `evaluator/tests/`: isolated Python regression tests.
- `evaluator/config/`: scoring policy; weights are explicitly **provisional**, not final contest weights.
- `evaluator/reference_results/<design>/`: packaged baseline/Resizer metrics and reference runtimes used for normalization and verification.
- `evaluator/third_party/displacement/`: reused ISPD26 displacement implementation and equivalent-cell mapping; reuse rather than duplicate.
- `benchmarks/<design>/`: DEF/Verilog inputs, corner SDCs, `benchmark.json`, `geometry.json`, and `mcmm.tcl`.
- `platform/asap7/`: LEF/Liberty data and platform utilities; `docker/`: tool-image construction and entrypoint.

## Development Commands

Run from the repository root. Every evaluation/run directory and score output file must be **new**, even if an existing directory is empty.

```bash
# Fast Python regression suite; no EDA executables required.
python3 -B -m unittest discover -s evaluator/tests -v
python3 -B -m unittest evaluator.tests.test_score -v

# Baseline integration smoke; requires the full EDA toolchain.
bash evaluator/eval.sh aes /tmp/aes-baseline

# Evaluate an existing candidate; no contestant runtime measurement or scoring.
bash evaluator/eval.sh aes /tmp/aes-evaluation \
  --def /path/to/aes.def --verilog /path/to/aes.v

# Run contestant tool, measure runtime, validate, and evaluate.
python3 evaluator/run_submission.py aes /tmp/aes-run -- bash /path/to/run.sh

# Score the completed measured submission separately.
python3 evaluator/score.py \
  --baseline evaluator/reference_results/aes/baseline.csv \
  --candidate /tmp/aes-run/evaluation/summary.csv \
  --reference evaluator/reference_results/aes/resizer.csv \
  --runtimes /tmp/aes-run/evaluation/runtimes.json \
  --output /tmp/aes-run/evaluation/score.json
```

Replace `aes` with `jpeg` throughout for JPEG. The contestant command receives appended positional arguments `<input_dir> <platform_dir> <output_dir> <top_module>`; outputs are named by **design**, not top module. Preserve the `--` separator and support paths containing spaces.

There is no repository-wide application build, package-install, lint, or formatter command configured. For edited shell scripts, use `bash -n path/to/script.sh`; do not introduce a new toolchain merely to format existing code.

## Code Conventions & Common Patterns

- Follow the touched file's style. Python uses four-space indentation, `snake_case`, uppercase constants/environment keys, `argparse`, `pathlib.Path`, and standard-library JSON/CSV/subprocess utilities. Avoid unrelated formatting rewrites.
- Keep orchestration blocking and sequential. There is no async or dependency-injection framework; existing seams are explicit function/path arguments and executable environment variables. State lives in artifact files and the process environment, not a shared service layer.
- Fail closed: validation raises `ValueError` or `legality.Illegal`; CLI boundaries use argparse errors or nonzero exits. Use checked subprocesses, preserve diagnostic logs, and never publish success artifacts after failed validation.
- Preserve fresh/exclusive output creation (`exist_ok=False`, `open("x")`). Timing refresh deliberately archives/replaces reports; do not generalize that exception into overwrite support.
- Bash uses `set -euo pipefail` and quoted paths. Tcl uses small `proc`s, `$::env(...)`, `file join`, dictionaries/lists, and direct OpenROAD/OpenSTA/OpenDB APIs—not an ORFS application layer.
- Extract QoR from dedicated evaluator data/reports, **not console logs**. Keep the fresh-process final timing reload. Timing/ERC source values use ps/fF; summary slack uses ns; displacement converts DBU to µm; power uses watts.
- Preserve legality invariants: clock tree/sinks, protected instances, die/I/O/PDN/blockages and contest timing/platform inputs remain fixed. Allowed data buffer/inverter changes must preserve polarity; other changes are limited to approved variants/equivalent-pin swaps.
- Displacement includes only surviving original eligible movable cells, using baseline eligibility. Scoring normalizes displacement against the Resizer reference, not the baseline; shared physical penalties are counted once across corners.

## Important Files

- `evaluator/internal/legality.py`: transform rules, structural checks, `finalize`, and `verify_certificate`; certification binds artifacts and measured runtime.
- `evaluator/internal/audit.py`: input provenance and Kepler Formal equivalence proofs.
- `evaluator/internal/metrics.py`: dedicated-report parsing, displacement, units, and summary CSV production.
- `evaluator/config/scoring_config.json`: weights, corner tolerances, and unit-specific epsilons; retain strict validation and provisional status.
- `benchmarks/<design>/benchmark.json` and `mcmm.tcl`: design identity and BC/TC/WC scenario mapping. Active constraints are `sdc/{BC,TC,WC}.sdc`, not the platform's generic `constraints.sdc`.
- `platform/asap7/util/manifest.json`, `libraries.tcl`, and `setRC.tcl`: hashed platform inputs, generated library mapping, and shared nominal RC model.
- `evaluator/README.md`: submission/output contract. Update it when changing CLIs, artifacts, certification, or executable selection; update `docker/README.md` for container workflow changes.

## Runtime/Tooling Preferences

Use Linux/POSIX, Python 3, Bash, and OpenROAD Tcl. Python evaluator/tests use the standard library; no Node/Bun or project package-manager workflow is required. Full evaluation needs `openroad` and `kepler-formal` on `PATH`, or `OPENROAD_EXE` and `KEPLER_FORMAL_EXE`.

The supported local environment is **`udxs/ispd27:v1-fixed`**. The published `v1` image has known Kepler/OpenROAD-path defects; current Docker source contains the repairs. Mount this repository: the image supplies tools, not contest inputs.

```bash
docker run -it --rm --user "$(id -u):$(id -g)" -e HOME=/tmp \
  -v "$PWD":/workspace/ISPD27-Contest -w /workspace/ISPD27-Contest \
  udxs/ispd27:v1-fixed
```

`-it` is required by the entrypoint's `/dev/tty` usage. Running as your UID avoids root-owned outputs. Bypassing the entrypoint also bypasses its Conda/tool exports; supply `/usr/local/bin:/install/OpenROAD/bin` on `PATH` and explicit tool executable paths yourself.

## Testing & QA

Tests use standard-library `unittest`: `test_*.py`, `unittest.TestCase`, descriptive `test_*` methods, `TemporaryDirectory` fixtures, and `subTest` for related invalid cases. Keep fixtures isolated from shipped benchmarks; use hand-calculated numeric expectations and assert rejection behavior, including absent success artifacts.

Coverage targets behavior: legality/protection and inversion rules, certificate tampering, ERC parsing, displacement eligibility, score boundaries/units, timeouts/process cleanup, and submission naming/positional contracts. No coverage-percentage gate is configured.

The unit suite does **not** run routing, timing extraction, or real formal proofs; submission tests substitute evaluation stages. For changes affecting those paths, also run the relevant full evaluation in the supported container and inspect `summary.csv`, validation reports, and completion markers. A direct evaluation cannot certify contestant runtime. `--allow-unverified` is explicitly a dummy-score mode, not a workaround for failed certification.
