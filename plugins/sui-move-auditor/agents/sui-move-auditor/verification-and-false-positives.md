# Playbook — Verification, Evidence & False-Positive Discipline

**Load this before finalizing any finding.** The auditor's zero-trust stance has no built-in
counterweight, so it will over-report unless every candidate passes these gates. A report full
of plausible-but-wrong findings is worse than a shorter accurate one — and in lending especially,
a naive "fix" can make the protocol *less* safe.

## Confidence tiers cap severity

Tag every finding `confirmed` / `likely` / `needs_review`, and let the tag hard-cap severity:

- **`needs_review`** (pattern resemblance only) → **max Medium**. A pure pattern match can never
  be High or Critical.
- **`likely`** (one strong signal, or a traced path without numbers) → may reach High.
- **`confirmed`** requires **two signals from *different* categories** (e.g. a missing-check
  reading *plus* an algebraic proof — not two patterns) → may reach Critical.

## Evidence strength & provenance

Rank signals from weakest to strongest: bare pattern resemblance → confirmed absence of a check
read directly in source → a fully traced call path → an algebraic proof with concrete numbers →
a concrete PoC step sequence → a matching abort in the project's own test output → live on-chain
state showing the precondition is real.

Tag each decisive claim by provenance: `[ZD-CODE]` in-scope source, `[ZD-PROD-SOURCE]` verified
published source, `[ZD-PROD-STATE]` live on-chain state (strong); `[ZD-MOCK]` test mock/fake dep,
`[ZD-DOC]` docs/comments, `[ZD-EXT-UNVERIFIED]` unverified external behavior (weak).

**Weak evidence cannot dismiss a finding.** A dismissal resting on `[ZD-MOCK]`, `[ZD-DOC]`, or
`[ZD-EXT-UNVERIFIED]` may **not** close the finding — downgrade to *questionable* / *over-
classified* instead. Only the strong tags support a dismissal.

## Two-gate feasibility check before any High/Critical

1. **Reachability gate** — name the attacker-accessible entry point, the full call path, and why
   the attacker can obtain every required object/capability.
2. **Math-bounds gate** — substitute real token decimals, supply, fee, and time ranges into the
   triggering expression and show the threshold is actually crossed.

Failing either forces a **downgrade, not a deletion**.

## Four-part naming test for High/Critical

A High or Critical must name all four: **attacker path, victim, invariant broken, harmful
postcondition**. Missing any one forces a downgrade or a *questionable* label. Banned as severity
justification on their own: "this enables extraction", "attacker can profit", "loss of funds
possible".

## Self-hallucination check (cheap, LLM-specific)

After concluding a finding, re-read the cited source *again*: can you name exact file and line;
does the code actually exist with that signature; did you re-read the function, callers, and
callees *after* forming the conclusion; are you reasoning "this looks like X" instead of tracing
data A→B→C? Any failure → drop or redo from scratch. Negative-PoC requirement for a dismissal:
write the normal trace *and* the attempted-exploit trace, and name the exact assert / `init`
constraint that makes the precondition unreachable — not "the type system probably prevents it."

## Move-specific false-positive catalog ("looks vulnerable, is safe")

Each pattern is safe by default; the **bug condition** is when it becomes real:

| Pattern | Why usually safe | Bug condition |
|---|---|---|
| `&mut T` with no auth check | An **owned-object** parameter *is* the gate — only the owner can pass it | The object is **shared** |
| Arithmetic overflow | Move **aborts**, not corrupts | The abort bricks state (abort-before-checkpoint) *or* the attacker profits from it |
| Reentrancy | No dynamic dispatch / callbacks in Move | (n/a — do not report EVM-style reentrancy) |
| Hot-potato non-repayment | Compiler-enforced within the PTB | (n/a) |
| Phantom type parameter | No runtime effect | A *non-phantom* generic left unconstrained |
| UID collision | Impossible | (n/a) |
| Shared-object TOCTOU within one tx | Consensus serializes access | Cross-transaction ordering, or a check spanning two txns |
| Missing denylist check in `transfer` | Validators block at execution | The cross-chain *receiving* epoch gap |
| Sandwich attack | Sui has **no public mempool** | You can argue validator collusion or a predictable keeper tx |

**Overflow carve-out (do not over-dismiss).** Overflow inside a *periodic accumulator* update is
NOT dismissible as "Move just aborts": if the abort precedes the timestamp/index checkpoint, the
delta grows on every retry → permanent freeze of every entry point touching the accumulator,
including admin-cancel. Three questions decide it: is this a periodic update; is the checkpoint
written *after* the aborting arithmetic; does the delta grow on retry?

## Move-aware trust model for input sources

Owned-object parameter → owner-trusted. **Shared-object parameter → untrusted** (anyone can
reference it). Capability-gated parameter → cap-holder-trusted. `Clock`/`TxContext` →
system-trusted. Plain non-object arguments → untrusted. A dynamic-field read inherits the parent's
trust. An external module's return value → untrusted unless its published source was verified.

**Declared-trusted roles are not vulnerabilities.** Read the docs/spec for which roles the protocol
*declares* trusted; "the trusted admin behaves maliciously" is not a finding. Two things stay
reportable regardless: **logic bugs inside admin functions**, and **blast-radius analysis** —
assume a privileged key is compromised: can it drain everything in one transaction, and are there
timelocks or limits? (This qualifies the agent's default "admin can drain = CRITICAL".)

## Discovery techniques worth running before you conclude coverage

- **Asymmetry diff.** Open counterpart pairs side by side — deposit/withdraw, mint/burn,
  borrow/repay, stake/unstake, and user-vs-admin versions — and diff line by line. A check present
  in one and missing in its twin is the highest-signal finding; admin flows are under-tested.
- **"Pattern everywhere except one place."** If `update_rewards`-before-balance-change appears in
  9/10 functions, the 10th is the bug. Enumerate occurrences, hunt the outlier.
- **Look for what's missing, by name:** solvency check after withdrawal, existence check before a
  table read, state update after a claim, withdrawal path for accrued fees, pause mechanism,
  minimum-amount floor, duplicate check on list append, slippage on *admin* AMM paths, token
  reclamation when an address is swapped out, decimal re-adjustment when a token type changes.
- **Absence rule.** A guard *elsewhere* in the module does not clear an unguarded call site. Walk
  every `borrow`/`remove`/`delete`/transfer/assert site individually.
- **Enumeration-completeness gate.** Grep count N (e.g. functions taking a capability) vs. analyzed
  count M; M < N = incomplete, say so.
- **Many-small-vs-one-large equivalence.** N small ops must leave the same end state as one large
  op; divergence = rounding leakage / fee-base error / state corruption. Pairs with the per-call-
  vs-per-transaction limit check.

## Reporting hygiene

- **Cross-finding dedup & consistency.** Before finalizing: merge overlapping root causes, check no
  attack path contradicts a protection documented in another finding, calibrate severity across the
  same class. Report **root causes, not each symptom**.
- **Chained findings.** Individually-blocked findings can combine — one's postcondition supplies
  another's missing precondition. A chain PoC must exercise the full sequence.
- **Status axis.** Track verification confidence (valid / questionable / dismissed / over-classified
  = real bug, inflated severity) separately from remediation status (open / fixed / acknowledged) —
  they are different axes.
- **Recoverability is a first-class field** for any DoS/abort finding: can it be retried in smaller
  pieces, is there an admin path, or are all entry points trapped? This — not the abort itself —
  drives severity.
- **"Verified clean" section.** List the checks that were run and came back clean, so coverage is
  visible and "checked and fine" is distinguished from "never looked at".

## Test-log & PoC discipline

- **Mine build/test output** (when the project compiles) for arithmetic aborts, assertion failures,
  gas/limit hits, and failing/skipped tests — tests execute paths static reading only inspects.
- **`#[expected_failure]` = developer acknowledging an abort.** Map its code to the firing assert;
  an expected *arithmetic-overflow* failure inside financial math is high priority — the overflow
  is real and merely fenced off in tests. Review skipped/disabled tests; they may have been
  disabled because they surfaced something.
- **Mock ≠ production.** A test passing because a mocked dependency behaved well is not evidence
  about production.
- **Before writing any exploit test**, answer precisely: what is the bug (function, module, missing
  check, line — not "access control is missing"); what observable before/after difference proves it
  (concrete field values); what is the exact assertion (a value comparison, not `assert!(worked)`).
  After a fixed number of failed attempts, conclude false-positive with documented reasoning rather
  than forcing it.
