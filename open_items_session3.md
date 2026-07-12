# Open Items — Session 3 Carryover

**Reference/tracking document — same tier as session_handoff.md, not a spec
file.** Supersedes both `open_items_production_readiness.md` (the original
design-time risk analysis) and `open_items_next_session.md` (session 2's
worklist). Both of those files may be removed from the project now — this
document is the complete, current record; nothing in either prior file is
still open except what is restated below.

---

## Resolved this session

Delivered as four patch files (none yet applied to the real codebase —
apply in the order listed, since the later two assume the earlier ones are
already in place for the specific regions they touch):

1. **`p8_p1_p15_p16.patch`** — P-8 (per-ticker corporate-event quarantine,
   full design incl. bulk-chunked API calls), P-1 (halt-status endpoint
   actually integrated, API-primary + tick-rate fallback), P-15 (timezone/DST
   note), P-16 (DuckDB backup/recovery note), plus this session's N-1
   (merged premarket cron process), N-2 (`ticker_cik_map` rename/quarantine
   column separation), N-3 (delta-pass redesign, 09:20 ET, sequential
   fetch), N-9 (04:00-pass coverage note), and minor operational notes (log
   rotation, holiday no-op).
2. **`survivorship_audit.patch`** — P-11's audit procedure and decision
   policy (`migration_tool.md`).
3. **`n5_n8_backtest_fixes.patch`** — N-5 (`is_tradable()` removed from
   BacktestEngine's entry-side flow, with reasoning committed to prevent
   re-adding it as a "fix" later), N-8 (Health Gate 1 model/feature-pipeline
   consistency check, new hard gate).
4. **`n7_execution_param_bounds.patch`** — N-7 (`get_execution_param()`
   single read-point with mathematically-derived hard bounds;
   `fit_execution_params()` relative-bound rejection via
   `execution.fit_rejection_multiplier`, default 3; health_report.md
   finding 9). **Depends on patch 1** — its `health_report.md` hunks are
   incremental against patch 1's finding-8 text, not against the pristine
   original.

P-0, P-2 through P-6, P-9, P-10, P-12, P-13, P-14 were already resolved in
the prior session (see git history / prior patches, not restated here).

---

## Carried forward — start here next session

### Intraday `corporate_events` staleness has no recompute path

**Discovered** during this session's real-world-deployment risk pass
(originally "Part 1" in that discussion), while designing N-3's delta-pass
timing.

**Problem.** `CachingIndicatorCalculator.session_start_compute()` applies
corporate-event bar adjustment exactly once, at Eager Pool time (~04:00-05:00
ET, right after Session Start Gating). `on_bar_close()`'s incremental
updates are, by explicit design constraint, O(1)-only — "*Full-window
recomputation from scratch on every call is not permitted for any indicator
in this category*" (`caching_calculator.md`). No mechanism anywhere
re-runs `session_start_compute()`, or otherwise corrects an already-running
session's indicator state, once a corporate event is later confirmed.

Compounding this: N-3's confirmed design has the 09:20 ET delta pass
deliberately skip `crawl_corporate_events()` (only the price-gap-based
`check_corporate_event_anomaly()` runs). This means `corporate_events` has
**no path to update at all between the 04:00 crawl and the evening crawl**
— if the 04:00 crawl misses a same-day event, there is no further
same-day opportunity for the system to learn about it, only to
price-gap-detect a symptom of it.

**Net effect:** a same-day corporate event that (a) the 04:00 crawl misses
AND (b) produces a price gap under `quarantine.price_anomaly_threshold`
(or is otherwise missed by that heuristic) leaves that ticker's indicators
silently distorted for the entire session, with literally no defense layer
left to catch it — not detection (already exhausted), not correction (does
not exist).

**Not resolved this session** — surfaced during N-3/N-9 discussion but
explicitly deferred; the fix requires deciding among at least three
different shapes of solution (give `corporate_events` a same-day update
path after all, perhaps reopening whether the delta pass should crawl a
narrow subset; build a live-mode recompute mechanism despite the O(1)
constraint, e.g. scoped to just the affected ticker on confirmation; or
accept and formally bound the residual risk via the existing quarantine
threshold alone) and none were evaluated in depth.

**Affected specs (anticipated):** `caching_calculator.md` (if a recompute
path is built), `metadata_crawler.md` (if the delta pass gains a narrow
crawl), `execution_common.md`/`db_schema.md`'s `quarantine.
price_anomaly_threshold` (if the resolution is "tighten this instead").

**Note for next session:** start here — this is the one item from this
session with genuine design work still to do, as opposed to the two below,
which need external inputs no design session can supply.

---

## Blocked — needs external measurement/data before design work

- **P-7 residual** (narrowed this session from the original full scope —
  see `p8_p1_p15_p16.patch`'s N-3/N-9 commentary for what's already
  resolved): whether the confirmed checkpoint policy (05:00 ET
  wait-deadline, 09:20 ET delta pass, 09:30 ET open) leaves comfortable
  margin in practice, or whether degraded-mode paths turn out to fire
  routinely rather than as rare exceptions; also whether P-7's original
  suggestion (narrow the corporate-events crawl to tickers with detectable
  pending events) turns out to be necessary. Needs staged-environment
  timing measurement — folded in: N-6's memory-profile measurement
  (`~11.9GB` Eager Pool estimate is unverified) can be measured in the same
  staging pass.
- **P-11 residual**: the actual survivorship-gap `gap_ratio` measurement.
  Procedure and decision policy are fully designed (see
  `survivorship_audit.patch`) — only the external delisting-list source
  and the resulting number are outstanding.

Both are "run the already-designed procedure against real
data/environment," not open design questions.
