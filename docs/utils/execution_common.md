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

---

### Tradability Gate

```python
def is_tradable(ticker: str, db_conn) -> bool:
    """
    Single gate for "should we allow a NEW entry into this ticker right
    now". Checked once, at the point EntryPointDetector confirms a
    candidate, before sizing/order submission.

    Currently checks only ticker_cik_map.status (rename-pending-confirmation
    case — see db_schema.md, metadata_crawler.md). Deliberately does NOT
    check tick-rate/staleness/halt — EntryPointDetector's own volume-gated
    filters (B/C/D — see 01_entry_detection.md) already fail on a
    genuinely stale or halted ticker, so a separate check here would be
    redundant (see the former open_items_production_readiness.md P-6
    countermeasure 3 discussion).

    Returns:
        True if the ticker is tradable (status != 'suspended', or no
        ticker_cik_map row exists at all — an unmapped ticker is not a
        rename-ambiguity case).
    """
    row = db_conn.execute(
        "SELECT status FROM ticker_cik_map WHERE ticker = ? "
        "ORDER BY last_seen_date DESC LIMIT 1", [ticker]
    ).fetchone()
    return row is None or row[0] != 'suspended'
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
                             ticker's currently-open positions
        total_notional:      sum(entry_price_i * quantity_i) across all
                             currently-open positions (any ticker)
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

    backtest: available_cash is the sequentially-decremented remaining_cash
              tracked by the caller across the run.
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
  # only backtest-simulation-specific keys (initial_cash, max_hold_bars,
  # entry_cooldown_minutes, entry_threshold, suppress_threshold,
  # dead_position_penalty_pct) — see 09_backtest_engine.md.
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
- Entry-side call order (both BacktestEngine and LiveModeRunner):
  `is_tradable()` → `compute_position_size(fill_price=p_entry, ...)` →
  `check_funds_available(quantity, p_entry, ...)` →
  `simulate_entry_fill(quantity=adjusted_quantity, p_entry=p_entry, ...)`.
  Sizing and the funds gate both use `p_entry` (always known immediately,
  before any fill simulation), not the slippage-adjusted fill price —
  `simulate_entry_fill()` is what actually determines the realized fill
  price, and it needs a quantity as *input*, so sizing cannot wait for it.
  This mirrors `data_boundary.md`'s existing use of `p_entry` as the
  "Backtest fill price reference" role.
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
