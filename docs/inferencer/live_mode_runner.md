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
    ├── CachingIndicatorCalculator    (injected into FeatureExtractor)
    ├── FeatureExtractor              (uses CachingCalculator)
    ├── Inferencer                    (owns FeatureExtractor)
    ├── EntryPointDetector            (used internally by Inferencer)
    ├── [Watchdog polling loop]       (async / threaded)
    └── [Position manager loop]       (async / threaded)
```

---

## Session Lifecycle

```
LiveModeRunner.start_session(today_date):

  1. Load historical bars from DuckDB (lookback_days from config)
     bars_by_ticker = {ticker: ohlcv_1min rows for lookback window}

  2. Load session_stats from DuckDB (precomputed_session_stats)
     session_stats_raw = SELECT * WHERE as_of_date = today AND n_sessions = config_n
     session_stats = load_session_stats(session_stats_raw, delta_minutes, session_mode)
     # applies delta smoothing in memory

  3. CachingIndicatorCalculator.session_start_compute(historical_bars)
     # Layer 1: pivot_points (fixed for session)
     # Layer 2 precalculation: all CONTINUOUS indicators up to D-1
     # per precalculate_bars config

  4. Inferencer initializes (calculate_required_history, load model)

  5. Start watchdog polling loop (async)
  6. Start position manager loop (async)
```

---

## Watchdog Polling Loop

Polls external watchdog service at `poll_interval_seconds` (default: 1s).

```
loop every poll_interval_seconds:

  1. Query watchdog service → list of ticker candidates

  2. For each ticker in candidates:
     a. Fetch current bars from trading API (1min chart up to now)
        → update intraday bars in memory / DuckDB
     b. Fetch current ticks from trading API (10-tick chart up to now)
        → update intraday ticks in memory

  3. CachingCalculator.on_bar_close(new_bar) if new 1min bar completed
     # incremental update for all Layer 2 indicators
     # on_regular_session_open() if 093000 bar just closed (gap_percentile)

  4. For each ticker with completed bar:
     a. Re-verify entry point candidate (detect())
     b. If confirmed: Inferencer.infer(bars, ticks, meta, entry, session_stats)
     c. If signal (up5 / up3): submit buy order via trading API
```

**Bar completion detection:**
A new 1min bar is considered complete when the API returns data for the bar
immediately following the last known bar (i.e., bar at hour H+1 appears → bar H is final).

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
  max_positions:                  5      # concurrent position limit
  stop_loss_pct:                  0.03   # 3% stop loss
  max_hold_bars:                  60     # max hold in 1min bars
  watchdog_url:                   "http://watchdog-service/candidates"
  trading_api_url:                "http://trading-api"
```

---

## Constraints

- LiveModeRunner instantiates CachingIndicatorCalculator and injects it into FeatureExtractor
- Inferencer is instantiated once per session with the DI FeatureExtractor
- session_stats loaded once at session start; not reloaded intra-session
- on_bar_close() is called by LiveModeRunner — not by IndicatorCalculator or FeatureExtractor
- Position manager loop runs independently of watchdog polling loop
- Trade execution (buy/sell API calls) is LiveModeRunner's responsibility — Inferencer only returns InferenceResult
- All inference_log writes include the active run_id
- LiveModeRunner does not modify DuckDB historical data — only reads for session init
