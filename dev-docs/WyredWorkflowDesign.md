# Wyred Workflow Design — intent to fab to test to 3D

*Design doc (no phasing — steps/milestones live in WyredForwardPlan.md when
scheduled). Covers the full workflow the g19++ track needs: KiCad export,
gerbers, SPICE, test definition, placement constraints, 3D models, and AI
visual inspection — cross-checked against the external reviewer's gap list in
`elecscad/rl/ga019/docs/g19 EDA Tool Comparison.md`.*

## 1. Principles (inherited, non-negotiable)

- **Delegate-don't-solve.** Wyred owns intent, derivation, and verification;
  layout/simulation/rendering engines are delegated (KiCad, ngspice) and
  reached by subprocess, never import.
- **Every stage gets an oracle.** A stage without a mechanical check is not
  built; it is deferred and recorded. New artifacts join the cross-path
  differential and the audit's from-disk re-derivation where applicable.
- **The contract is the seam.** Every new artifact type (`.testplan.json`,
  `.placement.json`, SPICE netlist, exporter outputs) is a versioned
  contract change: proposal in dev-docs → lands in wyred-contract.
- **Asserted vs realized.** Wyred declares intent and verifies realization;
  the realized world (routed PCB, measured board, rendered model) is always
  read back and compared, never trusted.

## 2. The workflow, end to end

```
intent (wyred modeller)
  └─ emit ─────────────► L1/L2/alloc/bom/pinmap/records/locks   [gate+audit: BUILT]
       ├─ kicad export ► .net + footprint bindings ─► KiCad layout (human/auto)
       │                                    │
       │                                    ├─► kicad-cli: DRC, gerbers, drill, pos
       │                                    ├─► pcb_extract ─► placement/agreement oracles
       │                                    └─► 3D: step/render (rung 1) · wyred-3d (rung 2)
       ├─ spice export ► .cir ─► ngspice ─► measured operating points
       └─ testplan ────► .testplan.json ─► bench card · firmware self-test
                                    └─► measured physical board  = FINAL DENOTATION
```

## 3. Stage designs

### 3.1 KiCad forward exporter (`wyred-kicad`) — the gating item

- **Emit:** L2 JSON → KiCad netlist s-expression (`.net`), with footprint
  bindings resolved from the parts library (parts gain `footprint=` attrs —
  a contract/parts-schema change). Refinements may pin footprints
  (`footprint` is already a lock-group decision class in the contract).
- **Gerbers:** after layout, pure delegation — `kicad-cli pcb export gerbers
  / drill / pos`. No wyred code beyond a convenience wrapper. DRC likewise
  delegated (`kicad-cli pcb drc --exit-code-violations`).
- **Oracle (round-trip):** export → import into a KiCad project →
  `kicad-cli sch export netlist` / `pcb_extract` → compare against the L2
  that produced it. This is a new two-engine differential in the existing
  gate style; it also permanently regression-tests the exporter.
- **Version pin:** KiCad major version recorded in the artifact (format
  drift is real; the corpus spans KiCad 4→10).

### 3.2 Placement constraints — roping in the reviewer's "spatial blindness" the wyred way

*(Already on the program's books: DslGen4 §8 deferred the
proximity/routability objective as "needs T2 placement".)*

- **Declare, don't route.** New L1 vocabulary, small: `near(a, b, max_mm=)`,
  `keepout(zone=, roles=)`, `edge(connector, side=)`, `thermal(role,
  copper_mm2=)`, `separation(net_class_a, net_class_b, min_mm=)` (creepage /
  isolation — the EVSE corpus board is the natural test case). Emitted as
  `.placement.json` with full provenance, same say-only-the-delta rules.
- **Verify from the realized board.** `pcb_extract` already parses
  `.kicad_pcb`; extend it to read positions/courtyards/zones and check every
  declared constraint against the laid-out board — a **placement-agreement
  oracle**, gate-wired like the board-agreement probes. Layout stays
  delegated; *conformance* becomes mechanical.
- **Hint export (later):** translate declarations into KiCad placement-rule
  files / plugin hints so the layout tool can honor them up front. Verification
  never depends on the hints being honored — the oracle reads the board.

### 3.3 Test definition (`.testplan.json` + `wyred-bench`) — the moat stage

- **Vocabulary:** `expect_rail(tp, volts, tol=)`, `expect_i2c_scan(bus,
  addrs=)`, `expect_current(rail, max_ma=, state=)`, `expect_signal(tp,
  freq=, duty=)`, testpoint roles (the Watchy corpus's `DebugTaps` already
  models these). Tolerances introduce **ranges** into the vocabulary — the
  first deliberate crack in the rigid-scalar model (see §4, continuous
  constraints).
- **Emit:** `.testplan.json` derived from declarations + records (rails,
  buses, addresses) + pinmap (which pad/testpoint carries what). Provenance
  per check: which declaration demanded it.
- **Generators (wyred-bench, planned member):** (a) human bench card
  (ordered checks, expected values, probe points); (b) firmware self-test
  scaffolding generated against the pinmap (the firmware-facing artifact was
  built for exactly this); (c) later: fixture/jig geometry via the 3D stage.
- **Oracle:** the measured board vs the testplan **is** the final exogenous
  denotation — the differential-oracle stack extended into physics. Results
  are recorded as a measurement artifact and compared mechanically; a failed
  check is a gate-red, same as any other disagreement.

### 3.4 SPICE data path — the third denotation

- **Emit:** L2 → `.cir` (ngspice dialect), a new `paths.py`-style emitter.
  Parts library gains `spice_model=` attrs (inline primitives for R/L/C/D/
  FETs; subcircuit refs for ICs, with model files resolved like footprints).
  Prior art: gens 1–3 emitted SPICE as a corpus denotation (see
  `elecscad/rl/ga003`); the format work is largely recoverable.
- **Scope rung 1:** analog subcircuits — the MPPT half-bridge, gate drive,
  filters. Digital ICs are stubbed (behavioral or omitted with a confessed
  `not_simulated` list — no silent absence).
- **Oracles:** (a) structural — the SPICE netlist joins the cross-path
  differential (same nodes/refdes as L2 or a code fires); (b) behavioral —
  ngspice operating point vs declared rails (the +3V3 node must sit at 3.3 V
  in DC analysis), a cheap sanity differential; deeper sims are the user's.

### 3.5 3D models — borrow first, then the anchorscad bridge

**Rung 1 — pure delegation (days, not weeks):**
- `kicad-cli pcb export step --include-tracks` (KiCad ≥ 8 exports board,
  copper, and placed component models) and `kicad-cli pcb render` for
  raytraced images from chosen viewpoints. Zero wyred code beyond wrappers.
- **Borrowed libraries — yes, legitimately:** KiCad's official libraries
  (symbols, footprints, `kicad-packages3D` STEP/WRL models) are
  **CC-BY-SA-4.0 with the KiCad libraries exception** — designs made with
  them carry no copyleft obligation; redistributing the libraries themselves
  requires attribution + share-alike, which a t9y-style pinned fork
  satisfies. `kicad-packages3D` is multi-GB → same treatment as wyred-t9y:
  pinned fork + sparse materialize-on-demand of only the models the parts
  library references. Footprint→model association ships in the footprints.
- This rung alone delivers "3D models of the actual boards".

**Rung 2 — `wyred-3d` (planned member): the pythonopenscad/anchorscad bridge.**
- Parse `.kicad_pcb` (pcb_extract heritage: outline, copper as polygons,
  drills, component positions/rotations) → **parametric 3D**: board substrate
  + extruded copper + placed component models (STEP/WRL converted or
  bounding-box proxies) as pythonopenscad/anchorscad objects.
- Why build it when rung 1 exists: (a) **enclosure co-design** — the board
  model becomes an anchorscad shape with anchors at connectors/mounting
  holes, reconnecting wyred to the AnchorSCAD mother project and the
  original "companion holes" vision (a case that needs a hole where the USB
  connector is gets it *derived*, with provenance); (b) deterministic
  headless renders under our control for the inspection loop; (c) fixture/
  jig generation for wyred-bench (pogo-pin plates located from testpoint
  coordinates — the testplan and the 3D model share provenance).

### 3.6 AI visual inspection (`wyred-agent`)

- Feed rendered views (rung 1 `kicad-cli pcb render`, later wyred-3d) plus
  the artifacts (netlist, placement.json, testplan) to a multimodal model
  with structured prompts: orientation sanity, silkscreen legibility,
  connector accessibility, obvious placement pathologies, "does the board
  match its BOM visually".
- **Status: advisory, not gate.** An LLM verdict is not a mechanical oracle;
  findings are review comments, never exit codes. If specific checks prove
  reliable and mechanizable (e.g. fiducial presence), they graduate to real
  oracles individually. This keeps the trust discipline intact.
- Same channel later serves **real AOI**: camera photos of the fabbed board
  vs renders — closing visual inspection against the physical denotation.

## 4. Reviewer's gap list — coverage map

| Comparison-doc shortcoming | Addressed here? | How / why not |
|---|---|---|
| Spatial geometry & routing absent | **Partially, deliberately** | Placement *constraints* declared + verified (§3.2); routing stays delegated forever — entrenched competitors, wrong moat. Gerbers/DRC via kicad-cli (§3.1). |
| No continuous constraint solver | **Cracked open, not solved** | Tolerances/ranges enter via testplan (§3.3) and placement (`max_mm`, `min_mm`). The full solver rung (AF matrices, UNSAT cores) remains Path B in the forward plan, demand-driven by a real board. |
| No live supply-chain integration | **Out, recorded** | Delegate-don't-solve; a parts-library backend adapter (stock/MPN enrichment at BOM time, off the trust path) can bolt on later without touching intent. Unchanged from the forward plan. |
| No visual feedback / iteration | **Yes** | §3.5 rung 1 renders + §3.6; wyred-ui reads the same artifacts. |
| Static manual part libraries | **Partially** | Borrowed KiCad libs give breadth of footprints/3D (§3.5); the packaging/qualified-naming story (forward plan Phase 2) remains the real fix. |
| Simulation absent | **Yes** | §3.4 SPICE path, scoped honestly. |
| Test/production loop absent | **Yes — and it's the moat** | §3.3; no competitor closes intent→measured-board. |

## 5. New oracles introduced (summary)

| Stage | Oracle | Gate-wired? |
|---|---|---|
| KiCad export | round-trip re-import differential | yes, with exporter |
| Placement | placement-agreement from pcb_extract positions | yes, when a laid-out board exists |
| SPICE | cross-path structural + operating-point vs declared rails | structural yes; behavioral per-board |
| Testplan | measured board vs plan (physical denotation) | yes — the point |
| 3D / inspection | none (advisory) | no — explicit non-oracle until checks mechanize |

## 6. Open decisions

- Footprint/spice-model attr schema shape in the parts library → contract
  bump proposal (single-writer flow).
- KiCad version floor (≥ 8 for STEP tracks/render; corpus extraction already
  handles older formats read-only).
- kicad-libs t9y-fork scope: full pinned forks vs sparse per-part fetch
  (leaning: fork symbols+footprints wholesale, sparse-materialize packages3D).
- Which board is the tracer bullet (drives which vocabulary lands first —
  EVSE would exercise separation/creepage; a personal board exercises
  authorability).
- STEP→pythonopenscad conversion route for rung 2 (cadquery/OCC import vs
  mesh proxies) — decide when rung 2 starts.
