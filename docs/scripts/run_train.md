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

# Saved to models/ directory:
#   models/YYYYMMDD_HHMMSS/{run_id}_{label}.lgb  (5 files)
#   models/YYYYMMDD_HHMMSS/{run_id}_meta.json
```

---

## Training Pipeline Steps

```
1. Load config from pipeline_config.yaml
2. Load train/val/test parquet files from data/processed/
3. Build feature names from FeatureExtractor.get_feature_names()
4. Identify categorical columns (sector_code)
5. Select model via config (`model.model_type` key) → instantiate via factory `create_model(config)`
6. Train 5 classifiers: `model.train(dataset)` where `dataset` wraps train/val splits
7. Evaluate on test set: eval_result = pipeline.evaluate(models, test, feature_names)
8. Print per-class AUC results
9. Print top-10 features per classifier
10. Save models: pipeline.save(models, run_id, eval_result, feature_names, categorical_cols)
11. Optionally run DimensionalityReducer if enabled in config
```

---

## CLI Interface

```bash
python scripts/run_train.py [--config CONFIG] [--data-dir DIR] [--run-id ID]

Options:
    --config        Path to pipeline_config.yaml (default: configs/pipeline_config.yaml)
    --data-dir      Directory containing train/val/test parquet files (default: data/processed)
    --run-id        Explicit run ID (default: YYYYMMDD_HHMMSS timestamp)
```

---

## Feature Names and Categorical Columns

Feature names are obtained deterministically via `FeatureExtractor.get_feature_names()`.
Categorical columns are determined from the config — specifically `sector_code` declared
in the feature groups section. If the config does not explicitly list categorical columns,
the script should infer `["sector_code"]` from the feature names (features ending with
"_code" are treated as categorical).

---

## Optional: Dimensionality Reduction

If `dimensionality_reducer` config is present and enabled, after initial training:

```
1. Train baseline on all features
2. Run DimensionalityReducer.run_all(X_train, X_valid, best_model)
3. Retrain on selected features
4. Compare AUC: if within tolerance -> save reduced model
5. Log both results
```

This is optional and controlled by a `--reduce` flag.

---

## Constraints

- All parameters from `pipeline_config.yaml` — nothing hardcoded
- Must not modify the preprocessed data files
- Must print summary results even if dimensionality reduction is run
- Model artifacts must include config snapshot for reproducibility
- Must handle empty DataFrames gracefully (exit with error message)
