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
CREATE TABLE ticker_cik_map (
    cik             VARCHAR NOT NULL,
    ticker          VARCHAR NOT NULL,
    first_seen_date VARCHAR NOT NULL,   -- 'YYYYMMDD', first observed
    last_seen_date  VARCHAR NOT NULL,   -- 'YYYYMMDD', most recent observed (upsert-refreshed)
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
    PRIMARY KEY (ticker, event_date, event_type)
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

-- Trade log (output of BacktestEngine — one row per executed trade)
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
    exit_reason             VARCHAR,
    is_dead_position        BOOLEAN      NOT NULL DEFAULT FALSE,
    slippage_pct            DOUBLE,
    quantity                INTEGER,
    partial_fills_count     INTEGER,
    unfilled_quantity        INTEGER,
    is_ambiguous            BOOLEAN      NOT NULL DEFAULT FALSE,
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

-- Indicator cache for live mode (used only when indicator_cache_mode = "db")
-- Populated by CachingIndicatorCalculator.persist_to_db() at session_start.
-- Loaded per-ticker on first watchdog event via load_from_db().
-- session_stats are NOT stored here — sourced from precomputed_session_stats separately.
-- Not created or queried when indicator_cache_mode = "memory" (default).
CREATE TABLE indicator_cache (
    session_date  VARCHAR  NOT NULL,   -- 'YYYYMMDD'
    ticker        VARCHAR  NOT NULL,
    layer         VARCHAR  NOT NULL,   -- 'layer1' | 'layer2'
    indicator     VARCHAR  NOT NULL,   -- e.g. 'ema_9', 'pivot_points', 'fibonacci'
    cache_data    BLOB     NOT NULL,   -- serialized numpy array / DataFrame (Arrow IPC or pickle)
    PRIMARY KEY (session_date, ticker, layer, indicator)
);
-- Retention: current session_date only; prior session_date rows purged at each session_start.
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
