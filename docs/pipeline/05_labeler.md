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
    columns: [ticker, date, hour,
              label_up5, label_up3, label_sw, label_dn3, label_dn5,
              is_dead_position, dead_position_case, is_ambiguous]
    # one row per entry point
    # exactly one label = 1 per row (mutually exclusive by construction)
    # is_dead_position = True if label assigned via next-day price
    # dead_position_case = "A" | "B" | "C" | None
    # is_ambiguous = True if same-bar breach of both +3pp and -3pp thresholds
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

## Same-Bar Ambiguity

A same-bar ambiguity occurs when a single 1-minute bar's **individual** high and low
simultaneously breach both the upside and downside thresholds relative to P_entry.
This is checked against the current bar's own OHLC values — not against the
cumulative max/min across previously scanned bars.

```
Ambiguity detection condition (checked per bar i during scan,
using bar i's individual high and low only):

    (bar_i.high - P_entry) / P_entry >= threshold_3pp
    AND
    (P_entry - bar_i.low)  / P_entry >= threshold_3pp

This condition captures bars where both thresholds were touched within
the same 1-minute window. Tick-level ordering is not recoverable from
1-minute OHLCV data, so breach direction cannot be determined with certainty.

Distinct from cumulative breach check:
    max_up(i)   = (max(high of valid bars t..i) - P_entry) / P_entry  ← cumulative
    max_down(i) = (P_entry - min(low of valid bars t..i)) / P_entry   ← cumulative

    Ambiguity uses bar_i.high and bar_i.low directly — not max_up(i)/max_down(i).
    A bar can be ambiguous even if prior bars have not yet reached threshold,
    as long as the current bar itself spans both thresholds.

When ambiguity detected:
    → Apply ambiguity_priority rule to determine breach direction
    → Set is_ambiguous = True on the output row
    → Label is still assigned (not skipped)
```

`is_ambiguous` is used downstream by ClassBalancer to optionally exclude
these samples from training, or by Trainer to apply reduced sample weight.
It does not affect label assignment logic itself.

**Priority rule on simultaneous breach:**
```yaml
labeler:
  ambiguity_priority: "up"   # "up" = treat as upside breach first (default)
                              # "down" = treat as downside breach first
```

---

## Core Logic

```
P_entry = t bar open price (from entry_points.p_entry)

--- Step 1: Build effective bar sequence ---

Use build_effective_bar_sequence() from utils.py.
Collect bars from t onward for this (ticker, date).
Collection stops at 15:59 bar — after-market bars are NOT included.
Halt classification applies across all time periods (pre/regular/after-market).

For each missing bar slot up to 15:59:
    if slot overlaps halt interval in halts_df → "halt" (excluded from valid count)
    else                                        → "no_trade" (OHLC = prior close, volume=0)

Target: 60 valid (non-halt) bars, within regular session boundary (≤ 15:59).

--- Step 2: Sequential bar scan ---

For each valid bar i in effective sequence (sequential):

    [Ambiguity check — uses bar i's individual high/low, not cumulative]
    if (bar_i.high - P_entry) / P_entry >= threshold_3pp
    AND (P_entry - bar_i.low) / P_entry >= threshold_3pp:
        → is_ambiguous = True
        → apply ambiguity_priority to select breach direction
        → treat selected direction as the first breach and proceed

    [Cumulative breach tracking]
    max_up(i)   = ( max(high of valid bars t..i) - P_entry ) / P_entry
    max_down(i) = ( P_entry - min(low of valid bars t..i)  ) / P_entry

    if max_up(i) >= threshold_3pp (first breach, upside):
        label_up3 = 1  (tentative)
        continue scanning for +5pp:
            if max_up(j) >= threshold_5pp before max_down(j) >= threshold_3pp:
                label_up5 = 1  (override)
        break

    if max_down(i) >= threshold_3pp (first breach, downside):
        label_dn3 = 1  (tentative)
        continue scanning for -5pp:
            if max_down(j) >= threshold_5pp before max_up(j) >= threshold_3pp:
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
        if no after-market tick: → Dead position (Step 4)

    pnl = (exit_price - P_entry) / P_entry
    apply label:
        pnl >= +threshold_5pp  → label_up5
        pnl >= +threshold_3pp  → label_up3
        -threshold_3pp < pnl < +threshold_3pp → label_sw
        pnl <= -threshold_5pp  → label_dn5
        pnl <= -threshold_3pp  → label_dn3

--- Step 4: Dead position ---

Dead position occurs only when:
    - Session close exit price cannot be determined (15:59 halt + no after-market data)

Note: build_effective_bar_sequence() collects bars up to 15:59 only.
60 valid bars exhausted before reaching 15:59 is not a separate exit case —
the scan always proceeds to 15:59 or until a ±3pp breach occurs.

In this case:
    Lookup next trading day via trading_calendar table.

    Case A — next trading day has_data=True AND ticker exists in ticker_data_coverage:
        exit_price = next day pre-market first tick
                     fallback: next day ohlcv_1min first bar open
        exit_price *= (1 - dead_position_penalty_pct)
        is_dead_position = True
        dead_position_case = "A"

    Case B — next trading day has_data=True AND ticker not in ticker_data_coverage:
        pnl = -1.0  (full loss — possible delisting)
        exit_price = 0
        is_dead_position = True
        dead_position_case = "B"

    Case C — next trading day is future or not in dataset:
        exit_price = P_entry * (1 - 0.5)
        is_dead_position = True
        dead_position_case = "C"
        Note: Case C typically indicates a dataset boundary condition
              (entry near the end of available data), not a genuine overnight hold.
              ClassBalancer can optionally exclude Case C samples from training.

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
        Returns label_matrix with 5 binary label columns,
        is_dead_position, dead_position_case, and is_ambiguous.
        One and only one label column is 1 per row.
        dead_position_case is None for non-dead-position rows.
        is_ambiguous is False unless same-bar individual breach detected.
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
        Collection stops at 15:59 bar — after-market bars excluded.
        """
        ...
```

---

## Constraints

- t bar high/low/close/volume must NOT be used — only t bar open (P_entry) is permitted as reference
- Halt bars are excluded from the 60-bar valid count — search extends to compensate
- Regular session close (15:59 bar) triggers immediate exit during scan
- after-market data usage is limited to 15:59 bar halt/no_data fallback only
- build_effective_bar_sequence() collects bars up to 15:59 only — after-market bars never included
- Dead position is the only case where after-market and next-day data are used
- Threshold values (threshold_3pp, threshold_5pp) must be read from config, not hardcoded
- `build_effective_bar_sequence()` is sourced from `utils.py` — do not reimplement
- `is_dead_position` flag must be set in all dead position cases regardless of label assigned
- `dead_position_case` must be set to "A", "B", or "C" for dead position rows; None otherwise
- `is_ambiguous` detection uses the current bar's individual high/low only —
  NOT cumulative max_up/max_down across previously scanned bars
- `is_ambiguous` is set before cumulative breach tracking proceeds for that bar
- `ambiguity_priority` controls breach direction on simultaneous breach — read from config

---

## Config Keys (pipeline_config.yaml)

```yaml
labeler:
  threshold_3pp: 0.03
  threshold_5pp: 0.05
  session_close_hour: "155900"
  after_market_close_hour: "200000"
  max_holding_bars: 60                # valid (non-halt) bars, within session boundary
  dead_position_penalty_pct: 0.05     # applied to next-day exit price (Case A)
  ambiguity_priority: "up"            # "up" | "down" — breach direction on same-bar ambiguity
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
| 5-bar halt mid-session, breach occurs at valid bar 61 | correct label (extended search within session) |
| bar_i.high >= +3pp AND bar_i.low <= -3pp (individual bar spans both), priority=up | label_up3, is_ambiguous=True |
| bar_i.high >= +3pp AND bar_i.low <= -3pp (individual bar spans both), priority=down | label_dn3, is_ambiguous=True |
| cumulative max_up >= +3pp but bar_i alone does not span -3pp | is_ambiguous=False |
| Normal breach (no same-bar span) | is_ambiguous=False |
| Dead position, next day data available for ticker | is_dead_position=True, dead_position_case="A" |
| Dead position, next day exists but ticker missing | is_dead_position=True, dead_position_case="B", pnl=-1.0 |
| Dead position, next day not in dataset | is_dead_position=True, dead_position_case="C" |
| Non-dead-position row | dead_position_case=None |
