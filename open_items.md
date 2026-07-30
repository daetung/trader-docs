# Open Items

**Reference/tracking document, not a spec file — but PATCH-DELIVERED like
one.** Renamed from `open_items_session6.md`; the name is now fixed and
carries no session number, and this is the only open-items file. There is no
per-session successor to write and no supersession to declare.

That change exists to remove a specific failure mode. Under the old scheme
each session rewrote the whole file, carrying unresolved items forward by
hand — and every defect found in Session 7's opening audit was a carry-
forward defect: a pointer to a file that no longer existed, a cross-
reference describing behaviour that had since been designed, one of three
copies of a schedule time left un-updated. A single file amended by patch
cannot fail that way, because unchanged items are never retyped.

It remains OUTSIDE the handoff's Spec File Structure — patch delivery is a
safety property of how it is edited, not a claim that it specifies anything.

This file holds **only what is not yet reviewed or resolved.** Descriptions
below are problem statements and first-pass thinking, not confirmed designs
— re-review from the problem each time. An item's text is NOT rewritten when
a session leaves it untouched; absence of edits means absence of review, not
confirmation that it still reads correctly.

---

## Suggested order

0. **LiveModeRunner process shutdown time** comes before everything else and
   is cheap: it is a missing value, not an open design question, and two
   things already lean on it (see below).
1. **`api_doc/` normalization** is a prerequisite for the item below —
   normalizing once and reconciling rate limits in the same pass avoids
   reading the raw docs twice.
2. **Rate-limit reconciliation** follows directly from 1; the global 20 TPS
   figure is known, per-endpoint figures are presumably in `api_doc/` but
   blocked on 1's normalization pass.
3. **Bid/ask spread — model-feature / entry-gate paths** is independent of
   both — blocked on calendar time (months of accumulated `bid_ask_snapshots`
   history), not on design time, so no urgency to sequence it ahead of the
   above.

---

## `api_doc/` normalization

**Problem.** Raw trading-API documentation files (an `api_doc/` folder —
WS quote, WS trade-price, and WS order-fill-inquiry JSON specs confirmed so
far; possibly more, not yet enumerated) were found in project knowledge
mid-session, not previously known to this process. User confirmed the
content is up to date but declined to have it reflected into spec design
directly until a normalization pass is done first — only the specific facts
the user explicitly confirmed in chat this session were used (the bid/ask
endpoint's existence and 5-level-book shape; the trading API's per-exchange
bulk ticker-master call structure), broader use deliberately deferred.

**Not yet designed.** What "normalization" concretely means here (a
canonical extracted-facts document; direct annotation of
`api_contract_checklist.md` rows as each is confirmed against it; something
else), and who does it on what timeline, were not discussed this session.

---

## Rate-limit reconciliation with the API spec sheet

**Problem.** Still pending, blocked on the same normalization gap as the
item above — a global **20 transactions/second** limit across the trading
API is known; per-endpoint limits are presumably in `api_doc/`, not yet
extracted. Logged provisionally in `docs/ops/api_contract_checklist.md`'s
T-3 row.

**Session 6 additions to the tension already flagged in Session 5**
(`position_check_interval_seconds` (5s) concurrency vs. the 20 TPS global
cap): three more sources of load now land on the same 5-second cadence —
the halt-check bulk call's ticker list, extended this session to also cover
in-flight exit orders, not just open positions; `signal_time_rest`'s
bid/ask query, fired at every entry signal (frequency-proportional, not
fixed-cadence, so its contribution isn't a flat per-cycle number the way
the others are); and `exit_order_type="limit"`'s per-cycle bid re-query for
each outstanding limit exit.

**Not yet designed** — intentionally; still needs the normalized spec
sheet's per-endpoint numbers. Revisit once the item above is done.

---

## Real-time bid/ask spread — model-feature / entry-gate paths

**Problem.** Both paths originally described here needed two things:
confirmation a collection mechanism exists, and months of accumulated
history. The first is now resolved — collection is running as of this
session (`db_schema.md`'s `bid_ask_snapshots`; see `session_handoff.md`
item E). The second is not: neither path is usable until enough history has
accumulated, which this session does not change the timeline on.

- **As a model feature.** Same shape as before: needs retraining against
  the new historical feature once enough of it exists.
- **As an execution-time gate.** Same divergence concern as before:
  BacktestEngine still cannot replay a live bid/ask gate against historical
  data older than this session's collection start date.

The processing-model concern from Session 5 (quotes as a continuous stream
not fitting `on_bar_close()`) is resolved as a side effect of this session's
collection design — both sources are point-in-time (REST query / tick
piggyback), not a continuous stream, so no separate buffering/aggregation
layer is needed regardless of which path above is eventually pursued.

**Not yet designed** — still. Not actionable again until `bid_ask_snapshots`
has enough history; revisit then, not before.

---

## `api_contract_checklist.md` — verify before Pilot

Not a new design problem — a pointer, so it isn't lost among the items
above. `docs/ops/api_contract_checklist.md` holds **16** unverified
vendor-contract assumptions (T-12, halt-mid-flight resting-order behavior,
and T-13, vendor bar-arrival latency, were added in Session 6; Session 7
added none), two graded **A** (T-1: REST/WS tick granularity; T-7:
fill-event stream ID stability). Both A-graded rows must be verified, or
their fallback paths confirmed sufficient, before Stage 2 (Pilot) — see that
file's own "When to use it" section. Not duplicated here; consult that file
directly rather than letting a second copy of its contents drift out of
sync.

Session 7 moved T-13 out of the "needs a measurement exercise" category and
into the self-measuring set: `health_report.md` finding 26 now accumulates
the latency curve during ordinary operation. Its **B** grade is unchanged —
grade is what being wrong costs, which self-measurement does not alter. The
count above is unaffected: a self-measuring row is still unverified until
the measurement has actually run.

---

## LiveModeRunner process shutdown time — unspecified

**Top priority.** No spec states when the live process exits, and no config
key holds the value. It is not merely undocumented — the two candidate
answers are already in tension in the existing specs:

- Session close logic centres on 15:59 / `session_close_exit_time`, and
  after-market bars are excluded from entry detection in every mode.
- But in-flight exit tracking has NO terminating time: it reprices against
  the bid every `position_check_interval_seconds` and escalates to market at
  `exit_order_stuck_minutes`, which presupposes the process outlives 16:00
  whenever a position could not be liquidated at close. Halted tickers are
  the separate case — those are handed to the next session by the Unified
  Overnight Policy (finding 14), not worked after hours.

**Two things already depend on the answer.** `health_report.md`'s
`drain_timeout_seconds: 600` was sized against roughly 50 minutes of margin
between a ~20:00 session end and the 21:00 evening batch; that arithmetic is
only as good as the 20:00 assumption. And the Watchdog Polling Loop's
behaviour between 16:00 and shutdown is undefined — with no candidates,
finding 26 simply collects no samples, which is benign, but it is unstated
rather than decided.

**Not yet designed.** Needs the exit time itself, a config key to hold it,
and a statement of what the polling loop does in that window.

---

## Who executes the schema DDL

**Problem.** `db_schema.md` holds every `CREATE TABLE`, and
`migration_tool.md` assumes the tables already exist — its six processing
steps ingest into them and call the populate helpers, and it issues no DDL
at all. Nothing in between says who creates a database in the first place.
Pre-existing, not introduced by Session 7; surfaced when three new tables
were added and had nowhere to be registered.

**Not yet designed.** Whether that is a bootstrap step in `migration_tool`,
a separate init entry point, or an idempotent create-if-absent on startup.
Note the related question of what happens when this file adds a column to a
table that already exists in a live database.

---

## No global purge / retention policy

**Problem.** Purging exists in exactly one place — `indicator_cache`, inside
`persist_to_db()`, per ticker, on a cold session start. Every other table
grows without bound, some deliberately (`tick_bar_aggregates` is documented
as a permanently-growing historical store, and Session 7's three new tables
are explicitly never purged). There is no retention config key anywhere.

**Design constraint, established in advance.** A single global retention key
would be dangerous rather than merely crude: the same key would reach
`ohlcv_1min` and `tick_10` (the training corpus) and `trade_log` (the P&L
record). One misconfiguration would be unrecoverable. Any design here should
be a per-table retention registry with permanently-retained tables excluded
structurally — not by defaulting their window to infinity.

**Not yet designed.** Not urgent while disk holds; listed so the constraint
above is not rediscovered under time pressure.

---

## Crash recovery for in-memory-aggregate findings

**Problem.** Findings 5, 8, 12, 21, 22, 24 and 25 are computed from tallies
LiveModeRunner holds in memory and hands to `health_report.py`. If the
session crashes, its scheduled session-end call never happens and those
findings are lost outright — no table holds their inputs.

Session 7 established the evening liveness probe as a partial backstop: it
sweeps every DB-computable finding for a crashed session, which recovers the
ones whose inputs happen to live in tables. The findings above are exactly
the ones it cannot reach.

**Not yet designed.** `health_events` (added in Session 7) is the obvious
vehicle for the event-shaped ones, which overlaps the item below. Findings
21 and 24 are not event-shaped — 21 is a set of session-start probe
measurements, 24 a staleness observation — so they need their own answer, or
an explicit decision that losing them on a crash is acceptable.

---

## `health_events` — adoption beyond finding 27

**Problem.** `health_events` was defined at full width in Session 7 but only
finding 27 writes to it; that was deliberate, to keep the schema decision
and the conversion work apart. The other event-shaped findings — 12, 13, 14,
15, 18, 25 — still report a same-day count with no record of WHEN each
occurrence happened, and no link to the alert that reported them.

**Not yet designed.** Each conversion means placing a
`record_health_event()` call at that finding's detection site and changing
the finding to aggregate the table instead of a tally, so the work is spread
across several specs and is best done as its own pass. Worth settling
alongside the crash-recovery item above, since converting a finding resolves
both questions for it at once.
