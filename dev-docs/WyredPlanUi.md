# WyredPlanUi — the artifact viewer, phased

> **SUPERSEDED (2026-07-13, DecisionLog): do not build from this plan.**
> Gianni redirected the UI to **gryth-ui** (plugin-based, glade-backed,
> agent collaboration). Requirements now live in `WyredUiUserJourneys.md`;
> this plan awaits redesign as gryth-ui plugin increments (with a Phase 0:
> gryth integration spike + entity-URI proposal). The surface inventory
> and acceptance ideas below remain useful raw material.

*Phased plan for [WyredUiDesign.md](./WyredUiDesign.md). Phases are
shippable milestones; steps one goal each, ~<500 LOC advisory; all steps
**opus-4.8** (the design doc made every architectural decision; this is
mechanical React). Repo: `wyred-ui` exclusively — zero contention with any
other workstream (fence: contract schemas + goldens only, no engine/harness
/audit imports or calls, ever). pnpm, never npm.*

## Invariants (every step)

- **Render-what-was-emitted** (design §2): no derivation, no fix-ups;
  schema-invalid files banner + raw view. Every displayed value must be
  traceable to bytes in a loaded artifact.
- **Goldens are the fixture corpus**: dev + snapshot tests run against
  `wyred-contract/goldens/ga019/` (read-only — never copied into wyred-ui
  except as test snapshots of RENDERED output).
- **The ArtifactStore seam is sacred**: no view touches loading directly;
  DirectoryStore/StaticStore behind the design §3.1 interface, grip-ready.
- No backend, no network at runtime, static deployment; `pnpm build` output
  runnable from any static server. No new runtime deps beyond React without
  a recorded reason in the plan-execution report.
- Nothing here touches run_gate, goldens, or any other repo's files. No
  commits.

## Phase 1 — skeleton + census + netlist explorer (milestone: point it at goldens, explore a real board)

- **1.1 Scaffold + stores** [~300]: Vite+React+TS scaffold (pnpm), the
  `ArtifactStore` interface, StaticStore (fetch) + DirectoryStore (FS
  Access API + drag-drop), stem/kind census parser, routing shell,
  light/dark. Acceptance: `pnpm dev` + `python3 -m http.server` over
  goldens shows the census matrix (18 stems × kinds, two honest partial
  rows for the L1-failing stems).
- **1.2 Schema validation wiring** [~250]: TS port of the minimal
  validator subset (the contract's `tools/validate.py` semantics: same
  keyword whitelist, fail-closed on unknown keywords) loading
  `wyred-contract/schemas/`; census cells color by validity;
  `SCHEMA_INVALID` banner + raw-JSON fallback view. Acceptance: all
  goldens validate in-browser; the contract's committed mutant fixtures
  banner correctly (drag-drop one in).
- **1.3 Netlist explorer** [~450]: components/nets linked tables per
  design §4.1 with provenance chips (`authored` / `for_demand`), terminal
  detail, filters, and the global cross-link index (refdes/net/demand/
  role/lock-group → view pivots). Acceptance: snapshot tests over
  watchy_v1_reva + mppt_2420_hc render stably; clicking any net name in a
  component's terminals lands on that net focused.

## Phase 2 — pin-map + BOM (milestone: the firmware engineer's and the buyer's views)

- **2.1 Pin-map/firmware view** [~350]: design §4.2 — allocation table
  with state + lock badges, print stylesheet. Acceptance: watchy_v1_reva
  shows all 16 author-pinned rows badged `pinned`+locked; watchy_v1_draft
  shows the two solver-sticky INT rows badged `solver`.
- **2.2 BOM view** [~250]: design §4.3 — line items, derived strings
  first-class, authored/generated totals. Acceptance: snapshot over
  mppt_2420_hc (140 line items; derivation strings render).

## Phase 3 — locks, lineage, ECO, intent (milestone: the review surface)

- **3.1 Locks & lineage** [~400]: design §4.4 — lock groups, series tree
  from forked_from, connector ICD tables, engine-reported/audit-checked
  labeling. Acceptance: watchy reva→revb fork renders as a two-node tree
  with the fork reason; connlock rows visible.
- **3.2 ECO diff + intent view** [~400]: design §4.5–4.6. Acceptance:
  watchy_v1_draft_btn3's pinmapdiff renders changed-only-btn3 with sticky
  survivals highlighted; intent_05b's escalation objects render rich.

## Deferred (design §5)

Verification dashboard, test-bench view, physical gallery (their file
formats are landing in the buildout); graph rendering; grip provider;
gate-log contract proposal.

## Dependency note

Phase 1.2 depends on the schemas package (staged in wyred-contract — read
from the working tree). Everything else needs only goldens. Phases are
sequential; steps within a phase can pair-parallel if two agents use
separate view directories, but v1 is fine single-agent.
