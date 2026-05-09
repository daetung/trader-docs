# Tool: Metadata Crawler + Daily Data Collector

**File:** `tools/collect_daily.py`
**Standalone CLI tool — runs daily via cron or task scheduler**

---

## Role

Two responsibilities combined into one daily-run tool:

1. **Metadata crawling** — fetch and upsert stock metadata (sector, market cap,
   shares outstanding, 52w high/low, avg volume) for all active tickers
2. **New data ingestion** — ingest newly collected JSON files (from existing
   collection scripts) into DuckDB in real-time format

These are combined because both run on the same daily schedule and share
the ticker list and DuckDB connection.

---

## Data Sources for Metadata

### Primary: yfinance
```python
import yfinance as yf
info = yf.Ticker("AAPL").info
# Fields: sector, marketCap, sharesOutstanding,
#         fiftyTwoWeekHigh, fiftyTwoWeekLow, averageVolume
```
- No API key required
- Bulk fetch via `yf.download()` for speed
- Rate limit: ~2,000 tickers/minute with batch requests

### Fallback: Financial Modeling Prep (FMP)
```python
# For tickers where yfinance returns None or incomplete data
GET https://financialmodelingprep.com/api/v3/profile/{ticker}?apikey={key}
# Fields: sector, mktCap, sharesOutstanding, range (52w)
```
- Free tier: 250 requests/day → use only for yfinance failures
- API key stored in `configs/secrets.yaml` (gitignored)

### Shares Outstanding fallback: SEC EDGAR
```python
# For maximum accuracy on shares_outstanding
GET https://data.sec.gov/submissions/CIK{cik}.json
# No API key required; rate limit 10 req/s
```
- Used when both yfinance and FMP return unreliable values
- CIK lookup table cached locally in `configs/cik_map.json`

---

## Additional Data Sources

### NYSE Trading Halts
```
Source: https://www.nyse.com/trade-halt-current
        https://www.nyse.com/trade-halt-history  (historical)

Crawl method:
    GET request → parse HTML table or CSV download
    Fields: ticker, date, halt_start, halt_end, reason_code

Frequency: daily (after market close)
Target table: trading_halts

reason_code values (common):
    T1   → News Pending
    T6   → Extraordinary Market Activity
    LUDP → Limit Up-Limit Down Pause
    M    → Market-wide circuit breaker
```

```python
def crawl_nyse_halts(date: str, db_conn) -> int:
    """
    Fetch halts for given date from NYSE.
    Upsert into trading_halts table.
    Returns: number of rows inserted.
    """
    url = f"https://www.nyse.com/trade-halt-history"
    # POST with date filter or parse CSV
    ...
```

### US Market Holidays
```
Source: pandas_market_calendars library (NYSE calendar)
        OR static crawl from https://www.nyse.com/markets/hours-calendars

Method:
    import pandas_market_calendars as mcal
    nyse = mcal.get_calendar("NYSE")
    holidays = nyse.holidays().holidays  # DatetimeIndex

Frequency: once per year (or on demand)
Target table: us_holidays
Populate 3 years forward on first run; refresh annually.
```

```python
def populate_us_holidays(years: list[int], db_conn) -> int:
    """
    Populate us_holidays table for given years.
    Skip years already fully populated.
    Returns: number of rows inserted.
    """
    ...
```

```bash
# Full daily run: metadata + ingest new JSON files
python tools/collect_daily.py \
    --db-path data/market.duckdb \
    --data-root /path/to/json/data \
    --date today

# Metadata only (no new JSON ingestion)
python tools/collect_daily.py \
    --db-path data/market.duckdb \
    --metadata-only

# Ingest only (no metadata fetch)
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

# Backfill metadata for all tickers in stock_meta (no date filter)
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

Required fields (all must be non-null for a ticker to be usable):
- `sector`
- `market_cap`
- `shares_outstanding`
- `price_52h`, `price_52l`
- `avg_volume`

Tickers with missing required fields after all sources are logged to
`tools/metadata_missing.log` — these tickers will be excluded from
feature extraction (MetaFeatureExtractor returns NaN for missing meta).

---

## New JSON Ingestion (Daily)

Reuses migration logic from `migrate_json_to_duckdb.py`.
Targets only today's date directory or zip by default.

```python
from tools.migrate_json_to_duckdb import migrate_date
migrate_date(data_root, db_conn, date=today)
```

Future data collected by existing scripts can be directed to:
- A new date directory: `{data_root}/{DATE}/`
- Or a new zip: `{data_root}/{DATE}.zip`

The tool auto-detects whichever format is present.

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
- Metadata upsert uses `updated_at = today` — always overwrites prior values
- Ingestion logic delegates to `migrate_json_to_duckdb.py` — no duplicated logic
- Must be safe to run multiple times on the same date (idempotent)
- Failed metadata tickers do not block ingestion — the two steps are independent
