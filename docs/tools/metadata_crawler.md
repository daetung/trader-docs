# Tool: Metadata Crawler + Daily Data Collector

**File:** `tools/collect_daily.py`
**Standalone CLI tool — runs daily via cron or task scheduler**

---

## Role

Three responsibilities combined into one daily-run tool:

1. **Metadata crawling** — fetch and upsert stock metadata (sector, market cap,
   shares outstanding, 52w high/low, avg volume) for all active tickers
2. **New data ingestion** — ingest newly collected JSON files into DuckDB
3. **Calendar and coverage update** — incrementally update `trading_calendar`
   and `ticker_data_coverage` for newly ingested dates

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

## CLI Usage

```bash
# Full daily run: metadata + ingest + calendar update
python tools/collect_daily.py \
    --db-path data/market.duckdb \
    --data-root /path/to/json/data \
    --date today

# Metadata only
python tools/collect_daily.py \
    --db-path data/market.duckdb \
    --metadata-only

# Ingest only (includes calendar update)
python tools/collect_daily.py \
    --db-path data/market.duckdb \
    --data-root /path/to/json/data \
    --ingest-only \
    --date 20250715

# Refresh metadata for specific tickers
python tools/collect_daily.py \
    --db-path data/market.duckdb \
    --tickers AAPL,MSFT,NVDA \
    --metadata-only

# Backfill metadata for all tickers
python tools/collect_daily.py \
    --db-path data/market.duckdb \
    --backfill-meta
```

---

## Metadata Fetch Logic

```
1. Load ticker list from DuckDB:
   SELECT DISTINCT ticker FROM ohlcv_1min

2. For each ticker (batched, 100 at a time):
   a. Fetch from yfinance
   b. If any required field is None → retry with FMP
   c. If shares_outstanding still None → query SEC EDGAR
   d. Upsert into stock_meta (INSERT OR REPLACE)

3. Log: fetched / failed / unchanged
```

Required fields: `sector`, `market_cap`, `shares_outstanding`, `price_52h`, `price_52l`, `avg_volume`

Failed tickers logged to `tools/metadata_missing.log`.

---

## Output / Logging

```
Daily Collection — 20250715
  [Metadata]
    Tickers processed : 11,847
    Updated           : 11,820
    Failed (all srcs) : 27     → tools/metadata_missing.log
    Duration          : 6m 14s

  [Ingestion — 20250715]
    Files found       : 23,694
    Imported          : 23,694
    Skipped           : 0
    Failed            : 0
    Rows inserted     : ohlcv_1min=4,621,260 | tick_10=21,048,000
    Duration          : 3m 41s

  [Calendar Update]
    trading_calendar   : 1 date upserted
    ticker_data_coverage: 11,847 rows upserted
    Duration           : 0m 12s
```

---

## Cron / Scheduler Setup

```bash
# Run after market close daily (weekdays only), 17:00 ET
0 17 * * 1-5 cd /path/to/stock-scalping && \
    source .venv/bin/activate && \
    python tools/collect_daily.py \
        --db-path data/market.duckdb \
        --data-root /path/to/json/data \
        --date today >> logs/collect_daily.log 2>&1
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
- yfinance is the primary source; FMP and EDGAR are fallbacks only
- Metadata upsert uses `updated_at = today`
- Ingestion logic delegates to `migrate_json_to_duckdb.py` — no duplicated logic
- Calendar/coverage update always runs after ingestion — not optional
- `populate_trading_calendar()` and `populate_ticker_coverage()` sourced from `utils.py`
- Must be safe to run multiple times on the same date (idempotent)
- Failed metadata tickers do not block ingestion — two steps are independent
- No session time filter on ingested data — all periods (040000~200000) stored
