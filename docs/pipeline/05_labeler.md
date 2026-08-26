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
    # Case A: ticker found in coverage for next day with has_data=True (data available)
    # Case B: ticker not found in coverage for next day with has_data=True (possible delisting)

corporate_events: pd.DataFrame
    columns: [ticker, event_date, event_type, value]
    # used only for dead position Case A/D overnight dividend/split adjustment
    # (see Step 4) — filtered internally per entry point to event_date IN (D, D+1]
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
    # dead_position_case = "A" | "B" | "C" | "D" | None
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
Case A: next day has_data=True AND ticker found in ticker_data_coverage for that date
        → next-day open available; apply dead_position_penalty_pct
        → if exit_price still cannot be resolved (extended halt into next day
          and beyond) → falls through to Case D instead

Case B: next day has_data=True AND ticker NOT found in ticker_data_coverage for that date
        → possible delisting; pnl = -1.0 (full loss), label assigned by pnl
          threshold (resolves to label_dn5) — see rationale below

Case C: no next day with has_data=True in dataset (boundary condition)
        → label_sw assigned directly; excluded from training (see ClassBalancer config)
        → dataset-boundary artifact, not a market outcome — label_sw retained
          (unlike Case B/D, there is no real trade outcome to score here)

Case D: Case A path entered, but exit_price unresolvable after fallback
        (pre-market first tick AND next-day first bar open both unavailable —
         extended/multi-day halt) → pnl = -1.0 (full loss), label assigned by
        pnl threshold (resolves to label_dn5), same mechanism as Case B

Case B/D rationale: pnl threshold definitions are literal (label_dn5 = pnl <=
-threshold_5pp); a full loss of capital is definitionally label_dn5 regardless
of whether the underlying cause is delisting (B) or extended halt (D) —
assigning label_sw here would train the model to treat capital-destroying
outcomes as neutral, and would also diverge from BacktestEngine (which already
scores both as pnl=-1.0). dead_position_case still distinguishes B and D for
downstream filtering (e.g. ClassBalancer exclusion options) even though both
now resolve to the same label.
```

---

## Core Logic

```
P_entry      = t bar open price (from entry_points.p_entry)
fill_second  = utils.hour_add_seconds(t_hour, 5)
ticks_from_t = ticks_df filtered to hour >= t_hour, sorted by (hour, seq_id)

--- Step 1: Build effective bar sequence ---

Use build_effective_bar_sequence() from utils.py, passing
execution.max_hold_bars as its target_valid_bars and the {date: session_close}
map as its closes. Neither has a default there: the cap is config's, and the
boundary is the calendar's.
Collect bars from t onward for this (ticker, date).
Collection stops at whichever comes first:
    - execution.max_hold_bars valid (non-halt) bars collected, or
    - last_bar(date) reached
After-market bars (hour > last_bar(date)) are never included.

Halt classification applies across all time periods (pre/regular/after-market).

For each missing bar slot up to last_bar(date):
    if slot overlaps halt interval in halts_df → "halt" (excluded from valid count)
    else                                        → "no_trade" (OHLC = prior close, volume=0)

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
    → Proceed to Step 3

--- Step 3: Time-limit / Session close exit ---

Triggered when track_label_breach() returns None (no ±3pp breach).
Exit path determined by which condition terminated Step 1:

    Case A — time-limit exit (execution.max_hold_bars valid bars collected before last_bar(date) reached):
        exit_price = close of last valid bar in effective_bars
        pnl = (exit_price - P_entry) / P_entry
        apply label by pnl threshold (same rules as Case B below)
        → no after-market fallback; no dead position

    Case B — session close exit (last_bar(date) reached within execution.max_hold_bars valid bars):
        exit_price = last_bar(date) close
        # last_bar(), not exit_deadline(): Case B is not an exit POLICY but the
        # terminal condition of Step 1's collection, so it must land where
        # collection stopped. BacktestEngine's session-close exit IS a policy
        # and uses exit_deadline(). At the default offset the two coincide; at
        # an offset of 10 the label observes to 15:59 while backtest exits at
        # 15:50. Not an error — the label is a training target and backtest
        # computes pnl from its own fills — but the observation window and the
        # exit instant then differ by nine minutes.

        if last_bar(date) is halt or no_data:
            fallback: first tick_10 row with hour > last_bar(date) (after-market)
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
    - Session close exit price cannot be determined (last_bar(date) halt + no after-market data)
    - Triggered from Step 3 Case B only — time-limit exit never leads to dead position

In this case:
    Lookup next day with has_data=True via trading_calendar table.

    Case A — has_data=True AND ticker exists in ticker_data_coverage:
        exit_price = next day pre-market first tick
                     fallback: next day ohlcv_1min first bar open
        if exit_price cannot be resolved from either source (NaN/unavailable):
            → proceed to Case D (below); do not assign a label here
        exit_price *= (1 - dead_position_penalty_pct)
        adjusted_P_entry = (P_entry - dividend_amount) / cum_split_ratio
            where cum_split_ratio = product of split/reverse_split 'value' in
                corporate_events WHERE ticker=? AND event_date IN (D, D+1]
            dividend_amount = 'dividend' value in corporate_events with
                event_date IN (D, D+1] (0.0 if none; same overnight window as
                cum_split_ratio — a split or ex-dividend date effective D+1
                is what would otherwise corrupt this comparison, since US
                splits/ex-dividend adjustments always take effect before
                market open)
        is_dead_position = True
        dead_position_case = "A"
        pnl = (exit_price - adjusted_P_entry) / adjusted_P_entry
        apply label by pnl threshold (same rules as Step 3)

    Case B — has_data=True AND ticker NOT in ticker_data_coverage:
        is_dead_position = True
        dead_position_case = "B"
        pnl = -1.0
        apply label by pnl threshold (same rules as Step 3) — resolves to label_dn5
        (possible delisting; full loss is the conservative, literal reading of
         pnl threshold definitions — see "Dead Position Cases" rationale above)

    Case C — no next day with has_data=True in dataset (boundary condition):
        is_dead_position = True
        dead_position_case = "C"
        assign label_sw directly
        (dataset boundary — actual price movement unknown)
        Note: ClassBalancer can optionally exclude Case C from training.

    Case D — Case A path entered, but exit_price unresolvable after fallback
             (extended/multi-day halt continuing past D+1):
        is_dead_position = True
        dead_position_case = "D"
        pnl = -1.0
        apply label by pnl threshold (same rules as Step 3) — resolves to label_dn5
        (same mechanism as Case B — capital is locked/at-risk with no resolvable
         exit price; dead_position_case="D" retained for downstream filtering)
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
        corporate_events:     pd.DataFrame,
    ) -> pd.DataFrame:
        """
        Generate label matrix for all entry points.

        The boundary maps are derived from the trading_calendar frame this
        method already receives, through utils.md's
        session_boundaries_from_frame(). No second query, and no second
        normalisation: that constructor solely owns which rows enter the maps.
        The frame is still needed in its own right, for has_data in dead
        position resolution, which the maps do not carry.
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
- Halt bars are excluded from the valid count — search extends to compensate
- Time-limit exit (Step 3 Case A): triggered when execution.max_hold_bars valid bars collected before last_bar(date);
  exit_price = last valid bar close; no after-market fallback; no dead position
- Session close exit (Step 3 Case B): triggered when last_bar(date) reached within execution.max_hold_bars valid bars;
  after-market fallback applies when last_bar(date) is halt/no_data
- Dead position triggered from Step 3 Case B only — time-limit exit never leads to dead position
- After-market data usage limited to last_bar(date) halt/no_data fallback only (Step 3 Case B)
- build_effective_bar_sequence() collects bars up to last_bar(date) only — after-market bars never included
- Dead position is the only case where after-market and next-day data are used
- Threshold values (threshold_3pp, threshold_5pp) must be read from config, not hardcoded
- `build_effective_bar_sequence()` is sourced from `utils.py` — do not reimplement
- `is_dead_position` flag must be set in all dead position cases regardless of label assigned
- `dead_position_case` must be set to "A", "B", "C", or "D" for dead position rows; None otherwise
- `is_ambiguous` sourced from `track_label_breach()` Stage 1 return value only;
  Stage 2 ambiguity does not affect the flag
- `ambiguity_priority` controls breach direction on simultaneous bundle breach — read from config
- `ticks_df` must be pre-loaded for the full day and passed explicitly;
  Labeler filters internally per entry point (hour >= t_hour)
- `exit_interpolation` passed through to `track_label_breach()` — read from config
- `ticker_data_coverage` must be pre-loaded and passed explicitly;
  used for dead position Case A vs Case B determination only
- Dead position Case A: pnl computed from next-day exit_price, dividend/split
  adjusted (see Step 4) via `corporate_events`; label assigned by threshold;
  if exit_price cannot be resolved, falls through to Case D instead
- Dead position Case B: pnl = -1.0 fixed; label assigned by threshold
  (resolves to label_dn5) — not label_sw
- Dead position Case C: label_sw assigned directly — no pnl threshold applied
  (dataset-boundary artifact only; distinct from B/D, which are real outcomes)
- Dead position Case D: pnl = -1.0 fixed; label assigned by threshold
  (resolves to label_dn5), identical mechanism to Case B; triggered only when
  Case A's exit_price cannot be resolved after fallback
- Case A/D overnight dividend/split lookup uses `corporate_events` rows passed
  in by the caller (see Input section) — consistent with how `trading_calendar`
  and `ticker_data_coverage` are already supplied; Labeler does not query
  DuckDB directly anywhere in this module

---

## Config Keys (pipeline_config.yaml)

```yaml
labeler:
  threshold_3pp: 0.03
  threshold_5pp: 0.05
  # session_close_hour REMOVED. It re-declared the session boundary under a
  # key of its own; the calendar is the single source whichever derivation a
  # site needs. It is NOT simply a duplicate of the former
  # execution.session_close_exit_time: the two held the same value but not the
  # same concept, this one being a COLLECTION bound. R-7's ground still
  # applies — "a value both engines must agree on cannot be declared on one
  # side only" — and the engines that must agree are backtest, live AND this
  # one. The boundary now resolves through
  # utils.md's last_bar(date), which is also what utils.md names as
  # build_effective_bar_sequence()'s collection bound, so the shared function
  # is no longer reached through this module's key on one side and
  # execution:'s on the other.
  # after_market_close_hour REMOVED. It duplicated the after-market boundary
  # already carried by trading_calendar.after_hours_end, which is what the
  # live process's R-9 hard cap resolves from and what metadata_crawler.md's
  # evening-cron derivation is stated against.
  # NOT the ingestion range, which also reads 200000: that range is fixed by
  # decision and does not move with the calendar, while after_hours_end does.
  # The two coincided at 200000, and grouping them as one boundary was the
  # coincidence doing the explaining.
  # Where this module needs it, it calls utils.md's after_hours_end(date)
  # against the map built below, rather than reading the column directly.
  # max_holding_bars REMOVED — execution.max_hold_bars is the single source.
  # R-6 established that and enumerated the consumers it knew of; this module
  # was not among them, because the key here was max_holDING_bars and no grep
  # on the established identifier could reach a near-miss name. That
  # enumeration is why a COUNT should not stand in for a list: it read as
  # complete while a member sat outside it.
  # A NEAR-MISS IDENTIFIER IS ITS OWN SWEEP AXIS — neither the literal axis
  # nor the prose axis reaches it.
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
| Neither ±3pp breached, last_bar(date) reached within execution.max_hold_bars bars | label_sw (session close exit) |
| Neither ±3pp breached, execution.max_hold_bars valid bars collected before last_bar(date) | label_sw (time-limit exit, last valid bar close) |
| Ambiguous bundle (both ±3pp in same bundle), priority="up" | label_up3/up5, is_ambiguous=True |
| Dead position Case A (ticker in coverage, has_data=True next day) | is_dead_position=True, case="A", label by pnl threshold (dividend/split adjusted) |
| Dead position Case B (ticker missing, has_data=True next day) | is_dead_position=True, case="B", pnl=-1.0, label_dn5 |
| Dead position Case C (no next day with has_data=True) | is_dead_position=True, case="C", label_sw |
| Dead position Case D (Case A entered, exit_price unresolvable — extended halt) | is_dead_position=True, case="D", pnl=-1.0, label_dn5 |
| Dead position Case A with split effective D+1 | pnl computed against split-adjusted P_entry, not raw P_entry |
| Halt bar skipped in valid count | label assigned after halt |
| Time-limit exit: execution.max_hold_bars valid bars collected, last_bar(date) not yet reached | exit at last valid bar close, no dead position |
