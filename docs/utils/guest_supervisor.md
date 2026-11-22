# Module: Guest Supervisor

**File:** `src/utils/guest_supervisor.py`

---

## Role

The common guest entry point and its tool registry, and resumable guest
works:
- `run_guest` — the one entry point of every registered tool;
- the tool registry — which tools run through `run_guest`, and how;
- works — the record of a step tool's computation, resumed across runs;
- new-run resolution — whether a run:
  - resumes a work;
  - creates one;
  - reuses a result;
  - is refused;
- dependency and sealed results — one work consuming another's result;
- the relaunch sweep `resume_guests` — the sole launcher of works.

Module dependencies:
- this module uses:
  - db_coordination.md:
    - `register()`;
    - units;
    - `run_info()`;
  - alert_delivery.md — `deliver()` for a scheduled refusal's first
    delivery, `write_to_log()` for the refusals recorded without dispatch
    and for finding 56's `code_version_unattested` records;
  - utils.md (Config Hashing for a work's config_hash; `generate_run_id()`
    for its work_id; Code Versions for the code checks);
- this module runs tool bodies by their registry module paths;
- a tool script only hands over to `run_guest`; tool bodies use neither
  this module nor db_coordination.md beyond what `run_guest` hands them;
- outside `run_guest`'s path and the sweep, this module is used only
  through:
  - its report functions (Reports, below) — by health_report.md, and by
    live_mode_runner.md through its session-start report;
  - `backup_resume_store()` — by metadata_crawler.md's `evening_backup`;
  - `restore_store()` — by the operator's recovery (shadow_retraining.md's
    Recovery);
- call order: `run_guest` → db_coordination.md (`register()`, unit) → the
  tool body.

---

## Tool Registry

The single list of the tools run through `run_guest`. Each entry:
- name;
- unit kind — per run | per chunk | per step (db_coordination.md's Unit
  scope);
- body module path, resolved at run time;
- whether the tool produces a sealed result (Sealed Results, below);
- per variant, the variant's inputs;
- the code areas it computes from (utils.md's Code Versions).

A per-step tool opens a work; a per-chunk or per-run tool opens none.

Registered:
- `run_preprocess` — per step; sealed result; areas: data;
- `pipeline_optimizer` — per step; areas: data and training; the variant's
  inputs:
  - `'exploitation'`: the content of `optimizer.selected_features_path`;
  - `'refit'`: the retained configuration (pipeline_optimizer.md);
  - `'full'`: none;
  - `'selection'`: none;
- `run_train` — per chunk; areas: data and training;
- `run_backtest` — per chunk; areas: data and training;
- `detection_benchmark` — per chunk; areas: data;
- `migration_tool` — per chunk; areas: data;
- `model_deploy` — per chunk; no areas;
- `init_db` — per run; no areas.

A file the registry names as an input of some variant is written by its
producer through temp + rename (pipeline_optimizer.md's `'selection'`
writing `selected_features.json`).

---

## Entry Point — `run_guest`

Forms:
- `run_guest <tool> [--trigger scheduled] [--supersede] [tool arguments]` —
  a new run, resolved by New-Run Resolution (below);
- `run_guest <tool> --resume-work <work_id>` — the sweep's command,
  composed from the work row.

Each form enters `register('guest', tool, trigger)` (db_coordination.md).

A new run:
- without `--trigger` is `'manual'`;
- `--supersede` is accepted only for a manual run; with
  `--trigger scheduled` it is an error at start.

The code check: for a tool with areas, `run_guest` checks them before
New-Run Resolution, `--resume-work` runs included (utils.md's Code
Versions); an unattested area ends the run `'blocked'` with cause
`code_unattested`, a finding 56 entry recorded through alert_delivery.md's
`write_to_log()` alone, not dispatched, under alert_key
`'code_version_unattested'`.

A tool that opens no work (a per-chunk or per-run tool):
- `run_guest` skips New-Run Resolution and work opening;
- `--trigger scheduled`, `--supersede` and `--resume-work` are errors at
  start;
- a self-terminated run ends without a relaunch.

A `--resume-work` run:
- takes the trigger from the work;
- opens that work directly and hands it to the body;
- ends `'normal'` without opening when, in this order:
  - `resume.lock` is held;
  - the work's code_versions differ from the lock's — the work turns
    superseded first (Code Versions of a Work, below);
  - the work is not `'active'`;
  - the work is held by a live run;
- with `--trigger` or `--supersede` is an error at start.

What the body receives:
- a step tool — a work opener, a unit opener and the step helpers (Steps,
  below), the body opening its work before its first step; the work opener:
  - opens the body's work by New-Run Resolution (below), run on the
    connection of the unit in which the body calls it, given:
    - the run's pin;
    - its tool snapshots;
    - a sealed-result tool's stale-batch test;
  - for a dependent, resolves its input work in the same resolution, given
    the input's:
    - covered config;
    - pin;
    - stale-batch test;
  - answers the body's reads of the tool's work records:
    - its unfinished work's config_hash, and whether that work is held by a
      live run;
    - the `updated_at` of its latest `'done'` work per variant
      (pipeline_optimizer.md's refresh run);
    - the pipeline_optimizer works carrying a given divergence week, with
      each work's status (Reports, below);
  - answering a read of the tool's unfinished work, applies the
    code-version supersede (Code Versions of a Work, below) first, in its own
    `resume.sqlite` transaction, then answers as with no unfinished work,
    naming the work it superseded;
- a per-chunk tool — a unit opener, each call opening one
  `guest_unit(tool, None)`;
- a per-run tool — the connection of one `guest_unit(tool, None)` opened
  around the body.

Units hold DB work only (db_coordination.md's Unit scope):
- a guest holds a unit only for a bounded chunk of DB work; the tool's own
  compute runs outside units;
- a DB function that computes internally runs inside the unit of its call,
  each call bounded — `populate_precomputed_session_stats()`,
  `compute_tick_bar_aggregates()`, the crawlers and
  `compute_vol_regime_holdout()` among them;
- between units a run waits for admission;
- no unit is opened while another unit of the same run is held;
- a run self-terminated while holding a unit (over
  `coordination.guest_max_hold_seconds` while an owner waits): a per-chunk
  tool re-runs from scratch; a step tool resumes through its work.

The change check: for a run whose tool has areas, the unit opener, before
opening each unit, compares the run's areas with their state at the run's
start (utils.md's Code Versions):
- the set of member files;
- each file's mtime and size;
- the listed distributions' versions.
On a difference it recomputes the areas' code_hash; one differing from the
code_hash attested at the start ends the run `'blocked'` with cause
`code_changed`, no unit opened, a finding 56 entry recorded through
`write_to_log()` as for `code_unattested`.

What the body works on, one of:
- the opened work — from the work opener, or handed by `run_guest` to a
  `--resume-work` run;
- a reused result;
- for a consumer, the awaited work.

Of these:
- each exposes:
  - its `work_id`;
  - its fixed inputs;
  - its seal hash;
- the opened work offers the child API (A Run's Work, below);
- a wait — a dependent's for its input works, a consumer's for the awaited
  work — holds no unit: a resolution that waits returns to the body, which
  releases its unit first.

Not through `run_guest`:
- `collect_daily.py` guest flags;
- health_report.md's CLI;
- db_coordination.md's `restore_db()`.

---

## Resume Store — `resume.sqlite`

`<coordination.dir>/resume.sqlite`, SQLite, this module's own store,
separate from db_coordination.md's coordination store and under the same
discipline:
- opened as the coordination store is;
- a state transition is one transaction;
- the journal is append-only.

An absent `resume.sqlite` is an empty store, created on first write;
"unreadable" means present but unreadable. Timestamps are UTC ISO-8601.

```sql
CREATE TABLE works (                    -- rows are never deleted
    work_id           TEXT PRIMARY KEY,
    tool              TEXT NOT NULL,
    resume_key        TEXT NOT NULL,    -- hash of tool and config_hash; not unique
    config_hash       TEXT NOT NULL,
    -- fixed inputs, set at creation:
    config_snapshot   TEXT NOT NULL,    -- JSON: the covered config with the
                                        -- variant's inputs, and code_versions:
                                        -- the lock's versions of the areas
                                        -- its config_hash holds
    pin               TEXT NOT NULL,    -- JSON, tool-defined
    tool_snapshots    TEXT,             -- JSON, tool-defined: rate_snapshot;
                                        -- the champion snapshot; a
                                        -- divergence-dispatched work's
                                        -- divergence week and target run_id
    input_works       TEXT,             -- JSON [{work_id, seal_hash}]; each
                                        -- seal_hash recorded at first
                                        -- consumption, fixed thereafter
    variant           TEXT,             -- tool-defined: pipeline_optimizer's
                                        -- phase; NULL otherwise
    status            TEXT NOT NULL,    -- 'active' | 'halted' | 'done' | 'superseded'
    trigger           TEXT NOT NULL,    -- that of the run that created the work
    run_id            TEXT,             -- the last run that opened it
    launch_pending_at TEXT,
    launch_attempts   INTEGER NOT NULL DEFAULT 0,
    progress_at       TEXT,
    seal_hash         TEXT,             -- sealed-result tools
    created_at        TEXT NOT NULL,
    updated_at        TEXT NOT NULL
);

CREATE TABLE work_journal (             -- append-only, never purged
    seq      INTEGER PRIMARY KEY AUTOINCREMENT,
    ts       TEXT NOT NULL,
    work_id  TEXT NOT NULL,
    event    TEXT NOT NULL,             -- 'created' | the status entered
                                        -- | 'opened' | 'deferral_notified'
    cause    TEXT,
    detail   TEXT                       -- JSON
);
```

- unfinished = `'active'` or `'halted'`; at most one unfinished work per
  tool;
- `work_id` is generated at creation: pipeline_optimizer's is its
  `optimizer_run_id` (`utils.generate_run_id()`, pipeline_optimizer.md);
  another tool's is `<tool>_` followed by `utils.generate_run_id()`, so the
  works one creation writes together never share an id;
- there is no launch command field: the sweep composes the command from
  the row;
- every status transition of a work, its creation included, is a journal
  row written in the transition's transaction with its cause:
  - a creation row records whether the creating run passed `--supersede`;
  - a supersede row names the superseding work;
- these fields of a work change only when a run opens the work:
  - `run_id` — the run records its own;
  - `launch_pending_at` — cleared;
  - `launch_attempts` — cleared;
- a run's opening of a work also writes a journal row with event
  `'opened'` and detail `{run_id}`, in the transaction that records the
  work's `run_id`.

---

## Config Hash and Fixed Inputs

- `config_hash` = utils.md's `config_hash()` over the config keys the tool
  uses, the variant's inputs, and code_versions holding the lock's area
  versions:
  - run_preprocess — the keys its Config Keys Used lists inside
    config_hash; the data version;
  - pipeline_optimizer — the full config with the dispatched phase, not
    `'refresh'`; the data and training versions;
- the fixed inputs, set when the work is created:
  - `config_snapshot`;
  - `pin`;
  - `tool_snapshots`;
  - `input_works`;
- every execution of a work reads the work's fixed inputs, never the
  current config or the variant's current inputs.

---

## Code Versions of a Work

An unfinished work whose code_versions differ from the lock's turns
superseded, in one `resume.sqlite` transaction:
- its journal row: cause `code_version_changed`, its detail naming the
  work's versions and the lock's;
- its unfinished dependents turn superseded with cause propagated
  (Dependency, below).
The run then proceeds as with no unfinished work. This precedes every other
handling of the unfinished work:
- a `--resume-work` run: after its `resume.lock` condition, before its
  not-`'active'` and held-by-a-live-run conditions; the run then ends
  `'normal'`;
- New-Run Resolution, for the run's own work and a dependent's input work;
- the refresh run's first look (pipeline_optimizer.md), before its held-work
  rule and its config_hash comparison.

---

## New-Run Resolution

Checked and created in one `resume.sqlite` transaction, after the
code-version supersede above.

With an unfinished work of the tool:
- a manual run with `--supersede` turns that work and its unfinished
  dependents superseded, then creates a new work;
- same key, `'active'`, not held by a live run → resume;
- same key, `'active'`, held by a live run:
  - a sealed-result tool's run waits as a consumer;
  - another tool's run is refused;
- same key, `'halted'`:
  - a manual run resumes it (`'halted'` → `'active'`);
  - a scheduled run is refused;
- another key → refused.

With no unfinished work:
- a sealed-result tool's run reuses the result of a `'done'` work with:
  - the same key;
  - the same pin;
  - a valid seal;
  - no stale batch, the staleness checked in the unit computing the run's
    pin;
- otherwise a new work is created with the run's pin.

A dependent (Dependency, below) resolves its own work, then its input work,
each by this resolution with the dependent's trigger:
- `--supersede` applies to its own work and its input work;
- the rows to create are created together, and any refusal creates none;
- a resumed dependent keeps its fixed inputs.

A refused run:
- ends `'blocked'` (db_coordination.md's end kinds);
- when scheduled, is a `scheduled_work_deferred` refusal (health_report.md's
  finding 54), its finding entry carrying `dispatched`: true | false:
  - the first scheduled refusal blocked by a given work: in the refusal's
    `resume.sqlite` transaction the blocking work's journal gains event
    `'deferral_notified'`; after that transaction commits, the refusal is
    delivered through alert_delivery.md's `deliver()`;
  - a later scheduled refusal blocked by a work carrying
    `'deferral_notified'` is recorded through `write_to_log()` alone and not
    dispatched;
  - a scheduled refusal blocked by `restore_store` is delivered on every
    refusal, nothing written to `resume.sqlite`.

While `resume.lock` is held the resolution refuses, the run ending
`'blocked'`. While `resume.sqlite` is unreadable the resolution raises, the
run ending `'error'`.

A standalone run that only reads a sealed result is a consumer holding no
work.

---

## Dependency

- a dependent waits until its input work is `'done'` with a valid seal;
- an input work turning `'halted'` ends the dependent's run `'blocked'`;
- an input work turning superseded turns its unfinished dependents
  superseded (cause: propagated);
- an input work with an invalid seal, or with a seal hash differing from
  the one recorded, turns the dependent superseded (cause: input invalid),
  the next new run creating a new work;
- a consumer ends `'blocked'` when its awaited work turns:
  - `'halted'`;
  - superseded;
  - `'done'` with an invalid seal.

---

## Sealed Results

- a sealed-result work is sealed in its own unit once every step's result
  passes the tool's done test, a step failing it being processed again
  first;
- it closes `'done'` only after `data/processed/resume/{work_id}/_done.json`,
  written through temp + rename, lists its files with row counts and key
  hashes; that seal's hash is recorded on the work;
- a seal is valid when:
  - `_done.json` is readable;
  - its hash equals the work's `seal_hash`;
  - every file it lists passes the result file check with its listed row
    count and key hash;
- the result file check — one check for every result file:
  - readable;
  - its metadata naming its work and slot;
  - its row count matching;
  - its schema hash equal to the current one.

---

## Work Directories

- every work's directory `data/processed/resume/{work_id}/` — a
  sealed-result work's result directory and pipeline_optimizer.md's
  reduction-report directory alike — is kept while referenced:
  - its own work unfinished;
  - an unfinished dependent recording it;
  - the latest `'done'` work of its key;
- unreferenced directories under `data/processed/resume/`, those with no
  work row included, are deleted by this module at each work close.

---

## A Run's Work

Child API, called by the body on its opened work:
- `work_progress()` — sets `progress_at`, immediately before each step
  commit;
- `work_close(status)`.

A live run whose work turns superseded ends `'superseded'`, checking at
`s.write()`, at `db_step()`'s commit and in its wait loop.

### Steps

A step of a step tool holds a read unit and a write unit, its compute
running between them outside any unit:
- the read unit — the step's done check, releasing the unit when done; the
  step's DB reads; its mark, where the step takes one;
- the write unit — the step's DB writes; `work_progress()`; the commit.

The step helpers a step tool receives with its unit opener:
- `with step(done, read=None, results=None) as s:`:
  - entering opens the read unit and runs done(conn):
    - true → `s.skipped` is True, and `s.results` is results(conn) when
      results is given;
    - false → `s.skipped` is False, and `s.inputs` is read(conn) when read
      is given;
  - the read unit commits the rows that DB functions called in it wrote,
    then is released;
  - the body computes under `if not s.skipped:` holding no unit; a component
    reading the DB during compute takes the unit opener as its connection
    provider;
  - `s.write(fn)`, in this order:
    - opens the write unit;
    - a work turned superseded → the run ends `'superseded'`, nothing
      written;
    - renames the step's result files from the temp files written during
      compute;
    - fn(conn);
    - `work_progress()`;
    - the commit;
  - a step not skipped ends with `s.write()`; with nothing to write, fn
    writes no row;
  - an exception in the body or in fn → no commit, the write unit rolled
    back;
- `with db_step(done) as d:`, for a step whose work is all DB work:
  - one unit; done(d.conn) true → `d.skipped` is True and the body does
    nothing;
  - otherwise the body works on d.conn; on leaving without an exception, in
    this order: the superseded check; `work_progress()`; the commit.
- A step's body opens no step.
- The helpers serve step tools only: a per-chunk tool opens its units
  through the unit opener; a per-run tool works on the connection of the one
  unit `run_guest` opens around its body.

The I/O rule of a guest's steps and chunks:
- a component neither writes to the DB nor commits; it returns the rows it
  produces — Trainer, Backtester, BacktestEngine and the Regime Holdout
  Report computation among them;
- the owner writes those rows: a step tool's coordinator or body, in the
  step's write unit, `work_progress()` immediately before the commit; a
  per-chunk tool's body, in its write chunk;
- a step's result files are written through temp + rename before the write
  unit's commit; model artifacts are saved by Trainer during compute;
- DB reads happen in the step's read unit, and during compute in units a
  connection provider opens, each one bounded read;
- rows a step produces are used from memory, never read back from the DB;
- the code after a step takes the step's results: from a computed step, its
  in-memory output; from a skipped step, rows read in its read unit when the
  done check holds;
- exceptions:
  - a DB function that computes internally writes inside the unit of its
    call;
  - a step whose work is all DB work runs in one unit, pipeline_optimizer.md's
    regime holdout step included;
  - a write conditioned on current rows, within the write unit's
    transaction.

---

## Relaunch Sweep — `resume_guests`

Run by its own crontab entry every minute, beside the dispatcher's
(metadata_crawler.md). The sole launcher of works.

- skips its minute while any of these holds:
  - the coordination store is unreadable;
  - `resume.sqlite` is unreadable;
  - `maintenance.lock` is held;
  - `resume.lock` is held;
- registers as an aux run only when it launches;
- per work, one `resume.sqlite` transaction holds:
  - its target check;
  - its attempt count;
  - its `launch_pending_at` write.

Target: an `'active'` work with no live run and no unexpired
`launch_pending_at`. With R = the run its `run_id` names (db_coordination.md's
`run_info()`):
- R NULL or unknown → launch;
- R ended `'self_terminated'` → launch;
- R crashed or ended `'blocked'`:
  - progress since R's registration (the work's own `progress_at` or an
    input work's) → launch;
  - else, any input work unfinished → wait;
  - else → `'halted'`;
- R ended `'normal'` or `'error'` with the work still `'active'` →
  `'halted'`.

Launch:
- runs `run_guest <tool> --resume-work <work_id>` and sets
  `launch_pending_at`;
- a pending unclaimed after `guest_supervisor.launch_pending_timeout_seconds`
  is a failed attempt;
- `guest_supervisor.launch_max_attempts` consecutive failed attempts →
  `'halted'`;
- attempts count only on an `'active'` work; a pending left on a work no
  longer `'active'` is cleared without counting.

---

## Backup and Restore

- `backup_resume_store(dest_path)` copies `resume.sqlite` through the
  SQLite backup API; metadata_crawler.md's `evening_backup` calls it for
  `backup_dir/resume_YYYYMMDD.sqlite`, beside the coordination copy;
- `restore_store(backup_path)` runs under this module's own lock
  `<coordination.dir>/resume.lock`, owned and opened by this module and
  outside db_coordination.md's lock files:
  - entering it is refused while any work's run is alive
    (db_coordination.md's `run_info()` alive);
  - it moves the unreadable store aside and copies the backup in;
  - it is listed in shadow_retraining.md's Recovery;
- while `resume.lock` is held:
  - New-Run Resolution refuses, the run ending `'blocked'`;
  - a `--resume-work` run ends `'normal'` without opening;
  - the sweep skips its minute.

---

## Reports

- `work_journal_view(since, until)` — the `work_journal` rows in the
  span (health_report.md's work-transition finding);
- `work_opened_by(run_id)` — the `work_id` of the work that run opened,
  from the journal's `'opened'` rows, or None (health_report.md's
  abnormal-end finding);
- `works_for_divergence_week(week_start)` — the pipeline_optimizer works
  carrying that divergence week in `tool_snapshots`, each with its status
  (shadow_retraining.md's divergence trigger, health_report.md's finding 6);
- outside this module's own paths they are read by health_report.md, in
  its own process or in live_mode_runner.md's session-start report;
- each raises on a `resume.sqlite` that is present but unreadable; the
  reading finding then reports the store unreadable (health_report.md).

---

## Config

```yaml
guest_supervisor:
  launch_pending_timeout_seconds: 300
  launch_max_attempts: 2
```

---

## Constraints

- Only this module reads or writes `resume.sqlite` and `resume.lock`
- A status transition and its `work_journal` row are one transaction
- A run's opening of a work and its `'opened'` row are one transaction
- Work rows are never deleted
- At most one unfinished work per tool
- Every execution of a work reads its fixed inputs, never the current
  config or the variant's current inputs
- The sweep is the sole launcher of works
- Tool bodies use neither this module nor db_coordination.md beyond what
  `run_guest` hands them
- A component neither writes to the DB nor commits; the owner writes its
  rows in the write unit (Steps)
- No unit is opened while another unit of the same run is held
