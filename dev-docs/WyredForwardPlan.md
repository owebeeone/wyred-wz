# ElecScad Forward Plan (post gen-4)

*DRAFT for discussion — simple bullets, not yet phased into <500-LOC steps.
Inputs: [StartHere.md](./StartHere.md) §8 paths A–G, [ga019 gen-4 review](../rl/ga019-gen4-review.md)
§3/§4, and the outside-in datapoint [g19 EDA Tool Comparison](../rl/ga019/docs/g19%20EDA%20Tool%20Comparison.md).*

**Ordering rationale (one line):** close the science first, then make the moats
visible while the agentic window is open, then do breadth on the surface people
will actually use. The EDA comparison promotes productization (F+E) above
breadth (A) relative to the gen-4 review's own A+C recommendation, because
Path A is better measured on the productized surface by a non-builder author.

## Phase 1 — Close & harden gen-4 (Paths C + D)

- Wire the board-agreement probes into the gate (third-party-tree presence
  check or CI) — kills the R16/R25 defect class.
- Run the human ratification pass: 13 recorded spec-deviation stances + 8 open
  items (RESUME.md / DslGen4 §8).
- Formalize grader-side external baseline retention.
- Close the §4.2.1 impostor hole: per-role part manifests in the refinement
  record.
- Ship the one-command consumer-side audit tool (crosscheck + rebuild +
  baseline verify over a directory).
- Declare gen-4 frozen.

## Phase 2 — Reconcile & productize (Paths F + E)

- Decide canonical codebase: promote ga019's design into `elecscad/`; retire or
  fold in the front-end spike.
- **Qualified registry keys:** replace the flat process-global
  `MODULES`/`INTENTS`/`REFINEMENTS` dicts (keyed by bare class name / intent
  string) with qualified keys or a per-build registration context. Composition
  already goes through class objects, so only the law-2 sweep, intent/refinement
  discovery, and the `--list`/`--exemplify` CLI need rework. Design the naming/
  versioning story for the strings that cross file boundaries (intent names,
  emit names).
- Clean public authoring API + install story (package, docs, examples written
  for authors rather than the scoring critic).
- Real bidirectional KiCad round-trip.
- LLM/agentic surface as a first-class deliverable (structured artifacts +
  MCP/skill layer) — the comparison's strongest wedge; JITX/tscircuit already
  ship theirs.
- Library/part packaging story (how a third-party parts library is published
  and imported without name collisions — follows from qualified keys).

## Phase 3 — Breadth on the product surface (Path A)

- Third/fourth boards from different families (an RF board; a 4-layer MCU board
  with a real alternate-function matrix).
- At least one encoded by an author who did NOT build the engine — turns the
  n=2/1-author efficiency trend into a distribution and tests authorability.
- Second connectivity engine for legacy schematics (minimal EESchema v4 text
  netlister) to restore the two-engine differential on MPPT-class sources.

## Phase 4 — A real solver rung (Path B)

- Typed alternate-function matrices as pool unit constraints.
- One board where lowest-free-unit is UNSAT and the ladder must emit a real
  UNSAT core at board scale.
- On-ramp to the constraint world GHz needs (differential pairs,
  length-matching).

## Phase 5 — GHz spike (Path G, parallel / slack-time)

- The Architecture §7 one-trace spike: scikit-rf Z0 → geometry → openEMS
  S-params → compare, hand-wired. Tractability probe only, not a build.
- Uncontested territory per the comparison (competitors attack generic routing,
  not physics-verified GHz).

## Explicitly deprioritized

- Live supply-chain/database integration — commodity plumbing,
  delegate-don't-solve; boltable later as a library-builder backend without
  touching the intent layer.
- Generic autorouting/layout — entrenched competitors (JITX, tscircuit); the
  two-layer architecture keeps layout downstream by design.

## Carried harness items (slot when touched)

- Harness v4: fix `mutate._mut_shoot_through` applicability for MOSFET-anchored
  invariants (P6, carried since M2).
