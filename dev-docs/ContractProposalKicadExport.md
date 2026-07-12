# Contract proposal — KiCad export (WyredPlanKicadExport step 1.1)

*Status: DRAFT awaiting Gianni's ratification. Single-writer flow: this
proposal lives in dev-docs; the contract delta below lands in
`wyred-contract/EMIT_CONTRACT.md` (version bump + changelog entry) only on
acceptance — and only AFTER the schemas workstream's changelog section
exists (WyredPlanIndex ordering rule 1). The KiCad-specific conventions are
in `wyred-kicad/EXPORT.md` (drafted alongside this; lands with acceptance).*

## 1. Contract delta (EDA-neutral — the only EMIT_CONTRACT change)

**New optional L2 component attr: `attrs["footprint"]`.**

- Type: string. An **opaque qualified binding token**; wyred assigns no
  semantics beyond identity. Recommended shape `<library>:<name>`
  (e.g. `Resistor_SMD:R_0402_1005Metric`) because that is what downstream
  EDA adapters consume, but the contract constrains only "non-empty string".
- Emitted **only when set** (the v3 ground-fields pattern): absent for every
  component of the frozen ga019 corpus, so **no golden changes** — this is
  a purely additive, backward-compatible delta. The existing oracle stack
  ignores it (free `attrs` dict).
- Rationale: `footprint` is already a decision class in
  `schema_l1.DECISION_CLASSES` — pinned/locked footprint decisions need a
  place to surface in the emitted L2. This attr is that place.
- **Future source recorded, out of scope here:** the wyred-parts workstream
  will let part definitions carry `footprint=` attrs that the resolver
  materializes onto emitted L2 components (the same materialization rule the
  SPICE proposal ratifies for `spice_model` — one mechanism, two consumers).
  Until then the exporter's sidecar map (below) is the interim binding
  source and the L2 attr is simply legal.
- Schema impact: one optional property added to `l2.schema.json`'s component
  `attrs` documentation (attrs is an open dict; the schema change is
  descriptive text + the changelog entry, not a structural change).

## 2. Export/round-trip report vocabulary (contract-adjacent, fixed here)

The exporter and its oracle communicate through a per-stem report sidecar
(`<stem>.kexport.json`, shape owned by wyred-kicad/EXPORT.md §5). The
**structured codes** are fixed by this proposal so Phase 2 is mechanical:

| code | tier | meaning |
|---|---|---|
| `KEXPORT_PARSE_FAIL` | A (gate-red) | an emitted `.net` fails parse-back |
| `KEXPORT_ROUNDTRIP_DISAGREE` | A | KiCad-realized board disagrees with source L2 (partition/refdes/pin set) |
| `KEXPORT_UNKNOWN_ROLE` | A | an L2 terminal role outside `TERMINAL_ROLES` reached the exporter (fail closed, never a silent `passive`) |
| `KEXPORT_FOOTPRINT_STUBBED` | B (confessed) | a component exported with the placeholder footprint; every instance named |
| `KEXPORT_NET_NAME_ESCAPED` | B | a net name required deterministic escaping; original + escaped recorded |
| `KEXPORT_TOOLCHAIN_UNAVAILABLE` | B | KiCad absent; round-trip not performed (loud skip, never silent green) |

Tier A codes are nonzero exits wherever they fire and gate-red when the
stage is wired into run_gate. Tier B codes are confessions: recorded,
counted, loudly printed, never exit-affecting on their own.

## 3. Footprint resolution ladder (normative order, EXPORT.md holds detail)

Per component, first hit wins; the outcome (`source`) is recorded per refdes
in the report: **(1)** pinned/locked footprint decision (lock class
`footprint`) → **(2)** L2 `attrs["footprint"]` → **(3)** sidecar map,
per-refdes override → **(4)** sidecar map, kind-level default → **(5)**
confessed placeholder (`wyred_stub:<kind>`) + `KEXPORT_FOOTPRINT_STUBBED`.
No silent defaults: rung 5 is always loud.

## 4. Ratification checklist

- [ ] `attrs["footprint"]` delta as specified (opaque token, emit-only-when-set)
- [ ] code vocabulary + tiers (§2)
- [ ] resolution ladder order (§3)
- [ ] EXPORT.md conventions (companion doc) — dialect, determinism, mappings
- [ ] landing order: after the schemas proposal's changelog section exists
