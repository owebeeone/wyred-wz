# Wyred placement plan — declare-don't-route, verify-from-the-board

*Phased plan elaborating WyredWorkflowDesign.md §3.2 (placement constraints).
Phases are shippable milestones; each step is one goal with an aspirational
< 500-LOC budget, a repo, a model assignment, and a mechanical acceptance
check. Foundational steps land first; steps are structured so independent
agents can pick them up with minimal cross-step coupling. Companion sources:
RunnerSplit.md (repo boundaries, composition rule), DecisionLog.md
(single-writer contract governance), `wyred/src/wyred/core.py` (the
marker-declaration pattern), `wyred/src/wyred/emit.py` (artifact writing),
`wyred-harness/harness/pcb_extract.py` (board extraction),
`wyred-examples/run_gate.py` (gate stages and probe wiring).*

## 0. Invariants (non-negotiable, every step)

- **`.placement.json` is emitted ONLY when placement declarations exist.**
  The current corpus declares none, so every step in this plan must leave
  all 107 goldens in `wyred-contract/goldens/ga019/` byte-identical and
  `wyred-examples/run_gate.py` green. The engine's existing `_put(doc, key,
  value, empty_default)` omit-when-empty pattern (engine.py `_emit_doc`) is
  the mechanism: an empty placement section is *absent*, not `[]`.
- **The engine never imports the harness** (and vice versa). Declaration and
  emission live in `wyred`; measurement of the realized board lives in
  `wyred-harness`; composition is subprocess-only at artifacts-on-disk
  (RunnerSplit.md composition rule).
- **Contract additions go through the single-writer flow.** The
  `.placement.json` schema is a versioned contract change: proposal in
  `dev-docs/` first; it lands in `wyred-contract` only via the contract
  owner (DecisionLog 2026-07-12, bootstrap decisions).
- **Asserted vs realized.** Wyred declares placement intent; the laid-out
  board is read back (`pcb_extract`) and *compared*, never trusted. Layout
  itself stays delegated forever.
- **Every check fails closed and loudly.** A constraint the checker cannot
  evaluate (unresolvable subject, missing geometry) is a structured code,
  never a silent skip — the F8 / PCB_NET_NAMES_MISSING discipline.

## 1. What exists that this builds on

- **Marker declarations** (`core.py`): `param` / `demand` / `provide` /
  `rail` / `ground` / `bond` / `bus` / `pool` / `mutual_exclusion` /
  `lock_group` / `use` — `Decl` subclasses harvested from class bodies in
  global creation order; identity minted from attribute names namespaced by
  instance path (law 8); `mutual_exclusion` already shows the
  declare-once-mint-per-instance namespacing placement needs.
- **The emit loop** (`emit.py`): per-artifact writers, double-emit
  byte-determinism, per-row provenance in the data paths (`for_demand` /
  `l1_role` attrs on pin-map components), and the only-when-applicable
  artifact pattern (`.baseline.json` only for locked emits,
  `.pinmapdiff.json` only with `incumbents`, `.connlock.json` only when a
  connector_pinout group fired). `.placement.json` follows the same shape.
- **`pcb_extract.py`**: a connectivity-only extractor today (footprints →
  refdes/value/pads→nets, both board dialects, real s-expression parser).
  It deliberately ignores positions, rotations, courtyards, the board
  outline, and zones — exactly the geometry placement verification needs.
- **`run_gate.py`**: emit → gate → audit → gate-wired board-agreement
  probes, each a subprocess from a neutral cwd, content-based
  probe-readiness checks, loud SKIP banners, ground-truth revision pins.
  The placement-agreement stage joins this pipeline in the same style.
- **Ground truth**: `third-party/watchy/Watchy.kicad_pcb` (via wyred-t9y)
  is the realized board for the demonstration; the corpus's
  `watchy_v1.py` encoding is the asserted side.

## 2. Phase 1 — Placement vocabulary + emitter (repo: `wyred`)

**Milestone:** an author can declare `near` / `keepout` / `edge` /
`thermal` / `separation` in any module body; declared constraints elaborate
into the L1 document and emit as `<name>.placement.json` with full per-row
provenance; the existing corpus (which declares nothing) emits
byte-identically to today. Shippable on its own: the artifact is useful to
a human laying out a board even before the Phase-2 checker exists.

### Step 1.1 — Semantics note + `.placement.json` contract proposal
**[dev-docs · doc-only · fable]** — *the design-sensitive step; everything
else in this plan is mechanical once this lands.*

Write `dev-docs/WyredPlacementSemantics.md`: the declaration semantics note
and the versioned contract proposal for `.placement.json`, in one document
(single-writer flow: proposal here, lands in `wyred-contract` separately).
It must settle, explicitly and with worked examples:

1. **Subject identity.** What `near(a, b, ...)`'s subjects name: role ids
   (dotted instance paths, law 8) referenced module-relatively and
   namespaced per instantiation — the `mutual_exclusion` precedent
   (declare `near("pmu", "mcu")` once inside a composite, get one
   constraint per instance of that composite). Also: how the checker joins
   asserted role ids to realized refdes — via the pin-map's
   `l1_role`/`for_demand` component attrs for wyred-emitted boards, and via
   an explicit, validated role→refdes correspondence table for exogenous
   boards (the Watchy demo). Silent fuzzy matching is not an option.
2. **Constraint identity minting.** Each constraint gets a minted id from
   its declaring attribute name namespaced by instance path
   (`case.near_usb`), so provenance, diffs, and (later) lock coverage have
   a stable handle — no author-written constraint names.
3. **Per-primitive semantics and measurement definitions** (the checker
   must be mechanical, so the *metric* is contract, not implementation):
   - `near(a, b, max_mm=)` — courtyard-to-courtyard nearest-edge distance
     vs center-to-center: pick one, state it, state the fallback when a
     footprint has no courtyard (pad bounding box, flagged in the result).
   - `keepout(zone=, roles=)` — what a *zone* is at intent time (no board
     outline exists at L1). Candidate semantics: a symbolic zone name that
     must match a named keepout/rule area on the realized board, with the
     listed roles' courtyards required to be outside it; a board lacking
     the named zone fails closed.
   - `edge(connector, side=)` — the side vocabulary (recommend
     `north/south/east/west` in the board file's axis frame), the frame
     definition (board-outline bounding box), and the "at the edge"
     tolerance (max courtyard-to-outline distance, a contract number).
   - `thermal(role, copper_mm2=)` — which copper counts: filled zones
     electrically connected to the net of the role's thermal/power pad(s),
     summed over which layers. State it precisely or narrow the primitive
     (e.g. v0 measures connected zone area on the pad's own layer only).
   - `separation(net_class_a, net_class_b, min_mm=)` — what a net class is
     at L1 (recommend: rail/ground names and role-set references, not a
     new decl, for v0), and clearance vs creepage: v0 is *clearance*
     (minimum straight-line copper-to-copper distance between the two
     classes' pads/zones); surface-path creepage is recorded as deferred.
     **The Tinkerforge EVSE board is the named eventual exercise for this
     primitive** — the proposal should sketch the EVSE mains/SELV split as
     its motivating worked example even though that board is not yet in
     the corpus.
4. **Say-only-the-delta.** Placement constraints have no defaults — each is
   a positive assertion; a module with none contributes nothing; an intent
   with none produces no L1 placement section and no `.placement.json`.
   `late(...)` resolves inside constraint attrs (numeric bounds may cascade
   from params) — `resolve_value` already recurses.
- 5. **Artifact schema.** Top-level shape of `.placement.json` (artifact
   name, stamp `(series, lock versions)` like the pin-map, constraint rows
   sorted by minted id), per-row provenance (constraint id, kind,
   declaring module class, instance path, resolved parameters), canonical
   JSON form (sorted keys, the number-formatting rule so byte-determinism
   holds for mm values), and the L1 `placement` section it derives from.
   Include one complete worked example artifact — Step 1.4 commits it as a
   fixture and Step 1.3's emitter must reproduce it byte-for-byte.
6. **Checker result codes** (reserved now, implemented in Phase 2):
   `PLACEMENT_NEAR_VIOLATION`, `PLACEMENT_KEEPOUT_VIOLATION`,
   `PLACEMENT_EDGE_VIOLATION`, `PLACEMENT_THERMAL_VIOLATION`,
   `PLACEMENT_SEPARATION_VIOLATION`, `PLACEMENT_SUBJECT_UNRESOLVED`,
   `PLACEMENT_GEOMETRY_MISSING` (fail-closed codes included).

*Acceptance (mechanical for a doc):* the note answers every question above
with a stated decision (not options); contains the complete worked-example
`.placement.json` and its source declaration snippet; names the contract
version bump it requests. Review sign-off recorded in the doc header.

### Step 1.2 — Marker declarations, harvest, elaboration into L1
**[wyred · ~250 LOC · opus-4.8, fable review of the surface]**
Depends on 1.1.

- `core.py`: `NearDecl` / `KeepoutDecl` / `EdgeDecl` / `ThermalDecl` /
  `SeparationDecl` (subclasses of `Decl`) + the `near()` / `keepout()` /
  `edge()` / `thermal()` / `separation()` markers, docstringed in the
  house voice; compose-time validation of vocabulary (e.g. unknown `side`
  is a structured `ModellerError`, like unknown lock decision classes).
- `engine.py`: harvest in `_walk` (namespacing module-relative subjects by
  instance path, exactly the mutex pattern), resolution of subject
  references against declared roles (dangling subject = structured load
  error), and a `placement` L1 section emitted via
  `_put(doc, "placement", rows, [])` so absence stays absent.
- The fable review is limited to the authoring surface: marker signatures,
  docstrings, and error vocabulary match the 1.1 semantics note.

*Acceptance:* `python3 -m wyred.emit --corpus-dir wyred-examples/corpus
--out <tmp>` then byte-compare against `wyred-contract/goldens/ga019/` —
identical, no new files; `run_gate.py` PASS end-to-end; the frozen 149
harness selftests PASS untouched; a minimal two-instance composite (test
scratch, not corpus) mints two independently-namespaced constraints and a
dangling subject raises the structured error.

### Step 1.3 — `.placement.json` emitter
**[wyred · ~200 LOC · opus-4.8]** Depends on 1.2 (and 1.1's schema).

- `emit.py` (with any row-building shared into `paths.py` alongside the
  other data paths): write `<name>.placement.json` **iff** the elaborated
  doc carries a placement section — the `.connlock.json`/`.baseline.json`
  only-when-applicable pattern. Rows carry the 1.1 provenance fields
  (`for_demand`-style: every row says which declaration demanded it and
  from which instance path), stamped like the pin-map.
- Covered by the existing double-emit determinism compare for free (the
  gate byte-compares the whole emitted tree), but the emit loop's own
  `paths_det` check extends to the new path when present.
- `rebuild.py`: `.placement.json` joins `python3 -m wyred.rebuild --all`
  as a secondary re-derivable from the on-disk L1 alone — the audit's
  `REBUILD_FROM_PRIMARIES` stage then covers it with zero audit changes.

*Acceptance:* golden byte-compare and `run_gate.py` PASS as in 1.2, plus an
explicit file-set assertion: emitting the current corpus produces **zero**
`*.placement.json` files. The 1.1 worked example reproduces byte-for-byte
from its source declarations. `python3 -m wyred.rebuild --dir <tree> --all`
PASSes on a tree containing a placement artifact.

### Step 1.4 — Probe corpus in a NEW test corpus
**[wyred-examples · ~150 LOC · opus-4.8]** Depends on 1.2 + 1.3.

- `wyred-examples/placement_corpus/` — a **separate** corpus package (the
  golden corpus `wyred-examples/corpus/` is not touched): one small intent
  per declaration kind (near, keepout, edge, thermal, separation), one
  composite-reuse case (two instances → two minted constraints), one
  `late()`-cascaded bound, one negative (dangling subject → load error).
- `wyred-examples/run_placement_probes.py` — a run_gate_tests.py-style
  driver: subprocess emit of the placement corpus into a temp dir,
  byte-compare `.l1.json` + `.placement.json` against committed expected
  fixtures (fixtures live next to the driver; they are contract-golden
  *candidates*, promoted to `wyred-contract/goldens/` only with the
  contract bump); assert the golden corpus still emits no placement files.

*Acceptance:* `python3 wyred-examples/run_placement_probes.py` PASS
(including the load-error probe firing its structured code);
`run_gate.py` PASS unchanged (the new corpus is not in its path).

## 3. Phase 2 — Verification (repos: `wyred-harness`, `wyred-examples`)

**Milestone:** a laid-out `.kicad_pcb` can be mechanically checked against
a `.placement.json` — the **placement-agreement oracle** — gate-wired in
run_gate whenever both an artifact and its board exist, demonstrated
against the real Watchy board with a full negative battery. Layout stays
delegated; conformance becomes mechanical.

### Step 2.1 — `pcb_extract` geometry extension
**[wyred-harness · ~300 LOC · opus-4.8]** Depends on nothing in Phase 1 —
**can start immediately, in parallel with 1.1.**

- Extend `pcb_extract.py` (or a sibling `pcb_geometry.py` sharing its
  parser) to additionally extract: per-footprint position + rotation
  (`(at x y rot)`), side (layer), courtyard polygons (`F.CrtYd`/`B.CrtYd`
  `fp_poly`/`fp_line`/`fp_rect`, with pad-bbox fallback flagged), board
  outline (`Edge.Cuts` `gr_line`/`gr_arc`/`gr_rect`/`gr_poly`, closed-loop
  assembly), filled zones (`zone` forms: net, layer(s), polygon, plus
  keepout/rule-area flags and names), and pad geometry (position, size,
  shape) for the separation metric. Both dialects (legacy `module` /
  modern `footprint`), same degradation-marker discipline for boards
  missing courtyards or a closed outline.
- Additive only: `CanonicalGraph` and every existing consumer unchanged;
  geometry rides in new structures keyed by refdes/net.

*Acceptance:* `python3 wyred-harness/harness/pcb_extract.py` smoke PASS
with new synthetic-dialect geometry checks added (both dialects) and new
real-board checks (Watchy + MPPT when the t9y tree is present: component
positions nonzero and within the board outline, outline closes, zone areas
positive); the frozen 149 selftests PASS; `run_gate_tests.py` PASS.

### Step 2.2 — Placement-agreement checker, gate-wired
**[wyred-harness + wyred-examples wiring · ~250 LOC · opus-4.8]**
Depends on 1.1 (schema + metrics) and 2.1. **Does not need 1.3:** it
consumes `.placement.json` from disk, so it is built and tested against
the 1.1 worked-example fixture — the artifact seam decouples the repos.

- `wyred-harness/harness/placement_agreement.py`: load a
  `.placement.json` + a board's extracted geometry + the subject-identity
  mapping (pin-map join or explicit correspondence table per 1.1), compute
  each constraint's 1.1-defined measurement, and report structured codes
  (the reserved `PLACEMENT_*` set) with measured-vs-declared values in
  every finding. Unresolvable subjects and missing geometry fail closed
  (`PLACEMENT_SUBJECT_UNRESOLVED` / `PLACEMENT_GEOMETRY_MISSING`).
- `run_gate.py`: a new stage after the board-agreement probes, run **iff**
  the artifact tree contains any `*.placement.json` AND a corresponding
  board exists (content-based readiness, same as `PROBE_REQUIRED_FILES`);
  a placement artifact with *no* board gets the loud SKIP banner naming
  what was not verified; no placement artifacts at all → one-line note,
  stage passes vacuously (this is what keeps today's gate green).

*Acceptance:* checker unit probes over hand-built fixture boards PASS (one
satisfied + one violated fixture per constraint class, every `PLACEMENT_*`
code demonstrated firing); `run_gate.py` on the current corpus PASS with
the vacuous one-liner; `run_gate_tests.py` PASS; 149 selftests untouched.

### Step 2.3 — Demonstration: Watchy placement + negative battery
**[wyred-examples · ~200 LOC · opus-4.8]** Depends on 1.2–1.4, 2.1, 2.2.

- Hand-author real placement constraints for the Watchy encoding in the
  **placement corpus** (e.g. `class WatchyV1Placed(WatchyV1,
  intent="watchy_v1_placed")` adding decls — subclassing reuses the golden
  encoding without touching it): USB port on an edge, antenna keepout,
  crystal near MCU, PMU near battery — constraints the real board actually
  satisfies.
- A `watchy_agreement.py`-style probe: emit the placement corpus, run the
  2.2 checker against the real `third-party/watchy/Watchy.kicad_pcb` (with
  the explicit role→refdes correspondence table, each row validated to
  exist with a plausible kind), exit 0 iff every declared constraint holds.
- **Negative battery (the R16/R25 / XPATH lesson: every code must fire or
  the run fails):** for each constraint class, a doctored extract (moved
  component, deleted zone, shrunk copper, narrowed clearance) must fire
  exactly its violation code; plus one unresolvable-subject probe firing
  the fail-closed code. All-must-fire, counted, or the probe run is red.
- Wire the probe into `run_gate.py`'s probe list with the same
  content-based readiness + ground-truth pin printing. The separation
  primitive's negative probe runs against a doctored fixture here; its
  *real-board* exercise is recorded as deferred to the **Tinkerforge EVSE**
  board joining the corpus (mains/SELV creepage-class separation).

*Acceptance:* the probe PASSes against the real t9y Watchy board with the
battery line printed (`N/N placement codes fired`); doctoring any single
fixture is caught; `run_gate.py` end-to-end PASS with the new stage RUN
(not skipped) when the t9y tree is materialized; goldens still
byte-identical (the demo lives entirely in the placement corpus).

## 4. Phase 3 — KiCad hint export (explicitly deferred / optional)

Translating placement declarations into KiCad placement-rule files or
plugin hints — so the layout tool can honor intent up front instead of
discovering violations at verification time — is deliberately **not
planned into steps**. It is pure convenience on the delegated side of the
fence: verification never depends on hints being honored (the oracle reads
the realized board regardless, Phase 2), so hints change no trust property
and earn no oracle. Build it demand-driven, when a real layout loop (the
tracer-bullet board, or the EVSE separation work) shows the round-trip
friction is worth the exporter; at that point it becomes a small
`wyred-kicad` step with the 2.2 checker as its own acceptance oracle
(hints honored ⇒ checker green on the resulting board).

## 5. Dependency map & parallelism

```
1.1 (fable: semantics + contract proposal) ──► 1.2 ──► 1.3 ──► 1.4 ──┐
   └────────────────────────► 2.2 ◄── 2.1 (parallel from day one)    ├─► 2.3
                                └────────────────────────────────────┘
```

- 2.1 has no Phase-1 dependency: start it alongside 1.1.
- 2.2 needs only 1.1's schema + fixtures, not the emitter — two agents can
  run the 1.2→1.4 chain and the 2.1→2.2 chain concurrently.
- 2.3 is the only join point, and Phase 3 has no steps at all.

## 6. Open questions Step 1.1 must settle (index)

Collected from §2/1.1 for tracking: subject-identity join (pin-map vs
explicit correspondence, and its failure mode); constraint-id minting;
near-distance metric + courtyard fallback; keepout zone identity at L1;
edge side vocabulary, frame, and tolerance; thermal copper accounting
(layers, connectivity); separation net-class referencing and clearance-vs-
creepage scope for v0; `late()` in constraint attrs; canonical JSON number
formatting for mm values; whether placement joins any lock decision class
now or later (recommend: later, with the contract bump that promotes the
fixtures to goldens).
