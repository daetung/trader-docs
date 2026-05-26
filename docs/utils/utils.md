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
        ohlcv_future: bars from t_hour onward for (ticker, date)
        halts_df:     trading_halts rows for (ticker, date)
        date:         'YYYYMMDD'
        t_hour:       'HHMMSS' — t bar open time (start of search)
        target_valid_bars: number of valid bars to collect (default: 60)

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
```

---

### Run ID Generation

```python
def generate_run_id() -> str:
    """
    Generate a unique run identifier in YYYYMMDD_HHMMSS format.
    Used by run_train.py, run_backtest.py, and PipelineOptimizer.

    Format: YYYYMMDD_HHMMSS
    Example: "20250715_143022"

    Uniqueness guaranteed by 1-second granularity.
    PipelineOptimizer trials are separated by >> 1 second per trial,
    so collision probability is negligible.
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

    Used by PipelineOptimizer (feature config override)
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
    """
    ...


def populate_ticker_coverage(
    db_conn: duckdb.DuckDBPyConnection,
    dates: list[str] | None = None,
) -> int:
    """
    Populate or refresh ticker_data_coverage table.
    Derives has_1min from ohlcv_1min, has_tick from tick_10.
    """
    ...
```

---

### Tick Price Tracking Utilities

These functions support fine-grained price tracking using tick_10 data.
Used by BacktestEngine (entry slippage, exit tracking) and Labeler (breach detection).

```
tick_10.hour = last tick timestamp of each 10-tick bundle (second precision)
bundle time range = (prev_bundle.hour, curr_bundle.hour]
```

```python
def find_fill_bundle(
    ticks: pd.DataFrame,
    fill_second: str,
) -> tuple[int, pd.Series | None, pd.Series | None]:
    """
    Identify the fill_bundle (first bundle whose hour >= fill_second)
    and its immediately preceding bundle (prev_bundle).

    fill_second falls within the range (prev_bundle.hour, fill_bundle.hour]
    by definition of tick_10.hour as the last tick timestamp of the bundle.

    Args:
        ticks:       tick_10 DataFrame, sorted by (hour, seq_id)
        fill_second: HHMMSS — target timestamp (e.g. t bar open + 5s)

    Returns:
        fill_idx:    iloc index of fill_bundle in ticks; -1 if not found
        prev_bundle: pd.Series of the bundle immediately before fill_bundle;
                     None if fill_bundle is the first row or not found
        fill_bundle: pd.Series of fill_bundle; None if not found
    """
    ...


def interpolate_bundle_price(
    prev_bundle: pd.Series | None,
    curr_bundle: pd.Series,
    target_second: str,
) -> float:
    """
    Estimate price at target_second using piecewise linear interpolation
    across three anchors, clamped to the bundle's observed price range.

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

    Args:
        ohlcv_future:       1-minute bars from t bar onward (inclusive), sorted by hour
        ticks_future:       tick_10 from t bar onward (inclusive), sorted by (hour, seq_id)
        fill_price:         slippage-adjusted entry price
        fill_second:        HHMMSS — t bar open + 5s (fill execution time)
        threshold_up:       take-profit threshold (e.g. 0.03 or 0.05)
        threshold_dn:       stop-loss threshold (e.g. 0.03)
        exit_interpolation: True = tick-level tracking; False = 1-minute bar only
        ambiguity_priority: "up" | "dn" — direction assigned on simultaneous bundle breach

    Returns:
        direction:   "up" | "dn" | None (neither threshold reached within supplied bars)
        exit_price:  estimated price at breach point
        exit_hour:   HHMMSS of the bar/bundle where breach occurred
        is_ambiguous: True if tp and sl thresholds were simultaneously satisfied
                      within the same 10-tick bundle (exit_interpolation=True)
                      or the same 1-minute bar (exit_interpolation=False)

    ---

    exit_interpolation=False:
        Scan 1-minute bars only.
        take-profit: bars t+1 onward (t bar excluded — fill precedes bar high)
        stop-loss:   bars t onward   (t bar included — immediate adverse move valid)
        exit_price   = target_price (fixed, no interpolation)
        is_ambiguous = True if a single bar's high >= tp_target AND low <= sl_target

    exit_interpolation=True  [default]:

        tp_target = fill_price * (1 + threshold_up)
        sl_target = fill_price * (1 - threshold_dn)

        [Phase 1 — within t bar, tick-level]

        find_fill_bundle() → fill_idx, prev_bundle, fill_bundle
        Iterate bundles from fill_idx+1 onward within t bar (hour < t_bar_close_hour):

            For each bundle b (with prev_b = preceding bundle):
                up_hit = b.high >= tp_target
                dn_hit = b.low  <= sl_target

                if up_hit AND dn_hit (simultaneous):
                    is_ambiguous = True
                    direction    = ambiguity_priority
                    exit_price   = interpolate_bundle_price(prev_b, b, b.hour)
                    exit_hour    = b.hour
                    return

                if up_hit:
                    direction  = "up"
                    exit_price = interpolate_bundle_price(prev_b, b, b.hour)
                    exit_hour  = b.hour
                    return

                if dn_hit:
                    direction  = "dn"
                    exit_price = interpolate_bundle_price(prev_b, b, b.hour)
                    exit_hour  = b.hour
                    return

        No breach within t bar → proceed to Phase 2.

        [Phase 2 — bars after t bar, 1-minute scan with tick refinement]

        For each 1-minute bar i (hour > t_bar_hour), in order:

            up_hit = bar_i.high >= tp_target
            dn_hit = bar_i.low  <= sl_target

            if up_hit AND dn_hit:
                Retrieve tick bundles for bar_i from ticks_future.
                Iterate bundles within bar_i in (hour, seq_id) order:
                    Apply same per-bundle logic as Phase 1.
                    If breach found → return with is_ambiguous as appropriate.
                If no tick data for bar_i:
                    is_ambiguous = True
                    direction    = ambiguity_priority
                    exit_price   = tp_target if direction == "up" else sl_target
                    exit_hour    = bar_i.hour
                    return

            elif up_hit:
                Locate first bundle in bar_i where high >= tp_target.
                exit_price = interpolate_bundle_price(prev_b, breach_b, breach_b.hour)
                exit_hour  = bar_i.hour
                direction  = "up"
                return

            elif dn_hit:
                Locate first bundle in bar_i where low <= sl_target.
                exit_price = interpolate_bundle_price(prev_b, breach_b, breach_b.hour)
                exit_hour  = bar_i.hour
                direction  = "dn"
                return

        return (None, fill_price, fill_second, False)   # no breach in supplied bars

    Constraints:
        - Tracking starts from fill_bundle_idx+1; fill_bundle itself excluded
          (fill_bundle contains the fill execution time — prices before fill_second
          within that bundle are excluded to avoid look-ahead)
        - Session close (15:59) and time-limit exits are caller's responsibility;
          track_price_breach() returns (None, ...) when neither threshold is reached
        - Caller must supply ohlcv_future including t bar as first row
          when exit_interpolation=True
        - ticks_future must be pre-filtered to the relevant ticker/date
    """
    ...


def track_label_breach(
    ohlcv_future: pd.DataFrame,
    ticks_future: pd.DataFrame,
    P_entry: float,
    fill_second: str,
    threshold_3pp: float,
    threshold_5pp: float,
    exit_interpolation: bool,
    ambiguity_priority: str,
) -> tuple[str | None, bool]:
    """
    Two-stage breach detection for label assignment.
    Used exclusively by Labeler. Wraps track_price_breach() twice.

    Stage 1 — detect first ±3pp breach:
        Calls track_price_breach(
            fill_price=P_entry,
            threshold_up=threshold_3pp,
            threshold_dn=threshold_3pp,
            ...
        )
        → direction_1, exit_price_1, exit_hour_1, is_ambiguous

    If direction_1 is None:
        → No ±3pp breach within supplied bars.
        → Return (None, False).
        → Caller proceeds to session-close / time-limit / dead-position path.

    Stage 2 — resolve up3/up5 or dn3/dn5:
        Inputs derived from Stage 1 result:
            ohlcv_future filtered to hour >= exit_hour_1
            ticks_future filtered to hour >= exit_hour_1
            fill_second  = exit_hour_1   (skip bundles at or before breach point)
            fill_price   = P_entry       (thresholds always relative to original P_entry)

        if direction_1 == "up":
            Calls track_price_breach(threshold_up=threshold_5pp, threshold_dn=threshold_3pp)
            direction_2 == "up"  → label = "up5"  (+5pp reached before -3pp cutoff)
            direction_2 != "up"  → label = "up3"  (-3pp cutoff or no further breach)

        if direction_1 == "dn":
            Calls track_price_breach(threshold_up=threshold_3pp, threshold_dn=threshold_5pp)
            direction_2 == "dn"  → label = "dn5"  (-5pp reached before +3pp cutoff)
            direction_2 != "dn"  → label = "dn3"  (+3pp cutoff or no further breach)

    Args:
        ohlcv_future:       1-minute bars from t bar onward (inclusive)
        ticks_future:       tick_10 from t bar onward (inclusive)
        P_entry:            t bar open price (label thresholds always relative to P_entry)
        fill_second:        HHMMSS — t bar open + 5s
        threshold_3pp:      first-breach threshold (e.g. 0.03)
        threshold_5pp:      extension threshold (e.g. 0.05)
        exit_interpolation: passed through to track_price_breach()
        ambiguity_priority: "up" | "dn" — consistent with labeler config

    Returns:
        label_direction: "up5" | "up3" | "dn3" | "dn5" | None
        is_ambiguous:    True if Stage 1 detected simultaneous bundle-level breach;
                         Stage 2 ambiguity does NOT affect this flag

    Constraints:
        - is_ambiguous sourced from Stage 1 only
        - Stage 2 fill_second = exit_hour_1 — bundles at breach point are excluded
        - All thresholds in Stage 2 remain relative to original P_entry, not breach price
        - This function does not handle session-close, time-limit, or dead-position exits;
          those are the caller's responsibility when label_direction is None
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

    Used exclusively by BacktestEngine after track_price_breach() confirms a
    tp/sl direction. Caller selects sell_rate based on exit direction
    (sell_rate_tp for take-profit, sell_rate_sl for stop-loss).

    Per-bundle fill logic (from breach_bundle_idx onward):
        if bundle overlaps halt interval in halts_df → skip
        per_tick_vol = bundle.volume / 10
        sellable     = floor(per_tick_vol * sell_rate)
        if sellable == 0 → skip  (rare edge case; volume/10 * sell_rate < 1)
        filled_qty   = min(remaining, sellable)
        fill_price   = interpolate_bundle_price(prev_bundle, bundle, bundle.hour)
        total_value += fill_price * filled_qty
        total_filled += filled_qty
        remaining   -= filled_qty
        if remaining == 0 → break

    Breach bundle handling:
        First fill opportunity is the breach bundle itself.
        fill_price = breach_price (already computed by caller via interpolate_bundle_price).
        sellable   = floor((breach_bundle.volume / 10) * sell_rate)
        if sellable == 0 → skip breach bundle, start from breach_bundle_idx + 1.

    Session close (155900 bar open):
        No forced liquidation — simulation continues into after-market ticks seamlessly.
        After-market ticks processed with identical per-bundle logic.

    Ticks exhausted with remaining > 0:
        unfilled_quantity = remaining
        Caller treats as dead-position equivalent (applies dead_position_penalty_pct).

    weighted_avg_exit_price:
        if total_filled > 0: Σ(fill_price_i * qty_i) / Σ(qty_i)
        else: breach_price  (extreme edge case — no fills executed)

    Args:
        ticks_exit:        tick_10 from breach_bundle_idx onward, full day
                           including after-market, sorted by (hour, seq_id)
        ohlcv_exit:        1-minute bars from breach bar onward (halt detection)
        position_size:     total shares to exit
        breach_bundle_idx: iloc index of breach bundle in ticks_exit
        breach_price:      estimated price at breach bundle
                           (from interpolate_bundle_price(), computed by caller)
        sell_rate:         fraction of per-tick volume available per tick
                           caller selects: sell_rate_tp or sell_rate_sl per direction
        halts_df:          trading_halts rows for this ticker/date

    Returns:
        weighted_avg_exit_price: float   — volume-weighted average fill price
        total_filled:            int     — shares actually filled
        unfilled_quantity:       int     — remaining shares (0 = fully closed)
        final_exit_hour:         str     — HHMMSS of last partial fill bundle

    Constraints:
        - sell_rate is direction-aware: caller passes sell_rate_tp or sell_rate_sl
          (sell_rate_tp > sell_rate_sl: take-profit exits in rising markets have
           more available buy-side depth than stop-loss exits in falling markets)
        - sellable = 0 treated as skip — expected to be rare since bundle.volume >= 10
          and typical sell_rate values (0.15~0.30) yield sellable >= 1 for most bundles
        - halt bundles skipped: volume unavailable during trading halt
        - session close does not interrupt simulation; after-market ticks processed
        - unfilled_quantity > 0 only when all ticks exhausted before full exit
        - all fill prices computed via interpolate_bundle_price() for consistency
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
- No pipeline business logic in this module — utility functions only
