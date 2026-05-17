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

    Halt classification applies across ALL time periods (pre/regular/after-market).
    halts_df is the authoritative source regardless of hour.

    For each missing bar slot in the sequence:
        if slot overlaps any halt interval in halts_df → classified as "halt"
            halt bars: excluded from valid bar count, OHLC = NaN, volume = 0
        else → classified as "no_trade"
            no_trade bars: included in valid count, OHLC = prior close (forward-fill), volume = 0

    Continues until target_valid_bars valid (non-halt) bars are collected
    or no more bars available.

    Args:
        ohlcv_future: bars from t_hour onward for (ticker, date)
        halts_df:     trading_halts rows for (ticker, date)
        date:         'YYYYMMDD'
        t_hour:       'HHMMSS' — t bar open time (start of search)
        target_valid_bars: number of valid bars to collect (default: 60)

    Returns:
        pd.DataFrame with columns [hour, open, high, low, close, volume, is_halt, is_valid]
    """
    ...
```

---

### Signal Resolution

```python
def resolve_signal(row: pd.Series, threshold: float) -> str | None:
    """
    Determine entry signal from model probability output.
    up5 takes priority over up3.

    Used by BacktestEngine and Inferencer.

    Args:
        row:       pd.Series with prob_up5, prob_up3 columns
        threshold: minimum probability to trigger entry

    Returns:
        "up5" | "up3" | None
    """
    if row["prob_up5"] >= threshold:
        return "up5"
    if row["prob_up3"] >= threshold:
        return "up3"
    return None
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

| File | Built by | Used by |
|---|---|---|
| `configs/sector_map.json` | FeatureExtractor (first run) | FeatureExtractor, Inferencer |
| `configs/day_of_week_map.json` | FeatureExtractor (first run) | FeatureExtractor, Inferencer |

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
- `resolve_signal()` is the single source of truth for entry signal thresholding
  — BacktestEngine and Inferencer both import from here
- `apply_overrides()` must deep-copy config — original must never be mutated
- `load_encoding_map()` returns empty dict (not error) when file does not exist
  — FeatureExtractor handles first-run map creation
- No pipeline business logic in this module — utility functions only
