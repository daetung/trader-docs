# Open Items — Session 6

**Reference/tracking document — same tier as `session_handoff.md`, not a spec
file.** Supersedes `open_items_session5.md` in full: the stuck exit-order
policy, the account fill-stream-disconnect-vs-Feed-Outage-trigger question,
and ticker-symbol normalization are all resolved this session. Real-time
bid/ask spread is partially resolved — collection is now designed and
running; the model-feature and entry-gate paths it was originally meant to
unblock remain open, restated below with their status updated. See the
patch set: `execution_common.patch`, `db_schema.patch`,
`api_contract_checklist.patch`, `health_report.patch`,
`metadata_crawler.patch`, `live_mode_runner.patch`, and the three deltas
against it — `bid_ask_and_ws_timing.patch`,
`warm_restart_subscription_scope.patch`, `shadow_ws_subscribe.patch` (apply
in that order). This file may be removed once its own items are resolved or
superseded, same as `open_items_session5.md` now is.

This file holds **only what is not yet reviewed or resolved.** Descriptions
below are problem statements and first-pass thinking, not confirmed designs
— re-review from the problem each time.

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
above. `docs/ops/api_contract_checklist.md` now holds **16** unverified
vendor-contract assumptions (2 added this session: T-12, halt-mid-flight
resting-order behavior; T-13, vendor bar-arrival latency), two graded **A**
(T-1: REST/WS tick granularity; T-7: fill-event stream ID stability) —
unchanged this session. Both A-graded rows must be verified, or their
fallback paths confirmed sufficient, before Stage 2 (Pilot) — see that
file's own "When to use it" section. Not duplicated here; consult that file
directly rather than letting a second copy of its contents drift out of
sync.

---

## Deliberately settled (do not re-open)

Recorded so they aren't re-derived. Carried forward from
`open_items_session5.md` plus this session's additions:

- Price-gap probe-threshold tuning (old N option A/C) — dropped; the vendor
  cross-check supersedes it.
- Per-position force-flatten on a breaker trip — rejected (R-4).
- Same-ticker entry hard block — rejected (R-5); the cooldown-based
  substitute depends on `entry_cooldown_minutes * 60 >=
  cancel_after_seconds`, which is now an explicit config-validation note,
  not an unstated assumption.
- Two-writer lock-handoff for the 09:20 recheck — rejected (R-1); in-process
  delegation replaced it.
- Poll-delay 5s alignment in backtest — removed (R-2 revision).
- Bundle-based backstop with low-trimming — dropped; REST-tick backstop at
  WS granularity replaced it.
- Circuit breaker in BacktestEngine — computed every run (same three
  metrics as live), never enforced, for the same reason Shadow mode doesn't
  enforce it: a real trip truncates exactly the data a run exists to score.
- WS sequence lease default — `shared`, not `dedicated` (R-4/R-7).
- Page an operator for a stuck exit order — not adopted; no paging
  infrastructure exists (Discord/email only), and the periodic
  bid-tracking reprice + stuck-timeout market escalation was judged
  sufficient.
- Do nothing beyond the existing alert for a stuck exit order — rejected;
  inconsistent with Exit Architecture's own "maximally aggressive, minimize
  exposure" philosophy, once the backtest comparison confirmed backtest
  doesn't actually have a deliberate give-up timeout either (only a
  data-exhaustion boundary).
- `min_absolute_miss_count` as Feed Outage condition 2's sample-size floor
  — rejected in favor of `min_watchlist_size`; the count-based floor
  distorts the effective percentage threshold near its own boundary, and
  can leave the condition inert below the count regardless of miss rate.
- Bulk pre-registration for investing.com ticker-symbol matching —
  rejected in favor of query-time normalization; bulk kept only for the
  trading API, where a wrong/missing symbol has order-submission stakes
  investing.com's silent-drop failure mode doesn't share.
- A dedicated full-depth (level 2-5) real-time WS quotes feed for bid/ask —
  not designed; a third consumer of the 2-sequence WS lease is a real
  complexity increase not justified given the months-long runway before
  any bid/ask use is possible regardless.
