# Playbook — Lending, Interest & Liquidation

Load when the target has borrow/repay, collateral, health factors, interest accrual, or
liquidation. Read the "known-good patterns" section at the end **before reporting** — the
zero-trust posture over-reports here in ways that would make a protocol *less* safe if acted on.

## Interest & accrual ordering

- **Settle interest before any parameter change.** Every setter touching an interest-rate
  model, reward rate, or exchange-rate parameter must flush accrued state at the *old* rate
  first; otherwise the new rate applies retroactively across the whole elapsed period. Fast
  tell: **a setter with no `Clock` parameter physically cannot accrue** — grep `fun set_.*rate`,
  `fun update_.*model`.
- **`accrue_interest` is the first op in the health path.** A pre-accrual health factor
  understates debt and makes an underwater position look healthy. Also verify oracle reads in
  the health path carry staleness/confidence checks and correct per-asset decimal normalization.
- **Refinance / index-reset skipping.** Borrow-then-refinance in one PTB can reset the
  position's `interest_index` without settling what was owed. Capitalize accrued interest
  before the index moves; require a cooldown; verify `accrue_interest` is idempotent within one
  timestamp.
- **Abort-before-checkpoint deadlock.** If overflow-prone arithmetic runs *before*
  `last_update_time`/index is written, every retry grows the delta and the pool freezes
  permanently — including admin-cancel paths. Never dismiss overflow in an accrual function as
  "just a DoS abort." (See verification-and-false-positives.md — overflow carve-out.)

## Solvency, health & withdrawal gating

- **Solvency evaluated on post-operation state.** Withdraw and borrow must check health on the
  state *after* the mutation.
- **Withdraw gated by the borrow threshold, not the liquidation threshold.** Gating on the
  looser liquidation threshold lets users park at exactly HF = 1.0, so any tick produces bad
  debt with no liquidation buffer.
- **Post-liquidation health must strictly improve.** The bonus can remove more collateral
  *value* than debt value and leave HF *lower*. Assert `hf_after > hf_before` (or full closure).
- **Health factor includes all components.** Accrued interest, pending rewards, and unrealized
  PnL omitted from the computation cause premature liquidations.
- **Sequential-threshold trap.** Two stacked health gates where the stricter one blocks entry
  (`hf < 9500` then `hf < 10000`) leaves positions in 0.95–1.0 unhealthy but unliquidatable;
  bad debt accrues silently. Liquidation entry should have exactly one gate at the true
  threshold; stricter values belong in *how much* (close factor), not *whether*.

## Close factor & per-transaction limits

- **Close factor enforced per transaction, not per call.** A PTB chains N partial
  liquidations; if each recomputes the cap against the *remaining* debt, a 50% close factor
  becomes 87.5% at N=3 and ~97% at N=5. Reference a debt snapshot taken at the first
  liquidation in the transaction. **Generalize: any per-call numeric limit** (rate limits,
  withdrawal caps, claim caps, cooldowns) is void unless tracked per-transaction.
- **Config updates that clobber runtime state.** An admin setter replacing a whole config
  struct also resets embedded accumulators/counters/timestamps/rate-limiter segments. Classify
  every written field as config vs. runtime. Attack: consume a rate limit → front-run the
  admin's update to reset it → consume it again.
- **Rolling limiter reduce/count asymmetry.** Segmented outflow limiters where `count` sums all
  live segments but `reduce` only touches `now % len` silently discard cross-boundary unwinds,
  blocking legitimate borrows/withdrawals for a full window. Boundary test: add L at
  `t = dur-1`, reduce L at `t = dur+1`, assert usage returns to 0.

## Liquidation must not be blockable, and must close economically

- **Enumerate everything that can make liquidation abort:** pause flags, cooldowns, pending
  withdrawal reservations, regulated-coin freezes on the transfer leg, unbounded loops over
  collateral lists, and — critically — insufficient idle cash when the path redeems collateral
  to underlying in the same tx. That last one fails *precisely at high utilization*, i.e.
  exactly when liquidation is needed → borrower holds utilization high to stay unliquidatable.
- **Liquidation economics must close.** Verify `bonus − protocol_fee − swap_fee/price_impact −
  gas > 0`. A bonus eaten by fees or by the cost of offloading illiquid collateral means no one
  liquidates. Illiquid collateral warrants a larger bonus.
- **Minimum position size.** Without one, dust positions cost more gas to liquidate than the
  bonus yields → permanent bad debt. Enforce at open, at partial repay (no sub-minimum
  remainder), and at partial liquidation.
- **Check-vs-settlement price-basis divergence.** If the liquidation *decision* uses one basis
  (mark/EMA/oracle median) and *settlement* uses another (tick-rounded/last-trade/index) — or
  differs by a fee or decimal scale — a hard `assert!` tying the two can fire for near-zero-
  equity positions and revert the whole liquidation. Keepers retry identical inputs → the
  position becomes permanently unliquidatable. Diff the price/rounding/unit/fee inputs of the
  gate vs. the settlement.

## Pause, bad debt & seizable collateral

- **Pause symmetry.** If repay/add-collateral is pausable but liquidation is not, paused
  borrowers are force-liquidated while helpless; and if interest accrues through a repay-blocked
  pause, healthy positions get mass-liquidated at unpause. Trace every pause-gated function and
  confirm repay / deposit-collateral / liquidate are treated consistently.
- **Repay-on-behalf path.** If any collateral is a regulated/freezable coin, a borrower blocked
  from transferring is forced into liquidation through no fault of their own. A third-party
  `repay_on_behalf` is the mitigation; an escrow fallback covers the seizure leg.
- **Bad-debt absorption path exists.** When collateral hits 0 with residual debt, something
  must write it off (insurance fund / socialized loss). Silent residual debt is protocol
  insolvency accruing quietly.
- **Pending withdrawal reduces seizable collateral.** Liquidation must override/cancel pending
  ops, not net against them (else collateral nets to ~0 and the position is unliquidatable).
- **Debt structs must not be transferable.** A debt-representing struct with `store` can be
  `public_transfer`ed onto an unwilling victim — debt structs should be `key`-only and bound to
  the borrower.

## Self-liquidation & self-match

- **Self-liquidation profitability.** No `liquidator != borrower` assert and `bonus >
  liquidation_fee` → borrowers profitably liquidate themselves.
- **Self-match protection compares owner addresses, not account IDs.** One owner holding two
  accounts crosses their unhealthy margin account against their clean one at the worst
  oracle-allowed price, bleeding value out; the shortfall becomes bad debt.
- **Margin/orderbook proxy skips post-trade health.** A proxy validating pool identity and
  price bounds but not rechecking account health *after* the trade lets a liquidatable account
  keep placing loss-making trades. Compare against borrow/withdraw, which usually do check.
- **Emergency mechanism entry/stop read the same metric.** ADL/circuit-breakers with an
  activation reading `reserve.total_debt` and a deactivation reading `emode_group.borrow_amount`
  fire on healthy groups and never stand down.

## Known-good patterns — DO NOT report these as bugs (false-positive suppressors)

Each is safe *by design*; only the stated condition makes it a real finding:

- **Spot price for the seize calculation alongside EMA/TWAP for eligibility** is *intentional* —
  EMA-priced seizes underpay liquidators and cause bad debt. Bug only if seize uses a
  manipulable price with no bound.
- **Flash loans deliberately not decrementing `cash`** is correct when a non-droppable receipt
  guarantees repayment within the PTB.
- **Blocking new borrows when cash falls below the reserve ratio** is protective, not a DoS.
- **Sanity gate before filing any liquidation finding:** would the proposed fix keep liquidation
  profitable at spot? If the fix makes liquidation unprofitable, it *causes* bad debt and is
  worse than the bug. Scale severity to economics — over-seizing by 0.1% is Low; blocking
  liquidation entirely is High.
