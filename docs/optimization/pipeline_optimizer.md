# Module: PipelineOptimizer

**File:** `src/optimization/optimizer.py`
**Depends on:** all pipeline modules

---

## Role

Training endpoint for the auto-scalping pipeline.
Manages the full preprocess → train → AUC evaluation → backtest cycle.
Supports nested validation for unbiased hyperparameter selection,
Successive Halving for efficient trial pruning, and volatility-based
regime holdout for robustness evaluation.
All results logged to DuckDB.

Supports three operational phases with distinct objectives:
- **Selection phase**: identify stable feature subset via one rolling fold pass + DimensionalityReducer
- **Exploitation phase**: optimize hyperparameters via nested validation with Successive Halving
- **Full phase**: reducer disabled entirely; all features used as-is

---

## Architecture Position

```
Training endpoint:  PipelineOptimizer
    └── Preprocessor.run(return_data=True)    ← returns full_labeled_df (unsplit), called once
    └── utils.compute_vol_regime_holdout()    ← regime holdout dates
    └── ClassBalancer.generate_folds()        ← outer and inner rolling folds
    └── Trainer.run(...)                      ← per-fold train (dry_run in workers)
    └── Coordinator DB flush                  ← sequential train_log INSERT
    └── Backtester.run(...)                   ← writes experiment_log

Live inference endpoint: Inferencer (separate module)
    └── EntryPointDetector.detect()
    └── FeatureExtractor.extract()            ← direct call (no Preprocessor)
    └── model.load() → resolve_signal()
```

PipelineOptimizer imports and calls `Preprocessor`, `Trainer`, `Backtester` directly.
It does not invoke run scripts as subprocesses.

---

## Operational Phases

### Phase: "selection"

**Objective:** Identify a stable feature subset from the full feature set
by running one complete pass through all rolling folds with DimensionalityReducer
enabled, then aggregating results via frequency voting.

```
- feature set: full (all feature groups enabled)
- DimensionalityReducer: enabled (run_reducer=True)
- Execution: one pass through all inner rolling folds (no trial loop)
- trial_idx: always 0 (single implicit trial)
- fold_idx: 0-based inner fold index; outer_fold_idx: -1 (non-nested)
- No backtest — returns empty DataFrame
- train_log: one row per fold
- auc_std recorded on fold_run_ids[-1] after all folds complete
```

```python
optimizer_run_id = utils.generate_run_id()

preprocessor    = Preprocessor(config, db_conn)
full_labeled_df = preprocessor.run(return_data=True)

balancer = ClassBalancer(config)
trainer  = Trainer(config, db_conn, optimizer_run_id=optimizer_run_id)

inner_cfg = config["class_balancer"]["inner_fold"]

fold_run_ids      = []
fold_aucs         = []
reduction_reports = []

for train, val, test, fold_meta in balancer.generate_folds(
    full_labeled_df,
    balance=config["class_balancer"]["apply_balance"],
    session_mode=config["entry_detector"]["session_mode"],
    window_weeks=inner_cfg["window_weeks"],
    val_weeks=inner_cfg["val_weeks"],
    test_weeks=inner_cfg["test_weeks"],
    step_weeks=inner_cfg["step_weeks"],
    embargo_days=inner_cfg["embargo_days"],
    max_folds=None,          # override config cap — selection phase uses all available folds
):
    fold_idx = fold_meta["fold_idx"]
    run_id   = f"{optimizer_run_id}_f{fold_idx}"   # structured ID — no collision risk
    result   = trainer.run(
        train, val, test,
        run_id=run_id,
        fold_idx=fold_idx,
        outer_fold_idx=-1,
        fold_train_end=fold_meta["fold_train_end"],
        run_reducer=True,
        phase="selection",
        trial_idx=0,
    )
    fold_run_ids.append(run_id)
    fold_aucs.append(result["auc_mean"])
    reduction_reports.append(result["reduction_report"])

auc_std = std(fold_aucs) if len(fold_aucs) > 1 else 0.0
db_conn.execute(
    "UPDATE train_log SET auc_std = ? WHERE run_id = ?",
    [auc_std, fold_run_ids[-1]]
)

# Frequency voting across folds
selected_features = frequency_vote(reduction_reports,
    threshold=config["dimensionality_reducer"]["vote_threshold"])
save_json(selected_features, config["optimizer"]["selected_features_path"])
save_json({"fold_reports": reduction_reports},
          "configs/feature_selection_log.json")
```

**Post-selection follow-up (manual, not automated by this phase):** review
`selected_features` for any of the 9 tick-derived indicators (see
02_indicator_calculator.md's Tick-Derived Indicator Scale Sensitivity
registry). Indicators that survived selection should have their
`precalculate_bars` config flipped from the default `0` to `"lookback"` for
live deployment — see the rationale note alongside that config block. This
step is deliberately not triggered automatically by `frequency_vote()`
completing; it is a config decision made once selected_features stabilizes,
not a per-run action.

---

### Phase: "exploitation"

**Objective:** Search over LightGBM hyperparameters to find the best configuration
using nested walk-forward validation (outer fold × inner fold × Successive Halving).
Outer fold evaluation is unbiased — never seen during inner trial search.

#### Data Preparation

```python
optimizer_run_id = utils.generate_run_id()

preprocessor    = Preprocessor(config, db_conn)
full_labeled_df = preprocessor.run(return_data=True)

selected_features = load_json(config["optimizer"]["selected_features_path"])

# Regime holdout — remove high-volatility dates before any fold generation
holdout_cfg   = config["optimizer"]["regime_holdout"]
holdout_dates = utils.compute_vol_regime_holdout(
    db_conn,
    vol_percentile=holdout_cfg["vol_holdout_percentile"],
    window_days=holdout_cfg["vol_window_days"],
    vol_metric=holdout_cfg["vol_metric"],
) if holdout_cfg["enabled"] else set()

remaining_df = full_labeled_df[~full_labeled_df["date"].isin(holdout_dates)]
holdout_df   = full_labeled_df[full_labeled_df["date"].isin(holdout_dates)]

balancer = ClassBalancer(config)
```

#### Hyperparameter Search Space

```yaml
optimizer:
  hyperparameter_search:
    max_trials: 30
    strategy: "random"
    random_state: 42
    search_space:
      lgbm_params:
        num_leaves:       [31, 63, 127]
        min_data_in_leaf: [50, 100, 200]
        feature_fraction: [0.5, 0.7, 0.9]
        lambda_l1:        [0.0, 0.1, 1.0]
      entry_detector:
        session_mode:     ["regular", "pre", "combined"]
```

#### Nested Validation — Outer Fold Loop

```python
outer_cfg  = config["class_balancer"]["outer_fold"]
inner_cfg  = config["class_balancer"]["inner_fold"]
eta        = config["optimizer"]["successive_halving"]["eta"]
max_trials = config["optimizer"]["hyperparameter_search"]["max_trials"]
n_parallel = config["optimizer"]["parallelism"]["n_parallel_trials"]

# Physical cores per worker (avoid HyperThreading cache contention)
import psutil
effective_num_threads = max(
    1,
    psutil.cpu_count(logical=False) // n_parallel
)

outer_best_configs   = []
outer_scores         = []

for outer_train_df, _, outer_test_df, outer_fold_meta in balancer.generate_folds(
    remaining_df,
    balance=False,
    session_mode=None,
    window_weeks=outer_cfg["window_weeks"],
    val_weeks=0,
    test_weeks=outer_cfg["test_weeks"],
    step_weeks=outer_cfg["step_weeks"],
    embargo_days=outer_cfg["embargo_days"],
):
    outer_fold_idx = outer_fold_meta["fold_idx"]

    best_config    = None
    best_inner_auc = 0.0

    # --- Successive Halving across inner trial loop ---
    search_space   = sample_hyperparams(config, max_trials)
    active_trials  = list(enumerate(search_space))
    trial_auc_history: dict[int, list[float]] = {i: [] for i in range(len(active_trials))}
    max_inner_folds = inner_cfg.get("max_folds", 4)

    for round_idx in range(max_inner_folds):

        # Coordinator pre-generates run_ids (avoid collision in parallel workers)
        round_run_ids = {
            trial_idx: f"{optimizer_run_id}_o{outer_fold_idx}_t{trial_idx}_f{round_idx}"
            for trial_idx, _ in active_trials
        }

        # Dispatch parallel workers
        round_results = _run_round_parallel(
            active_trials=active_trials,
            round_idx=round_idx,
            outer_train_df=outer_train_df,
            outer_fold_idx=outer_fold_idx,
            round_run_ids=round_run_ids,
            selected_features=selected_features,
            config=config,
            effective_num_threads=effective_num_threads,
            optimizer_run_id=optimizer_run_id,
            db_path=config["data_paths"]["db_path"],
        )

        # Coordinator: sequential DB flush (single write connection)
        with duckdb.connect(config["data_paths"]["db_path"]) as write_conn:
            for r in round_results:
                write_conn.execute("INSERT INTO train_log ...", r["train_log_row"])

        # Update AUC history
        for r in round_results:
            trial_auc_history[r["trial_idx"]].append(r["auc_mean"])

        # Successive Halving pruning (not on final round)
        if round_idx < max_inner_folds - 1 and len(active_trials) > 1:
            trial_means = {
                idx: mean(trial_auc_history[idx])
                for idx, _ in active_trials
                if trial_auc_history[idx]
            }
            sorted_trials  = sorted(trial_means.items(), key=lambda x: x[1], reverse=True)
            n_survive      = max(1, len(sorted_trials) // eta)
            surviving_idx  = {idx for idx, _ in sorted_trials[:n_survive]}
            pruned_idx     = {idx for idx, _ in sorted_trials[n_survive:]}

            # Mark pruned trial rows
            with duckdb.connect(config["data_paths"]["db_path"]) as write_conn:
                for r in round_results:
                    if r["trial_idx"] in pruned_idx:
                        write_conn.execute(
                            "UPDATE train_log SET is_pruned = TRUE WHERE run_id = ?",
                            [r["run_id"]]
                        )

            active_trials = [(idx, hp) for idx, hp in active_trials if idx in surviving_idx]

    # Final round: set auc_std and best_of_loop for completed trials
    completed_trial_means = {
        idx: mean(trial_auc_history[idx])
        for idx, _ in active_trials
    }
    best_trial_idx = max(completed_trial_means, key=completed_trial_means.get)
    best_config    = search_space[best_trial_idx]

    with duckdb.connect(config["data_paths"]["db_path"]) as write_conn:
        for idx, _ in active_trials:
            last_run_id = round_run_ids[idx]   # last completed round run_id
            aucs        = trial_auc_history[idx]
            write_conn.execute(
                "UPDATE train_log SET auc_std = ? WHERE run_id = ?",
                [std(aucs) if len(aucs) > 1 else 0.0, last_run_id]
            )
        write_conn.execute(
            "UPDATE train_log SET best_of_loop = TRUE WHERE run_id = ?",
            [round_run_ids[best_trial_idx]]
        )

    # --- Outer eval: train best_config on outer_train, backtest on outer_test ---
    outer_train_split, outer_val_split = utils.temporal_split_simple(
        outer_train_df,
        session_mode=best_config.get("entry_detector.session_mode"),
        val_fraction=0.15,
        embargo_days=outer_cfg["embargo_days"],
    )

    outer_run_id    = f"{optimizer_run_id}_o{outer_fold_idx}_eval"
    config_override = utils.apply_overrides(config, best_config)
    config_override["lgbm_params"]["num_threads"] = effective_num_threads

    outer_trainer = Trainer(config_override, db_conn, optimizer_run_id=optimizer_run_id)
    outer_trainer.run(
        outer_train_split, outer_val_split, outer_test_df,
        run_id=outer_run_id,
        fold_idx=-1,
        outer_fold_idx=outer_fold_idx,
        fold_train_end=outer_fold_meta["fold_train_end"],
        feature_config=best_config,
        feature_names=selected_features,
        run_reducer=False,
        phase="exploitation",
        trial_idx=-1,          # outer eval row — not part of inner trial loop
        dry_run=False,
    )

    backtester = Backtester(config_override, db_conn, optimizer_run_id=optimizer_run_id)
    outer_summary = backtester.run(
        outer_test_df,
        run_id=outer_run_id,
        fold_idx=-1,
        outer_fold_idx=outer_fold_idx,
        fold_test_start=outer_fold_meta["fold_test_start"],
        fold_test_end=outer_fold_meta["fold_test_end"],
        eval_type="outer_validation",
    )

    # --- Execution-variant evaluation (optimizer.execution_eval) ---
    # Same model, same fold, same data — only execution keys differ, so the
    # comparison is statistically PAIRED. Runs here and not in the inner
    # loop because execution parameters change neither the training set nor
    # the model: AUC is identical across their settings, and Successive
    # Halving prunes on AUC, so a flag toggled there would be pruned at
    # random. Cost is per outer fold, not per trial, and a backtest is far
    # cheaper than a training run.
    if config["optimizer"]["execution_eval"]["enabled"]:
        # PASS 1 IS THE BASELINE AND MUST RUN FIRST — theta is the q-th
        # quantile of E[r|0] over the on-time entries this pass ACCEPTS, so
        # any grid combination with late_entry_enabled: true is
        # underivable until it has completed. See execution_common.md's
        # Late-Entry Residual Edge.
        import itertools, json
        grid = config["optimizer"]["execution_eval"]["grid"]
        combos = [
            dict(zip(grid.keys(), values))
            for values in itertools.product(*grid.values())
        ]
        # Baseline first — see the comment above on theta's derivation.
        combos.sort(key=lambda c: c.get("late_entry_enabled", False))
        combos = combos[: config["optimizer"]["execution_eval"]["max_combinations"]]

        for i, variant in enumerate(combos):
            variant_override = utils.apply_overrides(config_override, variant)
            variant_backtester = Backtester(
                variant_override, db_conn, optimizer_run_id=optimizer_run_id
            )
            variant_backtester.run(
                outer_test_df,
                run_id=f"{outer_run_id}_x{i}",
                fold_idx=-1,
                outer_fold_idx=outer_fold_idx,
                fold_test_start=outer_fold_meta["fold_test_start"],
                fold_test_end=outer_fold_meta["fold_test_end"],
                eval_type="outer_validation",
                execution_variant=json.dumps(variant),
            )

    outer_best_configs.append(best_config)
    outer_scores.append(outer_summary["winning_rate"])

# --- Consensus config → final model ---
consensus_config = utils.compute_consensus_config(
    outer_best_configs,
    config["optimizer"]["hyperparameter_search"]["search_space"],
)
final_config_override = utils.apply_overrides(config, consensus_config)

final_train_split, final_val_split = utils.temporal_split_simple(
    remaining_df,
    session_mode=consensus_config.get("entry_detector.session_mode"),
    val_fraction=0.15,
    embargo_days=outer_cfg["embargo_days"],
)
final_run_id = utils.generate_run_id()
final_trainer = Trainer(final_config_override, db_conn, optimizer_run_id=optimizer_run_id)

# test_df: holdout_df (completely unseen, preferred) when holdout is enabled and non-empty;
#          final_val_split as fallback when holdout is disabled (val is next-best available).
#          Note: val_split is used by LightGBM early stopping but does not overlap with train.
final_test_df = (
    holdout_df
    if holdout_cfg["enabled"] and len(holdout_df) > 0
    else final_val_split
)

final_trainer.run(
    final_train_split, final_val_split, final_test_df,
    run_id=final_run_id,
    fold_idx=-1,
    outer_fold_idx=-1,
    feature_config=consensus_config,
    feature_names=selected_features,
    run_reducer=False,
    phase="exploitation",
    trial_idx=0,
    dry_run=False,
)

# --- Regime holdout robustness check ---
if holdout_cfg["enabled"] and len(holdout_df) > 0:
    regime_backtester = Backtester(
        final_config_override, db_conn, optimizer_run_id=optimizer_run_id
    )
    regime_backtester.run(
        holdout_df,
        run_id=final_run_id,
        fold_idx=-1,
        outer_fold_idx=-1,
        eval_type="regime_holdout",
    )
```

---

### Phase: "full"

**Objective:** Run pipeline with full feature set and no dimensionality reduction.
Intended for baseline measurement and debugging.

```
- feature set: full (all feature groups enabled)
- DimensionalityReducer: disabled (run_reducer=False)
- Execution: one pass through all inner rolling folds (no trial loop)
- trial_idx: always 0; fold_idx: 0-based; outer_fold_idx: -1
- Backtest: executed after fold pass completes
- auc_std recorded on fold_run_ids[-1]
```

```python
# (full phase fold loop — same structure as selection, with run_reducer=False and max_folds=None)
for train, val, test, fold_meta in balancer.generate_folds(
    full_labeled_df,
    balance=config["class_balancer"]["apply_balance"],
    session_mode=config["entry_detector"]["session_mode"],
    window_weeks=inner_cfg["window_weeks"],
    val_weeks=inner_cfg["val_weeks"],
    test_weeks=inner_cfg["test_weeks"],
    step_weeks=inner_cfg["step_weeks"],
    embargo_days=inner_cfg["embargo_days"],
    max_folds=None,          # override config cap — full phase uses all available folds
):
    fold_idx = fold_meta["fold_idx"]
    run_id   = f"{optimizer_run_id}_f{fold_idx}"   # structured ID — no collision risk
    result   = trainer.run(
        train, val, test,
        run_id=run_id,
        fold_idx=fold_idx,
        outer_fold_idx=-1,
        fold_train_end=fold_meta["fold_train_end"],
        run_reducer=False,
        phase="full",
        trial_idx=0,
    )
    fold_run_ids.append(run_id)
    fold_aucs.append(result["auc_mean"])

auc_std = std(fold_aucs) if len(fold_aucs) > 1 else 0.0
db_conn.execute(
    "UPDATE train_log SET auc_std = ? WHERE run_id = ?",
    [auc_std, fold_run_ids[-1]]
)

# Backtest on last fold's test split (baseline measurement)
backtester = Backtester(config, db_conn, optimizer_run_id=optimizer_run_id)
backtester.run(
    test,                    # last fold's test split (loop variable after exit)
    run_id=fold_run_ids[-1],
    fold_idx=-1,
    outer_fold_idx=-1,
    eval_type=None,
)
```

---

## Multiprocessing Design

### Worker Function

```python
def run_trial_round(
    trial_idx:        int,
    hyperparams:      dict,
    outer_train_df:   pd.DataFrame,  # Linux/WSL: CoW shared via fork
    outer_train_path: str | None,    # Windows spawn: parquet path instead
    inner_fold_idx:   int,
    run_id:           str,           # pre-generated by coordinator
    selected_features: list[str],
    config:           dict,
    effective_num_threads: int,
    optimizer_run_id: str,
    outer_fold_idx:   int,
    db_path:          str,
) -> dict | None:
    """
    Worker function. Runs one (trial, inner_fold) pair.
    Opens its own DuckDB read-only connection (no write).
    Returns result dict — train_log INSERT deferred to coordinator.
    db_conn=None passed to Trainer (dry_run=True).
    """
    import duckdb, sys
    db_conn = duckdb.connect(db_path, read_only=True)

    # Windows spawn: load outer_train_df from parquet
    if sys.platform == "win32" and outer_train_path is not None:
        outer_train_df = pd.read_parquet(outer_train_path)

    config_override = utils.apply_overrides(config, hyperparams)
    config_override["lgbm_params"]["num_threads"] = effective_num_threads

    inner_cfg = config["class_balancer"]["inner_fold"]
    inner_balancer = ClassBalancer(config_override)
    folds = list(inner_balancer.generate_folds(
        outer_train_df,
        balance=config_override["class_balancer"]["apply_balance"],
        session_mode=config_override["entry_detector"].get("session_mode"),
        window_weeks=inner_cfg["window_weeks"],
        val_weeks=inner_cfg["val_weeks"],
        test_weeks=inner_cfg["test_weeks"],
        step_weeks=inner_cfg["step_weeks"],
        embargo_days=inner_cfg["embargo_days"],
        max_folds=inner_cfg.get("max_folds"),
    ))

    if inner_fold_idx >= len(folds):
        return None

    train, val, test, fold_meta = folds[inner_fold_idx]

    trainer = Trainer(config_override, db_conn=None, optimizer_run_id=optimizer_run_id)
    result  = trainer.run(
        train, val, test,
        run_id=run_id,
        fold_idx=inner_fold_idx,
        outer_fold_idx=outer_fold_idx,
        fold_train_end=fold_meta["fold_train_end"],
        feature_config=hyperparams,
        feature_names=selected_features,
        run_reducer=False,
        phase="exploitation",
        trial_idx=trial_idx,
        dry_run=True,
    )

    db_conn.close()
    return {
        "trial_idx":     trial_idx,
        "run_id":        run_id,
        "auc_mean":      result["auc_mean"],
        "train_log_row": result["train_log_row"],
    }
```

### Platform Conditional — fork vs spawn

```python
import sys, multiprocessing as mp

if sys.platform == "win32":
    mp.set_start_method("spawn", force=True)
    # outer_train_df written to temp parquet; workers receive path
    import tempfile
    from pathlib import Path
    tmp_path = Path(tempfile.mktemp(suffix=".parquet"))
    outer_train_df.to_parquet(tmp_path)
    worker_data_kwarg = {"outer_train_path": str(tmp_path), "outer_train_df": None}
else:
    mp.set_start_method("fork", force=True)
    # outer_train_df shared via CoW — no serialization
    worker_data_kwarg = {"outer_train_path": None, "outer_train_df": outer_train_df}

# ... outer fold loop (generate_folds) ...

# Windows: cleanup temp parquet after outer fold completes
if sys.platform == "win32":
    tmp_path.unlink(missing_ok=True)
```

### Thread Allocation

```python
import psutil
effective_num_threads = max(
    1,
    psutil.cpu_count(logical=False) // n_parallel
)
# Physical cores only — HyperThreading excluded per LightGBM documentation.
# n_parallel = config["optimizer"]["parallelism"]["n_parallel_trials"]
# HT cores share L1/L2 cache; LightGBM histogram construction is cache-intensive,
# making HT threads slower rather than faster for this workload.
```

---

## auc_std Recording Rule

```
Completed trials (all Successive Halving rounds finished):
    auc_std = std(trial_auc_history[trial_idx]) if len > 1 else 0.0
    db_conn.execute("UPDATE train_log SET auc_std = ? WHERE run_id = ?",
                    [auc_std, fold_run_ids[-1]])
    fold_run_ids[-1] = last completed round's run_id for that trial

Pruned trials:
    auc_std remains NULL
    is_pruned = TRUE set on fold_run_ids[-1] (last submitted round)

Sequential mode (selection/full): fold_run_ids[-1] = MAX fold_idx,
    guaranteed by generate_folds() ascending order.
    run_id format: {optimizer_run_id}_f{fold_idx} — no collision risk.
Nested/parallel mode: fold_run_ids[-1] = last Successive Halving round completed.
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
          - "exploitation": nested validation with Successive Halving,
                            outer fold evaluation, consensus config, regime holdout
                            returns experiment_log rows (outer_validation + regime_holdout
                            if holdout enabled)
          - "full":         one fold pass, no reducer, backtest
                            returns experiment_log rows
          - "refresh":      routine retraining entry point — the two-depth
                            design (calendar-triggered refit vs.
                            divergence-triggered re-optimization) is
                            specified in docs/ops/shadow_retraining.md's
                            "Retraining Cadence" section, not in this file —
                            this phase has no dedicated algorithm of its
                            own; it dispatches to "full" or "exploitation"
                            depending on which trigger fired (see
                            `optimizer.refresh` config block below).
        """
        ...

    def best_config(self) -> dict:
        """
        Query experiment_log for the config with highest winning_rate
        belonging to this optimizer_run_id.

        UNCHANGED by execution_eval: this ranks MODEL configs and reads only
        rows with execution_variant IS NULL (the baseline pass of each outer
        fold).
        """
        ...

    def best_execution_variant(self) -> dict | None:
        """
        Rank execution variants for this optimizer_run_id on
        optimizer.execution_eval.rank_metric, aggregated across outer folds.
        Returns None when execution_eval is disabled or produced no rows.

        Ranks on avg_pnl_pct, NOT winning_rate, and this is the point rather
        than a detail. Every variant here is judged by the trades it ADDS or
        REMOVES at the margin; a variant that admits marginal-but-positive-EV
        trades lowers the win rate while raising the return, so ranking these
        on winning_rate would systematically reject exactly the variants
        worth having. best_config() is not switched to the same metric,
        because that would propagate into model selection, which this change
        has no business touching.

        Separate from best_config() for the same reason it runs at outer
        folds: the model config and the execution variant are independent,
        so one is chosen without reference to the other.
        """
        ...
```

---

## Config Keys

```yaml
optimizer:
  phase: "exploitation"     # "selection" | "exploitation" | "full" | "refresh"
  selected_features_path: "configs/selected_features.json"

  refresh:
    calendar_cadence_days: 30      # monthly-level; "refit only" trigger (dispatches
                                   # to "full" phase — same hyperparameters/features,
                                   # freshly-fit on extended data)
    divergence_reoptimize: true    # if the divergence check (see health_report.md /
                                   # shadow-retraining spec) fires as severe, dispatch
                                   # to "exploitation" instead of "full" for this cycle

  hyperparameter_search:
    max_trials: 30
    strategy: "random"
    random_state: 42
    search_space:
      lgbm_params:
        num_leaves:       [31, 63, 127]
        min_data_in_leaf: [50, 100, 200]
        feature_fraction: [0.5, 0.7, 0.9]
        lambda_l1:        [0.0, 0.1, 1.0]
      entry_detector:
        session_mode:     ["regular", "pre", "combined"]

  successive_halving:
    enabled: true
    eta: 3                  # top 1/eta fraction survives each round

  parallelism:
    n_parallel_trials: 4    # ProcessPoolExecutor max_workers

  execution_eval:
    enabled: false          # paired execution-variant evaluation at outer folds
    rank_metric: "avg_pnl_pct"
    max_combinations: 8     # grid is evaluated exhaustively — a backtest per
                            # combination per outer fold, so this is the cost
                            # ceiling, not a sampling budget
    late_entry_theta_quantile: 0.10
                            # DEFAULT q for theta's derivation, needed even
                            # when this whole block is disabled because the
                            # offline derivation still has to pick a
                            # quantile. The grid axis below overrides it per
                            # combination — the same relationship
                            # hyperparameter_search's entry_detector
                            # .session_mode has with its own declaration
    grid:
      late_entry_enabled:          [false, true]
      late_entry_theta_quantile:   [0.05, 0.10, 0.25]
    # WHY THIS EXISTS AT ALL: execution-layer parameters do not change the
    # training set or the model, so AUC is identical across their settings —
    # they are unsearchable in hyperparameter_search above, where Successive
    # Halving prunes on AUC and would drop them at random. Affected today:
    # late_entry_enabled, late_entry_theta_quantile, execution.use_all_cash,
    # execution.entry_order_type, and the three circuit-breaker limits that
    # run_backtest.md already computes distributions for so Pilot can
    # calibrate them — the distributions are measured, with no procedure for
    # turning them into values. This is that procedure.
    # late_entry_theta_quantile is DERIVATION-ONLY and is read here and by
    # the offline utility alone; the resolved scalar lives in
    # execution.late_entry_min_expected_return_pct, which is what both
    # engines read (execution_common.md).

  regime_holdout:
    enabled: true
    vol_holdout_percentile: 0.80   # top 20% most volatile dates excluded
    vol_window_days: 30
    vol_metric: "avg_intraday_range"

class_balancer:
  outer_fold:
    window_weeks: 16
    val_weeks:    0
    test_weeks:   6
    step_weeks:   6
    embargo_days: 5
  inner_fold:
    window_weeks: 8
    val_weeks:    2
    test_weeks:   2
    step_weeks:   2
    embargo_days: 5
    max_folds:    4    # cap for nested validation inner loop only;
                       # selection and full phases use max_folds=None
```

---

## Constraints

- `train_log` written by Trainer per fold; `experiment_log` written by Backtester
- In nested validation: workers use `dry_run=True`; coordinator performs all DB INSERTs
- `best_of_loop` set TRUE on the last round row of the winning inner trial per outer fold
- `auc_std` set on `fold_run_ids[-1]` via UPDATE — completed trials only;
  pruned trials retain NULL with is_pruned = TRUE
- Coordinator must NOT call `lgb.train()` before forking workers;
  LightGBM OpenMP initialization in parent process causes fork hang
- Platform-conditional multiprocessing:
  Linux/WSL: fork start method; outer_train_df passed in-memory (CoW)
  Windows: spawn start method; outer_train_df written to temp parquet,
           path passed to workers; file deleted via `tmp_path.unlink(missing_ok=True)`
           after outer fold loop completes
- `if __name__ == "__main__":` guard required for Windows spawn compatibility
- `psutil.cpu_count(logical=False)` used for num_threads allocation;
  HyperThreading excluded per LightGBM documentation (cache contention)
- run_ids for inner trial workers: coordinator pre-generates structured IDs
  as "{optimizer_run_id}_o{outer_fold_idx}_t{trial_idx}_f{inner_fold_idx}";
  generate_run_id() not called inside workers
- run_ids for sequential selection/full phase folds: structured as
  "{optimizer_run_id}_f{fold_idx}"; generate_run_id() not called inside fold loop
- Outer eval run_ids: structured format {optimizer_run_id}_o{outer_fold_idx}_eval;
  generate_run_id() not called inside the outer fold loop
- Phase transition is manual — no automatic switching between phases
- Selection phase does not run backtest — frequency voting only
- Full phase runs backtest after fold pass completes on last fold's test split
- Selection and full phases use max_folds=None in generate_folds() —
  overrides inner_fold.max_folds config cap (which is intended for exploitation inner loop only)
- `session_mode` is a first-class search variable — inner balancer applies session_mode
  per trial from config_override; outer fold generator uses session_mode=None
- Regime holdout: dates removed from remaining_df BEFORE outer fold generation;
  holdout_df used for both final model test_df (train_log AUC) and regime_holdout backtest
- final model test_df: holdout_df (primary, completely unseen) when holdout enabled and
  non-empty; final_val_split (fallback) when holdout disabled
- `compute_vol_regime_holdout()` uses ohlcv_1min regular session bars only
- `compute_consensus_config()` aggregates outer fold best_configs;
  continuous params → median → nearest grid value; categorical → mode
- `temporal_split_simple()` used for early stopping val in outer eval and final model;
  no balancing applied; embargo_days=outer_cfg["embargo_days"] always passed explicitly
- `best_train`, `best_val`, `best_fold_test` patterns from non-nested exploitation removed;
  outer eval uses outer_train_split / outer_val_split / outer_test_df directly
- optimizer_run_id format: YYYYMMDD_HHMMSS (generated once per optimizer run)
- "random" strategy uses random_state for reproducible non-replacement sampling
