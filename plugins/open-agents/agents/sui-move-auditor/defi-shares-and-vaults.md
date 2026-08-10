# Playbook — Share Accounting & Vault Inflation

Load when the target mints/burns shares against a pooled asset (vaults, LP tokens,
receipt tokens, `total_shares`/`total_assets`, `xToken`-style wrappers). Sui differs
from EVM here — read the "Sui donation nuance" note before reporting an inflation bug.

## Root-cause checks (do these first)

- **`total_assets` source.** Is share price computed from `balance::value(&pool.reserves)`
  (a *live* read) or from an internal counter mutated only by `deposit`/`withdraw`? The
  live-read design is what makes inflation/donation attacks possible — the tracked-counter
  design is structurally immune. This is the real check; the mint formula is downstream.
- **First-depositor / share-price inflation.** When `total_supply == 0` the opening
  depositor sets the share:asset ratio. Classic play: mint 1 share, inflate the asset side,
  later deposits round to 0 shares and are absorbed. Look for a mitigation: minimum first
  deposit, burned "dead" shares (mint to `@0x0`), or a virtual-share/offset in the formula.
- **Zero-share mint guard.** Any path that computes `shares == 0` while still taking the
  caller's coins is a silent loss. Require `assert!(shares > 0, ...)`. This is the payload
  of the inflation attack, not a separate bug.

## Sui donation nuance (avoid the false positive)

`public_transfer(coin, <shared object address>)` creates a **separate owned object**; it
does **not** join the shared object's `Balance<T>`. The EVM "donate to the vault" attack
does not port directly. The real Sui donation vectors are:

1. a `public` function that accepts `Coin<T>`/`Balance<T>` and `join`s it into reserves
   without a matching accounting update,
2. a sweep / `receive` / rescue function that pulls owned objects into reserves,
3. dynamic-field injection.

State which vector you found; do not report a bare `public_transfer` donation.

## Symmetry & round-trip

- **Mint/burn valuation symmetry.** Shares minted and redeemed at the same exchange-rate
  formula, with rounding that always favors the protocol (deposit rounds shares *down*,
  withdraw rounds tokens *down*).
- **Round-trip profitability.** `deposit(X)` immediately followed by `withdraw(all)` must
  return `<= X`. If it returns more, mint and burn use asymmetric rates or opposite rounding.

## Sui-specific lifecycle

- **Return-to-zero ≠ genesis.** Sui shared objects can never be deleted, so a vault drained
  back to `total_supply == 0` still holds residual rewards, unclaimed fees, dust, and stale
  timestamps. Re-entry recreates the first-depositor scenario *pre-seeded with value* —
  strictly worse than genesis. Test `total_supply == 0 && balance > 0`.
- **Default-valued fields on first use.** Struct numeric fields default to `0`. A function
  that subtracts/divides using `last_update`, `start_time`, or an index field before it is
  ever written produces absurd results (elapsed = full unix time). Trace the *first*
  invocation of each state-mutating function.
- **Pre-configuration window.** `init` runs once at publish, but full config often takes
  several follow-up admin txns. Enumerate which user-facing functions are callable during
  that gap and whether unconfigured (zero) parameters advantage early users. Look for an
  `is_initialized` / version gate.
- **Cross-address deposit default state.** For `deposit(recipient, coin)` with
  `recipient != sender`, check the newly created position's checkpoint. A fresh position with
  `reward_per_share_paid = 0` against a global index of N claims the entire history. (See also
  the staking playbook — uninitialized checkpoint.)

## Coin/Balance ghost accounting

- **`Coin<T>` (object) vs `Balance<T>` (bare value).** Audit every `into_balance`/`from_balance`
  boundary for a matching state update — a mint/credit to internal accounting with no backing
  on-chain asset movement is a phantom resource.
- **Missing claimed-flag after transfer.** A claim/refund/withdraw that moves funds but never
  flips `claimed`, decrements a balance, or consumes the certificate is re-callable to drain.
- **`destroy_zero` on a possibly-non-zero balance.** Common in close/liquidation remainder
  paths: either it aborts (permanent DoS on close) or discards funds. Verify zero is
  *guaranteed* and leftovers route back to the owner. (Inverse of zero-balance *pollution*.)
- **Dust griefing on closable objects.** An attacker sends dust so `destroy_zero` /
  `table::destroy_empty` can never succeed, permanently locking a victim's account.

## Fees & parameters

- **Fee-path bypass matrix.** Enumerate *every* exit path — normal, emergency, admin, batch —
  and confirm each routes through the same fee helper. If fees accrue into a counter/balance,
  verify a withdrawal function actually exists (accrued-but-unclaimable revenue is a finding).
- **Parameter interaction matrix.** Two independently-valid parameters can combine into an
  invalid economic state (reward rate constant while supply → 0 drives per-unit reward to ∞;
  two compounding fees both at max). Build a pairwise table; don't validate parameters in
  isolation.
- **Aggregate constraint across independently-settable weights.** When per-pool weights/splits
  live in separate dynamic fields, the setter often validates only the field it writes and
  never re-reads the others to check the sum.
- **`swap_remove` destroys index semantics.** `vector::swap_remove` / `table_vec::swap_remove`
  silently relocates the last element into the removed slot. Fine for sets; corrupts anything
  ordered — FIFO withdrawal queues, "first N depositors" bonuses, priority lists. Grep every
  call site and classify the container.
