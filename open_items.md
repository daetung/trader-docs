# Open Items

**Reference/tracking document, not a spec file — but PATCH-DELIVERED like
one.** Renamed from `open_items_session6.md`; the name is now fixed and
carries no session number, and this is the only open-items file. There is no
per-session successor to write and no supersession to declare.

That change exists to remove a specific failure mode. Under the old scheme
each session rewrote the whole file, carrying unresolved items forward by
hand — and every defect found in Session 7's opening audit was a carry-
forward defect: a pointer to a file that no longer existed, a cross-
reference describing behaviour that had since been designed, one of three
copies of a schedule time left un-updated. A single file amended by patch
cannot fail that way, because unchanged items are never retyped.

It remains OUTSIDE the handoff's Spec File Structure — patch delivery is a
safety property of how it is edited, not a claim that it specifies anything.

This file holds **only what is not yet reviewed or resolved.** Descriptions
below are problem statements and first-pass thinking, not confirmed designs
— re-review from the problem each time. An item's text is NOT rewritten when
a session leaves it untouched; absence of edits means absence of review, not
confirmation that it still reads correctly.

---

## Suggested order

Items still unresolved, as one flat list:
- **Daily raw market-data collection**.
- **Vectorizer method mapping completeness**.
- **Run launchers**, its crashed-runner part remaining.
- **Manual-intervention CLI** — unblocked.
- **WS/REST tape asymmetry and exit-trigger path transitions** — blocked on
  shadow-period data; only its deferred part remains.
- **Real-time bid/ask spread** — blocked on calendar time.
- **Lock measurements**.
- **Challenger shadow inference**.
- **Point-in-time security identity** — worked when the Security Identity
  Audit's identity_gap_ratio reaches its threshold and such a roster is
  obtainable.
- **Premarket entry enablement**.
- **Fundamentals acceptance-time cutoff** — worked together with a premarket
  or intraday fundamentals refresh.
- **Delayed quote stream unavailable**.
- **`api_contract_checklist.md` re-evaluation** — last; it collects what the
  other items establish.

---

## WS/REST tape asymmetry and exit-trigger path transitions

**Problem.** WS-primary and REST-backstop do NOT consume the same tick data.
The vendor's free real-time entitlement carries roughly half the trade
prints; the REST tick endpoint is vendor-guaranteed complete. Live's exit
trigger therefore runs on a different tape depending on which path is
active, and on a different tape again from backtest's.

The proposition exists in TWO copies, treated differently, and both
locations are named here so neither reads as authoritative by omission.

`live_mode_runner.md`'s Exit Architecture asserts the opposite — "both paths
consume identical tick data with an identical filter, there is NO
accuracy/filter asymmetry between them, only latency differs". That sentence
is deliberately LEFT IN PLACE for now, and this item is what marks it false.
It is not a description but a JUSTIFICATION: it is what licenses swapping the
two paths with no rule governing the swap, so removing it exposes an
undesigned region rather than fixing anything.

`09_backtest_engine.md`'s R-2 Constraint carried the same claim and HAS been
corrected, because the reasoning above does not transfer: that file has no
WS/REST branch, so the sentence licensed nothing there and deleting it left
no hole. Leaving both would have been worse than leaving neither — a reader
grepping the proposition would meet two copies, one marked false and one
not, and the unmarked one reads as authoritative. Its R-2 Constraint now
carries three kinds rather than two, this tape asymmetry being the second.

The 2-print guard breaks in BOTH directions on a half tape, so the error has
no fixed sign. A false positive when an intervening sub-threshold print was
not seen — the guard exists to reject one-off spikes, and an unseen middle
print manufactures persistence. A false negative when one of a genuine
consecutive pair is among the missing half. Which dominates depends on print
density, the axis `feed_coverage_daily` already buckets
(`delayed_prints_d1`..`d4`); that table's stated purpose, making the guard's
constants resolvable against data, presumes the guard unsettled, which the
sentence above contradicted from the neighbouring section.

**Settled since.** Guard-state semantics at both handover directions —
WS-death, where a "consecutive pair" could straddle the move from a half
tape to a full one, and WS recovery, which moves toward lower density so an
in-progress confirmation would weaken silently. Both are closed by the
pending state carrying its `source_path` and being discarded on any
mismatch, so no state survives a tape change
(`live_mode_runner.md`'s Exit Architecture).

**Deferred, and explicitly not to be decided yet.** Whether the full-tape
REST buffer should serve as a COMPARATOR rather than only a fallback. That
buffer is now maintained per held ticker every cycle regardless of stage,
since the indicator path needs its derived bundles in real mode too, so a
fast half tape and a slow full one are both in hand and discarding the latter
for detection is not self-evident. But the size and direction of the
divergence are unmeasured, and `exit_trigger_agreement_daily` is the
instrument built to measure them — its L-vs-M term is exactly the tape
effect. Deciding before that data exists would repeat this project's
most-reversed failure. Blocked on shadow-period data, not on design time.

Also deferred to the same instrument: telling a legitimately quiet ticker
from an undelivered one. The `breach_confirm_window_seconds` bound closes
the FALSE-FIRING half of that — a frozen pending state can no longer pair
ticks minutes apart — but not the DETECTION half, which needs the same
L-vs-M term.

---

## Manual-intervention CLI

**Problem.** A clean Session Shutdown cancels in-flight exit orders precisely
because an order left resting after the process dies could fill overnight
with nothing watching it. Broker Reconcile at the next session start cancels
pending ENTRY orders only. A session that did not reach clean shutdown
therefore leaves resting AFTER-MARKET exit limits covered by neither path —
and that is exactly the state `health_report.md`'s finding 11 detects and
asks for manual intervention in.

The operator's only route today is the broker's own app, which the
ghost-order rule made load-bearing-but-unenforceable when it adopted a
60-second cancel threshold: a manual order placed outside this system is
cancelled about a minute later. A CLI entry point over TradingAPI would give
that intervention a tracked route.

**Not yet designed.** At least four questions: whether it is a standalone
entry point in the manner of `metadata_crawler.md`, which runs outside any
LiveModeRunner session; whether it writes `live_positions` and in-flight
state or only calls the API, since a cancellation invisible to the DB leaves
the next reconcile with a stale picture; how it behaves when a runner IS
alive and both are acting on the same orders; and whether running it
alongside a live session is safe at all. The last two are entangled —
restricting it to "only when the runner is dead" answers the third by
construction, so the concurrency question should be settled before the
interface is drawn.

A further design target: on launch, the CLI pre-fills a sell order for each
position awaiting manual liquidation — `fraction_held` positions among them
(`live_mode_runner.md`'s settlement pending) — for the operator to submit
directly. Those sell orders are phase-validated new requests, so its client
is constructed with the closes and ends of `utils.load_session_boundaries()`
(trading_api.md's Layer Responsibility Split).

---

## Real-time bid/ask spread — model-feature / entry-gate paths

**Problem.** Both paths originally described here needed two things:
confirmation a collection mechanism exists, and months of accumulated
history. The first is resolved — collection is running. The second is not:
neither path is usable until enough history has accumulated.

- **As a model feature.** Needs retraining against the new historical
  feature once enough of it exists.
- **As an execution-time gate.** BacktestEngine still cannot replay a live
  bid/ask gate against historical data older than the collection start date.

A further consumer appeared:
- `market_buy_price_margin` sits under `execution:`, and backtest's
  market-BUY limit price reads it (09_backtest_engine.md);
- the exit ladder's keys sit under `execution:`, read by every engine —
  BacktestEngine prices its ladder from a spread estimate
  (execution_common.md).

- **As entry sizing.** A simulated market entry (backtest, shadow, pilot
  counterfactual) fills under the entry limit rule at its market-BUY limit
  price (execution_common.md's Fill Simulation Model), and the backtest
  funds check price is the market-BUY limit price (09_backtest_engine.md).
  The two engines do not size identically. This is NOT a defect to fix by
  making one call the other: the live side uses the better information and
  should, while the backtest side cannot. The remaining question is the
  measured gap between backtest's base price (last print × (1 + s / 2)) and
  live `ask1`, against accumulated `bid_ask_snapshots`, which is what the
  two paths above are already waiting on.

**Not yet designed** — still. Not actionable again until `bid_ask_snapshots`
has enough history; revisit then, not before.

---

## `api_contract_checklist.md` — verify before Pilot

Not a new design problem — a pointer, so it isn't lost among the items
above. `docs/ops/api_contract_checklist.md` holds the vendor-contract
assumptions the specs rest on, among them:

- unverified and graded **A** — the rows its Grade column marks A with an
  empty Measured column;
- measured: T-6
- retired: T-3, T-7, T-8, T-14, T-15, T-16, T-29

Rows are not removed as questions are settled, because that file's Role
and Constraints forbid deleting rows — a broker change would
make a settled question live again, and a deleted row would have to be
rediscovered. Each A-graded row must be verified before Stage 2 (Pilot) —
see that file's own "When to use it" section. Not duplicated here;
consult that file directly rather than letting a second copy of its contents
drift out of sync.

T-6 was MEASURED against both production and demo accounts, and rewritten
around what was actually found: subscription types are mutually exclusive per
connection, which removed the "WS sequence lease" the row had been anchored
to. T-9 kept its row but lost its config key, the key having been deleted
with that lease.

Four rows were ADDED this session and three RETIRED in the same session:
T-14 (second-resolution bars at `InputDivXtick=1`, `dataCnt` to 2000,
costing receive time rather than TPS), T-15 (demo and production return
identical `chart/min` and `orderbook` data on independent budgets) and T-16
(REST round-trip 400-600ms). Each was measured and its value transcribed
into the spec that depends on it, which is where a checked row's result
belongs — the checklist is a queue of work still to do, not a second copy
of the specs. T-17 is the exception and the one to look at: it records that
the watchdog list fires for every ticker crossing conditions A-G — an
assumption the prefix scan's early stop rests on, ACCEPTED rather than
proven, and measured only after the fact, by the evening detection-gap
stage.

**A re-evaluation pass is itself unresolved**, though it shrank. Three rows
moved: T-3 retired outright, its premise gone once per-endpoint TPS turned
out to be published and the SDK to pace against it; T-5 re-anchored from a
non-existent account-level endpoint to the per-ticker one; and T-11's
server-clock question answered negatively — no such endpoint is published,
and the nearest substitute is the broker-stamped timestamp carried on every
order and fill. Several rows remain answerable from vendor documentation
alone rather than from a live exercise — the fill inquiry's itemised mode
bears on T-21. T-1 acquired a METHOD this session (the delayed stream
carries the complete tape and is the instrument for comparing against the
live one, which `auxiliary_stream.md` builds) but having a method is not
having a result: it stays grade A and stays a Pilot precondition. Which of
the rest are answerable at a desk has not been sorted through.

---

## Lock measurements

**Problem.** The DB access coordination (db_coordination.md) relies on lock
behaviour not yet measured:
- the time for another process to take a lock after its holder is killed,
  on Linux and on Windows;
- that a forked child holds none of its parent's locks;
- Windows' lock-release delay after a crash.

---

## Run launchers

**Problem.** No process relaunches a crashed LiveModeRunner for warm
restart; the dispatcher does not.

---

## Premarket entry enablement

**Problem.** Entry points are regular-session only (data_boundary.md).
Enabling premarket entries needs, as prerequisites:
- a definition of minute_of_session for premarket entries;
- a live premarket evaluation path:
  - premarket scanning;
  - incremental backfill of today's bars;
  - the watchdog's premarket behaviour;
  - finding 29's scope;
- the train/live entry-population mismatch before the runner's Step 8;
- protection of entries before the 09:20 premarket recheck;
- verification of premarket fill and spread assumptions;
- evaluation-population filtering and Health Gate comparison of the session
  mode.

Sites to widen:
- scan()'s regular-phase exclusion step;
- Inferencer's step 1 check;
- the session_mode config comment;
- data_boundary.md's regular-only entry rule.

Already designed, carried here: the cooldown runs continuously across the
premarket/regular boundary with no reset at 09:30.

Related: execution_common.md's `use_loo` comment.

---

## Challenger shadow inference

**Problem.** Before approval, the challenger predicts beside the deployed
model in the session and its signals are recorded; once it exists, an
awaiting candidate that passed it is reported 'warn'.

---

## Point-in-time security identity

**Problem.** A security's identity over time:
- the CIK valid on each date per security segment;
- fundamentals read for that CIK;
- a reused symbol as separate segments.

Its source: a roster carrying CIK, the Survivorship Audit's source
(migration_tool.md). Worked when the Security Identity Audit's
identity_gap_ratio reaches its threshold and such a roster is obtainable.

---

## Fundamentals acceptance-time cutoff

**Problem.** A fundamentals fact is read for D when filed before D
(data_boundary.md's "Point-in-time correctness for fundamentals"). A fact
accepted on the prior trading day after 17:30 ET carries D's filing date and
waits a day. Live reads fundamentals collected by the prior trading day's
evening run alone.

First-pass thinking:
- source: the submissions feed's per-filing acceptance timestamp, joined to
  companyfacts facts by accession; its field and time zone verified against
  a live response; the migration backfill fetches the older submissions
  files;
- a per-row known date computed at collection, readers comparing it with D;
  a fact without an acceptance time keeps the filed-before-D rule;
- its gain depends on when live receives fundamentals:
  - the evening run alone: facts accepted between 17:30 ET and the prior
    trading day's `after_hours_end`;
  - a premarket fundamentals refresh: facts accepted up to that refresh,
    usable on D;
  - an intraday fundamentals refresh: facts accepted during D, usable for
    D's later entries — the cutoff per entry, accepted and received by a
    refresh before the entry's t bar open — training resolving each entry
    against the refresh schedule;
- worked together with a premarket or intraday fundamentals refresh, not
  alone.

---

## Delayed quote stream unavailable

**Problem.** The V10/V11 delayed quote stream exists in the vendor's API
documents only; measured, it delivers no data.

Resting on it:
- auxiliary_stream.md's delayed quote capture, its first consumer;
- metadata_crawler.md's Feed Coverage Analysis and db_schema.md's
  feed_coverage_daily;
- health_report.md's feed coverage reporting;
- live_mode_runner.md's Session Shutdown stage 2, held for
  `auxiliary_stream.tail_minutes`;
- api_contract_checklist.md's T-1 method paragraph;
- this file's WS/REST tape asymmetry density buckets
  (`delayed_prints_d1`..`d4`) and its `api_contract_checklist.md` item's T-1
  method sentence.

Open:
- what measures the live stream's missing prints instead, the REST 1-tick
  tape being the spec's complete tape;
- whether loop A stays for its next consumer (V61).

---

## Vectorizer method mapping completeness

**Problem.** The spec states that every computed indicator series is
vectorized — 04_feature_extractor.md's Processing Pipeline ("for each
indicator series, apply mapped transform method(s)") and its Strategy A
lists; pipeline_optimizer.md's default group_alternatives [hl_spread,
roll_spread], whose groups must be keys of get_feature_groups() — but
03_vectorizer.md's mapping states no method for sar, sar_trend, adx,
dmi_plus, dmi_minus, roc, cci, mfi, bb_upper, bb_middle, bb_lower,
vr_volatility, volume_ma_{n}, vr_volume, vwap, buyer_initiated_ratio,
tick_direction_momentum, roll_spread_pct and hl_spread_pct, and only
bb_pctb's primary method (its Statistical Summary's Applies to).

Also open:
- no rule for an output absent from the mapping;
- `vectorizer.*` sits inside config_hash (run_preprocess.md) and is declared
  nowhere;
- 03's per-method Applies to lines disagree with its table;
- 04's Strategy A list omits roc, stochastic, cci, mfi, vr_volatility and
  volume_ma.

First-pass thinking:
- every computed output holds a mapping row, FeatureExtractor init raising on
  one that holds none;
- methods after the existing rows' patterns: roc, cci, mfi and bb_pctb
  statistical_summary + shape_features; adx, dmi_plus and dmi_minus
  statistical_summary + linear_trend; sar_trend and vr_volatility
  statistical_summary; vr_volume, buyer_initiated_ratio, roll_spread_pct and
  hl_spread_pct statistical_summary + window_comparison;
  tick_direction_momentum statistical_summary + linear_trend; volume_ma_{n}
  as volume's row;
- the price-level series bb_upper, bb_middle, bb_lower, sar and vwap through
  level_distance on the t−1 value, a design call the spec does not yet make;
- 03 gains a Config Keys section declaring `vectorizer.mapping`, its Applies
  to lines pointing at the table;
- 04's Strategy A membership stated as every bar-derived CONTINUOUS
  indicator.

---

## Daily raw market-data collection

**Problem.** No component produces the JSON dump that New JSON Ingestion
(metadata_crawler.md) and migration_tool.md import — `{TICKER}_{DATE}_min.json`
and `{TICKER}_{DATE}_tick.json` under data_root (db_schema.md's File Naming
and Directory Convention):
- metadata_crawler.md ingests "newly collected JSON files", its evening slot
  derived from the dump's completion at 20:00 ET;
- architecture.md's pipeline starts at Raw JSON;
- utils.md's Numeric Rules name the dump among the broker API's values;
  auxiliary_stream.md's delayed quote capture states that bid_ask_snapshots
  and tick_10 come from REST polling;
- `chart/min` and `chart/tick` are drawn by the live session alone, which
  stores none of what they return in ohlcv_1min or tick_10.

Open, the collector's:
- universe per date: the tickers queried, from the trading API's ticker
  master (`inquiry/stock-ticker`, per exchange, no date), and whether the
  list is kept:
  - the per-date ticker lists stored today: live_session_state.all_tickers,
    on dates the runner started; ticker_cik_map.trading_api_seen_date, the
    latest date alone;
  - the ticker master serves no past date's list, so a date collected
    without a kept list holds no universe;
- queries: `chart/min` at 1 minute and `chart/tick` at 10 ticks — date range,
  `InputOrgAdjPrc`, market code, `dataCnt` and `cont_key` paging; the
  `InputOrgAdjPrc` setting the existing dump was collected under;
- output: the JSON dump under its file convention, or a direct DuckDB load;
- outcome per (ticker, date, resolution) — data, no records (message 2679),
  error — and whether a no-record query writes a file;
- timing: the vendor's cut a few seconds past 19:30; the completion the
  evening slot derives from (stated 20:00 ET, 17:00 on an early-close day);
  api_doc's tick-chart example shows a 19:59:57 print under its own date,
  against the measured cut, the measurement governing;
- call budget: the endpoints' per-account TPS, shared with the live session
  until its shutdown;
- failure: retry, and re-collection of a past date, how far back the vendor
  serves one; the file finding 59's remediation places in data_root.

Also open:
- finding 59's failed-file gap does not resolve when the re-imported file
  holds no in-range row, a zero-row import leaving no row and no
  source_changes row;
- a ticker missing both files on a date reads as a no-trade ticker-date, and
  a date whose dump holds part of the tickers raises no date gap; a dead
  position whose next has_data day is such a date resolves as Case B
  (05_labeler.md, 09_backtest_engine.md).

Stated intent (the original requirements): the separately developed JSON
collection script is modified so newly collected data loads into DuckDB,
split off as a tool, and merged with metadata crawling into one daily tool.

First-pass thinking:
- a collection stage in `collect_daily.py` ahead of New JSON Ingestion;
- the universe as the date's ticker master across NYSE, NASDAQ and AMEX;
- unadjusted prices (`InputOrgAdjPrc` 0), stored bars being raw and adjusted
  through corporate_events;
- a per-date manifest — every ticker queried and its outcome per resolution —
  and a per-file ingestion outcome record, against which finding 59 judges
  failed files and missing ticker-dates, the outcome record replacing the
  `collection_gaps` health_events failure records and finding 59's
  failed-file gap and its newness restated against it;
- Case B unchanged, a missing ticker-date remediated through finding 59 and
  source_changes.
