# Module: run_train.py

**File:** `scripts/run_train.py`

---

## Role

CLI entry point and callable module for LightGBM training.
Loads preprocessed train/val/test data, trains 5-class model, evaluates on test set,
saves model artifacts, and writes results to train_log table.

Called by PipelineOptimizer as part of the rolling fold loop.
Also runnable standalone for single training runs.

---

## Input / Output

**Input:**
```python
train_df: pd.DataFrame   # balanced training split
val_df:   pd.DataFrame   # validation split
test_df:  pd.DataFrame   # test split
# All DataFrames contain:
# [ticker, date, hour, p_entry, features..., label_up5..label_dn5,
#  is_dead_position, dead_position_case, is_ambiguous]
# Session filtering and balancing applied upstream by ClassBalancer.
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

# Written to DuckDB train_log table: one row per fold
```

---

## Training Pipeline Steps

```
1. Load config from pipeline_config.yaml
2. Determine feature_names:
   if feature_names argument provided (exploitation phase): use as-is
   else: FeatureExtractor.get_feature_names() (full feature set)
3. Prepare sample weights for ambiguous samples (Trainer responsibility):
   if config["trainer"]["use_ambiguous_sample_weight"] == True:
       weight_col = "__sample_weight__"
       train_df[weight_col] = 1.0
       train_df.loc[train_df["is_ambiguous"], weight_col] = \
           config["trainer"]["ambiguous_sample_weight"]
       sample_weight_col = weight_col
   else:
       sample_weight_col = None
   Note: is_ambiguous column is used here for weight assignment only —
         it is NOT included in feature_names passed to LightGBM
4. Identify categorical_cols from FeatureExtractor.get_feature_schema():
       extractor = FeatureExtractor(config)
       schema = extractor.get_feature_schema()
       categorical_cols = [
           name for name, ftype in schema.items()
           if ftype == "categorical"
       ]
       # For LightGBM: only "categorical" typed features are registered
       # as categorical_feature. All other types (binary, count, ordinal)
       # are treated as continuous by LightGBM.
       # Current categorical features: sector_code, day_of_week, halt_reason_code
5. Instantiate model via factory: model = create_model(config)
6. Train 5 classifiers:
       models = model.train(train_df, val_df, feature_names, categorical_cols,
                            sample_weight_col=sample_weight_col)
7. Evaluate on test set:
       eval_result = model.evaluate(models, test_df, feature_names)
8. Print per-class AUC results
9. Print top-10 features per classifier:
       model.feature_importance(models)
10. Save model artifacts:
       model.save(models, run_id, eval_result, feature_names, categorical_cols)
11. If run_reducer=True (selection phase — called by PipelineOptimizer only):
       X_train = train_df[feature_names]   # feature columns only
       X_val   = val_df[feature_names]
       train_labels = {
           label: train_df[f"label_{label}"]
           for label in ["up5", "up3", "sw", "dn3", "dn5"]
       }
       provider = create_importance_provider(
           model_type   = config["model"]["model_type"],
           models       = models,
           train_labels = train_labels,
           config       = config,
       )
       reducer = DimensionalityReducer(config)
       selected_features, reduction_report = reducer.run_all(X_train, X_val, provider)
       → selected_features included in returned result dict
       → reduction_report included in returned result dict
       → NO retraining here; PipelineOptimizer uses selected_features for voting only
12. Write to train_log (DuckDB):
       run_id, optimizer_run_id, run_at, phase, fold_idx, fold_train_end,
       feature_config, n_features, n_features_reduced,
       auc_*, auc_mean, auc_std=NULL, auc_reduced_mean=NULL,
       best_of_loop=False, notes=NULL
       Note: auc_std is NULL at write time — updated by PipelineOptimizer
             via UPDATE after all folds in a run/trial complete
             on the row with fold_run_ids[-1] (guaranteed MAX fold_idx)
```

---

## run_reducer vs --reduce: Critical Distinction

These two mechanisms trigger DimensionalityReducer but have different behaviors:

**`run_reducer=True` (called by PipelineOptimizer, selection phase):**
```
1. Run DimensionalityReducer on current fold's train/val
2. Return selected_features and reduction_report in result dict for frequency voting
3. NO retraining — PipelineOptimizer accumulates votes across folds
4. One model artifact saved (full feature model for this fold)
```

**`--reduce` CLI flag (standalone use only):**
```
1. Run DimensionalityReducer on train/val
2. Immediately retrain on selected_features
3. Save second model artifact with "_reduced" suffix
4. Report both baseline AUC and reduced AUC
```

The `--reduce` flag is for standalone experimentation.
PipelineOptimizer never uses it — it calls `run_reducer=True` instead.

---

## Class Interface

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
        fold_idx: int | None = None,
        fold_train_end: str | None = None,
        feature_config: dict | None = None,
        feature_names: list[str] | None = None,
        run_reducer: bool = False,
        phase: str | None = None,
    ) -> dict:
        """
        Train, evaluate, save artifacts, write to train_log.

        Returns dict containing:
            auc_mean:          float — mean test AUC across 5 classifiers
            auc_up5..auc_dn5:  float — per-classifier test AUC
            selected_features: list[str] | None — populated if run_reducer=True, else None
            reduction_report:  dict | None — populated if run_reducer=True, else None
                               includes importance_scores for PipelineOptimizer logging

        Args:
            train_df, val_df, test_df:
                Session-filtered, balanced splits from ClassBalancer.
                Must not contain p_entry or label columns in feature space.
            run_id:
                Generated via utils.generate_run_id() if not provided.
            fold_idx:
                0-based fold index for rolling folds.
                None for standalone CLI runs (single split).
            fold_train_end:
                Last date of train window ('YYYYMMDD').
                Obtained from fold_meta["fold_train_end"] in rolling mode.
                None for standalone CLI runs.
            feature_config:
                Active feature group config dict for train_log logging.
                None in standalone mode — full config snapshot used instead.
            feature_names:
                Explicit feature list (exploitation phase, selected_features).
                None = use FeatureExtractor.get_feature_names() (full set).
            run_reducer:
                True = run DimensionalityReducer, return selected_features
                       and reduction_report.
                       Used by PipelineOptimizer selection phase only.
                       Does NOT trigger retraining.
                False = skip reducer (default).
            phase:
                "selection" | "exploitation" | "full" | None (standalone).
                Recorded in train_log for diagnostics.
        """
        ...
```

---

## CLI Interface

```bash
python scripts/run_train.py [OPTIONS]

Options:
    --config, -c    Path to pipeline_config.yaml (default: configs/pipeline_config.yaml)
    --data-dir, -d  Directory containing train/val/test parquet files (default: data/processed)
    --run-id        Explicit run ID (default: YYYYMMDD_HHMMSS)
    --reduce        Run DimensionalityReducer AND retrain on selected features.
                    Saves a second "_reduced" model artifact.
                    Standalone use only — PipelineOptimizer does not use this flag.
    --features      Path to selected_features.json (load pre-selected feature list)
```

CLI always runs with `optimizer_run_id=None`, `fold_idx=None`, `fold_train_end=None` (standalone mode).

---

## train_log Write

```python
train_log_row = {
    "run_id":              run_id,
    "optimizer_run_id":    optimizer_run_id,
    "run_at":              run_at,
    "phase":               phase,             # "selection"|"exploitation"|"full"|None
    "fold_idx":            fold_idx,          # None for standalone; 0-based for rolling
    "fold_train_end":      fold_train_end,    # None for standalone
    "feature_config":      json.dumps(feature_config)
                           if feature_config is not None
                           else json.dumps(config),
    "n_features":          len(feature_names),
    "n_features_reduced":  len(selected_features) if run_reducer else None,
    "auc_up5":             eval_result["up5"]["test_auc"],
    "auc_up3":             eval_result["up3"]["test_auc"],
    "auc_sw":              eval_result["sw"]["test_auc"],
    "auc_dn3":             eval_result["dn3"]["test_auc"],
    "auc_dn5":             eval_result["dn5"]["test_auc"],
    "auc_mean":            eval_result["mean_test_auc"],
    "auc_std":             None,    # set by PipelineOptimizer via UPDATE
                                    # on fold_run_ids[-1] (MAX fold_idx)
                                    # after all folds in a run/trial complete
    "auc_reduced_mean":    None,    # set only when --reduce CLI flag used
    "best_of_loop":        False,   # set by PipelineOptimizer via UPDATE
    "notes":               None,
}
```

---

## Config Keys Used

```yaml
model.*
lgbm_params.*
dimensionality_reducer.*

trainer:
  use_ambiguous_sample_weight: false  # True = apply reduced weight to is_ambiguous=True samples
  ambiguous_sample_weight: 0.5        # weight value for ambiguous samples (0 < value < 1)
                                      # only used when use_ambiguous_sample_weight = True
                                      # ClassBalancer.ambiguous_sample_action must be "keep"
                                      # for weights to have effect (excluded samples are gone)
```

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
- `best_of_loop` updated by PipelineOptimizer via UPDATE, not by Trainer itself
- `auc_std` updated by PipelineOptimizer via UPDATE on fold_run_ids[-1] (MAX fold_idx), not by Trainer itself
- `fold_idx` and `fold_train_end` are None for standalone CLI runs
- `fold_train_end` sourced from fold_meta["fold_train_end"] in rolling mode —
  Trainer does not compute it independently
- Sample weight column (`__sample_weight__`) generated inside Trainer when
  `use_ambiguous_sample_weight=True`; must not appear in feature_names
- `categorical_cols` derived from `FeatureExtractor.get_feature_schema()` —
  no heuristic pattern matching; only "categorical" typed features registered
- `run_reducer=True` returns selected_features and reduction_report — no retraining within Trainer.run()
- `--reduce` CLI flag triggers full reduce-retrain cycle — saves `_reduced` artifact;
  this behavior is NOT replicated when run_reducer=True is called by PipelineOptimizer
- Session_mode filtering applied upstream by ClassBalancer — Trainer does not re-apply it
- `is_ambiguous` column used for weight assignment only —
  it is a metadata column and must never appear in feature_names
