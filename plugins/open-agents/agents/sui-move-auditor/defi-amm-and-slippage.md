# Playbook — AMM, Slippage & Market Manipulation

Load when the target is a DEX, router, CLMM, aggregator, or any function that swaps,
adds/removes liquidity, or reads a pool's spot state to make a decision. This is the
largest single gap for a DeFi audit — slippage protection is routinely present on swaps
and forgotten everywhere else.

## Slippage protection — presence and correctness

- **`min_amount_out` present and enforced.** Every swap/trade/exchange must take a
  caller-supplied minimum output and `assert!` it against the *actual post-fee* output. A
  missing parameter, or one defaulted/allowed to be `0`, lets a sandwich take essentially the
  whole trade.
- **Self-referential slippage (the subtle one).** Deriving `min_out` by quoting the *same
  pool* the swap executes against gives **zero** protection — the attacker moves reserves
  first and the "expected" value moves with it. The bound must come from the caller (off-chain)
  or an independent oracle. Flag any "auto-slippage" convenience feature. Check: does one
  function both `quote()` and `do_swap()` against the same pool object?
- **Slippage on LP add/remove, not just swaps.** `add_liquidity` needs `min_lp_out`;
  `remove_liquidity` needs `min_a` + `min_b`. LP mint/burn is sandwichable by skewing the pool
  ratio first; single-sided deposits are the worst case. Most commonly missed surface.
- **Multi-hop protects the *final* output.** A `min_b` checked after hop 1 with nothing on
  `coin_c` is false assurance. The assert must be on the last coin produced.
- **Per-boundary slippage in PTB-composable functions.** A composite entry point
  (`swap_and_deposit`) must enforce its own bound at *each* value-transforming step — "the
  caller will check" is no defense when the caller is the attacker.
- **Keeper/admin paths that spend protocol funds.** Harvest, compound, rebalance, and
  liquidation routines swap *protocol* money and are the most common place to find a hardcoded
  slippage constant or a literal `0` minimum. Scrutinize harder than user-facing swaps. Grep
  `const .*SLIP`.
- **Hardcoded slippage constants.** A fixed tolerance is simultaneously too loose for stable
  pairs and too tight in volatility (stuck withdrawals). Also verify a caller-supplied
  tolerance cannot be set to 100%.
- **Deadline.** Move has no implicit tx expiry. Time-sensitive ops should take `deadline_ms`
  and check it against `clock::timestamp_ms` **at function entry, before any state change** —
  otherwise a held tx executes at the worst price the min-out still permits.

## CLMM / concentrated liquidity

- **TWAP deviation gate before any liquidity redeployment.** Rebalances that pick tick ranges
  from spot are sandwichable: skew spot → force liquidity into a wrong range → revert skew.
  Any function that deploys/redeploys liquidity should compare spot against a TWAP with a
  bounded deviation.
- **TWAP parameters need their own bounds.** An admin who can set `max_deviation = 10000` or
  `twap_window = 0` has disabled the protection while the code that looks like protection
  remains. Enforce `MIN_TWAP_WINDOW` and a deviation floor in the setter. Generalizes to every
  "safety parameter" setter.
- **Tick / fixed-point overflow (Cetus 2024 class).** Overflow in tick↔`sqrt_price_x64`
  conversion yields a bogus price that drains the pool. Audit every operation on
  `sqrt_price_x64` and tick values, and verify the pool invariant is re-checked after
  *multi-tick* swaps — N small swaps can produce different state than one large swap.
- **Rounding dust across repeated rebalances.** `amount_in - amount_used` never returned to
  `idle_a`/`idle_b` accumulates permanently unwithdrawable. Quantify over N rebalances.
- **Manipulation-cost estimation for read pools.** For each external pool whose spot the
  protocol reads, estimate the cost of moving the price vs. what it unlocks. Sui-specific split:
  for a CLMM the cost depends on liquidity in the *active tick range*, not total TVL; for an
  order book it depends on book depth. Turns "oracle manipulable?" into a quantified argument.

## Flash-swap receipts

- **Repayment references the receipt, not the pool balance.** If repayment is validated as
  `pool.balance >= threshold` rather than `balance_after - balance_before >= amount + fee` (as
  recorded in the receipt), a donation or a concurrent deposit *in the same PTB* satisfies the
  check. Confirm the receipt cannot be wrapped or stored to escape the PTB.
- **Retroactive fee-rate change on unclaimed LP fees.** `set_protocol_fee` without harvesting
  first re-prices already-earned fees at the new rate.

## Auctions

- **Self-bidding resets the timer.** A position owner bids on their own liquidation auction to
  extend it indefinitely and dodge seizure. Check `sender != position_owner` and a
  `MAX_AUCTION_DURATION` cap on cumulative extensions.
- **Duration bounds & boundary off-by-one.** `duration = 1ms` enables instant seizure;
  `>=` vs `>` lets a bid and a seizure land in the same millisecond.
