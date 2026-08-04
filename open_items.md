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

1. **`docs/api/trading_api.md` — the module's undesigned interior** comes
   first: the call-point inventory it contains is what every other API-side
   item is measured against, and two of them cannot start without it.
2. **API call budget allocation** depends directly on 1 — a budget cannot be
   divided among consumers before the consumers are enumerated.
3. **Market-data tier and tick completeness** is independent of both and can
   be taken at any point; it is placed here because its answer may change
   what 2 has to budget for.
4. **Async boundary**, **early-close days** and **fill-inquiry page size** are
   independent of each other and of the above.
5. **`api_contract_checklist.md` re-evaluation** goes last by construction —
   it collects what the items above establish.
6. **Real-time bid/ask spread** is blocked on calendar time, not design time,
   so it is not sequenced against anything here.

---

## `docs/api/trading_api.md` — the module's undesigned interior

**Problem.** The module spec exists and states what was settled this session
— its layer position over the vendored SDK, the result contract (a rejected
order arrives as HTTP 200, so `rsp_cd == "00000"` is the only business
success test), symbol/exchange encoding at the point of call, the async
boundary, and the orderbook call being in scope. Its interior is not
designed, and the file marks the gap in its own "Not Yet Designed" section.

**Not yet designed.** Four questions, deliberately deferred together because
each constrains the others:

- **Call-point inventory** — which endpoints, with which arguments, derived
  from the caller specs' stated needs rather than from the vendor's
  catalogue. The vendor publishes roughly two dozen for overseas equities;
  the ones with a consumer here are closer to a dozen. A first pass exists
  in session notes but was explicitly NOT treated as confirmed.
- **Layer responsibility split** — which axes the module absorbs versus
  leaves to callers, beyond the result contract and symbol encoding already
  settled.
- **Response normalization** — the vendor returns every numeric as a string,
  order fields as fixed-width zero-padded values, and some fields as an
  empty string, while the spec set assumes numeric types throughout.
- **Whether the module stays one file** — the domain surface may split along
  the vendor's own quote/order/realtime axis. Fixing a file boundary before
  the call-point inventory exists would only require redrawing it.

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

**Not yet designed.** Blocked on the call-point inventory above.

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

## Market-data tier and tick completeness

**Problem.** The vendor states that the FREE real-time market-data
entitlement delivers roughly half of the trade prints, against the full
feed. Exit Architecture's WS-primary tp/sl detection with its two-print
guard, and the bar-latency finding that calibrates the bar-close grace
window, both read the trade stream as if it were the whole tape.

If the entitlement is what the account actually runs on, the two-print
guard's time-to-detection and the REST backstop's share of the work both
change, and this is close to what `api_contract_checklist.md`'s T-1 (a
grade A row) already asks in a different form. Whether a paid tier exists,
what it costs, and whether the delayed streams are relevant at all are
unexamined.

**Not yet designed.** Independent; may change what the budget-allocation
item above has to plan for, which is why it is sequenced ahead of it.

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

A third consumer appeared this session: the exit ladder's spread-position
pricing lives under `live_mode:` rather than `execution:` precisely because
BacktestEngine has no bid/ask model to mirror it. If backtest is ever
extended to replay `bid_ask_snapshots`, those keys move alongside the two
paths above.

**Not yet designed** — still. Not actionable again until `bid_ask_snapshots`
has enough history; revisit then, not before.

---

## `api_contract_checklist.md` — verify before Pilot

Not a new design problem — a pointer, so it isn't lost among the items
above. `docs/ops/api_contract_checklist.md` holds **15** unverified
vendor-contract assumptions, two graded **A** (T-1: REST/WS tick
granularity; T-7: fill-event stream ID stability). Both A-graded rows must
be verified, or their fallback paths confirmed sufficient, before Stage 2
(Pilot) — see that file's own "When to use it" section. Not duplicated here;
consult that file directly rather than letting a second copy of its contents
drift out of sync.

The count fell from 16 because T-6 was MEASURED this session, against both
production and demo accounts, and rewritten around what was actually found:
subscription types are mutually exclusive per connection, which removed the
"WS sequence lease" the row had been anchored to. T-9 kept its row but lost
its config key, the key having been deleted with that lease.

**A re-evaluation pass is itself unresolved.** Several rows are now
answerable from vendor documentation alone rather than from a live
measurement exercise — the fill inquiry's paging behaviour and its itemised
mode bear on T-7 and T-8, and the server-clock question is an
enumeration rather than an experiment. Which rows have moved into the
"answerable at a desk" category, and which genuinely still need the running
service, has not been sorted through.
