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

```
initial_cash: float    # from config; 0 = unlimited (inf mode)

Per-trade buy quantity:
    if initial_cash == 0 (inf mode):
        quantity = floor(t_bar_volume * 0.10)

    else:
        cash_based  = floor((initial_cash * 0.05) / fill_price)
        vol_based   = floor(t_bar_volume * 0.10)
        quantity    = min(cash_based, vol_based)

t_bar_volume: ohlcv_1min volume of the t bar
fill_price:   slippage-adjusted entry price (see Entry Slippage Model)
```

---

## Entry Slippage Model

Approximate fill price at t bar open + 5s using tick_10 data.

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

    if fill_bundle is not None:
        fill_price = utils.interpolate_bundle_price(prev_bundle, fill_bundle, fill_second)
    else:
        fill_price = p_entry    # no bundle within 100s — zero slippage fallback

slippage_pct = (fill_price - p_entry) / p_entry
```

---

## Entry Decision Logic

```python
from utils import resolve_signal

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
```python
def can_enter(
    ticker: str,
    current_hour: str,
    last_entry_hour: str | None,
    cooldown_minutes: int,
) -> bool:
    if last_entry_hour is None:
        return True
    current_min = utils.hour_to_minutes(current_hour)
    last_min    = utils.hour_to_minutes(last_entry_hour)
    return (current_min - last_min) >= cooldown_minutes
```

**Cooldown across session boundaries (session_mode="combined"):**
Cooldown is applied continuously across the full time axis regardless of
session boundaries. No cooldown reset at the pre-market/regular session boundary.

### Cash deduction order (when initial_cash > 0)
Entry candidates at the same bar are processed in the order they appear
in the predictions DataFrame (sort by ticker alphabetically as tiebreak).
Cash is deducted sequentially until exhausted.

---

## Exit Logic

### Take-profit / Stop-loss exit

```python
tp_pct = config["backtest"]["take_profit_up5"] if signal == "up5" \
         else config["backtest"]["take_profit_up3"]

direction, exit_price, exit_hour, is_ambiguous = utils.track_price_breach(
    ohlcv_future=bars_from_t,
    ticks_future=ticks_td[ticks_td["hour"] >= entry_hour],
    fill_price=fill_price,
    fill_second=fill_second,
    threshold_up=tp_pct,
    threshold_dn=config["backtest"]["stop_loss_pct"],
    exit_interpolation=config["backtest"]["exit_interpolation"],
)

if direction is not None:
    sell_rate = config["backtest"]["sell_rate_tp"] if direction == "up" \
                else config["backtest"]["sell_rate_sl"]

    weighted_avg_exit_price, total_filled, unfilled_qty, _ = utils.simulate_exit_fill(
        ticks_exit=ticks_td[ticks_td["hour"] >= exit_hour],
        ohlcv_exit=ohlcv_ticker[ohlcv_ticker["hour"] >= exit_hour],
        position_size=quantity,
        breach_bundle_idx=breach_bundle_idx,
        breach_price=exit_price,
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
    exit_price *= (1 - dead_position_penalty_pct)
    exit_reason = "dead_position"
    is_dead_position = True

Case B — next trading day has_data=True AND ticker NOT in ticker_data_coverage:
    pnl = -1.0  (full loss — possible delisting)
    exit_price = 0
    exit_reason = "dead_position_delisted"
    is_dead_position = True

Case C — next trading day not in dataset (dataset boundary):
    exit_price = p_entry * (1 - 0.5)
    exit_reason = "dead_position_no_data"
    is_dead_position = True
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
  initial_cash: 0                    # 0 = unlimited
  position_size_cash_pct: 0.05       # 5% of cash per trade
  position_size_vol_pct:  0.10       # 10% of t bar volume
  take_profit_up3: 0.03
  take_profit_up5: 0.05
  stop_loss_pct:   0.03
  max_hold_bars:   60
  entry_cooldown_minutes: 5
  entry_threshold:    0.5
  suppress_threshold: 0.5            # null = suppression disabled
  exit_interpolation: true           # default true; false = 1-minute bar only (asymmetric)
  sell_rate_tp: 0.30                 # fraction of per-tick volume for take-profit exits
  sell_rate_sl: 0.15                 # fraction of per-tick volume for stop-loss exits
  dead_position_penalty_pct: 0.05
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
- `simulate_exit_fill()` ticks exhaustion (unfilled_qty > 0) is partial fill — not dead position;
  `is_dead_position` remains False; diagnostic via `trade_log.unfilled_quantity`
- sell_rate_tp > sell_rate_sl: rising-market exits have more available buy-side depth
- Entry slippage search window: entry_hour to entry_hour + 100s (not limited to 1-minute t bar)
- Cooldown applied continuously across full time axis; no reset at session boundaries
- `entry_bar` and `exit_bar` stored as int via `utils.hour_to_int()`
- `trades_by_signal` and `trades_by_exit` JSON-serialized before writing to `experiment_log`
- Dead position trades included in winning_rate denominator
- Suppressed entries (suppress_threshold) not logged to trade_log
- `suppressed_count` tracked in summary for diagnostics
- `run()` returns `tuple[pd.DataFrame, dict]` — trade_log_df and summary_dict;
  DB writes (trade_log INSERT, experiment_log INSERT) performed by Backtester (run_backtest.py),
  not by BacktestEngine
- `resolve_signal()`, `build_effective_bar_sequence()`, `track_price_breach()`,
  `simulate_exit_fill()`, `find_fill_bundle()`, `interpolate_bundle_price()`,
  `hour_to_int()`, `hour_to_minutes()`, `hour_add_seconds()` — all sourced from `utils.py`
