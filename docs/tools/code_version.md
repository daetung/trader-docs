# Tool: Code Version

**File:** `tools/code_version.py`
**Standalone CLI tool — opens no `market.duckdb`; the only writer of `src/code_versions.json`**

---

## Role

Maintains the code-version lock `src/code_versions.json` (utils.md's Code
Versions). Processes check the areas they compute from against the lock; an
area is unattested when its current code_hash differs from the lock's or the
lock has no entry for it. This tool is how an area's version moves and how a
changed area is attested.

The lock records, per area:
- version;
- code_hash;
- recorded_at;
- the environment snapshot: each listed distribution's version and the
  Python version.

It also holds the distribution list; default:
- numpy;
- pandas;
- polars;
- scipy;
- duckdb;
- lightgbm;
- scikit-learn;
- shap.

---

## CLI Usage

```bash
python tools/code_version.py bump <area>
python tools/code_version.py record <area>
python tools/code_version.py golden --refresh <area>
python tools/code_version.py distributions [--add <name>] [--remove <name>]
```

- `bump <area>`: version + 1, an area without a lock entry starting at 1;
  generates the new version's golden outputs, then records the hash and
  snapshot.
- `record <area>`: runs the area's golden test; records the hash and
  snapshot under the same version only when the test passes, and refuses
  otherwise.
- `golden --refresh <area>`: regenerates the current version's golden
  outputs, only when the area's current hash equals the lock's.
- `distributions`: edits the lock's distribution list.

---

## Golden Tests

Per area. The fixture: a curated source dataset in `tests/golden/fixtures/`,
loaded into a temporary DuckDB; the test calls the area's computations
directly.

Data-area outputs on the fixture:
- the batch matrix content;
- labeled_samples;
- precomputed_session_stats;
- tick_bar_aggregates;
- the live-path feature vectors of caching_calculator and inferencer for the
  fixture's entry points.

Training-area outputs, from the frozen input matrix
`tests/golden/training/input`:
- predictions on the fixture split;
- the reduction report and selected features;
- the backtest's experiment_log totals.

The training-area fixture config fixes LightGBM's:
- deterministic;
- num_threads;
- seed.

Comparison is exact for every type, NaN positions included. Golden outputs
are stored in `tests/golden/{area}/v{version}/` with the fixture hash. The
golden tests also run in the pytest suite.

---

## Constraints

- Only this tool writes `src/code_versions.json`
- `record` refuses unless the area's golden test passes
- `golden --refresh` refuses unless the area's current hash equals the lock's
