# WyredPlanDocs — user documentation, phased

*Phased plan for the documentation decision (DecisionLog pending entry;
discussion 2026-07-13): per-member `docs/` for change-with-the-code
reference material + ONE new `wyred-docs` member for product documentation,
aggregated into a single published site at build time. All steps
**opus-4.8** — the information architecture and placement rule are decided
here; the writing is derivation from verified sources (the corpus, the
proposals, the CLIs themselves). Steps ~<500 LOC/prose-lines advisory.*

**Member name:** default `wyred-docs` (plural, matches the `docs/` dirs);
Gianni confirms or renames at the 1.1 creation step — it is one `gwz repo
create` argument either way.

## Invariants (every step)

- **Docs are NEVER normative.** EMIT_CONTRACT + schemas + the per-repo
  conventions docs (e.g. wyred-kicad/EXPORT.md) remain the sources of
  truth; user docs explain and LINK, never re-specify. A doc page that
  contradicts a contract file is a defect in the doc.
- **The placement rule:** a doc lives where its change-cause lives.
  Member-surface reference (CLI flags, code vocabularies, package APIs) →
  that member's `docs/`. Product-level material (tutorials, workflow,
  concepts) → `wyred-docs`. No content is ever duplicated across repos —
  the site build aggregates.
- **Docs join the oracle culture:** every fenced command block in user
  docs is either executed by the doc-test runner or explicitly marked
  `<!-- not-runnable: reason -->`. A failing example is a red run.
- No commits (working tree, house review flow). No new runtime
  dependencies in any code member; the docs toolchain (mkdocs) is confined
  to wyred-docs.
- Site publishing (Pages/wyred.dev) is a Gianni-gated account action —
  planned, never executed by an agent.

## Phase 1 — member, skeleton, aggregation, doc-tests (milestone: `mkdocs build` produces the unified site from live sources; doc-tests run)

- **1.1 Member + skeleton + collector** [wyred-docs · ~250 · opus]:
  `gwz repo create wyred-docs` (root README member-table update; CLAUDE.md
  fence: *"Product documentation only. Depends on wyred-contract for
  links/schemas; aggregates member `docs/` READ-ONLY at build time; never
  normative; never imports or runs member code except via the doc-test
  runner's subprocesses."*). mkdocs skeleton (material theme optional —
  match the gwz-cli site's conventions), nav structure per this plan's
  Phase-2 outline, and `tools/collect.py`: a dependency-free script that
  copies each member's `docs/` into `site-src/members/<member>/` at build
  time (workspace-relative discovery via `gwz ls --local` or a static
  member list; provenance header injected into every collected page:
  "source: <member>/docs/<file> — edit THERE").
  *Acceptance:* `python3 tools/collect.py && mkdocs build` exits 0 on the
  skeleton; collected pages carry provenance headers; `git status` in
  member repos untouched.
- **1.2 Doc-test runner** [wyred-docs · ~300 · opus]: `tests/run_doc_tests.py`
  — walks every `.md` under the site sources (own + collected), extracts
  fenced blocks tagged runnable (convention: ` ```console ` blocks whose
  first line starts `$ ` are runnable; `<!-- not-runnable: reason -->`
  immediately above a fence exempts it), executes them sequentially in a
  scratch sandbox against the workspace (PYTHONPATH per the block's
  declared member context, cwd per a `<!-- cwd: ... -->` annotation),
  asserting exit 0 and (when an `# expect:` comment is present) output
  substrings. Fail-closed: a runnable block with no way to execute is an
  error, not a skip.
  *Acceptance:* a committed self-test doc with one passing, one
  deliberately-failing (asserted to fail), and one exempted block behaves
  exactly as specified; runner exits 0 on the skeleton site.
- **1.3 Per-member reference docs** [each member's `docs/` · ~400 total ·
  opus, parallelizable per repo]: CLI reference pages written where they
  belong: `wyred/docs/` (emit, crosscheck, rebuild, --list/--exemplify),
  `wyred-harness/docs/` (gate, selftests — pointing at, not duplicating,
  the battery semantics), `wyred-audit/docs/` (the audit CLI, finding
  codes, consumer forms), `wyred-kicad/docs/` (export/roundtrip/fab/threed
  + a pointer to EXPORT.md, which stays where it is), `wyred-bench/docs/`
  (card, compare, verdict codes, measurement-record how-to),
  `wyred-contract/docs/` (how to read the schemas; artifact-kind index).
  Every flag documented is greppable in the code; every example runnable.
  *Acceptance:* collector picks all of them up; doc-tests green over every
  runnable example; a mechanical check that every `python3 -m <member>.*`
  entry point named in any README has a reference page (the coverage grep,
  first cut).

## Phase 2 — the product docs (milestone: a new user goes from zero to a gate-green board using only the site)

Sources for all steps: the corpus encodings (the best tutorials in
existence), the ratified proposals, run_gate.py's stage output, and the
DecisionLog for the "why". Voice: engineer-to-engineer, no marketing.

- **2.1 Getting started** [wyred-docs · ~350 · opus]: install/setup
  (workspace clone, materialize, gwz basics by link), then the
  first-board quickstart: author a ~10-line intent (modelled on
  intent_01), emit, read each artifact, run the gate, read a verdict.
  Every command doc-tested end-to-end.
- **2.2 Authoring tutorial I — the declarative surface** [~450 · opus]:
  modules/params/`late()`, demand/provide + interface families,
  rails/grounds/bonds/buses, companions + provenance. Sourced from
  corpus intents 01–06; every code sample is a real corpus excerpt or a
  doc-tested variant.
- **2.3 Authoring tutorial II — allocation & lifecycle** [~450 · opus]:
  pools + symbolic allocation, refinements (pin/bind), locks/freeze,
  baselines, ECO (unlocked) vs fork (locked), the pinmapdiff story.
  Sourced from 05a/05b + the Watchy lifecycle pair.
- **2.4 The workflow guide** [~400 · opus]: intent → emit → gate → audit →
  KiCad export → layout → DRC/gerbers → placement verification → testplan
  → bench card → measurements → comparator, with **what each stage
  proves** (the trust story told for users, not critics) and the loud-skip
  semantics. Includes the fleet/harness-board picture from
  WyredUiUserJourneys J3 at overview level.
- **2.5 Concepts + glossary + artifact index** [~350 · opus]: the
  two-layer model, say-only-the-delta, provenance, asserted-vs-realized,
  the code-vocabulary index (linking to member reference pages), the
  artifact-kind table (linking to schemas).

## Phase 3 — coverage + publish readiness (milestone: the site is honest, complete for what exists, and one command from publishable)

- **3.1 Coverage checks** [wyred-docs · ~150 · opus]: mechanical greps
  wired into `run_doc_tests.py`: every member CLI entry point has a
  reference page; every public code vocabulary (KEXPORT/PLACEMENT/XCIR/
  bench verdicts/audit findings) is linked from the index; every artifact
  kind in the schemas dir has an index row. Missing = red.
- **3.2 Publish pipeline (prepared, not executed)** [~150 · opus]: the
  build-and-deploy script (mkdocs → static output; gh-pages or wyred.dev
  target parameterized), documented in wyred-docs/README with the explicit
  "Gianni runs this" gate. No agent ever publishes.

## Deferred

UI documentation (waits on the gryth-ui track's shape); versioned-docs
(one site per release — needs a release to exist); localization;
API-from-docstrings generation (the reference pages are hand-written and
doc-tested for now — honest and small beats generated and unread).

## Dependency notes

1.1 → 1.2 → (1.3 ∥ 2.1) → 2.2 → 2.3 → 2.4 → 2.5 → 3.x. Step 1.3's six
per-member page-sets are independent of each other (six small parallel
agents or one). Nothing here touches engine/gate code; the only
non-wyred-docs writes are each member's own new `docs/` directory.