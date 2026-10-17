# Module: BacktestEngine

**File:** `src/backtest/engine.py`
**Depends on:** `docs/data/data_boundary.md`

---

## Role

Simulate trading performance of the trained model on the test split.
Compute winning rate and trade statistics using realistic position management,
slippage approximation, volume-capped partial exit simulation, and position sizing rules.

BacktestEngine accesses OHLCV and tick data directly from DuckDB —
callers do not pass chart data as arguments.

---

## Input

```python
predictions: pd.DataFrame
    # columns: [ticker, date, hour, p_entry,
    #           prob_up5, prob_up3, prob_sw, prob_dn3, prob_dn5,
    #           label_up5, label_up3, label_sw, label_dn3, label_dn5]
    # one row per entry point candidate in test split

config: dict
db_conn: duckdb.DuckDBPyConnection   # injected via constructor
```

OHLCV and tick data are fetched internally from DuckDB per ticker/date as needed.

---

## Class Interface

```python
class BacktestEngine:
    def __init__(
        self,
        config: dict,
        db_conn: duckdb.DuckDBPyConnection,
    ):
        """
        Loads the boundary maps via utils.md's load_session_boundaries(db_conn),
        once for the run. ACQUIRED rather than received because run_backtest.py
        holds none to give — unlike the Inferencer, whose LiveModeRunner already
        has them. The asymmetry is about which caller holds the maps, not about
        the kind of module.

        The trading_calendar read elsewhere in this file is a one-row
        next-trading-day lookup for dead position resolution and is unrelated.
        """
        ...

    def run(
        self,
        predictions: pd.DataFrame,
    ) -> tuple[pd.DataFrame, dict]:
        """
        Run full backtest simulation.
        Returns (trade_log_df, summary_dict).

        trade_log_df: one row per executed trade — matches trade_log table schema.
                      Written to DuckDB trade_log table by Backtester (run_backtest.py),
                      not by BacktestEngine itself.

        summary_dict keys:
            # Denominators below count OPENED trades only. Rows in the
            # never-opened family (exit_reason 'entry_canceled', and live's
            # 'entry_rejected' — see db_schema.md) have quantity=0 and
            # pnl_pct=NULL: no position ever existed, so counting them
            # would dilute winning_rate with non-events. They remain in
            # trade_log and stay visible via trades_by_exit.
            winning_rate        : float  — winning_trades / total_trades
            total_trades        : int    — opened trades only
            winning_trades      : int
            avg_pnl_pct         : float
            total_pnl_abs       : float
            avg_slippage_pct    : float
            dead_position_count : int
            dead_position_rate  : float
            suppressed_count    : int    — entries blocked by suppress_threshold
            # Entry-gate rejection counters (R-5), one per gate of the
            # Chronological Simulation's admission sequence. A candidate
            # stopped at a gate produces NO trade_log row, so these are its
            # only record. Diagnostic only — best_config() ranks on
            # winning_rate alone. Five, not eleven: `gate_result` has
            # eleven non-'submitted' values;
            # 'freeze'/'not_tradable'/'bar_integrity'/'no_terms' have no
            # backtest concept (N-5 — 'bar_integrity' is a live
            # bar-arrival-latency signal, so there is nothing for backtest
            # to detect, and 'no_terms' is the outcome of a vendor call
            # backtest never makes), 'breaker' is evaluated but never
            # enforced here (see breaker_* below, not a blocked-count,
            # which would wrongly imply enforcement), and 'error' has no
            # analogue since backtest makes no network calls. The counter
            # count is unchanged: six excluded of eleven, as five of ten. Key names use
            # inference_log.gate_result's own values so live and backtest
            # are directly comparable.
            gate_blocked_cap_tickers     : int  — execution.max_tickers
            gate_blocked_cap_per_ticker  : int  — execution
                                                  .max_positions_per_ticker
            gate_blocked_cooldown        : int  — can_enter()
            gate_blocked_sizing_zero     : int  — compute_position_size() → 0
            gate_blocked_funds           : int  — check_funds_available()
                                           # returned proceed=False. Always 0
                                           # when initial_cash == 0, because
                                           # the gate is never called — that
                                           # is "did not run", not "blocked
                                           # nothing". Counts only the full
                                           # skip; under use_all_cash: true a
                                           # short-on-cash candidate is
                                           # usually sized DOWN and proceeds,
                                           # which shows up in
                                           # trade_log.requested_quantity
                                           # instead.
            # R-4 breaker metrics — COMPUTED every run, same as live,
            # but NEVER ENFORCED (see "Circuit Breaker" note below this
            # table). Reporting the raw metrics rather than a trip count is
            # deliberate: with the thresholds defaulting to 0 (no limit), a
            # trip count would read zero on every run regardless of how
            # close a run came, telling Pilot nothing about where to set
            # them.
            breaker_max_realized_loss_abs   : float  — always populated
            breaker_max_realized_loss_pct   : float | None  — None when
                                           # initial_cash == 0 (inf mode has
                                           # no equity basis to divide by;
                                           # 0.0 would misread as "no loss")
            breaker_max_consecutive_losses  : int
            breaker_peak_entries_per_hour   : int
            trades_by_signal    : dict   — {signal: {count, win_rate, avg_pnl}}
            trades_by_exit      : dict   — {exit_reason: count}
        """
        ...
```

---

## DB Access Strategy

### OHLCV (per ticker)
```sql
SELECT * FROM ohlcv_1min
WHERE ticker = ?
  AND date IN (entry_dates_for_this_ticker)
ORDER BY date, hour
```
Loaded once per ticker at the start of that ticker's processing.
Covers entry dates only — not the full dataset.

### Tick data (per ticker/date — full day)
```sql
SELECT * FROM tick_10
WHERE ticker = ? AND date = ?
ORDER BY hour, seq_id
```
Loaded once per ticker/date combination.
Full day loaded (including after-market), expanded into synthetic ticks by
utils.md's `expand_synthetic_ticks()`, and filtered in memory:
- Entry fills and in-window breach: synthetic ticks whose second is greater
  than the anchor and at most anchor + cancel_after_seconds (the entry tick
  range — Entry Slippage Model)
- Exit tracking after the entry window: synthetic ticks through the
  continuation's end, then bundles passed as ticks_future to
  track_price_breach (Exit Logic)
- Exit fills: synthetic ticks whose second is greater than the exit anchor,
  through the full day (Exit fill simulation)

### Trading halts (per ticker/date)
```sql
SELECT * FROM trading_halts
WHERE ticker = ? AND date = ?
```
Loaded once per ticker/date alongside OHLCV and tick data.
Passed explicitly to all internal methods.

### Trading calendar (for dead position resolution)
```sql
SELECT * FROM trading_calendar
WHERE date > ? AND has_data = TRUE
ORDER BY date LIMIT 1
```
Filters to next date where `has_data = TRUE` — consistent with Labeler dead position logic.

### Ticker data coverage (for dead position Case B)
```sql
SELECT 1 FROM ticker_data_coverage
WHERE ticker = ? AND date = ?
```

### Corporate events (for dead position Case A/D dividend/split adjustment)
```sql
SELECT event_type, event_date, ratio, amount, source, basis_date FROM corporate_events
WHERE ticker = ? AND event_date > ? AND event_date <= ?
```
Queried only when Case A/D resolution is reached — not on every trade.
Same window and rationale as Labeler's equivalent lookup (see `05_labeler.md`).
The split factor is `utils.md`'s single rule, applied through
`cum_split_ratio()` over (D, D+1]; a 'dividend' row's amount is read only
through `utils.md`'s dividend amount function, never from `amount` directly.

**Iteration order is DATE-major, not ticker-major.** The concurrency caps
(`execution.max_tickers` / `execution.max_positions_per_ticker`) and the
cash-ordering rule below are both properties of a moment in time across
ALL tickers, so neither can be evaluated inside a per-ticker pass. Loading
stays keyed by (ticker, date) as below — only the ORDER of processing
changes. See "Chronological Simulation" for the loop itself.

For each (ticker, date) group, the following data is loaded from DuckDB once
and passed down to internal methods:

```python
# Loaded once per ticker
ohlcv_ticker = db_conn.execute("""
    SELECT * FROM ohlcv_1min
    WHERE ticker = ? AND date IN (?)
    ORDER BY date, hour
""", [ticker, *dates]).df()

# Loaded once per ticker/date
ticks_td = db_conn.execute("""
    SELECT * FROM tick_10 WHERE ticker = ? AND date = ?
    ORDER BY hour, seq_id
""", [ticker, date]).df()

halts_td = db_conn.execute("""
    SELECT * FROM trading_halts WHERE ticker = ? AND date = ?
""", [ticker, date]).df()

# halts_td passed explicitly to all internal methods
synthetic, effective_end = utils.expand_synthetic_ticks(ticks_td, halts_td)
anchor     = utils.hour_add_seconds(
    entry_hour, config["execution"]["entry_fill_delay_seconds"])
window_end = utils.hour_add_seconds(
    anchor, config["execution"]["cancel_after_seconds"])
# Entry fills and breach run on one timeline over `synthetic` — Entry
# Slippage Model and Exit Logic below.
```

---

## Position Sizing

Sizing moved to `docs/utils/execution_common.md`'s `compute_position_size()`
— shared with LiveModeRunner. BacktestEngine's role is to supply the two
`balance`-adjacent values correctly:

```
initial_cash:   float   # from config; fixed for the entire run — the
                        # `balance` argument to compute_position_size().
                        # 0 = unlimited (inf mode): position_size_cash_pct
                        # leg is skipped, sizing is vol_based-only.
remaining_cash: float   # runtime value, starts at initial_cash. REVOLVING:
                        # decremented by the actually-committed amount when
                        # an entry fills, and credited back when the
                        # position closes (see Chronological Simulation).
                        # A one-way budget would model a single deployment
                        # rather than a book that turns over, and would
                        # silently stop trading partway through any run with
                        # initial_cash > 0. NEVER passed as
                        # `balance` — see execution_common.md's Constraints
                        # for why sizing and funds-availability must not
                        # share the same value.
```

`inf mode` (`initial_cash == 0`): `check_funds_available()` is not called —
there is no remaining_cash to exhaust.

The actual call sequence (`compute_position_size()` →
`check_funds_available()` → `simulate_entry_fill()`), including which price
each step uses, is written out once in full in "Entry Slippage Model"
below rather than duplicated here — sizing and entry-fill simulation are
one continuous flow, not two independent steps.

---

## Entry Slippage Model

Simulate partial entry fills from the anchor, t bar open +
`entry_fill_delay_seconds`, over synthetic ticks expanded from tick_10 —
mirrors the exit side's `simulate_exit_fill()` rather than assuming a
single instant fill.

```
tick_10.hour = last tick timestamp of each 10-tick bundle (second
               precision); a second s denotes [s, s+1) (utils.md's
               second-stamp semantics)
synthetic    = utils.expand_synthetic_ticks(ticks_td, halts_td)'s ticks;
               a synthetic tick's second is the integer part of its time
anchor       = utils.hour_add_seconds(
    entry_hour, config["execution"]["entry_fill_delay_seconds"])
window_end   = utils.hour_add_seconds(
    anchor, config["execution"]["cancel_after_seconds"])

Entry tick range: synthetic ticks whose second is greater than anchor and
at most window_end. No tick in the range → zero fill → entry_canceled;
there is no p_entry fallback.

Procedure:
    # Sizing uses p_entry, not a post-slippage price — see execution_common.md's
    # Constraints for why this ordering is required (simulate_entry_fill()
    # takes quantity as input, so quantity cannot wait for its own output).
    quantity = execution_common.compute_position_size(
        balance=initial_cash, fill_price=p_entry, mgnrt=100,
        # mgnrt=100 is the CONSTANT, not a config key: backtest has no
        # broker to publish a rate and no margin to model. At 100 the
        # margin-unit form reduces to the previous notional formula exactly
        # (execution_common.md), which is why no backtest-side branch or
        # successor to the deleted sizing_basis is needed.
        t_bar_volume=t_bar_volume,
        ticker_margin_used=..., total_margin_used=...,
        position_size_cash_pct=config["execution"]["position_size_cash_pct"],
        position_size_vol_pct=config["execution"]["position_size_vol_pct"],
        per_ticker_share_cap_pct=config["execution"]["per_ticker_share_cap_pct"],
        exposure_cap_pct=config["execution"]["exposure_cap_pct"],
    )
    order_type = execution_common.permitted_order_type(
        "buy", config["execution"]["entry_order_type"],
        phase at the anchor)              # execution_common.md's Session Phase
    limit_price = None if order_type == "market" \
        else the Entry Limit Price rule  # execution_common.md: the
    # entry_gap-derived limit rounded DOWN to tick(price), capped at P_cap
    # when execution.entry_price_cap_enabled — S from this candidate's
    # prediction row. A configured market entry forced to limit takes it too.
    funds_price = p_entry if order_type == "market" else limit_price
    proceed, quantity = execution_common.check_funds_available(
        quantity, funds_price, remaining_cash,
        use_all_cash=config["execution"]["use_all_cash"],
    )
    if not proceed:
        → skip entry (insufficient funds even sized down, or use_all_cash=False
          and full quantity unaffordable)
    # Cash is NOT deducted here. It is deducted once, after the fill below
    # resolves, as total_filled * weighted_avg_fill_price — the only amount
    # actually committed. The former provisional quantity * p_entry
    # deduction over-charged every partially-filled entry and permanently
    # consumed capital on a fully-unfilled one (entry_canceled), with no
    # reconciliation step anywhere in the spec. Safe to defer because the
    # chronological simulation carries each candidate through its own fill
    # before advancing to the next event.

    rate = execution_common.entry_fill_rate(config, order_type)
    if config["backtest"]["entry_fill_rate_disabled"]:
        rate = 1        # all volume after the anchor; the switch stays a
                        # config and experiment_log state, not a rate

    # ONE TIMELINE: fills and the tp/sl breach test advance together, tick by
    # tick, over the entry tick range in time order.
    state, breach = None, None
    for tick in the entry tick range:
        if fills are open:        # closed after a phase boundary where
                                  # order_type is not permitted — the entry
                                  # remainder ends there (ended early)
            state = execution_common.simulate_entry_fill(
                state, ticks=[tick], quantity=quantity,
                entry_anchor_second=anchor, limit_price=limit_price,
                last_print=last print strictly before the anchor,
                rate=rate, halts_df=halts_td,
            )                     # step form, one span: the order is unchanged
        if state has filled:      # from the first fill on
            w = state's weighted-average fill price after this tick's step
            # tp_pct and execution.stop_loss_pct as in Exit Logic
            if tick.price >= w * (1 + tp_pct) or tick.price <= w * (1 - stop_loss_pct):
                breach = (direction, tick)
                break             # the entry stops, remainder canceled; the
                                  # exit starts on the quantity filled so far

    fill_price, total_filled, unfilled_qty, status = the summary of state
        # execution_common.md: weighted average, p_entry when nothing filled
    if total_filled == 0:
        # Fully unfilled — logged as its own trade_log row rather than
        # silently dropped (see db_schema.md's exit_reason='entry_canceled'):
        # trade_log row: exit_reason="entry_canceled", quantity=0,
        #                fill_price=p_entry, exit_bar=entry_bar,
        #                pnl_pct=NULL — skip remaining Exit Logic entirely
        #                for this candidate, proceed to next candidate
    else:
        # total_filled may be < quantity (order canceled at the window's end
        # or stopped by a breach) — trade proceeds sized down to
        # total_filled, no penalty (see execution_common.md Constraints:
        # entry-side unfilled remainder is canceled, not penalized, unlike
        # exit-side).
        quantity = total_filled
        # breach found → Exit Logic's take-profit / stop-loss exit at that
        # tick; none → Exit Logic's continuation after the window

slippage_pct = (fill_price - p_entry) / p_entry
```

---

## Entry Decision Logic

```python
from execution_common import resolve_signal

threshold          = config["backtest"]["entry_threshold"]
suppress_threshold = config["backtest"]["suppress_threshold"]  # None = disabled
signal = resolve_signal(row, threshold, suppress_threshold)
# → "up5" | "up3" | None
```

Entry executed only if signal is not None AND cooldown check passes.

**Suppression behavior:**
- If `prob_dn5 >= suppress_threshold` or `prob_dn3 >= suppress_threshold`,
  signal is None regardless of upside probabilities.
- `suppress_threshold: null` disables suppression.
- Suppressed entries are not logged to trade_log.

### Cooldown guard

`can_enter()` moved to `docs/utils/execution_common.md` — now shared with
LiveModeRunner (previously backtest-only by virtue of only being defined
here). See execution_common.md for the full spec and the
session_mode="combined" cross-boundary behavior.

### Chronological Simulation

One pass per DATE. Positions never span dates in normal flow (see
data_boundary.md — a dead position is resolved against D+1 data but holds
no slot into D+1; see Dead Position), so a date is a complete unit and
per-date processing bounds how much tick data must be resident at once.

```
for each date, in ascending order:
  merge into one time-ordered stream:
     - entry candidates for this date (all tickers)
     - scheduled exit events from positions opened earlier this date
     - revive credits scheduled by dead positions on earlier dates
  same-bar ties: predictions DataFrame order (ticker alphabetical tiebreak)
  reset last_entry_hour to None for every ticker (see Cooldown guard)
  active_positions = {}

  for event in the stream:
    exit event    -> remove from active_positions
                     remaining_cash += realized proceeds
    revive credit -> remaining_cash += the scheduled amount
    entry cand.   -> LATE-DETECTION MODEL first (see below), then gates,
                     in LiveModeRunner's step-5c.0 order:
                       1. distinct tickers in active_positions <
                          execution.max_tickers (skipped when this ticker
                          is already held)
                       2. this ticker's own count <
                          execution.max_positions_per_ticker
                       3. can_enter(...)   [cooldown]
                       4. compute_position_size(...)  [current margin used]
                       5. check_funds_available(...)  [current cash]
                     admitted -> resolve the trade (Entry Slippage Model +
                     Exit Logic), deduct the committed cash, add to
                     active_positions, and schedule its exit event at the
                     resolved exit_hour
                     blocked at any gate -> increment that gate's own
                     counter (gate_blocked_cap_tickers,
                     gate_blocked_cap_per_ticker, gate_blocked_cooldown,
                     gate_blocked_sizing_zero, gate_blocked_funds); no
                     trade_log row, since nothing was attempted. Gates 1
                     and 2 cannot both fire for one candidate — gate 1 only
                     applies when the ticker is not already held, gate 2
                     only when it is — so the counters never double-count
                     the same candidate

  Circuit breaker (R-4): after each exit event closes a position (same
  point live updates its own counters — see live_mode_runner.md's Circuit
  Breaker), update breaker_max_consecutive_losses and
  breaker_max_realized_loss_abs/_pct from that closed trade, and
  breaker_peak_entries_per_hour from the rolling window on each admitted
  entry. COMPUTED for every run, exactly as live computes them, but NEVER
  ENFORCED — no candidate is ever blocked by these values, regardless of
  what execution.intraday_loss_limit_pct / consecutive_loss_limit /
  entries_per_hour_limit are set to. The three thresholds live in the
  shared execution: block because backtest reads them as the SAME
  measurement basis live uses (so the two are comparable), not because
  backtest gates on them. Same policy as R-4's shadow-mode treatment, for
  the same reason: backtest is evaluating a known-good model against
  history, so it has no failure to catch, and a real trip would truncate
  exactly the data a run exists to score — see live_mode_runner.md's
  Circuit Breaker for the fuller argument, which applies here unchanged.
```

**Late-detection model.** Backtest takes every detected entry point; live
takes only what its watchdog scan OBSERVED. The two differ, and the
difference is correlated with bursts — the moments where opportunity
clusters — so leaving it unmodelled makes backtest most optimistic exactly
where it matters most. This applies REGARDLESS of whether late entry is
adopted; the flag below only decides what happens to the affected
candidates, not whether they are affected.

```
h = deterministic_hash(ticker, date, hour) -> [0, 1)

h >= backtest.late_detection_rate:
    head-detected. Unchanged.

h <  backtest.late_detection_rate:
    rotation-detected.
    if this ticker's IMMEDIATELY PRECEDING bar was dropped by this rule:
        EXEMPT — take the candidate normally. This is promotion:
        live promotes a ticker after a rotation-slot finding, so its next
        bar close is guaranteed to be seen.
    elif not execution.late_entry_enabled:
        DROP the candidate. No trade_log row and no gate counter — nothing
        was attempted; live never saw it.
    else:
        enter at t_open + delay, delay = h2 * backtest
        .late_detection_delay_max_s, h2 a second hash of the same key.
        Uniform delay follows from the rotation cursor's arbitrary phase;
        live_scan_daily can falsify it.
```

A HASH, not an RNG. No seed to manage, invariant to iteration order in this
loop, and — because it never reads the live database — a run today and the
same run next month produce identical results. Reading
`late_detection_rate` from accumulated live data AT RUN TIME would silently
break that, which is why it is a config value seeded from measurement
rather than a query.

The exemption makes this loop state-dependent (one bit per ticker), which is
the cost of representing promotion at all. It is also what makes promotion's
real value — how often a detection persists into the adjacent bar — directly
MEASURABLE in a backtest rather than assumed.

Defaults `late_detection_rate: 0.0` and `late_detection_delay_max_s: 0.0`
make the whole model a no-op, so current backtest output is preserved
byte-for-byte until a measured rate exists to seed it with.

**The parity gap runs the OTHER WAY most of the time, and the direction is
worth stating.** `entry_fill_delay_seconds` is sized against live's WORST
case — a bar close where the candidate superset is full, K orders queue
behind each other at the order endpoint's rate, and the sequence only just
fits. Ordinary bar closes carry far fewer candidates, so live fetches, infers
and submits well inside that budget and fills EARLIER than this model does.
Backtest is therefore CONSERVATIVE in the common case and converges to live
only at burst. It is not a bound in either direction: if measured inference
time turns out longer than assumed, the key moves up and the relationship
inverts — which is why it is a key rather than a constant.

A revive credit whose date is never processed (no candidates that day)
applies at the start of the next processed date — no capital is lost, and
nothing could have consumed it in the interval anyway.

`inf mode` note: with `initial_cash == 0` the cash leg and
`check_funds_available()` are both skipped, so sizing is `vol_based` only.
The caps then bound the NUMBER of concurrent positions but not their SIZE.
`total_pnl_abs` is therefore only meaningful when `initial_cash > 0`; use
`avg_pnl_pct` to compare configurations.

**Interaction with `execution.use_all_cash`** (see execution_common.md's
`check_funds_available()`): with the default `use_all_cash: true`,
"exhausted" no longer means the next candidate in line is skipped outright
— the first candidate that cannot be filled at its full requested quantity
is instead sized down to whatever `remaining_cash` allows (possibly a very
small quantity), and *that* trade proceeds. Only a candidate for which even
1 share is unaffordable is skipped. This changes which candidates end up
in `trade_log` for a cash-constrained run compared to the old all-or-nothing
behavior — set `execution.use_all_cash: false` to restore the previous
skip-on-shortfall behavior exactly.

---

## Exit Logic

### Take-profit / Stop-loss exit

```python
tp_pct = config["execution"]["take_profit_up5"] if signal == "up5" \
         else config["execution"]["take_profit_up3"]

# 1. A breach inside the entry window is the Entry Slippage Model's: tested
#    on synthetic ticks against the running weighted average.
# 2. Continuation, when the window found none: synthetic ticks after
#    window_end are tested against the FIXED weighted average fill_price,
#    through the effective end time (expand_synthetic_ticks()'s
#    effective_end) of B, the first bundle whose hour >= window_end.
# 3. Then track_price_breach() continues from B, B itself excluded:
direction, exit_price, exit_hour, is_ambiguous = utils.track_price_breach(
    ohlcv_future=bars_from_t,
    ticks_future=ticks_td[ticks_td["hour"] >= entry_hour],
    fill_price=fill_price,              # the fixed weighted average
    fill_second=B.hour,                 # window_end when no B exists that date
    threshold_up=tp_pct,
    threshold_dn=config["execution"]["stop_loss_pct"],
    exit_interpolation=config["execution"]["exit_interpolation"],
)
# A breach on a synthetic tick (steps 1–2) gives direction, exit_hour = that
# tick's second and exit_price = its price; is_ambiguous = False.

if direction is not None:
    # P-10 REVISED (R-2): the former poll-delay alignment
    # (ceil_to_interval to the live position_check_interval_seconds grid)
    # modeled the OLD periodic-poll live loop. Live now detects tp/sl at
    # TICK granularity (WS-primary / REST-backstop — see
    # live_mode_runner.md's Exit Architecture), so there is no poll delay
    # left to model: backtest uses the breach's raw exit_hour/exit_price —
    # track_price_breach()'s (already interpolate_bundle_price-precision),
    # or the breaching synthetic tick's — directly for the fill simulation
    # and for trade_log.exit_bar. The
    # relationship this corrects, STATED AGAINST LIVE'S REST 1-TICK BUFFER
    # and not against live generally: that buffer is vendor-guaranteed
    # complete AND print-ordered, so it is ground truth at tick granularity
    # and backtest approximates it — not the reverse, as the removed
    # poll-delay alignment implied. Live's exit TRIGGER path is WS-primary,
    # and the WS entitlement carries roughly half the prints, so the trigger
    # path is NOT the thing being ranked here; against it and tick_10
    # neither dominates (see Constraints' R-2 kinds (1) and (2)).
    effective_second = exit_hour        # raw, no ceil_to_interval
    effective_price  = exit_price       # raw interpolate_bundle_price output,
                                        # no re-interpolation at a delayed second

    exit_reason = "take_profit" if direction == "up" else "stop_loss"

    weighted_avg_exit_price, total_filled, unfilled_qty = simulate_exit(
        anchor=effective_second, reference_price=effective_price,
        quantity=quantity, exit_reason=exit_reason,
    )                                   # Exit fill simulation, below
    # ohlcv_exit is gone — it had no reader, and the slice passed here was
    # filtered on `hour` alone across a MULTI-DATE frame while the exit ticks were
    # correctly date-scoped, which is itself evidence nothing read it.

    if total_filled > 0 and unfilled_qty > 0:
        # PARTIAL fill. Not a dead position — all available ticks (including
        # after-market) were used. Unfilled portion penalized at flat
        # dead_position_penalty_pct.
        # total_filled > 0 is part of the test, not decoration: on its own
        # `unfilled_qty > 0` calls a TOTAL non-fill a partial fill, which
        # would put a row at unfilled_quantity/quantity == 1.0 in the same
        # category as one at 0.5.
        pnl_filled   = (weighted_avg_exit_price - fill_price) / fill_price
        pnl_unfilled = -config["backtest"]["dead_position_penalty_pct"]
        pnl = (pnl_filled * total_filled + pnl_unfilled * unfilled_qty) / quantity
    elif total_filled == 0:
        # ZERO fill, named separately. Still NOT a dead position: the
        # dead-position condition is absence of DATA, and this path was
        # entered with ticks available, so liquidity was absent rather than
        # the price being unresolvable. exit_price falls back to the
        # reference price and the flat penalty applies to the whole
        # quantity — deliberately not the same as dead-position Cases B/D's
        # pnl = -1.0, because a reference price was actually OBSERVED here
        # and the position can be expected to leave near it, whereas B/D
        # have no resolvable price at all.
        pnl = -config["backtest"]["dead_position_penalty_pct"]
    else:
        pnl = (weighted_avg_exit_price - fill_price) / fill_price

else:
    → proceed to session_close / time_limit check
```

`ceil_to_interval(hour, interval_seconds, align_to)`: rounds `hour` up to
the next `align_to`-aligned boundary at `interval_seconds` spacing — e.g.
`align_to="093000"`, `interval_seconds=5`, `hour="093512"` → `"093515"`
(next 5-second mark counting from 09:30:00, not from the input hour
itself). New small utility; belongs in `utils.py` alongside the other hour
utilities (not execution_common.md — it's a pure time-arithmetic function
with no execution-decision content, same category as `hour_add_seconds()`).

**`trade_log.exit_bar`**: the EXIT ANCHOR INSTANT, for every exit
reason. On the tp/sl path that is `effective_second` above rather than the
raw `exit_hour`. On the `time_limit` / `session_end` paths it is the instant
the trigger fired — previously undefined here, those two reasons having had
no `effective_second` to populate it from, which the generalised anchor
closes (execution_common.md). This file's exact trade_log-row-assembly code is outside this
section's scope to fresh-quote here; flagging so it isn't missed as a
separate small follow-up when this section is actually applied.

**sell_rate rationale:**
- take_profit exit (rising market): more buyers available → higher sell_rate
- stop_loss exit (falling market): fewer buyers available → lower sell_rate

### Session close exit (priority over time-limit)

```python
if current bar hour == exit_deadline(date):     # utils.md — PER DATE
    exit_reason = "session_end"
    # DEAD-POSITION RESOLUTION FIRST, ahead of any fill modelling:
    if exit_deadline(date) bar is halt/no_data:
        fallback: first tick_10 row with hour > exit_deadline(date)
        if none: → dead position          # no ticks at all → no simulator call
    mode, _ = execution_common.exit_fill_mode(
        config, "session_end", the exit's permitted order type)
    if mode == "reference":   # simulate_nondirectional_exit_fill false
        exit_price = the reference price (that bar's close, or the
                     after-market tick fallback above)
    else:
        anchor    = exit_deadline(date)
        reference = last print strictly before the anchor
        → simulate_exit(anchor, reference, quantity, "session_end")
```

The dead-position test stays AHEAD of the simulator because it is a
DATA-ABSENCE condition, not a failure to fill. Keeping it there preserves
both it and the exhaustion-is-not-a-dead-position rule below: the simulator
is only entered once at least one tick exists at or after the anchor, so a
zero fill there means liquidity was absent, which is a different fact.

### Time-limit exit

```python
if config["execution"]["max_hold_bars"] valid bars elapsed since entry
   (via build_effective_bar_sequence, which is passed THAT SAME key as its
    target_valid_bars — the parameter carries no default, because a default of
    60 would cap the sequence below the count this line tests and make the
    exit unreachable whenever the key exceeds 60):
    exit_reason = "time_limit"
    mode, _ = execution_common.exit_fill_mode(
        config, "time_limit", the exit's permitted order type)
    if mode == "reference":   # simulate_nondirectional_exit_fill false
        exit_price = close of last valid bar
    else:
        anchor    = the instant that count was reached
        reference = last print strictly before the anchor
        → simulate_exit(anchor, reference, quantity, "time_limit")
```

Note that once an exit reason is assigned, filling may continue past the
session close into the after-market — the simulator's own contract. "Session close takes
priority over time-limit" governs which reason is ASSIGNED, not whether an
in-progress fill is re-triggered.

### Exit fill simulation

```
simulate_exit(anchor, reference_price, quantity, exit_reason):
    order_type = execution_common.permitted_order_type(
        "sell", config["execution"]["exit_order_type"], phase at the anchor)
    mode, rate = execution_common.exit_fill_mode(config, exit_reason, order_type)
    if config["backtest"]["exit_fill_rate_disabled"]:
        rate = 1                # all volume after the anchor; a switch, not a rate
    ticks = synthetic ticks whose second is greater than the anchor, through
            the full day

    Spans — split where the order path changes, each one
    execution_common.simulate_exit_fill() call (batch form) taking the prior
    state, the span's order and the last print before the span's start:
      - market: one span from the anchor. At the session_close boundary a
        market remainder is not permitted; it continues as a limit ladder
        from that instant, k starting at exit_ladder_seed on the anchor's
        clock.
      - limit — configured, or switched at that boundary: re-quoted from the
        anchor every live_mode.position_check_interval_seconds (the
        re-quote cadence):
            L = m * (1 + s * (1/2 - k)), rounded UP to tick(price)
            m = the last synthetic tick's price before the re-quote instant
            s = execution_common.md's Exit Spread Estimator, from bundles
                sealed (effective end time) and bars completed before that
                instant
            k starts at execution.exit_ladder_seed; once the elapsed time
            since the anchor reaches execution.exit_order_stuck_minutes:
              - inside the regular session (utils.md's Session Boundary
                Derivations): the remainder escalates to market pricing;
              - outside it: k += execution.exit_ladder_increment per
                re-quote, up to execution.exit_ladder_cap.

    → weighted_avg_exit_price (reference_price when nothing filled),
      total_filled, unfilled_qty = quantity - total_filled
    partial_fills_count = the number of distinct integer seconds carrying
      an exit fill
```

---

## Ambiguity

`is_ambiguous = True` when tp_target and sl_target are simultaneously satisfied
within the same 10-tick bundle during `track_price_breach()` scan.

Replaces prior definition (1-minute bar level simultaneous breach).
A breach found on synthetic ticks — inside the entry window or its
continuation (Exit Logic) — is never ambiguous: each synthetic tick carries
one price.
Recorded in `trade_log.is_ambiguous` for post-hoc analysis.

---

## Dead Position

Occurs when:
1. Session_end exit price cannot be determined (exit_deadline(date) halt + no
   after-market data).

```
Case A — next trading day has_data=True AND ticker exists in ticker_data_coverage:
    exit_price = next day pre-market first tick
                 fallback: next day ohlcv_1min first bar open
    if exit_price cannot be resolved from either source (NaN/unavailable):
        → Case D (below)
    exit_price *= (1 - dead_position_penalty_pct)
    r = cum_split_ratio() over (D, D+1]   (utils.md's split factor; 1.0 if none)
    dividend_term = Σ over 'dividend' rows with event_date IN (D, D+1] of
        the net amount (utils.md's dividend amount function: the gross
        amount in its event_date basis × (1 − corporate_events.
        dividend_withholding_rate)) × the split factor over (D, that
        event_date]   (0.0 if none) — per share entered; same overnight
        window and rationale as Labeler's Case A (see 05_labeler.md); US
        splits and ex-dividend adjustments always take effect before
        market open
    revive amount = quantity × (r × exit_price + dividend_term)
        — quantity in D's basis
    pnl = (revive amount − quantity × fill_price) / (quantity × fill_price)
    exit_reason = "dead_position"
    is_dead_position = True
    trade_log row, in the basis as of its exit:
        fill_price = the simulated fill ÷ r
        exit_price = as resolved above
        quantity, requested_quantity = × r, rounded per utils.md's Numeric
            Rules (the product converted per trading_api.md's V1)
        slippage_pct = (simulated fill − p_entry) / p_entry, taken before
            the normalisation
    exit_bar / exit_date = the timestamp of whichever source actually
        resolved exit_price (pre-market first tick, or the first bar open
        on fallback) — NOT a fixed hour. This is the only Case whose
        exit_date is D+1.

Case B — next trading day has_data=True AND ticker NOT in ticker_data_coverage:
    pnl = -1.0  (full loss — possible delisting)
    exit_price = 0
    exit_reason = "dead_position_delisted"
    is_dead_position = True
    exit_bar = last_bar(D), exit_date = D

Case C — next trading day not in dataset (dataset boundary):
    exit_price = p_entry * (1 - config["backtest"]["dead_position_boundary_pct"])
                 # R-7: was a bare 0.5 literal while the sibling
                 # dead_position_penalty_pct was already config — same kind of
                 # value, two different treatments
    exit_reason = "dead_position_no_data"
    is_dead_position = True
    exit_bar = last_bar(D), exit_date = D — the synthetic exit_price needs no
        D+1 data, so it is already known at D's close

Case D — Case A path entered, but exit_price unresolvable after fallback
         (pre-market first tick AND next-day first bar open both unavailable
         — extended/multi-day halt):
    pnl = -1.0  (full loss — capital locked with no resolvable exit price)
    exit_price = 0
    exit_reason = "dead_position_extended_halt"
    is_dead_position = True
    exit_bar = last_bar(D), exit_date = D
    (mirrors Labeler's Case D — see 05_labeler.md — same mechanism as Case B)
```

**Early-close dates are ALWAYS replayed.** `live_mode.trade_early_close_days`
gates the day's entry pipeline in live — carried positions being managed by the
reduced session (live_mode_runner.md) — and is not read here — a known one-sided
divergence between the engines, in the same family as R-4's breaker being
computed but never enforced and shadow mode's evaluate-but-don't-enforce. It is
deliberate: those days are labelled correctly and kept in the corpus precisely
so backtest can measure whether they are worth trading, and a gate that switched
off both engines would make that measurement impossible. Nothing is recorded in
`experiment_log` for this, because nothing varies per run — see db_schema.md.

**Cases B/C/D's `exit_bar` is DERIVED, not the literal 155900 it once was.**
These three write a SYNTHETIC anchor — no liquidation happened — but
db_schema.md defines `exit_bar` as the exit anchor instant and `exit_date` as
the date it belongs to, and all three carry `exit_date = D`. On an early-close D
the literal named an instant that does not exist, so the pair (date, exit_bar)
pointed nowhere. `last_bar(D)`, NOT `exit_deadline(D)`: the anchor marks the end
of D's tape, which is a data fact, and it must not drift if someone later sets
the exit offset to ten minutes. Case A is unaffected — it already carries a real
D+1 resolution timestamp.

**Capital treatment.** At D's close the position releases its slot (it no
longer counts toward `execution.max_tickers` /
`execution.max_positions_per_ticker`) but returns NO cash — the whole
committed amount is written off at that moment. Cash comes back only as a
scheduled revive credit, under one rule with no per-Case special casing:

    revive amount = quantity * (r * exit_price + dividend_term)
                    — quantity in D's basis; r = 1 and dividend_term = 0
                    outside Case A, which reduces it to quantity * exit_price
    revive moment = that row's own exit_date / exit_bar
    The cash ledger's arithmetic runs in float — Case A's amounts are
    computed values (utils.md's Numeric Rules).

which resolves per Case without further rules: A revives at its D+1
resolution timestamp; C revives immediately at D's close, since its
synthetic price needs no D+1 data; B and D have `exit_price = 0`, so the
formula yields zero and the write-off is simply the result rather than a
special case. This is what keeps `total_pnl_abs` reconcilable against the
cash ledger in every Case — crediting cash back at D's close while
recording pnl = -1.0 (B/D) or -0.5 (C) would break that identity.

Only the credit is deferred across the date boundary, never the position:
the trade row is written complete at D's close (BacktestEngine already
reads D+1 data to resolve Case A), and the queue carries just
`(exit_date, exit_bar, revive amount)`. No slot is held and no exit
evaluation is pending, so the per-date decomposition of the Chronological
Simulation holds.

Dead position trades are included in the winning_rate denominator.

Note: `simulate_exit_fill()` exhausting all available ticks with remaining unfilled
quantity is **not** a dead position — the position is partially closed and does not
require an overnight hold. Handled with a blended PnL and a flat penalty on the
unfilled portion, for EVERY exit reason that reaches the simulator rather than the
tp/sl path alone, since `time_limit` and `session_end` now reach it too when
`execution.simulate_nondirectional_exit_fill` is on. Diagnostic via
`trade_log.unfilled_quantity / quantity`.

---

## Config Keys (pipeline_config.yaml)

```yaml
backtest:
  initial_cash: 0                    # 0 = unlimited; fixed `balance` for
                                      # execution_common.compute_position_size()
  entry_threshold:    0.5
  suppress_threshold: 0.5            # null = suppression disabled
  dead_position_boundary_pct: 0.5    # Case C write-down (dataset boundary)
  dead_position_penalty_pct: 0.05    # exit-side unfilled-quantity penalty only —
                                      # entry-side unfilled remainder is sized
                                      # down, not penalized (see Entry Slippage Model)
  entry_fill_rate_disabled:  false   # counterfactual analysis switches. When on,
  exit_fill_rate_disabled:   false   # BacktestEngine passes rate 1 for that side
                                      # to the fill simulator in place of the
                                      # selected rate — participation in all
                                      # traded volume after the anchor — everything
                                      # else intact: ticks are still walked
                                      # forward from the anchor, halt intervals
                                      # still exclude ticks, tick exhaustion still
                                      # leaves an unfilled remainder, and the
                                      # price is still
                                      # the volume-weighted path. Keeping the time
                                      # axis and price path is the point; a variant
                                      # that filled instantly at the reference price
                                      # would discard exactly what makes the result
                                      # readable (how long a liquidation took still
                                      # shows).
                                      # Under backtest:, NOT execution:, because
                                      # unlike simulate_nondirectional_exit_fill this
                                      # has no real-market counterpart — a venue
                                      # cannot be told to fill everything. Live must
                                      # not read it, and shadow must not either,
                                      # shadow existing to reproduce live.
                                      # SWITCHES, not `rate: 1.0`:
                                      # run_backtest.py's get_execution_param()
                                      # resolution overlays fitted values on the
                                      # configured rates (execution_common.md's
                                      # Rate Overlay), so a configured 1.0 can be
                                      # replaced by a fitted value. A switch
                                      # applies after selection
                                      # (entry_fill_rate() / exit_fill_mode()), in
                                      # BacktestEngine only.
                                      # The two also differ in kind — 1.0 asserts
                                      # full participation is realistic; the switch
                                      # asserts no participation model is in use,
                                      # which is neither a fitting target nor a
                                      # bound's subject.
                                      # Independently switchable, and the asymmetry
                                      # that allows is the use: disabling only the
                                      # exit side asks whether an edge survives
                                      # without exit-side liquidity while entry
                                      # sizing stays realistic.
                                      # Recorded per run in experiment_log's
                                      # entry_fill_rate_disabled /
                                      # exit_fill_rate_disabled columns
                                      # (db_schema.md), never as resolved rates.
  # entry_fill_delay_seconds is NOT declared here — it lives under execution:,
  # because live reads the same quantity (execution_common.md). It was a
  # hardcoded 5 at two sites in this file, the standard path and the
  # tick-bundle block, so changing one and not the other would have split
  # them silently.
  late_detection_rate:        0.0    # share of entry points live's watchdog
                                      # scan reaches only via the rotation
                                      # cursor. 0.0 = model disabled, current
                                      # output preserved exactly. Seeded from
                                      # live_scan_daily's measured rotation
                                      # figures, never read from the DB at run
                                      # time — see Chronological Simulation
  late_detection_delay_max_s: 0.0    # upper bound of the uniform delay applied
                                      # to a rotation-detected candidate when
                                      # execution.late_entry_enabled is true.
                                      # Ignored when it is false, since those
                                      # candidates are dropped rather than
                                      # delayed
# position_size_cash_pct, position_size_vol_pct, take_profit_up3/up5,
# stop_loss_pct, exit_interpolation, sell_rate_tp/sl, use_all_cash moved to
# the shared `execution:` section (see docs/utils/execution_common.md) —
# backtest and live read the same values rather than carrying independently
# -configurable copies of numbers that must stay identical between them.
```

---

## trade_log Schema Additions

```sql
-- Added columns (in addition to existing schema):
weighted_avg_exit_price  DOUBLE,    -- volume-weighted average fill price across partial fills
partial_fills_count      INTEGER,   -- number of distinct integer seconds carrying an
                                    -- exit fill (synthetic tick times by integer
                                    -- part) — live's exit_fill_seconds definition
unfilled_quantity        DECIMAL(18,6), -- shares remaining after ticks exhausted (0 = fully closed)
is_ambiguous             BOOLEAN,   -- True if simultaneous bundle-level tp/sl breach
exit_date                VARCHAR,   -- 'YYYYMMDD'; equals `date` for every exit resolved
                                    -- on the entry date. Differs for dead position
                                    -- Case A (D+1) and for live rows closed in a later
                                    -- session — the list is db_schema.md's. NOT NULL: an always-populated column
                                    -- keeps every date-scoped query free of COALESCE.
requested_quantity       DECIMAL(18,6), -- quantity actually SUBMITTED, i.e. after
                                    -- check_funds_available() sized it down. It
                                    -- decides whether an order filled in full,
                                    -- which ends fit_execution_params()'s
                                    -- eligible-volume window (shadow_retraining.md);
                                    -- a funds-driven reduction is kept out of it
                                    -- for that reason — same reason
                                    -- entry_rejected is excluded from fit_execution_params()
```

---

## Constraints

- `execution.intraday_loss_limit_pct` / `consecutive_loss_limit` /
  `entries_per_hour_limit` (R-4) are read here as a measurement basis only —
  BacktestEngine computes the same three metrics live does but never gates
  an entry on them (see Chronological Simulation's Circuit Breaker note)
- BacktestEngine accesses DuckDB directly — callers do not pass ohlcv/ticks as arguments
- Numeric decisions follow utils.md's Numeric Rules:
  - `track_price_breach()`'s breach tests are grid-value decisions;
  - `compute_position_size()` / `check_funds_available()` floors and
    comparisons are grid-value decisions;
  - BacktestEngine's `DOUBLE` inputs enter them by V1;
  - the fill simulators' output prices pass V1 before any sign decision, pnl
    or write
- The fill simulators take tick sequences in every engine, each tick's own
  volume entering the cumulative fill, min(order quantity, floor(rate ×
  cumulative tick volume after the anchor)):
  - here, synthetic ticks expanded from `tick_10` (utils.md's
    `expand_synthetic_ticks()`)
  - in live, the REST portion of the held-ticker tape (live_mode_runner.md)
- BacktestEngine reads rates from the config run_backtest.py resolved and
  overlaid (execution_common.md's `get_execution_param()`), selecting them
  through `entry_fill_rate()` / `exit_fill_mode()`; it does not resolve
  again
- `db_conn` injected via constructor for testability (mock DB in tests)
- OHLCV loaded once per ticker for all its entry dates
- Tick data loaded once per ticker/date (full day); filtered in memory by range
- `halts_td` passed explicitly to all internal methods — no internal DB queries after initial load
- Session close (exit_deadline(date) bar) triggers immediate exit — takes priority over time-limit
- `simulate_exit_fill()` is not called for `session_end` or `time_limit` exits
  while `execution.simulate_nondirectional_exit_fill` is false, its default —
  the bar close and the last valid bar's close are used directly, so current
  output is preserved BYTE-FOR-BYTE, the same guarantee `late_detection_rate:
  0.0` carries. Turning it on routes both through the simulator with the
  generalised anchor and `sell_rate_neutral`, which changes the exit price on
  the MAJORITY path from an instant, complete, frictionless fill at a bar
  close to a tick-level partial fill, and can produce unfilled remainders and
  their flat penalty where none existed — recorded here so a later session
  does not read that movement as a defect
- The key lives under `execution:`, not `backtest:`, because live shadow reads
  the same value: it is a SEMANTICS SELECTOR both engines share, and a
  backtest-only switch would make them diverge exactly when it is off, which
  is the divergence it exists to remove. With it off, live writes the
  reference price straight to `exit_price` rather than calling the simulator,
  so on and off branch over the SAME anchor
- After-market data used only as fallback when exit_deadline(date) bar is halt/no_data
- Dead position: session_end fallback fails only (exit_deadline(date) halt + no after-market data)
- Dead position lookup uses `has_data = TRUE` filter (not `is_trading_day`) — consistent with Labeler
- Dead position Case A pnl uses the split-normalised exit and the net dividend term
  (see Dead Position section) — `p_entry` is never adjusted;
  `corporate_events` rows for (ticker, event_date IN (D, D+1]) are loaded alongside
  `trading_calendar`/`ticker_data_coverage` for this lookup; same query pattern as Labeler
- Dead position Case D (`exit_reason = "dead_position_extended_halt"`) triggers only when
  Case A's exit_price cannot be resolved after fallback — pnl = -1.0, consistent with Case B
- `simulate_exit_fill()` ticks exhaustion is a partial fill — not a dead
  position — when `total_filled > 0 AND unfilled_qty > 0`. The `filled` half
  of that test is load-bearing: `unfilled_qty > 0` alone classifies a TOTAL
  non-fill as a partial fill and clears `is_dead_position` for it, leaving
  capital locked with nothing recording it. A zero fill is named separately
  and is still not a dead position, that condition being absence of data
  rather than failure to fill. `is_dead_position` remains False in both;
  diagnostic via `trade_log.unfilled_quantity`
- sell_rate_tp > sell_rate_sl: rising-market exits have more available buy-side depth
- Entry tick range: synthetic ticks whose second is greater than the anchor
  and at most anchor + cancel_after_seconds (not limited to the 1-minute t
  bar); no tick in the range → entry_canceled
- Entry fills and the tp/sl breach test run on one timeline: a breach inside
  the entry window stops the entry; after it, synthetic ticks are tested
  against the fixed weighted average through the effective end of the first
  bundle whose hour is at or after the window's last second, then
  `track_price_breach()` continues from that bundle, itself excluded
- Cooldown applied continuously across full time axis; no reset at session
  boundaries, but RESET at every date boundary — see execution_common.md's
  Cooldown Guard for why a carried-over `last_entry_hour` blocks the whole
  next day (hour_to_minutes() carries no date, so the delta goes negative)
- `entry_bar` stored as int via `utils.hour_to_int(entry_hour)`. `exit_bar` stored
  as int via `utils.hour_to_int(effective_second)`, which is now simply
  `exit_hour` itself (P-10 revised, R-2 — the poll-delay alignment was
  removed since live detects tp/sl at tick granularity; see Exit Logic).
  `trade_log.exit_bar` therefore reflects, on the tp/sl path,
  `track_price_breach()`'s raw,
  bundle-interpolated exit moment, or the breaching synthetic tick's
  second, directly. `exit_reason='entry_canceled'`
  rows (see Entry Slippage Model) set `exit_bar = entry_bar` — there is no
  distinct exit moment for a trade that never opened.
- `trades_by_signal` and `trades_by_exit` JSON-serialized before writing to `experiment_log`
- Dead position trades included in winning_rate denominator
- Never-opened rows (exit_reason 'entry_canceled'; live also 'entry_rejected')
  excluded from winning_rate / total_trades / avg_pnl_pct / total_pnl_abs —
  quantity=0 and pnl_pct=NULL mean no position existed. Still written to
  trade_log and counted in trades_by_exit; no separate summary column is
  added, since that JSON already carries the per-reason counts
- Processing is date-major and chronological, not ticker-major — the
  concurrency caps and the cash ordering are cross-ticker properties of a
  moment in time (see Chronological Simulation)
- Suppressed entries (suppress_threshold) not logged to trade_log
- `suppressed_count` tracked in summary for diagnostics
- `run()` returns `tuple[pd.DataFrame, dict]` — trade_log_df and summary_dict;
  DB writes (trade_log INSERT, experiment_log INSERT) performed by Backtester (run_backtest.py),
  not by BacktestEngine
- Sourced from `utils.py`:
  - `build_effective_bar_sequence()`
  - `track_price_breach()`
  - `expand_synthetic_ticks()`
  - `interpolate_bundle_price()`
  - `hour_to_int()`
  - `hour_to_minutes()`
  - `hour_add_seconds()`
- Sourced from `docs/utils/execution_common.md`'s module:
  - `resolve_signal()`
  - `can_enter()`
  - `simulate_exit_fill()`
  - `simulate_entry_fill()`
  - `compute_position_size()`
  - `check_funds_available()`
  - `permitted_order_type()`
  - `entry_fill_rate()`
  - `exit_fill_mode()`
- `is_tradable()` is deliberately NOT called here (N-5), unlike every other
  function in the bullet above. `ticker_cik_map`'s rename_pending /
  quarantine_reason (see db_schema.md) reflect data-pipeline/vendor-latency
  state as of right now, with no date-scoping — calling it here would check
  TODAY's suspension state against whatever historical date a given backtest
  run happens to be simulating, which is both non-deterministic (re-running
  the "same" backtest on a different day can change results) and, more
  fundamentally, the wrong question: both suspend reasons exist to hedge
  live, real-time uncertainty (is this really a rename? a real corporate
  event?) that a backtest already has resolved by the time anyone runs it.
  If a future change reintroduces an `is_tradable()` call here, re-read
  execution_common.md's Constraints (the matching note lives there) before
  assuming its absence was an oversight — it was not.
- `trading_halts` (used above for `halts_td`) is populated by the daily NYSE
  crawl (see metadata_crawler.md) and is training/backtest-only — LiveModeRunner
  never queries this table for real-time decisions; its live-mode halt signal
  is a separate tick-rate heuristic scoped to open positions (see
  live_mode_runner.md's Position Manager Loop)
- Backtest-vs-live exit asymmetry (R-2) — THREE distinct kinds, do not
  conflate. (1) ORDERING WITHIN A BUNDLE, irreducible — backtest reads
  10-tick `tick_10` bundles, and after the entry window and its
  continuation `track_price_breach()`'s Phase 1 tests
  `bundle.high >= tp_target or bundle.low <= sl_target`, so an intra-bundle
  extreme IS seen there; inside the window and its continuation the test
  runs on synthetic tick prices, which interpolate within a bundle and do
  not carry its high/low. What is lost is the ORDER of two extremes inside one
  bundle, which is exactly what `is_ambiguous` records and
  `ambiguity_priority` resolves. (Previously stated as backtest being unable
  to SEE intra-bundle spikes, and as live's WS STREAM being the "true" one —
  both wrong: the bundle carries the extreme, and the direction can invert,
  since a print in the half the WS entitlement drops is absent from live's
  WS path while still reflected in that bundle's high/low. The negation is
  scoped to the WS axis deliberately. Live's REST 1-tick tape IS complete,
  and against IT backtest does not invert — nothing is absent from a full
  tape — so calling live's stream untrue without qualification would deny
  the one live source that outranks `tick_10` on both axes.)
  (2) TAPE COMPLETENESS, open — the free real-time entitlement carries
  roughly half the trade prints while the REST tick endpoint is
  vendor-guaranteed complete, so live's WS path, live's REST backstop and
  backtest's `tick_10` do not run on the same tape. Registered in
  open_items.md as "WS/REST tape asymmetry and exit-trigger path
  transitions"; `exit_trigger_agreement_daily`'s L-vs-M term is what
  measures it.
  (3) The FORMER backstop-vs-WS FILTER asymmetry (a bundle-based backstop
  would have been less sensitive than WS's per-tick 2-print guard) does NOT
  apply — it never shipped; the live REST backstop polls individual ticks at
  WS granularity with the same 2-print guard from the start (see
  live_mode_runner.md's Exit Architecture). That premise stands; what does
  NOT follow from it, and used to be asserted here, is that only latency
  therefore differs between WS and its backstop — identical GRANULARITY and
  identical FILTER do not imply identical TAPE, which is (2). The
  corresponding sentence in live_mode_runner.md's Exit Architecture is
  deliberately left in place and marked false by the open item instead; it
  is corrected HERE because this file has no WS/REST branch for it to
  license, so removing it exposes nothing.
  (1) and (2) are both live open residuals. (1) is inherent to the
  historical dataset's granularity, not a design gap; (2) is a design gap
  and is blocked on shadow-period data.
