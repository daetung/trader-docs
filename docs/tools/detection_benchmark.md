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

```bash
# Full run: detect + label, report class counts and timing
python tools/detection_benchmark.py \
    --db-path data/market.duckdb \
    --config configs/pipeline_config.yaml

# Dry run: timing estimate only (no labeling, count detections only)
python tools/detection_benchmark.py \
    --db-path data/market.duckdb \
    --config configs/pipeline_config.yaml \
    --dry-run

# Override specific filter thresholds inline (no config file edit needed)
python tools/detection_benchmark.py \
    --db-path data/market.duckdb \
    --config configs/pipeline_config.yaml \
    --override "entry_detector.B_min_volume_1min=80000" \
    --override "entry_detector.G_min_ratio=3.0"

# Limit to specific date range
python tools/detection_benchmark.py \
    --db-path data/market.duckdb \
    --config configs/pipeline_config.yaml \
    --date-from 20250101 --date-to 20250331

# Limit to N random tickers (for quick sanity check)
python tools/detection_benchmark.py \
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
)
SELECT ticker, date, hour
FROM vol_avg
WHERE volume >= vol_ma_20 * 2.5   -- condition G threshold
```

The composite expression `(A AND B AND C AND D) AND (E OR F OR G)` is
assembled by combining SQL WHERE clauses per active condition.

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
  `trading_calendar`, `ticker_data_coverage`
- Labeling logic must call `Labeler` directly (reuse, do not reimplement)
- `Labeler.label()` requires `ticks_df`, `halts_df`, `trading_calendar`,
  and `ticker_data_coverage` — all must be loaded from DuckDB before calling
- `--override` applies in-memory only — config file is never modified
- SQL-based detection is mandatory for performance; Python-loop fallback only for unsupported conditions
- Benchmark results saved to `tools/benchmark_results/` — not to `experiment_log`
