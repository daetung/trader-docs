# Ops Doc: External API Contract Checklist

**No corresponding source file — a verification procedure and a record of what
was measured, not a code module.**

---

## Role

Every external assumption this system depends on, in one place: what we assume,
which spec breaks if the assumption is wrong, how badly, how to check it, and
where the measured value gets written.

The assumptions are load-bearing and unverified. They were made while designing
against vendor documentation and reasonable expectation, never against the
running services. This document exists so that the set is finite and visible
rather than scattered across a dozen spec files as parenthetical hopes.

**Code never reads this document.** Every value that a module needs lives in a
config key, named in the `config key` column below. This file is the record of
where that value came from and what it means; `pipeline_config.yaml` is what
the code consults. When the two disagree, this file is the account of what was
measured and the config is what is running — reconcile deliberately, do not
assume either is authoritative for the other's purpose.

**This file accumulates; it is not reset per session.** Unlike
`open_items.md`, which holds only what is still unresolved, a verified row is
not deleted — its measured value is filled in and it stays. Switching brokers, or a vendor changing its own
contract, makes the whole table live again, and a row that was deleted on
verification would have to be rediscovered from scratch.

---

## When to use it

- **Before Pilot (Stage 2, `shadow_retraining.md`).** Everything at risk grade
  A must be verified. Grade B should be verified; where it cannot be, the
  fallback path it depends on must be confirmed to work.
- **Continuously, for the self-measuring rows.** Some rows fill themselves in
  from ordinary operation — the retention boundary is probed every session
  start, the broker's rejection vocabulary accumulates through
  `health_report.md`'s findings, and the detection-gap figures accumulate in
  `live_scan_daily`. Those need reading, not measuring.
- **On any broker or vendor change.** Treat every row as unmeasured again.

---

## Risk grades

Graded by **what breaks if the assumption is wrong**, not by how likely it is
to be wrong. A near-certain assumption whose failure forces a redesign
outranks a shaky one that degrades gracefully.

- **A — the design has to change.** A spec's stated reasoning becomes false,
  not just a value becomes wrong.
- **B — safety margin is eaten.** The system keeps working but a guarantee it
  claims is weaker than stated, usually silently.
- **C — degrades gracefully.** A fallback already exists and is specified.

---

## Trading API

| # | Assumption | Consumed by | Grade | How to verify | Config key | Measured |
|---|---|---|---|---|---|---|
| T-1 | The REST tick endpoint returns the same granularity and schema as the WS trade stream | `live_mode_runner.md` Exit Architecture | **A** | Pull the same window from both, compare tick counts and fields | — (no key; a mismatch needs a normalisation layer, not a value) | |
| T-2 | Full-range-since-last-query retention: how far back, and what a first-ever query returns | Warm Restart gap-fill, Feed Outage Recovery | **B** | Self-measuring — the session-start retention probe records the oldest timestamp actually returned | `live_mode.retention_probe.assumed_days` (fallback only) | |
| T-3 | ~~Throughput ceiling, assumed ~100 tickers/sec~~ | ~~Chunked fetches; Eager Pool~~ | **RETIRED** | Nothing to verify — see below | — (key deleted) | **Retired** — premise gone |
| T-4 | Whether `AstkOrdAbleAmt` includes UNSETTLED SELL PROCEEDS — the settled-funds axis, the part of the original cash/buying-power/settled question still open | `session_start_cash` → `compute_position_size()` | **B** | `inquiry/deposit-detail` (CAZCQ01400, TPS 2, no input) returns a D+0..D+4 ladder — `DpsBaseDt0..4` against `AstkDps*`, `AstkUnsttSellAmt*`, `AstkUnsttBuyAmt*`. `AstkUnsttSellAmt0` is the discriminator: when it is nonzero, whether `AstkOrdAbleAmt` tracks `AstkDps0` or `AstkDps0` minus it settles the question from two vendor responses | — (no key; what remains is a FIELD CHOICE, not a value) | |
| T-5 | `Mgnrt0` from `inquiry/able-orderqty` IS the effective requirement, needing no combination — `Mgnrt` is the instrument rate and `OtptItemNm1` a label string, so there is no second RATE to reconcile against | Position sizing — `live_mode_runner.md`'s Per-Ticker Trading Terms | **B** | Call it for a ticker with and without an open position and confirm `Mgnrt0` is what the broker actually applies; the account-level override shows as `Mgnrt0 == 100` rather than as a separate figure | — (key deleted with `margin_ratio_url`) | |
| T-6 | WS connection limits: tickers per connection, connections per account, subscription types per connection, connections per IP per port | Exit Architecture; WS connections | **B** | Subscribe and connect past each limit and observe the failure | `execution.ws_ticker_limit` | **Measured** — 50 tickers per connection; 2 connections per account; ONE subscription type per connection (an account registration sent on a quote connection converts it and silently stops quote delivery); 30 connections per IP per port. Confirmed against production AND demo accounts |
| T-7 | ~~Account-wide fill event stream: whether individual fills carry a stable, unique ID; schema; heartbeat; reconnect; whether events missed while disconnected are replayed~~ | ~~In-flight order tracking (fill accounting invariant)~~ | **RETIRED** | Nothing to verify — see below | — | **Retired** — superseded by the order and fill ledger; residual question in T-21 |
| T-8 | ~~A REST order-status endpoint suitable as the exit-fill backstop, and whether it returns a given order's COMPLETE fill history or a paginated/windowed slice~~ | ~~In-flight order tracking (exits are REST-only)~~ | **RETIRED** | Nothing to verify — see below | — | **Retired** — superseded by fill folding's completeness condition |
| T-9 | Subscribe acknowledgement latency | Exit Architecture's post-subscribe REST gap-fill window | **C** | Time the round trip under load | — (no key; the gap-fill covers the window rather than a configured value sizing it) | |
| T-10 | The broker's rejection reason vocabulary | `trade_log.reject_reason`; any future normalisation | **C** | Self-measuring — `health_report.md`'s unrecognised-reason finding accumulates it | — (stored verbatim; no enum until this is known) | |
| T-11 | Whether a server-clock endpoint exists | `clock_check.source: "vendor_api"` | **C** | Endpoint documentation | `live_mode.clock_check.source` | |
| T-12 | Whether a resting order survives a trading halt on its ticker, is auto-canceled by the halt, or executes at the halt-resumption cross/auction | `live_mode_runner.md` Position Manager Loop — halt-clear handling for an in-flight exit order | **C** | Self-measuring — `health_report.md` finding 25 records every case where an in-flight exit order was found gone at halt-clear | — (no key; the design branches at halt-clear regardless of the answer — see below) | |
| T-13 | How long after a minute closes the trading API's bar endpoint typically has that minute's bar ready | `live_mode_runner.md` Bar-Close Authority (Feed Outage trigger condition 2) | **B** | Self-measuring — `health_report.md` finding 26 accumulates the observed bar-arrival latency as a cumulative curve (whole-second buckets), so the value is read from ordinary operation rather than measured in a separate exercise | `live_mode.bar_close_grace_seconds` | |
| T-14 | ~~Bar endpoint resolution and page ceiling~~ | ~~Prefix-scan sizing~~ | **RETIRED** | Nothing to verify — measured, then transcribed | — | **Retired** — the values now live in `trading_api.md`'s Call-Point Inventory |
| T-15 | ~~Demo/production data equivalence and budget independence~~ | ~~Combined two-account budget~~ | **RETIRED** | Nothing to verify — measured, then transcribed | — | **Retired** — equivalence recorded in `trading_api.md`'s Call-Point Inventory; per-APP governance in `sdk_dependency.md` |
| T-16 | ~~REST round-trip latency range~~ | ~~Prefix-scan slot allocation~~ | **RETIRED** | Nothing to verify — measured, then transcribed | — | **Retired** — 400-600ms recorded in `live_mode_runner.md`'s `scan:` keys; ongoing drift is `health_report.md`'s finding 32, not a checklist question |
| T-17 | The watchdog list fires for every ticker that crosses conditions A-G, so a ticker becoming eligible always rises to the head | `live_mode_runner.md` prefix scan — the soundness of stopping where bar delta stops | **B** | Self-measuring — the `evening_detection_gap` stage runs `detect()` over the day's ingested bars and compares against `inference_log` | — (no key; the rotation cursor and the promotion path bound the gap rather than a value sizing it) | |
| T-18 | `Mgnrt0` is INTRADAY-INVARIANT, so a rate acquired at watchdog first listing stands for the session | `live_mode_runner.md`'s Per-Ticker Trading Terms — acquisition once per ticker, and the rate pinned onto each `live_positions` row | **B** | Self-measuring at zero cost — the post-dispatch entry observation already calls `able-orderqty` and its response carries `Mgnrt0` beside `AstkOrdAbleQty`, so comparing it against the PERSISTED `live_ticker_terms` row needs no extra call. A mismatch raises the same warning. Comparing against the persisted row rather than an in-memory cache is what lets the measurement survive a crash and a warm restart, which is exactly when a rate change would be least visible | — (no key; a rate that moves needs a re-acquisition rule, not a value) | |
| T-19 | Whether a resting order survives the session-close / after-hours boundary and stays live overnight, or is canceled by the venue at that boundary | `live_mode_runner.md`'s Session Shutdown cancellation, the in-flight exit loop's amend reasoning, and the vanished-order rule's venue-cancellation cause | **B** | Self-measuring — the startup procedure classifies each 'open' logical order carried from a prior day, recording `live_orders.terminal_cause`: `carried_live_canceled` (still live at the broker) against `expired_at_boundary` (gone). A session that ended without Session Shutdown's cancels is the observation | — (no key; the answer settles which statements may assert, not a value) | |
| T-20 | Which order number is live after an amend is confirmed — the original or the amend's | `live_mode_runner.md`'s IS2 routing (`current_request_id`) and the exit ladder's amends | **C** | Amend a resting order; compare the OUTSTANDING list's order numbers before and after | — | |
| T-21 | Each execution is reported under exactly one order number, and the uniqueness scope of `ExecNo` | `live_fills`' primary key and fill folding (`live_mode_runner.md`) | **A** | Amend a partly filled order and fill it further; compare the REST itemised fills of both order numbers and their `ExecNo` values | — | |
| T-22 | The `AstkOrdStatCode` code set | `live_order_requests.broker_status` (stored raw) | **C** | Record the codes observed across new, amend, cancel, fill and rejection | — (no key; classification uses quantities and IS2 event types, not the code) | |
| T-23 | Whether a prior-day order can be cancelled with `OrgOrdNo` alone | `live_mode_runner.md`'s startup procedure — `carried_live_canceled` | **B** | Cancel an order carried from a prior day | — | |
| T-24 | Whether an IS2 event can precede the order API response | `live_mode_runner.md`'s IS2 routing (unresolved new-order events) | **C** | Compare IS2 receipt times with order-response receipt times | — | |
| T-25 | Whether `AstkExecBaseQty` reflects same-day executions immediately | `live_mode_runner.md`'s Positions branch comparison | **A** | Query `inquiry/balance-margin` immediately after a fill and compare with `live_fills` | — | |
| T-26 | When the broker reflects a split or reverse split in `AstkExecBaseQty` and `AstkAvrPchsPrc` relative to the effective date, and whether a sell is refused meanwhile | `live_mode_runner.md`'s event-day exit gate and Positions branch | **B** | Observe a holding across a split's effective date | `live_mode.unrecorded_event_detectors` | |
| T-27 | How the broker disposes of a split's or reverse split's fractional share — cash in lieu, rounding up, or a fractional balance — the `SmryNm` of its cash and movement rows, whether its cash row carries `AstkIsuNo`, whether it also appears as a sell in `inquiry/trading-history` or `inquiry/transaction-history`, its lag from `event_date`, and whether `QryTpCode` '1' and '2' exclude trade rows | `live_mode_runner.md`'s Positions branch event settlement, fill folding's settlement exclusion, broker measurement and summary vocabulary | **C** | Self-measuring — `health_report.md`'s `summary_vocabulary_candidate` finding | `live_mode.summary_match_mode`, `live_mode.cash_in_lieu_summary_names`, `live_mode.split_movement_summary_names`, `live_mode.cash_in_lieu_sources`, `live_mode.trade_summary_words`, `live_mode.summary_direction_words`, `live_mode.trade_history_query_mode` | **Partial** — `inquiry/trade-history` accepts an empty `AstkIsuNo` and returns the whole account; every deposit/withdrawal and in/out-transfer `SmryNm` contains '입금', '출금', '입고' or '출고'; trade rows carry '매수' or '매도' (e.g. '주식매수대금출금(외화)', '주식매도출고(외화)') |
| T-28 | Under `DpntBalTpCode='0'`, whether `inquiry/balance-margin` returns one row per ticker or one per balance type, and whether `AstkExecBaseQty` includes the fractional balance | `live_mode_runner.md`'s broker measurement | **C** | Query an account holding both a whole and a fractional balance under each `DpntBalTpCode`; fallback: the 'split' default | `live_mode.balance_query_mode` | |
| T-29 | Whether the order API sells a fractional balance with a fractional `AstkOrdQty` | `live_mode_runner.md`'s settlement pending — `event_settlement_stalled` `kind='fraction_held'` | **C** | Submit a sell for a fractional balance | — | **Partial** — the account holds fractional-share buy records ('소수점주식매수입고(외화)') |
| T-30 | Whether each `corporate_events` source restates a dividend `value` by later splits, and how a re-crawl that returns a restated amount is upserted | `utils.md`'s dividend amount function; `metadata_crawler.md`'s upsert | **C** | Compare a dividend across a later split in each source; fallback: the key's default | `corporate_events.dividend_restated_sources` | |
| T-31 | Whether any non-integer vendor numeric field arrives as a JSON number rather than a string | `trading_api.md`'s Response Normalization; `live_mode_runner.md`'s broker measurement | **C** | Inspect raw responses; fallback: a float is converted through `repr` | — | |
| T-32 | Whether the order API accepts `AstkOrdQty` and `AstkOrdPrc` as JSON numbers carrying up to 6 decimal places, and as strings | `trading_api.md`'s order request numeric format | **C** | Submit orders in each format; fallback: the 'number' default | `trading.order_numeric_format` | |
| T-33 | The dividend withholding rate applied to this account, and the `SmryNm` of dividend cash and tax rows | `utils.md`'s dividend amount function; `live_mode_runner.md`'s dividend withholding check | **C** | Self-measuring — `health_report.md`'s `dividend_withholding_mismatch` and `summary_vocabulary_candidate` findings | `corporate_events.dividend_withholding_rate`, `corporate_events.dividend_cash_summary_names`, `corporate_events.dividend_tax_summary_names` | **Partial** — `inquiry/trade-history` shows '배당금입금(외화)' and '배당세출금(외화)' |

**T-3 is RETIRED, not verified.** Its premise was that a real-world
throughput ceiling had to be measured because none was published. `api_doc/`
publishes per-endpoint TPS and the vendored SDK carries them as a table it
paces against automatically, so there is nothing left to measure. Client-side
chunking moved behind `trading_api.md` with the rest of transport, and its
config key was deleted rather than superseded. The row stays because this file
does not delete rows — a broker change would make the question live again, and
a deleted row would have to be rediscovered.

**T-5 was re-anchored TWICE, and the second time removed its premise.** The
original row assumed an account-level margin-ratio endpoint; the vendor
catalogue publishes none, so it was re-anchored to the per-ticker figure from
`inquiry/able-orderqty`, still asking how that figure COMBINES with an
account-level one. Measurement then showed there is no account-level RATE to
combine with: `OtptItemNm1` is a label string — "100% 계좌" | "컬러증거금" —
reporting whether a per-ticker 100% override applies, not a number, and
`Mgnrt0` already incorporates that override. The redesign that followed is
DONE, not deferred, so the row no longer points at an open item. What
survives is narrower and is what the row now asks: whether `Mgnrt0` is in
fact the rate the broker applies.

**T-4 lost two thirds of its question to the same measurement.** The
identity `AstkOrdAbleAmt * (100 / Mgnrt0) - @cost = AstkOrdAbleAmt1` fixes
`AstkOrdAbleAmt` as the PRE-LEVERAGE deposit and `AstkOrdAbleAmt1` as buying
power, which dissolves the cash-versus-buying-power dichotomy — both are
separate fields the design now names outright, so `execution.sizing_basis`
had nothing left to select between and was deleted. The settled-funds axis is
untouched, and it matters in practice rather than in principle: this strategy
turns over intraday, so unsettled sell proceeds are present continuously. The
row is re-anchored to that axis rather than retired, this file deleting no
rows.

**T-18 is not a value question.** If `Mgnrt0` proves to move intraday, no
number needs setting — the acquisition rule does, since a rate cached at
first listing and pinned at entry would then be stale by construction. It is
graded B rather than A because the design already survives a stale rate
one-sidedly: `live_positions.entry_mgnrt` records what each entry actually
used, so a moved rate makes past sizing suboptimal rather than
unreconstructable.

**T-1 acquired a method, and is not thereby closer to closed.** The delayed
quote stream (V10/V11) carries the COMPLETE tape without a separate
application, against the free real-time stream's roughly 50% of prints, so
comparing the two answers the granularity question with an instrument rather
than an invented exercise — `auxiliary_stream.md` builds it, now as an
in-process component of LiveModeRunner rather than a separate process. That
binds collection to the runner's lifetime: on a trading day where the runner
does not start, no delayed-side data is gathered. Accepted — the live side is
not written either, so the comparison would not be computable regardless, and
the coverage stage's verdict table already treats that as FAILURE. The row
stays
grade A and stays a Pilot precondition: having a method is not having a result.

**T-12 is deliberately answer-agnostic.** The halt-clear handling for an
in-flight exit order re-queries that order's status the instant the halt
clears, rather than assuming any of the three outcomes above — so it stays
correct whichever one turns out to be true. Verifying this row does not
unblock anything; it only allows removing the immediate re-query as a
now-provably-unnecessary step, should finding 25 accumulate enough
halt-clear events with zero disappearances to trust "always survives."

**T-13's failure direction is asymmetric.** A `bar_close_grace_seconds`
set shorter than the vendor's real typical latency makes Bar-Close
Authority over-count ordinary lag as a "missed deadline" — which, combined
with `min_watchlist_size`, risks a spurious Feed Outage freeze under
normal operation. Set longer than necessary, it only slows genuine-outage
detection. The seed value is chosen accordingly: conservative (higher)
until measured, the same one-sided-error posture this spec set takes
wherever a bound stands in for an unknown.

**T-14, T-15 and T-16 were opened and closed inside one session, which is
why they RETIRE rather than sit here as measured rows.** This file is a
check-ops queue: a row earns its place by naming work still to be done, and
once the check has run and its result is transcribed into the spec that
depends on it, that spec is the record and the row's job is over. T-6 is not
an inconsistency with this — it was carried in from an earlier session where
the measurement and the transcription did not coincide, and it stays
Measured as recorded there.

Each retirement note names WHERE the value went, which is what keeps the
retirement reversible: on a broker change, re-open the row and re-check
against that location rather than rediscovering it.

**T-17 is an accepted risk, recorded rather than resolved.** If the watchdog's
own firing condition is not a superset of A-G, a ticker can cross our
conditions without rising to the head, and the prefix scan's early stop will
not reach it. This was accepted deliberately: the loss is bounded by the
rotation cursor, which walks the whole list about once per bar, and by the
promotion path, which recovers a crossing found on an already-completed bar.
What makes acceptance defensible is that the residual is MEASURED rather than
assumed small — the evening stage compares an independent ground truth against
what live actually evaluated. Note the row's own limit: the rotation cursor
walks the LIST, so it can only find a ticker the watchdog listed at some point.
A ticker that never appears at all is invisible in-session and shows up only in
the evening comparison.

**T-6 was re-anchored, not merely measured.** Its original phrasing
described a "WS sequence lease" that no longer exists: subscription types
turn out to be mutually exclusive per connection, so the two connections
this system opens are both held for the whole session and nothing is leased,
shared or preempted. The row now records the connection limits themselves,
which is what the design actually rests on.

**T-9 lost its config key rather than its purpose.** It originally sized
`fill_stream_linger_seconds`, a key deleted with the lease. Subscribe
acknowledgement latency still has a live consumer — Exit Architecture
REST-gap-fills the window between the subscribe call and the subscription
going active — so the row stays, re-anchored, with the key column cleared.

**No row exists for market-sell conversion, deliberately.** The vendor
converts a market BUY into an unfavourably-priced limit; whether it does the
same to a market SELL was an open question, and it is CLOSED from vendor
documentation rather than deferred here: the rule is stated for buys only
and its stated reason (the reference price is set high, shrinking orderable
quantity) is buy-specific. Recorded so the question is not re-raised as a
gap.

**T-7 and T-8 are RETIRED.** The order and fill ledger (`db_schema.md`'s
`live_fills`; `live_mode_runner.md`'s fill folding) removed the mechanism
they backed. T-7's sub-items:
- 1 (a stable per-fill ID) — superseded by `live_fills`' `cum_after_qty` key
  and the REST itemised view's `ExecNo`
- 2 (an order's complete fill history) — retired with T-8
- 3 (one ID scheme across WS and REST) — answered from vendor documentation:
  WS IS2 carries no fill ID
- 4 (replay after reconnect) — covered by the REST backstop
- 5 (cumulative fields for a fallback) — moot, no fallback remains

Its residual question is T-21. T-8 is superseded by fill folding's
completeness condition: a REST itemised result replaces an order number's
rows only when its Σ `exec_qty` equals the broker cumulative for that order
number. The fill inquiry is account-wide rather than per-order, which is why
`live_mode_runner.md`'s exit backstop is one pair of scoped calls per cycle
rather than one call per outstanding order.

**T-1, T-21 and T-25 are the grade A rows.** Each backs a stated correctness
guarantee rather than a tunable value — a wrong answer means the design
itself is wrong, not just a config default. T-14 and T-15 were graded A while
open, but both retired within the session that raised them.

**T-1**: Exit Architecture states that WS and REST share one parser and one
2-print guard, and therefore that the two exit paths have no filter
asymmetry. If the granularities differ, that claim is false as written — a
normalisation layer is needed and the guard has to be re-derived for each
path. Verify this first.

**T-4 and T-5 WERE one question in two parts, and are no longer.** They were
one only while the balance figure's meaning was unknown: full deployment being
the intended operating point, reading buying power as cash would have
multiplied intended exposure by the leverage factor, and `sizing_basis`
existed to absorb whichever answer came back. Measurement named both fields
outright — `AstkOrdAbleAmt` the deposit, `AstkOrdAbleAmt1` the buying power —
so nothing is left to absorb and the key is deleted. The two rows now ask
about DIFFERENT axes: T-4 whether the deposit figure carries unsettled sell
proceeds, T-5 whether `Mgnrt0` is the rate the broker actually applies.
Neither answer constrains the other, and neither is a precondition for
reading the other's result.

---

## yfinance

| # | Assumption | Consumed by | Grade | How to verify | Config key | Measured |
|---|---|---|---|---|---|---|
| Y-1 | Split, reverse-split and dividend history is complete and timely enough for same-day corporate-event detection | `metadata_crawler.md`; `cum_split_ratio()` | **C** | Cross-check against investing.com — this is what the second vendor is for | — | |

---

## investing.com

| # | Assumption | Consumed by | Grade | How to verify | Config key | Measured |
|---|---|---|---|---|---|---|
| I-1 | Scraping the corporate-events calendar pages is permitted by the terms of service | `metadata_crawler.md` forward check | **C** | Read the ToS | — | |
| I-2 | The crawler's symbol extraction from the calendar's company cell, followed by `normalize_vendor_symbol()` folding, reaches an acceptable match rate against `active_ticker_universe` | Forward-check row matching | **C** | Self-measuring — `health_report.md`'s investing.com match-rate finding | — | |
| I-3 | Which vendor is right where investing.com and yfinance disagree about an event whose effective date has passed | `metadata_crawler.md`'s `upsert_corporate_event()` disagreement branch | **C** | Self-measuring — `corporate_event_conflicts` holds both values with `observed_at`, surfaced by `health_report.md` | — | |

**I-2 is deliberately observable rather than solved.** Matching now applies
query-time normalization (`metadata_crawler.md`'s
`crawl_corporate_events_investing()`, through `utils.md`'s
`normalize_vendor_symbol()`) rather than a naive exact match, but the
mismatch-rate finding stays in place — normalization is best-effort, not a
guarantee, so the residual gap remains worth tracking. That finding
(`health_report.md` finding 10) now reports PER SOURCE; the branch that
verifies this row is its investing.com tally, not the finding as a whole,
since a halt-feed mismatch says nothing about forward-check row matching.

---

## Constraints

- Grade reflects blast radius, not probability — do not re-rank on how likely
  a vendor is to surprise us
- A row is verified only against the environment that will actually run:
  paper-trading endpoints do not settle a production question
- Verified rows keep their measured value; nothing here is deleted on
  verification
- Where a row names a config key, the measured value belongs in
  `pipeline_config.yaml` under that key — recording it only in this table
  leaves the code running on its default
- The self-measuring rows (T-2, T-10, T-12, T-13, T-17, T-18, T-19, T-27, T-33, I-2,
  I-3) are
  filled in during ordinary operation and do not need a separate measurement
  exercise. Most accumulate through `health_report.md` findings; T-17
  accumulates in `live_scan_daily` instead, by way of the evening
  detection-gap stage, which `health_report.md` then reads. T-13 remains
  the only one of these that names a config key while graded above C —
  self-measuring describes how the value is obtained, not what its being
  wrong costs, so it does not lower a grade
- A grade-A row that cannot be verified before Pilot is a reason not to enter
  Pilot, not a reason to assume in its favour
