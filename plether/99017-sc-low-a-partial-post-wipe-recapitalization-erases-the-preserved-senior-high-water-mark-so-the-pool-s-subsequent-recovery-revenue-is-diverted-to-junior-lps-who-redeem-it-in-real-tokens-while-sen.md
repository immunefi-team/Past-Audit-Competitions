# \#99017 [SC-Low] A partial post-wipe recapitalization erases the preserved senior high-water mark, so the pool's subsequent recovery revenue is diverted to junior LPs, who redeem it in real tokens while senior LPs recover almost nothing

**Submitted on Nov 11th 2026 at 12:57:52 UTC by @t0x1c for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99017
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Summary and attack path

- `seniorHighWaterMark` is the senior entitlement that future revenue must restore after a loss. The protocol states the rule normatively in its accounting specification and restates it in the loss function's own NatSpec: the mark is **deliberately unchanged** by a loss.
- `absorbLoss` honours it. The zero-principal branch of `applyClaimantRecapitalizationIntent` does not: it **assigns** the mark from the recapitalization amount instead of taking the maximum against the existing mark. That branch is reached in exactly the post-loss state the rule was written for.
- Once the mark collapses, `distributeRevenue` fills only the new, smaller gap and routes all remaining revenue to junior. The destroyed senior entitlement never comes back, and senior redemption prices off the resulting principal. Four other writers of the same scalar preserve the rule, so this is an outlier rather than a policy.

### Attack path

This is a value-routing defect that fires on ordinary operation, not an exploit sequence.

1. Senior principal stands at `1,000,000` USDG with the high-water mark equal to it. Junior is funded. This is the steady state.
2. Trading losses wipe junior to zero and then senior to zero. `absorbLoss` correctly leaves the high-water mark at `1,000,000`, because future revenue is supposed to restore senior to that level before junior earns anything. The executed test asserts this leg works.
3. The owner recapitalizes a small amount, `100` USDG, which is the natural operational response to a wiped pool and is also the documented recovery route.
4. The zero-principal branch fires and **assigns** the high-water mark to `100` USDG. The senior tranche's `999,900` USDG restoration entitlement is destroyed in one statement, with no event distinguishing it from an ordinary bootstrap.
5. The pool recovers and earns `500,000` USDG of revenue. Senior principal stops at `100` USDG, because the gap is already full, and the entire `500,000` lands in `juniorPrincipal`. Senior LPs who supplied capital on the strength of the documented restoration entitlement receive none of the recovery; junior LPs receive all of it.

### Root cause

The protocol writes the rule down twice. In the accounting specification, reconciliation rule 3:

> Charge a shortfall to Junior first, then Senior. Losses do not reduce the Senior high-water mark.

[`ACCOUNTING_SPEC.md#L337`](https://github.com/immunefi-team/audit-comp-plether/blob/master/packages/perps/ACCOUNTING_SPEC.md#L337), with the scalar defined at [`ACCOUNTING_SPEC.md#L331-L332`](https://github.com/immunefi-team/audit-comp-plether/blob/master/packages/perps/ACCOUNTING_SPEC.md#L331-L332) as "the Senior entitlement that future revenue must restore after a loss". And in code, in the NatSpec of the loss function itself:

```solidity
/// @dev Principal subtraction saturates at zero if loss exceeds all claimed equity. The senior high-water mark is
///      deliberately unchanged so future revenue continues to restore senior impairment.
```

[`libraries/HousePoolWaterfallAccountingLib.sol#L160-L162`](https://github.com/immunefi-team/audit-comp-plether/blob/master/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L160-L162).

The violating branch assigns rather than maximizes:

```solidity
if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
    state.waterfall.seniorPrincipal += remaining;
    state.waterfall.seniorHighWaterMark =
        recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
    remaining = 0;
} else {
```

[`libraries/HousePoolPendingPreviewLib.sol#L118-L123`](https://github.com/immunefi-team/audit-comp-plether/blob/master/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L118-L123). The maximum is taken between the two **incoming** quantities only. The **existing** mark is not an operand, so a prior entitlement of any size is discarded. The guard on the branch, `seniorPrincipal == 0 && juniorPrincipal == 0`, is satisfied by a fresh deployment and equally by a fully loss-wiped pool, and the code cannot distinguish them.

Only the overwritten value is persisted, at [`HousePool.sol#L1878-L1884`](https://github.com/immunefi-team/audit-comp-plether/blob/master/packages/perps/src/HousePool.sol#L1878-L1884). The consequence is routed by `distributeRevenue`, whose own NatSpec states the intended behaviour: "Revenue first fills `seniorHighWaterMark - seniorPrincipal`. Remaining revenue accrues to junior principal" ([`libraries/HousePoolWaterfallAccountingLib.sol#L134`](https://github.com/immunefi-team/audit-comp-plether/blob/master/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L134), implemented at [`libraries/HousePoolWaterfallAccountingLib.sol#L147-L156`](https://github.com/immunefi-team/audit-comp-plether/blob/master/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L147-L156)). With the gap collapsed, "first fills the gap" is satisfied immediately and everything else goes to junior.

The sibling writers get it right, which is the strongest evidence that this branch is a mistake rather than a design decision. `absorbLoss` leaves the mark alone ([`#L167-L184`](https://github.com/immunefi-team/audit-comp-plether/blob/master/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L167-L184)); `distributeRevenue` ratchets it **upward** only, at [`#L154-L156`](https://github.com/immunefi-team/audit-comp-plether/blob/master/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L154-L156); the coupon path raises it; and the non-bootstrap recapitalization branch gap-fills and preserves it.

## Impact

- **Senior LP capital is permanently re-routed to junior LPs, and it is realized in redeemed token balances.** This is a transfer between two distinct user groups, not a rounding artefact or an accounting-only change. In an executed end-to-end run on the real `HousePool` and `TrancheVault`, through the protocol's own authorized entrypoints (deposit, loss, recapitalization, revenue, redemption), a preserved high-water mark of `1,001,009` USDG was collapsed to `100` USDG by a `100` USDG recapitalization, and of the `500,000` USDG of subsequent recovery revenue the junior LP **redeemed `497,535` USDG in settlement tokens while the senior LP redeemed `99.90`**. The recovery that reconciliation rule 3 reserves for senior restoration is realized by junior instead.
- **It inverts the tranche contract.** Senior supplies capital at a lower return in exchange for priority and a documented restoration entitlement. Junior takes first loss in exchange for the residual. After this branch fires, junior has taken the first loss **and** collects the recovery, which is the opposite of the advertised waterfall.
- **It is silent and irreversible.** Nothing distinguishes the collapse from a legitimate bootstrap, there is no event for it, and no later operation restores the mark: `distributeRevenue` only ratchets it up to principal, which is already capped at the collapsed value.
- **It relaxes a covenant as a side effect.** `juniorWithdrawalRatioCap` is computed against protected exposure derived from the senior principal and the high-water mark ([`libraries/HousePoolSeniorCapacityLib.sol#L64-L90`](https://github.com/immunefi-team/audit-comp-plether/blob/master/packages/perps/src/libraries/HousePoolSeniorCapacityLib.sol#L64-L90)), so collapsing the mark also widens how much junior may withdraw from the capacity that was meant to be retained for senior.

## Why the likely objections fail

**"The branch is for bootstrapping an empty pool, where assignment is correct."** Agreed for a genuinely fresh pool, and that is the problem: the guard is `seniorPrincipal == 0 && juniorPrincipal == 0`, which a loss-wiped pool satisfies identically. Taking the maximum against the existing mark is correct in **both** cases, since a fresh pool's mark is `0` and `max(0, remaining) == remaining`. The fix therefore costs the bootstrap case nothing, which the inlined test demonstrates directly.

**"This is a trusted-owner action."** The owner's action is recapitalizing a wiped pool, which is the documented recovery route and the only one available. The owner is not the beneficiary. The loss falls on senior LPs and the gain accrues to junior LPs, neither of whom took any action.

**Disclosed honestly:** the repository's own test suite contains two tests that encode the reset as intended, in `test/perps/spec/lp/HousePoolBootstrap.t.sol` and `test/perps/accounting/waterfall/HousePoolWaterfall.t.sol`. So the accounting specification and the test suite contradict each other on a value determinant. One of them is wrong. Either way the economic consequence falls on senior LPs who were told, in writing, that losses do not reduce their restoration entitlement, and a documented rule that the implementation violates is the finding whichever side is normative.

## Mitigation

One line. Take the maximum against the existing mark instead of assigning:

```solidity
uint256 candidate = recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
state.waterfall.seniorHighWaterMark =
    state.waterfall.seniorHighWaterMark > candidate ? state.waterfall.seniorHighWaterMark : candidate;
```

This is correct for a fresh bootstrap, where the stored mark is `0`, and preserves reconciliation rule 3 for a loss-wiped pool. If instead the reset is genuinely intended, the accounting specification's rule 3 and the `absorbLoss` NatSpec must be corrected, and senior LPs must be told that recapitalization extinguishes their restoration entitlement.

## Dup check

Not a duplicate, and distinct from the two disclosed findings that touch this scalar. The disclosed senior-high-water-mark finding is the withdrawal-scaling reduction in `HousePoolWaterfallAccountingLib.scaleSeniorOnWithdraw` (the mark reduced by `fundedShares / preBurnSupply` while principal is reduced by funded USDG), remediated in commit `9ab65f41` with `H' = H - floor(H * A / P)`. The separate disclosed recapitalization finding is the `allowRevenueContinuation` miscomputation in `applyClaimantRecapitalizationIntent`, remediated in commit `f852f969` so revenue continuation is recomputed after recapitalization. Neither touches the defect reported here: the **assignment** of `seniorHighWaterMark` in the zero-principal branch of `HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent` (`L118-L123`), which overwrites the existing mark instead of maximizing against it. The inline proof of concept is executed at the audited commit, after both remediations, and the overwrite still fires.


## Proof of Concept

## Proof of Concept

Two test files. The first (`SeniorHwmRealizedLoss.t.sol`) drives the real `HousePool` and `TrancheVault` end to end, with a differential that isolates the defect from any seed artifact and a realized run that proves the loss in redeemed LP token balances. The second isolates the single offending statement against a maximum-taking counterfactual.

### 1. Realized-loss integration test (real contracts, authorized paths)

Add `SeniorHwmRealizedLoss.t.sol` (below) to `packages/perps/test/poc/`, then from the repository root:

```bash
forge test --root packages/perps --match-path "test/poc/SeniorHwmRealizedLoss.t.sol" -vvv
```

`Expected: 2 passing.` It uses production and authorized entrypoints only: `CfdEngine.recapitalizePool` for the owner recapitalization (approve, `transferFrom`, record), `recordClaimantInflow(Revenue)` for the recovery revenue (the same path the repository's own `test_PendingRevenue` uses for a post-wipe pool), `reconcile` to settle, and the vaults' own request, settle and claim redemption. The loss is a physical raw-balance shortfall, asserted before and after; the recovery cash arrives by an explicit engine-to-pool transfer before recognition; and the realized run asserts redemption conservation. No source file is modified and no storage slot is forced.

**Executed result**

```text
[PASS] test_wipeCompletenessFlipsWhoReceivesTheRecovery   (differential, principal level)
  seed senior / junior shares:                 1000000000 / 1000000000
  pool raw backing before/after shortfall (A): 1202000000000 / 0
  pool raw backing before/after shortfall (B): 1202000000000 / 1000000      (1 USDG residual)
  [A full wipe]  HWM after recap:                100000000       (100 USDG)
  [A full wipe]  senior principal after revenue: 100000000       (100)
  [A full wipe]  junior principal after revenue: 500000000000    (500,000)
  [B 1 USDG res] HWM after recap:                1001009150684   (1,001,009; preserved exactly)
  [B 1 USDG res] senior principal after revenue: 500101000000    (500,101)
  [B 1 USDG res] junior principal after revenue: 0

[PASS] test_juniorRedeemsTheRecoveryInRealTokens          (realized ERC-20 balances)
  senior HWM before loss:              1001009150684
  senior HWM after a 100 USDG recap:   100000000
  pool deliverable before redemptions: 500100000000    (500,100 = 100 recap + 500,000 revenue)
  alice (senior) ERC20 received:       99901924        (99.90 USDG)
  bob (junior) ERC20 received:         497535086670    (497,535 USDG)
  total redeemed (alice + bob):        497634988594    (<= pool deliverable: conserved)
```

The differential is the decisive control. The two runs share the same seed shares and the same deposits and differ only by one USDG of residual senior principal after the loss: a full wipe takes the buggy branch and routes the `500,000` recovery to junior, while one USDG short of a full wipe takes the correct branch and routes it to senior. The swing is caused by the branch, not by share structure. The realized run then redeems through the real vaults: the senior LP receives `99.90` USDG and the junior LP receives `497,535` USDG. Both tests were executed and passed on `forge 1.5.1` / `solc 0.8.35` with `via_ir` (the repository default), at the audited commit, with no source file modified.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {HousePoolTestBase} from "../perps/shared/HousePoolTestBase.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";

/// @notice Realized-loss proof for the senior high-water-mark overwrite.
///         ACCOUNTING_SPEC.md rule 3: "Losses do not reduce the Senior high-water mark"; the
///         zero-principal branch of `applyClaimantRecapitalizationIntent` assigns it anyway. Both
///         tests drive the REAL `HousePool` and `TrancheVault`, recapitalize through the PRODUCTION
///         owner entrypoint `CfdEngine.recapitalizePool` (approve, transferFrom, record), and book
///         revenue through the authorized `recordClaimantInflow(Revenue)` path. No source is modified.
///
///         Test 1 is a differential that isolates the defect from any seed/share artifact: the SAME
///         funded state is run twice, differing only by ONE unit of residual senior principal after
///         the loss. A full wipe (both principals zero) takes the buggy branch and routes the recovery
///         to junior; one unit short of a full wipe takes the correct branch and routes it to senior.
///         Test 2 then realizes the vulnerable case in actual redeemed ERC-20 balances.
contract SeniorHwmRealizedLossTest is HousePoolTestBase {

    uint256 constant SENIOR_DEPOSIT = 1_000_000e6;
    uint256 constant JUNIOR_DEPOSIT = 200_000e6;
    uint256 constant RECAP = 100e6;
    uint256 constant REVENUE = 500_000e6;

    // Test 1: differential at the principal level (no redemption, so no degraded-mode confound).
    function test_wipeCompletenessFlipsWhoReceivesTheRecovery() public {
        _fundSenior(alice, SENIOR_DEPOSIT);
        _fundJunior(bob, JUNIOR_DEPOSIT);
        emit log_named_uint("seeded senior shares", SEEDED_SENIOR);
        emit log_named_uint("seeded junior shares", SEEDED_JUNIOR);
        emit log_named_uint("senior total share supply", seniorVault.totalSupply());
        emit log_named_uint("junior total share supply", juniorVault.totalSupply());

        uint256 hwm0 = pool.seniorHighWaterMark();
        uint256 snap = vm.snapshotState();

        // Scenario A: FULL wipe, both principals zero, the buggy recap branch.
        (uint256 hwmA, uint256 seniorA, uint256 juniorA) = _lossRecapRevenue(0);
        vm.revertToState(snap);
        // Scenario B: identical, but the loss leaves 1 USDG of senior principal, the correct branch.
        (uint256 hwmB, uint256 seniorB, uint256 juniorB) = _lossRecapRevenue(1e6);

        emit log_named_uint("[A full wipe]  HWM after recap", hwmA);
        emit log_named_uint("[A full wipe]  senior principal after revenue", seniorA);
        emit log_named_uint("[A full wipe]  junior principal after revenue", juniorA);
        emit log_named_uint("[B 1 USDG res] HWM after recap", hwmB);
        emit log_named_uint("[B 1 USDG res] senior principal after revenue", seniorB);
        emit log_named_uint("[B 1 USDG res] junior principal after revenue", juniorB);

        assertLt(hwmA, hwm0, "[A] BUG: the full-wipe recap collapsed the preserved senior HWM");
        assertEq(hwmB, hwm0, "[B] one unit of residual keeps the correct branch; the original HWM is preserved exactly");
        assertGt(juniorA, seniorA, "[A] the recovery is routed to junior");
        assertGt(seniorB, juniorB, "[B] the recovery is routed to senior restoration");
        // The whole swing turns on one unit of loss, so it is the branch, not seeds or share supply.
        assertGt(seniorB - seniorA, REVENUE / 2, "DIFFERENTIAL: senior receives the recovery only under the correct branch");
    }

    // Test 2: realized in redeemed ERC-20 token balances (vulnerable case).
    function test_juniorRedeemsTheRecoveryInRealTokens() public {
        _fundSenior(alice, SENIOR_DEPOSIT);
        _fundJunior(bob, JUNIOR_DEPOSIT);
        uint256 seniorShares = seniorVault.balanceOf(alice);
        uint256 juniorShares = juniorVault.balanceOf(bob);

        uint256 hwm0 = pool.seniorHighWaterMark();
        emit log_named_uint("senior HWM before loss", hwm0);
        (uint256 hwmAfterRecap,,) = _lossRecapRevenue(0);
        emit log_named_uint("senior HWM after a 100 USDG recap", hwmAfterRecap);
        assertLt(hwmAfterRecap, hwm0, "BUG: recapitalization reduced the preserved senior HWM");

        uint256 poolDeliverable = settlementToken.balanceOf(address(pool));
        _finishAsyncCooldown(seniorVault, alice);
        _finishAsyncCooldown(juniorVault, bob);
        uint256 aliceBefore = settlementToken.balanceOf(alice);
        uint256 bobBefore = settlementToken.balanceOf(bob);
        _redeemAsync(seniorVault, alice, seniorShares, true);
        _redeemAsync(juniorVault, bob, juniorShares, true);
        uint256 aliceReceived = settlementToken.balanceOf(alice) - aliceBefore;
        uint256 bobReceived = settlementToken.balanceOf(bob) - bobBefore;

        emit log_named_uint("pool deliverable before redemptions", poolDeliverable);
        emit log_named_uint("alice (senior) ERC20 received", aliceReceived);
        emit log_named_uint("bob (junior) ERC20 received", bobReceived);
        emit log_named_uint("total redeemed (alice + bob)", aliceReceived + bobReceived);
        assertGt(bobReceived, aliceReceived, "junior realizes the recovery in real tokens; senior does not");
        assertGe(bobReceived, REVENUE / 2, "junior redeems a large share of the 500k recovery");
        assertLt(aliceReceived, RECAP + 1e6, "senior redeems little more than the 100 USDG recap");
        assertLe(aliceReceived + bobReceived, poolDeliverable, "redemptions conserve: no settlement tokens are created");
    }

    /// @dev Funded pool, loss, production recapitalization, authorized revenue. Returns the post-recap
    ///      HWM and the senior/junior principals after the revenue reconciles. `seniorResidualAfterLoss`
    ///      chooses a full wipe (0) or a near wipe (>0).
    function _lossRecapRevenue(uint256 seniorResidualAfterLoss)
        internal
        returns (uint256 hwmAfterRecap, uint256 seniorPrincipalAfter, uint256 juniorPrincipalAfter)
    {
        uint256 hwm0 = pool.seniorHighWaterMark();

        // Loss fixture: a physical raw-balance shortfall (a documented loss), then the real waterfall.
        uint256 rawBefore = settlementToken.balanceOf(address(pool));
        settlementToken.burn(address(pool), pool.totalAssets() - seniorResidualAfterLoss);
        vm.prank(address(juniorVault));
        pool.reconcile();
        emit log_named_uint("  pool raw backing before shortfall", rawBefore);
        emit log_named_uint("  pool raw backing after shortfall", settlementToken.balanceOf(address(pool)));
        assertEq(pool.seniorPrincipal(), seniorResidualAfterLoss, "loss wiped senior to exactly the chosen residual");
        assertEq(pool.seniorHighWaterMark(), hwm0, "rule 3: the loss preserves the HWM");

        // Recapitalization through the PRODUCTION owner entrypoint: approve, transferFrom, record.
        settlementToken.mint(address(this), RECAP);
        settlementToken.approve(address(engine), RECAP);
        engine.recapitalizePool(RECAP);
        vm.prank(address(seniorVault));
        pool.reconcile();
        hwmAfterRecap = pool.seniorHighWaterMark();

        // Recovery revenue: the engine (the settlement actor) delivers the cash into the pool, then books
        // it through the authorized revenue-recognition entrypoint the protocol's own post-wipe test uses
        // (HousePoolWaterfall.test_PendingRevenue). accountExcess is the healthy-pool path and reverts
        // HousePool__NoExcessAssets while the pool is degraded.
        settlementToken.mint(address(engine), REVENUE);
        vm.startPrank(address(engine));
        settlementToken.transfer(address(pool), REVENUE);
        pool.recordClaimantInflow(REVENUE, IHousePool.ClaimantInflowKind.Revenue, IHousePool.ClaimantInflowCashMode.CashArrived);
        vm.stopPrank();
        vm.prank(address(juniorVault));
        pool.reconcile();

        seniorPrincipalAfter = pool.seniorPrincipal();
        juniorPrincipalAfter = pool.juniorPrincipal();
    }
}
```

### 2. Single-statement isolation and counterfactual

Add `SeniorHighWaterMark.t.sol` (below) to `packages/perps/test/poc/`, then from the repository root:

```bash
forge test --root packages/perps --match-path "test/poc/SeniorHighWaterMark.t.sol" -vv
```

`Expected: 2 passing.` `via_ir` is the repository default for this package, so the first build is slow and needs a high-memory host.

**What each test proves**

- `test_recapitalizationDestroysSeniorRestorationEntitlement` - the mark is preserved across the loss, then assigned down by the recapitalization branch; subsequent recovery revenue stops at the collapsed mark and the remainder lands in `juniorPrincipal`; and a maximum-taking counterfactual on the same state routes the whole recovery to senior instead, isolating the single statement as the cause.
- `test_siblingWritersPreserveTheRule` - `absorbLoss` preserves the mark, `distributeRevenue` leaves it untouched at the mark and ratchets it upward above the mark, and the non-bootstrap recapitalization branch gap-fills without touching it. The violating branch is the outlier.

**Executed result**

```text
forge 1.5.1 / solc 0.8.35 / via_ir, at the audited commit, no source file modified.
2 tests passed, 0 failed.
Emitted: H preserved at 1000000000000 across the loss (rule 3 working),
         then ASSIGNED 100000000 by recapitalization: 999,900 USDG of entitlement destroyed.
         500000000000 of recovery revenue re-routed from senior to junior.
         The max() counterfactual sends all of it to senior.
```

Note on the harness: the test drives the two in-scope libraries through thin local wrappers so the single branch is isolated and the counterfactual can be evaluated on identical state without modifying any source file.

### PoC test file (NEW, add to `packages/perps/test/poc/`): `SeniorHighWaterMark.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {HousePoolPendingPreviewLib} from "@plether/perps/libraries/HousePoolPendingPreviewLib.sol";
import {HousePoolWaterfallAccountingLib} from "@plether/perps/libraries/HousePoolWaterfallAccountingLib.sol";
import {Test} from "forge-std/Test.sol";

/// @notice ACCOUNTING_SPEC.md:337 reconciliation rule 3 states "Losses do not reduce the
///         Senior high-water mark". `absorbLoss` honours it (its own NatSpec: the mark is "deliberately
///         unchanged so future revenue continues to restore senior impairment"). The zero-principal
///         branch of `applyClaimantRecapitalizationIntent` then ASSIGNS the mark instead of max-ing it,
///         destroying the restoration entitlement in exactly the post-loss state rule 3 was written for.
contract WaterfallHarness {

    function absorbLoss(
        uint256 sp,
        uint256 jp,
        uint256 hwm,
        uint256 loss
    ) external pure returns (uint256, uint256, uint256) {
        HousePoolWaterfallAccountingLib.WaterfallState memory s =
            HousePoolWaterfallAccountingLib.WaterfallState(sp, jp, hwm);
        s = HousePoolWaterfallAccountingLib.absorbLoss(s, loss);
        return (s.seniorPrincipal, s.juniorPrincipal, s.seniorHighWaterMark);
    }

    function distributeRevenue(
        uint256 sp,
        uint256 jp,
        uint256 hwm,
        uint256 revenue
    ) external pure returns (uint256, uint256, uint256) {
        HousePoolWaterfallAccountingLib.WaterfallState memory s =
            HousePoolWaterfallAccountingLib.WaterfallState(sp, jp, hwm);
        s = HousePoolWaterfallAccountingLib.distributeRevenue(s, revenue);
        return (s.seniorPrincipal, s.juniorPrincipal, s.seniorHighWaterMark);
    }

    /// @dev Exercises the real in-scope branch at HousePoolPendingPreviewLib.sol:118-122.
    function recapitalize(
        uint256 sp,
        uint256 jp,
        uint256 hwm,
        uint256 seniorSupply,
        uint256 juniorSupply,
        uint256 amount,
        uint256 target
    ) external pure returns (uint256, uint256, uint256, uint256) {
        HousePoolPendingPreviewLib.PendingAccountingState memory st;
        st.waterfall = HousePoolWaterfallAccountingLib.WaterfallState(sp, jp, hwm);
        st.seniorSupply = seniorSupply;
        st.juniorSupply = juniorSupply;
        HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent(st, amount, target);
        return (
            st.waterfall.seniorPrincipal,
            st.waterfall.juniorPrincipal,
            st.waterfall.seniorHighWaterMark,
            st.unassignedAssets
        );
    }

}

contract SeniorHighWaterMarkTest is Test {

    WaterfallHarness internal h;

    uint256 internal constant SENIOR_PRINCIPAL = 1_000_000e6;
    uint256 internal constant JUNIOR_PRINCIPAL = 200_000e6;
    uint256 internal constant RECAP = 100e6;
    uint256 internal constant RECOVERY_REVENUE = 500_000e6;

    function setUp() public {
        h = new WaterfallHarness();
    }

    function test_recapitalizationDestroysSeniorRestorationEntitlement() public {
        // --- Step 1: senior at its high-water mark, junior funded.
        uint256 hwm0 = SENIOR_PRINCIPAL;
        emit log_named_uint("H before loss", hwm0);

        // --- Step 2: a trading loss wipes junior then senior to zero principal.
        //     absorbLoss PRESERVES H - this is spec rule 3 working correctly.
        (uint256 sp1, uint256 jp1, uint256 hwm1) =
            h.absorbLoss(SENIOR_PRINCIPAL, JUNIOR_PRINCIPAL, hwm0, SENIOR_PRINCIPAL + JUNIOR_PRINCIPAL);
        assertEq(sp1, 0, "senior principal wiped");
        assertEq(jp1, 0, "junior principal wiped");
        assertEq(hwm1, hwm0, "ACCOUNTING_SPEC.md:337 rule 3: losses do not reduce the Senior high-water mark");
        emit log_named_uint("H after loss (correctly preserved)", hwm1);

        // --- Step 3: owner recapitalizes 100 USDG into the loss-wiped pool.
        (uint256 sp2,, uint256 hwm2,) = h.recapitalize(sp1, jp1, hwm1, 1e18, 1e18, RECAP, RECAP);

        // --- Assert (a): H collapsed from 1,000,000 USDG to 100 USDG.
        assertEq(sp2, RECAP, "senior principal is the recapitalized amount");
        assertEq(
            hwm2,
            RECAP,
            "VIOLATION of ACCOUNTING_SPEC.md:337 rule 3: the recapitalization branch ASSIGNED the high-water mark"
        );
        assertLt(hwm2, hwm1, "the restoration entitlement was destroyed, not preserved");
        emit log_named_uint("H after recapitalization (DESTROYED)", hwm2);
        emit log_named_uint("entitlement destroyed (USDG 6dp)", hwm1 - hwm2);

        // --- Assert (b): subsequent recovery revenue stops at the NEW mark and the rest goes to junior.
        (uint256 sp3, uint256 jp3,) = h.distributeRevenue(sp2, 0, hwm2, RECOVERY_REVENUE);
        assertEq(sp3, RECAP, "senior principal stopped at the collapsed mark");
        assertEq(jp3, RECOVERY_REVENUE, "the entire recovery revenue was re-routed to JUNIOR principal");
        emit log_named_uint("senior principal after 500k revenue", sp3);
        emit log_named_uint("junior principal after 500k revenue", jp3);

        // --- Assert (c): the max-ing counterfactual. One line changed -> senior receives the revenue.
        uint256 hwmCounterfactual = hwm1 > hwm2 ? hwm1 : hwm2; // what `max` would have produced
        (uint256 sp4, uint256 jp4,) = h.distributeRevenue(sp2, 0, hwmCounterfactual, RECOVERY_REVENUE);
        assertEq(sp4, RECAP + RECOVERY_REVENUE, "with max() senior is restored by the full recovery revenue");
        assertEq(jp4, 0, "with max() junior receives nothing until senior is whole");

        // The single-line root cause, isolated: the delta between the two routings.
        assertEq(sp4 - sp3, RECOVERY_REVENUE, "value mis-routed away from senior LPs by the assignment");
        assertEq(jp3 - jp4, RECOVERY_REVENUE, "the same value landed in junior principal");
        emit log_named_uint("senior LP value mis-routed to junior (USDG 6dp)", sp4 - sp3);
    }

    /// @notice The outlier is the branch, not the policy: every other writer preserves the rule.
    function test_siblingWritersPreserveTheRule() public {
        // absorbLoss: preserves.
        (,, uint256 hA) = h.absorbLoss(1_000e6, 500e6, 1_000e6, 1_200e6);
        assertEq(hA, 1_000e6, "absorbLoss preserves H");

        // distributeRevenue never lowers H: at the mark the revenue goes to junior and H is untouched,
        // and above the mark H ratchets UP to principal. Neither direction reduces it.
        (, uint256 jB, uint256 hB) = h.distributeRevenue(1_000e6, 0, 1_000e6, 50e6);
        assertEq(hB, 1_000e6, "distributeRevenue leaves H untouched when senior is at its mark");
        assertEq(jB, 50e6, "and routes the revenue to junior");
        (,, uint256 hB2) = h.distributeRevenue(1_100e6, 0, 1_000e6, 0);
        assertEq(hB2, 1_100e6, "distributeRevenue ratchets H UPWARD when principal exceeds it");

        // The non-bootstrap recapitalization branch gap-fills and leaves H alone.
        (uint256 spC,, uint256 hC,) = h.recapitalize(400e6, 10e6, 1_000e6, 1e18, 1e18, 100e6, 100e6);
        assertEq(hC, 1_000e6, "the gap-fill branch correctly preserves H");
        assertEq(spC, 500e6, "and restores senior principal toward it");
    }

}
```
