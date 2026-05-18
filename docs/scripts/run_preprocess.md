# Script: run_preprocess.py

**File:** `scripts/run_preprocess.py`

---

## Role

CLI entry point that orchestrates the full preprocessing pipeline.
Supports three execution modes:
- **Standalone**: full pipeline, saves parquet to disk
- **Optimizer (in-memory)**: returns DataFrames without saving, called by PipelineOptimizer
- **Live (live_mode)**: runs EntryPointDetector + FeatureExtractor only, used by Inferencer

Preprocessor is responsible for feature extraction only. Session_mode filtering
and class balancing are handled entirely by ClassBalancer.split(), ensuring that
the preprocessed feature matrix can be reused across different session_mode
and balancing configurations without re-running feature extraction.

---

## Input / Output

**Input:**
```python
# From DuckDB:
ohlcv_1min: pd.DataFrame       # all sessions (no time filter)
tick_10: pd.DataFrame          # all sessions
stock_meta: pd.DataFrame
trading_halts: pd.DataFrame
trading_calendar: pd.DataFrame
ticker_data_coverage: pd.DataFrame
```

**Output (standalone mode):**
```python
# Saved to data/processed/:
train_features.parquet    # balanced training split (session-filtered)
val_features.parquet      # validation split (session-filtered)
test_features.parquet     # test split (session-filtered)

# Parquet column structure per file:
# [ticker, date, hour, p_entry] + [features...] + [label_up5..label_dn5, is_dead_position]
```

**Output (in-memory mode, return_data=True):**
```python
tuple[pd.DataFrame, pd.DataFrame, pd.DataFrame]  # (train, val, test)
```

**Output (live_mode=True):**
```python
pd.DataFrame  # feature matrix for current entry points (no labels)
```

---

## Pipeline Steps

```
[Standard / In-memory mode]

1. Connect to DuckDB and load all data
2. EntryPointDetector.scan() for each ticker → entry_points (all sessions)
3. Save entry_points to DuckDB entry_points table
4. Labeler.label() for all entry points → labeled_samples
5. Save labeled_samples to DuckDB labeled_samples table
6. FeatureExtractor.extract_batch() for each ticker → feature matrix
7. Merge features with labels on (ticker, date, hour)
   → labeled feature matrix: [ticker, date, hour, p_entry] + [features] + [labels, is_dead_position]
   → contains all sessions; no session_mode filter applied here
8. ClassBalancer.split(
       balance=config["class_balancer"]["apply_balance"],
       session_mode=config["entry_detector"]["session_mode"]
   ) → (train_balanced, val, test)
   → session_mode filter and balancing both applied inside split()
   → train, val, test all contain only target session's entry points
9. If return_data=True: return (train_balanced, val, test)
   Else: save parquet files to data/processed/

[Live mode — live_mode=True]

1. Connect to DuckDB and load recent OHLCV, ticks, meta
2. EntryPointDetector.scan() for current bars → entry_points
3. FeatureExtractor.extract_batch() → feature matrix (no labels)
4. Return feature matrix directly (no save, no labeling, no session filter)
```

---

## Class Interface

`run_preprocess.py` exposes a callable class for use by PipelineOptimizer and Inferencer:

```python
class Preprocessor:
    def __init__(self, config: dict, db_conn: duckdb.DuckDBPyConnection): ...

    def run(
        self,
        return_data: bool = False,
        live_mode: bool = False,
    ) -> tuple[pd.DataFrame, pd.DataFrame, pd.DataFrame] | pd.DataFrame | None:
        """
        return_data=False, live_mode=False : save parquet, return None
        return_data=True,  live_mode=False : return (train, val, test), no save
        live_mode=True                     : return feature matrix only (no labels)
        """
        ...

    def save(
        self,
        train: pd.DataFrame,
        val: pd.DataFrame,
        test: pd.DataFrame,
        run_id: str | None = None,
    ) -> None:
        """
        Save parquet files to data/processed/.
        If run_id provided, saves to data/processed/{run_id}/ for trial isolation.
        Called explicitly by PipelineOptimizer after AUC loop converges.
        """
        ...
```

---

## CLI Interface

```bash
python scripts/run_preprocess.py [--config CONFIG]

Options:
    --config        Path to pipeline_config.yaml (default: configs/pipeline_config.yaml)
```

CLI always runs in standalone mode (return_data=False, live_mode=False).

---

## Config Keys Used

```yaml
data_paths.*
entry_detector.*           # includes session_mode, volume_base_hour
indicators.*
vectorizer.*
labeler.*
class_balancer.*           # includes apply_balance; session_mode read from entry_detector
misc.lookback_bars
```

---

## Constraints

- Preprocessor is responsible for feature extraction only — no session_mode filter,
  no balancing applied inside Preprocessor
- Session_mode filtering and balancing are both handled by ClassBalancer.split()
- All data loaded from DuckDB in bulk (no per-ticker DB queries during extraction)
- `extract_batch()` called per ticker — no Python loop over `extract()` per entry point
- Empty DataFrames handled gracefully (saves _empty.parquet)
- `ClassBalancer.split()` is the single call — `balance()` is not called separately
- `balance` argument to `split()` read from `config["class_balancer"]["apply_balance"]`
- `session_mode` argument to `split()` read from `config["entry_detector"]["session_mode"]`
- `p_entry` included in parquet as identifier column, not feature
- In live_mode: Labeler and ClassBalancer are not instantiated
