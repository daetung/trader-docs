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
        │                    precomputed_session_stats populated (REFERENCE_SESSION baselines)
        ▼
 EntryPointDetector           Candidate entry point selection (fixed logic)
        │                     All sessions processed; session_mode filter at training stage
        │  (ticker, date, t)
        │
        ├─────────────────────────────────────────┐
        │                                         │
        ▼                                         ▼
 FeatureExtractor                              Labeler
   (internally calls:                    5 independent binary labels per entry point
    IndicatorCalculator                  Exit at last_bar(date) or the execution.max_hold_bars limit;
    → Vectorizer                         after-market fallback for session-end halt only
    + MetaFeatures                       is_dead_position flag for overnight hold cases
    + TemporalFeatures)                  [labels, is_dead_position,
   [features...]                          dead_position_case, is_ambiguous]
        │                                         │
        └──────────────────┬──────────────────────┘
                           ▼
                         merge                   join on (ticker, date, hour)
                           │  feature matrix [ticker, date, hour, p_entry, features...,
                           │                  labels, is_dead_position, dead_position_case, is_ambiguous]
                           ▼
                    ClassBalancer                Downsampling via split(balance=config.apply_balance)
                           │  train / validation / test
                           ▼
                   PipelineOptimizer            Training endpoint
                     ├── Preprocessor           run once → full_labeled_df (return_data=True)
                     ├── Trainer                per-fold train
                     │     └── LightGBM Pipeline    5 independent binary classifiers
                     │     └── DimensionalityReducer  ImportanceProvider-based (model-agnostic)
                     └── Backtester             writes experiment_log
                           │
                           ▼
                   BacktestEngine               DB-direct price tracking, slippage via tick_10
                           │                   Session-close priority exit, dead position handling
                           ▼
                    experiment_log (DuckDB)      Written by Backtester only
                    train_log (DuckDB)           Written by Trainer per trial
```

---

## Endpoint Architecture

```
Training endpoint:   PipelineOptimizer
                         └── Preprocessor (once) → Trainer (per fold) → Backtester

Live inference endpoint: LiveModeRunner (execution orchestrator)
                         │
                         ├── CachingIndicatorCalculator  (injected into FeatureExtractor)
                         │       Layer 1: session_start_compute() — pivot_points, gap_pct
                         │       Layer 2: precalculate + on_bar_close() incremental update
                         │       fibonacci: monotonic deque O(1) per bar
                         │       sr_levels: get_for_entry() per-entry-point recomputation
                         │
                         ├── FeatureExtractor.extract()  (DI: CachingIndicatorCalculator)
                         │       session_stats from precomputed_session_stats (loaded at session start)
                         │
                         ├── Inferencer  (service object — called per entry point)
                         │       _prepare_bars() auto-trim (t bar or later removed defensively)
                         │       EntryPointDetector.detect()
                         │       FeatureExtractor.extract()
                         │       model.predict() → resolve_signal()
                         │       inference_log writes (DuckDB)
                         │
                         ├── Watchdog Polling Loop  (1s interval)
                         │       watchdog.get_candidates() → cumulative working set
                         │       PREFIX SCAN — list is most-recently-fired-first,
                         │         so cost is D+1 per cycle, not working-set size
                         │         N=3 speculative head slots + M=1 rotation cursor
                         │       Shared lane (CAP 4/cycle, non-reserved chart/min
                         │         slots — carryover and promotion share this
                         │         with the bar-close superset-K fetch, which
                         │         preempts both; carryover before promotion)
                         │       TradingAPI fetch (bars + ticks) → on_bar_close()
                         │       detect → infer → buy order
                         │
                         └── Position manager loop  (independent interval)
                                 monitor open positions → price check → sell order

Data source unique to live mode:
    precomputed_session_stats (DuckDB)  ← REFERENCE_SESSION baselines
                                           loaded once at session start via load_session_stats()
                                           delta smoothing applied in memory

Standalone scripts:
    run_preprocess.py  ← Preprocessor wrapper (CLI)
    run_train.py       ← Trainer wrapper (CLI)
    run_backtest.py    ← Backtester wrapper (CLI, sole experiment_log writer)
```

---

## Docs Index

| Document | Description |
|---|---|
| `data/data_boundary.md` | **Read before any module implementation.** Shared boundary rules, P_entry definition, REFERENCE_SESSION boundary, leakage prevention, Corporate Event Adjustment Boundary (raw/adjusted/scalar-corrected layering) |
| `data/db_schema.md` | DuckDB schema, JSON ingestion logic, inference_log, precomputed_session_stats. Live-session diagnostics: live_halt_episodes (observed halt intervals, ticker-scoped) and exit_trigger_agreement_daily (live-vs-backtest exit triggering, decomposed into tape and algorithm terms) |
| `pipeline/01_entry_detection.md` | EntryPointDetector logic, session_mode, volume_base_hour, A-G monotonicity classification (which conditions may be evaluated mid-bar) |
| `pipeline/02_indicator_calculator.md` | All indicator methods, REFERENCE_SESSION indicators, VWAP reset_mode, missing bar classification, precalculate_bars config |
| `pipeline/03_vectorizer.md` | Time-series → vector transformation methods, REFERENCE_SESSION mapping |
| `pipeline/04_feature_extractor.md` | Integration entrypoint, DI constructor, extract_batch strategy (A~E), REFERENCE_SESSION handling, parquet column structure |
| `pipeline/05_labeler.md` | 5-class binary labeling, session-close exit, time-limit exit, dead position |
| `pipeline/06_class_balancer.md` | Downsampling strategy, split() and generate_folds() |
| `pipeline/07_lgbm_pipeline.md` | LightGBM 5-classifier structure and evaluation |
| `pipeline/08_dimensionality_reducer.md` | ImportanceProvider pattern, model-agnostic reduction |
| `pipeline/09_backtest_engine.md` | DB-direct backtest, tick_10 slippage, dead position handling (has_data=TRUE), late-detection modelling (deterministic-hash drop/delay mirroring what live actually observes), gated fill simulation for the direction-free exits, counterfactual participation-rate switches |
| `models/base_model.md` | Abstract base class for model trainers |
| `optimization/pipeline_optimizer.md` | Training endpoint, nested validation, successive halving, regime holdout, execution_eval (paired execution-variant evaluation at outer folds) |
| `ops/api_contract_checklist.md` | Vendor-contract assumption inventory (trading API, yfinance, investing.com), graded by blast radius, tracked to verification before Pilot. Accumulates: a verified row keeps its measured value rather than being deleted |
| `ops/shadow_retraining.md` | Shadow mode design, execution-parameter fitting (fit_execution_params()), retraining cadence |
| `scripts/run_preprocess.md` | Preprocessor class, return_data, training pipeline only |
| `scripts/run_train.md` | Trainer class, session_mode filter, train_log |
| `scripts/run_backtest.md` | Backtester class, sole experiment_log writer |
| `api/trading_api.md` | TradingAPI — the single caller-facing layer over the vendored trading-API SDK. Result contract (a rejected order returns HTTP 200), symbol/exchange encoding, async boundary. No other module imports the SDK |
| `api/sdk_dependency.md` | The vendored SDK as an external artifact — adoption form, the three fork modifications and why none could be avoided, and the costs accepted with it |
| `utils/execution_common.md` | Execution-time decisions shared between BacktestEngine and LiveModeRunner — signal resolution, cooldown guard, entry/exit fill simulation over a time-valued anchor, position sizing, late-entry residual-edge gate |
| `utils/utils.md` | Shared utilities: bar sequence, signal resolution, slippage/fill simulation, label breach detection, encoding maps, trading calendar, REFERENCE_SESSION stats (populate + load), corporate-event adjustment (cum_split_ratio, adjust_bars_for_corporate_events, adjust_tick_derived_series_for_corporate_events, estimate_historical_meta, ticker rename stitching) |
| `live/inferencer.md` | Live inference service object — _prepare_bars, infer/infer_batch, inference_log |
| `live/live_mode_runner.md` | Live mode execution orchestrator — prefix-scan watchdog loop, recovery lane, position manager loop, session lifecycle, ticker-scoped 1-tick acquisition, real-path-parallel shadow fills |
| `live/caching_calculator.md` | CachingIndicatorCalculator — Layer 1/2 cache, fibonacci deque, sr_levels per-entry-point |
| `live/auxiliary_stream.md` | In-process host (loop A) for non-trading WebSocket subscriptions on a separate account — delayed-quote capture for feed-coverage measurement is its first consumer |
| `tools/init_db.md` | Schema bootstrap CLI — creates every table from db_schema.md's canonical DDL; `--verify` reports schema drift read-only. Sole DDL-issuing component; prerequisite for migration_tool |
| `tools/migration_tool.md` | JSON → DuckDB migration, trading_calendar/coverage init, precomputed_session_stats population |
| `tools/detection_benchmark.md` | Entry detection threshold tuning + timing benchmark, plus the diagnostic delayed-entry decay sweep |
| `tools/metadata_crawler.md` | Daily metadata fetch, new data ingestion, session stats daily update, evening detection-gap stage |
| `tools/health_report.md` | Findings gathering, log/Discord/email alert delivery, alert-level config |
| `visualization/viz_connector.md` | Abstract base class for visualization backends |

---

## Open Items

| Item | When to resolve |
|---|---|
| MLPImportanceProvider implementation | MLP phase |

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
│   ├── ops/
│   │   ├── api_contract_checklist.md
│   │   └── shadow_retraining.md
│   ├── scripts/
│   │   ├── run_preprocess.md
│   │   ├── run_train.md
│   │   └── run_backtest.md
│   ├── api/
│   │   ├── trading_api.md
│   │   └── sdk_dependency.md
│   ├── utils/
│   │   ├── utils.md
│   │   └── execution_common.md
│   ├── live/
│   │   ├── inferencer.md
│   │   ├── live_mode_runner.md
│   │   ├── caching_calculator.md
│   │   └── auxiliary_stream.md
│   ├── tools/
│   │   ├── init_db.md
│   │   ├── migration_tool.md
│   │   ├── detection_benchmark.md
│   │   ├── metadata_crawler.md
│   │   └── health_report.md
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
│   ├── live/
│   │   ├── inferencer.py
│   │   ├── live_mode_runner.py
│   │   ├── caching_calculator.py
│   │   └── auxiliary_stream.py
│   ├── api/
│   │   └── trading_api.py
│   ├── utils/
│   │   └── utils.py
│   └── visualization/
│       └── viz_connector.py
├── scripts/
│   ├── run_preprocess.py
│   ├── run_train.py
│   └── run_backtest.py
├── tools/
│   ├── init_db.py
│   ├── migrate_json_to_duckdb.py
│   ├── collect_daily.py
│   └── detection_benchmark.py
├── api_doc/                       # vendor API documentation (markdown).
│                                  #   External read-only reference, not a
│                                  #   spec file — the source for the
│                                  #   session-phase order-type rules, the
│                                  #   per-endpoint rate limits and the
│                                  #   message-code vocabulary. Cited by
│                                  #   docs/api/trading_api.md and
│                                  #   docs/api/sdk_dependency.md
├── tests/
├── vendor/
│   └── dbsec-open-api/            # vendored trading-API SDK (see
│                                  #   docs/api/sdk_dependency.md).
│                                  #   Installed editable BY THE USER, never
│                                  #   by an agent. Deliberately outside src/:
│                                  #   the docs/ ↔ src/ correspondence does
│                                  #   not extend to it, and it has no
│                                  #   counterpart spec
├── configs/
│   ├── pipeline_config.yaml
│   ├── sector_map.json
│   ├── day_of_week_map.json
│   ├── halt_reason_code_map.json
│   ├── cik_map.json
│   ├── xbrl_tag_map.json
│   └── secrets.yaml               # gitignored
└── experiment_log.csv             # legacy; authoritative store is DuckDB
```
