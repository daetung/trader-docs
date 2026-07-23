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
`open_items_session*.md`, a verified row is not deleted — its measured value is
filled in and it stays. Switching brokers, or a vendor changing its own
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
  `health_report.md`'s findings. Those need reading, not measuring.
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
| T-3 | Throughput ceiling, assumed ~100 tickers/sec | Chunked fetches; Eager Pool | **B** | Ramp until rate limiting appears; measure both the 09:20–09:25 overlap and steady state | `live_mode.api_max_tickers_per_second` | |
| T-4 | What the balance query returns: cash, buying power, or settled funds | `session_start_cash` → `compute_position_size()` | **B** | Compare the returned figure against the broker's own statement | `execution.sizing_basis` selects the interpretation | |
| T-5 | A margin-ratio endpoint exists and returns the current requirement | Session Start Probes → `sizing_basis: "equity"` | **B** | Endpoint documentation; confirm the value moves with position state | `live_mode.margin_ratio_url` | |
| T-6 | One WS sequence carries up to 50 tickers, and at most 2 sequences may be open | Exit Architecture; WS sequence lease | **B** | Subscribe past both limits and observe the failure | `execution.ws_ticker_limit` | |
| T-7 | Account-wide fill event stream: whether individual fills carry a stable, unique ID; schema; heartbeat; reconnect; whether events missed while disconnected are replayed | In-flight order tracking (fill accounting invariant) | **A** | See the fill-accounting sub-items below the table | — | |
| T-8 | A REST order-status endpoint suitable as the exit-fill backstop, and whether it returns a given order's COMPLETE fill history or a paginated/windowed slice (shared checkpoint with T-7 — see below) | In-flight order tracking (exits are REST-only) | **C** | Endpoint documentation; poll a known order; check for pagination on an order with many fills | — | |
| T-9 | Subscribe/unsubscribe acknowledgement latency | `fill_stream_linger_seconds` default | **C** | Time the round trip under load | `live_mode.fill_stream_linger_seconds` | |
| T-10 | The broker's rejection reason vocabulary | `trade_log.reject_reason`; any future normalisation | **C** | Self-measuring — `health_report.md`'s unrecognised-reason finding accumulates it | — (stored verbatim; no enum until this is known) | |
| T-11 | Whether a server-clock endpoint exists | `clock_check.source: "vendor_api"` | **C** | Endpoint documentation | `live_mode.clock_check.source` | |
| T-12 | Whether a resting order survives a trading halt on its ticker, is auto-canceled by the halt, or executes at the halt-resumption cross/auction | `live_mode_runner.md` Position Manager Loop — halt-clear handling for an in-flight exit order | **C** | Self-measuring — `health_report.md` finding 25 records every case where an in-flight exit order was found gone at halt-clear | — (no key; the design branches at halt-clear regardless of the answer — see below) | |
| T-13 | How long after a minute closes the trading API's bar endpoint typically has that minute's bar ready | `live_mode_runner.md` Bar-Close Authority (Feed Outage trigger condition 2) | **B** | Observe actual bar arrival latency relative to minute-close under normal operation | `live_mode.bar_close_grace_seconds` | |

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
until measured, the same one-sided-error posture as `margin_ratio_fallback`.

**T-1 and T-7 are the grade A rows.** Both back a stated correctness
guarantee rather than a tunable value — a wrong answer means the design
itself is wrong, not just a config default.

**T-1**: Exit Architecture states that WS and REST share one parser and one
2-print guard, and therefore that the two exit paths have no filter
asymmetry. If the granularities differ, that claim is false as written — a
normalisation layer is needed and the guard has to be re-derived for each
path. Verify this first.

**T-7's sub-items**, in the order they should be checked (see
live_mode_runner.md's fill accounting invariant for why each matters):
1. Does each individual fill carry a stable, unique ID that survives
   reconnects and REST re-queries? This decides whether the PRIMARY
   fill-tracking mechanism (fill-ID set union) is available at all — without
   it, tracking falls back to cumulative-field assignment, a materially
   weaker guarantee.
2. Does T-8's REST endpoint return an order's COMPLETE fill history, or can
   it be paginated/windowed? (Shared with T-8 above — this is what
   determines whether a Warm Restart can rebuild `seen_fills` from one REST
   query.)
3. Do the WS stream and the REST endpoint use the SAME fill-ID scheme?
   Two different ID spaces for the same underlying fills would make
   deduplication silently fail across channels while appearing to work
   within each one.
4. On reconnect, are fills missed while disconnected replayed, or lost?
5. (Needed only if item 1 is No) Does the API expose a cumulative filled
   quantity and cumulative average price per order, usable by the fallback
   mechanism?

A wrong answer to 1 or 3 does not fail loudly — it produces occasional
double-counted fills, which live_mode_runner.md's monotonic-non-decrease
guard catches only sometimes (it rejects a DECREASE, not every
over-count). This is why T-7 is graded on the same tier as T-1 rather than
on the strength of its own fallback: the fallback exists so a No to item 1
degrades to a still-correct mechanism, not so item 1 can go unverified.

**T-4 and T-5 are one question in two parts.** Full deployment is the intended
operating point, so if the balance figure is buying power and it is read as
cash, intended exposure is multiplied by the leverage factor. `sizing_basis`
exists to absorb either answer, but it can only be set correctly once the
answer is known.

---

## yfinance

| # | Assumption | Consumed by | Grade | How to verify | Config key | Measured |
|---|---|---|---|---|---|---|
| Y-1 | Split, reverse-split and dividend history is complete and timely enough for same-day corporate-event detection | `metadata_crawler.md`; `cum_split_ratio()` | **C** | Cross-check against investing.com — this is what the second vendor is for | — | |

---

## investing.com

| # | Assumption | Consumed by | Grade | How to verify | Config key | Measured |
|---|---|---|---|---|---|---|
| I-1 | Corporate-events calendar pages: exact endpoints, and scraping is permitted by the terms of service | `metadata_crawler.md` forward check | **C** | Read the ToS; confirm page structure | — | |
| I-2 | Symbol format matches `active_ticker_universe` closely enough for naive matching | Forward-check row matching | **C** | Self-measuring — `health_report.md`'s investing.com match-rate finding | — | |

**I-2 is deliberately observable rather than solved.** Matching now applies
query-time normalization (`metadata_crawler.md`'s
`crawl_corporate_events_investing()`, the same rules as ticker-rename
detection) rather than a naive exact match, but the match-rate finding
(`health_report.md` finding 10) stays in place — normalization is
best-effort, not a guarantee, so the residual gap remains worth tracking.

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
- The self-measuring rows (T-2, T-10, T-12, I-2) are filled in by
  `health_report.md` findings during ordinary operation, and do not need a
  separate measurement exercise
- A grade-A row that cannot be verified before Pilot is a reason not to enter
  Pilot, not a reason to assume in its favour
