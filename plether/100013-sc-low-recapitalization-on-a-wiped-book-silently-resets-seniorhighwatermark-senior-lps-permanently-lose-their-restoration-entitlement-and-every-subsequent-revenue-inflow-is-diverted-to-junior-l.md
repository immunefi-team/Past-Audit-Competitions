# \#100013 [SC-Low] Recapitalization on a wiped book silently resets `seniorHighWaterMark` — senior LPs permanently lose their restoration entitlement and every subsequent revenue inflow is diverted to junior LPs

**Submitted on Nov 16th 2026 at 03:14:26 UTC by @OxSCSamurai for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100013
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Theft of unclaimed yield

## Description

Recapitalization on a wiped book silently resets `seniorHighWaterMark` — senior LPs permanently lose their restoration entitlement and every subsequent revenue inflow is diverted to junior LPs

**Program:** Plether Immunefi audit competition · commit `a82d4fef` · scope `packages/perps/src/**`

**Impact class (this program's list):** High — *"Theft of unclaimed yield"*. The destroyed entitlement is future revenue owed to senior LP shareholders; arguably Critical as a permanent loss of funds attribution, but since the principal was already market-wiped, High is the honest classification.

## 1. Summary

When a loss event wipes both tranche principals to zero, the spec guarantees the senior tranche's
high-water mark is preserved so future revenue restores senior before junior extracts surplus
(`ACCOUNTING_SPEC.md:337`: "Losses do not reduce the Senior high-water mark"). The code violates
this: on the first reconcile after an owner recapitalization, the bootstrap branch of
`applyClaimantRecapitalizationIntent` **overwrites `seniorHighWaterMark` with the recap amount,
discarding the preserved value entirely** — it is in `state` and is never read on that branch.

Any under-recap (recap smaller than the preserved HWM) therefore permanently shrinks senior's
protected restoration entitlement. The damage is **irreversible**: the non-bootstrap path only
gap-fills up to the *current* HWM, so no later recap or revenue can repair the crater. Every
subsequent revenue inflow that should have restored senior instead accrues to junior principal.

The PoC proves it end-to-end with **pure production trading revenue** — no pranked calls into
internal pool functions. After a real claims-regime wipe and a routine owner under-recap,
`seniorHighWaterMark` ratchets **$301,000 → $74,992** across the reconciles that the starving
claims regime re-fires. Four ordinary losing LONG round-trips then book $43,246 of pool revenue,
of which **$30,757.87 is captured by junior LPs** — value that, with the HWM preserved, pays
senior restoration in full ($0 to junior). Conservation holds exactly: the diverted amount is a
pure senior→junior misallocation, and the permanently destroyed entitlement equals
`preservedHWM − resetHWM` (~$226k in the companion test, bounded by the preserved mark).

**The payoff is deliberately harvestable.** Every step of this chain is public state — the
waterfall, `pendingRecapitalizationAssets`, `degradedMode`, the HWM itself — so a patient party
running a trivial monitor can watch an under-recap land in real time, enter junior the moment
the first diverted inflow revives the tranche and reopens deposits, and collect the ongoing
misroute indefinitely. The trigger needs no attacker; the theft has a beneficiary who can choose
to be one. Nothing about the harvest requires privileged access, timing races, or MEV — only a
read of state anyone can make and one ordinary deposit.

## 2. Affected code

| Anchor | Role |
|---|---|
| `HousePoolPendingPreviewLib.sol:118-121` | Bootstrap branch: `seniorPrincipal += remaining; seniorHighWaterMark = max(recapitalizationTargetAssets, remaining)` — preserved `state.waterfall.seniorHighWaterMark` never read |
| `HousePool.sol:691-693` (`assignUnassignedAssets`) | Sibling write: `seniorHighWaterMark = amount` when `seniorPrincipal == 0` — same unexplained reset on the owner assignment path |
| `HousePoolWaterfallAccountingLib.sol:162-163` | `absorbLoss` preserves HWM even at `principal == 0` — proving the protocol *deliberately* keeps the mark through a wipe |
| `HousePoolWaterfallAccountingLib.sol:114` | Comment: "Zero funding preserves even a wiped-out Senior's HWM" |
| `HousePoolPendingPreviewLib.sol:123-132` | Non-bootstrap branch only gap-fills to the *current* HWM → the reset is unrecoverable |

I enumerated all `seniorHighWaterMark` write sites in scope: the only unconditional *downward*
resets are the two anchors above. Every other site is an upward ratchet (tracking new principal
highs), a proportional withdrawal cut, or plumbing.

The vulnerable branch, verbatim:

```solidity
if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
    state.waterfall.seniorPrincipal += remaining;
    state.waterfall.seniorHighWaterMark =
        recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
    remaining = 0;
}
```

## 3. Mechanism

1. A claims regime (real losing positions → trader claims) wipes both tranche principals to 0.
   `absorbLoss` preserves `seniorHighWaterMark ≈ $301k`, as the spec mandates.
2. The owner performs the documented recovery: `recapitalizePool(R)` with `R < H` (under-recap —
   the common case; nothing prescribes the amount, `SECURITY.md:155-160`) and the pending recap
   bucket is fully cash-backed.
3. Reconcile reserves the pending recap before the waterfall; the claims-starved drain re-zeroes
   principals, so the bootstrap branch re-fires on each reconcile — each time writing
   `HWM = max(residualIntent, settleable)` with a shrinking residual. Observed ratchet:
   **$301k → $200k → $137.5k → $75k** (final ≈ the last residual settleable). Note the ratchet is
   *amplifier, not requirement* — a **single** bootstrap fire with `R < H` already craters the mark
   permanently; the repeated re-fires only deepen it.
4. Subsequent revenue inflows hit `distributeRevenue`, which gap-fills senior only up to the
   *cratered* HWM, then pays junior. The preserved ~$226k gap — senior's only remaining claim —
   is gone. Later recaps cannot restore it: once principals are nonzero, only the non-bootstrap
   branch runs and it fills to the *current* (cratered) mark, routing the excess to
   `unassignedAssets`.

## 4. Impact

- **Permanent loss of protected entitlement:** senior's restoration anchor is reduced from the
  preserved $301,000 to $74,992 — a ~$226,000 destruction of the only claim wiped senior LPs
  have left. Bounded by `preservedHWM − resetHWM`; scales with the wipe size, not the PoC.
- **Realized diversion, production path:** $30,757.87 of a single $43,246 revenue epoch captured
  by junior LPs (`PAUSERECReproHwmProd.t.sol`). Every further inflow below the lost gap routes to
  junior instead of senior — the misallocation repeats for the pool's remaining life. This is
  **deferred, not speculative**: the pool keeps operating, revenue keeps booking, and every dollar
  up to the destroyed gap is deterministically misrouted.
- **Secondary state corruption:** `SECURITY.md:1192` defines protected senior exposure
  `E = max(projected senior principal, projected senior high-water mark)` — the HWM *is* the
  protected exposure. Cratering it shrinks senior's protection floor everywhere that reads it
  (capacity limits, covenant bounds, withdrawal protection), not just in revenue routing.
- **The diversion is capturable, not just misrouted:** the stolen revenue accrues to *junior
  principal*. The first inflow revives the wiped junior tranche (an undeserved windfall for
  incumbent wiped junior holders at senior's expense), and once `juniorPrincipal > 0` the deposit
  guard (`HousePool.sol:494-498`, mirroring `ACCOUNTING_SPEC.md:326`) opens — any positioned
  entrant then captures the ongoing misroute pro-rata, which is the bulk of the destroyed
  entitlement since senior's gap-fill saturates at the cratered mark permanently. Everything in
  this chain is publicly observable state (`waterfall`, `pendingRecapitalizationAssets`,
  `degradedMode`), so a prepared party can watch for `R < HWM` and enter the moment deposits
  reopen — hours-to-days timing, no race required.
- **Victim:** senior-tranche LP shareholders — ordinary passive depositors whose tranche was
  market-wiped. Their loss needs no attacker, no owner misbehavior, and no unusual input: the
  trigger is the owner's documented routine recovery action itself.

## 5. Reachability — why this fires on normal use

- The wipe requires both principals at 0 — and this is a cliff, not a knife-edge. `absorbLoss`
  subtraction **saturates** at zero (`HousePoolWaterfallAccountingLib.sol:162`), so *any* claims
  total ≥ total tranche equity pins both principals at exactly 0. Verified empirically that the
  only full-wipe path is the claims regime: price-loss wipes are bounded below equity by the skew
  cap (`SOLVENCY_EXCEEDED` reverts a single $1.5M one-sided open attempt), while trader claims are
  payable from tranche equity and drain both to zero. Insolvency-scale claims are exactly the
  scenario `recapitalizePool` and `degradedMode` exist for — this bug lives inside the protocol's
  own documented catastrophe path, not in an exotic corner.
- `recapitalizePool` is *the* documented post-insolvency recovery op (`SECURITY.md:155-160`
  "an immediate, owner-only, non-reentrant recovery action"; `SECURITY.md:246` documents the
  recap → clear sequence the PoC performs). **Under-recap is not the edge case — it is the
  default.** The owner recaps the *claims gap* — the amount the pool is actually insolvent by —
  not the full historical senior entitlement. After a wipe, `HWM >> gap` almost by construction
  (the gap is what claimants are owed *now*; the HWM is the whole pre-loss entitlement). Any recap
  `R < HWM` triggers the reset, and restoring the entire preserved mark through a single treasury
  injection is the unlikely case, not the common one.
- The reset fires on the **first reconcile after recap** — any permissionless settlement or pool
  flow — before the owner could adjust, and the owner has no documented recovery for a cratered
  HWM. There is no second chance: the mark is destroyed inside the same transaction sequence the
  runbook prescribes.
- **The privileged step is a correct action, not an error or a misuse.** `recapitalizePool` and
  `assignUnassignedAssets` are the only two sources of the recap intent (both owner-only), and
  the triggering parameter — a recap sized to the claims gap — is the economically default choice
  for an operator whose documented job is to restore solvency, not to donate the full pre-loss
  senior entitlement. The bug spans the function's entire normal parameter range (`R < HWM` —
  always true when `HWM > gap`, which holds by construction after a wipe); there is no
  "misconfigured" value to blame. This is a code defect in the documented recovery path, not an
  opsec assumption about admin behavior.
- The trigger surface is doubled by the sibling: an owner choosing `assignUnassignedAssets` to
  restore senior from accumulated unassigned assets on a wiped book hits the same unconditional
  `seniorHighWaterMark = amount` reset at `HousePool.sol:691-693`.
- **Once armed, the bug is self-executing on innocent use.** The diversion requires no exploit
  transaction: every ordinary revenue inflow — trades, fees, carry, VPI — is misrouted by the
  waterfall automatically (the PoC's $30,757.87 diversion is booked by four vanilla round-trips).
  There is no attack path to discount: the protocol misallocates user funds simply by operating
  correctly. Armed by a routine owner action, executed by routine protocol use — the entire
  chain stays inside documented, intended flows.

## 6. The violated invariant — and why intended-design does not cover this

**I-HWM (high-water-mark integrity):** `seniorHighWaterMark` must always record the unimpaired
senior entitlement — the principal level future revenue must restore. It may rise via the
new-principal-high ratchet (`HousePoolWaterfallAccountingLib.sol:155`) and may fall only
proportionally to senior share withdrawals (`scaleSeniorOnWithdraw`). No loss, revenue,
recapitalization, or assignment event may reduce it below the surviving entitlement.

This is not an inferred rule: it is stated unconditionally across the spec set below, and it is
the pattern every other `seniorHighWaterMark` write site already follows — ratchet up,
proportional-withdrawal down, plumbing otherwise. The two anchors in §2 are the only
unconditional downward resets in scope: anomalies against the codebase's own invariant.

The violation is exact and deterministic: `301,000,018,264 → 74,992,137,056`, asserted at the
audited commit. And the precision matters: the *loss clause* held — `absorbLoss` preserved the
mark through the wipe, proving the entitlement survives a full loss. What the reset violates is
the record's semantic obligation: after the loss, the mark must keep recording the entitlement
it protects until that entitlement is restored or withdrawn. No clause releases that obligation
when a recap arrives; reading one in would unenforce the spec's core protection exactly when it
matters most.

The spec is explicit and unconditional:

| Document | Verbatim | Verdict |
|---|---|---|
| `ACCOUNTING_SPEC.md:332` | "`seniorHighWaterMark` records the Senior entitlement that future revenue must restore after a loss." | The reset destroys exactly this record |
| `ACCOUNTING_SPEC.md:337` | "Losses do not reduce the Senior high-water mark." | Unconditional — no wiped-book exception |
| `ACCOUNTING_SPEC.md:338` | "Route surplus first to restore Senior principal to its high-water mark, then to Junior." | The reset truncates the Senior fill target to the cratered mark — junior receives what the spec routes to Senior |
| `ACCOUNTING_SPEC.md:449-452` | recap "may restore an **existing protected senior entitlement**" | The code erases the entitlement it should restore |
| `WHITEPAPER.md:628` | "senior high-water mark: the unimpaired senior entitlement that future revenue [restores]" | Definition-level: the mark *is* the entitlement |
| `WHITEPAPER.md:772` | "Revenue restores impaired senior principal to its high-water mark before junior [extracts surplus]" | The exact routing this bug reverses |
| `WHITEPAPER.md:1657-1658` | Worked example: "a hypothetical $60m shock wipes junior and impairs senior to $40m **while the high-water mark remains $50m**" | The protocol's own worked case preserves the mark through a wipe |
| `SECURITY.md:394` | "High-water-mark protection: Senior impairment must be restored before junior extracts surplus" | The protection this reset removes |
| `SECURITY.md:1192` | "Active protected senior exposure is `E = max(projected senior principal, projected senior high-water mark)`" | The HWM is literally the protected exposure — resetting it corrupts every check that reads E |
| `HousePoolWaterfallAccountingLib.sol:9` | "The high-water mark tracks unimpaired senior entitlement: losses do not reduce [it]" | The library's own stated invariant |
| `HousePoolWaterfallAccountingLib.sol:114` | "Zero funding preserves even a wiped-out Senior's HWM." | The codebase itself states wiped seniors keep their mark; the bootstrap branch is the *only* path that destroys it |
| `ACCOUNTING_SPEC.md:326` | "a wiped tranche cannot be silently revived by an ordinary asynchronous deposit" | The spec treats wiped tranches as *persistent, protected* state — not dead books to reboot |

A "wiped book means fresh start" reading cannot hold: `absorbLoss` deliberately preserves the
mark through the wipe (`:162-163`), the senior shareholders still exist (`seniorSupply > 0` —
on a truly empty tranche the same write would be harmless bookkeeping; it is harmful precisely
because the wiped holders whose entitlement the mark protects are still there), the
whitepaper's worked wipe example keeps the mark at $50m while principal falls to $40m, and the
function's own docstring describes what it writes but never explains discarding the preserved
value. A behavior comment cannot override the protocol spec's unconditional preservation rule —
stated identically across four documents and the library's own file header.

Audit-coverage check: prior reports cover *adjacent* HWM code but not this — the withdrawn-share
HWM scaling issue and the wiped-state revenue→unassigned misroute are different bugs with
different fixes. No prior finding, program document, or known-limitation entry covers a recap
that truncates the preserved mark.

## 7. Distinctness from report #97728 (same subsystem, different root cause)

- **#97728:** `HousePool.payOut` never decrements `pendingRecapitalizationAssets` → the stale
  memo *inflates* the bootstrap entitlement (HWM overshoots; senior takes junior's share).
  Fix: decrement the memo at `payOut`.
- **This report:** the bootstrap branch ignores the *preserved* `seniorHighWaterMark` → the mark
  *deflates* on under-recap (junior takes senior's restoration). Fix: write
  `max(oldHWM, intent, settleable)` instead of `max(intent, settleable)`.

Opposite direction, opposite victim, disjoint fixes. In this PoC `pendingRecapitalizationAssets`
is fully cash-backed and drains to zero through the correct path — `payOut`/`settleTraderClaim`
are never invoked, and the #97728 fix would not touch this path.

## Link to Proof of Concept

https://gist.github.com/dappconsulting/920677bf4d4f775349eafe4a8495aeb7

## Proof of Concept

## 8. PoC

Secret Gist:
> https://gist.github.com/dappconsulting/920677bf4d4f775349eafe4a8495aeb7

Two self-contained test files accompany this report in the attached gist — both build only on
the repo's existing `BasePerpTest.sol` test base, `forge-std`, and in-scope `src/` (no local
helpers). To reproduce, at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`:

1. Copy both files to `packages/perps/test/perps/hunt/`.
2. From `packages/perps`, run:

```
forge test --match-path test/perps/hunt/PAUSERECReproHwmProd.t.sol -vv
forge test --match-path test/perps/hunt/RECAPLPHwmReset.t.sol -vv
```

Expected results:

| File | Tests | sha256 |
|---|---|---|
| `PAUSERECReproHwmProd.t.sol` | 1 passed; 0 failed | `6a76d5f64610600ab18d6df03fce225f8a878d5135d1ea8c74c483cab768d8af` |
| `RECAPLPHwmReset.t.sol` | 2 passed; 0 failed | `9f9073f7974e1751dc932ba4715c693534320fd28059bbf2343b2e070dda306f` |

What the tests prove:

- `test_HwmReset_ProductionRevenue_DivertsToJunior` (`PAUSERECReproHwmProd.t.sol`) — claims
  regime wipe → owner recap + `clearDegradedMode` (the documented recovery sequence) → **4 real
  losing LONG round-trips through `processOrderTyped` → `SettlementSidecar.executeClose` →
  `recordClaimantInflow`** book $43,246 of revenue → junior captures $30,757.87; identical
  history with HWM pinned pays junior $0 (senior restored $62,503 → $105,750). No `vm.prank` of
  pool internals anywhere — `degradedMode` is cleared via the documented `clearDegradedMode`
  owner call.
- `RECAPLPHwmReset.t.sol` (2 tests) — the ratchet 301k→200k→137.5k→75k and the **$226,007.88**
  conserved diversion (exactly `preservedHWM − resetHWM`), counterfactual-verified two ways:
  (a) HWM patched only before the revenue step; (b) HWM pinned after **every** reconcile — an
  exact simulation of the real fix. Identical result under both, ruling out a patching artifact.

Every dollar figure in this report comes from these tests' logged output and assertions.

## 9. Recommended fix

In the bootstrap branch, write the mark as the maximum of all three candidate values so the
preserved entitlement survives:

```solidity
state.waterfall.seniorHighWaterMark = max(state.waterfall.seniorHighWaterMark,
                                        recapitalizationTargetAssets, remaining);
```

Apply the same preservation rule to the `assignUnassignedAssets` sibling write
(`HousePool.sol:691-693`). Note this is backward-compatible with initial seeding: on a genuinely
fresh book `oldHWM == 0`, so `max(0, intent, settleable)` reduces to the current behavior — the
fix only activates the protection that already exists, it changes nothing else.

## 10. Severity

**High — theft of unclaimed yield.** The diverted value is future revenue contractually owed to
senior LP restoration, and the entitlement's destruction is permanent and mathematically
irrecoverable: the non-bootstrap path only gap-fills to the *current* mark, and junior captures
the diverted revenue, so senior principal can never reach the preserved mark through operation.
This is **deferred theft, not speculative loss** — the pool keeps booking revenue and every
dollar up to the destroyed gap is deterministically misrouted to the wrong tranche. The Critical
boundary is arguable (permanent loss of funds attribution), but the honest floor is High: the
underlying principal was already market-lost, so the incremental harm is the revenue entitlement.
No attacker capital, no owner misbehavior, no timing requirement — the bug fires inside the
protocol's own documented recovery procedure, and the diverted stream is deliberately harvestable
by any positioned party monitoring public state.
