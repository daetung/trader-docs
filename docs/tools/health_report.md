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

Two invocations per session: once at session start (reports on the
overnight/premarket batches feeding into Health Gate 1/2 — see
live_mode_runner.md), once at session end (reports on the session's own
inference/trade activity).

---

## Findings Gathered

```python
def gather_findings(db_conn, today_date, log_dir) -> dict:
    """
    1. batch_runs — status of every stage for today_date; a stage with no
       row at all (not even 'running') is reported distinctly from a
       'failed' row.
    2. ticker_cik_map — COUNT(*) WHERE status='suspended', with the
       (cik, ticker, suspend_reason) list for the console/detail view.
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
