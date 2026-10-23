# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

The public repository for the **ISPD 2027 Contest: MCMM Post-CTS Buffering and Sizing**. It is not an application — it ships contest *infrastructure*: public benchmarks, the ASAP7 platform, the evaluator/scoring toolchain, and a Docker environment. The "code" you will most likely change is the evaluator under [evaluator/](evaluator/).

Two designs are public: `aes` (top `aes_cipher_top`) and `jpeg` (top `jpeg_encoder`). Six public benchmarks are planned; final scores come from hidden benchmarks.

## Running the container

The supported evaluation environment is the `udxs/ispd27:v1` image — Ubuntu 24.04 with OpenROAD, Yosys, KLayout, Kepler-Formal, and Conda (see [docker/README.md](docker/README.md)). It ships the **tools only**: benchmarks, platform, and evaluator all come from this repo, so mount it.

The published image has two defects, both fixed in [docker/](docker/): `kepler-formal` is never installed (the apt step in [docker/image_init.sh](docker/image_init.sh) lacked `-y` and requested the nonexistent `python3.9-dev`, so the whole install aborted), and [docker/entrypoint.sh](docker/entrypoint.sh) exported `OPENROAD_EXE=/OpenROAD/build/bin/openroad`, which does not exist. Use the repaired image **`udxs/ispd27:v1-fixed`** (same base plus kepler-formal 0.5.0 and the corrected entrypoint), which is built locally on this machine.

```bash
docker run -it --rm \
  -v "$PWD":/workspace/ISPD27-Contest \
  -w /workspace/ISPD27-Contest \
  udxs/ispd27:v1-fixed

# Inside the container, verify the full toolchain (~5 min, output must be a fresh path):
bash evaluator/eval.sh aes /tmp/aes-baseline
```

- `-it` is **required**: [docker/entrypoint.sh](docker/entrypoint.sh) ends with `exec </dev/tty`, so the container fails to start without a TTY. The entrypoint prints the banner, activates Conda base, and drops into a shell with `openroad` and `kepler-formal` on `PATH`.
- If you bypass the entrypoint with `--entrypoint bash`, you must add `/usr/local/bin:/install/OpenROAD/bin` to `PATH` and set `OPENROAD_EXE` / `KEPLER_FORMAL_EXE` yourself.
- The container runs as **root** and writes root-owned files into the mounted repo. Add `--user $(id -u):$(id -g)` (and `-e HOME=/tmp` if Conda complains) to avoid that.
- On networks that block Docker Hub, pull through a registry mirror and tag the result (e.g. `docker tag docker.1ms.run/udxs/ispd27:v1 udxs/ispd27:v1`). The mirror in `/etc/docker/daemon.json` must also actually serve this repo — some public mirrors only proxy an allowlist.
- After `usermod -aG docker $USER`, the group applies only to **new** shells; use a fresh terminal or `sg docker -c '<command>'`.

## Running the evaluator

The evaluator requires **OpenROAD** (`openroad`) and **Kepler Formal** (`kepler-formal`) on `PATH`, or set `OPENROAD_EXE` / `KEPLER_FORMAL_EXE`. The supported environment is the `udxs/ispd27:v1-fixed` container (see [docker/](docker/)). Unit tests do not need these tools.

```bash
# Full flow: run a submitted tool, measure runtime, validate, evaluate
python3 evaluator/run_submission.py aes /tmp/aes-run -- bash /path/to/run.sh

# Score a completed run
python3 evaluator/score.py \
  --baseline evaluator/reference_results/aes/baseline.csv \
  --candidate /tmp/aes-run/evaluation/summary.csv \
  --reference evaluator/reference_results/aes/resizer.csv \
  --runtimes /tmp/aes-run/evaluation/runtimes.json \
  --output /tmp/aes-run/evaluation/score.json

# Evaluate an existing DEF/Verilog pair (no tool run, no runtime, no score)
bash evaluator/eval.sh aes /tmp/aes-evaluation --def /path/sub.def --verilog /path/sub.v
```

Swap `aes` for `jpeg` everywhere. The tool command receives positional args `<input_dir> <platform_dir> <output_dir> <top_module>` and must write `<output_dir>/<design>.def` and `<output_dir>/<design>.v` — **named by design, not top module**. See [evaluator/README.md](evaluator/README.md) for the full output contract and environment variables.

## Tests

```bash
python3 -B -m unittest discover -s evaluator/tests -v      # all 50 tests
python3 -B -m unittest evaluator.tests.test_score -v       # one module
python3 -B -m unittest evaluator.tests.test_score.ScoreTests.test_identical_within_tolerances_scores_zero  # one test
```

## Architecture

The evaluator is a pipeline of small standalone scripts, not a library. Data flows through files in a fresh output directory; each stage writes JSON/CSV that the next stage consumes.

- [evaluator/run_submission.py](evaluator/run_submission.py) — outermost harness. Creates the run dir, exports paths as env vars, runs the contestant command, then evaluates via `internal/run.py`, records wall-clock in `runtimes.json`, and seals results with `legality.finalize`.
- [evaluator/eval.sh](evaluator/eval.sh) → [evaluator/internal/run.py](evaluator/internal/run.py) — evaluates one DEF/Verilog pair. `run.py` sets **all** physical/timing settings as env vars (corners, layers, route params, resizer off) and hard-strips inherited overrides like `SPEF_PREFIX`, so a run is fully determined by the repo, not the caller's environment.
- [evaluator/internal/core_run.sh](evaluator/internal/core_run.sh) — stage runner that drives OpenROAD through [evaluation.tcl](evaluator/internal/evaluation.tcl) (baseline snapshot, Verilog check, main evaluation), then `refresh_timing.py`, `check_spef.py`, `metrics.py`, and `audit.py`.
- [evaluator/score.py](evaluator/score.py) — implements the PDF §3.3.1 formula. Metrics are normalized against `baseline.csv` (setup/power/ERC) and `resizer.csv` (displacement); runtimes against `runtimes.json`. Higher is better. Requires a verified legality certificate unless `--allow-unverified` (dummy score).
- [evaluator/internal/legality.py](evaluator/internal/legality.py) — the rule engine: parses DEF/Verilog snapshots and enforces allowed transforms (data buffers/inverters may be added/removed/resized with polarity preserved; other instances keep names and may only move to approved size/VT variants or equivalent-pin swaps; die, I/O, PDN, blockages, clock tree, clock sinks, protected cells fixed). `finalize`/`verify_certificate` hash the evaluated artifacts so scoring cannot use outputs modified after evaluation.

Supporting pieces: [metrics.py](evaluator/internal/metrics.py) (parses QoR from the DB, never the console), [audit.py](evaluator/internal/audit.py) (input digests + `kepler-formal` equivalence proofs), [config/scoring_config.json](evaluator/config/scoring_config.json) (weights/tolerances; `status: provisional`, dummy weights until after Alpha submission), [reference_results/](evaluator/reference_results/), and [third_party/displacement/](evaluator/third_party/displacement/) (reused ISPD26 helper).

## Benchmarks and platform

Each [benchmarks/<design>/](benchmarks/) holds `input.def`, `input.v`, `sdc/{BC,TC,WC}.sdc`, `mcmm.tcl` (scenario→SDC mapping), `benchmark.json` (periods, uncertainties, top module, paths), and `geometry.json`. [platform/asap7/](platform/asap7/) provides LEF, NLDM Liberty, and `util/{libraries.tcl,setRC.tcl,manifest.json,constraints.sdc}`. SDCs, libraries, RC settings, and the clock tree must stay unchanged — the evaluator enforces this.

## Gotchas

- Output/run directories **must not already exist** — every stage uses `exist_ok=False` / refuses to overwrite. Pick a fresh path each run.
- The flow limit is `LIMIT_SECONDS = 18000` (5 hours) for both tool and total flow, enforced in `run_submission.py`.
- Evaluation never runs OpenROAD Resizer (`RUN_RESIZER=0`). The Resizer only appears as a reference result for normalization.
- `run_submission.py`'s `--` separator is required; everything after it is the contestant command.
- Do not parse QoR from stdout/logs — metrics come from dedicated evaluator outputs.
