# Module: LiveModeRunner

**File:** `src/inference/live_mode_runner.py`

---

## Role

Execution orchestrator for live trading mode.
Manages two independent loops (watchdog polling and position monitoring),
coordinates session initialization, and integrates all live-mode components.

LiveModeRunner is the **execution subject** — Inferencer, CachingIndicatorCalculator,
and all other live-mode modules are service objects instantiated and owned by this class.

---

## Architecture

```
LiveModeRunner
    ├── calculators: dict[str, CachingIndicatorCalculator]
    │       One instance per ticker in today's tradable universe.
    │       Populated eagerly at session_start via parallel session_start_compute().
    │       Active watchlist populated lazily on a ticker's first appearance
    │       in the watchdog working set.
    ├── FeatureExtractor              (uses CachingIndicatorCalculator via DI)
    ├── Inferencer                    (owns FeatureExtractor)
    ├── EntryPointDetector            (used internally by Inferencer)
    ├── [Watchdog polling loop]       (async / threaded)
    └── [Position manager loop]       (async / threaded)
```

---

## DB Connection Management (P-5)

Two connections, not one — this is what makes the ad-hoc-operator-query and
premarket-cron contention problem tractable without new infrastructure:

```python
read_only_conn:  opened with read_only=True — used for every SELECT this
                 session issues (Session Lifecycle bulk loads, any other
                 read). Permits concurrent read-only connections from
                 elsewhere (ad-hoc operator queries) without contention.
write_lock = Lock()
writer_conn:     single connection, all writes funneled through it —
                 inference_log, indicator_cache (any mode as of R-2 — see
                 below; previously "db" mode only), trade_log (shadow
                 mode), batch_runs (this session's own live_session_start
                 / live_session_end markers — see Session Lifecycle Step
                 1c and Session Shutdown below), and, as of R-2,
                 live_positions and live_session_state (see "Session
                 Restart (Warm Start)" below) plus the In-Process
                 Premarket Recheck task's writes (R-1, see that section —
                 same funnel, no second writer).

def db_write(sql, params):
    with write_lock:
        writer_conn.execute(sql, params)
    # synchronous — returns only after commit. This is what makes
    # read-your-own-write correctness free: any db_read() called after
    # a db_write() returns is guaranteed to see it, with no queueing or
    # ordering logic needed.

def db_read(sql, params):
    return read_only_conn.execute(sql, params).fetchall()
    # always a fresh query — never reuse a long-lived transaction across
    # calls (DuckDB's MVCC snapshot would otherwise pin a stale view and
    # miss the writer's later commits — a subtle, hard-to-reproduce bug
    # if violated).
```

## Early-Close Participation Gate

Evaluated FIRST, before the premarket wait below — there is no reason to wait on
markers for a session that will not open.

```
closes, ends = load_session_boundaries(db_conn)     # utils.md, once per process
                                                    # DuckDB is already open here:
                                                    # this block writes
                                                    # live_session_state below.
today_close = session_close(today)                  # utils.md, per date
if today_close != the ordinary close
       and not config["live_mode"]["trade_early_close_days"]:   # default false
    write live_session_state.session_diagnostics: participation blocked,
        today's session_close, and the KEY'S VALUE AT THIS RUN
    exit — no connections opened, no watchlist, no scan
```

**A session-start gate, not an entry gate**, and the distinction is what leaves
`gate_result`'s value list (db_schema.md) untouched. `freeze_reasons` is
session-wide yet still records per candidate, so being global is not by itself
disqualifying — the real difference is WHEN the answer is fixed. A freeze can
lift mid-session, so each candidate must be asked again; an early close is
settled before the first tick and never changes, so asking per candidate would
write one identical value across every row of the day. The cost of the
alternative is measured, not assumed: adding `'no_terms'` produced SEVEN
declaration sites across FOUR files, and one cardinality comment was missed and
needed a separate correction. A twelfth value would buy a repetition of "this
day was not traded".

**Why `live_mode:` and not `execution:`.** `execution:` is the shared block, and
R-7 moved `session_close_exit_time` INTO it precisely because both engines had
to agree on that value. This key is the opposite case — BacktestEngine must
ignore it and always replay early-close dates, since measuring whether those
days are worth trading is the whole reason they are labelled and kept in the
corpus. Opposite rationale, opposite side.

**Why the key's value is written to `session_diagnostics`** even though the date
itself is recoverable by joining `trading_calendar`. The calendar says the day
was an early close; it cannot say whether the gate was on when that session ran.
Once participation is enabled, no past session's setting is reconstructible —
the same reason `entry_fill_rate_disabled` is stored rather than derived. This
does NOT extend to `experiment_log`: nothing varies per backtest run here (see
09_backtest_engine.md).

**Carried positions wait a day.** A position held into an unopened early-close
session is not managed that day. Unified Overnight Policy liquidates it at the
next session's first Position Manager evaluation regardless, and early closes sit
adjacent to holidays, where the carry already spans multiple days.

## Session Start Gating (P-5 temporal ownership)

Before opening either connection above, LiveModeRunner waits on the
premarket batch's completion markers — this is a *timing* concern (don't
open a connection while the premarket job might still be writing),
distinct from Health Gate 1's premarket-marker check (a *data-completeness*
concern evaluated afterward, once the session has already started):

```
wait_deadline = 05:00:00 ET   # up to 1 hour from the premarket batch's
                               # 04:00:00 ET start (see metadata_crawler.md)
poll batch_runs every N seconds (polling interval unspecified — cheap,
    low-stakes choice) for stage IN ('premarket_rename',
    'premarket_corporate_events') AND date = today_date, status='success'
if both reach 'success' before wait_deadline: proceed immediately
elif wait_deadline reached first: proceed anyway (this IS the "degraded
    start" path — Health Gate 1c's warn-and-suppress-affected-tickers
    behavior is what actually handles the consequences; this wait's only
    job is not opening a DB connection while the premarket job might still
    hold write access)
```

## Session Start Probes (R-7 / R-8)

Two cheap, one-shot checks run after Session Start Gating and before the
Watchdog loop opens. Each is a single call; none is on a hot path.

The margin ratio was a third until it became PER TICKER. It is acquired at
watchdog first listing instead — see "Per-Ticker Trading Terms" below — which
is neither session start nor one-shot, so it is not a probe.

**Clock offset (R-8).** Config `clock_check`: `source` (`"ntp_daemon"` default
| `"vendor_api"` | `"disabled"`), `max_offset_seconds` (default 1.0),
`on_exceed` (`"abort"` default | `"warn"`).

`"ntp_daemon"` reads the local NTP daemon's own reported offset — no vendor
contract needed, the dependency closes inside the host. `"vendor_api"` is a
PLACEHOLDER: whether the trading API exposes a server clock is itself
unverified (api_contract_checklist.md), so the option exists to be selected
later, not to be implemented now.

1.0s is not arbitrary. Bar-Close Authority judges in whole seconds, scoped to
the tickers actually fetched inside a minute's accumulation window (see Feed
Outage Recovery's trigger condition 2, not the full watchlist) — past ~1s of
skew, EVERY ticker classified in that window misses simultaneously, because
the deadline is wall-clock derived and shifts identically for all of them. So
1s is the boundary at which a clock fault starts impersonating a feed outage,
in any window whose working set is large enough
(`len(candidates) >= min_watchlist_size`) for condition 2 to evaluate at
all. Below that floor this particular impersonation risk is moot — condition
2 does not fire regardless of clock health, leaving condition 1 (explicit
connection failure) as the only detector.

`abort` is the default because a skewed clock disguises itself as other
faults: it looks like a dead feed, it freezes exits along with entries, and it
misaligns `fill_second` against the vendor's tick timestamps so that
`fit_execution_params()`'s counterfactual replay lands on the wrong ticks and
biases `buy_rate`/`sell_rate` with no error anywhere. Refusing to start beats
running with silently corrupted calibration.

An `abort` here fires strictly before health_report.py's own session-start
invocation, so without a deliberate exception the operator learns nothing —
"no positions today because of a clock fault" and "an ordinary quiet day"
look identical from outside. On abort, send health_report immediately (at
`abort` severity) before terminating — the same mid-session trigger
mechanism Circuit Breaker uses below, reused rather than duplicated.

**Retention boundary (R-7).** Config `retention_probe`: `enabled`,
`lookback_days` (ask from further back than retention is expected to reach),
`assumed_days` (fallback).

Measured, not modelled: request history for one liquid ticker from
`now - lookback_days` and record the OLDEST timestamp actually returned as
`retention_boundary`, a session constant. This sidesteps having to guess
whether the vendor counts trading days or calendar days — the returned
timestamp is the answer either way. On failure or an empty response, fall back
to `assumed_days` and raise a health_report finding.

Consumed by Warm Restart gap-fill and Feed Outage Recovery: compare the gap's
start against `retention_boundary` BEFORE attempting to fill it. Today an
unrecoverable gap is discovered by asking and getting nothing back; this makes
it a prior judgement, so the crash path skips a pointless round trip at its
most time-critical moment and goes straight to `restart_gap_exit`. The
measured value is also logged every session, which fills in
api_contract_checklist.md's retention row from ordinary operation rather than
from a one-off measurement.

### session_diagnostics write protocol (R-9)

The two probe results above, plus Health Gate 2's tier-fallback summary
and two running counters, are persisted to
`live_session_state.session_diagnostics` (JSON — see db_schema.md) instead
of being held as in-memory tallies handed to `health_report.py` at session
end. A crashed session previously lost findings 5, 8, 21 and 22 outright;
in the table the evening liveness probe can recover them.

**Always write-only whole-value replacement.** The runner accumulates the
diagnostics dict in memory and replaces the column with it. There is never
a read-modify-write of the JSON, at any of the write points below, so no
atomicity question arises.

Write points:
```
(a) immediately after the probes above, BEFORE Health Gate 2
      clock_offset_start, clock_offset_end=null, retention_boundary; each
      probe flagged 'disabled' | 'succeeded' | 'fell_back' so "never ran"
      and "ran and fell back" stay distinct
(b) immediately after Health Gate 2
      adds the tier-fallback summary (finding 5)
(c) on the existing bar_latency_daily flush cadence
      refreshes the running counters — halt-check signal_source counts
      (finding 8), inference_log dropped-row count (finding 22), and the
      per-ticker terms-acquisition tally: first-listing attempts, failures,
      and the BLOCKED TICKERS BY NAME (see "Per-Ticker Trading Terms").
      The blocked list lives here and NOT in live_scan_daily: that table's
      grain is (date) with scalar metric columns and cannot hold names, and
      splitting the count there from the names here would put one fact in
      two places under a length-agreement constraint
(d) at Session Shutdown
      fills clock_offset_end (finding 21)
```
(a) must precede Gate 2 rather than being folded into (b): `clock_check`
with `on_exceed: "abort"` terminates the session before Gate 2 is reached,
and the measured offset is the only evidence for why. The evening probe
recovers the crash FACT (finding 11) but never the measurements.

`clock_offset_end` is written as `null` at (a) rather than left absent, so
its absence at report time reads as "session did not reach shutdown" rather
than as a missing key.

A flush failure at (c) is swallowed per db_schema.md's standing rule, and
must NOT increment finding 22's own counter — otherwise a persistent write
fault feeds itself.

**On warm restart**, the probe values at (a) are OVERWRITTEN with the newly
measured ones and `restart_count` increments. They are current operating
settings, not a session baseline — unlike `session_start_cash`, which R-2
restores rather than re-queries precisely because it IS the baseline. The
counters at (c) RESUME from the persisted values instead of resetting to
zero; they are session totals.

## Session Shutdown

**Exit trigger (R-9).** The process exits at whichever of these comes
first:
```
(a) all positions are flat AND now >= exit_deadline(today)
(b) now >= after_hours_end(today) - session_hard_exit_offset_minutes  # cap
```
(a) is the ordinary path — on a normal day the 15:59 `session_end` exits
fill within minutes and the process leaves shortly after 16:00; on an
early-close day both instants move with the calendar, to 12:59 and 13:00.
(b) exists
because submitting an exit is not the same as filling one: a limit exit is
re-priced every `position_check_interval_seconds` and escalates to market
at `exit_order_stuck_minutes`, and exit tracking has NO give-up timeout, so
without a cap a single unfillable position would keep the process alive
indefinitely.

The cap resolves per date: that date's `after_hours_end` minus
`session_hard_exit_offset_minutes` — `'200000'` ordinarily, `'170000'` on an
early close, at the default offset of zero. It is NOT the after-market
INGESTION boundary, though the two once read as one number: that range is fixed
by decision and does not move with the calendar (data_boundary.md,
metadata_crawler.md), while this instant does. They coincided at `200000` and
the coincidence was doing the explaining.

The offset is kept rather than derived away because what the cap is FOR is a
policy: leaving margin before the 21:00 evening batch, which is what
health_report.md's `drain_timeout_seconds` is sized against. R-7 kept
`session_close_exit_offset_minutes` on the same ground. An early close ends the
extended session at 17:00 and so widens that margin rather than narrowing it.

**Positions still open at the hard cap** are NOT force-liquidated. They are
left as `live_positions` rows and handed to the next session's Broker
Reconcile under the Unified Overnight Policy (see below) — the same path a
halt-through-close position already takes. Forcing a market exit into
after-hours liquidity would be strictly worse than the overnight carry it
is trying to avoid, and the row is durable (R-2), so the next session
adopts it rather than discovering an orphan.

**In-flight exit orders are canceled at shutdown**, before the markers
below. An order left resting after the process dies could fill overnight
with nothing watching it, and the position's state would then be wrong at
the next session's reconcile. Note this is a wider cancellation than Broker
Reconcile's, which cancels pending ENTRY orders only.

LiveModeRunner's last action before process exit, after both loops have
been signaled to stop and any final position/order state has settled:
```
db_write(
    "UPDATE batch_runs SET status='success', finished_at=?
     WHERE stage='live_session_end' AND date=?",
    ...
)
# or INSERT if no 'running' row was created at session start — either is
# fine as long as exactly one live_session_end row per date results.

# R-2: also flip the session's own start marker (written 'running' at
# Session Lifecycle Step 1c) to 'success':
db_write(
    "UPDATE batch_runs SET status='success', finished_at=?
     WHERE stage='live_session_start' AND date=?",
    ...
)
# Invariant this maintains: a today live_session_start row still 'running'
# with no live_session_end row = crashed session — this is exactly what
# warm-restart detection keys on (see "Session Restart (Warm Start)"
# below). A clean shutdown always flips both markers together.
```
The evening batch job (metadata_crawler.md) waits on this marker before
its own first stage begins — see metadata_crawler.md's Dual Schedule.

---

## Session Lifecycle

```
LiveModeRunner.start_session(today_date):

  1. Fetch today's tradable ticker list from the trading API — its
     ticker-master endpoint, reached through TradingAPI (see
     docs/api/trading_api.md). NOT the watchdog: that module returns the
     entry-point working set, a different thing at a different scale.
         all_tickers: list[str]
         (tickers with prior-day data but no trading access today
          are automatically excluded by the API response)

  1a. Token coverage gate. Read the cached access token's expiry and
      require it to cover today's hard cap. TODAY's, not a fixed 20:00:
      demanding coverage to 20:00 on a 13:00-close day would abort a session
      whose own cap is 17:00.
      Insufficient coverage ABORTS the session; nothing here refreshes the
      token. Refresh happens only in its own scheduled stage
      (metadata_crawler.md) — refreshing at session start would spend the
      one-per-minute issuance budget with the session waiting on it, and
      would turn a loud batch failure into a quiet one.
      Tests EXPIRY, not presence: the SDK's revoke swallows a failed
      revocation while still clearing the local cache, so a reissue can
      return the SAME token with the SAME expiry and a presence check
      would pass on a refresh that never happened (see
      docs/api/sdk_dependency.md).
      This session's client runs with the SDK's automatic token reissue
      DISABLED — batch entry points keep it enabled, having no WS
      connection to disturb, but here an uncoordinated revoke would drop
      both WS connections mid-session.

  1b. Query account balance from trading API — `inquiry/balance-margin`'s
      `AstkOrdAbleAmt`, the UNCORRECTED DEPOSIT:
         session_start_cash: float
      Deliberately the pre-leverage figure rather than buying power. The
      measured identity `AstkOrdAbleAmt * (100 / Mgnrt0) - @cost =
      AstkOrdAbleAmt1` fixes the first as deposit and the last as buying
      power; sizing takes the deposit because `position_size_cash_pct` and
      both caps express exposure as fractions of OWN capital. The leverage
      correction is applied PER CALL from the ticker's own `Mgnrt0` (see
      "Per-Ticker Trading Terms"), never folded in here — `Mgnrt0` is
      per-ticker while this is one scalar.
      Fixed for the entire session — this is the `balance` argument to
      every `execution_common.compute_position_size()` call this session
      makes (see Watchdog Polling Loop step 5c.ii), mirroring backtest's
      `initial_cash` (see 09_backtest_engine.md's Position Sizing). NOT
      re-queried mid-session even if real account balance changes as
      trades fill — sizing is meant to use a fixed reference, same
      rationale as backtest's fixed initial_cash (see execution_common.md's
      Constraints for why sizing and funds-availability must not share a
      value). The funds-availability gate
      (`execution_common.check_funds_available()`) is what uses a
      real-time queried balance instead — see that call site.

  1c. Persist session start (R-2):
         db_write INSERT batch_runs stage='live_session_start',
             date=today, status='running'.
         db_write INSERT live_session_state (date, run_id,
             session_start_cash, started_at).
      This is the authoritative sizing basis for the session — a warm
      restart (see "Session Restart (Warm Start)" below) restores it
      rather than re-querying the balance, which would reflect a
      post-fills value, not the session's original basis.

  1d. Broker Reconcile (shared procedure — R-3, see "Broker Reconcile
      (shared procedure)" below): run at EVERY session start, cold start
      included. At cold start, any broker position found is by
      definition an overnight orphan — nothing has opened yet today. This
      is the standing guard against a prior-day position that never
      appeared in a warm restart (e.g. a cleanly-shut-down session that
      left a halt-through-close position). Warm restart already reconciles
      in its own step 1 and does not double-run this.

  2. Bulk load session_stats for ALL tickers from DuckDB
         session_stats_raw = SELECT * FROM precomputed_session_stats
                             WHERE as_of_date = today_date
                               AND n_sessions = config_n_sessions
         # as_of_date = today was inserted by collect_daily.py the previous evening
         # and contains baselines from the prior N sessions.

         all_stats_nested = build_session_stats_dict(
             session_stats_raw,
             delta_minutes=config["indicators"]["reference_session"]["delta_minutes"],
             session_mode=config["entry_detector"]["session_mode"],
             closes=closes,
         )
         # closes has been in scope since the early-close gate, which runs
         # before this Lifecycle.
         # all_stats_nested: {today_date: {ticker: {metric: {hour: smoothed_avg_value}}}}
         session_stats_bulk = all_stats_nested[today_date]
         # session_stats_bulk: {ticker: {metric: {hour: smoothed_avg_value}}}

  3. [Meta Bulk Load] Bulk load stock_meta for today, with per-field fallback:
         stock_meta_today = SELECT * FROM stock_meta WHERE date = ?
         # stock_meta is (ticker, date)-keyed (see db_schema.md V-1 fix); a row
         # exists only if today's crawl (see metadata_crawler.md dual-schedule)
         # succeeded for that ticker/field by session start.

         meta_bulk: dict[str, dict] = {}
         for ticker in all_tickers:
             meta_bulk[ticker] = {
                 field: (
                     stock_meta_today.get(ticker, field)
                     if ticker in stock_meta_today.index and pd.notna(stock_meta_today.loc[ticker, field])
                     else utils.estimate_historical_meta(ticker, today_date, field, db_conn)
                 )
                 for field in ["market_cap", "shares_outstanding", "price_52h", "price_52l"]
             } | {
                 "sector": most recent available stock_meta.sector for ticker
                 # sector has no derivation path — most recent snapshot regardless of date
             }
         # meta_bulk is looked up (not rebuilt) inside the watchdog loop's
         # Inferencer.infer() call — see Watchdog Polling Loop step 5b.
         # estimate_historical_meta() fallback should rarely trigger for
         # today's own date given the dual-crawl schedule, but is not
         # guaranteed-zero (vendor update latency — see metadata_crawler.md).
         # Moved ahead of Eager Pool (was Step 3b, after) specifically so
         # Health Gate 1 (next) can evaluate stock_meta coverage before
         # paying Eager Pool's cost — see Health Gate 1 rationale below.

  4. [Health Gate 1] Pre-Eager-Pool health checks — evaluated after Steps 2-3,
     before the expensive Eager Pool parallel compute. Two of four checks
     (a, b) are measured directly from data already loaded above; the other
     two each require their own lookup — c a batch_runs marker query, d
     (N-8) the model artifact's feature_names, loaded here rather than at
     its originally-later point (see d's own note):

     a. session_stats coverage:
            eligible = SELECT ticker FROM ticker_data_coverage
                       WHERE date = {prior_trading_day} AND has_1min = TRUE
            # tickers with no prior-day trading activity are excluded from
            # the denominator — their session_stats being empty is expected,
            # not a batch-health signal.
            coverage_rate = |{t in eligible ∩ all_tickers
                              : session_stats_bulk[t] not empty}|
                            / |eligible ∩ all_tickers|
            coverage_rate < 0.90 → ABORT SESSION (hard gate; P-0-class failure)

     b. stock_meta coverage (per field, then aggregated):
            for field in [market_cap, shares_outstanding, price_52h,
                          price_52l, sector]:
                eligible(field) = { t : EXISTS (SELECT 1 FROM stock_meta
                                    WHERE ticker=t AND date >= today - 20
                                    trading days AND {field} IS NOT NULL) }
                # a ticker that has never carried this field (vendor/ticker-
                # type limitation) is excluded from that field's denominator
                # — only a ticker that normally HAS the field but is missing
                # it today counts against coverage.
            coverage_rate = Σ_field |{(t,field) : t in eligible(field),
                                       real value present today}|
                            / Σ_field |eligible(field) ∩ all_tickers|
            coverage_rate < 0.80 → WARN
            coverage_rate < 0.50 → ABORT SESSION

     c. premarket completion marker:
            SELECT status FROM batch_runs
            WHERE stage IN ('premarket_rename', 'premarket_corporate_events',
                             'premarket_quarantine_check')
              AND date = today_date
            missing or status != 'success'
                → WARN + proceed with yesterday's state
                → suppress new entries on any ticker where
                  is_tradable() returns False
                  (enforced downstream — see Watchdog Polling Loop)
            # 'premarket_quarantine_check' added for P-8's
            # check_corporate_event_anomaly() — that stage's own writes
            # use ticker_cik_map.quarantine_reason (N-2 — see
            # db_schema.md), a column independent of the rename case's
            # rename_pending, so it needs no separate handling here beyond
            # being included in this marker check; a ticker it would have
            # evaluated but didn't reach is simply not yet protected
            # today, not incorrectly blocked (see that function).
            # 'premarket_quarantine_recheck' (the 09:20 ET pass, now a
            # LiveModeRunner in-process task — see "In-Process Premarket
            # Recheck" — R-1) is deliberately NOT in this list — it runs
            # well after Health Gate 1 (which fires shortly after Session
            # Start Gating, around 04:00-05:00 ET), so its completion
            # cannot be known yet at this check regardless of stage name.

     d. feature_names consistency (N-8):
            model_features = set(meta["feature_names"])   # from the run_id's
                                                            # model artifact
                                                            # meta.json (see
                                                            # base_model.md)
            extractor_features = set(FeatureExtractor(config).get_feature_names())
            mismatch = model_features.symmetric_difference(extractor_features)
            mismatch is non-empty
                → ABORT SESSION (hard gate)
            # No warn-and-proceed variant considered — a mismatch means
            # every inference this session would either KeyError on a
            # missing feature or silently ignore one the model never
            # trained on, depending on direction. Both are as severe as
            # the P-0-class session_stats coverage failure above; there is
            # no meaningful degraded mode for "the model and the feature
            # pipeline disagree on what a feature vector is."
            # Cheap relative to Eager Pool — reads two already-available
            # name lists, no additional DB or model-artifact I/O beyond
            # what Step 7's Inferencer init would load anyway (moved
            # forward from there to before Step 5's cost, same rationale
            # as the other three checks in this gate).

     Any ABORT SESSION outcome stops here — Eager Pool (Step 5) never runs.

  4b. [Late-start freshness branch — R-1] Evaluate once, here, before Eager
      Pool. Session Start Gating's own wait is bounded (hard deadline
      05:00 ET — see Session Start Gating), so this branch matters only
      when a manual restart (e.g. after a Health Gate 1 abort elsewhere in
      the morning) lands past recheck_time; it also correctly no-ops for
      an Eager Pool that is merely running late on a normal start.

         if now >= config["live_mode"]["premarket_recheck_time"]:
             # Past recheck_time already — run the FULL fresh-check bundle
             # NOW, BEFORE Eager Pool, so Step 0 computes on correct data
             # and no scoped recompute (caching_calculator.md, item N) is
             # needed on this path:
             #   quotes/halt fetch
             #   + check_corporate_event_anomaly (full universe, fresh)
             #   + yfinance narrow crawl for newly-quarantined tickers
             #   + crawl_corporate_events_investing(today)   (item N)
             run_premarket_recheck_bundle()
             in_process_recheck_done_today = True   # suppresses the
                                                     # scheduled task's own
                                                     # run at recheck_time
                                                     # today — see below
         else:
             pass   # normal path: the In-Process Premarket Recheck task
                    # (below) fires at recheck_time as scheduled.

  5. [Eager Pool] Parallel session_start_compute() for all tickers:
         Using worker pool (config: live_mode.session_start_workers)

         For each ticker in all_tickers (parallelized):
             historical_bars = utils.load_ohlcv_with_history(
                 ticker, lookback_start_date, today_date, db_conn
             )
             # Stitches in predecessor-ticker bars if a rename occurred within
             # the lookback window (see utils.md get_ticker_history() /
             # load_ohlcv_with_history()) — returns None-history fast path
             # (a single direct query) for the vast majority of tickers.
             # Loaded in the same per-ticker parallel worker, alongside
             # historical_bars — both must be ready before session_start_compute():
             tick_bar_history, tier_used = utils.load_tick_bar_aggregates_with_history(
                 ticker,
                 indicators=[i for i in configured_tick_indicators
                             if i.precalculate_bars == "lookback"],
                 lookback_start_date, today_date, db_conn
             ) if any_indicator_configured_lookback else (None, {})
             # tier_used: {indicator_name: 1|2|3} — which resolution tier
             # fired per requested indicator; empty dict when no indicator is
             # "lookback"-configured. Returned (not just used internally) so
             # Step 6 can aggregate it after all workers report back — see
             # Health Gate 2.
             # Corporate-event split adjustment is NOT applied here — that
             # happens inside session_start_compute() itself, anchored to
             # today_date (see caching_calculator.md).

             calc = CachingIndicatorCalculator(config)
             calc.session_start_compute(historical_bars, tick_bar_history)
             calc.set_session_stats(
                 session_stats_bulk.get(ticker, {})
             )

             with write_lock:   # R-2 GAP-FIX 9: persist_to_db() does its
                 # own DELETE-then-INSERT sequence on writer_conn with no
                 # locking of its own (see caching_calculator.md — its
                 # docstring never mentions write_lock). 8 parallel Eager
                 # Pool workers each calling this concurrently, unlocked,
                 # is exactly the hazard write_lock exists to prevent —
                 # unlike db_write()'s callers elsewhere, this call site
                 # was missed when db_write() became the funnel. Holding
                 # the lock for this whole call (not a single execute())
                 # serializes the DELETE+INSERT pair, not just each
                 # statement — necessary since a half-applied delete from
                 # one worker interleaved with another's insert is exactly
                 # the kind of race a per-statement lock would not prevent.
                 calc.persist_to_db(db_conn, today_date, ticker)
             # R-2: written in BOTH modes now, as a crash-recovery backup —
             # each worker persists its own ticker as soon as it finishes,
             # not batched at the end. A warm restart (see "Session Restart
             # (Warm Start)") reads this via load_from_db() to skip the
             # expensive historical_bars recompute.
             if config["live_mode"]["indicator_cache_mode"] == "db":
                 pass   # RAM released; calc discarded after persist (unchanged)
             else:  # "memory" (default)
                 calculators[ticker] = calc  # retain RAM copy — authoritative
                                             # for this running session; the
                                             # backup above is read only on a
                                             # warm restart, never in normal
                                             # operation.

             worker returns (ticker, tier_used) alongside the calc/persist result

         del session_stats_bulk   # RAM released after all tickers done

  6. [Health Gate 2] Post-Eager-Pool health check — tick_bar_aggregates
     Tier-2/3 fallback rate. Depends on Step 5's tier_used results, which is
     why this cannot be folded into Health Gate 1 (Step 4):

         tier_tally = Counter()
         for ticker, tier_used in step_5_results:
             for indicator, tier in tier_used.items():
                 tier_tally[tier] += 1
         total = sum(tier_tally.values())
         fallback_rate = (tier_tally[2] + tier_tally[3]) / total if total > 0 else None

         fallback_rate is None → no lookback-configured indicator exists yet;
             check trivially passes (all 9 tick-derived indicators
             currently ship precalculate_bars: 0)
         fallback_rate > 0.20 → WARN + proceed (self-healing case, but batch
             likely failed — alert operator)

  7. Inferencer init:
         Pass closes and ends in — this process holds them already, so the
             Inferencer does not open a second acquisition for the same fact
         calculate_required_history() → self.required_bars
         Load model artifacts (run_id from config)

  8. Start watchdog polling loop (async)
  9. Start position manager loop (async)
```

**Bulk Load Memory Profile** (Steps 2-3, 5 — renamed from "Phase 1" now that
the lifecycle no longer labels a single step that name; scope unchanged):
```
session_stats_bulk peak:  ~15,000 tickers × ~18KB ≈ ~270MB
                          Released after Step 5 completes.

tick_bar_history (Eager Pool, only if any tick-derived indicator is
configured "lookback" — none are, by default; all 9 currently ship with
precalculate_bars: 0):
  Per-ticker cost scales with bar count (like historical_bars), not tick
  count — bar-level EAV cache, not raw tick replay. Negligible at default
  config; re-profile if any indicator is switched to "lookback".

calculators pool ("memory" mode):
  ~15,000 tickers × ~790KB (Layer1 + Layer2 + session_stats) ≈ ~11.9GB sustained
  Recommended server: 32GB+ RAM.
  For memory-constrained environments: use indicator_cache_mode = "db"
  (50~200ms per-ticker load latency on a ticker's first appearance in the
  watchdog working set).
```

---

## In-Process Premarket Recheck (R-1; hosts item N's 09:20 crawl calls)

A LiveModeRunner background task, scheduled for
`live_mode.premarket_recheck_time` (default 09:20 ET) — unless
`in_process_recheck_done_today` was already set by the late-start branch
(Session Lifecycle Step 4b), in which case this task's scheduled run is
skipped for today. Replaces the former standalone `--premarket-recheck`
cron, which could not open the DB read-write while this process holds it
(DuckDB single-writer) — so the recheck, P-8's primary coverage layer,
would have failed silently every session. Running it in-process, through
this process's own writer, fixes that.

Task body (`run_premarket_recheck_bundle()` — shared with Step 4b's
late-start path; all fetches as a non-blocking background task so the 1s
watchdog and 5s position loops are not stalled):

```
1. quotes      = bulk_fetch_today_first_price(...)   # trading API, chunked
   halt_status = utils.query_halt_status(...)          # NOT a dbsec call
   # sequential, same as the 04:00 pass (see metadata_crawler.md)

2. check_corporate_event_anomaly(db_conn, config, quotes, halt_status)
   # full-universe fresh re-evaluation, NOT a delta — writes through
   # db_write() (the existing write_lock funnel; no new serialization
   # mechanism needed). batch_runs stage='premarket_quarantine_recheck'.

3. crawl_corporate_events_investing(today, db_conn)   # item N bulk vendor
   # investing.com, unconditional every recheck. Writes via db_write().

4. For any ticker whose quarantine_reason was NEWLY set in step 2, run a
   single-ticker yfinance narrow crawl:
       crawl_corporate_events(ticker, db_conn)
   (already a single-ticker function — no new function needed.)

5. Scoped-recompute trigger (item N): for any ticker that gained a new
   same-day corporate_events row in steps 3-4 AND whose calculator already
   exists (i.e. Eager Pool has run — always true for this scheduled path,
   since it only fires after Step 5), call that ticker's
   scoped_recompute() (caching_calculator.md) as a background task, then
   clear its quarantine_reason if set. NOT run on Step 4b's late-start
   path — there, Eager Pool hasn't executed yet, so Step 0 picks up the
   fresh corporate_events rows naturally and no scoped recompute applies.
```

All writes in steps 2-5 go through `db_write()`; no lock-handoff or
inter-process coordination (the two-cron collision class this replaces was
removed by N-1 and must not be reintroduced). Before starting, this task
checks whether the 04:00 `--premarket-open` pass is still mid-crawl
(`batch_runs` `stage='premarket_corporate_events'`, `status='running'`) and
defers rather than contend — the recheck-vs-04:00 guard is retained; the
recheck-vs-LiveModeRunner-writer contention (the reason this task exists at
all) is dissolved by sharing this process's own `db_write()` funnel.

Failure handling: if the fetches fail, log + write the recheck `batch_runs`
row `status='failed'`; no freeze. The recheck is a defense layer, not
session-critical — `is_tradable()` keeps using 04:00-pass state. Same
degraded philosophy as Health Gate 1c.

Manual-run caveat: `--premarket-recheck` (see metadata_crawler.md) survives
only as a manual/debug flag; run against a live session it will fail to
open RW. Manual use is for non-session contexts.

---

## Watchdog Polling Loop

Polls the watchdog at `poll_interval_seconds` (default: 1s). The watchdog
remains a separate external system applying its own entry-point-condition
matching against the live feed; what this codebase holds is the ACCESS
SURFACE to it — a project-internal module whose entire interface is one
call:

```python
import watchdog
watchdog.get_candidates() -> list[str]
```

No URL, no config key: it is a module call, not an HTTP request.

**What it returns is a CUMULATIVE WORKING SET, not a per-cycle delta.**
Up to 50 tickers, ordered most-recently-fired first, no duplicates; a
ticker that fires again moves to the top rather than being appended. A
ticker therefore stays in the list long after the event that put it
there, and the list is nearly always full during the regular session.

**Caller-side delta computation is impossible, not merely lossy.** When
the top-of-list ticker re-fires, the returned list is IDENTICAL to the
previous cycle's — same members, same order. No comparison of successive
snapshots can observe that event. Set difference misses every re-fire;
scanning down to the previous top misses a re-fire of the previous top
itself, and misses cases where the top and another ticker both update
within one poll interval. The list is state, not an event stream.

**Consequence — the model is inverted.** The watchdog list is the WORKING
SET and a bar close is the evaluation trigger, rather than a watchdog flag
triggering a fetch of that ticker's bar. Re-evaluation is driven by bars
arriving, which is the shape the rest of the pipeline already has:
`EntryPointDetector.detect()` operates on a bar sequence, and
`on_bar_close()`, the 2-print guard and `CachingIndicatorCalculator` are
all bar-driven. The watchdog edge was the exception, and it was never
observable.

Two concerns dissolve with the inversion rather than needing handling: no
warm-up baseline cycle is required, because nothing is diffed; and the
50-entry cap loses nothing, because an evicted ticker simply leaves the
working set rather than taking an unobserved event with it.

**A failing watchdog returns an empty list rather than raising.** Its
filter conditions make an all-day zero impossible, so an empty working set
inside the regular session is a fault signal rather than a quiet market —
recorded as `health_report.md` finding 29. Observation only: no freeze and
no Feed Outage trigger, since a dead watchdog costs entries, not
correctness, and a false freeze over an open position costs more than a
session that trades nothing. The regular-session scope is what removes the
need for a grace window — the list carries over already full from
premarket, so 09:30 is the boundary.

**The loop does NOT fetch a bar for every working-set ticker every cycle.**
That reading is what once made this section unbuildable: 50 per-ticker
fetches serialise well past `poll_interval_seconds`, because the SDK paces
at a minimum interval rather than allowing a burst. The working set is what
is ELIGIBLE to be looked at, not what is looked at each second.

**PREFIX SCAN.** The list is ordered most-recently-fired-first, so a ticker
that re-fires rises to the head. Scan from the head and stop at the first
ticker whose own BAR DATA has not advanced since this ticker was last
fetched: everything below it is, by the ordering, even staler in firing
terms. Per-cycle cost is therefore D+1 calls, where D is the number of
firings since the previous cycle — not the working-set size.

This is NOT in tension with "caller-side delta computation is impossible"
above. That finding concerns deltas of the LIST — which tickers are in it
and in what order — and it stands unchanged. What the scan tests is a delta
of each TICKER'S OWN BAR DATA, fetched per ticker, which the list cannot
express and never could.

**Slot allocation, and why the bound is latency rather than TPS.** Rate
governance paces at a MINIMUM INTERVAL, not a per-second bucket that can be
drained at once: each account allows a chart/min call every 250ms, so a
cycle offers each account slots at t=0, 250, 500 and 750ms. Two accounts
run in alternation (measured; that check is retired as
`api_contract_checklist.md` T-15 — the demo account returns identical
`chart/min` data on an independent budget), so:

```
t=0, t=250   both legs   →  SCAN — RESERVED  (N head + M rotation)
t=500, t=750 both legs   →  shared, up to 4 calls
```

The non-reserved half is a CAP shared by several consumers, not an
allocation belonging to one. `chart/min` has TWO consumers here — carryover
and promotion — plus the bar-close superset-K fetch, which preempts both at
bar close. Position Manager Loop Step 1 was a third until `bars_since_entry`
was removed; it fetches no bars now, so `trading_api.md`'s Call-Point
Inventory lists this endpoint against the scan alone. Every pairwise
precedence is stated — the lane yields to superset-K, carryover before
promotion — and re-derivable from EDF slack, so nothing here is left open.

Last scan call completes at `250ms + RTT`, i.e. 850ms worst case at the
measured 400-600ms round trip (that check is retired as
`api_contract_checklist.md` T-16; the `scan:` keys below are its record) —
inside a 1s cycle. Giving the whole
auxiliary leg to the recovery lane instead would force the scan onto one
leg, firing its last call at 750ms and completing at 1150-1350ms, so the
allocation is BY PACING SLOT, not by leg. N and M are config keys: if RTT
drifts up past ~600ms, N must come down.

**The three slot kinds and what each fetches.**

- **Head (N=3).** The top of the list. Forming bar only, so the smallest
  window that reaches this ticker's current bar open. Seen every second,
  so no crossing inside the bar can be missed.
- **Rotation (M=1).** One position per cycle through the remainder, so the
  whole list is walked in about 50 cycles. Fetches a 120-SECOND window and
  runs TWO evaluations on the one payload: the forming bar (as the head
  slots do) and the PRECEDING COMPLETED BAR at full strength. 120s is the
  exact minimum with margin — 60s of forming bar plus the 60s completed bar
  behind it — and it holds regardless of the cursor's phase. Because the
  rotation period (~50s) is shorter than a bar, some bars get two visits; a
  per-ticker last-evaluated-bar record suppresses the duplicate.
  `entry_points` is idempotent under INSERT OR IGNORE but `inference_log`
  is not, and a duplicated `signal_fired` row is a duplicated entry.
- **Shared, up to 4.** Carryover and promotion — see Recovery Lane
  below — sharing the cap with the bar-close superset-K fetch, which
  preempts them at bar close.

**A response that came back is EVALUATED, never discarded.** The head slots
fire SPECULATIVELY — all N go out before any of them lands, because waiting
on each to decide whether to send the next would cost D x RTT instead of one
RTT — so a cycle whose delta stops at position 1 still holds responses for
positions 2 and 3. Those are evaluated. Absence of a watchdog re-fire is NOT
absence of bar growth: B-G are cumulative and can cross with the watchdog
never firing at all, which is exactly the accepted gap below, and the data is
already paid for.

**V60 reorders the queue and decides nothing.** The realtime trade stream is
already subscribed for open positions (Exit Architecture); where it also
covers working-set tickers, its print RATE is used to reorder which ticker a
head slot reaches first. No threshold is ever evaluated against V60-derived
data, and no bar is ever built from it. Three things bar the measurement
role, and all three dissolve for a ranking one. A reactive subscription
yields a bar missing its PREFIX — V60 delivers prints only from subscription
onward, while a cumulative-volume condition needs them from bar open. The
free entitlement carries roughly half the prints, which is worst-case for a
volume threshold and is not a stable constant that could be scaled out.
And backtest cannot reproduce WHICH half is delivered, so any decision taken
from V60 data would be unreproducible. Ordering needs only monotonicity in
activity, not calibration, and an ordering error shifts detection by a cycle
without changing what is decided — which is also why PARTIAL coverage is
fine, and why V60 subscription slots go to positions first and the second
account's session is an optimisation rather than a prerequisite.

**Window size is one rule, not three constants.** `dataCnt` covers
(this ticker's last completed bar) minus (the last bar already held for
it). The head slot's minimal window, the rotation slot's 120s and a cold
ticker's backfill are the same call under that rule; a ticker absent from
10:30 and returning at 11:47 needs 77 bars, not a session. The calculator
pool persists per ticker across the session, so only the gap is refetched.

**Resolution is chosen by TARGET, not by slot.** Minute mode does not
return an incomplete bar at all, which is the entire reason
`InputDivXtick=1` is used here (measured; that check is retired as
`api_contract_checklist.md` T-14, recorded in `trading_api.md`'s Call-Point
Inventory); inverted, a completed bar never
needs second resolution. Head and rotation slots therefore run at second
resolution; backfill runs at MINUTE resolution, where one call covers a
whole session — 330-720 rows against the 2000-row `dataCnt` ceiling —
against roughly twelve paged calls at second resolution.

**Backfill has two forms.** AT REGULAR-SESSION OPEN it is a bulk burst:
every working-set ticker needs bars from `volume_base_hour` (default
04:00) onward, because conditions D, E and F are cumulative from it and
G's 20-bar and C's 5-bar windows both reach back into premarket — and
today's premarket bars are NOT in DuckDB, since ingestion is the evening
batch and `load_ohlcv_with_history()` returns prior days only. This is not
a spike to be smoothed: the scan is IDLE at open, because a ticker without
today's bars cannot be evaluated at all, so the full combined chart/min
budget is available and 50 tickers take about 6.25s against the 60s before
the first bar close. AFTER OPEN, a newly-appearing ticker's backfill is
issued AS its head slot rather than as an extra call, for the same reason:
that slot would otherwise produce nothing. Its response is not awaited
within the cycle — the ticker is unevaluable until it lands either way —
and it rejoins the ordinary scan from the cycle after arrival. This attaches
to WATCHLIST APPEND, which already fires on exactly this event — first
appearance this session, or return after eviction — so no new lifecycle
event appears, and a ticker with no DB history at all resolves through the
existing `preload_fail` path. NOTE the mode distinction: Watchlist Append
itself runs in BOTH `indicator_cache_mode` settings, while the calculator
restore and `session_stats` Phase 2 load nested inside it are `"db"`-mode
only. Backfill hangs off the hook, NOT off that nested branch — the eager
pool loads prior-day history from DuckDB in either mode and today's bars are
in neither, so a memory-mode session needs backfill exactly as much as a
db-mode one.

**ONE ASSUMPTION IS ACCEPTED, NOT PROVEN** (`api_contract_checklist.md`
T-17). Early stop is sound only if the watchdog fires for every ticker that
crosses conditions A-G. The watchdog applies its OWN matching, and a ticker
that crosses ours without firing its own does not rise to the head, so the
scan can stop above it.

**The rotation cursor ALONE does not reduce that to a delay, and reading it
that way is a mistake worth naming.** The cursor visits a ticker at an
arbitrary phase within the bar, so a crossing occurring AFTER that visit is
not seen until the next pass — which lands after the bar has closed. D and E
are cumulative across the day and survive to the next bar, but B, C, F and G
are bar-local and need not, so the bar can be lost outright rather than
merely detected late. What actually converts loss into delay is the rotation
slot's 120-second COMPLETED-BAR re-evaluation together with promotion, not
the cursor's existence. Tickers the watchdog does fire on are unaffected
either way — the head slots reach them within a cycle or two.

What makes acceptance defensible is that the residual is MEASURED rather
than assumed small: `metadata_crawler.md`'s `evening_detection_gap` stage
runs `detect()` over the day's ingested bars and compares against
`inference_log`. Note the limit of every in-session mitigation here: they all
walk the LIST, so a ticker the watchdog never lists at all is invisible until
that evening comparison.

**The watchdog module itself is OUT OF SCOPE**, and this is a different
kind of exclusion from the one above. It is the access surface to a
separate external system, specified here by its one call and nothing else,
and is not an implementation target of this spec set. Everything in this
loop is now buildable.

**Loop termination (R-9).** This loop runs only while entries are
structurally possible. At the top of every cycle, before Step 1:
```
if now >= exit_deadline(today):
    break        # exit the loop; the process itself lives on until
                 # Session Shutdown's exit trigger fires
```
Breaking between cycles rather than mid-cycle means no partial fetch or
half-applied `on_bar_close()` replay. Past that time no entry can be
submitted anyway (after-market bars are excluded from entry detection in
every mode), so nothing is lost — but the loop is stopped explicitly rather
than left polling an external service whose after-hours behaviour is not
part of this system's contract and whose responses would be exercising the
candidate path for no possible benefit. Close-out continues on the Position Manager Loop, which
is independent of this one and does its own bar/tick fetching.

A warm restart that lands after `exit_deadline(today)` still starts this
loop (Session Lifecycle Step 8) and it self-terminates on its first cycle
through the same guard — no separate branch is needed.

Two accepted consequences: Bar-Close Authority and its Bar-Arrival Latency
sampling (finding 26) stop collecting at `exit_deadline(today)`, and
watchdog-triggered Feed Outage Recovery stops evaluating there too. The
first is fine because the day's curve is finalised by the final
`bar_latency_daily` flush in Session Shutdown; the second because close-out
runs off wall-clock triggers and the WS/REST tick stream, not the bar
channel Feed Outage Recovery protects, and every remaining position is
already exiting.

```
loop every poll_interval_seconds:

  1. watchdog.get_candidates() → the current working set (cumulative,
     most-recently-fired first — see note above; NOT this cycle's new
     arrivals, which are not observable)
     If empty AND inside the regular session: record finding 29
        (once per episode, re-armed when the list refills)

  2. Allocate this cycle's slots (see PREFIX SCAN above), then for each
     ticker a slot resolves to:
     a. Fetch current bars from trading API (chart endpoint up to now —
        the API always returns the full range since the caller's last
        query, not just the latest bar). InputDivXtick=1 and a dataCnt
        sized by the deficit rule for head/rotation slots; minute
        resolution for a backfill
     b. Fetch current ticks from trading API (10-tick up to now).
        Indicator input only — NOT a detection input; the tick channel is
        unchanged by the scan design
     c. Update intraday state in memory / DuckDB

     Slots: N head (prefix scan, stop where this ticker's own bar data has
     not advanced), M rotation (120s window, evaluated twice — see below),
     and the shared slots. A cold ticker's head slot carries its backfill
     instead, and its response is not awaited this cycle.

  3. If one or more new 1min bars completed for ticker since last check:
         for each newly-arrived bar, in chronological order:
             calculators[ticker].on_bar_close(bar, ticks_for_bar)
             if 093000 bar just closed:
                 calculators[ticker].on_regular_session_open(bars_including_930)
         # Looping here (rather than assuming exactly one new bar) is what
         # lets a ticker evicted from the working set for a while and later
         # returning catch up correctly in the same step used for first entry
         # — see "Watchlist Append" below. on_bar_close() itself is O(1)
         # per call for every CONTINUOUS indicator (see caching_calculator.md),
         # so looping over several missed bars here is cheap regardless of
         # how many accumulated.

  4. If ticker appears in candidates for the first time this session,
     or is returning after one or more cycles absent (evicted past the
     50-entry cap, then fired again)
     → see "Watchlist Append" below (Step 3's loop already replays any
     bars missed in between; this step only concerns first-time calculator
     setup when indicator_cache_mode="db")

  5. For each ticker with completed bar and active calc:
     a. Mid-bar screen only (rotation slot, forming bar): evaluate B-G
        normally and RELAX A to PASS. A (0 < price <= 20) is the one
        non-monotone condition — it reverts in BOTH directions as price
        moves — so relaxing it is what makes the mid-bar result a PROVABLE
        SUPERSET of the bar-close candidate set rather than an arbitrary
        subset (see 01_entry_detection.md's monotonicity classification).
        The superset is a FETCH LIST, never an entry list.
     b. At bar close, call Inferencer.infer( bars, ticks,
            meta_bulk[ticker], entry,
            session_stats=calculators[ticker]._session_stats
        ) for EVERY member of the superset, with no detect() pre-filter
        here. Inferencer's own step-3 detect() does the rejecting and
        returns before feature extraction or model.predict(), so this is
        free — and it is what keeps the instrumentation intact:
        inference_log's no_detection event is the only record anywhere
        that a candidate was EVALUATED AND REJECTED, and pre-filtering
        would erase the distinction between "we looked and it failed" and
        "we never looked", which the evening gap measurement rests on.
        Bar-close re-evaluation applies the FULL expression to the
        COMPLETED bar, so the final entry decision is identical to what
        pure bar-close evaluation would have produced: the t-1
        reference-bar convention, the label anchor and train-serve parity
        are all untouched by the scan.
     c. If signal (up5 / up3):
        Fire an async, fire-and-forget REST bid/ask query for this ticker
        (5-level book) → `bid_ask_snapshots` (`source='signal_time_rest'`).
        Every confirmed signal, gate outcome and shadow/real status both
        irrelevant — a candidate the gates go on to block is exactly the
        kind of observation the execution-gate path of the bid/ask open
        item needs. Not on the entry-submission critical path: this call
        is not awaited before gate evaluation or order submission below
        proceed. It shares the orderbook endpoint's budget with the exit
        ladder's re-quote round-robin and PREEMPTS it at bar close — see
        Position Manager Loop's ladder block.
        0.  Entry gates (R-5) — evaluated in this order, before every step
            below. Enforced identically in shadow mode; unlike R-4's
            breaker (evaluate-but-don't-enforce in shadow), a shadow run
            that ignored these caps would record entries live could not
            have taken, overstating exactly what shadow exists to measure.
            - freeze_reasons: skip if any active reason's scope includes
              entry_submission (see Feed Outage Recovery). Record
              gate_result='freeze', or 'breaker' when the active reason is
              'breaker_trip' — see Circuit Breaker below.
            - Ticker cap: COUNT(DISTINCT ticker) over today's
              live_positions rows with status IN ('pending','partial_open',
              'open','halted','exiting'). If this ticker is not already
              among them AND that count >= execution.max_tickers → skip.
            - Per-ticker cap: this ticker's own row count over the same
              statuses >= execution.max_positions_per_ticker → skip.
            - can_enter(ticker, current_hour, last_entry_hour,
              execution.entry_cooldown_minutes) → skip if False.
            Both caps read live_positions DB status, never the
            in_flight_orders runtime cache (R-2: the row is the SSoT);
            pending rows count as reserved capacity. last_entry_hour is the
            SUBMISSION time of this ticker's most recent entry attempt this
            session, including attempts that ended 'canceled' or
            'entry_rejected' — the same definition Warm Restart's cooldown
            restore rebuilds from the DB.
            # Config validation: execution.entry_cooldown_minutes * 60 >=
            # execution.cancel_after_seconds must hold. The submission-time
            # cooldown clock is what stops a re-flagged ticker from
            # double-submitting while its own order is still in flight, but
            # that protection follows from the two values' relative size —
            # it is not structural, and a short cooldown configured against
            # a long cancel window removes it.
            Whichever gate stops the candidate, write its reason to
            inference_log.gate_result on THIS candidate's own
            event='signal_fired' row — 'freeze' | 'cap_tickers' |
            'cap_per_ticker' | 'cooldown', and likewise 'not_tradable',
            'bar_integrity' and 'no_terms' (step i), 'sizing_zero' /
            'funds' (step ii) below. A candidate
            reaching submission in step iii records 'submitted'. If step
            ii's check_funds_available() call raises (the one network call
            in this sequence — timeout, connection error) rather than
            returning a verdict, record 'error': the candidate otherwise
            vanishes from every count with no record it was ever
            evaluated. Every signal_fired row therefore ends with a
            terminal state, so non-'submitted' rows are the complete
            record of candidates lost at the gates — they get no
            trade_log row at all (a broker rejection is different: that IS
            'submitted', and shows up as trade_log
            exit_reason='entry_rejected'). This row is written as a SINGLE
            INSERT the instant the gate sequence concludes, whichever way —
            never as an UPDATE to a row Inferencer wrote earlier (see
            Constraints for why this one event is LiveModeRunner's to
            write). See db_schema.md's inference_log for the value list and
            why this is a column rather than a new event. health_report.md's
            finding 19 reads these rows directly, so no in-session tally has
            to be threaded through to it.
        i.  is_tradable(ticker, db_conn) — execution_common.md gate.
            False (ticker_cik_map.rename_pending = TRUE OR
            quarantine_reason IS NOT NULL — N-2 replaced the old single
            `status` column with these two, and the gate needs BOTH clear)
            → skip, no order.
            Then, in the same step because it is the other per-ticker
            eligibility verdict: skip if this ticker is in the
            bar_integrity exclusion set (see Bar-Close Authority's
            Bar-Arrival Latency Measurement) → gate_result='bar_integrity'.
            A runtime set membership test, so it costs nothing to place
            here rather than earlier; unlike the freeze_reasons check in
            step 0, it is per-ticker rather than session-wide.
            Then, third and last in this step: skip if this ticker has no
            `live_ticker_terms` row for today — its margin rate was never
            acquired, or acquisition failed and the ticker was given up for
            the session (see "Per-Ticker Trading Terms")
            → gate_result='no_terms'. A SEPARATE gate evaluated at the same
            point rather than folded into `is_tradable()`: that function is
            a pure `ticker_cik_map` read, and mixing session state into it
            changes what it is.
        ii. quantity = execution_common.compute_position_size(
                balance=session_start_cash, fill_price=entry["p_entry"],
                mgnrt=<this ticker's live_ticker_terms.mgnrt>,
                # the persisted per-ticker rate, never re-queried here (see
                # "Per-Ticker Trading Terms"). The step-i terms gate is what
                # guarantees a value exists to pass.
                t_bar_volume=...,
                ticker_margin_used=..., total_margin_used=...,
                # margin, not notional: each open or pending row contributes
                # at its OWN pinned live_positions.entry_mgnrt, so an open
                # position's margin never moves retroactively
                position_size_cash_pct=config["execution"]["position_size_cash_pct"],
                position_size_vol_pct=config["execution"]["position_size_vol_pct"],
                per_ticker_share_cap_pct=config["execution"]["per_ticker_share_cap_pct"],
                exposure_cap_pct=config["execution"]["exposure_cap_pct"],
            )
            # p_entry, not a post-slippage price — same ordering rationale
            # as 09_backtest_engine.md's Entry Slippage Model (sizing must
            # precede simulate_entry_fill(), which takes quantity as input)
            proceed, quantity = execution_common.check_funds_available(
                quantity, entry["p_entry"], available_cash,
                use_all_cash=config["execution"]["use_all_cash"],
            )
            available_cash is COMPUTED, not observed:
                available_cash = AstkOrdAbleAmt * (100 / Mgnrt0) - @cost
            `AstkOrdAbleAmt` from `inquiry/balance-margin` (3 TPS,
            account-scoped, one call) queried fresh immediately before this
            call; `Mgnrt0` and `@cost` from this ticker's persisted terms
            (see "Per-Ticker Trading Terms"). Reading `AstkOrdAbleAmt1`
            directly would be exact and need no arithmetic, but that field
            exists ONLY on `able-orderqty`, so a direct read puts a 2 TPS
            per-ticker call back on the entry path — the burst the
            acquisition point was moved to avoid. Computation is preferred
            wherever an observation would sit inside the bar-close deadline.
            The computed figure runs ALWAYS-HIGH — `@cost` is subtractive and
            derived once, from possibly-stale conditions — so the bias is
            one-sided toward over-permitting; step iii's own comparison is
            what surfaces drift.
            The comparison itself stays NOTIONAL against NOTIONAL. The
            margin-unit form sizing uses (execution_common.md) does not
            extend here: `available_cash` is already leverage-inclusive, so
            multiplying the left side by `Mgnrt0 / 100` would apply the same
            factor twice.
            if not proceed: skip, no order
            # A reduced quantity from use_all_cash does NOT re-enter the
            # sizing caps above. All four sizing terms are monotone in
            # quantity and combined by min(), so any value at or below
            # `quantity` satisfies all four by construction: sizing sets the
            # ceiling, this gate only lowers it.
        iii. limit_price = None if config["execution"]["entry_order_type"] == "market" \
                 else (entry["p_entry"] * (1 + config["execution"]["entry_gap_value"])
                       if config["execution"]["entry_gap_type"] == "percentage"
                       else entry["p_entry"] + config["execution"]["entry_gap_value"])
             if config["live_mode"]["stage"] == "shadow":
                 # REAL-PATH-PARALLEL INCREMENTAL, as on the exit side
                 # (Position Manager Loop Step 3). simulate_entry_fill()
                 # consumes ticks forward from its anchor through
                 # cancel_after_seconds and those do not exist yet at this
                 # instant, so this branch OPENS a pending window instead
                 # of resolving one.
                 db_write: INSERT live_positions (run_id, ticker, date,
                     entry_bar, order_id=NULL, limit_price,
                     submitted_at=now, signal, status='pending',
                     fill_price=NULL, fill_second=NULL, quantity=NULL,
                     is_shadow=TRUE)
                 # Same SSoT-first write as the real branch below. The row
                 # carries is_shadow because live_positions rows OUTLIVE
                 # their session: without it a shadow row left open at
                 # close is adopted next day by a real session's Broker
                 # Reconcile and really liquidated (db_schema.md).
                 subscribe to this ticker's realtime trade stream NOW —
                     submission, not fill, is the trigger, the SAME rule
                     and the SAME zero-fill unsubscribe path as the real
                     branch. The former shadow-only simplification rested
                     on simulate_entry_fill() resolving synchronously,
                     which it cannot; a branch disappears here rather than
                     a simplification being lost.
                 anchor = submitted_at
                     # Live OBSERVES this instant; backtest COMPUTES its
                     # equivalent as entry_hour +
                     # execution.entry_fill_delay_seconds. Two expressions
                     # of ONE moment, never composed — adding the delay to
                     # submitted_at double-counts a wait live has already
                     # lived through (execution_common.md).
                 Once the 1-tick buffer covers anchor + cancel_after_seconds,
                 recompute STATELESSLY from the anchor:
                 weighted_avg_fill_price, filled, unfilled, status =
                     execution_common.simulate_entry_fill(
                         ticks_entry=<1-tick buffer, anchor forward>,
                         quantity=quantity,
                         entry_anchor_second=anchor, p_entry=entry["p_entry"],
                         buy_rate=config["execution"]["buy_rate"],
                         halts_df=<live_halt_episodes rows for this ticker>,
                         cancel_after_seconds=config["execution"]["cancel_after_seconds"],
                         limit_price=limit_price,
                     )
                 # ohlcv_entry is GONE — it had no reader. p_entry stays:
                 # its zero-fill fallback role is inert (a zero-fill entry
                 # is 'canceled', quantity=0, pnl NULL, reaching no summary
                 # statistic) but it is still the sizing input above.
                 # entry_order_type is simulated the same way in shadow as
                 # in backtest — see execution_common.md's price-gate logic
                 then transition the row exactly as the real branch does:
                 filled == 0 → 'canceled' and unsubscribe; filled > 0 →
                 'open' with quantity = filled, trade_log written COMPLETE
                 in a single INSERT (is_shadow=TRUE).
             else:
                 order_id = submit order via trading API: quantity=`quantity`,
                     order_type=config["execution"]["entry_order_type"],
                     limit_price=limit_price (omitted/ignored for "market")
                 # R-2: write the pending row FIRST (SSoT — see
                 # db_schema.md's live_positions), for BOTH order types —
                 # closes the submit-before-fill-response crash window even
                 # for market orders, and gives a pending limit order a
                 # durable record a crash-recovery reconcile can match by
                 # order_id (see "Session Restart (Warm Start)" below).
                 db_write: INSERT live_positions (run_id, ticker, date,
                     entry_bar, order_id, limit_price, submitted_at=now,
                     signal, status='pending', fill_price=NULL,
                     fill_second=NULL, quantity=NULL, is_shadow=FALSE)
                 # R-7: the order API's response confirms ACCEPTANCE of the
                 # submission, not a fill — fills arrive on the separate
                 # fill channel (WS account stream / REST order query). The
                 # former "market: real fill returned synchronously" branch
                 # was wrong for this vendor and is removed. The only
                 # rejection knowable at this instant is structural and
                 # price-independent (account restriction, ticker not
                 # permitted, malformed order); a market order's
                 # insufficient-funds-at-actual-price rejection and every
                 # limit rejection surface later on the fill channel.
                 subscribe to this ticker's realtime trade stream NOW —
                     acceptance of the submission, not a fill, is the
                     trigger (see Exit Architecture's "Primary — WebSocket
                     tick stream"; moved earlier than the fill it used to
                     wait for, so a position is tracked for the entire
                     window it carries risk, not just from its first fill
                     onward). Skipped if already subscribed (a second
                     position opening on an already-tracked ticker shares
                     the existing subscription, unchanged from before).
                 if submission rejected structurally:
                     no order_id exists and no live_positions row is
                     written — nothing was accepted. Log trade_log row:
                     exit_reason='entry_rejected', quantity=0,
                     fill_price=p_entry, exit_bar=entry_bar, and
                     reject_reason = the broker's own refusal text, stored
                     verbatim (see db_schema.md's never-opened family).
                     Counts as a cooldown attempt via step 5c.0's
                     last_entry_hour; excluded from fit_execution_params()
                     (see shadow_retraining.md).
                 else:
                     in_flight_orders[order_id] = {ticker, side: 'entry',
                         submitted_at: now, limit_price,
                         requested_quantity: quantity}
                     # runtime cache over the live_positions row above —
                     # BOTH order types, tracked to fill/reject/cancel by
                     # Position Manager Loop's "In-flight order tracking"
                     # OBSERVATION ONLY, and only for entry_order_type
                     # "limit", which has a quotable price to send. Call
                     # able-orderqty at limit_price and compare its
                     # AstkOrdAbleQty against the quantity just submitted,
                     # and its Mgnrt0 against this ticker's persisted rate.
                     # A mismatch on either raises a warning; NEITHER blocks
                     # — check_funds_available() above is the gate, and this
                     # runs AFTER dispatch precisely so it cannot sit ahead
                     # of the submission deadline live_scan_daily's
                     # entry_submit_late measures. Best-effort: dropped
                     # rather than awaited if the call is slow or fails.
                     # The Mgnrt0 half is what measures intraday invariance
                     # (api_contract_checklist.md) at zero extra cost, the
                     # response carrying both values.
                     # MARKET entries skip this entirely and keep the
                     # existing balance-query path unchanged, having no
                     # quotable price to submit"
```

---

## Recovery Lane and Promotion

The recovery lane is carryover and promotion's share of the non-reserved
pacing slots at t=500 and t=750 on both legs — a share, not the whole cap;
see the slot allocation above. It exists because two things need a bar
outside the prefix scan's reach, and neither may be served by injecting the
ticker at the scan head.

**Head injection would corrupt the scan itself.** The prefix scan's early
stop is sound ONLY because the list is ordered most-recently-fired-first.
Placing a ticker into positions 0-2 for a reason unrelated to firing makes
the head no longer the most recent firing, and the early-stop test becomes
meaningless. The lane also bounds what head injection could not: rotation
visits one ticker per cycle and a promotion lasts one bar, so up to 60
promotions can accumulate against 3 head slots — at the limit the scan is
starved of its own purpose.

**Priority within the lane is CARRYOVER FIRST, then promotion.** Carryover
is a live watchdog firing not yet serviced — the tail of a cycle where D
exceeded N. Promotion is recovery of a bar already lost. Promotions that do
not fit are DROPPED, not queued: dropping returns the ticker to exactly the
no-promotion baseline, so the failure mode costs nothing, whereas a
promotion queued past the bar it was issued for outlives its own reason.

**AT BAR CLOSE the lane yields** to the superset-K fetch, the same
precedence by which the exit ladder yields to `signal_time_rest`.

**The 5-second figure is a DESIGN TARGET, not a measurement.** It is the
budget for bar close -> fetch the superset -> infer -> submit, and the
middle term has never been measured: model inference time is unknown. The
figure IS `execution.entry_fill_delay_seconds` — one quantity, not two
mirrored ones: BacktestEngine models the entry fill at it while this loop
treats it as the reference its own submission is measured against. It is a
config key for exactly this reason. Treat every "5-second deadline" in this file as
naming that key's current value rather than a fixed property of the system;
`live_scan_daily`'s stage timings below are what will settle it. Lane
calls completing after the cycle boundary are harmless in any case:
carryover is late against a 1s cycle, promotion against a 60s bar.

**Promotion is a schedule change, never an eligibility change.** When the
rotation slot's second evaluation finds a detection on the PRECEDING
COMPLETED bar, that detection's t bar is the bar currently forming, already
tau seconds old at the moment of the finding. Entering against it is
unrepresentable in the t-bar-open-plus-5s fill model and would force a
change to `09_backtest_engine.md`. So the ticker is PROMOTED — given
recovery-lane priority for one bar — and the forming bar's own close is
then evaluated on the ordinary fast path, with `p_entry`, the fill model
and backtest parity all intact. One bar is lost; the next is guaranteed to
be seen.

Nothing about the condition is waived. D and E are cumulative across the
day and survive to the next bar unconditionally; B, C, F and G are
bar-local and need not, but volume clusters, so persistence into the
adjacent bar is the common case rather than the exception. If A-G are not
true on the next bar, promotion produces nothing, which is the correct
outcome.

Promotion adjudication runs `detect()` ONLY — not `infer()`. Running
inference here would reach the model and leave `entry_points` and
`inference_log` rows for a signal that was never entered. The count goes to
`live_scan_daily.promotions_total` instead, which is also the in-session
measure of how often the rotation cursor arrived too late in a bar.

**Under `execution.late_entry_enabled` the finding takes a second path
alongside promotion, not instead of it.** The candidate is put through
`infer()` and then `execution_common.md`'s residual-edge gate, which prices
the drift since the t bar's open against the edge the model still expects;
if it passes, the entry is taken at `p_now`. Inference therefore RUNS FIRST
on this path, which live can afford because these candidates are the small
rotation-detected subset adjudicated mid-bar, outside the 5-second
bar-close deadline. Promotion still happens either way — it is the fallback
for a candidate the gate rejects, and it costs nothing when the gate
accepts. With the flag false (the default) only promotion runs and this
paragraph is inert.

Both gate outcomes are counted into `live_scan_daily.late_entry_gate_pass`
and `_reject`. The reject count is the one that matters: it measures what
the gate costs in signals the model wanted, which is not observable any
other way once the flag is on, and it is the input for deciding whether
theta's quantile is set too high.

This is the one place the scan produces an entry, and it does NOT weaken
the eligibility rule in Constraints: what qualifies is still the full A-G
expression over a COMPLETED bar. What the late path changes is WHEN that
qualifying bar is observed and at WHAT PRICE it is taken.

---

## Bar-Close Authority

Judges, ONCE PER ELAPSED MINUTE rather than once per Watchdog Polling Loop
cycle, whether each ticker in the current working set (not the full tradable
universe — see the loop's own note on what the watchdog returns) has a bar
for the most recently fully-elapsed minute, at whole-second precision.
Scoped to the working set deliberately: nothing outside it is fetched at
all, so nothing outside it has anything to judge, and checking the full
active watchlist would report a missed deadline for tickers nobody is
fetching.

**A ticker is judged only once it has actually been ASKED.** Under the
prefix scan a cycle fetches N head slots plus one rotation position, not
the whole working set, so `last_bar_hour[ticker]` for a ticker not reached
this cycle is merely STALE — it carries no information about whether the
vendor has that minute's bar. Counting stale as `missed` would place most
of the working set in the missed set on almost every cycle and trip Feed
Outage's ratio test under entirely normal operation. The gate is
`last_fetch_time[ticker]`, which the latency measurement below already
maintains for its own purposes.

**The denominator is guaranteed by the rotation cursor, not by luck.** M=1
advances one position per cycle against a working set capped at 50, so
every ticker is fetched at least once per ~50 cycles — inside one bar
period. The rotation slot is DEDICATED and never competes with the head
slots, so this holds however deep D runs in a given cycle. That is what
makes a per-minute judgement well-populated where a per-cycle one would
not be, and it is why the accumulation window below is exactly one minute.

The working set is CUMULATIVE, so it carries tickers that fired hours ago
and are legitimately quiet now. `missed` therefore mixes two causes — a
feed fault, and a minute with no trade in a thin name — and the second is
concentrated in the older, lower-ranked entries. Bar-Close Authority still
judges the whole working set, which keeps finding 26's latency sample
broad; it is Feed Outage's ratio test that is scoped to the recent end of
the list, for exactly this reason (see that section).

```
expected_minute = floor(now to the minute) - 1 minute   # the most
    # recently fully-elapsed minute; NOT the current, still-forming one
deadline = (end of expected_minute) + bar_close_grace_seconds

ACCUMULATION WINDOW for expected_minute = [deadline, deadline + 60s)

  At each Step 2a fetch falling inside that window, classify THIS ticker
  for expected_minute — first classification wins, so one ticker
  contributes at most one verdict per minute:

      'ok'      the response carries a bar for expected_minute
                (last_bar_hour[ticker] >= expected_minute)
      'missed'  it does not, and the fetch was issued after the deadline,
                so the vendor had its full grace window before we looked

  A ticker the scan never reaches inside the window is classified NEITHER.
  It is absent from the denominator, not counted as missed — the
  difference between "we asked and it was not there" and "we did not ask".

  At the window's close: evaluate Feed Outage trigger condition 2 over the
  classified set, emit finding 26's samples for the minute, then discard
  the classification and open the next minute's window.

  # last_bar_hour[ticker]: this ticker's own most recent bar hour as
  # actually returned by Step 2a, tracked per ticker across cycles —
  # read directly off the fetch response, not derived from
  # calculators[ticker], since that already absorbs Step 3's replay
  # and would hide exactly the lag being measured here
```

**The cost of the window is detection latency on condition 2 only.** A
systemic bar-feed failure now surfaces up to a minute after it starts,
where the per-cycle test could in principle have caught it within a cycle.
Condition 1 — an explicit connection or API-level failure — is unaffected
and still fires on the cycle it occurs, and that is the fast path for a
real outage. Condition 2 was always the slower "systemic signal even
without a hard connection error" detector, and under the prefix scan a
per-cycle version of it is not merely noisier but structurally wrong: it
would be reading tickers nobody asked about.

`bar_close_grace_seconds` (config, seed value — see
api_contract_checklist.md's T-13: how promptly the vendor typically has a
minute's bar ready is itself unverified) is why 1.0s of clock skew (see
Session Start Probes' `clock_check`) is the boundary at which a clock
fault starts impersonating a feed outage: the deadline above is wall-clock
derived, so a skewed local clock shifts every ticker's `expected_minute`
identically and simultaneously, not just one.

Individually, one ticker in `missed` is routine — this is exactly the case
Feed Outage Recovery (below) leaves un-frozen, caught up whenever that
ticker's next bar arrives via Step 3's ordinary multi-bar replay.
`missed` becomes a candidate systemic signal only in aggregate — see Feed
Outage Recovery's trigger condition 2, next.

### Bar-Arrival Latency Measurement (T-13)

`bar_close_grace_seconds` is a seed value because nobody has measured what
it is guarding against. This measurement is what turns it into an observed
quantity, satisfying api_contract_checklist.md's T-13 without a separate
verification exercise. It measures only bars that DO arrive; `missed` above
covers the ones that do not, and the two are reported together (finding 26)
because neither is interpretable alone.

Measured at Step 2a, where `last_bar_hour[ticker]` is updated — NOT in Step
3 — for the same reason `last_bar_hour` is read off the fetch response
rather than from `calculators[ticker]`: Step 3 absorbs multi-bar replay and
would hide the very lag being measured. Two further properties fall out of
that placement: the loop's hottest path (Step 3's `on_bar_close()` replay)
is untouched, and at most one sample per ticker per cycle is structurally
guaranteed, so a single slow bar cannot be counted repeatedly.

**The prefix scan shifts this curve's POPULATION, and the shift is not
neutral.** `e = min(delta, d)` admits a sample only when the ticker was
fetched recently enough to bound the overstatement, so head-slot tickers —
fetched every second, `delta` about 1s — supply nearly all admitted
samples, while a ticker reached only by the rotation cursor carries
`delta` up to ~50s and is admitted only on the rare pass that happens to
land within `bar_latency_max_error_seconds` of its bar close. The curve
therefore describes the vendor's latency ON ACTIVELY FIRING TICKERS. That
is the opposite of the bias this measurement already guards against by
admitting timestamp-discontinuous samples: thin names are the ones a
vendor is slowest on, and they are now the ones least represented, so
`bar_close_grace_seconds` calibrated from this curve runs OPTIMISTIC for
exactly the tickers most likely to breach it. The one-sided-error posture
already stated for this key — seed it conservative (higher) — is what
absorbs the gap until `excluded_no_prior_fetch` / `excluded_both_wide`
counts show how large it is; those two counters are now the measure of the
shift, not merely of rejected samples.

```
At Step 2a, for each candidate whose fetch returned a newer bar than
last_bar_hour[ticker] held:

  d     = now - (newest bar's hour + 60s)     # whole seconds
  delta = now - last_fetch_time[ticker]       # absent on a first poll
  e     = min(delta, d)                       # this sample's error bound

  if d < 0:
      negative sample — see "Premise violation" below. Not a latency
      sample; e is meaningless when d is.
  elif e <= live_mode.bar_latency_max_error_seconds:
      admit: bucket floor(d), class 'poll_continuous' if
      e <= poll_interval_seconds else 'wide_error'
  else:
      reject: count as excluded_no_prior_fetch (no delta) or
      excluded_both_wide (delta and d both large)

  last_fetch_time[ticker] = now      # every successful fetch, admitted
                                     # sample or not
```

**Why `e`, and why it needs the prior fetch.** `d` overstates the vendor's
true publish latency by however long the bar sat unobserved before we asked.
Knowing the ticker was fetched `delta` ago bounds that overstatement at
`delta`; on a first poll only the bar's own close bounds it, at `d` itself.
Taking the smaller of the two is what keeps a sample honest, and it is why
`last_fetch_time` exists at all: without it a bar that had been ready for 40
seconds before the first poll reached it would be recorded as 40 seconds of
vendor latency. The bound matters because the error is bounded at 60s
otherwise — a fetch returns only the newest bar, so the overstatement does
not grow with how long the ticker was away, but 60s dwarfs the 5s value
being calibrated.

Deliberately NOT filtered on bar-timestamp continuity, which was the obvious
alternative: a thin ticker that gets no zero-volume bar for a minute (see
Feed Outage Recovery's trigger condition 2 on exactly this uncertainty)
produces a bar whose timestamp skipped a minute but which arrived perfectly
on time. Rejecting those would discard valid samples from precisely the
thinnest tickers — the ones a vendor is slowest on — biasing the curve
optimistic. Such samples are admitted and separately counted
(`bar_gap_samples`), which is also what distinguishes structural
non-publication (large gap, small `d`) from real vendor lag (large gap,
large `d`).

`last_fetch_time` is runtime-only, same convention as `last_halt_state`: a
warm restart costs at most one skipped sample per ticker, on its first
post-restart fetch. Every exclusion case — session first appearance,
candidacy re-entry, post-outage recovery, an individual failed fetch — skips
exactly one cycle for that ticker, not a window.

Counters accumulate in memory as the UN-FLUSHED DELTA ONLY and are flushed
additively into `bar_latency_daily` (db_schema.md, which carries the merge
rule) through `db_write()`, once per minute at an in-minute offset rather
than on the minute boundary — the boundary is where bar arrival, Step 3's
replay, and `infer()` all land. The delta is zeroed after each flush,
including a forced one, so the periodic timer is reset by a forced flush
too; otherwise the next scheduled flush writes an empty delta. Runs
unconditionally in shadow mode: the bar/price channel is identical in both
modes, and this measures that channel, not the order path.

**Premise violation (`d < 0`).** A negative sample means a bar exists for a
minute that has not finished, which contradicts the `expected_minute`
reasoning this whole section rests on. Two causes, and the `|d|`
distribution in `bar_latency_daily` separates them: values near 60s mean the
vendor timestamps bars by CLOSE while this spec assumes OPEN, in which case
every `missed` verdict and every Feed Outage condition-2 evaluation is off by
one bar; small scattered values mean premature publication of a still-forming
bar, or a clock fault. Either way it is a design error, not a data blip, so
it is recorded per event via `utils.record_health_event()` and reported as
`health_report.md` finding 27, event-driven at abort severity rather than
waiting for session end.

On detection, in this order:
```
1. increment the negative counter (in-memory delta)
2. add ticker to the bar_integrity exclusion set
3. if len(exclusion set) >= live_mode.bar_integrity_freeze_ticker_count:
       add 'bar_integrity' to freeze_reasons, scope entry_submission
4. forced flush of bar_latency_daily (synchronous, via db_write)
5. record_health_event() + health_report.py invocation, subset-scoped to
   finding 27 (see health_report.md's Invocation Contract)
```
State changes precede the flush so the alert cannot report an exclusion set
missing the very ticker that triggered it. The flush happens whether or not
the alert itself is suppressed — the table is state, suppression is
transport. The health_report call returns as soon as it has written its log;
delivery is its own background concern (see health_report.md), so this loop
is never blocked on a socket.

**Why the exclusion is per-ticker and entry-only.** A prematurely-published
bar is fed to `on_bar_close()`, whose CONTINUOUS-indicator updates are
incremental and therefore irreversible for the session; that corrupted state
reaches new entries through `infer()`. It does NOT reach existing positions:
tp/sl runs off the WS tick stream (Exit Architecture), not the bar channel,
so open positions keep being managed and exited normally. Corruption also
cannot spread between tickers, since every affected structure is per-ticker.
Blocking new entries for that ticker alone is therefore sufficient, and the
session continues.

Rebuilding the corrupted calculator via `scoped_recompute()` (the mechanism
corporate-event handling uses above) is deliberately NOT done: with entries
for that ticker already blocked, a rebuild buys nothing and adds a recovery
path whose own correctness would need establishing.

The promotion at step 3 exists because a timestamp-convention mismatch
affects every ticker, so per-ticker exclusion would trip the whole watchlist
one ticker at a time while leaving tickers not yet flagged unprotected. The
promoted freeze blocks entry submission only and does not clear within the
session — there is no way to verify recovery from corrupted incremental
state, the same reason a breaker trip does not auto-clear. The exclusion set
itself is runtime-only; after a warm restart a ticker is re-excluded on its
next negative sample.

---

## Feed Outage Recovery

Distinct from the routine per-ticker catch-up already built into the
Watchdog Polling Loop (Step 3's multi-bar replay) and Bar-Close Authority —
those handle an individual ticker going quiet for a while, gracefully and
without any freeze. This section covers the qualitatively different case:
LiveModeRunner's own connectivity to the trading API is lost. The
watchdog is NOT covered here: it is reached as a module call, which has no
connection to lose, and its own failure mode is an empty working set,
handled by finding 29 without a freeze.

**Trigger** (either condition):
- An explicit connection/API-level failure (exception, timeout, non-200
  response) from the trading API, or
- More than 50% of the working set's MOST-RECENTLY-FIRED
  `live_mode.min_watchlist_size` tickers are classified `missed` in
  Bar-Close Authority's per-minute accumulation window, evaluated once at
  that window's close — a systemic signal even without a hard connection
  error. The ratio's denominator is the CLASSIFIED tickers among that MRU
  slice, never the slice itself: under the prefix scan a ticker the scan
  did not reach inside the window has not been asked, and counting it
  against the feed would manufacture the very freeze this condition is
  supposed to detect. The rotation cursor is what keeps the denominator
  from thinning — it reaches every working-set ticker within a bar period
  regardless of scan depth.
  Scoped to the recent end of the list because the working set is
  cumulative: an unscoped ratio counts tickers that fired hours ago and
  are legitimately quiet, which in this low-liquidity universe would
  produce false freezes as a matter of course. MRU rank is information
  from OUTSIDE the feed, which is why this scope does not fail the way a
  recent-BAR-activity scope would — that would empty its own population
  exactly when the feed dies. `min_watchlist_size` accordingly serves two
  roles now: the sample size, and the minimum working-set length below
  which this condition does not evaluate (condition 1 above remains
  the only detector for that cycle). Both `min_watchlist_size` and
  `bar_close_grace_seconds` are seed values, deliberately set conservative
  (harder to trigger) until measured — the vendor's actual quiet-minute
  behavior (whether a thin ticker with zero trades gets a zero-volume bar
  at all, or no bar) bears directly on how large a "normal" miss rate can
  be, and this condition is set with that in mind.

**Recovery procedure**, on detecting the trigger:
```
1. Freeze: add 'feed_outage' to freeze_reasons.

   freeze_reasons is a SET of active reasons, each carrying its own scope
   — not a single session-wide boolean. A boolean cannot represent two
   overlapping causes, and clearing it on recovery from one would silently
   release the others (e.g. a feed outage during a warm restart's cache
   reload would unfreeze entries the restart still needs held):
     'feed_outage'    → blocks entry_submission AND exit_submission
     'restart_warmup' → blocks entry_submission only (Warm Restart step 4)
     'breaker_trip'   → blocks entry_submission only (Circuit Breaker, R-4)
   A gate blocks whenever ANY active reason covers its scope, so releasing
   one reason leaves the others in force. Watchdog Polling Loop step 5c.0
   checks entry_submission; Position Manager Loop step 3 checks
   exit_submission. Exit *evaluation* is never frozen — it keeps running
   and accumulating bars/ticks; only order *submission* is held.

2. Reconnect: retry the trading API connection
   (backoff policy TBD — not specified here).

3. Catch up: once reconnected, this does NOT require a separate vendor
   endpoint — the trading API's existing "always returns full range since
   last successful query" behavior (already relied on elsewhere — see
   Watchdog Polling Loop Step 2a) means the
   very next ordinary bar-fetch call for each ticker naturally returns
   everything missed during the outage. Under the prefix scan that call
   arrives per ticker as the scan reaches it, not for the whole watchlist
   at once — head slots within a cycle or two for anything the watchdog is
   firing on, and the rotation cursor within one bar period for the rest —
   so catch-up is PROGRESSIVE, bounded by the rotation period rather than
   simultaneous. This costs nothing: a ticker not yet caught up is simply
   one that has not been re-evaluated yet, which is the ordinary steady
   state of the scan in any case. Step 5's exit re-evaluation is unaffected
   for a different reason than it once was: it runs on TICKS, so the scan's
   bar catch-up is not its input at all, and Position Manager fetches no
   bars now. The EXISTING
   Step 3 multi-bar replay loop (calculators[ticker].on_bar_close() called
   once per missed bar, in order) handles the catch-up with no new
   mechanism — the outage is, from the calculator's point of view, just an
   unusually long version of the same gap Step 3 already tolerates.
   (A gap long enough to exceed the trading API's own retention/buffer is
   a true, unrecoverable data gap — not engineered around here; falls
   through to whatever the vendor's own missing-data convention is.)

4. Reconcile: run the Broker Reconcile shared procedure (R-3 — see
   "Broker Reconcile (shared procedure)" below). The feed-outage-specific
   case (broker shows a position closed that LiveModeRunner still tracks
   as open → an exit order placed just before the outage evidently filled;
   adopt the broker's fill as authoritative, not simulated/estimated) is
   part of that shared procedure. `feed_gap_exit` (step 5 below) still
   applies only to positions that remained open through the outage.

5. Re-evaluate exits: for each position that remained open through the
   outage, apply the 2-PRINT GUARD over the now-caught-up TICKS once before
   unfreezing — the same form Warm Restart step 3c already uses for the
   identical question. NOT track_price_breach(), which is backtest-only as
   of R-2: the call named here was the stale side of that change, against
   two declarations in this file saying so. Any position whose exit condition
   would have triggered *during* the gap: exit at market immediately (not
   at the historical breach point — that price is stale by now), logged
   with `exit_reason='feed_gap_exit'` (see db_schema.md's trade_log) so
   this trade is excluded from ordinary strategy PnL attribution — it
   reflects outage handling, not strategy performance.

6. Unfreeze: remove 'feed_outage' from freeze_reasons — only this reason.
   If another reason is still active (e.g. 'restart_warmup'), its own
   scope stays in force and normal operation resumes only for what it
   does not cover.
```

## Circuit Breaker (R-4)

Protects against a failure mode stop-losses structurally cannot catch. A
stop-loss handles one trade going the wrong way, independently each time. It
does nothing about the same wrongness repeating: at
`position_size_cash_pct: 0.05` and `stop_loss_pct: 0.03`, each stopped trade
costs ~0.15% of the account, so a full book of 20 stopping out is ~3% — with
every stop-loss behaving exactly as designed. What the breaker targets is a
broken model or feature pipeline emitting bad entries continuously.

**Three thresholds** (execution_common.md; all default 0 = no limit):
`intraday_loss_limit_pct`, `consecutive_loss_limit`, `entries_per_hour_limit`.
They are deliberately staggered along the failure timeline — entry rate spikes
first, before any loss exists; then losing exits run; realised loss accumulates
last.

**On trip:** add `'breaker_trip'` to freeze_reasons, scope `entry_submission`
only, and db_write `live_session_state.breaker_tripped_at = now` (see
db_schema.md — this is what lets the trip survive a crash; see Warm Restart
below). Open positions keep running their own tp/sl — force-flatten is
rejected (market-impact risk for no gain). Does not auto-clear within the
session.

**State is updated on the events themselves, not polled.** All three quantities
change only at moments the system already handles: realised loss and the
consecutive counter at position close, entries-per-hour at submission — both
restricted to `is_shadow = FALSE` rows for a live session's own counters (a
shadow evaluation, below, tracks its own is_shadow=TRUE counters separately
so the two never mix). So the entry gate reads a value that is current by
construction. Evaluating on the Position Manager Loop's cadence and having
the 1s gate consume the last verdict was considered and rejected — it admits
up to five gate cycles between the condition becoming true and being
enforced. The one delay that remains is inherent: a close is not known until
the loop observes it, which no evaluation cadence changes.

**Consecutive counter** counts consecutive LOSING exits (`pnl_pct < 0`) and is
reset only by a profitable exit — see execution_common.md for why the
stop-loss-only reading leaves a hole. Operational-family and never-opened rows
are skipped entirely, incrementing nothing and resetting nothing. Where several
positions close in one cycle, order by `exit_bar`, then ticker alphabetically —
the same tiebreak backtest already uses.

**Shadow mode: evaluate, do not enforce.** A real trip would truncate exactly
the bad-day data shadow exists to observe. Because nothing is enforced,
`gate_result` still records `'submitted'`, so a would-have-tripped event CANNOT
be read from `gate_result` — it is reported as its own health_report finding.
Note this is the opposite policy from the position caps, which ARE enforced in
shadow: caps shape what live would have done, the breaker only reacts to it.

**On trip, health_report.py is invoked immediately** — a third trigger beside
session start and session end — at `abort` severity, so it reaches even
`alert_level: "abort_only"`. No new alerting infrastructure: the Discord and
email channels already exist and were simply never wired to a mid-session
event.

## Broker Reconcile (shared procedure) — R-3

One implementation, three call sites: (a) every Session Lifecycle start,
cold start included (Session Lifecycle Step 1d), (b) Feed Outage Recovery
step 4, (c) Warm Restart step 1 (R-2, below). Compares the trading API's
view (open orders + open positions) against `live_positions` rows.

**Orders** (broker open entry orders):
  - Match to `live_positions` rows with `status='pending'` by `order_id`.
  - Cancel (unknown staleness — conservative); the matched row transitions
    to `'canceled'` via the single canceled-transition point (see
    "Pending limit-entry tracking," R-2) — the same idempotent path an
    ordinary cancel-after-timeout uses, so re-running this step (e.g. a
    re-crash mid-recovery) is safe and cannot double-log.
  - A broker order with no matching pending row → record_health_event(
    finding_name='unknown_broker_order_or_position', detail={call_site,
    kind: 'order', order_id, ticker}) — see health_report.md finding 12.

**Positions** (broker open positions):
  - Match to `live_positions` rows (`status` `'open'`|`'halted'`).
  - Matched row's `date` is a PRIOR trading day → Unified Overnight Policy
    (below).
  - Matched row's `date` is TODAY (only possible at Feed Outage / Warm
    Restart, never at cold start) → keep managing normally (Feed Outage)
    or adopt exit-only (Warm Restart, per R-2).
  - Broker position with NO matching row → adopt conservatively; entry
    time is unknown, so treat as overnight (immediate liquidation, below).
    Also record_health_event(finding_name=
    'unknown_broker_order_or_position', detail={call_site, kind:
    'position', order_id: NULL, ticker}).
  - `live_positions` row `status='open'` with NO broker position →
    **reconcile_ghost**: transition the row to `'closed'`, trade_log
    `exit_reason='reconcile_ghost'`, `quantity=0`, PnL-excluded — there was
    no real position, so logging it as `stop_loss` etc. would fabricate a
    trade that never happened.
    **SUPPRESSED for `is_shadow=TRUE` rows.** A shadow row never had a
    broker counterpart, so an open row with no broker position is its
    EXPECTED state rather than a ghost. Without the exclusion this
    procedure destroys open shadow positions at two of its three call
    sites — Feed Outage step 4 MID-SESSION and Warm Restart step 1 — and
    does so silently, the resulting rows being PnL-excluded. The
    BROKER-side branches above are NOT suppressed in shadow: a broker
    order or position found while `stage == "shadow"` is a genuine anomaly
    (a config error, or residue from a prior real session) and
    'unknown_broker_order_or_position' stays its correct outcome. Shadow
    rows also carry no `order_id`, so the pending-order branch could never
    match them — stated here as intended rather than reached by a match
    failure.

**Cold-start note:** at cold start nothing has opened today, so ANY broker
position found is by definition an overnight orphan — it always routes to
the Unified Overnight Policy below.

**Health-event writes (R-9).** Both branches above call
`utils.record_health_event()` with `write_fn=db_write` — never a raw
`db_conn`, which would bypass `write_lock` (see utils.md). `call_site` is
one of `'session_start'` | `'warm_restart'` | `'feed_outage'`, so one
implementation serving three call sites stays distinguishable in the
table. Because this procedure is idempotent and re-runnable after a
mid-recovery re-crash, a repeat run legitimately re-records the same
condition; the events are timestamped occurrences, not a deduplicated set.
Per db_schema.md's standing rule, a failure of either write is swallowed
and must not abort the reconcile.

## Unified Overnight Policy — R-3

Any adopted position dated to a PRIOR trading day (halt-through-close,
unfilled/rejected EOD exit, crash orphan, or an unmatched broker position
of unknown date) is liquidated as soon as the ticker is tradable, at
whichever order type the current session phase permits (execution_common.md
— market is regular-session-only), `exit_reason='overnight_exit'`,
PnL-excluded. `exit_date` is the
date of that liquidation, NOT the row's `date` (the entry date) — these are
by definition different for this label, and may differ by more than one day
across a weekend, holiday, or multi-day halt. See db_schema.md's
`trade_log.exit_date`.

**`is_shadow=TRUE` rows are EXCLUDED from this policy's real
liquidation.** A shadow row carried across a day describes a position that
never existed at the broker, so submitting a market exit for it would place
a live order on the strength of a simulation. It is closed with the same
`overnight_exit` label and PnL exclusion, but by row transition alone with
no order submitted. This is the failure `live_positions.is_shadow` exists
to prevent, and it is reachable precisely because the incremental fill form
allows a shadow exit to still be unsettled at the close.

**Mechanism** — no special executor needed: the adopted position enters
the ordinary Position Manager Loop, where `execution.max_hold_bars` was
exceeded long ago, so the very first evaluation fires an immediate market
exit; only the label is special-cased for attribution.

**Label split vs. `restart_gap_exit` (R-2)** — same underlying mechanism
(carried position, max-hold exceeded, immediate liquidation, PnL-excluded),
distinguished by DATE for diagnosis only:
  - carried within the SAME trading day (a crash gap) → `restart_gap_exit`
  - carried ACROSS a trading day (a prior-day date)   → `overnight_exit`
Both are PnL-excluded, so strategy attribution is unaffected by which
label a given carried position gets — the split exists purely so the two
different root causes (same-day crash vs. multi-day carry) stay
distinguishable in `health_report.md` and `trade_log`.

---

## Session Restart (Warm Start) — R-2

Detected at startup: today's `live_session_start` row is `status='running'`,
there is no `live_session_end` row, and now is within session hours on a
trading day. (Else: ordinary cold start.) Distinct from Feed Outage
Recovery above — that handles connectivity loss within a still-running
process; this handles the process itself having died and restarted.

The procedure is idempotent from step 1 — a crash during recovery leaves it
safely re-runnable. Throughout, the `live_session_start` marker stays
`'running'` (only a clean resume/shutdown flips it — see Session Shutdown),
so a re-crash during recovery re-enters warm restart on the same signature.

```
1. Broker Reconcile (shared procedure — see R-3's "Broker Reconcile" for
   the fully general form; the behaviors relevant at this call site):
     - Open entry orders (broker): match to live_positions rows with
       status='pending' by order_id. Cancel all pending orders (unknown
       staleness — conservative); each matched row transitions to
       'canceled' via the single canceled-transition point (see Pending
       limit-entry tracking) — same idempotent path an ordinary expiry
       uses, so re-running this step is safe. A broker order with no
       matching pending row -> "unknown broker order" health_report finding.
     - Open positions (broker): match to live_positions rows
       (status 'open'|'halted'). A broker position with no matching row ->
       adopt conservatively and liquidate immediately (entry time
       unknown). A row with status='open' but no broker position ->
       reconcile_ghost (see R-3). Prior-trading-day positions -> overnight
       policy (R-3).

2. Restore session_start_cash from live_session_state (NOT re-queried —
   see Session Lifecycle Step 1c). Per-ticker trading terms are NOT
   re-acquired either: today's `live_ticker_terms` rows are read back as
   they stand, and each open position's own rate comes from its
   `live_positions` row rather than from either that table or the vendor
   (see "Per-Ticker Trading Terms"). A restart is the WORST burst case —
   many tickers wanted at once against a 2 TPS endpoint — so persistence is
   load-bearing here rather than incidental. The in-memory give-up set does
   NOT survive, by design.

3. Start Position Manager exit-only immediately — exits are
   indicator-independent (WS/REST tick driven; no Eager-Pool state needed
   to manage an exit). For each position with an active subscription
   pre-crash — status IN ('pending','partial_open','open','halted'); all
   four now carry a live tick subscription, since Exit Architecture's
   subscribe point moved to order submission rather than first fill, not
   just the fully-open pair this step originally covered:
     a. WS re-subscribe FIRST (see Exit Architecture), THEN
     b. REST gap-fill for the crash gap, deduped against the WS buffer by
        the global tick-dedup rule (see utils.md's stitch_ticks()). Doing
        (a) before (b) guarantees no realtime tick is lost in the seam.
     b2. Resolve any live_halt_episodes row left OPEN by the crash
        (halt_end IS NULL) by REPLAYING the tick-rate heuristic over (b)'s
        gap-fill ticks: no ticks, or tick_rate_per_min below
        halt_heuristic_tpm, across the gap leaves the interval open, and
        the first point crossing the threshold closes it at the ACTUAL
        resumption time. Neither assumption was acceptable — treating the
        interval as continuing lets the first post-restart cycle close it
        at `now`, inflating the duration by (now − actual resumption),
        while closing it at the last pre-crash observation deflates it by
        the same amount. The STATE self-corrects either way; the DURATION
        does not, and duration is why the table exists. Not new machinery:
        it applies the computation the loop would have performed to data
        that arrived late, the same idiom as Watchdog Step 3's multi-bar
        replay. The half-tape WS entitlement is harmless here because
        replay and live path apply the identical test, so both are biased
        the same way rather than acquiring an independent error term.
        Ordered between (b) and (c) because (c) reads its result.
     c. status IN ('open','halted'): apply the 2-print guard over gap-fill
        + live ticks; if a tp/sl breach is found within the gap, liquidate
        at current price, exit_reason='restart_gap_exit' (see
        db_schema.md). If the position's ELAPSED MINUS HALTED hold already
        exceeds config["execution"]["max_hold_bars"], skip retro-detection
        and liquidate immediately (still 'restart_gap_exit') — the same
        rule Position Manager Loop Step 2 applies in steady state, rather
        than the raw elapsed hold this step used to read.
        status IN ('pending','partial_open'): no `fill_price` yet, so no
        breach evaluation — same as the ordinary (non-restart) guard in
        Exit Architecture. Gap-filled ticks are retained by the dedup
        layer only; this order's own fill state is resolved by Step 1's
        Broker Reconcile, independently of this tick catch-up.

4. Entries stay frozen — add 'restart_warmup' to freeze_reasons
   (entry_submission scope only, so step 3's exit-only management above
   keeps submitting exits) — while the indicator cache reloads: if today's Eager-Pool backup exists in
   indicator_cache, restore each ticker via load_from_db() (skipping the
   historical_bars recompute) then replay today's bars via on_bar_close();
   if no backup exists (Eager Pool never ran this process), fall back to
   full session_start_compute(). Health Gate 1 re-runs unchanged (its
   premarket inputs are unchanged since morning). indicator_cache is NOT
   purged on this path (see db_schema.md).
   Idempotency of this step specifically (re-crash during recovery):
   load_from_db()'s result is the fixed Eager-Pool-only snapshot; the
   on_bar_close() replay that brings it current happens entirely in
   memory and is never itself written back to indicator_cache. So a
   re-crash mid-replay loses only that in-memory progress, not the
   backup it started from — the next restart attempt reloads the SAME
   snapshot and replays from scratch again, reaching the same end state.
   This makes step 4 all-or-nothing per ticker with no partial-state
   corruption possible, and needs no transactional handling beyond that.

5. On cache reload complete: clear 'restart_warmup'; entries resume.

5a. Scan state is deliberately NOT restored, and the omission is safe
   rather than overlooked. `last_bar_hour`, `last_fetch_time` and the
   rotation cursor all start empty: Bar-Close Authority's per-minute window
   simply leaves an unfetched ticker unclassified rather than counting it
   missed, the cursor restarts at position 0 and covers the list within a
   bar period, and the first pass's latency samples fall into the existing
   `excluded_no_prior_fetch` class. The deficit window rule then sees no
   bars held for any ticker and issues each one a backfill as its own head
   slot — the same path a cold ticker takes after open, at a moment when
   the scan has nothing else to do for those tickers anyway.

6. Cooldown restore: per ticker, last entry-attempt time = max over
   today's live_positions rows in {pending, open, canceled, closed} (read
   from DB status — the SSoT — not any in-memory cache) and trade_log rows
   with exit_reason IN ('entry_canceled', 'entry_rejected'). The
   pending_entries in-memory dict is rebuilt from live_positions rows with
   status='pending'.

7. Circuit breaker restore (R-4): recompute all three counters from
   today's `trade_log`, `is_shadow = FALSE` only (a shadow evaluation's
   counters, if this process also ran shadow, are recomputed the same way
   but restricted to `is_shadow = TRUE` and kept separate — see Circuit
   Breaker). Realised loss and the consecutive-loss counter are exact:
   both are cumulative or determined by exit order, neither decays with
   time. entries_per_hour is a ROLLING window and DOES decay during
   downtime, so recomputing it alone can under-report a real trip after a
   long outage. Recomputation therefore is NOT sufficient by itself:
     if live_session_state.breaker_tripped_at IS NOT NULL:
         restore 'breaker_trip' to freeze_reasons regardless of what the
         three recomputed counters show — this is the only path that
         upholds "no auto-clear within a session" (Circuit Breaker) across
         a crash. The recomputed counters still matter: they are what
         health_report.md reports going forward, and they are what a
         second, independent trip condition (post-restart) would be
         evaluated against.
```

## Watchlist Append

Called when a ticker appears in the watchdog working set for the first time
this session, **or** is returning after one or more cycles absent (i.e. it
had been evicted past the working set's 50-entry cap and has since fired
again — eviction is the only way a ticker leaves the set, and is routine
while the set is full). Both cases use the same procedure — a ticker's
absence from the working set for any stretch of time is not tracked or
treated specially; the calculator's bar history simply catches up to
however much was missed, same as it does for the ordinary first-time case:

```
When ticker X appears in candidates (first time this session, or again
after any gap):

  if ticker_X not in calculators:   # only possible when indicator_cache_mode == "db"
                                     # and this is genuinely the first appearance
      # Restore from DB, reload session_stats
      calc_X = CachingIndicatorCalculator(config)
      calc_X.load_from_db(db_conn, today_date, ticker_X)
      stats_X = load_session_stats(
          db_conn, ticker_X, today_date,
          n_sessions=config_n_sessions,
          delta_minutes=config_delta_minutes,
          session_mode=config_session_mode,
          closes=closes,
      )
      calc_X.set_session_stats(stats_X)
      calculators[ticker_X] = calc_X
  # indicator_cache_mode == "memory", or ticker_X already restored above:
  # calc_X already in calculators (eager pool, or restored on a prior
  # appearance this session) — no restore action needed either way.

  # Bar replay itself (session_start → current bar-1, or last-known-bar →
  # current bar-1 on a re-appearance) happens via Watchdog Polling Loop's
  # own Step 3 loop once this ticker's bars are fetched this cycle — not
  # duplicated here. Step 3's loop is written to handle any number of
  # missed bars uniformly, so first-appearance (potentially many bars since
  # session start) and re-appearance (typically few bars) are the same code
  # path with no special-casing.

  # Per-ticker trading terms are acquired HERE, on genuinely-first
  # appearance — see "Per-Ticker Trading Terms" below. The
  # live_ticker_terms row's existence for (ticker, today) is the guard, so
  # a re-appearance after eviction is a no-op with no new tracking state.
```

---

## Per-Ticker Trading Terms

The margin rate is PER TICKER and has NO account-level counterpart.
`inquiry/able-orderqty` returns `Mgnrt0` (적용증거금률), the effective
requirement, and it is taken AS-IS. Its neighbours are not a second rate to
combine with it: `Mgnrt` is the instrument rate, and `OtptItemNm1` is a LABEL
STRING — measured as "100% 계좌" | "컬러증거금" — reporting whether a
per-ticker 100% override is set at account level, not a number. `Mgnrt0`
already incorporates that override and reads 100 when it is off, so the
correction `100 / Mgnrt0` is the identity for a leverage-free account and no
branch is needed anywhere downstream.

`AstkOrdPrc` is a required request field but a DUMMY here: the rates do not
depend on it.

Background: the PDT designation and its $25,000 floor were removed from FINRA
Rule 4210 effective 2026-06-04, replaced by a risk-based INTRADAY margin
standard tied to live exposure. That is why this is asked rather than
encoded — the requirement moves with position state, brokers phase the new
framework in on their own schedules through 2027-10-20, and house rules may be
stricter than the regulatory floor. Modelling the rule in code would be wrong
for some broker at some date; asking is not. Asking PER TICKER does more of
that, not less.

**Acquired ONCE PER TICKER at watchdog first listing** (Watchlist Append
above) and persisted to `live_ticker_terms` (db_schema.md): the rate plus the
cost term, both from ONE response in one transaction. The cost term is
DERIVED rather than modelled — the same response carries `AstkOrdAbleAmt1`
beside `Mgnrt0`, so solving

```
AstkOrdAbleAmt * (100 / Mgnrt0) - @cost = AstkOrdAbleAmt1
```

against it yields `@cost` at zero extra cost, on a call already being made.
Every margin-related acquisition in this design shares that one call site.

**Never fetched on the entry path.** Per-entry acquisition was rejected on
BURST, not on steady-state budget: `able-orderqty` is 2 TPS and does not
double, being a `trading/` endpoint (trading_api.md), and
`execution.max_tickers` bounds the steady state — but simultaneous first
listings serialise at 2 TPS and would land inside the bar-close decision
deadline. First listing sits outside that deadline.

The rate in force at entry, or at submission for a pending row, is PINNED
ONTO THE `live_positions` ROW (db_schema.md). That row and `live_ticker_terms`
are NOT redundant and neither may be dropped for the other: the position row
is WHAT THAT ENTRY ACTUALLY USED, fixed against any later observation, while
`live_ticker_terms` is WHAT WAS OBSERVED FOR THAT TICKER THAT SESSION and is
the comparison baseline for the order-time check. They coincide whenever
`Mgnrt0` is 100 — which is exactly when the duplication looks removable and
is not.

**ACQUISITION FAILURE BLOCKS THE TICKER. There is no fallback rate.** A
substitute of 100 is correct when the real rate is 100 and silently
substitutes a DIFFERENT STRATEGY when it is not — a real 30 sizes 3.3x
smaller — and nothing downstream distinguishes the two cases. The
one-sided-error argument used elsewhere in this spec set ("under-sizing costs
opportunity, not solvency") does not carry: the exposure here is MEASUREMENT,
not solvency. A trade sized on a substituted rate still reaches `trade_log`,
where Pilot's `fit_execution_params()` pools it with the rest.

This PARTIALLY RETIRES R-8's "Never aborts: this refines sizing, it does not
authorise trading." That was decided when the rate was an ACCOUNT SCALAR,
where failure was global and the only alternative to a fallback was standing
down the session. A per-ticker rate creates a per-ticker option that did not
exist then, and skipping one ticker is not the same cost as standing down a
session.

Failure is the ABSENCE OF A ROW. There is no provenance column and no
nullable rate, so the state space is two states: terms held, or not.

**Retry is `trading_api.md`'s existing policy and nothing more.** The
re-attempt budget stays at its DEFAULT 0, so the SDK's own per-call
classification and backoff is the whole retry story and no new policy is
introduced. Recorded explicitly because that budget's stated rule reads the
other way at first glance: first listing sits inside a cyclic loop, and
cyclic callers pass 0 on the ground that their next cycle IS their retry. The
axis that rule draws is whether the next cycle's call fetches something NEW
or re-fetches THE SAME THING. Bars and ticks are the former; `Mgnrt0` is the
latter, so this is a one-shot caller embedded in a cycle, and treating the
next cycle as its retry would mean unbounded re-attempts — the burst the
acquisition point was moved to avoid.

Past that, the ticker is GIVEN UP for the session, held in a runner-side
IN-MEMORY SET. Not a row in `live_ticker_terms`: a failed attempt produced no
observation, and that table holds what the vendor said, so a failure row
would need a provenance column and a nullable rate back. Without the set,
"not yet listed" and "failed, given up" are both "no row" and the loop
re-calls every cycle. The set does not survive a warm restart, which gives
each failed ticker exactly one more attempt after one — bounded, and
desirable, since a transient vendor fault may have cleared by then.

Blocked tickers are counted AND NAMED in `session_diagnostics` (write point
(c) above) and surface as a health_report finding of the same class as
finding 8 — an infrastructure-health signal, distinct in kind from strategy
underperformance. That finding is what makes the no-fallback choice
survivable: silent trading stoppage is its principal risk, and this is the
instrument that shows it. It is also what measures the assumption the choice
rests on, that vendor-response failure is rare; a high observed rate is the
ground for revisiting it in favour of a fallback.

---

## on_bar_close() Sequence

```
CachingCalculator.on_bar_close(new_bar, ticks_for_bar):

  For each indicator with precalculate_bars > 0 (Layer 2):
    - CONTINUOUS (ma, ema, rsi, etc.): incremental append and recalculate
    - fibonacci: monotonic deque O(1) update (swing_high, swing_low)
    - vwap: accumulate today's cumsum
    - lee_ready, tpm, avg_vol_per_tick: accumulate today's ticks

  sr_levels and gap_percentile: NOT updated here.
    - sr_levels: recomputed per-entry-point via get_for_entry()
    - gap_percentile: fixed after on_regular_session_open()
```

---

## on_regular_session_open() Sequence

Called once when the 09:30:00 bar is confirmed closed (09:31 bar arrives).

```
CachingCalculator.on_regular_session_open(bars_including_930):
  gap_pct = calculator.gap_percentile(bars, date, n_sessions, session_stats)
  self._fixed["gap_pct"] = gap_pct
```

---

## get_for_entry() — per-entry-point dispatch

Called by FeatureExtractor.extract() in live mode (via CachingIndicatorCalculator).

```
CachingCalculator.get_for_entry(indicator, bars_up_to_t1):
  if indicator == "sr_levels":
      # per-entry-point recomputation (scipy prominence — not cacheable)
      return self.sr_levels(bars_up_to_t1.tail(window_bars), n_levels, window_bars)
  if indicator == "gap_pct":
      return self._fixed.get("gap_pct", float("nan"))   # scalar
  return self._cache[indicator]   # sliced to t-1 by FeatureExtractor
```

---

## Exit Architecture (WS-primary / REST-backstop) — R-2

tp/sl detection for OPEN POSITIONS ONLY (entry-side watchdog over the full
universe, below, is unchanged — a different problem at a different scale).
Feeds the exit-decision consumed by Position Manager Loop Step 2.

**Primary — WebSocket tick stream.** Subscribed at entry order submission
(gate-passed, order accepted — see Watchdog Polling Loop step 5c), not on
fill: a position carries the risk this stream exists to watch from the
moment an order is live, not just from its first fill, and this also
folds the ticker-cap accounting together — the subscription count and
`execution.max_tickers`'s count of `status IN ('pending', ...)` rows now
move in lockstep by construction, so subscriptions can never exceed
`execution.ws_ticker_limit` on their own. Subscriptions are per TICKER,
not per position, so several positions on one ticker share one
subscription. Price tracking owns the quote connection for the whole
session; see "WS connections" under Position Manager Loop for the
two-connection topology and why nothing is leased or shared.
Immediately after the subscribe call, REST gap-fill the short window
between the subscribe request and the subscription actually going active —
pure network/handshake latency now, not the fill-to-subscribe gap this
covered before subscription moved earlier — deduped by the global
tick-dedup rule (see utils.md's `stitch_ticks()`). Each inbound tick:
```
breach_up = price >= fill_price * (1 + tp_pct)
breach_dn = price <= fill_price * (1 - stop_loss_pct)
```
Ticks are received and deduped from the moment of subscription, but the
two lines above are only evaluated once `fill_price` is known (the
position's first fill has landed) — a tick arriving while
`status='pending'` has no reference price to compare against yet and is
simply not evaluated, not lost (the dedup layer has already recorded it,
so nothing arriving before the first fill needs a separate backfill once
`fill_price` becomes available).

Piggybacked on this same inbound-tick handler, no new subscription or
call: level-1 price, side-total resting size, and session-cumulative
filled volume are extracted from every tick into `bid_ask_snapshots`
(`source='ws_tick_piggyback'`) — see db_schema.md.

**2-print guard**: a breach fires only when the SAME direction breaches on
two consecutive raw ticks (no halt-resumption special case). Rationale: in
this universe, a single bad print (odd-lot / out-of-sequence correction)
tripping a stop-loss is a worse risk than a one-tick delay on a real move.
On a confirmed breach: set `status='exiting'` atomically (guards against
double submission), setting `live_positions.exiting_since = now` if it is
not already set (db_schema.md — never overwritten once populated), and
submit the sell (shadow: record hypothetical; order-type/pricing logic —
market vs. limit at the ladder's spread position — lives in Position
Manager Loop Step 2/3, shared
with the time_limit/session_end exit paths rather than duplicated here).
Unsubscribe when either: the position closes (as before), or the entry
order ends with zero shares filled (`entry_rejected` / `entry_canceled` —
nothing was ever exposed to track). A fill of any size
(`partial_open` or beyond) keeps the subscription regardless of what
becomes of the unfilled remainder.

**Backstop — periodic-loop REST tick polling (WS-dead only).** When the WS
stream for a position is down, the Position Manager Loop polls that
ticker's individual TICKS via REST at the SAME granularity WS would
deliver (not 10-tick OHLC bundles), and applies the SAME 2-print guard.
Because both paths consume identical tick data with an identical filter,
there is NO accuracy/filter asymmetry between them — only latency differs
(push-immediate vs. up-to-one-poll delay). A poll may return several
buffered ticks at once; they are evaluated in sequence, so detection
accuracy matches WS — only detection TIME lags. `utils.track_price_breach()`
is NOT used on this path; it is backtest-only as of this patch.

**Time-based triggers are not on the WS path.** `time_limit` and
`session_end` are wall-clock conditions — `time_limit` on elapsed minus
halted minutes, `session_end` on the clock alone — evaluated by the
periodic loop (Position Manager Loop Step 2) regardless of tick arrival —
a tick may never arrive for a low-liquidity or halted name near close, and
a time trigger must still fire. WS is fixed to price-breach only.

**R-3: moving `session_end` to WS was considered and rejected.** Each WS
tick does carry an `hour`, so a `now >= exit_deadline(today)` check is
technically expressible on the WS path — but that check would only ever
fire when a tick arrives, which is exactly the case that fails for a
low-liquidity or halted name near close: `session_end` must fire on schedule
whether or not a tick ever does. This is structural, not a tuning gap —
tying a schedule-based trigger to tick arrival trades away the one property
(fires regardless of tick activity) that makes it useful. On a normal,
liquid name where WS ticks arrive continuously, WS's price-breach path and
the periodic loop's `exit_deadline(today)` check can both become true in
the same iteration; see the ordering rule in Position Manager Loop Step 2
(tp/sl wins, `status='exiting'` guarantees a single submission).

**Concurrency.** WS readers and the periodic loop run on a single asyncio
event loop so position status transitions serialize naturally;
`status='exiting'` is the single-submission guard where WS and the
periodic loop could otherwise race (e.g. a simultaneous price breach and
`session_close` — see R-3 for the tp/sl-wins ordering rule).

**Config-driven, not hardcoded.** `execution.max_tickers` (the
subscription bound above), `execution.max_positions_per_ticker`, and
`position_check_interval_seconds` (the REST backstop's poll frequency) are
read from config at every use in this section and elsewhere — this spec
deliberately does not commit specific values beyond the defaults already in
Config Keys and `execution:`. Any future change to R-4's circuit-breaker
thresholds, R-5's entry-gate cap checks (Watchdog Polling Loop step 5c.0),
or `compute_position_size()`'s margin/exposure sums must read the same
config keys rather than hardcode a value independently. This is a standing
rule for future edits, not a pointer to unfinished work: R-4's thresholds,
freeze scope, and no-auto-clear behaviour are fully specified in Circuit
Breaker above, with their values in `execution:`.

Inf mode (`0`) on either cap means "unlimited" for BacktestEngine only.
LiveModeRunner clamps: `max_tickers` 0 → 50 (the vendor limit above, which
is not a configurable choice), `max_positions_per_ticker` 0 →
`floor(execution.max_hold_bars / execution.entry_cooldown_minutes)` — the
largest same-ticker stack the cooldown can admit inside one position's
maximum hold. Both clamps are derived, not configurable, in the same spirit
as `get_execution_param()`'s hard bounds.

---

## Position Manager Loop

Monitors open positions independently of the watchdog loop. A single
shared loop serves all open positions on one global timing grid (not a
per-position independent timer) — a shared clock beats per-position phase
alignment on every axis measured (compute batching, backtest reproducibility,
debuggability), with no compensating benefit found for per-position timing.

At position open (buy fill, real or shadow), the caller initializes:
```
# No bars_since_entry. Its only consumer was Step 2's time_limit, which is
# now elapsed-minus-halted minutes (below); session_end is pure wall clock
# and never read it, so the comment naming both was already wrong. Elapsed
# comes from live_positions.fill_second and halted minutes from this
# ticker's live_halt_episodes intervals — both durable, so this loop
# accumulates nothing and a warm restart restores nothing.
# No entry_ticks (R-2): tp/sl detection is WS-primary / REST-backstop over
# the real tick stream (see Exit Architecture below), not
# track_price_breach() — the former Phase-1 t-bar-ticks input is gone.
# track_price_breach() itself is unchanged and remains backtest-only.
```

**Ghost orders — the mirror of the vanished-order rule.** That rule handles a
TRACKED order that disappeared; this one handles an order present in the
account-wide OUTSTANDING response that this system does not track. The
detection material is already in hand every cycle, because the fill inquiry is
account-wide rather than per-order, so a ghost necessarily appears in it.

First observation records a first-seen instant per `order_id`, set ONCE and
never reset by re-observation — the same posture `live_positions.exiting_since`
already takes, so a resubmission cannot restart the clock. Held in memory, NOT
persisted: the restart case already has an owner in Broker Reconcile, which
queries broker outstanding orders at session start.

Two thresholds under `live_mode:`, with unrelated justifications, which is why
one strike count cannot serve:

- `ghost_order_alert_cycles` (3) — suppresses the SUBMISSION RACE, the
  unavoidable window between the vendor accepting an order and `order_id`
  returning in the response. A few cycles is all it needs. Alert only.
- `ghost_order_cancel_seconds` (60) — a settling interval before automatic
  cancellation, sized to let a transient resolve itself and NOT to summon an
  operator. Requiring human approval per occurrence costs more than cancelling
  does, and during an incident it would compound alert fatigue with continuing
  exposure.

**Ordering is structural, not arithmetic: an order is never cancelled before
it has been alerted.** The two thresholds carry different units, so a raised
`position_check_interval_seconds` can push the lower past the upper — at 25s
the lower reaches 75s against a 60s upper — which makes the alert a
PRECONDITION of cancellation rather than a value that happens to be smaller.

Auto-cancellation is the normal path, not a fallback for the unattended case,
and is recorded under its own `alert_key` since the system is cancelling an
order a human may have placed deliberately. The alert must name the rule and
the cancelled `order_id` explicitly enough that an operator recognises the
cause on first reading rather than re-placing the order — there is no
suppression switch and no self-limiting counter, both of which were considered
and rejected (a forgotten switch returns exposure to unbounded; a per-ticker
counter adds another unmeasured threshold).

**Manual intervention is to be routed through TradingAPI** so a manually
placed order is tracked and never reads as a ghost. This is an OPERATOR
CONVENTION and is unenforceable — the system cannot prevent an operator using
the broker's own app, and the convention is likeliest to break during exactly
the incident `health_report.md`'s finding 11 asks for manual intervention in.
At a 60-second threshold there is effectively no human-in-the-loop grace,
which is what makes the convention load-bearing. A CLI route for it is an open
item.

**In-flight order tracking** (real orders only — shadow mode has no real
order): covers BOTH order types and BOTH sides. Fills are not returned by
the order-submission API (R-7); they arrive on a separate channel, so every
accepted order is tracked here until it reaches a terminal state:
```
in_flight_orders: dict[order_id, dict]
# {ticker, side: 'entry'|'exit', submitted_at, limit_price,
#  requested_quantity, cum_filled_qty, weighted_avg_price}
# R-2: runtime CACHE only — the SSoT is the live_positions row (status
# 'pending'/'partial_open' for entries, 'exiting' for exits). Populated at
# submission, and rebuilt from live_positions WHERE status IN
# ('pending','partial_open','exiting') on a warm restart.
```

**WS connections.** TWO connections, both opened at session start and
held for the whole session: one carrying the quote stream (Exit
Architecture's price tracking, up to `execution.ws_ticker_limit` tickers)
and one carrying the account fill stream.

They cannot be combined. A connection carries ONE subscription type:
sending an account registration on a connection already carrying quote
subscriptions converts that connection and silently stops quote delivery
(measured against both production and demo accounts;
api_contract_checklist.md T-6). The realtime orderbook stream is the same
type as the quote stream and could share its connection, but is not
subscribed — the per-connection ticker budget goes to price tracking.

There is no lease, no refcount, no linger and no preemption. Two
connections are the account limit and exactly two are needed, so nothing
contends for one and there is no eviction rule to state.

**The subscription set is DERIVED, never held as independent state** — it
is `live_positions WHERE status IN ('pending','partial_open','exiting')`,
the same query that rebuilds `in_flight_orders`. One consequence covers
three cases with one mechanism: warm restart, re-establishment after the
escalation below, and the SDK clearing its own subscription list when its
receive loop exits.

**Reconnect.** The vendor caps connection attempts per minute but counts
in a one-second window, so an opening burst is legal provided attempts are
spaced above one second; the schedule is a burst followed by a fixed 10s
interval, and it is OURS rather than the SDK's (docs/api/sdk_dependency.md
explains why the SDK's own pacer is replaced). After
`live_mode.ws_reconnect_max_attempts` consecutive failures the ladder
escalates; there is NO permanent give-up within a session, the only bound
being the process itself at the R-9 hard cap — the same
principle already applied to an unfilled exit order below.

**Escalation ladder**, on repeated reconnect failure:
1. Retry to `ws_reconnect_max_attempts`. Absorbs transient loss and an
   orphaned session timing out on its own.
2. Clear the account's WS sessions and re-establish BOTH connections.
   This is SESSION-scoped, not connection-scoped: the vendor's
   session-reset call clears every WS session on the token's account, so
   escalating a failed quote connection also drops a healthy fill
   connection. Accepted on stream asymmetry — the collateral lands on the
   fill stream, whose REST substitute is lossless (order state is
   queryable, so latency costs nothing there), not on the tick stream,
   whose REST backstop preserves correctness but not timing.
   Rejected alternative: dropping the fill connection alone to free a
   slot. It does not converge — an orphaned session still holds the
   second slot, so the fill connection is locked out indefinitely.
3. Still failing after that is not a session-state problem; hand it to
   the existing Feed Outage machinery rather than inventing a new one.

A server-side subscription REJECTION is not observable through the stock
SDK, which is why the vendored copy exposes it — see
docs/api/sdk_dependency.md. Without it a position can open with no price
stream watching it while the subscription count still looks correct.

**Exits are REST-only.** Exit fills are polled on this loop's existing
cadence rather than carried on the account connection. Order state is queryable,
not a perishable event, so a missed push costs latency, not correctness —
and unlike an entry (whose fill starts tp/sl monitoring), knowing an exit
filled sooner changes no decision: the position is already closing and any
unfilled remainder stays tracked either way.

**Fill accounting invariant.** For a given order_id, `cum_filled_qty` must
always equal the broker's own count of that order's filled quantity — never
a value arrived at by ADDING reports from different channels together. WS
(primary) and REST (backstop) both report on the same underlying fills, so
naive accumulation double-counts on the routine case of a WS event and a
later REST poll describing the same fill, on WS reconnect if a missed event
is replayed, and on Warm Restart when `in_flight_orders` is rebuilt from
`live_positions` (whose `quantity` is already-filled shares) and then sees
further reports that repeat pre-crash fills. Entry-side double-counting
reaches `requested_quantity` early and confirms `open` on a wrong quantity;
exit-side is worse — it reaches `position.quantity` early and marks
`closed` while shares remain, and the remainder then has NO loop tracking
it until the next session's Broker Reconcile (finding 12) surfaces it as an
unknown broker position, exposed overnight in between. Failure in the
opposite direction (under-counting) only delays `closed`/`open`, which
stays tracked and is visible via the finding above — the two failure modes
are not symmetric, which is why the mechanism below is structured to make
over-counting impossible rather than merely unlikely.

**Fill-stream staleness detection.** No new freeze reason and no new
polling schedule — this reuses the REST backstop call this loop already
makes every `position_check_interval_seconds` cycle — account-wide, so
one call covers every order in `in_flight_orders` at once (see the
exit-side loop below). On each such call, before folding the result into
`seen_fills` as described above: if the REST response's own
`cum_filled_qty` is AHEAD of what `seen_fills`'s WS-derived state currently
reflects for that order_id, sustained across more than one consecutive
cycle, that is evidence the WS account fill stream has gone quiet while
REST keeps working — surfaced as `health_report.md` finding 24, warn
severity. Recorded via `record_health_event(finding_name=
'fill_stream_staleness', detail={order_id, entered_at, sustained_cycles})`
with `write_fn=db_write` (R-9), ONCE PER STALE EPISODE: written on entry
into the stale state and armed again once the state clears, so a second
episode on the same order IS recorded. This deliberately differs from
finding 18's once-per-order rule — an order with several fills can go
stale, recover, and go stale again, and that recurrence is precisely the
signal. Deliberately NOT a `freeze_reasons` trigger and NOT related to
Bar-Close Authority above: those judge the separate bar/price-data
channel, while this is scoped entirely to the account-wide fill-event
stream, and correctness here is already unaffected regardless — the same
fold-into-`seen_fills` mechanism absorbs whichever channel reports a given
fill first, so a stale WS stream costs detection latency, not correctness,
exactly as the REST backstop was already designed to tolerate.

Primary mechanism, used when individual fills carry a stable, unique ID
(api_contract_checklist.md T-7/T-8 confirm this before Pilot): maintain
`seen_fills[order_id]`, a map from fill ID to (qty, price), idempotently
updated by every report regardless of channel or whether it is a delta
(one fill) or a bundle (several fills, as a REST query may return) —
inserting under an already-seen ID is a no-op. `cum_filled_qty` and
`weighted_avg_price` are RECOMPUTED from that map's current contents after
each update, never incremented. This makes the result invariant to
duplicate reports, out-of-order arrival, and which channel reported first,
and it treats a WS delta and a REST bundle identically — both are just
fills to fold into the same map. On Warm Restart, `live_positions.quantity`
seeds a lower bound while `seen_fills` rebuilds from a fresh REST query, if
that query is confirmed to return an order's complete fill history rather
than a paginated or windowed slice.

Fallback mechanism, for a vendor without stable per-fill IDs: treat each
report's OWN cumulative fields, when present, as the current state rather
than a delta to add — `cum_filled_qty = max(cum_filled_qty,
reported_cum_qty)`, with `weighted_avg_price` taken from that SAME report
(never averaged separately from a different report's quantity, which would
pair a quantity and a price from different moments).

Either mechanism is guarded by the same check: a computed `cum_filled_qty`
that is LOWER than the value already held is never accepted — logged as a
health_report finding instead. This does not fire under either mechanism's
normal operation; it exists for the case an ID turns out not to be as
stable as assumed (e.g. reissued after reconnect) or a bundle arrives
truncated.

```
loop every position_check_interval_seconds (config, default: 5s):

  # The exit-side backstop is ACCOUNT-WIDE, not per-order: the vendor's
  # fill inquiry takes a date range, an optional ticker and a filled/
  # outstanding filter, with no order-number input. TWO scoped calls per
  # cycle, both feeding the pass below (see docs/api/trading_api.md):
  #   (a) FILLED, itemised, newest-first, first page only — feeds
  #       seen_fills. An unscoped itemised query re-reads the whole day
  #       every cycle and grows into continuation paging by afternoon.
  #   (b) OUTSTANDING — the orders still live at the broker.
  # (b) cannot replace (a): a fully filled order also disappears from
  # (b), so absence there alone does not separate cancellation from
  # completion.
  # Record (a)'s RETURNED ROW COUNT each cycle into live_scan_daily's
  # fill_page_rows_p50/_p95. The page size is a server-side parameter with
  # no request field and it varies per call, so this is the only way the
  # distribution is ever seen — and it is what closed the fill-inquiry
  # page-size open item, which was closed as instrumented rather than as
  # answered. Free: the rows are already in hand.

  For each order_id in in_flight_orders where side == 'exit':
    # WS account fill events are primary; the two calls above are the
    # backstop.
    # Fold every fill into seen_fills[order_id] (or the fallback path) as
    # described above; cum_filled_qty and weighted_avg_price are read from
    # the result, never accumulated directly from the raw event.
    if cum_filled_qty >= position.quantity:
        db_write: transition the live_positions row to status='closed';
            log the exit to trade_log with weighted_avg_price and
            exit_date/exit_bar of the final fill
        remove from in_flight_orders
    else:
        stay 'exiting' — NO give-up timeout WITHIN the session. An entry may
        abandon its unfilled remainder and settle for a smaller position, but
        an unsold remainder is still exposed to the very risk that triggered
        the exit, so the order stays tracked for as long as the session runs.
        # The one bound is the process itself (R-9): at
        # the R-9 hard cap the order is canceled and the
        # position is handed to the next session's Broker Reconcile under
        # the Unified Overnight Policy — see Session Shutdown. That is a
        # process-level cap, not a per-order give-up: nothing here abandons
        # an unfilled remainder while the session is still running.
        # Observability, not policy: an exit order open beyond a
        # configured age is surfaced as a health_report finding (18).
        # Forced liquidation / re-submission was deliberately deferred
        # past this observability-only baseline in an earlier session —
        # see below for what ended up designed on top of it.

        stuck_age = now - live_positions.exiting_since   # NOT submitted_at
                                                          # — see db_schema.md;
                                                          # a resubmission below
                                                          # must not reset this
        if stuck_age >= config["live_mode"]["exit_order_stuck_minutes"]:
            if this order_id has not already been recorded as stuck:
                record_health_event(finding_name='exit_order_stuck',
                    detail={order_id, age_seconds, cum_filled_qty,
                    quantity}) via write_fn=db_write
                # R-9 — ONCE per order_id, on the FIRST crossing. This
                # loop re-satisfies the condition every
                # position_check_interval_seconds, so an unconditional
                # write would emit hundreds of rows for one stuck order.
                # health_report.md finding 18 pairs this event count with
                # its existing point-in-time snapshot of orders still
                # outstanding at report time — the event answers "how
                # often", the snapshot "what is open right now".
            if market orders are permitted in the CURRENT session phase
               (execution_common.md — regular session only) and this
               order_id's own type != "market":
                cancel this order_id; submit a new market order for the
                    remaining (position.quantity - cum_filled_qty) shares;
                    update in_flight_orders to the new order_id
                # Final backstop inside regular hours, regardless of
                # exit_order_type (execution_common.md) — an escalation,
                # not a reset: live_positions.exiting_since is untouched,
                # so stuck_age keeps accumulating against the original
                # clock.
            elif market orders are NOT permitted in the current phase:
                k = min(k + config["live_mode"]["exit_ladder_increment"],
                        config["live_mode"]["exit_ladder_cap"])
                # Outside regular hours a market order is refused by the
                # venue, so the escalation is a LADDER on k rather than a
                # single step. It converges to market-order equivalence as
                # k grows — a sell limit fills against resting buyers at
                # THEIR prices — so the final-backstop property survives
                # the after-hours window. The cap bounds a runaway on
                # malformed or empty book data, not fill risk.
            # else: already market inside regular hours — nothing left to
            # escalate to. Stays tracked exactly as before; finding 18
            # keeps surfacing it.

        if this order_id's own type == "limit":
            # k governs the exit limit price THROUGHOUT, not just during
            # escalation: submitted at exit_ladder_seed, re-quoted at the
            # current k every cycle, and advanced by the ladder above once
            # past exit_order_stuck_minutes. This replaces the former
            # "amend to bid every cycle" rule.
            bid, ask = REST orderbook query, this ticker's current level-1
                (one call; both sides come back from it)
            target = ask - k * (ask - bid)     # execution_common.md
            if target != this order_id's current resting limit_price:
                amend this order_id's price to target — a single order
                    amendment, not cancel-and-resubmit, so the order_id
                    (and therefore fill tracking against it) is undisturbed
            # ROUND-ROBIN, not one query per outstanding exit per cycle.
            # The orderbook endpoint's combined budget is 4 calls/second,
            # so at P outstanding limit exits each is re-quoted every P/4
            # seconds rather than every cycle. This is what removes the
            # ceiling: querying all P every cycle saturates the bucket at
            # small P, while under round-robin no value of P can exceed it.
            # The baseline to compare against is the ORIGINAL design, which
            # already folded this query into the position_check_interval_seconds
            # cadence — 5s by default. Round-robin at P=10 re-quotes each exit
            # every 2.5s, BETTER than that baseline, and matches it at P=20.
            # NOT justified by exit_order_stuck_minutes: that key paces the
            # ladder's k escalation, while a re-quote tracks the BOOK — target
            # moves as soon as bid/ask move, at k unchanged. The relevant time
            # constant is how fast the book moves, which on these tickers is
            # seconds. Staleness is also asymmetric: a resting sell limit left
            # behind a rising book sits BELOW the new ask and fills cheap, which
            # is adverse selection, while a falling book only delays the fill.
            #
            # AT BAR CLOSE, signal_time_rest PREEMPTS the round-robin.
            # Same bucket, but the entry path sits inside the 5-second
            # decision deadline and the ladder does not.
            #
            # A move-triggered variant (skip the amend inside some
            # tolerance band) does NOT relieve the bucket and is not the
            # refinement to reach for: it saves the AMEND, while the
            # orderbook QUERY is what the bucket charges, and the book has
            # to be read to know whether the market moved at all.
            # An order originally submitted as MARKET is assumed NOT
            # amendable into a limit (the vendor does not document whether
            # its amend path permits an order-type change). It is expected
            # to be cancelled by the venue at the after-hours boundary in
            # any case, which the vanished-order rule below already covers.

        # Vanished-order rule — applies every cycle, not only at a halt
        # clear. If this order_id is absent from the OUTSTANDING call AND
        # is not accounted for by the FILLED call, submit a replacement for
        # the remaining quantity at the current phase's permitted type and
        # the current k; update in_flight_orders to the new order_id, and
        # leave live_positions.exiting_since untouched — a resubmission,
        # not a new exit.
        # One mechanism, four causes: cancellation during a halt
        # (api_contract_checklist.md T-12), cancellation by the venue at a
        # session-phase boundary, a cancel-and-resubmit whose second step
        # failed, and arbitrary broker cancellation.
        # DELIBERATELY ASYMMETRIC: absence from OUTSTANDING alone never
        # triggers a replacement. A replacement is irreversible and would
        # land on a possibly-closed position, whereas a delayed one is
        # recovered on a later cycle — seen_fills is fill-ID idempotent, so
        # a fill missed by one page folds in unchanged when it appears on
        # the next.

  For each order_id in in_flight_orders where side == 'entry':
    order_status from the WS account fill stream (primary) or a REST
        order query on this cadence (backstop) — folded into
        cum_filled_qty / weighted_avg_price via the same fill-tracking
        mechanism as the exit branch above
    if rejected (including a market order's insufficient-funds-at-actual-
        price rejection, and any limit rejection):
        db_write: transition the live_positions row to status='canceled'.
            Subordinate to that same write, log trade_log row:
            exit_reason='entry_rejected', quantity=0, fill_price=p_entry,
            exit_bar=entry_bar, reject_reason = the broker's refusal text
            verbatim. Never-opened family (db_schema.md): excluded from
            fit_execution_params(), counted as a cooldown attempt.
        remove from in_flight_orders
    elif filled in full (cum_filled_qty == requested_quantity):
        db_write: transition the live_positions row to status='open'
            (fill_price = weighted_avg_price, fill_second, quantity =
            cum_filled_qty)
        remove from in_flight_orders → proceeds into the open-position
            handling below
    elif filled in part (0 < cum_filled_qty < requested_quantity):
        db_write: transition the live_positions row to
            status='partial_open' (quantity = cum_filled_qty so far;
            requested_quantity unchanged). The filled shares are a real
            position from this moment: they enter exit management and
            count toward step 5c.0's caps, while the order itself stays
            in_flight awaiting further fills.
        # No 'full_open' status: the two ways partial_open ends — reaching
        # requested_quantity, or abandoning the remainder at
        # cancel_after_seconds — both mean "no longer awaiting fills",
        # which is exactly 'open'. This matches backtest, where a
        # partially-filled entry simply proceeds sized down.
    elif now - submitted_at >= config["execution"]["cancel_after_seconds"]:
        submit cancel request to trading API for order_id
        # race: cancel request may lose to a fill that happened moments
        # before it's processed — trading API's response to the cancel
        # attempt itself resolves this (a "too late, already filled"
        # response is treated as a fill, not a cancellation)
        if canceled (not a race-lost fill):
            if cum_filled_qty == 0:
                db_write: transition the live_positions row to
                    status='canceled' — THE SINGLE canceled-transition point
                    (R-2). Subordinate to that same write, log trade_log row:
                    exit_reason='entry_canceled', quantity=0, fill_price=p_entry,
                    exit_bar=entry_bar (see db_schema.md). Because the
                    transition is the single point (idempotent — a second
                    attempt to cancel an already-'canceled' row is a no-op),
                    a post-crash Broker Reconcile that also cancels this same
                    order cannot double-log entry_canceled.
            else:
                db_write: transition 'partial_open' → 'open' with
                    quantity = cum_filled_qty. A partially-filled order is
                    never 'canceled': shares were actually bought, so this
                    is a smaller position, not a non-event.
        remove from in_flight_orders

  For each open position:
    1.  Halt check — position-scoped only, not applied to new-entry
        candidates. API-primary, tick-rate fallback (see P-1's halt-status
        endpoint integration — utils.query_halt_status()):
        # Was Step 1a. The former Step 1 (fetch bars entry → now) is GONE
        # with bars_since_entry, its only product. Steps 2-4 keep their
        # numbers, so nothing referring to them shifts; health_report.md's
        # findings 8 and 25 now name this step as Step 1. The label keeps
        # its old width so the block below is not re-indented.
        # This step issues NO REST call — query_halt_status() is not a
        # dbsec call and the fallback reads the WS tick buffer — so the
        # Position Manager Loop is no longer a chart/min consumer at all.

        Once per Position Manager Loop iteration (not once per position —
        same single-shared-loop batching principle as the global polling
        design above; `execution.max_tickers × execution.max_positions_per_ticker`
        is small by design, so this bulk call is always small too):
        ```
        halt_status = utils.query_halt_status(
            tickers=[p.ticker for p in open_positions] +
                    [o["ticker"] for o in in_flight_orders.values()
                     if o["side"] == "exit"],
            # In-flight exit tickers folded into the SAME bulk call rather
            # than a second query — see "Halt-clear handling for an
            # in-flight exit order" below for why this list needed
            # extending at all. Deduplicated: a ticker already covered via
            # open_positions is not queried twice.
            # No URL and no chunk size: halt status is not a dbsec call and
            # its source is undecided (utils.md, open_items.md). The list is
            # small regardless (bounded by
            # execution.max_tickers × execution.max_positions_per_ticker,
            # the two axes that replaced the old single max_positions —
            # see execution_common.md's Config Keys), almost always one
            # chunk regardless of value.
        )
        ```

        For each open position:
        ```
        if halt_status is not None and position.ticker in halt_status:
            is_halted = halt_status[position.ticker]   # API authoritative
            signal_source = "api"
        else:
            # whole call failed, or this ticker missing from the response —
            # fall back to the tick-rate heuristic
            Fetch recent ticks from trading API (trailing
            halt_check_window_seconds, config default: 60s)
            tick_rate_per_min = len(ticks) * (60 / halt_check_window_seconds)
            is_halted = tick_rate_per_min < halt_heuristic_tpm (config, default: 10)
            signal_source = "tick_rate_fallback"
            # DISABLED in premarket: normal premarket liquidity in this
            # universe sits below halt_heuristic_tpm, so the heuristic
            # would report a session-wide halt. Outside the regular
            # session the fallback yields is_halted = False and the API
            # signal is the only halt authority.

        if is_halted:
            position.status = 'halted'
            alert (see health_report.md), tagged with signal_source for
            diagnosability (see health_report.md's new fallback-rate finding)
            → skip Steps 2-4 this iteration (no reliable price to act on)
            # R-3: if now >= exit_deadline(today)
            # (i.e. this position is skipping session_end specifically
            # because it cannot trade, not merely skipping an ordinary
            # mid-session iteration), log explicitly and
            # record_health_event(finding_name='overnight_halt_carry',
            # detail={ticker, position identifier}) via write_fn=db_write
            # (R-9 — see health_report.md finding 14) — the live_positions
            # row stays
            # status='halted' and is picked up by the next session's
            # Broker Reconcile under the Unified Overnight Policy (see
            # "Broker Reconcile (shared procedure)" below). The silent
            # skip becomes an owned, visible handoff instead of quietly
            # carrying a position no health check would otherwise surface.
            # Recorded once per position at the carry, not once per
            # iteration: the condition stays true on every subsequent
            # cycle. The same position is counted again by finding 15 when
            # it is liquidated next session — two moments of one carry,
            # deliberately not merged.
        else:
            position.status = 'active' (clears automatically once either
            signal recovers — no separate resumption-detection logic needed)
        ```

        No separate "resumption-auction exit" pre-registration mechanism —
        deliberately not built. A `'halted'` position is not removed from
        this loop's iteration set; it simply skips Steps 2-4 while halted.
        The instant the active signal (whichever of the two above produced
        it) recovers, `position.status` flips back to `'active'` and the
        very next iteration of this same loop resumes ordinary Step 2 exit
        evaluation automatically — the loop structure already IS the
        "pre-registered" re-check, with no extra state or broker-side
        resting order needed. (An actual resting auction order was
        considered and rejected: no confirmed trading-API support for
        halt-resumption order types, and an unprotected order risks a very
        unfavorable fill if the resumption gap moves against the position —
        same reasoning as Entry Slippage Model's limit-order option
        existing for the analogous entry-side concern.)

        **Halt-clear handling for an in-flight exit order.** A different
        problem from the above: that section governs a position that has
        NOT yet had an exit submitted (status still 'open'/'halted');
        this covers a position ALREADY `status='exiting'`, with a real
        order outstanding in `in_flight_orders`, whose ticker halts and
        then clears while that order is still unfilled. Whether the
        broker preserves, cancels, or cross-executes a resting order
        through a halt is unverified (api_contract_checklist.md T-12), so
        this is built to be correct under any of the three:

        ```
        last_halt_state: dict[ticker, bool]   # runtime only, this
            # iteration's is_halted vs. the PRIOR iteration's, per ticker
            # in the combined query above — not persisted; on a warm
            # restart the first post-restart iteration simply has no
            # "prior" to compare against, so at most one clear-edge event
            # is missed per crash, not a correctness gap (the order is
            # still tracked and re-evaluated on the next ordinary cycle
            # regardless)

        for ticker in in_flight_exit_tickers:
            was_halted = last_halt_state.get(ticker, False)
            is_halted  = halt_status.get(ticker, was_halted)  # missing
                # from the response this cycle → assume unchanged, do not
                # manufacture a spurious clear-edge from a partial response
            if was_halted and not is_halted:
                # clear edge — re-query THIS order's own status immediately,
                # not waiting for the next position_check_interval_seconds
                # cycle's ordinary fill-tracking pass
                run the exit-side loop's OUTSTANDING/FILLED pair for
                    this order immediately, rather than waiting for the
                    next position_check_interval_seconds cycle
                if the vanished-order rule finds it gone:
                    # broker canceled it during the halt — T-12's "auto-
                    # canceled" branch confirmed for this event
                    the replacement is submitted by that rule (In-flight
                        order tracking's exit-side loop), at the current
                        phase's permitted type and the current k — this
                        site adds no separate resubmission path, only the
                        immediate re-query that shortens the delay
                    record_health_event(finding_name=
                        'inflight_exit_gone_at_halt_clear',
                        detail={ticker, order_id}) via write_fn=db_write
                    # R-9 — see health_report.md finding 25. A failure of
                    # this write is swallowed and must not break the
                    # halt-clear resubmission above.
                # else: order still exists (survived the halt, or already
                # filled/partially filled through the resumption cross) —
                # no action; the ordinary fill-tracking pass on this same
                # loop's next cycle picks up whatever state it is in
            # live_halt_episodes (db_schema.md) is written from this same
            # transition: False → True INSERTs (ticker, date,
            # halt_start=now, source), True → False sets halt_end=now on
            # the open row. TICKER-scoped, so several positions on one
            # ticker share one interval instead of each storing it, and
            # source carries signal_source ('api' | 'tick_rate_fallback')
            # so the evening comparison against trading_halts can tell the
            # two apart. NEVER written into trading_halts itself.
            last_halt_state[ticker] = is_halted
        ```

    2. Exit decision for this position:
       # R-2: tp/sl breach comes from the Exit Architecture below
       # (WS-primary / REST-backstop, 2-print guard) — NOT from
       # utils.track_price_breach(), which is now backtest-only.
       if a confirmed tp/sl breach is pending for this position:
           exit_reason = "take_profit" if breach_direction == "up" else "stop_loss"
       elif elapsed_minutes(position.fill_second, now)
              - halted_minutes(position.ticker, position.fill_second, now)
              >= config["execution"]["max_hold_bars"]:
           exit_reason = "time_limit"        # wall-clock — this loop, not WS
           # Elapsed minus halted minutes IS backtest's valid-bar count:
           # build_effective_bar_sequence() excludes halt bars and counts
           # no_trade bars, so live reading max_hold_bars as minutes and
           # backtest counting valid bars are ONE quantity, not two
           # conventions (execution_common.md). Halted minutes intersect
           # [fill_second, now] with this ticker's live_halt_episodes; an
           # open interval counts to now.
       elif now >= exit_deadline(today):
           exit_reason = "session_end"       # wall-clock — this loop, not WS
       else:
           continue to next position (no exit yet)
       # Ordering when a tp/sl breach and session_close both become true in
       # the same iteration: tp/sl wins. status='exiting' (see Exit
       # Architecture) guarantees a single submission either way, and sets
       # live_positions.exiting_since = now if not already set — the
       # time_limit/session_end paths reach 'exiting' here rather than in
       # Exit Architecture, so the same "first time only" write applies at
       # this transition too (db_schema.md).

    3. if config["live_mode"]["stage"] == "shadow":
           # REAL-PATH-PARALLEL INCREMENTAL. simulate_exit_fill() consumes
           # ticks FORWARD from its anchor and those do not exist at this
           # instant, so it cannot resolve inline; deferring it to the
           # evening batch is equally rejected, because caps, cash,
           # cooldown, the circuit breaker and slot recycling all need
           # terminal outcomes in-session. Shadow therefore walks the real
           # path's beats with a tick poll where the broker poll sits, and
           # THIS step only opens the window — Step 2 has already set
           # status='exiting' and exiting_since. Settlement happens on a
           # later cycle of this same loop.
           sell_rate = execution.sell_rate_tp   if exit_reason == "take_profit"
                  else execution.sell_rate_sl   if exit_reason == "stop_loss"
                  else execution.sell_rate_neutral
           # A clock-triggered exit has no direction, so time_limit and
           # session_end must NOT fall through to _sl — doing so was
           # contaminating the population _sl is fitted on
           # (execution_common.md).
           anchor    = position.exiting_since      # a TIME, never an index
           reference = last print strictly before anchor, from the 1-tick
                       buffer for this ticker
           Each cycle, recompute STATELESSLY from the anchor (no cursor, no
           residual counter — the window only grows, so an intermediate
           answer is under-determined rather than wrong, and a warm restart
           restores no state because the anchor is a durable column):
           weighted_avg_exit_price, filled, unfilled, _ = execution_common.simulate_exit_fill(
               ticks_exit=<1-tick buffer, anchor forward>,
               position_size=position.quantity,
               exit_anchor_second=anchor, reference_price=reference,
               sell_rate=sell_rate,
               halts_df=<live_halt_episodes rows for this ticker>,
           )
           # ohlcv_exit is GONE — it had no reader (execution_common.md).
           Settled when filled == position.quantity, or at SESSION END,
           which is live's only available meaning for "ticks exhausted":
           it cannot distinguish "not yet arrived" from "no more". An
           unsettled shadow exit holds its slot to session end, exactly as
           an unfilled real sell order does.
           On settlement, in one pass over that same buffer:
             - trade_log row written COMPLETE in a single INSERT
               (is_shadow=TRUE), reference price recorded alongside the
               simulated fill for realized-vs-simulated comparison
             - exit_trigger_agreement_daily updated for this ticker: rerun
               the 2-print guard and utils.track_price_breach() over the
               full-tape buffer to obtain M and R against this exit's
               observed L (db_schema.md)
             - raw 1-tick rows ahead of the oldest remaining unresolved
               anchor are released
       else:
           if config["execution"]["exit_order_type"] == "market":
               order_id = submit order via trading API: quantity=`position.quantity`
                   - `cum_filled_qty so far` (0 for a fresh exit), order_type="market"
           else:  # "limit"
               bid, ask = REST orderbook query, this ticker's current
                   level-1 (one call; both sides come back together)
               k = config["live_mode"]["exit_ladder_seed"]
               order_id = submit order via trading API: quantity=`position.quantity`
                   - `cum_filled_qty so far`, order_type="limit",
                   limit_price=`ask - k * (ask - bid)` (execution_common.md)
           # Re-priced every position_check_interval_seconds cycle
           # thereafter while still outstanding (limit case only) — see
           # In-flight order tracking's exit-side loop below, which also
           # advances k past live_mode.exit_order_stuck_minutes. The
           # escalation is a market order INSIDE regular hours and a k
           # ladder outside them, since the venue refuses a market order
           # outside the regular session (execution_common.md).
           reference price still logged for the same comparison purpose
           # R-2: for tp/sl this is the OBSERVED price of the confirming
           # (2nd) breach tick from the WS/REST stream — a real
           # observation, not a bundle interpolation — improving the
           # realized-vs-simulated comparison's quality. It GENERALISES to
           # the last print before the trigger instant, which is defined
           # for all four exit reasons, so the former "not applicable for
           # time_limit/session_end" case no longer arises
           # (execution_common.md).

    4. Log exit to inference_log
```

---

## Config Keys

```yaml
live_mode:
  poll_interval_seconds:          1
  position_check_interval_seconds: 5    # shared timing grid — see Position Manager Loop
  scan:
    head_slots:                   3      # N — prefix-scan calls per cycle from
                                         # the head of the watchdog list
    rotation_slots:               1      # M — cursor positions advanced per
                                         # cycle through the remainder. At the
                                         # 50-ticker cap the whole list is
                                         # walked in ~50 cycles, about one bar
    shared_lane_slots:            4      # CAP on the non-reserved pacing
                                         # slots, not an allocation to one
                                         # consumer. Sized as (chart/min
                                         # combined budget) - (head_slots +
                                         # rotation_slots)
    rotation_window_seconds:      120    # forming bar (60s) + the completed
                                         # bar behind it (60s); phase-independent
    # N and M are BOUNDED BY LATENCY, NOT TPS. Pacing is a minimum interval
    # (250ms per account) and the two accounts alternate, so the last scan
    # call completes at 250ms + RTT = 850ms worst case at the measured
    # 400-600ms round trip (that check is retired as
    # api_contract_checklist.md T-16 — these keys are its record). A sustained
    # rise past ~600ms breaks the 1s cycle and head_slots must come down.
    # The shared slots deliberately sit at t=500/t=750 on
    # BOTH legs rather than a whole leg of its own — see the slot
    # allocation under Watchdog Polling Loop for why a whole-leg split
    # would push the scan's completion to 1150-1350ms.
  # max_positions removed — replaced by execution.max_tickers /
  # execution.max_positions_per_ticker (two independent axes; a single
  # global count could not express "5 positions, all on one ticker").
  # max_hold_bars removed (R-6) — execution.max_hold_bars is the single
  # source, already read by this file's restart_gap_exit / overnight_exit
  # cutoff and by backtest's time-limit exit.
  session_start_workers:          8      # parallel workers for session_start_compute()
  indicator_cache_mode:           "memory"
  # "memory" (default): all CachingCalculator state held in RAM after session_start_compute()
  #                     Lowest latency; ~12GB sustained on 32GB+ server.
  # "db":               Indicator cache persisted to indicator_cache table after
  #                     session_start_compute(); RAM released; loaded per-ticker
  #                     on first appearance in the watchdog working set
  #                     (50~200ms overhead).
  #                     Use for memory-constrained environments.
  # watchdog_url removed: the watchdog is a module call
  # (watchdog.get_candidates()), not an HTTP endpoint, so neither a URL
  # value nor a _url name was ever true of it.
  #
  # trading_api_url and trading_api_ticker_url removed. Endpoint addressing
  # belongs to the vendored SDK, which owns both the base URL and every
  # endpoint path (docs/api/sdk_dependency.md), and docs/api/trading_api.md's
  # boundary test puts it on the far side of the API layer from callers.
  # Session Lifecycle Step 1 reaches the ticker-master endpoint through
  # TradingAPI, not through a key here; query_halt_status(), the only other
  # consumer either key ever had, is not a dbsec call at all.

  # R-1: 09:20 recheck moved in-process (see "In-Process Premarket
  # Recheck" below) — no longer a cron. wall-clock America/New_York.
  premarket_recheck_time:         "09:20"

  # R-3: periodic-loop session_end trigger (Position Manager Loop Step 2)
  # — deliberately NOT on the WS path, see Exit Architecture's
  # "Time-based triggers" note. Matches backtest's close-exit
  # convention, both engines resolving through utils.md exit_deadline().
  # session_close_exit_time removed — promoted to execution: (R-7), where
  # BacktestEngine reads it too; a value both engines must agree on cannot be
  # declared on one side only (same defect R-6 fixed for max_hold_bars).
  # It no longer exists under that NAME either: execution: now carries
  # session_close_exit_offset_minutes, and the instant is resolved per date by
  # utils.md exit_deadline(). R-7's property is unchanged — one declaration,
  # both engines reading it — only its form moved from an absolute time to an
  # offset, so that being flat by 15:50 stays expressible.
  # fill_stream_mode / fill_stream_linger_seconds removed: a connection
  # carries one subscription type, so the fill stream needs a connection of
  # its own for the whole session. Nothing is leased, so there is no idle
  # hold to configure and no shared/dedicated choice to make — only the
  # former "dedicated" topology is achievable. See Position Manager Loop's
  # "WS connections".
  ws_reconnect_max_attempts:      5          # consecutive reconnect failures
                                             # before the escalation ladder
                                             # advances — see "WS connections"
  market_buy_price_margin:        1.05       # seed value — stage 1 of the
                                             # market-BUY funds gate sizes
                                             # against ask1 * this, since the
                                             # vendor converts a market BUY to
                                             # a limit at a reference price it
                                             # does not disclose
                                             # (execution_common.md's Session
                                             # Phase). Under live_mode:, not
                                             # execution:, for the same reason
                                             # as the ladder keys below —
                                             # BacktestEngine has no bid/ask
                                             # model, so simulate_entry_fill()
                                             # cannot read this and
                                             # approximates from bar data.
                                             # Set GENEROUS deliberately, on
                                             # the one-sided-error posture
                                             # this spec set takes wherever a
                                             # bound stands in for an unknown:
                                             # too high costs an occasionally
                                             # under-sized entry, too low a
                                             # vendor rejection the entry-side
                                             # path already handles
  exit_ladder_seed:               0.5        # spread position of a limit exit
                                             # at submission: ask - k*(ask-bid)
                                             # (execution_common.md). 0.5 = the
                                             # midpoint
  exit_ladder_increment:          0.1        # k advance per
                                             # position_check_interval_seconds
                                             # once past
                                             # exit_order_stuck_minutes
  exit_ladder_cap:                1.5        # k ceiling. Bounds a runaway on
                                             # malformed or empty book data,
                                             # not fill risk — k > 1.0 rests
                                             # below the bid deliberately.
                                             # Under live_mode:, not execution:,
                                             # because BacktestEngine has no
                                             # bid/ask model to mirror it
  clock_check:
    source:                       "ntp_daemon"  # | "vendor_api" | "disabled"
    max_offset_seconds:           1.0
    on_exceed:                    "abort"       # | "warn"
  retention_probe:
    enabled:                      true
    lookback_days:                14           # ask from further back than
                                               # retention is expected to reach
    assumed_days:                 5            # fallback when the probe fails
  # margin_ratio_url removed, and margin_ratio_fallback with it. The vendor
  # publishes no account-level margin-ratio endpoint and no account-level
  # RATE at all: OtptItemNm1 is a label string reporting whether a
  # per-ticker 100% override applies, not a number. The effective
  # requirement is inquiry/able-orderqty's Mgnrt0, taken as-is and already
  # incorporating that override, so there is nothing to combine and no
  # session scalar to hold. A fallback RATE is deliberately absent too:
  # acquisition failure BLOCKS the ticker rather than substituting a value
  # — see "Per-Ticker Trading Terms".
  exit_order_stuck_minutes:       10         # health_report finding 18's age
                                             # threshold
  trade_early_close_days:         false      # Early-Close Participation Gate.
                                             # LIVE ONLY — BacktestEngine ignores
                                             # it and always replays those dates.
                                             # Mechanism per option A, initial
                                             # policy per option B: the system
                                             # KNOWS the real close everywhere,
                                             # and separately declines to trade
                                             # on it until backtest says it is
                                             # worth doing. Same evaluate-but-
                                             # don't-enforce posture as R-4's
                                             # breaker and shadow mode.
  session_hard_exit_offset_minutes: 0        # R-9: process hard cap — see
                                             # "Session Shutdown". RENAMED AND
                                             # RE-MEANT from the former
                                             # session_hard_exit_time: "20:00".
                                             # It is now an OFFSET, resolved
                                             # per date as that date's
                                             # trading_calendar.after_hours_end
                                             # minus this many minutes —
                                             # 200000 ordinarily, 170000 on an
                                             # early close, where the venue's
                                             # extended session ends at 17:00.
                                             # NOT deleted in favour of pure
                                             # calendar derivation, on R-7's
                                             # ground for the same shape: 20:00
                                             # happened to equal the extended
                                             # session's end, but the value is
                                             # a POLICY — margin before the
                                             # 21:00 evening batch must stay
                                             # expressible, and that margin is
                                             # what health_report.md's
                                             # drain_timeout_seconds is sized
                                             # against. Default 0 puts the cap
                                             # at the extended-session end,
                                             # today's behaviour, so the
                                             # position is a choice rather than
                                             # an absence.
                                             # NOT the after-market ingestion
                                             # boundary, though the two once
                                             # read as one number: that range
                                             # is fixed by decision and does
                                             # not move (data_boundary.md,
                                             # metadata_crawler.md).
                                             # auxiliary_stream.md's
                                             # tail_minutes is the same instant
                                             # offset the other way and needs
                                             # no change of its own.
                                             # Ordinary exit is earlier:
                                             # all-flat past utils.md
                                             # exit_deadline(today).
  min_watchlist_size:             30         # seed value — Feed Outage
                                             # trigger condition 2 (see "Feed
                                             # Outage Recovery"). TWO roles,
                                             # both moving with this one
                                             # value: the ratio test reads the
                                             # MRU top N of the working set,
                                             # and a working set shorter than
                                             # N does not evaluate condition 2
                                             # at all. Read that section
                                             # before tuning — the sample's
                                             # composition and its size pull
                                             # sensitivity in opposite
                                             # directions. N bounds the
                                             # denominator; it does not set
                                             # it. Only tickers the scan
                                             # actually reached inside the
                                             # minute's window are counted,
                                             # so the effective denominator
                                             # is at most N
  bar_close_grace_seconds:        5          # seed value — Bar-Close
                                             # Authority's per-ticker
                                             # miss deadline; how long past
                                             # a minute's close a bar is
                                             # tolerated before counting as
                                             # missed (UNVERIFIED vendor
                                             # latency, see
                                             # api_contract_checklist.md T-13)
  bar_latency_max_error_seconds:  3          # = 3 x poll_interval_seconds.
                                             # Per-sample error bound for
                                             # T-13's latency measurement
                                             # (see Bar-Arrival Latency
                                             # Measurement): admit a sample
                                             # only if its own overstatement
                                             # is within this. Anchored to
                                             # poll_interval_seconds, NOT to
                                             # bar_close_grace_seconds:
                                             # deriving it from the value it
                                             # helps calibrate would make an
                                             # over-large grace admit
                                             # over-loose samples that then
                                             # confirm the over-large grace.
                                             # Re-examined only when
                                             # poll_interval_seconds changes,
                                             # or when finding 26 reports
                                             # heavy sample loss or a gap
                                             # between its two error classes.
                                             # NOTE: under the prefix scan a
                                             # BASELINE level of sample loss
                                             # is structural, not a fault —
                                             # rotation-only tickers carry a
                                             # delta up to ~50s and are
                                             # rejected by design. Judge this
                                             # key on CHANGE against that
                                             # baseline, not on absolute
                                             # loss; see the population-shift
                                             # note under Bar-Arrival Latency
                                             # Measurement
  bar_integrity_freeze_ticker_count: 5       # seed value — distinct tickers
                                             # with a negative bar-latency
                                             # sample (finding 27) before the
                                             # per-ticker exclusion is
                                             # promoted to a session-wide
                                             # entry freeze. A timestamp-
                                             # convention mismatch reaches
                                             # this within a minute or two;
                                             # isolated premature bars
                                             # should not
  # api_max_tickers_per_second removed. Its only stated use was chunk
  # sizing, which moved behind the API layer, and its value estimated
  # something the vendor publishes per endpoint and the SDK paces against
  # automatically. api_contract_checklist.md's T-3 is retired with it.

  # Halt detection (Position Manager Loop only — see is_tradable() in
  # execution_common.md for the separate, unrelated new-entry rename gate).
  # The dbsec vendor API publishes NO halt-status endpoint, so the primary
  # signal cannot come from it; utils.query_halt_status()'s source is
  # undecided and tracked in open_items.md, and it returns None until one is
  # chosen. The two keys below configure the tick-rate FALLBACK, which is
  # therefore the only specified path today.
  halt_check_window_seconds:      60
  halt_heuristic_tpm:             10      # ticks/min below this → position.status='halted'

  # Rollout stage (see Position Manager / Watchdog Loop stage branches and
  # shadow_retraining.md for the comparison methodology). REPLACES the
  # former `shadow_mode` boolean rather than sitting beside it: one fact
  # behind two independently editable keys is what the shared `execution:`
  # section exists to prevent, so this is the single declaration and every
  # branch reads `stage == "shadow"`.
  stage:                          shadow  # 'shadow' | 'pilot' | 'scale'
                                          # Recorded to live_session_state
                                          # at session start; Warm Restart
                                          # compares the two and ABORTS on
                                          # a mismatch, so a stage edited
                                          # between a crash and a restart
                                          # cannot silently switch the
                                          # session's mode.
                                          # Starting at 'pilot' or 'scale'
                                          # while intraday_loss_limit_pct,
                                          # consecutive_loss_limit and
                                          # entries_per_hour_limit all sit
                                          # at their 0 default is REFUSED
                                          # at startup — R-4 required this
                                          # in prose and had no mechanism
                                          # to enforce it until one
                                          # declarative key existed.
  shadow_duration_weeks:          0       # 0 = sync to class_balancer.outer_fold.test_weeks
                                          # (currently 6); >0 = explicit override
                                          # A duration, not a mode — unaffected by the above

# session_start_cash is not a config key — queried once from the trading
# API at Session Lifecycle Step 1 and held fixed for the session (the
# `balance` argument to execution_common.compute_position_size(), mirroring
# backtest's initial_cash — see 09_backtest_engine.md's Position Sizing).

# take_profit_up5/up3, stop_loss_pct, exit_interpolation, sell_rate_tp/sl,
# position_size_cash_pct/vol_pct, per_ticker_share_cap_pct, exposure_cap_pct
# moved to the shared `execution:` config section (see execution_common.md)
# — backtest and live must not carry independently-configurable copies of
# values that are supposed to be identical between them.
```

---

## Constraints

- Ticker list is sourced from the trading API at session_start (Session Lifecycle Step 1) —
  not from the watchdog, and not from DuckDB ohlcv_1min
  (today's tradable tickers only; prior-day data for non-tradable tickers is irrelevant)
- US stock splits always take effect before market open — no intra-session split handling needed
- LiveModeRunner maintains one CachingIndicatorCalculator per ticker in the tradable universe
  (eager pool initialized at session_start via parallel session_start_compute())
- Active watchlist (calculators with pending entry signals) is empty at session_start;
  tickers are appended on first appearance in the watchdog working set — not pre-populated
- session_stats bulk load (Step 2): loaded at session_start for ALL tickers from precomputed_session_stats
  (WHERE as_of_date = today — inserted by collect_daily.py the previous evening);
  used during session_start_compute() (Step 5); released from RAM after Step 5 completes
- Health Gate 1 (Step 4) runs after Steps 2-3, before Step 5's Eager Pool cost —
  session_stats/stock_meta coverage checks use only Step 2/3's already-loaded
  data; the premarket-marker check is a batch_runs lookup independent of
  either. Health Gate 2 (Step 6) is separate and runs after Step 5 specifically
  because its input (tier_used) does not exist until Eager Pool has run —
  it cannot be folded into Health Gate 1
- session_stats Phase 2 ("db" mode only): loaded per-ticker from DB on first appearance in
  the watchdog working set via load_session_stats(); in "memory" mode, session_stats
  retained in calc._session_stats
- Inferencer is instantiated once per session with the DI FeatureExtractor
- on_bar_close() called by LiveModeRunner for each ticker per bar — not by IndicatorCalculator
- Position manager loop runs independently of watchdog polling loop
- The watchdog scan governs WHAT IS FETCHED and WHICH TICKER IS
  PRIORITISED — it never changes an entry's ELIGIBILITY. Eligibility
  remains the full A-G expression over a COMPLETED bar in every path:
  mid-bar screening relaxes A only to build a fetch superset, and
  promotion only reschedules. (Under `execution.late_entry_enabled` a
  rotation-slot finding can produce an entry, which is why this is phrased
  as eligibility rather than as "scanning never produces an entry" — what
  changes there is WHEN a qualifying bar is observed and at WHAT PRICE it
  is taken, not what qualifies.)
- The bar-close sequence is instrumented END TO END, because the 5-second
  target is unverified and each stage moves independently:
  `barclose_fetch_ms` (bar close -> superset fetched), `infer_ms` (-> all
  candidates through `infer()`), `submit_dispatch_ms` (-> last entry order
  handed to `trading_api.md`'s dispatch queue), `broker_ack_ms`
  (dispatch -> broker's own order stamp) and `order_to_fill_ms` (order stamp
  -> execution stamp). The last two are NOT locally timed: everything past
  dispatch is on the far side of the async boundary, so both come from IS2's
  own fields — `Sastklclorddttm`/`Sastkorddttm` for the crossing and
  `Sastkorddttm` -> `Sastkexecdttm` for the fill, all broker-stamped and
  therefore free of local clock skew. `entry_submit_late` counts entries
  dispatched past their deadline; those are STILL SUBMITTED, since backtest
  models late detection but not late submission and dropping them would
  diverge from it in an unmodelled direction too
- The WATCHDOG LOOP's share of `live_scan_daily` is written once per session
  at shutdown, at the same layer as `bar_latency_daily`'s final flush
  (health_report.md's Shutdown Order and Drain) — scan depth histogram,
  carryover, rotation visits/hits/misses, superset size, bar-close fetch
  latency, the bar-close stage timings and promotions. These are the only
  in-session evidence that N, M and the 5-second target hold, and
  health_report.md's findings 30-32 read them
- That list is this LOOP's share, NOT the table. `live_scan_daily` has three
  other writers and none of them is here: `fill_page_rows_*` comes from the
  Position Manager Loop's fill inquiry, `late_entry_gate_pass`/`_reject`
  from the late-entry path, and `gap_*` from `metadata_crawler.md`'s evening
  detection-gap stage hours after this process has exited — which is why
  finding 33 reports the previous day when called at session end. The
  canonical key list is the schema comment in `db_schema.md`, not this
  bullet
- `live_halt_episodes` is written by the Position Manager Loop's halt check
  as intervals open and close, and any interval still OPEN at a clean
  shutdown is closed there, at the same layer as the `live_scan_daily`
  write above. After a crash with no restart the row stays open, which
  reads correctly as unresolved — no party observed a resumption. A crash
  followed by a restart resolves it instead by replay (Warm Restart step b2)
- `exit_trigger_agreement_daily` is written PER SETTLED EXIT rather than at
  shutdown. Its L term is what the live trigger path actually consumed, an
  observation that cannot be reconstructed afterwards from `tick_10`, so it
  has to be captured while this session's own 1-tick buffer still holds the
  window — which is also why the entry-side counterpart can live in the
  evening batch and this cannot
- Trade execution (buy/sell API calls) is LiveModeRunner's responsibility —
  Inferencer only returns InferenceResult
- All inference_log writes include the active run_id
- LiveModeRunner does not modify DuckDB historical data — only reads for session init
  (exception: entry_points INSERT via Inferencer; inference_log, below)
- inference_log has SPLIT ownership by event, not one writer: Inferencer
  INSERTs the inference-stage events (no_detection, preload_fail,
  bars_trimmed, no_signal, suppressed); LiveModeRunner INSERTs
  event='signal_fired' rows, once, the instant the step-5c.0 gate sequence
  ends — never by a later UPDATE (see db_schema.md's gate_result). This
  split is not a convenience, it is forced: every gate reads
  LiveModeRunner's own runtime state (freeze_reasons, live position counts
  from the DB, the cooldown cache, the breaker counters, the queried
  balance), none of which Inferencer has visibility into at the moment it
  returns an InferenceResult — so LiveModeRunner is the only party that
  can ever know how a signal_fired row ends. Multiple writers on one table
  already has a precedent (entry_points: Preprocessor and Inferencer, both
  INSERT OR IGNORE) — what's new here is splitting ownership by EVENT VALUE
  within a single table rather than by row-existence.
- session_start_compute() parallelized across workers; thread-safety per-ticker guaranteed
  (no shared state between CachingCalculator instances)
- Eager Pool bars loading (Step 3) uses `utils.load_ohlcv_with_history()`, not a raw
  `ohlcv_1min` query — stitches predecessor-ticker bars when a rename occurred
  within the lookback window; no-op fast path for the vast majority of tickers
- Corporate-event split adjustment of loaded bars happens inside
  `session_start_compute()` (see `caching_calculator.md`), anchored to
  `today_date` — not performed here in LiveModeRunner
- `meta_bulk` (Step 3b) is (ticker, date)-keyed at the source (`stock_meta`)
  but resolved to a flat per-ticker dict for today's date only, since live mode
  has exactly one date in play per session — unlike training's `extract_batch()`,
  which must stay date-keyed across a ticker's multiple historical entry dates
  (see `04_feature_extractor.md`)
- `meta_bulk` depends on `metadata_crawler.md`'s premarket crawl having run
  before session start for maximum freshness; `utils.estimate_historical_meta()`
  fallback covers any field still missing at session start (rare, but not
  guaranteed-zero — vendor update latency is a known, unresolvable-by-scheduling
  limitation, see `metadata_crawler.md`)
