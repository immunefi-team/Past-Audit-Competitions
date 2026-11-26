# \#99924 [SC-High] Permissionless protection retries obstruct the owner's amendment after a failed close

**Submitted on Nov 15th 2026 at 18:26:49 UTC by @legat for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99924
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Asset

https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src

## Tested version and environment

```text
Repository: https://github.com/immunefi-team/audit-comp-plether
Commit:     a82d4fefb02768e5b3bddcc97dee371eee24fea8
Category:   Smart Contract
Execution:  Offline Foundry test on the repository perps fixture
Token:      USDG, 6 decimals
Compiler:   Solidity 0.8.35, optimizer 200, via-IR
```

The PoC uses the scoped Engine, protection book, Router execution and settlement path. The fixture Router is the repository's `LegacyOrderRouterHarness`, a test subclass that adds a scalar `commitOrder` wrapper used only for the ordinary-close lock assertion; that wrapper delegatecalls production V3 commit. The fixture also uses `MockPyth` for signed price updates and `MockSettlementToken` for USDG balances. No in-scope accounting contract is mocked. Initial Junior/Senior assets are 30M/20M USDG, Senior cap 40M and 80%, VPI factor 0.01e18, execution fee 4 bps, frozen-close spread 50 bps. The price updates use zero confidence. The test models a 10,000-lot LONG, a 30-USDG initially signed explicit-fee ceiling and a later 100-USDG owner amendment.

## Impact

Impact: Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Summary

A protected close can fail when Friday's frozen-market spread raises its explicit fees above a ceiling the trader signed while the market was live. The protection then becomes Latched, and the documented remedy is for its owner to amend the retained ceiling before retry. A stranger can invoke the permissionless retry first, causing the protection to become Triggered with one pending child; the owner amendment then reverts. After each child expires, the stranger can expire and retry it atomically. The PoC repeats that sequence 61 times over 3,721 seconds while the owner's 10,000-lot position remains open.

The owner also has a real two-transaction escape: execute or expire the child to relatch, then amend. The PoC proves this escape succeeds if the owner wins the next ordering, and proves an attacker retry inserted between those two transactions blocks the amendment and uses 34,587 in-call gas units in the modeled failed owner call. In a separate conditional correct-price path, the protected close could have completed at 0.90 after an immediate amendment. The PoC then has the attacker maintain the child across the remaining 169,199 seconds of the roughly 47-hour frozen window, with 2,773 renewals and 47 successful insertions between the owner's terminalization and amendment transactions. When the market reopens with an authentic 1.00 mark, the trader's closed balance is 955.398315 USDG lower. The attacker does not control that later market move; the measured loss requires denial through the entire remaining frozen window.

## Finding Description

### Intended behavior

The trader's retained financial ceilings are meant to limit an automatic close. The published MED-9 sponsor comment says, “Tight limits may prevent execution until conditions improve or the owner amends them.” The fixed implementation and `CANONICAL_ENTRYPOINTS.md:142-148` give the owner `amendLatchedPositionProtectionCloseLimits` while protection is Latched and no Router order is pending. `packages/perps/DEPLOYMENT.md:419-421` instructs a worker to investigate persistent failures and not continuously queue the same failing attempt. A failed child should therefore leave a usable owner amendment opportunity before another permissionless retry occupies it.

### Actual behavior

`PositionProtectionBook.sol:315-328` requires Latched status and `pendingOrderCounts(owner) == 0` for amendment. Its public retry path at `:381-406` uses the same state but has no caller restriction, owner-priority interval or cooldown. Retry creates a new close with the retained old limits, changes status to Triggered and increments the pending count (`OrderLiquidationBatchLogic.sol:655-677`). Terminal failure or expiry relatches if the position still matches (`PositionProtectionBook.sol:637-667`; expiry at `OrderRouter.sol:208-232`, execution failure at `:314-329`). An attacker contract can combine public `expireOrder` and `retryPositionProtectionClose` in one transaction, so a separately transacting EOA cannot insert an amendment *inside that attacker transaction*.

The release frozen-close explicit fee combines execution fee and frozen spread (`CfdOrderPolicyEvaluatorBase.sol:416,514-516`). The PoC's 30-USDG cap accepts the ordinary roughly 4-USDG fee but not the frozen roughly 45–50-USDG spread. The first child fails with the typed `ExplicitFees` constraint; the unchanged position and Latched record then let the attacker retry. `OrderLiquidationBatchLogic.sol:313-315` blocks an ordinary manual close while protection stays active. Repeating the public retry keeps one pending child at the observed transaction boundaries.

The owner is not permanently locked. In frozen mode, the same-block and publish-time guards are skipped (`OrderRouterExecutionSidecar.sol:446-452`), so the owner can itself execute the pending child, let it fail terminally, and amend in the next transaction. Once the child deadline passes, the owner can instead expire it. The PoC shows that sequence succeeds if the owner wins the gap; if a stranger's retry is sequenced between the owner's terminalization and amendment, the amendment reverts. A supported smart account may batch terminalization, amendment and retry atomically under the trading-account identity. The canonical trading account documented at `packages/perps/DEPLOYMENT.md:307-308,425-433` is a SimpleAccount v0.8, so this batching escape is relevant to the deployed client, not a hypothetical alternate account.

**Known issue and intended-policy distinction.** Public MED-9 reported keeper retries lacking trader financial ceilings. Fix commit `8a2853f80bce067656c1fd0fcf2dadc5892e2aee` added retained ceilings and the owner amendment function; its amendment gates match current `PositionProtectionBook.sol:315-328`. This finding uses those retained ceilings as designed, then exploits the lack of owner priority after a failed attempt. It does not claim a keeper bypasses the limits. The MED-9 sponsor's stated endpoint was that the owner can amend; the public retry can obstruct that endpoint. `CANONICAL_ENTRYPOINTS.md:142-144` makes the initial trigger irreversible by design, which the PoC does not challenge. `packages/perps/DEPLOYMENT.md:296-297` says a trigger promises neither a fill price nor execution before liquidation; this report claims neither guarantee. It reports the untrusted caller's repeated interference with the documented owner remedy. If market conditions improve enough for the old cap, the next retry can execute without amendment; the PoC's Monday branch shows that exit, after the price opportunity has changed.

## Root Cause

The Latched/no-pending state is shared by the owner-only amendment and anyone's public retry. The book gives the latter no owner-priority window after terminal failure. The Router permits public expiry, and a contract can atomically expire then retry a still-failing child, repeatedly returning to Triggered with one pending order. An owner using separate transactions must win the interval between its own terminalization and amendment; otherwise a third party consumes the amendment state first (`PositionProtectionBook.sol:315-328,381-406,637-667`).

## Impact Explanation

The selected impact is user griefing. The attacker receives no victim margin or bounty; it pays gas to keep the old protection child live. The PoC measures 61 renewals over 3,721 seconds in its short griefing test, one blocked owner amendment call using 34,587 in-call EVM gas units, and the same 10,000-lot position still open while a no-attack owner-amended control closes. Its conditional price-path test separately maintains the denial for 169,199 seconds through the scheduled reopen, with 2,773 renewals and 47 simulated owner races. The call-gas figure excludes transaction intrinsic gas and chain-specific data fees. For an adverse correct price path, the control closes at a 0.90 mark with 20,947.197927 USDG in the trader account; after continuous denial through the frozen window, the market reopens at a 1.00 mark and the trader closes with 19,991.799612 USDG. The measured difference is 955.398315 USDG after the actual path's fees. This market move is conditional and is not caused by the attacker or a bad oracle report. On an unchanged mark, the demonstrated harm is the failed amendment and prolonged unwanted exposure; the PoC makes no guaranteed trading-loss claim.

## Likelihood Explanation

The Friday frozen transition and 50-bps spread are scheduled release behavior. A 30-USDG cap on this position permits normal execution but can reject frozen execution, so the initial failure needs no malicious admin or bad third-party oracle data. An attacker needs public retry/expiry calls, gas for repeated renewals and advantageous transaction ordering. The demonstrated conditional price loss requires 2,773 renewals and 47 successful owner-race insertions across the remaining 169,199-second frozen window; a one-hour block alone does not cause that loss because the frozen 0.90 mark remains executable. The owner can execute or expire a pending child in one transaction and amend in a second; to maintain denial, the attacker must have a retry sequenced between those owner transactions on each attempt. Arbitrum One's FCFS sequencer has no public mempool, so the attacker would need to observe the first owner transaction in a feed and win the next ordering, or submit blind retry transactions. An EOA owner can also pre-sign consecutive-nonce terminalization and amendment transactions and submit them back to back, leaving little opportunity for insertion. The attack window is the wait between the first confirmation and second submission in a sequential wallet flow. A SimpleAccount v0.8 owner able to batch the three calls atomically can escape without that gap. The PoC proves both orderings at the contract level; it does not claim that an attacker controls the sequencer or that the later 1.00 mark is certain.

## Attack Path / Reproduction Path

1. A trader opens a 10,000-lot LONG and sets a protected close with a 30-USDG maximum explicit fee. Normal execution fees fit that bound.
2. A keeper triggers the close just before Friday's frozen boundary. After the boundary, the retained cap rejects the close because the 50-bps frozen spread is included in explicit fees. The protection returns to Latched with no child pending.
3. A stranger calls public retry first. The protection becomes Triggered with a new child. The owner amendment and ordinary close both revert in that state.
4. The stranger renews the child 61 times over 3,721 seconds by atomically expiring and retrying each child after its 60-second window. The position remains open. A failed owner amendment call uses 34,587 measured in-call gas units.
5. If the owner itself executes/terminalizes a child and its amendment is next, the PoC shows amendment and close succeed. If a stranger retry is inserted between those owner transactions, the PoC shows the amendment reverts and the position stays open. A smart-account batch can remove that ordering gap.
6. On a separate conditional path, the attacker renews the child 2,773 times and preempts 47 owner recovery attempts for 169,199 seconds through the roughly 47-hour frozen window. The no-attack amended close at 0.90 leaves 20,947.197927 USDG; after the continuous denial and the next correct 1.00 mark, a live-mode retry closes with 19,991.799612 USDG. The delta is 955.398315 USDG, conditional on both that ordering and the authentic adverse market move.

## External Preconditions

- The victim trading account submits a protection whose signed explicit-fee ceiling is above live fees but below the later frozen spread plus fee, and a scheduled frozen period causes a terminal failure.
- The attacker gets its first public retry sequenced before the owner's amendment, and, when the owner uses separate terminalize/amend transactions, gets a retry between them often enough to maintain the denial. It can use a watcher feed or blind submissions; no privileged sequencer role is assumed.
- The victim submits terminalization and amendment sequentially and waits for the first transaction to confirm before sending the second. An EOA can instead pre-sign consecutive-nonce transactions and send them back to back; the supported SimpleAccount v0.8 can batch the calls atomically. Either approach leaves little or no attacker insertion window.
- The measured 955.398315-USDG price-path loss additionally requires uninterrupted attacker preemption across the remaining 169,199-second frozen window and an authentic market change from 0.90 at the control close to 1.00 at the later close. It is not an oracle-manipulation assumption.

## Internal Preconditions

- The original protection trigger remains active and its first close fails terminally with a typed explicit-fee constraint while the position still matches the stored side and size.
- The record is Latched with no pending child before each race. The stranger's retry changes it to Triggered and creates one pending child under the unchanged cap.
- The child can terminalize or expire; the protection relatches. The attacker needs to win the next ordering against the owner's amendment unless the owner batches the operations.

## Affected Code

- `packages/perps/src/PositionProtectionBook.sol:315-328`: amendment requires Latched status and zero pending orders.
- `packages/perps/src/PositionProtectionBook.sol:381-406,637-667`: permissionless retry occupies that state and terminal failure relatches.
- `packages/perps/src/OrderRouter.sol:191-232,314-329`: public expiry and execution-failure terminal hooks.
- `packages/perps/src/router/OrderLiquidationBatchLogic.sol:313-315,655-677`: manual-close lock and old limits propagated to retry.
- `packages/perps/src/CfdOrderPolicyEvaluatorBase.sol:416,514-516`: frozen spread enters the explicit-fee ceiling that makes the retry fail.

## Recommended Mitigation

Provide an owner operation that terminalizes a pending failed child, amends the retained ceilings and optionally retries atomically through the protocol, including for owners that submit separate transactions. Alternatively, reserve a bounded owner amendment interval after a terminal failure before any public retry may create the next child. Preserve trader authorization over the financial ceilings and ensure a stale child callback cannot restore the old limits.

## Proof of Concept

### Description

The single authored test file uses production protection and settlement paths in the pinned fixture. Four tests demonstrate: 61 attacker renewals and a still-open position; an owner-amended no-attack control; a two-transaction owner escape and attacker insertion between the two owner calls; and a conditional adverse correct-price control comparing actual trader balances after closing at 0.90 versus 1.00. The test prints the 34,587-unit failed owner-call measurement, 2,773 renewals and 47 owner-race insertions across 169,199 seconds, and the 955.398315-USDG balance difference. The canonical SimpleAccount batch is disclosed as an escape; it is not emulated by the EOA fixture.

### Where to place files and their names

Clone the pinned repository and save the complete authored source block below at `packages/perps/test/perps/poc/ProtectionAmendStarvationE2E.t.sol`. No other PoC file is authored. Source SHA-256: `8f16bee802c52018f2f071985cf0eac457c8ef6155f3a90732644fcfa73e061e`. Fresh log SHA-256: `266050a2c2ea38c8308aee46630117f19e0e3ddc5a03091fbdc37abb06a69f62`.

### Command

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git audit-comp-plether
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
mkdir -p packages/perps/test/perps/poc
# Save the authored source block below at packages/perps/test/perps/poc/ProtectionAmendStarvationE2E.t.sol
cd packages/perps
forge test --offline --gas-limit 10000000000 --match-path test/perps/poc/ProtectionAmendStarvationE2E.t.sol --match-test 'test_(AttackerKeepsOwnerFromAmendingForOverOneHour|ConditionalAdversePriceOpportunityAgainstControl|Control_OwnerAmendsWhenAttackerDoesNotRenew|OwnerTwoTransactionEscapeAndAttackerRace)' -vv
```

### Fresh run log

```text
No files changed, compilation skipped

Ran 4 tests for test/perps/poc/ProtectionAmendStarvationE2E.t.sol:ProtectionAmendStarvationE2E
[PASS] test_AttackerKeepsOwnerFromAmendingForOverOneHour() (gas: 66973604)
Logs:
  first renewal timestamp 1709935201
  last renewal timestamp 1709938922
  frozen spread bps 50
  initial explicit fee cap USDG atoms 30000000
  blocked seconds 3721
  renewal rounds 61
  victim open size 10000000000000000000000

[PASS] test_ConditionalAdversePriceOpportunityAgainstControl() (gas: 2733708025)
Logs:
  control close balance assets 20947197927
  delayed close balance assets 19991799612
  adverse-path loss atoms 955398315
  first attack timestamp 1709935201
  scheduled reopen timestamp 1710104400
  continuous denial seconds 169199
  renewal rounds through reopen 2773
  owner recovery races preempted 47

[PASS] test_Control_OwnerAmendsWhenAttackerDoesNotRenew() (gas: 7558607)
Logs:
  control amended cap USDG atoms 100000000
  control victim open size 0

[PASS] test_OwnerTwoTransactionEscapeAndAttackerRace() (gas: 10449753)
Logs:
  owner two-transaction escape size 0
  attacker preemption child 3
  victim failed amend call gas incl test call 34587
  attacker race still-open size 10000000000000000000000

Suite result: ok. 4 passed; 0 failed; 0 skipped; finished in 4.93s (4.97s CPU time)

Ran 1 test suite in 4.94s (4.93s CPU time): 4 tests passed, 0 failed, 0 skipped (4 total tests)
```

### Full authored source

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {PositionProtectionTest} from "../spec/trader/PositionProtection.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Untrusted actor. An atomic expiry/retry can occupy an amendment window;
///         the owner can also terminalize the child and amend in two transactions.
contract ProtectionRenewalActor {
    function renew(address router, address book, uint64 attemptId, uint64 protectionId)
        external
        returns (uint64)
    {
        (bool ok,) = router.call(abi.encodeWithSignature("expireOrder(uint64)", attemptId));
        require(ok, "expiry failed");
        return IPositionProtectionActions(book).retryPositionProtectionClose(protectionId);
    }

    function retry(address book, uint64 protectionId) external returns (uint64) {
        return IPositionProtectionActions(book).retryPositionProtectionClose(protectionId);
    }
}

contract ProtectionAmendStarvationE2E is PositionProtectionTest {
    ProtectionRenewalActor internal actor;
    // The fixture's Friday FAD begins 30 minutes before the FX close.
    uint256 internal constant FRIDAY_FROZEN_START = 1_709_933_400 + 30 minutes;
    uint256 internal constant CAP_BEFORE_FREEZE = 30e6;
    uint256 internal constant CAP_AFTER_FREEZE = 100e6;

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory params) {
        params = super._riskParams();
        params.vpiFactor = 0.01e18;
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 30_000_000e6;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 20_000_000e6;
    }

    function test_AttackerKeepsOwnerFromAmendingForOverOneHour() public {
        _startRecordingLogs();
        actor = new ProtectionRenewalActor();
        (uint64 protectionId, uint64 firstAttempt) = _triggerBeforeFreeze();
        _failFirstAttemptInFreeze(protectionId, firstAttempt);

        uint64 attemptId = actor.retry(address(protectionActions), protectionId);
        uint256 firstRenewalTime = vm.getBlockTimestamp();
        for (uint256 i; i < 61; ++i) {
            uint256 deadline = router.lifecycleBook().orderTiming(attemptId).executionDeadline;
            uint256 nextTime = deadline + 1;
            if (nextTime <= vm.getBlockTimestamp()) nextTime = vm.getBlockTimestamp() + 61;
            vm.warp(nextTime);
            attemptId = actor.renew(address(router), address(protectionActions), attemptId, protectionId);
            _assertOwnerAmendBlocked(protectionId);
        }

        console2.log("first renewal timestamp", firstRenewalTime);
        console2.log("last renewal timestamp", vm.getBlockTimestamp());
        assertGe(vm.getBlockTimestamp() - firstRenewalTime, 3600, "owner blocked for at least one hour");
        assertEq(router.pendingOrderCounts(ALICE), 1, "attacker keeps exactly one live child");
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(CfdTypes.Side.LONG, POSITION_SIZE, 0, 0, true);
        (uint256 remainingSize,,,,,,) = engine.positions(ALICE);
        assertEq(remainingSize, POSITION_SIZE, "victim position cannot exit through protection");
        console2.log("frozen spread bps", engine.frozenCloseSpreadBps());
        console2.log("initial explicit fee cap USDG atoms", CAP_BEFORE_FREEZE);
        console2.log("blocked seconds", vm.getBlockTimestamp() - firstRenewalTime);
        console2.log("renewal rounds", uint256(61));
        console2.log("victim open size", remainingSize);
    }

    function test_Control_OwnerAmendsWhenAttackerDoesNotRenew() public {
        _startRecordingLogs();
        actor = new ProtectionRenewalActor();
        (uint64 protectionId, uint64 firstAttempt) = _triggerBeforeFreeze();
        _failFirstAttemptInFreeze(protectionId, firstAttempt);

        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _amendedLimits());
        uint64 retryId = actor.retry(address(protectionActions), protectionId);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(retryId, _priceData(LONG_TAKE_PROFIT));
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "amended close works");
        (uint256 remainingSize,,,,,,) = engine.positions(ALICE);
        assertEq(remainingSize, 0, "control owner exits");
        console2.log("control amended cap USDG atoms", CAP_AFTER_FREEZE);
        console2.log("control victim open size", remainingSize);
    }

    function test_OwnerTwoTransactionEscapeAndAttackerRace() public {
        _startRecordingLogs();
        actor = new ProtectionRenewalActor();
        (uint64 protectionId, uint64 firstAttempt) = _triggerBeforeFreeze();
        _failFirstAttemptInFreeze(protectionId, firstAttempt);
        uint64 child = actor.retry(address(protectionActions), protectionId);
        uint256 startingState = vm.snapshotState();

        // A victim EOA can itself terminalize the child, then amend in its next nonce.
        vm.prank(ALICE);
        OrderV3Types.ExecutionResult memory failed = router.executeOrder(child, _priceData(LONG_TAKE_PROFIT));
        assertEq(uint8(failed.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(router.pendingOrderCounts(ALICE), 0);
        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _amendedLimits());
        uint64 recovered = actor.retry(address(protectionActions), protectionId);
        OrderV3Types.ExecutionResult memory closed = router.executeOrder(recovered, _priceData(LONG_TAKE_PROFIT));
        assertEq(uint8(closed.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        (uint256 ownerWinSize,,,,,,) = engine.positions(ALICE);
        assertEq(ownerWinSize, 0, "owner wins two-transaction race");
        console2.log("owner two-transaction escape size", ownerWinSize);

        assertTrue(vm.revertToState(startingState));
        vm.prank(ALICE);
        failed = router.executeOrder(child, _priceData(LONG_TAKE_PROFIT));
        assertEq(uint8(failed.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(router.pendingOrderCounts(ALICE), 0);
        uint64 attackerChild = actor.retry(address(protectionActions), protectionId);
        vm.prank(ALICE);
        uint256 beforeGas = gasleft();
        (bool ownerAmendOk,) = address(protectionActions).call(
            abi.encodeCall(IPositionProtectionActions.amendLatchedPositionProtectionCloseLimits,
                (protectionId, _amendedLimits()))
        );
        uint256 ownerCallGas = beforeGas - gasleft();
        assertFalse(ownerAmendOk, "attacker wins inter-transaction race");
        assertEq(router.pendingOrderCounts(ALICE), 1);
        (uint256 ownerLosesSize,,,,,,) = engine.positions(ALICE);
        assertEq(ownerLosesSize, POSITION_SIZE);
        console2.log("attacker preemption child", uint256(attackerChild));
        console2.log("victim failed amend call gas incl test call", ownerCallGas);
        console2.log("attacker race still-open size", ownerLosesSize);
    }

    function test_ConditionalAdversePriceOpportunityAgainstControl() public {
        _startRecordingLogs();
        actor = new ProtectionRenewalActor();
        (uint64 protectionId, uint64 firstAttempt) = _triggerBeforeFreeze();
        _failFirstAttemptInFreeze(protectionId, firstAttempt);
        uint256 branch = vm.snapshotState();

        // With no hostile retry, the owner closes at 0.90 after amending.
        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _amendedLimits());
        uint64 controlChild = actor.retry(address(protectionActions), protectionId);
        OrderV3Types.ExecutionResult memory control =
            router.executeOrder(controlChild, _priceData(LONG_TAKE_PROFIT));
        assertEq(uint8(control.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        uint256 controlBalance = clearinghouse.balanceAssets(ALICE);
        (uint256 controlSize,,,,,,) = engine.positions(ALICE);
        assertEq(controlSize, 0);

        assertTrue(vm.revertToState(branch));
        vm.warp(FRIDAY_FROZEN_START + 1);
        assertTrue(engine.isOracleFrozen(), "attack branch begins in freeze");
        uint64 attackChild = actor.retry(address(protectionActions), protectionId);
        uint256 firstAttackTime = vm.getBlockTimestamp();
        uint256 reopenTime = FRIDAY_FROZEN_START + 47 hours;
        uint256 renewals;
        uint256 ownerRaces;
        // A child is always pending at a transaction boundary. At each
        // simulated hourly owner recovery, the attacker wins the gap between
        // the owner's terminalization and its separately submitted amendment.
        while (true) {
            uint256 deadline = router.lifecycleBook().orderTiming(attackChild).executionDeadline;
            if (deadline + 1 >= reopenTime) break;
            vm.warp(deadline + 1);
            assertTrue(engine.isOracleFrozen(), "the old mark remains executable during denial");
            if (renewals % 60 == 0) {
                vm.prank(ALICE);
                router.expireOrder(attackChild);
                attackChild = actor.retry(address(protectionActions), protectionId);
                vm.prank(ALICE);
                (bool amended,) = address(protectionActions).call(
                    abi.encodeCall(IPositionProtectionActions.amendLatchedPositionProtectionCloseLimits,
                        (protectionId, _amendedLimits()))
                );
                assertFalse(amended, "attacker preempts owner's second transaction");
                ++ownerRaces;
            } else {
                attackChild = actor.renew(address(router), address(protectionActions), attackChild, protectionId);
            }
            ++renewals;
            assertEq(router.pendingOrderCounts(ALICE), 1, "one live child at every boundary");
        }
        assertGe(renewals, 2700, "denial covers the remaining frozen window");
        assertGe(ownerRaces, 40, "repeated owner attempts are preempted");
        // The attacker maintains the child until the scheduled reopen. A
        // correct new mark at 1.00 then removes the old 0.90 opportunity.
        vm.warp(reopenTime);
        assertFalse(engine.isOracleFrozen(), "market has reopened");
        uint256 lastDeadline = router.lifecycleBook().orderTiming(attackChild).executionDeadline;
        if (lastDeadline >= block.timestamp) vm.warp(lastDeadline + 1);
        uint64 lateChild = actor.renew(address(router), address(protectionActions), attackChild, protectionId);
        _refreshMark(MARK_PRICE);
        vm.warp(block.timestamp + 1);
        vm.roll(block.number + 1);
        OrderV3Types.ExecutionResult memory late = router.executeOrder(lateChild, _livePriceData(MARK_PRICE));
        assertEq(uint8(late.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        uint256 lateBalance = clearinghouse.balanceAssets(ALICE);
        (uint256 lateSize,,,,,,) = engine.positions(ALICE);
        assertEq(lateSize, 0);
        assertGt(controlBalance, lateBalance, "adverse correct mark erases owner price opportunity");
        console2.log("control close balance assets", controlBalance);
        console2.log("delayed close balance assets", lateBalance);
        console2.log("adverse-path loss atoms", controlBalance - lateBalance);
        console2.log("first attack timestamp", firstAttackTime);
        console2.log("scheduled reopen timestamp", reopenTime);
        console2.log("continuous denial seconds", reopenTime - firstAttackTime);
        console2.log("renewal rounds through reopen", renewals);
        console2.log("owner recovery races preempted", ownerRaces);
    }

    function _triggerBeforeFreeze() internal returns (uint64 protectionId, uint64 attemptId) {
        _setSeniorCapacity(40_000_000e6, 8000);
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6);
        assertEq(pool.maxSeniorShareBps(), 8000);
        assertEq(engine.frozenCloseSpreadBps(), 50, "release frozen spread");
        assertEq(engine.executionFeeBps(), 4, "release execution fee");
        _refreshMark(MARK_PRICE);
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        PositionProtectionTypes.PositionProtectionParams memory params = _params(LONG_TAKE_PROFIT, 0);
        params.closeLimits.maxExplicitFeesAssets = CAP_BEFORE_FREEZE;
        vm.prank(ALICE);
        protectionId = protectionActions.createPositionProtection(params);

        vm.warp(FRIDAY_FROZEN_START - 30);
        _refreshMark(MARK_PRICE);
        vm.warp(FRIDAY_FROZEN_START - 29);
        attemptId = _triggerAt(protectionId, LONG_TAKE_PROFIT);
    }

    function _failFirstAttemptInFreeze(uint64 protectionId, uint64 attemptId) internal {
        vm.warp(FRIDAY_FROZEN_START + 1);
        assertTrue(engine.isOracleFrozen(), "frozen period reached");
        OrderV3Types.ExecutionResult memory result = router.executeOrder(attemptId, _priceData(LONG_TAKE_PROFIT));
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Failed), "cap fails in freeze");
        assertEq(uint8(result.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation));
        OrderV3Types.CompactOutcome memory outcome =
            _verifiedOutcome(IOrderLifecycleBook(address(router.lifecycleBook())), attemptId);
        assertEq(uint8(outcome.failedConstraint), uint8(OrderV3Types.ConstraintKind.ExplicitFees));
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched)
        );
        assertEq(router.pendingOrderCounts(ALICE), 0, "owner could amend before public retry");
    }

    function _assertOwnerAmendBlocked(uint64 protectionId) internal {
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _amendedLimits());
    }

    function _amendedLimits() internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory limits) {
        limits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: CAP_AFTER_FREEZE
        });
    }
}
```
