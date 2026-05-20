# Script: run_preprocess.py

**File:** `scripts/run_preprocess.py`

---

## Role

CLI entry point that orchestrates the full preprocessing pipeline.
Supports three execution modes:
- **Standalone**: full pipeline, saves parquet to disk
- **Optimizer (in-memory)**: returns full labeled DataFrame without splitting or saving,
  called by PipelineOptimizer; fold generation delegated to ClassBalancer
- **Live (live_mode)**: runs EntryPointDetector + FeatureExtractor only, used by Inferencer

Preprocessor is responsible for feature extraction and labeling only.
Session_mode filtering, temporal splitting, and rolling fold generation
are handled entirely by ClassBalancer, ensuring that the preprocessed
feature matrix can be reused across different session_mode and split
configurations without re-running feature extraction.

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
train_features.parquet    # balanced training split (session-filtered, temporally split)
val_features.parquet      # validation split (session-filtered)
test_features.parquet     # test split (session-filtered)

# Parquet column structure per file:
# [ticker, date, hour, p_entry]
# + [features...]
# + [label_up5..label_dn5, is_dead_position, dead_position_case, is_ambiguous]
```

**Output (in-memory mode, return_data=True):**
```python
pd.DataFrame  # full labeled feature matrix, unsplit
              # all sessions, all entry points
              # ClassBalancer.generate_folds() or split() applied by caller
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
   max_entry_hour exclusion applied inside scan()
3. Save entry_points to DuckDB entry_points table
4. Labeler.label() for all entry points → labeled_samples
   (includes is_dead_position, dead_position_case, is_ambiguous)
5. Save labeled_samples to DuckDB labeled_samples table
6. FeatureExtractor.extract_batch() for each ticker → feature matrix
7. Merge features with labels on (ticker, date, hour)
   → labeled feature matrix:
     [ticker, date, hour, p_entry]
     + [features]
     + [labels, is_dead_position, dead_position_case, is_ambiguous]
   → contains all sessions; no session_mode filter applied here

8a. If return_data=True (optimizer mode):
    → return full_labeled_df directly (no split, no save)
    → ClassBalancer.generate_folds() or split() called by PipelineOptimizer

8b. If return_data=False (standalone mode):
    → ClassBalancer.split(
           balance=config["class_balancer"]["apply_balance"],
           session_mode=config["entry_detector"]["session_mode"]
       ) → (train_balanced, val, test)
       Note: split_method in config determines temporal vs rolling behavior;
             standalone mode always uses split() for a single saved output
    → save parquet files to data/processed/

[Live mode — live_mode=True]

1. Connect to DuckDB and load recent OHLCV, ticks, meta
2. EntryPointDetector.scan() for current bars → entry_points
   (max_entry_hour exclusion applied; after-market excluded)
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
    ) -> pd.DataFrame | tuple[pd.DataFrame, pd.DataFrame, pd.DataFrame] | None:
        """
        return_data=False, live_mode=False : split via ClassBalancer.split(),
                                             save parquet, return None
        return_data=True,  live_mode=False : return full_labeled_df (unsplit),
                                             no ClassBalancer call, no save
        live_mode=True                     : return feature matrix only (no labels)

        When return_data=True, the returned DataFrame contains all sessions
        and all entry points. Caller is responsible for fold generation
        via ClassBalancer.generate_folds() or split().
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
        Called explicitly by PipelineOptimizer after best trial is selected.
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

CLI always runs in standalone mode (`return_data=False`, `live_mode=False`).
Standalone mode uses `ClassBalancer.split()` regardless of `split_method` config,
producing a single train/val/test output for immediate use or inspection.

---

## Config Keys Used

```yaml
data_paths.*
entry_detector.*           # includes session_mode, volume_base_hour, max_entry_hour
indicators.*
vectorizer.*
labeler.*                  # includes ambiguity_priority, dead_position_penalty_pct
class_balancer.*           # split_method, temporal/rolling params, pre-balance filters
misc.lookback_bars
```

---

## Constraints

- Preprocessor is responsible for feature extraction and labeling only —
  no session_mode filter, no balancing, no fold generation applied inside Preprocessor
  when return_data=True
- Session_mode filtering, splitting, and fold generation are all handled by ClassBalancer
- All data loaded from DuckDB in bulk (no per-ticker DB queries during extraction)
- `extract_batch()` called per ticker — no Python loop over `extract()` per entry point
- Empty DataFrames handled gracefully (saves _empty.parquet)
- `ClassBalancer.split()` called only in standalone mode (return_data=False)
- `ClassBalancer.generate_folds()` is never called inside Preprocessor — caller's responsibility
- `balance` argument to `split()` read from `config["class_balancer"]["apply_balance"]`
- `session_mode` argument to `split()` read from `config["entry_detector"]["session_mode"]`
- `p_entry` included in parquet as identifier column, not feature
- `is_dead_position`, `dead_position_case`, `is_ambiguous` passed through from Labeler —
  stored as metadata columns in parquet, not features
- In live_mode: Labeler and ClassBalancer are not instantiated
- `max_entry_hour` exclusion applied inside EntryPointDetector.scan() —
  no additional filtering needed in Preprocessor
