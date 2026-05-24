# Module: FeatureExtractor

**File:** `src/preprocessing/feature_extractor.py`
**Depends on:** `docs/data/data_boundary.md`, `docs/pipeline/02_indicator_calculator.md`, `docs/pipeline/03_vectorizer.md`

---

## Role

Central integration entrypoint for all feature generation.
Orchestrates IndicatorCalculator → Vectorizer → MetaFeatures → TemporalFeatures
and outputs a single fixed-length feature vector per entry point.

---

## Input / Output

**Input:**
```python
bars: pd.DataFrame      # ohlcv_1min, strictly bars t-N ... t-1 (before t bar open)
ticks: pd.DataFrame     # tick_10, strictly ticks before t bar open
meta: dict              # stock_meta row for this ticker
entry: dict             # {ticker, date, hour, p_entry}
halts_df: pd.DataFrame  # trading_halts rows for this ticker-date
config: dict
```

**Output:**
```python
feature_vector: dict[str, float]
```

---

## Processing Pipeline

```
bars (t-N ... t-1)
    │
    ├─► MissingBarClassifier
    │       classify gaps as halt / no_trade (using halts_df)
    │       forward-fill no_trade bars; keep halt bars as NaN
    │       output: cleaned bars + market_structure_features dict
    │
    ├─► IndicatorCalculator
    │       run all enabled indicators per config
    │       output: dict[str → pd.DataFrame]
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

```
1. Compute all indicators once for the full ticker bars DataFrame

2. For each entry point in entry_points:
   a. Slice indicator time-series up to t-1 bar
   b. Apply Vectorizer → feature vector
   c. Append meta and temporal features

3. Concatenate all feature vectors → feature matrix (pd.DataFrame)
```

Indicator calculation is performed **once per ticker**, not once per entry point.

---

## Interface

```python
class FeatureExtractor:
    def __init__(self, config: dict, db_conn: duckdb.DuckDBPyConnection): ...

    def extract_batch(
        self,
        entry_points: pd.DataFrame,
        bars: pd.DataFrame,
        ticks: pd.DataFrame,
        meta: dict,
        halts_df: pd.DataFrame,
    ) -> pd.DataFrame:
        """
        Batch feature extraction for all entry points of a single ticker.
        bars and ticks must be strictly before t bar open (data boundary enforced by caller).
        halts_df passed explicitly for MissingBarClassifier and market_structure_features.
        """
        ...

    def extract(
        self,
        bars: pd.DataFrame,
        ticks: pd.DataFrame,
        meta: dict,
        entry: dict,
        halts_df: pd.DataFrame,
        return_intermediate: bool = False,
    ) -> dict[str, float] | tuple[dict[str, float], dict]:
        """
        Single entry point feature extraction.
        Used in live inference (Inferencer) and visualization.
        halts_df passed explicitly — consistent with extract_batch() pattern.
        """
        ...

    def get_feature_names(self) -> list[str]:
        """Return ordered list of all feature column names (excluding identifiers)."""
        ...

    def get_feature_schema(self) -> dict[str, str]:
        """
        Return {feature_name: dtype_str} for all features.
        dtype_str: "continuous" | "categorical"
        Used by Trainer to derive categorical_cols for LightGBM.
        """
        ...
```

---

## Temporal Features

```python
temporal_features = {
    "minute_of_session": int,          # minutes since session open (entry.hour reference)
    "day_of_week":       int,          # 0=Mon ... 4=Fri (encoded via day_of_week_map)
    "is_near_open":      int,          # 1 if within first 30 min of regular session
    "is_near_close":     int,          # 1 if within last 30 min of regular session
    "is_holiday_week":   int,          # 1 if trading week contains a US holiday
}
```

`entry.hour` is the clock time at which detection fired — not t bar OHLCV.
Holiday calendar sourced from `us_holidays` table in DuckDB.

---

## Market Structure Features

```python
market_structure_features = {
    "had_halt_today":            int,    # 1 if any halt before t-1 bar
    "bars_since_last_halt":      int,    # NaN if no halt today
    "halt_reason_code":          int,    # encoded via halt_reason_code_map; 0 = no halt
    "halt_count_today":          int,
    "missing_bar_count":         int,
    "synthetic_bar_ratio":       float,
    "consecutive_synthetic_max": int,
}
```

Halt data sourced from `halts_df` (passed explicitly — not fetched internally).

---

## Constraints

- Feature column names must be deterministic and stable across runs
- `extract_batch()` computes indicators once per ticker — not once per entry point
- Disabled feature groups produce no columns
- `sector_code`, `day_of_week`, `halt_reason_code` must be passed as categoricals to LightGBM
- NaN values permitted in continuous features — do not impute
- `p_entry` must never appear as a feature column
- `is_monday` and `is_friday` binary columns must NOT be generated
- Missing bar classification must run before IndicatorCalculator receives bars
- Halt bars (NaN) propagate naturally into indicator NaN — do not suppress
- Temporal features use `entry.hour` (t bar open time) as reference — not data leakage
- Encoding maps (`sector_map`, `day_of_week_map`, `halt_reason_code_map`) loaded via
  `utils.load_encoding_map()` internally — Inferencer does not inject maps
- `sector_code` unknown → 0; known sectors → 1, 2, 3, ...
- `halt_reason_code` no halt → 0; known codes → 1, 2, ...; unknown codes → -1
- All "categorical" typed features are guaranteed integers, never NaN
- `halts_df` passed explicitly to both `extract_batch()` and `extract()`;
  consistent with Labeler and BacktestEngine patterns (no internal DB queries)
- `sr_distance()` called directly by FeatureExtractor — never via `transform()`
- `is_dead_position`, `dead_position_case`, `is_ambiguous` passed through from Labeler —
  not computed by FeatureExtractor
