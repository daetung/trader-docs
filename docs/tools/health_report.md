# Tool: Health Report

**File:** `tools/health_report.py`
**Standalone CLI tool — run at LiveModeRunner session start and session end**

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

EVENT-DRIVEN invocations fire outside that schedule, always at `abort`
severity so delivery happens even under `alert_level: "abort_only"`. Two
triggers exist:
- A circuit-breaker trip (R-4) — entries are frozen for the rest of the
  session at that point, worth knowing immediately rather than at session
  end.
- A `clock_check` abort (R-8) at session start — this one fires BEFORE the
  scheduled session-start invocation above (the clock fault stops the
  session before it opens), so without it a clock-fault day and an
  ordinary quiet day would be indistinguishable to the operator from
  outside.
Neither needs new infrastructure — the Discord and email channels below
already existed and were simply never wired to a mid-session or
pre-session event.

---

## Findings Gathered

```python
def gather_findings(db_conn, today_date, log_dir) -> dict:
    """
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
       for SUMMARY lines only (see metadata_crawler.md's Output/Logging).
       A stage with a batch_runs 'success'/'failed' row but no matching
       SUMMARY line is flagged as a logging-mismatch finding on its own —
       distinct from the batch's own status.
    5. tick_bar_aggregates Tier-2/3 fallback rate — read directly from
       LiveModeRunner's Health Gate 2 result for the current session
       (passed in, not recomputed here — see live_mode_runner.md).
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
       (see live_mode_runner.md's Step 1a) tagged
       signal_source='tick_rate_fallback' vs. 'api'. Not recomputed here —
       LiveModeRunner tallies signal_source per check as the session runs
       and passes the aggregate in at each of health_report.py's two daily
       invocations, the same way Health Gate 2's tier_used tally already
       reaches finding 5 (see live_mode_runner.md's Health Gate 2 — this is
       the same pattern, a second aggregate threaded through the same
       existing call). Unlike findings 6/7, this is NOT a placeholder — no
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
        Manager Loop Step 1a).
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
        live_mode_runner.md's In-flight order tracking escalates the order
        to market as a final backstop (a no-op if already market). This
        finding still fires independently of that escalation — it reports
        exposure duration, not whether an escalation attempt has occurred,
        so a row here may be pre- or post-escalation depending on whether
        the market order has filled by report time.
    19. Entries lost at the entry gates (R-5) — today's inference_log rows
        with event='signal_fired', grouped by gate_result: 'submitted' plus
        one bucket per gate ('freeze', 'cap_tickers', 'cap_per_ticker',
        'cooldown', 'not_tradable', 'sizing_zero', 'funds'). Read straight
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
        over the same window are directly comparable; 'freeze' and
        'not_tradable' are live-only, by design rather than omission.
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
        when an exit cannot fill (no give-up timeout — see finding 18): no
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
    21. Session-start probe results (R-7/R-8) — the measured clock offset,
        margin_ratio, and retention_boundary, one line each. Recording the
        clock offset at session END as well catches an NTP daemon that died
        after the start gate passed, and gives a drift trend; periodic
        re-checking is unnecessary if the daemon is alive. A probe that fell
        back (margin_ratio to live_mode.margin_ratio_fallback, retention to
        assumed_days) is flagged distinctly from one that succeeded — the
        fallbacks are safe but they silently change sizing and gap-fill
        behaviour. Each measurement also fills in its row in
        api_contract_checklist.md from ordinary operation.
    22. inference_log rows dropped (R-8) — count of diagnostic log writes
        discarded by INSERT OR IGNORE on a PK collision, plus any suppressed
        by the write-failure guard. This is what makes INSERT OR IGNORE
        acceptable: the content of a dropped row is gone, but the fact that
        rows are being dropped is not. Nonzero after the R-8 key and
        timestamp changes indicates a real defect (duplicate logging, a clock
        moving backwards, a retry loop), not ordinary contention.
    23. Broker rejections without a recognised reason (R-8) — entry_rejected
        rows whose reject_reason has not been seen before. reject_reason is
        stored verbatim because the broker's vocabulary is an unverified
        contract (api_contract_checklist.md); this finding is how that
        vocabulary is actually discovered, so it feeds the checklist rather
        than signalling a fault on its own.
    24. Fill-stream staleness (account-wide WS channel) — flags when the
        REST backstop poll on Position Manager Loop's existing
        `position_check_interval_seconds` cadence reports fill progress for
        an in-flight order that the WS account fill stream has not yet
        delivered, sustained across more than one consecutive cycle while
        in-flight orders exist. No new freeze reason and no new polling —
        this reuses the REST call already made every cycle as fill
        tracking's backstop, comparing it against `seen_fills`'s WS-derived
        state. Warn severity only: correctness is unaffected (the same
        fill-accounting mechanism folds in whichever channel reports
        first), so this is a latency/observability signal, not a
        correctness one. Unrelated to Bar-Close Authority and the Feed
        Outage trigger — those judge the separate bar/price-data channel;
        this finding is scoped entirely to the account-wide fill-event
        stream and does not interact with `freeze_reasons`.
    25. In-flight exit order gone at halt-clear (R-2-style resumption
        handling) — count of times a ticker's halted→tradable transition
        (Position Manager Loop Step 1a) found an in-flight exit order for
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

    Returns: dict of {finding_name: {severity: 'ok'|'warn'|'abort', detail: ...}}
    """
    ...
```

---

## Log Writing (always, regardless of delivery)

```python
def write_to_log(findings: dict, log_path: Path) -> None:
    """
    Findings are always written to the local log file first, independent of
    whether any alert channel delivery succeeds below. Delivery and logging
    are deliberately independent — logging is not a "fallback" for a failed
    send, it always happens.
    """
    ...
```

---

## Alert Delivery (best-effort)

```python
def run_health_report(db_conn, today_date, log_dir, config) -> None:
    findings = gather_findings(db_conn, today_date, log_dir)
    write_to_log(findings, log_dir / f"health_report_{today_date}.log")

    if config["alerting"]["alert_level"] == "abort_only":
        send_findings = {k: v for k, v in findings.items() if v["severity"] == "abort"}
    elif config["alerting"]["alert_level"] == "warn_and_above":
        send_findings = {k: v for k, v in findings.items() if v["severity"] in ("warn", "abort")}
    else:  # "all" (default)
        send_findings = findings

    for channel in config["alerting"]["channels"]:
        ok = send(channel, send_findings, config)
        if not ok:
            write_to_log({"event": "alert_delivery_failed", "channel": channel},
                         log_dir / f"health_report_{today_date}.log")
```

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

## Config Keys (pipeline_config.yaml)

```yaml
alerting:
  alert_level: "all"                       # "all" | "warn_and_above" | "abort_only"
  channels: ["discord", "email"]           # empty list = log-only, no delivery attempted
  discord:
    webhook_url_env: "DISCORD_WEBHOOK_URL" # env var name, not the value — never in config
  email:
    smtp_host: "smtp.gmail.com"
    smtp_port: 587
    from_addr: ""
    to_addr: ""
    username_env: "ALERT_EMAIL_USERNAME"
    password_env: "ALERT_EMAIL_PASSWORD"
```

---

## Constraints

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
