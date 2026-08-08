# Playbook — Staking & Reward Accounting

Load when the target distributes rewards over a stake/deposit base (`reward_per_share`,
`acc_reward`, `reward_debt`, `cumulative_index`, staking pools, liquid staking). The single
highest-yield finding class here is the uninitialized checkpoint — read it first.

## Checkpoint & accumulator ordering (the core bugs)

- **Uninitialized per-user checkpoint = full historical reward credit.** Per-user positions
  hold a checkpoint (`last_index` / `reward_debt` / `reward_per_share_paid`). If the constructor
  leaves it at `0` while the pool's monotonically-growing accumulator sits at N, the first
  reward computation credits `N − 0` — potentially the entire reward pool. Usually masked
  because the SDK calls a sync setter right after construction, so it only surfaces when someone
  calls the constructor and claim path *directly*.
  **Sui amplifier:** every historical package version stays callable forever, so a constructor
  buggy in v1 and fixed in v2 remains exploitable against the live shared pool unless a version
  gate exists. Detection: grep checkpoint field names → read every constructor's initializer →
  map the call graph from each constructor to each reward-update function → check *every*
  package version.
- **Accumulator update precedes every balance change.** Safe order: bump the global index →
  settle the user's pending against their *old* balance → advance their snapshot → then change
  the balance. Any other order lets a new deposit earn historical rewards or a withdrawal
  forfeit earned ones. Multi-reward-token pools must settle *every* accumulator before any
  balance touch. Test: stake, accrue, stake more, claim immediately — payout must cover only the
  first tranche for the earlier period.
- **Every path modifying rate or total stake flushes the index first.** `add_rewards`,
  `set_reward_rate`, and similar admin functions are the usual offenders — user-facing paths
  call the update and the admin one forgets. Tell: the admin function has no `Clock` parameter.

## Reward source & precision

- **Rewards derived from a dedicated reward balance, not total pool balance.** Computing
  `new_rewards = balance::value(&pool.balance) − total_staked` conflates stake with rewards, so
  any balance-increasing path (a stray `balance::join`) distorts distribution. Require separate
  `Balance` fields and an explicit `add_rewards`.
- **Accumulator precision.** These patterns were designed around `uint256`; Move's `u64`/`u128`
  is far smaller. Accumulators should be `u128`, scaled by a large `PRECISION` (≥ 1e12),
  multiplying before dividing — otherwise the per-share increment truncates to `0` when total
  stake is large relative to the reward, silently destroying rewards for everyone.

## Flash-stake & timing

- **Flash stake via PTB / minimum stake duration.** A PTB can atomically flash-borrow → stake →
  trigger distribution → unstake → repay. Criterion for when a lock is actually needed: a
  *continuous* accumulator yields zero for a same-checkpoint stake/unstake *by construction* and
  needs no lock; a *discrete* distribution event is exploitable and requires either a minimum
  lock checked at unstake against the receipt's stake timestamp, or an eligibility snapshot taken
  before the distribution is executable. Classify the mechanism first.
- **Checkpoint timestamp granularity.** `clock::timestamp_ms` advances per *checkpoint*
  (~0.5–2s), not per transaction. Multiple txs in one checkpoint observe an identical timestamp,
  so any "time-weighted" defense with sub-checkpoint resolution sees zero elapsed time and is not
  a defense.
- **Flash-loan-triggered global cooldown DoS.** A debounce keyed *globally* (not per-user) on a
  permissionless function can be tripped with borrowed capital, blocking every legitimate caller
  for the window. Escalate if there is no admin reset or expiry.
- **Stale cached balance across mutating operations.** Caching `balance::value` early and
  reusing it after an operation that changed the balance (compounding) inflates the share
  computation. Prefer the return values of mutating operations over pre-read snapshots.

## Economic design & liquid staking

- **Emission sustainability horizon.** Project max emission over a day/week/year against what
  actually backs it. Can an epoch-based reward pool drain faster than it is replenished? Is any
  supply cap bypassable by parameter change rather than enforced by `TreasuryCap`?
- **Retroactive fee/rate changes on unclaimed earnings.** Raising a protocol fee without first
  settling pending rewards charges the new rate against earnings accrued under the old one.
  Settle first; consider a timelock so users can claim ahead of the change.
- **Liquid-staking receipt supply cap & conservation.** Verify receipt minting is
  capability-gated, `split`/`merge` conserves total value (`split` outputs sum to input),
  `total_issued <= total_stake` is enforced, and receipts carrying `store` cannot reach paths
  that bypass redemption.
- **Validator commission drift.** A protocol delegating on users' behalf should bound commission
  at delegation time *and* re-check at each reward epoch — a validator can raise commission after
  receiving the delegation and silently tax the yield. Check for auto-redelegation away from
  over-threshold validators.
