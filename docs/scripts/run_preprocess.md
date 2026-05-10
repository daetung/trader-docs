# Script: run_preprocess.py

**File:** `scripts/run_preprocess.py`

---

## Role

CLI entry point that orchestrates the full preprocessing pipeline:
1. Load data from DuckDB (OHLCV, ticks, stock_meta, trading_halts)
2. Run EntryPointDetector.scan() for each ticker -> entry_points
3. Save entry_points to DuckDB
4. Run Labeler.label() for all entry points -> labeled samples
5. Save labels to DuckDB
6. Run FeatureExtractor for each ticker's entries -> feature matrix
7. Merge features + labels -> full dataset
8. Apply ClassBalancer.balance() + .split() -> train/val/test
9. Save feature matrices to data/processed/

---

## Input / Output

**Input:**
```python
# From DuckDB:
ohlcv_1min: pd.DataFrame  # OHLCV 1-min bars
tick_10: pd.DataFrame     # 10-tick data
stock_meta: pd.DataFrame  # stock metadata
trading_halts: pd.DataFrame  # trading halt records
```

**Output:**
```python
# Saved to data/processed/:
train_features.parquet  # balanced training split
val_features.parquet    # validation split
test_features.parquet   # test split
```

---

## Pipeline Steps

```
1. Connect to DuckDB and load all data (OHLCV, ticks, meta, halts)
2. EntryPointDetector.scan() for each ticker -> entry_points
3. Save entry_points to DuckDB entry_points table
4. Labeler.label() for all entry points -> labeled samples
5. Save labeled samples to DuckDB labeled_samples table
6. FeatureExtractor.extract_batch() for each ticker -> feature matrix
7. Merge features with labels on (ticker, date, hour)
8. ClassBalancer.balance() -> downsampling
9. ClassBalancer.split() -> train/val/test
10. Save parquet files to data/processed/
```

---

## CLI Interface

```bash
python scripts/run_preprocess.py [--config CONFIG]

Options:
    --config        Path to pipeline_config.yaml (default: configs/pipeline_config.yaml)
```

---

## Config Keys Used

```yaml
data_paths.*           # all data path settings
entry_detector.*       # entry detection thresholds
indicators.*           # indicator parameters
vectorizer.*           # vectorizer settings
labeler.*              # labeling parameters
class_balancer.*       # balancing parameters
misc.lookback_bars     # passed to FeatureExtractor
```

---

## Constraints

- All data loaded from DuckDB in bulk (no per-ticker DB queries)
- Feature extraction uses extract_batch() — no Python loop over extract()
- Empty DataFrames handled gracefully (saves _empty.parquet)
- ClassBalancer applies balancing to train split only — val/test untouched
- All parameters from pipeline_config.yaml — nothing hardcoded
