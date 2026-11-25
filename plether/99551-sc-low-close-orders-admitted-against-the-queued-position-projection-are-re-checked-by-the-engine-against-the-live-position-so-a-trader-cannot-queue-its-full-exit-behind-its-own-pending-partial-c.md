# \#99551 [SC-Low] Close orders admitted against the queued-position projection are re-checked by the Engine against the live position, so a trader cannot queue its full exit behind its own pending partial close

**Submitted on Nov 14th 2026 at 06:41:14 UTC by @Valisthea for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99551
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Summary

A close order is admitted in two steps inside the same `commitOrder` call, and the two steps judge it against different positions:

1. The Router checks side and size against the account's **queued-position projection** (the live position adjusted by the account's earlier queued orders).
2. The Engine's close-commitment hook then re-checks the same request against the **live** position, including the partial-close carry and health rules.

A close that is a *full* close of the queued projection is therefore treated as a *partial* close of the live position. When the live position is under maintenance (exactly when a trader wants out), the documented full exit cannot be queued behind the trader's own earlier queued orders. The Router's own preview (`previewCloseAdmission`) only runs step 1, so it accepts requests that `commitOrder` then rejects.

## Documented behaviour

- `packages/perps/WORKING_WITH_AI_AGENTS.md:81-82`: for closes, the request must have "the same side as the account's queued-position projection, and a size no larger than that projection."
- `packages/perps/ACCOUNTING_SPEC.md:1016-1017`: "'Full' is evaluated against the position at each check: queued admission and live execution can differ if an earlier order fails."
- `packages/perps/ACCOUNTING_SPEC.md:1096-1098`: a queued full close can project the account flat and permit an opposite-side open, and a queued close must match the live side *at execution*.
- `packages/perps/src/router/OrderLiquidationBatchLogic.sol:138`: `previewCloseAdmission` "Applies the same prospective admission checks as fresh public close commitment."

## Root cause

- Router admission projects the position through the account's queued orders and validates the close against that projection (`packages/perps/src/router/OrderLiquidationBatchLogic.sol:344-402`, `_validateCloseAdmission` / `_queuedPosition`).
- The same commit then calls `OrderRouterBase._reserveCloseExecutionBounty` -> `CfdEngine.reserveCloseOrderExecutionBounty` (`packages/perps/src/CfdEngine.sol:598`) -> `CfdEngineSettlementSidecar.reserveCloseOrderExecutionBounty` (`packages/perps/src/CfdEngineSettlementSidecar.sol:335-352`) -> `CloseCommitmentLib.project` (`packages/perps/src/libraries/CloseCommitmentLib.sol:17-51`), which validates against the **live** Engine position: a size different from the live size is planned as a partial close and must pass the partial-close carry and health checks.
- `previewCloseAdmission` (`OrderLiquidationBatchLogic.sol:141-154`) runs only the Router step, so the preview and the commit disagree.

## Impact

Two concrete, documented flows fail (both reproduced in the PoC at release parameters):

- **Remainder exit behind a queued partial close.** A trader with LONG 100 queues a healthy partial close of 40. The mark then moves and the position is under maintenance. The full exit of the queued remainder (60) reverts with `CfdEngine__PartialCloseUnhealthy`, because the Engine judges it as a partial close of the live 100; a close of 100 is rejected by the Router with `OrderRouter__SizeExceedsQueued`. No full exit can be queued until the earlier partial resolves, at the moment the trader most needs to exit. Without the earlier queued partial, the same full exit at the same mark is admitted.
- **Close of a queued opposite-side open is judged by the old position's health.** After queuing a full close of an unhealthy LONG 100 and a SHORT 50 open (the documented flip), the close of the queued SHORT 50 reverts with `CfdEngine__PartialCloseUnhealthy` because of the unrelated live LONG's maintenance state. The identical flow with a SHORT 100 open is admitted only because its size coincides with the live LONG.

No value is lost (the trader can retry after the earlier orders execute or expire), so this is filed as Low: the contract fails to deliver the documented close admission and preview behaviour.

## Recommendation

Evaluate the Engine's close-commitment checks against the same queued projection the Router uses (pass the projected size/side into `CloseCommitmentLib.project`), or skip the partial-close health check at commitment when the request is a full close of the queued projection and leave live checks to execution, as `ACCOUNTING_SPEC.md:1016-1017` describes. Make `previewCloseAdmission` run the same Engine-side check so the preview matches `commitOrder`.

## Proof of Concept

## Proof of concept

Self-contained Foundry test at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. It inherits only the repository's `BasePerpTest`, uses the release risk parameters, and mocks only out-of-scope components (Pyth price updates and USDG).

### Steps

`test_A5` (remainder exit):
1. ALICE opens LONG 100 with 300 USDG margin; it fills at 1.00.
2. ALICE queues a healthy partial close of 40 (admitted).
3. The mark moves to 1.03; the position is under maintenance (`engineLens.isLiquidatableAt` is true).
4. ALICE commits a close of 60, the full remainder of the queued projection: reverts `CfdEngine__PartialCloseUnhealthy`.
5. ALICE commits a close of 100: reverts `OrderRouter__SizeExceedsQueued`.
6. Control: from the state before step 2, at the same 1.03 mark, a full close of 100 is admitted.

`test_A3b` (flip):
1. ALICE opens LONG 100 with a thin 100 USDG margin; the mark moves to 1.01 (under maintenance).
2. ALICE queues the full LONG close (admitted) and a SHORT 50 open (admitted).
3. ALICE commits the close of the queued SHORT 50: reverts `CfdEngine__PartialCloseUnhealthy`.
4. Control: the same flow with a SHORT 100 open lets the SHORT 100 close commit.

Save as `packages/perps/test/perps/accounting/close/QueuedCloseAdmissionPoC.t.sol` and run:

```bash
forge test --root packages/perps --match-path 'test/perps/accounting/close/QueuedCloseAdmissionPoC.t.sol' -vv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";

contract QueuedCloseAdmissionPoC is BasePerpTest {

    uint256 internal constant LOT_ = 1e20;
    address internal constant KEEPER0 = address(0xCAFE);
    address internal constant ALICE = address(0xA11CE);
    address internal constant BOB = address(0xB0B);
    bytes4 internal constant OK_ = bytes4(0);

    // Release risk parameters (packages/perps/DEPLOYMENT.md).
    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory params) {
        params = super._riskParams();
        params.vpiFactor = 0.01e18;
        params.maxSkewRatio = 0.4e18;
        params.maintMarginBps = 10;
        params.initMarginBps = 20;
        params.baseCarryBps = 500;
        params.minBountyAssets = 1e6;
        params.bountyBps = 10;
    }

    function setUp() public override {
        super.setUp();
        vm.deal(KEEPER0, 100 ether);
        _fundTrader(ALICE, 50_000e6);
        _fundTrader(BOB, 300_000e6);
        vm.warp(vm.getBlockTimestamp() + 1 days + 3 hours);
        _refresh(1e8);
        require(!engine.isFadWindow() && !engine.isOracleFrozen(), "setup must start in a live session");
    }

    function _pythAt(
        uint256 price,
        uint256 publishTime
    ) internal returns (bytes[] memory updateData) {
        bytes32[] memory feedIds = new bytes32[](2);
        feedIds[0] = bytes32(uint256(1));
        feedIds[1] = bytes32(uint256(2));
        baseMockPyth.setAllUniquePrices(feedIds, int64(int256(price)), 0, int32(-8), publishTime, publishTime - 1);
        updateData = new bytes[](1);
        updateData[0] = abi.encode(price);
    }

    function _refresh(
        uint256 price
    ) internal {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        bytes[] memory data = _pythAt(price, vm.getBlockTimestamp());
        vm.prank(KEEPER0);
        router.updateMarkPrice(data);
    }

    function _commit(
        address account,
        CfdTypes.Side side,
        uint256 lots,
        uint256 margin,
        bool isClose
    ) internal returns (uint64 orderId) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        orderId = router.nextCommitId();
        vm.prank(account);
        router.commitOrder(side, lots * LOT_, margin, 0, isClose);
    }

    function _fillHead(
        uint256 price
    ) internal returns (uint64 head) {
        head = router.nextExecuteId();
        require(head != 0, "no head");
        (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(head);
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 2);
        bytes[] memory data = _pythAt(price, uint256(pending.commitTime) + 1);
        vm.prank(KEEPER0);
        router.executeOrder(head, data);
    }

    function _size(
        address account
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    function _side(
        address account
    ) internal view returns (CfdTypes.Side side) {
        (,,,, side,,) = engine.positions(account);
    }

    /// @dev Same commit path as `_commit`, but returns the revert selector (zero on success) instead of bubbling.
    function _commitSelector(
        address account,
        CfdTypes.Side side,
        uint256 lots,
        uint256 margin,
        bool isClose
    ) internal returns (bytes4 selector) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        vm.prank(account);
        try router.commitOrder(side, lots * LOT_, margin, 0, isClose) returns (uint64) {
            selector = OK_;
        } catch (bytes memory err) {
            selector = err.length >= 4 ? bytes4(err) : bytes4(0xffffffff);
        }
    }

    // A5: queued partial close, then the position goes under maintenance at the cached mark. The close of the queued
    //     remainder is "full" at queued admission (ACCOUNTING_SPEC.md:1016-1017) but the Engine applies partial-close
    //     health to the live size; a close of the live size is rejected by the Router projection. No full exit can be
    //     queued until the earlier partial close resolves.
    function test_A5_RemainderCloseBehindQueuedPartialRejectedWhenUnderMaintenance() public {
        _commit(ALICE, CfdTypes.Side.LONG, 100, 300e6, false);
        _fillHead(1e8);
        uint256 beforePartial = vm.snapshotState();

        _commit(ALICE, CfdTypes.Side.LONG, 40, 0, true); // healthy partial close (admitted)
        _refresh(1.03e8); // LONG loses ~300 USDG at the cached mark: under maintenance
        assertTrue(engineLens.isLiquidatableAt(ALICE, 1.03e8, pool.totalAssets()), "under maintenance");

        assertEq(
            _commitSelector(ALICE, CfdTypes.Side.LONG, 60, 0, true),
            ICfdEngineTypes.CfdEngine__PartialCloseUnhealthy.selector,
            "remainder close (full per queued projection) is judged as a partial close of the live 100"
        );
        assertEq(
            _commitSelector(ALICE, CfdTypes.Side.LONG, 100, 0, true),
            IOrderRouterErrors.OrderRouter__SizeExceedsQueued.selector,
            "live-size close is rejected by the queued projection"
        );

        // Control: without the earlier queued partial, the full exit at the same unhealthy mark is admitted.
        vm.revertToState(beforePartial);
        _refresh(1.03e8);
        assertEq(_commitSelector(ALICE, CfdTypes.Side.LONG, 100, 0, true), OK_, "full exit admitted when unqueued");
    }

    // A3b: documented flip. The queued SHORT close is rejected by the HEALTH of the unrelated live LONG. Two branches
    //      with the same economic intent (close the whole queued SHORT) differ only in whether the SHORT size happens
    //      to equal the live LONG size.
    function test_A3b_QueuedShortCloseRejectedByUnrelatedLongHealth() public {
        _commit(ALICE, CfdTypes.Side.LONG, 100, 100e6, false); // thin pledge (~85 USDG after fee, VPI, reserve)
        _fillHead(1e8);
        assertEq(_size(ALICE), 100 * LOT_, "live LONG 100");
        _refresh(1.01e8); // LONG loses ~100 USDG at the cached mark: under maintenance
        assertTrue(engineLens.isLiquidatableAt(ALICE, 1.01e8, pool.totalAssets()), "LONG is under maintenance");

        // A full close of the unhealthy LONG is admitted (full closes carry no health check).
        _commit(ALICE, CfdTypes.Side.LONG, 100, 0, true);
        uint256 branch = vm.snapshotState();

        // Branch 1: queue SHORT 50, then the full close of that queued SHORT -> rejected as a PARTIAL close of the LONG.
        _commit(ALICE, CfdTypes.Side.SHORT, 50, 2000e6, false);
        bytes4 small = _commitSelector(ALICE, CfdTypes.Side.SHORT, 50, 0, true);
        assertEq(
            small,
            ICfdEngineTypes.CfdEngine__PartialCloseUnhealthy.selector,
            "close of the queued SHORT is judged by the live LONG's maintenance health"
        );

        // Branch 2 (control): identical shape with SHORT 100 -> the Engine sees a 'full' LONG close and admits it.
        vm.revertToState(branch);
        _commit(ALICE, CfdTypes.Side.SHORT, 100, 2000e6, false);
        assertEq(
            _commitSelector(ALICE, CfdTypes.Side.SHORT, 100, 0, true),
            OK_,
            "same intent admitted only because the size coincides with the live LONG"
        );
    }

}
```

### Output (run locally at a82d4fef)

```
[PASS] test_A3b_QueuedShortCloseRejectedByUnrelatedLongHealth() (gas: 7104262)
[PASS] test_A5_RemainderCloseBehindQueuedPartialRejectedWhenUnderMaintenance() (gas: 5262927)
Suite result: ok. 2 passed; 0 failed; 0 skipped
```
