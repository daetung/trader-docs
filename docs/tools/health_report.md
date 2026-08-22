# Tool: Health Report

**File:** `tools/health_report.py`
**Called IN-PROCESS, not as a subprocess** — findings 5, 8, 12, 21, 22, 24
and 25 are handed LiveModeRunner's in-memory aggregates directly, which a
separate process could not receive. A CLI entry point still exists for manual
runs, but those can only produce the DB/log-derived findings.
**Seven call paths** — two scheduled per session, five event-driven, two of
the five from outside any session. See Invocation Contract below.

---

## Role

Single alerting surface for the failure signals that otherwise sit unread
in logs and tables: `batch_runs` status, `ticker_cik_map` suspended count,
`inference_log` preload-failure rate, and `metadata_crawler.md`'s structured
per-run log files. No new infrastructure — everything read here already
exists for another purpose (see db_schema.md, metadata_crawler.md).

Two scheduled invocations per session: once at session start (reports on the
overnight/premarket batches feeding into Health Gate 1/2 — see
live_mode_runner.md), once at session end (reports on the session's own
inference/trade activity).

EVENT-DRIVEN invocations fire outside that schedule. Five triggers exist:
- A circuit-breaker trip (R-4) — entries are frozen for the rest of the
  session at that point, worth knowing immediately rather than at session
  end.
- A `clock_check` abort (R-8) at session start — this one fires BEFORE the
  scheduled session-start invocation above (the clock fault stops the
  session before it opens), so without it a clock-fault day and an
  ordinary quiet day would be indistinguishable to the operator from
  outside.
- A negative bar-latency sample (finding 27) — the premise Bar-Close
  Authority rests on has been contradicted, so waiting for session end
  would mean a full day of judgements made on a broken assumption.
- The evening job finding LiveModeRunner's session-end marker missing
  (finding 11), and
- The evening job aborting at its lock-probe deadline (finding 28).
  Both of these run in the evening batch process (metadata_crawler.md),
  outside any session — so no in-memory aggregate exists for them, and the
  second cannot reach the DB at all.

The first two send the WHOLE report; the last three are TARGETED, scoped to
the finding that triggered them. A breaker trip is worth its full context,
whereas a finding that can fire many times a minute must not drag the entire
report along each time. This distinction is what `findings_subset` exists
for — see Invocation Contract.

None of this needs new delivery infrastructure — the Discord and email
channels below already existed and were simply never wired to a mid-session,
pre-session, or post-session event.

---

## Findings Gathered

```python
def gather_findings(db_conn, today_date, log_dir,
                    subset: list[str] | None = None,
                    live_aggregates: dict | None = None) -> dict:
    """
    `subset` (None = every finding) is applied HERE, at collection, not as a
    filter over the finished result: this function otherwise sweeps
    batch_runs, ticker_cik_map, inference_log, trade_log, live_positions and
    the crawler log files on every call, and finding 27 can fire dozens of
    times a minute. Filtering afterwards would leave that full sweep running
    at that rate against a live session's own DB connection.
    `live_aggregates` carries the in-memory tallies LiveModeRunner owns (see
    Invocation Contract for which findings need it, and for what happens when
    a subset asks for one of them without it).

    1. batch_runs — status of every stage for today_date; a stage with no
       row at all (not even 'running') is reported distinctly from a
       'failed' row.
    2. ticker_cik_map — entry-blocked ticker count and detail (rewritten:
       the original `status`/`suspend_reason` columns this finding named
       were removed by N-2's restructure into two independent columns,
       `rename_pending` and `quarantine_reason` — see db_schema.md).
       Predicate is is_tradable()'s own gate, inverted:
       `WHERE rename_pending = TRUE OR quarantine_reason IS NOT NULL`.
       (`rename_pending IS NOT NULL` — the naive column-rename translation
       — is always true, since the column is `NOT NULL DEFAULT FALSE`, and
       would report the table's total row count instead of a blocked
       count.)
       Reported as COUNT(DISTINCT ticker), not COUNT(*): the PK is
       (cik, ticker), so a ticker under rename ambiguity legitimately holds
       multiple rows, and COUNT(*) over-counts exactly the case
       `rename_pending` exists to flag.
       The two columns are independent and a ticker can have both set, so
       report the union count above alongside — not summed with — a
       per-column tally: `rename_pending` count and `quarantine_reason`
       count. `quarantine_reason` is further broken out by its own value
       (currently only 'corporate_event_anomaly', but the column is
       explicitly reserved for future reasons — see db_schema.md), so a
       new reason registers here without a schema or finding change.
       Detail view: `(cik, ticker, rename_pending, quarantine_reason)`.
    3. inference_log — preload_fail rate for today's run_id:
       COUNT(*) FILTER (WHERE event='preload_fail') / COUNT(*)
       Alert threshold: TBD — not yet set (same deferral status as findings
       6 and 7's thresholds below; the rate itself is always computed and
       loggable, only the warn/abort cutoff is undecided).
    4. metadata_crawler structured logs — parse logs/metadata_crawler_{today_date}.log
       for SUMMARY lines only (see metadata_crawler.md's Output / Logging).
       A stage with a batch_runs 'success'/'failed' row but no matching
       SUMMARY line is flagged as a logging-mismatch finding on its own —
       distinct from the batch's own status.
    5. tick_bar_aggregates Tier-2/3 fallback rate — LiveModeRunner's Health
       Gate 2 result for the current session, not recomputed here (see
       live_mode_runner.md). Read (R-9) from
       live_session_state.session_diagnostics rather than passed in as an
       in-memory tally, so a crashed session's value survives and the evening
       liveness probe can recover it.
    6. Live winning-rate divergence vs. backtest — placeholder only;
       method not yet defined (see the shadow/retraining spec doc for the
       comparison methodology once shadow mode has run).
    7. Execution-parameter divergence (pilot stage onward) — mean absolute
       gap between trade_log.predicted_fill_price/predicted_weighted_avg_exit_price
       and their real (fill_price/weighted_avg_exit_price) counterparts,
       over the same cumulative pilot-period window fit_execution_params()
       uses (see shadow_retraining.md). Independent of finding 6 — a model
       divergence and an execution-parameter divergence are different
       failure modes (retrain vs. recalibrate) and must not be merged into
       one signal. Threshold TBD, same deferral as finding 6.
    8. Halt-check signal-source rate (N-4: data path matches finding 5's
       pattern) — fraction of today's Position Manager Loop halt checks
       (see live_mode_runner.md's Position Manager Loop Step 1) tagged
       signal_source='tick_rate_fallback' vs. 'api'. Not recomputed here —
       LiveModeRunner tallies signal_source per check as the session runs
       and persists the counts (R-9) to
       live_session_state.session_diagnostics on the bar_latency_daily flush
       cadence, the same place Health Gate 2's tier_used summary reaches
       finding 5. Stored as counts per source rather than a precomputed
       fraction — a multi-valued numerator/denominator is why
       session_diagnostics is JSON and not a scalar column. Loss on a crash
       is bounded by one flush interval instead of the whole session. Unlike findings 6/7, this is NOT a placeholder — no
       pilot-stage accumulation is needed, since the signal is tagged on
       every check from the day P-1's halt-status endpoint integration
       ships. A high fallback rate means the halt-status endpoint is
       degrading or down, not that the strategy itself is underperforming
       — this is an infrastructure-health signal, distinct in kind from
       findings 6/7. Threshold TBD (same deferral status as the rate
       itself always being computed and loggable, only the warn cutoff
       undecided).
    9. Execution-parameter fit rejections (N-7, pilot stage onward) — count
       of this week's fit_execution_params() run, per parameter, where a
       value cleared its sample-size gate but was rejected by the
       relative-bound check (landed outside
       [current/fit_rejection_multiplier, current*fit_rejection_multiplier]
       — see shadow_retraining.md and execution_common.md's
       fit_rejection_multiplier config). Also includes get_execution_param()
       hard-bound fallbacks (a stored value outside its mathematically
       valid range — see execution_common.md), though those should never
       occur under normal operation since the relative-bound check is what
       prevents a bad value from being written in the first place; a
       nonzero count here specifically would point at manual corruption or
       a write-path bug bypassing fit_execution_params() entirely, not at
       calibration behaving unexpectedly. A nonzero relative-bound
       rejection count means the CURRENT authoritative value is older than
       the sample-size gate alone would suggest — worth a human look at
       whether the fit reflects a genuine regime shift (in which case the
       multiplier itself may need revisiting) or noise. Independent of
       finding 7 — a rejected fit and a magnitude-off predicted-vs-actual
       gap are different failure modes. Never merged with finding 7 or
       with each other for the same reason findings 6 and 7 stay separate.
    10. investing.com match rate (item N) — per run, the fraction of
        scraped investing.com calendar rows that FAILED to match
        active_ticker_universe. A rising rate signals symbology drift
        beyond what query-time normalization (metadata_crawler.md's
        crawl_corporate_events_investing()) already resolves — kept in
        place after that normalization was added, since best-effort
        matching does not guarantee zero residual mismatches. Threshold
        TBD, same deferral status as finding 8's warn cutoff.
    11. Session end marker missing (R-2) — set when the evening job's
        DuckDB-lock liveness probe (see metadata_crawler.md's "Evening job
        start gate") finds LiveModeRunner dead with no clean shutdown
        (live_session_start still 'running', no live_session_end row).
        Distinct from a normal 'failed' status — this is a crash signal,
        not a stage failure.
    12. Unknown broker order/position at reconcile (R-2/R-3) — a broker
        open order or open position with no matching live_positions row at
        any Broker Reconcile call site (session start, warm restart, feed
        outage). Should not occur under normal operation; a nonzero count
        points at a gap in the reconcile/adopt logic, not at strategy
        performance.
        Source (R-9): aggregated from health_events where
        finding_name='unknown_broker_order_or_position', not from a passed-in
        tally. detail carries call_site ('session_start' | 'warm_restart' |
        'feed_outage'), kind ('order' | 'position'), order_id and ticker, so
        the three call sites stay distinguishable and each occurrence keeps
        its own time.
    Findings 13 and 15 stay on their existing `trade_log` queries rather than
    moving to `health_events` with the other event-shaped findings (R-9).
    Each already writes its own row — `exit_reason='restart_gap_exit'` /
    `'overnight_exit'` / `'reconcile_ghost'` — carrying both the occurrence
    and its time, and those rows already survive a crash and already reach the
    operator through the ordinary report and the evening liveness probe.
    Recording them in `health_events` as well would create a second source for
    one fact, which is the defect `corporate_events`' one-row invariant exists
    to prevent. The rule the split follows: an event belongs in `health_events`
    when it leaves no row anywhere else.
    13. restart_gap_exit count (R-2) — positions liquidated on warm restart
        for a tp/sl breach detected retroactively within the crash gap, or
        liquidated immediately because the gap already exceeded
        execution.max_hold_bars. A nonzero count is expected after any
        crash; the count itself is diagnostic (frequency/severity of
        crashes), not an error signal on its own.
    14. Position held overnight: halted through close (R-3) — count of
        positions that skipped `session_end` while `status='halted'` and
        were carried to the next session's Broker Reconcile. Surfaces what
        was previously a silent skip (see live_mode_runner.md's Position
        Manager Loop Step 1).
        Source (R-9): aggregated from health_events where
        finding_name='overnight_halt_carry'; detail carries ticker and the
        position identifier. Recorded once per position at the carry, not per
        loop iteration. The same position is counted again by finding 15 when
        it is liquidated next session — two moments of one carry, deliberately
        not merged.
    15. Overnight position liquidated at session start (R-3) — count of
        `exit_reason='overnight_exit'` liquidations from the Unified
        Overnight Policy, this session. Includes the `reconcile_ghost`
        count as a separate, distinct tally within the same finding (a row
        with no matching broker position is a data/bookkeeping issue, not
        an overnight-carry issue, even though both surface at the same
        Broker Reconcile call).
    16. Corporate-event vendor conflicts (item N) — rows added to
        corporate_event_conflicts (db_schema.md) since the last run, with
        the (ticker, event_date, event_type, both values, both sources)
        detail. A conflict is not itself an error — the investing.com value
        is kept per the confirmed tie-break and trading continues — but a
        rising count, or a conflict on a large split ratio rather than a
        dividend's trailing decimal, is worth a human look: it is the only
        place the two vendors' disagreement is visible, and it is also the
        practical evidence for whether
        quarantine.corporate_event_value_tolerance is set sensibly
        (see metadata_crawler.md's upsert_corporate_event()).
    17. Never-opened entry outcomes (R-5/R-7) — this session's counts of
        exit_reason='entry_canceled' (submitted, never filled, canceled at
        cancel_after_seconds) and exit_reason='entry_rejected' (the broker
        or the account refused the submission), reported as two separate
        tallies within one finding. Kept apart from findings 13/15's
        operational family (restart_gap_exit / overnight_exit /
        reconcile_ghost) because the two families are excluded from PnL for
        different reasons and mean different things: the operational ones
        held a real position that was closed for a non-strategy reason,
        while these never opened at all (see db_schema.md's two exclusion
        families). A rising entry_canceled count points at buy_rate /
        cancel_after_seconds being too tight and is calibration evidence
        (fit_execution_params() consumes it); a rising entry_rejected count
        points at the account or the broker and is not — it is excluded
        from that calibration (see shadow_retraining.md). The
        entry_rejected tally is broken out by trade_log.reject_reason
        (R-8), stored as the broker returned it: insufficient margin, a
        non-permitted ticker and a malformed order call for three
        different responses, and under the risk-based intraday margin
        regime that replaced the PDT rule in June 2026 a rejection count
        that rises with exposure is normal — without the reason there is
        nothing to separate that from a real fault.
    18. Exit order still in flight (R-7) — exit orders whose in_flight age
        exceeds live_mode.exit_order_stuck_minutes without completing, with the
        (ticker, order_id, age, cum_filled_qty / quantity) detail. An exit
        has no give-up timeout by design — an unsold remainder stays
        exposed to the very risk that triggered the exit — so on a thin
        name an order can in principle stay open indefinitely. The same
        threshold now also drives an automatic response: past it,
        live_mode_runner.md's In-flight order tracking escalates the
        order — to market inside the regular session (a no-op if already
        market), and outside it, where the venue refuses a market order, by
        advancing the exit limit's spread position one step per cycle
        toward and past the bid. This
        finding still fires independently of that escalation — it reports
        exposure duration, not whether an escalation attempt has occurred,
        so a row here may be pre- or post-escalation depending on whether
        the escalated order has filled by report time.
        TWO TALLIES (R-9), reported together. (a) EVENT COUNT — how many
        times the threshold was crossed this session, aggregated from
        health_events where finding_name='exit_order_stuck'; detail carries
        order_id, age_seconds and cum_filled_qty/quantity, and the event is
        written ONCE per order_id on its first crossing (the 5s loop
        re-satisfies the condition every cycle). (b) SNAPSHOT — the existing
        point-in-time query above, listing what is still outstanding at report
        time. The event answers "how often", the snapshot "what is open right
        now"; an order that got stuck and then filled before the report exists
        only in (a). Same shape as finding 15 carrying reconcile_ghost as a
        distinct tally.
    19. Entries lost at the entry gates (R-5) — today's inference_log rows
        with event='signal_fired', grouped by gate_result: 'submitted' plus
        one bucket per gate ('freeze', 'cap_tickers', 'cap_per_ticker',
        'cooldown', 'not_tradable', 'bar_integrity', 'no_terms',
        'sizing_zero', 'funds'). A COMPLETE enumeration of the gates, not a
        sample: 'error' and 'breaker' are absent because neither is a gate
        verdict — one is a raised exception, the other a session state.
        Read straight
        from the table, NOT passed in from LiveModeRunner — unlike findings
        5 and 8, whose aggregates travel in memory only because tier_used
        and signal_source have nowhere to be stored, gate_result has a
        column of its own (db_schema.md), so the query survives a crash and
        needs no in-session tally threaded through this call.
        A candidate stopped at a gate produces no trade_log row, so these
        rows are the only record it existed; the cap buckets in particular
        are the sole evidence for whether execution.max_tickers and
        execution.max_positions_per_ticker are set sensibly, since whether a
        given candidate hit a cap depends on the concurrent-position state
        at that instant and is not reconstructible afterwards.
        Not an error signal on its own — nonzero simply means the gates bind
        in practice. The same bucket names appear as BacktestEngine summary
        counters (09_backtest_engine.md), so a live session and a backtest
        over the same window are directly comparable; 'freeze',
        'not_tradable', 'bar_integrity' and 'no_terms' are live-only, by
        design rather than omission.
        A broker rejection is NOT one of these buckets: it occurs after
        submission, so it is counted under 'submitted' here and surfaces in
        finding 17 as exit_reason='entry_rejected'.

    20. Circuit-breaker metrics (R-4) — the session's peak realised loss,
        longest run of consecutive losing exits, and peak rolling-hour entry
        count. Computed EVERY session whether or not the thresholds are
        armed: they all default to 0 (no limit), and Pilot is expected to
        calibrate them, which requires the numbers to have been accumulating
        beforehand. Same shape as findings 3/6/7/8 — the quantity is always
        measured, only the warn cutoff is deferred. In shadow mode this
        finding also carries the would-have-tripped record, which cannot be
        read from gate_result (nothing is enforced there, so gate_result
        stays 'submitted').
        Also reports current UNREALISED drawdown — sum of (current price −
        entry price) over open/exiting positions — as a separate, clearly
        labelled observation. NOT a trip input (the three thresholds above
        stay realised-only, since unrealised swings would trip on ordinary
        intraday movement — see live_mode_runner.md's Circuit Breaker); this
        exists because the realised-only design has a structural blind spot
        when an exit cannot fill (see finding 18 — a stuck exit now
        escalates at the age threshold, but a market order that still
        cannot fill has nothing left to escalate to, and the after-hours
        ladder reaches a configured ceiling, so the blind spot narrows
        rather than closing): no
        trade_log row is written, so realised loss, the consecutive counter,
        and this finding's own trip logic all stay silent regardless of how
        large the position's unrealised loss grows. This observation is
        what lets Pilot judge whether the realised-only thresholds leave
        that gap too wide, without changing what actually trips.
        On restart, a trip found via `live_session_state.breaker_tripped_at`
        (see live_mode_runner.md's Warm Restart) is reported as RESTORED,
        distinct from a fresh trip this session — the underlying cause and
        its timing are what the earlier trip already established, not a new
        event to investigate.
    21. Session-start probe results (R-7) — the measured clock offset and
        retention_boundary, one line each. TWO probes, not three: the margin
        ratio was a third until it became per-ticker, and it is now acquired
        at watchdog first listing rather than at session start (see finding
        34 and live_mode_runner.md's Per-Ticker Trading Terms). Recording the
        clock offset at session END as well catches an NTP daemon that died
        after the start gate passed, and gives a drift trend; periodic
        re-checking is unnecessary if the daemon is alive. A probe that fell
        back (retention to assumed_days) is flagged distinctly from one that
        succeeded — the fallback is safe but it silently changes gap-fill
        behaviour. There is no margin counterpart to that any more: a failed
        terms acquisition BLOCKS the ticker rather than substituting a rate,
        which is why it surfaces as its own finding rather than as a
        fell-back flag here. Each measurement also fills in its row in
        api_contract_checklist.md from ordinary operation.
        Read (R-9) from live_session_state.session_diagnostics. The probe
        values are written there immediately after the probes and BEFORE
        Health Gate 2, because clock_check's on_exceed:"abort" terminates
        ahead of Gate 2 and the measured offset is the only evidence for that
        abort — the evening liveness probe recovers the crash FACT (finding
        11) but never the measurements. clock_offset_end is stored null at
        session start and filled at shutdown, so its absence reads as "never
        reached shutdown" rather than as a missing key. A warm restart
        overwrites the probe values with the newly measured ones and
        increments restart_count: these are current operating settings, not a
        session baseline. restart_count is reported alongside, and it
        legitimately disagrees with findings 11 and 13 — all three observe
        restart/crash from different angles (11 = a crash never cleanly
        resumed, 13 = positions liquidated because of a restart,
        restart_count = how many restarts occurred), so a restart with no open
        positions gives restart_count > 0 with finding 13 at zero.
    22. inference_log rows dropped (R-8) — count of diagnostic log writes
        discarded by INSERT OR IGNORE on a PK collision, plus any suppressed
        by the write-failure guard. This is what makes INSERT OR IGNORE
        acceptable: the content of a dropped row is gone, but the fact that
        rows are being dropped is not. Nonzero after the R-8 key and
        timestamp changes indicates a real defect (duplicate logging, a clock
        moving backwards, a retry loop), not ordinary contention.
        Read (R-9) from live_session_state.session_diagnostics, refreshed on
        the bar_latency_daily flush cadence. A failure of that flush is
        swallowed and does NOT increment this counter — otherwise a persistent
        write fault would feed itself.
    23. Broker rejections without a recognised reason (R-8) — entry_rejected
        rows whose reject_reason has not been seen before. reject_reason is
        stored verbatim because the broker's vocabulary is an unverified
        contract (api_contract_checklist.md); this finding is how that
        vocabulary is actually discovered, so it feeds the checklist rather
        than signalling a fault on its own.
    24. Fill-stream staleness (account-wide WS channel) — flags when the
        REST backstop on Position Manager Loop's existing
        `position_check_interval_seconds` cadence reports fill progress for
        an in-flight order that the WS account fill stream has not yet
        delivered, sustained across more than one consecutive cycle while
        in-flight orders exist. No new freeze reason and no new polling —
        this reuses the calls already made every cycle as fill tracking's
        backstop, comparing them against `seen_fills`'s WS-derived state.
        That backstop is ACCOUNT-WIDE rather than per-order (the vendor's
        fill inquiry takes no order number), so its cost does not scale
        with the number of outstanding orders and this finding adds none. Warn severity only: correctness is unaffected (the same
        fill-accounting mechanism folds in whichever channel reports
        first), so this is a latency/observability signal, not a
        correctness one. Unrelated to Bar-Close Authority and the Feed
        Outage trigger — those judge the separate bar/price-data channel;
        this finding is scoped entirely to the account-wide fill-event
        stream and does not interact with `freeze_reasons`.
        Source (R-9): aggregated from health_events where
        finding_name='fill_stream_staleness'; detail carries order_id,
        entered_at and sustained_cycles. Written ONCE PER STALE EPISODE — on
        entry into the stale state, re-armed when it clears — so a second
        episode on the same order IS counted. Deliberately unlike finding 18's
        once-per-order rule: an order with several fills can go stale, recover
        and go stale again, and that recurrence is the signal.
    25. In-flight exit order gone at halt-clear (R-2-style resumption
        handling) — count of times a ticker's halted→tradable transition
        (Position Manager Loop Step 1) found an in-flight exit order for
        that ticker had disappeared on immediate re-query, meaning the
        broker canceled it during the halt. Distinct from finding 18: that
        one is duration-based (still open past an age threshold); this one
        is event-based (gone at the specific instant a halt clears) and
        fires regardless of how long the halt lasted. Not an error signal
        on its own — the disappearance triggers automatic immediate
        resubmission (see live_mode_runner.md), so a nonzero count reports
        that the safety-net path ran, not that anything went unhandled.
        Self-measuring for api_contract_checklist.md's T-12: a nonzero
        count is direct evidence the broker does NOT always preserve a
        resting order through a halt.
        Source (R-9): aggregated from health_events where
        finding_name='inflight_exit_gone_at_halt_clear'; detail carries ticker
        and order_id.
    26. Bar-arrival latency (`bar_arrival_latency`, T-13) — read from
        `bar_latency_daily` (db_schema.md), not passed in: same reasoning as
        finding 19, the values have a table of their own. Severity always
        'ok' — this is calibration data, not a fault signal.
        Reported as a CUMULATIVE curve over whole-second buckets ("share of
        bars ready within X seconds"), never as a density histogram: each
        sample is an upper bound on the vendor's true latency, so the curve
        is a conservative floor on the truth, while reading a single bucket
        as "these took exactly X" would claim more than the data supports.
        Detail carries TODAY's curve AND the all-history curve. Today alone
        cannot recalibrate `bar_close_grace_seconds` (that needs months);
        history alone would dilute a single bad day into invisibility.
        Also reports, for the day: sample count, the two error classes
        separately (`poll_continuous` vs `wide_error` — if their curves
        diverge, admitting wide-error samples is skewing the result and
        `live_mode.bar_latency_max_error_seconds` should be tightened),
        rejected-sample counts, `bar_gap_samples`, and Bar-Close Authority's
        own `missed` count. The last one is not decoration: this finding sees
        only bars that ARRIVED, and a curve that looks excellent because the
        slow bars never showed up at all would otherwise read as good news.
    27. Bar-close premise violation (`bar_close_premise_violation`) — a bar
        observed for a minute that has not finished (negative `d`; see
        live_mode_runner.md's Bar-Arrival Latency Measurement). Severity
        'abort', event-driven. Detail carries the `|d|` distribution — near
        60s means the vendor timestamps bars by CLOSE where the spec assumes
        OPEN, small and scattered means premature publication or a clock
        fault — plus the per-event rows from `health_events` and the current
        `bar_integrity` exclusion list.
        Kept SEPARATE from finding 26 rather than folded in as an anomaly
        count, per this file's standing no-merge rule: 26 answers "what
        should grace be", 27 answers "is the model of the vendor wrong at
        all", and the responses are recalibration versus redesign.
        What this finding does NOT do is stop anything. Entries for the
        affected tickers are blocked by live_mode_runner.md's own gate, and
        past the promotion threshold entry submission freezes session-wide —
        but the session continues, so a timestamp-convention mismatch means
        the session keeps judging `expected_minute` an hour off, and a
        premature-bar fault means `infer()` keeps running on incremental
        indicator state that absorbed a partial bar. That is a knowingly
        accepted exposure, recorded here rather than in a session doc so it
        stays attached to the signal that detects it.
    28. Evening job deadline abort (`evening_job_deadline_abort`) — the
        evening batch gave up waiting for the DuckDB write lock at
        `evening_wait_hard_deadline` and aborted (metadata_crawler.md's
        evening job start gate). Severity 'abort'. Needs neither the DB nor
        any in-memory aggregate, which is the point: it fires precisely
        because the DB could not be opened.
        Named for the observed event, not a presumed cause — whether the
        holder is a live runner still working or a hung process is exactly
        what is unknown at this moment, and is what the manual intervention
        it requests has to determine.
        Consequence worth stating: tomorrow's Health Gate 1a will abort
        tomorrow's session on missing session stats. That is the intended
        fail-safe, reached loudly.

    29. Watchdog working set empty during the regular session
        (`watchdog_empty_working_set`) — `watchdog.get_candidates()`
        returned an empty list inside the regular session. The watchdog's
        filter conditions make an all-day zero impossible, so emptiness is
        a fault signal rather than a quiet market. Scoped to the regular
        session because the working set carries over already full from
        premarket, which is why no grace-window config key exists: 09:30
        is the boundary.
        Observation only — no freeze and no Feed Outage trigger. A dead
        watchdog costs entries, not correctness, and a false freeze over an
        open position costs more than a session that trades nothing.
        Level: warn. Correctness is unaffected and no position is touched,
        so not critical; but the state costs a whole session's entries, so
        not info either.
        Source (R-9): aggregated from health_events where
        finding_name='watchdog_empty_working_set'. The working set is
        persisted nowhere, so nothing at report time could reconstruct it.
        Written once on entry into each empty episode and re-armed when the
        list refills, the same discipline as finding 24 — and for the same
        reason, that a watchdog recovering and failing again is itself the
        signal.

    30. Scan carryover rate (`scan_carryover_rate`) — the share of cycles in
        which prefix-scan depth D exceeded `live_mode.scan.head_slots`,
        from `live_scan_daily.carryover_cycles` against the session's cycle
        count, with the `depth_hist_*` buckets carried in the detail. It
        answers one question only: is N large enough. A rising rate means
        firings are arriving faster than the head slots absorb them, and the
        response is N, not a threshold edit — but N is bounded by RTT, not
        by budget (live_mode_runner.md's slot allocation), so a rate that
        cannot be fixed by raising N is the signal that the working set or
        the watchdog's firing behaviour has changed shape.
        Level: warn. Carryover costs detection latency, not correctness.
        Threshold TBD — no baseline distribution exists yet, the same
        deferral status as findings 6/7.
    31. Rotation miss rate (`scan_rotation_miss_rate`) —
        `rotation_misses / rotation_visits` from `live_scan_daily`. This is
        the accepted watchdog-gap risk (`api_contract_checklist.md` T-17)
        actually realising: the rotation cursor found a ticker crossing the
        relaxed expression that the watchdog had not raised. A nonzero rate
        is EXPECTED, which is why this is a rate finding rather than an
        event one — what matters is movement, not presence.
        Level: warn. Threshold TBD, same deferral.
        Read alongside finding 33: this one sees only tickers the watchdog
        listed at some point, while 33 sees the ones it never listed at all.
    32. Bar-close fetch deadline (`barclose_fetch_deadline`) —
        `live_scan_daily.barclose_fetch_ms_p95` against the 5-second
        decision deadline the scan design targets. LOAD-BEARING: the entire
        mid-bar screen exists to shrink the bar-close burst from the working
        set to the candidate superset, and until this table existed nothing
        measured whether it succeeded. The detail carries
        `superset_k_p50`/`_p95` beside it, because a breached deadline
        caused by an oversized superset and one caused by vendor latency
        need opposite responses.
        Level: warn, escalating to abort is deliberately NOT provided — a
        missed deadline degrades entry timing, and freezing a session over
        it would cost more than the late entries it prevents.
    33. Detection gap (`detection_gap`) — `gap_total`, `gap_disagreed` and
        `gap_unevaluated` from the evening `evening_detection_gap` stage
        (metadata_crawler.md), reported as rates against the day's detected
        set. The two components are NOT summed and NOT compared to each
        other: `gap_disagreed` means the ticker was evaluated live and the
        bars later disagreed, which is a feed-quality signal in the same
        family as `feed_coverage_daily`, while `gap_unevaluated` means the
        scan never reached it at all. One calls for looking at the feed, the
        other at the scan.
        Also note the overlap the detail must state rather than resolve: a
        promoted ticker that produced no late entry appears in BOTH
        `gap_unevaluated` and `promotions_total` (see db_schema.md), so the
        two are never added.
        Level: warn. Threshold TBD, same deferral. Available only on the
        evening path, one day in arrears — the only finding here whose
        input is produced by a batch stage rather than by the session it
        describes.

    34. Per-ticker terms-acquisition failure
        (`ticker_terms_acquisition_failure`) — tickers blocked from entry
        for want of a `live_ticker_terms` row, against watchdog
        first-listing attempts. Read (R-9) from
        `live_session_state.session_diagnostics`, where LiveModeRunner
        tallies it as the session runs and flushes on the
        `bar_latency_daily` cadence; NOT recomputed here. Same class and
        same mechanics as finding 8 — an INFRASTRUCTURE-HEALTH signal,
        distinct in kind from strategy underperformance, so it is never
        read alongside the winning-rate findings.
        Detail view: the BLOCKED TICKERS BY NAME, on finding 2's pattern.
        A count alone does not let an operator act, and the block persists
        for the session once a ticker is given up (live_mode_runner.md's
        Per-Ticker Trading Terms), so identity is the actionable part.
        This finding is what makes the no-fallback design survivable:
        acquisition failure blocks the ticker rather than substituting a
        rate, so silent trading stoppage is that choice's principal risk
        and this is the instrument that shows it. It is also what measures
        the assumption the choice rests on — that vendor-response failure
        is rare — and a high rate here is the ground for revisiting it.
        Level: warn. Threshold TBD, same deferral as findings 3/6/7/8: the
        rate is always computed and loggable, only the warn cutoff is
        undecided.

    Returns: dict of {finding_name: {severity: 'ok'|'warn'|'abort', detail: ...}}
    """
    ...
```

---

## DB Health Observation (R-9)

A read-only summary emitted once at each FIRST-DB-ACCESS point: live session
start, evening batch start, and premarket batch start. It reports:

- the `data/market.duckdb` file size
- row counts for the fifteen purge-registry tables (see db_schema.md) and for
  the structurally-excluded corpus tables
- the latest `date` value present in each

**This is not a finding and not a Health Gate.** It is not collected by
`gather_findings()`, has no severity, no threshold, and no warn cutoff, and
it never blocks a session start — adding a new failure point to the
premarket path to report disk usage would be a poor trade. It is pure
observation, logged and printed.

Its purpose is to make the purge registry's `retention_days: inf` defaults
resolvable. Every registry entry starts at `inf` precisely because no growth
rate has ever been measured; this is what measures them, so an operator
eventually sets a window against data instead of a guess. `health_events` is
the entry most worth watching — R-9 widened its writers from one finding to
six and finding 29 has since made seven, and findings 18 and 24 both emit
repeatedly during a broker-latency episode.

**Feed coverage is reported the same way**, and for the same reason. When
`auxiliary_stream.enabled` is true, the evening batch's coverage stage writes
`feed_coverage_daily`, and this report carries the day's figures: total
coverage ratio, the per-bucket dropout curve, and the two price-extreme
counts. Like the observation above it is **not a finding** — not collected by
`gather_findings()`, no severity, no threshold, no warn cutoff, never
blocking. There is nothing to threshold against yet, which is the point: the
2-print guard's constants stand exactly where `retention_days: inf` stands
above, and this is what lets an operator set them against data instead of a
guess. It is config-gated because it serves a characterisation that ends,
rather than a standing operational need.

**Delivered on its own alert stream**, `alert_key: 'db_health_observation'`,
so it lands in `alert_log` like any other delivery and its remote arrival is
verifiable by querying `alert_log.outcome` rather than by asking someone
whether the mail showed up. Because suppression is per `(alert_key,
channel)`, this stream can only ever displace itself — it cannot mask a
session-start or breaker-trip alert, and those cannot mask it. It fires at
most three times a day, so `rate_limited` is not expected to engage.

---

## Log Writing (always, regardless of delivery)

```python
def write_to_log(findings: dict, log_path: Path) -> None:
    """
    Findings are always written to the local log file first, independent of
    whether any alert channel delivery succeeds below. Delivery and logging
    are deliberately independent — logging is not a "fallback" for a failed
    send, it always happens.

    A failure of THIS function (disk full, permissions) is caught and never
    propagated — a diagnostic write must not abort a trading session, the
    same rule db_schema.md states for diagnostic DB writes. It cannot be
    recorded in the log it just failed to write, so instead a synthetic
    `log_write_failed` finding (severity 'abort') is injected into the
    delivery payload: the two paths back each other up, and since they are
    independent by design a dead log still reaches the operator by alert. It
    carries its own alert_key so suppression can never hide it. If logging
    AND every channel fail together there is nothing left but stderr — an
    accepted terminal case, not a handled one.

    Called on a subset invocation too, where it records a PARTIAL set. Such
    an entry does not substitute for the scheduled whole-report write, and
    the log line is marked so a reader cannot mistake one for the other.
    """
    ...
```

---

## Alert Delivery (best-effort)

```python
def run_health_report(db_conn, today_date, log_dir, config,
                      findings_subset: list[str] | None = None,
                      suppressible: bool = False,
                      alert_key: str | None = None,
                      live_aggregates: dict | None = None) -> None:
    """See Invocation Contract below for the seven call paths and the
    argument each one passes. `alert_key` is required when suppressible."""
    findings = gather_findings(db_conn, today_date, log_dir,
                               subset=findings_subset,
                               live_aggregates=live_aggregates)
    write_to_log(findings, log_dir / f"health_report_{today_date}.log")

    if config["alerting"]["alert_level"] == "abort_only":
        send_findings = {k: v for k, v in findings.items() if v["severity"] == "abort"}
    elif config["alerting"]["alert_level"] == "warn_and_above":
        send_findings = {k: v for k, v in findings.items() if v["severity"] in ("warn", "abort")}
    else:  # "all" (default)
        send_findings = findings

    # Report severity = MAX over the findings it carries. A report is one
    # delivery, so it needs one grade; the suppression mode below is keyed on
    # it, and it is re-derived per call rather than fixed per path (the same
    # path yields different grades on different days).
    severity = max_severity(send_findings)
    mode = config["alerting"]["event_alert_suppression"][severity]["mode"] \
        if suppressible else "always"

    for channel in config["alerting"]["channels"]:
        # alert_level first, suppression second — the reverse order would let
        # a report that was never going to be sent consume the rate-limit
        # window and suppress the next one that would have been.
        enqueue(channel, alert_key, severity, send_findings, mode, config)
```

`enqueue()` returns once the `alert_log` row is written (db_schema.md) with
`dispatched_at` set and `outcome` NULL, or immediately with
`outcome='suppressed'`. **Delivery itself is asynchronous** — a background
worker per channel performs the actual send and fills in `completed_at` /
`outcome`. `run_health_report()` therefore returns as soon as logging is
done and never blocks its caller on a socket.

Recording `completed_at` on SUCCESS, not only on failure, is deliberate: the
pair with `dispatched_at` gives every send its own duration, per channel, and
that distribution is the evidence `drain_timeout_seconds` below will
eventually be re-tuned against. Its current value is a placeholder with
nothing measured behind it.

This is not optional polish. Finding 27 fires from inside the
`poll_interval_seconds` loop — the first event-driven trigger that does — and
both channels carry a 10s timeout, so a synchronous send would stall the
polling loop by up to 20s per detection. Under a timestamp-convention
mismatch, firing dozens of times a minute, that would halt the loop
entirely, inflate Bar-Close Authority's `missed` set, and trip Feed Outage
Recovery: the diagnostic destroying what it diagnoses. Threads rather than
asyncio, because the evening batch has no event loop.
Logging stays synchronous. The asynchrony is scoped to `send()` alone,
preserving "logging always happens, first".

**Suppression** (`mode`, per severity; `event_alert_timeout_seconds`, per
channel) applies ONLY to suppressible paths:
- `always` — no suppression. Debug/escape-hatch setting; also the default
  for `warn`/`ok`, which are currently unreachable here because every
  event-driven path is `abort` today. Provisioned for the case finding 27 is
  later downgraded while keeping its trigger.
- `once_per_session` — first occurrence only. Runtime state, like
  `last_halt_state`: a warm restart may cost one extra alert, which is not a
  correctness defect.
- `rate_limited` (default) — at most one send per `(alert_key, channel)` per
  `event_alert_timeout_seconds`, measured from the LAST SEND. Explicitly NOT
  a timer reset by each event: that variant goes permanently silent exactly
  when events are continuous, i.e. in the worst case. As specified, a
  sporadic fault alerts sporadically and a total fault alerts at the cap,
  so the rhythm of the alerts itself conveys severity — which is why no
  separate escalation-on-count rule is needed.

Every send carries the count suppressed on that `(alert_key, channel)` since
its last send. That is what makes suppression accountable without a separate
summary path — and the count legitimately DIFFERS between channels for one
alert_key, since each channel has its own timeout. Suppression is keyed on
the PATH, not the finding, because an invocation delivers a report.

Finding 27 therefore holds ONE key for the whole session, not one per ticker.
Per-ticker keys would multiply with the candidate count, exhaust
`max_pending_alert_keys` at once, and slice the rate limit so finely that it
would stop limiting anything — under the very fault where limiting matters
most. Which tickers are affected is carried in the finding's own detail.

Suppression state — last-send timestamps and the once-per-session flags — is
runtime only for every mode. A restart may cost one extra alert, which is
not a correctness defect, and this introduces no new persisted state.

Queue: per channel, so a dead email channel's timeouts cannot delay Discord
— the same independence the per-channel best-effort rule already asserts,
extended to time. Cap `max_pending_alert_keys` per channel, holding only the
LATEST pending entry per `alert_key`. Dropping the oldest instead would let
finding 27's stream evict a breaker trip or a session-end report; keyed this
way, a noisy path can only ever displace itself. Displaced and dropped
entries are recorded (`outcome='dropped_queue'`), not merely counted.

### Shutdown Order and Drain

At session end, four requirements fix the order: finding 26 reads
`bar_latency_daily`, findings 30-32 read `live_scan_daily`,
`gather_findings()` needs the DB, and the evening batch needs the write lock
released.
```
1. final flush of bar_latency_daily      (else finding 26 loses the last
                                          un-flushed minute)
1a. LiveModeRunner writes live_scan_daily (else findings 30-32 read a table
                                          this session never wrote — same
                                          layer as step 1, and for the same
                                          reason. gap_* are NOT written here:
                                          the evening detection-gap stage
                                          writes those hours later, which is
                                          why finding 33 reports the previous
                                          day on a session-end call)
1b. LiveModeRunner closes any OPEN live_halt_episodes interval, same layer
                                         (a clean shutdown observed the
                                          session's end, so leaving an
                                          interval open would assert an
                                          unresolved halt that nothing will
                                          resolve. After a CRASH the row
                                          legitimately stays open — no party
                                          observed a resumption — and a warm
                                          restart resolves it by replay
                                          instead. exit_trigger_agreement_daily
                                          needs no step here: it is written per
                                          settled exit, not at shutdown)
2. gather_findings + write_to_log
3. dispatch delivery                     (background)
4. drain, up to alerting.drain_timeout_seconds
5. flip live_session_end / live_session_start markers to 'success'
6. close the DB connection
```
Draining BEFORE closing the DB is what lets the background workers write
`completed_at` / `outcome` — including `outcome='dropped_drain'` for anything
still pending at the timeout. Nothing about delivery is therefore invisible.

The markers come AFTER the drain because the evening job's start gate treats
"marker present" as "lock released" (metadata_crawler.md). Writing them
before draining would leave that invariant false for up to the drain timeout.
Delaying them costs nothing: a crash leaves no marker at all, which is what
crash detection keys on.

Timing: the session process runs to `live_mode.session_hard_exit_time`
(default 20:00 ET — see live_mode_runner.md's Session Shutdown) and the
evening batch starts at 21:00, so a 600s drain leaves roughly 50 minutes'
margin. That margin is why `drain_timeout_seconds` cannot simply be raised.
The two values are coupled: raising `session_hard_exit_time` eats the same
margin as raising the drain, and the ordinary exit is earlier still (all
positions flat past `execution.session_close_exit_time`), so the 50 minutes
is a floor rather than the typical case.

Steps 2-4 failing does NOT stop 5-6. Holding the DB open on an alerting
failure would make the evening batch wait to its own deadline and abort,
costing two days of operation over an undelivered message.

Path (6)'s drain uses `abort_drain_timeout_seconds` instead: it has already
asked for manual intervention, so a process lingering an hour afterwards is
an operational trap.

### Discord

```python
import requests
from datetime import datetime, timezone

def send_discord_alert(webhook_url: str, title: str, severity: str, fields: dict) -> bool:
    """severity: 'ok' | 'warn' | 'abort'. Returns False on any failure —
    caller logs the failure per run_health_report() above."""
    color = {'ok': 0x2ECC71, 'warn': 0xF1C40F, 'abort': 0xE74C3C}[severity]
    embed = {
        "title": title,
        "color": color,
        "fields": [{"name": k, "value": str(v), "inline": True} for k, v in fields.items()],
        "timestamp": datetime.now(timezone.utc).isoformat(),
    }
    try:
        resp = requests.post(webhook_url, json={"embeds": [embed]}, timeout=10)
        return resp.status_code in (200, 204)
    except requests.RequestException:
        return False
```

### Email

```python
import smtplib
from email.mime.text import MIMEText

def send_email_alert(smtp_config: dict, title: str, severity: str, fields: dict) -> bool:
    body = "\n".join(f"{k}: {v}" for k, v in fields.items())
    msg = MIMEText(body, "plain")
    msg["Subject"] = f"[{severity.upper()}] {title}"
    msg["From"] = smtp_config["from_addr"]
    msg["To"] = smtp_config["to_addr"]
    try:
        with smtplib.SMTP(smtp_config["host"], smtp_config["port"], timeout=10) as server:
            server.starttls()
            server.login(smtp_config["username"], smtp_config["password"])
            server.send_message(msg)
        return True
    except (smtplib.SMTPException, OSError):
        return False
```

---

## Invocation Contract

Every call path, and what it passes. This table exists because the two
evening-job paths were previously documented only in metadata_crawler.md and
were absent from this file's own trigger list — the information was split
across three files and drifted.

| Path | `alert_key` | `findings_subset` | `suppressible` | DB | `log_dir` | `live_aggregates` |
|---|---|---|---|---|---|---|
| Scheduled, session start | `session_start` | None | False | yes | yes | — |
| Scheduled, session end | `session_end` | None | False | yes | yes | — |
| Breaker trip (R-4) | `breaker_trip` | None | False | yes | yes | — |
| `clock_check` abort (R-8) | `clock_check_abort` | None | False | yes | yes | — |
| Negative bar-latency sample | `bar_close_premise_violation` | finding 27 | True | yes | yes | — |
| Evening liveness probe | `evening_liveness_probe` | DB-only findings | False | yes | yes | — |
| Evening deadline abort | `evening_deadline_abort` | finding 28 | False | **no** | yes | — |

`log_write_failed` is an eighth `alert_key` but not a path — it is injected
into whichever delivery is in flight (see Log Writing) and is never
suppressed. `db_health_observation` (R-9) is a ninth: it is a delivery
stream, not a findings-gathering path, so it appears in `alert_log` and in
the suppression key space but has no row above — it passes no
`findings_subset` because it collects no findings at all (see DB Health
Observation).

**Requirement axes — three, not two.** Beyond the DB and `live_aggregates`,
`gather_findings()` also reads the crawler log FILES under `log_dir`, which
is neither. All three axes had been invisible until now because every call
gathered everything from inside a session, where all three were always
present.
- **`live_aggregates`**: EMPTY as of R-9 — no finding is computed this way
  any more. Findings 5, 8, 21 and 22 now read
  live_session_state.session_diagnostics, and 12, 24 and 25 now aggregate
  health_events, so all seven moved to the DB axis and none is lost when a
  session crashes. The axis and its column above are KEPT rather than
  deleted: it is a real requirement a future finding could have, and
  retaining it means the classification does not have to be rebuilt then.
  Every path's entry is `—` for now, including `clock_check_abort`, whose
  former `partial` described exactly the case R-9 removed — an abort before
  Gate 2 now leaves its probe measurements in the DB rather than in memory.
- **`log_dir` files**: findings 4, 10, and the missing-`SUMMARY` finding.
  Readable without the DB, which is why path (7) can produce anything at all.
- **DB**: findings 1, 2, 3, 5, 8, 9, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21,
  22, 23, 24, 25, 26, 27, 29, 30, 31, 32, 33. The last four read
  `live_scan_daily`; 33 alone reads a row written by the evening batch
  rather than by the session being reported, so on a session-end call it
  describes the PREVIOUS day.
- **Neither**: findings 6, 7 (placeholders) and 28.
- Finding 11 is produced BY the evening liveness probe rather than gathered
  during it — it is that path's own conclusion, not a query result.

**Path (6)'s subset is deliberately wider than finding 11.** That path exists
because the session crashed, which means its scheduled session-end call never
happened and its findings were never alerted anywhere — and the next
session-start call reads a different date. Sweeping every DB-computable
finding recovers the ones that survived in tables (12's orphans are
rediscovered by tomorrow's reconcile anyway, but 13, 17, 18 and 19 would
simply never be seen). As of R-9 this recovery is COMPLETE rather than
partial: findings 5, 8, 12, 21, 22, 24 and 25 were the ones it could not
reach, and all seven are now DB-computable, so a crashed session loses only
what the flush cadence had not yet written — at most one flush interval of
the running counters (findings 8 and 22).
If `live_session_start` has no row for the date, the session never started —
an ordinary non-trading outcome, not a crash — and the subset narrows to
finding 1. Reporting the full sweep there would emit an all-zero report that
reads like an incident.

**`today_date`** is the trading day the CALLING PROCESS started on, fixed at
start and never re-derived — the same rule metadata_crawler.md's Constraints
state for the evening batch, which matters because that batch can now cross
midnight while waiting on the lock.

**Errors, not silent skips.** An unknown name in `findings_subset`, a subset
naming a DB finding in a no-DB context, or — should the axis ever regain a
member — one naming a `live_aggregates` finding with none supplied, all
raise. Skipping quietly would turn a typo into an empty alert that looks like
good news.

**Absent is not zero (R-9).** Findings 5, 8, 21 and 22 read
`live_session_state.session_diagnostics`, and that table is LIVE-ONLY: no row
exists for a date no live session ran, and none ever exists on the
backtest side. Where the row is absent these four report NOT APPLICABLE —
never an error, and never 0. Both alternatives are wrong in the same
direction: raising would make an ordinary backtest invocation fail, and 0
would assert a measurement that was never taken ("no halt checks fell back to
the tick-rate heuristic" reads as healthy when in fact nothing was checked).
Same boundary handling as the six `gate_result` values that get no backtest
counter (see db_schema.md). That set is not the same as finding 19's
live-only three: it also holds 'breaker', which backtest evaluates without
enforcing, and 'error', which it cannot raise at all. This applies per finding, not per
call: a live session that crashed before its first flush has a row with the
counters absent, and those specific keys report not-applicable while the
probe values in the same row report normally.

---

## Config Keys (pipeline_config.yaml)

```yaml
alerting:
  alert_level: "all"                       # "all" | "warn_and_above" | "abort_only"
  channels: ["discord", "email"]           # empty list = log-only, no delivery attempted
  event_alert_suppression:                 # event-driven, single-report
    abort: { mode: "rate_limited" }        #   deliveries only — scheduled
    warn:  { mode: "always" }              #   batch reports are never
    ok:    { mode: "always" }              #   suppressed (see Constraints)
  max_pending_alert_keys: 10               # per channel; distinct alert_keys,
                                           # latest entry per key retained
  drain_timeout_seconds: 600               # normal shutdown: wait for pending
                                           # deliveries. Bounded by the ~50min
                                           # gap to the 21:00 evening batch
  abort_drain_timeout_seconds: 600         # abort shutdown (evening job
                                           # deadline abort) — manual
                                           # intervention is already pending,
                                           # so do not linger
  discord:
    webhook_url_env: "DISCORD_WEBHOOK_URL" # env var name, not the value — never in config
    event_alert_timeout_seconds: 10        # rate_limited window for this
                                           # channel; webhooks absorb a high
                                           # rate
  email:
    event_alert_timeout_seconds: 60        # SMTP cannot absorb Discord's rate;
                                           # each send still carries its own
                                           # suppressed count, so the coarser
                                           # window loses no information
    smtp_host: "smtp.gmail.com"
    smtp_port: 587
    from_addr: ""
    to_addr: ""
    username_env: "ALERT_EMAIL_USERNAME"
    password_env: "ALERT_EMAIL_PASSWORD"
```

---

## Constraints

- Severity is a DELIVERY GRADE and implies no operational response. Nothing
  stops, freezes, or aborts because a finding is `abort`; where a stop does
  happen (`clock_check`'s session abort, a breaker trip's entry freeze, the
  evening job's own abort) the owning module decides it and this tool merely
  reports it — the causation runs that way, never the reverse. Until
  findings 27/28 every `abort` finding happened to coincide with something
  stopping; that coincidence is over, and each finding's operational
  consequence, if any, is stated in its own entry above
- Call paths, their arguments, and which findings need the DB / `log_dir` /
  `live_aggregates` are specified in Invocation Contract above — not
  restated per finding, so there is one place to keep correct
- `event_alert_timeout_seconds` is per channel and has NO default. A channel
  added later must declare its own; no fallback is defined, deliberately, so
  that the decision is forced rather than silently inheriting a value sized
  for a different transport. No further channels are planned
- Suppression applies ONLY to event-driven, single-report deliveries.
  Scheduled invocations send every finding in one batched message, which
  suppression's per-`alert_key` model does not address, and at two calls a
  session they could not reach a rate limit anyway. This is a consequence of
  the DELIVERY UNIT, not of severity — which is why the `warn`/`ok` entries
  in `event_alert_suppression` are inert today rather than wrong
- `write_to_log()` always runs before any delivery attempt; delivery success
  or failure never affects whether findings are logged
- Delivery is best-effort per channel — a failure on one configured channel
  does not block attempting the others
- Delivery failure itself is logged (as `event=alert_delivery_failed`) so a
  silently-dead alert channel is eventually discoverable from the log alone
- Webhook URLs and SMTP credentials are never stored in `pipeline_config.yaml`
  directly — only the environment-variable name they're read from
  (`*_env` keys), consistent with `secrets.yaml`'s gitignore discipline in
  metadata_crawler.md
- `channels: []` (empty) means log-only — a valid, supported configuration,
  not an error state
- A `SUMMARY` line's absence from a crawler log (see metadata_crawler.md) is
  reported as its own finding, distinct from a nonzero `failed` count within
  a `SUMMARY` line that does exist
- Live winning-rate divergence (finding 6) and execution-parameter divergence
  (finding 7) are both placeholders in `gather_findings()` until their
  respective methodology/threshold is defined (see the shadow/retraining
  spec doc) — `health_report.py` must not compute or report either finding
  before then. They remain two separate findings even once defined — never
  merged into a single "things seem off" signal.
- Finding 8 (halt-check signal-source rate) is unlike findings 6/7: it has
  no accumulation gate and is always computable from the current session's
  halt checks alone — only its warn threshold is undecided. It must not be
  merged with finding 6 or 7 either, for the same reason those two stay
  separate from each other — a degraded halt-status endpoint, a model
  drift, and an execution-fill drift are three distinct failure modes with
  three distinct responses.
- Date-scoped queries over `trade_log` use `exit_date`, not `date`, wherever
  the question is "what did this session close". `date` is the ENTRY date,
  so a position carried across the boundary — dead position Case A, and
  live's `overnight_exit` — would otherwise be attributed to the day it was
  opened and vanish from the report for the day it was actually liquidated
  (see db_schema.md's `trade_log.exit_date`). Entry-side questions still
  read `date`
- Findings 17 and 18 are session-scoped and always computable — no pilot
  accumulation gate, unlike findings 6/7/9. Finding 18's age threshold is
  the only TBD among them
- Finding 9 (execution-parameter fit rejections) has nothing to report
  before Pilot stage produces its first fit_execution_params() run, but
  unlike findings 6/7 has no further threshold to defer once Pilot begins
  — rejection is a boolean per parameter per week against
  `fit_rejection_multiplier` (already a concrete config value, not TBD),
  not a magnitude needing a warn cutoff decided later.
