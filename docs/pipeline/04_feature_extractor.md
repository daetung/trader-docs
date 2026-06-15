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
    # flat dict of all scalar features
    # key format: "{indicator}_{transform}_{stat}"
    # e.g. "rsi_stats_mean", "volume_window_ratio", "dist_r1_pct"
```

---

## Constructor — Dependency Injection

```python
class FeatureExtractor:
    def __init__(
        self,
        config: dict,
        db_conn: duckdb.DuckDBPyConnection,
        calculator: IndicatorCalculator | None = None,
    ):
        """
        calculator: injectable IndicatorCalculator instance.
            Training:  IndicatorCalculator()  — stateless, default
            Live mode: CachingIndicatorCalculator()  — cache-aware subclass
            If None, defaults to IndicatorCalculator().

        Init validates config constraints:
            lookback_bars = config["indicators"]["lookback_days"] * 390
            max_window = max(sr_levels.window_bars, fibonacci.window_bars)
            if lookback_bars < max_window:
                raise ConfigurationError(
                    f"lookback_days*390={lookback_bars} < max indicator window={max_window}. "
                    f"Set lookback_days >= {ceil(max_window/390)}."
                )
            n_sessions and lookback_days are independent — no constraint between them.
        """
        ...
```

---

## Processing Pipeline

```
bars (t-N ... t-1)
    │
    ├─► MissingBarClassifier          (pre-processing step)
    │       classify gaps as halt / no_trade (using halts_df)
    │       forward-fill no_trade bars; keep halt bars as NaN
    │       covers all time periods (pre/regular/after-market)
    │       output: cleaned bars + market_structure_features dict
    │
    ├─► IndicatorCalculator (or CachingIndicatorCalculator)
    │       run all enabled indicators per config
    │       output: dict[str → pd.DataFrame]  (time-series per indicator)
    │
    ├─► Vectorizer
    │       for each indicator series, apply mapped transform method(s)
    │       sr_levels: call vectorizer.sr_distance() directly (not via transform())
    │       gap_pct: inline scalar — NOT passed to Vectorizer
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

`extract_batch()` must not recompute indicators per entry point where avoidable.
Different strategies apply per indicator category:

```
Strategy A — ticker당 1회 계산 (CONTINUOUS indicators):
    1. Compute indicators once for the full ticker bars DataFrame
       → dict[str, pd.DataFrame]  (time-series per indicator)
    2. For each entry point in entry_points:
       a. Slice indicator time-series up to t-1 bar (by hour index)
       b. Apply Vectorizer to sliced series → feature vector
    Applies to: ma, ema, macd, rsi, atr, bb, adx, dmi, sar, vr_volume,
                obv, ad, vwap, roll_spread, hl_spread, lee_ready, tpm,
                avg_vol_per_tick, rvol (today series), rel_dvol (today series),
                intra_season_{metric} (today series)

Strategy B — ticker당 1회 (monotonic deque):
    fibonacci_retracement: computed in a single O(N) pass over all ticker bars
    via monotonic deque for sliding window max/min.
    Result: time series with one fib level set per bar.
    Sliced per entry point at t-1 (same as Strategy A).

Strategy C — date당 1회 (sr_levels only):
    sr_levels uses scipy prominence — incremental computation not possible.
    Computed once per date using bars up to that date's last entry point t-1.
    All entry points within the same date share the same sr_levels (price_rN,
    prominence_rN). bars_since_rN is recomputed per entry point from
    pivot_hour_rN (cheap: count bars between pivot_hour and t-1).

    if config["indicators"]["sr_levels"].get("exact_per_entry", False):
        → per-entry-point: sr_levels(bars[:t-1_window], ...) per entry point
    else (default):
        → date당 1회 with bars_since recomputation

Strategy D — date당 1회 스칼라 (gap_percentile only):
    gap_percentile returns float — not a time series.
    Computed once per date (same value for all entry points of the same date).
    session_stats dict provides the baseline.
    NaN for t="093000" or pre-market entries.
    Inserted directly into feature vector as "gap_pct" (no Vectorizer step).

Strategy E — session_stats lookup (REFERENCE_SESSION baselines):
    rvol, rel_dvol, intraday_seasonality baselines loaded from
    session_stats dict (pre-loaded from precomputed_session_stats table).
    Baseline is pre-smoothed per delta_minutes at session_stats load time.
```

---

## Interface

```python
class FeatureExtractor:
    def __init__(
        self,
        config: dict,
        db_conn: duckdb.DuckDBPyConnection,
        calculator: IndicatorCalculator | None = None,
    ): ...

    def extract_batch(
        self,
        entry_points: pd.DataFrame,
        bars: pd.DataFrame,
        ticks: pd.DataFrame,
        meta: dict,
        halts_df: pd.DataFrame,
        session_stats: dict | None = None,
    ) -> pd.DataFrame:
        """
        Batch feature extraction for all entry points of a single ticker.
        bars and ticks must be strictly before t bar open (data boundary enforced by caller).
        halts_df passed explicitly for MissingBarClassifier and market_structure_features.
        session_stats: pre-loaded REFERENCE_SESSION baselines from precomputed_session_stats.
            Format: {metric: {hour: smoothed_avg_value}}
            If None, REFERENCE_SESSION indicators return NaN for baseline-dependent values.
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
        session_stats: dict | None = None,
    ) -> dict[str, float] | tuple[dict[str, float], dict]:
        """
        Single entry point feature extraction.
        Used in live inference (Inferencer) and visualization.
        halts_df passed explicitly — consistent with extract_batch() pattern.
        session_stats: same as extract_batch(). In live mode, supplied by LiveModeRunner.
        When return_intermediate=True, returns (feature_vector, intermediate_dict)
        where intermediate_dict contains per-stage outputs for visualization.
        """
        ...

    def get_feature_names(self) -> list[str]:
        """
        Return ordered list of all active feature column names.
        Excludes identifier columns (ticker, date, hour, p_entry) and
        metadata columns (is_dead_position, dead_position_case, is_ambiguous).
        Includes gap_pct if reference_session indicators are enabled.
        """
        ...

    def get_feature_schema(self) -> dict[str, str]:
        """
        Return {feature_name: dtype_str} for all features.
        dtype_str: "continuous" | "categorical"
        Used by Trainer to derive categorical_cols for LightGBM.
        """
        ...

    def calculate_required_history(self) -> dict:
        """
        Return minimum bars required for inference given current config.
        Used by Inferencer for preload validation.

        Returns:
            {
                "min_bars": int,           # lookback_days * 390
                "min_trading_days": int,   # lookback_days
            }

        Simplified: lookback_days * 390 is guaranteed >= max(window_bars)
        by init constraint. n_sessions is independent (baselines from
        precomputed_session_stats, not from bars window).
        """
        return {
            "min_bars": self.config["indicators"]["lookback_days"] * 390,
            "min_trading_days": self.config["indicators"]["lookback_days"],
        }
```

---

## Meta Features

```python
meta_features = {
    "log_market_cap":          float,   # log(market_cap); raw value in stock_meta
    "log_shares_outstanding":  float,   # log(shares_outstanding)
    "log_price_52h":           float,   # log(price_52h)
    "log_price_52l":           float,   # log(price_52l)
    "sector_code":             int,     # encoded via sector_map.json
                                        # unknown sector → 0
                                        # known sectors → 1, 2, 3, ...
                                        # registered as LightGBM categorical
}
```

Monetary fields are log-transformed at feature extraction time.
Raw values are stored in `stock_meta` — not transformed in DB.

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

`entry.hour` is the clock time at which detection fired — not t bar OHLCV data,
and its use as a temporal feature does not constitute data leakage.

Holiday calendar sourced from `us_holidays` table in DuckDB.
`day_of_week` encoding map loaded via `utils.load_encoding_map("day_of_week_map")`.
`is_monday` and `is_friday` binary features must NOT be generated (subsumed by `day_of_week`).

Note on scaling: LightGBM is tree-based and invariant to monotonic transforms.
`minute_of_session` is kept as raw integer.
Scaling will be applied at MLP comparison stage only.

---

## Market Structure Features

```python
market_structure_features = {
    # Halt-related
    "had_halt_today":            int,    # 1 if any halt before t-1 bar else 0
    "bars_since_last_halt":      int,    # bars since last halt ended; NaN if no halt today
    "halt_reason_code":          int,    # encoded via halt_reason_code_map.json
                                         # 0 = no halt today ("no_halt" entry in map)
                                         # registered as LightGBM categorical
                                         # always an integer — never NaN
    "halt_count_today":          int,    # total halts before t-1 bar

    # Missing bar composition
    "missing_bar_count":         int,    # total no_trade gap bars in lookback window

    # Synthetic bar quality indicators
    "synthetic_bar_ratio":       float,  # no_trade_bars / total_bars_in_window [0.0, 1.0]
    "consecutive_synthetic_max": int,    # longest run of consecutive no_trade bars
}
```

Halt data sourced from `halts_df` (passed explicitly — not fetched internally).

**halt_reason_code encoding:**
```
Loaded via utils.load_encoding_map("halt_reason_code_map").
Map file: configs/halt_reason_code_map.json

Reserved entry:
    "no_halt": 0   ← used when had_halt_today == 0
Known NYSE reason codes start from 1 (e.g. "T1": 1, "T6": 2, "LUDP": 3, ...)
Unknown reason codes: map to -1 at runtime (not stored in map file)

When had_halt_today == 0:
    halt_reason_code = 0   ("no_halt")
    bars_since_last_halt = NaN
    halt_count_today = 0
```

**synthetic_bar_ratio computation:**
```
lookback_bars = all bars in the t-N...t-1 window passed to IndicatorCalculator
no_trade_bars = bars classified as no_trade by MissingBarClassifier
                (halt bars excluded — they are NaN, not forward-filled)

synthetic_bar_ratio = len(no_trade_bars) / len(lookback_bars)
```

**consecutive_synthetic_max computation (FeatureExtractor responsibility):**
```
From classify_missing_bars() output:
    classification: dict[HHMMSS → "halt" | "no_trade"]

Steps:
    1. Sort classification keys (HHMMSS strings) in ascending order
    2. Iterate sorted keys; count consecutive "no_trade" runs
    3. consecutive_synthetic_max = max run length observed
    4. If no no_trade bars exist → consecutive_synthetic_max = 0
```

---

## Identifier and Metadata Columns

`p_entry` is stored as an identifier column — not a feature.
It must never appear in `get_feature_names()` output.
It is required by BacktestEngine for fill price calculation.

`is_dead_position`, `dead_position_case`, `is_ambiguous` are stored as
metadata columns — not features. They must not appear in `get_feature_names()`.
They are used by ClassBalancer for pre-balance filtering.

---

## Constraints

- Feature column names must be deterministic and stable across runs
- `extract_batch()` uses Strategy A/B (ticker당 1회) for CONTINUOUS indicators and fibonacci
- `extract_batch()` uses Strategy C (date당 1회) for sr_levels by default
- `extract_batch()` uses Strategy D (date당 1회 scalar) for gap_percentile
- Disabled feature groups (per config) produce no columns
- `sector_code`, `day_of_week`, `halt_reason_code` must be passed as categoricals to LightGBM
- NaN values permitted in continuous features — do not impute
- `p_entry` must never appear as a feature column
- `is_monday` and `is_friday` binary columns must NOT be generated
- Missing bar classification must run before IndicatorCalculator receives bars
- Halt bars (NaN) propagate naturally into indicator NaN — do not suppress
- Temporal features use `entry.hour` (t bar open time) as reference — not data leakage
- Encoding maps (`sector_map`, `day_of_week_map`, `halt_reason_code_map`) loaded via
  `utils.load_encoding_map()` internally — Inferencer does not load or inject maps
- `sector_code` unknown → 0 (not -1); known sectors → 1, 2, 3, ...
- `halt_reason_code` no halt → 0 ("no_halt" in map); known codes → 1, 2, ...; unknown codes → -1
- All "categorical" typed features in `get_feature_schema()` are guaranteed integers, never NaN
- `halts_df` passed explicitly to both `extract_batch()` and `extract()`;
  consistent with Labeler and BacktestEngine patterns (no internal DB queries)
- `consecutive_synthetic_max`: computed by FeatureExtractor from `classify_missing_bars()`
  classification dict, sorted by HHMMSS key ascending
- `sr_distance()` called directly by FeatureExtractor — never via `transform()`
- `synthetic_bar_ratio` and `consecutive_synthetic_max` computed from MissingBarClassifier
  output — no changes to IndicatorCalculator or Vectorizer required
- `is_dead_position`, `dead_position_case`, `is_ambiguous` passed through from Labeler —
  not computed by FeatureExtractor
- `gap_pct` scalar inserted directly into feature_vector — never passed to Vectorizer
- `session_stats` is None during training if precomputed_session_stats not available —
  REFERENCE_SESSION indicators return NaN in that case (not an error)
- `calculator` DI: IndicatorCalculator for training (stateless); CachingIndicatorCalculator
  for live mode (injected by LiveModeRunner)
- Init ConfigurationError if `lookback_days * 390 < max(sr_levels.window_bars, fibonacci.window_bars)`
- `n_sessions` and `lookback_days` are independent — no constraint between them
  (REFERENCE_SESSION baselines come from precomputed_session_stats, not from bars window)
