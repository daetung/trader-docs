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

## Session Lifecycle

```
LiveModeRunner.start_session(today_date):

  1. Fetch today's tradable ticker list from trading service API
         all_tickers: list[str]
         (tickers with prior-day data but no trading access today
          are automatically excluded by the API response)

  2. [Phase 1] Bulk load session_stats for ALL tickers from DuckDB
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

  3. [Eager Pool] Parallel session_start_compute() for all tickers:
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
             tick_bar_history = utils.load_tick_bar_aggregates_with_history(
                 ticker,
                 indicators=[i for i in configured_tick_indicators
                             if i.precalculate_bars == "lookback"],
                 lookback_start_date, today_date, db_conn
             ) if any_indicator_configured_lookback else None
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

         del session_stats_bulk   # Phase 1 RAM released after all tickers done

  3b. [Meta Bulk Load] Bulk load stock_meta for today, with per-field fallback:
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

  4. Inferencer init:
         calculate_required_history() → self.required_bars
         Load model artifacts (run_id from config)

  5. Start watchdog polling loop (async)
  6. Start position manager loop (async)
```

**Phase 1 memory profile:**
```
session_stats_bulk peak:  ~15,000 tickers × ~18KB ≈ ~270MB
                          Released after Step 3 completes.

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

```
loop every poll_interval_seconds:

  1. Query watchdog service → list of ticker candidates for this bar

  2. For each ticker in candidates:
     a. Fetch current bars from trading API (1min chart up to now)
     b. Fetch current ticks from trading API (10-tick up to now)
     c. Update intraday state in memory / DuckDB

  3. If new 1min bar completed for ticker:
         calculators[ticker].on_bar_close(new_bar, ticks_for_bar)
         if 093000 bar just closed:
             calculators[ticker].on_regular_session_open(bars_including_930)

  4. If ticker appears in candidates for the first time this session
     → see "First-Entry Watchlist Append" below

  5. For each ticker with completed bar and active calc:
     a. Re-verify entry point candidate (detect())
     b. If confirmed: Inferencer.infer(
            bars, ticks, meta_bulk[ticker], entry,
            session_stats=calculators[ticker]._session_stats
        )
     c. If signal (up5 / up3): submit buy order via trading API
```

---

## First-Entry Watchlist Append

Called when a ticker appears in watchdog candidates for the first time this session.

```
When ticker X first appears in candidates:

  if indicator_cache_mode == "memory":
      # calc_X already in calculators (eager pool) — no action needed
      pass

  elif indicator_cache_mode == "db":
      # Restore from DB, reload session_stats, replay today's bars
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

  # Replay today's intraday bars (session_start → current bar-1)
  # (historical_bars in eager pool covers D-1 and earlier only)
  today_bars_X = fetch_today_bars_from_api(ticker_X)
  for bar in today_bars_X[:-1]:   # all completed bars except current
      calculators[ticker_X].on_bar_close(bar, ticks_for(bar))
  if regular_session_opened:
      calculators[ticker_X].on_regular_session_open(today_bars_X)
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

Monitors open positions independently of the watchdog loop.

```
loop every position_check_interval_seconds (config, default: 5s):

  For each open position:
    1. Fetch current price from trading API
    2. Check exit conditions:
         up5 signal: pnl >= +5pp → close position
         up3 signal: pnl >= +3pp → close position
         stop loss:  pnl <= -stop_loss_pct (config) → close position
         time limit: bars elapsed >= max_hold_bars (config) → close position
    3. If exit condition met: submit sell order via trading API
    4. Log exit to inference_log
```

---

## Config Keys

```yaml
live_mode:
  poll_interval_seconds:          1
  position_check_interval_seconds: 5
  max_positions:                  5
  stop_loss_pct:                  0.03
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
- session_stats Phase 1 (bulk): loaded at session_start for ALL tickers from precomputed_session_stats
  (WHERE as_of_date = today — inserted by collect_daily.py the previous evening);
  used during session_start_compute(); released from RAM after eager pool initialization
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
