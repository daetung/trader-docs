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

**DB access role.** A per-chunk guest (db_schema.md's DB File Ownership
Windows; db_coordination.md's Unit scope), run through guest_supervisor.md's
`run_guest`:
- `run_guest run_train` enters `register('guest', 'run_train', trigger)` and
  hands the body a unit opener, each call opening one
  `guest_unit('run_train', None)`, which waits for admission without limit;
- the body opens one unit, for its train_log write (train_log Write,
  below); training runs outside units;
- the body calls neither `register()` nor `guest_unit()`;
- `scripts/run_train.py` run directly hands over to
  `run_guest run_train`;
- while an owner waits past `coordination.guest_max_hold_seconds`, the run
  ends by self-termination and is re-run from scratch.

```python
class Trainer:
    def __init__(
        self,
        config: dict,
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
        fold_train_start: str | None = None,
        fold_train_end: str | None = None,
        fold_train_days: int | None = None,
        feature_config: dict | None = None,
        feature_names:  list[str] | None = None,
        run_reducer:    bool = False,
        phase:          str | None = None,
        trial_idx:      int = 0,
        dry_run:        bool = False,
    ) -> dict:
        """
        Train model, evaluate, optionally run reducer, save artifact, and
        return train_log_row — never inserting it (train_log Write, below).

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
                non-nested runs (standalone, selection, full, non-nested exploitation)
                and for a final model (exploitation, refit).
            fold_train_start, fold_train_end:
                First and last date of the date range the model is actually
                trained on ('YYYYMMDD'). None for standalone.
            fold_train_days:
                Trading days that range holds. None for standalone.
                fold_train_start, fold_train_end and fold_train_days are
                passed by the caller — from fold_meta at the
                inner fold call sites, from the temporal_split_simple() train
                split for the outer eval and the final model. Trainer does not
                compute them.
            feature_config:
                Active hyperparameter config dict for this trial; for a
                selection-phase run, the run's stage descriptor
                (pipeline_optimizer.md).
                None in standalone mode — full config snapshot used instead.
            feature_names:
                Explicit feature list — the exploitation phase's
                selected_features; the selection phase's Stage A alternative
                feature sets, Stage B-1's input set (passed with
                run_reducer=True) and Stage B-2's voted sets
                (pipeline_optimizer.md).
                None = use FeatureExtractor.get_feature_names() (full set).
            run_reducer:
                True = run DimensionalityReducer, return selected_features
                       and reduction_report. Used by selection phase only.
                       Does NOT trigger retraining within Trainer.run().
                False = skip reducer (default).
            phase:
                "selection" | "exploitation" | "refit" | "full" | None (standalone).
                Recorded in train_log for diagnostics.
            trial_idx:
                0-based trial counter within the inner trial loop.
                -1 for outer evaluation rows (not part of inner trial search;
                exploitation or refit).
                -2 for a champion outer evaluation row (exploitation).
        0 for standalone/selection/full (single implicit trial), and also for
        the exploitation- or refit-phase final model — disambiguated from
        an exploitation inner trial 0 by outer_fold_idx: -1 for the final
        model, >=0 for an inner trial tied to a specific outer fold.
            dry_run:
                True = train and evaluate normally, but skip the model artifact
                       save. Used by parallel workers in nested validation.
                False = save the artifact (default).
                dry_run controls only the artifact save; train_log_row is
                always returned and the caller writes it.

        Returns:
            dict with keys:
                "auc_mean":          float   — mean test AUC across 5 classifiers
                "auc_std":           None    — populated by optimizer via UPDATE
                "reduction_report":  dict | None  — non-None when run_reducer=True
                "selected_features": list[str] | None  — non-None when run_reducer=True
                "train_log_row":     dict    — always present; the caller writes it
        """
        ...
```

---

## CLI Interface

```bash
run_guest run_train [OPTIONS]
# python scripts/run_train.py <args> hands over to run_guest run_train <args>

Options:
    --config, -c    Path to pipeline_config.yaml (default: configs/pipeline_config.yaml)
    --data-dir, -d  Directory containing train/val/test parquet files (default:
                    None, meaning data_paths.processed_dir)
    --run-id        Explicit run ID (default: YYYYMMDD_HHMMSS)
    --reduce        Run DimensionalityReducer AND retrain on selected features.
                    Saves a second "_reduced" model artifact only when the
                    reduced model's mean test AUC ≥ the full-feature model's
                    auc_mean − dimensionality_reducer.auc_tolerance;
                    auc_reduced_mean is recorded either way.
                    Standalone use only — PipelineOptimizer does not use this flag.
    --features      Path to selected_features.json (load pre-selected feature list)
```

CLI always runs with:
```
optimizer_run_id=None, fold_idx=-1, outer_fold_idx=-1,
fold_train_start=None, fold_train_end=None, fold_train_days=None,
trial_idx=0, dry_run=False
```

---

## train_log Write

```python
train_log_row = {
    "run_id":              run_id,
    "optimizer_run_id":    optimizer_run_id,        # None for standalone
    "run_at":              run_at,
    "phase":               phase,                   # "selection"|"exploitation"|"refit"|"full"|None
    "trial_idx":           trial_idx,               # -2 for a champion outer eval; -1 for
                                                    # outer eval; 0 for standalone/
                                                    # selection/full and for the exploitation
                                                    # or refit final model (outer_fold_idx=-1
                                                    # disambiguates it from inner trial 0);
                                                    # 0-based otherwise for exploitation
                                                    # inner trials
    "fold_idx":            fold_idx,                # -1 for standalone/outer eval
    "outer_fold_idx":      outer_fold_idx,          # -1 for non-nested runs
    "fold_train_start":    fold_train_start,        # None for standalone
    "fold_train_end":      fold_train_end,          # None for standalone
    "fold_train_days":     fold_train_days,         # None for standalone
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
                                    # on fold_run_ids[-1] after trial completes;
                                    # the selection phase: the std over folds
                                    # of the chosen candidate's auc_mean, set
                                    # on its run_id at the largest fold_idx —
                                    # with no eligible candidate, of the
                                    # baseline runs' auc_mean, on the baseline
                                    # run at the largest fold_idx
                                    # pruned trials retain NULL
    "auc_reduced_mean":    None,    # set only when --reduce CLI flag used,
                                    # whether or not the _reduced artifact is saved
    "best_of_loop":        False,   # set by PipelineOptimizer via UPDATE
    "is_pruned":           False,   # set TRUE by Coordinator via UPDATE for pruned trials
    "notes":               None,
}
```

Trainer.run() never inserts into train_log: this dict is returned under
`result["train_log_row"]`. run_train writes the row — its auc_reduced_mean
included under `--reduce` — in its write unit; the optimizer's coordinator
writes its rows in the step's write unit (pipeline_optimizer.md). An artifact
directory without a train_log row is an incomplete run.

---

## Config Keys Used

```yaml
model.*
lgbm_params.*
dimensionality_reducer.*
data_paths.processed_dir
drift.psi_bins

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
- Trainer saves `{run_id}_reference.json` with its artifacts, not under
  dry_run (base_model.md): per class probability, `drift.psi_bins` quantile
  bin edges of its val-split predictions and each bin's share; per feature of
  meta feature_names, its profile over the val split
- Must handle empty DataFrames gracefully (exit with error message)
- `run_id` generated via `utils.generate_run_id()` only when run_id argument is None
- In nested validation, coordinator pre-generates run_id and passes it explicitly —
  workers must NOT call generate_run_id() internally
- `experiment_log` is NOT written by run_train.py — only train_log
- `best_of_loop` updated by PipelineOptimizer via UPDATE, not by Trainer itself
- `is_pruned` updated by PipelineOptimizer Coordinator via UPDATE, not by Trainer
- `auc_std` updated by PipelineOptimizer via UPDATE on `fold_run_ids[-1]`; the
  selection phase sets it on the chosen candidate's run_id at the largest
  fold_idx (the baseline run's when no candidate is eligible)
- `fold_idx` default -1 for standalone (no rolling fold); 0-based for rolling folds
- `outer_fold_idx` default -1 for non-nested runs; 0-based for nested outer folds
- `trial_idx` default 0 for standalone/selection/full; -1 for outer eval rows;
  -2 for champion outer eval rows; 0-based for exploitation inner trials;
  also 0 for the exploitation or refit final model (outer_fold_idx=-1) — same
  "single implicit trial" semantics as standalone/selection/full, just
  reached via a different phase
- `fold_train_start`, `fold_train_end`, `fold_train_days` are passed by the caller —
  Trainer does not compute them
- Sample weight column (`__sample_weight__`) generated inside Trainer when
  `use_ambiguous_sample_weight=True`; must not appear in feature_names
- `categorical_cols` derived from `FeatureExtractor.get_feature_schema()` — no heuristic matching
- `run_reducer=True` returns selected_features and reduction_report — no retraining within Trainer.run()
- `--reduce` CLI flag triggers full reduce-retrain cycle; NOT replicated when run_reducer=True
  is called by PipelineOptimizer
- No session filter applies — entry points are regular-session only from `scan()` (data_boundary.md)
- `is_ambiguous` column used for weight assignment only — never appears in feature_names
- Trainer takes no db_conn and never inserts into train_log; `dry_run` controls
  only the artifact save; train_log_row is always in the result
- Calls neither `register()` nor `guest_unit()` — `run_guest` opens its units
- Built by PipelineOptimizer per step without a connection; the caller writes
  its train_log_row (pipeline_optimizer.md's Work)
