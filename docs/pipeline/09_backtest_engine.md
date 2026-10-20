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
            # never-opened family (db_schema.md's trade_log.exit_reason
            # values) have quantity=0 and
            # pnl_pct=NULL: no position ever existed, so counting them
            # would dilute winning_rate with non-events. They remain in
            # trade_log and stay visible via trades_by_exit.
            winning_rate        : float  — winning_trades / total_trades
            total_trades        : int    — opened trades only
            winning_trades      : int
            avg_pnl_pct         : float
            total_pnl_abs       : float
            avg_slippage_pct    : float
            dead_position_count : int    — trade_log rows with
                                           # is_dead_position TRUE, carried
                                           # rows whose remaining quantity
                                           # took Case B or carried Case C
                                           # pricing included
            dead_position_rate  : float  — the same numerator
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
            # analogue since backtest makes no network calls. Key names use
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
            negative_cash_count : int | None  — under
                                           # backtest.open_buy_cash_hold
                                           # 'none', the number of entry fill
                                           # increment debits after which
                                           # remaining_cash < 0; 0 under
                                           # 'order_amount'; None when
                                           # initial_cash == 0
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
  AND date IN (entry_and_carried_dates_for_this_ticker)
ORDER BY date, hour
```
Loaded once per ticker at the start of that ticker's processing.
Covers its entry dates and carried dates — not the full dataset.

### Tick data (per ticker/date — full day)
```sql
SELECT * FROM tick_10
WHERE ticker = ? AND date = ?
ORDER BY hour, seq_id
```
Loaded once per ticker/date combination — at each processed date's start,
for the tickers of its entry candidates and of its carried positions.
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

### Ticker data coverage (for dead position Case B and the carried-date state tests)
```sql
SELECT 1 FROM ticker_data_coverage
WHERE ticker = ? AND date = ?
```

### Corporate events (for dead position Case A/D and carried positions' dividend/split adjustment)
```sql
SELECT event_type, event_date, ratio, amount, source, basis_date FROM corporate_events
WHERE ticker = ? AND event_date > ? AND event_date <= ?
```
Queried only when Case A/D resolution is reached or a position is carried —
not on every trade:
- Case A/D: over (D, D+1], the same window and rationale as Labeler's
  equivalent lookup (see `05_labeler.md`);
- a carried position: over its carried interval, (the entry date, the
  completion date] (Carried position).
The split factor is `utils.md`'s single rule, applied through
`cum_split_ratio()`; a 'dividend' row's amount is read only through
`utils.md`'s dividend amount function, never from `amount` directly.

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

# Loaded once per ticker/date — entry and carried tickers of each
# processed date
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
                        # debited per backtest.open_buy_cash_hold (Entry
                        # Slippage Model) and credited back by the
                        # sell-proceeds credits, timed by
                        # backtest.sell_proceeds_settlement_days (see
                        # Chronological Simulation).
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
    last_print = the price of the last synthetic tick
        (utils.expand_synthetic_ticks()) whose time's integer part is less
        than the anchor; p_entry when no such tick exists
    if order_type == "market":
        # the MARKET-BUY LIMIT PRICE:
        s = execution_common.md's Exit Spread Estimator under
            execution.exit_spread_estimator, from bundles and bars sealed
            before the anchor
        market_buy_limit_price = last_print
            × (1 + s / 2)   when backtest.market_buy_spread_adjust is true
            × execution.market_buy_price_margin,
            rounded DOWN to tick(price)
    funds_price = market_buy_limit_price if order_type == "market" \
                  else limit_price
        # whatever the backtest.open_buy_cash_hold value; sizing above keeps
        # p_entry
    proceed, quantity = execution_common.check_funds_available(
        quantity, funds_price, remaining_cash,
        use_all_cash=config["execution"]["use_all_cash"],
    )
    if not proceed:
        → skip entry (insufficient funds even sized down, or use_all_cash=False
          and full quantity unaffordable)
    # Cash, per backtest.open_buy_cash_hold:
    # - 'none' (default): each entry fill increment's amount is debited from
    #   remaining_cash at its second; remaining_cash may go negative, and a
    #   negative value blocks the next candidate at check_funds_available();
    # - 'order_amount': at admission remaining_cash is debited the hold,
    #   requested quantity × the entry's order price (a limit entry's limit
    #   price, a market entry's market-BUY limit price); at entry end the
    #   hold minus Σ filled amounts is credited back
    #   backtest.open_buy_hold_release_seconds after entry end — not a
    #   sell-proceeds credit, so no backtest.sell_proceeds_settlement_days
    #   delay.

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
                entry_anchor_second=anchor,
                limit_price=market_buy_limit_price if order_type == "market"
                            else limit_price,
                    # a market entry: the entry limit rule applies at its
                    # market-BUY limit price, the rate staying
                    # entry_fill_rate(config, 'market'); the order type
                    # stays 'market' for every other purpose
                last_print=last_print,
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
        # execution_common.md: weighted average, p_entry when nothing filled;
        # the state's entry fill increments are the Chronological
        # Simulation's entry fill increment events
    entry_end_cause = 'filled' | 'window_end' | 'breach' | 'phase_boundary'
        # an entry with no tick in its range takes 'window_end'
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

One pass per PROCESSED DATE. A carried position spans dates (Carried
position, below); a dead position is resolved against D+1 data but holds no
slot into D+1 (see Dead Position). The tick data resident for a date is that
of its entry and carried tickers, so per-date processing still bounds how
much tick data must be resident at once.

```
processed dates, in ascending order — each date holding an entry candidate
or a carried position; after the run's last test date, the dates with DB
data while a carried position remains, with no entry candidates (Carried
position)

for each processed date:
  at the date's start, for the tickers of its carried positions: tick_10,
     trading_halts, ohlcv_1min and the ticker_data_coverage test (DB Access
     Strategy)
  merge into one time-ordered stream:
     - entry candidates for this date (all tickers), each at its entry
       anchor as the Entry Slippage Model computes it, a late-detection
       delayed entry included
     - entry fill increment and entry end events of admitted trades
     - exit events — exit fill increments and close completions — of this
       date's positions and of the positions carried into it
     - credits due: sell-proceeds credits (revive credits included) and
       hold releases
  same-instant ties: entry candidates before entry fill increment, entry
     end and exit events; candidates at one instant keep the predictions
     DataFrame order (ticker alphabetical tiebreak)
  reset last_entry_hour to None for every ticker (see Cooldown guard)
  active_positions = the positions carried from the previous processed date

  for event in the stream:
    entry fill increment -> under backtest.open_buy_cash_hold 'none',
                     remaining_cash -= its amount; the margin sums
                     (execution_common.md) update
    entry end     -> the pending remainder term drops; an entry_canceled
                     trade leaves the caps; under 'order_amount' its hold
                     release is scheduled (Entry Slippage Model)
    exit fill increment -> the breaker's realised loss accrues (below)
    close completion -> remove from active_positions; the close credit
                     (Carried position) is scheduled per
                     backtest.sell_proceeds_settlement_days; the
                     consecutive-loss counter updates
    credit        -> remaining_cash += the scheduled amount
    entry cand.   -> LATE-DETECTION MODEL first (see below), then gates,
                     in LiveModeRunner's step-5c.0 order, gates 1 and 2
                     counting only the positions dated the current date:
                       1. distinct tickers among them <
                          execution.max_tickers (skipped when this ticker
                          is already held among them)
                       2. this ticker's own count among them <
                          execution.max_positions_per_ticker
                       3. can_enter(...)   [cooldown]
                       4. compute_position_size(...)  [current margin used]
                       5. check_funds_available(...)  [current cash]
                     admitted -> resolve the trade (Entry Slippage Model +
                     Exit Logic), debit the hold under 'order_amount', add
                     to active_positions, and schedule its entry and exit
                     events
                     blocked at any gate -> increment that gate's own
                     counter (gate_blocked_cap_tickers,
                     gate_blocked_cap_per_ticker, gate_blocked_cooldown,
                     gate_blocked_sizing_zero, gate_blocked_funds); no
                     trade_log row, since nothing was attempted. Gates 1
                     and 2 cannot both fire for one candidate — gate 1 only
                     applies when the ticker is not already held, gate 2
                     only when it is — so the counters never double-count
                     the same candidate

  Pending: an admitted trade is pending from its admission to its entry
  end, the earliest of window_end, the ended-early instant and an
  entry-window breach tick. While pending it counts toward
  execution.max_tickers and execution.max_positions_per_ticker, a trade
  that ends entry_canceled included, and its margin follows
  execution_common.md's margin sums, updated at each entry fill increment.
  A carried position stays in active_positions for the margin sums,
  remaining_cash and its exit handling.

  Sell-proceeds credits — the close credit, a carried position's per-date
  credits included, and the dead-position revive credit: with
  backtest.sell_proceeds_settlement_days 0, each applies at its own
  instant; with N > 0, at the start of the Nth trading date after its own
  date per trading_calendar, through the revive-credit queue. Inert when
  initial_cash is 0. A hold release is not a sell-proceeds credit.

  Circuit breaker (R-4): realised loss per exit fill increment (a carried
  row's Case B or carried Case C remainder value and its split fraction
  value at their own instants — Carried position) and the consecutive-loss
  counter at close completion — see live_mode_runner.md's Circuit
  Breaker — update breaker_max_consecutive_losses and
  breaker_max_realized_loss_abs/_pct, and
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

A credit whose date is never processed (neither an entry candidate nor a
carried position that day) applies at the start of the next processed
date — no capital is lost, and nothing could have consumed it in the
interval anyway.

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
    # or the breaching synthetic tick's — directly as the fill simulation's
    # anchor only, not as trade_log.exit_bar (the close-completion instant —
    # Carried position's Close completion). The
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
    # exit_anchor_bar / exit_anchor_date = effective_second and its date

    weighted_avg_exit_price, total_filled, unfilled_qty = simulate_exit(
        anchor=effective_second, reference_price=effective_price,
        quantity=quantity, exit_reason=exit_reason,
    )                                   # Exit fill simulation, below
    # ohlcv_exit is gone — it had no reader, and the slice passed here was
    # filtered on `hour` alone across a MULTI-DATE frame while the exit ticks were
    # correctly date-scoped, which is itself evidence nothing read it.

    if unfilled_qty > 0:
        # A REMAINDER — partial or zero fill once the date's last synthetic
        # tick is read — is CARRIED (Carried position, below): no
        # dead_position_penalty_pct on it; its pnl is the completion row's.
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

**`trade_log.exit_bar`**: the close-completion instant (Carried position's
Close completion), for every exit reason — db_schema.md's
`trade_log.exit_reason` values section. The EXIT ANCHOR INSTANT is the fill
simulation's anchor, recorded in `exit_anchor_bar` / `exit_anchor_date`,
the reference mode included and NULL on never-opened rows:
- on the tp/sl path, `effective_second` above rather than the raw
  `exit_hour`;
- on the `time_limit` path, the instant the valid-bar count was reached;
- on the `session_end` path and the dead-position Cases reached through it,
  `exit_deadline(date)`.

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
        reference = the price of the last synthetic tick whose time's
                    integer part is less than the anchor (no fallback)
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
        reference = the price of the last synthetic tick whose time's
                    integer part is less than the anchor (no fallback)
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
            m = the close of the last bundle sealed (effective end time)
                before the re-quote instant
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
      total_filled, unfilled_qty = quantity - total_filled,
      the exit fill increments (execution_common.md's FillState) and the
      close-completion instant (Carried position's Close completion)
    partial_fills_count = the number of exit fill increments
    the date's last synthetic tick read with quantity remaining → the
      position is carried (Carried position)
```

### Carried position

**Carry.** When an exit's simulation reads the date's last synthetic tick
with quantity remaining, whether or not any of it filled, the position is
carried instead of taking `dead_position_penalty_pct` on the remainder:
- it enters the next processed date's `active_positions` (Chronological
  Simulation) and is carried date by date until its remainder fills;
- a trade belongs to the run of its entry date; a position still carried
  after a run's last test date keeps being carried on later dates with DB
  data, the date loop continuing with no entry candidates until no carried
  position remains; a regime holdout run follows the same rule.

**Carried date states**, tested at each carried date:
- the ticker in `ticker_data_coverage` with synthetic ticks that date → the
  exit simulation runs;
- in coverage with no synthetic ticks that date → carried to the next date;
- the next `has_data = TRUE` date with the ticker absent from
  `ticker_data_coverage` → Case B pricing on the remaining quantity;
- no later date in the dataset → CARRIED CASE C PRICING: the remaining
  quantity priced at the last synthetic tick price on the latest date the
  position had synthetic ticks, normalised to the completion date's basis,
  × (1 − `backtest.dataset_end_carry_haircut_pct`), `dead_position_case`
  'C'.

**Split normalisation.** At each carried date the remaining quantity is
multiplied by `utils.cum_split_ratio()` over (the previous date the
position was in, this date]; a fill increment's per-share cost is
`fill_price` ÷ `cum_split_ratio()` over (the entry date, the increment's
date].

**Split fraction disposal**, per `backtest.split_fraction_disposal`:
- a carried remainder's exit order quantity is the floor of the remaining
  quantity, and a remainder below one share runs no exit simulation;
- 'cash_in_lieu' and 'held' leave the fraction to the split fraction value
  (below), with `backtest.split_fraction_held_haircut_pct` under 'held';
- 'round_up' adds the ceiling − the remaining quantity at zero cost,
  excluded from `fill_price`, the per-share cost and `trade_log.quantity`,
  sold with the remainder, its proceeds in pnl.

**Carried date exit:**
- the exit simulation starts at that date's first session boundary
  (pre-market), the order type per `permitted_order_type()` at that phase;
- on a split or reverse split `event_date` of the ticker it starts at the
  first session boundary + `backtest.split_day_exit_delay_seconds`; when no
  synthetic tick follows that instant the position is carried to the next
  date, with no delay there;
- no exit trigger is evaluated — tp/sl, `time_limit` and `session_end`
  included; the remainder is only liquidated under its inherited
  `exit_reason`;
- the elapsed time compared with `execution.exit_order_stuck_minutes` runs
  from the position's original exit anchor;
- a limit remainder is quoted at `execution.exit_ladder_seed` at that
  date's anchor; at each later re-quote, with that elapsed time past
  `execution.exit_order_stuck_minutes`:
  - outside the regular session, k advances by
    `execution.exit_ladder_increment` up to `execution.exit_ladder_cap`;
  - inside the regular session, the remainder escalates to market pricing;
  - k does not carry over from the previous date;
- the ladder's m and s:
  - until that date's first bundle is sealed, m is the m held at the end of
    the previous date the position was carried in ÷
    `utils.cum_split_ratio()` over (that date, this date], and s is the s
    held then;
  - from that date's first sealed bundle on, m and s follow the Exit fill
    simulation's rules over that date's bundles and bars only;
  - a carried position holds its end-of-date m and s across the date
    boundary, the carry's first date taking the entry date's last sealed
    bundle close.

**Split fraction value.** Under 'cash_in_lieu' or 'held', the
below-one-share fraction a split leaves on a carried position:
- is valued at fraction × the position's exit-fill
  `weighted_avg_exit_price`, or, with no exit fill, fraction × the close of
  the last regular-session `ohlcv_1min` bar of the first trading date on or
  after `event_date` on which the ticker has a regular-session bar — the
  search moving date by date and, on reaching a date the ticker is absent
  from `ticker_data_coverage` or the dataset end, applying Case B or carried
  Case C pricing respectively to the fraction;
- under 'held', that value × (1 − `backtest.split_fraction_held_haircut_pct`);
- joins the close credit (`backtest.sell_proceeds_settlement_days` applies)
  and enters the breaker realised loss as value − fraction × `fill_price` ÷
  the split factor from the entry date through `event_date`;
- is excluded from `trade_log.weighted_avg_exit_price` and enters pnl only;
- its instant: the whole-share remainder's close-completion instant; with
  no whole-share remainder, the event date's first session boundary when
  the position has exit fills, otherwise the close of the bar that priced
  it;
- `exit_reason` keeps the inherited value. Dead position Case A is
  unchanged.

**Carried dividend.** For each 'dividend' `corporate_events` row with
`event_date` after the entry date and no later than the completion date,
the cash by `utils.dividend_gross_amount()`'s cash-flow reader rule — the
shares held being the remaining quantity at the start of that `event_date`,
in that date's basis:
- credited at the close-completion instant as part of the close credit,
  under `backtest.sell_proceeds_settlement_days`;
- enters pnl, not the breaker realised loss.
Dead position Case A's dividend term is unchanged.

**Carried exit penalty.** Each exit fill amount on a date after the
position's first exit date is taken × (1 − `backtest.carried_exit_penalty_pct`)
in the carried sell-proceeds credit, in pnl and in the breaker realised
loss; `trade_log.weighted_avg_exit_price` keeps the raw fill prices. The key
does not apply to the split fraction value, the carried dividend, or Case B
/ carried Case C pricing of a carried remainder.

**Cash and realised loss:**
- each exit date's fill amounts, × (1 − `backtest.carried_exit_penalty_pct`)
  on a date after the first exit date, are credited at the instant of the
  last synthetic tick that date's exit simulation read — the completion
  date's at the close-completion instant; each credit is a sell-proceeds
  credit under `backtest.sell_proceeds_settlement_days`, dated by its own
  date;
- the breaker realised loss accrues per exit fill increment at its second,
  as filled quantity × (p − c):
  - p the increment price, × (1 − `backtest.carried_exit_penalty_pct`) when
    the increment's date is after the first exit date;
  - c = `fill_price` ÷ `cum_split_ratio()` over (the entry date, the
    increment's date];
- a carried row's remaining quantity priced by Case B or carried Case C
  accrues its value − that quantity × c at that row's `exit_bar`, c taken at
  that date;
- no exit takes `dead_position_penalty_pct` on an unfilled remainder.

**Close completion.** The close-completion instant:
- the second of the last exit fill when the quantity fills completely, on
  whichever date that fill falls;
- the anchor when `exit_fill_mode()` returns the reference mode;
- on a carried row whose remaining quantity took Case B or carried Case C
  pricing, `last_bar` of the last date the position was carried in;
- a completion by the split fraction alone, the fraction value's instant.
`trade_log.exit_bar` stores its integer second via `utils.hour_to_int()`
and `exit_date` its date; the dead-position Cases resolved on the entry date
keep their own `exit_bar` / `exit_date` (Dead Position).

**Completion row.** One `trade_log` row per position, written at its close
completion, in the completion date's basis (db_schema.md's `trade_log`
basis comment):
- completed by exit fills: the inherited `exit_reason`, `is_dead_position`
  FALSE;
- remaining quantity priced by Case B or carried Case C at a carried date:
  the inherited `exit_reason`, `is_dead_position` TRUE, `dead_position_case`
  'B' or 'C', `exit_bar` / `exit_date` `last_bar` of the last date it was
  carried in and that date; the remainder value is credited at that
  `exit_bar` under `backtest.sell_proceeds_settlement_days`;
- pnl = (the first exit date's fill amounts + the later fill amounts × (1 −
  `backtest.carried_exit_penalty_pct`) + the remaining quantity × the Case
  price — 0 under Case B; under carried Case C the last synthetic tick
  price on the latest date the position had synthetic ticks, normalised to
  the completion date's basis, × (1 −
  `backtest.dataset_end_carry_haircut_pct`) — + the split fraction value +
  the carried dividend − `fill_price` × `quantity`) ÷ (`fill_price` ×
  `quantity`), the remainder term only on a Case B / carried Case C row;
- basis normalisation, R being `cum_split_ratio()` over (the entry date,
  the completion date]:
  - `fill_price` ÷ R;
  - `quantity` and `requested_quantity` × R;
  - `weighted_avg_exit_price` = Σ fill amounts ÷ Σ filled quantities, each
    normalised to the completion date;
  - `slippage_pct` taken before normalisation;
- `unfilled_quantity` = `quantity` − the quantity filled on the position's
  first exit date; 0 when the position closed on that date, and 0 on a
  dead-position Case row (A, B, C or D) resolved from its entry date;
- `carry_breakdown` on a row carried past its first exit date, each key in
  the completion date's basis, values through V1:
  - `carried_qty`, `carried_amount` — the exit fills after the first exit
    date, the amount before `backtest.carried_exit_penalty_pct`;
  - `carried_penalty` — the amount `backtest.carried_exit_penalty_pct`
    deducts;
  - `fraction_qty`, `fraction_value` — after the 'held' haircut;
  - `fraction_haircut` — the amount the 'held' haircut deducts;
  - `roundup_qty`;
  - `dividend`;
  - `dead_qty`, `dead_value`;
- `fill_increments`, when `backtest.record_fill_increments` is true —
  `{"entry": [[second, quantity, price], …], "exit": [[date, second,
  quantity, price], …]}` from the `FillState` fill increments, second as
  `utils.hour_to_int()`'s integer and date 'YYYYMMDD', quantities and prices
  raw as filled in their date's basis, through V1.

**Dataset-end carry haircut measurement.** `backtest.dataset_end_carry_haircut_pct`
is set by a human from this measurement: over a run's carried positions
completed by exit fills, 1 − Σ `carry_breakdown.carried_amount` ÷ Σ
(`carry_breakdown.carried_qty` × the last synthetic tick price of the
position's first exit date, normalised to the completion date's basis).

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
        the per-share net amount by utils.md's dividend_gross_amount()
        cash-flow reader rule × the split factor over (D, that
        event_date]   (0.0 if none) — per share entered; same overnight
        window and rationale as Labeler's Case A (see 05_labeler.md); US
        splits and ex-dividend adjustments always take effect before
        market open
    revive amount = quantity × (r × exit_price + dividend_term)
        — quantity in D's basis
    pnl = (revive amount − quantity × fill_price) / (quantity × fill_price)
    exit_reason = "dead_position"
    is_dead_position = True
    trade_log row, in the basis as of its exit (db_schema.md's trade_log
    basis comment):
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
db_schema.md defines `exit_bar` as the close instant and `exit_date` as
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
    revive moment = that row's own exit_date / exit_bar, a sell-proceeds
                    credit under backtest.sell_proceeds_settlement_days
                    (Chronological Simulation)
    The cash ledger's arithmetic runs in float — Case A's amounts are
    computed values (utils.md's Numeric Rules).

which resolves per Case without further rules: A revives at its D+1
resolution timestamp; C revives immediately at D's close, since its
synthetic price needs no D+1 data; B and D have `exit_price = 0`, so the
formula yields zero and the write-off is simply the result rather than a
special case. This is what keeps `total_pnl_abs` reconcilable against the
cash ledger in every Case — crediting cash back at D's close while
recording pnl = -1.0 (B/D) or -0.5 (C) would break that identity.

For a dead position, only the credit is deferred across the date boundary,
never the position: the trade row is written complete at D's close
(BacktestEngine already reads D+1 data to resolve Case A), and the queue
carries just `(exit_date, exit_bar, revive amount)`. No slot is held and no
exit evaluation is pending, so the per-date decomposition of the
Chronological Simulation holds for it. A carried position spans dates under
the carried-position rules (Carried position).

Dead position trades are included in the winning_rate denominator.

Note: `simulate_exit_fill()` reading the date's last synthetic tick with
quantity remaining is **not** a dead position — the remainder is carried
(Carried position), for EVERY exit reason that reaches the simulator rather
than the tp/sl path alone, since `time_limit` and `session_end` now reach it
too when `execution.simulate_nondirectional_exit_fill` is on.
`is_dead_position` is TRUE only on a carried row whose remaining quantity
takes Case B or carried Case C pricing. Diagnostic via
`trade_log.unfilled_quantity` — `quantity` − the quantity filled on the
position's first exit date (Carried position's Completion row).

---

## Config Keys (pipeline_config.yaml)

```yaml
backtest:
  initial_cash: 0                    # 0 = unlimited; fixed `balance` for
                                      # execution_common.compute_position_size()
  entry_threshold:    0.5
  suppress_threshold: 0.5            # null = suppression disabled
  dead_position_boundary_pct: 0.5    # Case C write-down (dataset boundary)
  dead_position_penalty_pct: 0.05    # Case A's exit price only (Dead
                                      # Position) — an exit remainder is
                                      # carried, and an entry-side unfilled
                                      # remainder is sized down, not penalized
                                      # (see Entry Slippage Model)
  sell_proceeds_settlement_days: 0   # trading dates after its own date at
                                      # whose start a sell-proceeds credit
                                      # applies; 0 = at its own instant.
                                      # Inert when initial_cash is 0
                                      # (Chronological Simulation)
  market_buy_spread_adjust:  true    # market-BUY limit price takes
                                      # × (1 + s / 2) (Entry Slippage Model)
  open_buy_cash_hold:        'none'  # 'none' | 'order_amount' — entry cash
                                      # (Entry Slippage Model)
  open_buy_hold_release_seconds: 0   # 'order_amount': seconds after entry
                                      # end at which the unfilled hold is
                                      # credited back
  carried_exit_penalty_pct:  0.05    # exit fill amounts on a date after the
                                      # first exit date (Carried position);
                                      # independent of
                                      # dead_position_penalty_pct
  split_fraction_disposal:   'cash_in_lieu'  # 'cash_in_lieu' | 'round_up' |
                                      # 'held' — a carried position's split
                                      # fraction (Carried position)
  split_fraction_held_haircut_pct: 0.05  # 'held' haircut on the split
                                      # fraction value; independent of
                                      # dead_position_penalty_pct
  split_day_exit_delay_seconds: 0    # on a split / reverse split event_date,
                                      # a carried exit starts at the first
                                      # session boundary + this
  dataset_end_carry_haircut_pct: 0.05  # carried Case C pricing; independent
                                      # of carried_exit_penalty_pct and
                                      # dead_position_boundary_pct; set from
                                      # the measurement in Carried position
  record_fill_increments:    true    # write trade_log.fill_increments
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
partial_fills_count      INTEGER,   -- the number of exit fill increments (per integer
                                    -- second — execution_common.md's FillState) —
                                    -- live's exit_fill_seconds definition
unfilled_quantity        DECIMAL(18,6), -- quantity − the quantity filled on the first exit
                                    -- date; 0 when the position closed on that date,
                                    -- and 0 on a dead-position Case row (A, B, C or D)
                                    -- resolved from its entry date; in the completion
                                    -- date's basis
is_ambiguous             BOOLEAN,   -- True if simultaneous bundle-level tp/sl breach
exit_date                VARCHAR,   -- 'YYYYMMDD'; equals `date` for every exit resolved
                                    -- on the entry date. Differs for dead position
                                    -- Case A (D+1), for a carried row closed on a later
                                    -- date than its entry date, and for live rows
                                    -- closed in a later session — the list is
                                    -- db_schema.md's. NOT NULL: an always-populated column
                                    -- keeps every date-scoped query free of COALESCE.
requested_quantity       DECIMAL(18,6), -- quantity actually SUBMITTED, i.e. after
                                    -- check_funds_available() sized it down. It
                                    -- decides whether an order filled in full,
                                    -- which ends fit_execution_params()'s
                                    -- eligible-volume window (shadow_retraining.md);
                                    -- a funds-driven reduction is kept out of it
                                    -- for that reason — same reason
                                    -- entry_rejected is excluded from fit_execution_params()
dead_position_case       VARCHAR,   -- 'A' | 'B' | 'C' | 'D' on the dead-position Case
                                    -- rows resolved on the entry date; 'B' | 'C' on a
                                    -- carried row whose remaining quantity took Case B
                                    -- or carried Case C pricing
carry_breakdown          VARCHAR,   -- JSON, rows carried past their first exit date
                                    -- (Carried position's Completion row)
exit_anchor_bar          INTEGER,   -- the exit anchor instant (Exit Logic), via
exit_anchor_date         VARCHAR,   -- utils.hour_to_int(); NULL on never-opened rows
entry_end_cause          VARCHAR,   -- 'filled' | 'window_end' | 'breach' |
                                    -- 'phase_boundary' (Entry Slippage Model)
fill_increments          VARCHAR,   -- JSON, when backtest.record_fill_increments is
                                    -- true (Carried position's Completion row)
-- Definitions: db_schema.md's trade_log; NULL on LiveModeRunner rows.
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
- OHLCV loaded once per ticker for all its entry and carried dates
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
  close to a tick-level partial fill, and can produce carried remainders
  where none existed — recorded here so a later session
  does not read that movement as a defect
- The key lives under `execution:`, not `backtest:`, because live shadow reads
  the same value: it is a SEMANTICS SELECTOR both engines share, and a
  backtest-only switch would make them diverge exactly when it is off, which
  is the divergence it exists to remove. With it off, live writes the
  reference price straight to `exit_price` rather than calling the simulator,
  so on and off branch over the SAME anchor
- After-market data used only as fallback when exit_deadline(date) bar is halt/no_data
- Dead position: session_end fallback fails only (exit_deadline(date) halt + no after-market data)
- Dead position lookup uses `has_data = TRUE` filter (not `is_trading_day`) — consistent with Labeler;
  so do the carried-date state tests (Carried position)
- Dead position Case A pnl uses the split-normalised exit and the net dividend term
  (see Dead Position section) — `p_entry` is never adjusted;
  `corporate_events` rows for (ticker, event_date IN (D, D+1]) are loaded alongside
  `trading_calendar`/`ticker_data_coverage` for this lookup; same query pattern as Labeler.
  A carried position's split normalisation and dividend follow the
  carried-position rules, its `corporate_events` rows covering the carried
  interval (Carried position)
- Dead position Case D (`exit_reason = "dead_position_extended_halt"`) triggers only when
  Case A's exit_price cannot be resolved after fallback — pnl = -1.0, consistent with Case B
- `simulate_exit_fill()` reading the date's last synthetic tick with
  quantity remaining — a partial or a zero fill — is not a dead position:
  the remainder is carried (Carried position). `is_dead_position` is TRUE
  only on a carried row whose remaining quantity takes Case B or carried
  Case C pricing; diagnostic via `trade_log.unfilled_quantity`
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
- `entry_bar` and `exit_bar` stored as int via `utils.hour_to_int()`; what
  `exit_bar` holds per exit reason is the `exit_bar` (close instant) column
  of db_schema.md's `trade_log.exit_reason` values section
- `trades_by_signal` and `trades_by_exit` JSON-serialized before writing to `experiment_log`
- Dead position trades included in winning_rate denominator
- Never-opened rows (the never-opened family of db_schema.md's
  `trade_log.exit_reason` values) excluded from winning_rate / total_trades /
  avg_pnl_pct / total_pnl_abs —
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
