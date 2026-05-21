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
-- Halts can occur across all time periods, not only regular session
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
    holiday_name  VARCHAR                 -- e.g. "Christmas Day", "Independence Day"
);

-- Trading calendar — all dates with trading day status and data availability
CREATE TABLE trading_calendar (
    date            VARCHAR   PRIMARY KEY,  -- 'YYYYMMDD'
    is_trading_day  BOOLEAN   NOT NULL,     -- NYSE calendar based
    is_holiday      BOOLEAN   NOT NULL,     -- cross-referenced with us_holidays
    has_data        BOOLEAN   NOT NULL,     -- ohlcv_1min has rows for this date
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
    hour        VARCHAR      NOT NULL,  -- t bar open time (entry execution bar)
    p_entry     DOUBLE       NOT NULL,  -- t bar open price
    PRIMARY KEY (ticker, date, hour)
);

-- Labeled samples (output of Labeler)
-- dead_position_case: "A" (next-day data available, ticker found) |
--                     "B" (next-day exists, ticker missing — possible delisting) |
--                     "C" (dataset boundary — next day not in dataset) |
--                     NULL (not a dead position)
-- is_ambiguous: TRUE if the bar that triggered the first breach had its individual
--               high >= +threshold_3pp AND low <= -threshold_3pp simultaneously.
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
-- fold_idx:      NULL for standalone (single split) runs; 0-based integer for rolling folds.
-- fold_train_end: last date of train window ('YYYYMMDD'); NULL for standalone.
-- auc_std:       std of AUC across all folds in the same run or trial.
--                NULL at write time — populated via UPDATE by PipelineOptimizer
--                on fold_run_ids[-1] (MAX fold_idx, guaranteed by generate_folds() order).
--                Interpretation: temporal stability of the model across market periods.
-- phase:         "selection"   — full feature set + reducer, no trial loop
--                "exploitation" — selected features, trial search loop
--                "full"        — full feature set, no reducer, one pass
--                NULL          — standalone run
-- feature_config: JSON of active feature group config (optimizer trials) or
--                 full config snapshot (standalone). Never NULL.
-- notes:         optional free-text memo; NULL by default.
CREATE TABLE train_log (
    run_id              VARCHAR      NOT NULL,
    optimizer_run_id    VARCHAR,
    run_at              VARCHAR      NOT NULL,
    phase               VARCHAR,                 -- "selection"|"exploitation"|"full"|NULL
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
                                                 -- set on fold_run_ids[-1] (MAX fold_idx) only
    auc_reduced_mean    DOUBLE,
    best_of_loop        BOOLEAN,
    notes               VARCHAR,
    PRIMARY KEY (run_id)
);

-- Experiment log (output of run_backtest.py — one row per completed backtest)
-- fold_idx, fold_test_start, fold_test_end: populated when called from rolling context;
--   NULL for standalone CLI runs. In standalone mode, fold_test_start/end derived
--   from test_df date range.
-- suppressed_count: entries blocked by suppress_threshold during backtest.
CREATE TABLE experiment_log (
    run_id              VARCHAR      NOT NULL,
    optimizer_run_id    VARCHAR,
    run_at              VARCHAR      NOT NULL,
    fold_idx            INTEGER,                 -- NULL for standalone
    fold_test_start     VARCHAR,                 -- 'YYYYMMDD'; NULL for standalone
    fold_test_end       VARCHAR,                 -- 'YYYYMMDD'; NULL for standalone
    winning_rate        DOUBLE,
    total_trades        INTEGER,
    winning_trades      INTEGER,
    avg_pnl_pct         DOUBLE,
    total_pnl_abs       DOUBLE,
    avg_slippage_pct    DOUBLE,
    dead_position_count INTEGER,
    dead_position_rate  DOUBLE,
    suppressed_count    INTEGER,
    trades_by_signal    VARCHAR,                 -- JSON: {"up3": {...}, "up5": {...}}
    trades_by_exit      VARCHAR,                 -- JSON: {"take_profit": n, ...}
    notes               VARCHAR,
    PRIMARY KEY (run_id)
);

-- Trade log (output of BacktestEngine)
-- optimizer_run_id is intentionally NOT included in trade_log.
-- Rationale: trade_log is normalized — optimizer_run_id is accessible via join:
--     trade_log.run_id → experiment_log.run_id → experiment_log.optimizer_run_id
-- This avoids redundant storage in a high-volume table.
CREATE TABLE trade_log (
    tr_id             VARCHAR     NOT NULL,  -- UUID
    run_id            VARCHAR,
    ticker            VARCHAR,
    date              VARCHAR,
    entry_bar         INTEGER,               -- HHMMSS as int (e.g. 93000, 100500)
    exit_bar          INTEGER,
    signal            VARCHAR,               -- "up3" | "up5"
    fill_price        DOUBLE,
    exit_price        DOUBLE,
    quantity          INTEGER,
    pnl_pct           DOUBLE,
    pnl_abs           DOUBLE,                -- NULL in inf mode
    exit_reason       VARCHAR,               -- "take_profit"|"stop_loss"|"session_end"
                                             -- |"time_limit"|"dead_position"
                                             -- |"dead_position_delisted"|"dead_position_no_data"
    slippage_pct      DOUBLE,
    cash_remaining    DOUBLE,                -- NULL in inf mode
    dead_position     BOOLEAN,
    PRIMARY KEY (tr_id)
);
```

---

## Common Query Patterns

```python
import duckdb
con = duckdb.connect("data/market.duckdb")

# Load N trading days of 1min bars for a ticker (all sessions)
df = con.execute("""
    SELECT * FROM ohlcv_1min
    WHERE ticker = ? AND date >= ? AND date <= ?
    ORDER BY date, hour
""", ["AAPL", "20250101", "20250630"]).df()

# Load 10-tick data strictly before t bar open (feature-safe)
ticks = con.execute("""
    SELECT * FROM tick_10
    WHERE ticker = ? AND date = ? AND hour < ?
    ORDER BY hour, seq_id
""", ["AAPL", "20250714", "093000"]).df()

# Load 10-tick data within t bar (backtest entry slippage only)
ticks_t = con.execute("""
    SELECT * FROM tick_10
    WHERE ticker = ? AND date = ? AND hour >= ? AND hour < ?
    ORDER BY hour, seq_id
""", ["AAPL", "20250714", "093000", "093100"]).df()

# Find next trading day for dead position resolution
next_day = con.execute("""
    SELECT date, has_data FROM trading_calendar
    WHERE date > ? AND is_trading_day = TRUE
    ORDER BY date LIMIT 1
""", ["20250714"]).fetchone()

# Rolling fold summary — AUC by fold for a given optimizer run
fold_summary = con.execute("""
    SELECT fold_idx, fold_train_end, auc_mean, auc_std, n_features, phase
    FROM train_log
    WHERE optimizer_run_id = ?
      AND fold_idx IS NOT NULL
    ORDER BY fold_idx
""", ["20250519_143022"]).df()

# Retrieve auc_std for a completed run or trial
# (recorded on fold_run_ids[-1] = MAX fold_idx row,
#  guaranteed by generate_folds() ascending yield order)
run_auc_std = con.execute("""
    SELECT auc_std, auc_mean, fold_idx
    FROM train_log
    WHERE optimizer_run_id = ?
      AND fold_idx = (
          SELECT MAX(fold_idx) FROM train_log
          WHERE optimizer_run_id = ?
      )
""", ["20250519_143022", "20250519_143022"]).fetchone()

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

# Reconstruct ohlcv_1min join from trade_log entry_bar
trades = con.execute("""
    SELECT t.*, o.high, o.low, o.close
    FROM trade_log t
    JOIN ohlcv_1min o
      ON o.ticker = t.ticker
     AND o.date   = t.date
     AND o.hour   = printf('%06d', t.entry_bar)
    WHERE t.run_id = ?
""", ["20250715_143022"]).df()

# Trace trade_log entries to optimizer_run_id via join
# (optimizer_run_id intentionally excluded from trade_log for normalization)
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
