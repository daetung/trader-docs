# Module: Vectorizer

**File:** `src/preprocessing/vectorizer.py`
**Depends on:** `docs/data/data_boundary.md`

---

## Role

Convert time-series DataFrames (output of IndicatorCalculator) into fixed-length feature vectors suitable for LightGBM input.
Each transformation method is defined independently and mapped to specific indicator types via config.

---

## Input / Output

**Input:**
```python
series: pd.Series  # single indicator time-series, index = bar timestamps
                   # covers bars t-N ... t-1 only — the CONTINUOUS
                   # vectorization window: every bar of the prior
                   # indicators.lookback_days trading days and the entry
                   # date through t-1; a REFERENCE today series instead
                   # covers the REFERENCE_SESSION today window
                   # (data_boundary.md), not the CONTINUOUS window
                   # positions are bar order: the existing bars, a minute
                   # without a bar taking no position
reference_price: float  # P_entry — passed in for distance-based transforms only
```

**Output:**
```python
dict[str, float]  # named scalar features, e.g. {"rsi_mean": 54.2, "rsi_slope": -0.3}
```

---

## Transformation Methods

### 1. Statistical Summary
```
Applies to: technical indicators (RSI, ATR, BB %B, etc.)

Output keys:
    {name}_mean, {name}_std, {name}_min, {name}_max,
    {name}_skew, {name}_kurt,
    {name}_p25, {name}_p75,
    {name}_last          ← most recent value (t-1 bar)
```

### 2. Rate of Change
```
Applies to: price close, volume

Output keys:
    {name}_pct_change_mean   ← mean of bar-over-bar % changes
    {name}_pct_change_std
    {name}_momentum_n        ← (last - first) / first over window
    {name}_norm_return       ← cumulative return normalized by std
```

### 3. Linear Trend
```
Applies to: price close, moving averages

Output keys:
    {name}_slope             ← OLS slope (normalized by price level), its
                               x-axis the bar positions (bar order)
    {name}_intercept         ← NOT emitted for obv/ad: the indicator mapping
                               routes those two here, and an intercept is a
                               level of an origin-dependent cumulative series
    {name}_r_squared
```

### 4. Window Comparison
```
Applies to: volume, TPM, avg_vol_per_tick, REFERENCE_SESSION today series

Split window into current_half and prev_half by bar order (a REFERENCE today
series: its today window split the same way):
Output keys:
    {name}_window_ratio      ← mean(current_half) / mean(prev_half)
                               NOT emitted for obv/ad: a ratio of two segments
                               of a cumulative series can divide by a
                               near-zero value, which the delta form cannot
    {name}_window_delta      ← mean(current_half) - mean(prev_half)
```

### 5. Shape Features
```
Applies to: price, oscillators (RSI, stochastic)

Output keys:
    {name}_zero_crossing_rate   ← crossings of mean per bar count
    {name}_peak_count           ← number of local maxima
    {name}_trough_count         ← number of local minima
    {name}_trend_reversal_count ← direction changes
    {name}_above_mean_ratio     ← fraction of bars above series mean
```

### 6. Level Distance
```
Applies to: fibonacci_retracement, pivot_points
            (constant-value absolute price series, one column per level)
Reference: P_entry (passed explicitly)

Dispatched via transform() — standard pd.Series input interface.
FeatureExtractor iterates over each level column and calls transform() per column.

Output keys (per level column, e.g. "fib_618", "r1", "pp"):
    {name}_dist_pct   ← (level_price - P_entry) / P_entry
                        positive = level is above P_entry (resistance)
                        negative = level is below P_entry (support)

NaN padding: if level column is NaN (e.g. insufficient data for swing detection),
             output {name}_dist_pct = NaN.
```

### 7. Multi-level S/R Distance
```
Applies to: sr_levels output from IndicatorCalculator
Reference: P_entry (passed explicitly)

This method is NOT dispatched via transform().
FeatureExtractor calls sr_distance() directly on the Vectorizer instance.
See "sr_distance dispatch" note below.

Input: sr_df (pd.DataFrame) with columns:
    price_rN, bars_since_rN, prominence_rN   (for N = 1..n_levels)
    price_sN, bars_since_sN, prominence_sN

Output keys (per level N = 1..n_levels):
    dist_rN_pct              ← (price_rN - P_entry) / P_entry, sign kept
    bars_since_rN            ← passed through from IndicatorCalculator output
    prominence_rN_pct        ← prominence_rN / P_entry

    dist_sN_pct              ← (P_entry - price_sN) / P_entry, sign kept
    bars_since_sN            ← passed through from IndicatorCalculator output
    prominence_sN_pct        ← prominence_sN / P_entry

    In training the f_e rescale precedes the division by P_entry.
    IndicatorCalculator's raw outputs are unchanged.

Composite — every level, price_rN and price_sN alike, by its side of P_entry,
a level above at distance (level - P_entry) / P_entry, one below at
(P_entry - level) / P_entry:
    nearest_resistance_dist  ← distance to the nearest level above P_entry
    nearest_support_dist     ← distance to the nearest level below P_entry
    resistance_density       ← mean distance of the levels above P_entry
    sr_asymmetry             ← nearest_resistance_dist - nearest_support_dist
The composites ignore NaN levels; a composite is NaN when no level lies on
its side.
```

NaN padding rule: if fewer than n_levels extrema found, pad missing levels with NaN.

### 8. S/R Channel Distance
```
Applies to: sr_channels output from IndicatorCalculator
Reference: P_entry (passed explicitly)

The dedicated method beside sr_distance(), likewise NOT dispatched via
transform(): FeatureExtractor calls sr_channel_distance() directly.

Input: sr_ch_df (pd.DataFrame) with columns, for N up to max_num_sr:
    sr_ch_hi_N, sr_ch_lo_N, sr_ch_strength_N   (NaN where absent)

Output keys (per band N):
    sr_ch_dist_N_pct         ← 0 when P_entry lies inside the band;
                               (lo - P_entry) / P_entry when the band lies above;
                               -(P_entry - hi) / P_entry when the band lies below
    sr_ch_width_N_pct        ← (hi - lo) / P_entry
    sr_ch_strength_N         ← passed through

Composite:
    sr_ch_nearest_resistance_dist ← the smallest (lo - P_entry) / P_entry over
                                    bands above P_entry
    sr_ch_nearest_support_dist    ← the smallest (P_entry - hi) / P_entry over
                                    bands below P_entry
    sr_ch_resistance_density      ← the mean of (lo - P_entry) / P_entry over
                                    bands above P_entry
    sr_ch_asymmetry               ← sr_ch_nearest_resistance_dist -
                                    sr_ch_nearest_support_dist
    sr_ch_in_channel              ← 1 when P_entry lies inside a band, else 0
The composites ignore NaN bands; a composite is NaN when no band lies on its
side. In training the f_e rescale precedes the distance computation.
```

---

## NaN Entries in a Window

Every method computes over the window's non-NaN entries:
- `statistical_summary`'s statistics over the non-NaN values;
- `linear_trend`'s OLS over the non-NaN points at their bar positions;
- `window_comparison`'s half means over each half's non-NaN values;
- `rate_of_change` over consecutive non-NaN pairs;
- `shape_features` over the sequence with NaN entries removed.

`{name}_last` is the last value as it stands, NaN included. An output is NaN
when the non-NaN count falls below what its statistic requires.

---

## Indicator → Method Mapping (default)

Configured in `pipeline_config.yaml`. Default mapping:

| Indicator | Primary Method | Secondary Method |
|---|---|---|
| price close | rate_of_change | linear_trend |
| volume | window_comparison | statistical_summary |
| RSI, stochastic | statistical_summary | shape_features |
| ATR, BB bandwidth | statistical_summary | — |
| TPM | window_comparison | statistical_summary |
| avg_vol_per_tick | window_comparison | — |
| **vol_weighted_buy_ratio** | **statistical_summary** | **window_comparison** |
| **avg_delta_per_tick** | **linear_trend** | **window_comparison** |
| **tick_realized_vol** | **linear_trend** | **window_comparison** |
| **path_efficiency** | **statistical_summary** | **window_comparison** |
| **vol_concentration** | **linear_trend** | **window_comparison** |
| **tick_burstiness** | **linear_trend** | **window_comparison** |
| MA, EMA, MACD | linear_trend | statistical_summary |
| fibonacci_retracement | level_distance | — |
| pivot_points | level_distance | — |
| sr_levels | sr_distance | — |
| sr_channels | sr_channel_distance | — |
| **rvol** | **statistical_summary** | **window_comparison** |
| **rel_dvol** | **statistical_summary** | **window_comparison** |
| **relative_avg_vol_per_tick** | **statistical_summary** | **window_comparison** |
| **intra_season_{metric}** | **statistical_summary** | **window_comparison** |
| **gap_pct** | **(Vectorizer unused — computed inline by FeatureExtractor)** | — |
| **obv, ad** | **linear_trend** | **window_comparison (delta only — see note)** |

`gap_percentile` is excluded from Vectorizer dispatch. FeatureExtractor inserts
the scalar value directly into the feature vector as `gap_pct`. No transform applied.

**relative_avg_vol_per_tick note:** unlike `gap_percentile` (a single scalar
per entry point), this indicator's own output (per 02_indicator_calculator.md)
is a per-bar ratio series spanning today's session-so-far — the same shape as
`rvol`/`rel_dvol`'s output. It is mapped identically for that reason, not as
a special case.

**obv/ad mapping rationale:** `obv`/`ad` are unbounded cumulative sums — their
absolute level (`statistical_summary`'s mean/min/max/last) is an artifact of
where the loaded bars window happened to start, not an economically meaningful
value, and is not comparable across samples. `rate_of_change`'s ratio form is
also unsuitable near a zero-crossing (division instability). `linear_trend`
(slope, r_squared) captures the classic OBV/AD interpretation — sustained
directional money flow — independent of the series' arbitrary origin, so it is
the primary method. `{name}_intercept` is EXCLUDED for these two indicators
specifically: an intercept is a level, and the same argument that rules out
`statistical_summary` rules it out. For `window_comparison` (secondary), only
`{name}_window_delta` (a difference) is used for `obv`/`ad` —
`{name}_window_ratio` is excluded for these two indicators specifically,
since a ratio of two segments of a cumulative series can still divide by a
near-zero value even though the delta form cannot.

After these exclusions every output remaining for `obv`/`ad` is
ORIGIN-INVARIANT: `slope` and `r_squared` are unchanged by adding a constant
to the series, and `window_delta` is a difference. That is what makes the two
paths agree. Both vectorize the same CONTINUOUS vectorization window
(`data_boundary.md`'s `bars t-N ... t-1`); what differs is the accumulation
ORIGIN carried into it, because the cumulative sum is computed before the
slice — training accumulates from the batch's lookback start, live from the
`lookback_days` seed window (`caching_calculator.md`). No re-accumulation
after slicing: it would break Strategy A's once-per-ticker computation and
the calculator's O(1) incremental discipline for no gain.

The origin-invariance rests on `02_indicator_calculator.md`'s Incremental
Computation Rule, I3: the window is rebased by exact subtraction to its first
non-NaN value before conversion to float, so the two paths' differing origins
never reach a float value.

---

## Interface

```python
class Vectorizer:
    def statistical_summary(self, series: pd.Series, name: str) -> dict[str, float]: ...
    def rate_of_change(self, series: pd.Series, name: str) -> dict[str, float]: ...
    def linear_trend(self, series: pd.Series, name: str) -> dict[str, float]: ...
    def window_comparison(self, series: pd.Series, name: str) -> dict[str, float]: ...
    def shape_features(self, series: pd.Series, name: str) -> dict[str, float]: ...
    def level_distance(self, series: pd.Series, reference_price: float,
                       name: str) -> dict[str, float]: ...
    def sr_distance(self, sr_df: pd.DataFrame, reference_price: float,
                    n_levels: int) -> dict[str, float]: ...
    def sr_channel_distance(self, sr_ch_df: pd.DataFrame,
                            reference_price: float,
                            max_num_sr: int) -> dict[str, float]: ...

    def transform(self, series: pd.Series, method: str,
                  name: str, **kwargs) -> dict[str, float]:
        """
        Dispatch to named method. Used by FeatureExtractor.

        Supported methods via transform(): statistical_summary, rate_of_change,
        linear_trend, window_comparison, shape_features, level_distance.

        level_distance requires reference_price passed as kwarg:
            transform(series, "level_distance", name, reference_price=P_entry)

        NOTE: sr_distance is explicitly excluded from transform() dispatch.
        sr_distance accepts pd.DataFrame (not pd.Series) and requires
        reference_price and n_levels arguments that do not fit the
        standard transform() interface.
        FeatureExtractor calls vectorizer.sr_distance() directly for sr_levels,
        and vectorizer.sr_channel_distance() for sr_channels, which is
        excluded the same way.
        Passing method="sr_distance" or method="sr_channel_distance" to
        transform() raises ValueError.
        """
        ...
```

---

## Constraints

- No indicator calculation logic here — all inputs come from IndicatorCalculator
- All method parameters (window splits) from `pipeline_config.yaml`
- Feature names must be deterministic and stable across runs (used as LightGBM column names)
- NaN values are allowed in output — LightGBM handles them natively
- Every method computes over a window's non-NaN entries; `{name}_last` keeps
  NaN as it stands — see "NaN Entries in a Window"
- `transform()` must raise `ValueError` if method="sr_distance" or
  method="sr_channel_distance" is passed
- `sr_distance()` and `sr_channel_distance()` are always called directly by
  FeatureExtractor, never via `transform()`
- `level_distance()` is dispatched via `transform()` with `reference_price` as kwarg
- `level_distance()` input is a pd.Series of a single level column (e.g. `fib_618`, `r1`)
  — FeatureExtractor iterates over level columns individually
- `sr_distance()` receives `bars_since_rN` and `prominence_rN` from IndicatorCalculator
  output directly — no recomputation; `bars_since_rN` passes through to the
  output dict, `prominence_rN` leaves it as `prominence_rN_pct`
  (÷ P_entry)
- `sr_channel_distance()` receives `sr_ch_hi_N`, `sr_ch_lo_N` and
  `sr_ch_strength_N` from IndicatorCalculator output directly;
  `sr_ch_strength_N` passes through
- `dist_rN_pct` sign convention: positive = level above P_entry (resistance);
  `dist_sN_pct` sign convention: positive = level below P_entry (support)
- Same sign convention applies to `level_distance()`: positive = level above P_entry
- `gap_pct` is never passed to any Vectorizer method — FeatureExtractor inline only
- `window_comparison()` on `obv`/`ad`: caller (FeatureExtractor) requests only
  `{name}_window_delta`; `{name}_window_ratio` must not be generated for these
  two indicators — see "obv/ad mapping rationale" above
- `linear_trend()` on `obv`/`ad`: caller (FeatureExtractor) requests only
  `{name}_slope` and `{name}_r_squared`; `{name}_intercept` must not be
  generated for these two indicators — an intercept is a LEVEL, and the same
  argument that excludes `statistical_summary` for a cumulative series
  excludes it. Same rationale paragraph as the line above
