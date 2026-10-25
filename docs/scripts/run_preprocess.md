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

Session_mode filtering, temporal splitting, and rolling fold generation
are handled entirely by ClassBalancer.

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
- the body opens its work in its first unit, before its first step, and
  holds one unit per step (Work, below);
- a step's DB writes form one transaction committed at the step's end, on
  the connection that step's unit opens;
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
# From DuckDB, per batch, on the batch's unit connection (Pipeline Steps, Step 1):
ohlcv_1min:                  pd.DataFrame   # the batch's load range, all sessions
tick_10:                     pd.DataFrame   # the batch's load range — full day per ticker/date
stock_meta:                  pd.DataFrame
trading_halts:               pd.DataFrame
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
- `config_hash` covers the keys Config Keys Used lists inside config_hash;
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
- the load range: [chunk start − `indicators.lookback_days` trading days,
  the next trading day with `has_data` after the chunk end among the pinned
  trading dates — the chunk end when none];
- a batch's reads end at its load range end, never after the pin's last
  trading date, for:
  - `ohlcv_1min`;
  - `tick_10`;
  - `trading_calendar`;
- Labeler Case C applies where no trading day with `has_data` follows
  within the pinned dates.

Each batch, in its own unit:
1. checks whether it is done (batch done test, below), and if so releases
   the unit without computing;
2. ensures `precomputed_session_stats` for its tickers × chunk dates under
   the work's fixed config, through `populate_precomputed_session_stats()`
   (utils.md; db_schema.md's Derived tables, Production);
3. takes its mark — the highest `source_changes` seq and that row's
   `recorded_at` (db_schema.md's Derived tables, Mark);
4. loads and runs Pipeline Steps 1–7 for its tickers, emitting for the
   chunk dates only:
   - entry points;
   - labels;
   - features;
5. writes `entry_points` and `labeled_samples` (Steps 3 and 5), commits,
   and records progress (`work_progress()`);
6. writes the batch matrix — an empty batch included, as a zero-row file
   carrying the full schema dtypes — to a temp file renamed to
   `data/processed/resume/{work_id}/batch_{group}_{chunk}.parquet`, its
   parquet metadata carrying:
   - work_id;
   - group;
   - chunk;
   - the row count;
   - the key-set hash;
   - the schema hash — the hash of `get_feature_names()`' ordered list with
     `get_feature_schema()`;
   - the mark.

Batch done test — a batch is done when:
- its file passes guest_supervisor.md's result file check;
- it is not stale (db_schema.md's Derived tables, Stale), by its mark and
  its footprint;
- every (ticker, date, hour) key in the file exists in `entry_points` and,
  under the work's config_hash, in `labeled_samples`, provisional rows
  included (anti-join count 0).
Any other batch is processed again, its file replaced.

A batch file's footprint:
- its tickers and their `ticker_history` predecessors;
- its load range, for:
  - `ohlcv_1min`;
  - `tick_10`;
  - `ticker_data_coverage`;
  - `trading_calendar`;
  - `trading_halts`;
  - `ticker_history`;
- `corporate_events` and `fundamentals_quarterly` dated through the load
  range end;
- the `precomputed_session_stats` footprints of its tickers × chunk dates,
  the group present or absent.
Values read from the latest `stock_meta` snapshot — `sector`, and
`estimate_historical_meta()`'s latest-snapshot tier with its split
adjustment — are outside it.

Seal — in its own unit, once every batch passes the batch done test, a
batch failing it being processed again first:
- `data/processed/resume/{work_id}/_done.json`, written through temp +
  rename, lists the batch files with their row counts and key-set hashes;
- `work_close('done')`, the seal's hash recorded on the work
  (guest_supervisor.md's Sealed Results).

Standalone part — a new run, outside the work, after its work part, whose
result is one of:
- sealed by the run;
- reused;
- awaited as a consumer.
The standalone part:
- loads the result through `load_result()` (Result Loader, below);
- splits it — `ClassBalancer.split(balance=class_balancer.apply_balance,
  session_mode=entry_detector.session_mode)` → (train_balanced, val, test);
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
[Per batch — Work, above: run inside the batch's unit, its connection the
 db_conn below]

1. Load the batch — T = its tickers and their ticker_history predecessors,
   R = its load range (Work, above):
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
       halts_df    = SELECT * FROM trading_halts WHERE ticker IN T
       calendar_df = SELECT * FROM trading_calendar WHERE date <= <R's end>
                     # R's end bounds the work's dataset (Work, above)
       closes, _   = session_boundaries_from_frame(calendar_df)
                     # utils.md — derived from the frame just loaded, not a
                     # second query. The after_hours_end map is discarded: the
                     # only consumer of it on this path is the Labeler, which
                     # derives its own from calendar_df.
                     # calendar_df is still passed on in its own right, for
                     # has_data in Labeler's dead position resolution, which
                     # the maps do not carry.
       coverage_df = SELECT * FROM ticker_data_coverage WHERE ticker IN T
       corp_events_df = SELECT * FROM corporate_events WHERE ticker IN T
                     # small table — the batch tickers' rows; its only reader on this
                     # path is the Labeler (dead position Case A/D).
                     # FeatureExtractor queries corporate_events itself
                     # (Strategy A/D — see Constraints below)

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
           session_mode=config["entry_detector"]["session_mode"],
           closes=closes,
       )
       # If precomputed_session_stats is empty (e.g., baseline not yet computed):
       # session_stats = {}  → extract_batch() receives session_stats=None per ticker
       # REFERENCE_SESSION indicators return NaN (not an error)

2. EntryPointDetector.scan() for each ticker → entry_points (all sessions),
   kept for the chunk dates only
   late-day exclusion applied inside scan() when max_entry_offset_minutes
   is not null (01_entry_detection.md)
   scan() retrieves p_entry from bars[i+1]["open"] (t bar open price)

   # scan() meta (V-1): date-keyed, {date: {"shares_outstanding": value}}
   # keyed on bars_for_ticker["date"].unique() — ALL dates present in this
   # ticker's bars, since entry dates aren't known until scan() runs
   # (narrower than this would be circular). Deliberately separate from
   # Step 6's ticker_meta (market_cap, shares_outstanding, price_52h,
   # price_52l and sector, keyed on entry dates only, a smaller
   # and cheaper set once entry_points exists) — not unified into one dict.
   scan_meta = {
       date: {
           "shares_outstanding": (
               meta_df.loc[(ticker, date), "shares_outstanding"]
               if (ticker, date) in meta_df.index
                  and pd.notna(meta_df.loc[(ticker, date), "shares_outstanding"])
               else utils.estimate_historical_meta(
                   ticker, date, "shares_outstanding", db_conn
               )
           )
       }
       for date in bars_for_ticker["date"].unique()
   }
   detector.scan(bars_for_ticker, ticker, meta=scan_meta, closes=closes)

3. Save entry_points to DuckDB entry_points table (INSERT OR IGNORE) —
   p_entry bound as `Decimal` through V1 into its DECIMAL(18,6) column (V4,
   utils.md's Numeric Rules)

4. Labeler.label() for all entry points → labeled_samples
       Per ticker/date:
           ohlcv_future_td = ohlcv_df filtered to (ticker, date, hour >= t_hour)
           ticks_td        = ticks_df filtered to (ticker, date), full day
           halts_td        = halts_df filtered to (ticker, date)
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

6. FeatureExtractor.extract_batch() for each ticker:
       bars_td   = ohlcv_df filtered to (ticker)
       # No date or hour axis: the batch carries several entry dates
       # (ticker_dates below) and several entry hours, so neither is unique
       # within it. extract_batch() slices per entry point internally; that
       # docstring is the contract's single site.

       # Ticker rename stitching (Item 4): if get_ticker_history(ticker, db_conn)
       # is not None (the common case is None — no query cost beyond the small
       # ticker_history lookup), additionally slice ohlcv_df for each
       # predecessor_ticker (dates before that predecessor's effective_date),
       # relabel the ticker column to the current ticker, and prepend to
       # bars_td. Predecessor rows are already present in the in-memory
       # ohlcv_df under their original symbol — no additional DB query needed
       # for the bars themselves, only for the small ticker_history table.

       ticks_td  = ticks_df filtered to (ticker)
       halts_td  = halts_df filtered to (ticker)   # every entry date of this
                                                  # extract_batch() call
       ticker_dates = entry_points_td["date"].unique()

       # Per-date meta resolution (V-1): stock_meta is (ticker, date)-keyed —
       # a single flat meta dict per ticker would apply one date's values to
       # every entry point of that ticker regardless of its own date, which is
       # exactly the staleness problem V-1 fixes. Build a date-keyed dict,
       # mirroring the session_stats dispatch pattern below:
       ticker_meta = {
           date: {
               field: (
                   meta_df.loc[(ticker, date), field]
                   if (ticker, date) in meta_df.index and pd.notna(meta_df.loc[(ticker, date), field])
                   else utils.estimate_historical_meta(ticker, date, field, db_conn)
               )
               for field in ["market_cap", "shares_outstanding", "price_52h", "price_52l"]
           } | {
               # sector has no derivation path — most recent available snapshot
               # regardless of entry date (see 04_feature_extractor.md Meta Features)
               "sector": meta_df[meta_df["ticker"] == ticker].sort_values("date")["sector"].iloc[-1]
           }
           for date in ticker_dates
       }

       # Fundamentals-derived yield features (earnings_yield, book_to_price,
       # sales_to_price, dilution_rate, cash_to_mcap, debt_to_mcap — see
       # 04_feature_extractor.md Meta Features) are resolved separately from
       # fundamentals_quarterly inside extract_batch() itself, via
       # utils.get_ttm_value() / an as-of most-recent-filed lookup — not
       # added to ticker_meta here. ticker_meta stays scoped to the original
       # 5 stock_meta-sourced fields; fundamentals_quarterly resolution does
       # not depend on entry_date-keyed dict construction the way stock_meta
       # did, since as-of filtering happens per-lookup inside extract_batch().

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
           session_stats=ticker_session_stats,
       )

7. Merge features with labels on (ticker, date, hour)
   → labeled feature matrix:
     [ticker, date, hour, p_entry]
     + [features...]
     + [labels, is_dead_position, dead_position_case, is_ambiguous]
   → contains all sessions; no session_mode filter applied here
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
- `execution.max_hold_bars`
- `corporate_events.dividend_restated_sources`
- `corporate_events.dividend_withholding_rate`

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
- Session_mode filtering, splitting, and fold generation handled by ClassBalancer
- Data loaded from DuckDB per batch, in bulk for the batch's tickers and load
  range (Step 1); the DB reads extraction itself makes:
  - `estimate_historical_meta()`;
  - `get_ticker_history()`;
  - FeatureExtractor's own `corporate_events` reads
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
- `halts_df` loaded per batch and passed explicitly to both Labeler (per
  (ticker, date)) and FeatureExtractor (per ticker — every entry date of its
  `extract_batch()` call)
- `coverage_df` loaded per batch at step 1 and passed explicitly to Labeler.label()
- `corp_events_df` loaded per batch at step 1 (the batch tickers' rows) and passed explicitly to
  Labeler.label() (Case A/D adjustment) — FeatureExtractor's corporate-event bars
  adjustment and gap_percentile dividend_amount lookup query `corporate_events`
  independently inside `extract_batch()` (see `04_feature_extractor.md`), not via
  a value passed in from this script. In both, a 'dividend' row's amount is read
  through utils.md's dividend amount function — `dividend_gross_amount()` in
  FeatureExtractor, its frame form `dividend_gross_from_events()` in the
  Labeler — never from `amount` directly
- `ticker_meta` (Step 6) is date-keyed, not a flat per-ticker dict — `stock_meta`'s
  schema is `(ticker, date)`-keyed (see V-1 fix); per-field fallback to
  `utils.estimate_historical_meta()` when a field is missing for a given date;
  `sector` always uses the ticker's most recent available snapshot regardless of date
- `scan_meta` (Step 2) is a separate, smaller date-keyed dict (shares_outstanding
  only) built BEFORE entry_points exists, keyed on all dates in the ticker's
  bars — not the same object as `ticker_meta`, which is keyed on entry dates
  only and holds market_cap, shares_outstanding, price_52h, price_52l and
  sector; the two are deliberately not unified
  since entry dates aren't known until after `scan()` returns
- Ticker rename stitching (Step 6): `get_ticker_history()` returns `None` for the
  vast majority of tickers (no query cost beyond the small `ticker_history` lookup);
  when it returns a history, predecessor-ticker rows are sliced from the
  already-loaded `ohlcv_df` (same table, different ticker label) and relabeled —
  no additional `ohlcv_1min` query needed
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
