# Script: run_preprocess.py

**File:** `scripts/run_preprocess.py`

---

## Role

CLI entry point that orchestrates the full preprocessing pipeline.
Supports two execution modes:
- **Standalone**: full pipeline, saves parquet to disk
- **Optimizer (in-memory)**: returns full labeled DataFrame without splitting or saving

Preprocessor is a training-only component. Live inference uses
`FeatureExtractor.extract()` directly via Inferencer — not Preprocessor.

Session_mode filtering, temporal splitting, and rolling fold generation
are handled entirely by ClassBalancer.

---

## Input / Output

**Session-window exclusion.** This entry point opens `market.duckdb` and so must NOT run while LiveModeRunner holds the connection — see db_schema.md's DB file ownership windows.

**Input:**
```python
# From DuckDB:
ohlcv_1min:                  pd.DataFrame   # all sessions (no time filter)
tick_10:                     pd.DataFrame   # all sessions — full day per ticker/date
stock_meta:                  pd.DataFrame
trading_halts:               pd.DataFrame
trading_calendar:            pd.DataFrame
ticker_data_coverage:        pd.DataFrame
precomputed_session_stats:   pd.DataFrame   # REFERENCE_SESSION baselines
                                            # loaded per date range from DuckDB
```

**Output (standalone mode):**
```python
# Saved to data/processed/:
train_features.parquet
val_features.parquet
test_features.parquet
```

**Output (in-memory mode, return_data=True):**
```python
pd.DataFrame  # full labeled feature matrix, unsplit
```

---

## Pipeline Steps

```
[Standard / In-memory mode]

1. Connect to DuckDB and load all data:
       ohlcv_df    = SELECT * FROM ohlcv_1min   ORDER BY ticker, date, hour
       ticks_df    = SELECT * FROM tick_10       ORDER BY ticker, date, hour, seq_id
       meta_df     = SELECT * FROM stock_meta    ORDER BY ticker, date
                     # (ticker, date)-keyed since stock_meta schema update —
                     # multiple rows per ticker now; NOT a single per-ticker snapshot
       halts_df    = SELECT * FROM trading_halts
       calendar_df = SELECT * FROM trading_calendar
       closes, _   = session_boundaries_from_frame(calendar_df)
                     # utils.md — derived from the frame just loaded, not a
                     # second query. The after_hours_end map is discarded: the
                     # only consumer of it on this path is the Labeler, which
                     # derives its own from calendar_df.
                     # calendar_df is still passed on in its own right, for
                     # has_data in Labeler's dead position resolution, which
                     # the maps do not carry.
       coverage_df = SELECT * FROM ticker_data_coverage
       corp_events_df = SELECT * FROM corporate_events
                     # small table — loaded in full, filtered internally by
                     # Labeler (dead position Case A/D) and by FeatureExtractor's
                     # Strategy A/D (bars adjustment, gap_percentile dividend_amount)

       # REFERENCE_SESSION baselines — bulk load once, shared across all tickers/dates
       session_stats_raw = SELECT * FROM precomputed_session_stats
                           WHERE n_sessions = config["indicators"]["reference_session"]["n_sessions"]
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

2. EntryPointDetector.scan() for each ticker → entry_points (all sessions)
   late-day exclusion applied inside scan() when max_entry_offset_minutes
   is not null (01_entry_detection.md)
   scan() retrieves p_entry from bars[i+1]["open"] (t bar open price)

   # scan() meta (V-1): date-keyed, {date: {"shares_outstanding": value}}
   # keyed on bars_for_ticker["date"].unique() — ALL dates present in this
   # ticker's bars, since entry dates aren't known until scan() runs
   # (narrower than this would be circular). Deliberately separate from
   # Step 6's ticker_meta (4 fields, keyed on entry dates only, a smaller
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

3. Save entry_points to DuckDB entry_points table (INSERT OR IGNORE)

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
       (corp_events_df: full table passed through, same pattern as calendar_df/
        coverage_df — Labeler filters internally to the overnight window it needs
        for dead position Case A/D dividend/split adjustment)

5. Save labeled_samples to DuckDB labeled_samples table (INSERT OR IGNORE)

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
       halts_td  = halts_df filtered to (ticker, date)
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

8a. If return_data=True (optimizer mode):
    → return full_labeled_df directly (no split, no parquet save)

8b. If return_data=False (standalone mode):
    → ClassBalancer.split(
           balance=config["class_balancer"]["apply_balance"],
           session_mode=config["entry_detector"]["session_mode"]
       ) → (train_balanced, val, test)
    → save parquet files to data/processed/
```

---

## Class Interface

```python
class Preprocessor:
    def __init__(self, config: dict, db_conn: duckdb.DuckDBPyConnection): ...

    def run(
        self,
        return_data: bool = False,
    ) -> pd.DataFrame | None:
        """
        return_data=False : split via ClassBalancer.split(), save parquet, return None.
                            DB tables always written (INSERT OR IGNORE).
        return_data=True  : return full_labeled_df (unsplit), no parquet save.
                            DB tables always written (INSERT OR IGNORE).

        Training mode only. Live inference uses FeatureExtractor.extract() directly
        via Inferencer — Preprocessor is not involved in live inference.
        """
        ...

    def save(
        self,
        train: pd.DataFrame,
        val: pd.DataFrame,
        test: pd.DataFrame,
        run_id: str | None = None,
    ) -> None:
        """
        Save parquet files to data/processed/.
        Called internally by run() in standalone mode (return_data=False).
        If run_id provided, saves to data/processed/{run_id}/ for trial isolation.
        """
        ...
```

---

## CLI Interface

```bash
python scripts/run_preprocess.py [--config CONFIG]
```

CLI always runs in standalone mode (`return_data=False`).

---

## Config Keys Used

```yaml
data_paths.*
entry_detector.*
indicators.*            # includes reference_session.n_sessions, reference_session.delta_minutes
vectorizer.*
labeler.*
class_balancer.*
misc.lookback_bars
```

---

## Constraints

- Preprocessor is responsible for feature extraction and labeling only
- Session_mode filtering, splitting, and fold generation handled by ClassBalancer
- All data loaded from DuckDB in bulk (no per-ticker DB queries during extraction)
- `precomputed_session_stats` loaded once at Step 1 for all dates and tickers
- `build_session_stats_dict()` returns `{as_of_date: {ticker: {metric: {hour: value}}}}`
- Per-ticker session_stats dispatch at Step 6: `session_stats.get(date, {}).get(ticker)`
  produces flat `{metric: {hour: value}}` or `None` (ticker absent for that date —
  should not occur in normal operation since `populate_precomputed_session_stats()`
  runs for every ticker with ohlcv_1min data, but handled gracefully regardless);
  collected into the date-keyed dict passed to `extract_batch()` as `session_stats`
- Empty `precomputed_session_stats` is not an error: `extract_batch()` receives `None`, REFERENCE_SESSION indicators return NaN
- `ticks_df` loaded as full day per ticker/date:
  - Labeler receives full day (hour >= t_hour filtered internally per entry point)
  - FeatureExtractor receives the unbounded per-ticker frame and slices per
    entry point internally
- `halts_df` loaded per ticker/date and passed explicitly to both Labeler and FeatureExtractor
- `coverage_df` loaded once at step 1 and passed explicitly to Labeler.label()
- `corp_events_df` loaded once at step 1 (full table) and passed explicitly to
  Labeler.label() (Case A/D adjustment) — FeatureExtractor's corporate-event bars
  adjustment and gap_percentile dividend_amount lookup query `corporate_events`
  independently inside `extract_batch()` (see `04_feature_extractor.md`), not via
  a value passed in from this script
- `ticker_meta` (Step 6) is date-keyed, not a flat per-ticker dict — `stock_meta`'s
  schema is `(ticker, date)`-keyed (see V-1 fix); per-field fallback to
  `utils.estimate_historical_meta()` when a field is missing for a given date;
  `sector` always uses the ticker's most recent available snapshot regardless of date
- `scan_meta` (Step 2) is a separate, smaller date-keyed dict (shares_outstanding
  only) built BEFORE entry_points exists, keyed on all dates in the ticker's
  bars — not the same object as `ticker_meta`, which is keyed on entry dates
  only and includes 3 additional fields; the two are deliberately not unified
  since entry dates aren't known until after `scan()` returns
- Ticker rename stitching (Step 6): `get_ticker_history()` returns `None` for the
  vast majority of tickers (no query cost beyond the small `ticker_history` lookup);
  when it returns a history, predecessor-ticker rows are sliced from the
  already-loaded `ohlcv_df` (same table, different ticker label) and relabeled —
  no additional `ohlcv_1min` query needed
- `extract_batch()` called per ticker — no Python loop over `extract()` per entry point
- Empty DataFrames handled gracefully (saves _empty.parquet)
- `ClassBalancer.split()` called only in standalone mode
- `ClassBalancer.generate_folds()` is never called inside Preprocessor
- `p_entry` included in parquet as identifier column, not feature
- `is_dead_position`, `dead_position_case`, `is_ambiguous` stored as metadata columns
- entry_points and labeled_samples DB saves use INSERT OR IGNORE for idempotency
- Late-day exclusion applied inside EntryPointDetector.scan(), and only when
  `entry_detector.max_entry_offset_minutes` is not null
- Standalone CLI always calls split() — split_method config enforced by PipelineOptimizer only
- live_mode is NOT supported — Preprocessor is training-only
