# Module: PipelineOptimizer

**File:** `src/optimization/optimizer.py`
**Depends on:** all pipeline modules

---

## Role

Training endpoint for the auto-scalping pipeline.
Manages the full preprocess → train → AUC evaluation → backtest cycle.
Searches feature combinations to find the pipeline configuration with the best winning rate.
All results logged to DuckDB.

Supports three operational phases with distinct objectives:
- **Selection phase**: identify stable feature subset via one rolling fold pass + DimensionalityReducer
- **Exploitation phase**: optimize model with confirmed feature subset via trial search loop
- **Full phase**: reducer disabled entirely; all features used as-is

---

## Architecture Position

```
Training endpoint:  PipelineOptimizer
    └── Preprocessor.run(return_data=True)   ← returns full_labeled_df (unsplit)
    └── ClassBalancer.generate_folds()        ← rolling walk-forward folds
    └── Trainer.run(...)                      ← per-fold train, writes train_log
    └── Backtester.run(...)                   ← writes experiment_log

Live inference endpoint: Inferencer (separate module)
    └── Preprocessor.run(live_mode=True)
    └── model.load() → resolve_signal()
```

PipelineOptimizer imports and calls `Preprocessor`, `Trainer`, `Backtester` classes directly.
It does not invoke run scripts as subprocesses.

---

## Operational Phases

### Phase: "selection"

**Objective:** Identify a stable feature subset from the full feature set
by running one complete pass through all rolling folds with DimensionalityReducer
enabled, then aggregating results via frequency voting.

```
- feature set: full (all feature groups enabled) — fixed
- DimensionalityReducer: enabled per fold
- Execution: one pass through all folds from generate_folds()
- Frequency voting: selected_features collected across all folds
  → features selected in >= vote_threshold fraction of folds → confirmed set
  → saved to configs/selected_features.json
  → per-fold selections and importance_scores saved to
    configs/feature_selection_log.json for audit
- train_log: one row per fold (fold_idx, fold_train_end populated)
- trial_idx: always 0 (single implicit trial, no search loop)
- No backtest — AUC and selected_features reported per fold only
- auc_std recorded on fold_run_ids[-1] row via UPDATE
```

```python
optimizer_run_id = utils.generate_run_id()

preprocessor = Preprocessor(config, db_conn)
full_labeled_df = preprocessor.run(return_data=True)

balancer = ClassBalancer(config)
trainer  = Trainer(config, db_conn, optimizer_run_id=optimizer_run_id)

feature_votes:          dict[str, int] = {}
feature_selection_log:  list[dict]     = []
fold_aucs:              list[float]    = []
fold_run_ids:           list[str]      = []
total_folds = 0

for train, val, test, fold_meta in balancer.generate_folds(
    full_labeled_df,
    balance=config["class_balancer"]["apply_balance"],
    session_mode=config["entry_detector"]["session_mode"],
):
    run_id = utils.generate_run_id()
    result = trainer.run(
        train, val, test,
        run_id=run_id,
        fold_idx=fold_meta["fold_idx"],
        fold_train_end=fold_meta["fold_train_end"],
        feature_config=None,
        feature_names=None,
        run_reducer=True,
        phase="selection",
        trial_idx=0,
    )
    fold_run_ids.append(run_id)
    for feature in result["selected_features"]:
        feature_votes[feature] = feature_votes.get(feature, 0) + 1
    feature_selection_log.append({
        "fold_idx":          fold_meta["fold_idx"],
        "fold_train_end":    fold_meta["fold_train_end"],
        "selected_features": result["selected_features"],
        "importance_scores": result["reduction_report"]["importance_scores"].to_dict(),
        "auc_mean":          result["auc_mean"],
    })
    fold_aucs.append(result["auc_mean"])
    total_folds += 1

vote_threshold    = config["optimizer"]["selection"]["vote_threshold"]
selected_features = sorted([
    f for f, count in feature_votes.items()
    if count / total_folds >= vote_threshold
])

# auc_std UPDATE on last fold row (fold_run_ids[-1] = MAX fold_idx)
auc_std = std(fold_aucs)
UPDATE train_log SET auc_std = auc_std WHERE run_id = fold_run_ids[-1]

save_json(selected_features,     config["optimizer"]["selected_features_path"])
save_json(feature_selection_log, config["optimizer"]["feature_selection_log_path"])
```

### Phase: "exploitation"

**Objective:** Maximize winning rate using the confirmed feature subset
by searching over feature_config combinations via a trial loop.
Each trial runs all rolling folds with the fixed selected feature set.

```
- feature set: loaded from configs/selected_features.json — fixed per run
- DimensionalityReducer: disabled (run_reducer=False)
- Execution: trial loop over search_space; each trial runs all folds
- trial_idx: 0-based per optimizer_run_id; written to train_log per fold
- auc_std stored on fold_run_ids[-1] row after each trial completes
- Backtest: executed for best trial using last fold's model and test set
- train_log: one row per fold per trial
```

```python
optimizer_run_id = utils.generate_run_id()

preprocessor    = Preprocessor(config, db_conn)
full_labeled_df = preprocessor.run(return_data=True)

selected_features = load_json(config["optimizer"]["selected_features_path"])

balancer = ClassBalancer(config)

best_fold_run_ids    = []
best_trial_auc_mean  = 0.0
best_feature_config  = None
best_config_override = None
best_train           = None
best_val             = None
best_fold_test       = None

for trial_idx, feature_config in enumerate(search_space):

    config_override = utils.apply_overrides(base_config, feature_config)
    trainer = Trainer(config_override, db_conn, optimizer_run_id=optimizer_run_id)

    fold_run_ids = []
    fold_aucs    = []

    for train, val, test, fold_meta in balancer.generate_folds(
        full_labeled_df,
        balance=config_override["class_balancer"]["apply_balance"],
        session_mode=config_override["entry_detector"]["session_mode"],
    ):
        run_id = utils.generate_run_id()
        result = trainer.run(
            train, val, test,
            run_id=run_id,
            fold_idx=fold_meta["fold_idx"],
            fold_train_end=fold_meta["fold_train_end"],
            feature_config=feature_config,
            feature_names=selected_features,
            run_reducer=False,
            phase="exploitation",
            trial_idx=trial_idx,
        )
        fold_run_ids.append(run_id)
        fold_aucs.append(result["auc_mean"])

    trial_auc_mean = mean(fold_aucs)
    trial_auc_std  = std(fold_aucs)

    # UPDATE auc_std on last fold row (fold_run_ids[-1] = MAX fold_idx)
    UPDATE train_log SET auc_std = trial_auc_std WHERE run_id = fold_run_ids[-1]

    if trial_auc_mean > best_trial_auc_mean:
        best_trial_auc_mean  = trial_auc_mean
        best_fold_run_ids    = fold_run_ids
        best_feature_config  = feature_config
        best_config_override = config_override
        best_train           = train    # last fold's train split of this trial
        best_val             = val      # last fold's val split of this trial
        best_fold_test       = test     # last fold's test split of this trial

    if trial_auc_mean >= auc_threshold:
        break

# Mark best trial's last fold row
UPDATE train_log SET best_of_loop = TRUE WHERE run_id = best_fold_run_ids[-1]

best_run_id = best_fold_run_ids[-1]

backtester = Backtester(best_config_override, db_conn, optimizer_run_id=optimizer_run_id)
summary    = backtester.run(best_fold_test, run_id=best_run_id)

# Save preprocessed data for best trial's last fold
preprocessor.save(best_train, best_val, best_fold_test, run_id=best_run_id)
```

### Phase: "full"

**Objective:** Run pipeline with full feature set and no dimensionality reduction.
Intended for baseline measurement and debugging before selection phase.

```
- feature set: full (all feature groups enabled) — fixed
- DimensionalityReducer: disabled (run_reducer=False)
- Execution: one pass through all folds (no trial loop)
- trial_idx: always 0 (single implicit trial)
- Backtest: executed after fold pass completes
- train_log: one row per fold
- auc_std recorded on last fold row
```

**Phase transition is always manual** — user edits `optimizer.phase` in config.

---

## Trial Loop Termination Conditions (Exploitation Phase)

```
Loop exits when any of the following:
1. trial_auc_mean >= auc_threshold   → proceed to backtest immediately
2. max_trials exceeded               → use best trial for backtest
3. Search space fully exhausted      → use best trial for backtest
```

---

## auc_std Recording Rule

```
After all folds complete for a given run or trial:
1. Compute auc_std = std(fold_aucs)
2. UPDATE train_log SET auc_std = <value> WHERE run_id = fold_run_ids[-1]
   # fold_run_ids[-1] is guaranteed to be MAX(fold_idx) because
   # generate_folds() always yields in ascending fold_idx order

All other fold rows retain auc_std = NULL.

Interpretation:
  Selection/Full: temporal stability of the full feature model
  Exploitation:   temporal stability of a specific feature_config trial
```

---

## optimizer_run_id Usage

- Generated once per `PipelineOptimizer.run()` call via `utils.generate_run_id()`
- Passed to all `Trainer` and `Backtester` instances within the same run
- Stored in `train_log.optimizer_run_id` and `experiment_log.optimizer_run_id`
- Enables grouping of all folds belonging to the same optimizer execution

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
        Execute phase-appropriate cycle:
          - "selection":    one fold pass, frequency voting, save selected_features
                            returns empty DataFrame (no backtest)
          - "exploitation": trial search loop with rolling folds, backtest best trial
                            returns experiment_log rows sorted by winning_rate desc
          - "full":         one fold pass, no reducer, backtest
                            returns experiment_log rows
        """
        ...

    def run_single(
        self,
        feature_config: dict,
        run_id: str | None = None,
    ) -> dict:
        """
        Run one complete fold pass for a given feature_config.
        Executes all rolling folds.
        Returns summary dict (also written to experiment_log if backtest runs).
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

`session_mode` override applied via dot-notation:
```python
{"entry_detector.session_mode": "pre"}
```

---

## Search Space Definition

```yaml
optimizer:
  phase: "selection"
  auc_threshold: 0.65
  max_trials: 50
  random_state: 42
  min_features: 5
  strategy: "grid"              # "grid" | "random"

  selected_features_path:       "configs/selected_features.json"
  feature_selection_log_path:   "configs/feature_selection_log.json"

  selection:
    vote_threshold: 0.70

  search_space:
    session_mode:
      options: ["regular", "pre", "combined"]

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
        options: ["uniform", "sample_weighted"]
```

---

## Scope of Automation

```
AUTOMATED (this module):
    Feature group ON/OFF combinations (exploitation phase trial loop only)
    Vectorizer method variants per indicator group
    importance_averaging strategy ("uniform" | "sample_weighted")
    session_mode ("regular" | "pre" | "combined")

MANUAL (edit pipeline_config.yaml directly):
    Individual indicator parameters
    Entry detection thresholds
    LightGBM hyperparameters
    Backtest settings (including sell_rate_tp, sell_rate_sl)
    Class balancer ratios
    Rolling fold parameters
    Phase transition (selection → exploitation → full)
```

---

## Constraints

- Each fold independently reproducible given feature_config and random_state
- All results written to DuckDB immediately after each fold — crash-safe
- Optimizer does NOT modify `pipeline_config.yaml`
- `train_log` written by Trainer per fold; `experiment_log` written by Backtester
- `best_of_loop` set TRUE on the last fold row of the best trial via UPDATE
- `auc_std` set on `fold_run_ids[-1]` via UPDATE — all other fold rows retain NULL
- `feature_selection_log.json` includes per-fold `importance_scores` from `reduction_report`
- Preprocessed parquet saved only for best trial's last fold via `preprocessor.save()`
- `best_train`, `best_val`, `best_fold_test` captured at best-trial update time;
  not sourced from loop-end variables (which may belong to a different trial)
- `optimizer_run_id` format: `YYYYMMDD_HHMMSS` (generated once per optimizer run)
- `"random"` strategy uses `random_state` for reproducible non-replacement sampling
- Phase transition is manual — no automatic switching between phases
- Selection phase does not run backtest — one fold pass only
- Full phase runs backtest after fold pass completes
- `session_mode` is a first-class search variable — results across session_modes
  are not cross-comparable by AUC
- `trial_idx` is 0-based per `optimizer_run_id` in exploitation;
  always 0 for selection/full/standalone — written to train_log per fold row
