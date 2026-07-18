# Module: BacktestEngine

**File:** `src/backtest/engine.py`
**Depends on:** `docs/data/data_boundary.md`

---

## Role

Simulate trading performance of the trained model on the test split.
Compute winning rate and trade statistics using realistic position management,
slippage approximation, volume-capped partial exit simulation, and position sizing rules.

BacktestEngine accesses OHLCV and tick data directly from DuckDB —
callers do not pass chart data as arguments.

---

## Input

```python
predictions: pd.DataFrame
    # columns: [ticker, date, hour, p_entry,
    #           prob_up5, prob_up3, prob_sw, prob_dn3, prob_dn5,
    #           label_up5, label_up3, label_sw, label_dn3, label_dn5]
    # one row per entry point candidate in test split

config: dict
db_conn: duckdb.DuckDBPyConnection   # injected via constructor
```

OHLCV and tick data are fetched internally from DuckDB per ticker/date as needed.

---

## Class Interface

```python
class BacktestEngine:
    def __init__(
        self,
        config: dict,
        db_conn: duckdb.DuckDBPyConnection,
    ): ...

    def run(
        self,
        predictions: pd.DataFrame,
    ) -> tuple[pd.DataFrame, dict]:
        """
        Run full backtest simulation.
        Returns (trade_log_df, summary_dict).

        trade_log_df: one row per executed trade — matches trade_log table schema.
                      Written to DuckDB trade_log table by Backtester (run_backtest.py),
                      not by BacktestEngine itself.

        summary_dict keys:
            winning_rate        : float  — winning_trades / total_trades
            total_trades        : int
            winning_trades      : int
            avg_pnl_pct         : float
            total_pnl_abs       : float
            avg_slippage_pct    : float
            dead_position_count : int
            dead_position_rate  : float
            suppressed_count    : int    — entries blocked by suppress_threshold
            trades_by_signal    : dict   — {signal: {count, win_rate, avg_pnl}}
            trades_by_exit      : dict   — {exit_reason: count}
        """
        ...
```

---

## DB Access Strategy

### OHLCV (per ticker)
```sql
SELECT * FROM ohlcv_1min
WHERE ticker = ?
  AND date IN (entry_dates_for_this_ticker)
ORDER BY date, hour
```
Loaded once per ticker at the start of that ticker's processing.
Covers entry dates only — not the full dataset.

### Tick data (per ticker/date — full day)
```sql
SELECT * FROM tick_10
WHERE ticker = ? AND date = ?
ORDER BY hour, seq_id
```
Loaded once per ticker/date combination.
Full day loaded (including after-market) — filtered in memory by hour range:
- Entry slippage: `hour >= entry_hour AND hour < hour_add_seconds(entry_hour, 100)`
- Exit tracking: `hour >= entry_hour onward` (passed as ticks_future to track_price_breach)
- Partial exit: passed as ticks_exit to simulate_exit_fill (breach bundle onward, full day)

### Trading halts (per ticker/date)
```sql
SELECT * FROM trading_halts
WHERE ticker = ? AND date = ?
```
Loaded once per ticker/date alongside OHLCV and tick data.
Passed explicitly to all internal methods.

### Trading calendar (for dead position resolution)
```sql
SELECT * FROM trading_calendar
WHERE date > ? AND has_data = TRUE
ORDER BY date LIMIT 1
```
Filters to next date where `has_data = TRUE` — consistent with Labeler dead position logic.

### Ticker data coverage (for dead position Case B)
```sql
SELECT 1 FROM ticker_data_coverage
WHERE ticker = ? AND date = ?
```

### Corporate events (for dead position Case A/D dividend/split adjustment)
```sql
SELECT event_type, value FROM corporate_events
WHERE ticker = ? AND event_date > ? AND event_date <= ?
```
Queried only when Case A/D resolution is reached — not on every trade.
Same window and rationale as Labeler's equivalent lookup (see `05_labeler.md`).

For each (ticker, date) group, the following data is loaded from DuckDB once
and passed down to internal methods:

```python
# Loaded once per ticker
ohlcv_ticker = db_conn.execute("""
    SELECT * FROM ohlcv_1min
    WHERE ticker = ? AND date IN (?)
    ORDER BY date, hour
""", [ticker, *dates]).df()

# Loaded once per ticker/date
ticks_td = db_conn.execute("""
    SELECT * FROM tick_10 WHERE ticker = ? AND date = ?
    ORDER BY hour, seq_id
""", [ticker, date]).df()

halts_td = db_conn.execute("""
    SELECT * FROM trading_halts WHERE ticker = ? AND date = ?
""", [ticker, date]).df()

# halts_td passed explicitly to all internal methods
fill_second  = utils.hour_add_seconds(entry_hour, 5)
search_limit = utils.hour_add_seconds(entry_hour, 100)
ticks_entry  = ticks_td[(ticks_td["hour"] >= entry_hour) &
                         (ticks_td["hour"] < search_limit)]

fill_idx, prev_bundle, fill_bundle = utils.find_fill_bundle(ticks_entry, fill_second)
fill_price = utils.interpolate_bundle_price(prev_bundle, fill_bundle, fill_second) \
             if fill_bundle is not None else p_entry

direction, exit_price, exit_hour, is_ambiguous = utils.track_price_breach(
    ohlcv_future=bars_from_t,
    ticks_future=ticks_td[ticks_td["hour"] >= entry_hour],
    fill_price=fill_price,
    fill_second=fill_second,
    threshold_up=tp_pct,
    threshold_dn=config["backtest"]["stop_loss_pct"],
    exit_interpolation=config["backtest"]["exit_interpolation"],
)
```

---

## Position Sizing

Sizing moved to `docs/utils/execution_common.md`'s `compute_position_size()`
— shared with LiveModeRunner. BacktestEngine's role is to supply the two
`balance`-adjacent values correctly:

```
initial_cash:   float   # from config; fixed for the entire run — the
                        # `balance` argument to compute_position_size().
                        # 0 = unlimited (inf mode): position_size_cash_pct
                        # leg is skipped, sizing is vol_based-only.
remaining_cash: float   # runtime value, starts at initial_cash, decremented
                        # by check_funds_available()'s caller as trades fill
                        # (or partially decremented — see
                        # execution.use_all_cash). NEVER passed as
                        # `balance` — see execution_common.md's Constraints
                        # for why sizing and funds-availability must not
                        # share the same value.
```

`inf mode` (`initial_cash == 0`): `check_funds_available()` is not called —
there is no remaining_cash to exhaust.

The actual call sequence (`compute_position_size()` →
`check_funds_available()` → `simulate_entry_fill()`), including which price
each step uses, is written out once in full in "Entry Slippage Model"
below rather than duplicated here — sizing and entry-fill simulation are
one continuous flow, not two independent steps.

---

## Entry Slippage Model

Simulate partial entry fills at t bar open + 5s using tick_10 data —
mirrors the exit side's `simulate_exit_fill()` rather than assuming a
single instant fill.

```
tick_10.hour = last tick timestamp of each 10-tick bundle (second precision)
fill_second  = utils.hour_add_seconds(entry_hour, 5)
search_limit = utils.hour_add_seconds(entry_hour, 100)

Rationale for 100s window:
    A 10-tick bundle represents 10 price ticks. In low-volume or post-halt
    scenarios, 10 ticks can span up to ~100s (1 tick/10s worst case).
    Extending beyond the 1-minute t bar boundary captures the fill bundle
    in these edge cases. Beyond 100s, p_entry fallback is preferred.

Procedure:
    ticks_t = tick_10 rows where hour >= entry_hour AND hour < search_limit
              sorted by (hour, seq_id)

    fill_idx, prev_bundle, fill_bundle = utils.find_fill_bundle(ticks_t, fill_second)

    # Sizing uses p_entry, not a post-slippage price — see execution_common.md's
    # Constraints for why this ordering is required (simulate_entry_fill()
    # takes quantity as input, so quantity cannot wait for its own output).
    quantity = execution_common.compute_position_size(
        balance=initial_cash, fill_price=p_entry, t_bar_volume=t_bar_volume,
        ticker_notional=..., total_notional=...,
        position_size_cash_pct=config["execution"]["position_size_cash_pct"],
        position_size_vol_pct=config["execution"]["position_size_vol_pct"],
        per_ticker_share_cap_pct=config["execution"]["per_ticker_share_cap_pct"],
        exposure_cap_pct=config["execution"]["exposure_cap_pct"],
    )
    proceed, quantity = execution_common.check_funds_available(
        quantity, p_entry, remaining_cash,
        use_all_cash=config["execution"]["use_all_cash"],
    )
    if not proceed:
        → skip entry (insufficient funds even sized down, or use_all_cash=False
          and full quantity unaffordable)
    remaining_cash -= quantity * p_entry   # provisional; see Cash deduction
                                            # order note below for the
                                            # use_all_cash interaction

    limit_price = None if config["execution"]["entry_order_type"] == "market" \
        else (p_entry * (1 + config["execution"]["entry_gap_value"])
              if config["execution"]["entry_gap_type"] == "percentage"
              else p_entry + config["execution"]["entry_gap_value"])

    if fill_bundle is not None:
        fill_price, total_filled, unfilled_qty, status = execution_common.simulate_entry_fill(
            ticks_entry=ticks_t, ohlcv_entry=bars_from_t, quantity=quantity,
            fill_bundle_idx=fill_idx, p_entry=p_entry,
            buy_rate=config["execution"]["buy_rate"],
            halts_df=halts_td, cancel_after_seconds=config["execution"]["cancel_after_seconds"],
            limit_price=limit_price,
        )
        if total_filled == 0:
            # Fully unfilled — logged as its own trade_log row rather than
            # silently dropped (see db_schema.md's exit_reason='entry_canceled'):
            # trade_log row: exit_reason="entry_canceled", quantity=0,
            #                fill_price=p_entry, exit_bar=entry_bar,
            #                pnl_pct=NULL — skip remaining Exit Logic entirely
            #                for this candidate, proceed to next candidate
        else:
            # total_filled may be < quantity (order canceled after
            # cancel_after_seconds) — trade proceeds sized down to total_filled,
            # no penalty (see execution_common.md Constraints: entry-side
            # unfilled remainder is canceled, not penalized, unlike exit-side).
            quantity = total_filled
    else:
        fill_price = p_entry    # no bundle within 100s — zero slippage fallback
        # quantity unchanged — full requested size, filled at p_entry

slippage_pct = (fill_price - p_entry) / p_entry
```

---

## Entry Decision Logic

```python
from execution_common import resolve_signal

threshold          = config["backtest"]["entry_threshold"]
suppress_threshold = config["backtest"]["suppress_threshold"]  # None = disabled
signal = resolve_signal(row, threshold, suppress_threshold)
# → "up5" | "up3" | None
```

Entry executed only if signal is not None AND cooldown check passes.

**Suppression behavior:**
- If `prob_dn5 >= suppress_threshold` or `prob_dn3 >= suppress_threshold`,
  signal is None regardless of upside probabilities.
- `suppress_threshold: null` disables suppression.
- Suppressed entries are not logged to trade_log.

### Cooldown guard

`can_enter()` moved to `docs/utils/execution_common.md` — now shared with
LiveModeRunner (previously backtest-only by virtue of only being defined
here). See execution_common.md for the full spec and the
session_mode="combined" cross-boundary behavior.

### Cash deduction order (when initial_cash > 0)
Entry candidates at the same bar are processed in the order they appear
in the predictions DataFrame (sort by ticker alphabetically as tiebreak).
Cash is deducted sequentially until exhausted.

**Interaction with `execution.use_all_cash`** (see execution_common.md's
`check_funds_available()`): with the default `use_all_cash: true`,
"exhausted" no longer means the next candidate in line is skipped outright
— the first candidate that cannot be filled at its full requested quantity
is instead sized down to whatever `remaining_cash` allows (possibly a very
small quantity), and *that* trade proceeds. Only a candidate for which even
1 share is unaffordable is skipped. This changes which candidates end up
in `trade_log` for a cash-constrained run compared to the old all-or-nothing
behavior — set `execution.use_all_cash: false` to restore the previous
skip-on-shortfall behavior exactly.

---

## Exit Logic

### Take-profit / Stop-loss exit

```python
tp_pct = config["execution"]["take_profit_up5"] if signal == "up5" \
         else config["execution"]["take_profit_up3"]

direction, exit_price, exit_hour, is_ambiguous = utils.track_price_breach(
    ohlcv_future=bars_from_t,
    ticks_future=ticks_td[ticks_td["hour"] >= entry_hour],
    fill_price=fill_price,
    fill_second=fill_second,
    threshold_up=tp_pct,
    threshold_dn=config["execution"]["stop_loss_pct"],
    exit_interpolation=config["execution"]["exit_interpolation"],
)

if direction is not None:
    # P-10 REVISED (R-2): the former poll-delay alignment
    # (ceil_to_interval to the live position_check_interval_seconds grid)
    # modeled the OLD periodic-poll live loop. Live now detects tp/sl at
    # TICK granularity (WS-primary / REST-backstop — see
    # live_mode_runner.md's Exit Architecture), so there is no poll delay
    # left to model: backtest uses track_price_breach()'s raw
    # exit_hour/exit_price (already interpolate_bundle_price-precision)
    # directly for the fill simulation and for trade_log.exit_bar. The
    # relationship this corrects: live is now ground truth at tick
    # granularity, and backtest approximates it — not the reverse, as the
    # removed poll-delay alignment implied.
    effective_second = exit_hour        # raw, no ceil_to_interval
    effective_price  = exit_price       # raw interpolate_bundle_price output,
                                        # no re-interpolation at a delayed second

    sell_rate = config["execution"]["sell_rate_tp"] if direction == "up" \
                else config["execution"]["sell_rate_sl"]

    weighted_avg_exit_price, total_filled, unfilled_qty, _ = execution_common.simulate_exit_fill(
        ticks_exit=ticks_td[ticks_td["hour"] >= effective_second],
        ohlcv_exit=ohlcv_ticker[ohlcv_ticker["hour"] >= effective_second],
        position_size=quantity,
        breach_bundle_idx=bundle_idx_at(effective_second),
        breach_price=effective_price,
        sell_rate=sell_rate,
        halts_df=halts_td,
    )

    if unfilled_qty > 0:
        # Not a dead position — all available ticks (including after-market) were used.
        # Unfilled portion penalized at flat dead_position_penalty_pct.
        pnl_filled   = (weighted_avg_exit_price - fill_price) / fill_price
        pnl_unfilled = -config["backtest"]["dead_position_penalty_pct"]
        pnl = (pnl_filled * total_filled + pnl_unfilled * unfilled_qty) / quantity
    else:
        pnl = (weighted_avg_exit_price - fill_price) / fill_price

    exit_reason = "take_profit" if direction == "up" else "stop_loss"

else:
    → proceed to session_close / time_limit check
```

`ceil_to_interval(hour, interval_seconds, align_to)`: rounds `hour` up to
the next `align_to`-aligned boundary at `interval_seconds` spacing — e.g.
`align_to="093000"`, `interval_seconds=5`, `hour="093512"` → `"093515"`
(next 5-second mark counting from 09:30:00, not from the input hour
itself). New small utility; belongs in `utils.py` alongside the other hour
utilities (not execution_common.md — it's a pure time-arithmetic function
with no execution-decision content, same category as `hour_add_seconds()`).

**`trade_log.exit_bar`**: should be populated from `effective_second`
above, not the raw `exit_hour` — the poll-delayed value is the realistic
one. This file's exact trade_log-row-assembly code is outside this
section's scope to fresh-quote here; flagging so it isn't missed as a
separate small follow-up when this section is actually applied.

**sell_rate rationale:**
- take_profit exit (rising market): more buyers available → higher sell_rate
- stop_loss exit (falling market): fewer buyers available → lower sell_rate

### Session close exit (priority over time-limit)

```python
if current bar hour == "155900":
    exit immediately at 15:59 bar close
    exit_reason = "session_end"
    if 15:59 bar is halt/no_data:
        fallback: first tick_10 row with hour > "155900"
        if none: → dead position
```

`simulate_exit_fill()` is NOT called for session_end exits —
exit_price is the bar close (or after-market tick fallback).

### Time-limit exit

```python
if 60 valid bars elapsed since entry (via build_effective_bar_sequence):
    exit_price = close of last valid bar
    exit_reason = "time_limit"
```

---

## Ambiguity

`is_ambiguous = True` when tp_target and sl_target are simultaneously satisfied
within the same 10-tick bundle during `track_price_breach()` scan.

Replaces prior definition (1-minute bar level simultaneous breach).
Recorded in `trade_log.is_ambiguous` for post-hoc analysis.

---

## Dead Position

Occurs when:
1. Session_end exit price cannot be determined (15:59 halt + no after-market data).

```
Case A — next trading day has_data=True AND ticker exists in ticker_data_coverage:
    exit_price = next day pre-market first tick
                 fallback: next day ohlcv_1min first bar open
    if exit_price cannot be resolved from either source (NaN/unavailable):
        → Case D (below)
    exit_price *= (1 - dead_position_penalty_pct)
    adjusted_p_entry = (p_entry - dividend_amount) / cum_split_ratio
        where cum_split_ratio = product of split/reverse_split 'value' in
            corporate_events WHERE ticker=? AND event_date IN (D, D+1]
        dividend_amount = 'dividend' value in corporate_events with
            event_date IN (D, D+1] (0.0 if none) — same overnight window and
            rationale as Labeler's Case A (see 05_labeler.md); US splits and
            ex-dividend adjustments always take effect before market open
    exit_reason = "dead_position"
    is_dead_position = True
    pnl = (exit_price - adjusted_p_entry) / adjusted_p_entry

Case B — next trading day has_data=True AND ticker NOT in ticker_data_coverage:
    pnl = -1.0  (full loss — possible delisting)
    exit_price = 0
    exit_reason = "dead_position_delisted"
    is_dead_position = True

Case C — next trading day not in dataset (dataset boundary):
    exit_price = p_entry * (1 - 0.5)
    exit_reason = "dead_position_no_data"
    is_dead_position = True

Case D — Case A path entered, but exit_price unresolvable after fallback
         (pre-market first tick AND next-day first bar open both unavailable
         — extended/multi-day halt):
    pnl = -1.0  (full loss — capital locked with no resolvable exit price)
    exit_price = 0
    exit_reason = "dead_position_extended_halt"
    is_dead_position = True
    (mirrors Labeler's Case D — see 05_labeler.md — same mechanism as Case B)
```

Dead position trades are included in the winning_rate denominator.

Note: `simulate_exit_fill()` exhausting all available ticks with remaining unfilled
quantity is **not** a dead position — the position is partially closed and does not
require an overnight hold. This case is handled in the tp/sl exit path using a blended
PnL with flat penalty for the unfilled portion. Diagnostic via `trade_log.unfilled_quantity / quantity`.

---

## Config Keys (pipeline_config.yaml)

```yaml
backtest:
  initial_cash: 0                    # 0 = unlimited; fixed `balance` for
                                      # execution_common.compute_position_size()
  max_hold_bars:   60
  entry_cooldown_minutes: 5
  entry_threshold:    0.5
  suppress_threshold: 0.5            # null = suppression disabled
  dead_position_penalty_pct: 0.05    # exit-side unfilled-quantity penalty only —
                                      # entry-side unfilled remainder is sized
                                      # down, not penalized (see Entry Slippage Model)
# position_size_cash_pct, position_size_vol_pct, take_profit_up3/up5,
# stop_loss_pct, exit_interpolation, sell_rate_tp/sl, use_all_cash moved to
# the shared `execution:` section (see docs/utils/execution_common.md) —
# backtest and live read the same values rather than carrying independently
# -configurable copies of numbers that must stay identical between them.
```

---

## trade_log Schema Additions

```sql
-- Added columns (in addition to existing schema):
weighted_avg_exit_price  DOUBLE,    -- volume-weighted average fill price across partial fills
partial_fills_count      INTEGER,   -- number of tick bundles used for exit fills
unfilled_quantity        INTEGER,   -- shares remaining after ticks exhausted (0 = fully closed)
is_ambiguous             BOOLEAN,   -- True if simultaneous bundle-level tp/sl breach
```

---

## Constraints

- BacktestEngine accesses DuckDB directly — callers do not pass ohlcv/ticks as arguments
- `db_conn` injected via constructor for testability (mock DB in tests)
- OHLCV loaded once per ticker for all its entry dates
- Tick data loaded once per ticker/date (full day); filtered in memory by range
- `halts_td` passed explicitly to all internal methods — no internal DB queries after initial load
- Session close (15:59 bar) triggers immediate exit — takes priority over time-limit
- `simulate_exit_fill()` not called for session_end exits (bar close used directly)
- After-market data used only as fallback when 15:59 bar is halt/no_data
- Dead position: session_end fallback fails only (15:59 halt + no after-market data)
- Dead position lookup uses `has_data = TRUE` filter (not `is_trading_day`) — consistent with Labeler
- Dead position Case A pnl uses dividend/split-adjusted p_entry (see Dead Position section) —
  `corporate_events` rows for (ticker, event_date IN (D, D+1]) are loaded alongside
  `trading_calendar`/`ticker_data_coverage` for this lookup; same query pattern as Labeler
- Dead position Case D (`exit_reason = "dead_position_extended_halt"`) triggers only when
  Case A's exit_price cannot be resolved after fallback — pnl = -1.0, consistent with Case B
- `simulate_exit_fill()` ticks exhaustion (unfilled_qty > 0) is partial fill — not dead position;
  `is_dead_position` remains False; diagnostic via `trade_log.unfilled_quantity`
- sell_rate_tp > sell_rate_sl: rising-market exits have more available buy-side depth
- Entry slippage search window: entry_hour to entry_hour + 100s (not limited to 1-minute t bar)
- Cooldown applied continuously across full time axis; no reset at session boundaries
- `entry_bar` stored as int via `utils.hour_to_int(entry_hour)`. `exit_bar` stored
  as int via `utils.hour_to_int(effective_second)`, which is now simply
  `exit_hour` itself (P-10 revised, R-2 — the poll-delay alignment was
  removed since live detects tp/sl at tick granularity; see Exit Logic).
  `trade_log.exit_bar` therefore reflects `track_price_breach()`'s raw,
  bundle-interpolated exit moment directly. `exit_reason='entry_canceled'`
  rows (see Entry Slippage Model) set `exit_bar = entry_bar` — there is no
  distinct exit moment for a trade that never opened.
- `trades_by_signal` and `trades_by_exit` JSON-serialized before writing to `experiment_log`
- Dead position trades included in winning_rate denominator
- Suppressed entries (suppress_threshold) not logged to trade_log
- `suppressed_count` tracked in summary for diagnostics
- `run()` returns `tuple[pd.DataFrame, dict]` — trade_log_df and summary_dict;
  DB writes (trade_log INSERT, experiment_log INSERT) performed by Backtester (run_backtest.py),
  not by BacktestEngine
- `build_effective_bar_sequence()`, `track_price_breach()`, `find_fill_bundle()`,
  `interpolate_bundle_price()`, `hour_to_int()`, `hour_to_minutes()`,
  `hour_add_seconds()` — sourced from `utils.py`
- `resolve_signal()`, `can_enter()`, `simulate_exit_fill()`, `simulate_entry_fill()`,
  `compute_position_size()`, `check_funds_available()` —
  sourced from `docs/utils/execution_common.md`'s module
- `is_tradable()` is deliberately NOT called here (N-5), unlike every other
  function in the bullet above. `ticker_cik_map`'s rename_pending /
  quarantine_reason (see db_schema.md) reflect data-pipeline/vendor-latency
  state as of right now, with no date-scoping — calling it here would check
  TODAY's suspension state against whatever historical date a given backtest
  run happens to be simulating, which is both non-deterministic (re-running
  the "same" backtest on a different day can change results) and, more
  fundamentally, the wrong question: both suspend reasons exist to hedge
  live, real-time uncertainty (is this really a rename? a real corporate
  event?) that a backtest already has resolved by the time anyone runs it.
  If a future change reintroduces an `is_tradable()` call here, re-read
  execution_common.md's Constraints (the matching note lives there) before
  assuming its absence was an oversight — it was not.
- `trading_halts` (used above for `halts_td`) is populated by the daily NYSE
  crawl (see metadata_crawler.md) and is training/backtest-only — LiveModeRunner
  never queries this table for real-time decisions; its live-mode halt signal
  is a separate tick-rate heuristic scoped to open positions (see
  live_mode_runner.md's Position Manager Loop)
- Backtest-vs-live exit asymmetry (R-2) — two DISTINCT kinds, do not
  conflate: (1) DATA-GRANULARITY, irreducible — backtest has only 10-tick
  OHLC bundles (`tick_10`) and cannot see intra-bundle spikes that live's
  true WS/REST tick stream detects; shadow mode quantifies this residual.
  (2) The FORMER backstop-vs-WS filter asymmetry (a bundle-based backstop
  would have been less sensitive than WS's per-tick 2-print guard) does
  NOT apply here — it never shipped; the live REST backstop polls
  individual ticks at WS granularity with the same 2-print guard from the
  start (see live_mode_runner.md's Exit Architecture), so only latency
  differs between WS and its own backstop, not accuracy. Only asymmetry
  (1) is a live open residual; it is inherent to the historical dataset's
  granularity, not a design gap.
