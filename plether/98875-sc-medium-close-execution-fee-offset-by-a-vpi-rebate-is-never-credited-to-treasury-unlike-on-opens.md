# \#98875 [SC-Medium] Close execution fee offset by a VPI rebate is never credited to treasury, unlike on opens

**Submitted on Nov 10th 2026 at 17:26:08 UTC by @DiFors for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98875
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

Summary
-------
When a voluntary close earns a VPI rebate that offsets the execution fee, the planner drops the fee entirely instead of crediting it to the protocol treasury. The trader is still charged the fee (the rebate paid out is reduced by F), but the pool keeps that amount and the treasury receives nothing. The identical netting is handled correctly elsewhere: an open whose VPI rebate offsets the fee, and a close whose fee is withheld from price gain, both credit the full fee to the treasury via the pool-funded top-up. Only the close-with-rebate path silently reroutes protocol fee revenue to LPs.

Affected code (frozen commit a82d4fef)
--------------------------------------
- packages/perps/src/libraries/CfdEnginePlanLib.sol
    - _planCloseActionSettlement (L1036-1075), fee reduced at L1066-1069
    - _planIsolatedCloseSettlement fee top-up computed from the reduced fee (L986-989)
- For comparison (correct behaviour):
    - CfdEnginePlanLib._applyOpenSettlementBufferAfterPlan (L718-748): open fee shortfall is topped up
    - CfdEngineSettlementSidecar.sol L498 and _settleProtocolFeeTopUp (L905-926): open passes the FULL assessed fee

Root cause
----------
_planCloseActionSettlement nets VPI, execution fee, frozen spread and unpaid carry into one signed action amount:

    actionNetAssets = vpiDelta + executionFee + frozenSpread + unpaidCarry

If actionNetAssets < 0 it becomes an action rebate (actionRebateAssets = -actionNetAssets) and no action charge is collected or withheld. The fee is then reduced to the "eligible" collected portion (L1066-1069):

    executionFeeEligibleAssets = nonClawbackWithheldAssets + nonVpiReserveCollectedAssets;   // = 0 when net is a rebate
    delta.executionFeeAssets = min(cs.executionFeeAssets, executionFeeEligibleAssets);     // = 0

The pool-funded top-up that credits the treasury is computed from this reduced value (L986-989):

    feeTopUpAssets = delta.executionFeeAssets - delta.actionProtocolFeeCreditedAssets;      // = 0

So with a VPI rebate R and fee F:
- trader receives R - F (the fee was charged by netting),
- pool pays out only R - F (it retains F),
- treasury receives 0 (F should have been topped up, exactly as on opens).
If 0 < R < F, only F - R is credited and R of the fee is lost to the treasury. Treasury loss per close = min(R, F).

Inconsistency with the rest of the protocol
-------------------------------------------
1. Opens: when a VPI rebate offsets the fee, directlyCreditedAssets = 0, feeShortfallAssets = F and the full F is topped up from free pool cash (L718-748). CfdEngineSettlementSidecar calls _settleProtocolFeeTopUp(host, delta.executionFeeAssets, ...) with the full assessed fee (L498).
2. Closes with price gain: a fee withheld from the gain is not cash-collected either, yet it is included in feeTopUpAssets and credited to the treasury.
3. Closes with a VPI rebate: economically the same netting (the pool pays the trader F less), but the fee is zeroed before the top-up.

Documentation states the fee belongs to the treasury:
- WHITEPAPER.md L810: "Execution fee | Protocol service revenue on traded notional | Treasury, only when cash credited"
- WHITEPAPER.md L411-412: "the protocol execution fee belongs to treasury rather than LP assets"
- WHITEPAPER.md L1455: "voluntary closes pay the 4-basis-point treasury execution fee"
In this path the fee is not "uncreditable": the pool retains the cash and free pool cash is available (the same top-up succeeds on opens).

Reachability
------------
Default deployment parameters (DEPLOYMENT.md: vpiFactor = 0.01e18, 4 bps fee). Any trader who opens in the skew-increasing direction (paying VPI, accruing positive vpiAccrued) and later closes while the skew persists receives a VPI rebate on close. Whenever that rebate is >= F (position notional above ~8% of pool depth in the PoC setup) the full close fee is lost to the treasury; for smaller rebates R of the fee is lost. No special role or timing is required; this happens in ordinary trading.

Impact
------
The protocol treasury systematically does not receive execution fees that traders actually pay on skew-healing closes; the value is retained by LPs instead. No user loses funds and no value leaves the system, but the protocol fails to deliver its documented fee revenue (Contract fails to deliver promised returns, but doesn't lose value). In the PoC: $200,000 close, F = $80, R = $199.56 -> treasury credit $0 instead of $80.

Not a known issue
-----------------
Cecuro LOW-26 / MED-6 concern the OPEN path planner and capping top-ups by free pool cash (an unfundable remainder is not a treasury claim). Here the fee is fundable (free cash exists and the pool kept F) and the top-up is never attempted, only on closes. The AI-scan VPI items concern pool-depth manipulation of the rebate size, not fee routing.

Recommendation
--------------
Keep the full assessed fee for the top-up on closes, mirroring opens: compute feeTopUpAssets from cs.executionFeeAssets (minus the cash-credited part), still capped by _freePoolCashForProtocolFee, so a fee netted against a VPI rebate is credited to the treasury from the cash the pool retained.

## Link to Proof of Concept

https://gist.github.com/DiMetelnyi/4c78b390626f8c9c70ec975089d9fd4a

## Proof of Concept

Runnable Foundry PoC through the public order path (router.commitOrder -> router.executeOrder with a mock Pyth update), on the frozen commit a82d4fef. Three tests: a control without VPI, the bug, and the same netting on an open for comparison.

Setup
1. git clone https://github.com/immunefi-team/audit-comp-plether && cd audit-comp-plether && git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
2. git submodule update --init lib/forge-std lib/openzeppelin-contracts lib/morpho-blue
3. Save the file below as packages/perps/test/perps/poc/CloseFeeVpiRebatePoC.t.sol
4. forge test --root packages/perps --match-path test/perps/poc/CloseFeeVpiRebatePoC.t.sol -vv

Steps (test_Bug_CloseFeeOffsetByVpiRebate_NeverReachesTreasury)
1. Set vpiFactor = 0.01e18 (DEPLOYMENT.md value), carry 0 for exact amounts, live market time.
2. ALICE opens LONG 200,000e18 at $1 (skew 0 -> long, pays VPI).
3. engineLens.previewClose confirms the close is valid and earns VPI rebate R = 199.56 >= F = 80.
4. ALICE fully closes at the same price.
5. Assertions: trader gains R - F, pool pays only R - F, treasury credit = 0.
Control (VPI off): the same close credits F = 80 to the treasury.
Comparison: an open whose VPI rebate offsets the fee credits the full F = 80 to the treasury.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {console2} from "forge-std/console2.sol";

/// @notice PoC: on close, an execution fee offset by a VPI rebate is never credited to the treasury.
/// @dev Every order uses the public path: router.commitOrder -> router.executeOrder (mock Pyth update).
///      Scenario 1 (control, VPI disabled): the close credits the full execution fee F to the treasury.
///      Scenario 2 (bug, vpiFactor = 0.01e18 as in DEPLOYMENT.md): the same close earns rebate R >= F -> treasury gets 0,
///                 although the trader only receives R - F (the trader did pay the fee).
///      Scenario 3 (comparison): an open with rebate R >= F -> treasury gets the full F (pool-funded top-up).
contract CloseFeeVpiRebatePoC is BasePerpTest {

    address private constant ALICE_ACC = address(0xA11CE);
    address private constant BOB_ACC = address(0xB0B);
    address private constant KEEPER = address(0xBEEF);
    uint256 private constant MARK = 1e8;
    uint256 private constant SIZE = 200_000e18; // $200,000 notional at a $1 mark
    uint256 private constant MARGIN = 20_000e6;
    uint256 private constant FUNDING = 50_000e6;
    uint256 private constant LIVE_TIME = 1_729_283_399 - 1 hours; // live market, outside the FAD window (same anchor as the project matrix test)
    uint256 private constant DEPLOYMENT_VPI_FACTOR = 0.01e18; // DEPLOYMENT.md: vpiFactor = 0.01e18

    uint256 private clientNonce;

    function setUp() public override {
        super.setUp();
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.maxExecutionWindowSeconds = 1 hours;
        _setRouterConfig(config);
    }

    // ----------------------------------------------------------------------
    // Scenario 1: control without VPI - the close credits fee F to the treasury
    // ----------------------------------------------------------------------
    function test_Control_CloseWithoutVpi_CreditsTreasuryFee() public {
        _configure(0);
        _fundTrader(ALICE_ACC, FUNDING);
        _openPublic(ALICE_ACC, CfdTypes.Side.LONG, SIZE, MARGIN);

        uint256 fee = _engineExecutionFeeAssets(SIZE, MARK);
        uint256 treasuryBefore = _treasury();
        _closePublic(ALICE_ACC, CfdTypes.Side.LONG, SIZE);
        uint256 treasuryDelta = _treasury() - treasuryBefore;

        console2.log("[control] close execution fee F       :", fee);
        console2.log("[control] treasury credit on close    :", treasuryDelta);
        assertGt(fee, 0, "fee must be non-zero");
        assertEq(treasuryDelta, fee, "without VPI the close credits the full execution fee to treasury");
    }

    // ----------------------------------------------------------------------
    // Scenario 2: bug - close with VPI rebate R >= F, treasury receives 0
    // ----------------------------------------------------------------------
    function test_Bug_CloseFeeOffsetByVpiRebate_NeverReachesTreasury() public {
        _configure(DEPLOYMENT_VPI_FACTOR);
        _fundTrader(ALICE_ACC, FUNDING);
        // The open increases skew (0 -> long), pays VPI and accrues positive vpiAccrued.
        _openPublic(ALICE_ACC, CfdTypes.Side.LONG, SIZE, MARGIN);

        // Closing at the same price returns skew to zero: VPI rebate.
        ICfdEngineTypes.ClosePreview memory preview = engineLens.previewClose(ALICE_ACC, SIZE, MARK);
        assertTrue(preview.valid, "close must be valid");
        uint256 fee = _engineExecutionFeeAssets(SIZE, MARK);
        uint256 rebate = uint256(-preview.vpiDeltaAssets);
        assertLt(preview.vpiDeltaAssets, 0, "close must earn a VPI rebate");
        assertGe(rebate, fee, "scenario requires rebate R >= fee F");

        uint256 traderBefore = clearinghouse.balanceAssets(ALICE_ACC);
        uint256 poolBefore = settlementToken.balanceOf(address(pool));
        uint256 treasuryBefore = _treasury();

        _closePublic(ALICE_ACC, CfdTypes.Side.LONG, SIZE);

        uint256 traderGain = clearinghouse.balanceAssets(ALICE_ACC) - traderBefore;
        uint256 poolOutflow = poolBefore - settlementToken.balanceOf(address(pool));
        uint256 treasuryDelta = _treasury() - treasuryBefore;

        console2.log("[bug] VPI rebate R on close            :", rebate);
        console2.log("[bug] assessed execution fee F         :", fee);
        console2.log("[bug] trader received (R - F)          :", traderGain);
        console2.log("[bug] pool cash paid out               :", poolOutflow);
        console2.log("[bug] treasury credit on close         :", treasuryDelta);
        console2.log("[bug] fee kept by LPs instead of treasury:", fee - treasuryDelta);

        // The trader effectively paid the fee: the rebate is reduced by F (zero price PnL, carry disabled).
        assertEq(traderGain, rebate - fee, "trader is charged the fee: receives R - F");
        // The pool paid out only R - F, i.e. it kept F.
        assertEq(poolOutflow, rebate - fee, "pool pays only R - F and keeps F");
        // The treasury receives nothing - fee F stays with LPs.
        assertEq(treasuryDelta, 0, "BUG: treasury receives zero of the fee the trader paid");
    }

    // ----------------------------------------------------------------------
    // Scenario 3: the same netting on an open - the treasury receives the full F
    // ----------------------------------------------------------------------
    function test_Compare_OpenFeeOffsetByVpiRebate_IsCreditedToTreasury() public {
        _configure(DEPLOYMENT_VPI_FACTOR);
        _fundTrader(BOB_ACC, FUNDING);
        _fundTrader(ALICE_ACC, FUNDING);
        // BOB creates short skew.
        _openPublic(BOB_ACC, CfdTypes.Side.SHORT, SIZE, MARGIN);

        // ALICE opens long, reducing skew: VPI rebate on the open (R >= F).
        uint256 fee = _engineExecutionFeeAssets(SIZE, MARK);
        uint256 treasuryBefore = _treasury();
        _openPublic(ALICE_ACC, CfdTypes.Side.LONG, SIZE, MARGIN);
        uint256 treasuryDelta = _treasury() - treasuryBefore;

        console2.log("[open] execution fee F                  :", fee);
        console2.log("[open] treasury credit on rebate open   :", treasuryDelta);
        assertEq(treasuryDelta, fee, "open with an offsetting VPI rebate still credits the full fee to treasury");
    }

    // ---------------------------- helpers ----------------------------

    function _configure(
        uint256 vpiFactor
    ) private {
        CfdTypes.RiskParams memory params = _riskParams();
        params.vpiFactor = vpiFactor;
        params.baseCarryBps = 0; // carry disabled so every amount is exact
        _setRiskParams(params);
        vm.warp(LIVE_TIME);
        vm.prank(address(router));
        engine.updateMarkPrice(MARK, uint64(vm.getBlockTimestamp()));
    }

    function _treasury() private view returns (uint256) {
        return clearinghouse.balanceAssets(engine.protocolTreasury());
    }

    function _openPublic(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin
    ) private {
        _commitAndExecute(account, _request(side, size, margin, false));
        (uint256 posSize,,,,,,) = engine.positions(account);
        assertGt(posSize, 0, "open must execute");
    }

    function _closePublic(
        address account,
        CfdTypes.Side side,
        uint256 size
    ) private {
        _commitAndExecute(account, _request(side, size, 0, true));
        (uint256 posSize,,,,,,) = engine.positions(account);
        assertEq(posSize, 0, "full close must execute");
    }

    function _commitAndExecute(
        address account,
        OrderV3Types.OrderRequest memory request
    ) private {
        vm.prank(account);
        uint64 id = router.commitOrder(request);
        bytes[] memory data = _mockPythUpdateData(MARK);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order must execute");
    }

    function _request(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) private returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = bytes32(uint256(0xC0FFEE00) + (++clientNonce));
        request.side = side;
        request.targetPrice = MARK;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.isClose = isClose;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + 1 hours),
            executionWindowSeconds: 3600,
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
        });
    }

}
```

Output (forge 1.5.1, solc 0.8.35)
```
Ran 3 tests for test/perps/poc/CloseFeeVpiRebatePoC.t.sol:CloseFeeVpiRebatePoC
[PASS] test_Bug_CloseFeeOffsetByVpiRebate_NeverReachesTreasury() (gas: 4463838)
Logs:
  [bug] VPI rebate R on close            : 199561045
  [bug] assessed execution fee F         : 80000000
  [bug] trader received (R - F)          : 119561045
  [bug] pool cash paid out               : 119561045
  [bug] treasury credit on close         : 0
  [bug] fee kept by LPs instead of treasury: 80000000

[PASS] test_Compare_OpenFeeOffsetByVpiRebate_IsCreditedToTreasury() (gas: 5104844)
Logs:
  [open] execution fee F                  : 80000000
  [open] treasury credit on rebate open   : 80000000

[PASS] test_Control_CloseWithoutVpi_CreditsTreasuryFee() (gas: 4275934)
Logs:
  [control] close execution fee F       : 80000000
  [control] treasury credit on close    : 80000000

Suite result: ok. 3 passed; 0 failed; 0 skipped
```
