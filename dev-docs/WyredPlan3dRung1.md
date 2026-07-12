# Wyred 3D — Rung 1 Plan (borrowed libraries + kicad-cli delegation)

*Phased plan for [WyredWorkflowDesign.md](./WyredWorkflowDesign.md) §3.5 **rung 1
only**: pinned forks of the official KiCad libraries with t9y-style
materialize-on-demand, plus `kicad-cli` STEP-export and render wrappers in
wyred-kicad. **Rung 2 — the pythonopenscad/anchorscad bridge (`wyred-3d`) — is
explicitly OUT of this plan**; it awaits its own design spec,
`Wyred3dDesign.md`, and nothing here may pre-commit its shape beyond keeping
the Phase 1 resolver consumable by it.*

Plan conventions: phases are shippable milestones; each step is one goal with
an aspirational **< 500 LOC** budget (advisory, not a limit); every step lists
repo / ~LOC / model and a **mechanical acceptance**. All steps are
**opus-4.8** — this is delegation plus the established wyred-t9y pattern; no
step surfaced a design decision needing fable (the two judgement calls in this
workstream — fork home and packages3D transfer strategy — are decided in this
document). Steps prepare working-tree changes only; **commits, pushes, tags
happen only on Gianni's explicit instruction** (house git rules).

## Invariants (non-negotiable)

- **No vendoring of KiCad libraries into any wyred repo.** Access is pinned
  GitHub forks + materialize-on-demand, exactly the wyred-t9y model
  (`update = none` submodules, pinned SHAs, `materialize.py`, gitignored
  `view/`). License mechanics recorded below.
- **Goldens and the gate are unaffected.** This workstream emits nothing into
  artifact trees; STEP/render outputs go to gitignored scratch/out
  directories. 3D stays **advisory, not oracle** (design §5) — the only
  mechanical checks introduced are repo-local tests.
- **GitHub account actions are gated.** Every step that creates a repo or a
  fork under `github.com/owebeeone` is marked **[REQUIRES GIANNI'S GO]** and
  must not be executed by an agent without it.
- **Forks are availability pins, not lines of development.** No wyred-side
  commits are ever added to them (t9y precedent).

## Observed local capabilities (probed 2026-07-12)

`/opt/homebrew/bin/kicad-cli` → symlink to
`/Applications/KiCad/KiCad.app/Contents/MacOS/kicad-cli`, **version 10.0.4**.
The app bundle ships the full official libraries under
`Contents/SharedSupport/` (symbols, footprints, **3.1 GB** `3dmodels`) — so
non-hermetic renders work today with zero library work; the Phase 1 forks
exist for pinned provenance and machines without a 3.1 GB KiCad install.

**`pcb export step` — flags observed (10.0.4):** `--include-tracks`,
`--include-pads`, `--include-zones`, `--include-inner-copper`,
`--include-silkscreen`, `--include-soldermask`, `--subst-models`,
`--board-only`, `--no-components`, `--component-filter`, `--net-filter`,
`--no-dnp`, `--no-unspecified`, `--fuse-shapes`, `--fill-all-vias`,
`--cut-vias-in-body`, `--min-distance`, `--drill-origin` / `--grid-origin` /
`--user-origin`, `--variant`, `-D/--define-var`, `--no-optimize-step`, `-f`.

**`pcb render` — flags observed (10.0.4):** `--side
{top,bottom,left,right,front,back}`, `--rotate 'X,Y,Z'` (isometric =
`'-45,0,45'`), `--zoom`, `--pan`, `--pivot`, `--perspective`, `--quality
{basic,high,user,job_settings}`, `--preset`, `--background
{default,transparent,opaque}`, `-w/-h` (default 1600×900), `--floor`, light
intensity/elevation controls, `--variant`, `-D`.

**Behavioral findings from live probes** (watchy = KiCad 7 format `20221018`;
mppt-1210-hus / mppt-2420-hc = KiCad 5 format `20171130`, all from
`wyred-t9y/view/`):

1. **Renders were byte-identical across back-to-back runs** (sha256 equal) for
   both watchy and mppt-1210-hus at `--quality basic`, same machine/session.
   This is a same-machine observation only — step 2.2 measures and records it
   properly; **no cross-machine byte-stability is assumed and no render hash
   ever becomes a golden.**
2. **STEP export exits 0 despite missing 3D models** — watchy produced a
   9.6 MB STEP in ~2 s while printing 68 `File not found` warnings to stderr.
   Wrappers MUST capture stderr and confess the missing-model list, or absence
   is silent (violates the no-silent-absence principle).
3. **`--define-var` does NOT resolve 3D path variables** (`${KICAD6_3DMODEL_DIR}`
   warnings unchanged). **Process environment variables DO**: setting
   `KICAD6_3DMODEL_DIR` to a directory containing the referenced files
   resolved them (proven with a scratch tree). Wrappers use env vars
   (`KISYS3DMOD`, `KICAD6..10_3DMODEL_DIR`) as the hermetic-resolution
   channel.
4. **The `.wrl`/`.step` twin problem is real.** KiCad 10's bundled 3dmodels
   ship `.step` only, while KiCad 5–7-era boards reference `.wrl`
   (mppt-2420-hc: `${KISYS3DMOD}/....wrl`; watchy: `${KICAD6_3DMODEL_DIR}/....wrl`;
   mppt-1210-hus mostly `.step`). `--subst-models` substituted the `.step`
   twin when the referenced `.wrl` path also existed, but not when it was
   absent → the sparse materializer fetches **both twins** per referenced
   model (upstream `kicad-packages3D` carries both).
5. **Some corpus model refs are permanently unresolvable** and must be
   confessed, not "fixed": watchy references 10 project-local
   `${KIPRJMOD}/3D/*.step` models pruned from the t9y corpus;
   mppt-1210-hus references a user-installed `LibreSolar.3dshapes` that is
   not part of the official `kicad-packages3D`.

## Fork home: decision

**Recommendation: a new sibling repo `wyred-klibs`** at
`/Users/owebeeone/limbo/wyred-klibs` (GitHub: `owebeeone/wyred-klibs`),
alongside `wyred-t9y` and outside `wyred-wz` — **not** an extension of
wyred-t9y.

| | extend wyred-t9y | new sibling `wyred-klibs` |
|---|---|---|
| Identity | t9y is a **frozen exogenous denotation** — byte-identity to the 2026-07-02 corpus, "nothing in wyred generates or edits them" | forward-path **tooling input** that will re-pin on KiCad upgrades — a different invariant class |
| Manifest fit | per-file sha256 enumeration (fine for 8 pruned designs) is absurd for ~15k footprints + on-demand models | verify via git's own content addressing (pinned tree/blob hashes) |
| Consumers | board-agreement probes / pcb_extract, via `wyred-wz/third-party` symlink | wyred-kicad now, wyred-3d later |
| Licensing | per-design patchwork incl. LICENSE-UNKNOWN special cases | uniform CC-BY-SA-4.0 + KiCad libraries exception |
| Shared | — | reuses materialize.py **patterns**: `update = none` submodules, pinned SHAs, `git archive` extraction, `--check`, idempotence |

Mixing the two would dilute t9y's frozen-corpus contract and bloat its
manifest for no gain; what is actually shared is the pattern, not the data.
Not inside `wyred-wz` either: workspace members are gated code repos, and
klibs is a data-mirror utility like t9y (a `wyred-wz/kicad-libs ->
../wyred-klibs/view` symlink can be added later if convenient, mirroring
`third-party -> ../wyred-t9y/view`).

---

## Phase 1 — `wyred-klibs`: pinned library forks + sparse materialize

**Milestone:** a fresh `git clone` (no submodule recursion) + one command
yields pinned, verified KiCad symbol/footprint trees and on-demand 3D models,
with license obligations satisfied. Nothing multi-GB is ever transferred.

Version pin policy (all three libraries): pin at the upstream release tag
matching the local KiCad train (**10.0.x**, matching kicad-cli 10.0.4), but
**record the commit SHA in the manifest, never the tag** (tags are mutable;
SHAs are the pin). Re-pins are deliberate manifest edits, single change per
KiCad upgrade.

### Step 1.1 — repo scaffold + wholesale forks of kicad-symbols and kicad-footprints

- **Repo:** `wyred-klibs` (new) · **~LOC:** ~150 (manifest tooling; the rest
  is gh/git commands) · **Model:** opus-4.8
- **[REQUIRES GIANNI'S GO]** — three GitHub account actions, listed
  separately so they can be approved as a batch:
  1. create `owebeeone/wyred-klibs`;
  2. fork `github.com/KiCad/kicad-symbols` → `owebeeone/klibs-kicad-symbols`;
  3. fork `github.com/KiCad/kicad-footprints` → `owebeeone/klibs-kicad-footprints`.
  (The `github.com/KiCad/*` repos are the official GitLab mirrors; forking is
  server-side, no local bandwidth.)
- Scaffold: `README.md` (usage + license summary, t9y README as the
  template), `.gitmodules` with `upstream/klibs-kicad-symbols` and
  `upstream/klibs-kicad-footprints` **carrying `update = none`**, gitignored
  `view/`, `manifest.json` schema v1: per-library `{kind, upstream, fork_url,
  submodule, pinned_sha, pinned_tree, license, materialize}` — no per-file
  hash lists; integrity rides on git content addressing.
- `materialize.py` (new, borrowing t9y's helpers): for the two wholesale
  trees — shallow submodule init with one-shot `update=checkout` override,
  fetch pinned SHA if absent, extract via `git archive <sha>` into
  `view/symbols/` and `view/footprints/` (worktree state never affects
  produced bytes), copy the upstream `LICENSE.md` to each view root.
  `--check` verifies `git rev-parse <sha>^{tree}` matches `pinned_tree` and
  spot-verifies view files against `git cat-file` blobs.
- **Acceptance (mechanical):** on a fresh clone: `python3 materialize.py`
  succeeds; `view/footprints/Resistor_SMD.pretty/R_0402_1005Metric.kicad_mod`
  and `view/symbols/Device.kicad_sym` exist; `view/*/LICENSE.md` present;
  `python3 materialize.py --check` exits 0; a second `materialize.py` run
  performs no network access and reports up-to-date; `git status` in
  wyred-klibs shows no view/ files tracked.

### Step 1.2 — kicad-packages3D pinned fork + sparse model materializer

- **Repo:** `wyred-klibs` · **~LOC:** ~250 · **Model:** opus-4.8
- **[REQUIRES GIANNI'S GO]**: fork `github.com/KiCad/kicad-packages3D` →
  `owebeeone/klibs-kicad-packages3D`.
- **Sparse is mandatory and shallow is not sparse:** the repo is multi-GB,
  and a `git fetch --depth 1 origin <sha>` still transfers every blob in
  that snapshot. The submodule must be a **partial clone**
  (`--filter=blob:none`, promisor remote); `git archive <sha> -- <paths>` (or
  `git cat-file` per path) then lazily fetches only the requested blobs.
  This is the one place the t9y recipe is extended rather than copied —
  t9y's sparse-checkout trick prunes the *worktree*, the blob filter prunes
  the *transfer*.
- Input: a list of model references (from step 1.3's resolver, or raw
  `(model ...)` paths). For each referenced model, fetch **both** the `.step`
  and `.wrl` twins when present upstream (probe finding 4), into
  `view/3dmodels/<Lib>.3dshapes/<Model>.{step,wrl}` + `view/3dmodels/LICENSE.md`.
- Idempotent and verifying: already-present files are checked against the
  git blob hash at the pinned SHA and skipped; `--check` re-verifies the
  whole materialized set; unresolvable requests are reported as a
  confessed-missing list (exit code distinguishes "missing upstream" from
  "fetch/verify failure").
- **Acceptance (mechanical):** given the union of model refs from the t9y
  watchy + both MPPT boards, materializer completes with
  `view/3dmodels` under ~50 MB (vs multi-GB full tree — assert with a size
  check); `R_0402_1005Metric.step` **and** `.wrl` both present; the
  `${KIPRJMOD}` and `LibreSolar.3dshapes` refs come back in the
  confessed-missing list, not as errors; second run is no-network no-op;
  `--check` exits 0.

### Step 1.3 — resolver: footprint id → footprint file → 3D model paths

- **Repo:** `wyred-klibs` (`klibs/resolver.py`, importable library + small
  CLI) · **~LOC:** ~200 incl. tests · **Model:** opus-4.8
- Lives beside the data it indexes; **both wyred-kicad (rung 1) and the
  future wyred-3d (rung 2) consume it** — this keeps wyred-kicad's boundary
  rule intact (no adapter-to-adapter import) and gives rung 2 a ready-made
  seam without prejudging its design.
- Two modes sharing one s-expr scan (no full parser — targeted extraction in
  the pcb_extract spirit):
  1. **footprint-id mode** (forward path): `"Resistor_SMD:R_0402_1005Metric"`
     → `view/footprints/Resistor_SMD.pretty/R_0402_1005Metric.kicad_mod` →
     its `(model ...)` entries;
  2. **board mode**: given a `.kicad_pcb`, enumerate all embedded `(model ...)`
     refs (boards carry footprint instances inline; no library needed).
- Path normalization: expand `${KISYS3DMOD}`, `${KICAD5..10_3DMODEL_DIR}` →
  `view/3dmodels`; classify every ref as `resolved` (file exists in view),
  `fetchable` (exists at the pinned packages3D SHA — feeds step 1.2), or
  `confessed-missing` (`${KIPRJMOD}`-relative, non-official libs). No
  silent drops.
- **Acceptance (mechanical):** unit tests for both modes and all three
  classifications; **differential test in the house style**: board-mode
  output for `wyred-t9y/view/watchy/Watchy.kicad_pcb` — its
  missing-model set must equal the `File not found` set kicad-cli emits on
  STEP export of the same board (two independent engines, one disagreement
  code); `python3 -m klibs.resolver --board <pcb> --json` round-trips
  through `json.loads`.

---

## Phase 2 — `wyred-kicad`: board 3D + renders via delegation

**Milestone:** one call produces a STEP model and a set of raytraced renders
for any corpus or wyred-produced board, hermetically resolved against the
pinned libraries, with every missing model confessed in a sidecar — proven on
the t9y watchy and MPPT boards.

Note: wyred-kicad currently contains only `CLAUDE.md`/`README.md`; these
wrappers are its first code. Boundary rules apply (depends on wyred-contract
only among workspace members; kicad-cli reached by subprocess; klibs' resolver
consumed as an external library, not a workspace import). The §3.1 netlist
exporter is a **separate workstream** — do not couple to it.

Coupling note for parallel agents: step 2.1 can be built and smoke-tested
against the local KiCad bundle's libraries (probe shows renders work today)
before Phase 1 lands; only its *hermetic mode* (env vars → klibs view) waits
on 1.2/1.3. Phase 1 and Phase 2 are parallelizable to that extent.

### Step 2.1 — STEP-export and render wrappers + capability probe

- **Repo:** `wyred-kicad` (`wyred_kicad/threed.py`) · **~LOC:** ~200 ·
  **Model:** opus-4.8
- `probe_capabilities()`: run `kicad-cli version` and parse
  `pcb export step --help` / `pcb render --help` into a feature set; hard-fail
  below the version floor (working floor: the locally observed **10.0.4**;
  design §6 keeps ≥ 8 as the eventual documented minimum — flags above are
  the 10.0.4 ground truth, re-probe rather than assume on other installs).
- `export_step(pcb, out, *, include_tracks=True, include_pads=True,
  include_zones=False, subst_models=True, model_dirs=...)`: gate
  `--include-*` flags on the probe result; set `KISYS3DMOD` and
  `KICAD5..10_3DMODEL_DIR` env vars to the klibs `view/3dmodels` (env vars,
  not `--define-var` — probe finding 3); capture stderr and parse
  `File not found` lines into a **sidecar provenance JSON**
  (`<out>.meta.json`: kicad-cli version, exact argv, env overrides, sha256 +
  size of the output, confessed missing-model list, wall time). Non-zero
  exit or absent output → hard fail; warnings never silently pass (finding 2).
- `render(pcb, out_dir, views=...)`: named viewpoint presets — `top`,
  `bottom`, `iso_front` (`--rotate '-45,0,45' --perspective`), `iso_back` —
  each preset a frozen argv fragment; output naming
  `<board>__<view>.png` + the same sidecar treatment; `--quality` and
  `--width/--height` pass-through with defaults (basic, 1600×900).
- Small CLI (`python -m wyred_kicad.threed <pcb> --out DIR`) so agents and
  humans get it without writing Python.
- **Acceptance (mechanical):** on watchy: STEP exists and is > 1 MB; sidecar
  parses as JSON, records version `10.0.x` and a missing-model list equal to
  the resolver's (the step 1.3 differential, now wired as a wyred-kicad
  test); renders exist for all presets at the requested dimensions (PNG
  header check); a deliberately bad pcb path exits non-zero.

### Step 2.2 — smoke: STEP + renders for the t9y watchy and MPPT boards

- **Repo:** `wyred-kicad` (test/script only) · **~LOC:** ~100 ·
  **Model:** opus-4.8
- Run the 2.1 wrappers over three boards spanning the format range:
  `wyred-t9y/view/watchy/Watchy.kicad_pcb` (KiCad 7 format),
  `wyred-t9y/view/libre-solar-mppt-1210-hus/kicad/mppt-1210-hus.kicad_pcb`
  and `.../libre-solar-mppt-2420-hc/kicad/mppt-2420-hc.kicad_pcb` (KiCad 5
  format) — once in hermetic mode (klibs view) and once against the local
  KiCad bundle, recording both sidecars.
- All outputs under a gitignored out/scratch directory — **nothing enters
  goldens, artifact trees, or the gate**; the corpus trees are read-only
  inputs (t9y byte-identity untouched — `materialize.py --check` still
  passes after the smoke).
- **Determinism: measure and record, do not assert.** Re-run each render and
  STEP export twice; compare sha256; write the observed stability into the
  smoke report (probe observed byte-identical renders back-to-back on this
  machine; **STEP is confirmed NOT byte-stable** — its `FILE_NAME` header
  embeds the output filename and a wall-clock timestamp, and two identical
  same-name exports hashed differently — the report must say so and any
  future STEP comparison must strip/ignore that header). Renders may not be
  byte-stable elsewhere; no hash is promoted to a golden or gate check.
- Confessed-missing lists asserted against a checked-in expectation per
  board (watchy: the 10 `${KIPRJMOD}` models + any `.wrl`-twin gaps;
  mppt-1210-hus: `LibreSolar.3dshapes`) — the expectation file is a test
  fixture in wyred-kicad, not an artifact.
- **Acceptance (mechanical):** for each of the 3 boards: STEP > 500 KB, ≥ 3
  render PNGs each > 20 KB with valid PNG headers; smoke report
  (JSON/markdown in the out dir) records per-file sha256 across the double
  runs and the hermetic-vs-local sidecar diff; `wyred-t9y materialize.py
  --check` passes afterward; `git status` across wyred-wz members and
  wyred-t9y shows no new tracked files outside wyred-kicad's own
  code/test-fixture additions.

---

## License mechanics (recorded per the invariant)

- **Upstream terms:** kicad-symbols, kicad-footprints, kicad-packages3D are
  **CC-BY-SA-4.0 with the KiCad libraries exception** — *designs produced
  using the libraries carry no copyleft obligation*; *redistributing the
  libraries themselves* requires attribution + share-alike + preserved
  license text.
- **How this plan satisfies them:** the GitHub forks are unmodified,
  full-history server-side copies with upstream LICENSE files intact —
  attribution and share-alike satisfied by construction; they exist for
  availability/pinning only, never divergence. `wyred-klibs` itself
  redistributes nothing: `view/` is gitignored and materialized on demand
  from the forks; the materializer copies the upstream `LICENSE.md` into
  every materialized view root so any downstream copy of a view tree carries
  its terms. Boards, STEPs, and renders produced *from* the libraries are
  covered by the libraries exception — no obligations attach to wyred
  outputs.
- Contrast with wyred-t9y's per-design patchwork (CERN-OHL variants, MIT,
  LICENSE-UNKNOWN download-only): klibs is uniform, so no vendored-override
  or never-redistribute machinery is needed.

## Out of scope / deferred

- **Rung 2 (`wyred-3d`, anchorscad/pythonopenscad bridge)** — awaits
  `Wyred3dDesign.md`. Rung 1's only obligations to it: the resolver (1.3)
  stays a plain importable library, and the STEP→pythonopenscad conversion
  route decision (design §6) is deliberately not taken here.
- Gate wiring of any 3D artifact — 3D is advisory (design §5); nothing here
  changes the gate or goldens.
- AI visual inspection (§3.6) — consumes rung 1 renders but is its own
  workstream.
- Windows/Linux kicad-cli path discovery — wrappers take an explicit binary
  path with the macOS default; portability when a second platform actually
  appears.
- Re-pin automation for KiCad upgrades — manual manifest edit is the v1
  mechanism.
