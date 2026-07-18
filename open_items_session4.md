# Open Items — Session 4

**Reference/tracking document — same tier as `session_handoff.md`, not a spec
file.** Supersedes `open_items_session3.md` (item N there is now resolved —
see the handoff and `n_investing_dual_vendor.patch`) and the rolled-back
`live_risk_responses.md`. Both prior files may be removed.

This file holds **only what is not yet reviewed or resolved.** Confirmed work
(N, R-1, R-2, R-3) is in the patch set and the handoff's decision record.

The five R-items below were identified in this session's live-risk pass but
deferred without design review. Descriptions are the *problem statement and
first-pass thinking only* — none is a confirmed design. Re-review from the
problem each time; earlier informal notes in conversation are not decisions.

---

## Start-here order

R-3 already absorbed R-2's reconcile skeleton, so these five are less
entangled than N/R-1/R-2/R-3 were. Suggested order and why:

1. **R-6** — smallest, now narrowed to almost nothing; clears the deck.
2. **R-5** — entry gates; has a hard prerequisite from R-2 (below).
3. **R-4** — circuit breaker; interacts with R-5's freeze-reason set.
4. **R-7** — vendor-contract cluster; mostly a staging checklist + one spec.
5. **R-8** — small operational notes; independent, can go anytime.

---

## R-4 — no session-level kill switch / circuit breaker

**Problem.** Nothing halts a misbehaving model or a feature bug firing
entries all day. With sizing caps defaulting to 0 (disabled), the only
brakes are cash exhaustion and per-position stop-losses. A feature bug or
regime break that makes the model emit bad entries continuously has, in
effect, no line of defense — asymmetric against the otherwise careful
shadow→pilot→scale staging, which has no in-flight auto-halt.

**Config-driven constraint (confirmed this session, applies here).** Any
threshold this item introduces must read from config, not a hardcoded
value — this was confirmed as a general structural principle (see
r2_crash_recovery.patch's Exit Architecture section) covering
`max_positions` and poll-frequency-adjacent values specifically, and
applies by extension to whatever breaker thresholds this item adds.

**First-pass thinking (not confirmed).** An entry-side breaker in
LiveModeRunner: intraday realized-loss limit, consecutive-stop-loss count,
entries-per-hour ceiling. Trip → freeze **entries only** (existing positions
keep their own stop-loss exits; force-flatten adds market-impact risk for no
clear gain). Never auto-clears intra-session. In shadow mode, *evaluate but
don't enforce* — a real trip would truncate the bad-day data shadow exists to
measure; record a "would have tripped" finding instead. Interacts with R-5's
freeze-reason set (a breaker freeze holds entries but not exits — same row as
`restart_warmup`). Values must be config, not hardcoded. Note honestly: there
is no push-alert infra, so a mid-session trip is only as visible as logging +
the session-end finding — real-time alerting is itself an unaddressed
operational gap.

**Open sub-questions for review.** Consecutive-loss counter reset semantics
(reset to 0 on any non-stop-loss exit?). The ~5s evaluation-vs-1s-entry-gate
latency window. Whether entries-per-hour ships enabled or is pilot-calibrated
from backtest trade-rate stats.

---

## R-5 — entry gates absent from the specified entry sequence

**Problem.** `execution_common.md` declares `can_enter()` shared by
LiveModeRunner and config declares `max_positions`, but the actual Watchdog
step-5c sequence enforces neither cooldown nor the position cap — implement
the spec literally and live ships without either. Also undefined: whether a
ticker re-flagged while its limit entry is still pending can double-submit,
and whether the cooldown clock is set at submission or fill.

**Hard prerequisite from R-2 (confirmed there, must be honored here).**
Pending entries are now a `live_positions` DB row (the SSoT). So R-5's
cooldown restore and `max_positions` counting **must read DB `status`**, not
the in-memory `pending_entries` cache. Pending rows count toward the cap
(reserved capacity). The submission-time cooldown clock is the natural fit —
it also closes the duplicate-while-pending race, and it aligns with the
pending row being written at submission time.

**Config-driven constraint (confirmed this session, applies here).** The
`max_positions` check below must read the config key directly, not a
value copied/hardcoded at gate-definition time — same principle as R-4's
note above; see r2_crash_recovery.patch's Exit Architecture section for
where this was confirmed.

**First-pass thinking (not confirmed).** A step-5c.0 gate sequence: freeze-
reason set empty → `open + pending < max_positions` → `can_enter()` →
(existing) is_tradable → sizing → funds → submit. No same-ticker hard block
(sizing's `ticker_notional` already contemplates concurrent same-ticker
positions; cooldown is the intended rate limiter). Accepted backtest micro-
divergence: backtest has no pending state, so "pending counts toward cap" has
no backtest analog — commit the reasoning rather than leave it to be "fixed."

**Interaction with sizing (flagged, needs its own look).**
`compute_position_size()`'s `ticker_notional`/`total_notional` sum only
*open* positions — **pending limit orders' notional is not counted.** With
`exposure_cap_pct`/`per_ticker_share_cap_pct` at their 0 default this is
inert, but the moment Pilot turns those caps on, a burst of pending orders
can breach the cap on aggregate fill. Consider adding pending notional to
both sums as part of R-5.

---

## R-6 — shared decision parameters across config blocks

**Problem, as narrowed this session.** The original R-6 premise (four shared
decision params wrongly duplicated across `live_mode:`/`backtest:`) was
**mostly wrong on re-check.** `execution_common.md` already declares
`backtest:` as the single home for `entry_threshold`/`suppress_threshold`,
and `inferencer.md` reads `suppress_threshold` from `config["backtest"]` —
those are already single-sourced. That leaves only:

- **`max_hold_bars` — a genuine duplicate.** `live_mode.max_hold_bars: 60`
  exists as its own key, directly contradicting `execution_common.md`'s
  statement that `backtest:` retains `max_hold_bars`. This is a pre-existing
  doc inconsistency, independent of this session. Which key the code actually
  reads must be pinned to one source.
- **`entry_cooldown_minutes` — re-check needed** before patching; likely
  already single-sourced (same pattern as the thresholds), not yet
  confirmed.

**Do not** re-derive the wider four-key `execution:` move — that was the
false-alarm framing. Scope is: resolve the `max_hold_bars` duplication to one
source (and delete `live_mode.max_hold_bars` if code reads `backtest:` /
`execution:`), confirm `entry_cooldown_minutes`, nothing more.

Note the coupling: R-2 already reuses `execution.max_hold_bars` (the 60-bar
hold) for the `restart_gap_exit` / `overnight_exit` cutoff. Whatever single
source R-6 picks, that reference must resolve to it.

---

## R-7 — unverified vendor-contract assumptions (cluster)

**Problem.** Several load-bearing trading-API and data-vendor assumptions
have never been checked against the real services. Most are staging-
measurement items; one (order rejection) warrants a spec now.

**Order rejection — spec candidate (not yet confirmed).** A rejected
submission is distinct from an unfilled-then-canceled order and has no path
today. First-pass: `exit_reason='entry_rejected'` (quantity 0, same row shape
as `entry_canceled`), **excluded from `fit_execution_params()` — and from its
entry-count gate entirely**, since a rejection is a broker/account/regulatory
outcome, not evidence about `buy_rate`/`cancel_after_seconds` (unlike
`entry_canceled`, which *is* included). Counts as a cooldown attempt.
health_report visibility: per-session rejected count.

**Staging-measurement checklist (accumulated across this session).** Verify
against the real environment before/at Pilot:
- investing.com corporate-events pages: exact endpoints, ToS/scraping
  viability, symbol format vs. our universe (feeds the normalization item).
- WebSocket exit stream: endpoint, subscription message schema, concurrent-
  subscription ceiling, payload schema, heartbeat/reconnect semantics,
  reconnect gap-fill contract.
- Whether the REST tick endpoint returns the **same granularity/schema** as
  the WS stream (both exit paths must share one parser; if not, a
  normalization layer is needed).
- Full-range-since-last-query retention window (how far back; first-ever
  query behavior) — load-bearing for catch-up, Feed Outage recovery, and
  warm-restart gap-fill.
- Market-order fill semantics (synchronous full fill? partial/fragmented?).
- Throughput ceiling (~100 tickers/sec assumed) **and** the specific
  09:20–09:25 combined load when the in-process recheck's chunked fetches
  overlap the watchdog/position loops.
- What the balance query returns for `session_start_cash` (cash vs. buying
  power vs. settled funds).

---

## R-8 — small operational notes

**Problem.** Three minor gaps, each independent.

- **PDT / account-type precondition.** Real-order stages assume a margin
  account maintaining ≥ $25k (US Pattern Day Trader rule) or a PDT-exempt
  structure — a precondition *of* the system, not something it engineers
  around. Belongs in the Pilot preconditions.
- **NTP clock discipline.** Bar-Close Authority's wall-clock deadlines and
  the freeze trigger assume ≤ ~1s skew against exchange time; the host must
  be NTP-synced. Operator-checklist companion to the existing DST note.
- **`inference_log` PK collision.** `logged_at` at second resolution collides
  when two events share (ticker, date, hour, second); give it sub-second
  (microsecond) precision. Preferred over `ON CONFLICT IGNORE` (drops a
  diagnostic) or PK redesign (readers key on nothing that breaks).

---

## New open item — `health_report.md` finding 2 reads dropped columns

**Discovered** during this session's second verification pass; pre-existing,
unrelated to any of this session's changes.

**Problem.** `health_report.md`'s finding 2 is specified as
`ticker_cik_map — COUNT(*) WHERE status='suspended'`, with a
`(cik, ticker, suspend_reason)` detail list. Both `status` and
`suspend_reason` were removed from `ticker_cik_map` by the earlier N-2
restructure, which replaced them with the independent `rename_pending` and
`quarantine_reason` columns. As written, finding 2 queries columns that no
longer exist.

**Not yet designed**, but the shape is probably obvious: count rows where
`rename_pending IS NOT NULL OR quarantine_reason IS NOT NULL` (matching
`is_tradable()`'s OR), and report the two reasons separately in the detail
view rather than as one `suspend_reason` field — the whole point of N-2
was that the two suspension causes are independent, and a single merged
count would hide which one is firing.

---

## New open item — ticker-symbol normalization

**Discovered** while wiring investing.com as a second corporate-events vendor
(item N).

**Problem.** Symbology may differ across every external source: yfinance,
investing.com, and the **trading API itself (bars/ticks included)** need not
match. N ships with naive `active_ticker_universe` matching as an interim,
which means an investing.com row for a differently-formatted symbol can
silently fail to match and be dropped — a false "no event." A unified
normalization layer (canonical internal symbol ↔ per-vendor symbol) is
needed. `health_report.md` gains an investing.com match-failure-rate finding
(added in N's patch) specifically to make this gap observable until the
normalization layer exists.

**Not yet designed.** Scope, canonical-form choice, and where normalization
sits relative to `ticker_cik_map` are all open.

---

## Deliberately settled (do not re-open)

Recorded so they aren't re-derived:
- Price-gap probe-threshold tuning (old N option A/C) — dropped; the vendor
  cross-check supersedes it.
- Per-position force-flatten on a breaker trip — rejected (R-4 first-pass).
- Same-ticker entry hard block — rejected (R-5 first-pass).
- Two-writer lock-handoff for the 09:20 recheck — rejected (R-1); in-process
  delegation replaced it.
- Poll-delay 5s alignment in backtest — removed (P-10 revision under R-2).
- Bundle-based backstop with low-trimming — dropped; REST-tick backstop at
  WS granularity replaced it.
