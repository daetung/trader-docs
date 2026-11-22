# Script: run_preprocess.py

**File:** `scripts/run_preprocess.py`

---

## Role

CLI entry point that orchestrates the full preprocessing pipeline. The
pipeline runs as a guest_supervisor.md work — a step tool with a sealed
result (Work, below):
- the work's batches write:
  - `entry_points`;
  - `labeled_samples`;
  - the batch matrix files;
- the work is sealed once every batch is done;
- a new run's standalone part, outside the work, loads the sealed result,
  splits it and saves parquet to disk;
- pipeline_optimizer.md reads its input work's sealed result through the
  result loader (Result Loader, below) and constructs no Preprocessor.

A Preprocessor work is executed only by a `run_guest run_preprocess`
process.

Preprocessor is a training-only component. Live inference uses
`FeatureExtractor.extract()` directly via Inferencer — not Preprocessor.

Temporal splitting and rolling fold generation are handled entirely by
ClassBalancer. Entry points are regular-session only, fixed at `scan()`
(data_boundary.md); no session filter applies anywhere downstream.

---

## Input / Output

**DB access role.** A step guest (db_schema.md's DB File Ownership Windows;
db_coordination.md's Unit scope), run through guest_supervisor.md's
`run_guest`:
- `run_guest run_preprocess` enters
  `register('guest', 'run_preprocess', trigger)` and hands the body a work
  opener and a unit opener;
- every unit the unit opener opens is a `guest_unit('run_preprocess', None)`,
  waiting for admission without limit;
- the body opens its work in its first unit, before its first step; each
  step holds a read unit and a write unit, its compute running between them
  outside any unit (guest_supervisor.md's Steps; Work, below);
- a step's DB writes form one transaction committed in its write unit;
- the body calls neither `register()` nor `guest_unit()`;
- `scripts/run_preprocess.py` run directly hands over to
  `run_guest run_preprocess`;
- a run ended by self-termination leaves its work to resume through
  guest_supervisor.md;
- it writes (Work, below):
  - `entry_points`;
  - `labeled_samples`;
  - `derived_configs`;
  - `precomputed_session_stats`, through the batch's ensure.

**Input:**
```python
# From DuckDB, per batch, in the batch's read unit (Pipeline Steps, Step 1),
# for its tickers and the chain symbols of each of their segments:
ohlcv_1min:                  pd.DataFrame   # the batch's load range, all sessions
tick_10:                     pd.DataFrame   # the batch's load range — full day per ticker/date
stock_meta:                  pd.DataFrame
trading_halts:               pd.DataFrame   # the load range and the halt_open_into
                                            # walk's earlier dates
corporate_events:            pd.DataFrame   # dated through the load range end
fundamentals_quarterly:      pd.DataFrame   # dated through the load range end
fx_rates:                    pd.DataFrame   # [load range start −
                                            #  fx_rates.max_staleness_days,
                                            #  load range end]
trading_calendar:            pd.DataFrame   # through the batch's load range end
ticker_data_coverage:        pd.DataFrame
precomputed_session_stats:   pd.DataFrame   # REFERENCE_SESSION baselines
                                            # the chunk dates, under its config hash
```

**Output (the work):**
- `entry_points` and `labeled_samples` rows (Pipeline Steps 3 and 5);
- per batch, `data/processed/resume/{work_id}/batch_{group}_{chunk}.parquet`;
- the seal, `data/processed/resume/{work_id}/_done.json`.

**Output (standalone part):**
```python
# Saved to data/processed/:
train_features.parquet
val_features.parquet
test_features.parquet
```

---

## Work

Config hash and fixed inputs (guest_supervisor.md's Config Hash and Fixed
Inputs):
- `config_hash` covers the keys Config Keys Used lists inside config_hash,
  with code_versions holding the data version (utils.md's Config Hashing);
- the pin:
  - `data_paths.db_path`;
  - the trading-date range of `ohlcv_1min` at work creation;
  - the sorted ticker list in that range and its hash — the batches' tickers
    are read from the pin, and New-Run Resolution's same-pin test compares
    the hash;
- every execution of the work reads its fixed inputs — the config snapshot
  and the pin — never the current config; the batches' tickers and chunk
  dates come from the pin.

Opening — a new run's first unit (a `--resume-work` run is handed its work
already opened):
- computes the run's pin;
- opens its work through the work opener with that pin, by
  guest_supervisor.md's New-Run Resolution:
  - the reuse case's stale-batch check — the batch done test's staleness,
    below, over the 'done' work's batch files — runs on this unit's
    connection;
  - a reused result ends the work part, nothing computed;
  - a run resolved as a consumer holds no work: it releases this unit and
    waits holding no unit, ending `'blocked'` when the awaited work turns
    (guest_supervisor.md's Dependency):
    - `'halted'`;
    - superseded;
    - `'done'` with an invalid seal.

Batches — a step is one batch (ticker group, date chunk):
- the pinned tickers, sorted, grouped by `preprocess.resume_batch_tickers`;
- the pinned trading dates cut into consecutive chunks of
  `preprocess.resume_batch_dates`;
- the processing unit is a security segment: a batch loads, beside its
  tickers, the chain symbols of each of their segments (utils.md's
  symbology layer);
- the load range: [the first element of utils.prior_trading_days(chunk
  start, `indicators.lookback_days`, closes), the next trading day with
  `has_data` after the chunk end among the pinned trading dates — the chunk
  end when none];
- a batch's reads end at its load range end, never after the pin's last
  trading date, for:
  - `ohlcv_1min`;
  - `tick_10`;
  - `trading_calendar`;
- Labeler Case C applies where no trading day with `has_data` follows
  within the pinned dates.

Each batch is a `step()` (guest_supervisor.md's Steps), done the batch done
test (below):
1. the read unit — when done, the unit is released without computing;
   otherwise its `read`:
   - ensures `precomputed_session_stats` for its tickers × chunk dates under
     the work's fixed config, through `populate_precomputed_session_stats()`
     (utils.md; db_schema.md's Derived tables, Production);
   - takes its mark — the highest `source_changes` seq and that row's
     `recorded_at` (db_schema.md's Derived tables, Mark);
   - loads its inputs (Pipeline Steps, Step 1), including corporate_events,
     fundamentals_quarterly, fx_rates, ticker_history and the scan meta; the
     halt_open_into walk beyond the load range runs here and loads its rows;
2. compute, holding no unit — Pipeline Steps 1–7 in memory for its tickers,
   except Step 6's market_cap and shares_outstanding estimates, read by
   `estimate_historical_meta()` in one unit per ticker that a connection
   provider (the unit opener) opens; emitting for the chunk dates only:
   - entry points;
   - labels;
   - features;
   the batch matrix — an empty batch included, as a zero-row file carrying
   the full schema dtypes — written to a temp file;
3. `s.write()` — the write unit: `entry_points` and `labeled_samples`
   (Steps 3 and 5), the batch file's rename to
   `data/processed/resume/{work_id}/batch_{group}_{chunk}.parquet` before
   the commit, `work_progress()`, the commit.
The batch file's parquet metadata carries:
   - work_id;
   - group;
   - chunk;
   - the row count;
   - the key-set hash;
   - the schema hash — the hash of `get_feature_names()`' ordered list with
     `get_feature_schema()`;
   - the mark;
   - the exclusion count of each bar-availability reason: no bar at t,
     insufficient bars (01_entry_detection.md);
   - the count of (ticker, date) whose first regular-session bar is not at
     093000.

Batch done test — a batch is done when:
- its file passes guest_supervisor.md's result file check;
- it is not stale (db_schema.md's Derived tables, Stale), by its mark and
  its footprint;
- every (ticker, date, hour) key in the file exists in `entry_points` and,
  under the work's config_hash, in `labeled_samples`, provisional rows
  included (anti-join count 0).
Any other batch is processed again, its file replaced.

A batch file's footprint:
- the chain symbols of its segments;
- its load range, for:
  - `ohlcv_1min`;
  - `tick_10`;
  - `ticker_data_coverage`;
  - `trading_calendar`;
  - `trading_halts`;
  - `ticker_history`;
- `trading_halts` and `ohlcv_1min` from the earliest date the resolver's
  halt_open_into walk read;
- `corporate_events`, `fundamentals_quarterly` and `fx_rates` dated through
  the load range end;
- the `precomputed_session_stats` footprints of its tickers × chunk dates,
  the group present or absent.
`sector` and `security_type`, read from the latest `stock_meta` snapshot,
and the `stock_meta` rows read for a date, all dated before it, are outside
it.

Seal — a `db_step()`, once every batch passes the batch done test, a
batch failing it being processed again first; its body runs every batch
done test and writes:
- `data/processed/resume/{work_id}/_done.json`, written through temp +
  rename, listing the batch files with their row counts and key-set hashes;
then `work_close('done')` follows, the seal's hash recorded on the work
(guest_supervisor.md's Sealed Results).

Standalone part — a new run, outside the work, after its work part, whose
result is one of:
- sealed by the run;
- reused;
- awaited as a consumer.
The standalone part:
- loads the result through `load_result()` (Result Loader, below);
- splits it — `ClassBalancer.split(balance=class_balancer.apply_balance)` →
  (train_balanced, val, test);
- saves through `save()`, each file through temp + rename.
A `--resume-work` run ends at its work part.

---

## Result Loader

`Preprocessor.load_result(work_id, seal_hash)` — the one reader of a
Preprocessor result, used by the standalone part and by
pipeline_optimizer.md:
1. the seal check — guest_supervisor.md's valid seal, against `seal_hash`;
2. each listed file's result file check (guest_supervisor.md);
3. concat;
4. dtypes from `get_feature_schema()`;
5. sort by (date, ticker, hour).

- `seal_hash` is the caller's record of the work's seal:
  - pipeline_optimizer.md's — from its fixed inputs (`input_works`);
  - the standalone part's — from the work its work part:
    - opened;
    - reused;
    - awaited.
- Empty batch files carry the full schema dtypes, so the concat keeps them.

---

## Pipeline Steps

```
[Per batch — Work, above: Step 1 in the batch's read unit, its connection
 the db_conn below; Steps 2–7 compute holding no unit; the writes of Steps 3
 and 5 and the batch file's rename in the batch's write unit]

1. Load the batch — T = its tickers and the chain symbols of each of their
   segments, R = its load range (Work, above):
       ohlcv_df    = SELECT * FROM ohlcv_1min   WHERE ticker IN T AND date IN R
                     ORDER BY ticker, date, hour
       ticks_df    = SELECT * FROM tick_10       WHERE ticker IN T AND date IN R
                     ORDER BY ticker, date, hour, seq_id
                     # R's start lies indicators.lookback_days trading days
                     # before the chunk start; a prev_close with no valid bar
                     # within lookback_days before the entry date is NaN
       meta_df     = SELECT * FROM stock_meta    WHERE ticker IN T ORDER BY ticker, date
                     # (ticker, date)-keyed since stock_meta schema update —
                     # multiple rows per ticker now; NOT a single per-ticker snapshot
       halts       = utils.halt_intervals(ticker, dates of R, src) per ticker
                     # the halt-interval resolver (utils.md): stored rows read
                     # through the symbology layer; src the frames loaded here
                     # and the DB outside R — the halt_open_into walk beyond R
                     # runs in this read unit and loads its rows
       calendar_df = SELECT * FROM trading_calendar WHERE date <= <R's end>
                     # R's end bounds the work's dataset (Work, above)
       closes, ends = session_boundaries_from_frame(calendar_df)
                     # utils.md — derived from the frame just loaded, not a
                     # second query. closes and ends are kept: ends goes to scan()
                     # and to extract_batch().
                     # calendar_df is still passed on in its own right, for
                     # has_data in Labeler's dead position resolution, which
                     # the maps do not carry, and for the calendar context
                     # (utils.calendar_context(), Step 6).
       coverage_df = SELECT * FROM ticker_data_coverage WHERE ticker IN T
       corp_events_df = SELECT * FROM corporate_events WHERE ticker IN T
                     # small table — the batch tickers' rows; its readers are
                     # the Labeler (dead position Case A/D) and
                     # FeatureExtractor's frame-backed reader
       fundamentals_df = SELECT * FROM fundamentals_quarterly WHERE ticker IN T
                     # dated through R's end; served through FeatureExtractor's
                     # reader
       fx_rates_df = SELECT * FROM fx_rates
                     WHERE date >= <R's start − fx_rates.max_staleness_days days>
                       AND date <= <R's end>
                     # served through FeatureExtractor's reader

       # REFERENCE_SESSION baselines — the chunk dates, after the batch's
       # ensure (Work, above); stats_config_hash = utils.config_hash() of
       # populate_precomputed_session_stats()'s Config Keys Used under the
       # work's config
       session_stats_raw = SELECT * FROM precomputed_session_stats
                           WHERE config_hash = stats_config_hash
                             AND as_of_date IN <chunk dates> AND ticker IN T
                           ORDER BY as_of_date, ticker, hour, metric

       # Apply delta smoothing in memory
       # session_stats: {as_of_date: {ticker: {metric: {hour: smoothed_avg_value}}}}
       session_stats = build_session_stats_dict(
           session_stats_raw,
           delta_minutes=config["indicators"]["reference_session"]["delta_minutes"],
           closes=closes,
           ends=ends,
       )
       # If precomputed_session_stats is empty (e.g., baseline not yet computed):
       # session_stats = {}  → extract_batch() receives session_stats=None per ticker
       # REFERENCE_SESSION indicators return NaN (not an error)

2. EntryPointDetector.scan() for each ticker → entry_points (regular-session
   only — scan()'s first exclusion step), kept for the chunk dates only
   scan() reads the stitched frame (the ticker's segments relabelled to it)
   and entries are kept only for dates inside the scanned symbol's own
   segment
   history, bar-availability and late-day exclusions applied inside scan()
   (01_entry_detection.md)
   scan() retrieves p_entry from bars[i+1]["open"] (t bar open price)

   # scan() meta (V-1): date-keyed, {date: {"shares_outstanding": value}}
   # keyed on bars_for_ticker["date"].unique() — ALL dates present in this
   # ticker's bars, since entry dates aren't known until scan() runs
   # (narrower than this would be circular). Deliberately separate from
   # Step 6's ticker_meta (market_cap, shares_outstanding, price_52h,
   # price_52l, sector and security_type, keyed on entry dates only, a
   # smaller and cheaper set once entry_points exists) — not unified into
   # one dict. Its estimates run in the read unit (Work, above).
   scan_meta = {
       date: {
           # estimate_historical_meta()'s tiers, its first the stock_meta
           # prior-day row (data_boundary.md's "Point-in-time correctness
           # for stock_meta"); None when unresolved — filter E false
           "shares_outstanding": utils.estimate_historical_meta(
               ticker, date, "shares_outstanding", closes,
               config["indicators"]["lookback_days"], db_conn
           )
       }
       for date in bars_for_ticker["date"].unique()
   }
   detector.scan(bars_for_ticker, ticker, meta=scan_meta, closes=closes,
                 ends=ends)

3. Save entry_points to DuckDB entry_points table (INSERT OR IGNORE) —
   p_entry bound as `Decimal` through V1 into its DECIMAL(18,6) column (V4,
   utils.md's Numeric Rules)

4. Labeler.label() for all entry points → labeled_samples
       Per ticker/date:
           ohlcv_future_td = ohlcv_df filtered to (ticker, date, hour >= t_hour)
           ticks_td        = ticks_df filtered to (ticker, date), full day
           halts_td        = the resolver's intervals for (ticker, date)
       labeler.label(
           entry_points_td, ohlcv_future_td, ticks_td, halts_td,
           calendar_df, coverage_df, corp_events_df
       )
       (includes is_dead_position, dead_position_case, is_ambiguous)
       (corp_events_df: the batch's frame passed through, same pattern as calendar_df/
        coverage_df — Labeler selects dead position Case A's event rows by the
        overnight window and passes the whole frame to utils.md's frame forms,
        since a restated dividend's split factor can need rows after D+1)

5. Save labeled_samples through the derived-table replacement (db_schema.md's
   Derived tables): the work's derived_configs row (INSERT OR IGNORE) first,
   then each key under the work's config_hash:
   - absent → inserted;
   - stored provisional or stale → replaced;
   - otherwise → kept;
   each written row with is_provisional = (dead_position_case = 'C') and
   source_seq = the batch's mark seq; p_entry bound as `Decimal` through V1,
   as in step 3

6. FeatureExtractor.extract_batch() for each ticker — the extractor
   constructed with a frame-backed reader over the read unit's
   corp_events_df, fundamentals_df and fx_rates_df:
       segments  = the ticker's ticker_segments() (read in the read unit)
       bars_td   = utils.slice_ticker_rows(ohlcv_df, ticker, segments)
       # No date or hour axis: the batch carries several entry dates
       # (ticker_dates below) and several entry hours, so neither is unique
       # within it. extract_batch() slices per entry point internally; that
       # docstring is the contract's single site.

       # Ticker rename stitching: slice_ticker_rows() relabels each
       # segment's symbol rows within its date range to the current ticker,
       # over the read unit's frames — the chain symbols' rows are already
       # present in the in-memory frames, so no DB query is needed here.

       ticks_td  = utils.slice_ticker_rows(ticks_df, ticker, segments)
       halts_td  = the resolver's intervals for the ticker, covering every
                   date of the frame
       ticker_dates = entry_points_td["date"].unique()

       # Per-date meta resolution (V-1): a single flat meta dict per ticker
       # would apply one date's values to every entry point of that ticker
       # regardless of its own date, which is exactly the staleness problem
       # V-1 fixes. Build a date-keyed dict, mirroring the session_stats
       # dispatch pattern below — data_boundary.md's "Point-in-time
       # correctness for stock_meta", the prior-day row being the stock_meta
       # row dated utils.prior_trading_days(date, 1, closes)[0]:
       ticker_meta = {
           date: {
               # the prior-day row's value, else estimate_historical_meta(),
               # read in one unit per ticker that the connection provider opens
               "market_cap": ...,
               # estimate_historical_meta(), its first tier the prior-day row
               "shares_outstanding": ...,
               # the metric of the batch's loaded precomputed_session_stats
               # group of (ticker, date) at hour '000000'; none → NaN
               "price_52h": ...,
               "price_52l": ...,
               # no derivation path — the chain-latest symbol's latest row
               # with the field set, whatever the date
               "sector": ...,
               "security_type": ...,
           }
           for date in ticker_dates
       }
       # The instrument rule (utils.md): a non-common security_type resolves
       # shares_outstanding and market_cap None, the prior-day row's values
       # included. A value may be None — a NaN feature.

       # Fundamentals-derived yield features (earnings_yield, book_to_price,
       # sales_to_price, dilution_rate, cash_to_mcap, debt_to_mcap — see
       # 04_feature_extractor.md Meta Features) are resolved separately inside
       # extract_batch() itself, over the reader's fundamentals_quarterly and
       # fx_rates rows, via utils.get_ttm_value() / an as-of
       # most-recent-filed lookup — not added to ticker_meta here.
       # ticker_meta stays scoped to its stock_meta-sourced and
       # caller-filled fields — market_cap, shares_outstanding, price_52h,
       # price_52l, sector, security_type; fundamentals resolution does not
       # depend on entry_date-keyed dict construction, since as-of filtering
       # happens per-lookup inside extract_batch().

       # Supply REFERENCE_SESSION baselines for this ticker's dates
       # session_stats format: {as_of_date: {ticker: {metric: {hour: value}}}}
       # Dispatch: select date and ticker dimensions for this specific ticker
       ticker_session_stats = {
           date: session_stats.get(date, {}).get(ticker)
           for date in ticker_dates
       }
       # ticker_session_stats: {date: {metric: {hour: value}} | None}
       # extract_batch() internally selects stats[entry_date] per entry point

       extractor.extract_batch(
           entry_points_td, bars_td, ticks_td, ticker_meta, halts_td,
           closes=closes,
           ends=ends,
           calendar_context=utils.calendar_context(calendar_df, ticker_dates),
           session_stats=ticker_session_stats,
       )

7. Merge features with labels on (ticker, date, hour)
   → labeled feature matrix:
     [ticker, date, hour, p_entry]
     + [features...]
     + [labels, is_dead_position, dead_position_case, is_ambiguous]
   → regular-session entries only, as scan() returned them
   → the chunk dates only; is_provisional and source_seq are not in it
     (labeled_samples columns only)

8. The batch matrix → the batch file (Work, above)
```

---

## Class Interface

```python
class Preprocessor:
    def __init__(self, config: dict, unit_opener): ...
        # config: the work's fixed config snapshot; unit_opener: run_guest's,
        # each batch's unit opened from it (Work, above)

    def run(self, work) -> None:
        """
        Run the opened work's batches and seal it (Work, above). Each batch's
        DB writes — entry_points (INSERT OR IGNORE), labeled_samples (the
        derived-table replacement) — are one transaction committed at the
        batch's end; Preprocessor issues no other commit.

        Training mode only. Live inference uses FeatureExtractor.extract() directly
        via Inferencer — Preprocessor is not involved in live inference.
        """
        ...

    @staticmethod
    def load_result(work_id: str, seal_hash: str) -> pd.DataFrame:
        """The Preprocessor result loader (Result Loader, above)."""
        ...

    def save(
        self,
        train: pd.DataFrame,
        val: pd.DataFrame,
        test: pd.DataFrame,
    ) -> None:
        """
        Save parquet files to data/processed/, each through temp + rename.
        Called by the standalone part, after load_result() and the split
        (Work, above).
        """
        ...
```

---

## CLI Interface

```bash
run_guest run_preprocess [--trigger scheduled] [--supersede] [--config CONFIG]
run_guest run_preprocess --resume-work <work_id>   # the sweep's command
# python scripts/run_preprocess.py <args> hands over to run_guest run_preprocess <args>
```

A new run runs its standalone part after its work part; a `--resume-work`
run ends at its work part (Work, above).

---

## Config Keys Used

```yaml
data_paths:                       # declared here; no other file declares it
  db_path:      data/market.duckdb
  processed_dir: data/processed
  model_dir:    models
  log_dir:      logs

preprocess:                       # declared here
  resume_batch_tickers:           # tickers per batch group; default set by measurement
  resume_batch_dates:             # trading dates per chunk; default set by measurement
```

Inside config_hash (guest_supervisor.md's Config Hash and Fixed Inputs):
- `entry_detector.*`
- `indicators.*`, including:
  - `reference_session.n_sessions`
  - `reference_session.delta_minutes`
  - `lookback_days`
- `vectorizer.*`
- `labeler.*`
- `execution.max_hold_minutes`
- `corporate_events.dividend_restated_sources`
- `corporate_events.dividend_withholding_rate`
- `halts.continuation`
- `fx_rates.max_staleness_days`
- `fundamentals.annual_fallback_max_age_months`
Health Gate 1d's feature-contract config_hash (live_mode_runner.md; utils.md's
deployability_checks()) is taken over this list, `labeler.*` excluded.

Outside config_hash:
- `data_paths.db_path`
- `data_paths.processed_dir`
- `data_paths.model_dir`
- `data_paths.log_dir`
- `class_balancer.*`
- `preprocess.resume_batch_tickers`
- `preprocess.resume_batch_dates`

---

## Constraints

- Preprocessor is responsible for feature extraction and labeling only
- Splitting and fold generation handled by ClassBalancer; no session filter
  applies (entries are regular-session only from `scan()`)
- Data loaded from DuckDB per batch, in bulk for the batch's tickers and load
  range, in its read unit (Step 1); the DB reads extraction itself makes:
  - `estimate_historical_meta()`: the scan meta's in the read unit, Step 6's
    in provider units;
  - `get_ticker_history()`: the read unit;
  - `corporate_events`: the reader
- `precomputed_session_stats` loaded per batch at Step 1 — the chunk dates and
  the batch tickers under its config hash — after the batch's ensure
- `build_session_stats_dict()` returns `{as_of_date: {ticker: {metric: {hour: value}}}}`
- Per-ticker session_stats dispatch at Step 6: `session_stats.get(date, {}).get(ticker)`
  produces flat `{metric: {hour: value}}` or `None` (ticker absent for that date —
  should not occur in normal operation since the batch ensures those groups
  through `populate_precomputed_session_stats()` first, but handled
  gracefully regardless);
  collected into the date-keyed dict passed to `extract_batch()` as `session_stats`
- Empty `precomputed_session_stats` is not an error: `extract_batch()` receives `None`, REFERENCE_SESSION indicators return NaN
- `ticks_df` loaded as full day per ticker/date:
  - Labeler receives full day (hour >= t_hour filtered internally per entry point)
  - FeatureExtractor receives the unbounded per-ticker frame and slices per
    entry point internally
- Halt intervals come from the halt-interval resolver in the batch's read
  unit and are passed explicitly to both Labeler (per (ticker, date)) and
  FeatureExtractor (per ticker — every date of its frame)
- `coverage_df` loaded per batch at step 1 and passed explicitly to Labeler.label()
- `corp_events_df` loaded per batch at step 1 (the batch tickers' rows) and passed explicitly to
  Labeler.label() (Case A/D adjustment); FeatureExtractor takes its rows through
  its reader (see `04_feature_extractor.md`). In both, a 'dividend' row's amount
  is read through `dividend_gross_from_events()`, never from `amount` directly
- `fx_rates` rows loaded per batch at Step 1, served through FeatureExtractor's
  reader
- `ticker_meta` (Step 6) is date-keyed, not a flat per-ticker dict: per field,
  `market_cap` the stock_meta prior-day row's value else
  `utils.estimate_historical_meta()`, `shares_outstanding` that function's
  tiers (its first the prior-day row), `price_52h` and `price_52l` the loaded
  precomputed_session_stats group's day-level metric; the instrument rule
  applied; `sector` and `security_type` always use the chain-latest symbol's
  latest row with the field set, whatever the date
- `scan_meta` (Step 2) is a separate, smaller date-keyed dict (shares_outstanding
  only) built BEFORE entry_points exists, keyed on all dates in the ticker's
  bars — not the same object as `ticker_meta`, which is keyed on entry dates
  only and holds market_cap, shares_outstanding, price_52h, price_52l, sector
  and security_type; they are deliberately not unified
  since entry dates aren't known until after `scan()` returns
- Ticker rename stitching (Step 6): `utils.slice_ticker_rows()` over the read
  unit's frames — the chain symbols' rows are already loaded, relabelled per
  segment; no additional query needed
- `extract_batch()` called per ticker — no Python loop over `extract()` per entry point
- Empty DataFrames handled gracefully in the standalone part's split output
  (saves _empty.parquet); an empty batch writes a zero-row batch file (Work)
- `ClassBalancer.split()` called only in the standalone part
- `ClassBalancer.generate_folds()` is never called inside Preprocessor
- `p_entry` included in parquet as identifier column, not feature
- `is_dead_position`, `dead_position_case`, `is_ambiguous` stored as metadata columns
- `entry_points` saves use INSERT OR IGNORE (Step 3); `labeled_samples` saves use
  the derived-table replacement (Step 5)
- `is_provisional` and `source_seq` are `labeled_samples` columns only — absent
  from the in-memory labeled frame and the batch matrix; feature metadata
  exclusion lists are unchanged
- A batch's DB writes are one transaction committed at its end; Preprocessor
  issues no other commit
- Every execution of the work reads its fixed inputs, never the current config;
  the batches' tickers and chunk dates come from the pin
- Calls neither `register()` nor `guest_unit()` — `run_guest` opens its units
- Late-day exclusion applied inside EntryPointDetector.scan(), and only when
  `entry_detector.max_entry_offset_minutes` is not null
- The standalone part always calls split() — split_method config enforced by PipelineOptimizer only
- live_mode is NOT supported — Preprocessor is training-only
