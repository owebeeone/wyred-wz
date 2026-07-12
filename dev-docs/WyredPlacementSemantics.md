# Placement semantics + `.placement.json` contract proposal (WyredPlanPlacement step 1.1)

*Status: DRAFT awaiting Gianni's ratification. Single-writer flow: lands in
wyred-contract (version bump + changelog entry) after the schemas proposal
and the KiCad proposal, per WyredPlanIndex ordering. Every question from the
plan's §6 is answered with a decision, not a menu. Checker result codes were
reserved in the plan and are restated in §8 unchanged.*

## 1. Subject identity (the load-bearing decision)

Constraint subjects name **role ids**: module-relative dotted attribute
paths, namespaced by instance path at elaboration — the `mutual_exclusion`
precedent. `near("pmu", "mcu")` declared inside a composite mints one
constraint per instance of that composite, its subjects fully qualified
(`main.pmu`, `main.mcu`). A subject that names no declared role in scope is
a compose-time `ModellerError` (dangling reference, law 10).

**Join to realized refdes:**
- **Wyred-emitted boards:** subject role id → the set of pinmap components
  whose `l1_role` attr equals the role id exactly. `for_demand`-generated
  companions are NOT subjects (they follow their anchor part in layout by
  their own logic; constraining them is a separate declaration if wanted).
  **Build status (recorded 2026-07-13, critic F2): SPECIFIED HERE, NOT YET
  BUILT.** The v0 checker implements only the correspondence-table join
  below; this pinmap-`l1_role` join is deferred until a laid-out
  wyred-emitted board exists to exercise it (no such board exists — the
  only realized boards are exogenous). Invoking the checker without a
  table fails closed (`PLACEMENT_SUBJECT_UNRESOLVED`). Ratifying this
  section ratifies the *semantics*; the deferral is tracked here and in
  DecisionLog, not in code comments alone.
- **Multi-part roles:** the constraint evaluates over **all** matched
  components — for `near(a, b)`, all cross pairs; each pair reported, all
  must hold (conservative, fails closed).
- **Empty match** → `PLACEMENT_SUBJECT_UNRESOLVED` (fail closed).
- **Exogenous boards** (the Watchy demo): an explicit correspondence table
  JSON `{role_id: [refdes, ...]}` supplied to the checker. Validated both
  ways before any measurement: every refdes must exist in the extract,
  every role id must exist in the placement artifact; any miss is
  `PLACEMENT_SUBJECT_UNRESOLVED`. **No fuzzy matching, ever.**

## 2. Constraint identity

Minted, never authored: `<instance path>.<declaring attribute name>`
(`case.near_usb`). Uniqueness is structural (attribute names are unique per
class body; instance paths are unique by law 8). Artifact rows sort by id.

## 3. Per-primitive semantics (the metric IS the contract)

All distances in mm, measured in the board plane (xy of the board file);
board side is ignored in v0 (recorded limitation — a top/bottom pair
measures in xy projection).

- **`near(a, b, max_mm=)`** — **courtyard-to-courtyard nearest-edge
  distance** (minimum gap between the two courtyard polygons; 0.0 when they
  touch or overlap). Center-to-center was rejected: it punishes large
  packages nonsensically. **Fallback** when a footprint has no courtyard:
  its pad bounding box, and the result row carries
  `metric_fallback: "pad_bbox"` — measured, flagged, never silent.
- **`keepout(zone=, roles=)`** — `zone` is a **symbolic name** that must
  match (exact string) a named keepout/rule area on the realized board.
  Semantics: no listed role's courtyard (fallback as above) may intersect
  the named area's polygon. A board with no rule area of that name →
  `PLACEMENT_GEOMETRY_MISSING` (fail closed — an unnamed keepout is an
  unverifiable keepout).
- **`edge(connector, side=, tol_mm=1.0)`** — frame is the **board-outline
  bounding box in board-file coordinates**; KiCad's y axis grows downward,
  so: `north` = min-y edge, `south` = max-y, `west` = min-x, `east` =
  max-x (stated here precisely because the y-inversion is a classic trap).
  Semantics: the subject's courtyard nearest-edge distance to the named
  bounding-box edge ≤ `tol_mm`. `tol_mm` is a **declared vocabulary
  default** (1.0 mm, recorded here, overridable) — a default value, not a
  silent one. Side vocabulary is the closed enum {north, south, east,
  west}; anything else is a compose-time load error.
- **`thermal(role, copper_mm2=)`** — v0 narrow scope: sum of **filled-zone
  areas** whose zone net equals the net of the subject's pads with terminal
  role `power_out` (or `power_in` when the subject has no `power_out`
  pads), counted **on the layers those pads sit on** only. Traces and plane
  layers the pads don't touch are out of v0 scope (recorded). A subject
  with no power-role pads → `PLACEMENT_SUBJECT_UNRESOLVED`.
- **`separation(class_a, class_b, min_mm=)`** — classes are **declared rail
  / ground names** from the L1 vocabulary (strings; a name that matches no
  declared rail/ground is a compose-time load error). v0 metric is
  **clearance**: minimum straight-line copper-to-copper distance between
  the two classes' **pads and filled zones**. Track segments are excluded
  in v0 and every separation result row confesses
  `copper_scope: "pads+zones"` — the scope is in the result, not the fine
  print. Surface-path **creepage is deferred**, recorded. Named eventual
  exercise: the Tinkerforge EVSE mains/SELV split (its motivating example:
  `separation("VAC_L", "GND", min_mm=6.0)` across the isolation boundary).

## 4. `late()` and say-only-the-delta

`late("field")` is legal in every numeric/string constraint parameter
(resolve_value already recurses) — parent params cascade into bounds. No
constraint has an implicit default *presence*: a module with no placement
declarations contributes nothing; an intent with none emits no L1 placement
section and no `.placement.json` (the only-when-declared invariant). The
single declared value default is `edge.tol_mm` (§3), recorded here.

## 5. The `.placement.json` artifact

Canonical JSON (house rule: `json.dumps(indent=2, sort_keys=True)` +
trailing newline). Authored numbers pass through **verbatim** (they are
deterministic from source; no derived numerics exist in this artifact —
all measurement is checker-side, and the checker reports
`round(measured, 4)` in its own results, which are never byte-goldened).

```json
{
  "artifact": "<name>",
  "path": "placement",
  "stamp": {"series": "A", "locks": {}},
  "constraints": [
    {
      "id": "usb_edge",
      "kind": "edge",
      "declared_by": {"module": "UsbAtEdge", "instance": ""},
      "subjects": ["usb"],
      "params": {"side": "south", "tol_mm": 1.0}
    },
    {
      "id": "near_pmu_mcu",
      "kind": "near",
      "declared_by": {"module": "UsbAtEdge", "instance": ""},
      "subjects": ["pmu", "mcu"],
      "params": {"max_mm": 8.0}
    }
  ]
}
```

Worked-example source (step 1.4 commits both as the fixture pair; the 1.3
emitter must reproduce the artifact byte-for-byte from this source):

```python
class UsbAtEdge(Module, intent="placement_example"):
    v33 = rail("+3V3", 3.3)
    gnd = ground("GND")
    usb = use(UsbPort)
    pmu = use(WatchyPmu)
    mcu = use(WatchyMcu)
    usb_edge = edge("usb", side="south")          # tol_mm default 1.0
    near_pmu_mcu = near("pmu", "mcu", max_mm=8.0)
```

L1 doc: a `placement` section carrying the same rows (minted ids,
subjects, resolved params), emitted via the omit-when-empty pattern.

## 6. Checker result shape (Phase 2 consumes; fixed here)

Per constraint: `{id, kind, verdict: PASS|FAIL|UNRESOLVED, declared:
{...params}, measured: {value_mm | area_mm2 | distance_mm, per_pair: [...]},
flags: [metric_fallback?, copper_scope?], code}` — measured values present
on FAIL as well (the finding names measured-vs-declared, XPATH discipline).

## 7. Lock coverage

**Deferred** (question 10): placement joins no lock decision class in v0.
The candidate class (`placement`) is reserved in prose only; it enters
`DECISION_CLASSES` with the contract bump that promotes the placement
fixtures to goldens, if ratified then.

## 8. Result codes (reserved in the plan, restated)

`PLACEMENT_NEAR_VIOLATION`, `PLACEMENT_KEEPOUT_VIOLATION`,
`PLACEMENT_EDGE_VIOLATION`, `PLACEMENT_THERMAL_VIOLATION`,
`PLACEMENT_SEPARATION_VIOLATION`, `PLACEMENT_SUBJECT_UNRESOLVED`,
`PLACEMENT_GEOMETRY_MISSING`. The first five are per-kind violations; the
last two are the fail-closed pair and are gate-red like any violation.

## 9. Ratification checklist

- [ ] subject identity: role-id join, multi-part all-pairs, no-fuzzy rule
- [ ] metrics as specified (esp. courtyard nearest-edge; KiCad y-inversion
      edge naming; separation v0 = clearance over pads+zones, confessed)
- [ ] `edge.tol_mm = 1.0` as the single declared vocabulary default
- [ ] artifact schema + verbatim-authored-numbers rule
- [ ] lock class deferred
- [ ] contract version bump text + changelog entry (drafted at landing)
