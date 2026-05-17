# Module: run_train.py

**File:** `scripts/run_train.py`

---

## Role

CLI entry point and callable module for LightGBM training.
Loads preprocessed train/val/test data, trains 5-class model, evaluates on test set,
saves model artifacts, and writes results to train_log table.

Called by PipelineOptimizer as part of the AUC loop.
Also runnable standalone for single training runs.

---

## Input / Output

**Input:**
```python
# From preprocessed parquet or in-memory DataFrames:
train_df: pd.DataFrame   # balanced training split
val_df:   pd.DataFrame   # validation split
test_df:  pd.DataFrame   # test split
# All DataFrames contain:
# [ticker, date, hour, p_entry, features..., label_up5..label_dn5, is_dead_position]
```

**Output:**
```python
# Printed to stdout:
#   - Per-class validation AUC (during training)
#   - Per-class test AUC + mean test AUC
#   - Feature importance top-10 per classifier
#   - Model save location

# Saved to models/lightgbm/{run_id}/:
#   {run_id}_up5.lgb, {run_id}_up3.lgb, {run_id}_sw.lgb,
#   {run_id}_dn3.lgb, {run_id}_dn5.lgb
#   {run_id}_meta.json

# Written to DuckDB train_log table:
#   one row per training run
```

---

## Training Pipeline Steps

```
1. Load config from pipeline_config.yaml
2. Load train/val/test (from parquet or in-memory DataFrames)
3. Apply session_mode filter to training data:
       if session_mode == "regular": keep hour 093000~155900 only
       if session_mode == "pre":     keep hour 040000~092900 only
       if session_mode == "combined": keep hour < 160000
4. Build feature_names via FeatureExtractor.get_feature_names()
5. Identify categorical_cols: features ending with "_code" + "day_of_week" + "halt_reason_code"
6. Instantiate model via factory: model = create_model(config)
7. Train 5 classifiers:
       models = model.train(train_df, val_df, feature_names, categorical_cols)
8. Evaluate on test set:
       eval_result = model.evaluate(models, test_df, feature_names)
9. Print per-class AUC results
10. Print top-10 features per classifier:
        model.feature_importance(models)
11. Save model artifacts:
        model.save(models, run_id, eval_result, feature_names, categorical_cols)
12. Optionally run DimensionalityReducer if --reduce flag is set:
        provider = create_importance_provider(...)
        selected_features, report = reducer.run_all(X_train, X_val, provider)
        Retrain on selected_features → eval_result_reduced
        Compare AUC: if within tolerance → save reduced model with suffix "_reduced"
13. Write to train_log (DuckDB):
        run_id, optimizer_run_id, run_at, feature_config,
        n_features, n_features_reduced, auc_*, auc_mean,
        auc_reduced_mean, best_of_loop=False, notes
```

---

## Class Interface

`run_train.py` exposes a callable class for use by PipelineOptimizer:

```python
class Trainer:
    def __init__(
        self,
        config: dict,
        db_conn: duckdb.DuckDBPyConnection,
        optimizer_run_id: str | None = None,
    ): ...

    def run(
        self,
        train_df: pd.DataFrame,
        val_df: pd.DataFrame,
        test_df: pd.DataFrame,
        run_id: str | None = None,
        feature_config: dict | None = None,
    ) -> dict:
        """
        Train, evaluate, save artifacts, write to train_log.
        Returns eval_result dict.
        optimizer_run_id passed via constructor (None for standalone).
        run_id generated via utils.generate_run_id() if not provided.
        feature_config: active feature group config for logging (None = full config).
        """
        ...
```

---

## CLI Interface

```bash
python scripts/run_train.py [--config CONFIG] [--data-dir DIR] [--run-id ID] [--reduce]

Options:
    --config        Path to pipeline_config.yaml (default: configs/pipeline_config.yaml)
    --data-dir      Directory containing train/val/test parquet files (default: data/processed)
    --run-id        Explicit run ID (default: YYYYMMDD_HHMMSS via utils.generate_run_id())
    --reduce        Run DimensionalityReducer after initial training
```

CLI always runs with `optimizer_run_id=None` (standalone mode).

---

## Session Mode Filtering

`session_mode` from `config["entry_detector"]["session_mode"]` is applied to training data:

```python
if session_mode == "regular":
    train_df = train_df[
        (train_df["hour"] >= "093000") & (train_df["hour"] <= "155900")
    ]
elif session_mode == "pre":
    train_df = train_df[
        (train_df["hour"] >= "040000") & (train_df["hour"] < "093000")
    ]
elif session_mode == "combined":
    train_df = train_df[train_df["hour"] < "160000"]
```

Val and test sets are not filtered — they retain all entry points for consistent evaluation.

---

## train_log Write

Written after every training run (standalone or optimizer loop):

```python
train_log_row = {
    "run_id":              run_id,
    "optimizer_run_id":    optimizer_run_id,   # None if standalone
    "run_at":              run_at,
    "feature_config":      json.dumps(feature_config),
    "n_features":          len(feature_names),
    "n_features_reduced":  len(selected_features) if reduced else None,
    "auc_up5":             eval_result["up5"]["test_auc"],
    "auc_up3":             eval_result["up3"]["test_auc"],
    "auc_sw":              eval_result["sw"]["test_auc"],
    "auc_dn3":             eval_result["dn3"]["test_auc"],
    "auc_dn5":             eval_result["dn5"]["test_auc"],
    "auc_mean":            eval_result["mean_test_auc"],
    "auc_reduced_mean":    eval_result_reduced["mean_test_auc"] if reduced else None,
    "best_of_loop":        False,   # updated by PipelineOptimizer after loop completes
}
```

`best_of_loop` is set to `True` by PipelineOptimizer via UPDATE after the AUC loop
selects the best trial to proceed to backtest.

---

## Constraints

- All parameters from `pipeline_config.yaml` — nothing hardcoded
- Must not modify the preprocessed data files
- Must print summary results even if dimensionality reduction is run
- Model artifacts must include config snapshot for reproducibility
- Must handle empty DataFrames gracefully (exit with error message)
- Variable names `model` (BaseModel instance) and `models` (trained booster dict) must not be mixed
- `run_id` generated via `utils.generate_run_id()`
- `experiment_log` is NOT written by run_train.py — only train_log
- `best_of_loop` updated by PipelineOptimizer, not by Trainer itself
