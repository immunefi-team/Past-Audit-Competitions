# \#99845 [SC-Medium] Close execution fee is lost for the treasury when the close has a VPI rebate

**Submitted on Nov 15th 2026 at 12:45:35 UTC by @Audittens for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99845
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro
When a position is closed with a VPI rebate, the close planner nets the execution fee against the rebate and then counts as "fee" only what is left after the netting. The trader still pays the fee (their rebate is smaller by the fee), but the netted part is never sent to the protocol treasury. It stays in the HousePool as LP value. With release parameters, on a live close without unpaid carry the treasury loses the smaller of the fee and the VPI rebate left after the lifetime clamp, and the whole 4 bps close fee when that rebate covers it, e.g. 400 USDG per 1,000,000 USDG closed. The open path handles the same situation correctly and credits the full fee.

## Vulnerability Details
`CfdEnginePlanLib._planCloseActionSettlement` adds the execution fee and the VPI delta into one signed amount:

```solidity
int256 actionNetAssets = cs.vpiDeltaAssets
    + int256(
        cs.executionFeeAssets + cs.frozenSpreadAssets + delta.pendingCarryAssets - delta.realizedCarryAssets
    );
...
if (actionNetAssets > 0) {
    ... // withhold from price gain / collect from the trader
} else if (actionNetAssets < 0) {
    delta.actionRebateAssets = uint256(-actionNetAssets);
}
...
uint256 executionFeeEligibleAssets = nonClawbackWithheldAssets + nonVpiReserveCollectedAssets;
delta.executionFeeAssets =
    cs.executionFeeAssets < executionFeeEligibleAssets ? cs.executionFeeAssets : executionFeeEligibleAssets;
```

If the rebate `R` is bigger than the fee `F`, nothing is withheld or collected, so `executionFeeEligibleAssets = 0` and `delta.executionFeeAssets = 0`. The trader is paid `R - F`.

After that, `_planIsolatedCloseSettlement` computes the pool top-up for the treasury from this already reduced value:

```solidity
uint256 feeTopUpAssets = delta.executionFeeAssets - delta.actionProtocolFeeCreditedAssets; // 0 - 0
```

So `CfdEngineSettlementSidecar._executeClose` pays the trader `R - F` from the pool and credits nothing to the treasury. The pool owed `R` and paid only `R - F`, so the fee `F` paid by the trader stays with LPs.

The same happens partially when the rebate is smaller than the fee: `F - R` is collected from the trader and only `F - R` reaches the treasury.

In general, for a live close without unpaid carry the treasury loses `min(F, R)`.

This is not a lack of cash. The pool has the cash (it just kept `F` by paying a smaller rebate), and the planner simply never asks for the top-up. Two other paths show that the fee is expected to reach the treasury in this situation:

* Open with a VPI rebate bigger than the fee. `_applyOpenSettlementBufferAfterPlan` keeps the full fee and plans a top-up of `fee - directlyCredited` from free pool cash, and `executeOpen` calls `_settleProtocolFeeTopUp(host, delta.executionFeeAssets, ...)` with the full fee. The trader gets `R - F`, the treasury gets `F`.
* Close with a price gain and no rebate. The fee is withheld from the gain (the pool pays a smaller payout) and is then topped up from the pool to the treasury.

Only the "fee withheld from a rebate" case on close is dropped. Also, the order receipt still reports the full fee as charged (`assessment.executionFeeAssets = delta.closeState.executionFeeAssets` in `CfdOrderPolicyEvaluatorBase`), so receipts show fees that the treasury never received.

### Example (release parameters: `vpiFactor = 0.01e18`, `executionFeeBps = 4`)
Pool depth is 10,000,000 USDG, price is 1.00, and the book is flat.

1. Alice opens LONG 1,000,000 USDG. VPI charge is about 500, fee is 400. The treasury gets 400.
2. Alice closes the whole position at the same price. VPI rebate is 499, fee is 400, so `actionNetAssets = -499 + 400 = -99`. The pool pays Alice 99, the treasury gets 0.
3. For comparison, Alice reopens the LONG and Bob opens SHORT 1,000,000 USDG into the same gap. Bob's open has the same 499 rebate and the same 400 fee. The pool pays out 499: 99 to Bob and 400 to the treasury.

Round trip: Alice paid 800 (two fees, correct), the treasury got 400 instead of 800, the pool kept the other 400.

## Impact Details
The protocol treasury does not receive execution fees that traders actually paid. The fees go to LPs instead, so no value leaves the protocol and traders are not affected.

* Loss per live close without unpaid carry, where the trader can pay the rest of the fee, is `min(fee, rebate)`. The rebate here is the VPI rebate left after the lifetime clamp. The loss goes up to the full fee: 4 bps of the closed notional. The rebate cannot exceed the VPI the closed position paid, so the loss is also at most that charge.
* No attacker and no special state is needed, not even an existing skew: a lone 1,000,000 USDG round trip in a flat 10,000,000 USDG pool loses the whole close fee. It happens on every such close (full, partial, TP/SL) whose VPI rebate is above zero after the clamp, also when the rebate is smaller than the fee and the trader receives no net payment. Closing on the heavy side of the skew is not enough by itself: the rebate is capped by the VPI the position paid earlier, so a position that paid none (for example, one opened into a rebate) gets no rebate on the close and its fee reaches the treasury.
* With `vpiFactor = 0.01` the rebate covers the whole fee once the skew before the close is above 4% of pool depth plus half of the closed size, if the position paid at least that much VPI. With a smaller skew the fee is lost partially.
* In a market that stays skewed to one side, a trader who opened on the heavy side paid VPI on the open and gets it back as a rebate on the close. When that rebate covers the fee, as in the example, the round trip generates only the opening fee for the treasury, so half of its fee revenue is lost. With a smaller rebate the loss is `min(fee, rebate)`. Execution fees are the main treasury revenue because the protocol liquidation share is 0 at release.

The docs say the opposite of what the code does: "the protocol execution fee belongs to treasury rather than LP assets" (WHITEPAPER.md). The rule "Treasury, only when cash credited" describes fees that cannot be funded, which is not the case here. The NatSpec of `CloseDelta.executionFeeAssets` does describe the value as "capped by eligible withheld price gain and collected action cash", but nothing documents that the capped-off part of a fee the trader paid should stay with LPs, and the open path does the opposite.

## References
* Fee netting and fee cap: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1041-L1073
* Top-up computed from the reduced fee: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L984-L987
* Close settlement: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L615-L628
* Open path that tops up the full fee: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L718-L749 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L498
* NatSpec of the capped close fee: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEnginePlanTypes.sol#L318
* Receipt reports the full fee: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L431
* "the protocol execution fee belongs to treasury rather than LP assets": https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md?plain=1#L411-L413

## Proof of Concept

The following PoC runs the example from the Vulnerability Details section through the production `OrderRouter.commitOrder(OrderRequest)` and `OrderRouter.executeOrder` with release risk parameters and a 10,000,000 USDG pool. One test runs the three steps:

1. Alice opens LONG 1,000,000 into a flat book. The treasury gets the 400 USDG fee.
2. Alice closes it. The close receipt reports a 400 USDG fee and a 499 USDG rebate, the pool pays out only 99, and the treasury gets 0.
3. Alice reopens, and Bob opens SHORT 1,000,000 into the same gap, with the same rebate and the same fee. The pool pays out 499 and the treasury gets the full 400.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {console} from "forge-std/console.sol";

contract PoC_CloseFeeAbsorbedByVpiRebate is BasePerpTest {

    address alice = makeAddr("alice");
    address bob = makeAddr("bob");
    address keeper = makeAddr("keeper");

    uint256 constant PRICE = 1e8;
    uint256 constant SIZE = 1_000_000e18; // 1,000,000 USDG notional at price 1.00, execution fee = 400 USDG

    // Release parameters from packages/perps/DEPLOYMENT.md (executionFeeBps = 4 is the engine default).
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

    // 2,000 USDG of seeds + 9,998,000 USDG junior deposit = 10,000,000 USDG pool.
    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 9_998_000e6;
    }

    function test_closeWithVpiRebate_treasuryGetsNoFee() public {
        _fundTrader(alice, 50_000e6);
        _fundTrader(bob, 50_000e6);

        // 1. Alice opens LONG 1,000,000 USDG into a flat book. She pays ~500 VPI and the 400 USDG fee.
        uint256 treasuryBefore = _treasury();
        _trade(alice, CfdTypes.Side.LONG, 20_000e6, false);
        console.log("1. Alice opens LONG 1,000,000 into a flat book");
        console.log("   Treasury credited:         %s", (_treasury() - treasuryBefore) / 1e6);

        // 2. Alice closes the whole position at the same price. She is owed a ~500 VPI rebate, more than the fee.
        treasuryBefore = _treasury();
        uint256 poolBefore = pool.totalAssets();
        OrderV3Types.OrderReceipt memory receipt = _trade(alice, CfdTypes.Side.LONG, 0, true);
        console.log("2. Alice closes LONG 1,000,000");
        console.log("   Fee in the receipt:        %s", receipt.economics.executionFeeAssets / 1e6);
        console.log("   VPI rebate in the receipt: %s", uint256(-receipt.economics.vpiAssets) / 1e6);
        console.log("   Pool paid out:             %s", (poolBefore - pool.totalAssets()) / 1e6);
        console.log("   Treasury credited:         %s", (_treasury() - treasuryBefore) / 1e6);
        assertEq(receipt.economics.executionFeeAssets, 400e6, "receipt reports the full close fee");
        assertEq(_treasury() - treasuryBefore, 0, "close fee never reaches the treasury");

        // 3. Alice reopens the same LONG, and Bob opens SHORT 1,000,000 USDG into the same gap. Bob's open has the
        //    same rebate and the same fee.
        _trade(alice, CfdTypes.Side.LONG, 20_000e6, false);
        treasuryBefore = _treasury();
        poolBefore = pool.totalAssets();
        receipt = _trade(bob, CfdTypes.Side.SHORT, 20_000e6, false);
        console.log("3. Bob opens SHORT 1,000,000 into the same gap");
        console.log("   Fee in the receipt:        %s", receipt.economics.executionFeeAssets / 1e6);
        console.log("   VPI rebate in the receipt: %s", uint256(-receipt.economics.vpiAssets) / 1e6);
        console.log("   Pool paid out:             %s", (poolBefore - pool.totalAssets()) / 1e6);
        console.log("   Treasury credited:         %s", (_treasury() - treasuryBefore) / 1e6);
        assertEq(_treasury() - treasuryBefore, 400e6, "open with the same rebate credits the full fee");
    }

    function _treasury() internal view returns (uint256) {
        return clearinghouse.balanceAssets(engine.protocolTreasury());
    }

    /// Commits a 1,000,000 USDG market order through the production `OrderRouter.commitOrder(OrderRequest)` and
    /// executes it at price 1.00.
    function _trade(
        address trader,
        CfdTypes.Side side,
        uint256 margin,
        bool isClose
    ) internal returns (OrderV3Types.OrderReceipt memory receipt) {
        uint256 window = router.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode("poc", router.nextCommitId())),
            side: side,
            sizeDelta: SIZE,
            marginDelta: margin,
            targetPrice: (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1, // no price limit
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

        bytes[] memory update = _mockPythUpdateData(PRICE);
        _startRecordingLogs();
        vm.prank(keeper);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, update);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order should execute");

        receipt = _verifiedReceipt(router.lifecycleBook(), orderId);
    }

}
```

Save this code to the file `packages/perps/test/perps/poc/PoC_CloseFeeAbsorbedByVpiRebate.t.sol` and run `forge test --root packages/perps --mc PoC_CloseFeeAbsorbedByVpiRebate -vv`. It will print the following logs:

```
Ran 1 test for test/perps/poc/PoC_CloseFeeAbsorbedByVpiRebate.t.sol:PoC_CloseFeeAbsorbedByVpiRebate
[PASS] test_closeWithVpiRebate_treasuryGetsNoFee() (gas: 11666197)
Logs:
  1. Alice opens LONG 1,000,000 into a flat book
     Treasury credited:         400
  2. Alice closes LONG 1,000,000
     Fee in the receipt:        400
     VPI rebate in the receipt: 499
     Pool paid out:             99
     Treasury credited:         0
  3. Bob opens SHORT 1,000,000 into the same gap
     Fee in the receipt:        400
     VPI rebate in the receipt: 499
     Pool paid out:             499
     Treasury credited:         400

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 52.00ms (27.39ms CPU time)
```

All values are in whole USDG. In step 2 the receipt shows a 499 rebate and a 400 fee, and the pool pays out only 99, so Alice paid the fee, but the treasury is credited 0 and the pool keeps the 400. In step 3 the same rebate and fee on an open make the pool pay out 499, of which the treasury gets the full 400.
