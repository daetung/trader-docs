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

P_entry is a reference value only. It is passed separately to the Labeler and BacktestEngine. It must never appear in the feature matrix.

---

## Feature Input Boundary

```
Allowed:   bars t-1, t-2, t-3, ..., t-N   (fully closed bars, OHLCV all confirmed)
Forbidden: t bar open, high, low, close, volume
Forbidden: any bar beyond t (t+1, t+2, ...)
```

A "fully closed bar" means all five fields (open, high, low, close, volume) are finalized. The t bar does not satisfy this condition at entry detection time.

**Lookback window (default):** last 5 trading days of 1min bars = ~1950 bars (390 min/day × 5)

Actual lookback per indicator is configured in `configs/pipeline_config.yaml`.

---

## Label Search Boundary

```
Search range: t bar open  →  t+59 bar close
              (60 minutes total)

Early termination: regular market session close (15:59 bar close, US Eastern)
                   whichever comes first
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

**t bar identification:**
The entry detector fires at the close of `t-1` bar. At that moment, the `t` bar open price is available via the first tick of the new bar or inferred from the next 1min bar's open field. The `t` bar's OHLCV is not complete and must not be used as feature input.

---

## 10-Tick Data Usage

10-tick data is used for two purposes only:

1. **Auxiliary indicator input** — TPM (ticks per minute), avg_vol_per_tick, and derived features. These are computed from ticks **up to and including t-1 bar's ticks only**.

2. **Backtest slippage estimation** — ticks within the `t` bar are used to approximate the fill price at t+5s. This is backtest-only and must not feed into the feature pipeline.

```
10-tick allowed for features:  ticks with timestamp < t bar open
10-tick allowed for backtest:  ticks with timestamp >= t bar open (slippage only)
```

---

## Summary Table

| Data | Feature Input | Label Calculation | Backtest Only |
|---|---|---|---|
| Bars t-N … t-1 (OHLCV) | YES | — | — |
| t bar open (P_entry) | NO | Reference price | Fill reference |
| t bar H/L/C/V | NO | NO | NO |
| Bars t+1 … t+59 (H/L) | NO | YES (threshold check) | — |
| 10-tick before t bar | YES | — | — |
| 10-tick within t bar | NO | NO | YES (slippage) |

---

## Checklist for Implementers

Before submitting any module, verify:

- [ ] No feature is derived from t bar's high, low, close, or volume
- [ ] P_entry (t bar open) does not appear in the feature matrix columns
- [ ] Label calculation starts from t bar open as the zero reference
- [ ] 10-tick data used in features is strictly filtered to before t bar open
- [ ] Backtest slippage uses only 10-tick data from within the t bar
