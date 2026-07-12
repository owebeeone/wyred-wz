# WyredUiDesign — the artifact viewer / review surface

> **Stack section (§7) SUPERSEDED (2026-07-13):** the UI is built on
> **gryth-ui** per Gianni's direction — see `WyredUiUserJourneys.md` (the
> requirements source) and the DecisionLog entry. §2 trust posture, §3.1
> ArtifactStore seam (now the glade doc-scope provider), and the §4
> surface inventory (now the plugin catalog) all still stand.

*Design doc (design only — the phased plan is a separate doc when scheduled).
Author context: written after the full workflow buildout, so the surface
inventory below covers every artifact kind the tree actually produces —
the ten contract kinds plus testplan, placement, cir/confession, kexport
reports, spiceop, fab manifests, and render/STEP outputs.*

## 1. What this is (and is not)

**wyred-ui is the review surface for wyred artifact trees.** A designer,
firmware engineer, or reviewer points it at an emitted `out/` directory (or
the goldens) and *reads* what the engine did and what the gates concluded:
what was generated and **why**, what got allocated where, what is locked,
what changed between revisions, and what the verdicts were.

**Non-goals (v1, explicit):** no authoring (intent is Python; the IDE is
the editor); no running (emit/gate/audit are CLIs; the UI never spawns
processes); no live board data; no schematic-drawing ambitions (KiCad owns
geometry — we render *logical* structure and *decisions*).

## 2. Trust posture — render what was emitted

The fence (`wyred-ui/CLAUDE.md`) is a trust property, same family as the
engine/harness split: **a viewer that re-derives is lying.** Rules:

- Every displayed value traces to a byte in an artifact file. The UI never
  recomputes an allocation, never re-runs a crosscheck, never "fixes up" a
  missing field. Absence renders as absence, loudly.
- **Schema validation is the one permitted computation** (it is checking,
  not deriving): every loaded file is validated against
  `wyred-contract/schemas/` in the browser; invalid files render with a
  prominent `SCHEMA_INVALID` banner and a raw-JSON fallback view — never a
  best-effort pretty view of corrupt data.
- Verdict-shaped fields the engine self-reports (`lifecycle.ok`,
  `connlock.ok`, `minimal_disturbance_violations`) are rendered **with
  their provenance labeled** ("engine-reported; audit counter-checks this")
  — the UI teaches the trust model instead of flattening it.

## 3. Data model

### 3.1 The ArtifactStore seam (the one abstraction that matters)

```ts
interface ArtifactStore {
  census(): Promise<ArtifactSet[]>;      // stems × kinds actually present
  load(stem: string, kind: Kind): Promise<Json>;   // raw parsed file
  raw(stem: string, kind: Kind): Promise<string>;  // exact bytes (hex/raw view)
  watch?(cb: (change: Change) => void): () => void; // optional live reload
}
```

v1 ships two implementations: **DirectoryStore** (File System Access API
picker + drag-drop of an artifact directory; Chrome-class browsers) and
**StaticStore** (fetch from any static file server — `python3 -m http.server`
over `out/` is the zero-install path). Everything above the store is
provider-agnostic — this seam is deliberately shaped so a future
**grip/gryth provider** (live shared workspace state) can slot in without
touching a single view. That resolves the DecisionLog stack question as:
**plain React + Vite + TypeScript now** (pnpm, per house rules), grip
integration deferred behind the seam — dogfooding grip is a real want, but
coupling two in-flight projects would put wyred-ui on gryth's critical
path for no v1 payoff.

### 3.2 The census

The tree's shape is `stem.kind.json` (+ non-JSON siblings: `.cir`, `.net`,
`.png`, `.step`). The census view is the home screen: a matrix of stems ×
kinds, color-coded by schema validity, with the two L1-failing stems
rendering as the honest partial rows they are.

## 4. Surfaces (the views)

Ordered by the value they surface; each names its source artifacts.

1. **Netlist explorer** (`l2`) — components and nets as linked tables
   (graph rendering deferred): filter by refdes/kind/net/role; every
   component row carries its **provenance chip** — `authored` vs
   `for_demand: <declaration>` — because *"75% generated, every part
   confesses why"* is THE product story and this is where it becomes
   visible. Terminal table per component with roles and nets.
2. **Pin-map / firmware view** (`pinmap`, `alloc`) — the firmware-facing
   table: demand → pool/unit → pad → net, with **allocation-state badges**
   (`pinned` / `solver` / sticky) and **lock badges** (locked_by group @
   version). This is the view a firmware engineer keeps open; it must be
   printable/exportable as-is.
3. **BOM view** (`bom`) — line items with the `derived` explanation
   strings rendered as first-class content (the "load cap 0 for xtal:
   2*(CL-Cstray) = 20pF" strings), authored/generated split totals.
4. **Locks & lineage** (`records`, `baseline`, `connlock`, `lifecycle`) —
   lock groups with versions and covers; the **series tree** (A → fork →
   B) drawn from `forked_from` records; connector ICD tables from
   `connector_pinout` rows; baseline comparison status with the
   engine-reported/audit-checked labeling from §2.
5. **ECO diff** (`pinmapdiff`) — the minimal-disturbance story rendered:
   changed/added/removed allocations side-by-side, sticky survivals
   highlighted, `minimal_disturbance_violations` as a verdict panel.
   Given two arbitrary stems (draft vs revised), the UI shows the emitted
   diff artifact only — it does not compute diffs (§2).
6. **Intent view** (`l1`) — roles, rails/grounds/bonds, buses, pools,
   declared invariants, escalations (rich objects), the placement and
   test declarations when present. The "what was asked" side of every
   other view's "what was done".
7. **Verification dashboard** (`kexport` reports, `spiceop`, audit/gate
   outputs when saved as files, schema validity from §2) — verdict codes
   grouped by tier with the code vocabulary rendered as human text; the
   battery count; agreement-probe floors. v1 renders what exists as
   files; a saved-gate-log format is a future contract proposal, not a
   UI invention.
8. **Test bench view** (`testplan`, `measurements`, bench cards) — checks
   with expected ranges, probe points cross-linked to the pin-map view,
   measurement results colored by comparator verdict codes.
9. **Physical gallery** (renders, STEP/fab manifests) — image grid of
   `pcb render` outputs + fab manifest metadata (version pins, hashes).
   Display only.

**Cross-linking rule (the interaction model):** every refdes, net name,
demand id, role id, lock group, and code is a link; clicking pivots to
that entity's home view with the entity focused. The entity index is built
from loaded artifacts only (no derivation — it's an index, not an inference).

## 5. v1 scope cut

Views 1–4 + the census + schema banners are v1. Views 5–6 are v1.5
(artifacts already exist; pure rendering). 7–9 land as their file formats
stabilize. Graph/visual netlist rendering, gate-log contract, grip
provider, and any write path are explicitly deferred.

## 6. Testing story (the UI's own oracle)

- **Goldens are the fixture corpus, free**: every view gets snapshot tests
  over the committed goldens (the same trees, forever byte-stable) — the
  ga019 goldens are to wyred-ui what they were to the engine port.
- Schema-validation wiring is tested with the contract's committed mutant
  fixtures (they must banner, not render).
- The UI is **advisory, never a gate** — no run_gate wiring, ever; its
  correctness bar is "faithfully renders bytes", enforced by snapshots.

## 7. Stack summary

Vite + React + TypeScript, pnpm; zero runtime dependencies beyond React
itself where practical (the schemas validator gets a small TS port or runs
the JSON files through a minimal checker — decide in the plan; NO network,
NO backend, fully static deployment). Dark/light per `prefers-color-scheme`.
Runs from `pnpm dev` against any artifact directory; ships as static files.
