# Data Boundary Rules

> **Read this document before implementing any pipeline module.**
> These rules define what data is permitted as input to each stage.
> Violating these rules introduces data leakage and invalidates all results.

---

## Entry Point Convention

```
t bar    = the 1-minute bar at which entry is executed
t-1 bar  = the last fully closed bar before entry (last bar visible to detector)

Detection fires after t-1 bar closes.
Entry execution is at t bar open + 5s.
P_entry = t bar open price (reference price for labels and backtest).
```

---

## Feature Input Boundary

```
PERMITTED as feature input:
    Bars t-1, t-2, ..., t-N     (fully closed OHLCV bars)
    tick_10 with timestamp < t bar open
    entry.hour (t bar open time) — temporal features only, not OHLCV

FORBIDDEN as feature input:
    t bar high / low / close / volume
    P_entry (t bar open price) — identifier column only, never a feature
    Any bar after t bar
```

P_entry is the execution reference price. It is stored as an identifier column
in parquet output and used by Labeler and BacktestEngine. It must never appear
as a feature column.

`entry.hour` is the clock time at which detection fired — it is not t bar OHLCV data
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

## Label vs. Backtest Reference Price

Labeler and BacktestEngine use the same P_entry definition but apply it to
different reference points. This separation is intentional:

```
Labeler:        threshold computed relative to P_entry (t bar open price)
                → measures signal quality independent of execution
                → labels reflect directional price movement from the ideal fill

BacktestEngine: threshold computed relative to fill_price (P_entry ± slippage)
                → measures realized P&L including execution friction
                → winning_rate in experiment_log reflects execution reality

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
   Provides finer-grained ambiguity detection (bundle level vs. bar level).

3. **Backtest slippage estimation** — ticks within and around the t bar and exit bar
   used to approximate fill prices and simulate partial exit fills.
   Backtest-only; must not feed into the feature pipeline.

```
10-tick allowed for features:          ticks with timestamp < t bar open
10-tick allowed for label calculation: ticks from t bar onward (breach detection via
                                        track_label_breach(); fill_second = t open + 5s;
                                        tracking starts after fill_bundle)
10-tick allowed for backtest:          ticks from t bar onward (entry slippage,
                                        exit tracking, partial fill simulation)
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
| 10-tick within t bar | NO | YES (track_label_breach) | YES (entry slippage) |
| 10-tick within exit bar | NO | YES (track_label_breach) | YES (exit slippage) |

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
- [ ] Labeler uses track_label_breach() with fill_second = t bar open + 5s
- [ ] Backtest slippage uses 10-tick data from within the t bar (search_limit = t open + 100s)
