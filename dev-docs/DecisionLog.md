# wyred DecisionLog

Append-only. One entry per decision; newest first. Format:
`## YYYY-MM-DD — title` / context / decision / consequences.

## 2026-07-13 — User documentation: per-member docs/ + a wyred-docs member (both)

The placement rule: **a doc lives where its change-cause lives.**
Member-surface reference (CLI flags, code vocabularies, APIs) → that
member's `docs/`, versioned and tested with the code. Product docs
(getting started, authoring tutorials, workflow guide, concepts,
glossary) → a new `wyred-docs` member (name confirmed at creation;
`wyred-doc` acceptable) that AGGREGATES member docs/ at build time
(collector script + mkdocs, the gwz-cli site precedent) — one published
site, zero duplicated content. Docs are never normative (contract/schemas
stay truth); docs join the oracle culture — every runnable example is
executed by a doc-test runner or explicitly marked not-runnable, wired
into the member's tests. dev-docs/ stays contributor/decision material.
Plan: WyredPlanDocs.md. Publishing is a Gianni-gated account action.

## 2026-07-13 — Buildout-critic findings F1/F2/F3 resolved

- **F1 (HIGH, testplan signal seam):** the signal-check `expect` shape is
  pinned FLAT (`{freq, freq_low, freq_high}` / `{duty, duty_low,
  duty_high}`, trio present iff declared) in ProposalTestplanContract §5 —
  the shape the engine emits. wyred-bench's comparator + card generator
  align to it; the divergent hand-authored fixture is retired in favor of
  the engine-emitted canonical (byte-identical across repos), with
  synthetic shapes (duty band etc.) kept under distinct filenames; the
  bench↔emitted-artifact seam is gate-wired so this class cannot recur
  silently.
- **F2 (MED, placement join modes):** the pinmap-`l1_role` join is now
  recorded as SPECIFIED-NOT-BUILT in WyredPlacementSemantics §1 itself
  (deferred until a laid-out wyred-emitted board exists); the
  correspondence-table join is the only v0 implementation, fail-closed.
- **F3 (LOW):** WyredPlanIndex status refreshed to staged-complete-
  awaiting-ratification.

## 2026-07-13 — wyred-ui Phase 0 + Phase 1 BUILT and verified (working tree)

Built the design+plan below through the Phase-1 milestone; all working-tree,
no commits. As-built:
- **gwz members added** to wyred-wz: gryth-ui (`bffbdd0`), grip-core
  (`97ff6c2`), grip-react (`c13b8a7`), glade-decl-ts (`4467e31`, grip-core's
  compile-wall dep) — cloned, origins→GitHub, `gwz capture`d.
- **wyred-ui workspace**: `@wyredjs/artifacts` (pure: kinds, ArtifactStore +
  StaticStore/DirectoryStore, design-set union, the validate.py port),
  `@wyredjs/plugin-wyred` (grips/taps + WyredStoreTap + census/raw views +
  set picker; plugin-api/grip-react/react as PEER deps), `app` (standalone
  host). no-react-state scanner + eslint bans adopted. 198 tests green.
- **gryth-ui test mount**: `src/plugins/wyred/` shim + `link:` dep (NOT
  `file:` — file: hardlink-copies and breaks the wyred-contract relative
  schema imports; link: symlinks so vite realpath-resolves them) + vite
  `server.fs.allow` for the wyred-wz sibling. gryth-ui 74 tests + full
  `tsc -b && vite build` green with the mount.
- **Verified in-browser** (both the gryth-ui mount and the standalone app):
  census over the real ga019 goldens renders 19 stems with the two L1-only
  stems as honest partial rows, the SPICE/pinmapdiff/baseline/connlock
  columns correct, all schema-valid in-browser, live watch loop; the 12
  contract mutants each render INVALID at exactly the oracle's
  (pointer, keyword); clicking an invalid cell opens the raw view with the
  SCHEMA_INVALID banner + exact emitted bytes.
- **Validator parity**: in-browser verdicts match validate.py file-for-file
  over 114 goldens + 12 mutants (mutant tests now assert the oracle's exact
  pointer+keyword). Documented ecosystem-correct divergences: JS RegExp vs
  Python re on line-terminator edge cases (schema $comment endorses the
  strict JS reading), and integer literals beyond double range at the
  caller's JSON.parse — neither exercised by any contract artifact.
- **Adversarial review** (3-lens: fence / validator-parity / house-rules) ran
  post-build; its 2 blockers (mount linking; regex parity) + 2 majors
  (picker stale-closure writes; integer overflow) + minors (NUL KEY_SEP
  making storeTap.ts binary; reload cache-bust; setDirectoryHandle epoch;
  raw-view spurious fetch; dead export) were all fixed or documented before
  this entry.
- **Entity-URI proposal** written: `WyredProposalEntityUri.md` (Phase 0.3) —
  `wyred://<set>/<stem>[/<kind>]#type:id`, awaiting ratification (journeys
  §9 Q4). Fence addendum added to wyred-ui/CLAUDE.md.

Next: Phase 2 (netlist / pinmap / bom + entity index & cross-links).

## 2026-07-13 — GrythWyredUiDesignPlan ratified ("go"); wyred-ui build starts

Gianni's "go" ratified `GrythWyredUiDesignPlan.md` (the how for the stack
reversal below) and authorized the build. Architecture: wyred-ui = three
packages — `@wyredjs/artifacts` (pure library, grip-free) /
`@wyredjs/plugin-wyred` (the gryth plugin; plugin-api/grip-react/react as
PEER deps) / `app` (standalone host = `@grythjs/desktop` + wyred plugins
only) — consuming gryth-ui/grip-core/grip-react as commit-pinned gwz
members of wyred-wz (the glade-decl-ts precedent; gryth-ui pinned at
`bffbdd0`, GLP-0006 P1.S4). Dev order: gryth-ui test mount FIRST (one shim
dir + one `file:` dep in gryth-wz/gryth-ui, reversible), standalone host at
Phase 1.5. Design sets (multi-tree fleets) are the workspace unit; entity
URIs get their own proposal (Phase 0.3) before view work; annotations are
local-first Phase 4; phases 0–4 zero glade, Phase 5 (shared design doc)
gates on gryth's seam. Fence re-scoped: emitted artifacts immutable; the UI
may AUTHOR provenance-stamped records (annotations now, J4 measurements
later); never runs the toolchain, never derives. gryth-ui coding rules
adopted wholesale (no React state — grips/taps only; scanner + eslint
bans). WyredPlanUi.md superseded. Per house convention, build output stays
UNCOMMITTED working tree pending commit-time review.

## 2026-07-13 — wyred-ui stack REVERSED by Gianni: gryth-ui, agents as collaborators

Supersedes the same-day plain-React entry below. Gianni's direction: the
UI is built on **gryth-ui** (plugin-based, glade-backed, agents connect
via session capabilities for human/AI collaboration). Requirements source:
`WyredUiUserJourneys.md` (J1–J6 + entity addressing). The ArtifactStore
seam survives as the glade doc-scope provider; the trust posture carries
unchanged. WyredPlanUi.md is superseded pending redesign (do not build
from it); WyredUiDesign.md §7 stack section superseded, rest stands.
Production-strategy colouring (J6): design decisions bend toward
AI-driven procurement — sourcing slots + release-bundle-as-negotiation-
artifact + uniform capability-scoped agents are the proposed lynch-pins,
pending Gianni's review (journeys doc §9).

## 2026-07-13 — wyred-ui: plain React + Vite behind a grip-ready store seam

Resolves the open bootstrap question (plain-React vs grip/glial client).
Decision (provisional, same commit-time review deal): **plain React + Vite
+ TypeScript now**; all data access behind the `ArtifactStore` interface
(WyredUiDesign §3.1) so a grip/gryth live provider can slot in later
without touching views. Rationale: dogfooding grip is a real want, but
coupling two in-flight projects puts wyred-ui on gryth's critical path for
zero v1 payoff; the seam preserves the option. Trust posture:
render-what-was-emitted — the viewer never derives, schema-validates
everything in-browser, and labels engine-self-reported verdict fields as
such. Design: WyredUiDesign.md; plan: WyredPlanUi.md.

## 2026-07-13 — Workflow-design proposals provisionally accepted ("go"); account batch executed

Gianni's "go" (2026-07-13) authorized: (a) the account-actions batch
(publish wyred-t9y; create wyred-klibs; fork KiCad/kicad-symbols,
kicad-footprints, kicad-packages3D as klibs-*), and (b) proceeding with the
opus-4.8 build fleet against the five staged proposals (schemas package
F1–F12; ContractProposalKicadExport + EXPORT.md; WyredPlacementSemantics;
ProposalTestplanContract RATIFY-1–7; WyredSpiceContract incl. the
intent_10_spice_divider golden-census event). **Provisional acceptance:**
builds proceed on these stances, but every contract-shaped change remains
UNCOMMITTED working tree; final ratification is the commit-time review.
Any stance Gianni reverses at review reverts the dependent code before
commit — the proposals, not the code, are the record of what was decided.

## 2026-07-12 — Ground-truth corpus moved to wyred-t9y (hybrid fork/vendor/download)

The third-party board trees (exogenous denotations) moved from the elecscad
symlink to `~/limbo/wyred-t9y` (plain git, NOT gwz-managed). Hybrid model:
six upstreams as submodules of owebeeone `t9y-*` GitHub forks pinned at the
provenance-recorded SHAs (availability + verifiable exogeneity; `update=none`
so naive recursive clones don't pull ~500 MB unpruned — `materialize.py`
drives init with sparse-checkout); MNT Reform vendored (GitLab upstream,
CERN-OHL-S permits redistribution); rpi-cm4io download-on-materialize with
pinned sha256 (LICENSE-UNKNOWN — never redistributed). `view/` reproduces the
pruned reference trees byte-identically (one vendored override:
`watchy/Watchy.kicad_prl`, a KiCad-generated settings file absent upstream).
`wyred-wz/third-party` now symlinks `../wyred-t9y/view`; `run_gate.py` gained
content-based probe-readiness checks and prints the ground-truth revision
pins when probes run. Since t9y is outside gwz, that printed pin line is the
audit-trail record of which ground truth a gate-green run used.

## 2026-07-12 — Phase-3 residual closure

Context: the phase-2 adversarial critic left four residuals against the
gate/audit build. All four are now closed or recorded:

- **R1 (MED) closed — the ECO minimal-disturbance verdict is no longer
  engine-self-certified.** `wyred-audit/src/wyred_audit/incumbency.py` carries
  the audit's own independent ports of the engine's `paths.diff_pinmaps` and
  `emit._check_incumbency` (== ga019 runner.py:685-711) and re-derives EVERY
  field of every `<name>.pinmapdiff.json` from the on-disk artifacts alone
  (the incumbent's pinmap+alloc plus the artifact's own), comparing field by
  field — `minimal_disturbance_violations` included (`PINMAPDIFF_DISAGREE`,
  `MINIMAL_DISTURBANCE_NOT_OK`). A lobotomized engine-side incumbency check
  now turns the audit red (fixtures `pinmapdiff_blanked`,
  `sticky_alloc_moved`) — the same restoration shape as the closed F3 finding.
- **R2 (LOW) closed — from-primaries rebuild honesty restored, corpus-less.**
  `python3 -m wyred.rebuild --dir <tree> [--artifact NAME | --all]`
  (`wyred/src/wyred/rebuild.py`) is the direct engine-side port of
  runner.py:554-574: every secondary path (bom/pinmap/records) re-derived
  from the on-disk PRIMARY artifacts alone (l2 + alloc + l1, no corpus) and
  byte-compared. `wyred-audit` composes it as a subprocess stage
  (`REBUILD_FROM_PRIMARIES`, needs only `--engine-src`), so a consumer
  holding artifacts + an engine checkout — no corpus — can again prove the
  secondaries are pure functions of the on-disk primaries. Distinct from the
  audit's whole-corpus re-emit, which needs `--corpus-dir` and proves emit
  determinism against the source corpus.
- **R3 (LOW) closed — gate.py has a committed regression suite.**
  `wyred-harness/tests/run_gate_tests.py` (own entry point; deliberately NOT
  wired into the frozen 149-check selftests) exercises the gate end-to-end
  via subprocesses in temp dirs: the clean examples corpus gates green with
  the full 26/26 battery line; the committed `tests/fixtures/f4_corpus/`
  fixture (the F4 declared-vs-oracle disagreement — the engine emits fine,
  the gate must fail naming ADDR_COLLISION); the applicability-lobotomy
  corpus (board modules removed → the "probe(s) never found an applicable
  artifact" verdict must be the run's ONLY failure); and a dropped-BOM-line
  tamper failing the cross-path differential. These verdicts were previously
  proven only with ad-hoc doctored corpora.
- **R4 (INFO) recorded — connlock `groups` derivation asymmetry, accepted as
  fails-closed.** The audit derives a connlock record's `groups` as the l1's
  lock groups covering `connector_pinout` at version >= 1; the engine
  recorded the groups it FROZE at that emit (freeze ∩ covers — freeze
  membership is not on disk). An inherited locked connector group that was
  not re-frozen at an emit would make the audit's set larger and fire a
  spurious `CONNLOCK_DISAGREE`: a false alarm, never a missed tamper. No
  corpus artifact exercises that shape today; the fix is deferred until a
  real corpus case does. Recorded in wyred-audit/README.md ("Explicitly
  deferred").

Evidence at closure: a fresh emit of wyred-examples/corpus is byte-identical
to wyred-contract/goldens/ga019; `run_gate.py` end-to-end PASS (board
probes ran); full-form audit PASS (all seven checks); `run_gate_tests.py`
and `run_fixture_tests.py` PASS; the 149 harness selftests PASS.

## 2026-07-12 — Runner-split deviations ratified by the phase-2 gate/audit build

Context: the phase-1 adversarial review flagged three deviations from ga019's
`runner.py`. The phase-2 build (`wyred-harness/harness/gate.py`, `wyred-audit`,
`wyred-examples/run_gate.py`) closes them by construction; the deviations are
now deliberate and ratified:

- **(a) F3 — emit-time lock/baseline snapshot code stays engine-side.**
  Baseline/lifecycle/connlock bytes are emit-time artifact CONTENT, so the
  snapshot code lives in `wyred/src/wyred/emit.py`. The trust property is
  restored outside the engine: `wyred-audit` independently re-derives every
  snapshot from disk (harness library) and byte-compares on every audit run —
  any drift between the two implementations turns the audit red.
- **(b) F4 — L2 emission gates on the declaration alone.** The engine emits L2
  for declared-clean intents without consulting the live oracle; oracle
  disagreement is the harness gate's verdict, now enforced end-to-end by
  `gate.py`'s expected-codes comparison (the phase-1 detection hole is closed).
- **(c) F5 — cross-boundary composition is subprocess-only.** Gate and audit
  reach engine-side checks (`paths.crosscheck`, `check_connector_locks`) only
  via the `python3 -m wyred.crosscheck` CLI; imports across the engine/harness
  fence remain forbidden (RunnerSplit.md composition rule). `wyred-audit`
  additionally carries its own independent connlock port, kept honest by the
  audit going red on drift.

## 2026-07-12 — Bootstrap decisions (workspace creation)

- **Name: wyred.** PyPI/crates/npm/brew/wyred.dev available; github.com/wyred
  is an unrelated user account (use owebeeone/*); nearest trademark neighbor is
  Wyred 4 Sound (consumer hi-fi) — check before commercial use.
- **Substrate: Python.** The authoring surface (marker-harvest metaprogramming)
  IS the measured product; performance lives in delegated tools. Rust deferred
  per-component (first candidate: wyred-audit as a static binary); the L1 JSON
  boundary keeps a future Rust resolver insertable via two-engine differential.
- **Topology: star, depth 1, contract at the center.** Nine members (see root
  README). Repo boundaries are AI fences forcing API hygiene; goldens-first
  bootstrap unblocks all tracks before the engine port lands.
- **Contract governance: single-writer.** Proposals in dev-docs/; versioned
  bumps land only in wyred-contract. Contract v0 = elecscad EMIT_CONTRACT v3
  verbatim + ga019 gate-green goldens.
- **elecscad stays as the research archive.** rl/ scorecards + reviews are the
  durable audit trail; wyred is the product. The old top-level elecscad/
  package spike is retired by this move (forward plan Path F resolved).
- **Two tracks:** g19++ (full-featured buildout, this workspace) and g20+
  (surface-dialect iterations over the frozen contract, scored on a declared
  new fitness axis). Vocabulary/facility changes belong to the g19++ track.

Open (not yet decided): gen-4 ratification pass (13 stances, 8 open items —
prerequisite before contract v1); wyred-ui plain-React vs grip/glial client;
which board is the tracer-bullet full-loop target.
