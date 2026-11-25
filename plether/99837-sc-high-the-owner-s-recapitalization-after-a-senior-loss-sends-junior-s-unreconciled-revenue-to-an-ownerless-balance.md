# \#99837 [SC-High] The owner's recapitalization after a Senior loss sends Junior's unreconciled revenue to an ownerless balance

**Submitted on Nov 15th 2026 at 12:40:57 UTC by @Audittens for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99837
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Permanent freezing of unclaimed yield

## Description

## Brief/Intro

When an LP loss is larger than Junior's principal, Junior is wiped to zero and Senior falls below its high-water mark, the amount Senior is owed back, while Junior holders keep their shares. The owner then restores Senior with `CfdEngine.recapitalizePool`, and any LP revenue after that belongs to Junior. The code says recapitalization is [always processed](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L7-L10) before revenue, and the specification says only value with no safe owner [belongs in](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L650) the pool's unassigned balance (`unassignedAssets`). Both fail when the recapitalization and revenue are settled in the same reconcile while Senior's principal is above zero: the revenue fills Senior's gap first, the recapitalization finds no gap left, and the same amount goes to the unassigned balance, where no Junior holder can claim it.

## Vulnerability Details

The pool records the two inflows differently. `recordClaimantInflow` always queues a recapitalization, but it queues revenue only when both principals are zero; otherwise revenue only raises the pool's assets:

[packages/perps/src/HousePool.sol:640-648](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L640-L648)
```solidity
if (cashMode == IHousePool.ClaimantInflowCashMode.CashArrived) {
    accountedAssets += amount;
}

if (kind == IHousePool.ClaimantInflowKind.Recapitalization) {
    _recordPendingClaimantInflow(kind, amount);
} else if (seniorPrincipal + juniorPrincipal == 0) {
    _recordPendingClaimantInflow(kind, amount);
}
```

`_reconcile` then [settles the two](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1564-L1598) in the wrong order. `planReconcile` takes the queued recapitalization out of the surplus and sends the rest, which holds the unqueued revenue, through `distributeRevenue`, which [restores Senior first](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L147-L158) up to its high-water mark and gives Junior only what is left. Only after that does `_applyPendingClaimantBucketsLive` apply the recapitalization, which can go only to Senior's remaining gap, and the rest of it goes to the unassigned balance:

[packages/perps/src/libraries/HousePoolPendingPreviewLib.sol:124-136](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L124-L136)
```solidity
        uint256 gap = state.waterfall.seniorHighWaterMark > state.waterfall.seniorPrincipal
            ? state.waterfall.seniorHighWaterMark - state.waterfall.seniorPrincipal
            : 0;
        if (gap > 0) {
            uint256 seniorAssignedAssets = remaining > gap ? gap : remaining;
            state.waterfall.seniorPrincipal += seniorAssignedAssets;
            remaining -= seniorAssignedAssets;
        }
    }
}
if (remaining > 0) {
    state.unassignedAssets += remaining;
}
```

The revenue has taken the gap the recapitalization was meant to fill, so Junior holders lose the smaller of the revenue, Senior's gap and the recapitalization.

Nothing about this needs a race. `recapitalizePool` [does not reconcile](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L632-L653) before it records the recapitalization, so every LP gain since the last reconcile, whether it arrived before or after the owner's call, is settled in the same batch. Any account can then trigger that reconcile: with no open position, `HousePool.settleLpEpoch(0, 0)` is [permissionless](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L797-L835) and needs only some LP epoch work, such as a matured redemption, and with open positions any keeper does the same through `OrderRouter.settleLpEpoch`.

Once revenue is in the unassigned balance, no Junior holder can claim it. The only way to move it is the owner's `assignUnassignedAssets`, which [mints the balance](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L660-L704) as new shares to one receiver the owner picks. While the balance is above zero, tranche deposits are also [held back](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1100-L1115).

The known issue [MED-12](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf) (page 33 of the Cecuro AI audit), "Mixed recovery inflows can strand LP revenue in ownerless pool assets", is listed as Addressed, and it names the same consequences: blocked deposits and a recovery that needs the owner to choose a receiver. But it reports the recovery after a wipeout, when both principals are zero, and its fix covers only that case. That is the only case in which revenue is queued behind the recapitalization, and the only one the project's regression test checks: `test_MixedPendingInflowsPreserveRecapitalizationPriorityAfterWipe` first calls [`_wipePrincipal`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/differential/HousePoolSnapshotParity.t.sol#L125-L133), which zeroes Senior as well. A loss larger than Junior normally leaves Senior impaired but above zero, which is the state the owner recapitalizes from, and the proof of concept below loses Junior's revenue there with the MED-12 fix in place.

### Attack scenario

No attacker is needed; the owner's documented recovery and routine activity are enough.

1. Senior holds 1,000,000 USDG and Junior 300,000 USDG, plus 1,000 USDG seeds each, under the release Senior caps. A trader's short of 500,000 units, opened at 1.00, is closed at 1.80 with a 400,000 USDG profit, and the LP loss is settled: Junior's principal becomes 0, Senior's 901,458 USDG, and Senior's high-water mark stays 1,000,028 USDG. Junior holders keep their shares.
2. The owner follows the [documented recovery](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L155-L163) and calls `CfdEngine.recapitalizePool` with 98,569 USDG, exactly Senior's gap.
3. A trader opens a long of 150,000 units at 1.80 through `OrderRouter.commitOrder` and closes it at 1.84. Its 6,000 USDG price loss and its charges are 6,110 USDG of LP revenue.
4. A Senior LP's routine redemption request matures, and anyone calls `HousePool.settleLpEpoch(0, 0)`. The reconcile gives the 6,110 USDG of revenue to Senior's gap, then applies the recapitalization to the smaller gap that remains and puts 6,110 USDG of it in the unassigned balance.
5. Senior is restored, Junior's principal is 0 and the Junior LP's shares are worth 0. Had anyone settled between steps 2 and 3, the same shares would be worth 6,090 USDG.

## Impact Details

Permanent freezing of unclaimed yield. Junior holders lose LP revenue they own, which ends in a balance no LP can claim. In the test, 6,110 USDG of LP revenue that belongs to Junior holders is left ownerless and the Junior LP's shares stay worth 0, against 6,090 USDG with one extra settle.

Junior holders permanently lose up to the smaller of the revenue, Senior's gap and the recapitalization per recovery. The value is not destroyed but ownerless, and the only way to move it on chain gives it to a receiver the owner picks. The owner could repay Junior holders off chain, but the protocol gives them no way to claim it.

Conditions:
- A loss larger than Junior's principal has left Senior's principal below its high-water mark but above zero.
- The owner recapitalizes with `recapitalizePool`, the documented recovery from that state.
- Some LP gain, such as a trader's loss, carry or a favourable mark, is unreconciled when the recapitalization is settled.

## Recommended Mitigation

In `_reconcile`, apply the settleable recapitalization before the waterfall runs, so that it fills Senior's gap first and `distributeRevenue` gives the remaining surplus, the unqueued revenue, to Junior. The amount applied this way must also leave the reservation. `planReconcile` subtracts the pending buckets it is given from distributable value, so it must be given only what is still pending, and the recapitalization bucket must be decreased by what was applied. Applying the recapitalization early while still reserving it would subtract it twice and book a loss. `_applyPendingClaimantBucketsLive` then applies only the revenue bucket.

Add a test like the MED-12 test with Senior's principal above zero, asserting that Junior's principal grows by the revenue and the unassigned balance stays zero.

## References

- [HousePool.sol:640-648](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L640-L648)
- [HousePool.sol:1564-1598](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1564-L1598)
- [HousePoolWaterfallAccountingLib.sol:147-158](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L147-L158)
- [HousePoolPendingPreviewLib.sol:116-136](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L116-L136)
- [CfdEngine.sol:632-653](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L632-L653)
- [HousePool.sol:660-704](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L660-L704)

## Proof of Concept

The test uses the release risk parameters and Senior caps from `DEPLOYMENT.md`. Everything goes through public entry points: `OrderRouter.commitOrder` and `OrderRouter.executeOrder` for the trades, including the one that causes the loss in step 1, the owner's `CfdEngine.recapitalizePool`, and a permissionless `HousePool.settleLpEpoch`. After step 2 the test saves the state and plays out the case where anyone settles before the trade, then reverts and plays out the scenario, so the Junior LP's shares can be compared. The owner recapitalizes exactly Senior's gap, so all 6,110 USDG of unassigned assets is Junior's revenue.

Paste the code into `test/POC.t.sol` at the repository root and run:

```bash
forge test --match-path test/POC.t.sol -vv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../packages/perps/test/perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {console} from "forge-std/console.sol";

contract Test is BasePerpTest {

    address internal juniorLp = makeAddr("juniorLp");
    address internal seniorLp = makeAddr("seniorLp");
    address internal winner = makeAddr("winner"); // the trader whose profit is the LP loss in step 1
    address internal trader = makeAddr("trader"); // the trader whose loss is the LP revenue in step 3
    address internal keeper = makeAddr("keeper");
    address internal anyone = makeAddr("anyone");

    function setUp() public override {
        super.setUp();
        // Release Senior caps from DEPLOYMENT.md.
        _setSeniorCapacity(40_000_000e6, 8000);
    }

    /// @dev Release risk parameters from DEPLOYMENT.md.
    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 10,
            initMarginBps: 20,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function test_recapSendsJuniorRevenueToUnassigned() public {
        // Step 1: Senior holds 1,000,000 USDG and Junior 300,000 USDG. A trader's short of 500,000 units, opened
        // at 1.00, is closed at 1.80 with a 400,000 USDG profit. The LP loss is settled: Junior's principal becomes
        // 0 and Senior's falls below its high-water mark.
        _fundJunior(juniorLp, 300_000e6);
        _fundSenior(seniorLp, 1_000_000e6);
        vm.warp(vm.getBlockTimestamp() + seniorVault.DEPOSIT_COOLDOWN());
        _fundTrader(winner, 50_000e6);
        _trade(winner, CfdTypes.Side.SHORT, 500_000e18, false, 25_000e6, 1e8);
        _trade(winner, CfdTypes.Side.SHORT, 500_000e18, true, 0, 1.8e8);
        _settleByAnyone();
        uint256 gap = pool.seniorHighWaterMark() - pool.seniorPrincipal();
        console.log("after the loss:");
        console.log("  Junior principal (USDG):                       ", pool.juniorPrincipal() / 1e6);
        console.log("  Senior principal (USDG):                       ", pool.seniorPrincipal() / 1e6);
        console.log("  Senior high-water mark (USDG):                 ", pool.seniorHighWaterMark() / 1e6);

        // Step 2: the owner calls `CfdEngine.recapitalizePool` with exactly Senior's gap.
        settlementToken.mint(address(this), gap);
        settlementToken.approve(address(engine), gap);
        engine.recapitalizePool(gap);
        console.log("owner's recapitalization (USDG):                 ", gap / 1e6);

        // For comparison: anyone settles between steps 2 and 3.
        uint256 snapshot = vm.snapshotState();
        _settleByAnyone();
        _losingRoundTrip();
        _settleByAnyone();
        uint256 juniorLpValueWithSettle = juniorVault.convertToAssets(juniorVault.balanceOf(juniorLp));
        vm.revertToState(snapshot);

        // Step 3: a trader opens a long of 150,000 units at 1.80 and closes it at 1.84. Its 6,000 USDG loss is
        // LP revenue.
        _losingRoundTrip();

        // Step 4: a Senior LP's routine redemption matures and anyone settles the epoch.
        _settleByAnyone();

        // Step 5: Junior's principal is 0 and the Junior LP's shares are worth 0.
        uint256 juniorLpValue = juniorVault.convertToAssets(juniorVault.balanceOf(juniorLp));
        console.log("after the recapitalization and the revenue are settled together:");
        console.log("  Junior principal (USDG):                       ", pool.juniorPrincipal() / 1e6);
        console.log("  unassigned assets (USDG):                      ", pool.unassignedAssets() / 1e6);
        console.log("  Junior LP's shares (USDG):                     ", juniorLpValue / 1e6);
        console.log("  Junior LP's shares, settled after step 2 (USDG):", juniorLpValueWithSettle / 1e6);

        assertEq(juniorLpValue, 0, "Junior LP's shares are worth 0");
    }

    /// @dev The Senior LP queues a small redemption; when it matures, anyone settles the LP epoch.
    function _settleByAnyone() internal {
        uint256 shares = seniorVault.balanceOf(seniorLp) / 1000;
        vm.prank(seniorLp);
        seniorVault.requestRedeem(shares, seniorLp, seniorLp);
        vm.warp(vm.getBlockTimestamp() + 2 hours);
        vm.roll(vm.getBlockNumber() + 1);
        vm.prank(anyone);
        pool.settleLpEpoch(0, 0);
    }

    function _losingRoundTrip() internal {
        _fundTrader(trader, 30_000e6);
        _trade(trader, CfdTypes.Side.LONG, 150_000e18, false, 15_000e6, 1.8e8);
        _trade(trader, CfdTypes.Side.LONG, 150_000e18, true, 0, 1.84e8);
    }

    /// @dev The account commits an open or a close through `OrderRouter.commitOrder`, and the keeper executes it
    ///      at `price`.
    function _trade(
        address account,
        CfdTypes.Side side,
        uint256 size,
        bool isClose,
        uint256 margin,
        uint256 price
    ) internal {
        uint256 window = router.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode(router.nextCommitId())),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp() + window),
                executionWindowSeconds: uint32(window),
                allowedExecutionModes: 7,
                expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
                maxExecutionBountyAssets: type(uint256).max,
                maxExecutionNotionalAssets: type(uint256).max,
                maxGrossAccountDebitAssets: type(uint256).max,
                maxActionChargeAssets: type(uint256).max,
                maxExplicitFeesAssets: type(uint256).max,
                maxPostPositionSize: type(uint256).max,
                minPostSettlementBalanceAssets: 0,
                minPostPositionEquityAssets: 0,
                maxPostLeverageBps: type(uint32).max
            })
        });
        vm.prank(account);
        uint64 orderId = router.commitOrder(request);
        bytes[] memory priceData = _mockPythUpdateData(price);
        vm.prank(keeper);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, priceData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order executes");
    }

}
```

Output:

```
Ran 1 test for test/POC.t.sol:Test
[PASS] test_recapSendsJuniorRevenueToUnassigned() (gas: 14611544)
Logs:
  after the loss:
    Junior principal (USDG):                        0
    Senior principal (USDG):                        901458
    Senior high-water mark (USDG):                  1000028
  owner's recapitalization (USDG):                  98569
  after the recapitalization and the revenue are settled together:
    Junior principal (USDG):                        0
    unassigned assets (USDG):                       6110
    Junior LP's shares (USDG):                      0
    Junior LP's shares, settled after step 2 (USDG): 6090

Suite result: ok. 1 passed; 0 failed; 0 skipped
```
