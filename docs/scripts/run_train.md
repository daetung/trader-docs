# Module: run_train.py

**File:** `scripts/run_train.py`

---

## Role

CLI entry point that orchestrates LightGBM training for the 5-class auto-scalping pipeline.
Loads preprocessed train/val/test parquet files, trains models, evaluates on test set,
and saves model artifacts.

---

## Input / Output

**Input:**
```python
# From preprocessed parquet files in data/processed/
train_df: pd.DataFrame   # balanced training split
val_df:   pd.DataFrame   # validation split
test_df:  pd.DataFrame   # test split
# All DataFrames contain: [features..., label_up5, label_up3, label_sw, label_dn3, label_dn5]
```

**Output:**
```python
# Printed to stdout:
#   - Per-class validation AUC (during training)
#   - Per-class test AUC + mean test AUC
#   - Feature importance top-10 per classifier
#   - Model save location

# Saved to models/lightgbm/{run_id}/ directory:
#   {run_id}_up5.lgb, {run_id}_up3.lgb, {run_id}_sw.lgb,
#   {run_id}_dn3.lgb, {run_id}_dn5.lgb
#   {run_id}_meta.json
```

---

## Training Pipeline Steps

```
1. Load config from pipeline_config.yaml
2. Load train/val/test parquet files from data/processed/
3. Build feature_names via FeatureExtractor.get_feature_names()
4. Identify categorical_cols: features ending with "_code" (e.g. ["sector_code"])
5. Instantiate model via factory: model = create_model(config)
6. Train 5 classifiers:
       models = model.train(train_df, val_df, feature_names, categorical_cols)
7. Evaluate on test set:
       eval_result = model.evaluate(models, test_df, feature_names)
8. Print per-class AUC results
9. Print top-10 features per classifier:
       model.feature_importance(models)
10. Save artifacts:
       model.save(models, run_id, eval_result, feature_names, categorical_cols)
11. Optionally run DimensionalityReducer if --reduce flag is set (see below)
```

---

## CLI Interface

```bash
python scripts/run_train.py [--config CONFIG] [--data-dir DIR] [--run-id ID] [--reduce]

Options:
    --config        Path to pipeline_config.yaml (default: configs/pipeline_config.yaml)
    --data-dir      Directory containing train/val/test parquet files (default: data/processed)
    --run-id        Explicit run ID (default: YYYYMMDD_HHMMSS timestamp)
    --reduce        Run DimensionalityReducer after initial training
```

---

## Feature Names and Categorical Columns

Feature names are obtained deterministically via `FeatureExtractor.get_feature_names()`.
Categorical columns are inferred from feature names: any feature ending with `"_code"`
is treated as categorical (e.g. `"sector_code"`). No hardcoding.

---

## Optional: Dimensionality Reduction

If `--reduce` flag is provided:

```
1. Train baseline model on all features (step 6 above)
2. Run DimensionalityReducer.run_all(X_train, X_val, models)
3. Retrain model on reduced feature set
4. Compare AUC: if within tolerance → save reduced model with suffix "_reduced"
5. Log both baseline and reduced results to stdout
```

Controlled exclusively by `--reduce` flag; not run by default.

---

## Constraints

- All parameters from `pipeline_config.yaml` — nothing hardcoded
- Must not modify the preprocessed data files
- Must print summary results even if dimensionality reduction is run
- Model artifacts must include config snapshot for reproducibility
- Must handle empty DataFrames gracefully (exit with error message)
- Variable names `model` (BaseModel instance) and `models` (trained booster dict) must not be mixed
