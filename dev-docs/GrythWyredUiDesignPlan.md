# GrythWyredUiDesignPlan — wyred review surfaces as a gryth-ui plugin family

*Status: **design + phased plan** for the ratified direction. The stack
decision is made and recorded in
[WyredUiUserJourneys.md](./WyredUiUserJourneys.md) (Gianni, 2026-07-13):
wyred-ui is built on gryth-ui — "wyred surfaces as gryth plugins over a
shared design document." This document is the rework that journeys §8 calls
for: it says HOW, against the surveyed reality of both trees (parallel code
surveys of `gryth-wz` @ gryth-ui `bffbdd0`, GLP-0006 P1.S4, and `wyred-wz`,
run 2026-07-12). The journeys (J1–J6) are the requirements source; §5.0
maps them to phases.*

*Relationship to prior docs: **[WyredUiDesign.md](./WyredUiDesign.md)
remains authoritative for surface content (§4), trust posture (§2), and the
goldens testing story (§6)**; its §7 plain-React stack section is superseded
(journeys §8), and its §3.1 ArtifactStore seam survives exactly as the seam
it was designed to be. **[WyredPlanUi.md](./WyredPlanUi.md) is superseded —
do not build from it** (journeys §8); its acceptance criteria are carried
forward verbatim below.*

---

## 1. Shape of the answer

**wyred-ui becomes the first external gryth-ui plugin family.** Three facts
from the surveys make this cheap now:

1. **wyred-ui has zero code** (fence docs + one bootstrap commit). Nothing
   migrates; we scaffold on the decided foundation directly.
2. **The gryth plugin contract is real, small, and tested.** Everything a
   plugin touches is `@grythjs/plugin-api` (~265 LOC;
   `gryth-wz/gryth-ui/packages/plugin-api/src/registry.ts`), proven by
   seven in-tree plugins (settings: 148 LOC total). `ToolId` is already an
   open string (`registry.ts:13`) — the "first external plugin" blocker in
   `PluginMigration.md` Phase 4 is already gone.
3. **Collaboration is no longer hypothetical.** gryth-ui HEAD has live,
   converging glade surfaces (chat groups, gwz ops) through glial mounts
   (`packages/glade/`, `chat/live.ts`, `gwz/live.ts`). The doc-scope
   design document the journeys assume has a working in-app mechanism to
   grow into — while everything below Phase 4 runs with **no glade, no
   node, no server**, exactly as gryth-ui itself does in mock mode.

**What the journeys change vs. a plain "hosted viewer":** the artifact tree
becomes (eventually) doc-scope shared state; views must update **live**
during the inner design loop (J1); boards come in **fleets** (J3 — DUT +
test harness as one design set); reviewers and **capability-scoped agents**
drop entity-anchored **annotations** (J1/J2); and two journeys author new
data — bench measurements (J4) and tracked share flows (J5). None of that
weakens the trust fence; §2 re-scopes it precisely so it holds with agents
in the loop.

**What it does not cost:** phases 0–3 depend only on `plugin-api` +
`desktop` + `@owebeeone/grip-{core,react}` as commit-pinned local checkouts
(§3.1) — wyred stays off gryth's critical path and vice versa. Phase 4+
(shared doc, agents) rides seams gryth is building anyway, and every phase
before it ships standalone value.

## 2. Invariants — the fence, re-scoped for journeys and hosts

- **Emitted artifacts are read-only, everywhere, forever.** The UI never
  mutates, re-derives, recomputes, or "fixes up" anything the toolchain
  emitted. Absence renders as absence, loudly. Schema validation stays the
  one permitted computation on emitted bytes (checking, not deriving);
  the entity index stays an index, not an inference; engine-self-reported
  verdicts keep their provenance labels ("engine-reported; audit
  counter-checks this").
- **The UI may AUTHOR, never ALTER.** The journeys add two authored
  categories, both new records with their own provenance — never edits to
  emitted files: (a) **collaboration data** (annotations, findings,
  stances, focus, chat) living in gryth doc surfaces *beside* the tree
  (J1); (b) **contract-shaped input records** (bench measurement records,
  J4) written as new files/records the *gate* consumes — the UI writes
  them where inputs live, stamped (author, testplan stamp, serial), and
  the engine's stale-stamp refusal stays the enforcement point. The
  viewer surfaces themselves remain pure renderers.
- **Advisory, never a gate — and never a runner.** emit/gate/audit remain
  CLIs (J1 says so explicitly). gryth's gwz plugin shows how easy grazel
  *exchanges* make "run a CLI from a window"
  (`gwz/live.ts` — `client.exchange('ws-razel','gwz.ops',…)`); the wyred
  family **refuses that affordance permanently**. If an ops panel is ever
  wanted it is a separate design with its own fence debate. (J5's
  connector "share flows" transmit ratified, hash-pinned assets — they
  transform nothing; they are a later, separately-fenced plugin family,
  §3.8.)
- **Depends on `wyred-contract` schemas + goldens only — never calls the
  engine or harness.** Host libraries (grip, plugin-api, desktop) are UI
  chrome, not data sources; displayed values still come only from bytes
  in artifact files. `wyred-ui/CLAUDE.md` gets one line naming the host
  deps (Phase 0.1).
- **The stronger-for-free pair:** gryth-ui bans React local state
  (`scripts/no-react-state.test.mjs`; CodingRules.md) — all state is
  grips. "Every displayed value traces to a byte in an artifact file"
  becomes mechanically auditable: value ← grip ← tap ← ArtifactStore ←
  file bytes, no `useState` side-channel for derived values to hide in.
  This matters MORE with agents in the loop (journeys §8) — an agent
  reads the same grips a human sees.
- **Goldens are the fixture corpus** (read-only; ga019 — 19 stems, 115
  files; counts read from the tree, never hardcoded, per WyredPlanIndex
  rule 4). pnpm, never npm. No new runtime deps beyond React + grip
  without a recorded reason.

## 3. Design

### 3.1 Where the code lives, and how the dependency works

The wyred plugin family lives in the **`wyred-ui` repo** (already a gwz
member, `owebeeone/wyred-ui`), which becomes a pnpm workspace:

```
wyred-ui/
  packages/artifacts/     @wyredjs/artifacts      — pure domain library:
                          Kind/stem types, census parser, design-set model,
                          ArtifactStore interface + DirectoryStore +
                          StaticStore (+ watch/poll), the validator port,
                          entity-URI + index builders.
                          ZERO deps (not even grip). Node-testable.
  packages/plugin-wyred/  @wyredjs/plugin-wyred   — the gryth plugin: grips,
                          taps, window tools, annotation channel, css.
                          Depends on plugin-api, grip-react (peer),
                          @wyredjs/artifacts.
  app/                    wyred-ui (private)      — the standalone host app
                          (§3.6): composition root + set-picker boot state.
                          Depends on @grythjs/desktop + the plugin.
```

(`@wyredjs` npm scope is provisional — open question O5. One plugin
package, not one per view: gryth's grouping rule is *one package per
provider seam*, and every wyred view flips against the same artifact
store.)

**Dependency mechanics — the `@grythjs/glade` precedent.** gryth-ui already
consumes packages from a *sibling gwz workspace* via `file:` deps
(`packages/glade/package.json` → `file:../../../../glade-wz/glial` etc.).
wyred-ui does the same in the opposite direction:

- `wyred-wz` adds **`gryth-ui`, `grip-core`, `grip-react` as gwz members**
  (the exact move `gryth-wz` made for `glade-decl-ts`), commit-pinned in
  `gwz.conf/gwz.lock.yml` — wyred controls when to roll the pin forward.
- **`plugin-wyred` declares `@grythjs/plugin-api`, `@owebeeone/grip-react`,
  and `react` as peerDependencies** (with `file:../../../gryth-ui/…`
  devDependencies for wyred-local dev/tests); the app additionally takes
  `@grythjs/desktop`. Peer-not-direct is the singleton rule made
  structural: whichever host bundles the plugin resolves those peers to
  *its own* copies — one `GripRegistry`, one grip-react, one React per
  running app, in both hosts. (The gryth packages are source-exporting —
  `exports: "./src/index.ts"`, no build step — which vite already handles
  inside gryth-ui itself.)
- **Two singletons are correctness constraints, not preferences** (from
  gryth-ui's own config): (a) one grip runtime — every wyred grip is
  minted via `defineGrip` **imported from `@grythjs/plugin-api`** (its
  `runtime.ts` singleton), never from grip-core directly; (b) one React
  and one grip-react copy — wyred-ui's workspace replicates gryth-ui's
  pnpm `overrides` + vite `dedupe`/`optimizeDeps.exclude` settings.
  Phase 0.1's spike exists to get exactly this right once.

Contract-churn containment: `@grythjs/plugin-api` is 0.1.0 and the contract
doc is "draft v2", so churn is expected. It is absorbed at two small
surfaces — the plugin's `index.ts` registration (settings' equivalent is 22
lines) and the host app's bootstrap (~25 lines) — because everything else
in `plugin-wyred` talks to grips, and `@wyredjs/artifacts` doesn't even
know grip exists.

### 3.2 The data layer — design sets, ArtifactStore kept, grips over it

**The design set is the workspace unit (J3), not the single tree.** A
design set is a named grouping of artifact trees — the DUT board(s) and
the pogo-pin test-harness board are separate wyred designs reviewed
together. Stems address boards *within* the set (stems are design names;
the entity-URI sketch `wyred://<design-set>/<stem>/<kind>#…` carries no
tree segment — journeys §7); the set resolves stem → tree, and a stem-name
collision across trees renders as a loud census error, never a silent
pick. A single-tree set is the degenerate case with zero UX overhead —
v1 opens one tree and gets a one-tree set implicitly; "add a tree to the
set" is an affordance, not a mode. (An engine-level "system/fleet" notion
is explicitly future — the set is a UI-level grouping for now, per J3.)

The ArtifactStore interface ships **verbatim from WyredUiDesign §3.1** in
`@wyredjs/artifacts` (census / load / raw / `watch?`), one store instance
per tree in the set. Per journeys §8: **DirectoryStore + StaticStore are
the dev/standalone modes; the glade-backed doc-scope store is the target
collaborative mode behind the same seam** (§3.7, Phase 5). What changes
from the old plan either way: **no view ever sees a store.** One tap layer
adapts stores to grips — "everything above the store is provider-agnostic"
now means *everything above the store is the context graph*.

**`watch` is promoted from optional to a v1 requirement** — J1's inner
loop is "emit runs → every open plugin view updates live." Dev-mode
implementations are honest about their transports: DirectoryStore
re-censuses on a poll tick + window-focus (FS Access API has no native
watch); StaticStore polls the census source (manifest etag when present).
The glade-backed store gets push for free later. Views never know which;
they see grip updates.

Grip inventory (all minted through the shared runtime; scope + tap-class
annotations follow gryth CodingRules):

| grip | class/scope | value |
|---|---|---|
| `Wyred.Set` (+`.Tap`) | 1 atom; environ now, doc-promotable | `{name?, trees: TreeRef[]}` where `TreeRef = {kind:'static', baseUrl} \| {kind:'directory', name}` — directory *handles* are runtime-only tap state (they don't serialize); only all-static sets are shareable/roamable, and directory-backed trees carry a "local only" badge from day one |
| `Wyred.Set.Status` (+`Reload` handle) | 2 status | per-tree `idle/loading/ready/error` + watch liveness; manual reload stays as the escape hatch |
| `Wyred.Census` | 2 source | unioned `ArtifactSet[]` across the set — stems × kinds present, per-file validity, per-stem source tree; the two L1-only goldens stems (`intent_03_addr_collision`, `intent_07_voltage_mismatch`) appear as the honest partial rows they are; cross-tree stem collisions flagged here |
| `Wyred.Art.L1` … `Wyred.Art.Records` | 2 source, per-destination | one typed grip per contract kind (11: l1, l2, alloc, bom, pinmap, pinmapdiff, records, baseline, connlock, lifecycle, cir), each resolving against the destination context's stem: `{status:'absent'\|'loading'\|'ok'\|'invalid', json?, errors?}` — absent is a rendered state, never a fabricated default |
| `Wyred.Art.Raw` | 2 source, per-destination (stem+kind) | exact bytes for the raw/hex view and the SCHEMA_INVALID fallback |
| `Wyred.Focus` (+`.Tap`) | 1 atom, workspace-level | the focused **entity URI** (journeys §7 shape, blessed in Phase 0.3) — the pivot target; **this is the grip that later gets a share** (§3.7) |
| `Wyred.Index` | 3 conversion | entity URI → home-view locations, built from *loaded* artifacts only |
| `Wyred.Annotations` (+handles) | 1 atom, **doc-scope-declared** | entity-anchored annotation records (§3.5) — local-first now, glade-mounted later, same grip |
| per-tab (via `tabTaps`) | 1 atom, instance | `Wyred.Tab.Stem` seeded from link params, plus each view's filters/selection/sort |

The **WyredStoreTap** (registered once at the workspace level) owns the
per-tree `ArtifactStore` instances (selected by each `TreeRef.kind`), runs
the watch/poll loop, and produces census + per-kind artifacts per
destination (the `destinationParamGrips` pattern the terminal contract
example uses for `Tool.SourceRef`). Loaded artifacts are cached tap-side
(class-2 cache; ga019 is 115 small files — eviction deferred, recorded).

**Deliberately not a matcher flip (yet).** gryth's Phase-3
`withOneOf(PROVIDER_GRIP,…)` matcher bindings are *unbuilt anywhere* in
gryth-ui today (survey: zero `addBinding` call sites; mocks bind
directly). Directory-vs-static is not a mock/real flip — it's one real
provider with two byte transports, so an internal switch on `TreeRef.kind`
is honest and avoids depending on machinery that doesn't exist. The
matcher flip is reserved for the **glade-backed store** (Phase 5) — at
which point wyred becomes the first real user of gryth's mock↔real
selector exactly as intended, with zero view changes.

**Validation** is a pure function in `@wyredjs/artifacts`: the TS port of
`wyred-contract/tools/validate.py` semantics — same keyword whitelist
(`type enum const required properties additionalProperties items
prefixItems minItems maxItems minimum pattern oneOf` + `$ref`/`$defs` +
annotation no-ops), **fail-closed on any unlisted keyword**, the two
`$ref` styles only. It runs inside WyredStoreTap on every load; invalid ⇒
the artifact grip carries `status:'invalid'` + errors, views banner and
offer the raw view, never a pretty rendering of corrupt data. The
contract's 12 committed mutant fixtures are the port's test corpus.

### 3.3 The plugin — one identity grip, the tool catalog

Registration is the settings-plugin shape (`plugins/settings/src/index.ts`):
`WYRED_PLUGIN = defineGrip<GrythPlugin>('Wyred.Plugin')`; importing the
module calls `addEntry(WYRED_PLUGIN, {tools})` and registers the root
taps. Tool ids are namespaced `wyred.*` (ToolId is an open string; we're
the first external family — don't squat bare names). File conventions
follow the house pattern: `index.ts` / `grips.ts` / `taps.ts` /
`views/*.tsx` / css / `ops.ts` + tests.

**Core tools (phases 1–4)** — content specs live in WyredUiDesign §4:

| tool id | role | window | link params |
|---|---|---|---|
| `wyred.census` | explorer | census matrix over the design set; tree/set picker affordance; validity coloring; the home surface | — |
| `wyred.netlist` | stage | netlist explorer (l2): provenance chips, terminal tables, filters | `{stem, focus?}` |
| `wyred.pinmap` | stage | pin-map/firmware view (pinmap+alloc): state + lock badges, series/lock-version stamps prominent (J2: "which pin-map am I coding against" never ambiguous) | `{stem, focus?}` |
| `wyred.bom` | stage | BOM with derived-explanation strings first-class; renders the `sourcing` slot verbatim if/when the contract gains it (J6 lynch-pin 1 — rendering costs nothing once emitted) | `{stem, focus?}` |
| `wyred.locks` | stage | locks & lineage (records/baseline/connlock/lifecycle): series tree, ICD tables, engine-reported/audit-checked labels | `{stem, focus?}` |
| `wyred.eco` | stage | ECO diff — renders the emitted `pinmapdiff` artifact only; the payload view for J2's change notifications | `{stem}` |
| `wyred.intent` | stage | intent view (l1): roles, rails, buses, pools, escalations | `{stem, focus?}` |
| `wyred.raw` | stage | raw bytes + validation errors; the SCHEMA_INVALID landing | `{stem, kind}` |
| `wyred.annotations` | crew | the finding/annotation browser (J1): open/resolved, by entity, by author (PrincipalRef), filterable; every record links to its anchor | `{filter?}` |

**Journey tools (deferred until their inputs exist — §5 Deferred):**
`wyred.coverage` (J3; blocked on the coverage *artifact* contract proposal
— coverage is emitted, never UI-derived), `wyred.bench` (J4
measurement-entry family; blocked on the ratified measurement-record
contract + the §2 authored-records fence mechanics), `wyred.release` +
connector plugins (J5; blocked on the release-bundle manifest proposal).
Naming them now keeps the catalog honest; building them is separate work.

Per-tab state rides `tabTaps(tabId, params)`: the stem atom seeds from the
opening link (so tabs rehydrate after reload and `Desktop.RetargetTab` can
re-point a window at another stem), filters/selection are per-window
atoms. `menuTitle` renders live counts from doc grips before any window
exists (census: "N stems · M partial"; annotations: open-finding count).
Multiple windows of one tool are free (per-tab contexts) — DUT pinmap next
to harness pinmap (J3) is not a feature, it's the shell.

### 3.4 Cross-linking = entity URIs + links, wires, pins

**Entity addressing is a Phase 0 deliverable** (journeys §7: "its own
short proposal before plugin work starts"). The proposal (0.3) blesses the
URI shape — sketch `wyred://<design-set>/<stem>/<kind>#<json-pointer-or-
entity-id>` — and the wyred-ui data model adopts it everywhere an entity
is named: `Wyred.Focus` holds a URI, `Wyred.Index` maps URIs to locations,
annotations anchor on URIs, chat links carry them. The entity ids inside
the fragment already exist and are stable (law-8 minting); the proposal is
the envelope, including how set-relative URIs resolve when the set is
local-only.

The interaction model — "every refdes, net, demand, role, lock group, and
code is a link; clicking pivots to that entity's home view" — maps 1:1
onto shell primitives that already exist (`registry.ts`):

- **Pivot** = `DESKTOP_OPEN_TOOL` with `{toolId, params:{stem, focus}}` —
  snapshot semantics, always a new window (shell policy v1).
- **Live follow** = `DESKTOP_OPEN_WIRED` / `OPEN_WIRED_PAIR`: a detail
  view opened as a *sink* wired to the census/netlist window's context
  reads the source's published selection live — the code plugin's
  explorer→viewer WTA pattern, applied to entities. One sink per
  (source, tool): re-clicking reuses the wire (the IDE "current editor"
  behavior).
- **Freeze** = `DESKTOP_PIN_TAB`: snapshot the wired view's params and cut
  the wire — "keep this ECO diff open while I compare three others" is
  the preview-vs-pinned split the shell already implements.
- **Cross-window focus** = writing `Wyred.Focus`; open wyred windows opt
  into following it (peer coupling via shared grip, per the contract's
  master/detail rule — never context parentage). This is also J1's
  "show me what you're looking at": following a *person* (or agent) is
  following the shared focus grip once it's promoted (§3.7).
- **Links travel as data.** A `ToolLink` is serializable by design
  ("links travel anywhere data does: chat messages, activity feeds,
  agent plans" — `registry.ts:29-31`). An annotation or chat message
  about C7 carries `{toolId:'wyred.bom', params:{stem,
  focus:'wyred://…#C7'}}`. That is the J1/J2 collaboration story and the
  agent story with **zero additional mechanism**.
- **Cross-board links (J3)** are ordinary entity links: the set resolves
  the stem, so "DUT testpoint ↔ harness pogo" pivots are just two URIs —
  the *join* between them comes from the future coverage artifact, never
  UI inference.

The entity index that powers pivots is built from loaded artifacts only;
an entity with no loaded home renders a dead chip with "not loaded /
absent", never a guessed link (absence loud, as always).

### 3.5 Annotations — the J1 channel, local-first

The annotation record (shape finalized alongside 0.3; collaboration data,
not a contract artifact):

```
{ id, anchor: <entity-URI>, author: PrincipalRef, body: text,
  links?: ToolLink[], status: 'open' | 'resolved',
  resolution?: { kind: 'intent-edit' | 'stance' | 'wontfix', note },
  created, updated }
```

- **Anchored, attributed, resolvable.** J1's loop: findings anchor to
  entities; each resolves into "back to the IDE" (intent-edit) or a
  recorded stance (DecisionLog culture, in-UI — the resolution note is
  exportable text for the log; the UI does not write DecisionLog.md).
- **Local-first, share-ready.** `Wyred.Annotations` is a doc-scope-
  *declared* atom that starts life local (exactly how gryth-ui treats
  `Doc.WorkspaceName` today — declared intent, mock persistence). Phase 5
  mounts it on glade (an append-shaped commons surface) without touching
  a view. Agents write it through the same handle grips humans do —
  capability scoping is gryth's enforcement work, attribution is carried
  from day one via `PrincipalRef` (stubbed identity labeled as such).
- **Rendered in place.** Views show anchor chips (count badges per
  entity, from a conversion over annotations × index); the
  `wyred.annotations` tool is the browse/triage surface. J2's "change
  notification whose payload IS the pinmapdiff" rides the same channel: a
  watch event on a subscribed series posts an annotation (author:
  `wyred-ui watcher`) linking to `wyred.eco` — no bespoke notification
  system invented; when gryth grows one, the record is already data.

### 3.5b Two hosts, one plugin body

The plugin body is host-agnostic (it talks to grips and `plugin-api`
surfaces only), so it runs in two compositions — in this order:

1. **The gryth-ui test mount (Phase 0.2, the first dev vehicle).**
   gryth-ui's composition root already auto-discovers app-local plugins
   (`src/plugins/index.ts` globs `./*/index.ts` eagerly — "adding a plugin
   is adding a directory", PluginMigration). The mount is one shim dir in
   `gryth-wz/gryth-ui/src/plugins/wyred/` (`import '@wyredjs/plugin-wyred'`)
   plus one `file:../../wyred-wz/wyred-ui/packages/plugin-wyred` dep — the
   `@grythjs/glade` → `glade-wz` sibling-workspace pattern, in reverse.
   Fully reversible (delete the dir + the dep line); whether it stays
   permanently, goes behind a build flag, or comes and goes is Gianni's
   call on the gryth side (3.4). Payoff: wyred windows develop **next to
   the live chat/gwz panels** from day 0 — the collaborative surround is
   real while the wyred data layer is still local — and no day-0 bet on
   `@grythjs/desktop` standalone hostability.
2. **The standalone wyred host (Phase 1.5).** `wyred-ui/app` composes
   `@grythjs/desktop` with only the wyred plugin: a ~25-line bootstrap
   mirroring `gryth-ui/src/bootstrap.tsx` minus `startGlade()`. This is
   the journeys-§8 "dev/standalone mode" as a product: `pnpm dev` against
   any out/ directory, offline, no server, no glade. The census window's
   empty-state set picker doubles as the pre-desktop boot surface
   (gryth's boot-flow contract explicitly allows primary UI full-bleed or
   in-chrome — same component both ways).

Because both checkouts of gryth-ui coexist (wyred-wz member pinned for
hermetic wyred dev; gryth-wz HEAD for the test mount), each side rolls its
pin deliberately; the peer-dependency rule (§3.1) is what keeps either
host on a single grip registry.

### 3.6 Scope map

Following gryth's doc/environ/instance model (grips declared with intended
scope per CodingRules):

- **doc** (the shared design document, when sharing exists):
  `Wyred.Set` (all-static form), `Wyred.Focus` once promoted,
  `Wyred.Annotations`, review chat. Until glade wiring lands these are
  ordinary local atoms — *declared* doc-intent, exactly how gryth-ui
  treats `Doc.WorkspaceName` today.
- **environ** (the user's desktop): which wyred windows are open, where,
  on which stems — the desktop document, owned by the shell; wyred
  contributes only serializable link params. (Desktop persistence itself
  is gryth work-in-flight — GrithReview-55 — and none of wyred's
  concern.)
- **instance** (this client, never replicated): per-window filters,
  selections, sort orders, hover, directory handles, watch/poll timers.

### 3.7 Collaboration and agents — the target, honestly gated

The substrate today (survey + `GrythGladeSeamAssessment.md`): glial mounts
are the canonical seam and chat/gwz panels converge live in-app; but
identity is stubbed (per-tab origins, `?principal=`), the client store is
in-memory, grants are **unenforced**, and environ replication is unbuilt.
The journeys' end-state — Aya connecting with a *scoped session
capability* (read artifacts, write annotations) — is exactly gryth's C3
roadmap; wyred's obligation is to be ready, not to build it:

- **Ready = everything is a grip.** Every surface a human uses (focus,
  annotations, links, tool-opening) is a grip an agent can hold — the
  gryth seam tests already prove agent-drives-UI through handles. When
  capability enforcement lands glade-side, wyred inherits it; until then
  agents are honest-by-convention like every other gryth participant.
- **The shared design document arrives in two steps.**
  **Phase 4 (local-first collaboration):** annotations + focus + set as
  declared doc-scope grips, links-in-chat — full J1 review flow on one
  desktop, agent-drivable, zero glade.
  **Phase 5 (the shared doc):** mount those grips on glade (commons
  value/log surfaces via `glialTap` — `chat/live.ts` is the in-tree
  template) and, behind the matcher flip, the **glade-backed
  ArtifactStore**: the doc scope holds the artifact tree; Directory/
  Static demote to dev/standalone modes (journeys §8). Transport shape
  inside that store — bytes carried as ops vs. doc-carried
  pointers+hashes with client fetch (the class-2 p2p cache story
  GrythVision defers) — is decided at 5.1 with glade-side input; both
  fit behind the seam, and hash-pinning ("same bytes", checking not
  deriving) is required either way.
- **Gating:** Phase 4 has no external dependencies. Phase 5 starts when
  gryth's seam offers commons surfaces an external plugin can declare
  without bespoke glade work (the demo-proposal Slice-0/3 shapes; chat
  already demonstrates the mount pattern). wyred does not push on that
  timeline — phases 0–4 are complete, shippable value without it, which
  is what keeps each project off the other's critical path.

### 3.8 Explicitly refused / deliberately later

No authoring of intent (the IDE owns it). No running (§2 — including via
grazel exchanges). No live board data in v1 (J4's instrument streaming is
future; typed entry first). No schematic geometry (KiCad owns it). No
derivation — the ECO view renders the emitted `pinmapdiff` only; coverage
renders the emitted coverage artifact only, when it exists. No editing of
emitted artifacts, ever; authored records per §2 only. No gate wiring,
ever. Graph-visual netlist rendering stays deferred. J5 connectors
(external transmission of ratified assets) are a **separately-fenced
future plugin family** — tracked-share semantics, destination adapters,
and "who sent what where" audit live in that design, not this one. J6
stays a colouring: the sourcing-slot and bundle-as-negotiation-artifact
lynch-pins are *contract* proposals (journeys §6), not UI work; the UI's
whole preparation is rendering slots verbatim and keeping agents
first-class, both already in this design.

## 4. Testing story

- **`@wyredjs/artifacts`** (node, vitest): census/set parser + validator +
  URI/index builders against the live goldens tree (counts read from the
  tree) and the 12 mutant fixtures (must report invalid; the oracle
  relationship to `validate.py` is file-for-file agreement over ga019 +
  fixtures). Watch/poll logic tested with a fake clock.
- **`@wyredjs/plugin-wyred`** (vitest, gryth-house style): views render
  **from grips** — set `Wyred.Art.*` / census / focus / annotations grips
  in a test context, assert rendered content (`expect.poll` on drips
  where async); per-view rendered-output snapshots over the named
  acceptance stems (watchy_v1_reva, mppt_2420_hc, watchy_v1_draft,
  watchy_v1_draft_btn3, intent_05b) as regression anchors;
  SCHEMA_INVALID banner tests over mutants; pivot tests (click →
  `Desktop.OpenTool` invoked with the right link) with a stub shell
  intent grip; annotation round-trip (write via handle → chip count +
  browser row update).
- **gryth-ui's coding rules are adopted wholesale, enforced the same two
  ways**: wyred-ui vendors the `scripts/no-react-state.test.mjs` scanner
  (all seven hooks banned; empty approvals registry to start; same-line
  approval markers only with recorded owner sign-off) into its own `pnpm
  test`, AND the matching eslint `no-restricted-syntax` bans into its
  flat config. The full CodingRules discipline comes with it: every atom
  grip declared with intended scope (doc/environ/instance) in its
  grouping comment; every writable atom paired with a `*.Tap` handle
  grip; gesture handlers read via `handle.get()`, never render closures;
  timers/animation live in taps, components stay pure. This is both
  §2's traceability claim made mechanical and the glade-readiness
  guarantee — glial mounts bind to exactly these scope-declared,
  handle-published atoms (`chat/live.ts`), so Phase 5 is a mount, not a
  rewrite.
- **Host smoke** (app): boots, registers, census opens on the goldens via
  StaticStore in CI (vitest + jsdom or a thin playwright pass — decide in
  0.2; DirectoryStore is manual-only, FS Access API being Chrome-class).

## 5. The plan

*Steps ≤ ~500 LOC aspirational; each names its dependencies. Phases are
shippable milestones. Steps marked **[fable]** shape contracts/seams; all
others are **[opus-4.8]** mechanical builds. All work in `wyred-ui/` (+
`wyred-wz` gwz config + one proposal doc in `dev-docs/`); nothing touches
the engine, harness, contract schemas, goldens, or any gryth-wz member.
No commits without the recorded go (house rule).*

### 5.0 Journey coverage (the requirements check)

| journey | served by | notes |
|---|---|---|
| J1 inner loop | P1 (watch, census) + P2 (views) + P4 (annotations, focus, agent-holdable grips) | full loop minus multi-user sync; that's P5 |
| J2 ICD sync | P2.2 (pinmap + stamps) + P3.1/3.2 (series, ECO) + P4.3 (diff-arrival notification) | agent-posted HAL diffs ride annotations/chat links |
| J3 fleet | design-set model in P1 (architecture) + cross-board links in P2.4 | coverage panel deferred on the coverage-artifact proposal |
| J4 bench | fence re-scope in §2 now | measurement-entry family deferred on ratified measurement record |
| J5 release | — | deferred on bundle-manifest proposal; connectors separately fenced |
| J6 production | BOM renders sourcing slots when the contract gains them | contract lynch-pins, not UI work; agent-readiness is P4's |

### Phase 0 — the seam spike + the URI (milestone: a hello-wyred window inside the real gryth-ui, and the entity address blessed)

- **0.1 Workspace wiring** [fable, ~120 + config]: add `gryth-ui`,
  `grip-core`, `grip-react` as `wyred-wz` gwz members (pinned); scaffold
  the `wyred-ui` pnpm workspace (three packages, §3.1); replicate the
  singleton constraints (pnpm `overrides`, vite `dedupe` +
  `optimizeDeps.exclude`, tsconfig accommodations for cross-workspace TS
  source — gryth-ui's own configs are the template); one-line CLAUDE.md
  fence addendum naming host deps + the §2 authored-records re-scope.
  Acceptance: `pnpm install` + `pnpm build` green in wyred-ui with both
  grip copies deduped (verify via `pnpm why`).
- **0.2 Plugin skeleton + gryth-ui test mount** [fable, ~200 wyred + ~10
  gryth]: `plugin-wyred` registering `Wyred.Plugin` with one stub tool
  via `addEntry` (peerDeps per §3.1); adopt the coding-rules enforcement
  (scanner + eslint bans); then the §3.5b shim — one
  `src/plugins/wyred/` dir + one `file:` dep line in
  `gryth-wz/gryth-ui`. Acceptance: the wyred tool appears in the REAL
  gryth-ui launcher next to chat/gwz; opening it creates a window with a
  per-tab context; wyred-ui's `pnpm test` runs the gate + a registry
  smoke test; gryth-ui's own `pnpm test` stays green with the mount
  present. *De-risks the cross-workspace resolution story (§3.1) for
  ~210 LOC without betting on standalone `@grythjs/desktop` hosting.*
- **0.3 Entity-URI proposal** [fable, doc only]: the short
  contract-adjacent proposal journeys §7 calls for — URI envelope over
  the existing stable ids, design-set segment semantics (stem-unique
  resolution, §3.2), fragment forms per entity type, local-set
  resolution rules. Lands in `dev-docs/` for Gianni's ratification
  (journeys §9 Q4); `Wyred.Focus`/index/annotations adopt it. (Parallel
  with 0.1/0.2; blocks 2.4 and 4.x, not Phase 1.)

### Phase 1 — data layer + census (milestone: point the wyred desktop at the goldens and explore the tree, live)

- **1.1 `@wyredjs/artifacts` core** [~380]: kinds/types (objects, not
  enums — house rule), `ArtifactStore` interface verbatim, StaticStore
  (fetch; census via optional `census.json` manifest → autoindex-HTML
  fallback, O1), census parser + **design-set union** (stem→tree
  resolution, collision detection), node tests over goldens. No grip
  anywhere. (Parallel with 0.2.)
- **1.2 Validator port** [~300]: `validate.py` semantics (§3.2 whitelist,
  fail-closed, two `$ref` styles) loading `wyred-contract/schemas/*`;
  oracle test: agreement with the goldens (all valid) + the 12 mutants
  (all invalid, right codes). (Parallel with 1.3.)
- **1.3 The tap layer** [~420]: `Wyred.Set` atom + WyredStoreTap
  (per-tree store selection, cache, census + 11 per-kind
  destination-param grips + raw + statuses, **watch/poll loop** as a tap
  per house pattern — the tap owns the timer, components stay pure),
  DirectoryStore (handle held tap-side, "local only" badge data).
  Depends 0.2 + 1.1. Tests: grips resolve against a test context over
  goldens-by-StaticStore; poll tick updates census (fake clock).
- **1.4 Census window + raw view + set picker** [~450]: the real
  `wyred.census` matrix over the set (validity coloring, partial rows
  honest, source-tree column when >1 tree), `wyred.raw` (bytes + errors +
  SCHEMA_INVALID banner), the set picker as the census window's
  empty-state (URL entry + directory pick + drag-drop + "add tree" —
  §3.5b: same component serves in-window and full-bleed), light/dark via
  desktop theme grips. Depends 1.2 + 1.3; developed in the gryth-ui test
  mount. Acceptance: `python3 -m http.server` over ga019 → census
  renders every stem/kind in the tree with the two L1-only stems
  partial; touching a file re-renders within one poll tick; drag a
  mutant fixture in → banner, raw fallback, never a pretty view.
- **1.5 Standalone host app** [~250]: `wyred-ui/app` — bootstrap
  (GripProvider + shell taps + `<Desktop/>`), pre-desktop boot state
  rendering the 1.4 picker full-bleed, wyred plugin as the only mounted
  family. Depends 0.2 + 1.4. Acceptance: `pnpm dev` in `wyred-ui/app`
  gives the full census→views flow offline with no gryth-wz checkout
  running and no glade; this is the journeys-§8 standalone mode as a
  product. *This is where the R2 desktop-hostability risk actually gets
  paid — after Phase 1, not before Phase 0.*

### Phase 2 — the working views (milestone: the firmware engineer's desktop — pinmap, netlist, BOM, side by side)

*2.1–2.3 are independent (separate view files) once Phase 1 lands; 2.4
needs 0.3 plus any one view.*

- **2.1 Netlist explorer** [~450]: components/nets linked tables,
  provenance chips (`authored` / `for_demand:<declaration>` — THE product
  story, rendered), terminal tables, filters (per-tab atoms). Acceptance:
  watchy_v1_reva + mppt_2420_hc render stably (snapshot).
- **2.2 Pin-map / firmware view** [~400]: demand → pool/unit → pad → net
  with allocation-state + lock badges; series + lock-version stamps
  prominent (J2); print affordance (O2). Acceptance: watchy_v1_reva shows
  all 16 author-pinned rows badged pinned+locked; watchy_v1_draft shows
  the two solver-sticky INT rows badged solver.
- **2.3 BOM view** [~300]: line items, derived strings first-class,
  authored/generated split totals; renders unknown-to-us extra fields
  (e.g. a future `sourcing` slot) in a raw-faithful details drawer rather
  than dropping them. Acceptance: mppt_2420_hc's 140 line items with
  derivation strings ("2*(CL−Cstray) = 20pF" class) rendered.
- **2.4 Entity index + cross-links** [~380]: `Wyred.Index` over entity
  URIs (0.3), `Wyred.Focus`, link chips in all shipped views, pivots via
  `Desktop.OpenTool`, live-follow opt-in via `Wyred.Focus`; wire/pin
  affordances where a source view exists (census→detail as
  `OpenWiredPair`). Acceptance: clicking a net in a component's terminals
  lands on that net focused in the netlist view (new window, correct
  URI); focus-follow updates an opted-in second window; a cross-tree URI
  (two goldens trees opened as one set) pivots correctly.

### Phase 3 — the full review surface (milestone: WyredUiDesign v1 + v1.5 complete, hosted)

*3.1–3.3 independent.*

- **3.1 Locks & lineage** [~450]: lock groups/covers, series tree from
  `forked_from`, connector ICD tables, engine-reported/audit-checked
  labeling. Acceptance: watchy reva→revb renders a two-node fork tree
  with reason; connlock rows visible.
- **3.2 ECO diff** [~300]: the emitted `pinmapdiff` rendered —
  changed/added/removed side-by-side, sticky survivals highlighted,
  `minimal_disturbance_violations` as a verdict panel. Acceptance:
  watchy_v1_draft_btn3 renders changed-only-btn3 with sticky survivals.
- **3.3 Intent view** [~350]: roles, rails/grounds/bonds, buses, pools,
  invariants, escalations as rich objects; placement/test declarations
  when present. Acceptance: intent_05b escalations render rich.
- **3.4 Settle the gryth-ui mount + review foundation** [~120, gryth
  owner's call]: the 0.2 test mount either becomes permanent, goes
  behind a build flag/env, or stays a dev-only convention — decide and
  record; wyred windows dock into HUB areas by role either way. Also the
  moment to propose a "review" foundation preset (pure data) if wanted.

### Phase 4 — the review loop, local-first (milestone: J1 on one desktop — findings anchored, focus followed, agent-drivable; no glade)

- **4.1 Annotation channel** [~450]: the §3.5 record + `Wyred.Annotations`
  doc-scope-declared atom + handles, the `wyred.annotations` browser
  tool, anchor chips + counts in shipped views, resolution flow
  (intent-edit / stance / wontfix, stance note exportable as text).
  Depends 0.3 + any Phase-2 view. Acceptance: annotate C7 from the BOM →
  chip appears in netlist + pinmap wherever C7 renders; resolve as
  stance → browser shows it closed with the note.
- **4.2 Review-session UX** [~300]: presenter/follow toggle over
  `Wyred.Focus`, participant/author badges from `PrincipalRef` (stub
  identity labeled as such), "local only" vs "shareable" set affordance.
- **4.3 Watch-driven notifications + links-in-chat** [~250]: subscribed-
  series watcher posts pinmapdiff-arrival annotations with `wyred.eco`
  links (J2, local form); wyred `ToolLink`s render/embed in the chat
  plugin's messages where its message shape permits (coordinate with
  gryth's chat decisions — don't fork the shape).
- *Agent check (no new code):* every 4.x surface is writable through
  handle grips — verified by a seam-style test driving annotate + focus +
  open-tool headlessly (the gryth `seam.test.ts` pattern).

### Phase 5 — the shared design document (milestone: two desktops, one board fleet; **gated on gryth seam per §3.7**)

- **5.1 Share decls + store decision** [fable, ~250]: mount `Wyred.Set`
  (all-static), `Wyred.Focus`, `Wyred.Annotations` as glial commons
  surfaces (`chat/live.ts` template); decide the glade-backed
  ArtifactStore transport (ops-carried bytes vs pointers+hashes with
  client fetch) with glade-side input; hash-pin chips either way.
  Local-only fallback preserved when no node is present.
- **5.2 Glade-backed store behind the matcher** [~400]: the store
  implementation per 5.1, introduced via `grok.addBinding(withOneOf(…))`
  — wyred as the first real user of gryth's provider flip; Directory/
  Static formally become dev/standalone modes (journeys §8). Zero view
  changes is the acceptance.
- **5.3 Multi-party review polish** [~250]: converging annotations/focus
  across two clients against a local node; attribution rendered
  throughout; J1 two-party walkthrough as the acceptance script.

### Deferred (dependencies named)

`wyred.coverage` (J3) — on the coverage-artifact contract proposal;
`wyred.bench` measurement-entry family (J4) — on the ratified
measurement-record contract (+ §2 authored-records mechanics; typed entry
before instrument streaming); `wyred.release` + connector plugins (J5) —
on the release-bundle-manifest proposal, connectors separately fenced;
verification dashboard / test-bench / physical gallery (design §4.7–4.9)
— as their file formats stabilize; graph netlist rendering; gate-log
contract proposal; engine-level design-set/system notion; sourcing-slot
contract work (J6 — contract, not UI); any write path to emitted
artifacts (never).

## 6. gryth-side dependencies

**Phases 0–4: one shim.** ToolId is already open; the desktop is already
a library; the only gryth-ui change is the 0.2 test mount — one
`src/plugins/wyred/` dir + one `file:` dep line, auto-discovered by the
existing plugin glob, reversible by deletion (permanence settled in 3.4).
Phase 4 is deliberately local-first. Niceties, when wanted: a "review"
foundation preset. **Phase 5 rides gryth's existing
agenda** (commons surfaces consumable by an external plugin; chat message
shape for embedded links) and adds no new demands to it. **The journeys'
agent end-state** (scoped session capabilities, enforced grants,
attribution) is gryth C3 roadmap — wyred's readiness obligation is
discharged by everything-is-a-grip (§3.7), verified by the Phase-4 agent
check.

## 7. Risks and open questions

- **R1 — plugin contract churn** (draft v2, 0.x): absorbed at the
  registration shim + bootstrap (§3.1); gwz pin means wyred rolls forward
  deliberately. Worst case is a mechanical shim rewrite measured in tens
  of lines.
- **R2 — `@grythjs/desktop` as a hostable library**: gryth-ui's app is
  its only consumer today; welcome/grid builtins and shell taps may
  assume app-local wiring. Deliberately deferred to Phase 1.5 (dev rides
  the gryth-ui test mount until then, which needs no standalone
  hosting); fallback is upstreaming small seams (likely trivial,
  possibly zero).
- **R3 — cross-workspace TS source compilation**: gryth-ui had to relax
  tsconfig strictness for the glade-wz deep-compile; wyred-ui may need
  the same accommodations toward gryth-ui. Known, contained, template
  exists.
- **R4 — journey scope creep**: J3/J4/J5 each carry a contract
  dependency; this plan builds none of them early and names all of them
  (§5 Deferred) so "the UI is waiting on a contract proposal" is always a
  visible, deliberate state — the WyredPlanIndex serialization culture
  applied to UI inputs.
- **O1 — StaticStore census discovery**: `census.json` manifest if
  present (a wyred-ui convention, NOT a contract artifact — proposing it
  as an emitted artifact is a future contract proposal in the gate-log's
  spirit), else parse the autoindex HTML (`python3 -m http.server` case),
  else explicit error with guidance. Decide the manifest shape in 1.1;
  revisit as a contract proposal when the coverage/bundle proposals go
  in (an emitted census/manifest would also give StaticStore watch a
  clean etag).
- **O2 — printing the pinmap** ("printable/exportable as-is") inside a WM
  desktop: recommended shape — a solo-view print route in the host app
  (render one tool full-page from a serialized ToolLink,
  `window.print()`); desktop-as-data makes this cheap. Decide in 2.2.
- **O3 — entity-URI shape** (journeys §9 Q4): 0.3 proposes; Gianni
  blesses. The design-set segment's stem-unique resolution rule (§3.2)
  is this plan's input to it.
- **O4 — journey v1 selection** (journeys §9 Q1): this plan implements
  the journeys' own recommendation — J1+J2 first (phases 1–4), J3/J4/J5
  deferred on their contracts, J6 as colouring. If Gianni picks
  differently, phases 2–4 reorder without architectural change.
- **O5 — npm scope/naming** (`@wyredjs`?) and whether `plugin-wyred` ever
  publishes (gryth publishes `@grythjs/*` via changesets when the
  contract stabilizes; wyred can stay `file:`-consumed indefinitely).
  Phase 0.1 records the choice.
- **O6 — census-count acceptance**: all acceptance text above reads
  counts from the tree (WyredPlanIndex rule 4); the ga019 numbers cited
  (19 stems / 115 files) are as-of this survey, correct until the next
  contract census event.
- **O7 — gryth maturity gate** (journeys §9 Q2): answered by this plan's
  structure — phases 0–4 need only what gryth-ui already has (contract +
  desktop + mock discipline); Phase 5's gate is stated in §3.7. No
  further gate needed.

## 8. Proposed DecisionLog entry (append on acceptance)

> **wyred-ui: architecture + plan for the gryth-ui direction (follows the
> 2026-07-13 stack decision recorded in WyredUiUserJourneys.md).**
> wyred-ui = three packages (`artifacts` pure library / `plugin-wyred`
> gryth plugin with plugin-api/grip-react/react as PEER deps / `app`
> standalone host = `@grythjs/desktop` + wyred plugins only), consuming
> gryth-ui/grip as commit-pinned gwz members via `file:` deps (the
> glade-wz precedent). Dev order: gryth-ui test mount first (one shim
> dir + one dep line in gryth-ui, reversible), standalone host at Phase
> 1.5. gryth-ui coding rules adopted wholesale (no React state — grips/
> taps only; scanner + eslint bans; scope-declared grips + `*.Tap`
> handles), which is also the glial/glade readiness guarantee. Design sets (multi-tree fleets)
> are the workspace unit; entity URIs (short proposal, pre-plugin-work)
> anchor focus, cross-links, and the local-first annotation channel; the
> ArtifactStore seam ships verbatim with Directory/Static as
> dev/standalone modes and the glade-backed doc-scope store as the target
> collaborative mode behind gryth's matcher flip. Fence re-scoped: emitted
> artifacts immutable; the UI may author provenance-stamped collaboration
> data and (later, J4) contract-shaped input records; never runs
> toolchain, never derives. Phases 0–4 (J1+J2 local-first) have zero
> glade/server dependency; Phase 5 (shared design document) gates on
> gryth's seam. Design+plan: GrythWyredUiDesignPlan.md; WyredPlanUi.md
> superseded; WyredUiDesign.md remains authoritative for surface content
> and trust rules, §7 stack section superseded per journeys §8.
