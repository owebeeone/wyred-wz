# wyred-wz — the wyred workspace

**wyred** is a declarative electronics design system — the productization of
elecscad's ga019 research result (measured (Δ=5.0, C=5); see
`dev-docs/WyredForwardPlan.md`). *You don't wire the board; the board is wyred.*

GWZ workspace: see `AGENTS_GWZ.md` to materialize; docs on gwz itself at
https://owebeeone.github.io/gwz-cli/ .

## Topology: a star, depth 1

Every member depends on **wyred-contract** (schemas + golden artifacts) and on
nothing else, unless its CLAUDE.md says otherwise. Members communicate through
committed contract artifacts, never through each other's code. Each member's
`CLAUDE.md` states its fence — agents working in a member obey that file.

| member | purpose | depends on |
|---|---|---|
| `wyred-contract` | versioned emit contract: schemas + goldens (the real API) | — |
| `wyred` | engine: authoring surface + elaborator + emitters | contract |
| `wyred-harness` | public read-only oracle stack | contract |
| `wyred-kicad` | KiCad adapters (extract, export, round-trip) | contract |
| `wyred-parts` | parts framework + stdlib + packaging | contract, wyred |
| `wyred-audit` | one-command consumer audit tool | contract, harness |
| `wyred-ui` | artifact viewer / review surface (mock-first) | contract |
| `wyred-agent` | MCP/skills agentic surface | contract, wyred, audit |
| `wyred-bench` | bench-card generator + measurement comparator (the oracle extended into physics) | contract |
| `wyred-examples` | reference designs + real-board ground truths + integration tests | all (the only one) |
| `wyred-docs` | product documentation (tutorials, workflow, concepts); aggregates every member's `docs/` into one site at build time | contract |

**Invariant:** `wyred` and `wyred-harness` never import each other — the
engine is independent of its checkers (the ga019 trust property, enforced by
repo boundary).

## Planned, not created

`wyred-solve` (real solver rung: AF matrices, CSP) · `wyred-fab`
(assembly/fab-house integration) · `wyred-physics` (GHz/openEMS loop).

## Conventions

- Plans/designs/project docs live in `dev-docs/` here at the root; decisions in
  `dev-docs/DecisionLog.md`.
- Contract changes are single-writer: propose in `dev-docs/`, land in
  `wyred-contract` with a version bump.
- `scratch/` (root and members) is git-ignored: local files to share with an AI
  without committing.
- Python throughout except `wyred-ui` (TypeScript/React, pnpm).
