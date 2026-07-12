# WyredUiUserJourneys — the workflows the UI must serve

*Companion to WyredUiDesign.md (which needs a stack rework after this —
see §8). Stack decision (Gianni, 2026-07-13): the UI is built on
**gryth-ui** — the plugin-based collaborative UI backed by glade, where
agents connect via session capabilities and collaborate alongside humans.
That decision reframes wyred-ui from "a viewer" to "wyred surfaces as
gryth plugins over a shared design document." These journeys are the
requirements source for that rework.*

## 0. Personas

- **Dana** — hardware designer; authors wyred intent, owns layout.
- **Femi** — firmware engineer; lives in the pin-map; consumes HAL stubs.
- **Tess** — test engineer; owns the bench, the harness boards, bring-up.
- **Rey** — reviewer/release owner; ratifies stances, cuts releases.
- **Aya** — an AI agent (any of: design reviewer, HAL generator,
  procurement negotiator); connects to the gryth doc with a **scoped
  session capability** — the same collaboration surface as a human, with
  narrower permissions.
- **Pat** — procurement/production (mostly future; journey J6 colours for
  them).

## J1 — Inner design loop (Dana + Aya-reviewer)

Dana edits intent in the IDE; emit + gate run (CLI); the gryth **doc
scope** holds the artifact tree, so every open plugin view updates live.
Dana asks Aya to review; Aya connects (session capability: read artifacts,
write annotations), walks the netlist/pin-map/BOM plugins, and drops
findings as **annotations anchored to entities** — a refdes, a net, a
lock group, a verdict code. Dana follows Aya's focus (session-scope
sharing — "open the debugger for me" generalized to "show me what you're
looking at"). Findings resolve into either intent edits (back to the IDE)
or recorded stances (DecisionLog culture, in-UI).

**Needs:** entity addressing (§7); annotation channel; live tree watch;
per-view plugins. **Exists today:** artifacts + gate verdicts as files.

## J2 — ICD sync across the hardware/firmware seam (Femi + Dana + Aya-HAL)

Femi subscribes to the pin-map plugin of a **frozen series**. An ECO
lands (unlocked path) or a fork (locked path): the series tree updates in
the doc, Femi gets a change notification whose payload IS the pinmapdiff
artifact — rendered as changed-rows-only with sticky survivals labeled.
Aya-HAL regenerates the firmware self-test/HAL stubs (wyred-bench
generator) against the new pin-map and posts the code diff to the doc for
Femi's review. Lock badges and `(series, lock-version)` stamps are on
every surface so "which pin-map am I coding against" is never ambiguous.

**Needs:** subscription/notification on artifact change; diff rendering;
agent-posted work products. **Exists:** pinmapdiff, locks, stamps,
firmware generator.

## J3 — DUT + test-harness co-design (Dana + Tess) — the fleet journey

Complex products are **board fleets**: the primary board(s) plus the
**pogo-pin test-harness board** that mates with the DUT's testpoints and
verifies operating parameters. The harness is itself a wyred design whose
key geometry is *derived*: DUT layout → testpoint coordinates
(pcb_extract geometry) → pogo placement constraints on the harness
(placement vocabulary) → fixture mechanicals (wyred-3d, rung 2).

Journey: Tess opens the fleet workspace — DUT and harness side by side.
The **coverage panel** shows every testplan check joined to its DUT
testpoint and its harness pogo: checks with no probeable point
(`TESTPLAN_UNPROBEABLE`), testpoints no harness pin reaches, pogos with
no purpose — each a first-class gap, rendered from emitted artifacts
(coverage is emitted by the toolchain as an artifact, not derived by the
UI — render-what-was-emitted holds). When Dana moves a testpoint in
layout, the harness's derived placement constraints go stale: the gate
catches it, the UI shows the disagreement on both boards, cross-linked.

**Needs:** multi-board workspace (a "design set" grouping of artifact
trees — UI-level now, engine-level system notion later); a coverage
artifact (contract proposal); DUT↔harness cross-links. **Exists:**
testplan/probe points, placement machinery, 3D chain (rung 2 pending).

## J4 — Bring-up bench session (Tess + the harness)

Tess at the bench: the bench-card plugin is live in one pane, the DUT
views in another. Measurements enter (typed now; instrument-streamed
later) into a measurement record **in the doc**; the comparator verdicts
appear per check; a failure cross-links check → net → component →
provenance ("this rail is derived from THAT declaration"). The completed
measurement record — board serial, testplan stamp, verdicts — is saved as
the bring-up session's artifact; the run_gate physical stage consumes it.
Multiple bring-up sessions accumulate per serial (the start of a
production-test history).

**Needs:** measurement-entry plugin writing contract-shaped records;
per-serial session history in the doc. **Exists:** testplan, comparator,
verdict codes, stale-stamp refusal.

## J5 — Release: the digital-asset bundle (Rey)

"Building" a board (or fleet) produces a **bundle of digital assets**
that must travel to different systems: gerbers/drill/pos (fab), BOM +
placements (assembly), pin-map + HAL stubs (firmware), testplan + bench
cards + fixture files (test), STEP/renders (mech/docs), audit +
agreement evidence (quality), baselines (the ICD anchor). Journey: Rey
opens the **release view** — the bundle manifest assembled from the
artifact tree: every asset hash-pinned, toolchain-version-pinned (the
fab_manifest pattern generalized), with the gate/audit evidence attached
("this bundle was gate-green at these floors on this ground truth").
Rey walks the ratification checklist (open stances block release — the
human pass as a UI-enforced journey, the 13-stances lesson). Then
**share flows**: each destination (fab house, assembly, customer,
verification system) is a gryth connector plugin; every share is a
tracked act in the doc — who sent what, where, when, which hash.

**Needs:** release-bundle manifest (contract proposal — the lynch-pin
candidate, §6); connector plugins; ratification-checklist state in the
doc. **Exists:** all the assets, fab_manifest precedent, audit evidence.

## J6 — Production strategy (Pat + Aya-procurement) — future, coloured for NOW

Not designed now; designed *toward*, deliberately. The future journey:
Aya-procurement takes the release bundle, locates suppliers, negotiates
quantity pricing — in the supplier's language — proposes alternates when
parts are unavailable, and brings the negotiation back as a structured
proposal: alternates land as **refinement-layer rebinds** (the two-layer
architecture's supply-chain story: intent untouched, `bind()` swapped,
engine re-verifies, gate re-runs), which Rey ratifies.

**The lynch-pin features that bend the design this way (proposed):**

1. **Sourcing slots in the BOM contract.** Every BOM line item gains a
   stable part-identity key (the wyred-parts qualified name) plus a
   structured, initially-empty `sourcing` slot: acceptable-alternates
   policy, quantity brackets, negotiated-terms attachment point. Costs a
   contract field now; gives every future procurement agent a machine
   surface instead of a CSV scrape.
2. **The release bundle as the negotiation artifact.** Quantities, test
   requirements, alternates policy, and acceptance evidence travel *in*
   the bundle — an agent negotiating with a supplier hands over exactly
   the ratified subset, hash-pinned. No parallel "procurement package"
   ever forks from the engineering truth.
3. **Agents are capability-scoped collaborators, uniformly.** The
   procurement agent uses the same gryth session-capability model as the
   design reviewer — no special integration per agent role. Building
   J1's agent plumbing correctly IS the production-strategy investment.

## 7. Cross-cutting requirement: entity addressing

Every journey anchors on entities: refdes, net, demand id, role id, lock
group, check id, constraint id, verdict code, artifact stem/kind, board
serial. Collaboration (annotations), agents (findings), and cross-links
all need **stable entity URIs** — proposed shape:
`wyred://<design-set>/<stem>/<kind>#<json-pointer-or-entity-id>`. This is
the one piece of new *contract-adjacent* design the gryth direction
forces early; it should be its own short proposal before plugin work
starts (ids already exist and are stable — law 8 minting did the hard
part years of discipline ago; this is just the URI envelope).

## 8. Consequences for the existing UI docs

- **WyredUiDesign.md**: §3.1's ArtifactStore seam SURVIVES — it becomes
  the glade-backed provider (the doc-scope artifact tree) with
  DirectoryStore/StaticStore demoted to dev/standalone modes. §7's
  plain-React stack section is SUPERSEDED by the gryth-ui decision. The
  trust posture (§2 render-what-was-emitted, schema banners, labeled
  self-reported verdicts) carries over unchanged — it matters MORE with
  agents in the loop.
- **WyredPlanUi.md**: superseded pending redesign — the phases become
  gryth-ui plugin increments, and a Phase 0 appears: gryth-ui/glade
  integration spike + the entity-URI proposal. Do not build from the old
  plan.
- **Surface inventory** (design §4) remains the plugin catalog; the
  journeys add: coverage panel (J3), measurement-entry (J4), release
  view + connectors (J5), annotation/finding channel (J1).
- **New contract proposals eventually needed:** coverage artifact (J3),
  release-bundle manifest (J5), sourcing slots (J6), entity URIs (§7).
  None block the current buildout; all are additive.

## 9. Open questions for Gianni's review

1. Which journeys are v1? (Recommendation: J1 + J2 first — they exercise
   the gryth plumbing with artifacts that exist today; J3/J4 need the
   coverage artifact and rung-2 3D; J5 next; J6 stays a colouring.)
2. gryth-ui maturity gate: what must exist in gryth-ui/glade before
   plugin work starts (it is mock-first and pre-provider today)?
3. The lynch-pin: adopt sourcing slots (§6.1) as a contract proposal
   now, or wait for the first real procurement conversation?
4. Entity-URI shape (§7) — bless the sketch or redesign?
