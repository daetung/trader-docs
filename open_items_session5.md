# Open Items — Session 5

**Reference/tracking document — same tier as `session_handoff.md`, not a spec
file.** Supersedes `open_items_session4.md` in full: R-4 (circuit breaker),
R-5 (entry gates), R-6 (shared config params), R-7 (order-rejection spec —
its staging-measurement checklist is superseded by the new permanent spec
file `docs/ops/api_contract_checklist.md`, not carried here), R-8 (three
operational notes), and the finding-2 dropped-column bug are all resolved
this session. See the patch set: `gate_result_and_gate_counters_DELTA.patch`,
`r4_r7_r8_breaker_probes_contracts_DELTA2.patch`,
`rename_and_gap_fixes_DELTA3.patch` (plus the base-layer patches ahead of
those). This file may be removed once its own items are resolved or
superseded, same as `open_items_session4.md` now is.

This file holds **only what is not yet reviewed or resolved.** Descriptions
below are problem statements and first-pass thinking, not confirmed designs
— re-review from the problem each time.

---

## Suggested order

1. **Stuck exit-order policy** and **account fill-stream disconnect** are
   related (both touch the fill-tracking / connection-health surface this
   session just finished) — natural to take together.
2. **Ticker-symbol normalization** is independent and has been open across
   two sessions now; the longer it sits, the more silent-drop risk
   `active_ticker_universe` naive matching carries.
3. **Rate-limit reconciliation** and **bid/ask spread** are both explicitly
   blocked on external input (the API spec sheet; a data-collection
   decision) — not blocked on design time, so no urgency to sequence them
   ahead of the above.

---

## Stuck exit-order policy: forced liquidation / re-submission

**Problem.** This session's fill-tracking design confirmed exits have no
give-up timeout by design — an unsold remainder is still exposed to the risk
that triggered the exit, so the order stays tracked until fully filled (see
`live_mode_runner.md`'s In-flight order tracking). On a genuinely illiquid
name this could in principle stay open indefinitely. `health_report.md`
finding 18 (`live_mode.exit_order_stuck_minutes`, default 10) makes this
observable, but no ACTION policy exists for what happens once it fires —
today it is a standing alert with no automated response.

**Deferred deliberately** during this session's WS-lease design, on the
reasoning that its natural trigger condition ("exit order open > N minutes,
unfilled") is the same condition finding 18 already evaluates, so the two
should be designed together rather than the trigger being invented twice.

**Not yet designed.** Candidate directions, none evaluated: escalate to a
more aggressive order type (market, if the original was limit); cancel and
re-submit at a wider price; page an operator for manual intervention; do
nothing beyond the existing alert (accept the standing exposure as a known,
monitored risk). The right choice likely depends on position size and how
thin the name is, neither of which this system currently classifies.

---

## Account fill-stream disconnect vs. Feed Outage trigger conditions

**Problem.** This session's fill-tracking design made the account-wide fill
event WS stream a live dependency for both entry confirmation and exit
completion. Feed Outage Recovery's trigger is currently two conditions: an
explicit connection/API-level failure, or >50% of the watchlist missing its
bar-close deadline in the same minute. Whether the fill stream's own
disconnection is already covered by the first condition, or needs to be
named as an explicit third trigger, was left open during this session's
design — the fill stream and the price-tick stream are different endpoints
per the WS sequence lease, so a failure isolated to one is not obviously
implied by the other.

**Not yet designed.** If treated as already covered, that should be stated
explicitly rather than left to inference. If it needs its own trigger, the
freeze/recovery scope (does a fill-stream-only outage need to freeze entries
AND exits, the same as today's Feed Outage, or is a narrower scope
correct — e.g. entries only, since exits already have a REST backstop and no
give-up timeout) is the open design question.

---

## Real-time bid/ask spread — auxiliary signal design

**Problem.** Intended use, per this session's discussion: an auxiliary
signal for bid/ask spread estimation — not a primary model feature. Two
distinct paths were distinguished and neither is designed:

- **As a model feature.** Blocked, not merely undesigned: no historical
  bid/ask data is collected anywhere in the DB (`tick_10` is trade data, not
  quotes), so it cannot be computed for training data — a feature that
  cannot be computed historically cannot be added without a preceding
  data-collection project (new snapshot table, months of accumulation,
  retraining). This path is closed until that project exists.
- **As an execution-time gate** (e.g., skip or delay an entry when spread
  exceeds a threshold). Feasible without new data collection — same
  category as `is_tradable()`, a runtime filter rather than a feature — but
  BacktestEngine cannot replay it, since backtest has no quote data either.
  That creates a live/backtest entry-count divergence larger than the
  micro-divergences already accepted elsewhere (those affect sizing or
  timing at the margin; this would change which candidates are entered at
  all).

**Also unresolved from this session's discussion:** processing model —
quotes arrive as a continuous stream (potentially many events/second), not
bar-aligned, so they do not fit `CachingCalculator`'s `on_bar_close()` model
without a separate buffering/aggregation layer.

**Not yet designed.** Which path (if either) to pursue, and — if the
execution-gate path — how to represent or bound the resulting backtest
divergence rather than let it go unremarked.

---

## Rate-limit reconciliation with the API spec sheet

**Problem.** Deferred pending integration of an external API spec sheet
(not yet provided to this process). Known so far: a **global 20
transactions/second** limit across the trading API; per-endpoint limits are
in the spec sheet, not yet available. Logged provisionally in
`docs/ops/api_contract_checklist.md`'s T-3 row.

**Specific tension flagged this session, to revisit once the sheet is
available.** `position_check_interval_seconds` (5s), `max_tickers` (10) ×
`max_positions_per_ticker` (2) = up to 20 concurrent positions, and the 20
TPS global cap may not be independent: a full book's Position Manager Loop
issues on the order of 20 near-simultaneous REST calls at the start of each
5-second cycle, before adding entry-time balance/margin queries, halt
checks, and exit-fill polling landing in the same window. Whether these
three values need to be jointly derived (rather than each defaulted
independently, as they were this session) is the open question. Also
flagged: an exit burst immediately after a Feed Outage thaw (all
`track_price_breach()`-confirmed exits accumulated during the freeze submit
in one cycle) as a scenario worth measuring alongside the existing
09:20–09:25 and steady-state throughput checks — already added to T-3's
"how to verify," not a separate item.

**Not yet designed** — intentionally; this needs the spec sheet's
per-endpoint numbers before a real design is possible. Revisit at spec-sheet
integration, not before.

---

## Ticker-symbol normalization

**Carried forward from `open_items_session4.md`, untouched across two
sessions now.** Discovered while wiring investing.com as a second
corporate-events vendor (item N); not addressed this session beyond
referencing it from `docs/ops/api_contract_checklist.md`'s I-2 row.

**Problem.** Symbology may differ across every external source: yfinance,
investing.com, and the trading API itself (bars/ticks included) need not
match. Item N ships with naive `active_ticker_universe` matching as an
interim, so an investing.com row for a differently-formatted symbol can
silently fail to match and be dropped — a false "no event." A unified
normalization layer (canonical internal symbol ↔ per-vendor symbol) is
needed. `health_report.md` finding 10 (investing.com match-failure rate) and
`api_contract_checklist.md`'s I-2 row both exist specifically to keep this
gap observable until the normalization layer is designed — neither closes
it.

**Not yet designed.** Scope, canonical-form choice, and where normalization
sits relative to `ticker_cik_map` are all open.

---

## `api_contract_checklist.md` — verify before Pilot

Not a new design problem — a pointer, so it isn't lost among the items
above. `docs/ops/api_contract_checklist.md` (new this session) holds 14
unverified vendor-contract assumptions, two graded **A** (T-1: REST/WS tick
granularity; T-7: fill-event stream ID stability). Both must be verified, or
their fallback paths confirmed sufficient, before Stage 2 (Pilot) — see that
file's own "When to use it" section. Not duplicated here; consult that file
directly rather than letting a second copy of its contents drift out of
sync.

---

## Deliberately settled (do not re-open)

Recorded so they aren't re-derived. Carried forward from
`open_items_session4.md` plus this session's additions:

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
- WS sequence lease default — `shared`, not `dedicated` (R-4/R-7). There is
  no second consumer yet, so time-division has zero current benefit; the
  lease mechanism exists so a future consumer (e.g. quote monitoring) has
  somewhere to plug in without a redesign, not because sharing is needed
  today.
- Exit fill tracking — REST-only, not leased onto the WS sequence. Order
  state is queryable (not a perishable event, unlike a price tick), and
  knowing an exit filled sooner than the next REST poll changes no
  decision, so the latency cost of skipping WS for exits is accepted.
- Real-time bid/ask spread as a **model feature** specifically — rejected
  for now (see the open item above for the execution-gate alternative,
  which is NOT rejected, only undesigned). The blocker is data collection,
  not judgment about the signal's value, so this is not permanently
  settled — revisit if a bid/ask collection project is ever undertaken for
  other reasons.
