# Module: EntryPointDetector

**File:** `src/detection/entry_detector.py`
**Depends on:** `docs/data/data_boundary.md` — read first

---

## Role

Identify candidate entry points from historical 1-minute bar data.
Each candidate is a `(ticker, date, t)` tuple where `t` is the bar index at which the entry detector fires.

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

## scan() vs detect() — Role Separation

```
scan():   Training only.
          Accepts full multi-date bars DataFrame including t bar.
          Iterates over bars, calls detect() at each t-1 bar position,
          and retrieves p_entry from bars[i+1]["open"] (t bar open price).
          If t bar does not exist (last bar in DataFrame), skip that entry point.
          Saves results to DuckDB entry_points table via Preprocessor.
          Must NOT be called in live inference mode.

detect(): Training and live inference.
          Evaluates composite filter conditions against t-1 bar and earlier only.
          Returns bool — does not access or return p_entry.
          In live inference, Inferencer calls detect() directly.
          p_entry is obtained from external real-time feed by Inferencer,
          not from bars DataFrame.
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
of session mode. Filtering by session mode is applied in ClassBalancer.split()
when loading preprocessed data.

---

## Late-Day Entry Exclusion

Entry points detected too close to session close are excluded to avoid
label horizon distortion. When fewer than 60 valid bars remain before 15:59,
the label is dominated by session-close exit logic rather than genuine
price movement — creating a systematic bias.

```
max_entry_hour: configurable cutoff (default: null — no cutoff applied)

When max_entry_hour is set (e.g. "150000"):
    entry points with t bar open time > max_entry_hour are excluded
    from scan() output.

Applied after after-market exclusion, before session_mode filtering.
```

Rationale: at 15:00, only ~60 valid bars remain before 15:59 close.
Entries at or after 15:00 are structurally label-distorted.
Setting `max_entry_hour: "150000"` is recommended for regular session mode.
For pre-market mode, this setting has no practical effect.

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

shares_outstanding for the SPECIFIC date being evaluated — resolved by the
caller (scan(), or Inferencer in live mode) via stock_meta(ticker, date) with
utils.estimate_historical_meta() fallback (see 04_feature_extractor.md V-1
fix — same shared utility, not duplicated here). This module still never
computes shares_outstanding itself (see Design Constraints) — it only
receives the already-resolved scalar for this method call's date.

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

## Monotonicity of the Conditions

A property of the conditions as written, recorded because a caller may
evaluate them against a bar that is still FORMING and needs to know which
verdicts can reverse. `detect()` itself is unchanged and always receives
bars strictly up to t-1.

The question is whether a condition, once TRUE while the bar is growing, can
become FALSE by the time that bar closes.

**MONOTONE — cannot revert.** In each, the left side only grows as volume
accumulates while the right side is fixed for the duration of the bar:

| | Left side | Right side while the bar forms |
|---|---|---|
| B | forming bar's volume | constant 50000 |
| C | mean of the last 5 bars, four of them closed | constant 1000 |
| D | cumulative volume from `volume_base_hour` | constant 100000 |
| E | that same cumulative / shares_outstanding | constant 5.0 |
| F | forming bar's volume | `0.70 * max(volume)` over PRIOR bars |
| G | forming bar's volume | ratio against a 20-bar mean |

F is monotone **because of a detail in its own definition**: `prior_bars =
today_bars.iloc[:-1]` excludes the reference bar from the maximum, so the
threshold is computed entirely from CLOSED bars and cannot move while the
reference bar grows. Had the max included the reference bar, F would be
neither monotone nor useful.

G is monotone despite the reference bar being INSIDE its own denominator.
Writing `v` for the forming bar's volume and `S` for the other 19 bars'
total, `v >= ((S + v) / 20) * 2.5` reduces to `v >= S / 7`, with `S` fixed —
so the condition is increasing in `v` like the rest.

**NON-MONOTONE — A, alone.** `price = bars.iloc[-1]["close"]` is the forming
bar's current close, and price moves in BOTH directions: a ticker can pass A
at 19.80 and close at 20.10, or fail it at 20.10 and close at 19.80. So a
mid-bar evaluation of the full expression is neither a subset nor a superset
of what bar close would produce, unless A is handled explicitly.

**What the property is for.** `live_mode_runner.md`'s watchdog scan
evaluates B-G normally on a forming bar and RELAXES A to PASS. Because
every other condition is monotone, that result is a provable SUPERSET of the
bar-close candidate set — it can contain a ticker that closes out of the
price range, but it cannot MISS one that qualifies. The scan uses it to
decide which tickers to fetch completed bars for; the entry decision is
always the full expression over the completed bar, so the reference-bar
convention and train-serve parity hold unchanged.

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
        bars: pd.DataFrame,           # strictly bars t-N ... t-1 (t bar excluded)
        date: str,                    # current trading date 'YYYYMMDD'
        shares_outstanding: int,      # from stock_meta
    ) -> bool:
        """
        Evaluate composite expression and return True if entry candidate.
        Expression and thresholds are read from pipeline_config.yaml.

        Training and live inference compatible.
        Does not access or return p_entry — caller is responsible for
        obtaining t bar open price from appropriate source:
            Training:      bars[i+1]["open"] in scan()
            Live inference: external real-time feed in Inferencer
        """
        ...

    def scan(
        self,
        bars: pd.DataFrame,           # full date range including t bar
        ticker: str,
        meta: dict,                   # date-keyed: {date: {"shares_outstanding": value}}
    ) -> pd.DataFrame:
        """
        Training only. Scan all bars for a ticker and return all detected
        entry points.

        bars must include the t bar for each detected entry point so that
        p_entry = bars[i+1]["open"] can be retrieved. If t bar does not
        exist (last bar in DataFrame), that entry point is skipped.

        meta: date-keyed dict, {date_str: {"shares_outstanding": value}}, one
            entry per date present in `bars`. Required because a single
            scan() call spans this ticker's ENTIRE bar history (many dates),
            and shares_outstanding can change over that span (splits) — a
            single flat value shared across all dates would apply one date's
            share count to every other date, the same staleness bug fixed in
            04_feature_extractor.md's extract_batch() `meta` parameter (see
            V-1 fix there for the full rationale). The caller
            (run_preprocess.md) resolves this per date via
            stock_meta(ticker, date) with utils.estimate_historical_meta()
            fallback, keyed on `bars["date"].unique()` — a superset of the
            dates that will actually become entry points, resolved before
            entry points are known (chicken-and-egg: scan()'s own output is
            what determines final entry dates).
            At each t-1 position, scan() looks up meta[date]["shares_outstanding"]
            for filter E and passes it as detect()'s scalar `shares_outstanding`
            argument for that call only.

        Must NOT be called in live inference mode.
        In live inference, Inferencer calls detect() directly and obtains
        p_entry from external real-time feed; shares_outstanding for detect()
        is resolved by Inferencer for that single date only (no date-keying
        needed there, since a live call handles exactly one date: today).

        Exclusion order applied inside scan():
          1. After-market entry points (hour > 155900) always excluded
          2. Late-day entry points (hour > max_entry_hour) excluded
             when max_entry_hour is not null
          3. Session_mode filtering applied at ClassBalancer.split() stage,
             not here

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
    "p_entry",   # float — t bar open price (bars[i+1]["open"] in training)
])
```

Note: `hour` in the output refers to the **t bar** (entry execution bar), not the t-1 reference bar. The relationship is:

```
detection fires at close of bar with hour = H
→ output hour = next bar open = H + 1 minute
→ p_entry = open price of that next bar (bars[i+1]["open"])
→ if bars[i+1] does not exist → skip (last bar edge case)
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
    meta: dict,                        # date-keyed, same shape as scan()
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

  # Late-day exclusion
  max_entry_hour: "150000"       # null = no cutoff; "150000" recommended for regular session
                                 # Applied in scan() before session_mode filter

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
- `detect()` receives bars strictly up to t-1; caller guarantees this boundary
- `scan()` receives bars including t bar; retrieves p_entry from bars[i+1]["open"]
- `scan()` is training only — must not be called in live inference
- `shares_outstanding` must never be computed inside this module — always passed in from stock_meta
- `scan()`'s `meta` parameter is date-keyed (`{date: {"shares_outstanding": value}}`),
  not a flat per-ticker dict — required because a single `scan()` call spans
  the ticker's entire bar history; resolution (real value with
  `utils.estimate_historical_meta()` fallback) is the caller's responsibility,
  consistent with this module never computing metadata values itself
- Filter F returns `False` (not an error) when no prior bars exist for the day (first bar of session edge case)
- After-market entry points (hour > 155900) must never be returned from `scan()`
- Late-day entry points (hour > max_entry_hour) excluded in `scan()` when max_entry_hour is not null
- `max_entry_hour` exclusion applied before session_mode filter — both are independent constraints
- `volume_base_hour` applied to conditions D, E, F only — condition G uses rolling window, no base hour filter
- Condition A is the only NON-MONOTONE condition (see Monotonicity of the
  Conditions). A change that makes any of B-G able to revert as its bar
  grows — most easily by letting F's maximum include the reference bar —
  silently invalidates the superset guarantee `live_mode_runner.md`'s
  mid-bar screen depends on, without any test in this module failing

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
| max_entry_hour = "150000", t bar hour = "150100" | scan() excludes from output |
| max_entry_hour = "150000", t bar hour = "150000" | scan() includes (boundary inclusive) |
| max_entry_hour = null | no late-day exclusion applied |
| t bar missing (last bar in DataFrame) | scan() skips that entry point |
