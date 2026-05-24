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
- **No time-of-day filtering at ingestion** — all bars including pre-market and after-market are stored

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

-- Stock metadata (refreshed daily by metadata crawling tool)
CREATE TABLE stock_meta (
    ticker             VARCHAR   PRIMARY KEY,
    sector             VARCHAR,
    market_cap         DOUBLE,
    shares_outstanding BIGINT,
    price_52h          DOUBLE,
    price_52l          DOUBLE,
    avg_volume         DOUBLE,
    updated_at         VARCHAR
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
-- dead_position_case: "A" (next-day data available, ticker found) |
--                     "B" (next-day exists, ticker missing — possible delisting) |
--                     "C" (dataset boundary — next day not in dataset) |
--                     NULL (not a dead position)
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
    dead_position_case  VARCHAR,                -- "A" | "B" | "C" | NULL
    is_ambiguous        BOOLEAN      NOT NULL DEFAULT FALSE,
    PRIMARY KEY (ticker, date, hour)
);

-- Train log (output of run_train.py — one row per fold)
-- trial_idx: 0-based counter per optimizer_run_id in exploitation phase;
--            always 0 for selection/full/standalone (single implicit trial).
--            NOT NULL DEFAULT 0 — schema consistent across all phases.
-- fold_idx:  NULL for standalone; 0-based integer for rolling folds.
-- fold_train_end: last date of train window ('YYYYMMDD'); NULL for standalone.
-- auc_std:   std of AUC across all folds in the same run or trial.
--            NULL at write time — populated via UPDATE by PipelineOptimizer
--            on fold_run_ids[-1] (MAX fold_idx row, guaranteed by generate_folds() order).
-- phase:     "selection" | "exploitation" | "full" | NULL (standalone)
-- feature_config: JSON of active feature group config (optimizer trials) or
--                 full config snapshot (standalone). Never NULL.
CREATE TABLE train_log (
    run_id              VARCHAR      NOT NULL,
    optimizer_run_id    VARCHAR,
    run_at              VARCHAR      NOT NULL,
    phase               VARCHAR,                 -- "selection"|"exploitation"|"full"|NULL
    trial_idx           INTEGER      NOT NULL DEFAULT 0,
                                                 -- exploitation: 0-based per optimizer_run_id
                                                 -- selection/full/standalone: always 0
    fold_idx            INTEGER,                 -- NULL for standalone; 0-based for rolling
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
    auc_std             DOUBLE,                  -- NULL until UPDATE by PipelineOptimizer
                                                 -- set on fold_run_ids[-1] only
    auc_reduced_mean    DOUBLE,
    best_of_loop        BOOLEAN,
    notes               VARCHAR,
    PRIMARY KEY (run_id)
);

-- Experiment log (output of run_backtest.py — one row per completed backtest)
-- fold_test_start, fold_test_end:
--   Optimizer context: derived from fold metadata.
--   Standalone mode:   derived from test_df["date"].min() and .max() (never NULL).
-- suppressed_count: entries blocked by suppress_threshold during backtest.
CREATE TABLE experiment_log (
    run_id              VARCHAR      NOT NULL,
    optimizer_run_id    VARCHAR,
    run_at              VARCHAR      NOT NULL,
    fold_idx            INTEGER,
    fold_test_start     VARCHAR,                 -- 'YYYYMMDD'; derived from test_df in standalone
    fold_test_end       VARCHAR,                 -- 'YYYYMMDD'; derived from test_df in standalone
    winning_rate        DOUBLE,
    total_trades        INTEGER,
    winning_trades      INTEGER,
    avg_pnl_pct         DOUBLE,
    total_pnl_abs       DOUBLE,
    avg_slippage_pct    DOUBLE,
    dead_position_count INTEGER,
    dead_position_rate  DOUBLE,
    suppressed_count    INTEGER,
    trades_by_signal    VARCHAR,                 -- JSON
    trades_by_exit      VARCHAR,                 -- JSON
    PRIMARY KEY (run_id)
);

-- Trade log (output of BacktestEngine — one row per executed trade)
CREATE TABLE trade_log (
    trade_id                VARCHAR      NOT NULL,
    run_id                  VARCHAR      NOT NULL,
    ticker                  VARCHAR      NOT NULL,
    date                    VARCHAR      NOT NULL,
    entry_bar               INTEGER      NOT NULL,  -- int(HHMMSS)
    exit_bar                INTEGER,
    signal                  VARCHAR,                -- "up5" | "up3"
    fill_price              DOUBLE,
    weighted_avg_exit_price DOUBLE,                 -- volume-weighted across partial fills
    pnl_pct                 DOUBLE,
    quantity                INTEGER,
    total_filled            INTEGER,                -- shares actually exited
    unfilled_quantity       INTEGER,                -- 0 = fully closed
    partial_fills_count     INTEGER,                -- tick bundles used for exit
    exit_reason             VARCHAR,                -- "take_profit"|"stop_loss"|"session_end"|
                                                    -- "time_limit"|"dead_position"|...
    slippage_pct            DOUBLE,
    is_ambiguous            BOOLEAN,                -- simultaneous bundle-level tp/sl breach
    is_dead_position        BOOLEAN,
    PRIMARY KEY (trade_id)
);
```

---

## Common Query Patterns

```python
# Load 1-minute bars for a ticker date range
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

# Find next trading day for dead position resolution
next_day = con.execute("""
    SELECT date, has_data FROM trading_calendar
    WHERE date > ? AND is_trading_day = TRUE
    ORDER BY date LIMIT 1
""", ["20250714"]).fetchone()

# Rolling fold summary — AUC by trial and fold for a given optimizer run
fold_summary = con.execute("""
    SELECT trial_idx, fold_idx, fold_train_end, auc_mean, auc_std, n_features, phase
    FROM train_log
    WHERE optimizer_run_id = ?
      AND fold_idx IS NOT NULL
    ORDER BY trial_idx, fold_idx
""", ["20250519_143022"]).df()

# Retrieve auc_std for a completed selection/full run (single run_id)
run_auc_std = con.execute("""
    SELECT auc_std, auc_mean
    FROM train_log
    WHERE run_id = ?
""", ["<fold_run_ids[-1]>"]).fetchone()

# Retrieve auc_std per trial for exploitation phase
trial_auc_std = con.execute("""
    SELECT trial_idx, auc_std, auc_mean
    FROM train_log
    WHERE optimizer_run_id = ?
      AND auc_std IS NOT NULL
    ORDER BY trial_idx
""", ["20250519_143022"]).df()

# Experiment log with fold period for rolling backtest
exp_results = con.execute("""
    SELECT run_id, fold_idx, fold_test_start, fold_test_end,
           winning_rate, total_trades, suppressed_count
    FROM experiment_log
    WHERE optimizer_run_id = ?
    ORDER BY fold_idx NULLS LAST
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
```

---

## Ingestion Rules

- All numeric fields from JSON are stored as strings → cast to DOUBLE/BIGINT on insert
- **No time-of-day filter** — all bars (pre-market, regular, after-market) are stored
- `seq_id` for tick_10: assigned as row-order index within each `(ticker, date, hour)` group, starting from 0
- Duplicate skip: check existence of `(ticker, date)` pair in target table before inserting; skip entire file if already present
- entry_points table: INSERT OR IGNORE — written by both Preprocessor (training) and Inferencer (live)
- After ingestion: `trading_calendar` and `ticker_data_coverage` updated via
  `utils.populate_trading_calendar()` and `utils.populate_ticker_coverage()`
