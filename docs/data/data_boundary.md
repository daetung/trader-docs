# Data Boundary Rules

> **All modules must follow these rules without exception.**
> Any code that violates these boundaries introduces data leakage and invalidates model results.

---

## Entry Point Definition

An entry point is defined by a tuple `(ticker, date, t)` where `t` is the 1-minute bar index at which the entry detector fires.

The detector fires **after `t-1` bar closes**, meaning the `t` bar has just opened but is not yet complete.

## Reference Bar Convention (Confirmed Design Decision)

```
Reference bar = t-1 bar, fixed in both training data and live inference.

Training data:  detection runs after t-1 bar close is recorded in historical data.
Live inference: even though the t bar is actively forming, detection still uses
                t-1 bar close as the reference. t bar real-time values are NOT used.

Entry execution: always at t bar open + 5s (next bar after detection fires).

Rationale:
  - Identical detection logic in training and inference → no train-serve skew
  - t bar real-time values cannot be reproduced from historical 1-min data
  - Slippage is bounded to t bar open vs t+5s fill price difference only
  - Do not change this convention without updating all dependent modules.
```

---

## Reference Price: P_entry

```
P_entry = open price of the t bar
```

**Role of P_entry:**

| Usage | Included | Reason |
|---|---|---|
| Feature input | NO | t bar is not a fully closed OHLCV bar |
| Label reference price | YES | Starting point for ±pp threshold calculation |
| Label search range start | YES | t bar open marks the entry moment |
| Backtest fill price reference | YES | Compared against t+5s estimated fill |
| Parquet identifier column | YES | Stored alongside ticker/date/hour for BacktestEngine use |

P_entry is stored in parquet files as an identifier column alongside `ticker`, `date`, `hour`.
It must never appear in `FeatureExtractor.get_feature_names()` output.

---

## Feature Input Boundary

```
Allowed:   bars t-1, t-2, t-3, ..., t-N   (fully closed bars, OHLCV all confirmed)
           may include pre-market and after-market bars within lookback window
Forbidden: t bar open, high, low, close, volume
Forbidden: any bar beyond t (t+1, t+2, ...)
```

A "fully closed bar" means all five fields (open, high, low, close, volume) are finalized.
The t bar does not satisfy this condition at entry detection time.

**Temporal features** use `entry.hour` (t bar open time) as their reference.
This is the clock time at which detection fired — it is not t bar OHLCV data
and does not constitute data leakage.

**Lookback window (default):** last 5 trading days of 1min bars = ~1950 bars (390 min/day × 5).
Actual lookback per indicator is configured in `configs/pipeline_config.yaml`.

---

## REFERENCE_SESSION Indicator Boundary

REFERENCE_SESSION indicators (rvol, rel_dvol, gap_percentile, intraday_seasonality)
use prior session data as a statistical baseline.

```
Prior session bars used as baseline:
    Status: fully closed (all OHLCV confirmed), no data leakage
    Source: precomputed_session_stats DuckDB table (pre-calculated offline)
    Scope:  prior N sessions (n_sessions config; independent of lookback_days)

Today's session bars used as the measured value:
    Boundary: same as feature input boundary — bars t-N ... t-1 only
    No future bars included

Leakage check:
    Prior session bars → raw price/volume (not model labels or predictions)
    n_sessions may reference data within or beyond the rolling window training period
    → NOT leakage (raw market data baseline, not label information)
```

**gap_percentile special rules:**

```
NaN conditions (must return NaN, not raise error):

1. t bar = "093000" (first regular session bar):
       today_open = open of 093000 bar = P_entry of this entry point
       Using P_entry as a feature is FORBIDDEN (data boundary violation)
       → return NaN

2. t < "093000" (pre-market entry):
       Regular session has not opened; today_open undefined
       → return NaN

3. Insufficient baseline data:
       count < n_sessions / 2 in precomputed_session_stats
       → return NaN

In all NaN cases: gap_pct feature = NaN in feature vector (LightGBM handles natively)
```

---

## Corporate Event Adjustment Boundary

Splits, reverse splits, and dividends require a THIRD category alongside
"raw" and "feature input" — some paths must use bars/prices exactly as they
were reported that day (raw), some must use bars corrected for split scale
(adjusted), and a few need a scalar correction applied at the point of use
rather than either of those. Mixing these up in either direction is a
correctness bug: feeding adjusted bars where raw is required corrupts actual
market-price/volume filters; feeding raw bars where adjustment is required
reintroduces the split-discontinuity problems this boundary exists to prevent.

```
Raw bars/ticks (adjustment forbidden — must reflect actual traded values):
    EntryPointDetector (filters operate on actual traded price/volume that day)
    Labeler's same-day breach tracking (track_label_breach, build_effective_bar_sequence)
    BacktestEngine's fill/exit simulation (find_fill_bundle, simulate_exit_fill)
    tick_10 — never adjusted, in any consumer
    P_entry — always the literal t bar open price, never adjusted

Adjusted bars (utils.adjust_bars_for_corporate_events() applied before use):
    FeatureExtractor/IndicatorCalculator input, all Strategy A/B indicators
        (anchored to the max date in the loaded range — today for live mode,
        the ticker's last loaded date for training; see 04_feature_extractor.md)
    caching_calculator.md session_start_compute() (anchored to today)
    populate_precomputed_session_stats() prior-session values, adjusted
        in place before aggregation into avg_value/std_value (utils.md D.)

Scalar corrections (neither raw-only nor bar-adjusted — a single value
derived from corporate_events applied at one specific comparison point):
    gap_percentile()'s dividend_amount        — ex-dividend cash-drop correction
    Dead position Case A/D's adjusted_p_entry — overnight split+dividend correction
                                                 (05_labeler.md, 09_backtest_engine.md)
    EntryPointDetector filter E's shares_outstanding — point-in-time share count
                                                        (utils.estimate_historical_meta())
```

Why EntryPointDetector and Labeler's same-day tracking stay raw even though
FeatureExtractor's bars are adjusted: these paths compare bars against
literal, contemporaneous thresholds (a $20 price cap, a 50,000-share volume
floor, a ±3pp move from the day's own P_entry) — the actual traded value IS
the correct value for these checks, on its own scale, with no cross-date
comparison involved. Adjustment only matters where values from DIFFERENT
dates are compared against each other on a common scale (indicator lookback
windows, prior-session baselines) — which is exactly the "adjusted bars"
category above, and nowhere else.

---

## Label Search Boundary

```
Search range: t bar open → session close (15:59 bar), up to 60 valid bars

Priority:
  1. ±3pp breach → immediate label assignment
  2. 15:59 bar reached → exit at 15:59 close, assign label by pnl
  3. 60 valid bars exhausted → exit at last valid bar close, assign label by pnl

Priority 2 and 3 are mutually exclusive — whichever condition is reached first determines
the exit. After-market fallback and dead position apply to priority 2 only.
Priority 3 exits at the last valid bar close directly, with no fallback needed.

After-market data used only as fallback when 15:59 bar is halt/no_data (priority 2 only).
Dead position (overnight hold) only when after-market fallback also unavailable.
No overnight holds in normal flow.
```

Price thresholds are calculated relative to P_entry:

```
max_up(i)   = ( max(High of bars t..t+i) - P_entry ) / P_entry
max_down(i) = ( P_entry - min(Low of bars t..t+i)  ) / P_entry
```

---

## Label vs. Backtest Reference Price

```
Labeler:        threshold computed relative to P_entry (t bar open price)
                → measures signal quality independent of execution

BacktestEngine: threshold computed relative to fill_price (P_entry ± slippage)
                → measures realized P&L including execution friction

Consequence:
    A trade labelled label_up5 (signal quality: good) may still produce
    a losing backtest result if slippage is large enough to push fill_price
    above the effective take-profit trigger.
    This is expected and desirable — it separates model quality from
    execution quality for diagnostic purposes.
```

---

## 1-Minute Bar Timestamp Convention

Raw JSON uses `Hour` field in `HHMMSS` format representing the **bar open time**.

```
Example: {"Hour": "093000", "Date": "20250714", ...}
→ This bar covers 09:30:00 to 09:30:59
→ Bar close time = 09:30:59 (last tick of that minute)
→ Bar is fully closed when the 09:31:00 bar begins
```

---

## 10-Tick Data Usage

tick_10 `hour` field represents the **last tick** timestamp of each 10-tick bundle (second precision).

10-tick data is used for three purposes:

1. **Auxiliary indicator input** — TPM, avg_vol_per_tick, and derived features.
   Computed from ticks with timestamp < t bar open only.

2. **Label breach detection** — ticks from t bar onward used by Labeler via
   `utils.track_label_breach()` to detect ±3pp/±5pp breaches at sub-minute precision.
   Tracking starts after the fill_bundle (fill_second = t bar open + 5s).
   Backtest-only rule does NOT apply — label calculation requires tick precision.

3. **Backtest slippage estimation** — ticks within and around the t bar and exit bar
   used to approximate fill prices and simulate partial exit fills.
   Backtest-only; must not feed into the feature pipeline.

4. **REFERENCE_SESSION baseline (offline)** — prior session 10-tick data used by
   `populate_precomputed_session_stats()` to compute buy_ratio_baseline and
   intra_tpm_baseline via Lee-Ready classification.
   Computed offline (migration/daily update); not computed at runtime.

```
10-tick allowed for features:          ticks with timestamp < t bar open
10-tick allowed for label calculation: ticks from t bar onward (breach detection)
10-tick allowed for backtest:          ticks from t bar onward (slippage simulation)
10-tick allowed for session stats:     all prior session ticks (offline computation only)
```

---

## Session Mode and Data Coverage

All data is stored without time-of-day filtering.
Pre-market, regular session, and after-market bars are all ingested and available.

Session mode (`entry_detector.session_mode`) controls which entry points are used during training:
```
"regular"  : regular session entry points only (093000~155900) [default]
"pre"      : pre-market entry points only (040000~092900)
"combined" : both pre-market and regular session
```

After-market entry points (hour > 155900) are excluded in all modes.
Session mode filtering is applied at the training stage — preprocessing runs for all entry points.

Session mode also controls which prior session bars are used in REFERENCE_SESSION baselines:
```
"regular"  : prior regular session bars only (093000~155900)
"pre"      : prior pre-market bars only (040000~092900)
"combined" : both pre-market and regular session bars
```

Applied in `load_session_stats()` at load time — not stored separately in precomputed_session_stats.

---

## Summary Table

| Data | Feature Input | Label Calculation | Backtest Only | Session Stats (offline) |
|---|---|---|---|---|
| Bars t-N … t-1 (OHLCV, all sessions) | YES | — | — | — |
| t bar open time (entry.hour) | YES (temporal only) | — | — | — |
| t bar open price (P_entry) | NO (identifier only) | Reference price | Fill reference | — |
| t bar H/L/C/V | NO | NO | NO | — |
| Bars t+1 … session close (H/L) | NO | YES (threshold check) | — | — |
| After-market bars | NO | Fallback only | Fallback only | — |
| 10-tick before t bar | YES | — | — | — |
| 10-tick within t bar | NO | YES (track_label_breach) | YES (entry slippage) | — |
| 10-tick within exit bar | NO | YES (track_label_breach) | YES (exit slippage) | — |
| Prior session bars (all) | NO (not in bars input) | — | — | YES (baseline) |
| Prior session 10-tick | NO | — | — | YES (buy_ratio, tpm) |

Corporate-event (split/dividend) adjustment is a separate, orthogonal axis not
shown in this table — see "Corporate Event Adjustment Boundary" above. As a
quick reference: every row that reads from `ohlcv_1min`/`tick_10` directly for
same-day use (Label Calculation, Backtest Only columns) stays raw; every row
consumed via FeatureExtractor/IndicatorCalculator or session-stats baselines
is bar-adjusted.

---

## Checklist for Implementers

Before submitting any module, verify:

- [ ] No feature is derived from t bar's high, low, close, or volume
- [ ] P_entry (t bar open price) does not appear in feature matrix columns
- [ ] P_entry is present as an identifier column in parquet output
- [ ] Temporal features use entry.hour (clock time only) — not t bar OHLCV
- [ ] Label calculation starts from t bar open as the zero reference
- [ ] When no ±3pp breach: if 60 valid bars collected before 15:59 → exit at last valid bar close
       (time-limit; no after-market fallback, no dead position);
       if 15:59 reached within 60 valid bars → exit at 15:59 close
       (session-end; with after-market fallback and dead position as last resort)
- [ ] After-market used only as fallback for 15:59 halt/no_data (session-end path only)
- [ ] 10-tick data used in features is strictly filtered to before t bar open
- [ ] Labeler uses track_label_breach() with fill_second = t bar open + 5s;
       tracking starts strictly after fill_bundle (fill_bundle excluded)
- [ ] Backtest entry slippage uses 10-tick data with search window = entry_hour to entry_hour + 100s
- [ ] Backtest exit slippage uses simulate_exit_fill() from breach bundle onward (full day ticks)
- [ ] REFERENCE_SESSION baselines sourced from precomputed_session_stats — not from bars input
- [ ] gap_percentile returns NaN for t="093000" or pre-market entries (not an error)
- [ ] Dead position lookup uses has_data=TRUE filter (not is_trading_day)
- [ ] EntryPointDetector and Labeler/BacktestEngine's same-day tracking read
       raw bars/ticks only — never `adjust_bars_for_corporate_events()` output
- [ ] FeatureExtractor/IndicatorCalculator and `populate_precomputed_session_stats()`
       always operate on corporate-event-adjusted bars, never raw
- [ ] gap_percentile's `dividend_amount`, dead position's `adjusted_p_entry`,
       and filter E's `shares_outstanding` are scalar corrections applied at
       their one specific comparison point — not a substitute for, and not
       satisfied by, bar-level adjustment elsewhere in the same module
