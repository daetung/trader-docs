# Stock Auto-Scalping System — Architecture Overview

## Project Summary

- **Goal**: Predict short-term price direction at scalping entry points using a LightGBM-based supervised classification pipeline
- **Primary deliverable**: Preprocessor–model pipeline with optimal winning rate
- **Data**: US market, multi-month history, 1min OHLCV + 10tick per ticker
- **Model**: LightGBM (5 independent binary classifiers) → MLP comparison in later phase

For detailed design rules, refer to the per-module docs listed in the [Docs Index](#docs-index) below.

---

## System Pipeline

```
Raw JSON (1min / 10tick)
        │
        ▼
   DataLoader                JSON → DuckDB / per-ticker DataFrame
        │
        ▼
 EntryPointDetector           Candidate entry point selection (fixed logic)
        │                     Refine criteria if sideways class is over-represented
        │  (ticker, date, t)
        ▼
 IndicatorCalculator          All indicator calculations → time-series DataFrames
        │                     Input: bars [..., t-2, t-1] (fully closed bars only)
        ▼
 Vectorizer                   Time-series → fixed-length feature vectors
        │
        ▼
 FeatureExtractor             Combine vectorized indicators + meta + temporal
        │  feature matrix
        ▼
 Labeler                      5 independent binary labels per entry point
        │                     Reference: P_entry = t bar open (not a feature input)
        ▼
 ClassBalancer                Downsampling (sideways class priority)
        │  random split
        ├── train / validation / test
        ▼
 LightGBM Pipeline            5 independent binary classifiers
   ├── +5pp classifier
   ├── +3pp classifier
   ├── sideways classifier
   ├── -3pp classifier
   └── -5pp classifier
        │  AUC per class
        ▼
 DimensionalityReducer        Correlation filter → importance filter → SHAP filter
        │
        ▼
 BacktestEngine               Winning rate with slippage approximation
        │
        ▼
 PipelineOptimizer            Feature combination / hyperparameter search (Agent)
        │
        ▼
 experiment_log.csv           Cumulative results per pipeline configuration
```

---

## Docs Index

| Document | Description |
|---|---|
| `data/data_boundary.md` | **Read before any module implementation.** Shared boundary rules, P_entry definition, leakage prevention |
| `data/db_schema.md` | DuckDB schema and JSON ingestion logic |
| `pipeline/01_entry_detection.md` | EntryPointDetector logic and interface |
| `pipeline/02_indicator_calculator.md` | All indicator methods, input/output spec |
| `pipeline/03_vectorizer.md` | Time-series → vector transformation methods |
| `pipeline/04_feature_extractor.md` | Integration entrypoint and viz connector interface |
| `pipeline/05_labeler.md` | 5-class binary labeling logic |
| `pipeline/06_class_balancer.md` | Downsampling strategy |
| `pipeline/07_lgbm_pipeline.md` | LightGBM 5-classifier structure and evaluation |
| `pipeline/08_dimensionality_reducer.md` | Feature selection pipeline |
| `pipeline/09_backtest_engine.md` | Backtest logic, slippage model, winning rate |
| `optimization/pipeline_optimizer.md` | Agent workflow and experiment logging |
| `tools/migration_tool.md` | JSON → DuckDB one-time and incremental migration |
| `tools/detection_benchmark.md` | Entry detection threshold tuning + timing benchmark |
| `tools/metadata_crawler.md` | Daily metadata fetch + new data ingestion |

---

## Directory Structure

```
stock-scalping/
├── CLAUDE.md
├── docs/
│   ├── architecture.md            ← this file
│   ├── data/
│   │   ├── data_boundary.md
│   │   └── db_schema.md
│   ├── pipeline/
│   │   ├── 01_entry_detection.md
│   │   ├── 02_indicator_calculator.md
│   │   ├── 03_vectorizer.md
│   │   ├── 04_feature_extractor.md
│   │   ├── 05_labeler.md
│   │   ├── 06_class_balancer.md
│   │   ├── 07_lgbm_pipeline.md
│   │   ├── 08_dimensionality_reducer.md
│   │   └── 09_backtest_engine.md
│   └── optimization/
│       └── pipeline_optimizer.md
├── data/
│   ├── raw/                       # original JSON (1min / 10tick)
│   ├── processed/                 # feature matrix (.parquet)
│   └── labeled/                   # feature + label
├── src/
│   ├── data/
│   │   └── loader.py
│   ├── detection/
│   │   └── entry_detector.py
│   ├── preprocessing/
│   │   ├── base.py
│   │   ├── indicator_calculator.py
│   │   ├── vectorizer.py
│   │   └── feature_extractor.py
│   ├── labeling/
│   │   └── labeler.py
│   ├── balancing/
│   │   └── balancer.py
│   ├── feature_selection/
│   │   └── reducer.py
│   ├── models/
│   │   ├── base_model.py
│   │   └── lgbm_pipeline.py
│   ├── backtest/
│   │   └── engine.py
│   └── visualization/
│       └── viz_connector.py
├── scripts/
│   ├── run_preprocess.py
│   ├── run_train.py
│   └── run_backtest.py
├── tools/
│   ├── migrate_json_to_duckdb.py  # one-time + incremental JSON migration
│   ├── collect_daily.py           # daily metadata + ingestion (cron)
│   └── detection_benchmark.py    # threshold tuning + timing
├── tests/
├── configs/
│   └── pipeline_config.yaml
└── experiment_log.csv
```

---

## Open Items

| Item | When to resolve |
|---|---|
| Full feature indicator list | Before implementing IndicatorCalculator |
| Per-indicator vectorizer method mapping | During vectorizer design |
| EntryPointDetector detailed logic | Before implementing detection module |
| LightGBM hyperparameter search range | After initial model implementation |
| Visualization tool spec | Separate design phase |
| Stock meta data source | During data pipeline design |
