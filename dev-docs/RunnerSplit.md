# Runner split — decomposing ga019's runner.py across the repo boundaries

*The one entanglement hotspot in the port. ga019's `runner.py` (~950 lines)
mixes concerns that now live in three repos. Agents porting `wyred` and
`wyred-harness` MUST follow this split; neither claims runner.py wholesale.*

Source: `/Users/owebeeone/limbo/elecscad/rl/ga019/runner.py`.

## The split

| runner.py concern | goes to | notes | status (2026-07-12) |
|---|---|---|---|
| corpus discovery + import loop (`discover()`) | `wyred` | generalize: discovery over a configured package, not hardcoded `corpus/` | BUILT — `wyred/src/wyred/emit.py` |
| elaboration + refinement emit loop, artifact writing | `wyred` | the emit half; ends at artifacts-on-disk | BUILT — `wyred/src/wyred/emit.py` |
| law-2 zero-arg sweep over MODULES | `wyred` | engine self-property, runs pre-emit | BUILT — `emit.py` law-2 sweep |
| oracle gating (layer-1 oracle, v3 stack, spec satisfaction) | `wyred-harness` | consumes artifacts from disk; no engine imports | BUILT — `wyred-harness/harness/gate.py` (L1 vs declared expected codes; v3 stack incl. check_allocations w/ record2 + external baseline); regression suite `tests/run_gate_tests.py` (phase-2 residual R3: clean baseline, committed F4-disagreement fixture, battery-lobotomy, tamper) |
| cross-path differential (XPATH_*) + 26-probe counter-battery + lobotomy check | `wyred-harness` | battery must fire all codes every run or the run fails | BUILT — `gate.py` (differential re-run from disk via `python3 -m wyred.crosscheck` subprocess; 26-probe battery + lobotomy verdict) |
| determinism check (double emit, byte-identical) | `wyred-harness` | needs two engine runs: invoke engine as a subprocess, compare bytes | BUILT — `gate.py` (two subprocess emits, byte-compared) |
| external lock baselines: verification (tamper probes, fork/lifecycle, connlock re-derivation) | `wyred-audit` | the consumer-trust half. Retention (the `<name>.baseline.json` snapshot at each locked emit) is engine-side in `wyred/src/wyred/emit.py` — baseline bytes are emit-time artifact content; the audit re-derives every snapshot independently and byte-compares (DecisionLog 2026-07-12, item a) | BUILT — `wyred-audit` (`baselines.py`, `connlock.py`; ECO/pinmapdiff + minimal-disturbance counter-check `incumbency.py`, phase-2 residual R1) |
| disk rebuild honesty (re-emit, byte-compare vs tree; from-primaries secondary-path rebuild, runner.py:554-574) | `wyred-audit` | shares subprocess-invocation pattern with harness determinism check; the from-primaries half is engine-side CLI `python3 -m wyred.rebuild` (corpus-less), composed by the audit as a subprocess | BUILT — `wyred-audit/src/wyred_audit/rebuild.py` (whole-corpus re-emit + `REBUILD_FROM_PRIMARIES` stage, phase-2 residual R2) |
| board-agreement probes (watchy_agreement, mppt_agreement) | `wyred-examples` | GATE-WIRED there (R16/R25 lesson) | BUILT + GATE-WIRED — `run_gate.py` stage 4 (`watchy_agreement.py`, `mppt_agreement.py`, recorded floors) |
| CLI (`--list`, `--exemplify`, arg parsing) | `wyred` (emit-side flags) | gate/audit get their own CLIs in their repos | BUILT — `python3 -m wyred.emit`; `harness/gate.py --tree --corpus-dir [--engine-src]`; `python3 -m wyred_audit --tree ...` |

## Composition rule

The end-to-end gate ("emit, then verify everything") is recomposed at the
**process level**, not by imports: a thin driver (lives in `wyred-examples`
for the integration bed; consumers use `wyred-audit`) that runs
`wyred` emit → `wyred-harness` gate → `wyred-audit` verify as subprocesses
over a shared artifact directory, failing on first nonzero exit. This
preserves the ga019 trust property: no verdict ever consults in-memory engine
state.

**BUILT** — the integration-bed driver is `wyred-examples/run_gate.py`
(emit → gate → audit → gate-wired board-agreement probes, each stage a
subprocess from a neutral cwd, zero cross-repo imports).

## Port order

1. `wyred` extracts the emit loop (acceptance: byte-identical goldens). — DONE
2. `wyred-harness` extracts gate + battery (acceptance: 149 selftests +
   identical verdicts on goldens). — DONE (`harness/gate.py`; committed
   regression suite `tests/run_gate_tests.py`)
3. `wyred-audit` extracts baselines/rebuild (acceptance: tamper fixtures fire).
   — DONE (`tests/run_fixture_tests.py`)
4. `wyred-examples` recomposes the full gate and wires the agreement probes in.
   — DONE (`run_gate.py`)
