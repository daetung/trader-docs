# Module: Inferencer

**File:** `src/inference/inferencer.py`

---

## Role

Live inference endpoint for the auto-scalping pipeline.
Receives real-time bar data, runs feature extraction, applies trained model,
and returns entry signals for live trading decisions.

Inferencer is a **service object** — it does not manage execution loops.
LiveModeRunner is the orchestrator that calls `infer()` at appropriate times.
See `docs/inferencer/live_mode_runner.md`.

---

## Architecture Position

```
Live inference endpoint: LiveModeRunner (execution orchestrator)
    └── Inferencer  (service object — called per entry point)
            └── EntryPointDetector.detect()            ← signal detection (not scan())
            └── [entry_points INSERT to DuckDB]        ← Inferencer responsibility
            └── FeatureExtractor.extract()             ← direct call (no Preprocessor)
                  (CachingIndicatorCalculator injected via DI)
            └── model.load(run_id)                     ← load trained artifacts
            └── model.predict(models, X)               ← probability output
            └── utils.resolve_signal(row, threshold, suppress_threshold)
            └── inference_log writes (DuckDB)
```

Inferencer does NOT use Preprocessor. Live inference uses `FeatureExtractor.extract()`
directly, sharing the same feature extraction code path as training without going through
the full preprocessing pipeline (no labeling, no DB bulk loads).

---

## Session Policy Reference

Indicators are classified by session policy (informational — no implementation change here):

```
SESSION_RESET  : VWAP — resets at session boundary
CONTINUOUS     : EMA, ATR, RSI, BB, ADX, OBV, fibonacci, etc.
                 Precalculated at session start via CachingIndicatorCalculator
REFERENCE_SESSION: RVOL, rel_dvol, gap_percentile, intraday_seasonality
                 Baseline from precomputed_session_stats (loaded at session start)
```

`calculate_required_history()` considers CONTINUOUS indicators only:
- `min_bars = lookback_days * 390` — not per-date. The 390 names the regular
  session while `bars` is loaded across the full 040000~200000 range, so the
  real supply is nearer 960 min/day; the margin is understated rather than at
  risk, on an early-close day as on any other.
- REFERENCE_SESSION baseline is satisfied by precomputed_session_stats independently

---

## Input / Output

**Input:**
```python
bars: pd.DataFrame
    # recent ohlcv_1min bars up to and including t-1 bar
    # t bar should NOT be included (auto-trimmed if present — see _prepare_bars)
    # all sessions included (no time filter)

ticks: pd.DataFrame
    # tick_10 data strictly before t bar open

meta: dict
    # flat {field: value}, already resolved by the caller (LiveModeRunner's
    # meta_bulk — see live_mode_runner.md) with per-field
    # utils.estimate_historical_meta() fallback where today's stock_meta
    # crawl is incomplete; not a raw stock_meta row passed through
    # unresolved — matches 04_feature_extractor.md extract()'s meta contract

entry: dict
    # {ticker, date, hour, p_entry}
    # hour = t bar open time
    # p_entry = t bar open price (from external real-time feed)

session_stats: dict | None
    # pre-loaded REFERENCE_SESSION baselines from precomputed_session_stats
    # supplied by LiveModeRunner at session start
    # format: {metric: {hour: smoothed_avg_value}}
```

**Output:**
```python
InferenceResult(
    ticker:     str,
    date:       str,
    hour:       str,
    p_entry:    float,
    signal:     str | None,    # "up5" | "up3" | None
    prob_up5:   float,
    prob_up3:   float,
    prob_sw:    float,
    prob_dn3:   float,
    prob_dn5:   float,
    suppressed: bool,
    run_id:     str,
) | None   # None on preload_fail, no_detection, session filter
```

---

## Bar Preparation — `_prepare_bars()`

```
_prepare_bars(bars, entry) -> pd.DataFrame | None:

1. Filter: remove bars with hour >= entry["hour"]
   (t bar or later — may be present due to API timing or partial bar delivery)

2. If rows were removed:
   → log to inference_log (event="bars_trimmed", fail_reason=f"trimmed {N} bars")

3. Validate: if len(cleaned_bars) < self.required_bars:
   → log to inference_log (event="preload_fail",
                            required_bars=self.required_bars,
                            actual_bars=len(cleaned_bars))
   → return None

4. return cleaned_bars
```

---

## Processing Flow

```
infer(bars, ticks, meta, entry, session_stats=None):

step 0: bars = self._prepare_bars(bars, entry)
        if bars is None: return None    # preload_fail logged in _prepare_bars

1. Session mode filter:
       session_mode = config["entry_detector"]["session_mode"]
       hour = entry["hour"]

       last = last_bar(entry["date"])           # utils.md — PER DATE
       if hour > last: return None              # after-market always excluded

       if session_mode == "regular" and not ("093000" <= hour <= last):
           return None
       if session_mode == "pre" and not ("040000" <= hour <= "092900"):
           return None
       # "combined" passes both pre-market and regular

2. Late-day filter:
       off = config["entry_detector"]["max_entry_offset_minutes"]
       if off is not None:                     # null = no cutoff
           max_entry_hour = last - off minutes # PER DATE, follows the calendar
           if hour > max_entry_hour:
               return None      # > operator, consistent with scan() exclusion

3. EntryPointDetector.detect(bars, date=entry["date"],
                              shares_outstanding=meta["shares_outstanding"])
       if not detected:
           log inference_log (event="no_detection")
           return None

4. Insert entry point to DuckDB:
       INSERT OR IGNORE INTO entry_points (ticker, date, hour, p_entry)

5. Load halts_df (Inferencer-side DB query):
       halts_df = db_conn.execute(
           "SELECT * FROM trading_halts WHERE ticker = ? AND date = ?",
           [entry["ticker"], entry["date"]]
       ).df()

6. FeatureExtractor.extract(bars, ticks, meta, entry, halts_df,
                             session_stats=session_stats,
                             session_close=session_close(entry["date"]))
       # is_near_close needs the exchange close. Flat, not the map: extract()
       # handles exactly one date, the same reason its meta stays flat.
       → feature_vector dict[str, float]
       (CachingIndicatorCalculator used via DI in FeatureExtractor)
       (Encoding maps loaded internally by FeatureExtractor)

7. X = pd.DataFrame([feature_vector])[feature_names]
   probs_df = model.predict(models, X)
   row = probs_df.iloc[0]

8. Signal resolution:
       signal = utils.resolve_signal(row, threshold, suppress_threshold)
       suppressed = (
           signal is None
           and suppress_threshold is not None
           and (row["prob_dn5"] >= suppress_threshold
                or row["prob_dn3"] >= suppress_threshold)
       )

9. Log to inference_log:
       event = "signal_fired" if signal else ("suppressed" if suppressed else "no_signal")
       INSERT INTO inference_log (logged_at, ticker, date, hour, event, signal,
                                  prob_up5, prob_up3, prob_sw, prob_dn3, prob_dn5, run_id)

10. Return InferenceResult(
       ticker=entry["ticker"], date=entry["date"], hour=entry["hour"],
       p_entry=entry["p_entry"], signal=signal,
       prob_up5=row["prob_up5"], prob_up3=row["prob_up3"], prob_sw=row["prob_sw"],
       prob_dn3=row["prob_dn3"], prob_dn5=row["prob_dn5"],
       suppressed=suppressed, run_id=self.run_id,
   )
```

---

## infer_batch()

```python
def infer_batch(
    self,
    candidates:      list[dict],
    bars_by_ticker:  dict[str, pd.DataFrame],
    ticks_by_ticker: dict[str, pd.DataFrame],
    meta_by_ticker:  dict[str, dict],
    session_stats:   dict | None = None,
) -> tuple[list[InferenceResult], dict]:
    """
    Process multiple candidates efficiently.
    halts_df loaded once per (ticker, date) pair — not per candidate.
    session_stats shared across all candidates (same session).
    meta_by_ticker: per-ticker resolved flat dicts, same per-field fallback
        contract as infer()'s meta (see Input/Output above) — not raw
        stock_meta rows passed through unresolved.

    Returns:
        (results, stats_dict)
        stats_dict: {
            "total":        int,   # total candidates processed
            "fired":        int,   # signals emitted (up5 or up3)
            "preload_fail": int,   # insufficient history
            "no_detection": int,   # detect() returned False
            "suppressed":   int,   # suppressed by suppress_threshold
            "no_signal":    int,   # inference completed, no signal
        }
    """
    halts_cache: dict[tuple[str, str], pd.DataFrame] = {}
    results = []
    stats = {"total": 0, "fired": 0, "preload_fail": 0,
             "no_detection": 0, "suppressed": 0, "no_signal": 0}

    for entry in candidates:
        stats["total"] += 1
        ticker, date = entry["ticker"], entry["date"]
        cache_key = (ticker, date)
        if cache_key not in halts_cache:
            halts_cache[cache_key] = db_conn.execute(
                "SELECT * FROM trading_halts WHERE ticker = ? AND date = ?",
                [ticker, date]
            ).df()
        result = self._infer_single(
            bars_by_ticker[ticker], ticks_by_ticker[ticker],
            meta_by_ticker[ticker], entry,
            halts_cache[cache_key], session_stats,
        )
        if result is not None:
            results.append(result)
            if result.signal:
                stats["fired"] += 1
            elif result.suppressed:
                stats["suppressed"] += 1
            else:
                stats["no_signal"] += 1

    return results, stats
```

`_infer_single()` performs steps 0–10 of `infer()` with pre-loaded `halts_df`.
`infer()` loads `halts_df` then delegates to `_infer_single()`.

---

## Late-Day Exclusion

```
max_entry_offset_minutes: if not null, signals with t bar open time later than
    last_bar(date) - max_entry_offset_minutes are not fired,
                consistent with scan() exclusion logic (> operator, boundary inclusive)
```

---

## Model Loading

```python
model   = create_model(config)
models, meta = model.load(run_id)
feature_names    = meta["feature_names"]
categorical_cols = meta["categorical_cols"]
```

`categorical_cols` loaded from model artifact meta — no heuristic inference at runtime.
`run_id` specified in config or passed at runtime.

---

## Interface

```python
@dataclass
class InferenceResult:
    ticker:     str
    date:       str
    hour:       str
    p_entry:    float
    signal:     str | None
    prob_up5:   float
    prob_up3:   float
    prob_sw:    float
    prob_dn3:   float
    prob_dn5:   float
    suppressed: bool
    run_id:     str


class Inferencer:
    def __init__(
        self,
        config: dict,
        db_conn: duckdb.DuckDBPyConnection,
        run_id: str,
        feature_extractor: FeatureExtractor,   # DI: CachingIndicatorCalculator injected
        closes: dict[str, str],
        ends: dict[str, str],
    ):
        """
        feature_extractor is injected with CachingIndicatorCalculator by LiveModeRunner.
        calculate_required_history() result stored as self.required_bars at init.

        closes/ends are RECEIVED, not loaded here: LiveModeRunner holds them from
        its early-close gate, which runs before this init, and a second load would
        be a second delivery path for one fact. They are held for the session the
        same way required_bars is — the calendar cannot change mid-session.
        """
        ...

    def infer(
        self,
        bars: pd.DataFrame,
        ticks: pd.DataFrame,
        meta: dict,
        entry: dict,
        session_stats: dict | None = None,
    ) -> InferenceResult | None: ...

    def infer_batch(
        self,
        candidates:      list[dict],
        bars_by_ticker:  dict[str, pd.DataFrame],
        ticks_by_ticker: dict[str, pd.DataFrame],
        meta_by_ticker:  dict[str, dict],
        session_stats:   dict | None = None,
    ) -> tuple[list[InferenceResult], dict]: ...
```

---

## Constraints

- Inferencer is a service object — does not manage polling loops, position monitoring, or trade execution
- LiveModeRunner instantiates Inferencer once per session and calls `infer()` per entry point
- `scan()` must NOT be called in live inference — `detect()` only
- `p_entry` must be provided from external real-time feed — not derived from bars
- `_prepare_bars()` auto-trims t bar or later bars defensively; caller (LiveModeRunner)
  should provide t-1 and earlier as a best practice
- Entry point insertion to `entry_points` table is Inferencer's responsibility
- `entry_points` INSERT uses INSERT OR IGNORE for idempotency
- All inference_log writes include `run_id`
- Encoding maps (`sector_map`, `day_of_week_map`, `halt_reason_code_map`) are loaded
  internally by FeatureExtractor via `utils.load_encoding_map()` —
  Inferencer does not load or inject encoding maps directly
- `categorical_cols` loaded from model artifact meta — no heuristic pattern matching
- `utils.resolve_signal()` used for signal thresholding — same function as BacktestEngine
- `suppress_threshold` read from `config["backtest"]["suppress_threshold"]`
- No data leakage: t bar OHLCV must not be used as feature input
- Preprocessor is NOT used in live inference — FeatureExtractor.extract() called directly
- After-market entry points suppressed regardless of session_mode
  (hour > last_bar(date))
- Session mode and late-day filters applied by Inferencer before detect()
- `max_entry_offset_minutes`: when not null, signals with t bar open time >
  max_entry_hour(date) are not fired, where max_entry_hour(date) = last_bar(date)
  - max_entry_offset_minutes (> operator consistent with scan() exclusion logic;
  boundary is inclusive). null disables the cutoff entirely
- `halts_df` queried from trading_halts table via db_conn at inference time;
  passed explicitly to FeatureExtractor.extract()
- `infer_batch()` caches halts_df per (ticker, date) — not per candidate
- `session_stats` is supplied by LiveModeRunner at session start; None is allowed (graceful degradation)
- Model artifacts loaded from `run_id` at init — not reloaded per call
- `feature_names` and `categorical_cols` derived from FeatureExtractor at init
