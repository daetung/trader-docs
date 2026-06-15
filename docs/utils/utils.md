# Module: Utils

**File:** `src/utils/utils.py`

---

## Role

Shared utility functions used across multiple pipeline modules.
Centralizes logic that would otherwise be duplicated, ensuring single-point
maintenance and consistency between training, backtest, and live inference.

---

## Functions

### Bar Sequence

```python
def build_effective_bar_sequence(
    ohlcv_future: pd.DataFrame,
    halts_df: pd.DataFrame,
    date: str,
    t_hour: str,
    target_valid_bars: int = 60,
) -> pd.DataFrame:
    """
    Build sequence of valid (non-halt) bars starting from t_hour.
    Used by Labeler and BacktestEngine for consistent bar counting.

    Collection boundary: bars up to and including 15:59 (session_close_hour) only.
    After-market bars (hour > "155900") are NEVER included regardless of
    target_valid_bars. If 15:59 is reached before target_valid_bars valid bars
    are collected, collection stops at 15:59.

    Halt classification applies across ALL time periods (pre/regular/after-market).
    halts_df is the authoritative source regardless of hour.

    For each missing bar slot in the sequence (up to 15:59):
        if slot overlaps any halt interval in halts_df → classified as "halt"
            halt bars: excluded from valid bar count, OHLC = NaN, volume = 0
        else → classified as "no_trade"
            no_trade bars: included in valid count, OHLC = prior close (forward-fill), volume = 0

    Continues until target_valid_bars valid (non-halt) bars are collected
    or 15:59 bar is reached (whichever comes first).

    Args:
        ohlcv_future:       bars from t_hour onward for (ticker, date)
        halts_df:           trading_halts rows for (ticker, date)
        date:               'YYYYMMDD'
        t_hour:             'HHMMSS' — t bar open time (start of search)
        target_valid_bars:  number of valid bars to collect (default: 60)

    Returns:
        pd.DataFrame with columns [hour, open, high, low, close, volume, is_halt, is_valid]
        Never contains bars with hour > "155900".
    """
    ...
```

---

### Signal Resolution

```python
def resolve_signal(
    row: pd.Series,
    threshold: float,
    suppress_threshold: float | None = None,
) -> str | None:
    """
    Determine entry signal from model probability output.
    up5 takes priority over up3.

    Suppression logic:
        If suppress_threshold is not None, downside classifiers (dn5, dn3)
        are checked first. If either exceeds suppress_threshold, the signal
        is suppressed regardless of upside probabilities.

        This prevents entering long positions when the model has strong
        conviction of a downside move, even if upside probabilities also
        happen to exceed threshold.

    Used by BacktestEngine and Inferencer.

    Setting suppress_threshold higher than entry_threshold results in more
    permissive suppression (fewer entries blocked). Setting them equal is
    the recommended starting point.

    Logic:
        if suppress_threshold is not None:
            if prob_dn5 >= suppress_threshold or prob_dn3 >= suppress_threshold:
                return None   # suppressed — downside conviction overrides upside signal
        if prob_up5 >= threshold:
            return "up5"
        if prob_up3 >= threshold:
            return "up3"
        return None
    """
    ...
```

---

### Tick and Price Utilities

```python
def find_fill_bundle(
    ticks: pd.DataFrame,
    fill_second: str,
) -> tuple[int, pd.Series | None, pd.Series | None]:
    """
    Find the tick bundle containing fill_second.
    Returns (fill_idx, prev_bundle, fill_bundle).

    fill_bundle: first bundle with hour >= fill_second.
    prev_bundle: bundle immediately before fill_bundle (None if fill_bundle is first).
    fill_idx:    iloc index of fill_bundle in ticks DataFrame.

    If no bundle satisfies hour >= fill_second, returns (-1, None, None).
    """
    ...


def interpolate_bundle_price(
    prev_bundle: pd.Series | None,
    curr_bundle: pd.Series,
    target_second: str,
) -> float:
    """
    Estimate the price at target_second within curr_bundle using linear interpolation.

    Anchors:
        t_prev → prev_bundle.close          (last tick of prev bundle)
        t_mid  → typical_price              (high + low + close) / 3 of curr_bundle
        t_curr → curr_bundle.close          (last tick of curr bundle)
        t_mid  = (t_prev + t_curr) / 2      (midpoint assumption for H/L occurrence)

    Formula:
        p_mid  = (curr_bundle.high + curr_bundle.low + curr_bundle.close) / 3
        t_tgt  = hour_to_int(target_second)
        t_prev = hour_to_int(prev_bundle.hour)
        t_curr = hour_to_int(curr_bundle.hour)
        t_mid  = (t_prev + t_curr) / 2

        if t_tgt <= t_mid:
            ratio = (t_tgt - t_prev) / (t_mid - t_prev)
            est   = prev_bundle.close + ratio * (p_mid - prev_bundle.close)
        else:
            ratio = (t_tgt - t_mid) / (t_curr - t_mid)
            est   = p_mid + ratio * (curr_bundle.close - p_mid)

        est = clamp(est, curr_bundle.low, curr_bundle.high)

    Fallback (prev_bundle is None):
        est = curr_bundle.open
        est = clamp(est, curr_bundle.low, curr_bundle.high)

    Fallback (zero-duration bundle, t_prev == t_curr):
        est = curr_bundle.close

    Args:
        prev_bundle:   pd.Series of the bundle before curr_bundle; None if not available
        curr_bundle:   pd.Series of the bundle containing target_second
        target_second: HHMMSS; must satisfy prev_bundle.hour < target_second <= curr_bundle.hour

    Returns:
        Estimated price (float), clamped to [curr_bundle.low, curr_bundle.high]
    """
    ...


def track_price_breach(
    ohlcv_future: pd.DataFrame,
    ticks_future: pd.DataFrame,
    fill_price: float,
    fill_second: str,
    threshold_up: float,
    threshold_dn: float,
    exit_interpolation: bool,
    ambiguity_priority: str = "up",
) -> tuple[str | None, float, str, bool]:
    """
    Detect the first tp/sl threshold breach from fill_second onward.
    Used directly by BacktestEngine (single threshold pair, immediate exit).

    Phase 1 — tick-level scan within t bar:
        Iterate tick bundles from fill_bundle_idx+1 onward within the t bar.
        For each bundle: check if high >= tp_target or low <= sl_target.
        Ambiguity: if both thresholds satisfied within the same bundle,
            use ambiguity_priority to break the tie.
        On breach: interpolate price at breach_second; record direction.

    Phase 2 — bar-level scan (t+1 bar onward):
        For each 1-minute bar: check high/low against thresholds.
        If exit_interpolation=True: use tick data for sub-minute price estimation.
        If exit_interpolation=False: use bar open as breach price (asymmetric).

    Returns:
        (direction, breach_price, breach_hour, is_ambiguous)
        direction: "up" | "dn" | None
        is_ambiguous: True if simultaneous bundle-level breach in Phase 1
    """
    ...


def track_label_breach(
    ohlcv_future: pd.DataFrame,
    ticks_future: pd.DataFrame,
    fill_price: float,
    fill_second: str,
    threshold_3pp: float,
    threshold_5pp: float,
    exit_interpolation: bool,
    ambiguity_priority: str = "up",
) -> tuple[str | None, bool]:
    """
    Two-stage label breach detection.
    Used exclusively by Labeler. Wraps track_price_breach() twice.

    Stage 1 — detect first ±3pp breach:
        track_price_breach(fill_price=P_entry, threshold_up=threshold_3pp,
                           threshold_dn=threshold_3pp, ...)
        → direction_1, exit_price_1, exit_hour_1, is_ambiguous

    If direction_1 is None:
        → Return (None, False). Caller handles session-close / time-limit / dead-position.

    Stage 2 — detect ±5pp extension (from exit_hour_1 onward):
        fill_second_2 = exit_hour_1 (bundles at Stage 1 breach excluded)
        if direction_1 == "up":
            track_price_breach(fill_price=P_entry, threshold_up=threshold_5pp,
                               threshold_dn=threshold_3pp, ...)
            up hit → label_up5; dn hit → label_up3
        if direction_1 == "dn":
            track_price_breach(fill_price=P_entry, threshold_up=threshold_3pp,
                               threshold_dn=threshold_5pp, ...)
            dn hit → label_dn5; up hit → label_dn3

    Args:
        threshold_3pp:      initial breach threshold (e.g. 0.03)
        threshold_5pp:      extension threshold (e.g. 0.05)
        exit_interpolation: passed through to track_price_breach()
        ambiguity_priority: "up" | "dn" — consistent with labeler config

    Returns:
        label_direction: "up5" | "up3" | "dn3" | "dn5" | None
        is_ambiguous:    True if Stage 1 detected simultaneous bundle-level breach

    Constraints:
        - is_ambiguous sourced from Stage 1 only
        - Stage 2 fill_second = exit_hour_1 — bundles at breach point are excluded
        - All thresholds in Stage 2 remain relative to original P_entry, not breach price
        - Does not handle session-close, time-limit, or dead-position exits
    """
    ...


def simulate_exit_fill(
    ticks_exit: pd.DataFrame,
    ohlcv_exit: pd.DataFrame,
    position_size: int,
    breach_bundle_idx: int,
    breach_price: float,
    sell_rate: float,
    halts_df: pd.DataFrame,
) -> tuple[float, int, int, str]:
    """
    Simulate partial exit fills across tick bundles from the breach point onward.
    Continues through session close into after-market until position is fully
    closed or all ticks are exhausted.

    Used exclusively by BacktestEngine after track_price_breach() confirms direction.
    Caller selects sell_rate based on exit direction:
        sell_rate_tp for take-profit; sell_rate_sl for stop-loss.

    Per-bundle fill logic (from breach_bundle_idx onward):
        if bundle overlaps halt interval in halts_df → skip
        per_tick_vol = bundle.volume / 10
        sellable     = floor(per_tick_vol * sell_rate)
        if sellable == 0 → skip
        filled_qty   = min(remaining, sellable)
        fill_price   = interpolate_bundle_price(prev_bundle, bundle, bundle.hour)
        total_value += fill_price * filled_qty
        total_filled += filled_qty

    Breach bundle handling:
        fill_price = breach_price (computed by caller via interpolate_bundle_price).
        sellable   = floor((breach_bundle.volume / 10) * sell_rate)
        if sellable == 0 → skip breach bundle, start from breach_bundle_idx + 1.

    Session close: no forced liquidation; after-market ticks processed identically.
    Ticks exhausted with remaining > 0: unfilled_quantity = remaining.

    weighted_avg_exit_price:
        Σ(fill_price_i * qty_i) / Σ(qty_i) if total_filled > 0 else breach_price.

    Args:
        ticks_exit:        tick_10 from breach_bundle_idx onward (full day, after-market included)
        ohlcv_exit:        1-minute bars from breach bar onward (halt detection)
        position_size:     total shares to exit
        breach_bundle_idx: iloc index of breach bundle in ticks_exit
        breach_price:      estimated price at breach bundle
        sell_rate:         fraction of per-tick volume available (sell_rate_tp or sell_rate_sl)
        halts_df:          trading_halts rows for this ticker/date

    Returns:
        (weighted_avg_exit_price, total_filled, unfilled_quantity, final_exit_hour)
    """
    ...
```

---

### Hour String Utilities

All functions operate on `HHMMSS` format strings (6-digit, zero-padded).

```python
def hour_to_int(hour: str) -> int:
    """
    Convert HHMMSS string to integer.
    Used for trade_log entry_bar / exit_bar storage.
    Example: "093000" → 93000
    """
    return int(hour)


def hour_to_minutes(hour: str) -> int:
    """
    Convert HHMMSS string to total minutes from midnight.
    Used for cooldown calculation in BacktestEngine.
    Example: "093000" → 570  (9*60 + 30)
    """
    return int(hour[:2]) * 60 + int(hour[2:4])


def hour_add_seconds(hour: str, seconds: int) -> str:
    """
    Add seconds to an HHMMSS string. Returns HHMMSS string.
    Used for t+5s target computation in entry slippage.
    Example: hour_add_seconds("093000", 5) → "093005"
    Does not handle day rollover (not needed within trading day).
    """
    h = int(hour[:2])
    m = int(hour[2:4])
    s = int(hour[4:6]) + seconds
    m += s // 60
    s  = s % 60
    h += m // 60
    m  = m % 60
    return f"{h:02d}{m:02d}{s:02d}"


def hhmmss_to_minutes(hour: str) -> int:
    """
    Convert 'HHMMSS' string to minutes since midnight.
    Used by load_session_stats() for delta smoothing window computation.
    Example: '093000' → 570, '155900' → 959.
    Alias for hour_to_minutes() with explicit name for clarity in session context.
    """
    return hour_to_minutes(hour)
```

---

### Run ID Generation

```python
def generate_run_id() -> str:
    """
    Generate a unique run identifier in YYYYMMDD_HHMMSS format.
    Used by run_train.py, run_backtest.py, and PipelineOptimizer (sequential contexts).

    Format: YYYYMMDD_HHMMSS
    Example: "20250715_143022"

    Uniqueness guaranteed by 1-second granularity for sequential callers.

    In nested validation (parallel workers via ProcessPoolExecutor), the coordinator
    pre-generates structured run_ids BEFORE dispatching workers to avoid collision:

        format:  {optimizer_run_id}_o{outer_fold_idx}_t{trial_idx}_f{inner_fold_idx}
        example: "20250715_143022_o1_t4_f2"

    In sequential selection/full phase fold loops, the coordinator similarly
    pre-generates structured run_ids to avoid collision between folds
    (fold processing may complete in under 1 second on small datasets):

        format:  {optimizer_run_id}_f{fold_idx}
        example: "20250715_143022_f2"

    Workers receive run_id as an explicit argument; generate_run_id() is NOT called
    inside worker functions or fold loops. The standard YYYYMMDD_HHMMSS format is
    used only in standalone contexts (standalone CLI, outer eval trainings).
    """
    from datetime import datetime
    return datetime.now().strftime("%Y%m%d_%H%M%S")
```

---

### Config Utilities

```python
def load_config(config_path: str) -> dict:
    """
    Load pipeline_config.yaml and return as dict.
    """
    import yaml
    with open(config_path) as f:
        return yaml.safe_load(f)


def apply_overrides(config: dict, overrides: dict) -> dict:
    """
    Apply key-value overrides to config in-memory (deep copy).
    Config file is never modified.

    Supports dot-notation keys for nested override:
        {"entry_detector.G_min_ratio": 3.0}
        → config["entry_detector"]["G_min_ratio"] = 3.0

    Used by PipelineOptimizer (hyperparameter config override)
    and detection_benchmark.py (--override CLI flag).
    """
    ...
```

---

### Encoding Map Utilities

Encoding maps persist categorical label → integer mappings across training and live inference.
Maps are stored in `configs/` as JSON files.

```python
def load_encoding_map(map_name: str, configs_dir: str = "configs") -> dict:
    """
    Load a categorical encoding map from JSON.
    Returns empty dict if file does not exist.

    Used by FeatureExtractor for consistent encoding.
    Inferencer does not load encoding maps directly —
    maps are loaded internally by FeatureExtractor at inference time.
    """
    ...


def save_encoding_map(map_name: str, mapping: dict, configs_dir: str = "configs") -> None:
    """
    Persist a categorical encoding map to JSON.
    Overwrites existing file.
    Called by FeatureExtractor when building maps for the first time.
    """
    ...
```

**Managed maps:**

| File | Built by | Used by | Notes |
|---|---|---|---|
| `configs/sector_map.json` | FeatureExtractor (first run) | FeatureExtractor | unknown → 0; known → 1, 2, ... |
| `configs/day_of_week_map.json` | FeatureExtractor (first run) | FeatureExtractor | always has value, no NaN |
| `configs/halt_reason_code_map.json` | FeatureExtractor (first run) | FeatureExtractor | "no_halt" → 0; known codes → 1, 2, ...; unknown → -1 |

---

### Trading Calendar Utilities

```python
def populate_trading_calendar(
    db_conn: duckdb.DuckDBPyConnection,
    date_range: list[str] | None = None,
) -> int:
    """
    Populate or refresh trading_calendar table.
    Uses pandas_market_calendars (NYSE) for is_trading_day / is_holiday.
    Sets has_data=True for dates present in ohlcv_1min.
    Safe to re-run (upsert).
    Returns: number of rows upserted.
    """
    ...


def populate_ticker_coverage(
    db_conn: duckdb.DuckDBPyConnection,
    dates: list[str] | None = None,
) -> int:
    """
    Populate or refresh ticker_data_coverage table.
    Groups ohlcv_1min and tick_10 by (ticker, date) to set has_1min, has_tick.
    Safe to re-run (upsert).
    Returns: number of rows upserted.
    """
    ...
```

---

### REFERENCE_SESSION Utilities

```python
def populate_precomputed_session_stats(
    db_conn: duckdb.DuckDBPyConnection,
    dates: list[str],
    n_sessions: int = 20,
) -> int:
    """
    Compute and store REFERENCE_SESSION baselines for given as_of_dates.

    For each as_of_date in dates:
        For each ticker with ohlcv_1min data:
            Compute per-bar average over prior n_sessions regular sessions.

    Metrics derived from ohlcv_1min:
        rvol_baseline      : cumulative volume per HHMMSS slot
        rel_dvol_baseline  : cumulative dollar volume per HHMMSS slot
        intra_vol_baseline : per-bar volume per HHMMSS slot
        intra_return_baseline: per-bar (close/prev_close - 1) per HHMMSS slot
        gap_pct_mean/std   : (today_093000_open - prev_155900_close) / prev_close
                             stored as mean and std (hour='000000')

    Metrics derived from tick_10 (lee_ready):
        buy_ratio_baseline : per-bar buyer_initiated_ratio per HHMMSS slot
        intra_tpm_baseline : per-bar ticks-per-minute per HHMMSS slot

    Stores: avg_value, std_value, count per (ticker, as_of_date, hour, metric, n_sessions)
    Delta smoothing NOT applied here — applied at load time via load_session_stats().
    Safe to re-run (INSERT OR IGNORE).
    Returns: number of rows inserted.
    """
    ...


def load_session_stats(
    db_conn: duckdb.DuckDBPyConnection,
    ticker: str,
    as_of_date: str,
    n_sessions: int,
    delta_minutes: int,
    session_mode: str,
) -> dict:
    """
    Load and smooth precomputed_session_stats for a ticker on a given date.

    Steps:
        1. Query precomputed_session_stats WHERE ticker=?, as_of_date=?, n_sessions=?
        2. Apply session_mode filter:
               "regular":  keep hours 093000~155900 only
               "pre":      keep hours 040000~092900 only
               "combined": keep 040000~155900
        3. Apply delta_minutes rolling average per metric:
               For each hour H, smoothed_value[H] =
                   mean(avg_value for hours in [H-delta, H+delta])
        4. Return dict: {metric: {hour: smoothed_avg_value}}

    Returns empty dict if no rows found (e.g. as_of_date before dataset start).
    """
    ...
```

---

### Optimizer Support Utilities

```python
def compute_vol_regime_holdout(
    db_conn: duckdb.DuckDBPyConnection,
    vol_percentile: float,
    window_days: int = 30,
    vol_metric: str = "avg_intraday_range",
) -> set[str]:
    """
    Compute holdout dates based on rolling volatility percentile.
    Uses regular session bars only (093000–155900) from ohlcv_1min.

    vol_metric = "avg_intraday_range":
        per date = mean of (high - low) / open across all tickers

    Rolling window: window_days calendar days.
    Holdout: dates with rolling_vol >= quantile(vol_percentile).
    Example: vol_percentile=0.80 → top 20% most volatile dates excluded.

    Returns set of 'YYYYMMDD' date strings to exclude from training.
    These dates are removed from labeled_df BEFORE outer fold generation:
        remaining_df = full_labeled_df[~full_labeled_df["date"].isin(holdout_dates)]
    """
    ...


def compute_consensus_config(
    best_configs: list[dict],
    search_space: dict,
) -> dict:
    """
    Aggregate per-outer-fold best hyperparameter configs into a single deployable config.

    Continuous / ordinal parameters (num_leaves, min_data_in_leaf,
    feature_fraction, lambda_l1):
        → median across outer folds → rounded to nearest valid grid value

    Categorical parameters (session_mode):
        → mode across outer folds
        → tiebreak: "combined" > "regular" > "pre" (data coverage priority)

    Args:
        best_configs: list of hyperparameter dicts, one per outer fold
        search_space: valid grid values per parameter (for rounding)

    Returns:
        consensus hyperparameter dict
    """
    ...


def temporal_split_simple(
    labeled_df:   pd.DataFrame,
    session_mode: str | None,
    val_fraction: float = 0.15,
    embargo_days: int = 5,
) -> tuple[pd.DataFrame, pd.DataFrame]:
    """
    Simple date-based train/val split without a test set.
    Used by PipelineOptimizer for outer fold final model training
    (val split for LightGBM early stopping only).

    Order of operations:
        1. Apply session_mode filter (if not None)
        2. Sort by date ascending
        3. Compute split boundary:
             val_dates = last val_fraction of unique dates
             train_dates = remaining (with embargo_days gap to val)
        4. Return (train_df, val_df) without balancing

    No downsampling applied — caller is responsible if needed.
    Returns (train_df, val_df) as plain DataFrames.
    """
    ...
```

---

## Constraints

- All functions are stateless — no module-level state or caching
- `build_effective_bar_sequence()` is the single source of truth for valid bar counting
  — Labeler and BacktestEngine both import from here; logic must not be duplicated
- `build_effective_bar_sequence()` never returns bars with hour > "155900" —
  after-market bars are excluded regardless of target_valid_bars
- `resolve_signal()` is the single source of truth for entry signal thresholding
  — BacktestEngine and Inferencer both import from here
- `resolve_signal()` suppression check always precedes entry signal check
  — suppression takes priority regardless of upside probability magnitude
- `suppress_threshold=None` disables suppression entirely
- `apply_overrides()` must deep-copy config — original must never be mutated
- `load_encoding_map()` returns empty dict (not error) when file does not exist
  — FeatureExtractor handles first-run map creation
- Encoding maps loaded exclusively by FeatureExtractor — Inferencer does not
  load or inject encoding maps directly
- All categorical encoding maps guarantee integer values, never NaN
- `halt_reason_code_map.json` must always contain `"no_halt": 0` as a reserved entry
- `track_price_breach()` is the low-level primitive for single tp/sl detection (BacktestEngine)
- `track_label_breach()` is the high-level wrapper for two-stage label detection (Labeler only)
- `simulate_exit_fill()` is called after track_price_breach() confirms direction;
  sell_rate selection (tp vs sl) is the caller's responsibility
- `generate_run_id()` not called inside parallel workers or sequential fold loops —
  coordinator pre-generates structured run_ids in all optimizer contexts
- `compute_vol_regime_holdout()` uses regular session bars only (093000–155900)
- `compute_consensus_config()` grid-rounding uses nearest valid value from search_space
- `temporal_split_simple()` applies no balancing — training-only utility for early stopping val
- `populate_precomputed_session_stats()` uses INSERT OR IGNORE — safe to re-run
- `load_session_stats()` never modifies DB — read-only
- `load_session_stats()` returns empty dict (not error) when no rows found
- Delta smoothing in `load_session_stats()` is in-memory only — DB rows unchanged
- `buy_ratio_baseline` and `intra_tpm_baseline` require tick_10 data — skipped if unavailable
- `hhmmss_to_minutes()` is an alias for `hour_to_minutes()` with explicit naming for clarity
- No pipeline business logic in this module — utility functions only
