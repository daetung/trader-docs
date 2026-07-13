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

## Per-Ticker Corporate-Event Anomaly Check (P-8)

Last-line defense against a vendor-latency miss: a same-day corporate
event (typically a reverse split) that neither premarket crawl caught
yet, which would otherwise leave every CONTINUOUS indicator for that
ticker badly distorted all session.

**Two distinct call patterns**, not one repeated twice — see "Premarket
Schedule" below for why they differ:

```
04:00 ET (--premarket-open, single process — see Premarket Schedule):
    1. detect_rename_candidates()                          # writes batch_runs
                                                             #   stage='premarket_rename'
    2. quotes       = bulk_fetch_today_first_price(...)     # trading API, chunked
       halt_status  = utils.query_halt_status(...)           # trading API, chunked
       # sequential, not parallel — see Premarket Schedule for why
    3. for ticker in active_ticker_universe:
           crawl_corporate_events(ticker, db_conn)            # yfinance — dominant cost
                                                             # writes batch_runs
                                                             #   stage='premarket_corporate_events'
    4. check_corporate_event_anomaly(db_conn, config, quotes, halt_status)
                                                             # writes batch_runs
                                                             #   stage='premarket_quarantine_check'

09:20 ET (--premarket-recheck, separate process):
    1. quotes       = bulk_fetch_today_first_price(...)     # trading API, chunked
       halt_status  = utils.query_halt_status(...)           # trading API, chunked
       # sequential — same function, same reasoning as above
    2. check_corporate_event_anomaly(db_conn, config, quotes, halt_status)
                                                             # writes batch_runs
                                                             #   stage='premarket_quarantine_recheck'
       # has_event_today still reflects only the 04:00 crawl — this pass
       # never re-crawls corporate_events (see Premarket Schedule)
```

Both quotes/halt fetches are sequential (not parallel) within themselves
— `utils.bulk_api_call_chunked()`'s chunking is already sequential per
endpoint for the same reason (an assumed server-side throughput ceiling,
not a per-connection one); running the two *different* endpoints (quotes,
halt) concurrently was considered and rejected for the same reason —
uncertain whether the trading API's request budget is shared across
endpoints, and a wrong guess there risks throttling both. ~150s each,
~300s combined, either pass — comfortably inside both passes' available
window (see Premarket Schedule).

```python
def bulk_fetch_today_first_price(
    tickers: list[str],
    trading_api_quotes_url: str,
    chunk_size: int,
) -> dict[str, float]:
    """
    Via utils.bulk_api_call_chunked() — NOT a true single call at universe
    scale (~15k tickers against an assumed ~100 tickers/sec real-world
    ceiling is ~150s even chunked; see that function).

    Returns {ticker: today_first_price} — a ticker with no premarket tick
    yet is absent, not 0/NaN; check_corporate_event_anomaly() already
    treats absence as "skip this ticker" (see below).
    """
    ...


def check_corporate_event_anomaly(
    db_conn, config, quotes: dict[str, float], halt_status: dict[str, bool] | None
) -> None:
    """
    Pure judgment step — takes quotes/halt_status as already-fetched
    arguments (see call patterns above) rather than fetching them itself.

    This offline batch context has no live tick stream, so there is no
    tick-rate fallback available here the way live_mode_runner.md has one
    for its own query_halt_status() call: if halt_status is None (total
    fetch failure) or a ticker is absent from it, that ticker is treated
    as NOT halted — the conservative direction. Suppressing a quarantine
    because halt status is merely unresolved would be wrong: this
    function's whole premise is that a false-positive quarantine (cost:
    one ticker's trades for one day) is far cheaper than trading a
    10x-distorted feature set (see the original P-8 problem statement) —
    the same asymmetry argues for resolving any ambiguity here toward
    quarantine-eligible, not away from it.

    For each ticker with a non-null quote in `quotes`:
        yesterday_close = <today's D-1 close, via the same halt-aware
            search utils.md's gap_pct C. logic uses for a single prior
            session: try D-1's 155900 bar close; if halt/missing,
            search backward within D-1's regular session for the last
            non-halt bar; if none found, skip this ticker (no baseline)>
        if yesterday_close is None: skip
        gap = abs(quotes[ticker] / yesterday_close - 1)
        has_event_today = EXISTS corporate_events
                           WHERE ticker = ? AND event_date = today
        is_halted = halt_status.get(ticker, False) if halt_status is not None else False

        if (gap > config["quarantine"]["price_anomaly_threshold"]
            and not has_event_today
            and not is_halted):
            UPDATE ticker_cik_map SET quarantine_reason = 'corporate_event_anomaly'
                WHERE ticker = ?
            # Does not touch rename_pending — see db_schema.md's
            # ticker_cik_map (N-2 restructure): the two suspend reasons are
            # now independent columns, so this write can never clobber an
            # unrelated rename-pending flag, and is_tradable() ORs both
            # — no code change needed there for this addition.
            log to tools/quarantine_candidates.log for manual review
        elif (current row has quarantine_reason = 'corporate_event_anomaly'
              and the condition above no longer holds):
            UPDATE ticker_cik_map SET quarantine_reason = NULL
                WHERE ticker = ?
            # Lets the 09:20 recheck self-clear a 04:00 false alarm caused
            # by thin premarket liquidity, without waiting for the
            # evening self-correction below.

    Writes its own batch_runs row — stage name depends on which of the two
    call patterns above invoked it ('premarket_quarantine_check' at 04:00,
    'premarket_quarantine_recheck' at 09:20) — status='running' at start,
    'success'/'failed' at completion, same convention as the other
    premarket stages.
    """
```

**Coverage note**: at 04:00 ET, premarket trading has barely opened —
most tickers have no tick yet, so `quotes` is sparse and the 04:00 pass
protects only the minority of tickers already trading. Real, near-full
coverage comes from the 09:20 recheck, once premarket liquidity has
built up. This is expected, not a defect — restated here because it is
easy to assume 04:00 is the primary defense when it is actually the
secondary one.

Evening self-correction for a quarantine set by this function is handled
by `self_correct_quarantine()` — see "Evening Ticker Rename
Self-Correction" below, which covers both self-correction functions
despite its (unchanged) section name.

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

Runs at premarket, as Step 1 of `--premarket-open` (N-1 — see "Dual
Schedule" below; no longer a separately-scheduled process alongside the
corporate-events crawl), **before** the crawl and before
`LiveModeRunner.start_session()` — a rename must be registered before
that day's Eager Pool bar loading, or the renamed ticker's history
lookback is silently truncated for that entire day.

```python
def detect_rename_candidates(db_conn) -> list[dict]:
    """
    1. Fetch SEC company_tickers.json → upsert ticker_cik_map
       (new (cik, ticker) pairings default to rename_pending=FALSE per
       db_schema.md — overridden to TRUE below when ambiguous. N-2: this
       and quarantine_reason, below check_corporate_event_anomaly(), are
       independent columns — see db_schema.md's ticker_cik_map)
    2. Find (cik, ticker_old) -> (cik, ticker_new) mapping changes where
       ticker_old has existing ohlcv_1min history (ticker_new's own history
       is not required — it doesn't exist yet at premarket)
    3. Unambiguous matches: INSERT OR IGNORE INTO ticker_history
       (effective_date = today, a best-effort estimate; self-corrected this
       evening — see below). ticker_cik_map row for ticker_new stays
       rename_pending=FALSE — no gate needed, identity is confirmed.
    4. Ambiguous / CIK-unmatched:
       UPDATE ticker_cik_map SET rename_pending = TRUE
           WHERE cik=? AND ticker=?
       (this is what LiveModeRunner's is_tradable() gate — see
       execution_common.md — actually reads; the log below is for human
       review, not machine consumption)
       log to tools/rename_candidates.log for manual review

    Fallback path (not integrated here): for cases SEC's company_tickers.json
    alone leaves ambiguous, a cross-check against the trading service API
    (the same API already queried in live_mode_runner.md's Session Lifecycle
    Step 1 for today's tradable ticker list) is intended as a secondary
    signal. Noted as a planned path only — the specific endpoint/call
    contract for this cross-check is not defined in this spec; SEC's
    company_tickers.json is the sole source actually implemented above.
    """
```

Scheduled automatically as Step 1 of `--premarket-open` (see "Dual
Schedule" below and Constraints) — no separate cron entry. Available
standalone as `--ticker-rename-only` for manual re-run/debugging only
(same "legacy flags remain for isolated re-runs" convention as
`--corporate-events-only` — see Constraints):
```bash
python tools/collect_daily.py --db-path data/market.duckdb --ticker-rename-only
```

Whichever path runs it, rename detection writes its own `batch_runs` row
(`stage='premarket_rename'`, `date=today`) — `status='running'` at start,
`status='success'`/`'failed'` at completion — same as before, independent
of the `stage='premarket_corporate_events'` row `crawl_corporate_events()`
writes later in the same `--premarket-open` process (see "Two distinct
call patterns" above). LiveModeRunner's Health Gate 1 reads both (see
live_mode_runner.md).

## Evening Ticker Rename Self-Correction

Runs during the evening full run, after ingestion:

```python
def self_correct_rename_effective_dates(db_conn, today) -> None:
    """
    For ticker_history rows with effective_date within the last 5 trading
    days: compare registered effective_date against
    MIN(date) FROM ohlcv_1min WHERE ticker = ticker_new.
    If different: UPDATE ticker_history SET effective_date = actual.

    For any ticker_new confirmed here (a row now exists in ohlcv_1min):
        UPDATE ticker_cik_map SET rename_pending = FALSE
            WHERE ticker = ticker_new
    — this is the only place rename_pending transitions back to FALSE for
    the rename case; a ticker flagged ambiguous at premarket stays
    rename_pending=TRUE (new entries blocked via is_tradable() — see
    db_schema.md's ticker_cik_map, N-2) until this same-evening check
    confirms it. Independent of, and never touches, quarantine_reason —
    see self_correct_quarantine() below.

    If ohlcv_1min still has no rows for ticker_new after grace_period
    (5 trading days): re-log to rename_candidates.log as a likely false
    positive from the premarket detector. rename_pending stays TRUE —
    resolution requires manual CLI action (--register-rename), not another
    automatic retry.
    """
```

Idempotent — safe to run every evening regardless of whether any rename is
pending correction.

```python
def self_correct_quarantine(db_conn, today) -> None:
    """
    Companion to self_correct_rename_effective_dates() above, same evening
    slot — corrects a quarantine set by check_corporate_event_anomaly()
    (see "Per-Ticker Corporate-Event Anomaly Check", P-8).

    For every ticker_cik_map row with quarantine_reason='corporate_event_anomaly':
        If corporate_events now has a row for (ticker, today) (the evening
        crawl, which runs after both premarket passes, may have caught the
        event both premarket passes missed): UPDATE quarantine_reason=NULL
        — the anomaly is now explained.
        Otherwise: leave quarantine_reason unchanged. Re-evaluated by
        tomorrow's premarket pass per check_corporate_event_anomaly() — a
        vendor delay stubborn enough to miss the evening crawl too can
        leave a ticker quarantined for more than one day; this is intended
        behavior, not a bug (see check_corporate_event_anomaly()'s
        false-positive-is-cheap rationale — the same asymmetry argues for
        leaving a still-unexplained ticker quarantined rather than
        timing it out).
    """
```

Idempotent — safe to run every evening regardless of whether any
quarantine is pending correction. Structurally cannot contend with
`self_correct_rename_effective_dates()` — since N-2's column split (see
db_schema.md), `quarantine_reason` and `rename_pending` are independent
columns, so the two self-correction functions each touch only their own
column and can never clobber the other's state, even for a ticker that
happens to be flagged by both at once.

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
# Premarket open, 04:00 ET — single process (N-1): rename detection,
# quotes/halt fetch, corporate-events crawl, anomaly check, in that order.
# Replaces running --corporate-events-only and --ticker-rename-only as two
# separate processes at this slot (see Constraints — that was a DuckDB
# single-writer collision risk, both processes writing ticker_cik_map at
# the same time; one sequential process removes the collision class).
python tools/collect_daily.py \
    --db-path data/market.duckdb \
    --premarket-open

# Premarket recheck, 09:20 ET — separate process (N-3): quotes/halt fetch
# + anomaly check only, no rename detection, no corporate-events crawl.
# Skips entirely (see Constraints) if the 04:00 process above is still
# running.
python tools/collect_daily.py \
    --db-path data/market.duckdb \
    --premarket-recheck
```

Timing: `--premarket-open` at 04:00 ET, well before premarket bar
accumulation begins. `--premarket-recheck` at 09:20 ET — chosen as the
latest point that comfortably fits its own cost (quotes + halt fetch,
sequential, ~150s each ≈ 300s total — see `check_corporate_event_anomaly()`
above) ahead of 09:30 regular-session open, while getting as close to open
as possible for freshness (premarket liquidity, and therefore
`bulk_fetch_today_first_price()` coverage, builds through the morning —
see that function's Coverage note). Deliberately NOT a third, later
recheck — a chain of ever-later passes trying to shave the remaining gap
closer to 09:30 reintroduces the same crawl-doesn't-fit problem this
09:20 design avoids by excluding the crawl entirely; see "Affected specs"
below for the residual this leaves.

`--premarket-open`'s `crawl_corporate_events()` step keeps the same
`INSERT OR IGNORE` idempotency as before — safe to re-run manually (e.g.
via the legacy `--corporate-events-only` flag) if needed outside the
normal schedule.

**This is risk mitigation, not a guarantee.** Split/dividend effective dates
are always before-market-open by convention (see `db_schema.md`
`corporate_events`), but *when yfinance's own data reflects that event* is
outside this tool's control — a genuinely late vendor update can still be
missed by the 04:00 crawl, and `--premarket-recheck` does not re-crawl to
catch it (see `check_corporate_event_anomaly()`'s reliance on price-gap
detection instead, which does not depend on `crawl_corporate_events()` at
all). This is a known, accepted limitation of relying on yfinance as the
sole corporate-events source, not a defect in the scheduling design; it
applies equally to `EntryPointDetector` filter E's point-in-time
`shares_outstanding` and to `gap_percentile()`'s `dividend_amount` for the
affected ticker/date only.

**Residual risk (tracked, not resolved here):** since corporate_events has
no path to update between the 04:00 crawl and the evening crawl, a
same-day event `check_corporate_event_anomaly()`'s price-gap heuristic
fails to flag (below `quarantine.price_anomaly_threshold`, or otherwise
missed) leaves that ticker's indicators silently distorted for the full
session with no further defense — there is no mechanism, at any point in
the day, that recomputes an already-running CachingIndicatorCalculator's
state once a corporate event is later confirmed (see
`caching_calculator.md`'s `on_bar_close()`, which deliberately permits
only O(1) incremental updates, never a full recomputation). Carried as an
open item — see `open_items_session3.md`.

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

# Premarket open (N-1) — the actual 04:00 ET scheduled entry (see Cron /
# Scheduler Setup); rename detection + quotes/halt fetch + corporate-events
# crawl + anomaly check, one sequential process
python tools/collect_daily.py \
    --db-path data/market.duckdb \
    --premarket-open

# Premarket recheck (N-3) — the actual 09:20 ET scheduled entry; quotes/halt
# fetch + anomaly check only, no crawl
python tools/collect_daily.py \
    --db-path data/market.duckdb \
    --premarket-recheck
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

Console output (human-readable, unchanged):
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

File log (machine-parseable, for `tools/health_report.md` — see that doc):
written to `logs/metadata_crawler_{date}.log` (one file per run date, not
appended across dates), every line tab-separated `key=value`, no free-form
text:

```
2026-07-11T04:23:01	INFO	ticker=AAPL	status=updated	source=yfinance
2026-07-11T04:23:47	WARN	ticker=XYZ	status=failed	source=yfinance	fallback=fmp	fallback_status=success
2026-07-11T05:47:33	SUMMARY	stage=evening_ingestion	date=20260711	processed=11847	updated=11820	failed=27
```

A `SUMMARY` line is written at the end of each stage. Its absence (rather
than a `failed` count) is `health_report.md`'s signal for an abnormal
termination, distinct from a high-but-nonzero failure count. Per-ticker
`INFO`/`WARN` lines are for human debugging; only `SUMMARY` lines are
parsed by `health_report.md`.

---

## Cron / Scheduler Setup

**Evening job start gate**: before its first stage (metadata/corporate-events
crawl) begins, the evening run polls `batch_runs` for
`stage='live_session_end' AND date=today, status='success'` — written by
LiveModeRunner at its own session shutdown (see live_mode_runner.md's
Session Shutdown). This is the DuckDB-ownership-handoff half of P-5's
temporal ownership design (the other half — LiveModeRunner waiting on the
premarket marker before opening its own connection — is in
live_mode_runner.md's Session Start Gating). No timeout specified here —
unlike the premarket wait, there is no equivalent "proceed in degraded mode"
option for the evening job; if LiveModeRunner's session hasn't ended yet,
the evening job simply keeps waiting (regular-session trading hours end well
before 17:00 ET in ordinary operation, so this is expected to be a very
short or zero wait in practice — a long wait here would itself indicate
something worth investigating, not something to silently override).

```bash
# Run after market close daily (weekdays only), 17:00 ET
# Full run: metadata + corporate events + ingest + calendar update + session stats
0 17 * * 1-5 cd /path/to/stock-scalping && \
    source .venv/bin/activate && \
    python tools/collect_daily.py \
        --db-path data/market.duckdb \
        --data-root /path/to/json/data \
        --date today >> logs/collect_daily.log 2>&1

# Premarket open (N-1) — replaces separate --ticker-rename-only and
# --corporate-events-only cron entries previously here, both at 04:00 ET
# (see "Dual Schedule" above and Constraints for the collision this
# merge removes)
0 4 * * 1-5 cd /path/to/stock-scalping && \
    source .venv/bin/activate && \
    python tools/collect_daily.py \
        --db-path data/market.duckdb \
        --premarket-open >> logs/collect_daily_premarket.log 2>&1

# Premarket recheck (N-3) — quotes/halt fetch + anomaly check only, no
# crawl; moved from ~09:25 ET to 09:20 ET after confirming the crawl
# cannot fit that window at realistic API throughput (see "Dual Schedule")
20 9 * * 1-5 cd /path/to/stock-scalping && \
    source .venv/bin/activate && \
    python tools/collect_daily.py \
        --db-path data/market.duckdb \
        --premarket-recheck >> logs/collect_daily_premarket.log 2>&1
```

---

## secrets.yaml (gitignored)

```yaml
# configs/secrets.yaml — DO NOT COMMIT
fmp_api_key: "your_key_here"
```

---

## Config Keys (pipeline_config.yaml)

```yaml
# Bulk today-vs-yesterday price quote endpoint for
# check_corporate_event_anomaly() (P-8) — a distinct endpoint from
# trading_api_url / trading_api_ticker_url (see live_mode_runner.md),
# since this crawler runs standalone, outside any LiveModeRunner session.
# URL path and response schema: TBD (API spec sheet).
trading_api_quotes_url: "http://trading-api/quotes/today"

# Chunk size for utils.bulk_api_call_chunked() — shared by
# bulk_fetch_today_first_price() (below) and, via the same physical
# pipeline_config.yaml, live_mode_runner.md's query_halt_status() call
# (see that file's Position Manager Loop Step 1a). Estimated against a
# ~100 tickers/sec real-world throughput ceiling — TBD pending the actual
# API spec sheet.
bulk_api_chunk_size: 50

quarantine:
  price_anomaly_threshold: 0.40   # |today_first_price / yesterday_close - 1|
                                  # above this, with no corporate_events row
                                  # for today and no halt explanation, quarantines
                                  # the ticker for the day — see
                                  # check_corporate_event_anomaly(). Threshold
                                  # is a placeholder pending shadow-stage tuning
                                  # against real momentum-gapper false-positive
                                  # rates (see original P-8 problem statement).
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
- The evening run writes three `batch_runs` rows as it progresses —
  `stage='evening_ingestion'` (around Steps 3-4 above), `stage='evening_tick_bar_aggregates'`
  (Tick Bar Aggregates Update), `stage='evening_session_stats'` (Session Stats
  Update) — each `status='running'` at its own start, `'success'`/`'failed'`
  at its own completion, independent of the other two. This lets
  LiveModeRunner's Health Gate 1 (live_mode_runner.md) distinguish exactly
  which stage of an evening run failed, not just that the run as a whole did.
- The evening run writes three `batch_runs` rows as it progresses —
  `stage='evening_ingestion'` (around Steps 3-4 above), `stage='evening_tick_bar_aggregates'`
  (Tick Bar Aggregates Update), `stage='evening_session_stats'` (Session Stats
  Update) — each `status='running'` at its own start, `'success'`/`'failed'`
  at its own completion, independent of the other two. This lets
  LiveModeRunner's Health Gate 1 (live_mode_runner.md) distinguish exactly
  which stage of an evening run failed, not just that the run as a whole did.
- `--premarket-open` (N-1; replaces running `--ticker-rename-only` and
  `--corporate-events-only` as two separate cron processes at 04:00 ET)
  runs `detect_rename_candidates()` → `bulk_fetch_today_first_price()` +
  `utils.query_halt_status()` → `crawl_corporate_events()` for all tickers
  → `check_corporate_event_anomaly()`, as one sequential process — no
  metadata fields, no ingestion, no calendar/session-stats steps. Two
  separate OS processes both writing `ticker_cik_map` at the same 04:00
  slot was a DuckDB single-writer collision risk (P-5) — merging into one
  process removes the collision class entirely rather than coordinating
  around it. Writes `batch_runs` rows for `stage='premarket_rename'`,
  `stage='premarket_corporate_events'`, and `stage='premarket_quarantine_check'`
  sequentially as each internal step completes, preserving the same
  per-stage failure granularity Health Gate 1 relies on (see
  live_mode_runner.md) even though it is now one process, not three.
  `--ticker-rename-only` and `--corporate-events-only` remain available as
  lower-level flags for manual re-run/debugging of one piece in isolation
  — only the automated cron schedule moves to `--premarket-open`.
- `--premarket-recheck` (N-3; the 09:20 ET pass) runs
  `bulk_fetch_today_first_price()` + `utils.query_halt_status()` →
  `check_corporate_event_anomaly()` only — no rename detection, no
  corporate-events crawl. Before doing any of its own work, checks
  `batch_runs` for `stage='premarket_corporate_events', date=today`; if
  `status='running'` (the 04:00 `--premarket-open` process is still
  mid-crawl), skips this pass entirely rather than risk the same
  single-writer collision `--premarket-open`'s own merge was meant to
  avoid — the still-running 04:00 process's eventual completion is what
  matters, not a second concurrent attempt. Writes its own `batch_runs`
  row: `stage='premarket_quarantine_recheck'`.
- Must be safe to run multiple times on the same date (idempotent) —
  `--premarket-open`'s four internal steps are each independently
  idempotent (`crawl_corporate_events()`'s `INSERT OR IGNORE`;
  `bulk_fetch_today_first_price()`/`query_halt_status()` are stateless
  reads; `check_corporate_event_anomaly()`'s UPDATE reflects only that
  run's fresh inputs, not an accumulating count) — re-running the whole
  flag is therefore also safe, though the collision guard above means
  `--premarket-recheck` specifically does not attempt this concurrently
  with a still-running `--premarket-open`.
- Log files (`tools/rename_candidates.log`, `tools/quarantine_candidates.log`,
  `tools/metadata_missing.log`) have no rotation/retention policy specified
  — grow unbounded. Low-cost operational gap, not a correctness one; a
  standard log-rotation tool (e.g. `logrotate`) applied externally is
  sufficient, no application-level change needed.
- On a non-trading day (market holiday, weekend), all scheduled crons
  above still fire and complete quickly as effective no-ops (no tickers
  to detect renames for, no new bars to crawl against) — each still
  writes its normal `batch_runs` row with `status='success'`. This is
  intended: `trading_calendar`'s `is_trading_day` is what downstream
  consumers (Health Gate 1, LiveModeRunner's own start condition — not
  specified here, out of scope) check to decide whether to actually run a
  session that day, not the presence or absence of a `batch_runs` row.
- Failed metadata tickers do not block ingestion — two steps are independent
- No session time filter on ingested data — all periods (040000~200000) stored
- US stock splits and ex-dividend adjustments always take effect before market
  open — no intra-session event handling needed
- Vendor update latency (yfinance not yet reflecting a same-day corporate
  event by the time either premarket crawl runs) is a known, accepted
  limitation — not resolvable by adjusting crawl timing further; downstream
  consumers (filter E, gap_percentile dividend_amount, meta resolution) treat
  this the same as any other missing-data case, not as an error
