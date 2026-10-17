# Ops Doc: Shadow Mode, Staged Rollout, and Retraining Cadence

**No corresponding source file — operational procedure, not a code module.**

---

## Role

Defines the path from a freshly-trained model to full live capital
deployment (shadow → pilot → scale), and the ongoing retraining cadence
once deployed. Supersedes the informal "no defined stage before real
orders" gap in the original design review.

---

## Stage 1: Shadow Mode

Full live pipeline runs end-to-end (detection, inference, sizing,
execution_common.md's shared decision logic) with `live_mode.stage: shadow`
— see live_mode_runner.md's Watchdog Polling Loop / Position Manager Loop
`stage == "shadow"` branches. That key REPLACES the former `shadow_mode`
boolean and also names Stages 2 and 3 (`pilot`, `scale`), which previously
had no record anywhere: they were distinguished only by config values and
were indistinguishable in the data. It is recorded to
`live_session_state.stage` at session start. Orders are never sent to the broker; hypothetical
fills are estimated via `execution_common.simulate_entry_fill()` /
`simulate_exit_fill()` (the same functions BacktestEngine uses) and logged
to `trade_log` with `is_shadow=TRUE`.

**Shadow parity.** The shadow takes only its fill model from backtest;
backtest simplifications beyond the fill model are not adopted. Everything
else follows the real path:
- detection timing;
- the reference price, and its refresh as (simulated) fills accrue;
- tp/sl evaluation starting at the first fill;
- an exit logical order's opening cancelling the entry remainder;
- freeze gating.

Shadow fills and pricing (live_mode_runner.md):
- fills are step-form simulator calls over the REST portion of the
  held-ticker tape:
  - each print with its own volume;
  - computed inline after each poll's append;
  - each step with a fill writing one `live_fills` row
    (`source='shadow'`);
- a shadow limit exit is priced and amended as the real path does, each
  ladder query writing a `bid_ask_snapshots` row, source
  `'exit_ladder_rest'` (db_schema.md):
  - level-1 from the REST orderbook query;
  - the real k ladder;
  - escalation.

**Duration:** `live_mode.shadow_duration_weeks` (config, default `0` = sync
to `class_balancer.outer_fold.test_weeks`, currently 6 weeks). An explicit
positive value overrides the sync. Rationale for defaulting to the outer
fold's test window: outer-fold test data is this project's existing
standard for "genuinely unbiased, untouched-by-tuning" evaluation — shadow
data is the live-deployment analog of that same standard, so reusing the
value keeps both meanings of "unbiased holdout" consistent rather than
picking an unrelated number.

**Comparison procedure**, run once at shadow period end:
1. Query `trade_log WHERE is_shadow = TRUE` for the shadow period's date
   range → shadow's realized winning_rate / avg_pnl_pct.
2. Run BacktestEngine standalone over the *same date range*, using the
   *same model* → backtest's predicted winning_rate / avg_pnl_pct for that
   period.
3. Compare. A material gap indicates a live-operational reality (feed
   timing, health-gate degraded-mode paths actually firing, halt/gap
   handling, etc.) that backtest's idealized replay does not capture —
   this is shadow's whole purpose, not a bug in either number.

## Stage 2: Pilot

Real orders, deliberately small size. No new sizing mechanism — set
`execution.exposure_cap_pct` and/or `execution.per_ticker_share_cap_pct`
(see execution_common.md's `compute_position_size()`, default `0` =
disabled) to a small nonzero fraction for the pilot period, then relax back
toward `0` (or a larger deliberate ceiling) at Stage 3. `stage: pilot`
from this stage onward, `stage: scale` at Stage 3.
Entering either while the circuit-breaker thresholds are still at their `0`
default is REFUSED AT STARTUP (live_mode_runner.md). R-4 already required
them to be set on entering this stage and had no mechanism to enforce it;
one declarative key gives it one.

**Circuit-breaker thresholds must be set on entering this stage (R-4).**
`execution.intraday_loss_limit_pct`, `consecutive_loss_limit` and
`entries_per_hour_limit` all default to `0` = no limit, which leaves the entire
trip path — the `'breaker_trip'` freeze reason, `gate_result='breaker'`, the
immediate alert — dormant. Entering Pilot on those defaults means real capital
runs with stop-losses and cash exhaustion as the only brakes, over code paths
that have never once executed. Calibrate from two sources: backtest's
entries-per-hour distribution (available now that BacktestEngine simulates
chronologically per date and counts gate rejections — 09_backtest_engine.md),
and health_report.md's three breaker metrics, which are computed every session
regardless of whether the thresholds are armed. Same class of stage
precondition as Stage 3's "at least one successful fit_execution_params()".

**Execution-parameter calibration** — each per order type; see
execution_common.md's `execution:` config and Fill-Rate Selection:
- `buy_rate`;
- `sell_rate_tp`;
- `sell_rate_sl`;
- `sell_rate_neutral`.

`cancel_after_seconds` is a policy value, owned by config and
pipeline_optimizer's execution_eval, and is not calibrated here.
Real fills only exist from this stage onward, which
is what makes this calibration possible at all — Shadow (Stage 1) has no
real fill to compare a prediction against, so this is Pilot-only, not
something Shadow's comparison already covers (Shadow's comparison is
aggregate winning_rate/avg_pnl_pct only — a different, independent check;
see the divergence-check note under Retraining Cadence below for how the
two relate).

*Data collection*: at each pilot-stage trade's close transition
(live_mode_runner.md's close pending — once the ticker's REST tape covers
the trade's last second), the records below are written on the same
`trade_log` row as the real fill columns, which is what lets that row be written
COMPLETE in a single INSERT rather than inserted and updated later:
- the counterfactual `predicted_*` columns, computed once by
  `simulate_entry_fill()` / `simulate_exit_fill()`'s batch form over the
  REST tape, in spans split at the real request path's changes
  (execution_common.md's Fill Simulation Model) — no incremental steps and
  no Warm Restart replay:
  - the columns:
    - `predicted_fill_price`;
    - `predicted_weighted_avg_exit_price`;
    - `predicted_partial_fills_count`;
    - `predicted_entry_filled_qty`;
    - `predicted_exit_filled_qty`;
  - entry:
    - anchor the 'new' request's `sent_at`;
    - the real order type and limit price (`live_order_requests.price`);
    - quantity `trade_log.requested_quantity`;
    - window to anchor + `cancel_after_seconds`, or to the ended-early end
      instant (below);
  - exit:
    - anchor the `reference_at` of the position's earliest exit logical
      order;
    - quantity the entry quantity;
    - fill mode and rate by `trade_log.exit_reason` through
      `exit_fill_mode()`;
    - the order-type and price path: the requests of all the position's
      exit logical orders in `sent_at` order, each request's type and price
      applying from its `sent_at`, with no fill between a logical order's
      terminal transition and the next logical order's first request;
    - halts skipped;
    - steps ending at the end of `last_exit_fill_at`'s second, the unfilled
      remainder kept;
  - the exit side is NULL:
    - for an exit_reason other than:
      - take_profit;
      - stop_loss;
      - time_limit;
      - session_end;
    - for a close completed by an adjustment;
    - on an 'entry_canceled' row, whose entry side is still computed;
- the eligible volume `entry_eligible_volume` / `exit_eligible_volume` —
  the fit's Σ Vᵢ terms (below) — over the same span split, before the
  tape's release.

`predicted_*` serve `health_report.md`'s finding 7; the fit reads the real
fills and the eligible volume.

Timing:
- the close transition waits until the ticker's tape covers the trade's
  last second, so no further data is owed then — the former session-end
  timing existed because `simulate_exit_fill()` needs tick data through the
  trade's end, which does not exist yet mid-session;
- running per trade lets the raw 1-tick retention bound advance as
  positions close, and is crash-tolerant per trade, where a session-end
  batch loses the whole day if the process dies first;
- the exception is an exit still unfilled at session end: Session
  Shutdown's carried case computes the same records, the exit steps ending
  at the stage 1 cancel instant, and stores them on the exit logical order
  Session Shutdown cancels (`live_orders`, live_mode_runner.md) unless its
  purpose is 'overnight_liquidation'; the carried position's eventual
  `trade_log` row takes them from there — covering the first exit session
  only.

*`fit_execution_params()`*: runs as the evening batch stage
`evening_execution_param_fit` (metadata_crawler.md):
- a data stage placed before `evening_retention_purge`, with its own
  `batch_runs` row;
- evaluated on every evening run, executing on the calendar week's last
  trading day — same Monday-start / >=3-trading-day convention as the
  divergence check;
- the **data it fits on accumulates across the whole pilot period rather
  than resetting each week** — pilot trade counts are small enough that a
  strict per-week window would rarely clear the sample-size gates below.

Estimation, per (rate, order type), a trade being classified by its first
request's actual order type — rate = Σ qᵢ ÷ Σ Vᵢ over the cumulative pilot
sample:
- qᵢ: the trade's real filled quantity, from `live_fills`' raw `exec_qty`:
  - entry fills for `buy_rate`;
  - exit fills for the `sell_rate_*`;
  - a carried position's first-exit-session fills;
- Vᵢ: the side's eligible volume (`trade_log.entry_eligible_volume` /
  `exit_eligible_volume`; a carried position's `live_orders` raw values) —
  the tick volume after the anchor on which the order could fill under the
  fill simulators' rule, along the real request path, halt-interval ticks
  excluded:
  - the ticks the rule counts:
    - every tick for market;
    - ticks at or below L for an entry limit;
    - ticks at or above the prevailing ladder price for an exit limit;
  - through the end of the second of the window's end:
    - the last real fill, when the order filled its requested quantity;
    - otherwise the entry window's end, or the exit's T_end
      (live_mode_runner.md's close pending);
    - for a carried position, the Session Shutdown stage 1 cancel instant;
- a row whose side's eligible-volume column is NULL is excluded from that
  rate's sample and gate count.

The fit runs no simulation.

Each (parameter, order type) pair is gated independently, using only the
trade_log rows relevant to it:

```
buy_rate:                        gated on its sample's cumulative pilot ENTRY count >= 30
sell_rate_tp:                    gated on cumulative exit_reason='take_profit' count >= 15
sell_rate_sl:                    gated on cumulative exit_reason='stop_loss' count >= 15
sell_rate_neutral:               gated on cumulative exit_reason IN
                                 ('time_limit','session_end') count >= 15
```

A pair whose gate isn't met yet is left unchanged this cycle — the other
pairs (if their own gates are met) still refit independently.

**Carried positions compare in one basis.** A position carried with an exit
logical order has `predicted_*` and eligible volume for its first exit
session only, while its `trade_log` real-side columns blend every session.
For such a position:
- `health_report.md`'s finding 7 compares the `predicted_*` stored on the
  exit logical order Session Shutdown canceled (live_mode_runner.md)
  against its raw entry fills and its exit fills whose `order_date` equals
  that order's first request's (`live_fills`);
- the fit takes qᵢ from those fills and Vᵢ from that order's raw eligible
  volume;
- neither reads `trade_log`'s blended columns.

A carried position with no exit logical order carrying `predicted_*` is
excluded from both, its gate counts included. Every other row compares
`trade_log`'s own columns.

**Entries that ended early.** An entry logical order ended early when it
ended in a status other than 'filled' or 'rejected' and not with
`terminal_cause='timeout'`, whether or not anything filled. Its end
instant is the earlier of the anchor + `cancel_after_seconds` and the
`sent_at` of its first 'cancel' request, or its `terminal_at` when it has
none — computed on time parts of the same session date, through
`utils.hour_add_seconds()`. Ending at that instant, each seeing the window
the real order had:
- the stored `predicted_*` of such an entry, the simulator's calls ceasing
  there (execution_common.md's Fill Simulation Model);
- its eligible volume.

`exit_reason='entry_canceled'` rows (see db_schema.md) are included in the
`buy_rate` sample and its gate count — an entry left unfilled is itself a
direct, relevant observation of participation (qᵢ = 0 against its Vᵢ), not
noise to exclude; one that ended early counts on its matched window.

`exit_reason='entry_rejected'` rows (R-7) are **excluded** from the
`buy_rate` sample — the opposite treatment, for the opposite reason. A
rejection comes from the broker or the account:
- a restriction;
- insufficient funds at the actual price;
- a non-permitted ticker.

It never comes from how the market absorbed the order, so it carries no
evidence about participation rate. Both labels belong to the never-opened
family in
db_schema.md and both count as cooldown attempts in live; they diverge only
here, at what they are evidence FOR.

`trade_log.requested_quantity` is the quantity actually SUBMITTED — i.e.
after `check_funds_available()` has sized the order down — and decides
whether an order filled in full, which ends its Vᵢ window at the last real
fill. A funds-driven reduction is kept out of it for that reason, so it
never reaches this calibration; only market-driven shortfall does. This
matters in practice rather than in principle: full deployment is the
intended operating point,
so the funds gate truncates routinely near the end of a session, and
measuring against the pre-gate size would bias `buy_rate` downward exactly
when the book is fullest.

**Relative-bound check (N-7)**, applied to each (parameter, order type)
pair that clears its sample-size gate above, before it is written:
```
current = get_execution_param(param_name, order_type, db_conn, config)
    # see execution_common.md — already applies the hard bound to `current`;
    # this stage holds a read-write connection, so a bound fallback writes the
    # health event execution_param_bound_fallback
if this is the pair's first-ever fit (no prior execution_params row for
(param_name, order_type)):
    skip this check — nothing to compare against yet; the hard bound
    inside get_execution_param() (see execution_common.md) is the only
    guard on a first fit
elif abs(log(fitted_value / current)) > log(config["execution"]["fit_rejection_multiplier"]):
    reject — do not write to execution_params this cycle. `current`
    remains authoritative. Write the health event
    execution_param_fit_rejected, detail {param, order_type, fitted,
    current} — health_report.md's finding 9, which the evening run's
    evening_end report includes when this stage executed the fit.
    # Deliberately relative to the CURRENT authoritative value, not the
    # seed — calibration is expected to legitimately move the
    # value away from the seed over successive weeks; comparing against
    # the seed forever would eventually reject correct, far-from-seed
    # fits as if they were errors.
else:
    proceed to write (below)
```
Sample-size gating (above) guards against too little data; this guards
against a small-but-sufficient sample that happens to be unrepresentative
— a different failure mode, not a stricter version of the same one, so it
is a separate check rather than a tighter sample-size threshold. Distinct
from `get_execution_param()`'s hard bound (execution_common.md) too — that
one is a mathematically-derived degeneracy check applied at every read
regardless of source; this one is specifically about a fit result jumping
implausibly far from where it already was.

A pair that clears BOTH its sample-size gate and the relative-bound check
gets a new row in `execution_params` — see db_schema.md:
- the row holds:
  - `order_type`;
  - the fitted value;
  - `fitted_at`;
  - the cumulative `sample_size` used;
  - the triggering `week_start`;
- `get_execution_param()` (execution_common.md) resolves the latest row per
  pair once per top-level run, in preference to the `execution:` config
  seed:
  - LiveModeRunner's session start;
  - run_backtest.py's start;
  - pipeline_optimizer's coordinator;
- the config value is the pre-pilot seed only.

*Divergence alert*: `health_report.py` gets a new finding, separate from
the winning-rate divergence — predicted-vs-actual fill price gap
exceeding a threshold (exact threshold TBD, deferred alongside the other
shadow-mode-adjacent numeric thresholds). This is deliberately a distinct
finding, not folded into the winning-rate divergence: a good model with
bad fill assumptions and a bad model with good fill assumptions are
different failure modes needing different responses (retrain vs.
recalibrate), and collapsing them into one signal would make both harder
to diagnose.

## Stage 3: Scale

Exposure caps relaxed to their steady-state operating values. Ordinary
retraining cadence (below) begins governing the deployed model from this
point.

**Gate on entering Stage 3**: at least one successful `fit_execution_params()`
refit (any (parameter, order type) pair clearing its sample-size gate at
least once) must have occurred during Pilot. Moving to Scale before any
calibration has happened defeats Pilot's purpose — exposure would increase
on execution assumptions that were never actually checked against real
fills.

---

## Retraining Cadence

Two independent triggers, whichever fires first:

**Calendar trigger** — `optimizer.refresh.calendar_cadence_days` (config,
default 30 / monthly-level). Dispatches to PipelineOptimizer's `"full"`
phase: existing hyperparameters and `selected_features.json` retained,
model refit on data extended through the current date. Deliberately
lightweight — a routine refresh is not expected to need a fresh
hyperparameter search every cycle.

**Divergence trigger** — live rolling winning rate compared against a
backtest-derived confidence interval, evaluated on a **calendar-week**
window (Monday-start, a week with fewer than 3 trading days is excluded —
reusing `ClassBalancer.generate_folds()`'s existing week-boundary
convention rather than introducing a second, differently-behaved
definition of "week" into the project):

```
For the most recently completed calendar week:
    live_winning_rate = trade_log winning rate for that week (is_shadow=FALSE)
    backtest_ci = bootstrap-resampled CI of winning rate over an
                  equivalent-length backtest window, using the currently
                  deployed model — computed by utils.bootstrap_ci() with
                  the optimizer.bootstrap keys (confidence 0.95); the
                  resampling unit this trigger passes is undecided
                  (open_items.md)
    live_winning_rate outside backtest_ci → divergence flag for that week
```

This is the same computation `health_report.md`'s divergence finding uses
(see that doc's `gather_findings()` placeholder) — implemented once, read
by both the per-session alert and the retraining trigger, not duplicated.

If a divergence flag fires: `optimizer.refresh.divergence_reoptimize`
(config, default `true`) dispatches to PipelineOptimizer's `"exploitation"`
phase instead of `"full"` — a severe enough live/backtest mismatch is
treated as grounds for a fresh hyperparameter search, not just a data
refresh, on the theory that the existing hyperparameters may no longer fit
the current market regime.

---

## Constraints

- Shadow mode's hypothetical fills reuse `execution_common.md`'s
  `simulate_entry_fill()`/`simulate_exit_fill()` — no separate shadow-only
  fill-estimation logic; the fill model is the only thing the shadow takes
  from backtest, everything else following the real path
- The divergence check's calendar-week convention must match
  `ClassBalancer.generate_folds()`'s exactly (Monday start, >= 3 trading
  days to count) — a second, independently-behaving definition of "week"
  in the same project is a correctness risk, not a style preference
- `shadow_duration_weeks: 0` (sync) reads `class_balancer.outer_fold.test_weeks`
  at the time shadow mode starts — not re-read continuously during the
  shadow period if that config changes mid-run
- Pilot-stage sizing uses the same `execution.exposure_cap_pct` /
  `per_ticker_share_cap_pct` mechanism as any other exposure limit — there
  is no pilot-specific config key
- Calendar and divergence retraining triggers are independent — either
  firing is sufficient; they are not required to agree
- `fit_execution_params()`'s (parameter, order type) pairs are gated and
  refit independently — one pair clearing its sample-size threshold does
  not imply or require the others have too
- `fit_execution_params()` estimates each rate as Σ real filled quantity ÷
  Σ eligible tick volume and runs no simulation; `predicted_*` serve
  finding 7 only; `cancel_after_seconds` is not calibrated
- `fit_execution_params()`'s sample accumulates across the entire pilot
  period (not reset per calendar-week) — only the run *cadence* is weekly,
  the data window is cumulative-since-pilot-start
- A position carried with an exit logical order is compared on these,
  never on `trade_log`'s blended real-side columns:
  - its raw entry fills from `live_fills`
  - its first exit session's exit fills from `live_fills`
  - its eligible volume from that exit logical order
- A carried position with no exit logical order carrying `predicted_*` is
  excluded
- An entry that ended early is simulated, and its eligible volume measured,
  on its matched window; it counts toward `buy_rate`'s sample and gate
- Model-retraining divergence (winning rate vs. backtest CI) and
  execution-parameter divergence (predicted vs. actual fill price) are
  independent checks with independent `health_report.md` findings — never
  collapsed into one signal, even though both reuse the calendar-week
  cadence
- `execution_params` table rows are the effective values once any exist for a
  given (`param_name`, `order_type`) pair; the `execution:` config block is a
  seed default, read only before the first fit — not a fallback re-consulted
  later

---

## Operational Notes

**Timezone / DST Discipline.** All schedule times in this system are
wall-clock America/New_York times, which observe DST:
- 03:00 ET `--token-refresh` — see metadata_crawler.md's "Overnight Token
  Refresh"; it writes no `batch_runs` row, its failure surfacing through
  the `token_refresh_failed` alert;
- 04:00 ET `--premarket-open`;
- 05:00 ET `premarket_wait_deadline` — the premarket lock probe's and
  Session Start Gating's deadline;
- 09:20 ET `--premarket-recheck` — R-1: the 09:20 pass is now a
  LiveModeRunner in-process task, not a cron entry — see
  metadata_crawler.md's "Dual Schedule", N-1/N-3;
- 09:30 regular session open;
- `last_bar(date)` session close;
- `session_close(date)` regular session close — past which the venue
  refuses market orders, so order type branches on it; see
  execution_common.md's session-phase table for the permission and
  utils.md's Session Boundary Derivations for the instant;
- `after_hours_end(date)` — also the live process's R-9 hard cap once
  `live_mode.session_hard_exit_offset_minutes` is subtracted, and so ONE
  entry here rather than two;
- 21:00 ET evening batch run;
- 23:30 ET `evening_wait_hard_deadline`.

The set above is every wall-clock time the system schedules or gates
behaviour against, whatever the mechanism:
- a cron entry;
- an in-process scheduled task;
- a market boundary the code branches on.

That criterion is why 09:20 stays listed after it ceased to be a cron. This
enumeration is a convenience copy, so it can be compared mechanically
against each value's authority — where the value is set:
- a config key;
- a crontab entry — one whose time a config key also states (03:00,
  `token_refresh_time`) is set from that key, a divergence resolved in the
  key's favour;
- `trading_calendar`, through utils.md's Session Boundary Derivations;
- execution_common.md's session-phase table literal.

The deployment host/container
must be NTP-synchronised as well (R-8): Bar-Close Authority's wall-clock
deadlines and the Feed Outage trigger assume sub-second skew against exchange
time, and this is enforced rather than merely recommended — LiveModeRunner's
`clock_check` probe aborts the session by default when the offset exceeds
`max_offset_seconds`. The host
must set its system TZ to `America/New_York` (not a fixed UTC offset), so
that all cron schedules, the in-process recheck's scheduled time, and
bar-close authority (see live_mode_runner.md)
shift correctly across the two annual transitions. Operator checklist: on
the Monday following each DST transition (mid-March, early November),
before trusting the session, verify:
- the 03:00 token refresh — each account's SDK token cache `expires_at`
  equalling that day's `token_refresh_time` + 24h, or that day's
  `token_refresh_failed` alert;
- every other scheduled run — that day's `batch_runs` rows landed at the
  expected wall-clock times.

**This list is where the system's two distinct close concepts became visible
side by side**, and naming them is what the calendar column made possible: the
LAST BAR (`last_bar()`, ordinarily 15:59) and the EXCHANGE CLOSE
(`session_close()`, ordinarily 16:00) differ by a minute and were both written
as bare times here. On an early-close day they are 12:59 and 13:00, and
`after_hours_end` is 17:00 rather than 20:00. The 21:00 and 23:30 slots do NOT
move — see metadata_crawler.md for why the evening cron is not pulled earlier.
The session-boundary entries above are now DERIVED rather than fixed, which does
not remove them from the enumeration: the criterion is that the system gates
behaviour on the value, not that the value is a literal. Membership rule for the
comparison described above — an entry is compared against its authority:
- a config key;
- a crontab entry;
- the `trading_calendar` column;
- the session-phase table literal.

**DuckDB Backup/Recovery.** For `data/market.duckdb`:
- Backup: the evening run's final stage, `evening_backup`
  (metadata_crawler.md's Nightly Backup), its window per db_schema.md's DB
  File Ownership Windows, the single site of that design.
- Retention: follows the posture the table purge registry takes — nightly
  copies are kept without limit until an operator sets a window against
  observed disk growth, so nothing is deleted by default and the growth is
  the operator's to watch.
- Recovery: a plain file-copy restore of the single `.duckdb` file — any
  existing `market.duckdb.wal` is removed before the copy is placed; no
  WAL/point-in-time replay involved.
