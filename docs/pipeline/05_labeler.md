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

ticks_df: pd.DataFrame
    columns: [ticker, date, hour, seq_id, open, high, low, close, volume]
    # tick_10 rows for this ticker-date, full day, sorted by (hour, seq_id)
    # filtered internally per entry point (hour >= t_hour)

halts_df: pd.DataFrame
    columns: [ticker, date, halt_start, halt_end, reason_code]
    # trading_halts table rows for this ticker-date

trading_calendar: pd.DataFrame
    columns: [date, is_trading_day, has_data]
    # used for dead position next-day lookup

ticker_data_coverage: pd.DataFrame
    columns: [ticker, date, has_1min, has_tick]
    # used for dead position Case A vs Case B determination
    # Case A: ticker found in coverage for next trading day (data available)
    # Case B: ticker not found in coverage for next trading day (possible delisting)
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
    # is_ambiguous = True if tp and sl thresholds simultaneously satisfied
    #                within the same 10-tick bundle during Stage 1 scan
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

## Bundle-Level Ambiguity

An ambiguity occurs when tp_target and sl_target are simultaneously satisfied
within the same 10-tick bundle during the Stage 1 scan inside `track_label_breach()`.

```
is_ambiguous = True when, within a single 10-tick bundle:
    bundle.high >= fill_price * (1 + threshold_3pp)   AND
    bundle.low  <= fill_price * (1 - threshold_3pp)

Replaces prior definition (1-minute bar level simultaneous breach).
Bundle-level detection is more precise — tick_10 spans sub-minute windows,
reducing the false-ambiguity rate from bar-level high/low comparisons.

When ambiguity detected:
    → ambiguity_priority determines breach direction ("up" or "dn")
    → is_ambiguous = True on output row
    → Label is still assigned (not skipped)
```

`is_ambiguous` is used downstream by ClassBalancer to optionally exclude
these samples from training, or by Trainer to apply reduced sample weight.

---

## Dead Position Cases

```
Case A: next trading day is_trading_day=True AND has_data=True
        AND ticker found in ticker_data_coverage for that date
        → next-day open available; apply dead_position_penalty_pct

Case B: next trading day is_trading_day=True AND has_data=True
        AND ticker NOT found in ticker_data_coverage for that date
        → possible delisting; label_sw assigned directly (cannot determine direction)

Case C: next trading day not available in dataset (boundary condition)
        → label_sw assigned directly; excluded from training (see ClassBalancer config)
```

---

## Core Logic

```
P_entry      = t bar open price (from entry_points.p_entry)
fill_second  = utils.hour_add_seconds(t_hour, 5)
ticks_from_t = ticks_df filtered to hour >= t_hour, sorted by (hour, seq_id)

--- Step 1: Build effective bar sequence ---

Use build_effective_bar_sequence() from utils.py.
Collect bars from t onward for this (ticker, date).
Collection stops at 15:59 bar — after-market bars are NOT included.
Halt classification applies across all time periods (pre/regular/after-market).

For each missing bar slot up to 15:59:
    if slot overlaps halt interval in halts_df → "halt" (excluded from valid count)
    else                                        → "no_trade" (OHLC = prior close, volume=0)

Target: 60 valid (non-halt) bars, within session boundary (≤ 15:59).
60 valid bars exhausted before reaching 15:59 is not a separate exit case —
the scan always proceeds to 15:59 or until a ±3pp breach occurs.

--- Step 2: Breach detection via track_label_breach() ---

label_direction, is_ambiguous = utils.track_label_breach(
    ohlcv_future       = effective_bars,          # from Step 1
    ticks_future       = ticks_from_t,
    fill_price         = P_entry,                 # P_entry is the reference fill price for Labeler
    fill_second        = fill_second,
    threshold_3pp      = config["labeler"]["threshold_3pp"],
    threshold_5pp      = config["labeler"]["threshold_5pp"],
    exit_interpolation = config["labeler"]["exit_interpolation"],
    ambiguity_priority = config["labeler"]["ambiguity_priority"],
)

if label_direction is not None:
    assign label from {"up5", "up3", "dn3", "dn5"}
    → proceed to output (skip Steps 3 and 4)

else:
    → No ±3pp breach within effective bar sequence
    → Proceed to Step 3 (session close exit)

--- Step 3: Session close exit ---

Triggered when track_label_breach() returns None (no ±3pp breach):

    exit_price = 15:59 bar close

    if 15:59 bar is halt or no_data:
        fallback: first tick_10 row with hour > "155900" (after-market)
        if no after-market tick: → Dead position (Step 4)

    pnl = (exit_price - P_entry) / P_entry
    apply label:
        pnl >= +threshold_5pp                        → label_up5
        threshold_3pp <= pnl < threshold_5pp         → label_up3
        -threshold_3pp < pnl < +threshold_3pp        → label_sw
        -threshold_5pp < pnl <= -threshold_3pp       → label_dn3
        pnl <= -threshold_5pp                        → label_dn5

--- Step 4: Dead position ---

Dead position occurs only when:
    - Session close exit price cannot be determined (15:59 halt + no after-market data)

In this case:
    Lookup next trading day via trading_calendar table.

    Case A — next trading day has_data=True AND ticker exists in ticker_data_coverage:
        exit_price = next day pre-market first tick
                     fallback: next day ohlcv_1min first bar open
        exit_price *= (1 - dead_position_penalty_pct)
        is_dead_position = True
        dead_position_case = "A"
        pnl = (exit_price - P_entry) / P_entry
        apply label by pnl threshold (same rules as Step 3)

    Case B — next trading day has_data=True AND ticker NOT in ticker_data_coverage:
        is_dead_position = True
        dead_position_case = "B"
        assign label_sw directly
        (direction cannot be determined — possible delisting)

    Case C — next trading day not in dataset (boundary condition):
        is_dead_position = True
        dead_position_case = "C"
        assign label_sw directly
        (dataset boundary — actual price movement unknown)
        Note: ClassBalancer can optionally exclude Case C from training.
```

**Key invariant:** Exactly one label equals 1 per entry point. All others are 0.

---

## Interface

```python
class Labeler:
    def __init__(self, config: dict): ...

    def label(
        self,
        entry_points:         pd.DataFrame,
        ohlcv_future:         pd.DataFrame,
        ticks_df:             pd.DataFrame,
        halts_df:             pd.DataFrame,
        trading_calendar:     pd.DataFrame,
        ticker_data_coverage: pd.DataFrame,
    ) -> pd.DataFrame:
        """
        Generate label matrix for all entry points.
        One and only one label column is 1 per row.
        dead_position_case is None for non-dead-position rows.
        is_ambiguous reflects bundle-level simultaneous breach in Stage 1 only.
        """
        ...
```

Note: `build_effective_bar_sequence()` is an internal delegation to
`utils.build_effective_bar_sequence()` and is not part of the public interface.

---

## Constraints

- t bar high/low/close/volume must NOT be used — only t bar open (P_entry) is permitted as reference
- Halt bars are excluded from the 60-bar valid count — search extends to compensate
- Regular session close (15:59 bar) triggers exit when track_label_breach() returns None
- After-market data usage is limited to 15:59 bar halt/no_data fallback only
- build_effective_bar_sequence() collects bars up to 15:59 only — after-market bars never included
- Dead position is the only case where after-market and next-day data are used
- Threshold values (threshold_3pp, threshold_5pp) must be read from config, not hardcoded
- `build_effective_bar_sequence()` is sourced from `utils.py` — do not reimplement
- `is_dead_position` flag must be set in all dead position cases regardless of label assigned
- `dead_position_case` must be set to "A", "B", or "C" for dead position rows; None otherwise
- `is_ambiguous` sourced from `track_label_breach()` Stage 1 return value only;
  Stage 2 ambiguity does not affect the flag
- `ambiguity_priority` controls breach direction on simultaneous bundle breach — read from config
- `ticks_df` must be pre-loaded for the full day and passed explicitly;
  Labeler filters internally per entry point (hour >= t_hour)
- `exit_interpolation` passed through to `track_label_breach()` — read from config
- `ticker_data_coverage` must be pre-loaded and passed explicitly;
  used for dead position Case A vs Case B determination only
- Dead position Case A: pnl computed from next-day exit_price; label assigned by threshold
- Dead position Case B: label_sw assigned directly — no pnl threshold applied
- Dead position Case C: label_sw assigned directly — no pnl threshold applied

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
  ambiguity_priority: "up"            # "up" | "dn" — breach direction on simultaneous bundle breach
  exit_interpolation: true            # passed to track_label_breach(); default true
```

---

## Test Cases Required (test_labeler.py)

| Scenario | Expected |
|---|---|
| +3pp hit at bar 10 (tick level), +5pp hit at bar 20, no -3pp | label_up5 |
| +3pp hit at bar 10, -3pp hit at bar 15 (before +5pp) | label_up3 |
| -3pp hit first, -5pp reached | label_dn5 |
| -3pp hit first, +3pp cuts off before -5pp | label_dn3 |
| Neither ±3pp breached, 15:59 exit | label_sw |
| Ambiguous bundle (both ±3pp in same bundle), priority="up" | label_up3/up5, is_ambiguous=True |
| Dead position Case A (ticker in coverage next day) | is_dead_position=True, case="A", label by pnl threshold |
| Dead position Case B (ticker missing from coverage next day) | is_dead_position=True, case="B", label_sw |
| Dead position Case C (dataset boundary) | is_dead_position=True, case="C", label_sw |
| Halt bar skipped in 60-bar count | label assigned after halt |
