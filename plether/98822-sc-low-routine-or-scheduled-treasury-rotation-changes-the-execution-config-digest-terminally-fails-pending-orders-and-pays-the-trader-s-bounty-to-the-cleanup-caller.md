# \#98822 [SC-Low] Routine or scheduled treasury rotation changes the execution-config digest, terminally fails pending orders, and pays the trader's bounty to the cleanup caller

**Submitted on Nov 10th 2026 at 12:01:07 UTC by @midkernel for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98822
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

`OrderLifecycleBook.currentExecutionConfigHash()` pins `engine.protocolTreasury()`, a payout recipient that is not an execution parameter and that changes on documented treasury operations, including an automatic successor activation at a lock expiry with no transaction. Every pending config-pinned order therefore commits a digest that becomes stale after a routine or scheduled treasury rotation. On the next execution attempt `OrderRouterExecutionSidecar` terminalizes the order as `ConfigMismatch`; for open orders `OrderRouter` pays the trader's reserved execution bounty to whoever performed the cleanup instead of executing the order, and for risk-reducing close orders the committed intent is discarded and the position stays open. The behavior is reproduced on the pinned release with four passing Foundry tests that call the production `OrderRouter`, `CfdEngine` and `MarginClearinghouse` entry points.

## Vulnerability Details

Affected release: `immunefi-team/audit-comp-plether` at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. The scope includes `packages/perps/src/OrderLifecycleBook.sol`, `packages/perps/src/CfdTreasury.sol`, `packages/perps/src/OrderRouter.sol` and `packages/perps/src/OrderRouterExecutionSidecar.sol`.

Root cause, source evidence. `OrderLifecycleBook.currentExecutionConfigHash()` hashes a fixed set of dependency identities and finalized admin versions, and it includes the current protocol treasury plus the pool mark staleness limit:

```solidity
bytes memory engineConfig = abi.encode(
    engine.planner(),
    engine.settlementSidecar(),
    engine.terminalNavBook(),
    engineAdmin,
    IOrderLifecycleAdminConfigView(engineAdmin).activeConfigVersion(),
    engine.protocolTreasury(),
    IOrderLifecyclePoolConfigView(HOUSE_POOL).markStalenessLimit()
);
```

The same value is committed at order submission: a fresh external request must supply the current nonzero hash, and the Router stores it in `bounds.expectedConfigHash` (documented in `SECURITY.md`, execution bounds section).

The treasury is not an execution input. `CfdTreasury` exposes two documented owner operations that change it:

1. `setProtocolTreasury(address)` rotates the treasury immediately whenever the current lock has expired. It requires no proposal and no timelock wait.
2. `proposeProtocolTreasury(address,uint256)` queues a successor during an active lock. `_activeTreasury()` resolves the successor by timestamp, so once `block.timestamp >= _currentTreasury.lockedUntil` every read of `protocolTreasury()` returns the successor even though no transaction has executed. `SECURITY.md` states this activation occurs "without a confirmation or keeper transaction".

Because the treasury is hashed without a separate config version, either operation changes `currentExecutionConfigHash()`.

Terminal path, source evidence. On execution `OrderRouterExecutionSidecar` compares the observed digest with the pinned one:

```solidity
bytes32 observedConfigHash = request.action == IOrderRouterExecutionHost.ItemAction.Execute
    && request.observedConfigHash != bytes32(0)
    ? request.observedConfigHash
    : book.currentExecutionConfigHash();
bytes32 expectedConfigHash = pending.bounds.expectedConfigHash;
if (expectedConfigHash != bytes32(0) && observedConfigHash != expectedConfigHash) {
    return _settleNonEngine(configRequest, OrderV3Types.TerminalReason.ConfigMismatch, ...);
}
```

The order is deleted as `Failed`. `OrderRouter._settleTerminalBounty` then reads the reserved bounty and pays it to `bountyRecipient`, which is the executor for a non-self cleanup:

```solidity
uint256 bountyAssets =
    clearinghouse.getBountyReservation(IMarginClearinghouse.BountyKind.Order, order.orderId).amountAssets;
...
if (bountyAssets != 0) {
    settlement.bountyRecipient = bountyRecipient;
    settlement.bountyDisposition = OrderV3Types.BountyDisposition.Paid;
    if (bountyRecipient == order.account) {
        clearinghouse.releaseReservedExecutionBountyToSource(order.account, bountyAssets);
    } else {
        engine.creditBounty(order.account, bountyRecipient, bountyAssets, accountingPrice, accountingPublishTime);
    }
}
```

A separate configuration bump is used elsewhere: finalizing router or oracle configuration increments `activeConfigVersion`, which intentionally invalidates config-pinned orders because the execution policy changed. A treasury rotation changes no execution policy, yet it reaches the same terminal path through the raw address in the digest.

Prerequisites, stated as positive conditions. The Engine owner performs a documented treasury rotation: an immediate `setProtocolTreasury` while the lock is unlocked, or a `proposeProtocolTreasury` whose successor is scheduled to activate at the current lock expiry. At least one config-pinned order is pending when the digest changes. A caller then executes or clears the order. The owner acts within documented procedures and needs no malicious intent. Liquidation closes and protection-trigger closes use `expectedConfigHash == 0` and are unaffected.

Runtime evidence. The four tests deploy the unmodified production contracts and exercise the real Router, Engine, Sidecar and Clearinghouse boundary. They assert the digest changes after an immediate rotation and after a timestamp-only successor activation, that the terminal reason is `ConfigMismatch`, that no position opens, that the committed margin is refunded, that the cleanup caller's clearinghouse balance increases by exactly the trader's reserved execution bounty, that the treasury balance does not move, that a pending risk-reducing close leaves the position open, and that an attached `PendingOpen` protection is terminally failed with its trigger bounty refunded to the trader.

## Impact Details

Selected program impact: Contract fails to deliver promised returns, but doesn't lose value (Low).

A trader commits and funds an order, and the Router reserves an execution bounty so that a keeper will execute it. After a documented treasury rotation or an automatic successor activation, the contract refuses to deliver that execution and instead terminalizes the order as a configuration mismatch. The trader's committed margin is refunded, but the execution they paid for does not occur. The reserved execution bounty is credited to the cleanup caller rather than used for the promised execution. A pending risk-reducing close is discarded and the position remains open, so the trader must notice and resubmit.

This is griefing-shaped behavior, but the demonstrated consequence stays in the Low tier because no principal, LP backing, or unclaimed yield is lost, the committed margin is returned, and the trader can resubmit. The only value that moves is the bounded reserved execution bounty, and the trader avoids that redirect by performing the terminal cleanup themselves, which routes the reservation back to their own account through the `bountyRecipient == order.account` branch.

Material limits, stated as positive conditions. The trigger is a legitimate documented owner action, not a malicious or out-of-procedure one. The activation time of a scheduled successor is public through `pendingProtocolTreasury()` and `protocolTreasuryLockedUntil()`, so the window is predictable. Only orders with a nonzero committed digest are affected; liquidation and protection-trigger closes are not. Damage scales with the number of pending orders up to the per-call cleanup bound. The digest composition, including the treasury, is described in `SECURITY.md`; the demonstrated defect is that pinning a non-execution recipient converts an ordinary rotation into a terminal delivery failure and a bounty redirect.

## References

- `OrderLifecycleBook.currentExecutionConfigHash()` at the affected revision: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderLifecycleBook.sol#L172-L202
- `CfdTreasury` immediate rotation, proposal and timestamp activation: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdTreasury.sol#L70-L160
- `OrderRouterExecutionSidecar` digest comparison and `ConfigMismatch`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L398-L414
- `OrderRouter._settleTerminalBounty` bounty disposition: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L314-L344
- `SECURITY.md` protocol treasury commitment and execution-config digest: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md
- Program scope and impact taxonomy: https://immunefi.com/audit-competition/plether/


## Proof of Concept

## Proof of Concept

> Warning: run this proof only in a disposable local checkout of the pinned release. It deploys contracts and moves test balances on a local EVM. Do not point it at a live network and do not use real keys.

### Affected and tested revisions

- Affected release: `immunefi-team/audit-comp-plether` at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.
- Tested release: the same commit, unmodified production sources under `packages/perps/src`.
- Toolchain: Foundry `forge 1.8.3` (`cae51ad458f6abb64852b7709eb784352429825d`), solc `0.8.35`, optimizer enabled with 200 runs.
- Build profile: the pinned `packages/perps/foundry.toml` sets `via_ir = true`. The primary run used that profile with a warm build cache. The fallback run used `FOUNDRY_VIA_IR=false`; both lanes are recorded below.

### Prerequisites and topology

- A Linux host with Foundry 1.8.3 installed and network access to clone the pinned repository.
- A disposable working copy of `packages/perps` from the affected commit.
- The test file below is saved at `packages/perps/test/reviewer-04/TreasuryConfigTransition.t.sol`.
- The test extends `BasePerpTest` from the same pinned release at `packages/perps/test/perps/BasePerpTest.sol`. That file performs the deployment used by the release test suite. It deploys the unmodified production contracts `CfdEngine`, `CfdEngineAdmin`, `CfdEnginePlanner`, `CfdEngineSettlementSidecar`, `OrderLifecycleBook`, `OrderRouter`, `OrderRouterExecutionSidecar`, `MarginClearinghouse`, `HousePool`, two `TrancheVault` instances, `PletherOracle`, `PositionProtectionBook` and `TerminalNavBookV2`.
- External stand-ins are limited to `MockSettlementToken` (the USDG-like settlement asset) and `MockPyth` (the signed price source). No production contract that is exercised by the finding is mocked. `OrderRouter` is deployed through `LegacyOrderRouterHarness` from the same release, which forwards the real `OrderRouter` entry points used here.
- The test calls the production external boundary: `router.commitOrder`, `router.executeOrder`, `router.updateMarkPrice`, `protectionActions.commitOpenOrderWithProtection`, `engine.setProtocolTreasury`, `engine.proposeProtocolTreasury`, `clearinghouse.getAccountAssetBuckets` and `clearinghouse.balanceAssets`.

Fixture data, identified as fixture and not part of the attack:

- `ALICE` is the trader account funded with `300_000e6` settlement units.
- `KEEPER` is the cleanup caller.
- The mark price starts at `1e8`, and the Engine cap is `2e8`.
- `MockPyth` feeds are advanced to produce a fresh mark for the attached protection case.
- Treasury addresses used in the test are fixture addresses chosen only to be nonzero and distinct from the current treasury.

### Setup

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
cd packages/perps
mkdir -p test/reviewer-04
```

Save the complete source below as `packages/perps/test/reviewer-04/TreasuryConfigTransition.t.sol` from the working directory `packages/perps`. No edits to production sources are required.

### Proof of concept source

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {IPerpsKeeper} from "@plether/perps/interfaces/IPerpsKeeper.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice PoC for reviewer-04 slice plether-admin-treasury-transition.
/// @dev Shows that an ordinary protocol-treasury rotation changes the execution-config digest and
///      terminally invalidates pending orders, paying the trader's execution bounty to the cleaner.
contract TreasuryConfigTransitionTest is BasePerpTest {

    address internal constant ALICE = address(0xA11CE);
    address internal constant KEEPER = address(0xE0EC);
    uint256 internal constant MARK_PRICE = 1e8;

    function setUp() public override {
        super.setUp();
        _fundTrader(ALICE, 300_000e6);
    }

    function test_TreasuryRotation_ChangesConfigHash_AndInvalidatesPendingOpen() public {
        bytes32 hashBefore = router.lifecycleBook().currentExecutionConfigHash();
        uint64 orderId = _commitOpen(ALICE, CfdTypes.Side.LONG, 10_000e18, 2_000e6);
        uint256 bounty = _executionBountyReserve(orderId);
        assertGt(bounty, 0, "open order must reserve an execution bounty");

        address nextTreasury = address(0xFEE50002);
        assertTrue(nextTreasury != engine.protocolTreasury());
        engine.setProtocolTreasury(nextTreasury);

        bytes32 hashAfter = router.lifecycleBook().currentExecutionConfigHash();
        assertTrue(hashBefore != hashAfter, "treasury rotation must change the execution config digest");

        uint256 aliceFreeBefore = clearinghouse.getAccountAssetBuckets(ALICE).freeSettlementAssets;
        uint256 keeperBefore = clearinghouse.balanceAssets(KEEPER);
        uint256 treasuryBefore = clearinghouse.balanceAssets(engine.protocolTreasury());
        (uint256 sizeBefore,,,,,,) = engine.positions(ALICE);

        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, new bytes[](0));

        assertEq(uint256(result.status), uint256(OrderV3Types.LifecycleStatus.Failed), "pending open is terminalized");
        assertEq(
            uint256(result.terminalReason),
            uint256(OrderV3Types.TerminalReason.ConfigMismatch),
            "treasury rotation is reported as configuration mismatch"
        );
        (uint256 sizeAfter,,,,,,) = engine.positions(ALICE);
        assertEq(sizeAfter, sizeBefore, "no position opens");
        assertEq(
            clearinghouse.getAccountAssetBuckets(ALICE).freeSettlementAssets - aliceFreeBefore,
            2_000e6,
            "committed margin is refunded"
        );
        assertEq(clearinghouse.balanceAssets(KEEPER) - keeperBefore, bounty, "cleaner receives the trader's bounty");
        assertEq(clearinghouse.balanceAssets(engine.protocolTreasury()), treasuryBefore, "no fee movement");
    }

    function test_ScheduledTreasuryActivation_InvalidatesOrdersWithoutAnyTransaction() public {
        engine.setProtocolTreasury(address(0xFEE50010), 30 seconds);
        uint256 activation = engine.protocolTreasuryLockedUntil();
        engine.proposeProtocolTreasury(address(0xFEE50011), 0);

        uint64 orderId = _commitOpen(ALICE, CfdTypes.Side.LONG, 10_000e18, 2_000e6);
        bytes32 hashBefore = router.lifecycleBook().currentExecutionConfigHash();

        vm.warp(activation);
        assertEq(engine.protocolTreasury(), address(0xFEE50011), "successor activates by timestamp");
        bytes32 hashAfter = router.lifecycleBook().currentExecutionConfigHash();
        assertTrue(hashBefore != hashAfter, "automatic activation changes the digest with no transaction");

        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, new bytes[](0));
        assertEq(
            uint256(result.terminalReason),
            uint256(OrderV3Types.TerminalReason.ConfigMismatch),
            "scheduled treasury activation invalidates the pending order"
        );
    }

    function test_TreasuryRotation_InvalidatesPendingRiskReducingClose() public {
        _open(ALICE, CfdTypes.Side.LONG, 20_000e18, 5_000e6, MARK_PRICE);
        (uint256 sizeBefore,,,,,,) = engine.positions(ALICE);
        assertGt(sizeBefore, 0);

        uint64 closeId = _commitClose(ALICE, CfdTypes.Side.LONG, 20_000e18);
        engine.setProtocolTreasury(address(0xFEE50020));

        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(closeId, new bytes[](0));
        assertEq(
            uint256(result.terminalReason),
            uint256(OrderV3Types.TerminalReason.ConfigMismatch),
            "risk-reducing close is terminally dropped"
        );
        (uint256 sizeAfter,,,,,,) = engine.positions(ALICE);
        assertEq(sizeAfter, sizeBefore, "position remains open after the close is discarded");
        assertEq(
            uint256(router.lifecycleBook().lifecycleStatus(closeId)),
            uint256(OrderV3Types.LifecycleStatus.Failed),
            "close order is permanently terminal"
        );
    }

    function test_TreasuryRotation_FailsAttachedPendingOpenProtection() public {
        _refreshMark(MARK_PRICE);
        IPositionProtectionActions protectionActions =
            IPositionProtectionActions(address(router.positionProtectionBook()));
        IPositionProtectionViews protectionViews = IPositionProtectionViews(address(router.positionProtectionBook()));

        OrderV3Types.OrderRequest memory request = _boundedOpenRequest(CfdTypes.Side.LONG, 10_000e18, 2_000e6);
        vm.prank(ALICE);
        (uint64 parentId, uint64 protectionId) = protectionActions.commitOpenOrderWithProtection(
            request,
            PositionProtectionTypes.PositionProtectionParams({
                takeProfitTriggerPrice: 90_000_000,
                stopLossTriggerPrice: 110_000_000,
                closeLimits: PositionProtectionTypes.CloseFinancialLimits({
                    maxExecutionNotionalAssets: type(uint256).max,
                    maxGrossAccountDebitAssets: type(uint256).max,
                    maxActionChargeAssets: type(uint256).max,
                    maxExplicitFeesAssets: type(uint256).max
                })
            })
        );
        assertEq(
            uint256(protectionViews.getPositionProtection(protectionId).status),
            uint256(PositionProtectionTypes.PositionProtectionStatus.PendingOpen)
        );
        uint256 protectionBounty = protectionViews.getPositionProtection(protectionId).triggerBountyAssets;
        assertGt(protectionBounty, 0, "protection must lock a trigger bounty");
        uint256 aliceFreeBefore = clearinghouse.getAccountAssetBuckets(ALICE).freeSettlementAssets;

        engine.setProtocolTreasury(address(0xFEE50030));

        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(parentId, new bytes[](0));
        assertEq(
            uint256(result.terminalReason),
            uint256(OrderV3Types.TerminalReason.ConfigMismatch),
            "attached parent open is invalidated"
        );
        assertEq(
            uint256(protectionViews.getPositionProtection(protectionId).status),
            uint256(PositionProtectionTypes.PositionProtectionStatus.Failed),
            "the staged protection is terminally failed"
        );
        assertEq(
            clearinghouse.getAccountAssetBuckets(ALICE).freeSettlementAssets - aliceFreeBefore,
            2_000e6 + protectionBounty,
            "parent margin and the protection bounty are refunded to the trader"
        );
    }

    function _refreshMark(
        uint256 price
    ) internal {
        if (block.timestamp <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, block.timestamp, previous));
        }
        bytes[] memory updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
        router.updateMarkPrice(updateData);
    }

    function _boundedOpenRequest(
        CfdTypes.Side side,
        uint256 sizeDelta,
        uint256 marginDelta
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = bytes32(uint256(router.nextCommitId()));
        request.side = side;
        request.sizeDelta = sizeDelta;
        request.marginDelta = marginDelta;
        request.targetPrice = side == CfdTypes.Side.LONG ? 1 : engine.CAP_PRICE();
        request.isClose = false;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(block.timestamp + router.maxExecutionWindowSeconds()),
            executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
            allowedExecutionModes: 1 | 2 | 4,
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
        });
    }

    function _commitOpen(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin
    ) internal returns (uint64 orderId) {
        orderId = router.nextCommitId();
        vm.prank(account);
        router.commitOrder(side, size, margin, MARK_PRICE, false);
    }

    function _commitClose(
        address account,
        CfdTypes.Side side,
        uint256 size
    ) internal returns (uint64 orderId) {
        orderId = router.nextCommitId();
        vm.prank(account);
        router.commitOrder(side, size, 0, 0, true);
    }

}
```

The helpers `_fundTrader`, `_open`, `_basePythFeedIds` and `_executionBountyReserve` come from the pinned `BasePerpTest` deployment helper in the same release.

### Exact commands

From the working directory `packages/perps`:

Primary command, as executed with the pinned `via_ir = true` profile and a warm build cache:

```bash
forge test --match-path 'test/reviewer-04/TreasuryConfigTransition.t.sol' -vv
```

Observed output:

```text
No files changed, compilation skipped

Ran 4 tests for test/reviewer-04/TreasuryConfigTransition.t.sol:TreasuryConfigTransitionTest
[PASS] test_ScheduledTreasuryActivation_InvalidatesOrdersWithoutAnyTransaction() (gas: 2171437)
[PASS] test_TreasuryRotation_ChangesConfigHash_AndInvalidatesPendingOpen() (gas: 2214692)
[PASS] test_TreasuryRotation_FailsAttachedPendingOpenProtection() (gas: 3052645)
[PASS] test_TreasuryRotation_InvalidatesPendingRiskReducingClose() (gas: 2612222)
Suite result: ok. 4 passed; 0 failed; 0 skipped; finished in 21.78ms (17.74ms CPU time)

Ran 1 test suite in 28.19ms (21.78ms CPU time): 4 tests passed, 0 failed, 0 skipped (4 total tests)
```

Fallback command for a cold or loaded host where via-IR compilation is slow. This rebuilds the project with solc 0.8.35, optimizer 200 runs, and via-IR disabled, then runs the same file. It was executed on the same host:

```bash
FOUNDRY_VIA_IR=false forge test --match-path 'test/reviewer-04/TreasuryConfigTransition.t.sol' -vv
```

Observed output of the fallback run (compiler warnings omitted):

```text
Ran 4 tests for test/reviewer-04/TreasuryConfigTransition.t.sol:TreasuryConfigTransitionTest
[PASS] test_ScheduledTreasuryActivation_InvalidatesOrdersWithoutAnyTransaction() (gas: 2100570)
[PASS] test_TreasuryRotation_ChangesConfigHash_AndInvalidatesPendingOpen() (gas: 2151502)
[PASS] test_TreasuryRotation_FailsAttachedPendingOpenProtection() (gas: 2978251)
[PASS] test_TreasuryRotation_InvalidatesPendingRiskReducingClose() (gas: 2539126)
Suite result: ok. 4 passed; 0 failed; 0 skipped; finished in 16.44ms (9.16ms CPU time)

Ran 1 test suite in 17.48ms (16.44ms CPU time): 4 tests passed, 0 failed, 0 skipped (4 total tests)
```

### Positive control

Positive control: the release suite test `test_SuccessAndExpiryBothCreditInternalBounties` in `test/perps/spec/keeper/OrderBountyLifecycle.t.sol` runs in the same deployment, under the same unchanged execution-config digest, and shows an order reaching the successful path. It commits an open order, executes it, asserts `LifecycleStatus.Executed`, asserts the position size becomes `50_000e18`, and asserts the keeper is credited the reserved bounty. It then shows that an unrelated terminal failure is `Expired`, not `ConfigMismatch`.

Command from `packages/perps`:

```bash
forge test --match-path 'test/perps/spec/keeper/OrderBountyLifecycle.t.sol' --match-test 'test_SuccessAndExpiryBothCreditInternalBounties' -vv
```

Observed output (compiler warnings omitted):

```text
Ran 1 test for test/perps/spec/keeper/OrderBountyLifecycle.t.sol:InternalBountyCustodyTest
[PASS] test_SuccessAndExpiryBothCreditInternalBounties() (gas: 5870344)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 17.21ms (8.05ms CPU time)

Ran 1 test suite in 24.69ms (17.21ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

Together, the control and the exploit show that the harness opens positions normally when the digest matches, and that the treasury rotation alone is what produces `ConfigMismatch`.

### Reproduction steps

1. Trader commits a config-pinned open order through the production `OrderRouter`, supplying the current nonzero `expectedConfigHash` and a margin. The Router reserves the execution bounty from the trader's free settlement balance and stores the digest in `bounds.expectedConfigHash`.
2. Engine owner performs a documented treasury rotation. The owner either calls `setProtocolTreasury(newAccount)` while the treasury lock is unlocked, or has already called `proposeProtocolTreasury(newAccount, nextLockDuration)` during an active lock.
3. Treasury successor becomes effective at the lock expiry by timestamp with no transaction in the scheduled case. The Engine's `protocolTreasury()` returns the successor, and `currentExecutionConfigHash()` changes because the treasury address is part of the hashed `engineConfig`.
4. Keeper observes the rotated treasury or the public activation time and calls `executeOrder(orderId, [])` or `clearInvalidOrders([orderId])` for the pending order.
5. `OrderRouterExecutionSidecar` compares the observed digest with the pinned `expectedConfigHash`, finds a mismatch, and terminalizes the order as `Failed` with `TerminalReason.ConfigMismatch`.
6. `OrderRouter._settleTerminalBounty` releases the reservation. The trader's committed margin returns to the trader's free settlement balance. Because the caller is the keeper and not the order account, the reserved execution bounty is credited to the keeper's clearinghouse balance.
7. Trader's close order leaves the position open because no close executes. For an attached protection parent, the staged protection is terminally failed and its trigger bounty is refunded to the trader.

### Expected and observed results

Expected result: the order is terminal `Failed` with `ConfigMismatch`, no position opens, the margin is refunded, and the reserved bounty is paid to the cleaner rather than used for execution. For a close, the position remains open. For an attached protection, the protection fails and its bounty is refunded.

Observed result, covered by the four passing tests:

- `test_TreasuryRotation_ChangesConfigHash_AndInvalidatesPendingOpen`: the digest differs after `engine.setProtocolTreasury`; the result is `Failed` with `ConfigMismatch`; the position size is unchanged; the trader's free settlement assets increase by exactly `2_000e6`; the keeper's clearinghouse balance increases by exactly the reserved bounty; the treasury clearinghouse balance is unchanged.
- `test_ScheduledTreasuryActivation_InvalidatesOrdersWithoutAnyTransaction`: after `vm.warp` to `protocolTreasuryLockedUntil()`, `engine.protocolTreasury()` returns the proposed successor and the digest changes without any transaction; the pending order is terminalized with `ConfigMismatch`.
- `test_TreasuryRotation_InvalidatesPendingRiskReducingClose`: a pending close is terminalized with `ConfigMismatch`, the position remains open, and the close order is permanently `Failed`.
- `test_TreasuryRotation_FailsAttachedPendingOpenProtection`: the attached parent open fails with `ConfigMismatch`, the protection status becomes `Failed`, and the trader is refunded the `2_000e6` parent margin plus the protection trigger bounty.

The exact pass output is recorded in the command section above.

### Cleanup

From `packages/perps`:

```bash
rm -f test/reviewer-04/TreasuryConfigTransition.t.sol
rmdir test/reviewer-04
forge clean
```

Delete the disposable clone afterwards. If the checkout is reused, confirm that `test/reviewer-04` no longer exists and that no production source was modified.

### Scope and material limits

Scope. The defect is in the in-scope `packages/perps/src` contracts: `OrderLifecycleBook`, `CfdTreasury`, `OrderRouter` and `OrderRouterExecutionSidecar`. The test file lives under `packages/perps/test`, which is out of program scope, and exists only to exercise the in-scope production code.

Material limits, stated as positive conditions:

- The trigger is a legitimate documented owner action. The Engine owner either rotates the treasury immediately while the lock is unlocked or schedules a successor during an active lock. No malicious owner behavior and no out-of-procedure action is required.
- The owner can perform the rotation at any time the documented lock rules allow. A scheduled successor makes the activation timestamp public through `pendingProtocolTreasury()` and `protocolTreasuryLockedUntil()`.
- Only orders committed with a nonzero `expectedConfigHash` are affected. Liquidation closes and protection-trigger closes use the zero unpinned marker and are unaffected.
- The trader's committed margin is refunded on cleanup. No principal, LP backing, or unclaimed yield is lost.
- The trader can avoid the bounty redirect by performing the terminal cleanup themselves. In that case `bountyRecipient == order.account`, and the Router releases the reservation back to the trader's account instead of crediting a third party.
- The affected work is bounded by the pending queue at the moment the digest changes. `clearInvalidOrders` processes up to `min(64, maxPruneOrdersPerCall)` orders per call.
- The digest composition, including the treasury, is described in `SECURITY.md`. The demonstrated defect is the resulting terminal delivery failure and bounty redirect after a documented rotation, not hidden or undocumented hashing behavior.

