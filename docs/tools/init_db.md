# Tool: Database Initialization

**File:** `tools/init_db.py`
**Standalone CLI tool — no pipeline dependencies required**

---

## Role

Creates the DuckDB database and every table in it. This is the **sole
DDL-issuing component in the system** — no other tool, script, or runtime
module issues `CREATE`, `ALTER`, or `DROP`. Every other component assumes
its tables already exist, and this is what makes that assumption true.

Peer of `migration_tool.md`, not a stage inside it. The new-database
workflow is:

```
tools/init_db.py            → creates the schema
tools/migrate_json_to_duckdb.py → ingests JSON into it
```

Neither imports the other. `migrate_json_to_duckdb.py` does not defensively
call this tool, and this tool knows nothing about ingestion — keeping them
separate means the destructive-looking one (schema creation) is always an
explicit, deliberate invocation rather than something that can happen as a
side effect of a data job.

**Concurrency assumption**: identical to `migration_tool.md`'s. Run
manually and rarely — in the ordinary case exactly once, when a database is
first created — never on an automated schedule, and never concurrently with
a live trading session or the evening/premarket batch jobs (see P-5's DB
ownership design in `live_mode_runner.md` / `metadata_crawler.md`, which
this tool does not participate in — no `batch_runs` marker writes here).
Running it while any of those hold the writer connection fails with a DuckDB
lock error rather than corrupting anything: an operator error to avoid, not
a case this tool detects or works around.

---

## Source of the DDL

`SCHEMA_STATEMENTS` is a list of SQL strings held in this script, transcribed
from **`db_schema.md`'s canonical DDL region** — the `sql` fence under its
`## Table Definitions` heading, which is the single source of truth for the
schema. That document states the transcription contract; this tool is its
only consumer.

The transcription is mechanical and must stay that way:

- copy each statement verbatim — never rewrite, reorder, or normalise
- drop `--` comments
- preserve document order
- split per statement

Statements are already written `CREATE TABLE IF NOT EXISTS` in `db_schema.md`,
so idempotency is not something this tool decides or adds.

**When `db_schema.md` changes, `SCHEMA_STATEMENTS` is re-transcribed from it.**
The design documents are maintained as a separate project from this codebase
and are read-only from here, so there is no build step, no generator, and no
runtime dependency on the documents being present on disk. The list in this
script IS the codebase's copy, and re-transcription is a deliberate step in
applying a schema change — see "Applying a schema change" below.

---

## CLI Usage

```bash
# Create the database and all tables (the ordinary case)
python tools/init_db.py --db-path data/market.duckdb

# Report drift between the live database and SCHEMA_STATEMENTS.
# Read-only: opens the DB, prints findings, changes nothing.
python tools/init_db.py --db-path data/market.duckdb --verify
```

---

## Processing Logic

```
1. Open data/market.duckdb read-write (creates the file if absent).

2. Execute every statement in SCHEMA_STATEMENTS, in order, inside a
   SINGLE TRANSACTION.
   # Atomicity matters here specifically: a partial failure must not leave
   # a half-built database that later tools would treat as valid. Either
   # every table exists or the file is untouched.

3. Commit, then report per table whether it was CREATED or ALREADY PRESENT.
   # IF NOT EXISTS makes step 2 a no-op for tables that already exist, so
   # re-running against a populated database is safe and never touches data.
```

---

## What this tool does NOT do

**It never alters an existing table.** A table already present is left
exactly as it is, even when `db_schema.md` has since changed its columns,
types, or primary key. `CREATE TABLE IF NOT EXISTS` does not compare the
existing table against the statement — it simply skips it.

This is a deliberate boundary, not an oversight. Schema changes in this
system are rare and consequential, and the operating model is a single
operator with a single database: an automatic migration path would be new
infrastructure serving a case that does not exist, while carrying the one
risk that actually matters here — silently rewriting a table that holds the
training corpus or the P&L record.

The consequence must be stated plainly, because it is the tool's one sharp
edge: **for anything other than a brand-new table, plain init is a silent
no-op.** Nothing errors, nothing warns, and the mismatch surfaces later as a
runtime "column not found" from whichever component tried to use the new
column. `--verify` exists so that silence is not the only option.

---

## `--verify` — read-only drift report

Introspects the live database (`information_schema` / `PRAGMA table_info`)
and compares it against `SCHEMA_STATEMENTS`. **It reports and does nothing
else** — no `CREATE`, no `ALTER`, no data movement, no writes of any kind.
That restraint is the point: it supports the operator's decision rather than
substituting for it, which is what keeps an incremental-migration path from
reappearing through the back door.

What it reports, by class of difference:

| Difference | Report |
|---|---|
| Table in `SCHEMA_STATEMENTS`, absent from the DB | re-run init — this class is absorbed automatically |
| Column in the statement, absent from the table | ALTER needed — names the table and the column |
| Column type, nullability, or PRIMARY KEY differs | REBUILD needed — names the table and the mismatch |
| Table in the DB, absent from `SCHEMA_STATEMENTS` | reported, not actioned — may be a retired table or a stale transcription |

Exit status is non-zero when any difference is found, so the check is usable
from a shell without parsing its output.

---

## Applying a schema change

The three classes above are the three ways `db_schema.md` can move ahead of a
live database, and they need different responses:

1. **New table.** Re-transcribe `SCHEMA_STATEMENTS`, re-run init. The new
   table is created empty; existing tables are untouched. Nothing else to do.
2. **New column on an existing table.** Re-transcribe, then apply an
   `ALTER TABLE ... ADD COLUMN` by hand. Init alone will not do it.
3. **Structural change — type, nullability, or primary key.** Not reachable
   by `ALTER`. Requires an explicit rebuild: create the new-shaped table
   under a temporary name, copy the data across, swap. By hand, deliberately,
   with the data volume in mind.

`--verify` is what tells you which class you are in. Run it after
re-transcribing and before assuming the change has landed.

---

## Output / Logging

Plain stdout; no log file and no `batch_runs` row (this tool is outside the
batch-marker scheme entirely — see the concurrency assumption above).

```
init:   created 31 tables, 0 already present    # fresh database
init:   created 1 table, 30 already present     # after a new table was added
verify: 31 tables checked, 0 differences
verify: 31 tables checked, 1 difference
        live_session_state: column 'session_diagnostics' missing — ALTER needed
```

---

## Constraints

- Must be runnable independently from the pipeline (no `src/` imports required)
- Issues the ONLY DDL in the system — no other component may create, alter,
  or drop a table
- `SCHEMA_STATEMENTS` is transcribed verbatim from `db_schema.md`'s canonical
  region; it is never hand-authored, extended, or edited independently of
  that document
- All statements execute in one transaction — a partial schema is never left
  behind
- Never alters or drops an existing table, under any flag
- `--verify` is strictly read-only
- No `batch_runs` marker, no scheduled invocation, never run concurrently
  with a live session or a batch job
