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

1. **Margin-ratio computation**: blocked on nothing, and it touches sizing,
   the schema and the API budget at once.
2. **Halt-status source** is independent and blocks nothing else, but the
   halt path has no primary signal until it lands. `live_halt_episodes`'
   `source` column now gives it an audit axis: once a real source exists its
   intervals can be compared against the heuristic's directly, which nothing
   in the system could do before.
3. **Async boundary**, **early-close days** and the
   **manual-intervention CLI** are independent of each other and of the
   above. The async boundary is now the more constrained of the three: the
   watchdog scan fires N speculative REST calls per cycle and alternates
   across two accounts, so how many clients exist and where the boundary
   falls has consequences it did not have before.
4. **WS/REST tape asymmetry and exit-trigger path transitions** is blocked on
   SHADOW-PERIOD DATA rather than on design time, so it is not sequenced
   against the items above. Its two undesigned sub-questions (guard state at
   each handover direction) could be taken at any point; its deferred part
   cannot.
5. **`api_contract_checklist.md` re-evaluation** goes last by construction —
   it collects what the items above establish.
6. **Real-time bid/ask spread** is blocked on calendar time, not design time,
   so it is not sequenced against anything here.

---

## WS/REST tape asymmetry and exit-trigger path transitions

**Problem.** WS-primary and REST-backstop do NOT consume the same tick data.
The vendor's free real-time entitlement carries roughly half the trade
prints; the REST tick endpoint is vendor-guaranteed complete. Live's exit
trigger therefore runs on a different tape depending on which path is
active, and on a different tape again from backtest's.

`live_mode_runner.md`'s Exit Architecture asserts the opposite — "both paths
consume identical tick data with an identical filter, there is NO
accuracy/filter asymmetry between them, only latency differs". That sentence
is deliberately LEFT IN PLACE for now, and this item is what marks it false.
It is not a description but a JUSTIFICATION: it is what licenses swapping the
two paths with no rule governing the swap, so removing it exposes an
undesigned region rather than fixing anything.

The 2-print guard breaks in BOTH directions on a half tape, so the error has
no fixed sign. A false positive when an intervening sub-threshold print was
not seen — the guard exists to reject one-off spikes, and an unseen middle
print manufactures persistence. A false negative when one of a genuine
consecutive pair is among the missing half. Which dominates depends on print
density, the axis `feed_coverage_daily` already buckets
(`delayed_prints_d1`..`d4`); that table's stated purpose, making the guard's
constants resolvable against data, presumes the guard unsettled, which the
sentence above contradicted from the neighbouring section.

**Two sub-questions, both undesigned and both takeable now:**

(a) Guard-state semantics at WS-death handover, where a "consecutive pair"
    may straddle the transition from a half tape to a full one.
(b) The same at WS recovery — subtler, because it moves TOWARD lower
    density, so an in-progress confirmation weakens silently.

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
## Margin-ratio computation

**Problem.** The spec set holds the margin ratio as a SESSION CONSTANT,
probed once at session start (`live_mode_runner.md`'s R-8) and consumed as a
scalar by `compute_position_size()` through `execution.sizing_basis`. The
vendor publishes no account-level margin-ratio endpoint at all: the
balance-margin call returns `AstkMgn`, an amount rather than a ratio, and the
per-ticker figure comes from `inquiry/able-orderqty`. The effective
requirement is the account-level figure and the per-ticker one taken
together, so the quantity the design treats as one number per session is
actually one number per ticker.

The affected surface is wider than the probe: `live_session_state` stores a
single `margin_ratio` column, `compute_position_size()` takes a scalar, and a
per-ticker query at every entry signal is a NEW consumer on the API budget
(`inquiry/able-orderqty`, TPS 2). The budget item that would once have
absorbed that is gone — every consumer it enumerated is now bounded — so
this item carries its own accounting: at TPS 2 per account and 4 combined a
per-signal draw is small, but it is signal-proportional rather than
fixed-cadence and nothing else currently competes for that endpoint.
`api_contract_checklist.md`'s T-5 is re-anchored to this shape, and T-4
(whether the balance figure is cash or buying power) is entangled with it —
both feed the same sizing decision.

**Not yet designed.** Blocked on nothing.

---

## Halt-status source

**Problem.** `utils.query_halt_status()` is specified as the single access
point for trading-halt state, with `live_mode_runner.md`'s tick-rate
heuristic as its FALLBACK. The primary does not exist: the dbsec vendor's
catalogue publishes 20 REST endpoints across quote and trading and none
returns halt state, so this cannot be served through `trading_api.md` at all.
The intent is another vendor's API or a web source; which one is undecided.

The SHAPE is open too, and it matters more than the identity. Halt data may
be a market-wide feed rather than a per-ticker query, which would settle
whether any chunking survives and whether the function's ticker list is a
request or a filter. Until a source is chosen the function returns `None`
unconditionally and both call sites take their fallback path, which is
specified and buildable — so this blocks nothing, but the halt path runs on
its fallback alone.

**Not yet designed.**

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

---

## Async boundary

**Problem.** The SDK's client is an async facade over a synchronous core:
every REST call is dispatched through a thread, and both rate-limit waits
and retry backoff block that thread for their duration. The WebSocket path
is natively async, and its message callbacks are synchronous functions
invoked from inside the receive loop — a callback that blocks stalls
reception for every subscription on that connection, and the tick handler
(2-print guard, `bid_ask_snapshots` piggyback) runs there.

The decision already taken is that only the API-calling layer becomes
async-premised; forcing the rest into a synchronous shape was judged the
larger risk. Where exactly that boundary falls in the loop structure — and
what the tick callback is allowed to do inline — is not designed.

**Not yet designed.** Independent of the other items; can be taken at any
point.

---

## Early-close days

**Problem.** The NYSE calendar library is already the holiday source, but
only its holiday set is read, and a holiday set cannot express a half day.
On an early-close day the regular session ends at 13:00 rather than 16:00.

`session_close_exit_time`, `session_hard_exit_time`, the session-phase
boundaries that now decide which order types are permitted, and the evening
batch's own gating all assume a 16:00 close. The same library's schedule
interface exposes a per-date close time, so the fix needs no new vendor —
but the affected surface is wider than any one of those keys.

**Not yet designed.**

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
above. `docs/ops/api_contract_checklist.md` holds **20 rows, of which 15 are
still unverified** assumptions, two of those graded **A** (T-1: REST/WS tick
granularity; T-7: fill-event stream ID stability). The numbers reconcile as
20 = 15 unverified + 1 measured (T-6) + 4 retired (T-3, T-14, T-15, T-16),
and the row count does not fall as questions are settled because that
file's Role and Constraints forbid deleting rows — a broker change would
make a settled question live again, and a deleted row would have to be
rediscovered. T-1 and T-7 are the only A-graded rows, and both must be
verified, or their fallback paths confirmed sufficient, before Stage 2
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
alone rather than from a live exercise — the fill inquiry's paging behaviour
and its itemised mode bear on T-7 and T-8. T-1 acquired a METHOD this
session (the delayed stream carries the complete tape and is the instrument
for comparing against the live one, which `auxiliary_stream.md` builds) but
having a method is not having a result: it stays grade A and stays a Pilot
precondition. Which of the rest are answerable at a desk has not been sorted
through.
