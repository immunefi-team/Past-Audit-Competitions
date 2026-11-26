# \#99592 [SC-Low] Plether: any ordinary order fill makes creating or replacing a stop-loss or take-profit revert until someone pays for a live mark refresh

**Submitted on Nov 14th 2026 at 11:58:21 UTC by @SombraX for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99592
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

# Any ordinary order fill makes creating or replacing a stop-loss or take-profit revert until someone pays for a live mark refresh

**Severity:** Low

**Impact:** Contract fails to deliver promised returns, but doesn't lose value

**Asset:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src (`PositionProtectionBook.sol`, `PletherOracle.sol`)

**Pin:** `a82d4fefb02768e5b3bddcc97dee371eee24fea8`

## Severity note

Filed as Low: take-profit and stop-loss are an active launch feature, and after any order fill they cannot be created or changed. The same behaviour matches the Medium row "Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)": one small fill by any account, about 0.45 USDG in the PoC, blocks every trader's protection creation again after each refresh. We leave the final level to triage.

## Eligibility under the known-issues rule

Cecuro MED-17 and MED-28 are marked Acknowledged in the listed audit report and the vendor comment on both says "Addressed", with fix commits `8f6cfd3d` (protection validation) and `81919218` (Oracle/Engine synchronization). The program states that for such findings "a report on the same root cause is valid only if it shows that the fix at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8 is incomplete or introduces a new bug". This report is the second case. It does not restate MED-17 or MED-28. It shows that their fix, as present at the pin, makes protection creation, replacement and open-with-protection revert after every ordinary order fill, a state that did not block these functions before the fix.

## What is and is not claimed

Claimed: after the fix for Cecuro MED-17 and MED-28, `createPositionProtection`, `replacePositionProtection` and `commitOpenOrderWithProtection` read the oracle's accepted live basket. Every ordinary delayed-order fill moves the Engine mark forward without moving that accepted basket, and from then on all three functions revert with `PletherOracle__PriceOutOfOrder` for every account. Time does not heal it. It ends only when someone submits a paid live mark refresh, and the next fill by anyone starts it again.

Not claimed: theft or a freeze of funds. Existing armed protections still trigger, and ordinary closes still work. A trader who refreshes the mark and creates the protection before another fill lands succeeds.

## Summary

Before the fix, protection creation validated against the Engine's cached mark. The fix switched creation, replacement and attached-open staging to `oracle.getLatestPrice(MarkRefresh)`, the oracle's accepted live basket. That view enforces publish order against the Engine: it reverts when the accepted basket is older than `engine.lastMarkTime()`.

Order execution prices each order at a historical tick after its commit and installs that tick as the Engine mark when it is newer. It does not update the oracle's accepted live basket. So the normal sequence "live refresh, then an order fills" leaves the Engine mark newer than the accepted basket, and the protection functions revert.

The most common user flow hits this directly: open a position, then set a stop-loss. The trader's own fill is what blocks the stop-loss.

## Root cause

- `PositionProtectionBook.sol:862-886` (`_freshProtectionObservation`), used by create (`:229`), replace (`:295`) and attached-open staging (`:463`), calls `oracle.getLatestPrice(IPletherOracle.PriceMode.MarkRefresh)` at `:880`.
- `PletherOracle.sol:571-580` builds that snapshot from `_acceptedLiveBasket` with publish-order enforcement, and `:892-895` reverts `PletherOracle__PriceOutOfOrder(snapshot.publishTime, markTime)` when the accepted basket is older than the Engine mark.
- `OrderRouterExecutionSidecar.sol:1053-1061` (`_updateEngineMark`): a fill installs its historical tick as the Engine mark when it is newer. The accepted live basket is not touched.

The project's own monitoring note records the divergence without connecting it to protections: `ORACLE_SYNCHRONIZATION.md:150-151`, "historical execution may advance the Engine without advancing the accepted live basket".

## Path

1. Anyone calls `router.updateMarkPrice(data)`. Accepted basket and Engine mark are both at time T0.
2. Any account commits an open; a keeper executes it with a tick after T0. The Engine mark is now at T1 > T0. The accepted basket is still at T0.
3. `createPositionProtection`, `replacePositionProtection` and `commitOpenOrderWithProtection` revert `PletherOracle__PriceOutOfOrder(T0, T1)` for every account, including the account whose order just filled.
4. Waiting changes nothing. A new paid `updateMarkPrice` with newer data restores the functions until the next fill by any account.

## Measurements

Release risk parameters, public entrypoints only.

| Test | Result |
|---|---|
| Control: open, live refresh, create | creation succeeds |
| Open, then create | reverts; accepted basket time 1709535601, Engine mark time 1709535602; still reverts 20 s later; succeeds after a live refresh |
| Another account's fill | the first account's replace reverts, and a third account's open-with-protection reverts; both succeed after a refresh |
| Ten rounds: refresh, one 1,000-token fill by another account, create | creation reverts in 10 of 10 rounds; the other account's balance falls by 4.50 USDG in total and it keeps the position it built |

## Impact

- A trader cannot place or change a stop-loss or take-profit after any fill, their own included, without first paying for a separate mark-refresh transaction, and loses that again if any fill lands between the refresh and the create. Each fill by any account starts a new blocked period that lasts until the next live refresh, so how often the feature is unavailable depends on how often orders fill relative to live refreshes; we did not measure that on a live market. An account that cannot send the refresh and the create atomically cannot rule out a fill landing between them.
- `commitOpenOrderWithProtection` is blocked the same way, so a position cannot be opened with its protection attached.
- A third party can keep it blocked on purpose for the cost of small fills (0.45 USDG per round in the PoC).
- The position stays open and unprotected during an adverse move. No funds are frozen or taken.

## Why this is not intended and not a known issue

- `SECURITY.md:563`: "Existing-position protection creation remains available in degraded mode because it adds no exposure." Creation is meant to stay available; here it is unavailable in the healthy state after routine activity.
- The vendor's MED-28 note says the guard "rejects arming when the accepted oracle mark is ahead of, or inconsistent with, the Engine mark. A qualifying mark must be installed before arming can proceed." That describes the oracle being ahead. The vendor tests reach the guard only that way, by calling the oracle directly as the router (`test/perps/spec/trader/PositionProtection.t.sol:276-289`). The state reported here is the opposite one, the Engine ahead, and it is produced by every order fill. No vendor test opens through the router and then creates a protection without a refresh in between.
- The project has already accepted the same impact as a finding. Cecuro LOW-73, "Trader cannot add new stop-loss protection while degraded", was fixed so that "A trader can arm protection for an existing LONG or SHORT ... while degraded", with the stated aim to preserve "the ability to protect an existing position". Here the same ability is lost in the healthy state, after routine activity.
- Not in the Cecuro report, the SC Audit Studio pre-audit, the AI scan or the listed `SECURITY.md` sections. The AI scan item "Later mark refresh can make live historical-price orders permanently out of order" is the mirror case (a refresh blocking an order). The public MED-18 same-second mark stall concerns mark refresh at an equal timestamp, a different check.

## Recommended fix

In `_freshProtectionObservation`, when the Engine mark is fresh and newer than the accepted basket, validate against the Engine mark and time, as the pre-fix code did, instead of reverting. Alternatively have order execution advance the oracle's accepted observation together with the Engine mark.



## Link to Proof of Concept

https://gist.github.com/dexloom/4faf98791ee49e434e8a2afcb1a47633

## Proof of Concept

## Proof of concept

Secret gist with README and test: https://gist.github.com/dexloom/4faf98791ee49e434e8a2afcb1a47633

```
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
# save the test below as packages/perps/test/perps/ProtectionCreateBlockedAfterFill.t.sol
cd packages/perps
forge test --match-path 'test/perps/ProtectionCreateBlockedAfterFill.t.sol' -vv
```

Output at the pin:

```
[PASS] test_control_createAfterLiveRefresh()
[PASS] test_createRevertsAfterOrdinaryFill()
  accepted live basket time: 1709535601
  engine mark time: 1709535602
[PASS] test_sustained_oneSmallFillPerRefreshKeepsCreationBlocked()
  rounds blocked: 10
  other account position (tokens, 18dp): 10000000000000000000000
  other account balance before (6dp): 50000000000
  other account balance after  (6dp): 49995499001
[PASS] test_thirdPartyFillBlocksCreateReplaceAndAttachedOpen()
Suite result: ok. 4 passed; 0 failed; 0 skipped
```

Test:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice Protection creation, replacement and open-with-protection read the oracle's accepted live basket
///         (`getLatestPrice(MarkRefresh)`), which reverts with PriceOutOfOrder whenever an ordinary order fill
///         (historical tick) has moved the Engine mark past the oracle's last accepted live basket.
///         Release risk parameters; every step uses public entrypoints; only Pyth is mocked.
contract ProtectionCreateBlockedAfterFillTest is BasePerpTest {

    address internal constant ALICE = address(0xA11CE);
    address internal constant BOB = address(0xB0B);
    address internal constant KEEPER = address(0xCAFE01);

    IPositionProtectionActions internal prot;
    IPositionProtectionViews internal protViews;
    IOrderLifecycleBook internal book;

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
        prot = IPositionProtectionActions(address(router.positionProtectionBook()));
        protViews = IPositionProtectionViews(address(router.positionProtectionBook()));
        book = IOrderLifecycleBook(address(router.lifecycleBook()));
        _fundTrader(ALICE, 50_000e6);
        _fundTrader(BOB, 50_000e6);
        _refreshLive(1e8);
    }

    function _liveData(
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

    function _refreshLive(
        uint256 price
    ) internal {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory data = _liveData(price);
        router.updateMarkPrice(data);
    }

    function _openReq(
        CfdTypes.Side side,
        uint256 lots,
        uint256 margin
    ) internal view returns (OrderV3Types.OrderRequest memory q) {
        q.clientOrderId = bytes32(uint256(keccak256(abi.encode("m17", router.nextCommitId()))) >> 8);
        q.side = side;
        q.sizeDelta = lots * 1e20;
        q.marginDelta = margin;
        q.targetPrice = side == CfdTypes.Side.LONG ? 1 : type(uint256).max;
        OrderV3Types.ExecutionBounds memory b;
        b.submitBy = uint64(vm.getBlockTimestamp() + 30);
        b.executionWindowSeconds = 60;
        b.allowedExecutionModes = 7;
        b.expectedConfigHash = book.currentExecutionConfigHash();
        b.maxExecutionBountyAssets = type(uint256).max;
        b.maxExecutionNotionalAssets = type(uint256).max;
        b.maxGrossAccountDebitAssets = type(uint256).max;
        b.maxActionChargeAssets = type(uint256).max;
        b.maxExplicitFeesAssets = type(uint256).max;
        b.maxPostPositionSize = type(uint256).max;
        b.maxPostLeverageBps = type(uint32).max;
        q.bounds = b;
    }

    function _params(
        uint256 tp,
        uint256 sl
    ) internal pure returns (PositionProtectionTypes.PositionProtectionParams memory p) {
        p.takeProfitTriggerPrice = tp;
        p.stopLossTriggerPrice = sl;
        p.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    function _outOfOrder() internal view returns (bytes memory) {
        (,, uint64 acceptedTime) = router.pletherOracle().getAcceptedBasketMark();
        return abi.encodeWithSelector(
            IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acceptedTime, engine.lastMarkTime()
        );
    }

    function _fillOpen(
        address who,
        CfdTypes.Side side
    ) internal {
        OrderV3Types.OrderRequest memory q = _openReq(side, 100, 500e6);
        vm.prank(who);
        uint64 id = router.commitOrder(q);
        vm.warp(vm.getBlockTimestamp() + 2);
        bytes[] memory data = _mockPythUpdateData(1e8);
        vm.prank(KEEPER);
        router.executeOrder(id, data);
        (uint256 size,,,,,,) = engine.positions(who);
        assertEq(size, 100 * 1e20, "open must fill");
    }

    /// @dev Control: position created, live mark refreshed afterwards -> protection can be created.
    function test_control_createAfterLiveRefresh() public {
        _fillOpen(ALICE, CfdTypes.Side.LONG);
        _refreshLive(1e8);
        vm.prank(ALICE);
        uint64 pid = prot.createPositionProtection(_params(0.9e8, 1.1e8));
        assertGt(pid, 0);
    }

    /// @dev After an ordinary delayed fill the Engine mark is newer than the oracle's accepted live basket.
    ///      Stand-alone protection creation then reverts inside the oracle view.
    function test_createRevertsAfterOrdinaryFill() public {
        _fillOpen(ALICE, CfdTypes.Side.LONG);
        (bool accepted,, uint64 acceptedTime) = router.pletherOracle().getAcceptedBasketMark();
        assertTrue(accepted);
        emit log_named_uint("accepted live basket time", acceptedTime);
        emit log_named_uint("engine mark time", engine.lastMarkTime());
        assertLt(acceptedTime, engine.lastMarkTime(), "fill advanced the Engine past the accepted basket");

        bytes memory expected = _outOfOrder();
        vm.expectRevert(expected);
        vm.prank(ALICE);
        prot.createPositionProtection(_params(0.9e8, 1.1e8));

        // time passing does not heal it (until the mark goes stale, which is a different revert)
        vm.warp(vm.getBlockTimestamp() + 20);
        vm.roll(vm.getBlockNumber() + 1);
        vm.expectRevert(expected);
        vm.prank(ALICE);
        prot.createPositionProtection(_params(0.9e8, 1.1e8));

        // only a new live mark refresh heals it
        _refreshLive(1e8);
        vm.prank(ALICE);
        uint64 pid = prot.createPositionProtection(_params(0.9e8, 1.1e8));
        assertGt(pid, 0);
    }

    /// @dev Any other account's fill re-blocks: BOB's unrelated order execution makes ALICE's
    ///      create / replace / attached-open-with-protection revert again.
    function test_thirdPartyFillBlocksCreateReplaceAndAttachedOpen() public {
        _fillOpen(ALICE, CfdTypes.Side.LONG);
        _refreshLive(1e8);
        vm.prank(ALICE);
        uint64 pid = prot.createPositionProtection(_params(0.9e8, 1.1e8));

        // BOB trades; nothing about ALICE changes
        _fillOpen(BOB, CfdTypes.Side.SHORT);

        bytes memory expected = _outOfOrder();
        vm.expectRevert(expected);
        vm.prank(ALICE);
        prot.replacePositionProtection(pid, _params(0.95e8, 1.05e8));

        // a fresh account cannot place an open with attached protection either
        address carol = address(0xCA201);
        _fundTrader(carol, 50_000e6);
        OrderV3Types.OrderRequest memory q = _openReq(CfdTypes.Side.LONG, 100, 500e6);
        vm.expectRevert(expected);
        vm.prank(carol);
        prot.commitOpenOrderWithProtection(q, _params(0.9e8, 1.1e8));

        // control: the same calls pass right after a live refresh
        _refreshLive(1e8);
        vm.prank(ALICE);
        prot.replacePositionProtection(pid, _params(0.95e8, 1.05e8));
        q = _openReq(CfdTypes.Side.LONG, 100, 500e6);
        vm.prank(carol);
        prot.commitOpenOrderWithProtection(q, _params(0.9e8, 1.1e8));
    }

    /// @dev Sustained case: each time ALICE pays for a live refresh, one small fill by another account lands before
    ///      her create and blocks it again. Ten rounds; ALICE never gets a protection. The other account's cost is
    ///      measured as its settlement balance change.
    function test_sustained_oneSmallFillPerRefreshKeepsCreationBlocked() public {
        _fillOpen(ALICE, CfdTypes.Side.LONG);
        uint256 bobBefore = clearinghouse.balanceAssets(BOB);
        uint256 blocked;
        for (uint256 round; round < 10; ++round) {
            _refreshLive(1e8); // ALICE (or anyone) pays for a live refresh
            // another account's 1,000-token open or increase fills before ALICE's second transaction
            OrderV3Types.OrderRequest memory q = _openReq(CfdTypes.Side.SHORT, 10, 50e6);
            vm.prank(BOB);
            uint64 id = router.commitOrder(q);
            vm.warp(vm.getBlockTimestamp() + 2);
            bytes[] memory data = _mockPythUpdateData(1e8);
            vm.prank(KEEPER);
            router.executeOrder(id, data);

            bytes memory expected = _outOfOrder();
            vm.expectRevert(expected);
            vm.prank(ALICE);
            prot.createPositionProtection(_params(0.9e8, 1.1e8));
            ++blocked;
        }
        assertEq(blocked, 10);
        (uint256 bobSize,,,,,,) = engine.positions(BOB);
        emit log_named_uint("rounds blocked", blocked);
        emit log_named_uint("other account position (tokens, 18dp)", bobSize);
        emit log_named_uint("other account balance before (6dp)", bobBefore);
        emit log_named_uint("other account balance after  (6dp)", clearinghouse.balanceAssets(BOB));
    }
}
```

