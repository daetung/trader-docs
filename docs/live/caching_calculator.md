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

**Same inputs, same values.** Caching changes what is recomputed, never what
is computed: for one bar and one ticker this class returns what
IndicatorCalculator returns, bitwise, under 02_indicator_calculator.md's
Incremental Computation Rule (I1–I5b).
The one admitted divergence is the multi-day
component of a tick-derived indicator declared `"session-only"`, whose live
window is today's ticks by declaration — see 02_indicator_calculator.md's
Live-mode windowing item, which owns that declaration and is the only place a
divergence may be introduced. The entries listed as not cached below are not
that exception: `sr_levels` and `sr_channels` are recomputed per entry point,
so they return base-class values.

---

## Class Structure

```python
class CachingIndicatorCalculator(IndicatorCalculator):
    """
    Live mode indicator calculator with two-layer cache.

    Layer 1 (fixed — set once at session start, never updated intra-session):
        pivot_points: based on prev session H/L/C — fixed for the day
        gap_pct:      based on today's regular open, the open of the first
                      regular-session bar — fixed after regular session open

    Layer 2 (precalculated at session start, incrementally updated on each bar close):
        All CONTINUOUS indicators up to D-1 bars at session start,
        then incrementally updated as today's bars arrive.
        fibonacci: monotonic deques over high and low — O(1) update per bar.

    Layer 2, session-scoped incremental (precalculate_bars 0, no prior-day
    seed; on_bar_close() appends from the day's first bar):
        vwap: SESSION_RESET — always bar-by-bar, no lookback option
        the REFERENCE_SESSION today series — rvol, rel_dvol,
              intraday_seasonality, relative_avg_vol_per_tick: each bar's
              today metric divided by its slot's baseline (session_stats)

    Not cached (computed on demand, session-only by default):
        sr_levels:   per-entry-point recomputation via get_for_entry()
                     (scipy prominence is not incrementally computable)
        sr_channels: per-entry-point recomputation via get_for_entry()
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

    # Keys: all indicators with precalculate_bars > 0 (except sr_levels,
    # sr_channels, gap_pct), and the session-scoped incremental entries —
    # vwap and the REFERENCE_SESSION today series
    # Value: time series DataFrame, grows one row per on_bar_close() call
```

---

## Methods

### `CachingIndicatorCalculator(config, ticker, session_date)`

```python
def __init__(self, config: dict, ticker: str, session_date: str) -> None:
    """
    Sets self._ticker = ticker, self._today_date = session_date and
    self._n_sessions = config["indicators"]["reference_session"]["n_sessions"].
    One instance serves one ticker for one session date.
    """
```

### `session_stats` (read-only property)

```python
@property
def session_stats(self) -> dict:
    """
    The baselines set_session_stats() set (self._session_stats). Read-only;
    LiveModeRunner passes the instance to the Inferencer, whose step 6 hands
    this property to extract() (inferencer.md).
    """
```

### `session_start_compute(historical_bars, tick_bar_history, db_conn)`

Called once by LiveModeRunner at session start (before market open).

```python
def session_start_compute(
    self,
    historical_bars: pd.DataFrame,
    tick_bar_history: pd.DataFrame | None,
    db_conn: duckdb.DuckDBPyConnection,
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
        tick-derived precalculate_bars value. Its columns carry
        tick_bar_aggregates' stored indicator names — lee_ready's is
        buyer_initiated_ratio (db_schema.md).
    db_conn: the calling thread's handle (live_mode_runner.md's P-5), read
        by Step 0.

    Step 0: Corporate-event bar adjustment (before any indicator computation)
        historical_bars = utils.adjust_bars_for_corporate_events(
            historical_bars, self._ticker, self._today_date, db_conn
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
            "lookback" → use the whole historical_bars
            "window"   → use historical_bars[-window_bars:]
            integer N  → use historical_bars[-N:]

        self._cache[indicator] = self._compute_indicator(indicator, bars_slice)

        fibonacci special case:
            Initialize the monotonic deques from historical_bars — `high` for
            swing_high, `low` for swing_low.
            self._fib_max_deque, self._fib_min_deque ready for O(1) updates.

    Step 2b: Layer 2 — precalculate from tick_bar_history (tick-derived
             indicators with precalculate_bars: "lookback" only)
        For each tick-derived indicator configured "lookback":
            self._cache[indicator] = tick_bar_history[indicator column],
                reindexed to historical_bars' index — a bar without a tick
                bundle NaN (M47)
        A "lookback" lee_ready seeds buyer_initiated_ratio, and
        tick_direction_momentum is computed over it through
        IndicatorCalculator's update step with indicators.lee_ready.momentum_n
        (02_indicator_calculator.md's I1, NaN-term handling).
        Indicators still configured 0 (the default) are unaffected — they
        remain in the session-only, on_bar_close()-accumulated path below.
        If tick_bar_history is None (no indicator uses "lookback"), this
        step is a no-op.
    """
```

### `on_regular_session_open(bars_through_regular_open, dividend_amount)`

Called once when the first regular-session bar (utils.session_phase()
'regular') is confirmed closed, the warm-restart replay included.

```python
def on_regular_session_open(
    self,
    bars_through_regular_open: pd.DataFrame,
    dividend_amount: float,
) -> None:
    """
    Computes gap_percentile (requires the regular open, the first
    regular-session bar's open).
    Records the hour of the bar it ran on (self._regular_open_hour), read
    for live_scan_daily's regular_open_total and regular_open_late at
    Session Shutdown (live_mode_runner.md).
    session_stats must be available (set via set_session_stats() before this call).
    dividend_amount: the ticker's ex-dividend gross amount for today, 0.0 if
        none — supplied by LiveModeRunner from its today-dividend map
        (live_mode_runner.md), the live counterpart of the scalar
        FeatureExtractor's Strategy D passes in training. No default: an
        omitted value would silently drop the correction. This class never
        looks it up.
    """
    self._fixed["gap_pct"] = self.gap_percentile(
        bars_through_regular_open, self._today_date, self._n_sessions,
        self._session_stats, dividend_amount,
    )
    self._regular_open_hour = bars_through_regular_open["hour"].iloc[-1]
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
        Exact arithmetic (02_indicator_calculator.md's Incremental
        Computation Rule):
        - the running sums and sums of squares are I2 exact accumulators;
        - a zero-range term follows I5a;
        - a halt contributes no term (I5b);
        - a NaN term follows NaN-term handling and W2.

    For fibonacci (monotonic deques):
        max deque.add(new_bar["high"]); min deque.add(new_bar["low"])
        self._cache["fibonacci"]["swing_high"] = max deque's max
        self._cache["fibonacci"]["swing_low"]  = min deque's min
        → fib levels recomputed from updated swing_high / swing_low

    For vwap (SESSION_RESET):
        Accumulate cumsum(typical_price * volume), cumsum(volume)
        → vwap value for this bar appended to self._cache["vwap"]

    For the REFERENCE_SESSION today series (rvol, rel_dvol,
    intraday_seasonality, relative_avg_vol_per_tick), from the day's first
    bar — each bar's today metric divided by its slot's baseline in
    self._session_stats, appended to self._cache[indicator]:
        - rvol and rel_dvol: from today's cumulative volume and dollar volume;
        - intraday_seasonality: from the bar's metric value, its tpm and
          buy_sell_ratio taken from the live tick-derived cache for that bar;
        - relative_avg_vol_per_tick: from the bar's avg_vol_per_tick.
        The values equal IndicatorCalculator's under the Incremental
        Computation Rule's I1.

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

    For sr_levels and sr_channels:
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
        Returns sr_df (single-row DataFrame with price_rN, bars_since_rN, etc.)

    sr_channels:
        Recomputed fresh from bars_up_to_t1 (02_indicator_calculator.md's
        sr_channels): the channel set of the latest recompute bar at or
        before t-1, NaN with none.
        Returns a single-row DataFrame with sr_ch_hi_N, sr_ch_lo_N,
        sr_ch_strength_N.

    gap_pct:
        Returns self._fixed.get("gap_pct", float("nan"))   (scalar float)

    pivot_points:
        Returns self._fixed["pivot_points"].loc[t-1_hour]  (sliced to t-1)

    All others:
        Returns self._cache[indicator][: t-1_hour]          (sliced to t-1)
        FeatureExtractor applies Vectorizer on the sliced series.
        The REFERENCE_SESSION today series take this default path, today's
        series through t-1; FeatureExtractor slices the REFERENCE_SESSION
        today window before the Vectorizer.
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

### `persist_to_db(db_conn)`

Serialize and store Layer 1 and Layer 2 to the `indicator_cache` table.
The table holds, per ticker, the seed computed on the current
`corporate_events` — the `session_start_compute()` body's result, before
any of today's bars are replayed.
Called in both `indicator_cache_mode` settings:
- by each Eager Pool worker as soon as it finishes its ticker;
- by a Scoped Mid-Session Recompute's new instance, for its step-1 seed
  (below);
- by the In-Process Premarket Recheck for a ticker with no calculator,
  after re-seeding it (live_mode_runner.md).

The stored rows serve:
- in either mode, as the R-2 crash-recovery backup;
- in "db" mode, also as the primary runtime store
  (live_mode_runner.md; db_schema.md's `indicator_cache`).

```python
def persist_to_db(
    self,
    db_conn: duckdb.DuckDBPyConnection,
) -> None:
    """
    Serializes self._fixed and self._cache to BLOBs and stores in indicator_cache,
    for self._ticker and self._today_date. The fibonacci entry's cache_data
    carries its deque state (self._fib_*).
    session_stats (self._session_stats) is NOT persisted here —
    it is reloaded separately via load_session_stats() when needed.

    Call after session_start_compute() + set_session_stats() are both complete.
    After this call:
    - "db" mode: the instance may be released from RAM;
    - "memory" mode: it stays the authoritative copy for the running
      session, and the stored rows are read only at a later start's Eager
      Pool step.
    Purges prior session_date entries for this ticker before inserting, and
    replaces the ticker's existing rows for the same session_date.
    """
```

### `load_from_db(db_conn)`

Restore Layer 1 and Layer 2 from the `indicator_cache` table.
Called:
- in "db" mode, on a ticker's first appearance in the watchdog working set;
- in "memory" mode, at every start's Eager Pool step, cold start and warm
  restart alike, for a ticker whose rows for today are present — a ticker
  without them is computed by `session_start_compute()` and persisted
  (live_mode_runner.md's Session Lifecycle step 5).

```python
def load_from_db(
    self,
    db_conn: duckdb.DuckDBPyConnection,
) -> None:
    """
    Restores self._fixed and self._cache from stored BLOBs, for self._ticker
    and self._today_date, the fibonacci deque state (self._fib_*) with them.
    Does NOT restore self._session_stats —
    caller must call set_session_stats() separately after this method.

    Expected call sequence (db mode, on a ticker's first appearance in
    the watchdog working set):
        calc = CachingIndicatorCalculator(config, ticker, today_date)
        calc.load_from_db(db_conn)
        stats = load_session_stats(db_conn, ticker, today_date, ...,
                                   closes=closes, ends=ends)
        calc.set_session_stats(stats)
        # then replay today's intraday bars via on_bar_close()

    A start restores a ticker through this method when today's rows for it
    are present — "memory" mode at its Eager Pool step, "db" mode at the
    ticker's first appearance (live_mode_runner.md).
    """
```

---

## Fibonacci Monotonic Deque

```python
# Internal state for fibonacci_retracement O(1) update
self._fib_max_deque:   collections.deque   # indices for sliding window max of high
self._fib_min_deque:   collections.deque   # indices for sliding window min of low
self._fib_high_window: collections.deque   # high values in window
self._fib_low_window:  collections.deque   # low values in window
self._fib_idx:         int                 # global bar counter

# on_bar_close() update:
def _update_fib(self, high: float, low: float) -> None:
    # expire stale entries from front of deques
    while self._fib_max_deque and self._fib_max_deque[0] <= self._fib_idx - window_bars:
        self._fib_max_deque.popleft()
    while self._fib_min_deque and self._fib_min_deque[0] <= self._fib_idx - window_bars:
        self._fib_min_deque.popleft()
    # maintain monotonic property
    while self._fib_max_deque and self._fib_high_window[...] <= high:
        self._fib_max_deque.pop()
    while self._fib_min_deque and self._fib_low_window[...] >= low:
        self._fib_min_deque.pop()
    # insert new value
    self._fib_max_deque.append(self._fib_idx)
    self._fib_min_deque.append(self._fib_idx)
    self._fib_high_window.append(high)
    self._fib_low_window.append(low)
    self._fib_idx += 1
    # current max/min — swing_high the maximum high, swing_low the minimum low
    swing_high = self._fib_high_window[self._fib_max_deque[0] - ...]
    swing_low  = self._fib_low_window[self._fib_min_deque[0] - ...]
    # recompute fib levels
    ...
```

---

## precalculate_bars Resolution

```
Config value  → Bars to use at session_start_compute()
"lookback"    → the whole historical_bars — every bar of the prior
                 indicators.lookback_days trading days, through
                 prior_trading_days() (the bars the broker supplied)
"window"      → historical_bars[-indicator.window_bars :]
0             → not precalculated (bar-by-bar in on_bar_close)
```

Indicators with `precalculate_bars: 0` (vwap, the REFERENCE_SESSION today
series, sr_levels, sr_channels, and all tick-derived
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
metadata_crawler.md), after the serving instance's `session_start_compute()`
already ran on stale (pre-event) data.

```python
def scoped_recompute(
    self,
    historical_bars: pd.DataFrame,
    tick_bar_history: pd.DataFrame | None,
    bars_today: pd.DataFrame,
    dividend_amount: float,
    db_conn: duckdb.DuckDBPyConnection,
) -> None:
    """
    One-shot correction for a single ticker whose indicators were seeded
    before its corporate event was known. db_conn is the calling thread's
    handle (live_mode_runner.md's P-5).

    LiveModeRunner calls it on a NEW instance (set_session_stats() first),
    never on the instance serving calculators[ticker]; bars_today is a
    snapshot taken when the recompute starts. The serving instance keeps
    answering until LiveModeRunner swaps the new one in at a Watchdog
    Polling Loop cycle boundary; the new instance has absorbed bars through
    the snapshot's last bar, and the Watchdog Polling Loop's Step 3
    multi-bar replay covers the bars after it (live_mode_runner.md).

      1. Re-run the session_start_compute() body — Step 0
         (adjust_bars_for_corporate_events, now seeing the new row) + Layer
         1 / Layer 2 seeding — exactly as at session start. This is a
         single-ticker full recompute, NOT an on_bar_close() call, so the
         O(1) per-call rule for on_bar_close() (see Constraints) is
         untouched. LiveModeRunner persists this step-1 seed through
         persist_to_db(), replacing the ticker's rows for today's
         session_date, before step 2.
      2. Replay today's bars so far via on_bar_close() from session start
         to the snapshot's last bar. Reuses the same cheap replay path Watchlist
         Append and Feed Outage Recovery already rely on (on_bar_close()
         is O(1) per bar, so replaying N bars is O(N) — fine for one
         ticker).
      3. If bars_today includes a regular-session bar, run
         on_regular_session_open(<bars through the first regular-session
         bar>, dividend_amount) after the replay — the replay is
         on_bar_close() only, so without this a recompute finishing after
         the first regular-session bar has closed would
         leave gap_pct as seeded before the event. dividend_amount is the
         ticker's today-dividend map entry, which LiveModeRunner re-reads
         before this call (live_mode_runner.md).

    After the swap, LiveModeRunner clears the ticker's
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
- `set_session_stats()` must be called before the ticker's first `on_bar_close()`
  — the REFERENCE_SESSION today series divide by its baselines from the day's
  first bar — and before `on_regular_session_open()`
- `on_regular_session_open()` runs once the first regular-session bar closes,
  and records that bar's hour
- `on_regular_session_open()` and `scoped_recompute()` take `dividend_amount` from LiveModeRunner's today-dividend map — this class never looks it up
- `on_bar_close()` must be called in bar-close order (chronological) — no random access
- Incrementally maintained values follow 02_indicator_calculator.md's
  Incremental Computation Rule:
  - running sums are I2 exact accumulators;
  - zero-range terms follow I5a;
  - a halt contributes no term (I5b);
  - NaN terms follow NaN-term handling and W2;
  - the result is bitwise equal to IndicatorCalculator's
- `sr_levels` and `sr_channels` are always recomputed fresh in `get_for_entry()` — never stored in Layer 2 cache
- The REFERENCE_SESSION today series are session-scoped Layer 2 entries:
  `precalculate_bars` 0, no prior-day history seeding them, values equal to
  IndicatorCalculator's under I1
- `gap_pct` is stored as a scalar in Layer 1 after `on_regular_session_open()` — never in Layer 2
- fibonacci deque state must be initialized — from historical_bars in `session_start_compute()`, or restored by `load_from_db()` — before `on_bar_close()` is called
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
- Thread safety of `_cache` and `_fixed` dicts is the responsibility of LiveModeRunner:
  one thread touches an instance at a time. Parallel session_start_compute() across tickers
  is safe as instances share no state; scoped_recompute() runs on a new instance swapped in
  afterwards for the same reason; each call's db_conn is the calling thread's handle
  (live_mode_runner.md's P-5)
- `persist_to_db()` is called in both `indicator_cache_mode` settings, after
  each Eager Pool worker, as the R-2 crash-recovery backup, and for a
  recheck re-seed (a Scoped Mid-Session Recompute's step-1 seed, or a
  ticker with no calculator)
- `load_from_db()` is called on a ticker's first appearance in "db" mode and
  at every start's Eager Pool step in "memory" mode, for a ticker whose rows
  for today are present
- `persist_to_db()` does not persist `_session_stats`;
  `load_from_db()` does not restore `_session_stats`
- After `persist_to_db()`:
  - "db" mode: the instance may be released from RAM;
  - "memory" mode: it stays the authoritative copy
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
