# Test-intent semantics + `.testplan.json` contract proposal (WyredPlanTestplan step 1.1)

*Status: DRAFT awaiting Gianni's ratification (lands in wyred-contract after
the schemas + kicad proposals, per WyredPlanIndex ordering). All seven
RATIFY items carry a decided stance below — each is a deliberate loosening
of the rigid-scalar model and none lands without your recorded sign-off in
DecisionLog. The design-time `VTOL`/`_VTOL = 0.05` constants are untouched.*

## 1. Vocabulary (marker declarations)

```python
expect_rail(tp, volts=None, tol=None, tol_pct=None)   # exactly one of tol/tol_pct
expect_i2c_scan(bus, addrs=None)                       # addrs default: derived
expect_current(rail, max_ma, state)                    # one-sided upper bound
expect_signal(tp, freq=None, freq_tol_ppm=None, freq_tol_pct=None,
              duty=None, duty_tol_pts=None)
```

- References are **L1 vocabulary only**: `tp`/`rail` name rails or demand
  ids, `bus` names a declared bus — never refdes (law 8 territory).
- `late(...)` is legal in every argument (resolve_value recurses).
- Check identity is minted: `<instance path>.<declaring attribute name>`.
- Structured load errors (law 10): `TEST_UNKNOWN_RAIL`, `TEST_UNKNOWN_BUS`,
  `TEST_UNKNOWN_DEMAND`, `TEST_BAD_TOLERANCE`.

## 2. Testpoint binding — inference, fail-closed

Testpoint-ness is **inferred from realized `kind == "test_point"`
components** (keeps L1 untouched; the Watchy corpus already realizes
DebugTaps as TP1–TP7). The emitter resolves each check to concrete probe
points `{refdes, pad, net}` from the pinmap: a rail check binds to the
test_point on that rail's net **plus a ground-reference test_point** (the
GND-net test_point; both required). A check whose target has no probeable
entry is `TESTPLAN_UNPROBEABLE` **at emit** — a structured failure, never a
silently thinner testplan. `expect_current` is the exception: it needs no
testpoint (probe method — shunt/supply readout — is a bench-card matter).

## 3. Where declarations live — the self-contained artifact

The testplan artifact carries its own `declarations` block (authored
content, elaborated from the markers) plus a `checks` block (derived from
declarations + records + pinmap). Consequences, ratified together:
`l1.json` never changes (goldens byte-identical; no harness/schema_l1
change in Phase 1); `checks` is re-derivable from on-disk inputs, so the
testplan joins `wyred.rebuild` / `REBUILD_FROM_PRIMARIES`; wyred-bench
consumers need the testplan (+ pinmap) only — bench cards derive without
the engine.

## 4. The RATIFY stances

- **RATIFY-1 — interval form: closed absolute interval; tolerance is
  REQUIRED.** `expect_rail` acceptance is `[volts − t, volts + t]` with
  `t = tol` (volts) or `t = volts × tol_pct/100` — exactly one of
  `tol`/`tol_pct` must be given; **omitting both is a load error**
  (`TEST_BAD_TOLERANCE`). Rationale: refusal over guessing — no universal
  measurement tolerance exists, and the one thing this vocabulary must
  never do is silently inherit the 0.05 V float-noise absorber.
- **RATIFY-2 — derived nominals: legal, one-way.** Omitted `volts` centers
  the interval on the L1 rail declaration's voltage
  (say-only-the-non-default). The derivation is one-way: nothing ever
  flows from a testplan back into design-time voltage semantics.
- **RATIFY-3 — the VTOL firewall stands.** `VTOL`/`_VTOL` remain
  float-noise absorbers, neither widened nor reused as measurement
  tolerance. Same unit, different meaning; any unification is its own
  future contract event.
- **RATIFY-4 — one-sided current bound + free-string states.**
  `expect_current` is `measured ≤ max_ma` (no lower bound in v0; a
  suspiciously-low current is real but out of scope, recorded). `state` is
  a free string in v0, matched **exactly** between testplan and measurement
  record (`STATE_MISMATCH` on any mismatch); an enum can be ratified later
  without breaking existing plans.
- **RATIFY-5 — per-quantity typed tolerances.** Frequency: exactly one of
  `freq_tol_ppm` (crystals) or `freq_tol_pct` (RC-class), required when
  `freq` is given. Duty: `duty_tol_pts` (absolute percentage points),
  required when `duty` is given. No cross-quantity defaults.
- **RATIFY-6 — I2C scan is EXACT-SET.** Expected addresses default from
  the L1/records bus attachments (`addrs=` overrides). A missing address
  is `I2C_SCAN_MISSING_ADDR`; an **unexpected extra address is
  `I2C_SCAN_UNEXPECTED_ADDR` and FAILS** — a rogue device is a
  disagreement, not a freebie.
- **RATIFY-7 — closed intervals; canonical numbers.** A measured value
  exactly at a bound **passes** (closed interval, consistent with
  RATIFY-1's `[..]` notation). Canonicalization: authored numbers pass
  through verbatim; **derived** numbers (computed bounds, derived
  nominals/addresses) are written as `round(x, 6)` JSON numbers; the
  comparator compares parsed floats against bounds exactly (no epsilon —
  instrument precision is the instrument's problem, and the bounds already
  encode the acceptable band).

## 5. `.testplan.json` shape

Canonical JSON, stamped like every artifact. Top level: `{artifact,
path: "testplan", stamp, declarations: [...], checks: [...]}`. Each check:
`{id, kind: rail|i2c_scan|current|signal, declared_by, subject
(rail/bus/demand name), expect (per kind, below), probe: {points:
[{refdes, pad, net}], ground_ref | null}, provenance: {declaration id,
derivation inputs}}`. Checks sorted by id.

**`expect` shapes are FLAT, pinned per kind** *(pinned 2026-07-13 closing
critic finding F1 — the earlier "freq…, duty…" underdetermination let the
engine and the bench comparator implement incompatible shapes; the flat
form below is what the engine emits and is now normative; nested forms
are illegal)*:

- `rail`: `{nominal, low, high}` (closed interval per RATIFY-1/7)
- `i2c_scan`: `{addrs: [int, ...]}` (exact-set per RATIFY-6)
- `current`: `{max_ma, state}` (one-sided per RATIFY-4)
- `signal`: `{freq, freq_low, freq_high}` and/or
  `{duty, duty_low, duty_high}` — each trio present **iff** its quantity
  was declared (RATIFY-5); never nested objects, never a partial trio.

Worked example (hand-derived from the committed watchy goldens —
every net/refdes/address verifiable there): `expect_rail` on `+3V3`
`tol_pct=5` → nominal 3.3, low 3.135, high 3.465, probe TP4 (net `+3V3`),
ground_ref TP5 (net `GND`); `expect_i2c_scan("I2C0")` → derived
`addrs: [24]` (0x18, accel, from records), probe TP7/TP6 (SDA/SCL).
**Exactly ONE canonical example artifact is committed** — the
engine-emitted `watchy_v1_bench.testplan.json` (byte-identical between
`wyred-examples/testplan_expected/` and the bench fixtures; a divergent
hand-authored twin is a defect, per F1). Synthetic comparator fixtures
exercising shapes the corpus deliberately does not declare (e.g. a duty
band) carry DISTINCT filenames.

## 6. Measurement record + verdict codes (v1-draft, final at step 2.2)

`<name>.measurements.json`: `{artifact, testplan_stamp: {series, locks},
board_serial, state?, results: {check_id: {value | values | addrs, unit,
note?}}}`. Comparator verdicts (contract vocabulary): `RAIL_OUT_OF_RANGE`,
`I2C_SCAN_MISSING_ADDR`, `I2C_SCAN_UNEXPECTED_ADDR`, `CURRENT_EXCEEDED`,
`SIGNAL_OUT_OF_RANGE`, `CHECK_UNMEASURED` (an unmeasured check FAILS — no
silent defaults extends into physics), `MEASUREMENT_UNKNOWN_CHECK`,
`UNIT_MISMATCH`, `STATE_MISMATCH`, plus the refusal rule: a measurement
record whose `testplan_stamp` differs from the testplan's own stamp is
**refused, not compared** (stale-stamp fixture proves it).

## 7. Ratification checklist

- [ ] vocabulary signatures + load-error set (§1)
- [ ] testpoint inference + ground-reference pairing + `TESTPLAN_UNPROBEABLE` (§2)
- [ ] self-contained artifact / declarations-block decision (§3)
- [ ] RATIFY-1 … RATIFY-7 stances (§4) — each individually
- [ ] artifact + measurement-record shapes and codes (§5–6)
- [ ] landing order after schemas + kicad proposals
