# WyredProposalEntityUri — stable entity URIs for wyred surfaces

*Status: **proposal**, awaiting Gianni's ratification (journeys §9 Q4 /
GrythWyredUiDesignPlan Phase 0.3). Scope: the URI **envelope** over ids
that already exist and are stable (law-8 minting did the hard part) — this
is contract-ADJACENT collaboration-layer design, not a contract change. No
artifact ever emits or contains these URIs; proposing that would be a
separate contract proposal.*

## 1. Why now

Every journey anchors on entities (WyredUiUserJourneys §7): annotations
(J1), agent findings (J1/J2), cross-links and shared focus (all), coverage
joins (J3), measurement cross-links (J4). All of them need one stable,
serializable way to say *"this refdes on this board in this fleet"* that
survives being embedded in a chat message, an annotation record, a
`ToolLink` param, or an agent's plan. The ids exist; the envelope doesn't.

## 2. The shape

```
wyred://<design-set>/<stem>[/<kind>]#<fragment>

fragment  = <type> ":" <id>          ; typed entity (preferred)
          | <json-pointer>           ; exact address ("/"-rooted, RFC 6901)
```

Formally (RFC 3986): `<design-set>` occupies the authority slot — accepted
and documented; the scheme is internal, never dereferenced over a network.
The URI names an entity **relative to a resolver that holds the design-set
definition** (the `Wyred.Set` grip locally; the shared set document in
Phase 5). URIs never embed transport — no store URLs, no file paths; the
set definition maps stems to trees/stores.

### 2.1 Segments

- **`<design-set>`** — the set's name, or `-` for "the resolver's current
  set" (the common local case: one unnamed set open on one desktop).
  Named sets matter once sets are shared (Phase 5) or once two sets are
  open at once; until then `-` keeps local URIs short and honest about
  their locality.
- **`<stem>`** — the design name (`watchy_v1_reva`). Stems resolve
  **set-wide and must be set-unique** (GrythWyredUiDesignPlan §3.2): the
  set resolves stem → tree; a cross-tree stem collision is a loud census
  error, never a silent pick. This is why the URI needs no tree segment.
- **`<kind>`** (optional) — one of the contract kinds (`l1 l2 alloc bom
  pinmap pinmapdiff records baseline connlock lifecycle cir`). Present:
  the URI is anchored to that artifact (required for json-pointer
  fragments; meaningful for typed fragments when the anchor is
  kind-specific, e.g. a BOM line). Absent: the entity belongs to the stem
  as a whole (a refdes exists in l2, bom, and pinmap at once) and the
  entity index picks the home view.

### 2.2 Typed fragments — the entity vocabulary

| type | id example | source of the id |
|---|---|---|
| `refdes` | `refdes:C7` | l2/bom/pinmap component identity |
| `net` | `net:VBAT` | l2 net names |
| `demand` | `demand:mcu.uart0` | l1/alloc/pinmap demand ids |
| `role` | `role:power.3v3` | l1 role ids |
| `pool` | `pool:gpio_a` | l1/alloc pool ids |
| `lockgroup` | `lockgroup:icd_j1` | records/connlock lock groups |
| `check` | `check:rail_3v3_range` | testplan check ids (J4) |
| `code` | `code:TESTPLAN_UNPROBEABLE` | verdict/gate code vocabulary |
| `stem` | `stem:watchy_v1_reva` | the design itself (fragment-free `wyred://-/watchy_v1_reva` is equivalent) |
| `kind` | — | not a fragment; use the path segment |

The vocabulary is **closed per revision of this proposal** — an unknown
type renders as a dead chip ("unrecognized entity type"), never a guess.
Additions (e.g. `serial:` for J4 board serials, `asset:` for J5 bundle
members) are one-line amendments here, made when their journeys build.

### 2.3 JSON-pointer fragments

`wyred://-/watchy_v1_reva/pinmap#/allocations/12/pad` — byte-exact
anchoring into one artifact, RFC 6901 semantics against the parsed JSON.
Requires the `<kind>` segment. This is the provenance-grade form (an
annotation that must survive entity renames); typed fragments are the
human/collaboration form. Both resolve through the entity index /
artifact cache — **resolution is lookup, never derivation**; a pointer
into an unloaded or absent artifact resolves to "not loaded / absent",
loudly, per the fence.

### 2.4 Encoding

Ids minted by the toolchain use `[A-Za-z0-9_.-]` in practice; any byte
outside the RFC 3986 `pchar`/`fragment` sets is percent-encoded. Set and
stem segments likewise. Comparisons are exact-match after percent-decoding
(no case folding — ids are case-significant).

## 3. Resolution rules

1. Resolve `<design-set>`: `-` → the resolver's current set; a name → that
   set if held, else **unresolvable** (rendered as such — a URI from chat
   for a set you don't hold is a dead chip with the set name shown).
2. Resolve `<stem>` within the set (set-unique; collision already faulted
   at census time).
3. If `<kind>` present, anchor to that artifact (absent artifact →
   "absent", schema-invalid → the raw/SCHEMA_INVALID view).
4. Resolve the fragment via the entity index (typed) or the parsed JSON
   (pointer). Index misses render as "not found in loaded artifacts".

`Wyred.Focus` holds exactly one of these URIs (or none). Annotations
anchor on one. `ToolLink` params carry them as strings
(`focus: 'wyred://-/watchy_v1_reva#refdes:C7'`).

## 4. Explicitly out of scope (recorded so absence is a decision)

- **Series/revision addressing** (J2's "frozen series @ lock-version") —
  stems already name revisions; addressing a *series* rides the
  lock/lineage vocabulary and is proposed when the J2 notification work
  builds. Reserved, undesigned.
- **Board serials** (J4) — `serial:` type reserved.
- **Cross-set links** — a URI names entities within one set; fleet-to-
  fleet linking has no journey yet.
- **Emitting URIs from the toolchain** — separate contract proposal if
  ever wanted.

## 5. Open questions for ratification

1. Bless `-` as the anonymous-set authority (alternative: require named
   sets from day one — heavier, but no rename migration later)?
2. Typed-fragment separator `:` (`refdes:C7`) — OK, or prefer `=`?
3. Is the closed type vocabulary in §2.2 the right starter set?
