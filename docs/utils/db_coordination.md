# Module: DB Coordination

**File:** `src/utils/db_coordination.py`

---

## Role

DB access coordination, through which every process reaches
`market.duckdb`:
- roles;
- the coordination store;
- lock files;
- admission and units;
- precedence;
- recovery.

Module dependencies:
- this module uses alert_delivery.md alone (Helper deliveries, below);
- guest_supervisor.md uses this module; tool bodies use neither
  guest_supervisor.md nor this module beyond what guest_supervisor.md's
  `run_guest` hands them;
- call order: guest_supervisor.md's run_guest → this module (register, unit)
  → the tool body.

---

## Mechanism

The coordination helper (this module) is the only code that reads or
writes the coordination store and the lock files, and every process that
opens `market.duckdb` opens it through the helper. db_schema.md's DB File
Ownership Windows states the roles and each role's wait spec; this module
owns the mechanism.

ROLES (db_schema.md's DB File Ownership Windows):
- owner — whether launched by the dispatcher or by hand:
  - `collect_daily.py --premarket-open`;
  - LiveModeRunner;
  - the full evening run;
- aux — a process registered with the helper that never opens
  `market.duckdb`:
  - the dispatcher (metadata_crawler.md);
  - the resume sweep `resume_guests` (guest_supervisor.md), registering
    only when it launches;
  - the overnight token refresh;
- guest — every other process that opens `market.duckdb`:
  - every other `collect_daily.py` flag;
  - health_report.md's CLI;
  - run_preprocess;
  - run_train;
  - run_backtest;
  - pipeline_optimizer's coordinator;
  - detection_benchmark;
  - init_db;
  - migration_tool;
  - the DuckDB recovery, `restore_db()` (Recovery, below).

FACT CLASSES — a coordination fact belongs to one of:
- (a) liveness — the OS lock files only;
- (b) current coordination state — the store's `run_state` and `holder`
  tables, authoritative;
- (c) history — the store's `run_journal` table:
  - written in the same transaction as each `run_state` write, state
    fields included;
  - written for each owner's holder transition;
  - a guest's holder transitions are kept as current state only;
- (d) each process's own records:
  - DB rows;
  - `live_session_state.session_diagnostics`;
  - each process's log.
Rules over the classes:
- coordination decisions read (a) and (b) only;
- reports read (c) and (d), and (a) to decide liveness;
- an owner's finding about its own wait carries the (b) holder in its
  detail;
- no fact is recomputed from a source of another class;
- `run_journal` rows carry no work outcome.

COORDINATION STORE — `<coordination.dir>/coordination.sqlite`, SQLite,
opened with:
- `journal_mode=WAL`;
- `synchronous=FULL`;
- `busy_timeout` = `coordination.busy_timeout_ms`.

```sql
CREATE TABLE run_state (
    key              TEXT PRIMARY KEY,  -- an owner's name ('premarket' |
                                        -- 'runner' | 'evening'); a guest's
                                        -- or aux run's run_id
    run_id           TEXT NOT NULL,
    role             TEXT,              -- 'owner' | 'guest' | 'aux'
    tool             TEXT,              -- entry point
    trigger          TEXT,              -- 'scheduled' | 'manual'
    pid              INTEGER,
    date_basis       TEXT,              -- 'YYYYMMDD', ET
    state            TEXT,              -- see States below
    state_at         TEXT,              -- UTC ISO-8601
    registered_at    TEXT,              -- UTC ISO-8601
    session_expected INTEGER NULL,      -- 0/1; premarket and evening rows only
    boundaries       TEXT NULL          -- JSON; premarket and evening rows only
);
-- A row per owner, keyed by owner name; the 'runner' row is the session
-- state. A row per guest or aux run, keyed by run_id.

CREATE TABLE holder (
    id            INTEGER PRIMARY KEY CHECK (id = 1),
    run_id        TEXT NULL,
    unregistered  INTEGER,              -- 0/1
    acquired_at   TEXT
);

CREATE TABLE run_journal (              -- append-only, never purged
    seq      INTEGER PRIMARY KEY AUTOINCREMENT,
    ts       TEXT,                      -- UTC ISO-8601
    run_id   TEXT,
    key      TEXT,
    role     TEXT,
    tool     TEXT,
    trigger  TEXT,
    event    TEXT,                      -- see Journal events below
    state    TEXT NULL,                 -- the state after the event
    detail   TEXT NULL                  -- JSON
);
```

States (`run_state.state`):
- non-terminal:
  - `registered`;
  - `waiting`;
  - `running`;
- terminal:
  - `done`;
  - `failed`;
  - `yielded`;
  - `not_expected`;
  - `crashed`.

Journal events (`run_journal.event`):
- `registered`;
- `state` — a state write; `detail` holds the changed fields, and a
  `waiting` write adds the holder `{run_id, role, tool, acquired_at}`;
- `acquired`, `released` — an owner's holder transition;
- `unregistered_holder` — a DuckDB open failing while `db.lock` is held;
- `crash_found` — `detail` `{crashed_run_id, key}`;
- `duplicate_owner_rejected` — `detail` `{key}`;
- `ended` — `detail` `{end_kind}`.

Writes and reads:
- a state transition is one transaction — `BEGIN IMMEDIATE`, the state
  update, the journal insert, `COMMIT`;
- a write to another run's row is a conditional `UPDATE` on that row's
  expected `run_id` and state;
- `SQLITE_BUSY` is retried;
- read transactions end at once.
A store that cannot be read or written counts as absent state wherever a
role's rule states nothing else; the report functions that raise on it
(Reports, below) are excepted.

LOCK FILES under `coordination.dir`:
- `db.lock` — the access lock, exclusive, held by the process using
  `market.duckdb` while its DuckDB connection is open;
- `maintenance.lock` — held by `rebuild_store()` and by `restore_db()`
  (Recovery, below); while it is held, `register()` refuses and the
  pre-registration read functions (`owner_status()`, `run_info()`) raise as
  on an unreadable store;
- a run lock per process, held from registration to exit:
  - `owner.<name>.lock` for an owner;
  - `runs/<run_id>.lock` for a guest or aux run.
Liveness:
- a run is alive while its run lock is held;
- a run has crashed when its state is `crashed`, or when its state is
  non-terminal and its run lock is free.
Primitive:
- POSIX `fcntl.lockf(fd, LOCK_EX | LOCK_NB)`;
- Windows `msvcrt.locking(fd, LK_NBLCK, 1)` at byte 0;
- never `fcntl.flock` or OFD locks.
Lock handling:
- a lock file is opened only by the helper, one fd per file per process;
- an in-process lock guards re-entry;
- lock files are opened for writing;
- `EACCES` or `EAGAIN` means held;
- holder identity is kept outside the lock file;
- whether a lock is held is decided by a non-blocking attempt, released at
  once when not proceeding;
- a lock the calling process itself holds is answered from the helper's
  in-process record, never by an attempt on its file;
- no child process holds its parent's locks.

RUN REGISTRATION — `register(role, tool, trigger, owner=None)`, a context
manager wrapping each process's entry point:
- `owner` names an owner's `run_state` key (the `run_state` DDL, above);
  `trigger` is the run's `run_state.trigger`;
- the dispatcher enters `register()` only when it launches a run;
- a second `register()` in one process is refused by the re-entry lock;
- a helper call before `register()` raises, except:
  - `owner_status()`;
  - `run_info()`;
  - `rebuild_store()`;
  - `restore_db()`, which enters `register()` itself;
- `register()` issues a `run_id` of the form `{YYYYMMDD}_{HHMMSS}_{pid}` —
  the registration instant in utils.md's `generate_run_id()` form, then the process
  id — and takes the run lock:
  - an owner retries briefly;
  - an owner whose fixed-name run lock is held by a live instance exits
    with an error, recorded in `run_journal` (`duplicate_owner_rejected`);
- an owner then reads its previous row and, finding it in a non-terminal
  state, records that run as crashed;
- the run writes its `run_state` row and a `registered` journal row;
- the run's end kind is recorded once, as an `ended` journal row —
  'normal' | 'error' | 'aborted' | 'hard_cap' | 'timed_out' |
  'self_terminated' | 'yielded' | 'not_expected' | 'superseded' |
  'blocked':
  - `end(end_kind)` records:
    - 'aborted';
    - 'hard_cap';
    - 'timed_out';
    - 'not_expected';
    - 'yielded';
    - 'superseded';
    - 'blocked';
  - 'aborted' is an owner giving up under its own rule;
  - 'superseded' is a run whose guest_supervisor.md work turned superseded;
  - 'blocked' is:
    - a new run refused by guest_supervisor.md's new-run resolution, guest_supervisor.md's
      restore_store() lock included;
    - a dependent ending because its input work is 'halted';
    - a consumer ending because the work it awaits turns:
      - 'halted';
      - superseded;
      - 'done' with an invalid seal;
  - the self-watchdog records 'self_terminated' before `os._exit`;
  - on leaving the `register()` context, an end kind already recorded is
    kept; otherwise a normal exit or a `SystemExit` with code 0 records
    'normal', and an exception or a non-zero `SystemExit` records 'error';
  - a crash is recorded by its finder (below);
- leaving the `register()` context drains pending deliveries before exit
  (Helper deliveries, below);
- on a normal exit a guest or aux run deletes its run lock file and its
  `run_state` row;
- a run found crashed is recorded once:
  - the finder sets its state to `crashed` by conditional `UPDATE` and
    journals it (`crash_found`);
  - a crashed guest's or aux run's lock file and row are then deleted by
    conditional delete;
- `acquire()` takes its role and owner name from the registration;
- owner run lock files are never deleted.

STORE AND LOCK ACCESS:
- only the helper reads or writes the coordination store and the lock
  files; no process:
  - opens `coordination.sqlite` by file I/O;
  - `ATTACH`es it;
  - loads DuckDB's sqlite extension;
- `market.duckdb` and `coordination.dir` sit on a local disk of the host
  every DB user runs on;
- the helper always opens `market.duckdb` read-write, and the connection it
  hands out is wrapped so that `close()` raises, every other attribute and
  method passing through — it is closed only by `Handle.close()` or the end
  of a `guest_unit`;
- a DuckDB open failing while `db.lock` is held:
  - is classified `{class, message}` — 'lock_conflict' when the message
    matches an entry of `coordination.lock_conflict_patterns`, 'unmatched'
    otherwise;
  - returns the run's state to its pre-acquisition state — an owner's
    `waiting`, a guest's `registered`;
  - releases `db.lock`;
  - records the holder as 'unregistered' (`unregistered_holder`, its
    `detail` carrying the classification);
  - retries per the wait spec;
  - for an owner, carries the classification into finding 28's
    `open_error` (health_report.md);
  - for a guest, is delivered as finding 49 (Helper deliveries, below);
- `holder_now()` reports the `holder` row only while `db.lock` is held.

ACQUIRE LOOP — a helper loop shared by every role, polling every
`coordination.poll_seconds`:
- phase A waits on the role's precondition (its wait spec, db_schema.md's
  DB File Ownership Windows), reading the store only;
- phase B takes `db.lock` by non-blocking attempts:
  - an owner writes `waiting` on a failed attempt, with the current holder
    (`run_id`, role, tool, `acquired_at`) in that journal row;
  - on success the `holder` row and state `running` are written in one
    transaction, then DuckDB is opened;
- the role's wait-spec deadlines end the loop with the spec's action;
- `on_event` receives these events, and each owner's spec file maps them
  to findings:
  - `predecessor_crashed`;
  - `waiting_started`;
  - `deadline_reached`;
  - `hard_cap`;
- close order:
  1. the final state;
  2. the DuckDB close;
  3. the `db.lock` release.

PRECEDENCE AND YIELD — runner > premarket > evening > guest:
- the evening run, started in or outside its window, calls `should_yield()`
  at each stage boundary;
- `should_yield()` is true when the premarket or the runner is `waiting`
  with its run lock held;
- a run that yields ends with state and end kind `yielded` and runs no
  further stage (metadata_crawler.md);
- the runner and the premarket never yield.

GUEST ADMISSION AND UNITS — a guest opens `market.duckdb` only inside
`guest_unit(tool, timeout)`, one open per unit; `file_unit(tool, timeout)`
is a guest unit holding `db.lock` with no DuckDB connection open. A unit of
either kind is admitted when:
- (i) `db.lock` is free;
- (ii) no owner is `waiting` with its run lock held;
- (iii) it is not the runner window, the runner window being:
  - (a) today's premarket `done` or `failed` with `session_expected` true
    and today's runner not ended (`done` or `not_expected`); or
  - (b) today's runner row in a non-terminal state, or `crashed` until
    that day's evening run starts;
- (iv) it is not within `coordination.guest_guard_minutes` before the next
  scheduled owner time.
`restore_db()`'s `file_unit` is exempt from (iii) while no owner run lock is
held, judged by non-blocking attempts; (i) and (ii) still apply to it.
With the store unreadable no unit is admitted. A unit not admitted waits up
to `timeout`:
- every `collect_daily.py` guest flag and health_report.md's CLI pass 0 —
  an error exit;
- other guests wait without limit.
Unit scope:
- a `collect_daily.py` guest flag's unit is one stage; not admitted at a
  stage boundary, it ends, the stages not run writing no row;
- a per-run guest holds one unit for its whole run, the connection that
  unit yields passed wherever the tool takes `db_conn`; while an owner waits
  past `coordination.guest_max_hold_seconds`, such a run ends by
  self-termination (below) and is re-run from scratch;
- a step guest holds one unit per step and, ended by self-termination,
  resumes through guest_supervisor.md;
- which tool is a per-run guest and which a step guest is
  guest_supervisor.md's tool registry;
- `restore_db()` holds one `file_unit` for the file replacement only
  (Recovery, below).
No guest is exempt beyond `restore_db()`'s exception above. A per-run guest
ended unadmitted is re-run from scratch; a step guest resumes through
guest_supervisor.md.
`in_runner_window()` answers (iii)'s runner window.

GUEST SELF-WATCHDOG — the helper runs a watchdog thread in every guest.
While an owner is `waiting` with its run lock held and the guest's current
unit has held `db.lock` longer than `coordination.guest_max_hold_seconds`,
the guest records end kind `self_terminated` and ends its process with
`os._exit`.

SESSION EXPECTATION AND BOUNDARIES in owner state:
- the premarket run records in its state today's `session_expected`
  (`trading_calendar.is_trading_day(today)`) and boundaries
  `{date, is_trading_day, session_close, after_hours_end, written_at,
  calendar_read_at}`;
- the evening run records, after its calendar update, the next weekday's
  `session_expected` and boundaries in the same form;
- a reader takes only an entry whose `date` equals its target date, the
  premarket's before the evening's;
- with no such entry, `session_expected` is true and `after_hours_end`
  '200000'.

DISPATCHER — `collect_daily.py --dispatch` (metadata_crawler.md) is an aux
run:
- it reads config only — no store state, and no lock beyond its run lock;
- it registers only when it launches a run.

CONFIG — the `coordination:` block in pipeline_config.yaml:

```yaml
coordination:
  dir: "data/coordination/"
  poll_seconds: 5
  busy_timeout_ms: 5000
  guest_guard_minutes: 10
  guest_max_hold_seconds: 300
  lock_conflict_patterns:      # a DuckDB open error matching one is a lock
    - "Conflicting lock is held"   # conflict, any other 'unmatched'
  db_damage_patterns:          # an owner's 'error' text matching one names
    - "Corrupt database file"      # a possibly damaged file
    - "Corrupt WAL file"
    - "Failed to load metadata pointer"
  schedule:                    # ET wall clock
    premarket:     {time: "04:00", days: weekdays, deadline: "05:00"}
    runner:        {time: "04:00", days: weekdays}
    evening:       {time: "21:00", days: weekdays, deadline: "23:30"}
    token_refresh: {time: "03:00", days: weekdays}
# Former top-level keys, moved here:
# - premarket_wait_deadline → schedule.premarket.deadline
# - evening_wait_hard_deadline → schedule.evening.deadline
# - token_refresh_time → schedule.token_refresh.time
```

Config check at load: `guest_max_hold_seconds + poll_seconds` <
`schedule.premarket.deadline − schedule.premarket.time`; on a violation the
helper refuses to start.

PROCESS LOG — `UNIT` lines. The helper writes each guest's `UNIT` lines to
the shared `logs/guest_units_{date}.log`, in the crawler log's line format
(timestamp, line type, tab-separated `key=value` — metadata_crawler.md):
- fields:
  - `event` — 'start' | 'end' | 'yield' | 'self_terminate';
  - `run_id`;
  - `tool`;
  - `unit`;
  - `hold_seconds`, on 'end' and 'self_terminate';
- written only while holding `db.lock`;
- written for every guest with no structured process log:
  - run_backtest;
  - run_train;
  - run_preprocess;
  - pipeline_optimizer;
  - detection_benchmark;
  - migration_tool;
  - init_db;
  - health_report.md's CLI;
  - duckdb_recovery;
- `collect_daily.py` guests write no `UNIT` lines.

HELPER DELIVERIES — the helper delivers DB-less through alert_delivery.md's
`deliver()`, building each finding entry itself (`gather_findings()` does
not collect these deliveries):
- an owner run recorded 'error' on leaving `register()` → finding 11 for that
  run alone, `alert_key` `owner_error_end` (health_report.md):
  - it carries the exception text or the exit code;
  - when that text matches an entry of `coordination.db_damage_patterns`,
    the alert also names a possibly damaged file and the DuckDB recovery
    (Recovery, below);
- a guest's DuckDB open failure while holding `db.lock` → finding 49,
  `guest_db_open_failed` (health_report.md):
  - detail `{run_id, tool, unit, open_error: {class, message}}`;
  - delivered at the run's first failure and at each change of class;
  - class 'unmatched' names a possibly damaged file and the DuckDB recovery;
- leaving the `register()` context drains pending deliveries
  (alert_delivery.md's `drain()`) before exit, up to the drain timeout
  health_report.md assigns to each path.

RECOVERY — the operator procedure and its known results are
shadow_retraining.md's Recovery bullet:
- in this order, after the operator has ended every live owner and every
  live guest:
  1. the coordination store, when it is unreadable — `rebuild_store(backup_path)`:
     - takes `maintenance.lock` first and releases it at its end;
     - refuses while any of these is held, judged by non-blocking attempts:
       - an owner run lock;
       - a `runs/*.lock`;
       - `db.lock`;
     - moves the unreadable `coordination.sqlite` with its `-wal` and `-shm`
       files to `coordination.sqlite.damaged_{UTC timestamp}`;
     - creates a fresh store and copies the backup's `run_journal` rows with
       their `seq` and `ts`;
     - `run_state` and `holder` start empty;
  2. the DuckDB file — `restore_db(backup_path)`:
     1. enters `register('guest', 'duckdb_recovery', 'manual')`;
     2. takes `maintenance.lock`, then refuses while any other guest run
        lock is held, judged by non-blocking attempts;
     3. copies the backup to `market.duckdb.restore` in `market.duckdb`'s
        directory;
     4. inside `file_unit('duckdb_recovery', None)`, removes
        `market.duckdb.wal`;
     5. then runs `os.replace(market.duckdb.restore, market.duckdb)`;
     a re-run overwrites a leftover `market.duckdb.restore`.

```python
def owner_status(name: str) -> dict | None:
    """
    The owner's current state: {run_id, state, state_at, date_basis,
    session_expected, boundaries, alive} — its run_state row, and `alive`
    true while that owner's run lock is held. None with no row.
    Callable before registration.
    """
    ...


def backup_store(dest_path: str) -> None:
    """
    Copy the coordination store to dest_path through the SQLite backup API.
    Called by metadata_crawler.md's evening_backup stage, which writes the
    resume store's copy (guest_supervisor.md's backup function) beside this
    one.
    """
    ...


def register(role: str, tool: str, trigger: str, owner: str | None = None):
    """
    Context manager wrapping the process's entry point — Run registration
    (above). Leaving it records the end kind unless end() already did, then
    drains pending deliveries (Helper deliveries, above).
    """
    ...


def end(end_kind: str) -> None:
    """
    Record the run's end kind: 'aborted' | 'hard_cap' | 'timed_out' |
    'not_expected' | 'yielded' | 'superseded' | 'blocked'.
    """
    ...


def acquire(spec, on_event: Callable[[str, dict], None]) -> "Handle | str":
    """
    Run the acquire loop (above) for the registered run's role and owner
    name under its wait spec `spec`. Returns a Handle, or an outcome:
    'aborted' | 'timed_out' | 'not_expected' | 'hard_cap'.
    """
    ...


class Handle:
    conn: "duckdb.DuckDBPyConnection"
    """The open connection to market.duckdb, read-write, wrapped so that
    close() raises; every other attribute and method passes through."""

    def should_yield(self) -> bool:
        """True when the premarket or the runner is `waiting` with its run
        lock held (Precedence and yield, above)."""
        ...

    def close(self, final_state: str) -> None:
        """Write final_state, close DuckDB, release db.lock — in that order.
        The only way an owner's connection is closed."""
        ...


def guest_unit(tool: str, timeout: float | None):
    """
    Context manager for one guest unit: entering waits for admission (Guest
    admission and units, above) up to `timeout` (None waits without limit)
    and opens market.duckdb read-write, yielding the connection wrapped as
    Handle.conn is; leaving closes it and releases db.lock — the only way a
    guest's connection is closed.
    """
    ...


def file_unit(tool: str, timeout: float | None):
    """
    Context manager for one guest unit holding db.lock with no DuckDB
    connection open; yields nothing. Admission, `timeout`, UNIT lines and
    the self-watchdog are those of guest_unit().
    """
    ...


def rebuild_store(backup_path: str) -> None:
    """Rebuild the coordination store (Recovery, above), holding
    maintenance.lock from its start to its end. Callable before
    registration."""
    ...


def restore_db(backup_path: str) -> None:
    """Restore market.duckdb from backup_path (Recovery, above). Enters
    register() itself, then takes maintenance.lock and refuses while any
    other guest run lock is held, judged by non-blocking attempts."""
    ...


def current_run() -> dict:
    """{run_id, role, tool, trigger, date_basis} of the registered run —
    alert_delivery.md's log file date reads its date_basis."""
    ...


def in_runner_window() -> bool:
    """Whether it is now the runner window of the guest admission test."""
    ...


# Reports — readers of run history. The store has no reader outside this
# module's functions.
# abnormal_runs(), run_info() and missed_scheduled_run() raise on a store
# that cannot be read; the absent-state rule does not apply to them
# (health_report.md's findings 1 and 11).

def abnormal_runs(since: str, until: str) -> list[dict]:
    """[{run_id, role, tool, end_kind, registered_at, ended_at}]"""
    ...


def run_info(run_id: str) -> dict | None:
    """
    {run_id, key, role, tool, trigger, pid, date_basis, registered_at,
    ended_at, end_kind, alive, events} — `events` being that run's
    run_journal rows. Callable before registration, as owner_status() is.
    `alive`: the run lock judged by a non-blocking attempt that opens the lock
    file without creating it, a missing file reading not alive. A run in a
    non-terminal state with its lock free returns end_kind 'crashed'. A
    run_id the store does not know returns None.
    """
    ...


def owner_runs(name: str, date: str) -> list[dict]:
    """[{run_id, trigger, registered_at, ended_at, end_kind, last_state}]"""
    ...


def missed_scheduled_run(name: str, date: str) -> bool:
    """
    True when `date` is one of that name's coordination.schedule `days` and
    no scheduled run of that name is registered for it.
    """
    ...


def holder_now() -> dict | None:
    """
    {run_id, unregistered, role, tool, acquired_at}, or None while db.lock
    is free.
    """
    ...


def journal_view(since: str, until: str, role: str | None) -> list[dict]:
    """run_journal rows — health_report.md's CLI `--runs` only."""
    ...
```


---

## Constraints

- The helper's in-process state is only:
  - its run record;
  - its lock record;
  - its re-entry lock
- The DB Access Coordination helper is the only reader and writer of
  `coordination.sqlite` and the lock files, and `market.duckdb` is opened
  only through `acquire()` or `guest_unit()`
- Whether a lock is held is decided by a non-blocking `fcntl.lockf` /
  `msvcrt.locking` attempt, never by `fcntl.flock` or OFD locks
- A `run_state` write and its `run_journal` row are one transaction
- The connection the helper hands out is closed only through
  `Handle.close()` or the end of a `guest_unit`
