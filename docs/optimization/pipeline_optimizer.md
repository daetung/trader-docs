# Module: PipelineOptimizer

**File:** `src/optimization/optimizer.py`
**Depends on:** all pipeline modules

---

## Role

Automate feature combination search to find the pipeline configuration
with the best winning rate. Manages the full train→evaluate→backtest cycle
per experiment run and logs all results to DuckDB.

---

## Scope of Automation

```
AUTOMATED (this module):
    Feature group ON/OFF combinations
    → which indicator groups are active (price, volume, tick, sr_levels, etc.)
    → which vectorizer methods are applied per indicator group

MANUAL (edit pipeline_config.yaml directly):
    Individual indicator parameters  (e.g. RSI window, BB k-value)
    Entry detection thresholds       (e.g. filter B min_volume)
    LightGBM hyperparameters
    Backtest settings
    Class balancer ratios
```

---

## Search Space Definition

Defined in `pipeline_config.yaml` under `optimizer.search_space`.

```yaml
optimizer:
  search_space:
    feature_groups:
      - name: price_indicators
        options: [on, off]
      - name: volume_indicators
        options: [on, off]
      - name: tick_indicators
        options: [on, off]
      - name: sr_levels
        options: [on, off]
      - name: meta_features
        options: [on, off]
      - name: temporal_features
        options: [on, off]
      - name: fibonacci
        options: [on, off]
      - name: pivot_points
        options: [on, off]

    vectorizer_methods:
      price_close:
        options:
          - [rate_of_change]
          - [rate_of_change, linear_trend]
          - [rate_of_change, shape_features]
      volume:
        options:
          - [window_comparison]
          - [window_comparison, statistical_summary]

  strategy: grid          # "grid" | "random"
  max_trials: 50          # ignored for grid if total combinations < 50
  random_state: 42
  min_features: 5         # skip configs producing fewer features than this
```

Grid search is default. Random search activates when `max_trials < total_combinations`.

---

## Experiment Cycle (per trial)

```
1. Apply feature config from search space
         ↓
2. Run FeatureExtractor.extract_batch() for all entry points
         ↓
3. Run ClassBalancer.split() → train_balanced, val, test
         ↓
4. Run model.train(train, val, feature_names, categorical_cols) + model.evaluate(models, test, feature_names)
         ↓
5. Run DimensionalityReducer.run_all()
   → if reduced AUC within tolerance: retrain on reduced feature set
         ↓
6. Run BacktestEngine.run() on test split predictions
         ↓
7. Log results to DuckDB experiment_log
         ↓
8. Print trial summary to stdout
```

---

## Experiment Log Schema (DuckDB)

Table: `experiment_log` (authoritative definition in `db_schema.md`, reproduced here for reference)

```sql
CREATE TABLE experiment_log (
    run_id              VARCHAR   PRIMARY KEY,   -- YYYYMMDD_HHMMSS_NNN (NNN=trial index)
    run_at              VARCHAR   NOT NULL,
    feature_config      VARCHAR   NOT NULL,      -- JSON: active groups + vectorizer methods
    n_features          INTEGER,
    n_features_reduced  INTEGER,                 -- after DimensionalityReducer; NULL if not run
    auc_up5             DOUBLE,
    auc_up3             DOUBLE,
    auc_sw              DOUBLE,
    auc_dn3             DOUBLE,
    auc_dn5             DOUBLE,
    auc_mean            DOUBLE,
    auc_reduced_mean    DOUBLE,                  -- mean AUC on reduced feature set; NULL if not run
    winning_rate        DOUBLE,
    total_trades        INTEGER,
    winning_trades      INTEGER,
    avg_pnl_pct         DOUBLE,
    total_pnl_abs       DOUBLE,
    avg_slippage_pct    DOUBLE,
    trades_by_signal    VARCHAR,                 -- JSON: {"up3": {...}, "up5": {...}}
    trades_by_exit      VARCHAR,                 -- JSON: {"take_profit": n, "stop_loss": n, "time_limit": n}
    notes               VARCHAR
);
```

---

## Interface

```python
class PipelineOptimizer:
    def __init__(
        self,
        config: dict,
        db_conn: duckdb.DuckDBPyConnection,
    ): ...

    def run(
        self,
        entry_points: pd.DataFrame,    # all labeled entry points (full dataset)
        ohlcv: dict[str, pd.DataFrame],
        ticks: dict[str, pd.DataFrame],
        meta_df: pd.DataFrame,
    ) -> pd.DataFrame:
        """
        Execute full search. Returns experiment_log as DataFrame,
        sorted by winning_rate descending.
        """
        ...

    def run_single(
        self,
        feature_config: dict,
        entry_points: pd.DataFrame,
        ohlcv: dict[str, pd.DataFrame],
        ticks: dict[str, pd.DataFrame],
        meta_df: pd.DataFrame,
        run_id: str | None = None,
    ) -> dict:
        """
        Run one full experiment cycle for a given feature config.
        Returns the result row dict (also written to DuckDB).
        """
        ...

    def best_config(self) -> dict:
        """
        Query experiment_log for the config with highest winning_rate.
        """
        ...
```

---

## Detection Parameter Search (Separate Tool)

Entry detection threshold tuning is handled by a separate lightweight tool,
not inside PipelineOptimizer, because it does not require model training.

See: `tools/detection_benchmark.py`

---

## Constraints

- Each trial must be independently reproducible given its `feature_config` and `random_state`
- All results written to DuckDB immediately after each trial (not batched) — crash-safe
- Optimizer does NOT modify `pipeline_config.yaml` — it reads and overrides in-memory only
- `run_single()` can be called standalone from CLI for manual one-off experiments
- `feature_config` column stores JSON string of active groups and vectorizer methods (not `feature_set`)
