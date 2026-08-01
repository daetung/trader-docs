# Tool: JSON → DuckDB Migration

**File:** `tools/migrate_json_to_duckdb.py`
**Standalone CLI tool — no pipeline dependencies required**

---

## Role

One-time (and incremental) migration of existing JSON files into DuckDB.
Supports both directory and zip formats. Skips already-imported data.
After migration, populates `trading_calendar`, `ticker_data_coverage`,
and `precomputed_session_stats` tables.

**Prerequisite — the database and its tables must already exist.** This
tool issues no DDL: its processing steps INSERT into tables and call the
populate helpers, all of which assume those tables are present. Creating
them is `tools/init_db.py`'s job (see `init_db.md`), so the new-database
workflow is `init_db.py` first, then this tool. The two are peers with the
same concurrency assumption below — neither imports the other, and this
tool does not defensively re-create anything.

**Concurrency assumption**: this tool is run manually/rarely, never on an
automated schedule, and is assumed to never run concurrently with a live
trading session or the evening/premarket batch jobs (see P-5's DB
ownership design in live_mode_runner.md / metadata_crawler.md, which this
tool does not participate in — no `batch_runs` marker writes here). Running
it while any of those hold the writer connection will fail with a DuckDB
lock error rather than corrupt data — an operator error to avoid, not a
case this tool detects or works around.

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

5b. Post-migration: crawl corporate events for all ingested tickers
     a. crawl_corporate_events(ticker, db_conn) for each ticker in
        all_ingested_tickers (function sourced from metadata_crawler.md;
        splits, reverse splits, and dividends — see Item 1+2+3 design)
     This step MUST run before Step 6 — populate_precomputed_session_stats()
     reads corporate_events to adjust prior-session baselines for splits and
     dividends (see utils.md), and an empty/incomplete corporate_events table
     at that point would silently produce unadjusted baselines for the
     entire historical backfill, not merely a partial gap.

6. Post-migration: compute REFERENCE_SESSION baselines
     a. populate_precomputed_session_stats(db_conn, all_ingested_dates,
                                           n_sessions=20)
     (function sourced from utils.py)
     Computes per-bar prior-session averages for all tickers and dates.
     Includes: rvol_baseline, rel_dvol_baseline, intra_vol_baseline,
               intra_return_baseline, intra_tpm_baseline, buy_ratio_baseline,
               gap_pct_mean, gap_pct_std.
     Halt handling applied per metric type (see utils.md for details):
       - Cumulative metrics: halt bars contribute volume=0; full-day no-data sessions excluded
       - Per-bar metrics: halt slots excluded per hour slot; count may differ by hour
       - gap_pct: nearest non-halt bar fallback for 155900/093000; session excluded if none
     Corporate-event adjustment (from Step 5b's data) applied per metric type
     before aggregation — see utils.md populate_precomputed_session_stats() D.
     buy_ratio_baseline requires tick_10 data (lee_ready classification).
     Safe to re-run (INSERT OR IGNORE).
     Note: Step 6 is skipped if --skip-session-stats flag is passed (Step 5b
     still runs in that case, since it is cheap and other consumers may need it).
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

## Post-Migration: Corporate Events

Must run before Precomputed Session Stats (below) — see rationale in Processing
Logic Step 5b.

```python
from metadata_crawler import crawl_corporate_events

all_ingested_tickers = con.execute(
    "SELECT DISTINCT ticker FROM ohlcv_1min"
).df()["ticker"].tolist()

for ticker in all_ingested_tickers:
    crawl_corporate_events(ticker, db_conn=con)
```

`crawl_corporate_events()`:
- Fetches split, reverse-split, and dividend history from yfinance
- Writes into `corporate_events` via the shared `upsert_corporate_event()`
  helper (item N) — safe to re-run, not a bare INSERT OR IGNORE
- See `metadata_crawler.md` for the full function definition

**Re-run note:** if `corporate_events` is populated or updated (e.g. a new
split discovered) *after* `precomputed_session_stats` already exists for the
affected ticker/date range, those rows are now stale (unadjusted, or adjusted
against an incomplete corporate_events snapshot). Re-running
`populate_precomputed_session_stats()` alone will NOT correct them —
`INSERT OR IGNORE` skips rows that already exist. Correcting stale rows
requires:
```sql
DELETE FROM precomputed_session_stats WHERE ticker IN (affected_tickers)
```
followed by re-invoking `populate_precomputed_session_stats()` for those
tickers' full date range.

**Ticker rename note:** `ticker_history` (used by `utils.load_ohlcv_with_history()`
for pre-rename bar continuity) is normally populated by an automated
premarket-detection + evening-self-correction mechanism (see
`metadata_crawler.md`) going forward. This migration tool covers historical
backfill only, before that mechanism was live — retroactive rename discovery
for the backfilled period is NOT automated (the CIK-mapping detector only
observes the *current* mapping state, not historical mapping changes), so
any renames within the backfilled date range must still be identified and
registered manually via `metadata_crawler.py --register-rename OLD NEW
EFFECTIVE_DATE` — a known, accepted limitation of the migration path
specifically, not of the ongoing daily mechanism.

**Stale session_stats from a corrected rename:** the same staleness problem
described in the Re-run note above (corporate_events discovered late) applies
when a `ticker_history.effective_date` is self-corrected by the evening
mechanism (see `metadata_crawler.md`) — any `precomputed_session_stats` rows
already computed for that ticker using the pre-correction (wrong)
`effective_date` were built from an incomplete or misaligned bar-loading
window and are now stale. Correct them the same way as the corporate_events
case:
```sql
DELETE FROM precomputed_session_stats WHERE ticker = affected_ticker
```
followed by re-invoking `populate_precomputed_session_stats()` for that
ticker's full date range.

---

## Post-Migration: Tick Bar Aggregates and Fundamentals Backfill

Two additional one-time backfills, independent of each other and of the
corporate-events/session-stats ordering above:

```python
from utils import compute_tick_bar_aggregates

for ticker in all_ingested_tickers:
    for indicator in [...]:  # all 9 registered tick-derived indicators —
                              # see 02_indicator_calculator.md's Tick-Derived
                              # Indicator Scale Sensitivity registry
        compute_tick_bar_aggregates(ticker, start_date, end_date, [indicator], db_conn)
        # INSERT OR IGNORE into tick_bar_aggregates — see db_schema.md
```

```python
# fundamentals_quarterly backfill, via SEC EDGAR XBRL companyfacts,
# keyed through ticker_cik_map (see db_schema.md, Open Item 2 design)
for ticker in all_ingested_tickers:
    fetch_and_upsert_companyfacts(ticker, db_conn)   # rate-limited 10 req/s
```

Both are large one-time backfills (tick_bar_aggregates over the full
migrated tick_10 history; fundamentals over ~12k tickers at 10 req/s) —
expect this step to take substantially longer than the other Post-Migration
steps. Both support `--skip-tick-bar-aggregates` / `--skip-fundamentals`
flags (same convention as `--skip-session-stats`) for deferring to the
first regular `collect_daily.py` evening run instead.

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
- Halt handling applied per metric type (see utils.md — populate_precomputed_session_stats)
- Corporate-event (split/dividend) adjustment applied per metric type before
  aggregation, using the `corporate_events` table populated in the prior step
  (see utils.md — populate_precomputed_session_stats() section D.)
- Stores avg_value, std_value, count per (ticker, as_of_date, hour, metric, n_sessions)
- Delta smoothing is NOT applied here — applied at load time via load_session_stats()
  or build_session_stats_dict()
- Safe to re-run (INSERT OR IGNORE)
- First n_sessions dates in the dataset will have count < n_sessions (uses available data)

---

## Post-Migration Warning: First Live Session Preparation

After migration completes, `precomputed_session_stats` is populated for all
`ingested_dates`. However, the first live trading session requires baselines
for `as_of_date = first_live_date` (the day after the last ingested date).

**This entry is NOT created automatically by migration.**

Before starting the first live session, run one of:

```bash
# Option A: Run collect_daily for the last ingested date (recommended)
# This ingests that date's data (if not yet ingested) and populates
# session_stats for the next trading day.
python tools/collect_daily.py \
    --db-path data/market.duckdb \
    --data-root /path/to/json/data \
    --date {last_ingested_date}

# Option B: Manually populate session_stats for the first live date
python - <<'EOF'
import duckdb
from utils import populate_precomputed_session_stats, get_next_trading_day
con = duckdb.connect("data/market.duckdb")
last_date = con.execute("SELECT MAX(date) FROM ohlcv_1min").fetchone()[0]
first_live_date = get_next_trading_day(last_date, con)
populate_precomputed_session_stats(con, dates=[first_live_date], n_sessions=20)
print(f"Session stats populated for {first_live_date}")
EOF
```

Failure to run this step will cause LiveModeRunner Phase 1 bulk load to return
an empty DataFrame for `as_of_date = first_live_date`, resulting in all
REFERENCE_SESSION indicators returning NaN for the first live session.

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
- Issues no DDL — `tools/init_db.py` (see `init_db.md`) must have created the
  schema first; this tool only INSERTs and populates
- Duplicate skip is at `(ticker, date)` granularity per table — entire file skipped if any row exists
- Zip files: extract to tempdir, process, clean up — do not leave temp files
- Time range filter: `hour >= '040000' AND hour <= '200000'` (no session filtering)
- Must handle malformed JSON files gracefully (log and continue, do not crash)
- Post-migration calendar/coverage population is mandatory — not optional
- Post-migration corporate events crawl (Step 5b) is mandatory — not optional,
  and always runs before session stats computation regardless of
  --skip-session-stats (cheap, and other consumers besides session_stats
  depend on corporate_events being populated)
- Post-migration session stats computation is mandatory unless --skip-session-stats passed
- `populate_trading_calendar()`, `populate_ticker_coverage()`, and `populate_precomputed_session_stats()` sourced from `utils.py`
- `crawl_corporate_events()` sourced from `metadata_crawler.py`, not `utils.py` —
  imported across tool boundaries for this one step

---

## Survivorship Audit (P-11, one-time, manual)

Unlike every other section in this file, NOT part of the automated
migration path — never invoked by `migrate_json_to_duckdb.py` itself, no
`batch_runs` marker, no scheduling. Run manually, once, when someone is
ready to supply the external comparison source below.

**What this checks:** whether the original JSON collection (the files
`migrate_json_to_duckdb.py` imports — see Input File Convention above,
whose own selection methodology predates and is external to this tool) is
missing tickers that were listed at some point during the collection
window but delisted (bankruptcy/liquidation specifically — see rationale
below) before or during collection. Distinct from Dead Position Case B
(`05_labeler.md` / `09_backtest_engine.md`), which already handles a
ticker that IS present and then stops — this checks for tickers absent
from `ohlcv_1min` entirely.

```python
def audit_survivorship_bias(
    db_conn, collection_start_date, collection_end_date, delisting_source
) -> dict:
    """
    1. present_tickers = SELECT DISTINCT ticker FROM ohlcv_1min

    2. expected_tickers = <every ticker listed at any point during
       [collection_start_date, collection_end_date], per delisting_source>
       delisting_source: external data, not yet selected (candidates: SEC
       EDGAR Form 25 filings, a purchased/scraped historical ticker
       roster) — this function's contract (an external source yielding a
       ticker-with-date-range list) is fixed now so the comparison logic
       can be designed against it before the source itself is chosen.

    3. missing = expected_tickers - present_tickers

    4. For each ticker in missing, classify delisting reason if
       delisting_source provides one: bankruptcy/liquidation vs.
       M&A/going-private vs. unknown. Only bankruptcy/liquidation
       plausibly understates dn5 exposure — a ticker acquired at a
       premium didn't crash first, so its absence doesn't bias downside
       risk the way a bankrupt ticker's does. Tickers with unknown reason
       are conservatively counted alongside bankruptcy/liquidation in
       gap_ratio (see below) rather than excluded, for the same
       resolve-ambiguity-toward-the-flagged-side reasoning
       check_corporate_event_anomaly() (P-8) uses.

    5. gap_ratio = |missing tickers classified bankruptcy/liquidation or
                     unknown| / |expected_tickers|

    Returns {gap_ratio, missing_tickers: [...], classified: {...}} — the
    full breakdown, not just the ratio, so a human reviewing the result
    can sanity-check the classification before the decision policy below
    acts on gap_ratio alone.
    """
    ...
```

**Decision policy** (fixed now; does not depend on the actual gap_ratio):

```
gap_ratio < quarantine.survivorship_gap_threshold (config, default 0.02):
    document the measured ratio; no further action.

gap_ratio >= threshold:
    if historical data for the missing tickers is obtainable (a real
    per-source decision at audit time, not knowable now):
        backfill via migrate_json_to_duckdb.py's existing per-ticker
        ingestion path, same as any other JSON backfill.
    else:
        attach a standing caveat to backtest/experiment_log reporting:
        "measured survivorship gap_ratio = X — dn5 exposure may be
        understated by an unquantified amount." Deliberately not
        converted into a numeric haircut applied to reported metrics —
        an invented correction factor would present false precision;
        stating the raw measured gap is more honest than manufacturing
        an adjusted number from it.
```

```yaml
# pipeline_config.yaml addition
quarantine:
  survivorship_gap_threshold: 0.02   # see audit_survivorship_bias() above
```
