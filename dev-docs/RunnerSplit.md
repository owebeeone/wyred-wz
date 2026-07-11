# Runner split — decomposing ga019's runner.py across the repo boundaries

*The one entanglement hotspot in the port. ga019's `runner.py` (~950 lines)
mixes concerns that now live in three repos. Agents porting `wyred` and
`wyred-harness` MUST follow this split; neither claims runner.py wholesale.*

Source: `/Users/owebeeone/limbo/elecscad/rl/ga019/runner.py`.

## The split

| runner.py concern | goes to | notes |
|---|---|---|
| corpus discovery + import loop (`discover()`) | `wyred` | generalize: discovery over a configured package, not hardcoded `corpus/` |
| elaboration + refinement emit loop, artifact writing | `wyred` | the emit half; ends at artifacts-on-disk |
| law-2 zero-arg sweep over MODULES | `wyred` | engine self-property, runs pre-emit |
| oracle gating (layer-1 oracle, v3 stack, spec satisfaction) | `wyred-harness` | consumes artifacts from disk; no engine imports |
| cross-path differential (XPATH_*) + 26-probe counter-battery + lobotomy check | `wyred-harness` | battery must fire all codes every run or the run fails |
| determinism check (double emit, byte-identical) | `wyred-harness` | needs two engine runs: invoke engine as a subprocess, compare bytes |
| external lock baselines: retention, tamper probes, fork verification | `wyred-audit` | the consumer-trust half |
| disk rebuild honesty (re-emit, byte-compare vs tree) | `wyred-audit` | shares subprocess-invocation pattern with harness determinism check |
| board-agreement probes (watchy_agreement, mppt_agreement) | `wyred-examples` | GATE-WIRED there (R16/R25 lesson) |
| CLI (`--list`, `--exemplify`, arg parsing) | `wyred` (emit-side flags) | gate/audit get their own CLIs in their repos |

## Composition rule

The end-to-end gate ("emit, then verify everything") is recomposed at the
**process level**, not by imports: a thin driver (lives in `wyred-examples`
for the integration bed; consumers use `wyred-audit`) that runs
`wyred` emit → `wyred-harness` gate → `wyred-audit` verify as subprocesses
over a shared artifact directory, failing on first nonzero exit. This
preserves the ga019 trust property: no verdict ever consults in-memory engine
state.

## Port order

1. `wyred` extracts the emit loop (acceptance: byte-identical goldens).
2. `wyred-harness` extracts gate + battery (acceptance: 149 selftests +
   identical verdicts on goldens).
3. `wyred-audit` extracts baselines/rebuild (acceptance: tamper fixtures fire).
4. `wyred-examples` recomposes the full gate and wires the agreement probes in.
