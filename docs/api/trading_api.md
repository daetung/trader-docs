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

---

## Scope

Endpoints reached through this module are limited to those with an actual
consumer in this system, not the vendor's full surface. The orderbook
(호가) call is included regardless of an immediate consumer, so that a
bid/ask need can be served without reopening this layer.

WebSocket subscription of the realtime orderbook stream is deliberately NOT
included: `bid_ask_snapshots`' two sources are the REST orderbook call and
the trade stream's own level-1 piggyback, and the per-connection
subscription budget is spent on price tracking.

---

## Not Yet Designed

Recorded so the boundary of what is settled stays visible. Each is an open
item — see `open_items.md`.

- **Call-point inventory.** Which endpoints, with which arguments, derived
  from the caller specs' stated needs rather than from the vendor's catalogue.
- **Layer responsibility split.** Which axes this module absorbs versus
  leaves to callers, beyond the result contract and symbol encoding settled
  above.
- **Response normalization.** The vendor returns every numeric as a string,
  order fields as fixed-width zero-padded values, and some fields as an
  empty string; the spec set assumes numeric types throughout.
- **Call budget allocation.** Per-endpoint rate limits are published and the
  SDK paces against them, but how the global budget is divided among the
  watchdog loop, the exit-side backstop, halt checks and orderbook snapshots
  is a single allocation problem, not a per-consumer one.
- **Whether this module stays one file.** The domain surface may split along
  the vendor's own quote/order/realtime axis; deferred until the call-point
  inventory exists, since fixing a file boundary first would only require
  redrawing it.

---

## Constraints

- No module other than this one imports `dbsec_sdk`
- `is_ok` and `__bool__` never cross this boundary
- Caller specs state intent and cite this module; they do not restate call
  mechanics. Inline call descriptions still present in caller specs are
  placeholders to be replaced, not a second source
- Vendor documentation (`api_doc/`) and the SDK repository are external
  read-only material and are not part of the Spec File Structure
