# Tool: JSON → DuckDB Migration

**File:** `tools/migrate_json_to_duckdb.py`
**Standalone CLI tool — no pipeline dependencies required**

---

## Role

One-time (and incremental) migration of existing JSON files into DuckDB.
Supports both directory and zip formats. Skips already-imported data.
After migration, initializes `trading_calendar`, `ticker_data_coverage`,
and `precomputed_session_stats` tables.

---

## Input File Convention

```
{data_root}/
├── {DATE}/
│   ├── {TICKER}_{DATE}_min.json
│   └── {TICKER}_{DATE}_tick.json
└── {DATE}.zip                      # flat zip, no sub-directory
    ├── {TICKER}_{DATE}_min.json
    └── {TICKER}_{DATE}_tick.json
```

Filename parsing:
```python
pattern = r"^(?P<ticker>.+)_(?P<date>\d{8})_(?P<resolution>min|tick)\.json$"
```

---

## CLI Usage

```bash
# Migrate all data under a root directory
python tools/migrate_json_to_duckdb.py \
    --data-root /path/to/json/data \
    --db-path data/market.duckdb

# Migrate a specific date only
python tools/migrate_json_to_duckdb.py \
    --data-root /path/to/json/data \
    --db-path data/market.duckdb \
    --date 20250714

# Dry run — count files and estimate rows without writing
python tools/migrate_json_to_duckdb.py \
    --data-root /path/to/json/data \
    --db-path data/market.duckdb \
    --dry-run

# Show progress bar
python tools/migrate_json_to_duckdb.py \
    --data-root /path/to/json/data \
    --db-path data/market.duckdb \
    --verbose

# Skip session stats computation (for large migrations where stats will be run separately)
python tools/migrate_json_to_duckdb.py \
    --data-root /path/to/json/data \
    --db-path data/market.duckdb \
    --skip-session-stats
```

---

## Processing Logic

```
1. Scan data_root for all .json files and .zip files
2. Parse (ticker, date, resolution) from filename
3. For each (ticker, date, resolution):
     a. Check DuckDB: if (ticker, date) already exists in target table → SKIP
     b. Load JSON → pd.DataFrame
     c. Cast all fields to correct types (strings → DOUBLE / BIGINT)
     d. Filter to valid trading hours: hour >= '040000' AND hour <= '200000'
        (pre-market, regular session, after-market; overnight excluded)
     e. For tick data: assign seq_id within each (ticker, date, hour) group
     f. Bulk insert into DuckDB (INSERT OR IGNORE)
4. Log: files processed / skipped / failed

5. Post-migration: initialize calendar and coverage tables
     a. populate_trading_calendar(db_conn, all_ingested_dates)
     b. populate_ticker_coverage(db_conn, all_ingested_dates)
     (both functions sourced from utils.py)

6. Post-migration: compute REFERENCE_SESSION baselines
     a. populate_precomputed_session_stats(db_conn, all_ingested_dates,
                                           n_sessions=20)
     (function sourced from utils.py)
     Computes per-bar prior-session averages for all tickers and dates.
     Includes: rvol_baseline, rel_dvol_baseline, intra_vol_baseline,
               intra_return_baseline, intra_tpm_baseline, buy_ratio_baseline,
               gap_pct_mean, gap_pct_std.
     buy_ratio_baseline requires tick_10 data (lee_ready classification).
     Safe to re-run (INSERT OR IGNORE).
     Note: Step 6 is skipped if --skip-session-stats flag is passed.
```

**Zip handling:**
```python
import zipfile, tempfile
with zipfile.ZipFile(zip_path) as zf:
    for name in zf.namelist():
        with tempfile.NamedTemporaryFile() as tmp:
            tmp.write(zf.read(name))
            process_json_file(tmp.name)
```

---

## seq_id Assignment (tick data)

```python
df["seq_id"] = df.groupby(["ticker", "date", "hour"]).cumcount()
```

---

## Post-Migration: Calendar and Coverage Initialization

After all files are ingested, two tables are populated via shared utils functions:

```python
from utils import populate_trading_calendar, populate_ticker_coverage

ingested_dates = con.execute(
    "SELECT DISTINCT date FROM ohlcv_1min ORDER BY date"
).df()["date"].tolist()

populate_trading_calendar(db_conn=con, date_range=ingested_dates)
populate_ticker_coverage(db_conn=con, dates=ingested_dates)
```

`populate_trading_calendar()`:
- Uses `pandas_market_calendars` (NYSE) to determine `is_trading_day`, `is_holiday`
- Sets `has_data=True` for dates present in `ohlcv_1min`
- Safe to re-run (upsert)

`populate_ticker_coverage()`:
- Groups `ohlcv_1min` and `tick_10` by (ticker, date) to set `has_1min`, `has_tick`
- Safe to re-run (upsert)

---

## Post-Migration: Precomputed Session Stats

```python
from utils import populate_precomputed_session_stats

populate_precomputed_session_stats(
    db_conn=con,
    dates=ingested_dates,
    n_sessions=20,      # prior session count for baseline
)
```

`populate_precomputed_session_stats()`:
- For each (ticker, as_of_date): computes per-bar average over prior n_sessions
- Metrics computed from ohlcv_1min: rvol_baseline, rel_dvol_baseline,
  intra_vol_baseline, intra_return_baseline
- Metrics computed from tick_10 (lee_ready): buy_ratio_baseline, intra_tpm_baseline
- Day-level metrics (hour='000000'): gap_pct_mean, gap_pct_std
- Stores avg_value, std_value, count per (ticker, as_of_date, hour, metric, n_sessions)
- Delta smoothing is NOT applied here — applied at load time via load_session_stats()
- Safe to re-run (INSERT OR IGNORE)
- First n_sessions dates in the dataset will have count < n_sessions (uses available data)

---

## Output / Logging

```
Migration summary:
  Scanned  : 48,300 files
  Imported : 45,820 files  (ohlcv_1min: 24,100 | tick_10: 21,720)
  Skipped  : 2,480 files   (already in DB)
  Failed   : 0 files
  Duration : 14m 32s
  Rows inserted: ohlcv_1min=9,399,000 | tick_10=43,200,000

Post-migration:
  trading_calendar        : 1,260 dates populated
  ticker_data_coverage    : 11,847 tickers × dates populated
  precomputed_session_stats: 11,847 tickers × 1,240 dates × 390 bars × 8 metrics populated
  Duration (session stats): 8m 14s
```

Failed files are logged to `tools/migration_errors.log` with filename and exception.

---

## Constraints

- Must be runnable independently from the pipeline (no src/ imports required, except utils.py)
- Duplicate skip is at `(ticker, date)` granularity per table — entire file skipped if any row exists
- Zip files: extract to tempdir, process, clean up — do not leave temp files
- Time range filter: `hour >= '040000' AND hour <= '200000'` (no session filtering)
- Must handle malformed JSON files gracefully (log and continue, do not crash)
- Post-migration calendar/coverage population is mandatory — not optional
- Post-migration session stats computation is mandatory unless --skip-session-stats passed
- `populate_trading_calendar()`, `populate_ticker_coverage()`, and `populate_precomputed_session_stats()` sourced from `utils.py`
