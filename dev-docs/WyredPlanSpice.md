# WyredPlanSpice — the SPICE data path (WorkflowDesign §3.4), phased

*Phased plan. Source design: [WyredWorkflowDesign.md](./WyredWorkflowDesign.md)
§3.4 ("SPICE data path — the third denotation"). Phases are shippable
milestones; each step is one goal with an **aspirational < 500 LOC** budget
(the per-step ~LOC below are targets, not limits). Per-step annotations:
**repo / ~LOC / model** (opus-4.8 for mechanical ports and emitters; fable
for contract/model-semantics design). Foundational-first: Phase 1 lands the
artifact and its structural trust before any simulator runs.*

## 0. Ground rules (non-negotiable invariants)

- **Emission gating — confession over absence.** A `<name>.cir` is emitted
  ONLY when (a) every component in the intent's L2 carries a resolvable
  spice model, or (b) the corpus **explicitly requests** SPICE emission for
  that intent, in which case the artifact carries a confessed
  `not_simulated` list naming EVERY unmodelled part. There is no silent
  absence and no silently partial deck: an unnamed missing model is a
  structured failure, never a quietly smaller circuit.
- **Goldens byte-identical.** No part in the frozen ga019 corpus carries a
  spice model, so the default gating means the existing emit tree is
  unchanged: a fresh emit of `wyred-examples/corpus` stays byte-identical
  to `wyred-contract/goldens/ga019/` at every step of this plan. New `.cir`
  goldens are **additive only** and land as a contract event (see 1.1) —
  regenerating or touching an existing golden is out of scope.
- **Contract via proposal.** Every schema change (the `spice_model` attr,
  the `.cir` artifact, the confession shape) is proposed in
  `wyred-wz/dev-docs/` and lands in `wyred-contract` via the single-writer
  flow. Nothing in this plan edits the contract as a side effect.
- **Delegation, not vendoring.** ngspice is reached by subprocess only,
  never imported, never vendored. Its availability is checked at runtime;
  absence produces a **loud skip banner naming exactly what was not
  verified** (the `run_gate.py _skip_probes_warning` pattern), never a
  silent green. When it runs, its version is pinned on stdout (the
  `_print_ground_truth_pin` / §3.1 KiCad-version-pin pattern).
- **Fences hold.** `wyred` never imports a checker; `wyred-harness` never
  imports the engine; composition is subprocess-only over artifacts on
  disk (RunnerSplit.md rule, DecisionLog F5). KiCad knowledge stays in
  wyred-kicad; SPICE emission knowledge lives in the engine's data-path
  layer, SPICE *verdict* knowledge in the harness.
- **No commits are authorized by this plan.** Work lands through the
  normal review flow; this document schedules work, it does not license
  git operations.

## 0.1 Prior art (gen 1 — locate before writing code)

The gen-1 experiments emitted SPICE as a second denotation and round-tripped
it. Skim these before 1.1/1.2; the format work is largely recoverable:

| path | what it holds |
|---|---|
| `/Users/owebeeone/limbo/elecscad/rl/ga005/emit_spice.py` | The gen-1 SPICE denotation emitter: element-letter map per component kind (`ELT`: source→V, R/C/D/M inline, everything structured→X subckt instance), ground net → node `0`, deterministic node numbering, unconnected terminal → own floating node, error/escalation contract emitted as deck comments. Connectivity-only by declared scope. |
| `/Users/owebeeone/limbo/elecscad/rl/harness/normalize.py` | `parse_spice` (§2, ~lines 81–314): the minimal deck **reader** back to a `CanonicalGraph` — logical-line continuation (`+`), `*` comments, `.subckt`/`.ends` skipping, element-letter dispatch, value parsing, source-volts recovery, node `0` → ground net, and the `STUBBED(SPICE)` confession note (prior art for the confession discipline: what SPICE cannot carry is escalated, never implied PASS). |
| `/Users/owebeeone/limbo/elecscad/rl/harness/roundtrip.py` | `agree()` — the two-engine differential the SPICE reader fed: every denotation of one design must normalize to LVS-equivalent connectivity. This is the shape 1.3 reuses. |
| `/Users/owebeeone/limbo/elecscad/rl/ga005/README.md` | Context: SPICE as the second denotation for the round-trip differential; no-silent-defaults framing. |

**Correction to the design doc's pointer:** `elecscad/rl/ga003/dsl/`
(engine.py / ir.py / library.py / reader.py) contains **no SPICE lowering**
— ga003's emitter is the neutral-JSON denotation only. The gen-1 SPICE work
lives in **ga005** plus the shared harness files above. WorkflowDesign §3.4's
"see `elecscad/rl/ga003`" should read ga005; noted here so the next reader
does not repeat the search.

Reusable conventions worth carrying forward: the kind→element-letter table
as a *normative contract table* (not per-part improvisation); ground → node
`0`; the reader/emitter pair as two independent implementations meeting at
the differential; confessions for everything the format cannot express.

## 0.2 Current-code anchors (wyred)

- `wyred/src/wyred/paths.py` — the data-path pattern to copy: pure-stdlib
  deterministic builders (`build_bom` / `build_pinmap` / `build_records`)
  over the SAME emitted documents (l2 + alloc + l1), never the modeller's
  internals; `crosscheck()` as the structured cross-path differential with
  `XPATH_*` codes; `json_str` canonical bytes.
- `wyred/src/wyred/parts.py` — `PartProto.attrs` / `register_builder`:
  where `spice_model` attrs enter; the resolver already materializes part
  attrs onto emitted L2 components, which is what makes the `.cir` a pure
  function of on-disk primaries (rebuild honesty, see 1.2).
- `wyred/src/wyred/emit.py` — the emit loop the `.cir` writer joins (emit
  twice, byte-compare); `wyred/src/wyred/rebuild.py` — the from-primaries
  rebuild the `.cir` must also join; `wyred/src/wyred/crosscheck.py` — the
  from-disk CLI the gate invokes as a subprocess.
- `wyred-harness/harness/gate.py` — `_xpath_probe_battery()`: the
  committed negative battery + lobotomy verdict the new `XCIR_*` codes must
  join (every probe fires every run or the run fails).
- `wyred-examples/run_gate.py` — the process-level composition + the loud
  SKIP banner pattern for delegated ground truth.

---

## Phase 1 — the emitter (milestone: `.cir` is a trusted structural artifact)

**Shippable increment:** every fully-modelled (or confessing) intent emits a
deterministic ngspice-dialect `.cir` that is a pure function of the on-disk
primaries, joins the cross-path differential with its own negatively-probed
failure codes, and survives the audit's rebuild-honesty check — before any
simulator exists. Value even if Phase 2 never runs: a third denotation any
external SPICE user can consume.

### Step 1.1 — Contract proposal: `.cir` conventions + `spice_model` schema + confession shape

- **Repo:** wyred-wz/dev-docs (proposal doc, e.g. `WyredSpiceContract.md`),
  landing in wyred-contract on acceptance. **~LOC:** doc only (+ small
  schema/golden additions landed by the single writer). **Model: fable**
  (contract/model-semantics design).
- **Goal:** one accepted proposal that settles every representational
  decision, so 1.2/1.3/2.x are mechanical. Decisions the proposal MUST
  answer (each with a chosen answer, not a menu):
  1. **`spice_model` attr shape** — a discriminated union on
     `PartProto.attrs["spice_model"]`: inline primitives for R/L/C/D/M
     (element letter + value/params) vs subcircuit refs for ICs
     (`subckt` name + model-file reference). Whether the element letter
     derives from the L2 `kind` via a normative contract table (ga005's
     `ELT` approach) or is declared per part — and what happens for kinds
     with no natural letter.
  2. **Passives policy** — do `kind`+`value` passives (the generated
     decoupling caps, pull-ups, load caps) auto-map to R/C/L/D cards via
     the contract table, or must every part carry an explicit model?
     Either way: no silent invention, and value canonicalization is
     specified (`"100nF"`, `"4.7k"`, `"SRC-12V"` → SPICE-legal tokens;
     subckt-name sanitization for values like `"MCU-3UART"`).
  3. **Node naming** — sanitized net names (ngspice accepts alphanumeric
     node names; keeps the deck human-debuggable and the differential
     cheap) vs ga005-style deterministic integers; ground → `0` mandatory;
     canonical element/node ordering so two emits are byte-identical.
  4. **Model-file resolution for subckt refs** — resolved like footprints
     (WorkflowDesign §3.1's planned `footprint=` attr): what a model ref
     names (qualified library key per the forward-plan Phase-2 naming
     story vs relative path), where model files live (wyred-parts / a
     t9y-style pinned tree), `.include`/`.lib` emission vs inlining model
     text into the deck (inlining keeps goldens self-contained and
     byte-stable; referencing risks drift — choose and justify).
  5. **`not_simulated` confession shape** — machine-readable inside the
     artifact (canonical JSON block in a structured comment header, or a
     `.cir.json` sidecar): every unmodelled refdes named, with its kind
     and (optionally) a reason; tamper-probeable by the battery.
  6. **Emission request channel** — how "explicitly requested" is declared:
     a corpus/intent declaration (e.g. an intent/refinement attr), NOT an
     emit CLI flag, so artifacts remain deterministic pure functions of
     the declared corpus.
  7. **Rebuild-honesty placement** — ratify that `spice_model` attrs are
     materialized onto emitted L2 component attrs at resolve, so the
     `.cir` is derivable from l2(+alloc) alone and joins
     `python3 -m wyred.rebuild` and the audit's `REBUILD_FROM_PRIMARIES`.
  8. **Declared-rail tolerance** — where the ± tol for the Phase-2 DC
     oracle lives (L2 `Net` attr, records, or library policy default with
     declared override). This is the first deliberate range in the
     contract (WorkflowDesign §3.3's "crack in the rigid-scalar model") —
     settle it here, once, not ad hoc in 2.2.
  9. **Simulator pinning** — where the ngspice version/dialect pin is
     recorded (deck header comment and/or the Phase-2 measurement record).
  10. **Golden additions** — the additive `.cir` golden set (and the new
      spice-modelled probe intent of 1.3) named explicitly, ga019 goldens
      untouched; EMIT_CONTRACT changelog entry + version bump text drafted.
- **Mechanical acceptance:** the proposal document exists in dev-docs and
  answers all ten items above with a single chosen design each; the
  drafted EMIT_CONTRACT delta (schema text + changelog entry + golden
  manifest) is attached; acceptance is recorded as a DecisionLog entry;
  the contract bump lands in wyred-contract by the single writer. No code.

### Step 1.2 — SPICE netlist emitter as a paths.py-style data path

- **Repo:** wyred. **~LOC:** ~300. **Model: opus-4.8** (mechanical, against
  the 1.1 contract). **Depends on:** 1.1 accepted.
- **Goal:** `build_cir(name, graph, alloc) -> str` in the `paths.py` style
  (pure stdlib, deterministic, consumes only the emitted documents), wired
  into the emit loop and the rebuild CLI.
  - Emit gating per invariant: all-modelled → emit; explicitly-requested →
    emit with the confessed `not_simulated` list; otherwise no `.cir`.
    A partially-modelled intent that neither qualifies nor confesses emits
    nothing and that is the *declared* behavior (the gate never sees a
    silently smaller deck).
  - Canonical bytes: sorted/stable element order, canonical node names,
    ground → `0`, stamp/provenance header comment (series + lock versions,
    same `_stamp` source as the JSON paths) — two emits byte-identical.
  - `python -m wyred.emit` writes `<out>/<name>.cir` (+ sidecar if 1.1
    chose one); `python -m wyred.rebuild` re-derives it from the on-disk
    primaries and byte-compares, same as bom/pinmap/records.
  - Port the ga005 `ELT` table / node conventions as the contract table
    from 1.1; carry the value-canonicalization rules.
- **Mechanical acceptance:**
  - `python3 -m wyred.emit --corpus-dir wyred-examples/corpus --out out`
    produces a tree **byte-identical to today's** (no ga019 part carries a
    model → zero `.cir` files → existing goldens untouched); full
    `run_gate.py` still PASSes end-to-end.
  - A committed spice-modelled mini-fixture corpus (unit-test scope)
    emits its `.cir` byte-identically across two emits and matches its
    committed golden; the explicitly-requested variant's confession names
    every unmodelled refdes; `python3 -m wyred.rebuild --dir <tree> --all`
    covers and passes the `.cir`.

### Step 1.3 — Structural oracle: `.cir` joins the cross-path differential

- **Repo:** wyred (crosscheck + minimal deck reader) + wyred-harness
  (battery probes) + wyred-examples (one new spice-modelled probe intent)
  + wyred-contract (its additive goldens, under the 1.1 contract event).
  **~LOC:** ~200. **Model: opus-4.8**. **Depends on:** 1.2.
- **Goal:** the `.cir` is *checked*, not trusted: a minimal deck reader
  (port of `parse_spice`'s subset — comments, continuations, element
  dispatch, subckt-instance connectivity) feeds new checks in
  `paths.crosscheck` / `python3 -m wyred.crosscheck`, and the gate's
  negative battery proves every new code fires.
  - **Checks (XPATH-style codes, e.g.):** `XCIR_COMPONENTS` (deck refdes
    set ≠ L2, minus the confessed `not_simulated` set — a confessed part
    absent is legal, an unconfessed absence or phantom element is not),
    `XCIR_NET_PARTITION` (deck node partition ≠ L2 net partition over the
    simulated subgraph — same LVS-lite relation as gen 1's `agree()`),
    `XCIR_ELEMENT` (element letter/value inconsistent with the component's
    kind/model per the contract table), `XCIR_CONFESSION` (the
    `not_simulated` list disagrees with which L2 components actually lack
    models — a forged or stale confession is a failure).
  - **Battery:** new probes in `_xpath_probe_battery()` doctoring a clean
    deck (drop an element, rewire one node, tamper the confession, rewrite
    a value) each naming the `XCIR_*` code that MUST fire; they join the
    lobotomy verdict (every probe fires every run).
  - **Applicability:** the gate fails when any probe never finds an
    applicable artifact, so the main examples corpus MUST contain at least
    one spice-modelled intent from this step on. Add one minimal probe
    intent (e.g. `intent_10_spice_divider`: source + resistive divider,
    fully modelled inline) to `wyred-examples/corpus` with its additive
    goldens (part of the 1.1 contract event). Conditional probe
    registration was considered and rejected: applicability conditioned on
    corpus content is exactly the hole the lobotomy verdict exists to
    close.
- **Mechanical acceptance:** `python3 -m wyred.crosscheck --dir out --all`
  covers `.cir` sets; full `run_gate.py` PASSes with the battery line
  counting the new probes (all fired); a `run_gate_tests.py`-style test
  (deliberately NOT wired into the frozen 149-check selftests, per the R3
  pattern) proves each `XCIR_*` code fires on its doctored deck and that a
  crosscheck lobotomy (checks blanked) turns the gate red; ga019 goldens
  bytes unchanged.

---

## Phase 2 — the behavioral rung (milestone: a DC operating point mechanically gates a rail)

**Shippable increment:** for any intent with a `.cir`, ngspice computes the
DC operating point and a mechanical oracle compares every declared power
rail against its simulated node voltage within the declared tolerance —
wired into the gate, proven by a corpus case including a committed negative.

**Repo choice, per fences (the "wyred-harness or examples" decision):** the
ngspice **runner and the operating-point oracle live in wyred-harness**
(2.1, 2.2): they are generic mechanical checkers computing verdicts from
contract artifacts re-read from disk (`.cir` + `l2`/records for declared
rails), with zero engine imports and zero corpus knowledge — exactly the
harness's charter ("depends on wyred-contract only; verdicts from artifacts
on disk"), and subprocess delegation to ngspice mirrors how `gate.py`
already reaches the engine. The **corpus case and its gate wiring live in
wyred-examples** (2.3): it is the integration point, the only repo allowed
to depend on everything, and the place where the R16/R25 lesson is law —
probes MUST be gate-wired. The engine (wyred) gains nothing in Phase 2:
simulation is a checker concern, and putting a runner in the engine would
let the emitter grade its own homework.

### Step 2.1 — ngspice subprocess runner + DC operating-point extraction

- **Repo:** wyred-harness. **~LOC:** ~200. **Model: opus-4.8.**
  **Depends on:** 1.1 (conventions only — can proceed against a
  hand-written conforming `.cir` fixture, in parallel with 1.2/1.3).
- **Goal:** `harness/spice_run.py` (name indicative): locate ngspice
  (PATH / env override), run `ngspice -b` batch mode on a `.cir` with an
  `.op` control section, parse node voltages into a structured dict, and
  write a measurement record (e.g. `<name>.spiceop.json`: node → volts,
  ngspice version string, deck sha256) — the asserted-vs-realized
  read-back, recorded not trusted.
  - **Availability = loud skip:** no ngspice → the banner names the
    artifact(s) not simulated and every check not verified (the
    `_skip_probes_warning` shape), distinct exit semantics from failure,
    never silent. Present → the version pin prints before any run.
  - Timeouts, nonzero-exit and unparseable-output are structured failures
    (a hung or crashed simulation is a red verdict on the run, not a
    skip).
- **Mechanical acceptance:** with ngspice installed, running the 1.2/1.3
  golden `.cir` (or the hand-written fixture) yields a measurement record
  whose node voltages match the analytically known values of the fixture
  divider; with ngspice removed from PATH (test harness manipulates env),
  the loud-skip banner appears, names the unverified checks, and the run's
  skip status is machine-distinguishable from PASS and FAIL; a corrupted
  deck produces a structured failure, not a skip.

### Step 2.2 — Operating-point vs declared-rails oracle, gate-wired when a `.cir` exists

- **Repo:** wyred-harness (oracle + gate wiring in `gate.py`).
  **~LOC:** ~150. **Model: opus-4.8.** **Depends on:** 2.1 + 1.2.
- **Goal:** for every artifact set with a `.cir`, map each L2 power net
  (declared `voltage`) to its deck node, and require the simulated DC
  value within the declared tolerance from 1.1 item 8 (the +3V3 net must
  sit at 3.3 V ± tol). New codes, e.g. `SPICE_RAIL_OFF` (out of
  tolerance), `SPICE_RAIL_UNRESOLVED` (a declared rail's node absent from
  the op results and not covered by the `not_simulated` confession —
  confession over absence, again).
  - **Gate wiring:** runs iff a `.cir` exists AND ngspice is available;
    ngspice absent → the 2.1 loud skip propagates into the gate output
    (named, counted, never green-by-omission); `.cir` absent → stage not
    applicable by design (Phase 1's gating already made absence honest).
  - Deeper analyses (transient, AC) are explicitly out of scope — the
    user's simulations are their own; this is the cheap sanity
    differential of WorkflowDesign §3.4.
- **Mechanical acceptance:** unit fixtures in the harness tests: an
  on-rail deck PASSes; a deliberately off-rail deck fires `SPICE_RAIL_OFF`
  naming net, declared volts, simulated volts, tol; a rail missing from
  the op output without confession fires `SPICE_RAIL_UNRESOLVED`; gate.py
  output shows the SPICE stage verdict per applicable artifact; with
  ngspice absent the full gate still completes with the loud skip and
  unchanged verdicts elsewhere.

### Step 2.3 — Corpus case: spice-modelled power tree + a committed negative

- **Repo:** wyred-examples (+ additive goldens in wyred-contract under the
  1.1 contract event). **~LOC:** ~200. **Model: opus-4.8.**
  **Depends on:** 1.3 + 2.2.
- **Goal:** prove the whole loop on a real corpus shape. In a **NEW probe
  corpus** (e.g. `wyred-examples/spice_probe/` — the frozen
  `corpus/lib_parts.py` and ga019 goldens are not touched), mirror
  `corpus/intent_02_power_tree.py` with spice-modelled parts: `Supply` →
  V source inline; `Ldo` → a subcircuit model (per 1.1 item 4) whose
  output voltage is set by a feedback divider inside the subckt; `Load` →
  R inline; the generated input/output caps map per the passives policy.
  Wire the probe corpus into `run_gate.py` as a gate-wired stage (emit →
  gate → spice), floors/expectations living IN the probe next to what it
  measures (the watchy/mppt probe pattern).
  - **The negative (lobotomy-proof):** a committed doctored variant —
    wrong divider value in the 3.3 V regulator's model → the +3V3 node
    sits off-rail → `SPICE_RAIL_OFF` MUST fire and MUST be that run's
    only failure; if it does not fire, the run fails. The negative runs
    every gate run ngspice runs (the battery discipline extended into
    physics-adjacent territory).
- **Mechanical acceptance:** full `run_gate.py` end-to-end PASS including
  the spice stage (positive corpus green: +3V3 = 3.3 V ± tol, +5V = 5.0 V
  ± tol, VIN = 12.0 V; negative fixture fires `SPICE_RAIL_OFF` and is
  verified to fire, every run); with ngspice absent the stage skips loudly
  and names both the unverified rails and the unexercised negative; ga019
  goldens bytes unchanged; the probe corpus's own goldens byte-identical
  across two emits.

---

## Phase 3 — analog showcase (deferred; pointer only)

Rung-1 scope from WorkflowDesign §3.4 applied to a real analog subcircuit:
give the MPPT half-bridge subset (`wyred-examples/corpus/lib_mppt.py` /
`mppt_2420_hc.py`; ground truth `third-party/libre-solar-mppt-2420-hc`)
spice models for the power stage — MOSFETs (M/subckt), gate drivers
(behavioral subckt), the LC output filter — with the digital side confessed
via `not_simulated`, and graduate from `.op` to a transient sanity check
(switch-node toggles, no shoot-through interval at the model level —
complementing the existing logic-level interlock invariant with an
electrical denotation). Deferred until the Phase-2 loop is proven and 1.1's
model-file-resolution story has survived contact with a subckt library of
real vendor models; scheduled by a future revision of this plan, not now.

---

## Dependency / parallelism summary

```
1.1 (fable, contract) ──► 1.2 (emitter) ──► 1.3 (structural oracle) ──► 2.3 (corpus + negative)
        │                                        ▲
        └──► 2.1 (ngspice runner, parallel) ──► 2.2 (op oracle) ────────┘
```

1.2 and 2.1 can proceed in parallel once 1.1 is accepted (2.1 needs only
the deck conventions plus a hand-written fixture). 1.3 and 2.2 are
independent of each other. 2.3 integrates everything.

## Open model-semantics questions (Phase 1.1 must settle; listed once)

The ten enumerated decisions in Step 1.1 — most consequential: (2) explicit
models for passives vs kind-table auto-mapping (silent-invention risk vs
authoring burden on every generated companion); (3) name-preserving nodes
vs numbered nodes (differential cost + debuggability vs gen-1 continuity);
(4) inline model text vs referenced model files (byte-stable self-contained
goldens vs a real library story — interacts with the forward plan's
qualified-naming Phase 2 and the future `footprint=` resolution); (5)
sidecar vs in-deck confession (both must be tamper-probeable); (8) where
the declared rail tolerance lives — the contract's first range.
