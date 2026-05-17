# Module: Labeler

**File:** `src/labeling/labeler.py`
**Depends on:** `docs/data/data_boundary.md` — read first

---

## Role

Generate 5 independent binary labels for each entry point based on future price movement relative to P_entry.

---

## Input / Output

**Input:**
```python
entry_points: pd.DataFrame
    columns: [ticker, date, hour, p_entry]

ohlcv_future: pd.DataFrame
    columns: [ticker, date, hour, open, high, low, close, volume]
    # bars from t onward, including pre-market and after-market bars
    # sourced from ohlcv_1min table

halts_df: pd.DataFrame
    columns: [ticker, date, halt_start, halt_end, reason_code]
    # trading_halts table rows for this ticker-date

trading_calendar: pd.DataFrame
    columns: [date, is_trading_day, has_data]
    # used for dead position next-day lookup
```

**Output:**
```python
label_matrix: pd.DataFrame
    columns: [ticker, date, hour, label_up5, label_up3, label_sw, label_dn3, label_dn5,
              is_dead_position]
    # one row per entry point
    # exactly one label = 1 per row (mutually exclusive by construction)
    # is_dead_position = True if label assigned via next-day price
```

---

## Label Definitions

| Label | Condition |
|---|---|
| `label_up5` | +3pp breached first AND +5pp reached before -3pp breach |
| `label_up3` | +3pp breached first AND +5pp not reached (or -3pp cuts off) |
| `label_sw` | Neither ±3pp breached within effective search range |
| `label_dn3` | -3pp breached first AND -5pp not reached (or +3pp cuts off) |
| `label_dn5` | -3pp breached first AND -5pp reached before +3pp breach |

---

## Core Logic

```
P_entry = t bar open price (from entry_points.p_entry)

--- Step 1: Build effective bar sequence ---

Use build_effective_bar_sequence() from utils.py.
Collect bars from t onward for this (ticker, date).
Halt classification applies across all time periods (pre/regular/after-market).

For each missing bar slot:
    if slot overlaps halt interval in halts_df → "halt" (excluded from valid count)
    else                                        → "no_trade" (OHLC = prior close, volume=0)

Target: 60 valid (non-halt) bars.

--- Step 2: Sequential bar scan ---

For each valid bar i in effective sequence (sequential):

    max_up(i)   = ( max(high of valid bars t..i) - P_entry ) / P_entry
    max_down(i) = ( P_entry - min(low of valid bars t..i)  ) / P_entry

    if max_up(i) >= 0.03 (first breach):
        label_up3 = 1  (tentative)
        continue scanning for +5pp:
            if max_up(j) >= 0.05 before max_down(j) >= 0.03:
                label_up5 = 1  (override)
        break

    if max_down(i) >= 0.03 (first breach):
        label_dn3 = 1  (tentative)
        continue scanning for -5pp:
            if max_down(j) >= 0.05 before max_up(j) >= 0.03:
                label_dn5 = 1  (override)
        break

    if bar hour == session_close_hour (155900):
        → Regular session close reached. Exit immediately.
        exit_price = bar close
        pnl = (exit_price - P_entry) / P_entry
        assign label by pnl threshold (see Step 3 below)
        break

--- Step 3: Session close exit ---

Triggered when 15:59 bar is reached during scan (before ±3pp breach):

    exit_price = 15:59 bar close

    if 15:59 bar is halt or no_data:
        fallback: first tick_10 row with hour > "155900" (after-market)
        if no after-market tick: exit_price = last valid bar close before 15:59

    pnl = (exit_price - P_entry) / P_entry
    apply label:
        pnl >= +0.05  → label_up5
        pnl >= +0.03  → label_up3
        -0.03 < pnl < +0.03 → label_sw
        pnl <= -0.05  → label_dn5
        pnl <= -0.03  → label_dn3

--- Step 4: 60 valid bars exhausted before session close ---

If 60 valid bars are exhausted before reaching 15:59 bar:
    exit_price = close of last valid bar
    pnl = (exit_price - P_entry) / P_entry
    assign label by pnl threshold (same as Step 3)

    Note: This occurs only when a large number of halt bars
    pushes the 60-bar window past the regular session boundary
    into after-market hours, and all remaining bars are also
    halt/no_trade. In practice this is a rare edge case.

--- Step 5: Dead position ---

Dead position occurs only when:
    - Session close exit price cannot be determined (15:59 halt + no after-market data)
    - AND 60 valid bars not yet exhausted

In this case:
    Lookup next trading day via trading_calendar table.

    Case A — next trading day has_data=True AND ticker exists in ticker_data_coverage:
        exit_price = next day pre-market first tick
                     fallback: next day ohlcv_1min first bar open
        exit_price *= (1 - dead_position_penalty_pct)
        is_dead_position = True

    Case B — next trading day has_data=True AND ticker not in ticker_data_coverage:
        pnl = -1.0  (full loss — possible delisting)
        exit_price = 0
        is_dead_position = True

    Case C — next trading day is future or not in dataset:
        exit_price = P_entry * (1 - 0.5)
        is_dead_position = True

    assign label by pnl threshold (same as Step 3)
```

**Key invariant:** Exactly one label equals 1 per entry point. All others are 0.

---

## Interface

```python
class Labeler:
    def label(
        self,
        entry_points: pd.DataFrame,
        ohlcv_future: pd.DataFrame,
        halts_df: pd.DataFrame,
        trading_calendar: pd.DataFrame,
    ) -> pd.DataFrame:
        """
        Returns label_matrix with 5 binary label columns + is_dead_position.
        One and only one label column is 1 per row.
        """
        ...

    def build_effective_bar_sequence(
        self,
        ohlcv_future: pd.DataFrame,
        halts_df: pd.DataFrame,
        date: str,
        t_hour: str,
        target_valid_bars: int = 60,
    ) -> pd.DataFrame:
        """
        Delegates to utils.build_effective_bar_sequence().
        Halt classification applies across all time periods.
        """
        ...
```

---

## Constraints

- t bar high/low/close/volume must NOT be used — only t bar open (P_entry) is permitted as reference
- Halt bars are excluded from the 60-bar valid count — search extends to compensate
- Regular session close (15:59 bar) triggers immediate exit during scan — after-market is fallback only for 15:59 halt/no_data
- After-market data usage is limited to 15:59 bar halt/no_data fallback and dead position resolution
- Threshold values (0.03, 0.05) must be read from config, not hardcoded
- `build_effective_bar_sequence()` is sourced from `utils.py` — do not reimplement
- `is_dead_position` flag must be set in all dead position cases regardless of label assigned

---

## Config Keys (pipeline_config.yaml)

```yaml
labeler:
  threshold_3pp: 0.03
  threshold_5pp: 0.05
  session_close_hour: "155900"
  after_market_close_hour: "200000"
  max_holding_bars: 60                # valid (non-halt) bars
  dead_position_penalty_pct: 0.05     # applied to next-day exit price (Case A)
```

---

## Test Cases Required (test_labeler.py)

| Scenario | Expected |
|---|---|
| +3pp hit at bar 10, +5pp hit at bar 20, no -3pp | label_up5 |
| +3pp hit at bar 10, -3pp hit at bar 15 (before +5pp) | label_up3 |
| -3pp hit first, -5pp reached | label_dn5 |
| -3pp hit first, +3pp cuts off before -5pp | label_dn3 |
| Scan reaches 15:59 bar, pnl = +2pp | label_sw |
| Scan reaches 15:59 bar, pnl = +4pp | label_up3 |
| 15:59 bar is halt, after-market tick available | label assigned from after-market tick |
| 15:59 bar is halt, no after-market data | dead position Case A or C |
| 5-bar halt mid-session, breach occurs at valid bar 61 | correct label (extended search) |
| +3pp and -3pp hit simultaneously (same bar) | up3 priority (configurable) |
| Dead position, next day data available for ticker | is_dead_position=True, Case A |
| Dead position, next day exists but ticker missing | is_dead_position=True, Case B, pnl=-1.0 |
| Dead position, next day not in dataset | is_dead_position=True, Case C |
