# WyredPlanContractSchemas — JSON Schema extraction for the emit contract

*Phased plan. Executes wyred-contract/CLAUDE.md's named first task: "extract
JSON Schemas from EMIT_CONTRACT.md prose." Model for every step: **opus-4.8**
(mechanical extraction with a strong oracle — the 107 ga019 goldens must all
validate). LOC budgets are aspirational (<500/step advisory).*

## Invariants (binding on every step)

1. **Schemas are DESCRIPTIVE of contract v0** (= the ga019 goldens as
   emitted). Any place a golden violates a drafted schema is a **finding to
   surface** in the findings log — never a golden to change, never a schema
   silently widened without a recorded finding. Goldens are untouched,
   byte-for-byte, throughout.
2. **wyred-contract is single-writer.** Every step leaves its output as
   uncommitted working-tree files in `wyred-contract/` plus an entry in the
   findings log (§ Findings below, extended in place). Nothing is committed;
   the whole Phase-1/Phase-2 output lands as **one staged proposal for
   Gianni's ratification** (DecisionLog entry + contract-version bump are
   part of ratification, not of these steps).
3. **Precedence when sources disagree:** goldens (normative fixtures per
   wyred-contract/CLAUDE.md) > harness/engine code (implicit schema) >
   EMIT_CONTRACT.md prose (documented intent). A disagreement at any tier is
   a finding; the schema encodes what the goldens actually do.
4. **No new dependencies anywhere.** wyred-contract allows "schema validation
   helpers" only; wyred-audit is pure-stdlib by fence. The validator is
   dependency-free (§ Step 1.2). The real `jsonschema` PyPI package may be
   used **only** as a throwaway dev-time cross-check oracle in a scratch
   venv — it never appears in any repo's requirements, imports, or tests.

## Source inventory (read before drafting anything)

| source | role |
|---|---|
| `wyred-contract/EMIT_CONTRACT.md` | prose contract: Part A (L2 `CanonicalGraph`), Part B (L1 intent doc), Part C (embedded allocation record), Part D (worked example) |
| `wyred-contract/goldens/ga019/` (107 files) | normative fixtures — the extraction oracle |
| `wyred-harness/harness/schema.py`, `schema_l1.py` | implicit schema: dataclasses + hand-rolled to/from-JSON with emit-only-when-set rules and controlled vocabularies |
| `wyred-harness/harness/allocation.py` (`snapshot_locks`, `check_lock_violations`) | baseline shape + lock-gate semantics |
| `wyred/src/wyred/emit.py`, `wyred/src/wyred/paths.py` | the actual **producers** of the seven artifact kinds the prose never specifies (bom/pinmap/records/baseline/connlock/pinmapdiff/lifecycle + the standalone alloc wrapper) |
| `wyred-audit/src/wyred_audit/` (`cli.py`, `harnesslib.py` locator pattern, `incumbency.py`, `connlock.py`) | Phase-2 integration surface and its composition fences |

## Artifact kinds in ga019 (all ten; 107 files total)

| kind | files | shape source | notes |
|---|---:|---|---|
| `l1` | 18 | EMIT_CONTRACT Part B + `schema_l1.to_json` | 8 distinct top-level key subsets observed (optional keys emitted only when non-empty); `forked_from` on `watchy_v1_revb` only |
| `l2` | 16 | EMIT_CONTRACT Part A + `schema.to_json` | fault-only intents 03/07 emit no l2; `ground_kind`/`ground_role`/`bond` emitted only when set |
| `alloc` | 16 | `wyred/emit.py` (`rr.alloc_json_str`) | **wrapper**, not bare Part C: `{allocation, bindings, bound_parts, connector_pinout, series, solver_version, stamp, wiring}` |
| `bom` | 16 | `wyred/paths.build_bom` | `{artifact, authored_total, component_total, generated_total, line_items[], path:"bom", stamp}`; `derived` map present exactly on `authored:false` items |
| `pinmap` | 16 | `wyred/paths.build_pinmap` | `{allocations[], artifact, components[], path:"pinmap", solver_version, stamp}`; terminal `net` may be `null` (unwired spare) |
| `records` | 16 | `wyred/paths.build_records` | `{allocation, artifact, bindings, bound_parts, connector_pinout, escalations[], path:"records", pool_spares, resolutions, series, solver_version, stamp}` (+ optional `forked_from` object) |
| `baseline` | 4 | `harness.allocation.snapshot_locks` + engine extra key | `{series, solver_version, groups:{name:{version, covers, entries}}}` **plus** `connector_pinout` rows added by `emit.py` (locked emits only: intent_05a_pinned, mppt_2420_hc_reva, watchy_v1_reva, watchy_v1_revb) |
| `connlock` | 3 | `wyred/emit.py` + `paths.check_connector_locks` | two variants: `fork_vs_parent_codes` present only on forks (watchy_v1_revb); mppt_2420_hc_reva, watchy_v1_reva, watchy_v1_revb |
| `pinmapdiff` | 1 | `wyred/paths.diff_pinmaps` + `emit._check_incumbency` | ECO diff (watchy_v1_draft_btn3): `{allocation:{added,changed,removed}, components_only_in_a/b, incumbents, minimal_disturbance_violations, stamp_a, stamp_b, terminals[]}`; terminal `a`/`b` nullable |
| `lifecycle` | 1 | `wyred/emit.py` fork path | watchy_v1_revb: `{artifact, child_stamp, forked_from, legal_fork_codes, new_series, ok, parent_stamp, tamper_locked_edit_codes, tamper_series_hand_edit_codes}` |

Shared sub-shapes to factor into common `$defs`: **stamp**
(`{series, locks:{<group>: <version int>}}`), **allocation record** (Part C:
entries + lock_groups + solver_version), **allocation entry**
(`{pool, unit, demand, chosen_by, state, locked_by}`), **connector-pinout
row** (`{connector, pin, function, net|null, locked_by|null}`), **bound-part
row** (`{chosen_by, part, refdes}`), **wiring row**
(`{demand, net, node:"REFDES.PIN", pool, unit}`), **forked_from record**
(`{series, reason}`).

## Findings already established during plan research (F1–F9)

These are pre-findings the schema drafts must encode and step 1.3 must
re-verify mechanically. Each is a prose/golden/harness disagreement or gap —
**all resolve in favor of the goldens** (invariant 1).

- **F1 — the prose contract covers 3 of 10 artifact kinds.**
  EMIT_CONTRACT's header says the builder emits "up to three JSON artifacts
  per intent"; ga019 emits up to ten kinds per artifact name. The standalone
  `.alloc.json` wrapper and bom/pinmap/records/baseline/connlock/pinmapdiff/
  lifecycle exist only in goldens + engine/harness code (pinmap is mentioned
  once, in passing, in Part D.3's notes as "not shown; a v3 data path").
  Consequence: seven-plus schemas are extracted from goldens + producer code,
  and Phase 2.2 proposes a prose addendum (Part E) pointing at them.
- **F2 — `Component.kind` vocabulary: 12 declared vs 35 used.**
  `schema.COMPONENT_KINDS` enumerates 12 kinds ("not enforced hard" per its
  own comment); the l2 goldens use 35 (antenna, buck_regulator,
  can_transceiver, charger, connector, current_sense, fiducial, flash, fuse,
  fuse_holder, inductor, jumper, led, mechanical, motor, resonator,
  supervisor, switch, test_point, thermistor, transistor, usb_protection,
  varistor, + the original 12). Prose never enumerates kinds at all.
  Consequence: `kind` is an open string in the schema, with the observed
  vocabulary listed non-normatively in the schema description.
- **F3 — ground-net `voltage` contradicts the prose field table.** Part A
  says voltage is "rail voltage in volts for power nets …, else `None`", yet
  every golden ground net carries `voltage: 0.0` — as does the prose's own
  worked example D.3. Internal prose inconsistency; schema allows
  `number|null` on every net kind.
- **F4 — `forked_from` means two different things.** In `l1` and `records`
  it is the object `{series, reason}` (Part C's series-fork record); in
  `lifecycle` it is a **string** naming the parent artifact
  (`"watchy_v1_reva"`). Legal per-kind, but a documentation hazard: each
  schema defines it independently and the descriptions cross-reference.
- **F5 — `snapshot: []` on a locked group.** Harness docstring says snapshot
  is "None if never locked", and Part C says locking "freezes the covered
  entries". watchy_v1_reva/_revb's `external-interface` group is locked
  (`version: 1`) with `snapshot: []` — its covers (`connector_pinout`) never
  materialize as allocation entries; those rows are baselined externally
  (F6). Schema: `null | array`, empty array legal at any version.
- **F6 — the baseline artifact is snapshot_locks output plus an extra key.**
  `harness.allocation.snapshot_locks` documents/returns
  `{series, solver_version, groups}`; the golden baselines additionally carry
  `connector_pinout` (added by `wyred/emit.py`, which notes "extra keys are
  ignored by the harness gate"). Schema follows the golden four-key shape;
  the harness docstring is the stale tier.
- **F7 — two escalation encodings under one field name.** L2
  `graph.escalations` is an array of **strings** (harness `List[str]`;
  golden intent_05b confirms); L1/records `escalations` are rich objects
  (`{code, msg, subjects, conflict, relaxation}`). Prose Part A never states
  the L2 element type. Per-kind schemas; description notes the split.
- **F8 — EMIT_CONTRACT.md has no changelog section.** wyred-contract's
  CLAUDE.md requires every change to be "recorded in this repo's changelog
  section of EMIT_CONTRACT.md" — that section does not exist yet. Creating
  it is part of the Phase-2.2 staged proposal.
- **F9 — node encodings differ by artifact.** L2 nets encode nodes as
  `[refdes, terminal]` pairs; alloc `wiring` and pinmap `allocations` encode
  the same thing as a dotted string `"REFDES.PIN"` (`"U1.U0_RX"`, `"U4.5"`).
  Legal per-kind; the schemas document both encodings and the hazard.

Non-findings verified while reading (no action): `solver_version` is a free
string (`"ga019-trivial-1"` vs prose's `"trivial-1"` example); every
`authored:false` BOM line item carries `derived` and no authored one does;
`qty`/`default` demand fields are near-unused in goldens (one `qty`, zero
`default`) but prose-specified, so they stay in the schema; golden files are
serialized `json.dumps(indent=2, sort_keys=True)` — key order is a
serialization convention, out of JSON Schema's scope (the validator may check
it as an opt-in canonical-form extra, § Step 1.2).

## Findings added during extraction

- **F10 — bom `derived` is a biconditional in the goldens, tighter than the
  producer.** `wyred/paths.py::build_bom` emits a line item's `derived` map
  only when it is non-empty (`if derived:`), so the producer guarantees only
  the *forward* implication `derived ⇒ authored:false`. But across all 16 bom
  goldens every `authored:false` (generated) line item carries a non-empty
  `derived` confession and no `authored:true` line item carries one — the full
  **iff**. Per invariant 1 (descriptive-of-goldens, goldens > code), the schema
  encodes the golden-observed biconditional via a two-branch `oneOf` in
  `bom.schema.json` (`line_item`): the `authored:true` branch omits `derived`
  (and `additionalProperties:false` forbids it), the `authored:false` branch
  requires it. A future generated-but-underived line would be a contract event,
  not a silent widening. (This was foreseen as a non-finding during research;
  promoted to a finding because it forced a schema-encoding decision stricter
  than the producer.)
- **F11 — landed net/node fields pinned to their golden-narrow non-null
  shape.** Three fields come from `dict.get(...)` look-ups that *can* return
  `None` in the producers yet are never null in any ga019 golden:
  `alloc.wiring[].net` (`resolve.py`: `self.b.node_net.get(node)`),
  `pinmap.allocations[].node` and `pinmap.allocations[].net` (`paths.py
  build_pinmap`: `w["node"]/w["net"] if w else None`). The schemas type all
  three as **non-null strings** (`common.defs.json#/$defs/wiring_row`,
  `pinmap.schema.json#/$defs/pinmap_alloc_row`), tight-to-goldens. By contrast
  the fields that *do* show null in the goldens — `connector_pinout_row.net`,
  `connector_pinout_row.locked_by`, `pinmap_terminal.net` — are typed
  `string|null`. Same class: the pin-map component provenance is a strict XOR
  (`l1_role` xor `for_demand`, 263/586 split, never both/neither) though
  `build_pinmap`'s two independent `if attrs.get(...)` guards could in
  principle emit both or neither; the schema encodes the XOR via `oneOf`.
  Widening any of these to the producer's looser shape would require a recorded
  contract event.
- **F12 — lock-gate code arrays are closed to their gate producer's two-code
  vocabulary; the emit-side sentinels are excluded.** The connlock and lifecycle
  artifacts carry arrays of gate codes. Deciding the connlock vocabulary from
  `wyred/paths.py::check_connector_locks` (the plan-required read): that function
  is the SOLE producer of every code that reaches `connlock.gate_clean`,
  `tamper_net_codes`, `tamper_series_codes`, and `fork_vs_parent_codes`, and it
  emits exactly two hardcoded literals — `CONNECTOR_LOCK_VIOLATION` and
  `CONNECTOR_SERIES_UNJUSTIFIED` (no other `"code":` path exists). The vocabulary
  is therefore **proven closed**, so `connlock.schema.json` enum-constrains those
  four arrays to those two codes (duplicates allowed — the gate emits one code per
  violating row). The parallel `wyred/emit.py::check_lock_violations` is likewise
  the sole producer of `lifecycle.legal_fork_codes`,
  `tamper_series_hand_edit_codes`, and `tamper_locked_edit_codes`, emitting exactly
  `LOCK_VIOLATION` and `SERIES_UNJUSTIFIED`; `lifecycle.schema.json` closes those
  three arrays the same way. Two caveats resolved in favor of the goldens
  (invariant 1/3): (a) `emit.py` can inject a non-gate SENTINEL string —
  `"(no parent baseline)"` into `fork_vs_parent_codes` (only when a fork's parent
  emitted no baseline, an `ok:false` broken-emit path) and
  `"(no locked entry to tamper)"` into `tamper_locked_edit_codes` (only when the
  parent has no locked entry to probe) — but NEITHER sentinel appears in ANY
  golden, so encoding it would be a silent widening; the schemas exclude it.
  (b) The goldens only ever exercise these codes in the tamper-probe arrays
  (`fork_vs_parent_codes`/`gate_clean`/`legal_fork_codes` are all `[]`); the
  enum is nonetheless the correct tight encoding because an empty array satisfies
  it and no golden is rejected. A future artifact carrying a sentinel, a third
  code, or a code in a now-empty array is a **contract event**, not a schema to
  pre-widen. (This was the connlock open-vs-closed decision the plan's Step 1.1c
  required to be made from `check_connector_locks` and recorded here.)

---

## Phase 1 — Extraction (milestone: all 107 goldens machine-validated against checked-in schemas)

Shippable increment: `wyred-contract/schemas/` with one draft-2020-12 schema
per artifact kind, a dependency-free validator CLI, and a green (or
findings-annotated) validation run — all staged, uncommitted.

Every schema file: `$schema: "https://json-schema.org/draft/2020-12/schema"`,
`$id: "https://wyred.dev/contract/v0/<kind>.schema.json"` (stable naming; the
host is a namespace, not a URL that must resolve), a top-level
`x-wyred-contract: {"contract": "v3-ga019", "schema_rev": 0}` stamp
(§ Step 2.2 fixes the convention), and `additionalProperties: false` wherever
the goldens show a closed shape (which is everywhere except `attrs` free
dicts) — descriptive means *tight*: the schema should reject anything the
emitter never produced, because every widening must be a conscious finding.
Shared sub-shapes go in `schemas/common.defs.json` and are referenced via
`$ref` with relative URIs; the Step-1.2 validator must support exactly that
reference style.

### Step 1.1a — Layer-doc schemas: `l1.schema.json`, `l2.schema.json`, `common.defs.json`
- **repo:** wyred-contract (working tree only) · **~LOC:** 350 (schema JSON + findings-log lines) · **model:** opus-4.8
- Extract Part B + `schema_l1.to_json` into `l1.schema.json`: all eleven
  object kinds (Role/Capability/Demand/Scope/Rail/Ground/Bond/Bus/Pool/
  InvariantDecl/Escalation) plus the embedded Part-C allocation record
  (entries, lock_groups incl. `owner`/`sync_point` optionality and F5's
  `snapshot: null|array`, `solver_version`), `forked_from` object, `layer:
  const 1`, and the emit-only-when-set optionality rules (`scopes`, `bonds`,
  `buses`, `pools`, `invariants`, `escalations`, `attrs` optional; `qty`
  default-elided; etc. — mirror `_put`). Extract Part A + `schema.to_json`
  into `l2.schema.json`: components/terminals (all seven always-emitted
  terminal keys), nets (`nodes` as `[refdes, terminal]` 2-tuples via
  `prefixItems`, F3 voltage `number|null`, optional ground fields with
  `GROUND_KINDS`/`GROUND_ROLES` enums), invariants, escalations as string
  array (F7). Enums: encode `TERMINAL_ROLES`, `NET_KINDS`, `LOGIC_FNS`,
  ground vocab, `CHOSEN_BY`, `ALLOC_STATES`, `DECISION_CLASSES`,
  `INVARIANT_KINDS` as enums (goldens stay within all of them); leave
  `Component.kind` and every `iface` open per F2.
- **Acceptance (mechanical):** in a scratch venv with the dev-only
  `jsonschema` oracle, all 18 `.l1.json` and 16 `.l2.json` goldens validate;
  a 5-mutant spot check (drop a required key, wrong enum value, extra
  top-level key, string-for-number, 3-element net node) is rejected. Any
  golden failure is either fixed by tightening the draft to the golden or
  recorded as a new finding — zero silent widenings (diff of schema vs the
  findings log must account for every `anyOf`/nullable).

### Step 1.1b — Solver data-path schemas: `alloc`, `records`, `bom`, `pinmap`
- **repo:** wyred-contract (working tree only) · **~LOC:** 330 · **model:** opus-4.8
- Extract from goldens + `wyred/src/wyred/datapaths.py` + `emit.py` (the
  producers; there is no prose — F1). Factor `stamp`, allocation record,
  connector-pinout row, bound-part row, wiring row into `common.defs.json`
  (extending 1.1a's file). Kind specifics: alloc wrapper's eight fixed keys;
  bom `line_items` with conditional `derived` (present iff
  `authored: false` — encode with `if/then` or two-branch `oneOf`, whichever
  the Step-1.2 keyword whitelist ends up carrying); pinmap `allocations`
  rows (entry + `net` + dotted `node`, F9) and components with optional
  `l1_role` XOR `for_demand` provenance and nullable terminal `net`; records'
  `pool_spares` (map pool → int array), `resolutions` (map demand-id →
  `"rail:...\|bus:...\|role:..."` string, pattern-constrained), rich
  escalation objects (F7), optional `forked_from` object (F4).
- **Acceptance (mechanical):** oracle-validates all 64 files of these four
  kinds; same 5-mutant spot check per kind; `path` field is `const` per kind
  (`"bom"`/`"pinmap"`/`"records"`); findings log updated for anything new.

### Step 1.1c — Lock/lifecycle-gate schemas: `baseline`, `connlock`, `pinmapdiff`, `lifecycle`
- **repo:** wyred-contract (working tree only) · **~LOC:** 250 · **model:** opus-4.8
- Extract from goldens + `harness/allocation.py::snapshot_locks` +
  `wyred/emit.py` lifecycle/connlock paths + `paths.diff_pinmaps`.
  Baseline: four keys incl. the engine-added `connector_pinout` (F6);
  `groups` as a map name → `{version, covers, entries}`. Connlock: optional
  `fork_vs_parent_codes`; code arrays constrained to the observed/gate code
  vocabulary (`CONNECTOR_LOCK_VIOLATION`, `CONNECTOR_SERIES_UNJUSTIFIED`)
  only if the producer code proves the vocabulary is closed — otherwise open
  strings with the known codes listed in the description (decide from
  `paths.check_connector_locks`; record the decision in the findings log).
  Pinmapdiff: nullable `a`/`b` terminal sides, `stamp_a`/`stamp_b`,
  incumbents string. Lifecycle: string `forked_from` (F4), the two
  tamper-code arrays, `ok` boolean.
- **Acceptance (mechanical):** oracle-validates all 9 files of these four
  kinds; mutant spot check; with 1.1a/1.1b this completes ten
  `<kind>.schema.json` files + `common.defs.json` under
  `wyred-contract/schemas/`, every golden covered by exactly one schema
  (filename suffix ↔ schema kind mapping is total).

### Step 1.2 — Dependency-free validator CLI: `wyred-contract/tools/validate.py`
- **repo:** wyred-contract (working tree only) · **~LOC:** 300 · **model:** opus-4.8
- **Honest evaluation of the options.** (a) Depending on the PyPI
  `jsonschema` package: correct and complete, but violates wyred-audit's
  pure-stdlib fence downstream and adds the contract repo's first dependency
  — rejected. (b) Vendoring `jsonschema`: thousands of LOC plus the
  `referencing` machinery; unauditable inside a repo whose CLAUDE.md allows
  only "schema validation helpers" — rejected. (c) **Recommended: a minimal
  purpose-built validator for the exact draft-2020-12 subset the ten schemas
  use**, with a hard **fail-closed rule**: on encountering any schema keyword
  outside its whitelist it errors out rather than silently ignoring it (the
  classic silent-pass failure mode of subset validators). This couples the
  schema-author vocabulary to the validator capability mechanically, and the
  documents remain fully standard draft 2020-12 — external consumers can
  still use ajv or python-jsonschema on them unchanged.
- Keyword whitelist (fixed by what 1.1a–c actually needed; trim to fit):
  `type, enum, const, required, properties, additionalProperties,
  patternProperties, items, prefixItems, minItems, maxItems, uniqueItems,
  pattern, minimum, anyOf, oneOf, if/then/else (or eliminate in 1.1b in
  favor of oneOf), $ref (relative-file + `#/$defs/...` only), $defs,
  description/$comment/$schema/$id/x-wyred-contract (annotation no-ops)`.
- CLI: `python3 tools/validate.py [--schemas schemas/] --tree <dir>`
  (default tree `goldens/ga019`), maps `*.<kind>.json` → `<kind>.schema.json`,
  prints one line per failure (file, JSON-pointer path, keyword, message),
  summary line `N/107 valid`, exit 0 iff all valid, 2 on setup error —
  mirroring wyred-audit's exit convention. Also `--file` for single-artifact
  use and `--canonical` opt-in (byte-compare against
  `json.dumps(json.load(f), indent=2, sort_keys=True)` — the goldens'
  serialization convention, kept out of the schemas themselves).
- **Acceptance (mechanical):** (i) self-test: a schema containing an
  unlisted keyword makes the validator exit 2 naming the keyword; (ii) over
  all 107 goldens the mini-validator's verdicts agree file-for-file with the
  dev-only `jsonschema` oracle run; (iii) the 1.1a–c mutant spot-check files
  all fail with pointers to the mutated location; (iv) `--canonical` passes
  on all 107 goldens.

### Step 1.3 — Full validation run + findings closure
- **repo:** wyred-contract (working tree only) · **~LOC:** 100 (thin test wrapper + findings/doc text) · **model:** opus-4.8
- Run `tools/validate.py` over all 107 goldens; write
  `wyred-contract/tests/run_schema_tests.py` (stdlib, subprocess-invokes the
  CLI; pristine goldens → exit 0; a small committed set of mutant fixtures
  under `wyred-contract/tests/fixtures/` → each fails naming its pointer) —
  mirroring wyred-audit's fixture-test convention. Never hand-edit goldens
  to make this pass (invariant 1).
- **Acceptance (mechanical): 100% of the 107 goldens pass, OR every failure
  is documented as a prose/golden/harness discrepancy finding** appended to
  this document's Findings section (F10+) with the schema decision it
  forced. `tests/run_schema_tests.py` exits 0 on the pristine tree and 1 on
  each mutant fixture. At this point the Phase-1 staged proposal is complete:
  `schemas/` (11 files), `tools/validate.py`, `tests/`, findings — presented
  for ratification as one unit.

## Phase 2 — Integration (milestone: schema validation is a live gate in the audit, and the prose contract points at its schemas)

Phase 2 may start after Phase 1's outputs exist in the working tree; steps
2.1 and 2.2 are independent of each other. Note: 2.1 lands in
**wyred-audit** (normal single-repo change, commit-ready on ratification of
the contract proposal it depends on); 2.2 extends the **wyred-contract**
staged proposal.

### Step 2.1 — Optional schema-validation stage in wyred-audit
- **repo:** wyred-audit · **~LOC:** 150 · **model:** opus-4.8
- **Composition decision (justified against the fences):** wyred-audit's
  fence bars importing **the engine** (`wyred`) — the producer under audit —
  and composes engine runs as subprocesses; but it explicitly *depends on*
  wyred-contract + wyred-harness, and consumes the harness by sys.path
  library import (`harnesslib.py`). The contract's validator is shared
  contract infrastructure, not the producer, so the harness precedent
  applies: **import as a library** via a `contractlib.py` locator
  (`--contract-src` / `$WYRED_CONTRACT_SRC` / sibling `wyred-contract/`
  checkout), not a subprocess. Subprocess would also satisfy the fences but
  adds a process boundary for no independence gain, and the validator is
  needed per-file (107 spawns vs one import). Keep the CLI subprocess-usable
  regardless (it already is, per 1.2) for the future Rust-rewrite path noted
  in wyred-audit's CLAUDE.md.
- Implementation: `src/wyred_audit/schema_stage.py` — locate contract
  checkout, load `tools/validate.py` as a module, validate every recognized
  `*.json` in `--tree` against its kind schema; findings surface as audit
  code `SCHEMA_INVALID` (one per file, with pointer detail). Wire into
  `cli.py` as a stage that **warns and skips** when no contract checkout or
  `schemas/` is found (consumer-form audits with artifacts only must keep
  working — same degradation pattern as the corpus-less rebuild skip). Add
  tamper fixtures via `tests/make_fixtures.py` (e.g. a bom line item missing
  `qty`, a pinmap terminal net as number) and cases in
  `tests/run_fixture_tests.py`.
- **Acceptance (mechanical):** `python3 -m wyred_audit --tree
  <ga019-goldens> --contract-src <wyred-contract>` exits 0 with the stage
  reporting 107/107; each new tamper fixture makes it exit 1 firing
  `SCHEMA_INVALID`; running without a contract checkout prints the skip
  warning and leaves the exit code unchanged from today's behavior;
  `run_fixture_tests.py` green.

### Step 2.2 — EMIT_CONTRACT.md cross-reference pass + version-stamp convention
- **repo:** wyred-contract (working tree only; extends the staged proposal) · **~LOC:** 50 (doc work) · **model:** opus-4.8
- Cross-reference: Part A gains a one-line pointer to
  `schemas/l2.schema.json`; Part B → `schemas/l1.schema.json` (+
  `common.defs.json`); Part C → the allocation `$defs` and
  `schemas/alloc.schema.json` (noting the F1 wrapper distinction). Add a
  short **Part E — emitted data-path & lifecycle artifacts** that names the
  seven prose-less kinds, one sentence each, and defers their normative
  shape to their schema files (schemas as descriptive appendix — this is
  the F1 closure, not a re-specification).
- Version stamping: fix the convention as — each schema carries
  `x-wyred-contract: {"contract": "<EMIT_CONTRACT version>", "schema_rev":
  <int>}`; `$id` path embeds `/v0/` and bumps only on breaking schema
  changes; any future contract-version bump updates every schema's
  `contract` field in the same proposal. Create the **changelog section** at
  the bottom of EMIT_CONTRACT.md (F8) with its first entry: the schema
  extraction itself, listing findings F1–F9 (and any F10+ from step 1.3) as
  ratified prose-vs-artifact clarifications.
- **Acceptance (mechanical):** every schema file's `x-wyred-contract` stamp
  is present and consistent (`tools/validate.py --selfcheck` or a
  10-line addition to `run_schema_tests.py` asserts it); every Part A/B/C
  section contains its schema link; the changelog section exists with the
  extraction entry; `git diff` in wyred-contract touches only
  EMIT_CONTRACT.md, `schemas/`, `tools/`, `tests/` — goldens byte-identical
  (`git status goldens/` clean).

## Out of scope (explicitly)

- Regenerating or "fixing" any golden (a contract event, not this plan).
- Enforcing the schemas inside wyred-harness or the wyred engine (the
  harness's deliberately tolerant `from_json` stays tolerant; over-strict
  schemas must not break candidates — the schemas gate *artifacts*, not
  candidate code).
- Committing anything in wyred-contract: ratification (commit + version
  bump + DecisionLog entry) is Gianni's, per single-writer.
