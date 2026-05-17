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

## Label Search Boundary

```
Search range: t bar open → session close (15:59 bar), up to 60 valid bars

Priority:
  1. ±3pp breach → immediate label assignment
  2. 15:59 bar reached → exit at 15:59 close, assign label by pnl
  3. 60 valid bars exhausted → exit at last valid bar close, assign label by pnl

After-market data used only as fallback when 15:59 bar is halt/no_data.
Dead position (overnight hold) only when after-market fallback also unavailable.
No overnight holds in normal flow.
```

Price thresholds are calculated relative to P_entry:

```
max_up(i)   = ( max(High of bars t..t+i) - P_entry ) / P_entry
max_down(i) = ( P_entry - min(Low of bars t..t+i)  ) / P_entry
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

10-tick data is used for two purposes only:

1. **Auxiliary indicator input** — TPM, avg_vol_per_tick, and derived features. Computed from ticks with timestamp < t bar open only.

2. **Backtest slippage estimation** — ticks within and around the t bar and exit bar are used to approximate fill prices. Backtest-only; must not feed into the feature pipeline.

```
10-tick allowed for features:  ticks with timestamp < t bar open
10-tick allowed for backtest:  ticks within t bar (entry slippage)
                                ticks within exit bar (exit slippage, if exit_interpolation=true)
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

---

## Summary Table

| Data | Feature Input | Label Calculation | Backtest Only |
|---|---|---|---|
| Bars t-N … t-1 (OHLCV, all sessions) | YES | — | — |
| t bar open time (entry.hour) | YES (temporal only) | — | — |
| t bar open price (P_entry) | NO (identifier only) | Reference price | Fill reference |
| t bar H/L/C/V | NO | NO | NO |
| Bars t+1 … session close (H/L) | NO | YES (threshold check) | — |
| After-market bars | NO | Fallback only | Fallback only |
| 10-tick before t bar | YES | — | — |
| 10-tick within t bar | NO | NO | YES (entry slippage) |
| 10-tick within exit bar | NO | NO | YES (exit slippage, optional) |

---

## Checklist for Implementers

Before submitting any module, verify:

- [ ] No feature is derived from t bar's high, low, close, or volume
- [ ] P_entry (t bar open price) does not appear in feature matrix columns
- [ ] P_entry is present as an identifier column in parquet output
- [ ] Temporal features use entry.hour (clock time only) — not t bar OHLCV
- [ ] Label calculation starts from t bar open as the zero reference
- [ ] Label search exits at 15:59 bar before exhausting 60 valid bars
- [ ] After-market used only as fallback for 15:59 halt/no_data
- [ ] 10-tick data used in features is strictly filtered to before t bar open
- [ ] Backtest slippage uses only 10-tick data from within the t bar or exit bar
