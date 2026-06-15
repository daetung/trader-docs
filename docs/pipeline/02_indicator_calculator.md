# Module: IndicatorCalculator

**File:** `src/preprocessing/indicator_calculator.py`
**Depends on:** `docs/data/data_boundary.md` — read first

---

## Role

Central repository for all technical indicator calculations.
Every indicator used anywhere in the pipeline must be defined here as a method.
Output is always a time-series DataFrame aligned to the input bar index,
except `gap_percentile()` which returns a scalar float.

---

## Input / Output

**Input:**
```python
bars: pd.DataFrame
    columns: [ticker, date, hour, open, high, low, close, volume]
    # strictly bars t-N ... t-1 (fully closed bars only)
    # t bar must be excluded before passing to this module
    # may include pre-market and after-market bars

ticks: pd.DataFrame  (optional, for tick-based indicators)
    columns: [ticker, date, hour, price, volume]
    # strictly ticks with timestamp < t bar open
```

**Output per method:**
```python
pd.DataFrame  # same index as input bars, one or more indicator columns
              # NaN for warm-up period is acceptable

# Exception: gap_percentile() returns float (scalar)
```

---

## Method Categories

### Trend Indicators
```
ma(bars, n)
    → simple moving average of close [col: ma_{n}]

ema(bars, n)
    → exponential moving average of close [col: ema_{n}]
    → standard params: n=9, n=20

macd(bars, fast, slow, signal)
    → macd_line, signal_line, histogram
    → default: fast=12, slow=26, signal=9

parabolic_sar(bars, af_start, af_step, af_max)
    → sar value per bar, trend direction (+1 up / -1 down)
    → default: af_start=0.02, af_step=0.02, af_max=0.2

adx(bars, n)
    → ADX (Average Directional Index) — trend strength (0~100)
    → +DI, -DI (directional indicators)
    → note: ADX is the smoothed absolute value of DMI; both are returned together
    → default: n=14

dmi(bars, n)
    → +DI, -DI (same calculation as adx(); prefer calling adx() to get all three)
    → included as alias for clarity; do not duplicate calculation logic
```

### Momentum / Oscillator Indicators
```
rsi(bars, n)
    → Relative Strength Index (0~100)
    → default: n=14

roc(bars, n)
    → Rate of Change: (close - close[n]) / close[n] * 100
    → default: n=12

stochastic(bars, n, m)
    → %K = (close - lowest_low_n) / (highest_high_n - lowest_low_n) * 100
    → %D = SMA(%K, m)
    → default: n=14, m=3

cci(bars, n)
    → Commodity Channel Index
    → CCI = (typical_price - SMA(typical_price, n)) / (0.015 * mean_deviation)
    → typical_price = (high + low + close) / 3
    → default: n=20

mfi(bars, n)
    → Money Flow Index (volume-weighted RSI, 0~100)
    → money_flow = typical_price * volume
    → default: n=14
```

### Volatility Indicators
```
bollinger_bands(bars, n, k)
    → upper_band, middle_band (SMA), lower_band
    → pct_b = (close - lower) / (upper - lower)
    → bandwidth = (upper - lower) / middle
    → default: n=20, k=2

atr(bars, n)
    → Average True Range
    → true_range = max(high-low, |high-prev_close|, |low-prev_close|)
    → default: n=14

vr_volatility(bars, n)
    → Volatility Ratio: ATR(n) / close  (normalized volatility)
    → distinct from Volume Ratio — see volume section below
    → default: n=14
```

### Volume Indicators
```
volume_ma(bars, n)
    → simple MA of volume [col: volume_ma_{n}]

vr_volume(bars, n)
    → Volume Ratio: current volume / volume_ma(n)
    → measures relative volume surge — primary entry point signal
    → default: n=20

obv(bars)
    → On-Balance Volume: cumulative sum of signed volume
    → +volume if close > prev_close, -volume if close < prev_close

ad(bars)
    → Accumulation/Distribution Indicator
    → money_flow_multiplier = ((close - low) - (high - close)) / (high - low)
    → ad = cumsum(money_flow_multiplier * volume)

vwap(bars, reset_mode, regular_start)
    → Volume-Weighted Average Price
    → vwap = cumsum(typical_price * volume) / cumsum(volume)

    reset_mode = "date" [default]:
        Reset at date boundary — accumulates from first bar of the date
        (includes pre-market bars if present)

    reset_mode = "regular_session":
        Reset at regular_start (default "093000") each date
        Bars before regular_start are excluded from VWAP calculation (NaN)

    note: VWAP reset_mode is read from config and passed by FeatureExtractor
```

### Level-Based Indicators
```
pivot_points(bars)
    → Classic Pivot Point based on previous session's H/L/C
    → PP  = (prev_high + prev_low + prev_close) / 3
    → R1  = 2 * PP - prev_low
    → R2  = PP + (prev_high - prev_low)
    → R3  = prev_high + 2 * (PP - prev_low)
    → S1  = 2 * PP - prev_high
    → S2  = PP - (prev_high - prev_low)
    → S3  = prev_low - 2 * (prev_high - PP)
    → output: constant value per session bar (step function)
    → "previous session" = previous date's regular session (093000~155900)
    → output columns: pp, r1, r2, r3, s1, s2, s3 (absolute prices)
    → distance features derived in Vectorizer via level_distance()

fibonacci_retracement(bars, window_bars)
    → Identify swing high and swing low within window
    → swing_high = max(close[-window_bars:])  ← implemented via monotonic deque
    → swing_low  = min(close[-window_bars:])  ← implemented via monotonic deque
    → Compute standard Fibonacci levels:
         level_0    = swing_low
         level_236  = swing_low + 0.236 * (swing_high - swing_low)
         level_382  = swing_low + 0.382 * (swing_high - swing_low)
         level_500  = swing_low + 0.500 * (swing_high - swing_low)
         level_618  = swing_low + 0.618 * (swing_high - swing_low)
         level_786  = swing_low + 0.786 * (swing_high - swing_low)
         level_1000 = swing_high
    → output columns: fib_0, fib_236, fib_382, fib_500, fib_618, fib_786, fib_1000
      (absolute prices — varies per bar as window slides)
    → distance features derived in Vectorizer via level_distance()
    → window_bars default: 390 (1 trading session)

    Implementation note (monotonic deque):
        When called on the full ticker bars DataFrame, fibonacci_retracement()
        uses a monotonic deque internally to compute sliding window max/min
        in O(N) total time (N = total bars), not O(N × window_bars).
        This enables the "ticker당 1회 계산" strategy in extract_batch():
        the output is a time series with one fib level set per bar,
        sliced per entry point at t-1. NaN for warm-up period (first
        window_bars-1 bars).
```

### Support / Resistance (Multi-level, Dynamic)
```
sr_levels(bars, n_levels, window_bars)
    → Identify top-N local maxima and minima by scipy prominence score
    → For each of top-N resistance levels (local maxima):
         price_rN      : absolute price of local maximum
         pivot_hour_rN : HHMMSS of the bar where local maximum occurred
         bars_since_rN : bars elapsed since local_max_N
                         (relative to last bar in passed window = t-1)
         prominence_rN : scipy prominence score
    → For each of top-N support levels (local minima):
         price_sN      : absolute price of local minimum
         pivot_hour_sN : HHMMSS of the bar where local minimum occurred
         bars_since_sN : bars elapsed since local_min_N
                         (relative to last bar in passed window = t-1)
         prominence_sN : scipy prominence score
    → Raw level data only — distance features (dist_rN_pct, dist_sN_pct,
      nearest_resistance_dist, sr_asymmetry, etc.) are computed by
      Vectorizer.sr_distance() using P_entry per entry point
    → Pad missing levels with NaN if fewer than n_levels extrema found
    → default: n_levels=3, window_bars=480 (~2 trading days)

    pivot_hour_rN / pivot_hour_sN:
        Absolute bar timestamp of the pivot. Used by extract_batch() when
        applying the "per-entry-point recomputation" strategy for sr_levels:
        bars_since_rN can be recomputed cheaply from pivot_hour without
        re-running scipy prominence.
        In the per-entry-point case, bars_since_rN from the fresh call is
        already t-1 relative — no recomputation needed.
```

### Bid-Ask Spread Indicators
```
lee_ready(ticks, bars)
    → Apply Lee-Ready (Tick Test) algorithm to tick_10 data
    → Classify each tick as buyer-initiated (+1) or seller-initiated (-1)
      Rule: uptick → buyer, downtick → seller, zero-tick → inherit prior direction
    → Per bar aggregation:
         buyer_initiated_ratio : buyer ticks / total ticks (0~1)
         tick_direction_momentum : EMA of buyer_initiated_ratio over last N bars
    → default: momentum_n = 5

roll_spread(bars, n)
    → Roll Model: spread = 2 * sqrt(-cov(ΔP_t, ΔP_{t-1}))
    → Uses 1min close prices; computed over rolling window n
    → Output: roll_spread_pct = roll_spread / close  (normalized)
    → If cov > 0 (model breaks down): return NaN
    → default: n = 20

hl_spread(bars)
    → Corwin-Schultz High-Low Spread estimator
    → Uses intraday high/low of each 1min bar
    → each 1min bar
    → spread = 2*(exp(alpha) - 1) / (1 + exp(alpha))
      where alpha derived from rolling 2-bar H/L ratios
    → Output: hl_spread_pct per bar (normalized by close)
```

### Missing Bar Classification
```
classify_missing_bars(bars, halts_df, date)
    → Detect gaps in bar sequence (missing HHMMSS within full trading day)
    → Covers pre-market, regular session, and after-market hours
    → For each missing bar slot:
         if slot overlaps any halt interval in halts_df → "halt"
             (halts can occur across all time periods, not only regular session)
         else                                           → "no_trade"
    → Output: dict with keys:
         missing_bar_slots  : list of HHMMSS strings
         classification     : dict[HHMMSS → "halt" | "no_trade"]
         halt_bar_count     : int
         no_trade_bar_count : int

    Filling strategy (applied before indicator calculation):
         no_trade bars: volume=0, OHLC = prior bar close (forward-fill)
         halt bars:     volume=0, OHLC = NaN (do NOT forward-fill)
```

### Tick-Derived Indicators
```
tpm(ticks, bars)
    → Ticks per minute, aligned to bar index

avg_vol_per_tick(ticks, bars)
    → Average volume per tick per bar
```

---

## REFERENCE_SESSION Indicators

Indicators that use prior sessions as a statistical baseline for current session comparison.
All computations reference only the passed `bars` DataFrame — no internal state.

REFERENCE_SESSION baselines (prior session averages) are pre-computed and stored in
`precomputed_session_stats` DuckDB table. At runtime, FeatureExtractor supplies
the pre-loaded baseline dict to avoid repeated DB lookups.

### Session Policy
```
SESSION_RESET  : VWAP — resets at session boundary; not applicable to prior session comparison
CONTINUOUS     : EMA, ATR, etc. — spans sessions naturally via lookback window
REFERENCE_SESSION: current session value normalized by prior session baseline
```

```
rvol(bars, date, n_sessions, session_mode)
    → Relative Volume: today's cumulative volume at time T
      divided by average cumulative volume at time T over prior N sessions
    → today_cumvol_at_T = sum(bars[date==today, hour<=T]["volume"])
    → baseline = mean over prior N sessions of same cumulative window
    → session_mode: follows entry_detector.session_mode
         "regular": prior regular session bars only (093000~155900)
         "pre":     prior pre-market bars (040000~092900)
         "combined": both
    → output: per-bar ratio series (today session only; prior session bars → NaN)
    → output column: rvol

rel_dvol(bars, date, n_sessions, session_mode)
    → Relative Dollar Volume: same as rvol but using close * volume (dollar volume)
    → output column: rel_dvol

gap_percentile(bars, date, n_sessions)
    → Today's gap = (today_regular_open - prev_close) / prev_close
      today_regular_open = open price of 093000 bar
      prev_close = close price of previous date's 155900 bar
    → Percentile rank of today's gap within prior N sessions' gap distribution
      (baseline loaded from precomputed_session_stats)
    → Returns float (scalar) — NOT a DataFrame
    → NaN conditions:
         - t bar = "093000" (today's open = t bar open = P_entry → FORBIDDEN)
         - t < "093000" (pre-market entry; regular session not yet open)
         - baseline has insufficient data (count < n_sessions / 2)
    → output: float | NaN  (single scalar per entry point)

intraday_seasonality(indicator_series, date, historical_bars,
                     n_sessions, delta_minutes, session_mode, metric)
    → Normalize a per-bar metric series against its prior-session baseline
      at the same time-of-day (±delta_minutes window)
    → indicator_series: pre-computed metric time series for today's bars
         (e.g. volume series from bars, tpm series from tpm(), etc.)
    → metric: "volume" | "price_return" | "tpm" | "buy_sell_ratio"
         buy_sell_ratio baseline loaded from precomputed_session_stats (tick-derived,
         pre-computed at migration step — not calculated at runtime)
    → delta_minutes: ±window for time averaging of baseline (default: 5)
    → Baseline is loaded from FeatureExtractor-supplied session_stats dict
      (derived from precomputed_session_stats table, smoothed per delta_minutes)
    → normalized_value[t_hour] = indicator_value[t_hour] / baseline[t_hour]
    → output: per-bar normalized series
    → output column: intra_season_{metric}  (e.g. intra_season_vol)
```

---

## Naming Conventions

| Category | Prefix / Pattern | Example |
|---|---|---|
| MA variants | `ma_{n}`, `ema_{n}` | `ma_20`, `ema_9` |
| Oscillators | lowercase name | `rsi`, `cci`, `mfi` |
| MACD outputs | `macd_line`, `macd_signal`, `macd_hist` | — |
| BB outputs | `bb_upper`, `bb_middle`, `bb_lower`, `bb_pctb`, `bb_bw` | — |
| ADX/DMI outputs | `adx`, `dmi_plus`, `dmi_minus` | — |
| SAR outputs | `sar`, `sar_trend` | — |
| Volume Ratio | `vr_volume` | distinct from `vr_volatility` |
| Pivot outputs | `pp`, `r1`..`r3`, `s1`..`s3` | absolute prices |
| Fibonacci outputs | `fib_0`, `fib_236`, `fib_382`, `fib_500`, `fib_618`, `fib_786`, `fib_1000` | absolute prices |
| S/R raw | `price_r{n}`, `pivot_hour_r{n}`, `bars_since_r{n}`, `prominence_r{n}` | `price_r1`, `pivot_hour_r1` |
| RVOL | `rvol` | `rvol_mean`, `rvol_last` |
| Relative dollar vol | `rel_dvol` | `rel_dvol_mean`, `rel_dvol_last` |
| Gap percentile | `gap_pct` | scalar, no suffix |
| Intraday seasonality | `intra_season_{metric}` | `intra_season_vol_mean` |

**VR disambiguation:** `vr_volume` = volume surge ratio. `vr_volatility` = ATR-normalized volatility.
Never use bare `vr` as a column name.

---

## Design Constraints

- All calculations must reference only the passed `bars` DataFrame — no internal state or global data access
- No feature extraction or vectorization logic in this file — computation only
- All window sizes and parameters must be accepted as arguments (not hardcoded)
- NaN handling: return NaN for warm-up period rows; do not forward-fill silently
- `dmi()` must not duplicate the calculation logic of `adx()` — call or delegate internally
- VWAP reset behavior is controlled by `reset_mode` argument — caller (FeatureExtractor) reads from config and passes explicitly
- `fibonacci_retracement()` and `pivot_points()` output level values as absolute prices, not distances — distance features are derived in Vectorizer via `level_distance()`
- `sr_levels()` outputs raw level data only (`price_rN`, `pivot_hour_rN`, `bars_since_rN`, `prominence_rN`) — P_entry is never passed to this method; distance features derived in Vectorizer via `sr_distance()`
- `bars_since_rN` is computed relative to the last bar in the passed window (always t-1) — it is a raw temporal measurement, not a derived feature
- `classify_missing_bars()` applies halt classification across all time periods — halts_df is the authoritative source regardless of hour
- `gap_percentile()` returns float, not DataFrame — FeatureExtractor handles inline (does not pass to Vectorizer)
- REFERENCE_SESSION baseline data must be supplied by caller (FeatureExtractor) via `session_stats` dict — IndicatorCalculator does not query DuckDB directly
- `intraday_seasonality()` takes a pre-computed indicator_series — does not internally call other indicator methods
- `fibonacci_retracement()` uses monotonic deque for O(N) sliding window max/min when processing full ticker bars

---

## Interface

```python
class IndicatorCalculator:
    # Trend
    def ma(self, bars: pd.DataFrame, n: int) -> pd.DataFrame: ...
    def ema(self, bars: pd.DataFrame, n: int) -> pd.DataFrame: ...
    def macd(self, bars: pd.DataFrame, fast: int, slow: int, signal: int) -> pd.DataFrame: ...
    def parabolic_sar(self, bars: pd.DataFrame, af_start: float,
                      af_step: float, af_max: float) -> pd.DataFrame: ...
    def adx(self, bars: pd.DataFrame, n: int) -> pd.DataFrame: ...
    def dmi(self, bars: pd.DataFrame, n: int) -> pd.DataFrame: ...

    # Momentum / Oscillator
    def rsi(self, bars: pd.DataFrame, n: int) -> pd.DataFrame: ...
    def roc(self, bars: pd.DataFrame, n: int) -> pd.DataFrame: ...
    def stochastic(self, bars: pd.DataFrame, n: int, m: int) -> pd.DataFrame: ...
    def cci(self, bars: pd.DataFrame, n: int) -> pd.DataFrame: ...
    def mfi(self, bars: pd.DataFrame, n: int) -> pd.DataFrame: ...

    # Volatility
    def bollinger_bands(self, bars: pd.DataFrame, n: int, k: float) -> pd.DataFrame: ...
    def atr(self, bars: pd.DataFrame, n: int) -> pd.DataFrame: ...
    def vr_volatility(self, bars: pd.DataFrame, n: int) -> pd.DataFrame: ...

    # Volume
    def volume_ma(self, bars: pd.DataFrame, n: int) -> pd.DataFrame: ...
    def vr_volume(self, bars: pd.DataFrame, n: int) -> pd.DataFrame: ...
    def obv(self, bars: pd.DataFrame) -> pd.DataFrame: ...
    def ad(self, bars: pd.DataFrame) -> pd.DataFrame: ...
    def vwap(self, bars: pd.DataFrame,
             reset_mode: str = "date",
             regular_start: str = "093000") -> pd.DataFrame: ...

    # Level-based
    def pivot_points(self, bars: pd.DataFrame) -> pd.DataFrame: ...
    def fibonacci_retracement(self, bars: pd.DataFrame,
                               window_bars: int) -> pd.DataFrame: ...
    def sr_levels(self, bars: pd.DataFrame, n_levels: int,
                  window_bars: int) -> pd.DataFrame: ...

    # Spread / Market Microstructure
    def lee_ready(self, ticks: pd.DataFrame, bars: pd.DataFrame,
                  momentum_n: int = 5) -> pd.DataFrame: ...
    def roll_spread(self, bars: pd.DataFrame, n: int = 20) -> pd.DataFrame: ...
    def hl_spread(self, bars: pd.DataFrame) -> pd.DataFrame: ...

    # Missing bar classification
    def classify_missing_bars(
        self, bars: pd.DataFrame,
        halts_df: pd.DataFrame,
        date: str,
    ) -> dict: ...

    # Tick-derived
    def tpm(self, ticks: pd.DataFrame, bars: pd.DataFrame) -> pd.DataFrame: ...
    def avg_vol_per_tick(self, ticks: pd.DataFrame, bars: pd.DataFrame) -> pd.DataFrame: ...

    # REFERENCE_SESSION
    def rvol(
        self, bars: pd.DataFrame, date: str,
        n_sessions: int, session_mode: str,
    ) -> pd.DataFrame: ...

    def rel_dvol(
        self, bars: pd.DataFrame, date: str,
        n_sessions: int, session_mode: str,
    ) -> pd.DataFrame: ...

    def gap_percentile(
        self, bars: pd.DataFrame, date: str,
        n_sessions: int, session_stats: dict,
    ) -> float: ...

    def intraday_seasonality(
        self,
        indicator_series: pd.Series,
        date: str,
        n_sessions: int,
        delta_minutes: int,
        session_mode: str,
        metric: str,
        session_stats: dict,
    ) -> pd.DataFrame: ...
```

---

## Config Keys (pipeline_config.yaml)

```yaml
indicators:
  lookback_days: 5              # CONTINUOUS indicators lookback (days)
                                # Constraint: lookback_days * 390 >= max(window_bars)
  ma_windows: [5, 10, 20]
  ema_windows: [9, 20]
  macd: {fast: 12, slow: 26, signal: 9}
  rsi_window: 14
  roc_window: 12
  stochastic: {n: 14, m: 3}
  cci_window: 20
  mfi_window: 14
  bollinger: {n: 20, k: 2}
  atr_window: 14
  adx_window: 14
  parabolic_sar: {af_start: 0.02, af_step: 0.02, af_max: 0.2}
  volume_ma_windows: [5, 20]
  vwap:
    reset_mode: "date"
    regular_start: "093000"
    precalculate_bars: 0        # SESSION_RESET — session start precalculation not applicable
  fibonacci:
    window_bars: 390
    precalculate_bars: "window" # precalculate window_bars worth of bars at session start
  lee_ready:
    momentum_n: 5
    precalculate_bars: 0        # tick-based — precalculation not applicable
  roll_spread_window: 20
  # hl_spread: no params (bar-level calculation)
  sr_levels:
    n_levels: 3
    window_bars: 480
    precalculate_bars: 0        # per-entry-point recomputation in live mode;
                                # date당 1회 per-entry-point in training
  tpm:
    precalculate_bars: 0        # tick-based
  avg_vol_per_tick:
    precalculate_bars: 0        # tick-based

  # REFERENCE_SESSION
  reference_session:
    n_sessions: 20              # prior sessions for baseline; independent of lookback_days
    delta_minutes: 5            # ±window for intraday_seasonality time averaging

  # precalculate_bars default for CONTINUOUS indicators (not explicitly listed above):
  # "lookback" → lookback_days * 390 bars precalculated at session start
  # Applied to: ma, ema, macd, parabolic_sar, adx, dmi, rsi, roc, stochastic,
  #             cci, mfi, bollinger_bands, atr, vr_volatility, volume_ma, vr_volume,
  #             obv, ad, roll_spread, hl_spread
```
