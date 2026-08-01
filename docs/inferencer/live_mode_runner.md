# Module: LiveModeRunner

**File:** `src/inference/live_mode_runner.py`

---

## Role

Execution orchestrator for live trading mode.
Manages two independent loops (watchdog polling and position monitoring),
coordinates session initialization, and integrates all live-mode components.

LiveModeRunner is the **execution subject** — Inferencer, CachingIndicatorCalculator,
and all other live-mode modules are service objects instantiated and owned by this class.

---

## Architecture

```
LiveModeRunner
    ├── calculators: dict[str, CachingIndicatorCalculator]
    │       One instance per ticker in today's tradable universe.
    │       Populated eagerly at session_start via parallel session_start_compute().
    │       Active watchlist populated lazily as watchdog events arrive.
    ├── FeatureExtractor              (uses CachingIndicatorCalculator via DI)
    ├── Inferencer                    (owns FeatureExtractor)
    ├── EntryPointDetector            (used internally by Inferencer)
    ├── [Watchdog polling loop]       (async / threaded)
    └── [Position manager loop]       (async / threaded)
```

---

## DB Connection Management (P-5)

Two connections, not one — this is what makes the ad-hoc-operator-query and
premarket-cron contention problem tractable without new infrastructure:

```python
read_only_conn:  opened with read_only=True — used for every SELECT this
                 session issues (Session Lifecycle bulk loads, any other
                 read). Permits concurrent read-only connections from
                 elsewhere (ad-hoc operator queries) without contention.
write_lock = Lock()
writer_conn:     single connection, all writes funneled through it —
                 inference_log, indicator_cache (any mode as of R-2 — see
                 below; previously "db" mode only), trade_log (shadow
                 mode), batch_runs (this session's own live_session_start
                 / live_session_end markers — see Session Lifecycle Step
                 1c and Session Shutdown below), and, as of R-2,
                 live_positions and live_session_state (see "Session
                 Restart (Warm Start)" below) plus the In-Process
                 Premarket Recheck task's writes (R-1, see that section —
                 same funnel, no second writer).

def db_write(sql, params):
    with write_lock:
        writer_conn.execute(sql, params)
    # synchronous — returns only after commit. This is what makes
    # read-your-own-write correctness free: any db_read() called after
    # a db_write() returns is guaranteed to see it, with no queueing or
    # ordering logic needed.

def db_read(sql, params):
    return read_only_conn.execute(sql, params).fetchall()
    # always a fresh query — never reuse a long-lived transaction across
    # calls (DuckDB's MVCC snapshot would otherwise pin a stale view and
    # miss the writer's later commits — a subtle, hard-to-reproduce bug
    # if violated).
```

## Session Start Gating (P-5 temporal ownership)

Before opening either connection above, LiveModeRunner waits on the
premarket batch's completion markers — this is a *timing* concern (don't
open a connection while the premarket job might still be writing),
distinct from Health Gate 1's premarket-marker check (a *data-completeness*
concern evaluated afterward, once the session has already started):

```
wait_deadline = 05:00:00 ET   # up to 1 hour from the premarket batch's
                               # 04:00:00 ET start (see metadata_crawler.md)
poll batch_runs every N seconds (polling interval unspecified — cheap,
    low-stakes choice) for stage IN ('premarket_rename',
    'premarket_corporate_events') AND date = today_date, status='success'
if both reach 'success' before wait_deadline: proceed immediately
elif wait_deadline reached first: proceed anyway (this IS the "degraded
    start" path — Health Gate 1c's warn-and-suppress-affected-tickers
    behavior is what actually handles the consequences; this wait's only
    job is not opening a DB connection while the premarket job might still
    hold write access)
```

## Session Start Probes (R-7 / R-8)

Three cheap, one-shot checks run after Session Start Gating and before the
Watchdog loop opens. Each is a single call; none is on a hot path.

**Clock offset (R-8).** Config `clock_check`: `source` (`"ntp_daemon"` default
| `"vendor_api"` | `"disabled"`), `max_offset_seconds` (default 1.0),
`on_exceed` (`"abort"` default | `"warn"`).

`"ntp_daemon"` reads the local NTP daemon's own reported offset — no vendor
contract needed, the dependency closes inside the host. `"vendor_api"` is a
PLACEHOLDER: whether the trading API exposes a server clock is itself
unverified (api_contract_checklist.md), so the option exists to be selected
later, not to be implemented now.

1.0s is not arbitrary. Bar-Close Authority judges in whole seconds, scoped to
each cycle's own `candidates` (see Feed Outage Recovery's trigger condition
2, not the full watchlist) — past ~1s of skew, EVERY candidate in a cycle
misses simultaneously, so 1s is the boundary at which a clock fault starts
impersonating a feed outage, in any cycle large enough
(`len(candidates) >= min_watchlist_size`) for condition 2 to evaluate at
all. Below that floor this particular impersonation risk is moot for the
cycle — condition 2 does not fire regardless of clock health, leaving
condition 1 (explicit connection failure) as the only detector.

`abort` is the default because a skewed clock disguises itself as other
faults: it looks like a dead feed, it freezes exits along with entries, and it
misaligns `fill_second` against the vendor's tick timestamps so that
`fit_execution_params()`'s counterfactual replay lands on the wrong ticks and
biases `buy_rate`/`sell_rate` with no error anywhere. Refusing to start beats
running with silently corrupted calibration.

An `abort` here fires strictly before health_report.py's own session-start
invocation, so without a deliberate exception the operator learns nothing —
"no positions today because of a clock fault" and "an ordinary quiet day"
look identical from outside. On abort, send health_report immediately (at
`abort` severity) before terminating — the same mid-session trigger
mechanism Circuit Breaker uses below, reused rather than duplicated.

**Margin ratio (R-8).** Queried from its own endpoint (config key for the path;
see api_contract_checklist.md) and held as a session constant, consumed by
`execution.sizing_basis` (execution_common.md). On failure: fall back to
`live_mode.margin_ratio_fallback` (default 4.0, a typical day-trading
buying-power multiple) plus a health_report finding. Never aborts: this
refines sizing, it does not authorise trading.

The fallback is deliberately NOT 1.0. `sizing_basis: "equity"` divides the
queried balance by margin_ratio; the safe direction for that division is the
LARGER divisor, because whether the balance figure itself is cash or buying
power is still unverified (api_contract_checklist.md T-4). If it is buying
power and the fallback were 1.0, "fully deployed" would silently mean the
full buying-power figure — the exact multiplication `sizing_basis` exists to
prevent, arriving at the one moment its own input is missing. A fallback
that turns out too conservative once T-4 is verified only under-sizes, which
costs opportunity, not solvency — the same one-sided-error reasoning
`sizing_basis`'s own default already applies.

Background: the PDT designation and its $25,000 floor were removed from FINRA
Rule 4210 effective 2026-06-04, replaced by a risk-based INTRADAY margin
standard tied to live exposure. That is why this is queried rather than
encoded — the requirement now moves with position state, brokers phase the new
framework in on their own schedules through 2027-10-20, and house rules may be
stricter than the regulatory floor. Modelling the rule in code would be wrong
for some broker at some date; asking is not.

One query at session start does not track intraday movement, and it is not
meant to. The binding constraint is enforced by the broker, and it becomes
visible to us as rejected submissions (`entry_rejected` with `reject_reason`).
Sizing's job is only to avoid submitting what will obviously be refused.

**Retention boundary (R-7).** Config `retention_probe`: `enabled`,
`lookback_days` (ask from further back than retention is expected to reach),
`assumed_days` (fallback).

Measured, not modelled: request history for one liquid ticker from
`now - lookback_days` and record the OLDEST timestamp actually returned as
`retention_boundary`, a session constant. This sidesteps having to guess
whether the vendor counts trading days or calendar days — the returned
timestamp is the answer either way. On failure or an empty response, fall back
to `assumed_days` and raise a health_report finding.

Consumed by Warm Restart gap-fill and Feed Outage Recovery: compare the gap's
start against `retention_boundary` BEFORE attempting to fill it. Today an
unrecoverable gap is discovered by asking and getting nothing back; this makes
it a prior judgement, so the crash path skips a pointless round trip at its
most time-critical moment and goes straight to `restart_gap_exit`. The
measured value is also logged every session, which fills in
api_contract_checklist.md's retention row from ordinary operation rather than
from a one-off measurement.

### session_diagnostics write protocol (R-9)

The three probe results above, plus Health Gate 2's tier-fallback summary
and two running counters, are persisted to
`live_session_state.session_diagnostics` (JSON — see db_schema.md) instead
of being held as in-memory tallies handed to `health_report.py` at session
end. A crashed session previously lost findings 5, 8, 21 and 22 outright;
in the table the evening liveness probe can recover them.

**Always write-only whole-value replacement.** The runner accumulates the
diagnostics dict in memory and replaces the column with it. There is never
a read-modify-write of the JSON, at any of the write points below, so no
atomicity question arises.

Write points:
```
(a) immediately after the probes above, BEFORE Health Gate 2
      clock_offset_start, clock_offset_end=null, margin_ratio,
      retention_boundary; each probe flagged 'disabled' | 'succeeded' |
      'fell_back' so "never ran" and "ran and fell back" stay distinct
(b) immediately after Health Gate 2
      adds the tier-fallback summary (finding 5)
(c) on the existing bar_latency_daily flush cadence
      refreshes the running counters — halt-check signal_source counts
      (finding 8) and inference_log dropped-row count (finding 22)
(d) at Session Shutdown
      fills clock_offset_end (finding 21)
```
(a) must precede Gate 2 rather than being folded into (b): `clock_check`
with `on_exceed: "abort"` terminates the session before Gate 2 is reached,
and the measured offset is the only evidence for why. The evening probe
recovers the crash FACT (finding 11) but never the measurements.

`clock_offset_end` is written as `null` at (a) rather than left absent, so
its absence at report time reads as "session did not reach shutdown" rather
than as a missing key.

A flush failure at (c) is swallowed per db_schema.md's standing rule, and
must NOT increment finding 22's own counter — otherwise a persistent write
fault feeds itself.

**On warm restart**, the probe values at (a) are OVERWRITTEN with the newly
measured ones and `restart_count` increments. They are current operating
settings, not a session baseline — unlike `session_start_cash`, which R-2
restores rather than re-queries precisely because it IS the baseline. The
counters at (c) RESUME from the persisted values instead of resetting to
zero; they are session totals.

## Session Shutdown

**Exit trigger (R-9).** The process exits at whichever of these comes
first:
```
(a) all positions are flat AND now >= config["execution"]["session_close_exit_time"]
(b) now >= config["live_mode"]["session_hard_exit_time"]   # hard cap
```
(a) is the ordinary path — on a normal day the 15:59 `session_end` exits
fill within minutes and the process leaves shortly after 16:00. (b) exists
because submitting an exit is not the same as filling one: a limit exit is
re-priced every `position_check_interval_seconds` and escalates to market
at `exit_order_stuck_minutes`, and exit tracking has NO give-up timeout, so
without a cap a single unfillable position would keep the process alive
indefinitely.

`session_hard_exit_time` defaults to `"20:00"` ET. This is not a new
number: it is the after-market ingestion boundary (`200000`, see
db_schema.md's Ingestion Rules) and it is the value health_report.md's
`drain_timeout_seconds` was already sized against — roughly 50 minutes of
margin before the 21:00 evening batch. Making it a config key turns that
arithmetic's assumption into a declared value.

**Positions still open at the hard cap** are NOT force-liquidated. They are
left as `live_positions` rows and handed to the next session's Broker
Reconcile under the Unified Overnight Policy (see below) — the same path a
halt-through-close position already takes. Forcing a market exit into
after-hours liquidity would be strictly worse than the overnight carry it
is trying to avoid, and the row is durable (R-2), so the next session
adopts it rather than discovering an orphan.

**In-flight exit orders are canceled at shutdown**, before the markers
below. An order left resting after the process dies could fill overnight
with nothing watching it, and the position's state would then be wrong at
the next session's reconcile. Note this is a wider cancellation than Broker
Reconcile's, which cancels pending ENTRY orders only.

LiveModeRunner's last action before process exit, after both loops have
been signaled to stop and any final position/order state has settled:
```
db_write(
    "UPDATE batch_runs SET status='success', finished_at=?
     WHERE stage='live_session_end' AND date=?",
    ...
)
# or INSERT if no 'running' row was created at session start — either is
# fine as long as exactly one live_session_end row per date results.

# R-2: also flip the session's own start marker (written 'running' at
# Session Lifecycle Step 1c) to 'success':
db_write(
    "UPDATE batch_runs SET status='success', finished_at=?
     WHERE stage='live_session_start' AND date=?",
    ...
)
# Invariant this maintains: a today live_session_start row still 'running'
# with no live_session_end row = crashed session — this is exactly what
# warm-restart detection keys on (see "Session Restart (Warm Start)"
# below). A clean shutdown always flips both markers together.
```
The evening batch job (metadata_crawler.md) waits on this marker before
its own first stage begins — see metadata_crawler.md's Dual Schedule.

---

## Session Lifecycle

```
LiveModeRunner.start_session(today_date):

  1. Fetch today's tradable ticker list from trading service API
         all_tickers: list[str]
         (tickers with prior-day data but no trading access today
          are automatically excluded by the API response)

  1b. Query account balance from trading API:
         session_start_cash: float
      Fixed for the entire session — this is the `balance` argument to
      every `execution_common.compute_position_size()` call this session
      makes (see Watchdog Polling Loop step 5c.ii), mirroring backtest's
      `initial_cash` (see 09_backtest_engine.md's Position Sizing). NOT
      re-queried mid-session even if real account balance changes as
      trades fill — sizing is meant to use a fixed reference, same
      rationale as backtest's fixed initial_cash (see execution_common.md's
      Constraints for why sizing and funds-availability must not share a
      value). The funds-availability gate
      (`execution_common.check_funds_available()`) is what uses a
      real-time queried balance instead — see that call site.

  1c. Persist session start (R-2):
         db_write INSERT batch_runs stage='live_session_start',
             date=today, status='running'.
         db_write INSERT live_session_state (date, run_id,
             session_start_cash, started_at).
      This is the authoritative sizing basis for the session — a warm
      restart (see "Session Restart (Warm Start)" below) restores it
      rather than re-querying the balance, which would reflect a
      post-fills value, not the session's original basis.

  1d. Broker Reconcile (shared procedure — R-3, see "Broker Reconcile
      (shared procedure)" below): run at EVERY session start, cold start
      included. At cold start, any broker position found is by
      definition an overnight orphan — nothing has opened yet today. This
      is the standing guard against a prior-day position that never
      appeared in a warm restart (e.g. a cleanly-shut-down session that
      left a halt-through-close position). Warm restart already reconciles
      in its own step 1 and does not double-run this.

  2. Bulk load session_stats for ALL tickers from DuckDB
         session_stats_raw = SELECT * FROM precomputed_session_stats
                             WHERE as_of_date = today_date
                               AND n_sessions = config_n_sessions
         # as_of_date = today was inserted by collect_daily.py the previous evening
         # and contains baselines from the prior N sessions.

         all_stats_nested = build_session_stats_dict(
             session_stats_raw,
             delta_minutes=config["indicators"]["reference_session"]["delta_minutes"],
             session_mode=config["entry_detector"]["session_mode"],
         )
         # all_stats_nested: {today_date: {ticker: {metric: {hour: smoothed_avg_value}}}}
         session_stats_bulk = all_stats_nested[today_date]
         # session_stats_bulk: {ticker: {metric: {hour: smoothed_avg_value}}}

  3. [Meta Bulk Load] Bulk load stock_meta for today, with per-field fallback:
         stock_meta_today = SELECT * FROM stock_meta WHERE date = ?
         # stock_meta is (ticker, date)-keyed (see db_schema.md V-1 fix); a row
         # exists only if today's crawl (see metadata_crawler.md dual-schedule)
         # succeeded for that ticker/field by session start.

         meta_bulk: dict[str, dict] = {}
         for ticker in all_tickers:
             meta_bulk[ticker] = {
                 field: (
                     stock_meta_today.get(ticker, field)
                     if ticker in stock_meta_today.index and pd.notna(stock_meta_today.loc[ticker, field])
                     else utils.estimate_historical_meta(ticker, today_date, field, db_conn)
                 )
                 for field in ["market_cap", "shares_outstanding", "price_52h", "price_52l"]
             } | {
                 "sector": most recent available stock_meta.sector for ticker
                 # sector has no derivation path — most recent snapshot regardless of date
             }
         # meta_bulk is looked up (not rebuilt) inside the watchdog loop's
         # Inferencer.infer() call — see Watchdog Polling Loop step 5b.
         # estimate_historical_meta() fallback should rarely trigger for
         # today's own date given the dual-crawl schedule, but is not
         # guaranteed-zero (vendor update latency — see metadata_crawler.md).
         # Moved ahead of Eager Pool (was Step 3b, after) specifically so
         # Health Gate 1 (next) can evaluate stock_meta coverage before
         # paying Eager Pool's cost — see Health Gate 1 rationale below.

  4. [Health Gate 1] Pre-Eager-Pool health checks — evaluated after Steps 2-3,
     before the expensive Eager Pool parallel compute. Two of four checks
     (a, b) are measured directly from data already loaded above; the other
     two each require their own lookup — c a batch_runs marker query, d
     (N-8) the model artifact's feature_names, loaded here rather than at
     its originally-later point (see d's own note):

     a. session_stats coverage:
            eligible = SELECT ticker FROM ticker_data_coverage
                       WHERE date = {prior_trading_day} AND has_1min = TRUE
            # tickers with no prior-day trading activity are excluded from
            # the denominator — their session_stats being empty is expected,
            # not a batch-health signal.
            coverage_rate = |{t in eligible ∩ all_tickers
                              : session_stats_bulk[t] not empty}|
                            / |eligible ∩ all_tickers|
            coverage_rate < 0.90 → ABORT SESSION (hard gate; P-0-class failure)

     b. stock_meta coverage (per field, then aggregated):
            for field in [market_cap, shares_outstanding, price_52h,
                          price_52l, sector]:
                eligible(field) = { t : EXISTS (SELECT 1 FROM stock_meta
                                    WHERE ticker=t AND date >= today - 20
                                    trading days AND {field} IS NOT NULL) }
                # a ticker that has never carried this field (vendor/ticker-
                # type limitation) is excluded from that field's denominator
                # — only a ticker that normally HAS the field but is missing
                # it today counts against coverage.
            coverage_rate = Σ_field |{(t,field) : t in eligible(field),
                                       real value present today}|
                            / Σ_field |eligible(field) ∩ all_tickers|
            coverage_rate < 0.80 → WARN
            coverage_rate < 0.50 → ABORT SESSION

     c. premarket completion marker:
            SELECT status FROM batch_runs
            WHERE stage IN ('premarket_rename', 'premarket_corporate_events',
                             'premarket_quarantine_check')
              AND date = today_date
            missing or status != 'success'
                → WARN + proceed with yesterday's state
                → suppress new entries on any ticker where
                  is_tradable() returns False
                  (enforced downstream — see Watchdog Polling Loop)
            # 'premarket_quarantine_check' added for P-8's
            # check_corporate_event_anomaly() — that stage's own writes
            # use ticker_cik_map.quarantine_reason (N-2 — see
            # db_schema.md), a column independent of the rename case's
            # rename_pending, so it needs no separate handling here beyond
            # being included in this marker check; a ticker it would have
            # evaluated but didn't reach is simply not yet protected
            # today, not incorrectly blocked (see that function).
            # 'premarket_quarantine_recheck' (the 09:20 ET pass, now a
            # LiveModeRunner in-process task — see "In-Process Premarket
            # Recheck" — R-1) is deliberately NOT in this list — it runs
            # well after Health Gate 1 (which fires shortly after Session
            # Start Gating, around 04:00-05:00 ET), so its completion
            # cannot be known yet at this check regardless of stage name.

     d. feature_names consistency (N-8):
            model_features = set(meta["feature_names"])   # from the run_id's
                                                            # model artifact
                                                            # meta.json (see
                                                            # base_model.md)
            extractor_features = set(FeatureExtractor(config).get_feature_names())
            mismatch = model_features.symmetric_difference(extractor_features)
            mismatch is non-empty
                → ABORT SESSION (hard gate)
            # No warn-and-proceed variant considered — a mismatch means
            # every inference this session would either KeyError on a
            # missing feature or silently ignore one the model never
            # trained on, depending on direction. Both are as severe as
            # the P-0-class session_stats coverage failure above; there is
            # no meaningful degraded mode for "the model and the feature
            # pipeline disagree on what a feature vector is."
            # Cheap relative to Eager Pool — reads two already-available
            # name lists, no additional DB or model-artifact I/O beyond
            # what Step 7's Inferencer init would load anyway (moved
            # forward from there to before Step 5's cost, same rationale
            # as the other three checks in this gate).

     Any ABORT SESSION outcome stops here — Eager Pool (Step 5) never runs.

  4b. [Late-start freshness branch — R-1] Evaluate once, here, before Eager
      Pool. Session Start Gating's own wait is bounded (hard deadline
      05:00 ET — see Session Start Gating), so this branch matters only
      when a manual restart (e.g. after a Health Gate 1 abort elsewhere in
      the morning) lands past recheck_time; it also correctly no-ops for
      an Eager Pool that is merely running late on a normal start.

         if now >= config["live_mode"]["premarket_recheck_time"]:
             # Past recheck_time already — run the FULL fresh-check bundle
             # NOW, BEFORE Eager Pool, so Step 0 computes on correct data
             # and no scoped recompute (caching_calculator.md, item N) is
             # needed on this path:
             #   quotes/halt fetch
             #   + check_corporate_event_anomaly (full universe, fresh)
             #   + yfinance narrow crawl for newly-quarantined tickers
             #   + crawl_corporate_events_investing(today)   (item N)
             run_premarket_recheck_bundle()
             in_process_recheck_done_today = True   # suppresses the
                                                     # scheduled task's own
                                                     # run at recheck_time
                                                     # today — see below
         else:
             pass   # normal path: the In-Process Premarket Recheck task
                    # (below) fires at recheck_time as scheduled.

  5. [Eager Pool] Parallel session_start_compute() for all tickers:
         Using worker pool (config: live_mode.session_start_workers)

         For each ticker in all_tickers (parallelized):
             historical_bars = utils.load_ohlcv_with_history(
                 ticker, lookback_start_date, today_date, db_conn
             )
             # Stitches in predecessor-ticker bars if a rename occurred within
             # the lookback window (see utils.md get_ticker_history() /
             # load_ohlcv_with_history()) — returns None-history fast path
             # (a single direct query) for the vast majority of tickers.
             # Loaded in the same per-ticker parallel worker, alongside
             # historical_bars — both must be ready before session_start_compute():
             tick_bar_history, tier_used = utils.load_tick_bar_aggregates_with_history(
                 ticker,
                 indicators=[i for i in configured_tick_indicators
                             if i.precalculate_bars == "lookback"],
                 lookback_start_date, today_date, db_conn
             ) if any_indicator_configured_lookback else (None, {})
             # tier_used: {indicator_name: 1|2|3} — which resolution tier
             # fired per requested indicator; empty dict when no indicator is
             # "lookback"-configured. Returned (not just used internally) so
             # Step 6 can aggregate it after all workers report back — see
             # Health Gate 2.
             # Corporate-event split adjustment is NOT applied here — that
             # happens inside session_start_compute() itself, anchored to
             # today_date (see caching_calculator.md).

             calc = CachingIndicatorCalculator(config)
             calc.session_start_compute(historical_bars, tick_bar_history)
             calc.set_session_stats(
                 session_stats_bulk.get(ticker, {})
             )

             with write_lock:   # R-2 GAP-FIX 9: persist_to_db() does its
                 # own DELETE-then-INSERT sequence on writer_conn with no
                 # locking of its own (see caching_calculator.md — its
                 # docstring never mentions write_lock). 8 parallel Eager
                 # Pool workers each calling this concurrently, unlocked,
                 # is exactly the hazard write_lock exists to prevent —
                 # unlike db_write()'s callers elsewhere, this call site
                 # was missed when db_write() became the funnel. Holding
                 # the lock for this whole call (not a single execute())
                 # serializes the DELETE+INSERT pair, not just each
                 # statement — necessary since a half-applied delete from
                 # one worker interleaved with another's insert is exactly
                 # the kind of race a per-statement lock would not prevent.
                 calc.persist_to_db(db_conn, today_date, ticker)
             # R-2: written in BOTH modes now, as a crash-recovery backup —
             # each worker persists its own ticker as soon as it finishes,
             # not batched at the end. A warm restart (see "Session Restart
             # (Warm Start)") reads this via load_from_db() to skip the
             # expensive historical_bars recompute.
             if config["live_mode"]["indicator_cache_mode"] == "db":
                 pass   # RAM released; calc discarded after persist (unchanged)
             else:  # "memory" (default)
                 calculators[ticker] = calc  # retain RAM copy — authoritative
                                             # for this running session; the
                                             # backup above is read only on a
                                             # warm restart, never in normal
                                             # operation.

             worker returns (ticker, tier_used) alongside the calc/persist result

         del session_stats_bulk   # RAM released after all tickers done

  6. [Health Gate 2] Post-Eager-Pool health check — tick_bar_aggregates
     Tier-2/3 fallback rate. Depends on Step 5's tier_used results, which is
     why this cannot be folded into Health Gate 1 (Step 4):

         tier_tally = Counter()
         for ticker, tier_used in step_5_results:
             for indicator, tier in tier_used.items():
                 tier_tally[tier] += 1
         total = sum(tier_tally.values())
         fallback_rate = (tier_tally[2] + tier_tally[3]) / total if total > 0 else None

         fallback_rate is None → no lookback-configured indicator exists yet;
             check trivially passes (all 9 tick-derived indicators
             currently ship precalculate_bars: 0)
         fallback_rate > 0.20 → WARN + proceed (self-healing case, but batch
             likely failed — alert operator)

  7. Inferencer init:
         calculate_required_history() → self.required_bars
         Load model artifacts (run_id from config)

  8. Start watchdog polling loop (async)
  9. Start position manager loop (async)
```

**Bulk Load Memory Profile** (Steps 2-3, 5 — renamed from "Phase 1" now that
the lifecycle no longer labels a single step that name; scope unchanged):
```
session_stats_bulk peak:  ~15,000 tickers × ~18KB ≈ ~270MB
                          Released after Step 5 completes.

tick_bar_history (Eager Pool, only if any tick-derived indicator is
configured "lookback" — none are, by default; all 9 currently ship with
precalculate_bars: 0):
  Per-ticker cost scales with bar count (like historical_bars), not tick
  count — bar-level EAV cache, not raw tick replay. Negligible at default
  config; re-profile if any indicator is switched to "lookback".

calculators pool ("memory" mode):
  ~15,000 tickers × ~790KB (Layer1 + Layer2 + session_stats) ≈ ~11.9GB sustained
  Recommended server: 32GB+ RAM.
  For memory-constrained environments: use indicator_cache_mode = "db"
  (50~200ms per-ticker load latency on first watchdog event).
```

---

## In-Process Premarket Recheck (R-1; hosts item N's 09:20 crawl calls)

A LiveModeRunner background task, scheduled for
`live_mode.premarket_recheck_time` (default 09:20 ET) — unless
`in_process_recheck_done_today` was already set by the late-start branch
(Session Lifecycle Step 4b), in which case this task's scheduled run is
skipped for today. Replaces the former standalone `--premarket-recheck`
cron, which could not open the DB read-write while this process holds it
(DuckDB single-writer) — so the recheck, P-8's primary coverage layer,
would have failed silently every session. Running it in-process, through
this process's own writer, fixes that.

Task body (`run_premarket_recheck_bundle()` — shared with Step 4b's
late-start path; all fetches as a non-blocking background task so the 1s
watchdog and 5s position loops are not stalled):

```
1. quotes      = bulk_fetch_today_first_price(...)   # trading API, chunked
   halt_status = utils.query_halt_status(...)          # trading API, chunked
   # sequential, same as the 04:00 pass (see metadata_crawler.md)

2. check_corporate_event_anomaly(db_conn, config, quotes, halt_status)
   # full-universe fresh re-evaluation, NOT a delta — writes through
   # db_write() (the existing write_lock funnel; no new serialization
   # mechanism needed). batch_runs stage='premarket_quarantine_recheck'.

3. crawl_corporate_events_investing(today, db_conn)   # item N bulk vendor
   # investing.com, unconditional every recheck. Writes via db_write().

4. For any ticker whose quarantine_reason was NEWLY set in step 2, run a
   single-ticker yfinance narrow crawl:
       crawl_corporate_events(ticker, db_conn)
   (already a single-ticker function — no new function needed.)

5. Scoped-recompute trigger (item N): for any ticker that gained a new
   same-day corporate_events row in steps 3-4 AND whose calculator already
   exists (i.e. Eager Pool has run — always true for this scheduled path,
   since it only fires after Step 5), call that ticker's
   scoped_recompute() (caching_calculator.md) as a background task, then
   clear its quarantine_reason if set. NOT run on Step 4b's late-start
   path — there, Eager Pool hasn't executed yet, so Step 0 picks up the
   fresh corporate_events rows naturally and no scoped recompute applies.
```

All writes in steps 2-5 go through `db_write()`; no lock-handoff or
inter-process coordination (the two-cron collision class this replaces was
removed by N-1 and must not be reintroduced). Before starting, this task
checks whether the 04:00 `--premarket-open` pass is still mid-crawl
(`batch_runs` `stage='premarket_corporate_events'`, `status='running'`) and
defers rather than contend — the recheck-vs-04:00 guard is retained; the
recheck-vs-LiveModeRunner-writer contention (the reason this task exists at
all) is dissolved by sharing this process's own `db_write()` funnel.

Failure handling: if the fetches fail, log + write the recheck `batch_runs`
row `status='failed'`; no freeze. The recheck is a defense layer, not
session-critical — `is_tradable()` keeps using 04:00-pass state. Same
degraded philosophy as Health Gate 1c.

Manual-run caveat: `--premarket-recheck` (see metadata_crawler.md) survives
only as a manual/debug flag; run against a live session it will fail to
open RW. Manual use is for non-session contexts.

---

## Watchdog Polling Loop

Polls external watchdog service at `poll_interval_seconds` (default: 1s).
The watchdog service is a separate external system, not part of this
codebase — it applies its own entry-point-condition matching against the
live feed and pushes a ticker's name into the candidate list the instant
its own criteria are met. "Candidates" therefore means "tickers the
watchdog service just flagged," not "all actively-tracked tickers" —
this is why bar/tick fetching below (Step 2) only concerns itself with
that ticker being flagged again, not with continuously polling every
ticker in the watchlist.

**Loop termination (R-9).** This loop runs only while entries are
structurally possible. At the top of every cycle, before Step 1:
```
if now >= config["execution"]["session_close_exit_time"]:
    break        # exit the loop; the process itself lives on until
                 # Session Shutdown's exit trigger fires
```
Breaking between cycles rather than mid-cycle means no partial fetch or
half-applied `on_bar_close()` replay. Past that time no entry can be
submitted anyway (after-market bars are excluded from entry detection in
every mode, and `max_entry_hour` has long since cut off), so nothing is
lost — but the loop is stopped explicitly rather than left polling an
external service whose after-hours behaviour is not part of this system's
contract and whose responses would be exercising the candidate path for no
possible benefit. Close-out continues on the Position Manager Loop, which
is independent of this one and does its own bar/tick fetching.

A warm restart that lands after `session_close_exit_time` still starts this
loop (Session Lifecycle Step 8) and it self-terminates on its first cycle
through the same guard — no separate branch is needed.

Two accepted consequences: Bar-Close Authority and its Bar-Arrival Latency
sampling (finding 26) stop collecting at `session_close_exit_time`, and
watchdog-triggered Feed Outage Recovery stops evaluating there too. The
first is fine because the day's curve is finalised by the final
`bar_latency_daily` flush in Session Shutdown; the second because close-out
runs off wall-clock triggers and the WS/REST tick stream, not the bar
channel Feed Outage Recovery protects, and every remaining position is
already exiting.

```
loop every poll_interval_seconds:

  1. Query watchdog service → list of ticker candidates for this bar
     (tickers the watchdog service's own entry-condition matching just
     flagged — see note above)

  2. For each ticker in candidates:
     a. Fetch current bars from trading API (1min chart up to now —
        the API always returns the full range since the caller's last
        query, not just the latest bar)
     b. Fetch current ticks from trading API (10-tick up to now)
     c. Update intraday state in memory / DuckDB

  3. If one or more new 1min bars completed for ticker since last check:
         for each newly-arrived bar, in chronological order:
             calculators[ticker].on_bar_close(bar, ticks_for_bar)
             if 093000 bar just closed:
                 calculators[ticker].on_regular_session_open(bars_including_930)
         # Looping here (rather than assuming exactly one new bar) is what
         # lets a ticker that drops out of candidacy for a while and later
         # returns catch up correctly in the same step used for first entry
         # — see "Watchlist Append" below. on_bar_close() itself is O(1)
         # per call for every CONTINUOUS indicator (see caching_calculator.md),
         # so looping over several missed bars here is cheap regardless of
         # how many accumulated.

  4. If ticker appears in candidates for the first time this session,
     or is returning after one or more missed poll cycles
     → see "Watchlist Append" below (Step 3's loop already replays any
     bars missed in between; this step only concerns first-time calculator
     setup when indicator_cache_mode="db")

  5. For each ticker with completed bar and active calc:
     a. Re-verify entry point candidate (detect())
     b. If confirmed: Inferencer.infer(
            bars, ticks, meta_bulk[ticker], entry,
            session_stats=calculators[ticker]._session_stats
        )
     c. If signal (up5 / up3):
        Fire an async, fire-and-forget REST bid/ask query for this ticker
        (5-level book) → `bid_ask_snapshots` (`source='signal_time_rest'`).
        Every confirmed signal, gate outcome and shadow/real status both
        irrelevant — a candidate the gates go on to block is exactly the
        kind of observation the execution-gate path of the bid/ask open
        item needs. Not on the entry-submission critical path: this call
        is not awaited before gate evaluation or order submission below
        proceed.
        0.  Entry gates (R-5) — evaluated in this order, before every step
            below. Enforced identically in shadow mode; unlike R-4's
            breaker (evaluate-but-don't-enforce in shadow), a shadow run
            that ignored these caps would record entries live could not
            have taken, overstating exactly what shadow exists to measure.
            - freeze_reasons: skip if any active reason's scope includes
              entry_submission (see Feed Outage Recovery). Record
              gate_result='freeze', or 'breaker' when the active reason is
              'breaker_trip' — see Circuit Breaker below.
            - Ticker cap: COUNT(DISTINCT ticker) over today's
              live_positions rows with status IN ('pending','partial_open',
              'open','halted','exiting'). If this ticker is not already
              among them AND that count >= execution.max_tickers → skip.
            - Per-ticker cap: this ticker's own row count over the same
              statuses >= execution.max_positions_per_ticker → skip.
            - can_enter(ticker, current_hour, last_entry_hour,
              execution.entry_cooldown_minutes) → skip if False.
            Both caps read live_positions DB status, never the
            in_flight_orders runtime cache (R-2: the row is the SSoT);
            pending rows count as reserved capacity. last_entry_hour is the
            SUBMISSION time of this ticker's most recent entry attempt this
            session, including attempts that ended 'canceled' or
            'entry_rejected' — the same definition Warm Restart's cooldown
            restore rebuilds from the DB.
            # Config validation: execution.entry_cooldown_minutes * 60 >=
            # execution.cancel_after_seconds must hold. The submission-time
            # cooldown clock is what stops a re-flagged ticker from
            # double-submitting while its own order is still in flight, but
            # that protection follows from the two values' relative size —
            # it is not structural, and a short cooldown configured against
            # a long cancel window removes it.
            Whichever gate stops the candidate, write its reason to
            inference_log.gate_result on THIS candidate's own
            event='signal_fired' row — 'freeze' | 'cap_tickers' |
            'cap_per_ticker' | 'cooldown', and likewise 'not_tradable' and
            'bar_integrity' (step i), 'sizing_zero' / 'funds' (step ii)
            below. A candidate
            reaching submission in step iii records 'submitted'. If step
            ii's check_funds_available() call raises (the one network call
            in this sequence — timeout, connection error) rather than
            returning a verdict, record 'error': the candidate otherwise
            vanishes from every count with no record it was ever
            evaluated. Every signal_fired row therefore ends with a
            terminal state, so non-'submitted' rows are the complete
            record of candidates lost at the gates — they get no
            trade_log row at all (a broker rejection is different: that IS
            'submitted', and shows up as trade_log
            exit_reason='entry_rejected'). This row is written as a SINGLE
            INSERT the instant the gate sequence concludes, whichever way —
            never as an UPDATE to a row Inferencer wrote earlier (see
            Constraints for why this one event is LiveModeRunner's to
            write). See db_schema.md's inference_log for the value list and
            why this is a column rather than a new event. health_report.md's
            finding 19 reads these rows directly, so no in-session tally has
            to be threaded through to it.
        i.  is_tradable(ticker, db_conn) — execution_common.md gate.
            False (ticker_cik_map.status = 'suspended') → skip, no order.
            Then, in the same step because it is the other per-ticker
            eligibility verdict: skip if this ticker is in the
            bar_integrity exclusion set (see Bar-Close Authority's
            Bar-Arrival Latency Measurement) → gate_result='bar_integrity'.
            A runtime set membership test, so it costs nothing to place
            here rather than earlier; unlike the freeze_reasons check in
            step 0, it is per-ticker rather than session-wide.
        ii. quantity = execution_common.compute_position_size(
                balance=session_start_cash, fill_price=entry["p_entry"],
                t_bar_volume=..., ticker_notional=..., total_notional=...,
                position_size_cash_pct=config["execution"]["position_size_cash_pct"],
                position_size_vol_pct=config["execution"]["position_size_vol_pct"],
                per_ticker_share_cap_pct=config["execution"]["per_ticker_share_cap_pct"],
                exposure_cap_pct=config["execution"]["exposure_cap_pct"],
            )
            # p_entry, not a post-slippage price — same ordering rationale
            # as 09_backtest_engine.md's Entry Slippage Model (sizing must
            # precede simulate_entry_fill(), which takes quantity as input)
            proceed, quantity = execution_common.check_funds_available(
                quantity, entry["p_entry"], available_cash,
                use_all_cash=config["execution"]["use_all_cash"],
            )
            available_cash queried fresh from trading API immediately
            before this call (see execution_common.md's Constraints)
            if not proceed: skip, no order
        iii. limit_price = None if config["execution"]["entry_order_type"] == "market" \
                 else (entry["p_entry"] * (1 + config["execution"]["entry_gap_value"])
                       if config["execution"]["entry_gap_type"] == "percentage"
                       else entry["p_entry"] + config["execution"]["entry_gap_value"])
             if config["live_mode"]["shadow_mode"]:
                 weighted_avg_fill_price, filled, unfilled, status =
                     execution_common.simulate_entry_fill(
                         ticks_entry=..., ohlcv_entry=..., quantity=quantity,
                         fill_bundle_idx=..., p_entry=entry["p_entry"],
                         buy_rate=config["execution"]["buy_rate"],
                         halts_df=...,
                         cancel_after_seconds=config["execution"]["cancel_after_seconds"],
                         limit_price=limit_price,
                     )
                 log hypothetical entry to trade_log (is_shadow=TRUE, quantity=filled)
                 # entry_order_type is simulated the same way in shadow as
                 # in backtest — see execution_common.md's price-gate logic
                 if filled > 0:
                     subscribe to this ticker's realtime trade stream NOW
                         (see Exit Architecture's "Primary — WebSocket tick
                         stream") — same "submission, not fill, is the
                         trigger" principle as the real branch below, but
                         simplified: simulate_entry_fill() already resolved
                         filled/unfilled synchronously, so there is no
                         async pending window to subscribe ahead of, and no
                         separate zero-fill unsubscribe path is needed —
                         a hypothetical position simply is not opened, and
                         no subscription is ever made, when filled == 0.
             else:
                 order_id = submit order via trading API: quantity=`quantity`,
                     order_type=config["execution"]["entry_order_type"],
                     limit_price=limit_price (omitted/ignored for "market")
                 # R-2: write the pending row FIRST (SSoT — see
                 # db_schema.md's live_positions), for BOTH order types —
                 # closes the submit-before-fill-response crash window even
                 # for market orders, and gives a pending limit order a
                 # durable record a crash-recovery reconcile can match by
                 # order_id (see "Session Restart (Warm Start)" below).
                 db_write: INSERT live_positions (run_id, ticker, date,
                     entry_bar, order_id, limit_price, submitted_at=now,
                     signal, status='pending', fill_price=NULL,
                     fill_second=NULL, quantity=NULL)
                 # R-7: the order API's response confirms ACCEPTANCE of the
                 # submission, not a fill — fills arrive on the separate
                 # fill channel (WS account stream / REST order query). The
                 # former "market: real fill returned synchronously" branch
                 # was wrong for this vendor and is removed. The only
                 # rejection knowable at this instant is structural and
                 # price-independent (account restriction, ticker not
                 # permitted, malformed order); a market order's
                 # insufficient-funds-at-actual-price rejection and every
                 # limit rejection surface later on the fill channel.
                 subscribe to this ticker's realtime trade stream NOW —
                     acceptance of the submission, not a fill, is the
                     trigger (see Exit Architecture's "Primary — WebSocket
                     tick stream"; moved earlier than the fill it used to
                     wait for, so a position is tracked for the entire
                     window it carries risk, not just from its first fill
                     onward). Skipped if already subscribed (a second
                     position opening on an already-tracked ticker shares
                     the existing subscription, unchanged from before).
                 if submission rejected structurally:
                     no order_id exists and no live_positions row is
                     written — nothing was accepted. Log trade_log row:
                     exit_reason='entry_rejected', quantity=0,
                     fill_price=p_entry, exit_bar=entry_bar, and
                     reject_reason = the broker's own refusal text, stored
                     verbatim (see db_schema.md's never-opened family).
                     Counts as a cooldown attempt via step 5c.0's
                     last_entry_hour; excluded from fit_execution_params()
                     (see shadow_retraining.md).
                 else:
                     in_flight_orders[order_id] = {ticker, side: 'entry',
                         submitted_at: now, limit_price,
                         requested_quantity: quantity}
                     # runtime cache over the live_positions row above —
                     # BOTH order types, tracked to fill/reject/cancel by
                     # Position Manager Loop's "In-flight order tracking"
```

---

## Bar-Close Authority

Judges, once per Watchdog Polling Loop cycle, whether each ticker in THAT
cycle's `candidates` (not the full tradable universe, and not the
cumulative set of tickers ever flagged this session — see the loop's own
note on what `candidates` means) has a bar for the most recently
fully-elapsed minute, at whole-second precision. Scoped to `candidates`
deliberately: a bar is only ever fetched (Step 2a) for a ticker the
watchdog just flagged, so a ticker that simply has not been re-flagged
recently has nothing to judge — checking against the full active
watchlist would misread ordinary watchdog silence as a missed deadline.

```
On every poll_interval_seconds cycle, after Step 2a's bar fetch:

  expected_minute = floor(now to the minute) - 1 minute   # the most
      # recently fully-elapsed minute; NOT the current, still-forming one
  deadline = (end of expected_minute) + bar_close_grace_seconds

  if now < deadline:
      skip this check this cycle — expected_minute's own grace window has
      not elapsed yet; checking earlier would just re-detect the same
      ordinary poll-to-poll lag every single cycle

  else:
      missed = [ticker for ticker in candidates
                if last_bar_hour[ticker] < expected_minute]
      # last_bar_hour[ticker]: this ticker's own most recent bar hour as
      # actually returned by Step 2a, tracked per ticker across cycles —
      # read directly off the fetch response, not derived from
      # calculators[ticker], since that already absorbs Step 3's replay
      # and would hide exactly the lag being measured here
```

`bar_close_grace_seconds` (config, seed value — see
api_contract_checklist.md's T-13: how promptly the vendor typically has a
minute's bar ready is itself unverified) is why 1.0s of clock skew (see
Session Start Probes' `clock_check`) is the boundary at which a clock
fault starts impersonating a feed outage: the deadline above is wall-clock
derived, so a skewed local clock shifts every ticker's `expected_minute`
identically and simultaneously, not just one.

Individually, one ticker in `missed` is routine — this is exactly the case
Feed Outage Recovery (below) leaves un-frozen, caught up whenever that
ticker's next bar arrives via Step 3's ordinary multi-bar replay.
`missed` becomes a candidate systemic signal only in aggregate — see Feed
Outage Recovery's trigger condition 2, next.

### Bar-Arrival Latency Measurement (T-13)

`bar_close_grace_seconds` is a seed value because nobody has measured what
it is guarding against. This measurement is what turns it into an observed
quantity, satisfying api_contract_checklist.md's T-13 without a separate
verification exercise. It measures only bars that DO arrive; `missed` above
covers the ones that do not, and the two are reported together (finding 26)
because neither is interpretable alone.

Measured at Step 2a, where `last_bar_hour[ticker]` is updated — NOT in Step
3 — for the same reason `last_bar_hour` is read off the fetch response
rather than from `calculators[ticker]`: Step 3 absorbs multi-bar replay and
would hide the very lag being measured. Two further properties fall out of
that placement: the loop's hottest path (Step 3's `on_bar_close()` replay)
is untouched, and at most one sample per ticker per cycle is structurally
guaranteed, so a single slow bar cannot be counted repeatedly.

```
At Step 2a, for each candidate whose fetch returned a newer bar than
last_bar_hour[ticker] held:

  d     = now - (newest bar's hour + 60s)     # whole seconds
  delta = now - last_fetch_time[ticker]       # absent on a first poll
  e     = min(delta, d)                       # this sample's error bound

  if d < 0:
      negative sample — see "Premise violation" below. Not a latency
      sample; e is meaningless when d is.
  elif e <= live_mode.bar_latency_max_error_seconds:
      admit: bucket floor(d), class 'poll_continuous' if
      e <= poll_interval_seconds else 'wide_error'
  else:
      reject: count as excluded_no_prior_fetch (no delta) or
      excluded_both_wide (delta and d both large)

  last_fetch_time[ticker] = now      # every successful fetch, admitted
                                     # sample or not
```

**Why `e`, and why it needs the prior fetch.** `d` overstates the vendor's
true publish latency by however long the bar sat unobserved before we asked.
Knowing the ticker was fetched `delta` ago bounds that overstatement at
`delta`; on a first poll only the bar's own close bounds it, at `d` itself.
Taking the smaller of the two is what keeps a sample honest, and it is why
`last_fetch_time` exists at all: without it a bar that had been ready for 40
seconds before the first poll reached it would be recorded as 40 seconds of
vendor latency. The bound matters because the error is bounded at 60s
otherwise — a fetch returns only the newest bar, so the overstatement does
not grow with how long the ticker was away, but 60s dwarfs the 5s value
being calibrated.

Deliberately NOT filtered on bar-timestamp continuity, which was the obvious
alternative: a thin ticker that gets no zero-volume bar for a minute (see
Feed Outage Recovery's trigger condition 2 on exactly this uncertainty)
produces a bar whose timestamp skipped a minute but which arrived perfectly
on time. Rejecting those would discard valid samples from precisely the
thinnest tickers — the ones a vendor is slowest on — biasing the curve
optimistic. Such samples are admitted and separately counted
(`bar_gap_samples`), which is also what distinguishes structural
non-publication (large gap, small `d`) from real vendor lag (large gap,
large `d`).

`last_fetch_time` is runtime-only, same convention as `last_halt_state`: a
warm restart costs at most one skipped sample per ticker, on its first
post-restart fetch. Every exclusion case — session first appearance,
candidacy re-entry, post-outage recovery, an individual failed fetch — skips
exactly one cycle for that ticker, not a window.

Counters accumulate in memory as the UN-FLUSHED DELTA ONLY and are flushed
additively into `bar_latency_daily` (db_schema.md, which carries the merge
rule) through `db_write()`, once per minute at an in-minute offset rather
than on the minute boundary — the boundary is where bar arrival, Step 3's
replay, and `infer()` all land. The delta is zeroed after each flush,
including a forced one, so the periodic timer is reset by a forced flush
too; otherwise the next scheduled flush writes an empty delta. Runs
unconditionally in shadow mode: the bar/price channel is identical in both
modes, and this measures that channel, not the order path.

**Premise violation (`d < 0`).** A negative sample means a bar exists for a
minute that has not finished, which contradicts the `expected_minute`
reasoning this whole section rests on. Two causes, and the `|d|`
distribution in `bar_latency_daily` separates them: values near 60s mean the
vendor timestamps bars by CLOSE while this spec assumes OPEN, in which case
every `missed` verdict and every Feed Outage condition-2 evaluation is off by
one bar; small scattered values mean premature publication of a still-forming
bar, or a clock fault. Either way it is a design error, not a data blip, so
it is recorded per event via `utils.record_health_event()` and reported as
`health_report.md` finding 27, event-driven at abort severity rather than
waiting for session end.

On detection, in this order:
```
1. increment the negative counter (in-memory delta)
2. add ticker to the bar_integrity exclusion set
3. if len(exclusion set) >= live_mode.bar_integrity_freeze_ticker_count:
       add 'bar_integrity' to freeze_reasons, scope entry_submission
4. forced flush of bar_latency_daily (synchronous, via db_write)
5. record_health_event() + health_report.py invocation, subset-scoped to
   finding 27 (see health_report.md's Invocation Contract)
```
State changes precede the flush so the alert cannot report an exclusion set
missing the very ticker that triggered it. The flush happens whether or not
the alert itself is suppressed — the table is state, suppression is
transport. The health_report call returns as soon as it has written its log;
delivery is its own background concern (see health_report.md), so this loop
is never blocked on a socket.

**Why the exclusion is per-ticker and entry-only.** A prematurely-published
bar is fed to `on_bar_close()`, whose CONTINUOUS-indicator updates are
incremental and therefore irreversible for the session; that corrupted state
reaches new entries through `infer()`. It does NOT reach existing positions:
tp/sl runs off the WS tick stream (Exit Architecture), not the bar channel,
so open positions keep being managed and exited normally. Corruption also
cannot spread between tickers, since every affected structure is per-ticker.
Blocking new entries for that ticker alone is therefore sufficient, and the
session continues.

Rebuilding the corrupted calculator via `scoped_recompute()` (the mechanism
corporate-event handling uses above) is deliberately NOT done: with entries
for that ticker already blocked, a rebuild buys nothing and adds a recovery
path whose own correctness would need establishing.

The promotion at step 3 exists because a timestamp-convention mismatch
affects every ticker, so per-ticker exclusion would trip the whole watchlist
one ticker at a time while leaving tickers not yet flagged unprotected. The
promoted freeze blocks entry submission only and does not clear within the
session — there is no way to verify recovery from corrupted incremental
state, the same reason a breaker trip does not auto-clear. The exclusion set
itself is runtime-only; after a warm restart a ticker is re-excluded on its
next negative sample.

---

## Feed Outage Recovery

Distinct from the routine per-ticker catch-up already built into the
Watchdog Polling Loop (Step 3's multi-bar replay) and Bar-Close Authority —
those handle an individual ticker going quiet for a while, gracefully and
without any freeze. This section covers the qualitatively different case:
LiveModeRunner's own connectivity to the trading API/watchdog service is
lost.

**Trigger** (either condition):
- An explicit connection/API-level failure (exception, timeout, non-200
  response) from the trading API or watchdog service itself, or
- `len(candidates) >= live_mode.min_watchlist_size` AND more than 50% of
  THIS cycle's `candidates` are in Bar-Close Authority's `missed` set —
  a systemic signal even without a hard connection error. The size floor
  exists because a bare percentage is only a meaningful systemic signal
  once the sample is large enough that a few individually-quiet tickers
  coinciding by chance cannot clear it on their own; below the floor, a
  cycle's candidate count is too small to support the claim either way,
  and this condition simply does not evaluate (condition 1 above remains
  the only detector for that cycle). Both `min_watchlist_size` and
  `bar_close_grace_seconds` are seed values, deliberately set conservative
  (harder to trigger) until measured — the vendor's actual quiet-minute
  behavior (whether a thin ticker with zero trades gets a zero-volume bar
  at all, or no bar) bears directly on how large a "normal" miss rate can
  be, and this condition is set with that in mind.

**Recovery procedure**, on detecting the trigger:
```
1. Freeze: add 'feed_outage' to freeze_reasons.

   freeze_reasons is a SET of active reasons, each carrying its own scope
   — not a single session-wide boolean. A boolean cannot represent two
   overlapping causes, and clearing it on recovery from one would silently
   release the others (e.g. a feed outage during a warm restart's cache
   reload would unfreeze entries the restart still needs held):
     'feed_outage'    → blocks entry_submission AND exit_submission
     'restart_warmup' → blocks entry_submission only (Warm Restart step 4)
     'breaker_trip'   → blocks entry_submission only (Circuit Breaker, R-4)
   A gate blocks whenever ANY active reason covers its scope, so releasing
   one reason leaves the others in force. Watchdog Polling Loop step 5c.0
   checks entry_submission; Position Manager Loop step 3 checks
   exit_submission. Exit *evaluation* is never frozen — it keeps running
   and accumulating bars/ticks; only order *submission* is held.

2. Reconnect: retry the trading API/watchdog service connection
   (backoff policy TBD — not specified here).

3. Catch up: once reconnected, this does NOT require a separate vendor
   endpoint — the trading API's existing "always returns full range since
   last successful query" behavior (already relied on elsewhere — see
   Watchdog Polling Loop Step 2a, Position Manager Loop Step 1) means the
   very next ordinary bar-fetch call for each active-watchlist ticker
   naturally returns everything missed during the outage. The EXISTING
   Step 3 multi-bar replay loop (calculators[ticker].on_bar_close() called
   once per missed bar, in order) handles the catch-up with no new
   mechanism — the outage is, from the calculator's point of view, just an
   unusually long version of the same gap Step 3 already tolerates.
   (A gap long enough to exceed the trading API's own retention/buffer is
   a true, unrecoverable data gap — not engineered around here; falls
   through to whatever the vendor's own missing-data convention is.)

4. Reconcile: run the Broker Reconcile shared procedure (R-3 — see
   "Broker Reconcile (shared procedure)" below). The feed-outage-specific
   case (broker shows a position closed that LiveModeRunner still tracks
   as open → an exit order placed just before the outage evidently filled;
   adopt the broker's fill as authoritative, not simulated/estimated) is
   part of that shared procedure. `feed_gap_exit` (step 5 below) still
   applies only to positions that remained open through the outage.

5. Re-evaluate exits: for each position that remained open through the
   outage, run exit evaluation (track_price_breach() over the now-caught-up
   bars/ticks) once before unfreezing. Any position whose exit condition
   would have triggered *during* the gap: exit at market immediately (not
   at the historical breach point — that price is stale by now), logged
   with `exit_reason='feed_gap_exit'` (see db_schema.md's trade_log) so
   this trade is excluded from ordinary strategy PnL attribution — it
   reflects outage handling, not strategy performance.

6. Unfreeze: remove 'feed_outage' from freeze_reasons — only this reason.
   If another reason is still active (e.g. 'restart_warmup'), its own
   scope stays in force and normal operation resumes only for what it
   does not cover.
```

## Circuit Breaker (R-4)

Protects against a failure mode stop-losses structurally cannot catch. A
stop-loss handles one trade going the wrong way, independently each time. It
does nothing about the same wrongness repeating: at
`position_size_cash_pct: 0.05` and `stop_loss_pct: 0.03`, each stopped trade
costs ~0.15% of the account, so a full book of 20 stopping out is ~3% — with
every stop-loss behaving exactly as designed. What the breaker targets is a
broken model or feature pipeline emitting bad entries continuously.

**Three thresholds** (execution_common.md; all default 0 = no limit):
`intraday_loss_limit_pct`, `consecutive_loss_limit`, `entries_per_hour_limit`.
They are deliberately staggered along the failure timeline — entry rate spikes
first, before any loss exists; then losing exits run; realised loss accumulates
last.

**On trip:** add `'breaker_trip'` to freeze_reasons, scope `entry_submission`
only, and db_write `live_session_state.breaker_tripped_at = now` (see
db_schema.md — this is what lets the trip survive a crash; see Warm Restart
below). Open positions keep running their own tp/sl — force-flatten is
rejected (market-impact risk for no gain). Does not auto-clear within the
session.

**State is updated on the events themselves, not polled.** All three quantities
change only at moments the system already handles: realised loss and the
consecutive counter at position close, entries-per-hour at submission — both
restricted to `is_shadow = FALSE` rows for a live session's own counters (a
shadow evaluation, below, tracks its own is_shadow=TRUE counters separately
so the two never mix). So the entry gate reads a value that is current by
construction. Evaluating on the Position Manager Loop's cadence and having
the 1s gate consume the last verdict was considered and rejected — it admits
up to five gate cycles between the condition becoming true and being
enforced. The one delay that remains is inherent: a close is not known until
the loop observes it, which no evaluation cadence changes.

**Consecutive counter** counts consecutive LOSING exits (`pnl_pct < 0`) and is
reset only by a profitable exit — see execution_common.md for why the
stop-loss-only reading leaves a hole. Operational-family and never-opened rows
are skipped entirely, incrementing nothing and resetting nothing. Where several
positions close in one cycle, order by `exit_bar`, then ticker alphabetically —
the same tiebreak backtest already uses.

**Shadow mode: evaluate, do not enforce.** A real trip would truncate exactly
the bad-day data shadow exists to observe. Because nothing is enforced,
`gate_result` still records `'submitted'`, so a would-have-tripped event CANNOT
be read from `gate_result` — it is reported as its own health_report finding.
Note this is the opposite policy from the position caps, which ARE enforced in
shadow: caps shape what live would have done, the breaker only reacts to it.

**On trip, health_report.py is invoked immediately** — a third trigger beside
session start and session end — at `abort` severity, so it reaches even
`alert_level: "abort_only"`. No new alerting infrastructure: the Discord and
email channels already exist and were simply never wired to a mid-session
event.

## Broker Reconcile (shared procedure) — R-3

One implementation, three call sites: (a) every Session Lifecycle start,
cold start included (Session Lifecycle Step 1d), (b) Feed Outage Recovery
step 4, (c) Warm Restart step 1 (R-2, below). Compares the trading API's
view (open orders + open positions) against `live_positions` rows.

**Orders** (broker open entry orders):
  - Match to `live_positions` rows with `status='pending'` by `order_id`.
  - Cancel (unknown staleness — conservative); the matched row transitions
    to `'canceled'` via the single canceled-transition point (see
    "Pending limit-entry tracking," R-2) — the same idempotent path an
    ordinary cancel-after-timeout uses, so re-running this step (e.g. a
    re-crash mid-recovery) is safe and cannot double-log.
  - A broker order with no matching pending row → record_health_event(
    finding_name='unknown_broker_order_or_position', detail={call_site,
    kind: 'order', order_id, ticker}) — see health_report.md finding 12.

**Positions** (broker open positions):
  - Match to `live_positions` rows (`status` `'open'`|`'halted'`).
  - Matched row's `date` is a PRIOR trading day → Unified Overnight Policy
    (below).
  - Matched row's `date` is TODAY (only possible at Feed Outage / Warm
    Restart, never at cold start) → keep managing normally (Feed Outage)
    or adopt exit-only (Warm Restart, per R-2).
  - Broker position with NO matching row → adopt conservatively; entry
    time is unknown, so treat as overnight (immediate liquidation, below).
    Also record_health_event(finding_name=
    'unknown_broker_order_or_position', detail={call_site, kind:
    'position', order_id: NULL, ticker}).
  - `live_positions` row `status='open'` with NO broker position →
    **reconcile_ghost**: transition the row to `'closed'`, trade_log
    `exit_reason='reconcile_ghost'`, `quantity=0`, PnL-excluded — there was
    no real position, so logging it as `stop_loss` etc. would fabricate a
    trade that never happened.

**Cold-start note:** at cold start nothing has opened today, so ANY broker
position found is by definition an overnight orphan — it always routes to
the Unified Overnight Policy below.

**Health-event writes (R-9).** Both branches above call
`utils.record_health_event()` with `write_fn=db_write` — never a raw
`db_conn`, which would bypass `write_lock` (see utils.md). `call_site` is
one of `'session_start'` | `'warm_restart'` | `'feed_outage'`, so one
implementation serving three call sites stays distinguishable in the
table. Because this procedure is idempotent and re-runnable after a
mid-recovery re-crash, a repeat run legitimately re-records the same
condition; the events are timestamped occurrences, not a deduplicated set.
Per db_schema.md's standing rule, a failure of either write is swallowed
and must not abort the reconcile.

## Unified Overnight Policy — R-3

Any adopted position dated to a PRIOR trading day (halt-through-close,
unfilled/rejected EOD exit, crash orphan, or an unmatched broker position
of unknown date) is liquidated at market as soon as the ticker is
tradable, `exit_reason='overnight_exit'`, PnL-excluded. `exit_date` is the
date of that liquidation, NOT the row's `date` (the entry date) — these are
by definition different for this label, and may differ by more than one day
across a weekend, holiday, or multi-day halt. See db_schema.md's
`trade_log.exit_date`.

**Mechanism** — no special executor needed: the adopted position enters
the ordinary Position Manager Loop, where `execution.max_hold_bars` was
exceeded long ago, so the very first evaluation fires an immediate market
exit; only the label is special-cased for attribution.

**Label split vs. `restart_gap_exit` (R-2)** — same underlying mechanism
(carried position, max-hold exceeded, immediate liquidation, PnL-excluded),
distinguished by DATE for diagnosis only:
  - carried within the SAME trading day (a crash gap) → `restart_gap_exit`
  - carried ACROSS a trading day (a prior-day date)   → `overnight_exit`
Both are PnL-excluded, so strategy attribution is unaffected by which
label a given carried position gets — the split exists purely so the two
different root causes (same-day crash vs. multi-day carry) stay
distinguishable in `health_report.md` and `trade_log`.

---

## Session Restart (Warm Start) — R-2

Detected at startup: today's `live_session_start` row is `status='running'`,
there is no `live_session_end` row, and now is within session hours on a
trading day. (Else: ordinary cold start.) Distinct from Feed Outage
Recovery above — that handles connectivity loss within a still-running
process; this handles the process itself having died and restarted.

The procedure is idempotent from step 1 — a crash during recovery leaves it
safely re-runnable. Throughout, the `live_session_start` marker stays
`'running'` (only a clean resume/shutdown flips it — see Session Shutdown),
so a re-crash during recovery re-enters warm restart on the same signature.

```
1. Broker Reconcile (shared procedure — see R-3's "Broker Reconcile" for
   the fully general form; the behaviors relevant at this call site):
     - Open entry orders (broker): match to live_positions rows with
       status='pending' by order_id. Cancel all pending orders (unknown
       staleness — conservative); each matched row transitions to
       'canceled' via the single canceled-transition point (see Pending
       limit-entry tracking) — same idempotent path an ordinary expiry
       uses, so re-running this step is safe. A broker order with no
       matching pending row -> "unknown broker order" health_report finding.
     - Open positions (broker): match to live_positions rows
       (status 'open'|'halted'). A broker position with no matching row ->
       adopt conservatively and liquidate immediately (entry time
       unknown). A row with status='open' but no broker position ->
       reconcile_ghost (see R-3). Prior-trading-day positions -> overnight
       policy (R-3).

2. Restore session_start_cash from live_session_state (NOT re-queried —
   see Session Lifecycle Step 1c).

3. Start Position Manager exit-only immediately — exits are
   indicator-independent (WS/REST tick driven; no Eager-Pool state needed
   to manage an exit). For each position with an active subscription
   pre-crash — status IN ('pending','partial_open','open','halted'); all
   four now carry a live tick subscription, since Exit Architecture's
   subscribe point moved to order submission rather than first fill, not
   just the fully-open pair this step originally covered:
     a. WS re-subscribe FIRST (see Exit Architecture), THEN
     b. REST gap-fill for the crash gap, deduped against the WS buffer by
        the global tick-dedup rule (see utils.md's stitch_ticks()). Doing
        (a) before (b) guarantees no realtime tick is lost in the seam.
     c. status IN ('open','halted'): apply the 2-print guard over gap-fill
        + live ticks; if a tp/sl breach is found within the gap, liquidate
        at current price, exit_reason='restart_gap_exit' (see
        db_schema.md). If the position's elapsed hold already exceeds
        config["execution"]["max_hold_bars"], skip retro-detection and
        liquidate immediately (still 'restart_gap_exit').
        status IN ('pending','partial_open'): no `fill_price` yet, so no
        breach evaluation — same as the ordinary (non-restart) guard in
        Exit Architecture. Gap-filled ticks are retained by the dedup
        layer only; this order's own fill state is resolved by Step 1's
        Broker Reconcile, independently of this tick catch-up.

4. Entries stay frozen — add 'restart_warmup' to freeze_reasons
   (entry_submission scope only, so step 3's exit-only management above
   keeps submitting exits) — while the indicator cache reloads: if today's Eager-Pool backup exists in
   indicator_cache, restore each ticker via load_from_db() (skipping the
   historical_bars recompute) then replay today's bars via on_bar_close();
   if no backup exists (Eager Pool never ran this process), fall back to
   full session_start_compute(). Health Gate 1 re-runs unchanged (its
   premarket inputs are unchanged since morning). indicator_cache is NOT
   purged on this path (see db_schema.md).
   Idempotency of this step specifically (re-crash during recovery):
   load_from_db()'s result is the fixed Eager-Pool-only snapshot; the
   on_bar_close() replay that brings it current happens entirely in
   memory and is never itself written back to indicator_cache. So a
   re-crash mid-replay loses only that in-memory progress, not the
   backup it started from — the next restart attempt reloads the SAME
   snapshot and replays from scratch again, reaching the same end state.
   This makes step 4 all-or-nothing per ticker with no partial-state
   corruption possible, and needs no transactional handling beyond that.

5. On cache reload complete: clear 'restart_warmup'; entries resume.

6. Cooldown restore: per ticker, last entry-attempt time = max over
   today's live_positions rows in {pending, open, canceled, closed} (read
   from DB status — the SSoT — not any in-memory cache) and trade_log rows
   with exit_reason IN ('entry_canceled', 'entry_rejected'). The
   pending_entries in-memory dict is rebuilt from live_positions rows with
   status='pending'.

7. Circuit breaker restore (R-4): recompute all three counters from
   today's `trade_log`, `is_shadow = FALSE` only (a shadow evaluation's
   counters, if this process also ran shadow, are recomputed the same way
   but restricted to `is_shadow = TRUE` and kept separate — see Circuit
   Breaker). Realised loss and the consecutive-loss counter are exact:
   both are cumulative or determined by exit order, neither decays with
   time. entries_per_hour is a ROLLING window and DOES decay during
   downtime, so recomputing it alone can under-report a real trip after a
   long outage. Recomputation therefore is NOT sufficient by itself:
     if live_session_state.breaker_tripped_at IS NOT NULL:
         restore 'breaker_trip' to freeze_reasons regardless of what the
         three recomputed counters show — this is the only path that
         upholds "no auto-clear within a session" (Circuit Breaker) across
         a crash. The recomputed counters still matter: they are what
         health_report.md reports going forward, and they are what a
         second, independent trip condition (post-restart) would be
         evaluated against.
```

## Watchlist Append

Called when a ticker appears in watchdog candidates for the first time this
session, **or** is returning after one or more missed poll cycles (i.e. it
was a candidate before, dropped out, and the watchdog service is flagging
it again). Both cases use the same procedure — a ticker's absence from
candidates for any stretch of time is not tracked or treated specially; the
calculator's bar history simply catches up to however much was missed,
same as it does for the ordinary first-time case:

```
When ticker X appears in candidates (first time this session, or again
after any gap):

  if ticker_X not in calculators:   # only possible when indicator_cache_mode == "db"
                                     # and this is genuinely the first appearance
      # Restore from DB, reload session_stats
      calc_X = CachingIndicatorCalculator(config)
      calc_X.load_from_db(db_conn, today_date, ticker_X)
      stats_X = load_session_stats(
          db_conn, ticker_X, today_date,
          n_sessions=config_n_sessions,
          delta_minutes=config_delta_minutes,
          session_mode=config_session_mode,
      )
      calc_X.set_session_stats(stats_X)
      calculators[ticker_X] = calc_X
  # indicator_cache_mode == "memory", or ticker_X already restored above:
  # calc_X already in calculators (eager pool, or restored on a prior
  # appearance this session) — no restore action needed either way.

  # Bar replay itself (session_start → current bar-1, or last-known-bar →
  # current bar-1 on a re-appearance) happens via Watchdog Polling Loop's
  # own Step 3 loop once this ticker's bars are fetched this cycle — not
  # duplicated here. Step 3's loop is written to handle any number of
  # missed bars uniformly, so first-appearance (potentially many bars since
  # session start) and re-appearance (typically few bars) are the same code
  # path with no special-casing.
```

---

## on_bar_close() Sequence

```
CachingCalculator.on_bar_close(new_bar, ticks_for_bar):

  For each indicator with precalculate_bars > 0 (Layer 2):
    - CONTINUOUS (ma, ema, rsi, etc.): incremental append and recalculate
    - fibonacci: monotonic deque O(1) update (swing_high, swing_low)
    - vwap: accumulate today's cumsum
    - lee_ready, tpm, avg_vol_per_tick: accumulate today's ticks

  sr_levels and gap_percentile: NOT updated here.
    - sr_levels: recomputed per-entry-point via get_for_entry()
    - gap_percentile: fixed after on_regular_session_open()
```

---

## on_regular_session_open() Sequence

Called once when the 09:30:00 bar is confirmed closed (09:31 bar arrives).

```
CachingCalculator.on_regular_session_open(bars_including_930):
  gap_pct = calculator.gap_percentile(bars, date, n_sessions, session_stats)
  self._fixed["gap_pct"] = gap_pct
```

---

## get_for_entry() — per-entry-point dispatch

Called by FeatureExtractor.extract() in live mode (via CachingIndicatorCalculator).

```
CachingCalculator.get_for_entry(indicator, bars_up_to_t1):
  if indicator == "sr_levels":
      # per-entry-point recomputation (scipy prominence — not cacheable)
      return self.sr_levels(bars_up_to_t1.tail(window_bars), n_levels, window_bars)
  if indicator == "gap_pct":
      return self._fixed.get("gap_pct", float("nan"))   # scalar
  return self._cache[indicator]   # sliced to t-1 by FeatureExtractor
```

---

## Exit Architecture (WS-primary / REST-backstop) — R-2

tp/sl detection for OPEN POSITIONS ONLY (entry-side watchdog over the full
universe, below, is unchanged — a different problem at a different scale).
Feeds the exit-decision consumed by Position Manager Loop Step 2.

**Primary — WebSocket tick stream.** Subscribed at entry order submission
(gate-passed, order accepted — see Watchdog Polling Loop step 5c), not on
fill: a position carries the risk this stream exists to watch from the
moment an order is live, not just from its first fill, and this also
folds the ticker-cap accounting together — the subscription count and
`execution.max_tickers`'s count of `status IN ('pending', ...)` rows now
move in lockstep by construction, so subscriptions can never exceed
`execution.ws_ticker_limit` on their own. Subscriptions are per TICKER,
not per position, so several positions on one ticker share one
subscription. Price tracking holds one of the two WS sequences the API
permits for the whole session; see "WS sequence lease" under Position
Manager Loop for how the second one is shared.
Immediately after the subscribe call, REST gap-fill the short window
between the subscribe request and the subscription actually going active —
pure network/handshake latency now, not the fill-to-subscribe gap this
covered before subscription moved earlier — deduped by the global
tick-dedup rule (see utils.md's `stitch_ticks()`). Each inbound tick:
```
breach_up = price >= fill_price * (1 + tp_pct)
breach_dn = price <= fill_price * (1 - stop_loss_pct)
```
Ticks are received and deduped from the moment of subscription, but the
two lines above are only evaluated once `fill_price` is known (the
position's first fill has landed) — a tick arriving while
`status='pending'` has no reference price to compare against yet and is
simply not evaluated, not lost (the dedup layer has already recorded it,
so nothing arriving before the first fill needs a separate backfill once
`fill_price` becomes available).

Piggybacked on this same inbound-tick handler, no new subscription or
call: level-1 price, side-total resting size, and session-cumulative
filled volume are extracted from every tick into `bid_ask_snapshots`
(`source='ws_tick_piggyback'`) — see db_schema.md.

**2-print guard**: a breach fires only when the SAME direction breaches on
two consecutive raw ticks (no halt-resumption special case). Rationale: in
this universe, a single bad print (odd-lot / out-of-sequence correction)
tripping a stop-loss is a worse risk than a one-tick delay on a real move.
On a confirmed breach: set `status='exiting'` atomically (guards against
double submission), setting `live_positions.exiting_since = now` if it is
not already set (db_schema.md — never overwritten once populated), and
submit the sell (shadow: record hypothetical; order-type/pricing logic —
market vs. limit-at-bid — lives in Position Manager Loop Step 2/3, shared
with the time_limit/session_end exit paths rather than duplicated here).
Unsubscribe when either: the position closes (as before), or the entry
order ends with zero shares filled (`entry_rejected` / `entry_canceled` —
nothing was ever exposed to track). A fill of any size
(`partial_open` or beyond) keeps the subscription regardless of what
becomes of the unfilled remainder.

**Backstop — periodic-loop REST tick polling (WS-dead only).** When the WS
stream for a position is down, the Position Manager Loop polls that
ticker's individual TICKS via REST at the SAME granularity WS would
deliver (not 10-tick OHLC bundles), and applies the SAME 2-print guard.
Because both paths consume identical tick data with an identical filter,
there is NO accuracy/filter asymmetry between them — only latency differs
(push-immediate vs. up-to-one-poll delay). A poll may return several
buffered ticks at once; they are evaluated in sequence, so detection
accuracy matches WS — only detection TIME lags. `utils.track_price_breach()`
is NOT used on this path; it is backtest-only as of this patch.

**Time-based triggers are not on the WS path.** `time_limit` and
`session_end` are wall-clock/bar-count conditions, evaluated by the
periodic loop (Position Manager Loop Step 2) regardless of tick arrival —
a tick may never arrive for a low-liquidity or halted name near close, and
a time trigger must still fire. WS is fixed to price-breach only.

**R-3: moving `session_end` to WS was considered and rejected.** Each WS
tick does carry an `hour`, so a `now >= session_close_exit_time` check is
technically expressible on the WS path — but that check would only ever
fire when a tick arrives, which is exactly the case that fails for a
low-liquidity or halted name near close: `session_end` must fire on schedule
whether or not a tick ever does. This is structural, not a tuning gap —
tying a schedule-based trigger to tick arrival trades away the one property
(fires regardless of tick activity) that makes it useful. On a normal,
liquid name where WS ticks arrive continuously, WS's price-breach path and
the periodic loop's `session_close_exit_time` check can both become true in
the same iteration; see the ordering rule in Position Manager Loop Step 2
(tp/sl wins, `status='exiting'` guarantees a single submission).

**Concurrency.** WS readers and the periodic loop run on a single asyncio
event loop so position status transitions serialize naturally;
`status='exiting'` is the single-submission guard where WS and the
periodic loop could otherwise race (e.g. a simultaneous price breach and
`session_close` — see R-3 for the tp/sl-wins ordering rule).

**Config-driven, not hardcoded.** `execution.max_tickers` (the
subscription bound above), `execution.max_positions_per_ticker`, and
`position_check_interval_seconds` (the REST backstop's poll frequency) are
read from config at every use in this section and elsewhere — this spec
deliberately does not commit specific values beyond the defaults already in
Config Keys and `execution:`. Any future change to R-4's circuit-breaker
thresholds, R-5's entry-gate cap checks (Watchdog Polling Loop step 5c.0),
or `compute_position_size()`'s notional/exposure sums must read the same
config keys rather than hardcode a value independently. This is a standing
rule for future edits, not a pointer to unfinished work: R-4's thresholds,
freeze scope, and no-auto-clear behaviour are fully specified in Circuit
Breaker above, with their values in `execution:`.

Inf mode (`0`) on either cap means "unlimited" for BacktestEngine only.
LiveModeRunner clamps: `max_tickers` 0 → 50 (the vendor limit above, which
is not a configurable choice), `max_positions_per_ticker` 0 →
`floor(execution.max_hold_bars / execution.entry_cooldown_minutes)` — the
largest same-ticker stack the cooldown can admit inside one position's
maximum hold. Both clamps are derived, not configurable, in the same spirit
as `get_execution_param()`'s hard bounds.

---

## Position Manager Loop

Monitors open positions independently of the watchdog loop. A single
shared loop serves all open positions on one global timing grid (not a
per-position independent timer) — a shared clock beats per-position phase
alignment on every axis measured (compute batching, backtest reproducibility,
debuggability), with no compensating benefit found for per-position timing.

At position open (buy fill, real or shadow), the caller initializes:
```
position.bars_since_entry  = []   # grown each iteration; used for
                                  # time_limit / session_end bar counting
# No entry_ticks (R-2): tp/sl detection is WS-primary / REST-backstop over
# the real tick stream (see Exit Architecture below), not
# track_price_breach() — the former Phase-1 t-bar-ticks input is gone.
# track_price_breach() itself is unchanged and remains backtest-only.
```

**In-flight order tracking** (real orders only — shadow mode has no real
order): covers BOTH order types and BOTH sides. Fills are not returned by
the order-submission API (R-7); they arrive on a separate channel, so every
accepted order is tracked here until it reaches a terminal state:
```
in_flight_orders: dict[order_id, dict]
# {ticker, side: 'entry'|'exit', submitted_at, limit_price,
#  requested_quantity, cum_filled_qty, weighted_avg_price}
# R-2: runtime CACHE only — the SSoT is the live_positions row (status
# 'pending'/'partial_open' for entries, 'exiting' for exits). Populated at
# submission, and rebuilt from live_positions WHERE status IN
# ('pending','partial_open','exiting') on a warm restart.
```

**WS sequence lease.** The API permits two concurrent WS sequences. Price
tracking (Exit Architecture) holds one for the whole session, since a
position is almost always open and one sequence multiplexes up to 50
tickers. The second is leased by refcount: acquired when in-flight entry
orders go 0 → 1, released when they return to 0 after
`fill_stream_linger_seconds` (a linger window, so a burst of entries does
not thrash subscribe/unsubscribe). `fill_stream_mode: "dedicated"` is the
same mechanism with the lease taken at session start and never released.

Constraints this places on any FUTURE second consumer of that sequence
(e.g. realtime quote monitoring): fill tracking preempts unconditionally,
so such a consumer must tolerate being evicted at any moment and must not
assume continuous observation. A use needing an unbroken series is not
compatible with shared mode.

**Exits are REST-only.** Exit fills are polled on this loop's existing
cadence rather than leased onto the WS sequence. Order state is queryable,
not a perishable event, so a missed push costs latency, not correctness —
and unlike an entry (whose fill starts tp/sl monitoring), knowing an exit
filled sooner changes no decision: the position is already closing and any
unfilled remainder stays tracked either way. Keeping exits off the lease
also removes the only unbounded holder, since an exit has no give-up
timeout (below).

**Fill accounting invariant.** For a given order_id, `cum_filled_qty` must
always equal the broker's own count of that order's filled quantity — never
a value arrived at by ADDING reports from different channels together. WS
(primary) and REST (backstop) both report on the same underlying fills, so
naive accumulation double-counts on the routine case of a WS event and a
later REST poll describing the same fill, on WS reconnect if a missed event
is replayed, and on Warm Restart when `in_flight_orders` is rebuilt from
`live_positions` (whose `quantity` is already-filled shares) and then sees
further reports that repeat pre-crash fills. Entry-side double-counting
reaches `requested_quantity` early and confirms `open` on a wrong quantity;
exit-side is worse — it reaches `position.quantity` early and marks
`closed` while shares remain, and the remainder then has NO loop tracking
it until the next session's Broker Reconcile (finding 12) surfaces it as an
unknown broker position, exposed overnight in between. Failure in the
opposite direction (under-counting) only delays `closed`/`open`, which
stays tracked and is visible via the finding above — the two failure modes
are not symmetric, which is why the mechanism below is structured to make
over-counting impossible rather than merely unlikely.

**Fill-stream staleness detection.** No new freeze reason and no new
polling schedule — this reuses the REST backstop call this loop already
makes every `position_check_interval_seconds` cycle for every order in
`in_flight_orders`. On each such call, before folding the result into
`seen_fills` as described above: if the REST response's own
`cum_filled_qty` is AHEAD of what `seen_fills`'s WS-derived state currently
reflects for that order_id, sustained across more than one consecutive
cycle, that is evidence the WS account fill stream has gone quiet while
REST keeps working — surfaced as `health_report.md` finding 24, warn
severity. Recorded via `record_health_event(finding_name=
'fill_stream_staleness', detail={order_id, entered_at, sustained_cycles})`
with `write_fn=db_write` (R-9), ONCE PER STALE EPISODE: written on entry
into the stale state and armed again once the state clears, so a second
episode on the same order IS recorded. This deliberately differs from
finding 18's once-per-order rule — an order with several fills can go
stale, recover, and go stale again, and that recurrence is precisely the
signal. Deliberately NOT a `freeze_reasons` trigger and NOT related to
Bar-Close Authority above: those judge the separate bar/price-data
channel, while this is scoped entirely to the account-wide fill-event
stream, and correctness here is already unaffected regardless — the same
fold-into-`seen_fills` mechanism absorbs whichever channel reports a given
fill first, so a stale WS stream costs detection latency, not correctness,
exactly as the REST backstop was already designed to tolerate.

Primary mechanism, used when individual fills carry a stable, unique ID
(api_contract_checklist.md T-7/T-8 confirm this before Pilot): maintain
`seen_fills[order_id]`, a map from fill ID to (qty, price), idempotently
updated by every report regardless of channel or whether it is a delta
(one fill) or a bundle (several fills, as a REST query may return) —
inserting under an already-seen ID is a no-op. `cum_filled_qty` and
`weighted_avg_price` are RECOMPUTED from that map's current contents after
each update, never incremented. This makes the result invariant to
duplicate reports, out-of-order arrival, and which channel reported first,
and it treats a WS delta and a REST bundle identically — both are just
fills to fold into the same map. On Warm Restart, `live_positions.quantity`
seeds a lower bound while `seen_fills` rebuilds from a fresh REST query, if
that query is confirmed to return an order's complete fill history rather
than a paginated or windowed slice.

Fallback mechanism, for a vendor without stable per-fill IDs: treat each
report's OWN cumulative fields, when present, as the current state rather
than a delta to add — `cum_filled_qty = max(cum_filled_qty,
reported_cum_qty)`, with `weighted_avg_price` taken from that SAME report
(never averaged separately from a different report's quantity, which would
pair a quantity and a price from different moments).

Either mechanism is guarded by the same check: a computed `cum_filled_qty`
that is LOWER than the value already held is never accepted — logged as a
health_report finding instead. This does not fire under either mechanism's
normal operation; it exists for the case an ID turns out not to be as
stable as assumed (e.g. reissued after reconnect) or a bundle arrives
truncated.

```
loop every position_check_interval_seconds (config, default: 5s):

  For each order_id in in_flight_orders where side == 'exit':
    # WS account fill events are primary; this poll is the backstop.
    # Fold every fill into seen_fills[order_id] (or the fallback path) as
    # described above; cum_filled_qty and weighted_avg_price are read from
    # the result, never accumulated directly from the raw event.
    if cum_filled_qty >= position.quantity:
        db_write: transition the live_positions row to status='closed';
            log the exit to trade_log with weighted_avg_price and
            exit_date/exit_bar of the final fill
        remove from in_flight_orders
    else:
        stay 'exiting' — NO give-up timeout WITHIN the session. An entry may
        abandon its unfilled remainder and settle for a smaller position, but
        an unsold remainder is still exposed to the very risk that triggered
        the exit, so the order stays tracked for as long as the session runs.
        # The one bound is the process itself (R-9): at
        # live_mode.session_hard_exit_time the order is canceled and the
        # position is handed to the next session's Broker Reconcile under
        # the Unified Overnight Policy — see Session Shutdown. That is a
        # process-level cap, not a per-order give-up: nothing here abandons
        # an unfilled remainder while the session is still running.
        # Observability, not policy: an exit order open beyond a
        # configured age is surfaced as a health_report finding (18).
        # Forced liquidation / re-submission was deliberately deferred
        # past this observability-only baseline in an earlier session —
        # see below for what ended up designed on top of it.

        stuck_age = now - live_positions.exiting_since   # NOT submitted_at
                                                          # — see db_schema.md;
                                                          # a resubmission below
                                                          # must not reset this
        if stuck_age >= config["live_mode"]["exit_order_stuck_minutes"]:
            if this order_id has not already been recorded as stuck:
                record_health_event(finding_name='exit_order_stuck',
                    detail={order_id, age_seconds, cum_filled_qty,
                    quantity}) via write_fn=db_write
                # R-9 — ONCE per order_id, on the FIRST crossing. This
                # loop re-satisfies the condition every
                # position_check_interval_seconds, so an unconditional
                # write would emit hundreds of rows for one stuck order.
                # health_report.md finding 18 pairs this event count with
                # its existing point-in-time snapshot of orders still
                # outstanding at report time — the event answers "how
                # often", the snapshot "what is open right now".
            if this order_id's own type != "market":
                cancel this order_id; submit a new market order for the
                    remaining (position.quantity - cum_filled_qty) shares;
                    update in_flight_orders to the new order_id
                # Final backstop, regardless of exit_order_type
                # (execution_common.md) — an escalation, not a reset:
                # live_positions.exiting_since is untouched, so stuck_age
                # keeps accumulating against the original clock.
            # else: already market — nothing left to escalate to. Stays
            # tracked exactly as before; finding 18 keeps surfacing it.

        elif config["execution"]["exit_order_type"] == "limit" and this
             order_id's own type == "limit":
            bid = REST query, this ticker's current bid (one call, folded
                into this loop's existing position_check_interval_seconds
                cadence — no separate polling schedule; exits are always a
                sell in this system, so the resting price tracks the bid,
                never the ask)
            if bid != this order_id's current resting limit_price:
                amend this order_id's price to bid — a single order
                    amendment, not cancel-and-resubmit, so the order_id
                    (and therefore fill tracking against it) is undisturbed
            # Re-quoted every cycle unconditionally, whether or not the
            # market moved much since the last check — the simplest
            # correct behavior. A move-triggered variant (skip the amend
            # call inside some tolerance band) is a possible later
            # refinement, not designed here.

  For each order_id in in_flight_orders where side == 'entry':
    order_status from the WS account fill stream (primary) or a REST
        order query on this cadence (backstop) — folded into
        cum_filled_qty / weighted_avg_price via the same fill-tracking
        mechanism as the exit branch above
    if rejected (including a market order's insufficient-funds-at-actual-
        price rejection, and any limit rejection):
        db_write: transition the live_positions row to status='canceled'.
            Subordinate to that same write, log trade_log row:
            exit_reason='entry_rejected', quantity=0, fill_price=p_entry,
            exit_bar=entry_bar, reject_reason = the broker's refusal text
            verbatim. Never-opened family (db_schema.md): excluded from
            fit_execution_params(), counted as a cooldown attempt.
        remove from in_flight_orders
    elif filled in full (cum_filled_qty == requested_quantity):
        db_write: transition the live_positions row to status='open'
            (fill_price = weighted_avg_price, fill_second, quantity =
            cum_filled_qty)
        remove from in_flight_orders → proceeds into the open-position
            handling below
    elif filled in part (0 < cum_filled_qty < requested_quantity):
        db_write: transition the live_positions row to
            status='partial_open' (quantity = cum_filled_qty so far;
            requested_quantity unchanged). The filled shares are a real
            position from this moment: they enter exit management and
            count toward step 5c.0's caps, while the order itself stays
            in_flight awaiting further fills.
        # No 'full_open' status: the two ways partial_open ends — reaching
        # requested_quantity, or abandoning the remainder at
        # cancel_after_seconds — both mean "no longer awaiting fills",
        # which is exactly 'open'. This matches backtest, where a
        # partially-filled entry simply proceeds sized down.
    elif now - submitted_at >= config["execution"]["cancel_after_seconds"]:
        submit cancel request to trading API for order_id
        # race: cancel request may lose to a fill that happened moments
        # before it's processed — trading API's response to the cancel
        # attempt itself resolves this (a "too late, already filled"
        # response is treated as a fill, not a cancellation)
        if canceled (not a race-lost fill):
            if cum_filled_qty == 0:
                db_write: transition the live_positions row to
                    status='canceled' — THE SINGLE canceled-transition point
                    (R-2). Subordinate to that same write, log trade_log row:
                    exit_reason='entry_canceled', quantity=0, fill_price=p_entry,
                    exit_bar=entry_bar (see db_schema.md). Because the
                    transition is the single point (idempotent — a second
                    attempt to cancel an already-'canceled' row is a no-op),
                    a post-crash Broker Reconcile that also cancels this same
                    order cannot double-log entry_canceled.
            else:
                db_write: transition 'partial_open' → 'open' with
                    quantity = cum_filled_qty. A partially-filled order is
                    never 'canceled': shares were actually bought, so this
                    is a smaller position, not a non-event.
        remove from in_flight_orders

  For each open position:
    1. Fetch current bars from trading API (entry → now; the API always
       returns the full range, so this naturally becomes
       position.bars_since_entry each call — no incremental bookkeeping
       needed)

    1a. Halt check — position-scoped only, not applied to new-entry
        candidates. API-primary, tick-rate fallback (see P-1's halt-status
        endpoint integration — utils.query_halt_status()):

        Once per Position Manager Loop iteration (not once per position —
        same single-shared-loop batching principle as the global polling
        design above; `execution.max_tickers × execution.max_positions_per_ticker`
        is small by design, so this bulk call is always small too):
        ```
        halt_status = utils.query_halt_status(
            tickers=[p.ticker for p in open_positions] +
                    [o["ticker"] for o in in_flight_orders.values()
                     if o["side"] == "exit"],
            # In-flight exit tickers folded into the SAME bulk call rather
            # than a second query — see "Halt-clear handling for an
            # in-flight exit order" below for why this list needed
            # extending at all. Deduplicated: a ticker already covered via
            # open_positions is not queried twice.
            trading_api_url=config["live_mode"]["trading_api_url"],
            chunk_size=config["live_mode"]["bulk_api_chunk_size"],
            # bulk_api_chunk_size defined once in metadata_crawler.md's
            # Config Keys (shared pipeline_config.yaml) — barely matters
            # here since open_positions is small (bounded by
            # execution.max_tickers × execution.max_positions_per_ticker,
            # the two axes that replaced the old single max_positions —
            # see execution_common.md's Config Keys), almost always one
            # chunk regardless of value.
        )
        ```

        For each open position:
        ```
        if halt_status is not None and position.ticker in halt_status:
            is_halted = halt_status[position.ticker]   # API authoritative
            signal_source = "api"
        else:
            # whole call failed, or this ticker missing from the response —
            # fall back to the tick-rate heuristic
            Fetch recent ticks from trading API (trailing
            halt_check_window_seconds, config default: 60s)
            tick_rate_per_min = len(ticks) * (60 / halt_check_window_seconds)
            is_halted = tick_rate_per_min < halt_heuristic_tpm (config, default: 10)
            signal_source = "tick_rate_fallback"

        if is_halted:
            position.status = 'halted'
            alert (see health_report.md), tagged with signal_source for
            diagnosability (see health_report.md's new fallback-rate finding)
            → skip Steps 2-4 this iteration (no reliable price to act on)
            # R-3: if now >= config["execution"]["session_close_exit_time"]
            # (i.e. this position is skipping session_end specifically
            # because it cannot trade, not merely skipping an ordinary
            # mid-session iteration), log explicitly and
            # record_health_event(finding_name='overnight_halt_carry',
            # detail={ticker, position identifier}) via write_fn=db_write
            # (R-9 — see health_report.md finding 14) — the live_positions
            # row stays
            # status='halted' and is picked up by the next session's
            # Broker Reconcile under the Unified Overnight Policy (see
            # "Broker Reconcile (shared procedure)" below). The silent
            # skip becomes an owned, visible handoff instead of quietly
            # carrying a position no health check would otherwise surface.
            # Recorded once per position at the carry, not once per
            # iteration: the condition stays true on every subsequent
            # cycle. The same position is counted again by finding 15 when
            # it is liquidated next session — two moments of one carry,
            # deliberately not merged.
        else:
            position.status = 'active' (clears automatically once either
            signal recovers — no separate resumption-detection logic needed)
        ```

        No separate "resumption-auction exit" pre-registration mechanism —
        deliberately not built. A `'halted'` position is not removed from
        this loop's iteration set; it simply skips Steps 2-4 while halted.
        The instant the active signal (whichever of the two above produced
        it) recovers, `position.status` flips back to `'active'` and the
        very next iteration of this same loop resumes ordinary Step 2 exit
        evaluation automatically — the loop structure already IS the
        "pre-registered" re-check, with no extra state or broker-side
        resting order needed. (An actual resting auction order was
        considered and rejected: no confirmed trading-API support for
        halt-resumption order types, and an unprotected order risks a very
        unfavorable fill if the resumption gap moves against the position —
        same reasoning as Entry Slippage Model's limit-order option
        existing for the analogous entry-side concern.)

        **Halt-clear handling for an in-flight exit order.** A different
        problem from the above: that section governs a position that has
        NOT yet had an exit submitted (status still 'open'/'halted');
        this covers a position ALREADY `status='exiting'`, with a real
        order outstanding in `in_flight_orders`, whose ticker halts and
        then clears while that order is still unfilled. Whether the
        broker preserves, cancels, or cross-executes a resting order
        through a halt is unverified (api_contract_checklist.md T-12), so
        this is built to be correct under any of the three:

        ```
        last_halt_state: dict[ticker, bool]   # runtime only, this
            # iteration's is_halted vs. the PRIOR iteration's, per ticker
            # in the combined query above — not persisted; on a warm
            # restart the first post-restart iteration simply has no
            # "prior" to compare against, so at most one clear-edge event
            # is missed per crash, not a correctness gap (the order is
            # still tracked and re-evaluated on the next ordinary cycle
            # regardless)

        for ticker in in_flight_exit_tickers:
            was_halted = last_halt_state.get(ticker, False)
            is_halted  = halt_status.get(ticker, was_halted)  # missing
                # from the response this cycle → assume unchanged, do not
                # manufacture a spurious clear-edge from a partial response
            if was_halted and not is_halted:
                # clear edge — re-query THIS order's own status immediately,
                # not waiting for the next position_check_interval_seconds
                # cycle's ordinary fill-tracking pass
                order_status = REST query, this order_id's current state
                if order_status says the order no longer exists:
                    # broker canceled it during the halt — T-12's "auto-
                    # canceled" branch confirmed for this event
                    submit a replacement order via the same exit_order_type
                        branching as initial submission (Position Manager
                        Loop Step 2/3) — market or limit-at-current-bid,
                        for the remaining unfilled quantity; update
                        in_flight_orders to the new order_id
                    # live_positions.exiting_since is NOT touched — this is
                    # a resubmission, not a new exit; exit_order_stuck_minutes
                    # keeps counting from the original clock (see In-flight
                    # order tracking's exit-side loop)
                    record_health_event(finding_name=
                        'inflight_exit_gone_at_halt_clear',
                        detail={ticker, order_id}) via write_fn=db_write
                    # R-9 — see health_report.md finding 25. A failure of
                    # this write is swallowed and must not break the
                    # halt-clear resubmission above.
                # else: order still exists (survived the halt, or already
                # filled/partially filled through the resumption cross) —
                # no action; the ordinary fill-tracking pass on this same
                # loop's next cycle picks up whatever state it is in
            last_halt_state[ticker] = is_halted
        ```

    2. Exit decision for this position:
       # R-2: tp/sl breach comes from the Exit Architecture below
       # (WS-primary / REST-backstop, 2-print guard) — NOT from
       # utils.track_price_breach(), which is now backtest-only.
       if a confirmed tp/sl breach is pending for this position:
           exit_reason = "take_profit" if breach_direction == "up" else "stop_loss"
       elif bars elapsed >= config["execution"]["max_hold_bars"]:
           exit_reason = "time_limit"        # wall-clock/bar-count — this loop, not WS
       elif now >= config["execution"]["session_close_exit_time"]:
           exit_reason = "session_end"       # wall-clock — this loop, not WS
       else:
           continue to next position (no exit yet)
       # Ordering when a tp/sl breach and session_close both become true in
       # the same iteration: tp/sl wins. status='exiting' (see Exit
       # Architecture) guarantees a single submission either way, and sets
       # live_positions.exiting_since = now if not already set — the
       # time_limit/session_end paths reach 'exiting' here rather than in
       # Exit Architecture, so the same "first time only" write applies at
       # this transition too (db_schema.md).

    3. if config["live_mode"]["shadow_mode"]:
           sell_rate = config["execution"]["sell_rate_tp"] if exit_reason == "take_profit" \
                       else config["execution"]["sell_rate_sl"]
           weighted_avg_exit_price, filled, unfilled, _ = execution_common.simulate_exit_fill(
               ticks_exit=..., ohlcv_exit=..., position_size=position.quantity,
               breach_bundle_idx=..., breach_price=breach_price,
               sell_rate=sell_rate, halts_df=...,
           )
           log hypothetical exit to trade_log (is_shadow=TRUE), breach_price
           also recorded alongside the simulated fill for future
           realized-vs-simulated comparison
       else:
           if config["execution"]["exit_order_type"] == "market":
               order_id = submit order via trading API: quantity=`position.quantity`
                   - `cum_filled_qty so far` (0 for a fresh exit), order_type="market"
           else:  # "limit"
               bid = REST query, this ticker's current bid (one call;
                   exits are always a sell in this system, so the resting
                   price tracks the bid — see execution_common.md's
                   exit_order_type)
               order_id = submit order via trading API: quantity=`position.quantity`
                   - `cum_filled_qty so far`, order_type="limit", limit_price=bid
           # Re-priced every position_check_interval_seconds cycle
           # thereafter while still outstanding (limit case only) — see
           # In-flight order tracking's exit-side loop below. Escalates
           # unconditionally to a market order at
           # live_mode.exit_order_stuck_minutes regardless of which branch
           # above was taken — same section.
           breach_price still logged for the same comparison purpose
           # R-2: breach_price here is now the OBSERVED price of the
           # confirming (2nd) breach tick from the WS/REST stream — a real
           # observation, not a bundle interpolation — improving the
           # realized-vs-simulated comparison's quality. For time_limit/
           # session_end exits (no breach), this field is not applicable.

    4. Log exit to inference_log
```

---

## Config Keys

```yaml
live_mode:
  poll_interval_seconds:          1
  position_check_interval_seconds: 5    # shared timing grid — see Position Manager Loop
  # max_positions removed — replaced by execution.max_tickers /
  # execution.max_positions_per_ticker (two independent axes; a single
  # global count could not express "5 positions, all on one ticker").
  # max_hold_bars removed (R-6) — execution.max_hold_bars is the single
  # source, already read by this file's restart_gap_exit / overnight_exit
  # cutoff and by backtest's time-limit exit.
  session_start_workers:          8      # parallel workers for session_start_compute()
  indicator_cache_mode:           "memory"
  # "memory" (default): all CachingCalculator state held in RAM after session_start_compute()
  #                     Lowest latency; ~12GB sustained on 32GB+ server.
  # "db":               Indicator cache persisted to indicator_cache table after
  #                     session_start_compute(); RAM released; loaded per-ticker on
  #                     first watchdog event (50~200ms overhead).
  #                     Use for memory-constrained environments.
  watchdog_url:                   "http://watchdog-service/candidates"
  trading_api_url:                "http://trading-api"
  trading_api_ticker_url:         "http://trading-api/tickers/today"

  # R-1: 09:20 recheck moved in-process (see "In-Process Premarket
  # Recheck" below) — no longer a cron. wall-clock America/New_York.
  premarket_recheck_time:         "09:20"

  # R-3: periodic-loop session_end trigger (Position Manager Loop Step 2)
  # — deliberately NOT on the WS path, see Exit Architecture's
  # "Time-based triggers" note. Matches backtest's 15:59 close-exit
  # convention.
  # session_close_exit_time removed — promoted to execution: (R-7), where
  # BacktestEngine reads it too; a value both engines must agree on cannot be
  # declared on one side only (same defect R-6 fixed for max_hold_bars).
  fill_stream_mode:               "shared"   # "shared" | "dedicated" — see
                                             # Position Manager Loop's WS
                                             # sequence lease
  fill_stream_linger_seconds:     30         # idle hold before releasing the
                                             # leased sequence, so a burst of
                                             # entries does not thrash
                                             # subscribe/unsubscribe
  clock_check:
    source:                       "ntp_daemon"  # | "vendor_api" | "disabled"
    max_offset_seconds:           1.0
    on_exceed:                    "abort"       # | "warn"
  retention_probe:
    enabled:                      true
    lookback_days:                14           # ask from further back than
                                               # retention is expected to reach
    assumed_days:                 5            # fallback when the probe fails
  margin_ratio_url:               "http://trading-api/margin"
  margin_ratio_fallback:          4.0        # used only if the query above
                                             # fails — see Session Start
                                             # Probes for why this is not 1.0
  exit_order_stuck_minutes:       10         # health_report finding 18's age
                                             # threshold
  session_hard_exit_time:         "20:00"    # R-9: process hard cap — see
                                             # "Session Shutdown". Wall-clock
                                             # America/New_York. Matches the
                                             # after-market ingestion boundary
                                             # (200000) and is the value
                                             # health_report.md's
                                             # drain_timeout_seconds was sized
                                             # against. Ordinary exit is
                                             # earlier: all-flat past
                                             # execution.session_close_exit_time.
  min_watchlist_size:             30         # seed value — Feed Outage
                                             # trigger condition 2's sample-
                                             # size floor (see "Feed Outage
                                             # Recovery"); below this many
                                             # candidates in a poll cycle,
                                             # the percentage check does not
                                             # evaluate at all
  bar_close_grace_seconds:        5          # seed value — Bar-Close
                                             # Authority's per-ticker
                                             # miss deadline; how long past
                                             # a minute's close a bar is
                                             # tolerated before counting as
                                             # missed (UNVERIFIED vendor
                                             # latency, see
                                             # api_contract_checklist.md T-13)
  bar_latency_max_error_seconds:  3          # = 3 x poll_interval_seconds.
                                             # Per-sample error bound for
                                             # T-13's latency measurement
                                             # (see Bar-Arrival Latency
                                             # Measurement): admit a sample
                                             # only if its own overstatement
                                             # is within this. Anchored to
                                             # poll_interval_seconds, NOT to
                                             # bar_close_grace_seconds:
                                             # deriving it from the value it
                                             # helps calibrate would make an
                                             # over-large grace admit
                                             # over-loose samples that then
                                             # confirm the over-large grace.
                                             # Re-examined only when
                                             # poll_interval_seconds changes,
                                             # or when finding 26 reports
                                             # heavy sample loss or a gap
                                             # between its two error classes
  bar_integrity_freeze_ticker_count: 5       # seed value — distinct tickers
                                             # with a negative bar-latency
                                             # sample (finding 27) before the
                                             # per-ticker exclusion is
                                             # promoted to a session-wide
                                             # entry freeze. A timestamp-
                                             # convention mismatch reaches
                                             # this within a minute or two;
                                             # isolated premature bars
                                             # should not
  api_max_tickers_per_second:     100        # assumed throughput ceiling used
                                             # for chunk sizing — UNVERIFIED,
                                             # see api_contract_checklist.md

  # Halt detection (Position Manager Loop only — see is_tradable() in
  # execution_common.md for the separate, unrelated new-entry rename gate).
  # trading_api_url above is the API-primary signal (see
  # utils.query_halt_status()); the two keys below configure the
  # tick-rate FALLBACK only, used when that call fails or omits a ticker.
  halt_check_window_seconds:      60
  halt_heuristic_tpm:             10      # ticks/min below this → position.status='halted'

  # Shadow mode (see Position Manager / Watchdog Loop shadow_mode branches
  # and new health_report.md / ops doc for the comparison methodology)
  shadow_mode:                    false
  shadow_duration_weeks:          0       # 0 = sync to class_balancer.outer_fold.test_weeks
                                          # (currently 6); >0 = explicit override

# session_start_cash is not a config key — queried once from the trading
# API at Session Lifecycle Step 1 and held fixed for the session (the
# `balance` argument to execution_common.compute_position_size(), mirroring
# backtest's initial_cash — see 09_backtest_engine.md's Position Sizing).

# take_profit_up5/up3, stop_loss_pct, exit_interpolation, sell_rate_tp/sl,
# position_size_cash_pct/vol_pct, per_ticker_share_cap_pct, exposure_cap_pct
# moved to the shared `execution:` config section (see execution_common.md)
# — backtest and live must not carry independently-configurable copies of
# values that are supposed to be identical between them.
```

---

## Constraints

- Ticker list is sourced from trading service API at session_start — not from DuckDB ohlcv_1min
  (today's tradable tickers only; prior-day data for non-tradable tickers is irrelevant)
- US stock splits always take effect before market open — no intra-session split handling needed
- LiveModeRunner maintains one CachingIndicatorCalculator per ticker in the tradable universe
  (eager pool initialized at session_start via parallel session_start_compute())
- Active watchlist (calculators with pending entry signals) is empty at session_start;
  tickers are appended as watchdog events arrive — not pre-populated
- session_stats bulk load (Step 2): loaded at session_start for ALL tickers from precomputed_session_stats
  (WHERE as_of_date = today — inserted by collect_daily.py the previous evening);
  used during session_start_compute() (Step 5); released from RAM after Step 5 completes
- Health Gate 1 (Step 4) runs after Steps 2-3, before Step 5's Eager Pool cost —
  session_stats/stock_meta coverage checks use only Step 2/3's already-loaded
  data; the premarket-marker check is a batch_runs lookup independent of
  either. Health Gate 2 (Step 6) is separate and runs after Step 5 specifically
  because its input (tier_used) does not exist until Eager Pool has run —
  it cannot be folded into Health Gate 1
- session_stats Phase 2 ("db" mode only): loaded per-ticker from DB on first watchdog event
  via load_session_stats(); in "memory" mode, session_stats retained in calc._session_stats
- Inferencer is instantiated once per session with the DI FeatureExtractor
- on_bar_close() called by LiveModeRunner for each ticker per bar — not by IndicatorCalculator
- Position manager loop runs independently of watchdog polling loop
- Trade execution (buy/sell API calls) is LiveModeRunner's responsibility —
  Inferencer only returns InferenceResult
- All inference_log writes include the active run_id
- LiveModeRunner does not modify DuckDB historical data — only reads for session init
  (exception: entry_points INSERT via Inferencer; inference_log, below)
- inference_log has SPLIT ownership by event, not one writer: Inferencer
  INSERTs the inference-stage events (no_detection, preload_fail,
  bars_trimmed, no_signal, suppressed); LiveModeRunner INSERTs
  event='signal_fired' rows, once, the instant the step-5c.0 gate sequence
  ends — never by a later UPDATE (see db_schema.md's gate_result). This
  split is not a convenience, it is forced: every gate reads
  LiveModeRunner's own runtime state (freeze_reasons, live position counts
  from the DB, the cooldown cache, the breaker counters, the queried
  balance), none of which Inferencer has visibility into at the moment it
  returns an InferenceResult — so LiveModeRunner is the only party that
  can ever know how a signal_fired row ends. Multiple writers on one table
  already has a precedent (entry_points: Preprocessor and Inferencer, both
  INSERT OR IGNORE) — what's new here is splitting ownership by EVENT VALUE
  within a single table rather than by row-existence.
- session_start_compute() parallelized across workers; thread-safety per-ticker guaranteed
  (no shared state between CachingCalculator instances)
- Eager Pool bars loading (Step 3) uses `utils.load_ohlcv_with_history()`, not a raw
  `ohlcv_1min` query — stitches predecessor-ticker bars when a rename occurred
  within the lookback window; no-op fast path for the vast majority of tickers
- Corporate-event split adjustment of loaded bars happens inside
  `session_start_compute()` (see `caching_calculator.md`), anchored to
  `today_date` — not performed here in LiveModeRunner
- `meta_bulk` (Step 3b) is (ticker, date)-keyed at the source (`stock_meta`)
  but resolved to a flat per-ticker dict for today's date only, since live mode
  has exactly one date in play per session — unlike training's `extract_batch()`,
  which must stay date-keyed across a ticker's multiple historical entry dates
  (see `04_feature_extractor.md`)
- `meta_bulk` depends on `metadata_crawler.md`'s premarket crawl having run
  before session start for maximum freshness; `utils.estimate_historical_meta()`
  fallback covers any field still missing at session start (rare, but not
  guaranteed-zero — vendor update latency is a known, unresolvable-by-scheduling
  limitation, see `metadata_crawler.md`)
