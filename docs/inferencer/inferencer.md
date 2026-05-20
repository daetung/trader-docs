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
    └── Preprocessor.run(live_mode=True)   ← feature extraction only (no labeling)
    └── model.load(run_id)                 ← load trained artifacts
    └── model.predict(models, X)           ← probability output
    └── utils.resolve_signal(row, threshold, suppress_threshold) ← entry decision
```

---

## Input / Output

**Input:**
```python
bars: pd.DataFrame
    # recent ohlcv_1min bars up to and including t-1 bar
    # all sessions included (no time filter)

ticks: pd.DataFrame
    # tick_10 data strictly before t bar open

meta: dict
    # stock_meta row for this ticker

entry: dict
    # {ticker, date, hour, p_entry}
    # hour = t bar open time
    # p_entry = t bar open price
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
    suppressed: bool,          # True if signal suppressed by suppress_threshold
    run_id:     str,           # model run_id used for inference
)
```

---

## Processing Flow

```
1. Preprocessor.run(live_mode=True)
       → EntryPointDetector.detect() on current bars
       → if not a candidate entry point: return None immediately
       → FeatureExtractor.extract() → feature_vector

2. model.predict(models, X) → probability scores

3. utils.resolve_signal(row, threshold, suppress_threshold) → signal

4. Return InferenceResult
```

live_mode excludes:
- Labeler (future prices unknown)
- ClassBalancer (training-only)

---

## Signal Resolution

Signal resolution uses the same `utils.resolve_signal()` function as BacktestEngine,
ensuring identical entry decision logic between backtest and live inference.

```python
from utils import resolve_signal

threshold          = config["backtest"]["entry_threshold"]
suppress_threshold = config["backtest"]["suppress_threshold"]  # None = disabled

signal = resolve_signal(row, threshold, suppress_threshold)
suppressed = (signal is None and (
    row["prob_dn5"] >= suppress_threshold or
    row["prob_dn3"] >= suppress_threshold
)) if suppress_threshold is not None else False
```

`suppressed=True` indicates the entry was blocked by downside conviction,
not by insufficient upside probability. This distinction is useful for
live monitoring and post-hoc analysis.

---

## Session Mode in Live Inference

`session_mode` config applies to live inference identically to training:
```
"regular"  : only fire signals during regular session (093000~155900)
"pre"      : only fire signals during pre-market (040000~092900)
"combined" : fire signals during pre-market and regular session
```

After-market signals (hour > 155900) suppressed in all modes.

Late-day exclusion also applies in live inference:
```
max_entry_hour: if set, signals after this hour are not fired
                even if entry detection and model conditions are met
```

---

## Model Loading

```python
model   = create_model(config)
models, meta = model.load(run_id)
feature_names    = meta["feature_names"]
categorical_cols = meta["categorical_cols"]
```

`run_id` specified in config or passed at runtime.
Encoding maps loaded via `utils.load_encoding_map()` — same maps used during training.

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
    suppressed: bool       # True if downside suppression triggered
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
    ) -> InferenceResult | None:
        """
        Run live inference for a single entry candidate.
        Returns None if entry detection does not fire.
        """
        ...

    def infer_batch(
        self,
        candidates: list[dict],
        bars_by_ticker: dict[str, pd.DataFrame],
        ticks_by_ticker: dict[str, pd.DataFrame],
        meta_by_ticker: dict[str, dict],
    ) -> list[InferenceResult]:
        """
        Run inference for multiple entry candidates (e.g. end-of-bar scan).
        Returns list of InferenceResult where signal is not None.
        Suppressed entries (signal=None, suppressed=True) are excluded
        from the returned list but counted in internal diagnostics.
        """
        ...
```

---

## Constraints

- Inferencer must use identical feature extraction logic as training pipeline
- Encoding maps (`sector_map`, `day_of_week_map`) loaded via `utils.load_encoding_map()` — must match training-time maps
- `utils.resolve_signal()` used for signal thresholding — same function as BacktestEngine
- `suppress_threshold` read from same config key as BacktestEngine (`config["backtest"]["suppress_threshold"]`)
- No data leakage: t bar OHLCV must not be used as feature input
- `live_mode=True` passed to Preprocessor — Labeler never instantiated
- After-market entry points suppressed regardless of session_mode
- `max_entry_hour` applied in live inference — consistent with training-time scan() behavior
- Full implementation deferred; this spec defines the interface contract only
