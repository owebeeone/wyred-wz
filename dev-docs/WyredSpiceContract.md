# `.cir` conventions + `spice_model` schema proposal (WyredPlanSpice step 1.1)

*Status: DRAFT awaiting Gianni's ratification (lands in wyred-contract
after the schemas/kicad proposals; this one also carries the program's ONE
sanctioned golden-census change — WyredPlanIndex rule 4). Prior art:
ga005's `emit_spice.py` conventions (kind→letter table, ground→0,
deterministic ordering) and the harness `parse_spice`/`roundtrip.agree()`
reader-differential pair. Each numbered item below is a decision.*

## 1. `spice_model` attr — discriminated union

On `PartProto.attrs["spice_model"]`, materialized onto emitted L2
component attrs at resolve (item 7):

```json
{"model": "primitive", "letter": "R|C|L|D|V|I", "value": "<canonical>", "params": {}}
{"model": "subckt", "name": "<SPICE-legal subckt name>", "text": "<inline .subckt ... .ends>"}
```

Element letters for primitives are DECLARED (part of the attr), but must
agree with the normative kind table (item 2) where the kind is tabled —
disagreement is `XCIR_ELEMENT`. MOSFETs and anything with more than a
2-terminal primitive shape are `subckt` in v0 (M-card pin-order traps are
not worth primitive treatment yet).

## 2. Passives policy — normative kind table, no per-part authoring

Components of kind `resistor`, `capacitor`, `inductor`, `diode` **auto-map**
via a closed contract table (kind → letter R/C/L/D) with `value` taken
verbatim from the L2 component (canonicalized per item 2a). Rationale over
explicit-everywhere: generated companions (decoupling caps, pull-ups) get
models with zero library churn; the invention risk is nil because the
table maps only KIND→LETTER while the VALUE is already oracle-checked by
the BOM crosscheck (the R16/R25 ghost is guarded by existing machinery),
and `XCIR_ELEMENT` verifies letter-vs-kind on every deck. Everything not
in the table requires an explicit `spice_model` or lands in the confession.

**2a. Value canonicalization:** L2 value strings → SPICE magnitudes:
`"100nF"` → `100n`, `"4.7k"` → `4.7k`, `"10uF"` → `10u` (strip unit
letters F/H/Ω/R-suffix conventions; MEG for 1e6 — `"4M7"`-style is out of
scope v0). A value that does not parse → the part is **not modelled**
(joins the confession); never a guessed number.

## 3. Node naming — name-preserving, sanitized; ground → `0`

Net names carry into the deck sanitized: `+` → leading `P` (`+3V3` →
`P3V3`), `-` → `M`, any other non-`[A-Za-z0-9_]` → `_`, digit-leading
names prefixed `N` (`3V3` → `N3V3`). Ground nets (kind `ground`) map to
node `0` unconditionally. Sanitization collisions (two nets mapping to one
token) are a hard emit error — deterministic, but refused rather than
suffixed, so the mapping stays bijective and the differential stays cheap.
The deck header comments record the net→node map for every non-identity
mapping. Rationale vs ga005's numbered nodes: human-debuggable decks and a
trivially checkable partition differential.

## 4. Model files — INLINE in v0

Subckt model text is carried **inline** in the attr and emitted into the
deck (sorted, deduplicated by subckt name; duplicate names with differing
text = emit error). Referenced model *files* (`.include`/`.lib`, qualified
library keys) are **deferred** to the wyred-parts qualified-naming
workstream — inline keeps `.cir` goldens self-contained, byte-stable, and
a pure function of on-disk primaries. The `model_ref` field name is
reserved now so the file story is additive later.

## 5. Confession — canonical JSON sidecar

`<name>.cir.json`: `{artifact, path: "cir_confession", stamp,
modelled: {refdes: "R|C|...|subckt:<name>"}, not_simulated:
[{refdes, kind, reason: "no_model"|"value_unparseable"}], node_map:
{net: node}}` — machine-readable, canonical, tamper-probeable
(`XCIR_CONFESSION` compares it against the L2 + the deck every crosscheck
run). The deck itself carries a one-line header comment
`* not_simulated: <n> (see <name>.cir.json)` so a human reading the deck
can't miss it.

## 6. Emission request channel — corpus data, never a CLI flag

Class attr on the intent root: `emit_spice = True` requests emission with
confession for partially-modelled intents; fully-modelled intents emit
unconditionally; neither → no `.cir`. This keeps artifacts a deterministic
pure function of the declared corpus (no flag-dependent trees).

## 7. Rebuild honesty — models ride the L2

`spice_model` attrs (and the auto-map table's applications) materialize
onto emitted L2 component `attrs`, so `.cir` + `.cir.json` are derivable
from `(l1, l2, alloc)` alone and join `python3 -m wyred.rebuild` and the
audit's `REBUILD_FROM_PRIMARIES`. (Same materialization rule the KiCad
proposal establishes for `footprint` — one mechanism, three consumers.)

## 8. Declared rail tolerance — the SIM tolerance, third and distinct

The Phase-2 DC oracle compares each declared power net's simulated node
voltage against its L2 `voltage`. Tolerance: **contract default ±2 %**,
overridable per rail via rail attr `sim_tol_pct`. Explicitly distinct from
(a) `VTOL` (design-time float noise) and (b) testplan measurement
tolerances (RATIFY-3 firewall) — three numbers, three meanings, named
`sim` to keep them unconfusable. `SPICE_RAIL_OFF` findings report
declared/simulated/tolerance; `SPICE_RAIL_UNRESOLVED` covers a declared
rail absent from the op results and not confessed.

## 9. Simulator pinning

Deck header comment records the emitting engine + contract rev; the
Phase-2 measurement record (`<name>.spiceop.json`) records the ngspice
version string and deck sha256. Absence of ngspice is a loud skip naming
every unverified rail (never silent green).

## 10. Golden census change (the one sanctioned event)

One new corpus intent `intent_10_spice_divider` (V source + resistive
divider + load, fully auto-map-modelled) lands in `wyred-examples/corpus`
with its full additive artifact set (incl. `.cir` + `.cir.json`) in
`wyred-contract/goldens/ga019`. Existing goldens untouched byte-for-byte.
After this event, census-dependent acceptance counts read from the tree
(WyredPlanIndex rule 4). EMIT_CONTRACT changelog entry + version bump
drafted at landing; `XCIR_*` codes (`XCIR_COMPONENTS`, `XCIR_NET_PARTITION`,
`XCIR_ELEMENT`, `XCIR_CONFESSION`) join the battery with committed probes.

## Ratification checklist

- [ ] union shape + subckt-for-MOSFETs v0 (§1)
- [ ] passives auto-map table + value canonicalization + refusal rule (§2)
- [ ] sanitized name-preserving nodes, bijection-or-refuse (§3)
- [ ] inline models v0, files deferred (§4)
- [ ] sidecar confession shape (§5) · request channel (§6) · L2 materialization (§7)
- [ ] sim tolerance ±2 % default + `sim_tol_pct` override (§8)
- [ ] the golden census event (§10) — this is the big one
