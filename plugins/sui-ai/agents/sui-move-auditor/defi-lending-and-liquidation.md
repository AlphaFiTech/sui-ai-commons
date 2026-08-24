# Playbook — Lending, Interest & Liquidation

Load when the target has borrow/repay, collateral, health factors, interest accrual, or
liquidation. Read the "known-good patterns" section at the end **before reporting** — the
zero-trust posture over-reports here in ways that would make a protocol *less* safe if acted on.

## Market creation & configuration validation

- **Formula-derived ceilings, not bare non-zero checks.** Market creation and every config
  setter must satisfy the solvency algebra — e.g. `liq_threshold · (1 + bonus + fee) < 1`, so a
  liquidation can actually restore health. `> 0` checks alone admit parameter sets under which
  every liquidation deepens insolvency.
- **LTV strictly below the liquidation threshold, with a gap.** Equal values park positions at
  exactly HF = 1.0 with no buffer; the first adverse tick makes them instantly liquidatable.
- **Cross-parameter consistency.** Enforce `max ≥ min` for every min/max pair; an inverted pair
  silently disables the limit or aborts every path that reads it.
- **Empty-config bricking guard.** A market creatable with an empty config that the normal
  setter path cannot then repair is bricked at birth — reject at creation, or verify the
  setters can reach every field from that state.
- **`window > 0` — and a positive floor on every divisor/duration parameter.** A zero slips
  past creation and freezes the first runtime path that divides by it.

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

## Oracle staleness & price representability

- **Same-PTB cache-warming/refresh reachability.** If reads require a fresh cache entry, the
  refresh must be reachable in the same PTB as the consuming call — including the
  empty-cache-at-birth case: a brand-new market with no cache entry yet must be servable, not
  bricked until an out-of-band refresh.
- **Saturating staleness math.** `now - updated_at` must not underflow on clock skew or
  future-dated inputs; an abort here takes down every read path at once.
- **Future-dated timestamp rejection.** A timestamp ahead of the chain clock must be rejected —
  accepted, a far-future entry never goes stale and freezes refresh logic.
- **Zero/near-zero price representability cliff.** A price that floors to zero in the protocol's
  representation makes a position simultaneously "liquidatable" and "impossible to liquidate"
  (its collateral values to nothing); analyze the representation's minimum against realistic
  prices for the smallest-priced supported asset.
- **No unpriced-asset $1 assumption.** An asset without a feed must never be silently assumed
  to have a fixed valuation; abort or exclude it explicitly.
- **`10^decimals` normalization in every USD × raw-unit comparison** — and one canonical price
  representation across health, seizure, and settlement reads. A basis mismatch between any two
  of them mis-sizes liquidations by orders of magnitude.

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

## Position lifecycle: zero-amount, zombie counters & terminal states

- **Zero-amount open/re-activate must abort.** Zero-amount records are free to mint yet count
  like real positions — the griefing primitive against every collective gate. Assert non-zero
  at open and on any path that re-activates a closed record.
- **Full close via the partial path must not leave a zombie counter.** If a position can reach
  zero through the partial-reduce path, open-position counters/registries must be decremented
  exactly as on the dedicated close path; a zombie counter blocks every gate that requires "no
  open positions" — wind-down, migration, final settlement.
- **No post-terminal-state face-value acceptance.** After a terminal state (expiry, settlement,
  write-down), repay/deposit/collateral paths must not accept the instrument at face value —
  that converts a recorded loss into a live exploit.
- **Destructive close-out requires independently verified zero balances.** Object deletion or
  coin burn on cleanup must check each contained balance is zero via its own read; inferring
  emptiness from a status flag or counter deletes whatever the flag missed.

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

## Multi-step admin recovery / force-close flows

For any admin flow spanning multiple transactions (start → per-type/per-account act → finish),
require all of the following by design:

- **Monotonic covered-counter fixed at flow start.** Each step must size its action against a
  snapshot taken when the flow started — never against a mutable field that other steps or
  concurrent user actions also move.
- **Explicit re-entry/concurrency guards.** Starting a second concurrent flow instance, or
  re-running a completed step, must be explicitly rejected by a flow-state check; a flow must
  never rely on incidental native aborts (arithmetic underflow, duplicate dynamic-field key) as
  its only protection — those are accidents of a particular implementation that any refactor
  can remove.
- **User paths guarded in both directions mid-flow.** Deposits AND withdrawals into the
  affected scope must be blocked (or explicitly reconciled) while the flow is in progress;
  guarding one direction only is the classic mirror-image regression.
- **Snapshot/close-out ordered after the covering deposit.** A step that snapshots balances for
  a later distribution must run after the deposit that funds it, never before.
- **Write-off requires a collateral-exhaustion check.** A bad-debt write-off step must verify
  seizable collateral is actually exhausted first — otherwise it socializes a loss that
  liquidation could still have covered.

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
- **Enumerate every fund-outflow entry point against the pause gate.** Withdraw, redeem, claim,
  sweep, fee-collect — confirm each one is individually covered by the pause gate. Per-function
  enumeration, not a sampled check: the unlisted path is the one that drains during an incident.
- **Pause must not asymmetrically consume user-facing time windows.** If a pause burns down a
  user's time window (e.g. a redemption window) while admin-favoring deadlines survive it
  intact, users exit the pause with rights expired and obligations alive. Check every deadline
  the protocol tracks against this invariant.
- **Sweep/fee destination is a fixed protocol address.** Sweep and fee-collection paths must pay
  a protocol-configured destination, never the calling cap holder — "whoever calls collects"
  turns a routine operational capability into a drain.
- **Three-question circuit-breaker check.** (i) Does the freeze also block fund *rescue* paths
  (repay, add-collateral, cancel) — i.e. is the blast radius asymmetric against users? (ii) Is
  the trigger capability unique — and if not, does the pause event identify the triggering
  holder? (iii) Does a single shared pause object serialize every market — a consensus
  bottleneck where one switch freezes all?

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
