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
  - alert_delivery.md (the `scheduled_work_deferred` delivery);
  - utils.md (Config Hashing for a work's config_hash; `generate_run_id()`
    for its work_id);
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
- unit kind — per run | per step (db_coordination.md's Unit scope);
- body module path, resolved at run time;
- whether the tool produces a sealed result (Sealed Results, below);
- per variant, the config keys naming input files.

Registered:
- `run_preprocess` — per step; sealed result;
- `pipeline_optimizer` — per step; input files:
  - variants `'full'` and `'exploitation'`: `optimizer.selected_features_path`;
  - variant `'selection'`: none;
- `run_train` — per run;
- `run_backtest` — per run;
- `detection_benchmark` — per run;
- `init_db` — per run;
- `migration_tool` — per run.

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

A `--resume-work` run:
- takes the trigger from the work;
- opens that work directly and hands it to the body;
- ends `'normal'` without opening when:
  - the work is not `'active'`;
  - the work is held by a live run;
  - `resume.lock` is held;
- with `--trigger` or `--supersede` is an error at start.

What the body receives:
- a step tool — a work opener and a unit opener, the body opening its work
  before its first step; the work opener:
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
  - answers the body's reads of the tool's work records — its unfinished
    work's config_hash, and the `updated_at` of its latest `'done'` work
    per variant (pipeline_optimizer.md's refresh run);
- a per-run tool — the connection of one `guest_unit(tool, None)` opened
  around the body.

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
                                        -- variant's input-file contents
    pin               TEXT NOT NULL,    -- JSON, tool-defined
    tool_snapshots    TEXT,             -- JSON, tool-defined (rate_snapshot)
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
                                        -- | 'opened'
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
  uses and the content hash of every input file of the work's variant:
  - run_preprocess — the keys its Config Keys Used lists inside
    config_hash;
  - pipeline_optimizer — the full config with the dispatched phase, not
    `'refresh'`;
- the fixed inputs, set when the work is created:
  - `config_snapshot`;
  - `pin`;
  - `tool_snapshots`;
  - `input_works`;
- every execution of a work reads the work's fixed inputs, never the
  current config or input files.

---

## New-Run Resolution

Checked and created in one `resume.sqlite` transaction.

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
- when scheduled, is delivered as `scheduled_work_deferred` through
  alert_delivery.md's `deliver()` (health_report.md), on every refusal.

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
- `work_progress()` — sets `progress_at`, at each step commit;
- `work_close(status)`.

A live run whose work turns superseded ends `'superseded'`, checking at each
step commit and in its wait loop.

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
  config or input files
- The sweep is the sole launcher of works
- Tool bodies use neither this module nor db_coordination.md beyond what
  `run_guest` hands them
