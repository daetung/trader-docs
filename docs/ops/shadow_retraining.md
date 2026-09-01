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

**Execution-parameter calibration** (`buy_rate`, `sell_rate_tp`,
`sell_rate_sl`, `sell_rate_neutral`, `cancel_after_seconds` — see
execution_common.md's `execution:` config). Real fills only exist from this stage onward, which
is what makes this calibration possible at all — Shadow (Stage 1) has no
real fill to compare a prediction against, so this is Pilot-only, not
something Shadow's comparison already covers (Shadow's comparison is
aggregate winning_rate/avg_pnl_pct only — a different, independent check;
see the divergence-check note under Retraining Cadence below for how the
two relate).

*Data collection*: AT EXIT COMPLETION, for every pilot-stage trade,
`simulate_entry_fill()`/`simulate_exit_fill()` are run counterfactually
against the same tick data the real order saw, writing
`trade_log.predicted_fill_price` / `predicted_weighted_avg_exit_price` /
`predicted_partial_fills_count` alongside the real `fill_price` /
`weighted_avg_exit_price` / `partial_fills_count` on the same row — which
is what lets that row be written COMPLETE in a single INSERT rather than
inserted and updated later.
This used to run at session end, and the reason given was that
`simulate_exit_fill()` needs that day's remaining tick data through session
close, which does not exist yet mid-session. That is exactly what the
real-path-parallel incremental form resolves (live_mode_runner.md): the
simulation walks forward from its anchor and recomputes until it settles,
and settlement IS the moment no further data is owed. Moving it also lets
the raw 1-tick retention bound advance as positions close instead of every
closed position pinning it to session end, collapses Shadow and Pilot onto
one mechanism — same function, same anchor, same instant, differing only in
whether a real fill sits beside the simulated one — and is crash-tolerant
per trade, where a session-end batch loses the whole day if the process dies
first. The single exception is an exit still unfilled at session end, which
is treated as settled at that moment, the same instant its slot is released.
`fit_execution_params()` itself is UNCHANGED and still runs at or after
session end: what moved is when `predicted_*` is populated, not when the fit
runs — a fit needs a population, not one trade.

*`fit_execution_params()`*: runs weekly (calendar-week, same Monday-start /
>=3-trading-day convention as the divergence check), but the **data it
fits on accumulates across the whole pilot period rather than resetting
each week** — pilot trade counts are small enough that a strict per-week
window would rarely clear the sample-size gates below. Each of the five
parameters is gated independently, using only the trade_log rows relevant
to it:

```
buy_rate, cancel_after_seconds:  gated on cumulative pilot ENTRY count >= 30
sell_rate_tp:                    gated on cumulative exit_reason='take_profit' count >= 15
sell_rate_sl:                    gated on cumulative exit_reason='stop_loss' count >= 15
sell_rate_neutral:               gated on cumulative exit_reason IN
                                 ('time_limit','session_end') count >= 15
```

A parameter whose gate isn't met yet is left unchanged this cycle — the
other four (if their own gates are met) still refit independently.
`exit_reason='entry_canceled'` rows (see db_schema.md) are included in the
entry-count denominator for `buy_rate`/`cancel_after_seconds` — a fully-
unfilled entry is itself a direct, relevant observation for those two
parameters, not noise to exclude.

`exit_reason='entry_rejected'` rows (R-7) are **excluded** from that same
denominator — the opposite treatment, for the opposite reason. A rejection
comes from the broker or the account (restriction, insufficient funds at
the actual price, a non-permitted ticker), never from how the market
absorbed the order, so it carries no evidence about participation rate or
about whether `cancel_after_seconds` is set too tight. Both labels belong
to the never-opened family in db_schema.md and both count as cooldown
attempts in live; they diverge only here, at what they are evidence FOR.

Fill-rate inputs read `trade_log.requested_quantity`, which is the quantity
actually SUBMITTED — i.e. after `check_funds_available()` has sized the
order down. A funds-driven reduction therefore never reaches this
calibration; only market-driven shortfall does. This matters in practice
rather than in principle: full deployment is the intended operating point,
so the funds gate truncates routinely near the end of a session, and
measuring against the pre-gate size would bias `buy_rate` downward exactly
when the book is fullest.

**Relative-bound check (N-7)**, applied to each parameter that clears its
sample-size gate above, before it is written:
```
current = get_execution_param(param_name, db_conn, config)  # see
                                                              # execution_common.md
                                                              # — already
                                                              # applies the
                                                              # hard bound
                                                              # to `current`
if this is the parameter's first-ever fit (no prior execution_params row):
    skip this check — nothing to compare against yet; the hard bound
    inside get_execution_param() (see execution_common.md) is the only
    guard on a first fit
elif abs(log(fitted_value / current)) > log(config["execution"]["fit_rejection_multiplier"]):
    reject — do not write to execution_params this cycle. `current`
    remains authoritative. Log + surface via health_report.md's finding 9.
    # Deliberately relative to the CURRENT authoritative value, not the
    # 0.1/30 seed — calibration is expected to legitimately move the
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

A parameter that clears BOTH its sample-size gate and the relative-bound
check gets a new row in `execution_params`
(fitted value, `fitted_at`, the cumulative `sample_size` used, and the
triggering `week_start`) — see db_schema.md. BacktestEngine and
LiveModeRunner both read the latest `execution_params` row per parameter,
via `get_execution_param()` (execution_common.md), in preference to the
`execution:` config seed at session start; the config value is the
pre-pilot seed only.

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
refit (any of the five parameters clearing its sample-size gate at least
once) must have occurred during Pilot. Moving to Scale before any
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
    backtest_ci = bootstrap-resampled 95% CI of winning rate over an
                  equivalent-length backtest window, using the currently
                  deployed model
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
  fill-estimation logic
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
- `fit_execution_params()`'s five parameters are gated and refit
  independently — one parameter clearing its sample-size threshold does
  not imply or require the others have too
- `fit_execution_params()`'s sample accumulates across the entire pilot
  period (not reset per calendar-week) — only the run *cadence* is weekly,
  the data window is cumulative-since-pilot-start
- Model-retraining divergence (winning rate vs. backtest CI) and
  execution-parameter divergence (predicted vs. actual fill price) are
  independent checks with independent `health_report.md` findings — never
  collapsed into one signal, even though both reuse the calendar-week
  cadence
- `execution_params` table rows are the effective values once any exist for
  a given `param_name`; the `execution:` config block is a seed default,
  read only before the first fit — not a fallback re-consulted later

---

## Operational Notes

**Timezone / DST Discipline.** All schedule times in this system (03:00 ET
`--token-refresh` — see metadata_crawler.md's "Overnight Token Refresh" —
04:00 ET `--premarket-open`, 09:20 ET `--premarket-recheck` — R-1: the
09:20 pass is now a LiveModeRunner in-process task, not a cron entry — see
metadata_crawler.md's "Dual Schedule", N-1/N-3 — 09:30 regular session
open, `last_bar(date)` session close, `session_close(date)` regular session
close — past which the venue refuses market orders, so order type branches on
it, see execution_common.md's session-phase table for the permission and
utils.md's Session Boundary Derivations for the instant — `after_hours_end(date)`,
which is also the live process's R-9 hard cap once
`live_mode.session_hard_exit_offset_minutes` is subtracted and so is ONE entry
here rather than two, 21:00 ET evening batch run, 23:30 ET
`evening_wait_hard_deadline`) are wall-clock America/New_York times, which
observe DST. The set above is every wall-clock time the system schedules or
gates behaviour against, whatever the mechanism — a cron entry, an
in-process scheduled task, or a market boundary the code branches on; that
criterion is why 09:20 stays listed after it ceased to be a cron. Authority
for each value is its own config key, this enumeration being a convenience
copy, so the two can be compared mechanically and a divergence resolved in
the config's favour. The deployment host/container
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
verify that day's `batch_runs` rows landed at the expected wall-clock
times before trusting the session.

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
comparison described above — an entry whose authority is a config key is compared
against that key; an entry whose authority is `trading_calendar` is compared
against the column.

**DuckDB Backup/Recovery.** `data/market.duckdb` is backed up via a
nightly file-level copy, scheduled in the quiescent window after the
evening batch completes and before the next premarket batch begins —
concretely, after `batch_runs` shows `stage='evening_session_stats',
status='success'` for the day (see db_schema.md's DB File Ownership
Windows, the single site of that design). Retention: N most recent
nightly copies (N TBD) plus one longer-retained weekly snapshot. Recovery
is a plain file-copy restore of the single `.duckdb` file — no
WAL/point-in-time replay involved.
