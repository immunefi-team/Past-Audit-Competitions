# \#98392 [SC-Low]  Recapitalizing a fully wiped pool erases Senior's unrecovered principal and hands its recovery to Junior

**Submitted on Nov 7th 2026 at 15:52:23 UTC by @Afriauditor for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98392
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  - Permanent freezing of funds

## Description


## Summary

Senior's high-water mark is the protocol's record of the principal Senior lost and is still owed: future revenue must repay it before Junior receives anything. A loss keeps the mark on purpose, and revenue arriving afterwards honours it. The one inflow that does not is the owner's documented recovery action. When both tranches have been wiped to exactly zero and the owner calls `recapitalizePool` with less than the mark, `applyClaimantRecapitalizationIntent` overwrites the mark with the recap amount (`HousePoolPendingPreviewLib.sol:117-122`). Senior's remaining claim is gone, and the revenue that would have repaid it goes to Junior, the first-loss tranche. If one atom of principal survives the same loss, the same recap keeps the mark in full.

In the PoC, Senior's 101,001 USDG claim is cut to 10,100 USDG by a 10% recap. The 90,901 USDG of later revenue that restores Senior in full when there is no recap goes entirely to Junior when there is one. A partial rescue leaves Senior worse off than no rescue at all.

## Description

**1. The mark is Senior's claim, and every other path keeps it.** The whitepaper defines the mark as "the unimpaired senior entitlement that future revenue must restore before junior receives residual value", with "Loss does not reduce H" and "Later revenue first restores S to H" (`WHITEPAPER.md:627-637`; `ACCOUNTING_SPEC.md:331-338`). The code agrees on the loss side: `absorbLoss` leaves the mark "deliberately unchanged so future revenue continues to restore senior impairment" (`HousePoolWaterfallAccountingLib.sol:161-163`), and a total wipe goes through that function. It agrees on the revenue side: with both principals at zero, `applyRevenueIntent` computes Senior's gap as the whole mark and fills it before Junior receives anything (`HousePoolPendingPreviewLib.sol:158-175`). The coupon path also pays Senior impairment first (`ACCOUNTING_SPEC.md:347-349`).

**2. A recap is documented as restoring that claim, not replacing it.** `recapitalizePool` is the owner's "recovery action" (`SECURITY.md:155-157`). The spec says a recap "may restore an existing protected senior entitlement … it mints no new senior LP shares" (`ACCOUNTING_SPEC.md:450-452`), is "intended to restore waterfall claimants", and is allocated "to senior restoration" (`ACCOUNTING_SPEC.md:634-643`; `WHITEPAPER.md:763-764`). Existing entitlement, restored, no new shares.

**3. The recap path discards the claim at exactly zero.** `HousePoolPendingPreviewLib.sol:117-122`:

```solidity
if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
    state.waterfall.seniorPrincipal += remaining;
    state.waterfall.seniorHighWaterMark =                 // overwrites Senior's existing mark
        recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
    remaining = 0;
} else {
    // any principal left: restores Senior toward its existing mark
}
```

`recapitalizationTargetAssets` is the full recorded recap, so the new mark is the recap amount. The stored mark is not consulted.

**4. It is a cliff, not a policy.** The same loss leaving one atom of principal takes the `else` branch and keeps Senior's full mark (PoC case C). No document gives a reason for an entitlement that flips on exact zero. The branch requires `seniorSupply > 0`, so it only runs while the pre-wipe Senior shares are live: `_isTerminallyWiped` blocks new deposits, not existing shares (`TrancheVault.sol:1639-1641`), and the recap accrues to those same shares. There is no fresh party whose expectations a reset protects; the only beneficiaries are Junior holders, who by design absorb losses first and are restored last. The reset makes Junior first-loss on the way down and first-recovery on the way up.

**5. The re-base was copied from a path where it makes sense.** `assignUnassignedAssets` also sets the mark to the deposited amount at zero Senior principal (`HousePool.sol:691-693`), but that path mints new shares at the current NAV, so a new mark for the new capital structure is coherent. A recap mints none (`ACCOUNTING_SPEC.md:450-452`, `:640-643`): it restores the same holders whose mark it then erases.

**6. The behaviour is recorded, not justified.** The branch's NatSpec describes the mechanics for exactly this state, "With senior owners and zero claimed principal", and calls it a bootstrap (`HousePoolPendingPreviewLib.sol:103-110`); it gives no reason for dropping the stored mark and does not reconcile that with the spec and whitepaper above. Two tests assert the new mark equals the recap (`TrancheRecoveryBehavior.t.sol:54`, `:609`), and both cite `ACCOUNTING_SPEC.md#ownership-routing-for-pool-inflows` as their source, a section that describes a recap only as restoration. Cecuro's MED-12 fix note shows a post-wipe recap followed by revenue to Junior, but never states the pre-wipe mark; with a pre-wipe mark equal to the recap, that outcome is exactly what the preserved-mark rule gives. The published rule that an LP can read says the claim survives; the only statements that it does not are a code comment and two test strings.

**7. The owner cannot avoid it.** The owner's only input is an amount, transferred and recorded in full (`CfdEngine.sol:632-651`). Nothing short of recapping the whole mark preserves Senior's claim, and a partial rescue leaves Senior worse off than no rescue at all (PoC A vs B).

**Fix.** On that branch, keep the larger of the existing mark and the recap target:

```solidity
uint256 target = recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
state.waterfall.seniorHighWaterMark =
    state.waterfall.seniorHighWaterMark > target ? state.waterfall.seniorHighWaterMark : target;
```

A genuinely fresh pool's mark is already zero, so bootstrap behaviour is unchanged. If the team instead stands by the reset, `WHITEPAPER.md:627-637`, `ACCOUNTING_SPEC.md:331-338` and `:450-452`, and the `absorbLoss` NatSpec all need to say that a total wipe followed by a recapitalisation re-bases Senior's entitlement to the recap amount, and the `absorbLoss` comment needs to stop promising the opposite; today a Senior LP reading every published document would conclude the claim survives.

## Impact

 (loss of funds: Senior's unrecovered principal).

- Senior holders permanently lose the part of their lost principal the recap does not cover: capital the protocol promised to repay from future revenue before Junior.
- That revenue goes to Junior holders instead, including anyone who buys Junior's near-worthless shares after the wipe and before the recovery.
- In the PoC (101,001 USDG Senior mark, 10% recap, then 90,901 USDG of revenue), Senior ends with 10,100 USDG instead of 101,001 USDG, and Junior receives 90,901 USDG. Without the recap, the same revenue restores Senior to 90,901 USDG.
-


## Proof of Concept

## Proof of Concept

File: `packages/perps/test/poc/SeniorMarkResetOnRecap.t.sol`, on the repo's `BasePerpTest` fixture. The loss is simulated the way the team's own wipe tests do it (`TrancheRecoveryBehavior.t.sol:583-589`), and the recap goes through the public `CfdEngine.recapitalizePool`.

```bash
cd packages/perps
forge test --match-path test/poc/SeniorMarkResetOnRecap.t.sol -vv
```

Three cases from one snapshot:

- A: exact wipe, owner recaps 10% of Senior's mark, then revenue arrives. The mark drops to the recap amount and the revenue goes to Junior.
- B: same wipe, same revenue, no recap. The revenue restores Senior.
- C: the loss leaves one atom of principal, then the same recap and revenue. Senior's mark is kept and Senior recovers in full.

```
[PASS] test_RecapAfterExactWipeErasesSeniorRecoveryClaim() (gas: 936582)
Logs:
  Senior high-water mark before the loss: 101000940638
  Owner recap: 10100094063
  Later revenue: 90900846575
  A exact wipe + recap:   Senior 10100094063
                          Junior 90900846575
  B exact wipe, no recap: Senior 90900846575
                          Junior 0
  C one atom left + recap: Senior 101000940638
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {console2} from "forge-std/console2.sol";

contract SeniorMarkResetOnRecapTest is BasePerpTest {

    address internal constant SENIOR_LP = address(0x5E1);
    address internal constant JUNIOR_LP = address(0x1E1);

    uint256 internal seniorMarkBefore;

    function setUp() public override {
        super.setUp();
        _fundSenior(SENIOR_LP, 100_000e6);
        _fundJunior(JUNIOR_LP, 100_000e6);
        seniorMarkBefore = pool.seniorHighWaterMark();
    }

    function test_RecapAfterExactWipeErasesSeniorRecoveryClaim() public {
        uint256 recap = seniorMarkBefore / 10; // owner rescues 10% of what Senior lost
        uint256 recovery = seniorMarkBefore - recap; // later revenue covering the rest
        uint256 snapshot = vm.snapshotState();

        // A. Exact wipe, partial recap, then recovery.
        _loss(0);
        assertEq(pool.seniorPrincipal() + pool.juniorPrincipal(), 0, "both tranches wiped to exactly zero");
        assertEq(pool.seniorHighWaterMark(), seniorMarkBefore, "the loss itself keeps Senior's mark");
        _recap(recap);
        assertEq(pool.seniorHighWaterMark(), recap, "the recap rewrites Senior's mark down to the recap amount");
        _revenue(recovery);
        uint256 seniorA = pool.seniorPrincipal();
        uint256 juniorA = pool.juniorPrincipal();

        // B. Same wipe and same later revenue, but no recap: Senior recovers first.
        vm.revertToState(snapshot);
        snapshot = vm.snapshotState();
        _loss(0);
        _revenue(recovery);
        uint256 seniorB = pool.seniorPrincipal();
        uint256 juniorB = pool.juniorPrincipal();

        // C. One atom of principal survives the loss: the same recap keeps the mark.
        vm.revertToState(snapshot);
        _loss(1);
        assertEq(pool.seniorPrincipal() + pool.juniorPrincipal(), 1, "one atom of principal survives");
        _recap(recap);
        assertEq(pool.seniorHighWaterMark(), seniorMarkBefore, "with one atom left, the recap keeps the mark");
        _revenue(recovery);
        uint256 seniorC = pool.seniorPrincipal();

        console2.log("Senior high-water mark before the loss:", seniorMarkBefore);
        console2.log("Owner recap:", recap);
        console2.log("Later revenue:", recovery);
        console2.log("A exact wipe + recap:   Senior", seniorA);
        console2.log("                        Junior", juniorA);
        console2.log("B exact wipe, no recap: Senior", seniorB);
        console2.log("                        Junior", juniorB);
        console2.log("C one atom left + recap: Senior", seniorC);

        assertEq(seniorA, recap, "A: Senior keeps only the recap");
        assertEq(juniorA, recovery, "A: the recovery owed to Senior goes to Junior");
        assertEq(seniorB, recovery, "B: without the recap, the same revenue restores Senior");
        assertEq(seniorC, seniorMarkBefore, "C: one atom of principal and Senior recovers in full");
    }

    /// @dev Simulates a trading loss that leaves `leftAssets` in the pool, as the team's own wipe tests do.
    function _loss(uint256 leftAssets) internal {
        uint256 total = pool.totalAssets();
        vm.prank(address(pool));
        settlementToken.transfer(address(0xdead), total - leftAssets);
        vm.prank(address(juniorVault));
        pool.reconcile();
    }

    /// @dev The documented owner recovery action, through the public engine entrypoint.
    function _recap(uint256 amount) internal {
        settlementToken.mint(address(this), amount);
        settlementToken.approve(address(engine), amount);
        engine.recapitalizePool(amount);
        vm.prank(address(juniorVault));
        pool.reconcile();
    }

    /// @dev Later revenue reaching the pool (for example collected carry or trader losses), as the team's tests model it.
    function _revenue(uint256 amount) internal {
        settlementToken.mint(address(pool), amount);
        vm.prank(address(engine));
        pool.recordClaimantInflow(
            amount, IHousePool.ClaimantInflowKind.Revenue, IHousePool.ClaimantInflowCashMode.CashArrived
        );
        vm.prank(address(juniorVault));
        pool.reconcile();
    }

}
```
