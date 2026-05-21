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

    Args:
        row:                pd.Series with prob_up5, prob_up3, prob_dn3, prob_dn5 columns
        threshold:          minimum probability to trigger entry signal
        suppress_threshold: minimum dn5 or dn3 probability to suppress entry;
                            None = suppression disabled (default)

    Returns:
        "up5" | "up3" | None

    Examples:
        # Normal entry — no suppression triggered
        prob_up5=0.60, prob_dn5=0.30, threshold=0.5, suppress_threshold=0.5
        → dn5(0.30) < suppress_threshold(0.5) → no suppression → "up5"

        # Suppression triggered by dn3
        prob_up3=0.55, prob_dn3=0.60, threshold=0.5, suppress_threshold=0.5
        → dn3(0.60) >= suppress_threshold(0.5) → suppressed → None

        # Suppression takes priority regardless of upside magnitude
        prob_up5=0.80, prob_dn5=0.70, threshold=0.5, suppress_threshold=0.5
        → dn5(0.70) >= suppress_threshold(0.5) → suppressed → None

        # suppression disabled (original behavior)
        suppress_threshold=None, prob_up5=0.60
        → suppression check skipped → "up5"

        suppress_threshold=None, prob_up3=0.55, prob_up5=0.40
        → suppression check skipped → "up3"

        suppress_threshold=None, prob_up5=0.40, prob_up3=0.40
        → suppression check skipped → neither >= threshold → None
    """
    # Step 1: suppression check (takes priority over entry signal)
    if suppress_threshold is not None:
        if row["prob_dn5"] >= suppress_threshold:
            return None
        if row["prob_dn3"] >= suppress_threshold:
            return None

    # Step 2: entry signal (up5 takes priority over up3)
    if row["prob_up5"] >= threshold:
        return "up5"
    if row["prob_up3"] >= threshold:
        return "up3"
    return None
```

**Config keys:**
```yaml
backtest:
  entry_threshold:    0.5   # minimum probability to enter long
  suppress_threshold: 0.5   # minimum dn probability to suppress entry
                            # null = suppression disabled
```

Both thresholds are independently configurable. Setting `suppress_threshold`
higher than `entry_threshold` results in more permissive suppression (fewer
entries blocked). Setting them equal is the recommended starting point.

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

    Args:
        config_path: path to pipeline_config.yaml

    Returns:
        dict of config values
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

    Args:
        config:    base config dict
        overrides: flat dict of dot-notation key → value

    Returns:
        New config dict with overrides applied (original unchanged)
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

    Args:
        map_name:    filename without extension (e.g. "sector_map", "day_of_week_map")
        configs_dir: directory containing JSON files (default: "configs")

    Returns:
        dict: {category_str: int_code}

    Used by FeatureExtractor and Inferencer for consistent encoding.
    """
    ...


def save_encoding_map(map_name: str, mapping: dict, configs_dir: str = "configs") -> None:
    """
    Persist a categorical encoding map to JSON.
    Overwrites existing file.

    Args:
        map_name:    filename without extension
        mapping:     {category_str: int_code}
        configs_dir: directory to write JSON

    Called by FeatureExtractor when building sector_map or day_of_week_map
    for the first time.
    """
    ...
```

**Managed maps:**

| File | Built by | Used by | Notes |
|---|---|---|---|
| `configs/sector_map.json` | FeatureExtractor (first run) | FeatureExtractor, Inferencer | unknown → 0; known → 1, 2, ... |
| `configs/day_of_week_map.json` | FeatureExtractor (first run) | FeatureExtractor, Inferencer | always has value, no NaN |
| `configs/halt_reason_code_map.json` | FeatureExtractor (first run) | FeatureExtractor, Inferencer | "no_halt" → 0; known codes → 1, 2, ...; unknown → -1 |

**halt_reason_code_map.json reserved entry:**
```json
{
    "no_halt": 0,
    "T1": 1,
    "T6": 2,
    "LUDP": 3,
    ...
}
```
- `"no_halt": 0` is always present — used when `had_halt_today == 0`
- Known NYSE reason codes start from 1
- Unknown reason codes encountered at runtime map to -1 (not stored in file)
- All categorical features are guaranteed integers, never NaN

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
    Cross-references us_holidays table for is_holiday.

    Args:
        db_conn:    DuckDB connection
        date_range: list of 'YYYYMMDD' dates to process;
                    if None, processes all dates in ohlcv_1min

    Returns:
        int: number of rows inserted or updated
    """
    ...


def populate_ticker_coverage(
    db_conn: duckdb.DuckDBPyConnection,
    dates: list[str] | None = None,
) -> int:
    """
    Populate or refresh ticker_data_coverage table.
    Derives has_1min from ohlcv_1min, has_tick from tick_10.

    Args:
        db_conn: DuckDB connection
        dates:   list of 'YYYYMMDD' dates to process;
                 if None, processes all dates in ohlcv_1min

    Returns:
        int: number of rows inserted or updated
    """
    ...
```

Called by:
- `migrate_json_to_duckdb.py`: initial populate after migration
- `collect_daily.py`: incremental update for new dates

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
- `suppress_threshold=None` disables suppression entirely — when None, the function
  skips the suppression block and evaluates upside probabilities only;
  behavior is identical to a version without suppression logic
- `apply_overrides()` must deep-copy config — original must never be mutated
- `load_encoding_map()` returns empty dict (not error) when file does not exist
  — FeatureExtractor handles first-run map creation
- All categorical encoding maps guarantee integer values, never NaN
- `halt_reason_code_map.json` must always contain `"no_halt": 0` as a reserved entry
- No pipeline business logic in this module — utility functions only
