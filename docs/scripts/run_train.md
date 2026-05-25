# Script: run_train.py

**File:** `scripts/run_train.py`

---

## Role

CLI entry point and callable class for model training.
Loads preprocessed data, trains LightGBM classifiers, optionally runs dimensionality reduction,
writes results to train_log, and saves model artifacts.

Called per fold by PipelineOptimizer (rolling walk-forward) or standalone via CLI.

---

## Class Interface

```python
class Trainer:
    def __init__(
        self,
        config: dict,
        db_conn: duckdb.DuckDBPyConnection | None,
        optimizer_run_id: str | None = None,
    ): ...

    def run(
        self,
        train_df:       pd.DataFrame,
        val_df:         pd.DataFrame,
        test_df:        pd.DataFrame,
        run_id:         str | None = None,
        fold_idx:       int = -1,
        outer_fold_idx: int = -1,
        fold_train_end: str | None = None,
        feature_config: dict | None = None,
        feature_names:  list[str] | None = None,
        run_reducer:    bool = False,
        phase:          str | None = None,
        trial_idx:      int = 0,
        dry_run:        bool = False,
    ) -> dict:
        """
        Train model, evaluate, optionally run reducer, write train_log, save artifact.

        Args:
            train_df, val_df, test_df:
                Preprocessed DataFrames (balanced train, val, test splits).
            run_id:
                Explicit run ID. If None, generated via utils.generate_run_id().
                In nested validation context, coordinator pre-generates run_id
                as "{optimizer_run_id}_o{outer_fold_idx}_t{trial_idx}_f{fold_idx}"
                and passes it here; generate_run_id() is NOT called inside workers.
            fold_idx:
                Rolling inner fold index (0-based). Default -1 for standalone/outer eval.
            outer_fold_idx:
                Outer fold index for nested validation (0-based). Default -1 for
                non-nested runs (standalone, selection, full, non-nested exploitation).
            fold_train_end:
                Last date of train window ('YYYYMMDD'). None for standalone.
                Sourced from fold_meta["fold_train_end"] — not computed independently.
            feature_config:
                Active hyperparameter config dict for this trial.
                None in standalone mode — full config snapshot used instead.
            feature_names:
                Explicit feature list (exploitation phase, selected_features).
                None = use FeatureExtractor.get_feature_names() (full set).
            run_reducer:
                True = run DimensionalityReducer, return selected_features
                       and reduction_report. Used by selection phase only.
                       Does NOT trigger retraining within Trainer.run().
                False = skip reducer (default).
            phase:
                "selection" | "exploitation" | "full" | None (standalone).
                Recorded in train_log for diagnostics.
            trial_idx:
                0-based trial counter within the inner trial loop.
                -1 for outer evaluation rows (not part of inner trial search).
                0 for standalone/selection/full (single implicit trial).
            dry_run:
                True = train and evaluate normally, but skip train_log INSERT
                       and model artifact save. Returns train_log_row in result dict.
                       Used by parallel workers in nested validation — coordinator
                       performs all DB writes after collecting worker results.
                False = write train_log INSERT and save artifact (default).
                db_conn may be None when dry_run=True.
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

CLI always runs with:
```
optimizer_run_id=None, fold_idx=-1, outer_fold_idx=-1,
fold_train_end=None, trial_idx=0, dry_run=False
```

---

## train_log Write

```python
train_log_row = {
    "run_id":              run_id,
    "optimizer_run_id":    optimizer_run_id,        # None for standalone
    "run_at":              run_at,
    "phase":               phase,                   # "selection"|"exploitation"|"full"|None
    "trial_idx":           trial_idx,               # -1 for outer eval; 0 for standalone;
                                                    # 0-based for exploitation inner trials
    "fold_idx":            fold_idx,                # -1 for standalone/outer eval
    "outer_fold_idx":      outer_fold_idx,          # -1 for non-nested runs
    "fold_train_end":      fold_train_end,          # None for standalone
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
                                    # on fold_run_ids[-1] after trial completes
                                    # pruned trials retain NULL
    "auc_reduced_mean":    None,    # set only when --reduce CLI flag used
    "best_of_loop":        False,   # set by PipelineOptimizer via UPDATE
    "is_pruned":           False,   # set TRUE by Coordinator via UPDATE for pruned trials
    "notes":               None,
}
```

When `dry_run=True`, this dict is included in the return value under `result["train_log_row"]`.
The coordinator performs the actual INSERT. When `dry_run=False`, Trainer performs INSERT directly.

---

## Config Keys Used

```yaml
model.*
lgbm_params.*
dimensionality_reducer.*

trainer:
  use_ambiguous_sample_weight: false
  ambiguous_sample_weight: 0.5
```

---

## Constraints

- All parameters from `pipeline_config.yaml` — nothing hardcoded
- Must not modify the preprocessed data files
- Must print summary results even if dimensionality reduction is run
- Model artifacts must include config snapshot for reproducibility
- Must handle empty DataFrames gracefully (exit with error message)
- `run_id` generated via `utils.generate_run_id()` only when run_id argument is None
- In nested validation, coordinator pre-generates run_id and passes it explicitly —
  workers must NOT call generate_run_id() internally
- `experiment_log` is NOT written by run_train.py — only train_log
- `best_of_loop` updated by PipelineOptimizer via UPDATE, not by Trainer itself
- `is_pruned` updated by PipelineOptimizer Coordinator via UPDATE, not by Trainer
- `auc_std` updated by PipelineOptimizer via UPDATE on `fold_run_ids[-1]`
- `fold_idx` default -1 for standalone (no rolling fold); 0-based for rolling folds
- `outer_fold_idx` default -1 for non-nested runs; 0-based for nested outer folds
- `trial_idx` default 0 for standalone/selection/full; -1 for outer eval rows;
  0-based for exploitation inner trials
- `fold_train_end` sourced from fold_meta["fold_train_end"] — Trainer does not compute it
- Sample weight column (`__sample_weight__`) generated inside Trainer when
  `use_ambiguous_sample_weight=True`; must not appear in feature_names
- `categorical_cols` derived from `FeatureExtractor.get_feature_schema()` — no heuristic matching
- `run_reducer=True` returns selected_features and reduction_report — no retraining within Trainer.run()
- `--reduce` CLI flag triggers full reduce-retrain cycle; NOT replicated when run_reducer=True
  is called by PipelineOptimizer
- Session_mode filtering applied upstream by ClassBalancer — Trainer does not re-apply it
- `is_ambiguous` column used for weight assignment only — never appears in feature_names
- `dry_run=True`: db_conn may be None; no INSERT, no artifact save; train_log_row in result
- `dry_run=False`: db_conn must be valid; performs INSERT and artifact save
