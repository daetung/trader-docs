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
- Both directory and zip formats are supported; zip is extracted to a temp location during ingestion
- Duplicate detection: if `(ticker, date, resolution)` already exists in DuckDB, the file is skipped

---

## Table Definitions

```sql
-- 1-minute OHLCV bars
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

-- 10-tick data
-- Note: multiple ticks can share the same HHMMSS within one second.
-- seq_id is a sequential counter assigned per (ticker, date, hour) at ingestion time.
CREATE TABLE tick_10 (
    ticker   VARCHAR      NOT NULL,
    date     VARCHAR      NOT NULL,
    hour     VARCHAR      NOT NULL,  -- 'HHMMSS', tick timestamp (second precision)
    seq_id   INTEGER      NOT NULL,  -- sequential index within same (ticker, date, hour)
    open     DOUBLE       NOT NULL,  -- Oprc
    high     DOUBLE       NOT NULL,  -- Hprc
    low      DOUBLE       NOT NULL,  -- Lprc
    close    DOUBLE       NOT NULL,  -- Prpr (last traded price)
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
    holiday_name  VARCHAR                 -- e.g. "Christmas Day", "Independence Day"
);

-- After-market tick prices (for labeling exit fallback)
-- Populated from tick_10 rows with hour > '155900'
-- Stored separately to keep tick_10 session-clean
CREATE TABLE after_market_ticks (
    ticker   VARCHAR   NOT NULL,
    date     VARCHAR   NOT NULL,
    hour     VARCHAR   NOT NULL,   -- 'HHMMSS', after 15:59
    seq_id   INTEGER   NOT NULL,
    price    DOUBLE    NOT NULL,
    volume   BIGINT    NOT NULL,
    PRIMARY KEY (ticker, date, hour, seq_id)
);

-- Entry points (output of EntryPointDetector)
CREATE TABLE entry_points (
    ticker      VARCHAR      NOT NULL,
    date        VARCHAR      NOT NULL,
    hour        VARCHAR      NOT NULL,  -- t bar open time (entry execution bar)
    p_entry     DOUBLE       NOT NULL,  -- t bar open price
    PRIMARY KEY (ticker, date, hour)
);

-- Labeled samples (output of Labeler)
CREATE TABLE labeled_samples (
    ticker      VARCHAR      NOT NULL,
    date        VARCHAR      NOT NULL,
    hour        VARCHAR      NOT NULL,
    p_entry     DOUBLE       NOT NULL,
    label_up5   INTEGER      NOT NULL,  -- 1 or 0
    label_up3   INTEGER      NOT NULL,
    label_sw    INTEGER      NOT NULL,
    label_dn3   INTEGER      NOT NULL,
    label_dn5   INTEGER      NOT NULL,
    PRIMARY KEY (ticker, date, hour)
);

-- Trade log (output of BacktestEngine)
-- entry_bar and exit_bar stored as HHMMSS-derived integers (e.g. "093000" → 93000).
-- BacktestEngine converts hour VARCHAR to integer before writing.
-- To join back to ohlcv_1min: CAST(entry_bar AS VARCHAR) zero-padded to 6 digits → hour.
CREATE TABLE trade_log (
    tr_id           VARCHAR     NOT NULL,  -- UUID
    run_id          VARCHAR,
    ticker          VARCHAR,
    date            VARCHAR,
    entry_bar       INTEGER,               -- HHMMSS as int (e.g. 93000, 100500)
    exit_bar        INTEGER,               -- HHMMSS as int
    signal          VARCHAR,               -- "up3" | "up5"
    fill_price      DOUBLE,
    exit_price      DOUBLE,
    quantity        INTEGER,
    pnl_pct         DOUBLE,
    pnl_abs         DOUBLE,                -- NULL in inf mode
    exit_reason     VARCHAR,               -- "take_profit" | "stop_loss" | "time_limit"
    slippage_pct    DOUBLE,
    cash_remaining  DOUBLE,                -- NULL in inf mode
    PRIMARY KEY (tr_id)
);

-- Experiment log (output of PipelineOptimizer and BacktestEngine)
-- Written by PipelineOptimizer after each trial.
-- BacktestEngine appends backtest summary columns to the matching run_id row.
CREATE TABLE experiment_log (
    run_id              VARCHAR      NOT NULL,  -- YYYYMMDD_HHMMSS or YYYYMMDD_HHMMSS_NNN (optimizer)
    run_at              VARCHAR      NOT NULL,  -- 'YYYYMMDD_HHMMSS'
    feature_config      VARCHAR      NOT NULL,  -- JSON: active feature groups + vectorizer methods
    n_features          INTEGER,
    n_features_reduced  INTEGER,                -- after DimensionalityReducer; NULL if not run
    auc_up5             DOUBLE,
    auc_up3             DOUBLE,
    auc_sw              DOUBLE,
    auc_dn3             DOUBLE,
    auc_dn5             DOUBLE,
    auc_mean            DOUBLE,
    auc_reduced_mean    DOUBLE,                 -- mean AUC on reduced feature set; NULL if not run
    winning_rate        DOUBLE,
    total_trades        INTEGER,
    winning_trades      INTEGER,
    avg_pnl_pct         DOUBLE,
    total_pnl_abs       DOUBLE,
    avg_slippage_pct    DOUBLE,
    trades_by_signal    VARCHAR,                -- JSON: {"up3": {...}, "up5": {...}}
    trades_by_exit      VARCHAR,                -- JSON: {"take_profit": n, "stop_loss": n, "time_limit": n}
    notes               VARCHAR,
    PRIMARY KEY (run_id)
);
```

---

## Common Query Patterns

```python
import duckdb
con = duckdb.connect("data/market.duckdb")

# Load N trading days of 1min bars for a ticker
df = con.execute("""
    SELECT * FROM ohlcv_1min
    WHERE ticker = ? AND date >= ? AND date <= ?
    ORDER BY date, hour
""", ["AAPL", "20250101", "20250630"]).df()

# Load 10-tick data strictly before t bar open (feature-safe)
# Order by hour, seq_id to preserve intra-second sequence
ticks = con.execute("""
    SELECT * FROM tick_10
    WHERE ticker = ? AND date = ? AND hour < ?
    ORDER BY hour, seq_id
""", ["AAPL", "20250714", "093000"]).df()

# Load 10-tick data within t bar (backtest slippage only)
ticks_t = con.execute("""
    SELECT * FROM tick_10
    WHERE ticker = ? AND date = ? AND hour >= ? AND hour < ?
    ORDER BY hour, seq_id
""", ["AAPL", "20250714", "093000", "093100"]).df()

# Daily cumulative volume for entry detection condition D
daily_vol = con.execute("""
    SELECT SUM(volume) as daily_volume
    FROM ohlcv_1min
    WHERE ticker = ? AND date = ? AND hour <= ?
""", ["AAPL", "20250714", "093000"]).fetchone()[0]

# Reconstruct ohlcv_1min join from trade_log entry_bar
# entry_bar is stored as int (e.g. 93000); zero-pad to 6 digits to get hour VARCHAR
trades = con.execute("""
    SELECT t.*, o.high, o.low, o.close
    FROM trade_log t
    JOIN ohlcv_1min o
      ON o.ticker = t.ticker
     AND o.date   = t.date
     AND o.hour   = printf('%06d', t.entry_bar)
    WHERE t.run_id = ?
""", ["20250715_143022"]).df()
```

---

## Ingestion Rules

- All numeric fields from JSON are stored as strings → cast to DOUBLE/BIGINT on insert
- Regular market session only: `hour >= '093000'` and `hour <= '155900'`
- Pre-market / after-hours data: excluded at ingestion time
- `seq_id` for tick_10: assigned as row-order index within each `(ticker, date, hour)` group, starting from 0
- Duplicate skip: check existence of `(ticker, date)` pair in target table before inserting; skip entire file if already present
