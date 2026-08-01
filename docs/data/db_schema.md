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

**Implementation contract (R-9).** The `sql` fence below is the CANONICAL
DDL REGION — the single source of truth for the schema, and the only part
of this document that becomes executable code. `tools/init_db.py` (see
init_db.md) is its sole consumer; no other component in the system issues
DDL. Any `sql` fence elsewhere in this document is illustrative and is NOT
an init target.

Transcription into `init_db.py`'s `SCHEMA_STATEMENTS` is mechanical, so
that implementing it requires no judgement: copy each statement verbatim,
drop `--` comments, preserve document order, split per statement. Never
rewrite, reorder, or normalise anything on the way through. Because the
transcription strips `--` to end of line, no string literal in this region
may contain `--`.

Statements are written `CREATE TABLE IF NOT EXISTS` rather than leaving
idempotency for the transcription to decide: re-running init against an
existing database is a per-table no-op.

What init does NOT do is alter a table that already exists. If this
document later changes a column or a key, init is silent about it — the
change is applied deliberately by the operator, not discovered at runtime.
`init_db.py --verify` exists to make that silence visible: it reports drift
read-only and alters nothing.

**Retention.** Every table below carries a `-- Retention:` line in one of
three states — structurally excluded (the purge code holds no reference to
the table and cannot reach it), session-scoped (owns its purge, already
implemented), or purge-registry member (listed in metadata_crawler.md's
evening purge stage, every one initialised to `retention_days: inf`, so
nothing is deleted until an operator sets a real window against observed
growth).

Structural exclusion is reserved for data whose loss would be
UNRECOVERABLE AND MATERIALLY HARMFUL — the training corpus, the P&L record,
historical facts that cannot be re-derived, and training-reproducibility
artifacts. It exists so that no misconfiguration can reach them, which is
why it is a structural property rather than an infinite default. Diagnostic
and operational tables do NOT qualify: losing diagnostics costs visibility,
not correctness, and it is recoverable in the sense that the system keeps
working. They are registry members at `inf` instead — identical behaviour
today, but the door stays open once growth has actually been measured.
Where a window would cost something specific, that table's own comment says
so.

```sql
-- 1-minute OHLCV bars (all sessions: pre-market, regular, after-market)
-- Retention: NEVER purged — structurally excluded from the purge registry
--   (training corpus).
CREATE TABLE IF NOT EXISTS ohlcv_1min (
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
-- Retention: NEVER purged — structurally excluded from the purge registry
--   (training corpus).
CREATE TABLE IF NOT EXISTS tick_10 (
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
-- Retention: NEVER purged — structurally excluded from the purge registry
--   (permanent historical store).
CREATE TABLE IF NOT EXISTS tick_bar_aggregates (
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

-- Bid/ask auxiliary signal collection (Real-time bid/ask spread open item).
-- Two sources, deliberately not normalized to a common shape — each stores
-- only what it actually provides; a source's columns it does not supply
-- are NULL on its own rows, not zero-filled or derived:
--   'signal_time_rest'  : fired async, fire-and-forget, at entry-signal
--                         detection (Watchdog Polling Loop step 5c, before
--                         gate evaluation — every signal, gated or not,
--                         real or shadow). Full 5-level book.
--   'ws_tick_piggyback'  : extracted from the same inbound tick handler
--                         Exit Architecture already processes (2-print
--                         guard) for a position's live tick subscription —
--                         no new subscription, no new call. Only covers
--                         the window a position is tracked, and only
--                         1st-level price with side-total size, not a
--                         5-level book.
-- Neither source's spread is stored — ask_price_1 - bid_price_1, computed
-- at query time, same convention as OHLCV storing O/H/L/C rather than a
-- derived range.
-- Retention: purge-registry member — date_column `date`, retention_days: inf
--   (see metadata_crawler.md's evening purge stage).
CREATE TABLE IF NOT EXISTS bid_ask_snapshots (
    ticker          VARCHAR NOT NULL,
    date            VARCHAR NOT NULL,   -- 'YYYYMMDD'
    hour            VARCHAR NOT NULL,   -- 'HHMMSS' — same type as tick_10 /
                                        -- ohlcv_1min / tick_bar_aggregates,
                                        -- so joins against tick_10 (the
                                        -- signal_time_rest capture point's
                                        -- own timestamp source) need no cast
    seq_id          INTEGER NOT NULL,   -- same convention as tick_10
    source          VARCHAR NOT NULL,   -- 'signal_time_rest' | 'ws_tick_piggyback'

    -- 'signal_time_rest' only — 5-level book; NULL on 'ws_tick_piggyback' rows
    bid_price_1 DOUBLE, bid_price_2 DOUBLE, bid_price_3 DOUBLE, bid_price_4 DOUBLE, bid_price_5 DOUBLE,
    ask_price_1 DOUBLE, ask_price_2 DOUBLE, ask_price_3 DOUBLE, ask_price_4 DOUBLE, ask_price_5 DOUBLE,
    bid_size_1  DOUBLE, bid_size_2  DOUBLE, bid_size_3  DOUBLE, bid_size_4  DOUBLE, bid_size_5  DOUBLE,
    ask_size_1  DOUBLE, ask_size_2  DOUBLE, ask_size_3  DOUBLE, ask_size_4  DOUBLE, ask_size_5  DOUBLE,

    -- 'ws_tick_piggyback' only; NULL on 'signal_time_rest' rows (there,
    -- level-1 price is bid_price_1/ask_price_1 above — not duplicated here)
    bid_size_total DOUBLE,   -- side-total resting size, not tied to one level
    ask_size_total DOUBLE,
    cum_bid_volume DOUBLE,   -- session-cumulative, buy-side filled volume
    cum_ask_volume DOUBLE,   -- session-cumulative, sell-side filled volume

    PRIMARY KEY (ticker, date, hour, seq_id, source)
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
-- Retention: NEVER purged — structurally excluded from the purge registry
--   (per-date snapshots; estimate_historical_meta() reads them back).
CREATE TABLE IF NOT EXISTS stock_meta (
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
-- Retention: NEVER purged — structurally excluded from the purge registry
--   (reference data).
CREATE TABLE IF NOT EXISTS ticker_cik_map (
    cik               VARCHAR NOT NULL,
    ticker            VARCHAR NOT NULL,
    first_seen_date   VARCHAR NOT NULL,   -- 'YYYYMMDD', first observed
    last_seen_date    VARCHAR NOT NULL,   -- 'YYYYMMDD', most recent observed (upsert-refreshed)
    rename_pending    BOOLEAN NOT NULL DEFAULT FALSE,
    quarantine_reason VARCHAR,            -- NULL | 'corporate_event_anomaly' today;
                                          -- reserved for future reasons (see above)
    trading_api_symbol   VARCHAR,         -- raw ticker code as the trading API's own
                                          -- master list returns it (NO exchange
                                          -- prefix — that is a call-time-only
                                          -- concatenation, never stored; see
                                          -- metadata_crawler.md's Trading-API
                                          -- Symbol Map). NULL until matched.
    trading_api_exchange VARCHAR,         -- 'NYSE' | 'NASDAQ' | 'AMEX' — which of the
                                          -- three per-exchange bulk calls returned
                                          -- this ticker; tagged from the call's own
                                          -- input parameter, not parsed from the
                                          -- response. Source of whatever exchange
                                          -- prefix the trading API's own symbol
                                          -- format requires at call time (exact
                                          -- prefix mapping: TBD, pending the
                                          -- normalization pass over the trading
                                          -- API's symbol-format documentation).
                                          -- NULL until matched.
    PRIMARY KEY (cik, ticker)
);

-- Quarterly/annual fundamentals from SEC EDGAR XBRL companyfacts (EAV — new
-- `metric` values register without a schema change). Sourced via
-- ticker_cik_map. Point-in-time correctness depends on `filed_date`, not
-- `fiscal_period_end` — see data_boundary.md.
-- Retention: NEVER purged — structurally excluded from the purge registry
--   (historical fact, not reconstructable).
CREATE TABLE IF NOT EXISTS fundamentals_quarterly (
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
-- Retention: NEVER purged — structurally excluded from the purge registry
--   (historical fact, not reconstructable).
CREATE TABLE IF NOT EXISTS ticker_history (
    current_ticker  VARCHAR NOT NULL,   -- current (post-rename) symbol
    previous_ticker VARCHAR NOT NULL,   -- prior symbol
    effective_date  VARCHAR NOT NULL,   -- 'YYYYMMDD', first date current_ticker is used
    rename_type     VARCHAR NOT NULL DEFAULT 'rename',
    PRIMARY KEY (current_ticker, effective_date)
);

-- Trading halts (crawled from NYSE, refreshed daily)
-- Retention: NEVER purged — structurally excluded from the purge registry
--   (historical fact, not reconstructable).
CREATE TABLE IF NOT EXISTS trading_halts (
    ticker        VARCHAR   NOT NULL,
    date          VARCHAR   NOT NULL,   -- 'YYYYMMDD'
    halt_start    VARCHAR   NOT NULL,   -- 'HHMMSS'
    halt_end      VARCHAR,              -- NULL if halt not yet resumed at crawl time
    reason_code   VARCHAR,              -- NYSE halt reason code (e.g. "T1", "T6", "LUDP")
    PRIMARY KEY (ticker, date, halt_start)
);

-- US market holidays (NYSE calendar)
-- Retention: NEVER purged — structurally excluded from the purge registry
--   (reference data).
CREATE TABLE IF NOT EXISTS us_holidays (
    date          VARCHAR   PRIMARY KEY,  -- 'YYYYMMDD'
    holiday_name  VARCHAR
);

-- Trading calendar — all dates with trading day status and data availability
-- Retention: NEVER purged — structurally excluded from the purge registry
--   (reference data).
CREATE TABLE IF NOT EXISTS trading_calendar (
    date            VARCHAR   PRIMARY KEY,  -- 'YYYYMMDD'
    is_trading_day  BOOLEAN   NOT NULL,
    is_holiday      BOOLEAN   NOT NULL,
    has_data        BOOLEAN   NOT NULL,
    updated_at      VARCHAR
);

-- Ticker-level data coverage per date
-- Retention: NEVER purged — structurally excluded from the purge registry
--   (coverage index over the corpus).
CREATE TABLE IF NOT EXISTS ticker_data_coverage (
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
-- Retention: purge-registry member — date_column `date`, retention_days: inf
--   (see metadata_crawler.md's evening purge stage).
CREATE TABLE IF NOT EXISTS batch_runs (
    stage        VARCHAR   NOT NULL,  -- 'premarket_rename' |
                                      -- 'premarket_trading_api_symbol_map' |
                                      -- 'premarket_corporate_events' |
                                      -- 'premarket_quarantine_check' |
                                      -- 'premarket_quarantine_recheck' |
                                      -- 'evening_ingestion' | 'evening_tick_bar_aggregates' |
                                      -- 'evening_session_stats' |
                                      -- 'evening_investing_forward_check' |
                                      -- 'live_session_start' | 'live_session_end'
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
-- Retention: NEVER purged — structurally excluded from the purge registry
--   (fitted-parameter history).
CREATE TABLE IF NOT EXISTS execution_params (
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
-- Retention: NEVER purged — structurally excluded from the purge registry
--   (historical fact, not reconstructable).
CREATE TABLE IF NOT EXISTS corporate_events (
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
-- Retention: NEVER purged — structurally excluded from the purge registry
--   (historical fact, not reconstructable).
CREATE TABLE IF NOT EXISTS corporate_event_conflicts (
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
-- Retention: NEVER purged — structurally excluded from the purge registry
--   (training reproducibility).
CREATE TABLE IF NOT EXISTS entry_points (
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
-- Retention: NEVER purged — structurally excluded from the purge registry
--   (training reproducibility).
CREATE TABLE IF NOT EXISTS labeled_samples (
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
-- Retention: purge-registry member — date_column `run_at`, retention_days: inf
--   (see metadata_crawler.md's evening purge stage).
CREATE TABLE IF NOT EXISTS train_log (
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
-- Retention: purge-registry member — date_column `run_at`, retention_days: inf
--   (see metadata_crawler.md's evening purge stage). fold_test_start/end are
--   the DATA window, never the run time — using them here would delete rows
--   the moment an old period is backtested.
CREATE TABLE IF NOT EXISTS experiment_log (
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
    -- Entry-gate rejection counters (R-5). One per gate in the step-5c.0
    -- sequence (see live_mode_runner.md); a candidate stopped at a gate has
    -- NO trade_log row, so these are the only record it existed.
    -- Diagnostic ONLY: best_config() ranks on winning_rate alone and never
    -- reads these.
    -- Nullable on purpose — NULL means "run predates the caps", 0 means
    -- "caps were active and blocked nothing". Those are different facts.
    -- Column naming: gate_blocked_* / breaker_* prefixes group these at a
    -- glance as the table grows, and gate_blocked_* matches
    -- inference_log.gate_result's own values name-for-name.
    -- Five counters, not ten: `gate_result` has ten non-'submitted'
    -- values, but five have no counter here — 'freeze', 'not_tradable' and
    -- 'bar_integrity' have no backtest concept at all (N-5, see
    -- execution_common.md; 'bar_integrity' is a live bar-arrival-latency
    -- verdict, and backtest has no bar arrival to be late);
    -- 'breaker' is evaluated in backtest but never enforced (see
    -- breaker_* below, which reports the underlying metrics instead of a
    -- blocked-count that would imply enforcement); 'error' has no
    -- backtest analogue since backtest makes no network calls.
    gate_blocked_cap_tickers     INTEGER,  -- candidates stopped by
                                 -- execution.max_tickers. Counts CANDIDATES,
                                 -- not distinct tickers — the name reads the
                                 -- other way at a glance.
    gate_blocked_cap_per_ticker  INTEGER,  -- candidates stopped by
                                 -- execution.max_positions_per_ticker.
                                 -- Mutually exclusive with the counter above
                                 -- by construction, not by gate order: the
                                 -- ticker cap only fires when the ticker is
                                 -- NOT already held, the per-ticker cap only
                                 -- when it IS. Summing them double-counts
                                 -- nothing.
    gate_blocked_cooldown        INTEGER,  -- candidates stopped by can_enter()
    gate_blocked_sizing_zero     INTEGER,  -- compute_position_size() returned 0
                                 -- (typically the vol_based leg on a thin
                                 -- t bar), so there was nothing to submit
    gate_blocked_funds           INTEGER,  -- check_funds_available() returned
                                 -- proceed=False. Counts only the FULL skip
                                 -- (not even one share affordable). Under the
                                 -- default use_all_cash: true, ordinary cash
                                 -- pressure shows up as a sized-DOWN order
                                 -- that proceeds — visible through
                                 -- trade_log.requested_quantity, not here —
                                 -- so a cash-starved run can legitimately
                                 -- report a LOW value here. With
                                 -- initial_cash: 0 the gate is never called
                                 -- and this is always 0: not "nothing was
                                 -- blocked", but "the gate never ran".
    -- R-4 breaker metrics (computed every session; thresholds default to 0
    -- = no limit, so these exist to give Pilot calibration data before any
    -- limit is armed — see shadow_retraining.md). Backtest COMPUTES these
    -- the same way live does but never enforces a trip (see
    -- 09_backtest_engine.md) — reporting raw metrics rather than a trip
    -- count keeps that true regardless of whether the thresholds are set,
    -- since a trip count would read as zero (and therefore uninformative)
    -- for every run while they default to disabled.
    breaker_max_realized_loss_abs   DOUBLE,   -- always populated
    breaker_max_realized_loss_pct   DOUBLE,   -- NULL when initial_cash=0 —
                                 -- inf mode has no equity basis to divide
                                 -- by, and 0 would misread as "no loss"
    breaker_max_consecutive_losses  INTEGER,
    breaker_peak_entries_per_hour   INTEGER,
    trades_by_signal     VARCHAR,                -- JSON
    trades_by_exit       VARCHAR,                -- JSON
    -- Column vs JSON: trades_by_exit is JSON because exit_reason is an OPEN,
    -- growing set (four new values landed in this session alone). The gate
    -- and breaker counters above are CLOSED sets known at design time, so
    -- they are columns — this is what makes `WHERE
    -- gate_blocked_cap_tickers > 0` a plain filter instead of a
    -- json_extract() + cast. DuckDB is columnar, so a query that does not
    -- select these columns does not pay for their presence; the column
    -- count on this table is not a performance concern at the row volumes
    -- this table sees (one row per backtest run).
    PRIMARY KEY (run_id)
);

-- Trade log (output of BacktestEngine — one row per executed trade;
-- also written by LiveModeRunner in shadow mode — see is_shadow below and
-- live_mode_runner.md's Shadow Mode section)
-- Retention: NEVER purged — structurally excluded from the purge registry
--   (the P&L record).
CREATE TABLE IF NOT EXISTS trade_log (
    run_id                  VARCHAR      NOT NULL,
    ticker                  VARCHAR      NOT NULL,
    date                    VARCHAR      NOT NULL,
    entry_bar               INTEGER      NOT NULL,  -- HHMMSS as integer
    exit_bar                INTEGER      NOT NULL,
    exit_date               VARCHAR      NOT NULL,  -- 'YYYYMMDD' — the date exit_bar
                                        -- belongs to. ALWAYS populated: equal to
                                        -- `date` for every exit resolved on the entry
                                        -- date, which is the overwhelming majority.
                                        -- NULL was rejected because every date-scoped
                                        -- query would then need COALESCE for a rare
                                        -- case. `date` alone is insufficient because
                                        -- it is the ENTRY date, and exit_bar is only
                                        -- HHMMSS: for a carried position the pair
                                        -- (date, exit_bar) reads as an exit BEFORE its
                                        -- own entry. Differs from `date` in exactly
                                        -- two situations:
                                        --   dead position Case A — resolved against
                                        --     D+1 data (09_backtest_engine.md)
                                        --   'overnight_exit' — liquidated on a later
                                        --     session, possibly several days later
                                        --     across a weekend, holiday, or multi-day
                                        --     halt (live_mode_runner.md)
                                        -- health_report.md's "what did this session
                                        -- close" queries key on this column, not
                                        -- `date`.
    signal                  VARCHAR      NOT NULL,  -- "up5" | "up3"
    fill_price              DOUBLE       NOT NULL,
    exit_price              DOUBLE,
    weighted_avg_exit_price DOUBLE,
    pnl_pct                 DOUBLE,
    exit_reason             VARCHAR,     -- TWO families below sit outside ordinary
                                        -- strategy PnL attribution, for DIFFERENT
                                        -- reasons. They are named and kept separate
                                        -- deliberately: health_report.md reports them
                                        -- as distinct findings, and
                                        -- 09_backtest_engine.md's summary denominators
                                        -- treat only one of them as a trade.
                                        --
                                        -- (1) NEVER-OPENED family — no position ever
                                        -- existed (quantity=0, pnl_pct=NULL), so there
                                        -- is no PnL to exclude; it structurally does
                                        -- not exist. Excluded from total_trades /
                                        -- winning_rate / avg_pnl_pct / total_pnl_abs,
                                        -- but still visible via trades_by_exit:
                                        --   'entry_canceled' — entry-side order fully
                                        --     unfilled by cancel_after_seconds; a real
                                        --     observation, not omitted from the table
                                        --     (see shadow_retraining.md — a direct
                                        --     signal for buy_rate/cancel_after_seconds
                                        --     being too tight, and must be visible to
                                        --     fit_execution_params()).
                                        --   'entry_rejected' (R-7, LIVE-ONLY) — the
                                        --     broker or the account refused the
                                        --     submission itself (account restriction,
                                        --     insufficient funds at the actual price,
                                        --     ticker not permitted). Same row shape as
                                        --     'entry_canceled', and it likewise counts
                                        --     as a cooldown attempt — but it is
                                        --     EXCLUDED from fit_execution_params(),
                                        --     because a refusal carries no evidence
                                        --     about how the market absorbed an order.
                                        --     The two labels are opposites on exactly
                                        --     that point.
                                        --
                                        -- (2) LIVE-ONLY OPERATIONAL family — a real
                                        -- position existed and produced real PnL, but
                                        -- was closed by operational handling rather
                                        -- than by the strategy, so it is excluded from
                                        -- attribution as reflecting handling, not
                                        -- strategy performance:
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
    reject_reason           VARCHAR,     -- R-8: the broker's own reason for refusing a
                                        -- submission. Populated ONLY on
                                        -- exit_reason='entry_rejected' rows; NULL
                                        -- everywhere else, and ALWAYS NULL in backtest,
                                        -- which has no broker to refuse anything.
                                        -- Stored as the broker returned it, NOT
                                        -- normalised to an enum: the vocabulary is an
                                        -- unverified vendor contract
                                        -- (api_contract_checklist.md), and defining an
                                        -- enum before seeing real values would mean
                                        -- inventing a mapping that silently collapses
                                        -- unmatched reasons. Normalisation is deferred
                                        -- until the vocabulary is measured.
                                        -- Load-bearing because insufficient margin, a
                                        -- non-permitted ticker and a malformed order
                                        -- need three different responses (add capital /
                                        -- fix the universe / fix the code). Under the
                                        -- risk-based intraday margin regime that
                                        -- replaced the PDT rule in June 2026, rejections
                                        -- RISING WITH EXPOSURE is normal behaviour, so
                                        -- without the reason there is no way to separate
                                        -- normal from pathological.
    requested_quantity      INTEGER,     -- quantity actually SUBMITTED — i.e. after
                                        -- check_funds_available() sized the order down
                                        -- to fit available cash. Deliberately NOT the
                                        -- pre-gate sizing output: quantity /
                                        -- requested_quantity is the fill-rate
                                        -- denominator fit_execution_params() reads
                                        -- (shadow_retraining.md), and a funds-driven
                                        -- reduction must never enter it — only
                                        -- market-driven shortfall. With full capital
                                        -- deployment as the intended operating point,
                                        -- the funds gate truncates routinely late in a
                                        -- session, so conflating the two would bias
                                        -- buy_rate downward exactly when the book is
                                        -- fullest.
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
-- Retention: purge-registry member — date_column `date`, retention_days: inf
--   (see metadata_crawler.md's evening purge stage).
CREATE TABLE IF NOT EXISTS inference_log (
    logged_at     VARCHAR  NOT NULL,   -- 'YYYYMMDD_HHMMSS_ffffff' (R-8) — microsecond
                                       -- suffix. This is a FORMAT change on a VARCHAR,
                                       -- not a TIMESTAMP precision change; fixed-width
                                       -- zero padding keeps lexicographic ordering
                                       -- equal to chronological ordering.
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
    gate_result   VARCHAR,             -- R-5: what became of this candidate at
                                       -- the entry gates. Populated on
                                       -- event='signal_fired' rows ONLY; NULL
                                       -- everywhere else. Values, in the same
                                       -- order as the step-5c.0 sequence in
                                       -- live_mode_runner.md:
                                       --   'submitted'      : cleared every gate
                                       --   'freeze'         : a freeze_reason
                                       --                      covering
                                       --                      entry_submission
                                       --                      (feed_outage or
                                       --                      restart_warmup)
                                       --   'error'          : the gate sequence
                                       --                      itself raised
                                       --                      (the only
                                       --                      network call
                                       --                      among the
                                       --                      gates is
                                       --                      check_funds_
                                       --                      available());
                                       --                      the candidate
                                       --                      is otherwise
                                       --                      unaccounted
                                       --                      for without
                                       --                      this
                                       --   'breaker'        : R-4 circuit-breaker
                                       --                      trip. A SEPARATE value
                                       --                      from 'freeze' even
                                       --                      though both work through
                                       --                      freeze_reasons: folding
                                       --                      them together would bury
                                       --                      the one event of the
                                       --                      three that most needs to
                                       --                      be seen.
                                       --   'cap_tickers'    : execution.max_tickers
                                       --   'cap_per_ticker' : execution
                                       --                      .max_positions_per_ticker
                                       --   'cooldown'       : can_enter()
                                       --   'not_tradable'   : is_tradable()
                                       --   'bar_integrity'  : this ticker is in
                                       --                      the bar-integrity
                                       --                      exclusion set — a
                                       --                      negative bar-latency
                                       --                      sample was observed
                                       --                      for it, so its
                                       --                      incremental
                                       --                      indicators may have
                                       --                      absorbed a
                                       --                      not-yet-complete bar
                                       --                      (live_mode_runner.md's
                                       --                      Bar-Close Authority;
                                       --                      health_report.md
                                       --                      finding 27). Blocks
                                       --                      NEW ENTRIES for that
                                       --                      ticker only —
                                       --                      existing positions
                                       --                      keep being managed
                                       --                      and exited, since
                                       --                      tp/sl runs off the WS
                                       --                      tick stream, not the
                                       --                      bar channel.
                                       --   'sizing_zero'    : quantity resolved to 0
                                       --   'funds'          : check_funds_available()
                                       -- A COLUMN rather than new event values,
                                       -- because gate_result is not a separate
                                       -- occurrence: it is how the
                                       -- 'signal_fired' event ENDED. One
                                       -- candidate evaluation is one event, and
                                       -- the gate verdict is that event's
                                       -- outcome, not a second event following
                                       -- it. (An earlier draft justified this by
                                       -- PK collision at second granularity;
                                       -- R-8 added `event` to the PK and
                                       -- microseconds to logged_at, so that
                                       -- reason no longer holds — the design
                                       -- stands on the one above, which is
                                       -- independent of key layout.)
                                       -- 'submitted' is explicit, not implied by
                                       -- NULL, so every signal_fired row carries
                                       -- a terminal state and
                                       -- COUNT(*) FILTER (WHERE gate_result <>
                                       -- 'submitted') is the total gate loss
                                       -- directly.
                                       -- Broker rejection is NOT in this set: it
                                       -- happens AFTER submission, so such a row
                                       -- is gate_result='submitted' and the
                                       -- outcome lands in trade_log as
                                       -- exit_reason='entry_rejected'.
                                       -- Shadow mode enforces the same gates, so
                                       -- these values mean the same thing there
                                       -- (with 'submitted' reading as "would
                                       -- have been submitted").
                                       -- Written ONCE, by LiveModeRunner, the
                                       -- instant the gate sequence ends —
                                       -- never by a later UPDATE. See
                                       -- Constraints below for why this one
                                       -- event value is LiveModeRunner's to
                                       -- write even though the rest of this
                                       -- table is Inferencer's. NULL on a
                                       -- signal_fired row means exactly one
                                       -- thing: the row predates this
                                       -- column, nothing else — a
                                       -- single-INSERT design has no
                                       -- "gate not yet evaluated"
                                       -- in-between state to also mean.
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
    PRIMARY KEY (logged_at, ticker, date, hour, event)
    -- R-8: `event` added to the PK. Without it, two DIFFERENT events about the
    -- same evaluation of the same ticker/bar in the same second collide — which
    -- is normal operation, not an anomaly (e.g. a 'bars_trimmed' warning
    -- alongside the 'signal_fired' outcome). The old PK asserted a uniqueness
    -- that was simply false. Microsecond logged_at above and this key answer
    -- different halves: the key makes the claim true, the timestamp closes the
    -- residual case of one event repeating within a second.
    -- Write mode: INSERT OR IGNORE, with the DROPPED-ROW COUNT surfaced by
    -- health_report.md. Plain INSERT was rejected: a session holding real
    -- positions must not die because a diagnostic log row collided. The usual
    -- objection to INSERT OR IGNORE — that it loses a diagnostic silently — is
    -- answered by counting the drops: the content is gone but the fact is not.
    -- INSERT OR REPLACE is worse than both (loses a row AND makes which one
    -- survives depend on ordering).
    -- General principle, applying to every diagnostic write in this system: a
    -- logging failure of ANY kind — PK collision, disk, lock, serialisation —
    -- must never abort trading. Wrap the write, count the failure, continue.
    -- Same discipline as health_report.md's write_to_log(), which always runs
    -- regardless of whether alert delivery succeeds.
    -- Note the PK's remaining scope is narrow: after the two changes above, the
    -- only thing it still prevents is duplicate insertion on replay, and live
    -- never re-logs a past bar.
);

-- Precomputed session statistics (REFERENCE_SESSION baselines)
-- Per-bar aggregated values from prior N sessions; delta smoothing applied at load time.
-- as_of_date=D: baselines computed from sessions [D-1, D-2, ..., D-N], intended for use on D.
--              Inserted by collect_daily.py the evening before D (after market close D-1).
--              LiveModeRunner queries WHERE as_of_date = today at session_start.
-- Retention: NEVER purged — structurally excluded from the purge registry
--   (historical baselines).
CREATE TABLE IF NOT EXISTS precomputed_session_stats (
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
-- Retention: session-scoped, its own mechanism — NOT in the purge registry.
-- Current session_date only, purged per ticker inside persist_to_db() at COLD
-- session_start. On a WARM RESTART (crash recovery, R-2 — see
-- live_mode_runner.md) the purge is SKIPPED — today's backup must survive so a
-- re-crash during recovery can re-restore from it.
CREATE TABLE IF NOT EXISTS indicator_cache (
    session_date  VARCHAR  NOT NULL,   -- 'YYYYMMDD'
    ticker        VARCHAR  NOT NULL,
    layer         VARCHAR  NOT NULL,   -- 'layer1' | 'layer2'
    indicator     VARCHAR  NOT NULL,   -- e.g. 'ema_9', 'pivot_points', 'fibonacci'
    cache_data    BLOB     NOT NULL,   -- serialized numpy array / DataFrame (Arrow IPC or pickle)
    PRIMARY KEY (session_date, ticker, layer, indicator)
);

-- Real (non-shadow) position lifecycle, persisted so a mid-session crash
-- is recoverable (R-2). The 'pending' row is written at ORDER SUBMISSION
-- time (not fill), so a limit order pending across a crash is
-- reconcilable by order_id, and the submission-vs-fill window is covered
-- even for market orders.
-- Retention: purge-registry member — date_column `date`, retention_days: inf
--   (see metadata_crawler.md's evening purge stage).
CREATE TABLE IF NOT EXISTS live_positions (
    run_id       VARCHAR NOT NULL,
    ticker       VARCHAR NOT NULL,
    date         VARCHAR NOT NULL,   -- 'YYYYMMDD'
    entry_bar    INTEGER NOT NULL,   -- HHMMSS, mirrors trade_log
    order_id     VARCHAR NOT NULL,   -- trading-API order id (submission)
    limit_price  DOUBLE,             -- NULL for market orders
    submitted_at VARCHAR NOT NULL,   -- 'YYYYMMDD_HHMMSS'
    signal       VARCHAR NOT NULL,   -- 'up5' | 'up3'
    status       VARCHAR NOT NULL,   -- 'pending'|'partial_open'|'open'|'halted'|
                                     --   'exiting'|'closed'|'canceled'
    exiting_since VARCHAR,           -- 'YYYYMMDD_HHMMSS' — set once, the FIRST
                                     --   instant status transitions to 'exiting';
                                     --   never overwritten thereafter, including
                                     --   by a stuck-timeout market escalation or a
                                     --   halt-clear resubmission (see
                                     --   live_mode_runner.md's In-flight order
                                     --   tracking) that assigns this position a
                                     --   new order_id. exit_order_stuck_minutes
                                     --   is measured against THIS column, not
                                     --   submitted_at, precisely so a resubmission
                                     --   cannot reset the clock. NULL until the
                                     --   first 'exiting' transition.
    fill_price   DOUBLE,             -- NULL until first fill; weighted average across
                                     --   partial fills once there is more than one
    fill_second  INTEGER,            -- HHMMSS of the first fill, NULL until then
    quantity     INTEGER,            -- shares filled SO FAR; NULL until first fill
    requested_quantity INTEGER,      -- shares submitted; fixed at submission. The
                                     --   in-flight tracker compares quantity against
                                     --   this to decide partial vs. complete (see
                                     --   live_mode_runner.md's In-flight order
                                     --   tracking), and it is what trade_log's own
                                     --   requested_quantity is written from.
    updated_at   VARCHAR NOT NULL,
    PRIMARY KEY (run_id, ticker, date, entry_bar)
);
-- status lifecycle:
--   'pending' -> 'open' -> 'exiting' -> 'closed'
--   'pending' -> 'partial_open' -> 'open' -> ... (R-7: fills arrive on a
--     separate channel, not in the order API's response, so an order can sit
--     partially filled. The filled shares ARE a real position from that
--     moment — they enter exit management and count toward
--     execution.max_tickers / max_positions_per_ticker — while the order
--     itself stays in flight awaiting the rest.)
--   'pending' -> 'canceled'   (expired or rejected with NOTHING filled)
--   'open'|'partial_open' <-> 'halted'  (position-scoped halt; clears itself)
-- No 'full_open': the two ways 'partial_open' ends — reaching
--   requested_quantity, or abandoning the remainder at cancel_after_seconds —
--   both mean "no longer awaiting fills", which is exactly 'open'. This also
--   matches backtest, where a partially-filled entry simply proceeds sized
--   down.
-- 'partial_open' NEVER goes to 'canceled': shares were actually bought, so
--   the outcome is a smaller position, not a non-event.
-- Never deleted intra-day -- closed/canceled rows retained so cooldown
--   state is derivable after a restart.
-- No entry_ticks column: tp/sl detection is WS/REST-tick driven (see
--   live_mode_runner.md's Exit Architecture), not track_price_breach()
--   in live mode, so no "t-bar ticks" field is needed here.

-- One row per session date; preserves the sizing basis across a restart.
-- Retention: purge-registry member — date_column `date`, retention_days: inf
--   (see metadata_crawler.md's evening purge stage).
CREATE TABLE IF NOT EXISTS live_session_state (
    date               VARCHAR NOT NULL,
    run_id             VARCHAR NOT NULL,
    session_start_cash DOUBLE  NOT NULL,
    started_at         VARCHAR NOT NULL,
    breaker_tripped_at VARCHAR,           -- R-4/Warm Restart: NULL until the
                                          -- circuit breaker trips this
                                          -- session, then the trip
                                          -- timestamp. Read (not written) by
                                          -- Warm Restart: the three breaker
                                          -- counters are recomputed from
                                          -- trade_log on every restart, but
                                          -- entries_per_hour is a ROLLING
                                          -- window that decays during
                                          -- downtime, so recomputing alone
                                          -- can let a real trip fall back
                                          -- under threshold after a long
                                          -- outage — which would silently
                                          -- violate R-4's "no auto-clear
                                          -- within a session". This column
                                          -- is what makes the trip persist
                                          -- regardless of what the
                                          -- recomputed counters say. Not a
                                          -- new table: live_session_state
                                          -- already exists for exactly this
                                          -- purpose (session_start_cash is
                                          -- restored from here, not
                                          -- re-queried).
    session_diagnostics JSON,             -- R-9: crash-durable home for the
                                          -- findings that used to be in-memory
                                          -- tallies handed to health_report.py
                                          -- at session end and lost outright if
                                          -- the session crashed (5, 8, 21, 22).
                                          -- ONE JSON column, not ~10 scalar
                                          -- ones: finding 21 is a set of probe
                                          -- measurements that grows as
                                          -- api_contract_checklist.md rows are
                                          -- filled in, and finding 8 is a
                                          -- multi-valued numerator/denominator
                                          -- (halt-check counts per
                                          -- signal_source), neither of which is
                                          -- a scalar. Not health_events either:
                                          -- that table is scoped to individual
                                          -- ABNORMAL events, while 21 records
                                          -- SUCCESSFUL probes and 8 counts
                                          -- mostly-normal checks — putting them
                                          -- there would dissolve the
                                          -- distinction it is built on.
                                          -- Keys: clock_offset_start,
                                          -- clock_offset_end, margin_ratio,
                                          -- retention_boundary (each probe also
                                          -- flagged 'disabled' | 'succeeded' |
                                          -- 'fell_back', so "never ran" and
                                          -- "ran and fell back" stay distinct —
                                          -- the fallbacks are safe but they
                                          -- silently change sizing and gap-fill
                                          -- behaviour); the finding-5
                                          -- tier-fallback summary; finding 8's
                                          -- halt-check source counts; finding
                                          -- 22's dropped-row count; and
                                          -- restart_count.
                                          -- clock_offset_end is written NULL at
                                          -- session start rather than left
                                          -- absent, so its absence at report
                                          -- time reads as "never reached
                                          -- shutdown" instead of as a missing
                                          -- key.
                                          -- Written ONLY by whole-value
                                          -- replacement, never read-modify-
                                          -- write — see live_mode_runner.md's
                                          -- "session_diagnostics write
                                          -- protocol" for the four write points
                                          -- and the warm-restart rule
                                          -- (probe values overwritten, counters
                                          -- resumed).
    PRIMARY KEY (date)
);

-- Bar-arrival latency accumulation (T-13 self-measurement — see
-- api_contract_checklist.md and health_report.md finding 26). One row per
-- (date, sample_class).
-- WHY A TABLE and not an in-memory tally handed to health_report.py the way
-- findings 5/8 are: those have nowhere to be stored; this does, and it needs
-- somewhere for two independent reasons.
--   (a) Crash loss is NOT random with respect to what is being measured. A
--       crash correlates with feed trouble — i.e. with exactly the
--       high-latency days — so losing those sessions biases the curve
--       optimistic, which is the one direction this measurement must not err.
--   (b) The process restarts every session, so a months-long accumulation —
--       which is what recalibrating bar_close_grace_seconds actually needs —
--       cannot live in memory at all.
-- Same reasoning finding 19 gives for reading gate_result from its own column
-- rather than being passed a tally.
-- Retention: purge-registry member — date_column `date`, retention_days: inf
-- (see metadata_crawler.md's evening purge stage). NOT structurally excluded:
-- losing latency history is recoverable and harmless, unlike the corpus or the
-- P&L record. But this table's value IS its multi-month accumulation, so any
-- window an operator eventually sets must be LONGER than the recalibration
-- horizon for bar_close_grace_seconds — a short one would delete the thing the
-- table exists to build. Rows are ~2-3 per day, so there is little to gain by
-- setting one at all.
-- Sample admission and the meaning of `d` are defined in live_mode_runner.md's
-- Bar-Close Authority, not here: d = now - (bar.hour + 60s), measured at the
-- Step 2a fetch that FIRST returned that bar, and a sample is admitted only
-- when its own error bound e = min(delta, d) is within
-- live_mode.bar_latency_max_error_seconds (delta = age of the prior successful
-- fetch for that ticker; absent on a first poll, where e = d).
CREATE TABLE IF NOT EXISTS bar_latency_daily (
    date          VARCHAR NOT NULL,   -- 'YYYYMMDD' — the trading day the WRITING
                                      --   PROCESS started on, fixed once at start
                                      --   and never re-evaluated (same rule
                                      --   metadata_crawler.md's Constraints state
                                      --   for the evening batch). A live session
                                      --   does not cross midnight, so here this is
                                      --   a consistency rule, not a live hazard.
    sample_class  VARCHAR NOT NULL,   -- 'poll_continuous' : e <= poll interval, the
                                      --                     tightest samples
                                      -- 'wide_error'      : admitted, but e larger
                                      -- 'negative'        : d < 0 — see below; NOT
                                      --                     a latency sample at all

    -- Latency distribution in whole-second buckets: bucket_k counts samples with
    -- floor(d) = k, bucket_overflow counts d >= 10s. Whole seconds because
    -- bar_close_grace_seconds is itself judged in whole seconds, so finer
    -- resolution could not change any decision this feeds.
    -- Read as a CUMULATIVE curve ("what fraction was ready within X seconds"),
    -- not as a density histogram. Every sample is an UPPER BOUND on the vendor's
    -- true publish latency (d overstates it by at most e, never understates it),
    -- so "97% within 5s" is a conservative floor on the truth — whereas reading
    -- bucket_5 as "these took 5s" would claim more than the data supports.
    -- Uniform 1s buckets rather than boundaries aligned to candidate grace
    -- values: uniform buckets cover the entire decision space and stay correct
    -- if the candidate set ever changes.
    -- On a 'negative' row these same buckets hold the |d| distribution instead:
    -- a pile-up near 60s means the vendor timestamps bars by CLOSE while
    -- Bar-Close Authority assumes OPEN (structural, every judgement off by one
    -- bar), while small scattered values mean premature publication of a
    -- still-forming bar or a clock fault. The distinction is diagnostically
    -- decisive and costs no extra columns.
    bucket_0        INTEGER NOT NULL DEFAULT 0,
    bucket_1        INTEGER NOT NULL DEFAULT 0,
    bucket_2        INTEGER NOT NULL DEFAULT 0,
    bucket_3        INTEGER NOT NULL DEFAULT 0,
    bucket_4        INTEGER NOT NULL DEFAULT 0,
    bucket_5        INTEGER NOT NULL DEFAULT 0,
    bucket_6        INTEGER NOT NULL DEFAULT 0,
    bucket_7        INTEGER NOT NULL DEFAULT 0,
    bucket_8        INTEGER NOT NULL DEFAULT 0,
    bucket_9        INTEGER NOT NULL DEFAULT 0,
    bucket_overflow INTEGER NOT NULL DEFAULT 0,   -- d >= 10s

    max_observed_d  INTEGER,            -- tail size, in seconds. Merged with
                                        --   GREATEST(existing, incoming) on flush,
                                        --   NOT summed — see the flush rule below.
                                        --   max|d| on a 'negative' row.

    -- Samples REJECTED for e > live_mode.bar_latency_max_error_seconds, split by
    -- why the bound was loose, because the two point at different things:
    excluded_no_prior_fetch INTEGER NOT NULL DEFAULT 0,  -- no prior successful
                                        --   fetch for the ticker, so e = d and d
                                        --   was already large: the candidate
                                        --   entered late relative to bar close
    excluded_both_wide      INTEGER NOT NULL DEFAULT 0,  -- delta AND d both large:
                                        --   a stretched poll cycle coinciding with
                                        --   a slow bar
                                        -- Both are 0 on 'poll_continuous' rows by
                                        --   construction: the threshold is a
                                        --   multiple of the poll interval, so a
                                        --   sample with e <= one interval can never
                                        --   exceed it.

    bar_gap_samples INTEGER NOT NULL DEFAULT 0,   -- admitted samples whose bar
                                        --   timestamp skipped >1 minute since the
                                        --   ticker's previous bar. Paired with the
                                        --   latency buckets this separates
                                        --   structural non-publication (large gap,
                                        --   small d — a thin ticker that gets no
                                        --   zero-volume bar) from genuine vendor
                                        --   lag (large gap, large d).

    PRIMARY KEY (date, sample_class)
);
-- Flush rule (live_mode_runner.md holds the counters between flushes): the
-- in-memory tally holds only the UN-FLUSHED DELTA and is zeroed after each
-- flush, and the flush is an ADDITIVE upsert. A tally holding the session total
-- against an additive upsert would re-add everything already written on every
-- flush. max_observed_d is the sole exception, merged with GREATEST rather than
-- added — two merge semantics in one row, so a blanket `+=` across all columns
-- is wrong. This is also what makes a warm restart consistent with no extra
-- logic: what was flushed is in the table, and only the un-flushed delta (under
-- one flush interval) is lost.

-- Health-event log — one row per discrete diagnostic event (health_report.md).
-- Different in kind from bar_latency_daily above: that aggregates NORMAL
-- samples by the tens of thousands, this records individual ABNORMAL events.
-- Written ONLY through utils.record_health_event() — no detection site mints
-- its own event_id and none writes this table directly, the same
-- single-access-point discipline utils.query_halt_status() has.
-- WHY: a finding that reports "N occurrences today" loses WHEN they occurred,
-- and the when is frequently the diagnosis. It also makes an alert traceable
-- back to the specific events it reported (see alert_log.event_ids below).
-- Retention: purge-registry member — date_column `date`, retention_days: inf
-- (see metadata_crawler.md's evening purge stage). Of the tables here this is
-- the one whose growth rate can actually matter: R-9 widened its writers from
-- one finding to six, and findings 18 and 24 both emit repeatedly during a
-- broker-latency episode. Its purpose — WHEN an occurrence happened, and
-- alert traceability via alert_log.event_ids — is short-horizon, so a window is
-- legitimate here once growth has been observed. The DB health observation
-- (health_report.md) reports its row count for exactly that purpose.
-- SCOPE, deliberately wider than today's use: the schema accepts any
-- event-shaped finding. Findings 12, 14, 18, 24, 25 and 27 write here (R-9).
-- Findings 13 and 15 deliberately do NOT: each already writes its own
-- trade_log row (exit_reason='restart_gap_exit' / 'overnight_exit' /
-- 'reconcile_ghost') carrying the occurrence and its time, so recording them
-- here too would be a second source for one fact — the defect
-- corporate_events' one-row invariant exists to prevent. The rule the split
-- follows: an event belongs here when it leaves no row anywhere else.
CREATE TABLE IF NOT EXISTS health_events (
    event_id     VARCHAR NOT NULL,   -- '{YYYYMMDD}_{HHMMSSmmm}_{4 random chars}',
                                     --   generated inside record_health_event()
                                     --   (see utils.md): time-sortable, and
                                     --   collision-free across the two processes
                                     --   that can write here — a live session and
                                     --   the evening batch — without coordination
    date         VARCHAR NOT NULL,   -- 'YYYYMMDD', writing process's start day,
                                     --   same rule as bar_latency_daily.date
    finding_name VARCHAR NOT NULL,   -- health_report.md's finding KEY, not its
                                     --   documentation number: the number is list
                                     --   order and shifts, the key does not
    occurred_at  VARCHAR NOT NULL,   -- 'YYYYMMDD_HHMMSSmmm' — when the EVENT
                                     --   happened, not when this row was written;
                                     --   the two differ whenever a write is
                                     --   retried or batched
    ticker       VARCHAR,            -- NULL for session-scoped events
    detail       VARCHAR,            -- JSON. Shape is per-finding and deliberately
                                     --   unconstrained — findings differ too much
                                     --   for a shared column set to fit any of them
    PRIMARY KEY (event_id)
);

-- Alert delivery log — one row per delivery ATTEMPT, i.e. per (alert, channel).
-- Fills a gap that stood until this design: a SUCCESSFUL send was recorded
-- nowhere at all. write_to_log() records what was FOUND, and
-- event=alert_delivery_failed records failures, so "what actually reached the
-- operator, and when" was not reconstructible — which left a silently
-- degrading channel invisible and gave an alert no traceable link to the events
-- it reported.
-- outcome deliberately covers the NON-sends too: an alert held back by rate
-- limiting, or dropped from a full queue, is precisely a thing that did not
-- reach the operator, and those are the cases a delivery log exists to account
-- for. Counting them in aggregate is not enough — that records the fact but
-- loses the content.
-- Retention: purge-registry member — date_column `date`, retention_days: inf
-- (see metadata_crawler.md's evening purge stage), same reasoning as the two
-- tables above: delivery history is operational, not a record whose loss is
-- unrecoverable. Note a window here also orphans older health_events.event_ids
-- references, which is a reason to keep any window no shorter than
-- health_events'.
-- ONE EXCEPTION, structural: the evening job's hard-deadline abort
-- (metadata_crawler.md's evening job start gate) cannot write here at all — it
-- is aborting precisely BECAUSE it never acquired the DB lock, which a hung
-- LiveModeRunner still holds. That alert is file-log-only by necessity, not by
-- choice. Deferring its row to whichever process next holds the lock was
-- considered and rejected: a pending-alert-write queue is new infrastructure
-- for the one case where a human is already reading the log.
CREATE TABLE IF NOT EXISTS alert_log (
    alert_id         VARCHAR NOT NULL,   -- same format and generator style as
                                         --   health_events.event_id. One id per
                                         --   ATTEMPT, so a report going to two
                                         --   channels produces two ids
    date             VARCHAR NOT NULL,   -- 'YYYYMMDD', writing process's start day
    alert_key        VARCHAR NOT NULL,   -- the INVOCATION PATH, not the finding:
                                         --   'session_start' | 'session_end' |
                                         --   'breaker_trip' | 'clock_check_abort' |
                                         --   'bar_close_premise_violation' |
                                         --   'evening_liveness_probe' |
                                         --   'evening_deadline_abort' |
                                         --   'log_write_failed'.
                                         --   Path-keyed rather than finding-keyed
                                         --   because an event-driven invocation
                                         --   delivers a REPORT, not one finding —
                                         --   so suppression is keyed on
                                         --   (alert_key, channel) as well
    channel          VARCHAR NOT NULL,   -- 'discord' | 'email'
    severity         VARCHAR NOT NULL,   -- 'ok' | 'warn' | 'abort' — the MAXIMUM
                                         --   severity across the findings this
                                         --   alert carried. A transport grade
                                         --   only; it implies no operational
                                         --   response (see health_report.md's
                                         --   Constraints)
    dispatched_at    VARCHAR NOT NULL,   -- 'YYYYMMDD_HHMMSSmmm', queue entry time
    completed_at     VARCHAR,            -- NULL until the background sender
                                         --   resolves. Still written for a
                                         --   drain-timeout drop, because the DB
                                         --   connection deliberately outlives the
                                         --   drain (see health_report.md's
                                         --   shutdown order)
    outcome          VARCHAR,            -- 'sent' | 'failed' | 'suppressed' |
                                         --   'dropped_queue' | 'dropped_drain'.
                                         --   NULL only while genuinely in flight
    suppressed_count INTEGER,            -- alerts suppressed on this
                                         --   (alert_key, channel) since its last
                                         --   send. Legitimately DIFFERS between
                                         --   channels for the same alert_key,
                                         --   since each channel carries its own
                                         --   timeout — not an inconsistency
    event_ids        VARCHAR,            -- JSON array of health_events.event_id.
                                         --   Forward: which events this alert
                                         --   reported. Reverse (a scan of this
                                         --   column): which alert, if any, ever
                                         --   carried a given event
    PRIMARY KEY (alert_id)
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
