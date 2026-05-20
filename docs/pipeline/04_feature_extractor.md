# Module: FeatureExtractor

**File:** `src/preprocessing/feature_extractor.py`
**Depends on:** `docs/data/data_boundary.md`, `docs/pipeline/02_indicator_calculator.md`, `docs/pipeline/03_vectorizer.md`

---

## Role

Central integration entrypoint for all feature generation.
Orchestrates IndicatorCalculator → Vectorizer → MetaFeatures → TemporalFeatures
and outputs a single fixed-length feature vector per entry point.
Also exposes a structured intermediate output interface for the Visualization tool.

---

## Input / Output

**Input:**
```python
bars: pd.DataFrame      # ohlcv_1min, strictly bars t-N ... t-1
ticks: pd.DataFrame     # tick_10, strictly ticks before t bar open
meta: dict              # stock_meta row for this ticker
entry: dict             # {ticker, date, hour, p_entry}
config: dict            # loaded from pipeline_config.yaml
```

**Output:**
```python
feature_vector: dict[str, float]
    # flat dict of all scalar features
    # key format: "{indicator}_{transform}_{stat}"
    # e.g. "rsi_stats_mean", "volume_window_ratio", "dist_r1_pct"
```

---

## Processing Pipeline

```
bars (t-N ... t-1)
    │
    ├─► MissingBarClassifier          (pre-processing step)
    │       classify gaps as halt / no_trade
    │       forward-fill no_trade bars; keep halt bars as NaN
    │       covers all time periods (pre/regular/after-market)
    │       output: cleaned bars + market_structure_features dict
    │
    ├─► IndicatorCalculator
    │       run all enabled indicators per config
    │       output: dict[str → pd.DataFrame]  (time-series per indicator)
    │
    ├─► Vectorizer
    │       for each indicator series, apply mapped transform method(s)
    │       output: dict[str, float]
    │
    ├─► MetaFeatureExtractor  (inline)
    │       log transform monetary fields
    │       encode sector as integer category
    │       output: dict[str, float]
    │
    ├─► TemporalFeatureExtractor  (inline)
    │       derive time/date features from entry.hour, entry.date
    │       lookup holiday calendar from DuckDB
    │       output: dict[str, float]
    │
    └─► merge all → single feature_vector
```

---

## extract_batch() Implementation Strategy

`extract_batch()` must not recompute indicators per entry point.
The correct strategy is:

```
1. Compute all indicators once for the full ticker bars DataFrame
   → dict[str, pd.DataFrame]  (time-series per indicator, full date range)

2. For each entry point in entry_points:
   a. Slice indicator time-series up to t-1 bar (by hour index)
   b. Apply Vectorizer to sliced series → feature vector
   c. Append meta and temporal features

3. Concatenate all feature vectors → feature matrix (pd.DataFrame)
```

Indicator calculation is performed **once per ticker**, not once per entry point.
Entry point-level slicing and vectorization is the only per-entry-point operation.
Python iteration over entry points for slicing is permitted.

---

## Meta Features

```python
meta_features = {
    "log_market_cap":       log(market_cap) if market_cap > 0 else NaN,
    "log_price_52h":        log(price_52h),
    "log_price_52l":        log(price_52l),
    "log_avg_volume":       log(avg_volume),
    "price_52_range_ratio": (p_entry - price_52l) / (price_52h - price_52l),
    "sector_code":          int  # from configs/sector_map.json; unknown → -1
}
```

Sector encoding map is loaded via `utils.load_encoding_map("sector_map")`.
Built once from all unique sectors in `stock_meta` and persisted to `configs/sector_map.json`.
`sector_code` must be registered as LightGBM categorical column.

---

## Temporal Features

```python
temporal_features = {
    # Time position — raw values (no scale transform for LightGBM)
    "minute_of_session": minutes_since_0930,       # int; negative for pre-market entries
    "hour_of_day":       int(hour[:2]),            # int

    # Categorical (LightGBM categorical)
    "day_of_week":       date.strftime("%a"),      # "Mon"/"Tue"/"Wed"/"Thu"/"Fri"
                                                   # encoded via day_of_week_map.json
                                                   # registered as LightGBM categorical

    # Binary flags
    "is_pre_market":     1 if hour < "093000" else 0,
    "is_first_30min":    1 if 0 <= minute_of_session < 30 else 0,
    "is_last_30min":     1 if minute_of_session >= 360 else 0,
    "is_pre_holiday":    1 if next trading day is a holiday else 0,
    "is_post_holiday":   1 if previous trading day was a holiday else 0,
}
```

Temporal features use `entry.hour` (t bar open time) as the reference.
`entry.hour` is the time at which detection fired — it is not t bar OHLCV data,
and its use as a temporal feature does not constitute data leakage.

Holiday calendar sourced from `us_holidays` table in DuckDB.
`day_of_week` encoding map loaded via `utils.load_encoding_map("day_of_week_map")`.
`is_monday` and `is_friday` binary features must NOT be generated (subsumed by `day_of_week`).

Note on scaling: LightGBM is tree-based and invariant to monotonic transforms.
`minute_of_session` is kept as raw integer.
Scaling will be applied at MLP comparison stage only.

---

## Market Structure Features

Derived from `trading_halts` table, missing bar classification, and
synthetic (no_trade) bar analysis within the lookback window.

```python
market_structure_features = {
    # Halt-related
    "had_halt_today":           1 if any halt before t-1 bar else 0,
    "bars_since_last_halt":     int / NaN,   # bars since last halt ended; NaN if no halt
    "halt_reason_code":         int,         # category; NaN if no halt today
                                             # registered as LightGBM categorical
    "halt_count_today":         int,         # total halts this session before t-1

    # Missing bar composition
    "missing_bar_count":        int,         # total no_trade gap bars in lookback window

    # Synthetic bar quality indicators
    # These capture the degree to which indicator values in the lookback
    # window may be distorted by forward-filled synthetic bars.
    "synthetic_bar_ratio":      float,       # no_trade_bars / total_bars_in_window
                                             # range [0.0, 1.0]
                                             # high value → many indicators computed
                                             # on forward-filled prices (reduced reliability)
    "consecutive_synthetic_max": int,        # longest run of consecutive no_trade bars
                                             # in lookback window
                                             # high value → extended flat price segments
                                             # distorting slope/trend indicators
}
```

**Synthetic bar ratio computation:**
```
lookback_bars = all bars in the t-N...t-1 window passed to IndicatorCalculator
no_trade_bars = bars classified as no_trade by MissingBarClassifier
                (halt bars excluded — they are NaN, not forward-filled)

synthetic_bar_ratio      = len(no_trade_bars) / len(lookback_bars)
consecutive_synthetic_max = max run length of consecutive no_trade bars
```

These features allow the model to learn when its indicator inputs are less
reliable due to illiquidity gaps, without modifying IndicatorCalculator itself.

---

## Visualization Interface

```python
@dataclass
class FeatureSnapshot:
    entry:             dict                    # {ticker, date, hour, p_entry}
    indicator_series:  dict[str, pd.DataFrame] # raw time-series per indicator
    feature_vector:    dict[str, float]        # final flat vector
    meta_features:     dict[str, float]
    temporal_features: dict[str, float]

def extract(
    self,
    bars, ticks, meta, entry,
    return_intermediate: bool = False,
) -> dict[str, float] | FeatureSnapshot: ...
```

Visualization tool calls `extract(..., return_intermediate=True)` to access
per-indicator time-series for plotting alongside final feature values.

---

## Interface

```python
class FeatureExtractor:
    def __init__(self, config: dict): ...

    def extract(
        self,
        bars: pd.DataFrame,
        ticks: pd.DataFrame,
        meta: dict,
        entry: dict,
        return_intermediate: bool = False,
    ) -> dict[str, float] | FeatureSnapshot: ...

    def extract_batch(
        self,
        entry_points: pd.DataFrame,
        bars: pd.DataFrame,
        ticks: pd.DataFrame,
        meta: dict,
    ) -> pd.DataFrame:
        """
        Batch feature extraction for one ticker.
        Indicators computed once for full bars DataFrame.
        Per-entry slicing and vectorization applied after.
        Returns feature matrix as pd.DataFrame.
        """
        ...

    def get_feature_names(self) -> list[str]:
        """
        Deterministic ordered list of all feature column names.
        Does NOT include p_entry, ticker, date, hour, or label columns.
        """
        ...
```

---

## Parquet Column Structure

Output parquet files contain the following column groups:

```
Identifier columns : ticker, date, hour, p_entry
Feature columns    : get_feature_names() — all float/int features
Label columns      : label_up5, label_up3, label_sw, label_dn3, label_dn5
Metadata columns   : is_dead_position, dead_position_case, is_ambiguous
```

`p_entry` is stored as an identifier column, not a feature.
It must never appear in `get_feature_names()` output.
It is required by BacktestEngine for fill price calculation.

`is_dead_position`, `dead_position_case`, `is_ambiguous` are stored as
metadata columns — not features. They must not appear in `get_feature_names()`.
They are used by ClassBalancer for pre-balance filtering.

---

## Constraints

- Feature column names must be deterministic and stable across runs
- `extract_batch()` computes indicators once per ticker — not once per entry point
- Disabled feature groups (per config) produce no columns
- `sector_code`, `day_of_week`, `halt_reason_code` must be passed as categoricals to LightGBM
- NaN values permitted — do not impute
- `p_entry` must never appear as a feature column
- `is_monday` and `is_friday` binary columns must NOT be generated
- Missing bar classification must run before IndicatorCalculator receives bars
- Halt bars (NaN) propagate naturally into indicator NaN — do not suppress
- Temporal features use `entry.hour` (t bar open time) as reference — this is not data leakage
- Encoding maps (sector, day_of_week) loaded via `utils.load_encoding_map()` for live inference compatibility
- `synthetic_bar_ratio` and `consecutive_synthetic_max` computed from MissingBarClassifier output
  — no changes to IndicatorCalculator or Vectorizer required
- `is_dead_position`, `dead_position_case`, `is_ambiguous` passed through from Labeler output
  — not computed by FeatureExtractor
