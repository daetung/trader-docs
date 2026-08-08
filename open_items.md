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

1. **API call budget allocation** comes first and is now UNBLOCKED: the
   call-point inventory it waited on exists (`trading_api.md`), with each
   endpoint's published TPS beside it. It is also load-bearing rather than
   optimising — the watchdog working-set model leaves the polling loop
   unbuildable until it resolves.
2. **Margin-ratio computation** is blocked on nothing and touches sizing,
   the schema and the budget at once, so its answer changes what 1 has to
   plan for.
3. **Halt-status source** is independent and blocks nothing else, but the
   halt path has no primary signal until it lands.
4. **Async boundary**, **early-close days**, **fill-inquiry page size** and
   the **manual-intervention CLI** are independent of each other and of the
   above.
5. **`api_contract_checklist.md` re-evaluation** goes last by construction —
   it collects what the items above establish.
6. **Real-time bid/ask spread** is blocked on calendar time, not design time,
   so it is not sequenced against anything here.

---

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
(`inquiry/able-orderqty`, TPS 2) that the budget item has to plan for.
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

## API call budget allocation

**Problem.** Supersedes the former "rate-limit reconciliation" item, whose
blocking condition is gone: per-endpoint limits are published (the SDK
carries them as a table and paces against them automatically, two-tier, app
level plus endpoint level), so nothing is waiting on their extraction any
more. What remains is not reconciliation but ALLOCATION — the endpoint
limits and the global app limit are both real, and several consumers draw
on the same budget on the same cadence.

The consumers, all of them: the watchdog loop's bar fetch across the
candidate set; the exit-side backstop's paired filled/outstanding inquiries;
the exit ladder's per-cycle orderbook query for each outstanding limit exit;
`signal_time_rest`'s bid/ask query at every entry signal
(frequency-proportional, not fixed-cadence); the halt-check bulk call; and
entry/exit order submission itself.

Two findings from this session bear on it directly. The exit-side fill
backstop is ACCOUNT-WIDE rather than per-order, so its cost does not scale
with the number of outstanding orders — a load source that was previously
assumed to scale does not. Against that, the watchdog loop's bar fetch does
scale with the candidate set, and the per-endpoint pacing is a minimum
INTERVAL rather than a burst allowance, so a naive per-ticker fetch
serialises: at the chart endpoint's published rate, fetching a full
candidate set exceeds the loop's own interval by an order of magnitude. A
bulk quote call accepting up to 50 symbols exists and is the obvious
substitute, but it returns a price snapshot rather than OHLCV bars, so it
does not directly serve `on_bar_close()`.

**Also absorbed here:** whether the exit-side REST tick backstop should
become an unconditional every-cycle poll rather than firing only when the WS
stream is down. It would give defense-in-depth against a silently-failed
subscription and remove the need to judge "is WS dead", but it is another
draw on the same budget and cannot be decided separately from the rest.

**Not yet designed.** No longer blocked: `trading_api.md`'s call-point
inventory now enumerates the consumers, with each endpoint's published TPS
beside it. Two figures from it bear directly on the allocation. `chart/min`
is TPS 4, so a per-ticker bar fetch across a full 50-ticker working set takes
12.5s against a 1s loop — the order-of-magnitude gap, quantified. And
cancellation is not a separate endpoint but `trading/…/order` with a
discriminator, so cancels draw on the SAME TPS-10 bucket as submissions.
Against that, filled and outstanding inquiries are ONE endpoint rather than
two, and the margin-ratio item above may add a per-entry-signal consumer.

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

## Fill-inquiry page size

**Problem.** The exit-side backstop reads the filled-orders inquiry
newest-first and takes only the first page. The page size is a server-side
internal parameter with no field in the request, and the returned row count
varies per call, so there is no guarantee a single page covers a full
position-check interval.

Correctness is not at stake — `seen_fills` is fill-ID idempotent, so a fill
missed by one page folds in unchanged when it appears on the next, and the
vanished-order rule is deliberately asymmetric for exactly this reason. What
is at stake is the latency bound on detecting a vanished order, and whether
continuation paging is needed at all.

**Not yet designed.** Self-measuring in the manner of the bar-latency
finding: recording the returned row count each cycle would accumulate the
distribution during ordinary operation rather than requiring a separate
measurement exercise. Whether to add it as a checklist row has not been
decided.

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
  receives `ticks_entry`, `ohlcv_entry` and `p_entry` — never a book — and so
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
above. `docs/ops/api_contract_checklist.md` holds **16 rows, of which 14 are
still unverified** assumptions, two graded **A** (T-1: REST/WS tick
granularity; T-7: fill-event stream ID stability). The three numbers
reconcile as 16 = 14 unverified + T-6 measured + T-3 retired, and the row
count does not fall as questions are settled because that file's Role and
Constraints forbid deleting rows — a broker change would make a settled
question live again, and a deleted row would have to be rediscovered. Both
A-graded rows must
be verified, or their fallback paths confirmed sufficient, before Stage 2
(Pilot) — see that file's own "When to use it" section. Not duplicated here;
consult that file directly rather than letting a second copy of its contents
drift out of sync.

T-6 was MEASURED against both production and demo accounts, and rewritten
around what was actually found: subscription types are mutually exclusive per
connection, which removed the "WS sequence lease" the row had been anchored
to. T-9 kept its row but lost its config key, the key having been deleted
with that lease.

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
