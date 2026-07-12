# wyred DecisionLog

Append-only. One entry per decision; newest first. Format:
`## YYYY-MM-DD — title` / context / decision / consequences.

## 2026-07-12 — Ground-truth corpus moved to wyred-t9y (hybrid fork/vendor/download)

The third-party board trees (exogenous denotations) moved from the elecscad
symlink to `~/limbo/wyred-t9y` (plain git, NOT gwz-managed). Hybrid model:
six upstreams as submodules of owebeeone `t9y-*` GitHub forks pinned at the
provenance-recorded SHAs (availability + verifiable exogeneity; `update=none`
so naive recursive clones don't pull ~500 MB unpruned — `materialize.py`
drives init with sparse-checkout); MNT Reform vendored (GitLab upstream,
CERN-OHL-S permits redistribution); rpi-cm4io download-on-materialize with
pinned sha256 (LICENSE-UNKNOWN — never redistributed). `view/` reproduces the
pruned reference trees byte-identically (one vendored override:
`watchy/Watchy.kicad_prl`, a KiCad-generated settings file absent upstream).
`wyred-wz/third-party` now symlinks `../wyred-t9y/view`; `run_gate.py` gained
content-based probe-readiness checks and prints the ground-truth revision
pins when probes run. Since t9y is outside gwz, that printed pin line is the
audit-trail record of which ground truth a gate-green run used.

## 2026-07-12 — Phase-3 residual closure

Context: the phase-2 adversarial critic left four residuals against the
gate/audit build. All four are now closed or recorded:

- **R1 (MED) closed — the ECO minimal-disturbance verdict is no longer
  engine-self-certified.** `wyred-audit/src/wyred_audit/incumbency.py` carries
  the audit's own independent ports of the engine's `paths.diff_pinmaps` and
  `emit._check_incumbency` (== ga019 runner.py:685-711) and re-derives EVERY
  field of every `<name>.pinmapdiff.json` from the on-disk artifacts alone
  (the incumbent's pinmap+alloc plus the artifact's own), comparing field by
  field — `minimal_disturbance_violations` included (`PINMAPDIFF_DISAGREE`,
  `MINIMAL_DISTURBANCE_NOT_OK`). A lobotomized engine-side incumbency check
  now turns the audit red (fixtures `pinmapdiff_blanked`,
  `sticky_alloc_moved`) — the same restoration shape as the closed F3 finding.
- **R2 (LOW) closed — from-primaries rebuild honesty restored, corpus-less.**
  `python3 -m wyred.rebuild --dir <tree> [--artifact NAME | --all]`
  (`wyred/src/wyred/rebuild.py`) is the direct engine-side port of
  runner.py:554-574: every secondary path (bom/pinmap/records) re-derived
  from the on-disk PRIMARY artifacts alone (l2 + alloc + l1, no corpus) and
  byte-compared. `wyred-audit` composes it as a subprocess stage
  (`REBUILD_FROM_PRIMARIES`, needs only `--engine-src`), so a consumer
  holding artifacts + an engine checkout — no corpus — can again prove the
  secondaries are pure functions of the on-disk primaries. Distinct from the
  audit's whole-corpus re-emit, which needs `--corpus-dir` and proves emit
  determinism against the source corpus.
- **R3 (LOW) closed — gate.py has a committed regression suite.**
  `wyred-harness/tests/run_gate_tests.py` (own entry point; deliberately NOT
  wired into the frozen 149-check selftests) exercises the gate end-to-end
  via subprocesses in temp dirs: the clean examples corpus gates green with
  the full 26/26 battery line; the committed `tests/fixtures/f4_corpus/`
  fixture (the F4 declared-vs-oracle disagreement — the engine emits fine,
  the gate must fail naming ADDR_COLLISION); the applicability-lobotomy
  corpus (board modules removed → the "probe(s) never found an applicable
  artifact" verdict must be the run's ONLY failure); and a dropped-BOM-line
  tamper failing the cross-path differential. These verdicts were previously
  proven only with ad-hoc doctored corpora.
- **R4 (INFO) recorded — connlock `groups` derivation asymmetry, accepted as
  fails-closed.** The audit derives a connlock record's `groups` as the l1's
  lock groups covering `connector_pinout` at version >= 1; the engine
  recorded the groups it FROZE at that emit (freeze ∩ covers — freeze
  membership is not on disk). An inherited locked connector group that was
  not re-frozen at an emit would make the audit's set larger and fire a
  spurious `CONNLOCK_DISAGREE`: a false alarm, never a missed tamper. No
  corpus artifact exercises that shape today; the fix is deferred until a
  real corpus case does. Recorded in wyred-audit/README.md ("Explicitly
  deferred").

Evidence at closure: a fresh emit of wyred-examples/corpus is byte-identical
to wyred-contract/goldens/ga019; `run_gate.py` end-to-end PASS (board
probes ran); full-form audit PASS (all seven checks); `run_gate_tests.py`
and `run_fixture_tests.py` PASS; the 149 harness selftests PASS.

## 2026-07-12 — Runner-split deviations ratified by the phase-2 gate/audit build

Context: the phase-1 adversarial review flagged three deviations from ga019's
`runner.py`. The phase-2 build (`wyred-harness/harness/gate.py`, `wyred-audit`,
`wyred-examples/run_gate.py`) closes them by construction; the deviations are
now deliberate and ratified:

- **(a) F3 — emit-time lock/baseline snapshot code stays engine-side.**
  Baseline/lifecycle/connlock bytes are emit-time artifact CONTENT, so the
  snapshot code lives in `wyred/src/wyred/emit.py`. The trust property is
  restored outside the engine: `wyred-audit` independently re-derives every
  snapshot from disk (harness library) and byte-compares on every audit run —
  any drift between the two implementations turns the audit red.
- **(b) F4 — L2 emission gates on the declaration alone.** The engine emits L2
  for declared-clean intents without consulting the live oracle; oracle
  disagreement is the harness gate's verdict, now enforced end-to-end by
  `gate.py`'s expected-codes comparison (the phase-1 detection hole is closed).
- **(c) F5 — cross-boundary composition is subprocess-only.** Gate and audit
  reach engine-side checks (`paths.crosscheck`, `check_connector_locks`) only
  via the `python3 -m wyred.crosscheck` CLI; imports across the engine/harness
  fence remain forbidden (RunnerSplit.md composition rule). `wyred-audit`
  additionally carries its own independent connlock port, kept honest by the
  audit going red on drift.

## 2026-07-12 — Bootstrap decisions (workspace creation)

- **Name: wyred.** PyPI/crates/npm/brew/wyred.dev available; github.com/wyred
  is an unrelated user account (use owebeeone/*); nearest trademark neighbor is
  Wyred 4 Sound (consumer hi-fi) — check before commercial use.
- **Substrate: Python.** The authoring surface (marker-harvest metaprogramming)
  IS the measured product; performance lives in delegated tools. Rust deferred
  per-component (first candidate: wyred-audit as a static binary); the L1 JSON
  boundary keeps a future Rust resolver insertable via two-engine differential.
- **Topology: star, depth 1, contract at the center.** Nine members (see root
  README). Repo boundaries are AI fences forcing API hygiene; goldens-first
  bootstrap unblocks all tracks before the engine port lands.
- **Contract governance: single-writer.** Proposals in dev-docs/; versioned
  bumps land only in wyred-contract. Contract v0 = elecscad EMIT_CONTRACT v3
  verbatim + ga019 gate-green goldens.
- **elecscad stays as the research archive.** rl/ scorecards + reviews are the
  durable audit trail; wyred is the product. The old top-level elecscad/
  package spike is retired by this move (forward plan Path F resolved).
- **Two tracks:** g19++ (full-featured buildout, this workspace) and g20+
  (surface-dialect iterations over the frozen contract, scored on a declared
  new fitness axis). Vocabulary/facility changes belong to the g19++ track.

Open (not yet decided): gen-4 ratification pass (13 stances, 8 open items —
prerequisite before contract v1); wyred-ui plain-React vs grip/glial client;
which board is the tracer-bullet full-loop target.
