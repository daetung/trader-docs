# Module: EntryPointDetector

**File:** `src/detection/entry_detector.py`
**Depends on:** `docs/data/data_boundary.md` — read first

---

## Role

Identify candidate entry points from historical 1-minute bar data.
Each candidate is a `(ticker, date, t)` tuple where `t` is the bar index of the **reference bar (t-1 bar)** — the last fully closed bar before the entry moment.

The output of this module is the input to the Preprocessor pipeline.

---

## Reference Bar Convention

```
Reference bar = t-1 bar (last fully closed 1-minute bar)

Detection fires after t-1 bar closes.
All conditions are evaluated against t-1 bar and earlier bars only.
Entry execution is always at the next bar open + 5s (t bar open + 5s).

This convention is identical in both training data and live inference.
→ No train-serve skew. Do not change this.
```

---

## Session Mode

Entry point detection scope is controlled by `session_mode` config:

```
"regular"  : entry points from regular session only (093000~155900) [default]
"pre"      : entry points from pre-market only (040000~092900)
"combined" : entry points from both pre-market and regular session
```

After-market (hour > 155900) entry points are excluded in all modes.

Session mode filtering is applied during the **training stage only**.
The preprocessing stage runs EntryPointDetector for all entry points regardless
of session mode. Filtering by session mode is applied in run_train.py when
loading preprocessed data.

---

## Filter Conditions

Each condition is implemented as an independent method returning `bool`.
The composite expression is assembled in `detect()` and configured via `pipeline_config.yaml`.

### A — Price Range
```
Current price is between $0 (exclusive) and $20 (inclusive).
Current price proxy = t-1 bar close.

method: filter_A_price_range(bars, max_price=20.0) -> bool

logic:
    price = bars.iloc[-1]["close"]   # t-1 bar close
    return 0 < price <= max_price
```

### B — 1-Minute Volume
```
t-1 bar volume >= 50,000.

method: filter_B_volume_1min(bars, min_volume=50000) -> bool

logic:
    return bars.iloc[-1]["volume"] >= min_volume
```

### C — 5-Bar Average Volume
```
Simple average of the last 5 bars (t-5 to t-1, inclusive) >= 1,000.

method: filter_C_avg_volume_5bar(bars, n=5, min_avg=1000) -> bool

logic:
    return bars.iloc[-n:]["volume"].mean() >= min_avg
```

### D — Daily Cumulative Volume
```
Cumulative volume from volume_base_hour through t-1 bar >= 100,000.

method: filter_D_daily_volume(bars, date, min_volume=100000) -> bool

logic:
    today_bars = bars[(bars["date"] == date) & (bars["hour"] >= volume_base_hour)]
    return today_bars["volume"].sum() >= min_volume

volume_base_hour: from config (default: "040000")
```

### E — Daily Volume Turnover Rate
```
Turnover rate = cumulative volume from volume_base_hour / shares_outstanding * 100 >= 5.0 (%)

shares_outstanding sourced from stock_meta table.

method: filter_E_turnover_rate(bars, date, shares_outstanding, min_rate=5.0) -> bool

logic:
    today_bars = bars[(bars["date"] == date) & (bars["hour"] >= volume_base_hour)]
    daily_volume = today_bars["volume"].sum()
    turnover = daily_volume / shares_outstanding * 100
    return turnover >= min_rate

volume_base_hour: from config (default: "040000")
```

### F — Proximity to Today's Maximum Volume Bar
```
t-1 bar volume is within 30% of the highest volume bar seen today,
excluding t-1 bar itself from the max calculation.

"Within 30%" means: t-1 bar volume >= today's max volume (excl. t-1) * 0.70

method: filter_F_volume_proximity(bars, date) -> bool

logic:
    today_bars = bars[(bars["date"] == date) & (bars["hour"] >= volume_base_hour)]
    prior_bars = today_bars.iloc[:-1]          # exclude t-1 bar
    if prior_bars.empty:
        return False
    max_volume = prior_bars["volume"].max()
    ref_volume  = today_bars.iloc[-1]["volume"] # t-1 bar
    return ref_volume >= max_volume * 0.70

volume_base_hour: from config (default: "040000")
```

### G — Volume Ratio vs 20-Bar Average
```
t-1 bar volume >= 250% of the mean volume of the 20-bar window
ending at t-1 bar (i.e., bars t-20 to t-1, inclusive).

The reference bar (t-1) IS included in the average denominator.

method: filter_G_volume_ratio(bars, n=20, min_ratio=2.5) -> bool

logic:
    window = bars.iloc[-n:]            # t-20 to t-1 (n=20 bars)
    avg_volume = window["volume"].mean()
    ref_volume  = bars.iloc[-1]["volume"]  # t-1 bar
    return ref_volume >= avg_volume * min_ratio
```

---

## Composite Expression

```
(A and B and C and D) and (E or F or G)
```

This is the default expression. It must be configurable without code changes.
See config section below for how to modify the expression.

---

## Interface

```python
class EntryPointDetector:

    def filter_A_price_range(
        self, bars: pd.DataFrame, max_price: float = 20.0
    ) -> bool: ...

    def filter_B_volume_1min(
        self, bars: pd.DataFrame, min_volume: int = 50000
    ) -> bool: ...

    def filter_C_avg_volume_5bar(
        self, bars: pd.DataFrame, n: int = 5, min_avg: float = 1000.0
    ) -> bool: ...

    def filter_D_daily_volume(
        self, bars: pd.DataFrame, date: str, min_volume: int = 100000
    ) -> bool: ...

    def filter_E_turnover_rate(
        self, bars: pd.DataFrame, date: str,
        shares_outstanding: int, min_rate: float = 5.0
    ) -> bool: ...

    def filter_F_volume_proximity(
        self, bars: pd.DataFrame, date: str, proximity_pct: float = 0.30
    ) -> bool: ...

    def filter_G_volume_ratio(
        self, bars: pd.DataFrame, n: int = 20, min_ratio: float = 2.5
    ) -> bool: ...

    def detect(
        self,
        bars: pd.DataFrame,           # all bars up to and including t-1
        date: str,                    # current trading date 'YYYYMMDD'
        shares_outstanding: int,      # from stock_meta
    ) -> bool:
        """
        Evaluate composite expression and return True if entry candidate.
        Expression and thresholds are read from pipeline_config.yaml.
        """
        ...

    def scan(
        self,
        bars: pd.DataFrame,           # full date range, multiple dates
        ticker: str,
        meta: dict,                   # stock_meta row for this ticker
    ) -> pd.DataFrame:
        """
        Scan all bars for a ticker and return all detected entry points.
        Includes all session modes — session_mode filtering applied at training stage.
        After-market entry points (hour > 155900) are always excluded.

        Returns:
            pd.DataFrame with columns [ticker, date, hour, p_entry]
            where hour = t bar open time, p_entry = t bar open price
        """
        ...
```

---

## Output Schema

```python
# Output of scan() — matches entry_points table in DuckDB
pd.DataFrame(columns=[
    "ticker",    # str
    "date",      # 'YYYYMMDD'
    "hour",      # 'HHMMSS' — t bar open time (NOT t-1 bar)
    "p_entry",   # float — t bar open price
])
```

Note: `hour` in the output refers to the **t bar** (entry execution bar), not the t-1 reference bar. The relationship is:

```
detection fires at close of bar with hour = H
→ output hour = next bar open = H + 1 minute
→ p_entry = open price of that next bar
```

---

## Class Balance Note

The sideways class is expected to dominate under this condition set.
If sideways samples exceed the configured ratio after initial scan, the detector
must support criteria refinement. Planned extension (not in this phase):

```python
def scan_with_balance_check(
    self,
    bars: pd.DataFrame,
    ticker: str,
    meta: dict,
    max_sideways_ratio: float = 0.6,   # from config
) -> pd.DataFrame:
    """
    Run scan(), check class distribution after labeling.
    If sideways ratio exceeds threshold, tighten filter G or F threshold
    and re-scan. Log which threshold was applied.
    """
    ...
```

This method is a placeholder. Implementation deferred until initial label
distribution is observed from the first full scan.

---

## Config Keys (pipeline_config.yaml)

```yaml
entry_detector:
  # Session mode
  session_mode: "regular"        # "pre" | "regular" | "combined"
  volume_base_hour: "040000"     # base hour for D, E, F conditions

  # Individual filter thresholds
  A_max_price: 20.0
  B_min_volume_1min: 50000
  C_avg_volume_bars: 5
  C_min_avg_volume: 1000
  D_min_daily_volume: 100000
  E_min_turnover_rate: 5.0       # percent
  F_proximity_pct: 0.30
  G_volume_ratio_bars: 20
  G_min_ratio: 2.5               # 250%

  # Composite expression
  expression: "(A and B and C and D) and (E or F or G)"

  # Class balance control (deferred)
  max_sideways_ratio: 0.6
```

---

## Design Constraints

- Each filter method must be independently testable (no cross-dependencies between filters)
- `detect()` must parse and evaluate `expression` from config — do not hardcode the boolean logic
- All bars passed in must already be filtered to `t-1 bar` and earlier — this module does not enforce the data boundary itself, but the caller (`scan()`) must guarantee it
- `shares_outstanding` must never be computed inside this module — always passed in from stock_meta
- Filter F returns `False` (not an error) when no prior bars exist for the day (first bar of session edge case)
- After-market entry points (hour > 155900) must never be returned from `scan()`
- `volume_base_hour` applied to conditions D, E, F only — condition G uses rolling window, no base hour filter

---

## Test Cases Required (tests/test_detector.py)

| Scenario | Expected |
|---|---|
| All A-D pass, E passes | detect() = True |
| All A-D pass, only G passes | detect() = True |
| All A-D pass, E/F/G all fail | detect() = False |
| A fails (price > 20) | detect() = False |
| B fails, E passes | detect() = False |
| F: t-1 bar is the only bar today | filter_F returns False |
| G: fewer than 20 bars available | use available bars, no error |
| Expression changed in config to "A and B" | detect() respects new expression |
| volume_base_hour = "093000", pre-market bars present | D/E/F exclude pre-market bars |
| after-market bar as t-1 | scan() excludes from output |
