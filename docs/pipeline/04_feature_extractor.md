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
bars: pd.DataFrame      # ohlcv_1min, strictly bars t-N ... t-1 (before t bar open);
                        # only the bars the broker supplied, a trading day
                        # without a 1-min bar absent from the frame
ticks: pd.DataFrame     # tick_10, strictly ticks before t bar open
meta: dict              # the caller's resolved meta for this ticker and date
                        # (Source resolution per field, below)
entry: dict             # {ticker, date, hour, p_entry}
halts_df: pd.DataFrame  # the halt-interval resolver's intervals (utils.md),
                        # covering every date of the frame
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
        reader=None,
        calculator: IndicatorCalculator | None = None,
    ):
        """
        reader: the caller-supplied reader FeatureExtractor reads these
            through — corporate_events, fundamentals_quarterly, fx_rates:
            Training:  a frame-backed reader over the batch read unit's
                       loaded rows (fx_rates rows dated in [load range
                       start − fx_rates.max_staleness_days, load range end])
            Live mode: a DB-backed reader on the runner's connection
            Optional: without one the extractor serves only feature names,
            feature groups and the schema.
        calculator: the constructor's IndicatorCalculator, which training and
            visualization use; IndicatorCalculator() when None.
            Live mode constructs the FeatureExtractor with the reader only and
            passes the ticker's CachingIndicatorCalculator per call, through
            extract()'s calculator.

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
    │       classify each minute without a bar as halt / no_trade (using
    │       halts_df); fills nothing — no bar is created and no halt row
    │       exists (02_indicator_calculator.md's I5b)
    │       covers all time periods (pre/regular/after-market)
    │       output: bars unchanged + market_structure_features dict
    │
    ├─► IndicatorCalculator (or CachingIndicatorCalculator)
    │       run all enabled indicators per config
    │       output: dict[str → pd.DataFrame]  (time-series per indicator)
    │
    ├─► Vectorizer
    │       for each indicator series, apply mapped transform method(s)
    │       sr_levels: call vectorizer.sr_distance() directly (not via transform())
    │       sr_channels: call vectorizer.sr_channel_distance() directly
    │       gap_pct: inline scalar — NOT passed to Vectorizer
    │       output: dict[str, float]
    │
    ├─► MetaFeatureExtractor  (inline)
    │       log transform monetary fields
    │       encode sector and security_type as integer categories
    │       output: dict[str, float]
    │
    ├─► TemporalFeatureExtractor  (inline)
    │       derive time/date features from entry.hour, entry.date
    │       read the entry date's calendar context (passed in)
    │       output: dict[str, float]
    │
    └─► merge all → single feature_vector
```

---

## extract_batch() Implementation Strategy

`extract_batch()` must not recompute indicators per entry point where avoidable.
Different strategies apply per indicator category:

```
Strategy A — once per ticker (CONTINUOUS indicators):

    Bar-derived (adj_bars; anchored to reference_date):
        0. Corporate-event bar adjustment (before indicator computation):
           reference_date = max date present in the bars DataFrame for this call
           adj_bars = utils.adjust_bars_from_events(bars, ticker,
                          reference_date, events_df)
           (events_df: the reader's corporate_events rows for the ticker)
           (single vectorized pass against corporate_events, a small table —
            no-op / near-zero cost for the vast majority of tickers with no
            split in the loaded range)
        1. Compute indicators once for the full ticker adj_bars DataFrame
           → dict[str, pd.DataFrame]  (time-series per indicator)
        2. For each entry point in entry_points:
           a. Slice indicator time-series up to t-1 bar (by hour index)
           a2. Scale-type rescale (only when f_e != 1.0 for this entry — see below):
               f_e = utils.split_ratio_from_events(ticker, entry_date,
                     reference_date, events_df)  (1.0 if no split
                     between entry_date and reference_date — the common case)
               if f_e != 1.0: apply per 02_indicator_calculator.md's
                     "Corporate Event Scale Sensitivity" table —
                     scale_type=price   → sliced value *= f_e
                     scale_type=volume  → sliced value /= f_e
                     scale_type=invariant → no change
           b. Apply Vectorizer to sliced (and rescaled, if applicable) series
              → feature vector
        Applies to: ma, ema, macd, rsi, atr, bb, adx, dmi, sar, vr_volume,
                    obv, ad, vwap, roll_spread, hl_spread

        obv/ad exception: both are unbounded cumulative sums (no session reset),
        so step 0's single-pass adjustment is only valid for tickers/date-ranges
        with no split inside the loaded bars range. When corporate_events shows a
        split within [bars.date.min(), reference_date] for this ticker,
        extract_batch() falls back to per-entry-date recomputation for obv/ad
        specifically: re-run adjust_bars_from_events() with
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
           (same "once per ticker" efficiency as bar-derived, but no adj_bars step)
           → dict[str, pd.DataFrame]
        2. For each entry point in entry_points:
           a. Slice to t-1 bar (by hour index)
           a2'. Tick-derived correction dispatch (replaces step 2a2 above for
                these indicators specifically):
               scale_type = registry lookup in 02_indicator_calculator.md's
                   Tick-Derived Indicator Scale Sensitivity table
               sliced = utils.adjust_tick_derived_series_from_events(
                   sliced, ticker, entry_date, scale_type, events_df
               )
               (anchored directly to entry_date — no reference_date detour,
               since these series were never reference_date-anchored to begin
               with; invariant-type indicators pass through this call as a
               no-op, so all tick-derived indicators share one dispatch path
               regardless of scale_type)
           a3'. Reindex to the bars' own index for the sliced window
                (the index bars/adj_bars carries for this ticker/date range)
                before applying step a2'. A (ticker, date, hour) without a
                tick bundle has no row in tick_bar_aggregates for any
                indicator (see db_schema.md) — reindexing surfaces a bar
                with no tick bundle as an explicit NaN at that position,
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

Strategy B — once per ticker (monotonic deque):
    fibonacci_retracement: computed in a single O(N) pass over all ticker bars
    via monotonic deque for sliding window max/min.
    Result: time series with one fib level set per bar.
    Sliced per entry point at t-1 (same as Strategy A).

Strategy C — per entry point (sr_levels, sr_channels):
    sr_levels uses scipy prominence over a sliding window, which cannot be
    reduced to an O(1) per-bar update; sr_channels likewise recomputes its
    channels from the pivot list.
    For each entry point, sr_levels is computed from the last
    indicators.sr_levels.window_bars bars ending at t-1 — the definition
    CachingIndicatorCalculator.get_for_entry() uses in live — and
    sr_channels the same way, by IndicatorCalculator.sr_levels() /
    sr_channels() per entry point; no precomputation path.
    They run on the same adj_bars frame as Strategy A, and Strategy A's f_e
    rescale applies to their price-type outputs (02_indicator_calculator.md's
    Corporate Event Scale Sensitivity) before the Vectorizer's distance
    computation — for sr_distance, before the division by P_entry.

Strategy D — once per date, scalar (gap_percentile only):
    gap_percentile returns float — not a time series.
    Computed once per date (same value for all entry points of the same date).
    session_stats dict provides the baseline.
    dividend_amount looked up once per date from the reader's
    corporate_events rows (event_type='dividend', event_date=today; 0.0 if
    none) — the row's gross amount through utils.dividend_gross_from_events(),
    never its raw value, and no withholding applied — and passed to
    IndicatorCalculator.gap_percentile() — corporate_events is read here,
    through the reader, not inside IndicatorCalculator (which remains
    DB-unaware).
    NaN unless the entry date's first regular-session bar's hour is before t
    (02_indicator_calculator.md).
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
    The REFERENCE today series — rvol, rel_dvol, intra_season_{metric},
    relative_avg_vol_per_tick — are each sliced to the REFERENCE_SESSION
    today window (data_boundary.md: the last
    indicators.reference_session.window_bars bars ending at t-1, never
    before the entry date's PREMARKET_OPEN) before the Vectorizer.
    The price_return today series takes prev_close from the previous
    existing bar.
```

---

## Interface

```python
class FeatureExtractor:
    def __init__(
        self,
        config: dict,
        reader=None,
        calculator: IndicatorCalculator | None = None,
    ): ...

    def extract_batch(
        self,
        entry_points: pd.DataFrame,
        bars: pd.DataFrame,
        ticks: pd.DataFrame,
        meta: dict,
        halts_df: pd.DataFrame,
        closes: dict[str, str],
        ends: dict[str, str],
        calendar_context: dict[str, dict],
        session_stats: dict | None = None,
    ) -> pd.DataFrame:
        """
        Batch feature extraction for all entry points of a single ticker.
        bars and ticks are UNBOUNDED per-ticker frames; the caller does not
        enforce the data boundary and cannot — a batch spans several entry
        hours and dates, so no single "before t bar open" predicate exists
        for it. The boundary is enforced INTERNALLY, by the per-entry slice
        to t-1 that Strategies A, B and C specify below.
        halts_df passed explicitly for MissingBarClassifier and market_structure_features.
        ends: the {date: after_hours_end} map run_preprocess.md keeps from
            session_boundaries_from_frame().
        calendar_context: {entry date: context} — utils.calendar_context()'s
            map over the batch's entry dates (Temporal Features, below).
        meta: date-keyed meta resolution for this ticker's entry dates.
            Format: {date_str: {field: value}}
            Keys are entry dates ('YYYYMMDD'); values are the per-field
            resolved dict described in "Meta Features" below (the stock_meta
            prior-day row and utils.estimate_historical_meta() per field;
            sector and security_type from the latest row with the field set;
            price_52h and price_52l filled by the caller).
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
        session_close: str,
        ends: dict[str, str],
        calendar_context: dict,
        return_intermediate: bool = False,
        session_stats: dict | None = None,
        calculator: IndicatorCalculator | None = None,
    ) -> dict[str, float] | tuple[dict[str, float], dict]:
        """
        Single entry point feature extraction.
        bars and ticks must be strictly before t bar open (data boundary
        enforced by caller). One `entry` means one t_hour, so caller
        enforcement is achievable here — unlike extract_batch() above.
        Used in live inference (Inferencer) and visualization.
        halts_df passed explicitly — consistent with extract_batch() pattern.
        ends: the {date: after_hours_end} map, passed by the Inferencer from
            LiveModeRunner; session_close stays flat.
        calendar_context: the entry date's calendar context, flat.
        meta: flat {field: value} dict for this single entry point's (ticker, date) —
            resolved by the caller the same way as extract_batch()'s per-date
            entries (the stock_meta prior-day row and
            utils.estimate_historical_meta() per field; sector and
            security_type from the latest row with the field set; price_52h
            and price_52l filled by the caller). Not date-keyed here, since a
            single entry point has exactly one date already.
        session_stats: flat single-date baseline for this entry point's ticker/date.
            Format: {metric: {hour: smoothed_avg_value}} | None
            In live mode, the Inferencer passes calculator.session_stats, the
            read-only property of the ticker's CachingIndicatorCalculator;
            the REFERENCE today series are CachingIndicatorCalculator's
            session-scoped incremental Layer 2 entries, returned through t-1
            by get_for_entry() and sliced here to the today window.
            If None, REFERENCE_SESSION indicators return NaN.
        calculator: None means the constructor's calculator; live mode passes
            the ticker's CachingIndicatorCalculator (live_mode_runner.md's
            calculators[ticker], through the Inferencer).
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

    def get_feature_groups(self) -> dict[str, list[str]]:
        """
        Map each feature group that produces columns under the active config
        to its feature column names; the union of the column lists is
        get_feature_names(). Every feature column belongs to exactly one
        group, named by the column's producer:
            - an IndicatorCalculator method, named as in
              02_indicator_calculator.md's Interface, its series' Vectorizer
              outputs included:
              - sr_distance's outputs → sr_levels;
              - sr_channels' Vectorizer outputs → sr_channels;
              - level_distance's outputs → fibonacci_retracement and
                pivot_points, each for its own levels;
              - gap_pct → gap_percentile;
              - intra_season_{metric} → intraday_seasonality;
            - a bar column the Vectorizer transforms directly
              (03_vectorizer.md's "price close" and "volume" mapping rows) →
              the column's name, close and volume;
            - this file's feature sections:
              - Meta Features → meta;
              - Temporal Features → temporal;
              - Market Structure Features → market_structure.
        A method whose output enters no feature column names no group.
        Used by pipeline_optimizer.md's optimizer.selection.group_alternatives
        validation.
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
        Return the history an entry needs given current config.
        Used by Inferencer for preload validation (its _prepare_bars() step 3
        checks the trading-day history and the bar count; a failure of either
        logs preload_fail).

        Returns:
            {
                "min_trading_days": int,   # lookback_days — the trading days
                                           # prior_trading_days() names, every
                                           # one of which must have bars
                                           # (utils.has_required_history())
                "min_bars": int,           # the entry-admission bar requirement:
                                           # bars in [the lookback start, t-1]
            }

        min_bars is the largest of:
            - indicators.sr_levels.window_bars;
            - sr_channels' requirement, max(width_range_bars,
              loopback_bars + 2 × pivot_period + 1);
            - indicators.fibonacci.window_bars;
            - H_max (02_indicator_calculator.md's I4).
        n_sessions is independent (baselines from precomputed_session_stats,
        not from bars window).
        """
        ...
```

---

## Meta Features

```python
meta_features = {
    "log_market_cap":          float,   # log(market_cap); raw value in stock_meta
    "log_shares_outstanding":  float,   # log(shares_outstanding)
    "log_price_52h":           float,   # log(price_52h), read from meta (the caller fills it)
    "log_price_52l":           float,   # log(price_52l), read from meta (the caller fills it)
    "sector_code":             int,     # encoded via sector_map.json
                                        # unknown sector → 0
                                        # known sectors → 1, 2, 3, ...
                                        # registered as LightGBM categorical
    "security_type_code":      int,     # encoded via security_type_map.json
                                        # (content fixed, written by
                                        # FeatureExtractor when absent):
                                        # unknown → 0; equity → 1; etf → 2;
                                        # fund → 3; adr → 4; warrant → 5;
                                        # unit → 6; right → 7; preferred → 8;
                                        # debt → 9
                                        # registered as LightGBM categorical;
                                        # sector_code's feature group (meta)

    # Fundamentals-derived (yield form — see rationale below), from the
    # reader's fundamentals_quarterly rows (db_schema.md): TTM values via
    # utils.get_ttm_value(ticker, date, metric, rows), instant values via the
    # as-of most-recent-filed lookup over the same rows, each returning
    # (value, unit). All read the facts filed before entry_date, filed_date <
    # entry_date (data_boundary.md point-in-time principle). get_ttm_value()'s
    # annual fallback: when the quarterly TTM is unavailable, the latest FY
    # value filed before entry_date
    # whose fiscal_period_end lies within
    # fundamentals.annual_fallback_max_age_months of it; else None.
    # A monetary fact meets market_cap in its own currency: each ratio below
    # is the value ÷ (market_cap × utils.fx_per_usd() of the value's
    # currency, from the reader's fx_rates rows), None → NaN.
    # NaN if the underlying fundamentals_quarterly data is unavailable for
    # this ticker/date — LightGBM handles natively, no imputation.
    "earnings_yield":          float,   # utils.get_ttm_value(ticker, date, "net_income", rows)
                                        # / (market_cap × fx_per_usd)
    "book_to_price":           float,   # most-recent as-of stockholders_equity
                                        # / (market_cap × fx_per_usd)
    "sales_to_price":          float,   # utils.get_ttm_value(ticker, date, "revenue", rows)
                                        # / (market_cap × fx_per_usd)
    "dilution_rate":           float,   # QoQ shares_outstanding change rate — see
                                        # "dilution_rate computation" below;
                                        # unchanged; an annual-only filer has none
    "cash_to_mcap":            float,   # most-recent as-of cash / (market_cap × fx_per_usd)
    "debt_to_mcap":            float,   # most-recent as-of total_liabilities
                                        # / (market_cap × fx_per_usd)
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
             entry_date's filing quarter (the reader's rows)
shares_t_1 = the PRIOR quarter's filed shares_outstanding value
ratio      = utils.split_ratio_from_events(ticker, shares_t_1's
                             fiscal_period_end, shares_t's
                             fiscal_period_end, events_df)
             # between the compared rows' instant dates (their
             # fiscal_period_end), not their filing dates
dilution_rate = (shares_t / (shares_t_1 * ratio)) - 1
```
The split-factor correction is required for the same reason as
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
computation)** — data_boundary.md's "Point-in-time correctness for
stock_meta": a `stock_meta` row's date is its crawl date, and a field for the
entry date D is read from the prior-day row, the row dated the trading day
`utils.prior_trading_days(D, 1, closes)` returns. Per field:
```
market_cap         = the prior-day row's value, else
                     utils.estimate_historical_meta(ticker, D, "market_cap",
                     closes, lookback_days, db_conn)
shares_outstanding = utils.estimate_historical_meta(ticker, D, "shares_outstanding",
                     closes, lookback_days, db_conn), its first tier the
                     prior-day row
                     # lookback_days: the caller's config
                     # indicators.lookback_days
price_52h, price_52l — filled by the caller: in training the metric of the
                     batch's loaded precomputed_session_stats group of
                     (ticker, D) at hour '000000', none → NaN; in live by
                     live_mode.price_extremes_source (live_mode_runner.md)
```
`sector` and `security_type` have no derivation path — always the latest
`stock_meta` row with the field set, whatever the entry date (accepted
look-ahead, unrelated to corporate-event scale).
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
    "minute_of_session": int,          # minutes from REGULAR_OPEN to entry.hour
    "day_of_week":       int,          # 0=Mon ... 4=Fri (encoded via day_of_week_map)
    "is_near_open":      int,          # 1 if within first 30 min of regular session
    "is_near_close":     int,          # 1 if entry.hour >= session_close(date)
                                       #   - 30 min; open on the right
    # From the entry date D's calendar context (utils.calendar_context();
    # db_schema.md's trading_calendar):
    "sessions_to_holiday":      int,   # the count of weekdays after D up to
                                       # and including next_closure_last_session;
                                       # 0 when next_closure_last_session is D
    "next_holiday_weekday":     int,   # the weekday of the first weekday after
                                       # next_closure_last_session — categorical,
                                       # encoded via day_of_week_map
    "next_closure_days":        int,   # the calendar days strictly between
                                       # next_closure_last_session and
                                       # next_closure_reopen
    "sessions_since_holiday":   int,   # the count of weekdays from
                                       # prev_closure_reopen up to and excluding D;
                                       # 0 when prev_closure_reopen is D
    "prev_holiday_weekday":     int,   # the weekday of the first weekday after
                                       # prev_closure_last_session — categorical,
                                       # encoded via day_of_week_map
    "prev_closure_days":        int,   # the calendar days strictly between
                                       # prev_closure_last_session and
                                       # prev_closure_reopen
    "next_session_early_close": int,   # 1 when next_session_close is earlier
                                       # than REGULAR_CLOSE, else 0
    # Reserved, not produced and outside get_feature_names():
    "is_early_close":           int,   # 1 when session_close(D) is earlier than
                                       # REGULAR_CLOSE; producing it is a
                                       # feature-set change made when
                                       # early-close sessions take entries
}
```

`entry.hour` is the clock time at which detection fired — not t bar OHLCV data,
and its use as a temporal feature does not constitute data leakage.

`is_near_close` anchors on the EXCHANGE close — `session_close(date)`, not
`last_bar(date)`. 160000 - 30 = 153000 gives 153000..155900, exactly 30 bars;
anchoring on the last bar would give 31. On an early-close day it yields
123000..125900, likewise 30. Held at a fixed 15:30-16:00 the feature reads 0 for
every row of an early-close day, even though that session did have a final
30 minutes — and since live inference runs this same code, both training and
serving are wrong in the same direction rather than skewed against each other.
`is_near_open` is unaffected: the 09:30 open does not move.

THIS SITE CARRIES NO CLOSE-TIME LITERAL. Its former definition was prose, so no
grep on 155900 or 160000 could reach it; it was found on the MEANING axis. Making
the value available and making this site read it are separate changes, and a
literal sweep closes only the second kind.

`session_close` and the calendar context reach this module as their OWN
arguments, alongside `halts_df` and `session_stats` — "passed explicitly, not
fetched internally" is this
module's standing rule and it never queries DuckDB directly. It is not folded
into `meta`: `meta` is date-keyed so the shape would fit, but it carries
stock_meta-sourced per-ticker values and a session close is neither.

The SHAPE differs by entry point, following `meta`'s own precedent below:
`extract_batch()` takes the `{date: session_close}` map and the calendar
context's `{entry date: context}` map because it spans many entry dates for
one ticker, while `extract()` takes that date's close and that date's
context flat because it handles exactly one. Both are needed —
`is_near_close` and the calendar-context features are in the shared feature
dict, so live serving computes them too.

The calendar context is the entry date's `trading_calendar` row, the
published schedule as it stood before D's session (data_boundary.md).
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
    # Halt-related — read from halts_df, the entry date's intervals,
    # inferred ones included
    "had_halt_today":            int,    # 1 if any halt before t-1 bar else 0
    "bars_since_last_halt":      int,    # existing bars since the halt ended;
                                         # NaN if no halt today
    "halt_reason_code":          int,    # encoded via halt_reason_code_map.json
                                         # 0 = no halt today ("no_halt" entry in map)
                                         # registered as LightGBM categorical
                                         # always an integer — never NaN
    "halt_count_today":          int,    # total halts before t-1 bar

    # No-trade minutes — minutes without a bar outside halts_df intervals,
    # counted from timestamps over the CONTINUOUS vectorization window
    "no_trade_minute_count":            int,    # their count
    "no_trade_minute_ratio":            float,  # their share of the window's
                                                # minutes [0.0, 1.0]
    "consecutive_no_trade_minute_max":  int,    # longest run of them
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

**no_trade_minute_ratio computation:**
```
window_minutes   = the minutes of the CONTINUOUS vectorization window
no_trade_minutes = minutes classified no_trade by MissingBarClassifier
                   (minutes without a bar outside halts_df intervals)

no_trade_minute_ratio = len(no_trade_minutes) / len(window_minutes)
```

**consecutive_no_trade_minute_max computation (FeatureExtractor responsibility):**
```
From classify_missing_bars() output:
    classification: dict[HHMMSS → "halt" | "no_trade"]

Steps:
    1. Sort classification keys (HHMMSS strings) in ascending order
    2. Iterate sorted keys; count consecutive "no_trade" runs
    3. consecutive_no_trade_minute_max = max run length observed
    4. If no no_trade minutes exist → consecutive_no_trade_minute_max = 0
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

- `obv`/`ad` requests to the Vectorizer exclude `{name}_window_ratio` and
  `{name}_intercept`. `03_vectorizer.md`'s Constraints name THIS caller as
  where both exclusions are enforced: every remaining output for those two
  indicators is origin-invariant, which is what makes the training and live
  accumulation origins agree
- Feature column names must be deterministic and stable across runs
- `extract_batch()` uses Strategy A/B (once per ticker) for CONTINUOUS indicators and fibonacci
- `extract_batch()` computes sr_levels and sr_channels per entry point (Strategy C)
- `extract_batch()` uses Strategy D (once per date, scalar) for gap_percentile
- `extract_batch()` internally dispatches session_stats[entry_date] per entry point before
  passing flat {metric: {hour: value}} to IndicatorCalculator REFERENCE_SESSION methods
- Disabled feature groups (per config) produce no columns
- `sector_code`, `security_type_code`, `day_of_week`, `next_holiday_weekday`,
  `prev_holiday_weekday`, `halt_reason_code` must be passed as categoricals to LightGBM
- NaN values permitted in continuous features — do not impute
- `p_entry` must never appear as a feature column
- `is_monday` and `is_friday` binary columns must NOT be generated
- Missing bar classification runs before IndicatorCalculator receives bars
  and classifies without filling: no bar is created and no halt row exists;
  halts follow 02_indicator_calculator.md's Incremental Computation Rule:
  - I5b;
  - NaN-term handling;
  - W2
- Temporal features use `entry.hour` (t bar open time) and the entry date's
  calendar context as reference — not data leakage
- Encoding maps (`sector_map`, `security_type_map`, `day_of_week_map`,
  `halt_reason_code_map`) loaded via `utils.load_encoding_map()` internally —
  Inferencer does not load or inject maps
- `sector_code` and `security_type_code` unknown → 0 (not -1); known values → 1, 2, 3, ...
- `halt_reason_code` no halt → 0 ("no_halt" in map); known codes → 1, 2, ...; unknown codes → -1
- All "categorical" typed features in `get_feature_schema()` are guaranteed integers, never NaN
- `halts_df` passed explicitly to both `extract_batch()` and `extract()`;
  consistent with Labeler and BacktestEngine patterns (no internal DB queries)
- `consecutive_no_trade_minute_max`: computed by FeatureExtractor from `classify_missing_bars()`
  classification dict, sorted by HHMMSS key ascending
- `sr_distance()` and `sr_channel_distance()` called directly by FeatureExtractor — never via `transform()`
- `no_trade_minute_ratio` and `consecutive_no_trade_minute_max` computed from MissingBarClassifier
  output — no changes to IndicatorCalculator or Vectorizer required
- `is_dead_position`, `dead_position_case`, `is_ambiguous` passed through from Labeler —
  not computed by FeatureExtractor
- `gap_pct` scalar inserted directly into feature_vector — never passed to Vectorizer
- `session_stats` is None during training if precomputed_session_stats not available —
  REFERENCE_SESSION indicators return NaN in that case (not an error)
- `calculator`: the constructor's IndicatorCalculator for training and visualization
  (stateless); in live mode the ticker's CachingIndicatorCalculator, passed per call to
  `extract()` (live_mode_runner.md's calculators[ticker], through the Inferencer)
- `n_sessions` and `lookback_days` are independent — no constraint between them
  (REFERENCE_SESSION baselines come from precomputed_session_stats, not from bars window)
- Strategy A bars are corporate-event-adjusted (`utils.adjust_bars_from_events()`
  over the reader's corporate_events rows, anchored to the max date in the
  loaded bars range) before indicator computation —
  raw, unadjusted bars must never reach IndicatorCalculator via this path; see
  `data_boundary.md` for the full raw-vs-adjusted boundary
- `obv`/`ad` require the per-entry-date fallback described in Strategy A whenever
  a split falls within the loaded bars range for that ticker — this is the only
  Strategy A indicator pair with this exception
- `meta_features` fields (`market_cap`, `shares_outstanding`) are resolved
  per-field by the caller — the stock_meta prior-day row and
  `utils.estimate_historical_meta()` (data_boundary.md's "Point-in-time
  correctness for stock_meta") — never from a single global `stock_meta`
  snapshot; `price_52h` and `price_52l` are filled by the caller; `sector`
  and `security_type` are the exceptions (no derivation, always the latest
  row with the field set)
- `gap_percentile()`'s `dividend_amount` is looked up from the reader's
  `corporate_events` rows by FeatureExtractor (Strategy D), read through
  `utils.dividend_gross_from_events()` (gross, no withholding), and passed as
  a scalar — IndicatorCalculator itself never reads `corporate_events`
- `extract_batch()`'s `meta` parameter is date-keyed (`{date: {field: value}}`),
  not a flat per-ticker dict — required because `stock_meta` is `(ticker, date)`-keyed;
  `extract()`'s `meta` remains flat since it handles exactly one date already
- Fundamentals-derived meta features (`earnings_yield`, `book_to_price`,
  `sales_to_price`, `dilution_rate`, `cash_to_mcap`, `debt_to_mcap`) resolve
  the reader's `fundamentals_quarterly` rows as-of `filed_date < entry_date`,
  never `fiscal_period_end` — see `data_boundary.md`'s point-in-time
  principle; NaN (not an error) if no eligible filing exists as of
  `entry_date`; `get_ttm_value()`'s annual fallback reads facts filed before
  entry_date
- A monetary fundamentals fact meets `market_cap` in its own currency:
  the ratio is the value ÷ (market_cap × `utils.fx_per_usd()`), the rate
  dated on or before the trading day before the entry date, None → NaN
- `dilution_rate`'s two-quarter shares_outstanding comparison must apply
  `utils.split_ratio_from_events()` between the compared rows' instant dates (their
  `fiscal_period_end`) before computing the QoQ
  ratio — a real split between quarters must not read as spurious dilution
  (same correction principle as Strategy A's `f_e` rescale, applied here to
  a fundamentals feature rather than a bar-derived indicator)
