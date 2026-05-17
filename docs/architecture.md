# Stock Auto-Scalping System — Architecture Overview

## Project Summary

- **Goal**: Predict short-term price direction at scalping entry points using a LightGBM-based supervised classification pipeline
- **Primary deliverable**: Preprocessor–model pipeline with optimal winning rate
- **Data**: US market, multi-month history, 1min OHLCV + 10tick per ticker (all sessions)
- **Model**: LightGBM (5 independent binary classifiers) → MLP comparison in later phase

For detailed design rules, refer to the per-module docs listed in the [Docs Index](#docs-index) below.

---

## System Pipeline

```
Raw JSON (1min / 10tick, all sessions)
        │
        ▼
   DataLoader                JSON → DuckDB / per-ticker DataFrame
        │                    trading_calendar + ticker_data_coverage populated
        ▼
 EntryPointDetector           Candidate entry point selection (fixed logic)
        │                     All sessions processed; session_mode filter at training stage
        │  (ticker, date, t)
        ▼
 IndicatorCalculator          All indicator calculations → time-series DataFrames
        │                     Input: bars [..., t-2, t-1] (fully closed bars only)
        ▼
 Vectorizer                   Time-series → fixed-length feature vectors
        │
        ▼
 FeatureExtractor             Combine vectorized indicators + meta + temporal
        │                     extract_batch(): indicators computed once per ticker
        │  feature matrix [ticker, date, hour, p_entry, features..., labels...]
        ▼
 Labeler                      5 independent binary labels per entry point
        │                     Exit at 15:59 bar; after-market fallback for halt only
        │                     is_dead_position flag for overnight hold cases
        ▼
 ClassBalancer                Downsampling via split(balance=config.apply_balance)
        │  train / validation / test
        ▼
 PipelineOptimizer            Training endpoint
   ├── Preprocessor           in-memory preprocess per trial
   ├── Trainer (AUC loop)     train until auc_threshold or max_trials
   │     └── LightGBM Pipeline    5 independent binary classifiers
   │     └── DimensionalityReducer  ImportanceProvider-based (model-agnostic)
   └── Backtester             run after AUC loop converges
        │
        ▼
 BacktestEngine               DB-direct price tracking, slippage via tick_10
        │                     Session-close priority exit, dead position handling
        ▼
 experiment_log (DuckDB)      Written by Backtester only
 train_log (DuckDB)           Written by Trainer per trial
```

---

## Endpoint Architecture

```
Training endpoint:   PipelineOptimizer
                         └── Preprocessor → Trainer (loop) → Backtester

Live inference endpoint: Inferencer  (interface defined; implementation deferred)
                         └── Preprocessor(live_mode=True) → model.load() → resolve_signal()

Standalone scripts:
    run_preprocess.py  ← Preprocessor wrapper (CLI)
    run_train.py       ← Trainer wrapper (CLI)
    run_backtest.py    ← Backtester wrapper (CLI, sole experiment_log writer)
```

---

## Docs Index

| Document | Description |
|---|---|
| `data/data_boundary.md` | **Read before any module implementation.** Shared boundary rules, P_entry definition, leakage prevention |
| `data/db_schema.md` | DuckDB schema and JSON ingestion logic |
| `pipeline/01_entry_detection.md` | EntryPointDetector logic, session_mode, volume_base_hour |
| `pipeline/02_indicator_calculator.md` | All indicator methods, VWAP reset_mode, missing bar classification |
| `pipeline/03_vectorizer.md` | Time-series → vector transformation methods |
| `pipeline/04_feature_extractor.md` | Integration entrypoint, extract_batch strategy, parquet column structure |
| `pipeline/05_labeler.md` | 5-class binary labeling, session-close exit, dead position |
| `pipeline/06_class_balancer.md` | Downsampling strategy, split() as single entry point |
| `pipeline/07_lgbm_pipeline.md` | LightGBM 5-classifier structure and evaluation |
| `pipeline/08_dimensionality_reducer.md` | ImportanceProvider pattern, model-agnostic reduction |
| `pipeline/09_backtest_engine.md` | DB-direct backtest, tick_10 slippage, dead position handling |
| `models/base_model.md` | Abstract base class for model trainers |
| `optimization/pipeline_optimizer.md` | Training endpoint, AUC loop, experiment cycle |
| `scripts/run_preprocess.md` | Preprocessor class, return_data, live_mode |
| `scripts/run_train.md` | Trainer class, session_mode filter, train_log |
| `scripts/run_backtest.md` | Backtester class, sole experiment_log writer |
| `utils/utils.md` | Shared utilities: bar_sequence, signal, hour conversion, run_id, config, encoding maps |
| `inferencer/inferencer.md` | Live inference endpoint interface |
| `tools/migration_tool.md` | JSON → DuckDB migration, trading_calendar/coverage init |
| `tools/detection_benchmark.md` | Entry detection threshold tuning + timing benchmark |
| `tools/metadata_crawler.md` | Daily metadata fetch + new data ingestion |
| `visualization/viz_connector.md` | Abstract base class for visualization backends |

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
│   ├── models/
│   │   └── base_model.md
│   ├── optimization/
│   │   └── pipeline_optimizer.md
│   ├── scripts/
│   │   ├── run_preprocess.md
│   │   ├── run_train.md
│   │   └── run_backtest.md
│   ├── utils/
│   │   └── utils.md
│   ├── inferencer/
│   │   └── inferencer.md
│   ├── tools/
│   │   ├── migration_tool.md
│   │   ├── detection_benchmark.md
│   │   └── metadata_crawler.md
│   └── visualization/
│       └── viz_connector.md
├── models/                        # trained model artifacts (.lgb, _meta.json)
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
│   │   ├── lgbm_pipeline.py
│   │   └── factory.py
│   ├── optimization/
│   │   └── optimizer.py
│   ├── backtest/
│   │   └── engine.py
│   ├── inference/
│   │   └── inferencer.py
│   ├── utils/
│   │   └── utils.py
│   └── visualization/
│       └── viz_connector.py
├── scripts/
│   ├── run_preprocess.py
│   ├── run_train.py
│   └── run_backtest.py
├── tools/
│   ├── migrate_json_to_duckdb.py
│   ├── collect_daily.py
│   └── detection_benchmark.py
├── tests/
├── configs/
│   ├── pipeline_config.yaml
│   ├── sector_map.json
│   ├── day_of_week_map.json
│   └── secrets.yaml               # gitignored
└── experiment_log.csv             # legacy; authoritative store is DuckDB
```

---

## Open Items

| Item | When to resolve |
|---|---|
| Full feature indicator list | Before implementing IndicatorCalculator |
| Per-indicator vectorizer method mapping | During vectorizer design |
| LightGBM hyperparameter search range | After initial model implementation |
| Visualization tool spec | Separate design phase |
| MLPImportanceProvider implementation | MLP phase |
| Inferencer full implementation | After training pipeline stabilizes |
