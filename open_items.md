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

1. **Manual-intervention CLI** is unblocked.
   The async boundary that used to sit here is closed: how many clients exist
   and where the boundary falls are both decided, and its shared review
   surface with (2)'s sub-questions no longer exists because those are
   settled too.
2. **WS/REST tape asymmetry and exit-trigger path transitions** is blocked on
   SHADOW-PERIOD DATA rather than on design time, so it is not sequenced
   against the items above. Only its deferred part remains.
3. **`api_contract_checklist.md` re-evaluation** goes last by construction —
   it collects what the items above establish.
4. **Real-time bid/ask spread** is blocked on calendar time, not design time,
   so it is not sequenced against anything here.
5. **Config-duplicating signature defaults** is unblocked but is hygiene
   rather than a design blocker, so it is not sequenced against the items
   above either. It is listed because a new item left out of this list is the
   carry-forward defect this file exists to prevent, inverted.
6. **Deferred DECIMAL type conversion** is unblocked and not sequenced
   against the items above.
7. **Fractional-balance sell path** is blocked on (6).
8. **Carried partially filled exit — engine divergence** is unblocked and not
   sequenced against the items above.
9. **No path from a trained model's `run_id` to the live model** is unblocked
   and not sequenced against the items above.
10. **Finding 6 divergence — resampling unit and threshold** is not sequenced
    against the items above.

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

Two more consumers appeared. The exit ladder's spread-position pricing lives
under `live_mode:` rather than `execution:` precisely because BacktestEngine
has no bid/ask model to mirror it, and `market_buy_price_margin` joins it
there for the same reason. If backtest is ever extended to replay
`bid_ask_snapshots`, those keys move alongside the two paths above.

- **As entry sizing.** The market-BUY funds gate prices against
  `ask1 * market_buy_price_margin` in live mode, while `simulate_entry_fill()`
  receives `ticks_entry` and `p_entry` — never a book — and so
  approximates the vendor's undisclosed conversion from bar data instead. The
  two do not size identically and the gap is unquantified. This is NOT a
  defect to fix by making one call the other: the live side uses the better
  information and should, while the backtest side cannot. What is undesigned
  is the backtest approximation itself — what it is, and how far it sits from
  the live figure — and that cannot be settled without measurement against
  accumulated `bid_ask_snapshots`, which is what the two paths above are
  already waiting on.

**Not yet designed** — still. Not actionable again until `bid_ask_snapshots`
has enough history; revisit then, not before.

---

## `api_contract_checklist.md` — verify before Pilot

Not a new design problem — a pointer, so it isn't lost among the items
above. `docs/ops/api_contract_checklist.md` holds the broker-behaviour
assumptions the specs rest on, among them:

- unverified and graded **A**: T-1 (REST/WS tick granularity), T-21 (each
  execution reported under exactly one order number), T-25
  (`AstkExecBaseQty` reflecting same-day executions immediately)
- measured: T-6
- retired: T-3, T-7, T-8, T-14, T-15, T-16

Rows are not removed as questions are settled, because that file's Role
and Constraints forbid deleting rows — a broker change would
make a settled question live again, and a deleted row would have to be
rediscovered. T-1, T-21 and T-25 are the only A-graded rows, and each must be
verified, or its fallback path confirmed sufficient, before Stage 2
(Pilot) — see that file's own "When to use it" section. Not duplicated here;
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

## Config-duplicating signature defaults

A LATENT risk, not an active defect. A caller that passes the argument is
unaffected, and the failure needs both an omission and a later config change.

**The class.** A function-argument default equal to the configured value.
Omitting the argument silently reads the default, and no test detects the
omission while the two agree.

**Members**, default site — config site:

- `02_indicator_calculator.md` `reset_mode` `"date"` — its own config block
- `02_indicator_calculator.md` `regular_start` `"093000"` — its own config block
- `utils.md` `ambiguity_priority` `"up"`, TWICE in one file — `05_labeler.md`
- `utils.md` `n_sessions` `20` — `02_indicator_calculator.md`
- `utils.md` `vol_metric` `"avg_intraday_range"` — `pipeline_optimizer.md`
- `utils.md` `embargo_days` `5` — `06_class_balancer.md`
- `01_entry_detection.md` `max_sideways_ratio` `0.6` — its own config block
- `execution_common.md` `use_all_cash` `True` — its own config block
- `08_dimensionality_reducer.md` `bottom_pct` `0.20` — its own config block

**Not members**, having no config counterpart: `utils.md`'s `window_days` and
`val_fraction`, `07_lgbm_pipeline.md`'s `importance_type`, and the local
defaults — `return_intermediate`, `balance`, `fold_idx`, `dry_run`,
`return_data`.

**`target_valid_bars` is excluded**, its default already removed. That one was
removed because it broke WITHIN one engine: `09_backtest_engine.md` counts
`config["execution"]["max_hold_bars"]` valid bars over a sequence the function
would have capped at 60, so raising the key past 60 made the time-limit exit
unreachable rather than merely late. No equivalent internal contradiction is
shown for the members above, and that is what makes the class latent.

**The open question is which resolution the class takes**, not whether each
site is a duplicate. Removal, as `target_valid_bars` took; or the `None`
sentinel already present in `06_class_balancer.md` for the same parameter
`utils.md` defaults to `5`; or a per-site judgement. Two patterns for one
parameter already coexist, which is itself the argument for settling it once.

**`01_entry_detection.md`'s `max_sideways_ratio` is the sharpest instance**:
its own inline comment reads `# from config` beside the hardcoded default.

---

## Deferred DECIMAL type conversion

**Problem.** Ledger quantities, per-share prices and cash amounts are to be
DECIMAL, and the rules are already decided — `utils.md`'s Ledger Numeric
Rules and the float boundary rules (V1 and V6 in `trading_api.md`'s Response
Normalization, V2–V5 in `utils.md`). What remains is applying them to the surface
that still carries INTEGER or DOUBLE:

- the existing columns of `live_order_requests`, `live_fills`, `live_orders`
  and `live_positions` (`db_schema.md`); `live_fills.cum_after_qty` is part of
  its primary key
- `trade_log`'s price and `predicted_*` columns
- `live_session_state.session_start_cash`
- `live_ticker_terms`' `mgnrt` and `order_cost`
- `experiment_log`'s amount totals
- `execution_common.md`'s floor computations: position sizing,
  `check_funds_available()`, the exposure limits, the fill simulator
- the exit ladder's limit pricing and tick rounding
- `09_backtest_engine.md`'s Case A cash arithmetic

Retired with it: V3's exception for writes to existing DOUBLE price columns,
V6's 'number' format, and `trading_api.md`'s integrality assertion for
INTEGER ledger columns.

Excluded: `corporate_events.value` stays DOUBLE (`utils.md`'s split-ratio
rule). A ratio such as 1:3 has no exact DECIMAL form, and the rule already
takes value × ratio and value ÷ ratio in DOUBLE and rounds the result.

Undecided: whether market data and feature tables are included.

**Findings for the next session** (undecided; raised this session):

- Principle raised: DECIMAL applies only where the source value is a decimal
  value from the broker API; a value whose source is a ratio such as 1:x
  stays DOUBLE.
- Under it, these stay on the surface: the existing ledger table columns
  (`live_positions.entry_mgnrt` included), `session_start_cash`,
  `live_ticker_terms`' `mgnrt` and `order_cost`, position sizing,
  `check_funds_available()`, the exposure limits, the exit ladder's limit
  pricing and tick rounding, and `trade_log`'s `fill_price`, `exit_price`
  and `weighted_avg_exit_price`.
- Under it, these leave the surface and stay DOUBLE: `trade_log`'s and
  `live_orders`' `predicted_*`, the fill simulator's floor computation,
  `experiment_log`'s amount totals, and `09_backtest_engine.md`'s Case A
  cash arithmetic, whose deferral would close as DOUBLE.
- Market data and feature tables, though fetched from the broker's quote
  endpoints, are proposed to stay DOUBLE — which would settle the Undecided
  line above. A market value copied into the ledger (`reference_price`) is
  already converted per V1.
- Values whose source is a ratio already stay DOUBLE and are not on the
  surface: split ratios and the factors derived from them, and configured
  or fitted rates.
- A restated dividend arrives rounded by its vendor after division, so
  `utils.dividend_gross_amount()` cannot recover the exact gross. The error
  lies within the upsert agreement test's tolerance, and no type choice
  removes it.

---

## Fractional-balance sell path

**Problem.** A split or reverse split can leave a fractional balance the
broker holds rather than settles in cash. Selling it needs fractional
`requested_qty` / `qty` / `exec_qty` columns, so it waits for the type
conversion above.

Meanwhile `live_mode_runner.md` records `event_settlement_stalled`
`kind='fraction_held'` at each Broker Reconcile call and the fraction is
handled manually.

Needs: `api_contract_checklist.md` T-29 (whether the order API sells a
fractional balance), and whether IS2 carries fractional quantities.

---

## Carried partially filled exit — engine divergence

**Problem.** When an exit fills only partly before session end, live and
shadow sell the remainder in a later session, while BacktestEngine ends the
row at D with `dead_position_penalty_pct` on the remainder. The two engines'
pnl for such a row therefore differ structurally.

Scope: backtest modelling. Not decided.

---

## No path from a trained model's `run_id` to the live model

**Problem.** The live model is the `run_id` set in config by hand
(`inferencer.md`). `shadow_retraining.md` declares the retraining scope
without containing a step that moves a newly trained `run_id` into that
config. The regime holdout gate (`pipeline_optimizer.md`) only checks the
configured `run_id` at session start; it does not choose one.

---

## Finding 6 divergence — resampling unit and threshold

**Problem.** `health_report.md`'s finding 6 and `shadow_retraining.md`'s
divergence trigger compare live winning rate against a backtest CI. The CI is
computed by `utils.bootstrap_ci()`, but the trigger's resampling unit and
its threshold are undecided.
