# \#99918 [SC-Insight] The MED-14 fix counts an expired queued open as live exposure: a flat trader's opposing open is refused until the dead order is pruned

**Submitted on Nov 15th 2026 at 18:07:12 UTC by @argo_the_hacker for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99918
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

**Severity: Insight.** Submitted under the program's Insight Reporting category, not as one of the listed bug impacts: no value moves, and the affected trader can cure it with one public call. The section "Conditions under which this matters" states when it is valuable.

**Target:** `immunefi-team/audit-comp-plether` @ `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, `packages/perps/src/router/OrderLiquidationBatchLogic.sol`: `_queuedPosition` (lines 367-402, its filter at line 383), as read by the new side check in `_validatedOpenBounty` (lines 424-428).

**Relation to prior findings:** a new bug introduced by the fix for Cecuro **MED-14** (commit `88ef3954`, #126, "fix(perps): validate open side against queued position"; Cecuro's MED-14 comment links plether-core `5029cde9`). The program's known-issues rule makes a report on a fixed root valid when it shows the fix "is incomplete or introduces a new bug". This report shows a new bug. Cecuro **MED-4** (`clearInvalidOrders`) is the cure, not the root. A separate report covers a different root in the same function (the price-dependent preview reads the live position). Fixing either one leaves the other in place.

## Summary

To fix MED-14, `_validatedOpenBounty` now projects the trader's live position through their earlier queued orders (`_queuedPosition`, `OrderLiquidationBatchLogic.sol:367-402`). It refuses an open on the other side of that projection with `PredictableOpenInvalid(MUST_CLOSE_OPPOSING)` (`OrderLiquidationBatchLogic.sol:424-428`). The projection skips orders cut off by the risk-off cutoff (`OrderLiquidationBatchLogic.sol:383`), because they will never execute. It does **not** skip orders that are past their `executionDeadline`, which will never execute either. The router itself treats such an order as dead: `clearInvalidOrders` and `expireOrder` remove it without oracle data, and `ACCOUNTING_SPEC.md` (Order State Model) says "expiry resolves through `Failed`".

So an expired, unpruned queued open still counts as live exposure. In the PoC a flat trader, DAVE, has one queued order: a LONG 200k open that expired unexecuted. With a fresh mark DAVE commits a SHORT 200k. The router's own commit lens reads that SHORT as `OK` for DAVE, and the identical SHORT from another flat account is admitted. DAVE's is refused `PredictableOpenInvalid(1)` (`MUST_CLOSE_OPPOSING`). The refusal does not depend on the mark and does not end with time: it is still in place one day later, with a stale mark and again with a fresh one. It ends only when someone prunes the dead order (`clearInvalidOrders`, `expireOrder`, or a keeper's FIFO pass reaching it). After that the identical SHORT is admitted and fills.

Before `88ef3954` the open path did not read the projection (only close admission did), so this SHORT was admitted. The refusal is new behaviour from the MED-14 fix.

## Relation to prior findings

**Cecuro MED-14** (Acknowledged, "Addressed"), "Trader can be blocked from queueing an open behind their closing order". Its impact: "Routine multi-step trading sequences become unavailable during normal fresh-mark operation." The maintainers' fix comment: "Open admission projects the live position through earlier queued orders for the same account before checking the requested side. … This is an admission projection, not a guarantee that the preceding close will execute: actual position, margin and risk checks run again at execution, so a failed earlier close can still make the later open fail." That caveat accepts **admitting** an open that may later fail because an earlier order fails at execution. This report is the opposite direction: a valid open is **refused** because of an earlier order that the router already knows, at commit time, can never execute (`block.timestamp > executionDeadline` is the exact test `clearInvalidOrders` applies). The projection loop already makes this distinction for risk-off-cut orders at line 383. It does not make it for expired ones.

**Cecuro MED-4** (Acknowledged, "Addressed with bounded permissionless cleanup"). Its comment: "`clearInvalidOrders(uint64[])` can remove expired or configuration-invalid orders anywhere in the global queue, including behind a live head, without oracle updates." That is the cure used in the PoC, and it is the router's own definition of a dead order. MED-4 is about keepers having to prune stale heads before reaching live orders. It does not cover admission counting dead orders as exposure.

Other nearby text, and why it is different:
- `SECURITY.md` Router and keeper limitations: "users cannot manually cancel queued intents" and "bounded cleanup means heavily expired queues may take multiple keeper calls to clear". These accept that dead orders can sit in the queue for a while. They do not say that a dead order keeps blocking the owner's new orders, and the projection rule in `ACCOUNTING_SPEC.md:1096` ("An open must match the position projected through earlier queued orders") is about orders that will be applied before the new one, which an expired order will not be.
- Pre-audit SC-I2 ("Missing Order Cancellation Mechanism") is about funds held in a pending order with no user cancel. Different root: here no funds are held up. A valid new order is refused.
- The Cantina and collaborative audit reports and the AI scan contain nothing on this path.

## Vulnerability details

`_queuedPosition` (`OrderLiquidationBatchLogic.sol:367-402`):

```solidity
        uint64 riskOffCutoff = IOrderRouterEmergencyAdmin(host.admin()).riskOffOrderCutoff();
        for (uint64 orderId = host.accountHeadOrderId(account); orderId != 0;) {
            (IOrderRouterAccounting.PendingOrderView memory pending, uint64 nextOrderId) =
                host.getPendingOrderView(orderId);
            if (riskOffCutoff == 0 || orderId > riskOffCutoff || pending.isClose) {
                if (pending.isClose) {
                    ...
                } else if (!queuedPosition.exists) {
                    queuedPosition.exists = true;
                    queuedPosition.side = pending.side;
                    queuedPosition.size = pending.sizeDelta;
                } else if (pending.side == queuedPosition.side) {
                    queuedPosition.size += pending.sizeDelta;
                }
            }
            orderId = nextOrderId;
        }
```

Every order still linked in the account list is applied unless the risk-off cutoff removed it. Expiry is never checked. `_validatedOpenBounty` (`OrderLiquidationBatchLogic.sol:423-428`) then refuses on the projected side, before any price-dependent check and whatever the mark's freshness:

```solidity
        // Project the live position through earlier account orders before checking the requested side.
        QueuedPositionView memory queuedPosition = _queuedPosition(host, engine, account);
        uint8 opposingCode = uint8(CfdEnginePlanTypes.OpenRevertCode.MUST_CLOSE_OPPOSING);
        if (queuedPosition.exists && queuedPosition.side != side) {
            revert OrderRouter__PredictableOpenInvalid(opposingCode);
        }
```

An account that is flat and whose only queued order is an expired LONG open is projected as LONG, so any SHORT open is refused `MUST_CLOSE_OPPOSING`. The planner, which plans on the live position, would have answered `OK` (the PoC reads the router's own commit lens: code 0).

Before `88ef3954`, `_queuedPosition` was called only from `_validateCloseAdmission` (pre-fix line 346), and `_validatedOpenBounty` classified opens through the live-position preview only. The fix added the call at line 424 and the unconditional refusal at lines 426-428, so the refusal of a valid open behind a dead order is new code.

The same loop also counts configuration-invalid orders, the other class `clearInvalidOrders` removes. This PoC exercises expiry only. Close admission reads the same projection (line 351), so an expired queued close likewise affects the owner's next close. That close-side behaviour predates `88ef3954` and is not claimed as new here. The fix below covers both.

## Impact

**Who:** any trader whose own queued open expired without being executed, and who then wants to open the other side. Orders bind to `msg.sender`: another account's expired order does not enter this projection, so no third party can cause it.

**What the contract fails to deliver:** the MED-14 remediation, whose aim is that open admission follow the account's queued exposure (the finding was that the precheck "makes admission depend on mark freshness rather than queued exposure"). Exposure that can no longer occur now blocks admission. A flat account is refused a valid open until it prunes its own dead order. The refusal does not lapse with time or mark refreshes (the PoC shows it 86,401 s after the deadline, with a stale and with a fresh mark).

**Conditions under which this matters (the program's Insight rule):**
- After a keeper or oracle outage. `SECURITY.md`'s own runbook ("Keeper outage: 1. Orders accumulate in the queue. 2. Users cannot manually cancel them. … 4. Expect bounded cleanup rather than instant queue drain.") describes exactly the state in which many accounts hold expired, unpruned opens. Each of those accounts is refused any opposite-side open until its dead order is pruned, while the keeper backlog is drained.
- In front-ends and integrations. The error is `PredictableOpenInvalid(MUST_CLOSE_OPPOSING)` on an account with no position. Nothing in the error points to an expired order, and users "cannot manually cancel queued intents". A client must know to call `clearInvalidOrders` or `expireOrder` on the right id first.
- The projection also counts configuration-invalid orders. A configuration change that invalidates pinned orders (for example a scheduled treasury rotation, which changes `currentExecutionConfigHash`) puts every account with a pinned order queued at that moment in the same state at once, until a prune. (This PoC exercises expiry only.)

**Limits, stated plainly:**
- No value moves and no funds are frozen. The expired order itself is settled by the ordinary prune (MED-4: "Each item uses the existing reservation, refund, bounty, protection and receipt rules").
- Only the owner of the expired order is affected, only for an open on the other side of it.
- The owner can cure it alone, in one transaction, and re-commit in the same block (`clearInvalidOrders([id])`, shown in the PoC). Any EOA can do the same with `expireOrder(id)`, and a keeper's FIFO pass terminalizes the order when it reaches the head.
- The order must first have expired unexecuted, which needs keepers not to execute it inside its window.


## Recommendation

Skip orders that can no longer execute in `_queuedPosition`, with the same predicate `clearInvalidOrders` already uses (`OrderRecoverySidecar.sol:91-93`), next to the existing risk-off skip. As a sketch:

```solidity
        IOrderLifecycleBook book = host.lifecycleBook();
        bytes32 configHash = book.currentExecutionConfigHash();
        for (uint64 orderId = host.accountHeadOrderId(account); orderId != 0;) {
            (IOrderRouterAccounting.PendingOrderView memory pending, uint64 nextOrderId) =
                host.getPendingOrderView(orderId);
            OrderV3Types.PendingIntent memory intent = book.pendingIntent(orderId);
            bool dead = block.timestamp > intent.timing.executionDeadline
                || (intent.bounds.expectedConfigHash != bytes32(0) && intent.bounds.expectedConfigHash != configHash);
            if (!dead && (riskOffCutoff == 0 || orderId > riskOffCutoff || pending.isClose)) {
                // ... unchanged projection ...
            }
            orderId = nextOrderId;
        }
```

The walk stays bounded by the per-account pending-order cap. Execution already treats these orders as dead, so admission and execution then agree. The same change also stops an expired queued close from affecting the owner's next close (close admission reads the same projection). An alternative is to prune the account's own dead orders at the start of `commitOrder`, but that moves cleanup work and its accounting into every commit. Skipping them in the read-only projection is the smaller change.

A regression test like `test_Property_ExpiredQueuedOpenDoesNotBlockOpposingOpen` above would cover it. The current suite checks the projection only with live orders.


## Link to Proof of Concept

https://gist.github.com/Richter2552/2fb427809e9bdfe7a374b0b0c1fe35f5

## Proof of Concept

## Proof of Concept

The program requires PoCs to run in the repository's Foundry test suite at the commit in scope without modifying the contracts in scope. The test file below extends the maintainers' `PositionProtectionTest` fixture **unchanged** (no hook is overridden; its `setUp` deploys and seeds the protocol with the maintainers' own helpers). After `setUp` it uses only public entry points from accounts with no role: production `OrderRouter.commitOrder(OrderRequest)` from the trader's own account, the permissionless `router.updateMarkPrice`, `router.executeOrder`, `router.expireOrder` and `router.clearInvalidOrders`. It also reads `previewOpenRevertCode` on the router's commit lens. Pyth is the maintainers' `MockPyth` (Pyth is outside the scope). The file pranks no protocol contract and writes no storage. The fixture's own tests are inherited, so run only the tests named below.

The **property** test states the behaviour the code should have and **fails** on the pinned code. Each **characterize/control** test passes and pins the exact behaviour.

Run instructions (from a checkout of `immunefi-team/audit-comp-plether` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8` with submodules; solc 0.8.35 via the repository's `packages/perps/foundry.toml`):

```bash
cp Card8785S3ExpiredQueuedOpen.t.sol packages/perps/test/perps/spec/trader/
cd packages/perps

forge test --match-contract '^Card8785S3ExpiredQueuedOpenTest$' --match-test '^test_Property_' -vv

forge test --match-contract '^Card8785S3ExpiredQueuedOpenTest$' --match-test '^test_(Control|Characterize)_' -vv
```

### PoC output (pinned code, unmodified contracts)

Run against a clean tree of the pinned commit with the file installed next to `PositionProtection.t.sol` (forge 1.8.1, solc 0.8.35, the repository's `packages/perps/foundry.toml`, which compiled 240 files). Result: the 4 characterize/control tests pass, and the property test fails. Excerpt of the output of the two commands above, compiler warnings and timing lines omitted (forge `-vv`; codes: 0 = `OK`, 1 = `MUST_CLOSE_OPPOSING`; sizes in 1e18 units):

```console
Ran 1 test for test/perps/spec/trader/Card8785S3ExpiredQueuedOpen.t.sol:Card8785S3ExpiredQueuedOpenTest
[FAIL: OrderRouter__PredictableOpenInvalid(1)] test_Property_ExpiredQueuedOpenDoesNotBlockOpposingOpen() (gas: 2702483)
Logs:
  expired LONG id           1
  its executionDeadline     1709535662
  now                       1709535663

Suite result: FAILED. 0 passed; 1 failed; 0 skipped

Ran 4 tests for test/perps/spec/trader/Card8785S3ExpiredQueuedOpen.t.sol:Card8785S3ExpiredQueuedOpenTest
[PASS] test_Characterize_ExpiredQueuedOpenBlocksOpposingOpen() (gas: 6703932)
Logs:
  expired LONG id           1
  its executionDeadline     1709535662
  now                       1709535663
  lens: DAVE SHORT 200k code (live position, flat) 0
  DAVE SHORT size after cleanup + execute 200000000000000000000000

[PASS] test_Characterize_RefusalPersistsUntilExpiredOrderIsPruned() (gas: 5094081)
Logs:
  expired LONG id           1
  its executionDeadline     1709535662
  now                       1709535663
  mark age at +1 day (s)   86400
  seconds past the deadline, still refused 86401

[PASS] test_Control_FlatAccountOpposingOpenAdmitted() (gas: 4109344)
Logs:
  expired LONG id           1
  its executionDeadline     1709535662
  now                       1709535663

[PASS] test_Control_LiveQueuedOpenBlocksOpposingOpen() (gas: 2212323)
Suite result: ok. 4 passed; 0 failed; 0 skipped
```

Expected output on a fixed contract: the property test passes. The two `test_Characterize_` tests then fail at their first `expectRevert`, because the refusal they pin no longer happens. Both controls still pass: a live (unexpired) opposing open is still refused.

### What the tests show

| step | test | result on the pinned code |
|---|---|---|
| DAVE (flat) commits LONG 200k with 20k margin; no keeper executes it; time passes its `executionDeadline`; any EOA refreshes the mark. DAVE is flat, the LONG is still in his queue, and the mark is fresh | setup of every test except the live control | — |
| control: in that state flat EVE, with no queued orders, is admitted the identical SHORT 200k; the commit lens reads `OK` for her | `test_Control_FlatAccountOpposingOpenAdmitted` | passes |
| control (what the MED-14 rule should do): while DAVE's LONG is still live, his SHORT is refused `MUST_CLOSE_OPPOSING` | `test_Control_LiveQueuedOpenBlocksOpposingOpen` | passes |
| property: DAVE's only queued order is expired, so his SHORT should be admitted | `test_Property_ExpiredQueuedOpenDoesNotBlockOpposingOpen` | **FAILS**: `OrderRouter__PredictableOpenInvalid(1)` (`MUST_CLOSE_OPPOSING`) |
| the commit lens reads `OK` for DAVE's SHORT; the commit is refused code 1; DAVE calls `clearInvalidOrders([id])` (1 cleared, queue empty); the identical SHORT is admitted and a role-less keeper fills it (DAVE SHORT 200k) | `test_Characterize_ExpiredQueuedOpenBlocksOpposingOpen` | passes |
| one day later (mark 86,400 s old) the SHORT is still refused code 1; after a fresh refresh it is still refused; a role-less EOA calls `expireOrder(id)` (status `Failed`); the identical SHORT is then admitted | `test_Characterize_RefusalPersistsUntilExpiredOrderIsPruned` | passes |

### Test file: `packages/perps/test/perps/spec/trader/Card8785S3ExpiredQueuedOpen.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {PositionProtectionTest} from "./PositionProtection.t.sol";
import {CfdEnginePlanTypes} from "@plether/perps/CfdEnginePlanTypes.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineLens} from "@plether/perps/interfaces/ICfdEngineLens.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @title The MED-14 fix counts an EXPIRED queued open as live exposure and refuses the account's opposing open
/// @notice Pinned commit a82d4fefb02768e5b3bddcc97dee371eee24fea8. Install next to PositionProtection.t.sol in
///         packages/perps/test/perps/spec/trader/ and run with the repository's own foundry.toml. The maintainers'
///         PositionProtectionTest fixture is used unchanged (no override); run only the tests named in this file.
/// @dev `_validatedOpenBounty` (router/OrderLiquidationBatchLogic.sol:404-455) now projects the live position through
///      the account's earlier queued orders (`_queuedPosition`, :367-402) and refuses an open on the other side of
///      the projection with `PredictableOpenInvalid(MUST_CLOSE_OPPOSING)` (:424-428). The projection skips orders
///      cut off by the risk-off cutoff (:383), but not orders past their `executionDeadline`, which the router itself
///      treats as dead (`clearInvalidOrders` / `expireOrder` remove them without oracle data; ACCOUNTING_SPEC.md
///      "Order State Model": expiry resolves through `Failed`). So a flat account whose only queued order is an
///      expired LONG open is refused a SHORT open that the planner itself reads as `OK`, until someone prunes the
///      dead order. Before 88ef3954 the open path did not read the projection and this SHORT was admitted.
///      After setUp every step goes through a public entry point from an account with no role: production
///      `OrderRouter.commitOrder`, `router.updateMarkPrice`, `router.executeOrder`, `router.expireOrder`,
///      `router.clearInvalidOrders`. Pyth is the maintainers' MockPyth. No protocol contract is pranked and no
///      storage is written by this file.
contract Card8785S3ExpiredQueuedOpenTest is PositionProtectionTest {

    address internal constant DAVE = address(0xDA7E); // ordinary trader
    address internal constant EVE = address(0xE7E); // ordinary trader, flat, no queued orders
    address internal constant ROLELESS_KEEPER = address(0x5EE5); // no role anywhere
    address internal constant REFRESHER = address(0x4EF4); // any EOA
    uint256 internal constant TRADER_FUNDS = 200_000e6;
    uint256 internal constant OPEN_SIZE = 200_000e18; // $200k notional at MARK_PRICE = 1e8 ($1)
    uint256 internal constant OPEN_MARGIN = 20_000e6;

    uint8 internal constant CODE_OK = uint8(CfdEnginePlanTypes.OpenRevertCode.OK);
    uint8 internal constant CODE_MUST_CLOSE_OPPOSING = uint8(CfdEnginePlanTypes.OpenRevertCode.MUST_CLOSE_OPPOSING);

    uint256 internal _clientNonce;

    // ---------------------------------------------------------------------------------------------------------------
    // Public-path helpers
    // ---------------------------------------------------------------------------------------------------------------

    function _book() internal view returns (IOrderLifecycleBook) {
        return IOrderLifecycleBook(address(router.lifecycleBook()));
    }

    function _tickNow(
        uint256 price
    ) internal view returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        uint256 nowTs = vm.getBlockTimestamp();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, nowTs, previous));
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

    /// Any EOA refreshes the cached mark through the permissionless router entry point.
    function _publicRefresh(
        uint256 price
    ) internal {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory data = _tickNow(price);
        vm.prank(REFRESHER);
        router.updateMarkPrice(data);
    }

    function _request(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin
    ) internal returns (OrderV3Types.OrderRequest memory request) {
        uint256 window = router.maxExecutionWindowSeconds();
        request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode("card-8785-s3", ++_clientNonce)),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: side == CfdTypes.Side.LONG ? 1 : type(uint256).max, // no slippage limit (source harness)
            isClose: false,
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

    function _commitOpen(
        address trader,
        CfdTypes.Side side
    ) internal returns (uint64 orderId) {
        OrderV3Types.OrderRequest memory request = _request(side, OPEN_SIZE, OPEN_MARGIN);
        vm.prank(trader);
        orderId = OrderRouter(address(router)).commitOrder(request);
    }

    function _expectOpenRefused(
        address trader,
        CfdTypes.Side side,
        uint8 code
    ) internal {
        OrderV3Types.OrderRequest memory request = _request(side, OPEN_SIZE, OPEN_MARGIN);
        vm.prank(trader);
        vm.expectRevert(abi.encodeWithSelector(IOrderRouterErrors.OrderRouter__PredictableOpenInvalid.selector, code));
        OrderRouter(address(router)).commitOrder(request);
    }

    /// A role-less keeper executes `orderId` with the first Pyth tick after its commit time.
    function _keeperExecute(
        uint64 orderId
    ) internal {
        (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(orderId);
        uint64 committedAt = pending.commitTime;
        if (vm.getBlockTimestamp() < uint256(committedAt) + 2) {
            vm.warp(uint256(committedAt) + 2);
        }
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory data = _mockUniquePythUpdateDataAt(MARK_PRICE, committedAt + 1, committedAt);
        vm.prank(ROLELESS_KEEPER);
        OrderV3Types.ExecutionResult memory res = router.executeOrder(orderId, data);
        assertEq(uint8(res.status), uint8(OrderV3Types.LifecycleStatus.Executed), "ghost: order executed by keeper");
    }

    /// The commit lens the router itself uses for the price-dependent screen.
    function _preview(
        address account,
        CfdTypes.Side side
    ) internal view returns (uint8) {
        return ICfdEngineLens(OrderRouter(address(router)).engineLensForCommit())
            .previewOpenRevertCode(account, side, OPEN_SIZE, OPEN_MARGIN, engine.lastMarkPrice());
    }

    function _size(
        address account
    ) internal view returns (uint256 size, CfdTypes.Side side) {
        (size,,,, side,,) = engine.positions(account);
    }

    /// DAVE (flat) commits LONG 200k with a fresh mark. No keeper executes it inside its execution window, so it
    /// expires and stays in the account list unpruned. The mark is then refreshed (fresh again) by any EOA.
    function _daveExpiredQueuedLong() internal returns (uint64 staleOpen) {
        _fundTrader(DAVE, TRADER_FUNDS);
        _fundTrader(EVE, TRADER_FUNDS);
        _publicRefresh(MARK_PRICE);
        staleOpen = _commitOpen(DAVE, CfdTypes.Side.LONG);

        uint64 deadline = _book().orderTiming(staleOpen).executionDeadline;
        vm.warp(uint256(deadline) + 1); // strictly after the deadline: the router treats it as expired
        _publicRefresh(MARK_PRICE);

        (uint256 size,) = _size(DAVE);
        assertEq(size, 0, "ghost: DAVE is flat (the LONG never executed)");
        assertGt(vm.getBlockTimestamp(), deadline, "ghost: DAVE's LONG is past its execution deadline");
        assertEq(router.pendingOrderCounts(DAVE), 1, "ghost: the expired LONG is still in DAVE's queue (unpruned)");
        assertEq(engine.lastMarkTime(), vm.getBlockTimestamp(), "ghost: cached mark is fresh (refreshed this block)");
        console2.log("expired LONG id          ", staleOpen);
        console2.log("its executionDeadline    ", deadline);
        console2.log("now                      ", vm.getBlockTimestamp());
    }

    // ---------------------------------------------------------------------------------------------------------------
    // Controls
    // ---------------------------------------------------------------------------------------------------------------

    /// CONTROL: in the same state, a flat account with no queued orders (EVE) is admitted the identical SHORT, and the
    /// commit lens reads OK for it. The SHORT itself is valid.
    function test_Control_FlatAccountOpposingOpenAdmitted() public {
        _daveExpiredQueuedLong();
        assertEq(_preview(EVE, CfdTypes.Side.SHORT), CODE_OK, "lens: EVE's SHORT reads OK");
        uint64 id = _commitOpen(EVE, CfdTypes.Side.SHORT);
        assertGt(id, 0, "control: the SHORT is admitted for a flat account");
    }

    /// CONTROL (the MED-14 rule working as intended): while DAVE's LONG is still LIVE (inside its window), his SHORT
    /// is refused MUST_CLOSE_OPPOSING. The LONG will execute first, so the SHORT would face an opposite position.
    function test_Control_LiveQueuedOpenBlocksOpposingOpen() public {
        _fundTrader(DAVE, TRADER_FUNDS);
        _publicRefresh(MARK_PRICE);
        uint64 liveOpen = _commitOpen(DAVE, CfdTypes.Side.LONG);
        vm.warp(vm.getBlockTimestamp() + 1);
        assertLe(vm.getBlockTimestamp(), _book().orderTiming(liveOpen).executionDeadline, "ghost: LONG still live");
        _expectOpenRefused(DAVE, CfdTypes.Side.SHORT, CODE_MUST_CLOSE_OPPOSING);
    }

    // ---------------------------------------------------------------------------------------------------------------
    // Property: fails on the pinned code
    // ---------------------------------------------------------------------------------------------------------------

    /// PROPERTY: an expired queued order can never execute, so it must not count as exposure. DAVE is flat and his
    /// only queued order is expired; his SHORT (valid, as the control and the lens show) should be admitted.
    /// FAILS on the pinned code: `OrderRouter__PredictableOpenInvalid(1)` (MUST_CLOSE_OPPOSING).
    function test_Property_ExpiredQueuedOpenDoesNotBlockOpposingOpen() public {
        _daveExpiredQueuedLong();
        uint64 id = _commitOpen(DAVE, CfdTypes.Side.SHORT);
        assertGt(id, 0, "the opposing open is admitted once the expired order is ignored");
    }

    // ---------------------------------------------------------------------------------------------------------------
    // Characterization: passes on the pinned code
    // ---------------------------------------------------------------------------------------------------------------

    /// CHARACTERIZE: the exact refusal and its cause. The commit lens (live position: flat) reads OK for DAVE's SHORT;
    /// only the queued-position side check refuses it. One public `clearInvalidOrders` (here by DAVE) removes the
    /// expired LONG, and the identical SHORT is then admitted and fills.
    function test_Characterize_ExpiredQueuedOpenBlocksOpposingOpen() public {
        uint64 staleOpen = _daveExpiredQueuedLong();
        uint8 lensCode = _preview(DAVE, CfdTypes.Side.SHORT);
        console2.log("lens: DAVE SHORT 200k code (live position, flat)", lensCode);
        assertEq(lensCode, CODE_OK, "lens: the planner reads DAVE's SHORT as OK");

        _expectOpenRefused(DAVE, CfdTypes.Side.SHORT, CODE_MUST_CLOSE_OPPOSING);

        uint64[] memory toClear = new uint64[](1);
        toClear[0] = staleOpen;
        vm.prank(DAVE);
        (, uint256 cleared) = OrderRouter(address(router)).clearInvalidOrders(toClear);
        assertEq(cleared, 1, "public cleanup removes the expired LONG");
        assertEq(router.pendingOrderCounts(DAVE), 0, "DAVE's queue is empty");

        uint64 id = _commitOpen(DAVE, CfdTypes.Side.SHORT);
        assertGt(id, 0, "after cleanup the identical SHORT is admitted");
        _keeperExecute(id);
        (uint256 size, CfdTypes.Side side) = _size(DAVE);
        assertEq(size, OPEN_SIZE, "the SHORT fills");
        assertEq(uint8(side), uint8(CfdTypes.Side.SHORT));
        console2.log("DAVE SHORT size after cleanup + execute", size);
    }

    /// CHARACTERIZE: the refusal does not depend on the mark and does not end with time. One day later (stale mark)
    /// and again after a fresh refresh, DAVE's SHORT is still refused MUST_CLOSE_OPPOSING. It ends only when someone
    /// prunes the dead order: here a role-less keeper's `expireOrder` (no oracle data) resolves it `Failed`.
    function test_Characterize_RefusalPersistsUntilExpiredOrderIsPruned() public {
        uint64 staleOpen = _daveExpiredQueuedLong();

        vm.warp(vm.getBlockTimestamp() + 1 days);
        console2.log("mark age at +1 day (s)  ", vm.getBlockTimestamp() - engine.lastMarkTime());
        _expectOpenRefused(DAVE, CfdTypes.Side.SHORT, CODE_MUST_CLOSE_OPPOSING);

        _publicRefresh(MARK_PRICE);
        assertEq(engine.lastMarkTime(), vm.getBlockTimestamp(), "ghost: mark fresh again");
        _expectOpenRefused(DAVE, CfdTypes.Side.SHORT, CODE_MUST_CLOSE_OPPOSING);
        console2.log("seconds past the deadline, still refused",
            vm.getBlockTimestamp() - _book().orderTiming(staleOpen).executionDeadline);

        vm.prank(ROLELESS_KEEPER);
        OrderV3Types.ExecutionResult memory res = router.expireOrder(staleOpen);
        assertEq(uint8(res.status), uint8(OrderV3Types.LifecycleStatus.Failed), "expiry resolves through Failed");
        assertEq(router.pendingOrderCounts(DAVE), 0, "DAVE's queue is empty");

        uint64 id = _commitOpen(DAVE, CfdTypes.Side.SHORT);
        assertGt(id, 0, "after the prune the identical SHORT is admitted");
    }

}
```

