# Tool: JSON → DuckDB Migration

**File:** `tools/migrate_json_to_duckdb.py`
**Standalone CLI tool — a per-chunk guest run through guest_supervisor.md's `run_guest`; no pipeline dependencies beyond `src/utils`**

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
same DB access role below — neither imports the other, and this
tool does not defensively re-create anything.

**DB access role**: a per-chunk guest (db_schema.md's DB File Ownership
Windows; db_coordination.md's Unit scope), run through guest_supervisor.md's
`run_guest`. This tool is run manually and rarely, never on an automated
schedule:
- `run_guest migration_tool` enters
  `register('guest', 'migration_tool', trigger)` and hands the body a unit
  opener, each call opening one `guest_unit('migration_tool', None)`, which
  waits for admission without limit;
- the body's units, each step's connection the unit's:
  - ingestion: one unit per file;
  - calendar and coverage initialization: one unit;
  - one unit per ticker: corporate events; tick bar aggregates;
    fundamentals backfill;
  - trading_halts backfill: one unit per date;
  - session stats: one unit per as_of_date;
  - Ingestion-Range Cleanup:
    - one unit enumerating the affected dates — those holding an
      `ohlcv_1min`, `tick_10` or `tick_bar_aggregates` row at an hour outside
      the ingestion range;
    - one unit per affected date: steps 1 and 2 for that date, step 1's
      moves, deletions and `source_changes` rows in one transaction;
    - one unit, run whether or not a date is affected: step 3 over every
      `ticker_data_coverage` row, then `populate_ticker_coverage()` over
      every date holding an `ohlcv_1min` or `tick_10` row; then step 4,
      `populate_trading_calendar()` with date_range None;
  - a read that enumerates a step's chunk keys, its ticker or date list,
    runs in a unit of its own before those chunks;
- the body calls neither `register()` nor `guest_unit()`;
- `tools/migrate_json_to_duckdb.py` run directly hands over to
  `run_guest migration_tool`;
- while an owner waits past `coordination.guest_max_hold_seconds`, the run
  ends by self-termination and is re-run.

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
run_guest migration_tool \
    --data-root /path/to/json/data \
    --db-path data/market.duckdb

# Migrate a specific date only
run_guest migration_tool \
    --data-root /path/to/json/data \
    --db-path data/market.duckdb \
    --date 20250714

# Dry run — count files and estimate rows without writing
run_guest migration_tool \
    --data-root /path/to/json/data \
    --db-path data/market.duckdb \
    --dry-run

# Show progress bar
run_guest migration_tool \
    --data-root /path/to/json/data \
    --db-path data/market.duckdb \
    --verbose

# Skip session stats computation (for large migrations where stats will be run separately)
run_guest migration_tool \
    --data-root /path/to/json/data \
    --db-path data/market.duckdb \
    --skip-session-stats

# Ingestion-Range Cleanup — runs the cleanup alone; another migration
# argument but --verbose given with it is an error at start
run_guest migration_tool \
    --ingestion-range-cleanup \
    --db-path data/market.duckdb

# python tools/migrate_json_to_duckdb.py <args> hands over to run_guest migration_tool <args>
```

---

## Processing Logic

```
1. Scan data_root for all .json files and .zip files
2. Parse (ticker, date, resolution) from filename
3. For each (ticker, date, resolution):
     a. Check DuckDB: if (ticker, date) already exists in target table → SKIP
     b. Load JSON → pd.DataFrame
     c. Parse all fields to their column types (strings → DECIMAL(18,6)
        prices / BIGINT volumes; db_schema.md's Ingestion Rules)
     d. Filter to valid trading hours: hour >= '040000' AND hour < '193000'
        (pre-market, regular session, after-market; the overnight and the
        vendor cut's uneven tail excluded); a tick file's rows at hours in
        [193000, 200000) go to tick_10_tail, in the file's import
        transaction:
        - seq_id as tick_10's, per (ticker, date, hour);
        - the duplicate skip stays on tick_10;
        - INSERT OR IGNORE absorbs a re-import's tail rows
     e. For tick data: assign seq_id within each (ticker, date, hour) group
     f. Bulk insert into DuckDB (INSERT OR IGNORE), appending source_changes
        rows per imported (table, ticker, date) in the import's transaction
        (db_schema.md's Source changes)
4. Log: files processed / skipped / failed

5. Post-migration: initialize calendar and coverage tables
     a. populate_trading_calendar(db_conn, every date from one calendar year
        before the first ingested date through one year past the run day)
     b. populate_ticker_coverage(db_conn, all_ingested_dates)
     (both functions sourced from utils.py)

5b. Post-migration: crawl corporate events for all ingested tickers
     a. crawl_corporate_events(ticker, db_conn) for each ticker in
        all_ingested_tickers (function sourced from tools/collect_daily.py,
        metadata_crawler.md;
        splits, reverse splits, and dividends — see Item 1+2+3 design)
     This step MUST run before Step 6 — populate_precomputed_session_stats()
     reads corporate_events to adjust prior-session baselines for splits and
     dividends (see utils.md), and an empty/incomplete corporate_events table
     at that point would silently produce unadjusted baselines for the
     entire historical backfill, not merely a partial gap.

5c. Post-migration: trading_halts backfill
     a. crawl_nyse_halts(date, db_conn) for every date in all_ingested_dates
        (function sourced from tools/collect_daily.py, metadata_crawler.md)
     A date whose crawl fails is listed as failed and the step continues.
     Runs before Step 6.

6. Post-migration: compute REFERENCE_SESSION baselines
     a. populate_precomputed_session_stats(db_conn, config, all_ingested_dates)
        under the deployment config (pipeline_config.yaml)
     (function sourced from utils.py)
     Computes per-bar prior-session averages for all tickers and dates.
     Includes:
       - rvol_baseline;
       - rel_dvol_baseline;
       - intra_vol_baseline;
       - intra_return_baseline;
       - intra_tpm_baseline;
       - buy_ratio_baseline;
       - intra_avg_vol_per_tick_baseline;
       - gap_pct_mean;
       - gap_pct_std.
     Bar handling per metric type, over the bars the broker supplied (see
     utils.md for details):
       - Cumulative metrics: counted from the session's start over the
         existing bars; full-day no-data sessions excluded
       - Per-minute metrics: a baseline at slot H averages only the sessions
         that have a bar at H — count may differ by slot; intra_return's
         prev_close at H is the close of the last existing bar before H
       - gap_pct: the regular open is the open of the date's first
         regular-session bar (session_phase() 'regular'), the prior close
         the close of the last existing bar at or before last_bar(date); a
         D-k with no regular-session bar is excluded
     Corporate-event adjustment (from Step 5b's data) applied per metric type
     before aggregation — see utils.md populate_precomputed_session_stats() D.
     buy_ratio_baseline requires tick_10 data (lee_ready classification).
     Re-running, per group (db_schema.md's Derived tables):
       - an absent group is computed;
       - a stale group is recomputed;
       - a current group is skipped.
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

calendar_dates = <every date from one calendar year before ingested_dates[0]
                  through one year past the run day — the US-Eastern date of
                  this run>
populate_trading_calendar(db_conn=con, date_range=calendar_dates)
populate_ticker_coverage(db_conn=con, dates=ingested_dates)
```

`populate_trading_calendar()`:
- Uses `pandas_market_calendars` (NYSE) to determine `is_trading_day`, `is_holiday`
- Computes the calendar context columns — `next_closure_last_session`,
  `next_closure_reopen`, `prev_closure_last_session`, `prev_closure_reopen`,
  `next_session_close` — from the library's schedule, writing them on a
  row's insert, on a row whose value is NULL whatever its date, and
  otherwise only while the row's date is after the run day (utils.md);
  rows filled at backfill carry the library's schedule then
- Sets each date of its range's `has_data` to whether `ohlcv_1min` holds that
  date, TRUE to FALSE included
- Safe to re-run (upsert)
- An existing database gains this span's rows by a re-run of step 5a

`populate_ticker_coverage()`:
- Sets, for each (ticker, date) of its dates holding an `ohlcv_1min` or
  `tick_10` row, `has_1min` and `has_tick` each to whether its table holds a
  row
- Safe to re-run (upsert)

---

## Post-Migration: Corporate Events

Must run before Precomputed Session Stats (below) — see rationale in Processing
Logic Step 5b.

```python
from collect_daily import crawl_corporate_events   # tools/collect_daily.py

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

A `corporate_events` or `ticker_history` change after
`precomputed_session_stats` exists turns the groups it reaches stale
(db_schema.md's Derived tables); their producer's next run recomputes them.

**Ticker rename note:** `ticker_history` (used by `utils.load_ohlcv_with_history()`
for pre-rename bar continuity) is normally populated by an automated
premarket-detection + evening-self-correction mechanism (see
`metadata_crawler.md`) going forward. This migration tool covers historical
backfill only, before that mechanism was live — retroactive rename discovery
for the backfilled period is NOT automated (the CIK-mapping detector only
observes the *current* mapping state, not historical mapping changes), so
any renames within the backfilled date range must still be identified and
registered manually via `tools/collect_daily.py --register-rename OLD NEW
EFFECTIVE_DATE` — a known, accepted limitation of the migration path
specifically, not of the ongoing daily mechanism.

---

## Post-Migration: Trading Halts

Must run before Precomputed Session Stats (below) — Processing Logic Step 5c.

```python
from collect_daily import crawl_nyse_halts   # tools/collect_daily.py

for date in ingested_dates:
    crawl_nyse_halts(date, db_conn=con)   # a failed date is listed as failed;
                                          # the step continues
```

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
        compute_tick_bar_aggregates(ticker, start_date, end_date, [indicator],
                                    config["indicators"]["lee_ready"]["momentum_n"],
                                    db_conn)
        # INSERT OR IGNORE into tick_bar_aggregates under the current data
        # version — see db_schema.md
```

```python
# fundamentals_quarterly backfill, via SEC EDGAR XBRL companyfacts,
# keyed through ticker_cik_map (see db_schema.md, Open Item 2 design)
for ticker in all_ingested_tickers:
    fetch_and_upsert_companyfacts(ticker, db_conn)   # rate-limited 10 req/s;
                                                     # each inserted or changed
                                                     # (ticker, filed_date) appends
                                                     # a source_changes row
                                                     # (db_schema.md)
    # Per filing (accn), a metric's values come from the highest-priority
    # tag holding facts in that filing (utils.md's resolve_xbrl_tag_value());
    # each row carries its companyfacts unit. A row with unit NULL is not
    # read; this backfill, re-run, repopulates such rows.

# then fx_rates: the evening_fx_rates stage's function (metadata_crawler.md),
# after the fundamentals backfill — the currencies its units name
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
    config=config,      # the deployment config (pipeline_config.yaml);
                        # indicators.reference_session.n_sessions sets the
                        # prior session count
    dates=ingested_dates,
)
```

`populate_precomputed_session_stats()`:
- For each (ticker, as_of_date): computes per-bar average over prior n_sessions
- Metrics computed from ohlcv_1min: rvol_baseline, rel_dvol_baseline,
  intra_vol_baseline, intra_return_baseline
- Metrics computed from tick_10 (lee_ready):
  - buy_ratio_baseline;
  - intra_tpm_baseline;
  - intra_avg_vol_per_tick_baseline
- Day-level metrics (hour='000000'): gap_pct_mean, gap_pct_std
- Halt handling applied per metric type (see utils.md — populate_precomputed_session_stats)
- Corporate-event (split/dividend) adjustment applied per metric type before
  aggregation, using the `corporate_events` table populated in the prior step
  (see utils.md — populate_precomputed_session_stats() section D.)
- Stores avg_value, std_value, count per (config_hash, ticker, as_of_date, hour, metric)
- Delta smoothing is NOT applied here — applied at load time via load_session_stats()
  or build_session_stats_dict()
- Re-running, per group:
  - an absent group is computed;
  - a stale group is recomputed, its rows replaced;
  - a current group is skipped
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

# Option B: the ingest-only partial run for the last ingested date — a guest
# (metadata_crawler.md's CLI); its session stats update computes the next
# trading day's groups, the first live date's, under the deployment config;
# an already-ingested date's files are skipped
python tools/collect_daily.py \
    --db-path data/market.duckdb \
    --data-root /path/to/json/data \
    --ingest-only \
    --date {last_ingested_date}
```

Failure to run this step leaves LiveModeRunner's Session Lifecycle Step 2
load with no group for `as_of_date = first_live_date`; Health Gate 1a's
session_stats coverage check then blocks participation for the first live
session (live_mode_runner.md).

---

## Output / Logging

```
Migration summary:
  Scanned  : 48,300 files
  Imported : 45,820 files  (ohlcv_1min: 24,100 | tick_10: 21,720)
  Skipped  : 2,480 files   (already in DB)
  Failed   : 0 files
  Duration : 14m 32s
  Rows inserted: ohlcv_1min=9,399,000 | tick_10=43,200,000 | tick_10_tail=21,720

Post-migration:
  trading_calendar        : 1,260 dates populated
  ticker_data_coverage    : 11,847 tickers × dates populated
  precomputed_session_stats: 11,847 tickers × 1,240 dates × the premarket and
                             regular slots of an ordinary day × 9 metrics
                             populated (a capacity estimate)
  Duration (session stats): 8m 14s
```

Failed files are logged to `tools/migration_errors.log` with filename and
exception, and each is recorded beside it through `record_health_event()` with
`direct_write` — finding `collection_gaps`, its ticker, detail {date,
resolution, file, error}.

---

## Constraints

- Must be runnable independently from the pipeline: no `src/` imports beyond
  `src/utils` (utils, and guest_supervisor for the hand-over to `run_guest`)
- Calls neither `register()` nor `guest_unit()` — the body opens its units
  through the unit opener `run_guest` hands it
- Issues no DDL — `tools/init_db.py` (see `init_db.md`) must have created the
  schema first; this tool only INSERTs, deletes (the Ingestion-Range Cleanup)
  and populates
- Duplicate skip is at `(ticker, date)` granularity per table — entire file skipped if any row exists
- Zip files: extract to tempdir, process, clean up — do not leave temp files
- Time range filter: `hour >= '040000' AND hour < '193000'` for `ohlcv_1min`
  and `tick_10` (no session filtering):
  - a date's files run from the preceding evening's 20:00 (overnight) to a
    vendor cut a few seconds past 19:30; the range keeps [04:00, 19:30),
    excluding the overnight and the cut's uneven tail;
  - it is fixed, not per-date, an early-close day ending at 17:00 inside it;
  - a tick file's rows at hours in [193000, 200000) go to `tick_10_tail`
    (step 3d)
- Must handle malformed JSON files gracefully (log and continue, do not crash)
- Post-migration calendar/coverage population is mandatory — not optional
- Post-migration corporate events crawl (Step 5b) is mandatory — not optional,
  and always runs before session stats computation regardless of
  --skip-session-stats (cheap, and other consumers besides session_stats
  depend on corporate_events being populated)
- Post-migration trading_halts backfill (Step 5c) runs before session stats
  computation; a date whose crawl fails is listed as failed and the step
  continues
- Post-migration session stats computation is mandatory unless --skip-session-stats passed
- `populate_trading_calendar()`, `populate_ticker_coverage()`, and `populate_precomputed_session_stats()` sourced from `utils.py`
- `crawl_corporate_events()` and `crawl_nyse_halts()` sourced from
  `tools/collect_daily.py`, not `utils.py` — imported across tool boundaries
  for these steps
- A cleanup run performs the Ingestion-Range Cleanup alone; the processing
  steps, the post-migration steps and their mandatory rules are a migration
  run's

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
       roster) — this function's contract is fixed now so the comparison
       logic can be designed against it before the source itself is
       chosen: its rows carry ticker, date range, delisting reason, CIK, and
       SIC when the source provides it. One source serves this audit and
       the Security Identity Audit (below).

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
        the operator adds a quarantine.data_caveats entry naming the
        measured gap_ratio — "measured survivorship gap_ratio = X — dn5
        exposure may be understated by an unquantified amount."
        Deliberately not converted into a numeric haircut applied to
        reported metrics — an invented correction factor would present
        false precision; stating the raw measured gap is more honest than
        manufacturing an adjusted number from it.
```

---

## Security Identity Audit (one-time, manual)

Run manually, once, right after the Survivorship Audit (P-11), after the
metadata crawl has written `stock_meta` rows. Like it, NOT part of the
automated migration path. It reads the Survivorship Audit's
`delisting_source`.

```python
def audit_security_identity(db_conn, config, delisting_source) -> dict:
    """
    Samples: labeled_samples of the deployment config's preprocess
    config_hash.

    Q1 — the securities whose chain-latest symbol has no ticker_cik_map row,
         each with its last date with bars.

    Q2, mapping and snapshot — groups:
           - the sample's ticker with or without a ticker_cik_map row;
           - the chain-latest symbol with or without a stock_meta snapshot;
         per group: count, share, Case B rate (dead_position_case 'B');
         identity_gap_ratio = the samples without a mapping or without a
         snapshot ÷ all samples.

    Q2, shares none — the samples whose shares_outstanding resolves none
         through estimate_historical_meta() — its closes
         utils.load_session_boundaries()'s, its lookback_days the config's
         indicators.lookback_days — each under its first matching cause:
           - no ticker_cik_map row;
           - a non-common security_type — warrant, unit, right, preferred,
             debt;
           - security_type etf or fund;
           - a sample date on or before the ticker's earliest
             fundamentals_quarterly shares_outstanding filed_date;
           - other, a ticker without such a row included;
         shares_none_ratio = these samples ÷ all samples.

    Q3 — the symbols whose ticker_data_coverage dates with has_1min TRUE
         leave a gap of at least quarantine.reuse_gap_trading_days trading
         days, listed as reuse candidates.

    Unit consistency — per ticker on the crawled dates, the vendor
         shares_outstanding ÷ the fundamentals shares_outstanding as of the
         date, listed when outside [1 ÷ quarantine.unit_ratio_band,
         quarantine.unit_ratio_band].

    Test issues — the ohlcv_1min symbols the symbol directory flags Test
         Issue Y, listed; the audit fetches the symbol directory itself.

    Returns the full breakdown.
    """
    ...
```

**Decision policy** (fixed now):
- the operator adds a `quarantine.data_caveats` entry naming the measured
  ratios, kept until the open item "Point-in-time security identity" is
  applied;
- `identity_gap_ratio` ≥ `quarantine.identity_gap_threshold` with a roster
  carrying CIK obtainable → that open item is worked.

```yaml
# pipeline_config.yaml addition
quarantine:
  survivorship_gap_threshold: 0.02   # see audit_survivorship_bias() above
  identity_gap_threshold: 0.02       # audit_security_identity(); a placeholder
  reuse_gap_trading_days: 60         # its Q3; a placeholder
  unit_ratio_band: 1.5               # its unit consistency; a placeholder
  data_caveats: []                   # operator-written strings, each naming a
                                     # measured data gap; the system never
                                     # writes it. Shown in run_backtest.md's
                                     # Output Format and model_deploy.md's
                                     # Main view
```

---

## Ingestion-Range Cleanup (one-time, manual)

Run manually, once, as a guest, on a database ingested under the former
range; a re-run is harmless. Its units are the DB access role's above.

1. It moves the `tick_10` rows at hours in [193000, 200000) to `tick_10_tail`
   and deletes the other `ohlcv_1min` and `tick_10` rows outside the
   ingestion range, appending a `source_changes` row per (table, ticker,
   date) it removes rows from, in its transaction.
2. It deletes the `tick_bar_aggregates` rows at hours outside the range.
3. It deletes the `ticker_data_coverage` rows whose (ticker, date) keeps
   neither an `ohlcv_1min` nor a `tick_10` row.
4. It runs `populate_trading_calendar()` with date_range None.
