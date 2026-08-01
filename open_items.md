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
