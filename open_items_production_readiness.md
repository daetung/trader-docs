# Production Readiness — Open Items

**Status: UNCONFIRMED.** This document structures the production-deployment
risk analysis performed at design time. Nothing here is a confirmed design
decision — each item's "Example countermeasures" are candidate options for
review, not adopted specifications. Items graduate out of this file only via
an explicit design-session confirmation, at which point the relevant spec
files are updated and the item is removed here.

Severity tiers:
- **A (Blocking)** — must be resolved before live capital deployment
- **B (Important)** — resolve during early shadow/pilot operation
- **C (Accepted/Deferred)** — known limitations; re-confirm acceptance only

---

## P-0 [Immediate correction] Delivered patch self-contradiction

**Problem.** `remaining_9_files.patch`, metadata_crawler.md "Tick Bar
Aggregates Update" hunk: opening sentence says "Runs **after** Session Stats
Update" while the same paragraph (correctly, per the confirmed decision)
explains that session stats *sources its tick-derived baselines from
tick_bar_aggregates*, so this step must complete **first**. The opening
sentence is the error.

**Impact.** If applied as-is and implemented literally, the evening run
ordering inverts and every tick-derived baseline
(`buy_ratio_baseline`, `intra_tpm_baseline`,
`intra_avg_vol_per_tick_baseline`) is computed from a
tick_bar_aggregates state that excludes that same day's data —
a silent one-day staleness on every baseline, permanently.

**Fix (exact).** In that hunk, replace:
`Runs after Session Stats Update, populating ...`
with:
`Runs after ingestion and before Session Stats Update, populating ...`
No other change. Correction patch to be issued on confirmation.

---

# Tier A — Blocking

## P-1. No intraday halt feed

**Problem structure.**
- Trigger: any LULD / news-pending halt occurring during the live session —
  a routine event in the sub-$20 momentum universe this system targets.
- Mechanism: `trading_halts` is populated by the *daily* crawl only.
  Intraday, `halts_df` for today is empty or stale, so
  `classify_missing_bars()` classifies live halt gaps as `no_trade` →
  forward-filled synthetic bars flow into every indicator; and the position
  manager has no signal that exit orders cannot fill.
- Blast radius: per-halted-ticker, but affects the exact tickers the
  detector is most likely to have entered (halts follow the volatility this
  system trades).

**Example countermeasures.**
1. Integrate a real-time halt source (Nasdaq Trader halt feed / NYSE halts
   page polling / broker API halt flag) into LiveModeRunner: maintain an
   in-memory `live_halts` overlay merged with `trading_halts` before any
   `halts_df` consumer runs. Evening crawl remains the historical source of
   record; overlay is discarded after the evening run persists the official
   list.
2. Cheaper heuristic fallback (weaker): treat N consecutive zero-tick bars
   on a ticker with an open position or recent signal as "suspected halt" —
   suppress new entries, alert operator. Does not distinguish halt from
   illiquidity; acceptable only as an interim.
3. Position-manager rule regardless of source: on halt detection with an
   open position, mark position `halted`, pre-stage exit order for
   resumption auction, and alert — never silently keep tracking as if
   tradable.

**Affected specs:** live_mode_runner.md, caching_calculator.md (halts_df
freshness assumption), 09_backtest_engine.md (documents that backtest halts
are end-of-day-known — a train/serve information asymmetry worth noting).

---

## P-2. Tick-indicator train-serve skew at default config

**Problem structure.**
- Trigger: always, at current defaults — all 9 tick-derived indicators ship
  `precalculate_bars: 0` (session-only) in live, while training's Strategy A
  computes them over the full multi-day lookback.
- Mechanism: `window_comparison` halves in training span days; in live they
  span minutes-since-open. Early-session entries feed the model feature
  values (or min-sample NaNs) drawn from a distribution the model never saw
  in training. The min-sample guard prevents garbage values but *creates*
  a NaN pattern that training data does not contain.
- Blast radius: every live inference in the first portion of each session —
  precisely the highest-signal window for this strategy.

**Example countermeasures (mutually exclusive — decision required).**
1. **Flip to "lookback" before go-live**: backfill `tick_bar_aggregates`
   (migration step already specced), switch the tick indicators' config to
   `"lookback"`, and re-verify memory profile. Training and live then share
   the same window semantics. Cost: backfill time + Eager Pool memory delta
   (profiled as small).
2. **Retrain to session-only semantics**: modify training's Strategy A to
   compute tick-derived indicators using only same-day data per entry
   (mirroring live's accumulation), including reproducing the min-sample
   NaN guard in training. Cost: a training-pipeline change + retrain; keeps
   live memory at absolute minimum.
3. Hybrid: adopt (1) for the volume/price-scale indicators where cross-day
   context is plausibly informative, (2)-style session-only for the rest —
   requires per-indicator declaration via the existing checklist item 4,
   which the checklist already supports.

**Affected specs:** 02 (config defaults), 04 (Strategy A tick path),
caching_calculator.md, run_preprocess.md (option 2 only).

---

## P-3. Execution realism — no self-impact model, no entry-side partial fill, no position sizing

**Problem structure.**
- Trigger: every trade, in proportion to order size vs. displayed liquidity.
- Mechanism: fill estimation interpolates *observed* prices and caps
  participation via `sell_rate` — both assume the strategy's own order does
  not move the market. The target universe (low-float, low-price, momentum)
  is where that assumption fails hardest. Additionally: exit side has a
  partial-fill simulator; entry side has none (t+5s single fill price) —
  entries into thin books are modeled as instantly complete. And no spec
  defines how position size is chosen at all (shares per signal, per-ticker
  cap, aggregate exposure cap).
- Blast radius: systematic — backtest winning_rate and avg_pnl_pct are
  biased upward by an amount that grows with capital deployed, so the bias
  is smallest exactly when it is being validated (small pilot) and largest
  when it matters (scaled capital).

**Example countermeasures.**
1. Position sizing spec (prerequisite for everything else): e.g.
   `quantity = min(fixed_dollar / P_entry, participation_cap ×
   trailing_bar_volume, per_ticker_share_cap)` + aggregate open-position
   count/exposure limits. Backtest and live must share this function
   (utils-level, like resolve_signal).
2. Entry-side partial fill: mirror `simulate_exit_fill()` for entries —
   per-bundle `buy_rate` participation from t+5s onward, unfilled remainder
   after N seconds canceled (and the trade sized down accordingly). This
   also naturally penalizes signals on illiquid names.
3. Impact haircut as a stopgap: add a configurable adverse-slippage term
   scaling with `quantity / bundle_volume` to both entry and exit fills —
   crude, but moves the backtest bias to the conservative side while (2) is
   built.
4. Empirical calibration loop: during shadow/pilot, log realized fill vs.
   simulated fill per trade; fit the haircut/participation parameters from
   that data rather than guessing.

**Affected specs:** 09_backtest_engine.md, utils.md (sizing function,
entry-fill simulator), live_mode_runner.md (order placement),
experiment_log (new columns for realized-vs-simulated slippage if (4)).

---

## P-4. No session-start health gate / circuit breaker

**Problem structure.**
- Trigger: any upstream batch failure — evening run crash, yfinance outage,
  SEC endpoint failure, partial ingestion — discovered only implicitly.
- Mechanism: the system degrades *silently by design* (NaN fallbacks
  everywhere are individually correct), so a catastrophic input regression
  (e.g. session_stats empty for ALL tickers) manifests as "model receives a
  feature distribution it has never seen" while trading continues normally.
- Blast radius: entire session, all tickers, undetected until PnL damage.

**Example countermeasures.**
1. Pre-open health gate in LiveModeRunner Phase 1, evaluated after bulk
   loads, before the watchdog starts. Example checks and default actions:
   - `session_stats` coverage < 90% of Eager Pool tickers → **abort session**
     (hard gate; this is the P-0-class failure mode).
   - `tick_bar_aggregates` Tier-2 fallback rate > 20% → warn + proceed
     (self-healing case, but batch likely failed — alert operator).
   - `stock_meta` today-row coverage < 80% → warn; < 50% → abort.
   - premarket rename/corporate-events run completion marker absent →
     warn + proceed with yesterday's state, suppress new entries on any
     ticker with an unresolved rename candidate.
2. Completion markers: each batch stage writes a `(stage, date, status,
   finished_at)` row to a small `batch_runs` table; the gate reads it.
   Cheap, also solves P-5's ordering half.
3. Intra-session kill-switch thresholds (separate decision): e.g. realized
   session PnL < -X%, or signal rate > N× historical mean (model gone
   haywire) → suppress new entries, alert.

**Affected specs:** live_mode_runner.md (new Phase 0/gate step),
db_schema.md (`batch_runs`), metadata_crawler.md / migration_tool.md
(marker writes).

---

## P-5. DuckDB multi-process access policy undefined

**Problem structure.**
- Trigger: premarket cron (writes: ticker_cik_map, ticker_history,
  corporate_events) overlapping in time with LiveModeRunner startup
  (reads everything; writes indicator_cache + inference_log; holds the
  connection all session), plus any ad-hoc operator query.
- Mechanism: DuckDB is single-writer. Concurrent write attempts fail with
  lock errors; a long-lived RW connection in LiveModeRunner blocks the
  premarket job (or vice versa) nondeterministically depending on start
  timing.
- Blast radius: nondeterministic startup failures — the worst kind for a
  time-critical premarket sequence.

**Example countermeasures.**
1. Strict temporal ownership + markers (pairs with P-4.2): premarket job is
   the sole writer until it writes its completion marker; LiveModeRunner
   waits on the marker (with timeout → degraded start per P-4), then opens
   its connection. Evening job starts only after LiveModeRunner's
   end-of-session release.
2. Connection-mode split: LiveModeRunner opens `read_only=True` for the
   market-data connection and funnels its own writes (inference_log,
   indicator_cache) through a single dedicated writer thread/connection —
   DuckDB permits concurrent read-only connections alongside one writer
   less contentiously than multiple RW.
3. Physical split: move live-written tables (inference_log,
   indicator_cache) to a second .duckdb file owned exclusively by
   LiveModeRunner. Removes the contention class entirely at the cost of
   cross-file joins for analysis.

**Affected specs:** live_mode_runner.md, metadata_crawler.md,
db_schema.md (file topology note), architecture.md.

---

## P-6. Live bar-close semantics and feed-gap recovery undefined

**Problem structure.**
- Trigger: (a) illiquid tickers with multi-minute tick silences — is the
  09:41 bar "closed" when no 09:42 tick has arrived by 09:42:30? — and
  (b) any watchdog/feed outage mid-session.
- Mechanism: `on_bar_close()` requires strictly chronological calls, and
  `bars_trimmed` handles the *detection* of anomalies but no spec defines
  bar-close authority (tick-arrival vs. wall-clock timer) or the replay
  procedure after a gap (which bars to synthesize, how to rebuild
  Layer 2 state, what to do with positions opened before the gap).
- Blast radius: per-ticker for (a) — systematically biased against the
  illiquid names most prone to it; global for (b).

**Example countermeasures.**
1. Bar-close authority: wall-clock timer is authoritative — at HH:MM:SS+ε
   (e.g. +2s grace), the HH:MM bar closes with whatever ticks arrived; zero
   ticks → the bar enters the same no_trade/halt classification path as
   historical data (consistent with training's treatment).
2. Gap recovery procedure: on reconnect, (i) freeze new entries globally,
   (ii) fetch missed bars from the data vendor's intraday endpoint (or
   synthesize per rule 1 if unavailable), (iii) replay through
   on_bar_close() in order, (iv) reconcile open positions against current
   broker state before unfreezing. Positions whose exit conditions
   triggered during the gap: exit at market on reconnect, log with a
   distinct exit_reason (`feed_gap_exit`) so they're excluded from strategy
   PnL attribution.
3. Per-ticker staleness guard: a ticker whose last tick is older than N
   minutes is excluded from new entries regardless of feed health
   (liquidity gate doubling as a data-quality gate).

**Affected specs:** live_mode_runner.md (primary), caching_calculator.md
(replay contract), trade_log/inference_log (new event/exit_reason values).

---

# Tier B — Important (shadow/pilot phase)

## P-7. Premarket timeline feasibility unmeasured

**Problem.** The pre-04:00-ET chain (SEC fetch + rename detect → full-universe
corporate-events crawl (~15k yfinance calls — not actually "lightweight" at
universe scale) → Eager Pool load for ~15k tickers) has no measured
duration, no per-stage SLA, and no defined degraded mode if it overruns
into the pre-market session it exists to protect.

**Example countermeasures.** Measure each stage in staging; define SLA +
overrun policy (e.g. corporate-events crawl incomplete by 04:00 → proceed
with evening-run state + suppress entries on tickers with same-day events
detected later; Eager Pool incomplete → tickers not yet loaded are simply
not tradable until their worker finishes — already the natural behavior,
just needs stating). Consider splitting the corporate-events premarket
refresh to only tickers with detectable pending events rather than the full
universe, if measurement shows the full crawl doesn't fit.

**Affected specs:** metadata_crawler.md (SLA + degraded-mode section),
live_mode_runner.md.

## P-8. No per-ticker quarantine for missed same-day corporate events

**Problem.** Vendor-latency misses are accepted as unresolvable at the
*data* layer, but the *trading* layer currently has no last-line defense: a
missed 1:10 reverse split means every CONTINUOUS indicator for that ticker
is ~10× distorted all day, and the system will happily trade it.

**Example countermeasure.** Session-start (and premarket-refresh-time)
anomaly check per ticker: `|today_first_price / yesterday_close - 1| >
threshold` (e.g. 40%) AND no corporate_events row for today AND no halt
explanation → quarantine ticker for the day (no entries; alert for manual
review). False-positive cost is one day of missed trades on a genuinely
gapping ticker — cheap relative to trading a 10×-distorted feature set.
Threshold and interaction with legitimate momentum gappers (this strategy's
bread and butter!) need explicit tuning — that tension is exactly why this
is a design decision, not an obvious add.

**Affected specs:** live_mode_runner.md, possibly 01_entry_detection.md
(quarantine as a filter).

## P-9. Monitoring/alerting undefined

**Problem.** All failure signals land in logs/tables nobody watches:
inference_log preload_fail rates, rename_candidates.log, metadata_missing.log,
Tier-2 rates, EDGAR/yfinance failure counts.

**Example countermeasure.** Minimal alert set wired to one channel
(email/messenger): the P-4 gate results; daily batch completion/failure;
rename candidates pending > 0; preload_fail rate > threshold; live
winning-rate rolling divergence vs. backtest expectation (feeds P-12).
Implementation can be a single `tools/health_report.py` run at session
start + end — no new infrastructure required.

**Affected specs:** new tool doc; metadata_crawler.md cron additions.

## P-10. Live exit-path parity with BacktestEngine unenforced

**Problem.** Backtest exits flow through track_price_breach() +
simulate_exit_fill(); the live position-manager loop's decision logic is
under-specified and could be implemented independently, silently diverging
from what the backtest validated (e.g. bar-level vs. tick-level breach
checks, ambiguity handling).

**Example countermeasure.** Extend the "single source of truth" constraint
pattern (already used for resolve_signal / build_effective_bar_sequence) to
exit *decision* logic: position manager must call the same
track_price_breach() on live-accumulated ticks/bars; only the *order
placement* differs from backtest. State this as a hard constraint in
utils.md + live_mode_runner.md.

**Affected specs:** utils.md (constraint), live_mode_runner.md (position
manager section).

## P-11. Survivorship bias in the training dataset unverified

**Problem.** Case B handles delistings *present in the data*; the open
question is whether tickers delisted before/during the JSON collection
window are in the dataset at all. If the collection process only ever
fetched then-active tickers, the training set over-represents survivors and
the model's dn5 exposure is understated.

**Example countermeasure.** One-time audit: cross-reference the dataset's
ticker×date coverage against an external delisting list for the same
period; quantify the gap. If material, either backfill delisted tickers'
data (if obtainable) or document the bias magnitude and reflect it as a
haircut on backtested winning-rate expectations.

**Affected specs:** none until audited — findings would land in
migration_tool.md (backfill) or a data-quality doc.

## P-12. No retraining cadence / drift response / shadow-mode stage

**Problem.** The optimization machinery is strong but produces a static
deployment; no spec defines when to retrain, what divergence triggers
investigation, or a no-capital shadow stage before real orders.

**Example countermeasure.** Staged rollout spec: (1) shadow mode — full
live loop, orders logged not sent, N weeks; compare inference_log-derived
hypothetical PnL vs. backtest on the same dates; (2) pilot — minimal fixed
size; (3) scale. Retraining trigger examples: calendar (monthly rolling
refresh) OR divergence (live rolling winning rate outside backtest CI for
K sessions) — whichever first. Ties into P-9's divergence alert.

**Affected specs:** new ops doc; run_train.md / pipeline_optimizer.md
(refresh entry point).

---

# Tier C — Accepted limitations (re-confirm only)

## P-13. Vendor concentration (yfinance) + weak fallback tier
FMP free tier (250 req/day) is a per-ticker fallback, not an outage
fallback. Accepted unless budget for a paid secondary source is approved.
Documented in metadata_crawler.md already.

## P-14. dilution_rate blind between quarterly filings
S-1/424B/ATM dilution invisible until the next 10-Q. Known and stated in
04_feature_extractor.md. An 8-K/prospectus parser is a separate
infrastructure project — explicitly out of scope for now.

## P-15. Server timezone / DST discipline undocumented
All schedules are ET; deployment must pin TZ (e.g. host or container set to
America/New_York) and the two DST transition days each year should be on an
operator checklist. One paragraph in an ops doc suffices.

## P-16. Single-file DuckDB backup/recovery policy absent
Nightly file-level backup after the evening run completes (quiescent
window) + retention policy. One paragraph in an ops doc; interacts with
P-5's ownership windows.

---

## Suggested resolution order

P-0 (correction patch) → P-2 decision (a/b/c — gates retraining scope) →
P-3.1 sizing spec (gates all execution work) → P-1, P-4+P-5 (as one
ops-hardening package) → P-6 → shadow mode (P-12) begins → B-tier items
resolved during shadow using its measurements (P-7, P-9, P-10, P-11) →
P-8 threshold tuned on shadow data.
