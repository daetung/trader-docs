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
                   # covers bars t-N ... t-1 only
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
    {name}_slope             ← OLS slope (normalized by price level)
    {name}_intercept
    {name}_r_squared
```

### 4. Window Comparison
```
Applies to: volume, TPM, avg_vol_per_tick, REFERENCE_SESSION today series

Split window into current_half and prev_half:
Output keys:
    {name}_window_ratio      ← mean(current_half) / mean(prev_half)
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
    price_rN, pivot_hour_rN, bars_since_rN, prominence_rN   (for N = 1..n_levels)
    price_sN, pivot_hour_sN, bars_since_sN, prominence_sN

Output keys (per level N = 1..n_levels):
    dist_rN_pct              ← (price_rN - P_entry) / P_entry
    bars_since_rN            ← passed through from IndicatorCalculator output
    prominence_rN            ← passed through from IndicatorCalculator output

    dist_sN_pct              ← (P_entry - price_sN) / P_entry
    bars_since_sN            ← passed through from IndicatorCalculator output
    prominence_sN            ← passed through from IndicatorCalculator output

Composite:
    nearest_resistance_dist  ← min(dist_r1..rN)
    resistance_density       ← mean(dist_r1..rN)
    nearest_support_dist     ← min(dist_s1..sN)
    sr_asymmetry             ← nearest_resistance_dist - nearest_support_dist

Note: pivot_hour_rN / pivot_hour_sN are NOT included in output features —
they are internal to IndicatorCalculator for bars_since recomputation.
```

NaN padding rule: if fewer than n_levels extrema found, pad missing levels with NaN.

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
| **rvol** | **statistical_summary** | **window_comparison** |
| **rel_dvol** | **statistical_summary** | **window_comparison** |
| **relative_avg_vol_per_tick** | **statistical_summary** | **window_comparison** |
| **intra_season_{metric}** | **statistical_summary** | **window_comparison** |
| **gap_pct** | **(Vectorizer 미사용 — FeatureExtractor inline 처리)** | — |
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
the primary method. For `window_comparison` (secondary), only
`{name}_window_delta` (a difference) is used for `obv`/`ad` —
`{name}_window_ratio` is excluded for these two indicators specifically,
since a ratio of two segments of a cumulative series can still divide by a
near-zero value even though the delta form cannot.

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
        FeatureExtractor calls vectorizer.sr_distance() directly for sr_levels.
        Passing method="sr_distance" to transform() raises ValueError.
        """
        ...
```

---

## Constraints

- No indicator calculation logic here — all inputs come from IndicatorCalculator
- All method parameters (window splits, prominence settings) from `pipeline_config.yaml`
- Feature names must be deterministic and stable across runs (used as LightGBM column names)
- NaN values are allowed in output — LightGBM handles them natively
- `transform()` must raise `ValueError` if method="sr_distance" is passed
- `sr_distance()` is always called directly by FeatureExtractor, never via `transform()`
- `level_distance()` is dispatched via `transform()` with `reference_price` as kwarg
- `level_distance()` input is a pd.Series of a single level column (e.g. `fib_618`, `r1`)
  — FeatureExtractor iterates over level columns individually
- `sr_distance()` receives `bars_since_rN` and `prominence_rN` from IndicatorCalculator
  output directly — no recomputation; pass-through to output dict
- `pivot_hour_rN` / `pivot_hour_sN` from sr_levels output are NOT included in Vectorizer output
- `dist_rN_pct` sign convention: positive = level above P_entry (resistance);
  `dist_sN_pct` sign convention: positive = level below P_entry (support)
- Same sign convention applies to `level_distance()`: positive = level above P_entry
- `gap_pct` is never passed to any Vectorizer method — FeatureExtractor inline only
- `window_comparison()` on `obv`/`ad`: caller (FeatureExtractor) requests only
  `{name}_window_delta`; `{name}_window_ratio` must not be generated for these
  two indicators — see "obv/ad mapping rationale" above
