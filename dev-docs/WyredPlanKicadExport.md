# Wyred plan — KiCad export workstream (`wyred-kicad`)

*Phased plan for WyredWorkflowDesign.md §3.1 (the gating item of the g19++
workflow): the L2→KiCad netlist exporter, its round-trip oracle, and the fab
delegation wrappers. Phases are milestones (coherent, shippable increments);
steps are one goal each, aspirationally < 500 LOC (advisory, not a hard
limit); foundational steps land first and cross-step coupling is minimized so
steps can be picked up independently. No code in this document — plan only.*

Source design: [WyredWorkflowDesign.md](./WyredWorkflowDesign.md) §3.1.
Boundary rules honored throughout: `wyred-kicad/CLAUDE.md` (depends on
wyred-contract only, no engine/harness imports, KiCad knowledge never leaks
out of wyred-kicad), `wyred-contract/CLAUDE.md` (single-writer),
[RunnerSplit.md](./RunnerSplit.md) (process-level composition, never imports).

---

## Invariants (binding on every step)

- **Goldens stay byte-identical.** This workstream adds **no** new emit-time
  artifacts to existing corpus output. The exporter is a *post-emit consumer*:
  it reads `*.l2.json` (and friends) from disk and writes its outputs to its
  own output directory — never into `wyred-examples/out/` as part of emit,
  never into `wyred-contract/goldens/`. A fresh emit of
  `wyred-examples/corpus` must remain byte-identical to
  `wyred-contract/goldens/ga019` after every step.
- **`run_gate.py` stays green.** Every step ends with
  `python3 /Users/owebeeone/limbo/wyred-wz/wyred-examples/run_gate.py`
  exiting 0. New gate stages join as *stage-when-present* with the loud-SKIP
  pattern (the board-agreement-probe precedent) — never a silent skip, never
  a hard new dependency for existing consumers.
- **Cross-repo composition is subprocess-only.** wyred-kicad never imports
  wyred, wyred-harness, or wyred-audit (and vice versa); stages meet only at
  artifacts on disk. KiCad itself is reached by subprocess only (`kicad-cli`,
  or KiCad's own bundled Python interpreter for pcbnew scripting) — never by
  in-process import from wyred code. Read-direction code needed here is
  **ported** (copied into `wyred_kicad/`), not imported from wyred-harness,
  whose frozen 149-check selftests keep their own copies untouched.
- **Agents never commit.** Steps leave changes in the working tree; Gianni
  reviews and commits.
- **Contract changes land only via a proposal step.** Single-writer flow: the
  proposal is written in `wyred-wz/dev-docs/`, and lands in `wyred-contract`
  (with a version bump and changelog entry) only after acceptance — never as
  a side effect of feature work in another repo. Regenerating goldens is a
  contract event, and this plan requires none.

---

## Observed toolchain reality (recorded 2026-07-12, kicad-cli 10.0.4 at `/opt/homebrew/bin/kicad-cli`)

Recorded here because the plan's mechanics depend on them; re-verify against
the pinned KiCad version at execution time.

- Top-level subcommands: `fp, jobset, pcb, sch, sym, version`.
- `kicad-cli sch`: `erc, export, upgrade`. `sch export` offers
  `bom, dxf, hpgl, netlist, pdf, ps, python-bom, svg`.
- `kicad-cli sch export netlist --format` accepts
  `kicadsexpr` (default), `kicadxml, cadstar, orcadpcb2, spice, spicemodel,
  pads, allegro`. A live export (10.0.4) emits the header
  `(export (version "E") (design (source ...) (date ...) (tool "Eeschema
  10.0.4")) ...)` — the same `(export (version "E") ...)` dialect
  `wyred-harness/harness/kicad_adapter.py` already parses (its canned sample
  is verified against 10.0.4; the dialect has been stable since KiCad 5).
  **This dialect is what our exporter emits.**
- `kicad-cli pcb`: `drc, export, import, render, upgrade`. `pcb export`
  offers `3dpdf, brep, drill, dxf, gencad, gerbers, glb, hpgl(dead), ipc2581,
  ipcd356, odb, pdf, ply, pos, ps, stats, step, stl, stpz, svg, u3d, vrml,
  xao`. DRC: `kicad-cli pcb drc --exit-code-violations --format json
  [--schematic-parity]`.
- **There is NO netlist/project import subcommand.** `kicad-cli pcb import`
  converts only *non-KiCad* board formats (`pads, altium, eagle, cadstar,
  fabmaster, pcad, solidworks`); `sch` has no import at all. The design's
  round-trip ("import into a KiCad project") therefore CANNOT be done with
  kicad-cli alone. The viable headless vehicle, verified locally: the KiCad
  app bundle ships its own Python (`/Applications/KiCad/KiCad.app/Contents/
  Frameworks/Python.framework/Versions/Current/bin/python3`, Python 3.9.13)
  with `pcbnew` 10.0.4 importable — a netlist→board builder script run under
  *that* interpreter as a subprocess is still pure delegation (KiCad's own
  engine realizes the netlist and writes the `.kicad_pcb`). Phase 2 is
  planned around this, with kicad_adapter-style graceful degradation when
  the toolchain is absent.
- Version pinning: `kicad-cli version --format plain|commit|about`.
- Operational rules already hard-won in kicad_adapter.py and inherited here:
  judge kicad-cli success by exit code + non-empty output file only (stderr
  always carries harmless Fontconfig noise); legacy Eeschema v4 `.sch` can
  never be netlisted headlessly (exit 0, silently empty — refuse, don't
  trust).

Golden corpus shape (input universe): `wyred-contract/goldens/ga019` holds 18
artifact stems, of which **16 have an `.l2.json`** (the two L1-error cases
`intent_03_addr_collision` and `intent_07_voltage_mismatch` do not — the
exporter must skip these *loudly*, listing what it skipped and why). No
golden L2 component today carries a `footprint` attr (checked: 0 of 82
components on `watchy_v1_reva`), which is why footprint binding needs both a
contract attr (for future emits) and a sidecar map (for the frozen corpus).

---

## Phase 1 — L2→KiCad netlist exporter (milestone: any L2 artifact becomes a KiCad-loadable `.net`, deterministically, with tests)

Shippable increment: `python3 -m wyred_kicad.export` turns every golden L2
into a syntactically valid, deterministic KiCad s-expression netlist with
footprint bindings, and a committed test suite proves it over the whole
corpus. After this phase a human can take a wyred board into KiCad layout by
hand (GUI netlist import), even before the Phase-2 oracle exists.

### Step 1.1 — Contract proposal: `.net` export conventions + `footprint` attr schema

- **Repo:** `wyred-wz/dev-docs` (proposal) + `wyred-contract` (landing, after
  acceptance) + `wyred-kicad` (EXPORT.md conventions doc).
- **~LOC:** ~150 (documentation deltas only; no code).
- **Model:** fable (contract/semantic design — the only step that needs it).
- **Goal:** one accepted design for (a) what the exporter emits and (b) how
  footprints are declared, respecting the repo boundaries: EDA-*neutral*
  schema lands in wyred-contract; KiCad-*specific* conventions land in
  wyred-kicad (its CLAUDE.md forbids KiCad knowledge leaking out).
- **Deliverables:**
  - `dev-docs/ContractProposalKicadExport.md` — the proposal, covering:
    - **Contract delta (EDA-neutral, lands in EMIT_CONTRACT.md):** an
      optional per-component `attrs["footprint"]` string — an opaque
      qualified binding token (recommended shape `<lib>:<name>`), emitted
      only when set (same backward-compatible pattern as the v3 ground
      fields), ignored by the existing oracle stack. Rationale: `footprint`
      is already a decision class in `schema_l1.DECISION_CLASSES`, so pinned
      /locked footprint decisions get a place to surface in L2. Also specify
      how the parts library (wyred-parts, when it lands) will carry
      `footprint=` attrs on part definitions — recorded as the future source
      of these attrs, explicitly out of scope for this workstream.
    - **Exporter conventions (KiCad-specific, land in
      `wyred-kicad/EXPORT.md`):** target dialect `(export (version "E") ...)`
      (`kicadsexpr`); determinism rules — no wall-clock `(date)`, `(source)`
      is the artifact stem not an absolute path, `(tool "wyred-kicad
      <version>")`, stable ordering of comps/libparts/nets (byte-identical
      double export is an acceptance gate); refdes/value/pin mapping from
      `schema.CanonicalGraph` fields; net naming (including how wyred net
      names map to KiCad net-name lexical rules); terminal role →
      KiCad pintype mapping (inverse of kicad_adapter's `_PINTYPE_ROLE`,
      with the lossy cases named, not hidden);
      **footprint resolution ladder** — pinned decision (lock class
      `footprint`) → L2 `attrs["footprint"]` → sidecar map
      (`--footprints map.json`: kind-level defaults + per-refdes overrides;
      the interim source for the frozen goldens) → confessed placeholder
      footprint plus a loud STUBBED note in the export report (no silent
      defaults, kicad_adapter precedent); KiCad-version pin recorded in the
      export report sidecar.
    - **Round-trip report shape + structured codes** consumed by Phase 2
      (e.g. `KEXPORT_ROUNDTRIP_DISAGREE`, `KEXPORT_PARSE_FAIL`,
      `KEXPORT_FOOTPRINT_STUBBED`, `KEXPORT_TOOLCHAIN_UNAVAILABLE`) so 2.2
      is purely mechanical.
  - After Gianni accepts: the contract delta lands in
    `wyred-contract/EMIT_CONTRACT.md` with a version bump + changelog entry
    (single-writer flow); `wyred-kicad/EXPORT.md` lands in wyred-kicad.
    **No golden changes.** Gianni commits.
- **Acceptance (mechanical):**
  `test -f /Users/owebeeone/limbo/wyred-wz/dev-docs/ContractProposalKicadExport.md && test -f /Users/owebeeone/limbo/wyred-wz/wyred-kicad/EXPORT.md && grep -q footprint /Users/owebeeone/limbo/wyred-wz/wyred-contract/EMIT_CONTRACT.md && [ -z "$(git -C /Users/owebeeone/limbo/wyred-wz/wyred-contract status --porcelain -- goldens)" ]`

### Step 1.2 — The exporter: L2 JSON → KiCad netlist s-expression

- **Repo:** `wyred-kicad`.
- **~LOC:** ~350 (package skeleton + minimal local L2 reader + writer + CLI).
- **Model:** opus-4.8 (mechanical build against 1.1's accepted conventions).
- **Goal:** `python3 -m wyred_kicad.export` converts one L2 artifact — or a
  whole artifact tree — into `.net` files with footprint bindings, exactly
  per EXPORT.md.
- **Deliverables:**
  - `wyred-kicad/wyred_kicad/` package: `schema_io.py` (minimal, local,
    dict-based reader for the EMIT_CONTRACT L2 shape — wyred-kicad depends on
    wyred-contract *documents* only, so it owns its own small reader rather
    than importing harness code), `netwrite.py` (the s-expr writer),
    `export.py` + `__main__` CLI:
    `python3 -m wyred_kicad.export --l2 FILE | --tree DIR [--footprints MAP]
    --out DIR`.
  - `--tree` mode walks `*.l2.json`, emits `<stem>.net` plus a per-stem
    export report JSON (footprint ladder outcomes, stub confessions,
    exporter version) into `--out`; artifacts without `.l2.json` are listed
    as skipped with the reason.
  - A committed sidecar `wyred-kicad/footprints/ga019.map.json` giving
    kind-level default footprints (plus any per-refdes overrides needed) for
    the golden corpus, so all 16 sets export without placeholder confessions
    — placeholders remain exercised via a test fixture, not the corpus.
  - Determinism: two runs over the same tree are byte-identical (no dates,
    no absolute paths, stable ordering).
- **Independence note:** needs only 1.1's EXPORT.md; does not depend on 1.3,
  Phase 2, or any harness/engine code.
- **Acceptance (mechanical):**
  `cd /Users/owebeeone/limbo/wyred-wz && python3 -m wyred_kicad.export --tree wyred-contract/goldens/ga019 --footprints wyred-kicad/footprints/ga019.map.json --out /tmp/kexp.1 && python3 -m wyred_kicad.export --tree wyred-contract/goldens/ga019 --footprints wyred-kicad/footprints/ga019.map.json --out /tmp/kexp.2 && diff -r /tmp/kexp.1 /tmp/kexp.2 && ls /tmp/kexp.1/*.net | wc -l | grep -q 16`
  (run with `PYTHONPATH=/Users/owebeeone/limbo/wyred-wz/wyred-kicad`), plus
  `run_gate.py` still exits 0 and goldens porcelain-clean.

### Step 1.3 — Exporter tests over all 16 golden artifact sets

- **Repo:** `wyred-kicad`.
- **~LOC:** ~150 new test code (the ported parser is a file copy with
  import-path adjustments — counted as a port, not against the budget).
- **Model:** opus-4.8.
- **Goal:** a committed regression suite proving every golden L2 exports to a
  netlist that parses back to the same connectivity, with syntax validated
  against the observed KiCad grammar.
- **Deliverables:**
  - Port into `wyred_kicad/`: `kicad_adapter.parse_exported_netlist` plus the
    `normalize.py` s-expr helpers and the minimal `schema.py` graph classes
    it needs (port sources per wyred-kicad/CLAUDE.md:
    `/Users/owebeeone/limbo/wyred-wz/wyred-harness/harness/kicad_adapter.py`,
    `normalize.py`, `schema.py`). Harness copies stay untouched.
  - `wyred-kicad/tests/run_export_tests.py` (own entry point, subprocess
    style like `run_gate_tests.py`): for each of the 16 golden sets —
    export → parse back with the ported (independently written, hardened
    against genuine kicad-cli output) parser → assert refdes set, per-pin net
    partition, net kinds, and footprint bindings match the source L2;
    placeholder-footprint fixture produces the confessed STUBBED report
    entry; the two L1-only stems are reported skipped; double-export
    byte-identity.
  - **Syntax validation, honestly scoped:** kicad-cli has no netlist
    import/validate subcommand (observed above), so 1.3's "kicad-cli syntax
    validation" is: (a) grammar conformance of our output against a live
    kicad-cli reference — when kicad-cli is present, netlist the adapter's
    known-good probe schematic and assert our emitted dialect matches its
    observed structure (header, section shapes); (b) full parse-back of all
    16 exports. True KiCad-engine *consumption* of our `.net` is Phase 2.1's
    job and is explicitly not claimed here. When kicad-cli is absent the
    reference check degrades to a loud skip line (UNAVAILABLE_NOTE pattern),
    never a silent pass.
- **Acceptance (mechanical):**
  `python3 /Users/owebeeone/limbo/wyred-wz/wyred-kicad/tests/run_export_tests.py`
  exits 0; `python3 /Users/owebeeone/limbo/wyred-wz/wyred-examples/run_gate.py`
  exits 0.

---

## Phase 2 — Round-trip oracle (milestone: the exporter is oracle-backed and gate-wired; a tampered export turns the gate red)

Shippable increment: export → KiCad's own engine realizes the netlist as a
board → read back → differential against the source L2, wired into
`run_gate.py` as a stage-when-present. This is the "every stage gets an
oracle" obligation for §3.1, and it permanently regression-tests the
exporter.

### Step 2.1 — Import harness: netlist → KiCad board → re-extraction

- **Repo:** `wyred-kicad`.
- **~LOC:** ~200 new (builder script + driver CLI; the `pcb_extract.py` port
  is a file copy, counted as a port).
- **Model:** opus-4.8.
- **Goal:** a subprocess pipeline that makes KiCad itself consume our `.net`
  and hands back a `CanonicalGraph`, degrading loudly (never silently) when
  the toolchain is absent.
- **Deliverables:**
  - Port `pcb_extract.py` into `wyred_kicad/` (read-direction heritage; same
    port rules as 1.3).
  - `wyred_kicad/boardbuild.py`: a **pcbnew builder script executed under
    KiCad's bundled Python as a subprocess** (interpreter discovery mirrors
    `find_kicad_cli`: env override → app-bundle path; returns None-equivalent
    when absent). It parses the `.net` (stdlib-only inside the script),
    creates a board, synthesizes one minimal footprint per component with
    one pad per terminal (no dependence on footprint *libraries* — the
    `footprint` binding string is stamped on the footprint but library
    lookup is out of scope, keeping the oracle hermetic), binds pads to
    nets, and saves `.kicad_pcb`. KiCad's own engine thus performs the
    netlist→board realization and writes the file.
  - `python3 -m wyred_kicad.roundtrip --net FILE --out DIR`: build board →
    re-extract via the ported `pcb_extract` → optionally also
    `kicad-cli pcb export ipcd356` as KiCad's own netlist-of-the-board
    re-export (a second, fully-KiCad-authored read-back leg when available)
    → write a round-trip report JSON including the pinned KiCad version
    (`kicad-cli version --format about`, bundled-python `pcbnew.Version()`).
  - Toolchain-absent behavior: exit 0 with a report containing
    `KEXPORT_TOOLCHAIN_UNAVAILABLE` and a loud note — the kicad_adapter
    graceful-degradation contract.
- **Independence note:** consumes any `.net` file; can be developed against a
  canned Phase-1 export fixture, independent of 2.2.
- **Risk logged:** pcbnew's SWIG Python API varies across KiCad majors (and
  KiCad is migrating toward the IPC API); the builder script pins to the
  bundled interpreter and asserts `pcbnew.Version()` matches kicad-cli's,
  recording both in the report. If the SWIG route proves unusable at some
  future KiCad version, the fallback rung is parse-back-only round-trip
  (1.3's oracle) plus a recorded deferral — never a pretend import.
- **Acceptance (mechanical):**
  `python3 -m wyred_kicad.export --l2 /Users/owebeeone/limbo/wyred-wz/wyred-contract/goldens/ga019/intent_01_sensor_node.l2.json --footprints /Users/owebeeone/limbo/wyred-wz/wyred-kicad/footprints/ga019.map.json --out /tmp/krt && python3 -m wyred_kicad.roundtrip --net /tmp/krt/intent_01_sensor_node.net --out /tmp/krt`
  exits 0 and `/tmp/krt/intent_01_sensor_node.kicad_pcb` exists (or the
  report carries `KEXPORT_TOOLCHAIN_UNAVAILABLE` on a machine without KiCad).

### Step 2.2 — Differential vs source L2, gate-wired; tampered export fires

- **Repo:** `wyred-kicad` (differential + tests) and `wyred-examples`
  (run_gate stage — the only repo allowed to compose everything).
- **~LOC:** ~200 (differential wrapper + gate stage + negative tests; the
  `kicad_adapter.compare` port is a file copy, counted as a port).
- **Model:** opus-4.8 (codes and report shape were fixed in 1.1; this is
  mechanical).
- **Goal:** the round-trip becomes a verdict: source L2 vs KiCad-realized
  board disagreement is a structured, gate-red failure, and the negative
  test proves the oracle can fire.
- **Deliverables:**
  - `wyred_kicad/differential.py`: ported `compare()` (Rand-index similarity
    + named-net-merge warnings) applied L2-graph vs round-trip-extracted
    graph, **tightened for this use**: unlike the cross-*engine* board case,
    the exporter round-trip has no legitimate testpoint/logo noise, so the
    pass bar is exact partition agreement on the exported node universe
    (footprint-stub terminals excluded by explicit, reported rule — every
    exclusion named, none silent). Emits the 1.1 structured codes
    (`KEXPORT_ROUNDTRIP_DISAGREE` etc.) in the report; nonzero exit on any
    Tier-A code. CLI: `python3 -m wyred_kicad.differential --l2 FILE
    --roundtrip DIR`.
  - `run_gate.py` stage **5, stage-when-present**: runs iff the sibling
    `wyred-kicad` checkout and its footprint map are present (content-based
    readiness check, PROBE_REQUIRED_FILES precedent) — export the emitted
    `out/` tree's L2s to a *temp* directory (never into `out/`; the
    no-new-emit-artifacts invariant), round-trip + differential each, fail
    on first nonzero exit; loud SKIP banner naming exactly what was not
    verified when absent. Subprocess-only, matching every other stage.
  - Negative tests in `wyred-kicad/tests/run_roundtrip_tests.py`: (a) a
    tampered `.net` (one node moved to another net after export) makes the
    differential exit nonzero naming `KEXPORT_ROUNDTRIP_DISAGREE`; (b) a
    tampered source L2 against an honest export likewise fires; (c) the
    toolchain-absent path yields the loud degradation, not a pass-shaped
    silence.
- **Acceptance (mechanical):**
  `python3 /Users/owebeeone/limbo/wyred-wz/wyred-kicad/tests/run_roundtrip_tests.py`
  exits 0 (tamper cases fire) **and**
  `python3 /Users/owebeeone/limbo/wyred-wz/wyred-examples/run_gate.py`
  exits 0 with the new stage either RUN or loudly SKIPPED, and
  `[ -z "$(git -C /Users/owebeeone/limbo/wyred-wz/wyred-contract status --porcelain -- goldens)" ]`.

---

## Phase 3 — Fab delegation wrappers (milestone: one command from a laid-out board to a fab pack with a pinned toolchain)

Shippable increment: the pure-delegation conveniences of §3.1 — gerbers,
drill, position files, and DRC via kicad-cli — with the KiCad version pinned
in the output. No wyred logic beyond invocation, manifesting, and exit-code
discipline; layout itself remains delegated forever.

### Step 3.1 — `wyred_kicad.fab`: gerber/drill/pos/DRC wrapper + version-pinned manifest

- **Repo:** `wyred-kicad`.
- **~LOC:** ~150.
- **Model:** opus-4.8.
- **Goal:** `python3 -m wyred_kicad.fab --board FILE.kicad_pcb --out DIR`
  runs, as subprocesses: `kicad-cli pcb drc --exit-code-violations --format
  json`, `kicad-cli pcb export gerbers`, `... drill` (with `--generate-map`),
  `... pos --format csv --units mm`, and writes `fab_manifest.json`
  recording: full `kicad-cli version --format about` output (the version
  pin the design mandates — format drift is real, the corpus spans KiCad
  4→10), the exact argv of every invocation, exit codes, and sha256 of every
  produced file. DRC violations propagate as the wrapper's own nonzero exit.
  stderr judged by the exit-code+output-file rule, never inspected.
- **Determinism, honestly scoped:** gerber/drill outputs embed creation
  dates, so fab outputs are *manifested* (hashed, versioned, argv-recorded
  for the audit trail), never byte-goldened. The manifest itself is
  deterministic apart from the recorded hashes.
- **Test:** `wyred-kicad/tests/run_fab_tests.py` runs the wrapper against a
  real third-party board (`third-party/watchy/Watchy.kicad_pcb` via the
  `WYRED_THIRD_PARTY` resolution pattern), asserts the manifest names all
  four output families + the version pin, and that a board with known DRC
  violations (fixture: the Watchy board with an injected overlapping-track
  edit in a temp copy, or `--severity-error` on a board that carries any)
  yields nonzero. Loud skip when the t9y view or kicad-cli is absent.
- **Independence note:** consumes any `.kicad_pcb`; independent of Phases
  1–2 (parallelizable with Phase 2 once 1.1's report conventions exist).
- **Acceptance (mechanical):**
  `python3 /Users/owebeeone/limbo/wyred-wz/wyred-kicad/tests/run_fab_tests.py`
  exits 0; `python3 /Users/owebeeone/limbo/wyred-wz/wyred-examples/run_gate.py`
  exits 0.

---

## Step dependency map (parallelism)

```
1.1 (fable: conventions + contract proposal)
 ├─► 1.2 (exporter) ─► 1.3 (exporter tests) ─► 2.2 (differential + gate wiring)
 ├─► 2.1 (import harness; needs only a canned .net fixture) ──┘
 └─► 3.1 (fab wrappers; independent of 1.2/1.3/2.x code)
```

1.1 is the single foundational step. After it lands, 1.2, 2.1 (against a
canned fixture), and 3.1 can proceed in parallel by different agents; 2.2 is
the only step that joins two streams.

## Format risks (recorded)

1. **No headless netlist import in kicad-cli** (verified on 10.0.4): the
   design's round-trip phrasing requires the pcbnew bundled-Python route
   (2.1). SWIG-API drift across KiCad majors is the main fragility; the
   fallback is parse-back-only round-trip plus a recorded deferral.
2. **Netlist dialect is undocumented-but-stable**: `(export (version "E"))`
   has held from KiCad 5 through 10, but there is no formal grammar spec;
   our defense is the 1.3 live-reference check plus the hardened parser.
3. **KiCad version drift**: corpus experience spans KiCad 4→10 (legacy `.sch`
   headless netlisting silently empty; `hpgl` export dropped in 10). Every
   output that touches KiCad carries the version pin; EXPORT.md states the
   supported floor (KiCad ≥ 8, per the design's open decision).
4. **Fab outputs are not byte-deterministic** (embedded dates): manifested
   with hashes, never goldened (3.1).
5. **Footprint truth**: goldens carry no footprint attrs and wyred-parts is
   not yet built; the sidecar map is the interim binding source and real
   `footprint=` part attrs arrive with the wyred-parts workstream (hook
   recorded in 1.1, out of scope here).
