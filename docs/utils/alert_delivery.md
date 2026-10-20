# Module: Alert Delivery

**File:** `src/utils/alert_delivery.py`

---

## Role

Delivery of findings to the operator: the log record, the channel queues,
suppression, the senders and the drain. Its callers:
- health_report.md's `run_health_report()`, for every path in its
  Invocation Contract;
- utils.md's DB Access Coordination helper, for its own deliveries
  (`owner_error_end`, `guest_db_open_failed` — utils.md's Helper deliveries).

What a path delivers, its arguments and whether it needs the DB are
health_report.md's Invocation Contract; this file holds how a delivery is
logged, queued, sent and drained, whatever the path.

---

## Log Writing (always, regardless of delivery)

```python
def write_to_log(findings: dict, log_path: Path) -> None:
    """
    Appends ONE JSONL record per invocation:

      {"ts":"<UTC ISO-8601>","alert_key":"<the invoking path's alert_key>",
       "scope":"whole"|"partial","severity":"ok"|"warn"|"abort",
       "finding_count":<int>,"findings":{...}}

    `scope` is what marks a subset invocation, so a reader cannot mistake
    one for a whole report. `severity` is the highest among that
    invocation's findings, and `findings` is the findings dict the caller
    passed to deliver(), verbatim. The scalar fields lead so the head of a
    line is readable without parsing the body, and health_report.md's CLI's
    in-window mode selects on them.

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

    `log_write_failed` also covers a failed coordination-store write
    (utils.md's DB Access Coordination):
    - an owner injects it into its next delivery;
    - a guest or aux run writes it to stderr.

    Called on a subset invocation too, where it records a PARTIAL set. Such
    an entry does not substitute for the scheduled whole-report write, and
    the log line is marked so a reader cannot mistake one for the other.
    """
    ...
```

---

## Delivery (best-effort)

```python
def deliver(alert_key: str, findings: dict, config: dict, log_dir: Path,
            db_conn=None, suppressible: bool = False,
            scope: str = "whole") -> None:
    """Log, filter by alert_level, then enqueue per channel. A None db_conn
    takes the DB-less handling below. `alert_key` is required when
    suppressible."""
    today_date = utils.current_run()["date_basis"]   # the calling run's
                                                   # registration date
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
        enqueue(channel, alert_key, severity, send_findings, mode, config, db_conn)
```

`enqueue()` (module-internal) returns once the `alert_log` row is written
(db_schema.md) with `dispatched_at` set and `outcome` NULL, or immediately
with `outcome='suppressed'`.

A path whose DB column in health_report.md's Invocation Contract reads
**no** is delivered with a `None` `db_conn`. On such a path:
- no `alert_log` row is written — `enqueue()` skips the row;
- the send outcome is written to the `log_dir` log;
- the process drains delivery, up to the path's drain timeout
  (health_report.md's drain-timeout list), before it exits.

**Delivery itself is asynchronous** — a background worker per channel
performs the actual send and fills in `completed_at` / `outcome`.
`deliver()` therefore returns as soon as logging is done and never blocks
its caller on a socket.

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

---

## Drain

```python
def drain(timeout: float) -> None:
    """Wait for pending deliveries up to `timeout` seconds; anything still
    pending is recorded outcome='dropped_drain'."""
    ...
```

Draining BEFORE closing the DB is what lets the background workers write
`completed_at` / `outcome` — including `outcome='dropped_drain'` for anything
still pending at the timeout. Nothing about delivery is therefore invisible.
Which timeout a path drains with is health_report.md's drain-timeout list;
where a session drains is health_report.md's Shutdown Order and Drain.

---

## Senders

Module-internal; the background workers call them.

### Discord

```python
import requests
from datetime import datetime, timezone

def send_discord_alert(webhook_url: str, title: str, severity: str, fields: dict) -> bool:
    """severity: 'ok' | 'warn' | 'abort'. Returns False on any failure —
    caller logs the failure per deliver() above."""
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
  event_alert_suppression:                 # event-driven, single-report
    abort: { mode: "rate_limited" }        #   deliveries only — scheduled
    warn:  { mode: "always" }              #   batch reports are never
    ok:    { mode: "always" }              #   suppressed (see Constraints)
  max_pending_alert_keys: 10               # per channel; distinct alert_keys,
                                           # latest entry per key retained
  drain_timeout_seconds: 600               # normal shutdown: wait for pending
                                           # deliveries. Bounded by the ~50min
                                           # gap to the 21:00 evening batch
  abort_drain_timeout_seconds: 600         # paths whose finding requests
                                           # manual intervention —
                                           # health_report.md's drain-timeout
                                           # list: owner_lock_wait,
                                           # owner_error_end,
                                           # guest_db_open_failed 'unmatched' —
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

- `write_to_log()` always runs before any delivery attempt; delivery success
  or failure never affects whether findings are logged
- `event_alert_timeout_seconds` is per channel and has NO default. A channel
  added later must declare its own; no fallback is defined, deliberately, so
  that the decision is forced rather than silently inheriting a value sized
  for a different transport. No further channels are planned
- Suppression applies ONLY to event-driven, single-report deliveries. Scheduled
  invocations send every finding in one batched message, which suppression's
  per-`alert_key` model does not address, and at one call per scheduled point
  they could not reach a rate limit anyway. This is a consequence of the
  DELIVERY UNIT, not of severity — which is why the `warn`/`ok` entries in
  `event_alert_suppression` are inert today rather than wrong
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
