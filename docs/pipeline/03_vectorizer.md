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
Applies to: volume, TPM, avg_vol_per_tick

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

### 6. Multi-level S/R Distance
```
Applies to: sr_levels output from IndicatorCalculator
Reference: P_entry (passed explicitly)

Output keys (per level N = 1..n_levels):
    dist_rN_pct              ← (local_max_N - P_entry) / P_entry
    bars_since_rN            ← bars elapsed since local_max_N
    prominence_rN            ← scipy prominence score

    dist_sN_pct              ← (P_entry - local_min_N) / P_entry
    bars_since_sN
    prominence_sN

Composite:
    nearest_resistance_dist  ← min(dist_r1..rN)
    resistance_density       ← mean(dist_r1..rN)
    nearest_support_dist     ← min(dist_s1..sN)
    sr_asymmetry             ← nearest_resistance_dist - nearest_support_dist
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
| MA, EMA, MACD | linear_trend | statistical_summary |
| sr_levels | sr_distance | — |

---

## Interface

```python
class Vectorizer:
    def statistical_summary(self, series: pd.Series, name: str) -> dict[str, float]: ...
    def rate_of_change(self, series: pd.Series, name: str) -> dict[str, float]: ...
    def linear_trend(self, series: pd.Series, name: str) -> dict[str, float]: ...
    def window_comparison(self, series: pd.Series, name: str) -> dict[str, float]: ...
    def shape_features(self, series: pd.Series, name: str) -> dict[str, float]: ...
    def sr_distance(self, sr_df: pd.DataFrame, reference_price: float,
                    n_levels: int) -> dict[str, float]: ...

    def transform(self, series: pd.Series, method: str,
                  name: str, **kwargs) -> dict[str, float]:
        """Dispatch to named method. Used by FeatureExtractor."""
        ...
```

---

## Constraints

- No indicator calculation logic here — all inputs come from IndicatorCalculator
- All method parameters (window splits, prominence settings) from `pipeline_config.yaml`
- Feature names must be deterministic and stable across runs (used as LightGBM column names)
- NaN values are allowed in output — LightGBM handles them natively
