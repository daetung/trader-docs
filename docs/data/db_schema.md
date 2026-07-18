# Database Schema — DuckDB

## Technology Choice

**DuckDB** — single `.duckdb` file, no server required.

Rationale:
- Columnar storage (Parquet-style) → fast reads on specific tickers and date ranges
- Direct pandas/polars integration
- No infrastructure overhead for local + WSL development
- Handles multi-million row analytical queries efficiently

Database file location: `data/market.duckdb`

---

## Raw Input Format

Source JSON structure (per file: one ticker, one date, one resolution):

```json
[
  {"Hour": "093000", "Date": "20250714", "Prpr": "120.03", "Oprc": "120.03",
   "Hprc": "120.03", "Lprc": "120.03", "CntgVol": "5"},
  ...
]
```

Field mapping (applies to both 1min and 10tick):

| JSON field | Description | Target column |
|---|---|---|
| `Date` | YYYYMMDD | `date` |
| `Hour` | HHMMSS (bar/tick timestamp) | `hour` |
| `Oprc` | Open price | `open` |
| `Hprc` | High price | `high` |
| `Lprc` | Low price | `low` |
| `Prpr` | Close / last traded price | `close` |
| `CntgVol` | Volume | `volume` |

---

## File Naming and Directory Convention

```
{data_root}/
├── {DATE}/
│   ├── {TICKER}_{DATE}_min.json     ← 1-minute OHLCV
│   └── {TICKER}_{DATE}_tick.json    ← 10-tick OHLCV
└── {DATE}.zip                        ← flat zip (no sub-directory)
    ├── {TICKER}_{DATE}_min.json
    └── {TICKER}_{DATE}_tick.json
```

- `DATE` format: `YYYYMMDD`
- `RESOLUTION` values: `min` | `tick`
- `data_root` is configurable (CLI argument or config file)
- Both directory and zip formats are supported
- Duplicate detection: if `(ticker, date, resolution)` already exists in DuckDB, the file is skipped
- **No session-of-day filtering at ingestion** — all bars across pre-market, regular,
  and after-market sessions (`hour` 040000~200000) are stored; overnight (outside
  this range) is excluded — see `migrate_json_to_duckdb.py`'s time-range filter

---

## Table Definitions

```sql
-- 1-minute OHLCV bars (all sessions: pre-market, regular, after-market)
CREATE TABLE ohlcv_1min (
    ticker   VARCHAR      NOT NULL,
    date     VARCHAR      NOT NULL,  -- 'YYYYMMDD'
    hour     VARCHAR      NOT NULL,  -- 'HHMMSS', bar open time
    open     DOUBLE       NOT NULL,
    high     DOUBLE       NOT NULL,
    low      DOUBLE       NOT NULL,
    close    DOUBLE       NOT NULL,
    volume   BIGINT       NOT NULL,
    PRIMARY KEY (ticker, date, hour)
);

-- 10-tick data (all sessions: pre-market, regular, after-market)
-- hour: HHMMSS representing the LAST tick timestamp of each 10-tick bundle (second precision)
-- seq_id: sequential counter assigned per (ticker, date, hour) at ingestion time
CREATE TABLE tick_10 (
    ticker   VARCHAR      NOT NULL,
    date     VARCHAR      NOT NULL,
    hour     VARCHAR      NOT NULL,  -- 'HHMMSS', last tick of bundle (second precision)
    seq_id   INTEGER      NOT NULL,  -- sequential index within same (ticker, date, hour)
    open     DOUBLE       NOT NULL,  -- Oprc (first tick of bundle)
    high     DOUBLE       NOT NULL,  -- Hprc
    low      DOUBLE       NOT NULL,  -- Lprc
    close    DOUBLE       NOT NULL,  -- Prpr (last tick of bundle)
    volume   BIGINT       NOT NULL,  -- CntgVol
    PRIMARY KEY (ticker, date, hour, seq_id)
);

-- Bar-level cache of tick-derived indicator primitives (EAV — new `indicator`
-- values register without a schema change; mirrors indicator_cache's shape
-- but is a PERSISTENT, growing historical store like ohlcv_1min /
-- precomputed_session_stats, not a session-purged transient cache).
-- Populated offline (collect_daily.py, via utils.compute_tick_bar_aggregates(),
-- which calls the SAME IndicatorCalculator functions used everywhere else —
-- not a reimplementation, so cached and on-the-fly values cannot diverge).
-- Values are RAW/native-scale for that bar's own date. Correction to a
-- specific entry's basis happens downstream via
-- utils.adjust_tick_derived_series_for_corporate_events() — identically
-- whether the value came from this table or the Tier-2 on-the-fly fallback
-- in utils.load_tick_bar_aggregates_with_history().
-- A (ticker, date, hour) with zero ticks (halt / no-trade slot) has no row
-- for any indicator — callers reindex against the full expected bar index
-- (same one used for bars) so gaps surface as explicit NaN, never a silently
-- shortened window (see 02_indicator_calculator.md Tick-Derived Indicator
-- Scale Sensitivity).
CREATE TABLE tick_bar_aggregates (
    ticker      VARCHAR NOT NULL,
    date        VARCHAR NOT NULL,
    hour        VARCHAR NOT NULL,   -- bar open time, 'HHMMSS'
    indicator   VARCHAR NOT NULL,   -- 'tpm' | 'avg_vol_per_tick' | 'buyer_initiated_ratio' |
                                    -- 'vol_weighted_buy_ratio' | 'avg_delta_per_tick' |
                                    -- 'tick_realized_vol' | 'path_efficiency' |
                                    -- 'vol_concentration' | 'tick_burstiness'
    value       DOUBLE,             -- NULL if indicator undefined for this bar
                                    -- (e.g. <2 tick bundles for a delta/interval-based
                                    -- indicator — see 02_indicator_calculator.md)
    PRIMARY KEY (ticker, date, hour, indicator)
);

-- Stock metadata (crawled daily; append-only per date — see metadata_crawler.md)
-- PRIMARY KEY includes date: a row exists only for dates actually crawled.
-- Dates before schema deployment, or individual fields that failed to crawl on
-- a given date, have no row / no value here — callers fall back to
-- utils.estimate_historical_meta(), applied per FIELD (not per row):
--   shares_outstanding, market_cap, price_52h, price_52l — when the real
--     crawled value is unavailable, derived from ohlcv_1min + corporate_events,
--     or from fundamentals_quarterly (see below) where that source covers
--     the metric
--   sector — has no derivation path; callers always use the most recent
--     available snapshot regardless of entry date (accepted approximation)
-- FeatureExtractor's MetaFeatures and EntryPointDetector's filter E (condition E)
-- share this same fallback utility rather than each implementing their own.
CREATE TABLE stock_meta (
    ticker             VARCHAR   NOT NULL,
    date               VARCHAR   NOT NULL,   -- 'YYYYMMDD', date actually crawled
    sector             VARCHAR,
    market_cap         DOUBLE,
    shares_outstanding BIGINT,
    price_52h          DOUBLE,
    price_52l          DOUBLE,
    avg_volume         DOUBLE,
    updated_at         VARCHAR,
    PRIMARY KEY (ticker, date)
);

-- CIK (SEC identifier) mapping per ticker, upserted daily from SEC's
-- company_tickers.json. Backs both ticker-rename auto-detection (a rename
-- keeps the same CIK; a merger/spin-off does not) and fundamentals_quarterly's
-- lookup key. first_seen/last_seen let a (cik, ticker) pairing be recognized
-- as historical vs. current without deleting superseded rows.
-- rename_pending/quarantine_reason back LiveModeRunner's is_tradable() gate
-- (see execution_common.md): is_tradable() returns tradable only when
-- BOTH are clear (rename_pending=FALSE AND quarantine_reason IS NULL).
-- N-2 restructure: originally a single status/suspend_reason pair, which
-- meant the P-8 quarantine write (below) and the rename-detection write
-- (metadata_crawler.md) could clobber each other's state for a ticker
-- flagged by both at once (rename-ambiguous immediately after a
-- corporate event is exactly the case where this could co-occur).
-- Independent columns make that structurally impossible rather than
-- guarding against it at each of the four write sites — see
-- metadata_crawler.md's detect_rename_candidates(),
-- self_correct_rename_effective_dates(), check_corporate_event_anomaly(),
-- self_correct_quarantine().
-- quarantine_reason is VARCHAR (not BOOLEAN) to leave room for future
-- non-'corporate_event_anomaly' reasons without another schema change —
-- rename_pending stays BOOLEAN since the rename case has exactly one
-- reason by construction (there is no second way to become rename-pending).
CREATE TABLE ticker_cik_map (
    cik               VARCHAR NOT NULL,
    ticker            VARCHAR NOT NULL,
    first_seen_date   VARCHAR NOT NULL,   -- 'YYYYMMDD', first observed
    last_seen_date    VARCHAR NOT NULL,   -- 'YYYYMMDD', most recent observed (upsert-refreshed)
    rename_pending    BOOLEAN NOT NULL DEFAULT FALSE,
    quarantine_reason VARCHAR,            -- NULL | 'corporate_event_anomaly' today;
                                          -- reserved for future reasons (see above)
    PRIMARY KEY (cik, ticker)
);

-- Quarterly/annual fundamentals from SEC EDGAR XBRL companyfacts (EAV — new
-- `metric` values register without a schema change). Sourced via
-- ticker_cik_map. Point-in-time correctness depends on `filed_date`, not
-- `fiscal_period_end` — see data_boundary.md.
CREATE TABLE fundamentals_quarterly (
    ticker            VARCHAR NOT NULL,
    cik               VARCHAR NOT NULL,
    fiscal_period_end VARCHAR NOT NULL,   -- 'YYYYMMDD', reporting period end
    filed_date        VARCHAR NOT NULL,   -- 'YYYYMMDD', filing date — as-of queries
                                          -- must filter on this, not fiscal_period_end,
                                          -- to avoid using pre-disclosure information
    metric            VARCHAR NOT NULL,   -- 'shares_outstanding' | 'net_income' |
                                          -- 'stockholders_equity' | 'revenue' |
                                          -- 'eps_diluted' | 'total_assets' |
                                          -- 'total_liabilities' | 'cash' |
                                          -- 'operating_income' | 'operating_cash_flow'
    period_months     INTEGER,            -- NULL = instant (shares_outstanding,
                                          -- stockholders_equity, total_assets, etc.);
                                          -- 3 = quarterly duration; 12 = annual duration
                                          -- (Q4-only duration values are derived at
                                          -- read time as FY - (Q1+Q2+Q3), never stored)
    value             DOUBLE  NOT NULL,
    PRIMARY KEY (ticker, fiscal_period_end, metric, filed_date)
);

-- Ticker symbol rename history (plain renames only — mergers/spin-offs excluded,
-- since those are treated as distinct securities, not continuations).
-- effective_date provenance: detected at premarket from a ticker_cik_map
-- mapping change (best-effort estimate, since that day's ohlcv_1min does not
-- exist yet), then self-corrected the same evening once ohlcv_1min for the
-- new ticker is actually available (see metadata_crawler.md). Ambiguous or
-- CIK-unmatched cases (e.g. tickers not registered with the SEC) are not
-- auto-registered — logged to tools/rename_candidates.log for manual review
-- via the same CLI used for manual entry (see metadata_crawler.md
-- --register-rename / --list-rename-candidates).
-- Used by utils.get_ticker_history() / load_ohlcv_with_history() to stitch
-- pre-rename bars into a continuous series addressed under the current symbol.
CREATE TABLE ticker_history (
    current_ticker  VARCHAR NOT NULL,   -- current (post-rename) symbol
    previous_ticker VARCHAR NOT NULL,   -- prior symbol
    effective_date  VARCHAR NOT NULL,   -- 'YYYYMMDD', first date current_ticker is used
    rename_type     VARCHAR NOT NULL DEFAULT 'rename',
    PRIMARY KEY (current_ticker, effective_date)
);

-- Trading halts (crawled from NYSE, refreshed daily)
CREATE TABLE trading_halts (
    ticker        VARCHAR   NOT NULL,
    date          VARCHAR   NOT NULL,   -- 'YYYYMMDD'
    halt_start    VARCHAR   NOT NULL,   -- 'HHMMSS'
    halt_end      VARCHAR,              -- NULL if halt not yet resumed at crawl time
    reason_code   VARCHAR,              -- NYSE halt reason code (e.g. "T1", "T6", "LUDP")
    PRIMARY KEY (ticker, date, halt_start)
);

-- US market holidays (NYSE calendar)
CREATE TABLE us_holidays (
    date          VARCHAR   PRIMARY KEY,  -- 'YYYYMMDD'
    holiday_name  VARCHAR
);

-- Trading calendar — all dates with trading day status and data availability
CREATE TABLE trading_calendar (
    date            VARCHAR   PRIMARY KEY,  -- 'YYYYMMDD'
    is_trading_day  BOOLEAN   NOT NULL,
    is_holiday      BOOLEAN   NOT NULL,
    has_data        BOOLEAN   NOT NULL,
    updated_at      VARCHAR
);

-- Ticker-level data coverage per date
CREATE TABLE ticker_data_coverage (
    ticker      VARCHAR   NOT NULL,
    date        VARCHAR   NOT NULL,
    has_1min    BOOLEAN   NOT NULL,
    has_tick    BOOLEAN   NOT NULL,
    PRIMARY KEY (ticker, date)
);

-- Completion markers for daily batch stages (metadata_crawler.md's Dual
-- Schedule + LiveModeRunner's own session-end marker). Backs two things:
-- (1) LiveModeRunner's Health Gate 1/2 (live_mode_runner.md) reads this to
--     detect a stale/failed/still-running upstream batch before trading
--     starts; (2) DuckDB access-ownership sequencing (P-5): the premarket
--     stage's completion marker gates LiveModeRunner's session start, and
--     LiveModeRunner's own 'live_session_end' row gates the evening stage's
--     start — see live_mode_runner.md and metadata_crawler.md.
CREATE TABLE batch_runs (
    stage        VARCHAR   NOT NULL,  -- 'premarket_rename' | 'premarket_corporate_events' |
                                      -- 'evening_ingestion' | 'evening_tick_bar_aggregates' |
                                      -- 'evening_session_stats' | 'live_session_start' |
                                      -- 'live_session_end'
    date         VARCHAR   NOT NULL,  -- 'YYYYMMDD' — the trading day this batch targets
    status       VARCHAR   NOT NULL,  -- 'running' | 'success' | 'failed'
    started_at   TIMESTAMP NOT NULL,
    finished_at  TIMESTAMP,           -- NULL while status='running'
    PRIMARY KEY (stage, date)
);
-- 'live_session_start' (R-2): written 'running' at Session Lifecycle Step
-- 1c, flipped to 'success' at clean Session Shutdown alongside
-- 'live_session_end'. A today row still 'running' with no
-- 'live_session_end' row is the crash signature warm-restart detection
-- keys on — see live_mode_runner.md.

-- Fitted execution-simulation parameters (buy_rate, sell_rate_tp, sell_rate_sl,
-- cancel_after_seconds), refit periodically from pilot-stage predicted-vs-actual
-- fill comparisons (see trade_log.predicted_* columns and
-- shadow_retraining.md's fit_execution_params()). The `execution:` config
-- block values (execution_common.md) are seed defaults only, used until the
-- first row lands here for a given param_name; BacktestEngine and
-- LiveModeRunner both read the latest row per param_name at session start
-- in preference to the config seed — this table is never written by hand.
CREATE TABLE execution_params (
    param_name   VARCHAR   NOT NULL,  -- 'buy_rate' | 'sell_rate_tp' | 'sell_rate_sl' |
                                      -- 'cancel_after_seconds'
    value        DOUBLE    NOT NULL,
    fitted_at    TIMESTAMP NOT NULL,
    sample_size  INTEGER   NOT NULL,  -- cumulative pilot-stage trade count used —
                                      -- see shadow_retraining.md for why this
                                      -- accumulates across weeks rather than
                                      -- resetting each calendar-week cycle
    week_start   VARCHAR   NOT NULL,  -- 'YYYYMMDD' — calendar-week this fit run
                                      -- was triggered on (the run cadence, not
                                      -- the data window — see above)
    PRIMARY KEY (param_name, fitted_at)
);

-- Corporate events (splits, reverse splits, dividends)
-- Populated by metadata_crawler via yfinance splits/dividends history.
-- US stock splits and ex-dividend adjustments always take effect before market
-- open — no intra-session events occur.
-- Used by populate_precomputed_session_stats() to adjust prior-session baseline
-- values in place (volume/price scale for splits; gap_pct prev_close for both)
-- so that baselines remain consistent with current share/price scale — see
-- utils.md populate_precomputed_session_stats() for the adjustment formulas.
-- Also used by adjust_bars_for_corporate_events() (utils.md) for split-scale
-- correction of raw bars feeding CONTINUOUS/cumulative indicators, by
-- adjust_tick_derived_series_for_corporate_events() (utils.md) for the
-- entry-date-direct correction of tick-derived indicator series, and by
-- gap_percentile() / dead-position pnl (labeler.md, backtest_engine.md) for
-- scalar dividend/split adjustment across overnight boundaries.
CREATE TABLE corporate_events (
    ticker      VARCHAR NOT NULL,
    event_date  VARCHAR NOT NULL,   -- 'YYYYMMDD' (effective date, before market open)
    event_type  VARCHAR NOT NULL,   -- 'split' | 'reverse_split' | 'dividend'
    value       DOUBLE  NOT NULL,   -- split/reverse_split: ratio (>1.0 or <1.0)
                                    -- dividend: per-share cash amount (USD, >0)
    source      VARCHAR NOT NULL DEFAULT 'yfinance',  -- 'yfinance' | 'investing'
                                    -- which vendor supplied the value now
                                    -- stored. NOT part of the primary key —
                                    -- see the invariant below.
    PRIMARY KEY (ticker, event_date, event_type)
);
-- item N INVARIANT: exactly ONE row per (ticker, event_date, event_type),
-- always. This is load-bearing, not incidental: cum_split_ratio() (utils.md)
-- computes the PRODUCT of every matching row, so a second row for the same
-- event — e.g. the same 2:1 split reported by both vendors — would yield
-- 2.0 * 2.0 = 4.0 and silently mis-scale every adjusted price and volume
-- that depends on it (adjust_bars_for_corporate_events(),
-- adjust_tick_derived_series_for_corporate_events(),
-- populate_precomputed_session_stats() section D, gap_percentile(),
-- Labeler Case A, BacktestEngine dead-position). No exception raised, no
-- log line — just wrong numbers. Any future vendor added here must respect
-- this invariant.
--
-- Vendor disagreement is therefore resolved AT WRITE TIME, by the shared
-- upsert_corporate_event() helper (metadata_crawler.md), never by keeping
-- both rows here:
--   no existing row               -> INSERT
--   existing row, values agree    -> no-op (agreement needs no second row)
--   existing row, values disagree -> corporate_events keeps the
--                                    investing.com value (the confirmed
--                                    tie-break: date-scoped same-day query,
--                                    treated as fresher), AND both values
--                                    are recorded in corporate_event_conflicts
--                                    below so the disagreement is fully
--                                    inspectable without endangering the
--                                    one-row invariant.
-- Consequence: no reader needs tie-break logic, and none was added.

-- Vendor disagreements for the same event (item N). Diagnostic only — no
-- runtime consumer reads this table; it exists so that "the two vendors
-- disagreed about this split/dividend" is a recoverable fact rather than
-- something a blind overwrite silently discarded. Surfaced by
-- health_report.md.
CREATE TABLE corporate_event_conflicts (
    ticker        VARCHAR   NOT NULL,
    event_date    VARCHAR   NOT NULL,   -- 'YYYYMMDD'
    event_type    VARCHAR   NOT NULL,   -- 'split' | 'reverse_split' | 'dividend'
    kept_source   VARCHAR   NOT NULL,   -- vendor whose value is in corporate_events
    kept_value    DOUBLE    NOT NULL,
    other_source  VARCHAR   NOT NULL,   -- vendor whose value was NOT kept
    other_value   DOUBLE    NOT NULL,
    observed_at   TIMESTAMP NOT NULL,
    PRIMARY KEY (ticker, event_date, event_type, other_source)
);

-- Entry points (output of EntryPointDetector)
-- Written by Preprocessor (training) and Inferencer (live) via INSERT OR IGNORE
CREATE TABLE entry_points (
    ticker      VARCHAR      NOT NULL,
    date        VARCHAR      NOT NULL,
    hour        VARCHAR      NOT NULL,  -- t bar open time
    p_entry     DOUBLE       NOT NULL,  -- t bar open price
    PRIMARY KEY (ticker, date, hour)
);

-- Labeled samples (output of Labeler)
-- dead_position_case: "A" (next day has_data=True, ticker found in coverage) |
--                     "B" (next day has_data=True, ticker missing — possible delisting) |
--                     "C" (no next day with has_data=True — dataset boundary) |
--                     "D" (Case A entered but exit_price unresolvable — extended halt) |
--                     NULL (not a dead position)
-- Case B and D both resolve to label_dn5 via pnl=-1.0 threshold (not label_sw);
-- only Case C (dataset-boundary artifact, not a market outcome) retains label_sw.
-- is_ambiguous: TRUE if tp and sl thresholds simultaneously satisfied within the same
--               10-tick bundle during Stage 1 of track_label_breach().
--               Label is still assigned (via ambiguity_priority rule).
--               ClassBalancer can exclude these from train; Trainer can down-weight them.
CREATE TABLE labeled_samples (
    ticker              VARCHAR      NOT NULL,
    date                VARCHAR      NOT NULL,
    hour                VARCHAR      NOT NULL,
    p_entry             DOUBLE       NOT NULL,
    label_up5           INTEGER      NOT NULL,  -- 1 or 0
    label_up3           INTEGER      NOT NULL,
    label_sw            INTEGER      NOT NULL,
    label_dn3           INTEGER      NOT NULL,
    label_dn5           INTEGER      NOT NULL,
    is_dead_position    BOOLEAN      NOT NULL DEFAULT FALSE,
    dead_position_case  VARCHAR,                -- "A" | "B" | "C" | "D" | NULL
    is_ambiguous        BOOLEAN      NOT NULL DEFAULT FALSE,
    PRIMARY KEY (ticker, date, hour)
);

-- Train log (output of run_train.py — one row per training run or fold)
-- trial_idx:      -1  = outer evaluation row (not part of inner trial search)
--                  0  = standalone / selection / full (single implicit trial);
--                       also: exploitation final model (outer_fold_idx=-1)
--                 >=0 = exploitation inner trial (0-based per optimizer_run_id)
-- fold_idx:       -1  = standalone / outer eval (no rolling fold structure)
--                 >=0 = rolling inner fold index (0-based)
-- outer_fold_idx: -1  = non-nested run (standalone, selection, full, non-nested exploitation)
--                 >=0 = nested validation outer fold index (0-based)
-- fold_train_end: last date of train window ('YYYYMMDD'); NULL for standalone.
-- auc_std:        std of AUC across all folds in the same run or trial.
--                 Pruned trials retain auc_std = NULL; is_pruned = TRUE on fold_run_ids[-1].
-- phase:          "selection" | "exploitation" | "full" | NULL (standalone)
-- is_pruned:      TRUE if this row is the last round of a Successive Halving pruned trial
-- feature_config: JSON of active hyperparameter config (optimizer trials) or
--                 full config snapshot (standalone). Never NULL.
CREATE TABLE train_log (
    run_id              VARCHAR      NOT NULL,
    optimizer_run_id    VARCHAR,
    run_at              VARCHAR      NOT NULL,
    phase               VARCHAR,                 -- "selection"|"exploitation"|"full"|NULL
    trial_idx           INTEGER      NOT NULL DEFAULT 0,
    fold_idx            INTEGER      NOT NULL DEFAULT -1,
    outer_fold_idx      INTEGER      NOT NULL DEFAULT -1,
    fold_train_end      VARCHAR,                 -- 'YYYYMMDD'; NULL for standalone
    feature_config      VARCHAR      NOT NULL,
    n_features          INTEGER,
    n_features_reduced  INTEGER,
    auc_up5             DOUBLE,
    auc_up3             DOUBLE,
    auc_sw              DOUBLE,
    auc_dn3             DOUBLE,
    auc_dn5             DOUBLE,
    auc_mean            DOUBLE,
    auc_std             DOUBLE,
    auc_reduced_mean    DOUBLE,
    best_of_loop        BOOLEAN,
    is_pruned           BOOLEAN      NOT NULL DEFAULT FALSE,
    notes               VARCHAR,
    PRIMARY KEY (run_id)
);

-- Experiment log (output of run_backtest.py — one row per completed backtest)
-- fold_idx:        -1  = standalone / outer eval (not a rolling inner fold)
--                  >=0 = rolling inner fold index
-- outer_fold_idx:  -1  = non-nested run; >=0 = nested outer fold index
-- eval_type:       NULL              = standard backtest (standalone or non-nested exploitation)
--                  "outer_validation"= nested validation outer fold evaluation
--                  "regime_holdout"  = regime holdout robustness check
-- fold_test_start, fold_test_end:
--   Optimizer context: derived from fold_meta["fold_test_start/end"].
--   Standalone mode:   derived from test_df["date"].min() and .max() (never NULL).
-- suppressed_count: entries blocked by suppress_threshold during backtest.
CREATE TABLE experiment_log (
    run_id               VARCHAR      NOT NULL,
    optimizer_run_id     VARCHAR,
    run_at               VARCHAR      NOT NULL,
    fold_idx             INTEGER      NOT NULL DEFAULT -1,
    outer_fold_idx       INTEGER      NOT NULL DEFAULT -1,
    eval_type            VARCHAR,                -- NULL | "outer_validation" |
                                                 -- "regime_holdout"
    fold_test_start      VARCHAR,                -- 'YYYYMMDD'
    fold_test_end        VARCHAR,                -- 'YYYYMMDD'
    winning_rate         DOUBLE,
    total_trades         INTEGER,
    winning_trades       INTEGER,
    avg_pnl_pct          DOUBLE,
    total_pnl_abs        DOUBLE,
    avg_slippage_pct     DOUBLE,
    dead_position_count  INTEGER,
    dead_position_rate   DOUBLE,
    suppressed_count     INTEGER,
    trades_by_signal     VARCHAR,                -- JSON
    trades_by_exit       VARCHAR,                -- JSON
    PRIMARY KEY (run_id)
);

-- Trade log (output of BacktestEngine — one row per executed trade;
-- also written by LiveModeRunner in shadow mode — see is_shadow below and
-- live_mode_runner.md's Shadow Mode section)
CREATE TABLE trade_log (
    run_id                  VARCHAR      NOT NULL,
    ticker                  VARCHAR      NOT NULL,
    date                    VARCHAR      NOT NULL,
    entry_bar               INTEGER      NOT NULL,  -- HHMMSS as integer
    exit_bar                INTEGER      NOT NULL,
    signal                  VARCHAR      NOT NULL,  -- "up5" | "up3"
    fill_price              DOUBLE       NOT NULL,
    exit_price              DOUBLE,
    weighted_avg_exit_price DOUBLE,
    pnl_pct                 DOUBLE,
    exit_reason             VARCHAR,     -- includes 'entry_canceled' (quantity=0,
                                        -- fill_price=p_entry, exit_bar=entry_bar) —
                                        -- entry-side order fully unfilled by
                                        -- cancel_after_seconds; a real observation,
                                        -- not omitted from the table (see
                                        -- shadow_retraining.md — this case is a
                                        -- direct signal for buy_rate/cancel_after_seconds
                                        -- being too tight, and must be visible to
                                        -- fit_execution_params()).
                                        -- Also includes the LIVE-ONLY, PnL-EXCLUDED
                                        -- operational family — exits driven by
                                        -- operational handling rather than by the
                                        -- strategy, and therefore excluded from
                                        -- ordinary strategy PnL attribution since
                                        -- they reflect handling, not strategy
                                        -- performance:
                                        --   'feed_gap_exit' — exit condition
                                        --     triggered during a Feed Outage
                                        --     Recovery gap (live_mode_runner.md).
                                        --   'restart_gap_exit' (R-2) — tp/sl
                                        --     breached during a crash gap,
                                        --     detected retroactively on warm
                                        --     restart via gap-fill ticks + the
                                        --     2-print guard, and liquidated at
                                        --     CURRENT price (the historical breach
                                        --     price is stale by then). If the gap
                                        --     already exceeds
                                        --     execution.max_hold_bars,
                                        --     retro-detection is skipped and the
                                        --     position is liquidated immediately,
                                        --     still 'restart_gap_exit'.
                                        --   'overnight_exit' (R-3) — a position
                                        --     carried from a PRIOR trading day
                                        --     (halt-through-close, unfilled EOD
                                        --     exit, or an unmatched broker
                                        --     position of unknown date),
                                        --     liquidated at market as soon as
                                        --     tradable. Shares its liquidation
                                        --     mechanism with 'restart_gap_exit';
                                        --     the two differ only by whether the
                                        --     carry was same-day or cross-day.
                                        --   'reconcile_ghost' (R-3) — a
                                        --     live_positions row with
                                        --     status='open' but no matching
                                        --     broker position at Broker Reconcile
                                        --     (quantity=0; there was never a real
                                        --     fill to attribute).
                                        -- See live_mode_runner.md's "Broker
                                        -- Reconcile (shared procedure)" and
                                        -- "Unified Overnight Policy."
    is_dead_position        BOOLEAN      NOT NULL DEFAULT FALSE,
    slippage_pct            DOUBLE,
    quantity                INTEGER,
    partial_fills_count     INTEGER,
    unfilled_quantity        INTEGER,
    is_ambiguous            BOOLEAN      NOT NULL DEFAULT FALSE,
    is_shadow               BOOLEAN      NOT NULL DEFAULT FALSE,  -- TRUE = hypothetical
                                        -- fill from LiveModeRunner shadow mode, not a
                                        -- real BacktestEngine run; run_id distinguishes
                                        -- which shadow session (not a real backtest run_id)
    predicted_fill_price              DOUBLE,  -- pilot-stage only: simulate_entry_fill()
                                        -- run counterfactually against the same tick
                                        -- data as the real fill above, at session end
                                        -- (see shadow_retraining.md Stage 2). NULL for
                                        -- backtest/shadow rows (no "real" counterpart
                                        -- to predict against) and for scale-stage rows.
    predicted_weighted_avg_exit_price DOUBLE,  -- same, exit side
    predicted_partial_fills_count     INTEGER, -- same, exit side — diagnostic
                                        -- counterpart to partial_fills_count above
    PRIMARY KEY (run_id, ticker, date, entry_bar)
);

-- Inference log (live mode inference events and preload failures)
CREATE TABLE inference_log (
    logged_at     VARCHAR  NOT NULL,   -- 'YYYYMMDD_HHMMSS'
    ticker        VARCHAR  NOT NULL,
    date          VARCHAR  NOT NULL,
    hour          VARCHAR  NOT NULL,
    event         VARCHAR  NOT NULL,
    -- event values:
    --   "signal_fired"   : inference completed, signal emitted (up5 or up3)
    --   "no_signal"      : inference completed, no signal
    --   "suppressed"     : signal suppressed by suppress_threshold
    --   "preload_fail"   : insufficient bar history
    --   "bars_trimmed"   : t bar or later detected and auto-trimmed from input
    --   "no_detection"   : EntryPointDetector.detect() returned False
    signal        VARCHAR,             -- "up5" | "up3" | NULL
    prob_up5      DOUBLE,
    prob_up3      DOUBLE,
    prob_sw       DOUBLE,
    prob_dn3      DOUBLE,
    prob_dn5      DOUBLE,
    required_bars INTEGER,             -- populated on preload_fail only
    actual_bars   INTEGER,             -- populated on preload_fail only
    fail_reason   VARCHAR,             -- populated on preload_fail / bars_trimmed
    run_id        VARCHAR  NOT NULL,
    PRIMARY KEY (logged_at, ticker, date, hour)
);

-- Precomputed session statistics (REFERENCE_SESSION baselines)
-- Per-bar aggregated values from prior N sessions; delta smoothing applied at load time.
-- as_of_date=D: baselines computed from sessions [D-1, D-2, ..., D-N], intended for use on D.
--              Inserted by collect_daily.py the evening before D (after market close D-1).
--              LiveModeRunner queries WHERE as_of_date = today at session_start.
CREATE TABLE precomputed_session_stats (
    ticker      VARCHAR  NOT NULL,
    as_of_date  VARCHAR  NOT NULL,   -- 'YYYYMMDD': baseline for this trading date
    hour        VARCHAR  NOT NULL,   -- 'HHMMSS': time slot; '000000' for day-level metrics
    metric      VARCHAR  NOT NULL,
    -- metric values:
    --   "rvol_baseline"        : prior N sessions avg cumulative volume at this hour
    --   "rel_dvol_baseline"    : prior N sessions avg cumulative dollar volume at this hour
    --   "intra_vol_baseline"   : prior N sessions avg volume at this hour slot
    --   "intra_return_baseline": prior N sessions avg price_return at this hour slot
    --   "intra_tpm_baseline"   : prior N sessions avg tpm at this hour slot
    --   "buy_ratio_baseline"   : prior N sessions avg buy_ratio at this hour slot (tick-derived)
    --   "intra_avg_vol_per_tick_baseline" : prior N sessions avg avg_vol_per_tick at this
    --                            hour slot (tick-derived; sourced from tick_bar_aggregates,
    --                            same as buy_ratio_baseline/intra_tpm_baseline — see
    --                            utils.md populate_precomputed_session_stats() D.)
    --   "gap_pct_mean"         : prior N sessions gap mean  (hour='000000')
    --   "gap_pct_std"          : prior N sessions gap std   (hour='000000')
    n_sessions  INTEGER  NOT NULL,
    avg_value   DOUBLE   NOT NULL,   -- per-bar average over prior N sessions (no delta applied)
    std_value   DOUBLE,              -- standard deviation (optional; used for gap_pct normalization)
    count       INTEGER  NOT NULL,   -- actual sessions contributing to this average.
                                     -- May be < n_sessions for several reasons:
                                     --   (a) near dataset start (insufficient history)
                                     --   (b) halt bars reduce valid session count at
                                     --       specific hour slots (per-bar metrics only)
                                     --   (c) halt at 155900/093000 reduces gap_pct count
                                     -- Granularity: per (ticker, as_of_date, hour, metric)
                                     -- count may differ by hour slot within same ticker/date.
    PRIMARY KEY (ticker, as_of_date, hour, metric, n_sessions)
);
-- Note: delta_minutes smoothing is applied in-memory at load time
-- (build_session_stats_dict() or load_session_stats()) — not stored in this table.
-- Changing delta_minutes requires no DB recomputation.

-- Indicator cache for live mode. Primary runtime store when
-- indicator_cache_mode = "db" (see below). Also written REGARDLESS of mode
-- as a crash-recovery backup (R-2) — persist_to_db() is called per ticker
-- right after each Eager-Pool worker finishes, so a warm restart can
-- restore via load_from_db() instead of recomputing historical_bars; in
-- "memory" mode the RAM copy stays authoritative for the running session
-- and this backup is read only on a warm restart.
-- Populated by CachingIndicatorCalculator.persist_to_db() at session_start.
-- Loaded per-ticker on first watchdog event via load_from_db() ("db" mode),
-- or by a warm restart (any mode — R-2, see live_mode_runner.md).
-- session_stats are NOT stored here — sourced from precomputed_session_stats separately.
CREATE TABLE indicator_cache (
    session_date  VARCHAR  NOT NULL,   -- 'YYYYMMDD'
    ticker        VARCHAR  NOT NULL,
    layer         VARCHAR  NOT NULL,   -- 'layer1' | 'layer2'
    indicator     VARCHAR  NOT NULL,   -- e.g. 'ema_9', 'pivot_points', 'fibonacci'
    cache_data    BLOB     NOT NULL,   -- serialized numpy array / DataFrame (Arrow IPC or pickle)
    PRIMARY KEY (session_date, ticker, layer, indicator)
);
-- Retention: current session_date only, purged at COLD session_start. On a
-- WARM RESTART (crash recovery, R-2 — see live_mode_runner.md) the purge
-- is SKIPPED — today's backup must survive so a re-crash during recovery
-- can re-restore from it.

-- Real (non-shadow) position lifecycle, persisted so a mid-session crash
-- is recoverable (R-2). The 'pending' row is written at ORDER SUBMISSION
-- time (not fill), so a limit order pending across a crash is
-- reconcilable by order_id, and the submission-vs-fill window is covered
-- even for market orders.
CREATE TABLE live_positions (
    run_id       VARCHAR NOT NULL,
    ticker       VARCHAR NOT NULL,
    date         VARCHAR NOT NULL,   -- 'YYYYMMDD'
    entry_bar    INTEGER NOT NULL,   -- HHMMSS, mirrors trade_log
    order_id     VARCHAR NOT NULL,   -- trading-API order id (submission)
    limit_price  DOUBLE,             -- NULL for market orders
    submitted_at VARCHAR NOT NULL,   -- 'YYYYMMDD_HHMMSS'
    signal       VARCHAR NOT NULL,   -- 'up5' | 'up3'
    status       VARCHAR NOT NULL,   -- 'pending'|'open'|'closed'|'canceled'
    fill_price   DOUBLE,             -- NULL until 'open'
    fill_second  INTEGER,            -- HHMMSS, NULL until 'open'
    quantity     INTEGER,            -- NULL until 'open'
    updated_at   VARCHAR NOT NULL,
    PRIMARY KEY (run_id, ticker, date, entry_bar)
);
-- status lifecycle: 'pending' -> 'open' -> 'closed'; branch 'canceled'
--   (pending expired/canceled/rejected before fill).
-- Never deleted intra-day -- closed/canceled rows retained so cooldown
--   state is derivable after a restart.
-- No entry_ticks column: tp/sl detection is WS/REST-tick driven (see
--   live_mode_runner.md's Exit Architecture), not track_price_breach()
--   in live mode, so no "t-bar ticks" field is needed here.

-- One row per session date; preserves the sizing basis across a restart.
CREATE TABLE live_session_state (
    date               VARCHAR NOT NULL,
    run_id             VARCHAR NOT NULL,
    session_start_cash DOUBLE  NOT NULL,
    started_at         VARCHAR NOT NULL,
    PRIMARY KEY (date)
);
```

---

## Common Query Patterns

```python
# Load OHLCV for a specific ticker and date range
bars = con.execute("""
    SELECT * FROM ohlcv_1min
    WHERE ticker = ?
      AND date >= ? AND date <= ?
    ORDER BY date, hour
""", ["AAPL", "20250101", "20250630"]).df()

# Load 10-tick data strictly before t bar open (feature-safe)
ticks = con.execute("""
    SELECT * FROM tick_10
    WHERE ticker = ? AND date = ? AND hour < ?
    ORDER BY hour, seq_id
""", ["AAPL", "20250714", "093000"]).df()

# Load 10-tick data for entry slippage (100s window, not limited to t bar)
ticks_entry = con.execute("""
    SELECT * FROM tick_10
    WHERE ticker = ? AND date = ?
      AND hour >= ? AND hour < ?
    ORDER BY hour, seq_id
""", ["AAPL", "20250714", "093000", "093140"]).df()  -- entry_hour + 100s

# Load full-day tick data for exit tracking and partial fill simulation
ticks_full = con.execute("""
    SELECT * FROM tick_10
    WHERE ticker = ? AND date = ?
    ORDER BY hour, seq_id
""", ["AAPL", "20250714"]).df()

# Find next day with has_data=True for dead position resolution
# (used by Labeler and BacktestEngine — consistent filter)
next_day = con.execute("""
    SELECT date, has_data FROM trading_calendar
    WHERE date > ? AND has_data = TRUE
    ORDER BY date LIMIT 1
""", ["20250714"]).fetchone()

# Load REFERENCE_SESSION baselines for a ticker on a given trading day
session_stats_raw = con.execute("""
    SELECT hour, metric, avg_value, std_value, count
    FROM precomputed_session_stats
    WHERE ticker = ? AND as_of_date = ? AND n_sessions = ?
    ORDER BY metric, hour
""", ["AAPL", "20250715", 20]).df()
# After loading, apply delta_minutes smoothing in memory via load_session_stats()

# Bulk load session_stats for ALL tickers — live Phase 1 (single date)
session_stats_bulk = con.execute("""
    SELECT ticker, as_of_date, hour, metric, avg_value, std_value, count
    FROM precomputed_session_stats
    WHERE as_of_date = ? AND n_sessions = ?
    ORDER BY ticker, metric, hour
""", ["20250715", 20]).df()
# Pass to build_session_stats_dict(); access result[today_date] for ticker-keyed dict

# Bulk load session_stats for ALL tickers and dates — training (all dates)
session_stats_all = con.execute("""
    SELECT ticker, as_of_date, hour, metric, avg_value, std_value, count
    FROM precomputed_session_stats
    WHERE n_sessions = ?
    ORDER BY as_of_date, ticker, metric, hour
""", [20]).df()
# Pass to build_session_stats_dict(); access result[date][ticker] per entry point

# Check for corporate events (splits, reverse splits, dividends) for a ticker
events = con.execute("""
    SELECT ticker, event_date, event_type, value
    FROM corporate_events
    WHERE ticker = ?
      AND event_date >= ? AND event_date <= ?
    ORDER BY event_date
""", ["AAPL", "20250101", "20250715"]).df()

# Rolling fold summary — inner trial AUC by trial and fold for a given optimizer run
fold_summary = con.execute("""
    SELECT trial_idx, fold_idx, outer_fold_idx, fold_train_end,
           auc_mean, auc_std, is_pruned, n_features, phase
    FROM train_log
    WHERE optimizer_run_id = ?
      AND fold_idx >= 0
    ORDER BY outer_fold_idx, trial_idx, fold_idx
""", ["20250519_143022"]).df()

# Best trial per outer fold
best_per_outer = con.execute("""
    SELECT outer_fold_idx, trial_idx, auc_mean, auc_std, feature_config
    FROM train_log
    WHERE optimizer_run_id = ?
      AND best_of_loop = TRUE
      AND trial_idx >= 0
    ORDER BY outer_fold_idx
""", ["20250519_143022"]).df()

# Successive Halving pruning statistics per outer fold
pruning_stats = con.execute("""
    SELECT outer_fold_idx,
           COUNT(*) FILTER (WHERE is_pruned = TRUE)          AS pruned_trials,
           COUNT(*) FILTER (WHERE is_pruned = FALSE
                              AND trial_idx >= 0)            AS completed_trials
    FROM train_log
    WHERE optimizer_run_id = ?
      AND fold_idx >= 0
    GROUP BY outer_fold_idx
    ORDER BY outer_fold_idx
""", ["20250519_143022"]).df()

# Nested validation outer fold results
outer_results = con.execute("""
    SELECT outer_fold_idx, fold_test_start, fold_test_end,
           winning_rate, total_trades, eval_type
    FROM experiment_log
    WHERE optimizer_run_id = ?
      AND eval_type = 'outer_validation'
    ORDER BY outer_fold_idx
""", ["20250519_143022"]).df()

# Regime holdout robustness result
robustness = con.execute("""
    SELECT eval_type, winning_rate, total_trades, avg_pnl_pct
    FROM experiment_log
    WHERE optimizer_run_id = ?
      AND eval_type = 'regime_holdout'
""", ["20250519_143022"]).df()

# Ambiguous sample rate in labeled_samples
ambig_rate = con.execute("""
    SELECT
        COUNT(*) FILTER (WHERE is_ambiguous)       AS ambiguous_count,
        COUNT(*)                                    AS total_count,
        COUNT(*) FILTER (WHERE is_ambiguous) * 1.0
            / COUNT(*)                              AS ambiguous_rate
    FROM labeled_samples
""").df()

# Dead position case distribution
dead_dist = con.execute("""
    SELECT dead_position_case, COUNT(*) AS count
    FROM labeled_samples
    WHERE is_dead_position = TRUE
    GROUP BY dead_position_case
    ORDER BY dead_position_case
""").df()

# Partial fill analysis — unfilled position rate
unfilled_rate = con.execute("""
    SELECT
        COUNT(*) FILTER (WHERE unfilled_quantity > 0) AS unfilled_count,
        COUNT(*)                                       AS total_trades,
        AVG(partial_fills_count)                       AS avg_bundles_to_exit
    FROM trade_log
    WHERE run_id = ?
""", ["20250715_143022"]).df()

# Trace trade_log entries to optimizer_run_id via join
trades_by_optimizer = con.execute("""
    SELECT t.*, e.optimizer_run_id
    FROM trade_log t
    JOIN experiment_log e ON t.run_id = e.run_id
    WHERE e.optimizer_run_id = ?
""", ["20250519_143022"]).df()

# Retrieve auc_std for a completed selection/full run (single run_id)
run_auc_std = con.execute("""
    SELECT auc_std, auc_mean
    FROM train_log
    WHERE run_id = ?
""", ["<fold_run_ids[-1]>"]).fetchone()

# Retrieve auc_std per completed trial for exploitation (excludes pruned)
trial_auc_std = con.execute("""
    SELECT trial_idx, outer_fold_idx, auc_std, auc_mean
    FROM train_log
    WHERE optimizer_run_id = ?
      AND auc_std IS NOT NULL
      AND is_pruned = FALSE
    ORDER BY outer_fold_idx, trial_idx
""", ["20250519_143022"]).df()

# Experiment log for standard backtest results (inner fold or standalone)
exp_results = con.execute("""
    SELECT run_id, fold_idx, fold_test_start, fold_test_end,
           winning_rate, total_trades, suppressed_count
    FROM experiment_log
    WHERE optimizer_run_id = ?
      AND (eval_type IS NULL OR eval_type NOT IN ('outer_validation', 'regime_holdout'))
    ORDER BY fold_idx
""", ["20250519_143022"]).df()
```

---

## Ingestion Rules

- All numeric fields from JSON are stored as strings → cast to DOUBLE/BIGINT on insert
- **No session filtering** — all sessions (pre-market, regular, after-market;
  `hour` 040000~200000) are stored; overnight is excluded by the ingestion
  time-range filter — see `migrate_json_to_duckdb.py` / `metadata_crawler.md`
- `seq_id` for tick_10: assigned as row-order index within each `(ticker, date, hour)` group, starting from 0
- Duplicate skip: check existence of `(ticker, date)` pair in target table before inserting; skip entire file if already present
- entry_points table: INSERT OR IGNORE — written by both Preprocessor (training) and Inferencer (live)
- After ingestion: `trading_calendar` and `ticker_data_coverage` updated via
  `utils.populate_trading_calendar()` and `utils.populate_ticker_coverage()`
- After calendar/coverage update: `precomputed_session_stats` updated via
  `utils.populate_precomputed_session_stats()` for the next trading day
