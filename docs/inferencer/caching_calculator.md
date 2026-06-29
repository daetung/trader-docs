# Module: CachingIndicatorCalculator

**File:** `src/inference/caching_calculator.py`

---

## Role

Cache-aware subclass of IndicatorCalculator for live mode.
Maintains per-session and per-bar indicator caches to avoid redundant recomputation.
Injected into FeatureExtractor by LiveModeRunner as a drop-in replacement for
the stateless IndicatorCalculator used in training.

IndicatorCalculator remains stateless (training). CachingIndicatorCalculator
adds state only via inheritance — the base class interface is preserved.

---

## Class Structure

```python
class CachingIndicatorCalculator(IndicatorCalculator):
    """
    Live mode indicator calculator with two-layer cache.

    Layer 1 (fixed — set once at session start, never updated intra-session):
        pivot_points: based on prev session H/L/C — fixed for the day
        gap_pct:      based on today's 093000 open — fixed after regular session open

    Layer 2 (precalculated at session start, incrementally updated on each bar close):
        All CONTINUOUS indicators up to D-1 bars at session start,
        then incrementally updated as today's bars arrive.
        fibonacci: monotonic deque — O(1) update per bar.

    Not cached (computed on demand):
        sr_levels: per-entry-point recomputation via get_for_entry()
                   (scipy prominence is not incrementally computable)
        vwap, lee_ready, tpm, avg_vol_per_tick: SESSION_RESET or tick-based;
                   precalculate_bars=0 — computed bar-by-bar from today's data
    """
```

---

## Cache Layers

### Layer 1 — Session Fixed

```python
self._fixed: dict[str, any]
    # Populated by session_start_compute() and on_regular_session_open()
    # Never modified after population

    "pivot_points": pd.DataFrame    # constant step-function for today's session
    "gap_pct":      float | NaN     # set by on_regular_session_open()
```

### Layer 2 — Bar-Close Updated

```python
self._cache: dict[str, pd.DataFrame]
    # Populated at session_start_compute() for D-1 bars
    # Extended by on_bar_close() as today's bars arrive

    # Keys: all indicators with precalculate_bars > 0 (except sr_levels, gap_pct)
    # Value: time series DataFrame, grows one row per on_bar_close() call
```

---

## Methods

### `session_start_compute(historical_bars)`

Called once by LiveModeRunner at session start (before market open).

```python
def session_start_compute(self, historical_bars: pd.DataFrame) -> None:
    """
    historical_bars: full lookback window bars up to D-1 (last closed session).
    Covers lookback_days trading days.

    Step 1: Layer 1 — fixed indicators
        self._fixed["pivot_points"] = self.pivot_points(historical_bars)
        # gap_pct: deferred to on_regular_session_open()

    Step 2: Layer 2 — precalculate from config
        For each indicator where config.precalculate_bars > 0:
            "lookback" → use historical_bars[-lookback_days*390:]
            "window"   → use historical_bars[-window_bars:]
            integer N  → use historical_bars[-N:]

        self._cache[indicator] = self._compute_indicator(indicator, bars_slice)

        fibonacci special case:
            Initialize monotonic deque from historical_bars.
            self._fib_max_deque, self._fib_min_deque ready for O(1) updates.
    """
```

### `on_regular_session_open(bars_with_930)`

Called once when 09:30 bar is confirmed closed (09:31 bar arrives).

```python
def on_regular_session_open(self, bars_with_930: pd.DataFrame) -> None:
    """
    Computes gap_percentile (requires 093000 open price).
    session_stats must be available (set via set_session_stats() before this call).
    """
    self._fixed["gap_pct"] = self.gap_percentile(
        bars_with_930, self._today_date, self._n_sessions, self._session_stats
    )
```

### `on_bar_close(new_bar, ticks_for_bar)`

Called by LiveModeRunner each time a new 1min bar is confirmed closed.

```python
def on_bar_close(
    self,
    new_bar: pd.Series,
    ticks_for_bar: pd.DataFrame | None = None,
) -> None:
    """
    Incrementally updates Layer 2 cache.

    For CONTINUOUS indicators (ema, rsi, atr, etc.):
        Append new_bar to internal rolling window.
        Recompute indicator for the new tail (incremental or full window).

    For fibonacci (monotonic deque):
        deque.add(new_bar["close"])
        self._cache["fibonacci"]["swing_high"] = deque.max
        self._cache["fibonacci"]["swing_low"]  = deque.min
        → fib levels recomputed from updated swing_high / swing_low

    For vwap (SESSION_RESET):
        Accumulate cumsum(typical_price * volume), cumsum(volume)
        → vwap value for this bar appended to self._cache["vwap"]

    For lee_ready, tpm, avg_vol_per_tick (tick-based):
        Process ticks_for_bar → append bar aggregation to cache

    For sr_levels:
        NOT updated here — computed on demand in get_for_entry()

    For gap_pct:
        NOT updated here — fixed after on_regular_session_open()
    """
```

### `get_for_entry(indicator, bars_up_to_t1)`

Called by FeatureExtractor.extract() for each indicator lookup in live mode.

```python
def get_for_entry(
    self,
    indicator: str,
    bars_up_to_t1: pd.DataFrame,
) -> pd.DataFrame | float:
    """
    Returns indicator value(s) for the entry point at t-1.

    sr_levels:
        Recomputed fresh using bars_up_to_t1.tail(window_bars).
        scipy prominence — not incrementally cacheable.
        Returns sr_df (single-row DataFrame with price_rN, pivot_hour_rN, etc.)

    gap_pct:
        Returns self._fixed.get("gap_pct", float("nan"))   (scalar float)

    pivot_points:
        Returns self._fixed["pivot_points"].loc[t-1_hour]  (sliced to t-1)

    All others:
        Returns self._cache[indicator][: t-1_hour]          (sliced to t-1)
        FeatureExtractor applies Vectorizer on the sliced series.
    """
```

### `set_session_stats(session_stats)`

Stores delta-smoothed session stats for use by REFERENCE_SESSION indicators.

```python
def set_session_stats(self, session_stats: dict) -> None:
    """
    Called by LiveModeRunner after load_session_stats() at session start.
    session_stats: {metric: {hour: smoothed_avg_value}}
    """
    self._session_stats = session_stats
```

### `persist_to_db(db_conn, session_date, ticker)`

Serialize and store Layer 1 and Layer 2 to the `indicator_cache` table.
Used only when `config["live_mode"]["indicator_cache_mode"] == "db"`.

```python
def persist_to_db(
    self,
    db_conn: duckdb.DuckDBPyConnection,
    session_date: str,
    ticker: str,
) -> None:
    """
    Serializes self._fixed and self._cache to BLOBs and stores in indicator_cache.
    session_stats (self._session_stats) is NOT persisted here —
    it is reloaded separately via load_session_stats() when needed.

    Call after session_start_compute() + set_session_stats() are both complete.
    After this call, the instance may be released from RAM.
    Purges prior session_date entries for this ticker before inserting.
    """
```

### `load_from_db(db_conn, session_date, ticker)`

Restore Layer 1 and Layer 2 from the `indicator_cache` table.
Used only when `config["live_mode"]["indicator_cache_mode"] == "db"`.

```python
def load_from_db(
    self,
    db_conn: duckdb.DuckDBPyConnection,
    session_date: str,
    ticker: str,
) -> None:
    """
    Restores self._fixed and self._cache from stored BLOBs.
    Does NOT restore self._session_stats —
    caller must call set_session_stats() separately after this method.

    Expected call sequence (db mode, first watchdog event for ticker):
        calc = CachingIndicatorCalculator(config)
        calc.load_from_db(db_conn, today_date, ticker)
        stats = load_session_stats(db_conn, ticker, today_date, ...)
        calc.set_session_stats(stats)
        # then replay today's intraday bars via on_bar_close()
    """
```

---

## Fibonacci Monotonic Deque

```python
# Internal state for fibonacci_retracement O(1) update
self._fib_max_deque: collections.deque   # indices for sliding window max
self._fib_min_deque: collections.deque   # indices for sliding window min
self._fib_window:    collections.deque   # actual close values in window
self._fib_idx:       int                 # global bar counter

# on_bar_close() update:
def _update_fib(self, close: float) -> None:
    # expire stale entries from front of deques
    while self._fib_max_deque and self._fib_max_deque[0] <= self._fib_idx - window_bars:
        self._fib_max_deque.popleft()
    while self._fib_min_deque and self._fib_min_deque[0] <= self._fib_idx - window_bars:
        self._fib_min_deque.popleft()
    # maintain monotonic property
    while self._fib_max_deque and self._fib_window[...] <= close:
        self._fib_max_deque.pop()
    while self._fib_min_deque and self._fib_window[...] >= close:
        self._fib_min_deque.pop()
    # insert new value
    self._fib_max_deque.append(self._fib_idx)
    self._fib_min_deque.append(self._fib_idx)
    self._fib_window.append(close)
    self._fib_idx += 1
    # current max/min
    swing_high = self._fib_window[self._fib_max_deque[0] - ...]
    swing_low  = self._fib_window[self._fib_min_deque[0] - ...]
    # recompute fib levels
    ...
```

---

## precalculate_bars Resolution

```
Config value  → Bars to use at session_start_compute()
"lookback"    → historical_bars[-lookback_days * 390 :]
"window"      → historical_bars[-indicator.window_bars :]
0             → not precalculated (bar-by-bar in on_bar_close)
```

Indicators with `precalculate_bars: 0` (vwap, lee_ready, tpm, avg_vol_per_tick, sr_levels):
- Not included in Layer 2 cache at session start
- Computed or accumulated as today's bars/ticks arrive

---

## Constraints

- `CachingIndicatorCalculator` preserves the full `IndicatorCalculator` interface
- `IndicatorCalculator` remains stateless — used unchanged in training
- CachingIndicatorCalculator is only instantiated by LiveModeRunner — never in training
- LiveModeRunner maintains one CachingIndicatorCalculator instance per ticker in the
  tradable universe (pool size = all tickers from trading API at session_start)
- `session_start_compute()` must be called before any `get_for_entry()` call
- `set_session_stats()` must be called before `on_regular_session_open()` or any REFERENCE_SESSION indicator call
- `on_bar_close()` must be called in bar-close order (chronological) — no random access
- `sr_levels` is always recomputed fresh in `get_for_entry()` — never stored in Layer 2 cache
- `gap_pct` is stored as a scalar in Layer 1 after `on_regular_session_open()` — never in Layer 2
- fibonacci deque state must be initialized from historical_bars in `session_start_compute()` before `on_bar_close()` is called
- Thread safety of `_cache` and `_fixed` dicts is the responsibility of LiveModeRunner
  (single-threaded access recommended; parallel session_start_compute() across tickers is safe
  as instances share no state)
- `persist_to_db()` / `load_from_db()`: used only when `indicator_cache_mode = "db"`
  (default: `"memory"` — these methods not called in standard operation)
- `persist_to_db()` does not persist `_session_stats`;
  `load_from_db()` does not restore `_session_stats`
- After `persist_to_db()`, instance may be released from RAM
- After `load_from_db()`, `set_session_stats()` must be called before any use
