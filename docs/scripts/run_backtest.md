# Script: run_backtest.py

**File:** `scripts/run_backtest.py`

---

## Role

CLI entry point and callable module for backtest simulation.
Loads test data, applies model predictions, runs BacktestEngine,
writes results to trade_log and experiment_log tables.

`run_backtest.py` is the **sole writer** of the `experiment_log` table.
Called by PipelineOptimizer after the exploitation/full phase completes
and the best trial or consensus config is selected. Also runnable standalone.

---

## Input / Output

**Input:**
```python
test_df: pd.DataFrame   # test split with features, labels, p_entry
model_dir: Path | None  # directory containing trained model artifacts (optional)
```

**Output:**
```python
# Printed to stdout:
#   - Total trades, winning trades, winning rate
#   - Avg PnL, total PnL, avg slippage
#   - Dead position count and rate
#   - Suppressed entry count
#   - Trade breakdown by signal and exit reason
#   - Label base rates (from test_df directly)
#   - Dead position case distribution (from test_df)

# Written to DuckDB:
#   - trade_log table: individual trade records
#   - experiment_log table: summary row (sole writer)
```

## Pipeline Steps

```
1. Load config from pipeline_config.yaml
2. Load test_features.parquet from data/processed/ (or accept in-memory DataFrame)
3. Determine feature column names (exclude labels, identifiers, metadata columns)
   Metadata columns excluded from features: is_dead_position, dead_position_case, is_ambiguous
4. Resolve model source and build predictions DataFrame:
   a. If model_dir is explicitly provided (non-None):
      → load from model_dir / run_id
   b. Elif config["model"]["model_dir"] exists and is non-empty:
      → model_dir = Path(config["model"]["model_dir"])
      → load from config model_dir / run_id
      (covers optimizer-called context where model_dir=None but artifacts were saved by Trainer)
   c. Else (pre-computed probabilities — pure standalone path):
      → Use test_df columns directly (prob_* and label_* must exist)

   For paths (a) and (b):
      model = create_model(config); models, meta = model.load(run_id)
      probs_df = model.predict(models, X_test)
      Build predictions DataFrame (ticker, date, hour, p_entry, probs, labels)
5. Instantiate BacktestEngine(config, db_conn)
6. trade_log, summary = engine.run(predictions)
7. Compute label base rates from test_df:
       base_rates = {label: test_df[f"label_{label}"].mean() for label in LABELS}
8. Compute dead position case distribution from test_df:
       dead_case_dist = test_df[test_df["is_dead_position"]]["dead_position_case"].value_counts()
9. Print results to stdout
10. Write trade_log to DuckDB trade_log table
11. Write experiment_log row to DuckDB — full field list in
        "experiment_log Write" below (kept 1:1 with summary_dict; see
        Constraints)
```

---

## Class Interface

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
        test_df:         pd.DataFrame,
        run_id:          str,
        model_dir:       Path | None = None,
        fold_idx:        int = -1,
        outer_fold_idx:  int = -1,
        fold_test_start: str | None = None,
        fold_test_end:   str | None = None,
        eval_type:       str | None = None,
        execution_variant: str | None = None,
    ) -> dict:
        """
        Run backtest, write trade_log and experiment_log.
        Returns summary dict.

        Args:
            test_df:         test split DataFrame
            run_id:          matches the train_log run_id for this backtest
            model_dir:       directory containing model artifacts.
                             Resolution order:
                               1. model_dir if explicitly provided (non-None)
                               2. Path(config["model"]["model_dir"]) if set in config
                               3. Use test_df prob_* columns directly (pre-computed)
            fold_idx:        inner rolling fold index (0-based); -1 for standalone/outer eval
            outer_fold_idx:  outer fold index for nested validation (0-based);
                             -1 for non-nested runs
            fold_test_start: first date of test window ('YYYYMMDD');
                             None for standalone (derived from test_df date range)
            fold_test_end:   last date of test window ('YYYYMMDD');
                             None for standalone (derived from test_df date range)
            eval_type:       None               → standard backtest (standalone or non-nested)
                             "outer_validation" → nested validation outer fold evaluation
                             "regime_holdout"   → regime holdout robustness check
            execution_variant: JSON of the execution-key overrides this run
                             was given, or None for the baseline. Set by
                             PipelineOptimizer's execution_eval loop; always
                             None from the CLI. An IDENTIFIER, not a metric —
                             see Constraints on why it sits outside the 1:1
                             rule.

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

CLI always runs with:
```
optimizer_run_id=None, fold_idx=-1, outer_fold_idx=-1,
eval_type=None, execution_variant=None (standalone mode)
```

---

## experiment_log Write

`run_backtest.py` is the sole writer of `experiment_log`. Row is always INSERT (never upsert).
`run_id` must not already exist in `experiment_log` — collision raises error.

```python
experiment_log_row = {
    "run_id":               run_id,
    "optimizer_run_id":     optimizer_run_id,    # None if standalone
    "run_at":               run_at,
    "fold_idx":             fold_idx,            # -1 if standalone/outer eval
    "outer_fold_idx":       outer_fold_idx,      # -1 if non-nested
    "eval_type":            eval_type,           # None | "outer_validation" |
                                                 # "regime_holdout"
    "execution_variant":    execution_variant,   # None = baseline pass
    "fold_test_start":      fold_test_start,     # None → derived from test_df.date.min()
    "fold_test_end":        fold_test_end,       # None → derived from test_df.date.max()
    "winning_rate":         summary["winning_rate"],
    "total_trades":         summary["total_trades"],
    "winning_trades":       summary["winning_trades"],
    "avg_pnl_pct":          summary["avg_pnl_pct"],
    "total_pnl_abs":        summary["total_pnl_abs"],
    "avg_slippage_pct":     summary["avg_slippage_pct"],
    "dead_position_count":  summary["dead_position_count"],
    "dead_position_rate":   summary["dead_position_rate"],
    "suppressed_count":     summary["suppressed_count"],
    "gate_blocked_cap_tickers":     summary["gate_blocked_cap_tickers"],
    "gate_blocked_cap_per_ticker":  summary["gate_blocked_cap_per_ticker"],
    "gate_blocked_cooldown":        summary["gate_blocked_cooldown"],
    "gate_blocked_sizing_zero":     summary["gate_blocked_sizing_zero"],
    "gate_blocked_funds":           summary["gate_blocked_funds"],
    "breaker_max_realized_loss_abs":  summary["breaker_max_realized_loss_abs"],
    "breaker_max_realized_loss_pct":  summary["breaker_max_realized_loss_pct"],
    "breaker_max_consecutive_losses": summary["breaker_max_consecutive_losses"],
    "breaker_peak_entries_per_hour":  summary["breaker_peak_entries_per_hour"],
    "trades_by_signal":     json.dumps(summary["trades_by_signal"]),
    "trades_by_exit":       json.dumps(summary["trades_by_exit"]),
}
```

`fold_test_start` and `fold_test_end` are sourced from fold_meta when called by
PipelineOptimizer: `fold_meta["fold_test_start"]`, `fold_meta["fold_test_end"]`.
In standalone mode, these are derived from `test_df["date"].min()` and `.max()`.

---

## Output Format

```
BACKTEST RESULTS
============================================================

Total trades:        1234        # opened trades only — never-opened rows
                                 # (entry_canceled / entry_rejected) are
                                 # excluded from this and every rate below
Winning trades:      567
Winning rate:        45.9%
Avg PnL:             0.02%
Total PnL:           1234.56
Avg slippage:        0.0012%
Dead positions:      12  (0.97%)
Suppressed entries:  45

--- Entries Blocked at Gates ---
  max_tickers:              31
  max_positions_per_ticker: 12
  cooldown:                 88
  sizing (quantity=0):       4
  funds:                     0   # always 0 when initial_cash=0 — gate not called

--- Circuit Breaker Metrics (computed, not enforced here) ---
  Max realized loss:        -412.30  (-1.8%)
  Max consecutive losses:   6
  Peak entries/hour:        14

--- By Signal ---
  up3: 456 trades, win_rate=44.5%, avg_pnl=0.01%
  up5: 778 trades, win_rate=46.8%, avg_pnl=0.03%

--- By Exit Reason ---
  take_profit:              567
  stop_loss:                456
  session_end:              143
  time_limit:                56
  dead_position:              8
  dead_position_delisted:     2
  dead_position_no_data:      1
  dead_position_extended_halt: 1
  entry_canceled:             9   # never-opened: listed here, excluded from
                                  # Total trades / Winning rate / Avg PnL

--- Label Base Rates (test set) ---
  up5:  18204 events, base_rate=6.4%
  up3:  52811 events, base_rate=18.5%
  sw:   ...
  dn3:  ...
  dn5:  ...

--- Dead Position Case Distribution (test set) ---
  Case A (next-day data available):  8
  Case B (ticker missing next day):  2
  Case C (dataset boundary):         1
  Case D (extended halt):            1

Run ID: 20250715_143022
Fold:   -1 (standalone)
Elapsed: 45.2s
```

---

## Constraints

- `experiment_log` is written only by `run_backtest.py` — no other module writes to it
- INSERT only — no upsert; run_id collision raises error
- Model loading via BaseModel interface: `create_model → load → predict`
- `test_df` must contain `p_entry` column (loaded from parquet identifier columns)
- Label base rates computed from `test_df` directly — not from BacktestEngine summary
- Dead position case distribution computed from `test_df` directly
- `optimizer_run_id` passed via constructor; `None` for standalone CLI runs
- `run_id` generated via `utils.generate_run_id()` if not explicitly provided
- `fold_idx` default -1 for standalone CLI runs; 0-based for rolling inner folds
- `outer_fold_idx` default -1 for non-nested runs; 0-based for nested outer folds
- `eval_type` default None for standalone; set explicitly by PipelineOptimizer
- In standalone mode, `fold_test_start`/`fold_test_end` derived from test_df date range
- `execution_variant` is an IDENTIFIER column, alongside `run_id`,
  `fold_idx`, `outer_fold_idx` and `eval_type` — it sits OUTSIDE the 1:1
  rule below, which binds `summary_dict` keys to METRIC columns only.
  Worth stating because the distinction is easy to misread: a new column
  that is not in `summary_dict` looks like a violation and is not one. It
  is written by adding a column to an existing table, which is
  `init_db.md`'s class 2 — re-transcribe, then `ALTER TABLE ... ADD COLUMN`
  by hand; plain init will not do it
- `summary_dict`'s key set and `experiment_log`'s metric columns are 1:1 —
  every key BacktestEngine's `run()` returns has a matching column here,
  and every metric column here is sourced from a `summary_dict` key. A name
  on only one side is a defect, not a style choice; this single rule
  replaces enumerating each metric's "must be present" individually as the
  metric count grows (currently: `suppressed_count`, the five
  `gate_blocked_*` counters, and the four `breaker_*` metrics)
- The `gate_blocked_*` and `breaker_*` metrics are diagnostic only —
  `PipelineOptimizer.best_config()` ranks on `winning_rate` and never reads
  them. `best_execution_variant()` DOES rank on a metric column
  (`avg_pnl_pct`), but only across rows sharing an `optimizer_run_id` and an
  outer fold, so it compares execution variants rather than model configs
  and does not change what `best_config()` selects. The gate counters exist so a run whose edge was capped out is
  distinguishable from one that simply produced few signals, which
  `winning_rate` alone cannot show; the breaker metrics exist so Pilot has
  real distributions to calibrate `execution.intraday_loss_limit_pct` /
  `consecutive_loss_limit` / `entries_per_hour_limit` against before they
  are armed (see shadow_retraining.md)
- `gate_blocked_funds` and `breaker_max_realized_loss_pct` are both
  conditioned on `backtest.initial_cash > 0`: at the `0` default,
  `check_funds_available()` is never called (so the former is always 0 —
  "the gate did not run", not "nothing was blocked") and there is no
  equity basis to express a loss as a percentage against (so the latter is
  `None`, not `0.0`, which would misread as "no loss")
- Never-opened rows (`exit_reason='entry_canceled'`; live also
  `'entry_rejected'`) are written to `trade_log` but excluded from
  `total_trades` / `winning_rate` / `avg_pnl_pct` / `total_pnl_abs` —
  `quantity=0` and `pnl_pct=NULL` mean no position existed, so counting them
  would dilute the rates with non-events. They stay visible through
  `trades_by_exit`, which is why no separate summary column is added for
  them (see 09_backtest_engine.md)
- Metadata columns (`is_dead_position`, `dead_position_case`, `is_ambiguous`) must not
  be passed to model.predict() as features
- `model_dir` resolution order: explicit arg → config["model"]["model_dir"] fallback →
  pre-computed prob_* columns in test_df; path (c) requires prob_* columns to exist
- PipelineOptimizer calls `backtester.run()` with `model_dir=None` — the config fallback
  path (b) ensures artifacts saved by Trainer are correctly located at inference time
