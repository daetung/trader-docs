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

Supports these operational phases, each with its own objective:
- **Selection phase**: choose the feature set — Stage A, a competition among
  alternative feature groups, then Stage B, a reducer-config search scored by
  per-fold retraining on each voted set
- **Exploitation phase**: optimize hyperparameters via nested validation with Successive Halving
- **Refit phase**: retrain the deployed model's retained configuration on
  current data — outer evals and a final model, no search
- **Full phase**: reducer disabled entirely; all features used as-is — a
  manual baseline and debugging phase

---

## Architecture Position

**DB access role.** A step guest (db_schema.md's DB File Ownership Windows;
db_coordination.md's Unit scope), run through guest_supervisor.md's
`run_guest`, and a dependent of the Preprocessor work it uses (Work, below):
- `run_guest pipeline_optimizer` (CLI, below) enters
  `register('guest', 'pipeline_optimizer', trigger)` and hands the body a
  work opener and a unit opener;
- every unit the unit opener opens is a
  `guest_unit('pipeline_optimizer', None)`, waiting for admission without
  limit;
- a step holds a read unit and a write unit, its compute — its workers
  included — running between them outside any unit, through
  guest_supervisor.md's step helpers (Work, below);
- BacktestEngine's preloads run in units its connection provider, the unit
  opener, opens; Trainer and Backtester write nothing and return their rows,
  which the coordinator writes in the step's write unit, one transaction
  committed with `work_progress()`; workers open no connection;
- the body calls neither `register()` nor `guest_unit()`;
- a run ended by self-termination leaves its work to resume through
  guest_supervisor.md.

```
Training endpoint:  PipelineOptimizer
    └── its input work's sealed result        ← Preprocessor.load_result() → full_labeled_df (unsplit)
    └── utils.compute_vol_regime_holdout()    ← regime holdout dates, over the pinned range
    └── ClassBalancer.generate_folds()        ← outer and inner rolling folds
    └── Trainer.run(...)                      ← per-fold train (dry_run in workers),
                                                returns train_log_row
    └── Backtester.run(...)                   ← backtest; returns its rows, writes nothing
    └── Coordinator writes                    ← train_log rows and Backtester.write(),
                                                in the step's write unit

Live inference endpoint: Inferencer (separate module)
    └── EntryPointDetector.detect()
    └── FeatureExtractor.extract()            ← direct call (no Preprocessor)
    └── model.load() → resolve_signal()
```

PipelineOptimizer reads its input work's sealed result through
`Preprocessor.load_result()` (run_preprocess.md's Result Loader) and imports
and calls `Trainer` and `Backtester` directly. It constructs no
Preprocessor and invokes no run script as a subprocess.

---

## Work

The optimizer runs as a guest_supervisor.md work — a step tool without a
sealed result — and is a dependent of its input work, the Preprocessor work
it uses.

Config hash and fixed inputs (guest_supervisor.md's Config Hash and Fixed
Inputs):
- `config_hash` covers the full config with the dispatched phase, not
  `'refresh'`, the variant's inputs, and code_versions holding the data and
  training versions (utils.md's Code Versions); the variant's inputs, carried
  in the config snapshot:
  - `'exploitation'`: the content of `optimizer.selected_features_path`;
  - `'refit'`: the retained configuration (Retained configuration, below);
  - `'full'` and `'selection'`: none;
- variant — the dispatched phase;
- `work_id` = `optimizer_run_id`, generated at work creation and reused by a
  resumed work;
- pin — its input work's pin;
- its tool snapshots, outside config_hash:
  - `rate_snapshot` — the rates resolved once at work creation by
    execution_common.md's `resolve_rates()`, with their `rate_sources`;
  - the champion snapshot (Champion snapshot, below);
  - a divergence-dispatched work's divergence week and target run_id
    (Refresh run, below);
- `input_works` — its input work, the seal hash recorded at first
  consumption;
- every execution of the work reads its fixed inputs, never the current
  config or the variant's current inputs: the config is the work's config
  snapshot overlaid with its `rate_snapshot` (execution_common.md's Rate
  Overlay), `selected_features` is the snapshot's content of
  `optimizer.selected_features_path`, and the retained configuration is the
  snapshot's.

Opening — a new run's first unit (a `--resume-work` run is handed its work
already opened):
- computes the input work's pin (run_preprocess.md's Work) and resolves the
  rates (`resolve_rates()`);
- resolves its own work, then its input work, by guest_supervisor.md's
  New-Run Resolution with the run's trigger:
  - `--supersede` applies to its own work and its input work;
  - the rows to create are created together, and any refusal creates none;
  - the input's config_hash is computed from the preprocess keys of this
    config (run_preprocess.md's Config Keys Used, inside config_hash);
  - the reuse case's stale-batch check is run_preprocess.md's;
  - a created work takes the computed pin and its tool snapshots — the
    resolved rates, the champion snapshot and, for a divergence-dispatched
    work, the divergence week and target run_id; a resumed work keeps its
    fixed inputs;
- a manual run reads the retained configuration in this unit (Retained
  configuration, below);
- the run is refused at opening — no work is created and the run ends
  `'error'` naming the reason — when:
  - a manual `'refit'` run has no retained configuration;
  - with `optimizer.execution_eval.enabled`, the phase runs execution
    variants and a θ-using combination has no twin among the combinations
    `max_combinations` keeps (Phase "exploitation"'s execution variants),
    the reason naming the combination;
- a created input work is executed by the `run_guest run_preprocess`
  process the sweep launches (guest_supervisor.md);
- the unit is released before any wait for the input work (Data
  preparation, below).

Data preparation, in every phase, holding no unit:
- waits until its input work is `'done'` with a valid seal
  (guest_supervisor.md's Dependency):
  - an input work turning `'halted'` ends the run `'blocked'`;
  - this work turns superseded when its input work:
    - turns superseded;
    - is found with an invalid seal;
    - is found with a seal hash differing from the one recorded;
- loads the result:
  `full_labeled_df = Preprocessor.load_result(input work_id, its recorded seal hash)`.

Steps — each through the step helpers (below):
- selection:
  - each (fold, alternative);
  - the group decision;
  - each (fold, reducer config);
  - each (fold, candidate);
  - the closing step;
- full — each fold; the closing backtest;
- exploitation:
  - the regime holdout computation and record;
  - each (outer fold, Successive Halving round), the final round also
    carrying `auc_std` and `best_of_loop`;
  - each outer eval;
  - each champion outer eval;
  - each outer fold's execution variants;
  - the closing step, producing:
    - the consensus;
    - the final model;
    - the candidate derivation of θ;
    - the regime holdout report;
    - the champion comparison;
    - the model_candidates row;
- refit:
  - the regime holdout computation and record;
  - each outer eval;
  - the closing step, producing:
    - the final model;
    - the candidate derivation of θ;
    - the regime holdout report;
    - the model_candidates row.

In the code below, steps run through guest_supervisor.md's step helpers:
- `with step(done, read=None, results=None) as s:` — its read unit runs
  `done(conn)` and, for a step not done, `read(conn)` into `s.inputs`; the
  body computes under `if not s.skipped:` holding no unit, a component
  reading the DB taking the unit opener as its connection provider;
  `s.write(fn)` opens the write unit and, after the superseded check, renames
  the step's result files from their temp files, runs `fn(conn)`, then
  `work_progress()` and the commit;
- an exploitation Successive Halving round takes `results`, the round's
  `train_log` rows, as `s.results` when the round is done; no other step
  takes results;
- the regime holdout computation and record is a `with db_step(done) as d:`,
  its work all DB work in one unit.

Resume — a run resuming the work:
- a step whose done check holds is skipped; each step's completion is judged
  from its DB rows or its result file, never from the work record;
- the regime holdout is read from the work's `regime_holdout_dates` rows
  when any exist, otherwise computed over the pinned range and recorded;
  no separate completion record;
- rates come from `rate_snapshot`, not re-resolved;
- the search space is regenerated from `random_state`;
- Successive Halving state (per-trial AUC history, pruning) is rebuilt from
  completed rounds' `train_log` rows (`auc_mean`, `is_pruned`).

Refresh run — the phase `'refresh'`, given by `--phase refresh`, launched by
metadata_crawler.md's dispatcher at `optimizer.refresh.schedule` as
`run_guest pipeline_optimizer --phase refresh --trigger scheduled`
(shadow_retraining.md); it decides the dispatched phase (`'refit'` or
`'exploitation'`) before opening its work. In one unit, which also reads the
retained configuration and carries it into the opening:
- it first looks for the tool's unfinished work, the work opener applying
  the code-version supersede first (guest_supervisor.md's Code Versions of a
  Work) — a work it supersedes answers as none:
  - held by a live run, and the refresh run would have resumed it — its
    config_hash matches the current config's under `'refit'` or
    `'exploitation'` → the run ends `'normal'` without evaluating the
    triggers and without a refusal;
  - held by a live run otherwise → the run is refused, a scheduled refusal;
  - not held → its config_hash is compared with the hashes of the current
    config under `'refit'` and under `'exploitation'` — with no retained
    configuration, under `'exploitation'` alone; a match resolves it by
    New-Run Resolution under the matching phase; no match → the run is
    refused, the triggers not evaluated;
  - none → the calendar and divergence triggers (shadow_retraining.md's
    Retraining Cadence) are evaluated; no trigger firing → the run ends
    `'normal'` without a work; otherwise it resolves its work under the
    dispatched phase;
- the dispatched phase:
  - the divergence trigger fired and `optimizer.refresh.divergence_reoptimize`
    is true → `'exploitation'`;
  - otherwise, a fired trigger → `'refit'`;
  - `'refit'` with no retained configuration → `'exploitation'`;
- a work dispatched by the divergence trigger carries in its tool snapshots
  its divergence week, the last completed week's week_start, and the target
  run_id, written with the work in New-Run Resolution's transaction;
- the calendar trigger's last refit is the `updated_at` of the latest
  pipeline_optimizer work row closed `'done'` whose variant is `'refit'` or
  `'exploitation'` (guest_supervisor.md's work records; a closed row is not
  updated again); these do not satisfy the cadence:
  - a refused run;
  - a superseded work;
  - a halted work;
- a `--resume-work` run skips the trigger evaluation.

Retained configuration — of the deployed model R, the run_id of
model_deployments' latest row:
- the `feature_config` of R's final-model `train_log` row — optimizer_run_id
  set, phase `'exploitation'` or `'refit'`, outer_fold_idx -1, fold_idx -1,
  trial_idx 0;
- the `feature_names` of R's meta.json.
There is no retained configuration when any of these holds:
- model_deployments has no row;
- R has no such `train_log` row;
- R's feature_names are not a subset of
  `FeatureExtractor(config).get_feature_names()` under the current config;
- a feature_config key is not a path of the current config.
A run reads it once: a refresh run in its first unit (Refresh run, above),
carrying it into its opening; a manual run in its opening unit.

Champion snapshot — an `'exploitation'` or `'refit'` work's, in its tool
snapshots, outside config_hash:
- set at work creation, in New-Run Resolution's transaction, from the read
  that gives the run's retained configuration; a resumed work keeps it;
- with a model_deployments row: R's run_id, and for `'exploitation'` R's
  retained configuration when one exists;
- with none: no champion snapshot.
model_candidates.champion_run_id is the snapshot's run_id, NULL without a
snapshot. An `'exploitation'` work whose snapshot holds a retained
configuration runs champion outer evals (Phase "exploitation", below).

---

## Operational Phases

### Phase: "selection"

**Objective:** Choose the feature set over the selection phase's rolling
folds through Stage A, a competition among alternative feature groups
(`optimizer.selection.group_alternatives`), then Stage B, a search over
reducer configs (`optimizer.selection.search_space`) whose voted sets are
scored by per-fold reducer-off retraining, the pick made through
`dimensionality_reducer.auc_tolerance`.

```
- feature set: Stage A — per alternative, get_feature_names() minus the
  columns of its set's other groups; Stage B — its input set, then each
  candidate's voted set
- DimensionalityReducer: Stage B-1 only (run_reducer=True)
- Execution: per fold, each stage's runs (no trial loop)
- trial_idx: always 0 (single implicit trial)
- fold_idx: 0-based inner fold index; outer_fold_idx: -1 (non-nested)
- No backtest — returns empty DataFrame
- train_log: one row per run, phase 'selection'; feature_config holds the
  run's stage descriptor — Stage A: the alternative; Stage B-1: the reducer
  config overrides; Stage B-2: the candidate
- auc_std: set by the closing step on the chosen candidate's run_id at the
  largest fold_idx; with no eligible candidate, on the baseline run at the
  largest fold_idx
```

Selection-start validation raises ConfigurationError when (Config Keys):
- a `group_alternatives` group is not a key of `get_feature_groups()`;
- a group appears in more than one set;
- the `search_space` grid holds more points than `max_candidates`.

```python
optimizer_run_id = work.work_id                  # Work, above
full_labeled_df  = <Data preparation, Work above>
resume_dir       = f"data/processed/resume/{optimizer_run_id}"
sel_cfg          = config["optimizer"]["selection"]
inner_cfg        = config["class_balancer"]["inner_fold"]   # 06_class_balancer.md

extractor    = FeatureExtractor(config)
all_features = extractor.get_feature_names()
groups       = extractor.get_feature_groups()    # group name → its columns
validate_selection_config(sel_cfg, groups)       # the validation above

def minus(names, drop_groups):
    drop = {col for g in drop_groups for col in groups[g]}
    return [f for f in names if f not in drop]

balancer = ClassBalancer(config)
folds = list(balancer.generate_folds(
    full_labeled_df,
    balance=config["class_balancer"]["apply_balance"],
    window_weeks=inner_cfg["window_weeks"],
    val_weeks=inner_cfg["val_weeks"],
    test_weeks=inner_cfg["test_weeks"],
    step_weeks=inner_cfg["step_weeks"],
    embargo_days=inner_cfg["embargo_days"],
    max_folds=None,          # selection phase uses all available folds
))
fold_idxs = [fold_meta["fold_idx"] for *_, fold_meta in folds]

def selection_run(fold, run_id, names, descriptor, cfg=config,
                  run_reducer=False, report_path=None):
    # Step: one selection run — done when its run_id is in train_log
    train, val, test, fold_meta = fold
    with step(done=lambda conn: in_train_log(conn, run_id)) as s:
        if not s.skipped:
            result = Trainer(cfg, optimizer_run_id=optimizer_run_id).run(
                train, val, test,
                run_id=run_id,
                feature_names=names,
                feature_config=descriptor,     # the run's stage descriptor
                fold_idx=fold_meta["fold_idx"],
                outer_fold_idx=-1,
                fold_train_start=fold_meta["fold_train_start"],
                fold_train_end=fold_meta["fold_train_end"],
                fold_train_days=fold_meta["fold_train_days"],
                run_reducer=run_reducer,
                phase="selection",
                trial_idx=0,
            )
            if report_path:                    # temp file; s.write() renames
                save_json(result["reduction_report"], report_path)
            s.write(lambda conn: insert_train_log(conn, result["train_log_row"]))

# --- Stage A: the group competition ---
# Each set is evaluated independently; the groups of every other set stay
# in. k: the alternative's position in group_alternatives flattened in set
# order, then group order.
alternatives = [(alts, g) for alts in sel_cfg["group_alternatives"] for g in alts]
for fold, fold_idx in zip(folds, fold_idxs):
    for k, (alts, g) in enumerate(alternatives):
        selection_run(fold, f"{optimizer_run_id}_f{fold_idx}_g{k}",
                      minus(all_features, [h for h in alts if h != g]),
                      {"stage": "A", "alternative": g})

# Step: the group decision — done when group_decision.json exists
decision_path = f"{resume_dir}/group_decision.json"
with step(done=lambda conn: os.path.exists(decision_path),
          read=lambda conn: <train_log.auc_mean of every Stage A run_id>) as s:
    if not s.skipped:
        # score: a run's train_log.auc_mean
        # fold winner: the highest score; a tie goes to the earlier-listed group
        # set winner: the most fold wins; a tie goes to the earlier-listed group
        save_json(<each set's winner>, decision_path)   # temp file; s.write() renames
        s.write(lambda conn: None)
losing_groups = <every set's groups other than its winner, from decision_path>
input_set     = minus(all_features, losing_groups)       # Stage B's input set

# --- Stage B: the reducer-config search ---
# grid: the Cartesian product of search_space in key order, the last key
# varying fastest; a candidate is a grid point, c its position in grid order.
# A reducer config is a grid point without its vote_threshold, taken in grid
# order with duplicates dropped; j is its position in that order.
VT = "dimensionality_reducer.vote_threshold"
grid            = cartesian(sel_cfg["search_space"])
reducer_configs = unique_in_order([drop_key(p, VT) for p in grid])

def report_path(fold_idx, j):
    return f"{resume_dir}/reduction_f{fold_idx}_r{j}.json"

# Stage B-1: the reducer, per fold and per reducer config
for fold, fold_idx in zip(folds, fold_idxs):
    for j, rc in enumerate(reducer_configs):
        selection_run(fold, f"{optimizer_run_id}_f{fold_idx}_r{j}",
                      input_set, {"stage": "B-1", "overrides": rc},
                      cfg=utils.apply_overrides(config, rc),
                      run_reducer=True, report_path=report_path(fold_idx, j))

# Stage B-2: each candidate's voted set, then reducer-off runs on it
voted = {}
for c, point in enumerate(grid):
    j = reducer_configs.index(drop_key(point, VT))
    voted[c] = frequency_vote([load_json(report_path(i, j)) for i in fold_idxs],
                              threshold=point[VT])
if not any(voted.values()):
    raise <error naming the work>   # writes neither selected_features.json nor
                                    # feature_selection_log.json; sets no auc_std
candidates = <each c in grid order, skipped when voted[c] is empty or equals
              an earlier candidate's voted set>
for fold, fold_idx in zip(folds, fold_idxs):
    for c in candidates:
        selection_run(fold, f"{optimizer_run_id}_f{fold_idx}_c{c}",
                      voted[c], {"stage": "B-2", "candidate": grid[c]})

# Step: the closing step — done when auc_std is set on the pick's run_id; its
# read unit recomputes the pick from train_log, on resume as well
def pick(conn):
    # baseline: per fold, the auc_mean of the fold's Stage B-1 run at reducer
    #   config position 0 (_r0); the baseline score is their mean over folds
    # a candidate's score: the mean over folds of its runs' auc_mean
    # eligible: score >= baseline score - dimensionality_reducer.auc_tolerance
    # chosen, among eligible: the fewest features, then the higher score,
    #   then the earlier grid order; none eligible → no chosen candidate
    # std_run_id: the chosen candidate's run_id at the largest fold_idx, or
    #   with none chosen the baseline run at the largest fold_idx
    return <chosen, std_run_id, the per-fold auc_means read>

with step(done=lambda conn: auc_std_set(conn, pick(conn).std_run_id),
          read=pick) as s:
    if not s.skipped:
        chosen, std_run_id, fold_aucs = s.inputs
        selected_features = voted[chosen] if chosen is not None else input_set
        # each file a temp file; s.write() renames each before the commit
        save_json(selected_features, config["optimizer"]["selected_features_path"])
        save_json(<feature_selection_log, below>, "configs/feature_selection_log.json")
        auc_std = std(fold_aucs) if len(fold_aucs) > 1 else 0.0  # over folds, of the
                                                    # chosen candidate's or baseline's auc_mean
        s.write(lambda conn: conn.execute(
            "UPDATE train_log SET auc_std = ? WHERE run_id = ?",
            [auc_std, std_run_id]))
```

`configs/feature_selection_log.json` holds:
- the group decisions, per set: each fold's auc_mean per alternative, each
  fold's winner, the set winner;
- the baseline: per fold and its mean;
- the candidate table, per candidate:
  - its overrides and its vote_threshold;
  - its feature count;
  - its per-fold and mean auc_mean;
  - its skip reason, one of empty voted set or duplicate set;
  - its eligibility;
  - its chosen flag;
- the chosen candidate's reducer config reports, per fold; none when no
  candidate is eligible.

**Post-selection follow-up (manual, not automated by this phase):** review
`selected_features` for any of the 9 tick-derived indicators (see
02_indicator_calculator.md's Tick-Derived Indicator Scale Sensitivity
registry). Indicators that survived selection should have their
`precalculate_bars` config flipped from the default `0` to `"lookback"` for
live deployment — see the rationale note alongside that config block. This
step is deliberately not triggered automatically by the selection phase
writing selected_features.json; it is a config decision made once
selected_features stabilizes,
not a per-run action.

---

### Phase: "exploitation"

**Objective:** Search over LightGBM hyperparameters to find the best configuration
using nested walk-forward validation (outer fold × inner fold × Successive Halving).
Outer fold evaluation is unbiased — never seen during inner trial search.

#### Data Preparation

```python
optimizer_run_id = work.work_id                  # Work, above
full_labeled_df  = <Data preparation, Work above>

selected_features = <the work's config snapshot: the content of
                     optimizer.selected_features_path>   # fixed input, never the file
champion          = <the work's champion snapshot (Champion snapshot, Work
                     above): its run_id and, when present, its retained
                     configuration; None without a snapshot>
unit_opener       = <the unit opener run_guest hands the body — the
                     connection provider of every Backtester below>

# Regime holdout — remove high-volatility dates before any fold generation.
# Consequence worth stating: every outer and inner fold below is generated
# from remaining_df, the volatility-truncated remainder, while the final
# model is evaluated on holdout_df. Configuration selection therefore runs
# on a different population from the final evaluation, and an
# eval_type='outer_validation' score is not an estimate over the whole
# distribution (db_schema.md's experiment_log).
holdout_cfg = config["optimizer"]["regime_holdout"]
start_date, end_date = <the work's pinned trading-date range>

# Step: the regime holdout computation and record — a db_step(), its work
# all DB work in one unit; shared by the refit phase. The set is computed once
# per work over the pinned range and recorded: the work's
# regime_holdout_dates rows when any exist, otherwise computed and recorded —
# no separate completion record. Every row of this optimizer_run_id that
# excludes or tests on it reads the same set (db_schema.md).
with db_step(done=lambda conn: False) as d:
    conn = d.conn
    holdout_vol = <regime_holdout_dates rows of optimizer_run_id: {date: rolling_vol}>
    if not holdout_vol and holdout_cfg["enabled"]:
        holdout_vol = utils.compute_vol_regime_holdout(
            conn,
            start_date,
            end_date,
            vol_percentile=holdout_cfg["vol_holdout_percentile"],
            window_days=holdout_cfg["vol_window_days"],
            vol_metric=holdout_cfg["vol_metric"],
        )
        if holdout_vol:
            conn.executemany(
                "INSERT INTO regime_holdout_dates (optimizer_run_id, date, rolling_vol) VALUES (?, ?, ?)",
                [(optimizer_run_id, d, v) for d, v in sorted(holdout_vol.items())],
            )
holdout_dates = set(holdout_vol)

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
```

#### Nested Validation — Outer Fold Loop

```python
# Rates: the work's rate_snapshot (Work, above) — resolved once at work
# creation by execution_common.md's resolve_rates() and overlaid on this
# in-memory `config` (execution_common.md's Rate Overlay); a resumed work
# overlays the same snapshot and never re-resolves. The config file is never
# written. Every apply_overrides() below applies on top of the overlay:
#   - workers and Backtesters receive the resolved config;
#   - every Backtester receives rate_sources;
#   - BacktestEngine does not resolve again.
rate_sources = <the work's rate_snapshot rate_sources>

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

for outer_train_df, _, outer_test_df, outer_fold_meta in balancer.generate_folds(
    remaining_df,
    balance=False,
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
    search_space   = sample_hyperparams(config, max_trials)   # from random_state —
                                                            # the same on resume
    active_trials  = list(enumerate(search_space))
    trial_auc_history: dict[int, list[float]] = {i: [] for i in range(len(active_trials))}
    max_inner_folds = inner_cfg["max_folds"]

    for round_idx in range(max_inner_folds):

        # Coordinator pre-generates run_ids (avoid collision in parallel workers)
        round_run_ids = {
            trial_idx: f"{optimizer_run_id}_o{outer_fold_idx}_t{trial_idx}_f{round_idx}"
            for trial_idx, _ in active_trials
        }

        final_round = round_idx == max_inner_folds - 1

        # Step: this (outer fold, round) — done when its run_ids are in
        # train_log; the final round also carries auc_std and best_of_loop.
        # Its results, for a skipped round: the round's train_log rows, read in
        # its read unit
        with step(done=lambda conn: in_train_log(conn, *round_run_ids.values()),
                  results=lambda conn: <train_log rows of round_run_ids>) as s:
            if not s.skipped:
                # Dispatch parallel workers — holding no unit
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
                )

                history = {r["trial_idx"]: trial_auc_history[r["trial_idx"]] + [r["auc_mean"]]
                           for r in round_results}

                # Successive Halving pruning (not on final round)
                pruned_idx = set()
                if not final_round and len(active_trials) > 1:
                    trial_means = {
                        idx: mean(history[idx])
                        for idx, _ in active_trials
                        if history.get(idx)
                    }
                    sorted_trials  = sorted(trial_means.items(), key=lambda x: x[1], reverse=True)
                    n_survive      = max(1, len(sorted_trials) // eta)
                    pruned_idx     = {idx for idx, _ in sorted_trials[n_survive:]}

                # Final round: auc_std and best_of_loop for completed trials
                if final_round:
                    completed_trial_means = {
                        idx: mean(history[idx])
                        for idx, _ in active_trials
                    }
                    best_trial_idx = max(completed_trial_means, key=completed_trial_means.get)

                def write(conn):
                    # Coordinator: the round's rows in the step's write unit
                    for r in round_results:
                        insert_train_log(conn, r["train_log_row"])
                    for r in round_results:          # mark pruned trial rows
                        if r["trial_idx"] in pruned_idx:
                            conn.execute(
                                "UPDATE train_log SET is_pruned = TRUE WHERE run_id = ?",
                                [r["run_id"]]
                            )
                    if final_round:
                        for idx, _ in active_trials:
                            aucs = history[idx]
                            conn.execute(
                                "UPDATE train_log SET auc_std = ? WHERE run_id = ?",
                                [std(aucs) if len(aucs) > 1 else 0.0, round_run_ids[idx]]
                            )
                        conn.execute(
                            "UPDATE train_log SET best_of_loop = TRUE WHERE run_id = ?",
                            [round_run_ids[best_trial_idx]]
                        )
                s.write(write)

        # The round's rows (trial_idx, auc_mean, is_pruned, best_of_loop) —
        # from memory on a computed round (round_results with pruned_idx and
        # best_trial_idx), s.results on a skipped one
        rows = <the round's rows>
        for row in rows:
            trial_auc_history[row.trial_idx].append(row.auc_mean)
        active_trials = [(idx, hp) for idx, hp in active_trials
                         if idx in {row.trial_idx for row in rows if not row.is_pruned}]

    best_trial_idx = <trial_idx of the best_of_loop row among the final round's rows>
    best_config    = search_space[best_trial_idx]

    # --- Outer eval: train best_config on outer_train, backtest on outer_test ---
    outer_train_split, outer_val_split = utils.temporal_split_simple(
        outer_train_df,
        val_fraction=config["class_balancer"]["early_stopping_val_fraction"],
        embargo_days=outer_cfg["embargo_days"],
    )
    # train_log's fold_train_* describe the dates actually trained on — the
    # train split, not outer_fold_meta's window, which also holds the val
    # split and its embargo.
    outer_train_dates = outer_train_split["date"]

    outer_run_id    = f"{optimizer_run_id}_o{outer_fold_idx}_eval"
    config_override = utils.apply_overrides(config, best_config)
    config_override["lgbm_params"]["num_threads"] = effective_num_threads

    def outer_eval(run_id, cfg, feature_config, feature_names, trial_idx, eval_type):
        # Step: one outer eval — done when run_id is in experiment_log
        with step(done=lambda conn: in_experiment_log(conn, run_id)) as s:
            if not s.skipped:
                train_result = Trainer(cfg, optimizer_run_id=optimizer_run_id).run(
                    outer_train_split, outer_val_split, outer_test_df,
                    run_id=run_id,
                    fold_idx=-1,
                    outer_fold_idx=outer_fold_idx,
                    fold_train_start=outer_train_dates.min(),
                    fold_train_end=outer_train_dates.max(),
                    fold_train_days=outer_train_dates.nunique(),
                    feature_config=feature_config,
                    feature_names=feature_names,
                    run_reducer=False,
                    phase="exploitation",
                    trial_idx=trial_idx,   # not part of inner trial loop
                    dry_run=False,
                )
                backtester = Backtester(cfg, unit_opener, optimizer_run_id=optimizer_run_id,
                                        rate_sources=rate_sources)
                bt_result = backtester.run(
                    outer_test_df,
                    run_id=run_id,
                    fold_idx=-1,
                    outer_fold_idx=outer_fold_idx,
                    fold_test_start=outer_fold_meta["fold_test_start"],
                    fold_test_end=outer_fold_meta["fold_test_end"],
                    fold_test_days=outer_fold_meta["fold_test_days"],
                    eval_type=eval_type,
                )

                def write(conn):
                    insert_train_log(conn, train_result["train_log_row"])
                    backtester.write(conn, bt_result)
                s.write(write)

    # The challenger outer eval
    outer_eval(outer_run_id, config_override, best_config, selected_features,
               trial_idx=-1, eval_type="outer_validation")

    # --- Champion outer eval: the deployed model's retained configuration on
    # the same outer_train and outer_test (Champion snapshot, Work above) ---
    if champion is not None and champion.retained is not None:
        champ_override = utils.apply_overrides(config, champion.retained.feature_config)
        champ_override["lgbm_params"]["num_threads"] = effective_num_threads
        outer_eval(f"{optimizer_run_id}_o{outer_fold_idx}_champ", champ_override,
                   champion.retained.feature_config, champion.retained.feature_names,
                   trial_idx=-2, eval_type="champion_validation")

    # --- Execution-variant evaluation (optimizer.execution_eval) ---
    # Same model, same fold, same data — only execution keys differ, so the
    # comparison is statistically PAIRED. Runs here and not in the inner
    # loop because execution parameters change neither the training set nor
    # the model: AUC is identical across their settings, and Successive
    # Halving prunes on AUC, so a flag toggled there would be pruned at
    # random. Cost is per outer fold, not per trial, and a backtest is far
    # cheaper than a training run.
    ee = config["optimizer"]["execution_eval"]
    if ee["enabled"]:
        # THE TWIN RULE — a combination uses θ when late_entry_enabled or
        # entry_price_cap_enabled is true. Its θ is the q-th quantile of its
        # twin's on_time_expected_returns, so it is underivable until its twin
        # has run: its twin is the first combination with the same keys except
        # late_entry_theta_quantile, with late_entry_enabled false and
        # entry_price_cap_enabled false. A run whose kept combinations leave a
        # θ-using one without a twin is refused at opening (Work's Opening).
        # See execution_common.md's Late-Entry Residual Edge.
        import itertools, json
        grid = ee["grid"]
        combos = [
            dict(zip(grid.keys(), values))
            for values in itertools.product(*grid.values())
        ]
        def uses_theta(c):
            return bool(c.get("late_entry_enabled") or c.get("entry_price_cap_enabled"))
        # θ-using combinations last — see the twin rule above.
        combos.sort(key=uses_theta)
        combos = combos[: ee["max_combinations"]]
        variant_run_ids = [f"{outer_run_id}_x{i}" for i in range(len(combos))]

        # Step: this outer fold's execution variants — done when every variant
        # run_id of the fold is in experiment_log
        with step(done=lambda conn: in_experiment_log(conn, *variant_run_ids)) as s:
            if not s.skipped:
                backtesters, results = {}, {}
                for i, variant in enumerate(combos):          # in sorted order
                    variant_override = utils.apply_overrides(config_override, variant)
                    if uses_theta(variant):
                        returns = results[twin_of(combos, i)]["on_time_expected_returns"]
                        q = variant.get("late_entry_theta_quantile",
                                        ee["late_entry_theta_quantile"])
                        theta = (quantile_linear(returns, q)   # linear interpolation
                                 if len(returns) >= ee["late_entry_theta_min_entries"]
                                 else None)
                        variant_override = utils.apply_overrides(variant_override, {
                            "execution.late_entry_min_expected_return_pct": theta,
                            "execution.late_entry_theta_run_id": outer_run_id,
                            "execution.late_entry_theta_q": q,
                        })
                    backtesters[i] = Backtester(
                        variant_override, unit_opener, optimizer_run_id=optimizer_run_id,
                        rate_sources=rate_sources,
                    )
                    results[i] = backtesters[i].run(
                        outer_test_df,
                        run_id=variant_run_ids[i],
                        model_run_id=outer_run_id,   # the outer eval's model
                        fold_idx=-1,
                        outer_fold_idx=outer_fold_idx,
                        fold_test_start=outer_fold_meta["fold_test_start"],
                        fold_test_end=outer_fold_meta["fold_test_end"],
                        fold_test_days=outer_fold_meta["fold_test_days"],
                        eval_type="outer_validation",
                        execution_variant=json.dumps(variant),  # the combination's keys alone
                    )

                def write(conn):                     # one transaction
                    for i in results:
                        backtesters[i].write(conn, results[i])
                s.write(write)

    outer_best_configs.append(best_config)
    # each fold's winning_rate is already persisted as an
    # eval_type='outer_validation' experiment_log row; no second copy here

# Step: the closing step, producing:
#   - the consensus;
#   - the final model;
#   - the candidate derivation of θ;
#   - the regime holdout report;
#   - the champion comparison;
#   - the model_candidates row;
# done when this optimizer_run_id's final model train_log row exists (phase
# 'exploitation', outer_fold_idx -1, fold_idx -1, trial_idx 0 —
# final_model_row_exists() tests the work's phase); final_run_id is generated
# in the step, so its completion is judged by that row. Its read unit reads the
# Regime Holdout Report's DB inputs and the champion comparison's (below).
with step(done=lambda conn: final_model_row_exists(conn, optimizer_run_id),
          read=lambda conn: <the Regime Holdout Report's and the champion
                             comparison's DB inputs>) as s:
    if not s.skipped:
        # --- Consensus config → final model ---
        consensus_config = utils.compute_consensus_config(
            outer_best_configs,
            config["optimizer"]["hyperparameter_search"]["search_space"],
        )
        final_config_override = utils.apply_overrides(config, consensus_config)

        final_train_split, final_val_split = utils.temporal_split_simple(
            remaining_df,
            val_fraction=config["class_balancer"]["early_stopping_val_fraction"],
            embargo_days=outer_cfg["embargo_days"],
        )
        final_run_id = utils.generate_run_id()
        final_train_dates = final_train_split["date"]   # as outer_train_dates above

        # test_df: holdout_df (completely unseen, preferred) when holdout is enabled and non-empty;
        #          final_val_split as fallback when holdout is disabled (val is next-best available).
        #          Note: val_split is used by LightGBM early stopping but does not overlap with train.
        final_test_df = (
            holdout_df
            if holdout_cfg["enabled"] and len(holdout_df) > 0
            else final_val_split
        )

        final_result = Trainer(final_config_override, optimizer_run_id=optimizer_run_id).run(
            final_train_split, final_val_split, final_test_df,
            run_id=final_run_id,
            fold_idx=-1,
            outer_fold_idx=-1,
            fold_train_start=final_train_dates.min(),
            fold_train_end=final_train_dates.max(),
            fold_train_days=final_train_dates.nunique(),
            feature_config=consensus_config,
            feature_names=selected_features,
            run_reducer=False,
            phase="exploitation",
            trial_idx=0,
            dry_run=False,
        )

        # --- Candidate derivation of θ (Model Candidate, below) ---
        late_entry_theta = derive_candidate_theta(final_config_override, final_run_id,
                                                  final_val_split)

        # --- Regime holdout robustness check ---
        regime = verdict_row = None
        if holdout_cfg["enabled"] and len(holdout_df) > 0:
            regime_backtester = Backtester(
                final_config_override, unit_opener, optimizer_run_id=optimizer_run_id,
                rate_sources=rate_sources,
            )
            # No fold_test_* argument: run_backtest derives them from holdout_df. The
            # date pair is then the ends of a non-contiguous set, whose members are in
            # regime_holdout_dates.
            regime = regime_backtester.run(
                holdout_df,
                run_id=final_run_id,
                fold_idx=-1,
                outer_fold_idx=-1,
                eval_type="regime_holdout",
            )

            # Report and verdict — see "Regime Holdout Report and Verdict" below.
            # Always computed, in this step, from s.inputs and from memory. The
            # verdict blocks a session's participation at live_mode_runner.md's
            # Step 7 alone; model_candidates' status at writing (Model Candidate,
            # below) and model_deploy's decisions also read it.
            verdict_row = regime_holdout_report(s.inputs, regime["experiment_log_row"],
                                                final_result["train_log_row"])

        # --- Champion comparison — see "Champion Comparison" below ---
        comparison = champion_comparison(s.inputs, champion, phase="exploitation")

        def write(conn):
            insert_train_log(conn, final_result["train_log_row"])
            if regime is not None:
                regime_backtester.write(conn, regime)
                insert_regime_holdout_verdict(conn, verdict_row)
            # the model_candidates row and its status, in this transaction
            write_candidate(conn, final_run_id, "exploitation", champion, verdict_row,
                            comparison, late_entry_theta)        # Model Candidate, below
        s.write(write)
```

#### Regime Holdout Report and Verdict

Computed at the end of every exploitation or refit run that writes a
'regime_holdout' `experiment_log` row, in the closing step. It writes one
`regime_holdout_verdict` row (db_schema.md), in the closing step's write
unit, and prints the report.

- **Inputs:**
  - read in the closing step's read unit:
    - this `optimizer_run_id`'s 'outer_validation' `experiment_log` rows
      with `execution_variant` NULL;
    - the outer eval `train_log` rows, those with `trial_idx` -1;
    - the `trade_log` rows of those 'outer_validation' runs;
  - taken from memory:
    - the regime run's `experiment_log_row`;
    - the final model's `train_log_row`.
- **Report items:**
  - from the 'regime_holdout' `experiment_log` row: `total_trades`,
    `fold_test_days`, `winning_rate`, `avg_pnl_pct`
  - from the final model's `train_log` row (test = `holdout_df`): `auc_mean`
  - each beside the mean, min and max of the same item over this
    `optimizer_run_id`'s 'outer_validation' `experiment_log` rows with
    `execution_variant` NULL — for `auc_mean`, over the outer eval `train_log`
    rows
- **Bootstrap CI (report only):**
  - population: the `trade_log` rows of those 'outer_validation' runs, entered
    trades only, resampled per trade by `utils.bootstrap_ci()`; sample size =
    the regime `total_trades`; `optimizer.bootstrap` sets resamples,
    confidence and seed
  - `winning_rate` (share of `pnl_pct > 0`) and `avg_pnl_pct` (mean
    `pnl_pct`) each get `ci_low`, `ci_high`, `in_ci` (the regime value lies in
    [`ci_low`, `ci_high`]) and `ci_position` ('below' | 'inside' | 'above')
  - no gate check reads them; `auc_mean` gets no CI
- **Gate checks:**
  - regime `winning_rate` ≥ the outer folds' min `winning_rate`
  - final model `auc_mean` ≥ the outer eval rows' min `auc_mean`
- **Verdict:** 'pass' when every check passes, 'fail' otherwise. When the
  regime `total_trades` < `optimizer.regime_holdout.gate.min_trades`, the
  checks are not evaluated and the verdict is
  `optimizer.regime_holdout.gate.insufficient_sample`'s value.
- **Row:** `optimizer_run_id`, `final_run_id`, `detail` (JSON of the report
  items, the CI fields and each check's result), `verdict`, `evaluated_at`.

#### Champion Comparison

In the exploitation closing step. Its inputs, read in the closing step's read
unit beside the Regime Holdout Report's: this `optimizer_run_id`'s
'champion_validation' `experiment_log` rows and the `trade_log` rows of those
runs.

- **Unit:** (outer fold k, a trading date of outer_test_df_k).
- **A unit's value:** the sum of `pnl_pct` over the trades entered that date
  and counted in `total_trades`, the challenger's `execution_variant` NULL
  outer eval minus the champion outer eval, a model without trades
  contributing 0.
- **CI:** `utils.bootstrap_ci()` over the units' values, statistic mean,
  sample_size the unit count, with `optimizer.bootstrap`'s keys.
- **Verdict**, decided in this order:
  - 'not_applicable': a refit work;
  - 'no_champion': no champion snapshot;
  - 'champion_unavailable': a champion snapshot without a retained
    configuration;
  - 'insufficient_sample': the dates on which either model traded are fewer
    than `optimizer.promotion.min_trade_days`;
  - 'pass': `ci_low` ≥ −`optimizer.promotion.noninferiority_margin`;
  - 'fail': otherwise.
- **Detail** (`model_candidates.comparison_detail`), NULL when the verdict is
  'not_applicable', 'no_champion' or 'champion_unavailable':
  - Δ: the mean of the units' values;
  - the CI;
  - the date counts: the units, and the dates on which either model traded;
  - each model's `winning_rate`, `avg_pnl_pct` and `total_trades`.

#### Model Candidate

The 'exploitation' and 'refit' closing steps write one `model_candidates`
row (db_schema.md) in their write unit's transaction:
- `final_run_id`, the key; `optimizer_run_id`; `phase`;
- `champion_run_id`: the champion snapshot's run_id; NULL without a snapshot;
- `regime_verdict`: the step's `regime_holdout_verdict` row's verdict; NULL
  when the step writes none;
- `regime_gate_blocked`: `utils.regime_gate()`'s blocked over the step's
  `regime_holdout_verdict` row, under the work's config; none when the step
  writes no such row;
- `comparison_verdict` and `comparison_detail` (Champion Comparison, above);
- `late_entry_theta` — the candidate derivation of θ: a twin pass of the final
  model over final_val_split, under the final model's config with
  `late_entry_enabled` and `entry_price_cap_enabled` false, writing no rows;
  per q of the θ quantile set (Config Keys' `execution_eval`) and of
  `execution.late_entry_theta_q` when set, θ — the q-th quantile by linear
  interpolation of the pass's `on_time_expected_returns`, null below
  `optimizer.execution_eval.late_entry_theta_min_entries` — as JSON
  `{q: {theta, n}}`;
- `created_at`; `decided_at` and `decision_note` NULL.

Status at writing, in this order:
- `champion_run_id` differs from the run_id of model_deployments' latest row,
  a NULL `champion_run_id` standing for no deployment → `'superseded'`,
  superseding none;
- `regime_gate_blocked` true or the comparison `'fail'` → `'rejected'`,
  superseding none;
- otherwise `'awaiting_approval'`, and every older `'awaiting_approval'` row
  turns `'superseded'` in the same transaction, its `decided_at` set and its
  `decision_note` NULL.

---

### Phase: "refit"

**Objective:** Retrain the deployed model's retained configuration (Retained
configuration, Work above) on current data, without a search.

Data preparation and the regime holdout computation and record are the
exploitation phase's (Data Preparation, above). The phase runs none of:
- an inner search;
- Successive Halving;
- execution variants;
- the consensus;
- champion outer evals.

```python
# remaining_df, balancer, unit_opener: Data Preparation, above
rate_sources   = <the work's rate_snapshot rate_sources>   # as in the exploitation phase
outer_cfg      = config["class_balancer"]["outer_fold"]
retained       = <the work's config snapshot: the retained configuration —
                  feature_config and feature_names>   # fixed input
refit_override = utils.apply_overrides(config, retained.feature_config)

for outer_train_df, _, outer_test_df, outer_fold_meta in balancer.generate_folds(
    remaining_df,
    balance=False,
    window_weeks=outer_cfg["window_weeks"],
    val_weeks=0,
    test_weeks=outer_cfg["test_weeks"],
    step_weeks=outer_cfg["step_weeks"],
    embargo_days=outer_cfg["embargo_days"],
):
    outer_fold_idx = outer_fold_meta["fold_idx"]
    outer_train_split, outer_val_split = utils.temporal_split_simple(
        outer_train_df,
        val_fraction=config["class_balancer"]["early_stopping_val_fraction"],
        embargo_days=outer_cfg["embargo_days"],
    )
    outer_train_dates = outer_train_split["date"]
    outer_run_id      = f"{optimizer_run_id}_o{outer_fold_idx}_eval"

    # Step: this outer eval — done when outer_run_id is in experiment_log; as
    # the exploitation phase's outer eval, with phase 'refit'
    with step(done=lambda conn: in_experiment_log(conn, outer_run_id)) as s:
        if not s.skipped:
            train_result = Trainer(refit_override, optimizer_run_id=optimizer_run_id).run(
                outer_train_split, outer_val_split, outer_test_df,
                run_id=outer_run_id,
                fold_idx=-1,
                outer_fold_idx=outer_fold_idx,
                fold_train_start=outer_train_dates.min(),
                fold_train_end=outer_train_dates.max(),
                fold_train_days=outer_train_dates.nunique(),
                feature_config=retained.feature_config,
                feature_names=retained.feature_names,
                run_reducer=False,
                phase="refit",
                trial_idx=-1,          # outer eval row
                dry_run=False,
            )
            backtester = Backtester(refit_override, unit_opener,
                                    optimizer_run_id=optimizer_run_id,
                                    rate_sources=rate_sources)
            bt_result = backtester.run(
                outer_test_df,
                run_id=outer_run_id,
                fold_idx=-1,
                outer_fold_idx=outer_fold_idx,
                fold_test_start=outer_fold_meta["fold_test_start"],
                fold_test_end=outer_fold_meta["fold_test_end"],
                fold_test_days=outer_fold_meta["fold_test_days"],
                eval_type="outer_validation",      # execution_variant NULL
            )

            def write(conn):
                insert_train_log(conn, train_result["train_log_row"])
                backtester.write(conn, bt_result)
            s.write(write)

# Step: the closing step — as the exploitation phase's, without the consensus;
# done when this optimizer_run_id's final model train_log row exists (phase
# 'refit', outer_fold_idx -1, fold_idx -1, trial_idx 0). It produces:
#   - the final model on remaining_df: config refit_override, feature_config
#     and feature_names the retained ones, phase 'refit', run_id from
#     utils.generate_run_id();
#   - the candidate derivation of θ;
#   - the regime backtest, the Regime Holdout Report and its verdict;
#   - the champion comparison's verdict 'not_applicable';
#   - the model_candidates row (Model Candidate, above).
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
# Rates: the work's rate_snapshot — as in the exploitation phase
rate_sources = <the work's rate_snapshot rate_sources>

optimizer_run_id = work.work_id                  # Work, above
full_labeled_df  = <Data preparation, Work above>
unit_opener      = <the unit opener run_guest hands the body>
balancer  = ClassBalancer(config)
inner_cfg = config["class_balancer"]["inner_fold"]
fold_run_ids = []

# (full phase fold loop — one run per fold, run_reducer=False and max_folds=None)
for train, val, test, fold_meta in balancer.generate_folds(
    full_labeled_df,
    balance=config["class_balancer"]["apply_balance"],
    window_weeks=inner_cfg["window_weeks"],
    val_weeks=inner_cfg["val_weeks"],
    test_weeks=inner_cfg["test_weeks"],
    step_weeks=inner_cfg["step_weeks"],
    embargo_days=inner_cfg["embargo_days"],
    max_folds=None,          # override config cap — full phase uses all available folds
):
    fold_idx = fold_meta["fold_idx"]
    run_id   = f"{optimizer_run_id}_f{fold_idx}"   # structured ID — no collision risk
    fold_run_ids.append(run_id)
    # Step: this fold — done when its run_id is in train_log
    with step(done=lambda conn: in_train_log(conn, run_id)) as s:
        if not s.skipped:
            result = Trainer(config, optimizer_run_id=optimizer_run_id).run(
                train, val, test,
                run_id=run_id,
                fold_idx=fold_idx,
                outer_fold_idx=-1,
                fold_train_start=fold_meta["fold_train_start"],
                fold_train_end=fold_meta["fold_train_end"],
                fold_train_days=fold_meta["fold_train_days"],
                run_reducer=False,
                phase="full",
                trial_idx=0,
            )
            s.write(lambda conn: insert_train_log(conn, result["train_log_row"]))

# Step: the closing backtest — done when fold_run_ids[-1] is in experiment_log
with step(done=lambda conn: in_experiment_log(conn, fold_run_ids[-1]),
          read=lambda conn: [<train_log.auc_mean of run_id> for run_id in fold_run_ids]
          ) as s:
    if not s.skipped:
        fold_aucs = s.inputs
        auc_std = std(fold_aucs) if len(fold_aucs) > 1 else 0.0

        # Backtest on last fold's test split (baseline measurement)
        backtester = Backtester(config, unit_opener, optimizer_run_id=optimizer_run_id,
                                rate_sources=rate_sources)
        bt_result = backtester.run(
            test,                    # last fold's test split (loop variable after exit)
            run_id=fold_run_ids[-1],
            fold_idx=-1,
            outer_fold_idx=-1,
            eval_type=None,
        )

        def write(conn):
            conn.execute(
                "UPDATE train_log SET auc_std = ? WHERE run_id = ?",
                [auc_std, fold_run_ids[-1]]
            )
            backtester.write(conn, bt_result)
        s.write(write)
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
) -> dict | None:
    """
    Worker function. Runs one (trial, inner_fold) pair.
    Returns result dict — the train_log row is written by the coordinator.
    A worker opens NO database connection and takes no `db_path`: Trainer
    takes no connection, and `dry_run=True` skips only its artifact save.
    The round's workers run during the step's compute, while the coordinator
    holds no unit (Work's step helpers).
    """
    import sys

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
        window_weeks=inner_cfg["window_weeks"],
        val_weeks=inner_cfg["val_weeks"],
        test_weeks=inner_cfg["test_weeks"],
        step_weeks=inner_cfg["step_weeks"],
        embargo_days=inner_cfg["embargo_days"],
        max_folds=inner_cfg["max_folds"],
    ))

    if inner_fold_idx >= len(folds):
        return None

    train, val, test, fold_meta = folds[inner_fold_idx]

    trainer = Trainer(config_override, optimizer_run_id=optimizer_run_id)
    result  = trainer.run(
        train, val, test,
        run_id=run_id,
        fold_idx=inner_fold_idx,
        outer_fold_idx=outer_fold_idx,
        fold_train_start=fold_meta["fold_train_start"],
        fold_train_end=fold_meta["fold_train_end"],
        fold_train_days=fold_meta["fold_train_days"],
        feature_config=hyperparams,
        feature_names=selected_features,
        run_reducer=False,
        phase="exploitation",
        trial_idx=trial_idx,
        dry_run=True,
    )

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
    conn.execute("UPDATE train_log SET auc_std = ? WHERE run_id = ?",
                 [auc_std, fold_run_ids[-1]])     # in the final round's write unit
    fold_run_ids[-1] = last completed round's run_id for that trial

Pruned trials:
    auc_std remains NULL
    is_pruned = TRUE set on fold_run_ids[-1] (last submitted round)

Sequential mode (full): fold_run_ids[-1] = MAX fold_idx,
    guaranteed by generate_folds() ascending order.
    run_id format: {optimizer_run_id}_f{fold_idx} — no collision risk.
Selection phase: run_id formats {optimizer_run_id}_f{fold_idx}_g{k} (Stage A),
    {optimizer_run_id}_f{fold_idx}_r{j} (Stage B-1) and
    {optimizer_run_id}_f{fold_idx}_c{c} (Stage B-2); auc_std set on the chosen
    candidate's run_id at the largest fold_idx, or with no eligible candidate
    on the baseline run (_r0) at the largest fold_idx.
Nested/parallel mode: fold_run_ids[-1] = last Successive Halving round completed.
```

---

## optimizer_run_id Usage

- Generated at work creation via `utils.generate_run_id()` — the work's
  `work_id` — and reused by a resumed work
- Passed to all `Trainer` and `Backtester` instances within the same work
- Stored in `train_log.optimizer_run_id` and `experiment_log.optimizer_run_id`
- Enables grouping of all folds belonging to the same optimizer execution

---

## Interface

```python
class PipelineOptimizer:
    def __init__(
        self,
        config: dict,       # the work's config snapshot with its rate_snapshot
        unit_opener,        # run_guest's; each step's unit opened from it
    ): ...

    def run(self, work) -> pd.DataFrame:
        """
        Execute the opened work's phase-appropriate cycle (Work, above):
          - "selection":    Stage A and Stage B, save selected_features
                            returns empty DataFrame (no backtest)
          - "exploitation": nested validation with Successive Halving,
                            outer fold evaluation, champion outer evals,
                            consensus config, regime holdout
                            returns experiment_log rows (outer_validation,
                            champion_validation, + regime_holdout if holdout
                            enabled); also writes regime_holdout_dates,
                            with a regime_holdout row one regime_holdout_verdict
                            row, and the model_candidates row
          - "refit":        outer evals and a final model on the retained
                            configuration, regime holdout
                            returns experiment_log rows (outer_validation +
                            regime_holdout if holdout enabled); also writes
                            regime_holdout_dates, with a regime_holdout row one
                            regime_holdout_verdict row, and the model_candidates
                            row
          - "full":         one fold pass, no reducer, backtest — a manual
                            baseline and debugging phase
                            returns experiment_log rows
          - "refresh":      routine retraining entry point — the two-depth
                            design (calendar-triggered refit vs.
                            divergence-triggered re-optimization) is
                            specified in docs/ops/shadow_retraining.md's
                            "Retraining Cadence" section, not in this file —
                            this phase has no dedicated algorithm of its
                            own; the refresh run (Work, above) decides
                            "refit" or "exploitation" before opening its
                            work, and the work runs that phase (see
                            `optimizer.refresh` config block below).
        """
        ...

    def best_config(self) -> dict | None:
        """
        Query experiment_log for the config with highest winning_rate
        belonging to this optimizer_run_id, over its rows with
        execution_variant NULL and eval_type NULL or 'outer_validation';
        'champion_validation' and 'regime_holdout' rows are not read. None
        when no such row exists.

        NO AUTOMATIC CONSUMER: nothing in the pipeline calls this. The
        deployed configuration comes from the consensus path above, which
        ranks inner trials on AUC and never on winning_rate, and the deployed
        model is model_deployments' latest row (model_deploy.md). This is a query, and
        a caller reading it should weigh total_trades itself — the ranking
        applies no sample-size guard.

        UNCHANGED by execution_eval: this ranks MODEL configs and reads only
        rows with execution_variant IS NULL (the baseline pass of each outer
        fold, or the full phase's closing backtest).
        """
        ...

    def best_execution_variant(self) -> dict | None:
        """
        Rank execution variants for this optimizer_run_id: per
        execution_variant, the mean, over the outer folds whose execution
        variants step is done, of the fold's
        optimizer.execution_eval.rank_metric value — "sum_pnl_pct", the
        default: avg_pnl_pct × total_trades, 0 with no trade; or an
        experiment_log metric column.
        Returns None when execution_eval is disabled or produced no rows.

        The backtest fill-rate switches (backtest.entry_fill_rate_disabled /
        exit_fill_rate_disabled) are a property of a RUN, not a variant axis:
        execution_eval fixes them for the whole run and cannot vary them
        across variants. Mixing states within one run would rank on the
        SWITCH rather than on the parameters, the frictionless side scoring
        higher on avg_pnl_pct almost regardless of what else changed.
        This contract is UNCHANGED and stays state-agnostic — the caller
        runs it once per switch state and compares within a state, never
        across. Running with a switch on is NOT refused: this axis ranks
        execution PARAMETER SETS (entry_gap_type / entry_gap_value,
        exit_ladder_seed, cancel_after_seconds, entry_order_type), and those
        still separate with the participation model disabled — a different
        limit_price still gates different ticks, a different
        cancel_after_seconds still bounds a different consumption window, and
        the time axis and volume-weighted price path are preserved. The
        separation NARROWS rather than vanishes, and that narrowing is the
        use: with liquidity friction removed, the pure effect of the price
        and timing parameters is isolated, where today the two are entangled
        and it is not visible which drives the ranking. The two states
        selecting different variants is itself the finding — it says the best
        parameters depend on liquidity friction. Both switches are recorded
        per run in experiment_log (db_schema.md), so a ranking is never read
        outside the state it was produced in.

        Ranks on a return metric — sum_pnl_pct by default — NOT
        winning_rate, and this is the point rather
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

## CLI

```bash
run_guest pipeline_optimizer [--phase <phase>] [--trigger scheduled] [--supersede] [--config CONFIG]
run_guest pipeline_optimizer --resume-work <work_id>   # the sweep's command
```

The phase is `optimizer.phase`, or `--phase <phase>`, which overrides it for
the run; `--phase` with `--resume-work` is an error at start. The scheduled
'refresh' run is `run_guest pipeline_optimizer --phase refresh --trigger
scheduled`, launched by metadata_crawler.md's dispatcher at
`optimizer.refresh.schedule` (Work, above).

---

## Config Keys

```yaml
optimizer:
  phase: "exploitation"     # "selection" | "exploitation" | "refit" | "full" | "refresh"
  selected_features_path: "configs/selected_features.json"

  selection:
    group_alternatives:            # competing sets, each a list of feature group
      - [sr_levels, sr_channels]   # names; a group is named by its columns'
      - [hl_spread, roll_spread]   # producer (04_feature_extractor.md's
      - [obv, ad]                  # get_feature_groups()); within a set the
                                   # earlier-listed group wins ties
    search_space:                  # Stage B's grid; full dot-notation config
                                   # paths, applied through utils.apply_overrides()
      dimensionality_reducer.importance_averaging: ["uniform", "sample_weighted"]
      dimensionality_reducer.vote_threshold: [0.5, 0.67]
      dimensionality_reducer.importance_filter.bottom_pct: [0.1, 0.2, 0.3]
      dimensionality_reducer.correlation_filter.threshold: [0.90, 0.95]
    max_candidates: 24             # a grid holding more points raises
                                   # ConfigurationError at selection start

  refresh:
    schedule: {time: "23:40", days: weekdays}
                                   # the scheduled refresh run's launch
                                   # (metadata_crawler.md's dispatcher);
                                   # coordination.schedule stays the owners'
    calendar_cadence_days: 30      # monthly-level; "refit only" trigger (dispatches
                                   # to "refit" — the deployed model's retained
                                   # configuration, freshly fit on extended data)
    divergence_reoptimize: true    # when the divergence trigger fires
                                   # (shadow_retraining.md), dispatch to
                                   # "exploitation" instead of "refit" for this cycle
    divergence:                    # the weekly divergence evaluation
      window_weeks: 4              # (shadow_retraining.md); placeholders
      min_days: 10                 # pending measurement
      min_trades: 30

  promotion:                       # placeholders pending measurement
    noninferiority_margin: 0.005   # the champion comparison
    min_trade_days: 20             # the champion comparison
    probation_sessions: 20         # a deployed run_id's probation: its first
                                   # sessions after its latest model_deployments
                                   # row (shadow_retraining.md)

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

  successive_halving:
    enabled: true
    eta: 3                  # top 1/eta fraction survives each round

  parallelism:
    n_parallel_trials: 4    # ProcessPoolExecutor max_workers

  execution_eval:
    enabled: false          # paired execution-variant evaluation at outer folds
    rank_metric: "sum_pnl_pct"
                            # "sum_pnl_pct" (avg_pnl_pct × total_trades, 0
                            # with no trade) or an experiment_log metric column
    max_combinations: 8     # grid is evaluated exhaustively — a backtest per
                            # combination per outer fold, so this is the cost
                            # ceiling, not a sampling budget
    late_entry_theta_quantile: 0.10
                            # DEFAULT q for θ's derivation, needed even when
                            # this whole block is disabled. Its readers: the
                            # candidate derivation, the live derivation, and
                            # the grid, whose axis below overrides it per
                            # combination. The θ quantile set: this value and
                            # the grid's late_entry_theta_quantile values,
                            # whatever enabled
    late_entry_theta_min_entries: 30
                            # fewer on-time entries → θ null; a placeholder
                            # pending measurement
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
    # late_entry_theta_quantile is DERIVATION-ONLY, read by the candidate
    # derivation, the live derivation and the grid; the resolved scalar lives
    # in execution.late_entry_min_expected_return_pct, which is what both
    # engines read (execution_common.md).

  regime_holdout:
    enabled: true
    vol_holdout_percentile: 0.80   # top 20% most volatile dates excluded
    vol_window_days: 30
    vol_metric: "avg_intraday_range"
    gate:                          # utils.regime_gate(): blocks a session's
                                   # participation at live_mode_runner.md's
                                   # Step 7 alone; model_candidates' status at
                                   # writing and model_deploy also read it
      enabled: false               # false = report only
      min_trades: 30               # below it the gate checks are not evaluated
      insufficient_sample: "pass_with_warning"   # "pass_with_warning" | "fail"
      missing_result: "pass_with_warning"        # "pass_with_warning" | "fail";
                                                 # no verdict row for the run_id

  bootstrap:                       # utils.bootstrap_ci(); also the divergence
    n_resamples: 1000              # trigger (shadow_retraining.md) and the
    confidence: 0.95               # champion comparison
    seed: 0
```

`class_balancer` — its `outer_fold` and `inner_fold` blocks and
`early_stopping_val_fraction` among them — is declared in
06_class_balancer.md's Config Keys; `inner_fold.max_folds` caps the nested
validation inner loop only, the selection and full phases passing
`max_folds=None`.

---

## Constraints

- `train_log` rows are returned by Trainer and written by the coordinator in
  the step's write unit; `experiment_log` and `trade_log` rows by
  `Backtester.write()` there
- In nested validation: workers use `dry_run=True` (no artifact save); the
  coordinator writes every row
- `best_of_loop` set TRUE on the last round row of the winning inner trial per outer fold
- `auc_std` set on `fold_run_ids[-1]` via UPDATE — completed trials only;
  pruned trials retain NULL with is_pruned = TRUE; the selection phase sets it
  on the chosen candidate's run_id at the largest fold_idx (the baseline
  run's with no eligible candidate)
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
- run_ids for sequential full phase folds: structured as
  "{optimizer_run_id}_f{fold_idx}"; for selection phase runs:
  "{optimizer_run_id}_f{fold_idx}_g{k}", "{optimizer_run_id}_f{fold_idx}_r{j}"
  and "{optimizer_run_id}_f{fold_idx}_c{c}"; generate_run_id() not called
  inside fold loop
- Outer eval run_ids: structured format {optimizer_run_id}_o{outer_fold_idx}_eval;
  execution variants {optimizer_run_id}_o{outer_fold_idx}_eval_x{i};
  champion outer evals {optimizer_run_id}_o{outer_fold_idx}_champ;
  generate_run_id() not called inside the outer fold loop
- Phase transition is manual — no automatic switching between phases
- Selection phase does not run backtest — Stage A and Stage B score on
  `train_log.auc_mean`
- Full phase runs backtest after fold pass completes on last fold's test split
- Selection and full phases use max_folds=None in generate_folds() —
  overrides inner_fold.max_folds config cap (which is intended for exploitation inner loop only)
- Regime holdout: dates removed from remaining_df BEFORE outer fold generation;
  holdout_df used for both final model test_df (train_log AUC) and regime_holdout backtest
- final model test_df: holdout_df (primary, completely unseen) when holdout enabled and
  non-empty; final_val_split (fallback) when holdout disabled
- The holdout date set is written to `regime_holdout_dates` once per exploitation or refit work,
  computed over the work's pinned range, with each date's rolling_vol; the
  regime_holdout backtest passes no fold_test_* argument
- `fold_train_start` / `fold_train_end` / `fold_train_days`: from fold_meta at the inner
  fold call sites; from the temporal_split_simple() train split for the outer eval and
  the final model — never outer_fold_meta's train window
- `fold_test_days` passed from fold_meta wherever fold_test_start / fold_test_end are
- Regime holdout report and verdict are computed whenever a regime_holdout row is
  written; the gate blocks a session's participation at live_mode_runner.md's
  Step 7 alone, and model_candidates' status at writing and model_deploy's
  decisions also read the verdict (utils.regime_gate())
- `compute_vol_regime_holdout()` uses ohlcv_1min regular session bars only
- `compute_consensus_config()` aggregates outer fold best_configs;
  params → median → nearest grid value
- `temporal_split_simple()` used for early stopping val in outer eval and final model;
  no balancing applied; val_fraction=class_balancer.early_stopping_val_fraction and
  embargo_days=outer_cfg["embargo_days"] always passed explicitly
- `best_train`, `best_val`, `best_fold_test` patterns from non-nested exploitation removed;
  outer eval uses outer_train_split / outer_val_split / outer_test_df directly
- optimizer_run_id format: YYYYMMDD_HHMMSS (its generation: optimizer_run_id
  Usage, above)
- Calls neither `register()` nor `guest_unit()` — its units open through the
  unit opener and the step helpers `run_guest` hands it
- A step's DB writes are one transaction in its write unit, committed after
  `work_progress()`; Trainer and Backtester write nothing (guest_supervisor.md's
  I/O rule)
- Every execution of the work reads its fixed inputs, never the current config
  or the variant's current inputs
- `selected_features.json` is written through temp + rename (the selection
  phase's closing step) — an input of the 'exploitation' variant
  (guest_supervisor.md's Tool Registry)
- The wait for the input work holds no unit
- "random" strategy uses random_state for reproducible non-replacement sampling
