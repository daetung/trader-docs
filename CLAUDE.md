# CLAUDE.md — Stock Auto-Scalping System

## One-line Summary
LightGBM-based US stock scalping entry point prediction pipeline.
Goal: find the preprocessor–model combination with optimal winning rate.

---

## Stack and Environment

```
Python       3.12+
DB           DuckDB  (data/market.duckdb — no server required)
ML           LightGBM, scikit-learn, shap
Data         pandas, polars, numpy, scipy
Config       PyYAML  (configs/pipeline_config.yaml)
Test         pytest
Lint         ruff, mypy
Venv         source .venv/bin/activate  ← run at session start
```

---

## Required Reading Before Any Implementation

1. `docs/architecture.md` — full pipeline overview and docs index
2. `docs/data/data_boundary.md` — **data leakage rules shared by all modules**
3. `docs/api/sdk_dependency.md` — before touching anything under `vendor/`,
   or any code that reaches the trading API

For each module, read only the corresponding `docs/pipeline/0N_*.md` file.
Do not load other module docs unless checking an interface boundary.

---

## Data Boundary Rules (Critical — Do Not Skip)

```
Feature input:         bars t-1, t-2, ..., t-N  (fully closed OHLCV bars only)
P_entry:               t bar open  →  reference value only, NOT a feature
Forbidden as input:    t bar high / low / close / volume
Label search range:    t bar open → 15:59 close or 60th valid bar close (whichever first)
10-tick for features:  ticks with timestamp < t bar open only
10-tick for backtest:  ticks from t bar onward
                       (entry slippage: entry_hour ~ entry_hour+100s;
                        exit tracking: full day incl. after-market)
REFERENCE_SESSION:     prior session baselines sourced from precomputed_session_stats
                       (offline DB table) — NOT from bars feature window
                       gap_percentile returns NaN for t="093000" or pre-market entries
Fundamentals:          fundamentals_quarterly must be read as-of a date via
                       filed_date, NEVER fiscal_period_end — filtering on the
                       latter leaks pre-disclosure information into features
                       computed for dates before the filing existed
                       (see data_boundary.md, db_schema.md)
```

---

## Coding Rules

- Type hints required on all functions and methods
- Write corresponding test file alongside each new module
- Max file size: ~1000 lines — split by class or responsibility if approaching limit
- All indicator calculations must live in `indicator_calculator.py` only
- All feature combinations and parameters must be in `pipeline_config.yaml` — no hardcoding
- Monetary features (market cap, price, 52w high/low): apply log transform at feature extraction time, store raw values in DB
- Categorical features (sector etc.): declare explicitly as LightGBM categorical columns
- Do not use PCA or other transformation-based dimensionality reduction — feature interpretability must be preserved

---

## Current Task Queue

> Updated between sessions. Check this before starting work.

- [x] loader.py — DuckDB schema creation and JSON ingestion script (48 tests, 97% coverage)
- [x] entry_detector.py — EntryPointDetector (77 tests, 98% coverage)
- [x] indicator_calculator.py — all indicator methods (130 tests, 95% coverage)
  ↳ SPEC UPDATED: REFERENCE_SESSION indicators (rvol, rel_dvol, gap_percentile, intraday_seasonality),
    fibonacci monotonic deque (O(N) single-pass), sr_levels pivot_hour_rN/sN columns,
    precalculate_bars config per indicator — re-review required
- [x] vectorizer.py — 6 transformation methods (38 tests, 96% coverage)
  ↳ SPEC UPDATED: REFERENCE_SESSION indicators added to mapping table (rvol, rel_dvol,
    intra_season → statistical_summary + window_comparison); gap_pct excluded from Vectorizer;
    pivot_hour_rN not included in sr_distance output — re-review required
- [x] labeler.py — 5-class binary labeling + unit tests (55 tests, 100% coverage)
  ↳ SPEC UPDATED: is_ambiguous (individual bar check), dead_position_case A/B/C — re-review required
- [x] feature_extractor.py — 57 tests, 96% coverage
  ↳ SPEC UPDATED: DI constructor (calculator parameter), session_stats parameter,
    extract_batch strategy A~E (fibonacci single-pass, sr_levels date당 1회, gap_pct inline),
    calculate_required_history() simplified, init ConfigurationError for window constraint,
    n_sessions independent of lookback_days — re-review required
- [x] balancer.py — 35 tests, 94% coverage
  ↳ SPEC UPDATED: generate_folds() added (rolling walk-forward), fold_meta yield,
    temporal split, ambiguous_sample_action "exclude" option, Case C exclusion —
    PARTIAL REWORK REQUIRED before proceeding
- [x] reducer.py — 52 tests, 100% coverage
  ↳ SPEC UPDATED: shap_subsample_n config added, shap_filter() subsampling logic — re-review required
- [ ] base_model.py — 29 tests, 100% coverage
- [x] lgbm_pipeline.py — 52 tests, 92% coverage
  ↳ SPEC UPDATED: sample_weight_col parameter added — re-review required
- [ ] engine.py — BacktestEngine (40 tests, 97% coverage)
  ↳ SPEC UPDATED: dead position trading_calendar query changed to has_data=TRUE
    (previously is_trading_day=TRUE — Issue V fix) — re-review required
- [x] viz_connector.py — Abstract base class + factory (14 tests, 100% coverage)
- [x] run_preprocess.py — full pipeline orchestrator (12 tests, 93% coverage)
  ↳ SPEC UPDATED: session_stats loading (Step 1) from precomputed_session_stats,
    session_stats passed to extract_batch() (Step 6) — re-review required
- [ ] run_train.py — 28 tests, 99% coverage, APPROVED
  ↳ SPEC UPDATED: fold_idx/fold_train_end/phase params, run_reducer vs --reduce distinction,
    sample weight handling (trainer.* config) — implement with updated spec
- [ ] run_backtest.py — 32 tests, 99% coverage, APPROVED
  ↳ SPEC UPDATED: suppressed_count, fold_idx/fold_test_start/fold_test_end in experiment_log —
    implement with updated spec
- [x] migrate_json_to_duckdb.py — 12 tests, 96% coverage
  ↳ SPEC UPDATED: Step 6 added — populate_precomputed_session_stats() after calendar/coverage init
    — re-review required
- [x] collect_daily.py — 50 tests, 99% coverage
  ↳ SPEC UPDATED: Session Stats Update step added (compute next-day baselines via
    populate_precomputed_session_stats() after ingestion) — re-review required
- [x] detection_benchmark.py — 21 tests, 100% pass
- [ ] utils.py — SPEC UPDATED: populate_precomputed_session_stats(), load_session_stats(),
    hhmmss_to_minutes() added — implement new functions
- [ ] optimizer.py — PipelineOptimizer (feature combination search, selection/exploitation/full phases,
    rolling fold loop, frequency voting)

- [ ] inferencer.py — Live inference service object
    (previously deferred; full spec now defined — implement with updated spec)
- [ ] live_mode_runner.py — Live mode execution orchestrator
    (new module: watchdog loop, position manager loop, session lifecycle, CachingCalculator management)
- [ ] caching_calculator.py — CachingIndicatorCalculator for live mode
    (new module: Layer 1/2 cache, fibonacci monotonic deque, sr_levels per-entry-point)

---

## Re-review Policy for SPEC UPDATED modules

When a module is marked "SPEC UPDATED":
1. Read the updated spec doc before any implementation work
2. Identify delta between current implementation and new spec
3. Add/modify only what changed — do not rewrite the entire module
4. Re-delegate to @tdd-guide for affected test cases only
5. Re-delegate to @python-reviewer after @tdd-guide passes
6. Remove the "SPEC UPDATED" annotation once @python-reviewer approves

---

## Sub-agents

Available agents in `.claude/agents/`:

| Agent | When to invoke | Expected output |
|---|---|---|
| `@tdd-guide` | After any module implementation is complete | Pass/fail report + coverage % |
| `@python-reviewer` | After @tdd-guide reports all tests passing | Spec compliance + lint report |
| `@e2e-runner` | To execute scripts and return stdout/stderr | Raw execution output |

Orchestrator rules:
- Always delegate testing to @tdd-guide — never run pytest yourself
- Do not invoke @python-reviewer if @tdd-guide reports any failures
- If @tdd-guide reports failure: fix the code, then re-delegate to @tdd-guide
- Do not mark a task complete in the task queue until @python-reviewer verdict is APPROVED
- Standard cycle for every module: **implement → @tdd-guide → @python-reviewer → mark done**

---

## Prompting Templates

### Single module implementation (standard)
```
Start an implementation session.

Target module: {module_name}
Spec: docs/pipeline/{0N_module_name}.md

Steps:
1. Read docs/data/data_boundary.md
2. Read docs/pipeline/{0N_module_name}.md
3. Implement src/{path}/{module_name}.py
4. Delegate to @tdd-guide when done
5. If @tdd-guide passes, delegate to @python-reviewer
6. Report final result

Do not proceed past step 3 without confirming
the implementation matches the spec.
```

### Spec-updated module re-review
```
Re-review session for SPEC UPDATED module.

Target module: {module_name}
Spec: docs/pipeline/{0N_module_name}.md

Steps:
1. Read the updated spec doc
2. Diff against current implementation in src/{path}/{module_name}.py
3. Apply only the changed portions
4. Delegate to @tdd-guide for affected test cases
5. If @tdd-guide passes, delegate to @python-reviewer
6. Report final result and remove SPEC UPDATED annotation from CLAUDE.md
```

### Short form (when task queue is up to date)
```
Start implementation session.
Pick the first unchecked item from the task queue.
Follow the standard cycle: implement → @tdd-guide → @python-reviewer.
Report when approved.
```

### Optimization cycle
```
Run one optimization cycle.

Steps:
1. Run: python scripts/run_preprocess.py --config configs/pipeline_config.yaml
   Success: feature matrix saved to data/processed/

2. Run: python scripts/run_train.py
   Success: AUC per class printed, model saved to models/

3. Delegate to @e2e-runner:
   "Run DimensionalityReducer.run_all() on the trained model.
    Report: input feature count, output feature count, removed features."

4. Run: python scripts/run_train.py --features selected_features.json
   Success: reduced AUC within 0.005 of baseline AUC

5. Run: python scripts/run_backtest.py
   Success: winning rate appended to experiment_log

Final report: baseline AUC / reduced AUC / winning rate / features used
```

### Design session (no files written)
```
This is a design session. Do not create or modify any files.
Do not write implementation code.

[paste relevant doc section or question]

Output: prose explanation + recommendation only.
```

---

## Session Start Checklist

1. `conda activate venv`
2. `git status`
3. Check task queue above
4. Read `docs/data/data_boundary.md` and the relevant pipeline doc before implementing

---

## Hard Rules

- Never use t bar H/L/C/V as feature input
- Never put P_entry (t bar open) into the feature matrix
- Never hardcode feature combinations or model parameters in source code
- Never push directly to main branch
- Never define indicator calculation logic outside `indicator_calculator.py`
- Never invoke @python-reviewer before @tdd-guide passes
- Never mark a task complete without @python-reviewer APPROVED verdict
- **Never install a package or otherwise modify the environment.** Every
  installation — the vendored trading-API SDK and third-party libraries
  alike — is performed by the user. If something is missing, say so and
  stop; do not `pip install`, do not edit the venv, do not add a
  dependency file entry and install from it. SDK-specific detail lives in
  `docs/api/sdk_dependency.md`

---

## Git Rules
- Commit md changes separately from code changes
- Commit message format:
    docs: {module} — {change summary}
    feat: implement {module}
    fix:  {module} — {issue}
    test: add tests for {module}
- After any md file update: git add docs/ && git commit -m "docs: {change summary}"
- After module implementation approved by @python-reviewer: git commit -m "feat: {module}"
- Stage only changed files (no git add -A blindly)
- Never push to remote without explicit instruction
