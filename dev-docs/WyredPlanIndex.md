# Wyred plan index — the six agent workstreams + coordination rules

*Companion to the six WyredPlan*.md docs (all elaborating
WyredWorkflowDesign.md). Each plan is one agent's workstream. This index
holds what no single plan can: cross-plan ordering, shared-file collision
rules, the aggregated approval/ratification queue, and the recorded
duplications. Reviewed 2026-07-12 (Fable pass over all six drafts).*

## The workstreams

| plan | scope | fable steps | everything else |
|---|---|---|---|
| [WyredPlanContractSchemas.md](./WyredPlanContractSchemas.md) | schemas + validator + findings F1–F9 | — | opus-4.8 |
| [WyredPlanKicadExport.md](./WyredPlanKicadExport.md) | L2→.net exporter, round-trip oracle, fab wrappers | 1.1 | opus-4.8 |
| [WyredPlanPlacement.md](./WyredPlanPlacement.md) | placement vocabulary → `.placement.json` → agreement oracle | 1.1 | opus-4.8 |
| [WyredPlanTestplan.md](./WyredPlanTestplan.md) | test vocabulary → `.testplan.json` → wyred-bench + comparator | 1.1 | opus-4.8 |
| [WyredPlanSpice.md](./WyredPlanSpice.md) | `.cir` data path + ngspice DC oracle | 1.1 | opus-4.8 |
| [WyredPlan3dRung1.md](./WyredPlan3dRung1.md) | wyred-klibs pinned forks + STEP/render delegation | — | opus-4.8 |

Out of scope by decision: wyred-3d rung 2 (awaits `Wyred3dDesign.md`), AI
visual inspection (parked).

## Ordering rules (the part that prevents collisions)

1. **Contract events are serialized, schemas first.** The schemas plan
   creates the EMIT_CONTRACT changelog section and Part E (finding F8/F1
   closure) that every other contract proposal appends to. Order:
   **schemas Phase 1+2.2 → kicad 1.1 → then placement/testplan/spice 1.1
   in any order.** All land via the single-writer flow.
2. **`wyred` engine edits are serialized across workstreams.** Placement
   1.2–1.3, testplan 1.2–1.3, and spice 1.2 all touch `core.py`,
   `engine.py`, `paths.py`, `emit.py`, `rebuild.py`. One workstream's
   engine steps land (and pass the golden byte-compare) before the next
   starts — or agents use worktrees with explicit rebase; never two agents
   in `wyred/src/wyred/` concurrently. Everything outside the engine
   (harness checkers, wyred-bench, wyred-kicad, wyred-klibs, schemas) is
   genuinely parallel.
3. **`run_gate.py` / `gate.py` edits are serialized** (kicad 2.2,
   placement 2.2–2.3, spice 2.2–2.3 all add stages/probes). Canonical
   stage order when all present: emit → gate (incl. SPICE op-oracle when a
   `.cir` exists) → audit → board-agreement probes → placement-agreement →
   kicad round-trip. Stages are *named*, not numbered — plans saying
   "stage 5" mean position-in-this-order, and every stage keeps the
   stage-when-present + loud-SKIP discipline.
4. **The golden census changes exactly once in this program.** SPICE 1.1/1.3
   adds `intent_10_spice_divider` (+ its artifact set incl. `.cir`) to the
   main corpus as a contract event. Acceptance commands in other plans that
   hardcode today's census (107 files, 16 l2 sets, 18 stems) are correct
   **before** that event; after it, read counts from the tree/manifest.
   Plans should not be "fixed" retroactively — this note is the record.
5. **GitHub account actions batch for approval** (see queue below); agents
   never execute them without the recorded go.

## Recorded duplications (accepted, with the honesty mechanism)

- `pcb_extract.py` / `kicad_adapter.py` / `schema.py` / `normalize.py`
  ported into `wyred_kicad/` (kicad plan 1.3/2.1) — the **third** home of
  this code (harness keeps its frozen copies; audit carries independent
  ports). Accepted per the fences; **mitigation to include in kicad 1.3:**
  a drift test asserting the ported files differ from the harness copies
  only in recorded ways (import lines + named deltas), so silent divergence
  is caught the F3 way.
- Placement 2.1 extends the **harness** `pcb_extract` with geometry; the
  wyred-kicad copy stays connectivity-only until the round-trip needs
  geometry — if that day comes, the geometry extension is re-ported, not
  imported, and joins the drift test.

## Aggregated approval / ratification queue (Gianni)

**Account actions (batch):** publish `owebeeone/wyred-t9y` (pending from
the t9y build); create `owebeeone/wyred-klibs`; fork `KiCad/kicad-symbols`,
`KiCad/kicad-footprints`, `KiCad/kicad-packages3D` as `klibs-*`; create
`wyred-bench` remote when testplan 2.1 runs.

**Contract ratifications, in landing order:**
1. Schemas staged proposal: `schemas/` + validator + findings **F1–F9**
   (+ any F10+), changelog section, Part E.
2. KiCad 1.1: `footprint` attr + `.net`/report conventions.
3. Placement 1.1: the ten placement semantics (subject-identity join is the
   load-bearing one) + `.placement.json`.
4. Testplan 1.1: **RATIFY-1…7** (tolerance semantics — the deliberate crack
   in the rigid-scalar model) + `.testplan.json` + measurement record.
5. SPICE 1.1: the ten `.cir`/`spice_model` decisions (passives auto-map vs
   explicit models is the silent-invention-risk one) + the golden census
   change.

**Standing from gen-4 (unchanged, still open):** the 13 spec-deviation
stances + 8 open items in `elecscad` (`rl/RESUME.md`), now joined by the
port-era DecisionLog entries already recorded here.

## Status (2026-07-13)

- [x] **Schemas Phase 1+2 — BUILT, verified, critic-fixed.** 107/107 goldens
      validate; oracle parity file-for-file (incl. 2,943-mutation fuzz, zero
      deltas); 12 committed mutant fixtures; audit `SCHEMA_INVALID` stage;
      Part E + changelog in EMIT_CONTRACT. Staged uncommitted in
      wyred-contract — **awaiting ratification** (findings F1–F12).
- [x] **All four fable 1.1 proposals DRAFTED, awaiting ratification** (in
      landing order): `ContractProposalKicadExport.md` +
      `wyred-kicad/EXPORT.md` · `WyredPlacementSemantics.md` ·
      `ProposalTestplanContract.md` · `WyredSpiceContract.md`.
- [x] **Kicad / placement / testplan / spice / 3D-rung-1 builds — STAGED
      COMPLETE (2026-07-13), awaiting commit-time ratification.** The
      wyred-buildout-phase1 workflow (22 agents, 0 errors) built all
      planned steps as uncommitted working tree; independent verify PASS
      (run_gate green end-to-end: 20 artifacts, battery 30/30, kicad
      round-trip 18/18, watchy placement 4/4, spice DC oracle on
      ngspice-46; sanctioned census: 107 goldens untouched + the 8
      intent_10_spice_divider files). Buildout-critic findings F1 (testplan
      signal-shape seam) / F2 (placement join-mode deferral recording) /
      F3 (this status) resolved 2026-07-13 — see DecisionLog.
- [x] 3D rung 1 — BUILT (wyred-klibs + threed wrappers), staged.
- [ ] Single-writer landings of the contract deltas + goldens additions —
      at Gianni's commit-time review, in the §"Contract ratifications"
      order above.
- [ ] `Wyred3dDesign.md` — next fable design task (rung 2).
- [ ] wyred-ui — separate track: `GrythWyredUiDesignPlan.md` ratified by
      Gianni (see DecisionLog); journeys in `WyredUiUserJourneys.md`.
