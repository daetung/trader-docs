# Tool: Auxiliary Stream Host

**File:** `tools/collect_auxiliary_stream.py`
**Standalone CLI tool — runs alongside a live session, not inside it**

---

## Role

Hosts WebSocket subscriptions that are **not part of trading** and cannot fit
inside the live session's own budget. Delayed-quote capture, for feed-coverage
measurement, is its first consumer — not its definition.

The defining property is where the subscriptions live: a separate 모의투자
account with its own app key, its own token cache, and its own session
allowance, driven from a separate OS process. Production keeps its two
sessions for V60 and IS2, which `api_contract_checklist.md` T-6 measured as
necessarily separate connections.

Named for that property rather than for the first consumer. A name drawn from
coverage measurement would have become false the moment a second consumer
arrived, and a name is the costliest thing to change by patch.

---

## Why a separate account

Two limits stack, and only the account split clears both.

**Sessions are per app.** The SDK's own hint for error 10011 reads
`앱·계좌당 최대 2세션`. Production's two are committed. An app registration is
effectively an account — `prd_app_key` and `vtl_app_key` are separate
registrations the vendor treats as separate accounts — so a second account
carries its own two rather than sharing production's. The vendor allows 30
apps per IP for production plus 30 for demo, so this does not approach a
ceiling (`sdk_dependency.md`).

**Token caches would otherwise collide.** The SDK's cache is one file whose
mode it validates by content: a mismatch returns `None`, `auto_token` then
issues, and the save overwrites, so two clients in different modes erase each
other against a one-per-minute issuance limit. `sdk_dependency.md`'s fork
modification 3 makes the cache path injectable, and this process supplies its
own.

---

## Why a separate process

Not for the cache — fork modification 3 already handles that. **For the event
loop.**

`trading_api.md`'s Async Boundary states that a blocking WebSocket callback
stalls reception for every subscription on that connection. Two clients
sharing one event loop would extend that across connections, so this process's
buffering and file writes could stall production's tick handler — which is the
2-print guard's path, and therefore a direct trading risk. Separate processes
also make the SDK's `threading.Lock` refresh guard irrelevant rather than
misleading.

Rate governance does **not** force separation: the SDK's rate controller is
per client and the two accounts carry different app keys, so their server-side
buckets are independent.

---

## Consumers

### Delayed quote capture (V10/V11) — the first consumer

The vendor's free real-time entitlement delivers roughly half the trade
prints; the base entitlement is 15 minutes delayed but carries **all** of
them, and needs no application. The delayed stream is therefore the only
source that can say which prints the live stream is missing —
`bid_ask_snapshots` and `tick_10` come from REST polling and carry their own
sampling axis, so they can establish that prints differ but never which are
absent.

- **Subscription set mirrors production's V60 set**, derived from
  `live_positions` rather than through a new coordination channel: Exit
  Architecture subscribes on entry fill and unsubscribes on exit, so the
  subscribed set is the set of open positions. The same
  `execution.ws_ticker_limit` bound therefore governs both sides without being
  restated.
- **One session only.** Production is already subscribed to V60 for these
  tickers, so duplicating a realtime subscription here would add nothing. That
  also leaves the demo account's second session free, and means T-6's
  unanswered quote-versus-quote exclusivity question does not arise.
- **`RealYn`** (0: delayed, 1: realtime), present in both V60 and V61 bodies,
  is a label of which stream is being received — not a health signal.
  Entitlement lapse needs no detector: the vendor notifies separately.

### Realtime orderbook (V61) — known next consumer

`trading_api.md`'s Scope excludes V61 from production for a budget reason —
the per-connection subscription budget is spent on price tracking — and that
reason is exactly what this process removes, since its budget is a different
account's. Recorded so the extension axis is visible as real rather than
speculative. It is **not** admitted to scope here: the exclusion stands until
someone designs the consumer and its storage. It would consume the demo
account's remaining session; a third consumer would need a third account.

---

## Output

Per-second aggregates, written **append-only** as JSONL, one file per day:

```
data/feed_coverage/live/YYYYMMDD.jsonl        ← written by LiveModeRunner
data/feed_coverage/delayed/YYYYMMDD.jsonl     ← written by this process
```

The directory names the **analysis**, not the writing process, so the two
sides of one comparison sit together. A future consumer such as V61 gets its
own purpose-named directory. Each file has exactly one writer, so DuckDB's
per-file single-writer constraint (`live_mode_runner.md`'s P-5) is not
engaged by either — these are not database files.

Recorded per `(ticker, second)`: print count, price min and max, and summed
`exevol`. **Print-level matching is impossible and is not attempted**: the
V60 body carries no execution ID and `loctime` is second-granularity, so two
prints in the same second at the same price and size are indistinguishable.
Second-bucket comparison is enough to detect bias.

JSONL rather than Parquet, deliberately. Parquet is columnar with a footer
written at close, so it cannot append and a five-second flush would rewrite
the whole file each time. The compression advantage that would otherwise
favour it dissolves because raw is purged after analysis — retention is one
day, or two while a retry is pending.

**Flush cadence.** Callbacks mutate an in-memory dict only; the flush writes
on a simple cadence. Only **sealed** seconds are written — those strictly
older than the current wall-clock second — so a second is never emitted twice
in partial form. That matters because the comparison counts prints per
second: a split row would read as two seconds at half the density each, which
is exactly the shape systematic omission produces.

**Unflushed aggregates are lost on abrupt process death, accepted
deliberately.** This data characterises the feed rather than driving a
decision, and a partial day does not invalidate the statistic.

---

## Timing asymmetry

Rows here arrive roughly fifteen minutes after the live rows for the same
second. Any consumer joining the two must treat a missing delayed row for a
recent second as **not yet arrived**, not as absent — the join is meaningful
only for seconds older than the delay.

Production's session ends at `after_hours_end(today)`; this process runs on
until the delayed tail is complete, roughly fifteen minutes later. The evening
batch's analysis stage (`metadata_crawler.md`) clears that by about an hour.

---

## Cron / Scheduler Setup

```bash
# Auxiliary stream host — starts with the session, outlives it by the
# delayed stream's lag
0 4 * * 1-5 cd /path/to/stock-scalping && \
    source .venv/bin/activate && \
    python tools/collect_auxiliary_stream.py \
        --consumer delayed_quote >> logs/auxiliary_stream.log 2>&1
```

---

## Config Keys

```yaml
auxiliary_stream:
  enabled:            true      # master switch; false disables the process
                                # and the health-report observation section
  sdk_config_path:    "configs/dbsec_demo.yaml"
                                # the demo account's SDK Config — this
                                # project owns the location and injects it
                                # (sdk_dependency.md)
  token_cache_path:   "data/.dbsec_token_demo.json"
                                # injected via fork modification 3; MUST
                                # differ from production's, or the two
                                # clients erase each other's cache
  flush_seconds:      5         # sealed-second flush cadence
  tail_minutes:       20        # how long to run past session end, covering
                                # the delayed stream's ~15-minute lag
```

---

## Constraints

- Runs in its own OS process — never inside LiveModeRunner's event loop
- Uses a 모의투자 account distinct from production's, with its own app key,
  SDK config path and token cache path
- Trades nothing, holds nothing, and submits no orders. It is not
  multi-account trading, which `sdk_dependency.md` keeps out of scope
- Writes only to `data/feed_coverage/<side>/`; issues no DDL, opens no
  database connection, and writes no `batch_runs` row — the evening analysis
  stage owns that marker
- Its subscription set is derived from `live_positions`, read-only
- Never subscribes V60: production already holds that subscription for the
  same tickers
- A new consumer is added here rather than to production's session; if it
  would exceed the demo account's two sessions, it needs its own account
