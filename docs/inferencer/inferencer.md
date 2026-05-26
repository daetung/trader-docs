# Module: Inferencer

**File:** `src/inference/inferencer.py`

> **Status:** Interface defined. Full implementation deferred until training pipeline stabilizes.

---

## Role

Live inference endpoint for the auto-scalping pipeline.
Receives real-time bar data, runs feature extraction, applies trained model,
and returns entry signals for live trading decisions.

---

## Architecture Position

```
Live inference endpoint: Inferencer
    └── EntryPointDetector.detect()            ← signal detection (not scan())
    └── [entry_points INSERT to DuckDB]        ← Inferencer responsibility
    └── Preprocessor.run(live_mode=True)       ← feature extraction only (no labeling)
    └── model.load(run_id)                     ← load trained artifacts
    └── model.predict(models, X)               ← probability output
    └── utils.resolve_signal(row, threshold, suppress_threshold) ← entry decision
```

---

## Input / Output

**Input:**
```python
bars: pd.DataFrame
    # recent ohlcv_1min bars up to and including t-1 bar
    # t bar must NOT be included
    # all sessions included (no time filter)

ticks: pd.DataFrame
    # tick_10 data strictly before t bar open

meta: dict
    # stock_meta row for this ticker

entry: dict
    # {ticker, date, hour, p_entry}
    # hour = t bar open time
    # p_entry = t bar open price (from external real-time feed)
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
)
```

---

## Processing Flow

```
1. EntryPointDetector.detect(bars, date, shares_outstanding)
       → if not a candidate entry point: return None immediately

2. Obtain p_entry from external real-time feed
       → p_entry = t bar open price from real-time data source
       → NOT derived from bars DataFrame

3. Insert entry point to DuckDB entry_points table:
       INSERT OR IGNORE INTO entry_points (ticker, date, hour, p_entry)

4. Preprocessor.run(live_mode=True)
       → halts_df = db_conn.execute(
             "SELECT * FROM trading_halts WHERE ticker = ? AND date = ?"
         ).df()
       → FeatureExtractor.extract(bars, ticks, meta, entry, halts_df) → feature_vector
       (Encoding maps loaded internally by FeatureExtractor via utils.load_encoding_map())

5. model.predict(models, X) → probability scores

6. utils.resolve_signal(row, threshold, suppress_threshold) → signal

7. Return InferenceResult
```

live_mode excludes:
- scan() (not called — detect() used instead)
- Labeler (future prices unknown)
- ClassBalancer (training-only)

---

## Signal Resolution

```python
from utils import resolve_signal

threshold          = config["backtest"]["entry_threshold"]
suppress_threshold = config["backtest"]["suppress_threshold"]  # None = disabled

signal = resolve_signal(row, threshold, suppress_threshold)
suppressed = (signal is None and suppress_threshold is not None and (
    row["prob_dn5"] >= suppress_threshold or
    row["prob_dn3"] >= suppress_threshold
))
```

---

## Session Mode in Live Inference

```
"regular"  : only fire signals during regular session (093000~155900)
"pre"      : only fire signals during pre-market (040000~092900)
"combined" : fire signals during pre-market and regular session
```

After-market signals (hour > 155900) suppressed in all modes.

Late-day exclusion applies in live inference:
```
max_entry_hour: if set, signals with t bar open time > max_entry_hour are not fired,
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
    ): ...

    def infer(
        self,
        bars: pd.DataFrame,
        ticks: pd.DataFrame,
        meta: dict,
        entry: dict,
    ) -> InferenceResult | None: ...

    def infer_batch(
        self,
        candidates: list[dict],
        bars_by_ticker: dict[str, pd.DataFrame],
        ticks_by_ticker: dict[str, pd.DataFrame],
        meta_by_ticker: dict[str, dict],
    ) -> list[InferenceResult]: ...
```

---

## Constraints

- Inferencer must use identical feature extraction logic as training pipeline
- `scan()` must NOT be called in live inference — `detect()` only
- `p_entry` must be provided from external real-time feed — not derived from bars
- Entry point insertion to `entry_points` table is Inferencer's responsibility
- `entry_points` INSERT uses INSERT OR IGNORE for idempotency
- Encoding maps (`sector_map`, `day_of_week_map`, `halt_reason_code_map`) are loaded
  internally by FeatureExtractor via `utils.load_encoding_map()` —
  Inferencer does not load or inject encoding maps directly
- `categorical_cols` loaded from model artifact meta — no heuristic pattern matching
- `utils.resolve_signal()` used for signal thresholding — same function as BacktestEngine
- `suppress_threshold` read from `config["backtest"]["suppress_threshold"]`
- No data leakage: t bar OHLCV must not be used as feature input
- `live_mode=True` passed to Preprocessor — Labeler never instantiated
- After-market entry points suppressed regardless of session_mode
- `max_entry_hour`: signals with t bar open time > max_entry_hour are not fired
  (> operator consistent with scan() exclusion logic; boundary is inclusive)
- `halts_df` queried from trading_halts table via db_conn at inference time;
  passed explicitly to FeatureExtractor.extract()
- Full implementation deferred; this spec defines the interface contract only
