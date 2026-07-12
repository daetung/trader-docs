# Open Items — Next Session

**Reference/tracking document — same tier as open_items_production_readiness.md
and session_handoff.md, not a spec file.** Supersedes
open_items_production_readiness.md as the active worklist; that file is left
unmodified as the historical record of the original design-time risk
analysis. Status below reflects this session's work (patches 1-9).

---

## Resolved this session (patches 1-9 — see session record for detail)

All of the following were reviewed, decided, and written into patches. Not
reopened here; listed only so the next session doesn't need to re-derive
status from the original document.

| Item | Resolution | Patches |
|---|---|---|
| P-0 | Confirmed already fixed in metadata_crawler.md; no action | — |
| P-2 | precalculate_bars:0 default rationale documented; switch-to-lookback deferred to post-selection | 1 |
| P-3 | execution_common.md built out fully: sizing, funds gate, entry/exit fill simulation, order type (market/limit), impact haircut note | 1, 7, 8, 9 |
| P-1 | Trading-API-primary + tick-rate heuristic fallback (position-scoped), halt handling | 3a/3b |
| P-4 | Health Gate 1/2, batch_runs, is_tradable(), Session Lifecycle restructure | 2, 3a/3b, 7 |
| P-5 | Temporal ownership (1hr premarket window) + read_only/writer split | 2, 3a |
| P-6 | Bar-close authority, per-ticker catch-up (Watchlist Append), staleness guard skipped (redundant w/ detector filters), on_bar_close() O(1) rule, **system-level Feed Outage Recovery** | 3a/3b, 4, 9 |
| P-9 | health_report.py: structured logs, batch_runs/ticker_cik_map/inference_log findings, Discord+email, execution-param divergence finding | 5, 8 |
| P-10 | track_price_breach() shared w/ live, poll-delay simulation in backtest, global shared loop (vs. per-position, analyzed) | 3b, 7 |
| P-12 | Shadow→Pilot→Scale staged rollout, fit_execution_params() calibration loop, retraining triggers (calendar+divergence) | 6, 8 |
| (2 pre-existing doc findings) | caching_calculator.md stale example list; CLAUDE.md fundamentals boundary rule | 4, 6 |

**Also resolved, not in the original document**: `check_funds_available()`
use_all_cash design, entry-side sizing/fill ordering (p_entry-based),
predicted-vs-actual fill logging (`trade_log.predicted_*`), `execution_params`
table — all emerged from patch-accuracy audits during this session, not from
new open items.

---

## Carried forward — start here next session

### P-8. No per-ticker quarantine for missed same-day corporate events

**Not discussed this session at all** — was next in the original group plan,
session moved to patch generation before reaching it. Full original problem
statement (unchanged, still accurate as far as this session's other changes
go — nothing resolved above touches this):

**Problem.** Vendor-latency misses are accepted as unresolvable at the data
layer, but the trading layer currently has no last-line defense: a missed
1:10 reverse split means every CONTINUOUS indicator for that ticker is ~10x
distorted all day, and the system will happily trade it.

**Example countermeasure.** Session-start (and premarket-refresh-time)
anomaly check per ticker: `|today_first_price / yesterday_close - 1| >
threshold` (e.g. 40%) AND no corporate_events row for today AND no halt
explanation → quarantine ticker for the day (no entries; alert for manual
review). Threshold and interaction with legitimate momentum gappers (this
strategy's bread and butter) need explicit tuning.

**Affected specs (anticipated):** live_mode_runner.md (likely another
Health-Gate-adjacent or per-ticker check, similar pattern to `is_tradable()`
— worth checking whether this folds into that gate or needs its own),
possibly 01_entry_detection.md.

**Note for next session**: this session's Health Gate 1/2 and `is_tradable()`
infrastructure (batch_runs, ticker_cik_map.status) may be directly reusable
scaffolding for however P-8's quarantine flag ends up stored — worth
checking before designing a new mechanism from scratch.

---

## Blocked — needs real-world measurement/audit before design work

- **P-7** (premarket timeline feasibility): needs staged-environment timing
  measurement of the pre-04:00 chain before an SLA/degraded-mode can be
  designed. Not actionable in a design-only session.
- **P-11** (survivorship bias): needs a one-time audit cross-referencing
  dataset ticker×date coverage against an external delisting list. Not
  actionable in a design-only session.

## Low-priority / accepted (Tier C, unchanged)

- **P-13** (yfinance vendor concentration): accepted, already documented in
  metadata_crawler.md.
- **P-14** (dilution_rate blind between filings): accepted, already
  documented in 04_feature_extractor.md.
- **P-15** (timezone/DST discipline): one paragraph needed in an ops doc —
  candidate location is shadow_retraining.md or a new minimal ops-checklist
  doc. Not done this session; low effort whenever picked up.
- **P-16** (DuckDB backup/recovery policy): one paragraph needed, interacts
  with P-5's ownership windows (already built — see batch_runs). Same
  low-effort, whenever-convenient status as P-15.
