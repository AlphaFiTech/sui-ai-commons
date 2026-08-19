# Playbook — Sui-Native Pitfalls

Load for any audit — these are platform-level hazards that source-level DeFi checklists miss.
Several have permanent-brick or silent-auth-bypass failure modes with no EVM analogue.

## On-chain randomness (`sui::random`)

- **Test-and-abort / gas-refund lottery.** An `entry` fn consuming `sui::random` (`0x8`) that
  returns/emits/branches on the roll *before* the irreversible consequence lands lets a PTB read
  the outcome and `abort` when unfavorable, retrying until it wins. Trace each
  `random::generate_*` value forward to its use site: the payout/mint must be irreversible within
  the same call, and nothing derived from the roll may be observable-then-abortable. **Critical**
  for lotteries/loot boxes/raffles.
- **Predictable pseudo-randomness.** `tx_context::digest`, `object::uid_to_bytes`, `epoch`, and
  `epoch_timestamp_ms` are all deterministic/known pre-execution — an attacker simulates locally
  and submits only winning txns. The only safe source is `sui::random::Random` (VRF, v1.22+).
  Grep those four APIs feeding any selection/winner/shuffle/rarity logic.
- `sui::random` in a `public` (not `entry`) function is rejected by the framework — relevant only
  as a won't-compile flag during incomplete/source review.

## Signature verification & replay

- **Unchecked verify return value.** `ed25519_verify` / `ecdsa_*` returns `bool`; if the return
  is not `assert!`-ed, *every* signature passes — silent auth bypass. **Critical.**
- **secp256k1 malleability as a replay key.** Raw `(r, s)` signature bytes used as a
  replay-prevention `Table` key are bypassed by resubmitting `(r, n−s)` — different bytes, same
  signer. The replay key must be the message hash, not signature bytes (ed25519 strict-mode is
  non-malleable).
- **Missing domain separation / chain-id binding.** A signed message omitting chain id + this
  package/contract address replays across testnet/mainnet or sibling deployments. Deserialize the
  signed payload and confirm both are inside it.
- **Nonce replay.** Verified signature with no per-signer nonce consumed / no used-message-hash
  marker → the same authorization executes repeatedly.
- **Missing outcome params in the signed message.** If `recipient` (or amount, deadline) is a
  call argument but *not* inside the signed bytes, the submitter redirects funds while the
  signature stays valid. List every outcome-affecting param and confirm each is in the payload;
  verify BCS field order matches between signer and verifier.
- **No expiration.** Absent a `deadline`/`expires_at` field asserted against `clock`, the
  authorization is lifetime and survives whitelist/KYC revocation.
- **Signatures validated against live mutable policy, not signing-time policy.** If threshold /
  signer-set is read from admin-mutable state rather than snapshotted into the signed message, a
  deliberately-not-executed 2-of-3 request becomes valid after a routine threshold-lowering, with
  no fresh approval. Bind a monotonic `signer_policy_nonce` (bumped on every signer/threshold
  mutation) into the signed message. *Distinct from nonce replay — the request can be unused yet
  mis-validated.*

## Cross-chain / bridge / ZK

- **Recipient semantic mismatch (address vs object ID).** A source chain that locks/burns on any
  nonzero `bytes32` recipient, while the Move destination parses it as an `address` and aborts if
  the account/object doesn't exist, strands funds: locked on source, VAA never claimable. Prove
  source, docs/UI, and Move destination interpret the recipient bytes as the same identity type;
  confirm a refund/rescue path.
- **Abort-after-finality lock.** Any destination abort past the finality point (unsupported type,
  denylist, missing policy, `EAccountNotFound`) makes a valid message permanently unclaimable.
  Enumerate every such abort; require a recovery path.
- **Fake bridge package object.** A VAA/attestation object accepted by type-name only is spoofed
  by deploying a homonymous-type package. Verify the object's *originating package address*, not
  just struct layout; validate guardian-set-index currency and emitter chain/address.
- **Bridged-token `TreasuryCap` custody.** The mint cap for a wrapped `Coin<BridgedType>` must be
  held exclusively by the bridge; a shared object wrapping it without gating = anyone mints.
- **ZK nullifier not enforced.** Proof verified but no on-chain nullifier `Table` checked/stored →
  the same proof replays as multiple votes/claims. Require `assert!(!contains(nullifiers, n))`
  before accept + insert after; nullifier derived from a private input, not submitter-controlled
  public input; table is shared, not owned.

## Hard platform limits (non-gas, permanent-brick)

- **Per-transaction dynamic-field child-object cache ceiling.** The object runtime caps the
  number of *distinct* dynamic-field children loaded per transaction (default 1,000); the 1,001st
  aborts with `MEMORY_LIMIT_EXCEEDED`. Three traps: the budget is cumulative across *all* PTB
  commands (not per-command), it is not a gas limit (raising gas does nothing; gas headroom is
  not evidence of safety), and `sui move test` does *not* enforce it (green tests prove nothing —
  only a node dry-run at projected scale does). Every atomic multi-entity op (settlement, sweep,
  epoch roll, mass liquidation, migration, keeper PTB) needs a child-budget model summed across
  commands. Severity turns on resumability and whether a cheap permissionless path can mint new
  distinct keys to inflate the count.
- **Object byte-size cap (~256KB).** A `vector`/`VecMap`/`VecSet` field on a `has key` struct is
  serialized *inline*; permissionless appends grow the object monotonically until *every* write to
  it aborts — a protocol constant, not a gas budget. Especially a `vector` redundant with an
  existing `Table` (pure liability, removable). Use `Table`/`Bag`/`TableVec` or events.
- Per-transaction created/transferred/deleted object-ID counts are separately capped.

## Standard-object & framework misuse

- **`public(package) entry` is directly callable** — the `entry` modifier defeats package-only
  visibility, exposing "internal" admin logic. Enumerate every `public(package) entry`.
- **Sender taken as an `address` parameter** instead of `tx_context::sender(ctx)` for an
  ownership/auth decision — trivially spoofed. (A legitimate *recipient* address param is a
  distinct, lesser finding.)
- **Object-reference fields typed `address` instead of `ID`** — no compile-time guarantee it
  refers to an object; user addresses get confused with object IDs.
- **Passed-in data-bearing object not canonicity-checked.** `&Pool` / `&mut Vault` proves the
  *type*, never that this is *the* protocol instance. Any function reading reserves/prices/ratios/
  permissions from a passed-in object must assert `object::id(obj)` against a registry/allow-list/
  stored expected ID — else an attacker mints their own `Pool` with fake reserves.
- **Kiosk royalty bypass via self-purchase.** A freely-transferable `KioskOwnerCap` lets the owner
  list at price 0 and self-purchase, skipping `TransferPolicy` royalties/allowlist.
- **`Publisher` not secured post-init.** Left in the deployer EOA it lets a key-compromise spoof
  `Display` metadata, claim type ownership, or reconfigure transfer policies. Wrap in governance,
  or `package::burn_publisher` if unneeded. It stays valid across upgrades; multiple `package::
  claim` calls split authority across several `Publisher` objects.
- **Secret/PII in `event::emit` or `Display<T>` templates.** Chain storage is public; "private"
  fields are readable straight from indexers/wallets regardless of accessor visibility.
- **Two-step authority handoff.** Direct `transfer(admin_cap, addr)` with no propose→accept
  (where `accept` asserts `sender == proposed_admin`) is unrecoverable on a typo — the cap *is*
  the only key.
- **Denylist enforcement is validator-level, not Move code.** Flagging a missing denylist check in
  a regulated-coin `transfer` is a false positive (validators block sending pre-execution). The
  *real* risk is the ~24h epoch gap for *receiving* — burn-on-source/mint-on-destination can
  strand funds if the recipient is blocked between epochs.

## Fixed-point library specifics

- **Hidden multiply-before-divide overflow inside a helper `mul`.** `A.mul(B).div(C)` looks safe
  but `float`/`decimal`/`wad_ray` `mul` computes `(a*b)/WAD` internally and aborts *before*
  `div(C)` normalizes. Open the helper, derive its `VALUE_MAX`, prove `A*B ≤ bound` at each call,
  and build a threshold table with real token decimals (e.g. 500k USDC overflows after ~10h idle
  in a reward accrual).
- **Bit-shift silent wrapping.** `<<`/`>>` do **not** abort on overflow in Move (unlike `+ − *`) —
  `1u64 << 64 == 0`. An off-by-one in a `checked_shl` bound (`<` vs `<=`) silently corrupts
  instead of aborting. Check every shift's guard boundary and that shift amount ≤ bit width.
- **Packed-field mask width mismatch.** A getter `(v >> shift) & mask` that decodes fewer bits than
  the writer's true max (a count of 16 in a 4-bit field decodes to 0 at the power-of-two boundary)
  flips `is_empty`/`can_close` predicates. Distinguish index-max from count-max; boundary-test 0,
  1, max_index, max_count, 8, 16, 32 round-trip.
- **Narrowing casts** (u128→u64, u64→u8) need an explicit `assert!(value <= MAX_TYPE)`. Watch for
  **double scaling** — an interest index applied twice in one calculation.

## Deployed-package (bytecode) review

When the target is a published package with no matching source, disassemble it (`sui move
disassemble`) and apply the same checks; cite by module, basic-block label, instruction index.

- **Survives compilation:** abilities, visibility, `entry` markers, full signatures. **Does not:**
  constant/error-code names, local names, comments, macro sugar. OTW structs show a synthetic bool
  field.
- **`#[test_only]` is compiled out** — it cannot be grepped in a deployed package, and a leaked
  test helper ships as an unauthenticated mint/admin bypass. The *shape* is the signal: walk every
  public/entry fn whose name (`*_for_testing`, `mint_for_testing`, debug setters) or whose body
  (privileged action, no authorization) is test scaffolding. Applies to source review too.
- **`#[spec_only]` is NOT compiled out — it ships to production.** `#[test_only]`/`#[test]` are
  *built-in* attributes the standard toolchain strips; `#[spec_only]` is not built-in at all — it is
  meaningful **only to the `sui-prover` toolchain**, and `sui move build` / `sui client publish` do
  **not** run the prover. The base compiler treats it as an *unknown attribute*
  (`warning[W02018]: Unknown attribute 'spec_only'. Custom attributes must be wrapped in 'ext'`) and
  compiles the annotated function into the published bytecode unchanged — `#[ext(...)]` wrapping only
  silences that warning; no custom attribute ever strips code. Verified on sui 1.76.1: a
  `#[spec_only] public fun` verifier helper disassembles out of the deployed module as a live
  `public` function. Because it visually rhymes with `#[test_only]`, developers assume the same
  "stripped in prod" guarantee — so a verifier-only helper written to drain/mint/bypass
  authorization for proof purposes ships as a live, usually public, unauthenticated function (the
  publicly disclosed "$300K back door" class, pika/@pikapikasui, 2026-08-19).
  Detection: grep `#[spec_only]` and any `use prover::` imports — ANY `#[spec_only]` on a function/module is a red
  flag, and a privileged unauthenticated body (drain/mint/set-admin/supply-mut) marked `#[spec_only]`
  is **CRITICAL**; an `unknown attribute` (W02018) warning on a security-relevant item is the tell
  that the marker is inert, so treat it as a finding, not noise (`warnings = "allow"` or an un-gated
  build hides it entirely); and unlike `#[test_only]`, spec-only code IS present in deployed
  bytecode, so walk public/entry fns for prover-helper shapes (`*_spec`, `spec_*`, `prove`, `ghost`,
  `axiom`) with the same suspicion as test-helper shapes. Correct remediation keeps prover specs in a
  separate spec-only module/package that is never published (or a build step enforcing the
  separation) — never the attribute alone.
- **Guard-completeness idiom:** verify an authorization guard is a 3-part opcode sequence (push
  field/getter → `Eq/Lt/Gt` → `BrFalse` to `LdConst` + `Abort`) present on *every* basic block
  reaching the privileged `Call`. A guard elsewhere in the module does not count.
- **Verify deployed bytecode matches reviewed source**, and prefer MVR / pinned `Move.lock`
  revisions over floating git deps (resists named-address hijacking).
