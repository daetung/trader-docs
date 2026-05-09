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
    # bars from t onward, including after-market bars if available
    # sourced from ohlcv_1min table

halts_df: pd.DataFrame
    columns: [ticker, date, halt_start, halt_end, reason_code]
    # trading_halts table rows for this ticker-date

after_market_ticks: pd.DataFrame   (optional)
    columns: [ticker, date, hour, price, volume]
    # tick_10 rows after '155900' for fallback exit price
```

**Output:**
```python
label_matrix: pd.DataFrame
    columns: [ticker, date, hour, label_up5, label_up3, label_sw, label_dn3, label_dn5]
    # one row per entry point
    # exactly one label = 1 per row (mutually exclusive by construction)
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

Collect all bars from t onward for this (ticker, date).
For each bar slot in regular session (t ~ 15:59):
    if bar exists in ohlcv_future → include
    if bar is missing:
        classify via halts_df:
            "halt"     → exclude from search count (do not consume valid_bar quota)
            "no_trade" → include as zero-volume bar (OHLC = prior close)

Target: 60 valid (non-halt) bars.
Search continues past regular session close if halt duration consumed bars,
up to session end of day only.

--- Step 2: After-market fallback ---

If 60 valid bars exhausted without ±3pp breach AND regular session closes:
    Position must be closed (no overnight holds).
    Exit price = first available after-market tick price (hour > '155900')
    If no after-market tick available: exit price = 15:59 bar close

    Compute final pnl = (exit_price - P_entry) / P_entry
    Apply label:
        if pnl >= +0.05 → label_up5
        if pnl >= +0.03 → label_up3
        if -0.03 < pnl < +0.03 → label_sw
        if pnl <= -0.05 → label_dn5
        if pnl <= -0.03 → label_dn3

--- Step 3: Sequential bar scan (regular search) ---

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

if no breach after all valid bars + after-market fallback:
    label_sw = 1
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
        after_market_ticks: pd.DataFrame | None = None,
    ) -> pd.DataFrame:
        """
        Returns label_matrix with 5 binary label columns.
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
        Build sequence of valid (non-halt) bars.
        Fills no_trade gaps with forward-fill.
        Excludes halt bars from valid count.
        """
        ...
```

---

## Constraints

- t bar high/low/close/volume must NOT be used — only t bar open (P_entry) is permitted as reference
- Halt bars are excluded from the 60-bar valid count — search extends to compensate
- After-market exit is applied only after regular session close with no breach
- After-market price: first tick_10 row with hour > '155900'; fallback to 15:59 bar close
- Threshold values (0.03, 0.05) must be read from config, not hardcoded
- No overnight holds — all positions resolved within the trading day + after-market

---

## Config Keys (pipeline_config.yaml)

```yaml
labeler:
  threshold_3pp: 0.03
  threshold_5pp: 0.05
  session_close_hour: "155900"
  after_market_close_hour: "200000"   # latest after-market tick to consider
  max_holding_bars: 60                # valid (non-halt) bars
```

---

## Test Cases Required (test_labeler.py)

| Scenario | Expected label |
|---|---|
| +3pp hit at bar 10, +5pp hit at bar 20, no -3pp | label_up5 |
| +3pp hit at bar 10, -3pp hit at bar 15 (before +5pp) | label_up3 |
| -3pp hit first, -5pp reached | label_dn5 |
| -3pp hit first, +3pp cuts off before -5pp | label_dn3 |
| No ±3pp breach in 60 valid bars, after-market price +2pp | label_sw |
| No ±3pp breach, after-market price +4pp | label_up3 |
| 5-bar halt mid-session, breach occurs at valid bar 61 | correct label (extended search) |
| Halt covers session close, no after-market data | label_sw (15:59 fallback) |
| +3pp and -3pp hit simultaneously (same bar) | up3 priority (configurable) |
