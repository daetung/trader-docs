# Module: TradingAPI

**File:** `src/api/trading_api.py`

---

## Role

The single caller-facing layer over the vendored trading-API SDK
(`vendor/dbsec-open-api/` — see `sdk_dependency.md`). Every trading-API call
in this system goes through it; no other module imports `dbsec_sdk` directly.

The SDK owns transport: token lifecycle, rate governance, retry, continuation
(`cont_yn`/`cont_key`), and the WebSocket connection/reconnection mechanism.
This module owns everything vendor-shaped that a caller must never see, plus
the policies the SDK has no concept of.

**Boundary test.** A caller that can observe `"00000000009.4300"`, `"FNTSLA"`,
`cont_key`, or `rsp_cd` is reading through a hole in this layer. Those four
are the concrete form of the boundary; any new surface is judged against them.

**Naming.** Adopted from the vocabulary the caller specs already use — the
"trading API" of `live_mode_runner.md` and `metadata_crawler.md` is this
module. Not "wrapper" (names a relation, not a responsibility; both layers
wrap something) and not "interface", which in this doc set already means a
class's abstract method contract (`viz_connector.md`, `base_model.md`).

---

## Result Contract

**This is a safety requirement, not a style preference.**

The SDK's `APIResponse.is_ok` reports HTTP 2xx only, and `__bool__` returns
`is_ok`. The vendor returns HTTP 200 with a business error code in the body
for a rejected order — insufficient margin, market closed, insufficient sell
quantity are all HTTP 200. A caller writing `if resp:` would treat a rejected
order as accepted.

`is_ok` and `__bool__` are therefore never exposed. Business success is
`rsp_cd == "00000"`; anything else is converted here into a result a caller
can branch on, carrying the vendor's own `rsp_msg` verbatim for
`trade_log.reject_reason` (`db_schema.md`).

The vendor's message-code table is documented (`api_doc/`), but
`reject_reason` stays verbatim regardless — see `api_contract_checklist.md`
T-10 for why no enum is defined before the observed vocabulary is known.

**Two code families, and only one crosses.** The vendor's codes split into an
`IGW*` family that `api_doc/` lists under HTTP 400/500, and a numeric business
family (3563 regular session closed, 3589 before open, 00700 insufficient
orderable amount, 20910 wash trade) that arrives as HTTP 200 with
`rsp_cd != "00000"`. The split is not a new policy — it is the contract above
read the other way round: if a rejected order comes back as HTTP 200 with a
business code, then the `IGW*` family is by construction the SDK's, whose
`is_ok` reports HTTP 2xx only and whose retry classification acts on it.
Handing an `IGW*` code to a caller re-leaks transport, so it does not cross.
The business family does, and reaches `reject_reason` verbatim.

The session-phase codes 3563/3589/3590 are surfaced DISTINCTLY rather than as
ordinary rejections. This module validates order type against session phase
before the call (below), so one of these arriving means that validation
failed — a different event from a legitimate business refusal.

The family-to-HTTP-status correlation is read off `api_doc/`'s grouping and
should be confirmed against live responses.

---

## Symbol and Exchange Encoding

Assembled here, at the point of call. `ticker_cik_map` stores
`trading_api_symbol` without an exchange prefix and `trading_api_exchange`
separately (`db_schema.md`, `metadata_crawler.md`) precisely so that the two
encodings the vendor uses can both be built from one stored form:

- WebSocket `tr_key` — exchange prefix concatenated with the symbol
  (`FY`/`FN`/`FA` for NYSE/NASDAQ/AMEX)
- REST — exchange and symbol as separate request fields

Callers pass a ticker. They do not construct either form.

---

## Async Boundary

The SDK's client is an async facade over a synchronous core; every REST call
is dispatched through `asyncio.to_thread`, and rate-limit waits and retry
backoff both block a pool thread for their duration. REST therefore needs an
event loop only at this boundary — caller loop bodies stay synchronous.

The WebSocket path is different and is the constraint that matters: its
receive loop is natively async and its message callbacks are synchronous
functions invoked from inside that loop. **A callback that blocks stalls
reception for every subscription on that connection.** The tick handler
(`live_mode_runner.md`'s Exit Architecture, including the 2-print guard and
the `bid_ask_snapshots` piggyback) runs there.

The SDK logs and swallows a callback exception rather than propagating it,
so the callback owns its own error path; a raised exception is otherwise
visible only in the SDK's log.

**This module stays one file, and the REST/WS distinction above is why the
question was worth asking rather than an answer to it.** The seam does not
hold: `POST /api/v1/websocket/disconnectSession` is a REST call on the
:8443 domain that resets WebSocket sessions on :7070, so the reconnect
schedule this project owns (`sdk_dependency.md`'s fork modification 2)
reaches across the seam inside a single control flow. The vendor's other
axis, quote versus trading, is a naming seam only — same concurrency regime,
and the result contract, symbol encoding, normalization, partial-failure
representation, re-attempt budget and paging are all shared, so splitting
there would either duplicate them or require a fourth shared file. The
caller-facing surface was never in question: `architecture.md` fixes this as
the single caller-facing layer and Constraints as the only importer of
`dbsec_sdk`.

---

## Scope

Endpoints reached through this module are limited to those with an actual
consumer in this system, not the vendor's full surface. The orderbook
(호가) call is included regardless of an immediate consumer, so that a
bid/ask need can be served without reopening this layer.

WebSocket subscription of the realtime orderbook stream (V61) is deliberately
NOT included: `bid_ask_snapshots`' two sources are the REST orderbook call and
the trade stream's own level-1 piggyback, and the per-connection subscription
budget is spent on price tracking.

That exclusion rests on a budget premise, and the premise is now separable:
`auxiliary_stream.md` runs non-trading subscriptions on a different account
with its own session allowance. V61 is recorded there as a known future
consumer. This does not admit it into scope here — the exclusion above stands
until someone designs the consumer and its storage — but it records where the
question would be reopened.

---

## Call-Point Inventory

Derived from the caller specs' stated needs, not from the vendor's catalogue —
which is why the list is shorter than what the vendor publishes (11 quote
endpoints plus 9 trading endpoints for overseas equities). TPS is the vendor's
published per-endpoint rate FOR ONE APP — see the sixth note below for what
two apps make of it. Every consumer is now bounded against these numbers and
the call-budget open item is closed, so the column is a standing reference for
future call sites rather than an input to an unresolved allocation.

| Need | Call site | Endpoint | TPS |
|---|---|---|---|
| Tradable ticker list | Session Lifecycle Step 1; `build_trading_api_symbol_map()` (three per-exchange calls) | `quote/…/inquiry/stock-ticker` | 2 |
| Balance and deposit | Step 1b; the funds gate; Broker Reconcile | `trading/…/inquiry/balance-margin` | 3 |
| Per-ticker margin ratio | position sizing — undesigned, see `open_items.md` | `trading/…/inquiry/able-orderqty` | 2 |
| Bars, minute OR second resolution | Watchdog Step 2a (prefix scan, rotation window, backfill) | `quote/…/chart/min` | 4 |
| Ticks | Watchdog Step 2b; Exit Architecture's REST tick backstop (WS-dead only) | `quote/…/chart/tick` | 4 |
| Bulk price snapshot | `bulk_fetch_today_first_price()` | `quote/…/inquiry/multiprice` | 2 |
| Level-1 orderbook | the exit ladder; `signal_time_rest` | `quote/…/inquiry/orderbook` | 2 |
| Order submit, amend, cancel | Step 5c; Position Manager; Shutdown; Broker Reconcile | `trading/…/order` | 10 |
| Filled and outstanding orders | the exit-side backstop | `trading/…/inquiry/transaction-history` | 2 |
| WebSocket session reset | the reconnect path | `ws_common/ws_session_disconnect` | 1 |
| Realtime trade stream | Exit Architecture's tick handler; the watchdog scan's ranking signal | WebSocket `V60` | — |
| Realtime order/fill stream | in-flight order tracking | WebSocket `IS2` | — |

Six things the table settles that prose had left ambiguous:

- **Cancellation is not its own endpoint.** It is `trading/…/order` with
  `OrdTrdTpCode=2` and `OrgOrdNo`, so cancels draw on the SAME TPS-10 bucket
  as submissions rather than a separate one.
- **Filled and outstanding are one endpoint.** `inquiry/transaction-history`
  covers both, so the exit-side backstop's paired inquiries cost less than a
  count of calls suggests.
- **`chart/min` carries SECOND resolution as well as minute.**
  `InputDivXtick=1` returns second-granularity rows from the same endpoint —
  therefore the same TPS bucket — and `dataCnt` reaches 2000 rows in one
  response, costing receive time rather than TPS (measured; that check is
  retired as `api_contract_checklist.md` T-14 — this entry is its record).
  Second resolution is what lets the watchdog loop observe a FORMING
  bar, which minute mode does not return at all; inverted, a completed bar
  never needs it, which is why backfill runs in minute mode where one call
  covers a session. The loop's cost against this bucket is D+1 calls per
  cycle, not the working set (`live_mode_runner.md`'s prefix scan), and the
  mid-bar scan and the bar-close fetch share the bucket.
- **The two WebSocket streams are separate rows because they are separate
  connections.** `api_contract_checklist.md` T-6 measured that subscription
  types are mutually exclusive per connection; V60 registers per ticker
  (`tr_type` 1/2) and IS2 registers per account (`tr_type` 3, no `tr_key`).
  They also back different checklist rows — T-1 and T-7 respectively.
- **Session reset is already in the SDK**, exposed as
  `client.apis.ws_common.ws_session_disconnect()`, so this project does not
  build the call. Its scope is every session for the token's account, so
  recovering one connection resets the other.
- **The TPS column is per APP, and TWO apps are in use.** `chart/min` and
  `inquiry/orderbook` return identical data on the demo and production
  accounts, and each account carries its own allowance at both the app and the
  endpoint tier (measured; that check is retired as
  `api_contract_checklist.md` T-15 — this entry and `sdk_dependency.md`'s
  rate governance note are its record). ONLY THE `quote/` ENDPOINTS DOUBLE.
  The endpoint prefix in the table is what sorts them: a `quote/` call
  returns the same data whichever account asks, so either account can serve
  it, while a `trading/` call is scoped to the account that owns the
  positions and the cash. Placing an order on the demo account does not add
  order capacity — it places an order in a paper account.

```
  quote/  (doubles)     chart/min 8   chart/tick 8   orderbook 4
                        multiprice 4  stock-ticker 4
  trading/ (does not)   order 10      transaction-history 2
                        balance-margin 3   able-orderqty 2
```

  The watchdog scan's slot allocation is written against the doubled
  `chart/min` figure. Note the buckets with contending consumers: the exit
  ladder re-quotes round-robin inside `orderbook`'s 4, and `signal_time_rest`
  preempts it at bar close; inside `chart/min`'s non-reserved half, carryover
  and promotion share the cap with the bar-close superset-K fetch, which
  preempts both. Position Manager Loop is no longer among the `chart/min`
  consumers — its per-position bar fetch went with `bars_since_entry`
  (live_mode_runner.md). It does draw `chart/tick`: the WS-dead exit backstop,
  and, at stage `shadow`, the ticker-scoped 1-tick buffer the fill
  simulations run over.

**Leg assignment, and the one rule that decides it.** A `trading/` call is
scoped to the account that owns the positions and the cash, so it has no
choice of leg. A `quote/` call has a free choice, since either account
returns the same data. That asymmetry loads the two apps very differently
against the app-level tier, because only production carries order traffic:

```
production leg    all trading/ calls  +  the scan's reserved half
demo leg          everything else quote/
```

Two `quote/` consumers are exceptions and stay on BOTH legs:

- **The watchdog scan.** Its 850ms bound comes from taking t=0 and t=250 on
  each leg; confining it to one doubles its pacing interval and the bound
  breaks. Half stays on production by necessity.
- **The exit ladder's orderbook round-robin.** Pinning it to demo alone
  would cap it at that leg's endpoint ceiling of 2/s, and at
  `max_tickers`'s default that returns re-quote latency to exactly what the
  original per-cycle design had — giving back the improvement for nothing.
  It runs on both legs and YIELDS under the dispatch priority below when
  production is busy, which is strictly better than a static assignment: it
  degrades only while contended instead of always.

**Dispatch priority is EARLIEST DEADLINE FIRST, and no class list is
needed.** Every request carries the deadline it must meet, and ordering
falls out of that:

| Request | Deadline |
|---|---|
| Exit submission (tp/sl) | immediate |
| Entry submission | t bar open + `execution.entry_fill_delay_seconds` |
| Ghost / stuck-order cancel | 60s |
| Ladder amend | next re-quote |

Slack is `deadline - now - expected RTT`; a request with slack yields to one
without. This is what lets an entry burst and the scan share the order and
`chart/min` endpoints without either being statically starved. A ladder
amend dropped under pressure is not a loss — the next rotation visit
re-quotes from a fresher book, so discarding it is the correct handling
rather than a degradation to be avoided.

The queue belongs HERE and not in the SDK: its pacer is instance-internal
and cannot be reordered from above, and `sdk_dependency.md`'s standing rule
is that anything solvable in this module is not solved by a fork.

**This module owns dispatch ORDER, not the structure that consumes it.** The
mechanics past dispatch are already described above under Async Boundary —
each REST call goes through `asyncio.to_thread`, and rate waits and retry
backoff block a pool thread — so they are not out of scope here. What is
undesigned is how that meets the caller's loop structure, and that is the
async boundary open item's question rather than this one. The queue answers
only WHICH call is handed over next.

Entry submission is ordered by `execution.entry_fill_delay_seconds` but is
NOT hard-gated on it: an entry past its reference time is still submitted
and counted (`live_scan_daily.entry_submit_late`). The value is the
reference both engines share — BacktestEngine models the fill at it — not a
cancellation threshold.

---

**Two needs are NOT reachable here, and their absence is the finding.**
Trading-halt status has no endpoint in the vendor's catalogue at all, so
`utils.query_halt_status()` is not a call of this module and its source is
tracked in `open_items.md`. A server-clock endpoint is likewise absent, which
is what `api_contract_checklist.md` T-11 asks about; the nearest available
substitute is the broker-stamped timestamp on every order and fill (below).

---

## Layer Responsibility Split

Four tests, applied in order. The first three restate this file's Role
sentence; the fourth is derived from `utils.query_halt_status()`'s statement
that each call site interprets `None` per its own context.

1. Is it a transport mechanism? → **SDK**
2. Is it vendor-shaped, and must a caller never see it? → **this module**
3. Is it a policy the SDK has no concept of, whose answer is the same for
   every caller? → **this module**
4. Does the answer vary by caller context? → **caller**

| Axis | SDK | This module | Caller |
|---|---|---|---|
| Paging | `cont_yn`/`cont_key` mechanism | pages to a caller-supplied bound | supplies the bound |
| Retry | per-call classification and backoff | above-SDK re-attempt budget | when to degrade |
| Partial failure | — | the representation | what "absent" means |
| Degraded mode | — | — | interpreting `None` |
| Session-phase order types | — | validation before the call | the table; sizing |
| Order identity | — | vendor format → opaque handle | tracking, idempotency |
| WebSocket budget | connect, reconnect | connection allocation under T-6 exclusivity | which tickers |

**Paging is decided by this file's own boundary test, not by preference.**
Returning one page plus a continuation handle would put `cont_key` in a
caller's hands, and `cont_key` is one of the four things the test names. So
this module pages until a caller-supplied bound is covered and returns a
completed set. Bounds accept both fill-ID and timestamp, and both upper and
lower, so one method serves the exit backstop's "newer than last seen",
Broker Reconcile's "today's fills", and any bounded window; time bounds map
onto the vendor's own `InputDate1`/`InputDate2`. This is robust to the
varying page size that was the fill-inquiry open item's central worry — an
item now closed, its distribution accumulating in `live_scan_daily`'s
`fill_page_rows_p50`/`_p95` during ordinary operation rather than through a
separate measurement exercise. Note that an ID bound inherits `api_contract_checklist.md`
T-7's risk — fill-ID ordering is exactly what that grade A row questions —
while a time bound does not.

**Retry has three layers, not two.** The SDK owns per-call retry
classification and backoff execution. Above it sits the question of whether a
call that exhausted the SDK's retries is re-attempted at all, which the SDK
has no concept of. The MECHANISM is this module's; the VALUE is the caller's,
supplied per call as a re-attempt budget defaulting to ZERO. That follows the
cascade honestly rather than forcing test 3: re-attempt need is not uniform,
because a cyclic caller's next cycle IS its retry — the watchdog loop at
`poll_interval_seconds` and the Position Manager at
`position_check_interval_seconds` both pass 0, since re-attempting inside the
call only spends the same rate budget earlier. One-shot callers with no next
cycle — Session Start Probes, the crawler's universe-scale paths — are the
only ones with a reason to pass non-zero. A re-attempt re-enters through the
SDK and is therefore paced by the rate governor the SDK already owns; no
second pacer appears. The result carries the attempt count, so how many
attempts happened stays observable rather than hidden by a second layer.
**The re-attempt policy itself is not designed** — ownership is settled here,
the policy is not.

**Writes are never re-attempted, regardless of the budget a caller passes.**
This is a hard rule, not a default: the vendor exposes no idempotency key and
`live_positions` writes its `'pending'` row at SUBMISSION time, so a
submission that failed locally but landed server-side would produce an order
the system does not track. Recovery for writes belongs to the paths that
already exist — the vanished-order rule and Broker Reconcile.

**Session-phase order types stay in `execution_common.md`.** Concealment was
never available: `check_funds_available()` and `simulate_entry_fill()` both
size against the market-BUY-to-limit conversion, and the latter is
BacktestEngine's, which never reaches this layer. The two do not size
IDENTICALLY — `simulate_entry_fill()` cannot read
`live_mode.market_buy_price_margin` and approximates the conversion from bar
data, an asymmetry `execution_common.md` states outright and `open_items.md`
carries — but that sharpens this point rather than weakening it: the backtest
side needs these semantics and has no way to obtain them from this layer. The
boundary test forbids
leaking vendor REPRESENTATIONS; it does not license hiding vendor SEMANTICS
with an economic consequence. What this module owns is validation — refusing
an order type the current session phase does not permit, before it reaches
the vendor, because letting it through returns an `rsp_cd` the caller would
have to decode.

---

## Response Normalization

Every field is declared, and declaration puts it in one of three categories.
A blanket "convert what looks numeric" is not available: it would turn
`rsp_cd`'s `"00000"` into `0` and break the Result Contract's own comparison.

- **CONVERT** — declared with a target type. Prices and amounts are `float`;
  quantities are `int`; order identifiers become strings, this system's
  canonical form for an opaque handle.
- **PRESERVE** — declared, passed through unchanged: `rsp_cd`, `rsp_msg`,
  `AstkRjtCode`, `AstkRjtRsnCnts`, symbols.
- **DROP** — undeclared. Dropped rather than passed through: passing through
  lets a caller observe vendor forms, while rejecting would break production
  on a harmless vendor field addition. Dropping avoids both, and repeats one
  level down the posture Scope already takes toward endpoints.

**The wire type is not trusted.** The vendor returns the same doc-declared
`number` both ways — the deposit-detail call returns
`"AstkDps0": "348555423.000000"` as a quoted string while the order call
returns `"OrdNo": 14` as a JSON number. Neither `api_doc/`'s type column nor
the JSON type is a contract, so every CONVERT field accepts string or number
and parses to its declared target.

**An empty string maps to `None`, never to `0`.** Per-field semantics for
`""` are not knowable, `None` propagates as a visible absence while `0.0`
propagates as a plausible wrong number, and the schema already accepts it
(`exit_price DOUBLE`, `limit_price DOUBLE` NULL for market orders). The
vendor's own use of `0` as a REAL value settles it: `AstkOrdPrc` is
documented as `0` for a market order, so conflating `""` with `0` would make
a market order indistinguishable from an unknown price.

**Quantities are `int` in both directions**, and the integrality check is the
guard that makes that safe rather than a feature handling an expected case.
The vendor sends `"1.000000"`-shaped strings, so a safe `int()` must parse as
float and assert integrality first — the check costs the same under either
target and is therefore not a reason to prefer `float`. Given equal cost,
`int` wins because `db_schema.md` is `INTEGER` throughout: emitting `float`
would put a float-to-int conversion at every writer, each a site where
truncation could later be introduced. A non-integral value is an ERROR — a
wrong field mapped, a vendor change, or a parse fault. `Decimal` is NOT
introduced: `db_schema.md` uses `DOUBLE` 61 times and `DECIMAL` zero times,
and normalization inherits that rather than inventing a type policy.

**Temporal fields are per-format, and none is an identity conversion.** Four
vendor forms exist against four schema forms that match none of them
directly: `YYYYMMDD` (`OrdDt`, `SettDt`), `HHMMSSmmm` at length 9
(`TrdTime`), `YYYYMMDDHHMMSSmmm` at length 17 (`AstkOrdDttm`, `AstkExecDttm`
and their `Lcl` twins), and the chart endpoints' separate `Date` plus `Hour`.
The schema is second-precision throughout, so the 17-character form is split
— and MILLISECONDS ARE CARRIED by this layer rather than truncated at it.
Truncation is the DB writer's act; destroying data at the boundary is worse
than carrying it.

**Zone mapping is fixed here so it cannot be guessed at a call site.** The
`Lcl`-prefixed field is America/New_York and the unprefixed one is Korea.
Everything reaching the schema uses the `Lcl` side, since the whole spec set
operates in America/New_York and choosing wrong shifts every timestamp by
13–14 hours silently. The unprefixed side is not dropped: if the pair is two
different clocks — broker-stamped versus venue-stamped — rather than one
instant in two zones, its delta is a latency observable in the manner of
`bar_latency_daily`. That also gives a broker-side clock reference on every
order and fill, which is the nearest thing available to the server-clock
endpoint `api_contract_checklist.md` T-11 wants and the vendor does not
publish.

**All of this applies to WebSocket bodies too**, which is where the boundary
test's own examples live: V61 carries `"FNTSLA"` as `tr_key` and IS2 carries
`"Sastkordprc": "00000000009.4300"` — two of the four named forms, both
arriving over WebSocket. IS2 also carries the
`Sastkorddttm`/`Sastklclorddttm` pair at length 17, and its own example shows
`"Sastkexecdttm": ""` on a not-yet-filled order, which is exactly the case
that maps to `None`.

---

## Configuration Ownership

One rule: **a value the vendor or the SDK defines is either not configuration
at all or belongs to the SDK's own `Config`; only values this system decides
live in `pipeline_config.yaml`, and nothing is duplicated across the two
surfaces.**

Applying it:

- Every endpoint URL key is DELETED — addressing is the SDK's, and this
  file's boundary test puts it on the far side from callers. That removes
  `live_mode_runner.md`'s `trading_api_url`, `trading_api_ticker_url` and
  `margin_ratio_url`, and `metadata_crawler.md`'s `trading_api_quotes_url`.
- `bulk_api_chunk_size` and `api_max_tickers_per_second` are DELETED. The
  endpoint payload cap and the rate ceilings are published values the SDK
  already carries; estimating them on this side duplicated the vendor.
- What survives on this side is the SDK configuration file path and the token
  cache path, both PER ACCOUNT and both this project's to choose — see
  `sdk_dependency.md`, whose fork modification 3 makes the cache path
  injectable.
- Policy keys stay, because they express how much of a vendor allowance we
  elect to use rather than what the allowance is. `execution.ws_ticker_limit`
  is the example, and it must not exceed `DBSecWebSocket._MAX_SUBSCRIPTIONS`,
  which is 50 and raises above it.

---

## Constraints

- No module other than this one imports `dbsec_sdk`
- `is_ok` and `__bool__` never cross this boundary
- Caller specs state intent and cite this module; they do not restate call
  mechanics. Inline call descriptions still present in caller specs are
  placeholders to be replaced, not a second source
- Vendor documentation (`api_doc/`) and the SDK repository are external
  read-only material and are not part of the Spec File Structure
- Order submission, amendment and cancellation are never re-attempted at this
  layer, whatever re-attempt budget a caller passes
- `execution.ws_ticker_limit` must not exceed `DBSecWebSocket._MAX_SUBSCRIPTIONS`
  (50). That is an SDK-side constant rather than a vendor-documented one, so
  it is a re-sync surface: an upstream change to it silently changes our
  ceiling
- WebSocket connect is paced at 6 TPM by the SDK's own instance-level limiter,
  i.e. one connect per 10s. The reconnect schedule `live_mode_runner.md` owns
  is read against that floor — a burst faster than 6/min is absorbed by the
  SDK rather than reaching the server
