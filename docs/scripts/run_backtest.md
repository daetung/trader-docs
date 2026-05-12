# Script: run_backtest.py

**File:** `scripts/run_backtest.py`

---

## Role

CLI entry point that simulates trading performance of trained model predictions.
Loads test data, applies model predictions (or uses test_df columns directly),
and runs the BacktestEngine to compute winning rate and trade statistics.

---

## Input / Output

**Input:**
```python
test_df: pd.DataFrame  # test_features.parquet with features, labels, p_entry
model_dir: Path | None  # directory containing trained model artifacts (optional)
```

**Output:**
```python
# Printed to stdout:
#   - Total trades, winning trades, winning rate
#   - Avg PnL, total PnL, avg slippage
#   - Trade breakdown by signal (up3, up5) and exit reason
#   - Label base rates (computed directly from test_df — not from BacktestEngine)

# Written to DuckDB:
#   - trade_log table: individual trade records
#   - experiment_log table: summary statistics
```

---

## Pipeline Steps

```
1. Load config from pipeline_config.yaml
2. Load test_features.parquet from data/processed/
3. Determine feature column names (exclude labels, probs, identifiers)
4. If model_dir provided:
   a. Load model: model = create_model(config); models, meta = model.load(run_id)
   b. Apply predictions: probs_df = model.predict(models, X_test)
   c. Build predictions DataFrame (ticker, date, hour, p_entry, probs, labels)
   Else:
   a. Use test_df columns directly as predictions
      (prob_* and label_* columns must already exist)
5. Load OHLCV and tick data from DuckDB for prediction tickers
6. Run BacktestEngine.run(predictions, ohlcv, ticks) → (trade_log, summary)
7. Compute label base rates directly from test_df:
       base_rates = {label: test_df[f"label_{label}"].mean() for label in LABELS}
8. Print results to stdout (see Output Format below)
9. Write trade_log to DuckDB trade_log table
10. Write summary to DuckDB experiment_log table (matching run_id)
```

---

## Model Prediction Path

When model_dir is provided:
```
1. Load model via create_model(config) → model.load(run_id)
2. Apply predictions via model.predict(models, X) → prob_* columns
3. Merge labels from test_df
4. Build predictions DataFrame with all required columns
```

When model_dir is omitted:
```
1. Use test_df columns directly (prob_* and label_* columns must exist)
2. p_entry column must be present (warning if missing, defaults to 100.0)
```

---

## CLI Interface

```bash
python scripts/run_backtest.py [OPTIONS]

Options:
    --config, -c        Path to pipeline_config.yaml (default: configs/pipeline_config.yaml)
    --data-dir, -d      Directory containing test_features.parquet (default: data/processed)
    --model-dir, -m     Directory containing trained model artifacts (optional)
    --run-id            Explicit run ID (default: YYYYMMDD_HHMMSS timestamp)
```

---

## Config Keys Used

```yaml
backtest.*              # backtest parameters
data_paths.duckdb_path  # DuckDB database path
data_paths.processed_dir  # test parquet location
misc.random_state
```

---

## Output Format

```
BACKTEST RESULTS
============================================================

Total trades:   1234
Winning trades: 567
Winning rate:   45.9%
Avg PnL:        0.02%
Total PnL:      1234.56
Avg slippage:   0.0012%

--- By Signal ---
  up3: 456 trades, win_rate=44.5%, avg_pnl=0.01%
  up5: 778 trades, win_rate=46.8%, avg_pnl=0.03%

--- By Exit Reason ---
  take_profit: 567
  stop_loss: 456
  time_limit: 211

--- Label Base Rates (test set) ---
  up5: 18204 events, base_rate=6.4%
  up3: 52811 events, base_rate=18.5%
  sw:  ...
  dn3: ...
  dn5: ...

Run ID: 20250715_143022
Elapsed: 45.2s
```

Note: Label base rates are computed by `run_backtest.py` directly from `test_df` label columns.
They are not part of the BacktestEngine summary output.

---

## Constraints

- Model loading and prediction via BaseModel interface: create_model → load → predict
- test_df must contain p_entry column (t bar open price)
- OHLCV and tick data must be available in DuckDB for prediction tickers
- Individual trade records written to trade_log table in DuckDB
- Summary statistics written to experiment_log table in DuckDB
- Label base rates computed in run_backtest.py, not BacktestEngine
- If no model_dir provided, test_df must have prob_* and label_* columns
- BacktestEngine uses 10-tick data for entry slippage, OHLCV H/L for exit pricing
