# Module: BacktestEngine

**File:** `src/backtest/engine.py`
**Depends on:** `docs/data/data_boundary.md`

---

## Role

Simulate trading performance of the trained model on the test split.
Compute winning rate and trade statistics using realistic position management,
slippage approximation, and position sizing rules.

---

## Input

```python
predictions: pd.DataFrame
    # columns: [ticker, date, hour, p_entry,
    #           prob_up5, prob_up3, prob_sw, prob_dn3, prob_dn5,
    #           label_up5, label_up3, label_sw, label_dn3, label_dn5]
    # one row per entry point candidate in test split

ohlcv: dict[str, pd.DataFrame]
    # {ticker: full ohlcv_1min DataFrame} for all tickers in test split
    # needed for price path replay during hold period

ticks: dict[str, pd.DataFrame]
    # {ticker: tick_10 DataFrame} for t bar slippage estimation

config: dict
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

t_bar_volume:   1-minute bar volume of the t bar (ohlcv_1min)
                (10-tick full volume not available for all bars → use 1min bar)
fill_price:     slippage-adjusted entry price (see below)
```

---

## Slippage Model (Entry)

Approximate fill price at t bar open + 5s using 10-tick data.

```
1. Collect tick_10 rows where:
       ticker = entry.ticker
       date   = entry.date
       hour   = entry.hour  (t bar — within-bar ticks)

2. Sort by (hour, seq_id)

3. Estimate t+5s boundary:
   - 1-minute bar contains ~6 ticks on average
   - "5 seconds into the bar" ≈ first min(n_ticks_available, 1) tick(s)
   - Use the close price of the first available tick as fill_price
   - If no ticks available for t bar: fill_price = p_entry (t bar open, zero slippage)

4. slippage_pct = (fill_price - p_entry) / p_entry
```

---

## Entry Decision Logic

BacktestEngine receives raw probability output from `model.predict()` and applies
threshold comparison to determine the signal. The model is responsible for probabilities only.

```python
threshold = config["backtest"]["entry_threshold"]

def resolve_signal(row, threshold) -> str | None:
    """Returns "up5", "up3", or None. up5 takes priority over up3."""
    if row["prob_up5"] >= threshold: return "up5"
    if row["prob_up3"] >= threshold: return "up3"
    return None
```

---

## Position Management

### Entry condition
```
Entry is executed only if:
    1. resolve_signal(probs, threshold) returns "up3" or "up5"
    2. Cooldown check passes (see below)
```

### Cooldown guard
```
Cooldown is measured in minutes to correctly handle empty bars (bars with no trades).
Bar count is not used because gaps in ohlcv_1min make bar-index distance unreliable.

config key: entry_cooldown_minutes (replaces entry_cooldown_bars)

def can_enter(ticker, current_hour: str, last_entry_hour: str | None, cooldown_minutes: int) -> bool:
    if last_entry_hour is None:
        return True
    current_min  = int(current_hour[:2]) * 60 + int(current_hour[2:4])
    last_min     = int(last_entry_hour[:2]) * 60 + int(last_entry_hour[2:4])
    return (current_min - last_min) >= cooldown_minutes
```

### Cash deduction order (when initial_cash > 0)
```
Entry candidates at the same bar are processed in the order they appear
in the predictions DataFrame (sort by ticker alphabetically as tiebreak).
Cash is deducted sequentially until exhausted.
```

### Exit conditions
```
take-profit:
    target_pct = 0.03 if signal == "up3" else 0.05
    Exit when: max(high of bars t+1..t+59) - fill_price >= target_pct * fill_price
    Exit price ≈ fill_price * (1 + target_pct)  [conservative estimate]

stop-loss (either condition triggers first):
    1. Price drops: min(low of bars t..t+59) <= fill_price * (1 - 0.03)
       Exit price ≈ fill_price * (1 - 0.03)
    2. Time limit: 60 bars elapsed since entry (t+60 bar open)
       Exit price = close of t+59 bar

Price path replay uses ohlcv_1min high/low per bar (not tick data for exit).
10-tick data is only used for entry slippage estimation.
```

---

## Output

```python
trade_log: pd.DataFrame
    # Matches trade_log table schema in db_schema.md.
    # entry_bar and exit_bar are stored as HHMMSS-derived integers
    # (e.g. hour "093000" → 93000) before writing to DuckDB.
    # Conversion: int(hour_str) — leading zeros dropped naturally by int().
    columns: [
        tr_id,               # UUID generated per trade
        run_id,
        ticker,
        date,
        entry_bar,           # int: e.g. 93000, 100500
        exit_bar,            # int: e.g. 94500
        signal,              # "up3" or "up5"
        fill_price,
        exit_price,
        quantity,
        pnl_pct,             # (exit_price - fill_price) / fill_price
        pnl_abs,             # pnl_pct * fill_price * quantity; NaN in inf mode
        exit_reason,         # "take_profit" | "stop_loss" | "time_limit"
        slippage_pct,
        cash_remaining,      # after this trade; NaN in inf mode
    ]

summary: dict
    {
        "total_trades":      int,
        "winning_trades":    int,        # trades where exit_reason == "take_profit"
        "winning_rate":      float,      # winning_trades / total_trades
        "avg_pnl_pct":       float,
        "total_pnl_abs":     float,      # only meaningful if not inf mode
        "avg_slippage_pct":  float,
        "trades_by_signal":  {"up3": {...}, "up5": {...}},   # JSON-serialized on DB write
        "trades_by_exit":    {"take_profit": n, "stop_loss": n, "time_limit": n},  # JSON-serialized on DB write
    }
```

Results are written to DuckDB:
- `trade_log` table: one row per trade
- `experiment_log` table: summary columns appended to the matching `run_id` row

---

## Interface

```python
class BacktestEngine:
    def __init__(self, config: dict, db_conn: duckdb.DuckDBPyConnection): ...

    def run(
        self,
        predictions: pd.DataFrame,
        ohlcv: dict[str, pd.DataFrame],
        ticks: dict[str, pd.DataFrame],
    ) -> tuple[pd.DataFrame, dict]: ...  # (trade_log, summary)

    def estimate_fill_price(
        self,
        ticker: str,
        entry: dict,
        ticks_t_bar: pd.DataFrame,
    ) -> tuple[float, float]:            # (fill_price, slippage_pct)
        ...

    def replay_price_path(
        self,
        bars_future: pd.DataFrame,       # t to t+59 bars
        fill_price: float,
        signal: str,
    ) -> tuple[float, str]:              # (exit_price, exit_reason)
        ...
```

---

## Config Keys (pipeline_config.yaml)

```yaml
backtest:
  initial_cash: 0                  # 0 = unlimited
  position_size_cash_pct: 0.05     # 5% of cash per trade
  position_size_vol_pct:  0.10     # 10% of t bar volume
  take_profit_up3: 0.03
  take_profit_up5: 0.05
  stop_loss_pct:   0.03
  max_hold_bars:   60
  entry_cooldown_minutes: 5        # minimum elapsed minutes between entries per ticker
  entry_threshold: 0.5             # model probability threshold for entry signal
```

---

## Constraints

- Exit price path uses ohlcv_1min H/L only — no tick data for exits
- 10-tick data used only for entry slippage estimation
- Cash deduction is sequential in prediction DataFrame order
- `entry_bar` and `exit_bar` written to `trade_log` as HHMMSS-derived integers; conversion is `int(hour_str)`
- `trades_by_signal` and `trades_by_exit` JSON-serialized before writing to `experiment_log`
- trade_log written to DuckDB `trade_log` table; summary written to `experiment_log` table with matching `run_id`
- inf mode (initial_cash=0): `pnl_abs`, `cash_remaining` set to NaN; winning_rate is the primary metric
- Cooldown measured in minutes (`entry_cooldown_minutes`), not bar count, to handle empty bars correctly
- Entry signal resolved by BacktestEngine via threshold comparison on `predict()` output — not by model
