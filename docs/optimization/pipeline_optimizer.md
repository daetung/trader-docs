# Module: PipelineOptimizer

**File:** `src/optimization/optimizer.py`
**Depends on:** all pipeline modules

---

## Role

Training endpoint for the auto-scalping pipeline.
Manages the full preprocess → train → AUC loop → backtest cycle.
Searches feature combinations to find the pipeline configuration with the best winning rate.
All results logged to DuckDB.

---

## Architecture Position

```
Training endpoint:  PipelineOptimizer
    └── Preprocessor.run(return_data=True)   ← in-memory, no parquet save
    └── Trainer.run(...)                      ← AUC loop, writes train_log
    └── Backtester.run(...)                   ← writes experiment_log

Live inference endpoint: Inferencer (separate module)
    └── Preprocessor.run(live_mode=True)
    └── model.load() → resolve_signal()
```

PipelineOptimizer imports and calls `Preprocessor`, `Trainer`, `Backtester` classes directly.
It does not invoke run scripts as subprocesses.

---

## Scope of Automation

```
AUTOMATED (this module):
    Feature group ON/OFF combinations
    → which indicator groups are active
    → which vectorizer methods are applied per indicator group
    → importance_averaging strategy ("uniform" | "sample_weighted")

MANUAL (edit pipeline_config.yaml directly):
    Individual indicator parameters
    Entry detection thresholds
    LightGBM hyperparameters
    Backtest settings
    Class balancer ratios
```

---

## Search Space Definition

Defined in `pipeline_config.yaml` under `optimizer.search_space`:

```yaml
optimizer:
  auc_threshold: 0.65          # minimum mean AUC to proceed to backtest
  max_trials: 50
  random_state: 42
  min_features: 5
  strategy: "grid"             # "grid" | "random"

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

    dimensionality_reducer:
      importance_averaging:
        options:
          - "uniform"
          - "sample_weighted"
```

---

## Full Experiment Cycle

```
optimizer_run_id = utils.generate_run_id()

For each feature_config in search_space:

    [Preprocess — in-memory]
    preprocessor = Preprocessor(config_override, db_conn)
    train, val, test = preprocessor.run(return_data=True)

    [AUC Loop]
    best_run_id = None
    best_auc    = 0.0

    trainer = Trainer(config_override, db_conn, optimizer_run_id=optimizer_run_id)
    run_id  = utils.generate_run_id()
    result  = trainer.run(train, val, test, run_id=run_id, feature_config=feature_config)

    if result["auc_mean"] >= auc_threshold:
        best_run_id = run_id
        best_auc    = result["auc_mean"]
        → proceed to backtest
    else:
        → adjust feature_config or retry up to max_trials
        → if max_trials exhausted or search space depleted:
              best_run_id = run_id with highest auc_mean so far
              → proceed to backtest with best available

    [Mark best trial in train_log]
    UPDATE train_log SET best_of_loop = TRUE WHERE run_id = best_run_id

    [Backtest]
    backtester = Backtester(config_override, db_conn, optimizer_run_id=optimizer_run_id)
    summary    = backtester.run(test, run_id=best_run_id)

    [Save preprocessed data for best trial]
    preprocessor.save(train, val, test, run_id=best_run_id)

    [Print trial summary]
    print(f"run_id={best_run_id} auc={best_auc:.4f} winning_rate={summary['winning_rate']:.3f}")
```

---

## AUC Loop Termination Conditions

```
Loop exits when any of the following:

1. auc_mean >= auc_threshold        → proceed to backtest immediately
2. max_trials exceeded              → use best auc_mean trial for backtest
3. Search space fully exhausted     → use best auc_mean trial for backtest
```

---

## optimizer_run_id Usage

- Generated once per `PipelineOptimizer.run()` call via `utils.generate_run_id()`
- Passed to all `Trainer` and `Backtester` instances within the same run
- Stored in `train_log.optimizer_run_id` and `experiment_log.optimizer_run_id`
- Enables grouping of all trials belonging to the same optimizer execution

---

## Interface

```python
class PipelineOptimizer:
    def __init__(
        self,
        config: dict,
        db_conn: duckdb.DuckDBPyConnection,
    ): ...

    def run(self) -> pd.DataFrame:
        """
        Execute full search across all feature configs.
        Returns experiment_log rows as DataFrame, sorted by winning_rate descending.
        """
        ...

    def run_single(
        self,
        feature_config: dict,
        run_id: str | None = None,
    ) -> dict:
        """
        Run one full experiment cycle for a given feature config.
        Returns summary dict (also written to experiment_log).
        """
        ...

    def best_config(self) -> dict:
        """
        Query experiment_log for the config with highest winning_rate
        belonging to this optimizer_run_id.
        """
        ...
```

---

## Config Override Mechanism

PipelineOptimizer applies feature_config overrides to config in-memory.
`pipeline_config.yaml` is never modified.

```python
config_override = utils.apply_overrides(base_config, feature_config)
```

---

## Constraints

- Each trial independently reproducible given feature_config and random_state
- All results written to DuckDB immediately after each trial — crash-safe
- Optimizer does NOT modify `pipeline_config.yaml`
- `run_single()` callable standalone from CLI for manual one-off experiments
- `train_log` written by Trainer; `experiment_log` written by Backtester
- `best_of_loop` updated via UPDATE by PipelineOptimizer after loop completes
- Preprocessed parquet saved only for best trial via `preprocessor.save()`
- `optimizer_run_id` format: `YYYYMMDD_HHMMSS` (same as run_id, generated once per optimizer run)
