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
    ticker, so a separate check here would be redundant (see the former
    open_items_production_readiness.md P-6 countermeasure 3 discussion).

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
    t_bar_volume: int,
    ticker_notional: float,
    total_notional: float,
    position_size_cash_pct: float,
    position_size_vol_pct: float,
    per_ticker_share_cap_pct: float,
    exposure_cap_pct: float,
) -> int:
    """
    Compute per-trade buy quantity. Shared by BacktestEngine and
    LiveModeRunner — same formula, different `balance` source:
        backtest: initial_cash (config, fixed for the whole run)
        live:     session_start_cash (queried once from the trading API at
                  session start, fixed for the session)
    `balance` is deliberately NOT a running/decrementing value — see
    Constraints below for why, and see the separate funds-availability gate.

    cash_based = floor((balance * position_size_cash_pct) / fill_price)
    vol_based  = floor(t_bar_volume * position_size_vol_pct)

    per_ticker_share_cap_pct == 0 → ticker_cap_qty = inf (uncapped)
    else:
        ticker_cap_qty = floor(
            max(0, per_ticker_share_cap_pct * balance - ticker_notional)
            / fill_price
        )

    exposure_cap_pct == 0 → exposure_cap_qty = inf (uncapped)
    else:
        exposure_cap_qty = floor(
            max(0, exposure_cap_pct * balance - total_notional)
            / fill_price
        )

    quantity = min(cash_based, vol_based, ticker_cap_qty, exposure_cap_qty)

    Args:
        balance:            fixed reference capital (see above) — not a
                             running/decrementing value
        fill_price:          slippage-adjusted entry price
        t_bar_volume:        ohlcv_1min volume of the t bar
        ticker_notional:     sum(entry_price_i * quantity_i) across this
                             ticker's currently-open positions, PLUS its
                             submitted-but-unfilled ones (R-5: live
                             live_positions status 'pending'/'partial_open'
                             — priced at limit_price, or p_entry for a
                             market order). Pending notional is reserved
                             capacity: excluding it lets a burst of
                             simultaneous submissions breach the cap on
                             aggregate fill, which is inert only while
                             per_ticker_share_cap_pct is 0 and becomes real
                             the moment Pilot sets it. BacktestEngine has no
                             pending state, so there the two sums coincide.
        total_notional:      the same sum across all positions, any ticker,
                             on the same open-plus-pending basis
        position_size_cash_pct:   fraction of balance per trade (cash leg)
        position_size_vol_pct:    fraction of t bar volume per trade
                             (liquidity leg)
        per_ticker_share_cap_pct: 0 = disabled; >0 = cap on this ticker's
                             cumulative notional as a fraction of balance
        exposure_cap_pct:    0 = disabled; >0 = cap on total notional
                             across all open positions as a fraction of
                             balance

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
    live:     available_cash is queried fresh from the trading API
              immediately before this check.

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
  max_hold_bars:             60    # R-6: single source. Backtest's
                                   # time-limit exit and live's
                                   # restart_gap_exit / overnight_exit cutoff
                                   # both read this one key.
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
  session_close_exit_time: "155900"  # R-7: single source. Was declared under
                                   # live_mode: while 09_backtest_engine.md
                                   # carried the bare literal — the same
                                   # one-sided declaration of a value both
                                   # engines must agree on that R-6 fixed for
                                   # max_hold_bars.
  ws_ticker_limit:           50    # R-7: tickers one WS sequence can carry.
                                   # A VENDOR fact, not a preference — kept in
                                   # config rather than as a literal because
                                   # it varies by broker and is measured, not
                                   # chosen. See api_contract_checklist.md,
                                   # which names this key as where its
                                   # measurement is recorded. max_tickers: 0
                                   # (unlimited) clamps to this in live.
  sizing_basis:          "equity"  # R-8: "equity" | "buying_power". LIVE ONLY
                                   # — backtest has no margin and no
                                   # margin_ratio to query, so BacktestEngine
                                   # ignores this key and always sizes on
                                   # initial_cash directly.
                                   # "equity": divide the broker's reported
                                   # balance by the session's margin_ratio
                                   # before passing it as compute_position_size
                                   # ()'s `balance`, so "fully deployed" means
                                   # 100% of OWN capital.
                                   # Default is equity because the failure is
                                   # one-sided: what the balance endpoint
                                   # returns is still unverified
                                   # (api_contract_checklist.md), and reading
                                   # buying power as if it were cash silently
                                   # multiplies intended exposure by the
                                   # leverage factor.
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
  buy_rate:                    0.1  # seed value — conservative starting point,
                                    # refined by fit_execution_params() once
                                    # pilot-stage predicted-vs-actual data
                                    # accumulates (see shadow_retraining.md)
  cancel_after_seconds:        30   # seed value — same refinement path as buy_rate
  entry_order_type:      "market"  # "market" | "limit"
  entry_gap_type:    "percentage"  # "percentage" | "absolute" — limit only
  entry_gap_value:            0.0  # limit only. 0 = limit at p_entry exactly;
                                    # positive = willing to pay up to that much
                                    # more. percentage: p_entry*(1+value);
                                    # absolute: p_entry+value
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
    cancel_after_seconds — BacktestEngine and LiveModeRunner both call this
    rather than querying execution_params directly, so the read-time
    hard-bound defense below exists in exactly one place (same
    single-source-of-truth pattern as resolve_signal(), can_enter()).

    1. row = latest execution_params row for param_name, if any
       (see db_schema.md's execution_params)
    2. value = row.value if row exists else config["execution"][param_name]
       (the seed default)
    3. Hard-bound check (N-7) — mathematically derived, not configurable:
           buy_rate, sell_rate_tp, sell_rate_sl: valid range is (0, 1].
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
    ohlcv_exit: pd.DataFrame,
    position_size: int,
    breach_bundle_idx: int,
    breach_price: float,
    sell_rate: float,
    halts_df: pd.DataFrame,
) -> tuple[float, int, int, str]:
    """
    Simulate partial exit fills across tick bundles from the breach point onward.
    Continues through session close into after-market until position is fully
    closed or all ticks are exhausted.

    Caller selects sell_rate based on exit direction:
        sell_rate_tp for take-profit; sell_rate_sl for stop-loss.

    Per-bundle fill logic (from breach_bundle_idx onward):
        if bundle overlaps halt interval in halts_df → skip
        per_tick_vol = bundle.volume / 10
        sellable     = floor(per_tick_vol * sell_rate)
        if sellable == 0 → skip
        filled_qty   = min(remaining, sellable)
        fill_price   = interpolate_bundle_price(prev_bundle, bundle, bundle.hour)
        total_value += fill_price * filled_qty
        total_filled += filled_qty

    Breach bundle handling:
        fill_price = breach_price (computed by caller via interpolate_bundle_price).
        sellable   = floor((breach_bundle.volume / 10) * sell_rate)
        if sellable == 0 → skip breach bundle, start from breach_bundle_idx + 1.

    Session close: no forced liquidation; after-market ticks processed identically.
    Ticks exhausted with remaining > 0: unfilled_quantity = remaining.

    weighted_avg_exit_price:
        Σ(fill_price_i * qty_i) / Σ(qty_i) if total_filled > 0 else breach_price.

    Note: this per-bundle fill price does not include an explicit
    market-impact term — it assumes participation at sell_rate does not
    materially move the observed price. This approximation is expected to
    degrade as sell_rate increases; the point at which impact becomes
    material is unknown at design time and should be estimated from
    realized-vs-simulated fill data during shadow/pilot calibration (see
    shadow_retraining.md's fit_execution_params() — sell_rate_tp/sell_rate_sl
    are refit there alongside buy_rate/cancel_after_seconds).
    """
    ...
```

---

### Entry Fill Simulation

```python
def simulate_entry_fill(
    ticks_entry: pd.DataFrame,
    ohlcv_entry: pd.DataFrame,
    quantity: int,
    fill_bundle_idx: int,
    p_entry: float,
    buy_rate: float,
    halts_df: pd.DataFrame,
    cancel_after_seconds: int,
    limit_price: float | None,
) -> tuple[float, int, int, str]:
    """
    Simulate partial entry fills across tick bundles from t+5s onward.
    Mirrors simulate_exit_fill() for the entry side — same per-bundle
    participation-rate structure, applied to a buy instead of a sell.

    limit_price: None for entry_order_type="market" (no price gate — every
    bundle is eligible regardless of price, matching current behavior
    exactly). A float for entry_order_type="limit" (see caller — computed
    from p_entry, execution.entry_gap_type, execution.entry_gap_value).

    Per-bundle fill logic (from fill_bundle_idx onward, within
    cancel_after_seconds of t+5s):
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

    Unfilled remainder after cancel_after_seconds elapsed since t+5s:
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
- `compute_position_size()` and `check_funds_available()` are intentionally
  separate — `balance` in sizing is fixed for the run/session; only the
  funds-availability gate uses a running/decrementing or freshly-queried value
- `per_ticker_share_cap_pct` and `exposure_cap_pct` default to 0 (disabled) —
  at default config, behavior is identical to the pre-existing cash/vol-only sizing
- `check_funds_available()`'s `use_all_cash` defaults to `True` — the default
  behavior sizes a trade down to fit remaining cash rather than skipping it
  outright; set `False` to restore the older all-or-nothing behavior
- Entry-side call order, LiveModeRunner ONLY (N-5 — deliberately not
  BacktestEngine; see below): `is_tradable()` →
  `compute_position_size(fill_price=p_entry, ...)` →
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
- `simulate_exit_fill()` is called after `utils.track_price_breach()` confirms
  direction; sell_rate selection (tp vs sl) is the caller's responsibility
- `simulate_entry_fill()` mirrors `simulate_exit_fill()`'s per-bundle structure;
  unlike the exit side, an unfilled remainder is canceled (order timeout) rather
  than penalized
- `simulate_exit_fill()` is called by BacktestEngine's Exit Logic and by live
  mode's shadow-mode branch; `simulate_entry_fill()` is called by
  BacktestEngine's Entry Slippage Model and by live mode's shadow-mode
  branch — neither is called by live mode's real order path (real fills
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
  sell_rate_tp, sell_rate_sl, cancel_after_seconds — BacktestEngine and
  LiveModeRunner must not query `execution_params` directly, for the same
  reason they must not duplicate `resolve_signal()`/`can_enter()`. Its
  hard-bound fallback is deliberately not configurable (mathematically
  derived from the fill-simulation formulas, not a judgment call) and is
  a distinct, independent mechanism from `fit_execution_params()`'s
  relative-bound rejection (see shadow_retraining.md) — the latter guards
  what gets written, this guards what gets read, and neither substitutes
  for the other.
