# Design Session Handoff

## Project
LightGBM-based US stock scalping entry point prediction pipeline.
Goal: maximize winning rate via preprocessor-model optimization.
Stack: Python 3.12, DuckDB, LightGBM, pandas, scipy

## Instructions
- This is a design session. Do not produce code files or implementation.
- Ignore task queue until explicitly asked to modify it.
- Respond in Korean, except when outputting .md spec file content.
- Do not re-discuss finalized decisions unless explicitly asked.

---

## Revised Files (replace originals in Project knowledge)

| File | Path in Project |
|---|---|
| `01_entry_detection.md` | docs/pipeline/ |
| `03_vectorizer.md` | docs/pipeline/ |
| `04_feature_extractor.md` | docs/pipeline/ |
| `05_labeler.md` | docs/pipeline/ |
| `06_class_balancer.md` | docs/pipeline/ |
| `07_lgbm_pipeline.md` | docs/pipeline/ |
| `08_dimensionality_reducer.md` | docs/pipeline/ |
| `09_backtest_engine.md` | docs/pipeline/ |
| `pipeline_optimizer.md` | docs/optimization/ |
| `run_preprocess.md` | docs/scripts/ |
| `run_train.md` | docs/scripts/ |
| `inferencer.md` | docs/inferencer/ |
| `utils.md` | docs/utils/ |
| `db_schema.md` | docs/data/ |

## Unchanged Files (originals still valid)
- `data_boundary.md`, `02_indicator_calculator.md`, `03_vectorizer.md` (base)
- `base_model.md`, `viz_connector.md`, `tools/*.md`, `architecture.md`
- `run_backtest.md`, `CLAUDE.md`
