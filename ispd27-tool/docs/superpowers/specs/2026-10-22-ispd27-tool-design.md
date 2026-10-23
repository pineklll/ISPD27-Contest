# ISPD27 Submission Tool — A-lite Design

Date: 2026-10-22
Revision: A-lite, following the supplied architecture review
Status: revised draft for approval; no implementation authorized by this document alone

## 1. Intent and scope

Build a focused contestant optimizer for **ISPD 2027: MCMM Post-CTS Buffering and Sizing**. It consumes the supplied legalized post-CTS DEF/Verilog and timing scenarios, makes legal ECOs, and emits a consistent, legalized DEF/Verilog pair with measured setup improvement.

Priority: a correct end-to-end optimization loop, then timing-aware algorithms, then infrastructure justified by profiling. Preserve the useful Python/Tcl boundary without building a general ECO framework.

The requested architecture revision is **A-lite**: OpenROAD is the only mutable design authority; Python owns decisions and immutable, versioned observations. The initial submission path supports sizing, then buffering/hold repair and batching. Equivalent-pin swapping remains an explicitly staged P4 capability. Native/ML backends are deferred, not represented by empty directories or generic interfaces.

## 2. Contest contract and nonnegotiable correctness

### 2.1 Invocation, inputs and outputs

```text
run.sh <input_dir> <platform_dir> <output_dir> <top_module>
```

Exactly four positional arguments. The launcher may run from any cwd; paths containing spaces must work. Resolve shipped code from the launcher location, never the contest repository cwd.

Inputs include `input.def`, `input.v`, `benchmark.json`, `mcmm.tcl` and scenario SDCs. The public release uses `sdc/{BC,TC,WC}.sdc`; those names are examples, not algorithm constants. The platform supplies LEF/Liberty plus `util/libraries.tcl`, `util/setRC.tcl` and `util/manifest.json`.

The launcher supplies `INPUT_DEF`, `INPUT_VERILOG`, `OUTPUT_DEF`, `OUTPUT_VERILOG`, `BENCHMARK_DIR`, `MCMM_CONFIG` and `PLATFORM_DIR`. The output directory exists and is empty on entry. Required output filenames come from `OUTPUT_DEF` and `OUTPUT_VERILOG`; never reconstruct them from the top module or directory basename. Public filenames use design names (`aes`, `jpeg`), not top-module names.

Use the supplied platform argument/environment, not repository-relative paths in benchmark metadata. Conflicting invocation/environment identities fail clearly. Work files live under a private subdirectory of this fresh output directory; required outputs must not already exist.

### 2.2 Packaging and runtime

Submit `TeamID_solution.zip` containing `setup.sh`, `run.sh` and required runtime code/data; record its POSIX `cksum`. `setup.sh` is idempotent and must finish within two hours. `run.sh` operates offline with Python 3.9+ standard library and the supplied OpenROAD executable. No pip installation or venv is needed for this delivery.

Resources: 1 NVIDIA A100 (40 GB), 16 physical cores/threads, 64 GB RAM, 200 GB disk. Neither GPU nor ML packages are required. Tool runtime and total-flow runtime are each limited to `18000` seconds; external evaluation consumes part of that same flow allowance.

Development target: `udxs/ispd27:v1-fixed`, OpenROAD commit `8443f6ff398e3c4e06cb65a05a0abf22734ad345`. Resolve `OPENROAD_EXE` or `openroad`; do not assume the defective published `v1` image has working tool paths. Kepler Formal is a development/evaluator dependency, not a production optimizer dependency.

### 2.3 Allowed changes and authorization

- Insert/remove/replace data-path buffers and inverters while preserving end-to-end source identity and inversion parity.
- Resize/change VT of other cells only within an officially approved equivalent-master group, with matching pin interfaces and Liberty functions.
- Swap input pins only through a Boolean-function symmetry; unsupported sequential/tristate semantics are not guessed.
- Move eligible cells only to legal placements: correct rows/sites/orientation, no overlaps or blocked-region violations, inside the permitted placement region.

Preserve die/core, rows/tracks, I/O geometry/types, PDN, blockages, macros/physical cells, protected identity/placement, clock topology/masters/locations/sinks, existing PG connectivity, SDCs, libraries, scenario definitions and nominal RC policy.

Vendor the authoritative `asap7_equivalent_cell_list.csv` and applicable notices. Every inserted or changed master must exist in the approved catalog and loaded libraries; non-repeater replacements must remain in the same approved group. Liberty similarity does not authorize a missing master. Prohibited changed/inserted masters match `*x1p*_ASAP7*`, `*xp*_ASAP7*`, `SDF*`, `ICG*`.

Protection is derived from the immutable baseline OpenDB snapshot: macro/physical masters; `FIRM`/`LOCKED`/`COVER` status; or any pin/net with signal type `CLOCK`. Timing can conservatively supplement protection, not erase baseline flags. Protected masters/placement/clock/PG fields are fixed, but protected register data inputs may be rebuffered if their resolved source/parity is unchanged. Every nonprotected output instance must be `PLACED`.

New repeaters require connected signal pins; explicit POWER/GROUND connections use baseline nets of the same type. Implicit PG connections are permitted where the baseline flow uses them. Existing component region/halo/properties and PG records remain fixed.

### 2.4 Official evaluation remains authoritative

The official pipeline checks structural/fixed-data legality, DEF/Verilog consistency, clock integrity and formal equivalence, then routes, extracts parasitics and obtains final MCMM timing from a fresh-process SPEF reload. It computes scores from its own artifacts, never contestant logs.

An **unchanged baseline is valid calibration but rejected as a measured contestant submission**. P0 must therefore prove a real allowed change, not copying input files. A changed result can still score poorly; eligibility and improvement are separate gates.

Official hold tolerances (`1.0 ns` TNS / `0.01 ns` WNS in the current provisional config) define score penalties, not universal absolute legality cutoffs. Baselines may already exceed them. Final weights are provisional; production search does not import reference CSVs or official weights.

Normal scoring requires the measured runtime/certificate from `run_submission.py` and packaged baseline/Resizer normalization inputs. Direct `eval.sh` results have no certified contestant runtime. `--allow-unverified` is explicitly numerical experimentation, never a repair for failed certification.

## 3. Goals, non-goals and anti-goals

**Goals**

- Quickly establish a real changed submission and a reproducible MCMM sizing loop.
- Improve setup TNS/WNS with per-scenario hold/ERC limits anchored to the original input.
- Keep Python candidate generation testable against captured snapshots and OpenROAD modifications transactional.
- Compare results through a small harness invoking the real evaluator/scorer.
- Preserve Snapshot/Move JSON as future extension seams without implementing backends now.

**Non-goals**

- Custom STA, Liberty parser, router, signoff engine, GUI, service, database or distributed execution.
- Full Python design IR, a second mutable netlist, algorithm registry/plugin framework, generic backend protocol or experiment database.
- Independent implementation of all official score arithmetic; use the official scorer in development.
- Mandatory production global routing/SPEF extraction at every accepted change, or a pool of routed search checkpoints.
- C++/GPU/ML integration before measured need; immediate superiority over Resizer on every design.

**Anti-goals**

- Weakening clock/placement/equivalence or MCMM hold/ERC checks to simplify architecture.
- Parsing console logs for metrics; silently accepting missing/stale measurements.
- Benchmark-specific heuristics or runtime access to evaluator/reference/harness files.
- Calling a baseline fallback or an unverified numerical score a successful submission.

## 4. Approaches and selected tradeoff

**Full A — layered versioned pipeline.** Preserves separation, but the earlier proposal introduced a complete design/catalog model, registry/protocol layers, separate emitter/rollback components and broad experiment plumbing before a working loop. Replaced by A-lite.

**A-lite — selected by the requested revision.** Retain Python decisions, a persistent OpenROAD client, scenario-aware measurements, atomic Move batches and a minimal evaluation harness. Merge emission/checkpoint execution into the client, logging into one module and objectives into `scoring.py`. Query library facts directly from OpenROAD. Python never projects an independently mutable copy of the design.

**Pure B — Tcl-first flow.** Short initialization path, but moves decision algorithms into Tcl and weakens the intended Python experimentation boundary. Rejected; reuse the evaluator's small Tcl patterns rather than its whole orchestration shape.

The tradeoff is deliberate: fewer state/serialization boundaries and earlier real runs, not fewer safety checks. The suggested 30–40% complexity reduction is a motivation, not a measured performance or code-size claim.

## 5. Minimal architecture

### 5.1 Files

```text
ispd27-tool/
├── run.sh
├── setup.sh
├── pyproject.toml
├── Makefile
├── README.md
├── THIRD_PARTY_NOTICES
├── data/asap7_equivalent_cell_list.csv
├── src/ispd27tool/
│   ├── __main__.py              # parse/execute; no separate CLI framework
│   ├── config.py                # invocation and numeric search budgets
│   ├── optimizer.py             # accepted state, trials, decisions, deadlines
│   ├── scoring.py               # baseline-relative feasibility and objective
│   ├── logging.py               # JSONL events and final labeled summary
│   ├── openroad/
│   │   ├── client.py            # processes, ECO dispatch, checkpoint/export
│   │   ├── protocol.py          # private file framing and safe Tcl encoding
│   │   ├── checks.py            # immutable audit comparisons and authorization
│   │   └── scripts/
│   │       ├── init.tcl         # load design/library/SDCs/scenes/RC once
│   │       ├── analyze.tcl      # global MCMM metrics, paths, target facts
│   │       ├── eco.tcl          # apply, legalize, checkpoint and restore
│   │       └── export.tcl       # physical/netlist/audit export
│   ├── model/
│   │   ├── timing.py            # Snapshot and compact immutable observations
│   │   └── moves.py             # Candidate and explicit ECO operations
│   └── algorithms/
│       ├── sizing.py
│       └── buffering.py
├── schemas/
│   ├── snapshot_v1.json
│   └── move_v1.json
├── harness/                     # development only; excluded from ZIP
│   ├── regress.py              # inspect/calibrate/submit/evaluate/matrix
│   ├── score.py                # official scorer invocation and raw deltas
│   └── package.py              # small explicit-allowlist ZIP builder
├── tests/
└── dist/
```

`algorithms/pin_swap.py` is added only when entering P4. No `backends/`, registry, generic algorithm contract, standalone library parser, general design model, emitter, rollback package, report package or four-family schema set is created.

`openroad/checks.py` remains separate because contest-specific structural/fixed-data checks are necessary safety code, not an algorithm framework. It may inspect complete immutable audit exports transiently; it must not maintain or mutate a shadow design or simulate ECO execution.

### 5.2 Ownership and communication

- **OpenROAD:** live cells/nets/connectivity/placement, timing state and library truth; all ECO mutations and legalizer operations happen here.
- **Client:** one sequential persistent search process, safe typed dispatch, checkpoint I/O, scene/RC reconstruction, artifact extraction and clean final reload. No candidate ranking.
- **Optimizer:** immutable input/current measurements, attempted candidate keys, chosen profile, convergence/deadlines and checkpoint identities. No mutable cell/net graph.
- **Algorithms:** direct pure functions over Snapshot observations; no subprocess or file I/O. The optimizer calls sizing/buffering explicitly; no registration/discovery system.
- **Checks:** candidate-local authorization before mutation and complete actual-state invariant comparisons afterward. Compare baseline to current for immutable rules and current to restored state for rollback proof.
- **Harness:** only development-time access to official evaluator/scorer. One fresh result directory per run; retain raw deltas, timings and failure reasons, not an experiment database.

Snapshot and Move are the only public versioned JSON contracts. Private client request IDs/result files need framing and correlation, not another published schema/API. Typed Python data is compiled into safely quoted Tcl operation arguments; Tcl need not parse JSON. Use dedicated atomically published result files for structured data; stdout/stderr are diagnostic logs only.

## 6. Read-only observations and explicit moves

### 6.1 Snapshot v1

`schema: ispd27.snapshot/1`; `design_version`; `generation`; `state` (`accepted` or `trial`); optional `candidate_id`; `fidelity: placement`; explicit units (ns/fF/um/um²/W).

Contains:

- Scenario IDs, logical modes, exact STA constraint-mode identifiers, corner, SDC and library identities, clock periods/physical-vs-virtual propagation and coverage information.
- Global setup/hold nonpositive WNS/TNS and violating endpoint counts, global ERC sums/violators and power for every scenario.
- Bounded setup/hold critical paths with endpoint/pin/arc delay, arrival, required, transition, slew and load.
- Read-only target cells and one-hop affected net neighborhoods, including all sinks needed to specify a net cut, placement/protection reasons and component constraints.
- Library facts and approved alternatives only for queried targets/needed repeaters, obtained from OpenROAD; no full Python library universe or name-derived drive-strength model.
- Input/export identities and references to immutable audit/checkpoint artifacts, not handles that algorithms can use to mutate the engine.

A bounded observation is not a partial timing decision: every acceptance uses complete scenario metrics over all relevant endpoints. Path/neighborhood limits only bound proposal data. Refresh those observations after accepted ECOs; do not permanently optimize only the original critical endpoints.

Complete canonical DEF/Verilog, protected/clock records and source/parity audit exports are private validation artifacts, not a second public design schema. Transient audit dictionaries may be constructed to compare actual states and then discarded. There is no Python `apply_to_graph` or inverse-move path.

### 6.2 Move v1

`schema: ispd27.move/1`; `design_version`; `generation`; candidate `id`; `strategy`; ordered `moves`; optional predicted benefit, never used as correctness evidence.

- `resize`: exact instance, expected original master and authorized replacement master.
- `insert_repeater`: original net and driver endpoint, exact selected sink endpoints, approved master, deterministic new instance/net names and proposed coordinates. The original net remains upstream; selected sinks move downstream.
- `remove_repeater`: exact instance, expected input/output nets; reconnect all output sinks to the input net and remove only the now-unused output net. Reject ambiguous drivers or changed parity at surviving endpoints.
- `swap_pins` (P4): complete input-pin bijection `destination_pin -> original_source_pin`, including unchanged inputs; require function symmetry.

An endpoint is an exact `(instance, pin)` pair; empty instance denotes a top-level port. Ordered moves may reference names created earlier in the same candidate. Validate compound inverter edits as a whole, not as individually equivalent intermediate edits. Names are stable and collision-checked in OpenROAD.

`generation` changes whenever a process is rebuilt; accepted `design_version` advances once per accepted candidate. Trial observations keep the base version and carry trial state/candidate identity. Old generation/version proposals are rejected before mutation; algorithms receive accepted snapshots only.

## 7. Initialization, MCMM analysis and the optimization loop

### 7.1 Initialize once; correctness starts in P0/P1

Load the supplied LEFs/Liberties and physical DEF in the verified order, separately check supplied Verilog connectivity, then source MCMM declarations. For the current input schema read `mcmm_corners`, `scenario_mode`, `scenario_sdc`, `mcmm_functional_mode` and `corner_libs`; do not regex-parse Tcl or guess a future scan-mode configuration syntax.

One logical functional mode may require multiple STA constraint modes: the current `functional_BC`, `functional_TC`, `functional_WC` contexts prevent same-named clocks with different periods overwriting each other. Read each SDC into its own provided mode, bind its libraries, propagate only nonvirtual clocks, and apply no additional OCV derating. Internal scenario identities allow multiple modes per corner; extend only configuration discovery when a new documented release requires it.

Derive baseline protection from OpenDB before trial mutations; freeze protected locations during legalization without disabling allowed register data-input reconnection. Source nominal `setRC.tcl`, generate placement parasitics and capture immutable baseline `B0`. Search and final local comparison use that same analysis fidelity. Time is converted from ps to ns once; capacitance remains fF.

### 7.2 Trial and rollback

```text
initialize engine; check input; capture B0, accepted snapshot and checkpoint
while budget permits and untried proposals remain:
    call selected sizing/buffering function directly on accepted snapshot
    choose a same-version candidate; check targets and authorization
    checkpoint accepted OpenROAD state
    apply ordered moves; legalize with protection; refresh parasitics
    export actual invariant audit; measure all MCMM scenarios
    if actual state is legal and feasible and setup objective improves:
        accept once; advance design_version; refresh accepted observations
    else:
        restore checkpoint; rebuild timing/RC state; remeasure actual state
    record result; never retry the same candidate at the same accepted version
stage exact output pair; clean reload and local validation; publish or recover
```

Use one candidate per trial initially; later batch nonconflicting moves into one atomic candidate. Do not independently accept candidates generated against a now-stale state.

Prefer verified `write_db`/`read_db` checkpoints if supported and semantically faithful in the pinned image; reliable clean DEF/Verilog reload is the fallback. Restoring OpenDB alone does not restore STA constraints/scenes/parasitics. Rebuild those and validate/reanalyze before resuming. Checkpoint execution is in the client; checkpoint selection/acceptance is in the optimizer. Never infer inverse moves.

A timed-out, malformed or partially failed mutation invalidates the client session. Restart, increment generation, restore the accepted design and remeasure; a prior cached report alone is not recovery. Fatal boot/input errors do not become ordinary rejected candidates.

### 7.3 Feasibility and initial objective

The following are **tool policy**, separate from official hold penalties. For every scenario, require hold TNS/WNS `candidate >= B0 - 1e-6 ns`; ERC sums may exceed B0 only by numerical allowance: slew `1e-6 ns` (= `0.001 ps`), cap `1e-6 fF`, fanout `1e-6`. Always anchor to B0, never the previous accepted result. Reject missing/nonfinite metrics, mismatched scenario sets and illegal physical/graph transitions.

For feasible states maximize:

```text
Q(T; B0) = sum_s alpha_s * (
    w_tns * (T.setup_tns[s] - B0.setup_tns[s]) / max(abs(B0.setup_tns[s]), 1e-6 ns)
  + w_wns * (T.setup_wns[s] - B0.setup_wns[s]) / max(abs(B0.setup_wns[s]), 1e-6 ns))
setup_wns = min(0, reported_worst_slack)
w_tns = w_wns = 1.0; alpha_s = 1 / scenario_count by default
accept iff feasible and Q(trial; B0) > Q(current; B0) + 1e-9
```

Explicit scenario weights must be positive and normalized. Ties keep the current state. This setup-only internal objective excludes provisional power/displacement/overflow/runtime weights; those metrics remain visible through official evaluation. Cross-corner setup tradeoffs follow the scalar objective and must be reported per scenario, never presented as improvement in every individual metric.

A hold repair can compensate a setup-promising candidate within the same atomic trial; a hold-only change without setup-objective gain is not committed independently. Rejecting a candidate does not disable an algorithm while untried candidates remain. Converge after complete proposal exhaustion; batching/search refinements must preserve this behavior.

## 8. Legality without a second mutable design

`openroad/checks.py` and actual OpenROAD audits implement the evaluator's rules locally, without runtime evaluator imports. Reuse/adapt its canonical DEF/Boolean/source-resolution checks with license notices rather than inventing different semantics.

Before mutation check versions, expected masters/connections, approved groups, protected targets and operation parameters against the live engine. After a complete trial and legalization compare actual exported state against the immutable baseline:

- Fixed DEF sections, component attributes, protected physical/clock/PG records, legal placement and top-level interface must remain valid.
- Resolve sources through actual repeater chains and compare source instance/pin and inversion parity at surviving logic inputs, protected data inputs and top-level outputs, modulo validated pin symmetries.
- Reject repeater cycles, multiple drivers, unsupported signal INOUT/tristate behavior and missing signal connections.
- Distinguish ordinary logic substitutions from coordinated repeater replacements; end-to-end graph checks govern repeater polarity.
- Check both DEF and Verilog represent the same actual cells/connectivity before final publication.

These transient read-only checks do not authorize Python to execute ECOs in a shadow graph. Formal equivalence remains an independent official/harness proof; construction-based local checks are not marketed as formal certification.

## 9. Minimal harness and measurement boundaries

`harness/regress.py` provides inspect, baseline calibration, measured submission, direct evaluation and a small profile matrix. `harness/score.py` invokes the official `evaluator/score.py` and emits raw per-scenario deltas; it does not reimplement the fourteen-term scoring policy.

- Calibration runs official baseline evaluation into a fresh directory and compares with packaged reference metrics. No baseline contestant eligibility or expected zero score.
- Measured submission uses `run_submission.py` with the preserved `--` command separator. Only this path supplies certified contestant runtime.
- Direct evaluation uses captured DEF/Verilog and labels runtime/scoring as unverified unless independently measured and certified by the official harness.
- Matrix rows cover baseline, sizing-only, buffering-only, combined and P4 pin-swap-only when implemented. Preserve failures as rows, not missing data.
- Store configuration/input/tool identities, per-scenario setup/hold/ERC/power, official displacement/overflow, accepted/rejected ECO counts, stage times and raw deltas in simple JSON/CSV/JSONL files. No experiment database, registry or generic execution framework.

Final production validation is intentionally smaller than the previous plan: clean-reload the exact staged output pair, rebuild supplied MCMM constraints/placement parasitics, verify graph/physical consistency and compare with B0. Label these metrics `placement`; do not claim routed/signoff equivalence.

**Authoritative routed quality, fresh SPEF timing, formal proof and certification remain required external phase/release gates through the official evaluator.** A locally accepted placement gain may fail that gate; improve the heuristic using recorded evidence rather than changing labels or adding an unrequested production routing framework.

## 10. Runtime, output publication and failure behavior

Use a monotonic deadline. Planner defaults: tool limit `12600 s`, reserve `1800 s` inside it for finalization, leaving `5400 s` of the five-hour flow limit for external evaluation. These are configurable downward safety margins, not a guarantee about hidden evaluation duration. Per-request limits never exceed the remaining phase budget.

Read `NUM_CORES` (default 8, positive integer, capped at 16). Initial Python algorithms are single-threaded. Stable proposal order, fixed seeds and deterministic names; only budget exhaustion is wall-clock-dependent.

Keep OpenROAD processes in the tool's inherited process group so organizer cleanup reaches them; never detach workers. Track/terminate/reap owned subprocesses on timeout/exit without killing the parent group.

Retain baseline/current checkpoints and a bounded last changed clean-reload-validated checkpoint; do not accumulate full snapshots per trial. Preserve diagnostics and small metric traces, remove obsolete work artifacts after they are no longer needed.

Publish only after both staged files pass local consistency/placement/MCMM checks. Required output paths are fresh; the caller consumes them after successful exit. On recoverable late failure, emit the last changed clean-reload-validated checkpoint with exit zero. If none exists, exit nonzero and preserve diagnostics; a baseline debug copy is never a successful output. `logging.py` writes one JSONL event stream and a final summary with explicit fidelity and `tool_validated_not_certified` status.

## 11. Development phases and acceptance gates

| Phase | Scope | Required evidence |
| --- | --- | --- |
| P0: real valid submission | Initialization, exact output contract, master/protection checks and one actual legal ECO | Official measured submission completes with `changed_from_baseline`, formal/clock/placement checks and runtime certification; copying baseline is insufficient |
| P1: minimum MCMM loop | Sizing, full-scenario timing/ERC, checkpoint apply/accept/reject/reload, baseline-relative constraints and final local validation | Real accepted and rejected trials, stale/session recovery proof, no false success; mandatory MCMM/hold/ERC checks are already present |
| P2: quality | Critical-path/variant ranking, measured sensitivity and refreshed observations | AES setup TNS/WNS improvement with per-scenario hold/ERC parity; report which setup components improve rather than hiding tradeoffs |
| P3: search capability | Branch buffering, repeater edits, compensating hold repair, safe atomic batching and measured incremental evaluation when justified | Combined routed setup objective improves over sizing-only on both public designs without violating feasibility; correct full analysis remains fallback |
| P4: advanced optimization | Function-symmetric pin swapping; separately decide whether sophisticated search/native/ML work is justified | Real pin-swap equivalence/rejection and timing evidence; no native/ML implementation is promised or scaffolded without measured need and a separate decision |

P0/P1 are vertical priorities, not permission to postpone correctness. Run the actual changed launcher through official smoke evaluation before broad harness/backend work. Validate the extracted offline package for the P3 release and repeat those checks for the P4 pin-swap extension. Algorithmic quality gates are empirical: keep failed evidence and tune within the owning phase, never substitute legality alone for improvement.

## 12. Testing and distribution

Use standard-library `unittest`, isolated temporary fixtures and deterministic behavioral tests: invalid/stale Move inputs, unit conversions, actual graph/physical constraints, baseline-relative hold/ERC boundaries, accept/reject transitions and publication failures. Recorded Snapshot fixtures test proposal decisions without pretending to execute EDA. Do not add source-text/default/copy-only tests or a new test framework.

Exercise real OpenROAD initialization, same-named scenario clocks, physical/virtual propagation, every implemented ECO, legalization, restore and final reload. Use official measured AES/JPEG submissions and inspect completion markers, certificates and dedicated reports. Tests of Python decisions alone are not implementation proof.

`Makefile` targets: `test`, `lint`, `run-local`, `dist`. Python-only lint is syntax checking and `bash -n`, not a new formatter toolchain. `harness/package.py` creates `dist/<TEAM_ID>_solution.zip` from an explicit runtime allowlist, retaining executable modes and notices while excluding harness/tests/docs/experiments/caches. Validate clean extraction, setup idempotence, offline execution from unrelated cwd and exact output names with the original tool checkout inaccessible. Keep final archive identity/checksum with official results.

## 13. Remaining builder decisions and evidence

The implementation plan pins callable names, small target limits and test fixtures. Builder decisions remaining inside this design: exact proven OpenROAD commands, faithful DB-vs-clean-reload checkpoint selection, bounded path detail and deterministic proposal placement heuristics. Verify these in the supported image; absence of a needed API is not evidence that a fake/empty adapter is acceptable.

Future multi-mode configuration syntax, revised weights, incremental-update behavior and hidden-benchmark evaluation duration cannot be inferred from the current release. Keep the internal scenario model general, update input discovery only from documented releases and record runtime/quality limits honestly.

## 14. Source references

Paths are relative to the contest repository root; they are development references, not production dependencies.

- `evaluator/run_submission.py:54-86`: launcher environment, measured runtime, fresh artifacts and completion.
- `evaluator/internal/legality.py:239-390,439-449`: protection, approved groups, source/parity checks, PG invariants and unchanged-submission rejection.
- `evaluator/internal/evaluation.tcl:145-176`: independent SDC modes, nonvirtual propagated clocks and nominal RC.
- `evaluator/internal/refresh_timing.py:27-81`: authoritative routed fresh-process timing.
- `evaluator/score.py:198-239`: certificate/runtime and packaged-reference requirements.
- Supplied A-lite review (`local://paste-1.md`): simplify state ownership and infrastructure; the selected architecture and phase corrections are recorded above so the ephemeral paste is not needed to implement them.
