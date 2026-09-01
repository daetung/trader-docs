# Module: CachingIndicatorCalculator

**File:** `src/live/caching_calculator.py`

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

    Not cached (computed on demand, session-only by default):
        sr_levels: per-entry-point recomputation via get_for_entry()
                   (scipy prominence is not incrementally computable)
        vwap: SESSION_RESET — always bar-by-bar, no lookback option
        lee_ready, tpm, avg_vol_per_tick (and other tick-derived indicators —
                   see 02_indicator_calculator.md's Tick-Derived Indicator
                   Scale Sensitivity registry): default precalculate_bars=0,
                   computed bar-by-bar from today's ticks; precalculate_bars:
                   "lookback" is also a valid per-indicator config, in which
                   case Layer 2 IS seeded at session start from
                   tick_bar_history (see session_start_compute() Step 2b
                   below) — these are not permanently uncacheable, only
                   session-only by default
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

### `session_start_compute(historical_bars, tick_bar_history=None)`

Called once by LiveModeRunner at session start (before market open).

```python
def session_start_compute(
    self,
    historical_bars: pd.DataFrame,
    tick_bar_history: pd.DataFrame | None = None,
) -> None:
    """
    historical_bars: full lookback window bars up to D-1 (last closed session).
    Covers lookback_days trading days.
    tick_bar_history: wide-format tick-derived indicator history (one column
        per indicator), for whichever tick-derived indicators are configured
        with precalculate_bars: "lookback" (02_indicator_calculator.md). Loaded
        by the SAME per-ticker parallel Eager Pool worker that loads
        historical_bars — via utils.load_tick_bar_aggregates_with_history() —
        not staggered across a separate round; both must be available before
        this call. None if no indicator in config uses "lookback" for its
        tick-derived precalculate_bars value.

    Step 0: Corporate-event bar adjustment (before any indicator computation)
        historical_bars = utils.adjust_bars_for_corporate_events(
            historical_bars, self._ticker, reference_date=self._today_date, db_conn
        )
        # Anchored to today (self._today_date) — since live mode has exactly
        # one date in play per session, this is always a no-op unless a split
        # occurred within the loaded lookback window. No per-entry rescale is
        # needed afterward (unlike training's Strategy A, which must rescale
        # per entry date because a single ticker-batch spans many dates) —
        # here f_e is implicitly 1.0 for every value derived from these bars,
        # since "today" is the only date being computed against.
        # tick_bar_history is NOT adjusted at this step — it stays raw/native
        # per its own date, same as training; correction (where scale_type
        # requires it) happens downstream via
        # utils.adjust_tick_derived_series_for_corporate_events(), anchored
        # to entry_date, at FeatureExtractor's slicing step — not here.

    Step 1: Layer 1 — fixed indicators
        self._fixed["pivot_points"] = self.pivot_points(historical_bars)
        # gap_pct: deferred to on_regular_session_open()

    Step 2: Layer 2 — precalculate from config (bar-derived indicators)
        For each indicator where config.precalculate_bars > 0:
            "lookback" → use historical_bars[-lookback_days*390:]
            "window"   → use historical_bars[-window_bars:]
            integer N  → use historical_bars[-N:]

        self._cache[indicator] = self._compute_indicator(indicator, bars_slice)

        fibonacci special case:
            Initialize monotonic deque from historical_bars.
            self._fib_max_deque, self._fib_min_deque ready for O(1) updates.

    Step 2b: Layer 2 — precalculate from tick_bar_history (tick-derived
             indicators with precalculate_bars: "lookback" only)
        For each tick-derived indicator configured "lookback":
            self._cache[indicator] = tick_bar_history[indicator column]
        Indicators still configured 0 (the default) are unaffected — they
        remain in the session-only, on_bar_close()-accumulated path below.
        If tick_bar_history is None (no indicator uses "lookback"), this
        step is a no-op.
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
        Recompute indicator for the new tail — O(1) incremental update only
        (e.g. Wilder's smoothing for rsi/atr/adx, recursive ema, running
        sum/sum-of-squares for bollinger's rolling std, cumulative sum for
        obv). Full-window recomputation from scratch on every call is not
        permitted for any indicator in this category — see the catch-up
        rationale in live_mode_runner.md's Watchdog Polling Loop (Step 3's
        multi-bar replay loop is only cheap because every indicator here is
        O(1) per call).

    For fibonacci (monotonic deque):
        deque.add(new_bar["close"])
        self._cache["fibonacci"]["swing_high"] = deque.max
        self._cache["fibonacci"]["swing_low"]  = deque.min
        → fib levels recomputed from updated swing_high / swing_low

    For vwap (SESSION_RESET):
        Accumulate cumsum(typical_price * volume), cumsum(volume)
        → vwap value for this bar appended to self._cache["vwap"]

    For lee_ready, tpm, avg_vol_per_tick, and other tick-derived indicators
    configured precalculate_bars: 0 (default — see 02_indicator_calculator.md's
    Tick-Derived Indicator Scale Sensitivity registry for the current set,
    e.g. vol_weighted_buy_ratio, avg_delta_per_tick, tick_realized_vol,
    path_efficiency, vol_concentration, tick_burstiness):
        Process ticks_for_bar → append bar aggregation to cache

    For tick-derived indicators configured precalculate_bars: "lookback":
        Same bar-by-bar accumulation as above — Step 2b only affects the
        session_start_compute() seed value; on_bar_close()'s per-bar
        aggregation logic is identical either way.

    For sr_levels:
        NOT updated here — computed on demand in get_for_entry() from the
        bars_up_to_t1 passed in at call time.

    For gap_pct:
        NOT updated here — fixed after on_regular_session_open()

    Rule for any future indicator that cannot be reduced to an O(1)
    incremental update (e.g. another scipy-prominence-style computation):
    it MUST follow the sr_levels pattern above — excluded from this
    method's per-bar path entirely, computed fresh in get_for_entry() from
    the caller-supplied bars window. It must NOT be added to the CONTINUOUS
    category with a "full window" fallback. This keeps every call to this
    method cheap regardless of how many bars are being replayed in one
    batch (see live_mode_runner.md's Watchdog Polling Loop and Position
    Manager Loop).
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

    Expected call sequence (db mode, on a ticker's first appearance in
    the watchdog working set):
        calc = CachingIndicatorCalculator(config)
        calc.load_from_db(db_conn, today_date, ticker)
        stats = load_session_stats(db_conn, ticker, today_date, ...,
                                   closes=closes)
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
                 (390 = an ordinary session's bar count, used as a WINDOW SIZE
                  over the loaded frame; not a per-date session boundary. The
                  frame spans 040000~200000, so the slice is conservative)
"window"      → historical_bars[-indicator.window_bars :]
0             → not precalculated (bar-by-bar in on_bar_close)
```

Indicators with `precalculate_bars: 0` (vwap, sr_levels, and all tick-derived
indicators at their default — lee_ready, tpm, avg_vol_per_tick, and the 6
added since — see 02_indicator_calculator.md's Tick-Derived Indicator Scale
Sensitivity registry for the current full set):
- Not included in Layer 2 cache at session start
- Computed or accumulated as today's bars/ticks arrive

---

## Scoped Mid-Session Recompute (item N)

Triggered by LiveModeRunner (not by `on_bar_close()`) when an
already-Eager-Pooled ticker gains a NEW same-day `corporate_events` row —
from either vendor (yfinance narrow crawl or investing.com bulk; see
metadata_crawler.md), after this instance's `session_start_compute()`
already ran on stale (pre-event) data.

```python
def scoped_recompute(
    self,
    historical_bars: pd.DataFrame,
    tick_bar_history: pd.DataFrame | None,
    bars_today: pd.DataFrame,
) -> None:
    """
    One-shot correction for a single ticker whose indicators were seeded
    before its corporate event was known.

      1. Re-run the session_start_compute() body — Step 0
         (adjust_bars_for_corporate_events, now seeing the new row) + Layer
         1 / Layer 2 seeding — exactly as at session start. This is a
         single-ticker full recompute, NOT an on_bar_close() call, so the
         O(1) per-call rule for on_bar_close() (see Constraints) is
         untouched.
      2. Replay today's bars so far via on_bar_close() from session start
         to the current bar. Reuses the same cheap replay path Watchlist
         Append and Feed Outage Recovery already rely on (on_bar_close()
         is O(1) per bar, so replaying N bars is O(N) — fine for one
         ticker).

    After this returns, LiveModeRunner clears the ticker's
    quarantine_reason if it was set (the distortion is corrected; no
    reason to keep blocking entries) — see live_mode_runner.md.

    Cost note: touches ONE ticker, not the universe-wide Eager Pool. Runs
    as a background task off the hot loops.
    """
    ...
```

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
- `session_start_compute()` must apply `utils.adjust_bars_for_corporate_events()`
  (Step 0) before computing Layer 1/Layer 2 — this is unconditional, not
  optional, though it is a no-op for the vast majority of tickers/days
  `scoped_recompute()` (item N) re-applies this same Step 0 when a
  corporate event is discovered after session start — the only sanctioned
  mid-session re-application, and still a single-ticker one-shot, never
  folded into `on_bar_close()`.
- `obv`/`ad`'s training-specific per-entry-date fallback (see
  `04_feature_extractor.md` Strategy A) does not apply here: live mode
  recomputes `historical_bars` fresh each session from a short
  `lookback_days` window, uniformly anchored to today via Step 0 — there is
  no multi-date range within a single `session_start_compute()` call for a
  split to fall between, unlike training's ticker-batch spanning many
  historical entry dates
- Thread safety of `_cache` and `_fixed` dicts is the responsibility of LiveModeRunner
  (single-threaded access recommended; parallel session_start_compute() across tickers is safe
  as instances share no state)
- `persist_to_db()` / `load_from_db()`: used only when `indicator_cache_mode = "db"`
  (default: `"memory"` — these methods not called in standard operation)
- `persist_to_db()` does not persist `_session_stats`;
  `load_from_db()` does not restore `_session_stats`
- After `persist_to_db()`, instance may be released from RAM
- After `load_from_db()`, `set_session_stats()` must be called before any use
- `session_start_compute()`'s `tick_bar_history` parameter and Step 2b affect
  only WHICH bars seed `self._cache` for tick-derived indicators configured
  `precalculate_bars: "lookback"` — `persist_to_db()`/`load_from_db()`
  serialize/restore `self._cache` generically regardless of what seeded it,
  so no change to either method is required by this parameter's addition
- `utils.load_tick_bar_aggregates_with_history()`'s 3-tier fallback (cache
  hit / on-the-fly compute / genuine data absence — see utils.md) is
  resolved entirely within that utility, before `tick_bar_history` reaches
  this module — `session_start_compute()` always receives a ready-to-use
  wide DataFrame (or `None`), never a tier indicator
