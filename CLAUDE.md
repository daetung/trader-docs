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

For each module, read only the corresponding `docs/pipeline/0N_*.md` file.
Do not load other module docs unless checking an interface boundary.

---

## Data Boundary Rules (Critical — Do Not Skip)

```
Feature input:       bars t-1, t-2, ..., t-N  (fully closed OHLCV bars only)
P_entry:             t bar open  →  reference value only, NOT a feature
Forbidden as input:  t bar high / low / close / volume
Label search range:  t bar open → t+59 bar close  (max 60 min)
10-tick for features: ticks with timestamp < t bar open only
10-tick for backtest: ticks within t bar (slippage estimation only)
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

- [ ] loader.py — DuckDB schema creation and JSON ingestion script
- [ ] entry_detector.py — EntryPointDetector
- [ ] indicator_calculator.py — base structure + price/volume/tick indicators
- [ ] vectorizer.py — 6 transformation methods
- [ ] labeler.py — 5-class binary labeling + unit tests (start here)
- [ ] feature_extractor.py
- [ ] balancer.py
- [ ] reducer.py
- [ ] base_model.py
- [ ] lgbm_pipeline.py
- [ ] engine.py
- [ ] viz_connector.py
- [ ] run_preprocess.py
- [ ] run_train.py
- [ ] run_backtest.py
- [ ] migrate_json_to_duckdb.py
- [ ] collect_daily.py
- [ ] detection_benchmark.py

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
   Success: winning rate appended to experiment_log.csv

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
