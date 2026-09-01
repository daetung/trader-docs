# Module: Execution Common

**File:** `src/utils/execution_common.py`

---

## Role

Execution-time decision logic shared between BacktestEngine (simulated) and
LiveModeRunner (real). Functions here answer "should we enter/exit, and at
what size/price" — as opposed to `utils.py`, which additionally serves
Labeler/training-side consumers (`build_effective_bar_sequence()`,
`track_price_breach()`, `find_fill_bundle()`, `interpolate_bundle_price()`
stay in utils.md because Labeler depends on them via `track_label_breach()`).

Functions moved here are backtest-only *today* only because live does not
yet call them (e.g. fill simulation) — not because they are architecturally
tied to backtest. Splitting this module out keeps `utils.py` from growing
indefinitely as live mode's execution surface expands.

---

## Functions

### Signal Resolution

```python
def resolve_signal(
    row: pd.Series,
    threshold: float,
    suppress_threshold: float | None = None,
) -> str | None:
    """
    Determine entry signal from model probability output.
    up5 takes priority over up3.

    Suppression logic:
        If suppress_threshold is not None, downside classifiers (dn5, dn3)
        are checked first. If either exceeds suppress_threshold, the signal
        is suppressed regardless of upside probabilities.

        This prevents entering long positions when the model has strong
        conviction of a downside move, even if upside probabilities also
        happen to exceed threshold.

    Used by BacktestEngine and Inferencer.

    Setting suppress_threshold higher than entry_threshold results in more
    permissive suppression (fewer entries blocked). Setting them equal is
    the recommended starting point.

    Logic:
        if suppress_threshold is not None:
            if prob_dn5 >= suppress_threshold or prob_dn3 >= suppress_threshold:
                return None   # suppressed — downside conviction overrides upside signal
        if prob_up5 >= threshold:
            return "up5"
        if prob_up3 >= threshold:
            return "up3"
        return None
    """
    ...
```

---

### Late-Entry Residual Edge

```python
def late_entry_expected_return(
    row: pd.Series,
    p_ref: float,
    p_now: float,
) -> float:
    """
    Expected return of taking a signal LATE, at p_now, when its label anchor
    is p_ref (the t bar's open).

    Used by BacktestEngine and LiveModeRunner when execution
    .late_entry_enabled is true. Both engines must compute it identically —
    the gate is what decides whether a rotation-detected candidate is taken
    at all, so a divergence here is a divergence in which trades exist.

    Logic:
        d = (p_now - p_ref) / p_ref          # drift since the anchor
        S = sum over classes of P(class) * target_multiple(class)
        return S / (1 + d) - 1

    Enter iff the result exceeds execution.late_entry_min_expected_return_pct
    — equivalently, iff S > (1 + theta) * (1 + d).

    NOT a gate on drift itself. Drift correlates POSITIVELY with model
    probability: a working model's high-probability points are the ones
    about to move, and drift is the first seconds of that move, so a
    drift threshold preferentially removes the model's strongest signals.
    Dividing by (1 + d) instead prices the drift against the edge that
    remains — a high-conviction signal tolerates a large d because its
    remaining room is large, while a marginal one fails on a small one.

    d < 0 is NOT blocked. Entering after a decline is arithmetically
    FAVOURABLE against the same forward path, and the formula already
    reflects that. Whether the conditional distribution ALSO degrades there
    is a measurement, not an assumption, and no asymmetric floor is
    imposed ahead of it.

    Reads model output, so INFERENCE RUNS FIRST in both engines. Live can
    afford this: late-entry candidates are the small rotation-detected
    subset and are adjudicated mid-bar, outside the 5-second bar-close
    deadline. Running inference first also preserves the instrumentation —
    a rejected candidate still leaves an inference_log row, so the gate's
    cost in blocked signals is measurable.
    """
    ...
```

**theta is DERIVED, not guessed.**

```
theta = Q_q({ E[r | 0] : on-time entries ACCEPTED in the baseline
                         backtest pass })
      = Q_q({ S_i - 1 })          since d = 0 for an on-time entry

default q = optimizer.execution_eval.late_entry_theta_quantile (0.10)
```

It asks of a late entry only what an on-time entry already clears. Not
simply `> 0`: `buy_rate` and slippage lift the real bar above zero, and a
marginal trade consumes cash and a slot under `exposure_cap_pct` /
`max_tickers` that a better on-time entry would otherwise use.

**Anchoring is what makes it robust to calibration.** theta and S come from
the same probability vector, so if probabilities are systematically
inflated both sides inflate together and the inequality is largely
preserved — absolute `E[r]` is biased, the COMPARISON is not. The same
cancellation covers `target_multiple` being a label target rather than a
realised return. This is a NEW dependency worth naming: until now the
probability vector was used only for ranking (`resolve_signal()`'s
threshold comparisons), and here it enters an expected-value arithmetic
directly.

**Derivation is circular unless ordered**, so evaluation is two-pass: pass 1
runs `late_entry_enabled: false` and collects the accepted on-time
`E[r | 0]` distribution, theta is resolved from it per grid quantile, then
passes 2..n run. See `pipeline_optimizer.md`'s `execution_eval`.

**TWO config keys, not one.**
`execution.late_entry_min_expected_return_pct` holds the RESOLVED scalar and
is what both engines read — live must never recompute theta at run time or
it diverges from backtest — while
`optimizer.execution_eval.late_entry_theta_quantile` is derivation-only and
is read by the offline utility and the grid alone.

---

### Cooldown Guard

```python
def can_enter(
    ticker: str,
    current_hour: str,
    last_entry_hour: str | None,
    cooldown_minutes: int,
) -> bool:
    if last_entry_hour is None:
        return True
    current_min = utils.hour_to_minutes(current_hour)
    last_min    = utils.hour_to_minutes(last_entry_hour)
    return (current_min - last_min) >= cooldown_minutes
```

**Cooldown across session boundaries (session_mode="combined"):**
Cooldown is applied continuously across the full time axis regardless of
session boundaries. No cooldown reset at the pre-market/regular session boundary.

**Cooldown across DATE boundaries: reset.** A different axis from the
session rule above, and not to be conflated with it — `last_entry_hour` is
reset to `None` at every date boundary. `hour_to_minutes()` is
minutes-since-midnight and carries no date, so a `last_entry_hour` carried
over from a late prior-day entry makes `current_min - last_min` NEGATIVE
for the next morning's candidates: the guard then returns False and blocks
that ticker for essentially the whole following day. LiveModeRunner gets
this for free, since the state is session-scoped and a session is one date;
BacktestEngine must reset explicitly as it advances from one date to the
next (see 09_backtest_engine.md's Chronological Simulation).

---

### Tradability Gate

```python
def is_tradable(ticker: str, db_conn) -> bool:
    """
    Single gate for "should we allow a NEW entry into this ticker right
    now". Checked once, at the point EntryPointDetector confirms a
    candidate, before sizing/order submission.

    Checks ticker_cik_map.rename_pending (rename-pending-confirmation case)
    and .quarantine_reason (P-8's same-day corporate-event anomaly case —
    see metadata_crawler.md's check_corporate_event_anomaly()) — tradable
    only when BOTH are clear. The two are independent columns (N-2 — see
    db_schema.md's ticker_cik_map) specifically so this OR-of-two-blockers
    check can be a flat condition with no cross-reason interaction to
    reason about. Deliberately does NOT check tick-rate/staleness/halt —
    EntryPointDetector's own volume-gated filters (B/C/D — see
    01_entry_detection.md) already fail on a genuinely stale or halted
    ticker, so a separate check here would be redundant.

    Returns:
        True if the ticker is tradable (rename_pending=FALSE AND
        quarantine_reason IS NULL, or no ticker_cik_map row exists at all
        — an unmapped ticker is not a rename-ambiguity or quarantine case).
    """
    row = db_conn.execute(
        "SELECT rename_pending, quarantine_reason FROM ticker_cik_map "
        "WHERE ticker = ? ORDER BY last_seen_date DESC LIMIT 1", [ticker]
    ).fetchone()
    return row is None or (row[0] is False and row[1] is None)
```

---

### Position Sizing

```python
def compute_position_size(
    balance: float,
    fill_price: float,
    mgnrt: float,
    t_bar_volume: int,
    ticker_margin_used: float,
    total_margin_used: float,
    position_size_cash_pct: float,
    position_size_vol_pct: float,
    per_ticker_share_cap_pct: float,
    exposure_cap_pct: float,
) -> int:
    """
    Compute per-trade buy quantity. Shared by BacktestEngine and
    LiveModeRunner — same formula, different `balance` source:
        backtest: initial_cash (config, fixed for the whole run)
        live:     session_start_cash — the UNCORRECTED DEPOSIT
                  (balance-margin's AstkOrdAbleAmt), queried once at session
                  start and fixed for the session
    `balance` is deliberately NOT a running/decrementing value — see
    Constraints below for why, and see the separate funds-availability gate.

    EVERY BALANCE-DERIVED TERM IS IN MARGIN UNITS. One definition governs:

        margin_of(price, qty, rate) = price * qty * (rate / 100)

    so the three of them read as ONE statement: the margin this entry locks
    stays within a fraction of the deposit. `balance` stays the uncorrected
    deposit in all three; the leverage correction lives in the denominator
    and in the deductions, never in `balance` itself.

        m_now = fill_price * (mgnrt / 100)

    cash_based = floor((position_size_cash_pct * balance) / m_now)
    vol_based  = floor(t_bar_volume * position_size_vol_pct)   # UNCHANGED

    per_ticker_share_cap_pct == 0 → ticker_cap_qty = inf (uncapped)
    else:
        ticker_cap_qty = floor(
            max(0, per_ticker_share_cap_pct * balance - ticker_margin_used)
            / m_now
        )

    exposure_cap_pct == 0 → exposure_cap_qty = inf (uncapped)
    else:
        exposure_cap_qty = floor(
            max(0, exposure_cap_pct * balance - total_margin_used)
            / m_now
        )

    quantity = min(cash_based, vol_based, ticker_cap_qty, exposure_cap_qty)

    position_size_cash_pct reaching mgnrt / 100 is therefore EXACTLY full
    deposit deployment, and is where the liquidation-defence ceiling sits:
    above it an entry draws on credit rather than on its own capital.

    Pre-correcting `balance` instead (effective_balance = balance *
    100 / mgnrt) was REJECTED although algebraically equivalent for
    cash_based and ticker_cap: it makes `balance` per-ticker, breaking its
    "fixed reference for the run/session" constraint, and it gives
    exposure_cap — an ACCOUNT-WIDE limit — a per-ticker baseline.

    At mgnrt == 100 every term reduces to the previous notional formula
    EXACTLY (m_now == fill_price, margin_of == notional), so BacktestEngine
    passes the constant 100 and needs no branch. That is why the former
    live-only `sizing_basis` key has no successor rather than a backtest
    counterpart — there is nothing left to select between.

    Args:
        balance:            fixed reference capital (see above) — not a
                             running/decrementing value, and NOT
                             margin-corrected
        fill_price:          slippage-adjusted entry price
        mgnrt:               this ticker's margin rate as a PERCENT. Live
                             passes live_ticker_terms.mgnrt (the vendor's
                             Mgnrt0 — see live_mode_runner.md's Per-Ticker
                             Trading Terms); BacktestEngine passes the
                             constant 100. NOT a config key: it is a
                             per-ticker vendor fact, and no account-level
                             rate exists to hold instead.
        t_bar_volume:        ohlcv_1min volume of the t bar
        ticker_margin_used:  sum(margin_of(entry_price_i, quantity_i,
                             rate_i)) across this ticker's currently-open
                             positions, PLUS its submitted-but-unfilled ones
                             (R-5: live_positions rows with
                             lifecycle='live' AND entry_state='awaiting' —
                             priced at
                             limit_price, or p_entry for a market order).
                             Each row contributes at its OWN PINNED RATE
                             (live_positions.entry_mgnrt), never the current
                             one, so an open position's margin cannot move
                             retroactively. Pending margin is reserved
                             capacity: excluding it lets a burst of
                             simultaneous submissions breach the cap on
                             aggregate fill, which is inert only while
                             per_ticker_share_cap_pct is 0 and becomes real
                             the moment Pilot sets it. BacktestEngine has no
                             pending state, so there the two sums coincide.
        total_margin_used:   the same sum across all positions, any ticker,
                             on the same open-plus-pending basis
        position_size_cash_pct:   fraction of balance per trade, MEASURED IN
                             MARGIN (cash leg)
        position_size_vol_pct:    fraction of t bar volume per trade
                             (liquidity leg) — the one term the margin unit
                             does not touch, being a liquidity bound rather
                             than a capital one
        per_ticker_share_cap_pct: 0 = disabled; >0 = cap on the margin this
                             ticker's positions lock, as a fraction of
                             balance
        exposure_cap_pct:    0 = disabled; >0 = cap on margin locked across
                             all open positions, as a fraction of balance

    Returns:
        int quantity (>= 0)
    """
    ...


def check_funds_available(
    quantity: int,
    fill_price: float,
    available_cash: float,
    use_all_cash: bool = True,
) -> tuple[bool, int]:
    """
    Funds-availability gate — deliberately separate from
    compute_position_size(). Sizing answers "how big should this trade be
    given fixed balance and caps"; this answers "do we actually still have
    the cash right now". Conflating the two (e.g. sizing off a
    decrementing running balance) causes position size to shrink as the
    session progresses even though the caps did not change — this
    function keeps that a pure availability check instead.

    backtest: available_cash is remaining_cash — the caller's REVOLVING
              capital: decremented by the amount actually committed when an
              entry fills, and credited back when the position closes (see
              09_backtest_engine.md's Chronological Simulation). Not a
              one-way budget; a monotonically-decreasing remaining_cash
              models a single deployment rather than a book that turns
              over, and silently stops trading partway through any run
              with initial_cash > 0.
    live:     available_cash is COMPUTED, not observed:

                  available_cash = AstkOrdAbleAmt * (100 / mgnrt) - @cost

              `AstkOrdAbleAmt` from inquiry/balance-margin (3 TPS,
              account-scoped), queried fresh immediately before this check;
              `mgnrt` and `@cost` from this ticker's persisted
              live_ticker_terms row (live_mode_runner.md's Per-Ticker
              Trading Terms). Reading AstkOrdAbleAmt1 directly would be
              exact and need no arithmetic, but that field exists ONLY on
              able-orderqty, so a direct read puts a 2 TPS per-ticker call
              back on the entry path — the burst the acquisition point was
              moved to avoid. The computed figure runs ALWAYS-HIGH, `@cost`
              being subtractive and derived once from possibly-stale
              conditions, so the bias is one-sided toward over-permitting;
              the order-time observation is what surfaces drift.

    THE COMPARISON STAYS NOTIONAL AGAINST NOTIONAL. compute_position_size()'s
    margin-unit form does NOT extend here. Its three balance-derived terms
    are one question in three forms, so a split unit there risks one being
    fixed alone; this gate asks a different question — what can actually be
    paid now — and `available_cash` above is already leverage-inclusive.
    Multiplying the left side by mgnrt / 100 would apply the same factor
    twice: returning the right side to deposit terms leaves `@cost`
    dimensionally wrong, and leaving it as-is loosens the gate by exactly
    that factor.

    use_all_cash:
        quantity * fill_price <= available_cash → always (True, quantity),
        regardless of this flag — no adjustment needed when funds are
        already sufficient.

        Otherwise (insufficient funds for the full requested quantity):
            True (default): adjusted = floor(available_cash / fill_price)
                adjusted == 0 → (False, 0) — not even one share affordable
                else → (True, adjusted) — proceed sized down to fit
                available_cash exactly
            False: → (False, 0) — skip the trade entirely, no partial size

    Returns:
        (proceed, adjusted_quantity) — caller uses adjusted_quantity (not
        the original quantity argument) for the trade when proceed is True.
    """
    ...
```

**Config keys (shared — see `execution:` section in relevant script/runner docs):**
```yaml
execution:
  position_size_cash_pct:   0.05   # 5% of balance per trade
  position_size_vol_pct:    0.10   # 10% of t bar volume
  per_ticker_share_cap_pct: 0.0    # 0 = disabled; >0 = fraction of balance
  exposure_cap_pct:         0.0    # 0 = disabled; >0 = fraction of balance
  max_tickers:               10    # R-5: max DISTINCT tickers held at once
                                   # (open + pending). 0 = unlimited, and
                                   # BacktestEngine only — LiveModeRunner
                                   # clamps 0 to 50, the WS price-tracking
                                   # sequence's vendor ticker limit, which is
                                   # not a configurable choice.
  max_positions_per_ticker:   2    # R-5: max concurrent positions on ONE
                                   # ticker. 0 = unlimited, backtest only —
                                   # live clamps 0 to floor(max_hold_bars /
                                   # entry_cooldown_minutes), the largest
                                   # same-ticker stack the cooldown can admit
                                   # inside one position's maximum hold.
                                   # Both clamps are derived, not
                                   # configurable — same character as
                                   # get_execution_param()'s hard bounds.
                                   # Replaces the former live_mode
                                   # .max_positions, which was a single
                                   # global count and so could not express
                                   # "N positions, all on one ticker".
  max_hold_bars:             60    # R-6: single source. Consumers are given
                                   # as a LIST, not a count: a count reads as
                                   # complete while a member sits outside it,
                                   # which is how the Labeler was missed.
                                   #   - backtest's time-limit exit
                                   #   - live's ORDINARY Position Manager
                                   #     time_limit exit
                                   #   - live's restart_gap_exit /
                                   #     overnight_exit cutoff
                                   #   - the Labeler's collection stop
                                   # The first and the last reach it by the
                                   # same route: 09_backtest_engine.md and
                                   # 05_labeler.md each pass this key into
                                   # utils.md's build_effective_bar_sequence()
                                   # as target_valid_bars, which carries no
                                   # default of its own. A grep on this key
                                   # therefore reaches that near-miss argument
                                   # name from either end.
                                   # One key serves both readings because they
                                   # are the same quantity:
                                   #   valid bars (build_effective_bar_sequence)
                                   #     == elapsed minutes − halted minutes
                                   # — that function excludes halt bars from the
                                   # valid count and INCLUDES forward-filled
                                   # no_trade bars, so live counting elapsed
                                   # minutes less halted ones (from
                                   # live_halt_episodes) lands on the same
                                   # number backtest reaches by counting bars.
                                   # The key NAME stays: backtest genuinely
                                   # counts bars, and renaming would break
                                   # R-6's single-source record without
                                   # changing anything the key means.
  entry_cooldown_minutes:     5    # R-5/R-6: single source — can_enter()'s
                                   # cooldown_minutes for both engines.
                                   # entry_cooldown_minutes * 60 >=
                                   # cancel_after_seconds must hold: live's
                                   # submission-time cooldown is what stops a
                                   # re-flagged ticker double-submitting while
                                   # its own order is still in flight, and
                                   # that protection follows from these two
                                   # values' relative size, not from any
                                   # structural guard.
  session_close_exit_offset_minutes: 1
                                   # R-7: single source. Was declared under
                                   # live_mode: while 09_backtest_engine.md
                                   # carried the bare literal — the same
                                   # one-sided declaration of a value both
                                   # engines must agree on that R-6 fixed for
                                   # max_hold_bars.
                                   # RENAMED AND RE-MEANT from the former
                                   # session_close_exit_time: "155900". It is now
                                   # an OFFSET, resolved per date through
                                   # utils.md's exit_deadline() as that date's
                                   # session_close minus this many minutes —
                                   # 155900 ordinarily, 125900 on an early close.
                                   # NOT deleted in favour of pure calendar
                                   # derivation: 155900 happened to equal the
                                   # last bar, but the value is a POLICY —
                                   # being flat by 15:50 must stay expressible —
                                   # and deriving it entirely from the calendar
                                   # would remove that freedom. R-7's
                                   # single-source property survives untouched:
                                   # both engines resolve from the same calendar
                                   # and the same offset.
                                   # An OFFSET here while trading_calendar
                                   # .after_hours_end is a COLUMN is not an
                                   # inconsistency — a measured FACT about the
                                   # venue gets a column, a chosen POLICY gets an
                                   # offset. after_hours_end is not ours to
                                   # choose; this is.
                                   # THE ENGINES THAT MUST AGREE are backtest,
                                   # live AND the labeler: 05_labeler.md carried
                                   # its own session_close_hour: "155900" until
                                   # this change deleted it. R-7's "both engines"
                                   # framing named a set it did not enumerate.
  breach_confirm_window_seconds: 10
                                   # A pending first breach is discarded if no
                                   # confirming second tick arrives within this
                                   # window. "Two consecutive raw ticks" is
                                   # SEQUENCE adjacency, not time adjacency, so
                                   # without a bound a quiet ticker leaves a
                                   # pending state alive indefinitely and pairs
                                   # ticks minutes apart. Pre-measurement
                                   # placeholder: it covers several REST
                                   # backstop poll cycles at
                                   # position_check_interval_seconds and is a
                                   # sixth of a 1-minute bar against
                                   # max_hold_bars. It joins the guard
                                   # constants feed_coverage_daily's density
                                   # buckets exist to resolve against data.
  ws_ticker_limit:           50    # R-7: tickers ONE WS CONNECTION can carry.
                                   # A VENDOR fact, not a preference — kept in
                                   # config rather than as a literal because
                                   # it varies by broker and is measured, not
                                   # chosen. See api_contract_checklist.md,
                                   # which names this key as where its
                                   # measurement is recorded. max_tickers: 0
                                   # (unlimited) clamps to this in live.
  # sizing_basis removed. It existed to select an INTERPRETATION of a
  # balance figure whose meaning was unverified. Both fields are now named
  # outright — AstkOrdAbleAmt is the deposit, AstkOrdAbleAmt1 the buying
  # power — so there is nothing to select between, and the leverage term
  # enters compute_position_size() as a per-ticker argument rather than as a
  # session-wide interpretation of `balance`.
  # R-4 circuit-breaker thresholds. All THREE default to 0 = NO LIMIT, matching
  # exposure_cap_pct / per_ticker_share_cap_pct's "off until Pilot" pattern.
  # 0 is unambiguous for each: "trip after 0 consecutive losses" or "after a 0%
  # loss" has no sensible reading other than disabled.
  # These are not three views of one signal — they fire at different points on
  # the failure timeline. A feature bug that emits garbage entries shows up
  # first as an entry-rate spike (before any loss exists), then as a run of
  # losing exits, and only last as accumulated realised loss.
  intraday_loss_limit_pct:   0.0   # realised loss for the session, as a
                                   # fraction of the same equity basis used for
                                   # sizing. Realised only: unrealised swings
                                   # would trip on ordinary intraday movement.
                                   # Accept that this makes it a LAGGING
                                   # signal — if the model breaks, positions
                                   # are already at full exposure before any of
                                   # them closes. The other two cover that
                                   # window; this one is the backstop.
  consecutive_loss_limit:      0   # consecutive LOSING exits (pnl_pct < 0),
                                   # not consecutive stop-losses, and reset
                                   # only by a PROFITABLE exit. What is being
                                   # detected is "the model is systematically
                                   # wrong", for which the exit label is
                                   # incidental: a time_limit exit at -0.3% is
                                   # equally evidence. Under a stop-loss-only
                                   # reading, "3 stops, one small time_limit
                                   # loss, 3 stops" never reaches any
                                   # threshold while the account bleeds.
                                   # PnL-excluded exits (operational family)
                                   # and never-opened rows neither increment
                                   # nor reset — they are not the strategy's
                                   # verdict on anything.
  entries_per_hour_limit:      0   # ROLLING 60 minutes, not clock-hour
                                   # buckets: bucketing lets 59 entries at
                                   # 10:59 and 59 more at 11:00 both pass.
                                   # The only one of the three that can fire
                                   # before any loss is realised.
  use_all_cash:              true  # see check_funds_available() — size down
                                   # to fit remaining cash instead of
                                   # skipping when funds are insufficient
                                   # for the full requested quantity
  take_profit_up5:          0.05
  take_profit_up3:          0.03
  stop_loss_pct:             0.03
  exit_interpolation:        true  # false = 1-minute bar only (asymmetric)
  sell_rate_tp:               0.30 # fraction of per-tick volume, take-profit exits
  sell_rate_sl:               0.15 # fraction of per-tick volume, stop-loss exits
  sell_rate_neutral:          0.15 # fraction of per-tick volume, DIRECTION-FREE
                                    # exits (time_limit, session_end). The other
                                    # two rates are indexed on direction — a
                                    # take-profit exit sells into a rising market
                                    # with more buyers, a stop-loss into a falling
                                    # one with fewer — and a clock-triggered exit
                                    # has no direction, having fired precisely
                                    # because NEITHER threshold was crossed.
                                    # Letting those fall through to sell_rate_sl,
                                    # as the previous `else` did, routed the
                                    # MAJORITY exit path into sell_rate_sl's
                                    # fitted population and stopped that value
                                    # measuring falling-market depth at all.
                                    # SEEDED at sell_rate_sl's value, so behaviour
                                    # is unchanged until fit_execution_params()
                                    # has pilot data to separate them.
  simulate_nondirectional_exit_fill: false
                                    # SEMANTICS SELECTOR read by BOTH engines, which
                                    # is why it sits here rather than under backtest:.
                                    # false (default): time_limit and session_end do
                                    # NOT go through simulate_exit_fill(). Backtest
                                    # takes the bar close / last valid bar's close;
                                    # live writes the REFERENCE PRICE straight to
                                    # exit_price. Those are the same number when the
                                    # trigger falls on a bar boundary, and live has no
                                    # bars to take a close from in any case, so on and
                                    # off branch over the SAME anchor rather than over
                                    # two notions of the exit moment.
                                    # true: both engines route those two reasons
                                    # through the simulator with the generalised
                                    # anchor and sell_rate_neutral.
                                    # A backtest-only switch was rejected: the two
                                    # engines would then diverge exactly when it was
                                    # off, which is the divergence this removes.
                                    # Current backtest output is preserved
                                    # BYTE-FOR-BYTE at the default, as with
                                    # backtest.late_detection_rate: 0.0.
  buy_rate:                    0.1  # seed value — conservative starting point,
                                    # refined by fit_execution_params() once
                                    # pilot-stage predicted-vs-actual data
                                    # accumulates (see shadow_retraining.md)
  cancel_after_seconds:        30   # seed value — same refinement path as buy_rate
  entry_fill_delay_seconds:     5   # seconds after the t bar's open that an
                                    # entry is expected to be ON. ONE quantity
                                    # read by both engines, which is why it
                                    # sits here beside the rest of the fill
                                    # family rather than under backtest: —
                                    # BacktestEngine models the fill AT it,
                                    # while live treats it as the reference
                                    # its own submission is measured against
                                    # (live_scan_daily's stage timings) and as
                                    # the ordering key entry submissions carry
                                    # into trading_api.md's dispatch queue.
                                    # NOT A HARD DEADLINE on the live side: an
                                    # entry past it is still submitted and
                                    # counted as entry_submit_late, because
                                    # backtest models late DETECTION but not
                                    # late SUBMISSION and dropping would
                                    # diverge in an unmodelled direction too.
                                    # It WILL move — live's inference time is
                                    # unmeasured, so the 5 is a design target.
                                    # Read from CONFIG ONLY, not through
                                    # get_execution_param()/execution_params,
                                    # despite sitting beside keys that are:
                                    # a runtime override would desync live
                                    # from the backtest run that produced the
                                    # model it is trading
  late_entry_enabled:       false  # take a rotation-detected candidate at
                                   # p_now instead of dropping its bar. Read
                                   # by BOTH engines: backtest's
                                   # late-detection model branches on it
                                   # (09_backtest_engine.md) and live's scan
                                   # gates on it (live_mode_runner.md).
                                   # Unlike market_buy_price_margin this
                                   # belongs here rather than under
                                   # live_mode: — p_now comes from the
                                   # rotation slot's second-resolution close
                                   # in live and from tick_10 in backtest,
                                   # so there is no bid/ask dependency to
                                   # keep it out of the shared block
  late_entry_min_expected_return_pct: null
                                   # RESOLVED theta — see Late-Entry
                                   # Residual Edge. null = the gate admits
                                   # nothing, so late_entry_enabled: true
                                   # with theta unset is inert rather than
                                   # unbounded. Derived offline from a
                                   # baseline backtest pass; never computed
                                   # at run time by either engine
  entry_order_type:      "market"  # "market" | "limit"
  # market_buy_price_margin lives under live_mode:, not here — it prices
  # against ask1 and BacktestEngine has no bid/ask model, the same reason
  # the exit ladder's keys sit there (live_mode_runner.md).
  entry_gap_type:    "percentage"  # "percentage" | "absolute" — limit only
  entry_gap_value:            0.0  # limit only. 0 = limit at p_entry exactly;
                                    # positive = willing to pay up to that much
                                    # more. percentage: p_entry*(1+value);
                                    # absolute: p_entry+value
  exit_order_type:       "market"  # "market" | "limit" — symmetric to
                                    # entry_order_type. NOT hard-fixed to
                                    # "market": an unprotected market order
                                    # carries slippage risk of unmeasured
                                    # size (the same one-sided-error posture
                                    # this spec set takes wherever a bound
                                    # stands in for an unknown). No
                                    # gap_type/gap_value counterpart: when
                                    # "limit", the resting price is a SPREAD
                                    # POSITION tracked against live bid/ask
                                    # (see Exit Limit Pricing below) rather
                                    # than a fixed offset from a reference
                                    # price. Escalation past
                                    # live_mode.exit_order_stuck_minutes is a
                                    # market order INSIDE the regular session
                                    # and a spread-position ladder outside it,
                                    # since the venue refuses a market order
                                    # outside regular hours (see Session Phase
                                    # and Order Types below).
  use_loo:                   false  # RESERVED, not implemented. A future
                                    # premarket entry path submitting a
                                    # limit-on-open order at the 09:19 bar
                                    # (the vendor's cutoff is 10 minutes
                                    # before the open). Premarket entry is
                                    # not enabled in this phase, and the
                                    # constraints are already known: 09:19
                                    # precedes the 09:20 premarket recheck
                                    # that gates quarantine, gap_pct does not
                                    # exist until the 09:30 bar confirms, and
                                    # an open-price fill has no
                                    # simulate_entry_fill() counterpart.
  # Moved from 09_backtest_engine.md's former `backtest:` block — backtest
  # and live must not carry independently-configurable copies of values
  # that are supposed to be identical between them. `backtest:` retains
  # only backtest-simulation-specific keys (initial_cash, entry_threshold,
  # suppress_threshold, dead_position_penalty_pct) — see
  # 09_backtest_engine.md. max_hold_bars (R-6) and entry_cooldown_minutes
  # (R-5) moved here too: LiveModeRunner reads both, so neither was ever
  # backtest-simulation-specific.

  # N-7: rejection multiplier for fit_execution_params()'s relative-bound
  # check (see shadow_retraining.md) — a refit landing outside
  # [current_value / fit_rejection_multiplier, current_value *
  # fit_rejection_multiplier] is rejected for that cycle. Deliberately
  # relative to the current authoritative value, not an absolute range —
  # simulate_exit_fill()'s own docstring already says the true participation
  # rate is unknown at design time and must come from calibration; a fixed
  # absolute range would reject a genuinely-correct calibration result that
  # happens to differ a lot from the 0.1/30 seed defaults, defeating the
  # whole reason fit_execution_params() exists.
  fit_rejection_multiplier: 3
```

### Execution-Parameter Resolution

```python
def get_execution_param(param_name: str, db_conn, config) -> float:
    """
    Single read point for buy_rate, sell_rate_tp, sell_rate_sl,
    sell_rate_neutral, cancel_after_seconds — BacktestEngine and
    LiveModeRunner both call this
    rather than querying execution_params directly, so the read-time
    hard-bound defense below exists in exactly one place (same
    single-source-of-truth pattern as resolve_signal(), can_enter()).

    1. row = latest execution_params row for param_name, if any
       (see db_schema.md's execution_params)
    2. value = row.value if row exists else config["execution"][param_name]
       (the seed default)
    3. Hard-bound check (N-7) — mathematically derived, not configurable:
           buy_rate, sell_rate_tp, sell_rate_sl, sell_rate_neutral: valid
               range is (0, 1].
               Derived directly from simulate_entry_fill()/
               simulate_exit_fill()'s floor(per_tick_vol * rate) — 0 is
               permanent zero-fill degeneracy, anything above 1.0 asserts
               participation exceeding a bundle's own volume.
           cancel_after_seconds: valid range is > 0. <= 0 is a degenerate
               "canceled before submission" value.
       value outside its range → log + fall back to
       config["execution"][param_name] (the seed), ignoring whatever was
       stored. A stored value failing this can only mean manual corruption
       or a future write-path bug reaching execution_params directly,
       bypassing fit_execution_params() — the relative-bound check there
       (see shadow_retraining.md) is what prevents a bad value from being
       written under normal operation; this is defense-in-depth at the
       read side, not the primary mechanism.
    4. return value
    """
    ...
```

---

### Exit Fill Simulation

```python
def simulate_exit_fill(
    ticks_exit: pd.DataFrame,
    position_size: int,
    exit_anchor_second: int,
    reference_price: float,
    sell_rate: float,
    halts_df: pd.DataFrame,
) -> tuple[float, int, int, str]:
    """
    Simulate partial exit fills across tick bundles from the ANCHOR onward.
    Continues through session close into after-market until position is fully
    closed or all ticks are exhausted.

    ANCHOR, generalised from the former (breach_bundle_idx, breach_price):
    every exit has a (trigger instant, reference price at that instant), and
    a breach is merely how tp/sl produces one.
        exit_anchor_second: HHMMSS of the trigger. A TIME, not an index —
            live recomputes statelessly from the anchor each pass and rebuilds
            its buffer doing so, which leaves an iloc unstable between passes
            while a time is stable. The index derivation the caller used to do
            (bundle_idx_at()) happens HERE, so both engines pass the same kind
            of thing: backtest the instant its exit fired, live
            live_positions.exiting_since.
        reference_price: the last print strictly BEFORE that instant, for all
            four exit reasons. Backtest's former "close of the last valid bar"
            / "last bar close" is not the general form — a 1-minute bar's
            close IS that minute's last print, so the two agree exactly when
            the trigger falls on a bar boundary (session_end) and differ only
            when it does not (time_limit, whose anchor is arbitrary). The
            bar-shaped version was an artifact of backtest living on bars.

    ohlcv_exit is GONE. No documented reader ever existed: every value a bar
    could have supplied already has a dedicated argument (halts_df) or comes
    from the ticks (bundle.volume, interpolate_bundle_price), and backtest's
    own slice was filtered on `hour` alone across a multi-date frame, which a
    real reader would have surfaced. If an explicit market-impact term is ever
    added and wants a minute's total volume as its denominator, the argument
    can return then.

    Caller selects sell_rate by exit reason:
        sell_rate_tp for take-profit; sell_rate_sl for stop-loss;
        sell_rate_neutral for time_limit and session_end, which have no
        direction. There is no `else` fallthrough to sell_rate_sl.

    Per-bundle fill logic (from the anchor bundle onward):
        if bundle overlaps halt interval in halts_df → skip
        per_tick_vol = bundle.volume / 10
        sellable     = floor(per_tick_vol * sell_rate)
        if sellable == 0 → skip
        filled_qty   = min(remaining, sellable)
        fill_price   = interpolate_bundle_price(prev_bundle, bundle, bundle.hour)
        total_value += fill_price * filled_qty
        total_filled += filled_qty

    Anchor bundle handling:
        fill_price = reference_price.
        sellable   = floor((anchor_bundle.volume / 10) * sell_rate)
        if sellable == 0 → skip the anchor bundle, start from the next one.

    Session close: no forced liquidation; after-market ticks processed identically.
    Ticks exhausted with remaining > 0: unfilled_quantity = remaining.

    weighted_avg_exit_price:
        Σ(fill_price_i * qty_i) / Σ(qty_i) if total_filled > 0 else
        reference_price.

    Note: this per-bundle fill price does not include an explicit
    market-impact term — it assumes participation at sell_rate does not
    materially move the observed price. This approximation is expected to
    degrade as sell_rate increases; the point at which impact becomes
    material is unknown at design time and should be estimated from
    realized-vs-simulated fill data during shadow/pilot calibration (see
    shadow_retraining.md's fit_execution_params() — sell_rate_tp/sell_rate_sl
    and sell_rate_neutral are refit there alongside
    buy_rate/cancel_after_seconds).
    """
    ...
```

---

### Entry Fill Simulation

```python
def simulate_entry_fill(
    ticks_entry: pd.DataFrame,
    quantity: int,
    entry_anchor_second: int,
    p_entry: float,
    buy_rate: float,
    halts_df: pd.DataFrame,
    cancel_after_seconds: int,
    limit_price: float | None,
) -> tuple[float, int, int, str]:
    """
    Simulate partial entry fills across tick bundles from the ANCHOR onward.
    Mirrors simulate_exit_fill() for the entry side — same per-bundle
    participation-rate structure, applied to a buy instead of a sell, and the
    same time-valued anchor with index derivation inside.

    entry_anchor_second is the ONE anchor that does not collapse to a single
    column across the two engines: backtest COMPUTES it as
    entry_hour + execution.entry_fill_delay_seconds, live OBSERVES it as
    live_positions.submitted_at. They are two expressions of one moment and
    must never be composed — adding the delay to submitted_at double-counts a
    wait live has already lived through.

    ohlcv_entry is GONE, for the same reason as ohlcv_exit. p_entry STAYS:
    its zero-fill fallback role is inert here (a zero-fill entry is
    'canceled', quantity=0, pnl NULL, and reaches no summary statistic), but
    it is still compute_position_size()'s sizing input.

    limit_price: None for entry_order_type="market" (no price gate — every
    bundle is eligible regardless of price, matching current behavior
    exactly). A float for entry_order_type="limit" (see caller — computed
    from p_entry, execution.entry_gap_type, execution.entry_gap_value).

    Per-bundle fill logic (from the anchor bundle onward, within
    cancel_after_seconds of entry_anchor_second):
        if bundle overlaps halt interval in halts_df → skip
        bundle_price = interpolate_bundle_price(prev_bundle, bundle, bundle.hour)
        if limit_price is not None and bundle_price > limit_price:
            → skip this bundle only (price gate — not a cancellation; a
              later bundle within the same cancel_after_seconds window may
              still qualify if price comes back down, which is common
              enough in this universe's volatility that treating a single
              price-exceeded bundle as a hard cancel would forgo real
              fill opportunities)
        per_tick_vol = bundle.volume / 10
        buyable      = floor(per_tick_vol * buy_rate)
        if buyable == 0 → skip
        filled_qty   = min(remaining, buyable)
        fill_price   = bundle_price
        total_value += fill_price * filled_qty
        total_filled += filled_qty

    Unfilled remainder after cancel_after_seconds elapsed since
    entry_anchor_second:
        order canceled — trade sized down to total_filled (not treated as
        a dead position or penalized; simply a smaller trade than intended).
        unfilled_quantity = remaining at cancellation. This is the only
        cancellation trigger — deliberately not price-based (a bundle
        exceeding limit_price is skipped, not canceled; see above).

    weighted_avg_fill_price:
        Σ(fill_price_i * qty_i) / Σ(qty_i) if total_filled > 0 else p_entry.

    Returns:
        (weighted_avg_fill_price, total_filled, unfilled_quantity, status)
        status: "filled" | "partial" | "canceled" (total_filled == 0) —
        single status regardless of whether the shortfall was volume-driven
        or price-gate-driven; not subdivided further (see Constraints).
    """
    ...
```

---

### Session Phase and Order Types

**Single source. Every submission site consults this** — entry submission,
exit submission, the stuck-exit escalation, the halt-clear replacement, and
the Unified Overnight Policy's liquidation (all in `live_mode_runner.md`).
The vendor permits order types by session phase:

| Phase (America/New_York) | limit | market |
|---|---|---|
| Premarket 04:00 – regular open | yes | no |
| Regular regular open – `session_close` | yes | yes |
| After-hours `session_close` – `after_hours_end` | yes | no |

**The boundaries are PER DATE**, resolved through the named accessors in
`utils.md`'s Session Boundary Derivations — `session_close(date)` for the
Regular/After-hours seam and `after_hours_end(date)` for the upper bound —
against the map this process loaded at entry. Which of them move, and which
stay literals, is stated there rather than here. Ordinarily this yields the
familiar 04:00 / 09:30 / 16:00 / 20:00; on an early-close day it yields
04:00 / 09:30 / 13:00 / 17:00.

WHY THIS TABLE IS THE SHARPEST EARLY-CLOSE FAILURE. Held at a fixed 16:00, the
window 13:00–16:00 on a half day reads as Regular while the venue is already in
after-hours and refuses market orders. Every consumer above is affected, and the
worst is the stuck-exit escalation: it escalates to a market order "inside the
regular session", so the final backstop would submit an order the venue rejects,
precisely when an exit is already stuck.

Phase is evaluated against America/New_York wall clock, never against the
trade stream's session marker. That rule is not specific to phase and is
stated with the boundaries themselves, in `utils.md`'s Session Boundary
Derivations.

Two consequences worth stating, because both were verified against vendor
documentation rather than assumed:

- **A market BUY is submitted as a limit** at a price unfavourable to the
  buyer, with the reference price set high, so orderable quantity is lower
  than for an equivalent limit order. The vendor sets that reference by its
  own undisclosed criterion, so the number cannot be read from the response
  before submission — which leaves the funds check sizing against a quantity
  the spec did not state. It is therefore TWO-STAGE in live mode. Stage 1, local:
  `check_funds_available()` sizes against the observed `ask1` times
  `live_mode.market_buy_price_margin`, an observable bound replacing an
  unquantified one — and it costs no extra API call, since
  `signal_time_rest` already queries level-1 bid/ask at every entry signal.
  `simulate_entry_fill()` does NOT read that key and cannot: it receives
  `ticks_entry` and `p_entry`, never a book, so BacktestEngine
  approximates the conversion from bar data instead. The two therefore do not
  size identically, and how far apart they are is unquantified — an open item,
  entered alongside the other bid/ask asymmetries. Stage 2, authority: the vendor's own insufficient-funds refusal is
  the gate's real outcome, which the entry-side rejection path already
  anticipates (`live_mode_runner.md`). The margin is set GENEROUS
  deliberately — the same one-sided-error posture this spec set takes
  wherever a bound stands in for an unknown. Too large costs an occasionally
  under-sized entry; too small costs a vendor rejection that is already
  handled.
- **A market SELL is not converted.** The vendor states the conversion for
  buys only and its reason is buy-specific, so the exit-side escalation's
  final-backstop guarantee holds wherever a market order is permitted at
  all.

### Exit Limit Pricing

A limit exit rests at a SPREAD POSITION, anchored at the ask:

```
price = ask1 - k * (ask1 - bid1)
```

| k | price | |
|---|---|---|
| 0.0 | ask1 | least aggressive |
| 0.5 | midpoint | `live_mode.exit_ladder_seed` |
| 1.0 | bid1 | immediately marketable |
| >1.0 | below bid1 | sweeps deeper resting bids |

**k is monotonic in aggression because a sell limit fills against buyers at
or above its price** — so LOWER is the aggressive direction, and anchoring
at the ask makes increasing k mean increasing aggression.

An `entry_gap_value`-style offset was rejected for this: the midpoint's
distance from the bid is proportional to the spread, which a fixed offset
from one side cannot express, and a percentage offset would read 0.5 as
50% below the bid.

The ladder that advances k while an exit is stuck, and its seed/increment/
cap keys, live under `live_mode:` rather than here — BacktestEngine has no
bid/ask model to mirror them. If backtest is later extended to replay
`bid_ask_snapshots` history, they move here for the same reason
`max_hold_bars` and `session_close_exit_time` did (the latter now
`session_close_exit_offset_minutes`).

---

## Constraints

- All functions are stateless — no module-level state or caching
- `resolve_signal()` is the single source of truth for entry signal thresholding
  — BacktestEngine and Inferencer both import from here
- `resolve_signal()` suppression check always precedes entry signal check
  — suppression takes priority regardless of upside probability magnitude
- `suppress_threshold=None` disables suppression entirely
- `can_enter()` is the single source of truth for cooldown gating — BacktestEngine
  and LiveModeRunner both import from here
- `is_tradable()` is checked before sizing, not after — an untradable ticker
  should never reach `compute_position_size()` at all
- The per-ticker terms gate (live_mode_runner.md's Per-Ticker Trading Terms)
  is evaluated at the SAME POINT but BESIDE `is_tradable()`, never folded
  into it: that function is a pure `ticker_cik_map` read, and mixing session
  state into it changes what it is. The gate is structural rather than a
  policy layered on top — a ticker with no `live_ticker_terms` row for today
  has no `mgnrt` to pass, so it cannot be sized at all
- `compute_position_size()` and `check_funds_available()` are intentionally
  separate — `balance` in sizing is fixed for the run/session; only the
  funds-availability gate uses a running/decrementing or freshly-queried value
- `per_ticker_share_cap_pct` and `exposure_cap_pct` default to 0 (disabled) —
  at default config, behavior is identical to the pre-existing cash/vol-only sizing
- `check_funds_available()`'s `use_all_cash` defaults to `True` — the default
  behavior sizes a trade down to fit remaining cash rather than skipping it
  outright; set `False` to restore the older all-or-nothing behavior
- A quantity reduced by `use_all_cash` does NOT re-enter
  `compute_position_size()`'s caps. All four sizing terms are monotone in
  quantity and combined by `min()`, and `margin_of()` is linear in quantity
  so the margin-unit form preserves that — any value at or below the
  returned `quantity` satisfies all four by construction. Sizing sets the
  ceiling; the gate only lowers it. Stated because nothing said so before
  and a reader had to derive it
- `p_entry`'s three roles SPLIT when `late_entry_enabled` is true, having
  coincided until now. As the `entry_points` primary-key component and
  detection record: t-bar open, UNCHANGED. As the label anchor in
  `labeled_samples`: t-bar open, UNCHANGED — moving it would redefine past
  labels and contaminate the training corpus. As the SIZING input: `p_now`,
  the actual expected fill. The entry-side call order below pins
  `compute_position_size(fill_price=p_entry, ...)`, so the DEFINITION of the
  value passed there changes even though the signature does not.
  `trade_log` already records the real fill and needs nothing
- Entry-side call order, LiveModeRunner ONLY (N-5 — deliberately not
  BacktestEngine; see below): `is_tradable()` + the terms gate →
  `compute_position_size(fill_price=p_entry, mgnrt=<this ticker's>, ...)` →
  `check_funds_available(quantity, p_entry, ...)` →
  `simulate_entry_fill(quantity=adjusted_quantity, p_entry=p_entry, ...)`.
  Sizing and the funds gate both use `p_entry` (always known immediately,
  before any fill simulation), not the slippage-adjusted fill price —
  `simulate_entry_fill()` is what actually determines the realized fill
  price, and it needs a quantity as *input*, so sizing cannot wait for it.
  This mirrors `data_boundary.md`'s existing use of `p_entry` as the
  "Backtest fill price reference" role.
- `is_tradable()` is intentionally NOT part of BacktestEngine's entry-side
  flow (N-5), even though it is Sizing→Funds→Fill's first step for
  LiveModeRunner above. `ticker_cik_map`'s rename_pending/quarantine_reason
  reflect data-pipeline/vendor-latency state AS OF NOW, not as of whatever
  historical date a backtest run happens to be simulating — there is no
  date-scoping on that table (see db_schema.md's ticker_cik_map). Calling
  `is_tradable()` from BacktestEngine would therefore make backtest results
  depend on the CURRENT contents of a table unrelated to the dates being
  simulated (non-deterministic re-runs of the "same" backtest, and
  systematic contamination of any backtest window that overlaps a
  currently-suspended ticker — most relevant for P-12's live-vs-backtest
  divergence check, since "recent" and "current" nearly coincide there).
  More fundamentally: both suspend reasons exist to hedge LIVE, real-time
  uncertainty about information a backtest already has resolved by the
  time anyone runs it (was this really a rename? was this really an
  unconfirmed corporate event?) — by backtest time by definition the
  answer is known, so the hedge has nothing left to hedge against. This is
  not a coverage gap to close with better date-scoping; the check does not
  belong in a hindsight-informed simulation at all. See
  09_backtest_engine.md's Constraints for the corresponding removal from
  that module's sourced-function list.
- `simulate_exit_fill()`'s contract is a TICK-LEVEL FILL MODEL OVER AN
  ANCHORED WINDOW. It was previously written as "called after
  `utils.track_price_breach()` confirms direction", which bound the function
  to the tp/sl path and is what excluded `time_limit` and `session_end` from
  fill simulation at all — residue from that binding, not a modelling
  decision: no rationale for the exclusion appears anywhere in the spec set.
  Direction confirmation is one caller's way of producing an anchor.
  sell_rate selection by exit reason remains the caller's responsibility
- `simulate_entry_fill()` mirrors `simulate_exit_fill()`'s per-bundle structure;
  unlike the exit side, an unfilled remainder is canceled (order timeout) rather
  than penalized
- `simulate_exit_fill()` is called by BacktestEngine's Exit Logic and by live
  mode's `stage == "shadow"` branch; `simulate_entry_fill()` is called by
  BacktestEngine's Entry Slippage Model and by live mode's `stage ==
  "shadow"` branch. In live those calls are REAL-PATH-PARALLEL INCREMENTAL,
  never inline: both consume ticks forward from the anchor, which do not
  exist at the decision instant, so live opens a pending window and
  recomputes statelessly from the anchor until settlement
  (live_mode_runner.md). Neither is called by live mode's real order path (real fills
  come from the trading API; for `entry_order_type="limit"`, the real path
  instead tracks the order via LiveModeRunner's Position Manager Loop
  `pending_entries` mechanism until filled or timed out — see
  live_mode_runner.md)
- `simulate_entry_fill()`'s price gate (a bundle priced above `limit_price`
  is skipped, not canceled) is deliberate — this universe's volatility
  means a single bundle exceeding the limit is not reliably a sign the
  price won't come back within the same `cancel_after_seconds` window;
  `status` is therefore not subdivided by shortfall cause (volume vs
  price) — a single `"canceled"` covers both
- `get_execution_param()` (N-7) is the sole read point for buy_rate,
  sell_rate_tp, sell_rate_sl, sell_rate_neutral, cancel_after_seconds —
  BacktestEngine and
  LiveModeRunner must not query `execution_params` directly, for the same
  reason they must not duplicate `resolve_signal()`/`can_enter()`. Its
  hard-bound fallback is deliberately not configurable (mathematically
  derived from the fill-simulation formulas, not a judgment call) and is
  a distinct, independent mechanism from `fit_execution_params()`'s
  relative-bound rejection (see shadow_retraining.md) — the latter guards
  what gets written, this guards what gets read, and neither substitutes
  for the other.
