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
    hold_minutes: int,
) -> tuple[str, pd.DataFrame]:
    """
    Resolve the holding deadline from t_hour and the existing bars up to it.
    Used by Labeler and BacktestEngine for consistent holding.

    Holding deadline: the instant at which hold_minutes minutes have elapsed
    from t_hour, wall clock, the minutes inside halts_df's intervals
    excluded.

    Collection boundary: bars up to and including last_bar(date) only.
    After-market bars (hour > last_bar(date)) are NEVER included, wherever
    the deadline falls. If last_bar(date) comes before the deadline,
    collection stops there.

    PER-DATE, not the fixed 155900 this once assumed. On an early-close day the
    session ends at 13:00, so the boundary is 125900 and there are no bars after
    it.
    last_bar(), NOT exit_deadline(): this is a data fact (the final bar of the
    session), not the exit policy. They coincide only while
    execution.session_close_exit_offset_minutes is 1.

    Bars: only the bars the broker supplied — a minute without trades has no
    bar and none is created, and no halt row exists in the bar frame.
    Halts: the intervals of halts_df, whatever the hour — the halt-interval
    resolver's intervals for date (halt_intervals(), below); an interval whose
    end is NULL covers that date from its start on.

    Args:
        ohlcv_future:  bars from t_hour onward for (ticker, date)
        halts_df:      the halt-interval resolver's intervals for date
        date:          'YYYYMMDD'
        t_hour:        'HHMMSS' — t bar open time (start of search)
        closes:        the {date: session_close} map, for last_bar(date).
                       Without it this function had no way to know the
                       boundary its own docstring states above.
        hold_minutes:  the holding minutes. NO DEFAULT — the caller passes
                       execution.max_hold_minutes. A default equal to the
                       configured value is a silent bypass no test detects
                       while the two agree, and here it once broke WITHIN one
                       engine: 09_backtest_engine.md checked the key's worth
                       of holding over a sequence this function would have
                       capped at 60, so raising the key past 60 made its
                       time-limit exit unreachable rather than merely late.

    Returns:
        (deadline, bars):
        - deadline: the holding deadline instant, 'HHMMSS';
        - bars: pd.DataFrame [hour, open, high, low, close, volume], the
          existing bars from t_hour up to the deadline, bounded by
          last_bar(date). Never contains bars with hour > last_bar(date).
    """
    ...
```

---

### Halt Interval Resolver

Every halt interval in the system is taken from this resolver: each halts_df
supply site, and each per-date consumer — build_effective_bar_sequence(), the
Labeler, BacktestEngine — receiving its date's intervals.

Config `halts.continuation` selects how a halt left open at a trading day's
end reaches the following days:
- `"none"` (default): stored trading_halts rows only;
- `"until_first_bar"`: the continuation rule below. Its cost: the minutes
  between a resumption and a thin ticker's first trade are classified halt.

Switching it is a human config change made after metadata_crawler.md's Halt
Comparison metrics (live_scan_daily `halt_open_end_carried` /
`halt_open_end_uncarried`) are read. A changed value re-keys derived rows
through config_hash, and Health Gate 1d's feature-contract hash blocks a model
trained under the other value.

```python
def halt_intervals(
    ticker: str,
    dates: list[str],
    src,
) -> dict[str, pd.DataFrame]:
    """
    Per date, the merged halt intervals of the security, each with start,
    end, reason_code and an inferred flag.

    - Stored trading_halts rows are read through the symbology layer
      (Ticker Identity Utilities).
    - Under "until_first_bar", a date d receives [PREMARKET_OPEN, d's first
      existing bar) when halt_open_into(ticker, d, src) holds and d has a bar
      and no row starting at or before its first bar. A date without a bar
      and without any row is covered whole when halt_open_into holds, and
      passes the open halt on.
    - An inferred interval takes the originating row's reason_code.
    - Overlapping intervals merge into one, keeping the earlier interval's
      reason_code.
    - Bar presence and a date's first bar hour come from ohlcv_1min only.
    - Under "none": the stored rows, merged.

    src: the caller's loaded frames within their range and the DB outside
    it; a guest caller's DB reads happen inside one of its units —
    run_preprocess.md's batch read unit runs the halt_open_into walk beyond
    the load range and loads its rows.
    """
    ...


def halt_open_into(ticker: str, d: str, src) -> bool:
    """
    True when the previous trading day of d (trading_calendar) ends in a row
    whose halt_end is NULL, or is covered whole by a continuation.
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
def interpolate_bundle_price(
    prev_bundle: pd.Series | None,
    curr_bundle: pd.Series,
    t_prev: float | None,
    t_curr: float,
    t_target: float,
) -> float:
    """
    Estimate the price at t_target within curr_bundle using linear interpolation.

    Times are continuous — seconds since midnight, float:
    - expand_synthetic_ticks() (below) passes each bundle's effective end
      time and the synthetic tick's time;
    - a caller holding an HHMMSS second s passes s's seconds since midnight
      (HH × 3600 + MM × 60 + SS) — track_price_breach() does so for bundle
      hours and breach_second.

    Anchors:
        t_prev → prev_bundle.close          (last tick of prev bundle)
        t_mid  → typical_price              (high + low + close) / 3 of curr_bundle
        t_curr → curr_bundle.close          (last tick of curr bundle)
        t_mid  = (t_prev + t_curr) / 2      (midpoint assumption for H/L occurrence)

    Formula:
        p_mid  = (curr_bundle.high + curr_bundle.low + curr_bundle.close) / 3
        t_mid  = (t_prev + t_curr) / 2

        if t_target <= t_mid:
            ratio = (t_target - t_prev) / (t_mid - t_prev)
            est   = prev_bundle.close + ratio * (p_mid - prev_bundle.close)
        else:
            ratio = (t_target - t_mid) / (t_curr - t_mid)
            est   = p_mid + ratio * (curr_bundle.close - p_mid)

        est = clamp(est, curr_bundle.low, curr_bundle.high)

    Fallback (prev_bundle is None):
        est = curr_bundle.open
        est = clamp(est, curr_bundle.low, curr_bundle.high)

    Fallback (zero-duration bundle, t_prev == t_curr):
        est = curr_bundle.close

    Args:
        prev_bundle: pd.Series of the bundle before curr_bundle; None if not available
        curr_bundle: pd.Series of the bundle containing t_target
        t_prev:      prev_bundle's time; None when prev_bundle is None
        t_curr:      curr_bundle's time
        t_target:    must satisfy t_prev < t_target <= t_curr

    Returns:
        Estimated price (float), clamped to [curr_bundle.low, curr_bundle.high]
    """
    ...


def expand_synthetic_ticks(
    ticks: pd.DataFrame,
    halts_df: pd.DataFrame,
) -> tuple[pd.DataFrame, pd.Series]:
    """
    Expand one (ticker, date)'s tick_10 bundles into synthetic ticks — the
    tick sequence BacktestEngine's fill simulation and in-window breach
    evaluation run on (09_backtest_engine.md; execution_common.md's
    simulate_entry_fill() / simulate_exit_fill()).

    Runs:
    - a run is a maximal sequence of consecutive bundles sharing one hour h;
    - a run of k bundles expands to 10k synthetic ticks, its bundles taking
      them in (hour, seq_id) order, ten each.

    Placement, per run:
    - interval: [h0 + 1, h + 1), h0 being the hour of the bundle preceding
      the run on the same date; [h, h + 1) when none precedes it on that
      date;
    - only the interval's non-halt time is used (halts_df, the halt-interval
      resolver's intervals for the date, an interval whose end is NULL
      covering that date from its start on);
    - tick j, j = 0 .. 10k − 1, sits where the non-halt time elapsed since
      the interval start is j·Δ, Δ = the interval's non-halt length / 10k.

    Per synthetic tick:
    - volume = its bundle's volume / 10;
    - price = interpolate_bundle_price() over effective times — prev_bundle
      and t_prev the preceding bundle and its effective end time (None for
      the date's first bundle), curr_bundle and t_curr the tick's own bundle
      and its effective end time, t_target the tick's time.

    A bundle's effective end time is its last synthetic tick's time.

    Times are continuous (seconds since midnight, float). Synthetic ticks are
    compared with HHMMSS seconds by the integer part of their time (Hour
    String Utilities' second-stamp semantics).

    Returns:
        synthetic:     pd.DataFrame [time, price, volume], ordered by time
        effective_end: pd.Series of effective end times, aligned to ticks' index
    """
    ...
```

---

### Price Breach Detection

```python
class AmbiguityRule:
    """
    Resolution of an ambiguous bundle — tp and sl reached inside one tick_10
    bundle.

    AmbiguityRule(priority, seed):
        priority: "up" | "dn" | "ohlc_path"
        construction raises ConfigurationError for any other priority, and
        for "ohlc_path" without an integer seed.

    "up" / "dn": that direction is reached first.
    "ohlc_path":
        - the bundle's open nearer its high → the high is reached first;
        - nearer its low → the low is reached first;
        - equidistant → the lowest bit of SHA-256 over the UTF-8 text
          "{seed}|{the bundle's origin symbol}|{date}|{hour}|{seq_id}", the
          origin symbol being the symbology layer's origin-symbol column:
          0 → the high first, 1 → the low first.
        The open-distance comparison is a grid-value decision (Numeric
        Rules, below).

    Built at init:
        - BacktestEngine: from execution.ambiguity_priority and
          execution.ambiguity_seed, passed to track_price_breach();
        - Labeler: from labeler.ambiguity_priority with no seed ("up" | "dn"),
          passed to track_label_breach().
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
    ambiguity: AmbiguityRule,
) -> tuple[str | None, float, str, bool]:
    """
    Detect the first tp/sl threshold breach from fill_second onward.
    Callers:
    - BacktestEngine, after the entry window, fill_price being the entry's
      fixed weighted-average fill price (single threshold pair, immediate
      exit; 09_backtest_engine.md) — a breach inside the window is
      BacktestEngine's own, evaluated on synthetic ticks;
    - track_label_breach() (below), for the Labeler.

    Phase 1 — tick-level scan within t bar:
        Iterate tick bundles after the bundle containing fill_second — the
        first bundle with hour >= fill_second, itself excluded — within the
        t bar.
        For each bundle: check if high >= tp_target or low <= sl_target.
        Each breach test is a grid-value decision (Numeric Rules, below).
        Ambiguity: if both thresholds satisfied within the same bundle,
            the AmbiguityRule `ambiguity` (no default) decides which is
            reached first.
        On breach: interpolate price at breach_second
            (interpolate_bundle_price(), its times continuous); record
            direction.

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
    ambiguity: AmbiguityRule,
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
        ambiguity:          the Labeler's AmbiguityRule, built from
                            labeler.ambiguity_priority; no default

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

Second-stamp semantics:
- an HHMMSS second s — a vendor hour or a 1-tick second — denotes the
  interval [s, s + 1);
- a continuous time falls in s when its integer part is s's seconds since
  midnight.

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
    Used for:
    - t+5s target computation in entry slippage;
    - every time computation on a fill-simulation anchor — the entry delay
      and the cancel_after_seconds window (execution_common.md).
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
    Its callers:
        - run_train.md's and run_backtest.md's standalone default run_id;
        - pipeline_optimizer.md's optimizer_run_id, at work creation;
        - pipeline_optimizer.md's exploitation and refit closing steps' final
          model run_id;
        - guest_supervisor.md's work_id of a tool other than
          pipeline_optimizer, after `<tool>_`.
    db_coordination.md's register() run_id takes its form.

    Format: YYYYMMDD_HHMMSS
    Example: "20250715_143022"

    Uniqueness guaranteed by 1-second granularity for sequential callers.

    In nested validation (parallel workers via ProcessPoolExecutor), the coordinator
    pre-generates structured run_ids BEFORE dispatching workers to avoid collision:

        format:  {optimizer_run_id}_o{outer_fold_idx}_t{trial_idx}_f{inner_fold_idx}
        example: "20250715_143022_o1_t4_f2"

        outer eval:           {optimizer_run_id}_o{outer_fold_idx}_eval
        execution variant:    {optimizer_run_id}_o{outer_fold_idx}_eval_x{i}
        champion outer eval:  {optimizer_run_id}_o{outer_fold_idx}_champ

    In sequential fold loops, the coordinator similarly pre-generates
    structured run_ids to avoid collision between folds (fold processing may
    complete in under 1 second on small datasets):

        full phase:
            format:  {optimizer_run_id}_f{fold_idx}
            example: "20250715_143022_f2"
        selection phase (pipeline_optimizer.md's Stage A and Stage B):
            Stage A:    {optimizer_run_id}_f{fold_idx}_g{k}
            Stage B-1:  {optimizer_run_id}_f{fold_idx}_r{j}
            Stage B-2:  {optimizer_run_id}_f{fold_idx}_c{c}

    Workers receive run_id as an explicit argument; generate_run_id() is NOT called
    inside worker functions or fold loops. The standard YYYYMMDD_HHMMSS format is
    used only by the callers above.
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

### Config Hashing

The single definition of canonical JSON and of config_hash. Every
config_hash is computed here:
- guest_supervisor.md's work identity;
- each derived table's producer (db_schema.md's Derived tables);
- deployability_checks() (below), for live_mode_runner.md's Health Gate 1
  check d and model_deploy.md.

The content gains a `code_versions` member holding area versions, the
lock's (Code Versions, below):
- the data version:
  - the Preprocessor work and labeled_samples;
  - precomputed_session_stats;
- the data and training versions: the pipeline_optimizer work.
derived_configs.config_json carries the versions inside code_versions; no
code_hash enters it.

Canonical JSON of a value:
- keys sorted at every level;
- UTF-8;
- no insignificant whitespace;
- values as the YAML loader parses them, serialized by the standard JSON
  encoder.

```python
def canonical_json(value) -> str:
    """The canonical JSON text of `value` (rules above)."""
    ...


def config_hash(content) -> str:
    """
    The hash of canonical_json(content). The content is the caller's: a
    producer's Config Keys Used, or a work's covered config with its input
    files' contents (guest_supervisor.md).
    """
    ...


def deployability_checks(meta: dict, config: dict, lock: dict) -> list[dict]:
    """
    The failed checks with their details; an empty list when the model is
    deployable under config and the lock:
        - the model's feature_names are a subset of
          FeatureExtractor(config).get_feature_names();
        - for each derived table the runner reads (precomputed_session_stats),
          the config_hash from the model's meta.json config snapshot equals
          the one from config;
        - the feature-contract config_hash — config_hash() over the keys
          run_preprocess.md's Config Keys Used lists inside config_hash,
          `labeler.*` excluded, with code_versions holding the data version:
          the model's, from its meta.json config snapshot and recorded data
          version, equals the deployment's, from config and the lock's data
          version.
    Called by live_mode_runner.md's Health Gate 1 check d (its BLOCK and
    cause detail unchanged) and by model_deploy.md.
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
| `configs/security_type_map.json` | FeatureExtractor (when absent) | FeatureExtractor | content fixed: unknown → 0; equity → 1; etf → 2; fund → 3; adr → 4; warrant → 5; unit → 6; right → 7; preferred → 8; debt → 9 |

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
open and the regular open are the exception, named constants below
(PREMARKET_OPEN, REGULAR_OPEN): the NYSE regular open does not move except
under exchange outage, which the calendar library would not reflect in any
case.

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

NEXT_SESSION_CLOSE IS NOT A DERIVATION. trading_calendar's next_session_close
is the calendar context's record of the next trading day's close as it stood
before the date's session (Trading Calendar Utilities' calendar_context()),
outside these derivations and read by the calendar context alone; the opening
"Two columns" above is about the date's own boundaries, session_close and
after_hours_end.

```python
PREMARKET_OPEN = "040000"
REGULAR_OPEN = "093000"
REGULAR_CLOSE = "160000"   # the ordinary exchange close: a comparison
                           # reference, never a fallback for a date absent
                           # from the map


def session_phase(
    date: str,
    hhmmss: str,
    closes: dict[str, str],
    ends: dict[str, str],
) -> str:
    """
    The session phase of an instant: 'premarket' | 'regular' | 'after_hours'
    | 'closed'. Every determination of an instant's session phase calls this.
    - 'premarket'   when PREMARKET_OPEN <= hhmmss < REGULAR_OPEN;
    - 'regular'     when REGULAR_OPEN <= hhmmss < session_close(date);
    - 'after_hours' when session_close(date) <= hhmmss < after_hours_end(date);
    - 'closed'      otherwise.
    An instant with fractional seconds is classified by its integer second.
    The boundary maps are taken per the call-site notation above.
    """
    ...


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

    A DATA FACT. It is where bar collection stops and where a prior session's
    close price is read. It does not move with policy.

    Callers: build_effective_bar_sequence()'s collection boundary;
    gap_pct's prev_close anchor (this file, 02_indicator_calculator.md,
    migration_tool.md, metadata_crawler.md — every copy of one anchor);
    05_labeler.md's Steps 1/3/4. The phase tests — load_session_stats() /
    build_session_stats_dict()'s slots, 01_entry_detection.md's scan()
    regular-phase step, inferencer.md's step 1 check — are session_phase()
    users.

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
    Sets each date of its range's has_data to whether ohlcv_1min holds that
    date, TRUE to FALSE included.
    Safe to re-run (upsert).
    Appends a source_changes row (ticker NULL) per inserted or changed date,
    in the transaction of its writes; an unchanged date appends none
    (db_schema.md's Source changes).

    FORWARD FILL — ONE YEAR AHEAD, ROLLING. The run day: the US-Eastern date on
    which a run executes.
    date_range=None populates from the earliest date already present through one
    year past the run day, and the daily run extends that horizon by a day each
    time it runs rather than refreshing it in an annual batch, so the horizon never
    reaches an expiry cliff. get_next_trading_day()'s "calendar should be
    populated far enough ahead by populate_trading_calendar()" is met by the
    call sites' date ranges:
        - migration_tool.md's calendar initialization: every date from one
          calendar year before its first ingested date through one year past
          the run day;
        - metadata_crawler.md's daily Calendar and Coverage Update: the
          processed date and every date from the run day through one year
          past it;
        - migration_tool.md's Ingestion-Range Cleanup: date_range None, from
          the earliest date present through one year past the run day.
    What the rolling horizon closes: on reaching the last populated date the
    daily collection raises at get_next_trading_day(), tomorrow's
    precomputed_session_stats is
    never written, and Health Gate 1a blocks tomorrow's participation — the ending
    health_report.md finding 28 describes for an evening-job abort
    (`owner='evening'`). The raise rule on a missing
    session_close (below) makes the dependency heavier still.
    A forward row carries is_trading_day, is_holiday, session_close,
    after_hours_end and the calendar context columns, and has_data=FALSE.
    That FALSE is correct, not a gap.

    CALENDAR CONTEXT COLUMNS — next_closure_last_session, next_closure_reopen,
    prev_closure_last_session, prev_closure_reopen, next_session_close
    (db_schema.md's trading_calendar; a closure is a run of consecutive
    non-trading days holding a weekday), each as the schedule stood before the
    row's date's session. Computed from the library's schedule whatever the
    table's own date range, and written:
    - on a row's insert;
    - on a row whose value is NULL, whatever its date;
    - otherwise only while the row's date is after the run day: the daily
      near-term rewrite carries announced closures up to the evening before,
      and a row whose date is reached keeps its values, date_range=None
      re-runs included.

    THE DAILY RUN REWRITES A NEAR-TERM WINDOW, it does not only append. An
    unscheduled early close — weather, national mourning — is announced days
    ahead and must overwrite a session_close written a year earlier. Append-only
    extension would leave the stale '160000' in place for exactly the case that
    cannot be predicted a year out.

    Returns: number of rows upserted.
    """
    ...


def prior_trading_days(date: str, n: int, closes: dict[str, str]) -> list[str]:
    """
    The n trading days before date, in ascending order, taken from the closes
    map's dates. Raises when the map holds fewer.

    The lookback start is its first element: run_preprocess.md's load range
    start and live_mode_runner.md's lookback_start_date, set at the session
    load, are defined through it.
    """
    ...


def has_required_history(
    bar_dates,
    entry_date: str,
    closes: dict[str, str],
    lookback_days: int,
) -> bool:
    """
    True when every date of prior_trading_days(entry_date, lookback_days,
    closes) is in bar_dates.

    Applied at:
    - EntryPointDetector.scan(), an exclusion step between the regular-phase
      step and the late-day step;
    - Inferencer's _prepare_bars() step 3;
    - metadata_crawler.md's evening detection-gap filter list.
    """
    ...


def calendar_context(
    calendar_df: pd.DataFrame,
    dates: list[str],
) -> dict[str, dict]:
    """
    Per date, its calendar context: the trading_calendar row's context columns
    (populate_trading_calendar() above) —
    - next_closure_last_session;
    - next_closure_reopen;
    - prev_closure_last_session;
    - prev_closure_reopen;
    - next_session_close.
    Read by FeatureExtractor's temporal features: run_preprocess.md's Step 6
    passes extract_batch() calendar_context(calendar_df, its entry dates);
    live_mode_runner.md loads today's row at Session Lifecycle's start and
    reads it through this function.
    """
    ...


def populate_ticker_coverage(
    db_conn: duckdb.DuckDBPyConnection,
    dates: list[str] | None = None,
) -> int:
    """
    Populate or refresh ticker_data_coverage table.
    For each (ticker, date) of its dates holding an ohlcv_1min or tick_10 row,
    sets has_1min and has_tick each to whether its table holds a row.
    Safe to re-run (upsert).
    Appends no source_changes row: its rows are covered by the source_changes
    rows of the same (ticker, date) that the ingestion or migration_tool.md's
    Ingestion-Range Cleanup appends (db_schema.md's Source changes).
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


def ticker_segments(
    ticker: str,
    start: str,
    end: str,
    db_conn: duckdb.DuckDBPyConnection,
) -> list[tuple[str, str, str]]:
    """
    The symbology layer: a security's symbols as date-ranged segments,
    [(symbol, date_from, date_to)], over its whole rename chain —
    predecessors and successors — within [start, end].

    A chain [(A, e1), (B, e2)] queried as C gives:
        - A before e1;
        - B in [e1, e2);
        - C from e2.
    A symbol denotes the security only inside its own segment; outside it
    the symbol is another security.

    The chain comes from ticker_history. CIK is a rename-detection source,
    not a key.

    Limit: a rename or symbol reuse absent from ticker_history is not
    detected.

    Tables read through the layer:
        - ohlcv_1min;
        - tick_10;
        - tick_bar_aggregates;
        - trading_halts;
        - ticker_data_coverage;
        - stock_meta.
    A read of these tables that serves a security's frame or history over
    dates goes through the layer (metadata_crawler.md's quarantine-check
    yesterday_close and evening detection-gap filter's bar history among
    them). The security of (symbol, date) is resolved this way by scan,
    extract_batch() and populate_precomputed_session_stats()' groups.
    A read keyed by the stored symbol itself stays raw:
        - ingestion and its skip checks;
        - universe enumeration;
        - rename detection and effective-date self-correction;
        - the evening_tick_bar_aggregates stage's per-symbol computation and
          writes;
        - the Halt Comparison's halt_live_only and halt_nyse_only row
          comparison.
    """
    ...


def load_ticker_rows(
    table: str,
    ticker: str,
    start: str,
    end: str,
    db_conn: duckdb.DuckDBPyConnection,
    row_filter=None,
) -> pd.DataFrame:
    """
    A layered table's rows for the security over [start, end]: one query per
    ticker_segments() segment, each under its own symbol and date range, the
    `ticker` column relabelled to the current ticker and an origin-symbol
    column kept.
    """
    ...


def slice_ticker_rows(
    df: pd.DataFrame,
    ticker: str,
    segments: list[tuple[str, str, str]],
) -> pd.DataFrame:
    """
    load_ticker_rows() over frames already loaded: each segment's symbol rows
    within its date range, relabelled to the current ticker, the
    origin-symbol column kept.
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
    stored under the other symbols of its chain: load_ticker_rows() on
    ohlcv_1min.

    Each ticker_segments() segment is read under its symbol over the
    segment's date range; the `ticker` column is relabelled to the current
    ticker and the origin-symbol column kept. A ticker without history has a
    single segment, one direct query. Returned DataFrame is sorted by
    (date, hour) and its `ticker` column is normalized to the current ticker
    throughout.

    Used by live_mode_runner.md's Session Lifecycle Step 5 (Eager Pool)
    historical bar load, where bars are
    fetched per-ticker via a direct DB query (unlike run_preprocess.md, which
    bulk-loads each batch's tickers and the chain symbols of each of their
    segments over the batch's load range and performs the equivalent
    relabeling in-memory through slice_ticker_rows() instead of via a second
    query — see run_preprocess.md Step 6 for that variant).
    """
    ...


def normalize_vendor_symbol(
    symbol: str,
) -> str:
    """
    Fold a vendor's symbol string toward the SEC notation this project's
    `ticker` column carries, so a caller can match it against a ticker set
    it already holds.

    A DIFFERENT AXIS from get_ticker_history() above. That one resolves one
    security's symbol changing over TIME; this one resolves two sources
    spelling the SAME security differently at one INSTANT — case, separator,
    and class-suffix rendering (a warrant reaching us as `ACHR.W` from one
    source and `UCFIW` from another). Neither substitutes for the other.

    Pure string transform. Takes no db_conn, reads no table, and returns no
    CIK: a caller that needs the row looks it up itself with the returned
    candidate. That keeps this callable from a context holding no database
    handle.

    BEST-EFFORT, NOT A GUARANTEE. A vendor may spell a security in a way no
    rule here folds. match_vendor_symbol() below runs this fold as its rule
    stage, and its loose and name stages take part of what the fold misses;
    what no stage matches is the residue. Where a caller has no manual path
    for the residue, the path is measured rather than assumed away —
    health_report.md's finding 10 counts, per source, the items matched only
    at the loose or name stage against the items matched at any stage, and
    counts the unmatched items apart. Where a caller has one, the residue
    goes there instead: build_trading_api_symbol_map()'s candidates log is
    worked off by manual registration, making it a backlog rather than a rate
    to watch. Callers treat a non-match as unmatched, never as a different
    security.

    Called by:
        - match_vendor_symbol(), its rule stage
    """
    ...


def build_symbol_index(
    db_conn: duckdb.DuckDBPyConnection,
    scope: str,
) -> dict:
    """
    The index match_vendor_symbol() matches against, built over
    ticker_cik_map's current rows (db_schema.md's current-row rule).

    Scopes:
        - 'tradable':
          - rows — current rows whose trading_api_seen_date equals the
            latest trading_api_seen_date among current rows of the same
            trading_api_exchange;
          - exchange — from trading_api_exchange;
          - English names — from trading_api_name when it contains no
            Hangul, and from sec_title;
        - 'sec':
          - rows — current rows;
          - name — sec_title;
          - no exchange.

    Built by:
        - live_mode_runner.md — the 'tradable' index, handed to the
          halt-status poller at its start;
        - metadata_crawler.md — the 'tradable' index:
          - the --premarket-open process;
          - the guest flags re-running its steps: --corporate-events-only
            and --premarket-recheck;
          - the evening run, for its ticker-master check and forward check;
        - metadata_crawler.md's build_trading_api_symbol_map() — the 'sec'
          index;
        - metadata_crawler.md's metadata fetch — the 'sec' index, ahead of
          its per-ticker loop.
    """
    ...


def match_vendor_symbol(
    symbol: str,
    name: str | None,
    exchange_hint: str | None,
    index: dict,
) -> tuple[str | None, str]:
    """
    The single path from a vendor's symbol to this project's ticker.
    Returns (ticker | None, stage).

    Stages, in order:
        1. exact — the symbol as given;
        2. rule — normalize_vendor_symbol()'s fold;
        3. loose — key = the symbol's uppercase letters and digits only;
           accepted only when the key maps to exactly one index ticker and,
           where the source supplies an exchange, that ticker's exchange
           agrees;
        4. name — only against index tickers holding an English name;
           accepted only when, after normalization, one side's token set
           contains the other's for exactly one ticker. Name normalization:
             - drops case;
             - drops punctuation;
             - drops legal-form tokens (Inc, Corp, Ltd, plc …);
             - keeps security-type tokens (unit, warrant, class …).
    More than one candidate at a stage → no match at that stage. Unmatched
    after every stage → (None, 'unmatched'), never a different security.

    Called by, each with its index:
        - live_mode_runner.md's halt-status poller and its single-fetch
          entry — the 'tradable' index, exchange_hint from the feed's Mkt
          (NASDAQ → NASDAQ; Non-NASDAQ → NYSE or AMEX);
        - metadata_crawler.md's crawl_corporate_events_investing(), at each
          of its call sites (metadata_crawler.md) — its caller's 'tradable'
          index, no exchange_hint;
        - metadata_crawler.md's build_trading_api_symbol_map() — the 'sec'
          index; exact and rule matches applied, loose and name matches
          written to tools/trading_api_symbol_candidates.log as proposed
          matches for manual registration;
        - metadata_crawler.md's metadata fetch — the 'sec' index, exact and
          rule stages, for each Nasdaq Trader symbol directory row not
          flagged Test Issue Y, the row's symbol fields in turn
          (nasdaqlisted.txt: Symbol; otherlisted.txt: ACT Symbol, then CQS
          Symbol, then NASDAQ Symbol).
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
    Cumulative product of split/reverse_split 'ratio' in corporate_events for
    this ticker with event_date IN (from_date, to_date]. Returns 1.0 if none
    (the common case — no query cost beyond a small corporate_events lookup,
    since the table itself is small regardless of match).

    THE split factor rule — one rule, with the (from_date, to_date] window as
    its parameter. Its sole owner is split_ratio_from_events() below: this
    function SELECTs the ticker's split/reverse_split rows and delegates to
    it, so the two entry points cannot diverge. db_schema.md's
    live_position_state view implements the same rule in SQL; no other
    formulation exists.

    Shared primitive used by:
        - live_mode_runner.md's trade_log close normalisation, and the
          predicted-against-real comparison of finding 7 /
          fit_execution_params() (shadow_retraining.md)
        - populate_precomputed_session_stats()'s price_52h / price_52l, each
          bar's prices split-adjusted to D
        - estimate_historical_meta()'s market_cap, and split_basis_value()'s
          ratio (below)
    FeatureExtractor (its f_e rescale and dilution_rate's ratio), the
    Labeler, and BacktestEngine (dead position Case A/D, the carried
    increment, the carried split normalisation, the carried dividend) read
    the frame form split_ratio_from_events() below, from the rows they hold.

    Query:
        SELECT ticker, event_date, event_type, ratio FROM corporate_events
        WHERE ticker = ? AND event_type IN ('split', 'reverse_split')
          AND event_date > ? AND event_date <= ?
        -- the rows split_ratio_from_events() reads; it returns their
        -- product, 1.0 if no rows
    """
    ...


def split_ratio_from_events(
    ticker: str,
    from_date: str,
    to_date: str,
    events_df: pd.DataFrame,
) -> float:
    """
    The split factor over (from_date, to_date] from an already-loaded
    corporate_events frame, for a consumer that holds the table already —
    run_preprocess.md SELECTs it at Step 1 and passes it on to
    05_labeler.md, which never queries DuckDB.

    SOLE OWNER OF THE SPLIT FACTOR RULE: the product of 'ratio' over this
    ticker's rows with event_type IN ('split', 'reverse_split') and
    event_date IN (from_date, to_date]; 1.0 if none. cum_split_ratio()
    above delegates here.

    Called by:
        - cum_split_ratio() (above)
        - dividend_gross_from_events() (below)
        - adjust_bars_from_events() and adjust_tick_derived_series_from_events()
          (below)
        - 05_labeler.md dead position Case A (r over (D, D+1], and the
          dividend term's factor over (D, that event_date])
        - 04_feature_extractor.md's f_e rescale and dilution_rate's ratio,
          over its reader's corporate_events rows
        - 09_backtest_engine.md, over its preload's corporate_events rows:
          dead position Case A/D (the single-day overnight window D→D+1; the
          dividend term reads dividend_gross_from_events() below — this
          function returns the split ratio component only), the carried
          increment, the carried split normalisation (over the windows of its
          carried-position rule, not the overnight window) and the carried
          dividend
    """
    ...


def dividend_gross_amount(
    ticker:      str,
    event_date:  str,
    value:       float,
    source:      str,
    basis_date:  str,
    config:      dict,
    db_conn:     duckdb.DuckDBPyConnection,
) -> float:
    """
    Gross per-share amount of a 'dividend' corporate_events row, in its
    event_date's basis. SELECTs the ticker's split/reverse_split rows and
    delegates to dividend_gross_from_events() below, which solely owns the
    rule, so the two entry points cannot diverge.

    value: the row's `amount` (DECIMAL(18,8)), taken as float at entry.

    Returns a per-share dividend amount (Numeric Rules), rounded to 8
    decimal places — dividend_gross_from_events()'s result.

    EVERY reader of a 'dividend' amount reads it through this function or
    dividend_gross_from_events() — no caller reads corporate_events.amount
    for a dividend directly.

    Gross, before withholding. The dividend cash rule, stated here once:
    per 'dividend' event, the net amount (gross × (1 −
    corporate_events.dividend_withholding_rate), the rate defaulting to
    0.15) × the shares held at the start of that event_date, in that
    date's basis. Cash-flow readers apply it, each keeping its own event
    window and numeric steps:
        - live_mode_runner.md's dividend cash at close
        - the dead position Case A dividend term (05_labeler.md /
          09_backtest_engine.md)
        - 09_backtest_engine.md's carried dividend
    A price reader applies no withholding:
        - gap_percentile()'s dividend_amount (04_feature_extractor.md in
          training, live_mode_runner.md's today-dividend map in live)
        - populate_precomputed_session_stats()'s dividend_at (below)

    Called by:
        - live_mode_runner.md's position close (dividend cash)
        - live_mode_runner.md's today-dividend map (gap_percentile's
          dividend_amount in live)
        - populate_precomputed_session_stats() (below)
        - metadata_crawler.md's upsert agreement test
    """
    ...


def dividend_gross_from_events(
    ticker:      str,
    event_date:  str,
    value:       float,
    source:      str,
    basis_date:  str,
    config:      dict,
    events_df:   pd.DataFrame,
) -> float:
    """
    The same gross amount from an already-loaded corporate_events frame, for
    05_labeler.md (see split_ratio_from_events() above). value is the row's
    `amount`, taken as float at entry. SOLE OWNER OF THE
    RULE:
        value × split_ratio_from_events(ticker, event_date, basis_date, events_df)
            when source is in config corporate_events.dividend_restated_sources
            (default ['yfinance'] — a source that restates past dividends by
            later splits, basis_date being when the value was fetched)
        value otherwise
    The product is taken in DOUBLE, and the result is rounded to 8 decimal
    places — a per-share dividend amount (Numeric Rules).
    Gross, before withholding, exactly as dividend_gross_amount() above.

    events_df must hold the ticker's split/reverse_split rows through
    basis_date, which may lie well after event_date — a caller passes the
    frame whole rather than pre-filtering it to its own window.

    Called by:
        - dividend_gross_amount() (above)
        - 05_labeler.md dead position Case A
        - 04_feature_extractor.md's gap_percentile dividend_amount lookup,
          over its reader's corporate_events rows
        - 09_backtest_engine.md dead position Case A and its carried
          dividend, over its preload's corporate_events rows
    """
    ...


def adjust_bars_for_corporate_events(
    bars: pd.DataFrame,
    ticker: str,
    reference_date: str,
    db_conn: duckdb.DuckDBPyConnection,
) -> pd.DataFrame:
    """
    Adjust OHLCV bars to reference_date's split scale. SELECTs the ticker's
    corporate_events rows — the security's events under its chain-latest
    symbol — and calls its frame form adjust_bars_from_events() below, which
    owns the computation:

    For each bar with date=D_bar < reference_date:
        ratio = split_ratio_from_events(ticker, D_bar, reference_date, events_df)
        open, high, low, close /= ratio
        volume *= ratio
    Bars with date >= reference_date, or no applicable splits (ratio=1.0):
        no-op — returned unchanged.

    Vectorized via a single merge against corporate_events (a small table) —
    one pass over `bars`, not a per-row loop. For the vast majority of
    tickers (no split in the loaded range), this reduces to a no-op copy.

    Does NOT adjust for dividends — dividends do not create a continuous
    price-level discontinuity the way splits do, so they are out of scope
    for this bar-level correction. Each is a separate scalar correction
    owned by its caller:
        - gap_pct's overnight prev_close comparison —
          02_indicator_calculator.md gap_percentile()
        - dead-position pnl — 05_labeler.md / 09_backtest_engine.md dead
          position Case A/D
        - carried-position pnl — 09_backtest_engine.md's carried dividend

    Called by:
        - live/caching_calculator.md session_start_compute(),
          reference_date = today
    FeatureExtractor uses the frame form below.

    Returns a new DataFrame — does not mutate the input `bars` in place.
    """
    ...


def adjust_bars_from_events(
    bars: pd.DataFrame,
    ticker: str,
    reference_date: str,
    events_df: pd.DataFrame,
) -> pd.DataFrame:
    """
    The frame form of adjust_bars_for_corporate_events(): the same
    adjustment from an already-loaded corporate_events frame, events_df in
    place of db_conn, its ratio split_ratio_from_events() over events_df. It
    takes the rows of the ticker it is passed, the chain-latest symbol.

    Called by:
        - adjust_bars_for_corporate_events() (above)
        - 04_feature_extractor.md extract_batch() Strategy A step 0 and the
          obv/ad per-entry re-adjustment, over its reader's corporate_events
          rows; reference_date = the max date in the loaded bars range for
          that call (today in live, the last date in the ticker's loaded
          range for training)

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

    SELECTs the ticker's corporate_events rows — the security's events under
    its chain-latest symbol — and calls its frame form
    adjust_tick_derived_series_from_events() below, which owns the
    computation:

    For each row with date=D_row in `series`:
        ratio = split_ratio_from_events(ticker, D_row, anchor_date, events_df)
        scale_type == "price"     → value /= ratio
        scale_type == "volume"    → value *= ratio
        scale_type == "invariant" → no change (present for API symmetry, so
                                     ALL tick-derived indicators can be routed
                                     through this same function regardless of
                                     scale_type — see
                                     02_indicator_calculator.md's
                                     Tick-Derived Indicator Scale Sensitivity
                                     registry for which indicator uses which)

    Optimization: checks the split factor over (series.date.min(), anchor_date]
    once first; if 1.0 (the common case — no split in range), skips
    the per-row loop entirely and returns the input unchanged. Same cost
    profile as adjust_bars_for_corporate_events().

    Does not mutate the input `series` — always returns a new copy. Never
    touches tick_10 — operates only on an already-computed indicator output
    (whether sourced from tick_bar_aggregates or the Tier-2 on-the-fly
    fallback in load_tick_bar_aggregates_with_history()).

    FeatureExtractor's tick-derived dispatch path uses the frame form below.
    """
    ...


def adjust_tick_derived_series_from_events(
    series: pd.DataFrame,
    ticker: str,
    anchor_date: str,
    scale_type: str,
    events_df: pd.DataFrame,
) -> pd.DataFrame:
    """
    The frame form of adjust_tick_derived_series_for_corporate_events(): the
    same correction from an already-loaded corporate_events frame, events_df
    in place of db_conn, its ratio split_ratio_from_events() over events_df.
    It takes the rows of the ticker it is passed, the chain-latest symbol.

    Called by:
        - adjust_tick_derived_series_for_corporate_events() (above)
        - 04_feature_extractor.md extract_batch()'s tick-derived dispatch
          path (a2'), anchor_date = entry_date (per entry point), over its
          reader's corporate_events rows
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
    momentum_n: int,
    db_conn: duckdb.DuckDBPyConnection,
) -> pd.DataFrame:
    """
    Compute bar-level tick-derived indicator values for a ticker over a date
    range, directly from raw tick_10 + ohlcv_1min.

    momentum_n: the configured lee_ready.momentum_n, passed to its lee_ready()
    call; no default — its callers pass it. It acts only on
    tick_direction_momentum, which tick_bar_aggregates does not store, so it
    enters neither tick_bar_aggregates' key nor precomputed_session_stats'
    config_hash.

    Thin wrapper only — internally calls IndicatorCalculator's own
    avg_vol_per_tick() / tpm() / lee_ready() / etc. (whichever `indicators`
    lists) rather than reimplementing their formulas, so a value computed
    here can never diverge from a value computed the same way elsewhere
    (Strategy A during training, or the Tier-2 fallback below).

    Output: long/EAV format matching tick_bar_aggregates — columns
    [ticker, date, hour, indicator, value]. Rows with fewer than the
    indicator's required tick-bundle count (e.g. <2 bundles for a
    delta/interval-based indicator) get value=NULL rather than being omitted,
    so downstream reindexing against the bars' own index still finds a row
    to report NaN from.

    Called by:
        - the writers of tick_bar_aggregates, each writing under the current
          data version (db_schema.md's tick_bar_aggregates):
          - metadata_crawler.md's evening_tick_bar_aggregates stage;
          - migration_tool.md's backfill;
          - populate_precomputed_session_stats() (below), for a (ticker,
            date) without current-version rows, for every registered
            tick-derived indicator, INSERT OR IGNORE in its stats
            transaction — its compute reading through the symbology layer
            and each written row stored under its origin symbol;
        - load_tick_bar_aggregates_with_history()'s Tier 2 (below; result
          used in-memory only, writing nothing)
    """
    ...


def load_tick_bar_aggregates_with_history(
    ticker: str,
    indicators: list[str],
    lookback_start_date: str,
    today_date: str,
    momentum_n: int,
    db_conn: duckdb.DuckDBPyConnection,
) -> pd.DataFrame:
    """
    Load historical tick-derived indicator values for live mode's Eager Pool,
    for indicators configured with precalculate_bars: "lookback"
    (02_indicator_calculator.md). Returns wide format (one column per
    indicator) for direct use seeding CachingIndicatorCalculator's Layer 2.

    3-tier resolution per (ticker, date) in [lookback_start_date, today_date),
    through the symbology layer for today's symbol — each segment's symbol
    read and computed under its own name:
        Tier 1 — tick_bar_aggregates has rows of the current data version for
            this date: read directly. Rows of another data version are never
            read.
        Tier 2 — no current-version rows, but ticker_data_coverage.has_tick is
            True for this date (the evening_tick_bar_aggregates stage hasn't
            reached it yet, the ticker is newly listed, or the data version
            changed): call compute_tick_bar_aggregates() for this date on the
            fly. Result used for this session only — writing nothing
            (db_schema.md's tick_bar_aggregates names the writers).
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

    momentum_n: the configured lee_ready.momentum_n, without a default; it
    reaches Tier 2's compute_tick_bar_aggregates() call and does not affect
    the returned values.

    The returned columns carry tick_bar_aggregates' stored indicator names —
    lee_ready's is buyer_initiated_ratio (db_schema.md's tick_bar_aggregates).
    """
    ...
```

```python
def estimate_historical_meta(
    ticker: str,
    date: str,
    field: str,
    closes: dict[str, str],
    lookback_days: int,
    db_conn: duckdb.DuckDBPyConnection,
) -> float | int | None:
    """
    Derive a point-in-time stock_meta field value for (ticker, D = date) —
    market_cap when the prior-day row lacks it; shares_outstanding always,
    its first tier the prior-day row — from values known before D's session
    (data_boundary.md's "Point-in-time correctness for stock_meta"). Its
    sources:
        - stock_meta rows dated before D;
        - fundamentals_quarterly;
        - ohlcv_1min;
        - corporate_events.
    The prior-day row: the stock_meta row dated the trading day
    prior_trading_days(D, 1, closes) returns (db_schema.md stock_meta — a
    row's date is its crawl date, and it serves the trading days after it).
    It reads through the symbology layer (Ticker Identity Utilities).

    closes: the {date: session_close} map, read by:
        - the prior-day row;
        - last_bar() of a date, for a regular-session close — tier 1's
          implied shares and market_cap's close;
        - market_cap's close search.
    lookback_days: indicators.lookback_days; market_cap's close search alone
    reads it, shares_outstanding not.

    The instrument rule: a non-common security_type — warrant, unit, right,
    preferred, debt — resolves shares_outstanding and market_cap none in
    every tier, including the prior-day row's values, tier 1's implied shares
    and the estimates. security_type adr skips the fundamentals tier; its
    other tiers stand. security_type is read as sector is: the chain-latest
    symbol's latest row with security_type set, whatever the date.

    Supported fields and derivation:
        shares_outstanding — the tiers, in order:
            1. The prior-day row's shares_outstanding through
               split_basis_value() (below), the tier 2 value as anchor. With
               that value absent and the prior-day row's market_cap set:
               market_cap ÷ the last regular-session close of the row's date,
               only when no split or reverse_split lies in (the row's date,
               D]. Otherwise none.
            2. fundamentals_quarterly tier: most recent row with
               filed_date < date (see data_boundary.md — never
               fiscal_period_end) and metric="shares_outstanding", then
               adjusted for any split between that row's fiscal_period_end —
               the XBRL fact's instant date — and date via cum_split_ratio()
               (the value is as of its instant date, so splits after that
               date must still be applied). A non-positive value is skipped.
               Fundamentals rows are stored and read under the ticker's
               current ticker_cik_map pairing, applied to every date; a
               ticker without one has none.
               Uses get_ttm_value()-adjacent as-of lookup logic (see
               Fundamentals Utilities below), not a TTM sum itself since
               shares_outstanding is an instant, not a duration, metric.
            3. The most recent stock_meta row dated before D whose
               shares_outstanding is set, its value as is, only when no split
               or reverse_split lies in (its date, D]; otherwise none.
            (buyback/issuance activity after a row's date is NOT captured by
            tiers 1 and 3; tier 2 closes this gap for tickers/dates
            fundamentals_quarterly covers)
            Limits:
            - an adr on a date with a split or reverse_split in (the row's
              date, D] resolves shares none;
            - a multi-class issuer's per-class shares are not available; the
              company-level value stands.

        market_cap:
            close = the last regular-session bar's close on the latest of the
                    trading days prior_trading_days(D, lookback_days, closes)
                    returns that has one
            shares = estimate_historical_meta(ticker, D, "shares_outstanding",
                                              closes, lookback_days, db_conn)
            return close ÷ cum_split_ratio(ticker, the close's date, D) × shares
            No such close, or no shares → None.

        price_52h / price_52l:
            Not supported — prior-session baselines, precomputed_session_stats'
            day-level metrics (populate_precomputed_session_stats() below);
            the caller fills meta's price fields (run_preprocess.md Step 6,
            live_mode_runner.md Step 3).

        sector, security_type:
            Not supported — no derivation path. Callers use the chain-latest
            symbol's latest stock_meta row with the field set, whatever the
            date; do not call this function with field="sector" or
            field="security_type".

    Called by:
        - 04_feature_extractor.md Meta Features resolution (training, per
          entry date) and by run_preprocess.md Step 6 (building the date-keyed
          `ticker_meta` dict)
        - 01_entry_detection.md filter E (condition E, turnover rate) — same
          shares_outstanding resolution, shared rather than duplicated
        - metadata_crawler.md's Detection-Gap Analysis — the
          shares_outstanding passed to detect() for the date
        - live_mode_runner.md Step 3 — market_cap when the prior-day row
          lacks it; shares_outstanding always, its first tier the prior-day
          row
        - migration_tool.md's Security Identity Audit — its Q2 shares none

    Returns None if derivation itself is impossible — the caller takes a NaN
    feature; condition E false.
    """
    ...


def split_basis_value(
    value: float,
    anchor: float | None,
    ratio: float,
    field: str,
) -> float | None:
    """
    A stock_meta value read for a date D from a row dated before it, on D's
    split basis.

    - ratio: cum_split_ratio(row date, D); with ratio 1.0 the value is
      returned as is.
    - With anchor None, returns None.
    - The pre-split expectation: anchor × ratio for price_52h and price_52l;
      anchor ÷ ratio for shares_outstanding.
    - |ln(value ÷ anchor)| ≤ |ln(value ÷ pre-split expectation)| → the value
      as is; otherwise the value converted: ÷ ratio for a price field,
      × ratio for shares_outstanding.

    Callers:
        - estimate_historical_meta()'s shares_outstanding tier 1, anchor the
          tier 2 value;
        - live_mode_runner.md Step 3's price fields under
          live_mode.price_extremes_source 'vendor', anchor today's group's
          metric.
    """
    ...
```

---

### Fundamentals Utilities

Fundamentals rows are stored and read under the ticker's current
ticker_cik_map pairing, applied to every date; a ticker without one has none.
A row whose unit is NULL is not read; migration_tool.md's fundamentals
backfill, re-run, repopulates such rows.

```python
def resolve_xbrl_tag_value(
    companyfacts_json: dict,
    tag_priority_list: list[str],
) -> list[dict] | None:
    """
    Given a company's raw SEC EDGAR companyfacts JSON and a metric's ordered
    tag priority list (configs/xbrl_tag_map.json's list for that metric, its
    entries taxonomy-qualified — the companyfacts taxonomy key with the
    concept), return the metric's values per filing, each with its unit, or
    None if none of the tags are present at all for this company.

    The merge: per filing (accn), the metric's values come from the
    highest-priority tag holding facts in that filing; within it, per period:
        - shares_outstanding: the 'shares' facts;
        - another metric: the USD-denominated unit when present; else, when
          every fact of the period carries the same unit, that unit; else
          not stored.
    The stored row's unit is the companyfacts unit string of the fact
    (db_schema.md's fundamentals_quarterly).

    configs/xbrl_tag_map.json's default priority:
        - shares_outstanding: dei:EntityCommonStockSharesOutstanding, then
          us-gaap:CommonStockSharesOutstanding;
        - every other metric: its us-gaap tags, then its ifrs-full tags, each
          in this order:
          - net_income: us-gaap:NetIncomeLoss, us-gaap:ProfitLoss, then
            ifrs-full:ProfitLossAttributableToOwnersOfParent,
            ifrs-full:ProfitLoss;
          - revenue: us-gaap:Revenues,
            us-gaap:RevenueFromContractWithCustomerExcludingAssessedTax,
            us-gaap:RevenueFromContractWithCustomerIncludingAssessedTax,
            us-gaap:SalesRevenueNet, then ifrs-full:Revenue;
          - stockholders_equity: us-gaap:StockholdersEquity,
            us-gaap:StockholdersEquityIncludingPortionAttributableToNoncontrollingInterest,
            then ifrs-full:EquityAttributableToOwnersOfParent,
            ifrs-full:Equity;
          - cash: us-gaap:CashAndCashEquivalentsAtCarryingValue, us-gaap:Cash,
            then ifrs-full:CashAndCashEquivalents;
          - total_liabilities: us-gaap:Liabilities, then ifrs-full:Liabilities;
          - total_assets: us-gaap:Assets, then ifrs-full:Assets;
          - operating_income: us-gaap:OperatingIncomeLoss, then
            ifrs-full:ProfitLossFromOperatingActivities;
          - operating_cash_flow:
            us-gaap:NetCashProvidedByUsedInOperatingActivities,
            us-gaap:NetCashProvidedByUsedInOperatingActivitiesContinuingOperations,
            then ifrs-full:CashFlowsFromUsedInOperatingActivities;
          - eps_diluted: us-gaap:EarningsPerShareDiluted,
            us-gaap:EarningsPerShareBasicAndDiluted, then
            ifrs-full:DilutedEarningsLossPerShare.

    This resolves cross-filer tag fragmentation (e.g. revenue reported under
    RevenueFromContractWithCustomerExcludingAssessedTax by one filer,
    Revenues by another) — configs/xbrl_tag_map.json is the source of each
    metric's priority list, keyed by metric name, extendable by editing that
    JSON alone (no code change to add a new fallback tag for an existing
    metric).

    Does not itself apply any as-of filtering — returns whatever facts exist
    for the resolved tag; callers (estimate_historical_meta(),
    get_ttm_value()) apply the filed_date < date filter themselves.

    An instant fact (shares_outstanding and the other period_months NULL
    metrics) has no period end of its own: its row takes the fact's instant
    date as fiscal_period_end (db_schema.md's fundamentals_quarterly).

    """
    ...


def get_ttm_value(
    ticker: str,
    date: str,
    metric: str,
    rows: pd.DataFrame,
) -> tuple[float, str] | None:
    """
    Trailing-twelve-month value for a duration-type fundamentals_quarterly
    metric (e.g. "net_income", "revenue"), as of `date`, with its unit.

    rows: a fundamentals_quarterly frame — FeatureExtractor's reader's rows —
    in place of db_conn; it takes the rows of the ticker it is passed, as
    adjust_bars_from_events() takes its ticker's events. FeatureExtractor's
    instant as-of lookups and dilution_rate's shares_outstanding rows read
    the same rows.

    As-of filtering: only rows filed before date (filed_date < date) are
    eligible — see
    data_boundary.md's point-in-time principle (never fiscal_period_end).

    Resolution:
        Sum the four most recent eligible quarters' values for this metric
        (period_months=3 rows), ending at or before `date`.
        Q4 does not exist as a filed row on its own — where the most recent
        eligible quarter would be Q4, derive it as
        FY_value - (Q1 + Q2 + Q3) using the matching period_months=12 row
        and the three preceding period_months=3 rows for the same fiscal year.

    Returns (value, unit): the TTM and Q4 components share one unit, else
    None.

    Annual fallback: when the quarterly TTM is unavailable — fewer than 4
    eligible quarters, or the FY+3Q components for a Q4 derivation — the
    latest FY value (period_months 12) filed before date whose fiscal_period_end
    lies within fundamentals.annual_fallback_max_age_months of date; else
    None — caller treats this the same as any other unavailable fundamentals
    value (NaN feature).

    Not applicable to instant-type metrics (shares_outstanding,
    stockholders_equity, total_assets, etc. — period_months=NULL) — those are
    point-in-time by nature and use a plain as-of most-recent-filed lookup
    instead, not a TTM sum (see estimate_historical_meta()'s shares_outstanding
    tier 2 for that pattern); those lookups also return (value, unit).
    """
    ...


def fx_per_usd(
    currency: str,
    entry_date: str,
    closes: dict[str, str],
    rates: pd.DataFrame,
    max_staleness_days: int,
) -> float | None:
    """
    Units of currency per USD for a fundamentals ratio on entry_date, from
    fx_rates rows (db_schema.md's fx_rates):
        - 'USD' → 1.0;
        - otherwise the per_usd of the latest rate dated on or before the
          trading day prior_trading_days(entry_date, 1, closes) returns, no
          more than max_staleness_days calendar days before it — never a
          later rate;
        - otherwise None.
    max_staleness_days: config fx_rates.max_staleness_days.

    A monetary fact meets market_cap in its own currency: each of these is
    the value ÷ (market_cap × fx_per_usd of the value's currency), None → NaN:
        - earnings_yield;
        - book_to_price;
        - sales_to_price;
        - cash_to_mcap;
        - debt_to_mcap.
    dilution_rate is unchanged; an annual-only filer has none.
    """
    ...
```

---

### REFERENCE_SESSION Utilities

```python
def populate_precomputed_session_stats(
    db_conn: duckdb.DuckDBPyConnection,
    config: dict,
    dates: list[str],
    tickers: list[str] | None = None,
) -> int:
    """
    The producer of precomputed_session_stats (db_schema.md's Derived
    tables). For the given as_of_dates and tickers (None = every ticker with
    ohlcv_1min data), under `config`, per (ticker, as_of_date) group:
        - an absent group is computed;
        - a stale group (db_schema.md's Derived tables, Stale) is
          recomputed, its rows replaced in one transaction;
        - a current group is skipped.

    Config Keys Used:
        - indicators.reference_session.n_sessions
        - indicators.reference_session.price_extremes_sessions
        - corporate_events.dividend_restated_sources
    Their config_hash (Config Hashing, above), with code_versions holding the
    data version, leads the rows' key. The function writes that config's
    derived_configs row with INSERT OR IGNORE in the transaction of its rows,
    before them.

    Loads the boundary maps at its own start, under Session Boundary
    Derivations' WHO LOADS rule: it owns db_conn and loads them itself,
    whichever caller reaches it:
        - metadata_crawler.md's evening session-stats stage;
        - migration_tool.md;
        - live_mode_runner.md;
        - run_preprocess.md's batch.
    gap_pct's prev_close anchors on last_bar(D-k) for each prior
    session, and the slot rule below calls session_phase(), so the maps are
    needed here and not merely passed through.

    For each group:
        Compute per-bar average over the prior n_sessions trading sessions,
        n_sessions = config["indicators"]["reference_session"]["n_sessions"].

    Slots: a prior session D-k contributes to slot H only when
    session_phase(D-k, H, closes, ends) is 'premarket' or 'regular';
    after-hours slots, an early-close session's included, contribute nothing.
    The tick-derived baselines follow the same slot rule.

    Metrics derived from ohlcv_1min:
        rvol_baseline      : cumulative volume per HHMMSS slot, accumulated
                             from PREMARKET_OPEN
        rel_dvol_baseline  : cumulative dollar volume per HHMMSS slot,
                             accumulated from PREMARKET_OPEN
        intra_vol_baseline : per-bar volume per HHMMSS slot
        intra_return_baseline: per-bar (close/prev_close - 1) per HHMMSS slot
        gap_pct_mean/std   : (today_regular_open - prev_close) / prev_close
                             stored as mean and std (hour='000000')
        price_52h / price_52l : for (ticker, D) at hour '000000' — the
                             highest high / the lowest low of the
                             regular-session bars of the
                             indicators.reference_session.price_extremes_sessions
                             trading days before D, D excluded, each bar's
                             prices split-adjusted to D through
                             cum_split_ratio(); avg_value the extreme,
                             std_value NULL; count the trading days used —
                             with fewer days available, those days; with
                             none, no row

    Metrics derived from tick_10, via tick_bar_aggregates (see db_schema.md —
    this function reads the same table Strategy A's live-mode fallback does,
    rather than re-deriving from raw tick_10 independently), through the
    symbology layer. A (ticker, date) without current-data-version rows is
    resolved through compute_tick_bar_aggregates() for every registered
    tick-derived indicator, its compute reading through the layer, the rows
    written with INSERT OR IGNORE under the current data version in its stats
    transaction, each under its origin symbol:
        buy_ratio_baseline               : per-bar buyer_initiated_ratio per HHMMSS slot
        intra_tpm_baseline                : per-bar ticks-per-minute per HHMMSS slot
        intra_avg_vol_per_tick_baseline    : per-bar avg_vol_per_tick per HHMMSS slot

    Corporate-event and ticker-rename awareness: prior-session raw values are
    read through the symbology layer — load_ohlcv_with_history(), i.e.
    load_ticker_rows() on ohlcv_1min (so a ticker rename within the prior
    n_sessions window does not silently drop history) — then adjusted per
    corporate_events before aggregation (see D. below). ticker_history is
    small and pre-loaded once as an in-memory dict at the start of this
    function's execution — no per-ticker DB query added to the inner loop for
    tickers with no rename (the vast majority).

    Halt handling per metric type:

    A. Cumulative metrics (rvol_baseline, rel_dvol_baseline):
        A minute without a bar — halted or not traded — contributes nothing
        to that session's cumsum (a halt naturally interrupts volume flow —
        this is correct behavior).
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

    B. Per-bar metrics:
        - intra_vol_baseline;
        - intra_return_baseline;
        - intra_tpm_baseline;
        - buy_ratio_baseline;
        - intra_avg_vol_per_tick_baseline.
        A per-minute baseline at slot H averages only the sessions that have
        a bar at H:
            count[H] reflects only those sessions.
            Other hours of a session without a bar at H are unaffected.
        intra_return_baseline's prev_close at slot H is the close of the
        last existing bar before H.

    C. Day-level metrics (gap_pct_mean, gap_pct_std), stored at hour='000000':
        gap_pct = (today_regular_open - prev_close) / prev_close

        prev_close determination (for each prior session D-k):
            - the close of the last existing bar at or before last_bar(D-k)
              in D-k's regular session;
            - no such bar: exclude the session.
            PER-DATE, not 155900. When D-k was an early close, 155900 does not
            exist; anchoring on last_bar(D-k) reads the real final bar
            directly.

        today_regular_open determination (for each prior session D-k):
            - the open of D-k's first regular-session bar (session_phase()
              'regular'), as 02_indicator_calculator.md's gap_percentile()
              takes today_regular_open;
            - a D-k with no regular-session bar is excluded.

        Sessions excluded from either determination reduce count.

    D. Corporate-event adjustment (applied to prior-session raw values BEFORE
       A/B/C aggregation — not a post-hoc correction of the stored mean/std,
       since the individual per-session values are not retained after
       aggregation and cannot be un-averaged later):

        For ticker T, as_of_date D, prior session D-k — the security's events
        read under its chain-latest symbol:
            cum_ratio(D-k) = cum_split_ratio(T, D-k, D, db_conn)
                             (product of split/reverse_split 'ratio' with
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
                        where split_at(D-k) = 'split'/'reverse_split' ratio
                            with event_date=D-k (1.0 if none)
                        dividend_at(D-k) = dividend_gross_amount() of the
                            'dividend' row with event_date=D-k (0.0 if none)
                    gap_pct(D-k) = (today_regular_open(D-k) - adjusted_prev_close(D-k))
                                   / adjusted_prev_close(D-k)
                    (applied using the prev_close and today_regular_open
                    values already determined in C above — this adjustment is
                    independent of, and applied after, C's determinations)

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

    Stores: avg_value, std_value, count per (config_hash, ticker, as_of_date,
    hour, metric), each group's rows carrying source_seq — the highest
    source_changes seq when its computation started (db_schema.md's Derived
    tables, Mark).
    count may differ by hour slot within the same ticker/as_of_date (per-bar metrics).
    Delta smoothing NOT applied here — applied at load time via load_session_stats()
    or build_session_stats_dict().
    Re-running computes absent and stale groups only; a current group is
    never rewritten.
    Returns: number of rows written.
    """
    ...


def load_session_stats(
    db_conn: duckdb.DuckDBPyConnection,
    ticker: str,
    as_of_date: str,
    config_hash: str,
    delta_minutes: int,
    closes: dict[str, str],
    ends: dict[str, str],
) -> dict:
    """
    Load and smooth precomputed_session_stats for a ticker on a given date.

    Steps:
        1. Query precomputed_session_stats WHERE config_hash=?, ticker=?, as_of_date=?
        2. Keeps the rows whose slot H has session_phase(as_of_date, H, closes, ends)
           'premarket' or 'regular', and the day-level rows (hour '000000').
        3. Apply delta_minutes rolling average per metric:
               For each hour H, smoothed_value[H] =
                   mean(avg_value for hours in [H-delta, H+delta]), the window
                   clipped to H's phase
               Uses hhmmss_to_minutes() for window arithmetic.
        4. Return dict: {metric: {hour: smoothed_avg_value}}

    Returns empty dict if no rows found (e.g. as_of_date before dataset start).
    """
    ...


def build_session_stats_dict(
    raw_df: pd.DataFrame,
    delta_minutes: int,
    closes: dict[str, str],
    ends: dict[str, str],
) -> dict:
    """
    Convert a bulk-loaded precomputed_session_stats DataFrame into a nested dict.

    Input raw_df columns: ticker, as_of_date, hour, metric, avg_value, std_value, count
    (typically loaded with a WHERE config_hash = ? filter applied before calling this function)

    Output format:
        {as_of_date: {ticker: {metric: {hour: smoothed_avg_value}}}}

    Processing:
        1. Keeps the rows whose slot H has session_phase(as_of_date, H, closes, ends)
           'premarket' or 'regular', and the day-level rows (hour '000000').
        2. For each (as_of_date, ticker, metric):
               Apply delta_minutes smoothing over the time dimension:
               smoothed[H] = mean(avg_value for hours within ±delta_minutes of H),
               the window clipped to H's phase
               Uses hhmmss_to_minutes() for window arithmetic.
        3. Build nested dict structure.

    Used by:
        - run_preprocess.py Step 1 (training batch load, the chunk dates):
              raw_df = SELECT * FROM precomputed_session_stats WHERE config_hash = ?
                       AND as_of_date IN <chunk dates> AND ticker IN <batch tickers>
              result: {as_of_date: {ticker: {metric: {hour: value}}}}
              dispatch: result[date][ticker] per entry point

        - live_mode_runner.md's Session Lifecycle Step 2 (live bulk load, single date):
              raw_df = SELECT * FROM precomputed_session_stats WHERE as_of_date = ? AND config_hash = ?
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
    start_date: str,
    end_date: str,
    vol_percentile: float,
    window_days: int,
    vol_metric: str,
) -> dict[str, float]:
    """
    Compute holdout dates based on rolling volatility percentile, over
    [start_date, end_date] — the caller's pinned date range.
    Uses regular session bars only (093000~last_bar(date)) from ohlcv_1min.

    Loads the boundary maps at its own start, under Session Boundary
    Derivations' WHO LOADS rule: it owns db_conn and pipeline_optimizer.md
    preloads nothing. It reads ohlcv_1min rather than precomputed_session_stats,
    so nothing another consumer already holds would serve here.

    vol_metric = "avg_intraday_range":
        per date = mean of (high - low) / open across all tickers

    window_days and vol_metric have no default: the caller passes the
    configured values (vol_window_days, vol_metric).

    Rolling window: window_days calendar days.
    Holdout: dates with rolling_vol >= quantile(vol_percentile).
    Example: vol_percentile=0.80 → top 20% most volatile dates excluded.

    Returns {'YYYYMMDD': rolling_vol} for the dates to exclude from training;
    the keys are the holdout set. These dates are removed from labeled_df
    BEFORE outer fold generation:
        remaining_df = full_labeled_df[~full_labeled_df["date"].isin(holdout_dates)]
    The caller, pipeline_optimizer.md, computes it once per work over the
    work's pinned date range and records the returned mapping in
    regime_holdout_dates (db_schema.md).
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

    Args:
        best_configs: list of hyperparameter dicts, one per outer fold
        search_space: valid grid values per parameter (for rounding)

    Returns:
        consensus hyperparameter dict
    """
    ...


def temporal_split_simple(
    labeled_df:   pd.DataFrame,
    val_fraction: float,
    embargo_days: int,
) -> tuple[pd.DataFrame, pd.DataFrame]:
    """
    Simple date-based train/val split without a test set. val_fraction and
    embargo_days have no default: the caller passes the configured values.

    Order of operations:
        1. Sort by date ascending
        2. Compute split boundary:
             val_dates = last val_fraction of unique dates
             train_dates = remaining (with embargo_days gap to val)
        3. Return (train_df, val_df) without balancing

    No downsampling applied — caller is responsible if needed.
    Returns (train_df, val_df) as plain DataFrames.
    """
    ...


def regime_gate(verdict_row: dict | None, config: dict) -> dict:
    """
    The regime holdout gate's decision. Returns:
        - verdict: the regime_holdout_verdict row's verdict; with no row,
          optimizer.regime_holdout.gate.missing_result;
        - blocked: optimizer.regime_holdout.gate.enabled and the verdict
          'fail'.
    A session's participation is blocked by the gate at
    live_mode_runner.md's Session Lifecycle Step 7 alone; model_candidates'
    status at writing and model_deploy.md's decisions also read the verdict.

    Callers:
        - live_mode_runner.md's Step 7;
        - model_deploy.md;
        - pipeline_optimizer.md's 'exploitation' and 'refit' closing steps
          (model_candidates.regime_gate_blocked, under the work's config).
    """
    ...


def live_late_entry_theta(
    run_id: str,
    quantiles: list[float],
    min_entries: int,
    conn,
) -> dict:
    """
    The live derivation of θ:
        - population: inference_log rows of run_id on the sessions of run_id,
          taken whole in ascending date until their count first reaches
          min_entries, each row having:
          - event 'signal_fired';
          - entry_path 'bar_close';
          - gate_result 'submitted' or 'cap_inert';
        - per row: late_entry_expected_return(row, p_ref, p_now = p_ref)
          under the config the live gate uses;
        - per q: the q-th quantile by linear interpolation, with the
          population's count;
        - a count short of min_entries → the count alone, no θ.
    Its quantiles: the θ quantile set and execution.late_entry_theta_q when
    set.
    """
    ...
```

---

### Bootstrap Confidence Interval

```python
def bootstrap_ci(
    values:       Sequence,
    statistic:    Callable[[np.ndarray], float],
    sample_size:  int,
    n_resamples:  int,
    confidence:   float,
    seed:         int,
) -> tuple[float, float]:
    """
    Percentile bootstrap CI of `statistic` over resamples of `sample_size`
    drawn with replacement from `values`. Returns (ci_low, ci_high).
    n_resamples, confidence and seed come from config optimizer.bootstrap
    (defaults 1000, 0.95, 0) — seeded so a rerun reproduces the interval.

    The resampling unit is the caller's: `values` is a sequence of unit
    rows, one per unit, and `statistic` receives the resampled rows. A caller
    passing one value per unit gets the result it got before rows existed.

    Called by:
        - pipeline_optimizer.md's Regime Holdout Report — values = per-trade
          pnl_pct of the outer_validation baseline runs, sample_size = the
          regime total_trades; statistic = share of pnl_pct > 0
          (winning_rate) or mean (avg_pnl_pct)
        - pipeline_optimizer.md's champion comparison — values = the units'
          values; statistic mean; sample_size = the unit count
        - divergence_check() (below)
    """
    ...


def divergence_check(observations, reference) -> dict:
    """
    A day-block bootstrap of the winning rate, through bootstrap_ci():
        - an observation unit is a trading day; a reference unit is (outer
          fold, trading date);
        - a unit's row is (winning trades, trades);
        - the statistic is the pooled winning rate of the resampled units;
        - sample_size is the count of observation days with a trade;
        - n_resamples, confidence and seed are optimizer.bootstrap's.
    The reference is the 'outer_validation' runs with execution_variant NULL
    of the target model's work. The metric is the winning rate alone.

    Verdicts, decided in this order:
        - 'insufficient_sample': observation days with a trade <
          optimizer.refresh.divergence.min_days, or trades <
          optimizer.refresh.divergence.min_trades;
        - 'adverse': the observed winning rate < ci_low;
        - 'within': otherwise.
    The result also carries in_ci and ci_position: 'below' | 'inside' |
    'above'.

    These compute through it:
        - health_report.md's finding 6;
        - shadow_retraining.md's divergence retraining trigger;
        - probation after a deployment.
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
    returning halt state. So this cannot be served through trading_api.md.
    The source is the Nasdaq Trader trade-halt feed, and this function does
    not fetch it. live_mode_runner.md's halt-status poller holds it and keeps
    an in-process snapshot; in a process without the runner, the poller's
    single-fetch entry fills that process's snapshot with one fetch. That
    snapshot is what this function reads.

    Takes no URL and no chunk size, and no config key backs it. The former
    `trading_api_url` reuse went with that key. The poll interval and the
    freshness ceiling are the poller's config keys, not this function's,
    because a market-wide feed is fetched once per interval rather than per
    call. The feed URL is not a config key at all: it is stated in the
    poller's Source paragraph, as every external source URL in this project
    is. The poll floor is grounded in one named publisher's terms, and a
    configurable URL would decouple the floor from what it bounds.

    Returns {ticker: is_halted} for every requested ticker. The source is a
    market-wide feed of OUTSTANDING halts, so a fresh snapshot is a complete
    observation of what is halted: a requested ticker absent from it is
    not-halted, and the returned dict says so rather than omitting the key.
    That reading belongs to this source's shape. It does NOT generalize: a
    per-ticker source, which this contract was first written against, cannot
    distinguish "not halted" from "not recognized", and absence there would
    stay unknown. Feed symbols are resolved to this project's tickers by
    match_vendor_symbol() against the 'tradable' index before the requested
    tickers are matched.

    Returns None (not an exception) when no snapshot is available — the
    poller has not produced one yet, or the one it holds is older than its
    freshness ceiling, whose config key is a multiple of the poll interval.
    A stale snapshot is NOT returned quietly: silently serving one would let
    a dead poller read as a market with nothing halted. This function does
    not decide the degraded-mode behavior for that case; each call site
    interprets None per its own context, since the call sites have different
    fallback signals available:
        - live_mode_runner.md's Position Manager Loop: falls back to the
          existing tick-rate heuristic per ticker.
        - the premarket passes: no live-tick fallback exists there, and the
          heuristic is disabled outside the regular session anyway, so an
          unresolved ticker is passed to
          metadata_crawler.md's check_corporate_event_anomaly() and treated
          as not-halted (conservative direction — see that function for why).

    This function's contract — bulk ticker list in, {ticker: bool} | None
    out — is fixed, so every call site is designed against it.

    Called by:
        - live_mode_runner.md's Position Manager Loop, Step 1 (halt check,
          source-primary + tick-rate fallback; was Step 1a before the former
          bar-fetch Step 1 was removed)
        - metadata_crawler.md's 04:00 --premarket-open pass, after the
          halt-status poller's single-fetch entry fills its snapshot
        - metadata_crawler.md's manual --premarket-recheck, after the same
          single fetch
        - live_mode_runner.md's In-Process Premarket Recheck (09:20)
      The premarket callers pass the result INTO
      check_corporate_event_anomaly(); that function takes halt_status as an
      already-fetched argument and calls nothing here itself.
    """
    ...
```

---

### Numeric Rules

Binding on every numeric value the system stores, computes or decides on.

- **Ledger values:** quantities, per-share prices and cash amounts in
  `live_order_requests`, `live_fills`, `live_orders`, `live_positions`,
  `live_position_adjustments`, the view `live_position_state` and
  `trade_log` (db_schema.md).
- **Kinds:**
  - GRID VALUES — prices, amounts, quantities, broker-reported rates, config
    values and fitted parameters after V1, and values formed from these by
    +, −, ×, ÷ (returns, thresholds, totals, quantity × rate);
  - ESTIMATES — indicator outputs, series statistics, smoothing,
    interpolation, model outputs.
- **Types, by source:**
  - stored `DECIMAL` — values from the broker API (its responses, the JSON
    dump loading `ohlcv_1min`, `tick_10` and `tick_10_tail`, and
    `bid_ask_snapshots`), and
    amounts, prices and quantities from yfinance, SEC companyfacts and
    investing.com;
  - stored `DOUBLE` — `a:b` split ratios and the factors derived from them;
  - computed values follow these rules, so these stay `DOUBLE`:
    - the predicted price columns (`predicted_fill_price`,
      `predicted_weighted_avg_exit_price`);
    - `pnl_pct`;
    - `slippage_pct`;
    - `experiment_log` totals;
    - `precomputed_session_stats`;
    - `tick_bar_aggregates`;
    - `indicator_cache`.
- **Widths** (db_schema.md):
  - quantities, per-share prices and broker rates `DECIMAL(18,6)`;
  - cash amounts and Σ quantity × price `DECIMAL(28,6)`;
  - per-share dividend amounts `DECIMAL(18,8)`.
- **Rounding:** to the scale of the value's width, ties away from zero —
  DuckDB `CAST` and `round`; Python `ROUND_HALF_UP`, never the `Decimal`
  default:
  - 8 decimal places for a per-share dividend amount;
  - 6 for every other value.
- **Per-share dividend amounts:**
  - `corporate_events.amount`;
  - `corporate_event_conflicts.kept_amount` and `other_amount`;
  - the results of `dividend_gross_amount()` and
    `dividend_gross_from_events()` — value × split ratio taken in `DOUBLE`,
    rounded to 8;
  - the per-share net amount in live_mode_runner.md's dividend cash,
    converted by V1 at 8 decimal places.
- **Arithmetic:**
  - ledger values in Python `Decimal`, division in Python `Decimal` then
    rounded;
  - elsewhere either kind may be computed in float.
- **DuckDB `DECIMAL`:**
  - one multiplication operand is widened to `DECIMAL(38,6)` first —
    `DECIMAL(18,6) × DECIMAL(18,6)` is `DECIMAL(18,12)` and overflows above
    six integer digits;
  - no `DECIMAL` division in DuckDB, where `DECIMAL ÷ DECIMAL` returns
    `DOUBLE`.
- **Split ratio:**
  - `corporate_events.ratio` is `DOUBLE`;
  - value × ratio and value ÷ ratio are taken in `DOUBLE` and the result
    rounded;
  - the ratio itself is not rounded and never enters `Decimal` arithmetic on
    its own.
- **Decisions:**
  - comparison, equality and dedup, floor/ceil/int, and sign on grid values
    return what `Decimal` evaluation returns, live and offline, float
    operands entering by V1;
  - admitted equivalent: grid values compared in float against a threshold
    computed in `Decimal` and rounded to the 6-decimal grid — up for ≥ and <,
    down for ≤ and >;
  - decisions involving an estimate run in float through one implementation
    shared by live and offline.
- **Grid-value decision sites** (execution_common.md, 05_labeler.md,
  09_backtest_engine.md, live_mode_runner.md):
  - tp/sl breach tests (`track_price_breach()`, live breach_up/breach_dn);
  - `is_ambiguous`;
  - AmbiguityRule "ohlc_path"'s open-distance comparison;
  - the Labeler's pnl label boundaries and data_boundary.md's
    max_up/max_down thresholds;
  - `compute_position_size()` and `check_funds_available()` floors and
    comparisons, BacktestEngine's `DOUBLE` inputs entering by V1;
  - the fill simulator's cumulative fill — min(order quantity,
    floor(rate × cumulative tick volume after the anchor)) — and limit-price
    gate,
    its arithmetic float and its output prices passing V1 before any sign
    decision, pnl or write;
  - `stitch_ticks()`'s dedup.
- **Not a grid-value decision site:** the late-entry residual-edge
  comparison (execution_common.md) involves an estimate and follows the
  estimate rule above.

**Crossings — the float boundary rules.**
- V1 (numbers in) and V6 (order requests out) are stated in trading_api.md's
  Response Normalization.
- V1 governs every float entering `Decimal` arithmetic, a vendor value or
  not.
- An estimate or float value entering storage, the ledger or a grid-value
  decision passes V1.

The rest:

- **V2 — no mixing:** `Decimal` and float are neither combined nor compared;
  a float config value or threshold is converted by V1 first.
- **V3 — out:**
  - ratios and statistics leave as float, computed in `Decimal` and
    converted once;
  - a grid value entering an estimate computation is converted to float
    once;
  - exception: V6's 'number' format, whose fate follows
    api_contract_checklist.md T-32.
- **V4 — DB write:** `DECIMAL` columns are bound with `Decimal`; a float bind
  is allowed only for the split-ratio results above (value × ratio and
  value ÷ ratio).
- **V5 — DB read:**
  - a path reading ledger or accounting values is admissible only when a
    contract test asserts that `DECIMAL` columns arrive as `Decimal`;
    `fetchall()` is such a path;
  - an analysis read of a `DECIMAL` column takes float through `.df()` or
    `CAST(... AS DOUBLE)`; `.df()` serves analysis and report paths only.

---

### Health Event Recording

```python
def direct_write(db_conn) -> Callable[[str, tuple], None]:
    """
    The write_fn offline callers pass to record_health_event() and any other
    function taking a write_fn. Returns a closure that runs
    db_conn.execute(sql, params). Satisfies the contract live_mode_runner.md's
    db_write() owns. No lock: an offline caller holds `db.lock` while its
    DuckDB connection is open (db_coordination.md), so there is
    no concurrent writer to exclude.
    """
```

```python
def record_health_event(
    finding_name: str,
    occurred_at: str,
    ticker: str | None,
    detail: dict,
    write_fn: Callable[[str, tuple], None],   # the contract live_mode_runner.md's
                                              # db_write() owns; see below
) -> str:
    """
    Append one row to `health_events` (see db_schema.md) and return the
    generated `event_id`.

    Sole access point for health-event writes — no detection site mints its
    own identifier, for the same single-source-of-truth reason
    `query_halt_status()` is the sole halt-status READ access point — the
    verb differs, the discipline does not (see Constraints, which splits
    that read access from the poller's fetch ownership).

    `event_id` format: '{YYYYMMDD}_{HHMMSSmmm}_{4 random chars}' — the same
    date_time shape as `run_id` elsewhere, extended to millisecond precision
    plus a random suffix so that two events in the same millisecond, or events
    of the processes writing this table — a live session, the evening batch
    and a guest ingestion run (`run_guest migration_tool` or
    `collect_daily.py --ingest-only`); see metadata_crawler.md and
    migration_tool.md — cannot collide without coordination.

    Takes `write_fn`, NOT a `db_conn`. In a live session every write must go
    through LiveModeRunner's `db_write()` funnel (see live_mode_runner.md);
    a function handed a raw connection would bypass `write_lock`, which is
    exactly the defect `persist_to_db()` carried until R-2 GAP-FIX 9. The
    contract is `Callable[[str, tuple], None]` and it is owned by
    `db_write()` in live_mode_runner.md; this signature cites it rather than
    declaring a second one. Offline callers pass `direct_write(db_conn)`,
    the wrapper this module supplies for that purpose, which satisfies the
    same contract.

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
        - live_mode_runner.md's exit-side order tracking, on an order's FIRST
          crossing of exit_order_stuck_minutes (finding 18)
        - live_mode_runner.md's fill-stream staleness detection, on entry into
          each stale episode (finding 24)
        - live_mode_runner.md's Watchdog Polling Loop, on entry into each
          episode of an empty working set inside the regular session
          (finding 29)
        - live_mode_runner.md's fill folding, on a sustained ledger /
          broker-cumulative mismatch for an order number (fill_ledger_mismatch)
        - live_mode_runner.md's startup procedure, on an 'intent' request
          matching several broker orders (intent_match_ambiguous)
        - live_mode_runner.md's exit rejection handling, on a rejection streak
          reaching its threshold (exit_rejection_streak)
        - live_mode_runner.md's Broker Reconcile and Position Manager event-day
          exit gate, on a deferred ticker (reconcile_deferred)
        - live_mode_runner.md's settlement pending handling
          (event_settlement_stalled)
        - live_mode_runner.md's dividend withholding check
          (dividend_withholding_mismatch)
        - live_mode_runner.md's broker summary classification, on an
          unclassified row (summary_vocabulary_candidate)
        - live_mode_runner.md's Session Lifecycle step 7, on a failed regime
          holdout gate (regime_holdout_gate_failed) and at every session start
          (regime_holdout_verdict)
        - live_mode_runner.md's Session Lifecycle step 1e rate resolution, on
          a hard-bound fallback (execution_param_bound_fallback)
        - the evening_execution_param_fit stage's reads of the current value
          (metadata_crawler.md), on a hard-bound fallback
          (execution_param_bound_fallback)
        - the evening_execution_param_fit stage (metadata_crawler.md), on a
          fit rejected by the relative-bound check
          (execution_param_fit_rejected)
        - live_mode_runner.md's Per-Ticker Tick Buffer, on a failed
          held-interval alignment (tick_alignment_failed)
        - live_mode_runner.md's loop A thread, from its top-level exception
          handler (auxiliary_loop_stopped)
        - migration_tool.md's shared ingestion logic, with direct_write, on a
          file that fails to import (collection_gaps, finding 59)
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
    Single source of truth for concatenating REST-fetched vendor 10-tick
    bundles onto an existing tick buffer/history. Used EVERYWHERE bundles are
    stitched (live_mode_runner.md's Per-Ticker Tick Buffer): vendor 10-tick
    fetches — Watchdog Step 2b's, and a held ticker's in Step 2b form.

    Held-ticker REST 1-tick rows are not stitched here: each REST 1-tick
    poll's response replaces every held row, WS or REST, at or after its
    requested second (live_mode_runner.md's held-ticker tape).

    TICK IDENTITY RULE (global): two ticks are the SAME only when `hour`
    AND ALL OHLCV fields match. `hour` alone is NOT a key — multiple ticks
    can share one wall-clock second (sub-second events collapse to the same
    `hour`), so deduping on `hour` alone would wrongly drop distinct ticks.
    Dedup on the full (hour, open, high, low, close, volume) tuple, compared
    in `Decimal` (Numeric Rules, above) — before live broker market data is
    converted to float for the indicator and feature path.

    Known limitation: two genuinely distinct fills that happen to coincide
    on `hour` + OHLCV merge into one. Negligible given 10-tick bundling in
    the historical data this dedup rule was designed against (it implies
    same-second, same-price-band activity within one bundle); recorded, not
    engineered around.

    1-TICK ROWS ARE OUTSIDE THIS RULE:
    - 1-tick response rows carry no intra-second ordinal — the `nnnn` tail
      of the vendor's cont_key, `YYYYMMDDHHMMSSnnnn`, is valid as a query
      continuation key but does not appear in response rows;
    - row order within a second is the response's order;
    - 1-tick identity is handled by the held-ticker tape's interval
      replacement (above), and the tuple dedup here applies to vendor
      10-tick bundles.
    UNSEALED-BUNDLE DISCARD RULE — owned here, referenced by every fetching
    site (live_mode_runner.md):
    - live buffers hold SEALED bundles and 1-tick rows only;
    - a vendor 10-tick response's terminal bundle is unsealed: its `hour`
      and OHLCV update with every arriving print, so the same bundle fetched
      twice does not compare equal under the rule above and both copies
      survive as a phantom;
    - every vendor 10-tick response therefore has its terminal bundle
      discarded on receipt, before the response is stitched; this function
      never receives it.

    Returns: existing with incoming appended, deduped per the rule above,
    re-sorted by (hour, seq_id) if present.
    """
    ...
```

---

### Code Versions

Code areas — each Python file under src/, scripts/ and tools/ belongs to one
area or is excluded:
- the training area:
  - src/balancing/;
  - src/feature_selection/;
  - src/models/;
  - src/optimization/;
  - src/backtest/;
  - src/utils/execution_common.py;
  - scripts/run_train.py;
  - scripts/run_backtest.py;
- excluded:
  - src/live/live_mode_runner.py;
  - src/live/auxiliary_stream.py;
  - src/api/trading_api.py;
  - src/visualization/;
  - src/utils/db_coordination.py;
  - src/utils/guest_supervisor.py;
  - src/utils/alert_delivery.py;
  - tools/;
- the data area: every other file, including:
  - src/detection/;
  - src/preprocessing/;
  - src/labeling/;
  - src/utils/utils.py;
  - src/live/caching_calculator.py;
  - src/live/inferencer.py;
  - scripts/run_preprocess.py.
A data-area file imports no training-area module, except inferencer's use of
execution_common for signal resolution; a test enforces it.

The lock file src/code_versions.json records, per area:
- version;
- code_hash;
- recorded_at;
- the environment snapshot: each listed distribution's version and the
  Python version.
The lock also holds the distribution list; default:
- numpy;
- pandas;
- polars;
- scipy;
- duckdb;
- lightgbm;
- scikit-learn;
- shap.
Only tools/code_version.py writes the lock (code_version.md).

An area's code_hash is the hash of:
- each member file's path with its AST dump, docstrings removed;
- the installed version of each distribution on the lock's list;
- the Python version.

The start check: a process checks the areas it computes from; an area is
unattested when its current code_hash differs from the lock's or the lock has
no entry for it. Areas per site:
- data:
  - run_preprocess;
  - detection_benchmark;
  - migration_tool;
  - the evening_tick_bar_aggregates stage;
  - the evening_session_stats stage;
- data and training:
  - pipeline_optimizer;
  - run_train;
  - run_backtest;
- none: init_db and every other site.
Where it runs and what an unattested area ends: guest_supervisor.md's
run_guest, before New-Run Resolution (`--resume-work` runs included), the run
ending 'blocked' with cause code_unattested; the evening_tick_bar_aggregates
and evening_session_stats stages, at the stage under every collect_daily.py
invocation running them, the stage's
batch_runs row 'failed' and its SUMMARY line naming code_unattested
(metadata_crawler.md); LiveModeRunner's data-area check at process start
(live_mode_runner.md). Delivered as health_report.md's finding 56
code_version_unattested, except LiveModeRunner's, reported through the start
outcome.

The change check, beside the start check: for a run whose tool has areas,
run_guest's unit opener, before opening each unit, compares the run's areas
with their state at the run's start:
- the set of member files;
- each file's mtime and size;
- the listed distributions' versions.
On a difference it recomputes the areas' code_hash; one differing from the
code_hash attested at the start ends the run 'blocked' with cause
code_changed, no unit opened.

Golden tests, per area (code_version.md):
- the fixture: a curated source dataset in tests/golden/fixtures/, loaded
  into a temporary DuckDB; the test calls the area's computations directly;
- data-area outputs on the fixture:
  - the batch matrix content;
  - labeled_samples;
  - precomputed_session_stats;
  - tick_bar_aggregates;
  - the live-path feature vectors of caching_calculator and inferencer for
    the fixture's entry points;
- training-area outputs, from the frozen input matrix
  tests/golden/training/input:
  - predictions on the fixture split;
  - the reduction report and selected features;
  - the backtest's experiment_log totals;
- the training-area fixture config fixes LightGBM's deterministic,
  num_threads and seed;
- comparison is exact for every type, NaN positions included;
- golden outputs are stored in tests/golden/{area}/v{version}/ with the
  fixture hash;
- the golden tests also run in the pytest suite.

---

## Constraints

- All functions are stateless — no module-level state or caching
- `build_effective_bar_sequence()` is the single source of truth for the holding
  deadline and the bars up to it
  — Labeler and BacktestEngine both import from here; logic must not be duplicated
- `build_effective_bar_sequence()` never returns bars with hour > last_bar(date) —
  after-market bars are excluded wherever the holding deadline falls
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
  applies wherever REST vendor 10-tick bundles are concatenated — see that
  function above. 1-tick rows carry no intra-second ordinal and are not
  stitched: their identity is handled by the held-ticker tape's interval
  replacement (live_mode_runner.md). Only SEALED bundles are ever
  stitched — an unsealed terminal bundle mutates between fetches and would
  dedup as two distinct rows
- `track_label_breach()` is the high-level wrapper for two-stage label detection (Labeler only)
- `interpolate_bundle_price()`'s times are continuous — seconds since
  midnight — never `hour_to_int()` values; `expand_synthetic_ticks()` places
  BacktestEngine's synthetic ticks and prices them through it
- `generate_run_id()` not called inside parallel workers or sequential fold loops —
  coordinator pre-generates structured run_ids in every optimizer worker,
  fold loop, outer eval and execution variant; the final model run_id is
  generate_run_id()'s
- `compute_vol_regime_holdout()` uses regular session bars only
  (093000~last_bar(date))
- `compute_consensus_config()` grid-rounding uses nearest valid value from search_space
- `temporal_split_simple()` applies no balancing — training-only utility for early stopping val
- `populate_precomputed_session_stats()` computes absent and stale groups only,
  replacing a stale group's rows in one transaction; a current group is never
  rewritten
- `populate_precomputed_session_stats()` missing-bar handling must be applied per metric type:
  - cumulative metrics take a minute without a bar as contributing nothing
    AND exclude early-close sessions wholesale;
  - per-bar metrics average at a slot only the sessions with a bar there and
    need no early-close handling of their own — an early-close session simply
    supplies no sample past its close, which the existing per-slot count
    already expresses;
  - intra_return_baseline's prev_close is the last existing bar before the
    slot;
  - gap_pct's prev_close is the last existing bar at or before
    last_bar(D-k), while a D-k with no regular-session bar is excluded (see
    function docstring for details)
- `load_session_stats()` → single (config_hash, ticker, as_of_date) DB query; output `{metric: {hour: value}}`
  (flat, no ticker/date dimension); used for live mode Phase 2 (per-ticker on demand)
- `build_session_stats_dict()` → bulk conversion of pre-loaded DataFrame;
  output `{as_of_date: {ticker: {metric: {hour: value}}}}`; used for training bulk load
  and live_mode_runner.md's Session Lifecycle Step 2 (session-start all-ticker bulk)
- `load_session_stats()` never modifies DB — read-only
- `load_session_stats()` returns empty dict (not error) when no rows found
- Delta smoothing in both `load_session_stats()` and `build_session_stats_dict()` is
  in-memory only — DB rows unchanged
- Baselines requiring tick_10 data, skipped if unavailable:
  - `buy_ratio_baseline`;
  - `intra_tpm_baseline`;
  - `intra_avg_vol_per_tick_baseline`
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
  (Step 6, slice_ticker_rows() over the read unit's frames) must produce
  identical results for the same ticker/date range —
  the two exist only because live mode and training load bars via different
  mechanisms (per-ticker query vs. per-batch bulk load), not because the
  underlying stitching logic differs
- `cum_split_ratio()` returns `1.0` (not an error or None) when no splits
  exist in the given range — this is the common case and must be a true no-op
  when multiplied against downstream values
- `adjust_bars_for_corporate_events()` does not mutate its input `bars`
  DataFrame — always returns a new copy, even when the adjustment is a no-op
- `adjust_bars_for_corporate_events()` handles splits only, not dividends,
  since dividends don't create the continuous bar-level price discontinuity
  that motivates a full-series bar adjustment; dividend adjustment is a
  separate scalar correction owned by each caller:
  - `gap_percentile()`'s `dividend_amount`
  - dead position Case A's dividend term
  - 09_backtest_engine.md's carried dividend
- `dividend_gross_amount()` and its frame form `dividend_gross_from_events()`
  are the only readers of a 'dividend' row's `amount`; the frame form owns the
  rule, it returns the gross amount in the `event_date` basis, and only
  cash-flow readers apply `corporate_events.dividend_withholding_rate`
- The split factor is a single rule, with its window as a parameter, owned by
  `split_ratio_from_events()`; `cum_split_ratio()` delegates to it and the
  view `live_position_state` implements the same rule
- `compute_vol_regime_holdout()` returns `{date: rolling_vol}` over the date
  range it is given; its caller computes it once per work over the work's
  pinned range and records it in `regime_holdout_dates`
- `bootstrap_ci()` is shared by the regime holdout report, the champion
  comparison and `divergence_check()`; its seed is fixed by config so a rerun
  reproduces the interval
- Numeric values follow the Numeric Rules section above; V1–V5 are binding
  on every path that reads, computes, decides on or writes them
- `estimate_historical_meta()` must never be called with `field="sector"` or
  `field="security_type"` — these fields have no derivation path; callers use the
  chain-latest symbol's latest stock_meta row with the field set, whatever
  the date
- `estimate_historical_meta()`'s `shares_outstanding` derivation does not
  capture buyback/issuance activity after the date of the stock_meta row it
  reads (tiers 1 and 3); tier 2 covers what `fundamentals_quarterly` holds;
  this is a known, accepted limitation, distinct from the corporate-event
  scale-correction problem
- `estimate_historical_meta()` resolves `shares_outstanding` and `market_cap`
  none for a non-common security_type and skips its fundamentals tier for an
  adr
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
- `compute_tick_bar_aggregates()` is the sole IndicatorCalculator wrapper
- `load_tick_bar_aggregates_with_history()`'s Tier 2 (on-the-fly compute)
  result is never written back to `tick_bar_aggregates` — the table's writers
  are metadata_crawler.md's evening_tick_bar_aggregates stage,
  migration_tool.md's backfill and `populate_precomputed_session_stats()`,
  each under the current data version; Tier 2 existing at all is expected to
  be rare in steady-state operation, not a sign of a broken pipeline
- Tick-derived indicators requiring more than one tick bundle to be defined
  (e.g. any delta- or interval-based indicator between bundles) must return
  NULL/NaN for bars with fewer bundles than required, rather than raising or
  silently omitting the row — see 02_indicator_calculator.md's Tick-Derived
  Indicator Scale Sensitivity registry for the current minimum-bundle-count
  requirements per indicator
- `get_ttm_value()` and `estimate_historical_meta()`'s `fundamentals_quarterly`
  tier filter on `filed_date < date`, never `fiscal_period_end` — see
  data_boundary.md's point-in-time principle; this applies to any future
  fundamentals_quarterly consumer as well
- `fx_per_usd()` reads only a rate dated on or before the trading day before
  the entry date, never a later rate — a monetary fact meets `market_cap` in
  its own currency at that rate (data_boundary.md's "Point-in-time
  correctness for fundamentals")
- `estimate_historical_meta()` reads, for date D, stock_meta rows dated
  before D, the prior-day row first, and bars of the trading days before D —
  data_boundary.md's "Point-in-time correctness for stock_meta"
- `resolve_xbrl_tag_value()`'s tag-priority list per metric lives in
  `configs/xbrl_tag_map.json`, not hardcoded in this function — adding a
  fallback tag for an existing metric is a config change, not a code change;
  the per-filing merge (the highest-priority tag holding facts in that
  filing) and the unit choice (shares_outstanding's 'shares' facts; another
  metric's USD unit, else the period's one shared unit, else not stored) are
  the function's
- `query_halt_status()` is the sole LIVE halt-status READ access point, and
  live_mode_runner.md's Halt-Status Poller is the sole FETCH owner of that
  source, through its polling loop in the runner process and through its
  single-fetch entry in any other — one side reads a snapshot, the other holds the feed that fills
  it, and no third site does either, for the same single-source-of-truth
  reason. It is NOT a trading-API endpoint: the dbsec catalogue has none, so this
  function sits outside trading_api.md's boundary entirely. The scope is
  LIVE status, not halts as such: metadata_crawler.md's
  `crawl_nyse_halts()` does query a halt source directly, the NYSE page
  filling `trading_halts` as a historical record, and the two staying
  separate is what lets the evening comparison read as two publishers
  rather than one source against itself
- `query_halt_status()` returning `None` is a failure to SERVE, not "no
  tickers halted" — either the fetch failed or the snapshot sits past
  `halt_snapshot_max_age_seconds`, and to a caller those are one case.
  Callers must not conflate it with an empty result; conflating them would
  mean a dead poller silently reads as "everything is fine"
- `populate_trading_calendar()` never rewrites the calendar context columns
  on a row dated on or before the run day, a NULL excepted
