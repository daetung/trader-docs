# Tool: Detection Benchmark

**File:** `tools/detection_benchmark.py`
**Standalone CLI tool**

---

## Role

Measure the speed and output of entry detection + labeling
across the full DuckDB dataset for a given set of filter thresholds.
Supports dry-run (timing only) and full-run (point counts per class) modes.

Primary use case: tune entry detection thresholds by observing
class sample counts and computation time before committing to a full pipeline run.

---

## CLI Usage

**DB access role.** A per-chunk guest (db_schema.md's DB File Ownership
Windows; db_coordination.md's Unit scope), run through guest_supervisor.md's
`run_guest`:
- `run_guest detection_benchmark` enters
  `register('guest', 'detection_benchmark', trigger)` and hands the body a
  unit opener, each call opening one `guest_unit('detection_benchmark', None)`,
  which waits for admission without limit;
- the body opens its units by Units, below, and calls neither `register()`
  nor `guest_unit()`;
- `tools/detection_benchmark.py` run directly hands over to
  `run_guest detection_benchmark`;
- while an owner waits past `coordination.guest_max_hold_seconds`, the run
  ends by self-termination and is re-run from scratch.

```bash
# python tools/detection_benchmark.py <args> hands over to
# run_guest detection_benchmark <args>

# Full run: detect + label, report class counts and timing
run_guest detection_benchmark \
    --db-path data/market.duckdb \
    --config configs/pipeline_config.yaml

# Dry run: timing estimate only (no labeling, count detections only)
run_guest detection_benchmark \
    --db-path data/market.duckdb \
    --config configs/pipeline_config.yaml \
    --dry-run

# Override specific filter thresholds inline (no config file edit needed)
run_guest detection_benchmark \
    --db-path data/market.duckdb \
    --config configs/pipeline_config.yaml \
    --override "entry_detector.B_min_volume_1min=80000" \
    --override "entry_detector.G_min_ratio=3.0"

# Limit to specific date range
run_guest detection_benchmark \
    --db-path data/market.duckdb \
    --config configs/pipeline_config.yaml \
    --date-from 20250101 --date-to 20250331

# Limit to N random tickers (for quick sanity check)
run_guest detection_benchmark \
    --db-path data/market.duckdb \
    --config configs/pipeline_config.yaml \
    --sample-tickers 100
```

---

## Implementation Strategy

Detection conditions A, B, C, D, E, G are expressed as DuckDB SQL window
functions and aggregations to maximize speed.
Condition F (proximity to daily max volume) uses a window function.

```sql
-- Example: condition G as SQL window function
WITH vol_avg AS (
    SELECT
        ticker, date, hour,
        volume,
        AVG(volume) OVER (
            PARTITION BY ticker, date
            ORDER BY hour
            ROWS BETWEEN 19 PRECEDING AND CURRENT ROW
        ) AS vol_ma_20
    FROM ohlcv_1min
    WHERE ticker IN (SELECT unnest($chunk))   -- the chunk's tickers (Units)
)
SELECT ticker, date, hour
FROM vol_avg
WHERE volume >= vol_ma_20 * 2.5   -- condition G threshold
```

```sql
-- Example: condition E (turnover rate) with per-date shares_outstanding —
-- a fast SQL approximation of utils.estimate_historical_meta(), without
-- split_basis_value() and without tier 1's implied shares, for benchmark
-- speed only. The canonical logic (used by 01_entry_detection.md in the
-- actual pipeline) lives in utils.py; small SQL/Python discrepancies here
-- are acceptable since this tool is diagnostic-only (threshold tuning), not
-- the training or live path itself. Per (ticker, date), in this order:
--   1. the stock_meta row dated the prior trading day by trading_calendar,
--      when no split or reverse_split lies in (that day, date];
--   2. else the latest positive fundamentals_quarterly shares_outstanding
--      with filed_date < date, on a row whose unit is 'shares', multiplied
--      by the split product over (its fiscal_period_end, date];
--   3. else the most recent stock_meta row dated before date, when no split
--      or reverse_split lies in (its date, date];
--   4. else NULL — condition E false.
-- The instrument rule, through the latest security_type: a non-common type
-- (warrant, unit, right, preferred, debt) → NULL in every tier; adr → tier 2
-- skipped, its other tiers standing.
--
-- Without this, a naive join against the current stock_meta snapshot would
-- apply today's share count to every historical date, systematically
-- under-counting turnover for dates before any split (see V-1 fix).
WITH chunk_ticker_dates AS (
    SELECT DISTINCT ticker, date FROM ohlcv_1min
    WHERE ticker IN (SELECT unnest($chunk))
),
trading_days AS (
    SELECT date, LAG(date) OVER (ORDER BY date) AS prior_date
    FROM trading_calendar
    WHERE is_trading_day
),
splits AS (
    SELECT ticker, event_date, ratio FROM corporate_events
    WHERE event_type IN ('split', 'reverse_split')
),
latest_type AS (
    -- the latest stock_meta row with security_type set, whatever the date
    SELECT ticker, arg_max(security_type, date) AS security_type
    FROM stock_meta
    WHERE security_type IS NOT NULL
    GROUP BY ticker
),
tier1 AS (
    -- the prior trading day's row, no split or reverse_split in (that day, date]
    SELECT d.ticker, d.date, m.shares_outstanding AS shares
    FROM chunk_ticker_dates d
    JOIN trading_days td ON td.date = d.date
    JOIN stock_meta m ON m.ticker = d.ticker AND m.date = td.prior_date
    WHERE m.shares_outstanding IS NOT NULL
      AND NOT EXISTS (
          SELECT 1 FROM splits s
          WHERE s.ticker = d.ticker
            AND s.event_date > td.prior_date AND s.event_date <= d.date
      )
),
tier2 AS (
    -- the latest positive 'shares' fact filed by date, times the split
    -- product over (its fiscal_period_end, date]; product via exp(sum(ln(.)))
    -- since ratios are always positive (DuckDB has no PRODUCT aggregate)
    SELECT d.ticker, d.date,
           f.value * COALESCE((
               SELECT EXP(SUM(LN(s.ratio))) FROM splits s
               WHERE s.ticker = d.ticker
                 AND s.event_date > f.fiscal_period_end AND s.event_date <= d.date
           ), 1.0) AS shares
    FROM chunk_ticker_dates d
    JOIN fundamentals_quarterly f
      ON f.ticker = d.ticker AND f.filed_date < d.date
    WHERE f.metric = 'shares_outstanding' AND f.unit = 'shares' AND f.value > 0
    QUALIFY ROW_NUMBER() OVER (
        PARTITION BY d.ticker, d.date
        ORDER BY f.filed_date DESC, f.fiscal_period_end DESC
    ) = 1
),
tier3 AS (
    -- the most recent earlier row, no split or reverse_split in (its date, date]
    SELECT d.ticker, d.date, m.shares_outstanding AS shares
    FROM chunk_ticker_dates d
    JOIN stock_meta m ON m.ticker = d.ticker AND m.date < d.date
    WHERE m.shares_outstanding IS NOT NULL
      AND NOT EXISTS (
          SELECT 1 FROM splits s
          WHERE s.ticker = d.ticker
            AND s.event_date > m.date AND s.event_date <= d.date
      )
    QUALIFY ROW_NUMBER() OVER (
        PARTITION BY d.ticker, d.date ORDER BY m.date DESC
    ) = 1
),
shares_at_date AS (
    SELECT d.ticker, d.date,
           CASE
               WHEN t.security_type IN ('warrant', 'unit', 'right',
                                        'preferred', 'debt') THEN NULL
               ELSE COALESCE(
                   t1.shares,
                   CASE WHEN t.security_type = 'adr' THEN NULL ELSE t2.shares END,
                   t3.shares
               )
           END AS shares_outstanding
    FROM chunk_ticker_dates d
    LEFT JOIN latest_type t ON t.ticker = d.ticker
    LEFT JOIN tier1 t1 ON t1.ticker = d.ticker AND t1.date = d.date
    LEFT JOIN tier2 t2 ON t2.ticker = d.ticker AND t2.date = d.date
    LEFT JOIN tier3 t3 ON t3.ticker = d.ticker AND t3.date = d.date
),
daily_volume AS (
    SELECT ticker, date, SUM(volume) AS vol
    FROM ohlcv_1min
    WHERE ticker IN (SELECT unnest($chunk))
      AND hour >= '040000'   -- volume_base_hour
    GROUP BY ticker, date
)
SELECT dv.ticker, dv.date
FROM daily_volume dv
JOIN shares_at_date sd ON sd.ticker = dv.ticker AND sd.date = dv.date
WHERE dv.vol / sd.shares_outstanding * 100 >= 5.0   -- condition E threshold;
                                                    -- NULL shares → false
```

The composite expression `(A AND B AND C AND D) AND (E OR F OR G)` is
assembled by combining SQL WHERE clauses per active condition.

---

## Units

The body opens its units through the unit opener:
- a first unit enumerates the run's tickers — those with data in the date
  range, or `--sample-tickers`' sample of them — and records the run's mark
  (Input Change, below);
- one unit per chunk of `detection_benchmark.chunk_tickers` tickers (config,
  default 1000), holding the chunk's reads:
  - its detection queries, each taking `ticker IN chunk`;
  - the Labeler's inputs (Constraints), each taking `ticker IN chunk` —
    the symbology layer's reads taking the chain symbols of the chunk's
    tickers' segments: the Labeler's next-day reads for Case A/B, which
    resolve the successor symbol, and the halt-interval resolver's stored
    rows;
- labeling runs on the loaded frames outside units;
- the decay sweep's sample, drawn over every chunk's labeled entry points,
  is read in units per chunk of its tickers the same way;
- a last unit counts the input change (Input Change, below).
The summary combines the chunks' results.

```yaml
detection_benchmark:
  chunk_tickers: 1000
```

---

## Input Change

A run reads across units, so its input can change between them:
- its first unit records the run's mark, the highest `source_changes` seq;
- its last unit counts the `source_changes` rows with a higher seq that
  intersect the run's read set:
  - `source` one of the tables it read;
  - `ticker` NULL or a symbol of the chains it read;
  - `date_from`–`date_to`, a NULL end open, holding a date it read;
- its printed summary flags a non-zero count with the count (Output).

---

## Output

**Dry run:**
```
Detection Benchmark (dry-run)
  Date range    : 20250101 ~ 20250630  (126 trading days)
  Tickers scanned: 11,847
  Ticker-days   : 1,492,722

  Estimated time (SQL-based detection): ~12 minutes
  [timing based on 1,000 ticker-day sample × extrapolation]

  Conditions active: A B C D E G  (F disabled)
  Expression: (A AND B AND C AND D) AND (E OR G)
```

**Full run:**
```
Detection Benchmark (full run)
  Detection elapsed : 11m 43s
  Labeling elapsed  : 4m 12s
  Total elapsed     : 15m 55s

  Entry points detected: 284,731

  Label distribution:
    label_up5  :  18,204  ( 6.4%)
    label_up3  :  52,811  (18.5%)
    label_sw   : 163,420  (57.4%)
    label_dn3  :  41,200  (14.5%)
    label_dn5  :   9,096  ( 3.2%)

  Sideways ratio: 57.4%  [threshold: 60%]  ← OK
  Minority class: label_dn5 = 9,096 samples

  Config used:
    B_min_volume_1min : 50000
    G_min_ratio       : 2.5
    [full config snapshot saved to tools/benchmark_results/YYYYMMDD_HHMMSS.json]
```

Either mode's output ends, when the input-change count is non-zero, with:
```
  Input changed: 3 source_changes rows since the run's mark
```

---

## Delayed-Entry Decay Sweep

**DIAGNOSTIC ONLY.** It sizes no threshold: `execution_common.md`'s
residual-edge gate divides by `(1 + d)` and so self-adjusts to whatever
drift distribution exists. What this sweep answers is how much traffic the
late-entry path would carry and whether the gate's inputs behave sanely
before it is switched on — not whether to switch it on, which is
`pipeline_optimizer.md`'s `execution_eval` question.

```bash
run_guest detection_benchmark \
    --db-path data/market.duckdb \
    --config configs/pipeline_config.yaml \
    --decay-sweep 0,5,10,20,30,40,50,60 \
    --decay-sample 5000
```

For each delta, re-label a stratified sample of detected entry points as if
entry had happened delta seconds after the t bar's open instead of at it.

**`Labeler` is NOT modified.** Its own `ticks_df` input is sliced to
`hour >= (t_open + delta)` using `tick_10`'s absolute `HHMMSS`, and the
anchor price is substituted; adjudication logic is reused untouched, which
is what this tool's "reuse, do not reimplement" constraint requires. TWO
things move together and moving only one is the trap: shifting the anchor
without shifting the scan start counts the movement DURING the wait as
movement AFTER entry, which reads as free profit.

**The anchor is bundle-resolution-bounded, and the bound is reported rather
than hidden.** `tick_10.hour` is the LAST tick of a bundle, so the target
instant usually falls inside one and is not recoverable:

```
p_lo(delta) = close of the last bundle at or before the target — under-reacts,
              OPTIMISTIC about how far price has already moved
p_hi(delta) = close of the first bundle at or after the target — PESSIMISTIC
```

Both are reported as an interval; the pessimistic `p_hi` is what any figure
downstream should be read against, and the interval WIDTH is itself the
diagnostic — a wide one means this measurement cannot carry weight at that
delta. Entry points are structurally in dense-bundle territory (condition B
requires 50,000 shares in the reference bar), so the interval should be
narrow where it matters, and is worth checking rather than assuming.

**Stratified sample, not the full set.** The per-entry-point slice boundary
differs for every point, so a shared `ticks_df` cannot express it and
batching is defeated — the full 284k-point set is not viable at this cost.
Sampling is stratified BY LABEL CLASS, because survival is a statistic and
stratification is what gets usable counts into `up5` and `dn5`.

Reported per delta: `survival_up5` / `survival_up3` (label still holds after
the wait), `already_moved` (the threshold was already crossed at
`t_open + delta`, so the move the detection was aiming at is spent),
`adverse` (price moved against the position during the wait), and
`pnl_delta_p50` / `_p95`.

Written to the same `tools/benchmark_results/` JSON as any other run, under
a `decay_sweep` key. Not to `experiment_log` — that table is for model runs.

---

## Result Persistence

Each run saves a result JSON to `tools/benchmark_results/`:

```json
{
  "run_at": "20250715_143022",
  "config_snapshot": { ... },
  "overrides": { "B_min_volume_1min": 80000 },
  "total_detections": 284731,
  "label_counts": { "up5": 18204, "up3": 52811, "sw": 163420, "dn3": 41200, "dn5": 9096 },
  "detection_time_s": 703,
  "labeling_time_s": 252
}
```

Benchmark results are NOT written to `experiment_log` (that table is for model runs only).

---

## Constraints

- Must not modify any DuckDB table — read-only access to:
  `ohlcv_1min`, `stock_meta`, `tick_10`, `trading_halts`,
  `trading_calendar`, `ticker_data_coverage`, `corporate_events`,
  `fundamentals_quarterly`, `ticker_history`, `source_changes`
- Labeling logic must call `Labeler` directly (reuse, do not reimplement) —
  including the decay sweep, which shifts `Labeler`'s INPUTS (sliced
  `ticks_df`, substituted anchor) and never its logic or its signature
- `Labeler.label()` requires `ticks_df`, `halts_df`, `trading_calendar`,
  `ticker_data_coverage`, and `corporate_events` — all must be loaded from
  DuckDB before calling (see 05_labeler.md — corporate_events added for
  dead position Case A/D dividend/split adjustment); `halts_df` is the
  halt-interval resolver's intervals for the date (utils.md's
  halt_intervals()); the next-day reads for Case A/B resolve the successor
  symbol through the symbology layer
- Condition E's SQL (per-date shares_outstanding: the prior trading day's
  stock_meta row, the fundamentals tier on 'shares' rows, the latest earlier
  stock_meta row, each guarded against splits; the instrument rule through
  the latest security_type) is a benchmark-speed approximation of
  `utils.estimate_historical_meta()` without `split_basis_value()` and
  without tier 1's implied shares — acceptable here since this tool is
  diagnostic-only, but `01_entry_detection.md` itself must call the real
  utility function, not reimplement this SQL
- `--override` applies in-memory only — config file is never modified
- SQL-based detection is mandatory for performance; Python-loop fallback only for unsupported conditions
- Benchmark results saved to `tools/benchmark_results/` — not to `experiment_log`
- Calls neither `register()` nor `guest_unit()` — the body opens its units
  through the unit opener `run_guest` hands it
