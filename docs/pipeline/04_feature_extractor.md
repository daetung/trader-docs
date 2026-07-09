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

    Bar-derived (adj_bars; anchored to reference_date):
        0. Corporate-event bar adjustment (before indicator computation):
           reference_date = max date present in the bars DataFrame for this call
           adj_bars = utils.adjust_bars_for_corporate_events(bars, ticker,
                          reference_date, db_conn)
           (single vectorized pass against corporate_events, a small table —
            no-op / near-zero cost for the vast majority of tickers with no
            split in the loaded range)
        1. Compute indicators once for the full ticker adj_bars DataFrame
           → dict[str, pd.DataFrame]  (time-series per indicator)
        2. For each entry point in entry_points:
           a. Slice indicator time-series up to t-1 bar (by hour index)
           a2. Scale-type rescale (only when f_e != 1.0 for this entry — see below):
               f_e = cum_split_ratio(entry_date, reference_date)  (1.0 if no split
                     between entry_date and reference_date — the common case)
               if f_e != 1.0: apply per 02_indicator_calculator.md's
                     "Corporate Event Scale Sensitivity" table —
                     scale_type=price   → sliced value *= f_e
                     scale_type=volume  → sliced value /= f_e
                     scale_type=invariant → no change
           b. Apply Vectorizer to sliced (and rescaled, if applicable) series
              → feature vector
        Applies to: ma, ema, macd, rsi, atr, bb, adx, dmi, sar, vr_volume,
                    obv, ad, vwap, roll_spread, hl_spread,
                    rvol (today series), rel_dvol (today series),
                    intra_season_{metric} (today series)

        obv/ad exception: both are unbounded cumulative sums (no session reset),
        so step 0's single-pass adjustment is only valid for tickers/date-ranges
        with no split inside the loaded bars range. When corporate_events shows a
        split within [bars.date.min(), reference_date] for this ticker,
        extract_batch() falls back to per-entry-date recomputation for obv/ad
        specifically: re-run adjust_bars_for_corporate_events() with
        reference_date=entry_date on bars up to that entry, then compute obv/ad
        fresh for that entry (no step 2a2 rescale needed in the fallback path,
        since the bars are already anchored to the entry's own date). All other
        bar-derived Strategy A indicators are self-referential within their own
        transform (statistical_summary, rate_of_change, linear_trend,
        window_comparison, shape_features all operate on the series' own
        internal shape — see 03_vectorizer.md) and do not need this fallback;
        step 2a2 rescale is sufficient for them since none of these five
        transforms compare a raw, unadjusted external reference against the
        indicator series.

    Tick-derived (raw ticks; never adj_bars-anchored — tick_10 is never
    adjusted, in any consumer, so this bar-derived pipeline's premise does
    not apply — see 02_indicator_calculator.md's Tick-Derived Indicator Scale
    Sensitivity registry and data_boundary.md):
        1. Compute per-bar values once for the full ticker's raw ticks
           (same "ticker당 1회" efficiency as bar-derived, but no adj_bars step)
           → dict[str, pd.DataFrame]
        2. For each entry point in entry_points:
           a. Slice to t-1 bar (by hour index)
           a2'. Tick-derived correction dispatch (replaces step 2a2 above for
                these indicators specifically):
               scale_type = registry lookup in 02_indicator_calculator.md's
                   Tick-Derived Indicator Scale Sensitivity table
               sliced = utils.adjust_tick_derived_series_for_corporate_events(
                   sliced, ticker, entry_date, scale_type, db_conn
               )
               (anchored directly to entry_date — no reference_date detour,
               since these series were never reference_date-anchored to begin
               with; invariant-type indicators pass through this call as a
               no-op, so all tick-derived indicators share one dispatch path
               regardless of scale_type)
           a3'. Reindex to the full expected bar index for the sliced window
                (same bar index bars/adj_bars uses for this ticker/date range)
                before applying step a2'. A (ticker, date, hour) with zero
                tick bundles (halt / no-trade slot) has no row in
                tick_bar_aggregates for any indicator (see db_schema.md) —
                reindexing surfaces this as an explicit NaN at that position,
                rather than silently shrinking the window handed to
                Vectorizer's window_comparison/statistical_summary (which
                would understate the true window length and skew the
                resulting mean/std/ratio). Applies identically whether the
                slice was sourced from tick_bar_aggregates directly or via
                the Tier 2 on-the-fly fallback in
                utils.load_tick_bar_aggregates_with_history().
           b. Apply Vectorizer to corrected series → feature vector
        Applies to: current tick-derived indicators registered in
                    02_indicator_calculator.md's Tick-Derived Indicator Scale
                    Sensitivity table (tpm, avg_vol_per_tick, lee_ready's
                    outputs, vol_weighted_buy_ratio, avg_delta_per_tick,
                    tick_realized_vol, path_efficiency, vol_concentration,
                    tick_burstiness) — registering a new tick-derived indicator
                    there is sufficient to route it through this dispatch;
                    no FeatureExtractor code change needed per new indicator.

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
    dividend_amount looked up once per date from corporate_events
    (event_type='dividend', event_date=today; 0.0 if none) and passed to
    IndicatorCalculator.gap_percentile() — corporate_events is queried here,
    not inside IndicatorCalculator (which remains DB-unaware).
    NaN for t="093000" or pre-market entries.
    Inserted directly into feature vector as "gap_pct" (no Vectorizer step).

Strategy E — session_stats lookup (REFERENCE_SESSION baselines):
    rvol, rel_dvol, intraday_seasonality, relative_avg_vol_per_tick baselines
    loaded from session_stats dict (pre-loaded from precomputed_session_stats
    table). Baseline is pre-smoothed per delta_minutes at session_stats load
    time. extract_batch() selects session_stats[entry_date] for each entry
    point, then passes the flat {metric: {hour: value}} dict to
    IndicatorCalculator. relative_avg_vol_per_tick reads
    session_stats["intra_avg_vol_per_tick_baseline"] — sourced from
    tick_bar_aggregates (see db_schema.md, utils.md
    populate_precomputed_session_stats()), not computed from raw ticks at
    this layer.
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
        meta: date-keyed stock_meta resolution for this ticker's entry dates.
            Format: {date_str: {field: value}}
            Keys are entry dates ('YYYYMMDD'); values are the per-field
            resolved dict described in "Meta Features" below (real crawled
            value where available, else utils.estimate_historical_meta()
            fallback per field; sector always the most recent snapshot).
            extract_batch() internally selects meta[entry_date] for each entry
            point before computing meta_features. This mirrors the
            session_stats dispatch pattern below — required because
            stock_meta is now (ticker, date)-keyed (see V-1 fix, db_schema.md):
            a single flat dict shared across all of a ticker's entry points
            would apply one date's values to every other date, reintroducing
            the staleness problem the (ticker, date) schema was meant to fix.
            The caller (run_preprocess.md Step 6 / Inferencer for live) builds
            this dict — FeatureExtractor does not query stock_meta directly.
        session_stats: pre-loaded REFERENCE_SESSION baselines from precomputed_session_stats.
            Format: {date_str: {metric: {hour: smoothed_avg_value}}} | None
            Keys are as_of_date strings ('YYYYMMDD') mapping to single-ticker baseline dicts.
            extract_batch() internally selects stats[entry_date] for each entry point,
            then passes the flat {metric: {hour: value}} dict to IndicatorCalculator.
            If session_stats is None, entry_date is missing from session_stats, or
            session_stats[entry_date] is itself None (ticker absent for that date —
            possible if populate_precomputed_session_stats() has not yet run for
            this specific ticker/date combination), REFERENCE_SESSION indicators
            return NaN. This third case is not currently expected in normal
            operation (see run_preprocess.md constraints) but is handled
            identically to the other two for safety.
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
        meta: flat {field: value} dict for this single entry point's (ticker, date) —
            resolved by the caller the same way as extract_batch()'s per-date
            entries (real value with utils.estimate_historical_meta() fallback
            per field; sector from most recent snapshot). Not date-keyed here,
            since a single entry point has exactly one date already.
        session_stats: flat single-date baseline for this entry point's ticker/date.
            Format: {metric: {hour: smoothed_avg_value}} | None
            In live mode, supplied by LiveModeRunner via CachingIndicatorCalculator._session_stats.
            If None, REFERENCE_SESSION indicators return NaN.
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

    # Fundamentals-derived (yield form — see rationale below), from
    # fundamentals_quarterly (db_schema.md) via utils.get_ttm_value() /
    # utils.estimate_historical_meta()-style as-of lookup. All as-of
    # filed_date <= entry_date (data_boundary.md point-in-time principle).
    # NaN if the underlying fundamentals_quarterly data is unavailable for
    # this ticker/date — LightGBM handles natively, no imputation.
    "earnings_yield":          float,   # utils.get_ttm_value(ticker, date, "net_income", ...)
                                        # / market_cap
    "book_to_price":           float,   # most-recent as-of stockholders_equity
                                        # / market_cap
    "sales_to_price":          float,   # utils.get_ttm_value(ticker, date, "revenue", ...)
                                        # / market_cap
    "dilution_rate":           float,   # QoQ shares_outstanding change rate — see
                                        # "dilution_rate computation" below
    "cash_to_mcap":            float,   # most-recent as-of cash / market_cap
    "debt_to_mcap":            float,   # most-recent as-of total_liabilities / market_cap
}
```

Monetary fields are log-transformed at feature extraction time.
Raw values are stored in `stock_meta` — not transformed in DB.

**Fundamentals yield-form rationale:** this system's universe ($20-and-under
tickers) has a high proportion of unprofitable or negative-book-value
companies. Direct-form ratios (PER = price/earnings, PBR = price/book) are
ill-defined or numerically explosive when the denominator is near zero or
negative. Inverting to price-in-the-denominator form (earnings/price,
book/price) keeps the feature well-behaved and numerically stable across
the full range, including negative values, which remain meaningful signal
(a very negative `earnings_yield` is a real, valid data point — not an
error) rather than an edge case requiring special handling.

**dilution_rate computation:**
```
shares_t   = most recent fundamentals_quarterly shares_outstanding as-of
             entry_date's filing quarter
shares_t_1 = the PRIOR quarter's filed shares_outstanding value
ratio      = cum_split_ratio(ticker, shares_t_1's fiscal_period_end,
                             shares_t's fiscal_period_end, db_conn)
dilution_rate = (shares_t / (shares_t_1 * ratio)) - 1
```
The `cum_split_ratio()` correction is required for the same reason as
Strategy A's `f_e` rescale (see above): a real stock split between the two
quarters would otherwise appear as a spurious multi-hundred-percent
"dilution" (e.g. a 4:1 split reads as shares_t / shares_t_1 ≈ 4.0), when no
actual share issuance occurred. `dilution_rate` is this system's most
directly relevant fundamentals feature — QoQ share-count changes from
registered-direct offerings and ATM programs are a primary driver of this
universe's low-float volatility, and the split correction ensures the
feature isolates genuine issuance/buyback activity from mechanical
share-count changes.

**Source resolution per field (caller responsibility, before meta_features
computation):** `stock_meta` is keyed `(ticker, date)` — a row/field exists
only for dates actually crawled. For each of `market_cap`,
`shares_outstanding`, `price_52h`, `price_52l`, resolved independently per
field (not per row):
```
value = stock_meta.get(ticker, entry_date, field)   # real crawled value, if present
if value is None:
    value = utils.estimate_historical_meta(ticker, entry_date, field, db_conn)
    # derived from ohlcv_1min + corporate_events — see utils.md
```
`sector` has no derivation path — always use the most recent available
`stock_meta` snapshot regardless of entry date (accepted approximation,
unrelated to corporate-event scale).
`EntryPointDetector`'s filter E (condition E, turnover rate) uses this same
`shares_outstanding` resolution via the identical utility — see
`01_entry_detection.md`.

For `extract_batch()`, this per-field resolution is performed once per
(ticker, date) by the caller and packaged into the date-keyed `meta`
parameter described in the Interface section above — not recomputed inside
FeatureExtractor itself.

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
- `extract_batch()` internally dispatches session_stats[entry_date] per entry point before
  passing flat {metric: {hour: value}} to IndicatorCalculator REFERENCE_SESSION methods
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
- Strategy A bars are corporate-event-adjusted (`utils.adjust_bars_for_corporate_events()`,
  anchored to the max date in the loaded bars range) before indicator computation —
  raw, unadjusted bars must never reach IndicatorCalculator via this path; see
  `data_boundary.md` for the full raw-vs-adjusted boundary
- `obv`/`ad` require the per-entry-date fallback described in Strategy A whenever
  a split falls within the loaded bars range for that ticker — this is the only
  Strategy A indicator pair with this exception
- `meta_features` fields (`market_cap`, `shares_outstanding`, `price_52h`,
  `price_52l`) are resolved per-field from `stock_meta(ticker, entry_date)`
  with fallback to `utils.estimate_historical_meta()` — never from a single
  global `stock_meta` snapshot; `sector` is the sole exception (no fallback,
  always most-recent snapshot)
- `gap_percentile()`'s `dividend_amount` is looked up from `corporate_events`
  by FeatureExtractor (Strategy D) and passed as a scalar — IndicatorCalculator
  itself never queries `corporate_events`
- `extract_batch()`'s `meta` parameter is date-keyed (`{date: {field: value}}`),
  not a flat per-ticker dict — required because `stock_meta` is `(ticker, date)`-keyed;
  `extract()`'s `meta` remains flat since it handles exactly one date already
- Fundamentals-derived meta features (`earnings_yield`, `book_to_price`,
  `sales_to_price`, `dilution_rate`, `cash_to_mcap`, `debt_to_mcap`) resolve
  `fundamentals_quarterly` as-of `filed_date <= entry_date`, never
  `fiscal_period_end` — see `data_boundary.md`'s point-in-time principle;
  NaN (not an error) if no eligible filing exists as of `entry_date`
- `dilution_rate`'s two-quarter shares_outstanding comparison must apply
  `cum_split_ratio()` between the two filing dates before computing the QoQ
  ratio — a real split between quarters must not read as spurious dilution
  (same correction principle as Strategy A's `f_e` rescale, applied here to
  a fundamentals feature rather than a bar-derived indicator)
