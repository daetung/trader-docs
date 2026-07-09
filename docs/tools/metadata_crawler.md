# Tool: Metadata Crawler + Daily Data Collector

**File:** `tools/collect_daily.py`
**Standalone CLI tool — runs daily via cron or task scheduler**

---

## Role

Five responsibilities combined into one daily-run tool, split across two
schedule points (see "Dual Schedule" below — evening full run, plus a
lightweight premarket corporate-events refresh):

1. **Metadata crawling** — fetch and upsert stock metadata (sector, market cap,
   shares outstanding, 52w high/low, avg volume) for all active tickers
2. **Corporate events crawling** — fetch and upsert split/reverse-split/dividend
   history for all active tickers into `corporate_events` table
3. **New data ingestion** — ingest newly collected JSON files into DuckDB
4. **Calendar and coverage update** — incrementally update `trading_calendar`
   and `ticker_data_coverage` for newly ingested dates
5. **Session stats update** — compute and store `precomputed_session_stats`
   for newly ingested dates (REFERENCE_SESSION baselines)
6. **Tick bar aggregates update** — compute and store `tick_bar_aggregates`
   for newly ingested dates (see db_schema.md, utils.compute_tick_bar_aggregates())
7. **Ticker rename detection + fundamentals collection** — premarket CIK-map-based
   rename detection (with evening self-correction) and incremental
   fundamentals_quarterly collection (see new sections below)

---

## Data Sources for Metadata

### Primary: yfinance
```python
import yfinance as yf
info = yf.Ticker("AAPL").info
# Fields: sector, marketCap, sharesOutstanding,
#         fiftyTwoWeekHigh, fiftyTwoWeekLow, averageVolume
```

### Fallback: Financial Modeling Prep (FMP)
```python
GET https://financialmodelingprep.com/api/v3/profile/{ticker}?apikey={key}
```
- Free tier: 250 requests/day → use only for yfinance failures
- API key stored in `configs/secrets.yaml` (gitignored)

### Shares Outstanding fallback: SEC EDGAR
```python
GET https://data.sec.gov/submissions/CIK{cik}.json
```
- No API key required; rate limit 10 req/s
- CIK lookup table cached locally in `configs/cik_map.json`

---

## Data Sources for Corporate Events

### yfinance splits and dividends history
```python
import yfinance as yf
splits = yf.Ticker("AAPL").splits
# Returns: pd.Series with DatetimeIndex → ratio (float)
# Example: 2024-06-10 → 4.0 (4:1 split)
# Reverse splits: ratio < 1.0 (e.g. 0.1 for 1:10 reverse split)

dividends = yf.Ticker("AAPL").dividends
# Returns: pd.Series with DatetimeIndex → per-share cash amount (float)
# Example: 2024-08-09 → 0.25
```

Corporate events collection runs alongside metadata crawling (same ticker batch).
Upserts into `corporate_events` table (INSERT OR IGNORE).

```python
def crawl_corporate_events(ticker: str, db_conn) -> int:
    """
    Fetch split, reverse-split, and dividend history from yfinance and upsert
    into corporate_events. Determines event_type from split ratio:
    >1.0 → 'split', <1.0 → 'reverse_split'. Dividends stored separately with
    event_type='dividend' and value=per-share cash amount.
    Returns: number of new rows inserted (splits + dividends combined).
    """
    inserted = 0

    splits = yf.Ticker(ticker).splits
    for date, ratio in splits.items():
        event_type = "split" if ratio > 1.0 else "reverse_split"
        db_conn.execute("""
            INSERT OR IGNORE INTO corporate_events (ticker, event_date, event_type, value)
            VALUES (?, ?, ?, ?)
        """, [ticker, date.strftime("%Y%m%d"), event_type, float(ratio)])
        inserted += 1

    dividends = yf.Ticker(ticker).dividends
    for date, amount in dividends.items():
        db_conn.execute("""
            INSERT OR IGNORE INTO corporate_events (ticker, event_date, event_type, value)
            VALUES (?, ?, 'dividend', ?)
        """, [ticker, date.strftime("%Y%m%d"), float(amount)])
        inserted += 1

    return inserted
```

Notes:
- Both splits and dividends collected in the same function call (single
  yfinance ticker session per invocation) — see Items 1+2+3 design; dividend
  event_type is no longer reserved-only, it is actively collected
- yfinance split/dividend history is comprehensive for US equities; no
  fallback needed for either
- Safe to re-run (INSERT OR IGNORE)

---

## Additional Data Sources

### NYSE Trading Halts
```
Source: https://www.nyse.com/trade-halt-current
        https://www.nyse.com/trade-halt-history  (historical)

Fields: ticker, date, halt_start, halt_end, reason_code
Target table: trading_halts
Note: Halts occur across all time periods — no time filter applied.
```

```python
def crawl_nyse_halts(date: str, db_conn) -> int:
    """
    Fetch halts for given date from NYSE.
    Upsert into trading_halts table.
    Returns: number of rows inserted.
    """
    ...
```

### US Market Holidays
```
Source: pandas_market_calendars library (NYSE calendar)

Method:
    import pandas_market_calendars as mcal
    nyse = mcal.get_calendar("NYSE")
    holidays = nyse.holidays().holidays

Frequency: once per year (or on demand)
Target table: us_holidays
Populate 3 years forward on first run; refresh annually.
```

---

## New JSON Ingestion (Daily)

Reuses migration logic from `migrate_json_to_duckdb.py`.
Targets only today's date directory or zip by default.

```python
from tools.migrate_json_to_duckdb import migrate_date
migrate_date(data_root, db_conn, date=today)
```

Time range filter: `hour >= '040000' AND hour <= '200000'` (no session filtering).

---

## Calendar and Coverage Update (Daily)

After each ingestion, incrementally update calendar tables via shared utils functions:

```python
from utils import populate_trading_calendar, populate_ticker_coverage

populate_trading_calendar(db_conn=con, date_range=[today])
populate_ticker_coverage(db_conn=con, dates=[today])
```

These functions are safe to call multiple times (upsert).

---

## Session Stats Update (Daily)

After calendar and coverage update, compute REFERENCE_SESSION baselines for
the next trading day. This ensures session_stats are available before market open.

```python
from utils import populate_precomputed_session_stats

# Compute baselines for tomorrow (as_of_date = next trading day)
# so that LiveModeRunner can query them at session start.
next_trading_day = get_next_trading_day(today, db_conn)

populate_precomputed_session_stats(
    db_conn=con,
    dates=[next_trading_day],   # as_of_date for which we need baselines
    n_sessions=20,
)
```

`populate_precomputed_session_stats()`:
- Computes per-bar averages for `as_of_date = next_trading_day` using
  the prior 20 sessions' data (now available since today's data just ingested)
- Halt handling applied per metric type (see utils.md for details)
- Corporate-event adjustment applied per metric type before aggregation,
  using `corporate_events` as populated by this same evening run (see
  utils.md populate_precomputed_session_stats() section D.)
- Safe to re-run (INSERT OR IGNORE)
- Runs after ingestion — ensures today's data is included in the baseline

---

## Tick Bar Aggregates Update (Daily)

Runs after ingestion and before Session Stats Update, populating
`tick_bar_aggregates` (see db_schema.md) for today's newly-ingested data —
this ordering matters: `populate_precomputed_session_stats()` sources its
tick-derived baselines (`buy_ratio_baseline`, `intra_tpm_baseline`,
`intra_avg_vol_per_tick_baseline`) from `tick_bar_aggregates`, so this step
must complete first within the same evening run for tomorrow's baselines to
reflect today's data.

```python
from utils import compute_tick_bar_aggregates

for ticker in todays_ingested_tickers:
    compute_tick_bar_aggregates(ticker, today, today, all_registered_indicators, db_conn)
    # INSERT OR IGNORE into tick_bar_aggregates
```

Skipped if `--skip-tick-bar-aggregates` is passed (session stats step still
runs — falls back to Tier 2/3 on-the-fly resolution per utils.md).

---

## Premarket Ticker Rename Detection

Runs at premarket (same schedule slot as the corporate-events-only refresh),
**before** `LiveModeRunner.start_session()` — a rename must be registered
before that day's Eager Pool bar loading, or the renamed ticker's history
lookback is silently truncated for that entire day.

```python
def detect_rename_candidates(db_conn) -> list[dict]:
    """
    1. Fetch SEC company_tickers.json → upsert ticker_cik_map
    2. Find (cik, ticker_old) -> (cik, ticker_new) mapping changes where
       ticker_old has existing ohlcv_1min history (ticker_new's own history
       is not required — it doesn't exist yet at premarket)
    3. Unambiguous matches: INSERT OR IGNORE INTO ticker_history
       (effective_date = today, a best-effort estimate; self-corrected this
       evening — see below)
    4. Ambiguous / CIK-unmatched: log to tools/rename_candidates.log
    """
```

```bash
# Premarket schedule addition (~04:00 ET, alongside corporate-events-only):
python tools/collect_daily.py --db-path data/market.duckdb --ticker-rename-only
```

## Evening Ticker Rename Self-Correction

Runs during the evening full run, after ingestion:

```python
def self_correct_rename_effective_dates(db_conn, today) -> None:
    """
    For ticker_history rows with effective_date within the last 5 trading
    days: compare registered effective_date against
    MIN(date) FROM ohlcv_1min WHERE ticker = ticker_new.
    If different: UPDATE ticker_history SET effective_date = actual.
    If ohlcv_1min still has no rows for ticker_new after grace_period
    (5 trading days): re-log to rename_candidates.log as a likely false
    positive from the premarket detector.
    """
```

Idempotent — safe to run every evening regardless of whether any rename is
pending correction.

---

## Fundamentals Incremental Collection (Daily)

```python
def collect_fundamentals_incremental(db_conn) -> int:
    """
    Reuses ticker_cik_map from the same day's premarket refresh (no second
    company_tickers.json fetch). For each ticker with a CIK, fetch SEC EDGAR
    submissions feed; if new filings exist since the last stored filed_date
    for that ticker, fetch companyfacts and upsert into fundamentals_quarterly
    via resolve_xbrl_tag_value() (see utils.md) per configs/xbrl_tag_map.json
    tag priority. Skipped if --skip-fundamentals.
    """


---

## Dual Schedule: Evening Full Run + Premarket Corporate-Events Refresh

The evening run (17:00 ET, below) computes `precomputed_session_stats` for
tomorrow using corporate_events as crawled at that time. This ordering must
not change — moving the full crawl to premarket would mean session_stats
baselines are computed BEFORE that day's own corporate_events refresh,
reintroducing the staleness problem baselines are meant to avoid.

Instead, a second, lightweight, corporate-events-only crawl runs at premarket,
supplementing (not replacing) the evening run — this improves freshness for
same-day consumers that don't depend on session_stats' ordering constraint:
`EntryPointDetector` filter E (condition E, turnover), `gap_percentile()`'s
`dividend_amount` lookup, and `04_feature_extractor.md`'s per-date meta
resolution all only need "today's own corporate_events to be as fresh as
possible" — they don't recompute any aggregate that would be invalidated by
a second crawl.

Ticker rename detection (see "Premarket Ticker Rename Detection" below)
shares this same premarket slot and the same rationale — it must complete
before `LiveModeRunner.start_session()`'s Eager Pool bar loading, for the
same reason the corporate-events refresh must: a rename or split discovered
only at evening (after that day's session already ran) is one day too late
for that day's own bar loading, even though it's still in time for the
following day's `precomputed_session_stats` (evening) and future sessions.

```bash
# Corporate-events-only refresh, run twice before/around market open:
python tools/collect_daily.py \
    --db-path data/market.duckdb \
    --corporate-events-only

# Ticker rename detection, same premarket window (see "Premarket Ticker
# Rename Detection" below):
python tools/collect_daily.py \
    --db-path data/market.duckdb \
    --ticker-rename-only
```

Recommended timing: once well before premarket bar accumulation begins
(e.g. ~04:00 ET), and once again just before regular session open (~09:25 ET)
as a re-check. `crawl_corporate_events()`'s `INSERT OR IGNORE` makes running
it twice (or more) safe — a second run either finds nothing new (no-op) or
catches an event the first run missed.

**This is risk mitigation, not a guarantee.** Split/dividend effective dates
are always before-market-open by convention (see `db_schema.md`
`corporate_events`), but *when yfinance's own data reflects that event* is
outside this tool's control — a genuinely late vendor update can still be
missed by both premarket crawls. This is a known, accepted limitation of
relying on yfinance as the sole corporate-events source, not a defect in the
scheduling design; it applies equally to `EntryPointDetector` filter E's
point-in-time `shares_outstanding` and to `gap_percentile()`'s
`dividend_amount` for the affected ticker/date only.

---

## CLI Usage

```bash
# Full daily run: metadata + corporate events + ingest + calendar update + session stats
python tools/collect_daily.py \
    --db-path data/market.duckdb \
    --data-root /path/to/json/data \
    --date today

# Metadata only (includes corporate events)
python tools/collect_daily.py \
    --db-path data/market.duckdb \
    --metadata-only

# Ingest only (includes calendar update and session stats)
python tools/collect_daily.py \
    --db-path data/market.duckdb \
    --data-root /path/to/json/data \
    --ingest-only \
    --date 20250715

# Skip session stats computation
python tools/collect_daily.py \
    --db-path data/market.duckdb \
    --data-root /path/to/json/data \
    --date today \
    --skip-session-stats

# Skip tick bar aggregates / fundamentals collection independently
python tools/collect_daily.py --db-path data/market.duckdb --date today --skip-tick-bar-aggregates
python tools/collect_daily.py --db-path data/market.duckdb --date today --skip-fundamentals

# Premarket-only: ticker_cik_map refresh + rename candidate detection
# (see "Premarket Ticker Rename Detection" below)
python tools/collect_daily.py --db-path data/market.duckdb --ticker-rename-only

# List unresolved rename candidates for manual review
python tools/collect_daily.py --list-rename-candidates

# Manually register a rename (SEC-unmatched tickers only)
python tools/collect_daily.py --register-rename OLD_TICKER NEW_TICKER 20260715

# Refresh metadata for specific tickers
python tools/collect_daily.py \
    --db-path data/market.duckdb \
    --tickers AAPL,MSFT,NVDA \
    --metadata-only

# Backfill metadata for all tickers
python tools/collect_daily.py \
    --db-path data/market.duckdb \
    --backfill-meta

# Corporate events only (premarket refresh — see Dual Schedule above)
python tools/collect_daily.py \
    --db-path data/market.duckdb \
    --corporate-events-only
```

---

## Metadata Fetch Logic

```
1. Load ticker list from DuckDB:
   SELECT DISTINCT ticker FROM ohlcv_1min

2. For each ticker (batched, 100 at a time):
   a. Fetch metadata from yfinance
   b. If any required field is None → retry with FMP
   c. If shares_outstanding still None → query SEC EDGAR
   d. Upsert into stock_meta (INSERT OR IGNORE, keyed on (ticker, date=today))
      — see db_schema.md V-1 fix: stock_meta is now (ticker, date)-keyed,
      append-only per date, not a single overwritten row per ticker.
      Partial field success is preserved at the row level: if a run only
      resolves some fields for today (e.g. sector but not market_cap), the
      row is still inserted with NULL for the unresolved fields — a
      subsequent same-day re-run does NOT overwrite already-resolved fields
      (INSERT OR IGNORE is row-level, keyed on (ticker,date); to update
      individual NULL fields within an existing same-day row, an explicit
      UPDATE ... WHERE field IS NULL is used instead of a blind re-insert)
   e. Fetch split/dividend history from yfinance → upsert into corporate_events
      via crawl_corporate_events() (INSERT OR IGNORE — safe to re-run; no
      fallback needed for either)

3. Log: fetched / failed / unchanged
```

Required fields: `sector`, `market_cap`, `shares_outstanding`, `price_52h`, `price_52l`, `avg_volume`

Failed tickers logged to `tools/metadata_missing.log`. A field-level failure
(some but not all required fields resolved) does not block ingestion or
corporate events collection — see Constraints.

---

## Output / Logging

```
Daily Collection — 20250715
  [Metadata]
    Tickers processed   : 11,847
    Updated             : 11,820
    Failed (all srcs)   : 27     → tools/metadata_missing.log
    Duration            : 6m 14s

  [Corporate Events]
    Split events upserted : 3
    Reverse splits        : 0
    Duration              : included in metadata timing

  [Ingestion — 20250715]
    Files found       : 23,694
    Imported          : 23,694
    Skipped           : 0
    Failed            : 0
    Rows inserted     : ohlcv_1min=4,621,260 | tick_10=21,048,000
    Duration          : 3m 41s

  [Calendar Update]
    trading_calendar    : 1 date upserted
    ticker_data_coverage: 11,847 rows upserted
    Duration            : 0m 12s

  [Session Stats — next trading day: 20250716]
    precomputed_session_stats: 11,847 tickers × 390 bars × 8 metrics computed
    Duration            : 1m 48s
```

---

## Cron / Scheduler Setup

```bash
# Run after market close daily (weekdays only), 17:00 ET
# Full run: metadata + corporate events + ingest + calendar update + session stats
0 17 * * 1-5 cd /path/to/stock-scalping && \
    source .venv/bin/activate && \
    python tools/collect_daily.py \
        --db-path data/market.duckdb \
        --data-root /path/to/json/data \
        --date today >> logs/collect_daily.log 2>&1

# Premarket corporate-events-only refresh (weekdays only), ~04:00 ET
# See "Dual Schedule" above — supplements the evening run, does not replace it
0 4 * * 1-5 cd /path/to/stock-scalping && \
    source .venv/bin/activate && \
    python tools/collect_daily.py \
        --db-path data/market.duckdb \
        --corporate-events-only >> logs/collect_daily_premarket.log 2>&1

# Second premarket re-check, just before regular session open, ~09:25 ET
25 9 * * 1-5 cd /path/to/stock-scalping && \
    source .venv/bin/activate && \
    python tools/collect_daily.py \
        --db-path data/market.duckdb \
        --corporate-events-only >> logs/collect_daily_premarket.log 2>&1
```

---

## secrets.yaml (gitignored)

```yaml
# configs/secrets.yaml — DO NOT COMMIT
fmp_api_key: "your_key_here"
```

---

## Constraints

- `secrets.yaml` must be in `.gitignore` — never commit API keys
- yfinance is the primary metadata source; FMP and EDGAR are fallbacks only
- Corporate events (splits, reverse splits, dividends) fetched from yfinance
  only — no fallback needed for any of the three
- `crawl_corporate_events()` fetches both splits and dividends in one call —
  they are not separate functions, to avoid two yfinance ticker sessions
- Metadata upsert is `INSERT OR IGNORE` keyed on `(ticker, date=today)` — not
  `INSERT OR REPLACE` keyed on `ticker` alone (see db_schema.md V-1 fix);
  `updated_at` still set to today for the freshness of whichever fields
  resolved this run
- corporate_events uses INSERT OR IGNORE — same as before, unaffected by the
  stock_meta schema change
- Ingestion logic delegates to `migrate_json_to_duckdb.py` — no duplicated logic
- Calendar/coverage update always runs after ingestion — not optional
- Session stats update runs after calendar/coverage update — not optional unless --skip-session-stats
- Session stats update always uses the EVENING run's corporate_events state —
  the ordering (crawl → ingest → calendar/coverage → session stats) within a
  single evening run must not change; the premarket refresh is intentionally
  a separate, later, supplementary crawl that session_stats does NOT re-read
- `populate_trading_calendar()`, `populate_ticker_coverage()`, and `populate_precomputed_session_stats()` sourced from `utils.py`
- `--corporate-events-only` runs only `crawl_corporate_events()` for all
  tickers — no metadata fields, no ingestion, no calendar/session-stats steps
- Must be safe to run multiple times on the same date (idempotent) — including
  multiple `--corporate-events-only` runs on the same day
- Failed metadata tickers do not block ingestion — two steps are independent
- No session time filter on ingested data — all periods (040000~200000) stored
- US stock splits and ex-dividend adjustments always take effect before market
  open — no intra-session event handling needed
- Vendor update latency (yfinance not yet reflecting a same-day corporate
  event by the time either premarket crawl runs) is a known, accepted
  limitation — not resolvable by adjusting crawl timing further; downstream
  consumers (filter E, gap_percentile dividend_amount, meta resolution) treat
  this the same as any other missing-data case, not as an error
