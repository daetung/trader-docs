# Tool: Metadata Crawler + Daily Data Collector

**File:** `tools/collect_daily.py`
**Standalone CLI tool — runs daily via cron or task scheduler**

---

## Role

Seven responsibilities combined into one daily-run tool, split across two
schedule points (see "Dual Schedule" below — evening full run, plus a
lightweight premarket corporate-events refresh), with a third, unrelated
schedule entry for the overnight token refresh (see "Overnight Token
Refresh"):

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
Writes to `corporate_events` via the shared `upsert_corporate_event()`
helper (item N), not a bare INSERT OR IGNORE — see below.

```python
def crawl_corporate_events(ticker: str, db_conn) -> int:
    """
    Fetch split, reverse-split, and dividend history from yfinance and upsert
    into corporate_events via the shared upsert_corporate_event() helper
    (item N — the same helper crawl_corporate_events_investing() uses below,
    so a yfinance write and a later investing.com write for the same event
    are both conflict-checked against whatever is already there, regardless
    of which vendor wrote first; without this, a write ordered before an
    investing.com write for the same event would bypass conflict detection
    entirely via a bare INSERT OR IGNORE). Determines event_type from split
    ratio: >1.0 → 'split', <1.0 → 'reverse_split'. Dividends stored
    separately with event_type='dividend' and value=per-share cash amount.
    Returns: number of rows newly inserted or updated (a no-op agreement
    with an existing row is not counted — see upsert_corporate_event()).
    """
    inserted = 0

    splits = yf.Ticker(ticker).splits
    for date, ratio in splits.items():
        event_type = "split" if ratio > 1.0 else "reverse_split"
        if upsert_corporate_event(
            ticker, date.strftime("%Y%m%d"), event_type, float(ratio),
            "yfinance", db_conn,
        ):
            inserted += 1

    dividends = yf.Ticker(ticker).dividends
    for date, amount in dividends.items():
        if upsert_corporate_event(
            ticker, date.strftime("%Y%m%d"), "dividend", float(amount),
            "yfinance", db_conn,
        ):
            inserted += 1

    return inserted
```

Notes:
- Both splits and dividends collected in the same function call (single
  yfinance ticker session per invocation) — see Items 1+2+3 design; dividend
  event_type is no longer reserved-only, it is actively collected
- yfinance split/dividend history is comprehensive for US equities; no
  fallback needed for either
- Writes via upsert_corporate_event() (item N), not a bare INSERT — safe to
  re-run (a no-op on an already-agreeing row), and symmetric with
  crawl_corporate_events_investing()'s own write path — see that function's
  Second Vendor section below for the helper's definition.

### Second Vendor: investing.com (item N)

Price-gap detection (`check_corporate_event_anomaly()`, below) has a
structural noise floor — a small dividend (~0.5% gap) cannot be separated
from ordinary premarket noise by any threshold choice. A second, date-scoped
vendor calendar sidesteps the price signal entirely instead of trying to
tune around that floor.

```python
def crawl_corporate_events_investing(date: str, db_conn) -> int:
    """
    Scrapes investing.com's calendar for ALL of `date`'s splits and
    dividends in a single page-load per type — unlike
    crawl_corporate_events()'s per-ticker yfinance loop, there is no ticker
    iteration here.

    Two scraped pages (separate endpoints, placeholders — see Config Keys):
        SPLIT_CALENDAR_URL     = <TBD placeholder>
        DIVIDEND_CALENDAR_URL  = <TBD placeholder>

    Filters scraped rows to active_ticker_universe via query-time symbol
    normalization — the same case/separator/class-suffix rules
    detect_rename_candidates() already applies, reused here rather than
    reimplemented, since both are "does this vendor's symbol string
    identify a ticker we track" problems (resolved this design pass; no
    longer a naive exact match). Normalization is best-effort, not a
    guarantee — health_report.md finding 10 keeps tracking the residual
    mismatch rate — then writes each row through the shared
    upsert_corporate_event() helper below with source='investing'.
    Does NOT write corporate_events directly: the one-row-per-event
    invariant (see db_schema.md) and the vendor-disagreement handling both
    live in that helper, so neither vendor's crawler can bypass them.

    Returns: number of rows newly inserted or updated in corporate_events
    (a no-op agreement with an existing row is not counted).
    """
    ...
```

Notes:
- Called at 04:00 IN ADDITION TO the yfinance per-ticker loop above (not a
  replacement — coverage safety, neither vendor assumed 100%) and again at
  09:20 (in-process — see live_mode_runner.md's In-Process Premarket
  Recheck)
- Also called forward-looking, for the next trading day, during the
  evening run (investing.com's calendar is populated ahead of the
  effective date)
- Symbol matching applies the same query-time normalization rules as
  ticker-rename detection (case/separator/class-suffix) — no longer naive
  exact matching. Best-effort, not a guarantee: residual mismatches stay
  visible via health_report.md finding 10's match-rate tracking rather
  than being treated as a solved-and-closed gap.
- Coverage assumption: investing.com's calendar is assumed to span the
  full US-listed universe this system trades, not a large-cap subset —
  observed as such this session but not independently verified against a
  vendor spec sheet. Treated the same as active_ticker_universe for
  matching purposes; if coverage later proves partial, "not found in
  investing.com's calendar" stops being a safe proxy for "no event" and
  this crawl's absence-of-a-row can no longer be read as confirmation.

### Shared Corporate-Event Write Path (item N)

Both vendors' crawlers write through this one helper. Neither writes
`corporate_events` directly — the one-row-per-event invariant that
`cum_split_ratio()` depends on (see db_schema.md) is enforced here, in a
single place, rather than trusted to each caller's choice of
`INSERT OR IGNORE` / `INSERT OR REPLACE`.

```python
def upsert_corporate_event(
    ticker: str, event_date: str, event_type: str,
    value: float, source: str, db_conn,
) -> bool:
    """
    Vendor-agnostic write path for corporate_events.

        no existing row for (ticker, event_date, event_type)
            -> INSERT this row (source recorded).
        existing row, values AGREE within tolerance
            -> no-op. Agreement needs no second row, and rewriting only
               churns `source` for no gain.
        existing row, values DISAGREE
            -> corporate_events keeps (or is updated to) the investing.com
               value — the confirmed tie-break, since investing.com is a
               date-scoped same-day query and treated as fresher — and BOTH
               values are written to corporate_event_conflicts (db_schema.md)
               so the disagreement survives for inspection.

    Agreement tolerance: relative, config
    `quarantine.corporate_event_value_tolerance` (placeholder default; the
    right magnitude is unknown until real cross-vendor values are observed
    — the two vendors are expected to differ in rounding, e.g. a dividend
    of 0.25 vs 0.2500, which must count as agreement, while a genuine
    0.25-vs-0.30 disagreement must not). Compared as
    abs(a - b) <= tol * max(abs(a), abs(b)).

    Returns True if corporate_events was inserted or updated, False on a
    no-op agreement.
    """
    ...
```

Note that a conflict does NOT quarantine the ticker or block anything —
one vendor being wrong about a dividend's third decimal is not grounds to
stop trading it. The conflict row and its health_report finding are for a
human to look at, and for judging whether the tolerance above is set
sensibly once real data exists.

### Shared Corporate-Event Write Path (item N)

Both vendors' crawlers write through this one helper. Neither writes
`corporate_events` directly — the one-row-per-event invariant that
`cum_split_ratio()` depends on (see db_schema.md) is enforced here, in a
single place, rather than trusted to each caller's choice of
`INSERT OR IGNORE` / `INSERT OR REPLACE`.

```python
def upsert_corporate_event(
    ticker: str, event_date: str, event_type: str,
    value: float, source: str, db_conn,
) -> bool:
    """
    Vendor-agnostic write path for corporate_events.

        no existing row for (ticker, event_date, event_type)
            -> INSERT this row (source recorded).
        existing row, values AGREE within tolerance
            -> no-op. Agreement needs no second row, and rewriting only
               churns `source` for no gain.
        existing row, values DISAGREE
            -> corporate_events keeps (or is updated to) the investing.com
               value — the confirmed tie-break, since investing.com is a
               date-scoped same-day query and treated as fresher — and BOTH
               values are written to corporate_event_conflicts (db_schema.md)
               so the disagreement survives for inspection.

    Agreement tolerance: relative, config
    `quarantine.corporate_event_value_tolerance` (placeholder default; the
    right magnitude is unknown until real cross-vendor values are observed
    — the two vendors are expected to differ in rounding, e.g. a dividend
    of 0.25 vs 0.2500, which must count as agreement, while a genuine
    0.25-vs-0.30 disagreement must not). Compared as
    abs(a - b) <= tol * max(abs(a), abs(b)).

    Returns True if corporate_events was inserted or updated, False on a
    no-op agreement.
    """
    ...
```

Note that a conflict does NOT quarantine the ticker or block anything —
one vendor being wrong about a dividend's third decimal is not grounds to
stop trading it. The conflict row and its health_report finding are for a
human to look at, and for judging whether the tolerance above is set
sensibly once real data exists.

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
    3b. crawl_corporate_events_investing(today, db_conn)     # item N: investing.com
                                                             # bulk (own db_conn — no
                                                             # LiveModeRunner running
                                                             # yet, no write-serialization
                                                             # concern at this hour)
    4. check_corporate_event_anomaly(db_conn, config, quotes, halt_status)
                                                             # writes batch_runs
                                                             #   stage='premarket_quarantine_check'

09:20 ET (R-1: LiveModeRunner in-process task — see live_mode_runner.md's
"In-Process Premarket Recheck"; --premarket-recheck below is manual/debug
only, cannot open the DB read-write during a live session):
    1. quotes       = bulk_fetch_today_first_price(...)     # trading API, chunked
       halt_status  = utils.query_halt_status(...)           # trading API, chunked
       # sequential — same function, same reasoning as above
    2. check_corporate_event_anomaly(db_conn, config, quotes, halt_status)
                                                             # writes batch_runs
                                                             #   stage='premarket_quarantine_recheck'
       # full-universe fresh re-evaluation, NOT a delta
    3. crawl_corporate_events_investing(today, db_conn)      # item N: yes, this
                                                             # pass DOES refresh
                                                             # corporate_events now
                                                             # (contrast the old
                                                             # design, which never
                                                             # re-crawled at 09:20)
    4. yfinance narrow crawl (crawl_corporate_events(ticker, db_conn)) for
       any ticker newly quarantined in step 2                      # item N
    5. scoped-recompute trigger for any ticker that gained a new same-day
       corporate_events row in steps 3-4 and already has a calculator
       (caching_calculator.md's scoped_recompute())                # item N
       # writes through db_write() — this process's own writer, no
       # inter-process coordination (see live_mode_runner.md)
```

Both quotes/halt fetches are sequential (not parallel) within themselves.
The quotes side batches behind TradingAPI, which paces against the
endpoint's published rate; the halt side is a different vendor entirely
(`utils.query_halt_status()`), so nothing about the two is shared. Running
them concurrently was considered and rejected: the gain is bounded by the
slower of two independent budgets, while a wrong guess about either risks
throttling both. Minutes each, comfortably inside both passes' available
window (see Premarket Schedule).

```python
def bulk_fetch_today_first_price(
    tickers: list[str],
) -> dict[str, float]:
    """
    Through TradingAPI (docs/api/trading_api.md), which owns endpoint
    addressing and the batching the endpoint's symbol cap requires. Takes
    no URL and no chunk size. NOT a true single call at universe scale —
    ~15k tickers against the published per-endpoint rate still takes
    minutes — but the caller does not express that.

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
            session: try D-1's last_bar(D-1) close; if halt/missing,
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
              and has_event_today):
            UPDATE ticker_cik_map SET quarantine_reason = NULL
                WHERE ticker = ?
            # item N fix: CLEAR depends on has_event_today ALONE. The
            # prior version re-tested the full inverted SET condition
            # (gap / has_event_today / is_halted) — so an unrelated halt
            # (making is_halted True) could wrongly clear a
            # still-unexplained quarantine. is_halted and gap are SET-time
            # false-positive guards only; at CLEAR time the only thing
            # that means "the gap is now explained" is a corporate_events
            # row existing for today.
            # Lets the 09:20 recheck (or item N's narrow/bulk crawls)
            # self-clear a 04:00 false alarm caused by thin premarket
            # liquidity, without waiting for the evening self-correction
            # below.

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
NOT per-date and deliberately so: the filter exists to exclude OVERNIGHT, and an
early-close day has no data past its own `after_hours_end` (17:00), so the fixed
upper bound still admits everything real and still excludes overnight.

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

## Evening Forward-Looking Corporate-Events Check (item N)

After Session Stats Update, query investing.com's calendar for the NEXT
trading day — investing.com is forward-looking (scheduled events are shown
ahead of their effective date), unlike yfinance's `crawl_corporate_events()`
which only reflects events already recorded as having happened.

```python
crawl_corporate_events_investing(next_trading_day, db_conn)
```

Does NOT affect tonight's `precomputed_session_stats` — those are past-date
baselines already computed above; a future-dated corporate_events row cannot
enter their computation, so this step is safe to run after the
ordering-sensitive session-stats step rather than before it. Writes
`batch_runs` `stage='evening_investing_forward_check'`
(`status='running'` at start, `'success'`/`'failed'` at completion).

---

## Feed Coverage Analysis (`stage='evening_feed_coverage'`)

Reads the day's raw per-second aggregates from
`data/feed_coverage/{live,delayed}/YYYYMMDD.jsonl`, writes one
`feed_coverage_daily` row per `(date, ticker)`, then disposes of the raw
pair. The live side is written by LiveModeRunner and the delayed side by
`auxiliary_stream.md`; each file has exactly one writer, so neither engages
P-5's single-writer constraint — they are not database files.

**Placed fifth, before Retention Purge.** `feed_coverage_daily` is a
purge-registry member, so running after the purge would leave the registry
counting a table whose row for that date does not yet exist. It
reads no DB table and depends on no other stage, so a failure here blocks
nothing downstream.

It runs after the delayed side is complete. That stream trails real time by
roughly fifteen minutes against a 20:00 session end, which the 21:00 slot
clears by about an hour.

**SKIP and FAILURE are different outcomes, and conflating them breaks in one
direction or the other** — treating an absent day as failure would
accumulate a retry every day, while treating a dead collector as a
skip would lose the data silently. There is no deliberately-disabled case:
loop A always starts and always subscribes, the former
`auxiliary_stream.enabled` switch having been retired.

| Condition | Outcome |
|---|---|
| Neither file exists | SKIP — no-op `batch_runs` row, the same shape the non-trading-day crons already write |
| Exactly one file exists | FAILURE — one process writes both sides now, so live-only means loop A's thread died and delayed-only means the runner never wrote its own aggregates; loop A's per-session health event corroborates the former |
| Either file fails to parse | FAILURE |
| A file stops mid-session | SUCCESS — analysed for what it holds |

A file stopping mid-session is not a failure because partial capture was
accepted when the flush cadence was chosen: the callback holds sealed
seconds in memory and an abrupt death loses the last interval.

**The ticker population is the DELAYED side's.** A ticker present in live but
absent from delayed would compute as total dropout, and that is
indistinguishable from the collector having subscribed it LATE — the gap
between a lost set publication and the next `lifecycle` transition. Eviction
is no longer part of that ambiguity: it is marked, so a ticker uncovered for
the whole session by eviction is NOT excluded — a row is written with every
counter at zero and `delayed_uncovered_seconds` carrying the interval. Rows
with zero denominators are therefore an expected output and every ratio
consumer must handle them. The delayed stream
carries the complete tape, which is what makes it the correct population. The
EXCLUDED-TICKER count goes to the log and to `health_report.md`'s observation
section only; it does not get a `batch_runs` column, since a large count is
already a signal without changing the schema for a diagnostic. Per-second
non-coverage is different in kind — it distorts counters on a row that DOES
exist — and is carried by `delayed_uncovered_seconds`.

**Uncovered records.** A delayed-side record carrying the `covered` key is
not an aggregate: it is collected into that ticker's uncovered second-set and
enters none of `delayed_seconds`, `delayed_prints`, `delayed_volume`, the
density-bucket keying, or the second-set `live_zero_seconds_*` tests against.
When the live side is streamed against the delayed side, a `(ticker, second)`
in that set is SKIPPED ON BOTH SIDES — a second the delayed side could not
observe reaches none of `live_zero_seconds_d*`,
`live_missed_high_seconds`, `live_missed_low_seconds`.
`delayed_uncovered_seconds` is written as the size of that set. These records
are sparse and do not shift the memory figure below.

**A partial failure writes nothing.** If some tickers aggregate and others
fault, the whole date fails rather than being written short: this table
exists to measure a coverage RATIO, and a partial write biases exactly the
quantity it reports. Which tickers faulted, and why, go to the log and the
observation section — visible without being recorded as data.

**Disposal is carried in the filename, not in a table.** On success the raw
pair is deleted. On failure it is renamed with a `.failed` suffix, and the
next evening's stage picks up both that day's pair and any suffixed pair. A
pair that fails while already suffixed is deleted: exactly one retry, with no
counter to store and no unbounded accumulation, and the state is visible from
a directory listing. **The two files are renamed and deleted TOGETHER,
never independently** — otherwise the next run pairs a fresh file against a
stale one. A pair found in mismatched state is squared up before analysis
rather than analysed.

**Memory.** The delayed side is loaded in full first, because the density
buckets key on ITS prints-per-second and `live_zero_seconds_*` needs its
second-set; the live side then streams against it. At the 50-ticker
subscription ceiling this is roughly 1.2M entries, on the order of
150–200 MB — stated rather than left implicit, in the manner of
`live_mode_runner.md`'s Bulk Load Memory Profile. If that proves too large,
the fallback is a per-ticker two pass: 50 file scans for negligible memory.

**Re-running a date is safe.** The stage deletes that date's
`feed_coverage_daily` rows and re-inserts, rather than merging — see
`db_schema.md`, where the contrast with `bar_latency_daily`'s additive upsert
is recorded.

---

## Detection-Gap Analysis (`stage='evening_detection_gap'`)

Runs `EntryPointDetector.detect()` in memory over the day's ingested
`ohlcv_1min` for every non-quarantined ticker holding bars, and compares the
result against what the live session actually evaluated, recorded in
`inference_log`. Writes three `live_scan_daily` metrics for the date and
disposes of nothing.

**Why it exists.** The watchdog scan's early stop is sound only if the
watchdog fires for every ticker that crosses conditions A-G
(`api_contract_checklist.md` T-17), and that is an ACCEPTED, not proven,
assumption. The in-session mitigations — the rotation cursor and promotion —
all walk the watchdog LIST, so a ticker the watchdog never lists at all is
invisible from inside the session by construction. This stage is the only
independent ground truth in the system, which is what makes accepting the
assumption defensible rather than merely convenient.

**Placed sixth, immediately before Retention Purge.** It needs the day's
bars, so it must run after ingestion; it writes a purge-registry member, so
it must run before the purge, for the same reason Feed Coverage Analysis
does. It is non-destructive and nothing downstream depends on it.

**It must NOT write `entry_points`.** That table has no writer discriminator
and is written by both Preprocessor and Inferencer (see `db_schema.md`'s
Ingestion Rules), so writing there would both pollute the training corpus
with rows that were never live decisions and make the comparison
self-referential — this stage would be measuring itself. `detect()` runs in
memory and only counts are persisted.

**Both sides must carry the same filters.** `session_mode`, the late-day
cutoff (`entry_detector.max_entry_offset_minutes`, skipped entirely when null)
and the after-market exclusion are applied by Inferencer BEFORE `detect()`, and
it logs nothing when they reject. The late-day cutoff is now PER DATE — it
resolves as `last_bar(date) - max_entry_offset_minutes` — so the evening pass
must resolve it against the SESSION'S date, not the date the pass runs on.
While it was an absolute time the two agreed trivially; they no longer do.
Comparing an unfiltered evening pass against that would turn every deliberate
exclusion into a false gap — the filters go on both sides or the number means
nothing.

The three categories, and the distinction that carries the diagnosis:

| Category | Metric | Meaning |
|---|---|---|
| `detect()` true, an `inference_log` row exists with `no_detection` | `gap_disagreed` | The ticker WAS evaluated live and rejected. Not a scan gap — a data disagreement, i.e. the bars differ between live and post-ingestion. An independent second read on feed quality alongside `feed_coverage_daily` |
| `detect()` true, no `inference_log` row at all | `gap_unevaluated` | Never evaluated. The scan never reached it |
| Either, summed | `gap_total` | |

`gap_disagreed` versus `gap_unevaluated` is the whole point of routing
bar-close evaluation through `infer()` rather than pre-filtering
(`live_mode_runner.md`): the `no_detection` event is the only record
anywhere that a candidate was evaluated AND rejected, and without it "we
looked and it failed" collapses into "we never looked".

**Carryover's share of `gap_unevaluated` is already measured in-session** as
`live_scan_daily.carryover_tickers`, so the watchdog's own share is
recoverable by subtraction. No per-ticker roster table is added for it: the
first days of data size the residual before any storage is built for it.

**Universe scoping has a known residual.** The population is tickers holding
today's `ohlcv_1min`, minus quarantine. The session's actual tradable list is
persisted nowhere, so tickers the session could not have traded are still
counted. Conditions A (price <= 20), B (50000) and D (100000) make the
passing set small enough that this is observable rather than dominant —
stated rather than silently absorbed.

---

## Retention Purge (R-9) — final evening stage

The last stage of the evening run, after every other stage has completed.
Driven by an explicit registry — a table absent from it is UNREACHABLE by
this code, which is the whole point: structural exclusion rather than an
infinite default means no misconfiguration can delete the training corpus or
the P&L record (see db_schema.md's Retention note).

```python
# {table: (date_column, retention_days)} — the ONLY tables this stage can touch
PURGE_REGISTRY = {
    "inference_log":      ("date",   float("inf")),
    "live_positions":     ("date",   float("inf")),
    "live_session_state": ("date",   float("inf")),
    "batch_runs":         ("date",   float("inf")),
    "bid_ask_snapshots":  ("date",   float("inf")),
    "bar_latency_daily":  ("date",   float("inf")),
    "feed_coverage_daily": ("date",  float("inf")),
    "live_scan_daily":    ("date",   float("inf")),
    "live_halt_episodes": ("date",   float("inf")),
    "live_ticker_terms":  ("date",   float("inf")),
    "exit_trigger_agreement_daily": ("date", float("inf")),
    "health_events":      ("date",   float("inf")),
    "alert_log":          ("date",   float("inf")),
    "train_log":          ("run_at", float("inf")),
    "experiment_log":     ("run_at", float("inf")),
}

for table, (date_column, retention_days) in PURGE_REGISTRY.items():
    if retention_days == float("inf"):
        continue                      # nothing deleted; still counted below
    cutoff = today_date - retention_days
    db_write(f"DELETE FROM {table} WHERE {date_column} < ?", [cutoff])
```

**Every entry ships at `inf`, so this stage deletes nothing today.** That is
deliberate, not a placeholder: no growth rate for these tables has ever been
measured, so any window set now would be a guess, and deletion is not
reversible. health_report.md's DB Health Observation is what measures them;
an operator sets a real window once there is data to set it against. Writing
the mechanism now rather than later means that change is a config edit, not a
new stage built under disk pressure.

`date_column` differs per table because the tables date themselves
differently — `train_log` and `experiment_log` use `run_at` (execution time),
NOT `fold_test_start`/`fold_test_end`, which are the DATA window: backtesting
an old period writes a row whose data window is years old, and purging on
that would delete a row the moment it was created.

Runs inside the evening batch's existing writer connection — no separate
connection and no new lock interaction — and writes its own `batch_runs`
`stage='evening_retention_purge'` row (`'running'` at start,
`'success'`/`'failed'` at completion). Logs one line per table
(`"N tables scanned, 0 rows deleted"` while every window is `inf`), so the
stage's own no-op is visible rather than silent. That per-table line is IN
ADDITION TO this stage's `SUMMARY` line, not a substitute for it — every
stage writes one (see Output / Logging and Constraints), and omitting it
here would make health_report.md's finding 4 report an abnormal termination
every evening for a stage that ran perfectly.

Placed last because it is the only destructive stage: everything that reads
this evening's data has already run, and a failure here cannot cost the run's
actual work.

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
    alone leaves ambiguous, a cross-check against the trading API's
    ticker-master endpoint (the same source live_mode_runner.md's Session
    Lifecycle Step 1 reads for today's tradable ticker list) is intended
    as a secondary signal. Noted as a planned path only — the specific
    endpoint/call contract for this cross-check is not defined in this
    spec; SEC's company_tickers.json is the sole source actually
    implemented above.
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

## Premarket Trading-API Symbol Map

Runs at premarket, immediately after Premarket Ticker Rename Detection
(Step 1) within the same `--premarket-open` process, so matching reads
that day's just-refreshed rename state. A different problem from the
rename mechanism above: rename detection tracks a CIK's symbol changing
over time within our own universe, while this step maps our symbol to the
trading API's own symbol for that same, unchanged identity — needed
because the two vendors need not format a ticker the same way (e.g. a
class-suffix ticker rendered differently by each).

```python
def build_trading_api_symbol_map(db_conn) -> list[dict]:
    """
    1. Three bulk calls to the trading API's own ticker-master endpoint,
       one per listing exchange (NYSE, NASDAQ, AMEX) — a single call each,
       full universe returned per call, no per-ticker round trip. Exchange
       is the call's own input parameter, not a response field: every
       ticker returned by a given call is tagged with that call's exchange
       directly, never parsed out of the response.
    2. Match each returned ticker to a ticker_cik_map row: exact match
       first, then the same normalization rules as
       detect_rename_candidates() (case, separator, class-suffix handling)
       — reused rather than reimplemented, since both are "does this
       vendor's symbol string identify a row we already track" problems.
    3. Matched: UPDATE ticker_cik_map SET trading_api_symbol = ?,
       trading_api_exchange = ? WHERE cik=? AND ticker=? (see
       db_schema.md — trading_api_symbol stores the raw code only, never
       an exchange-prefixed form; the prefix is assembled at the point of
       an actual trading-API call, from trading_api_exchange, so a future
       change to the prefix convention touches only that call site, not
       stored data).
    4. Unmatched after normalization: log to
       tools/trading_api_symbol_candidates.log for manual review — same
       "automated first pass, manual fallback for the residue" shape as
       detect_rename_candidates(), resolved via
       --register-trading-api-symbol (see CLI Usage).
    5. No active_ticker_universe filter anywhere above: a ticker absent
       from ticker_cik_map simply has no row to match against and is
       skipped at step 2 by construction — universe scoping is an
       entry-point-detection concern (01_entry_detection.md), not this
       step's; every ticker the trading API supports is in scope here.

    Full-universe overwrite every run, not a delta: all three calls return
    the complete current set each time, so there is nothing to track
    between runs and no separate delta-detection logic exists.
    """
```

**Partial-failure handling.** If any of the three per-exchange calls fails,
the whole step is reported to `batch_runs` as `status='failed'` for
`stage='premarket_trading_api_symbol_map'` — a 2-of-3 success is not
written as a partial update. Yesterday's `trading_api_symbol` /
`trading_api_exchange` values are left untouched either way, never cleared
on failure — the same "preserve the last-known-good value rather than
silently run on an incomplete refresh" posture as the quarantine and
corporate-events handling elsewhere in this file.

Writes its own `batch_runs` row (`stage='premarket_trading_api_symbol_map'`,
`date=today`) — `status='running'` at start, `status='success'`/`'failed'`
at completion, same convention as the other premarket stages.

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

## Overnight Token Refresh

**Not a crawl. It shares this tool's process and scheduler, nothing else** —
placed here because a scheduled entry point already exists, and a bare cron
job would have no `batch_runs` marker and so no way to be noticed when it
fails.

The trading API's access token lives 24 hours from issuance, and a reissue
inside that window returns the SAME token with the SAME expiry — the only
way to obtain a fresh 24 hours is to revoke and reissue. Left alone, the
expiry therefore drifts a little each day and eventually lands inside a
trading session. Refreshing at a FIXED time removes that class of failure
outright: expiry becomes a constant, always the same wall-clock time on the
following day, and never inside a session.

```bash
# Overnight token refresh — config token_refresh_time (this file's Config
# Keys), default "03:00" America/New_York. ONE STAGE PER ACCOUNT: the same
# structure duplicated, not one stage extended.
python tools/collect_daily.py \
    --db-path data/market.duckdb \
    --token-refresh --account production
python tools/collect_daily.py \
    --db-path data/market.duckdb \
    --token-refresh --account demo
```

The one-per-minute issuance limit is PER ACCOUNT (measured), so the two do
not interfere and no ordering between them is fixed here. Separate stages
also give each its own idempotency skip and its own retry budget, which is
why no partial-failure rule is needed.

**Why after midnight, and why that early.** After midnight because the
previous evening run can overrun past its own deadline, and because
`batch_runs.date` must then be the trading day the stage targets rather
than the previous one. That early because refresh cadence and token
lifetime are both 24 hours, so the outgoing token expires at the very
moment the new one is requested — there is no previous token to fall back
on, and a failed refresh blocks the entire day. The only remedy is retry
against a one-per-minute issuance limit, which makes lead time the sole
recovery capacity; the dead window between the evening batch and the
premarket batch is where the most of it is available.

**Idempotency.** Each account's stage writes its own marker —
`stage='overnight_token_refresh_production'` or
`'overnight_token_refresh_demo'`, `'running'` at
its start, then `'success'` or `'failed'` — and is SKIPPED when today's row
for THAT stage already exists. A duplicate revoke-and-reissue would spend
the issuance budget and can leave no usable token at all.

**Not a gate.** The marker is for observability and the skip above, not for
blocking. A failed refresh followed by a batch's own automatic reissue
leaves a token that genuinely covers the session, and
`live_mode_runner.md`'s session-start gate tests that requirement directly
rather than inferring it from this stage's status. The failure still
surfaces through finding 1.

**Blast radius is bounded by P-5, not by luck.** Revocation is
account-wide, but the premarket batch, the live session and the evening
batch are already strictly serialised by their completion markers, so no
other API consumer exists at 03:00. Batch entry points keep the SDK's
automatic reissue enabled, having no WebSocket connection to disturb; the
live session does not (see `live_mode_runner.md`).

---

## Dual Schedule: Evening Full Run + Premarket Corporate-Events Refresh

The evening run (21:00 ET, below) computes `precomputed_session_stats` for
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

# Premarket recheck — MANUAL/DEBUG ONLY (R-1). The automated 09:20 ET
# recheck is now a LiveModeRunner in-process task (see
# live_mode_runner.md's "In-Process Premarket Recheck"). This flag remains
# for running the quotes/halt + anomaly-check pass by hand OUTSIDE a live
# session — it cannot open the DB read-write while LiveModeRunner holds it.
python tools/collect_daily.py \
    --db-path data/market.duckdb \
    --premarket-recheck
```

Timing: `--premarket-open` at 04:00 ET, well before premarket bar
accumulation begins. The 09:20 ET recheck (R-1: now in-process, not this
cron) is chosen as the latest point that comfortably fits its own cost
(quotes + halt fetch, sequential, ~150s each ≈ 300s total — see
`check_corporate_event_anomaly()` above) ahead of 09:30 regular-session
open, while getting as close to open as possible for freshness (premarket
liquidity, and therefore `bulk_fetch_today_first_price()` coverage, builds
through the morning — see that function's Coverage note). Deliberately NOT
a third, later recheck — a chain of ever-later passes trying to shave the
remaining gap closer to 09:30 reintroduces the same crawl-doesn't-fit
problem this 09:20 design avoids by excluding the full-universe crawl
entirely (item N's narrow/bulk crawls at 09:20 are targeted and cheap —
see live_mode_runner.md — not the same "third recheck" this paragraph
rejects).

`--premarket-open`'s `crawl_corporate_events()` step keeps the same
re-run safety as before, now via `upsert_corporate_event()` (item N)
rather than a bare `INSERT OR IGNORE` — safe to re-run manually (e.g.
via the legacy `--corporate-events-only` flag) if needed outside the
normal schedule.

**Two vendors, not a guarantee (item N).** Split/dividend effective dates
are always before-market-open by convention (see `db_schema.md`
`corporate_events`), but *when a vendor's own data reflects that event* is
outside this tool's control. The 04:00 pass now queries BOTH yfinance
(`crawl_corporate_events()`, per-ticker) AND investing.com
(`crawl_corporate_events_investing()`, single date-scoped bulk query) —
either catching an event the other missed closes that gap immediately,
since both run before Eager Pool consumes `corporate_events`. This is
meaningfully stronger than the single-vendor case, but not a guarantee: a
genuinely late update from BOTH vendors is still possible, and this
residual is what `check_corporate_event_anomaly()`'s price-gap detection
exists to catch (see below) — a defense that does not depend on either
crawl at all.

**Residual risk (narrowed by item N; fully closed only once R1 also
lands).** Patch N by itself closes the 04:00 gap (dual-vendor) and adds a
forward-looking investing.com check the evening before (see "Evening
Forward-Looking Corporate-Events Check" above) — but the 09:20 pass
described just above this paragraph is, by itself, still the OLD
`--premarket-recheck` (quotes/halt + anomaly-check only, no crawl of any
kind) until `r1_inprocess_recheck.patch` is also applied. That patch moves
this pass in-process and is what actually wires up the 09:20 investing
bulk crawl, the yfinance narrow crawl for newly-quarantined tickers, and
`caching_calculator.md`'s scoped mid-session recompute trigger (see that
file) — none of which fire from anything in THIS patch alone. Applied
together (N then R1, per the handoff's patch order), the intraday-staleness
gap that motivated this whole item is closed for anything either vendor
reflects by 09:20; `check_corporate_event_anomaly()`'s price-gap heuristic
remains the third and final defense layer for the residual neither vendor
catches in time.

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

# List unresolved trading-API symbol match candidates for manual review
# (see "Premarket Trading-API Symbol Map")
python tools/collect_daily.py --list-trading-api-symbol-candidates

# Manually register a trading-API symbol match the automated pass could
# not resolve
python tools/collect_daily.py --register-trading-api-symbol TICKER TRADING_API_SYMBOL EXCHANGE

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

# Premarket recheck — MANUAL/DEBUG ONLY (R-1). No longer the scheduled
# 09:20 ET entry — that is now a LiveModeRunner in-process task (see
# live_mode_runner.md's "In-Process Premarket Recheck"). This flag runs
# the quotes/halt + anomaly-check pass by hand, outside a live session.
python tools/collect_daily.py \
    --db-path data/market.duckdb \
    --premarket-recheck

# Overnight token refresh (see "Overnight Token Refresh" above and the cron
# entry below). Unlike its neighbours in this section, which are manual,
# debug or partial-run flags, this one's primary use is the scheduled
# entry — it is listed here for completeness, not because it is normally
# invoked by hand.
python tools/collect_daily.py \
    --db-path data/market.duckdb \
    --token-refresh
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
      via crawl_corporate_events(), which writes through the shared
      upsert_corporate_event() helper (item N) — safe to re-run; no
      fallback needed for either

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
                               (390 = an ordinary session; a capacity estimate,
                                not a per-date assertion)
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

**Evening job start gate (R-2: DuckDB-lock liveness probe).** Before its
first stage (metadata/corporate-events crawl) begins, the evening run
resolves LiveModeRunner's liveness — this closes the failure mode where a
crash (not a clean shutdown) turned into a silent, unbounded wait and,
via next-day Health Gate 1a, a two-day outage:
```
loop:   # the evening half of db_schema.md's DB file ownership windows
  if batch_runs has stage='live_session_end', date=today, status='success':
      → open RW, run normally.  # clean shutdown — the original path
  else:
      attempt to open market.duckdb read-WRITE:
        if open SUCCEEDS (no writer holds the lock):
            → LiveModeRunner is not running. Marker absent + lock free on
              a trading day past close = crashed (or never-started)
              session. KEEP this connection (do not close/reopen — avoids
              a race), record a health_report finding ("session end marker
              missing — LiveModeRunner did not shut down cleanly"), and
              run the evening batch normally — ingestion / session-stats
              inputs are runner-independent, so the run is valid.
              # R-9: the DB-only findings subset swept here now also
              # recovers 5, 8, 12, 21, 22, 24 and 25. Those were the
              # in-memory tallies this probe could never reach; findings
              # 5/8/21/22 now live in
              # live_session_state.session_diagnostics and 12/24/25 in
              # health_events, so a crashed session's findings are
              # recovered in full rather than in part — see
              # health_report.md's Invocation Contract.
        if open FAILS (lock held):
            → runner still alive (or hung). Sleep and retry, BUT: if now >
              evening_wait_hard_deadline (config, default 23:30 ET): alert
              + abort this evening run (manual intervention). A hung-past-
              deadline run aborts loudly tonight; tomorrow's Health Gate 1a
              then correctly aborts tomorrow on missing session stats — the
              right fail-safe, now reached noisily instead of silently.
```
This is the DuckDB-ownership-handoff half of P-5's temporal ownership
design (the other half — LiveModeRunner waiting on the premarket marker
before opening its own connection — is in live_mode_runner.md's Session
Start Gating). The original reasoning ("don't write while the runner
might still be writing") is preserved exactly — the loop still only
proceeds once the lock is free, either via the clean-shutdown marker or
via the lock-probe fallback above.

```bash
# Run after the day's JSON dump completes (weekdays only), 21:00 ET — NOT
# merely after market close: the dump ends at 20:00 ET, which is also the
# ingestion range's own upper bound (see Constraints)
# Full run: metadata + corporate events + ingest + calendar update + session stats
0 21 * * 1-5 cd /path/to/stock-scalping && \
    source .venv/bin/activate && \
    python tools/collect_daily.py \
        --db-path data/market.duckdb \
        --data-root /path/to/json/data \
        --date today >> logs/collect_daily.log 2>&1

# Overnight token refresh — 03:00 ET, weekdays
# Weekday-only is correct: no API consumer exists at the weekend (both
# other entries are 1-5), and Monday 03:00 has no token to revoke at all —
# the one issued Friday expired Saturday — so it is a pure issue, which is
# the safer path rather than a gap.
0 3 * * 1-5 cd /path/to/stock-scalping && \
    source .venv/bin/activate && \
    python tools/collect_daily.py \
        --db-path data/market.duckdb \
        --token-refresh >> logs/collect_daily_token.log 2>&1

# Premarket open (N-1) — replaces separate --ticker-rename-only and
# --corporate-events-only cron entries previously here, both at 04:00 ET
# (see "Dual Schedule" above and Constraints for the collision this
# merge removes)
0 4 * * 1-5 cd /path/to/stock-scalping && \
    source .venv/bin/activate && \
    python tools/collect_daily.py \
        --db-path data/market.duckdb \
        --premarket-open >> logs/collect_daily_premarket.log 2>&1

# (R-1: the 09:20 ET recheck is no longer a cron entry. It now runs as a
# LiveModeRunner in-process task — see live_mode_runner.md's "In-Process
# Premarket Recheck." --premarket-recheck survives as a manual/debug flag
# only; it cannot open the DB read-write during a live session.)
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
# R-2: hard deadline for the evening job's lock-probe wait (see Cron /
# Scheduler Setup's "Evening job start gate"). A lock-held (hung-runner)
# evening job aborts with an alert past this wall-clock time.
# Sized against the 21:00 ET start, not independently: a deadline at the
# start time itself would leave no wait at all, and exceeding it costs two
# days of operation (this run aborts, and tomorrow's Health Gate 1a then
# aborts on missing session stats), so a transient lock must not reach it.
evening_wait_hard_deadline: "23:30"   # ET

# Wall-clock time of the overnight token refresh (see "Overnight Token
# Refresh"). America/New_York, like every other time in this system.
# AFTER MIDNIGHT deliberately: the evening run can overrun past its own
# deadline above, and batch_runs.date must be the trading day the stage
# targets rather than the previous one. Early rather than close to the
# session because refresh cadence and token lifetime are both 24h, so a
# failed refresh has no previous token to fall back on and its only remedy
# is retry against a one-per-minute issuance limit — lead time IS recovery
# capacity.
token_refresh_time: "03:00"           # ET

# trading_api_quotes_url removed. The bulk today-vs-yesterday price quote
# that check_corporate_event_anomaly() (P-8) uses is reached through
# TradingAPI like every other vendor call; endpoint addressing is the SDK's
# (docs/api/trading_api.md's Configuration Ownership). This crawler running
# standalone, outside any LiveModeRunner session, does not change that — the
# module is a layer, not a session-scoped object.
#
# bulk_api_chunk_size removed with it. Chunking sits behind TradingAPI,
# which knows the endpoint's published symbol cap, so callers pass a ticker
# list of any length. The key's own value was estimated against an assumed
# ~100 tickers/sec ceiling — a figure the vendor publishes per endpoint and
# the SDK paces against automatically.

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
- corporate_events writes go through the shared upsert_corporate_event()
  helper (item N), not a bare INSERT OR IGNORE — unaffected by the
  stock_meta schema change (V-1), a separate table
- Ingestion logic delegates to `migrate_json_to_duckdb.py` — no duplicated logic
- Calendar/coverage update always runs after ingestion — not optional
- Session stats update runs after calendar/coverage update — not optional unless --skip-session-stats
- Session stats update always uses the EVENING run's corporate_events state —
  the ordering (crawl → ingest → calendar/coverage → session stats) within a
  single evening run must not change; the premarket refresh is intentionally
  a separate, later, supplementary crawl that session_stats does NOT re-read
- `populate_trading_calendar()`, `populate_ticker_coverage()`, and `populate_precomputed_session_stats()` sourced from `utils.py`
- The evening run writes seven `batch_runs` rows as it progresses —
  `stage='evening_ingestion'` (around Steps 3-4 above), `stage='evening_tick_bar_aggregates'`
  (Tick Bar Aggregates Update), `stage='evening_session_stats'` (Session Stats
  Update), `stage='evening_investing_forward_check'` (Evening
  Forward-Looking Corporate-Events Check, item N),
  `stage='evening_feed_coverage'` (Feed Coverage Analysis),
  `stage='evening_detection_gap'` (Detection-Gap Analysis), and finally
  `stage='evening_retention_purge'` (Retention Purge, R-9 — last, after
  every other stage, being the only destructive one) — each
  `status='running'` at its own start, `'success'`/`'failed'`
  at its own completion, independent of the others. Each also writes its own
  `SUMMARY` line to the run log (see Output / Logging), since
  `health_report.md` treats a missing `SUMMARY` for a stage that has a
  `batch_runs` row as an abnormal-termination signal in its own right. This
  lets LiveModeRunner's Health Gate 1 (live_mode_runner.md) distinguish
  exactly which stage of an evening run failed, not just that the run as a
  whole did.
- `--premarket-open` (N-1; replaces running `--ticker-rename-only` and
  `--corporate-events-only` as two separate cron processes at 04:00 ET)
  runs `detect_rename_candidates()` → `build_trading_api_symbol_map()` →
  `bulk_fetch_today_first_price()` +
  `utils.query_halt_status()` → `crawl_corporate_events()` for all tickers
  → `check_corporate_event_anomaly()`, as one sequential process — no
  metadata fields, no ingestion, no calendar/session-stats steps. Two
  separate OS processes both writing `ticker_cik_map` at the same 04:00
  slot was a DuckDB single-writer collision risk (P-5) — merging into one
  process removes the collision class entirely rather than coordinating
  around it. Writes `batch_runs` rows for `stage='premarket_rename'`,
  `stage='premarket_trading_api_symbol_map'`,
  `stage='premarket_corporate_events'`, and `stage='premarket_quarantine_check'`
  sequentially as each internal step completes, preserving the same
  per-stage failure granularity Health Gate 1 relies on (see
  live_mode_runner.md) even though it is now one process, not three.
  `--ticker-rename-only` and `--corporate-events-only` remain available as
  lower-level flags for manual re-run/debugging of one piece in isolation
  — only the automated cron schedule moves to `--premarket-open`.
- The 09:20 ET recheck (R-1: now a LiveModeRunner in-process task — see
  live_mode_runner.md's "In-Process Premarket Recheck"; `--premarket-recheck`
  itself survives only as a manual/debug flag, not the automated path) runs
  `bulk_fetch_today_first_price()` + `utils.query_halt_status()` →
  `check_corporate_event_anomaly()` (full-universe fresh re-evaluation) →
  `crawl_corporate_events_investing()` (item N bulk vendor) → a yfinance
  narrow crawl for any newly-quarantined ticker (item N) → a scoped-recompute
  trigger for tickers that gained a new same-day corporate_events row (item
  N, caching_calculator.md). Before doing any of its own work, the task
  checks `batch_runs` for `stage='premarket_corporate_events', date=today`;
  if `status='running'` (the 04:00 `--premarket-open` process is still
  mid-crawl), it defers rather than contend — the recheck-vs-04:00 guard is
  retained unchanged. The recheck-vs-LiveModeRunner-writer contention that
  motivated moving this in-process in the first place is not merely guarded
  against but dissolved outright: all writes go through this process's own
  `db_write()` funnel (see live_mode_runner.md), so there is no second
  writer to collide with. Writes its own `batch_runs` row:
  `stage='premarket_quarantine_recheck'`.
- Must be safe to run multiple times on the same date (idempotent) —
  `--premarket-open`'s four internal steps are each independently
  idempotent (`crawl_corporate_events()`'s `upsert_corporate_event()`
  write path — a no-op on an already-agreeing row, item N;
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
- On a non-trading day (market holiday, weekend), the two CRAWL crons
  above still fire and complete quickly as effective no-ops (no tickers
  to detect renames for, no new bars to crawl against) — each still
  writes its normal `batch_runs` row with `status='success'`. The token
  refresh is the exception and is NOT a no-op: it revokes and reissues on
  any day it fires, which is intended. Skipping it on non-trading days was
  rejected — that would add a `trading_calendar` dependency at 03:00 to
  save one wasted issuance. This is
  intended: `trading_calendar`'s `is_trading_day` is what downstream
  consumers (Health Gate 1, LiveModeRunner's own start condition — not
  specified here, out of scope) check to decide whether to actually run a
  session that day, not the presence or absence of a `batch_runs` row.
- Failed metadata tickers do not block ingestion — two steps are independent
- No session time filter on ingested data — all periods (040000~200000) stored.
  The bound stays fixed on an early-close day; see the Time range filter note
- The evening cron must not start before that day's JSON dump has finished.
  On an early-close day the dump completes at 17:00 rather than 20:00, so the
  21:00 slot gains margin instead of losing it. THE CRON IS NOT MOVED EARLIER TO
  MATCH: the truncation risk below is one-sided — waiting longer than necessary
  costs nothing, while ingesting before the dump finishes is permanent. This is
  why the exit path and the ingestion path do NOT share one boundary key even
  though both concern the after-market end.
  The ingestion range's upper bound (`200000`) and the dump's completion are
  the same instant by construction, so the 21:00 ET slot is derived from that
  boundary rather than chosen independently — if the dump moves, this cron
  moves with it. This is load-bearing because the duplicate-skip rule above
  is keyed on `(ticker, date)`: a date ingested while its after-market window
  was still being written is skipped by every later run, so the truncation is
  permanent rather than self-healing.
- The evening run may cross midnight while the lock probe waits — with the
  deadline at 23:30 ET that is an ordinary outcome, not an anomaly. The date
  basis is therefore fixed once at process start and never re-evaluated:
  `--date today` resolves to the trading day the process began on, for every
  stage and every `batch_runs` row the run writes.
- US stock splits and ex-dividend adjustments always take effect before market
  open — no intra-session event handling needed
- Vendor update latency (yfinance not yet reflecting a same-day corporate
  event by the time either premarket crawl runs) is a known, accepted
  limitation — not resolvable by adjusting crawl timing further; downstream
  consumers (filter E, gap_percentile dividend_amount, meta resolution) treat
  this the same as any other missing-data case, not as an error
