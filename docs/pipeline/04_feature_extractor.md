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

Sector encoding map is built once from all unique sectors in `stock_meta`
and persisted to `configs/sector_map.json`.
`sector_code` must be registered as LightGBM categorical column.

---

## Temporal Features

```python
temporal_features = {
    # Time position — raw values (no scale transform for LightGBM)
    "minute_of_session": minutes_since_0930,       # 0~389, int
    "hour_of_day":       int(hour[:2]),            # 9~15, int
    "month":             date.month,               # 1~12, int (categorical treatment optional)

    # Categorical (LightGBM categorical)
    "day_of_week":       date.strftime("%a"),      # "Mon"/"Tue"/"Wed"/"Thu"/"Fri"
                                                   # encoded via day_of_week_map.json
                                                   # registered as LightGBM categorical

    # Binary flags
    "is_first_30min":    1 if minute_of_session < 30 else 0,
    "is_last_30min":     1 if minute_of_session >= 360 else 0,
    "is_pre_holiday":    1 if next trading day is a holiday else 0,
    "is_post_holiday":   1 if previous trading day was a holiday else 0,
}
```

Holiday calendar sourced from `us_holidays` table in DuckDB (see db_schema.md).
`day_of_week` encoding map persisted to `configs/day_of_week_map.json`.
`is_monday` and `is_friday` binary features removed — subsumed by `day_of_week` categorical.

Note on scaling: LightGBM is tree-based and invariant to monotonic transforms.
`minute_of_session` (0~389) is kept as raw integer.
Scaling will be applied at MLP comparison stage only (`configs/pipeline_config.yaml: scaling.enabled`).

---

## Market Structure Features

Derived from `trading_halts` table and missing bar classification.

```python
market_structure_features = {
    "had_halt_today":       1 if any halt before t-1 bar else 0,
    "bars_since_last_halt": int / NaN,   # bars since last halt ended; NaN if no halt
    "halt_reason_code":     int,         # category; NaN if no halt today
                                         # registered as LightGBM categorical
    "halt_count_today":     int,         # total halts this session before t-1
    "missing_bar_count":    int,         # no_trade gap bars in lookback window
}
```

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
        """Vectorized batch for one ticker. Returns feature matrix."""
        ...

    def get_feature_names(self) -> list[str]:
        """Deterministic ordered list of all feature column names."""
        ...
```

---

## Constraints

- Feature column names must be deterministic and stable across runs
- `extract_batch()` must not call `extract()` in a Python loop
- Disabled feature groups (per config) produce no columns
- `sector_code`, `day_of_week`, `halt_reason_code` must be passed as categoricals to LightGBM
- NaN values permitted — do not impute
- `p_entry` must never appear as a feature column
- `is_monday` and `is_friday` binary columns must NOT be generated (replaced by `day_of_week`)
- Missing bar classification must run before IndicatorCalculator receives bars
- Halt bars (NaN) propagate naturally into indicator NaN — do not suppress
