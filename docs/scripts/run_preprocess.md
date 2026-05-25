# Script: run_preprocess.py

**File:** `scripts/run_preprocess.py`

---

## Role

CLI entry point that orchestrates the full preprocessing pipeline.
Supports two execution modes:
- **Standalone**: full pipeline, saves parquet to disk
- **Optimizer (in-memory)**: returns full labeled DataFrame without splitting or saving

Preprocessor is a training-only component. Live inference uses
`FeatureExtractor.extract()` directly via Inferencer — not Preprocessor.

Session_mode filtering, temporal splitting, and rolling fold generation
are handled entirely by ClassBalancer.

---

## Input / Output

**Input:**
```python
# From DuckDB:
ohlcv_1min:           pd.DataFrame   # all sessions (no time filter)
tick_10:              pd.DataFrame   # all sessions — full day per ticker/date
stock_meta:           pd.DataFrame
trading_halts:        pd.DataFrame
trading_calendar:     pd.DataFrame
ticker_data_coverage: pd.DataFrame
```

**Output (standalone mode):**
```python
# Saved to data/processed/:
train_features.parquet
val_features.parquet
test_features.parquet
```

**Output (in-memory mode, return_data=True):**
```python
pd.DataFrame  # full labeled feature matrix, unsplit
```

---

## Pipeline Steps

```
[Standard / In-memory mode]

1. Connect to DuckDB and load all data:
       ohlcv_df    = SELECT * FROM ohlcv_1min   ORDER BY ticker, date, hour
       ticks_df    = SELECT * FROM tick_10       ORDER BY ticker, date, hour, seq_id
       meta_df     = SELECT * FROM stock_meta
       halts_df    = SELECT * FROM trading_halts
       calendar_df = SELECT * FROM trading_calendar
       coverage_df = SELECT * FROM ticker_data_coverage

2. EntryPointDetector.scan() for each ticker → entry_points (all sessions)
   max_entry_hour exclusion applied inside scan()
   scan() retrieves p_entry from bars[i+1]["open"] (t bar open price)

3. Save entry_points to DuckDB entry_points table (INSERT OR IGNORE)

4. Labeler.label() for all entry points → labeled_samples
       Per ticker/date:
           ohlcv_future_td = ohlcv_df filtered to (ticker, date, hour >= t_hour)
           ticks_td        = ticks_df filtered to (ticker, date), full day
           halts_td        = halts_df filtered to (ticker, date)
       labeler.label(
           entry_points_td, ohlcv_future_td, ticks_td, halts_td,
           calendar_df, coverage_df
       )
       (includes is_dead_position, dead_position_case, is_ambiguous)

5. Save labeled_samples to DuckDB labeled_samples table (INSERT OR IGNORE)

6. FeatureExtractor.extract_batch() for each ticker:
       bars_td   = ohlcv_df filtered to (ticker, date, hour < t_hour)  [t-1 and earlier]
       ticks_td  = ticks_df filtered to (ticker, date, hour < t_hour)  [before t bar]
       halts_td  = halts_df filtered to (ticker, date)
       extractor.extract_batch(entry_points_td, bars_td, ticks_td, meta_td, halts_td)

7. Merge features with labels on (ticker, date, hour)
   → labeled feature matrix:
     [ticker, date, hour, p_entry]
     + [features...]
     + [labels, is_dead_position, dead_position_case, is_ambiguous]
   → contains all sessions; no session_mode filter applied here

8a. If return_data=True (optimizer mode):
    → return full_labeled_df directly (no split, no parquet save)

8b. If return_data=False (standalone mode):
    → ClassBalancer.split(
           balance=config["class_balancer"]["apply_balance"],
           session_mode=config["entry_detector"]["session_mode"]
       ) → (train_balanced, val, test)
    → save parquet files to data/processed/
```

---

## Class Interface

```python
class Preprocessor:
    def __init__(self, config: dict, db_conn: duckdb.DuckDBPyConnection): ...

    def run(
        self,
        return_data: bool = False,
    ) -> pd.DataFrame | None:
        """
        return_data=False : split via ClassBalancer.split(), save parquet, return None.
                            DB tables always written (INSERT OR IGNORE).
        return_data=True  : return full_labeled_df (unsplit), no parquet save.
                            DB tables always written (INSERT OR IGNORE).

        Training mode only. Live inference uses FeatureExtractor.extract() directly
        via Inferencer — Preprocessor is not involved in live inference.
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
```

CLI always runs in standalone mode (`return_data=False`).

---

## Config Keys Used

```yaml
data_paths.*
entry_detector.*
indicators.*
vectorizer.*
labeler.*
class_balancer.*
misc.lookback_bars
```

---

## Constraints

- Preprocessor is responsible for feature extraction and labeling only
- Session_mode filtering, splitting, and fold generation handled by ClassBalancer
- All data loaded from DuckDB in bulk (no per-ticker DB queries during extraction)
- `ticks_df` loaded as full day per ticker/date:
  - Labeler receives full day (hour >= t_hour filtered internally per entry point)
  - FeatureExtractor receives ticks before t bar only (hour < t_hour)
- `halts_df` loaded per ticker/date and passed explicitly to both Labeler and FeatureExtractor
- `coverage_df` loaded once at step 1 and passed explicitly to Labeler.label()
- `extract_batch()` called per ticker — no Python loop over `extract()` per entry point
- Empty DataFrames handled gracefully (saves _empty.parquet)
- `ClassBalancer.split()` called only in standalone mode
- `ClassBalancer.generate_folds()` is never called inside Preprocessor
- `p_entry` included in parquet as identifier column, not feature
- `is_dead_position`, `dead_position_case`, `is_ambiguous` stored as metadata columns
- entry_points and labeled_samples DB saves use INSERT OR IGNORE for idempotency
- `max_entry_hour` exclusion applied inside EntryPointDetector.scan()
- Standalone CLI always calls split() — split_method config enforced by PipelineOptimizer only
- live_mode is NOT supported — Preprocessor is training-only
