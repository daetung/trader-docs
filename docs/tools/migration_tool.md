# Tool: JSON → DuckDB Migration

**File:** `tools/migrate_json_to_duckdb.py`
**Standalone CLI tool — no pipeline dependencies required**

---

## Role

One-time (and incremental) migration of existing JSON files into DuckDB.
Supports both directory and zip formats. Skips already-imported data.

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
     d. Filter to regular session: hour >= '093000' and hour <= '155900'
     e. For tick data: assign seq_id within each (ticker, date, hour) group
     f. Bulk insert into DuckDB (INSERT OR IGNORE)
4. Log: files processed / skipped / failed
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

## Output / Logging

```
Migration summary:
  Scanned  : 48,300 files
  Imported : 45,820 files  (ohlcv_1min: 24,100 | tick_10: 21,720)
  Skipped  : 2,480 files   (already in DB)
  Failed   : 0 files
  Duration : 14m 32s
  Rows inserted: ohlcv_1min=9,399,000 | tick_10=43,200,000
```

Failed files are logged to `tools/migration_errors.log` with filename and exception.

---

## Constraints

- Must be runnable independently from the pipeline (no src/ imports required)
- Duplicate skip is at `(ticker, date)` granularity per table — entire file skipped if any row exists
- Zip files: extract to tempdir, process, clean up — do not leave temp files
- Session filter applied at ingestion — pre/after market data discarded permanently
- Must handle malformed JSON files gracefully (log and continue, do not crash)
