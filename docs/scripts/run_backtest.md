# Script: run_backtest.py

**File:** `scripts/run_backtest.py`

---

## Role

CLI entry point and callable module for backtest simulation.
Loads test data, applies model predictions, runs BacktestEngine,
writes results to trade_log and experiment_log tables.

`run_backtest.py` is the **sole writer** of the `experiment_log` table.
Called by PipelineOptimizer after AUC loop converges and a winning trial is selected.
Also runnable standalone.

---

## Input / Output

**Input:**
```python
test_df: pd.DataFrame   # test_features.parquet with features, labels, p_entry
model_dir: Path | None  # directory containing trained model artifacts (optional)
```

**Output:**
```python
# Printed to stdout:
#   - Total trades, winning trades, winning rate
#   - Avg PnL, total PnL, avg slippage
#   - Dead position count and rate
#   - Trade breakdown by signal and exit reason
#   - Label base rates (from test_df directly)

# Written to DuckDB:
#   - trade_log table: individual trade records
#   - experiment_log table: summary row (sole writer)
```

---

## Pipeline Steps

```
1. Load config from pipeline_config.yaml
2. Load test_features.parquet from data/processed/ (or accept in-memory DataFrame)
3. Determine feature column names (exclude labels, identifiers)
4. If model_dir provided:
   a. model = create_model(config); models, meta = model.load(run_id)
   b. probs_df = model.predict(models, X_test)
   c. Build predictions DataFrame (ticker, date, hour, p_entry, probs, labels)
   Else:
   a. Use test_df columns directly (prob_* and label_* must exist)
5. Instantiate BacktestEngine(config, db_conn)
6. trade_log, summary = engine.run(predictions)
7. Compute label base rates from test_df:
       base_rates = {label: test_df[f"label_{label}"].mean() for label in LABELS}
8. Print results to stdout
9. Write trade_log to DuckDB trade_log table
10. Write experiment_log row to DuckDB:
        run_id, optimizer_run_id, run_at,
        winning_rate, total_trades, winning_trades,
        avg_pnl_pct, total_pnl_abs, avg_slippage_pct,
        dead_position_count, dead_position_rate,
        trades_by_signal (JSON), trades_by_exit (JSON)
```

---

## Class Interface

`run_backtest.py` exposes a callable class for use by PipelineOptimizer:

```python
class Backtester:
    def __init__(
        self,
        config: dict,
        db_conn: duckdb.DuckDBPyConnection,
        optimizer_run_id: str | None = None,
    ): ...

    def run(
        self,
        test_df: pd.DataFrame,
        run_id: str,
        model_dir: Path | None = None,
    ) -> dict:
        """
        Run backtest, write trade_log and experiment_log.
        Returns summary dict.
        optimizer_run_id passed via constructor (None for standalone).
        """
        ...
```

---

## CLI Interface

```bash
python scripts/run_backtest.py [OPTIONS]

Options:
    --config, -c        Path to pipeline_config.yaml (default: configs/pipeline_config.yaml)
    --data-dir, -d      Directory containing test_features.parquet (default: data/processed)
    --model-dir, -m     Directory containing trained model artifacts (optional)
    --run-id            Explicit run ID (default: YYYYMMDD_HHMMSS via utils.generate_run_id())
```

CLI always runs with `optimizer_run_id=None` (standalone mode).

---

## experiment_log Write

`run_backtest.py` is the sole writer of `experiment_log`. Row is always INSERT (never upsert).
`run_id` must not already exist in `experiment_log` — collision raises error.

```python
experiment_log_row = {
    "run_id":              run_id,
    "optimizer_run_id":    optimizer_run_id,   # None if standalone
    "run_at":              run_at,
    "winning_rate":        summary["winning_rate"],
    "total_trades":        summary["total_trades"],
    "winning_trades":      summary["winning_trades"],
    "avg_pnl_pct":         summary["avg_pnl_pct"],
    "total_pnl_abs":       summary["total_pnl_abs"],
    "avg_slippage_pct":    summary["avg_slippage_pct"],
    "dead_position_count": summary["dead_position_count"],
    "dead_position_rate":  summary["dead_position_rate"],
    "trades_by_signal":    json.dumps(summary["trades_by_signal"]),
    "trades_by_exit":      json.dumps(summary["trades_by_exit"]),
}
```

---

## Output Format

```
BACKTEST RESULTS
============================================================

Total trades:        1234
Winning trades:      567
Winning rate:        45.9%
Avg PnL:             0.02%
Total PnL:           1234.56
Avg slippage:        0.0012%
Dead positions:      12  (0.97%)

--- By Signal ---
  up3: 456 trades, win_rate=44.5%, avg_pnl=0.01%
  up5: 778 trades, win_rate=46.8%, avg_pnl=0.03%

--- By Exit Reason ---
  take_profit:              567
  stop_loss:                456
  session_end:              143
  time_limit:                56
  dead_position:             10
  dead_position_delisted:     1
  dead_position_no_data:      1

--- Label Base Rates (test set) ---
  up5:  18204 events, base_rate=6.4%
  up3:  52811 events, base_rate=18.5%
  sw:   ...
  dn3:  ...
  dn5:  ...

Run ID: 20250715_143022
Elapsed: 45.2s
```

---

## Constraints

- `experiment_log` is written only by `run_backtest.py` — no other module writes to it
- INSERT only — no upsert. run_id collision raises error
- Model loading via BaseModel interface: `create_model → load → predict`
- `test_df` must contain `p_entry` column (loaded from parquet identifier columns)
- Label base rates computed from `test_df` directly — not from BacktestEngine summary
- `optimizer_run_id` passed via constructor; `None` for standalone CLI runs
- `run_id` generated via `utils.generate_run_id()` if not explicitly provided
