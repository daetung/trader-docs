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
    closes: dict[str, str],
    target_valid_bars: int,
) -> pd.DataFrame:
    """
    Build sequence of valid (non-halt) bars starting from t_hour.
    Used by Labeler and BacktestEngine for consistent bar counting.

    Collection boundary: bars up to and including last_bar(date) only.
    After-market bars (hour > last_bar(date)) are NEVER included regardless of
    target_valid_bars. If last_bar(date) is reached before target_valid_bars valid
    bars are collected, collection stops there.

    PER-DATE, not the fixed 155900 this once assumed. On an early-close day the
    session ends at 13:00, so the boundary is 125900 and there are no bars after
    it. Filling to 155900 there would manufacture ~180 no_trade slots — which
    count as VALID by the rule below — and drive every label toward label_sw
    while reading a normal session as heavily synthetic through
    04_feature_extractor.md's synthetic_bar_ratio / missing_bar_count /
    consecutive_synthetic_max. This boundary is what keeps those three correct;
    the coupling is stated because neither side can be changed alone.
    last_bar(), NOT exit_deadline(): this is a data fact (the final bar of the
    session), not the exit policy. They coincide only while
    execution.session_close_exit_offset_minutes is 1.

    Halt classification applies across ALL time periods (pre/regular/after-market).
    halts_df is the authoritative source regardless of hour.

    For each missing bar slot in the sequence (up to last_bar(date)):
        if slot overlaps any halt interval in halts_df → classified as "halt"
            halt bars: excluded from valid bar count, OHLC = NaN, volume = 0
        else → classified as "no_trade"
            no_trade bars: included in valid count, OHLC = prior close (forward-fill), volume = 0

    Continues until target_valid_bars valid (non-halt) bars are collected
    or last_bar(date) is reached (whichever comes first).

    Args:
        ohlcv_future:       bars from t_hour onward for (ticker, date)
        halts_df:           trading_halts rows for (ticker, date)
        date:               'YYYYMMDD'
        t_hour:             'HHMMSS' — t bar open time (start of search)
        closes:             the {date: session_close} map, for last_bar(date).
                            Without it this function had no way to know the
                            boundary its own docstring states four times above.
        target_valid_bars:  number of valid bars to collect. NO DEFAULT — the
                            caller passes execution.max_hold_bars. A default
                            equal to the configured value is a silent bypass no
                            test detects while the two agree, and here it broke
                            WITHIN one engine: 09_backtest_engine.md checks that
                            key's worth of valid bars elapsed over a sequence
                            this function would have capped at 60, so raising
                            the key past 60 made its time-limit exit
                            unreachable rather than merely late.

    Returns:
        pd.DataFrame with columns [hour, open, high, low, close, volume, is_halt, is_valid]
        Never contains bars with hour > last_bar(date).
    """
    ...
```

---

### Signal Resolution

`resolve_signal()` moved to `docs/utils/execution_common.md` — shared
execution-time logic between BacktestEngine and LiveModeRunner now lives
there rather than growing utils.py indefinitely. See execution_common.md
for the full spec.

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
```

---

### Price Breach Detection

```python
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
        Iterate tick bundles from the entry fill bundle onward, exclusive,
        within the t bar. (Named by position rather than by
        `fill_bundle_idx`: that argument no longer exists on
        simulate_entry_fill(), which now takes a time-valued
        entry_anchor_second and derives the index internally —
        execution_common.md.)
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
```

---

### Exit Fill Simulation

`simulate_exit_fill()` moved to `docs/utils/execution_common.md`, alongside
its new entry-side counterpart `simulate_entry_fill()` — both are backtest
consumers today, but live entirely without label/training involvement
(unlike `track_price_breach()`, which stays here — see execution_common.md's
Role section for the split rationale). See execution_common.md for the full
spec.

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
    Used by load_session_stats() and build_session_stats_dict() for delta smoothing
    window computation.
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

### Session Boundary Derivations

Two columns, and they are not symmetric. `trading_calendar.session_close` holds
the exchange close, and everything else in the system derives from it through
exactly one of the derivations below. `trading_calendar.after_hours_end` holds
the end of the venue's extended session and is consumed verbatim wherever it is
needed: it has NO derivations, and none should be invented here for symmetry.

WHICH BOUNDARIES MOVE: every boundary sourced from `trading_calendar` — the
close, which is at once Regular's upper bound and After-hours' lower bound in
`execution_common.md`'s session-phase table, and `after_hours_end`, which is
that table's upper bound. Between them they touch every row of it. The premarket
open is the exception and stays a literal: the NYSE regular open does not move
except under exchange outage, which the calendar library would not reflect in
any case.

CALL-SITE NOTATION. Signatures below take the boundary map (and, for
`exit_deadline()`, config) explicitly — nothing here queries DuckDB, matching
`halts_df` and `session_stats`. Other specs write the single-argument forms
`last_bar(date)` / `exit_deadline(today)` for readability; the map argument is
ELIDED IN PROSE, never fetched internally. A call site that has no map in scope
is a design error at that site, not a case for a lookup inside these functions. They are named so that no call site can confuse them — that legibility is
the point, and it is why the arithmetic is NOT inlined at each use site.

WHO LOADS. A process loads the maps once at its entry and passes them down;
callers pass no `dates`, since the default loads every row and the corpus runs
to thousands of rows in total. The exception is a function IN THIS FILE that
owns a `db_conn` AND is reached from a tool entry point which preloads nothing
— `populate_precomputed_session_stats()` and `compute_vol_regime_holdout()` —
which loads them at its own start. That is a class, not a per-function
exception, and it does not breach the rule above: those functions load the maps
and then call these accessors WITH the argument, rather than putting a lookup
inside them.

THE AUTHORITY IS WALL CLOCK, not a market-data field. The trade stream carries
a session marker, but a tick-borne signal is unavailable exactly when no tick
arrives — the same ground on which R-3 rejected putting `session_end` on the WS
path. Every boundary here is evaluated against America/New_York wall clock,
`exit_deadline()` and `after_hours_end()` included. In LIVE mode that
evaluation sits under `live_mode.clock_check`'s offset discipline; the training
and backtest paths have no live clock to check and read their dates from stored
data, so the discipline is named here as live-only rather than as a property of
the boundaries. `execution_common.md`'s session-phase table inherits this rather
than restating it.

```python
def session_close(date: str, closes: dict[str, str]) -> str:
    """
    That date's EXCHANGE close, 'HHMMSS' — the trading_calendar column verbatim.
    '160000' ordinarily, '130000' on an early-close day.

    `closes` is the {date: session_close} map — the FIRST of the two that
    load_session_boundaries() returns, the second being {date: after_hours_end}
    for after_hours_end() below.

    RAISES on a date absent from the map. It must NOT fall back to '160000':
    that fallback restores the silent 16:00 assumption this whole mechanism
    exists to remove, and it would fail exactly on the days that differ. Same
    posture as get_next_trading_day() raising on an unpopulated calendar.
    """
    ...


def last_bar(date: str, closes: dict[str, str]) -> str:
    """
    session_close(date) - 1 minute. The final bar of the regular session:
    '155900' ordinarily, '125900' on an early-close day.

    A DATA FACT. It is where bar collection stops, where a prior session's close
    price is read, and the upper bound of the "regular" session_mode range. It
    does not move with policy.

    Callers: build_effective_bar_sequence()'s collection boundary;
    gap_pct's prev_close anchor (this file, 02_indicator_calculator.md,
    migration_tool.md, metadata_crawler.md — every copy of one anchor);
    load_session_stats() / build_session_stats_dict()'s session_mode ranges;
    05_labeler.md's Steps 1/3/4; 01_entry_detection.md's session_mode and
    max_entry_hour; inferencer.md's session-mode filter.

    The "- 1 minute" is a bar-resolution assumption. It is deliberately HERE and
    nowhere else: the column holds the exchange close precisely so that a change
    of bar resolution touches one derivation instead of every site.
    """
    ...


def exit_deadline(date: str, closes: dict[str, str], config: dict) -> str:
    """
    session_close(date) - execution.session_close_exit_offset_minutes.

    A POLICY. Being flat by 15:50 must stay expressible, which is why the offset
    survives as a config key rather than collapsing into the calendar.

    Callers: every former reader of execution.session_close_exit_time — the
    Position Manager Loop's session_end trigger, Session Shutdown condition (a),
    the Watchdog Loop's self-termination guard, BacktestEngine's session-close
    exit.

    NOT INTERCHANGEABLE WITH last_bar(). They coincide only because the offset
    defaults to 1. At an offset of 10, exit_deadline is 15:50 while last_bar is
    still 15:59, and a gap_pct anchor following the exit deadline would be
    silently wrong by nine minutes. One literal 155900 was serving both meanings
    before this split; each former site was classified into exactly one of the
    derivations above, and that classification is what surfaced the conflation.
    """
    ...


def after_hours_end(date: str, ends: dict[str, str]) -> str:
    """
    trading_calendar.after_hours_end for that date, VERBATIM. '200000'
    ordinarily, '170000' on an early-close day, where the venue's extended
    session ends at 17:00.

    `ends` is the {date: after_hours_end} map — the SECOND of the two that
    load_session_boundaries() returns, stated here because session_close()
    states the same for its own map.

    RAISES on a date absent from the map. It must NOT fall back to '200000':
    that fallback restores the silent 20:00 assumption this mechanism exists to
    remove, and it is wrong on exactly the days that matter. Identical posture
    to session_close()'s refusal to fall back to '160000'. Nothing carried this
    rule before — the column had no accessor to host it.

    Callers: execution_common.md's session-phase table, for After-hours' upper
    bound; live_mode_runner.md's process hard cap, which subtracts
    live_mode.session_hard_exit_offset_minutes from this value rather than
    equalling it; 05_labeler.md's after-market boundary.

    NO DERIVATIONS, deliberately. session_close earned three names because three
    different quantities come off it; this column is consumed as-is at every
    site. Stated so that a later session does not build a sibling family here
    for symmetry's sake.
    """
    ...


def load_session_boundaries(
    db_conn: duckdb.DuckDBPyConnection,
    dates: list[str] | None = None,
) -> tuple[dict[str, str], dict[str, str]]:
    """
    Load the canonical {date: session_close} and {date: after_hours_end} maps —
    two flat maps, both columns in ONE query. Called ONCE PER PROCESS at entry,
    not alongside session_stats_bulk: the earliest consumer is
    live_mode_runner.md's early-close participation gate, which is evaluated
    before the Session Lifecycle that loads those stats.

    Normalisation is NOT performed here. This function SELECTs and delegates to
    session_boundaries_from_frame() below, so the two entry points cannot
    diverge on which rows enter the maps.

    NOT folded into the session_stats dict. That dict's grain is
    (date, ticker, metric, hour) while a close is one per DATE, so folding would
    replicate one value across ~15,000 tickers per day — 15,000 declaration sites
    of a single fact — and would make the close inherit session_stats' "absent is
    fine, degrade to NaN" contract, which is the opposite of the raise rule above.
    Labeler and BacktestEngine receive no session_stats at all yet need the close
    most, so folding would also leave a second delivery path in place for the same
    fact.

    Rows number in the thousands even across the whole corpus. Vectorised
    consumers attach it to their own frame — bars["session_close"] =
    bars["date"].map(closes) — which is a derivation of this map, not a second
    source. dates=None loads every row, which is what every caller passes:
    narrowing buys nothing at this size and a caller cannot enumerate the prior
    sessions it will ask about without a calendar it does not yet hold.
    """
    ...


def session_boundaries_from_frame(
    calendar_df: pd.DataFrame,
) -> tuple[dict[str, str], dict[str, str]]:
    """
    The same two maps, built from an already-loaded trading_calendar frame, for
    consumers that hold the table already — run_preprocess.md SELECTs it at
    Step 1 and passes it on to 05_labeler.md — so that neither issues a second
    query for a fact it has in hand.

    SOLE OWNER OF THE NORMALISATION, which is why the name does not near-miss
    load_session_boundaries(): that one delegates here.

    Rows whose session_close is NULL are EXCLUDED, so a non-trading day raises
    as absent rather than returning a null and pushing a None branch into every
    accessor. Membership is is_trading_day, NEVER has_data: a forward-filled row
    carries has_data=FALSE and today's row has no bars at session start, so a
    has_data filter — the condition 09_backtest_engine.md's next-trading-day
    lookup uses, and the one an implementer is likeliest to copy — would break
    the early-close gate on its first call.
    """
    ...
```

---

### Trading Calendar Utilities

```python
def populate_trading_calendar(
    db_conn: duckdb.DuckDBPyConnection,
    date_range: list[str] | None = None,
) -> int:
    """
    Populate or refresh trading_calendar table.
    Uses pandas_market_calendars (NYSE) for is_trading_day / is_holiday, and its
    SCHEDULE interface — not merely its holiday set — for session_close: the
    per-date market_close, stored as 'HHMMSS' wall clock ('160000' ordinarily,
    '130000' on an early-close day). A holiday set cannot express a half day,
    which is why reading only holidays left the whole system assuming 16:00.
    after_hours_end is NOT available from that library — schedule() returns
    market_open/market_close only — so it is written from the confirmed venue
    fact: '200000' ordinarily, '170000' on an early-close day. See db_schema.md's
    column-class membership rule on trading_calendar.
    Sets has_data=True for dates present in ohlcv_1min.
    Safe to re-run (upsert).

    FORWARD FILL — ONE YEAR AHEAD, ROLLING.
    date_range=None populates from the earliest date already present through one
    year past today, and the daily run extends that horizon by a day each time it
    runs rather than refreshing it in an annual batch, so the horizon never
    reaches an expiry cliff. us_holidays already carries the correct convention
    ("3 years forward on first run; refresh annually" — metadata_crawler.md); this
    table did not, and get_next_trading_day()'s "calendar should be populated far
    enough ahead by populate_trading_calendar()" was an ASSUMPTION stated in a
    parenthetical that no call site implemented — migration passes ingested_dates
    (past) and the daily run passes [today].
    What that closes: on reaching the last populated date the daily collection
    raises at get_next_trading_day(), tomorrow's precomputed_session_stats is
    never written, and Health Gate 1a aborts tomorrow's session — the ending
    health_report.md finding 28 describes. The raise rule on a missing
    session_close (below) makes the dependency heavier still.
    A forward row carries is_trading_day, is_holiday, session_close and
    after_hours_end, and has_data=FALSE. That FALSE is correct, not a gap.

    THE DAILY RUN REWRITES A NEAR-TERM WINDOW, it does not only append. An
    unscheduled early close — weather, national mourning — is announced days
    ahead and must overwrite a session_close written a year earlier. Append-only
    extension would leave the stale '160000' in place for exactly the case that
    cannot be predicted a year out.

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


def get_next_trading_day(
    date: str,
    db_conn: duckdb.DuckDBPyConnection,
) -> str:
    """
    Return the next date after `date` with is_trading_day=TRUE in trading_calendar.

    Used by metadata_crawler.md's daily collection to determine the as_of_date
    for populate_precomputed_session_stats(), and by migration_tool.md's
    post-migration first-live-session setup.

    Query:
        SELECT date FROM trading_calendar
        WHERE date > ? AND is_trading_day = TRUE
        ORDER BY date LIMIT 1

    Note: is_trading_day (NYSE calendar), not has_data — this looks forward to a
    future date that has not been ingested yet, so has_data is not yet TRUE for
    it. Distinct from the has_data=TRUE lookup used by Labeler/BacktestEngine
    for dead position resolution (which looks for a date that already has
    ingested data).

    Returns: 'YYYYMMDD' string. Raises if no such date exists in trading_calendar
    (calendar should be populated far enough ahead by populate_trading_calendar()).
    """
    ...
```

---

### Ticker Identity Utilities

```python
def get_ticker_history(
    ticker: str,
    db_conn: duckdb.DuckDBPyConnection,
) -> list[tuple[str, str]] | None:
    """
    Return the predecessor-symbol chain for a ticker due to plain renames
    (mergers/spin-offs excluded — those are distinct securities, not
    continuations, and are never registered in ticker_history).

    Returns None if no history exists — the common case for the vast majority
    of tickers. Callers must check for None first and take the fast path
    (no further lookup) rather than assuming an empty list.

    If history exists, returns [(previous_ticker, effective_date), ...] sorted
    oldest-first, e.g. a ticker renamed twice (A→B→C) returns
    [("A", "20220101"), ("B", "20230601")] when queried as "C".

    Query:
        SELECT previous_ticker, effective_date FROM ticker_history
        WHERE current_ticker = ? AND rename_type = 'rename'
        ORDER BY effective_date ASC
        -- recursively resolve if previous_ticker itself has an earlier entry
        -- as current_ticker (chain support)

    ticker_history is populated by an automated premarket-detection +
    evening-self-correction mechanism (see metadata_crawler.md), with manual
    CLI entry retained as a fallback for SEC-unmatched tickers — rows are no
    longer manual-SQL-only.
    """
    ...


def load_ohlcv_with_history(
    ticker: str,
    start_date: str,
    end_date: str,
    db_conn: duckdb.DuckDBPyConnection,
) -> pd.DataFrame:
    """
    Load 1-minute bars for a ticker over [start_date, end_date], including bars
    stored under predecessor symbols (see get_ticker_history()).

    Internally calls get_ticker_history(ticker, db_conn) first; if None
    (common case), issues a single direct query — no additional overhead
    beyond the small ticker_history lookup:
        SELECT * FROM ohlcv_1min WHERE ticker=? AND date>=? AND date<=?

    If history exists, issues one additional query per predecessor covering
    only the date range before that predecessor's effective_date, relabels
    the `ticker` column to the current ticker, and UNIONs with the current
    ticker's own rows. Returned DataFrame is sorted by (date, hour) and its
    `ticker` column is normalized to the current ticker throughout.

    Used by live_mode_runner.md's Phase 1 historical bar load, where bars are
    fetched per-ticker via a direct DB query (unlike run_preprocess.md, which
    bulk-loads the entire ohlcv_1min table into memory and performs the
    equivalent predecessor-row relabeling in-memory instead of via a second
    query — see run_preprocess.md Step 6 for that variant).
    """
    ...
```

---

### Corporate Event Utilities

```python
def cum_split_ratio(
    ticker: str,
    from_date: str,
    to_date: str,
    db_conn: duckdb.DuckDBPyConnection,
) -> float:
    """
    Cumulative product of split/reverse_split 'value' in corporate_events for
    this ticker with event_date IN (from_date, to_date]. Returns 1.0 if none
    (the common case — no query cost beyond a small corporate_events lookup,
    since the table itself is small regardless of match).

    Shared primitive used by:
        - adjust_bars_for_corporate_events() (below) — bar-level correction
        - 04_feature_extractor.md Strategy A per-entry rescale (f_e)
        - 05_labeler.md / 09_backtest_engine.md dead position Case A/D
          (there, from_date/to_date are the single-day overnight window D→D+1,
          and dividend_amount is looked up separately by the caller — this
          function returns the split ratio component only)

    Query:
        SELECT value FROM corporate_events
        WHERE ticker = ? AND event_type IN ('split', 'reverse_split')
          AND event_date > ? AND event_date <= ?
        -- product of all matching rows; 1.0 if no rows
    """
    ...


def adjust_bars_for_corporate_events(
    bars: pd.DataFrame,
    ticker: str,
    reference_date: str,
    db_conn: duckdb.DuckDBPyConnection,
) -> pd.DataFrame:
    """
    Adjust OHLCV bars to reference_date's split scale using cum_split_ratio().

    For each bar with date=D_bar < reference_date:
        ratio = cum_split_ratio(ticker, D_bar, reference_date, db_conn)
        open, high, low, close /= ratio
        volume *= ratio
    Bars with date >= reference_date, or no applicable splits (ratio=1.0):
        no-op — returned unchanged.

    Vectorized via a single merge against corporate_events (a small table) —
    one pass over `bars`, not a per-row loop. For the vast majority of
    tickers (no split in the loaded range), this reduces to a no-op copy.

    Does NOT adjust for dividends — dividends affect gap_pct's overnight
    prev_close comparison and dead-position pnl (both handled as separate
    scalar corrections by their respective callers — see
    02_indicator_calculator.md gap_percentile() and 05_labeler.md /
    09_backtest_engine.md dead position Case A/D) but do not create a
    continuous price-level discontinuity the way splits do, so they are out
    of scope for this bar-level correction.

    Called by:
        - 04_feature_extractor.md extract_batch() Strategy A, reference_date =
          the max date in the loaded bars range for that call (today for live
          mode's session_start_compute(), the last date in the ticker's
          loaded range for training)
        - inferencer/caching_calculator.md session_start_compute(),
          reference_date = today

    Returns a new DataFrame — does not mutate the input `bars` in place.
    """
    ...


def adjust_tick_derived_series_for_corporate_events(
    series: pd.DataFrame,
    ticker: str,
    anchor_date: str,
    scale_type: str,
    db_conn: duckdb.DuckDBPyConnection,
) -> pd.DataFrame:
    """
    Correct a tick-derived indicator's raw output series to anchor_date's
    native scale, directly — no reference_date detour.

    Symmetric in spirit to adjust_bars_for_corporate_events(), but for a
    different situation: tick_10 (the real input behind tick-derived
    indicators) is never itself adjusted (see data_boundary.md), so a
    tick-derived indicator's raw per-bar output is always already expressed
    in that bar's own native date's scale — it was never moved to
    reference_date's basis the way adj_bars is. Correcting it therefore goes
    straight to whatever date the caller actually needs (anchor_date —
    normally entry_date), not through reference_date first.

    For each row with date=D_row in `series`:
        ratio = cum_split_ratio(ticker, D_row, anchor_date, db_conn)
        scale_type == "price"     → value /= ratio
        scale_type == "volume"    → value *= ratio
        scale_type == "invariant" → no change (present for API symmetry, so
                                     ALL tick-derived indicators can be routed
                                     through this same function regardless of
                                     scale_type — see
                                     02_indicator_calculator.md's
                                     Tick-Derived Indicator Scale Sensitivity
                                     registry for which indicator uses which)

    Optimization: checks cum_split_ratio(ticker, series.date.min(), anchor_date,
    db_conn) once first; if 1.0 (the common case — no split in range), skips
    the per-row loop entirely and returns the input unchanged. Same cost
    profile as adjust_bars_for_corporate_events().

    Does not mutate the input `series` — always returns a new copy. Never
    touches tick_10 — operates only on an already-computed indicator output
    (whether sourced from tick_bar_aggregates or the Tier-2 on-the-fly
    fallback in load_tick_bar_aggregates_with_history()).

    Called by:
        - 04_feature_extractor.md extract_batch()'s tick-derived dispatch
          path, anchor_date = entry_date (per entry point)
    """
    ...
```

---

### Tick Bar Aggregate Utilities

```python
def compute_tick_bar_aggregates(
    ticker: str,
    start_date: str,
    end_date: str,
    indicators: list[str],
    db_conn: duckdb.DuckDBPyConnection,
) -> pd.DataFrame:
    """
    Compute bar-level tick-derived indicator values for a ticker over a date
    range, directly from raw tick_10 + ohlcv_1min.

    Thin wrapper only — internally calls IndicatorCalculator's own
    avg_vol_per_tick() / tpm() / lee_ready() / etc. (whichever `indicators`
    lists) rather than reimplementing their formulas, so a value computed
    here can never diverge from a value computed the same way elsewhere
    (Strategy A during training, or the Tier-2 fallback below).

    Output: long/EAV format matching tick_bar_aggregates — columns
    [ticker, date, hour, indicator, value]. Rows with fewer than the
    indicator's required tick-bundle count (e.g. <2 bundles for a
    delta/interval-based indicator) get value=NULL rather than being omitted,
    so downstream reindexing against the full bar index still finds a row
    to report NaN from.

    Called by:
        - collect_daily.py (offline; result INSERT OR IGNORE'd into
          tick_bar_aggregates — see metadata_crawler.md)
        - load_tick_bar_aggregates_with_history()'s Tier 2 fallback (below;
          result used in-memory only for that session, never written back to
          tick_bar_aggregates — writing is collect_daily.py's job alone)
    """
    ...


def load_tick_bar_aggregates_with_history(
    ticker: str,
    indicators: list[str],
    lookback_start_date: str,
    today_date: str,
    db_conn: duckdb.DuckDBPyConnection,
) -> pd.DataFrame:
    """
    Load historical tick-derived indicator values for live mode's Eager Pool,
    for indicators configured with precalculate_bars: "lookback"
    (02_indicator_calculator.md). Returns wide format (one column per
    indicator) for direct use seeding CachingIndicatorCalculator's Layer 2.

    3-tier resolution per (ticker, date) in [lookback_start_date, today_date):
        Tier 1 — tick_bar_aggregates has rows for this date: read directly.
        Tier 2 — no cached rows, but ticker_data_coverage.has_tick is True for
            this date (collect_daily.py's batch simply hasn't reached it yet,
            or ticker is newly listed): call compute_tick_bar_aggregates() for
            this date on the fly. Result used for this session only — not
            written to tick_bar_aggregates (collect_daily.py owns writes).
        Tier 3 — has_tick is False for this date (genuine data absence):
            NaN for this date. Only this tier is a true missing-data gap in
            the sense precomputed_session_stats also uses; Tier 2 is an
            expected, cheap, self-healing case in normal operation.

    Tier 2 is expected to be rare in steady-state operation (daily batch
    keeps tick_bar_aggregates current) — when it does happen, only that one
    ticker's Eager Pool worker is slowed, not session start as a whole
    (parallel workers are independent).

    Called by:
        - live_mode_runner.md's Eager Pool, in the same per-ticker parallel
          worker call that also loads historical_bars via
          load_ohlcv_with_history() — both must be available before that
          worker's session_start_compute() call, not staggered across
          separate rounds.
    """
    ...
```

```python
def estimate_historical_meta(
    ticker: str,
    date: str,
    field: str,
    db_conn: duckdb.DuckDBPyConnection,
) -> float | int | None:
    """
    Derive a point-in-time stock_meta field value for (ticker, date) from
    ohlcv_1min + corporate_events, for use when stock_meta has no real crawled
    row/value for that specific date (see db_schema.md stock_meta — schema is
    (ticker, date)-keyed; rows exist only for dates actually crawled).

    Supported fields and derivation:
        shares_outstanding:
            1. Real crawled stock_meta(ticker, date) value, if present (unchanged).
            2. fundamentals_quarterly tier: most recent row with
               filed_date <= date (see data_boundary.md — never
               fiscal_period_end) and metric="shares_outstanding", then
               adjusted for any split between that filing's fiscal_period_end
               and date via cum_split_ratio() (XBRL values are as of their
               own filing date, so subsequent splits must still be applied).
               Uses get_ttm_value()-adjacent as-of lookup logic (see
               Fundamentals Utilities below), not a TTM sum itself since
               shares_outstanding is an instant, not a duration, metric.
            3. Fallback (unchanged): current_value = most recent
               stock_meta.shares_outstanding for ticker;
               ratio = cum_split_ratio(ticker, date, most_recent_date, db_conn);
               return current_value / ratio
               (buyback/issuance activity between quarters is NOT captured
               this way — only split-driven share-count changes are
               reversible from ohlcv_1min + corporate_events alone; tier 2
               closes this gap for tickers/dates fundamentals_quarterly
               covers, tier 3 remains the safety net for the rest)

        market_cap:
            close = close price for (ticker, date) from ohlcv_1min
                    (last regular-session bar of that date)
            shares = estimate_historical_meta(ticker, date, "shares_outstanding", db_conn)
            return close * shares

        price_52h / price_52l:
            bars = adjust_bars_for_corporate_events(
                       ohlcv_1min rows for ticker over trailing 252 trading days
                       ending at date, ticker, reference_date=date, db_conn)
            return MAX(bars.high) / MIN(bars.low) respectively
            (split-adjusted so the 252-day window isn't corrupted by a split
            crossing it — reuses the same bar-adjustment utility as Strategy A)

        sector:
            Not supported — no derivation path. Callers must use the most
            recent available stock_meta snapshot for this field regardless of
            date; do not call this function with field="sector".

    Called by:
        - 04_feature_extractor.md Meta Features resolution (training, per
          entry date) and by run_preprocess.md Step 6 (building the date-keyed
          `ticker_meta` dict)
        - 01_entry_detection.md filter E (condition E, turnover rate) — same
          shares_outstanding resolution, shared rather than duplicated
        - live_mode_runner.md, only as a fallback if the current day's
          stock_meta crawl is incomplete for a field (rare — see
          metadata_crawler.md dual-schedule design)

    Returns None if derivation itself is impossible (e.g. no ohlcv_1min data
    for the ticker before `date` at all) — caller treats this the same as a
    missing stock_meta value (NaN feature, or exclusion from filter E as
    appropriate to that caller's own missing-data handling).
    """
    ...
```

---

### Fundamentals Utilities

```python
def resolve_xbrl_tag_value(
    companyfacts_json: dict,
    tag_priority_list: list[str],
) -> tuple[str, float] | None:
    """
    Given a company's raw SEC EDGAR companyfacts JSON and a metric's ordered
    tag-name priority list (e.g. configs/xbrl_tag_map.json's list for
    "revenue"), return the (tag_name, value) from the first tag in the
    priority list that has any reported facts, or None if none of the tags
    are present at all for this company.

    This resolves cross-filer tag fragmentation (e.g. revenue reported under
    RevenueFromContractWithCustomerExcludingAssessedTax by one filer,
    Revenues by another) — configs/xbrl_tag_map.json is the source of each
    metric's priority list, keyed by metric name, extendable by editing that
    JSON alone (no code change to add a new fallback tag for an existing
    metric).

    Does not itself apply any as-of filtering — returns whatever facts exist
    for the resolved tag; callers (estimate_historical_meta(),
    get_ttm_value()) apply the filed_date <= date filter themselves.

    Multiple units/contexts per tag (e.g. consolidated vs. segment,
    original vs. restated filing) are a known parsing subtlety not fully
    resolved by tag-priority alone — flagged here as an implementation-time
    concern, not a blocker for this function's design.
    """
    ...


def get_ttm_value(
    ticker: str,
    date: str,
    metric: str,
    db_conn: duckdb.DuckDBPyConnection,
) -> float | None:
    """
    Trailing-twelve-month value for a duration-type fundamentals_quarterly
    metric (e.g. "net_income", "revenue"), as of `date`.

    As-of filtering: only rows with filed_date <= date are eligible — see
    data_boundary.md's point-in-time principle (never fiscal_period_end).

    Resolution:
        Sum the four most recent eligible quarters' values for this metric
        (period_months=3 rows), ending at or before `date`.
        Q4 does not exist as a filed row on its own — where the most recent
        eligible quarter would be Q4, derive it as
        FY_value - (Q1 + Q2 + Q3) using the matching period_months=12 row
        and the three preceding period_months=3 rows for the same fiscal year.

    Returns None if fewer than 4 eligible quarters (or the FY+3Q components
    for a Q4 derivation) are available as of `date` — caller treats this the
    same as any other unavailable fundamentals value (NaN feature).

    Not applicable to instant-type metrics (shares_outstanding,
    stockholders_equity, total_assets, etc. — period_months=NULL) — those are
    point-in-time by nature and use a plain as-of most-recent-filed lookup
    instead, not a TTM sum (see estimate_historical_meta()'s shares_outstanding
    tier 2 for that pattern).
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

    Loads the boundary maps at its own start, under Session Boundary
    Derivations' WHO LOADS rule: it owns db_conn and its callers
    (metadata_crawler.md, migration_tool.md) preload nothing. gap_pct's
    prev_close anchors on last_bar(D-k) for each prior session, so the maps are
    needed here and not merely passed through.

    For each as_of_date in dates:
        For each ticker with ohlcv_1min data:
            Compute per-bar average over prior n_sessions trading sessions.

    Metrics derived from ohlcv_1min:
        rvol_baseline      : cumulative volume per HHMMSS slot
        rel_dvol_baseline  : cumulative dollar volume per HHMMSS slot
        intra_vol_baseline : per-bar volume per HHMMSS slot
        intra_return_baseline: per-bar (close/prev_close - 1) per HHMMSS slot
        gap_pct_mean/std   : (today_regular_open - prev_close) / prev_close
                             stored as mean and std (hour='000000')

    Metrics derived from tick_10, via tick_bar_aggregates (see db_schema.md —
    this function reads the same table Strategy A's live-mode fallback does,
    rather than re-deriving from raw tick_10 independently):
        buy_ratio_baseline               : per-bar buyer_initiated_ratio per HHMMSS slot
        intra_tpm_baseline                : per-bar ticks-per-minute per HHMMSS slot
        intra_avg_vol_per_tick_baseline    : per-bar avg_vol_per_tick per HHMMSS slot

    Corporate-event and ticker-rename awareness: prior-session raw values are
    read via load_ohlcv_with_history() (so a ticker rename within the prior
    n_sessions window does not silently drop history), then adjusted per
    corporate_events before aggregation (see D. below). ticker_history is
    small and pre-loaded once as an in-memory dict at the start of this
    function's execution — no per-ticker DB query added to the inner loop for
    tickers with no rename (the vast majority).

    Halt handling per metric type:

    A. Cumulative metrics (rvol_baseline, rel_dvol_baseline):
        Halt bars within a session contribute volume=0 to that session's cumsum
        (halt naturally interrupts volume flow — this is correct behavior).
        Sessions with no ohlcv rows for that date (full-day no-data) are
        excluded from count entirely.
        EARLY-CLOSE SESSIONS ARE EXCLUDED WHOLESALE, on the same footing — a
        session that was not in progress across the whole slot range cannot
        supply a cumulative curve over it.
        WHY WHOLESALE AND NOT PER-SLOT, since B below is per-slot: a cumulative
        curve inherits from its neighbours, so a sample set that CHANGES between
        adjacent slots puts a step in the baseline itself. With 19 normal
        sessions and one early close, baseline[125900] averages 20 samples and
        baseline[130100] averages 19, and rvol = today_cum(T) / baseline(T)
        therefore steps at 13:00 for every date whose lookback contains that day
        — an artifact of sample composition, not of the market. It is worse than
        a fixed artifact: the window rolls, so the step appears, moves and
        vanishes as the early-close day enters and leaves the lookback, and a
        non-stationary artifact is not something the model absorbs. Delta
        smoothing does not localise it either — a [H-delta, H+delta] window
        straddling 13:00 mixes 20-sample and 19-sample values, spreading the
        discontinuity across the whole smoothing width.
        NORMALISATION was rejected: scaling by session length assumes volume is
        linear in it, and an early close is thin partly because fewer
        participants show up, not only because it is shorter. Before 13:00 there
        is nothing to normalise — those values are already correct — so the
        adjustment would corrupt good samples to fix absent ones.
        Cost: count = 19 rather than 20 at every slot, for roughly 20 as_of_dates
        per early-close day. The count column already records this and consumers
        already handle count < n_sessions, which occurs throughout the dataset's
        opening window.
        PARTIAL-SESSION BEHAVIOUR IS NOW STATED, having been undefined: whether
        the cumsum was `sum(hour <= T)` (which would carry an early-close
        session's total forward into every later slot, depressing the mean ~2.5%)
        or required a bar at slot T (dropping count) was never specified.
        Wholesale exclusion makes the question moot rather than answering it.

    B. Per-bar metrics (intra_vol_baseline, intra_return_baseline,
                        intra_tpm_baseline, buy_ratio_baseline):
        If bar H is a halt bar in session S (OHLC=NaN):
            Session S is excluded from the average for hour H only.
            count[H] reflects only sessions with non-halt bar at that slot.
            Other hours of session S are unaffected.

    C. Day-level metrics (gap_pct_mean, gap_pct_std), stored at hour='000000':
        gap_pct = (today_regular_open - prev_close) / prev_close

        prev_close determination (for each prior session D-k):
            1. Try D-k last_bar(D-k) close
            2. If halt or missing: search backward for last non-halt bar
               in D-k 093000~last_bar(D-k) (latest HHMMSS with non-NaN OHLC)
            3. If no non-halt bar found in D-k regular session: exclude session
            PER-DATE, not 155900. When D-k was an early close, 155900 does not
            exist and step 1 always misses; step 2's halt fallback then walks
            back to the real final bar and happens to land correctly — the right
            answer for the wrong reason, a halt device silently absorbing a
            structural difference in session length. Anchoring on last_bar(D-k)
            makes step 1 hit directly and leaves the fallback to do only the job
            it was built for.

        today_regular_open determination (for each prior session D-k):
            1. Try D-k 093000 bar open
            2. If halt or missing: search forward for first non-halt bar
               in D-k 093000~100000 (earliest HHMMSS with non-NaN open)
            3. If no non-halt bar found in first 60 minutes: exclude session

        Sessions excluded from either determination reduce count.

    D. Corporate-event adjustment (applied to prior-session raw values BEFORE
       A/B/C aggregation — not a post-hoc correction of the stored mean/std,
       since the individual per-session values are not retained after
       aggregation and cannot be un-averaged later):

        For ticker T, as_of_date D, prior session D-k:
            cum_ratio(D-k) = cum_split_ratio(T, D-k, D, db_conn)
                             (product of split/reverse_split 'value' with
                             event_date IN (D-k, D]; 1.0 if none — the common
                             case, no additional cost beyond the small
                             corporate_events lookup)

            Applied per metric:
                rvol_baseline, intra_vol_baseline (volume-based):
                    adjusted_volume = raw_volume * cum_ratio(D-k)
                    (post-split share count is higher → historical volume
                    scaled up to match current scale)

                rel_dvol_baseline (dollar volume = price * volume):
                    No adjustment needed — price scales down by 1/cum_ratio
                    while volume scales up by cum_ratio; product invariant.

                intra_return_baseline (per-bar % return):
                    No adjustment needed — percentage return is scale-invariant.

                gap_pct_mean/std:
                    adjusted_prev_close(D-k) =
                        (raw_prev_close(D-k) - dividend_at(D-k)) / split_at(D-k)
                        where split_at(D-k) = 'split'/'reverse_split' value
                            with event_date=D-k (1.0 if none)
                        dividend_at(D-k) = 'dividend' value with
                            event_date=D-k (0.0 if none)
                    gap_pct(D-k) = (today_regular_open(D-k) - adjusted_prev_close(D-k))
                                   / adjusted_prev_close(D-k)
                    (applied using the halt-fallback prev_close/today_open
                    values already determined in C above — this adjustment is
                    independent of, and applied after, the halt fallback)

                buy_ratio_baseline (a ratio, tick-derived): no adjustment needed
                    — scale-invariant, same reasoning as intra_return_baseline.

                intra_tpm_baseline (tick count, not volume): no adjustment
                    needed — a count is unaffected by a split, same reasoning
                    as any other invariant tick-derived quantity (see
                    02_indicator_calculator.md's Tick-Derived Indicator Scale
                    Sensitivity registry).

                intra_avg_vol_per_tick_baseline (volume-based, tick-derived):
                    adjusted_value = raw_value * cum_ratio(D-k) — same
                    direction and reasoning as rvol_baseline/intra_vol_baseline
                    above (post-split volume scales up to match current
                    share-count basis). This is the one tick-derived baseline
                    metric that DOES need correction — unlike buy_ratio/
                    intra_tpm above, its underlying indicator (avg_vol_per_tick)
                    is volume-scale_type, not invariant. Explicitly named here
                    (rather than left implicit) so a future tick-derived
                    baseline addition is not mis-classified by omission —
                    every tick-derived metric this function computes must
                    appear in this list with an explicit adjustment call,
                    not rely on absence-from-D meaning "no adjustment".

        This replaces session-exclusion as the split/dividend handling
        approach — adjustment preserves sample count (count is unaffected by
        D; only A/B/C reduce count) instead of discarding sessions outright.

    Stores: avg_value, std_value, count per (ticker, as_of_date, hour, metric, n_sessions)
    count may differ by hour slot within the same ticker/as_of_date (per-bar metrics).
    Delta smoothing NOT applied here — applied at load time via load_session_stats()
    or build_session_stats_dict().
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
    closes: dict[str, str],
) -> dict:
    """
    Load and smooth precomputed_session_stats for a ticker on a given date.

    Steps:
        1. Query precomputed_session_stats WHERE ticker=?, as_of_date=?, n_sessions=?
        2. Apply session_mode filter:
               "regular":  hours 093000~last_bar(as_of_date)
               "pre":      hours 040000~092900
               "combined": hours 040000~last_bar(as_of_date)
               Day-level metrics (hour='000000') always included regardless of session_mode.
           The last sentence is COPIED VERBATIM from build_session_stats_dict()
           below, where it was already stated and here was not. Read literally,
           the "regular" range excluded hour='000000' and therefore dropped
           gap_pct_mean/gap_pct_std — so a ticker loaded in the session-start bulk
           kept its gap_pct baseline while the same ticker arriving mid-session
           through Phase 2 lost it and gap_percentile returned NaN. One
           proposition, two copies, only one of them stated. The wording matches
           to the character on purpose: two copies that differ read as two
           propositions to the next sweep.
        3. Apply delta_minutes rolling average per metric:
               For each hour H, smoothed_value[H] =
                   mean(avg_value for hours in [H-delta, H+delta])
               Uses hhmmss_to_minutes() for window arithmetic.
        4. Return dict: {metric: {hour: smoothed_avg_value}}

    Returns empty dict if no rows found (e.g. as_of_date before dataset start).
    """
    ...


def build_session_stats_dict(
    raw_df: pd.DataFrame,
    delta_minutes: int,
    session_mode: str,
    closes: dict[str, str],
) -> dict:
    """
    Convert a bulk-loaded precomputed_session_stats DataFrame into a nested dict.

    Input raw_df columns: ticker, as_of_date, hour, metric, avg_value, std_value, count
    (typically loaded with a WHERE n_sessions = ? filter applied before calling this function)

    Output format:
        {as_of_date: {ticker: {metric: {hour: smoothed_avg_value}}}}

    Processing:
        1. Filter rows by session_mode (apply hour range filter per metric):
               "regular":  hours 093000~last_bar(as_of_date)
               "pre":      hours 040000~092900
               "combined": hours 040000~last_bar(as_of_date)
               Day-level metrics (hour='000000') always included regardless of session_mode.
        2. For each (as_of_date, ticker, metric):
               Apply delta_minutes smoothing over the time dimension:
               smoothed[H] = mean(avg_value for hours within ±delta_minutes of H)
               Uses hhmmss_to_minutes() for window arithmetic.
        3. Build nested dict structure.

    Used by:
        - run_preprocess.py Step 1 (training bulk load, multi-date):
              raw_df = SELECT * FROM precomputed_session_stats WHERE n_sessions = ?
              result: {as_of_date: {ticker: {metric: {hour: value}}}}
              dispatch: result[date][ticker] per entry point

        - live_mode_runner.py session_start Phase 1 (live bulk load, single date):
              raw_df = SELECT * FROM precomputed_session_stats WHERE as_of_date = ? AND n_sessions = ?
              result: {today_date: {ticker: {metric: {hour: value}}}}
              access: session_stats_bulk = result[today_date]
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
    Uses regular session bars only (093000~last_bar(date)) from ohlcv_1min.

    Loads the boundary maps at its own start, under Session Boundary
    Derivations' WHO LOADS rule: it owns db_conn and pipeline_optimizer.md
    preloads nothing. It reads ohlcv_1min rather than precomputed_session_stats,
    so nothing another consumer already holds would serve here.

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

### Trading API Utilities

```python
# bulk_api_call_chunked() removed. Chunking is vendor-shaped on BOTH axes —
# rate governance and the endpoint payload cap are each published and each
# carried by the SDK — so it sits on the far side of trading_api.md's
# boundary test. Neither caller survived that move: the quote path chunks
# behind TradingAPI, and query_halt_status() is not a dbsec call at all.


def query_halt_status(
    tickers: list[str],
) -> dict[str, bool] | None:
    """
    Bulk-query halt status for the given tickers — the single access point
    regardless of caller.

    NOT a dbsec vendor call. That vendor's catalogue publishes no
    halt-status endpoint: 20 REST endpoints across quote and trading, none
    returning halt state. So this cannot be served through trading_api.md,
    and the source is another vendor's API or a web source — undecided, and
    tracked in open_items.md. Its SHAPE is open too: halt data may be a
    market-wide feed rather than a per-ticker query, which would remove any
    need for chunking here.

    Takes no URL and no chunk size, and no config key backs it. The former
    `trading_api_url` reuse went with that key, and inventing a replacement
    now would assume a URL-shaped source that a market-wide feed may not be.

    Returns {ticker: is_halted} for tickers the source recognizes. A ticker
    requested but absent from the response is the caller's problem to handle
    (treated as "unknown", not as False) — this function does not silently
    default a missing ticker to not-halted. A market-wide source is filtered
    to the requested tickers, so the contract holds either way.

    Returns None (not an exception) on total failure of the query. This
    function does not decide the degraded-mode behavior for that case; each
    call site interprets None per its own context, since the two current
    call sites have different fallback signals available:
        - live_mode_runner.md's Position Manager Loop: falls back to the
          existing tick-rate heuristic per position.
        - metadata_crawler.md's check_corporate_event_anomaly(): no
          live-tick fallback exists in that offline batch context: treats
          an unresolved ticker as not-halted (conservative direction — see
          that function for why).

    UNTIL A SOURCE IS CHOSEN this returns None unconditionally, which both
    call sites already handle. Stating that is deliberate: the fallback
    paths are designed and specified, so an always-None keeps the Position
    Manager Loop buildable, whereas marking the function unimplemented would
    put a second unbuildable point in it.

    This function's contract — bulk ticker list in, {ticker: bool} | None
    out — is fixed now so both call sites can be designed against it
    immediately.

    Called by:
        - live_mode_runner.md's Position Manager Loop, Step 1 (halt check,
          source-primary + tick-rate fallback; was Step 1a before the former
          bar-fetch Step 1 was removed)
        - metadata_crawler.md's check_corporate_event_anomaly() (premarket
          quarantine check, P-8)
    """
    ...
```

---

### Health Event Recording

```python
def record_health_event(
    finding_name: str,
    occurred_at: str,
    ticker: str | None,
    detail: dict,
    write_fn: Callable[[str, tuple], None],
) -> str:
    """
    Append one row to `health_events` (see db_schema.md) and return the
    generated `event_id`.

    Sole access point for health-event writes — no detection site mints its
    own identifier, for the same single-source-of-truth reason
    `query_halt_status()` is the sole halt-status-endpoint access point.

    `event_id` format: '{YYYYMMDD}_{HHMMSSmmm}_{4 random chars}' — the same
    date_time shape as `run_id` elsewhere, extended to millisecond precision
    plus a random suffix so that two events in the same millisecond, or two
    processes writing this table (a live session and the evening batch — see
    metadata_crawler.md), cannot collide without coordination.

    Takes `write_fn`, NOT a `db_conn`. In a live session every write must go
    through LiveModeRunner's `db_write()` funnel (see live_mode_runner.md);
    a function handed a raw connection would bypass `write_lock`, which is
    exactly the defect `persist_to_db()` carried until R-2 GAP-FIX 9. Offline
    callers pass a direct wrapper.

    A write failure is caught here and never propagated — a diagnostic write
    must not abort trading (db_schema.md states this as a general principle
    for every diagnostic write in this system). The failure is counted, and
    the returned `event_id` stays valid for the caller's own alert payload,
    which reaches the operator whether or not the row landed.

    Called by:
        - live_mode_runner.md's Bar-Close Authority, on a negative bar-latency
          sample (health_report.md finding 27)
        - live_mode_runner.md's Broker Reconcile, on an unmatched broker order
          or position (finding 12)
        - live_mode_runner.md's Position Manager Loop Step 1, when a halted
          position is carried past session_end (finding 14), and when a
          halted→tradable transition finds the in-flight exit order gone
          (finding 25)
        - live_mode_runner.md's in-flight exit tracking, on an order's FIRST
          crossing of exit_order_stuck_minutes (finding 18)
        - live_mode_runner.md's fill-stream staleness detection, on entry into
          each stale episode (finding 24)
        - live_mode_runner.md's Watchdog Polling Loop, on entry into each
          episode of an empty working set inside the regular session
          (finding 29)
    Findings 13 and 15 deliberately do NOT call this — each already writes its
    own trade_log row, so a health_events row would be a second source for one
    fact. The rule: an event is recorded here when it leaves no row anywhere
    else. See db_schema.md's health_events SCOPE note.
    """
    ...
```

---

### Tick Stitching (R-2)

```python
def stitch_ticks(existing: pd.DataFrame, incoming: pd.DataFrame) -> pd.DataFrame:
    """
    Single source of truth for concatenating REST-fetched ticks onto an
    existing tick buffer/history. Used EVERYWHERE ticks are stitched: WS
    gap-fill, the WS-dead REST backstop, and warm-restart gap-fill catch-up
    (see live_mode_runner.md).

    TICK IDENTITY RULE (global): two ticks are the SAME only when `hour`
    AND ALL OHLCV fields match. `hour` alone is NOT a key — multiple ticks
    can share one wall-clock second (sub-second events collapse to the same
    `hour`), so deduping on `hour` alone would wrongly drop distinct ticks.
    Dedup on the full (hour, open, high, low, close, volume) tuple.

    Known limitation: two genuinely distinct fills that happen to coincide
    on `hour` + OHLCV merge into one. Negligible given 10-tick bundling in
    the historical data this dedup rule was designed against (it implies
    same-second, same-price-band activity within one bundle); recorded, not
    engineered around.

    THAT JUSTIFICATION DOES NOT CARRY TO 1-TICK DATA, where a row is one
    print: same-second, same-price, same-size prints are ordinary rather
    than implausible, so the limitation would stop being negligible. It
    does not become a problem, because 1-tick responses carry an
    INTRA-SECOND ORDINAL — the `nnnn` tail of the vendor's cont_key,
    `YYYYMMDDHHMMSSnnnn` — which is the same concept `tick_10.seq_id`
    encodes as groupby(ticker, date, hour).cumcount(). The `if present`
    branch below is therefore PRESENT on the live 1-tick path, and the
    ordinal is a true key there rather than a sort aid.
    Live buffers hold SEALED bundles and 1-tick rows only. A vendor 10-tick
    response's terminal bundle is unsealed and its `hour` and OHLCV update
    with every arriving print, so the same bundle fetched twice does not
    compare equal under the rule above and both copies survive as a
    phantom; it is discarded at the source rather than handled here
    (live_mode_runner.md).

    Returns: existing with incoming appended, deduped per the rule above,
    re-sorted by (hour, seq_id) if present.
    """
    ...
```

---

## Constraints

- All functions are stateless — no module-level state or caching
- `build_effective_bar_sequence()` is the single source of truth for valid bar counting
  — Labeler and BacktestEngine both import from here; logic must not be duplicated
- `build_effective_bar_sequence()` never returns bars with hour > last_bar(date) —
  after-market bars are excluded regardless of target_valid_bars
- `apply_overrides()` must deep-copy config — original must never be mutated
- `load_encoding_map()` returns empty dict (not error) when file does not exist
  — FeatureExtractor handles first-run map creation
- Encoding maps loaded exclusively by FeatureExtractor — Inferencer does not
  load or inject encoding maps directly
- All categorical encoding maps guarantee integer values, never NaN
- `halt_reason_code_map.json` must always contain `"no_halt": 0` as a reserved entry
- `track_price_breach()` is the low-level primitive for single tp/sl
  detection in BACKTEST ONLY (R-2). Live no longer calls it — live tp/sl
  detection is WS-primary / REST-backstop over the real tick stream (see
  live_mode_runner.md's Exit Architecture). The function's own contract is
  unchanged; only its live call site was removed. (Formerly also called by
  LiveModeRunner's Position Manager Loop.)
- `stitch_ticks()`'s tick-identity rule (hour + full OHLCV, not hour alone)
  applies wherever REST ticks are concatenated — see that function above.
  At 1-tick granularity the rule is completed by the intra-second ordinal
  rather than weakened: the tuple alone cannot separate two identical
  same-second prints, which the ordinal does. Only SEALED bundles are ever
  stitched — an unsealed terminal bundle mutates between fetches and would
  dedup as two distinct rows
- `track_label_breach()` is the high-level wrapper for two-stage label detection (Labeler only)
- `generate_run_id()` not called inside parallel workers or sequential fold loops —
  coordinator pre-generates structured run_ids in all optimizer contexts
- `compute_vol_regime_holdout()` uses regular session bars only
  (093000~last_bar(date))
- `compute_consensus_config()` grid-rounding uses nearest valid value from search_space
- `temporal_split_simple()` applies no balancing — training-only utility for early stopping val
- `populate_precomputed_session_stats()` uses INSERT OR IGNORE — safe to re-run
- `populate_precomputed_session_stats()` halt handling must be applied per metric type:
  cumulative metrics include halt bars as volume=0 AND exclude early-close
  sessions wholesale; per-bar metrics exclude halt slots per hour and need no
  early-close handling of their own — an early-close session simply supplies no
  sample past its close, which the existing per-slot count already expresses;
  gap_pct uses nearest non-halt bar fallback (see function docstring for details)
- `load_session_stats()` → single (ticker, as_of_date) DB query; output `{metric: {hour: value}}`
  (flat, no ticker/date dimension); used for live mode Phase 2 (per-ticker on demand)
- `build_session_stats_dict()` → bulk conversion of pre-loaded DataFrame;
  output `{as_of_date: {ticker: {metric: {hour: value}}}}`; used for training bulk load
  and live mode Phase 1 (session_start all-ticker bulk)
- `load_session_stats()` never modifies DB — read-only
- `load_session_stats()` returns empty dict (not error) when no rows found
- Delta smoothing in both `load_session_stats()` and `build_session_stats_dict()` is
  in-memory only — DB rows unchanged
- `buy_ratio_baseline` and `intra_tpm_baseline` require tick_10 data — skipped if unavailable
- `hhmmss_to_minutes()` is an alias for `hour_to_minutes()` with explicit naming for clarity
- `get_next_trading_day()` filters on `is_trading_day=TRUE` (NYSE calendar), not
  `has_data=TRUE` — it looks forward to a future date, distinct from the
  `has_data=TRUE` lookup used for dead position resolution (past/present date
  that has already been ingested)
- `get_ticker_history()` returns `None` (not an empty list) when no rename
  history exists — this is the default/fast path for the vast majority of
  tickers; callers must branch on `None` explicitly rather than assuming a
  list and checking its length
- `load_ohlcv_with_history()` and run_preprocess.md's in-memory equivalent
  (Step 6) must produce identical results for the same ticker/date range —
  the two exist only because live mode and training load bars via different
  mechanisms (per-ticker query vs. one-time bulk load), not because the
  underlying stitching logic differs
- `cum_split_ratio()` returns `1.0` (not an error or None) when no splits
  exist in the given range — this is the common case and must be a true no-op
  when multiplied against downstream values
- `adjust_bars_for_corporate_events()` does not mutate its input `bars`
  DataFrame — always returns a new copy, even when the adjustment is a no-op
- `adjust_bars_for_corporate_events()` handles splits only, not dividends —
  dividend adjustment is a separate scalar correction owned by each caller
  (`gap_percentile()`'s `dividend_amount`, dead position Case A/D's
  `adjusted_p_entry`) since dividends don't create the continuous bar-level
  price discontinuity that motivates a full-series bar adjustment
- `estimate_historical_meta()` must never be called with `field="sector"` —
  sector has no derivation path; callers use the most recent stock_meta
  snapshot for sector regardless of date
- `estimate_historical_meta()`'s `shares_outstanding` derivation does not
  capture buyback/issuance activity between quarters — only split-driven
  share-count changes are reversible from `ohlcv_1min` + `corporate_events`;
  this is a known, accepted limitation (see Open Items: fundamentals history
  collection) distinct from the corporate-event scale-correction problem this
  session's other changes solve exactly
- `populate_precomputed_session_stats()`'s corporate-event adjustment (D.) is
  applied to individual per-session raw values BEFORE aggregation into
  avg_value/std_value — it cannot be applied as a post-hoc correction to the
  stored mean/std, because the individual per-session values needed for
  correct adjustment are not retained after aggregation (this is also why
  session_stats correction is persisted in `precomputed_session_stats` at
  all, unlike bar-level correction which is computed fresh on every call —
  see `adjust_bars_for_corporate_events()`, which has no persisted-table
  equivalent because `ohlcv_1min` rows are never lossy-aggregated)
- No pipeline business logic in this module — utility functions only
- `adjust_tick_derived_series_for_corporate_events()` anchors directly to
  whatever `anchor_date` the caller supplies (normally entry_date) — it does
  NOT go through reference_date the way `adjust_bars_for_corporate_events()`
  does, because tick-derived series were never reference_date-anchored to
  begin with; do not add a reference_date parameter to this function
- `compute_tick_bar_aggregates()` must call IndicatorCalculator's own
  per-indicator methods (avg_vol_per_tick(), tpm(), etc.) rather than
  reimplementing their formulas — this is what guarantees a cached
  tick_bar_aggregates value and a freshly-computed one can never diverge
- `load_tick_bar_aggregates_with_history()`'s Tier 2 (on-the-fly compute)
  result is never written back to `tick_bar_aggregates` — only
  collect_daily.py's offline batch writes that table; Tier 2 existing at all
  is expected to be rare in steady-state operation, not a sign of a broken
  pipeline
- Tick-derived indicators requiring more than one tick bundle to be defined
  (e.g. any delta- or interval-based indicator between bundles) must return
  NULL/NaN for bars with fewer bundles than required, rather than raising or
  silently omitting the row — see 02_indicator_calculator.md's Tick-Derived
  Indicator Scale Sensitivity registry for the current minimum-bundle-count
  requirements per indicator
- `get_ttm_value()` and `estimate_historical_meta()`'s `fundamentals_quarterly`
  tier both filter on `filed_date <= date`, never `fiscal_period_end` — see
  data_boundary.md's point-in-time principle; this applies to any future
  fundamentals_quarterly consumer as well
- `resolve_xbrl_tag_value()`'s tag-priority list per metric lives in
  `configs/xbrl_tag_map.json`, not hardcoded in this function — adding a
  fallback tag for an existing metric is a config change, not a code change
- `query_halt_status()` is the sole halt-status access point — neither
  live_mode_runner.md nor metadata_crawler.md queries a halt source
  directly, for the same single-source-of-truth reason
  `compute_tick_bar_aggregates()` is the sole IndicatorCalculator wrapper.
  It is NOT a trading-API endpoint: the dbsec catalogue has none, so this
  function sits outside trading_api.md's boundary entirely
- `query_halt_status()` returning `None` is a call failure, not "no tickers
  halted" — callers must not conflate the two; conflating them would mean a
  dead endpoint silently reads as "everything is fine"
