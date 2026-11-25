# \#99688 [SC-Low] An increase executes above the trader's action-charge and gross-debit limits

**Submitted on Nov 14th 2026 at 20:01:30 UTC by @nemveer for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99688
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro
A trader can set financial limits on an increase order and still have that increase execute above both limits. Committing the increase collects carry accrued on the existing position, but the Router reports zero collected carry to the order's lifecycle record. The execution policy then checks the new trade charge without adding the carry already paid at commitment. In a public Router test, Alice's 10,000-token LONG increases to 20,000 tokens although the combined action charge and her actual account debit each exceed the limits she set.

## Vulnerability Details
Alice has an open position with accrued carry and submits a V3 increase through `OrderRouter.commitOrder`, setting `maxActionChargeAssets` and `maxGrossAccountDebitAssets`. Both are intended to cap what the order may charge her. The protocol's agent guide defines action charge to include carry, VPI and fees.

The increase follows the open-order reservation branch. `OrderReservationAccounting._reserveExecutionBounty` sets `effects.bountyFromFreeAssets` but never sets `effects.carryCollectedAssets` (`packages/perps/src/router/OrderReservationAccounting.sol:108-128`). Reserving the bounty calls `MarginClearinghouse.lockReservedSettlement`, which checkpoints and collects Alice's accrued carry (`packages/perps/src/MarginClearinghouse.sol:818-823`). `OrderCommitHandler._createPendingOrder` then records the incomplete `effects` in the lifecycle Book before reserving committed margin; that later reservation can also checkpoint carry (`packages/perps/src/router/OrderCommitHandler.sol:27-35`, `packages/perps/src/MarginClearinghouse.sol:620-633`).

`OrderLifecycleBook.recordCommitment` checks `effects.carryCollectedAssets` against Alice's action-charge limit and adds it to the bounty for the gross-debit check. For this increase, it checks and stores zero even though carry was taken from Alice's settlement balance (`packages/perps/src/OrderLifecycleBook.sol:327-360`). At execution, `CfdOrderPolicyEvaluator.assessCommittedOrder` adds recorded commitment carry to the assessment only for closes. The increase is assessed against the post-commit account state, so its previously collected carry is absent from the execution assessment as well (`packages/perps/src/CfdOrderPolicyEvaluator.sol:45-67`).

The limit can therefore pass twice: once at commitment with zero reported carry and once at execution with only the new trade charge. A close does not have this gap because its reservation path returns the collected carry in `effects`.

## Impact Details
The PoC opens a 10,000-token LONG, lets carry accrue for 30 days, and commits a 10,000-token increase with these bounds:

| Amount | USDG |
| --- | ---: |
| Carry collected at commitment | 0.329691 |
| Action charge assessed for execution alone | 4.074850 |
| Alice's `maxActionChargeAssets` | 4.239695 |
| **Combined action charge** | **4.404541** |
| Gross debit assessed for execution alone | 4.274850 |
| Alice's `maxGrossAccountDebitAssets` | 4.439695 |
| **Actual Clearinghouse debit from before commitment through execution** | **4.604541** |

Both combined amounts exceed Alice's respective limits by **0.164846 USDG**. The increase nevertheless executes and doubles her position to 20,000 tokens. If the commitment carry were included in the execution check, this bounded order would fail instead of charging the new trade cost and increasing Alice's exposure. The carry was already owed on her original position; the harm is that the protocol executes an additional trade despite the ceiling she placed on its total financial effect. The lifecycle Book also records zero commitment carry, understating the debit to clients that use its order record.



## Proof of Concept

## Proof of Concept
Attach the single file `INS10_FinancialBoundsBypass.t.sol`. It uses the repository's existing `OrderRouterTestBase` fixture and MockPyth for the price update; no contract under `packages/perps/src` is modified. From the repository root:

```sh
cp submissions/final/INS-10_increase_commit_carry_bounds/INS10_FinancialBoundsBypass.t.sol packages/perps/test/poc/INS10_FinancialBoundsBypass.t.sol
FOUNDRY_TEST=test/poc forge test --root packages/perps --match-path 'test/poc/INS10_FinancialBoundsBypass.t.sol' --offline --out out-ins10-report --cache-path cache-ins10-report -vv
```

The test opens Alice's position, advances 30 days, and submits the bounded increase. It asserts the carry debit and the zero carry recorded in the Book, then has Bob execute the order as an ordinary permissionless keeper. It checks both cap breaches, the final Clearinghouse debit and the resulting 20,000-token position. The end-to-end test passed **1/1** against the competition revision `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Copy this single file to packages/perps/test/poc/ before running it.
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {OrderRouterTestBase} from "../perps/shared/OrderRouterTestBase.sol";

contract INS10_FinancialBoundsBypassTest is OrderRouterTestBase {
    uint256 private constant ACTION_CAP = 4_239_695; // 4.239695 USDG
    uint256 private constant GROSS_CAP = 4_439_695; // 4.439695 USDG

    function _boundedIncrease() internal view returns (OrderV3Types.OrderRequest memory request) {
        request = OrderV3Types.OrderRequest({
            clientOrderId: bytes32(uint256(0x10)),
            side: CfdTypes.Side.LONG,
            sizeDelta: 10_000e18,
            marginDelta: 500e6,
            targetPrice: 1,
            isClose: false,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(block.timestamp + 60),
                executionWindowSeconds: 60,
                allowedExecutionModes: 7,
                expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
                maxExecutionBountyAssets: type(uint256).max,
                maxExecutionNotionalAssets: type(uint256).max,
                maxGrossAccountDebitAssets: GROSS_CAP,
                maxActionChargeAssets: ACTION_CAP,
                maxExplicitFeesAssets: type(uint256).max,
                maxPostPositionSize: type(uint256).max,
                minPostSettlementBalanceAssets: 0,
                minPostPositionEquityAssets: 0,
                maxPostLeverageBps: type(uint32).max
            })
        });
    }

    function test_IncreaseExecutesWithActualDebitAboveBothFinancialCaps() public {
        // Alice opens a position, then leaves it untouched while carry accrues.
        vm.prank(alice);
        uint64 first = router.commitOrder(CfdTypes.Side.LONG, 10_000e18, 1000e6, 1, false);
        bytes[] memory openingPrice = _mockPythUpdateData();
        router.executeOrder(first, openingPrice);
        (uint256 startingSize,,,,,,) = engine.positions(alice);
        assertEq(startingSize, 10_000e18);

        vm.warp(block.timestamp + 30 days);
        vm.roll(block.number + 1);

        // _boundedIncrease reads the Book, so build it before vm.prank(alice).
        OrderV3Types.OrderRequest memory request = _boundedIncrease();
        uint256 balanceBefore = clearinghouse.balanceAssets(alice);
        vm.prank(alice);
        uint64 increaseId = OrderRouter(payable(address(router))).commitOrder(request);

        uint256 carryCollectedAtCommit = balanceBefore - clearinghouse.balanceAssets(alice);
        assertEq(carryCollectedAtCommit, 329_691);

        OrderV3Types.PendingIntent memory pending = router.lifecycleBook().pendingIntent(increaseId);
        assertEq(pending.commitment.carryCollectedAssets, 0, "Book omits the collected carry");

        // The execution assessment sees only the new trade charge, not prior carry.
        bytes[] memory executionPrice = _mockPythUpdateData();
        OrderV3Types.ExecutionAssessment memory assessment =
            policyEvaluator.assessCommittedOrder(address(engine), increaseId, bob, 1e8);
        assertEq(assessment.actionChargeAssessedAssets, 4_074_850);
        assertEq(assessment.grossAccountDebitAssets, 4_274_850);
        assertLe(assessment.actionChargeAssessedAssets, ACTION_CAP);
        assertLe(assessment.grossAccountDebitAssets, GROSS_CAP);
        assertGt(assessment.actionChargeAssessedAssets + carryCollectedAtCommit, ACTION_CAP);
        assertGt(assessment.grossAccountDebitAssets + carryCollectedAtCommit, GROSS_CAP);

        // Any keeper can now execute Alice's increase despite the combined charge.
        vm.deal(bob, 1 ether);
        vm.prank(bob);
        router.executeOrder(increaseId, executionPrice);

        (uint256 finalSize,,,,,,) = engine.positions(alice);
        assertEq(finalSize, 20_000e18, "increase executed");

        uint256 actualDebit = balanceBefore - clearinghouse.balanceAssets(alice);
        assertEq(actualDebit, 4_604_541);
        assertGt(actualDebit, GROSS_CAP, "actual debit exceeds Alice's bound");

        emit log_named_decimal_uint("carry collected at commit", carryCollectedAtCommit, 6);
        emit log_named_decimal_uint("new trade action charge", assessment.actionChargeAssessedAssets, 6);
        emit log_named_decimal_uint("actual action charge including carry", assessment.actionChargeAssessedAssets + carryCollectedAtCommit, 6);
        emit log_named_decimal_uint("actual account debit", actualDebit, 6);
    }
}
```
