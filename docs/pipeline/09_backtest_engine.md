# Module: BacktestEngine

**File:** `src/backtest/engine.py`
**Depends on:** `docs/data/data_boundary.md`

---

## Role

Simulate trading performance of the trained model on the test split.
Compute winning rate and trade statistics using realistic position management,
slippage approximation, and position sizing rules.

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

### Tick data (per ticker/date — entry slippage)
```sql
SELECT * FROM tick_10
WHERE ticker = ? AND date = ?
ORDER BY hour, seq_id
```
Loaded once per ticker/date combination.
Used for both entry slippage (t bar ticks) and exit slippage (exit bar ticks).
Stored in memory and filtered by hour during processing.

### Trading halts (per ticker/date)
```sql
SELECT * FROM trading_halts
WHERE ticker = ? AND date = ?
```
Used by `build_effective_bar_sequence()` for valid bar counting.

### Trading calendar (for dead position resolution)
```sql
SELECT * FROM trading_calendar
WHERE date > ? ORDER BY date LIMIT 1
```
Used to find next trading day when dead position occurs.

### Ticker data coverage (for dead position Case B)
```sql
SELECT 1 FROM ticker_data_coverage
WHERE ticker = ? AND date = ?
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
fill_price:   slippage-adjusted entry price (see below)
```

---

## Entry Slippage Model

Approximate fill price at t bar open + 5s using tick_10 data.

tick_10 `hour` field represents the **last tick** timestamp of each 10-tick bundle.

```
1. Filter tick_10 for t bar: hour >= entry_hour AND hour < entry_hour + 1min

2. Sort by (hour, seq_id)

3. Compute target_second = utils.hour_add_seconds(entry_hour, 5)

4. Locate surrounding bundles:
   prev_tick: last bundle where hour <= target_second
   next_tick: first bundle where hour > target_second

5. Both exist → linear interpolation:
   t_prev = utils.hour_to_int(prev_tick.hour)
   t_next = utils.hour_to_int(next_tick.hour)
   ratio  = (utils.hour_to_int(target_second) - t_prev) / (t_next - t_prev)
   fill_price = prev_tick.close + ratio * (next_tick.open - prev_tick.close)

6. Only prev_tick exists:
   fill_price = prev_tick.close

7. Only next_tick exists:
   fill_price = next_tick.open

8. No ticks available:
   fill_price = p_entry  (zero slippage fallback)

slippage_pct = (fill_price - p_entry) / p_entry
```

---

## Entry Decision Logic

```python
from utils import resolve_signal

threshold = config["backtest"]["entry_threshold"]
signal = resolve_signal(row, threshold)   # → "up5" | "up3" | None
```

Entry executed only if signal is not None AND cooldown check passes.

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

### Cash deduction order (when initial_cash > 0)
```
Entry candidates at the same bar are processed in the order they appear
in the predictions DataFrame (sort by ticker alphabetically as tiebreak).
Cash is deducted sequentially until exhausted.
```

---

## Exit Logic

### Regular exit conditions (checked per bar during price path replay)

```
take-profit:
    target_pct = 0.03 if signal == "up3" else 0.05
    if max(high of bars t+1..current) - fill_price >= target_pct * fill_price:
        → exit triggered

stop-loss:
    if min(low of bars t..current) <= fill_price * (1 - 0.03):
        → exit triggered

session close (priority over time-limit):
    if current bar hour == "155900":
        → exit immediately at 15:59 bar close
        exit_reason = "session_end"
        if 15:59 bar is halt/no_data:
            fallback: first tick_10 row with hour > "155900"
            if none: last valid bar close

time-limit:
    if 60 valid bars elapsed since entry (via build_effective_bar_sequence):
        exit_price = close of last valid bar
        exit_reason = "time_limit"
```

### Exit price estimation (tick_10 based)

**take-profit and stop-loss exits:**

Default (`exit_interpolation: false`):
```
exit_price = target_price  (fixed)
```

When `exit_interpolation: true`:
```
[take-profit]:
    Locate exit bar's tick_10 bundles where high >= target_price
    prev_tick: last bundle with high < target_price
    next_tick: first bundle with high >= target_price
    if both exist:
        ratio = (target_price - prev_tick.close) / (next_tick.high - prev_tick.close)
        exit_price = prev_tick.close + ratio * (next_tick.high - prev_tick.close)
        exit_price = min(exit_price, next_tick.high)   # conservative upper bound
    else:
        exit_price = target_price  (fallback)

[stop-loss]:
    Same logic, using low <= target_price
    exit_price = max(exit_price, next_tick.low)   # conservative lower bound
```

**session_end exit:**
```
exit_price = 15:59 bar close
tick_10 fallback: first tick_10 row with hour > "155900" (after-market)
final fallback: last valid bar close before 15:59
```

**time-limit exit:**
```
exit_price = close of last valid bar in effective bar sequence
```

### Dead position

Occurs only when session_end exit price cannot be determined
(15:59 halt + no after-market data) and 60 valid bars not exhausted.

```
Lookup next trading day via trading_calendar:

Case A — next day has_data=True AND ticker in ticker_data_coverage:
    exit_price = next day pre-market first tick
                 fallback: next day ohlcv_1min first bar open
    exit_price *= (1 - dead_position_penalty_pct)
    exit_reason = "dead_position"
    dead_position = True

Case B — next day has_data=True AND ticker NOT in ticker_data_coverage:
    exit_price = 0  (full loss — possible delisting)
    pnl_pct = -1.0
    exit_reason = "dead_position_delisted"
    dead_position = True

Case C — next day is future or not in dataset:
    exit_price = fill_price * (1 - 0.5)
    exit_reason = "dead_position_no_data"
    dead_position = True
```

Dead position trades are included in winning rate calculation.
`dead_position` flag enables separate aggregation in reporting.

---

## Output

```python
trade_log: pd.DataFrame
    columns: [
        tr_id,               # UUID generated per trade
        run_id,
        ticker,
        date,
        entry_bar,           # int: HHMMSS-derived (e.g. 93000, 100500) via int(hour_str)
        exit_bar,            # int: HHMMSS-derived
        signal,              # "up3" or "up5"
        fill_price,
        exit_price,
        quantity,
        pnl_pct,             # (exit_price - fill_price) / fill_price
        pnl_abs,             # pnl_pct * fill_price * quantity; NaN in inf mode
        exit_reason,         # "take_profit"|"stop_loss"|"session_end"|"time_limit"
                             # |"dead_position"|"dead_position_delisted"|"dead_position_no_data"
        slippage_pct,
        cash_remaining,      # after this trade; NaN in inf mode
        dead_position,       # BOOLEAN — True if overnight hold occurred
    ]

summary: dict
    {
        "total_trades":         int,
        "winning_trades":       int,        # exit_reason == "take_profit"
        "winning_rate":         float,      # winning_trades / total_trades (dead positions included)
        "avg_pnl_pct":          float,
        "total_pnl_abs":        float,
        "avg_slippage_pct":     float,
        "dead_position_count":  int,
        "dead_position_rate":   float,      # dead_position_count / total_trades
        "trades_by_signal":     {"up3": {...}, "up5": {...}},
        "trades_by_exit":       {"take_profit": n, "stop_loss": n,
                                 "session_end": n, "time_limit": n,
                                 "dead_position": n, ...},
    }
```

Results written to DuckDB:
- `trade_log` table: one row per trade
- `experiment_log` table: summary columns written by `run_backtest.py`

---

## Interface

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
    ) -> tuple[pd.DataFrame, dict]: ...  # (trade_log, summary)

    def estimate_fill_price(
        self,
        entry_hour: str,
        p_entry: float,
        ticks_t_bar: pd.DataFrame,
    ) -> tuple[float, float]:            # (fill_price, slippage_pct)
        ...

    def replay_price_path(
        self,
        bars_future: pd.DataFrame,
        ticks: pd.DataFrame,             # full ticker/date tick_10 for exit interpolation
        fill_price: float,
        signal: str,
    ) -> tuple[float, str, bool]:        # (exit_price, exit_reason, dead_position)
        ...
```

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
  entry_threshold: 0.5
  exit_interpolation: false          # true = tick_10 interpolation for exit price
  dead_position_penalty_pct: 0.05    # applied to next-day exit price (Case A)
```

---

## Constraints

- BacktestEngine accesses DuckDB directly — callers do not pass ohlcv/ticks as arguments
- `db_conn` injected via constructor for testability (mock DB in tests)
- OHLCV loaded once per ticker for all its entry dates
- Tick data loaded once per ticker/date; filtered by hour in memory
- Session close (15:59 bar) triggers immediate exit — takes priority over time-limit
- After-market data used only as fallback when 15:59 bar is halt/no_data
- Dead position: only when session_end fallback also fails
- `entry_bar` and `exit_bar` stored as int via `int(hour_str)` (utils.hour_to_int)
- `trades_by_signal` and `trades_by_exit` JSON-serialized before writing to `experiment_log`
- Dead position trades included in winning_rate denominator
- `resolve_signal()` sourced from `utils.py`
- `build_effective_bar_sequence()` sourced from `utils.py`
- `hour_to_int()`, `hour_to_minutes()`, `hour_add_seconds()` sourced from `utils.py`
