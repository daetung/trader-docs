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
    │       Active watchlist populated lazily as watchdog events arrive.
    ├── FeatureExtractor              (uses CachingIndicatorCalculator via DI)
    ├── Inferencer                    (owns FeatureExtractor)
    ├── EntryPointDetector            (used internally by Inferencer)
    ├── [Watchdog polling loop]       (async / threaded)
    └── [Position manager loop]       (async / threaded)
```

---

## DB Connection Management (P-5)

Two connections, not one — this is what makes the ad-hoc-operator-query and
premarket-cron contention problem (see former open_items_production_readiness.md
P-5) tractable without new infrastructure:

```python
read_only_conn:  opened with read_only=True — used for every SELECT this
                 session issues (Session Lifecycle bulk loads, any other
                 read). Permits concurrent read-only connections from
                 elsewhere (ad-hoc operator queries) without contention.
write_lock = Lock()
writer_conn:     single connection, all writes funneled through it —
                 inference_log, indicator_cache (if indicator_cache_mode=
                 "db"), trade_log (shadow mode), batch_runs (this
                 session's own live_session_end marker — see Session
                 Shutdown below).

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

## Session Shutdown

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
```
The evening batch job (metadata_crawler.md) waits on this marker before
its own first stage begins — see metadata_crawler.md's Dual Schedule.

---

## Session Lifecycle

```
LiveModeRunner.start_session(today_date):

  1. Fetch today's tradable ticker list from trading service API
         all_tickers: list[str]
         (tickers with prior-day data but no trading access today
          are automatically excluded by the API response)

  1b. Query account balance from trading API:
         session_start_cash: float
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
         )
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
            # 'premarket_quarantine_recheck' (N-3's 09:20 ET pass) is
            # deliberately NOT in this list — it runs well after Health
            # Gate 1 (which fires shortly after Session Start Gating,
            # around 04:00-05:00 ET), so its completion cannot be known
            # yet at this check regardless of stage name.

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

             if config["live_mode"]["indicator_cache_mode"] == "db":
                 calc.persist_to_db(db_conn, today_date, ticker)
                 # RAM released; calc discarded after persist
             else:  # "memory" (default)
                 calculators[ticker] = calc  # retain in RAM

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
             check trivially passes (see P-2 in the former
             open_items_production_readiness.md — all 9 tick-derived
             indicators currently ship precalculate_bars: 0)
         fallback_rate > 0.20 → WARN + proceed (self-healing case, but batch
             likely failed — alert operator)

  7. Inferencer init:
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
  (50~200ms per-ticker load latency on first watchdog event).
```

---

## Watchdog Polling Loop

Polls external watchdog service at `poll_interval_seconds` (default: 1s).
The watchdog service is a separate external system, not part of this
codebase — it applies its own entry-point-condition matching against the
live feed and pushes a ticker's name into the candidate list the instant
its own criteria are met. "Candidates" therefore means "tickers the
watchdog service just flagged," not "all actively-tracked tickers" —
this is why bar/tick fetching below (Step 2) only concerns itself with
that ticker being flagged again, not with continuously polling every
ticker in the watchlist.

```
loop every poll_interval_seconds:

  1. Query watchdog service → list of ticker candidates for this bar
     (tickers the watchdog service's own entry-condition matching just
     flagged — see note above)

  2. For each ticker in candidates:
     a. Fetch current bars from trading API (1min chart up to now —
        the API always returns the full range since the caller's last
        query, not just the latest bar)
     b. Fetch current ticks from trading API (10-tick up to now)
     c. Update intraday state in memory / DuckDB

  3. If one or more new 1min bars completed for ticker since last check:
         for each newly-arrived bar, in chronological order:
             calculators[ticker].on_bar_close(bar, ticks_for_bar)
             if 093000 bar just closed:
                 calculators[ticker].on_regular_session_open(bars_including_930)
         # Looping here (rather than assuming exactly one new bar) is what
         # lets a ticker that drops out of candidacy for a while and later
         # returns catch up correctly in the same step used for first entry
         # — see "Watchlist Append" below. on_bar_close() itself is O(1)
         # per call for every CONTINUOUS indicator (see caching_calculator.md),
         # so looping over several missed bars here is cheap regardless of
         # how many accumulated.

  4. If ticker appears in candidates for the first time this session,
     or is returning after one or more missed poll cycles
     → see "Watchlist Append" below (Step 3's loop already replays any
     bars missed in between; this step only concerns first-time calculator
     setup when indicator_cache_mode="db")

  5. For each ticker with completed bar and active calc:
     a. Re-verify entry point candidate (detect())
     b. If confirmed: Inferencer.infer(
            bars, ticks, meta_bulk[ticker], entry,
            session_stats=calculators[ticker]._session_stats
        )
     c. If signal (up5 / up3):
        i.  is_tradable(ticker, db_conn) — execution_common.md gate.
            False (ticker_cik_map.status = 'suspended') → skip, no order.
        ii. quantity = execution_common.compute_position_size(
                balance=session_start_cash, fill_price=entry["p_entry"],
                t_bar_volume=..., ticker_notional=..., total_notional=...,
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
            available_cash queried fresh from trading API immediately
            before this call (see execution_common.md's Constraints)
            if not proceed: skip, no order
        iii. limit_price = None if config["execution"]["entry_order_type"] == "market" \
                 else (entry["p_entry"] * (1 + config["execution"]["entry_gap_value"])
                       if config["execution"]["entry_gap_type"] == "percentage"
                       else entry["p_entry"] + config["execution"]["entry_gap_value"])
             if config["live_mode"]["shadow_mode"]:
                 weighted_avg_fill_price, filled, unfilled, status =
                     execution_common.simulate_entry_fill(
                         ticks_entry=..., ohlcv_entry=..., quantity=quantity,
                         fill_bundle_idx=..., p_entry=entry["p_entry"],
                         buy_rate=config["execution"]["buy_rate"],
                         halts_df=...,
                         cancel_after_seconds=config["execution"]["cancel_after_seconds"],
                         limit_price=limit_price,
                     )
                 log hypothetical entry to trade_log (is_shadow=TRUE, quantity=filled)
                 # entry_order_type is simulated the same way in shadow as
                 # in backtest — see execution_common.md's price-gate logic
             else:
                 submit order via trading API: quantity=`quantity`,
                 order_type=config["execution"]["entry_order_type"],
                 limit_price=limit_price (omitted/ignored for "market")
                 if order_type == "limit":
                     pending_entries[order_id] = {ticker, submitted_at: now,
                         limit_price, quantity}
                     # tracked to fill/cancel by Position Manager Loop — see
                     # that section's "Pending limit-entry tracking"
                 # "market": real fill returned synchronously, same as before
```

---

## Feed Outage Recovery

Distinct from the routine per-ticker catch-up already built into the
Watchdog Polling Loop (Step 3's multi-bar replay) and Bar-Close Authority —
those handle an individual ticker going quiet for a while, gracefully and
without any freeze. This section covers the qualitatively different case:
LiveModeRunner's own connectivity to the trading API/watchdog service is
lost.

**Trigger** (either condition):
- An explicit connection/API-level failure (exception, timeout, non-200
  response) from the trading API or watchdog service itself, or
- More than 50% of the active watchlist simultaneously misses its
  wall-clock bar-close deadline (see Bar-Close Authority) within the same
  minute — a systemic signal even without a hard connection error.

**Recovery procedure**, on detecting the trigger:
```
1. Freeze: set session-wide freeze flag = True.
   Watchdog Polling Loop step 5c and Position Manager Loop step 3 both
   check this flag first — while True, no new entries are submitted and
   no exits are submitted (exit *evaluation* — track_price_breach() —
   still runs and still accumulates bars/ticks normally; only order
   *submission* is held).

2. Reconnect: retry the trading API/watchdog service connection
   (backoff policy TBD — not specified here).

3. Catch up: once reconnected, this does NOT require a separate vendor
   endpoint — the trading API's existing "always returns full range since
   last successful query" behavior (already relied on elsewhere — see
   Watchdog Polling Loop Step 2a, Position Manager Loop Step 1) means the
   very next ordinary bar-fetch call for each active-watchlist ticker
   naturally returns everything missed during the outage. The EXISTING
   Step 3 multi-bar replay loop (calculators[ticker].on_bar_close() called
   once per missed bar, in order) handles the catch-up with no new
   mechanism — the outage is, from the calculator's point of view, just an
   unusually long version of the same gap Step 3 already tolerates.
   (A gap long enough to exceed the trading API's own retention/buffer is
   a true, unrecoverable data gap — not engineered around here; falls
   through to whatever the vendor's own missing-data convention is.)

4. Reconcile: query the trading API for all currently-open positions
   (broker's view) and compare against LiveModeRunner's own open-position
   tracking. Discrepancies:
     - broker shows a position closed that LiveModeRunner still tracks as
       open → an exit order placed just before the outage evidently filled;
       adopt the broker's fill (price/time from the broker's own record) —
       this is authoritative, not a simulated/estimated value.
     - broker shows a position open that LiveModeRunner has no record of →
       log and alert (see health_report.md) — should not happen if entry
       order confirmation is itself reliable, but not assumed impossible.

5. Re-evaluate exits: for each position that remained open through the
   outage, run exit evaluation (track_price_breach() over the now-caught-up
   bars/ticks) once before unfreezing. Any position whose exit condition
   would have triggered *during* the gap: exit at market immediately (not
   at the historical breach point — that price is stale by now), logged
   with `exit_reason='feed_gap_exit'` (see db_schema.md's trade_log) so
   this trade is excluded from ordinary strategy PnL attribution — it
   reflects outage handling, not strategy performance.

6. Unfreeze: clear the session-wide freeze flag. Normal operation resumes.
```

## Watchlist Append

Called when a ticker appears in watchdog candidates for the first time this
session, **or** is returning after one or more missed poll cycles (i.e. it
was a candidate before, dropped out, and the watchdog service is flagging
it again). Both cases use the same procedure — a ticker's absence from
candidates for any stretch of time is not tracked or treated specially; the
calculator's bar history simply catches up to however much was missed,
same as it does for the ordinary first-time case:

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
```

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

## Position Manager Loop

Monitors open positions independently of the watchdog loop. A single
shared loop serves all open positions on one global timing grid (not a
per-position independent timer) — see open_items_production_readiness.md's
former P-10 discussion for why a shared clock beats per-position phase
alignment on every axis measured (compute batching, backtest reproducibility,
debuggability), with no compensating benefit found for per-position timing.

At position open (buy fill, real or shadow), the caller initializes:
```
position.entry_ticks       = tick_10 rows for the t bar (fixed for the
                              position's lifetime — Phase 1 input to
                              track_price_breach())
position.bars_since_entry  = []   # grown each iteration below
```

**Pending limit-entry tracking** (real orders only, `execution.entry_order_type
== "limit"`; not applicable to `"market"` or to shadow mode, which has no
real order to be pending): submitted at Watchdog Polling Loop step 5c.iii,
tracked here rather than in a separate loop — reuses the same
`position_check_interval_seconds` cadence already running:
```
pending_entries: dict[order_id, dict]  # {ticker, submitted_at, limit_price, quantity}
# populated by Watchdog Polling Loop immediately after limit order submission
```

```
loop every position_check_interval_seconds (config, default: 5s):

  For each order_id in pending_entries:
    order_status = query trading API for order_id status
    if filled (fully or partially):
        remove from pending_entries → open a position for the filled
        quantity (proceeds into the open-position handling below)
    elif now - submitted_at >= config["execution"]["cancel_after_seconds"]:
        submit cancel request to trading API for order_id
        # race: cancel request may lose to a fill that happened moments
        # before it's processed — trading API's response to the cancel
        # attempt itself resolves this (a "too late, already filled"
        # response is treated as a fill, not a cancellation)
        if canceled (not a race-lost fill):
            log trade_log row: exit_reason='entry_canceled', quantity=0,
            fill_price=p_entry, exit_bar=entry_bar (see db_schema.md)
        remove from pending_entries

  For each open position:
    1. Fetch current bars from trading API (entry → now; the API always
       returns the full range, so this naturally becomes
       position.bars_since_entry each call — no incremental bookkeeping
       needed)

    1a. Halt check — position-scoped only, not applied to new-entry
        candidates. API-primary, tick-rate fallback (see P-1's halt-status
        endpoint integration — utils.query_halt_status()):

        Once per Position Manager Loop iteration (not once per position —
        same single-shared-loop batching principle as the global polling
        design above; `max_positions` is small by design, so this bulk
        call is always small too):
        ```
        halt_status = utils.query_halt_status(
            tickers=[p.ticker for p in open_positions],
            trading_api_url=config["live_mode"]["trading_api_url"],
            chunk_size=config["live_mode"]["bulk_api_chunk_size"],
            # bulk_api_chunk_size defined once in metadata_crawler.md's
            # Config Keys (shared pipeline_config.yaml) — barely matters
            # here since open_positions is small (bounded by
            # max_positions), almost always one chunk regardless of value.
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

        if is_halted:
            position.status = 'halted'
            alert (see health_report.md), tagged with signal_source for
            diagnosability (see health_report.md's new fallback-rate finding)
            → skip Steps 2-4 this iteration (no reliable price to act on)
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

    2. tp_pct = config["execution"]["take_profit_up5"] if position.signal == "up5" \
                else config["execution"]["take_profit_up3"]
       direction, breach_price, breach_hour, is_ambiguous = utils.track_price_breach(
           ohlcv_future=position.bars_since_entry,
           ticks_future=position.entry_ticks,
           fill_price=position.fill_price,
           fill_second=position.fill_second,
           threshold_up=tp_pct,
           threshold_dn=config["execution"]["stop_loss_pct"],
           exit_interpolation=config["execution"]["exit_interpolation"],
       )
       if direction is not None:
           exit_reason = "take_profit" if direction == "up" else "stop_loss"
       elif bars elapsed >= max_hold_bars (config):
           exit_reason = "time_limit"
       elif approaching session close (15:59):
           exit_reason = "session_end"
       else:
           continue to next position (no exit yet)

    3. if config["live_mode"]["shadow_mode"]:
           sell_rate = config["execution"]["sell_rate_tp"] if exit_reason == "take_profit" \
                       else config["execution"]["sell_rate_sl"]
           weighted_avg_exit_price, filled, unfilled, _ = execution_common.simulate_exit_fill(
               ticks_exit=..., ohlcv_exit=..., position_size=position.quantity,
               breach_bundle_idx=..., breach_price=breach_price,
               sell_rate=sell_rate, halts_df=...,
           )
           log hypothetical exit to trade_log (is_shadow=TRUE), breach_price
           also recorded alongside the simulated fill for future
           realized-vs-simulated comparison
       else:
           submit sell order via trading API (real fill from broker;
           breach_price still logged for the same comparison purpose)

    4. Log exit to inference_log
```

---

## Config Keys

```yaml
live_mode:
  poll_interval_seconds:          1
  position_check_interval_seconds: 5    # shared timing grid — see Position Manager Loop
  max_positions:                  5
  max_hold_bars:                  60
  session_start_workers:          8      # parallel workers for session_start_compute()
  indicator_cache_mode:           "memory"
  # "memory" (default): all CachingCalculator state held in RAM after session_start_compute()
  #                     Lowest latency; ~12GB sustained on 32GB+ server.
  # "db":               Indicator cache persisted to indicator_cache table after
  #                     session_start_compute(); RAM released; loaded per-ticker on
  #                     first watchdog event (50~200ms overhead).
  #                     Use for memory-constrained environments.
  watchdog_url:                   "http://watchdog-service/candidates"
  trading_api_url:                "http://trading-api"
  trading_api_ticker_url:         "http://trading-api/tickers/today"

  # Halt detection (Position Manager Loop only — see is_tradable() in
  # execution_common.md for the separate, unrelated new-entry rename gate).
  # trading_api_url above is the API-primary signal (see
  # utils.query_halt_status()); the two keys below configure the
  # tick-rate FALLBACK only, used when that call fails or omits a ticker.
  halt_check_window_seconds:      60
  halt_heuristic_tpm:             10      # ticks/min below this → position.status='halted'

  # Shadow mode (see Position Manager / Watchdog Loop shadow_mode branches
  # and new health_report.md / ops doc for the comparison methodology)
  shadow_mode:                    false
  shadow_duration_weeks:          0       # 0 = sync to class_balancer.outer_fold.test_weeks
                                          # (currently 6); >0 = explicit override

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

- Ticker list is sourced from trading service API at session_start — not from DuckDB ohlcv_1min
  (today's tradable tickers only; prior-day data for non-tradable tickers is irrelevant)
- US stock splits always take effect before market open — no intra-session split handling needed
- LiveModeRunner maintains one CachingIndicatorCalculator per ticker in the tradable universe
  (eager pool initialized at session_start via parallel session_start_compute())
- Active watchlist (calculators with pending entry signals) is empty at session_start;
  tickers are appended as watchdog events arrive — not pre-populated
- session_stats bulk load (Step 2): loaded at session_start for ALL tickers from precomputed_session_stats
  (WHERE as_of_date = today — inserted by collect_daily.py the previous evening);
  used during session_start_compute() (Step 5); released from RAM after Step 5 completes
- Health Gate 1 (Step 4) runs after Steps 2-3, before Step 5's Eager Pool cost —
  session_stats/stock_meta coverage checks use only Step 2/3's already-loaded
  data; the premarket-marker check is a batch_runs lookup independent of
  either. Health Gate 2 (Step 6) is separate and runs after Step 5 specifically
  because its input (tier_used) does not exist until Eager Pool has run —
  it cannot be folded into Health Gate 1
- session_stats Phase 2 ("db" mode only): loaded per-ticker from DB on first watchdog event
  via load_session_stats(); in "memory" mode, session_stats retained in calc._session_stats
- Inferencer is instantiated once per session with the DI FeatureExtractor
- on_bar_close() called by LiveModeRunner for each ticker per bar — not by IndicatorCalculator
- Position manager loop runs independently of watchdog polling loop
- Trade execution (buy/sell API calls) is LiveModeRunner's responsibility —
  Inferencer only returns InferenceResult
- All inference_log writes include the active run_id
- LiveModeRunner does not modify DuckDB historical data — only reads for session init
  (exception: inference_log and entry_points INSERT via Inferencer)
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
